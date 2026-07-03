# Pipeline Design — CGM → Chronos → Cognitive Score

*Technical reference for the `cgm_tsfm/` package. Cites the three reference codebases so every design choice is traceable. Absolute repo paths are in `README.md`.*

---

## 1. The prediction problem (data contract)

- **One sample = one cognitive-test session.** Each session has a variable-length array of CGM readings taken *before* the test (`Glucose_Before_Test`), at **5-minute cadence** (Dexcom G6), and up to three continuous cognitive scores.
- **Targets (3, modeled separately):** `grids_cognitive_score`, `symbols_cognitive_score`, `prices_cognitive_score` — the ARC/PARC **Grids / Symbols / Prices** subtests. (`diabetes-fitbit/analysis/glucose_cognitive_ml/config.py:13-17`.)
  - ⚠️ **Prices is inverted in the raw data (higher = worse).** We negate it so "higher = better" for all three (`config.py: PRICES_IS_INVERTED`). Flagged for confirmation in `04_OPEN_QUESTIONS.md`.
- **Grouping key = `subid`.** A subject contributes ~32–67 sessions. All CV must group by subject to avoid leakage.
- **Real dataset size:** 980 sessions / 20 subjects → 913 / 19 after dropping empty-glucose sessions (from the `diabetes-fitbit` notebook outputs). Files (`Cohort{1,2}_...glucose.csv`) are gitignored and not yet in hand — see open questions.

Our `data.CGMDataset` holds exactly this: `windows` (list of 1-D arrays), `subjects`, `sessions`, `targets` (dict). `data.load_real_data()` mirrors `diabetes-fitbit`'s loader (same column harmonization + the `eval`-based `parse_glucose` that tolerates Cohort2's literal `nan` tokens), so it runs unchanged on the real CSVs. `data.generate_synthetic_data()` emits the identical schema for offline development.

---

## 2. Windowing

Chronos ingests variable-length series natively (it left-pads internally), so windowing is optional. `WindowConfig` exposes:
- `min_readings=3` — drop too-short sessions (matches `diabetes-fitbit/features.py`).
- `max_readings` — keep only the most-recent N (e.g. **30 readings = 2.5 h**, the K01's hypothesis window; `None` = keep the full pre-clipped array).

> **Open issue (see `04_OPEN_QUESTIONS.md`):** the real `Glucose_Before_Test` is *already clipped upstream* to some undocumented lookback, and the *raw continuous* CGM stream isn't in the `diabetes-fitbit` repo. To standardize the lookback (or use a longer context) we likely need the upstream merge script / raw Dexcom files.

---

## 3. The frozen-encoder recipe (Chronos)  — Task 3's core deliverable

Verified against the ICML repo (`representations-in-tsfms/efficiency/tsfm_similarity/hooked_models/hooked_chronos.py`) and the installed `chronos==2.3.0` API:

```
windows: list of 1-D float tensors (raw mg/dL, variable length, NaN allowed)
   │   BaseChronosPipeline.embed(context)          # chronos 2.3.0
   ▼
embeddings: (B, num_patches+1, d_model)            # +1 = [REG]/EOS token
   │   mean over the patch/token axis  (ICML: np.mean(rep, axis=1))
   ▼
pooled: (B, d_model)                               # d_model=512 for bolt-small
```

Design facts baked into `encoders.ChronosEncoder`:
- **No external normalization.** Chronos mean-scales each series internally; feed raw mg/dL. (The ICML wrapper does no user-side normalization either.)
- **NaN-tolerant.** Missing readings map to a mask token — no gap-filling needed. A real advantage for gappy CGM.
- **`BaseChronosPipeline` auto-selects Bolt vs T5** from the checkpoint name, so one code path serves both families. (Ben used Chronos-**Bolt** via `.encode()`; the ICML paper studied Chronos-**T5** via `.embed()`. We default to `amazon/chronos-bolt-small`; swappable.)
- **Embeddings are cached to `.npy`** keyed by (model, pooling, data hash) — the forward pass is the expensive step; extract once, reuse across all 3 targets and every regressor (mirrors the ICML `.npy` caching).
- A **`mock` encoder** (summary statistics) lets the whole pipeline run with no download.

**Empirically confirmed working:** on synthetic data, `chronos-bolt-small` produced `(938, 512)` embeddings on CPU and the downstream CV ran clean.

---

## 4. Arm A — frozen embeddings + classical regressor

`regression.run_arm_a()`. This is the ICML SVM-classification experiment
(`.../experiments/classification/classification_svm_experiment.py`) converted to regression, and it deliberately mirrors `diabetes-fitbit`'s protocol so the two arms are **directly comparable — only the feature set differs** (512-d Chronos embedding vs 43 hand-crafted glycemic features).

`run_arm_a` is feature-agnostic (it takes any `(N, d)` matrix), which is what makes the head-to-head trivial: `handcrafted.extract_handcrafted_features()` is a **faithful port of Mack's 43 features** (`diabetes-fitbit/.../features.py`, verified numerically identical — max abs diff 0.0 over 200 random windows), and `run_headtohead.py` pushes both the Chronos embeddings and the 43 features through the *same* `run_arm_a` and prints a per-target verdict. One command = the K01's "raw-data vs feature-engineering" comparison, made fair.

- **Models:** Ridge (α grid), SVR (RBF, C/γ grid), Linear — plus **XGBoost if installed** — and always a **`DummyRegressor(mean)` baseline** (R² is measured relative to it).
- **Evaluation:** **nested CV** — outer 5-fold `GroupKFold` on `subid` (unbiased estimate), inner 3-fold `GroupKFold` `GridSearchCV` (tuning, `neg_mean_squared_error`). `StandardScaler` fit on train folds only. A hard `assert` guards against subject leakage on every fold (mirrors `diabetes-fitbit/regression.py:33-35`).
- **Metrics:** RMSE, MAE, R² (mean ± std across outer folds) — same as `diabetes-fitbit`.
- **Built-in diagnostic:** `cv_scheme="group"` vs `"session"`. If group-CV R²≈0 but session-CV R²>0, the signal is subject-baseline, not generalizable — the exact check Liuyi's team used.
- **Optional PCA** (`pca_components`, fit *per fold* inside the pipeline → leakage-safe) reduces the high-dim embeddings that make linear models ill-conditioned. Sweepable (`run_sweep --kind pca`) and usable in the head-to-head (`run_headtohead --pca N`). On synthetic data PCA≈16–32 improves grouped-CV R² over the full 512-d and eliminates the `LinAlgWarning`.
- **Optional within-subject target normalization** (`target_norm="center"|"zscore"`, via `data.within_subject_normalize`) subtracts each subject's own mean score, so the model predicts *deviation from personal baseline* — a direct test of the "signal is subject-baseline" hypothesis. Toggle: `run_headtohead --target-norm center`, `run_sweep --kind targetnorm`. ⚠️ Oracle centering (uses a subject's own sessions): answers "is there a within-subject signal?", not "can we predict a new subject" — same framing as diabetes-fitbit's `next_steps`. On synthetic data raw R²=−0.38 → centered ≈0 (baseline stripped); with an injected within-subject signal, centered R² rises to ≈0.93, confirming the pipeline detects within-subject effects when present.

**Validated behavior (synthetic data, real Chronos-bolt-small):**
- Ridge/SVR sit near the mean-baseline; **plain Linear on raw 512-d embeddings catastrophically overfits** (R² ≈ −3 to −5) → *regularization or dimensionality reduction on TSFM embeddings is mandatory.*
- ⚠️ **Do not read the synthetic numbers as Chronos-vs-features evidence:** the synthetic "true signal" is by construction a function of summary statistics, which favors the hand-crafted/mock features. The honest Chronos-vs-features comparison must be run on the **real** data.

---

## 5. Arm B — frozen encoder + trainable head (adapting Ben's code) — Task 4

`ben_adapter/`. This is the literal "adapt/generalize Ben's code to one channel of CGM" task. It reuses Ben's *frozen-encoder + MLP-head* structure and drops the parts that don't apply to a single window → single score.

`FoundationModelRegressor` (`ben_adapter/model.py`) vs Ben's `FoundationModelClassifier` (`mod-actigraphy-advanced/src/models/pretrained_models.py:270-704`):

| Ben's classifier | Our regressor | Why |
|---|---|---|
| `num_channels = 2` (activity+light), hardcoded (`:340`) | **1 channel** (glucose) | Liuyi: single-channel CGM |
| `forward(activity, light, activity_mask, light_mask, day_mask)` (`:595`) | **`forward(glucose, glucose_mask)`** | one signal, one mask |
| across-days **causal transformer** over `(B, D, T)` (`:509-593`) | **dropped** | each sample is ONE pre-test window (D=1) → no day sequence to attend over |
| channel-combine (`concat_project`/`add`, `:401-422`) | **dropped** | only one channel |
| head `Linear(hidden, num_classes)` (`:384-390`) | head `Linear(hidden, num_targets=1)` | regression scalar |
| Bolt `.encode()` low-level wrapper | `BaseChronosPipeline.embed()` + mean-pool | version-robust; same recipe as Arm A |

`CGMRegressionModule` (`ben_adapter/lightning_module.py`) vs Ben's `FinetuneLightningModule` (`mod-actigraphy-advanced/src/models/modules.py:330-795`):

| Ben's module | Our module | Why |
|---|---|---|
| `if task=="ptb_classification": classification else: regression` (`:354-359`) | **regression only, explicit flag** | Ben's string check made *any* other task silently regress — fragile |
| `labels/365.0` … `preds*365.0` (`:588-591`) | **z-score with train-set target mean/std** (`register_buffer`) | `/365` is gestational-age-specific; a general per-target normalizer works for any cognitive score and any scale (incl. Prices' larger range) |
| MSE loss + MSE/RMSE/R² metrics (`:391,432-448`) | **MSE loss + MSE/RMSE/R²/MAE** | kept Ben's regression metrics, added MAE |
| encoder trainable? (frozen inside model) | encoder **frozen**; only head params in optimizer | zero-shot embeddings |

**Grouped-CV training is wired** (`ben_adapter/datamodule.py` + `train.py`): the frozen encoder is run once and cached, then `run_arm_b()` trains one MLP head per **`GroupKFold(5)`** fold (same deterministic folds as Arm A) with a grouped `GroupShuffleSplit` val split for early stopping — a fixed architecture + early stopping in place of Arm A's inner grid search. It returns the same result shape as `run_arm_a`, so Arm B joins the head-to-head as a third contender (`run_headtohead --with-arm-b`).

**Validated:** `smoke_test.py` (forward → `(B,1)`, head learns) + a full 3-target × 5-fold training run on cached Chronos embeddings. Full training on the **real** CSVs is the remaining step (blocked on data access).

---

## 6. Why two arms

Arm A (frozen embeddings + Ridge/SVR) is the **fast, directly-comparable-to-Mack** path and the literal ICML pattern — the quickest route to a real head-to-head "raw representations vs hand-crafted features" number under identical grouped CV. Arm B (trainable head) is the **flexible** path and the explicit Ben-adaptation deliverable; it also opens the door to later fine-tuning (LoRA) or a multi-window/temporal extension. Both share the same frozen Chronos encoder and the same windows, so they are two heads on one backbone.
