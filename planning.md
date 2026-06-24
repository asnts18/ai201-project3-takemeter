# TakeMeter — Planning Document

## 1. Community

**Chosen community:** r/soccer (Reddit), specifically match discussion threads for high-profile games. The source thread is the *Portugal vs Uzbekistan | FIFA World Cup 2026 | Group Stage* match thread.

**Why r/soccer?** It is one of the largest sports communities on Reddit, with hundreds of active commenters during major matches. Crucially, the discourse is text-driven and diverse: a single thread contains emotional outbursts, tactical breakdowns, player critiques, and sweeping hot takes — often within the same minute of match time. This variety makes it a strong candidate for classification because there are genuinely distinct registers of communication happening simultaneously. The community also has implicit norms: match threads reward quick reactions, deeper tactical threads reward analysis, and the line between a sharp observation and a bad-faith hot take is frequently contested.

**Why is it a good fit for a classification task?** The discourse is varied enough that labeling is non-trivial (it can't be solved with keywords alone), high-volume enough to collect 200+ examples easily, and grounded in community norms that produce real edge cases rather than artificial ones.

---

## 2. Labels

Four labels were defined to capture the dominant registers of discourse in r/soccer match threads.

### reaction
A raw, emotional, or humorous response to something that just happened in the match. The post conveys feeling — surprise, delight, frustration, amusement — without offering any reasoning, judgment, or argument.

**Examples:**
- *"They really thought Ronaldo was completely washed. Lmfao."*
- *"I physically cringed watching that Uzbekistan backpass go straight to Ronaldo."*

### critique
A negative judgment directed at a specific player, coach, team, or decision. The post calls something out as wrong, poor, or worthy of change. It is evaluative rather than purely emotional, but the reasoning (if any) serves the judgment rather than explaining the game.

**Examples:**
- *"Martinez refuses to adapt his system regardless of the opponent. That will hurt Portugal against Colombia."*
- *"Leao's body language when he doesn't get the ball is genuinely embarrassing for a player at his level."*

### analysis
A post that explains *why* something happened using tactical, positional, or strategic reasoning. The post goes beyond evaluating and tries to illuminate a mechanism — a formation, a player role, a spatial pattern, a substitution effect.

**Examples:**
- *"Portugal's 4-3-3 worked well today because Uzbekistan sat deep, which let the fullbacks drive forward with space."*
- *"What made Ronaldo's second goal so clever was the late run — he timed his arrival to the delivery perfectly."*

### hot take
A bold, sweeping, or provocative claim about the bigger picture beyond the immediate match. The post invites debate rather than reaction and typically makes an argument about football at large: history, records, tournament outcomes, confederations, or meta-questions about the format.

**Examples:**
- *"Ronaldo scoring in a World Cup at 41 is the single most impressive individual achievement in football history."*
- *"How many blowouts do we need before people start admitting the 48-team format was a mistake?"*

---

## 3. Hard Edge Cases

**The main ambiguous boundary is between critique and analysis.** A post can simultaneously call out a flaw (critique) and explain why it's a flaw with tactical reasoning (analysis). The deciding factor is: *which is the primary purpose?* If the post's goal is to assign blame or call for change, it is a critique even if it includes an explanatory clause. If the goal is to illuminate a mechanism that happens to reflect poorly on someone, it is analysis.

*Example borderline post:* "Portugal's press is too disorganized — they keep getting caught in transition because the midfield doesn't track back."
- This sounds like both. Decision rule: the opening claim ("too disorganized") is a judgment and the explanation is subordinate to it. → **critique**

**The secondary ambiguous boundary is between hot take and reaction.** A hot take is a macro-level argument; a reaction is a micro-level feeling. The question is whether the post is making a claim about the world or expressing a feeling about what just happened.

*Example borderline post:* "Is Portugal good or is Uzbekistan bad. Guess we'll never know."
- This is phrased as a rhetorical question with no real argument. It doesn't advocate a position so much as deflect one. → **hot take** (meta-commentary on the match's evidential value), but closely contested.

**Handling ambiguous cases during annotation:** Apply the primary-purpose rule (what is the post *trying to do*?) and pick the label that best fits the dominant intent. Document every genuinely difficult case in the README's annotation section. If a case is 50/50 after applying the rule, default to the label with fewer examples at that point to keep the distribution balanced.

---

## 4. Data Collection Plan

**Source:** Reddit match thread comments from the Portugal vs Uzbekistan FIFA World Cup 2026 Group Stage thread. Comments were copy-pasted directly from the live thread.

**Volume per label:** Target 50 per label for a balanced 200-example dataset.

**Collection approach:**
- Read through the thread sequentially and label as you go.
- If a label falls below 40 examples after reading the full thread, supplement with adjacent threads (e.g. post-match discussion, daily thread from the same day) specifically targeting under-represented labels.
- Analysis and hot take are historically under-represented in match threads (which skew heavily reactive and critical). If either falls below 40 after the main thread, pull from tactical discussion threads or "Change My View" style posts within r/soccer.

**Final distribution achieved (200 examples):**
| Label | Count |
|-------|-------|
| analysis | 54 |
| critique | 50 |
| reaction | 48 |
| hot take | 48 |

---

## 5. Evaluation Metrics

**Primary metric: macro-averaged F1-score.** The dataset is nearly balanced (48–54 per class), so accuracy is a reasonable signal, but macro F1 is preferred because it weights all four labels equally. A model that excels at *reaction* (the largest and most distinctive class) but fails on *analysis* and *hot take* would show misleadingly high accuracy while being useless in practice.

**Per-class metrics (precision and recall per label):** These are essential for diagnosing failure modes. Expected failure modes differ by class:
- *reaction* vs *critique*: the model may over-predict reaction for any negative-sounding post.
- *analysis* vs *critique*: the model may fail to distinguish tactical reasoning from evaluative judgment.
- *hot take* vs *reaction*: short, punchy posts may be misclassified in either direction.

**Confusion matrix:** Required to identify which label pairs are most often confused. This informs whether the boundary issues are symmetric (both directions) or directional.

**Why accuracy alone is insufficient:** With four balanced classes, a model that always predicts the most common label would achieve 27% accuracy — a trivial baseline. Accuracy also masks class-specific failures. A community moderation tool that labels all analysis as critique (or vice versa) is not useful even if its overall accuracy looks acceptable.

---

## 6. Definition of Success

**Minimum threshold for "good enough":**
- Overall accuracy ≥ 70% on the held-out test set
- Macro F1 ≥ 0.65 across all four labels
- No single label with F1 < 0.50 (no class should be functionally unlearned)

**Aspirational threshold for "genuinely useful":**
- Overall accuracy ≥ 80%
- Macro F1 ≥ 0.75
- All per-class F1 scores ≥ 0.65

**Evaluation specificity check (1b):** These thresholds are objectively measurable from the test set evaluation output. There is no ambiguity in determining whether they were hit. The fine-tuned model will be compared against a zero-shot Llama 3.3-70B baseline on the same test set using the same metrics.

---

## 7. AI Tool Plan

### 7a. Label Stress-Testing
Before annotating the full dataset, the label definitions and edge case rules were given to Claude with a prompt asking it to pre-label the collected data. The pre-labeling was treated as a first-pass suggestion, not ground truth. Every label was verified against the definitions in Section 2 and the decision rules in Section 3.

### 7b. Failure Analysis
After generating test set predictions from both the fine-tuned model and the Llama zero-shot baseline, the full list of wrong predictions (text + true label + predicted label) will be passed to Claude with a prompt asking it to identify patterns: are errors concentrated in a particular label? Do they tend to occur with short posts, posts with profanity, posts containing player names, or posts that mix multiple registers? Claude's pattern hypotheses will then be verified manually by checking whether the flagged pattern holds across at least 3–4 independent error examples before being included in the evaluation report.

---

## 8. Train / Validation / Test Split

| Split | Examples | Share |
|-------|----------|-------|
| Train | 140 | 70% |
| Validation | 30 | 15% |
| Test | 30 | 15% |

Stratified split to maintain label balance across all three sets. The test set is held out until final evaluation — it is never used for hyperparameter tuning or early stopping decisions.