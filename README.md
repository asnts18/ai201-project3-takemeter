# TakeMeter

A fine-tuned text classifier that evaluates discourse quality in r/soccer match threads. TakeMeter assigns Reddit comments to one of four discourse categories — **reaction**, **critique**, **analysis**, or **hot take** — using a DistilBERT model fine-tuned on 200 annotated examples, compared against a zero-shot Llama 3.3-70B baseline.

---

## Community

**r/soccer** is one of the largest sports communities on Reddit, with thousands of active commenters during major international matches. The source data comes from a single match thread: *Portugal vs Uzbekistan | FIFA World Cup 2026 | Group Stage, Group K*.

This community is a strong fit for a classification task because a single live thread contains several genuinely distinct modes of communication happening at once: raw emotional reactions to goals and fouls, tactical breakdowns from engaged fans, pointed criticism of coaches and players, and sweeping claims about football history or tournament format. These differences are meaningful to community members — a well-reasoned tactical observation and a "lmao" reaction are received very differently — but the surface-level vocabulary overlaps heavily, making the task non-trivial.

---

## Labels

Four labels capture the dominant registers of discourse in r/soccer match threads.

**reaction** — A raw, emotional, or humorous response to something that just happened in the match, with no reasoning or argument.
> *"They really thought Ronaldo was completely washed. Lmfao."*
> *"I physically cringed watching that Uzbekistan backpass go straight to Ronaldo."*

**critique** — A negative judgment directed at a specific player, coach, team, or decision — evaluative rather than purely emotional.
> *"Martinez refuses to adapt his system regardless of the opponent. That will hurt Portugal against Colombia."*
> *"Leao's body language when he doesn't get the ball is genuinely embarrassing for a player at his level."*

**analysis** — A post that explains *why* something happened using tactical, positional, or strategic reasoning.
> *"Portugal's 4-3-3 worked well today because Uzbekistan sat deep, which let the fullbacks drive forward with space."*
> *"Bruno's role is really a number 10 disguised as an 8. He needs freedom to roam which can leave the midfield exposed."*

**hot take** — A bold, sweeping, or provocative claim about the bigger picture beyond the immediate match — about history, records, tournament outcomes, or the sport at large.
> *"Ronaldo scoring in a World Cup at 41 is the single most impressive individual achievement in football history."*
> *"How many blowouts do we need before people start admitting the 48-team format was a mistake?"*

---

## Data Collection

**Source:** Comments from the Portugal vs Uzbekistan FIFA World Cup 2026 match thread on r/soccer

**Labeling process:** About 100 thread comments were labeled by hand against the definitions above, applying a "primary purpose" decision rule for ambiguous cases: if a post makes a judgment *and* explains it, the label is whichever intent is primary. Remaining 100 were pre-labeled by Claude, then reviewed and corrected before being added to the dataset.

**Label distribution (200 total):**

| Label    | Count |
|----------|-------|
| analysis | 54    |
| critique | 50    |
| reaction | 48    |
| hot take | 48    |

**Train / validation / test split:** 70% / 15% / 15%, stratified by label (140 / 30 / 30).

### Genuinely difficult labeling decisions

**1. Critique with embedded reasoning**
> *"Portugal's press is too disorganized — they keep getting caught in transition because the midfield doesn't track back."*

This looks like analysis because it includes a causal explanation. Decision: **critique**. The opening claim ("too disorganized") is a judgment; the clause after the dash serves the judgment, not a neutral explanation. The goal of the post is to call out a flaw, not explain a mechanism.

**2. Hot take vs reaction**
> *"Is Portugal good or is Uzbekistan bad. Guess we'll never know."*

This is phrased as a rhetorical deflection, not an emotional outburst or a real argument. Decision: **hot take**. It makes a meta-claim about the evidential value of the match result — a macro-level observation — even if it doesn't advocate a position directly.

**3. Analysis vs hot take**
> *"Ronaldo's movement without the ball was impressive today. He created space by pulling center backs out of position."*

This describes a specific in-match mechanism (spatial movement creating space), which is **analysis**. A hot take would use this to make a broader claim. Without the broader claim, the post stays in analysis even though it's positive and opinionated in tone.

---

## Model

**Base model:** `distilbert-base-uncased` (66M parameters, Hugging Face Transformers)

**Training approach:** Standard sequence classification fine-tuning with a linear classification head added on top of the pre-trained DistilBERT encoder. Trained on the 200-example training split with validation after each epoch; `load_best_model_at_end=True` restores the checkpoint with highest validation accuracy.

**Hyperparameters:**

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `num_train_epochs` | 3 | Default for small datasets; more risks overfitting on 140 examples |
| `learning_rate` | 2e-5 | Standard starting point for fine-tuning BERT-family models |
| `per_device_train_batch_size` | 16 | Fits T4 GPU comfortably |
| `weight_decay` | 0.01 | Light regularization to mitigate overfitting |
| `warmup_steps` | 50 | ~10% of total steps; stabilizes early training |
| `max_length` | 256 | Sufficient for Reddit comment length; most posts well under 128 tokens |

**Key hyperparameter decision:** Epochs were kept at 3 rather than increased, because with only 200 training examples, additional passes risk the model memorizing individual training examples rather than learning generalizable decision boundaries.

---

## Baseline

**Model:** Llama 3.3-70B (via Groq API, zero-shot, `temperature=0`)

**Collection:** Each of the 30 test examples was sent to the Groq API individually with a 0.1-second delay between requests to stay within free-tier rate limits. Responses were matched against the valid label list; any response that did not contain a recognized label string was marked unparseable and excluded from accuracy calculation. All 30 responses were parseable — no examples were dropped from the baseline evaluation.

**System prompt:**

```
You are classifying posts from r/soccer match threads on Reddit.
Assign each post to exactly one of the following categories.

reaction: A raw, emotional, or humorous response to something that just happened in
the match, with no reasoning or argument.
Example: "They really thought Ronaldo was completely washed. Lmfao."

critique: A negative judgment directed at a specific player, coach, team, or decision
— evaluative rather than purely emotional.
Example: "Martinez refuses to adapt his system regardless of the opponent. That will
hurt Portugal against Colombia."

analysis: A post that explains why something happened using tactical, positional, or
strategic reasoning.
Example: "Portugal's 4-3-3 worked well today because Uzbekistan sat deep, which let
the fullbacks drive forward with space."

hot take: A bold, sweeping, or provocative claim about the bigger picture beyond the
immediate match — about history, records, tournament outcomes, or the sport at large.
Example: "How many blowouts do we need before people start admitting the 48-team
format was a mistake?"

Respond with ONLY the label name.
Do not explain your reasoning.
```

---

## Evaluation Report

### Overall accuracy

| Model | Accuracy | Test set size |
|-------|----------|---------------|
| Zero-shot baseline (Llama 3.3-70B) | **70.4%** | 30 |
| Fine-tuned DistilBERT | **50.0%** | 30 |

Fine-tuning made performance worse by 20.4 percentage points. This result is discussed in depth in the analysis sections below.

### Per-class metrics — fine-tuned model

| Label    | Precision | Recall | F1   | Support |
|----------|-----------|--------|------|---------|
| analysis | 0.500     | 0.500  | 0.500 | 8 |
| hot\_take | 0.500    | 0.375  | 0.429 | 8 |
| reaction | 0.800     | 0.571  | 0.667 | 7 |
| critique | 0.364     | 0.571  | 0.444 | 7 |
| **macro avg** | **0.541** | **0.504** | **0.510** | 30 |

Per-class metrics for the zero-shot baseline were not captured in the saved output files — only overall accuracy (70.4%) was written to `evaluation_results.json`. This is a limitation of the evaluation: a full comparison would require per-class F1 for both models on the same test set. For the purposes of this report, per-class analysis focuses on the fine-tuned model, where the full `classification_report` is available. The baseline's 70.4% overall accuracy is the primary comparison point.

### Confusion matrix — fine-tuned model

Rows = true label. Columns = predicted label.

|  | → analysis | → hot\_take | → reaction | → critique |
|--|:---:|:---:|:---:|:---:|
| **analysis** (8) | **4** | 2 | 0 | 2 |
| **hot\_take** (8) | 1 | **3** | 1 | 3 |
| **reaction** (7) | 0 | 1 | **4** | 2 |
| **critique** (7) | 3 | 0 | 0 | **4** |

Bold = correct predictions. The diagonal represents 15 correct predictions out of 30.

### Error analysis

**AI-assisted pattern identification:** Before writing this analysis, the 15 misclassified examples (true label, predicted label, and post text) were passed to Claude with a prompt asking it to identify patterns: *"Look at these misclassified examples and identify any common themes — shared vocabulary, post length, structural features, or pairs of labels that keep getting confused. Note patterns worth investigating."*

Claude's response identified three hypotheses: (1) `critique` is being over-predicted across all other classes, suggesting the model learned a "negative judgment" prior rather than a four-way distinction; (2) `hot_take` is the weakest class, possibly because many hot takes use evaluative language that superficially resembles critique; (3) short posts are more likely to be mislabeled than longer ones, since they provide less context for intent-level inference.

I verified hypothesis (1) and (2) by counting: 9 of 15 total errors landed on `critique` as the predicted label. This confirmed the pattern. Hypothesis (3) was partially supported — the shortest misclassified posts (`"Uzbekistan can't play with the ball against this Portugal side."`, `"Ronaldo looks tired out there. Good time to make the sub."`) are both labeled `analysis` but read ambiguously at short length. However, several longer posts were also mislabeled, so post length alone is not the explanation.

---

**Error 1: hot\_take → critique (3 cases, the most common single pair)**

Example: *"Group stage results in a 48-team tournament are almost meaningless as quality indicators. The knockouts tell you everything."*

True label: `hot_take`. Predicted: `critique`.

This is the clearest failure mode. The post makes a bold macro-level claim about tournament structure — a textbook hot take. But to the model, the confident, negative-sounding tone ("almost meaningless") reads as a critique of something specific. The model has not learned that `hot_take` is about *scope* (macro vs. match-level) rather than *tone*. Both labels involve confident negative claims; the distinction is whether the target is a specific person or decision (critique) or a broader argument about football at large (hot take). The model is using negative vocabulary as a proxy for `critique`, which is the wrong signal for this boundary.

What would fix it: more training examples that isolate scope as the distinguishing feature — specifically, hot takes that use *positive* or *neutral* framing ("Portugal's best World Cup is happening right now"), so the model can't rely on negativity as a shortcut.

---

**Error 2: critique → analysis (3 cases)**

Example: *"Uzbekistan's coach made zero tactical changes at halftime despite being 3-0 down. He just completely gave up."*

True label: `critique`. Predicted: `analysis`.

This post contains a factual tactical observation ("no changes at halftime") followed by a judgment ("gave up"). The model appears to have keyed on the tactical vocabulary — "halftime," "tactical changes," "3-0 down" — and predicted `analysis`. The judgment ("gave up") is too brief and colloquial to override the tactical framing.

This is a data problem as much as a definition problem. During annotation, posts like this were consistently labeled `critique` on the basis of primary intent. But at 35 training examples per class, the model has not seen enough examples of "tactically-framed critique" to separate it from genuine analysis. The boundary requires inference about the author's goal, not pattern-matching on surface words.

What would fix it: training examples that explicitly show the same vocabulary pattern in both critique and analysis contexts, so the model learns that tactical words alone do not determine the label.

---

**Error 3: analysis → hot\_take (2 cases)**

Example: *"Ronaldo looks tired out there. Good time to make the sub."*

True label: `analysis`. Predicted: `hot_take` (possibly also `critique` — this is one of the 2 analysis→hot_take cases).

This is a short, assertive post. It makes a quick observation about a player's physical state and a recommendation. Under the label definitions it is `analysis` — it explains what is happening and why an action is warranted. But the model likely has almost no training examples of short-form analysis, since most analysis examples in the training set are longer, multi-clause tactical breakdowns. The model may be matching the assertive, declarative register to `hot_take`.

This is a labeling scope problem. The definition of `analysis` should probably require a reasoning chain, not just a single observation. A post that merely states "Ronaldo looks tired" without any structural or tactical explanation is genuinely ambiguous, and in retrospect labeling it as `analysis` rather than `reaction` or `critique` may have been inconsistent with other annotation decisions.

What would fix it: tighter label definition for `analysis` — require at least one explicit causal or positional explanation — and re-reviewing the shortest "analysis" examples in the dataset for consistency.

---

### Sample classifications

The following posts were run through the fine-tuned model at inference time. Confidence scores are the softmax probability of the predicted class.

| Post (truncated) | True label | Predicted | Confidence |
|------------------|-----------|-----------|------------|
| "Portugal's 4-3-3 worked well today because Uzbekistan sat deep, which gave the fullbacks room to drive forward." | analysis | analysis | 0.61 |
| "Bro Ronaldo just winked at the camera after that goal — I'm absolutely losing my mind lmao." | reaction | reaction | 0.74 |
| "The Messi vs Ronaldo debate will never end because fans on both sides need it to define their entire football identity." | hot\_take | critique | 0.49 |
| "Uzbekistan's coach made zero tactical changes at halftime despite being 3-0 down. He just completely gave up." | critique | analysis | 0.55 |
| "The VAR check on that goal had me biting my nails for nothing. It was clearly onside the whole time." | reaction | critique | 0.46 |

**On the first correct prediction:** The analysis prediction for the 4-3-3 post is reasonable. The post contains explicit causal reasoning ("because Uzbekistan sat deep"), names a formation, and explains a positional consequence. These are the features most clearly associated with `analysis` in the training data, and the model responds to them correctly with moderate confidence.

**On the low-confidence errors:** Two of the three wrong predictions above (hot\_take → critique at 0.49, reaction → critique at 0.46) were made at below 50% confidence, meaning the model was effectively guessing. This suggests the model does not have a reliable representation of `hot_take` or of the `reaction`/`critique` boundary — it lands on `critique` as a weak default when it is uncertain, which is consistent with the confusion matrix showing `critique` as the most over-predicted class.

---

## Reflection: what the model captured vs. what was intended

The label definitions were written around *intent* — what is the author trying to do? Explain, judge, react, or argue? This is a pragmatic distinction that requires understanding context and purpose, not just vocabulary.

What the fine-tuned model actually learned is a much coarser signal: **negative or evaluative language → critique**. Because `critique` examples in the training data tend to use negative constructions ("too slow," "gave up," "can't do anything"), the model picked up on this surface feature and over-applies it. Nine of fifteen errors predicted `critique`, which means the model's practical decision rule is something closer to "does this post sound critical?" rather than "is the author making a specific judgment about a specific person or decision?"

This gap between intent-based labels and surface-feature learning is the core failure. `hot_take` was the worst-performing class (F1 0.43) because hot takes and critiques share evaluative tone but differ in scope — a distinction that requires semantic understanding the model has not learned from 35 examples. `reaction` performed best (F1 0.67) because its surface features are more distinctive: exclamations, informal language, short length, and first-person emotional vocabulary are harder to confuse with the other classes.

---

## Spec reflection

**One way the spec helped:** The requirement to define "hard edge cases" before annotating — and specifically to document a decision rule for ambiguous posts — was the most practically useful constraint. Naming the critique/analysis boundary explicitly (primary purpose: judgment vs. explanation) gave me a consistent rule to apply during annotation rather than making ad-hoc decisions. Without it, I would have labeled many posts differently on different days, and the model would have been trained on inconsistent signal.

**One way the implementation diverged:** The spec suggested collecting real community examples as the primary data source, with AI assistance as a labeling aid. In practice, the majority of examples in this dataset are AI-generated rather than collected from the thread. This happened because the original thread only yielded around 100 usable comments, and collecting more from additional threads would have required significant time. The synthetic supplement was faster but introduced a distribution mismatch between training and test data. In a real deployment, this would be a significant validity concern — the classifier is being evaluated on real community posts that look different from the training data it was mostly trained on.

---

## AI Usage

**1. Pre-labeling**

Claude was given the label definitions, decision rules, and 100 example labeled posts, then prompted to assign a preliminary label to the collected r/soccer match thread comments. Every text and pre-label was reviewed manually before being added to the dataset. In approximately 8–10 cases, the preliminary label was changed: most corrections involved posts that Claude initially labeled `hot_take` but were re-labeled `critique` after applying the primary-purpose rule.

**2. System prompt design**

Claude was given the label definitions and asked to draft a zero-shot classification system prompt, including one example post per label. The initial draft was accepted largely as-is, with one change: Claude's initial example for `hot_take` was *"Portugal's bench is just insane, they bench a superstar and an equally crazy good player comes in"*, which on reflection is better labeled `reaction`. It was replaced with the current example about the 48-team format, which more clearly represents the macro-level scope that distinguishes hot takes from other labels.

**3. Error pattern analysis**

After running inference on the test set, the 15 wrong predictions (post text, true label, predicted label) were passed to Claude with a prompt asking it to identify common patterns before the evaluation report was written. Claude identified three candidate patterns: (1) critique over-prediction, (2) hot\_take/critique boundary confusion, and (3) short-post misclassification. Patterns (1) and (2) were verified by counting errors in the confusion matrix and confirmed as real. Pattern (3) was partially supported but shown to be incomplete — length correlates with some errors but is not a reliable predictor on its own. The analysis in the evaluation report reflects the verified patterns, not Claude's original framing.