# super_flight — a completion-first async I/O framework

A runtime like tokio, but scoped to I/O only, with pluggable backends
(io_uring, xNVMe, portable fallback) and pluggable I/O schedulers.

## Core design decision: completion-first

Tokio's reactor is readiness-based (epoll): "tell me when I can read, then I
read." That model fits sockets but is wrong for files and NVMe, where the
kernel interfaces that matter (io_uring, xNVMe, SPDK) are completion-based:
"here is the operation and the buffer; tell me when it is done."

super_flight is designed completion-first. The public API is "submit an
operation, await its completion." Readiness-style backends are emulated behind
that API (via a thread pool), never the reverse.

Two consequences follow:

1. **Owned buffers.** Once an op is submitted, the kernel owns the buffer
   until completion. APIs therefore take and return owned buffers
   (`Vec<u8>` / pooled buffers), not `&mut [u8]` borrows.
2. **Safe cancellation.** Dropping a future before its op completes must not
   free the buffer. The in-flight op table, not the future, owns the buffer;
   a dropped future orphans the op, and the buffer is released when the
   completion is reaped.

## Layers

Workspace with one crate per layer:

```
crates/
  sf-core      # Backend trait, op types, completion types. No runtime.
  sf-fallback  # Blocking thread-pool backend. Portable; reference impl.
  sf-uring     # io_uring backend (Linux).
  sf-xnvme     # xNVMe backend (FFI).
  sf-sched     # Pluggable scheduler policies (FIFO, deadline, fair-queue).
  sf-runtime   # Thread-per-core drivers, futures integration, buffer pool.
examples/
benches/
```

### sf-core: the pluggable contract

```rust
trait Backend {
    fn submit(&mut self, ops: &mut OpQueue) -> io::Result<usize>;
    fn reap(&mut self, completions: &mut Vec<Completion>, timeout: Option<Duration>) -> io::Result<usize>;
    fn capabilities(&self) -> Caps; // registered buffers? fixed files? polling?
}
```

Fixed, small op set to start: `Open`, `Read`, `Write`, `Fsync`, `Close`.
Later: vectored ops, registered-buffer variants, `Fallocate`, `Statx`.
Each op carries a user token so completions route back to wakers.

### Backends

- **sf-fallback**: a bounded blocking thread pool executing ops with plain
  syscalls. Runs on any OS; doubles as the reference implementation for
  differential testing.
- **sf-uring**: starts with plain submission + `io_uring_enter`; then adds
  registered files, registered buffers, and SQPOLL as capability-gated
  features. Built on the `io-uring` crate.
- **sf-xnvme**: FFI to xNVMe queue pairs, for NVMe passthrough and
  user-space queues.

### sf-runtime

- Thread-per-core: each core owns one backend instance, one op table, one
  local task queue. No work stealing initially; futures are `!Send`.
- Driver loop: poll tasks -> flush scheduler -> submit batch -> reap
  completions -> wake wakers.
- Buffer pool with alignment support (O_DIRECT needs aligned buffers);
  pool-backed buffers map onto registered buffers when the backend has them.

### sf-sched: the scheduler layer

A `Scheduler` trait sits between user submissions and the backend:

```rust
trait Scheduler {
    fn enqueue(&mut self, op: Op, class: IoClass);
    fn next_batch(&mut self, budget: usize, out: &mut OpQueue);
    fn on_complete(&mut self, c: &Completion);
}
```

It decides ordering, batching, and in-flight depth per class. Initial
policies: FIFO (baseline), Deadline (latency-sensitive reads jump the line),
and fair queueing across classes with per-class rate limits. This is where
foreground point-reads get protected from background scans.

### Observability

Built in from M1: in-flight depth, queue depth per class, submission batch
size, completion latency histograms. Exposed as a snapshot API; no logging
in the hot path.

## Milestones

| M | Deliverable | Gate |
|---|---|---|
| M1 | sf-core + sf-fallback + minimal runtime | e2e test: open/read/write/fsync a file through the runtime |
| M2 | sf-uring | same test suite passes; differential test vs fallback on random op sequences |
| M3 | buffer pool + O_DIRECT | aligned reads verified; registered buffers behind capability flag |
| M4 | sf-sched | policy A/B on synthetic mixed load (point reads vs scans) shows deadline policy protecting read p99 |
| M5 | sf-xnvme | op test suite passes on an NVMe namespace |
| M6 | bench harness | QD sweep, 4K randread, 128K seq, mixed — vs tokio, tokio-uring, glommio |

## Open questions

- Cancellation surface: expose explicit `cancel()` (io_uring supports async
  cancel) or only drop semantics?
- `Send` futures later via a sharded runtime, or stay thread-per-core forever?
- Does sf-sched live above the backend (portable) or exploit io_uring chained
  ops / priorities when available? Likely both: portable core, capability
  hooks.
