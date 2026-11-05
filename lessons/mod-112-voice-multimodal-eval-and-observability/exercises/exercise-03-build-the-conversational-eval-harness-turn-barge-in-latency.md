# exercise-03: Build the Conversational Eval Harness — Turn-Taking, Barge-In, TTFA, Mouth-to-Mouth, Interrupt Recovery

**Estimated effort:** 3 hours · **Output:** a conversational eval harness, a scripted-scenario suite, a mouth-to-mouth p95 gate aligned to the mod-109 budget, a conversational release-assurance packet

## Objective

Build the harness that measures the voice agent *as a conversation*: turn-taking correctness against ground-truth boundaries, barge-in reaction time, time-to-first-audio, mouth-to-mouth p95 enforcement, and interrupt-recovery correctness. The harness drives the actual production pipeline (do not mock STT / LLM / TTS calls); the outputs feed the Chapter 4 CI gate and tie back to the mod-109 budget as the pass/fail threshold.

## Prerequisites

- [Chapter 3 — The Conversational Eval Harness](../03-conversational-eval-harness.md).
- The mod-109 latency budget — the specific product-committed p95 mouth-to-mouth number and its decomposition.
- A pipeline from mod-111 (Pipecat / LiveKit Agents) wired end-to-end with the per-leg OTel spans and events from mod-109 Chapter 2.
- [Montreal Forced Aligner](https://montreal-forced-aligner.readthedocs.io/) or [WhisperX](https://github.com/m-bain/whisperX) for deriving ground-truth turn boundaries on labelled audio.
- Python 3.11+. `numpy`, `scipy`, `matplotlib` for distributions; the OTel SDK for reading back the per-leg events.

## Steps

1. **Collect a scripted scenario set.** 30–60 scenarios with ground-truth turn boundaries encoded. Mix: short clean turns, long turns, short-pause-then-continue (tests MEOT / FEOT boundary), rapid back-and-forth, inter-word pauses, in-car background, call-centre narrowband. Use pre-recorded user audio so `t_end_truth` is exact, not inferred.

2. **Build a replay harness.** `run_turn(user_audio, scenario)` streams the pre-recorded audio into the pipeline and captures: `t_end_detected` (VAD / endpoint), `t_first_audio_out` (TTS first byte at the egress transport), `t_first_audio_at_client` (if a client-side probe is available — otherwise approximate via server egress + RTT), the OTel span for the turn, and the final transcript.

3. **Turn-taking metrics.** For every scenario, compute signed boundary error `t_end_detected − t_end_truth`, and classify as TEOT / FEOT / MEOT against configurable tolerances. Report per-stratum FEOT and MEOT rates and the signed boundary-error distribution. Bootstrap 95% CIs.

4. **Barge-in metrics.** Add scripted barge-in scenarios (user starts at known offsets into agent audio). Measure reaction time (first user frame → last agent audio frame played out) and count frames-after-cancel. Report p50 / p95 / p99.

5. **TTFA and mouth-to-mouth distributions.** From the per-turn OTel events (`asr.final`, `llm.first_token`, `tts.first_byte`, playout boundary), derive TTFA per turn. Report distribution per stratum and a stacked contribution breakdown. Derive mouth-to-mouth per turn; produce the p95.

6. **Enforce mouth-to-mouth p95 vs. mod-109 budget.** The harness's top-line pass/fail is "p95 mouth-to-mouth ≤ budget". If not, fail and print the per-leg contribution so the author sees which leg owns the overrun.

7. **Interrupt-recovery scenario suite.** Build the labelled scenarios from Chapter 3: interrupt-then-silence, interrupt-then-redirect, interrupt-mid-disclosure, repeated-interrupt. Each scenario has an expected behaviour. Use a judge (from exercise-04) to classify the observed behaviour on the transcript. Report pass count per scenario type. Flag any disclosure-re-read failure as a hard gate.

8. **Emit the release-assurance conversational packet.** YAML/JSON following the Chapter 7 schema: turn-taking stats, barge-in distributions, TTFA + per-leg breakdown, mouth-to-mouth p95 vs. budget pass/fail, interrupt-recovery scenario roll-up.

## Starter guidance

- **Pre-recorded user audio is essential.** Letting a human speak each scenario destroys reproducibility. Spend the time to curate a pre-recorded scenario set — it pays back for the life of the product.
- **Forced alignment is noisy.** MFA on word boundaries is usable; on utterance *endings* (including silent trailing content) it has a few hundred ms of play. Verify against hand labels on the hardest cases.
- **Barge-in offsets should span the agent's audio.** Test interrupts at 100 ms in, 500 ms in, mid-sentence, between sentences. The failure mode often depends on *when* in the turn the interrupt happens.
- **Mouth-to-mouth is the number you commit to.** TTFA is instrumentation; mouth-to-mouth is the user experience. Gate on mouth-to-mouth.
- **Stratified from day one.** Even the first 30 scenarios should cover a clean-audio bucket and a noisy-audio bucket; otherwise you will not catch acoustic-condition regressions.
- **Reuse the Chapter 1 stratification taxonomy** — same bucket definitions across ASR / conversational eval, so the reports are joinable.

## Acceptance criteria

- [ ] A versioned scenario set (≥30 scenarios) with pre-recorded user audio and encoded ground-truth boundaries.
- [ ] A replay harness that drives the actual pipeline (not mocks) and captures OTel per-leg events.
- [ ] Turn-taking: signed boundary-error distribution, FEOT / MEOT rates per stratum, bootstrap CIs.
- [ ] Barge-in: reaction-time distribution (p50/p95/p99), frames-after-cancel count, stratified.
- [ ] TTFA distribution with per-leg stacked contribution.
- [ ] Mouth-to-mouth p95 computed and compared to the mod-109 budget; a documented pass/fail.
- [ ] Interrupt-recovery suite with labelled expected behaviours and per-scenario pass/fail; any disclosure-re-read failure is a hard gate.
- [ ] A machine-readable `conversational_packet.yaml` / `conversational_packet.json`.

## Stretch goals

- **Client-side playout probe.** If you can instrument the browser or client SDK to timestamp the first-audio-byte at playout, close the loop on the true mouth-to-mouth. Otherwise you are measuring server-egress + modelled RTT.
- **Realtime pipeline variant.** If you shipped both a cascaded and a realtime pipeline (mod-111), run the suite on both and emit a side-by-side packet.
- **Multi-turn scenarios.** Build 10-turn conversation scripts and report conversation-level pass/fail. A single bad turn in a good conversation hurts less than a cascade of marginal turns.
- **Cross-region runs.** Execute the harness from two regions to catch region-specific transport pathology.
- **Scenario-synthesis tool.** Use your TTS to generate alternative phrasings of each scenario to stress-test turn-taking without a human recording session. Flag synthetic scenarios separately in the packet.

## Keep it

The packet feeds exercise-04's CI gate. The scripted scenarios flow into exercise-05 as the regression-fixture seed (a canonical set that cannot regress). The per-leg attribution in the stacked TTFA plot is the same signal the Chapter 6 OTel instrumentation exposes — the conversational harness is a consumer of that instrumentation, not a parallel source.
