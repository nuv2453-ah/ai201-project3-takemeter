# TakeMeter - Planning

## Community
r/LetsTalkMusic and r/indieheads. r/LetsTalkMusic is rich in self-text discussion threads with
high comment counts (100+), which produces varied discourse: theory-backed arguments,
opinionated assertions, and personal anecdotes all show up in the same thread. r/indieheads adds
Daily Music Discussion, General Discussion, and Concert Roll Call threads that skew toward
shorter reactions and quick takes. Together they give enough range to make the three-way
distinction non-trivial - a real classification task rather than a keyword match.

## Labels
1. analysis - makes a structured argument backed by specific, verifiable evidence (named
   techniques, artists, songs, dates, or direct comparisons); strip the attitude and a checkable
   claim remains.
   - "Black Sabbath's debut dropped in February 1970, and it's the first album that's metal the whole way through."
   - "In terms of songwriting Bush is more conventional - single linear themes - while Amos is often surrealist."
2. hot_take - a confident opinion or claim asserted with little or no supporting evidence; it
   asserts rather than argues.
   - "Sabbath are just a blues rock group."
   - "Spotify can get fucked."
3. reaction - a personal anecdote, vibe, or in-the-moment emotional response with no general argument.
   - "I bought it in a charity shop for a pound because I like the Supremes."
   - "The Strokes - my first time ever seeing them. Splurged on floors. Stoked!"

## Hard Edge Cases
The recurring ambiguity is the analysis/hot_take blend: a comment wraps an opinion in
analytical-sounding language, or cites one decorative fact. Decision rule: strip away the opinion
and the personal framing - is there still a specific, checkable claim (a date, technique, direct
comparison)? Yes -> analysis. An assertion with rhetorical flourish and no support -> hot_take.
If what remains is a person and a feeling -> reaction.

### Three difficult annotation cases

1. "Streaming services compress the audio, which reduces the sound quality."
   - Candidates: analysis vs hot_take. Reads with the punchy tone of a hot take, but the core
     claim (lossy streaming compresses audio) is specific and verifiable, not just opinion.
   - Decision: analysis.

2. "I rebuy my favorite movie soundtracks on CD because streaming never has all the tracks - it's
    hard to get the streaming rights for each one."
   - Candidates: reaction vs analysis. Framed as a personal buying habit, but it gives a reasoned,
     generalizable explanation (licensing gaps cause missing tracks) rather than just a feeling.
   - Decision: analysis. (Reaction is defensible here - a genuine boundary case.)

3. "The Pi song goes hard."
   - Candidates: hot_take vs reaction. No anecdote or event, so not a clean reaction, but no
     evidence either - just a confident evaluative judgment about the music.
   - Decision: hot_take.

## Data Collection Plan
210 comments pulled from high-comment threads across both subreddits: on r/LetsTalkMusic, the
Kate Bush thread, the "first heavy metal song / Helter Skelter" thread, "what was the last CD you
bought," "reddit music taste," and "hidden treasures"; on r/indieheads, the Strokes / Albert
Hammond Jr. news thread and the June 2026 Concert Roll Call. Target distribution ~40% analysis,
~35% hot_take, ~25% reaction, with no class above ~40%. If a class is underrepresented after the
first pass, pull more from a thread biased toward it (the metal-origins thread for analysis, the
Concert Roll Call for reactions). Achieved: analysis 79 (37.6%), hot_take 65 (31.0%), reaction 66 (31.4%).

## Evaluation Metrics
Overall accuracy plus per-class precision, recall, and F1, and a confusion matrix. Accuracy alone
hides class-imbalance effects - a model can look fine by nailing the most common class while
failing a rarer one. Per-class F1 shows whether each distinction is actually learned, and the
confusion matrix shows which label pair gets mixed up and in which direction (the analysis/hot_take
boundary is the expected trouble spot).

## Definition of Success
Per-class F1 >= 0.70 for all three labels on the held-out test set, AND at least a 15-point accuracy
improvement over the zero-shot Groq llama-3.3-70b-versatile baseline on the same test set. Below
that bar the classifier isn't reliable enough to flag take quality in a real community tool.

## AI Tool Plan
- Label stress-testing: ask an LLM to generate boundary-case posts (analysis/hot_take blends,
  reactions that cite specific details) and hand-label them to pressure-test the definitions before annotating.
- Failure analysis: after fine-tuning, feed misclassified examples to an LLM with the label
  definitions and ask it to surface error patterns, then verify each pattern against the actual cases.
