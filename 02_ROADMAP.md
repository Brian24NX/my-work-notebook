# Roadmap & Status

*Living plan. Statuses reflect work completed after Liuyi's answers. The four numbered tasks are Liuyi's original assignment (`liuyi_task_assignment.png`), reinterpreted per his clarifications.*

---

## Status of the four assigned tasks

| # | Task (as reinterpreted) | Status | Evidence |
|---|---|---|---|
| 1 | Dev environment on **compute2** | ⏸ **Deferred** (per earlier instruction) | Flagged for scheduling — see open questions Q6. |
| 2 | Understand Ben's codebase + TSFM input formats | ✅ **Done** | `PROGRESS_NOTES.md` + `01_PIPELINE_DESIGN.md §3,§5`. Confirmed Ben's `ChronosEncoder` is channel-agnostic and the 2-channel hardcode lives only in `FoundationModelClassifier`. |
| 3 | Understand the TSFM raw input format & how to convert to it | ✅ **Done** (via the ICML repo, per Liuyi) | Studied `representations-in-tsfms`; distilled the exact `(B,1,L)→embed→mean-pool→(B,d)` recipe and **implemented + ran it** (`encoders.py`, real `chronos-bolt-small`, 512-d embeddings). |
| 4 | Adapt Ben's code for regression output (single channel) | ✅ **Prototype done** | `ben_adapter/` — single-channel `FoundationModelRegressor` + regression `CGMRegressionModule`, smoke-tested. Full training pending real data. |

**Beyond the four tasks**, I also built the whole runnable pipeline (both arms, synthetic-data harness, grouped-CV evaluation, embedding cache) so it's drop-in for the real CSVs.

---

## Milestones

### M0 — Scaffold & validate offline ✅ (done)
- [x] Self-contained `cgm_tsfm/` package, real-schema-compatible loader + synthetic generator.
- [x] Chronos embedding extractor (Bolt/T5, mean-pool, cache) + `mock` encoder.
- [x] Arm A: nested GroupKFold regression (Ridge/SVR/Linear/XGB), RMSE/MAE/R², group-vs-session diagnostic.
- [x] Arm B: single-channel regressor + Lightning module (Ben-adaptation), smoke-tested.
- [x] Arm B **grouped-CV training** wired (`ben_adapter/datamodule.py` + `train.py`): shared `GroupKFold(5)` folds, grouped val early-stopping; validated on a full 3-target × 5-fold run.
- [x] Real `chronos-bolt-small` run end-to-end on synthetic data.
- [x] Mack's 43 hand-crafted features ported (`handcrafted.py`, verified numerically identical to his originals) + one-command **head-to-head** runner (`run_headtohead.py`, 2- or 3-way with `--with-arm-b`).

### M1 — Run on the real data ⛔ (blocked ONLY on data access — tooling is built)
- [ ] Obtain `Cohort1_scores_merged_with_glucose.csv` + `Cohort2_scores_with_glucose.csv` (or run where they live). → **Ask Liuyi (Q1).**
- [ ] `python -m cgm_tsfm.run_headtohead --encoder chronos --real` → the **Chronos-embeddings vs Mack's-43-features** table for all 3 targets under identical grouped CV (the head-to-head runner already exists and is validated on synthetic data).
- [ ] Also run `--cv session` for the subject-baseline diagnostic, and `run_demo --real` for the full per-model breakdown.
- [ ] Report R²/RMSE/MAE per target with the mean-baseline reference; interpret via the subject-baseline diagnostic.

### M2 — Strengthen the raw-data arm (only if M1 shows life, or to characterize the ceiling)
- [x] **Dimensionality reduction wired** (PCA before Ridge/SVR, fit per-fold in the pipeline — leakage-safe). Exposed as a sweep axis (`--kind pca`) and a head-to-head flag (`run_headtohead --pca N`). On synthetic data PCA≈16–32 improves grouped-CV R² over full 512-d and removes the ill-conditioning (`LinAlgWarning`). Real-data value TBD.
- [x] **Sweep harness built + validated** (`run_sweep.py`): checkpoint (bolt tiny/mini/small/base — Bolt has no "large" — + a T5 variant), window (2 h / 2.5 h / 3 h / full), pooling (mean/last), and **PCA** (none/8/16/32/64/128). Ran clean on synthetic data (numbers not meaningful there); ready for `--real`.
- [ ] Tune Arm B (head width/depth, dropout, epochs); the grouped-CV training harness is built — this is now a hyperparameter sweep, not new plumbing.
- [x] **Within-subject target normalization** wired as a toggle (`--target-norm center|zscore`) across Arm A, Arm B, the head-to-head, and the sweep (`--kind targetnorm`). Directly tests the subject-baseline hypothesis: on synthetic data raw R²=−0.38 → centered ≈0 (baseline removed), matching diabetes-fitbit. Validated that it *detects* a within-subject signal when one is injected (synthetic `within_subject_signal` knob → centered R²≈0.93). Uses oracle per-subject centering (documented) — a characterization tool, not a new-subject predictor.

### M3 — Rigor & write-up
- [ ] Fixed subject-level CV folds shared across both arms and all targets (reproducibility).
- [ ] Optional: LoRA fine-tuning of the encoder (vs frozen) — only if frozen shows promise.
- [ ] Results doc + figures; fold into the K01's "raw-data vs feature-engineering" comparison.

---

## Key decision points (need Liuyi's input — details in `04_OPEN_QUESTIONS.md`)

1. **Data access & the window.** Get the merged CSVs; define the pre-test lookback; is the *raw* CGM stream available (not just the pre-clipped array)?
2. **Prices inversion + score semantics.** Confirm negation; what do Grids/Symbols/Prices actually measure (units)?
3. **Success criteria given Mack's null results.** Is the goal to confirm/refute with learned representations, or to chase a specific hypothesis (e.g. hypoglycemia windows, within-subject effects)?
4. **Encoder family/size.** Standardize on Chronos-Bolt vs T5, and which checkpoint size.

---

## Risks / honest caveats

- **The target may have little generalizable signal.** Mack's arm found none; a frozen general-purpose TSFM won't create signal. Frame deliverables as a *fair comparison and characterization*, not a promised accuracy win.
- **High-dim embeddings + ~19 subjects → overfitting.** Grouped CV + regularization/PCA are non-negotiable (already built in; Linear-model blow-up demonstrates the failure mode).
- **Synthetic ≠ real.** The synthetic harness validates plumbing and the CV diagnostic only; it is not evidence about Chronos-vs-features on real data.
- **No GPU assumed.** Everything runs on CPU with `bolt-small`; larger checkpoints will want compute2 (ties back to Task 1).
