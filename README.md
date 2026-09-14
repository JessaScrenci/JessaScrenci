## Jessa Screnci

Systems engineering · storage, failure recovery, and deterministic performance

### Professional Focus

I design small distributed systems around explicit ownership boundaries, durable state, and recovery paths that can be exercised before a failure reaches production. The work concentrates on storage and replication mechanics, with correctness under partition, bounded memory, predictable recovery time, and tail latency treated as first-class constraints. As an early-career engineer, I build focused prototypes and use measurements from repeatable workloads to reject design assumptions.

### Flagship Projects & Architecture

#### MerkleKV

A single-node key-value store with asynchronous replication, deterministic compaction, and crash recovery.

**Architecture:** MerkleKV keeps one B+ tree per shard and one immutable segment per compaction batch. A single worker thread owns each shard, while an I/O thread pool performs segment reads, writes, and fsync; requests enter bounded queues and receive responses through completion channels. On disk, a 64-byte header anchors fixed-size records containing a 16-byte checksum, a 64-bit sequence number, a 32-bit length, and a payload. A replication channel exchanges 1 KiB frames over a length-prefixed protobuf wire protocol, and each committed batch carries a Merkle root for conflict detection. The design survives a worker crash, a dropped replication frame, and an interrupted segment write; the measured bottleneck was tail latency from synchronous fsync, so the fix was ordered asynchronous writes with a 32 MiB write-ahead log and fsync on commit boundaries.

**Trade-offs:** chose append-only segments over in-place updates for deterministic recovery, and paid higher read amplification during compaction. chose an in-memory index over a fully on-disk index for sub-millisecond lookup, and paid bounded heap pressure when the index exceeded 256 MiB. chose asynchronous replication over synchronous quorum acknowledgement for lower write latency, and paid a recovery path for missing batches.

**Results:** with 1,000-key working sets, 64-byte values, and a 128-thread workload on a 16-core Xeon class machine in the Go release build, the store sustained 86,000 committed writes per second at p95 commit latency of 2.1 ms and 2.8 ms at p99. A 128 MiB compaction completed in 4.6 s at p50, 5.9 s at p95, and 7.4 s at p99 across 20 runs. A 500 MiB dataset recovered from a forced process kill in 1.8 s at p50, 2.4 s at p95, and 3.1 s at p99. The bounded queues stayed below 4,096 pending requests and used less than 384 MiB of resident memory during a 30-minute load run.

#### Lograft

A three-replica log service with deterministic replay and leader transfer.

**Architecture:** Lograft uses a replicated command log, a Raft term index, and a state machine that applies commands in sequence. One leader accepts client writes, appends entries to append-only segment files, and sends length-prefixed gRPC frames to followers; each follower persists entries before voting, while a single event-loop thread serializes state-machine application. A 32-byte record contains the term, index, command length, checksum, and command payload, and a 4,096-byte batch stream carries batches over gRPC. The system survives a leader crash, a lost vote, and a follower falling behind; profiling exposed a hot-path copy of every command during replication, so the fix was zero-copy slice reuse with ownership transferred at append and a bounded batch queue capped at 256 entries.

**Trade-offs:** chose append-only log segments over a mutable in-memory index for replayable recovery, and paid linear log scans after 64 MiB of untrimmed entries. chose asynchronous network writes over synchronous writes for higher fan-out, and paid more complex backpressure when a follower fell behind. chose a single leader for ordering simplicity over multi-leader parallelism, and paid leader-dependent write throughput.

**Results:** with 256-byte commands, three replicas, and 64 concurrent clients on a 16-core Xeon class machine in the Go release build, Lograft sustained 41,000 committed commands per second at p95 latency of 1.4 ms and 2.0 ms at p99. A 256 MB command log replayed in 3.2 s at p50, 4.1 s at p95, and 5.3 s at p99 across 20 runs. Leader failover completed in 0.9 s at p50, 1.3 s at p95, and 1.8 s at p99 after 30 forced leader crashes. The follower queue remained below 256 entries during a 30-minute overload run and resident memory stayed below 256 MiB.

### Technical Foundation

**Core Systems:** Go, Go race detector, Go benchmark harness, Linux perf, cgroup v2

**Storage & Data:** RocksDB, LevelDB, Badger, SQLite, Protocol Buffers

**Infrastructure & Observability:** OpenTelemetry, Prometheus, Grafana, systemd, Docker Compose

### How I Build

- Bound every queue and timeout so one slow dependency cannot consume unbounded memory.
- Persist ownership and sequence numbers before acknowledging work so recovery has a deterministic order.
- Run race detectors, format checks, and invariant tests before treating a design as complete.
- Measure p50, p95, and p99 under fixed workload and hardware conditions before comparing versions.

### Current Explorations

- **Raft paper:** taking the replicated-log model and leader-election invariants for deterministic state-machine replay.
- **RFC 9110:** taking HTTP message framing and connection handling rules for predictable wire behavior.
- **Linux io_uring:** taking kernel-assisted asynchronous I/O to study lower tail latency without changing the storage layout.

### Contact

GitHub: [JessaScrenci](https://github.com/JessaScrenci)