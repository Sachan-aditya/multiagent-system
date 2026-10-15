# Part 1 of 13: Concept Map, Glossary, and CS Fundamentals

> Level tags: `[SDE1]` `[SDE2]` `[Both]`. Priority: P0 = must know, P1 = should know, P2 = nice to have.
> All numbers marked *(illustrative)* are teaching examples, not provider prices or measured metrics.
> Every "Your turn" question has a model answer in the **Answer Key** at the end. Try it yourself first.

---

## 1. System Design Concept Map

### 1.1 The request path

```
 User/Client
     |
     v
 DNS --> CDN/Edge --> Load Balancer (L4/L7) --> API Gateway / Reverse Proxy
                                                      |
                                                      v
                                   Compute: VM | Container/K8s | Serverless
                                                      |
              +------------------+--------------------+------------------+
              v                  v                    v                  v
           Cache (Redis)    Primary DB          Message Queue /     Object Store
                            + Replicas          Event Stream        (S3/Blob)
                            (SQL/NoSQL)         (Kafka/SQS)              |
                                                      |                  v
                                                      v            Data Lake / Warehouse
                                               Async Workers        (analytics, OLAP)
                                               (emails, AI jobs)

 Cross-cutting layers (touch every box above):
 Reliability (HA, DR) | Observability (logs, metrics, traces) | Security (authN/Z, encryption)
 Cost (FinOps)        | CI/CD & Operations                   | AI layer (LLM, RAG, agents)
```

### 1.2 How the concepts interact (the part most people miss)

| If you change... | It directly affects... | Because... |
|---|---|---|
| Autoscale app pods from 5 to 50 | DB connections | each pod holds a pool; 50 pods x 20 conns = 1,000 connections |
| Add a cache | Consistency, cost, failure modes | stale data possible; cache outage dumps load on the DB |
| Add retries | Load during outages | retries multiply traffic; can turn a blip into an outage |
| Go multi-region | Latency, cost, consistency | replication lag, egress charges, conflict handling |
| Move to microservices | Latency, ops burden, consistency | network hops, distributed tracing, no cross-service transactions |
| Use serverless compute | Cold starts, DB connection storms, cost curve | many short-lived instances; pay per request |
| Increase log verbosity | Cost, performance | ingestion and storage cost grows with traffic |
| Tighten SLO (99.9% to 99.99%) | Architecture and bill | needs redundancy, multi-AZ or region, more ops effort |

### 1.3 Learning roadmap (fundamentals to advanced)

1. **Foundations** `[Both]`: OS, networking, DB theory, Linux debugging (this part)
2. **Core building blocks** `[Both]`: databases, indexes, caching, queues, load balancing
3. **Design skills** `[Both]`: API design, LLD, patterns, HLD problem solving
4. **Operations** `[SDE2 heavy]`: observability, reliability, CI/CD, incidents
5. **Advanced** `[SDE2]`: distributed transactions, consensus, multi-region, cost engineering, AI infrastructure

---

## 2. Glossary of Essential Terms

| Term | Meaning |
|---|---|
| ACID | Atomicity, Consistency, Isolation, Durability: transaction guarantees |
| Availability | Fraction of time a system serves requests correctly |
| Backpressure | Signaling producers to slow down when consumers can't keep up |
| Blast radius | How much of the system/users a failure affects |
| Bulkhead | Isolating resources so one failing part can't sink everything |
| Cache-aside | App checks cache, on miss reads DB then fills cache |
| CAP theorem | Under a network partition, choose consistency or availability |
| CDC | Change Data Capture: streaming DB changes to other systems |
| Circuit breaker | Stops calling a failing dependency for a cool-down period |
| Consistent hashing | Key-to-node mapping that moves few keys when nodes change |
| Control plane / data plane | Management/config layer vs. layer that handles actual traffic |
| CQRS | Separate models for writes (commands) and reads (queries) |
| Cold start | Extra latency when a serverless instance starts from scratch |
| Connection pool | Reusable set of open DB connections |
| Deadlock | Two or more tasks each waiting for a resource the other holds |
| Durability | Committed data survives crashes |
| Egress | Data leaving a cloud network (often billed) |
| Eventual consistency | Replicas converge if updates stop; reads may be stale meanwhile |
| Fan-out | One event triggering many downstream actions |
| Idempotency | Repeating an operation has the same effect as doing it once |
| Index selectivity | How well an index narrows rows (high cardinality = selective) |
| Isolation level | How much concurrent transactions can see each other's changes |
| Jitter | Random delay added to retries to avoid synchronized spikes |
| Leader election | Choosing one node to coordinate |
| Little's Law | Concurrency = throughput x latency |
| Load shedding | Deliberately rejecting some work to protect the system |
| MVCC | Multi-Version Concurrency Control: readers use snapshots, don't block writers |
| MTTD / MTTR | Mean time to detect / recover |
| Multi-tenancy | One system serving multiple customers with logical isolation |
| Observability | Ability to understand internal state from logs, metrics, traces |
| OLTP / OLAP | Transactional workloads vs analytical workloads |
| p50/p95/p99 | Latency percentiles; p99 = slowest 1% boundary |
| Partition / shard | A slice of data placed on a node |
| Quorum | Majority agreement (e.g., 2 of 3 replicas) |
| Rate limiting | Capping requests per client per time window |
| Read replica | Copy of a DB serving reads, usually asynchronous |
| Replication lag | Delay between primary write and replica visibility |
| RPO | Recovery Point Objective: max tolerable data loss (time) |
| RTO | Recovery Time Objective: max tolerable downtime |
| Saga | Sequence of local transactions with compensating actions |
| Saturation | How "full" a resource is (queue depth, pool usage) |
| SLI / SLO / SLA | Measured indicator / internal target / contractual promise |
| Split brain | Two nodes both believe they are leader |
| Stateless service | Holds no per-user state between requests; easy to scale out |
| Thundering herd | Many clients hitting a resource simultaneously (e.g., after cache expiry) |
| TTL | Time to live: expiry time for cache entries, DNS records, etc. |
| WAL | Write-Ahead Log: changes logged before applying to data files |

---

## 3. CS Fundamentals (Section 22 of the guide)

Each topic follows the mandatory template: What, Why, How, Alternatives, When/When not, Trade-offs, Production problems, Example, Mistakes, Interview Q&A, Your turn.

---

### 3.1 Processes, Threads, and Context Switching `[Both]` P0

**What.** A *process* is a running program with its own isolated address space. A *thread* is an execution path inside a process; threads of one process share heap and globals but each has its own stack and registers.

**Why.** CPUs are fast, I/O is slow. While one task waits on disk or network, the CPU should run another. Processes add isolation (a crash doesn't take down others); threads add cheap sharing.

**How it works.**
- The OS keeps a control block per process/thread (registers, program counter, stack pointer, state).
- A **preemptive scheduler** gives each runnable thread a time slice (typically milliseconds).
- A **context switch** saves the running thread's registers and loads another's. Switching between processes also swaps page tables (and may flush TLB entries), so it is costlier than a thread switch.
- Direct cost is on the order of microseconds *(illustrative)*, but the indirect cost (cold CPU caches, TLB misses) often dominates.

**Alternatives.**
| Model | Example | Good for |
|---|---|---|
| Multi-process | Gunicorn workers, Chrome tabs | isolation, bypassing Python's GIL for CPU work |
| Multi-thread | Java servlet pools | shared state, CPU + I/O mix |
| Event loop (single thread, non-blocking I/O) | Node.js, asyncio | many idle I/O-bound connections |
| Lightweight/virtual threads | Go goroutines, Java 21 virtual threads | huge concurrency with simple blocking-style code |

**When / when not.** CPU-bound work: roughly one worker per core. I/O-bound: event loop or lightweight threads. Don't create thousands of OS threads, since each reserves stack memory (commonly around 1 MB of virtual space on the JVM, configurable) and adds scheduler load.

**Trade-offs.** Threads: easy sharing but race conditions and deadlocks. Processes: safe isolation but need IPC and more memory. Event loop: very efficient but one blocking call stalls everything.

**Production problems.**

*Thread pool exhaustion*
- Symptom: latency climbs, then timeouts and 503s, though CPU is low.
- Root cause: a slow dependency holds threads while waiting. Little's Law: needed concurrency = RPS x latency. At 500 RPS and 0.2 s latency you need ~100 busy threads; if the dependency slows to 2 s you need ~1,000, so a pool of 200 saturates and requests queue *(illustrative)*.
- Detect: thread dump (`jstack`, `py-spy dump`), pool active/queue metrics, p99 latency rising while CPU is flat.
- Mitigate: shed load, restart if deadlocked, scale out, fail fast on the slow dependency.
- Permanent fix: timeouts on every outbound call, bulkheads (separate pools per dependency), circuit breakers, async I/O.
- Prevention: alert on pool saturation and queue depth.

*Context-switch thrashing*
- Symptom: high system CPU, low throughput, with hundreds of runnable threads on few cores.
- Detect: `vmstat 1` (`cs` column), `pidstat -w`. Fix: bound pool sizes, reduce lock contention.

**Example.** Python web API doing CPU-heavy image resizing: threads don't help because of the GIL (only one thread runs Python bytecode at a time), so use multiple processes or offload to a worker queue.

**Common mistakes.** "More threads = faster." Confusing *concurrency* (dealing with many things) with *parallelism* (doing many things simultaneously). Unbounded thread creation.

**Interview Q&A.**
- *Process vs thread?* Process: isolated memory, heavier. Thread: shares memory in a process, lighter, needs synchronization.
- *Why is a process switch costlier?* Page-table swap, TLB invalidation, and colder caches.
- *How would you size a thread pool?* CPU-bound: ~cores. I/O-bound: roughly cores x (1 + wait/compute), and verify with load testing and Little's Law.

**Your turn (Q1).** A service has a 100-thread pool, handles 300 RPS, and its main dependency's latency rises from 100 ms to 1 s. What happens, and what three changes would you make?

---

### 3.2 Virtual Memory and Paging `[Both]` P1

**What.** Each process sees its own contiguous virtual address space. The OS and MMU map virtual pages (commonly 4 KB) to physical RAM frames through page tables.

**Why.** Isolation between processes, simple memory allocation, memory-mapped files, and the ability to use disk as overflow.

**How it works.**
- Virtual address = page number + offset. The page table maps page number to frame. The **TLB** caches recent translations.
- **Minor page fault:** page is in memory but not yet mapped. **Major page fault:** page must be read from disk (swap or file), which is thousands of times slower than RAM.
- Under memory pressure, the OS evicts pages (approximate LRU) to swap. If memory is exhausted, the Linux **OOM killer** terminates a process.
- **Copy-on-write:** `fork()` shares pages until one side writes.
- **Huge pages** (e.g., 2 MB) reduce TLB misses.

**Alternatives.** Segmentation, or no virtual memory (some embedded systems). Application-level: manual memory pools, off-heap memory.

**Trade-offs.** Safety and flexibility vs. translation overhead and unpredictable page-fault latency.

**Production problems.**

*Swap thrashing*
- Symptom: system crawls, p99 latency explodes, disk busy, CPU idle.
- Detect: `vmstat 1` (`si`/`so` columns), `free -m`, major faults via `sar -B`.
- Fix: add memory or reduce working set; set memory limits correctly; for latency-critical databases keep swap minimal.

*OOMKilled container*
- Symptom: pod restarts, exit code 137, `OOMKilled` in `kubectl describe pod`.
- Root cause: process (or JVM heap + off-heap + threads) exceeded the cgroup memory limit.
- Fix: raise the limit or fix the leak; size JVM heap below the container limit with headroom for metaspace, threads, and native memory.

*Transparent Huge Pages latency*
- Vendors of Redis and MongoDB document recommending THP be disabled because of latency spikes; verify in current docs for your version.

**Example.** Redis forks to write an RDB snapshot. Copy-on-write means pages modified during the snapshot get duplicated, so a write-heavy instance can need substantially more memory than its dataset, and may be OOM-killed if the host has no headroom.

**Common mistakes.** Treating "memory used" as RSS only. Setting container limit equal to JVM max heap. Assuming swap is always bad or always fine.

**Interview Q&A.**
- *What is a page fault?* Access to a virtual page not currently mapped to RAM; the kernel handles it by mapping or loading the page.
- *Why does fork() seem cheap?* Copy-on-write.
- *Pod keeps getting OOMKilled but heap looks fine. Why?* Non-heap memory (threads, native buffers, metaspace, mmap) counts toward the cgroup limit.

**Your turn (Q2).** Your Redis instance with 8 GB data on a 12 GB host is OOM-killed during snapshots. Explain why and give two fixes.

---

### 3.3 Synchronization and Deadlocks `[Both]` P0

**What.** Synchronization coordinates access to shared state. A *deadlock* is a state where tasks wait on each other forever.

**Why.** Without it, concurrent read-modify-write operations corrupt data (race conditions).

**How it works.**
- **Mutex:** one holder at a time. **Semaphore:** N permits. **Read-write lock:** many readers or one writer. **Condition variable:** wait until a condition is true. **Atomics / CAS:** lock-free compare-and-swap.
- Deadlock needs all four **Coffman conditions**: mutual exclusion, hold-and-wait, no preemption, circular wait.
- Break any one to prevent deadlock. Most practical: **global lock ordering** (kills circular wait) or **timeouts/tryLock**.

**Alternatives.** Immutable data, message passing (actors, channels), optimistic concurrency (version checks), single-writer designs, database transactions.

**When / when not.** Locks for short critical sections on shared in-memory state. Avoid holding locks across I/O or network calls.

**Trade-offs.** Coarse locks: simple but low throughput. Fine-grained locks: faster but deadlock-prone. Lock-free: fast but hard to get right.

**Production problems.**

*Application deadlock*
- Symptom: requests hang, CPU idle, thread pool full.
- Root cause: `transfer(A,B)` locks A then B while `transfer(B,A)` locks B then A.
- Detect: thread dump shows threads BLOCKED waiting on each other (`jstack` reports deadlocks).
- Mitigate: restart the instance. Fix: always lock in a consistent order (e.g., by account id).

*Database deadlock*
- Symptom: occasional "deadlock detected" errors.
- Root cause: transactions update the same rows in different orders. The DB detects the cycle and aborts one transaction.
- Fix: consistent update ordering, shorter transactions, and **retry** the aborted transaction.
- Detect: `pg_locks` / `pg_stat_activity` (PostgreSQL), `SHOW ENGINE INNODB STATUS` (MySQL).

**Example.** Two concurrent withdrawals both read balance 100, each subtract 80, and both write 20. Fix: atomic `UPDATE accounts SET balance = balance - 80 WHERE id = ? AND balance >= 80`, check rows affected.

**Common mistakes.** Locking in inconsistent order. Holding a lock while calling an external service. Assuming "it works in testing" means no race.

**Interview Q&A.**
- *Four conditions for deadlock?* Mutual exclusion, hold-and-wait, no preemption, circular wait.
- *Mutex vs semaphore?* Mutex has an owner and protects a critical section; semaphore is a counter allowing N concurrent holders.
- *How do you prevent DB deadlocks?* Consistent ordering, short transactions, indexes to reduce locked rows, retry on deadlock errors.

**Your turn (Q3).** Write the bug in a `transfer(from, to, amount)` that uses two locks, then fix it.

---

### 3.4 I/O Models, Event Loops, and epoll `[Both]` P1

**What.** Ways a program waits for I/O: blocking, non-blocking, multiplexed (`select`/`poll`/`epoll`), and asynchronous (`io_uring`).

**Why.** The **C10K problem**: one thread per connection doesn't scale to tens of thousands of mostly-idle connections.

**How it works.** With `epoll` (Linux), a thread registers many file descriptors with the kernel and asks "which are ready?" The kernel returns only the ready ones, so one thread serves thousands of sockets. An **event loop** repeatedly pulls ready events and runs short callbacks. Nginx, Node.js, and Redis are built this way.

**Alternatives.** Thread-per-request (simple, heavier), goroutines and virtual threads (blocking-style code, scheduled efficiently by the runtime), actor systems.

**When / when not.** Event loops shine for many I/O-bound connections (proxies, chat, gateways). They are poor for CPU-heavy handlers unless work is offloaded.

**Trade-offs.** Efficiency and low memory vs. "colored function" complexity (async/await everywhere) and the rule that you must never block the loop.

**Production problems.**

*Blocking call in the event loop*
- Symptom: all requests stall together, latency spikes across unrelated endpoints, CPU may be pegged on one core.
- Root cause: sync file read, CPU-heavy JSON parsing, a regex with catastrophic backtracking, or `requests.get` inside asyncio.
- Detect: event-loop lag metrics, profiler flame graph, `py-spy`, `--cpu-prof` in Node.
- Fix: move CPU work to worker threads/processes, use async clients.

*File descriptor exhaustion*
- Symptom: "too many open files", failed accepts.
- Detect: `ulimit -n`, `lsof -p <pid> | wc -l`. Fix: raise limits appropriately, close leaked sockets, pool connections.

**Example.** Node.js API parses a 20 MB JSON body synchronously; during that time the single thread can't answer health checks, so the load balancer marks the instance unhealthy.

**Common mistakes.** Thinking async makes CPU work faster. Mixing blocking libraries into async code.

**Interview Q&A.**
- *Why is epoll better than select?* It avoids scanning all fds each call and scales with the number of *ready* fds, not total fds.
- *Is Node.js single-threaded?* JS code runs on one thread; I/O and some tasks use a background thread pool (libuv).

**Your turn (Q4).** An async Python API intermittently freezes for 2-3 seconds across all endpoints. List how you would find the cause.

---

### 3.5 TCP Internals `[Both]` P0

**What.** TCP provides a reliable, ordered, connection-oriented byte stream over unreliable IP.

**Why.** Applications want reliability without implementing retransmission and ordering themselves.

**How it works.**
- **Handshake:** SYN, SYN-ACK, ACK (1 RTT before data). TLS 1.3 adds another RTT (TLS 1.2: 2).
- **Reliability:** sequence numbers, ACKs, retransmission on timeout or duplicate ACKs.
- **Flow control:** receiver advertises a window so it isn't overwhelmed.
- **Congestion control:** slow start, then additive increase / multiplicative decrease (Cubic, BBR). New connections start slow, which is why connection reuse matters.
- **Teardown:** FIN/ACK. The side that closes first enters **TIME_WAIT** (60 s on Linux) holding the port.

**Alternatives.** UDP (no reliability; used for DNS, video, games), **QUIC/HTTP/3** (reliability over UDP, integrated TLS, no head-of-line blocking across streams).

**When / when not.** TCP for most request/response and streaming needs; UDP/QUIC when latency or loss tolerance matters.

**Trade-offs.** Reliability and fairness vs. handshake latency and head-of-line blocking.

**Production problems.**

*Ephemeral port exhaustion / TIME_WAIT pile-up*
- Symptom: outbound calls fail with "cannot assign requested address" under load.
- Root cause: many short-lived outbound connections; Linux's default ephemeral range is roughly 28k ports (32768-60999), each held ~60 s after close. Hundreds of new connections/second to one destination exhausts it.
- Detect: `ss -s`, `ss -tan state time-wait | wc -l`.
- Fix: HTTP keep-alive and connection pools, reuse clients, fewer short-lived connections. Avoid blindly tuning kernel flags.

*Idle-timeout mismatch causing resets*
- Symptom: sporadic "connection reset" or 502 on first request after idle.
- Root cause: load balancer or NAT drops idle connections after N seconds while the client still thinks they're open.
- Fix: keep-alive timeout on app server longer than the LB idle timeout, client-side connection validation or TCP keepalives.

*Retransmissions and packet loss*
- Symptom: latency spikes without CPU change. Detect: `ss -ti`, `netstat -s`, `tcpdump`/Wireshark.

**Example *(illustrative)*.** Cross-region RTT 80 ms. New connection + TLS 1.3 handshake = ~160 ms before the first request byte; the request itself costs another 80 ms. With keep-alive, only the 80 ms remains. At thousands of calls this is why connection reuse is a major latency and cost lever.

**Common mistakes.** Creating a new HTTP client per request. Assuming TCP guarantees *message* boundaries (it's a byte stream). Setting aggressive kernel tweaks without understanding.

**Interview Q&A.**
- *Explain the three-way handshake.* SYN, SYN-ACK, ACK to sync sequence numbers and agree on parameters.
- *What is TIME_WAIT for?* Ensures delayed packets from the old connection don't corrupt a new one and that the final ACK is delivered.
- *Why is HTTP/3 over UDP?* To avoid TCP head-of-line blocking and speed up handshakes while implementing reliability per stream in user space.

**Your turn (Q5).** Your service makes 2,000 short HTTP calls/second to one partner API and starts failing with connection errors. Diagnose and fix.

---

### 3.6 What Happens When You Type a URL `[Both]` P0

**What.** The full journey from browser to rendered page; the classic question that tests breadth.

**How it works (and where it breaks).**

```
1. Browser parses URL, checks HSTS list, checks caches
2. DNS: browser cache -> OS cache -> recursive resolver -> root -> TLD -> authoritative
3. TCP handshake to IP (or QUIC for HTTP/3)
4. TLS handshake: certificate validation, key exchange, session keys
5. HTTP request (HTTP/2 or HTTP/3, headers, cookies)
6. CDN/edge: cache hit? else forward to origin
7. Load balancer -> reverse proxy / API gateway -> app instance
8. App: authN/Z -> cache lookup -> DB/queue/other services
9. Response travels back; browser parses HTML, fetches CSS/JS/images, renders
```

**Production problems by hop.**

| Hop | What breaks | How you see it |
|---|---|---|
| DNS | Wrong record, low/high TTL during failover, resolver outage | NXDOMAIN, users in some regions can't reach site; check with `dig` |
| TLS | Expired or mismatched certificate, missing intermediate | browser security warnings, client handshake failures; `openssl s_client` |
| CDN | Stale content, bad cache key, failed purge | users see old version; check cache-status headers |
| Load balancer | No healthy targets, idle-timeout mismatch | 502/503/504 |
| App | Slow handlers, exhausted pools | high p99, 5xx |
| DB/cache | Slow queries, connection exhaustion | latency in traces |

**Interview Q&A.**
- *Where would you add caching in this path?* Browser, CDN, reverse proxy, application cache (Redis), DB buffer cache. Each layer trades freshness for speed.
- *502 vs 504?* 502: upstream returned an invalid response or closed the connection. 504: upstream didn't respond in time.

**Your turn (Q6).** Users in one country report your site is unreachable, but others are fine. List your investigation order.

---

### 3.7 L4 vs L7 Load Balancing `[Both]` P1

**What.** Distributing traffic across servers. **L4** works on TCP/UDP (IP + port). **L7** understands HTTP (paths, headers, cookies).

**Why.** Scale horizontally, remove single points of failure, enable zero-downtime deploys.

**How it works.**
- **Algorithms:** round robin, weighted, least connections, least response time, IP hash, consistent hashing (for cache locality).
- **Health checks** remove bad targets. **L7** adds TLS termination, path/host routing, header rewriting, retries, WAF, auth.
- L4 is faster and simpler; L7 is smarter and costs more CPU per request.

**Alternatives.** DNS round robin (no health awareness, TTL caching), client-side load balancing (gRPC, service mesh sidecars), anycast and global load balancing.

**When / when not.** L4 for raw TCP, extreme throughput, non-HTTP protocols. L7 for HTTP routing, microservices, canary routing.

**Trade-offs.** L7 flexibility vs. extra latency/cost; sticky sessions simplify stateful apps but unbalance load and break on instance loss.

**Production problems.**

*gRPC/HTTP2 imbalance behind an L4 balancer*
- Symptom: one pod hot, others idle after scaling out.
- Root cause: HTTP/2 multiplexes many requests over one long-lived connection, and an L4 balancer balances connections, not requests.
- Fix: L7 balancer or mesh that balances per request, or client-side balancing with periodic connection rotation.

*Health check misconfiguration*
- Symptom: all targets marked unhealthy (global outage) or traffic sent to unready pods.
- Root cause: health check hits a deep dependency (a DB blip then removes every instance), or checks too shallow.
- Fix: separate liveness (process alive) from readiness (can serve), avoid cascading dependency checks.

*Sticky sessions* causing uneven load and session loss on scale-in. Fix: externalize session state (Redis) or use signed tokens.

**Example.** After deploying 10 new pods, the load graph shows 2 pods at 90% CPU and 8 near idle because clients hold persistent HTTP/2 connections opened before the scale-out.

**Common mistakes.** Deep health checks that fail globally. Forgetting the LB itself can be a single point of failure or a bottleneck.

**Interview Q&A.**
- *L4 vs L7?* Connection-level forwarding vs request-aware routing.
- *How do you do blue-green or canary with an LB?* Weighted target groups or header/cookie routing at L7.

**Your turn (Q7).** After scaling a gRPC service from 4 to 12 pods, traffic is still concentrated on the original 4. Why, and how do you fix it?

---

### 3.8 ACID, Transactions, and Isolation Levels `[Both]` P0

**What.** A transaction groups operations into one logical unit. **ACID:** Atomicity (all or nothing), Consistency (invariants preserved), Isolation (concurrent transactions don't corrupt each other), Durability (committed = survives crash).

**Why.** Money transfers, inventory, bookings: partial or interleaved updates produce wrong data.

**How it works.** Atomicity and durability use a **write-ahead log** and undo information. Isolation uses locks and/or MVCC snapshots.

**Anomalies and levels (SQL standard view).**

| Anomaly | Meaning |
|---|---|
| Dirty read | read uncommitted data |
| Non-repeatable read | same row read twice gives different values |
| Phantom | same query returns different set of rows |
| Lost update | two read-modify-write cycles overwrite each other |
| Write skew | two transactions each read overlapping data and update different rows, jointly violating an invariant (e.g., two doctors both go off-call) |

| Level | Dirty | Non-repeatable | Phantom |
|---|---|---|---|
| Read Uncommitted | possible | possible | possible |
| Read Committed | prevented | possible | possible |
| Repeatable Read | prevented | prevented | possible (standard) |
| Serializable | prevented | prevented | prevented |

*Real engines differ from the standard:* PostgreSQL defaults to Read Committed, its Repeatable Read is snapshot isolation (stronger on phantoms, but still allows write skew), and its Serializable uses SSI. MySQL InnoDB defaults to Repeatable Read using gap/next-key locks. Always check the docs for the version you run.

**Alternatives.** Optimistic concurrency (version column), `SELECT ... FOR UPDATE`, atomic updates, application-level idempotency, sagas for cross-service work, MongoDB single-document atomicity (multi-document transactions exist but cost more).

**When / when not.** Use stronger isolation for money and inventory; avoid long or wide transactions, which hurt concurrency.

**Trade-offs.** Higher isolation means more blocking, more aborts and retries, lower throughput.

**Production problems.**

*Overselling inventory (lost update)*
- Symptom: stock goes negative or two buyers get the last item.
- Root cause: read stock, check in app, then write, with interleaving.
- Fix: `UPDATE stock SET qty = qty - 1 WHERE id=? AND qty >= 1` and check affected rows; or row lock; or version check with retry.

*Serialization failures*
- Symptom: occasional "could not serialize access" errors. Fix: application must retry the whole transaction.

*Long-running transactions*
- Symptom: lock waits, replication lag, table bloat. Detect: `pg_stat_activity` oldest `xact_start`. Fix: keep transactions short, no user interaction or external calls inside.

**Example.** Two users reserve the last seat. Without an atomic conditional update, both succeed. With the conditional update, the second affects 0 rows and gets "sold out".

**Common mistakes.** Believing "ACID" means every DB is serializable by default. Doing network calls inside transactions. Not retrying on serialization failures.

**Interview Q&A.**
- *Explain write skew.* Two transactions read overlapping data, update different rows, and together break an invariant; snapshot isolation doesn't prevent it.
- *Pessimistic vs optimistic locking?* Lock first and block others vs. detect conflict at write time via a version and retry; optimistic wins under low contention.

**Your turn (Q8).** Design the DB operations to prevent double-booking of a meeting room under concurrent requests. Mention isolation or locking choice.

---

### 3.9 Indexes: B-Tree vs LSM-Tree `[Both]` P0

**What.** An index is an auxiliary sorted structure that lets the DB find rows without scanning everything.

**Why.** A scan of 100M rows is O(n); a B-tree lookup is O(log n), typically a few page reads.

**How it works.**

*B+Tree (PostgreSQL, MySQL InnoDB, MongoDB WiredTiger):* balanced tree of pages, leaf nodes sorted and linked for range scans; updates modify pages in place (random I/O, page splits). Read-optimized.

*LSM-tree (Cassandra, RocksDB, many time-series stores):* writes go to a WAL and an in-memory memtable, flushed as immutable sorted files (SSTables), merged in background **compaction**. Reads may check several files, helped by Bloom filters. Write-optimized with sequential I/O.

**Compound indexes.** Order matters: an index on `(user_id, created_at)` serves `WHERE user_id=?` and `WHERE user_id=? ORDER BY created_at` but not `WHERE created_at=?` alone (leftmost-prefix rule). MongoDB guideline: **ESR**, Equality fields first, then Sort, then Range. A **covering index** contains all queried columns, avoiding table lookups.

**Selectivity.** An index on a low-cardinality column (e.g., boolean) rarely helps; the planner may prefer a scan.

**Alternatives.** Hash indexes (equality only), full-text/inverted indexes, bitmaps, partial indexes (index only a subset), materialized views, denormalization.

**Trade-offs.** Every index speeds reads and slows writes, uses memory and disk. LSM: higher write throughput, but read and space amplification and compaction stalls; B-tree: predictable reads, more write amplification.

**Production problems.**

*Missing or wrong index*
- Symptom: latency grows as data grows, DB CPU high, slow query log entries.
- Detect: slow query log, `EXPLAIN` shows Seq Scan / `COLLSCAN`, `totalDocsExamined` far above `nReturned`.
- Fix: add the right compound index, build it concurrently/online (`CREATE INDEX CONCURRENTLY` in PostgreSQL; for MongoDB follow the current docs for index builds on busy clusters).

*Index not used*
- Causes: function on the column (`WHERE lower(email)=...`), type mismatch, leading wildcard `LIKE '%x'`, stale statistics, low selectivity.

*Too many indexes*
- Symptom: slow inserts/updates, large storage. Fix: audit unused indexes via index usage stats.

*Compaction stalls (LSM)* causing write latency spikes; monitor pending compactions.

**Example.** Query: `orders WHERE user_id=42 AND status='PAID' ORDER BY created_at DESC LIMIT 20`. Index `(user_id, status, created_at DESC)` serves filter and sort with no extra sort step.

**Common mistakes.** Indexing every column. Wrong column order. Forgetting that indexes cost writes. Testing with tiny data where scans look fast.

**Interview Q&A.**
- *Why B+tree for databases?* High fanout keeps the tree shallow (few disk reads), and linked leaves make range scans efficient.
- *When would you pick LSM storage?* Write-heavy workloads (ingest, time series) where sequential writes matter more than read latency.
- *What is a covering index?* An index containing all fields needed by the query, so no base-table fetch.

**Your turn (Q9).** A query `WHERE tenant_id=? AND created_at > ? ORDER BY created_at` is slow on a 200M-row table. What index would you create and how do you verify it's used?

---

### 3.10 WAL and MVCC `[SDE2]` P1

**What.** **WAL:** write changes to a sequential log and flush it before acknowledging commit. **MVCC:** keep multiple versions of rows so readers see a consistent snapshot without blocking writers.

**Why.** Crash safety without writing every data page synchronously; high read/write concurrency.

**How it works.**
- On crash, the DB replays the WAL to restore committed changes and discards uncommitted ones. The WAL also feeds **replication** and **point-in-time recovery**.
- In PostgreSQL, updates create new tuple versions; old versions remain until **VACUUM** reclaims them. InnoDB keeps old versions in undo logs.

**Alternatives.** Lock-based concurrency only (readers block writers), shadow paging, in-memory databases with snapshots.

**Trade-offs.** MVCC: great concurrency but storage overhead and cleanup work. WAL: durability cost on fsync latency; batching (group commit) helps throughput.

**Production problems.**

*Table/index bloat and slow vacuum (PostgreSQL)*
- Symptom: tables grow though row count is stable, queries slow down.
- Root cause: heavy updates/deletes plus long-running transactions that prevent cleanup of old versions.
- Detect: dead tuple stats, `pg_stat_user_tables`, oldest transaction age.
- Fix: tune autovacuum, kill/avoid long transactions, rewrite bloated tables when necessary.

*WAL disk full*
- Root cause: a replica or replication slot that falls behind forces the primary to retain WAL. Detect: replication slot lag. Fix: remove abandoned slots, fix the lagging consumer, alert on slot lag and disk.

*Transaction ID wraparound (PostgreSQL)*: if freezing doesn't keep up the database can be forced to stop accepting writes. Monitor autovacuum health.

**Example.** A forgotten `BEGIN` in a BI session stays open for days; vacuum can't remove dead rows, table doubles in size, and primary disk fills.

**Common mistakes.** Ignoring idle-in-transaction sessions. Disabling autovacuum "for performance".

**Interview Q&A.**
- *How does a DB recover after a crash?* Replays WAL (redo committed changes), rolls back uncommitted work.
- *How does MVCC let readers not block writers?* Readers read an older consistent version; writers create new ones.

**Your turn (Q10).** Primary DB disk is filling with WAL files though traffic is normal. What are the likely causes and your first three checks?

---

### 3.11 Normalization vs Denormalization `[Both]` P1

**What.** Normalization structures data to remove redundancy (1NF atomic values, 2NF no partial dependency, 3NF no transitive dependency). Denormalization intentionally duplicates data for read speed.

**Why.** Normalized: one place to update, no anomalies. Denormalized: fewer joins, faster reads, fits document/NoSQL models.

**How it works.** Normalized orders schema: `users`, `orders`, `order_items`, `products` joined by keys. Denormalized: copy `product_name` and `price_at_purchase` into `order_items`, which is also the *correct* modeling for history.

**Alternatives.** Materialized views, read models (CQRS), caching, embedding vs referencing in MongoDB.

**When / when not.** Normalize OLTP source-of-truth data by default. Denormalize measured hot read paths, analytics, and data that should be a snapshot in time.

**Trade-offs.** Read speed vs write complexity and consistency risk.

**Production problems.**
- *Update anomalies:* a customer name stored in 5 places updated in 3. Fix: single source of truth plus async propagation, or accept staleness explicitly.
- *Unbounded embedded arrays (MongoDB)* causing huge documents and write amplification. Fix: reference or bucket pattern.
- *Over-normalized reporting queries* with 12 joins timing out. Fix: read model or warehouse.

**Common mistakes.** Denormalizing before measuring. Normalizing analytics data into slow joins.

**Interview Q&A.**
- *When to denormalize?* When a measured read path is too slow and the duplicated data's staleness is acceptable or maintainable.
- *Embed or reference in MongoDB?* Embed when data is read together, bounded in size, and owned by the parent; reference when it's shared, large, or unbounded.

**Your turn (Q11).** Model a comments system for blog posts in MongoDB. Embed or reference? Defend with growth and access patterns.

---

### 3.12 Reading Query Plans (`EXPLAIN`) `[Both]` P0

**What.** `EXPLAIN` shows how the database will (or did, with `ANALYZE`) execute a query.

**How to read it (PostgreSQL-style, illustrative).**

```
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42 ORDER BY created_at DESC LIMIT 20;

Limit  (cost=0.43..8.9 rows=20)  (actual time=0.05..0.12 rows=20 loops=1)
  -> Index Scan Backward using idx_orders_user_created on orders
       Index Cond: (user_id = 42)
Planning Time: 0.2 ms   Execution Time: 0.2 ms
```

What to look for:
- **Scan type:** Seq Scan (full table) vs Index Scan vs Index Only Scan.
- **Join algorithm:** Nested Loop (good with small outer + indexed inner), Hash Join (large unsorted sets), Merge Join (sorted inputs).
- **Estimated vs actual rows:** large mismatch means stale statistics (`ANALYZE`) or correlated columns.
- **Sorts/spills to disk, high loops count, rows removed by filter.**

**MongoDB:** `explain("executionStats")`: `COLLSCAN` vs `IXSCAN`; compare `totalDocsExamined` and `totalKeysExamined` to `nReturned`; an in-memory `SORT` stage suggests the index doesn't cover the sort.

**Production problems.**
- *Plan regression after data growth or stats change:* query was fast for months, suddenly slow. Detect: compare plans, check last analyze time. Fix: refresh stats, add index, hint/rewrite.
- *Parameter-sensitive plans:* fast for one tenant, slow for a huge one.
- *OFFSET pagination on deep pages* scans and discards rows. Fix: keyset pagination (`WHERE id > last_id ORDER BY id LIMIT n`).

**Common mistakes.** Trusting `EXPLAIN` without `ANALYZE` on a representative dataset. Looking only at cost, not actual time and rows. Running `EXPLAIN ANALYZE` on destructive statements in production (it executes them).

**Interview Q&A.**
- *Query slow only in production, fast locally?* Data volume and distribution differ; compare plans, check statistics and missing indexes.
- *Why keyset over OFFSET?* OFFSET cost grows with page depth; keyset uses an index seek.

**Your turn (Q12).** `EXPLAIN ANALYZE` shows estimated rows 10 but actual rows 2,000,000, and the planner chose a nested loop. What happened, and how do you fix it?

---

### 3.13 Memory Management and Garbage Collection `[Both]` P1

*Pick your language (Python, Java, Go, JavaScript, C#) and I'll specialize this section. General principles below.*

**What.** Automatic memory management reclaims objects no longer reachable.

**Why.** Prevents manual `free` bugs (use-after-free, double free), at some runtime cost.

**How it works.**
- **Reference counting** (CPython): count references, free at zero; needs a cycle detector for cycles.
- **Tracing GC** (JVM, Go, .NET, V8): find reachable objects from roots; **generational** collectors exploit that most objects die young. Some phases **stop the world** (pause application threads); modern collectors (G1, ZGC, Go's concurrent GC) reduce pauses.

**Alternatives.** Manual memory (C/C++), ownership and borrowing (Rust), arenas and object pools.

**Trade-offs.** Productivity and safety vs. pauses, memory overhead, and less predictable latency.

**Production problems.**
- *GC pauses causing p99 spikes.* Detect: GC logs, pause metrics. Fix: tune heap, reduce allocation rate, pick a low-pause collector, reuse objects.
- *Memory leaks in managed languages:* unbounded caches/maps, listeners never removed, thread-locals, growing queues. Detect: heap dumps and growth trend. Fix: bounded caches with eviction, weak references where appropriate.
- *Heap vs container limit mismatch* leading to OOMKilled (see 3.2).

**Common mistakes.** Thinking GC languages can't leak. Setting heap equal to container memory. Caches without size limits.

**Interview Q&A.**
- *How can you leak memory in Java/Python?* Objects remain reachable, e.g., static collections, caches without eviction, closures capturing large objects.
- *Why generational GC?* Most objects are short-lived, so collecting the young generation frequently is cheap.

**Your turn (Q13).** Service memory grows steadily until restart every 2 days. How do you confirm a leak and find it?

---

### 3.14 Linux Debugging Toolkit `[Both]` P0

**What.** A small set of tools to answer "what is this machine doing?" using the **USE method** (Utilization, Saturation, Errors per resource).

| Resource | Commands | Look for |
|---|---|---|
| CPU | `top`, `htop`, `mpstat -P ALL 1`, `pidstat 1` | user vs system vs iowait vs steal; run queue |
| Memory | `free -m`, `vmstat 1`, `/proc/meminfo` | swap in/out, available memory, OOM in `dmesg` |
| Disk | `iostat -x 1`, `df -h`, `df -i`, `iotop` | `%util`, await, full disk, inode exhaustion |
| Network | `ss -s`, `ss -tanp`, `sar -n DEV 1`, `tcpdump`, `curl -w`, `dig` | retransmits, TIME_WAIT counts, connection states |
| Files | `lsof -p PID`, `ulimit -n` | fd leaks |
| Processes | `ps aux`, `strace -p`, `perf`, `py-spy`, `jstack` | stuck syscalls, hot functions, blocked threads |
| Logs | `journalctl -u svc`, `dmesg -T` | kernel and OOM messages |

**"Server is slow" checklist.** (1) `uptime` for load. (2) `top`: who uses CPU? Is `wa` (I/O wait) or `st` (steal) high? (3) `free -m` / `vmstat`: swapping? (4) `iostat -x`: disk saturated? (5) `ss -s`: connection pile-up? (6) app logs and thread dump. (7) What changed recently?

**Production problems.**
- *Disk full (data or inodes)* stops writes and logging. Detect with `df -h` and `df -i`. Fix: rotate/compress logs, delete safely, add retention policy and alerts at 80%.
- *High iowait* from a saturated disk: check `iostat`, find the heavy process with `iotop`, consider IOPS limits of cloud volumes.
- *CPU steal* on shared VMs (noisy neighbor): move instance type or host.

**Common mistakes.** Guessing instead of measuring. Looking only at averages. Restarting before capturing a thread dump or heap dump.

**Interview Q&A.**
- *Server CPU is low but the app is slow. Why?* Waiting on I/O, locks, or downstream calls; check iowait, thread states, pool usage, and traces.
- *How do you find which process is using a port?* `ss -ltnp` or `lsof -i :PORT`.

**Your turn (Q14).** A host shows load average 40 on 8 cores, but `%us` is only 10%. What do you suspect and which commands do you run?

---

## 4. Part 1 Review

### Summary

You now have the system map, the glossary, and the foundations: concurrency (threads, locks, I/O models), memory, networking (TCP, DNS/TLS journey, load balancing), and database internals (transactions, indexes, WAL/MVCC, plans). The recurring production themes: **unbounded resources** (threads, connections, memory, queues), **missing timeouts**, **long-lived transactions/connections**, and **measure before guessing**.

### Five review questions (Answer Key below)

- **R1.** Why can adding more threads reduce throughput?
- **R2.** Why does reusing HTTP connections reduce latency and failures?
- **R3.** Why can a deep health check cause a bigger outage than the failure it detects?
- **R4.** What is the difference between a lost update and write skew?
- **R5.** Why can an idle open transaction cause disk and performance problems in PostgreSQL?

### Mock question (answer it yourself first)

**M1.** "Your API's p99 latency jumped from 200 ms to 6 s at 2 PM. CPU is 20%. Database CPU is normal. Walk me through diagnosis and mitigation." Cover: what you check first, how you isolate the layer, at least four hypotheses, and what you'd do in the first 15 minutes.

---

## 5. Answer Key

*Model answers are concise. In a real interview, add examples and trade-offs from your own experience.*

**Q1.** Needed concurrency = 300 RPS x 1 s = 300 threads, but the pool has 100, so requests queue, latency grows well beyond 1 s, and timeouts and 503s follow even though CPU is low (before the change: 300 x 0.1 = 30 threads, comfortable). Changes: (1) strict timeouts on the dependency call and fail fast; (2) circuit breaker/bulkhead so one slow dependency can't consume the whole pool, plus fallback or cached responses; (3) async/non-blocking client or a larger bounded pool plus load shedding and autoscaling driven by queue depth/latency, not CPU.

**Q2.** Redis `fork`s for the snapshot; copy-on-write duplicates every page modified while the child writes. Under heavy writes, memory can approach 2x the dataset plus overhead, exceeding 12 GB, so the OOM killer fires. Fixes: provision more RAM headroom (leave substantial room for fork), schedule snapshots at low-write times or use replicas to snapshot, reduce dataset/eviction policy, use AOF or a different persistence strategy, and monitor `used_memory` and fork latency.

**Q3.** Bug: `transfer(A,B)` locks `from` then `to`; a concurrent `transfer(B,A)` locks in the opposite order, so each holds one lock and waits for the other. Fix: acquire locks in a global order, e.g., always lock the lower account id first, then the higher; alternatively use `tryLock` with timeout and retry, or do the transfer as a single atomic DB transaction with ordered row locking.

**Q4.** Suspect something blocking the event loop (sync I/O, CPU-heavy parsing, regex backtracking, a sync library like `requests`) or GC. Steps: check event-loop lag, take `py-spy dump`/profile during a freeze, enable asyncio debug mode (slow callback warnings), correlate freezes with specific endpoints or payload sizes, check for cron/background tasks, then move blocking work to executors or async libraries.

**Q5.** 2,000 new connections/sec exhaust ephemeral ports (TIME_WAIT ~60 s holds ports) and may hit partner rate limits or SYN issues; also TLS handshake CPU. Confirm with `ss -s` and TIME_WAIT counts. Fix: reuse a long-lived HTTP client with keep-alive and connection pooling (HTTP/2 multiplexing if supported), bounded concurrency, retries with backoff and jitter. Avoid relying on kernel tuning as the main fix.

**Q6.** Check: reproduce from that region (external probes); DNS resolution from there (`dig` against local resolvers, geo-DNS records, DNSSEC issues); CDN/edge POP health and routing; ISP/BGP or government blocking; TLS/cert chain issues from that region; WAF/geo rules accidentally blocking; recent changes. Mitigate by rerouting/failing over the region's edge or fixing the DNS/WAF rule.

**Q7.** gRPC uses long-lived HTTP/2 connections; an L4 balancer distributes connections not requests, and existing clients keep their old connections, so new pods get nothing. Fixes: L7/gRPC-aware balancer or service mesh (per-request balancing), client-side load balancing with service discovery, set max connection age so clients reconnect, and use headless service discovery with round_robin.

**Q8.** Table `bookings(room_id, start, end)`. Options: (a) transaction that locks the room row (`SELECT ... FOR UPDATE`), checks for overlapping bookings, inserts; (b) database exclusion constraint on overlapping time ranges (PostgreSQL `EXCLUDE USING gist` on a range type) so the DB enforces it; (c) optimistic approach with retry. The constraint approach is the most robust because the invariant is enforced in the database regardless of app bugs; overlap checks in app code alone are vulnerable to write skew under snapshot isolation.

**Q9.** Index `(tenant_id, created_at)` (equality first, then range/sort, per the ESR guideline), created online/concurrently. Verify with `EXPLAIN (ANALYZE, BUFFERS)`: an index scan using the new index, no separate sort, low rows removed by filter, and `totalDocsExamined` close to `nReturned` in MongoDB. Also check index size, write overhead, and test on production-like data.

**Q10.** Likely: an inactive or lagging replication slot (replica down, CDC consumer stalled), WAL archiving failing, long-running transaction/checkpoint settings, or a bulk write job. Checks: replication slot lag (`pg_replication_slots`), archive status/errors in logs, long transactions in `pg_stat_activity`. Then remove abandoned slots or fix the consumer; alert on slot lag and disk usage.

**Q11.** Reference (separate `comments` collection with `post_id`, indexed by `(post_id, created_at)`), because comments are unbounded and can reach thousands, an embedded array would grow the post document toward the 16 MB limit and cause write amplification. Optionally embed the last few comments or a comment count in the post for fast display (the subset/computed pattern), and paginate the rest.

**Q12.** The planner's statistics were wrong (stale stats, or correlated columns/skew), so it believed only ~10 rows would match and chose a nested loop, which is catastrophic for 2M rows. Fixes: run `ANALYZE`, increase statistics target or create extended statistics for correlated columns, add/adjust an index, rewrite the query, and as a last resort use planner hints/settings.

**Q13.** Confirm: plot heap/RSS over time and GC behavior (sawtooth with rising floor = leak); check whether it's heap vs native memory. Find: take heap dumps/snapshots at two times and diff (`jmap`/MAT, `tracemalloc`, Chrome heap snapshot), look for growing collections and retained references. Typical culprits: unbounded caches, listeners/subscriptions, thread-locals, per-request data stored globally. Fix with bounded caches/eviction, and add a memory alert and a regression test under load.

**Q14.** High load with low user CPU suggests many threads blocked on I/O or locks (uninterruptible "D" state), disk saturation, NFS/network storage stalls, swap thrashing, or high `iowait`/`steal`. Run `top` (check `wa`, `st`), `vmstat 1`, `iostat -x 1`, `ps -eo state,pid,cmd | grep ' D'`, `dmesg -T`, and `ss -s`.

**R1.** Past the point of useful parallelism, more threads add context-switch overhead, cache/TLB pollution, lock contention, and memory use, so total throughput falls.

**R2.** It avoids repeated TCP/TLS handshakes (saving round trips and CPU), skips slow-start on new connections, and avoids ephemeral port exhaustion/TIME_WAIT buildup, which removes a class of connection errors.

**R3.** If a health check calls a shared dependency, a brief DB blip makes every instance report unhealthy, so the load balancer removes them all and a partial degradation becomes a total outage. Keep liveness shallow; make readiness reflect ability to serve; degrade gracefully.

**R4.** Lost update: two transactions read the same row and the later write overwrites the earlier (same data item). Write skew: two transactions read overlapping data and update *different* rows, jointly violating an invariant; snapshot isolation can allow it, serializable isolation or explicit constraints/locks prevent it.

**R5.** The open transaction pins an old snapshot, so vacuum can't remove dead row versions (bloat), may hold locks that block other work, and can block WAL/replication cleanup, causing disk growth and slower queries.

**M1 (outline of a strong answer).** (1) Confirm scope and start time: which endpoints, regions, customers; what changed at ~2 PM (deploy, config, flag, traffic, cert, cron, dependency). (2) Use traces/metrics to find where time is spent: client, LB, app queue, outbound calls, DB, cache. (3) Hypotheses with low CPU: thread/connection pool exhaustion, slow third-party or internal dependency, lock contention/deadlock, DNS or network latency/retransmits, GC pauses or swap, cache stampede/miss storm, replica lag or connection waits on DB side (DB CPU normal doesn't mean no waits). (4) Check: pool saturation and queue depth, thread dump, outbound latency by dependency, `ss -s`, GC logs, DB active/waiting sessions and lock waits. (5) First 15 minutes: roll back the latest change or flip the flag; add timeouts or fail fast; shed load or scale out if pool-bound; fail over or bypass the slow dependency; communicate status. (6) Afterward: root cause with evidence, add timeouts, bulkheads, alerts on saturation, and a postmortem.

---

### Next: Part 2 of 13

**Part 2 covers:** DSA patterns (Section 19) in parallel with SQL practice (Section 23). Reply **"continue"** to receive it, and tell me your language (Python, Java, C++, JavaScript, or Go) and your target (SDE1, SDE2, or both) so I can tailor the code and depth.
