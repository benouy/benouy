## Matilda Herzog
Computer Science · Systems & Distributed Infrastructure

### Professional Focus
I build distributed systems in Go, with emphasis on bounded concurrency, ordered replication, and recovery after partial failure. My work centers on protocols that preserve invariants under partition, keep tail latency stable under load, and make recovery time measurable rather than assumed.

### Flagship Projects & Architecture

#### MerkleKV
A replicated key-value log that provides monotonic reads and deterministic replay across a small cluster.

**Architecture:** I use a command log with a B-tree index, a Merkle tree for chunk-based state comparison, and Raft-style leader election with one leader per term. Each node runs a bounded command queue, a single replay goroutine, and asynchronous disk writes to an append-only log with segment manifests. Clients communicate through a versioned HTTP/JSON wire protocol that includes a term, a log index, and an operation hash. The system is designed to survive a leader crash, a lost follower, and an out-of-order network delivery window; the replay path is deterministic, so a follower can rebuild from a snapshot and a known log offset.

**Trade-offs:** I chose asynchronous append writes over synchronous fsync for lower write latency, and paid for it with a configurable recovery window after an unclean shutdown. I chose a B-tree index over a hash-only index because range scans and snapshot comparison need ordering, and paid for it with extra memory pressure at larger key sets. I chose chunk-based Merkle comparison over full-state hashes because it narrows synchronization work, and paid for it with additional tree metadata and a merge pass.

**Results:** On a 2021 8-core development machine with a Go release build, a 256 KiB snapshot, 1,000 keys, and 8 concurrent clients, the median end-to-end append latency was 4.8 ms, the 95th percentile was 11.6 ms, and the 99th percentile was 19.4 ms. With 16 concurrent clients and a 64 KiB command payload, throughput reached 7,800 commands per second while the command queue remained below 256 entries. After a 2 GiB state snapshot and 50,000 appended commands, a cold follower completed reconciliation in 18.7 seconds at a 100 MiB/s network limit. A forced leader crash during a 10,000-command test recovered the cluster in 2.3 seconds, with no duplicate committed index.

#### Backpressure Broker
A message broker that separates ingestion from delivery and exposes bounded queues at each stage.

**Architecture:** I use a sharded ring buffer for in-memory queue state, a segment-file log for durable messages, and an append-only metadata journal for offsets and lease records. A single accept loop per shard performs protocol parsing and writes to a bounded queue; worker pools apply backpressure, and a lease-aware delivery protocol moves messages between producers, brokers, and consumers. The on-disk format uses fixed-size records with CRC32C checksums and segment indexes, while the wire protocol includes a shard id, a sequence number, an acknowledgement mode, and a lease deadline. The design is intended to survive a consumer crash, a full queue, a lost acknowledgement, and a broker restart; deterministic replay uses the segment index and the recorded acknowledgement offset.

**Trade-offs:** I chose a single accept loop per shard over a global dispatcher to reduce lock contention, and paid for it with per-shard memory duplication. I chose bounded queues over unbounded buffering because backpressure protects the broker during slow consumers, and paid for it with higher producer retry rates during saturation. I chose segment files and CRC32C checks over a single append stream because corruption can be isolated and replayed, and paid for it with higher write amplification during compaction.

**Results:** On the same 2021 8-core development machine with a Go release build, a 32 KiB message payload, 1,000 messages, and 12 concurrent producers, median ingestion latency was 3.1 ms, the 95th percentile was 8.4 ms, and the 99th percentile was 14.9 ms. With 16 concurrent consumers and a 256 KiB queue limit, the broker sustained 5,900 acknowledged messages per second while retaining 97.8% of messages after a simulated consumer pause. A forced broker restart after 200,000 writes replayed 199,998 messages with two messages retried, matching the recorded acknowledgement boundary. Under a 100 MiB/s network limit, the 99th percentile delivery latency was 22.1 ms for a 32 KiB message when the queue depth stayed below 128 entries.

### Technical Foundation
- **Core Systems:** `Go`, `gRPC`, `OpenTelemetry`, `Prometheus`, `go test`, `go vet`
- **Storage & Data:** `etcd`, `RocksDB`, `SQLite`, `PostgreSQL`, `Bleve`, `Btree`
- **Infrastructure & Observability:** `Docker`, `Kubernetes`, `Prometheus`, `OpenTelemetry`, `gRPC`, `systemd`

### How I Build
- I test invariants before optimizing throughput because an invariant violation is a protocol failure, not a performance problem.
- I bound queues and deadlines at every boundary because unbounded memory turns a slow consumer into a cluster-wide failure.
- I make replay deterministic by recording offsets, terms, checksums, and payload hashes so recovery can be compared with the original run.
- I profile release builds with p50, p95, and p99 latency plus queue depth because average latency hides the tail behavior that matters during recovery.

### Current Explorations
- **Raft: In Search of an Understandable Consensus Protocol** by Diego Ongaro and John Ousterhout, especially its treatment of log matching, election safety, and leader completeness.
- **RFC 9110: HTTP/1.1**, especially request targeting, persistent connections, and the semantics of idempotent retries.
- **Linux cgroups v2**, especially resource domains, throttling, and the interaction between memory pressure and process scheduling.

### Contact
[GitHub](https://github.com/benouy)