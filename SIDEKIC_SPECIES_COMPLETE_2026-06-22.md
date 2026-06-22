# SIDEKIC — Species-ID Done (PR #15 → #16): The Complete Story + What's Next

> Comprehensive, illustrated summary of everything since PR #15 — the species-ID
> effort, now finished with real results and shipped as **PR #16**. Then: what's
> next. Companion to the `SIDEKIC_EXPLAINER_{DETECTION,COUNTING,SPECIES}_*.md`
> trilogy. Lives at `~/SIDEKIC_SPECIES_COMPLETE_2026-06-22.md`.

---

## 0. TL;DR

Since PR #15 we **solved species-ID well enough to beat the best off-the-shelf
tool**, by training our own classifier on our own infrared footage:

```
   off-the-shelf SpeciesNet  →  our fine-tuned model (v2)
   macro-F1   10.2%          →  17.7%   (1.7×)
   micro-F1   21.8%          →  37.7%   (1.7×)
   commits on 36% of crops   →  76%     (the "everything is just 'animal'" problem: solved)
```

All of it is now **PR #16** (in review). With detection and counting already done,
**all three upstream pipeline stages are complete.** Next is *integration*
(wiring the proven pieces into the live pipeline) and *team review* of the stack.

---

## 1. Recap: where we were at PR #15

The pipeline is **Video → Detect → Count → Species → (Distance).** By PR #14,
Detect (~92%) and Count (±1 animal) were done. **PR #15** opened the Species stage
by building two things:

- **The "ruler"** — `species_eval`, which scores *presence-in-set* (did we recover
  the set of species in each clip?) with precision/recall/F1.
- **The baseline** — measured **SpeciesNet** (Google's off-the-shelf camera-trap
  classifier, geofenced to Congo). Verdict: it's **out of domain on grayscale
  infrared** and refuses to name a species on **64% of frames**.

That 64% "roll-up to 'animal'" was the problem the rest of this work attacked.

---

## 2. The problem, illustrated

```
   Our footage: grayscale infrared, night, cluttered forest
   ┌──────────────────────────┐
   │ ░▓ leaves ▓░ branches ░▓ │      SpeciesNet was trained on COLOR DAYLIGHT
   │ ░  ╭────╮ glow-eye  ▓░░  │      photos → on our images it's "out of domain"
   │ ▓  │ 🦌 │ small duiker ░ │      → unsure → it punts:
   │ ░  ╰────╯  ░▓░▓░░▓░░░░░  │
   └──────────────────────────┘        frame → "animal"   (64% of the time)
                                        frame → "blue duiker" (only 36%)
```

You can't *tune* your way out of a domain mismatch — you have to **train on the
domain**. That's the whole idea of what followed.

---

## 3. The journey since PR #15 (the experiment log)

Six steps, all on branch `brian/speciesnet-v403b` (now PR #16):

```
 ① v4.0.3b variant test ....... tried SpeciesNet's other flavor → WORSE (16.7% vs
                                 34.8%). Confirms: not a settings problem, a domain
                                 problem. Model-swap lever exhausted.   [dead end]

 ② fine-tune pipeline ......... built extract → train → eval. The key trick below. ↓

 ③ SpeciesNet, same test set .. re-ran SpeciesNet on the EXACT 297 held-out clips
                                our model is tested on → fair head-to-head bar.

 ④ fine-tune v1 ............... 6,031 crops, 19 species. Beat SpeciesNet on the fair
                                (same-denominator) score + solved the roll-up, but
                                trailed slightly on one metric (undertrained tail).

 ⑤ fine-tune v2 (more data) ... 21,890 crops (3.6×), 22 species. ↓ the decisive win.

 ⑥ v2 results + ship .......... wins EVERY metric, both scoring views → PR #16.
```

### The trick that made fine-tuning cheap (step ②)

We needed crop-level labels ("this box = blue duiker") but the experts only label
*whole videos*. The shortcut:

```
   A clip whose expert label lists EXACTLY ONE species
   ⇒ every MegaDetector box in it MUST be that species ⇒ free, clean labels.

   single-species clip, label = {blue duiker}
   ┌───────────────────────────────────────────┐
   │  ╭──╮     ╭──╮            ╭──╮             │  every crop →
   │  │🦌│     │🦌│            │🦌│             │  "blue duiker"
   │  ╰──╯     ╰──╯            ╰──╯             │  automatically
   └───────────────────────────────────────────┘
   90% of DJEKE clips are single-species → ~tens of thousands of free labels.
```

We split cameras so the model is *tested on stations it never trained on* (no
"memorizing the background" cheating), then fine-tune a standard image model
(EfficientNetV2-S) on the crops.

---

## 4. The scoreboard — progression (the headline)

Scored **fairly** over all 297 held-out test clips, same ruler (a "punt" counts as
a miss — the honest lens for a pipeline that must handle *all* footage):

```
   metric        SpeciesNet      v1          v2 (final)
   ─────────────────────────────────────────────────────
   Macro-F1        10.2%   →    12.7%   →    17.7%    ▲ 1.7× SpeciesNet
   Micro-F1        21.8%   →    32.8%   →    37.7%    ▲ 1.7×
   precision       26.6%   →    37.4%   →    44.7%
   recall          18.5%   →    29.3%   →    32.6%
   commits/crop      36%   →      76%   →      76%    ▲ roll-up solved
   ─────────────────────────────────────────────────────
   (as-run macro: SpeciesNet 24.2 → v1 21.3 → v2 27.9 — v2 wins this view too)
```

```
   Macro-F1 climb:
     SpeciesNet  ██████████░░░░░░░░░░  10.2%
     v1          █████████████░░░░░░░  12.7%
     v2          ██████████████████░░  17.7%   ← our model, IR-trained
```

**What more data fixed (v1→v2):** the long tail recovered (African forest elephant
**0% → 67%** F1, agile mangabey 50 → 67), and the look-alike duikers got less
confused (Peter's duiker up). Roll-up stayed solved (commits on 76% of crops vs
SpeciesNet's 36%).

**Honest caveat:** absolute macro-F1 is 17.7% — the *ultra-rare* species (1–3 test
clips: some birds, galagos, mongooses) are still 0%, but that's capped by how few
clips exist, not by the method. Verdict: **a model trained on our own IR footage
beats the off-the-shelf one decisively, and the trajectory is steeply up.**

---

## 5. What shipped: PR #16

**https://github.com/CongoApe-SIDEKIC/SIDEKIC/pull/16** — "Species-ID: fine-tuned
classifier beats off-the-shelf SpeciesNet on grayscale IR." Stacked on #15;
reviewers Nick, Noah, danni. Contains all 6 commits: the variant test, the
fine-tune pipeline (`extract_species_crops` / `train_species_classifier` /
`eval_finetune_species` + sbatches), the fair-comparison tools
(`run_speciesnet_videos`, `compare_species_models`), and the results doc
(`docs/species_finetune_v1.md`).

---

## 6. Where the whole pipeline stands now

```
   VIDEO ─▶ DETECT ─────▶ COUNT ──────▶ SPECIES ─────▶ DISTANCE ─▶ [Dashboard UI]
            ✅ ~92%        ✅ ±1 animal   ✅ beats SN     (Nick's lane)  (proposal)
            (#6–#11)       (#12–#14)      (#15, #16)
            MegaDetector   IoU linking    fine-tuned EffNetV2-S
            + SAM3 seg                     on IR crops
   ───────────────────────────────────────────────────────────────────────────
   All three upstream stages: DONE and measured. Open PRs #6–#16 await review.
   Nothing running on the cluster.
```

---

## 7. What's next (prioritized roadmap)

### 🥇 Priority 1 — get the PR stack reviewed & merged  *(team-gated, the real unblock)*
PRs **#6–#16** are all open and unmerged — that's the backlog gating everything.
The single highest-leverage thing now is your **tech lead reviewing them
bottom-up** (#6, #7, #10, #11, #12 first; the rest stack on those). *Action: nudge
the team to start the review.* Until this drains, we're (correctly) not piling on
more.

### 🥈 Priority 2 — productionize the pipeline end-to-end  *(the real engineering frontier)*
Today the proven pieces live as **separate scripts / benchmarks**. The next big
build is wiring them into **one production pipeline** that runs on the full ~100k
clips and emits the lab's exact output schema:

```
   clip ─▶ MegaDetector (per-frame boxes) ─▶ SAM3 (segment each box)
        ─▶ IoU linker (count individuals) ─▶ fine-tuned classifier (species)
        ─▶ write the lab's output TSV (per-video: species set, counts, …)
```

⚠️ This touches **Noah's core `batch_dataset_processing.py`** (CODEOWNERS) → align
with the team first. **Good non-blocking move:** I can build it as a *self-contained
POC* on a new branch (doesn't modify Noah's file) that demonstrates the full
detect→count→species flow on a sample and emits the output schema — then propose it
for integration. *(Say the word and I'll start it.)*

### 🥉 Priority 3 — optional / others' lanes
- **Species-ID refinements** — *recommend NOT pursuing now* (diminishing returns;
  the remaining gaps are rare-species-data-capped, and counting/distance don't need
  fine species). Revisit only if the lab specifically wants better rare-species ID.
- **Distance** — Nick's lane (DepthAnything + geolocation); our species/count output
  feeds it.
- **Dashboard UI** — the FiftyOne foundation exists (PR #8); building out the
  browse/filter/correct workflow from the proposal is a later, separable effort.

### The one thing for you to do now
**Nudge your tech lead to start reviewing the #6–#16 stack.** That's what unblocks
merging, which unblocks productionizing. Everything else can wait on it.

---

## 8. Glossary delta (new terms since PR #15)

- **SpeciesNet** — Google's off-the-shelf camera-trap species classifier; out of
  domain on our IR footage.
- **Roll-up / abstain** — when a classifier gives up and says generic "animal"
  instead of a species. SpeciesNet did this 64% of the time; ours, 24%.
- **Fine-tuning** — continuing a pretrained model's training on *your* data so it
  specializes (here: EfficientNetV2-S on DJEKE IR crops).
- **Weak supervision** — getting labels cheaply/indirectly (single-species clip ⇒
  all its crops are that species).
- **Station-disjoint split** — train/test on *different cameras* so the model can't
  cheat by memorizing a background. Prevents "leakage."
- **Macro-F1 vs micro-F1** — macro averages per-species (rare species count
  equally); micro pools all decisions (common species dominate). We report both.
- **Same-denominator scoring** — grading both models over *all* test clips (a punt
  = a miss), so a model isn't rewarded for skipping hard clips. The honest
  head-to-head.

---

*Last updated 2026-06-22. Species-ID stage: complete (PR #16, in review). Next:
review the stack, then productionize the full pipeline.*
