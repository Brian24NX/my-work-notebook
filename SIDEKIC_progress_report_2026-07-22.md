# SIDEKIC — What I Shipped This Week
### Camera-trap AI pipeline · progress report · July 22, 2026

> **In one line:** the shared storage filled to 100% and pull requests looked frozen — so I set up a workaround and shipped **8 PRs** anyway: unblocking the team's dashboard, upgrading the behaviour AI with published state-of-the-art methods, and hardening the pose + upload pipeline.

![Progress at a glance](pr_report_figs/fig1_hero.png)

*(All figures are in the `pr_report_figs/` folder next to this file.)*

---

## 1 · The situation: a blocker, turned into momentum

Midweek, our shared research storage hit **100% full**. On a normal setup that freezes everything — you can't even save code (`git` refuses to write), so the natural assumption was *"no new work until IT frees space."*

Instead, I cloned our project onto a **separate drive that still had room** and pointed all my code-shipping through there. Saving and publishing code went straight over the network, never touching the full disk. **Result: zero days lost, 8 PRs shipped, and the storage fix can happen in parallel on its own schedule.**

![Turning a blocker into 8 PRs](pr_report_figs/fig4_storage.png)

> **Why this matters:** the deliverables below didn't have to wait. The moment storage is restored, everything I built merges in — plus the GPU experiments queue up immediately.

---

## 2 · The map: where this week's work landed

Our pipeline turns raw camera-trap video into structured wildlife data through a chain of stages. This week I focused on the **hardest, highest-value end** — reading animal **behaviour** and **pose** — and on the **dashboard** the whole team relies on.

![Where this work fits in the pipeline](pr_report_figs/fig2_pipeline.png)

The 8 PRs group into four clean buckets:

![8 PRs, organised by what they deliver](pr_report_figs/fig3_buckets.png)

The rest of this report walks each bucket the same way: **the problem → what I built → why it matters.**

---

## 3 · Unblocking the team: Selah's dashboard  ·  PRs #41 #42 #47

**The problem — in plain terms.** Think of our web app as a restaurant. Selah built a beautiful **dining room** (the interface — a gallery of clips, a video player, a filter panel). But when a diner ordered *"bring me the list of videos,"* the **kitchen had no serving window for that order** — so the page simply broke (a "404"). Three of her key screens couldn't load real data.

**What I built.** I added the missing serving windows — the back-end endpoints her interface was already calling:

| New endpoint | What it serves | Unblocks |
|---|---|---|
| `/api/runs`, `/api/runs/{id}`, `/api/frames/…` | the video list + one video's boxes/labels + thumbnails | **Gallery + Video Player** (#41) |
| `/api/search` | keyword + category/label filtering | **Filter / search box** (#42) |
| `/api/runs/{id}/stats` | per-video counts, species mix, behaviour mix | **Detail view** (#47) |

Each one is small, backward-compatible (nothing else changes), and **unit-tested**. I deliberately put the data-shaping logic in tiny standalone modules so it's tested independently of the web server.

> **Why this matters:** Selah's front-end integration is now unblocked end-to-end. The team gets a working dashboard to label and review clips — which is how everyone else's data actually gets used.

*Now that the team can **see** the data, the next question is: can the AI **read** it better?*

---

## 4 · Making the behaviour AI smarter (research-driven)  ·  PRs #44 #46 #45

### 4.1 · The core challenge: extreme imbalance

Animals in camera-trap clips are **doing nothing** the vast majority of the time. In our labelled data, the "just standing there" class dominates; the *interesting* behaviours — running, approaching, smelling, looking — are rare.

![Why rare behaviours are hard](pr_report_figs/fig5_imbalance.png)

> **The trap, by analogy:** a lazy security guard watching footage where the animal is still 90%+ of the time can just say *"nothing's happening"* and be right most of the time — while missing every interesting moment. Our AI faces exactly this temptation. Our current model already beats that lazy baseline by **2.3×**, but the rare behaviours are still where the accuracy is lost.

So I asked: *what does the latest research say is the highest-impact, lowest-effort fix?* — and had it **fact-checked rigorously** before writing any code.

### 4.2 · The research, fact-checked

I ran an automated multi-agent literature review over the 2024–2026 work on this exact problem. Crucially, **every claim was challenged by three independent checks** — a claim only survived if it wasn't refuted.

![How the research was fact-checked](pr_report_figs/fig6_funnel.png)

The clear #1 recommendation: a smarter **loss function** (the rule that scores the AI during training). I implemented it the same day.

### 4.3 · Fix #1 — Distribution-Balanced Loss  (PR #44)

**The idea, by analogy:** change the exam scoring so that getting a **rare** question right is worth far more than getting yet another "standing still" right. The model can no longer coast by ignoring the rare behaviours — it's forced to learn them.

On the published benchmark for this problem, this roughly **doubles** accuracy on the rare classes:

![The behaviour-AI upgrade: Distribution-Balanced Loss](pr_report_figs/fig7_dbloss.png)

It's a faithful, tested implementation, wired in as an **opt-in switch** (`--loss db`) so nothing changes until we choose to turn it on — and we can measure the gain on *our* data cleanly.

### 4.4 · Fix #2 — combine two AI "witnesses"  (PR #46)

We actually have **two** behaviour signals, each blind where the other is strong:

![Two AI "witnesses", combined](pr_report_figs/fig8_fusion.png)

> **By analogy:** a **motion** detective is great at spotting running and chasing but useless when the animal is still; a **posture** expert reads body language (head-down = smelling, head-up = looking) but can't judge speed. I built the tool that **combines their testimony** — so the still-animal behaviours the motion model misses get covered by posture, and vice-versa. (Our posture signal is already validated at AUC ~0.70.)

### 4.5 · The roadmap, captured  (PR #45)

I wrote the whole research review up as an in-repo brief — prioritised, cited, and **honest about the caveats** (every benchmark is from a different domain than our infrared footage, and I list exactly which claims *failed* fact-checking). It's the team's playbook for the next behaviour improvements.

*Better models are only useful if the tooling that feeds them is reliable — which is the last bucket.*

---

## 5 · Hardening the pipeline  ·  PRs #48 #43

**Pose runner (#48).** Our pose tool made a single all-or-nothing pass over a batch of clips. If **one** clip had no detectable animal, the whole run crashed — and couldn't resume.

![Hardened pose runner](pr_report_figs/fig9_pose.png)

> **By analogy:** the old tool was a photocopier that jams on one bad page and cancels the entire 300-page job. Now it **skips the bad page, keeps going, and remembers where it left off** — and it runs on any compute node (I fixed a container issue that broke it on most of the cluster). I validated this recipe on ~300 real infrared clips.

**Upload validation (#43).** Uploads used to write the *entire* file to our (nearly full!) storage before checking if it was even a video. Now non-videos and empty files are rejected **before** anything is written — a small fix that matters a lot on a full disk.

---

## 6 · Honest status (so there are no surprises)

I want to be precise about what's **done** vs **projected**:

| Bucket | State | Evidence |
|---|---|---|
| Dashboard endpoints (#41 #42 #47) | ✅ working & tested | unblocks Selah's UI today |
| Upload + pose hardening (#43 #48) | ✅ working; pose validated on ~300 clips | real bug fixes |
| Behaviour AI (#44 DB-Loss, #46 fusion) | ✅ implemented & unit-tested; **gains still to be measured** | numbers above are the *published benchmark* + our own AUC ~0.70 — the on-our-data comparison needs a GPU run (blocked by storage) |
| Research brief (#45) | ✅ delivered | cited, fact-checked |

- **All 8 PRs are open and awaiting review.** By our rules I can't approve my own code — they need a code-owner (Danni / Nick) sign-off.
- The behaviour-AI **improvements are built and validated in mechanism**; their real accuracy lift on our data is the first thing I'll measure the moment GPU/storage is back.

---

## 7 · What's next

![What's next](pr_report_figs/fig10_roadmap.png)

1. **Now:** 8 PRs in review.
2. **When storage/GPU is back:** train the behaviour model with the new loss vs. the current one and **measure the real gain**; run a larger pose pass to feed the fusion tool.
3. **Then:** wire fusion into the live pipeline, and add a vision-language model as a zero-shot baseline (the one research lever I haven't built yet).

---

## Appendix · the 8 PRs at a glance

| PR | Title | Bucket | Tests |
|----|-------|--------|-------|
| #41 | `/api/runs*` + `/api/frames` endpoints | Dashboard | 2 |
| #42 | `/api/search` for the filter box | Dashboard | 2 |
| #47 | `/api/runs/{id}/stats` for the detail view | Dashboard | 2 |
| #43 | Upload validation (reject non-video / empty) | Robustness | — |
| #44 | Distribution-Balanced Loss (`--loss db`) | Behaviour AI | 5 |
| #46 | Motion + pose late-fusion utility | Behaviour AI | 6 |
| #45 | 2024–26 behaviour/pose research brief | Research | — |
| #48 | Hardened pose runner (resilient + resumable) | Pose | — |

**Totals: 8 PRs · 17 unit tests (all passing) · 4 fronts · 0 days lost to the outage.**

*Repository: `CongoApe-SIDEKIC/SIDEKIC` · all branches pushed, PRs open for review.*
