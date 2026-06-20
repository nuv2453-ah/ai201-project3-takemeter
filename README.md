# TakeMeter

A fine-tuned text classifier that evaluates discourse quality in online music discussion
communities, distinguishing structured analysis from opinion assertions and personal reactions.

## Community

r/LetsTalkMusic and r/indieheads. Both are active, text-heavy subreddits where the same thread
can contain detailed musicological arguments, one-line opinions, and personal anecdotes about
buying CDs. The range of discourse styles makes the three-way classification non-trivial — a
keyword approach would fail because all three label types discuss the same artists, albums, and
topics.

## Label Taxonomy

**analysis** — a structured argument backed by specific, verifiable evidence: named techniques,
artists, songs, dates, or direct comparisons. Strip the attitude and a checkable claim remains.
- "Black Sabbath's debut dropped in February 1970, and it's the first album that's metal the whole way through."
- "In terms of songwriting Bush is more conventional — single linear themes — while Amos is often surrealist."

**hot_take** — a confident opinion or claim asserted with little or no supporting evidence; it
asserts rather than argues.
- "Sabbath are just a blues rock group."
- "Spotify can get fucked."

**reaction** — a personal anecdote, vibe, or in-the-moment emotional response with no general
argument.
- "I bought it in a charity shop for a pound because I like the Supremes."
- "The Strokes — my first time ever seeing them. Splurged on floors. Stoked!"

## Data Collection

210 comments collected from seven threads across both subreddits: the Kate Bush accessibility
thread (144 comments), the Helter Skelter / first metal song debate (113 comments), "What was
the last CD you bought and why?" (175 comments), "Is there a sense of what reddit music taste
entails?" (66 comments), "Do you believe in hidden treasures?" (52 comments), the Albert Hammond
Jr. / Strokes news thread, and the June 2026 Concert Roll Call.

**Label distribution:** analysis 79 (37.6%), hot_take 65 (31.0%), reaction 66 (31.4%). No class
exceeds 40%.

### Difficult-to-label examples

1. "Streaming services compress the audio, which reduces the sound quality." — Could be
   hot_take (punchy tone) or analysis (verifiable technical claim). Decision: analysis, because
   the core claim survives if you strip the attitude.

2. "I rebuy my favorite movie soundtracks on CD because streaming never has all the tracks."
   — Could be reaction (personal habit) or analysis (reasoned explanation about licensing gaps).
   Decision: analysis, because the reasoning generalizes beyond the individual.

3. "The Pi song goes hard." — Could be reaction (expressing a vibe) or hot_take (evaluative
   judgment). Decision: hot_take, because it asserts quality rather than describing a personal
   experience.

## Fine-Tuning Approach

**Base model:** distilbert-base-uncased (66M parameters), fine-tuned on Google Colab with a T4 GPU.

**Training setup:** 3 epochs, learning rate 2e-5, batch size 16, using the Hugging Face Trainer API.
Default hyperparameters were kept because the dataset is small (147 training examples) and
DistilBERT's recommended fine-tuning range is 2e-5 to 5e-5. With only 147 examples, more epochs
or a higher learning rate risked overfitting.

**Data split:** 147 train / 31 validation / 32 test (70/15/15), stratified.

## Baseline

**Model:** Groq llama-3.3-70b-versatile (zero-shot, temperature 0).

**Prompt:** provided the three label definitions with one example each, instructed the model to
output only the label name. No few-shot examples from the training set were included — the
baseline reflects pure zero-shot performance with definitions only.

## Evaluation Report

### Overall accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (Groq Llama 3.3 70B) | 0.938 |
| Fine-tuned DistilBERT | 0.594 |

Fine-tuning regression: -0.344. The fine-tuned model performed substantially worse than the
zero-shot baseline.

### Per-class metrics — fine-tuned model

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| analysis | 0.55 | 1.00 | 0.71 | 12 |
| hot_take | 0.62 | 0.50 | 0.56 | 10 |
| reaction | 1.00 | 0.20 | 0.33 | 10 |
| **weighted avg** | **0.71** | **0.59** | **0.54** | **32** |

### Per-class metrics — baseline

| Label | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| analysis | 1.00 | 1.00 | 1.00 | 12 |
| hot_take | 1.00 | 0.80 | 0.89 | 10 |
| reaction | 0.83 | 1.00 | 0.91 | 10 |
| **weighted avg** | **0.95** | **0.94** | **0.94** | **32** |

### Confusion matrix — fine-tuned model

| | Predicted: analysis | Predicted: hot_take | Predicted: reaction |
|---|---|---|---|
| **True: analysis** | 12 | 0 | 0 |
| **True: hot_take** | 5 | 5 | 0 |
| **True: reaction** | 5 | 3 | 2 |

The dominant error pattern is hot_take and reaction being predicted as analysis. The model
learned to over-predict analysis (22 predictions vs 12 true), while under-predicting hot_take
(3 predictions vs 10 true). This is visible in hot_take and reaction being predicted as analysis. Reaction has 0.20 recall — the model catches only
2 out of 10 reactions.

### Wrong prediction analysis

**Error 1:** "Sabbath are just a blues rock group." (true: hot_take, predicted: analysis)
This is a pure opinion with no supporting evidence — a textbook hot take. But it names a specific
band and genre, which are the surface features the model associates with analysis. The model
learned that mentioning artists and genres signals analysis, without learning that an unsupported
assertion about them is a hot take.

**Error 2:** "I was at their show and went to the merch booth after; I only had 15 bucks so I
grabbed a CD because I felt bad buying nothing." (true: reaction, predicted: analysis)
A personal anecdote with specific details (dollar amount, merch booth, show). The model treats
specificity as an analysis signal, but this is narrative specificity — a story, not an argument. The
model cannot distinguish "specific because it's evidence" from "specific because it's a memory."

**Error 3:** "Now THAT is some high quality information, never knew any of that before. Thanks
for the history lesson!" (true: reaction, predicted: hot_take)
An enthusiastic response to someone else's post. The model likely keyed on the evaluative
phrasing ("high quality information") rather than recognizing that the comment is reacting to
another person's content, not making its own claim.

### Sample classifications

| Post | Predicted | Confidence | Correct? |
|---|---|---|---|
| "Black Sabbath's debut dropped in February 1970..." | analysis | 0.52 | Yes |
| "Sabbath are just a blues rock group." | analysis | 0.38 | No (true: hot_take) |
| "I bought it in a charity shop for a pound..." | reaction | 0.41 | Yes |
| "The Pi song goes hard." | hot_take | 0.37 | Yes |
| "Lemonade by Beyonce, for the visual album DVD." | analysis | 0.34 | No (true: reaction) |

The correct analysis prediction is reasonable: the comment cites a specific date (February 1970)
and makes a verifiable historical claim, which are exactly the features the label definition calls for.

## Reflection: What the Model Learned vs. What Was Intended

The model learned that **mentioning specific artists, albums, dates, and genre names signals
analysis**. This is a shallow proxy for what the label actually captures. The real distinction is
whether the comment *makes an argument backed by evidence* versus *merely referencing those
things in passing*. A reaction that says "I bought Sabbath's debut because I love it" mentions
the same entities as an analysis of Sabbath's historical significance, but only one is making an
argument.

The 70B baseline model already understands this distinction — it can parse whether a comment
is asserting, arguing, or narrating, because that's a pragmatic-language skill that scales with
model size. DistilBERT at 66M parameters, trained on 147 examples, falls back on lexical
shortcuts: if the text mentions a band name and a date, predict analysis.

This suggests the task as defined is **above the complexity threshold for small-model
fine-tuning with this dataset size**. The labels capture a pragmatic distinction (is this person
arguing, opining, or narrating?) that requires understanding intent, not just content. To close the
gap, the most promising directions would be: (1) a much larger annotated dataset (1000+
examples) so the model sees enough variety to learn the distinction, (2) fine-tuning a larger
base model that already has some pragmatic understanding, or (3) redefining the labels around
more surface-level features that a small model can actually learn from limited data.

## Spec Reflection

**How the spec helped:** the requirement to define labels with one-sentence definitions and
identify an edge case before annotating forced the analysis/hot_take decision rule upfront. Without
that, the boundary would have drifted during annotation — some days a comment with one cited
fact would be analysis, other days hot_take. Having the rule written down ("strip the opinion; is a
checkable claim left?") kept annotation consistent across 210 examples.

**Where implementation diverged:** the planning document set a success threshold of per-class
F1 >= 0.70 and a 15-point accuracy improvement over baseline. The fine-tuned model achieved
neither — it regressed by 34 points. Rather than treating this as a failure to hide, the evaluation
report leans into it: the regression itself is the finding, and the analysis of why it happened
(pragmatic distinctions exceed small-model capacity at this dataset size) is more useful than a
model that barely clears an arbitrary bar.

## AI Usage

1. **Dataset labeling:** directed Claude to read 210 comments from six r/LetsTalkMusic and
   r/indieheads threads and assign each one a label (analysis, hot_take, or reaction) based on
   the definitions in planning.md. Reviewed the output and corrected borderline cases — for
   example, flipping comments that cited a single fact in passing from analysis to hot_take, and
   confirming that personal-habit comments with generalizable reasoning stayed as analysis.

2. **Error pattern analysis:** after fine-tuning, pasted the 13 misclassified examples into Claude
   and asked it to identify common patterns. It surfaced the "specificity as false analysis signal"
   pattern — that the model treats any mention of a named artist or date as evidence of analysis,
   even in reactions and hot takes. Verified this by checking the confusion matrix: 11 of 13 errors
   were predictions of analysis when the true label was hot_take or reaction.

## Demo Video

https://youtu.be/1yLtOszq6SE
