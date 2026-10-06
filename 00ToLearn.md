# 00 · How to Learn With This Repo

> Read this once (10 minutes), pick a plan, then open [System Design Fundamentals.md](System Design Fundamentals.md). Come back to this file when you change goals.

---

## 1. What ML system design actually tests

An ML system design interview is not a knowledge quiz — it's **simulated technical leadership under ambiguity**. You get a vague prompt ("Design fraud detection for our payments platform") and 35–45 minutes. What's being scored:

| Signal | What it looks like | Where this repo teaches it |
|---|---|---|
| **Scoping** | Numbers on the whiteboard in the first 5 minutes: QPS, users, p99, latency budget | [System Design Fundamentals.md](System Design Fundamentals.md) Q7 |
| **ML framing** | Task, unit of prediction, label definition, primary + guardrail metrics | [System Design Fundamentals.md](System Design Fundamentals.md) Q4–5 |
| **Architecture fluency** | The end-to-end diagram, drawn and *narrated* — not recited | [End-to-End ML Architecture.md](End-to-End ML Architecture.md) Q8 |
| **Depth on demand** | Instant, concrete answers when the interviewer drills one component | all chapters |
| **Trade-off literacy** | Every choice stated as "X over Y because Z, at the cost of W" | every "sound bite" section |
| **Production sense** | Failure modes, degradation, monitoring, rollback, cost, security | [Online Inference & Serving.md](Online Inference & Serving.md)–[Cost, Security & Practical Trade-offs.md](Cost, Security & Practical Trade-offs.md) |
| **Structure under pressure** | The same skeleton every time, no matter the prompt | [System Design Cases.md](System Design Cases.md) |

The failure mode this repo exists to prevent: knowing fifteen model architectures and no serving path, or buzzing confidently about feature stores while your metrics increase and your business metrics decrease ([Model Training & Evaluation.md](Model Training & Evaluation.md) Q28).

---

## 2. Pick your plan

### 🌱 Plan A — "Ground up" (3–4 weeks, 1–2 h/day)
*You're new to ML systems or switching from research/analytics into engineering roles.*

1. **Week 1 — Chapters 1–2.** Fundamentals, the 12 components, the full architecture. Don't memorize — *redraw* Q8's diagram from memory each morning until it takes <2 minutes.
2. **Week 2 — Chapters 3–5.** Data/features (the senior chapter — go slow), evaluation, serving. End each chapter by reciting its checklist out loud.
3. **Week 3 — Chapters 6–9 + one case.** Reliability, monitoring, deployment, cost/security. Run Q59 end-to-end as your first full 35-minute case, timed, spoken aloud.
4. **Week 4 — Cases.** Two cases per day on a whiteboard/paper ([System Design Cases.md](System Design Cases.md)), alternating with re-skimming your weakest theory chapter. Finish with the full checklist in [10_Summary.md](10_Summary.md).

### 🚀 Plan B — "Interview in two weeks" (10–14 days, 2 h/day)
*You know ML; you need interview shape and production vocabulary.*

1. **Days 1–2:** Skim [System Design Fundamentals.md](System Design Fundamentals.md)–[End-to-End ML Architecture.md](End-to-End ML Architecture.md) fast; spend the time *drawing* — both diagrams from memory, five times each.
2. **Days 3–5:** **Full read of [Data & Feature Engineering.md](Data & Feature Engineering.md)–[Online Inference & Serving.md](Online Inference & Serving.md).** These three (data, evaluation, serving) are where most strong-candidate interviews are actually won or lost. Do every chapter checklist.
3. **Days 6–7:** [Scalability, Reliability & Availability.md](Scalability, Reliability & Availability.md)–[Model Deployment Strategies.md](Model Deployment Strategies.md). Learn the deployment patterns as *composition* (shadow → canary → blue-green → warm standby).
4. **Days 8–9:** [Cost, Security & Practical Trade-offs.md](Cost, Security & Practical Trade-offs.md) + first five cases of [System Design Cases.md](System Design Cases.md), spoken aloud, timed.
5. **Days 10+:** Remaining cases, then **[10_Summary.md](10_Summary.md) only** (sections A–D) the final day. Cases per day until you can whiteboard Q59/Q61/Q67 cold ([System Design Cases.md](System Design Cases.md) practice plan).

### ⚡ Plan C — "Interview tomorrow" (3–4 hours)
*The panic plan. It works because it's the 20% that carries 80% of the signal.*

1. **[10_Summary.md](10_Summary.md) in full** (45 min) — skeleton, diagrams, decision tables, seniority sentences.
2. **[End-to-End ML Architecture.md](End-to-End ML Architecture.md) Q8** and **[Online Inference & Serving.md](Online Inference & Serving.md) opening diagram** — redraw both from memory (20 min).
3. **Three cases aloud:** Q59 (recsys), Q61 (fraud), Q67 (RAG) against the skeleton, 15 min each in [System Design Cases.md](System Design Cases.md) (45 min).
4. Re-read **[System Design Fundamentals.md](System Design Fundamentals.md) Q7** (requirements checklist) — the first five minutes of your interview (15 min).
5. Stop. Sleep. Section G of the summary is the full protocol.

---

## 3. The practice method (this is what makes it stick)

Reading builds recognition. Interviews need **retrieval under time pressure**. Three drills:

**Drill 1 — Diagram burn-in (10 min/day, early weeks).**
From a blank page, draw (a) the end-to-end architecture and (b) the serving path, then check against [End-to-End ML Architecture.md](End-to-End ML Architecture.md) Q8 / [Online Inference & Serving.md](Online Inference & Serving.md). Stop when both draw themselves.

**Drill 2 — Case sprints (20–40 min, from week 3, the core drill).**
Pick a case from [System Design Cases.md](System Design Cases.md) *without* reading its answer. Speak through the 7-step skeleton out loud (literally aloud — silent practice skips the hard part), drawing as you go. Timer at 35 minutes. Then read the model answer and mark three things:
- What you missed entirely (→ that's your reading assignment)
- What you included but can't defend one level deeper (→ weakness masquerading as strength; go read the cross-referenced question)
- Where you took >5 minutes without stating a decision (→ practice the sound bites)

**Drill 3 — Component rapid-fire (10 min, interview week).**
Have a friend/bot/random question generator drill single questions from [10_Summary.md](10_Summary.md) section F; answer in ≤2 min with one diagram or table each. Anything you can't do in 2 minutes: re-read that section.

**Spaced repetition for the checklists:** every chapter ends with one. On day N+1 and N+4, redo the previous chapter's checklist *from memory* before starting new material. Nothing else needs flashcards — the questions are already atomic.

---

## 4. Chapter dependency map

```
         ┌──────────── 00ToLearn (you are here) ────────────┐
         ▼                                                  │
     [01.md] Fundamentals ──► [02.md] Architecture ──► [03.md] Data/Features
                                                          │
              ┌───────────────────────────────────────────┘
              ▼
          [04.md] Training/Eval ──► [05.md] Serving ──► [06.md] Scale/Reliability
              │                                          │
              │        ┌─────────────────────────────────┘
              ▼        ▼
          [07.md] Monitoring/Drift ──► [08.md] Deployment ──► [09.md] Cost/Security
                                      │
                                      ▼
          [10.md] Ten Design Cases (uses everything) ──► [10_Summary.md] Cheat sheet
```

Chapters 1–2 are load-bearing: take them slowly. If you're time-boxed, [Data & Feature Engineering.md](Data & Feature Engineering.md)→[Online Inference & Serving.md](Online Inference & Serving.md) is the minimum-production-grade core, and cases in [System Design Cases.md](System Design Cases.md) will pull you back to what you skipped — that's fine; *"jump back when the case needs it"* is how most engineers actually use this.

---

## 5. How to read each chapter

1. **"In this chapter" table** — the one-look map; your notes can literally be this table expanded by one line per question.
2. **Each Q:** read the **Answer** header sentence, then decide: skim to the table/diagram, or read fully if the checklist item for it is unchecked.
3. **Sound bites** — read them *out loud once*; they compress a full section into second-nature phrasing.
4. **End checklist** — the exit gate. Moving on with unchecked items is allowed exactly once per chapter, then it's debt.

---

## 6. Prerequisites (honest version)

| Prerequisite | Minimum to get full value | Patch it fast with |
|---|---|---|
| ML basics | What classification/regression/ranking is; train vs test; overfitting | Any ML course's first weeks |
| Mild coding/systems maturity | Why load balancers, caches, replicas exist | Any general system design primer's first chapter |
| Statistics | p-values vs CIs at an intuition level | [System Design Fundamentals.md](System Design Fundamentals.md) Q5–6 + [Model Training & Evaluation.md](Model Training & Evaluation.md) Q26 in this repo |
| Cloud familiarity | Nice-to-have; tool names are illustrative, not endorsements | Chapter tables name the usual suspects per slot |
| Interview practice | Not required to start — that's what [System Design Cases.md](System Design Cases.md) is for | Drill 2 above, weekly |

Tool names (Feast, Tecton, MLflow, Triton, KServe, Airflow…) appear to make the discussion concrete. **Never** present tools as the answer in interviews — present the *capability* ("a low-latency online feature look-up layer") and name tools as examples.

---

## 7. FAQ

**Q: Should I use this or a standard system design course?**
Both if you can — the general course gives you the boring-but-load-bearing base ([Scalability, Reliability & Availability.md](Scalability, Reliability & Availability.md) leans on it hard); this repo covers everything it doesn't: the ML-specific axes. If you only pick one and the role is ML-titled, pick this.

**Q: I'm a researcher, not an engineer. Which parts matter most?**
[System Design Fundamentals.md](System Design Fundamentals.md) Q4–7, [Data & Feature Engineering.md](Data & Feature Engineering.md) (all), [Model Training & Evaluation.md](Model Training & Evaluation.md) (all), and [Monitoring, Drift & Retraining.md](Monitoring, Drift & Retraining.md) — scoping, data, evaluation, and monitoring are exactly what proves you can move a model from notebook to product.

**Q: I'm aiming for MLOps/platform roles. Priority order?**
[End-to-End ML Architecture.md](End-to-End ML Architecture.md) → [Online Inference & Serving.md](Online Inference & Serving.md) → [Scalability, Reliability & Availability.md](Scalability, Reliability & Availability.md) → [Model Deployment Strategies.md](Model Deployment Strategies.md) → [Data & Feature Engineering.md](Data & Feature Engineering.md) → the rest of [Cost, Security & Practical Trade-offs.md](Cost, Security & Practical Trade-offs.md) → cases.

**Q: Do I need to memorize the sound bites?**
No — memorize the *decision tables* behind them. A sound bite recited without its backing table collapses under the first follow-up question. The tables are [10_Summary.md](10_Summary.md) section C.

**Q: When do I stop studying?**
When you can run any of the ten cases in [System Design Cases.md](System Design Cases.md) cold, against the skeleton, with at least one voluntary cost trade-off and one failure story, inside 40 minutes. That's the bar; more study past it has steeply diminishing returns.

---

*Ready? → [01.md — System Design Fundamentals](01.md)*
