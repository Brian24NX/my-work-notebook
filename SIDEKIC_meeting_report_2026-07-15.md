# SIDEKIC — Progress Report
### Camera-trap video → wildlife data, end to end · Sanz Lab / DTRC 2026
**Brian Zhou · for the 2026-07-15 check-in · RIS Compute2 (H100)**

> *Figures live in `~/report_figs/`. This report is the meeting version — visual and high-level.
> A companion deep-dive (`SIDEKIC_deepdive_distance_behavior.md`) has the full technical detail.*

---

## Executive summary

**SIDEKIC now runs end to end and has processed the entire DJEKE archive — all 100 cameras.** Since the MVP we finished the two hardest stages, **Distance** and **Behavior**, and produced the lab's two deliverable tables for the full archive.

Three headlines for today:

1. **The pipeline is complete and has been run over all 100 cameras.** Two tables are produced: **speciesID** (19,424 rows) and the complete **distances** table (76,751 rows — species, count, distance, reaction, behavior, every column filled).
2. **Distance is at 0.86 m and beats the best published infrared pipeline (1.85 m) by ~2×.** We ran a 9-method bake-off; a per-camera **quadratic** fit won. The fit is now *tapped out* — the next gain comes from a better foot signal (pose), not a better curve.
3. **Behavior works at 2.3× the naive baseline** (macro-F1 0.36) on a brutally imbalanced problem (~96% "stay"), on a clean, leak-free, camera-disjoint scoreboard. Reliable on the common behaviors; the rare ones are example-starved (a *data* problem we've scoped).

Everything below is honest about what's solid and what's indicative.

---

## 1 · What SIDEKIC does

![SIDEKIC pipeline](report_figs/fig_pipeline.png)

Point it at a folder of clips; it runs five stages and writes two tables. Each stage is **tailored to our footage** — dense forest, and a mix of daytime-color and low-contrast night-infrared. Off-the-shelf models trained on daytime web images fail here, which is *why* every stage had to be adapted.

![Real DJEKE footage](report_figs/fig_domain.png)

---

## 2 · The deliverables (all 100 cameras)

![Deliverable composition](report_figs/fig_deliverable.png)

- **speciesID table** — species + count per clip. 19,424 rows, 100 cameras.
- **distances table** — one row per animal per 2 seconds, with **distance + reaction + behavior**. 76,751 rows, 100 cameras, every column filled.
  - **62,570 rows** from the 64 hand-measured cameras (the validated half).
  - **14,181 rows** from the 36 remaining cameras, distances computed **with no extra fieldwork**.

---

## 3 · How good is each stage? (honest scorecard)

![Accuracy scorecard](report_figs/fig_scorecard.png)

| Stage | Metric | Result | Confidence |
|---|---|---|---|
| Detect | recall | **~92%** | ✅ solid |
| Count | within ±1 individual | **~90%** | ✅ solid |
| Species | micro-F1 | **~57%** (beats off-the-shelf on our footage) | ✅ solid |
| Distance (measured cameras) | error | **0.86 m** | ✅ hand-validated |
| Distance (computed cameras) | error | **~1.28 m** | 🟡 model estimate |
| Behavior | macro-F1 | **0.36** (2.3× the 0.157 floor) | 🟡 indicative |

**Read the distances table by confidence:** detect / count / species / distance are the dependable columns; **behavior is indicative** — trust "stay" and look/smell, treat the rare/fast behaviors as leads.

---

## 4 · Distance — the headline technical result

**The idea:** from a *fixed* camera, an animal's **foot position in the frame** encodes how far away it is.

![Distance geometry](report_figs/fig_geometry.png)

**What we did since the MVP:** for the 64 cameras with ground-truth tape measurements, we ran a **9-method bake-off** to find the best foot-position → distance curve. A per-camera **quadratic** won.

![Distance calibration evolution](report_figs/fig_distance_evolution.png)

![9-method bake-off](report_figs/fig_bakeoff.png)

We got here in steps: ordinary least-squares (0.95 m) → a robust line that resists bad points (Theil-Sen, 0.90 m) → a **curve** that matches the real perspective geometry (**quadratic, 0.86 m**, now **PR #34**). Notably, the *textbook* pinhole formula was tested too and **lost** — the real curve isn't a clean pinhole, so the empirical quadratic wins.

**Where the remaining error lives:**

![Where the distance error lives](report_figs/fig_distance_decomp.png)

Well-sited cameras are already at **0.70 m**. The average is pulled up by **14 "off-kilter" cameras** aimed too low/flat — those are a *siting* problem (~2 m under **any** method), not something a better fit can fix. **The curve-fitting is now tapped out (~0.85 m); the next real gain is a better foot signal — pose keypoints — not a better curve.**

**The 36 cameras with no tape measurements** got distances from a **shared install-geometry model** — calibrated on the 64 measured cameras and transferred, **with zero extra fieldwork** (~1.28 m).

**Answering Danni's question — how many labels does a *new* camera need?**

![Label budget](report_figs/fig_label_budget.png)

~**5–6** ground-truth points get a well-sited camera to ~1 m; off-kilter cameras never get there regardless (re-site, don't re-label).

**Versus the field:** the closest published infrared distance pipeline reports **1.85 m** — **we're ~2× better.** Off-the-shelf depth models were tested and failed on our footage.

![SOTA positioning](report_figs/fig_sota.png)

---

## 5 · Behavior — 2.3× the naive baseline

**The challenge** is extreme imbalance: ~96% of the time the animal is just "stay."

![Behavior imbalance](report_figs/fig_behavior_imbalance.png)

So the bar to clear is a naive model that *always* predicts "stay" (macro-F1 **0.157**). Our model beats it **2.3×**:

![Behavior vs floor](report_figs/fig_behavior_vs_floor.png)

**How it works** — the *BehaveAI* method, run headless: turn movement into color, detect the moving blobs with a YOLO detector, and read off the active behaviors; "stay" is the default.

![Behavior method](report_figs/fig_behavior_method.png)

**Per-behavior**, it's strong where there's data and weak where there isn't:

![Behavior by class](report_figs/fig_behavior_byclass.png)

The weak classes (run / approach / rest) are **example-starved** (rest has ~7 test clips) — a *data* problem, not a tuning problem. This was measured on a **clean, camera-disjoint scoreboard** (we found and fixed a data-leak that had flattered an earlier number — we'd rather report the honest 0.36).

**On the tools you asked about (Danni):** *BehaveAI* is exactly the method we run; *PoseR* is the same skeleton family as Noah's merged work; *SLEAP* needs us to hand-label keypoints; and **AniMo is the wrong instrument for us** — it's a text-to-3D-animation *generator* with no public code, not a rare-class fix. The best cheap next step is loss-balancing + temporal smoothing on the model we already have.

**Versus the field:** on the MammAlps benchmark a heavyweight video model scores ~0.45 mAP on a *less* imbalanced set; we're at ~0.37 mAP on a harder 96%-stay set — a sane, competitive range.

---

## 6 · Status & what's next

**On `main` (merged):** Detect · Count · Species · **Distance (#26)** · **Productionizing (#23)**.
**Open pull requests:** **#33** (behavior) · **#34** (quadratic distance calibration).
**Produced:** both deliverable tables, all 100 cameras.

**Next levers, ranked:**
1. **Distance:** better foot signal via **pose keypoints** (the fit itself is maxed out).
2. **Behavior:** loss-balancing + logit-adjustment + temporal smoothing (biggest cheap win), then a motion-vs-pose head-to-head and ensemble.
3. **Re-site the 14 off-kilter cameras** for future seasons (a field action, not a modeling one).

**Asks for today:** a code-owner review on **#33** and **#34**; and a steer on which next lever to prioritize.

---

*Prepared with an AI pair-programmer (Claude Code). All numbers are reproducible on the cluster; see the companion deep-dive for methods, code, and commit references.*
