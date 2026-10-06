# 10. Summary — Cheat Sheet, Glossary & Master Checklist

> One page of patterns, one glossary, one 68-question index. This is your 24-hours-before-the-interview file.
> Full detail lives in the chapter files — every table below links back.

---

## A. The reusable interview skeleton (memorize this, not scripts)

```
0. Requirements   functional + numbers (QPS, p99, users) + constraints        [01.md Q7]
1. ML framing     task · unit of prediction · target/label window · metrics   [01.md Q4–5]
2. Data           sources → pipeline → features → freshness → training rows   [03.md]
3. Modeling       baseline ladder → model choice → training DAG → evaluation  [04.md]
4. Serving        batch/online/stream · Client→LB→API→Features→Model→Response [05.md]
5. Scale & HA     stateless scaling · degradation ladder · failure stories    [06.md]
6. Monitor & loop 4 monitoring layers · drift · triggers · safe rollout       [07.md] [08.md]
7. Trade-offs     cost · security · simpler-alternative · build order         [09.md]
```

State assumptions out loud. End with a phased build order ("v1: batch + rules → v2: online model → v3: real-time features").

---

## B. The two diagrams to draw from memory

**End-to-end ML system** ([End-to-End ML Architecture.md](End-to-End ML Architecture.md) Q8):

```
Data Sources → Ingestion → Storage → Validation → Feature Store ─→ Training → Evaluation → Registry → Deployment → Inference → Monitoring → Retraining
   (batch + streaming planes; feature store & registry are the contracts; the loop is the system)
```

**Online serving path** ([Online Inference & Serving.md](Online Inference & Serving.md)):

```
Client → Load Balancer → API Service → Feature Retrieval → Model Server → Prediction → Response
                                          (online store, cache)              │
                                             └── async prediction logging ───┴──→ monitoring/retraining
```

---

## C. Decision tables (the exam's favorite questions)

**Serving pattern** ([Online Inference & Serving.md](Online Inference & Serving.md) Q30): stale-is-fine → **batch** · interaction-time decision → **online** · continuous events, no request → **streaming** · no network → **on-device** · commodity LLM → **hosted API**. Most real systems: batch base + real-time delta.

**Metric by task** ([Model Training & Evaluation.md](Model Training & Evaluation.md) Q24): imbalanced → **PR-AUC, precision@capacity** · balanced binary → ROC-AUC/F1 · ranking → **NDCG@k** · retrieval → **recall@k** · regression → MAE/RMSE/quantile · generation → groundedness, win-rate. Always + calibration + slices + guardrails.

**Imbalance playbook** ([Model Training & Evaluation.md](Model Training & Evaluation.md) Q25): metric → threshold → class weights/focal loss → resample (train only) → hard negatives/features.

**Drift** ([Monitoring, Drift & Retraining.md](Monitoring, Drift & Retraining.md) Q43–44): **data drift = P(x)** (PSI/KS, no labels needed) · **concept drift = P(y|x)** (labels + proxies) · always triage pipeline-bug first.

**Deployment** ([Model Deployment Strategies.md](Model Deployment Strategies.md)): **shadow** = zero risk, no outcomes · **canary** = bounded risk + real signal (default) · **A/B** = causal decision · **blue-green** = atomic switch + instant rollback · compose as shadow → canary(A/B) → blue-green → warm standby.

**Storage by access pattern** ([End-to-End ML Architecture.md](End-to-End ML Architecture.md) Q14): bulk scans → lake · SQL analytics → warehouse · ms lookups → Redis/DynamoDB (online store) · immutable artifacts + lineage → registry · append-heavy prediction logs → lake/Parquet.

**Cost levers in ROI order** ([Cost, Security & Practical Trade-offs.md](Cost, Security & Practical Trade-offs.md) Q54): demand (batch-precompute, caches, cascades) → model (quantize/distill/compile) → hardware (spot, utilization, tiering) → pipeline (incremental retrains, streaming only where it pays).

**Latency levers** ([Online Inference & Serving.md](Online Inference & Serving.md) Q33/[Scalability, Reliability & Availability.md](Scalability, Reliability & Availability.md) Q40): parallel feature fetch → warm pools → compile/quantize/distill → deadline batching → 3-level caching → shed-before-stall. Fix p99, not p50.

---

## D. The sentences that signal seniority

- "The model is 10–20% of the system; I design the other 80%." ([System Design Fundamentals.md](System Design Fundamentals.md) Q1)
- "Offline metrics gate; online metrics decide." ([System Design Fundamentals.md](System Design Fundamentals.md) Q6)
- "One feature definition, two materializations — parity by construction." ([End-to-End ML Architecture.md](End-to-End ML Architecture.md) Q13 / [Data & Feature Engineering.md](Data & Feature Engineering.md) Q18)
- "Triggers start runs; gates make decisions." ([Monitoring, Drift & Retraining.md](Monitoring, Drift & Retraining.md) Q47)
- "A fast cached answer beats a slow timeout." ([Scalability, Reliability & Availability.md](Scalability, Reliability & Availability.md) Q38)
- "Shadow shows what the model would say; A/B shows what users would do." ([Model Deployment Strategies.md](Model Deployment Strategies.md) Q52)
- "I ship the simplest model that clears the business bar." ([Cost, Security & Practical Trade-offs.md](Cost, Security & Practical Trade-offs.md) Q58)
- "Rollback is a versioned stack: model + feature schema + thresholds." ([Model Deployment Strategies.md](Model Deployment Strategies.md) Q53)
- "Features and predictions are logged per request — that log is tomorrow's training data." ([End-to-End ML Architecture.md](End-to-End ML Architecture.md) Q9)
- "Fallback rate and feature freshness are SLOs too." ([Scalability, Reliability & Availability.md](Scalability, Reliability & Availability.md) Q36)

---

## E. Glossary (rapid-fire)

| Term | One-line definition | Deep dive |
|---|---|---|
| Training–serving skew | Features computed differently offline vs online; the classic silent killer | [End-to-End ML Architecture.md](End-to-End ML Architecture.md) Q13, [Data & Feature Engineering.md](Data & Feature Engineering.md) Q18 |
| Point-in-time correctness | Training rows use feature values as they existed at prediction time; prevents leakage | [End-to-End ML Architecture.md](End-to-End ML Architecture.md) Q13 |
| Feature store | System defining features once and materializing them to offline (training) + online (serving) stores | [Data & Feature Engineering.md](Data & Feature Engineering.md) Q20 |
| Data drift (covariate shift) | Input distribution P(x) changes over time | [Monitoring, Drift & Retraining.md](Monitoring, Drift & Retraining.md) Q43 |
| Concept drift | Input–output relationship P(y\|x) changes | [Monitoring, Drift & Retraining.md](Monitoring, Drift & Retraining.md) Q44 |
| Label drift | Output distribution P(y) changes | [Monitoring, Drift & Retraining.md](Monitoring, Drift & Retraining.md) Q43 table |
| PSI | Population stability index; standard drift score (alert > ~0.2) | [Monitoring, Drift & Retraining.md](Monitoring, Drift & Retraining.md) Q43 |
| MCAR/MAR/MNAR | Missingness mechanisms; MNAR means imputation is dangerous | [Data & Feature Engineering.md](Data & Feature Engineering.md) Q17 |
| PR-AUC | Precision–recall area; the honest metric under extreme imbalance | [Model Training & Evaluation.md](Model Training & Evaluation.md) Q24 |
| NDCG@k | Position-weighted ranking quality metric | [Model Training & Evaluation.md](Model Training & Evaluation.md) Q24 |
| Calibration | Predicted probabilities matching observed frequencies (ECE/Brier) | [Model Training & Evaluation.md](Model Training & Evaluation.md) Q24 |
| SMOTE / focal loss | Minority oversampling / loss focusing hard examples | [Model Training & Evaluation.md](Model Training & Evaluation.md) Q25 |
| Champion–challenger | Incumbent model vs candidates through gates | [Model Training & Evaluation.md](Model Training & Evaluation.md) Q26 |
| Walk-forward evaluation | Rolling-origin temporal validation; measures decay rate | [Model Training & Evaluation.md](Model Training & Evaluation.md) Q27 |
| Batch / online / streaming inference | Precomputed / per-request / per-event prediction patterns | [Online Inference & Serving.md](Online Inference & Serving.md) Q29–30 |
| Dynamic batching | Accumulating requests briefly to batch on GPU; cap queue time | [Online Inference & Serving.md](Online Inference & Serving.md) Q31 |
| Cascade | Cheap model first, escalate uncertain/high-value cases | [Cost, Security & Practical Trade-offs.md](Cost, Security & Practical Trade-offs.md) Q57 |
| Load-shedding / admission control | Dropping/degrading low-priority traffic under overload | [Scalability, Reliability & Availability.md](Scalability, Reliability & Availability.md) Q38 |
| Degradation ladder | Fallback chain: model → cache → small model → rules → hide feature | [Online Inference & Serving.md](Online Inference & Serving.md) Q35 |
| Bulkhead / circuit breaker | Fault isolation between dependencies / auto-trip to fallback | [Scalability, Reliability & Availability.md](Scalability, Reliability & Availability.md) Q39 |
| Blue-green | Two full environments, atomic LB switch, instant rollback | [Model Deployment Strategies.md](Model Deployment Strategies.md) Q49 |
| Canary | Small % traffic, stepwise expansion, auto-abort | [Model Deployment Strategies.md](Model Deployment Strategies.md) Q50 |
| Shadow deployment | Mirror traffic, log predictions, zero user impact | [Model Deployment Strategies.md](Model Deployment Strategies.md) Q51 |
| Interleaving | Mixing two rankers' results in one list; sensitive ranking comparison | [Model Deployment Strategies.md](Model Deployment Strategies.md) Q52 |
| SRM | Sample-ratio mismatch; broken randomization check before reading A/B results | [Model Deployment Strategies.md](Model Deployment Strategies.md) Q52 |
| Model registry | Versioned store of artifacts + lineage + stage; the only path to prod | [End-to-End ML Architecture.md](End-to-End ML Architecture.md) Q10 |
| Matured cohort | Prediction group whose label window has fully elapsed; avoids timing bias | [Monitoring, Drift & Retraining.md](Monitoring, Drift & Retraining.md) Q46 |
| Two-tower model | Dual encoders (user/item) for retrieval via embedding similarity | [System Design Cases.md](System Design Cases.md) Q59 |
| ANN index | Approximate nearest-neighbor index (HNSW/IVF) for embedding retrieval | [System Design Cases.md](System Design Cases.md) Q67 |
| RAG | Retrieval-augmented generation; grounding LLM answers in retrieved docs | [System Design Cases.md](System Design Cases.md) Q67 |
| Prompt injection | Untrusted content carrying instructions; treat retrieved text as data | [Cost, Security & Practical Trade-offs.md](Cost, Security & Practical Trade-offs.md) Q55, [System Design Cases.md](System Design Cases.md) Q68 |
| DP-SGD | Differentially-private training; limits memorization of individuals | [Cost, Security & Practical Trade-offs.md](Cost, Security & Practical Trade-offs.md) Q56 |
| Goodhart's law | When a measure becomes a target, it stops being a good measure | [System Design Fundamentals.md](System Design Fundamentals.md) Q5 |
| Two-phase migration | Add-new → migrate → deprecate-old; how renames survive | [Data & Feature Engineering.md](Data & Feature Engineering.md) Q22 |

---

## F. Master checklist — all 68 questions

Tick each only when you can answer it out loud, in 2–3 minutes, with a diagram or decision table. Chapter links go to the full answers.

**1. Fundamentals — [System Design Fundamentals.md](System Design Fundamentals.md)**
- [ ] 1. What is ML system design?
- [ ] 2. How is it different from traditional system design?
- [ ] 3. Major components of an end-to-end ML system?
- [ ] 4. Translate a business problem into an ML problem?
- [ ] 5. Define objective and success metrics?
- [ ] 6. Offline ML metrics vs online business metrics?
- [ ] 7. Requirements to clarify before designing?

**2. End-to-End Architecture — [End-to-End ML Architecture.md](End-to-End ML Architecture.md)**
- [ ] 8. What does a typical end-to-end ML system look like?
- [ ] 9. Data flow from collection to prediction?
- [ ] 10. Design a training pipeline?
- [ ] 11. Design an inference pipeline?
- [ ] 12. Training-time vs serving-time infrastructure?
- [ ] 13. Keeping training and inference features consistent?
- [ ] 14. Where should data, features, models, predictions be stored?
- [ ] 15. Design for continuous retraining?

**3. Data & Feature Engineering — [Data & Feature Engineering.md](Data & Feature Engineering.md)**
- [ ] 16. Design the data pipeline?
- [ ] 17. Handle missing, noisy, incorrect data at scale?
- [ ] 18. Prevent training–serving skew?
- [ ] 19. Feature engineering in production?
- [ ] 20. What is a feature store and why useful?
- [ ] 21. Real-time vs batch features?
- [ ] 22. Dealing with changing schemas?

**4. Training & Evaluation — [Model Training & Evaluation.md](Model Training & Evaluation.md)**
- [ ] 23. Choose a model for production?
- [ ] 24. Select the evaluation metric?
- [ ] 25. Handle highly imbalanced data?
- [ ] 26. Compare new model vs production model?
- [ ] 27. Offline evaluation before deployment?
- [ ] 28. Offline metrics improve but business metrics worsen?

**5. Online Inference & Serving — [Online Inference & Serving.md](Online Inference & Serving.md)**
- [ ] 29. Ways to serve a model?
- [ ] 30. Batch vs real-time inference?
- [ ] 31. Design a low-latency prediction API?
- [ ] 32. Scale an inference service?
- [ ] 33. Factors affecting inference latency?
- [ ] 34. Handle millions of prediction requests?
- [ ] 35. Handle model-server failure?

**6. Scalability, Reliability & Availability — [Scalability, Reliability & Availability.md](Scalability, Reliability & Availability.md)**
- [ ] 36. Highly available ML system?
- [ ] 37. Horizontally scale inference?
- [ ] 38. Handle traffic spikes?
- [ ] 39. Design fault tolerance?
- [ ] 40. Reduce inference latency?
- [ ] 41. Balance latency, accuracy, cost?

**7. Monitoring, Drift & Retraining — [Monitoring, Drift & Retraining.md](Monitoring, Drift & Retraining.md)**
- [ ] 42. What to monitor in production ML?
- [ ] 43. What is data drift?
- [ ] 44. What is concept drift?
- [ ] 45. Detect performance degradation?
- [ ] 46. Monitor with delayed labels?
- [ ] 47. What should trigger retraining?
- [ ] 48. Safely deploy a retrained model?

**8. Deployment Strategies — [Model Deployment Strategies.md](Model Deployment Strategies.md)**
- [ ] 49. Blue-green deployment?
- [ ] 50. Canary deployment?
- [ ] 51. Shadow deployment?
- [ ] 52. A/B testing for models?
- [ ] 53. Roll back a poor performer?

**9. Cost, Security & Trade-offs — [Cost, Security & Practical Trade-offs.md](Cost, Security & Practical Trade-offs.md)**
- [ ] 54. Reduce cost of large-scale ML?
- [ ] 55. Secure an inference API?
- [ ] 56. Protect sensitive training data?
- [ ] 57. Trade off accuracy, latency, scalability, cost?
- [ ] 58. When to choose a simpler model?

**10. System Design Cases — [System Design Cases.md](System Design Cases.md)**
- [ ] 59. E-commerce recommendation system
- [ ] 60. Movie/video recommendation system
- [ ] 61. Fraud detection system
- [ ] 62. Spam/email classification system
- [ ] 63. Search ranking system
- [ ] 64. Real-time price prediction system
- [ ] 65. Customer churn prediction system
- [ ] 66. Image classification at scale
- [ ] 67. Large-scale RAG system
- [ ] 68. LLM-powered AI assistant with tools

---

## G. The 24-hour checklist

1. Re-read section A + B; redraw both diagrams from memory twice.
2. Skim section C tables; recite each decision rule aloud.
3. Pick your 3 weakest cases from section F and re-run them aloud against the skeleton ([System Design Cases.md](System Design Cases.md)).
4. Read section D out loud once — these are your transition sentences.
5. Sleep. Sleep beats one more cram chapter; recall decays, and you measured that ([Monitoring, Drift & Retraining.md](Monitoring, Drift & Retraining.md) Q45).

Good luck. 🚀
