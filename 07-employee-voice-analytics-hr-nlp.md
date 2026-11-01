---
title: "Turning exit interviews into attrition signals: a dual-runtime HR-NLP toolkit"
subtitle: "How employee-voice-analytics combines multi-aspect sentiment, a 28-emotion taxonomy, zero-shot HR classification, BERTopic themes, flight-risk scoring, and PII guardrails — in one library that runs locally or on Databricks"
date: 2026-11-01
author: Royce Lam
tags: nlp, hr-analytics, sentiment-analysis, bertopic, keybert, goemotions, privacy, databricks, python
target: Medium / dev.to (cross-post)
hero_image: https://raw.githubusercontent.com/rollroyces/employee-voice-analytics/main/docs/assets/hero.jpg
hero_file: assets/eva-hero.jpg
hero_caption: Feedback streams resolving into a flight-risk gauge and topic clusters.
repo: https://github.com/rollroyces/employee-voice-analytics
---

# Turning exit interviews into attrition signals

HR has a data problem that isn't really about data. Every organization collects an enormous amount of unstructured employee feedback — exit interviews, annual survey free-text, pulse check comments, Glassdoor reviews — and then reads maybe ten of it. The rest sits in a SharePoint folder.

The reason isn't effort. It's that turning free text into something you can act on requires answering several different questions, and each question needs a different modeling approach. "How do people feel?" is sentiment. "What are they talking about?" is topic modeling. "Which of our taxonomy categories does this belong to?" is zero-shot classification. "Is this person about to leave?" is a composite of signals that doesn't have an off-the-shelf model.

[`employee-voice-analytics`](https://github.com/rollroyces/employee-voice-analytics) is a toolkit that does all five. This post is about how the pieces fit and why the privacy layer isn't an afterthought.

## Five capabilities, five different techniques

**1. Multi-aspect sentiment + GoEmotions taxonomy.**

Not one sentiment label — three layers. First a polarity: positive / neutral / negative. Then **28 fine-grained emotions** from the GoEmotions taxonomy. Then a **per-aspect** breakdown across eight HR-relevant dimensions: Compensation, Manager, Team, Workload, Growth, Tools, Culture, WorkLifeBalance.

This matters because "negative" is useless on its own. "Negative about Workload" is a staffing signal. "Negative about Compensation" is a band signal. The aspect dimension is where the actionability lives, and it has to be predicted separately from the polarity — a statement can be positive overall and negative about your promotion policy.

**2. HR taxonomy classification — zero-shot.**

A configurable taxonomy (Compensation, Leadership, Workload, DE&I, Career Growth, Tools, …) classified **zero-shot**, so you can change the taxonomy without retraining anything. This is the capability that makes the tool usable across organizations: your taxonomy is different from anyone else's, and it changes every year.

**3. Dynamic thematic discovery — BERTopic / KeyBERT.**

Static taxonomies capture what you already know to look for. Thematic discovery finds the thing nobody was looking for — the emerging concern that shows up in 30 exit interviews across 6 months but isn't in your taxonomy. BERTopic handles the clustering, KeyBERT gives you interpretable keywords per topic.

The "dynamic" part matters: run it quarterly and compare topic distributions across quarters. A topic growing from 4% to 15% of volume quarter-over-quarter is an early warning that no fixed taxonomy would flag.

**4. Qualitative flight-risk indicators.**

This is the layer that doesn't exist in any standard NLP library, because it can't — it's a composite. The toolkit computes:

- **Push / pull factor flags** — which HR dimensions are pushing someone out (bad) and pulling them to stay (good)
- **Q-over-q risk velocity** — is this person's risk *increasing* compared to their own previous responses, regardless of absolute level
- A **0–100 RiskScore** and a **RiskBand**

The velocity component is the interesting one. A RiskScore of 40 that was 20 last quarter is a different situation than a RiskScore of 40 that was 70 last quarter and is now improving. Most sentiment dashboards only show the current level and miss that entirely.

**5. Enterprise privacy guardrails.**

Feedback text contains names, emails, salary figures, and health disclosures. The toolkit applies:

- A **PII scrubber** (regex-based, or **Presidio** for something stronger)
- **k-anonymity suppression** for any slice below a minimum threshold

The k-anonymity piece is the one that's easy to skip and expensive to get wrong. If you slice feedback by department and only two people in that department responded, "the two people who left said X" is re-identifiable even after names are scrubbed — especially if the reader knows the team is small. Suppressing small slices isn't a nice-to-have; it's what makes the output safe to circulate to managers at all.

## Dual runtime: local CLI and Databricks Spark

The same library runs in two places:

- **Local CLI** — `eva analyze` for a laptop, a script, a one-off analysis
- **Databricks notebook** — `databricks_notebook.py` for scale, scheduled jobs, and when the feedback lives in Delta tables
- **Synthetic generator** — `eva generate-sample` for testing pipelines without real employee data

That last one is more useful than it sounds. When you're developing a pipeline that touches employee data, being able to run end-to-end on synthetic inputs means you're not handling production PII during development. It's a privacy practice as much as a convenience feature.

## The medallion architecture

```
Input (CSV / XLSX / JSON / Parquet / Delta)
   │
   ├── Bronze ── ingest_bronze()    raw + normalised columns, persisted as Delta
   ├── Silver ── cleaned, deduped, PII-scrubbed, aspect/emotion-enriched
   └── Gold ───  aggregated metrics, topics, RiskScore, suppressed small slices
```

Bronze / Silver / Gold is the standard lakehouse layering, and it's the right shape here because the **privacy scrub happens in Silver** — before anything becomes a published slice. Once data is in Gold, it's already been through scrubbing and k-anonymity suppression, so downstream consumers can't accidentally bypass the guardrails.

Doing the scrubbing earlier — say, in the notebook — means every downstream query has to remember to do it. Doing it once, in a layer, means the default is safe.

## Licensing

Dual-licensed: **AGPL-3.0-or-later**, with a commercial license available (`COMMERCIAL_LICENSE.md` in the repo).

For a toolkit that processes employee feedback, the license choice matters operationally. AGPL means if you run this as a service for a company, you'd owe source disclosure; the commercial route exists for organizations where that's not viable — and, frankly, where a procurement team needs a contract.

## Reading the output

A toolkit like this is only as good as its output contract, so it's worth being explicit about what comes back.

At the document level you get: per-aspect polarity and confidence, the emotion taxonomy labels, the classification against each configured taxonomy category, and any PII that was scrubbed (with the span removed from the text). At the cohort level you get topic distributions with keywords, push/pull factor aggregates, RiskScore distributions with band counts, and the trend of q-over-q velocity.

The cohort output is where the tool earns its keep, and there's one rule I'd underline: **always read the velocity and the distribution, not just the average.** A mean RiskScore of 45 across a quarter tells you almost nothing. A mean of 45 where the distribution is bimodal — one tight cluster at 20 and one cluster at 75 — tells you that you have a small group in serious trouble, which is actionable. A mean of 45 with everything at 45 means the score isn't discriminating and probably needs recalibrating against outcomes.

Same for topics: the interesting signal is in the *shape* of the distribution across quarters, not in the single most-discussed topic. "Termination process" being the top topic is expected; "Termination process" going from 2% to 19% of volume in two quarters is a finding.

## Honest limitations

- **RiskScore is a heuristic composite, not a validated clinical instrument.** It's a prioritization aid for where to look, not a decision about a person. I've deliberately not built anything that takes an employment action.
- **Taxonomy quality is the ceiling.** Zero-shot classification is only as good as the taxonomy definitions you write. Vague category descriptions produce vague classifications.
- **BERTopic on small corpora is unstable.** Under a few hundred documents, topic assignment shifts between runs. Use it on volume, not on one quarter's exit interviews.
- **Aspect sentiment needs enough per-aspect examples.** Classifying "negative about Workload" requires having seen people actually talk about workload.
- **The privacy layer is a strong default, not a guarantee.** Regex scrubbing misses context-dependent PII. If you're deploying this on real employee data, treat the guardrails as one layer of a broader compliance program.

## Where to start

```bash
pip install employee-voice-analytics
eva generate-sample --n 500 --out feedback.csv
eva analyze feedback.csv --taxonomy hr_taxonomy.yaml --report report.json
```

Repo is [github.com/rollroyces/employee-voice-analytics](https://github.com/rollroyces/employee-voice-analytics). If you're doing people analytics and want to compare notes on taxonomy design, risk scoring, or the privacy layer, open an issue or email me.

— Royce