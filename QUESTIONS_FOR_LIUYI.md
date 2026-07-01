# Questions for Liuyi

*Logged while working through the tasks in `liuyi_task_assignment.png` (2026-07-01). Full findings/reasoning behind each question are in `PROGRESS_NOTES.md` — this is just the distilled list to bring to Liuyi.*

1. **Where are the two CGM raw CSV files?** They weren't included with the other materials (K01 proposal, reference papers, task list) and aren't anywhere on my computer. I can't do task 3 (understand the raw format, convert it) without them.

2. **What exactly are the "two CSV files"?** My best guess from the K01 proposal is (a) a Dexcom Clarity CGM export (glucose every ~5 min) and (b) either the PARC app's cognitive test scores (Symbols RT / Grids Error Score) or the Fitbit activity/sleep data — but the proposal doesn't pin this down and I don't want to assume wrong. Can you confirm which two?

3. **Channel mismatch: Ben's model is hardcoded to exactly 2 input channels (activity + light).** CGM only gives one native channel (glucose). Do you want me to:
   - (a) generalize Ben's model to accept an arbitrary number of channels (cleaner, more work), or
   - (b) always pair glucose with a second channel — e.g., Fitbit activity/sleep, mirroring Ben's activity+light structure?
   
   Note this has a wrinkle either way: per the K01 proposal, Fitbits are only worn by participants **≥13 years old** — the 9–12 age group would only ever have single-channel glucose data. Should younger participants be modeled differently, or excluded from any two-channel approach?

4. **Single-target vs. multi-target regression?** The K01 proposal's analysis plan calls for predicting four separate continuous outcomes (Mean Symbols RT, Mean Grids Error Score, and the SD of each). Ben's code currently assumes exactly one regression target at a time. Should I set up the adaptation for one target at a time first (simpler, closer to a direct port of Ben's existing pattern), or aim for joint multi-target regression from the start?

5. **Raw-signal (TSFM) vs. feature-engineering arm, or both?** The K01 proposal's Aim 1 exploratory analysis explicitly plans to compare "raw data learning" (what Ben's TSFM approach does) against "feature engineering" (what Ben's `FeatureMLP`/hand-crafted-features baseline does), across several ML model types (logistic regression, SVM, gradient boosting, neural nets). Is my task just the raw-data/TSFM arm for now, or should I also be thinking about replicating the feature-engineering comparison?

6. **The two reference papers turned out not to be about CGM or about Chronos/TimesFM specifically** — one is a general TSFM-interpretability paper (studies MOMENT/Chronos/MOIRAI internals, no health data except one ECG example), the other is Google's SensorFM paper (Fitbit/Pixel Watch data only, explicitly no glucose data anywhere in it). Both were useful for general concepts (tokenization/patching, normalization pitfalls, missing-data handling, realistic expectations for regression accuracy on "soft" targets), but neither gives CGM-specific or Chronos/TimesFM-specific implementation guidance. Is that expected/intentional (i.e., they're meant as conceptual background, and Ben's actual code is the implementation reference), or is there a more directly relevant paper I should be reading instead?

7. **Timeline for the compute2 environment setup (task 1)?** I set this aside per your instruction to focus on tasks 2–4 first, but wanted to flag it so it doesn't get forgotten — let me know when you want me to pick it back up.
