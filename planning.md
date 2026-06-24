# TakeMeter — Planning Document
**Community:** r/nba
**Task:** Classify posts into analysis, hot_take, or reaction
**Author:** Kishan KC

---

## 1. Community

r/nba is one of the largest sports communities on Reddit, with millions of members
actively discussing games, players, trades, and league news daily. The discourse
ranges widely in quality — from stat-backed arguments and tactical breakdowns to
gut-reaction posts after a big play. This makes it an ideal fit for a classification
task: the signal differences between post types are real, meaningful to community
members, and observable in the text itself. Unlike communities where quality is
purely subjective, r/nba has an implicit shared standard — regulars actively call
out bad takes and praise good analysis — which means the label distinctions are
grounded in actual community norms, not just personal judgment.

---

## 2. Label Taxonomy

### `analysis`
A post that makes a structured argument backed by specific statistics, historical
comparisons, or tactical observations — the evidence is concrete and verifiable,
not just decorative.

- **Example 1:** "Jokic is the most efficient offensive big in NBA history — his
  career PER of 31.1 is the highest ever recorded, and he shoots 57% from the
  field while averaging a triple-double."
- **Example 2:** "The Celtics' defensive rating drops 8 points when Tatum guards
  the perimeter vs. the paint — you can see it in their last 10 games, opponents
  are hunting that mismatch every possession."

### `hot_take`
A bold, confident opinion stated without supporting evidence — the post asserts
a claim rather than arguing for it, and the claim generalizes beyond a single event.

- **Example 1:** "LeBron is the GOAT and it's not even close. Jordan wouldn't
  survive in today's game."
- **Example 2:** "The Warriors dynasty is the most overrated run in NBA history.
  Anyone could've won with that roster."

### `reaction`
An immediate emotional response to a specific game, play, or news event — the
post expresses a feeling in the moment with little to no argument.

- **Example 1:** "CURRY JUST HIT THAT FROM HALF COURT NO WAY 😭😭😭"
- **Example 2:** "I can't believe they traded him. Season is over. I'm done."

---

## 3. Hard Edge Cases

### Boundary 1: `hot_take` vs `reaction`
Both can respond to a specific event and use emotional language. The decision rule:

> If the post is primarily venting a feeling about a single moment or game, it is
> `reaction`. If it is primarily asserting a claim about a player or team that
> generalizes beyond this event — even if triggered by it — it is `hot_take`.

**Ambiguous example:** *"LeBron dropped 38 and still lost — dude is so overrated lol"*
This looks like a reaction to a game, but "overrated" is a generalized claim about
LeBron's career value. → **Label: `hot_take`**

### Boundary 2: `hot_take` vs `analysis`
A post that cites one cherry-picked stat to support a bold claim is still a
`hot_take` — the stat is decorative, not genuinely reasoning. A post is `analysis`
only if removing the opinion framing would still leave a substantive, verifiable
argument standing on its own.

**Ambiguous example:** *"KD is overrated — he never won without a superteam"*
This cites an implicit historical fact but makes no verifiable structured argument.
→ **Label: `hot_take`**

**Decision rule:** Ask — if I removed all opinion words, would a real argument
remain? If yes → `analysis`. If no → `hot_take`.

---

## 4. Data Collection Plan

### Source
Public posts and comments from r/nba collected manually via Reddit's web interface.
No authentication required — all content is publicly visible.

### Target Distribution
| Label | Target Count | Minimum |
|---|---|---|
| `analysis` | 70 | 60 |
| `hot_take` | 70 | 60 |
| `reaction` | 70 | 60 |
| **Total** | **210** | **200** |

Aiming for roughly equal distribution across all three labels to avoid the model
learning to predict one class constantly.

### Collection Strategy
- Browse r/nba post comments and top-level posts across different threads:
  game threads (rich in `reaction`), discussion threads (rich in `analysis` and
  `hot_take`), and "unpopular opinion" or "hot take" threads
- Collect text into a CSV with columns: `text`, `label`, `notes`
- Use an LLM (Claude) to pre-label batches of ~50 posts at a time using the
  definitions above, then manually review and correct every single label before
  saving

### If a Label is Underrepresented
If any label falls below 60 examples after 200 posts, deliberately seek out
thread types known to produce that label — game threads for `reaction`,
analytical post threads for `analysis`, debate/opinion threads for `hot_take`.

### Pre-labeling Disclosure
Claude will be used to pre-label examples. Every pre-assigned label will be
manually reviewed and corrected. Pre-labeled examples will be flagged in the
`notes` column as "pre-labeled, reviewed".

---

## 5. Evaluation Metrics

### Why Accuracy Alone Is Not Enough
If the dataset has equal class distribution (~33% per label), a model that always
predicts `hot_take` would achieve 33% accuracy — clearly useless. Even with
balanced data, accuracy hides which specific boundaries the model is failing on.

### Metrics I Will Use

**Per-class F1 score** — the primary metric. F1 is the harmonic mean of precision
and recall, and it tells me whether the model is learning each boundary, not just
the easiest one. I need all three classes to have meaningful F1, not just overall
accuracy.

**Confusion matrix** — shows exactly which label pairs are being confused and in
which direction. A model that confuses `hot_take` → `reaction` is making a
different kind of mistake than one that confuses `hot_take` → `analysis`, and the
confusion matrix surfaces that.

**Overall accuracy** — reported for both models for direct comparison, but
interpreted alongside F1.

**Baseline delta** — the gap between Groq zero-shot accuracy and fine-tuned
accuracy. This tells me whether fine-tuning actually helped and by how much.

---

## 6. Definition of Success

The classifier is genuinely useful if:

- **Overall accuracy ≥ 70%** on the test set for the fine-tuned model
- **Per-class F1 ≥ 0.60 for all three labels** — no label is being completely
  ignored by the model
- **Fine-tuned model outperforms the Groq zero-shot baseline by at least 10
  percentage points** in overall accuracy
- The confusion matrix shows no single off-diagonal cell accounting for more
  than 40% of one class's errors (i.e., no complete boundary collapse)

If the model hits these thresholds, it would be reliable enough to use as a
lightweight triage tool — e.g., flagging low-quality posts in a community
moderation dashboard.

---

## 7. AI Tool Plan

### Label Stress-Testing
I will give Claude my label definitions and edge case rules and ask it to generate
10–15 posts that sit at the boundary between two labels. If it produces posts I
cannot classify cleanly with my current rules, I will tighten the definitions
before annotating 200 examples. This stress-test happens before any data collection.

### Annotation Assistance
I will use Claude to pre-label batches of ~50 posts at a time. My prompt will
include the full label definitions and decision rules from Section 2 and 3 above,
and instruct it to output only the label name and a one-sentence reason. I will
review and correct every pre-assigned label myself — no batch will be accepted
without genuine review. Pre-labeled examples will be flagged in the `notes` column.

### Failure Analysis (Stretch: Error Pattern Analysis)
After fine-tuning, I will paste all misclassified test examples into Claude and
ask it to identify common patterns — e.g., short posts, sarcastic phrasing, a
specific label pair that keeps getting confused. I will then verify those patterns
myself by re-reading the examples, and report both what the AI found and what I
confirmed or corrected.

---

## 8. Stretch Features

### Error Pattern Analysis (Selected)
After evaluating the fine-tuned model, I will systematically analyze wrong
predictions to identify whether errors cluster around a specific label pair,
post length, use of sarcasm, or topic type. I will use Claude to surface candidate
patterns, then verify manually. Findings will be reported in the README evaluation
section.

---

*Document last updated: Milestone 2*
*Will be updated before starting stretch features per spec requirements*