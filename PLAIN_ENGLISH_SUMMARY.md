# Plain-English Summary — What We Built & What to Tell Liuyi

*A no-jargon catch-up. If you read nothing else in this project, read this. It explains every technical word the first time it shows up.*

*🗣️ For the meeting itself, there's an even shorter phone version: [`MEETING_CHEATSHEET.md`](MEETING_CHEATSHEET.md).*

---

## 0. The 30-second version

We're studying: **when a kid with Type 1 Diabetes has their blood sugar go up or down, does it change how well they do on a quick thinking test a few minutes later?**

Your job is to try a **modern AI approach** to answer this, and compare it fairly against the older approach a labmate (Mack) already tried. Over these sessions we **built the entire software pipeline that does this**, tested it end-to-end on stand-in data, and got it ready so that the day the real data arrives, it's basically one command. We can't get "real answers" yet because we don't have the real data file — but everything is built and waiting.

---

## 1. What the project is actually about (no jargon)

- Kids with **Type 1 Diabetes (T1D)** have blood sugar (**glucose**) that swings up and down a lot.
- They wear a **CGM** = *Continuous Glucose Monitor* — a small sensor that measures glucose automatically every 5 minutes. So we get a long list of glucose numbers over time.
- They also play short thinking games on a phone app several times a day. Three games/scores: **Grids, Symbols, Prices** (these test memory and mental speed).
- **The question:** using the glucose readings from the ~couple hours *before* a game, can we predict the game score? If yes, that means short-term blood sugar affects thinking in the moment.

That's it. Glucose readings in → predicted thinking-test score out.

---

## 2. Mini-dictionary (the words that'll come up)

- **Model** — a computer program that makes predictions after learning from examples.
- **Predict / regression / target** — "regression" just means *predicting a number* (the test score). The number we're trying to predict is the **target**.
- **Feature engineering** — the *old* approach: a human decides which summary numbers to squeeze out of the glucose (e.g. average glucose, how bumpy it was, how long it was too low). Then the model predicts from those human-chosen summaries.
- **Raw-data learning** — the *new* approach (yours): don't hand-pick summaries; feed the raw glucose sequence to a smart AI and let *it* figure out what matters.
- **Foundation model** — a big AI that already learned general patterns from a huge amount of data, so you can reuse it instead of training your own from scratch. (ChatGPT is a foundation model for *language*.)
- **TSFM** = *Time-Series Foundation Model* — the same idea, but for **time series** (numbers that change over time, like glucose). It already learned the general "shapes" that time-series can take.
- **Chronos** — the specific TSFM we use. It's made by Amazon and it's free to download. Think "ChatGPT, but for sequences of numbers over time."
- **Embedding** — the output of the AI when you feed it the glucose window: a list of numbers (ours has 512 of them) that acts like a **fingerprint of the glucose curve's shape**. We then predict the test score from this fingerprint.
- **R² ("R-squared")** — the report card for predictions. **1.0 = perfect. 0 = no better than just guessing the average score every time. Below 0 = actually worse than guessing the average.** This is the single most important number to watch.
- **Baseline** — the "just guess the average" strategy. Any real model has to beat this to be worth anything.
- **Cross-validation** — a fair way to test a model: train it on some data, test it on *different* data it hasn't seen, and repeat. Prevents fooling yourself.
- **Grouped by subject / leakage** — a crucial fairness rule: **never test the model on a kid it already saw during training.** Otherwise it can "cheat" by remembering that specific kid instead of learning a real glucose→thinking rule. ("Leakage" = accidental cheating like that.)
- **Overfitting** — when a model *memorizes* the practice examples instead of learning the real pattern — so it looks great in practice but fails on new data. A constant danger.
- **Synthetic data** — fake data we generated that has the exact same *shape and format* as the real data, so we can build and test all the software before the real data is available.

---

## 3. The one big thing you MUST understand before the meeting

**Mack already tried the old (feature-engineering) approach, and it basically didn't work.** His models could **not** predict the thinking scores any better than just guessing the average (R² came out negative). And the diagnostics suggest **most of the difference between kids is just "some kids score higher than others in general"** (their personal baseline) — *not* "their blood sugar in the last hour changed their score."

Why this matters for you:
- A fancy AI **cannot invent a signal that isn't there.** So we should **not** promise Liuyi that our approach will suddenly work great.
- What our approach *can* honestly do: **test the new method fairly** against the old one, using the exact same fair rules, and **directly check the "it's just baseline" theory.** That is a legitimate, publishable scientific contribution even if the answer is "still weak."

Saying this out loud in the meeting shows maturity — you understand the science, not just the code.

---

## 4. What we actually built (in plain words)

Think of it as an assembly line for the question "glucose window → predicted score." We built the whole line:

1. **A data reader** that understands the real data's format (one row per game session, with the glucose readings before it and the three scores). We also built a **synthetic-data generator** that fakes this exact format, so the whole line runs *today* without the real file.
2. **The AI fingerprint step** — feeds each glucose window into Chronos and gets the 512-number fingerprint (embedding). We actually downloaded and ran the real Amazon Chronos model to confirm it works.
3. **The prediction + report-card step** — predicts each score and measures R² *fairly* (grouped by kid, so no cheating).
4. **Two versions of the approach** (we call them "arms"):
   - **Arm A** = AI fingerprint + a simple, standard predictor. Fast. This is the quickest fair comparison to Mack.
   - **Arm B** = AI fingerprint + a small trainable neural network. More flexible. **This is the part where Liuyi asked you to "adapt Ben's code"** — Ben is another student whose code we used as a template; we rewired it from his 2-sensor setup to our 1-sensor (glucose-only) setup and switched it to predict a number.
5. **A fair copy of Mack's old approach** — we copied his exact 43 hand-picked glucose summaries into our code (and *verified* they're numerically identical to his), so the comparison is truly apples-to-apples: same everything, only the method differs.
6. **"Knobs" to test different settings**, each of which answers a real question:
   - **Which size of Chronos** works best (bigger isn't always better)?
   - **How much glucose history** to use (2 hours? 3 hours? all of it)?
   - **PCA** — a math trick that shrinks the 512-number fingerprint down to its ~32 most useful numbers. Needed because 512 numbers is too many and causes overfitting. (In our tests it did help.)
   - **Within-subject normalization** — instead of predicting the raw score, predict *how much this kid did better or worse than their own usual score.* This is the direct test of the "it's just baseline" theory from Section 3.
7. **Auto-generated result tables** — every run writes clean tables into the `results/` folder, and there's a plain-English `results/README.md` that ties them together for a report.

**Bottom line:** the machine is fully built and tested. It's parked, waiting for the real fuel (the data).

---

## 5. How this maps to the 4 tasks Liuyi gave you

| Task | Plain meaning | Status |
|---|---|---|
| 1. compute2 environment | Set up the lab's shared computer | ⏸ On hold (Liuyi said focus on the others first) |
| 2. Understand the code + how the AI takes input | Learn the tools | ✅ Done |
| 3. Understand the AI's input format | Figure out exactly how to feed data to Chronos | ✅ Done — Liuyi pointed us to a research paper's code; we studied it **and implemented it** |
| 4. Adapt Ben's code for our case | 1 glucose input, predict a number, 3 scores | ✅ Done (and tested) |

Plus a lot of extra: the full pipeline, the fair comparison to Mack, the knobs, and the auto-reports.

---

## 6. What to TELL Liuyi (you can almost say these word-for-word)

1. "I understood your answers and re-scoped the work: **one input channel (just glucose), one score at a time (three separate models for Grids/Symbols/Prices), the raw-data/AI approach only.**"
2. "I learned how Chronos takes its input by studying the ICML paper's code you recommended — and I **implemented that recipe**, so Task 3 is done in practice, not just in theory."
3. "I **adapted Ben's code** to our single-glucose-channel, predict-a-number case — that's Task 4."
4. "I built the **whole pipeline end-to-end** and confirmed it runs, including downloading and running the real Amazon Chronos model. It's on **synthetic (stand-in) data** for now because we don't have the real file yet."
5. "I also **copied Mack's exact 43 features into our code** (verified identical) so we can compare the AI approach vs his feature approach **fairly, under the same rules**."
6. "I added tools to test model size, history-window length, a dimensionality trick (PCA), and — importantly — a **within-subject test** that directly checks whether the signal is just per-kid baseline."
7. "Everything auto-writes result tables, and there's a plain-English results summary. **The only thing blocking real results is access to the data.**"

## 7. What to ASK Liuyi (the decisions we need)

*(These are in `docs/04_OPEN_QUESTIONS.md` in more detail — here they are in plain words.)*

1. **Can I get the real data file?** (The merged glucose+scores CSVs — they're on your machine, not shared yet.) *This is the one thing stopping real results.*
2. **How much glucose history should count as "before the test," and do we have the full raw glucose stream** or only the pre-clipped piece?
3. **The "Prices" score is backwards** (higher = worse). I'm flipping it — is that right? And what do the three scores actually measure?
4. **Given Mack's approach found little signal, what would count as success** for us — confirming/refuting it fairly, or chasing a specific idea (like only low-blood-sugar moments)?
5. **Which Chronos version** should we standardize on, and **when should I set up the compute2 machine** (Task 1)?

## 8. The honest caveat (worth saying too)

"All my current numbers are from **synthetic (fake) stand-in data**, so they're a **plumbing test, not a scientific result** — they prove the machine works, not that glucose predicts thinking. And since Mack's careful attempt found little signal, I'm not expecting a miracle; the value is a **fair, honest comparison** and a **direct test of the baseline theory**. The real numbers come the moment I get the data."

*(Supervisors trust students more when they don't oversell. This paragraph does a lot of work.)*

---

## 9. Where things live (so you can find stuff)

- **`PLAIN_ENGLISH_SUMMARY.md`** ← you are here.
- **`README.md`** — the technical front page + all the commands to run things.
- **`docs/`** — the detailed writeups:
  - `00_PROJECT_OVERVIEW.md` — the story and the plan.
  - `01_PIPELINE_DESIGN.md` — how the machine works, technically.
  - `02_ROADMAP.md` — what's done and what's next (a checklist).
  - `03_MEETING_BRIEF.md` — a one-pager for the meeting (more technical than this file).
  - `04_OPEN_QUESTIONS.md` — the questions for Liuyi, in detail.
- **`cgm_tsfm/`** — the actual code (the assembly line).
- **`results/`** — the output tables + a `results/README.md` that explains them.

---

### If you remember only 5 things
1. **Goal:** predict a kid's thinking-test score from their recent blood sugar.
2. **Method:** feed raw glucose to an AI (Chronos) instead of hand-picking summaries.
3. **Reality:** the older approach barely worked; the signal looks like per-kid baseline. We won't oversell.
4. **We built:** the entire pipeline + a fair comparison + analysis knobs + auto-reports, tested on stand-in data.
5. **We need:** the real data file to produce real numbers — that's the main ask for Liuyi.
