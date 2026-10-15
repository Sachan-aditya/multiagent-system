# Part 3 of 13: Databases, Caching, and Storage (Sections 1 and 7)

> Level tags: `[SDE1]` `[SDE2]` `[Both]`; priority P0 (must know), P1 (should), P2 (nice to have).
> **Pricing and product-limit notes:** all dollar figures here are *illustrative teaching numbers*, not provider prices. Product names, tiers, and limits change often (especially MongoDB Atlas and Azure Cosmos DB offerings). Anything marked **verify** must be checked against the provider's current official documentation and pricing page before you rely on it. I have no live access to those pages in this conversation.
> Each topic follows the template: what, why, how, alternatives, when/when not, trade-offs, production problems, example, mistakes, Q&A, your turn. Answers to all "Your turn" questions are in the **Answer Key**.

---

## 1. Concept Map for This Part

```
                 +-----------------------------+
 App pods  --->  | Connection pool / proxy     |  (3.8)
                 +--------------+--------------+
                                |
        +-----------+-----------+------------+----------------+
        v           v                        v                v
   Cache (Redis)  Primary DB  --replication--> Read replicas   Queue / CDC stream
     (3.9)        (3.1-3.3)       (3.5)           (3.5)            (3.12)
                      |                                             |
                 Shards/partitions (3.6)                    Warehouse / lake (3.12)
                      |
                Backups, PITR (3.14)   Tiered object storage, retention (3.13)
```

Decisions interact: **shard key** drives hot partitions; **pool sizes** times **pod count** drives connection limits; **cache** changes consistency and failure modes; **pricing model** changes what "efficient" means; **replication mode** decides your RPO.

---

## 3.1 SQL vs NoSQL and How to Choose `[Both]` P0

**What.** *Relational (SQL)* databases store rows in tables with a schema, joins, and ACID transactions. *NoSQL* is an umbrella for document (MongoDB), key-value (Redis, DynamoDB), wide-column (Cassandra), graph (Neo4j), and search (Elasticsearch) stores, each optimized for a specific access pattern.

**Why.** Relational: flexible ad-hoc queries and strong integrity. NoSQL: horizontal scale, flexible schema, or very low latency for known access patterns.

**How it works (design stance).**
- **SQL:** model the *data*, then query it any way you like (normalize, join).
- **Most NoSQL:** model the *queries* first, then shape data to serve them (denormalize, embed, pre-compute).

**Decision tree.**

```
Need multi-row ACID transactions, complex joins, ad-hoc reporting?
  YES -> Relational (PostgreSQL/MySQL). Start here unless a requirement forces otherwise.
  NO  -> What is the dominant access pattern?
        - Lookup by key at sub-millisecond latency, ephemeral data, counters   -> Redis
        - Hierarchical documents read/written as a unit, evolving schema       -> Document DB (MongoDB)
        - Massive write volume, time-ordered, known query paths, multi-DC      -> Wide-column (Cassandra)
        - Full-text search, relevance ranking, faceting                        -> Search engine
        - Relationship traversal (friends-of-friends, fraud rings)             -> Graph DB
        - Large-scale analytics/aggregations                                   -> Columnar warehouse (OLAP)
```

**Alternatives and hybrid reality.** Most production systems are **polyglot**: PostgreSQL as source of truth + Redis for cache + a search index + a warehouse. PostgreSQL itself covers a lot (JSONB documents, full-text search, extensions such as vector search, partitioning), which is why "start with PostgreSQL" is a defensible default.

**When not to use NoSQL:** when you don't yet know your access patterns, need cross-entity transactions, or the team can't operate it. **When not to use SQL:** when the access pattern is a single-key lookup at massive scale, or write throughput exceeds what one primary can handle and you can't shard cleanly.

**Trade-offs.** SQL: flexibility and integrity vs. harder horizontal write scaling. NoSQL: scale and latency vs. weaker queries, manual integrity, and migration pain when access patterns change.

**Production problems.**
- *Wrong-database regret:* a document store chosen "for flexibility" later needs joins and transactions. Symptom: application-side joins, N+1 reads, inconsistent data. Fix: introduce relational store for relational parts; use CDC to sync.
- *Schemaless chaos:* documents drift in shape over time; code handles 5 versions of the same record. Fix: schema validation, versioned documents, migration jobs.
- *Premature polyglot:* five databases for a small team multiplies on-call burden. Prevention: add a datastore only when a measured requirement can't be met by the current one.

**Example.** E-commerce: orders/payments in PostgreSQL (transactions), product search in Elasticsearch/OpenSearch, session/cart cache in Redis, clickstream to a warehouse.

**Common mistakes.** "NoSQL scales, SQL doesn't" (single-node PostgreSQL handles far more than most startups need). Choosing by popularity. Ignoring operational skill.

**Interview Q&A.**
- *SQL vs NoSQL?* Start from access patterns, consistency needs, and scale; SQL for relational/transactional, NoSQL when a specific pattern (key lookup, huge writes, documents-as-a-unit) dominates.
- *Can PostgreSQL replace MongoDB?* Often yes for JSON documents (JSONB); not when you need MongoDB-style native horizontal sharding with simpler ops, depending on scale.

**Your turn (Q1).** A startup's product is a multi-tenant SaaS with invoices, subscriptions, and user-uploaded forms of varying structure. Pick a datastore setup and justify each choice.

---

## 3.2 PostgreSQL vs MongoDB vs Redis vs Cassandra `[Both]` P0

**Comparison matrix** *(general characteristics; verify details for your version)*

| Dimension | PostgreSQL | MongoDB | Redis | Cassandra |
|---|---|---|---|---|
| Model | Relational + JSONB | Document (BSON) | In-memory data structures (key-value) | Wide-column |
| Consistency | Strong on primary; replicas async by default | Tunable via read/write concern; strong on primary with `majority` | Strong on single node; replication async | Tunable per query (ONE, QUORUM, ALL) |
| Transactions | Full ACID, multi-row | Single-document atomic; multi-document transactions available (higher cost) | Single command atomic; MULTI/Lua scripts | Lightweight transactions (Paxos) for narrow cases; no general multi-row ACID |
| Scaling writes | Vertical; sharding via extensions/partitioning | Native sharding | Cluster (16,384 hash slots) | Native, linear scale-out (leaderless) |
| Query power | Rich SQL, joins, window functions | Aggregation pipeline, limited joins ($lookup) | Key access + data-structure commands | Query-by-partition-key only; no joins |
| Storage engine | Heap + B-tree, MVCC | WiredTiger (B-tree based) | Memory (+ RDB/AOF persistence) | LSM-tree |
| Latency profile | ms | ms | sub-ms | ms, high write throughput |
| Best for | Source of truth, relational data, reporting | Aggregate-oriented documents, rapid iteration, horizontal scale | Cache, sessions, rate limits, leaderboards, queues | Time series, IoT, write-heavy, multi-DC availability |
| Watch out for | Single-primary write ceiling, vacuum/bloat | Poor shard key, large documents (16 MB limit), unindexed queries | Memory cost, eviction, persistence trade-offs | Tombstones, wide partitions, data modeling rigidity, repair ops |

**How each works (key internals).**
- **PostgreSQL:** process-per-connection, MVCC with VACUUM, WAL for durability and replication, planner uses statistics.
- **MongoDB:** replica set (primary + secondaries, automatic election); sharded cluster = `mongos` routers + config servers + shards; WiredTiger B-tree storage; write concern (`w: "majority"`) and read concern decide durability/visibility.
- **Redis:** mostly single-threaded command execution on in-memory structures (strings, hashes, lists, sets, sorted sets, streams); persistence via RDB snapshots and/or AOF; replication is asynchronous.
- **Cassandra:** partition key hashed onto a ring; each partition replicated to N nodes; writes go to commit log + memtable, flushed to SSTables; compaction merges; deletes are *tombstones*.

**When not to use.** Redis as the only store of critical data (memory-bound, durability trade-offs). Cassandra when you need ad-hoc queries or transactions. MongoDB for heavily relational data with many cross-collection joins. PostgreSQL as an unbounded write-scaling firehose without partitioning/sharding.

**Production problems.**
- *Cassandra tombstone pile-up:* deletes create tombstones that reads must scan until compaction; symptom: read timeouts on partitions with heavy deletes/TTL. Fix: model to avoid deleting inside hot partitions, tune gc_grace and compaction, time-window compaction for TTL data.
- *Wide/hot partitions (Cassandra/Mongo/Cosmos):* one key gets disproportionate load. Fix: composite or bucketed keys.
- *MongoDB unindexed query / COLLSCAN:* primary CPU spikes. Fix: compound index per ESR rule, check `explain`.
- *PostgreSQL bloat/vacuum:* see Part 1 (3.10).
- *Redis memory exhaustion:* eviction of useful keys or `OOM command not allowed`. Fix: eviction policy, TTLs, key-size review, memory alerts.

**Example.** Chat app: user profiles and billing in PostgreSQL; message history in Cassandra (partition by conversation, cluster by time); presence and typing indicators in Redis.

**Common mistakes.** Treating Redis as a durable database by default. Picking Cassandra for flexibility (it's the opposite: you design tables per query). Ignoring operational burden of self-hosting any of them.

**Interview Q&A.**
- *PostgreSQL vs MongoDB for a growing SaaS?* Default PostgreSQL for relational/transactional data; MongoDB if data is naturally document-shaped, schema changes constantly, and you expect horizontal write scale; evaluate team skills and operational model. (Full case study in Part 11.)
- *Why is Cassandra fast at writes?* Sequential appends to commit log + memtable (LSM), no read-before-write for plain inserts.

**Your turn (Q2).** Explain why "query-first" modeling is required in Cassandra and design a table for "latest 50 messages in a conversation".

---

## 3.3 Deployment Models: Serverless vs Dedicated, Shared vs Isolated, Managed vs Self-Hosted `[SDE2]` P1

**What.**
- **Serverless database:** capacity abstracted away; scales (sometimes to zero) automatically; billed on consumption (requests, compute units, storage).
- **Dedicated/provisioned:** you choose instance size or throughput; billed per hour/month whether used or not.
- **Shared vs isolated infrastructure:** shared = multi-tenant pool of resources (cheaper, noisy-neighbor risk, tighter limits); isolated = reserved resources, private networking, predictable performance.
- **Managed vs self-hosted:** vendor runs patching, backups, failover; self-hosted gives control and potentially lower unit cost but heavy operational burden.

**Why.** Different workloads have different shapes: spiky and unpredictable, steady and heavy, or compliance-sensitive.

**How the named models work (verify current details in official docs).**
- **MongoDB Atlas:** *Dedicated clusters* (M10 and up; choose tier, storage, replica set or sharded; predictable performance; features like private networking and fine-grained backup options). *Serverless instances:* pay per operations and storage, scale automatically, with feature/limit restrictions. MongoDB has been restructuring its low-end offerings (shared tiers and Serverless toward "Flex" clusters), so **verify what's currently available for new deployments** before designing around Serverless.
- **Azure Cosmos DB (NoSQL API):** throughput measured in **Request Units (RU)**. *Serverless:* pay per RU consumed, no provisioning, with per-container limits and no multi-region distribution (verify). *Provisioned throughput:* **manual** (fixed RU/s) or **autoscale** (scales between a fraction of a configured max RU/s and the max, billed at a different rate than manual; verify exact ratio), and optional multi-region replication. Data is split by **partition key** into logical partitions (with a size cap; verify) mapped to physical partitions; throughput is divided across physical partitions, so a hot partition gets throttled (HTTP 429) even if total RU/s are unused.
- **Cosmos DB for MongoDB, vCore-based:** a cluster-oriented model with dedicated compute/storage tiers (instead of RUs), closer to Atlas dedicated pricing logic. Product naming around this has been evolving; **verify current name and features**.

**Alternatives.** Self-managed MongoDB/PostgreSQL on VMs or Kubernetes (operators); managed PostgreSQL (RDS/Aurora/Cloud SQL/Azure Database for PostgreSQL) with serverless variants (e.g., Aurora Serverless v2; verify).

**When to use which.**
| Situation | Leaning |
|---|---|
| Prototype, dev/test, very spiky or low traffic, unknown load | Serverless/consumption |
| Steady, predictable traffic with high utilization | Dedicated/provisioned (cheaper per request) |
| Strict latency, large data, advanced features, private networking | Dedicated |
| Compliance requiring isolation | Isolated/dedicated, private endpoints |
| Small team, no DBA | Managed |
| Extreme scale where vendor margins dominate AND you have deep ops skill | Possibly self-hosted |

**Trade-offs.** Serverless: zero idle cost and no capacity planning vs. higher per-request cost at scale, cold/scale-up latency, limits, less control, bill surprises. Dedicated: predictable performance and cost vs. paying for idle capacity and doing capacity planning. Managed: less toil vs. less control and vendor lock-in. Self-hosted: control and unit cost vs. on-call, upgrades, backups, security are all yours (hidden cost = engineer time).

**Production problems.**
- *Serverless bill shock:* a runaway loop or full scan consumes millions of RUs/operations. Detect: cost anomaly alerts, RU/ops metrics per container/collection. Fix: query/index fixes, caps/budgets, rate limits. Prevention: budget alerts, load tests that report cost per request.
- *Throttling (429s) from hot partition or RU cap:* retries amplify load. Fix: better partition key, client retry with backoff honoring `retry-after`, raise throughput, cache.
- *Noisy neighbor / shared-tier limits* causing latency variance. Fix: move to dedicated.
- *Hidden limits* (connections, document size, ops/sec, storage per partition, feature gaps). Prevention: read the limits page before committing.
- *Migration lock-in:* RU-based modeling doesn't map 1:1 to other systems.

**Example.** Internal tool with 2 requests/second average and weekend silence: serverless is far cheaper than a dedicated tier. Public API with 800 requests/second steady for 12 hours/day: dedicated/provisioned usually wins (see break-even in 3.4).

**Common mistakes.** Choosing serverless "because it scales" without pricing the sustained workload. Ignoring partition-key design in RU-based systems. Treating managed as "no ops".

**Interview Q&A.**
- *MongoDB Serverless vs Dedicated?* Consumption pricing and automatic scaling vs. provisioned, predictable performance and fuller features; decide by traffic shape, latency needs, feature requirements, and break-even volume. (Full case study in Part 11.)
- *Cosmos DB serverless vs provisioned vs autoscale?* Serverless for sporadic low traffic; manual provisioned for steady predictable load; autoscale for variable load with a known maximum; always check partition distribution and RU cost per operation.

**Your turn (Q3).** Your Cosmos DB container is throttling (429) even though the dashboard shows only 30% of provisioned RU/s consumed. Explain why and the fixes.

---

## 3.4 Pricing Models and Break-Even Math `[SDE2]` P1

**What.** Pay-per-request (serverless), provisioned capacity, autoscaling, and reserved/committed pricing differ in how cost scales with load.

**Core formulas** *(all numbers illustrative)*

```
Serverless cost      C_s = R * p                (R = requests/month, p = price per request)
Provisioned cost     C_p = F                    (fixed monthly cost for capacity that covers PEAK)
Break-even requests  R* = F / p
Break-even avg RPS   = R* / 2,592,000           (seconds in a 30-day month)
Utilization          U  = average load / provisioned capacity
Effective cost/req   = F / R = (F / capacity) / U   -> halving utilization doubles unit cost
```

**Worked example *(illustrative)*.** Serverless price p = $0.25 per million requests; a provisioned setup costs F = $400/month and can handle a peak of 1,000 RPS.
- Break-even: R* = 400 / 0.25 x 1e6 = 1.6 billion requests/month = ~617 RPS *average*.
- If your average is 50 RPS (peak 400 RPS): serverless = 50 x 2.592e6 x 0.25e-6 = ~$32/month. Provisioned = $400. Serverless wins 12x.
- If your average is 800 RPS: serverless = ~$518/month vs. $400 provisioned. Provisioned wins, and gains more as utilization rises.
- Peak-to-average matters: provisioned must cover the peak. With peak/avg = 10x, utilization is ~10% and unit cost is 10x higher, which is exactly why spiky workloads favor consumption or autoscaling.

**Autoscaling nuance.** Autoscale tiers charge a premium per unit of capacity in exchange for tracking load, so they pay off when average utilization of a fixed provisioning would be low (roughly below the inverse of the premium ratio; verify the provider's multiplier). Autoscale also has a *floor* (minimum scaled capacity) and a *ceiling* (max), and scale-up isn't instantaneous.

**Reserved/committed discounts.** Commit to a baseline (1- or 3-year) for steady load; keep burst on-demand/autoscale. Risk: committing to capacity you later migrate away from.

**Hidden costs checklist.** Storage and backup retention; cross-AZ and cross-region replication traffic; egress; reads on replicas; indexing storage; minimum charges per container/instance; support plans; engineer time (self-hosted); over-provisioned non-prod environments; retries and throttling waste.

**How to measure.** Cost per 1,000 requests, per tenant, per feature; RU or read/write units per operation type; utilization (P50/P95) against provisioned capacity; month-over-month cost vs. request growth (is marginal cost flat or rising?).

**Cheaper alternatives.** Add caching to cut read volume; fix N+1 and full scans; batch writes; smaller documents/projections; archive cold data; right-size non-prod and schedule it off nights/weekends.

**Production problems.**
- *Unbounded autoscale during attack/retry storm:* bill spikes. Fix: max caps, rate limits, WAF.
- *Overprovisioned "just in case":* 10% utilization for months. Fix: monthly right-sizing review.
- *Commitment mismatch:* reserved capacity unused after migration. Fix: commit only to proven baseline.

**Common mistakes.** Comparing sticker prices without utilization. Forgetting peak provisioning. Ignoring the cost of reliability (replicas, multi-region multiply cost).

**Interview Q&A.**
- *How do you decide serverless vs provisioned?* Compute break-even volume from prices; compare to measured average and peak load; add feature, latency, and limit constraints; revisit as traffic grows.
- *What makes unit cost rise unexpectedly?* Low utilization, hot partitions causing over-provisioning, inefficient queries, retries, cross-region traffic.

**Your turn (Q4).** Provisioned: $900/month for 2,000 RPS peak. Consumption: $0.30 per million operations. Your average is 300 RPS with a 2,000 RPS peak. Which is cheaper, and at what average RPS do they break even?

---

## 3.5 Replication, Read Replicas, Sharding, Partitioning `[Both]` P0

**What.**
- **Replication:** copies of the same data on multiple nodes (availability, durability, read scale).
- **Read replica:** a replica used to serve reads (usually asynchronous).
- **Partitioning:** splitting a table into pieces *within one database* (e.g., by date) for manageability and pruning.
- **Sharding:** splitting data across *multiple database servers* so each holds a subset (write and storage scale-out).
- **Vertical scaling:** bigger machine. **Horizontal scaling:** more machines.

**Why.** One node has limits on CPU, RAM, disk, IOPS, and is a single point of failure.

**How it works.**
- *Replication modes:* **synchronous** (commit waits for replica: no data loss, higher latency, availability depends on replica) vs **asynchronous** (fast, but replica lag and potential data loss on failover; **RPO > 0**) vs semi-sync. Leader-follower (PostgreSQL, MongoDB replica set) vs leaderless (Cassandra, Dynamo-style).
- *Failover:* detect primary failure, elect/promote a replica, repoint clients. Clients must retry and reconnect.
- *Sharding strategies:* **range** (good for range scans, risk of hot ranges), **hash** (even spread, bad for range queries), **directory/lookup** (flexible, extra hop and a lookup service to run). **Consistent hashing** moves only ~1/N of keys when nodes change.
- *Partitioning in a single DB:* range by time makes dropping old data instant (`DROP PARTITION`) and lets the planner prune.

**Scaling ladder (try in order; each step adds complexity).**
1. Fix queries and indexes. 2. Cache. 3. Scale vertically. 4. Add read replicas. 5. Partition large tables. 6. Archive cold data. 7. Shard (last resort; hard to undo).

**Alternatives.** Distributed SQL (e.g., Spanner-like/CockroachDB-like systems: horizontal scale with SQL, at cost of latency and complexity; verify specifics), managed auto-sharding (Cosmos DB, DynamoDB), CQRS read stores.

**When not to use.** Read replicas when the app can't tolerate stale reads and can't route "read-your-writes" requests to the primary. Sharding before exhausting steps 1-6.

**Trade-offs.** Replicas: read scale and HA vs. lag and failover complexity. Sharding: scale vs. cross-shard queries/transactions, rebalancing pain, operational complexity, harder analytics.

**Production problems.**
- *Replication lag / stale reads:* user updates profile, refreshes, sees old data. Detect: replica lag metric. Fix: read-your-writes routing (read from primary for N seconds after a write, or session/causal consistency), lag-aware routing, fix lag source (long transactions, big batch jobs, under-sized replica).
- *Failover surprises:* connection storm after promotion, lost recent writes with async replication, split brain without proper fencing. Prevention: tested failover drills, retry with backoff and jitter, fencing, quorum-based election.
- *Replica as analytics server:* long queries conflict with replication apply (PostgreSQL recovery conflicts) or add lag. Fix: dedicated analytics replica or warehouse.
- *Resharding downtime:* adding shards moves data. Prevention: choose key and pre-split with headroom; use online migration tooling.
- *Cross-shard queries and transactions* slow or unsupported. Fix: model so most operations hit one shard (tenant-aligned), use sagas.

**Example.** SaaS with `tenant_id` shard key: tenant queries hit a single shard; a giant tenant becomes a hot shard, so large tenants get dedicated shards.

**Common mistakes.** Sharding too early. Assuming replicas give strong consistency. Using replicas as backups (they replicate deletes and corruption instantly).

**Interview Q&A.**
- *Replication vs sharding?* Replication copies the same data for availability/read scale; sharding splits different data across nodes for write/storage scale. Real systems use both (each shard is replicated).
- *Sync vs async replication?* Sync = zero data loss (RPO 0) but latency and availability coupling; async = fast but may lose recent writes on failover.
- *How do you handle replication lag?* Read-your-writes routing, monotonic reads via sticky replica, lag thresholds, causal consistency features.

**Your turn (Q5).** After enabling read replicas, users report "I saved my settings but the page shows old values." Diagnose and give two fixes with trade-offs.

---

## 3.6 Shard-Key Selection and Hot Partitions `[SDE2]` P0

**What.** The **shard/partition key** decides which node stores a record. Poor choices create **hot partitions**: one node saturated while others idle.

**Good key properties.**
1. **High cardinality** (many distinct values).
2. **Even distribution** of both data *and* traffic.
3. **Query isolation:** the most common queries include the key, so they hit one shard (targeted, not scatter-gather).
4. **Stable:** values rarely change (changing a key may mean moving the record).
5. **Not monotonic** if using range sharding (timestamps/auto-increment ids send all new writes to the last shard).

| Candidate key | Verdict |
|---|---|
| `created_at` (range) | Bad: all inserts hit the newest shard |
| `country` | Bad: low cardinality, skewed (one country dominates) |
| `user_id` (hashed) | Good for user-centric access; celebrity users can still run hot |
| `tenant_id` | Good isolation; skew if one tenant is huge |
| `(tenant_id, hash(user_id) % 16)` composite/bucketed | Spreads big tenants; costs scatter across 16 buckets for tenant-wide queries |

**Mitigations for hot keys.** Add a random/bucket suffix to spread writes (and fan-in on read); cache hot reads; split the hot tenant to a dedicated shard; use write batching/queues; for counters use sharded counters and aggregate.

**Cosmos DB specifics (verify limits).** The partition key must be chosen at container creation; RU/s is spread across physical partitions, so skew throttles one partition while others are idle; cross-partition queries cost more RUs; logical partitions have a size cap.

**Production problems.**
- *Hot shard:* symptom: one node at 95% CPU/IOPS or throttling (429/`ProvisionedThroughputExceeded`) while cluster average is low. Detect: per-partition/shard metrics (not just cluster-level). Mitigate: cache, rate-limit the hot key, temporary scale. Fix: re-key or bucket.
- *Scatter-gather queries:* latency = slowest shard, and cost scales with shard count. Fix: include the shard key, denormalize, secondary index/search store.
- *Unbalanced chunks after bulk load* (e.g., MongoDB chunk migration lag). Fix: pre-split and balance, hashed keys for ingestion.

**Example.** Social app sharded by `user_id`: a celebrity's profile reads overwhelm one shard. Fix: cache their profile aggressively, replicate the hot key across nodes, or serve via CDN.

**Common mistakes.** Optimizing for even *data* but not even *traffic*. Choosing a key that fits today's query but not tomorrow's. Assuming hashing fixes hot *keys* (it only fixes hot *ranges*).

**Interview Q&A.**
- *How do you choose a shard key?* High cardinality, even load, aligns with dominant queries, immutable, avoid monotonic; validate with real traffic distribution.
- *Hashed vs ranged sharding?* Hash = even distribution, poor range queries; range = efficient range scans, risk of hotspots.

**Your turn (Q6).** Design the shard key for a time-series IoT platform (10M devices, queries are "readings for device X in time range") and avoid hot partitions.

---

## 3.7 Consistency, Quorums, CAP/PACELC, and Multi-Region `[Both basics, SDE2 depth]` P1

**What.** *Strong consistency:* reads see the latest committed write. *Eventual consistency:* replicas converge if writes stop; reads may be stale meanwhile. *Intermediate models:* read-your-writes, monotonic reads, session, bounded staleness, causal.

**Why it matters.** Replicas and regions improve availability and latency but make "latest" ambiguous.

**How it works.**
- **Quorum (leaderless/tunable systems):** N replicas, write acknowledged by W, read from R. If **R + W > N**, read and write sets overlap, so a read sees the latest acknowledged write (absent edge cases like sloppy quorums). Example: N=3, W=2, R=2: 4 > 3, strongly consistent reads; W=1, R=1: fast but eventually consistent.
- **CAP:** during a network **P**artition, you choose **C**onsistency (reject/err) or **A**vailability (serve possibly stale). It says nothing about normal operation.
- **PACELC:** *if Partition: A or C; Else: Latency or Consistency.* This captures the everyday trade-off: stronger consistency usually costs latency (more round trips, cross-region waits).
- **Cosmos DB** offers five levels (strong, bounded staleness, session, consistent prefix, eventual); **MongoDB** uses read/write concerns (`majority`) and causal consistency sessions; **Cassandra** uses per-query consistency levels. Verify exact semantics and multi-region limitations in docs.
- **Multi-region patterns:** active-passive (one write region, failover; RPO depends on replication lag), active-active (multi-write; needs conflict resolution: last-writer-wins, CRDTs, app-level merge), read-local/write-global.

**When to use strong vs eventual.** Strong: money, inventory, uniqueness, permissions. Eventual: feeds, counters, analytics, caches, recommendations. Often *mixed* within one app.

**Trade-offs.** Strong: simpler reasoning, higher latency, lower availability in partitions. Eventual: speed and availability, harder application logic and anomaly handling. Multi-region: lower user latency and regional resilience vs. cost (replication traffic, extra capacity), conflict handling, and complexity.

**Production problems.**
- *Stale read bugs:* duplicate submissions, "ghost" missing data. Fix: read-your-writes, idempotency keys.
- *Multi-region write conflicts:* last-writer-wins silently drops updates. Fix: partition ownership by region, CRDTs, or avoid multi-write for conflict-prone data.
- *Region failover with lag:* data loss beyond expected RPO. Fix: monitor replication lag, test failover, set RPO expectations explicitly.
- *Cross-region latency in synchronous writes* (e.g., 80 ms RTT per commit). Fix: single write region near users plus async replicas.
- *Clock skew* in LWW conflict resolution. Fix: hybrid logical clocks, avoid wall-clock-only resolution.

**Example *(illustrative)*.** Cross-region synchronous commit with 80 ms RTT adds ~80 ms to every write; 12 sequential DB writes in a request add ~1 second. Redesign to batch or write locally with async replication.

**Common mistakes.** Saying "CAP means pick 2 of 3." Using eventual consistency for uniqueness constraints. Going multi-region without a clear RPO/RTO and cost estimate.

**Interview Q&A.**
- *Explain CAP and its limitation.* Under partition choose C or A; limitation: only about partitions, ignores latency; PACELC extends it.
- *What does R+W>N guarantee?* Overlap of read and write replica sets, so reads include the latest acknowledged write (assuming no sloppy quorum or concurrent writes anomalies).

**Your turn (Q7).** N=5 replicas. Pick R and W for (a) strongly consistent reads, (b) write-heavy workload tolerating stale reads. State trade-offs.

---

## 3.8 Connection Pooling and Connection Limits `[Both]` P0

**What.** Opening a DB connection is expensive (TCP, TLS, auth, session setup; in PostgreSQL a whole backend process). A **pool** keeps connections open and reuses them. DBs cap total connections (`max_connections`; PostgreSQL's default is 100, verify for your version/platform).

**Why.** Reduce latency and DB load; protect the DB from connection floods.

**How it works.**
- **App-side pool** (HikariCP, SQLAlchemy pool): each app instance holds N connections.
- **External pooler/proxy** (PgBouncer, RDS Proxy, ProxySQL): many client connections multiplexed onto few server connections. PgBouncer modes: session, transaction, statement; *transaction pooling* gives the most multiplexing but breaks session-level features (session variables, some prepared-statement usage, advisory locks across transactions).

**The math.**

```
Total possible connections = (#app instances) x (pool size per instance) + admin/migration/other clients
Must stay below DB max_connections (with headroom, e.g., 80%).

Needed pool size (Little's Law):  busy_connections ~= QPS x average_query_time_seconds
Example (illustrative): 2,000 QPS x 5 ms = 10 busy connections on average.
Add headroom for variance (maybe 2-3x), not 100s.
```

A database with 8 cores is often **faster with ~20-50 active connections than with 500**, because too many concurrent queries cause context switching and lock contention. A smaller pool plus queuing at the app gives better throughput and tail latency.

**Alternatives.** Serverless-friendly data APIs or proxies; reduce per-instance pool size; use async drivers with bounded concurrency; read replicas to spread load.

**Trade-offs.** Bigger pools increase concurrency but risk DB overload; too small causes app-side waits. Transaction-mode pooling gives scale but restricts features.

**Production problems.**
- *Connection exhaustion on autoscale:* HPA scales pods from 10 to 60; 60 x 20 = 1,200 connections > `max_connections` = 500, so new pods fail with "too many connections" and the outage deepens. Detect: `numbackends`/`pg_stat_activity` count, pool wait time, error logs. Fix: pooler, smaller per-pod pool, cap max replicas.
- *Serverless connection storm:* 1,000 concurrent function instances each open a connection. Fix: RDS Proxy/PgBouncer or HTTP data APIs, reuse connections across invocations, limit function concurrency.
- *Connection leaks:* connections not returned (missing `close`/`finally`). Symptom: pool exhausted gradually, restarts "fix" it. Detect: pool metrics (active vs idle), leak-detection thresholds. Fix: context managers, timeouts.
- *Idle-in-transaction connections* holding locks (see Part 1). Fix: `idle_in_transaction_session_timeout`, short transactions.
- *Failover storm:* every client reconnects at once. Fix: jittered backoff, pooler that absorbs reconnects.

**Example.** A Node service with 40 pods x pool 25 = 1,000 connections to a PostgreSQL primary with `max_connections` 300. Fix: PgBouncer in transaction mode, pool of 100 server connections shared by all pods.

**Common mistakes.** "More connections = more throughput." Setting pool size equal to expected concurrent requests. Forgetting migration jobs and BI tools also consume connections.

**Interview Q&A.**
- *Why can adding app servers crash the DB?* Each adds a pool; total connections exceed DB limits or overload it. Use a pooler and cap pool sizes.
- *How to size a pool?* Little's Law on query time and QPS, bounded by DB core count and `max_connections`; verify under load test.

**Your turn (Q8).** 50 pods, pool size 30, DB `max_connections` = 800, serverless cron jobs add up to 100 connections at the top of each hour. Will you hit the limit? Propose a fix.

---

## 3.9 Caching Patterns and Redis `[Both]` P0

**What.** A cache stores computed or fetched data closer/faster than the source of truth.

**Why.** Cut latency and protect the database/downstream services; absorb read spikes.

**Patterns.**

| Pattern | Flow | Pros | Cons |
|---|---|---|---|
| **Cache-aside** (lazy) | App reads cache; on miss reads DB, populates cache | Simple; only caches what's used; cache failure degrades, not breaks | First request slow; stale until TTL/invalidation |
| **Read-through** | Cache library loads from DB on miss | Cleaner app code | Needs cache-provider support |
| **Write-through** | Write goes to cache and DB synchronously | Cache always fresh | Higher write latency; caches data never read |
| **Write-behind** | Write to cache, async flush to DB | Fast writes, batching | **Data loss if cache crashes before flush**; complexity |
| **Write-around** | Write to DB only, cache fills on read | Avoids polluting cache | Read after write misses |

**Metrics and math.**

```
Hit rate h = hits / (hits + misses)
DB load = (1 - h) x read RPS                 e.g., 10,000 RPS x (1 - 0.95) = 500 DB reads/s
Average latency = h x T_cache + (1 - h) x (T_cache + T_db)
                  e.g., h=0.95, T_cache=1 ms, T_db=20 ms -> 0.95x1 + 0.05x21 = 2.0 ms  (vs 20 ms uncached)
Going from 95% to 99% hit rate cuts DB load by 5x (500 -> 100 RPS) *(illustrative)*.
```
Small hit-rate changes near the top matter a lot, and cache **outages** instantly multiply DB load by 1/(1-h) = 20x at h = 95%.

**Cache-aside with stampede protection and jittered TTL (Python sketch).**

```python
import json, random, time

NULL = "__null__"                                  # negative-caching marker

def get_user(r, db, uid, ttl=300, max_wait_tries=20):
    key = f"user:{uid}"
    for _ in range(max_wait_tries):
        v = r.get(key)
        if v is not None:
            return None if v == NULL else json.loads(v)
        token = str(random.random())
        if r.set(f"lock:{key}", token, nx=True, ex=10):       # only one loader proceeds
            try:
                row = db.fetch_user(uid)
                if row is None:
                    r.set(key, NULL, ex=60)                   # negative cache: stops penetration
                else:
                    r.set(key, json.dumps(row), ex=ttl + random.randint(0, 60))   # TTL jitter
                return row
            finally:
                if r.get(f"lock:{key}") == token:            # only release our own lock
                    r.delete(f"lock:{key}")                  # (use Lua for atomic compare-and-delete)
        time.sleep(0.05)                                      # others wait briefly, then re-read cache
    return db.fetch_user(uid)                                 # fallback after bounded waiting
```

**TTL and eviction.** TTL bounds staleness. When memory fills, Redis evicts per policy: `allkeys-lru`, `allkeys-lfu`, `volatile-lru`, `volatile-ttl`, or `noeviction` (writes fail with OOM errors). Cache workloads usually want an `allkeys-*` policy with a `maxmemory` set; verify defaults for your deployment.

**Redis specifics.** Data structures: strings, hashes, lists, sets, **sorted sets** (leaderboards, rate limiters), streams, bitmaps, HyperLogLog. Persistence: **RDB** snapshots (compact, can lose minutes) and **AOF** (append log, less loss, bigger/slower); neither makes Redis a substitute for a durable DB by default. Replication is asynchronous; Redis Cluster splits keys over 16,384 hash slots; multi-key operations need keys in the same slot (hash tags).

**Alternatives.** CDN/edge cache for static or public content; in-process (local) cache for tiny hot data (zero network hop, but per-instance inconsistency); database buffer cache; materialized views (3.10); HTTP caching headers (ETag/Cache-Control).

**When caching hurts.** Highly write-heavy or low-reuse data (low hit rate, just added latency and complexity); data needing strict read-after-write correctness without an invalidation strategy; tiny datasets the DB already keeps in memory.

**Invalidation.** The hard problem. Options: TTL only (simple, stale), delete-on-write (common), update-on-write, versioned keys (`user:42:v7`), event/CDC-driven invalidation.
*Race to know:* reader A misses and reads old value from DB; writer B updates DB and deletes cache; A then writes the **old** value into the cache, and it stays stale until TTL. Mitigations: short TTLs, versioned keys, delayed double-delete, CDC-based invalidation, or storing a version and rejecting older writes.

**Production problems.**
- *Cache stampede / thundering herd:* a popular key expires (or the cache restarts) and thousands of requests hit the DB at once. Symptom: DB CPU spike and latency every N minutes or after deploy. Mitigate: request coalescing/locks (above), serve-stale-while-revalidate, TTL jitter, pre-warming, probabilistic early refresh.
- *Cache penetration:* requests for keys that never exist (attacks or bugs) bypass the cache every time. Fix: negative caching, Bloom filter in front, input validation, rate limits.
- *Hot key:* one key overloads a single Redis node. Detect: `redis-cli --hotkeys`/monitoring. Fix: local in-process cache with short TTL, replicate key under multiple suffixed names, request coalescing.
- *Cold cache after deploy/failover* hammering the DB. Fix: warm-up, gradual traffic shift, capacity planning for cache-miss load.
- *Stale data/inconsistency* from failed invalidation. Fix: TTL as a safety net, versioning, CDC invalidation.
- *Memory pressure/eviction of useful keys,* big keys blocking Redis (`DEL` on a huge key blocks; use `UNLINK`). Fix: size limits, compress/shard big values.
- *Write-behind data loss.* Fix: use only where loss is tolerable, or use durable queue.
- *Cache as single point of failure:* the app errors when Redis is down. Fix: treat the cache as optional, timeouts, circuit breaker, DB protection (rate limit/shed load).

**Example.** Product page: cache-aside with 5-minute TTL plus jitter; inventory count is *not* cached (needs correctness) or cached with a very short TTL and verified at checkout.

**Common mistakes.** No TTL. Caching errors/nulls accidentally for too long. Cache keys without versioning on schema change. Assuming cache can't fail.

**Interview Q&A.**
- *Cache-aside vs write-through?* Cache-aside fills lazily on reads and tolerates cache loss; write-through keeps cache fresh on writes at the cost of write latency and caching unread data.
- *How do you prevent a cache stampede?* Locks/coalescing, TTL jitter, stale-while-revalidate, pre-warming.
- *How do you keep cache and DB consistent?* You can't make it perfect cheaply: use TTL as a bound, delete on write, versioning, CDC for stronger guarantees, and decide which data must bypass the cache.

**Your turn (Q9).** Your homepage API at 8,000 RPS has a 97% cache hit rate. The cache cluster restarts. Estimate DB load before and after, and describe a plan to survive that event.

---

## 3.10 Materialized Views and Read Optimization `[Both]` P1

**What.** A **materialized view** stores the *result* of a query physically and refreshes it periodically or incrementally. Related tools: summary tables, read models (CQRS), pre-aggregated counters, search indexes.

**Why.** Expensive joins/aggregations run once, and reads become cheap lookups.

**How it works.** `CREATE MATERIALIZED VIEW daily_revenue AS SELECT ...; REFRESH MATERIALIZED VIEW CONCURRENTLY daily_revenue;` (PostgreSQL: concurrent refresh requires a unique index and avoids blocking reads; verify). Alternatives to batch refresh: triggers, application-maintained summary rows, or streaming aggregation via CDC.

**Alternatives and the read-optimization toolbox.** Indexes (first!), covering indexes, query rewrite, caching, read replicas, denormalization, pagination (keyset), projection (select only needed fields), batching (avoid N+1), CDN for public data.

**When not to use.** Data that must be real-time exact; very high-churn sources where refresh cost approaches query cost; when a simple index fixes the problem.

**Trade-offs.** Staleness and storage vs. read speed; refresh jobs add load and operational surface.

**Production problems.**
- *Refresh blocking reads* (non-concurrent refresh takes an exclusive lock). Fix: concurrent refresh or swap tables.
- *Stale dashboards* with silent refresh failures. Detect: freshness metric/alert ("last refreshed at"). Fix: alerting, retries.
- *Refresh cost grows with data* until it can't finish within its schedule. Fix: incremental refresh or partitioned aggregation.
- *Summary tables drift from source of truth* due to bugs. Fix: periodic reconciliation job comparing sums.

**Example.** Leaderboard: maintain a Redis sorted set updated on each score event rather than recomputing `ORDER BY score DESC LIMIT 100` over millions of rows.

**Common mistakes.** Reaching for materialization before indexing. No freshness monitoring.

**Interview Q&A.**
- *Materialized view vs cache?* Both precompute; a materialized view lives in the database (SQL-queryable, refreshed by the DB), a cache is external, key-based, TTL-driven.

**Your turn (Q10).** A dashboard query joins 5 tables and takes 30 s. List your optimization steps in order, and when you'd introduce a materialized view.

---

## 3.11 Schema Evolution and Migrations `[Both]` P0

**What.** Changing schemas (or document shapes) on a live system without downtime or breaking running application versions.

**Why.** During a rolling deploy, **old and new app versions run simultaneously** against the same database; schema changes must be compatible with both.

**How it works: expand / migrate / contract.**

```
Goal: rename column  users.fullname -> users.display_name

1. EXPAND:    add new column display_name (nullable). Deploy no behavior change.
2. DUAL-WRITE: deploy app version that writes BOTH columns, reads old (or new with fallback).
3. BACKFILL:  copy old -> new in small batches (throttled), verify counts/checksums.
4. SWITCH:    deploy app that reads/writes only the new column.
5. CONTRACT:  after a bake period and rollback window, drop the old column.
Each step is independently deployable and reversible (until the final drop).
```

Rules: add columns as nullable/with cheap defaults; create indexes **concurrently/online**; never rename or drop in the same release that stops using it; avoid long exclusive locks (check the lock level of each DDL operation in your engine/version); make migrations idempotent and backward compatible; use feature flags for behavior switches.
*Document databases:* version documents (`schema_version`), write code that reads old and new shapes, migrate lazily on read or via background jobs.

**Alternatives.** Online schema change tools (gh-ost, pt-online-schema-change for MySQL; pg_repack and similar for PostgreSQL; verify suitability), blue-green databases (replicate, switch), strangler migration to a new store.

**When not to use full expand/contract.** Internal tools with maintenance windows where downtime is acceptable and the risk is low.

**Trade-offs.** Safety and zero downtime vs. multiple releases, temporary dual-write complexity, and extra storage.

**Production problems.**
- *Migration locks a hot table:* `ALTER TABLE` takes an exclusive lock or rewrites the table; all queries queue, pool exhausts, outage. Prevention: understand lock behavior, set `lock_timeout`, run off-peak, use online tools, add columns without volatile defaults where the engine rewrites.
- *Old app breaks on new schema (or vice versa)* mid-deploy. Prevention: backward-compatible steps, contract tests.
- *Backfill overloads DB:* huge `UPDATE` causes lag and bloat. Fix: batches with sleeps, throttle on replication lag, off-peak.
- *Irreversible migration:* dropped column then rollback needed. Prevention: keep contract step delayed; take a restore point.
- *Index build blocks writes.* Fix: concurrent build; watch for failed/invalid indexes.
- *Environments drift* (hand-applied hotfix). Fix: migration tooling as the only path, schema drift detection.

**Example.** Adding `NOT NULL` column with default on a 500M-row table: do it as nullable add, backfill in batches, then add constraint (validate separately where supported), never in one blocking statement.

**Common mistakes.** Combining destructive change with deploy. Testing migrations only on tiny datasets. No rollback plan.

**Interview Q&A.**
- *How do you rename a column with zero downtime?* Expand/migrate/contract (above).
- *Rollback vs roll-forward?* Rollback is easy before irreversible steps; after destructive DDL prefer roll-forward fixes; design migrations so app rollback stays possible.

**Your turn (Q11).** Plan a zero-downtime change splitting `orders.address` (text) into `street, city, zip` for a 300M-row table with rolling deploys.

---

## 3.12 OLTP vs OLAP, Warehouses, Lakes, ETL/ELT, CDC, Batch vs Stream `[Both basics, SDE2 depth]` P1

**What.**
- **OLTP:** many small transactions, point reads/writes, row-oriented, low latency.
- **OLAP:** fewer, large scans and aggregations over history, **column-oriented** storage, throughput-oriented.
- **Data warehouse:** structured, modeled data for BI (star schema). **Data lake:** raw files (often Parquet) in object storage; schema-on-read. **Lakehouse:** lake storage with table formats giving warehouse-like features (verify tool specifics).
- **ETL:** transform before loading. **ELT:** load raw first, transform inside the warehouse (common with cheap scalable warehouses).
- **CDC (Change Data Capture):** stream row changes from the database log (WAL/binlog/oplog) to other systems.
- **Batch vs stream:** periodic bulk processing vs continuous event processing.

**Why.** Analytics queries (full scans, big GROUP BYs) harm OLTP performance, and columnar layouts and compression make scans far cheaper.

**How it works.**
- *Columnar storage* reads only needed columns and compresses similar values well; great for `SUM(amount) GROUP BY day`, poor for fetching one full row.
- *CDC pipeline:* DB log -> connector (e.g., Debezium-style) -> Kafka -> sinks (warehouse, search index, cache invalidation). Reading the log avoids dual-write bugs and doesn't query tables.
- *Delivery semantics:* pipelines are usually at-least-once; make sinks **idempotent** (upserts keyed by primary key + version) to get effectively-once results.
- *Stream processing concepts:* event time vs processing time, windows, watermarks for late data.

**Alternatives.** Read replica for light reporting; materialized views; HTAP databases (mixed workloads; verify trade-offs); dual-writes (avoid: they diverge on partial failure); scheduled exports.

**When not to use.** A warehouse for point-lookups; streaming when daily batch meets the business need (streaming is costlier to build and operate); a lake without governance (becomes a "data swamp").

**Trade-offs.** Freshness vs. cost and complexity; schema-on-read flexibility vs. data quality risk; stream correctness (late/out-of-order data) vs. batch simplicity.

**Production problems.**
- *Analytics on the OLTP primary:* heavy queries evict caches, hold snapshots, inflate lag. Fix: replica/warehouse.
- *CDC lag or stuck connector:* downstream data stale; on PostgreSQL an inactive replication slot makes the primary retain WAL until disk fills. Detect: connector lag, slot lag. Fix: alerting, slot cleanup, backpressure.
- *Duplicate or out-of-order events:* double counting in aggregates. Fix: idempotent upserts, event versions/sequence, dedupe keys.
- *Schema changes breaking pipelines.* Fix: schema registry, compatibility rules, contract tests.
- *Small-files problem in lakes:* many tiny files slow queries and inflate metadata cost. Fix: compaction, batching writes.
- *Silent data quality drift.* Fix: row-count/null-rate checks, freshness SLAs, reconciliation with source.
- *Backfills reprocessing everything* with side effects. Fix: idempotent, partition-scoped reprocessing.

**Example.** Orders in PostgreSQL -> CDC -> Kafka -> (a) warehouse for finance reports, (b) search index for support tools, (c) cache invalidation topic.

**Common mistakes.** Treating the lake as a backup. Building streaming for a daily report. Ignoring late-arriving data.

**Interview Q&A.**
- *OLTP vs OLAP?* Workload shape and storage orientation (row vs column); separate them to protect transactional latency.
- *Why CDC over dual-writes?* The log is the single ordered source of truth; dual-writes can partially fail and diverge.
- *ETL vs ELT?* ELT loads raw data first and transforms in the warehouse, leveraging its compute and keeping raw data for reprocessing.

**Your turn (Q12).** A nightly Spark/SQL job reading the production primary makes checkout slow between 1-3 AM. Diagnose and redesign.

---

## 3.13 Storage Types, Tiering, Retention, Archival, and Deletion `[Both]` P1

**What.**
- **Object storage** (S3/Blob style): flat namespace of immutable objects over HTTP; very durable, cheap, virtually unlimited; high latency vs disk; no in-place edits.
- **Block storage** (EBS/managed disks): raw volumes attached to a VM/DB; low latency, IOPS/throughput provisioned.
- **File storage** (NFS/SMB/EFS-like): shared POSIX-like file system.
- **Tiers:** hot (frequent access, higher storage price, low access price), warm/infrequent, cold/archive (very cheap storage, retrieval fees, minimum storage durations, retrieval delay from minutes to hours). Verify each provider's tier names, prices, and minimum durations.

**Why.** Match cost to access frequency; most data is rarely read after it's a few weeks old.

**How it works.** *Lifecycle rules* automatically transition or expire objects by age/prefix/tag. For databases, **time-based partitioning** turns "delete rows older than 90 days" into `DROP PARTITION` (instant, no bloat) instead of a giant `DELETE`. TTL indexes (MongoDB, Redis TTL, Cassandra TTL) expire records automatically.

**Retention and deletion policy design.** Define per data class: retention period, legal/compliance requirements, deletion method (soft-delete flag vs hard delete), propagation to replicas/backups/caches/search/analytics, and audit evidence. Privacy laws can require erasure on request; backups complicate this (typical approach: bounded backup retention and a deletion ledger replayed after restore; confirm with legal/compliance).

**Alternatives.** Compress and archive to Parquet in a lake; downsample time series (keep rollups, drop raw); keep only aggregates.

**When not to use cold tiers.** Data read frequently, or small objects where per-request/retrieval fees exceed savings.

**Trade-offs.** Cost savings vs. retrieval latency/fees and early-deletion penalties; soft-delete keeps recoverability but bloats tables and risks compliance gaps.

**Production problems.**
- *Unbounded growth:* logs/events/audit tables never purged; storage and backup cost and query time balloon. Fix: retention policy, partitioning, TTL, archival.
- *Archive retrieval surprise:* restoring cold data takes hours and costs fees during an incident. Prevention: know retrieval SLAs; keep recent data hot.
- *Lifecycle rule deleting needed data* (misconfigured prefix). Prevention: versioning, object lock/soft-delete window, test rules on a sample prefix.
- *Large `DELETE` operations* causing lock contention, WAL growth, and replication lag. Fix: batch deletes, partition drops.
- *Small-object overhead:* millions of tiny objects raise request costs and slow listing. Fix: batch into larger files.
- *Orphaned objects* (DB row deleted but blob remains, or vice versa). Fix: reconciliation jobs, outbox/transactional deletion workflow.
- *Public bucket exposure.* Fix: block public access by default, access reviews.
- *Block volume IOPS limits* throttling DB. Fix: provision IOPS or larger volume class, monitor queue depth.

**Example.** Uploads: store files in object storage, metadata in PostgreSQL; lifecycle: hot 30 days, infrequent-access to 180 days, archive after, delete at 7 years per policy.

**Common mistakes.** Using object storage for random small writes or as a database. Forgetting storage minimum-duration fees. Deleting from the DB but not from search/cache/backups.

**Interview Q&A.**
- *Object vs block vs file?* Object: scalable immutable blobs over HTTP; block: raw low-latency disks for databases; file: shared hierarchical file access.
- *How do you delete data older than 1 year from a 5 TB table?* Time partitioning with partition drops; if not partitioned, batched deletes with throttling and index support, then reclaim space.

**Your turn (Q13).** Design the retention strategy for 2 TB/day of application logs: hot search 7 days, compliance retention 1 year, cost-conscious. Include the cost levers.

---

## 3.14 Backups, Snapshots, and Point-in-Time Recovery `[Both]` P0

**What.** Copies of data that let you restore after corruption, deletion, ransomware, or disaster. **PITR** restores to an arbitrary moment by replaying logs on top of a base backup.

**Why.** Replication is *not* backup: it copies mistakes instantly (a bad `DELETE` replicates to every replica).

**How it works.**
- **Full backup + incrementals/differentials** reduce size and time. **Snapshots** (volume-level) are fast and often incremental but may be crash-consistent only (not application-consistent) unless coordinated.
- **PITR:** base backup + continuous WAL/oplog archiving; restore base then replay to a target timestamp.
- **RPO** (max acceptable data loss) drives backup/log frequency; **RTO** (max acceptable downtime) drives restore speed and architecture.
- **3-2-1 rule:** 3 copies, 2 media/types, 1 off-site (different account/region). Add **immutability** (object lock/WORM) against ransomware and compromised credentials.

**Alternatives.** Cross-region replicas (DR, not backup); logical dumps (portable, slow to restore at scale); delayed replica (replica intentionally N hours behind); event sourcing replay.

**Trade-offs.** Frequency vs cost; restore time at TB scale is real (restoring 5 TB can take hours); encryption and access controls add key-management risk (lost keys = lost backups).

**Production problems.**
- *Untested backups:* the restore fails or is incomplete when needed. Prevention: scheduled automated **restore drills** with validation queries and measured RTO.
- *Backups in the same blast radius:* same account/region/credentials as production; ransomware or mistaken deletion removes both. Fix: separate account, immutability, restricted delete permissions.
- *PITR gap:* WAL archiving silently failing. Detect: archive lag alerts. Fix: monitor archiver success and recovery window.
- *Snapshot inconsistency:* restored DB needs crash recovery or has torn state across volumes. Fix: DB-native backups or quiesce/consistent snapshot groups.
- *Restore time exceeds RTO* because of data size. Fix: smaller shards, replicas promoted for failover, tiered restore strategy, practice.
- *Expired/lost encryption keys.* Fix: key backup/rotation procedure tested during drills.
- *Backup retention vs privacy deletion obligations* (see 3.13).

**Example *(illustrative)*.** RPO 5 minutes, RTO 1 hour: continuous WAL archive + daily base backup + warm standby replica in another zone; quarterly region-failure drill measuring actual recovery time.

**Common mistakes.** "We have replicas, so we have backups." Never testing restore. Not backing up configuration and secrets needed to run the restored system.

**Interview Q&A.**
- *Replica vs backup?* Replica protects availability; backup protects against logical corruption and deletion.
- *How would you recover from an accidental `DROP TABLE` at 14:03?* Stop writes if needed, restore base backup to a *separate* instance, replay logs to just before 14:03 (PITR), export the table, and reintegrate; then add safeguards (permissions, delayed replica).

**Your turn (Q14).** Your team's RPO is 1 minute and RTO is 15 minutes for a 4 TB database. Is nightly snapshot + replica enough? Explain what you'd add.

---

## 3.15 Serialization, Compression, and Data Formats `[Both]` P2

**What.** How data is encoded for storage and transfer: **JSON** (text, human-readable, schemaless), **Protocol Buffers/Avro/Thrift** (compact binary with schemas), **Parquet/ORC** (columnar files for analytics), **CSV** (simple, ambiguous).

**Why.** Format drives size, CPU cost, network/egress cost, compatibility, and query speed.

**How it works.**
- *Binary schema formats:* smaller and faster to parse; **schema evolution rules** let producers and consumers change independently (protobuf: add optional fields, **never reuse or renumber field tags**; Avro: reader/writer schema resolution with defaults).
- *Columnar formats:* group values by column with per-column compression and statistics (min/max) enabling skipped blocks.
- *Compression:* gzip (good ratio, slower), snappy/lz4 (fast, lower ratio), zstd (strong ratio/speed balance; verify for your tooling). Trade CPU for storage and bandwidth.

**When to use.** JSON for public APIs and human debugging; protobuf/Avro for internal service-to-service and Kafka with a schema registry; Parquet for analytics and lakes.

**Trade-offs.** Readability and flexibility vs. size, speed, and strictness; compression saves egress/storage but adds CPU and latency.

**Production problems.**
- *Breaking schema change* crashes consumers (renamed/removed field, reused protobuf tag). Prevention: compatibility checks in CI, schema registry.
- *Huge JSON payloads* inflating egress and latency. Fix: pagination, field selection, compression, binary formats internally.
- *Precision and type bugs:* floats for money, 64-bit ints in JavaScript (loses precision above 2^53), timezone-naive timestamps. Fix: integer cents/decimals, ISO-8601 with UTC, ids as strings in JSON.
- *CPU spikes from compression* of high-throughput paths. Fix: faster codec, compress larger batches, measure.
- *Poison messages* that can't be deserialized block a consumer. Fix: dead-letter queue with the raw payload.

**Example *(illustrative)*.** 10 KB JSON event at 5,000 events/s = 50 MB/s = ~130 TB/month raw; protobuf plus compression might cut that severalfold, which matters when it crosses regions (egress charges).

**Common mistakes.** Evolving schemas without compatibility rules. Using CSV for anything with commas/newlines/quotes in data. Compressing already-compressed data.

**Interview Q&A.**
- *Why Parquet for analytics?* Columnar layout and compression mean queries read only needed columns and skip blocks by statistics.
- *How do you evolve a Kafka event schema safely?* Backward/forward-compatible changes only (add optional fields with defaults), enforce with a schema registry, deploy consumers before producers for additions.

**Your turn (Q15).** A service sends 3 KB JSON events at 2,000/s cross-region. Estimate monthly transfer and name three ways to reduce cost.

---

## 4. Part 3 Review

### Summary

**Choose databases by access pattern and consistency need, default to PostgreSQL for relational data, and add components only for measured requirements.** Scale in a deliberate ladder (query/index fixes, cache, vertical, replicas, partitioning, shard last). Most outages here come from the same few things: **connection exhaustion, hot keys/partitions, stale or missing invalidation, stampedes after cold starts, migrations that lock, untested backups, and unbounded growth**. Pricing decisions come down to **utilization and peak-to-average ratio**: consumption wins when load is spiky or low; provisioned wins when steady and high.

### Decision checklist before picking infrastructure

1. What are the top 5 queries and their volumes (reads/writes per second, peak vs average)?
2. What consistency does each need (strong vs stale-OK)?
3. Data size now and in 1/3/5 years? Retention requirements?
4. RPO/RTO targets and compliance/isolation needs?
5. Team skills and on-call capacity?
6. What is the cost per 1,000 requests at today's and 10x load?
7. What is the simplest option that meets all of the above?

### Five review questions

- **R1.** Why is replication not a backup?
- **R2.** Why can adding pods bring down a database?
- **R3.** Give two reasons a cache can make a system *less* reliable.
- **R4.** What property of a shard key matters more than "even data distribution" in practice?
- **R5.** Why is `DROP PARTITION` better than `DELETE` for retention?

### Mock question

**M3.** "Your product catalog API serves 5,000 RPS, 98% reads, backed by a single PostgreSQL primary. p99 latency is 900 ms and CPU is 85%. Walk me through what you'd do, in order, to fix this without a rewrite. Then explain how you'd decide whether to add Redis, add read replicas, or shard. Include the failure modes of each."

---

## 5. Answer Key

**Q1.** PostgreSQL as the system of record for invoices, subscriptions, and users (transactions, constraints, reporting). User-uploaded forms with varying structure: PostgreSQL JSONB column per form instance with a validated schema version (indexed GIN where needed), or a separate document store only if document volume/shape demands it. Files in object storage with metadata in PostgreSQL. Redis for sessions/rate limits/cache. Add a search index only if full-text across forms becomes a requirement. Multi-tenancy: `tenant_id` on every table with indexes leading by it; consider row-level security. Justification: keep operational surface small, and exploit JSONB instead of adding a second database prematurely.

**Q2.** Cassandra has no joins and efficient reads require knowing the partition key, so you create one table per query. Example: `CREATE TABLE messages_by_conversation (conversation_id uuid, sent_at timeuuid, sender_id uuid, body text, PRIMARY KEY ((conversation_id), sent_at)) WITH CLUSTERING ORDER BY (sent_at DESC);` then `SELECT * FROM ... WHERE conversation_id = ? LIMIT 50;` reads one partition, already sorted newest first. For very active conversations, add a time bucket to the partition key (e.g., `(conversation_id, month)`) to keep partitions bounded.

**Q3.** Throughput (RU/s) is divided evenly across physical partitions, so a hot logical partition can exhaust its share and get 429s while total consumption is low (30%). Also a single expensive cross-partition query can spike. Fixes: better partition key with high cardinality and even traffic (or a synthetic/composite key), spread hot keys with bucketing, cache hot reads, add indexing/query fixes to reduce RU per operation, client retries honoring `retry-after` with backoff, and only then raise RU/s (which helps only proportionally because the hot partition's share grows slowly).

**Q4.** Consumption cost at 300 RPS average: 300 x 2,592,000 s = 777.6M operations/month x $0.30/M = ~$233/month. Provisioned = $900. Consumption is cheaper (~4x). Break-even: $900 / $0.30 per M = 3,000M ops/month = ~1,157 RPS average. Caveat: a 2,000 RPS peak must be allowed by the consumption model's limits (verify throttling/burst limits); also compare operational and feature differences. If average climbs toward ~1,100 RPS, switch or commit.

**Q5.** Cause: replication lag; the write went to the primary but the next read went to a replica not yet caught up. Fix 1: **read-your-writes routing**, send the user's reads to the primary for a short window (or until the replica reaches a recorded log position/token). Trade-off: more primary load, routing complexity. Fix 2: **session/causal consistency** or lag-aware replica selection (skip replicas above a lag threshold). Trade-off: depends on DB support and adds latency or staleness tuning. Also reduce the root cause of lag (long transactions, big batches, undersized replica) and, for the settings page specifically, optimistic UI update on the client.

**Q6.** Key: `(device_id, time_bucket)` with `time_bucket` = day or hour, hashed for placement by device (or hash(device_id) as partition key and time as clustering/sort key within). Reasons: 10M devices give high cardinality; queries "device X, time range" hit a small number of partitions (the buckets covering the range); bucketing bounds partition size. Avoid timestamp-only keys (all current writes hit one range). If some devices are far chattier, add a sub-bucket suffix or sample/aggregate on ingest. Use TTL/lifecycle for retention and rollups for older ranges.

**Q7.** (a) Strong reads: R + W > 5, e.g., W = 3, R = 3 (majority/majority). Tolerates 2 node failures for both reads and writes; higher latency than R/W = 1. (b) Write-heavy tolerating stale reads: W = 1, R = 1 (or W = 1, R = 2). Lowest write latency and highest availability, but stale reads and risk of lost acknowledged writes if the single acknowledging replica fails before replicating. Middle: W = 2, R = 1 improves durability with fast reads but doesn't guarantee overlap (2+1 = 3 not > 5).

**Q8.** 50 x 30 = 1,500 steady-state potential connections, already above 800 even before cron jobs; plus up to 100 more at the top of the hour. Even if pools are only partly filled at idle, the maximum can exceed limits during scale-up. Fixes: put PgBouncer/RDS Proxy (transaction pooling) in front (e.g., 200 server connections shared by everything), reduce per-pod pool to ~5-10 sized via Little's Law, cap HPA max replicas, stagger/limit cron concurrency and route them through the pooler, set alerts on connection count and pool wait time.

**Q9.** Before: DB load = 8,000 x 0.03 = 240 reads/s. After restart (cold cache): ~8,000 reads/s initially, ~33x more. Plan: protect the DB with request coalescing (single-flight per key), rate limiting/load shedding for non-critical endpoints, serve stale or static fallbacks, staged traffic ramp, pre-warm hot keys (top N by recent traffic), TTL jitter, cache cluster redundancy (replicas, multi-AZ) so restarts aren't total, capacity-test the DB for a miss storm, and circuit breakers to fail gracefully rather than melt down.

**Q10.** Order: (1) look at the plan (`EXPLAIN ANALYZE`) and fix indexes/stats; (2) rewrite the query (remove unneeded joins/columns, pre-filter, avoid functions on indexed columns); (3) check data volume/partitioning; (4) cache if parameters repeat; (5) pre-aggregate with summary tables or a **materialized view** when the query is inherently heavy, the result tolerates staleness (minutes/hours), and it's read far more than refreshed; (6) move to a replica/warehouse if it contends with OLTP. Add freshness monitoring.

**Q11.** Expand: add nullable `street, city, zip`. Deploy app v2 that writes both old `address` and the new fields (dual-write), reading from the new fields with fallback to parsing old. Backfill in throttled batches by primary key range (parse, handle unparseable rows into an exceptions table), monitor replication lag and bloat, verify counts/samples. Deploy v3 that reads/writes new fields only. Add constraints/indexes online (concurrently). After a bake period and rollback window, contract: drop `address`. Keep each step reversible; use feature flags; test on a production-sized copy; set `lock_timeout`.

**Q12.** The batch job competes with OLTP for CPU, I/O, cache (buffer eviction), and possibly locks/snapshots (long-running transactions delay vacuum and add replication lag). Confirm with `pg_stat_activity`, I/O metrics, cache hit ratio during that window. Redesign: run analytics against a read replica or, better, a warehouse fed by CDC/ELT; schedule heavy jobs off a dedicated replica; incrementally process changes instead of full scans; add resource limits/timeouts; monitor replica lag; and keep the primary for transactions.

**Q13.** 2 TB/day: ~14 TB per 7-day hot window (before replicas/indexing overhead), ~730 TB/year raw. Hot (search index, 7 days) on fast storage with replicas only as needed; after 7 days, drop the index and keep compressed logs in object storage (columnar/compressed formats, e.g., zstd/Parquet where applicable) in infrequent-access/cold tiers with lifecycle rules; delete at 12 months (or per compliance). Cost levers: sampling/dropping debug logs at source, structured logging with only needed fields, shorter hot retention, compression, tiering, avoiding cross-region transfer, fewer index replicas, querying archived logs on demand rather than keeping them indexed, and cost dashboards per service. Ensure retention is legally verified and backups don't violate deletion obligations.

**Q14.** No. Nightly snapshot gives an RPO up to ~24 hours, far from 1 minute; a replica gives high availability but not protection from logical corruption, and async replication may lose seconds of data. Add: continuous WAL/oplog archiving for PITR (RPO ~ seconds to a minute), synchronous or semi-synchronous standby if RPO must be near-zero (accepting latency), automated failover tested for 15-minute RTO, a delayed replica or immutable off-account backups, and regular restore drills measuring real RTO (restoring 4 TB from backup may exceed 15 minutes, so the failover replica is the primary RTO mechanism while backups cover disasters and corruption).

**Q15.** 3 KB x 2,000/s = 6 MB/s; per 30-day month: 6 MB x 2,592,000 s = ~15.5 TB. Ways to reduce: compress (batching + zstd/gzip, often large savings on JSON), switch to a compact binary schema (protobuf/Avro), drop unneeded fields/send deltas, batch events to reduce overhead, keep producers and consumers in the same region or use regional processing and send aggregates cross-region, and use private/inter-region links priced better than public egress (verify provider pricing).

**R1.** Replicas faithfully replicate deletes, corruption, and bad deployments within seconds; backups are point-in-time, isolated copies you can roll back to.

**R2.** Each pod has its own pool; scale-out multiplies total connections beyond `max_connections` or overloads the DB's concurrency capacity; the new pods also fail to connect, deepening the outage. Use a pooler and caps.

**R3.** (i) Stale/inconsistent data from failed invalidation, (ii) the DB becomes dependent on the cache's hit rate, so a cache outage, cold start, or stampede multiplies DB load (e.g., 20x at a 95% hit rate) and can cause a cascading failure.

**R4.** Even **traffic** distribution (and query isolation): a key that evenly spreads data but concentrates requests (celebrity user, giant tenant) still creates a hot shard.

**R5.** `DROP PARTITION` removes a whole file/segment instantly with minimal WAL, no dead tuples/bloat, and no long locks; `DELETE` touches each row, generates WAL and dead rows, causes vacuum work and replication lag.

**M3 (outline of a strong answer).** *Step 0:* measure; get top queries by total time (`pg_stat_statements`), p99 by endpoint, DB wait events, connection count, cache hit ratio. *Step 1: fix queries/indexes* (missing indexes, N+1, `SELECT *`, unbounded lists, deep OFFSET) since 85% CPU with 98% reads usually means a few heavy queries. *Step 2: connection pooling* to cut overhead. *Step 3: caching* for hot catalog reads (cache-aside + TTL + jitter + coalescing); at a 95% hit rate DB reads drop from ~4,900/s to ~245/s *(illustrative)*; failure modes: stampede, stale prices/inventory (don't cache or use very short TTL for price/stock, validate at checkout), cache outage, hot keys. *Step 4: read replicas* when reads remain heavy and some staleness is OK; failure modes: replication lag, read-your-writes, failover handling. *Step 5: vertical scale* as the quick lever. *Step 6: partition large tables, archive old data.* *Step 7: shard* only if write volume or data size exceeds one primary after all the above; choose key, accept cross-shard complexity. Decision rules: repetitive hot reads -> cache; read volume with tolerable staleness -> replicas; single-node write/storage ceiling -> shard. Validate each change with load tests and monitor p99, DB CPU, hit rate, lag; define rollback.

---

### Next: Part 4 of 13
Compute, networking, and distributed systems (Sections 2, 4, 6). Reply **"continue"**.
