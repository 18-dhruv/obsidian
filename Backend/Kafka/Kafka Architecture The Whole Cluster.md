# Kafka Architecture: The Whole Cluster

Oct 31, 2026 · @dhruv

Related: [[How a Kafka Consumer Works End to End]]

## At a glance

Kafka is brokers storing replicated partition logs, plus a controller quorum that owns the cluster's metadata.

![[kafka-cluster-overview.png]]

Topic `orders` has 3 partitions and replication factor 2: every broker leads one partition and follows another. Clients only talk to leaders; the controllers stay out of the data path.

## Layer 1: Storage — what a partition really is

A topic is just a name; the real unit is the **partition**, an append-only log of files on one broker's disk.

- **Segments.** A partition is a folder of segment files, 1 GiB each by default (`log.segment.bytes`). Only the newest, the *active* segment, takes writes. Each segment has an **offset index** (`log.index.interval.bytes`, every 4 KiB) so the broker can jump to an offset without scanning ([Broker configs](https://kafka.apache.org/43/configuration/broker-configs/)).
- **Dhruv asked:** So KRaft maintains all the offsets per segment?
- **Claude answered:** No. KRaft (the controllers) never sees segments or message offsets; it only tracks cluster metadata such as which brokers exist and who leads each partition. Segments and their offset indexes are local files that each broker manages on its own disk, for its own replicas. Each segment file is even named after its first offset (e.g. `00000000000000368769.log` with a matching `.index`), so to find offset 368800 the broker picks that segment by name, then uses the index to jump close and scans the last few KiB. Consumer *committed* offsets are a third thing again: they live in `__consumer_offsets`, managed by the group coordinator.
- **Retention** works on whole segments: by default old segments are deleted after 7 days (`log.retention.hours=168`). **Compaction** (`cleanup.policy=compact`) instead keeps at least the last value per key, which is how `__consumer_offsets` works ([Design: Log Compaction](https://kafka.apache.org/43/design/design/#log-compaction)).
- **Why it's fast.** Kafka writes sequentially, relies on the OS page cache instead of its own heap cache, and sends bytes from that cache straight to the socket with `sendfile` (zero-copy). Consumers that are caught up are served from memory, with no disk reads ([Design: Persistence, Efficiency](https://kafka.apache.org/43/design/design/#persistence)).
- **Batching end to end.** The producer sends batches, the broker appends them to the log as-is, and consumers fetch large linear chunks. Compressed batches stay compressed on disk and on the wire.

## Layer 2: Replication — how data survives a dead broker

Every partition is copied to N brokers (the replication factor, often 3): one **leader** takes all writes, the rest are **followers** that fetch from the leader exactly like a consumer does ([Design: Replication](https://kafka.apache.org/43/design/design/#replication)).

![[kafka-replication-high-watermark.png]]

Offsets 6 and 7 sit only on the leader and one follower, so they aren't committed yet; when broker 3 fetches them the high watermark moves to 8 and consumers can read them.

- **ISR (in-sync replicas).** The replicas that are caught up with the leader. A follower that hasn't caught up for `replica.lag.time.max.ms` (30 s) is dropped from the ISR.
- **Committed = in the whole ISR.** A message counts as committed only when every in-sync replica has it, and consumers see only committed messages (the **high watermark**), so a leader failover can't take back what they read.
- **Producer `acks`.** `0` = don't wait, `1` = leader wrote it, `all` = the full ISR has it. With `acks=all` plus `min.insync.replicas=2` (default 1), a write either lands on at least 2 brokers or the producer gets an error ([Broker configs: min.insync.replicas](https://kafka.apache.org/43/configuration/broker-configs/#brokerconfigs_min.insync.replicas)).
- **Leader dies.** The controller promotes a replica from the ISR. If no in-sync replica is left, `unclean.leader.election.enable` (default `false`) decides: stay unavailable, or promote an out-of-sync replica and lose data.
- **Rule of thumb.** With f+1 replicas Kafka tolerates f failures without losing committed messages.

## Layer 3: Control plane — the controllers

Controllers never touch your messages; they own the cluster's **metadata log**: which brokers are alive, which topics and partitions exist, each partition's leader and ISR ([KRaft vs ZooKeeper](https://kafka.apache.org/43/getting-started/zk2kraft/)).

- A small quorum (usually 3) agrees on that log with the Raft protocol; one controller is active at a time and another takes over if it fails.
- Brokers register with the controller, heartbeat every `broker.heartbeat.interval.ms` (2 s) and pull metadata changes from it. Miss heartbeats for `broker.session.timeout.ms` (9 s) and the broker is fenced; its partitions get new leaders ([Broker configs](https://kafka.apache.org/43/configuration/broker-configs/#brokerconfigs_broker.session.timeout.ms)).
- One process can be broker, controller or both (`process.roles`). Small setups combine them; production usually separates them.

## Layer 4: Producer path — the mirror of the consumer

The producer sends each batch straight to the partition's leader, with no routing tier in between ([Design: The Producer](https://kafka.apache.org/43/design/design/#the-producer)).

![[kafka-producer-path.png]]

The same key always hashes to the same partition, which is why ordering in Kafka is per key, not per topic.

1. **Bootstrap and metadata**, exactly like the consumer: one broker from `bootstrap.servers` returns which broker leads each partition.
2. **Pick a partition.** With a key, a hash of the key picks the partition, so one key always lands in one partition: that's where per-key ordering comes from. Without a key, messages are spread across partitions.
3. **Batch.** The producer buffers messages per partition up to a size or time bound (`batch.size`, `linger.ms`), compresses the batch, and sends it in one request.
4. **Wait for `acks`** (Layer 2), then retry on failure.
5. **Idempotence and transactions.** The broker gives each producer an ID and drops duplicates by sequence number, so retries don't create duplicates. With a `transactional.id`, writes to several partitions and the consumer's offset commit succeed or fail together: Kafka's exactly-once ([Design: Message Delivery Semantics](https://kafka.apache.org/43/design/design/#message-delivery-semantics)).

## Layer 5: Coordinators — jobs on ordinary brokers

Two roles run on regular brokers, each backed by a compacted internal topic:

| Coordinator | Handles | Internal topic |
| --- | --- | --- |
| Group coordinator | Group membership, rebalances, committed offsets | `__consumer_offsets` |
| Transaction coordinator | Open transactions, commit/abort markers | `__transaction_state` |

Each group id (or transactional id) maps to one partition of its internal topic, and the broker leading that partition is its coordinator, so the load spreads across the cluster. Both topics have 50 partitions and replication factor 3 by default ([Broker configs: offsets.topic.*](https://kafka.apache.org/43/configuration/broker-configs/#brokerconfigs_offsets.topic.num.partitions)).

## Layer 6: Around the cluster

- **Kafka Connect** (part of Apache Kafka) moves data in and out without code, using source and sink connectors for databases, S3, Elasticsearch and more.
- **Kafka Streams** (part of Apache Kafka) is a Java library for stream processing inside your own app: joins, aggregations, windows, with exactly-once support.
- **Schema Registry** is a Confluent add-on, not Apache Kafka: it stores Avro, Protobuf or JSON schemas so producers and consumers agree on message formats.

## Summary

| Part | Owns | Talks to |
| --- | --- | --- |
| Broker | Partition logs, serves produce and fetch | Producers, consumers, other brokers, controllers |
| Leader replica | All writes and reads for one partition | Producers, consumers, its followers |
| Follower replica | A copy of the log, ready to take over | The leader |
| Controller quorum | Metadata log, leader elections | Brokers |
| Group coordinator | Consumer groups and offsets | Consumers |
| Transaction coordinator | Exactly-once transactions | Transactional producers |

**Interview answer:** Kafka is a cluster of brokers storing partitioned, replicated, append-only logs. Producers write to each partition's leader, followers replicate it, and consumers pull from it. A KRaft controller quorum owns the cluster metadata and elects leaders when a broker fails. Group and transaction coordinators, running on ordinary brokers, handle consumer groups and exactly-once delivery.

## Sources

Apache Kafka 4.3 documentation:

- [Design](https://kafka.apache.org/43/design/design/): persistence, efficiency, producer, replication, delivery semantics, log compaction
- [Broker Configs](https://kafka.apache.org/43/configuration/broker-configs/)
- [Protocol](https://kafka.apache.org/43/design/protocol/)
- [KRaft vs ZooKeeper](https://kafka.apache.org/43/getting-started/zk2kraft/)
