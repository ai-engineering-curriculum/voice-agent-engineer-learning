# exercise-05: Ship Online Eval — Sampled Judges, Implicit Feedback, Drift Detection, Incident Back-Feed

**Estimated effort:** 2 hours · **Output:** a sampled online-eval pipeline, an implicit-feedback dashboard, drift detectors for key metrics, a documented incident-to-fixture procedure

## Objective

Extend the offline eval surface from exercises 01–04 into production traffic. Sample calls for judge-as-LLM scoring, pipe implicit user-feedback signals as metrics, wire drift detectors on WER / MOS proxy / mouth-to-mouth / policy score, and operationalise the regression-fixture back-feed so that every incident leaves the eval set stronger than it found it.

## Prerequisites

- [Chapter 5 — Online Eval, Feedback, and Drift](../05-online-eval-feedback-and-drift.md).
- Exercises 01–04 completed; the judges, harnesses, and gate baseline are reusable here.
- Observability plane available (OTel backend from exercise-06 if that is done first, or a provisional metrics store). Either order works; the two exercises are complementary.
- Legal / DPO sign-off on using production transcripts for online eval under your current consent record (mod-108 Chapter 2 / Chapter 4). If this is not already covered, document the blocker and run the exercise against a replayed-production dataset instead.

## Steps

1. **Stratified sampling policy.** Define the base sample rate (1–5% suggested), the trigger rules (low ASR confidence; escalations; mentions of "human"/"representative"; SLO violations; new flows / segments), and the stratification axes that drive oversampling. Encode as a config file.

2. **Sampler implementation.** A service (or a periodic job) that: reads the OTel call trace, decides sample vs. skip per rule, writes the sampled call manifest to a queue. Attach `sample_probability` to each sampled call so you can weight on aggregation.

3. **Online judge runner.** Consume the queue, pull the transcript (from your trace store or your call recordings), run the exercise-04 judges, write scores to the metrics store with the call's standard labels (`voice.call.id`, `voice.turn.id`, strata, flow).

4. **Implicit feedback instrumentation.** Pipe the following signals as OTel metrics / events:
   - Call abandonment by phase (opening / discovery / resolution / closing).
   - Repeated-question detection (dedup user turns within a short window using text-similarity).
   - User-silence distribution post-agent turn.
   - Explicit-escalation-to-human rate.
   - Post-call CSAT (if a survey exists).
   Each metric carries the standard call / turn labels.

5. **Build the drift detectors.**
   - **Univariate**: rolling-window delta (last N hours vs. last 7N hours) with bootstrap CI; add CUSUM / EWMA for the key SLO metrics (WER, mouth-to-mouth p95, policy score).
   - **Input distribution**: KS or Jensen–Shannon on the acoustic-condition bucket histogram, call-length histogram, codec distribution, topic-cluster histogram. Reference tools: [Evidently](https://github.com/evidentlyai/evidently) or [Alibi Detect](https://github.com/SeldonIO/alibi-detect) for the stats plumbing.
   Alert routing: high-business-impact flow → on-call page; others → weekly review digest. Correlate signals before paging.

6. **Dashboards.** At least two:
   - **Per-flow quality dashboard.** For each named flow, show: p95 mouth-to-mouth, deflection, judge scores for each dimension, implicit-feedback rates, trend arrows.
   - **Drift status board.** Current detector state per metric, per stratum, with alert status and run-book links.

7. **Incident-to-fixture procedure.** Write a one-page runbook that defines:
   - How an incident is captured (trace ID, call recording, trace export).
   - How to add the capture to `regressions/` in the eval set, with expected behaviour.
   - Review + sign-off on the fixture (voice-agent-engineer + ops owner).
   - How the fixture gets removed (documented rationale; never silent).
   Run one fixture through the procedure as a dry-run from a replayed incident.

8. **Rollout wiring.** Define the canary / regional-canary / shadow modes available to you and the metric thresholds that automatically roll back. The thresholds reuse the exercise-04 hard gates. Document how an operator pushes a rollback manually.

## Starter guidance

- **Sample weights are the single most-forgotten detail.** The top-line "online judge score" has to be weighted by `1 / sample_probability`, or the number lies about traffic composition. Bake weights into the aggregation at the point of collection.
- **Implicit feedback as *signal*, not label.** Every one of the implicit signals has a legitimate non-quality explanation. Monitor correlated moves; do not train on implicit feedback as if it were ground truth.
- **Drift on 5-minute windows pages too often.** Rolling-window + CUSUM over hours/days gives you alerts that are mostly real. Tune the window to the metric's natural variance.
- **Low-confidence ASR → human review, not judge.** Judges on noisy transcripts produce noisy judgements. Route the hard tail to a human-reviewer queue.
- **Privacy audit for the queues.** Everywhere a transcript sits — the judge queue, the review queue, the dashboards — is in scope for your mod-108 retention clock.
- **A silent eval regression is a worse failure than a loud one.** Design the drift detector to err toward false positives early; relax thresholds only after you have real baselines.

## Acceptance criteria

- [ ] A sampling-policy config with base rate + trigger rules + stratification, implemented in a sampler service.
- [ ] An online judge runner that scores sampled transcripts and writes to the metrics store with standard labels.
- [ ] At least four implicit-feedback signals instrumented as metrics with call/turn labels.
- [ ] Drift detectors (rolling window, CUSUM or EWMA, distribution drift) running on at least WER / MOS proxy / mouth-to-mouth p95 / policy score / one input distribution.
- [ ] Two dashboards: per-flow quality and drift status, each read-only linkable.
- [ ] A one-page incident-to-fixture runbook with one dry-run fixture added through it.
- [ ] A rollout configuration showing shadow / canary / regional-canary modes and the auto-rollback thresholds.
- [ ] Legal / DPO sign-off noted (or the blocker documented and replayed-production used instead).

## Stretch goals

- **Automated human-review queue.** Build the review tool that surfaces judge-flagged transcripts to an internal reviewer with a one-click "confirm / override" that back-feeds calibration data.
- **Drift explanation.** When a drift alert fires, auto-run a slice-finder (most-affected stratum) and attach to the alert. Turns a page into actionable information.
- **Cohort analysis.** Group users into cohorts (new / repeat / escalated-prior) and compare quality across cohorts. Often surfaces segmentation-related issues.
- **Replay from fixtures nightly.** Nightly job replays the `regressions/` set through the live build and emits a dashboard panel. Catches drift between what the PR gate measured and what production is doing.
- **Second-judge disagreement loop.** Any high-stakes transcript gets scored by two judges; disagreement is surfaced on the dashboard and resolved by human review.

## Keep it

This exercise is the beating heart of the eval surface. Every subsequent incident is a fixture addition; every subsequent release is a new baseline; every subsequent model-version bump is caught here. Exercise-06 wires the telemetry pipe; this exercise is its downstream consumer.
