# 03. The Conversational Eval Harness — Turn-Taking, Barge-In, TTFA, Mouth-to-Mouth Latency, Interrupt Recovery

## Why this chapter

ASR and TTS evals (Chapters 1 and 2) tell you whether the two ends of the pipeline work in isolation. They say nothing about whether the conversation *flows*. A system can have perfect WER, perfect MOS, and still feel broken because it cuts the user off, or hesitates for half a second after every turn, or fails to recover from an interrupt. Those are properties of the pipeline *as a running conversation*, not of its parts.

This chapter defines the conversational metrics the module owns: turn-taking correctness against ground-truth boundaries, barge-in reaction time distribution, time-to-first-audio distribution, mouth-to-mouth (end-to-end) p95 enforcement, and interrupt-recovery correctness. Every metric here ties back to the budget you committed to in [mod-109](../mod-109-latency-budget-architecture/README.md) and to the VAD / endpointing / interrupt design you built in [mod-106](../mod-106-vad-turn-taking-barge-in/README.md). The purpose of this chapter is to make those commitments *measurable*.

## Turn-taking correctness

A **turn boundary** is the moment the user stops speaking and the system can act. The system detects it (VAD + endpointing, mod-106); evaluation compares that detection to the ground-truth boundary.

Build the ground truth with forced-alignment on labelled audio. [Montreal Forced Aligner (MFA)](https://montreal-forced-aligner.readthedocs.io/) is the canonical open tool; [WhisperX](https://github.com/m-bain/whisperX) produces word-timestamped alignments that are usable at the sentence level. The output is a timestamped boundary per utterance: `t_end_truth`. Your system emits `t_end_detected` when it declares the turn over.

Four categories of turn-boundary behaviour, worth tracking separately:

- **True End-Of-Turn (TEOT) detection.** `t_end_detected` fires within a tolerance window of `t_end_truth`. Report the distribution of `t_end_detected − t_end_truth` (signed; positive means the system waited too long, negative means it jumped the gun).
- **False End-Of-Turn (FEOT) — cutoff.** The system declared turn-end while the user was still speaking. Count these; they are the "cutoff a customer" failures. A high rate here degrades perceived empathy more than almost any other failure.
- **Missed End-Of-Turn (MEOT) — hang.** The user has clearly finished (`t_end_truth` passed), the system has not detected it within a reasonable window (say, 1.5 s). Count these; they are the "awkward silence" failures.
- **Overlap handling.** The user started talking while the system was speaking. Did the system interrupt itself, continue, or restart? This is the barge-in metric below.

Report the signed boundary-error distribution and the FEOT / MEOT rates per stratum. The stratification set is the same as Chapter 1's: acoustic condition, speaker demographic, call phase, utterance length. The same demographic that has a higher WER usually has a higher FEOT rate too — turn-taking failures are not uniformly distributed.

Tolerance windows are product-specific. A healthcare triage agent should err on the side of MEOT (wait longer; never cut the user off); a drive-through order agent should err toward faster TEOT (short, structured turns). Document the asymmetric tolerance before you measure, so the pass/fail is deterministic.

### Collecting ground-truth boundaries

Three practical sources:

1. **Forced alignment on recordings with known transcripts.** MFA + a per-language acoustic model gives you word-level timestamps. The utterance boundary is the end timestamp of the last content word, with a configurable trailing-silence allowance.
2. **Recorded dialogues with explicit silence markers.** A test harness that plays a scripted user clip and a known inter-turn pause into the pipeline — the boundary is in the script, not inferred.
3. **Human-labelled turn boundaries.** Slow but necessary for naturalistic conversation data. Use the same double-annotation + IRR practice as Chapter 1.

Prefer (2) for regression testing because it is reproducible; use (1) at scale for production-shape samples; use (3) sparingly for the hardest cases.

## Barge-in reaction time

**Barge-in** is the user speaking while the agent is speaking. The agent is supposed to detect it, stop speaking, and listen. The metric is the distribution of **barge-in reaction time**: from the first frame of user speech during agent audio, to the last frame of agent audio played out. This includes VAD detection latency, the orchestrator's cancel, the TTS cutoff, and the jitter-buffer drain.

Measure it by scripting a test harness that plays agent TTS and user audio simultaneously, with the user audio starting at a known offset into the agent turn, then measuring the agent audio's actual tail end at playout. Report:

- **p50, p95, p99** reaction time, in ms.
- **Distribution shape.** Multimodal distributions almost always indicate two different code paths (e.g., mid-sentence vs. between-sentences cutoff behaviour).
- **Dropped-frames count.** If the orchestrator's cancel races with the TTS flush, you can get audio frames played *after* the cancel. These are the "bot kept talking for half a sentence" failures.

Target budgets live in mod-106 and mod-109. A common product target is p95 barge-in reaction time under ~500 ms end-to-end; validate against your own product's commitment.

Three failure modes you will see:

- **The orchestrator detected the barge-in but the TTS provider's streaming connection is already committed.** You stopped sending tokens but the provider's remaining audio is in flight. Fix: cap the provider's audio buffer; use a streaming TTS that honours cancel.
- **The VAD does not trigger over the agent's own audio.** Acoustic echo cancellation is doing its job but is suppressing the real user signal. Fix: AEC tuning, mod-106.
- **The jitter buffer holds the audio.** The agent's audio has already been handed to the client but is in the playout queue. Fix: trim at the transport layer, mod-107.

Which of the three you are hitting is readable from the span breakdown you set up in Chapter 6; without per-leg instrumentation on the barge-in path, the fix is a guess.

## Time-to-first-audio (TTFA)

**TTFA** is from the end of the user's turn to the first audio byte the user hears from the agent. In the latency-budget vocabulary (mod-109), it is the "turn gap" — the user-perceivable silence. It is composed of ASR final latency, orchestration / prompt build, LLM TTFT, TTS TTFB, transport RTT; the per-leg attribution is the whole point of the mod-109 instrumentation you already have.

Report TTFA distribution as p50 / p95 / p99. Report per-leg contribution as a stacked distribution so you can tell *which* leg owns a latency spike without instrumenting a whole call. If the p95 moves by 200 ms, the stacked view answers "LLM TTFT or TTS TTFB?" in one glance.

A TTFA eval harness has a few structural requirements:

- **Scripted, reproducible user audio.** Pre-recorded utterances with known boundaries, so `t_end_truth` is known and not inferred.
- **Clock-synchronised endpoints.** The harness that records "first audio out" has to share a clock with the one that records "user audio end". A monotonic clock inside one process is fine; across processes you use OTel span events + the clock discipline in mod-109 Chapter 2.
- **Run the harness on the real pipeline, not a mock.** Mocked LLM / TTS calls have clean latency distributions; the real tail is where the bugs are. Keep a small eval set and run it on the real pipeline.

The acceptance threshold is the product's mouth-to-mouth budget from mod-109 minus the user's own speech-to-silence latency. Common voice-agent product targets are in the 500 ms – 1 s p95 range; verify against your own number before you cite anyone else's.

## Mouth-to-mouth p95 enforcement

**Mouth-to-mouth latency** is from the user's last audible sound to the first audible sound of the agent's response — a superset of TTFA that bakes in the playout-side transport. It is the number a customer actually experiences; it is the number you commit to in the mod-109 budget.

The eval harness's job is to enforce it:

1. **Compute mouth-to-mouth per turn** on a scripted eval set. Dozens of turns is enough for a point estimate; hundreds are enough for a stable p95.
2. **Report the distribution** per stratum. Tail latencies cluster by acoustic condition, by region / network, and by prompt length.
3. **Gate on p95, not mean.** A mean under budget with a long tail still produces awful calls; the commitment is on p95 (sometimes p99) because that is the user-experience metric.
4. **Document the test's network and region**. A p95 measured on a dev machine with a local LLM endpoint tells you nothing about a production p95 across WAN.
5. **Enforce in CI (Chapter 4).** A PR that moves p95 beyond the budget fails the eval gate. Not after a canary; at the PR.

The enforcement gate is the single most important conversational-eval output. Everything else in this chapter is diagnostic.

## Interrupt-recovery correctness

An interrupt is handled; now what? The agent has to decide whether to:

- **Resume** — continue where it left off, with or without recap.
- **Restart** — re-read the full previous turn (e.g., for critical disclosures the user missed).
- **Yield** — listen for a follow-up without resuming.

The state machine for this lives in your orchestration layer (mod-111 Chapter 5). The eval harness has to verify that the state machine does what was specified.

Build an interrupt-recovery test suite with labelled scenarios:

- **Interrupt-then-silence.** User barges in, then stops without speaking. Agent should yield briefly, then prompt.
- **Interrupt-then-redirect.** User barges in with a new question. Agent should answer the new question, not the old one.
- **Interrupt-mid-disclosure.** User barges in during a compliance-required disclosure. Agent must re-read the disclosure after the user's turn (mod-108 Chapter 3; EU AI Act Article 50 openings, HIPAA notices, etc., cannot be skipped).
- **Repeated-interrupt.** User barges in twice in rapid succession. Agent should not loop.

Each scenario is pass/fail against a specified behaviour. The suite is a regression gate: if a model or prompt change breaks disclosure-re-read, the PR fails.

Interrupt-recovery is also content-eval-adjacent (Chapter 4) because the "did the agent answer the right question after the interrupt?" judgement is a judge-as-LLM call; the timing and state-machine properties are this chapter's; the correctness-of-content is next chapter's.

## Confidence and sample sizes

Conversational metrics often have small effective-sample sizes because real conversations are long. A test corpus of 100 calls produces hundreds of turn boundaries but maybe only a few dozen barge-ins. Three practical consequences:

- **Bootstrap every CI.** Resample turns (or calls — depending on what you're measuring) with replacement 1000 times; report 2.5 / 97.5 percentiles. A p95 point-estimate with no CI is a vanity number.
- **Augment with synthetic / scripted scenarios.** A 24-scenario interrupt suite run 10 times per PR gives you 240 observations with known ground truth — more than a corpus of 100 natural calls provides on the rarest failures.
- **Separate online from offline.** The offline harness is reproducible; the online harness (Chapter 5) has volume but is noisy on rare events.

## A reference pipeline

```python
# Illustrative. The pipeline-under-test exposes `run_turn(user_audio)` and emits
# OTel spans with the events from mod-109 Chapter 2.

def eval_turn_taking(pipeline, scenarios):
    rows = []
    for s in scenarios:
        result = pipeline.run_turn(s.user_audio, truth=s)
        rows.append({
            "scenario_id": s.id,
            "feot": result.t_end_detected < s.t_end_truth - s.feot_tol,
            "meot": result.t_end_detected > s.t_end_truth + s.meot_tol,
            "boundary_err_ms": (result.t_end_detected - s.t_end_truth) * 1000,
            "ttfa_ms": (result.t_first_audio_out - s.t_end_truth) * 1000,
            "mouth_to_mouth_ms": (result.t_first_audio_at_client - s.t_last_user_sound) * 1000,
            "stratum": s.stratum,
        })
    return rows

def eval_barge_in(pipeline, scenarios):
    rows = []
    for s in scenarios:
        result = pipeline.run_barge_in(
            agent_prompt=s.agent_prompt,
            user_interrupt_audio=s.user_audio,
            interrupt_start_t=s.interrupt_start_t,
        )
        rows.append({
            "scenario_id": s.id,
            "reaction_time_ms": (result.t_agent_audio_ended - result.t_user_speech_started) * 1000,
            "frames_after_cancel": result.frames_played_after_cancel,
            "stratum": s.stratum,
        })
    return rows

def eval_interrupt_recovery(pipeline, scenarios, judge):
    rows = []
    for s in scenarios:
        result = pipeline.run_interrupt_recovery(s)
        rows.append({
            "scenario_id": s.id,
            "expected_behaviour": s.expected,  # resume / restart / yield
            "observed_behaviour": judge.classify(result.transcript, s),
            "disclosure_re_read": judge.disclosure_re_read(result.transcript, s)
                if s.requires_disclosure else None,
        })
    return rows
```

The pipeline runs against your real orchestrator; the scenarios come from a scripted scenario set + a forced-aligned production sample; the judge for interrupt recovery is the Chapter 4 judge-as-LLM.

## What belongs in the release-assurance packet

For every orchestration or model change, the packet must include:

- Turn-boundary error distribution, FEOT rate, MEOT rate, per stratum, with CIs.
- Barge-in reaction time distribution (p50 / p95 / p99), per stratum, with CIs, with frames-after-cancel count.
- TTFA distribution per stratum, with per-leg stacked contribution.
- Mouth-to-mouth p95 vs. the mod-109 budget, with pass/fail.
- Interrupt-recovery suite pass rate; every scenario listed, every expected behaviour verified; any failure linked to the state-machine case it reveals (mod-111 Chapter 5).

This packet, plus the ASR packet (Chapter 1) and the TTS packet (Chapter 2), is what the Chapter 4 CI gate enforces.

## Summary

- **Turn-taking correctness** is the signed distribution of `t_end_detected − t_end_truth`, plus the FEOT (cutoff) and MEOT (hang) rates, per stratum.
- **Barge-in reaction time** is measured end-to-end from first user frame to last agent audio frame; report p50 / p95 / p99 and frames-after-cancel.
- **TTFA** and **mouth-to-mouth** latencies are the user-visible silences; enforce the mouth-to-mouth p95 against the mod-109 budget as a pass/fail gate.
- **Interrupt-recovery correctness** is a scenario-based pass/fail suite covering resume / restart / yield cases and mandatory-disclosure re-read.
- **Augment natural-call samples with scripted scenarios** to get enough statistical power on rare events; bootstrap CIs on everything.
- Produce a **release-assurance packet** combining turn-taking, barge-in, TTFA, mouth-to-mouth, and interrupt-recovery results, stratified; this is one of three inputs to the Chapter 4 CI gate.

Next chapter: content eval — judge-as-LLM on call transcripts (correctness, groundedness, PII leakage, policy compliance), task-level correctness against gold scripts, deflection / escalation rate against business baseline, and wiring the whole eval surface into a CI gate so a PR can't merge if voice quality regresses.
