# SIDEKIC — Progress Update

**DTRC 2026 camera-trap analysis pipeline** · Focus: **Distance** and **Behavior recognition**
Prepared 2026-07-08 (covering recent work) · Brian Zhou

> Context: the pipeline already processes the full **14,577-clip** DJEKE archive for
> detection, individual count, and species. This update covers the two newest stages —
> **distance** ("how far is each animal?") and **behavior** ("what is it doing?").

---

## Highlights
- ✅ **Distance stage completed and validated** — median error **0.95 m** across 64 stations; PR in review.
- ✅ **Two automated quality flags added** — camera-reaction detection and low-confidence-station flagging.
- ✅ **Rigorously evaluated a label-free calibration alternative** and documented why the validated method stays.
- 🔄 **Built and benchmarked a headless behavior detector** — **beats the baseline ~2.2×** on a first pass.

---

## 1. Distance — "how far is each animal from the camera?"

Estimating per-animal distance (for abundance analysis) on grayscale infra-red forest footage.

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

**Status:** complete; pull request in review. A geometry-based extension could calibrate the
*un*-measured stations without new ground truth — pending camera-installation specs from the field team.

---

## 2. Behavior recognition — "what is the animal doing?"

Classifying behavior (stay / run / look / smell / approach). Hard because **~95% of observations
are "stay"** — the informative behaviors are rare.

- **Reproducible evaluation harness + baseline** — established the "always predict stay" baseline
  (**macro-F1 0.157**) as the bar any real model must beat.
- **Evaluated candidate tools** (BehaveAI, PoseR, SLEAP, AniMo) and implemented a **motion-based
  behavior detector** running **headless on the H100 cluster**, auto-labeled from ground truth — no
  manual annotation required.
- **First result: macro-F1 0.351 / mAP 0.354 — ~2.2× the baseline.** Strong recall on the active
  behaviors (running, looking, sniffing); precision is improving through tuning.

**Status:** approach validated and improving; a scaled-up retrain is running now for a stronger number.

---

## Status at a glance
| Workstream | Status | Key result |
|---|---|---|
| Distance calibration | ✅ complete · in review | **0.95 m** median error, 64 stations |
| Distance quality flags | ✅ complete | camera-reaction + 14/64 low-confidence |
| Alt. (label-free) calibration | ✅ evaluated | doesn't generalize → validated method retained |
| Behavior recognition | 🔄 validated · improving | **macro-F1 0.351** (~2.2× baseline) |

## Next steps
1. Merge the Distance (and Productionizing) pull requests — in review.
2. Complete the scaled behavior retrain → improved score.
3. Geometry-based distance for un-measured stations — pending field-installation specs.
