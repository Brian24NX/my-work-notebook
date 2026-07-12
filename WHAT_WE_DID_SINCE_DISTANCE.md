# What we've built since Distance — the plain-English recap

*A jargon-free walkthrough of the Distance, Productionizing, and Behavior work, for Brian.
Written 2026-07-07, updated 2026-07-09. (Fuller technical version: `~/SIDEKIC_since_distance_deepdive.md`.)*

---

## The big picture first

Think of the whole project as an **assembly line** that takes a raw camera-trap clip and
adds one label at each station:

```
  🎥 clip → [is there an animal?] → [how many?] → [what species?] → [how far away?] → [what's it doing?]
              DETECT                COUNT          SPECIES           DISTANCE          BEHAVIOR
              (done earlier)                                         ← we are here →
```

Everything since "Distance" is the **last two stations** — **Distance** ("how far?") and
**Behavior** ("what's it doing?") — plus **gluing the whole line into one push-button
program** ("Productionizing"). One thread at a time below.

---

## THREAD 1 — DISTANCE: "how far is each animal from the camera?"

**Goal:** for each animal, output how many meters it is from the camera (the lab uses this
to estimate animal abundance). **Why it's hard:** our footage is night-vision (grayscale,
grainy) in dense forest — you can't just eyeball distance.

### What we tried that failed (and why)
- An off-the-shelf "how far is this?" AI (**depth estimation**). It basically **guessed.**
  Analogy: it was trained on normal daytime color photos; showing it our grayscale
  night-vision footage is like asking someone who's only seen color photos to judge distance
  from a black-and-white X-ray.
- One universal formula for all cameras. **Also failed** — every camera is aimed slightly
  differently, so no single formula fits all.

### What worked: give each camera its own cheat-sheet
Key insight: **where an animal's feet sit in the picture tells you how far it is.** Feet near
the bottom = close; feet up toward the middle = far (like how, in any photo, things at the
bottom are near you).

```
   ┌─────────────────────┐
   │        🐗  ← far     │   feet high in frame  = ~8 m
   │      (higher up)     │
   │   🐗  ← near         │   feet low in frame   = ~1–2 m
   └─────────────────────┘
```

For each of the 64 cameras where the lab had already hand-measured some real distances, we
built a little **per-camera formula**: "feet at *this* height → *this* many meters." Result:
**accurate to about 1 meter (0.95 m)** — which is roughly the best physically possible (the
animal's feet wiggle enough that no method does much better).

**How we checked it's honest:** "leave-one-out" — hide one measurement, predict it from the
others, see how close. Like quizzing yourself with a flashcard you haven't peeked at.

### Colleen's idea → the "camera got bumped" detector
Colleen pointed out: sometimes an animal **bumps the camera** and the whole picture jolts
(like someone knocking your phone mid-video) — the distance then is worthless. So I built a
detector that spots a sudden whole-frame jolt and **flags that clip as "animal reacted" and
blanks the distance** — but *keeps the clip*, because "an animal touched the camera" is itself
interesting. (This also filled in a column we'd been leaving empty.)

### The "which cameras can we trust?" flag
Some cameras are aimed poorly (mounted low, staring flat into bushes), so their distance
guesses are shaky. I made an automatic **"don't fully trust these" list: 14 of 64 cameras.**
The cause turned out to be **ground slope** (on a hillside the view is squished), not bumping
— Jake later confirmed the bumped clips were already thrown out.

### Danni's big idea — and an honest "it didn't work"
Danni asked: *instead of hand-measuring, can we use the "setup videos"?* (Each camera has a
video where a fieldworker lays **measuring tapes** on the ground at 1-meter marks.) If that
worked, we'd never need hand-measuring. I tested it thoroughly: **it works on the tidy setup
videos but falls apart on messy ones** (tapes buried in leaves, distance signs too blurry to
read). Not reliable enough to replace hand-measuring — so we kept the hand-measured
cheat-sheets as the deliverable.

**One honest moment:** while testing this, I made a measurement mistake — I let the "answer
key" leak into the "study material," making one number look better than it was. **Danni
caught it.** I retracted that number and fixed the code so it can't happen again. (Normal
science — the point is we don't fool ourselves.)

### Jake's field info → the shortcut that finished the job (all 100 cameras)
Jake told us the cameras are set up in a **standardized way** (fixed height ~1 m, fixed angle).
That was the key: if every camera is installed the same, **one geometry formula works for all of
them** — so for the 36 cameras we never hand-measured, we could compute distance from the shared
setup, **with no fieldwork**. I tested it honestly (fit the formula on 63 cameras, predict the
64th it never saw): **~1.28 m accuracy with zero measuring** (~1.15 m on normally-sited cameras).
So I ran it on those 36 cameras → **14,181 distances**, and **now the whole archive (all 100
cameras) has distances.** I sanity-checked them against the measured ones: same typical distance
(4 m median), and 97% of animals land in the range the formula was built on. ✅

**Danni's follow-up** ("how many hand-measurements does a *new* camera need to stay near 1 m?"):
I answered it from the data — **about 5–6 per camera** gets you to ~1 m; with only 2–4, leaning on
the shared formula keeps it ~1.1–1.2 m. Concrete guidance for future deployments.

**The review:** Danni reviewed the distance code and **approved it** — with a few sharp notes I
fixed (a possible crash, an over-sensitive flag, a subtle validation tweak); a code-review bot's
notes too.

**Distance bottom line:** ✅ **done for all 100 cameras** — 0.95 m on the 64 measured, ~1.28 m on
the 36 computed — with trust-flags and reaction-flags. **PR #26 is approved and ready to merge.** 🏁

---

## THREAD 2 — PRODUCTIONIZING: "make it one push-button program"

Individually the stations work; **productionizing = gluing them into one program** you run
on a clip and it spits out the answer table (species + count for every clip). It already
produced the deliverable: **all 14,577 clips processed.**

**Danni's review comment:** our program had a *home-made copy* of one piece (the bit that
tracks "which blob is which animal across frames"). She said: use the **official, tested
version**, not a duplicate. Fixed — plus a few safety cleanups a code-review bot flagged.

**Status:** ✅ done, waiting for Danni to click "approve." (It overlaps a little with work
Noah did — Danni will decide how they fit together.)

---

## THREAD 3 — POSE / BEHAVIOR: "what is the animal doing?"

**Goal:** label the behavior — standing, running, sniffing, looking around, approaching.

**The hard part — a lopsided problem:** **95% of the time the animal is just standing.** The
interesting behaviors are rare. Analogy: learning a language where 95% of every sentence is
the word "the" — it's hard to learn the rare, meaningful words.

**Your decision:** Noah already built a behavior model, but you chose to build your **own**
version and let the best one win — on your own branch, so it touches no one else's work.

### First I built a fair scoreboard
Before trying any tool, I built a **referee**: a scoring program that grades any behavior tool
the same way, on the same held-out cameras, so "who's better" is objective. And I measured the
**"lazy baseline": if you just guess 'standing' every time, you score 0.157.** Any real tool
must beat 0.157 to be worth anything.

### Danni suggested 4 tools; we started with BehaveAI
Danni named four (BehaveAI, PoseR, SLEAP, AniMo). I researched all four and picked
**BehaveAI** first because it's the most *different*: it spots animals by their **movement**
(turning motion into colored trails), not their shape. That suits us because **a running
animal "lights up" even in the dark.**

**Quick sanity check first:** before installing anything, I tested whether the "motion →
color" trick even shows an animal in *our* night footage. It does — a running duiker lit up
vividly. (Caveats: it can't detect a *standing* animal — no motion — and very far animals are
faint. So we use it to catch the *active* behaviors and keep "standing" as the default.)

### The clever part — running it "for you" without a laptop
BehaveAI is a **click-around app** that needs a screen and you hand-drawing boxes. Our cluster
is a **screenless server**, so normally you'd run BehaveAI on your laptop and click through it
yourself. You asked "can you do it for me?" — so instead of the app, **I rebuilt its *method*
to run automatically on the server:** turn clips into motion-color frames → auto-draw the
boxes from the motion → auto-label them from the ground-truth we already have (no
hand-drawing) → train a detector → test → score vs the 0.157 floor.

### The result: it works — and we made it better, then made it *honest*
The first trial **beat the 0.157 bar** — the motion-based detector scored about **0.35** (~2.2×
the lazy baseline). Good, but it **over-fired**: it cried "running!" at wind-blown leaves. So I
did a round of **precision fixes** — showing it examples of "motion that is NOT a behavior" (wind,
a standing animal) labeled as *nothing*, training only on the animal's own motion, and requiring a
behavior to show up across *several* frames (not a one-frame fluke).

**Then I caught a bug in our own scoreboard** (the thing that grades the model). Its way of
splitting cameras into "study" vs "exam" sets leaned on a label that was blank for one clip, which
quirkily let a sliver of one camera sit on both sides. I fixed it to split by the actual camera,
then re-ran a clean **head-to-head**: plain model vs. the precision-fixed model, on the fixed
scoreboard.

**Clean result: the precision fixes won.** Overall quality up ~7%, the strongest behavior (look)
clearly better, and the improved model is **much better behaved** — it works at a normal
sensitivity, where the old one only worked cranked to an extreme. Honest final number: **macro-F1
0.359 (~2.3× the baseline)** on a trustworthy, leak-free scoreboard. The stubborn weak behaviors
(run / rest / approach) are simply **starved for examples** — "rest" has only 7 test clips, so no
method will shine there.

*(Same "less flattering but trustworthy" pattern as before: fixing the scoreboard nudged the
number from 0.384 down to the honest 0.359 — the earlier figure was on the flawed split.)*

**The other 3 tools** (PoseR / SLEAP / AniMo) are the *skeleton-based* route — the same family as
**Noah's model, which has since been merged into the main pipeline.** So our motion-based line is
the genuinely *different* approach; whether to also pursue the skeleton route is a next-step choice.

---

## Where we are right now
| thread | status |
|---|---|
| **Distance** | ✅ **done, all 100 cameras** (0.95 m measured / ~1.28 m computed); **PR #26 approved, ready to merge** |
| **Productionizing** | ✅ done, review fixed; **PR #23 waiting for Danni** |
| **Behavior** | ✅ **validated & honest** — motion-based detector, macro-F1 0.359 (~2.3× baseline); committed on your branch |

**What's left is mostly people/decisions, not blocked work:**
- **Merge** PR #26 (approved) and nudge Danni on #23.
- **Your behavior branch → a PR — there's a wrinkle:** while our line was in progress, **Noah's
  skeleton-based behavior work merged into main**, and both use a file named `eval_behavior.py`
  (his grades *his* model; ours is a *tool-agnostic* scoreboard that grades *any* model the same
  way). Also, our branch was started off a very old copy of the project. So turning our line into a
  clean PR needs a quick decision on how the two fit together — I've laid out the options in my
  message. **I did not push anything.**
- **Behavior next (optional):** more labeled examples for the rare behaviors, or try the skeleton
  route — or treat this as a clean stopping point.
