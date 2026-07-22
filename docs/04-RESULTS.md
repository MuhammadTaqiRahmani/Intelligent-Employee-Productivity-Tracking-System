# Results

**Intelligent Employee Productivity Tracking System (PTA)**

Consolidated outcome of the modelling work. This states what was built, what the
numbers are, and — just as important — what they do and do **not** support. How
the models are built is in [ML Models](03-ML-MODELS.md).

**Dataset:** 2026-07-02 → 07-18 (17 days) · 4 subjects · 4 machines
**Shipped versions:** CDM `v3` · PSM `v4` · GPRM neural `v1` + rule engine
**Evaluation window:** frozen at `window_start < 2026-07-19` (the training
collection), so every number below comes from one run on stable data and the
figures agree with each other.

---

## 1. Headline results

| Model | Task | Metric | **Test** | Baseline | Verdict |
|---|---|---|---|---|---|
| **CDM v3** | Cheat / automation detection | ROC-AUC ↑ | **0.9558** | 0.099 (prevalence) | **Works** |
| **PSM v4** | Engagement forecast (next 5 min) | ROC-AUC ↑ | **0.8635** | 0.8563 (persistence) | **Works** |
| **GPRM** | Next-day growth | MAE ↓ | 0.3772 | **0.3649** (train mean) | No demonstrated skill → rule engine ships |

Two of three models genuinely generalise. That conclusion rests on the
generalisation analysis below, not on any single score.

---

## 2. Generalisation — the decisive evidence

The same metric on every split. A small train→test gap is what real learning
looks like; a large one is memorisation.

| Model | Metric | Train | Validation | **Test** | Gap (train → test) |
|---|---|---|---|---|---|
| **CDM v3** | ROC-AUC ↑ | 0.9550 | 0.9549 | **0.9558** | **−0.0008** |
| **PSM v4** | ROC-AUC ↑ | 0.9033 | 0.9107 | **0.8635** | **+0.0399** |
| GPRM | MAE ↓ | 0.2119 | 0.2562 | 0.3772 | +0.1653 |

### Against each model's baseline, per split

| Model | Split | Model | Baseline (persistence / mean) | Winner |
|---|---|---|---|---|
| PSM v4 | train | 0.9033 | 0.7919 | **model** |
| PSM v4 | val | 0.9107 | 0.8693 | **model** |
| PSM v4 | test | 0.8635 | 0.8563 | **model** |
| GPRM | train | 0.2119 (MAE) | 0.2612 | model |
| GPRM | test | 0.3772 (MAE) | 0.3649 | **baseline** |

CDM v3 and PSM v4 beat their baselines on **all three splits**. GPRM beats its
baseline only on data it was fitted to — the signature of memorisation, which is
why the transparent rule engine is the production growth path.

---

## 3. CDM v3 in detail — the model that works

Frozen evaluation, test set 13,600 windows, 9.9% injected (1,344 windows).

| Metric | Test value |
|---|---|
| ROC-AUC | **0.9558** |
| PR-AUC | **0.6377** (random = 0.099 → **6.5× lift**) |
| Recall | 0.8810 |
| Precision | 0.5676 |
| F1 | 0.6904 |

Splits: train 63,458 · val 13,599 · test 13,600 · 200 features.

### Per cheat vector — consistent, not keyed to one artefact

| Vector | Windows | ROC-AUC | Recall |
|---|---|---|---|
| `bot_rhythm` | 288 | 0.9629 | 0.9236 |
| `replay_macro` | 264 | 0.9635 | 0.9205 |
| `burst_injection` | 264 | 0.9808 | 0.9773 |
| `focus_thrash` (alt-tab) | 264 | 0.9465 | 0.8485 |
| `app_cycling` | 264 | 0.9245 | 0.7311 |

The last two vectors were added specifically so that *faking* productivity —
rapid alt-tabbing, jigglers, cycling apps to look busy — is caught, not just
classic bots. All five score ROC 0.92–0.98.

**Caveat:** precision 0.57 at the F1-optimal threshold means roughly four in ten
flagged windows are false positives. The score is a **triage signal for review,
not a verdict**; raising the threshold trades recall for precision.

---

## 4. Engineering — defects found and corrected by measurement

Each of these would have silently corrupted results. Every one was noticed in a
metric, traced to a root cause, fixed, and the fix verified.

| Defect | Effect | Fix |
|---|---|---|
| Heavy-tailed scaling collapse | val loss 3,700 vs train 0.27; one feature at 10,595 σ | signed `log1p` → scale → clip ±10 σ |
| Ill-posed anomaly injection | ROC-AUC 0.52 (chance) — "perfect regularity" matched ordinary sparse windows | volume *and* regularity together, in 2-minute episodes |
| Circular productivity label | would report a high score while proving nothing | forward-looking, app-weighted, per-user-median target |
| Identity shortcut in PSM | ROC-AUC 0.72 from user identity alone | per-user median label (centre on each user's own usual) |
| Calibration batch-rank bug | 61% of windows flagged | fixed thresholds from the training score distribution |
| Coverage drift (11 `network_*` columns) | input width changed mid-collection | `min_nonzero_frac` coverage rule + frozen preprocessor |
| Gaming scored as productive | a game could raise the productivity score | app classification + work-weighting (gaming weight 0.0) |
| Single-seed variance (±0.022) | one seed was not a trustworthy report | 5-seed PSM ensemble |
| Duplicate predictions on re-run | rows doubled | unique constraint + `ON CONFLICT DO NOTHING` |

The first two moved CDM from **0.52 (chance) to 0.96**.

---

## 5. What was achieved

**Engineering**

- Reproducible pipeline from three feature tables → trained models → stored
  predictions, in scripted phases.
- Data foundation verified by automated checks per table (chronological splits,
  every subject in every split, scaler fit on train only, no leakage, no NaNs,
  correct sequence shapes).
- Per-user chronological splitting; coverage-rule feature selection; frozen
  deployment preprocessor so a growing DB never drifts the input width.

**Modelling**

- Three architectures implemented and trained: **TranAD** (transformer
  reconstruction), a **TFT-style** temporal network, a **dilated-TCN + N-HiTS**
  forecaster — each evaluated against an explicit baseline so every number is
  interpretable.
- CDM and PSM promoted through measured, versioned improvements (`v1 → v3`,
  `v1 → v4`) rather than asserted.

**Serving**

- Models exported to ONNX and running on the server; predictions written to
  versioned tables; live dashboard at `pta.emotionflow.site` with model-results,
  live-session and methodology pages.

---

## 6. What the results do **not** support

- **No absolute productivity accuracy.** No human productivity labels exist. PSM
  v4 forecasts a behavioural proxy — whether a person will be more engaged than
  their own recent usual — and beats persistence on every split. That is
  engagement forecasting, not productivity as a manager would judge it.
- **CDM's metrics come from synthetic anomalies.** They measure sensitivity to
  automation-like deviation, not real-world cheat prevalence, and cannot cover a
  strategy nobody anticipated.
- **No growth forecasting claim.** 24 training days across 4 subjects cannot fit a
  temporal forecaster; GPRM neural is a pipeline-integration result only, and the
  rule engine is transparent rather than validated against outcomes.
- **Four subjects, 17 days.** Normal behaviour is learned from a small cohort.

---

## 7. What would change these conclusions

The failure modes of GPRM (and PSM's remaining gap) are **data-volume
signatures** — validation error rising while training error falls. The planned
**2-month collection** (~60 days per subject) directly addresses this and would
make GPRM's intended 30-day context usable for the first time. Retraining as a
new version needs no schema change: the prediction tables are keyed on
`(user, window/date, model_version)`.

Also outstanding from the collection side: the process and focus capture services
crash intermittently (17–44% of active hours), the largest source of missing
data.
