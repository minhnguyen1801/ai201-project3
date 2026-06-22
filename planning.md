# Project 3 — Text Classifier for r/leagueoflegends Posts

## Community

**Subreddit:** r/leagueoflegends (~7M members, Reddit's largest *League of Legends* community)

r/leagueoflegends is a strong fit for a three-way text classifier for several reasons:

- **Volume and freshness.** It is one of the most active gaming subreddits on Reddit, generating hundreds of posts per day. This makes it easy to collect a balanced dataset within the project's data-collection window, and ensures the classifier is trained on current discourse (live patches, recent esports events, fresh skin reveals).
- **Genuinely distinct discourse modes.** The community does not just do one thing. On any given day the front page mixes (a) actionable gameplay knowledge ("how to play X matchup"), (b) substantive arguments about balance and Riot's design ("Y is overtuned and here's why"), and (c) raw emotional reactions ("THAT BARON STEAL ENDED MY LIFE"). These three modes map cleanly onto our STRATEGY / OPINION / HYPE taxonomy.
- **Community-recognized distinctions.** The difference between a guide, a balance-complaint, and a hype clip is something the community itself recognizes — these correspond loosely to how users self-sort content (guide flairs, discussion flairs, esports/clip flairs). Labels that are meaningful to the community are easier to annotate consistently and are more defensible.
- **Rich, self-contained text.** Most posts carry enough text (title + body) to classify without external context such as embedded video, which keeps the task tractable for a text-only model.

The combination of high volume, three naturally-occurring discourse modes, and community-recognized boundaries makes the label distinctions both meaningful and learnable.

## Labels

The taxonomy has **3 mutually-exclusive, exhaustive labels.**

### STRATEGY
Posts that provide **actionable gameplay information**: champion guides, build recommendations, macro/micro mechanics tips, game-system explanations.
**Test:** "Could another player use this to win more games?"

- **Example 1:** "How to play Jhin into a poke lane — buy a Doran's Shield, hold your W until they step up for a CS, and roam mid the moment your support has priority. Here's the trinket timing."
- **Example 2:** "Macro tip for low elo: after you take a tower, don't immediately recall — ping your team to group and contest the next objective. Vision before objectives wins games."
- **Uncertain case:** "Riot stealth-buffed Conqueror's healing again and now bruisers are unkillable in extended fights — you basically have to itemize Grievous Wounds turn one." This *reads* like a build tip (buy Grievous Wounds) but the dominant purpose is arguing the rune is overtuned → leans **OPINION**. Borderline because it does contain a usable directive.

### OPINION
Posts that make a **substantive argument or judgment** about the game's state, balance, Riot's design decisions, or the competitive meta. The post is persuading you that something *is* a certain way.
**Test:** "Is the post arguing something is good/bad/broken about design or balance?"

- **Example 1:** "The new turret-plating gold is too high — it rewards early leads so hard that comebacks barely exist anymore. Riot needs to scale plating value down or games will keep snowballing."
- **Example 2:** "Vanguard anti-cheat was the right call. Yes it's invasive, but the scripting epidemic in high elo was killing the competitive integrity of the ladder, and the data since launch backs that up."
- **Uncertain case:** "Faker is the GOAT and nobody is close." Feels like a hot take (argument), but if it's a single emotional line with zero supporting reasoning it slides toward **HYPE**. Borderline depending on whether reasoning is offered.

### HYPE
Posts expressing **immediate excitement, celebration, or emotional reaction** to a play, esports result, skin reveal, or memorable moment. Little to no reasoning.
**Test:** "Is the post primarily sharing a feeling rather than advising or arguing?"

- **Example 1:** "T1 JUST WON WORLDS AGAIN I AM SHAKING, FAKER FOREVER 🏆🐐"
- **Example 2:** "The new Spirit Blossom Yasuo splash art is absolutely GORGEOUS, instabuying the second it drops."
- **Uncertain case:** "Insane outplay from my Riven game last night [clip] — flash-E over the wall to dodge the ult was so clean." Mostly celebration (HYPE), but if the body breaks down *how* to execute the flash-E it could tip to **STRATEGY**. Borderline by dominant purpose.

## Hard Edge Cases and Decision Rules

1. **Strategy vs Opinion.** If the post tells you *HOW to win* → **Strategy**. If it argues *WHAT IS WRONG* with the game → **Opinion**.
   *Tiebreaker:* a post that opens with "I think X is broken/OP/bad" → **Opinion**, even if it includes some gameplay info. The framing signals persuasive intent over instructional intent.

2. **Opinion vs Hype.** If the post makes a claim **supportable by reasoning** → **Opinion**. If it's **purely emotional with no argument** → **Hype**. The litmus is whether you could disagree with the post on substance, or only react to its mood.

3. **Strategy vs Hype.** Classify by **dominant purpose**. Hype never gives advice; Strategy never primarily expresses excitement. A clip post with a celebratory tone but a step-by-step breakdown of the mechanic is Strategy; the same clip with only "SO CLEAN 🔥" is Hype.

## Data Collection Plan

- **Source:** Reddit API via **PRAW** (Python Reddit API Wrapper), authenticated with a registered Reddit script app (client id / secret / user agent stored outside the repo).
- **Target volume:** ~**70–80 posts per label** (~210–240 total) of human-confirmed, labeled examples.
- **Sampling strategy:** Pull from multiple PRAW endpoints to diversify — `subreddit.hot`, `.new`, `.top(time_filter="month"/"year")`, plus flair-targeted searches (e.g., guide/discussion/esports flairs) to seed each class. Capture `title`, `selftext`, `flair`, `score`, `created_utc`, and `permalink` for traceability.
- **Filtering:** Drop posts with empty/near-empty bodies that aren't self-explanatory by title, removed/deleted posts, bot/mod stickies, and pure-link posts with no text. De-duplicate by post id.
- **Imbalance handling strategy:** HYPE and OPINION are naturally more frequent than long-form STRATEGY guides, so raw sampling will skew. To reach a roughly balanced ~70–80/label:
  - **Targeted oversampling** of the minority class via flair- and keyword-targeted queries (e.g., "guide", "how to", "matchup" for STRATEGY).
  - **Cap the majority classes** during collection so no single label dominates.
  - If residual imbalance remains at train time, apply **class weights** in the loss / **stratified train-test splits**, and report metrics **per class** (not just aggregate) so minority-class performance stays visible.
  - Document final per-class counts in the annotation log.

## Evaluation Metrics

We evaluate with a suite, not a single number:

- **Overall accuracy** — headline summary of correct predictions / total.
- **Per-class F1** — harmonic mean of precision and recall for each of STRATEGY / OPINION / HYPE; the primary guard against a model that "wins" on the majority class while failing a minority class.
- **Per-class precision and recall** — to diagnose *how* a class fails (over-predicting vs. missing it).
- **Confusion matrix** — the full 3×3 to surface systematic confusions (we expect STRATEGY↔OPINION and OPINION↔HYPE bleed, per the edge cases).

**Why accuracy alone is insufficient:** with any class imbalance, accuracy is dominated by the majority class. A classifier that always predicts the most common label could post a deceptively high accuracy while being useless on the other two classes. Per-class F1 and the confusion matrix expose exactly that failure, and align directly with our taxonomy's promise that all three distinctions are meaningful.

## Definition of Success

The classifier is considered successful if **all** of the following hold on the held-out test set:

- [ ] **Overall accuracy ≥ 75%**
- [ ] **No class F1 < 0.60** (every label is usefully learned, not just the easy ones)
- [ ] **Fine-tuned model beats the zero-shot baseline by ≥ 10 points** (accuracy / macro-F1), demonstrating that fine-tuning on our annotated data adds real value over an off-the-shelf prompt.

## AI Tool Plan

We will use AI assistance (e.g., Claude) at three points in the pipeline, with humans retaining final judgment:

- **Label stress-testing.** Before annotating, prompt the model with the taxonomy and ask it to generate adversarial / boundary posts that sit between labels (e.g., a balance-complaint that smuggles in a build tip). Use these to sharpen the decision rules and catch definition gaps early.
- **Annotation assistance.** Use the model as a *first-pass* labeler that proposes a label **plus a one-line rationale** for each collected post. Human annotators confirm or override — speeding throughput while keeping a human in the loop. Disagreements between model and human are flagged for adjudication and feed the inter-annotator analysis.
- **Failure analysis.** After training, feed misclassified test examples (with true vs. predicted labels) to the model and ask it to cluster the errors into patterns (e.g., "OPINION posts with strong emotional language misread as HYPE"). These clusters drive targeted data collection and rule refinement.

## Stretch Feature Plans

- [ ] **Inter-annotator reliability** — multiple annotators on a shared subset; compute Cohen's/Fleiss' κ and adjudicate disagreements.
- [ ] **Confidence calibration** — analyze predicted-probability vs. observed accuracy (reliability diagram / ECE); flag low-confidence predictions for review.
- [ ] **Error pattern analysis** — systematic clustering of misclassifications, tied to the confusion matrix and AI-assisted failure analysis.
- [ ] **Deployed interface** — a simple web app (e.g., Streamlit/Gradio) where a user pastes a post and gets a predicted label + confidence.

## Annotation Log

*(To be filled during Milestone 3.)*

| Date | Annotator | Posts Annotated | Label Distribution (STR / OPN / HYP) | Model–Human Agreement | Disagreements Adjudicated | Notes |
|------|-----------|-----------------|--------------------------------------|-----------------------|---------------------------|-------|
|      |           |                 |                                      |                       |                           |       |
