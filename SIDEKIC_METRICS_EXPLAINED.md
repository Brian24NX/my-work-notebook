# SIDEKIC — Counting Metrics Explained (MAE, bias, ±1, and "PR #12")

> Plain-language guide to the numbers in the counting-validation table, so the
> metrics make sense on sight. Companion to the other `SIDEKIC_EXPLAINER_*.md`
> files. Lives on the cluster (`/home/zhou.n/SIDEKIC_METRICS_EXPLAINED.md`),
> survives closing the laptop.

The table that prompted this:

```
count definition    MAE    bias              within ±1
max_per_frame       1.32   −0.38             79%
n_tracks_min2       1.59   +0.00 (unbiased)  74%
n_tracks (raw)      2.62   +1.44             47%
```

---

## 0. What "PR #12" means

**PR = Pull Request** — a proposed change to the shared code, submitted so
teammates can review it *before* it merges into the official codebase. The number
is just creation order — **#12 = the 12th** in our repo.

**PR #12 specifically = the cross-frame track-linking code** — the piece that
turns "boxes on individual frames" into "this clip has N individual animals."
The table above is the **evidence that PR #12's counting works**, by comparing our
counts to the experts'.

---

## 1. The setup — two counts per clip

For each clip we have:
- **our count** — what the pipeline computed
- **expert count** — the lab's `number_of_individuals` (the truth)

For each clip, the **error**:

```
   error = our_count − expert_count
   +  = we OVERcounted     −  = we UNDERcounted     0 = exactly right
```

All three metrics just summarize these errors over the 34 clips, three different ways.

---

## 2. MAE — Mean Absolute Error  ("how far off, typically?")

Average of the error **sizes** (drop the +/− sign with absolute value).

```
clip │ expert │ our count │ error │ |error|
  A  │   1    │    1      │   0   │   0
  B  │   2    │    1      │  −1   │   1
  C  │   1    │    2      │  +1   │   1
  D  │   3    │    2      │  −1   │   1
  E  │   1    │    1      │   0   │   0
                                MAE = (0+1+1+1+0)/5 = 0.6
```

**MAE 1.32 → on a typical clip we're off by ~1.3 animals.** Lower = better.

---

## 3. Bias — Mean (signed) Error  ("which way do we lean?")

Average of the errors **keeping the +/− sign**.

```
   bias (table above) = (0 −1 +1 −1 0)/5 = −0.2   → slight UNDERcount
```

- **positive** → systematically overcount
- **negative** → systematically undercount
- **~0** → balanced (over- and under-errors cancel)

### The key insight: MAE ≠ bias

They measure different things. Errors `+2, −2, +1, −1`:

```
   MAE  = (2+2+1+1)/4 = 1.5     ← mistakes are BIG
   bias = (+2−2+1−1)/4 = 0      ← but they CANCEL → "unbiased"
```

**"Unbiased" does NOT mean "accurate"** — it means mistakes balance out, not that
they're small. That's exactly `n_tracks_min2`: bias 0.00 (no lean) yet MAE 1.59
(still ~1.6 off per clip). And `max_per_frame` is a bit biased (−0.38) but has the
smallest MAE — more accurate overall, with a mild undercount tendency.

```
   MAE  = how BIG are the mistakes   (size)
   bias = which DIRECTION on average (over vs under)
   You want BOTH small.
```

---

## 4. "within ±1" — how often are we close?

Fraction of clips where the error is 0 or ±1 (exactly right, or off by one animal).

**Read "±1" as "plus or minus one" = off by AT MOST one animal, in either
direction.** It INCLUDES the exact matches — it is *not* "off by exactly 1." So a
clip is "within ±1" if our number is the true number, or one too many, or one too
few (error is −1, 0, or +1).

Worked example (5 clips):

```
  clip │ expert (true) │ ours │ error = ours − true │ within ±1?
  ─────┼───────────────┼──────┼─────────────────────┼───────────
   A   │       2       │  2   │         0           │  ✓ exact
   B   │       1       │  2   │        +1           │  ✓ one too many
   C   │       3       │  2   │        −1           │  ✓ one too few
   D   │       4       │  1   │        −3           │  ✗ off by 3
   E   │       1       │  1   │         0           │  ✓ exact
```
   4 of the 5 clips are within ±1  →  "80% within ±1" for this tiny set.

On the number line it's just the middle three buckets:

```
   error:  −3   −2   −1    0   +1   +2   +3
           ✗    ✗   ✓✓✓  ✓✓✓ ✓✓✓   ✗    ✗
                     └──── within ±1 ────┘
                  (off by one or less, both directions)
```

**within ±1 = 79% → on ~4 of 5 clips our count matched exactly or was off by one.**
We use ±1 (not exact) because counting wild animals from ~30 sampled frames is
hard, and "within one individual" is good enough for abundance trends.

---

## 5. The three "count definitions" (what our count even is)

After detecting animals on each frame, there are 3 ways to turn that into a
single per-clip count:

```
  max_per_frame  = the most boxes seen in any ONE frame
                   (if you ever see 2 at once, there are ≥2 — needs no linking, robust)
  n_tracks       = number of linked tracks (each track = one individual's path)
  n_tracks_min2  = tracks that appear in ≥2 frames (ignore 1-frame blips/noise)
```

### Why their numbers differ

```
  max_per_frame  → slight UNDERcount (bias −0.38): misses groups whose members
                   are never ALL on screen in the same frame.
  n_tracks (raw) → big OVERcount (bias +1.44): we sample every ~10th frame, so a
                   moving animal jumps far between sampled frames; its boxes don't
                   overlap, the linker thinks each is a NEW animal → one animal
                   FRAGMENTS into several phantom tracks.
  n_tracks_min2  → unbiased (0.00): dropping 1-frame blips cancels much of the
                   fragmentation, but it's noisier (MAE 1.59).
```

Fragmentation illustration (one animal, sampled frames far apart):

```
  frame 1      frame 11     frame 21
   🦌·····      ···🦌··       ·····🦌      ← SAME animal, but boxes don't overlap
   [box]        [box]         [box]
   track A      track B?      track C?     ← linker invents 3 "individuals"  ✗
```

---

## 6. The table, read in plain English

- **`max_per_frame` (MAE 1.32, bias −0.38, 79%)** — best simple counter; off by
  ~1.3 on average, leans slightly low, within one animal 4/5 of the time.
- **`n_tracks_min2` (1.59, 0.00, 74%)** — no directional lean but noisier.
- **`n_tracks` raw (2.62, +1.44, 47%)** — overcounts badly due to fragmentation;
  don't use as-is.

**Bottom line: counting works to ~±1 individual ~79% of the time.**

---

## 7. How it improved after this table

This was the FIRST validation (PR #12, 34 clips). Then:

- **PR #13** — scaled to **400 clips**; the numbers held.
- **PR #14** — fixed fragmentation with a *center-distance* linker (link an
  animal's boxes across frames even when they don't overlap, if their centers are
  close). Result: raw `n_tracks` MAE **2.62 → 1.10**, and `n_tracks_min2` →
  **0.83** — now the best estimator (and still unbiased).

```
   metric          PR #12 (34 clips)   PR #14 (improved, 400 clips)
   n_tracks_min2   MAE 1.59            MAE 0.83
   n_tracks (raw)  MAE 2.62            MAE 1.10
```

---

## One-line glossary

- **PR #N** — the Nth proposed code change awaiting review; #12 = the counting code.
- **error** — our_count − expert_count (per clip).
- **MAE** — average mistake *size* (animals). Lower better.
- **bias** — average *signed* mistake; +over / −under / 0 balanced.
- **within ±1** — % of clips off by ≤1 animal.
- **max_per_frame / n_tracks / n_tracks_min2** — three ways to count individuals
  from per-frame boxes (most-in-a-frame / all tracks / tracks seen ≥2 frames).
- **fragmentation** — one animal split into several tracks because widely-spaced
  sampled frames don't overlap; the main count error, fixed in PR #14.
