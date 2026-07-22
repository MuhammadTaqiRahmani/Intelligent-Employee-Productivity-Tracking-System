# Backend API & Database

**Intelligent Employee Productivity Tracking System (PTA)**

Everything the endpoint trackers produce lands in one central PostgreSQL 16
database. This document covers the storage model, the canonical identity layer,
the server-side feature aggregation, and the stateless HTTP API that serves
predictions to the dashboard.

Data flows **downward only**, and the raw layer is the single source of truth:

```
RAW events  →  FEATURES (aggregated)  →  PREDICTIONS  →  API  →  Dashboard
   (append-only truth)   (rebuildable)      (rebuildable)
```

---

## 1. Database layers

| Layer | Tables | Role |
|---|---|---|
| **Raw** | 20 event tables (process, keyboard, mouse, focus, activities) | Source of truth; one family per capture domain |
| **Identity** | `employee_identity`, `employee_identity_override` | Maps each device to a canonical employee (§3) |
| **Feature** | `unified_time_series`, `cheat_detection_features`, `employee_growth_features` | ML-ready engineered features (§4) |
| **Watermark** | `uts_watermark`, `cheat_detection_watermark`, `employee_growth_watermark` | How far each aggregator has processed |
| **Prediction** | `productivity_predictions`, `cheat_predictions`, `cheat_alerts`, `growth_predictions` | Model outputs served by the API (§5) |
| **Staging** | `stg_*` (internal) | Small per-device scratch tables used inside the aggregators |

Because the raw layer is append-only and everything below it is derived, any
feature or prediction can be recomputed from scratch. That makes aggregation
changes safe and reversible — a redesign never risks the historical record.

---

## 2. Connection pooling — PgBouncer

Each monitored machine runs five write-heavy services. With a fleet of machines,
`machines × services` direct connections would exhaust PostgreSQL's connection
limit. **PgBouncer in transaction-pooling mode** multiplexes thousands of client
connections onto a small pool of real backend connections.

Two design points make this correct rather than merely convenient:

- **Named prepared statements** sized to the pooler, so statement caching and
  transaction pooling coexist (an unnamed-statement workaround was rejected in
  favour of `max_prepared_statements` configured on the pooler).
- **Idempotent writes** on the prediction tables (unique constraints +
  `ON CONFLICT DO NOTHING`), so a retried transaction — a normal event under
  pooling — never creates duplicates.

---

## 3. Canonical identity layer

A single employee may use more than one machine, and two *different* people can
share the same operating-system username. Raw events therefore cannot be keyed
directly on the OS username.

**Principle: identity is device-centric — one machine = one employee.**

- `employee_identity` maps each `device_id` to a `canonical_user_id`, resolved
  from observed keyboard/mouse activity per device.
- When one username appears on multiple devices, the id is **device-qualified**
  (e.g. `alice@laptop-1`, `alice@laptop-2`), so two people can never be merged
  into one row.
- `employee_identity_override` is a manual mapping that always wins — used to
  attach real names, and to correct a genuine collision (two people who happened
  to share a username were separated here).
- A scheduled refresh keeps the mapping current, so a newly-seen machine is
  picked up automatically.

The feature aggregators read identity from this layer, so every downstream
feature and prediction is labelled with a **stable canonical identity** rather
than a raw, ambiguous username. Consumer-side `v_*` views join feature tables to
the canonical identity, so identity corrections take effect without rewriting the
aggregated data.

---

## 4. Feature aggregation

Three server-side PL/pgSQL functions (~4,000 lines total) turn raw events into
the three feature tables. Each produces **one row per (time window × device)** —
never blending multiple machines into a single row.

| Feature table | Function | Window | Feeds |
|---|---|---|---|
| `unified_time_series` (UTS) | `populate_unified_time_series` | **1 minute** | Productivity Scoring Model (PSM) |
| `cheat_detection_features` (CDF) | `populate_cheat_detection_features` | **5 seconds** | Cheat Detection Model (CDM) |
| `employee_growth_features` (EGF) | `populate_employee_growth_features` | **1 day** | Growth Prediction & Recommendation (GPRM) |

The three feature tables are **wide by design** — roughly 211, 321 and 214
columns respectively — because they encode an explicit, auditable feature
catalogue rather than opaque embeddings.

### Aggregation design

- **Watermark-incremental.** Each function processes only the window range since
  its last run, then advances its watermark. This keeps aggregation cheap as the
  raw tables grow to millions of rows.
- **Per-device partitioning.** Every window is aggregated separately per device,
  keyed by canonical identity, so features for different employees are never
  mixed. (An earlier bug that blended all users into one row was found and fixed;
  correctness here is now enforced by construction.)
- **Staging tables.** For each device, the relevant slice of each raw table is
  materialised into a small `stg_*` table first, so the heavy aggregation reads
  tiny inputs instead of repeatedly scanning the full raw tables.
- **Idempotent.** Re-running a window replaces (not duplicates) its output, so
  any range can safely be recomputed.

```
raw event tables → stg_* per-device slice → aggregation fn (1 row/window×device)
        watermark ↕ read/advance                    ↓
                                              feature table
```

---

## 5. Prediction tables

Model outputs are precomputed and stored, so the API never runs a model on the
request path.

- `productivity_predictions` — score, level, role alignment, per-category
  breakdown.
- `cheat_predictions` + `cheat_alerts` — risk score, dominant vector, evidence;
  an alert row is written only when a threshold is crossed.
- `growth_predictions` — growth score, trajectory, forecasts, strengths /
  weaknesses, ranked recommendations.

Every prediction table is keyed on **`(user, window/date, model_version)`**. This
is what lets a retrained model (`v2`, `v3`, …) land *beside* the existing rows
instead of overwriting them, so versions can be compared directly and a rollback
is just a version filter.

---

## 6. Serving API — Rust / Axum

A **stateless** HTTP API (Axum, Rust) serves reads from the prediction and
feature tables. Because predictions are precomputed, read latency is low and
independent of model complexity, and the API tier can be scaled horizontally
behind a load balancer.

| Endpoint | Purpose |
|---|---|
| `GET /health` | Liveness check |
| `GET /api/v1/dashboard` | Fleet-level summary |
| `GET /api/v1/employees` | Employee list |
| `GET /api/v1/employees/:id/stats` | Per-employee aggregate stats |
| `GET /api/v1/employees/:id/timeline` | Per-employee activity timeline |
| `GET /api/v1/inference/:id/productivity` | Latest productivity prediction (PSM) |
| `GET /api/v1/inference/:id/cheat-detection` | Latest cheat risk (CDM) |
| `GET /api/v1/inference/:id/growth` | Latest growth prediction (GPRM) |
| `GET /api/v1/ml/*` | ML result/metric endpoints backing the dashboard |

All endpoints are keyed by the **canonical** identity, consistent with the
feature and prediction layers.

The API is **read-only** with respect to the data pipeline: it never mutates raw
events, features or predictions. Aggregation and scoring run as separate
scheduled jobs, so a burst of dashboard traffic can never interfere with the
write path.

---

## 7. Deployment & operations

- **In-process ONNX inference.** Models are exported to ONNX and run *inside* the
  scoring job / API process — no separate model server and no network hop on the
  prediction path.
- **TLS / reverse proxy.** nginx terminates TLS in front of the API and the
  dashboard; the public dashboard is served at `pta.emotionflow.site`.
- **Scheduled pipeline.** Aggregation and scoring run on a schedule; each uses a
  file lock so a slow run is never joined by the next one.
- **Automated disk safety.** A scheduled guard watches disk usage. On pressure it
  first trims logs; only if that is insufficient does it archive the oldest
  **already-aggregated** raw data off-node and prune it — never touching the
  engineered feature tables, and never deleting raw data a feature aggregator has
  not yet consumed.

---

## 8. Scaling model

```
N machines (5 services each)
        │  pooled connections
        ▼
   PgBouncer ──► PostgreSQL primary ──► aggregation + scoring jobs
                      │ streaming replication
                      ▼
                 read replica* ──► stateless API tier* ──► dashboard
                                        ▲
                                  load balancer*
```

The application code does not change between a single-node deployment and a
horizontally-scaled fleet. Items marked `*` (read replica, multi-instance API,
load balancer) are part of the scaling design; a small deployment runs a single
node. The design targets were validated in staged phases (PgBouncer → read
replica → nginx load balancing).

---

## 9. Design principles

- **Raw is the source of truth; everything else is derived** and rebuildable.
- **Device-centric identity** that never merges two people.
- **Incremental, idempotent aggregation** keeps cost bounded and any window
  recomputable.
- **Precompute predictions, serve them cheaply** — the API stays fast and
  stateless.
- **Safety by construction** — pooling, per-device partitioning, idempotent
  writes and the disk guard are arranged so routine operations cannot corrupt the
  engineered data the ML layer depends on.
