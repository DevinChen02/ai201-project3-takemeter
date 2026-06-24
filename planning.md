# TakeMeter — planning.md

A text classifier for **discourse quality on r/nba** — separating substantiated argument from confident noise from raw emotion. This document locks the design decisions that constrain everything downstream: data sources, annotation effort, the model's learning task, and what evaluation actually tells us.

---

## 1. Community

**Choice: r/nba.**

**Why this community.** The distinction I want to model — *argument vs. assertion vs. reaction* — isn't something I have to impose on r/nba; it's already the community's native vocabulary. Users openly call each other's comments "bad takes" or praise a "great breakdown," so the labels reflect a quality axis the community itself polices. Practically, discourse is overwhelmingly text (post-game threads, daily discussion, analysis posts), so collecting 200+ public comments via the Reddit API is straightforward.

**Why it's a good fit for classification (varied enough to be interesting).** A single event — a buzzer-beater, a trade, an injury — reliably generates *all three* of my classes side by side in the same thread. That co-occurrence is what makes the task non-trivial and the boundaries observable rather than theoretical: the same topic ("is SGA the MVP?") appears as a stat-driven breakdown, as a bare hot take, and as an emotional one-liner, so the model has to learn *discourse form*, not *topic*. The community also sits in a genuine, ongoing tension between debate culture and engagement bait, which means the high-quality/low-quality signal is real and contested, not manufactured.

---

## 2. Labels

Three labels, partitioned by the comment's **dominant function**.

### `analysis`
*The comment makes a structured argument supported by specific, verifiable evidence (statistics, on/off splits, tactical/film observation, or historical comparison) such that the reasoning would survive a fact-check.*

> "Calling this a down year for Jokić is just not watching the games. His assists are up to ~11 a night while his usage dropped a few points — he's creating *more* on *lower* volume. The Nuggets are top-5 in half-court points-per-possession with him on the floor and bottom-10 when he sits. That's not decline, that's a roster problem around him."

> "The Knicks' switch-everything scheme works against guard-heavy teams, but it's exactly why Boston cooked them — switch a center onto Tatum on the perimeter and you've got a guard on Porziņģis in the post, so you lose both matchups. When they stopped switching in the 2nd half the lead went from 18 to 6. The personnel can't execute that scheme against this much length."

### `hot_take`
*The comment states a bold or contrarian opinion with confidence but no genuine support — it asserts rather than argues, and any "evidence" is vague, cherry-picked, or decorative.*

> "Wembanyama is already a top-5 player in the league. I don't care what the record says — watch him for ten minutes and tell me there are four guys you'd rather have."

> "Trae Young is the most overrated player of the last decade. Pads stats on a bad team and vanishes the second the games actually matter. Screenshot this — he never makes a conference finals."

### `reaction`
*The comment is an immediate emotional response to a specific event (a play, game, trade, or injury), expressing a feeling rather than making a claim or argument.*

> "WHAT A SHOT. I'm actually shaking. Two guys in his face, down 1, 0.4 on the clock — I cannot believe that went in. THIS is why I watch basketball."

> "Been a Grizzlies fan for 15 years and watching Ja go down like that again genuinely made me sick. Just gutted. Don't even have words right now."

---

## 3. Hard edge cases
 
**The genuinely ambiguous case is `analysis` vs. `hot_take` — the "decorative stats" problem.** A confident opinion *with a number attached* can masquerade as analysis without doing any real reasoning:
 
> "SGA is the MVP and it's not close. 32-6-6 on 53% as the engine of the 1-seed. The numbers don't lie."
 
This cites real stats but delivers them as a mic-drop, not as an argument. This is the boundary where annotation errors — and later, model errors — will concentrate.
 
**How I'll handle it during annotation — primary decision rule:** strip the opinion framing and ask whether a genuine argument remains.
- Evidence is *load-bearing* (you could reconstruct the argument from the facts alone) → **`analysis`**.
- Evidence is *decorative* (one cherry-picked stat, or "the numbers don't lie" standing in for reasoning) → **`hot_take`**.
- *Applied:* strip "it's not close / numbers don't lie" from the SGA comment and you're left with a one-line stat dump → **`hot_take`**. Strip the framing from the Jokić comment and a multi-step argument survives → **`analysis`**.
**Two secondary boundaries, with rules:**
- `hot_take` vs `reaction`: label by the comment's *primary payload*. An emotional state triggered by an event ("I'm shaking") → `reaction`; an assertion about the world ("the league is rigged") with emotion as wrapper → `hot_take`.
- `reaction` vs `analysis`: a vent that names a *mechanism with at least implicit evidence* leans `analysis`; pattern-language used purely as an intensifier ("it ALWAYS happens, so frustrating") with no specifics leans `reaction`.
**Multi-function comments** (a comment that reacts, then asserts, then backs into analysis) are labeled by the clause that would survive a one-line summary. Genuinely balanced cases break toward the higher-evidence label *only if the evidence is non-decorative* (`analysis` > `hot_take` > `reaction`), so weak-evidence posts don't inflate `analysis`. These rules go into a written annotation guide *before* labeling starts.
 
### Documented cases that gave me pause during annotation (Milestone 3)
 
These are the real judgment calls from labeling, with the decision and the rule that settled it. All are also flagged in the `notes` column of `labeled_data.csv`.
 
1. **"SGA is the MVP and it's not close. 32-6-6 on 53% as the engine of the 1-seed. The numbers don't lie."** — Candidate labels: `analysis` (cites stats) vs `hot_take`. **Decided `hot_take`.** This is the canonical decorative-stats case: strip "it's not close / numbers don't lie" and you're left with a one-line stat dump, not a structured argument. Real stats ≠ analysis; the stats have to do argumentative work.
2. **"There is NO WAY that was a foul. The refs cost us this game... This league is rigged, I'm done."** — Candidate labels: `reaction` (in-the-moment heat) vs `hot_take`. **Decided `hot_take`.** The emotion is the wrapper but the *payload* is an assertion about the world ("the league is rigged"). Primary-payload rule: claim is load-bearing → `hot_take`.
3. **"That 4th-quarter collapse was so predictable. Every time they go iso-heavy down the stretch it falls apart and it happened AGAIN tonight. So frustrating."** — Candidate labels: `analysis` (names a tactical pattern) vs `reaction`. **Decided `reaction`.** It labels a cause ("iso-heavy") but gives no specifics — no possessions, no numbers, no alternative — and the dominant register is frustration. Pattern-language as a vent intensifier → `reaction`. This is the closest call in the set; a reasonable annotator could go `analysis`, which is exactly why it needed a written rule.
4. **"Jokic is washed in the clutch. He's shooting under 40% in the last two minutes of close games this year."** — Candidate labels: `analysis` vs `hot_take`. **Decided `hot_take`.** A single cherry-picked clutch split (tiny sample, no context) deployed to support a sweeping claim. Cherry-picked, decorative evidence → `hot_take`.
5. **"I'm so tired of the 'he can't shoot' takes — he's at 38% from three this month on real volume, go look it up."** — Candidate labels: `reaction`/`hot_take` (irritated, argumentative tone) vs `analysis`. **Decided `analysis`.** The opposite of the SGA case: strip the irritation and a genuine, checkable evidence-based point remains (a real stat over a real sample, used to rebut a claim). Tone is emotional; *function* is argument → `analysis`.
6. **"Luka is a defensive disaster, full stop. His team's defensive rating is several points worse with him on the floor, case closed."** — Candidate labels: `analysis` (on-off number) vs `hot_take`. **Decided `hot_take`.** Cites an on-off-ish figure but with zero context (lineups, matchups, scheme) and as a conversation-ender. The number is decoration on a pre-formed opinion, not the basis of reasoning → `hot_take`.
**Pattern across the hard cases:** nearly all of them live on the `analysis`↔`hot_take` boundary, and the deciding question is always the same — *does the evidence survive stripping the opinion framing?* This confirms the Milestone 2 prediction that this is the boundary where model errors will concentrate, and it's the boundary the confusion matrix in §5 should be read against.

---

## 4. Data collection plan

**Source.** r/nba via the official Reddit API (PRAW). I'll pull *comments*, not just top-level posts — the substantive discourse lives in comment threads. Filter out `[deleted]`/`[removed]`, bot comments, and obvious copypasta/jokes (these get excluded from the dataset rather than forced into a sincere label, since sarcasm-vs-sincere is a separate, harder problem). De-duplicate near-identical comments.

**Stratified sampling to fight imbalance from the start.** Left to a naive scrape, `reaction` will dominate (game-thread one-liners are the highest-volume comment type). So I sample by thread type:
- Analysis-flaired posts, "effortpost"/OC threads, and serious discussion threads → richer in `analysis` and `hot_take`.
- Post-game and game threads → richer in `reaction`.
- For the rare classes, I'll also do *targeted candidate mining*: heuristically surface likely-`analysis` comments (those containing stat patterns — "per game", "%", "on/off", "PPP") and then human-verify each, rather than assuming the heuristic is the label.

**Counts.** Target **~225–250 labeled comments**, with a **hard floor of ~65 per class** so every class is both learnable and evaluable. (A class below ~40–50 examples won't train or evaluate reliably.)

**If a label is underrepresented after 200 examples:**
1. **Targeted oversampling at the source** — go back to the thread types and the candidate-mining heuristic most likely to contain that class, and collect to the floor.
2. **If it's still rare** after focused collection, that's evidence the label is too narrow — **merge it or redefine the boundary** rather than ship a class the model can't learn.
3. **Training-time vs. eval-time imbalance are handled separately.** If the *natural* distribution stays skewed, I address it in *training* (class weights or resampling) but **evaluate on the natural distribution** (and report both) so the metrics reflect real-world performance, not a re-balanced fiction.

**Splits & annotation rigor.** Stratified train/val/test by label. The **test set is collected from a different time window** than train/val to catch drift and prevent leakage from same-thread near-duplicates. I'll **double-annotate a 50–100 comment subset** to measure inter-annotator agreement (see §5).

---

## 5. Evaluation metrics

**Why accuracy alone is misleading here.** The classes are imbalanced and the errors are not equally costly. If `reaction` is ~55% of the data, a degenerate model that predicts `reaction` for *everything* scores **~55% accuracy** while being completely useless — its macro-F1 would be ~0.24. Accuracy lets the majority class hide total failure on the two classes I actually care about.

**The metric suite I'll report:**

- **Macro-F1 as the headline number.** It averages per-class F1 with equal weight regardless of class frequency, so a model can't win by nailing the majority class while ignoring the minority ones. This is the aggregate that actually tracks "did the model learn all three distinctions."
- **Per-class precision, recall, and F1.** The aggregate hides the rare-but-important `analysis` class. I need to see each class on its own — especially whether `analysis` recall and precision are usable, since that's the class a real tool would act on.
- **Confusion matrix.** This is essential *for this specific task* because the whole design hinges on the `analysis`↔`hot_take` boundary. The matrix tells me *where* errors live: errors concentrated on `analysis`↔`hot_take` are the *expected* hard boundary (and a sign the labels are otherwise clean), whereas `reaction` leaking into both classes would mean the taxonomy isn't working.
- **Baselines for reference.** Metrics are meaningless in isolation, so I report against (a) a **majority-class baseline** and (b) a **TF-IDF + logistic-regression baseline**. Macro-F1 of 0.60 means one thing if the TF-IDF baseline gets 0.40 and a very different thing if it already gets 0.62.
- **Inter-annotator agreement (the human ceiling).** Because the boundaries are partly subjective, the model's achievable performance is *bounded by how consistently humans apply the labels*. On the double-annotated subset I compute two numbers: **Cohen's κ** as the agreement statistic, and a **human macro-F1** — scoring one annotator against the other as if the second were gold — which puts the ceiling on the *same scale* as the model's macro-F1 so the Tier-2 comparison in §6 is well-defined (comparing a macro-F1 directly to a κ value would be a scale mismatch). κ also acts as a **trust gate on the labels: if κ < 0.60 the boundary is too noisy to grade a model against, so I revise the annotation guide and re-label before reporting any model numbers.** Reporting model performance *without* this ceiling would make the numbers uninterpretable — a model at 0.75 macro-F1 is excellent against a human macro-F1 of ~0.80 and mediocre against ~0.95.

**Task-specific weighting.** I'm anchoring the project to a concrete downstream use: a reader-facing **"show me the analysis" quality filter** that surfaces high-substance comments. That makes `analysis` the *actionable* class and **precision-on-`analysis`** the metric that matters most — a false positive shows a user a hot take dressed as analysis, which erodes trust in the tool. (For a different use case — a moderator-facing "flag low-effort hot takes" tool — the priority would flip to **recall-on-`hot_take`**, since a missed hot take is worse than a false alarm a mod can dismiss. I name the use case explicitly so the metric priority is a decision, not an accident.) If the filter uses a confidence threshold, I'll also do a brief **calibration** check (reliability diagram / ECE) so the threshold means what it says — a reported diagnostic, **not** a pass/fail success gate.

---

## 6. Definition of success

Success is tiered, and every target is **relative to the measured human ceiling** (the human macro-F1 from §5) through a **deterministic rule fixed in advance** (e.g., Tier 2 = *ceiling − 7*): the agreement study fills in the ceiling number, not the pass line. A model can't be meaningfully asked to beat humans at a partly-subjective task — and it also can't be graded against a target I get to pick after seeing its score.

**Tier 1 — Proof of concept (research-interesting).** Macro-F1 beats *both* the majority baseline and the TF-IDF baseline, with the **bootstrap 95% CI on each gap excluding 0** (a CI rather than an eyeballed "clear margin," since the test set is small). And the confusion matrix shows errors are **concentrated on the `analysis`↔`hot_take` boundary** — operationally, **≥ 60% of all off-diagonal (misclassified) mass falls in the two `analysis`↔`hot_take` cells**, while `reaction`'s combined leakage into the other classes stays below its own recall. This shows the model learned discourse form, not noise.

**Tier 2 — Genuinely useful.** Macro-F1 lands **no more than 7 points below the human macro-F1 ceiling** (target = *ceiling − 7*, fixed once the agreement study reports the ceiling — one number, not a 5–10 point range to negotiate after the fact), with **no class collapsing** (**every class recall ≥ 0.50**, so no label is effectively ignored). At this point the classifier is roughly as consistent as a single human annotator across all three categories.

**Tier 3 — Good enough to deploy in a real community tool.** Deployment has an *asymmetric, precision-weighted* bar driven by the use case: a tool that surfaces low-quality content labeled as high-quality is **worse than no tool**, because it actively damages user trust. So for the "show me the analysis" filter I require:
- **`analysis` precision ≥ 0.80 while retaining ≥ 0.60 recall on `analysis`** — the recall/coverage floor is essential, because precision alone is trivially gamed by abstaining on everything but the one or two easiest comments. Within the floor, recall can be traded down for precision, but not to zero.
- **Threshold chosen on validation, then applied unchanged and reported once on the time-separated test set** — so 0.80 isn't in-sample optimism from tuning on the data I grade against. "Low-confidence" is a concrete probability cutoff fixed at the same time.
- Because `analysis` is small in the **held-out, time-separated** test set, I report a **bootstrap 95% CI on precision and require the point estimate ≥ 0.80 with the lower CI bound ≥ 0.70**, so a single test comment can't flip the verdict and the number reflects performance under drift rather than in-sample optimism.
- A **human-in-the-loop / abstention guardrail**: predictions below the confidence cutoff are routed to "unclassified" rather than force-labeled, and any moderation-facing action keeps a human in the loop. This operates *within* the coverage floor above, so it can't be used to inflate precision past the bar.

**What I would *not* accept as deployable:** a strong aggregate macro-F1 that's secretly carried by the `reaction` class while `analysis` precision sits at, say, 0.55 — that tool would mislabel nearly half of what it promotes and would erode the community's trust faster than it adds value. The aggregate looking healthy is not sufficient; the actionable class has to clear its own bar.

---

## 7. AI Tool Plan

This project generates no code, so AI tools assist *judgment*, not implementation. They show up in exactly three places — **before** annotation, **during** annotation, and **after** evaluation — and in all three the rule is the same: **AI proposes, I dispose.** Nothing an AI produces enters the gold labels, the inter-annotator agreement subset, or the time-separated test set without human verification, because those are the very artifacts that must stay independent for §5's human ceiling and §6's success bars to mean anything.

### Label stress-testing (before annotation)

**The move.** Before I label a single real comment, I hand the AI my three definitions (§2) and the hard-edge-case write-up (§3) and ask it to generate **5–10 synthetic comments that deliberately sit on the `analysis`↔`hot_take` boundary** — the "decorative stats" trap from §3 — plus a couple on the softer `hot_take`↔`reaction` and `reaction`↔`analysis` seams.

**Why, and the pass/fail test.** A definition that reads clean in prose can still be unfalsifiable in practice, and adversarial boundary examples are the cheapest way to find that out. The test is blunt: I apply my own §3 decision rule (*"strip the opinion framing — is the evidence load-bearing or decorative?"*) to each generated post. If I can classify all of them cleanly and confidently, the definitions hold. If even two or three leave me genuinely stuck, that same ambiguity *will* recur hundreds of times once I'm annotating for real — so I tighten the offending definition or add a tie-breaker rule to the annotation guide **now**, before annotating 200 examples, and re-run the generation against the revised rules until the boundary posts classify cleanly.

**Verification / guardrail.** These synthetic posts are a *diagnostic for the definitions only*. They are **never added to the dataset** — model-written text would contaminate an otherwise organic, human-authored corpus. Their sole output is edits to the written annotation guide.

### Annotation assistance (during annotation)

**Decision: yes, but strictly as a triage draft, and only on part of the data.** I'll use an LLM to **pre-label** the bulk training/validation pool, then review every pre-label by hand and overwrite freely. The LLM is a first pass that speeds my review — **not** an annotator of record.

**Tool & tracking.** I'll use an instruction-tuned frontier LLM (e.g., GPT-4o or Claude) through its chat/API interface, prompted with the same annotation guide a human annotator would use. Every row carries provenance columns — `llm_label`, `final_label`, and `human_changed` (boolean) — so I can (a) **disclose exactly which examples were AI-pre-labeled** in the AI-usage section, and (b) report my **override rate** (how often I disagreed with the LLM), which doubles as a cheap signal of where the labels are hardest.

**Hard exclusions — what the LLM never touches:**
- The **double-annotated agreement subset (§5)** and the **time-separated test set (§6)** are labeled by hand, from scratch, with no LLM suggestion visible. If the LLM seeded the labels I grade against, the κ ceiling and the test metrics would be measuring *agreement with the LLM*, not a clean human standard — and a model trained on LLM-anchored labels would then post inflated, circular numbers.
- I review the remaining pre-labels in a way that mitigates **automation bias** (the pull to rubber-stamp a plausible-looking suggestion): I read the comment and form my own judgment before letting the suggested label settle anything close to the boundary.

### Failure analysis (after evaluation)

**The move.** Once I have a trained model, I export its **wrong predictions** — the off-diagonal of the confusion matrix, each row carrying the comment text, true label, and predicted label — and ask an AI tool to **cluster them and propose recurring error patterns** before I write the evaluation narrative.

**What I'm looking for.** Concretely: (a) is the model failing exactly where §3 predicted — decorative stats pulling `hot_take` into `analysis`? (b) are there *confounds* I never designed for — does it treat raw length, the mere presence of a `%` token, or all-caps as a class signal? (c) is one direction of the `analysis`↔`hot_take` confusion dominating the other? Patterns like these are what turn a metrics table into an actual diagnosis instead of a score.

**Verification — patterns are hypotheses, not findings.** An LLM asked to find patterns will happily invent one, so every proposed cluster gets checked against the real examples: I **read the specific comments it cites, confirm the pattern is genuinely present, and quantify how many of the errors it actually covers** — a "pattern" that explains 2 of 40 misclassifications is noise, not a finding. Only patterns I've personally re-verified on the underlying comments make it into the write-up. The AI narrows *where I look*; it never gets the final word on what's true.
