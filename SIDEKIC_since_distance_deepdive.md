# SIDEKIC — Deep-Dive: Everything We've Done Since Distance

*A complete, plain-English walkthrough — with diagrams, real numbers, worked examples, and a
glossary — of the Distance, Productionizing, and Behavior work. Written so you can read it
top-to-bottom and be fully oriented on **what we did, why, the results, and what's next.***

Prepared 2026-07-08, updated 2026-07-09 · Brian Zhou. (Companion docs: `SIDEKIC_HANDOFF.md` = terse technical
reference; `SIDEKIC_progress_2026-07-07.md` = leadership slide-style summary. This one is the
full story.)

> **How to read this:** skim the **30-second dashboard** and the **big-picture diagram** first.
> Then each stage has the same shape — *the problem → what failed → what worked → results →
> status*. Technical words are explained inline and collected in the **Glossary** at the end.

---

## 0. The 30-second dashboard

| Stage | What it answers | Status | Headline result |
|---|---|---|---|
| 1 Detect | Is there an animal? | ✅ merged | ~92% of animals found |
| 2 Count | How many? | ✅ merged | ~90% within ±1 individual |
| 3 Species | What species? | ✅ merged | micro-F1 ~57% (beats off-the-shelf) |
| **4 Distance** | **How far away?** | ✅ done (all 100 stations), **PR #26 approved** | **0.95 m** measured + **~1.28 m** parametric |
| **Productionizing** | **One push-button program** | ✅ done, **PR #23 in review** | **14,577 clips processed** |
| **5 Behavior** | **What is it doing?** | 🔄 validated · **PR #33 open** | **macro-F1 0.359 (~2.3× baseline)** |

**One-sentence status:** the first four stages are finished (Distance **#26 approved**; Productionizing
**#23** pending), and the behavior stage is a validated model that beats the baseline ~2.3× — now up as
**PR #33**.

---

## 1. The big picture — what the whole system is

SIDEKIC is an **assembly line** for camera-trap video. Each "station" adds one label:

```
   🎥 one video clip
        │
        ▼
   ┌──────────┐   ┌──────────┐   ┌───────────┐   ┌────────────┐   ┌──────────────┐
   │ 1 DETECT │──▶│ 2 COUNT  │──▶│ 3 SPECIES │──▶│ 4 DISTANCE │   │ 5 BEHAVIOR   │
   │ animal?  │   │ how many │   │ which     │   │ how far    │   │ what is it   │
   │          │   │          │   │ species   │   │ (metres)   │   │ doing        │
   └──────────┘   └──────────┘   └───────────┘   └────────────┘   └──────────────┘
        └──────────── one program runs 1→3 (“Productionizing”) ─────────┘
        │                                                                │
        ▼                                                                ▼
   OUTPUT A: speciesID table                                    OUTPUT B: distances table
   (one row per clip:                                           (one row per animal, per 2 s:
    species + count)                                             distance, reaction, behavior)
```

**The two deliverables the lab wants** are those two tables (they match the exact column formats
the lab gave us). Everything we build feeds one of those two tables.

**The single biggest lesson across all of it:** *off-the-shelf AI trained on normal daytime color
photos fails on our footage,* which is **grayscale night-vision (infra-red) in dense forest**.
We hit this three separate times (detection, species, distance) and each time the fix was a
model/method tailored to our data.

---

## 2. Timeline — what we actually did since Distance

```
  DISTANCE ───────────────────────────────────────────▶ (done, all 100 stations, PR #26)
    ├─ tried off-the-shelf depth AI ............ failed (out of domain)
    ├─ built per-station calibration (001-064) . ✅ ~0.95 m
    ├─ added camera-reaction flag (Colleen) .... ✅
    ├─ added low-confidence-station flag ....... ✅ 14/64
    ├─ tested Danni's tape-video idea .......... ✅ evaluated → doesn't generalize
    ├─ parametric model → 065-100 (no GT) ...... ✅ ~1.28 m, 14,181 distances  (NEW)
    ├─ GT-label budget for new stations ........ ✅ ~5-6 labels → ~1 m         (NEW)
    └─ PR #26 review → **Danni approved** ...... ✅ ready to merge             (NEW)

  PRODUCTIONIZING ────────────────────────────────────────▶ (done, PR #23)
    ├─ one end-to-end program .................. ✅ 14,577 clips
    └─ addressed review (Danni + bot) .......... ✅

  BEHAVIOR ───────────────────────────────────────────────▶ (validated → PR #33)
    ├─ built a fair scoreboard + baseline ...... ✅ floor = 0.157
    ├─ evaluated 4 tools, picked BehaveAI ...... ✅
    ├─ derisk: motion works on night video ..... ✅
    ├─ rebuilt BehaveAI's method headless ...... ✅
    ├─ first result .............................. ✅ 0.351
    ├─ fixed a GPU-waste / thrash issue ........ ✅
    ├─ precision fixes (hard negatives, …) ..... ✅ beat the plain baseline (A/B)
    ├─ fixed a scoreboard leak (NA-site) ....... ✅ split now by physical camera
    ├─ clean, honest result .................... ✅ macro-F1 0.359 / mAP 0.371 (~2.3×)
    └─ opened PR #33 (additive) ................ ✅ renamed our scoreboard; Noah's untouched
```

The rest of this doc explains each of these lines in depth.

---

# 3. STAGE 4 — DISTANCE  ("how far is each animal?")

### 3.1 The problem, with a concrete example
A clip shows a duiker (small forest antelope) at night. The lab needs: *how many metres is it
from the camera?* They use these distances for **abundance estimation** (roughly, how densely
animals populate the forest). Eyeballing distance from grainy night-vision is unreliable, so we
need the computer to output a number like "3.2 m."

### 3.2 What we tried first — and why it failed
| Attempt | Idea | Result | Why it failed |
|---|---|---|---|
| Depth-Anything-3 | An off-the-shelf "how far is everything?" AI | corr **+0.22** with truth (≈ random) | Trained on daytime color photos; our IR night footage is out of its world |
| One global formula | A single distance formula for all cameras | corr **+0.09** | Every camera is aimed differently — no single formula fits |
| Near/mid/far bins | Just classify coarse distance bands | ~chance (35%) | Same reason — cross-camera geometry varies |

*Plain terms: "correlation" (corr) is "how well the guess tracks the truth," from 0 (random) to
1 (perfect). +0.22 is basically guessing.*

### 3.3 What worked — a per-camera "cheat sheet"
**Key insight:** in a *fixed* camera, **where an animal's feet sit in the frame tells you how far
it is.** Feet near the bottom = close; feet higher up = far. (Like any photo: near things sit low,
far things sit high.)

```
   ┌───────────────────────────┐
   │            🐗  feet high   │  →  ~8 m   (far)
   │          (near horizon)    │
   │                            │
   │   🐗  feet low             │  →  ~1.5 m (near)
   │   (bottom of frame)        │
   └───────────────────────────┘
     For EACH camera we fit its own line:   distance ≈ a × (foot height) + b
```

We built this per-camera line for all **64 cameras** where the lab had already hand-measured some
real distances (2,114 measured points total).

![Distance error — our per-station calibration vs off-the-shelf depth AI](progress_charts/distance_error.png)

**Result: median error 0.95 m** — vs ~1.5 m for the off-the-shelf approach. About 1 m is roughly
the *physical floor* here (an animal's feet shuffle enough that no method does much better).

**How we know it's honest — "leave-one-out" (LOO):** to test the cheat sheet without cheating, we
hide one measured point, rebuild the line from the *others*, then predict the hidden one and check
the error. Repeat for every point. It's like quizzing yourself with a flashcard you covered up —
you can't fool yourself. The 0.95 m is this honest, held-out number.

### 3.4 Two automatic quality flags
Real footage has messy cases, so we added two safety checks:

**(a) Camera-reaction flag** *(Colleen's idea)* — sometimes an animal **bumps the camera** and the
whole picture jolts (like knocking your phone mid-video). That distance reading is worthless.
```
   frame t   :  🐗 standing, camera still      → distance valid
   frame t+1 :  💥 whole scene JOLTS            → animal reacted → distance = NA (but keep the clip)
```
We detect the jolt (a sudden shift of the *entire* frame, distinct from an animal just moving) and
mark `reaction.yn = Y`, blank the distance, **but keep the clip** — "animal reacted to camera" is
itself interesting to researchers. *(This also fills in a column we'd been leaving empty.)*

**(b) Low-confidence-station flag** — some cameras give shaky distances no matter what. We
auto-flag them: **14 of 64**. We traced the cause to **ground slope** (a camera on a hillside sees
the ground foreshortened, so foot-height stops predicting distance). Jake later confirmed the
truly-*bumped* cameras had already been removed from this dataset — so it's terrain, not knocks.

### 3.5 Danni's idea — and an honest "it doesn't work"
Danni asked a smart question: *instead of hand-measuring distances, can we use the "setup videos"?*
(Each camera has a video where a fieldworker lays **measuring tapes** on the ground at 1-metre
marks.) If it worked, we'd never need hand-measuring again — huge for the un-measured cameras.

We tested it rigorously across all 64 cameras. **Verdict: it works on the tidy setup videos but
does not generalise:**
- reference-video-only error: **2.3 m** (vs 0.95 m hand-measured),
- its geometry barely matched the truth (correlation **−0.08**),
- and only **44 of 64** setup videos even had detectable tapes (leaves cover them; distance signs
  are too blurry to read).

So it's **bottlenecked by how cleanly each setup video was filmed** — not fixable in software. We
kept the hand-measured method as the deliverable.

> **A note on rigor (and a good catch by Danni):** while testing this, I made a measurement mistake
> — I let the "answer key" leak into the "study material," which made one number look better than it
> was. Danni caught it; I retracted that number and fixed the code so it can't recur. This is normal,
> healthy science — the point is we don't fool ourselves.

### 3.6 The parametric model — distances for the 36 un-measured cameras (065–100)
Cameras 065–100 were never hand-measured, so the per-camera cheat sheet (3.3) can't be built for
them. But Jake told us the cameras are installed in a **standardised way** (≈1 m high, fixed angle,
same orientation) — which means **one shared camera geometry applies to every camera.** So instead
of a separate cheat sheet per camera, we fit **one geometry formula** (real physics: a camera at a
known height + tilt maps foot-position → distance) on the 64 measured cameras and reuse it everywhere.

**Does that actually transfer to a camera we never measured?** We tested it honestly: fit the geometry
on 63 cameras, predict the 64th (which had *no* say in the fit), check the error. Repeat for each.

| How we'd distance a new camera | Error | Needs |
|---|---|---|
| Per-camera cheat sheet (3.3) | 0.95 m | that camera's own hand-measured points |
| **Shared geometry, no measuring** | **1.28 m** | nothing — just the standardised install |
| Shared geometry + that camera's slope | 1.01 m | one slope number |

So with **zero fieldwork** we get ~1.28 m (and ~1.15 m on normally-sited cameras). We ran it on the
**3,642 animal-bearing clips** in 065–100 → **14,181 distances across all 36 cameras.** **The archive
now has distances for all 100 cameras.**

**Sanity check (there's no ground truth for 065–100, so we can't score it directly):** the predicted
distances look just like the measured ones — same **4.0 m median** — and **97%** of animals appear in
the same part of the frame the model was trained on (so it's barely extrapolating).

![065–100 parametric distances — sanity-checked against ground truth](progress_charts/distance_065_qc.png)

### 3.7 How many ground-truth labels does a *new* camera need?
Danni asked a practical question for future deployments: if hand-measuring helps, **how many measured
points per camera** do we need to stay near 1 m? We answered it from the existing data:

![How many GT labels for ~1 m](progress_charts/distance_label_budget.png)

- **~5–6 measured points** per well-sited camera → ~1 m.
- With only **2–4**, anchor to the shared geometry + a small correction → ~1.1–1.2 m (fitting a fresh
  line from 2 points is wildly unstable — 3.7 m).
- Sloped cameras never reach 1 m no matter how many points → they need **re-siting, not more labels.**

Concrete rule for the field team: *collect ~5 distance points at each new camera.*

### 3.8 Distance — deliverable & status
- **Deliverables:** the `distances` table for **001–064** (`~/distance_out/distances_djeke.tsv`, hand-
  calibrated) **and 065–100** (`~/distance_065/distances_065.tsv`, parametric) — **all 100 cameras.**
- **Code:** `calibrate_distance.py`, `emit_distances.py`, `flag_offkilter.py`, `detect_camera_jerk.py`,
  `validate_reference_calibration.py`, `parametric_distance.py`, `distance_detect_065.py`,
  `distance_label_budget.py`, `distance_065_qc.py`.
- **Review addressed:** Danni's PR #26 review (a crash on markers-only calibration, an over-sensitive
  reaction flag, + minor notes) and 6 Copilot notes — all fixed, **28 tests pass**, replies posted. Her
  final note (each transfer fold was seeded from the all-data fit) is fixed too; re-seeding **confirmed
  the 1.28 m was already honest** (unchanged, to 5 decimals).
- **Status:** complete (all 100 cameras); **PR #26 is approved by Danni — ready to merge.** 🏁

---

# 4. PRODUCTIONIZING  ("make it one push-button program")

Individually the stations work; productionizing **glues detect → count → species into one program**
you point at a clip folder and it writes the `speciesID` table.

```
   folder of clips ─▶ pipeline_poc.py ─▶ speciesID table
                       (MegaDetector → count → species classifier → join metadata)
```

- **Ran on the full DJEKE archive: 14,577 clips, 100 cameras, 0 crashes, ~24 h.**
- Species **micro-F1 56.3%**; counts **90% within ±1**. *(F1 = a balanced accuracy score, explained
  in the glossary.)*
- **Deliverable:** `~/pipeline_poc/speciesID_djeke_v3.tsv`, browsable in a visual dashboard
  (`djeke_speciesID_v3_review`).
- **Review addressed:** Danni noted we had a home-made *copy* of the counting logic → switched to the
  official tested version; a review bot flagged a few robustness nits → fixed.
- **Status:** complete; **pull request #23** in review. *(Note: it overlaps a detection+species
  backend Noah merged recently — Danni will decide how they combine.)*

---

# 5. STAGE 5 — BEHAVIOR  ("what is the animal doing?")

This is the newest and most involved thread. Goal: label behavior — **stay, run, look, smell,
approach, rest** — per animal, per moment.

### 5.1 Why it's genuinely hard: the data is 96% "just standing"

![The behavior challenge — 96% of the data is “STAY”](progress_charts/behavior_imbalance.png)

Almost everything an animal does on camera is **stand there** ("STAY"). The *interesting*
behaviors (running, sniffing, looking around) are rare — under 5% combined. **Analogy:** learning a
language where 95% of every sentence is the word "the" — the rare, meaningful words are hard to
learn because there are so few examples.

*(Silver lining: measured per whole-clip rather than per-second, the rare behaviors have usable
counts — LOOK in 709 clips, SMELL 506, RUN 407, APPROACH 212 — enough to train on.)*

### 5.2 The decision, and the first thing I built (a fair scoreboard)
You chose to build **your own behavior line** (on a separate branch), in parallel with Noah's — may
the best one win. Before trying any tool, I built a **referee**: a scoring program that grades any
behavior model the *same way*, on the *same held-out cameras*, so "who's better" is objective.

**The baseline ("floor"):** if you cheat and just guess "STAY" every single time, you score
**macro-F1 = 0.157**. That's the bar any real model must beat.

### 5.3 The four tools Danni suggested, and how they fit
| Tool | What it is | Needs | Our take |
|---|---|---|---|
| **BehaveAI** | Detects behavior from **motion** (no skeleton needed) | just video | **Tried first** — most different, best fit for "spot the moving animal at night" |
| PoseR | Behavior from a **skeleton** (same method Noah used) | a pose/skeleton first | queued; needs a pose source |
| SLEAP | A **pose/skeleton** estimator | hand-labeled keypoints | possible pose source (needs labeling) |
| AniMo | **Generates** synthetic motion (data augmentation) | — | later, if data stays scarce |

The pose-based tools (PoseR, AniMo) need a **skeleton** first, so they're a coupled sub-project for
later. BehaveAI stands alone → we started there.

### 5.4 BehaveAI in one picture, and the derisk
BehaveAI's trick: **turn motion into color** — a moving animal leaves a colored trail — then a
detector spots it. Perfect for us, because **a moving animal lights up even in the dark.**

**Before installing anything, I derisked the core question — does this even work on our IR footage?**
```
   raw night frame  ─▶  motion-color frame
   (duiker hard to see)  (duiker GLOWS red/green)   ✅ yes, it works on grayscale IR
```
Caveats found: it can't see a *stationary* animal (no motion), and far animals are faint — so we use
it as a **"catch the active behaviors" detector**, and keep "STAY" as the default answer.

### 5.5 The clever part — running a laptop GUI tool as an automated cluster job
BehaveAI is a **click-around desktop app** (needs a screen + you hand-drawing boxes on frames). Our
cluster is a screenless server. You asked "can you do it for me?" — so instead of the app, **I
rebuilt its *method* to run fully automatically on the cluster:**
```
  clips ─▶ make motion-color frames ─▶ auto-draw boxes on the moving blob
        ─▶ auto-label them from our ground truth (no hand-drawing!)
        ─▶ train a YOLO detector ─▶ test on held-out cameras ─▶ score vs the 0.157 floor
```
*(YOLO = a fast, standard object-detector. "Auto-label from ground truth" = we already know each
clip's behavior, so we tag the frames automatically instead of by hand.)*

### 5.6 The results

![Behavior recognition — 2.3× the naive baseline](progress_charts/behavior_macrof1.png)

| Version | macro-F1 | mAP | vs floor |
|---|---|---|---|
| Baseline (guess STAY) | 0.157 | — | 1× |
| Plain motion-YOLO | 0.345 | 0.346 | 2.2× |
| **+ precision fixes** | **0.359** | **0.371** | **2.3×** |

The precision fixes (hard negatives, largest-blob labels, multi-frame aggregation) beat the plain
version in a **controlled A/B** — same data, same training — on overall quality (**mAP 0.371 vs
0.346**), and the improved model is far better *calibrated* (works at a normal sensitivity; the plain
one only worked cranked to an extreme). All numbers are on the **corrected, leak-free scoreboard**
(see the note below).

**Honest per-behavior breakdown** (this is what matters, not a single vanity number):

![Behavior detector — F1 by behavior](progress_charts/behavior_f1_by_class.png)

It's genuinely decent at **look (0.48)** and **smell (0.46)**, weak at run/approach/rest — those three
are example-starved (rest has only 7 test clips, so no method will shine there).

> **A trap we avoided (worth understanding):** an earlier chart showed "100% recall" on look/smell.
> That sounds amazing but is **misleading** — it meant the model shouted "look/smell!" on almost
> *every* clip, so it trivially "caught" all the real ones while being wrong ~85% of the time.
> **Analogy:** a smoke alarm that beeps 24/7 catches 100% of fires but is useless. The honest
> measure is **F1** (above), which balances *catching them* against *false alarms*.

**We then did the precision work** — hard negatives (showing it wind/vegetation labeled as "nothing"),
largest-blob labels, and multi-frame aggregation. On the clean scoreboard it beat the plain version
(mAP 0.371 vs 0.346) and is much better-calibrated. The remaining weak behaviors (run/rest/approach)
are **example-starved**, not a tuning problem — so the next lever is more labels or a pose-based method.

> **A scoreboard bug we caught + fixed (worth knowing):** the way we split cameras into "study" vs
> "exam" sets relied on a camera-site label; one clip's label was blank, which quirkily became its own
> "exam" camera and dragged one real camera (DJK052) into *both* sides. We fixed the split to key off
> the **physical camera in the filename**. The honest number moved slightly (0.384 → **0.359**) — the
> earlier figure was on the flawed split. Same "less flattering but trustworthy" pattern as the
> 100%-recall catch above.

### 5.7 A lesson we learned the hard way: right hardware for the job
While scaling up, a build job accidentally ran on the shared **login node** and thrashed for
**113 minutes**. Two fixes, now standing rules:
```
   GPU  = a giant printing press → great at one thing: neural-net math (training)      → use for training/inference
   CPU  = versatile workers      → decoding videos, prepping data, glue logic          → use for data prep
   RULE: always a COMPUTE NODE (never the shared login node); request a GPU only when the job trains a model.
```
Fixed the build to run in parallel across CPU cores on a compute node → the H100 is no longer held
idle during data prep.

### 5.8 Behavior — status & the PR
- **Code (PR branch `brian/behavior-motion-yolo`):** `behavior_eval_harness.py` (the tool-agnostic
  scoreboard), `build_motion_yolo_dataset.py`, `motion_yolo_infer.py`, `behaveai_to_behavior.py`,
  `run_superanimal_pose.py` + slurm jobs. **13 tests pass.**
- **Result:** validated on a **leak-free scoreboard**, **macro-F1 0.359 / mAP 0.371 (~2.3× baseline)**;
  the precision fixes beat the plain baseline in a controlled A/B.
- **PR #33 opened (additive).** While our line was in progress, Noah's *skeleton-based* behavior model
  merged into `main` (#30) — and both lines happened to have a file named `eval_behavior.py` (his grades
  *his* model; ours is the tool-agnostic scoreboard). So we opened the PR the clean way: a fresh branch
  off `main`, our scoreboard **renamed `behavior_eval_harness.py`** so **Noah's file is untouched** —
  purely additive (10 new files). The PR invites the team to decide which scoreboard to standardize on.
- **What's left:** the rare behaviors (run/rest/approach) are example-starved; the skeleton route (PoseR /
  SuperAnimal) is the other option if we want a higher ceiling.

---

# 6. Where we are now (dashboard)

| Thread | State | Key result | Waiting on |
|---|---|---|---|
| Distance (001–064) | ✅ done | 0.95 m, hand-calibrated | **#26 approved → your merge** |
| Distance (065–100) | ✅ done | ~1.28 m parametric, 14,181 distances, QC-clean | **#26 approved → your merge** |
| Distance flags | ✅ done | reaction + 14/64 low-confidence | — |
| Reference-video calibration | ✅ evaluated, ruled out | doesn't generalize | — |
| PR #26 review | ✅ addressed + **approved** | bugs fixed, 28 tests, Danni approved | your merge click |
| Productionizing | ✅ done | 14,577 clips | Danni to merge #23 |
| Behavior | ✅ validated · **PR #33 open** | macro-F1 0.359 / mAP 0.371 (2.3×), leak-free | review of #33 |

**Everything substantive is done or improving; the two finished stages are gated on a teammate's
review, not on more work from us.**

---

# 7. What's next (prioritized)

1. **Merge #26 (distance) — it's approved** (one click). Then nudge Danni on **#23** (productionizing)
   and get **#33** (behavior) reviewed. *Highest leverage.*
2. **Behavior — the precision pass is done** (it beat the plain baseline in a clean A/B). The next
   behavior lever is more labeled examples for the rare behaviors (run/rest/approach), or the pose route.
3. **Pose-based behavior track** (skeleton → PoseR) — queued behind the BehaveAI result.
4. **Scale to other sites** (GTAP, MONDIKA) once their footage is mounted.

*Distance is now complete for all 100 cameras. Jake's install specs would only fine-tune the ~14
sloped cameras — a bonus, not a blocker (and he may not have exact numbers, which is fine).*

---

# 8. Glossary (plain-English)

- **Recall** — of the real cases, what fraction did we catch. (100% = missed none — but says nothing
  about false alarms.)
- **Precision** — of the cases we flagged, what fraction were right. (Low precision = crying wolf.)
- **F1** — one number balancing precision and recall (high only if *both* are good). The honest
  single score.
- **macro-F1** — the average F1 across behaviors (so rare behaviors count as much as common ones).
- **mAP** — "mean average precision"; a threshold-free overall quality score for a detector.
- **Baseline / floor** — the score of a trivial strategy (here, "always guess STAY" = 0.157); a real
  model must beat it.
- **Calibration** — tuning a per-camera formula from known reference points (our distance cheat sheet).
- **Leave-one-out (LOO)** — honesty test: predict each point from all the *others*.
- **Correlation (corr)** — how well a guess tracks the truth, 0 (random) → 1 (perfect).
- **Foot position / foot_y** — where an animal's feet sit vertically in the frame (our distance cue).
- **YOLO** — a fast, widely-used object-detector (finds + labels boxes in an image).
- **Motion encoding** — turning frame-to-frame movement into color, so motion is visible.
- **Camera-site split** — testing on cameras the model never trained on (so scores are honest, not memorized).
- **GPU vs CPU** — GPU = massively-parallel math (neural-net training); CPU = general step-by-step
  work (video decode, data prep). Match the job to the hardware.
- **PR (pull request)** — a proposed code change awaiting a teammate's review before merging.

---

# 9. Appendix — where things live / how to re-run

**Deliverables**
- Species + counts: `~/pipeline_poc/speciesID_djeke_v3.tsv`
- Distances 001–064 (hand-calibrated): `~/distance_out/distances_djeke.tsv`
- Distances 065–100 (parametric): `~/distance_065/distances_065.tsv`
- Behavior model + predictions: `~/behavior_eval/yolo_runs/motion/`, `~/behavior_eval/motion_yolo_pred.tsv`

**Re-run (all on a compute node, never the login node)**
```bash
# distance (001-064, hand-calibrated)
sbatch slurm/distance_calibration.sbatch          # per-camera calibration
sbatch slurm/emit_distances.sbatch                # write the distances table
# distance (065-100, parametric, no ground truth)
python scripts/parametric_distance.py fit         # fit the shared geometry on 001-064
sbatch slurm/distance_065.sbatch                  # detect + emit for 065-100
# productionizing
sbatch slurm/pipeline_djeke_v3.sbatch             # full pipeline over DJEKE
# behavior (build + train + score, one job)
sbatch slurm/behavior_motion_yolo.sbatch          # motion-YOLO trial
python scripts/behavior_eval_harness.py --pred <preds> --min-score 0.9   # score vs the 0.157 floor
```

**Pull requests:** **#26 (distance) — approved, ready to merge**; **#23** (productionizing) — in review;
**#33 (behavior, motion-based) — open** (branch `brian/behavior-motion-yolo`).
