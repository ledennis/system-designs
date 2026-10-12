# Database Selection Guide (System Design)

Use this to narrow down a database choice, then justify it from the **access pattern**, **scale**, **consistency needs**, and **failure behavior**. Always acknowledge the trade-off: most workloads can be served by more than one option.

## Quick decision questions
1. What is the **access pattern**? (lookup by key, range scans, joins, full-text search, graph traversal, time series)
2. How much **data** and what **read/write QPS**? Does it fit one machine, or must it shard?
3. What **consistency** is required? (strong/transactional vs eventual)
4. Is the **schema** fixed and relational, or flexible and evolving?
5. What are the **latency** and **availability** targets? Is multi-region required?

## Comparison table

| Type | Examples | Strengths | Weaknesses | Best fit |
|---|---|---|---|---|
| **Relational (SQL)** | PostgreSQL, MySQL | ACID transactions, joins, strong consistency, mature tooling, flexible ad-hoc queries, enforced schema and constraints | Harder to scale writes horizontally (sharding is manual and complex), joins across shards are painful, vertical scaling limits | Structured data with relationships, transactions (payments, orders, inventory), moderate scale |
| **Key-value** | DynamoDB, Redis, Riak | Very fast lookups by key, simple model, horizontal scaling, predictable latency | Limited querying (key access only, or limited secondary indexes), no joins, data modeling must follow access patterns up front | URL shorteners, sessions, caching, user profiles, feature flags |
| **Wide-column** | Cassandra, HBase, Bigtable | Massive write throughput, linear horizontal scaling, tunable consistency, good multi-region replication, handles huge datasets | Query patterns must be designed up front, limited joins and ad-hoc queries, eventual consistency trade-offs, operational complexity (Cassandra) | Time-series-like data, messaging, activity feeds, logging, IoT at large scale |
| **Document** | MongoDB, Couchbase, Firestore | Flexible schema, nested data maps well to objects, good developer experience, secondary indexes, horizontal scaling | Weaker multi-document transactions (improving, but costlier), joins are limited, data duplication, inconsistent schemas if undisciplined | Content/catalog data, user-generated content, rapidly evolving schemas |
| **In-memory store** | Redis, Memcached | Sub-millisecond latency, great for hot data, rich structures in Redis (sorted sets, lists, pub/sub) | Memory is expensive, limited capacity, durability is weaker (Memcached has none; Redis persistence has trade-offs), cold-cache stampede risk | Caching, leaderboards, rate limiting, sessions |
| **Search engine** | Elasticsearch, OpenSearch, Solr | Full-text search, relevance ranking, aggregations, fast filtering across many fields | Not a primary source of truth, eventual consistency, expensive to run, reindexing and cluster tuning overhead | Search boxes, log analytics, autocomplete |
| **Graph** | Neo4j, Neptune | Natural for relationship traversal, multi-hop queries are fast, expressive query languages | Harder to scale horizontally, smaller ecosystem, overkill for simple data | Social networks, recommendations, fraud detection, knowledge graphs |
| **Time-series** | InfluxDB, TimescaleDB, Prometheus | Optimized for timestamped writes, compression, downsampling and retention policies, time-range queries | Poor fit for general-purpose workloads, high-cardinality data can be costly, limited joins | Metrics, monitoring, IoT sensor data, financial ticks |
| **Object / blob storage** | S3, GCS, Azure Blob | Extremely cheap and durable, effectively unlimited scale, great for large files | High latency vs databases, no querying (without extra tooling), not for small frequent updates | Images, video, backups, data lake files |
| **Distributed SQL / NewSQL** | Spanner, CockroachDB, YugabyteDB | SQL plus horizontal scaling, strong consistency across regions, ACID transactions at scale | Higher write latency (consensus across nodes/regions), operational cost, more complex than single-node SQL | Global transactional systems that outgrow a single SQL instance |
| **Columnar / analytics (OLAP)** | BigQuery, Redshift, ClickHouse, Snowflake | Fast aggregations across huge datasets, compression, great for reporting | Poor for row-level OLTP, slower single-row writes and updates, batch-oriented | Data warehouses, dashboards, BI, analytics |

## Consistency quick reference

| Style | Meaning | Typical examples |
|---|---|---|
| **Strong consistency** | Reads always see the latest committed write | Single-node SQL, Spanner, DynamoDB strongly consistent reads |
| **Eventual consistency** | Replicas converge over time, reads may be briefly stale | DynamoDB default reads, Cassandra (at low consistency levels), cross-region replication |
| **Tunable** | You choose per query (e.g., quorum reads/writes) | Cassandra, DynamoDB (per-read option) |

## CAP theorem

### The three properties
- **Consistency (C):** every read sees the most recent write (or returns an error). All nodes appear to hold the same data at the same time. This is *linearizability*, not the "C" in ACID.
- **Availability (A):** every request to a non-failing node receives a (non-error) response, even if it might not contain the latest data.
- **Partition tolerance (P):** the system keeps working even when the network drops or delays messages between nodes.

### The theorem
During a **network partition**, a distributed system must choose between **consistency** and **availability**. It can't guarantee both.

### Why "pick 2 of 3" is misleading
- Partitions **will** happen in any distributed system (network failures, region outages), so **P is not optional**.
- The real choice is: **when a partition happens, do I favor C or A?**
- When there is no partition, a system can offer both consistency and availability. CAP only describes behavior *during* a partition.
- It's a spectrum, not a switch. Many databases are tunable per query.

### CP vs AP

| | **CP (consistency over availability)** | **AP (availability over consistency)** |
|---|---|---|
| **During a partition** | Rejects or blocks requests it can't guarantee are correct (returns errors or times out) | Keeps serving requests, possibly with stale or divergent data |
| **After the partition heals** | Nothing to reconcile, since writes were refused | Must reconcile conflicts (last-write-wins, vector clocks, merge logic) |
| **Choose it when** | Wrong data is worse than no data: money, inventory, unique IDs, leader election, locks | Downtime is worse than slightly stale data: feeds, carts, caches, view counts, redirects |
| **Typical examples** | Spanner, etcd/ZooKeeper, HBase, MongoDB (default, majority reads/writes), single-leader SQL with synchronous replication | Cassandra, DynamoDB (default eventually consistent reads), Couchbase, Riak, DNS |

Note: placement is a simplification. Many systems are **tunable** (Cassandra quorum reads/writes, DynamoDB strongly consistent reads, MongoDB read/write concerns), so say "it depends on configuration."

### Tunable consistency (quorum math)
With `N` replicas, write quorum `W`, and read quorum `R`:
- **`R + W > N`** → reads overlap with the latest write (strong-ish consistency).
- **`R + W ≤ N`** → faster and more available, but reads may be stale.
- Example: `N = 3`, `W = 2`, `R = 2` → consistent reads, tolerates 1 node down. `W = 1`, `R = 1` → fastest, but eventual consistency.

### PACELC (the fuller picture)
CAP says nothing about normal operation. **PACELC** extends it:
- **If there's a Partition (P):** choose **A**vailability or **C**onsistency.
- **Else (E):** choose **L**atency or **C**onsistency.

Even without a partition, replicating synchronously for consistency costs latency. This explains why a system like DynamoDB (eventual reads) is fast and cheap, while Spanner (strong consistency) pays extra write latency for cross-region consensus.

| System | During partition | Normal operation |
|---|---|---|
| DynamoDB / Cassandra (defaults) | A | L |
| Spanner / CockroachDB | C | C |
| MongoDB (defaults) | C | C |
| Single-node SQL | n/a (not distributed) | C |

### How CAP applies to the URL shortener
- **Requirement:** latency and availability over strict consistency. A few seconds of staleness is fine for new, expired, or deleted links. This is an **AP** leaning, so DynamoDB with eventually consistent reads fits.
- **Exception: uniqueness.** A short code must never map to two URLs. Handle this with a **conditional write** (strong consistency on that one operation) or **disjoint ID ranges per region**, rather than making the entire system CP.
- **Region failure:** serve redirects from a healthy region using asynchronously replicated data (AP). The trade-off is that links created seconds before the failure might not have replicated yet.
- **Coordinator (ID block leasing):** this needs to be **CP** (consensus-backed like etcd/ZooKeeper) so it never re-issues the same ID range. Servers keep working from their leased blocks if the coordinator is briefly unavailable.

### Common interview mistakes to avoid
- Saying "we pick any 2 of 3" or treating CA as a real option for a distributed system.
- Labeling a whole system "CP" or "AP" instead of reasoning per operation (reads vs writes, redirects vs creation).
- Confusing CAP's "consistency" with ACID's "consistency."
- Forgetting to say *what happens to the user* during a partition (error, stale data, or queued write).
- Not mentioning conflict resolution when choosing AP.

### One-line answer to practice
> "Partitions are unavoidable, so during one I have to choose between consistency and availability. For redirects I choose availability and accept slightly stale reads, but for short-code uniqueness I enforce consistency with a conditional write and a replicated coordinator."

## Common pairings
- **SQL + Redis cache:** relational source of truth with a cache in front for hot reads.
- **DynamoDB/Cassandra + Redis:** key lookups at scale with a cache for hot keys.
- **Primary DB + Elasticsearch:** source of truth plus a search index fed asynchronously.
- **OLTP DB + queue + warehouse:** transactional system streaming events to an analytics store.
- **Database + S3:** metadata in the database, large files in object storage.

## Narrowing it down by scenario

| If the requirement is... | Lean toward... | Why |
|---|---|---|
| Lookup by a single key, huge scale, simple records | Key-value (DynamoDB) or wide-column | Horizontal scaling by key, predictable latency |
| Transactions across multiple rows/entities | Relational or distributed SQL | ACID guarantees, joins |
| Heavy writes, append-style data, multi-region | Cassandra / wide-column | High write throughput, tunable consistency |
| Flexible, nested, evolving schema | Document store | Schema flexibility |
| Search and filtering text | Elasticsearch/OpenSearch (alongside a primary DB) | Inverted indexes, ranking |
| Relationship traversal | Graph DB | Fast multi-hop queries |
| Metrics and time-range queries | Time-series DB | Compression, retention, time-based queries |
| Large files (images, videos) | Object storage + CDN | Cheap, durable, scalable |
| Reports across billions of rows | Columnar warehouse | Fast aggregations |

## How to justify a choice in an interview
1. **State the access pattern** ("lookup by short code, no joins").
2. **State the scale** ("~3 TB over 5 years, ~20K reads/sec at peak").
3. **State the consistency need** ("eventual consistency is acceptable on reads").
4. **Name the choice** and the features that matter (horizontal scaling, TTL, conditional writes).
5. **Acknowledge the alternative** ("a relational DB would also work at this size, but...").
6. **Cover failure and scaling** (replication, sharding, hot keys, cache behavior).

## Notes to fill in as I practice
- Systems where I've chosen each database and why:
- Trade-offs I got wrong or forgot to mention:
- Follow-up questions interviewers asked:
