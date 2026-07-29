# Final Year Project Report

**Intelligent Employee Productivity Tracking System (PTA)**
Department of Software Engineering, Sir Syed University of Engineering & Technology

📄 **[Download the full report (Word)](Intelligent-Employee-Productivity-Tracking-System-FYP-Report.docx)** — complete document with title pages, certificate, all chapters, figures and tables.

This page is the report in brief: what problem it solves, how, what was actually
achieved, and — just as deliberately — what it does not claim. The full technical
detail behind each part lives in [`../docs/`](../docs/).

---

## Objective

To design, implement and evaluate an end-to-end system that captures endpoint
behavioural telemetry from consenting participants, engineers it into
multi-granularity feature representations, and applies machine learning to **three
distinct questions** — current productivity, activity authenticity, and capability
development — while **reporting honestly on which of those questions the collected
evidence can and cannot answer**.

## The problem

Conventional workforce monitoring records *how long* an employee is present, not
*how* they work. In a distributed, remote-first workplace this falls short in five
concrete ways:

- it measures presence and input volume, not the character of the work;
- it cannot tell human-generated activity from automation, because the timing
  resolution needed to expose machine regularity is never stored;
- it flags statistical outliers without inferring intent, producing false-alarm
  rates that make the output unusable in practice;
- its predictive models are oriented toward *negative* outcomes such as attrition,
  offering nothing to an employee trying to grow; and
- it presents model confidence as though it were a direct measurement of a person.

A systematic review of forty papers (2020–2026) distilled these into three research
gaps: monitoring infers **outliers rather than intent**, predictive HR analytics is
dominated by **attrition rather than growth**, and endpoint behaviour is largely
**invisible to network-level security**.

## The solution

PTA is a deployed, tri-model behavioural-telemetry platform.

```
CAPTURE                STORE                 AGGREGATE               PREDICT & SERVE
5 Rust services   →   PostgreSQL 16     →   3 SQL feature       →   3 ML models   →  API → Dashboard
per machine           (raw events,          tables, per             (ONNX,
(+ launcher)          pooled, resilient)    device × window)        in-process)
```

1. **Capture** — five independent Rust services per Windows machine (process/ETW,
   keyboard, mouse, focus, activity derivation) supervised by one launcher. Only
   behavioural *dynamics* are recorded — timing, counts, kinematics — never
   keystroke content or screen contents.
2. **Store** — a central PostgreSQL 16 database behind PgBouncer, with a stable
   **device-centric canonical identity** that survives shared usernames and
   multi-machine users.
3. **Engineer** — watermark-driven, in-database functions roll raw events into three
   feature tables at **5-second, 1-minute and 1-day** granularity — one per model's
   question. A second, model-side stage adds heavy-tail-safe scaling, coverage-rule
   feature selection and application work-weighting.
4. **Model** — a **TranAD** reconstruction detector for automation, a **Temporal
   Fusion Transformer** engagement scorer, and a dilated-TCN growth forecaster
   backed by a **transparent rule engine** for the production growth path.
5. **Serve** — three ONNX models run in-process behind a stateless API (three
   load-balanced instances) and a public dashboard.

## What was achieved

Evaluation uses a **per-user chronological split** over a 17-day, four-subject
collection, comparing every model against an explicit baseline on all three splits.

| Model | Metric | Test | Baseline | Verdict |
|---|---|---|---|---|
| **Cheat detection (CDM)** | ROC-AUC ↑ | **0.9558** | 0.0988 (PR-AUC basis) | Generalises — train→test gap **−0.0008** |
| **Productivity (PSM)** | ROC-AUC ↑ | **0.8635** | 0.8563 (persistence) | Beats baseline on all splits, narrowly |
| **Growth (GPRM)** | MAE ↓ | 0.3772 | 0.3649 (train mean) | Beats baseline **only** on fitted data → ships as a rule engine |

Cheat detection is the strongest result: **PR-AUC 0.6377** against a 0.0988 random
baseline, recall **0.8810**, consistent across independent synthetic automation
vectors — including behavioural evasions for which it saw no labelled example. Its
two most decisive design choices moved it **from chance (0.52) to 0.9558**.

Nine defects found *by measurement rather than by a crash* are documented in the
report; the same discipline is what exposed and then closed the gap above. On the
project server the system runs at a **152 MB** model footprint over an **88 GB**
database.

## What it does *not* claim

Stated plainly in the report, because a result is only as honest as its boundaries:

- **No labelled productivity accuracy** — there are no human productivity labels, so
  the productivity model is validated as an engagement forecaster, not a truth meter.
- **Growth is unvalidated as a forecaster** — 17 days is too little longitudinal
  data to fit one; the shipped growth engine is *transparent*, not *proven*.
- **Cheat detection is tested against synthetic injection**, not a live adversary.
- **One deployment, one small cohort** — no population-level or production-load claim.

## Report structure

The full document ([download](Intelligent-Employee-Productivity-Tracking-System-FYP-Report.docx))
follows the university FYP template:

| Chapter | Contents |
|---|---|
| **1 · Introduction** | Overview, problem statement, objectives and status, research questions, scope |
| **2 · Literature Review** | Review methodology, productivity metrics, collection architectures, cheat detection, growth analytics, ethics, gaps and positioning |
| **3 · Design** | Process model, architecture, storage & identity design, the three model designs, data design, security & privacy, interface |
| **4 · System Development** | Technology stack, endpoint services, aggregation pipelines, ML pipeline, deployment, serving API & dashboard, rule-based growth engine |
| **5 · Testing** | Unit, integration and automation testing; datasets and splits |
| **6 · Results & Evaluation** | Verification checks, generalisation, per-model results, ablation, resource footprint |
| **7 · Conclusion & Future Work** | Scaling in time and machines |

## Related material in this repository

- [`../docs/`](../docs/) — the technical documentation set (tracker endpoints,
  backend & database, ML models, results).
- [`../report-figures/`](../report-figures/) — every figure used in the report and
  the companion research paper, in PNG and SVG.
- [`../diagrams/`](../diagrams/) — architecture and pipeline diagrams.
- [`../README.md`](../README.md) — project overview and repository layout.
