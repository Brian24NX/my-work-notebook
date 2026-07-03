# Project Overview & Guidebook — CGM → Cognition via Time-Series Foundation Models

*Brian Zhou, supervised by Liuyi. Written after receiving Liuyi's answers to the initial questions (`Questions_LiuyiAnswer.docx`). This is the "read me first" narrative; technical details are in `01_PIPELINE_DESIGN.md`, the plan is in `02_ROADMAP.md`, and the meeting one-pager is `03_MEETING_BRIEF.md`.*

---

## 1. What the project is (one paragraph)

We want to predict a youth-with-T1D's **real-time cognitive performance** on a short mobile test (the ARC/PARC app's **Grids**, **Symbols**, and **Prices** subtests) from the **continuous-glucose-monitor (CGM) readings recorded in the window just before that test**. The distinctive method: instead of hand-engineering glucose features, we feed the *raw* glucose window through a **frozen pretrained time-series foundation model (TSFM), Amazon Chronos**, and learn a small regressor on top. This is the "raw-data learning" arm that the K01 grant proposes to compare against classical "feature engineering."

---

## 2. The pivot — how Liuyi's answers changed the plan

My original notes (`PROGRESS_NOTES.md`, 2026-07-01) assumed we'd get two CGM CSVs and port Ben's actigraphy model. **Liuyi's answers redirected this substantially.** The corrected understanding:

| Topic | What I originally assumed | What Liuyi clarified |
|---|---|---|
| The "two CSV files" in the task | Our CGM + cognitive data | **Ben's watch data (sleep + light intensity)** — a *different* project (his early paper). We don't have them; Liuyi's IRB for Ben's data is pending. |
| Do I need that data for Task 3? | Yes, blocked without it | **No.** Task 3 = *understand the TSFM input format*. Data availability is not required. |
| How to do Task 3 concretely | Wait for CGM files | **Study & run the ICML'25 paper's repo** (`representations-in-tsfms`) — it's Chronos-specific, ships runnable data, and Liuyi reproduced it. "If we understand its pipeline, we know how to apply TSFM to CGM." |
| Channels | 2 (glucose + Fitbit) | **1 channel only** — just CGM. "Forget Fitbit for now. Adapt/generalize Ben's code to accept one channel of CGM." |
| Regression targets | Maybe joint multi-target | **Single-target. Three separate pipelines/models** — one each for Grids, Symbols, Prices. |
| My scope | Maybe both ML arms | **Raw-data / TSFM arm only.** Mack owns the feature-engineering arm (and it didn't perform well — see §4). |
| The reference papers | Not CGM-specific, so tangential | The **ICML'25 paper *is* the relevant TSFM/Chronos reference**; implementation transfers directly to CGM. |

**Net:** the project is now cleanly scoped — *one channel (CGM), one target at a time (×3 models), raw-data/TSFM arm only, learn the input format from the ICML repo, adapt Ben's code.*

---

## 3. The three moving parts (codebases) and what each is for

1. **`diabetes-fitbit`** (Liuyi's repo — the real study).
   `analysis/glucose_cognitive_ml/` is Mack's hand-crafted-feature + classical-ML pipeline. It defines the **real data schema and the three targets**, and its grouped-CV protocol is the yardstick our TSFM arm must match for a fair comparison. *This is where the real data (`data/Merged_glucose_data/`, gitignored) lives.*
2. **`representations-in-tsfms`** (ICML'25 — the TSFM reference Liuyi pointed to).
   Its `HookedChronos` + SVM-classification experiment is the **exact template for "raw window → Chronos embedding → downstream model."** We convert its classification to regression. (It ships no CGM data — its demo data is ECG5000 via the `momentfm` package.)
3. **`mod-actigraphy-advanced`** (Ben's repo — the architecture template).
   Ben's `FoundationModelClassifier` + `FinetuneLightningModule` are the **code we adapt** for the trainable-head version (single channel, regression). Kept as reference only — not developed.

---

## 4. The most important scientific fact (and why it shapes everything)

**Mack's classical feature-engineering arm found essentially no predictive signal.** Across 43 hand-crafted glycemic features and 4 model families, under the same nested subject-grouped CV we will use:

- **Regression R² is negative for all three targets** (worse than just predicting the mean) — in every CV scheme tried (nested GroupKFold, leave-one-patient-out, session-level, normalized targets, augmentation).
- **Classification AUROC ≈ 0.5** (chance).
- Direct Spearman correlations of glycemic metrics vs. scores were essentially null (a handful of |ρ|<0.08 "hits" that wouldn't survive multiple-comparison correction).
- **Key diagnostic:** *session-level* CV (ignoring patient boundaries) did modestly better than *leave-one-patient-out*, which means most of any apparent signal is **subject baseline**, not a generalizable glucose→cognition relationship.

**What this means for our TSFM arm — set expectations honestly:**
- A frozen, general-purpose TSFM will **not manufacture signal that isn't there.** We should not promise big R² gains.
- The scientifically valuable questions the TSFM arm *can* answer are:
  1. **Does raw-representation learning beat hand-crafted features** on this hard target, under the *identical* grouped-CV protocol? (This is the K01's planned comparison, now with a fair head-to-head.)
  2. Do learned embeddings capture **temporal structure** (trajectories, excursion timing) that 43 static summary features throw away?
  3. Is the subject-baseline effect so dominant that **within-subject modeling / normalization** is required before any glucose signal is visible?
- Our pipeline is built to answer these cleanly, and it **reproduces the subject-baseline diagnostic on synthetic data** (group-CV R²≈0, session-CV R²>0), so we can tell "no signal" apart from "a pipeline bug."

---

## 5. What has been built (runnable today)

A self-contained package, `cgm_tsfm/`, implementing both arms end-to-end, validated on synthetic data that matches the real schema (so it's drop-in for the real CSVs):

- **Arm A** (frozen Chronos embeddings + Ridge/SVR, nested GroupKFold): `data.py`, `encoders.py`, `regression.py`, `run_demo.py`.
- **Arm B** (frozen Chronos encoder + trainable regression head, adapted from Ben's code): `ben_adapter/`.
- Both run offline (a `mock` encoder) *and* with real Chronos. See `README.md` for commands and `01_PIPELINE_DESIGN.md` for how each piece maps to the reference repos.

---

## 6. Where to go next

See `02_ROADMAP.md`. The immediate asks for Liuyi are in `03_MEETING_BRIEF.md` and `04_OPEN_QUESTIONS.md` — chiefly: access to the real merged CSVs, the definition of the pre-test window (and whether the *raw* CGM stream is available, not just the pre-clipped array), confirmation of the Prices inversion and what the three scores measure, and what "success" looks like given Mack's null results.
