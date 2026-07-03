# Meeting Brief — Updates for Liuyi

*One-pager for the upcoming supervisor meeting. Prepared by Brian. Backing detail in the other `docs/` files.*

---

## 1. I understood the pivot from your answers

- The "two CSVs" = **Ben's watch data (sleep + light)**, a different project — not our data. Ben's code is a *template*, not built for us. ✔ Won't chase that data.
- **Task 3 = understand the TSFM input format**, done against the **ICML'25 `representations-in-tsfms` repo** (Chronos-specific, ships data, you reproduced it). Data availability is not a blocker. ✔
- Scope locked: **1 channel (CGM only), single-target, three separate models (Grids / Symbols / Prices), raw-data/TSFM arm only** (Mack owns feature-engineering). ✔

## 2. What I did

- **Studied all three codebases** (Ben's `mod-actigraphy-advanced`, the ICML `representations-in-tsfms`, and your `diabetes-fitbit`) and extracted the exact Chronos input recipe:
  `raw glucose window → BaseChronosPipeline.embed() → (B, patches+1, d_model) → mean-pool over tokens → (B, d_model)`. No normalization needed (Chronos mean-scales internally); NaN-tolerant (good for gappy CGM).
- **Built a runnable pipeline** (`cgm_tsfm/`) with the **real data schema** (`subid`, `session`, `Glucose_Before_Test`, the 3 scores), a synthetic generator so it runs today, and an embedding cache. It's **drop-in for the real CSVs**.
- **Arm A** (frozen Chronos embeddings + Ridge/SVR, nested **GroupKFold by `subid`**, RMSE/MAE/R²) — mirrors your `diabetes-fitbit` protocol exactly, so results are directly comparable to Mack's.
- **Arm B (Task 4)** — adapted Ben's `FoundationModelClassifier`→ single-channel `FoundationModelRegressor` and his `FinetuneLightningModule`→ regression `CGMRegressionModule` (replaced the `/365` gestational-age hack with a general per-target z-score; replaced the fragile task-name string check). Smoke-tested: the head learns.
- **Ran it end-to-end with real `chronos-bolt-small`**: 938 windows → 512-d embeddings → grouped nested CV on all 3 targets. Works.

## 3. What I found (calibrating expectations honestly)

- I saw in your `next_steps` notebook that **Mack's feature-engineering arm has no predictive power** (negative R², chance AUROC), and that **session-CV beats leave-one-patient-out → the signal is mostly subject-baseline**, not generalizable glucose→cognition.
- My pipeline **reproduces that exact diagnostic** on synthetic data (group-CV R²≈0, session-CV R²>0), so we can trust it to tell "no signal" apart from "a bug."
- Practical lesson from the real Chronos run: **plain linear regression on 512-d embeddings overfits badly** (R²≈−4); Ridge/SVR + grouped CV (both built in) are essential.
- ⚠️ My synthetic numbers are **not** evidence about Chronos-vs-features — the synthetic signal was defined via summary stats, which favors hand-crafted features. **The real comparison needs the real data.**

## 4. What I need from you (decisions)

1. **The data.** Can I get `Cohort1_scores_merged_with_glucose.csv` + `Cohort2_scores_with_glucose.csv` (they're gitignored)? Then I can run the real head-to-head immediately.
2. **The window / raw stream.** `Glucose_Before_Test` is pre-clipped upstream — is the **raw continuous CGM stream** available, and what **lookback** should I standardize (the K01 hypothesis targets ~2 h pre-test)?
3. **Prices + score meaning.** Confirm `prices_cognitive_score` is inverted (I negate it). What do Grids/Symbols/Prices actually measure (units)?
4. **Success criteria.** Given Mack's null result, is the goal to *confirm/refute with learned representations under identical CV*, or to chase a specific hypothesis (hypoglycemia windows / within-subject effects)?
5. **Encoder choice.** OK to standardize on **Chronos-Bolt** (newer/faster) vs the T5 the ICML paper studied? Which size?
6. **compute2 (Task 1)** — when do you want me to pick this up? (Larger checkpoints will need it.)

## 5. Immediate next step (the moment I get the data)

`python -m cgm_tsfm.run_headtohead --encoder chronos --real` → the first real **Chronos-embeddings vs Mack's-43-features** table, per target, under identical subject-grouped nested CV, with a per-target verdict. (Mack's 43 features are ported into `cgm_tsfm/handcrafted.py` and verified numerically identical to his originals, so the only thing that differs in the comparison is the representation.)
