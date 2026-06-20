# SIDEKIC — How Counting Works (PR #13 & #14), and Why Species-ID Came Next

> The missing logical bridge: detection → **counting** → species-ID. This is the
> "what happened in PR #13 and #14, and why that led to PR #15" story, in plain
> language with diagrams. Companion to `SIDEKIC_EXPLAINER_SPECIES_2026-06-20.md`
> and `SIDEKIC_METRICS_EXPLAINED.md`. Lives at
> `~/SIDEKIC_EXPLAINER_COUNTING_2026-06-20.md`.

---

## 0. The one-paragraph map

```
   VIDEO ─▶ DETECTION ─▶ COUNTING ─▶ SPECIES ─▶ DISTANCE
            (find each    (how many   (name it)  (how far)
             animal)       individuals?)
            ✅ MegaDetector  ▲▲▲           ❌ ← became PR #15
                          PR #12 built it
                          PR #13 stress-tested it (found a bug)
                          PR #14 fixed the bug
```

Detection finds animals **in each frame**. But "boxes in frames" is **not** the
same as "how many animals are in this clip." Turning boxes into a head-count is a
separate, surprisingly tricky job — that's **counting**, which is what PR #12–#14
are about. Once counting worked, the only unsolved *upstream* stage left was
**naming the species** → that's why PR #15 started the species track.

---

## 1. Why counting is its own hard problem

The same animal shows up in **many frames**. If you just counted boxes, you'd
count one animal dozens of times.

```
   One blue duiker, standing in front of the camera for a whole clip:

   frame 1   frame 2   frame 3   ...   frame 30
    ┌──┐      ┌──┐      ┌──┐            ┌──┐
    │🦌│      │🦌│      │🦌│    ...     │🦌│      ← 30 boxes
    └──┘      └──┘      └──┘            └──┘
                                                  
   Counting boxes  →  "30 animals"   ✗  WRONG
   The truth       →  "1 animal"     ✓
```

So counting needs a way to say *"all of these boxes are the **same** individual.”*
That grouping is called **tracking** (or **linking**), and each group is a
**"track"** = the path of one individual through the clip.

```
   the count we actually want  =  the number of distinct TRACKS
                               =  the number of individual animals
```

---

## 2. How the counter works: boxes → tracks → count

The method is **IoU linking**. Here's the only piece of jargon you need:

> **IoU = "Intersection over Union"** = a 0-to-1 score for **how much two boxes
> overlap**. 0 = no overlap at all; 1 = the boxes are identical.

```
   no overlap            some overlap          near-identical
   ┌──┐ ┌──┐             ┌──┐                  ┌──┐
   │  │ │  │             │ ┌┼─┐                │▒▒│
   └──┘ └──┘             └─┼┘ │                └──┘
   IoU = 0               └──┘                  IoU ≈ 1
                         IoU ≈ 0.3
```

**The rule:** if a box in one frame overlaps a lot (high IoU) with a box in the
next frame, they're almost certainly the **same animal** → link them into one
track.

```
   frame 1          frame 2          frame 3
   ┌────┐           ┌────┐           ┌────┐
   │ 🦌 │  high IoU  │ 🦌 │  high IoU  │ 🦌 │     all linked → ONE track
   └────┘  ───────▶ └────┘  ───────▶ └────┘
   (animal barely moves between frames, so the boxes overlap)

   COUNT = number of tracks = 1   ✓
```

That's the whole idea: **detect boxes → link overlapping boxes into tracks →
count the tracks.**

---

## 3. The three ways to turn tracks into a count

There's more than one sensible way to produce the final number (details in
`SIDEKIC_METRICS_EXPLAINED.md` §5):

| count definition | what it means | character |
|---|---|---|
| **max_per_frame** | the most boxes seen in any *single* frame | simple, robust; slightly undercounts groups |
| **n_tracks** | the number of linked tracks | the "real" idea, but sensitive to bugs |
| **n_tracks_min2** | tracks seen in **≥2** frames (ignore 1-frame blips) | best after the PR #14 fix |

Keep these three in mind — the PR #13/#14 story is mostly about getting
**n_tracks** to behave.

---

## 4. Reading the scorecard: MAE, bias, and "within ±1" (the clear version)

For each clip we compare **our count** to the **expert's true count**:

```
   error  =  our_count  −  expert_count
             +  = we counted too many
             −  = we counted too few
             0  = exactly right
```

Three ways to summarize the errors over all the clips:

- **MAE (Mean Absolute Error)** = the average *size* of the error (ignore + / −).
  "MAE 0.83" = on a typical clip we're off by about 0.83 animals. **Lower = better.**
- **Bias** = the average error *keeping the sign* — which way we lean.
  + = we tend to overcount, − = undercount, ~0 = balanced.
- **within ±1** = ⬇ explained right below, since this is the one you asked about.

### "within ±1" — the part that was confusing

**Read "±1" out loud as "plus or minus one."** It means **off by at most one
animal, in either direction.** Crucially, it **includes the exact matches** — it
is *not* "off by exactly 1." A clip is "within ±1" if our number is:

```
   the true number          (error  0)   ✓
   one TOO MANY             (error +1)   ✓
   one TOO FEW             (error −1)   ✓
   anything bigger      (error ±2, ±3…)  ✗
```

So "within ±1" = the fraction of clips whose error is **−1, 0, or +1**.

**Worked example — 5 clips:**

```
  clip │ expert (true) │ ours │ error = ours − true │ within ±1?
  ─────┼───────────────┼──────┼─────────────────────┼───────────
   A   │       2       │  2   │         0           │  ✓ exact
   B   │       1       │  2   │        +1           │  ✓ one too many
   C   │       3       │  2   │        −1           │  ✓ one too few
   D   │       4       │  1   │        −3           │  ✗ off by 3
   E   │       1       │  1   │         0           │  ✓ exact
```

4 of the 5 clips are within ±1, so this set is **"80% within ±1."**

On the number line, "within ±1" is just the middle three buckets:

```
   error:   −3    −2    −1     0    +1    +2    +3
             ✗     ✗     ✓     ✓     ✓     ✗     ✗
                         └──── within ±1 ────┘
                      (off by one or less, both directions)
```

**Why ±1 and not exact?** Counting wild animals from ~30 sampled frames is genuinely
hard, and for the lab's goal — **abundance trends** (is this species common or rare
at this site over time?) — being within one individual is plenty accurate. "Exact"
would be a needlessly harsh bar. So "79–88% within ±1" reads as **"~4 out of 5
clips, our count is essentially correct."**

---

## 5. PR #13 — scaling the test (small sample → 400 clips)

PR #12 first validated counting on a **tiny sample** (34 clips). A number from 34
clips might just be luck. **PR #13's job was to make the result trustworthy** by
testing at scale.

```
   PR #12:  34 clips   →  "looks like it works"   (could be luck)
   PR #13:  400 clips across 40 camera stations  →  "it really works" (or not)
```

**What PR #13 did:** ran the full detect→count pipeline on **400 clips** spanning
**40 stations**, then scored both detection recall and counting against the expert
GT, all in one combined report.

**What held up (great news):**
- Detection recall **92.2%** (basically unchanged from the small sample) → finding
  animals is robustly solved.
- `max_per_frame` counting: **MAE 0.84, 88% within ±1** → the simple counter is
  solid.

**What broke (the important discovery):**
- The **raw `n_tracks`** count got **worse** at scale: **MAE 3.91** (overcounting
  badly). The cause is a bug called **fragmentation** — explained next. *This
  discovery is exactly what motivated PR #14.*

```
   PR #13 counting results:
     max_per_frame   MAE 0.84   ✓ good
     n_tracks (raw)  MAE 3.91   ✗ overcounts → something's wrong → fix in #14
```

---

## 6. The fragmentation bug (why PR #14 was needed)

We don't analyze *every* frame (too slow) — we **sample** frames spread across the
clip. But a **moving** animal travels a long way between sampled frames, so its box
in one sampled frame **doesn't overlap** its box in the next.

```
   sampled frame 1     sampled frame 2     sampled frame 3
   (t = 0s)            (t ≈ 1s)            (t ≈ 2s)
   ┌──┐
   │🦌│····· walking ·····▶
   └──┘                ┌──┐
                       │🦌│····· walking ·····▶
                       └──┘                ┌──┐
                                           │🦌│
                                           └──┘
   The boxes never overlap  →  IoU = 0  →  the linker thinks:
        "new animal!"  "new animal!"  "new animal!"
   →  ONE animal is split ("fragmented") into 3 phantom tracks  →  OVERCOUNT
```

That's **fragmentation**: one real individual shattered into several tracks because
widely-spaced sampled frames don't overlap. It's why raw `n_tracks` ballooned to
MAE 3.91 at scale.

---

## 7. PR #14 — the center-distance fix

**The fix is intuitive:** even if two boxes don't overlap, if their **centers** are
**close together** (within a few box-widths), they're probably still the same
animal — so link them anyway. This stitches the fragments back into one track.

```
   BEFORE #14 (overlap/IoU only):          AFTER #14 (+ center-distance):
   ┌──┐                                     ┌──┐
   │🦌│   ┌──┐   ┌──┐                        │🦌│┄┄┄┄┐┄┄┄┄┐     centers are
   └──┘   │🦌│   │🦌│                         └──┘  ┌──┐  ┌──┐    close → linked
          └──┘   └──┘                              │🦌│  │🦌│
   IoU=0 between them                              └──┘  └──┘
   → 3 separate tracks  ✗                   → 1 track  ✓
```

**The payoff (measured on the counting test set):**

```
   metric            before #14      after #14
   n_tracks (raw)    MAE 3.91   ──▶  MAE 1.10     (huge improvement)
   n_tracks_min2     MAE 1.53   ──▶  MAE 0.83     ← now the BEST counter,
                                                    ~85% within ±1, ~unbiased
```

After PR #14, **counting is genuinely good**: off by less than one animal on
average, right-or-off-by-one ~85% of the time, and no systematic over/undercount.
That closed out the counting job.

---

## 8. The bridge to PR #15 (the logical relation you wanted)

Here's the clean throughline:

```
   VIDEO ─▶ DETECT ─────▶ COUNT ─────▶ SPECIES ─────▶ DISTANCE
            ✅ #6–#11      ✅ #12–#14    ❌ open         (teammate's lane)
            ~92% found     ±1, ~85%      ← PR #15 starts here
```

1. **Detection** got solved first (MegaDetector, ~92%) — PRs up through #11.
2. **Counting** built on detection's boxes — PR #12 created it, **#13 proved it at
   scale and exposed fragmentation, #14 fixed it.** Counting is now ±1-accurate.
3. Notice: **detection and counting never need to know the species.** A "track" is
   just "one individual," whether it's a duiker or a gorilla. So species-ID is a
   **cleanly separable** stage — nothing upstream depends on it.
4. That made **species-ID the obvious next frontier**: it's the last *upstream*
   stage that wasn't solved, and it unlocks the richer questions ("*which* animals,
   not just *how many*"). So **PR #15** began the species track — building the
   species "ruler," then the SpeciesNet baseline, then (now) fine-tuning.

In one sentence: **once we could reliably find animals (#≤11) and count them
(#12–#14), the only thing left to do upstream was name them — and that's PR #15
onward**, which the species explainer picks up.

---

## 9. Glossary (counting terms)

- **Box / bounding box** — the rectangle MegaDetector draws around an animal.
- **Track** — the linked set of boxes that are the same individual across frames;
  one track = one animal.
- **Tracking / linking** — the act of grouping boxes into tracks.
- **IoU (Intersection over Union)** — 0–1 overlap score between two boxes; high IoU
  ⇒ same animal.
- **Center-distance fallback** — PR #14's fix: link boxes whose centers are close
  even when they don't overlap (defeats fragmentation).
- **Fragmentation** — one animal wrongly split into several tracks because sampled
  frames are too far apart to overlap → overcounting.
- **Sampling / sampled frames** — analyzing a spread of frames (e.g. ~30 per clip)
  instead of all of them, for speed.
- **count definitions** — `max_per_frame` (most boxes in one frame), `n_tracks`
  (number of tracks), `n_tracks_min2` (tracks seen in ≥2 frames).
- **error** — our_count − expert_count for a clip (+ over, − under, 0 exact).
- **MAE (Mean Absolute Error)** — average *size* of error; lower is better.
- **bias** — average *signed* error; which way we lean (+over / −under / ~0 even).
- **within ±1** — % of clips off by at most one animal (error −1, 0, or +1),
  *including exact matches.*
- **station** — one camera location (`DJK001`…`DJK100`).
- **recall** — % of truly-present animals/species we successfully detect.
- **PR (Pull Request)** — a reviewable bundle of code changes; numbered by order.

---

*Last updated 2026-06-20. Read this one first, then
`SIDEKIC_EXPLAINER_SPECIES_2026-06-20.md` for the species-ID track (PR #15→now).*
