# 2. End-to-End ML Architecture

> **Section 2 of 10** · Questions 8–15 · Estimated study time: 4–5 hours
> Prerequisite: [01 - System Design Fundamentals.md](01 - System Design Fundamentals.md) Q3 (the 12 components). This chapter designs those components concretely.

---

## In this chapter

| # | Question | Key takeaway |
|---|----------|--------------|
| 8 | What does a typical end-to-end ML system look like? | The canonical diagram — be able to draw it cold |
| 9 | How does data flow from collection to prediction? | Five stages: capture → land → validate → featurize → serve |
| 10 | How would you design a training pipeline? | Orchestrated DAG: validate → features → train → eval → register |
| 11 | How would you design an inference pipeline? | Preprocess → retrieve → predict → postprocess → act, with fallbacks |
| 12 | Training-time vs serving-time infrastructure | Different goals: throughput/reproducibility vs latency/availability |
| 13 | Training–serving feature consistency | Same code path, point-in-time joins, or parity tests |
| 14 | Where data, features, models, predictions live | Match storage engine to access pattern |
| 15 | Continuous retraining | Scheduled + triggered retraining behind a gated deployment loop |

---

## Q8. What does a typical end-to-end ML system look like?

**Answer.** The canonical architecture, in one diagram. Practice drawing it in under two minutes with the ~15 boxes below; then annotate it per case.

```
                            ┌──────────────────────────────┐
                            │        DATA SOURCES          │
                            │ app events · OLTP DBs · 3P   │
                            └──────┬───────────────┬───────┘
                      batch (ETL)  │               │  streaming (CDC/queue)
                                   ▼               ▼
                   ┌──────────────────────┐   ┌──────────────────┐
                   │   DATA LAKE (raw)    │   │  STREAM / QUEUE  │
                   │  S3/GCS + Iceberg    │   │  Kafka / Kinesis │
                   └──────────┬───────────┘   └────────┬─────────┘
                              ▼                        │
                   ┌──────────────────────┐            │
                   │ VALIDATE / TRANSFORM │◄───────────┘
                   │  Spark · dbt · GE    │
                   └──────────┬───────────┘
                              ▼
                   ┌──────────────────────┐        ┌────────────────────┐
                   │   FEATURE STORE      │───────►│ TRAINING PIPELINE  │
                   │  offline + online    │ point- │  train · tune ·    │
                   └──────────┬───────────┘ in-time┘  evaluate          │
                              │              snapshot └─────────┬──────────┘
                              ▼                                 ▼
                   ┌──────────────────────┐        ┌────────────────────┐
                   │   MODEL REGISTRY     │───────►│   DEPLOYMENT       │
                   │ versions · lineage   │ deploy │  canary / shadow   │
                   └──────────▲───────────┘        └─────────┬──────────┘
                              │ register                     ▼
                              │                   ┌────────────────────┐
                              │                   │  INFERENCE SERVING │
                              │                   │  online + batch    │
                              │                   └─────────┬──────────┘
                              │                             ▼
                              │                   ┌────────────────────┐
                              └───────────────────┤ MONITORING +       │
                                  retraining      │ PREDICTION LOGGING │
                                  trigger         └────────────────────┘
```

**How to narrate it (this is the scoreable part):**
1. **Two data planes:** batch (lake) for history and training; streaming for freshness. They converge at validation/feature layers.
2. **Two ML planes:** the *training plane* (offline, throughput-oriented) and the *serving plane* (online, latency-oriented). The feature store and registry are the contracts between them.
3. **One control loop:** monitoring watches serving + labels, triggers retraining; new models re-enter through the registry and gated deployment. *The loop, not the pipeline, is the system.*

**Interview tip:** after drawing, immediately add the three cross-cutting concerns — prediction logging (for future training data), data validation (garbage in, garbage out at scale), and access control (features and predictions contain PII). Mentioning them unprompted is a senior signal.

---

## Q9. How does data flow from collection to prediction?

**Answer.** Five stages; each has latency, volume, and correctness pitfalls.

**Stage 1 — Collection (client & services → transport).**
- Events: page views, clicks, transactions, sensor readings — emitted with schema, timestamp, and IDs.
- Design points: **client-side buffering/batching**, **exactly-once or at-least-once semantics** (dedupe downstream by event ID), **timestamping at source** (event time vs processing time — critical for feature correctness), PII minimization at the edge.

**Stage 2 — Ingestion (transport → storage).**
- Streaming: Kafka/Kinesis/Pub/Sub; producers → partitions keyed by entity ID (user_id) so per-entity features aggregate in order.
- Batch: scheduled ETL pulls from OLTP replicas / APIs into the lake (incremental, partitioned by date).
- CDC (Debezium) streams row changes from operational DBs — the bridge between OLTP and analytics.

**Stage 3 — Storage & validation.**
- Raw immutable zone (lake), then validated/transformed zone after quality checks: schema conformance, null/range/distribution checks (Great Expectations/Deequ), dedupe, late-data handling (watermarks).
- Rule: **never train directly on raw**; always on the validated layer, so that a broken upstream source fails validation instead of poisoning the model.

**Stage 4 — Feature computation.**
- Batch features (Spark/dbt → offline store) for history; streaming features (Flink/Kafka Streams → online store) for freshness; point-in-time joins assemble training rows (detailed in [03 - Data & Feature Engineering.md](03 - Data & Feature Engineering.md) Q16–22).

**Stage 5 — Serving the prediction.**
- Online path: request → fetch features (online store, cache) → model → postprocess (thresholding, business rules, ranking blends) → response **and** log the (features, model version, prediction) tuple.
- That logged tuple becomes **tomorrow's training data** — closing the loop. Label ingestion joins actual outcomes back by request ID later.

**Failure thinking per stage:** collection can drop/duplicate (→ dedupe keys); ingestion can lag (→ freshness monitoring, backpressure); storage can corrupt (→ schema contracts + immutable raw); features can skew (→ shared code, Q13); serving can fail (→ fallbacks, [05 - Online Inference & Serving.md](05 - Online Inference & Serving.md) Q35).

---

## Q10. How would you design a training pipeline?

**Answer.** A training pipeline is an **orchestrated, versioned, repeatable DAG** — not a notebook. Requirements: reproducibility, data validation, gated promotion, and idempotent re-runs.

```
┌────────┐  ┌────────────┐  ┌───────────┐  ┌────────┐  ┌──────────┐  ┌──────────┐
│ Trigger│─►│ Data       │─►│ Feature   │─►│ Train  │─►│ Evaluate │─►│ Register │
│ sched/ │  │ extraction │  │ material- │  │ + tune │  │ + slice  │  │ + docs   │
│ event  │  │ + validate │  │ ization   │  │        │  │  report  │  │ (gated)  │
└────────┘  └────────────┘  └───────────┘  └────────┘  └──────────┘  └──────────┘
```

**Stage-by-stage design:**

1. **Trigger:** cron (daily/weekly), new-data event, or drift alert ([07 - Monitoring, Drift & Retraining.md](07 - Monitoring, Drift & Retraining.md) Q47). Parameterized: `run(dataset_snapshot, config_version)`.
2. **Data extraction & validation:** snapshot deterministic partitions (e.g. last 180 days); run schema + distribution checks; fail fast and loudly. Record the dataset hash/fingerprint — this is lineage.
3. **Feature materialization:** point-in-time join of labels + features from the feature store's offline store (Q13). Freeze the feature set version.
4. **Training & tuning:** train with fixed seed, logged hyperparameters, environment pinned (container digest). Distributed training (parameter servers / all-reduce, e.g. Horovod/Torch DDP) when data or model is large. Tuning via random/Bayesian search with early stopping — not exhaustive grids. Track every experiment (MLflow/W&B).
5. **Evaluation:** held-out test set (time-based split, never random for temporal data), **slice metrics** (per segment, region, device), calibration check, comparison table vs current prod model (Q26), plus bias/fairness checks where required. Output: a model report card, stored with the model.
6. **Registration (gated):** if eval passes thresholds → push artifact to registry with lineage (data hash, code commit, feature versions, metrics). Deployment to production is a *separate* CI/CD decision with human or automated approval ([08 - Model Deployment Strategies.md](08 - Model Deployment Strategies.md)).
7. **Idempotency & repeatability:** same inputs → same artifact (modulo GPU nondeterminism); re-running a failed step doesn't duplicate data (write to temp, atomic swap).

**Scaling choices:** small data → single machine, sklearn/XGBoost; large tabular → Spark ML / distributed XGBoost; deep learning → GPU cluster with DDP; huge sparse recsys models → parameter servers or embedding sharding.

**Interview sound bite:**
> "I treat a training pipeline like any production DAG: parameterized, validated inputs, fixed seeds, pinned environments, experiment tracking, evaluation gates, and a registry as the only path to deployment. Notebooks are for exploration, never for production training."

---

## Q11. How would you design an inference pipeline?

**Answer.** The inference (serving) path turns a trained model into a product decision, per request, under latency and availability constraints.

```
Client → LB → API service ─┬─► 1. validate/parse request
                           ├─► 2. retrieve features (online store, cache)
                           ├─► 3. preprocess (same transforms as training!)
                           ├─► 4. model predict (ensemble/optional)
                           ├─► 5. postprocess (threshold, rank, rules)
                           ├─► 6. log prediction + features + model version
                           └─► 7. respond (with timeout budget per stage)
```

**Design details per stage:**

1. **Request validation:** shape/type/range checks; reject early with 4xx rather than let garbage reach the model; request IDs for tracing and label joining.
2. **Feature retrieval:** online store lookup (Redis/DynamoDB) + in-process cache; fetch in parallel; missing features → impute with *the same defaults as training* and flag. Budget: ~10–20 ms.
3. **Preprocessing:** identical transformations as training — ideally **the same code object** (Q13): categorical encodings, normalization, tokenization.
4. **Prediction:** model loaded as a versioned artifact; concurrency via batching if throughput-bound; GPU/CPU pool; p99-aware batching timeout. For LLMs: streaming tokens, token budget, tool-call loop.
5. **Postprocessing & policy:** apply threshold (e.g. P(fraud) > 0.7 → block), business rules, fairness constraints, ranking blends (relevance × recency × revenue), diversity re-ranking. Keep policy **outside** the model so it can change without retraining.
6. **Prediction logging:** fire-and-forget (async producer) log of `{request_id, timestamp, user, features, model_version, prediction}` — the seed of future training data and drift monitoring. Never let logging failure block the response.
7. **Response:** include prediction + model version + enough metadata for downstream debugging; degrade gracefully (default ranking, cached scores, "safe" response) when any stage times out ([05 - Online Inference & Serving.md](05 - Online Inference & Serving.md) Q35).

**Cross-cutting:** per-stage timeouts summing to the p99 budget; tracing (OpenTelemetry) spanning client → features → model; canary-aware routing headers; autoscaling on QPS/latency ([06 - Scalability, Reliability & Availability.md](06 - Scalability, Reliability & Availability.md)).

**Batch inference variant:** same stages, minus latency pressure — orchestrate nightly scoring of millions of rows into a predictions table the product reads (cheaper, hours-fresh; see [05 - Online Inference & Serving.md](05 - Online Inference & Serving.md) Q30).

---

## Q12. What is the difference between training-time and serving-time infrastructure?

**Answer.**

| Dimension | Training infrastructure | Serving infrastructure |
|---|---|---|
| Goal | Throughput: process TBs, finish in hours | Latency & availability: ms responses, always up |
| Access pattern | Bulk sequential scans of large datasets | Millions of tiny random point lookups |
| Hardware | Big/accelerated (GPU/TPU clusters), spot instances OK | Right-sized (CPU for tree models, GPU for DL), always-on |
| Elasticity | Burst to N nodes, release after | Steady fleet + autoscaling on traffic |
| Storage | Data lake / warehouse, columnar scans | Online store (Redis/DynamoDB) + model in RAM |
| Data consistency | Point-in-time snapshots, batch correctness | Freshness + consistency per entity, low read latency |
| Failure impact | Job fails → re-run (annoying, not user-facing) | Failure → user-facing outage (needs redundancy, fallbacks) |
| Cost profile | Spike cost, optimize via spot/preemptible, batch windows | Continuous cost, optimize $/1k requests |
| Key metrics | Time-to-train, $/training run, eval quality | p50/p99 latency, QPS, availability, $/1k req |
| Typical stack | Spark, Kubeflow/Airflow, Ray, GPU nodes, MLflow | KServe/Triton/TorchServe/TF Serving, Redis, API services |

**Why they must stay connected despite differing:** the feature definitions, preprocessing code, and model artifacts flow from the training plane into the serving plane through the **feature store** and **model registry**. Those two systems are the *contract* between the planes; keeping the contract tight is what prevents training–serving skew (next question).

**Cost consequence:** because profiles differ, don't run training on serving clusters; do use spot GPUs for training, and reserve serving capacity for p99 traffic. Conversely, idle serving capacity at 3 a.m. is often repurposed for batch inference.

---

## Q13. How should training and inference features remain consistent?

**Answer.** Training–serving skew — features computed one way offline and another online — is one of the top real-world causes of "worked in the lab, failed in prod." Four defenses, in increasing strength:

**1. Feature store with shared transformation logic (the standard answer).**
Feature definitions live in **one place** (code + config), and are *materialized* to two stores:
- **Offline store** (warehouse/lake tables): historical point-in-time-correct values for training.
- **Online store** (Redis/DynamoDB): latest values, ms lookups for serving.
Because both are generated from the same definitions and code, parity is by construction. Tools: Feast, Tecton, SageMaker/Vertex/Databricks Feature Store.

**2. Point-in-time correctness for training rows.**
When building training data, join each label to feature values **as they existed at label time** — not the latest values. Naive joins leak future information ("data leakage") and inflate offline metrics that then collapse online. Implement via feature-store time-travel queries or careful as-of joins; log features *at serving time* so real traffic rows are automatically point-in-time-correct.

**3. Same code, different environments (for stateful transforms).**
Tokenizers, normalization stats, encoding maps, imputation defaults: fit on training data, **version the fitted artifacts**, and load the *same artifact* in serving. Never recompute "mean" from serving traffic.

**4. Parity tests and skew monitoring (the safety net).**
- Offline: golden-customer test — compute features for one entity through the batch path and the online path; diff.
- Online: log features at serving time and compare their live distribution to the training distribution ([07 - Monitoring, Drift & Retraining.md](07 - Monitoring, Drift & Retraining.md) Q42); alarm on divergence.

**Common skew causes to name:** different languages/tools offline (Spark, Python) vs online (Java, Go); newest-value joins in training; late-arriving data included offline but not online; time-zone handling; null/imputation defaults differing; feature recomputed after model snapshot (updated aggregates).

**Interview sound bite:**
> "I prevent skew by construction — one feature definition, materialized to offline and online stores, with point-in-time joins for training — and by detection — logged serving features compared against training distributions with alerting."

---

## Q14. Where should data, features, models, and predictions be stored?

**Answer.** Match each artifact to its access pattern:

| Artifact | Access pattern | Right storage | Wrong-looking-but-common mistakes |
|---|---|---|---|
| **Raw data** | Write-once, bulk scans, replay/audit | Object lake (S3/GCS) in open formats (Parquet, Delta/Iceberg) partitioned by date | Landing raw in a warehouse at row-store prices; mutable raw (breaks lineage) |
| **Cleaned/curated data** | SQL analytics, joins, DBT models | Warehouse (BigQuery/Snowflake/Redshift) or lakehouse tables | Analytics in the operational DB |
| **Labels** | Append + joins by entity & time | Warehouse tables keyed by (entity, event_time) | Overwriting labels, losing history |
| **Features — offline** | Point-in-time scans for training (TB) | Offline store: warehouse/lakehouse tables with event-timestamp columns | Recomputing history per training run |
| **Features — online** | Key-value lookups <10 ms (millions/sec) | Online store: Redis/DynamoDB/HBase; local cache in front | Serving features out of the warehouse (too slow, too costly) |
| **Models** | Immutable artifact + metadata + lineage | Model registry (MLflow/SageMaker/Vertex) + artifact store (S3); container images for serving | Models on a shared NFS with manual copies; no version history |
| **Predictions (online)** | Append-only, high write volume; joined later with labels for training & drift | Log pipeline → lake (Parquet) + optionally key-value for product reads (e.g. cached recs) | Logging only the prediction, not features & model version |
| **Predictions (batch)** | Bulk reads by product services | Predictions table in warehouse/DB or cache pre-warmed | Regenerating on the fly what could be precomputed |
| **Metadata (runs, metrics, lineage)** | Audit, comparison, governance | Experiment tracker (MLflow) + data catalog | Tribal knowledge, stale wikis |

**Retention & lifecycle rules worth stating:**
- Raw: immutable, long retention (replayability), tiered storage for cost.
- Prediction logs: months of history for drift/retraining; compress, partition by date.
- Online store: TTL per feature group (freshness semantics).
- Models: keep N versions + the champion; artifacts immutable.

**Cost angle:** warehouse scans are the most expensive per byte — keep training-scale data in the lake and only aggregates/features in the warehouse ([09 - Cost, Security & Practical Trade-offs.md](09 - Cost, Security & Practical Trade-offs.md) Q54).

---

## Q15. How would you design an ML system that supports continuous retraining?

**Answer.** Continuous retraining = an automated, gated loop. The system retrains on fresh data, evaluates against the incumbent, and promotes only if better — with humans able to veto.

```
 ┌────────────────────────────────────────────────────────────────────┐
 │                    CONTINUOUS RETRAINING LOOP                      │
 │                                                                    │
 │  Fresh data ──► Automated training pipeline (Q10)                  │
 │                    │                                              │
 │                    ▼                                              │
 │                Evaluation gates:                                   │
 │                 • beats champion offline (primary + slices)        │
 │                 • guardrails pass (latency, size, fairness)        │
 │                    │                                              │
 │                    ▼                                              │
 │                Register as candidate ──► Shadow/canary deploy      │
 │                    │                            │                  │
 │                    ▼                            ▼                  │
 │                Monitoring: online metrics, drift, cost            │
 │                    │                                              │
 │                    ├─ healthy ──► promote to champion              │
 │                    └─ regressed ─► auto-rollback → open incident   │
 └────────────────────────────────────────────────────────────────────┘
```

**Design decisions to spell out:**

1. **Trigger policy (what starts a run):**
   - *Scheduled:* daily/weekly — the default; predictable cost and cadence.
   - *Data-triggered:* N new labeled rows or a new day partition arrived.
   - *Performance-triggered:* drift alarm or business-metric drop ([07 - Monitoring, Drift & Retraining.md](07 - Monitoring, Drift & Retraining.md) Q47).
   Combine: schedule + triggers, with a cooldown to prevent thrash.

2. **What is retrained:** full retrain (simple, expensive, most robust) vs warm-start from champion weights (cheap, risk of compounding bias) vs incremental/online learning (rare; needs careful validation and rollback). Also decide: retrain *model only*, or also *re-learn preprocessing stats* (usually yes — otherwise skew).

3. **Evaluation gates before promotion:** must beat champion on primary metric by a margin (avoid noise-promotions), pass slice/feasibility checks, model-size/latency budgets, calibration. Champion must be pinned by version, not "whatever's in prod."

4. **Safe rollout:** candidate → shadow (score silently) → canary (small % traffic with auto-abort) → full ([08 - Model Deployment Strategies.md](08 - Model Deployment Strategies.md)). Auto-rollback criteria defined *before* launch (error rate, latency, business guardrail).

5. **State & idempotency:** each run gets a versioned dataset snapshot + config hash; runs are idempotent; the registry is the single source of truth of "what's deployed where."

6. **Cost control:** retraining budget per week; skip runs when data delta is small; use spot instances; prefer warm-starts when drift is mild.

7. **Failure handling:** a failed run alerts but never silently leaves "no model"; the previous champion keeps serving. Retraining failures are monitored like any production job.

**Anti-patterns to call out:** retraining on live-served predictions with no exploration/unbiased data (feedback-loop bias); auto-promotion without gates; retraining on data the model itself influenced without randomization; no rollback path.

**Interview sound bite:**
> "Continuous retraining is CI/CD for models: trigger → validated data → train → gate vs the pinned champion → progressive rollout with auto-rollback. Automation does the labor; gates do the judgment."

---

## Chapter 2 — Interview checklist

- [ ] I can draw the full end-to-end diagram from memory in <2 minutes and narrate the two planes + control loop.
- [ ] I can describe the 5 data-flow stages and one failure mode at each.
- [ ] I can spec a training DAG with validation, seeds, tracking, eval gates, and registration.
- [ ] I can spec an inference path with per-stage latency budgets, logging, and graceful degradation.
- [ ] I know the 4 defenses against training–serving skew and can name common skew causes.
- [ ] I can pick the right storage for each artifact by access pattern.
- [ ] I can design the retraining loop with triggers, gates, progressive rollout, and rollback.

**Next → [03.md — Data & Feature Engineering](03.md):** going deeper on the left side of the diagram — pipelines, quality, feature stores, and schemas that change under you.
