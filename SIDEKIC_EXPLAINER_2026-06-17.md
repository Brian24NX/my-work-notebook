# SIDEKIC Explainer — Wiring & Counting (PR #11 + #12), in plain words (2026-06-17)

> **What this is.** A from-scratch, plain-language explainer of the jargon in
> yesterday's work (PR #11 = MegaDetector→SAM 3 wiring; PR #12 = cross-frame
> counting): *box-prompt wiring, frame / per-frame, seed+propagate, cross-frame
> IoU track linking* — with analogies, the comparison table decoded, and the
> full pipeline.
>
> **Where it lives.** `/home/zhou.n/SIDEKIC_EXPLAINER_2026-06-17.md` — on the RIS
> cluster (permanent, survives closing your laptop). Companions:
> `SIDEKIC_EXPLAINER_2026-06-11.md` (whole project) and `…_2026-06-16.md`
> (detection terms). Resume the chat with `claude --resume`.

---

## The setup (so the terms have a home)

A video clip is a flipbook of still images called **frames**. We look at ~26
frames spread across each clip.

```
  one clip  =  [frame1][frame2][frame3] ........ [frame26]
                 🦌      🦌      🦌                  🦌
```

Two AI models, with different jobs:

```
  MegaDetector = the SPOTTER  → draws a BOX around each animal: "it's HERE" 📦
  SAM 3        = the ARTIST   → traces the exact OUTLINE of the animal ✏️
                               (and can try to FOLLOW it frame to frame)
```

Goal of the whole pipeline: **video → "how many animals, what species, how far."**
The foundation is reliably *finding and outlining* the animals in every frame.

---

## Term 1: "box-prompt wiring"  (PR #11)

SAM 3 must be **told where to look**. Two ways to tell it:

```
  TEXT prompt:  "find a gorilla"                 (words)  ← the old way
  BOX prompt:   "outline what's in THIS box" 📦  (a box)  ← the new way
```

**"Wiring"** = connecting two parts so one feeds the other. So **box-prompt
wiring** = the plumbing that hands **MegaDetector's boxes** to **SAM 3 as
prompts**:

```
   MegaDetector 📦 ──(hand the box over)──► SAM 3 ✏️ "outline what's in this box"
```

**Analogy:** MegaDetector is a spotter who points *"animal — right there!"*; SAM 3
is the artist who traces whatever's at that spot. The wiring is the hand-off.

Why: MegaDetector is ~2× better at *finding* animals (esp. tiny duikers) than
SAM 3's word search — so MegaDetector finds, SAM 3 just outlines.

---

## Term 2: "per-frame" vs "seed + propagate"  (PR #11)

Two ways to run SAM 3 across the 26 frames:

### Way A — "seed + propagate" (let SAM 3 *track*)
```
  frame 1:  MegaDetector points → SAM 3 grabs it    ✅   ← "SEED" (point once)
  frame 2:  SAM 3 follows it on its own             ✅   ┐
  frame 3:  SAM 3 follows it on its own             ✅   │ "PROPAGATE"
  frame 4:  SAM 3 LOST it (tiny duiker, night blur) ❌   │ (follows by itself)
  frame 5+: gone                                    ❌   ┘
```
- **seed** = point at the animal once, in the first frame.
- **propagate** = SAM 3 then follows that animal forward by itself (= tracking).
- **Problem we found:** SAM 3 loses small, camouflaged animals after a few frames.

**Analogy:** spot a friend once, then keep your eyes locked on them in the crowd —
easy for a tall friend in red, you lose a short friend in camo within seconds.

### Way B — "per-frame" (re-point *every* frame)
```
  frame 1:  MegaDetector points → SAM 3 outlines  ✅
  frame 2:  MegaDetector points → SAM 3 outlines  ✅
  frame 3:  MegaDetector points → SAM 3 outlines  ✅
  ...every frame, fresh
```
- **per-frame** = treat each frame independently; MegaDetector re-finds the
  animal every frame, SAM 3 just outlines it. No relying on SAM 3 to follow.

**Analogy:** instead of tracking your friend, take a fresh photo each second and
re-spot whoever's in it.

---

## The comparison table, decoded

Numbers are **"animal handled in X of 26 frames"** (higher = better).

```
                     MegaDetector    SAM 3            SAM 3
                     FINDS it        seed+propagate   per-frame
  ─────────────────────────────────────────────────────────────────
  elephant           21/26           3/26  ❌lost      21/26  ✅ all back
  DJK001_0432        20/26           0/26  ❌lost all   16/26  ✅ mostly back
  hard duiker         8/26           3/26              4/26
```

Read the **elephant** row → : MegaDetector found it 21 times → SAM 3 *tracking*
held only 3 (lost it!) → SAM 3 *per-frame* got all 21 back.

**Verdict (PR #11):** `seed+propagate` fails (SAM 3 can't track these animals);
`per-frame` works (it outlines whatever MegaDetector finds). ➡️ **Use per-frame.**

The **duiker** is the honest hard case: MegaDetector itself only found it 8/26
(tiny + camouflaged), and SAM 3 outlined 4 of those 8. Hard at every step.

---

## Term 3: "cross-frame IoU track linking"  (PR #12)

Per-frame leaves a gap: each frame's boxes are **independent** — we don't know if
frame 5's animal is the *same individual* as frame 6's. To **count individuals**,
we must connect the same animal across frames.

- **cross-frame** = across different frames (frame-to-frame).
- **track** = one individual's path through the clip (its box frame5→6→7…).
- **IoU** = how much two boxes overlap (0 = none, 1 = identical). High overlap
  between consecutive frames ⇒ same animal (barely moved).

**cross-frame IoU track linking** = link boxes across frames when they overlap a
lot → each animal becomes one track → **count tracks = count individuals.**

```
  frame5 [box A] ──overlap high──► frame6 [box A'] ──► frame7 [box A'']
         = ONE track = ONE individual 🦌
  frame5 [box B] ──► frame6 [box B'] ──► ...
         = a SECOND track 🦌
  → 2 tracks → "2 animals in this clip"
```

**Analogy:** photos every second at a party; circle each person in each photo.
Linking = drawing lines for "this circle = same person as that one" so you count
**distinct** people, not total circles. We validated counts land within ±1 of the
experts ~79% of the time.

---

## The whole pipeline, end to end

```
   📹  video clip
        │
        ▼  MegaDetector — FIND animals (a box each), every frame   ◄ PR #10, #11
        │     "animal here 📦"  (91% — 2× SAM 3)
        ▼  SAM 3 — OUTLINE each box, PER-FRAME (not tracking)      ◄ PR #11
        │     ✏️ exact shape
        ▼  IoU LINKING — connect boxes across frames into tracks   ◄ PR #12
        │     🦌→🦌→🦌 = one individual
        ▼  COUNT tracks = # individuals (±1 of experts ~79%)
        ▼
   📊  "this clip: 2 blue duikers"  →  the science the lab wants
```

---

## What it means for the future

1. The **find → outline → count** chain is proven on small samples (find 91%,
   per-frame outline ≈ MegaDetector, count within ±1).
2. **Now (scaling):** confirm those numbers across ~400 clips, not ~50.
3. **Then:** put this chain into the real pipeline code ("productionize"), add
   **species ID** (gorilla vs duiker), and **distance** (Nick's depth) — the
   lab's actual deliverables.

**One-line story:** *we found the best way to wire MegaDetector (the finder) into
SAM 3 (the outliner) — re-detect every frame ("per-frame") instead of trusting
SAM 3 to track ("seed+propagate") — then link the per-frame detections across
frames to count individual animals.*

---

## PRs that did this
- **#10** MegaDetector benchmark (2× the recall of SAM 3).
- **#11** MegaDetector → SAM 3 box-prompt wiring; per-frame beats seed+propagate.
- **#12** cross-frame IoU track linking → individual counts (±1 of experts ~79%).
