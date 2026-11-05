# exercise-06: Multimodal Benchmark Walkthrough — MMMU, MM-Vet, Grounding Accuracy

**Estimated effort:** 1 hour · **Output:** a benchmark-run writeup + a grounding-accuracy replay pipeline

## Objective

Run a representative multimodal benchmark subset against the model(s) you selected in exercise-02, build the **grounding-accuracy replay** pipeline that consumes the audit log from exercise-04, and interpret both. The deliverable is a short note that argues — from numbers — which model you would default to in your scenario, and what the eval is and is not telling you.

This is the eval half of the module; mod-112 builds on this with the broader observability harness.

## Prerequisites

- [Chapter 6 — Evaluating Multimodal Turns](../06-evaluating-multimodal-turns.md).
- The `VisionClient` from exercise-02 and the grounded-claims audit log from exercise-04.
- Access to the benchmarks: [MMMU](https://mmmu-benchmark.github.io/) ([Hugging Face dataset](https://huggingface.co/datasets/MMMU/MMMU)) and [MM-Vet](https://github.com/yuweihao/MM-Vet) (or [MM-Vet v2](https://github.com/yuweihao/MM-Vet-v2)). Read each benchmark's license before using it.

## Steps

1. **Pick a benchmark subset.** You are not trying to publish a leaderboard entry; you are learning the eval. A reasonable subset:
   - **MMMU (validation split)** — sample 50–100 items across at least three disciplines.
   - **MM-Vet** — the test set is ~200 items; run the whole thing or a 100-item stratified subset.
   Document the sampling methodology.

2. **Run the benchmark against at least two models** from exercise-02 (e.g., GPT-4o and Claude; add Gemini for a three-way). For each item, record the model's answer, latency, and token usage.

3. **Grade the responses.**
   - MMMU: follow the benchmark's official grading methodology — typically an LLM-as-judge or a rule-based match against the gold answer. Document which you used.
   - MM-Vet: follow the project's recommended judging prompt (it uses GPT-4 as an evaluator by default). Alternative judges (Claude, Gemini) can be used; log which judge you used and treat scores from different judges as not directly comparable.

4. **Report model deltas.** For each benchmark and each model, report:
   - Overall score.
   - A per-discipline / per-capability breakdown (MMMU's discipline axis, MM-Vet's capability axis).
   - Latency per call (p50 / p95).
   - Token cost per call.
   Where you ran the same items on multiple models, tabulate the delta.

5. **Build the grounding-accuracy replay pipeline.** Using the audit log from exercise-04:
   - Read every logged grounded claim.
   - Re-run the OCR check (using the stored image hash + a server-side cached image, or re-fetching from your storage).
   - Compute `quote_accuracy`, `region_accuracy` (if you have a reference set of regions; otherwise skip), and `grounding_coverage`.
   - Aggregate over the whole dataset.
   This is a replay, not a new inference; it reads cached audit records and produces metrics.

6. **Design your in-domain pairwise rubric.** You will not run a full human eval in this exercise, but you should write the artefacts for one:
   - A rubric covering: looked-at-right-region, claim-matches-image, refusal-appropriateness, speakable-voice-output, citable-grounded-evidence.
   - A representative sample of 20 (image, question) pairs drawn from your exercise-02 and exercise-04 test sets.
   - A short reviewer-instruction document.

7. **Write a short note.** 1–2 pages. Cover:
   - The benchmark deltas and which model you would default to for your exercise-01 scenario, with reasoning.
   - The grounding-accuracy numbers from your replay and what they imply about your current prompt / verification.
   - The things the benchmarks did not tell you — specifically, what about your scenario you would still need in-domain pairwise eval to confirm.
   - Which regressions you would gate releases on in mod-112's harness.

## Starter guidance

- **Read the benchmark's rules.** MMMU specifies the answer format and the matching rule; deviations give you a score that does not match published numbers. Follow the methodology exactly for the subset you run.
- **Dates matter.** Your benchmark score is only comparable to other scores of the same benchmark version, run on the same model version, with the same judging prompt. Record all three.
- **Judges can be biased.** An LLM-as-judge with the same model family as the evaluated model tends to score that family higher. Prefer a judge model from a different provider than the one you are grading, and cross-check on a subset with a different judge.
- **The benchmark subset is a filter.** If one of your candidate models does badly on the benchmarks, skip the full in-domain eval for that model.
- **Grounding replay is cheap compared to re-inference.** Leave it running on the audit log on a schedule; it will catch regressions in grounding without any new user traffic.

## Acceptance criteria

- [ ] MMMU subset (≥50 items) + MM-Vet (≥100 items) run against at least two models.
- [ ] Grading methodology documented (official judge, version, prompt).
- [ ] Reports: overall score, per-axis breakdown, latency, cost per model.
- [ ] Grounding-accuracy replay pipeline consuming the exercise-04 audit log and producing quote-accuracy and grounding-coverage metrics.
- [ ] In-domain pairwise rubric with 5 items + a 20-pair sample + a reviewer-instruction document (artefacts only; the eval itself is optional for this exercise).
- [ ] A 1–2-page note recommending a default model and listing the release-gating regressions.

## Stretch goals

- **LLaVA-OneVision benchmarks** in parallel, especially if you are considering a self-hosted model. Compare open-weight results against the hosted providers.
- **Run the in-domain pairwise eval.** Recruit three reviewers, run the 20-pair sample, measure inter-rater agreement, and report a preference ranking. This is the real eval the acceptance-criteria version only sketches.
- **Benchmark contamination check.** Pick ten items from MMMU, ask the model to answer them *without* the image. If it gets more than a couple right, the benchmark is partially contaminated for that model. Report what you found.
- **Automated regression suite.** Wire the benchmark + grounding-accuracy replay into CI (GitHub Actions or equivalent). On each release of your agent, run the subset and fail the build on regression beyond a tolerance. Mod-112 will formalise this; a starter version here is useful.

## Keep it

This eval harness is the plug into mod-112. The benchmark subset, rubric, and grounding-accuracy replay are the three artefacts that make the next module's broader eval story concrete. Keep them in a form that can be run from CI without human intervention.
