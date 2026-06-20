# SIDEKIC — The Species-ID Track, Explained (everything since PR #14)

> Plain-language, crystal-clear walkthrough of what we did, what's running now,
> and where we're headed — written so the tech terms make sense on sight.
> Companion to `SIDEKIC_EXPLAINER_*.md` and `SIDEKIC_METRICS_EXPLAINED.md`.
> Lives at `~/SIDEKIC_EXPLAINER_SPECIES_2026-06-20.md` (survives laptop-close).

---

## 0. The 30-second version

The pipeline has three big jobs: **find the animal**, **count the animals**,
**name the species**.

```
  ┌─────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │  VIDEO  │──▶│ 1. DETECTION │──▶│ 2. COUNTING  │──▶│ 3. SPECIES   │
  │ (clip)  │   │ find animals │   │ how many?    │   │ which animal?│
  └─────────┘   └──────────────┘   └──────────────┘   └──────────────┘
                      ✅ SOLVED          ✅ WORKS          ◀── WE ARE HERE
                     (~92% found)      (~±1 animal)        (the hard part)
```

- **Up through PR #14** we finished jobs 1 and 2 (detection + counting).
- **Since PR #14** we've been working *only* on job 3 (species-ID), because
  that's the one still unsolved.
- **The blocker:** the camera footage is **grayscale night-vision**, and the
  best off-the-shelf species AI (SpeciesNet) wasn't trained on that kind of
  image, so it refuses to commit to a species **64% of the time**.
- **What we're doing right now:** training our *own* species AI on *our* night
  footage, to see if a custom model beats the off-the-shelf one. The training
  is running on the cluster as you read this; the answer lands ~3:15 AM.

That's the whole story. The rest of this doc explains every piece of it.

---

## Table of contents

1. [Where PR #14 left us](#1-where-pr-14-left-us)
2. [Why species-ID is the hard one (the "domain" problem)](#2-why-species-id-is-the-hard-one)
3. [Step 1 — building the "ruler" + the SpeciesNet baseline (PR #15)](#3-step-1)
4. [Step 2 — the v4.0.3b experiment (a dead end, ruled out)](#4-step-2)
5. [Step 3 — fine-tuning our own model (what's running now)](#5-step-3)
6. [The scoreboard + what we're waiting for](#6-scoreboard)
7. [What happens next (the decision tree)](#7-whats-next)
8. [PR vs local: what's actually been published](#8-pr-vs-local)
9. [Glossary — every term in one place](#9-glossary)
10. [Where everything lives (files, jobs, branches)](#10-where-things-live)

---

## 1. Where PR #14 left us

A quick recap so the starting line is clear. By **PR #14**, two of the three
pipeline jobs were done and measured:

| Job | How it works | Status at PR #14 |
|---|---|---|
| **1. Detection** | **MegaDetector** (an AI built for camera traps) draws a **box** around each animal in each frame | ✅ **~92% of animals found** |
| **2. Counting** | Link the boxes of the same animal across frames into "tracks", then count tracks | ✅ **within ~1 animal, ~85% of the time** |
| **3. Species-ID** | *Name* the animal in each box (blue duiker? gorilla?) | ❌ not started yet |

> **PR = Pull Request** = a bundle of code changes submitted for teammates to
> review before it joins the official codebase. The number is just creation
> order. PR #14 was the counting/linker improvement.

So after #14, the natural next question was: **can we name the species?** That's
everything this doc covers.

---

## 2. Why species-ID is the hard one

### 2a. What the footage actually looks like

These are **camera-trap clips from the Congo rainforest at night**. The cameras
shoot in **infrared (IR)**, which produces **grayscale (black-and-white)**
images — no color, often grainy, animals half-hidden in dense forest.

```
   A typical DJEKE frame:  grayscale, night, cluttered forest
   ┌───────────────────────────┐
   │  ░░▓▓░ leaves ░▓░░▓░░▓░░  │
   │ ░▓░  ╭────╮ glowing eye   │   ← a small duiker (antelope),
   │ ░░▓  │ 🦌 │ ░░▓░ branches │     gray-on-gray, easy to miss
   │ ▓░░  ╰────╯ ░▓░░▓░░▓░░░░  │
   └───────────────────────────┘
```

### 2b. The key idea: "in-domain" vs "out-of-domain"

AI models learn from examples. Whatever kind of images a model was *trained* on
is its **"domain."**

- Most species AIs were trained on **color, daytime** wildlife photos.
- Our footage is **grayscale, nighttime** IR.

When you feed a model images **unlike anything it trained on**, it's
**"out-of-domain"** — and it gets confused. This is also called **domain shift**.

```
   what SpeciesNet trained on        what we feed it
   ┌──────────────────┐              ┌──────────────────┐
   │  color, daylight │   ≠ domain   │ grayscale, night │
   │  🦓 clear, sharp │ ───shift───▶ │  🦌 gray, grainy │
   └──────────────────┘              └──────────────────┘
        confident                       confused → punts
```

### 2c. What "punting" looks like (the 64% problem)

A smart model, when unsure, won't guess wildly — it **"rolls up"** to a vague
answer like just **"animal"** instead of risking a wrong species. We measured
that the off-the-shelf model does this on **64% of frames**. (It literally
mislabeled an African elephant as an "american black bear" internally, then its
safety check caught the absurdity and downgraded the answer to "animal.")

```
   Frame → SpeciesNet → "blue duiker"   ← committed to a species  (36% of frames)
   Frame → SpeciesNet → "animal"        ← punted / rolled up      (64% of frames)
                          ▲
              this 64% is the whole problem we're attacking
```

**Finding the animal is solved. Naming it on night footage is not.** That gap is
the entire species-ID project.

---

## 3. Step 1 — building the "ruler" + the SpeciesNet baseline (PR #15) {#3-step-1}

Before improving anything, you need a way to *measure* it. You can't tell if a
model is good without a fair test. So Step 1 built two things:

### 3a. The "ruler" — how we score species predictions

The expert ground-truth (GT) labels tell us, for each video, **which species are
present**. Importantly, the lab confirmed the metric is **"presence-in-set"**,
not "top-1":

- **Top-1** would mean: the model must name the *single* main species. (Too strict
  / wrong question here.)
- **Presence-in-set** means: did the model recover **the set of species in the
  video**? A clip can have a duiker *and* a mongoose — we check whether we found
  each one.

```
   Video's true species (GT):  { blue duiker , mongoose }
   Our prediction:             { blue duiker }
                                 ▲              ▲
                            got this one    MISSED this one
   → scored as: 1 correct, 1 missed   (partial credit, measured precisely)
```

We score it with **precision / recall / F1** (defined fully in the glossary, and
in `SIDEKIC_METRICS_EXPLAINED.md`):

- **Precision** = of the species we *named*, how many were really there? (Did we
  make stuff up?)
- **Recall** = of the species really there, how many did we *find*? (Did we miss
  things?)
- **F1** = a single score combining both (high only when *both* are high).
- **Macro-F1** = average the F1 across species, **every species counts equally**
  — so rare animals matter as much as common ones. *This is our headline number.*
- **Micro-F1** = pool every decision together — dominated by the common species.

This ruler is `species_eval` — it's the same yardstick we use for **every** model,
so all comparisons are fair. (This is the part that became **PR #15**.)

### 3b. The baseline — how good is the off-the-shelf model?

We then measured **SpeciesNet** (Google's free, open camera-trap species
classifier) on our clips, **geofenced** to the Congo.

> **Geofence** = tell the model "this photo is from the Republic of Congo (code
> `COG`)" so it only considers species that actually live there — it won't guess
> "kangaroo" in the Congo.

**Result:** macro-F1 **≈ 34%**, and that 64%-roll-up problem. It's *good where it
commits* (common duikers, elephant) but punts far too often on the IR footage.
That gave us a number to beat.

---

## 4. Step 2 — the v4.0.3b experiment (a dead end, ruled out) {#4-step-2}

Before building anything custom, we tried the cheapest possible fix: **does a
different flavor of SpeciesNet do better?** SpeciesNet ships two variants:

```
   v4.0.3a "always-crop"          v4.0.3b "full-image"
   ┌──────────────────┐           ┌──────────────────┐
   │ ░░░╭────╮░░leaves │           │ ░░░╭────╮░░leaves │
   │ ░░ │ 🦌 │ ░░░░░░░ │           │ ░░ │ 🦌 │ ░░░░░░░ │
   │ ░░ ╰────╯ ░branch │           │ ░░ ╰────╯ ░branch │
   └──────────────────┘           └──────────────────┘
     zoom into the box,             classify the WHOLE
     THEN classify  ────▶ better      frame  ────▶ worse
```

We ran both on the identical clips through the identical ruler:

| | v4.0.3a (always-crop) | v4.0.3b (full-image) |
|---|---|---|
| Macro-F1 | **34.8%** | **16.7%** (≈ half) |
| Frames rolled up to "animal" | 64% | **75%** (worse) |
| Blue-duiker recall | 90% | 47% |

**Verdict:** the full-image variant is **decisively worse**. Cropping to the
animal helps; classifying the whole cluttered frame hurts. **Lesson: swapping
model variants is a dead end** — both are out-of-domain on IR. The only lever
left that actually attacks the 64% roll-up is **training our own model on our own
footage.** Which is Step 3.

---

## 5. Step 3 — fine-tuning our own model (what's running now) {#5-step-3}

### 5a. What "fine-tuning" means

You don't train an image AI from zero — that needs millions of images. Instead
you take a model **pre-trained** on a giant generic image collection (**ImageNet**,
14M everyday photos) that already knows edges, textures, shapes, and you
**"fine-tune"** it: keep that general visual knowledge, and nudge it with *your*
specific images so it learns *your* specific task (Congo IR species).

```
   pre-trained model            +  our IR crops      =  fine-tuned model
   (knows "what images          (teaches it "this       (knows OUR species
    look like" in general)       gray blob = duiker")     in OUR domain)
```

Our base model is **EfficientNetV2-S** — a standard, efficient, well-proven image
classifier (a "**backbone**" = the reusable feature-extracting body of the model).

### 5b. The labeling problem — and the clever trick

To train a species classifier, you need **labeled crops**: a picture of *one*
animal with its *correct species name*. But our expert GT is **per-video**, not
per-animal:

```
   GT gives us:        video "DJK001_IMG_0002"  →  { blue duiker }
   Training needs:     this exact crop          →  "blue duiker"
                       (a specific box, specific frame)
```

**The trick — "weak supervision" via single-species clips:**

> If a video's GT lists **exactly ONE species**, then **every** animal box
> MegaDetector finds in that video **must be that species**. Free, clean labels —
> no manual annotation.

```
   Single-species clip, GT = { blue duiker }
   ┌──────────────────────────────────────────────┐
   │ frame 1   frame 2   frame 3   frame 4   ...   │
   │  ╭──╮      ╭──╮       (none)    ╭──╮           │
   │  │🦌│      │🦌│                 │🦌│           │  every box
   │  ╰──╯      ╰──╯                 ╰──╯           │   ▼ labeled
   │  "blue     "blue               "blue          │  "blue duiker"
   │   duiker"   duiker"             duiker"        │   automatically
   └──────────────────────────────────────────────┘
```

Multi-species clips are ambiguous (which box is which?), so we **skip them for
training** (and keep them for testing). The payoff is huge — we measured it:

```
   DJEKE clips by species count:
     blank (no animal)   1,264  ( 9%)
     ★ single-species   13,016  (90%)  ← FREE clean training labels
     multi-species         154  ( 1%)  ← skip for training
```

**90% of all clips give us free labels.** That's why fine-tuning is even feasible.
After filtering to species with enough data, we have **22 trainable species**.

### 5c. The three-stage pipeline (what's actually running)

```
  ┌────────────────────────────────────────────────────────────────────┐
  │  STAGE A — EXTRACT CROPS         (cluster job 1701237, ~3 h)        │
  │  single-species clips ─▶ MegaDetector finds boxes ─▶ cut out each   │
  │  box ─▶ save as a labeled crop (label = the clip's 1 GT species)    │
  │  → ~2,800 clips → tens of thousands of labeled IR crops on disk     │
  └───────────────────────────────┬────────────────────────────────────┘
                                   │ (auto-starts when A succeeds)
                                   ▼
  ┌────────────────────────────────────────────────────────────────────┐
  │  STAGE B — TRAIN                 (cluster job 1701264, ~20 min)     │
  │  feed the crops to EfficientNetV2-S, ~18 passes ("epochs") over the │
  │  data; it learns "this gray shape = blue duiker / gorilla / ..."    │
  │  → saves the best "checkpoint" (the trained model file)             │
  └───────────────────────────────┬────────────────────────────────────┘
                                   │ (auto-starts when B succeeds)
                                   ▼
  ┌────────────────────────────────────────────────────────────────────┐
  │  STAGE C — EVALUATE              (cluster job 1701265, ~10 min)     │
  │  run the trained model on HELD-OUT clips it never saw ─▶ score with │
  │  the same `species_eval` ruler ─▶ macro-F1 vs SpeciesNet's 24.2%    │
  └────────────────────────────────────────────────────────────────────┘
```

These three are **chained** on the cluster with a SLURM **"afterok" dependency** —
each starts automatically only if the previous one *succeeds*. The whole thing
runs unattended (you can close your laptop).

### 5d. The one rule that makes the test honest: no "leakage"

If the model trained on clips from **camera #7** and we also *test* it on camera
#7, it might just be memorizing that camera's background — not really learning the
animal. That's **"data leakage"**, and it makes results look better than they are.

We prevent it with a **station-disjoint split** (a "station" = one camera
location, `DJK001`–`DJK100`):

```
   100 camera stations, split by number so cameras NEVER overlap:
   ┌──────────────────────────────────────────────────────────┐
   │ TRAIN  = 80 stations   (the model learns from these)      │
   │ VAL    = DJK005,015,…  (10 stations, to pick the best     │
   │                         checkpoint during training)        │
   │ TEST   = DJK010,020,…  (10 stations, the FINAL exam —      │
   │                         the model has NEVER seen these)    │
   └──────────────────────────────────────────────────────────┘
```

> **Train / Val / Test** — the three classic data splits.
> **Train** = study material. **Val** ("validation") = practice quizzes to tune
> things mid-training. **Test** = the final exam, touched only once at the end.

### 5e. Handling the "long tail" (class imbalance)

Some species are everywhere (blue duiker: 4,781 clips), some are rare (galago:
~41). If you don't correct for this, the model just learns "always guess blue
duiker." We fix it two ways: **cap** the common species (max 150 clips each so
they don't dominate) and use a **class-weighted loss** (the model is penalized
*more* for getting a rare species wrong). This is standard for **long-tailed**
wildlife data.

### 5f. The fair fight we set up

SpeciesNet's original 34% was measured on a *different* small clip set. To make
the comparison airtight, we re-ran SpeciesNet on the **exact same 297 held-out
test clips** the fine-tuned model will face (we verified the clip lists are
byte-for-byte identical). That gives the real bar:

```
   SAME 297 test clips, SAME ruler:
     SpeciesNet (off-the-shelf) ....... macro-F1  24.2%   ◀ the bar to beat
     Fine-tuned (ours) ................ macro-F1  ??.?%   ◀ lands ~3:15 AM
```

---

## 6. The scoreboard + what we're waiting for {#6-scoreboard}

```
   Model / experiment                         Macro-F1     Note
   ───────────────────────────────────────────────────────────────────
   "always predict the commonest species"        0.8%     useless floor
   SpeciesNet v4.0.3b (full-image)              16.7%     ruled out (worse)
   SpeciesNet v4.0.3a, held-out test set        24.2%     ◀ THE BAR
   SpeciesNet v4.0.3a, baseline-60 set          34.8%     (easier old set)
   Fine-tuned EfficientNetV2-S (ours)            pending   ◀ the answer
   ───────────────────────────────────────────────────────────────────
```

**What we're waiting for:** the fine-tuned number, on the held-out test set, vs
**24.2%**. That single comparison answers "is training our own model worth it?"

---

## 7. What happens next (the decision tree) {#7-whats-next}

```
                    Fine-tuned result lands (~3:15 AM)
                                  │
                ┌─────────────────┴─────────────────┐
                ▼                                     ▼
     CLEAR WIN (e.g. ≫ 24.2%)              TIE / LOSS (≈ or < 24.2%)
                │                                     │
                ▼                                     ▼
   Species-ID is VIABLE.                  Fine-tuning on this data isn't
   Next steps:                            enough. Options:
   • productionize: wire                  • more data / more crops per species
     MD → crop → our classifier           • a stronger backbone
     into the live pipeline               • accept species-ID is hard for now
     (touches teammate Noah's code        AND remember: COUNTING + DISTANCE
      → align with the team first)         DON'T need fine species, so the
   • add more species / more data          core deliverables aren't blocked.
   • write it up for the lab              Either way: we report the finding
                                           (a negative result is still a result).
```

**Big-picture direction (regardless of the number):**

1. **Detection & counting are done** — those deliverables stand.
2. **Species-ID is the research frontier** — tonight tells us how far custom
   training closes the gap.
3. **The lab dashboard** (from the project proposal) eventually consumes all of
   this: detect → count → species → distance, browsable/correctable in a UI.
4. **All the species work is staged locally** and becomes PR(s) once your tech
   lead clears the current review queue (see §8).

---

## 8. PR vs local: what's actually been published {#8-pr-vs-local}

Important for your mental model — **only PR #15 is "out there."** Everything after
is committed **locally only**, on purpose, because you asked to hold new PRs until
your tech lead reviews the existing stack.

```
   PUBLISHED (a real PR, your lead can see it):
     PR #15  species ruler + SpeciesNet baseline   ← you OK'd this one push

   STAGED LOCALLY (committed, NOT pushed — waiting on your go-ahead):
     • v4.0.3b experiment writeup
     • the entire fine-tune pipeline (extract / train / eval scripts + sbatches)
     • the same-set SpeciesNet comparison
     (all on branch  brian/speciesnet-v403b)
```

When you say the word, this becomes one (or a few) clean PRs. Until then it's all
saved and safe, just not submitted.

---

## 9. Glossary — every term in one place {#9-glossary}

**General pipeline**
- **PR (Pull Request)** — a bundle of code changes submitted for review before
  merging. Numbered by creation order.
- **MegaDetector** — an AI that *finds* animals (draws boxes); built for camera
  traps, works great on IR. It does NOT name species.
- **SpeciesNet** — Google's open AI that *names* species. Out-of-domain on our IR.
- **Bounding box / box** — the rectangle around a detected animal.
- **Crop** — the image you get by cutting out just the box region.

**The footage**
- **Camera trap** — a motion-triggered camera in the wild.
- **Station** — one camera location (`DJK001`…`DJK100`).
- **IR / infrared / night-vision** — how the cameras see in the dark → grayscale.
- **Grayscale** — black-and-white (no color).

**Model concepts**
- **Domain** — the kind of images a model was trained on.
- **Out-of-domain / domain shift** — feeding a model images unlike its training →
  it gets confused. (Our core problem.)
- **Roll-up / punt / abstain** — when an unsure model gives a vague answer
  ("animal") instead of a species. Happens on 64% of our frames.
- **Geofence** — restricting predictions to species that live in a given country
  (`COG` = Republic of Congo).
- **Fine-tuning** — taking a pre-trained model and continuing its training on your
  own data so it specializes.
- **Pre-trained** — a model that already learned general visual features (here,
  from **ImageNet**, 14M everyday photos).
- **Backbone** — the reusable feature-extracting body of an image model
  (ours: **EfficientNetV2-S**).
- **Checkpoint** — a saved snapshot of a trained model (a file you can load later).
- **Epoch** — one full pass of the model over all the training data.

**Data & labels**
- **Ground truth (GT)** — the expert's correct answers we measure against.
- **Weak supervision** — getting labels cheaply/indirectly (here: a single-species
  clip labels all its crops for free).
- **Single-species clip** — a video whose GT lists exactly one species.
- **Train / Val / Test** — study set / tuning set / final-exam set.
- **Station-disjoint split** — splitting data by camera so the same camera is never
  in both train and test.
- **Leakage** — accidentally letting test info into training → fake-good results.
- **Long tail / class imbalance** — a few species are very common, many are rare.
- **Class-weighted loss** — penalizing mistakes on rare species more, to counteract
  imbalance.

**Scoring (full detail in `SIDEKIC_METRICS_EXPLAINED.md`)**
- **Presence-in-set** — score whether we recovered the *set* of species in a video.
- **Precision** — of what we predicted, how much was correct (did we invent things?).
- **Recall** — of what was really there, how much we found (did we miss things?).
- **F1** — single score; high only when precision AND recall are both high.
- **Macro-F1** — average F1 across species, all species equal (our headline).
- **Micro-F1** — pooled across all decisions, common species dominate.
- **Support** — how many test videos a species truly appears in.

**Cluster / infrastructure**
- **SLURM** — the cluster's job scheduler (you submit jobs, it runs them on GPUs).
- **Job** — one submitted unit of work (has a number, e.g. `1701237`).
- **afterok dependency** — "only start job B if job A succeeds" → chains our stages.
- **H100** — the (top-tier) NVIDIA GPU our jobs run on.
- **venv (virtual environment)** — an isolated Python setup. We keep three so they
  don't break each other: `md-venv` (MegaDetector + training), `sn-venv`
  (SpeciesNet), `.venv` (the SAM3 pipeline + the scorer).

---

## 10. Where everything lives (files, jobs, branches) {#10-where-things-live}

**Scripts (in the repo, `scripts/` and `slurm/`):**
| File | What it does |
|---|---|
| `analyze_species_trainset.py` | counts the free labels (the 90% / 22-species numbers) |
| `extract_species_crops.py` (+sbatch) | Stage A — make labeled crops |
| `train_species_classifier.py` (+sbatch) | Stage B — train the model |
| `eval_finetune_species.py` (+sbatch) | Stage C — score the model |
| `run_speciesnet_videos.py` (+sbatch) | SpeciesNet on the same test clips (the 24.2% bar) |
| `species_eval.py` | the ruler (presence-in-set scoring) — this is PR #15 |

**Cluster jobs (this run):**
| Job | Stage | State |
|---|---|---|
| `1701237` | extract crops | running ~3 h |
| `1701264` | train | queued (afterok) |
| `1701265` | eval + score | queued (afterok) → **result here** |
| `1701695` | SpeciesNet baseline on test set | ✅ done (24.2%) |

**Outputs land in:** `~/sn_logs/speval-1701265.out` (the headline result),
`~/species_crops/` (the crops), `~/species_model/best.pt` (the trained model).

**Branch:** `brian/speciesnet-v403b` (all species work, committed locally, unpushed).

---

*Last updated 2026-06-20. If a future you is reading this: check
`~/.claude/projects/.../memory/sidekic-status.md` for the very latest state.*
