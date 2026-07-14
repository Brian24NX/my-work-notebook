# SIDEKIC — Deep Dive: Distance & Behavior (and everything we did this afternoon)

### A long-form walkthrough written to *teach*, not to summarize.
**Brian Zhou · 2026-07-14 · Sanz Lab / DTRC 2026**

> This is the "hone my knowledge" document. It is deliberately long. Read it slowly — every
> concept is built up from first principles, with the actual numbers and code from our pipeline.
> Figures are in `~/report_figs/`. The short meeting version is `SIDEKIC_meeting_report_2026-07-15.md`.

---

## Table of contents

- [0. Orientation — where we are, and what changed this afternoon](#0)
- [PART A — DISTANCE](#partA)
  - [A1. The core idea: a foot position is a distance](#a1)
  - [A2. Two kinds of cameras: measured vs computed](#a2)
  - [A3. What is "robustness fitting"? (the big teaching section)](#a3)
  - [A4. Why a *quadratic* — and why the textbook physics *lost*](#a4)
  - [A5. The 9-method bake-off, read carefully](#a5)
  - [A6. Where the error actually lives](#a6)
  - [A7. The fit is tapped out → the SAM3 experiment & pose](#a7)
  - [A8. Computed cameras (065–100) & the label budget](#a8)
  - [A9. How we compare to the published field](#a9)
- [PART B — BEHAVIOR](#partB)
  - [B1. Why behavior is genuinely hard here](#b1)
  - [B2. The four tools Danni named — what each *actually is*](#b2)
  - [B3. Our method: motion-YOLO, step by step](#b3)
  - [B4. Results, and the honest scoreboard-leak story](#b4)
  - [B5. Where behavior goes next (Tiers 0–3)](#b5)
- [PART C — The complete all-100 deliverable](#partC)
- [PART D — Current state, PRs, next levers](#partD)
- [PART E — Glossary](#partE)
- [Appendix — files, commits, how to reproduce](#appendix)

---

<a name="0"></a>
## 0. Orientation — where we are, and what changed this afternoon

The whole pipeline is a five-stage assembly line:

![pipeline](report_figs/fig_pipeline.png)

Detect / Count / Species were done and merged earlier. **Distance** and **Behavior** are the two stages we've been perfecting, and they're what this document is about.

**What changed this afternoon (the work the crash dropped from the chat, but which is safe in git + memory + files):**

| Thing | Before this afternoon | After |
|---|---|---|
| Distance fit | robust line (Theil-Sen), 0.90 m | **quadratic curve, 0.855 m** — after a 9-method bake-off (**PR #34**) |
| Foot signal | box-bottom | tested a **SAM3 mask** for a better foot point → **no-go on IR** (documented) |
| Behavior tools | motion-YOLO built | **SOTA review** of AniMo / BehaveAI / PoseR / SLEAP → AniMo is the wrong tool; next levers ranked |
| Deliverable | 001–064 complete | **all 100 cameras** complete (76,751 rows) |

None of that is lost. It lives in: the **memory file** (`~/.claude/.../sidekic-status.md`), the **git history / PR #34**, and **files on disk** (`~/research_distance_sota.md`, `~/research_behavior_sota.md`, `~/distance_calib_methods.py`, `~/sam_foot.py`, the deliverable TSVs). The workflow is fully intact.

---

<a name="partA"></a>
# PART A — DISTANCE

<a name="a1"></a>
## A1. The core idea: a foot position *is* a distance

A camera trap is **bolted in place**. It never moves. That one fact is the whole trick.

When a camera is fixed, the ground in front of it maps predictably onto the image. An animal standing **close** has its feet **low in the frame** (near the bottom); an animal standing **far** has its feet **high in the frame** (near the horizon line). So the single number we need is **where the animal's feet are, vertically, in the image** — we call it `foot_y` (the bottom edge of the detection box, divided by image height, so it's 0–1).

![geometry](report_figs/fig_geometry.png)

Look at the **left** panel. The key, slightly counter-intuitive, fact is in the picture: as an animal walks *away*, its feet climb toward the horizon **more and more slowly**. Near the camera, one step changes `foot_y` a lot; far from the camera, a big distance change barely moves `foot_y`. That "compression" at range is exactly why the relationship **curves** — hold that thought for A4.

So the distance stage is really just: **learn the function `foot_y → distance` for each camera**, then apply it to every detection. "Learning that function" is a *curve-fitting* problem, and that is where all the interesting work is.

<a name="a2"></a>
## A2. Two kinds of cameras: measured vs computed

- **Cameras 001–064 — "measured."** The field team physically tape-measured reference distances, giving us ~30 ground-truth `(foot_y, distance)` points per camera (~2,114 points total). For these we fit the curve **directly** from truth. This is the validated half.
- **Cameras 065–100 — "computed."** No one measured these. For them we can't fit a per-camera curve — so we use a **shared geometry model** learned from the measured cameras and transfer it (A8). No fieldwork required.

Everything in A3–A7 is about the *measured* cameras (where the fitting science happens).

<a name="a3"></a>
## A3. What is "robustness fitting"? (the big teaching section)

You asked exactly this, so let's build it up properly.

### A3.1 What "fitting" means
We have a cloud of points for one camera: each point is `(foot_y, measured_distance)`. We want a **function** `d = f(foot_y)` that passes through the cloud as well as possible, so that for a *new* detection we can plug in its `foot_y` and read off a distance. "Fitting" = choosing `f` to minimize the mismatch between `f(foot_y)` and the measured distances.

### A3.2 How we grade a fit — leave-one-out (LOO)
We never grade a fit on the same points we fit it on (that flatters it). Instead, **leave-one-out**:

```
for each reference point i in a camera:
      fit the curve on all points EXCEPT i
      predict point i, record |prediction − truth|
report the average error   →  "LOO MAE" (mean absolute error, in metres)
```

LOO simulates "how well would this predict a point it has never seen?" — an honest estimate of real-world accuracy. Every distance number in this document is a LOO MAE.

### A3.3 The problem robustness solves: outliers
Real reference points are noisy. Occasionally the animal-detector puts the box bottom in the wrong place (a shadow, an occluding log), or a tape measurement is mis-recorded. That single point is an **outlier** — far off the true line.

The classic fit, **ordinary least squares (OLS)**, chooses the line that minimizes the **sum of *squared* errors**. Squaring is the catch: a point that's 4 m off contributes 16, a point 1 m off contributes 1. So **one bad point dominates the whole fit** and drags the line toward itself. OLS is *not robust*.

"**Robustness fitting**" = fitting methods designed so that a handful of bad points **can't** wreck the fit. Here are the ones we tested, from least to most sophisticated:

| Method | One-line intuition | Robust? | Shape it assumes |
|---|---|---|---|
| **OLS** | minimize squared error | ❌ (outliers dominate) | straight line |
| **Theil-Sen** | slope = **median** of the slopes between all pairs of points | ✅✅ (median ignores extremes) | straight line |
| **Huber** | squared penalty for small errors, **linear** for big ones | ✅ (middle ground) | straight line |
| **RANSAC** | randomly try small subsets, keep the line with the most "inliers" | ✅✅ (for *many* points) | straight line |
| **Isotonic** | no shape — just force distance to **always decrease** as foot_y rises | ✅ (flexible) | any monotone curve |
| **Quadratic** | fit a **parabola** (one gentle bend) | partially | curved |
| **Pinhole ("Physical")** | the textbook camera equation: `1/distance` is linear in foot_y | — | curved (from physics) |

**Theil-Sen**, the one we adopted first (PR #34's predecessor), is worth understanding because it's so elegant: take every possible *pair* of points, compute the slope of the line through that pair, and use the **median** of all those slopes. To move a median you'd have to corrupt *half* the points — so a few outliers are simply invisible to it. (In code it's ~5 lines of numpy: `np.triu_indices` for the pairs, `np.median` for the slope.)

**RANSAC** deserves a note because it's famous and we *rejected* it: it works by throwing away points that don't agree with a random consensus. That's fantastic when you have thousands of points and lots of junk — but we have only ~30 points per camera, so throwing points away costs us more than the outliers do. It scored **worse**. A good reminder that the "best" method depends on the data you actually have.

### A3.4 The result of just adding robustness

![distance evolution](report_figs/fig_distance_evolution.png)

Going from OLS (0.950 m) to a robust **line** (Theil-Sen, 0.905 m) bought us ~5%. Real, but modest. That told us something important: **most of our error was not from outliers.** If it were, robustness would have helped a lot more. The error was coming from somewhere else — the *shape*.

<a name="a4"></a>
## A4. Why a *quadratic* — and why the textbook physics *lost*

Back to the compression idea from A1. Because far-away feet bunch up near the horizon, the true `foot_y → distance` relationship is a **curve**, not a line:

*(right panel of the geometry figure above)* — the blue curve is the real relationship; the red dashed straight line **can't** follow it. A line is forced to compromise: it's wrong in the near field *and* wrong in the far field. No amount of making the line *robust* fixes a line that's the **wrong shape**. This is "model mis-specification," and it was our dominant error.

So we replaced the line with a **quadratic** — `distance = a·foot_y² + b·foot_y + c`, three numbers per camera. A parabola has exactly one gentle bend, which matches the perspective curvature well. It dropped us to **0.855 m** — a bigger jump than robustness gave us, confirming the diagnosis.

**The genuinely interesting twist:** the *textbook* answer to "what's the right curve?" is the **pinhole camera model**, which says `1/distance` should be linear in `foot_y`. We tested it (it's the "Physical" / "Inv-Linear" methods in the bake-off). **It lost** — 0.979 m, *worse* than a plain OLS line. Why? The pinhole formula assumes a **perfectly flat ground plane** and exact camera geometry. Our forest floor is uneven, cameras are tilted at odd angles, and the "foot point" is the bottom of a bounding box (not the true ground contact). Under those violations, the clean physics equation mis-fits, and a **flexible empirical curve that just bends to match the data** (the quadratic) wins. *Empirical beats theoretical when the theory's assumptions are broken* — a nice, real lesson.

**Implementation detail worth knowing (PR #34):** a parabola can do crazy things *outside* the range of foot positions you actually fit it on (it can turn around and start increasing). So the calibration stores, per camera, the three coefficients **plus the observed `foot_y` range**, and when applying it we **clamp** the input to that range. Safe, and it never extrapolates a nonsense distance.

<a name="a5"></a>
## A5. The 9-method bake-off, read carefully

We didn't just guess — we ran all nine fits through the same leave-one-out grader (`~/distance_calib_methods.py`) and ranked them:

![bake-off](report_figs/fig_bakeoff.png)

Three things to read *out* of this chart, because they're subtle:

1. **Quadratic wins the overall median (0.855 m)** and is the choice we shipped.
2. **Isotonic (0.866 m) actually wins *more individual cameras* (28 of 64)** and is best on the badly-sited cameras (1.78 m vs quad's 2.01 m). So why not isotonic? Because it's **non-parametric** — instead of 3 tidy numbers it stores a whole step-function per camera, which is clumsier to serialize and can overfit tiny samples. Quadratic gives *nearly* the same accuracy in a compact, smooth form. This is an engineering trade-off (accuracy vs. simplicity), and it's worth being able to explain it.
3. **All three inverse-distance / pinhole methods are at the bottom (0.99–1.22 m), greyed out as "rejected."** That's the physics-lost story from A4, shown quantitatively.

Two more numbers from the same run that bound the problem:
- **Oracle = 0.734 m** — if a magic genie picked the *perfect* method for each camera, we'd get here. It's the ceiling for "better fitting."
- **Honest per-camera selection = 0.842 m** — if we actually try to pick the best method per camera (by nested leave-one-out, no cheating), we get essentially what the single quadratic already gives. **Translation: fiddling with the fit is done. 0.84–0.86 m is the wall.**

<a name="a6"></a>
## A6. Where the error actually lives

Averages hide structure. Break the error down by camera type:

![where error lives](report_figs/fig_distance_decomp.png)

- **Well-sited measured cameras: 0.70 m.** This is our real capability when the camera is aimed sensibly.
- **All measured cameras: 0.855 m.** The average is dragged up by...
- **14 "off-kilter" cameras: ~2 m under *every* method.** These are aimed too low or too flat, so the ground barely projects into the frame and `foot_y` stops encoding distance. We *detect* these automatically (`flag_offkilter.py`) and flag them low-confidence. **No fitting method rescues them** — the fix is physical (re-aim the camera next season). This matters: it separates a *modeling* limit from a *field* limit, and only the field one is left here.
- **Computed cameras (065–100): ~1.28 m** — see A8.

<a name="a7"></a>
## A7. The fit is tapped out → the SAM3 experiment & pose

If the curve is maxed out at ~0.85 m, the only way down is a **better input signal**. Right now the "foot point" is just the **bottom of the bounding box**, which is a biased proxy — it includes a bit of leg, shadow, or gap under the belly. A more precise ground-contact point would cut error *before* the curve ever sees it.

Two ways to get a better foot point, and we tried the cheaper one this afternoon:

- **SAM3 segmentation mask (tested → no-go).** SAM ("Segment Anything Model") draws a pixel-perfect outline of an object; the lowest row of the animal's *mask* is a much better ground-contact point than a box bottom. We wired it up (`~/sam_foot.py`). **It failed on our infrared footage**: SAM3's box-prompt path is really a *concept detector* that returned **zero masks** on low-contrast IR frames, even at zero confidence. This is the **same infrared domain-gap** that killed the off-the-shelf depth models — more evidence for our running thesis that *models trained on daytime web images don't transfer to IR camera traps*. Documented and shelved.
- **Pose keypoints (the next thing to try).** A pose model (e.g. DLC **SuperAnimal-Quadruped**, zero-shot) predicts the animal's joints, including **paws**. The midpoint of the two lowest paws is an unbiased ground-contact point. This is the recommended next distance lever — it attacks the signal, not the curve. (Noah already has the DLC container running for the behavior side, so the tooling exists.)

<a name="a8"></a>
## A8. Computed cameras (065–100) & the label budget

For the 36 cameras with **no** field measurements, we can't fit a per-camera curve. Instead we fit **one shared install-geometry model** on the measured cameras — a physically-parameterized version, `d = h / tan(θ + arctan((v−0.5)·k))`, where `h`, `θ`, `k` describe how the camera is mounted. We then transfer it.

We validated the transfer honestly with **leave-station-out**: hold out an entire camera, fit on the other measured ones, predict the held-out camera. Result: **1.28 m** (1.15 m on clean cameras). That's the number we quote for the computed half — larger than the 0.86 m measured half, and clearly labeled as a model estimate.

**Danni's question — "how many ground-truth points does a *new* camera need?"** We simulated it:

![label budget](report_figs/fig_label_budget.png)

- Fitting a fresh curve from **2 points** is unstable (~3.7 m). By **5–6 points** a well-sited camera reaches ~1 m.
- The parametric model **+ a 1-parameter correction** is much more forgiving at low label counts (~1.1–1.2 m at just 2–4 points), because it already "knows the shape" and only needs anchoring.
- **Off-kilter cameras stay above ~2 m no matter how many labels you collect** — again, re-site, don't re-label.

Practical guidance for the field team: **~5–6 well-placed reference points per new, well-sited camera** buys ~1 m.

<a name="a9"></a>
## A9. How we compare to the published field

We did a full SOTA review (`~/research_distance_sota.md`). The short version:

![sota](report_figs/fig_sota.png)

- The **closest published infrared distance pipeline** (Haucke/"timmh," MegaDetector + depth + SAM) reports **1.85 m**. We're at **0.86 m** — roughly **2× better**, because our geometric foot-point backbone sidesteps the metric-depth problem that IR breaks.
- **Off-the-shelf metric depth** (Depth-Anything-3) was tested on our data and barely correlated with truth (corr +0.22) — dead on IR, exactly as the literature warns. Our approach *is* the current correct recipe; the realistic gains are the incremental foot-signal/curve refinements above, not a model swap.

---

<a name="partB"></a>
# PART B — BEHAVIOR

<a name="b1"></a>
## B1. Why behavior is genuinely hard here

The behavior stage reads a short clip of an animal and tags what it's doing — a **multi-label** problem (an animal can be looking *and* smelling at once) over classes like STAY / LOOK / SMELL / RUN / APPROACH / REST.

The killer is **class imbalance**:

![imbalance](report_figs/fig_behavior_imbalance.png)

Roughly **96% of the time the animal is just "stay."** That has two nasty consequences:

1. A lazy model that **always predicts "stay"** already looks ~96% "accurate" by naive accuracy. So plain accuracy is a useless metric here.
2. The interesting behaviors are **rare**, so there's little data to learn them from.

We handle (1) by scoring with **macro-F1** (averages the score *per class*, so rare classes count as much as STAY) and **mAP**, and by setting a concrete **floor**: a model that always predicts STAY gets macro-F1 **0.157**. *Any* real model must beat 0.157 to have earned its keep.

We also evaluate **leave-camera-out** (train on some cameras, test on entirely different ones) so we measure *generalization to new cameras*, not memorization of a camera's background.

<a name="b2"></a>
## B2. The four tools Danni named — what each *actually is*

You asked specifically what these are. Full analysis is in `~/research_behavior_sota.md`; here's the teaching version.

**BehaveAI** — *motion → color → YOLO.* This is a **motion-based** method: it converts movement between frames into a color image, then runs a YOLO object-detector to find and classify the moving things. It ships as a GUI app, **but the *method* is exactly what we run** (we rebuilt it headless so it runs on the cluster). PLOS Biology 2025. → **This is our approach.**

**PoseR** — *skeleton → ST-GCN.* A **pose-based** method: first estimate the animal's skeleton (joints), then feed the moving skeleton to a graph neural network (ST-GCN) that classifies the motion pattern. It's a napari GUI plugin. Crucially, this is the **same family as Noah's already-merged skeleton behavior work** — so as a *separate* tool it adds little novelty for us.

**SLEAP** — *pose estimation, but you label it yourself.* A high-quality multi-animal pose tracker, but it has **no zero-shot animal model** — you must hand-annotate keypoints on your own footage to train it. Data-efficient (<200 labels) but a labeling burden. → A fallback only.

**AniMo** — **the one to flag.** AniMo is a **text-to-3D-animal-motion *generator*** (arXiv:2512.10352): you type "a leopard runs" and it synthesizes a 3D animation. It has **no public code**, a fixed skeleton topology, and — most importantly — **it does not solve our problem.** Our problem is *recognizing* rare behaviors in 2D infrared clips; AniMo *generates* 3D motion from text. It's the **wrong instrument** for rare-class augmentation. (If we ever wanted synthetic data for the rare tail, the on-target tool is a text-to-*video* tail-balancer like Gen2Balance, not AniMo.) **This is the useful thing to tell Danni:** we evaluated it and it's a category mismatch, not a tuning knob.

The one-line mental model:

```
   BehaveAI  = motion signal  → YOLO         (what we run)
   PoseR     = skeleton signal → ST-GCN       (= Noah's family)
   SLEAP     = make your own skeletons (label) (fallback)
   AniMo     = generate motion from text       (wrong task for us)
```

<a name="b3"></a>
## B3. Our method: motion-YOLO, step by step

![behavior method](report_figs/fig_behavior_method.png)

Here's the whole pipeline, unpacked:

```
1. RAW FRAMES         take frames from the clip.
2. MOTION → COLOUR    subtract consecutive frames; where things moved, paint it in colour
                      (direction/speed of motion become hue/brightness). Stationary background
                      goes flat/grey. Now "movement" is literally visible as coloured shapes.
3. MOTION BLOBS       the coloured shapes are the moving animal; draw boxes around them.
4. YOLO11 DETECTOR    train YOLO (an object detector) to look at the motion-colour frame and
                      classify each blob as run / look / smell / approach / rest.
5. STAY PRIOR         if there's no meaningful motion → "stay" (the default).
```

Why this shape? Because **motion is illumination-robust**: it doesn't care whether the frame is daytime-color or night-IR, which sidesteps the domain-gap that breaks appearance-based models. Its blind spot is the flip side: **it can't see stationary behavior** (a motionless "look"), so STAY is the safe default and the model specializes in the *active* behaviors.

We added three **precision levers** (this afternoon's lineage of work) to stop it over-calling behaviors:
- **largest-blob-only labels** — label the dominant mover, not every flicker.
- **hard negatives** — feed it motion frames that contain *no* behavior (wind, vegetation) as empty "background," so it learns what *not* to fire on.
- **sustained aggregation** — only accept a behavior if it persists for **≥2 frames** in a 2-second window (kills one-frame false alarms).

<a name="b4"></a>
## B4. Results, and the honest scoreboard-leak story

![behavior vs floor](report_figs/fig_behavior_vs_floor.png)

We beat the floor **2.3×**: macro-F1 **0.359**, mAP **0.371** (floor 0.157). Per behavior:

![behavior by class](report_figs/fig_behavior_byclass.png)

- **Strong** where there's data: SMELL (AP 0.51), LOOK (0.49).
- **Weak** where there isn't: APPROACH (0.16), RUN (0.12), REST (~0.01, only ~7 test clips).

The weak classes are **example-starved** — this is a **data** problem, not a modeling problem, and it tells us exactly where to spend labeling effort next.

**The honesty story (worth telling out loud).** An earlier run reported a *higher* 0.398. When I audited the split, I found a **data leak**: one clip had a blank camera ID that sorted to the front of the list and slipped into the *test* set while its sibling clips were in *train* — so the model had effectively seen the test camera. I fixed the split to key off the **station parsed from the video name** (truly camera-disjoint), re-ran, and got the **honest 0.359**. We chose to report the lower, correct number. (This is the same "don't fool ourselves" discipline as leave-one-out on the distance side.)

<a name="b5"></a>
## B5. Where behavior goes next (Tiers 0–3)

From the SOTA review, ranked by payoff-per-effort:

- **Tier 0 (hours, no new data) — the biggest cheap win.** Swap the training loss to a **class-balanced / focal / asymmetric** loss and add **logit adjustment** (a principled per-class threshold based on how rare the class is — our current threshold-tuning is the ad-hoc version of this), plus **temporal smoothing** over per-frame scores. Directly targets the rare tail; strengthens PR #33.
- **Tier 1 (moderate) — motion vs pose head-to-head.** Run DLC SuperAnimal zero-shot to get skeletons, train a **PoseConv3D** behavior model on the *same* split, and compare it to motion-YOLO on our tool-agnostic scoreboard. (Why bother? The **ASBAR** study found pose **matches/beats** video on wild apes at ~20× less data and **generalizes across sites better** — but it *also* found pose doesn't fix the rare-class tail. So pose is about *cross-camera robustness*, not the rare behaviors.)
- **Tier 2 (cheap once both exist) — ensemble.** Late-fuse motion + pose. They fail differently (motion = movers/near, blind on stationary; pose = background-free, needs decent keypoints), so combining them is likely the headline result.
- **Tier 3 (heavy, only if stuck) — generative tail balancing / active learning.** Mine the unlabeled cameras for likely rare-behavior clips for Danni to label (best labeled-data ROI), or synthesize tail-class video.

**External yardstick:** on the MammAlps benchmark a heavyweight VideoMAE model gets ~0.45 mAP on a *less*-imbalanced set; our ~0.37 mAP on a harder 96%-STAY set is a **sane, competitive range** — we're not off in the weeds.

---

<a name="partC"></a>
# PART C — The complete all-100 deliverable

![deliverable](report_figs/fig_deliverable.png)

The distances table is the richer of the two deliverables — one row per animal per 2 seconds, every column filled, for **all 100 cameras** (76,751 rows). We built it by:
1. running the behavior model over the 065–100 clips (the last missing piece), then
2. **assembling** the halves (`~/assemble_all100.py`): 001–064 (measured distance, validated) + 065–100 (parametric distance + predicted species/count + behavior).

**Read it honestly:**
- **001–064** is the hand-validated half (measured distance ~0.86 m).
- **065–100** is model output: parametric distance (~1.28 m), *predicted* species/count (no ground truth exists there), timestamp metadata `NA`, per-detection `focal.id`.
- **behavior_string** everywhere carries the model's ~0.36 macro-F1 — reliable STAY + look/smell, indicative on the rest.

*(Note: the shipped distances currently use the linear calibration; the quadratic upgrade (#34) improves the calibration, and refreshing the 76,751 rows to reflect it is a day-long re-emit we deferred so the table stays internally consistent for the meeting.)*

---

<a name="partD"></a>
# PART D — Current state, PRs, next levers

**Merged to `main`:** Detect · Count · Species · **Distance (#26)** · **Productionizing (#23)**.
**Open PRs:** **#33** behavior (motion-YOLO + tool-agnostic scoreboard) · **#34** quadratic distance calibration.
**Produced & verified:** both deliverable tables, all 100 cameras.

**The next levers, in priority order:**
1. **Distance → pose feet.** The curve is tapped out (~0.85 m); a pose-based ground-contact point is the only lever left that moves the needle. (SAM3 was tried and is out.)
2. **Behavior → Tier 0 loss-balancing.** Biggest cheap macro-F1 gain; then the motion-vs-pose head-to-head and ensemble.
3. **Field → re-site the 14 off-kilter cameras.** A physical fix for the biggest chunk of distance error.
4. **Reviews & merges:** #33 and #34 need a code-owner (Danni/Noah).

---

<a name="partE"></a>
# PART E — Glossary

- **foot_y** — the vertical position of a detection's box-bottom in the frame, normalized 0–1. Our distance signal.
- **calibration** — the learned function `foot_y → distance` for a camera.
- **OLS (ordinary least squares)** — the standard line fit; minimizes squared error; sensitive to outliers.
- **robust fit** — a fit that a few bad points can't wreck (Theil-Sen, Huber, RANSAC…).
- **Theil-Sen** — robust line: slope = median of all pairwise slopes.
- **Huber** — line fit that penalizes big errors linearly (not squared) so outliers count less.
- **RANSAC** — fit by random-consensus; great with lots of data + junk, poor on our small samples.
- **isotonic regression** — a shape-free fit constrained only to be monotonic (always decreasing here).
- **quadratic fit** — a parabola `a·x²+b·x+c`; captures the perspective curvature; our winner.
- **pinhole model** — the textbook camera equation (`1/distance` linear in foot_y); theoretically right, empirically worse here (broken flat-ground assumption).
- **LOO MAE (leave-one-out mean absolute error)** — hold out one point, predict it, average the errors; our honest accuracy metric, in metres.
- **off-kilter camera** — a camera aimed so low/flat that foot_y stops encoding distance; a *siting* problem (~2 m under any fit).
- **parametric / shared-geometry model** — one physically-parameterized calibration learned on measured cameras and transferred to unmeasured ones.
- **leave-station-out** — hold out a whole camera to test transfer (the distance analog of leave-camera-out).
- **SAM (Segment Anything Model)** — pixel-level object segmentation; we tried its mask for a better foot point (failed on IR).
- **pose estimation** — predicting an animal's joints/keypoints (incl. paws); the next distance signal and a behavior route.
- **multi-label** — a clip can carry several behavior tags at once.
- **class imbalance** — some classes vastly outnumber others (96% STAY here).
- **macro-F1** — F1 averaged per class, so rare classes count equally; the right behavior metric.
- **mAP (mean average precision)** — threshold-free detection/classification quality; higher is better.
- **floor** — the score of the trivial always-predict-STAY model (0.157); the bar to beat.
- **YOLO** — a fast object detector (we use YOLO11) — the classifier in motion-YOLO.
- **ST-GCN / PoseConv3D** — neural nets that classify behavior from a moving skeleton (pose-based).
- **motion-to-color** — encoding inter-frame movement as a color image so a detector can "see" motion.
- **hard negatives** — training examples that look tempting but should fire nothing (wind, vegetation).
- **data leak** — test information sneaking into training; inflates scores; we found and fixed one in behavior.

---

<a name="appendix"></a>
# Appendix — files, commits, how to reproduce

**Distance**
- Calibration + quadratic: `scripts/calibrate_distance.py` (PR #34, commit `f5da85a`); fits `fit_quadratic`, applies clamped to observed foot_y range.
- Bake-off (9 methods, this afternoon): `~/distance_calib_methods.py` — re-run with `<repo>/.venv/bin/python ~/distance_calib_methods.py`.
- SAM3 foot-mask experiment (no-go): `~/sam_foot.py`.
- Parametric (065–100): `scripts/parametric_distance.py`, `scripts/distance_detect_065.py`.
- Off-kilter / reaction flags: `scripts/flag_offkilter.py`, `scripts/detect_camera_jerk.py`.
- SOTA brief: `~/research_distance_sota.md`.

**Behavior**
- Method: `scripts/build_motion_yolo_dataset.py`, `scripts/motion_yolo_infer.py`; scoreboard `scripts/behavior_eval_harness.py` (PR #33).
- Model: `~/behavior_eval/yolo_runs/motion_prec/weights/best.pt`.
- SOTA brief (AniMo/BehaveAI/PoseR/SLEAP + tiers): `~/research_behavior_sota.md`.

**Deliverable**
- Assembly: `~/assemble_all100.py`, `~/behavior_for_distances.py` → `~/distance_out/distances_all100_full.tsv` (76,751 rows) and `~/pipeline_poc/speciesID_djeke_v3.tsv` (19,424 rows).

**Figures**
- All regenerated by `~/gen_report_figs.py` (`<repo>/.venv/bin/python ~/gen_report_figs.py` → `~/report_figs/`). Exact bake-off numbers came from the live `~/distance_calib_methods.py` run.

**Where nothing gets lost**
- Durable state: the memory file `~/.claude/projects/-home-zhou-n/memory/sidekic-status.md`, the git history / PRs, and the files above. A dropped chat session does not lose the work.

---

*Written with an AI pair-programmer (Claude Code). Take your time with it — that's what it's for.*
