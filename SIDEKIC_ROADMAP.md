# SIDEKIC Summer Roadmap (2026-06-04)

Synthesized from: code audit of the repo + 7 web-researched domain briefs + an
adversarial feasibility pass. (Box proposal was login-gated; built on WORKFLOW.md +
code + on-disk data.)

## Direct answer: fine-tune? train? — NO, not first.
Build the **measurement system** first (eval harness + labeled benchmark), and confirm
whether labels even exist. Train late, only where a frozen benchmark proves a gap.

**Why (audit findings):** 14,780 real DJEKE clips + SAM 3.1 weights exist and detection
runs — BUT **zero labels/ground-truth on disk, zero eval/metrics code, zero training
code.** "Species ID" today = which SAM3 prompt fired (no classifier; Zamba not in code;
MBaza is a passthrough column). Count/distance/pose are stubbed or disconnected. So G1
("improve") and G2 ("compare") are unmeasurable today — no ruler.

## The fork that decides the summer (ask the team first)
Do structured DJEKE species/count labels exist at all? (The expected expert TSV is
missing; only Mondika individual-ID filename notes are on disk — wrong site/schema.)
- Labels exist → comparison-study summer.
- They don't → annotation-first summer (FiftyOne as labeling tool, week 1).
**Annotation time, not GPU, is the real critical path.**

## Highest-ROI code change: the ~1-day track-ID fix
SAM3 computes stable track IDs but the wrapper discards them before COCO export.
Persist `track_id` (sam3_wrapper.py + coco_export.py) → unblocks unique counting,
per-animal distance, pose, behavior, re-ID at once.

## Per-goal
- Species (G1a): crop-then-classify on SAM3 masks; BENCHMARK candidates head-to-head
  (DeepForestVision [CC BY-NC-SA; "beats Zamba/MBaza/SpeciesNet" = unverified claim],
  SpeciesNet [Apache, geofence COG/COD/GAB], BioCLIP-2, MBaza, Zamba). Pick by measured
  macro-F1. Fine-tune head later only for flagged gaps (gorilla, duiker) — hours on H100.
- Count (G1b): engineering, not modeling. After track-ID fix: report 3 definitions
  (max-per-frame / distinct-track / cross-prompt-deduped), clips->events, score vs human
  counts (MAE+bias). Hand-audit detection precision/recall + double-count first.
- Distance (G1c): wire DA3 per-detection (ground-contact point, MEDIAN depth);
  per-site regression calibration from REFERENCE VIDEOS *if* they have known-distance
  markers (~35% error cut). Blocked until xformers rebuilt for cu124.
- Pose (G1d): current head = human COCO-17 on apes (silent bug). Swap to ape model
  (ViTPose/HRNet on OpenApePose). Hardest/lowest-priority — own track or DEFER.
- Comparative (G2) — the spine: one eval harness, standardized metrics with
  camera-level bootstrap CIs + paired McNemar; HOLD-OUT-CAMERA + temporal splits
  (random splits leak); human ceiling via double-coded gold set (Cohen's kappa).
- Viz (G3): FiftyOne hub (Apache, no Docker, COCO+video, embeddings/CLIP search, map);
  also the ANNOTATION tool. Compound query = structured filter once count/species/
  distance are fields. Mirror to Camtrap-DP/DuckDB for SQL across years.

## First week (mostly not blocked on team)
1. Data inventory + locate labels (ask Crickette/Noah/field team) — decides the fork.
2. Track-ID fix + re-run smoke.
3. Detection-quality baseline from existing COCO (blank rate, per-prompt firing, hand
   precision/recall, double-count) — real signal, no new models.
4. Stand up FiftyOne (COCO->FiftyOne loader in empty src/sidekic/ui/) — viewer + labeler.
5. Freeze benchmark split (hold-out cameras + temporal cutoff), versioned file.
6. Scale SAM3 to a few thousand clips via sbatch (the substrate).
7. Rebuild xformers (cu124) for depth; merge PR #2 + fix path rot.

## Top bets
1. Build the ruler before forging swords. 2. The 1-day track-ID fix. 3. Crop-then-
classify species, winner chosen by measurement. 4. Depth = zero-shot DA3 + reference
calibration + contact-point median. 5. FiftyOne week 1 as annotation + query hub.

## Open questions for the team
- WHERE are the expert labels? Any human-coded species/count/distance/individual data?
- Annotator time + who can make ape/duiker species + individual calls?
- Enrolled catalog of known individuals (for re-ID)?
- Are REFERENCE VIDEOS true calibration clips (known-distance markers)?
- License posture: non-commercial weights OK (DeepForestVision/DA3-nested) or permissive
  required (Zamba MIT / SpeciesNet Apache / DA3METRIC Apache)?
- Scope: prioritize species+count+distance over pose/behavior/re-ID if time short?
  (Behavior + re-ID are zero-code today; likely out of scope for 10 weeks.)

## Descoped for this summer (per stress-test)
Pose head-swap (own track or defer), behavior (CARe/DeepWild — zero code; CARe =
Animal Kingdom action-recognition, confirm w/ team), full re-ID, 250k-scale MegaDetector
blank-filter (working set is ~15k not 250k), NL-text-to-SQL stretch.
