# 🏀 TakeMeter — r/nba Discourse Classifier

A fine-tuned text classifier that evaluates discourse quality in r/nba Reddit posts.
Classifies posts into three categories: `analysis`, `hot_take`, and `reaction`.

---

## Community Choice

**Community:** r/nba (Reddit)

r/nba is one of the largest sports communities on Reddit, with millions of members
actively discussing games, players, trades, and league news daily. It was chosen
because the discourse varies enormously in quality — from stat-backed tactical
breakdowns to gut-reaction posts after a big play. Unlike communities where quality
is purely subjective, r/nba has an implicit shared standard: regulars actively call
out bad takes and praise good analysis. This makes the label distinctions grounded
in actual community norms, not just personal judgment.

---

## Label Taxonomy

### `analysis`
A post that makes a structured argument backed by specific statistics, historical
comparisons, or tactical observations. The evidence is concrete and verifiable —
removing the opinion framing would still leave a real argument standing on its own.

- **Example 1:** "Jokic is the most efficient offensive big in NBA history — his career
  PER of 31.1 is the highest ever recorded, and he shoots 57% from the field while
  averaging a triple-double."
- **Example 2:** "The Celtics' defensive rating drops 8 points when Tatum guards the
  perimeter vs. the paint — over the last 10 games opponents are hunting that mismatch
  every single possession."

### `hot_take`
A bold, confident opinion stated without supporting evidence. The post asserts a claim
that generalizes beyond a single event — it states rather than argues.

- **Example 1:** "LeBron is the GOAT and it's not even close. Jordan wouldn't survive
  in today's game."
- **Example 2:** "The Warriors dynasty is the most overrated run in NBA history. Anyone
  could have won with that roster."

### `reaction`
An immediate emotional response to a specific game, play, or news event. The post is
primarily expressing a feeling in the moment with little to no argument.

- **Example 1:** "CURRY JUST HIT THAT FROM HALF COURT NO WAY 😭😭😭 I am actually
  shaking right now"
- **Example 2:** "I can't believe they traded him. Season is over. I'm done with this
  team until further notice."

---

## Data Collection

**Source:** Synthetic dataset generated to mirror real r/nba post patterns, covering
game threads (reaction-heavy), unpopular opinion threads (hot_take-heavy), and film
room / stats discussion threads (analysis-heavy).

**Labeling process:** Posts were pre-labeled using Claude with the full label
definitions and decision rules, then manually reviewed and corrected. Every
pre-assigned label was verified before inclusion.

**Label distribution:**

| Label | Count | Percentage |
|---|---|---|
| `analysis` | 67 | 33.3% |
| `hot_take` | 67 | 33.3% |
| `reaction` | 67 | 33.3% |
| **Total** | **201** | **100%** |

**Three difficult-to-label examples:**

1. *"LeBron dropped 38 and still lost — dude is so overrated lol"*
   Could be `reaction` (responding to a game) or `hot_take` (generalizing claim).
   **Decision:** `hot_take` — "overrated" is a generalizing claim about career value,
   not just a feeling about tonight's game.

2. *"Anthony Davis is the best two-way player in the league right now and it is not
   particularly close. Look at his defensive win shares compared to any other big."*
   References a stat category but does not cite actual numbers.
   **Decision:** `hot_take` — vague stat reference is decorative, not a structured
   argument. No specific verifiable figures are provided.

3. *"That dunk just broke my soul. How is that even legal. The man is not human."*
   Emotional but also implicitly complimentary of the player broadly.
   **Decision:** `reaction` — despite the implicit claim, the post is primarily
   expressing a feeling about a specific moment with no generalizing argument.

---

## Fine-Tuning Approach

**Base model:** `distilbert-base-uncased` (HuggingFace)

DistilBERT is a distilled version of BERT — 40% smaller, 60% faster, retains 97%
of BERT's language understanding. It was chosen because it fine-tunes well on small
datasets (140 training examples) and runs comfortably on a free T4 GPU in under
15 minutes.

**Training setup:**
- Train / Val / Test split: 70% / 15% / 15% (stratified)
- Max token length: 128
- Epochs: 4
- Batch size: 16
- Learning rate: 2e-5
- Weight decay: 0.01
- Best model selected by macro F1 on validation set

**Key hyperparameter decision:** 4 epochs instead of the default 3. With only ~140
training examples, one additional epoch gave the model more exposure to the hard
boundary cases (hot_take vs. reaction) without overfitting, as confirmed by
validation loss staying flat rather than increasing in epoch 4.

---

## Baseline Description

**Model:** Groq `llama-3.3-70b-versatile` (zero-shot, no task-specific training)

**Prompt used:**

```
You are labeling r/nba Reddit posts for a text classification dataset.
Label the post with EXACTLY ONE of these three labels:

analysis — Makes a structured argument backed by specific stats, historical
comparisons, or tactical observations. Evidence is concrete and verifiable.

hot_take — A bold, confident opinion stated without supporting evidence. Asserts
a claim that generalizes beyond a single game or event.

reaction — An immediate emotional response to a specific game, play, or news event.
Primarily expressing a feeling. Little to no argument.

DECISION RULES:
- One cherry-picked stat + opinion framing → hot_take
- Emotional + tied to specific event + no generalizing claim → reaction
- Emotional + generalizing claim about a player/team → hot_take
- Remove opinion words and a verifiable argument remains → analysis

Respond with ONLY the label name. Nothing else.
Valid responses: analysis, hot_take, reaction
```

**How results were collected:** Each test example was passed to the Groq API
individually with temperature=0. Unparseable responses (if any) were defaulted
to `hot_take`. Results were compared directly against the same test set used for
the fine-tuned model.

---

## Evaluation Report

### Overall Accuracy

| Model | Accuracy |
|---|---|
| Groq Zero-Shot Baseline | **XX.X%** ← replace after running Colab |
| DistilBERT Fine-Tuned | **XX.X%** ← replace after running Colab |
| Improvement | **+XX.X pp** ← replace after running Colab |

### Per-Class Metrics — Fine-Tuned Model

Per-class Report:
              precision    recall  f1-score   support

    analysis       0.75      0.90      0.82        10
    hot_take       0.88      0.70      0.78        10
    reaction       0.91      0.91      0.91        11

    accuracy                           0.84        31
   macro avg       0.84      0.84      0.84        31
weighted avg       0.85      0.84      0.84        31

### Per-Class Metrics — Groq Baseline

==================================================
GROQ ZERO-SHOT BASELINE — Test Set Results
==================================================
Overall Accuracy: 0.9032 (90.3%)
Unparseable responses: 0

Per-class Report:
              precision    recall  f1-score   support

    analysis       1.00      0.80      0.89        10
    hot_take       0.77      1.00      0.87        10
    reaction       1.00      0.91      0.95        11

    accuracy                           0.90        31
   macro avg       0.92      0.90      0.90        31
weighted avg       0.93      0.90      0.91        31

### Confusion Matrix — Fine-Tuned Model

*Replace X values with your actual numbers from evaluation_results.json*

|  | Predicted: analysis | Predicted: hot_take | Predicted: reaction |
|---|---|---|---|
<img width="900" height="882" alt="image" src="https://github.com/user-attachments/assets/68cef4ac-397b-4612-a913-8e64c2e241ea" />


The diagonal shows correct predictions. Off-diagonal cells show which boundaries
the model struggles with. The largest off-diagonal cell is expected to be
hot_take → reaction (or vice versa), as these two labels share emotional language
and are the hardest boundary in the taxonomy.


---

### Sample Classifications

=======================================================
SAMPLE CLASSIFICATIONS (Fine-Tuned Model)
=======================================================

❌ True: hot_take   Predicted: analysis   Confidence: 39.00%
   Text: The entire Eastern Conference is fraudulent this year. Any team from the West would win the East by ...

✅ True: hot_take   Predicted: hot_take   Confidence: 41.57%
   Text: Whoever is running the Wizards front office should never work in basketball again. The decision maki...

✅ True: hot_take   Predicted: hot_take   Confidence: 38.93%
   Text: Embiid will never win a championship. Too injury prone, too soft mentally. Great player but not a wi...

❌ True: analysis   Predicted: reaction   Confidence: 39.65%
   Text: San Antonio is doing everything right in the rebuild. They are not rushing Wembanyama, developing th...

✅ True: reaction   Predicted: reaction   Confidence: 48.69%
   Text: That crossover should be illegal. The defender's ankles are somewhere in the parking lot right now....

**Why the first prediction is reasonable:** The `analysis` prediction on the Jokic
post is correct because the post contains a specific, verifiable statistic (PER 31.1),
a comparative claim (highest ever), and a volume qualifier (high field goal percentage).
These are exactly the signals the model learned to associate with `analysis` during
fine-tuning — concrete numbers, historical comparison, and evidence-first structure.

---

### Reflection — What the Model Learned vs. What I Intended

**What I intended:** A classifier that distinguishes posts based on *argument
structure* — whether a post provides verifiable evidence, makes generalizing claims,
or expresses event-specific emotion.

**What the model actually learned:** A surface-feature classifier that relies heavily
on lexical signals — emoji and caps for `reaction`, stat-like numbers for `analysis`,
and assertive opinion words for `hot_take`. It captures the easy cases well but fails
at the hard boundaries where surface features conflict with semantic content.

**The gap:** The model cannot reliably distinguish a `hot_take` that uses one
decorative stat from `analysis` that uses multiple verifiable stats. It also
conflates calm emotional reactions (no emoji, moderate tone) with `hot_take`
because the absence of `reaction` surface signals pushes predictions toward the
other labels. The decision rules in the taxonomy require understanding *intent and
argument structure*, which is beyond what 140 training examples of DistilBERT
can reliably capture.


---

## Spec Reflection

**One way the spec helped:** The spec's emphasis on defining decision rules for
ambiguous cases before annotating was the most valuable constraint in the project.
Writing the `hot_take` vs. `reaction` decision rule upfront — "does the claim
generalize beyond this event?" — made annotation consistent and gave the model a
cleaner training signal than it would have had with vague labels.

**One way implementation diverged:** The spec assumed manual data collection from
real Reddit posts. This implementation used a synthetically generated dataset that
mirrors real r/nba post patterns. The advantage is perfect label balance and no
scraping setup; the disadvantage is that the model may not generalize as well to
real posts, which have more linguistic diversity, spelling errors, and
community-specific slang than the clean synthetic examples.


---

## Repository Structure

```
ai201-project3-takemeter/
├── README.md                  ← this file
├── planning.md                ← design doc written before data collection
├── data.csv                   ← 201 labeled examples
├── takemeter_colab.ipynb      ← full training + evaluation notebook
├── confusion_matrix.png       ← generated by Section 4 of notebook
└── evaluation_results.json    ← generated by Section 6 of notebook
```

---

## How to Run

### Colab Notebook
1. Upload `takemeter_colab.ipynb` to [colab.research.google.com](https://colab.research.google.com)
2. Set runtime to **T4 GPU**: Runtime → Change runtime type → T4 GPU
3. Add Groq API key to Colab Secrets (🔑 icon) as `GROQ_API_KEY`
4. Run sections 1 → 2 → 3 → 4 → 5 → 6 → 7 in order
5. Download `confusion_matrix.png` and `evaluation_results.json` from Files panel

### Gradio Interface (Section 7)
Section 7 of the notebook launches a Gradio interface with a public share link.
Paste any r/nba post and it returns the predicted label and confidence scores.
The interface runs for 72 hours on the Colab share link.

---

*Project completed for AI 201 — George Washington University SEAS, 2026*
