# SIDEKIC — Project Handoff & Summary

**Sanz Lab camera-trap analysis pipeline (DTRC 2026).** Turns raw camera-trap video
into structured ecological data: for every clip → is there an animal, how many, what
species, how far, and (what behavior). Runs on RIS Compute2 (H100 SLURM).

Author: Brian Zhou. Last updated: 2026-07-03.

---

## 1. The pipeline at a glance

```
  VIDEO CLIP
     │
     ▼
 ┌──────────┐   ┌──────────┐   ┌───────────┐   ┌────────────┐   ┌──────────────────┐
 │ 1 DETECT │──▶│ 2 COUNT  │──▶│ 3 SPECIES │──▶│ 4 DISTANCE │   │ 5 POSE / BEHAVIOR │
 │MegaDetect│   │IoU+centre│   │fine-tuned │   │per-station │   │ DLC SuperAnimal + │
 │ V6 ~92%  │   │ linker   │   │EffNetV2-S │   │ calibration│   │ ST-GCN (Noah)     │
 │ recall   │   │ ~±1 ind. │   │ (v3,300px)│   │  ~0.95 m   │   │ ~0.50 mAP         │
 └──────────┘   └──────────┘   └───────────┘   └────────────┘   └──────────────────┘
   PR #10/#11     PR #12/#14     PR #15/#16       PR #26           PR #17/#24/#25
     │              │              │                │                  │
     └──── productionized end-to-end: scripts/pipeline_poc.py (PR #23) ┘
                          │
                          ▼
        OUTPUT SCHEMAS:  speciesID (per clip)   +   distances (per individual, per 2 s)
```

**Two big recurring lessons across the project:**
1. *Off-the-shelf RGB/daylight models fail on our grayscale-IR forest footage* — SAM3
   (detection), SpeciesNet (species), and Depth-Anything-3 (distance) were all
   out-of-domain. The fix each time was a camera-trap-native model or per-station
   calibration.
2. *Validate at scale before trusting a result* — several approaches looked great on a
   small sample and collapsed on more data (DA3 depth, the distance offset).

---

## 2. Stages 1–3 (foundation — merged to `main`)

| stage | approach | key result | code |
|---|---|---|---|
| **1. Detect** | MegaDetector V6 (`MDV6-mit-yolov9-c`), animal class, replaced SAM3 (43% recall) | **~92% recall** on 400-clip scaled benchmark; duikers 94–99% | `scripts/run_megadetector*.py` |
| **2. Count** | IoU + **center-distance** linking → cross-frame `track_id` → individual count | **~90% within ±1** individual, MAE 0.64 | `src/sidekic/tracking/iou_linker.py` |
| **3. Species** | Fine-tuned **EfficientNetV2-S** on MD crops; weak supervision (single-species clips → free labels); station-disjoint split | **beats off-the-shelf SpeciesNet** (micro-F1 ~57% vs ~22%); SpeciesNet was out-of-domain on IR | `scripts/train_species_classifier.py`, `src/sidekic/eval/species_eval.py` |

Species went v1 (6 k crops) → v2 (22 k) → **v3 (train at 300 px = best)**. A bigger
backbone (v4) and an ensemble were tested; v3 is the production model. Full-archive
results are under **Productionizing** below. Review dashboard: FiftyOne dataset
`djeke_speciesID_v3_review` (14,577 clips).

---

## 3. Stage 4 — DISTANCE  *(detailed)*

**Goal:** estimate each animal's distance from the camera in metres, and emit the
lab's `distances` output schema. **Status: done for DJK001–064 (PR #26); 065–100
deferred; 2-D skipped.**

### 3.1 What did NOT work (and why)
```
 monocular depth (Depth-Anything-3)   corr +0.22 vs GT   ← out-of-domain on grayscale IR
 single global geometry model         corr +0.09         ← every camera's geometry differs
 coarse near/mid/far bins (x-station) ~chance (35%)       ← same reason
```
(Noah's DLC/DA3 depth backend is the same depth approach — same result. Cross-checked.)

**Conclusion: distance is a per-station calibration problem.** In a fixed camera an
animal's *foot position in the frame* maps cleanly to ground distance — but the mapping
differs per camera, so nothing transfers.

### 3.2 What works — per-station calibration
Fit each station's `foot_y → distance` curve from **our own ground truth** = the expert
distance labels paired with the animal's MegaDetector foot position.

```
   near ─ high foot_y ────────────────────── far ─ low foot_y
   (bottom of frame)                          (near horizon)
        │  fit per station: distance = a·foot_y + b   │
        ▼                                              ▼
     ~1–2 m                                          ~8–10 m
```
- **`scripts/calibrate_distance.py`** — fits + leave-one-out validates per station,
  saves `station_calibration.json` (+ an `apply_calibration()` helper). Unit-tested
  (`tests/test_calibrate_distance.py`).
- **Result: all 64 DJK001–064 stations calibrated, LOO MAE median 0.95 m / mean
  1.13 m** — vs ~1.5 m for depth or predict-the-mean. (Note: ~1 m is roughly the
  *floor* — animal foot-position noise dominates, so no model beats it much in 1-D.)

### 3.3 Emitting the output
**`scripts/emit_distances.py`** → MegaDetector per 2-s timestamp → IoU+center-distance
tracking (a stable `focal.id` per individual) → per-station calibration → `distance_in_m`,
joining species/count from the speciesID output. Emits the exact
`1_example_output_distances.txt` schema. `reaction.yn` / `behavior_string` are left `NA`
(that's the behavior stage). Output: `~/distance_out/distances_djeke.tsv` (DJK001–064).

### 3.4 The reference clips & the un-labeled stations (065–100)
Each station has a `Reference_DJK*.avi` calibration clip: **red/white meter tapes (1 m
spacing) + a center pole + a person walking the marks holding numbered signs.**
- **`scripts/detect_reference_markers.py`** finds the tape rows (vivid-red HSV segmentation).
- **`scripts/calibrate_from_markers.py`** turns tape rows + their distances into a
  station calibration (no expert GT needed).
- **Open issue for 065–100:** the tape *spacing* is 1 m, but the **absolute anchor
  (which tape is nearest in-frame) varies per camera** — validated on DJK001, its
  nearest visible tape is ~3 m (nearest = 3 m → 0.70 m error; nearest = 1 m → 1.62 m).
  So each station needs its **sign number** to anchor. Since 065–100 is bonus (unlabeled)
  footage, distance there is **deferred** — the method is proven; it just needs the
  per-station sign anchors (a quick manual read, or OCR) if the lab ever wants it.

### 3.5 Decisions made
- **2-D / inter-individual distances: not built.** No lateral (X) ground truth, so the
  classic CV homography/pinhole approach can't recover between-animal distances
  reliably. (A full parametric-pinhole model was prototyped/assessed with Danni; the
  1-D accuracy is floored ~1 m and the 2-D needs lateral GT we don't have.)

### 3.6 Re-run
```bash
# calibrate (from expert GT) + validate — writes station_calibration.json
sbatch slurm/distance_calibration.sbatch
# emit the distances schema for DJK001-064
sbatch slurm/emit_distances.sbatch          # -> ~/distance_out/distances_djeke.tsv
```

---

## 4. PRODUCTIONIZING — the end-to-end runner  *(detailed)*  (PR #23)

**Goal:** one script that takes a clip through detect → count → species and emits the
lab's `speciesID` schema — the thing that actually produced the full-archive deliverable.

```
 clip → decode frames → MegaDetector per frame → IoU+centre linker (num.individuals)
      → fine-tuned EffNetV2-S (v3) per crop → per-clip species set + confidence
      → join per-video metadata → speciesID row(s)
```
- **`scripts/pipeline_poc.py`** — the runner. Resumable (`--resume`), per-clip error
  isolation, `--all` sweep mode. Points at `species_model_v3`. (Uses a compact copy of
  the center-distance linker; swaps to `sidekic.tracking.iou_linker` now that #18 is on main.)
- **`scripts/score_pipeline_output.py`** — end-to-end scorer vs GT (species presence-in-set
  + counts). Reuses the canonical `sidekic.eval.species_eval` ruler.

**Output schema** (`speciesID_djeke_v3.tsv`):
`video.name, camera_site, year..sec, predicted.species, confidence, num.individuals`

### Full-archive result
Ran over **all of DJEKE — 14,577 clips, 100 stations (incl. ~5,800 never-analyzed
DJK065–100), 0 errors, ~24 h:**
```
 SPECIES  micro-F1 56.3% (all 100 stations) · 57.5% (DJK001-064) · exact-set 41%
 COUNTS   within ±1: 90%   ·   MAE 0.64
```
Deliverable: `~/pipeline_poc/speciesID_djeke_v3.tsv`, browsable in the
`djeke_speciesID_v3_review` FiftyOne dashboard.

### Re-run
```bash
sbatch slurm/pipeline_djeke_v3.sbatch       # full DJEKE sweep + self-score
```

---

## 5. Stage 5 — POSE / BEHAVIOR  *(Noah's work — summary so we're aligned)*

Noah Wolk built this whole stage (PR #17 pose, #24 behavior, #25 UI). Two parts:

### 5.1 Pose — DeepLabCut SuperAnimal (PR #17)
- **Zero-shot SuperAnimal-Quadruped** pose estimation: runs on detected frames, matches
  each predicted individual to its detection box, writes keypoints + a quadruped
  skeleton for the dashboard.
- Runs inside a **DeepLabCut Enroot/Pyxis container** (its deps conflict with SAM3), as a
  service launched by the dashboard. Lives in Noah's storage (`/storage3/.../noahw/`).
- **Gaps:** zero-shot (unvalidated on our IR); **quadruped-only** (fits duikers/hogs/
  elephants, *not* primates/apes — a schema mismatch for the lab's flagship species).

### 5.2 Behavior — ST-GCN multi-label (PR #24, UI #25)
- **Skeleton-based spatio-temporal graph CNN** on the pose keypoints → **multi-label**
  behavior per 2-s window (STAY/REST/RUN/LOOK/SMELL/APPROACH/…). Features: position +
  confidence + velocity + bone vectors.
- Trained on the GT `behavior_string`; **~0.504 mAP** (velocity+bone); per-class
  threshold calibration; camera-site split; eval script + inference wrapper + dashboard
  wiring.
- **The core challenge (from our analysis of the GT):** behavior is **~95% "STAY"** —
  the interesting behaviors are rare and fine-grained. Movement/tracking alone can only
  separate STAY vs RUN coarsely, so the fine behaviors genuinely need the pose signal.
- **Gaps:** duiker-heavy training, rare behaviors data-starved, quadruped-only, minimal
  error analysis. These are the natural places to contribute if we extend it.

---

## 6. Deliverables & where they live
| what | path |
|---|---|
| **Species + counts** (all 100 stations) | `~/pipeline_poc/speciesID_djeke_v3.tsv` |
| **Distances** (DJK001–064) | `~/distance_out/distances_djeke.tsv` |
| **Per-station distance calibration** | `~/distance_cal/station_calibration.json` |
| **Review dashboard** | FiftyOne dataset `djeke_speciesID_v3_review` (14,577 clips) |
| GT (from the lab) | `~/shared_from_desktop/1_ground_truth_{species_IDs,distances}.txt` |

Launch the dashboard:
```bash
.venv/bin/python -c "import fiftyone as fo; fo.launch_app(fo.load_dataset('djeke_speciesID_v3_review'),port=5151,remote=True).wait()"
# then from your laptop:  ssh -N -L 5151:localhost:5151 zhou.n@c2-login-001.ris.wustl.edu  → http://localhost:5151
```

---

## 7. Pull requests
| PR | stage | status |
|---|---|---|
| #10, #11 | Detect (MegaDetector, MD→SAM3 POC) | merged |
| #12, #14 | Count (IoU + center-distance linker) | merged |
| #15, #16 | Species (eval harness + fine-tune pipeline) | merged |
| #18, #19 | re-lands of #13/#14/#16 onto main | merged |
| **#23** | **Productionizing** (end-to-end pipeline) | open — awaiting review |
| **#26** | **Distance** stage | open — awaiting review |
| #17, #24, #25 | Pose / Behavior (Noah) | open |

---

## 8. Environments (RIS Compute2)
| venv | use |
|---|---|
| `~/md-venv` | MegaDetector + torch 2.6.0+cu124 + cv2 (detect, count, species inference, distance) |
| `<repo>/.venv` | SAM3 + `sidekic` (editable) + FiftyOne + scorers/tests |
| `~/sn-venv` | SpeciesNet (baseline benchmark) |
| `~/depth-venv` | Depth-Anything-3 (distance feasibility — deprecated) |
| Noah's DLC container | DeepLabCut SuperAnimal pose (Enroot/Pyxis) |

⚠️ Never run heavy work on the login node — submit `srun`/`sbatch` GPU jobs (H100,
`-A compute2-workshop -p general-gpu --gpus=1`). Never `uv run` (it re-syncs and breaks
the torch cu124 pin) — use `.venv/bin/python`.

---

## 9. What's next (for whoever continues)
1. **Merge #23 (productionizing) + #26 (distance)** — the last two of my PRs.
2. **DJK065–100 distance** (optional/bonus) — detect markers + per-station sign anchors
   (manual or OCR), then `calibrate_from_markers.py` → `emit_distances.py`.
3. **Pose/behavior refinements** (Noah) — species generalization (esp. primates),
   rare-behavior imbalance, validation on IR.
4. **Scale to other sites** — GTAP (54 k) / MONDIKA (25 k) clips are *not* on this
   storage; the species pipeline is ready to run on them once mounted (note: the species
   model's vocabulary is DJEKE-trained).
5. **Human-in-the-loop** — the dashboard supports browse/filter/correct; exporting
   corrected labels back to a clean TSV closes the loop.
```
