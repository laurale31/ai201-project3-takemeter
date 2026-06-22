# planning.md — TakeMeter: r/DetectiveConan Discourse Classifier

## Community

r/DetectiveConan is the main English-language Reddit community for the manga/anime
series *Detective Conan* (also known as *Case Closed*). The sub is active across
both episodic reactions and long-running story arc discussions, making it a strong
fit for a classification task — posts range from deep lore theory-crafting to
immediate emotional reactions to chapter drops, with plenty of confident opinions
in between. The discourse is text-heavy and varied enough that the distinctions
between labels are real and meaningful to regular participants.

## Labels

**`analysis`** — The post makes a structured argument about the plot, characters,
case writing, or story arcs, supported by specific verifiable evidence from the
series (a named chapter, a character's behavior pattern, a recurring clue). The
reasoning would hold up even if you stripped the opinion framing.

- Example 1: "Rum's identity is almost certainly Karasuma Renya himself — the
  'that person' phrasing used by org members implies someone mythologized rather
  than actively present, and the 1000-year-old darkness hint maps to a figurehead,
  not an operative."
- Example 2: "Conan's disguise plots have gotten lazier since volume 60 — compare
  the Haibara intro arc where the disguise had real stakes vs. recent cases where
  it's resolved in one panel with no tension."

**`hot_take`** — A bold, confident opinion stated without genuine supporting
reasoning. May gesture at evidence, but the claim drives the post, not the argument.

- Example 1: "Sera is the most underutilized character in the whole series and
  Gosho just forgot about her."
- Example 2: "The Black Organization arc should have ended 30 volumes ago — the
  series has been stalling forever."

**`reaction`** — An immediate emotional response to a specific chapter, episode,
or moment. Expressing a feeling, not making an argument.

- Example 1: "THAT CHAPTER. Akai and Amuro in the same frame finally?? I've been
  waiting years for this."
- Example 2: "Just finished the Vermouth arc for the first time. I can't believe
  I slept on this series."

## Hard Edge Cases

**The decorated-evidence problem:** Posts that cite specific facts (chapter numbers,
character names, event counts) but use them for rhetorical effect rather than as
part of a genuine argument. These look like `analysis` on the surface but are
actually `hot_take`.

Decision rule: If the post provides specific, verifiable evidence that would support
the claim even if you removed the opinion framing, label it `analysis`. If the
evidence is vague, cherry-picked, or decorative — just enough to sound credible
but not genuinely reasoning — label it `hot_take`.

Example ambiguous post: "The Rum reveal is taking too long — Gosho has introduced
4 suspects over 200 chapters with no payoff, that's just bad writing."
→ The numbers are deployed for effect, not as part of a structured argument.
→ Label: `hot_take`

**The reaction-with-reasoning problem:** A post that starts as an emotional response
but includes a sentence of genuine reasoning. Decision rule: if the emotional
framing dominates and the reasoning is incidental, label it `reaction`. If the
reasoning is the main point and emotion is the framing, label it `analysis` or
`hot_take` accordingly.

## Data Collection Plan

- Source: r/DetectiveConan posts and comments (public)
- Collection method: Manual copy-paste into a CSV spreadsheet
- Target: ~70 examples per label (210 total) to ensure no label exceeds 70%
- If a label is underrepresented after 150 examples: actively seek that label type
  (e.g., search older theory megathreads for `analysis`, chapter discussion threads
  for `reaction`)
- Saved as: `data/detective_conan_labeled.csv` with columns: text, label, notes

## Evaluation Metrics

**Accuracy** tells me the overall fraction correct, but isn't enough on its own —
if one label dominates predictions, accuracy can look fine while the model is
ignoring two labels entirely.

**Per-class F1** (harmonic mean of precision and recall) is the right primary
metric here because all three labels matter equally. A model that gets `reaction`
right but fails on `analysis` and `hot_take` is not useful.

**Confusion matrix** tells me *which* label pairs the model confuses, which is
more actionable than aggregate numbers. I expect the `analysis`/`hot_take` boundary
to be the hardest — the confusion matrix will show whether that's true.

I'll report: overall accuracy, per-class F1 for all three labels, and a confusion
matrix for both the baseline and fine-tuned model.

## Definition of Success

The classifier is genuinely useful if:
- Overall accuracy ≥ 0.75 on the test set
- Per-class F1 ≥ 0.65 for all three labels (no label is being ignored)
- Fine-tuned model meaningfully outperforms the Groq zero-shot baseline
  (at least +10 points accuracy)

A model that hits these thresholds could realistically be used to tag posts in
the community or surface high-quality analysis threads.

## AI Tool Plan

**Label stress-testing:** Give Claude my label definitions and edge case rule and
ask it to generate 8–10 borderline posts that sit between `analysis` and `hot_take`.
If I can't classify them cleanly, tighten the definitions before annotating 200
examples.

**Annotation assistance:** I may use an LLM to pre-label batches of 20–30 examples
using my definitions, then review and correct every assignment myself. I will track
which examples were pre-labeled in the `notes` column of my CSV and disclose this
in the AI usage section of the README.

**Failure analysis:** After fine-tuning, I will paste my list of wrong predictions
into Claude and ask it to identify patterns (e.g., short posts, sarcasm, a specific
label pair). I will verify those patterns by re-reading the examples myself before
writing up my evaluation.