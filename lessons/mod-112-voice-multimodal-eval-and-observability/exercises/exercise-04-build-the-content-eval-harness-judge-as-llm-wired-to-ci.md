# exercise-04: Build the Content Eval Harness and the CI Merge Gate

**Estimated effort:** 2 hours · **Output:** calibrated judge-as-LLM graders, a gold-script DAG for a scripted flow, a CI job that merges-or-blocks a PR on regressions, a content release-assurance packet

## Objective

Build the judge-as-LLM graders for correctness / groundedness / PII / policy / tone; write a gold-script flow DAG for one of your product's scripted journeys; and wire the harnesses from exercises 01–03 plus this one into a single CI merge gate that blocks a PR when the agent's voice quality regresses.

## Prerequisites

- [Chapter 4 — Content Eval and the CI Gate](../04-content-eval-judge-as-llm-and-ci.md).
- Exercises 01 (ASR), 02 (TTS), and 03 (conversational) completed; their packets are the inputs this exercise wires together.
- A capable judge model accessible via API (e.g., `claude-sonnet-4-6`, `gpt-4o-2024-08-06`, or the current house judge — follow the ai-eval-engineer boundary from Chapter 7). Use a different vendor than the agent's LLM to limit self-preference bias.
- A small set of human-labelled transcripts (≥100, double-annotated) for judge calibration — same vendor pipeline as the gold-set labels from exercise-01.
- A CI system you can wire a status check into (GitHub Actions, GitLab CI, Buildkite, or similar).

## Steps

1. **Author grader prompts.** One prompt per dimension (correctness, groundedness, PII leak, policy compliance, tone). Each prompt: task-specific framing, numbered rubric with explicit anchors, 2–4 example pairs, structured JSON output with `rationale`, `score`, `violations`. Keep dimensions separate — do not multi-head one prompt.

2. **Calibrate against humans.** Run each judge on the labelled set. Report agreement rate, Cohen's kappa, and a confusion matrix per dimension. Measure the four big biases on the calibration set: position, length, self-preference, formatting. Set a floor (e.g., kappa ≥ 0.6) below which the dimension is not CI-gated.

3. **Gold-script flow DAG.** Pick one scripted flow in your product (KYC check, order-taking, triage, outbound reminder). Encode it as a DAG with required nodes and acceptable paths. Build a replay checker that walks a transcript against the DAG and emits node-coverage + end-to-end flow pass.

4. **Deterministic PII regex layer.** Build a regex-based PII scanner (card numbers, SSNs, phone numbers, emails) that runs *before* the PII judge and emits counts. Regex misses are a floor; the judge catches semantic leaks.

5. **Baseline manifest.** Produce the current state's packet for all four harnesses (ASR, TTS, conversational, content). Commit it as the baseline. Any PR that regresses beyond threshold from this baseline fails the gate.

6. **Define the gate configuration.** A config file with:
   - **Hard gates** (block merge): in-domain WER worst-bucket regression > X; UTMOS CI proxy regression > Y; mouth-to-mouth p95 > budget; policy-compliance score < floor; interrupt-recovery suite missing any scenario pass.
   - **Soft gates** (flag for review): judge-score drift within the kappa noise band, FLEURS / LibriSpeech / TED-LIUM 3 regressions, deflection rate drift.
   Each threshold has a comment with the math that chose it.

7. **CI job.** Write the CI config that: (a) checks out the PR build, (b) runs the four harnesses against the fixed replay set, (c) compares results against the baseline, (d) sets the PR status to pass/fail with a comment listing specific regressions, and (e) uploads the new packet as an artefact.

8. **Prove it blocks a bad PR.** Introduce a deliberate regression (e.g., a prompt change that drops the opening policy disclosure). Run the CI job. Confirm it fails with a clear explanation. Revert the regression; confirm the gate re-passes.

9. **Emit the content release-assurance packet.** Judge calibration report (per dimension + bias measurements), scores per dimension stratified, task-level correctness for the flow DAG, deflection + escalation rate against baseline, CI gate configuration with threshold math.

## Starter guidance

- **One rubric per dimension.** Resist the temptation to multi-head into one judge call; the loss in calibration swamps the token savings.
- **Structured output is non-negotiable.** Use the vendor's JSON mode / structured output / tool-use calling. Free-form rationales that you regex out are a source of flaky evals.
- **Pick a judge model that isn't your agent's model.** Judge self-preference is the easiest bias to avoid by choice of model.
- **Calibration data lives with the ai-eval-engineer pipeline.** Reuse the labelling infra (Chapter 7); do not re-invent it in the voice stack.
- **Baseline updates are reviewed changes.** When a PR legitimately changes behaviour, the baseline update is part of the PR; require explicit reviewer sign-off on baseline changes. Audit baseline edits separately from code.
- **CI cost control.** Full eval on every PR gets expensive quickly. Pattern: a small eval (hundreds of samples) on every PR; a full eval nightly on `main`. Document the gap.

## Acceptance criteria

- [ ] Five grader prompts (correctness, groundedness, PII, policy, tone) with rubric anchors, examples, and structured JSON output.
- [ ] Judge calibration report: kappa / agreement / confusion matrix per dimension on a ≥100-transcript labelled set, plus the four bias measurements.
- [ ] A gold-script flow DAG for one scripted flow, with a replay checker producing node coverage and end-to-end pass.
- [ ] A deterministic PII regex layer wired in front of the PII judge.
- [ ] A committed baseline manifest covering all four harnesses.
- [ ] A gate configuration file with hard/soft thresholds and the math that chose each.
- [ ] A CI job that runs on every PR, blocks on hard gates, flags on soft gates, and comments the PR with specifics.
- [ ] A reproducible demo of a deliberate regression blocked by the gate, then re-passed after revert.
- [ ] A machine-readable `content_packet.yaml` / `content_packet.json`.

## Stretch goals

- **Judge ensembles.** Run two different judge models on the same transcripts; disagreement is a signal to escalate to human review.
- **Shadow gate.** Instead of (or alongside) replay, run the candidate build against production traffic in shadow mode and compare metrics vs. control. Requires a shadow lane in deployment.
- **Flow-specific CMOS on the judge.** For your most-sensitive dimension (e.g., policy compliance in a regulated vertical), add a human-review sampling rate — a weekly audit of judge-flagged transcripts to maintain calibration.
- **Multi-flow gold DAGs.** Encode a second scripted flow and verify the harness scales.
- **Evidence-attribute wiring.** For every judge output, attach a `voice.content.*` OTel attribute on the turn span (Chapter 6) so the trace reflects the judge verdict.

## Keep it

The CI gate is the enforcement surface for the whole module. Every subsequent model / prompt / vendor change in the voice stack runs through it. Exercises 05 (online eval) and 06 (OTel instrumentation) extend the same baseline into production traffic.
