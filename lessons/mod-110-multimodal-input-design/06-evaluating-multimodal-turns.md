# 06. Evaluating Multimodal Turns

## Why this chapter

A vision-enabled voice agent that you cannot evaluate is a vision-enabled voice agent you cannot ship with confidence. The temptation is to run the standard LLM text-eval suite and declare victory — but a model's text performance says essentially nothing about whether it will correctly read the error message in a screenshot of your product. And a benchmark score tells you about the model baseline, not about how your pipeline is using it.

This chapter covers three layers of evaluation that a voice-agent engineer owns:

1. **Public multimodal benchmarks** — a baseline for model choice.
2. **In-domain pairwise human eval** — the judgement of whether this agent, with this prompt and this grounding scheme, is actually good on *your* traffic.
3. **Vision-grounding accuracy** — a programmatic check that the agent's claim used the image, not just rhymed with it.

Mod-112 covers the broader eval and observability harness; this chapter is the multimodal-specific pieces that plug into it.

## Public multimodal benchmarks: what they do and do not tell you

Three benchmarks are the current reference points for multimodal model evaluation. All three are **model-level** benchmarks — they tell you how a model performs on a standardised test, not how your agent built on that model performs on your traffic.

### MMMU — Massive Multi-discipline Multimodal Understanding

[MMMU](https://mmmu-benchmark.github.io/) ([arXiv:2311.16502](https://arxiv.org/abs/2311.16502)) is a college-level multimodal reasoning benchmark across six disciplines (Art, Business, Science, Health & Medicine, Humanities, Tech & Engineering). Problems contain images and text; the model has to do subject-matter reasoning. The follow-up [MMMU-Pro](https://arxiv.org/abs/2409.02813) is a harder, vision-only version.

Use MMMU to answer: "Among candidate vision-enabled LLMs, which has the broadest multimodal reasoning capability?" Do not use it to answer: "Will the model handle my specific support-screenshot use case?"

### MM-Vet — Multimodal "veterinary" eval

[MM-Vet](https://github.com/yuweihao/MM-Vet) ([arXiv:2308.02490](https://arxiv.org/abs/2308.02490)) evaluates six integrated multimodal capabilities (recognition, OCR, knowledge, language generation, spatial awareness, math). Unlike MMMU's subject tests, MM-Vet is deliberately about the *combinations* — can the model read the text in the image *and* do arithmetic on it *and* explain the result.

Useful for an agent that needs the *integration* rather than any single skill.

### LLaVA-OneVision benchmarks

[LLaVA-OneVision](https://arxiv.org/abs/2408.03326) is both a model family and a set of evaluations used to compare open multimodal models across single-image, multi-image, and video tasks. The benchmark results tables in the paper and on the project's repo are a reasonable snapshot of where the open ecosystem is; useful if you are considering a self-hosted model path.

### How to actually use benchmarks

- **Treat them as filter, not decision.** If a model scores in the bottom third of MMMU, it probably is not your choice. If it scores in the top third, you still have to run your own in-domain eval before shipping.
- **Watch for contamination.** Benchmarks age; leaderboards drift as providers tune to them. The number you care about is the delta against the *same model's previous release*, not against a competitor on a benchmark everyone has seen.
- **Benchmark on the same resolution / detail you will ship at.** A model's MMMU score at high resolution does not predict its behaviour on a 512 px crop.
- **Run the benchmark on your inference path**, not the vendor's. If you are self-hosting a quantised version, the quantised version is what you are shipping.

## In-domain human eval: the one that actually matters

Benchmarks tell you what the model *can* do. Pairwise human eval on your traffic tells you what your *agent* actually does. Three design decisions make or break this layer.

### Collect a representative multimodal set

A multimodal eval set is harder to assemble than a text-only one. You need:

- **Real user images / frames**, with consent. Synthetic or scraped images miss everything specific to your product: UI chrome, your branded forms, your customers' environments, your product's specific error states.
- **Coverage of the modality manifest (Chapter 1).** If your production traffic is 60% uploaded photos / 30% tool-returned artefacts / 10% screen share, the eval set should roughly match.
- **Inclusion of the hard cases.** Pre-filter your production traffic for turns where the agent was uncertain, the user followed up for clarification, or an operator overrode the agent. Those are the turns that move the needle.
- **Explicit "vision was irrelevant" cases.** Turns where the user attached an image out of habit but the question was answerable without it. The eval has to penalise an agent that pivots to describe the image instead of answering the question.

A reasonable starting-point set is 200–500 turns. Smaller is noisy; larger costs more per iteration.

### Pairwise, not pointwise

Rather than scoring one agent response on a Likert scale (reviewers calibrate against each other inconsistently), present **two agent responses side-by-side** and ask the reviewer which is better. Pairwise reviews are faster, more reliable, and give you a usable Elo-style ranking even across heterogeneous changes (new model, new prompt, new router).

The pair you compare depends on what you are evaluating:

- **Model swap**: same prompt + same router, different model.
- **Router change**: same model + same prompt, different routing policy (the pair can include an "unmatched" case where one side is text-only and one is vision-upgraded).
- **Grounding change**: same model + same router, different grounding instructions.

### Rubric, not just preference

A pure "which is better" question gives you a single number. For a multimodal agent, you want structured feedback:

- Did the agent **look at the right region** of the image?
- Did the agent's claim **match what the image actually shows**?
- If the agent **refused**, was the refusal appropriate for the image?
- Was the **voice response** speakable (plain prose, no markdown, no bare coordinates)?
- Was the **grounded evidence** citable (did the quoted text actually appear)?

Each of these is a separate rubric item. The reviewer marks each; the aggregate forms a scorecard that is more actionable than a single preference number.

### Who reviews

Internal operators or domain experts for high-stakes domains (medical, financial, legal). Crowd workers for broad-consumer products *with* a clear rubric and gold-standard calibration items. Never only engineers — engineering tastes diverge from user preferences in reliable, embarrassing ways.

## Vision-grounding accuracy: the programmatic check

Human eval is slow. For the specific question "did the model's claim actually use the image", you can measure continuously, without humans, by comparing claims against programmatic ground truth.

### The metric

For every turn where the agent emitted a grounded claim (Chapter 4) that included quoted text or a bounding box:

- **Quote accuracy** = fraction of quoted strings that appear verbatim (or with fuzzy match ≥ τ) in the OCR of the claimed region.
- **Region accuracy** = fraction of bounding boxes whose IoU with a reference region (from OCR output, from a separate detection model, or from a labelled dataset) exceeds a threshold.
- **Grounding coverage** = fraction of turns where at least one grounding record was emitted when one *should* have been (measured against the eval set's labels).

The three together summarise "did the model see what it claimed to see."

### How to collect it

The grounded-claims log from Chapter 4 is already structured to compute these metrics; wire a pipeline that:

1. Reads every grounded claim from the audit log.
2. Re-runs OCR on the referenced region (or pulls the OCR that was done at emission time).
3. Computes the per-claim metrics.
4. Aggregates to per-day / per-cohort / per-model rolling numbers.

Treat quote accuracy ≥ 95% as a working floor for a production agent; below that you have a hallucination problem that the human eval will also catch, less efficiently.

### Where this disagrees with the model's self-reported confidence

A model that is 95% confident on a claim that fails OCR verification is a model you should not trust. The gap between "model confidence" and "grounded accuracy" is itself a metric; monitor it per cohort and per model release.

## Regression testing a multimodal agent

Multimodal regressions are often subtle: a model update changes how frequently it refuses on borderline images, a prompt tweak changes the citation shape, a router change silently drops vision on 10% of the turns that used to get it. Three things to run on every release:

- **Benchmark baseline.** Run MMMU or MM-Vet subset on the exact inference path. A regression here is a model-level regression; raise with the inference team or the provider.
- **Pairwise eval on a frozen dataset.** 200 turns, pairwise against the last release. Quality should hold or improve.
- **Grounding-accuracy replay.** Replay the last week's grounded claims against the new agent, measure the three accuracy metrics, alert on regressions.

All three plug into mod-112's broader harness; keep them as named suites so they can be referenced from a release checklist.

## What the eval cannot tell you

A programmatic eval cannot tell you whether the agent was *useful*. Did the user actually resolve their problem? Did the agent's grounded answer help them act? Those are product-level outcomes — handoff rate, task completion, user satisfaction — that require product analytics on top of the eval. Mod-112 covers the broader observability story; this module's responsibility is to make sure the multimodal path is measurable at all.

## Summary

- Use **MMMU, MM-Vet, and LLaVA-OneVision** benchmarks as a **filter** on model choice. They are not a substitute for in-domain eval.
- **Pairwise human eval** on representative in-domain multimodal turns is the primary quality signal; design the rubric around grounding, refusal appropriateness, and voice-speakable output.
- **Grounding accuracy** is measurable programmatically via OCR cross-check, bounding-box IoU, and coverage metrics. Build the pipeline on the grounded-claims log from Chapter 4.
- **Regression-test** every release against a frozen benchmark subset, a frozen pairwise set, and a grounding-accuracy replay.
- The eval does not tell you whether the agent was useful — product-level outcome metrics live at the mod-112 layer.

Next chapter: the boundary to the applied-ai-engineer and nlp-engineer roles — what you own, what you escalate, and how to package the request.
