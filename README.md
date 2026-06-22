# TakeMeter: r/DetectiveConan Discourse Classifier

A fine-tuned text classifier that categorizes Reddit posts from r/DetectiveConan into three types: structured analysis, hot takes, and emotional reactions. Built as part of AI201 Project 3.

---

## Community Choice

I picked r/DetectiveConan because it's one of those communities where post quality varies a lot and in pretty distinct ways. Some people write out multi-paragraph theories with actual evidence from specific chapters. Some people just say "Rum is obviously Kuroda" and leave it at that. And then there's the whole category of posts that are just someone screaming in caps because a chapter dropped. Those three modes felt genuinely separable, which made it a good fit for a classification task.

The community is also text-heavy by nature since it's built around a mystery series, so people tend to actually write things out rather than just posting memes.

---

## Label Taxonomy

**`analysis`** — The post makes a structured argument about the plot, characters, case writing, or story arcs, backed by specific verifiable evidence from the series (a named chapter, a character's behavior pattern, a recurring clue). The reasoning holds up even if you remove the opinion framing.

Example 1: "Haibara's character arc is the most thematically complete in the series. She starts as someone who sees death as inevitable and slowly learns to value her own survival through her relationship with the Detective Boys. The shift isn't announced, it's shown through her behavior in the Vermouth arc where she chooses to trust Conan despite everything."

Example 2: "The reason Conan's disguise plots have become less tense over time is structural: in early arcs, being discovered as Shinichi meant immediate danger from the Black Organization. But now that the org is fractured and Bourbon already knows, the stakes of exposure have been diluted."

**`hot_take`** — A bold confident opinion stated without genuine supporting reasoning. The post might gesture at evidence but the claim is driving the bus, not the argument. If something looks like evidence but it's just there to make the assertion sound credible, it still counts as a hot take.

Example 1: "Sera is the most underutilized character in the whole series and Gosho just forgot about her."

Example 2: "The Black Organization arc should have ended 30 volumes ago. The series has been stalling forever."

**`reaction`** — An immediate emotional response to a specific chapter, episode, or moment. The post is expressing a feeling rather than making any kind of argument.

Example 1: "THAT CHAPTER. Akai and Amuro in the same frame finally?? I've been waiting years for this."

Example 2: "Just finished the Vermouth arc for the first time. I can't believe I slept on this series."

### The hard edge case: decorated evidence

The trickiest posts to label are ones that cite specific facts (chapter numbers, character names, counts of events) but use them for rhetorical effect rather than as part of a genuine argument. These look like `analysis` on the surface.

Decision rule: if the specific evidence would support the claim even without the opinion framing, label it `analysis`. If the evidence is vague, cherry-picked, or just there to sound credible, label it `hot_take`.

Example of this in practice: "The Rum reveal is taking too long. Gosho has introduced 4 suspects over 200 chapters with no payoff, that's just bad writing." The numbers are deployed for effect rather than as part of a structured argument about pacing. That's a `hot_take`.

---

## Data Collection

**Source:** r/DetectiveConan posts and comments, collected manually via copy-paste into a spreadsheet. Public posts only.

**Process:** I collected 45 real posts from the subreddit, then used Claude to generate synthetic examples to fill out the dataset to 210 total (70 per label). Every generated example was reviewed against the label definitions before being kept. The notes column in the CSV flags which rows were generated.

**Label distribution:**

| Label | Count |
|-------|-------|
| analysis | 70 |
| hot_take | 70 |
| reaction | 70 |
| **Total** | **210** |

### Three difficult annotation decisions

**1. The Kogoro=Karasuma theory post**
A very long post arguing that Kogoro Mouri is secretly Renya Karasuma, with an elaborate chain of reasoning involving APTX, Vermouth's protection, and Kogoro's unexplained backstory. It reads like a fan fic but it does build a structured argument using specific canon details. I labeled it `analysis` because the reasoning is genuine even if the conclusion is wild. Under my decision rule, the evidence would support the claim even without the opinion framing.

**2. The APTX inconsistency post**
Someone noticed that Mary Akai de-aged much more dramatically than Conan and Haibara did and asked how Vermouth could still be an adult if she used the same drug. I labeled this `analysis` because it's using specific canon data points (Mary's age, Vermouth's unchanged appearance over decades) to identify a real logical gap in the plot, not just expressing frustration.

**3. The movie review posts**
Several posts were post-watch reviews like "this movie was a blast, the mystery was good, the motive was weak." These sit between `reaction` and `hot_take` because they're expressing impressions but also making evaluative claims. I labeled them `reaction` because the emotional framing of having just finished something dominated over any structured critique.

---

## Fine-Tuning Approach

**Base model:** `distilbert-base-uncased` from HuggingFace

**Training setup:** Fine-tuned on Google Colab with a T4 GPU. Used the default hyperparameters from the starter notebook: 3 epochs, learning rate 2e-5, batch size 16, weight decay 0.01, 50 warmup steps.

**One hyperparameter decision worth noting:** I kept the learning rate at 2e-5 rather than bumping it up. With only 147 training examples, a higher learning rate risks overfitting fast, especially on a dataset where the `analysis`/`hot_take` boundary is subtle. 2e-5 is the standard starting point for fine-tuning BERT-family models on small datasets and it made sense to stick with it here.

**Dataset split:** 70/15/15 (147 train, 31 validation, 32 test), stratified by label.

---

## Baseline Description

**Model:** Groq's `llama-3.3-70b-versatile`, zero-shot (no examples provided).

**Prompt used:**

```
You are a classifier for Reddit posts from r/DetectiveConan.

Classify each post into exactly one of these three labels:

analysis — The post makes a structured argument about the plot, characters, case writing,
or story arcs, supported by specific verifiable evidence from the series (a named chapter,
a character's behavior pattern, a recurring clue). The reasoning would hold up even if you
stripped the opinion framing.

hot_take — A bold, confident opinion stated without genuine supporting reasoning. May
gesture at evidence, but the claim drives the post, not the argument. If specific evidence
is present but used decoratively rather than as a real argument, still label it hot_take.

reaction — An immediate emotional response to a specific chapter, episode, or moment.
Expressing a feeling, not making an argument.

Rules:
- Output ONLY the label. Nothing else. No punctuation, no explanation.
- The label must be exactly one of: analysis, hot_take, reaction
```

**How results were collected:** The notebook ran the Groq API on every example in the test set sequentially with a 0.1s delay between requests. All 32 responses were parseable.

---

## Evaluation Report

### Overall results

| Model | Accuracy |
|-------|----------|
| Zero-shot Groq baseline | 1.000 |
| Fine-tuned DistilBERT | 0.688 |

The fine-tuned model underperformed the baseline by 31 percentage points. That gap is worth examining carefully rather than just treating as a failure, and I'll get into why below.

### Per-class metrics

**Fine-tuned DistilBERT:**

| Label | Precision | Recall | F1 | Support |
|-------|-----------|--------|----|---------|
| analysis | 0.53 | 1.00 | 0.69 | 10 |
| hot_take | 0.75 | 0.27 | 0.40 | 11 |
| reaction | 1.00 | 0.82 | 0.90 | 11 |
| **macro avg** | **0.76** | **0.70** | **0.66** | **32** |

**Groq baseline:**

| Label | Precision | Recall | F1 | Support |
|-------|-----------|--------|----|---------|
| analysis | 1.00 | 1.00 | 1.00 | 10 |
| hot_take | 1.00 | 1.00 | 1.00 | 11 |
| reaction | 1.00 | 1.00 | 1.00 | 11 |

### Confusion matrix (fine-tuned model)

<img width="1050" height="750" alt="confusion_matrix" src="https://github.com/user-attachments/assets/823e86d2-01c3-4d84-abe2-93b216f561d4" />


### Three wrong predictions analyzed

**Wrong prediction 1:**
> "The series should have ended at volume 50. Everything after is bonus content at this point."
> True: `hot_take` | Predicted: `analysis` (confidence: 0.36)

This is a one-sentence opinion with zero supporting reasoning. There's no evidence, no argument, nothing. But the model predicted `analysis` anyway, probably because the post is talking about the series' plot structure in a way that pattern-matches to how analysis posts also talk about plot structure. The model learned "talks about the series" rather than "argues about the series."

**Wrong prediction 2:**
> "HOT TAKE: TEAM CONAN > BLACK ORGANIZATION"
> True: `hot_take` | Predicted: `analysis` (confidence: 0.36)

This one is almost funny. The post literally has the words "HOT TAKE" in it. But the model still called it `analysis`. This shows the model isn't reading meaning, it's pattern-matching on surface features, and a short post about the Black Organization apparently looks like analysis to it regardless of what the post actually says.

**Wrong prediction 3:**
> "Watching Conan try to protect everyone while being a literal child's body is genuinely heartbreaking when you think about it for too long."
> True: `reaction` | Predicted: `hot_take` (confidence: 0.35)

This one is more understandable. The post is expressing a feeling but it's a reflective one rather than a direct emotional explosion like "THAT CHAPTER." The sentence structure is more measured, which probably made it look more like a take than a reaction. The boundary between a thoughtful emotional observation and a hot take is genuinely fuzzy here, and the model's low confidence (0.35) suggests it wasn't sure either.

### Sample classifications

| Post (truncated) | Predicted Label | Confidence |
|------------------|-----------------|------------|
| "The Scarlet Return arc is structurally the strongest multi-episode arc in the series..." | analysis | 0.82 |
| "Haibara is a better character than Ran and it's not even close." | hot_take | 0.71 |
| "JUST FINISHED THE VERMOUTH ARC AND I CANNOT BREATHE..." | reaction | 0.94 |
| "Amuro/Bourbon is the most structurally complex character introduced post-volume 60. He is simultaneously working for the organization, working for the police..." | analysis | 0.78 |
| "Gosho should just end the series already." | hot_take | 0.61 |

The Vermouth arc reaction post at 0.94 confidence makes sense: all-caps, exclamation points, direct emotional statement, nothing that looks like an argument. The model learned that register well. The Scarlet Return analysis post at 0.82 also makes sense: it names a specific arc, talks about structure, uses words like "resolves" and "setup" that signal argumentative writing. Those are the easy cases.

---

## What the Model Learned vs. What I Intended

I wanted the model to learn the difference between *asserting* and *arguing*. What it actually learned was closer to the difference between *calm analytical vocabulary* and *emotional vocabulary*.

That's why `reaction` works well (F1 = 0.90): reaction posts have very distinct surface signals like caps, exclamation points, phrases like "I cannot breathe" or "I'm not okay." The model can catch those without understanding anything about the underlying distinction.

`analysis` also has high recall (1.00) but terrible precision (0.53). The model learned that "posts about plot, characters, and series structure" are analysis. But `hot_take` posts are also about those things. Since DistilBERT is a small model working with 147 training examples, it can't learn the subtle structural difference between a post that *argues* something and a post that just *states* something confidently. So it defaults to predicting `analysis` whenever it's unsure, which is most of the time for hot takes.

The decision boundary I cared about most was the `analysis`/`hot_take` distinction. That's also the one the model failed hardest on. 8 out of 11 hot takes in the test set were misclassified as analysis. All 8 wrong predictions were low-confidence (0.36 to 0.41), meaning the model wasn't sure but kept guessing analysis anyway. That's a sign the training signal wasn't strong enough to learn the distinction, not that the model learned the wrong boundary with confidence.

To actually fix this, I'd probably need more examples specifically of short opinionated posts (the hard hot takes), and maybe some contrastive pairs: the same claim written once as a hot take and once as a structured argument.

---

## Spec Reflection

The spec was most useful in Milestone 1 when it walked through the difference between weak and strong label taxonomies. The analysis/hot_take/reaction framework came directly from the example taxonomy in the spec (which used an NBA community), and adapting it to Detective Conan was mostly a matter of grounding it in the actual vocabulary of that community.

One place my implementation diverged from the spec: the spec says to run the baseline before fine-tuning so you're comparing models on an untouched test set. But the notebook runs in Section 1 through 6 order, with fine-tuning in Section 3 and the baseline in Section 5. In practice the test set is locked from the beginning by the train/val/test split in Section 2, so both models see the same test set regardless of which runs first. The spec's concern about test set contamination was valid in principle but the notebook architecture already handles it.

---

## AI Usage

**Label stress-testing:** Before collecting data, I gave Claude my label definitions and the `analysis`/`hot_take` edge case scenario and asked it to generate 5 borderline posts to test the definitions. Two of them (the decorated-evidence post and the reaction-with-reasoning post) exposed gaps in my original definitions that I had to tighten before I started annotating. This was genuinely useful because it caught ambiguity before I had labeled 200 examples with inconsistent rules.

**Dataset generation:** I collected 45 real posts manually, then asked Claude to generate additional labeled examples to reach 210 total with exactly 70 per label. I provided my label definitions and decision rule and reviewed every generated example before keeping it. A few generated hot_take posts were too subtle and got rewritten to be more clearly opinionated. The notes column in the CSV marks all generated rows.

**Failure analysis:** After seeing the wrong predictions from Section 4, I used Claude to identify patterns across all 10 errors. It flagged the `hot_take -> analysis` direction as the dominant pattern (8/10 errors), which I then verified by reading through each wrong prediction myself. The pattern held up. I also noticed independently that the confidence scores were all clustered between 0.35 and 0.41, which Claude hadn't flagged, so the manual review added something the AI analysis missed.
