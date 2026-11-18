# sf-blobfs: a filesystem for LSM / btree engines on raw NVMe

A minimal filesystem over SPDK or io_uring_cmd NVMe passthrough, purpose-built
for storage engines that write large blocks: LSM trees (SSTs + WAL) and btree
page stores. Layers on super_flight's `sf::raw::Namespace` (see DESIGN.md), so
the I/O backend (io_uring block path, io_uring_cmd passthrough, SPDK via
xNVMe) is pluggable underneath and tokio integration comes from the runtime
layer, not from the filesystem.

## Why this is tractable

A general filesystem is a multi-year project. An LSM/btree-scoped one is not,
because the workload deletes the hard parts:

- Thousands of large files (MBs-GBs), not millions of small ones: the whole
  file table fits in RAM permanently.
- SSTs are write-once: created, appended sequentially, sealed, read-only,
  then deleted whole. No overwrite, no sparse files, no mmap, no hardlinks.
- Compaction above the FS is the garbage collector: whole-file delete frees
  whole extents, so the FS needs no GC, no defragmentation daemon.
- No directories (flat namespace + name prefixes), no permissions, no xattrs.

Prior art to study: Ceph BlueFS (metadata-journal design, built to host
RocksDB on raw devices) and SPDK BlobFS/blobstore (extent/cluster design).

## On-disk format

```
LBA 0      superblock A        (4 KB, generation + pointer to journal)
LBA k      superblock B        (alternate; recovery picks higher valid gen)
           metadata journal    (extent-chained log of file-table mutations)
           data extents        (the rest of the namespace)
```

- **Superblocks**: two copies, checksummed, generation-numbered; written
  alternately. Recovery reads both, takes the newest valid one.
- **Metadata journal**: every mutation (create, extent-append, seal, delete,
  rename) is a checksummed record. The full file table is rebuilt into RAM by
  replay at mount. When the journal exceeds a threshold, it is compacted: the
  current table is written as one snapshot record to a fresh extent chain and
  the superblock flips to it.
- **File table (RAM, always)**: name -> {id, state (open/sealed), length,
  extent list, block size, mode (blob | page-region)}.
- **Allocator**: extent-based, large extents (1-64 MB, configurable per
  volume). Free space tracked as a free-extent list by size class. An SST is
  a handful of extents; large reads are near-sequential on the device.

## File model and crash consistency

Blob mode (LSM): `create -> append -> seal -> read -> delete`.

- Appends write data extents first, then journal the new length. Only
  sealing makes the length authoritative: on recovery, an unsealed file is
  truncated to its last journaled length, so torn tails are harmless.
- Sealed files are immutable: no torn-write exposure at all.
- WAL files are the exception (append + fdatasync-equivalent): each
  sync journals the durable length, and flush maps to NVMe Flush (FUA
  writes where supported).

Page mode (btree): a file is a fixed-size page region.

- Aligned in-place page writes at page granularity.
- Torn-page protection: if the device's atomic write limit (AWUPF /
  NAWUPF from Identify) covers the page size, rely on it; otherwise
  copy-on-write page pairs with a per-page generation bit.

All data I/O is aligned direct I/O; data block sizes 64 KB-1 MB (the
large-block design point). Per-block checksums are optional (engines like
RocksDB carry their own); metadata is always checksummed.

## Concurrency model

- Single writer per file (natural for SSTs and WALs); any number of readers.
- Metadata mutations serialize through one journal writer (an actor/mutex;
  mutation rate is low: file-level events only).
- Data-path reads/writes go straight to the raw namespace via extent lookup,
  no metadata lock on the hot path (extent lists of sealed files are
  immutable; open files pin their tail extent).

## Milestones

| M | Deliverable | Gate |
|---|---|---|
| M0 | Format spec + deterministic device simulator (trait over "the device" with power-cut injection at any write boundary) | property tests + 10k randomized crash points replay to a consistent table |
| M1 | Format bring-up on a plain file via psync backend: superblocks, journal, create/append/seal/read/delete, mount/recovery | M0 crash suite passes against the real code |
| M2 | LSM adapter trait (RocksDB-Env-shaped: sequential writer, random reader, rename, sync) | synthetic LSM workload (SST writes + WAL appends + deletes + reopen) runs end to end |
| M3 | Real device: io_uring block path, then io_uring_cmd passthrough (LBA translation, alignment, Identify-driven geometry) | same suite green on /dev/ngXnY |
| M4 | Async API via tokio (super_flight embedded flavor) | concurrent readers + writer under multi-thread tokio; no blocking in workers |
| M5 | SPDK backend via xNVMe, polled completions | correctness suite green; CPU-per-IO vs passthrough measured |
| M6 | Page mode for btree engines | crash-injection verifies no torn pages on devices with and without atomic-write support |
| M7 | Benchmarks vs ext4/XFS + O_DIRECT on identical hardware | SST ingest MB/s, random 64K-1MB read IOPS/latency, mixed-load p99; published in repo |

M0 is the highest-leverage step: a deterministic simulator makes crash
consistency testable thousands of times per second before touching hardware,
and every later backend must pass the same suite.

## Open questions

- Extent size fixed per volume vs per file class (WAL wants smaller)?
- Should seal also record a whole-file checksum for scrub?
- Snapshot/clone of sealed files (cheap: shared extents + refcount) now or
  later?
- Multi-namespace layout: journal on its own namespace for isolation?
