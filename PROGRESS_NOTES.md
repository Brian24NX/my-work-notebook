# Progress Notes — Applying TSFMs to the CGM Dataset

*Brian's notes on the tasks Liuyi assigned (see `liuyi_task_assignment.png`). Task 1 (dev env on compute2) is deferred per Brian's own instruction — everything below covers tasks 2–4. Written 2026-07-01.*

**Sources used**: Ben's `mod-actigraphy-advanced` codebase (read directly, file:line citations below), `Ray K01.pdf` (the grant proposal driving this project), `7669_Exploring_Representations.pdf` (TSFM interpretability, ICML 2025, CMU Auton Lab), `2605.22759v2.pdf` ("Towards a General Intelligence and Interface for Wearable Health Data" / SensorFM, Google Research). For full background on Ben's project generally (not just the TSFM piece), see `/Users/brian/Desktop/mod-actigraphy-advanced/docs/PROJECT_OVERVIEW.md` — this note only covers the parts relevant to input formats and regression, in more depth than that doc goes into.

---

## Task 2: Ben's codebase, high-level + TSFM input formats

### High-level (30-second version)

Ben's project predicts preterm birth from actigraphy (wrist motion + light) using, among other models, two frozen pretrained time-series foundation models — Amazon's **Chronos-Bolt** and Google's **TimesFM** — as feature encoders, with a small trainable transformer + classification head stacked on top. This is the direct architectural template Liuyi wants applied to CGM (glucose) data instead of actigraphy data.

### The exact input format each TSFM wrapper expects

Both encoders live in `src/models/pretrained_models.py` and share the same interface contract (`TimeSeriesEncoderBase.encode()`, line 17): **input `(N, C, T)`, output `(N, C, d_model)`** — N samples, C channels/variates, T timesteps, NaN = missing. Everything else about how each one gets there differs:

**`ChronosEncoder` (line 39-113)**
- Wraps `ChronosBoltModelForForecasting.from_pretrained(...)` from the `chronos-forecasting` pip package (line 54, 65).
- Chronos-Bolt's native `.encode()` only takes 2D `(batch, seq_len)`, so the wrapper flattens channels into the batch dim first: `(N, C, T) → (N*C, T)` (line 99), calls `self.model.encode(flat)` → `(N*C, num_patches, d_model)`, reshapes back to `(N, C, P, d)`, then **pools over patches** (mean or last, configurable) → `(N, C, d_model)` (lines 101-113).
- **No fixed sequence length required** — Chronos handles arbitrary-length input internally (it patches/tokenizes on its own).
- Missing values: NaN-in, handled internally by Chronos's own missing-value convention.

**`TimesFMEncoder` (line 116-267)** — meaningfully more involved:
- Wraps `timesfm.TimesFm(...)` from the `timesfm` pip package (line 138, 152), loaded with the model's own hyperparameters (`context_len=512, input_patch_len=32, output_patch_len=128, num_layers=20, model_dims=1280` for the 200M checkpoint — these **must match whatever checkpoint you load**, they're not arbitrary).
- **TimesFM needs a fixed-length input** (`context_len`, default 512). The wrapper pads (with zeros, front-padded) or truncates (drops from the front — i.e., **right-aligned**, keeping the most recent `context_len` timesteps) to hit that exact length (lines 211-225).
- Missing values are NOT NaN-native here — the wrapper converts NaN → an explicit **padding mask** (1=missing, 0=valid) + zero-fills the actual value (lines 202-209), because TimesFM's own preprocessing (`_preprocess_input`) expects a `(values, padding_mask)` pair, not NaNs.
- TimesFM also requires an explicit **frequency category token** (line 227-230: `freq = 0` for "high frequency / sub-daily", 1 = medium, 2 = low) — a coarse hint about the sampling rate, passed alongside the values.
- Pools over **valid (non-padded) patches only** (mean or last-valid) → `(N, C, d_model)` (lines 245-267).

**`FoundationModelClassifier`** (line 270-704, aliased as both `ChronosWrapper` and `TimesFMWrapper` — same class, line 701-703) — the actual model used in training:
- **Requires 3D patient-level input**: `activity: (B, D, T)`, hard-asserted (line 620-623: `assert activity.ndim == 3`). It will not accept 2D day-level input directly.
- **Hardcoded to exactly 2 input channels** — `num_channels = 2  # activity + light` is a literal constant (line 340), and `forward()`/`_encode_and_transform()` take two explicitly named tensor arguments (`activity`, `light`), stacked as `torch.stack([activity, light], dim=2)` (line 545). **This is not a generic multi-channel model — it's specifically wired for exactly two named signals.**
- Pipeline per day: mask invalid timepoints as NaN → stack the 2 channels → flatten to `(B*D, 2, T)` → encode with the frozen TSFM → `(B*D, 2, d_enc)` → reshape to `(B, D, 2, d_enc)` → **combine the 2 channels** into one embedding per day via `concat_project` (concat + linear projection) or `add` (sum, optional projection) → `(B, D, d_model)` (lines 401-422) → optional padding tokens between days, optional CLS token, + positional embeddings → **causal transformer** (so a given day can't attend to future days — appropriate for "how far in advance can we predict" style tasks) → pool → classification head.

### What this means for CGM data specifically

1. **Channel count mismatch.** CGM gives you one native channel: glucose (mg/dL) over time. Ben's model hardcodes two. Before this can "just work" on CGM data, someone needs to either (a) generalize `FoundationModelClassifier` to accept an arbitrary channel count instead of the hardcoded `2`, or (b) always pair glucose with a second channel — which the K01 proposal actually gives a natural candidate for: **Fitbit activity/sleep data**, collected for participants ≥13 years old (see Task 3 below). Younger participants (9–12) don't wear a Fitbit, so they'd only ever have the one glucose channel — meaning a real design decision is needed here, not just a code change (see question in `QUESTIONS_FOR_LIUYI.md`).
2. **Chronos vs. TimesFM tradeoff for glucose specifically.** Per `7669_Exploring_Representations.pdf` (the interpretability paper), Chronos represents input by **scaling then quantizing into a fixed vocabulary of 4096 discrete tokens** — i.e., it bins continuous values. TimesFM (like MOMENT in that paper) uses **continuous patch embeddings** instead. For glucose, where the clinically meaningful signal can be a difference of a few mg/dL (e.g., trending toward hypoglycemia), it's worth empirically checking whether Chronos's quantization resolution loses meaningful precision — that paper doesn't test this directly, it's a specific risk worth validating on real CGM data once available.
3. **Normalization can silently discard the thing that matters most.** The same paper found that standard TSFM input normalization tends to strip out *absolute amplitude* information, keeping mostly shape/trend. For glucose, **absolute level is often the clinically critical variable** (a reading of 70 vs. 250 mg/dL means very different things), not just the shape of the curve. This needs to be checked, not assumed, before trusting a TSFM embedding for anything downstream.
4. **TimesFM's fixed `context_len` forces a decision about window length.** CGM readings come every ~5 minutes (per the K01 proposal, Dexcom G6). A `context_len=512` window is therefore ~42.7 hours of continuous glucose — much longer than Ben's "one day" framing. Liuyi/the team will need to decide what window is actually meaningful for predicting *momentary* cognitive performance (the K01's own hypothesis-driven analysis, described below, looks at glucose in just the **2 hours prior to a cognitive test** — a much shorter and more targeted window than either a full day or TimesFM's default 512-step context).
5. **The second reference paper (SensorFM) is good conceptual background but doesn't touch Chronos/TimesFM or CGM at all** — it's a different, proprietary Google model trained on Fitbit/Pixel Watch data, with zero glucose content anywhere in the 75 pages. Its useful transferable ideas: (a) missingness handled via a learnable "mask token" + only computing loss on real (not synthetic) values — a good pattern if we ever fine-tune rather than just doing zero-shot inference; (b) regression from wearable embeddings is well-supported architecturally, but even at massive scale, "soft"/subjective targets (mood/anxiety scores) only reach r≈0.4–0.5, versus r≈0.8–0.9 for "hard" targets (age, weight) — a realistic expectation-setting data point for a cognitive-performance regression target, which is arguably similarly "soft."

---

## Task 3: CGM raw CSV format — currently blocked

**The two CSV files mentioned in the task aren't on this machine.** I searched Desktop, Downloads, Documents, and the project folders — nothing matching CGM/glucose/Dexcom data turned up (only unrelated old coursework CSVs from a 2022 folder). This needs to come from Liuyi or wherever the study's data is stored — logged in `QUESTIONS_FOR_LIUYI.md`.

**What I can infer about the likely format from the K01 proposal** (not confirmed — needs checking against the actual files once available):
- **CGM stream**: Dexcom G6, glucose sampled every ~5 minutes via Dexcom Clarity's data-sharing/export (p.5, 8 of the proposal). Typical Dexcom Clarity CSV exports (general knowledge, not confirmed for this study) include a timestamp column, an "event type" column (glucose readings are usually tagged EGV = "estimated glucose value," interleaved with calibration/insulin/carb event rows that aren't glucose readings), and the glucose value in mg/dL — worth checking for exactly this structure once the real file is in hand, since non-EGV rows would need filtering out.
- **Cognitive test data**: the PARC app produces two scores per session — **Symbols RT** (median response time in seconds, processing speed) and **Grids Error Score** (average Euclidean distance error, working memory) — 5 sessions/day for 10 days (proposal p.5). This is plausibly the second CSV, but could equally be Fitbit data instead — genuinely unclear without seeing the files (see questions log).

**Planned conversion once the files exist** (mirroring Ben's ETL pattern in `src/etl/extract_actigraphy.py` and `src/etl/extract_labels.py`, adjusted for this study):
1. Parse each CSV into a clean `(timestamp, value)` series per participant, filtering to actual glucose readings if the Dexcom export includes non-reading event rows.
2. Decide on windowing: Ben's pipeline windows actigraphy into noon-to-noon daily rows (`reshape_to_daily()` in `extract_actigraphy.py`) because pregnancy is a long, continuous, multi-month record. This study is a **fixed 10-day enrollment** with **discrete cognitive-test timestamps** (5/day) — so the natural window here is probably "N hours of CGM immediately preceding each cognitive test event," not a full calendar day. This is a structurally different windowing problem than Ben's, not a drop-in reuse of `reshape_to_daily()`.
3. Output target format, following Ben's convention: Parquet with explicit timestamp/value/mask columns, one file per participant, labels (Symbols RT / Grids Error Score, or normalized versions) in a separate file — see `config/data/mod.yaml` in Ben's repo for the exact schema convention to mirror.

---

## Task 4: Adapting Ben's code for regression output

"Converting the transformer's header into a single header output" = the final `nn.Linear` layer of the classification head. Checked exactly how this currently works in `src/models/modules.py`:

**Ben's code already has a single-output regression path — it's just narrowly hardcoded to one specific task.** In `FinetuneLightningModule.__init__` (line 353-359):

```python
if task == "ptb_classification":
    self.num_classes = 2
    self.is_classification = True
else:  # gestational_age regression
    self.num_classes = 1
    self.is_classification = False
```

`self.num_classes` is passed straight into the model's constructor (line 382: `model_kwargs["num_classes"] = self.num_classes`), which becomes the output width of the final `Linear(head_hidden_dim, num_classes)` layer in `FoundationModelClassifier.classifier` (`pretrained_models.py` line 384-390). So for gestational age, the "header" already outputs a single scalar per sample. Then in `_shared_step` (`modules.py` line 583-591):

```python
else:
    labels = labels.float() / 365.0
    logits = logits.squeeze(-1)
    loss = self.loss_fn(logits, labels)
    preds = logits * 365.0
```

`.squeeze(-1)` turns the `(B, 1)` output into `(B,)`, and `MSELoss` is used (set at line 391: `self.loss_fn = nn.MSELoss()`). **This is the exact mechanism task 4 is asking about — it already exists, but only for one hardcoded task.**

**What actually needs adapting for the CGM/cognition project:**
1. **The task-name check is a string literal, not a general flag.** `if task == "ptb_classification"` means *any other task name at all* currently falls into the regression branch. That's fragile — it happens to work by accident, not by design. Adapting this properly means replacing the hardcoded check with an explicit `is_classification` flag read from the task config (Ben's `config/task/*.yaml` files already have per-task config — just none of them currently carry this flag explicitly).
2. **The `/365.0` and `*365.0` are gestational-age-specific**, not general-purpose normalization — this scales days into a roughly-O(1) range for training stability. Whatever cognitive metric Brian ends up predicting (Symbols RT in seconds, or Grids Error Score as a Euclidean distance) will have its own natural scale, and will need its own normalization constant (or better, a general per-dataset z-score computed from the training set, rather than another hardcoded magic number).
3. **Single- vs. multi-target regression is an open design question, not just an implementation detail.** The K01 proposal's own Aim 1 analysis plan (p.7) calls for predicting **four separate continuous outcomes**: Mean Symbols RT, Mean Grids Error Score, SD of Symbols RT, SD of Grids Error Score. Ben's current code assumes exactly one regression target (`num_classes=1`, then squeezed to a plain scalar). Supporting multiple simultaneous targets would mean *not* squeezing, keeping `logits` as `(B, N)`, and computing per-target loss/metrics — a bigger change than just flipping a flag. Worth confirming with Liuyi whether the immediate goal is one target at a time (simplest, reuses Ben's pattern almost as-is) or joint multi-target regression (see questions log).

---

## Summary

| Task | Status |
|---|---|
| 1. Dev env on compute2 | Deferred (per Brian) |
| 2. Understand Ben's codebase / TSFM input formats | Done — see above |
| 3. Understand CGM CSV raw format, convert to TSFM format | **Blocked** — CSV files not available yet |
| 4. Adapt Ben's code for regression output | Understood + planned — see above; not yet implemented since it depends on task 3's data being available to test against |
