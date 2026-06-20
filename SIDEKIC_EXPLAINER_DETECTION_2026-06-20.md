# SIDEKIC — How Detection Works (PRs #6–#11): SAM3 → MegaDetector

> The origin story: how we discovered our first detector was failing, proved it,
> and swapped in a better one. This is the **first** chapter of the arc; read it,
> then `SIDEKIC_EXPLAINER_COUNTING_2026-06-20.md` (PR #12–#14), then
> `SIDEKIC_EXPLAINER_SPECIES_2026-06-20.md` (PR #15→now). Lives at
> `~/SIDEKIC_EXPLAINER_DETECTION_2026-06-20.md`.

---

## 0. The one-paragraph map

```
   VIDEO ─▶ DETECT ─────────────▶ COUNT ─▶ SPECIES ─▶ DISTANCE
            ▲▲▲▲▲▲                 (#12-14)  (#15→)
            this whole doc
   #6  build a way to MEASURE detection
   #7  measure it → SAM3 finds only 43% of animals  (the bombshell)
   #8  build a VIEWER to see the misses
   #9  try to fix SAM3 by tuning → proven impossible (it's a domain problem)
   #10 swap in MegaDetector → finds 91%  (more than double)  ← the pivot
   #11 rewire: MegaDetector finds, SAM3 outlines  ← the detection front-end
```

The project began with **one** AI (SAM 3.1) meant to do everything. We built tools
to grade it (#6), and once expert labels arrived we found it only spotted **43%**
of animals — missing **two-thirds** of the common duikers (#7). We built a viewer
to see the misses (#8), tried to tune SAM3 and proved tuning couldn't save it (#9),
so we brought in **MegaDetector**, a camera-trap specialist, which found **91%**
(#10), and rewired things so MegaDetector finds the animals and SAM3 just outlines
them (#11). That rewired detector is what the counting stage (#12) builds on.

---

## 1. The cast — SAM3 vs MegaDetector, and three sub-jobs

Detection actually bundles **three different sub-jobs**, and it matters which tool
does which:

```
   DETECT  = WHERE is the animal?      → draw a box      ┌──────┐
   SEGMENT = trace its exact outline   → a pixel "mask"  │ ╭🦌╮ │  box = DETECT
   TRACK   = follow the SAME animal    → give it an ID   │ (██) │  outline = SEGMENT
             across frames                                └──────┘
```

**The two tools:**

| | **SAM 3.1** (the original) | **MegaDetector** (the specialist) |
|---|---|---|
| What it's for | general-purpose: **detect + segment + track**, anything | **only detects** animals (boxes) |
| How you aim it | **text prompts** ("find the duiker") | nothing — it just knows "animal" |
| Trained on | internet images (mostly **color, daylight**) | **camera-trap** photos, **including IR night** |
| On our IR footage | struggles (out-of-domain) | excellent |

> **SAM = "Segment Anything Model"** (Meta's). **Prompt** = telling SAM in words
> or with a box what to look for. **MegaDetector** = a free, widely-used model made
> specifically for camera-trap projects.

The plot of #6–#11 is essentially: *we started with the generalist, found it
couldn't see our animals, and brought in the specialist.*

---

## 2. PR #6 — building a way to measure detection at all

**You can't improve what you can't measure.** Before judging SAM3, we needed a
tool to run it on a sample and summarize what came out. PR #6 built that
**baseline analyzer** and ran SAM3 on a **60-clip sample** across 20 camera
stations.

**What it found (warning signs):**
- **60.8% of clips produced ZERO detections** — SAM3 saw nothing in most clips.
- **3 of 9 text prompts never fired at all** — e.g. "gorilla", "antelope",
  "duiker" returned nothing, ever.
- **26% of frames double-counted** (overlapping prompts firing on one animal).
- A **memory bug**: the batch script rebuilt the whole model for every clip and
  never freed the GPU memory (**VRAM**), so it **crashed (OOM)** after ~50 clips.

> **VRAM** = the GPU's memory. **OOM = "Out Of Memory"** = the job ran out of VRAM
> and crashed. **Baseline** = a first measurement you compare everything against.

**The catch:** at this point we had **no expert labels yet**. So "zero detections"
was ambiguous — was the clip *actually empty*, or did SAM3 *miss* an animal? We
couldn't tell the difference. That's what PR #7 resolved.

---

## 3. PR #7 — the bombshell: 42.6% recall

Then the expert **ground-truth labels arrived** (which species are truly in each
video). Now we could compute the metric that matters: **recall.**

> **Recall** = of the animals that are **truly there**, what fraction did we
> **find**? (100% = we caught everything; 50% = we missed half.)

```
   Recall illustration:
     truly-present animals:  🦌 🦌 🦌 🦌 🦌 🦌 🦌 🦌 🦌 🦌   (10)
     SAM3 actually found:    🦌 🦌 🦌 🦌 ✗ ✗ ✗ ✗ ✗ ✗        (4)
     recall = 4/10 = 40%   →  it MISSED 6 of the 10
```

**The result for SAM3:**
- **42.6% recall** — it found fewer than half the animals.
- **0% false-positive rate** — on truly-empty clips it fired 0 times. So SAM3
  **doesn't hallucinate; it just MISSES.** (Quietly missing is arguably worse — you
  don't even know it failed.)
- Worst on the **most common, most important** animals: **blue duiker 31.8%,
  Peter's duiker 14.3%** → we were **losing ~2/3 of the dominant small-duiker
  clips**, while big animals (elephant, chimp) fired fine.

> **False positive (FP)** = firing on something that isn't there (a "false alarm").
> **False negative** = missing something that IS there (a "miss"). SAM3's problem
> was all misses, no false alarms.

This was the bombshell: **the detector at the front of the whole pipeline was
missing most of our animals.** Everything downstream (count, species, distance) is
built on detection — if detection misses the animal, nothing else can recover it.

---

## 4. PR #8 — building the viewer (FiftyOne) to SEE the misses

Numbers tell you *that* something's wrong; you also want to **see** it. PR #8 built
a loader for **FiftyOne** — a visual dashboard for image datasets.

```
   FiftyOne dashboard (conceptually):
   ┌───────────────────────────────────────────────┐
   │ [clip] [clip] [clip] [clip] [clip] [clip] ...  │  scroll thumbnails
   │  boxes drawn on each, GT species labels shown   │
   │  filter:  false_negative == True  ◀── shows ONLY the missed clips
   └───────────────────────────────────────────────┘
```

It overlays the detections, shows the expert species, and adds a **`false_negative`
flag** (animal present + nothing detected) so you can filter straight to the misses
and look at them. This is also the **seed of the lab's eventual dashboard UI** from
the project proposal (browse, filter, correct labels).

---

## 5. PR #9 — trying to fix SAM3 (and proving you can't by tuning)

Before abandoning SAM3, we did our due diligence: **can we fix it with settings?**
Two experiments:

```
   Arm A — lower the "confidence threshold"   (accept less-certain detections)
            conf 0.4 → 0.2   →  RESULT: no change at all (the knob did nothing)

   Arm B — reword the text prompts            (try "small antelope", "forest
            antelope", "deer", etc.)          →  RESULT: +1 clip. Basically nothing.
```

> **Confidence threshold** = how sure the model must be before it reports a
> detection. Lower = more (but riskier) detections. For SAM3 here it was a no-op.

**Verdict: tuning is a dead end.** This isn't a settings problem — it's a
**model/domain problem.** SAM3 was trained on **color, daylight** internet images;
our footage is **grayscale infrared forest at night**. A small camouflaged duiker
in IR clutter is simply *unlike anything SAM3 learned*, so no threshold or prompt
rescues it.

> This is the **exact same villain** — **"out-of-domain / domain shift"** — that
> later shows up in species-ID (SpeciesNet also fails on IR). See the species
> explainer. The lesson repeats: *when a general model is out-of-domain, swap in
> something built for your domain (or train your own).*

That conclusion is what justified changing detectors.

---

## 6. PR #10 — the pivot: MegaDetector more than doubles recall

We brought in **MegaDetector**, which is **purpose-built for camera traps,
including IR night footage**, and ran it on the **same 60 clips** through the
**same recall ruler** — a fair, apples-to-apples test.

```
   SAME clips, SAME metric:
     SAM3 (general)        recall  42.6%   ███████░░░░░░░░░░
     MegaDetector (camera) recall  90.9%   ██████████████████   ← MORE THAN 2×
```

- **Recall 90.9% vs 42.6%** — more than double.
- **Blue duiker 96.3% vs 31.8%** — the duiker problem essentially *solved*.
- **Trade-off:** at a very low threshold MegaDetector had more false alarms (it's
  "trigger-happy"), but unlike SAM3 its threshold **actually works**, so it's
  tunable: at conf 0.5 → **78% recall at 20% FP**; at conf 0.6 → **73% recall at 0%
  FP**. The lab picks the recall/precision balance.

**Verdict: MegaDetector becomes the detection front-end.** This is the pivot the
whole rest of the pipeline rests on.

---

## 7. PR #11 — rewiring: MegaDetector finds, SAM3 outlines

MegaDetector only draws **boxes**. We still wanted **outlines (segmentation)** and
**tracking**. So PR #11 wired the two tools together: **MegaDetector finds the
animals → SAM3 outlines/segments what MegaDetector found.** We tested two ways to
do it:

```
   "SEED + PROPAGATE"                      "PER-FRAME"
   show SAM3 the animal in frame 1,        re-detect with MegaDetector on
   let it follow on its own                EVERY frame, segment each one
   ┌──┐ ┌─?┐ ┌─?┐ ┌─?┐                    ┌──┐ ┌──┐ ┌──┐ ┌──┐
   │🦌│ │  │ │  │ │  │                    │🦌│ │🦌│ │🦌│ │🦌│
   └──┘ └──┘ └──┘ └──┘                    └──┘ └──┘ └──┘ └──┘
   SAM3 LOSES the small IR animal          tracks MegaDetector's ~91%
   after ~3 frames  ✗                      coverage  ✓
```

- **Seed + propagate FAILED** — SAM3's own tracker lost the small IR animal after
  ~3 of 26 frames.
- **Per-frame WON** — re-running MegaDetector each frame matched its full ~91%
  coverage.

**Two consequences that carry forward:**
1. The detection front-end is **"MegaDetector per-frame, then SAM3 segments each
   box."**
2. **SAM3's *tracking* was unusable** on these animals → so to follow animals
   across frames for counting, we needed a *different* method. **That's exactly why
   the counting stage (PR #12) uses IoU linking** instead of SAM3 tracking. (This
   is the handoff into the counting explainer.)

---

## 8. The complete arc (all three chapters)

```
   VIDEO
     │
     ▼  DETECTION   (this doc, PRs #6–#11)
   ┌──────────────────────────────────────────────────────────┐
   │ #6 measure → #7 SAM3 only 43% → #8 viewer → #9 tuning     │
   │ fails (domain problem) → #10 MegaDetector 91% (pivot) →   │
   │ #11 wire MD-per-frame + SAM3-segment                      │
   └──────────────────────────────────────────────────────────┘
     │  ~92% of animals found (and outlined)
     ▼  COUNTING   (PRs #12–#14)
   ┌──────────────────────────────────────────────────────────┐
   │ link boxes into tracks (IoU) → #13 scale test finds       │
   │ fragmentation bug → #14 center-distance fix → ±1 accurate │
   └──────────────────────────────────────────────────────────┘
     │  how many individuals (off by <1, ~85% of the time)
     ▼  SPECIES-ID   (PR #15 → now)
   ┌──────────────────────────────────────────────────────────┐
   │ build the ruler + SpeciesNet baseline → variant test      │
   │ (dead end) → fine-tune our OWN model on IR (running now)   │
   └──────────────────────────────────────────────────────────┘
     │
     ▼  DISTANCE   (teammate Nick's lane)
```

**The single theme that ties it all together:** the recurring enemy is
**"out-of-domain on grayscale IR."** It beat SAM3 at detection — we won by swapping
in a camera-trap-native detector (MegaDetector). It's *also* beating SpeciesNet at
species-ID — and since no off-the-shelf classifier handles IR well, we're trying to
win by **training our own** (the fine-tune, running now). Same villain, same
playbook: **measure it, prove the general tool is out-of-domain, then bring in (or
build) the right-domain tool.**

---

## 9. Glossary

- **SAM 3.1 (Segment Anything Model)** — Meta's general-purpose model; can detect,
  segment, and track via text/box prompts. The project's original detector;
  out-of-domain on IR.
- **MegaDetector** — a model built specifically to find animals in camera-trap
  photos (incl. IR night). Boxes only — no species. Our detection front-end.
- **Prompt (text/box)** — how you tell SAM what to look for (words, or a box).
- **Detect / Segment / Track** — find WHERE (box) / trace the OUTLINE (mask) /
  follow the SAME animal across frames (ID).
- **Recall** — of the animals truly present, the fraction we found. The headline
  detection metric.
- **False positive (FP)** — a false alarm (detected something not there).
- **False negative** — a miss (failed to detect something that's there).
- **Confidence threshold** — how sure the model must be to report a detection;
  lower = more detections. Tunable on MegaDetector, a no-op on SAM3 here.
- **Out-of-domain / domain shift** — feeding a model images unlike its training
  data → it fails. The core reason SAM3 (and later SpeciesNet) struggled on IR.
- **VRAM / OOM** — GPU memory / "out of memory" crash (the PR #6 scaling bug).
- **Baseline** — a first measurement used as the comparison point.
- **Ground truth (GT)** — the expert's correct answers we grade against.
- **FiftyOne** — a visual dashboard for image datasets (PR #8 viewer; dashboard
  seed).
- **false_negative flag** — a marker for "animal present but nothing detected", to
  filter straight to the misses.
- **Seed + propagate vs per-frame** — two ways to wire MD→SAM3; per-frame won
  because SAM3's tracker loses small IR animals.
- **COCO** — the standard file format the detections are saved in (boxes + frames).
- **Station** — one camera location (`DJK001`…`DJK100`).
- **PR (Pull Request)** — a reviewable bundle of code changes; numbered by order.

---

*Last updated 2026-06-20. This is chapter 1 of 3 — continue with the COUNTING then
SPECIES explainers for the full detection → counting → species-ID arc.*
