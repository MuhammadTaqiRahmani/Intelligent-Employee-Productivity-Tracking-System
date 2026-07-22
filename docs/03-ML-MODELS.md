# Machine-Learning Models — Design, Algorithms & Preprocessing

**Intelligent Employee Productivity Tracking System (PTA)**

Three models sit on top of the three feature tables, one per time granularity.
Each answers a different question that needs a different kind of evidence, so each
uses a different architecture rather than one model forced to do everything.

| Model | Question | Feature table | Window | Core algorithm |
|---|---|---|---|---|
| **CDM** — Cheat Detection Model | Is this activity genuinely human? | `cheat_detection_features` | 5 sec | **TranAD** — transformer reconstruction anomaly detector |
| **PSM** — Productivity Scoring Model | Will this person be engaged next? | `unified_time_series` | 1 min | **TFT-style** temporal network (variable selection → BiLSTM → attention → GRN → multi-task heads) |
| **GPRM** — Growth Prediction & Recommendation | Is this person developing over time? | `employee_growth_features` | 1 day | **Dilated TCN + N-HiTS** (neural) / **within-subject rule engine** (production) |

Models are trained offline in Python/PyTorch and exported to **ONNX** so
inference runs in-process — no separate model server.

> **Dataset for all results below:** 2026-07-02 → 07-18 (the collection window),
> 4 subjects on 4 machines. Full numbers are in
> [Results](04-RESULTS.md). This document is about *how the models are built*.

---

## 1. Shared preprocessing

All three models sit behind the same preprocessing discipline. This is where most
of the project's hard-won correctness lives — several silent-failure bugs were
found and fixed here (see [Results §4](04-RESULTS.md)).

### 1.1 Per-user chronological split (70 / 15 / 15)

Splitting is **per-user and chronological**: for every subject, the earliest 70%
of their windows are train, the next 15% validation, the last 15% test.

- **Chronological** so validation and test are always *later in time* than
  training — the model is judged on its future, never on interpolation.
- **Per-user** (not one global date cut) because one subject stops producing data
  on 07-12; a global cut would delete that subject from the test set entirely.
  Every subject appears in all three splits.

### 1.2 Heavy-tail-safe scaling

Counter features (bytes sent, context switches, event counts) are extremely
heavy-tailed. A raw `StandardScaler` once emitted a validation z-score of
**10,595 σ** from a single burst, driving validation loss to 3,700 while training
loss was 0.27. The fix, applied to every model:

```
signed log1p   →   standardise (fit on TRAIN only)   →   clip at ±10 σ
```

`log1p` is monotonic and sign-preserving and compresses the tail; the clip caps
any residual outlier. Train and validation now both peak at |z| = 10.

### 1.3 Coverage-rule feature selection

Feature tables are wide, and some columns are never populated or switch on
mid-collection (11 `network_*` columns began reporting partway through). Selecting
features naively let coverage drift change the input width over time.

The selector keeps a column only if it is populated on at least a minimum
fraction of the **training** rows (`min_nonzero_frac`), with an explicit
`keep_sparse` whitelist for rare-but-decisive signals (e.g. the alt+tab / window
switch shortcut counts, which are sparse but are exactly what cheat detection
needs). Selection is fit on train alone, so it can never leak.

### 1.4 Frozen preprocessor (deployment)

The live database grows every day, which shifts coverage percentages and would
drift the selected column set away from the one the model was trained on — until
the input width no longer matched the network. The deployment path therefore
**pickles the exact fitted preprocessor** next to the model and only ever calls
`transform()` on live data (never re-fits selection). Input width is constant
forever, so a model trained on the frozen window scores today's data without
drift.

### 1.5 Application classification (productivity weighting)

`appclass.py` maps ~47 applications to a category and a **work weight**:

| Category | Examples | Work weight |
|---|---|---|
| development / office | `Code.exe`, IDEs, office apps | 1.0 |
| browser / system | browsers, OS shell | 0.5 |
| gaming / social | `cs2.exe`, games, social apps | 0.0 |

This is what stops a game from scoring as productive: the productivity target is
weighted by the in-focus app's work weight, so time in a game contributes 0. The
capture layer's own coarse "browser = productive" flag is deliberately **not**
trusted — productivity is recomputed here from classified behaviour.

---

## 2. CDM — Cheat / Automation Detection

### 2.1 Approach

**Unsupervised anomaly detection.** Nobody labelled a window as "cheating", so
the model only ever sees ordinary recorded behaviour and learns to reconstruct
it; the **reconstruction error is the anomaly score**. Automation (bots, macros,
jigglers, alt-tab thrashing) is identified by the regularity and volume a human
cannot produce, which reconstructs poorly.

### 2.2 Algorithm — TranAD, layer by layer

TranAD (Tuli et al., 2022) is a transformer encoder with **two decoders** and a
**two-phase adversarial reconstruction** objective:

```
input window sequence (30 × 5-sec windows = 2.5 min of context)
      │
   Transformer ENCODER  (d_model 64, 4 heads, 2 layers)
      │  ├─────────────► Decoder 1  → first-phase reconstruction (O1)
      │  └─────────────► Decoder 2  → second-phase reconstruction (O2)
      │
   Phase 1: reconstruct the window (focus score = 0)
   Phase 2: re-condition on the phase-1 error and reconstruct again (adversarial)
      │
   anomaly score = reconstruction error across the two phases
```

| Setting | Value |
|---|---|
| Input table | `cheat_detection_features` (5-sec windows) |
| Sequence | 30 windows = 2.5 min of context, stride 3 in training |
| Architecture | d_model 64 · 4 heads · 2 encoder layers · 2 decoders |
| Optimiser | AdamW, lr 1e-3, weight decay 0.01, grad-clip 1.0 |
| Features (v3 frozen) | 200 (coverage-rule selected) |

**Why a transformer reconstruction model?** Attention over a 30-window sequence
captures the short-range *rhythm* of input, which is the exact signal that
separates a person from an automation tool — a purely per-window classifier would
have no access to timing regularity across windows.

### 2.3 Evaluation by synthetic injection

Because there are no real cheat labels, the detector is scored against
**synthetic automation episodes** injected into held-out test windows — each
editing the real feature columns of the behaviour it imitates, over contiguous
2-minute episodes inside genuinely active stretches:

| Vector | Signature imposed |
|---|---|
| `bot_rhythm` | sustained input at machine-perfect periodicity |
| `replay_macro` | one action replayed — diversity / entropy destroyed |
| `burst_injection` | superhuman volume, still machine-regular |
| `focus_thrash` | rapid alt-tab / window-switch cycling |
| `app_cycling` | cycling apps to fake activity / evade idle detection |

The last two were added for the demo requirement that *faking* productivity —
alt-tabbing, jigglers, app cycling — must be caught, not just classic bots. The
detection threshold is chosen on **validation only** and applied unchanged to
test.

### 2.4 Calibration

A live risk band ("safe / suspicious / high-risk") must be stable window-to-window
— an early bug ranked each window *inside its own batch*, so 61% got flagged. The
deployment model instead uses **fixed thresholds** taken once from the model's own
score distribution on training data (97.5th / 99.5th percentiles), stored beside
the weights.

### 2.5 Result

Test **ROC-AUC 0.9558** (train 0.955 / val 0.9549 → gap −0.0008, i.e. genuine
generalisation), PR-AUC 0.6377 at 9.9% prevalence (**6.5× lift**), consistent
across all five vectors (ROC 0.92–0.98). Precision 0.57 at the F1-optimal
threshold means it is a **triage signal**, not a verdict.

---

## 3. PSM — Productivity / Engagement Scoring

### 3.1 The label problem (and why the design is careful)

The dataset has **no human productivity labels**, and the obvious heuristic label
is circular (every input to it is also a model feature) and partly empty. PSM
therefore forecasts a **falsifiable proxy**:

> Given the past 60 minutes of behaviour, will this person be **more engaged than
> their own recent usual** over the next 5 minutes?

Key label properties, each fixing a specific failure:

- **Forward-looking.** The target is computed only from windows *after* the input
  sequence ends, so it can never leak into the features.
- **App-weighted (v4).** Engagement is `input events × work_weight` (see §1.5), so
  gaming does not count as productive.
- **Per-user median threshold (v4).** The binary label is relative to *each user's
  own* median, not a global one. An earlier version let a model score ROC-AUC
  0.72 from user identity alone; per-user centering removes that shortcut.

### 3.2 Algorithm — TFT-style network, layer by layer

A Temporal-Fusion-Transformer-style network processes 60 one-minute windows:

```
60 × 1-min windows, features arranged in 10 behavioural groups
      │
   Group Variable-Selection Network  (learns which feature groups matter)
      │
   Bidirectional LSTM  (local sequence encoding)
      │
   Gated skip connection  +  static enrichment
      │
   Multi-head temporal attention  (long-range context)
      │
   Gated Residual Network (GRN)
      │
   Multi-task heads:
     • engagement (BCE)               ← the scored output
     • P10 / P50 / P90 quantiles (pinball loss)   ← uncertainty band
     • per-category / behavioural-state / role-intent heads
```

Feature groups (93 features in v4): `mouse` 17, `system` 13, `other` 12, `app`
12, `keyboard` 10, `focus` 8, `activity` 8, `temporal` 6, `composite` 4,
`security` 1.

| Setting (v4) | Value |
|---|---|
| Sequence | 60 × 1-min windows (1 hour of context) |
| Architecture | d_model 48, dropout 0.3, weight decay 0.05, lr 5e-4 |
| Loss | BCE (engagement) + pinball (P10/P50/P90) |
| `persistence_skip` | a learned skip that adds the current engagement level directly to the logit, so the network only has to model the *residual* over persistence |
| Ensemble | 5 seeds `[7, 1, 13, 29, 3]`, averaged |

**Why this architecture?** Variable selection makes *which behavioural group
matters* interpretable; the BiLSTM captures local minute-to-minute dynamics;
temporal attention supplies longer context; the quantile heads give an
uncertainty band rather than a bare point estimate. The **5-seed ensemble** was
adopted because a single seed's test AUC varied by ±0.022 on this small dataset —
one seed alone was not a trustworthy report.

### 3.3 Result

Test **ROC-AUC 0.8635** against a persistence baseline of 0.8563 — PSM v4 beats
persistence on **all three splits** (train 0.903 vs 0.792, val 0.911 vs 0.869,
test 0.863 vs 0.856), with a modest train→test gap of 0.040. Honest framing: the
target is strongly autocorrelated, so persistence is already strong; PSM's added
value is the per-category breakdown, behavioural state and uncertainty bands that
persistence cannot provide, plus the app-weighting that keeps gaming out of the
score.

---

## 4. GPRM — Growth Prediction, and why a rule engine ships

### 4.1 Neural model

**Dilated TCN blocks + an N-HiTS forecasting head** over daily growth features.
Dilated convolutions give a wide receptive field over days cheaply; N-HiTS is a
hierarchical-interpolation forecaster designed for long-horizon time series. The
architecture targets a 30-day context.

### 4.2 Why it is not the production path

With only **24 training subject-days**, the neural model **memorised**: MAE 0.212
on days it was fitted to but 0.377 on unseen days, *worse* than a constant-mean
baseline (0.365). Validation error rising while training error falls is the
textbook signature of insufficient data, not a bug.

### 4.3 Production growth: within-subject rule engine

Rather than serve an unvalidated number, growth ships as a **transparent rule
engine**. It compares a person only to **their own earlier days**, using a robust
effect size (median shift ÷ baseline interquartile range) across four weighted
dimensions — skill acquisition, focus quality, consistency, work quality.
Engagement (hours worked) is measured and reported but carries **zero weight**, so
the score cannot be improved by simply working longer. Every strength, weakness
and recommendation stores the signal name, the observed value and the baseline it
was judged against, so any conclusion can be audited.

Both engines write to `growth_predictions`, keyed on `(user, date,
model_version)`, so the neural model can be promoted without a schema change once
~60 days of history exist.

---

## 5. Serving

- Models exported to **ONNX**; inference runs in-process in the scoring job.
- Predictions are written to the versioned prediction tables and served read-only
  by the API (see [Backend & Database](02-BACKEND-API-AND-DATABASE.md)).
- Every prediction row carries its `model_version`, so `v3`/`v4` results sit
  beside earlier ones and a rollback is a version filter.

---

## 6. What the models do **not** claim

- **No absolute productivity accuracy** — there are no human productivity labels;
  PSM forecasts a behavioural proxy.
- **CDM metrics come from synthetic anomalies** — they measure sensitivity to
  automation-like deviation, not real-world cheat prevalence.
- **No growth forecasting claim** — 24 training days cannot fit a temporal
  forecaster; the rule engine is transparent, not validated against outcomes.
- **Small cohort** — 4 subjects, 17 days. Normal behaviour is learned from few
  people; the planned 2-month collection is the single change most likely to
  strengthen every conclusion here.
