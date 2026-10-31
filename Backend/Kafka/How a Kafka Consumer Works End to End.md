# How a Kafka Consumer Works: End to End

Oct 30, 2026 · @dhruv

Every fact below is checked against the official Apache Kafka 4.3 documentation, linked inline and in Sources. Config defaults are from the Apache Java client reference. Confluent.Kafka (.NET) runs on librdkafka, which keeps its own config reference, so verify defaults there for production.

## Core idea: Kafka is a pull model

The consumer is never notified. Kafka's design has producers push data to the broker and consumers pull it from the broker ([Design: Push vs. pull](https://kafka.apache.org/43/design/design/#push-vs-pull)).

- **Fetch by offset.** The consumer sends fetch requests to the broker leading each partition it reads. Each request names an offset, and the broker returns a chunk of log starting there ([Design: The Consumer](https://kafka.apache.org/43/design/design/#the-consumer)).
- **Position is one number.** Per partition, the consumer's position is a single integer: the offset of the next message to consume. Kafka checkpoints it periodically ([Design: Consumer Position](https://kafka.apache.org/43/design/design/#consumer-position)).
- **"New" means a higher offset.** Anything at or after the group's position is new for that group.
- **Long polling avoids busy-waiting.** Instead of the consumer asking "anything new?" in a tight loop and getting empty answers, the consumer sends one fetch request and the broker doesn't answer right away; it keeps that single request waiting (open). It replies as soon as data arrives, or when a timeout passes. The consumer gets new events almost instantly without spamming the broker ([Design: Push vs. pull](https://kafka.apache.org/43/design/design/#push-vs-pull)).

Why Kafka chose pull, per the design docs:

- In a push system the broker sets the rate, so a slow consumer gets overwhelmed. With pull, a consumer just falls behind and catches up later.
- Pull allows aggressive batching: the consumer takes all available messages after its position, up to a configurable maximum.
- Messages are retained, not deleted on read, so a consumer can rewind to an old offset and re-consume, for example after fixing a bug.

## Step 0: How the consumer knows the bootstrap brokers

It doesn't discover them on its own. You pass them in the `bootstrap.servers` config.

> **Dhruv asked:** If I mention only one broker in the config, will it still find the other brokers that exist in that cluster?

> **Claude answered:** Yes, naming even one broker is enough. The client connects to it, asks for cluster metadata, and that one broker replies with the list of all other brokers. From then on the client talks to every broker directly. So the list is used only for the first connection. Order doesn't matter, it need not include every broker, and listing 2–3 is recommended so startup still works if one is down ([Consumer configs: bootstrap.servers](https://kafka.apache.org/43/configuration/consumer-configs/#consumerconfigs_bootstrap.servers)).

```csharp
var config = new ConsumerConfig {
    BootstrapServers = "broker1:9092,broker2:9092",
    GroupId = "order-service"
};
```

The protocol guide describes the client's routine: cycle through the bootstrap URLs until one connects, then fetch cluster metadata. Best practice is two or three hosts, or a load balancer in front of them.

**Dhruv asked:** How does a load balancer help in connecting with a broker?

**Claude answered:** It helps only with that first connection. You put one stable address (e.g. kafka.mycompany.com:9092) in bootstrap.servers, and the load balancer forwards it to any healthy broker, which returns the metadata. After that the client connects to each broker directly, using the addresses from advertised.listeners, not through the load balancer. The benefit is that brokers can be added, removed or replaced without changing the client config; the catch is that every broker must still be reachable directly by clients ([Protocol: Partitioning and bootstrapping](https://kafka.apache.org/43/design/protocol/#partitioning-and-bootstrapping)).

**Dhruv asked:** So we put the load balancer's host:port in the config, it connects us to a healthy broker, we get the whole cluster's metadata, and if brokers change we don't have to change the config?

**Claude answered:** Yes, that's right, with one correction. You still specify an address, just the load balancer's instead of a broker's. And the load balancer is only the front door: after the metadata arrives, the client talks to each broker directly at its advertised address, so traffic doesn't flow through the load balancer and every broker must be reachable from the client.

Where the value comes from in a .NET service (common deployment practice, not from the Kafka docs):

- **appsettings.json**, e.g. `"Kafka": { "BootstrapServers": "broker1:9092,broker2:9092" }`, overridden per environment.
- **Environment variables**, e.g. `Kafka__BootstrapServers=...` in Docker or docker-compose.
- **Kubernetes Service DNS**, e.g. `kafka.kafka-ns.svc.cluster.local:9092`.
- **Managed Kafka endpoint** from Confluent Cloud, Azure Event Hubs (Kafka endpoint) or AWS MSK.
- **Config or secrets store** such as Azure App Configuration or Key Vault.

**Losing every broker.** If all brokers the client knows become unreachable, `metadata.recovery.strategy` (default `rebootstrap`) makes the Java client repeat the bootstrap using `bootstrap.servers` ([Consumer configs](https://kafka.apache.org/43/configuration/consumer-configs/#consumerconfigs_metadata.recovery.strategy)).

**Dhruv asked:** Here, by "Java client" do you mean the consumer server?

**Claude answered:** No. "Java client" means Apache Kafka's official client library for Java (the `KafkaConsumer` class), the code that runs inside your consumer application and talks to the brokers. It isn't a server. Your .NET service uses a different client library, Confluent.Kafka (built on librdkafka), which does the same job; its setting names and defaults can differ, so check its docs for this one.

**How one broker knows the rest.** Every broker can answer a metadata request describing topics, partitions, partition leaders, and each broker's host and port ([Protocol](https://kafka.apache.org/43/design/protocol/#partitioning-and-bootstrapping)). In KRaft mode, controllers manage metadata and brokers serve data requests; ZooKeeper-mode settings are removed ([KRaft vs ZooKeeper](https://kafka.apache.org/43/getting-started/zk2kraft/)).

> **Dhruv asked:** What are the advantages and disadvantages of KRaft and ZooKeeper mode, and why should I use one?
>
> **Claude answered:** Use KRaft. Kafka 4.x runs only in KRaft mode; ZooKeeper mode exists only in 3.x and earlier, so it isn't really a choice for new work.
>
> |  | KRaft | ZooKeeper (legacy) |
> | --- | --- | --- |
> | Where metadata lives | Inside Kafka, in a metadata log managed by a quorum of controllers using the Raft protocol | In a separate ZooKeeper ensemble |
> | Systems to run | One (Kafka only) | Two (Kafka + ZooKeeper), each to deploy, secure and monitor |
> | Failover and shutdown | Quorum controller handles leader transfers and metadata updates, which the docs describe as more reliable and efficient | Controller relied on ZooKeeper, with extra retry settings for controlled shutdown |
> | Support | Current and only mode in 4.x | Removed; available only on old 3.x clusters |
> | Gotchas | Some ZooKeeper-era dynamic configs (e.g. `advertised.listeners`) now need a restart; policy plugins run on controllers; config values limited in size | Long-proven, but no future: new features target KRaft only |
>
> When ZooKeeper still matters: only if you're maintaining an existing 3.x cluster, and then the goal is to migrate to KRaft ([KRaft vs ZooKeeper](https://kafka.apache.org/43/getting-started/zk2kraft/)).

**Gotcha: advertised listeners.** The addresses clients receive come from each broker's `advertised.listeners`, which exists for cases such as cloud setups where the bind address isn't what clients should use. The addresses are published to and managed by the controller; if unset, `listeners` is used, and 0.0.0.0 cannot be advertised ([Broker configs: advertised.listeners](https://kafka.apache.org/43/configuration/broker-configs/#brokerconfigs_advertised.listeners)). In KRaft it is no longer a dynamic config, so changing it needs a restart ([KRaft vs ZooKeeper](https://kafka.apache.org/43/getting-started/zk2kraft/)). Classic Docker failure: you bootstrap via `localhost:9092`, metadata returns `kafka-container:9092`, your machine can't resolve it, and fetches fail.

## Phase 1: Startup and cluster metadata

The running example: a .NET service consuming topic `orders` (3 partitions) in consumer group `order-service`.

**1. The service creates the consumer.**

```csharp
var config = new ConsumerConfig {
    BootstrapServers = "broker1:9092,broker2:9092",
    GroupId = "order-service",
    AutoOffsetReset = AutoOffsetReset.Earliest,
    EnableAutoCommit = false
};
using var consumer = new ConsumerBuilder<string, string>(config).Build();
consumer.Subscribe("orders");
```

`Subscribe` only records what you want. The real network work starts on the first `Consume()` call.

**2. Bootstrap and metadata.** The consumer connects to any one bootstrap broker and sends a Metadata request: which brokers exist, which partitions `orders` has, and which broker leads each partition. It gets back something like:

| Partition | Leader |
| --- | --- |
| orders-0 | broker1 |
| orders-1 | broker2 |
| orders-2 | broker3 |

The bootstrap list is only an entry point. The client caches this metadata and refreshes it when it hits an error: a socket error, or a broker reporting it no longer hosts the partition ([Protocol](https://kafka.apache.org/43/design/protocol/#partitioning-and-bootstrapping)). The Java client also forces a refresh every `metadata.max.age.ms` (default 5 minutes) to discover new brokers or partitions ([Consumer configs](https://kafka.apache.org/43/configuration/consumer-configs/#consumerconfigs_metadata.max.age.ms)).

## Phase 2: Joining the consumer group

Kafka has two group protocols, chosen on the client by `group.protocol`. The default is `classic`. The newer `consumer` protocol (KIP-848) has been production-ready since Kafka 4.0, and Kafka 5.0 is expected to make it the default ([Consumer Rebalance Protocol](https://kafka.apache.org/43/operations/consumer-rebalance-protocol/), [Consumer configs: group.protocol](https://kafka.apache.org/43/configuration/consumer-configs/#consumerconfigs_group.protocol)).

> **Dhruv asked:** Why do we need a group coordinator?
>
> **Claude answered:** Because consumers in a group don't talk to each other, yet they must agree on who reads which partition. The coordinator is the one broker acting as the group's referee.
>
> **Dhruv asked:** So that broker will not have partitions?
>
> **Claude answered:** No, it still does. The coordinator is an ordinary broker that keeps storing partitions and serving producers and consumers; coordinating is just an extra job on top. Each group maps to one partition of the internal offsets topic, and the broker leading that partition becomes that group's coordinator, so different groups usually land on different brokers and the work is spread out. It:
>
> - **Tracks membership:** knows which consumers are in the group right now.
> - **Dhruv asked:** How does the cluster work: is there a controller, others handle data, and on the consumer side one broker is the group coordinator?
> - **Claude answered:** Yes, that's the right picture, as three roles.
> - (1) **Controllers** (a small KRaft quorum, often 3) manage cluster metadata: which brokers are alive, which topics and partitions exist, and which broker leads each partition, electing a new leader when one fails.
> - (2) **Brokers** store partitions and serve all produce and fetch requests; each partition has one leader broker plus follower replicas.
> - (3) **Group coordinator** isn't a separate machine but an extra job a broker takes on for specific groups, while still storing partitions; different groups usually get different brokers. A process can be a broker, a controller, or both, via `process.roles` ([Broker configs](https://kafka.apache.org/43/configuration/broker-configs/#brokerconfigs_process.roles)). So your consumer talks to partition leaders to fetch data and to its group's coordinator for membership and offsets, while controllers work behind the scenes keeping metadata correct.
> - **Dhruv asked:** So brokers don't have partitions of their own; each holds a set of partitions and maintains copies?
> - **Claude answered:** Close. Each broker holds some partitions, and every partition is copied to N brokers, where N is the topic's replication factor. For each partition, exactly one copy (Dhruv asked: here does one copy mean a broker? Claude answered: not quite. A copy, called a replica, is the partition's data, its log files, stored on one broker. Each copy of a partition sits on a different broker, so a partition with 3 copies lives on 3 brokers. But a broker isn't a copy: one broker holds replicas of many different partitions at once) is the **leader** (all writes go to it, and consumers read from it by default) and the rest are **followers** that pull from the leader to stay in sync. So one broker is leader for some partitions and follower for others; not every broker has every partition. Example: 3 brokers, `orders` with 3 partitions and replication factor 2 → broker1 leads orders-0 and follows orders-2, broker2 leads orders-1 and follows orders-0, broker3 leads orders-2 and follows orders-1. If a leader's broker dies, the controller promotes an in-sync follower ([Design: Replication](https://kafka.apache.org/43/design/design/#replication)).
> - **Detects failures:** a consumer that stops heartbeating (Dhruv asked: what does heartbeating mean? Claude answered: a heartbeat is a tiny "I'm still alive" message the consumer sends the coordinator every few seconds, about every 3 s by default, from a background thread. It carries no data; its only purpose is to prove the consumer is alive. If none arrive within the session timeout, 45 s by default, the coordinator assumes the consumer crashed and gives its partitions to others) is removed ([Consumer configs: session.timeout.ms](https://kafka.apache.org/43/configuration/consumer-configs/#consumerconfigs_session.timeout.ms)).
> - **Runs rebalances:** when members join or leave, it redistributes partitions so each partition has exactly one reader in the group.
> - **Dhruv asked:** What if a consumer group has more members than partitions? **Claude answered:** The extra members sit idle: they stay in the group and heartbeat but get no partitions, because a partition can't be split between two consumers in the same group. Example: 3 partitions, 5 consumers → 3 each get one partition and 2 get nothing. They aren't wasted, though: they act as hot standbys and take over immediately if an active consumer dies. Takeaway: the partition count is the ceiling on parallelism within one group, so to scale consumption, add partitions, not just instances. (Share groups, a newer group type, are the exception: there, more consumers than partitions is allowed ([Design: The Share Consumer](https://kafka.apache.org/43/design/design/#the-share-consumer)).)
> - **Stores committed offsets:** handles OffsetCommit and OffsetFetch, so a restarted or replacement consumer knows where to resume.
>
> Without it, two instances could read the same partition (duplicates) or a partition could be left unread, and progress would be lost on restart.

|  | Classic (default) | Consumer (KIP-848) |
| --- | --- | --- |
| Enable with | `group.protocol=classic` | `group.protocol=consumer` |
| Group APIs | FindCoordinator, JoinGroup, SyncGroup, Heartbeat | ConsumerGroupHeartbeat |
| Who assigns partitions | Group leader (a consumer) runs a client-side assignor | The coordinator, with a server-side assignor |
| Assignors | `partition.assignment.strategy`, default [Range, CooperativeSticky], so Range is used | `group.consumer.assignors`, default `uniform,range` |
| Heartbeat / session timeout | Client: 3 s / 45 s | Broker: 5 s / 45 s |
| Rebalance style | Whole-group, unless cooperative-sticky | Fully incremental, no global synchronization barrier |

Steps under the classic protocol:

**3. Find the group coordinator.** One broker manages the `order-service` group. The consumer sends FindCoordinator to learn which one.

**4. JoinGroup.** The consumer registers as a member subscribed to `orders`. For a new group, the coordinator waits `group.initial.rebalance.delay.ms` (default 3 s) for more members before the first rebalance ([Broker configs](https://kafka.apache.org/43/configuration/broker-configs/#brokerconfigs_group.initial.rebalance.delay.ms)). One member is chosen as **group leader**.

**5. Partition assignment (SyncGroup).** The group leader computes the assignment and the coordinator hands each member its share. With 2 instances running:

| Instance | Partitions |
| --- | --- |
| A | orders-0, orders-1 |
| B | orders-2 |

Key rule: each partition is consumed by exactly one consumer within a group at any time ([Design: Consumer Position](https://kafka.apache.org/43/design/design/#consumer-position)). With 4 instances on 3 partitions, one instance sits idle. Under the consumer protocol, steps 3 to 5 collapse into heartbeats that carry the server's assignment. Check your client library's docs before setting `group.protocol=consumer` from .NET.

## Phase 3: Where to start reading

**6. OffsetFetch.** For each assigned partition, the consumer asks the coordinator for the group's **last committed offset**. Kafka stores the consumer's position as a message in an internal topic, `__consumer_offsets` ([Design: Message Delivery Semantics](https://kafka.apache.org/43/design/design/#message-delivery-semantics)).

- **Offset found** (say orders-0 → 42): reading starts at 42.
- **No offset found**, or the offset no longer exists because the data was deleted: `auto.offset.reset` decides ([Consumer configs: auto.offset.reset](https://kafka.apache.org/43/configuration/consumer-configs/#consumerconfigs_auto.offset.reset)).
  - `latest` (the **default**) starts at the newest offset, so only events produced from now on.
  - `earliest` starts from the oldest event still in the log.
  - `by_duration:<ISO-8601>` starts that far back from now, e.g. `by_duration:PT1H`.
  - `none` throws an exception to the consumer.

The example config in Phase 1 sets `Earliest` explicitly, overriding the default. The docs warn that with `latest`, adding partitions can lose messages produced to new partitions before consumers reset.

Committed offsets don't live forever. For subscribed groups, they expire `offsets.retention.minutes` (default 10,080, i.e. 7 days) after the group becomes empty ([Broker configs: offsets.retention.minutes](https://kafka.apache.org/43/configuration/broker-configs/#brokerconfigs_offsets.retention.minutes)). After that, `auto.offset.reset` applies again.

## Phase 4: The fetch loop

**7. Fetch from partition leaders.** Fetch requests must go to the broker currently leading that partition. A request sent to the wrong broker gets a NOT_LEADER_OR_FOLLOWER error, and the client refreshes metadata ([Protocol](https://kafka.apache.org/43/design/protocol/#partitioning-and-bootstrapping)).

**8. Long polling.** The broker answers once `fetch.min.bytes` of data is available (default 1 byte, so effectively as soon as anything exists). If not enough data arrives, it answers after `fetch.max.wait.ms` (default 500 ms) ([Consumer configs](https://kafka.apache.org/43/configuration/consumer-configs/#consumerconfigs_fetch.max.wait.ms)). So when the producer appends offset 42, the waiting fetch returns almost immediately.

**Client-side buffering.** The Java consumer caches records from each fetch and returns them incrementally from each `poll()`, up to `max.poll.records` (default 500) per call ([Consumer configs](https://kafka.apache.org/43/configuration/consumer-configs/#consumerconfigs_max.poll.records)). Confluent.Kafka (librdkafka) similarly prefetches into a local queue that `Consume()` reads from; that is a .NET client detail, not from the Kafka docs.

**9. Your code processes the message.**

```csharp
while (!ct.IsCancellationRequested)
{
    var r = consumer.Consume(ct);   // blocks until a message arrives
    ProcessOrder(r.Message.Value);  // business logic
    consumer.Commit(r);             // step 10
}
```

**10. Commit the offset.** After processing, the consumer sends OffsetCommit to the coordinator: orders-0 is processed up to 42, next is 43.

- **Auto commit** is on by default (`enable.auto.commit=true`), committing every `auto.commit.interval.ms` (default 5 s) in the background ([Consumer configs](https://kafka.apache.org/43/configuration/consumer-configs/#consumerconfigs_enable.auto.commit)).
- **Process, then commit** gives **at-least-once**: a crash between processing and commit means those messages are processed again. Idempotent processing makes duplicates harmless ([Design: Message Delivery Semantics](https://kafka.apache.org/43/design/design/#message-delivery-semantics)).
- **Commit, then process** gives **at-most-once**: a crash after committing loses the unprocessed messages.
- Kafka guarantees at-least-once by default. Exactly-once for Kafka-to-Kafka pipelines uses a transactional producer plus `isolation.level=read_committed` ([Design: Using Transactions](https://kafka.apache.org/43/design/design/#using-transactions)).

The loop repeats: fetch → process → commit.

## Phase 5: Staying alive in the group

**11. Heartbeats.** Under the classic protocol, the consumer sends heartbeats every `heartbeat.interval.ms` (default 3 s, typically no more than 1/3 of the session timeout). If the broker receives none within `session.timeout.ms` (default 45 s), it removes the consumer and starts a rebalance. Under the consumer protocol, both values are broker-controlled instead: `group.consumer.heartbeat.interval.ms` (5 s) and `group.consumer.session.timeout.ms` (45 s) ([Consumer configs](https://kafka.apache.org/43/configuration/consumer-configs/#consumerconfigs_session.timeout.ms), [Broker configs](https://kafka.apache.org/43/configuration/broker-configs/#brokerconfigs_group.consumer.session.timeout.ms)).

**12. Poll-interval check.** If your code doesn't call `poll()` / `Consume()` within `max.poll.interval.ms` (default 5 min), for example because processing is stuck, the consumer is considered failed and the group rebalances. A static member (`group.instance.id` set) instead stops heartbeating, and its partitions move only after the session timeout ([Consumer configs: max.poll.interval.ms](https://kafka.apache.org/43/configuration/consumer-configs/#consumerconfigs_max.poll.interval.ms)).

## Phase 6: Rebalancing

**13. A rebalance reassigns partitions when the group changes.** Triggers:

- a new instance joins (scaling up)
- an instance misses heartbeats or exceeds the poll interval
- an instance leaves cleanly
- the subscribed topics' partitions change

After a rebalance, each consumer resumes from the committed offset of every partition it now owns. That is why committing matters: if instance A processed orders-0 up to offset 50 but last committed 45, and the partition moves to B, then B re-processes 45 to 50.

Ways to reduce rebalance pain, per the docs:

- **Cooperative-sticky** (classic) keeps as many existing assignments as possible and rebalances cooperatively ([Consumer configs: partition.assignment.strategy](https://kafka.apache.org/43/configuration/consumer-configs/#consumerconfigs_partition.assignment.strategy)).
- The **consumer protocol** is fully incremental, with no global synchronization barrier ([Consumer Rebalance Protocol](https://kafka.apache.org/43/operations/consumer-rebalance-protocol/)).
- **Static membership** (`group.instance.id`) gives each instance a persistent id, so restarts and deploys don't trigger a rebalance ([Design: Static Membership](https://kafka.apache.org/43/design/design/#static-membership)).

## Phase 7: Shutdown

**14. Clean exit.**

```csharp
consumer.Close();
```

`Close()` commits final offsets when auto commit is on and leaves the group (LeaveGroup is one of the protocol's APIs, [Protocol: Api Keys](https://kafka.apache.org/43/design/protocol/#api-keys)). The coordinator can then rebalance right away instead of waiting for the session timeout. A static member's partitions are not reassigned until its session timeout expires. In a .NET Worker Service, call `Close()` in `StopAsync` or after the loop exits on cancellation.

## Summary

| # | Step | Talks to |
| --- | --- | --- |
| 0 | Read `bootstrap.servers` from config | your config |
| 1–2 | Bootstrap, fetch and cache cluster metadata | any broker |
| 3 | Find group coordinator | any broker |
| 4–5 | Join group, get partitions (classic: JoinGroup + SyncGroup; consumer protocol: heartbeats) | coordinator |
| 6 | Fetch committed offsets, else `auto.offset.reset` (default `latest`) | coordinator |
| 7–8 | Long-poll fetch from current offset | partition leaders |
| 9–10 | Process, then commit (auto commit is on by default) | your code, coordinator |
| 11–12 | Heartbeats and poll-interval checks | coordinator |
| 13 | Rebalance on membership change | coordinator |
| 14 | Close and leave group | coordinator |

**Interview answer:** Kafka consumers pull; nothing notifies them. On startup the consumer connects to a broker from `bootstrap.servers` and caches cluster metadata. It joins its consumer group through the group coordinator, gets partitions assigned, and looks up the group's committed offsets, falling back to `auto.offset.reset`. It then long-polls each partition leader from that offset; new events are simply offsets beyond its position. It commits offsets after processing for at-least-once delivery, and heartbeats keep it in the group. When membership changes, a rebalance moves partitions and consumers resume from committed offsets.

## Sources

Apache Kafka 4.3 documentation:

- [Design](https://kafka.apache.org/43/design/design/): push vs pull, consumer position, delivery semantics, static membership, transactions
- [Protocol](https://kafka.apache.org/43/design/protocol/): bootstrapping, metadata, leaders, API keys
- [Consumer and Share Consumer Configs](https://kafka.apache.org/43/configuration/consumer-configs/)
- [Broker Configs](https://kafka.apache.org/43/configuration/broker-configs/)
- [Consumer Rebalance Protocol](https://kafka.apache.org/43/operations/consumer-rebalance-protocol/)
- [KRaft vs ZooKeeper](https://kafka.apache.org/43/getting-started/zk2kraft/)
