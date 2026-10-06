# 4. Model Training & Evaluation

> **Section 4 of 10** · Questions 23–28 · Estimated study time: 3–4 hours
> The chapter where you prove the model is *worth* deploying — and know what to do when offline and online disagree.

---

## In this chapter

| # | Question | Key takeaway |
|---|----------|--------------|
| 23 | Choosing a model for production | Start simple, match model to constraints, ensembles later |
| 24 | Selecting the evaluation metric | Metric follows task + class balance + business asymmetry |
| 25 | Highly imbalanced data | Resample/focal loss/thresholds — and PR-AUC, not accuracy |
| 26 | Comparing new model vs production model | Same data, same eval, champion–challenger, then online |
| 27 | Offline evaluation before deployment | Time-based splits, slices, calibration, shadow replay |
| 28 | Offline metrics improve but business metrics worsen | Diagnose proxy mismatch, shift, loops, systems effects |

---

## Q23. How do you choose a model for a production ML system?

**Answer.** Model choice is an **optimization under constraints** (latency, cost, explainability, team skills, deadline), not a leaderboard sport. A disciplined sequence:

**Step 1 — Start with a strong simple baseline.**
- Logistic regression / linear models with good features.
- Gradient-boosted trees (XGBoost/LightGBM) — the default winner on tabular data.
- For text/images: fine-tune a small pretrained transformer/CNN before dreaming of large models.
- Heuristic/majority baseline to measure against.

Why: simple baselines quantify how much signal sophisticated modeling adds, train fast, are easy to debug, cheap to serve, and often reach 90–95% of achievable quality. The gap between baseline and complex model is your "complexity budget" to spend consciously ([09 - Cost, Security & Practical Trade-offs.md](09 - Cost, Security & Practical Trade-offs.md) Q58).

**Step 2 — Match model family to data & task:**

| Data / task | Default choice | Why |
|---|---|---|
| Tabular, heterogeneous features | GBDT (XGBoost/LightGBM/CatBoost) | Handles mixed types, non-linearities, missing values, little tuning |
| Tabular, huge scale + embeddings (recsys/ads) | GBDT on engineered features, or DL with embeddings (DLRM/Two-tower) | Scale + sparse interactions |
| Text classification | Fine-tuned transformer (DistilBERT-class) → larger if needed | Pretrained language knowledge |
| Text generation / RAG | LLM (hosted or self-hosted) + grounding | Reasoning + language; retrieval adds facts |
| Images | CNN (efficient variants) or ViT fine-tune | Inductive bias or scale |
| Sequence/time-series | Gradient boosting on lag features; specialized forecasters (Prophet-class for scale, DL for many series) | Tabular tricks are shockingly competitive |
| Similarity/retrieval | Two-tower embeddings + ANN index | ms retrieval over millions of items |
| Cold start / almost no labels | Rules + semi-supervised + active learning | Bootstrapping the flywheel |

**Step 3 — Filter by hard constraints (the kill criteria):**
- **Latency:** p99 budget rules out big transformers; 10 ms → GBDT/small net/distilled models.
- **Explainability:** credit/medical → monotonic GBDT, scorecards, SHAP on tree models.
- **Cost per 1k predictions** and training budget.
- **On-call maturity:** can the team operate GPUs and custom serving, or only sklearn-on-EC2?
- **Fairness/auditability requirements** (structured models with monitored slices).

**Step 4 — Compare candidates fairly:** identical data splits, same features, fixed seeds, multiple runs (report mean ± std), tune each candidate with a comparable budget (no strawman GBDT vs hero-tuned NN). Evaluate on the metric from Q24 and on slices.

**Step 5 — Decide with a portfolio mindset:** ship the simple model first (gets the loop running, establishes MLOps), then iterate complexity only where the offline+online lift justifies it. Many production systems are **ensembles**: GBDT + simple NN, or cascade (cheap model handles 95% of traffic, big model the rest) ([09 - Cost, Security & Practical Trade-offs.md](09 - Cost, Security & Practical Trade-offs.md) Q57).

**Interview sound bite:**
> "I start with logistic regression and GBDT baselines, then let hard constraints — p99 latency, cost, explainability, team operability — eliminate candidates before accuracy ever comes up. I ship the simplest model that clears the business bar, and add complexity only when measured lift justifies its operational cost."

---

## Q24. How do you select the appropriate evaluation metric?

**Answer.** Metric selection = **task type × class balance × error asymmetry × decision mechanism.** A wrong metric optimizes the wrong thing convincingly.

**Step 1 — Follow the decision the prediction drives.** Ask: what action follows from the output, and what does each error cost?

| Decision | Costly error | Metric family |
|---|---|---|
| Fraud: block transaction | False positive (angry customer) but FN is lost money → asymmetric | Precision@k, recall@fixed-precision, Fβ (β>1 if recall matters more) |
| Spam: move to folder | FP hides an important email | Precision-first, F0.5, or ranking metrics |
| Medical screening | FN misses disease | Recall-first (sensitivity), then specificity |
| Ranking feeds/search | Bad top slots | NDCG@k, MRR, MAP, precision@k |
| Regression pricing | Big misses | RMSE (big misses hurt) or MAE (robust) |
| Forecasting inventory | Stockouts vs overstock | Quantile/pinball loss, WAPE by segment |
| Retrieval for RAG | Missing the right doc | Recall@k |
| LLM assistant | Wrong facts, refusals | Groundedness, win-rate vs baseline, task success rate |

**Step 2 — Respect class balance:**

| Situation | Avoid | Prefer |
|---|---|---|
| Imbalanced (fraud 0.1%) | Accuracy, ROC-AUC (can look great while useless) | PR-AUC, F1/Fβ, recall@precision, lift/gain charts |
| Balanced | — | Accuracy, ROC-AUC fine |
| Multi-class | Per-class accuracy averaged blindly | Macro-F1 (equal class weight) vs weighted-F1 (equal example weight); confusion matrix always |
| Ranking | Pointwise accuracy | Listwise: NDCG@k (position-weighted) |

Why ROC-AUC misleads under imbalance: with 99.9% negatives, FPR of 1% still means ~10 false alarms per ~1 true positive for rare classes; PR curves expose this directly.

**Step 3 — Thresholds are part of the metric.** A classifier's real output is a *decision* at a threshold: report the full curve (PR/ROC), then choose the operating point from business costs — e.g. "maximize recall subject to precision ≥ 90%". Re-tune the threshold when costs or class priors shift; it's free accuracy compared to retraining.

**Step 4 — Calibration & uncertainty.** If outputs feed thresholds, pricing, or humans ("72% risk"), check **calibration** (reliability diagrams, ECE, Brier score); calibrate with Platt/isotonic on a held-out set.

**Step 5 — Always add slices.** A single aggregate number hides winners and losers. Evaluate per segment: geo, device, new-vs-power users, class, time period. Guardrail slices from Q5 apply here too.

**Step 6 — Secondary metrics for robustness:** performance vs label lag, degradation curves over time (how fast does quality decay → informs retraining cadence, [07 - Monitoring, Drift & Retraining.md](07 - Monitoring, Drift & Retraining.md)), inference cost and latency per candidate.

**Interview sound bite:**
> "The metric must mirror the decision: I derive it from the action the prediction triggers, the cost asymmetry of the errors, and the class balance — PR-AUC and threshold tuning for rare-event problems, NDCG@k for ranking, calibrated probabilities wherever a threshold acts on them — and I never report an aggregate without per-slice breakdowns."

---

## Q25. How would you handle highly imbalanced data?

**Answer.** Imbalance (fraud 1:1000, ad click 1:100, disease 1:10,000) breaks naive training: the model minimizes loss by predicting the majority class, and accuracy becomes meaningless. Attack on four fronts, in order of typical impact:

**1. Fix the metric and the operating point first (no data changes needed).**
- PR-AUC as headline; report recall@precision=90% or precision@k matching operational capacity (fraud team can review 1,000 cases/day → optimize precision@1000/day).
- Tune the **decision threshold** on a validation set to the business asymmetry; consider **calibrated probabilities** then thresholding.
- If using ROC-AUC, also quote PR-AUC; never accuracy.

**2. Resampling — on the training set only, never on validation/test.**
- **Over-sampling the minority:** random duplication (risk: overfitting via memorization) or **SMOTE** (interpolate synthetic minority points; works for continuous features, questionable for sparse/categorical/high-dim; variants for categorical: SMOTE-NC).
- **Under-sampling the majority:** random (wastes data) or informed (Tomek links, NearMiss); often combined: under-sample most + mild over-sample.
- Scale reality check: with 1B rows, under-sampling is usually the cheap win; with 10k rows, over-sample/SMOTE.

**3. Cost-sensitive learning — usually the cleanest at scale.**
- **Class weights** (inverse-frequency or set from the confusion matrix costs): supported by logistic regression, SVMs, XGBoost (`scale_pos_weight`), focal loss for NNs.
- **Focal loss** (down-weights easy examples, focuses gradient on hard ones) — standard for dense detectors, useful for extreme imbalance.
- **Threshold moving** post-training (free; tune per quarter).
- **Anomaly-detection framing** when positives are extremely rare and poorly sampled: Isolation Forest / autoencoder reconstruction error — but if you have good labeled fraud, supervised usually wins.

**4. Data & label strategy (the highest-leverage, longest-term):**
- **Hard negative mining:** collect the negatives that fooled the current model (e.g. transactions that were flagged but legitimate, or vice versa) and add them to training.
- **Active learning:** route ambiguous cases to human labelers.
- **Better features beat rebalancing:** velocity, device, and graph features separate classes that raw amounts cannot.
- **Ensemble/stacking** and per-slice models where the minority lives in a specific segment.
- **Beware evaluation leakage from resampling:** SMOTE and duplicates across folds inflate metrics — split first, resample within train folds only.

**Practical defaults I'd state in an interview:** GBDT with `scale_pos_weight`, PR-AUC + threshold tuning, precision@capacity as the operational metric, hard-negative mining in the next data cycle, and monitoring recall on live confirmed labels ([07 - Monitoring, Drift & Retraining.md](07 - Monitoring, Drift & Retraining.md)).

**What NOT to do:** oversample before splitting; report accuracy; tune threshold on the test set; SMOTE categorical IDs; assume resampling fixes a features problem.

**Interview sound bite:**
> "For extreme imbalance I change the metric before the data: PR-AUC, recall at a fixed precision that matches operational capacity, and a threshold tuned to error costs. Then class weights or focal loss in training, modest resampling within train folds only, and long-term, hard-negative mining and features that actually separate the classes."

---

## Q26. How do you compare a new model against the existing production model?

**Answer.** The champion–challenger process — offline gate, shadow, then online. Never compare a new model against a *paper* number.

**Stage 1 — Freeze the comparison frame.**
- Same test windows, same feature snapshot, same evaluation code.
- The champion is a **pinned registry version** with its original eval config, not "whatever's deployed".
- Multiple time windows: last 7/30/90 days; check consistency (a challenger that wins only on last week is suspect).

**Stage 2 — Offline comparison (cheap gate):**
1. **Headline metric** with confidence intervals (bootstrap over time blocks) — is the lift outside noise? Small lifts (±0.2%) demand tighter eval sets or longer windows.
2. **Slice table:** per segment/geo/device/class — no slice regresses beyond tolerance (guardrail slices from Q5).
3. **Calibration comparison** if thresholds act on probabilities.
4. **Head-to-head disagreement analysis:** on how many rows do they differ? Where the challenger wins, are the wins the *cases you care about*? (e.g. challenger catches 20% more fraud but on low amounts).
5. **Operational profile:** latency p50/p99, memory, cost per 1k predictions — a model that's better but 5× slower may lose ([09 - Cost, Security & Practical Trade-offs.md](09 - Cost, Security & Practical Trade-offs.md) Q57).
6. **Failure taxonomy:** sample 50–100 disagreements and hand-label them; produces a qualitative story ("challenger wins on new merchants, loses on recurring subscriptions").

**Stage 3 — Shadow evaluation (no user impact):**
- Deploy challenger alongside champion; mirror live traffic; compare predictions on real production distribution without exposing outputs ([08 - Model Deployment Strategies.md](08 - Model Deployment Strategies.md) Q51).
- Catches serving-time issues offline eval can't: feature skew, latency under load, logging gaps.

**Stage 4 — Online comparison (causal gate):**
- **A/B test** ([08 - Model Deployment Strategies.md](08 - Model Deployment Strategies.md) Q52): randomize traffic, primary business metric + guardrails, pre-computed sample size, run full weekly cycles, analyze with proper tests (and watch for novelty effects in week 1).
- Interleaving (for ranking problems) can detect smaller differences with less traffic.
- Ship only if primary metric lifts significantly and guardrails hold; otherwise keep champion and document what was learned.

**Decision matrix I'd narrate:**

| Offline | Shadow | Online | Decision |
|---|---|---|---|
| ✅ better | ✅ same behavior | ✅ lift | Ship progressively ([08 - Model Deployment Strategies.md](08 - Model Deployment Strategies.md)) |
| ✅ better | ⚠️ latency/skew issues | — | Fix serving first |
| ✅ better | ✅ | ❌ no lift / guardrail drop | Don't ship; investigate offline–online gap (Q28) |
| ❌ worse | — | — | Archive; record learnings in registry |

**Interview sound bite:**
> "Champion–challenger: pinned champion, identical eval windows with confidence intervals, per-slice guardrails, disagreement analysis, then shadow on live traffic to catch serving issues, then an A/B test as the causal gate. Offline numbers select candidates; online numbers select models."

---

## Q27. How would you perform offline evaluation before deployment?

**Answer.** A complete offline evaluation protocol has seven components:

**1. Correct data splits — the most common silent error.**
- **Temporal split** for time-dependent data (train on months 1–6, validate 7, test 8+): mimics production, where you predict the *future*.
- **Random splits** only for i.i.d.-ish data; never for temporal, or you leak the future (user's *later* behavior predicting their *earlier* label).
- **Group-aware splits** when the same entity must not straddle sets (same user in train and test → leakage for personalization problems).
- For temporal data, use **walk-forward / rolling-origin evaluation**: evaluate on several successive windows, not one — this also measures how fast quality decays (retraining cadence signal).

**2. Point-in-time-correct features** ([03 - Data & Feature Engineering.md](03 - Data & Feature Engineering.md) Q13/Q18) — training rows must contain only information available at prediction time. Leak-free or the rest doesn't matter.

**3. Metric suite, not a single number:** headline metric (Q24) + calibration + slices + secondary robustness metrics + business-weighted score if costs are known.

**4. Baseline ladder:** heuristic rules → simple model → current champion. A candidate is only interesting if it beats the *champion*, not the heuristic.

**5. Statistical rigor:** confidence intervals via bootstrap (blocked by time), significance vs champion, minimum-detectable-effect thinking: "our eval set can only detect lifts >1%; below that, go straight to shadow/A/B."

**6. Robustness & stress tests:**
- **Degraded-input tests:** missing fields, stale features, anomalous values — model should degrade gracefully (this is serving reality).
- **Slice stress:** performance on rare segments; new-category handling (UNKNOWN treatment).
- **Adversarial/edge probes** for security-sensitive models (fraud, moderation): can trivial perturbations flip decisions?
- **Fairness checks** where applicable: per-group metrics within tolerance.

**7. Serving-parity rehearsal:** evaluate the *deployed artifact* (the exact container), not the training-time model object — catches serialization/precision/preprocessing drift; plus a latency benchmark under representative load, and a small **backtest replay**: run the model over last month's real requests (features logged at serving time) and score outcomes — the best available proxy before shadow.

**Output:** a model report card stored in the registry — metrics + slices + calibration + latency/cost + lineage (data hash, code commit) — the artifact reviewers gate on ([02 - End-to-End ML Architecture.md](02 - End-to-End ML Architecture.md) Q10).

**Interview sound bite:**
> "Offline eval = temporal/group-aware splits with point-in-time features, a metric suite with slices and calibration, a champion-relative baseline ladder, bootstrap CIs, robustness tests on degraded inputs — and I evaluate the exact serving artifact with a latency benchmark, not the notebook model."

---

## Q28. What would you do if offline metrics improve but business metrics become worse?

**Answer.** A classic gap (Q6). Treat it as an incident with a diagnosis loop, and don't ship (or roll back) while investigating.

**Step 1 — Verify the measurement itself.**
- Is the A/B correct? **SRM check** (sample-ratio mismatch — randomization broken?), guardrail metrics sane, enough runtime, novelty/primacy effects, seasonality overlapping?
- Is the offline gain real and leak-free? (Temporal split? Point-in-time features? Resampling leakage?) A "leaked" model often *loses* online.
- Are business metrics attributed correctly — model version tagging on every prediction/log so effects map to the right model?

**Step 2 — Diagnose the gap by class:**

| Cause | Signature | Test / fix |
|---|---|---|
| **Proxy mismatch** | Offline metric ↑ but it's not the business driver | Offline predicted clicks better; business wants revenue — evaluate revenue-weighted ranking |
| **Distribution shift** | Live inputs differ from validation set | Compare logged serving features vs training features ([07 - Monitoring, Drift & Retraining.md](07 - Monitoring, Drift & Retraining.md) Q42); retrain on recent window |
| **Training–serving skew** | Offline great, online degraded, feature distributions differ at serving | Parity tests, logged-vs-batch feature diff (Q18) |
| **Feedback loops** | Model changes user behavior → training data no longer i.i.d. | E.g. optimizing CTR → clickbait → long-term engagement ↓; add diversity guardrails, explore unbiased data |
| **System/UX effects** | Latency p99 worse, different ordering/presentation, caching | Perf tracing; isolate model effect from system effect |
| **Slice losses masked by aggregate gains** | Aggregate ↑, key segment ↓ | Slice the A/B by user cohorts, geos, platforms |
| **Long-term vs short-term** | CTR ↑ now, retention ↓ in 4 weeks | Add delayed business metrics to guardrails; extend test horizon |
| **Threshold/policy miscalibration** | Probabilities shifted with new model; old threshold now wrong | Re-tune threshold on calibration of the new model |

**Step 3 — Decide:** roll back or hold the challenger; re-scope with the corrected objective (sometimes the *metric* was wrong, not the model — re-derive from business goal, Q5); document in the model registry so the next challenger avoids the same trap.

**Step 4 — Institutionalize:** add the missing guardrail that would have caught this (e.g. diversity, long-term retention), strengthen skew tests, and log the disagreement cases for the next training round.

**Interview sound bite:**
> "First I audit the measurement — SRM, leakage, attribution. Then I classify the gap: proxy mismatch, shift, skew, feedback loops, or system effects — using logged feature distributions and slice-level A/B results. I roll back while investigating, fix the root cause, and add the guardrail that would have caught it earlier."

---

## Chapter 4 — Interview checklist

- [ ] I pick models by constraint-elimination and always present a baseline ladder.
- [ ] I derive metrics from the decision the prediction drives, and match them to class balance.
- [ ] For imbalance: metric → threshold → class weights/focal loss → resampling (train-only) → data strategy.
- [ ] I run champion–challenger with pinned versions, CIs, slices, disagreement analysis, shadow, then A/B.
- [ ] My offline protocol uses temporal/group-aware splits, point-in-time features, and stress tests.
- [ ] When offline ↑ but business ↓, I audit measurement first, then classify the gap by cause.

**Next → [05.md — Online Inference & Serving](05.md):** the latency-critical right side of the architecture — serving patterns, APIs, and scaling to millions of requests.
