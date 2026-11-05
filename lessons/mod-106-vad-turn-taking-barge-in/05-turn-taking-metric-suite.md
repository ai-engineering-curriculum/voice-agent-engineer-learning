# 05. The Turn-Taking Metric Suite

## Why this chapter

Everything in Chapters 1–4 is tunable: the VAD threshold, the hangover, the EOU model choice, the backchannel classifier. None of it is defensible without numbers. "The barge-in feels snappy" is not a metric; "p95 interrupt reaction time is 180 ms on our telephony audio corpus" is. This chapter is the metric suite a voice-agent engineer ships alongside the pipeline, the ground-truth labels those metrics score against, and the handoff into Module 112's evaluation harness.

Five metrics carry the suite:

1. **Turn-taking correctness** — fraction of turn boundaries where the bot's commit decision matched a human-labelled ground-truth boundary (within a tolerance window).
2. **False-turn-grab rate (FTGR)** — bot interrupted the user when the user was not yielding the turn. Per-minute-of-call normalisation.
3. **Missed-end-of-utterance rate (MissedEOU)** — bot failed to recognise a turn-end and kept the user waiting. Per-turn normalisation.
4. **Median + p95 bot-response gap** — time between the user's actual end-of-speech and the bot's first spoken audio frame.
5. **p95 user-interrupt reaction time** — time between user speech onset during bot-speaks and bot audio stopping.

These are the five numbers you put on a dashboard. Anything less is incomplete; anything more is probably a derivative of these five.

## The ground-truth corpus

You cannot report any of these metrics without a labelled corpus of turn boundaries. Build it once; reuse across the module's exercises, Module 112's harness, and the capstone projects.

### What is a "turn"

For labelling purposes, define a turn operationally rather than linguistically:

- **Turn boundary** = a moment in time where an ideal agent would have begun responding.
- **Backchannel** = a short utterance by the listener that an ideal agent would **not** have reacted to as a turn boundary.
- **Barge-in** = a moment where the listener's utterance should cause the active speaker (the bot, in most cases) to stop.

These definitions are UX-driven, not linguist-driven. Two labellers agreeing on an operational definition is more useful than two labellers each applying their preferred theory.

### Source audio

Use the audio your product actually sees. In order of preference:

1. **Your own call logs** (with consent and compliant retention — see Module 108). Real distribution, real accents, real background noise, real failure modes. Sample stratified by (device type, time of day, customer segment) so the corpus does not skew to the easy cases.
2. **Realistic public corpora.**
   - [Switchboard-1 Release 2 (LDC97S62)](https://catalog.ldc.upenn.edu/LDC97S62) — telephony English conversation. Dialog-act annotations include backchannel tags (see [SwDA](http://compprag.christopherpotts.net/swda.html)); useful for backchannel work.
   - [Fisher English Training Part 1 (LDC2004S13)](https://catalog.ldc.upenn.edu/LDC2004S13) / [Part 2 (LDC2005S13)](https://catalog.ldc.upenn.edu/LDC2005S13) — larger English telephony corpus.
   - [CallHome / CallFriend](https://catalog.ldc.upenn.edu/LDC97S42) — multilingual telephony.
   - [AMI meeting corpus](https://groups.inf.ed.ac.uk/ami/corpus/) — multi-party meetings with turn and overlap annotations; useful for overlap and barge-in.
3. **Synthesised stress corpora.** If a real failure mode is rare in your logs (e.g., aggressive mid-utterance pauses, speakers in loud environments), augment by recording it deliberately. A small set of 50–100 scripted audio clips covering each failure mode you want to measure beats scraping for naturally-occurring examples you'll never find enough of.

### Labelling

For each clip in the corpus, annotate:

- `turn_boundary_ms` — one or more points in time (relative to the clip start) where the "current speaker" finished their turn.
- `barge_in_events_ms` — points in time where a listener (user or bot) legitimately took the floor during the other party's turn.
- `backchannel_events_ms` — points in time where a listener produced a non-floor-taking vocalisation.
- Optional: `speaker_label` per annotation when the audio has multiple speakers.

Two labellers per clip; measure inter-rater agreement (Cohen's κ on boundary presence within a tolerance window; typical tolerances: 200 ms). Target κ ≥ 0.75. Below that, the labelling guide is underspecified and the metrics will be noise.

Tool: [Label Studio](https://labelstud.io/) or [Praat](https://www.fon.hum.uva.nl/praat/) for audio span annotation. Export JSONL with per-event timestamps.

Rough corpus-size calibration: a few thousand turns is enough for the four rate metrics to have ± 1 % confidence intervals. For a new deployment, 1–2 hours of labelled audio stratified across your production distribution gets you started; 10 hours is a defensible production corpus.

## Metric 1: turn-taking correctness

**Definition.** For each ground-truth turn boundary at `t*`, the pipeline is **correct** if it commits a turn (fires `response.create`, closes the input buffer, or whatever your commit action is) at some `t_commit` with `|t_commit - t*| ≤ τ`, where τ is a tolerance window (typical: 300 ms).

```
correctness = (# correct commits) / (# ground-truth turn boundaries)
```

Counting rules:

- A commit with no nearby ground-truth boundary is a **false commit** (a false turn-grab if it happens mid-utterance; a spurious commit on noise otherwise).
- A ground-truth boundary with no nearby commit within τ is a **missed boundary**.
- Each commit matches at most one boundary; use nearest-match within τ.

Report correctness overall, and stratified by:

- Utterance length (short / medium / long) — short-utterance correctness is often very different from long-utterance.
- Presence of mid-utterance pauses — pauses >500 ms within an utterance are the hard case; report separately.
- Noise condition — clean, moderate, noisy. If you labelled noise class per clip.

### Why τ = 300 ms

Not sacred. 300 ms is wide enough to cover acoustic + classifier + network latency on a reasonable stack; narrower windows (200 ms) are stricter; wider windows (500 ms) are more forgiving but closer to the "sluggish" threshold users can feel. Pick explicitly and keep it fixed across the dashboard history, or every refresh of the number looks like a change of pipeline.

## Metric 2: false-turn-grab rate (FTGR)

**Definition.** Rate at which the pipeline commits a turn during the user's utterance (i.e., not at a ground-truth boundary), normalised per minute of user speech.

```
FTGR = (# false commits during user speech) / (total minutes of labelled user speech)
```

Characteristic ranges:

- **FTGR < 0.3 / min** — good. Users rarely feel cut off.
- **FTGR 0.3–1 / min** — marginal. Noticeable on long calls.
- **FTGR > 1 / min** — the top-of-defect report item on your tracker.

FTGR degrades fast with tight silence-duration hangover and loose EOU thresholds. Chapters 1 and 2 are the tuning surface.

**Important stratification.** Report FTGR separately for (a) mid-utterance pauses — the EOU failure mode of interest — and (b) backchannels wrongly taken as turn-ends — the Chapter 4 failure mode. They are fixed by different tuning levers.

## Metric 3: missed-end-of-utterance rate (MissedEOU)

**Definition.** Rate at which the pipeline fails to commit a turn within `τ_miss` of a ground-truth turn boundary. Different `τ_miss` from the `τ` used in correctness:

- `τ_miss = 1500 ms` is a sensible default. Beyond that the user has either repeated themselves or given up.

```
MissedEOU = (# missed boundaries) / (# ground-truth turn boundaries)
```

A missed EOU is the user staring at silence after they finished talking, wondering if the bot is still there. Users file these as "the bot was dead."

Characteristic ranges:

- **MissedEOU < 1 %** — good.
- **MissedEOU 1–5 %** — noticeable.
- **MissedEOU > 5 %** — the users think the pipeline is broken.

Opposite trade-off from FTGR: long silence-durations and conservative EOU thresholds push MissedEOU up. The tuning is zero-sum against FTGR; pick the (FTGR, MissedEOU) operating point that fits the UX, not a single-metric optimum. Chapter 1's "variable hangover driven by EOU confidence" is how you escape the trade-off.

## Metric 4: bot-response gap (median + p95)

**Definition.** For each turn boundary the pipeline committed correctly, the response gap is the wall-clock delay from the ground-truth end-of-speech `t*` to the bot's first spoken audio frame reaching the user's device.

```
response_gap_ms = t_first_audio_at_device - t*
```

Report median and p95. The p95 is the UX metric; the median is the "best case" indicator. A 400 ms median with a 1400 ms p95 is a product with inconsistent snappiness — tail matters more than typical.

Target ranges for a voice agent:

- **Median 300–600 ms, p95 < 1200 ms** — competitive.
- **Median 600–1000 ms, p95 < 2000 ms** — acceptable but noticeably slow.
- **p95 > 2500 ms** — users describe the bot as "laggy."

Measurement harness: see Module 109 for the full latency-budget breakdown (STT final, LLM first-token, TTS first-byte, network jitter). This chapter's metric is the aggregate end-to-end gap; Module 109 is where you decompose it.

A note on where `t*` comes from. On labelled corpora, `t*` is the human-annotated boundary. On live production you do not have `t*` directly; approximate as "the moment the VAD declared end-of-speech" and subtract the known VAD hangover — that gives you the user-perceived end-of-speech moment. Not quite ground-truth but good enough to track trends.

## Metric 5: p95 user-interrupt reaction time

**Definition.** For each labelled barge-in event, the reaction time is the wall-clock delay from user speech onset `t_onset` to bot audio stopping at the user's device.

```
reaction_ms = t_bot_audio_stopped_at_device - t_user_onset_at_mic
```

Report p50 and p95. The target per Chapter 3 is `p95 < 200 ms`. In practice:

- **p95 < 200 ms** — "feels human."
- **p95 200–400 ms** — acceptable; users notice but tolerate.
- **p95 > 500 ms** — users repeat themselves. Common on stacks without an interruptible player.

Measuring at the user's device requires client-side instrumentation: a timestamp when the client stops writing audio to the sink. A server-side approximation (`response.cancel` timestamp) misses the player-flush latency and underestimates reaction time by 50–200 ms; it is better than no measurement, but note which measurement you're reporting.

**Stratification.** Report separately for (a) barge-ins classified correctly as turn-grabs and (b) backchannels that incorrectly triggered a cancel. The second number is the one users complain about even though technically the cancel was "fast."

## The five-metric dashboard

What goes on the single-page dashboard:

| Metric | p50 or rate | p95 | Target |
|---|---|---|---|
| Turn-taking correctness | — | — | > 90 % |
| False-turn-grab rate | — / min | — | < 0.3 / min |
| Missed-EOU rate | % | — | < 1 % |
| Bot-response gap | ms | ms | p95 < 1200 ms |
| User-interrupt reaction time | ms | ms | p95 < 200 ms |

With stratifications below:

- By noise condition (clean / moderate / noisy).
- By utterance length (short < 1 s / medium 1–5 s / long > 5 s).
- By mid-utterance pause presence.
- By language and accent class (if multilingual).
- By device class (headset / speakerphone / telephony).

A dashboard that shows just the overall number hides the real failure modes. The stratifications are where you discover that your p95 reaction time is fine on headsets and terrible on speakerphones.

## Integration with Module 112

Module 112 is the evaluation-and-observability module; Chapter 5 of this module is a direct input. The handoff:

- **The labelled corpus** lives in the shared eval-data repository. Define its schema here; Module 112 defines the harness that reads it.
- **The scoring script** is a reusable artefact: given (pipeline commit events log, labelled corpus), emit the five metrics + stratifications. Author it in Exercise 5 of this module; reuse in Module 112.
- **Alerting.** Production should alert when any metric drifts 2σ from its 14-day baseline. Module 112 owns the alerting plumbing; this module defines the metric contract.
- **Regression harness.** Every pipeline change — new VAD threshold, new EOU model, new classifier — runs the corpus and reports the five metrics as a regression suite. CI blocks on FTGR / MissedEOU / reaction-time regressions.

## Simulation harness for pre-deployment evaluation

Before you touch production, replay the labelled corpus through your pipeline offline. The harness:

1. For each clip, feed audio frames into the pipeline at realtime rate.
2. Capture every commit decision, every cancel, every bot-audio emission.
3. Align against the ground-truth labels; emit the five metrics.
4. Run on every CI pull request. Fail the build on regression.

Two practicalities. Replay requires the pipeline to accept audio from a file source as if it were a mic; build the file-source adapter early. And realtime replay (not faster) is necessary because many timing bugs only appear at realtime rate — a 5× replay hides buffering issues.

## A/B testing in production

Once the regression harness is in place, A/B new configs:

- Randomise sessions into treatment and control at session start.
- Instrument per-session the five metrics (stratified by whatever cohort you care about).
- Require sample sizes of at least a few hundred sessions per arm before concluding. Turn-taking is a tail-dominated metric; small samples hide large regressions in p95.
- Keep sessions in an arm sticky — a user who hits control on turn 1 should be in control on turn 50. Mid-session arm switching corrupts the data.

## Qualitative companions to the quantitative suite

Numbers alone miss conversational texture. Twice a release, listen to ten sessions end-to-end:

- Pick five sessions from the worst-decile p95 reaction time.
- Pick five sessions random from the modal distribution.
- Write down what you hear. The notes catch failure modes the five metrics miss — robotic resumption after barge-in, incorrect backchannel acknowledgements, discourse-marker misses.

This is not optional. Teams that ship only on the dashboard regress on the qualitative experience even as the numbers improve, because the dashboard measures the things it was designed to measure.

## Common anti-patterns

- **Reporting only overall numbers.** The stratifications are where you find the defects.
- **Using τ = ∞ for correctness.** Every commit matches something eventually; the metric degrades to "did the pipeline commit at all." Keep τ tight (200–300 ms).
- **Measuring reaction time at the server.** Underestimates by the player-flush latency. Measure at the user's device.
- **Treating FTGR and MissedEOU as independent.** They trade off against each other; moving the silence-duration hangover moves both in opposite directions. Report the operating point, not a single number.
- **No regression harness.** Tuning without replay is tuning on vibes. Build the replay harness before the second tuning iteration.
- **Over-weighting the median over the p95.** Users feel the tail. The dashboard should lead with p95.
- **Sampling only easy audio.** A corpus that is 90 % clean headset audio predicts headset-only performance. Stratify.

## Summary

- Five metrics carry the suite: turn-taking correctness, false-turn-grab rate, missed-EOU rate, bot-response gap (p50 + p95), and user-interrupt reaction time (p50 + p95). Target ranges are named in each section.
- The metrics score against a labelled corpus. Build it once: 1–2 hours of audio stratified across your production distribution to start; 10 hours is a defensible production corpus. Two labellers per clip, Cohen's κ ≥ 0.75.
- Stratify by noise condition, utterance length, mid-utterance pauses, language / accent, and device class. Overall numbers hide defects.
- FTGR and MissedEOU trade off against each other via the silence-duration hangover. Report the operating point; use variable hangover (Chapter 2) to push the frontier.
- Reaction time is measured at the user's device, not the server. Player-flush latency is 50–200 ms and matters.
- The scoring script, the replay harness, and the alerting hooks are Module 112's handoff artefacts. Author them here; reuse there.
- Supplement the five metrics with regular listening sessions. The dashboard measures what it was designed to measure; qualitative review catches the rest.

Next chapter: reading the turn-taking literature at engineer altitude — Skantze's 2021 survey, VAP's ICASSP 2022 paper, and the LiveKit turn-detector methodology — enough to defend your VAD / EOU / barge-in choices to a reviewer.
