# SIDEKIC — Progress Update

**DTRC 2026 camera-trap analysis pipeline** · Focus: **Distance** and **Behavior recognition**
Prepared 2026-07-08 (covering recent work) · Brian Zhou

> Context: the pipeline already processes the full **14,577-clip** DJEKE archive for
> detection, individual count, and species. This update covers the two newest stages —
> **distance** ("how far is each animal?") and **behavior** ("what is it doing?").

---

## Highlights
- ✅ **Distance stage now covers all 100 stations** — **0.95 m** on the 64 measured stations, plus a **parametric model that adds the 36 previously-deferred stations with no new fieldwork** (~1.28 m, QC-validated); merged to main.
- ✅ **Quantified the ground-truth-label budget** — **~5–6 labels per new station → ~1 m**, giving the field team a concrete target for future deployments.
- ✅ **Two automated quality flags added** — camera-reaction detection and low-confidence-station flagging.
- ✅ **Rigorously evaluated a label-free calibration alternative** and documented why the validated method stays.
- ✅ **Built and benchmarked a headless behavior detector** — **beats the baseline ~2.3×** (macro-F1 0.359) on a corrected, leak-free scoreboard; precision fixes validated by a controlled A/B.

---

## 1. Distance — "how far is each animal from the camera?"

Estimating per-animal distance (for abundance analysis) on grayscale infra-red forest footage.

![Distance error — our per-station calibration vs off-the-shelf depth AI](progress_charts/distance_error.png)

- **Per-station calibration → 0.95 m median error** (leave-one-out validated) across all 64
  ground-truthed stations — vs ~1.5 m for off-the-shelf depth models, which are out-of-domain
  on night-vision footage. This is near the physical floor for the method.
- **Camera-reaction flag** — automatically detects when an animal disturbs the camera and marks
  that distance unusable, while keeping the clip (animal–camera reactions are of ecological interest).
- **Low-confidence-station flag** — 14 of 64 stations flagged for review; root-caused to **terrain
  slope**, not camera disturbance (consistent with field-team input).
- **Evaluated a label-free alternative** (calibrating from the tape "setup" videos): it works on
  clean recordings but does **not** generalize across the archive (variable recording quality), so
  the validated per-station method is retained. The evaluation itself was hardened after a
  peer-caught methodology issue.

**Extended to the 36 un-measured stations (065–100).** A parametric camera-geometry model —
calibrated on the 64 ground-truthed stations and relying on the standardized camera install —
now assigns distances to the previously-deferred stations with **no new fieldwork**: 14,181
distances, QC-validated against the ground-truth distribution (identical **4.0 m median**; 97%
of detections fall within the calibrated range). **The full DJEKE archive now has distances for
all 100 stations.** Expected error ~1.28 m (from leave-one-station-out cross-validation).

![065–100 parametric distances — sanity-checked against ground truth](progress_charts/distance_065_qc.png)

**How many GT labels does a new station need?** ~**5–6 labels** per well-sited station reach
~1 m; with only 2–4, anchoring to the standardized-install geometry keeps error ~1.1–1.2 m
(a fresh per-station fit is unstable that low). Sloped stations don't reach 1 m at any label
count — they need re-siting, and are auto-flagged.

![GT-label budget to reach ~1 m](progress_charts/distance_label_budget.png)

**Status:** complete for all 100 stations; merged to main. Jake's installation specs
will further tighten the 36 parametric stations (physically anchor the geometry + per-station slope).

---

## 2. Behavior recognition — "what is the animal doing?"

Classifying behavior (stay / run / look / smell / approach). Hard because **~95% of observations
are "stay"** — the informative behaviors are rare.

![Behavior recognition — 2.3× the naive baseline](progress_charts/behavior_macrof1.png)

![Behavior detector — F1 by behavior (honest operating point)](progress_charts/behavior_f1_by_class.png)

- **Reproducible evaluation harness + baseline** — established the "always predict stay" baseline
  (**macro-F1 0.157**) as the bar any real model must beat.
- **Evaluated candidate tools** (BehaveAI, PoseR, SLEAP, AniMo) and implemented a **motion-based
  behavior detector** running **headless on the H100 cluster**, auto-labeled from ground truth — no
  manual annotation required.
- **Result: macro-F1 0.359 / mAP 0.371 — ~2.3× the baseline.** Strongest on **look & smell**
  (F1 ≈ 0.46–0.48); run/rest/approach stay weak — they're example-starved (rest: 7 test clips).
- **Precision fixes validated by a controlled A/B** — hard negatives + largest-blob labels +
  multi-frame aggregation beat the plain version on overall quality (**mAP 0.371 vs 0.346**) and are
  far better-calibrated (usable at a normal sensitivity, not only cranked to an extreme).
- **Corrected the evaluation scoreboard** — a stray missing camera-site had made one station leak
  across the train/test split; it's now split by physical camera. The clean **0.359** is slightly
  below the earlier (flawed-split) 0.384 — this is the honest, defensible number.

**Status:** validated on a leak-free scoreboard; precision fixes shipped. The remaining weak
behaviors need more labeled examples, not more model tuning.

---

## Status at a glance
| Workstream | Status | Key result |
|---|---|---|
| Distance — all 100 stations | ✅ complete · merged to main | **0.95 m** (64 measured) + **~1.28 m** parametric (36 new) |
| Distance quality flags | ✅ complete | camera-reaction + 14/64 low-confidence |
| Alt. (label-free) calibration | ✅ evaluated | doesn't generalize → validated method retained |
| Behavior recognition | 🔄 validated · leak-free scoreboard | **macro-F1 0.359** (~2.3× baseline) |

## Next steps
1. **#26 + #23 merged to main** ✅ — now finishing the deliverable (writing behavior into the distances table, running) and getting **#33** (behavior) reviewed.
2. Behavior: precision fixes done + scoreboard corrected; next is more labeled examples for the rare behaviors, or a pose-based approach.
3. Refine the 36 parametric stations with Jake's installation specs (anchor geometry + per-station slope).
