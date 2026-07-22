# Intelligent Employee Productivity Tracking System (PTA)

**Final Year Project · Sir Syed University of Engineering & Technology**

A behavioural-analytics platform that captures low-level activity signals on each
machine, aggregates them server-side into ML-ready features, and produces
**productivity**, **cheat-detection** and **growth** predictions served through an
API and a live dashboard.

Conventional time trackers record *hours*. PTA reasons about *how* someone works:
whether input is genuinely human or automated, whether attention is focused or
idle, and whether capability is improving over time — three questions that each
need a different kind of evidence, and therefore a different model.

**Live dashboard:** [pta.emotionflow.site](https://pta.emotionflow.site)

---

## The pipeline

```
CAPTURE            STORE               AGGREGATE            PREDICT & SERVE
5 Rust services →  PostgreSQL 16   →   3 SQL feature    →   3 ML models  →  API → Dashboard
per machine        (raw event         tables (per            (ONNX,
(+ launcher)       tables, pooled)    device × window)       in-process)
```

1. **Capture** — five independent Rust services per Windows machine (process/ETW,
   keyboard, mouse, focus, activity derivation), supervised by one launcher.
2. **Store** — every signal written to a central PostgreSQL database through
   PgBouncer, as append-only raw event tables (the source of truth).
3. **Aggregate** — server-side PL/pgSQL functions roll raw events into three
   feature tables at three granularities (1 min / 5 sec / 1 day), keyed by a
   canonical per-device identity.
4. **Predict & serve** — three models write versioned predictions, exposed by a
   stateless Rust/Axum API.

---

## Headline results

Frozen evaluation on the collection window (2026-07-02 → 07-18, 4 subjects). Full
detail and honest limitations in [Results](docs/04-RESULTS.md).

| Model | Task | Test | Baseline | Verdict |
|---|---|---|---|---|
| **CDM v3** | Cheat / automation detection (ROC-AUC) | **0.9558** | 0.099 | Works — beats baseline on all splits, gap −0.0008 |
| **PSM v4** | Engagement forecast (ROC-AUC) | **0.8635** | 0.8563 | Works — beats persistence on all three splits |
| **GPRM** | Next-day growth (MAE) | 0.3772 | 0.3649 | No skill on 24 days → transparent rule engine ships |

CDM detects classic bots **and** fake-productivity behaviour (alt-tab thrashing,
jigglers, app cycling); gaming is prevented from scoring as productive by app
work-weighting.

---

## Documentation

| Document | Contents |
|---|---|
| [System Architecture & Design](docs/SYSTEM_ARCHITECTURE.md) | End-to-end design reference, with diagrams |
| [Endpoint Trackers](docs/01-TRACKER-ENDPOINTS.md) | The five capture services + launcher: what each records and how |
| [Backend API & Database](docs/02-BACKEND-API-AND-DATABASE.md) | Storage layers, canonical identity, feature aggregation, pooling, the Axum API, deployment |
| [ML Models](docs/03-ML-MODELS.md) | Design, algorithm per layer, and preprocessing for CDM / PSM / GPRM |
| [Results](docs/04-RESULTS.md) | Consolidated numbers, generalisation evidence, defects fixed, and limits |
| [Architecture Diagrams](docs/architecture-diagrams.md) | Diagram sources |

Rendered figures live in [`/report-figures`](report-figures) and
[`/diagrams`](diagrams).

---

## Repository layout (backend source is kept in a separate private repo)

| Path | Contents |
|---|---|
| `process-monitoring-service/` | Process / ETW capture agent |
| `keys_input-monitoring-service/` | Keyboard capture service |
| `mouse_input-monitoring-service/` | Mouse capture agent + backend |
| `focus-monitoring-service/` | Window-focus capture service |
| `activities-monitoring-service/` | Activity derivation syncer |
| `pta-launcher/` | Supervisor that starts all agents on a machine |
| `pta-backend-api/` | Axum HTTP API + SQL for identity / prediction tables |
| `pta-ml/` | Python ML package (PSM / CDM / GPRM, training, ONNX export) |

---

## Team

Final Year Project team — behavioural telemetry, data engineering, machine
learning and deployment.

- **Machine learning & preprocessing** (lead)
- **Endpoint capture services**
- **API, deployment & dashboard**
- **Data engineering**
