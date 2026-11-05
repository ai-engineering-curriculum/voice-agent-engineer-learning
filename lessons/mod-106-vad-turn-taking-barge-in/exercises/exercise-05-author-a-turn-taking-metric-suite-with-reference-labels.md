# exercise-05: Author a Turn-Taking Metric Suite with Reference Labels

**Estimated effort:** 3 hours · **Output:** a labelled multi-condition corpus, a scoring script, a dashboard of the five metrics, and a regression-harness contract ready to hand to Module 112.

## Objective

Operationalise Chapter 5. Build the reference-label corpus, the scoring script that computes the five metrics plus stratifications, and the replay harness that runs your Exercise 1–4 pipeline against the corpus and emits a dashboard. By the end you should be able to answer "what is our p95 interrupt reaction time on noisy telephony audio with mid-utterance pauses?" from a script and a corpus, not from vibes.

This is the exercise that turns the module's engineering into a continuous evaluation surface — the thing Module 112 inherits as the voice-agent-specific slice of its harness.

## Prerequisites

- [Chapter 5 — The Turn-Taking Metric Suite](../05-turn-taking-metric-suite.md).
- Exercises 1–4 completed. You have:
  - `StreamingVAD`.
  - EOU variable-hangover commit loop.
  - Barge-in cancel + interruptible player.
  - Backchannel classifier.
  - Multiple small corpora (clean / noisy VAD, EOU, barge-in, backchannel).
- Python 3.10+; `pandas` or equivalent for the stratified aggregation.
- A labelling tool: [Label Studio](https://labelstud.io/) or [Praat](https://www.fon.hum.uva.nl/praat/), or any JSONL-based manual workflow.

## Requirements

### Part A — the reference corpus

Combine and extend your prior corpora into a single reference set. **Target: ≥ 60 min of audio, ≥ 150 labelled turn boundaries, ≥ 30 labelled barge-in events, ≥ 30 labelled backchannel events.**

Stratification:

- **By noise condition:** clean (> 20 dB SNR), moderate (10–15 dB), noisy (0–5 dB).
- **By device class:** headset / speakerphone / telephony (simulated via μ-law 8 kHz if needed).
- **By utterance length:** short (< 1 s), medium (1–5 s), long (> 5 s).
- **By mid-utterance pause:** present / absent.
- **By language:** primary + at least one secondary (if the deployment is multilingual; otherwise skip).

Each clip's labels:

```jsonc
{
  "clip_id": "clip-0042",
  "audio_path": "audio/clip-0042.wav",
  "sample_rate": 16000,
  "duration_ms": 15420,
  "stratification": {
    "noise": "moderate",
    "device": "speakerphone",
    "language": "en-US"
  },
  "turn_boundaries_ms": [3200, 9800, 14900],    // ground-truth turn ends
  "barge_in_events": [                            // user barges in on bot audio
    {"user_onset_ms": 5100, "transcript": "wait, no"}
  ],
  "backchannel_events": [                         // user emits backchannel during bot audio
    {"user_onset_ms": 7400, "duration_ms": 350, "transcript": "mm-hm"}
  ],
  "has_mid_utterance_pause": true,
  "utterance_length_class": "medium"
}
```

Document the labelling policy in a `CORPUS.md`: how is a "turn boundary" defined, how is a "barge-in" distinguished from a "backchannel," what tolerance window governs the matching.

Two labellers on at least 20 % of the corpus (overlap set); report Cohen's κ on boundary presence within a 300 ms tolerance. Target κ ≥ 0.75.

### Part B — the scoring script

Build `score.py` that takes:

- A labels JSONL (from Part A).
- A pipeline-events log (what your pipeline emitted during the corpus replay).

… and emits the five metrics plus stratifications:

```python
metrics = compute_metrics(labels_jsonl, events_log)
# metrics = {
#   "overall": {
#     "turn_correctness": 0.91,
#     "ftgr_per_min": 0.24,
#     "missed_eou_rate": 0.011,
#     "response_gap_ms": {"p50": 420, "p95": 1180},
#     "interrupt_reaction_ms": {"p50": 140, "p95": 210},
#   },
#   "by_noise": {...},
#   "by_device": {...},
#   "by_utterance_length": {...},
#   "by_pause_presence": {...},
#   "by_language": {...},
# }
```

Implementation notes:

- Match commits to ground-truth turn boundaries greedily within τ = 300 ms. Count non-matched commits as false commits; count non-matched boundaries as misses.
- Normalise FTGR per minute of **user speech**, not per minute of audio. (Noise-only minutes shouldn't be in the denominator.)
- Use `MissedEOU` with τ_miss = 1500 ms as the "has the pipeline given up" threshold.
- For `response_gap_ms` and `interrupt_reaction_ms`, require 30+ events per bucket before reporting; otherwise mark as `n/a`.
- Report percentiles using linear interpolation; document the method.

### Part C — the replay harness

Build `replay.py` that:

1. Reads a clip from the corpus.
2. Instantiates your pipeline (VAD + EOU + barge-in + player) in a test-harness mode where audio flows from a file source at realtime rate.
3. Captures every pipeline event: VAD state transitions, commits, cancels, bot audio start / stop, classifier outputs.
4. Writes an events log compatible with the scoring script.
5. Supports a `--config` argument for A/B — swap VAD threshold, swap EOU variant, swap classifier variant.

Run the harness against the full corpus for at least three configs:

- **Baseline.** Silence-only endpointing, no backchannel classifier, vanilla barge-in.
- **EOU-enhanced.** Chapter 2 variable-hangover from Exercise 2.
- **Full stack.** EOU + Chapter 4 backchannel classifier + Chapter 3 interruptible player.

### Part D — the dashboard

Emit a Markdown table that any reviewer can read:

```markdown
| Metric | Baseline | EOU | Full | Target |
|---|---|---|---|---|
| Turn correctness | 82.4 % | 89.1 % | 90.7 % | > 90 % |
| FTGR / min | 0.74 | 0.31 | 0.22 | < 0.3 |
| Missed-EOU | 0.9 % | 1.4 % | 1.2 % | < 1 % |
| Response gap p50 / p95 (ms) | 380 / 1100 | 450 / 1340 | 440 / 1300 | p95 < 1200 |
| Interrupt reaction p50 / p95 (ms) | 190 / 320 | 190 / 320 | 180 / 210 | p95 < 200 |
```

Plus stratified sub-tables (by noise, by device, by language). Note any bucket where n < 30 events.

### Part E — Module 112 handoff contract

Write a short `HANDOFF.md` describing:

- The labels JSONL schema (Part A), with versioning.
- The events-log schema the pipeline emits (Part C).
- The scoring-script API (Part B), as a function signature.
- The alerting thresholds you would set (e.g., "alert if FTGR drifts by 2σ from 14-day baseline").
- The regression-harness contract: "every PR touching VAD / EOU / barge-in runs `replay.py` on the corpus and blocks if any metric regresses > 10 %."

This file is the interface Module 112's harness will consume.

### Part F — write-up

Answer:

- Which config landed closest to the target for each metric? Where did they still miss?
- Which stratification revealed the biggest hidden defect — noise, device, language?
- Did your full-stack config buy improvements in some metrics at the cost of others? (FTGR vs. missed-EOU is the classic trade-off.)
- If the corpus were twice as large, which metric's confidence interval would tighten most? (Equivalent: where did small-n undermine the current eval?)
- One listening pass of five random clips from the worst-decile p95 reaction time: what did you hear that the dashboard didn't catch?

## Starter guidance

- **Labels are expensive. Prioritise.** If you can only label 30 min of audio, pick the stratifications carefully. Clean + noisy headset is the first cut; add speakerphone and telephony on the next pass.
- **Treat τ (correctness tolerance) and τ_miss as dials.** Vary them on a secondary pass and see how the metrics move; the right τ is the one where your eval differentiates good from bad behaviour.
- **Realtime replay.** Not faster. Partial-ASR timing changes at non-realtime rates and the metrics will be wrong.
- **Deterministic pipeline runs.** `temperature=0` on the LLM, seed any random steps, so run-to-run variance is small.
- **Log by event, not by aggregate.** The scoring script re-aggregates. The pipeline's events log should be the raw stream of state changes with timestamps.
- **Invert the harness during development.** Hand-write a tiny events log and a tiny labels file; verify the scoring script computes what you expect; then plug the pipeline in.

## Acceptance criteria

- [ ] Reference corpus with the structure and stratifications from Part A; `CORPUS.md` with the labelling policy; Cohen's κ ≥ 0.75 on the overlap set.
- [ ] `score.py` producing the metrics object with the overall + five stratifications.
- [ ] `replay.py` running the pipeline against the corpus at realtime rate for at least three configurations.
- [ ] A Markdown dashboard (Part D) with overall + stratified tables, Target column, and `n/a` where buckets are too small.
- [ ] `HANDOFF.md` describing the labels schema, events-log schema, scoring API, alerting thresholds, and regression-harness contract.
- [ ] One-page write-up answering the five questions above.
- [ ] Listening-pass notes on five worst-decile clips.

## Stretch goals

- **Confidence intervals.** Compute bootstrap CIs on each metric. Note where buckets with n < 30 produce useless intervals.
- **Alerting simulation.** Generate a synthetic 30-day trend of metrics (e.g., add random drift). Implement a 2σ alert rule; verify it triggers on real drift and does not trigger on noise.
- **Replay perturbation tests.** Inject known failure modes (artificially raise the VAD threshold, artificially increase hangover) and confirm the metric suite catches them.
- **Module 112 integration.** If you have access to the Module 112 harness skeleton, plug the scoring script in behind the handoff interface. Verify the pipeline runs through the shared harness unchanged.
- **Cost model.** Add a sixth column to the dashboard: per-session cost (in cents) for each config. This is the number product will ask for.

## Non-goals

- Shipping a continuous eval pipeline. You produce the contract and the artefacts; Module 112 ships the running pipeline.
- Replacing Module 112. This exercise is the voice-agent-specific slice of it.
- Reaching every target on first run. Measurement first; optimisation after. If your Full stack is below target on p95 reaction time, name it — Module 109's latency budget work is what moves it.

## Keep it

This exercise's artefacts are the ones you'll reuse the longest: the labelled corpus feeds every future turn-taking refactor; the scoring script catches regressions silently added by model vendor changes; the dashboard answers "did it get better" from a shared language. Commit all three; do not treat them as throwaway.
