# Part 4 of 13: Compute, Networking, and Distributed Systems (Sections 2, 4, 6)

> Tags: `[SDE1]` `[SDE2]` `[Both]`; priority P0/P1/P2. Numbers marked *(illustrative)* are teaching examples, not provider prices or measured metrics. Items marked **verify** depend on provider/version; check official docs.
> Template per topic: what, why, how, alternatives, when/when not, trade-offs, production problems (symptom / cause / detect / fix), example, mistakes, Q&A, your turn. Answers are in the **Answer Key**.

---

## Part A: Cloud Compute (Section 2)

### 4.1 VMs vs Containers vs Serverless `[Both]` P0

**What.** *VM:* full guest OS on a hypervisor. *Container:* isolated process sharing the host kernel (namespaces + cgroups). *Serverless function (FaaS):* you upload code; the platform runs it per event and bills per invocation/duration.

**Why.** Trade isolation strength, startup time, density, and operational burden.

**How.** VMs virtualize hardware (strong isolation, minutes to boot, GBs). Containers package app + dependencies in an image (seconds to start, MBs-GBs, weaker isolation since the kernel is shared). FaaS runs your function in a managed sandbox (often microVMs) that is created on demand (**cold start**) and reused while warm.

| | VM | Container | Serverless |
|---|---|---|---|
| Startup | tens of seconds-minutes | ~1-10 s | ms-seconds (cold start varies by runtime/size) |
| Unit billed | instance-hour | node/pod resources | invocation + GB-seconds |
| Idle cost | full | node cost | none (scale to zero) |
| Control | highest | high | lowest (time/memory/payload limits; **verify**) |
| Ops burden | OS patching, scaling | orchestrator ops | minimal |
| Good for | legacy, stateful, special kernels/GPUs | microservices, steady workloads | spiky/event-driven, glue code, low traffic |

**When not.** Serverless for long-running, steady high-throughput, latency-critical (cold starts), or connection-heavy (DB) workloads. Containers/K8s for tiny teams with simple needs (consider a managed container service). VMs for workloads needing special isolation or kernel features.

**Trade-offs.** Control vs. toil; idle cost vs. per-request premium; isolation vs. density.

**Production problems.**
- **Cold-start latency spikes (serverless).** Symptom: p99 jumps for first requests after idle/scale-out. Cause: new sandbox + runtime init + dependency load + VPC/DB connect. Detect: init duration metrics, p99 vs p50. Fix: smaller packages, lazy imports, provisioned/min-warm concurrency, lighter runtime. Prevent: load test from zero.
- **Concurrency limits/throttling.** Symptom: 429s/throttle errors in bursts. Cause: account/region concurrency caps. Fix: reserved concurrency, queue in front, request quota raise.
- **DB connection storms from functions** (see Part 3, 3.8). Fix: proxy/pooler, limit concurrency.
- **Noisy neighbor on shared VMs / CPU steal.** Detect: `%steal` in `top`/`mpstat`. Fix: dedicated/larger instances.
- **Container escape / shared-kernel risk** for untrusted code. Fix: stronger sandboxing (microVMs/gVisor-style), no privileged containers.

**Example.** Image-thumbnail endpoint with 20 requests/min: Lambda-style FaaS, near-zero cost. Core API with 1,000 steady RPS: containers on Kubernetes/managed container service.

**Common mistakes.** "Serverless is always cheaper." Running a stateful database in ephemeral containers without volumes/operators. Ignoring image size and startup.

**Q&A.** *Container vs VM?* Containers share the host kernel (lighter, faster, weaker isolation); VMs have their own kernel. *Why do cold starts happen?* The platform must create a sandbox, load code, and initialize the runtime before the first request.

**Your turn (Q1).** Pick VM, container, or serverless for (a) nightly 10-minute data job, (b) public REST API at steady 800 RPS, (c) webhook receiver averaging 3 req/s with 500 req/s bursts. Justify with cost and latency.

---

### 4.2 Kubernetes Essentials: Requests, Limits, Scheduling, Probes `[Both]` P0

**What.** Kubernetes schedules **pods** (one or more containers) onto **nodes**, restarts failures, and exposes them via Services.

**How it works.**
- **Requests** = resources the scheduler *reserves* (placement decision). **Limits** = hard cap enforced at runtime. CPU over limit => **throttled** (CFS quota); memory over limit => **OOMKilled**.
- **QoS classes:** *Guaranteed* (requests == limits for all containers), *Burstable*, *BestEffort* (evicted first under pressure).
- **Probes:** *startup* (is it done booting?), *readiness* (send traffic?), *liveness* (restart me?). Readiness removes from Service endpoints; liveness restarts.
- **Rolling update** (`maxSurge`, `maxUnavailable`), **PodDisruptionBudget** (limit voluntary disruptions), **anti-affinity/topology spread** (spread across nodes/AZs), **graceful termination** (SIGTERM, `terminationGracePeriodSeconds`, `preStop`).

```yaml
containers:
- name: api
  image: registry/api:1.8.3          # immutable tag, never :latest
  resources:
    requests: { cpu: "500m", memory: "512Mi" }
    limits:   { memory: "512Mi" }     # CPU limit often omitted to avoid throttling (a debated practice)
  readinessProbe: { httpGet: { path: /ready, port: 8080 }, periodSeconds: 5 }
  livenessProbe:  { httpGet: { path: /live,  port: 8080 }, periodSeconds: 10, failureThreshold: 3 }
  lifecycle: { preStop: { exec: { command: ["sleep","10"] } } }   # let LB deregister first
```

**Capacity planning: CPU vs memory.** CPU is compressible (throttle, slower); memory is incompressible (kill). Size memory from measured peak + headroom; size CPU from throughput targets (requests/s per core x load). Use requests close to real usage (over-requesting wastes money; under-requesting causes noisy-neighbor and eviction).

**Alternatives.** Managed container services (ECS/Cloud Run/Container Apps; **verify**), PaaS, plain VMs with systemd, Nomad.

**When not.** Small teams with a handful of services and simple scaling: the operational overhead may exceed the benefit.

**Trade-offs.** Powerful standardization and self-healing vs. steep learning curve, YAML/ops complexity, and cost of the control plane and platform team.

**Production problems.**
- **OOMKilled (exit 137).** Cause: memory limit too low/leak/JVM heap near limit. Detect: `kubectl describe pod`, container restarts. Fix: raise limit, fix leak, tune heap to ~70-75% of limit.
- **CPU throttling latency.** Symptom: p99 spikes while average CPU is low. Cause: tight CPU limit (CFS quota). Detect: `container_cpu_cfs_throttled_periods_total`. Fix: raise/remove CPU limit, right-size requests.
- **CrashLoopBackOff.** Cause: bad config/secret, failing dependency at boot, failing liveness probe. Detect: `kubectl logs --previous`. Fix: fix config, add startup probe, don't let liveness depend on downstreams.
- **Pending pods.** Cause: insufficient node resources, quota, affinity/taints. Detect: `kubectl describe pod` events. Fix: cluster autoscaler, adjust requests/quotas.
- **Readiness/liveness misconfig causing restart loops** (liveness too aggressive during GC/slow start). Fix: startup probe, generous thresholds.
- **Dropped requests during deploy.** Cause: pod killed before LB deregisters it, no graceful shutdown. Fix: `preStop` delay, handle SIGTERM (stop accepting, drain, exit), PDB.
- **Node pressure eviction, noisy neighbors.** Fix: right-size requests, set limits, QoS Guaranteed for critical pods.
- **Image pull failures / registry rate limits.** Fix: private registry/cache, image pull secrets, smaller images.

**Common mistakes.** Using `:latest`. Liveness probe checking the database. Requests = 0 (BestEffort). One replica for a "highly available" service.

**Q&A.** *Requests vs limits?* Requests drive scheduling; limits cap usage (CPU throttles, memory kills). *Liveness vs readiness?* Liveness: restart if broken; readiness: stop routing traffic until ready.

**Your turn (Q2).** p99 doubles during traffic peaks although CPU averages 35%, memory is fine, and pods have a 1-CPU limit. Diagnose and fix.

---

### 4.3 Autoscaling, Scale-to-Zero, Cold Starts `[Both]` P0

**What.** **HPA** (horizontal pod autoscaler) changes replica count; **VPA** changes pod requests; **Cluster Autoscaler** adds/removes nodes; event-driven autoscalers (e.g., KEDA-style, **verify**) scale on queue depth.

**How.** HPA formula: `desiredReplicas = ceil(currentReplicas x currentMetric / targetMetric)`. Example: 10 pods at 90% CPU, target 60%: `ceil(10 x 90/60) = 15`. Scale-up is limited by pod startup + node provisioning time (minutes if a new node is needed). Scale-down uses stabilization windows to avoid flapping.

**Scale-to-zero vs always-on.** Zero saves idle cost; first request pays cold start. Always-on (min replicas >= 2 across AZs) gives consistent latency and HA.

**Alternatives.** Scheduled scaling (known daily peaks), predictive scaling, over-provisioned buffer (headroom pods), queue-based scaling for workers.

**Trade-offs.** Cost efficiency vs. reaction lag; scaling on CPU is simple but misleading for I/O-bound services (scale on RPS, latency, or queue depth instead).

**Production problems.**
- **HPA flapping.** Symptom: replicas oscillate. Cause: noisy metric, short windows. Fix: stabilization windows, smoother metric.
- **Scale-up too slow for spikes.** Symptom: latency/5xx at the start of a surge. Cause: pod start + node provisioning. Fix: headroom, scheduled/predictive scaling, faster images, lower min-readiness time.
- **Autoscale overwhelms the DB** (more pods = more connections). Fix: pooler, max replicas cap, scale DB-side limits (Part 3).
- **Scaling on the wrong signal** (CPU low but queue growing). Fix: queue-depth or latency-based metrics.
- **Autoscaling cost explosion during attack/retry storm.** Fix: max replicas, WAF/rate limiting, budget alerts.
- **VPA + HPA fighting** on same metric. Fix: don't combine on CPU/memory.

**Example *(illustrative)*.** Worker pool scaled on SQS-like queue depth: target 100 messages per pod; depth 5,000 => 50 pods; depth 0 => min 1-2.

**Common mistakes.** No max replica cap. Scaling stateful pods like stateless. Ignoring node provisioning delay.

**Q&A.** *HPA vs VPA?* HPA adds pods (stateless scale-out); VPA resizes pods (right-sizing, needs restarts). *Why is CPU a poor autoscaling signal for I/O-bound APIs?* CPU stays low while threads/connections saturate.

**Your turn (Q3).** Your API's latency degrades when RPS passes 1,200, but HPA targeting 70% CPU never triggers. Why, and what metric would you scale on?

---

### 4.4 Pricing Models, Quotas, IaC, and Environments `[SDE2]` P1

**What.** *On-demand* (pay as you go, flexible), *reserved/committed* (discount for 1-3 year commitment), *spot/preemptible* (large discounts but can be reclaimed on short notice; **verify** discount and notice times by provider), *dedicated hosts* (compliance, licensing, isolation).

**How to choose.**
- Baseline steady load -> commitments.
- Fault-tolerant, stateless, interruptible work (batch, CI runners, stateless workers behind queues) -> spot.
- Spiky/unknown -> on-demand/autoscale.
- Mixed fleets: on-demand floor + spot burst, spread across instance types/AZs to reduce simultaneous reclamation.

**Infrastructure as Code (IaC).** Terraform/Pulumi/CloudFormation-style declarative definitions in version control; **plan -> review -> apply**; remote state with locking; modules; policy-as-code (e.g., deny public buckets). **Environment isolation:** separate accounts/projects (or at least VPCs and IAM) for dev/staging/prod; prod access restricted; staging mirrors prod shape at smaller scale; ephemeral preview environments auto-destroyed. **Quotas:** cloud service limits and Kubernetes `ResourceQuota`/`LimitRange` protect against runaway usage.

**Trade-offs.** Commitments: savings vs. lock-in and forecast risk. Spot: cost vs. interruption handling complexity.

**Production problems.**
- **Spot interruption kills work.** Fix: handle termination notice, checkpoint, retry/idempotent jobs, diversify pools.
- **Configuration drift** (manual console changes). Detect: drift detection (`terraform plan`). Fix: all changes via IaC, restrict console writes.
- **State file corruption/lock conflicts, leaked secrets in state.** Fix: remote encrypted state, locking, least privilege.
- **Quota hit during an incident** (cannot launch instances). Fix: know and pre-raise quotas, monitor usage.
- **Non-prod cost leakage** (staging 24/7 at prod size). Fix: schedules, smaller sizes, TTL on preview environments.
- **Over-commit** (reserved capacity unused after migration). Fix: commit to proven baseline only.

**Common mistakes.** One account for everything. Applying Terraform from laptops without review. Spot for singleton stateful services.

**Q&A.** *When use spot?* For stateless, retryable, or checkpointed workloads that tolerate interruption. *Why IaC?* Reproducibility, review, drift detection, disaster recovery.

**Your turn (Q4).** Design the fleet for a video-transcoding worker pool (jobs take 2-15 minutes) with a cost goal. Which purchasing models and what safeguards?

---

## Part B: Scalability and Distributed Systems (Section 4)

### 4.5 Load Balancing, Stateless vs Stateful, Service Discovery, API Gateways `[Both]` P0

(L4/L7 internals were covered in Part 1 (3.7); this adds architecture.)

**What.** *Stateless service:* any instance can handle any request (state lives in DB/cache/token). *Stateful:* instance holds data/sessions (databases, WebSocket gateways, caches). *Service discovery:* how callers find healthy instances (DNS, Kubernetes Services, registries like Consul/Eureka, mesh). *API gateway:* single entry for routing, authN/Z, rate limiting, TLS, request transformation, and observability.

**Why.** Statelessness enables horizontal scaling and easy replacement; gateways centralize cross-cutting concerns.

**How.** Externalize sessions (signed tokens or Redis), make handlers idempotent, store uploads in object storage. Discovery: client-side (client picks from registry; low hop count, smarter client) vs server-side (LB/DNS; simpler clients). Gateway sits behind the LB, applies policies, routes by path/host.

**Alternatives.** Direct service-to-service via mesh (sidecar proxies handle retries/mTLS/discovery), BFF per client type, Ingress controllers.

**When not.** A heavyweight gateway for two internal services; sticky sessions as a crutch for state that belongs in a store.

**Trade-offs.** Gateway = single choke point (HA required, adds latency, can become a "god" layer); mesh = powerful but significant ops/resource overhead.

**Production problems.**
- **Gateway as bottleneck/SPOF.** Cause: single instance or heavy per-request logic. Fix: scale out, keep gateway logic thin, multi-AZ.
- **Stale service discovery** (DNS TTL/caching, dead instances still listed). Symptom: intermittent connection errors after deploys. Fix: health-based deregistration, lower TTL, client retries on other instances.
- **Session loss on scale-in** with sticky sessions. Fix: external session store.
- **Config drift in routing rules,** accidental open routes. Fix: config as code, review, tests.

**Q&A.** *Why stateless?* Horizontal scale and resilience. *Where does state go?* DB, cache, object storage, signed tokens.

**Your turn (Q5).** Your service stores shopping carts in server memory with sticky sessions. List the failure modes at scale and the redesign.

---

### 4.6 Synchronous vs Asynchronous; Queues vs Streams; Kafka vs RabbitMQ vs Cloud Queues `[Both]` P0

**What.** *Sync:* caller waits for the result (HTTP/gRPC). *Async:* caller hands work to a broker and continues; results via callback/polling/events. *Queue:* each message processed by one consumer, then removed. *Stream/log:* ordered, retained, replayable; multiple independent consumer groups.

**Why.** Decouple producers from consumers, absorb bursts, retry independently, isolate failures, scale workers separately.

**How.**
- **Kafka:** topics split into **partitions** (ordering only *within* a partition); consumer groups assign partitions to consumers (max parallelism = #partitions); consumers track **offsets**; data retained by time/size, replayable; producer `acks=all` + `min.insync.replicas` + replication for durability; idempotent producer and transactions for stronger guarantees.
- **RabbitMQ:** broker pushes messages from **exchanges** to **queues** with routing rules; per-message ack/nack, TTL, **dead-letter exchanges**, priorities; prefetch controls consumer concurrency.
- **Cloud queues (e.g., SQS-style):** managed, near-infinite scale; *standard* = at-least-once, best-effort ordering; *FIFO* = ordering per message group with deduplication window (**verify** limits); **visibility timeout** hides an in-flight message; unprocessed messages reappear; DLQ after N receives.

| Need | Lean |
|---|---|
| Task queue, routing, per-message ack, low ops (managed) | Cloud queue / RabbitMQ |
| Event log, replay, high throughput, multiple consumers, stream processing | Kafka (or managed equivalent) |
| Simple background jobs in a small app | DB-backed or Redis-based job queue (accept limits) |

**Backlog math.**

```
Backlog growth rate = arrival rate - service rate
Time to drain       = backlog / (service rate - arrival rate)        (only if service > arrival)
Example (illustrative): arrival 1,200 msg/s, 10 workers x 100 msg/s = 1,000 msg/s capacity
   => backlog grows 200 msg/s = 720,000 msgs/hour. Need >= 12 workers; with 20 workers (2,000/s),
      a 720,000 backlog drains in 720,000 / (2,000 - 1,200) = 900 s = 15 min.
```

**When not.** Async for flows needing an immediate answer with strong consistency (e.g., "is this username available?"); a streaming platform for a few jobs/hour.

**Trade-offs.** Resilience and elasticity vs. eventual consistency, harder debugging, duplicate handling, operational cost (Kafka clusters are heavy; managed offerings reduce toil).

**Production problems.**
- **Consumer lag / backlog growth.** Symptom: rising lag, delayed side effects. Cause: slow consumers, downstream dependency, too few partitions, rebalance storms. Detect: lag per partition/consumer group, queue depth, oldest-message age. Fix: scale consumers (up to partition count), fix slow dependency, batching, increase partitions (plan: changes key-to-partition mapping).
- **Poison message** blocks partition/queue forever (consumer crashes on each retry). Fix: bounded retries, **dead-letter queue**, schema validation.
- **Rebalance storms** (Kafka consumers join/leave). Symptom: processing pauses repeatedly. Cause: slow `poll` loops exceeding `max.poll.interval`, flapping pods. Fix: tune timeouts, static membership, cooperative rebalancing, lighter processing per poll.
- **Hot partition** from skewed keys. Fix: better key, salting where ordering isn't needed.
- **Lost messages** (producer `acks=1`, no replication; consumer commits offset before processing). Fix: `acks=all`, commit after successful processing, transactional outbox.
- **Unbounded queue growth masking an outage.** Fix: queue-age alerts, TTLs, load shedding.
- **Message visibility timeout too short** => duplicate processing. Fix: extend/heartbeat, idempotent consumers.

**Example.** Order placed -> `OrderCreated` event on Kafka -> inventory, email, analytics consumers each in their own group. A new consumer group can replay history without affecting others.

**Common mistakes.** Treating a queue as exactly-once. Assuming global ordering. Too few partitions to scale later. No DLQ or monitoring of queue age.

**Q&A.** *Kafka vs RabbitMQ?* Kafka: durable log, replay, partition-ordered, high throughput, pull model. RabbitMQ: flexible routing, per-message ack, push model, task-queue semantics. *How does Kafka scale consumers?* Partitions are divided among consumers in a group.

**Your turn (Q6).** Consumers of a Kafka topic (12 partitions) fall behind at peak. You have 30 consumer pods. What's wrong, and what are three fixes?

---

### 4.7 Delivery Semantics, Idempotency, Duplicates, and Ordering `[Both]` P0

**What.** *At-most-once* (may lose, never duplicates), *at-least-once* (never loses, may duplicate), *exactly-once* (as an end-to-end guarantee, effectively achieved by **at-least-once delivery + idempotent processing**).

**Why.** Networks and processes fail between "did the work" and "acknowledged the work", so duplicates or losses are unavoidable; design for them.

**How: idempotency.**
- Client sends an **idempotency key** with a request; server stores `(key -> status, response)` under a **unique constraint** and replays the stored result for repeats.
- Consumers dedupe on a message/event id, or make operations naturally idempotent (`SET status='SHIPPED'` instead of `counter += 1`; upserts).
- **Transactional outbox:** write the business change *and* an outbox row in one DB transaction; a relay publishes the outbox to the broker (at-least-once) => no dual-write inconsistency. Consumers are idempotent.

```python
def charge(db, gateway, idem_key, request):
    row = db.get(idem_key)
    if row and row.status == "DONE":
        return row.response                              # replay stored result
    if row and row.status == "IN_PROGRESS":
        raise ConflictError("request in progress")       # or wait/poll
    db.insert(idem_key, status="IN_PROGRESS", request_hash=hash(request))   # UNIQUE(idem_key)
    resp = gateway.charge(request, idempotency_key=idem_key)  # pass key downstream too
    db.update(idem_key, status="DONE", response=resp)
    return resp
# Crash window: after gateway.charge but before db.update. Passing the key to the gateway makes the retry safe.
```

**Ordering.** Total order across a distributed system is expensive. Usually guarantee order *per key* (partition by entity id) and make consumers tolerate out-of-order via version numbers/timestamps ("apply only if event.version > stored.version").

**Alternatives.** Distributed transactions (2PC; blocking, rarely used across services), single writer per entity, event sourcing.

**Trade-offs.** Idempotency storage and logic vs. correctness under retries; strict ordering limits parallelism.

**Production problems.**
- **Duplicate side effects** (double charge/email). Cause: retries/redelivery without dedupe. Detect: duplicate business ids, support tickets, reconciliation. Fix: idempotency keys + unique constraints. Prevent: chaos tests that redeliver messages.
- **Dedup table grows forever.** Fix: TTL matched to max retry horizon.
- **Out-of-order events overwrite newer state.** Fix: versioning, per-key ordering.
- **Same idempotency key, different payload.** Fix: store and compare request hash, return 422/409.
- **Dual-write inconsistency** (DB commit succeeds, publish fails). Fix: outbox/CDC.

**Q&A.** *Can you get exactly-once?* Not literally over unreliable networks; you get effectively-once via at-least-once + idempotency (and some systems provide transactional exactly-once within their own boundary). *What is the outbox pattern?* Persist events in the same transaction as state change, publish asynchronously.

**Your turn (Q7).** Design idempotent handling for "POST /payments" including concurrent duplicate requests and a crash after the provider call.

---

### 4.8 Resilience: Timeouts, Retries, Circuit Breakers, Bulkheads, Backpressure, Load Shedding, Rate Limiting `[Both]` P0

**What.** A toolkit that keeps a failing dependency from taking down callers.

**How each works.**
- **Timeouts/deadlines:** every network call has one; propagate a *deadline* so downstream work stops when the caller has given up.
- **Retries with exponential backoff + jitter:** retry only transient, **idempotent** failures; add randomness to desynchronize clients.
- **Circuit breaker:** closed (normal) -> open after failure threshold (fail fast) -> half-open (probe) -> closed.
- **Bulkhead:** separate pools/queues per dependency or tenant so one cannot exhaust shared resources.
- **Backpressure:** bounded queues; tell producers to slow down (blocking, 429/`Retry-After`, reactive streams).
- **Load shedding:** under overload, reject low-priority or excess work early to preserve goodput.
- **Rate limiting:** cap per client (token bucket allows bursts; leaky bucket smooths; sliding window is precise).

```python
import random, time

def call_with_retry(fn, max_attempts=4, base=0.1, cap=5.0, retryable=(TimeoutError, ConnectionError)):
    for attempt in range(max_attempts):
        try:
            return fn()
        except retryable:
            if attempt == max_attempts - 1:
                raise
            time.sleep(random.uniform(0, min(cap, base * 2 ** attempt)))   # "full jitter"

class CircuitBreaker:                     # not thread-safe; add a lock in real code
    def __init__(self, fail_threshold=5, reset_timeout=30.0):
        self.ft, self.rt = fail_threshold, reset_timeout
        self.failures, self.opened_at, self.state = 0, 0.0, "CLOSED"
    def call(self, fn):
        if self.state == "OPEN":
            if time.monotonic() - self.opened_at < self.rt:
                raise RuntimeError("circuit open: failing fast")
            self.state = "HALF_OPEN"                     # allow a probe
        try:
            result = fn()
        except Exception:
            self.failures += 1
            if self.state == "HALF_OPEN" or self.failures >= self.ft:
                self.state, self.opened_at = "OPEN", time.monotonic()
            raise
        self.failures, self.state = 0, "CLOSED"
        return result
```

**Retry amplification math.** If each of 3 layers retries 3 times, a failing bottom service sees up to 3^3 = 27x the load. Fix: retry at one layer only, use **retry budgets** (e.g., retries <= ~10% of requests), and circuit breakers.

**Alternatives/complements.** Hedged requests (send a second request after p95 latency, for idempotent reads), caching/stale fallback, graceful degradation (return partial/cached results), queue-based leveling.

**When not to retry.** Non-idempotent operations without an idempotency key; client errors (4xx); when the system is overloaded (retrying adds load).

**Trade-offs.** Resilience adds latency/complexity; aggressive breakers can cause false opens; shedding means intentionally failing some users to save the rest.

**Production problems.**
- **Retry storm.** Symptom: brief blip becomes sustained outage with 5-10x traffic. Cause: synchronized retries without backoff/jitter, stacked retries. Detect: request rate to dependency vs. user rate, retry counters. Mitigate: shed load, disable retries/feature flag, open breaker. Fix: backoff+jitter, retry budgets, single retry layer.
- **Cascading failure.** Symptom: one slow dependency => thread pools saturate => unrelated endpoints fail. Cause: missing timeouts/bulkheads. Fix: timeouts, bulkheads, circuit breakers, fallbacks.
- **Timeout mismatch** (inner timeout > outer). Work continues after the caller gave up. Fix: deadline propagation, outer > inner budget.
- **Thundering herd after recovery.** Fix: jitter, gradual ramp, warm caches.
- **Unbounded queues** hide overload until memory dies. Fix: bounded queues, backpressure.
- **Rate limiter misconfiguration** (blocks legit users/doesn't stop abuse). Fix: per-key limits, dry-run mode, metrics.

**Example.** Product page calls recommendations (non-critical). With a 150 ms timeout + breaker + fallback to "popular items", recommendation outages don't affect page availability.

**Common mistakes.** Default infinite timeouts. Retrying on every error. Sharing one thread pool for all dependencies.

**Q&A.** *Why add jitter?* Prevent clients retrying in lockstep. *Circuit breaker vs retry?* Retry handles transient blips; breaker stops hammering a dependency that's persistently failing.

**Your turn (Q8).** Service A calls B calls C; C slows to 5 s. A and B have 30 s timeouts and 3 retries each. Describe what happens and fix it.

---

### 4.9 Distributed Transactions and Sagas `[SDE2]` P1

**What.** A business operation spanning multiple services/databases (order + payment + inventory).

**Why.** Each service owns its data (no shared ACID transaction).

**How.**
- **Two-phase commit (2PC):** coordinator asks all to prepare, then commit. Strong atomicity but **blocking** if the coordinator fails, high latency, poor availability; rare across microservices.
- **Saga:** a sequence of **local transactions**; if a step fails, run **compensating actions** for completed steps (cancel payment, release stock). Two styles: **orchestration** (a central coordinator drives steps and state; clear flow, one place to debug) and **choreography** (services react to events; loosely coupled, flow becomes implicit and hard to follow).
- Requirements: every step and compensation must be **idempotent and retryable**; persist saga state; handle "pivot" steps that cannot be undone (e.g., email sent) by ordering them last.

```
Order Saga (orchestrated):
 CreateOrder(PENDING) -> ReserveInventory -> ChargePayment -> ConfirmOrder
   failure at ChargePayment => ReleaseInventory -> CancelOrder(FAILED)
```

**Alternatives.** Single database transaction (keep related data together: modular monolith), workflow engines with durable execution (Temporal-style; **verify**), event sourcing.

**When not.** If strong cross-entity consistency is essential and volume is modest: keep it in one database/service.

**Trade-offs.** Availability and autonomy vs. temporary inconsistency (intermediate states visible), compensation complexity, and difficult testing.

**Production problems.**
- **Stuck sagas** (step times out, compensation fails). Detect: sagas older than N minutes by state. Fix: retries with backoff, timeouts, manual-intervention queue, dashboards.
- **Non-idempotent compensation** (refund twice). Fix: idempotency keys.
- **Visible inconsistent state** (inventory reserved, order not confirmed). Fix: "pending" states with TTL, reservations expire.
- **Lost events** between steps. Fix: outbox, durable workflow state.
- **Choreography spaghetti.** Fix: switch to orchestration for complex flows, document event flow.

**Q&A.** *Why avoid 2PC in microservices?* Blocking, coordinator failure, latency, availability impact. *Orchestration vs choreography?* Central coordinator vs event-driven reactions; orchestration is easier to reason about for complex flows.

**Your turn (Q9).** Design the saga for booking a flight + hotel + car, where the hotel step fails after the flight is ticketed (non-refundable window).

---

### 4.10 Locks, Leader Election, Consensus, and Time `[SDE2]` P1

**What.** Coordinating nodes: mutual exclusion across processes, electing a single leader, agreeing on state.

**How it works.**
- **Consensus (Paxos/Raft):** a majority quorum agrees on a replicated log; a leader is elected per *term*; writes commit once a majority has them. Tolerates `f` failures with `2f+1` nodes (3 nodes tolerate 1; 5 tolerate 2). Systems: etcd, Consul, ZooKeeper (ZAB).
- **Distributed locks:** a lock with TTL in Redis (`SET key token NX PX ttl`) is fine for **efficiency** (avoid duplicate work) but **not safe for correctness**: a paused/slow holder can lose the lock after TTL and still write. For correctness use **fencing tokens** (monotonic number checked by the resource), or a consensus-backed lock service (etcd/ZooKeeper), or better, design with idempotency/optimistic concurrency.
- **Leader election:** via lease in a consensus store; leader renews; followers watch.
- **Clocks:** wall clocks drift and jump; don't use them to order events across machines. Use logical/vector clocks, hybrid logical clocks, or a single sequencer.

**Alternatives.** Database row lock/`SELECT ... FOR UPDATE SKIP LOCKED`, unique constraints, optimistic concurrency (version columns), single-writer partitioning (each entity owned by one worker via consistent hashing/partitions).

**When not.** Distributed locks when a unique constraint or idempotent operation solves it.

**Trade-offs.** Consensus: strong guarantees vs. latency (majority round trip) and availability loss without quorum.

**Production problems.**
- **Lock expiry while working (split brain of workers).** Symptom: two workers do the same job, double writes. Cause: GC pause/network delay > TTL. Fix: fencing tokens, idempotent operations, lease renewal with heartbeat.
- **Split brain / two leaders after partition.** Fix: quorum-based election, fencing the old leader (STONITH-style/epoch checks).
- **Leader flapping.** Cause: aggressive timeouts, overloaded nodes. Fix: tune timeouts, dedicated coordination nodes.
- **Clock skew breaking TTL/ordering/JWT validation.** Detect: NTP offset metrics. Fix: NTP/chrony monitoring, tolerance windows, logical clocks.
- **Coordination service outage** halting everything that depends on it. Fix: HA (3-5 nodes across AZs), degrade gracefully.

**Q&A.** *Why odd number of nodes?* Majority quorum: 3 tolerates 1 failure (4 still tolerates only 1 but costs more). *Is Redis SETNX a safe lock?* For efficiency yes; for correctness no, because expiry can occur while the holder is still running.

**Your turn (Q10).** Two cron workers sometimes both run the nightly billing job. Explain how, and give two safe designs.

---

## Part C: Networking and Communication (Section 6)

(TCP, DNS journey, TLS handshake, L4/L7 were in Part 1; here: protocol choices, real-time, security, topology, and cost.)

### 4.11 HTTP Versions; REST vs GraphQL vs gRPC `[Both]` P0

**What / how.**
- **HTTP/1.1:** one request per connection at a time (head-of-line blocking at the HTTP layer), keep-alive reuse; browsers open ~6 connections per host. **HTTP/2:** multiplexed streams over one TCP connection, header compression; but TCP-level head-of-line blocking remains on packet loss. **HTTP/3:** over QUIC (UDP): independent streams, faster handshakes, better on lossy mobile networks.
- **REST:** resources + HTTP verbs; cacheable (CDN/HTTP caching), simple, ubiquitous. Over/under-fetching.
- **GraphQL:** single endpoint; client specifies fields; avoids over-fetching and multiple round trips; needs **query complexity/depth limits**, caching is harder, **N+1 resolver problem** (use batching/dataloader).
- **gRPC:** HTTP/2 + Protocol Buffers; strongly typed contracts, streaming (client/server/bidirectional), efficient; needs proxy support for browsers (gRPC-web); less human-debuggable.

| | REST | GraphQL | gRPC |
|---|---|---|---|
| Best for | Public APIs, CRUD, caching | Many client types needing flexible views | Internal service-to-service, low latency, streaming |
| Caching | Easy (HTTP) | Hard | Custom |
| Typing/contracts | OpenAPI (optional) | Schema | Protobuf (strict) |
| Pitfall | Chatty clients | Expensive queries, N+1 | LB imbalance on long-lived connections (Part 1), browser support |

**API design essentials (SDE1/2 favorites).** Nouns for resources, proper status codes (200/201/204, 400/401/403/404/409/422/429, 5xx), **pagination** (cursor/keyset), **idempotency** for POST via keys, **versioning** (path or header) and **backward compatibility** (add fields, don't remove/rename), consistent errors, rate-limit headers.

**Production problems.**
- **GraphQL abuse/expensive queries.** Symptom: DB/CPU spikes from deeply nested queries. Fix: depth/complexity limits, persisted queries, timeouts, dataloader batching.
- **gRPC load imbalance** (covered Part 1). Fix: L7/mesh/client-side balancing.
- **Breaking API changes.** Symptom: older mobile apps fail. Fix: additive changes only, deprecation windows, contract tests, versioning.
- **Chatty REST clients** causing latency. Fix: aggregation endpoint/BFF, GraphQL, batch endpoints.
- **Large payloads/unbounded lists.** Fix: pagination, compression, field selection.

**Q&A.** *REST vs gRPC?* REST for broad/public compatibility and caching; gRPC for typed, efficient internal calls and streaming. *Why is HTTP/3 faster on bad networks?* Per-stream loss recovery and faster handshakes avoid TCP head-of-line blocking.

**Your turn (Q11).** Mobile clients make 8 REST calls to render a home screen over high-latency networks. Propose two redesigns with trade-offs.

---

### 4.12 Real-Time Communication: WebSockets vs SSE vs Polling `[Both]` P1

**What / how.**
- **Short polling:** client asks every N seconds (simple, wasteful, delayed). **Long polling:** server holds the request until data or timeout (near real-time over plain HTTP; many open requests).
- **Server-Sent Events (SSE):** one-way server->client stream over HTTP, auto-reconnect with `Last-Event-ID`; great for feeds, progress, LLM token streaming; works through most proxies.
- **WebSockets:** full-duplex over a persistent connection (chat, games, collaborative editing). Stateful: each connection pins to a server.

| Need | Choose |
|---|---|
| Occasional updates, simplicity | Polling |
| Server push only (notifications, LLM streaming) | SSE |
| Bidirectional low-latency (chat, multiplayer) | WebSockets |

**Scaling WebSockets.** Connections are long-lived and memory-bound (per-connection buffers); hold hundreds of thousands per server with tuned OS limits *(illustrative; benchmark)*. Keep a routing map (user -> gateway node) in Redis; publish messages through a pub/sub or broker so any node can reach any user; heartbeats/pings to detect dead connections; backoff + jitter on client reconnect.

**Production problems.**
- **Reconnect storm** after deploy/failover: all clients reconnect at once. Fix: jittered backoff, connection draining, gradual rollout.
- **Idle timeouts killing connections** (LB/proxy). Fix: heartbeats below the smallest idle timeout.
- **File descriptor / memory exhaustion.** Detect: `ulimit -n`, memory per connection. Fix: raise limits, shard gateways, cap per-user connections.
- **Sticky routing and uneven load.** Fix: connection-count-aware balancing, rebalance on deploy.
- **Missed messages during reconnect.** Fix: sequence numbers/last-seen id, replay buffer.
- **Auth on long-lived connections** (token expiry). Fix: re-auth messages, short-lived tokens refreshed in-band.

**Q&A.** *WebSocket vs SSE?* WebSocket is bidirectional; SSE is simpler, one-way, HTTP-native, auto-reconnect. *How do you broadcast to users on different WebSocket servers?* Pub/sub backplane (Redis/Kafka) plus user->node routing.

**Your turn (Q12).** Your notification WebSocket fleet is redeployed and the DB spikes for 2 minutes. Why, and how do you prevent it?

---

### 4.13 TLS, Proxies, CDN, VPC Networking, Private Endpoints, Service Mesh, Egress `[Both basics, SDE2 depth]` P1

**What / how.**
- **TLS:** encrypts and authenticates; certificate chain validated against trusted CAs; **termination** at LB/proxy (offload; traffic inside needs protection) or end-to-end; **mTLS** (both sides present certs) for service-to-service identity. Certificate **rotation** should be automated (ACME/cert-manager).
- **Reverse proxy / ingress:** terminates TLS, routes by host/path, rate limits, caches, compresses (Nginx, Envoy, HAProxy, cloud LBs).
- **CDN:** caches content at edge locations close to users; cache keys (URL + selected headers/query), TTLs, purge/invalidation, origin shield.
- **VPC:** private network; **subnets** (public: route to internet gateway; private: no inbound from internet), **route tables**, **security groups/NACLs**, **NAT gateway** (private instances reach the internet outbound), **private endpoints/VPC endpoints** (reach cloud services without public internet/NAT).
- **Service mesh:** sidecar (or ambient) proxies give mTLS, retries, timeouts, traffic shifting, telemetry without app changes (Istio/Linkerd-style).

**Cost and latency effects (verify provider pricing).**
- **Egress to the internet** and **cross-region** transfer are typically charged per GB; **cross-AZ** traffic is often charged as well; intra-AZ often free.
- **NAT gateways** usually charge per hour *and* per GB processed: high-volume traffic to cloud services through NAT can be a big hidden cost; **private endpoints** can bypass NAT data-processing fees.
- Serving static assets via **CDN** reduces origin load and often egress cost.

```
Illustrative egress math: 50 TB/month to the internet at $0.08/GB  = 50,000 GB x $0.08 = $4,000/month
Same traffic served mostly from CDN cache (90% hit rate): origin egress ~5 TB => ~$400 + CDN charges
```

**Alternatives.** Direct public endpoints with IP allowlists (weaker), VPN/peering/transit gateway for private connectivity, mesh-less libraries (client libraries for retries/timeouts).

**Trade-offs.** Mesh = uniform policy, but extra latency/CPU/memory per pod and operational complexity. Private networking = security and cost benefits, more setup. CDN = speed/cost, but stale-content and invalidation issues.

**Production problems.**
- **Expired certificate => outage.** Detect: certificate-expiry monitoring (alert at 30/14/7 days), synthetic checks. Fix: automated renewal, ownership of every cert, inventory.
- **NAT SNAT port exhaustion.** Symptom: sporadic outbound connection failures at high connection rates. Fix: connection reuse, more NAT IPs/gateways, private endpoints.
- **Surprise NAT/egress/cross-AZ bill.** Detect: cost allocation by usage type, VPC flow logs. Fix: endpoints, same-AZ routing, CDN, compression, caching.
- **CDN serving stale/wrong content** (bad cache key, missing `Vary`, failed purge). Fix: versioned asset URLs (hash in filename), correct `Cache-Control`, purge automation.
- **Security group too open / public subnet DB.** Fix: least-privilege rules, private subnets, regular audits.
- **Mesh sidecar misconfiguration/overhead** breaking traffic or adding latency. Fix: staged rollout, resource limits, observability.
- **DNS failure/slow resolution** (CoreDNS overload, `ndots` misconfig causing many lookups). Fix: node-local DNS cache, trailing-dot FQDNs, scale CoreDNS.

**Q&A.** *Where should TLS terminate?* Typically at the edge LB/proxy, with re-encryption or mTLS internally for sensitive/zero-trust environments. *Why can NAT be expensive?* Per-GB processing charges for all outbound traffic through it.

**Your turn (Q13).** Your cloud bill shows a large "NAT Gateway data processed" line item from services reading S3-style object storage. Explain and fix.

---

### 4.14 Authentication vs Authorization; OAuth 2.0, OIDC, JWT, API Keys `[Both]` P0

**What.** *AuthN:* who are you? *AuthZ:* what may you do?

**How.**
- **OAuth 2.0:** delegated authorization via access tokens. Flows: **Authorization Code + PKCE** (web/mobile/SPA user login; the secure default), **Client Credentials** (service-to-service), refresh tokens for long sessions. **OpenID Connect (OIDC)** adds authentication on top (ID token, user info).
- **JWT:** signed JSON token (header.payload.signature). Validate **signature, `exp`, `iss`, `aud`, `nbf`** (and allowed algorithm); JWTs are **signed, not encrypted** by default; don't put secrets in them. Short-lived access tokens (minutes) + refresh tokens (rotate; revocable).
- **Sessions (opaque IDs in HttpOnly, Secure, SameSite cookies) + server store:** easy revocation, server lookup per request.
- **API keys:** identify an *application/client*, not a user; store only **hashes**; scope, rotate, rate-limit per key.
- **AuthZ models:** RBAC (roles), ABAC (attributes), ReBAC (relationships); enforce **server-side on every request**, including **object-level checks** (is this order yours?).

| | JWT (stateless) | Opaque session/token |
|---|---|---|
| Validation | Local (signature) | Lookup (cache/DB) |
| Revocation | Hard (wait for expiry or keep a denylist) | Easy |
| Scale | Great for many services | Central store dependency |

**Alternatives.** mTLS for service identity, SAML for enterprise SSO, passkeys/WebAuthn for phishing-resistant login, API gateway token introspection.

**Trade-offs.** Stateless tokens vs revocation; token lifetime (security vs refresh load); centralized authZ service (consistency, latency) vs library-based checks.

**Production problems.**
- **Broken object-level authorization (IDOR).** Symptom: users read others' data by changing ids. Detect: security tests, anomaly in access logs. Fix: check ownership/tenant on every object access, tenant-scoped queries.
- **JWT pitfalls:** accepting `alg: none`/wrong algorithm, not checking `aud`/`iss`, long lifetimes, tokens stored in `localStorage` (XSS exposure; prefer HttpOnly cookies or in-memory with care). Fix: strict validation libraries, short expiry, rotation.
- **Clock skew** rejecting valid tokens. Fix: small leeway (e.g., 60 s), NTP.
- **Leaked API keys/tokens in logs, repos, URLs.** Fix: secret scanning, redaction, rotate immediately, never put tokens in query strings.
- **Missing revocation after compromise/logout.** Fix: short-lived tokens, refresh-token rotation and reuse detection, denylist.
- **OAuth misconfig** (open redirect URIs, missing `state`/PKCE). Fix: exact redirect URI matching, PKCE, `state`.
- **Auth service outage** blocking everything. Fix: HA, local JWT validation with cached keys (JWKS), graceful degradation.
- **Confused deputy / privilege escalation** in service-to-service calls. Fix: propagate end-user identity, least-privilege service accounts.

**Q&A.** *AuthN vs AuthZ?* Identity vs permissions. *How do you revoke a JWT?* Short expiry, deny-list of token IDs, rotating refresh tokens, or switch to opaque tokens for high-risk systems. *Why PKCE?* Protects the authorization code from interception for public clients.

**Your turn (Q14).** Design auth for a B2B SaaS: browser app, mobile app, public API for customers, and internal microservices. Choose flows and token types.

---

## Part D: Review

### Summary

Compute choices set the **cost curve and failure modes** (cold starts, throttling, OOM). Distributed systems are about **failure handling**: timeouts everywhere, retries only with backoff/jitter/budgets, circuit breakers and bulkheads to contain damage, queues with DLQs and lag alerts, **idempotency** for duplicates, sagas instead of distributed ACID, fencing tokens instead of trusting TTL locks. Networking drives **latency, security, and cost** (connection reuse, egress, NAT, certificates). Most real outages in this part come from: *unbounded resources, missing timeouts, retry amplification, expired certs, scaling dependencies (DB connections) without limits, and surprising bills.*

### Five review questions
- **R1.** Why can retries make an outage worse, and what three controls limit that?
- **R2.** Why isn't a Redis TTL lock sufficient for correctness?
- **R3.** Why is "exactly once" really "at-least-once + idempotency"?
- **R4.** Why can CPU limits hurt latency in Kubernetes even at low average CPU?
- **R5.** When is SSE better than WebSockets?

### Mock question
**M4.** "A payment API intermittently charges customers twice. It runs on 20 pods behind a load balancer, calls an external payment provider, and publishes `PaymentCompleted` to Kafka. Walk through every way duplicates can occur and how you'd fix and prevent each, end to end."

---

## Answer Key

**Q1.** (a) Nightly 10-minute job: serverless/batch container or spot VMs, paying only while running (check function max duration; otherwise container batch job). (b) Steady 800 RPS: containers (managed container service or Kubernetes) with autoscaling and commitments for baseline; per-request serverless pricing would exceed provisioned cost at sustained load (Part 3 break-even logic) and add cold-start risk. (c) Webhook receiver with 3 req/s average and bursts to 500: queue in front plus serverless/containers with autoscaling; accept webhook fast, enqueue, process async; provisioned/min-warm concurrency if burst latency matters; idempotent processing because webhooks retry.

**Q2.** Likely **CPU throttling**: a 1-CPU limit enforces a quota per 100 ms scheduling period; bursts (GC, request spikes, multi-threaded work) hit the quota and threads stop until the next period, spiking tail latency while the *average* looks low. Confirm via `cfs_throttled_periods`/throttled-time metrics. Fixes: raise or remove the CPU limit (keep an accurate request), reduce burstiness/thread count, tune GC/runtime concurrency to the quota, and scale out earlier.

**Q3.** The service is likely I/O-, thread-, or connection-bound; CPU stays low while a pool or downstream limit saturates, so CPU-based HPA never fires. Scale on a signal that tracks saturation: requests per second per pod, p95/p99 latency, in-flight requests/concurrency, thread-pool or connection-pool utilization, or queue depth. Also verify downstream capacity (DB), since scaling pods won't help if the bottleneck is elsewhere.

**Q4.** Workers are stateless and jobs retryable, so use **spot/preemptible** instances for most capacity, spread across instance types/AZs, with a small on-demand floor for baseline SLA; autoscale on queue depth. Safeguards: queue with visibility timeout/lease so interrupted jobs are retried, checkpoint long jobs (or chunk into 2-minute segments so loss is small), handle the interruption notice (stop taking new work, checkpoint, requeue), idempotent output paths, DLQ for repeated failures, max spend/instance caps, and a commitment only for the steady baseline if one exists.

**Q5.** Failures: pod restart/deploy/scale-in loses carts; sticky sessions unbalance load and hot-pin users; an instance crash logs users out; memory grows with active carts; can't scale in smoothly; multi-AZ failover loses state. Redesign: store carts in Redis (TTL) or a database keyed by user/session id, or in a signed client token if small; make the service stateless; use idempotent cart updates (set quantity rather than increment), handle concurrent updates with versions; remove stickiness.

**Q6.** With 12 partitions, a consumer group can only use 12 consumers in parallel; the other 18 pods are idle. Fixes: (1) increase partitions (plan for key-mapping change and order implications) and consumers; (2) make per-message processing faster (batch DB writes, async I/O, remove slow calls, parallelize *within* a consumer while respecting ordering per key); (3) fix hot keys/skewed partitions and slow downstream dependencies; also tune `max.poll` settings, and consider moving heavy work to a separate topic/worker tier. Monitor lag per partition.

**Q7.** Require an `Idempotency-Key` header. On request: insert `(key, request_hash, status=IN_PROGRESS)` under a unique constraint. If insert fails: if `DONE`, return the stored response; if `IN_PROGRESS`, return 409/`Retry-After` (or wait briefly); if the hash differs, return 422. Call the provider *passing the same idempotency key* so retries are deduplicated there. Save response and mark `DONE`. For a crash after the provider call but before saving: a retry re-calls the provider with the same key, which returns the original result instead of charging twice; additionally run a reconciliation job comparing provider records to local state, and expire old keys after the retry horizon.

**Q8.** C slowing to 5 s ties up threads in B and A for 5+ s each; with 3 retries at each layer, C receives up to 9 attempts per original request (3x3), amplifying load on an already-slow service; thread pools saturate, latency climbs to many seconds, and unrelated endpoints on A/B fail (cascading failure). Fix: tight timeouts with a budget (e.g., A 2 s > B 1.2 s > C 0.8 s), retry at only one layer with backoff+jitter and a retry budget, circuit breakers and bulkheads, fallbacks/degradation, load shedding, and deadline propagation so work stops when callers give up.

**Q9.** Orchestrated saga ordering by reversibility: perform reversible/cancellable steps first and the non-refundable ticketing last (pivot step), e.g., `HoldFlight(reservation, expires)` -> `BookHotel` -> `BookCar` -> `TicketFlight` (pivot, last). If the hotel fails before ticketing: release the flight hold (compensate). If ticketing already happened (non-refundable) and a later step fails: policy decision, either retry the failed step (with alternates: another hotel), or offer rebooking/credit; compensations exist where possible (cancel hotel/car) and the flight compensation is "issue credit/manual review". Make each step idempotent, persist saga state, use timeouts and a manual-intervention queue.

**Q10.** Both workers believed they were leader/held the lock: a TTL lock expired during a long GC pause or slow run, or two instances started concurrently on different hosts/time skew, or the cron ran on multiple replicas. Safe designs: (1) run the job through a **DB-backed unique constraint**: insert a row `(job_name, run_date)` with a unique key; only the inserter proceeds; make billing steps idempotent per invoice; (2) use a lease in a **consensus store (etcd/ZooKeeper)** with **fencing tokens** checked when writing; or schedule via a single-instance scheduler (K8s CronJob with `concurrencyPolicy: Forbid`) plus idempotent processing.

**Q11.** (1) **BFF/aggregation endpoint:** one request returns the home-screen payload assembled server-side (fewer round trips, tailored payload; trade-off: another service to maintain, coupling to UI). (2) **GraphQL:** client requests exactly what it needs in one query (flexible; trade-off: query cost controls, caching complexity, N+1 resolver risk). Also: HTTP/2/3, response compression, caching headers, prefetching, pagination.

**Q12.** All clients reconnect simultaneously (reconnect storm), each re-authenticating and loading state/backlog from the DB, a synchronized thundering herd. Prevent: connection draining with gradual rollout (rolling in small batches), client reconnect with exponential backoff + jitter, rate-limit reconnects/server-side admission control, cache session/presence state in Redis, replay small buffers instead of DB reads, and pre-warm capacity.

**Q13.** Traffic to object storage from private subnets goes through the NAT gateway and is billed per GB processed. Fix: use a **gateway/interface private endpoint** for the storage service (route table entry) so traffic bypasses NAT and public internet; also check cross-AZ routing, compress/cache hot objects, and use cost attribution (VPC flow logs) to confirm sources. Verify endpoint pricing for your provider; gateway endpoints for some services have no hourly/data charge, others (interface endpoints) do.

**Q14.** Browser app: Authorization Code + PKCE via an OIDC provider; short-lived access token (~5-15 min) with rotating refresh token, stored in HttpOnly SameSite cookies (BFF pattern) rather than `localStorage`. Mobile: Authorization Code + PKCE with system browser, tokens in secure OS storage. Customer public API: per-tenant **API keys** (hashed at rest, scoped, rotatable, rate-limited) or OAuth client credentials for higher assurance. Internal microservices: mTLS or signed service tokens (client credentials), propagate end-user identity/tenant claims, least-privilege service accounts. Authorization: RBAC with tenant scoping, object-level checks in every service, centralized policy where practical. Add audit logging, token rotation, rate limiting, and secret management.

**R1.** Synchronized or stacked retries multiply load on a dependency that is already struggling (e.g., 3 layers x 3 attempts = 27x), turning a blip into a sustained outage. Controls: exponential backoff with jitter, retry budgets/only one retry layer, circuit breakers (plus load shedding and deadlines).

**R2.** TTL locks expire independent of the holder's progress; a paused/slow holder can continue writing after losing the lock, so two actors proceed. Use fencing tokens checked by the resource, consensus-backed locks, or idempotent/optimistic designs.

**R3.** Networks can lose acknowledgments, so senders must retry (at-least-once); duplicates are then neutralized by idempotent handlers/dedupe keys, which gives effectively-once *outcomes*.

**R4.** CPU limits are enforced as quota per scheduling period; bursts consume the quota quickly and threads are paused until the next period, increasing tail latency even when the long-run average is low.

**R5.** When you only need server-to-client push (notifications, progress, LLM streaming): SSE is simpler, runs over plain HTTP, auto-reconnects, and passes proxies/CDNs more easily; no need for bidirectional messaging.

**M4 (outline of a strong answer).** *Duplicate sources:* (1) client/network retries (timeout then retry, double-click); (2) load balancer or gateway retries; (3) service retries to the provider after timeout where the first call actually succeeded; (4) crash after provider success before persisting; (5) Kafka producer retry without idempotence or outbox causing duplicate events; (6) consumer redelivery (rebalance, commit-after-crash) causing duplicate downstream effects (e.g., two ledger entries or emails); (7) concurrent duplicate requests hitting different pods simultaneously. *Fixes:* client-supplied idempotency key enforced by a unique constraint with `IN_PROGRESS/DONE` states and stored responses; pass the key to the provider; transactional outbox for `PaymentCompleted` (same DB transaction as state change) with relay publishing at-least-once; idempotent producer settings (`enable.idempotence`, `acks=all`); idempotent consumers keyed by event/payment id with unique constraints or "apply if version greater"; commit offsets after processing; reconciliation job against provider reports; one retry layer with backoff and no retries on non-idempotent calls without keys. *Prevent/detect:* metrics for duplicate-key hits, redelivery counters, daily reconciliation, chaos tests (kill pods mid-flow, duplicate messages), alerts on duplicate payment ids, and runbooks for refunds.

---

### Next: Part 5 of 13
Design patterns, architecture patterns, LLD, and concurrency (Sections 5 and 20).
