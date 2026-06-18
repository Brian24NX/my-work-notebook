# SIDEKIC — A Plain-English Overview

A from-scratch guide to the whole project. Re-read it whenever the jargon piles up.
Last updated: 2026-06-10.

---

## 1. What *exactly* is SIDEKIC?

**SIDEKIC is an automated video-analysis pipeline for chimpanzee/ape camera-trap
footage**, built for the **Sanz Lab** (WashU) and the **CongoApe** research effort.
The lab has *hundreds of thousands* of camera-trap video clips from field sites in
the Congo Basin:

- **DJEKE** — 14,780 clips
- **GTAP** — 54,830 clips
- **MONDIKA** — 25,428 clips
- (plus reference/calibration videos, "Save the Chimps", etc.)

A human watching all of that is impossible. So SIDEKIC's job is to **turn raw video
into structured scientific data**, answering: *Is there an animal in this clip? What
species? Which individual? How far from the camera? What is it doing?*

Think of it as an **assembly line**: a video goes in one end, and out the other end
comes a rich, queryable record of every animal, frame by frame.

---

## 2. The pipeline (the "assembly line")

Each clip flows through these stages. Each is a separate model/component:

| # | Stage | Model / tool | What it answers |
|---|-------|-------------|-----------------|
| 1 | **Detect & track** | **SAM 3.1** | *Where* is each animal in each frame, and is it the *same* animal across frames? |
| 2 | **Species ID** | **Zamba** | *What species* is it (chimp, gorilla, etc.)? |
| 3 | **Depth / geolocation** | **Depth Anything 3** | *How far* is the animal from the camera; where in real-world space? |
| 4 | **Pose** | pose model (runs in a container) | Body posture / keypoints (limbs, joints). |
| 5 | **Behavior** | behavior model | *What is it doing* (feeding, traveling, etc.)? |
| 6 | **Ape ID (re-ID)** | individual re-identification | *Which specific individual* is this (by face/features)? |

Stages 1–2 are the "front of the line" (find & label the animal); 3–6 are the richer
downstream science. **Not all stages are built yet** — see §6.

---

## 3. Jargon decoder (the terms that confuse everyone)

### SAM — Segment Anything Model
A **foundation model from Meta** for image/video segmentation. SIDEKIC uses
**SAM 3.1** (the gated `facebook/sam3.1` weights, ~3.5 GB).

What makes it special here:
- It's **promptable with text** — you give it a word like `"chimpanzee"` or
  `"animal"` and it finds and outlines matching objects. No training on *your*
  specific animals required to get started.
- It produces a **segmentation mask** (pixel-perfect outline) per object, from which
  SIDEKIC computes a **bounding box** (the rectangle around the animal).
- The **"multiplex video predictor"** version **tracks** objects across frames — it
  assigns each animal an **object ID** and follows it as the video plays. That's the
  "track" in "detect & track."

So SAM = *the eyes of the pipeline*. It does stage 1.

### COCO output — the data format SAM's results are saved in
**COCO** ("Common Objects in Context") is a **standard JSON format** for
object-detection datasets. The whole computer-vision world uses it, so saving in
COCO means any standard tool can read SIDEKIC's results.

A COCO file is basically:
- a list of **images** (or frames),
- a list of **categories** (e.g. "chimpanzee"),
- a list of **annotations** — one per detected object, each with: which image, which
  category, the bounding box `[x, y, width, height]`, the mask, a score, etc.

**This is the project's lightweight, shareable "answer file."** Instead of re-running
the heavy SAM model, anyone can just read the small COCO JSON to see what was found.

> **What we just worked on (PR #4):** we added a **`track_id`** field to each COCO
> annotation. Previously COCO told you "there's an animal in frame 23" and "there's
> an animal in frame 24" but *not* that it's the **same** animal. `track_id` threads
> SAM's tracking identity into the COCO file, so now you can follow one individual
> through the whole clip — which unlocks counting distinct animals, trajectories,
> distance-over-time, etc.

### FiftyOne panel / the dashboard
**FiftyOne** is an open-source app for **visualizing and curating** computer-vision
datasets. It has a **web dashboard** (the "FiftyOne App") where you:
- browse the video frames/images in a grid,
- see the **detections drawn on top** (boxes/masks with labels and `track_id`),
- **filter** (e.g. "show me only clips where a chimp was detected"),
- and **annotate / correct** labels by hand.

So the "FiftyOne panel" *is* the dashboard — it's how a human **looks at** the COCO
output and **fixes/creates labels**. It's the bridge between the automated pipeline
and human experts.

> **Status:** FiftyOne is **not installed yet** on the cluster. Standing it up
> (a "COCO → FiftyOne loader" in `src/sidekic/ui/`) is a planned near-term task. So
> right now there is no live dashboard — that's part of what's coming.

### Fine-tuning — and why we are *not* doing it yet
"Fine-tuning" = taking a pre-trained model (SAM, Zamba, etc.) and **further training
it on your own labeled data** so it gets better at *your* specific animals/conditions.

**The single most important strategic decision this summer: do NOT fine-tune first.**
The reasoning:

- You can't improve what you can't **measure**. To know if fine-tuning helped, you
  need a **labeled benchmark** (ground-truth "right answers") and an **eval harness**
  (code that scores the model against them).
- Right now there are **14,780 DJEKE clips but ZERO labels / metrics / training code
  on disk**. So today, model quality is literally unmeasurable.
- Therefore the **real critical path is getting labels/annotations**, *not* GPU
  horsepower or training.

**Order of operations:**
1. Build the **eval harness** + a **labeled benchmark** (ground truth).
2. Measure the current pipeline's quality (baseline).
3. *Then*, if needed, fine-tune — and you'll be able to prove it helped.

Fine-tuning itself (when it eventually happens) would mean things like swapping in a
trained detection/pose "head," or training the species/re-ID models on lab-labeled
examples. Most of that is **deferred to later this summer**.

### Smaller terms
- **Eval harness** — code that runs the pipeline on test clips and computes metrics
  (precision/recall, how often it falsely fires, double-counts, etc.).
- **Benchmark split** — a frozen set of hold-out clips (specific cameras + time
  periods) reserved for honest scoring, so you don't "cheat" by testing on data you
  tuned on.
- **Track ID** — the per-individual identity number (what PR #4 added to COCO).
- **Smoke test** — a tiny end-to-end run (e.g. 1 clip, 3 frames) just to confirm the
  whole pipeline is wired up and runs, not to measure quality.

---

## 4. The tech stack & infrastructure

- **Compute:** WashU **RIS Compute2**, a **SLURM cluster** with **H100 GPUs**. Heavy
  work runs as submitted GPU jobs (`srun` / `sbatch`), **never on the login node**.
- **Language/env:** **Python 3.12**, managed by **uv** (a fast Python package manager).
- **Deep learning:** **PyTorch 2.6.0** pinned to a **CUDA 12.4** build (the cluster's
  GPU driver requires it — a newer build silently breaks GPU access).
- **Models:** SAM 3.1 (installed editable + patched), Zamba, Depth Anything 3, pose
  (containerized via Pyxis/Enroot — no Docker/sudo on the cluster).
- **Repo:** `CongoApe-SIDEKIC/SIDEKIC` (private GitHub). Code lives at
  `/storage1/fs1/crickette.sanz/Active/DTRC_2026/SIDEKIC` on the cluster; data sits
  alongside under `DTRC_2026/`.
- **Output format:** COCO JSON (per §3).
- **Planned viewer / annotation:** FiftyOne.

> ⚠️ Two cluster gotchas worth knowing: (1) **never run `uv run`** in this repo — it
> re-resolves and breaks the carefully-pinned torch; always use `.venv/bin/python`.
> (2) The torch build **must** be the cu124 one or the GPU goes invisible.

---

## 5. Who's doing what (the team lanes)

Work is split into **lanes** so people don't collide:

- **You (Brian) + Claude** → the **upstream** lane: detection schema, counting, the
  **eval harness**, and **annotation**. (This is where `track_id` and the FiftyOne
  work live.)
- **Nick (NBStephens)** — project lead → the **depth + geolocation** lane (distance
  from camera, real-world location). Branch `feature/depthanything`.
- **Noah (NoahWolk1)** → the **pose** lane (pose estimation in containers on the
  cluster). Branch `noah/compute2-pose-service`.
- **Danni Beaulieu** → **code owner** — reviews PRs.

Your work sits *upstream* and **feeds** Nick's depth stage (e.g. `track_id` lets
distance be tracked per individual over time).

---

## 6. Where things stand right now

**Done / merged:**
- Pipeline runs end-to-end on the cluster; SAM 3.1 detect+track on a real clip → COCO
  works (smoke-tested on an H100).
- Cluster setup + repo rules merged (PRs #2, #3).
- **`track_id` in COCO** — approved by both reviewers; being finalized (PR #4).

**Near-term roadmap (mostly *not* blocked on GPU):**
1. **Ask the team where expert species/count labels live** (none found on disk — this
   decision drives everything).
2. ✅ Track-ID fix (PR #4).
3. **Detection-quality baseline** from existing COCO output (blank-rate, false fires,
   hand precision/recall, double-counting, distinct-track counts).
4. **Stand up FiftyOne** as the viewer + annotation tool.
5. **Freeze a benchmark split** and scale SAM to a few thousand clips.
6. Misc. plumbing (rebuild `xformers` for the depth stage; fix config paths).

**Deferred to later:** pose head-swap, behavior, full re-ID, and any model fine-tuning.

---

## TL;DR

> SIDEKIC turns huge piles of **raw chimp camera-trap video** into **structured data**
> (who / where / what / how-far / doing-what), using a chain of AI models — starting
> with **SAM** (finds & tracks animals), saving results in **COCO** (a standard JSON
> format), viewable/annotatable in a **FiftyOne dashboard**. The smart play this
> summer is **not to train/fine-tune yet** — first build the **labeled benchmark +
> scoring harness** so quality is *measurable*, because right now there are tons of
> videos but **zero labels**. Your lane is the upstream detection/counting/eval/
> annotation work that everything else builds on.
