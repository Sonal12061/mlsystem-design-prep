# ML System Design: Tweet Toxicity Classifier

> A structured thinking guide for approaching text classification problems — using tweet toxicity detection as the reference problem.

---

## Table of Contents
- [Problem Understanding](#1-problem-understanding)
- [Data Landscape](#2-data-landscape)
- [Defining the Output](#3-defining-the-output)
- [System Mode](#4-system-mode-batch-vs-real-time)
- [Downstream Impact](#5-downstream-impact)
- [Production Context](#6-production-context)
- [Scale and Geography](#7-scale-and-geography)
- [Design Roadmap](#design-roadmap)

---

## 1. Problem Understanding

Before touching architecture or models, the first instinct should be to understand what the problem is actually asking. "Classify tweets as toxic or non-toxic" is deceptively simple. The real question is: **what does toxic mean in this context, and who gets to define it?**

A good starting point is to resist jumping into solutions. Every assumption made without verification is a potential rework later. The goal of early exploration is to shrink the solution space — not to impress with knowledge, but to eliminate irrelevant paths.

**Core concern:** Is this even a well-defined classification problem? Or is it a labeling problem disguised as one?

---

## 2. Data Landscape

### What to look for first

The nature of the data dictates the entire learning strategy. Two things need to be established early:

**a) Are labels present?**

| Data State | Implication |
|---|---|
| Labeled dataset available | Supervised classification is viable |
| No labels | Must consider weak supervision, human annotation pipelines, or semi-supervised approaches before modeling |

This is not just a technical question — it is a resource and timeline question. Labeling data is expensive and slow, and its absence fundamentally changes the project plan.

**b) Is it purely text?**

Tweets are not always text-only. They can include images, videos, GIFs, and links. A text-only model will miss toxicity embedded in memes or images. Knowing the data modality early determines whether a unimodal or multimodal system is needed — two very different engineering efforts.

**What the answers unlock:**
Once the data is known to be labeled and text-only, the problem can be confidently scoped as a supervised text classification task. Everything downstream of this confirmation is now tractable.

---

## 3. Defining the Output

### Binary vs. Multi-class

The number of output classes is not a minor detail — it shapes the entire model design, evaluation strategy, and business logic.

**Binary (Toxic / Non-Toxic)**
- Simpler model output, single probability score
- Threshold tuning is straightforward
- Evaluation centers on precision-recall tradeoff at a chosen operating point

**Multi-class (Hate Speech / Threat / Harassment / Spam / etc.)**
- Each class can have independent class imbalance
- Misclassification costs differ per class — conflating a death threat with spam is not the same as the reverse
- May warrant a hierarchical approach: *Is it toxic?* → *If yes, what kind?*

**Analogy:** A hospital emergency room running binary triage asks: *is the patient critical or stable?* A specialty routing system asks: *is it cardiac, neurological, or orthopedic?* The routing question changes the entire staffing and equipment model.

**What the answer unlocks:**
Class definition drives the metric choice. For binary, the question becomes what threshold to operate at and what error is more costly. For multi-class, it becomes which category errors are acceptable and which are not.

---

## 4. System Mode: Batch vs. Real-time

### Why this matters architecturally

The same model can power two fundamentally different systems depending on when predictions are needed.

| Dimension | Batch | Real-time |
|---|---|---|
| Latency tolerance | Minutes to hours | Sub-second |
| Use case | Nightly content audits, reporting | Pre-publish filtering, live moderation |
| Infrastructure | Scheduled pipelines, distributed processing | Low-latency serving, streaming queues |
| Model constraints | Larger models are acceptable | Model size and inference speed are first-class concerns |

**Common mistake:** Committing to design both systems without being asked. In practice, designing both simultaneously means designing neither well. The right approach is to scope to one mode, establish it thoroughly, and only extend if explicitly required.

**What the answer unlocks:**
Batch mode shifts focus to throughput, pipeline reliability, and cost. Real-time mode shifts focus to latency budgets, model compression, and serving infrastructure. These are different design conversations.

---

## 5. Downstream Impact

### Who is consuming the output?

This is one of the most overlooked questions in classification system design. The model does not exist in isolation — its predictions feed into something. That something determines **what the model must optimize for**, not just how the output is formatted.

| Consumer | Implication |
|---|---|
| Human moderator reviewing flagged content | High recall preferred — catch as many toxic tweets as possible; humans filter false alarms |
| Automated removal system | High precision required — false positives mean wrongly deleted content, with legal and reputational risk |
| Analytics / reporting dashboard | Calibrated probability scores matter — raw confidence values must be meaningful |
| Appeal or review queue | Confidence score alongside the label helps humans assess borderline cases |

**Analogy:** A smoke detector in a home versus one in a hospital ICU. Both perform the same detection task. But the ICU detector is tuned for high precision — a false alarm means evacuating patients on life support. The downstream consequence changes the acceptable error profile entirely.

**What the answer unlocks:**
The primary metric — precision, recall, F1, or AUC — cannot be chosen without knowing the downstream consumer. This is not a modeling preference; it is a product requirement.

---

## 6. Production Context

### Is there an existing system, and how will success be measured?

Understanding whether a model already exists in production changes two things: how success is defined, and how rollout should be planned.

**No existing model (greenfield)**
- The new model becomes the baseline
- Business metrics need to be defined from scratch: what does "working" look like in terms of moderation team efficiency, user reports, or content appeals?
- Offline evaluation metrics (F1, AUC-ROC) are necessary but not sufficient — online validation through staged rollout is needed

**Existing model or rule-based system**
- The new model must outperform the old one on a defined metric
- A/B testing infrastructure is required to run both systems in parallel and measure lift
- Regression testing matters — the new model should not regress on cases the old one handled well

### Monitoring considerations

Text data drifts faster than most other data types. Slang evolves, new forms of toxicity emerge, and cultural context shifts. A model trained at a point in time can silently degrade within months. Monitoring must account for:

- **Data drift:** Are incoming tweets looking different from training data?
- **Prediction drift:** Is the distribution of predicted labels shifting?
- **Business metric drift:** Are appeal rates, moderation escalations, or false positive reports trending upward?

**Typical rollout path:** Shadow mode → Canary release → A/B test → Full rollout

---

## 7. Scale and Geography

### Is one model expected to serve all regions?

Toxicity is culturally relative. A word that is a slur in one language may be neutral in another. A phrase that signals aggression in one culture may be idiomatic in another. A single global model trained predominantly on English data will have systematic blind spots.

**Design options:**

| Approach | When to use |
|---|---|
| Single multilingual model (e.g., XLM-RoBERTa) | When coverage and consistency matter more than per-region precision |
| Language detection → region-specific models | When toxicity patterns differ significantly enough across regions to warrant separate training |
| Base model + region-specific fine-tuning | Balanced approach — shared representations with locally adapted decision boundaries |

**Analogy:** A global legal firm has one company policy document, but each regional office has a local lawyer who understands local law. The base model is the company policy. Regional fine-tuning is the local lawyer.

**What the answer unlocks:**
This determines the data collection strategy. A global system needs labeled data across languages and cultures — which is expensive, slow, and introduces annotation consistency challenges. Flagging this early sets realistic expectations on data effort.

---

## Design Roadmap

Once the above questions are answered, the design can proceed in a structured sequence:

```
1. Data Pipeline          → Source (API, internal DB), ingestion, storage
2. Preprocessing          → Tokenization, URL/hashtag/emoji handling, language detection
3. Labeling Strategy      → If labels are absent: annotation guidelines, inter-annotator agreement
4. Feature Engineering    → TF-IDF as baseline → contextual embeddings (BERT, RoBERTa, XLM-R)
5. Modeling               → Start simple, iterate toward complexity based on metric gap
6. Evaluation             → Offline metrics aligned with downstream consumer requirements
7. Serving                → Batch pipeline or real-time endpoint based on mode decision
8. Monitoring             → Drift detection, prediction distribution tracking, business metric alerts
9. Rollout                → Shadow → Canary → A/B → Full production
```

---

## Key Principles

- **Clarify before designing.** Every unresolved assumption becomes a rework risk.
- **The downstream consumer defines the metric.** Optimization targets flow from product requirements, not modeling preferences.
- **Scope is a design decision.** Choosing what not to build first is as important as choosing what to build.
- **Toxicity is not universal.** A system designed without regional and linguistic awareness will fail quietly at scale.
- **Offline metrics are necessary, not sufficient.** A model that performs well on a test set must still be validated in production conditions.