# Principal Engineer's Guide to System Design, Cloud Architecture, Cost Engineering, Production Systems, and SDE1/SDE2 Interview Preparation

Act as a Principal Software Architect, Staff Engineer, Distributed Systems Specialist, Database Architect, and Cloud FinOps Engineer.

I want to develop a deep understanding of how real-world production systems are designed, deployed, scaled, optimized, and operated.

Do not focus on only one technology or one architectural decision. Build a comprehensive engineering analysis covering the concepts below and explain how they interact with each other.

Use practical examples from backend services, microservices, databases, cloud platforms, SaaS products, and AI applications.

---

## 0. Universal Explanation Rules (Apply to Every Concept, Section, and Track)

**Target level.** At the start, ask me whether I am targeting SDE1, SDE2, or both. Tag every topic as `[SDE1]`, `[SDE2]`, or `[Both]`, and with a priority: `P0` (must know), `P1` (should know), `P2` (nice to have). Calibrate depth to the level.

**Mandatory template.** For EVERY concept, technology, pattern, or algorithm, cover all of these fields. Do not skip any:

1. **What it is**: one-line definition.
2. **What problem it solves and why it exists**: the pain that led to its invention.
3. **How it works internally**: the mechanism (data structures, protocol steps, algorithms, memory/disk behavior), not just a definition.
4. **Alternatives**: at least 2-3, and when each is the better choice.
5. **When to use it and when NOT to use it.**
6. **Trade-offs**: performance, cost, complexity, consistency, operational burden.
7. **Production problems**: what actually goes wrong in real systems (use the production problem template below).
8. **A realistic production example**: with numbers (clearly labeled as illustrative).
9. **Common mistakes and misconceptions.**
10. **Likely interview questions and follow-ups**: each with a concise answer.
11. **One question for me to answer myself**: do not reveal the answer until I reply.

**Production problem template.** For every production problem, use this structure:

- **Symptom**: what users and dashboards show.
- **Root cause**: why it happens technically.
- **How to detect**: exact metrics, logs, traces, queries, or commands.
- **Immediate mitigation**: what to do in the first 15 minutes (rollback, scale, shed load, flag off, failover).
- **Permanent fix**: the engineering change.
- **Prevention**: design choices, tests, alerts, and limits that stop recurrence.
- **Interview angle**: how an interviewer would probe this.

**Three explanation levels.** Explain each concept as: (1) Beginner: what it means; (2) Engineer: how it works technically; (3) Architect: how it affects system-wide trade-offs.

**Reasoning rules.**

- Teach reasoning, not memorization. Every recommendation must include "because...".
- Show what breaks when the wrong choice is made.
- Use diagrams (ASCII or Mermaid), real commands, real queries, and real configuration snippets.
- Use worked numeric examples and label them as illustrative unless they come from official documentation.
- Never present a single option as universally best.

---

## 1. Database Architecture and Deployment

Compare and explain:

- Serverless vs dedicated database infrastructure
- Shared vs isolated infrastructure
- Managed vs self-hosted databases
- SQL vs NoSQL
- MongoDB vs PostgreSQL vs Redis vs Cassandra
- MongoDB Atlas vs Azure Cosmos DB deployment models
- Provisioned capacity vs autoscaling vs consumption-based pricing
- Read replicas vs replication vs sharding
- Horizontal vs vertical scaling
- Single-region vs multi-region databases
- Strong consistency vs eventual consistency
- Transactions, isolation levels, and concurrency control
- Indexing, compound indexes, query execution plans, and index selectivity
- Connection pooling and connection limits
- Database partitioning and shard-key selection
- Hot partitions and uneven workloads
- Caching, materialized views, and read optimization
- Data modeling and denormalization
- Database migrations and schema evolution
- OLTP vs OLAP and analytical workloads
- Data archival, retention, and deletion policies

Explain the benefits, drawbacks, failure modes, cost implications, and appropriate use cases for each concept.

### Production Problems to Cover in This Section

Apply the production problem template (symptom, root cause, detection, mitigation, permanent fix, prevention, interview angle) to each:

- Slow queries caused by a missing or unused index, low-selectivity index, or wrong compound index order
- Full collection or table scans after data growth
- Connection pool exhaustion and "too many connections" errors, especially when app pods autoscale
- Hot partition or hot shard from a poor shard key (monotonically increasing keys, low cardinality)
- Replication lag causing stale reads or "I just saved it but can't see it" bugs
- Failover causing a connection storm, dropped writes, or a stuck primary election
- Long-running transactions causing lock contention, deadlocks, and table bloat (MVCC vacuum issues in PostgreSQL)
- Schema migrations that lock tables or break running application versions
- N+1 query patterns and unbounded result sets
- Unbounded document or array growth (MongoDB 16 MB document limit)
- Serverless or RU-based throttling (429 errors) and surprise cost spikes
- Backups that exist but were never restore-tested
- Disk full, index bloat, and write amplification
- Split brain and lost writes after network partitions
- Noisy neighbor effects on shared infrastructure
- Analytical queries hurting OLTP performance
- Data retention failures causing unbounded storage growth

---

## 2. Cloud Infrastructure and Compute

Compare:

- Virtual machines vs containers vs serverless functions
- Kubernetes vs managed container services vs serverless compute
- Dedicated hosts vs shared virtual machines
- CPU-based vs memory-based capacity planning
- Requests and limits in Kubernetes
- Autoscaling, cluster scaling, and pod scaling
- Horizontal Pod Autoscaler vs Vertical Pod Autoscaler
- Scale-to-zero vs always-on services
- Cold starts and startup latency
- Spot instances vs reserved capacity vs on-demand pricing
- Managed services vs self-managed infrastructure
- Regional infrastructure vs availability zones
- Infrastructure as Code
- Deployment environments and environment isolation
- Resource quotas and capacity limits

Explain when each model is appropriate and how compute choices influence database performance and total cloud costs.

### Production Problems to Cover in This Section

Apply the production problem template to each:

- OOMKilled pods from memory limits set too low or memory leaks
- CPU throttling from low CPU limits causing latency spikes despite low average CPU
- CrashLoopBackOff from bad config, missing secrets, or failing dependencies at startup
- Pods stuck Pending from quota, node capacity, or affinity rules
- HPA flapping (scale up/down oscillation) and scaling lag versus traffic spikes
- Autoscaled pods overwhelming the database with new connections
- Cold starts on serverless causing p99 latency spikes
- Spot instance interruptions killing workloads without graceful shutdown
- Readiness and liveness probe misconfiguration causing restart loops or traffic to unready pods
- Node pressure, eviction, and noisy neighbors
- Image pull failures, registry rate limits, and huge images slowing deploys
- DNS resolution failures and CoreDNS overload inside clusters
- Ephemeral storage exhaustion and log volume filling disks
- Over-provisioning (idle cost) versus under-provisioning (outages)
- Configuration drift between environments
- Serverless concurrency limits and throttling

---

## 3. Cloud Cost Engineering and FinOps

Analyze:

- Fixed costs vs variable costs
- Pay-per-request vs provisioned capacity
- Cost per API request, customer, transaction, and workload
- Cost allocation by service, team, environment, and tenant
- Compute, database, storage, networking, and observability costs
- Data transfer, internet egress, cross-region traffic, and inter-service communication
- Idle resources and overprovisioning
- Autoscaling efficiency
- Reserved capacity and commitment discounts
- Cost anomaly detection and budget alerts
- Cost attribution for shared infrastructure
- Total cost of ownership (TCO)
- Unit economics and marginal cost
- Cost-performance optimization
- FinOps dashboards and chargeback/showback
- Development, staging, and production cost optimization
- The cost of reliability and high availability

For every major architectural choice, answer:

1. What does it cost?
2. Which usage patterns make it economical?
3. At what workload does the choice become inefficient?
4. What are the hidden costs?
5. How should the cost be measured?
6. What alternative architecture could reduce the cost?

### Production Problems to Cover in This Section

Apply the production problem template to each:

- Surprise bills from internet egress, cross-AZ, and cross-region traffic
- NAT gateway charges from high-volume outbound traffic
- Log and metric ingestion costs growing faster than traffic (high-cardinality metrics, debug logs left on)
- Idle development and staging environments running 24/7
- Orphaned resources (volumes, snapshots, load balancers, IPs)
- Unbounded autoscaling during attacks or retry storms
- Runaway LLM or agent spend from loops and oversized prompts
- Untagged resources making cost attribution impossible
- Reserved capacity or commitments bought and then unused after an architecture change
- Shared-infrastructure cost allocation disputes between teams or tenants
- Over-engineered high availability that exceeds actual business need
- Cost anomalies detected only at month end

---

## 4. Scalability and Distributed Systems

Cover:

- Load balancing and traffic distribution
- Horizontal and vertical scaling
- Stateless vs stateful services
- Service discovery
- API gateways
- Synchronous vs asynchronous communication
- Message queues and event streaming
- Kafka vs RabbitMQ vs cloud-native queues
- Event-driven architecture
- Eventual consistency
- Distributed transactions
- Saga patterns and compensating actions
- Idempotency
- Retries, exponential backoff, and jitter
- Timeouts and deadlines
- Circuit breakers
- Bulkheads
- Backpressure and load shedding
- Rate limiting and quotas
- At-least-once vs at-most-once vs effectively-once processing
- Duplicate message handling
- Ordering guarantees
- Distributed locks and leader election
- Cache invalidation
- CAP theorem and its practical limitations
- Consensus and coordination
- Service degradation during overload

Use concrete request flows and failure scenarios to explain these concepts.

### Production Problems to Cover in This Section

Apply the production problem template to each:

- Retry storms amplifying a partial outage into a full outage
- Thundering herd after a cache expiry, restart, or recovery
- Cascading failure from one slow dependency exhausting threads or connections
- Duplicate processing and duplicate side effects (double charge, double email) from at-least-once delivery
- Poison messages blocking a queue or consumer
- Consumer lag and queue backlog growth
- Out-of-order events and stale overwrites
- Lost updates and write skew from missing concurrency control
- Distributed lock expiry while the holder is still working (lock safety, fencing tokens)
- Split brain and leader election flapping
- Saga partial failure leaving inconsistent state
- Clock skew breaking TTLs, ordering, and token validation
- Hot keys and uneven partition load in Kafka or sharded systems
- Missing timeouts causing hung requests
- Missing backpressure causing memory exhaustion
- Misconfigured rate limiters (too strict or ineffective) and rate limiter bypass

---

## 5. System Design Patterns and Architectural Decisions

Explain:

- Monolith vs modular monolith vs microservices
- Service-oriented architecture
- Domain-driven design
- Bounded contexts
- Hexagonal architecture and clean architecture
- Layered architecture
- CQRS
- Event sourcing
- Strangler migration pattern
- Backend for Frontend
- API composition
- Database per service vs shared database
- Synchronous orchestration vs choreography
- Control plane vs data plane
- Multi-tenant vs single-tenant architecture
- Cell-based architecture
- Plugin-based architecture
- Feature flags
- Configuration management
- Service contracts and backward compatibility
- Architecture Decision Records (ADRs)

For each pattern, explain the problem it solves, its operational complexity, cost implications, alternatives, and when it is unnecessary.

### Production Problems to Cover in This Section

Apply the production problem template to each:

- Distributed monolith: microservices that must deploy together
- Chatty services and latency amplification from deep synchronous call chains
- Shared database coupling between services
- Data inconsistency across services without a reconciliation strategy
- Event schema changes breaking consumers
- Saga and orchestration complexity out of control
- Premature microservices causing operational overload for a small team
- Noisy tenant degrading other tenants in multi-tenant systems
- Feature flag debt and flags that can't be safely removed
- Configuration drift and unsafe config changes
- Breaking API changes and missing backward compatibility
- Over-abstraction and speculative generality

---

## 6. Networking and Communication

Cover:

- DNS and service discovery
- TCP vs UDP
- HTTP/1.1 vs HTTP/2 vs HTTP/3
- REST vs GraphQL vs gRPC
- WebSockets vs Server-Sent Events vs polling
- TLS and certificate management
- Reverse proxies and ingress controllers
- CDN and edge caching
- VPCs, subnets, routing, and NAT
- Private endpoints and network isolation
- Network latency and connection reuse
- Cross-region communication
- API gateways and service meshes
- Bandwidth costs and egress optimization
- Authentication vs authorization
- OAuth 2.0, OpenID Connect, JWT, and API keys

Explain how networking choices affect latency, security, availability, scalability, and cost.

### Production Problems to Cover in This Section

Apply the production problem template to each:

- Expired TLS certificates causing outages
- DNS caching and TTL problems during failover
- Load balancer 502/503/504 errors and idle-timeout mismatches between LB, proxy, and app
- Connection resets from keep-alive mismatches
- NAT port exhaustion (SNAT) under high outbound connection rates
- Missing connection reuse causing latency and CPU waste from repeated TLS handshakes
- HTTP/2 head-of-line blocking and gRPC load balancing imbalance (long-lived connections pinned to one pod)
- WebSocket scaling issues: sticky sessions, connection limits, reconnect storms
- CORS misconfiguration
- JWT expiry, clock skew, revocation gaps, and leaked tokens
- OAuth misconfiguration (redirect URI, scope, token storage)
- Cross-region latency and egress surprise
- MTU and fragmentation issues
- Service mesh sidecar overhead and misconfiguration

---

## 7. Storage, Caching, and Data Processing

Analyze:

- Object vs block vs file storage
- Hot, warm, and cold storage
- Data lakes and data warehouses
- Redis and in-memory caching
- Cache-aside, write-through, and write-behind
- TTL and eviction strategies
- Cache stampedes and cache penetration
- CDN caching
- Batch vs stream processing
- ETL vs ELT
- Change data capture
- Data replication and synchronization
- Data compression and serialization
- Retention and lifecycle policies
- Backup, snapshots, and point-in-time recovery

Explain when caching improves performance and when it introduces consistency problems or additional cost.

### Production Problems to Cover in This Section

Apply the production problem template to each:

- Cache stampede after expiry or restart
- Cache penetration (queries for keys that never exist) and cache poisoning
- Stale cache from failed invalidation and race conditions between writes and cache fills
- Hot keys overloading a single Redis node
- Eviction of hot data and memory pressure in Redis
- Cold cache after deploy or failover causing database overload
- Write-behind data loss on crash
- CDN serving stale or wrong content, failed purges, and cache key mistakes
- Redis persistence and failover data loss
- Large keys and large objects blocking Redis or networks
- CDC lag and replication delay in pipelines
- Small-files problem in data lakes
- Backups and snapshots that cannot be restored
- Serialization and schema evolution breaking consumers
- Lifecycle policies deleting data that was still needed

---

## 8. Reliability, Availability, and Disaster Recovery

Cover:

- Availability, reliability, and durability
- SLI, SLO, and SLA
- Error budgets
- Mean time to detect and recover
- Single points of failure
- Redundancy and failover
- Active-active vs active-passive deployments
- Multi-zone vs multi-region architecture
- RTO and RPO
- Backup testing and disaster recovery drills
- Health checks and readiness probes
- Graceful shutdown
- Rolling, blue-green, and canary deployments
- Chaos engineering
- Incident response and postmortems
- Fault isolation and blast radius
- Dependency failure and cascading failures
- Retry storms and overload protection

Show how reliability requirements change the infrastructure design and monthly budget.

### Production Problems to Cover in This Section

Apply the production problem template to each:

- Failover that was never tested and fails when needed
- Hidden single points of failure (DNS, auth service, shared config, one AZ)
- Health check flapping and false failovers
- Cascading failures and retry storms during recovery
- Thundering herd when a failed dependency returns
- Bad deployments reaching all users because there was no canary
- Split brain after failover
- Backup corruption or backups stored in the same failure domain
- Outdated disaster recovery runbooks and missing access during an emergency
- Unrealistic SLOs, and SLIs that don't reflect user experience
- Alert silence (no alert fired) versus alert fatigue (alerts ignored)
- Blast radius too large because of shared resources
- Multi-region failover with replication lag causing data loss beyond the stated RPO

---

## 9. Observability and Performance Engineering

Explain:

- Logs, metrics, and distributed traces
- OpenTelemetry
- Request IDs and correlation IDs
- RED and USE monitoring methods
- Throughput, concurrency, and saturation
- p50, p95, and p99 latency
- CPU, memory, disk, and network utilization
- Database slow-query analysis
- Connection pool monitoring
- Application profiling
- Distributed tracing
- Synthetic monitoring
- Load testing, stress testing, and soak testing
- Capacity planning
- Performance regression testing
- Alert fatigue and actionable alerts
- Cost observability

Explain how to distinguish an application bottleneck from a database, network, infrastructure, or third-party dependency bottleneck.

### Production Problems to Cover in This Section

Apply the production problem template to each:

- Missing correlation IDs making cross-service debugging impossible
- High-cardinality metrics exploding cost and crashing the metrics backend
- Log volume cost and log pipeline backpressure
- Sampled-out traces missing the exact failing request
- Averages hiding tail latency (always look at p95/p99)
- No saturation metrics (queue depth, pool utilization, thread utilization)
- Dashboards that show system health but not customer impact
- Alert fatigue from noisy, non-actionable alerts
- Time skew across hosts making log correlation unreliable
- Observability stack outage during the very incident you need it for
- Misattributing a database, network, or third-party slowdown to the application

---

## 10. Security and Governance

Cover:

- Authentication and authorization
- Least privilege and RBAC
- Secrets management and key rotation
- Encryption at rest and in transit
- Network segmentation
- Threat modeling
- Rate limiting and abuse prevention
- Audit logs
- Data classification and privacy
- Tenant isolation
- Vulnerability management
- Dependency and container scanning
- Supply-chain security
- Compliance requirements
- Backup security
- Disaster recovery access controls

Discuss the cost and operational trade-offs of security controls without recommending unsafe shortcuts.

### Production Problems to Cover in This Section

Apply the production problem template to each:

- Secrets committed to git or leaked in logs and CI output
- Over-privileged IAM roles and long-lived credentials
- Public storage buckets and exposed databases
- Broken object-level authorization (IDOR) and tenant data leaks
- SQL, NoSQL, and command injection
- SSRF reaching cloud metadata endpoints
- Unpatched dependencies and vulnerable container images
- Supply-chain compromise via malicious or typosquatted packages
- Tokens that are not revoked after logout or compromise
- PII in logs, traces, and analytics
- Rate limit bypass and credential stuffing
- Expired or unrotated keys and certificates
- Backup data accessible to too many people

---

## 11. CI/CD and Production Operations

Explain:

- Continuous integration and continuous delivery
- Automated testing and quality gates
- Container images and registries
- Kubernetes deployment strategies
- Database migration strategies
- Immutable infrastructure
- GitOps
- Environment promotion
- Rollback vs roll-forward
- Progressive delivery
- Configuration drift
- Release observability
- Infrastructure drift detection
- Dependency management
- Production change management
- Operational runbooks

Show how to design deployments that minimize downtime and prevent application/database version incompatibilities.

### Production Problems to Cover in This Section

Apply the production problem template to each:

- Database migrations incompatible with the old app version during rolling deploys (use expand/contract)
- Rollback impossible because of a destructive schema change
- Configuration drift and "works in staging, fails in prod"
- Flaky tests ignored until they hide a real failure
- Secrets exposed in pipeline logs
- Deploys without canary, health gates, or automated rollback
- Mutable image tags (`latest`) causing unreproducible deploys
- GitOps drift and manual hotfixes that get overwritten
- Dependency conflicts and unpinned versions breaking builds
- Long-lived feature branches and big-bang releases
- Deploys during peak traffic without a change-management process
- Missing release markers in observability, making "what changed?" hard to answer

---

## 12. AI Application and Agent Infrastructure

Cover:

- LLM inference cost and token economics
- Hosted APIs vs self-hosted models
- GPU vs CPU inference
- Synchronous vs asynchronous inference
- Streaming responses
- Semantic caching
- Prompt caching
- Retrieval-augmented generation
- Vector databases and hybrid search
- Embedding generation and storage costs
- Model routing
- Rate limits and concurrency
- Queues for long-running AI jobs
- Agent orchestration
- Tool calling and MCP
- Context management
- Evaluation and observability
- Model fallbacks and provider failures
- Cost per user request and successful task
- Agent retries and runaway execution
- Isolation and permissions for tool execution

Explain how traditional distributed-systems principles apply to AI-powered products.

### Production Problems to Cover in This Section

Apply the production problem template to each:

- Runaway agent loops and repeated tool calls burning tokens
- Token cost blowups from oversized context, long histories, and unbounded outputs
- Context window overflow and truncation losing critical instructions
- Hallucinated tool calls, wrong arguments, and malformed JSON
- Prompt injection through retrieved documents, web pages, emails, or tool outputs
- Over-privileged tools and agents able to take destructive actions
- Provider rate limits (429), outages, and latency spikes without fallback
- Slow time-to-first-token and streaming disconnects
- Stale or poisoned RAG indexes and embedding model drift requiring re-indexing
- Retrieval returning irrelevant or unauthorized documents (access control leaks)
- Semantic cache returning wrong answers for near-duplicate queries
- Non-deterministic outputs breaking tests and evaluations
- Long-running jobs timing out without queues and checkpoints
- Idempotency failures when agent steps are retried (duplicate emails, tickets, payments)
- No evaluation or regression testing after prompt or model changes

---

## 13. Mandatory Architecture Case Studies

Use the concepts above to analyze these decisions in depth:

1. MongoDB Serverless vs Dedicated.
2. Azure Cosmos DB RU-based serverless vs provisioned throughput/autoscale vs vCore-based deployments.
3. PostgreSQL vs MongoDB for a growing SaaS product.
4. Redis caching vs direct database reads.
5. Monolith vs microservices.
6. Kubernetes vs managed serverless compute.
7. Synchronous APIs vs queues and asynchronous processing.
8. Single-region vs multi-region deployment.
9. Managed cloud services vs self-hosted infrastructure.
10. Hosted LLM APIs vs self-hosted models.

For each case study, provide:

- Requirements and assumptions
- Architecture diagram
- Request or data flow
- Options and alternatives
- Advantages and disadvantages
- Monthly cost model
- Performance and scalability implications
- Reliability and security considerations
- Operational burden
- Migration complexity
- Failure scenarios
- Decision matrix
- Recommendation and the conditions that would change it

Do not assume one solution is universally best.

---

## 14. Quantitative Engineering

Use formulas and worked examples wherever possible.

Include:

- Break-even analysis
- Cost per request
- Cost per successful transaction
- Requests per second and concurrency
- Little's Law
- Capacity and utilization calculations
- Latency budgets
- Storage growth projections
- Bandwidth calculations
- Availability calculations
- Replication and redundancy costs
- Cache hit-rate calculations
- Queue backlog and processing capacity
- Database connection capacity
- TCO over 1, 3, and 5 years

Clearly distinguish illustrative numbers from actual provider prices and measured production metrics.

---

## 15. Engineering Decision Framework

For every important decision, use this process:

1. Define the problem.
2. Identify functional and non-functional requirements.
3. Establish workload assumptions.
4. Identify constraints and failure scenarios.
5. Generate at least three viable alternatives when appropriate.
6. Estimate costs and operational complexity.
7. Compare performance, scalability, reliability, and security.
8. Identify trade-offs and hidden risks.
9. Recommend the simplest architecture that meets requirements.
10. Define measurable validation criteria.
11. Define the conditions that would justify changing the decision.

Do not recommend complex infrastructure merely because it is popular or technically impressive.

---

## 16. Expected Deliverables

Produce:

- A system design concept map
- A comparison matrix of architectural choices
- A glossary of essential engineering terms
- Architecture diagrams
- Request-flow and data-flow diagrams
- Cost calculation examples
- Decision trees
- Failure-mode analyses
- Practical production scenarios
- A list of monitoring metrics
- A set of architecture review questions
- A prioritized learning roadmap from fundamentals to advanced topics

Use official provider documentation for current product features, limits, and pricing. Cite sources and state the date of pricing information.

Explain each concept at three levels:

1. Beginner: what it means.
2. Engineer: how it works technically.
3. Architect: how it affects system-wide trade-offs.

Start with the concept map and the most important foundational concepts. Then proceed systematically through the full subject. Do not reduce this to a short overview; divide the work into manageable parts while preserving depth.

---

## 17. Interview Preparation Mode

In addition to teaching system design, prepare me for real software engineering interviews at product companies and technology-driven organizations.

Cover these four tracks separately.

### Track A: Data Structures and Algorithms

- Arrays, strings, hash maps, linked lists
- Stacks, queues, trees, graphs, heaps
- Binary search, sorting, recursion, backtracking
- Dynamic programming and greedy algorithms
- Time and space complexity
- Problem-solving patterns and edge cases

### Track B: Low-Level Design (LLD)

- OOP principles and SOLID
- Composition vs inheritance
- Design patterns and their practical use
- UML and class diagrams
- API and interface design
- Extensibility, testability, and maintainability
- Design exercises: parking lot, notification service, payment system, elevator, and rate limiter

### Track C: High-Level Design (HLD)

Teach me to solve problems such as:

- Design URL shortener
- Design WhatsApp or a chat system
- Design YouTube or a video platform
- Design a notification system
- Design a rate limiter
- Design an e-commerce platform
- Design a file storage service
- Design a job scheduler
- Design a distributed cache
- Design a multi-tenant SaaS platform
- Design an AI agent platform

For every problem, follow this interview sequence:

1. Clarify functional requirements.
2. Establish non-functional requirements.
3. Estimate traffic, storage, and bandwidth.
4. Define APIs and data models.
5. Draw the high-level architecture.
6. Explain the request and data flows.
7. Identify bottlenecks and failure scenarios.
8. Compare alternative designs.
9. Estimate infrastructure costs.
10. Explain scaling, security, observability, and reliability.
11. Identify trade-offs and future improvements.

### Track D: Database and Infrastructure Interviews

Prepare me to answer questions about:

- SQL vs NoSQL
- MongoDB Serverless vs Dedicated
- PostgreSQL vs MongoDB
- Indexes and query plans
- Replication, sharding, and partitioning
- Caching and invalidation
- Message queues and event-driven architecture
- Kubernetes and autoscaling
- CAP theorem and consistency
- Distributed transactions and idempotency
- Cloud cost optimization
- Incident debugging and production bottlenecks

For each question, provide:

- A concise interview answer.
- A deeper technical explanation.
- A practical production example.
- Common follow-up questions.
- Trade-offs and common mistakes.
- A mock interview question that I must answer myself.

### Interview Simulation

Conduct mock interviews one question at a time. Do not reveal the ideal answer immediately.

Evaluate my response on technical correctness, depth, clarity, trade-off reasoning, and practical experience. Identify gaps, ask follow-up questions, and then show a stronger model answer.

Adapt the difficulty to my current level and progressively increase it. Help me explain concepts in my own words rather than memorize definitions.

---

## 18. Production Failure Catalog and Incident Debugging

Teach me how real production systems fail and how engineers diagnose and fix them.

### 18.1 Incident Debugging Playbook

Teach and drill this sequence:

1. Confirm the impact: who is affected, since when, how badly (error rate, latency, revenue).
2. Ask "what changed?": deploys, config, flags, traffic, dependencies, certificates, cron jobs.
3. Check the golden signals: latency, traffic, errors, saturation.
4. Isolate the layer: client, CDN, load balancer, application, cache, database, queue, network, third-party.
5. **Mitigate first, investigate second**: rollback, scale out, shed load, disable a feature flag, fail over.
6. Find the root cause with evidence (metrics, logs, traces, query plans, profiles).
7. Fix permanently, add tests and alerts, and write a blameless postmortem.

### 18.2 Required Incident Scenarios

For each scenario, provide: symptoms, metrics and logs to check, ranked hypotheses, exact commands or queries, immediate mitigation, permanent fix, postmortem lessons, and one mock question for me.

1. API p99 latency suddenly 10x higher while average looks normal
2. 5xx spike right after a deployment
3. Database CPU at 100%
4. Connection pool exhausted and requests timing out
5. Memory leak and OOMKilled pods
6. Queue backlog growing and consumers falling behind
7. Duplicate payments or duplicate emails
8. Cache stampede after a Redis restart
9. Stale reads from replication lag
10. Cascading failure from one slow downstream dependency
11. Retry storm amplifying an outage
12. Disk full on a database or log host
13. Expired TLS certificate
14. DNS failure or misconfiguration
15. Cloud bill spike (egress, logs, NAT, autoscaling)
16. Noisy-neighbor tenant degrading others
17. Hot partition or hot shard
18. Deadlocks in the database
19. N+1 queries after a new feature
20. Schema migration locking a production table
21. Split brain or two leaders after a network partition
22. LLM provider outage or rate limiting
23. Runaway AI agent loop and token cost explosion
24. Region or availability-zone outage
25. Clock skew breaking token validation or ordering

### 18.3 Postmortem Template

Summary, impact, timeline, root cause, contributing factors, detection (how long to detect and recover), what went well, what went badly, action items with owners, and lessons learned. Show one fully worked example.

---

## 19. Data Structures and Algorithms: Detailed Requirements (extends Track A)

For each pattern below, apply the Section 0 template and also provide:

- How to **recognize** the pattern from the problem statement
- A reusable code template (state the language I choose)
- **Why it works** (the invariant) and why the brute force fails
- Alternatives and when they win
- Time and space complexity with derivation
- Edge cases checklist
- Common mistakes and off-by-one traps
- 5 practice problems from easy to hard
- Where it appears in real systems (production relevance)
- One mock question for me

### Patterns to cover

- Arrays and strings: two pointers, sliding window (fixed and variable), prefix sums, Kadane
- Hash maps and sets: frequency counting, grouping, two-sum variants, LRU/LFU cache
- Linked lists: reversal, fast and slow pointers, cycle detection, merge, dummy node
- Stacks and queues: monotonic stack, next greater element, expression evaluation, deque
- Binary search: on sorted arrays, on rotated arrays, and **on the answer space**
- Sorting: merge sort, quicksort, counting sort, custom comparators, interval merging
- Trees: traversals (recursive and iterative), BST operations, LCA, serialization, diameter, path sums
- Heaps: top-K, k-way merge, median of a stream, scheduling
- Graphs: BFS, DFS, connected components, cycle detection, topological sort, union-find, shortest paths (BFS, Dijkstra, Bellman-Ford), minimum spanning tree
- Tries: prefix search, autocomplete
- Recursion and backtracking: subsets, permutations, combinations, N-Queens, pruning
- Dynamic programming: 1D, 2D, knapsack, LIS, LCS, edit distance, partition, grid paths, state machines, memoization vs tabulation, space optimization
- Greedy: intervals, scheduling, exchange arguments and how to prove or disprove greedy
- Bit manipulation, math tricks, and intervals
- Complexity analysis: amortized analysis, recursion trees, the Master theorem

### Problem-solving communication routine

Drill this routine: clarify the problem and constraints, give a brute force first, find the bottleneck, optimize, dry-run on an example, state complexity, then test edge cases. Teach me to think aloud.

### Level targets (illustrative, adjust to my timeline)

- **SDE1**: strong on arrays, strings, hash maps, linked lists, stacks, trees, BFS/DFS, binary search, basic DP. Mostly easy and medium problems.
- **SDE2**: all of the above plus graphs, heaps, advanced DP, backtracking, tries, union-find, and medium-hard problems solved cleanly under time pressure, with optimization discussion.

### Production problems in algorithms

Cover: O(n^2) code that passes tests but fails at scale, recursion stack overflow, integer overflow, floating-point errors, unbounded memory from naive approaches, and pathological inputs (hash collisions, quicksort worst case).

---

## 20. Low-Level Design and Concurrency: Detailed Requirements (extends Track B)

### 20.1 Core topics (apply the Section 0 template to each)

- OOP: encapsulation, abstraction, inheritance, polymorphism
- SOLID: for each principle show a **violating design**, the bug or change request that exposes it, and the refactor
- Composition vs inheritance
- Design patterns, each with problem, structure, when to use, **when not to use**, and alternatives: Strategy, Factory, Abstract Factory, Builder, Singleton (and its pitfalls), Observer, State, Command, Decorator, Adapter, Facade, Chain of Responsibility, Template Method, Proxy
- Dependency injection and inversion of control
- API and interface design: naming, idempotency, versioning, error handling, pagination
- Testability: mocking, seams, pure functions
- UML class, sequence, and state diagrams
- Error handling, immutability, and defensive design

### 20.2 Design exercises

For each: requirements, entities, class diagram, key interfaces, patterns used and why, extensibility scenarios ("what if we add X?"), thread-safety discussion, tests, and a mock question.

Parking lot, elevator, notification service, payment system, rate limiter, LRU cache, BookMyShow, Splitwise, ATM, vending machine, chess, logger framework, pub-sub system, task scheduler, library management, food delivery, URL shortener (class level), and an AI agent tool registry.

### 20.3 Concurrency and multithreading

Apply the Section 0 template to each:

- Processes vs threads vs async/event loops
- Race conditions, critical sections, mutexes, semaphores, read-write locks
- Deadlock (four conditions), livelock, starvation, and how to prevent them
- Atomic operations, volatile, and memory visibility
- Thread pools and executors: sizing, queues, rejection policies
- Producer-consumer, blocking queues, and condition variables
- Thread-safe singleton, thread-safe LRU cache, thread-safe rate limiter
- Async/await vs threads and the GIL (for Python)
- Optimistic vs pessimistic locking

**Production problems:** thread pool exhaustion, deadlock in production, lock contention hurting throughput, race conditions causing rare data corruption, blocking calls inside async code, unbounded queues causing memory exhaustion, and leaked threads or connections.

### 20.4 Machine coding round format

Teach me to deliver a working, runnable, extensible solution in 60-90 minutes: requirement clarification (5-10 min), entity design (10 min), core implementation (40 min), tests and edge cases (15 min), and extensibility discussion (10 min). Include a checklist of what interviewers grade: working code, clean structure, naming, extensibility, edge cases, and tests.

### 20.5 Production problems in design

God classes, tight coupling, leaky abstractions, untestable code, breaking API changes, over-engineering with unnecessary patterns, and anemic domain models.

---

## 21. High-Level Design: Extensions (extends Track C)

### 21.1 Additional problems

In addition to the Track C list, cover: Twitter/news feed, Uber/ride matching, Google Drive/Dropbox sync, search autocomplete, payment gateway, API gateway, web crawler, ticket booking with high contention (Ticketmaster), leaderboard, metrics and monitoring system, log aggregation, key-value store, unique ID generator, pub-sub/message queue, CI/CD platform, and a RAG-based AI assistant.

### 21.2 Required additions for every HLD problem

On top of the 11-step sequence in Track C, always include:

- **Capacity math** shown step by step (QPS, peak QPS, storage per year, bandwidth, cache size)
- **Data model** with indexes and the reason for the chosen database
- **Failure scenarios and production problems** specific to the design (what breaks at 10x scale, and what you would see on the dashboards)
- **Consistency decisions**: where strong consistency is needed and where eventual is fine
- **Hot spots**: celebrity users, viral content, hot keys
- **Cost estimate** (illustrative)
- **Simplest version first**, then how it evolves as scale grows (SDE1 stops earlier; SDE2 goes deeper)
- **Interviewer follow-ups** and how to handle "what if X fails?"

---

## 22. Computer Science Fundamentals

Apply the Section 0 template to each topic.

### Operating systems

Processes vs threads, context switching, scheduling, virtual memory and paging, stack vs heap, deadlocks, synchronization primitives, system calls, file descriptors, IPC, and I/O models (blocking, non-blocking, epoll, async).

### Networking

OSI and TCP/IP models, TCP handshake and teardown, flow and congestion control, TIME_WAIT, UDP, HTTP request lifecycle, "what happens when you type a URL in the browser", DNS resolution, TLS handshake, L4 vs L7 load balancing, NAT, and proxies.

### Database theory

ACID, normalization and denormalization, B-tree vs LSM-tree storage engines, write-ahead logging, MVCC, isolation levels and their anomalies (dirty read, non-repeatable read, phantom, write skew), join algorithms, query optimizer basics, and reading `EXPLAIN` output.

### Language and runtime (for my chosen language)

Memory model, garbage collection, references vs values, strings and immutability, exceptions, generics, async runtime, and common performance pitfalls.

### Linux and debugging tools

`top`, `htop`, `ps`, `netstat`/`ss`, `lsof`, `strace`, `tcpdump`, `curl`, `dig`, `df`, `iostat`, `journalctl`, and profilers.

**Production problems:** memory leaks, file descriptor exhaustion, too many TIME_WAIT sockets, swap thrashing, zombie processes, CPU steal, and disk I/O saturation.

---

## 23. SQL and Database Practice

- Joins (inner, left, self, cross), aggregation, GROUP BY and HAVING, subqueries, CTEs, and window functions (`ROW_NUMBER`, `RANK`, `LAG`, `LEAD`, running totals)
- Classic patterns: top-N per group, deduplication, gaps and islands, second-highest salary, consecutive events, pivoting
- Index design for a given query, reading `EXPLAIN`/`EXPLAIN ANALYZE`, and fixing slow queries
- Transactions and isolation level demonstrations
- MongoDB equivalents: aggregation pipeline, compound indexes, and `explain()`
- 50 graded practice problems (easy to hard) with solutions revealed only after I attempt them

**Production problems:** slow queries from missing indexes, lock waits, deadlocks, N+1 queries, unbounded `SELECT *`, and bad pagination (`OFFSET` on huge tables vs keyset pagination).

---

## 24. Behavioral Rounds, Project Deep-Dives, and Communication

### 24.1 Behavioral preparation

Teach the STAR method (Situation, Task, Action, Result). Help me build a **story bank of 10 stories**, each reusable for several questions:

conflict with a teammate, disagreeing with a manager, failure or mistake, tight deadline, ambiguity, ownership beyond my role, mentoring someone, a hard technical decision, a production incident, learning something quickly, and customer focus.

Map stories to common frameworks (Amazon leadership principles and similar). For every story, give feedback on specificity, my personal contribution ("I" vs "we"), measurable results, and what I learned.

### 24.2 Resume and project deep-dive

For each project on my resume, drill me on: the problem, why this design, alternatives considered, scale and numbers, the biggest bottleneck, a failure I faced, how I debugged it, what I would change, and how I tested and monitored it. Interviewers probe until they find where my knowledge ends, so teach me to answer honestly and confidently.

### 24.3 Level expectations

- **SDE1**: writes correct, clean code, learns quickly, and completes well-scoped tasks with guidance.
- **SDE2**: owns features end to end, makes and defends design decisions, handles ambiguity, debugs production issues, reviews code, and mentors juniors.

### 24.4 Other

How to ask good questions to the interviewer, how to handle questions I can't answer, and the basics of offer evaluation and negotiation.

---

## 25. Level-Specific Plan: SDE1 vs SDE2

### 25.1 Round weighting (typical, varies by company)

Produce a table showing, for SDE1 and SDE2: DSA, LLD/machine coding, HLD, CS fundamentals, database, behavioral, and project deep-dive, with approximate emphasis and the topics to skip at each level.

### 25.2 Priority tiers

List P0/P1/P2 topics for each level across all 27 sections. Be explicit that SDE1 candidates can safely skip or skim FinOps depth, cell-based architecture, multi-region active-active, chaos engineering, and consensus internals, and that SDE2 candidates should know basics of all of them with real depth in caching, databases, queues, API design, and failure handling.

### 25.3 Study timelines

Generate week-by-week plans for 8, 12, and 16 weeks, with a daily routine (DSA practice, one concept study, one design or LLD exercise, one mock or review), weekly mock interviews, and revision cycles with spaced repetition.

### 25.4 Readiness checklist

Provide a self-assessment rubric (1-5) for each topic area, and the criteria for "interview ready" at each level.

---

## 26. Mock Interview Engine

Ask my level (SDE1 or SDE2), target company type, and weak areas before starting.

### Modes

DSA, LLD/machine coding, HLD, database and infrastructure, production debugging ("the site is slow, what do you do?"), behavioral, and project deep-dive.

### Rules

1. One question at a time. Wait for my answer. Do not reveal the ideal answer first.
2. Score my answer from 1 to 5 on: technical correctness, depth, clarity, trade-off reasoning, production awareness, and communication.
3. Ask 2-3 follow-up questions that probe my weak points.
4. Then show a stronger model answer, and explain why it is stronger.
5. Keep a **weakness tracker** and revisit weak topics later with fresh questions (spaced repetition).
6. Increase difficulty gradually as I improve.
7. Help me explain concepts in my own words rather than memorize definitions.
8. At the end of each session, give a summary: scores, top 3 gaps, and a short practice plan.

---

## 27. Quality Rules and Delivery Order

### 27.1 Honesty and accuracy rules

- Use official documentation for current features, limits, and pricing. Cite sources and state the date of any pricing.
- Clearly label illustrative numbers versus real provider prices versus measured metrics.
- Say so when unsure. Do not invent API names, configuration flags, or limits.
- Flag anything that is version-specific or changes frequently.
- Recommend the simplest architecture that meets the requirements. Never recommend unsafe security shortcuts.
- Warn me that citations or book/paper references may be inaccurate and must be verified.

### 27.2 Delivery order

Do not try to deliver everything in one response. Work in parts, and after each part give: a short summary, 5 review questions, and 1 mock question. Wait for me to say "continue" before moving to the next part.

Recommended order:

1. Concept map, glossary, and CS fundamentals (Section 22)
2. DSA patterns (Section 19) in parallel with SQL (Section 23)
3. Databases, caching, and storage (Sections 1, 7)
4. Compute, networking, and distributed systems (Sections 2, 4, 6)
5. Design patterns, LLD, and concurrency (Sections 5, 20)
6. HLD problems (Sections 17 Track C, 21)
7. Reliability, observability, security, and CI/CD (Sections 8, 9, 10, 11)
8. Production failure catalog (Section 18)
9. Cost engineering and quantitative engineering (Sections 3, 14)
10. AI application infrastructure (Section 12)
11. Case studies (Section 13)
12. Behavioral and project deep-dive (Section 24)
13. Full mock interview loops (Section 26)
