# What we've built since Distance — the plain-English recap

*A jargon-free walkthrough of the Distance, Productionizing, and Behavior work, for Brian.
Written 2026-07-07. (The technical version lives in `~/SIDEKIC_HANDOFF.md`.)*

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

### Jake's field info → a possible future shortcut
Jake told us the cameras are set up in a **standardized way** (fixed height ~1 m, fixed
angle). If we know the exact setup numbers, we might compute distance from geometry alone —
no hand-measuring — for the cameras we haven't measured. **He's sending the setup protocol;
that idea is on hold until it arrives.**

**Distance bottom line:** ✅ done and delivered for the 64 measured cameras (~1 m accuracy,
with trust-flags and reaction-flags). Unmeasured cameras are a "later, if wanted" item.

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

### It's running right now
That whole thing is **submitted and cooking on the cluster (job 1993844)** — no install
needed (the server already had the engine). In a few hours it'll tell us **whether
motion-based behavior detection beats the 0.157 bar.** I'm watching it and will report the
number automatically.

**The other 3 tools:** scoped and queued behind BehaveAI's result — PoseR (same approach Noah
used, needs skeleton-tracking first), SLEAP (needs hand-labeling), AniMo (a "generate fake
training data" idea for later).

---

## Where we are right now
| thread | status |
|---|---|
| **Distance** | ✅ done (~1 m), reaction + trust flags; waiting for Danni to merge PR #26 |
| **Productionizing** | ✅ done, review fixed; waiting for Danni to merge PR #23 |
| **Behavior** | 🔄 the BehaveAI-style trial is **running now** — real score vs 0.157 coming soon |

**Waiting on people, not on me:** Danni (approve the two finished PRs), Jake (send the camera
setup protocol → unlocks the distance shortcut), and the behavior trial to finish.
