# SIDEKIC — Detection Work Explainer & Glossary (2026-06-16)

> **What this is.** A plain-language explainer of the terms and the work on the
> *detection* part of SIDEKIC (POC, confidence, sweep, per-frame, plus
> flowcharts of past / present / future). Companion to the broader
> `~/SIDEKIC_EXPLAINER_2026-06-11.md`.
>
> **Where it lives.** `/home/zhou.n/SIDEKIC_EXPLAINER_2026-06-16.md` — on the RIS
> cluster (permanent, backed up). Survives closing your laptop. Re-read anytime;
> resume the chat with `claude --resume`.

---

# Part 1 — The four terms

## 🧪 POC = "Proof of Concept"
A quick, rough version built **only to answer "does this idea even work?"** —
before investing in a polished version.

> **Analogy:** before a real steel bridge, you build a cardboard model to check
> the design. The cardboard bridge is the POC.

**Ours:** wiring MegaDetector → SAM 3 *just enough*, on 3 clips, to test if
MegaDetector's boxes can feed SAM 3. Not the final pipeline — just a test.

## 🎚️ Confidence (and the "threshold")
When a detector finds something it also reports **how sure it is**, 0 (no clue)
to 1 (certain) — the **confidence score**. The **confidence threshold** is the
cutoff: "only keep detections you're at least *this* sure about."

```
   confidence threshold = the "how sure must it be?" dial

   0.2 ◄─────────────────────────────────► 0.8
  LOOSE                                  STRICT
  keep anything ≥20% sure        keep only ≥80% sure
  → finds MORE animals           → fewer false alarms
  → but more false alarms        → but misses more animals
```

> **Analogy:** a spam-filter dial. Low → catches all spam but flags real emails
> (false alarms). High → only obvious spam flagged, but sneaky spam slips through
> (misses). You tune it for balance.

## 🔁 Sweep (the "confidence sweep")
Trying a **range** of settings and measuring each, to find the sweet spot.

> **Analogy:** tasting soup while slowly adding salt — you "sweep" the salt to
> find the best taste.

**Our MegaDetector sweep:**
```
  conf   animals found   false-alarm rate
  0.2  →   91%      ❗ 80%     (catches everything, many false alarms)
  0.5  →   78%        20%     ← balanced sweet spot
  0.6  →   73%      ✅  0%     (zero false alarms, still beats SAM3's 43%)
```

## 🎞️ "Per-frame" job
A video = a stack of still images (**frames**); we sample ~26 per clip. Two ways
to process them:

```
  APPROACH 1: "seed + propagate" (TRACK)     APPROACH 2: "PER-FRAME"
  ───────────────────────────────────       ────────────────────────────
  Frame 0: MegaDetector finds animal         Frame 0: MegaDetector → SAM3 ✓
           → SAM3 segments it ✓              Frame 1: MegaDetector → SAM3 ✓
  Frame 1: SAM3 follows it    ✓              Frame 2: MegaDetector → SAM3 ✓
  Frame 2: SAM3 follows it    ✓              Frame 3: MegaDetector → SAM3 ✓
  Frame 3: SAM3 LOST it       ✗              Frame 4: MegaDetector → SAM3 ✓
  Frame 4: (nothing)          ✗                ...every frame on its own
       → only 3/26 frames 😞                      → ~all frames 🎯

  ✓ knows "same animal" across frames        ✗ no "same animal" link (yet)
  ✗ fragile — loses small IR animals         ✓ robust — matches MegaDetector
```

**Per-frame** = run MegaDetector on *every* frame and let SAM 3 segment each one
fresh, instead of detecting once and *tracking*. More reliable coverage; the
trade-off is losing the "same individual across frames" link (added back later).

---

# Part 2 — The journey in flowcharts

## 🗺️ The big picture — what SIDEKIC does
```
   📹 Camera-trap video
         │
         ▼
   ┌──────────────────┐
   │ 1. DETECTION     │  Find the animals  ◄── ⭐ WE'VE BEEN HERE
   └──────────────────┘     (everything below is useless if this fails)
         │
         ▼
   ┌──────────────────┐
   │ 2. SPECIES ID    │  What is it? (gorilla, duiker…)
   └──────────────────┘
         │
         ▼
   ┌──────────────────┐
   │ 3. DISTANCE      │  How far from camera? (Nick's depth work)
   └──────────────────┘
         │
         ▼
   📊 Counts · species · distances → the science
```

## ⏪ What we HAVE done (the detection investigation)
```
  Expert labels arrived
        │   (finally we can MEASURE quality)
        ▼
  Built the "ruler" (recall scoring) ──────────────► PRs #6, #7
        │
        ▼
  Measured SAM 3  →  42.6% recall   ❌ misses duikers (small IR animals)
        │
        ▼
  Tried tuning it (threshold, prompts) → no help ❌ DEAD END → PR #9
        │
        ▼
  Tried MegaDetector (built for camera traps) → 90.9% ✅ WIN → PR #10
        │                                  (duikers: 32% → 96%)
        ▼
  Decision: MegaDetector finds animals, SAM 3 segments them
```

## ▶️ What we ARE doing
```
  Wiring MegaDetector → SAM 3, tested 2 ways on the H100:

     ┌── seed + propagate (track) ──► FAILED: 3/26 frames; threshold
     │                                doesn't help (SAM3 tracker loses
     │                                the small IR animal)
     │
     └── PER-FRAME seeding ─────────► WORKS: ~80–100% of MegaDetector's
                                      boxes segmented (elephant 21/26)

  → Verdict: use MegaDetector → SAM 3 PER-FRAME
```

## ⏩ What we WILL do
```
  Commit the POC + open a PR (team review)
        │
        ▼
  Plug MegaDetector → SAM 3 (per-frame) in as the real DETECTION + SEGMENT step
        │
        ▼
  Add cross-frame IDs (link per-frame boxes into tracks)
        │
        ▼
  Unlock the downstream science:
        ├─► COUNT individuals
        ├─► SPECIES ID (gorilla vs duiker vs…)
        └─► DISTANCE (feeds Nick's depth work)
```

---

# Part 3 — Latest results (as of 2026-06-16)

**The detector→segmenter wiring is proven.** On the same clips:

| clip | MegaDetector finds | SAM 3 *seed+propagate* | SAM 3 *per-frame* |
|---|---|---|---|
| elephant (DJK006) | 21/26 | 3/26 | **21/26** (100% of MD) |
| DJK001_0432 | 20/26 | 0/26 | **16/26** |
| hard duiker (DJK001_0043) | 8/26 | 3/26 | **4/26** |

- **Seed+propagate is a dead end** here — SAM 3 loses small IR animals after a
  few frames, and the confidence dial can't fix it (SAM 3 emits everything at
  confidence 1.0, so there's nothing to filter).
- **Per-frame works** — SAM 3 segments ~80–100% of what MegaDetector finds. The
  hardest duiker is limited both by MegaDetector (8/26) and SAM 3's segmentation
  of a tiny camouflaged target (4 of those 8).

**One-sentence summary:** *we proved SAM 3 can't reliably find the animals, found
that MegaDetector can (2× better), and wired MegaDetector in front of SAM 3 —
using per-frame segmentation, since SAM 3's tracking isn't usable on this
footage.*

---

# Open PRs (snapshot)
- #6 detection-baseline · #7 recall-harness+GT · #8 FiftyOne dashboard ·
  #9 tuning-is-a-dead-end · #10 MegaDetector benchmark (2× recall).
- POC code (MegaDetector→SAM3 per-frame) staged on branch
  `brian/megadetector-sam3-poc`, ready to commit + PR.
