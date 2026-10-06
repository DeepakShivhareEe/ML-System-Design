# 1. System Design Fundamentals

> **Section 1 of 10** · Questions 1–7 · Estimated study time: 3–4 hours

---

## In this chapter

| # | Question | Key takeaway |
|---|----------|--------------|
| 1 | What is ML system design? | Designing the *system around* the model, not the model itself |
| 2 | How is it different from traditional system design? | Data dependencies, feedback loops, probabilistic outputs |
| 3 | Major components of an end-to-end ML system | Ingestion → training → serving → monitoring → retraining |
| 4 | Translating a business problem into an ML problem | Business goal → ML task → target definition → data check |
| 5 | Defining objectives and success metrics | Pair a primary metric with guardrail metrics |
| 6 | Offline vs online metrics | Offline predicts quality; online measures value |
| 7 | Requirements to clarify before designing | Functional + non-functional + constraints + data |

---

## Q1. What is ML system design?

**Answer.**
ML system design is the discipline of designing the **complete end-to-end infrastructure, data flow, and processes** required to build, deploy, monitor, and maintain a machine learning model in production — treating the model as one component inside a much larger system.

It answers questions such as:

- Where does the data come from, and how does it flow to the model?
- How is the model trained, evaluated, versioned, and deployed?
- How does the model serve predictions at scale, within latency and cost budgets?
- How do we detect when the model degrades, and how does it improve over time?
- What happens when something fails — and something *will* fail?

**What it is NOT:**
- It is not about inventing new model architectures (that is ML research).
- It is not about tuning hyperparameters (that is ML engineering at the model level).
- It is not about writing one training notebook (that is prototyping).

**Why it matters.** In real production systems, the model code is typically a small fraction — often estimated under 10–20% — of the total codebase (a widely cited observation from Google's *"Hidden Technical Debt in Machine Learning Systems"*, NeurIPS 2015). The surrounding infrastructure — data pipelines, feature computation, serving, monitoring — dominates the engineering effort and the failure modes.

```
                 ┌─────────────────────────────────────────────┐
                 │            THE ML SYSTEM ( iceberg )        │
      visible ──►│  ┌───────────────────────┐                  │
                 │  │   ML MODEL / CODE     │   ~10–20%        │
                 │  └───────────────────────┘                  │
      hidden  ──►│  data collection · verification · feature   │
                 │  engineering · training infra · serving     │
                 │  infra · monitoring · retraining loops      │
                 └─────────────────────────────────────────────┘
```

**What an interviewer checks in an ML system design round:**

1. Can you **scope** the problem and make reasonable assumptions?
2. Do you think about **data first** (availability, volume, freshness, labels)?
3. Can you design for **both training and serving**, and keep them consistent?
4. Do you reason about **trade-offs** — latency vs accuracy, cost vs freshness, complexity vs maintainability?
5. Do you include **evaluation, monitoring, and iteration**, i.e. the loop, not just the pipeline?

**Mental model to carry through the whole repo:**

> *An ML system is not a model. It is a data pipeline that produces a model, a serving path that consumes it, and a feedback loop that improves it — all running continuously, under constraints, in production.*

---

## Q2. How is ML system design different from traditional system design?

**Answer.**
Traditional system design (URL shorteners, chat apps, rate limiters) deals with **deterministic** behavior: same input → same output, correctness is verifiable, and the main enemies are load, latency, and failure. ML system design inherits all of that **plus a second set of problems caused by data and statistics**.

| Dimension | Traditional system | ML system |
|---|---|---|
| **Core logic** | Hand-written deterministic rules | Learned statistical function `f(x)` |
| **Output** | Exact, deterministic, verifiable | Probabilistic, approximate, imperfect by design |
| **Correctness** | Can be unit-tested (input → expected output) | Cannot be fully tested; measured statistically on samples |
| **Behavior over time** | Stable unless code changes | Can silently degrade **without any code change** (data/concept drift) |
| **Primary dependency** | Code + hardware | Code + hardware + **data quality + distribution** |
| **Deployment** | Ship the code, done | Ship the code *and* the model *and* its data dependencies; versions can disagree |
| **Feedback loops** | Rare (system rarely changes user behavior) | Common — the model's own predictions change future training data |
| **Monitoring** | Uptime, latency, error rates | Those **plus** data drift, prediction distribution, label lag, business metrics |
| **Failure mode** | Loud: crash, timeout, 5xx | Often **silent**: returns confident but wrong/stale predictions |
| **Testing** | Unit/integration tests suffice | + data validation, model evaluation, shadow/canary testing, bias checks |
| **Retraining** | N/A | Continuous or scheduled lifecycle; every deployment strategy applies to *weights* too |
| **Versioning** | Code version | Code version **+ model version + feature version + data snapshot** |

**Key consequences that drive design decisions:**

1. **Data is a first-class citizen.** Pipelines must validate, version, and monitor data the way code is linted, tested, and reviewed.
2. **Training–serving consistency** becomes a real engineering problem (see [03 - Data & Feature Engineering.md](03 - Data & Feature Engineering.md) Q18): features computed offline in Python/batch must match features computed online in milliseconds.
3. **Silent failure** means monitoring must watch *distributions*, not just *health* — a service returning 200s with garbage predictions is "up" but broken.
4. **Feedback loops** — e.g. a recommendation model changes what users see, which changes what users click, which changes the training data — can bias the model over time and must be understood (exploration, randomization, logging of unbiased data).
5. **Rollback is harder** — rolling back model weights may also require rolling back features and data snapshots in lockstep.

**How to say it in an interview:**
> "Everything in traditional design still applies — scale, latency, reliability — but ML adds three new axes: the system depends on data that changes, its output is stochastic and only statistically verifiable, and its own behavior feeds back into its future training data. So I design for validation, consistency between training and serving, and continuous monitoring and retraining."

---

## Q3. What are the major components of an end-to-end ML system?

**Answer.** A production ML system has **12 canonical components**. Memorize this list — every design case in [10 - System Design Cases.md](10 - System Design Cases.md) is a variation of it.

```
┌─────────────┐   ┌─────────────┐   ┌──────────────┐   ┌───────────────┐
│ Data Sources│──►│  Ingestion  │──►│ Data Storage │──►│ Data          │
│ (events, DBs│   │ (Kafka/Kin- │   │ (Lake/Ware-  │   │ Processing &  │
│ 3P APIs)    │   │  esis/CDC)  │   │  house)      │   │ Validation    │
└─────────────┘   └─────────────┘   └──────────────┘   └──────┬────────┘
                                                              ▼
┌──────────────┐   ┌──────────────┐   ┌───────────────┐   ┌───────────────┐
│   Model      │◄──│  Training    │◄──│    Feature    │◄──│               │
│   Registry   │   │  Pipeline    │   │    Store      │   │  (continued)  │
└──────┬───────┘   └──────▲───────┘   └───────────────┘   └───────────────┘
       ▼                  ▲
┌──────────────┐   ┌──────┴───────┐
│   Model      │   │  Monitoring  │◄──────────────────────────────┐
│   Deployment │   │  & Alerting  │───────────────┐               │
└──────┬───────┘   └──────────────┘               │               │
       ▼                                          │               │
┌──────────────┐                           ┌──────┴──────┐  ┌─────┴────────┐
│  Inference / │──────────────────────────►│ Predictions │  │  Retraining  │
│  Serving     │                           │  & Feedback │  │  Trigger     │
└──────────────┘                           └─────────────┘  └──────────────┘
```

| # | Component | Purpose | Typical tools |
|---|-----------|---------|---------------|
| 1 | **Data sources** | Where raw data originates: user events, transactional DBs, 3rd-party APIs, logs, images | App SDKs, OLTP DBs, SaaS APIs |
| 2 | **Data ingestion** | Reliable transport into the platform; batch loads or streaming | Kafka, Kinesis, Pub/Sub, Fivetran, Spark/Hive jobs, CDC (Debezium) |
| 3 | **Data storage** | Durable landing of raw + processed data | Data lake (S3/GCS + Delta/Iceberg), warehouse (BigQuery/Snowflake/Redshift), feature store |
| 4 | **Data processing & validation** | Cleaning, joins, aggregations, schema & quality checks | Spark, Flink, dbt, Great Expectations, Deequ |
| 5 | **Feature engineering** | Transform raw data into model inputs; compute once, reuse in training & serving | Feature stores (Feast, Tecton, SageMaker Feature Store) |
| 6 | **Training pipeline** | Data → features → train → validate → produce model artifact; orchestrated, reproducible | Kubeflow, Airflow, MLflow Projects, SageMaker, Ray |
| 7 | **Model evaluation** | Offline metrics on held-out sets + slices + comparison vs incumbent | Custom + MLflow/Evidently metrics, offline replay |
| 8 | **Model registry** | Versioned store of model artifacts + metadata + lineage + stage (staging/prod) | MLflow Registry, SageMaker Model Registry, Vertex Model Registry |
| 9 | **Model deployment** | Move an approved model into serving infra safely | CI/CD + blue-green/canary/shadow ([08 - Model Deployment Strategies.md](08 - Model Deployment Strategies.md)) |
| 10 | **Inference / serving** | Low-latency prediction API or batch scoring jobs | KServe, TorchServe, Triton, TF Serving, custom FastAPI/gRPC |
| 11 | **Monitoring & alerting** | System health + data drift + model quality + business metrics | Prometheus/Grafana, Evidently/Arize/WhyLabs, cloud-native |
| 12 | **Retraining loop** | Automated trigger → new data → retrain → evaluate → redeploy | Orchestrator (Airflow/Kubeflow) + registry + CI/CD |

**How to present it in an interview:** draw the two rows — *"offline: data → features → training → evaluation → registry"* and *"online: serving → predictions → feedback → monitoring → retraining"* — and say every design decision you make afterward belongs to one of those two paths. That immediately signals structure.

**Depth control:** mention all 12, then go deep only on the 2–3 that matter for the given case (e.g. feature store + freshness for fraud detection; monitoring + retraining for recommendations).

---

## Q4. How do you translate a business problem into an ML problem?

**Answer.** Use a **five-step framing loop**. Interviewers reward explicit scoping more than exotic models.

**Step 1 — Understand the business goal and quantify it.**
"Not enough users finish checkout" → business goal: increase checkout conversion by X%. Always ask: what does success look like *in money, retention, or user satisfaction*?

**Step 2 — Decide if ML is even needed.**
ML is justified when the problem is (a) a *prediction/pattern-recognition* task, (b) too complex for hand-written rules, and (c) data exists or can be collected. If 10 rules capture 95% of it, ship rules. This trade-off is revisited in [09 - Cost, Security & Practical Trade-offs.md](09 - Cost, Security & Practical Trade-offs.md) Q58.

**Step 3 — Formalize as an ML task.** Map goal → task type → input/output:

| Business problem | ML formalization | Output |
|---|---|---|
| Reduce customer churn | Binary classification (churn next 30d?) | P(churn) |
| Increase e-commerce revenue | Learning-to-rank over catalog items | Ordered list of items |
| Block fraudulent payments | Binary classification with extreme imbalance + real-time budget | P(fraud) + decision |
| Auto-tag support tickets | Multi-class / multi-label text classification | Label set |
| Forecast demand per SKU | Time-series regression | Quantity per SKU-week |
| "Answer questions over company docs" | RAG: retrieval + LLM generation | Grounded answer + citations |
| Personalize video feed | Two-tower retrieval + ranking, engagement prediction | Ranked feed |

**Step 4 — Define the prediction target precisely (the step people skip).**
- **Unit of prediction:** one row = one user? one user×item pair? one session?
- **Time semantics:** predict *at time t* using only information available *before t* (point-in-time correctness).
- **Label window:** churn "in the next 30 days" — 30 must be fixed.
- **Watch for proxy-target bugs:** "deleted account" ≠ "churned" (dormant users); "clicked" ≠ "satisfied" (clickbait bias).

**Step 5 — Check feasibility against data and constraints.**
Data available? volume? labels — do they exist, or must they be generated (logs, annotation, weak labels)? latency budget (fraud = <100 ms; weekly demand forecast = hours)? regulatory constraints (fair lending, GDPR)? Then state success criteria (Q5) and only *then* start designing.

**One-paragraph interview template:**
> "The business goal is X, measured by Y. I'll formalize it as a [task type] where the model predicts [target] for [unit] using data available at prediction time. The label is defined as [event] within [window]. Success means [primary model metric] improving while [guardrail business metric] holds. The constraints are [latency / cost / regulation / fairness]."

---

## Q5. How do you define the objective and success metrics for an ML system?

**Answer.** A rigorous metric definition has **four layers**:

**1. Business objective (north star).** Revenue, retention, cost saved, user trust. Non-negotiable anchor.

**2. Primary model metric (optimization target).** The metric the model/training actually optimizes. Must be:
- Aligned with the business objective (predicting the proxy you actually care about);
- Computable offline, reasonably quickly, on held-out data;
- Robust to the data's quirks (imbalance → PR-AUC not accuracy; ranking → NDCG/MRR not accuracy).

| Task | Typical primary metrics |
|---|---|
| Binary classification (balanced) | ROC-AUC, F1 |
| Binary classification (imbalanced: fraud, spam) | PR-AUC, recall @ fixed precision, Fβ |
| Regression (pricing, demand) | MAE (robust, interpretable), RMSE (penalizes big misses), MAPE (scale-free) |
| Ranking (search, recsys) | NDCG@k, MAP, MRR, recall@k → then precision@k |
| Retrieval (RAG) | Recall@k / hit-rate of retrieved set |
| Generation (LLM) | Task win-rate, groundedness/citation precision, human/LLM-judge score |
| Forecasting | WAPE/MASE per series group |

**3. Guardrail metrics (don't-break-the-business).** Held constant while optimizing the primary metric. Examples: latency p99, cost/query, complaint rate, diversity of recommendations, unsubscribe rate, fairness slices (accuracy parity across demographic groups).

**4. Operational SLOs.** Availability (99.9%), latency p50/p95/p99, error rate, freshness of features.

**Rules of thumb:**
- **One primary metric + 2–4 guardrails.** If everything is a priority, nothing is.
- Metrics must survive a **sad-path test**: "If this number goes up but revenue goes down, would we notice?" If not, add the connecting business metric to the dashboard.
- Define the **minimum lift** that justifies deployment (e.g. +0.5% conversion with p<0.05 in an A/B test, no guardrail regressions).
- Beware **Goodhart's law**: when a measure becomes a target, it ceases to be a good measure (optimize CTR → clickbait). Guardrails and human review counteract this.

**Interview one-liner:**
> "I'd optimize one primary offline metric aligned to the north star, hold 2–3 guardrail metrics fixed, validate the lift online with an A/B test, and only ship if the business metric improves without guardrail regressions."

---

## Q6. What is the difference between offline ML metrics and online business metrics?

**Answer.**

| | **Offline metrics** | **Online metrics** |
|---|---|---|
| Computed on | Historical held-out / validation data | Live traffic in production |
| Measures | Statistical quality of predictions | Actual user/business value |
| Latency of signal | Minutes–hours after training | Days–weeks (need traffic + labels) |
| Cost to obtain | Cheap, repeatable, no user impact | Expensive: needs real users, A/B split, can lose money |
| Causality | Correlational (historical patterns) | Experimental → **causal** (randomized treatment) |
| Examples | AUC, F1, NDCG@10, RMSE, perplexity, recall@k | Conversion rate, revenue/session, retention D7, CTR, latency p99, cost/day |
| Used for | Gating: "is this model worth testing?" | Decision: "do we ship this model?" |

**Why both are needed — the offline→online gap.** Offline gains routinely fail to materialize online because:

1. **Distribution shift** — validation data ≠ tomorrow's live traffic.
2. **Proxy mismatch** — offline metric (click prediction AUC) ≠ business goal (revenue).
3. **Feedback loops** — the new model changes what users see; historical data never contained such states.
4. **System effects** — latency, ordering, caching, and UI position change user behavior in ways offline replay can't capture.
5. **Aggregate masking** — offline averages hide that gains concentrate in one slice and losses in another (always report **slices**: new vs returning users, geos, platforms).

**Standard gating flow:**

```
Train → offline eval (cheap gate) → shadow (no user impact)
      → A/B test (causal gate) → ship if primary ↑ and guardrails hold
```

**Interview nuance worth saying:** "Offline metrics are for *iteration speed*, online metrics are for *truth*. I'd never ship on offline numbers alone, but I'd never run an A/B for every experiment either — offline gates protect users and budget from weak candidates."

---

## Q7. What requirements should you clarify before designing an ML system?

**Answer.** Before drawing any box, ask (or state as assumptions) in **four groups**. In interviews, spend the first 5 minutes here — it prevents designing the wrong system entirely.

**A. Functional requirements**
1. What exactly does the system predict/generate, for whom, and when? (unit of prediction)
2. Inputs available at prediction time — which features, from which sources, how fresh?
3. Output semantics: probability, class, ranking, generated text? Who consumes it (human UI, another service, downstream decision)?
4. Decision policy: is the model output thresholded to an action (block/allow), shown to users (ranked feed), or logged for analysts?
5. Personalization required? Per-user or global model?

**B. Non-functional requirements (numbers, not adjectives)**
1. **Scale:** QPS at average and peak; users; events/day; data volume (TB/day?).
2. **Latency:** p50/p99 budget; does it include feature retrieval and network?
3. **Availability:** 99.9%? Can predictions be stale or cached during an outage?
4. **Freshness:** how stale may a prediction/feature be? (1 h-old recommendations OK; 1 h-old fraud signal is not.)
5. **Throughput of data:** events/sec, training-set size, retraining cadence.

**C. Constraints**
1. **Cost budget** — inference cost per 1k predictions is often the binding constraint.
2. **Latency vs accuracy** — a 2 ms model at 90% quality often beats a 200 ms model at 93%.
3. **Compliance & privacy** — GDPR/CCPA (deletion, purpose limitation), HIPAA/PCI, model explainability mandates (credit scoring).
4. **Fairness** — protected attributes, slice-level performance floors.
5. **Team & ecosystem reality** — existing cloud, on-call maturity, batch-only vs streaming experience.
6. **Label availability & delay** — determines whether real-time retraining is even possible.

**D. Success criteria & lifecycle**
1. Primary metric + guardrails + minimum acceptable lift (Q5).
2. Rollout plan: shadow → canary → full ([08 - Model Deployment Strategies.md](08 - Model Deployment Strategies.md)).
3. Rollback policy and owner.
4. Expected model lifecycle: how often retraining, who approves, what triggers it ([07 - Monitoring, Drift & Retraining.md](07 - Monitoring, Drift & Retraining.md)).

**Red-flag answers that show seniority:**
- "If labels arrive days later, real-time retraining isn't the right requirement — we'd retrain daily on D-1 labels."
- "Before building a streaming feature pipeline, is batch freshness of 1 h acceptable? It's 10× cheaper."
- "If the product can tolerate it, we serve a cached fallback during model outages — availability of the *product* over availability of the *model*."

---

## Chapter 1 — Interview checklist

- [ ] I can define ML system design as *system around the model*, and cite the ~10–20% model-code observation.
- [ ] I can contrast traditional vs ML design on at least 5 dimensions (drift, silent failure, feedback loops, versions, testing).
- [ ] I can draw the 12-component architecture in <2 minutes and name tools for each.
- [ ] I can turn a vague business goal into task + target + label definition in one paragraph.
- [ ] I always pair a primary metric with guardrails, and separate offline gating from online truth.
- [ ] I open every design with functional, non-functional, constraint, and lifecycle requirements.

**Next → [02.md — End-to-End ML Architecture](02.md):** the same components, now designed concretely: pipelines, storage choices, training/inference consistency, and continuous retraining.
