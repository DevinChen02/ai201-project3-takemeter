# TakeMeter — Classifying Discourse Quality on r/nba

**AI201 · Project 3 — Text Classification**

TakeMeter is a three-way text classifier that separates basketball discourse by its *dominant
function*: a substantiated **argument** (`analysis`), a confident-but-unsupported **assertion**
(`hot_take`), or a raw emotional **reaction** (`reaction`). The goal is to model *discourse form*,
not topic — the same subject ("is SGA the MVP?") shows up as all three in the same thread, so the
classifier has to learn how a claim is made, not what it is about.

This README is the final report. The working notes, full label rationale, edge-case rulebook, and
the success criteria fixed in advance live in [planning.md](planning.md).

---

## 1. Community and Task

**Community:** [r/nba](https://www.reddit.com/r/nba/) — chosen because the `analysis` / `hot_take` /
`reaction` distinction is the community's *native* vocabulary. Users already police a quality axis
("great breakdown" vs. "bad take"), so the labels reflect a real, contested signal rather than one
imposed from outside. A single event reliably produces all three classes side by side, which makes
the boundaries observable.

**Task:** single-label classification into three classes. The actionable downstream use case is a
reader-facing **"show me the analysis" quality filter**, which makes `analysis` the class that
matters most and **precision-on-`analysis`** the metric a deployment would be judged on.

---

## 2. Label Definitions

| Label | One-line definition |
|-------|---------------------|
| `analysis` | A structured argument supported by specific, verifiable evidence (stats, on/off splits, tactical/film observation, historical comparison) — the reasoning would survive a fact-check. |
| `hot_take` | A bold or contrarian opinion asserted confidently but with no genuine support; any "evidence" is vague, cherry-picked, or decorative. |
| `reaction` | An immediate emotional response to a specific event (a play, game, trade, injury) — it expresses a feeling, not a claim. |

**The hard boundary is `analysis` ↔ `hot_take`** — the "decorative stats" problem, where a confident
opinion *with a number attached* masquerades as analysis. The decision rule used for every
borderline case (in both annotation and the baseline prompt):

> **Strip the opinion framing and ask whether a genuine argument remains.** If the evidence is
> *load-bearing* (you could reconstruct the argument from the facts alone) → `analysis`. If it is
> *decorative* (a cherry-picked stat, or "the numbers don't lie" standing in for reasoning) →
> `hot_take`.

Two secondary rules: `hot_take` vs. `reaction` is decided by the comment's *primary payload* (a claim
about the world → `hot_take`; a feeling triggered by an event → `reaction`); `reaction` vs.
`analysis` requires a named mechanism *with* evidence to lean `analysis`, otherwise pattern-language
used as a vent intensifier → `reaction`.

---

## 3. Dataset and Annotation

| | |
|---|---|
| **Total labeled examples** | 203 |
| **Class balance** | `reaction` 71 · `analysis` 66 · `hot_take` 66 |
| **Split (stratified 70/15/15, `random_state=42`)** | train 142 · val 30 · test 31 |
| **Test-set class balance** | `analysis` 10 · `hot_take` 10 · `reaction` 11 |
| **Columns** | `text`, `label`, `notes` |

Labels were assigned by hand against the written rulebook in [planning.md](planning.md). The hardest
judgment calls are documented inline in the `notes` column of
[labeled_data.csv](labeled_data.csv) — each borderline row records the candidate labels, the final
decision, and the rule that settled it. As predicted in planning, nearly every flagged case sits on
the `analysis` ↔ `hot_take` seam (e.g. *"SGA is the MVP and it's not close. 32-6-6 on 53% … the
numbers don't lie"* → `hot_take`, because the stats are decorative once you strip the mic-drop).

> **Note on the data.** The corpus is clean and prototypical — comments sit clearly within their
> category with little of the noise, sarcasm, or fragmentation of raw scraped Reddit. This matters
> for interpreting the results below: a cleanly separable dataset is exactly the regime where a
> strong zero-shot model excels and a small fine-tuned model has little to add.

---

## 4. Models

### Fine-tuned model — DistilBERT
`distilbert-base-uncased` with a 3-class classification head, fine-tuned on the 142 training
examples.

| Hyperparameter | Value | Notes |
|---|---|---|
| Epochs | 3 | Starter default for small datasets |
| Learning rate | 2e-5 | Standard BERT-family starting point |
| Train batch size | 16 | Fits a T4 GPU |
| Weight decay | 0.01 | |
| Warmup steps | 50 | |
| Max sequence length | 256 tokens | |
| Best-model selection | by validation accuracy | `load_best_model_at_end=True` |

Hyperparameters were left at the starter defaults; no changes were made.

### Baseline — zero-shot LLM (Groq)
`llama-3.3-70b-versatile`, prompted at `temperature=0` with a system prompt that names the task,
defines all three labels in plain language, gives one example per label, and embeds the
*strip-the-framing* decision rule for the hard `analysis` ↔ `hot_take` case. The model is asked to
output only the label string. All 31 test responses parsed cleanly.

---

## 5. Evaluation Report

All numbers are computed on the **same locked 31-example test set** for both models. Metrics are
reported from [evaluation_results.json](evaluation_results.json) and the notebook outputs.

### 5.1 Headline results

| Model | Accuracy | Macro-F1 |
|---|---|---|
| Zero-shot baseline (Groq `llama-3.3-70b`) | **1.000** | **1.00** |
| Fine-tuned DistilBERT | 0.903 | 0.90 |
| **Difference (fine-tuned − baseline)** | **−0.097** | **−0.10** |

**The headline finding is a regression: fine-tuning the small model made things *worse* than the
zero-shot baseline.** This is a real and interpretable result, not a bug — see §6 and §8.

> **Read the baseline's 100% with caution.** It is a point estimate on 31 examples with no
> confidence interval. On a cleanly separable dataset, with a 70B instruction-tuned model and a
> prompt that hands it the exact decision rule, a perfect score is plausible — but it says more about
> the data being separable than about the model being infallible in the wild.

### 5.2 Per-class metrics — fine-tuned DistilBERT

| Class | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `analysis` | 0.91 | 1.00 | 0.95 | 10 |
| `hot_take` | 0.89 | 0.80 | 0.84 | 10 |
| `reaction` | 0.91 | 0.91 | 0.91 | 11 |
| **macro avg** | **0.90** | **0.90** | **0.90** | 31 |
| weighted avg | 0.90 | 0.90 | 0.90 | 31 |

`hot_take` is the weak class: lowest recall (0.80) and the only class that leaks into *both* others.
`analysis` has perfect recall — the model never misses a real analysis post.

### 5.3 Per-class metrics — baseline (Groq)

| Class | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `analysis` | 1.00 | 1.00 | 1.00 | 10 |
| `hot_take` | 1.00 | 1.00 | 1.00 | 10 |
| `reaction` | 1.00 | 1.00 | 1.00 | 11 |
| **macro avg** | **1.00** | **1.00** | **1.00** | 31 |

### 5.4 Confusion matrix — fine-tuned model

Rows are the true label, columns are the predicted label. (Also saved as
[confusion_matrix.png](confusion_matrix.png).)

| True ↓ / Pred → | `analysis` | `hot_take` | `reaction` |
|---|---|---|---|
| **`analysis`** | **10** | 0 | 0 |
| **`hot_take`** | 1 | **8** | 1 |
| **`reaction`** | 0 | 1 | **10** |

**Reading it.** All three errors involve `hot_take`. Two true `hot_take`s were lost — one to
`analysis`, one to `reaction` — and one true `reaction` was misread as `hot_take`. Crucially, the
errors do **not** concentrate on the `analysis` ↔ `hot_take` boundary the way planning §6 predicted:
only **1 of 3** off-diagonal cells (≈33%) sits there, short of the ≥60% the plan set as a Tier-1
condition. Instead, `hot_take` is confused in *every* direction — consistent with it being defined
*negatively* (a claim without real evidence), so it has no positive lexical signature of its own and
ends up the residual class.

---

## 6. Error Analysis

### 6.1 AI-assisted pattern surfacing (and what I discarded)

I exported the three misclassified rows (text, true label, predicted label, confidence) and asked an
LLM to cluster them and propose recurring error patterns *before* writing this section. It returned
six candidate patterns. I then re-read the three comments myself to verify each. With only three
errors, "patterns" are fragile, so I kept only what the actual examples support:

| Candidate pattern (AI-proposed) | Verdict after re-reading | 
|---|---|
| All errors involve `hot_take` (as true or predicted) | **Kept** — true for 3/3. |
| Form/function mismatch: surface vocabulary points to the wrong class | **Kept** — clear in errors #1 and #2. |
| No concrete numeric anchor (per-game %, on/off, splits) present | **Kept** — true for 3/3; the model leans hard on stat tokens as the `analysis` cue. |
| Uniformly low confidence (~0.35) on every error | **Kept** — and it extends to *correct* predictions too (see §8). |
| Misclassifications correlate with post length | **Discarded** — the three posts vary in length; n=3 cannot support this. |
| Sarcasm is driving the errors | **Discarded** — none of the three are sarcastic. |

The one correction I made to the AI's framing: it initially reported the errors as "concentrated on
the `analysis` ↔ `hot_take` boundary." The confusion matrix shows only one of three errors there, so
I downgraded that to "`hot_take` is confused in all directions," which is what the data actually
shows.

### 6.2 Three misclassified examples, analyzed

**Error #1 — `reaction` → `hot_take` (confidence 0.35)**
> *"I FORGOT THIS TEAM COULD BE FUN. That was the most alive I've felt watching them in years, what a
> game."*

This is a textbook `reaction` — a feeling about a game ("most alive I've felt," "what a game"). The
model likely keyed on the all-caps declarative opening, which *structurally* resembles a bold
`hot_take` assertion, and on the fact that the post names **no concrete event** (no play, no score,
no clock) — unlike the prototypical training `reaction`s ("down one, 0.4 on the clock"). A
low-information, event-free reaction lands outside the pattern the model learned for the class.

**Error #2 — `hot_take` → `analysis` (confidence 0.35)**
> *"The Mavericks are a one-man show that'll collapse the second Luka has an off night. No structure,
> no defense, all hero ball."*

This is the canonical hard boundary, in the *vocabulary* direction. The post uses analysis-shaped
language ("no structure, no defense, all hero ball") but cites zero evidence — it is a confident
sweeping claim, i.e. a `hot_take`. The model mistook the **form** of tactical analysis for its
**function**. This is the exact mirror of the "decorative stats" trap from planning §3: here it is
*decorative tactical vocabulary*. The model learned "tactical words → `analysis`" but did not learn
that the evidence has to be load-bearing.

**Error #3 — `hot_take` → `reaction` (confidence 0.35)**
> *"Bron stat-pads in garbage time to keep the streak alive. The longevity is impressive but half
> those numbers are empty and everyone knows it."*

A dismissive, evidence-free claim ("half those numbers are empty," sealed with the decorative
"everyone knows it") — a `hot_take`. It mentions stats only to wave them away, so the model finds no
`analysis` signal; the casual, dismissive register ("everyone knows it," "empty") then reads as
venting, pulling it toward `reaction`. The model had no positive cue for `hot_take` and defaulted to
the emotional class.

### 6.3 Guiding questions

- **Which labels are being confused?** Every error touches `hot_take`. It is simultaneously
  under-recalled (0.80) and over-predicted onto a true `reaction`. The model handles `analysis` (1.00
  recall) and `reaction` well; `hot_take` is where the boundary is unlearned.
- **Why is that boundary hard?** `hot_take` is defined by *absence* — a claim without real evidence,
  emotion without a concrete event. It has no positive lexical signature, so the model falls back on
  surface cues (analytical vocabulary, all-caps, stat tokens) that point at the *other* classes.
- **Is this a labeling problem or a data/model problem?** A **data/model** problem. All three
  misclassified posts were labeled *confidently* during annotation — none was flagged as borderline
  in the `notes` column. The labels are clean; the model simply hasn't learned the boundary. This
  points to training-data distribution and model capacity, not annotation inconsistency.
- **What would need to change to fix it?** More `hot_take` examples that *look like* the other classes
  (tactical-sounding-but-unsupported takes; dismissive stat-waving) so the model learns function over
  form; and enough training signal (more data and/or epochs, plus calibration) to push predictions
  off the ~0.35 fence — see §8.

---

## 7. Sample Classifications (live demo)

Five-ish fresh posts run through the **fine-tuned** model, each shown with its predicted label and
softmax confidence. These inputs have no gold label; the "✓ reasonable" notes assess whether the
prediction matches the obviously-correct class.

| # | Post | Predicted | Confidence |
|---|------|-----------|-----------:|
| 1 | "This team is an absolute joke, trade everyone and rebuild right now!" | `hot_take` | 35.60% |
| 2 | "If you look at the synergy between the point guard and the center, their pick-and-roll efficiency is top 5 in the league." | `analysis` | 38.29% |
| 3 | "I literally threw my remote at the TV when he missed that free throw. Unbelievable!" | `reaction` | 35.54% |
| 4 | "He's the greatest player of all time and it's not even close, anyone who disagrees doesn't know basketball." | `hot_take` | 36.71% |

**Why the correct ones are reasonable:**
- **#3 (`reaction`, ✓):** This is a clear, reasonable prediction — it is a raw emotional response
  ("I literally threw my remote") to a specific event (a missed free throw), with no claim or
  argument, which is exactly the `reaction` definition.
- **#4 (`hot_take`, ✓):** Reasonable — a sweeping, evidence-free superlative ("greatest of all time,
  it's not even close") sealed with a decorative dismissal ("anyone who disagrees doesn't know
  basketball"), the textbook `hot_take` shape.
- **#2 (`analysis`, ✓):** Reasonable and, tellingly, the model's *most confident* prediction (38.29%)
  — it names a specific mechanism (PG–center pick-and-roll synergy) backed by a checkable claim
  ("top 5 in the league"), matching the `analysis` definition's "tactical observation + verifiable
  evidence."

The thing to notice across the whole table: **every confidence is between 35% and 39%**, barely
above the 33% chance line for a 3-class problem. The model is *correct but not confident* — the
subject of the reflection below.

---

## 8. Reflection: What the Model Captured vs. What I Intended

I intended the model to learn a **functional** distinction — *is this comment arguing, asserting, or
feeling?* What it actually learned is a **lexical surface** approximation of that distinction, and
the two come apart in exactly the places the design cared about most.

**What it captured well.** The `analysis` class — perfectly recalled. But the live demo and the error
confidences reveal *how*: the model latched onto the strongest surface correlate of `analysis` in my
clean training set — the presence of concrete statistical and tactical tokens (percentages, on/off
language, "efficiency," "scheme"). When those tokens are present, it is right; the decision boundary
is essentially "does this read like stats/tactics?"

**What it missed.** The whole point of the project — the `hot_take` boundary, where *form and function
diverge*. It cannot tell decorative tactical vocabulary (Error #2) from a real argument, and it
cannot recognize a `hot_take` that lacks any positive signature (Error #3). It overfit to *form*
(analytical-sounding words → `analysis`, all-caps → `hot_take`) and missed *function* (does the
evidence do argumentative work?). That is the exact gap my label definitions were written to close.

**The calibration story is the deepest gap.** Every prediction — right or wrong, on the test set and
the live demo — sits at ~0.35–0.38 confidence. For three classes, that is barely off the uniform
prior. The model did not so much *learn* the boundaries as *lean* toward them: 3 epochs on 142 clean
examples at lr 2e-5 nudged the logits just far enough to win a 90% accuracy coin-flip, while leaving
the softmax nearly flat. This has a concrete consequence for the planned deployment: the
"show-me-the-analysis" filter in planning §6 relies on a **confidence threshold** to route
low-confidence posts to "unclassified." With every probability pinned near 0.35, *any* usable
threshold would abstain on essentially everything. The point-estimate `analysis` precision (0.91)
clears the planned 0.80 bar, but the mechanism that was supposed to make that precision trustworthy
is inoperable until the model is calibrated.

**Why the baseline won.** Put together, the regression makes sense. The dataset is cleanly separable;
a 70B zero-shot model handed the decision rule reads *function* directly and nails all 31; the small
fine-tuned model, underfit and uncalibrated, can only approximate *form* and loses the cases where
form lies. For this dataset, the better classifier is the well-prompted large model — fine-tuning a
small one would need more (and more adversarial) `hot_take` data, more epochs, and an explicit
calibration step before it could compete.

### Results vs. the success criteria fixed in [planning.md](planning.md) §6

| Tier | Bar (set in advance) | Outcome |
|---|---|---|
| **Tier 1** — proof of concept | Beat baselines; ≥60% of errors on `analysis`↔`hot_take` | **Not met.** Lost to the zero-shot baseline (0.90 vs 1.00); only ~33% of errors on the predicted boundary. |
| **Tier 2** — genuinely useful | Macro-F1 within 7 pts of a measured human ceiling; no class recall < 0.50 | **Partial / not adjudicable.** No κ human-ceiling study was run (see §9), so "within 7 pts" can't be checked; but no class collapsed (min recall 0.80). |
| **Tier 3** — deployable | `analysis` precision ≥ 0.80 (CI lower bound ≥ 0.70) with recall ≥ 0.60, via a fixed confidence threshold | **Point estimate passes (P=0.91, R=1.00); robustness fails.** No bootstrap CI (n=10 support), and the confidence threshold is unusable given ~0.35 calibration. |

---

## 9. Spec Reflection

**One way the spec guided the implementation.** Planning §5 insisted that **accuracy alone is
misleading** and required macro-F1 + per-class metrics + a confusion matrix. That decision is what
made the project legible: the 90% accuracy looks healthy, but the per-class table and confusion
matrix are what revealed that `hot_take` is the single weak class and that errors do *not* fall where
predicted. The spec's *strip-the-framing* decision rule (§3) also flowed straight into the baseline
prompt — embedding that rule verbatim is very likely part of why the zero-shot model scored so
cleanly on the borderline `analysis`/`hot_take` posts.

**One way the implementation diverged from the spec, and why.** Planning §5–§6 specified a far richer
evaluation harness than was actually run: a **majority-class baseline** and a **TF-IDF + logistic
regression baseline**, a **Cohen's κ inter-annotator-agreement study** to fix a human-performance
ceiling, a **time-separated test set**, and **bootstrap 95% confidence intervals** on the key gaps
and on `analysis` precision. The executed notebook implements a single random stratified 70/15/15
split and compares against **one** baseline — a strong zero-shot LLM — using point estimates only.
The divergence is largely a function of the provided starter pipeline, which is built around the
DistilBERT-vs-Groq comparison on one split; the κ study (needs a second annotator), the extra
baselines, and the CIs would all be additional code outside that scaffold. The cost of the divergence
is real: without a human ceiling I can't situate 0.90 macro-F1 against what a consistent human would
score, and without CIs I can't tell whether the baseline's 100% is statistically distinguishable from
the fine-tuned model on a test set this small — exactly the caution planning §6 was written to
enforce.

---

## 10. AI Usage

Per the AI tool plan in planning §7, the rule throughout was **AI proposes, I dispose** — nothing an
AI produced entered the gold labels without human verification. Gold labels in
[labeled_data.csv](labeled_data.csv) were assigned by hand (the `notes` column records the human
judgment calls); the file carries no AI-generated labels. AI was used at two specific points:

**Instance 1 — Label-definition stress-testing (before annotation).**
- *What I directed it to do:* I gave an LLM the three label definitions and the "decorative stats"
  edge case and asked it to generate synthetic comments that deliberately sit on the
  `analysis` ↔ `hot_take` boundary.
- *What it produced:* a handful of adversarial borderline posts (confident opinions with a stat
  bolted on; tactical-sounding claims with no real evidence).
- *What I changed / overrode:* I applied my own strip-the-framing rule to each. Where posts were
  genuinely ambiguous, I tightened the annotation guide — adding the "evidence must be load-bearing"
  tie-breaker and the primary-payload rule for `hot_take` vs. `reaction`. I **discarded the synthetic
  posts entirely**: none were added to the dataset, to keep the corpus human-authored. Their only
  output was edits to the rulebook.

**Instance 2 — Failure-analysis clustering (after evaluation).**
- *What I directed it to do:* I exported the three misclassified test rows (text, true, predicted,
  confidence) and asked an LLM to cluster them and propose recurring error patterns.
- *What it produced:* six candidate patterns (all-errors-touch-`hot_take`; form/function mismatch;
  missing numeric anchor; uniformly low confidence; plus post-length and sarcasm).
- *What I changed / overrode:* I re-read the three comments and verified each pattern myself. I
  **discarded "post length" and "sarcasm"** as unsupported at n=3, and I **corrected** the AI's claim
  that errors "concentrate on the `analysis` ↔ `hot_take` boundary" — the confusion matrix shows only
  1 of 3 errors there. Only the four human-verified patterns made it into §6.

*(No LLM pre-labeling was used as the annotator of record. Had a triage pre-labeling pass been used,
it would be disclosed here with an override rate; the committed dataset reflects hand-assigned labels
with manually written edge-case notes.)*

---

## 11. Limitations and Future Work

- **Tiny test set (31).** All metrics are point estimates with no confidence intervals; a single
  comment moves accuracy by ~3 points. The baseline's 100% and the fine-tuned 90% may not be
  statistically distinguishable.
- **Clean, prototypical data.** The corpus lacks the noise, sarcasm, and fragmentation of real
  scraped Reddit, which inflates the zero-shot baseline and understates how hard the task is in the
  wild.
- **Severe under-confidence / poor calibration.** ~0.35 confidence on every prediction makes the
  planned confidence-threshold deployment unusable; calibration (temperature scaling) and more
  training would be the first fix.
- **`hot_take` is under-learned.** It needs more — and more adversarial — examples (form-vs-function
  cases) to give the model a positive signal for the class.
- **Evaluation harness gaps.** No human-ceiling (κ) study, no majority/TF-IDF baselines, no bootstrap
  CIs, no time-separated test split — all specified in planning but not executed (§9).

---

## 12. Repository Contents

| File | What it is |
|---|---|
| [Copy_of_ai201_project3_takemeter_starter_clean.ipynb](Copy_of_ai201_project3_takemeter_starter_clean.ipynb) | End-to-end pipeline: load → split → tokenize → fine-tune DistilBERT → evaluate → Groq baseline → compare → live demo |
| [labeled_data.csv](labeled_data.csv) | 203 hand-labeled comments (`text`, `label`, `notes`) |
| [evaluation_results.json](evaluation_results.json) | Saved metrics (baseline acc, fine-tuned acc, improvement, test size, label map, model) |
| [confusion_matrix.png](confusion_matrix.png) | Fine-tuned model confusion matrix (also written out as a table in §5.4) |
| [planning.md](planning.md) | Design notes: label rulebook, edge cases, data/eval plan, success tiers, AI tool plan |
| [README.md](README.md) | This report |

### Reproducing the results
The notebook is written for **Google Colab with a T4 GPU** (it uses `google.colab` upload/secrets and
a `GROQ_API_KEY`). To reproduce: open the notebook in Colab, set the runtime to T4 GPU, add your
`GROQ_API_KEY` to Colab Secrets, upload [labeled_data.csv](labeled_data.csv), and run all cells. The
split is seeded (`random_state=42`), so the DistilBERT metrics are deterministic; the Groq baseline
runs at `temperature=0` for stability.
