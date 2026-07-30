# Data Engineering: The Two-Phase Pipeline

Raw behavioural telemetry is never fed to a model directly. Between the tracker and
the neural networks sit **two independent data-engineering phases**, deliberately
separated because they answer different questions:

- **Phase 1 — server-side aggregation** runs *inside PostgreSQL*, in PL/pgSQL. It
  answers **"what happened"**: it condenses millions of raw events into a handful of
  engineered feature tables at three time granularities.
- **Phase 2 — model-side preprocessing** runs *in Python*, at training and scoring
  time. It answers **"how do we present this to a network without leaking
  information or drifting over time"**: numeric coercion, feature selection,
  encoding, heavy-tail-safe scaling, splitting, and windowing.

```
 TRACKERS         PHASE 1  (in database, PL/pgSQL)        PHASE 2  (in Python, at model time)
 5 services   →   20 raw    →  3 aggregation   →  3 feature   →  load via   →  preprocess  →  models
 per machine      event         functions          tables         v_* views     + window       (CDM/PSM/GPRM)
                  tables        (+ 3 watermarks)   (the "3+3")
```

Everything below is drawn from the actual code; file references are listed under
[Sources](#sources).

---

## Phase 1 — Server-side aggregation (PL/pgSQL, in-database)

The first stage runs entirely inside the database. The design principle is that the
expensive raw-to-feature computation should run **next to the data**, not by dragging
millions of rows across the network into an application.

### Input: 20 raw event tables, five families

The trackers write into roughly twenty raw event tables. Every aggregation function
reads from **five master event tables** — `process_events`, `activities`,
`focus_events`, `key_events`, `mouse_events` — the primary source per family
(process, activity, focus, keyboard, mouse), joining a few related tables such as
`process_resource_history` where needed.

### The "3 + 3" tables it produces

Three PL/pgSQL functions, one per target, each write **one feature table** and are
paired with **one watermark table** — the 3 + 3 tables that aggregation produces:

| Aggregation function | Feature table | Window | Cols | Watermark table | Feeds |
|---|---|---|---|---|---|
| `populate_cheat_detection_features()` | `cheat_detection_features` | **5 seconds** | ~321 | `cheat_detection_watermark` | CDM (automation detection) |
| `populate_unified_time_series()` | `unified_time_series` | **1 minute** | ~211 | `uts_watermark` | PSM (productivity) |
| `populate_employee_growth_features()` | `employee_growth_features` | **1 day** | ~214 | `employee_growth_watermark` | GPRM (growth) |

Each granularity is matched to the question its model must answer: automation is
betrayed by sub-second timing regularity (5 s), productivity is a per-minute profile
(1 min), and growth is a matter of days (1 day). Every row is aggregated **per
device** and keyed by the **canonical identity**, so features for two people are
never blended into one row.

### Techniques used in Phase 1

These are the methods that make in-database aggregation cheap and correct as the raw
tables grow to tens of millions of rows.

- **Watermark-incremental processing.** Each function records how far it has
  processed in its watermark table (`last_processed_end` / `last_processed_date`).
  On the next run it starts from the watermark and aggregates only the new window
  range, so cost depends on *newly arrived* data, not on total history.

- **Smart gap-skip.** Before doing any heavy work, the function takes the `MIN`
  timestamp across all five raw families and jumps the start straight to where data
  actually exists. This prevents running the heavy aggregation CTEs over empty time
  ranges left behind by service downtime. If a range contains no events at all, the
  watermark is simply advanced and the function exits.

- **Batch capping.** In automatic (watermark) mode a run is capped to a bounded slice
  (e.g. a 10-minute window for the 1-minute table) so a single scheduled call can
  never balloon. Explicit start/end parameters override the cap for backfills.

- **Grid alignment.** Windows are aligned to fixed boundaries with `date_trunc` —
  a 5-second grid for cheat features, minute boundaries for the time series, calendar
  days for growth — so window edges are deterministic and reproducible.

- **CTE-based aggregation (no staging tables).** The computation is expressed as a
  chain of Common Table Expressions inside the function (the 1-minute function uses
  eleven), one family at a time, joined on the aligned window key. There are **no
  physical staging tables**; the intermediate results live only for the duration of
  the query.

- **Idempotent re-runs.** Each function processes a target range with a
  `DELETE`-then-`INSERT` for that range, so re-running a window **replaces** its
  output rather than duplicating it. Any range can be safely recomputed.

### Output exposed through canonical views

The feature tables keep their raw per-device values; a set of **canonical `v_*`
views** (`v_cheat_detection_features`, `v_unified_time_series`,
`v_employee_growth_features`) sits on top and exposes the resolved canonical
identity. Consumers read the views, never the base tables, so identity resolution
stays auditable and reversible. These views are the hand-off point to Phase 2.

---

## Phase 2 — Model-side preprocessing (Python, at model time)

When the backend loads the feature tables into the ML models, the second phase runs.
It is separate from Phase 1 by design: Phase 1 said *what happened*; Phase 2 decides
*how to present it to a network*. Everything here is fit on the **training split
only**, so no future or validation information leaks into the transform.

### Entry point: loading through the canonical views

The loader pulls the three canonical `v_*` views into pandas DataFrames from a
read replica, with time-range filtering and row caps so a multi-million-row table is
never dragged fully into memory. This view-based read is the "backend → model" path.

### Application work-weighting (a Phase-2 correction)

Before scoring productivity, each window is re-classified by application. This is
done in Phase 2 because the classification columns shipped by Phase 1 were
**unusable** — `is_productivity_app` marked `cs2.exe`, `VALORANT` and `TslGame`
(PUBG) as productive. Phase 2 rebuilds the classification from `active_app_name`
against a hand-curated category list and assigns a **work weight**:

| Category | Weight | Examples |
|---|---|---|
| development, office | **1.0** (productive) | `code.exe`, `winword.exe`, `excel.exe` |
| browser, communication, system, unknown | **0.5** (neutral) | browsers, chat, unclassified |
| gaming, social, entertainment | **0.0** (distraction) | `cs2.exe`, `VALORANT`, streaming |

The weight multiplies the window's input events (`productive_events = total_input_events × work_weight`),
so **time in a game contributes nothing** to a productivity score — closing the most
obvious way to game the system. Window *titles* reveal content and are OFF by
default, consistent with the project's metadata-not-content principle.

### The preprocessing transform, fit on train only

The `Preprocessor` performs these steps in order:

1. **Numeric coercion.** PostgreSQL `NUMERIC`/`Decimal` object columns are coerced to
   `float64` and `NULL` is filled with `0.0`.

2. **Coverage-rule feature selection.** Columns that are never populated, or that
   only begin reporting part-way through the collection, are dropped — decided on the
   training split alone. Without this, all-`NULL` columns become constant `0.0`
   features that waste model capacity (the 1-minute table alone ships ~73 such
   columns). A `keep_sparse` whitelist protects rare-but-decisive signals (e.g. the
   window-switch shortcut that betrays alt-tab thrashing).

3. **Cyclical encoding.** Temporal columns (hour, day-of-week, …) are encoded as
   `sin`/`cos` pairs so that, for example, 23:00 and 00:00 sit next to each other.
   These already live in `[-1, 1]` and are **kept unscaled**.

4. **Categorical integer-encoding.** Low-cardinality categoricals get an integer code
   from a vocabulary built on the training split; `0` is reserved for
   unseen/unknown values at scoring time.

5. **Heavy-tail-safe scaling.** Behavioural counters (bytes sent, context switches,
   focus milliseconds) are extremely heavy-tailed — one burst can sit thousands of
   standard deviations from the mean and, with a plain scaler, produce `|z|` in the
   thousands on unseen data. A **signed `log1p`** transform (`sign(x)·log1p(|x|)`)
   compresses that tail while staying monotonic and sign-preserving.

6. **Standardisation on train.** A `StandardScaler` is fit on the signed-log values of
   the **training split only**, then applied to all splits.

7. **Clipping.** Scaled values are clipped to **±10σ** (`CLIP_SIGMA = 10`) to bound
   whatever outliers survive the log compression.

   The full transform is: `x → sign(x)·log1p(|x|) → (x − mean)/scale → clip(±10σ)`,
   with cyclical columns appended unscaled.

### Splitting and windowing (no leakage)

- **Per-user chronological split — 70 / 15 / 15.** Within *each* user, the earliest
  70 % of windows are training, the next 15 % validation, the last 15 % test. A model
  is always judged on a subject's *future*, never on interpolation, and every subject
  appears in all three folds. (A single global date cut was rejected because a
  subject who stops early would vanish from the test set.)

- **Per-user sequence windowing.** Rows are assembled into sliding windows of shape
  `(N, L, F)` **within each user only** — no window ever spans two people. Training
  can stride to skip near-duplicate overlapping windows and save memory; scoring
  always uses stride 1 so every row gets a prediction.

### Freezing the transform for serving

At deployment the fitted preprocessor is **frozen and exported to JSON**: the exact
feature column order, the scaler mean/scale vectors, the clip bound, and the cyclical
columns. The serving layer reproduces the training-time transform **in-process**
alongside the ONNX models — no PyTorch, no re-fitting on live data — so the mapping
from raw features to model input is a constant and a familiar source of silent drift
becomes a guarantee.

---

## Why two phases

| | Phase 1 | Phase 2 |
|---|---|---|
| **Runs where** | Inside PostgreSQL | In Python, at model time |
| **Language** | PL/pgSQL | pandas / NumPy / scikit-learn |
| **Question** | What happened? | How to present it to a network? |
| **Unit of work** | One window × device | One training/scoring run |
| **Key methods** | Watermark-incremental, gap-skip, batch-cap, CTEs, grid alignment, idempotent DELETE+INSERT | Coercion, coverage-rule selection, cyclical + categorical encoding, signed-log scaling, clip ±10σ, per-user split, sequence windowing, frozen export |
| **Output** | 3 feature tables (+ 3 watermarks), read via `v_*` views | Model-ready tensors `(N, L, F)` and a frozen preprocessor |

Splitting the work this way means the heavy reduction happens once, in the database,
and is shared by every model; while the model-specific presentation — which columns,
which scaling, which windows — stays in Python where it can be versioned, tested, and
frozen for reproducible serving.

---

## Sources

Every claim above is taken from the working code and SQL:

| Topic | File |
|---|---|
| 1-minute aggregation, watermark, gap-skip, batch-cap, CTEs | `process-monitoring-service/sql/unified_time_series/populate_unified_time_series.sql` |
| 5-second cheat features, grid alignment | `process-monitoring-service/sql/cheat_detection/002_populate_cheat_detection_features.sql` |
| Daily growth features | `process-monitoring-service/sql/growth_prediction/002_populate_employee_growth_features.sql` |
| Watermark tables | `.../cheat_detection/001_create_cheat_detection_features.sql`, `.../growth_prediction/001_create_employee_growth_features.sql` |
| Canonical `v_*` views loaded into models | `pta-ml/pta_ml/data/loader.py`, `pta-ml/pta_ml/config.py` |
| Full Phase-2 transform (coerce, select, encode, log, scale, clip, split, window) | `pta-ml/pta_ml/data/preprocess.py` |
| Application work-weighting | `pta-ml/pta_ml/features/appclass.py` |
| Frozen preprocessor export for serving | `pta-ml/scripts/phase7b_export_preprocessing.py` |

See also [`03-ML-MODELS.md`](03-ML-MODELS.md) for the models that consume these
features and [`02-BACKEND-API-AND-DATABASE.md`](02-BACKEND-API-AND-DATABASE.md) for
the storage and identity layer.
