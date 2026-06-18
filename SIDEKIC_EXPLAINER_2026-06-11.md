# SIDEKIC — Full Explainer & Session Backup (2026-06-11)

> **What this file is.** A complete, self-contained writeup of the orientation
> conversation between Brian and Claude on 2026-06-11 — what the project is, why
> we're doing what we're doing, where we left off, what's next, a plain-language
> glossary of every term/tech, and one real code block explained. Saved so the
> conversation's content isn't lost.
>
> **Where it lives & why.** This file is on the **RIS cluster** at
> `/home/zhou.n/SIDEKIC_EXPLAINER_2026-06-11.md` (permanent, backed-up home
> storage) — NOT on the Mac. Shutting the Mac down does not affect it.
>
> **To resume the actual chat:** the full transcript auto-saves under
> `~/.claude/projects/`. Run `claude --resume` (pick this session) or
> `claude --continue` (latest) to keep talking with full context. This `.md` is
> the human-readable summary; the transcript is the verbatim record.
>
> **Live state was verified against GitHub during this session** (PRs, branch,
> commits) — not just recalled from memory.

---

# 1. What is this project, really?

## The science behind it

This is for the **Sanz Lab** (Prof. Crickette Sanz, WashU). They study **wild
great apes — gorillas and chimpanzees — in the Congo rainforest**, especially a
site called the **Djéké Triangle** (that's what "DJEKE" / "DJK" means everywhere
in the code and data).

They collect data with **camera traps**: small weatherproof cameras strapped to
trees that automatically record a short video clip whenever something moves or
gives off heat (an animal walking past). Over years these pile up an *enormous*
number of clips:

| Site | # of video clips |
|---|---|
| DJEKE (Djéké Triangle) | **14,780** |
| GTAP | 54,830 |
| MONDIKA | 25,428 |
| **Total** | **~100,000+** |

No human can watch 100,000 clips. So the lab wants software that watches the
video *for* them and answers:

- **Is there an animal in this clip, and where?** (detection)
- **What species is it?** (gorilla, chimp, duiker, elephant…)
- **How many individuals?** (counting)
- **How far apart are they?** (social spacing → needs 3D/depth)
- Later: **what are they doing?** (pose → behavior) and **which specific known
  individual is it?** (re-identification)

## The software: SIDEKIC

**SIDEKIC is that software pipeline** — a chain of state-of-the-art AI models,
each handling one stage. A video clip flows through it like an assembly line:

```
 Video clip
    │
    ▼
[1] SAM 3.1        →  Find & outline animals, track them frame-to-frame   ← (YOUR main area)
    │
    ▼
[2] Species ID     →  Label each animal with a species
    │
    ▼
[3] Depth Anything 3 → Build 3D depth so you can measure distances        ← (Nick owns this)
    │
    ▼
[4] Pose (BBoxMaskPose) → 17 body keypoints, "what pose is it in"          ← (Noah owns this)
    │
    ▼
[5] Behavior / Individual ID  →  (future; probably out of scope this summer)
    │
    ▼
 Structured results (COCO JSON files)
```

**Your role (you + Claude, "Brian and Claude"):** you build and improve the
**front of the pipeline** — making detection trustworthy, making counting
possible, and building the tools to *measure* whether any of it works. Teammates
own later stages:

- **Nick (NBStephens, the lead)** — depth / distance / geolocation track.
- **Noah** — pose track (Docker-based).
- **Danni** — code reviewer (code owner).

You stay upstream and feed them clean detections.

---

# 2. The key strategy — *why* we're doing what we're doing

This is the single most important idea. Once it clicks, everything makes sense.

When we audited the project we found: there are **14,780 DJEKE clips but ZERO
labels and ZERO metrics on disk.** "Labels" = a human-made answer key (*"clip
DJK042 contains 2 gorillas at these locations"*). **Without an answer key you
cannot tell whether the AI is doing well or terribly. There's no ruler.**

So the roadmap's headline decision:

> **Do NOT train or fine-tune any AI models first. Build the *ruler* first** —
> the measurement system (an "eval harness") and a small human-labeled
> benchmark. Train later, and only where the ruler proves a real gap.

Catchphrase: **"Build the ruler before forging swords."**

The biggest open question for the whole summer:

> **Do expert species/count labels exist anywhere?** We searched the disk and
> found *none* for DJEKE. If they exist off-disk → this becomes a "compare our
> AI to existing labels" summer. If they truly don't → this becomes an
> **"annotation-first"** summer (humans label a sample first). **Still must ask
> the team.** *(As of end-of-session 2026-06-11, Brian was getting new info from
> the tech lead — see Section 7.)*

---

# 3. Where we left off (verified live against GitHub this session)

Repo: `CongoApe-SIDEKIC/SIDEKIC`. Current branch checked out: `brian/detection-baseline`.

| PR | Status | What it did |
|---|---|---|
| #1 | ✅ Merged | Apple Silicon (Mac) install support + a workflow doc |
| #2 | ✅ Merged | **Cluster support** — made SIDEKIC run on RIS Compute2 (patches, ffmpeg, GPU job scripts) |
| #3 | ✅ Merged | CODEOWNERS file (review rules) |
| #4 | ✅ Merged | **The track_id fix** — the high-value ~1-day change |
| #5 | 🟡 **Open** | Docs/polish follow-up to #4 (small review nits) |
| #6 | 🟡 **Open** | **The detection-quality baseline** — our most recent work (current branch) |

## The two pieces worth understanding deeply

### (a) The track_id fix (PR #4 — merged)
SAM 3.1 already computes a stable **track ID** for each animal it follows across
frames (so it knows "the gorilla in frame 5 is the same as in frame 6"). The old
code **threw that ID away** before saving. We fixed the wrapper to *keep* it and
write it into the output. Why it matters: without a per-animal ID you can't count
unique individuals, measure per-animal distance, attach a pose to the right
animal, or re-identify anyone. **This one ~1-day change unblocks counting,
distance, pose, and re-ID all at once** — the highest-ROI change in the roadmap.

### (b) The detection-quality baseline (PR #6 — open, our latest)
The "ruler, part one." We wrote an analysis tool (`src/sidekic/eval/`) that reads
the detector's output and measures **how it behaves** — without needing labels.
We ran SAM 3.1 over a sampled **60 clips across 20 camera stations** on an H100
GPU (SLURM job 1463344, 32 min), then analyzed. Findings:

1. **60.8% of clips produce *zero* detections** (77% of all frames blank). Camera
   traps do fire on empty scenes — but *without labels we can't yet tell
   "genuinely empty" from "the AI missed the animal."* **This ambiguity is
   exactly what annotation would resolve → strongest argument for annotation-first.**
2. **One prompt does all the work; 3 are dead.** The prompt `animal` produced 71%
   of all detections. The prompts `gorilla`, `antelope`, `duiker` **never fired
   once** — candidates to drop/rethink.
3. **~26% of active frames double-count** — `chimpanzee`, `ape`, `primate` fire on
   the *same* animal at once, so raw counts over-count individuals.
4. **Tracking works** — thanks to PR #4, tracks are stable (no 1-frame flickers).
5. **A GPU memory leak blocks scaling** — 9 of 60 clips crashed with "out of
   memory" because the batch script rebuilds the model for every clip and never
   frees the old one. Must fix before running thousands of clips.

---

# 4. What's next (options + recommendation)

From the baseline report's "recommended follow-ups" + the roadmap's week-1 list:

1. **⭐ Stand up FiftyOne (recommended)** — a visual tool to actually *see* the
   detections on the video frames, and to *annotate* (have a human draw the
   answer key). This is the **critical path** (annotation) AND the best way to
   make the data concrete. Not blocked on the team. Plumbing already proven:
   FiftyOne 1.17 installed; our `track_id` field confirmed to round-trip cleanly.
   Remaining work = write a small loader in the (currently empty) `src/sidekic/ui/`.
2. **Prune prompts + add dedup, re-measure** — drop the 3 dead prompts, merge the
   overlapping primate prompts, add IoU+track dedup so detections become
   individual counts; re-measure on a small sample. Quick, self-contained wins.
3. **Fix the GPU memory leak** — separate PR (touches Noah's batch script): load
   the model once + free VRAM, so we can scale to thousands of clips.
4. **Ask the team about labels** — the fork that decides the whole summer.

**Why FiftyOne first:** it does double duty — the annotation tool the whole
"build the ruler" strategy depends on, *and* the fastest way to literally look at
what the detector is doing.

---

# 5. Glossary — every term and tech, in plain language

## The cluster / where the code runs
- **RIS Compute2** — WashU's shared supercomputer. You *request* time on it.
- **SLURM** — the cluster's "line manager." Submit a job; it finds a free
  machine, runs it, returns the output.
- **Login node** (`c2-login-001`) — the machine you land on. A *waiting room*:
  fine for editing files, git, installing packages, but it has **no GPU**.
  **Rule: never run heavy AI work here.**
- **Compute node** — a powerful machine SLURM gives you for the real work; this
  is where the GPU lives.
- **`srun`** = "give me a machine and run this *now*, interactively." **`sbatch`**
  = "queue this job script, run it when a machine frees up." `squeue --me` =
  "show my jobs."
- **partition** (`-p general-gpu`) — a pool of machines. You use `general-gpu`;
  the `workshop` pool rejects you (not in that Unix group).
- **GPU** — the chip that makes AI fast. **H100** = the top-tier NVIDIA GPU here.
  **VRAM** = the GPU's own memory (80 GB on an H100). Run out → **"CUDA OOM"
  (out of memory)** → the crash that hit 9 of our clips.
- **VRAM leak** — a bug where memory is used but never returned, so it fills up
  until OOM. Our scaling blocker.

## The AI models
- **SAM 3.1** ("Segment Anything Model 3.1", by Meta) — stage 1. Give it **text
  prompts** (literally words like `"gorilla"`, `"animal"`) and it finds and
  outlines matching objects in the video.
  - **Detection** = box around an object. **Segmentation** = exact outline (pixel
    mask). **Tracking** = following the same object across frames.
  - **prompt** = one of the words you ask it to look for. Our config used 9:
    `animal, monkey, chimpanzee, ape, primate, elephant, antelope, duiker, gorilla`.
  - **track_id** = the stable ID SAM gives each tracked individual (saved by PR #4).
- **Depth Anything 3** (ByteDance) — turns flat 2D video into a 3D depth map, to
  measure real-world distances in meters. Nick's lane.
- **Pose / BBoxMaskPose** — finds 17 body **keypoints** (shoulders, hips, etc.)
  to describe posture. Noah's lane. Runs in Docker (a sealed software box).

## Measurement vocabulary (the "ruler")
- **Label / ground truth / annotation** — the human-made answer key. We have
  **none** yet for DJEKE. This is the gap.
- **eval harness** — code that scores the pipeline's output. Our baseline tool is
  the first piece.
- **benchmark** — a fixed, labeled test set you measure against every time.
- **hold-out** — a slice of data set aside, never trained on, used only for
  honest testing.
- **precision / recall** — precision = "of what the AI flagged, how much was
  right?"; recall = "of the real animals, how many did the AI catch?" **Needs
  labels** — which is why our baseline is "label-free" and can only describe
  *behavior*, not correctness.
- **blank rate** — fraction of frames/clips with zero detections (our 77% / 60.8%).
- **IoU** ("Intersection over Union") — a 0-to-1 overlap score for two boxes.
  0 = no overlap, 1 = identical. IoU ≥ 0.5 = a **double-count** (two prompts
  boxing the same animal).
- **COCO format** — the standard JSON format for storing detections (boxes,
  categories, IDs). SIDEKIC writes COCO; FiftyOne reads COCO.
- **FiftyOne** — open-source app to visually browse images/video with detections
  overlaid, and to annotate. Our viewer + labeling tool.

## Git / GitHub workflow
- **branch** — your private copy of the code to work on without disturbing `main`.
- **commit** — one saved change with a message. (Yours always credit both you and
  Claude — your standing rule.)
- **PR (pull request)** — a request to merge your branch into `main`, reviewed
  first.
- **CODEOWNERS / ruleset** — rules requiring review before merging. Side effect
  here: you can't push extra commits to an *existing* PR branch, so we stack a
  new branch instead (why #5 is separate from #4).

## Environment / Python
- **uv** — a fast Python package manager. **venv** — an isolated Python install
  for this project. **torch (PyTorch)** — the core AI library. **CUDA / cu124** —
  the GPU driver layer; our torch must be the **cu124** build to match the
  cluster's driver.
- ⚠️ **Hard rule: never run `uv run` in this repo** — it secretly upgrades torch
  and breaks the GPU. Always use `.venv/bin/python` directly.

---

# 6. A real code block from our work, explained

From `src/sidekic/eval/detection_baseline.py` — the function behind the
"double-count" finding:

```python
def iou_xywh(a: list[float], b: list[float]) -> float:
    """IoU of two boxes in COCO [x, y, w, h] format."""
    ax1, ay1, aw, ah = a          # box A: top-left corner (x,y) + width + height
    bx1, by1, bw, bh = b          # box B: same
    ax2, ay2 = ax1 + aw, ay1 + ah # compute A's bottom-right corner
    bx2, by2 = bx1 + bw, by1 + bh # compute B's bottom-right corner
    ix1, iy1 = max(ax1, bx1), max(ay1, by1)  # top-left of the OVERLAP region
    ix2, iy2 = min(ax2, bx2), min(ay2, by2)  # bottom-right of the OVERLAP region
```

**In words:** each detection box is stored as `[x, y, width, height]`. To measure
how much two boxes overlap, find the rectangle where they intersect (its corners
are `max` of the two top-lefts and `min` of the two bottom-rights), compute that
overlap area, and divide by the *combined* area of both boxes. That ratio is
**IoU** (0 to 1). The rest of the function finishes the area math and returns the
number.

**Why it exists:** when IoU ≥ 0.5, two prompts are almost certainly boxing the
*same* animal. Counting how often that happens gave us the **"26% of active
frames double-count"** finding. This tiny geometry function is the literal engine
behind one of our headline numbers.

The file's top comment states the philosophy:

> *"The SIDEKIC roadmap calls for an eval harness first: there are no expert
> species/count labels on disk yet, so we cannot compute precision/recall. What
> we can do today is measure how the detector behaves… Those unsupervised signals
> are the first quantitative read on the pipeline and tell us what is worth
> annotating."*

That sentence *is* the strategy from Section 2, written into the code.

---

# 7. Open decision (pending as of end-of-session 2026-06-11)

Claude offered four next-step directions (Section 4). Brian replied that he had
**new info coming from the tech lead** and would relay it before choosing.

**Most decision-relevant thing to relay:** anything about **labels / ground
truth** — do human-coded species/count (or distance / individual-ID) data exist
anywhere off-disk? That's the fork (annotation-first vs. comparison-study).
Secondary but useful: anything about depth, pose, scaling, deadlines, datasets,
or a change in priorities.

---

# Quick recap

- **The project:** automatically watch the Sanz Lab's ~100k Congo ape
  camera-trap videos and pull out species, counts, distances, etc.
- **Your part:** the front of the pipeline + the measurement tools, feeding Nick
  (depth) and Noah (pose).
- **The strategy:** build the ruler before training anything; the open question
  is whether human labels exist.
- **Done:** cluster works, track_id saved (PR #4), first detection baseline
  (PR #6 open).
- **Findings:** 60.8% blank clips, 3 dead prompts, 26% double-counting, tracking
  works, a memory leak blocks scaling.
- **Next:** recommended = FiftyOne (see + annotate) — pending tech-lead info.

---

# Reference: key paths & how to pick back up

- **Project root:** `/storage1/fs1/crickette.sanz/Active/DTRC_2026/SIDEKIC`
- **Data root:** `/storage1/fs1/crickette.sanz/Active/DTRC_2026`
- **Our eval code:** `src/sidekic/eval/{detection_baseline.py, sample_clips.py}`
- **Baseline report:** `docs/detection_baseline_djeke_v1.{md,json}`
- **Roadmap:** `~/SIDEKIC_ROADMAP.md` · **Cluster guides:** `~/RIS_HANDOFF.md`,
  `~/RIS_SETUP.md`
- **Resume this chat:** `claude --resume` (transcripts in `~/.claude/projects/`)
- **GPU rule:** never the login node — `srun -A compute2-workshop -p general-gpu
  --gpus=1 …`; never `uv run` (use `.venv/bin/python`).
