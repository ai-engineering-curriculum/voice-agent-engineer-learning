# 04. Content Eval — Judge-as-LLM on Transcripts, Task-Level Correctness, and the CI Gate

## Why this chapter

Chapters 1–3 produce three release-assurance packets: an ASR packet (WER, CER, MER, calibration, strata), a TTS packet (MOS, CMOS, SIG/BAK/OVRL, correlation), and a conversational packet (turn-taking, barge-in, TTFA, mouth-to-mouth, interrupt recovery). All three measure the *plumbing*. None of them answers the question that matters to the business: did the agent handle the call correctly?

That question lives at the content level — the semantic correctness of what the agent said and did. This chapter builds the content eval harness: judge-as-LLM graders on call transcripts, task-level correctness against gold scripts, deflection / escalation rate against the business baseline, and the CI wiring that turns every one of the four packets into a merge gate. By the end of the chapter, a pull request that regresses voice quality cannot land.

## Judge-as-LLM on call transcripts

The core pattern is **LLM-as-judge**: use a capable model, with a careful rubric, to grade the agent's output. For the voice stack, the grader runs on the **conversation transcript** (user turns + agent turns, as text produced by Chapter 1's ASR evaluated against the ground truth where you have it, or the model's own `instructed` text where you don't — see "transcript source" below). The paper to anchor against is [Zheng et al., "Judging LLM-as-a-Judge"](https://arxiv.org/abs/2306.05685); the practical patterns below reflect field experience with those judges on conversational data, with caveats about where they fail.

### What you grade

For a voice agent, the useful grader dimensions are:

- **Correctness.** Did the agent give the right answer, given the user's actual question? A gold-script match is one way to grade; a free-form rubric ("was the information factually correct, according to these policy snippets?") is another.
- **Groundedness.** Did every substantive claim the agent made trace to the retrieval context, the tool outputs, or the system prompt? Ungrounded claims are hallucinations, even when they happen to be correct.
- **PII leak.** Did the agent reveal personal information it should not have (full card numbers, SSNs, other-customer PII)? The judge is scanning a transcript for policy-violating disclosures; pair this with a deterministic regex layer as the first line of defence, because judges miss edge cases.
- **Policy compliance.** Did the agent follow the mandatory script pieces (call opening per mod-108 Chapter 3, disclaimer, closing)? Did it respect the escalation rules? Did it refuse out-of-scope requests appropriately?
- **Tone and persona fit.** Matches the brand's voice guidelines; refuses to be rude back; does not lecture. This is the softest grader dimension and the one most prone to disagreement — treat with the most caution.

Each dimension gets its own prompt and its own binary or Likert output. A *single* judge call that rates all five at once produces worse results than one call per dimension; the extra tokens are the cost of calibration.

### Grader-prompt structure

A judge prompt should be:

- **Task-specific, not generic.** "You are grading whether the agent gave the correct policy answer for a healthcare support call" grades better than "Rate this conversation 1–5."
- **Rubric-first.** Give the judge a numbered rubric with explicit anchors ("Score 1 if …; score 5 if …"). Rubrics reduce variance and are the single biggest knob for grader reliability.
- **Example-anchored.** Include 2–4 canonical examples per output score. Few-shot examples constrain the judge's distribution.
- **Chain-of-thought-permitted, but structured.** The judge thinks step-by-step in a scratchpad field, then emits a final JSON object with the score. Parsing a single JSON block out of a free-form rationale is the common bug; make the rationale a field in the JSON output or use structured-output / JSON-mode.
- **Order-and-position randomised** on A/B comparisons. Judges are biased toward the first option, the longer option, and the option with more structured formatting; see [Zheng et al.](https://arxiv.org/abs/2306.05685) for measurements. Randomise A/B order per call; truncate both sides to the same length; use the same formatting conventions.

### Transcript source

What transcript is the judge grading?

- **For offline eval**, use the *reference* transcript from your gold set as the user side (so the grader is not confused by ASR noise), paired with the *actual* agent audio transcribed by a second, independent ASR (so hallucinated TTS text does not go uncaught). Document which ASR you used for the agent side; cross-check with the agent's own log of what it was instructed to say.
- **For online eval** (Chapter 5), you do not have a reference transcript for the user side. Use the production ASR's output; accept that ASR errors are a confound; sample enough traffic that the signal rises above the noise. Flag transcripts where the ASR confidence is low and pull them to a human-review queue instead of a judge queue.

### Calibration — the quiet failure mode

A judge's score is only as good as its calibration against humans. Before you trust a judge-as-LLM in a gate, validate:

- Pick 200–500 transcripts with human labels (binary or ordinal).
- Run the judge.
- Report **agreement rate**, **Cohen's kappa**, **confusion matrix** per dimension, **per-stratum** agreement.
- Decide a floor (common published patterns: kappa ≥ 0.6, or agreement ≥ 85% on the dominant class) below which the judge is not trusted and the dimension goes to human review.

Re-validate on a cadence (quarterly, or whenever you change the judge model or prompt). A judge that was 0.7 kappa against humans on `gpt-4-turbo` is not necessarily 0.7 kappa on a replacement model.

### Known judge biases to measure, not ignore

- **Position bias.** Option A is favoured. Fix: randomise order.
- **Length bias.** The longer answer is favoured. Fix: truncate both sides or control for length in the prompt.
- **Self-preference.** A judge model prefers outputs from the same family. Fix: use a different vendor for the judge than for the agent, when possible.
- **Sycophancy.** Judges overrate agreeable, hedged output. Fix: rubric anchors + example-anchored prompts.
- **Formatting bias.** Judges prefer well-formatted output. Fix: strip or normalise formatting before judging.

All of these are measurable on your calibration set; measure them, publish them, and compensate for the ones that survive.

## Task-level correctness against gold scripts

Some calls are *scripted*: a diagnostic flow, a verification flow, an outbound reminder, a KYC walkthrough. For these you have a gold script — the canonical set of question-answer pairs the agent is supposed to produce. Task-level correctness is the pass/fail against the gold script:

- Encode the gold script as a **flow DAG** with required nodes and acceptable paths through them.
- Replay the call transcript against the DAG: did the agent hit every required node? Did it take an acceptable path? Did it collect the required pieces of information?
- Report **node coverage** per flow and **end-to-end flow pass rate**.

This is more deterministic than judge-as-LLM for the kinds of flows it covers, and it is cheaper to run. Use it for structured flows; use judge-as-LLM for the open-ended parts.

A warning: a gold script is a specification, and specifications go stale. When the flow changes in production (ops updates a policy, legal updates a disclosure), the gold script has to change *first*; otherwise the eval will fail on every new release for the right reason — the thing that changed — which is noise, not signal. Treat the gold script as code: version it, review PRs against it, release it on the same cadence as the agent.

## Deflection and escalation rate against the business baseline

Two business-facing content metrics anchor the system against its purpose:

- **Deflection rate.** The fraction of calls the agent resolved end-to-end without a human. "Resolved" needs an operational definition — a closed ticket with no reopens in 7 days is a common one, but it has to come from the business, not from the eval harness. Deflection against baseline is the top-line ROI number in most contact-centre AI deployments.
- **Escalation rate.** The fraction of calls routed to a human. Appropriate escalation is *good*; the metric you watch is whether the rate is drifting outside the band your operations team committed to serving.

Both numbers are *outcome* metrics, not quality metrics. They move for reasons other than model quality (product changes, promotion campaigns, seasonality). Do not treat them as the primary release gate. Do treat them as the business-facing scorecard the module reports.

A good practice: pair the deflection / escalation rate with a sample-reviewed **human audit** — a weekly human review of a stratified sample of escalated calls and a stratified sample of claimed-deflected calls. The audit catches "falsely resolved" calls (the ticket closed because the customer gave up, not because the agent helped) and "inappropriately escalated" calls (the agent could have handled it).

## Eval-gated CI — the merge gate

The eval harnesses from Chapters 1–3 and the content harness from this chapter all have the same job: produce a packet of numbers that compares the current commit's agent against the previous release's agent, and refuse to merge the PR if any of the numbers moves outside the acceptance band.

### Gate design

A reasonable gate configuration:

- **Hard gates — PR fails if any of these regresses**:
  - WER on the in-domain gold set, worst-bucket and overall, versus previous release.
  - UTMOS CI proxy on a fixed TTS eval set, versus previous release.
  - Mouth-to-mouth p95 on the conversational eval set, versus mod-109 budget *in absolute terms*.
  - Policy-compliance pass rate from judge-as-LLM on a fixed replay set, versus a policy floor.
  - Interrupt-recovery suite pass count on the labelled scenarios.
- **Soft gates — PR flags for review, does not block**:
  - Correctness / groundedness / tone judge scores, if the regression is within the judge's calibration noise band.
  - Deflection / escalation rate from the last 24 hours (noisy, slow, better suited to the Chapter 5 drift detector).
  - FLEURS / LibriSpeech / TED-LIUM 3 numbers (acceptable to be stable; a regression there is a signal but not necessarily a blocker if the in-domain number is fine).

Every gate's threshold has to be defensible: "we gate on WER regression > 0.3 percentage points because the gold-set 95% CI is ±0.15 points and we have decided we care about 2× the noise band." A threshold without that math is not reproducible.

### CI mechanics

Two shapes work:

- **Replay gate.** On every PR, run the eval harness against a fixed replay set of audio inputs through the candidate agent build. Collect the metrics. Compare to a baseline manifest. Fail or pass. This is the standard pattern; it requires that your pipeline be runnable headless against pre-recorded audio.
- **Shadow gate.** For infrastructure changes (not model/prompt changes), compare against production traffic in shadow mode — the candidate build receives real traffic in parallel with the current build; metrics are compared but no responses are delivered. Shadow gates give realistic distributions at the cost of needing a shadow lane in your deployment.

Either way, the gate runs on CI and the merge is blocked by the CI status. Write the gate as code and check it into the repo so it's reviewable.

### Baseline manifest and drift

The baseline that CI compares against is itself a versioned artefact — the manifest of "this commit's eval numbers". When a PR legitimately changes behaviour (new feature, prompt rewrite, voice change), update the baseline as part of the PR; require a reviewer to approve the baseline change; audit baseline changes separately from code changes.

Without this discipline the baseline drifts upward slowly and the gate becomes "better than it was last Thursday", which is not the same as "meets the product bar".

### Failure modes to watch

- **Flakiness.** LLM / ASR / TTS calls into vendor APIs fail intermittently. Make the gate retry a bounded number of times and treat vendor errors as separate from eval regressions. A repeated vendor error means the gate's dependencies are broken; the gate should fail differently than a quality regression.
- **Cost.** Running a full eval on every PR gets expensive. Common pattern: a small eval on every PR (hundreds of samples); a full eval nightly or on `main`. The small eval can miss regressions the full eval catches; expect occasional post-merge rollbacks and design for them.
- **Budget creep.** "Gate's too strict" is a common complaint. Resolve by moving metrics between hard and soft gates, not by relaxing thresholds. If a threshold is wrong, change it with the math, not by waving it away.

## A reference pipeline

```python
# Illustrative. Read your current provider SDKs for the actual judge-call syntax.
from pydantic import BaseModel

class JudgeOutput(BaseModel):
    rationale: str
    score: int  # 1..5
    violations: list[str]  # for policy / PII graders

JUDGE_PROMPT_POLICY = """
You are evaluating whether a voice agent followed the required call-opening policy.

Required elements:
- Identifies itself as an AI
- States the company name
- Gives the recording disclosure if the state requires it (list: {two_party_states})

Rubric:
- Score 5 if all required elements were present.
- Score 3 if one element was missing but the rest were present.
- Score 1 if more than one element was missing, or if the agent made a materially
  misleading statement about its identity.

Output JSON: { "rationale": str, "score": int, "violations": [str] }
"""

def grade_policy(transcript, state):
    resp = llm_json(
        model="gpt-4o-2024-08-06",  # or current judge model
        prompt=JUDGE_PROMPT_POLICY.format(two_party_states=", ".join(STATES_TWO_PARTY)),
        content=transcript,
        schema=JudgeOutput,
    )
    return resp

def grade_pii(transcript):
    # deterministic regex + judge-as-LLM second pass
    hits = PII_REGEX.findall(transcript)
    judge = llm_json(
        model="gpt-4o-2024-08-06",
        prompt=JUDGE_PROMPT_PII,
        content=transcript,
        schema=JudgeOutput,
    )
    return {"regex_hits": hits, "judge": judge}

def eval_release_assurance(agent_build, eval_set):
    rows = []
    for call in eval_set:
        result = agent_build.replay(call.audio_in)
        rows.append({
            "call_id": call.id,
            "wer": wer_vs_gold(result.user_transcript, call.gold_user_transcript),
            "utmos": utmos(result.agent_audio),
            "mouth_to_mouth_p95": result.turn_metrics.mouth_to_mouth_p95,
            "policy": grade_policy(result.transcript, call.state),
            "groundedness": grade_groundedness(result.transcript, call.retrieval),
            "pii": grade_pii(result.transcript),
            "task_pass": task_dag_matches(result.transcript, call.gold_flow),
        })
    return rows

def ci_gate(current_rows, baseline_manifest):
    deltas = compare(current_rows, baseline_manifest)
    hard_fails = [d for d in deltas if d.is_hard and d.regressed]
    if hard_fails:
        return {"pass": False, "reasons": hard_fails}
    return {"pass": True, "deltas": deltas}
```

Pipe the CI gate's output into the PR status check; block merge on failure; annotate the PR with the specific regressions so the author does not have to hunt.

## What belongs in the release-assurance packet

The content packet complements Chapters 1–3 with:

- **Judge-as-LLM scores** on correctness, groundedness, PII leak, policy compliance, and tone — per stratum, with CIs.
- **Judge calibration report** — latest human-vs-judge kappa per dimension, versioned with the judge prompt and model.
- **Task-level correctness** — flow pass rate per scripted flow, node coverage, and the version of the gold script used.
- **Deflection and escalation rate** vs. baseline — flagged, not gated.
- **CI gate configuration** — which metrics are hard, which are soft, what the thresholds are, and when they were last updated.

With Chapters 1–3's packets, this is the full offline eval story. Chapter 5 extends it to online traffic and drift; Chapter 6 wires all of it into the observability plane.

## Summary

- **Judge-as-LLM** grades correctness, groundedness, PII leakage, policy compliance, and tone — one dimension per call, rubric-first, with structured output.
- **Calibrate judges against humans** periodically; publish kappa and agreement per dimension; refuse to gate on an uncalibrated judge.
- Measure known **judge biases** (position, length, self-preference, sycophancy, formatting) on your calibration set; compensate in the prompt or in the data pipeline.
- **Task-level correctness** against a versioned gold flow DAG is the deterministic complement for scripted flows.
- **Deflection / escalation rate** are outcome metrics against a business baseline — report them, don't primarily gate on them.
- The **CI gate** is the merge-blocking enforcement of all four packets (ASR, TTS, conversational, content); hard gates on the in-domain / product-target metrics, soft gates on the public-corpus and secondary metrics.
- **Baseline manifests** are versioned artefacts; a PR that changes behaviour updates the baseline through a reviewed change, not silently.

Next chapter: online eval on production traffic — judge-as-LLM at sample rate, implicit user-feedback signals, drift detection on WER / MOS / latency, and the regression-fixture back-feed from every incident.
