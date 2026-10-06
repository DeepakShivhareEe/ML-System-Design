# 6. Scalability, Reliability & Availability

> **Section 6 of 10** · Questions 36–41 · Estimated study time: 3–4 hours
> The SLO chapter: keeping predictions fast, correct, and *available* — under load, failure, and budget pressure.

---

## In this chapter

| # | Question | Key takeaway |
|---|----------|--------------|
| 36 | High availability for an ML system | Redundancy + degradation ladder + progressive delivery |
| 37 | Horizontally scaling inference | Stateless tiers, SLO autoscaling, pre-warm, shared-dependency capacity |
| 38 | Traffic spikes | Pre-warm, load-shed, queue, degrade — prioritized admission control |
| 39 | Fault tolerance by design | Redundancy, bulkheads, backpressure, chaos drills |
| 40 | Reducing inference latency | Fix tails: profile, compile, batch with deadlines, cache, cascade |
| 41 | Latency vs accuracy vs cost | The production triangle; quantify with cost-of-delay |

**The canonical serving path (from [05 - Online Inference & Serving.md](05 - Online Inference & Serving.md), reused all chapter):**

```
Client → Load Balancer → API Service → Feature Retrieval → Model Server → Prediction → Response
```

---

## Q36. How would you make an ML system highly available?

**Answer.** Availability for an ML system = availability of the **whole prediction path** — features, model, policy — plus the **product's** ability to survive ML failure. Classic HA techniques + ML-specific degradation.

**1. Eliminate single points of failure, tier by tier:**
- **LB:** redundant pair / managed anycast LB across AZs.
- **API tier:** ≥2 replicas per AZ; stateless; rolling deploys.
- **Model tier:** N+1 warm replicas per AZ; models versioned and locally cached; no replica depends on a shared NFS to load.
- **Feature tier:** the often-forgotten dependency — Redis/DynamoDB clusters replicated across AZs with failover; local in-process caches cushion blips ([05 - Online Inference & Serving.md](05 - Online Inference & Serving.md) Q31).
- **Data plane:** managed multi-AZ Kafka/warehouse; serving must not *synchronously* depend on anything batch.
- **Multi-region** when the SLO demands: active-active with regional stacks (feature stores, models), latency-based routing, defined drain procedures.

**2. Health, readiness, and rollback (contain the failures you cause yourself):**
- Health checks that exercise real inference ([05 - Online Inference & Serving.md](05 - Online Inference & Serving.md) Q35); readiness gates before traffic; auto-rollback on canary regression ([08 - Model Deployment Strategies.md](08 - Model Deployment Strategies.md) Q49/Q53).
- Deploy during low-traffic windows; keep the previous model warm (blue-green) for instant revert.

**3. Graceful degradation ladder (pre-decided, pre-built, chaos-tested):**

```
L0 full model path → L1 cached/last-known-good scores → L2 warm fallback model
→ L3 business rules / popularity defaults → L4 functional degradation
```
Availability math worth quoting: with per-AZ failure probability p and enough replicas, redundancy converts AZ loss from an outage into a capacity dip — *if* the remaining fleet can absorb the load (hence headroom + load-shedding).

**4. Protect against silent failures (ML-specific):** a system returning fast 200s with stale features or broken scores is "up" but wrong. HA therefore includes: feature-freshness alarms, score-distribution monitoring ([07 - Monitoring, Drift & Retraining.md](07 - Monitoring, Drift & Retraining.md) Q45), and fallback-rate as an SLO (alert when >X% of traffic degrades).

**5. Capacity for failure:** headroom ≥ 1 AZ's worth of replicas; load tests at N-1 capacity; autoscaling policies tested with drills, not assumed.

**Interview sound bite:**
> "HA = redundant warm capacity across AZs for every tier — including the feature store — health checks that exercise real inference, progressive delivery with auto-rollback, and a chaos-tested degradation ladder so the *product* survives ML failure. And because ML fails silently, fallback rate and feature freshness are themselves SLOs."

---

## Q37. How would you horizontally scale an ML inference service?

**Answer.** Horizontal scaling = replicate stateless units behind a load balancer + automate capacity. The ML-specific wrinkles are model loading time, GPU sharing, and the feature store being a shared ceiling.

**1. Make the unit of scale stateless:**
- API service: no sticky sessions, no local mutable state; request context in the request, state in stores.
- Model server: models loaded from versioned artifacts (immutable), no per-replica training or stateful caches that must be coherent.

**2. Autoscaling done right:**
- Metrics: QPS-per-replica, CPU/GPU utilization, or *latency-SLO-driven* scaling (scale when p99 approaches budget — the most user-aligned signal).
- **Scale-out lag is the killer:** a new replica must pull the model artifact (GBs) and warm caches before serving → pod startup 30 s–5 min. Mitigations: pre-warmed pools, model images baked into the container (not pulled at boot), predictive scaling before known peaks, fast artifact local cache.
- Scale-in stability: slow scale-down, drain connections, avoid flapping on bursty traffic.

**3. Size the shared dependencies with the fleet:** the feature store (Redis cluster QPS), the prediction log (Kafka partitions), and the LB all scale *with* replicas — the fleet is only as scalable as its slowest shared tier ([05 - Online Inference & Serving.md](05 - Online Inference & Serving.md) Q34).

**4. Partitioning & routing options as scale grows:**
- Heterogeneous pools per model class (CPU pool for GBDT, GPU pool for NNs) and per priority tier.
- Consistent-hash routing by entity when per-entity caches help (cache-aware LB).
- Cell-based architecture at extreme scale: independent cells (LB + API + models + local feature cache) failing independently; a cell's blast radius is itself.

**5. Efficiency compounding:** every 2× model speedup (compile/quantize/distill, [09 - Cost, Security & Practical Trade-offs.md](09 - Cost, Security & Practical Trade-offs.md) Q54) halves the fleet — horizontal scaling and per-request cost reduction multiply, they don't substitute.

**Capacity math template:** replicas = peak QPS × p99 service time × safety factor (1.3–2) ÷ per-replica concurrency; then load-test N-1.

**Interview sound bite:**
> "Stateless replicas behind LBs with latency-SLO-driven autoscaling; the wrinkle ML adds is scale-out lag from model loading — so I bake models into images, keep pre-warmed pools, and scale predictively. I size shared tiers — feature store, logging — with the fleet, split pools by model class and priority, and let model-efficiency gains multiply the fleet."

---

## Q38. How would you handle traffic spikes?

**Answer.** Spikes (launches, sales events, viral moments, retries) are capacity events *and* control events. Strategy: **predict, pre-scale, prioritize, degrade.**

**1. Predict & pre-scale:** known events get scheduled capacity (auto-scaling by calendar); predictive scaling on leading indicators (marketing calendar, queue depth); keep burst headroom (~30–50%).

**2. Absorb (queues for what can wait):** asynchronous/batch workloads (reporting scores, backfills) go through queues with backpressure — they stretch, the interactive path doesn't. Interactive traffic never queues unboundedly; it sheds.

**3. Shed with priorities (admission control):**
- Classify traffic: business-critical (checkout fraud) vs important (feed ranking) vs deferrable (recs widget refresh).
- Under overload: protect p99 for high tiers by shedding/rejecting or degrading low tiers *early* — a fast cached answer beats a slow timeout for everyone behind you in the queue.
- Load-shed signals: queue depth, in-flight requests vs limit, CPU saturation — drop *before* latency explodes (hysteresis to avoid flapping).

**4. Degrade the response, not the product:** the [05 - Online Inference & Serving.md](05 - Online Inference & Serving.md) Q35 ladder applies under load too — cached predictions, smaller fallback model, rules-based defaults. Serving last-hour cached recs during a spike is invisible to users; 30-second timeouts are not.

**5. Defend against self-inflicted spikes:** exponential backoff + jitter on every client/retry path, circuit breakers, request coalescing (identical concurrent requests share one computation — thundering-herd protection), per-client rate limits so one misbehaving caller can't DDoS the fleet.

**6. Post-spike:** review which tier degraded first, whether alarms fired in the right order, and adjust capacity/shed rules. Spikes are the cheapest chaos test you'll ever get — instrument and learn from them.

**Interview sound bite:**
> "Predict and pre-scale what's schedulable; queue what can wait; under overload apply prioritized admission control — shed low tiers early to cached or rule-based fallbacks so critical p99 holds — and protect the system from itself with backoff, jitter, coalescing, and rate limits. Then treat every real spike as a drill report."

---

## Q39. How would you design fault tolerance into an ML system?

**Answer.** Fault tolerance = the system provides correct-enough service *despite* component failures. Design at three levels:

**1. Component level — redundancy + failover:**
- Replicas per tier across AZs ([Q36](#q36-how-would-you-make-an-ml-system-highly-available)); leader election / managed failover for stateful stores; N+1 capacity so failover is a dip, not an outage.
- Idempotent, retried operations with budgets (retries within deadline, exponential backoff + jitter); dedupe keys for at-least-once pipelines.

**2. Interaction level — contain cascades:**
- **Bulkheads:** separate connection/thread pools per dependency so a slow feature store can't exhaust the model server's threads.
- **Circuit breakers:** trip to fallback after error-rate thresholds; half-open probes for recovery.
- **Timeouts everywhere** with propagated deadlines; fail fast, fall back, never hang.
- **Backpressure** over unlimited queueing for async work; load-shedding for interactive work.

**3. ML-data level — the faults unique to ML:**
- **Bad data containment:** validation gates + quarantine ([03 - Data & Feature Engineering.md](03 - Data & Feature Engineering.md) Q16/Q17) so a corrupted upstream fails safe instead of training a broken model; training jobs consume only validated snapshots.
- **Bad model containment:** registry-gated deployment only; canary auto-abort; previous model kept warm for instant rollback; the degradation ladder covers "model present but wrong."
- **Stale everything:** if retraining stops (pipeline down), the champion keeps serving — models don't expire in hours. Alert on pipeline staleness separately from serving health.
- **Label/feedback failures:** missing labels degrade monitoring quality gradually — monitor label-arrival rates too.

**4. Prove it, don't hope it:** chaos drills — kill model pods, blackhole Redis, corrupt a canary's features, throttle the prediction log — and assert the ladder engages and SLOs hold. Game days beat runbooks nobody has opened ([05 - Online Inference & Serving.md](05 - Online Inference & Serving.md) Q35).

**Interview sound bite:**
> "Redundant failover per tier, bulkheads and circuit breakers between dependencies, deadlines with fallbacks everywhere — plus ML-specific containment: validation gates stop bad data at the boundary, registry gates and canary auto-abort stop bad models, and stale models keep serving while pipelines are down. Then I chaos-test the whole ladder."

---

## Q40. How would you reduce ML inference latency?

**Answer.** A prioritized playbook — measure first, then attack in this order (biggest wins first in most systems):

**1. Measure at p99 per stage** (feature fetch / preprocess / model / postprocess) — never optimize an aggregate; [05 - Online Inference & Serving.md](05 - Online Inference & Serving.md) Q33 lists the decomposition.

**2. Feature path (usually #1):**
- Parallelize all feature-group fetches; batch KV reads (mget/pipeline) instead of N round-trips.
- Local in-process cache for hot entities (TTL-bounded); colocate API and feature store in-region/AZ.
- Precompute aggregates offline ([03 - Data & Feature Engineering.md](03 - Data & Feature Engineering.md) Q21); shrink the feature vector to what the model uses.

**3. Model path:**
- **Compile & quantize:** ONNX Runtime/TensorRT; INT8/FP16 — often 2–5× on NNs at negligible accuracy loss (re-verify on slices!).
- **Distill** large teachers into small students when quality allows; pick smaller architectures sized to the latency budget.
- **Batching with a deadline:** dynamic batching (Triton) raises GPU throughput; cap queue wait to protect p99 ([05 - Online Inference & Serving.md](05 - Online Inference & Serving.md) Q31).
- Hardware fit: GBDT on CPU, NN on GPU; right-size, don't default. For LLMs: KV-cache reuse, continuous batching, streaming, speculative decoding.

**4. Demand side:**
- Cache predictions (keyed by model version + inputs) and product views (pre-ranked lists); TTL = staleness tolerance.
- Cascades: cheap path first, big model only for uncertain cases ([09 - Cost, Security & Practical Trade-offs.md](09 - Cost, Security & Practical Trade-offs.md) Q57).

**5. Tail-specific fixes:** warm pools + readiness gates (cold starts), GC tuning / ZGC or arena allocation (pauses), connection pooling/keep-alive (handshakes), hedged requests for stragglers on idempotent reads, shed before you stall.

**6. Re-measure and lock it in:** latency regression tests in CI (per-stage budgets fail the build), p99 dashboards per stage, load tests at peak shape.

**Interview sound bite:**
> "Profile p99 per stage, then in order: parallelize and cache feature fetches, compile/quantize/distill the model and batch with a deadline, cache predictions and cascade cheap-first, and kill tail sources — cold starts, GC, handshakes. Finally, per-stage latency budgets enforced in CI so regressions can't ship."

---

## Q41. How would you balance latency, accuracy, and infrastructure cost?

**Answer.** Name it as a **three-way trade governed by business value per request**, then make the trade explicit and quantified:

**1. Derive the value curve from the business:**
- What is one ms of p99 worth? (Conversion elasticity on checkout pages is real and measurable; a fraud decision delayed 100 ms blocks a payment queue.)
- What is one point of AUC worth? (Translate to fraud losses saved, incremental revenue from better ranking.)
- What does the fleet cost per 1M requests? (Compute + capacity headroom + ops.)
- These three curves *intersect* at the right operating point; most teams never draw them and overbuy accuracy they can't serve.

**2. Practical resolution order:**
1. **Fix the latency budget** from product requirements (p99 ≤ 100 ms on checkout, ≤ 300 ms on feeds).
2. **Choose the cheapest architecture meeting the budget** — cascade of cheap-first models with escalation on uncertainty.
3. **Buy accuracy within the budget:** bigger model only if offline lift survives the serving constraint; accuracy that can't be served at p99 is imaginary.
4. **Spend the cost budget on the highest-value traffic:** more compute for checkout-fraud, less for feed tail; batch-precompute the long tail ([Q30](#q30-batch-inference-vs-real-time-inference--when-would-you-use-each)).

**3. Techniques that relax the triangle instead of trading it:**
- **Distillation/quantization:** big-model quality at small-model latency (verify on slices).
- **Caching:** repeated inputs cost zero and add no latency.
- **Cascades:** mean accuracy of the *system* approaches the big model while mean latency/cost approach the small one.
- **Batch precompute:** infinite compute budget per prediction when freshness allows.
- **Async UX:** where product allows, compute during user dwell (pre-rank while the page renders).

**4. Governance:** cost and latency are guardrail metrics on the model scorecard ([04 - Model Training & Evaluation.md](04 - Model Training & Evaluation.md) Q26); every challenger must report all three axes; revisit quarterly as hardware (cheaper inference) and models (better small models) move the frontier.

**Worked micro-example (state one like this):** fraud at checkout — budget p99 100 ms, $/1k capped. Solution: rules (0.5 ms) → GBDT with 40 features (5 ms) catches 92% of fraud; a heavy DL ensemble adds +0.7% but needs 80 ms — deploy it only for transactions >$500 (cascade), keeping mean cost near the GBDT path while the high-value tail gets the big model. Total: accuracy of the ensemble where it pays, cost of the GBDT everywhere else.

**Interview sound bite:**
> "I set the latency budget from the product, then spend the cost budget where business value is highest: cascades with cheap-first escalation, distillation and caching to relax the trade, batch precompute for the long tail. Every challenger reports accuracy, p99, and $/1k — a model that wins offline but blows the budget loses."

---

## Chapter 6 — Interview checklist

- [ ] I can walk the HA story tier-by-tier including the feature store, plus ML-specific silent-failure SLOs.
- [ ] I can scale horizontally with the model-loading lag called out, and size shared dependencies with the fleet.
- [ ] I have a spike playbook: pre-scale → queue → prioritize → shed → degrade.
- [ ] I design fault tolerance at component, interaction, and ML-data levels — and chaos-test it.
- [ ] I reduce latency stage-by-stage at p99, with CI-enforced budgets.
- [ ] I resolve latency/accuracy/cost with business value curves, cascades, and a worked example.

**Next → [07.md — Monitoring, Drift & Retraining](07.md):** keeping the system honest after it ships.
