# Open Questions for Liuyi (v2 — post-answers)

*Supersedes the pre-answer `QUESTIONS_FOR_LIUYI.md` (kept for history). These are the questions raised by actually building the pipeline and reading `diabetes-fitbit`. Ordered by how much they block progress.*

---

### Q1 — Access to the real merged CSVs  🔴 *blocks the real run*
The pipeline is built and validated; the only thing between me and a real result is the data. Can I get **`Cohort1_scores_merged_with_glucose.csv`** and **`Cohort2_scores_with_glucose.csv`** (they're gitignored in `diabetes-fitbit`, so not in the clone)? Or should I run in an environment where they already live? *This is your project's own data (Mack used it), separate from the pending IRB for Ben's data — so I assume it's accessible, but confirming.*

### Q2 — The pre-test window & the raw CGM stream  🔴 *shapes the model input*
In `diabetes-fitbit`, `Glucose_Before_Test` is **already clipped to some lookback upstream** (the merge script isn't in the repo), and only that clipped array is stored — not the raw continuous CGM. For a TSFM I'd ideally control the window. Two sub-questions:
- (a) Is the **upstream merge script / raw Dexcom export** available, so I can define the lookback myself (e.g. exactly 2 h before each test, per the K01 hypothesis)?
- (b) If not, what **lookback is baked into the current arrays**, so I can document it? (I'll default to "use the full clipped window" until told otherwise; I can also cap to the most-recent 30 readings = 2.5 h.)

### Q3 — Prices inversion + what the three scores measure  🟠 *affects modeling & interpretation*
Your `next_steps` notebook flags that **`prices_cognitive_score` is inverted (higher = worse)**. I currently **negate it** so all three targets read "higher = better" (toggle: `config.PRICES_IS_INVERTED`). Please confirm that's what you want. Also: the code never documents **what Grids / Symbols / Prices actually measure** (accuracy? reaction time? a composite?) or their units — knowing this helps me sanity-check predictions and choose sensible normalization. (Dr. Hassenstab's PARC/ARC docs, maybe?)

### Q4 — What does "success" look like, given Mack's null result?  🟠 *defines the deliverable*
Mack's 43-feature arm has negative R²/chance AUROC everywhere, and session-CV > LOPO says the signal is mostly subject-baseline. So for the TSFM arm, is the goal:
- (a) a **fair confirm/refute** — "do learned Chronos representations beat hand-crafted features under identical grouped CV?" (my current default), or
- (b) chase a **specific hypothesis** (e.g. only hypoglycemia windows, or within-subject/normalized targets, or subject-as-covariate)?
Either is easy to run; I want to aim at the right target.

### Q5 — Chronos family & checkpoint size  🟡 *easy to change*
The ICML paper studied Chronos-**T5**; Ben used Chronos-**Bolt**. My code supports both via `BaseChronosPipeline` and defaults to **`amazon/chronos-bolt-small`** (fast, CPU-friendly). OK to standardize on Bolt, and any preference on size (small → base → large)? Larger = richer embeddings but needs compute2.

### Q6 — compute2 environment (Task 1)  🟡 *scheduling*
Still deferred, as you said. Everything so far runs on my laptop CPU with `bolt-small`. I'll need compute2 for larger checkpoints / Arm-B training at scale — when do you want me to set it up?

### Q7 — Frozen vs fine-tuned encoder  🟢 *later*
Currently the encoder is **frozen** (zero-shot embeddings — the ICML approach). Worth trying **LoRA fine-tuning** later, or keep it frozen for the first comparison? (I'd only invest here if the frozen arm shows promise.)

---

*Resolved by your last round of answers (recorded so we don't re-litigate): the "two CSVs" were Ben's watch data; Task 3 uses the ICML repo, not our data; single channel; single target × 3 models; TSFM arm only; the ICML paper is the relevant Chronos reference.*
