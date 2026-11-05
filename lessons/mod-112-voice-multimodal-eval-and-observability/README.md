# mod-112-voice-multimodal-eval-and-observability: Voice and Multimodal Evaluation + Observability

**Estimated effort:** 14 hours · **Role:** Voice & Multimodal Agent Engineer · **Level:** L35

## What this module is for

Modules 101–111 built the voice agent: a pipeline you can defend component-by-component, running on an orchestration framework, inside a latency budget, under a compliance envelope. This module is how you prove it still works — on every PR, on every production call, on every vendor-side model bump. Evaluation and observability are the discipline that stands between a demo that works and a product you can defend to a customer, a regulator, or an on-call engineer at 2am.

The scope is deliberately voice-specific. The generic LLM-app eval stack (datasets, judge rubrics, bootstrapping, calibration) is the `ai-eval-engineer` role's work, and you reuse their patterns here (Chapter 7 names the boundary). What this module owns is the audio-first part: WER / CER / MER against real datasets with inter-rater reliability on your labels; subjective + automated MOS with the P.808 / P.835 / CMOS / UTMOS / DNSMOS stack; conversational metrics that tie turn-taking, barge-in, and mouth-to-mouth latency back to the mod-109 budget; content eval that merges judge-as-LLM with the voice-specific policy and compliance requirements from mod-108; online eval on production with implicit-feedback signals, drift detection, and a regression-fixture back-feed; and the OpenTelemetry instrumentation that puts every one of those metrics on the same correlated trace plane.

At the end of this module the eval-gated CI pipeline refuses to merge a PR that regresses voice quality, and the release-assurance packet that flows to the governance-family `ai-evaluation-engineer` is complete and defensible.

## Learning objectives

- Build the **ASR eval harness** — WER / CER / MER against FLEURS (multilingual), LibriSpeech (clean English), TED-LIUM 3 (conference English), and an in-domain gold set; inter-rater reliability on the gold labels; confidence-calibration plots; stratify by acoustic condition and by speaker demographic.
- Build the **TTS eval harness** — ITU-T P.808 crowdsourced MOS (Prolific / MTurk under P.808 protocol), P.835 signal / background / overall MOS, CMOS pairwise comparison with A/B anchors; UTMOS automated prediction as the CI-friendly proxy; DNSMOS for voice-quality monitoring in production.
- Build the **conversational eval harness** — turn-taking correctness vs. ground-truth boundaries, barge-in reaction time distribution, time-to-first-audio distribution, p95 mouth-to-mouth latency enforcement, interrupt-recovery correctness; aligned to the mod-109 budget.
- Build the **content eval harness** — judge-as-LLM on call transcripts (correctness, groundedness, PII leak, policy compliance), task-level correctness against gold scripts, deflection + escalation rate against business baseline; wire into eval-gated CI so a PR cannot merge if the voice eval regresses.
- Ship **online eval on production traffic** — judge-as-LLM at sample rate; implicit user-feedback signals (call-abandonment, repeated-question, hold time, escalation-to-human, post-call CSAT); drift detection on WER / MOS / latency; regression-fixture back-feed from every incident.
- Instrument with **OpenTelemetry GenAI semantic conventions + voice-specific attributes** — turn-id, speaker-id, modality, TTFB per leg, barge-in reaction time, WER-of-turn, MOS proxy per turn; wire into Langfuse / LangSmith / Arize Phoenix / Weave / Braintrust; stream WebRTC-stats into the same observability plane.
- Cite the **role boundary** to `ai-eval-engineer` (L30, Engineering family) on deep eval methodology and to `ai-evaluation-engineer` (L35, Governance family) on release-assurance packet composition.

## Lecture chapters

1. [01 — The ASR Eval Harness: WER, CER, MER on FLEURS + LibriSpeech + TED-LIUM 3 + Your Gold Set](01-asr-eval-harness.md)
2. [02 — The TTS Eval Harness: P.808, P.835, CMOS, UTMOS, DNSMOS](02-tts-eval-harness.md)
3. [03 — The Conversational Eval Harness: Turn-Taking, Barge-In, TTFA, Mouth-to-Mouth, Interrupt Recovery](03-conversational-eval-harness.md)
4. [04 — Content Eval: Judge-as-LLM on Transcripts, Task-Level Correctness, and the CI Gate](04-content-eval-judge-as-llm-and-ci.md)
5. [05 — Online Eval: Sampled Judge Scores, Implicit Feedback, Drift Detection, and Fixture Back-Feed](05-online-eval-feedback-and-drift.md)
6. [06 — OpenTelemetry GenAI Semantic Conventions + Voice-Specific Attributes + WebRTC Stats](06-otel-voice-attrs-webrtc-stats.md)
7. [07 — Role Boundary: `ai-eval-engineer` (L30) vs. `ai-evaluation-engineer` (L35)](07-role-boundary-to-eval-engineers.md)

## Exercises

- [exercise-01 — Build the ASR Eval Harness (FLEURS + In-Domain)](exercises/exercise-01-build-the-asr-eval-harness-fleurs-plus-in-domain.md) (3 h)
- [exercise-02 — Build the TTS Eval Harness (P.808, CMOS, UTMOS)](exercises/exercise-02-build-the-tts-eval-harness-p808-cmos-utmos.md) (3 h)
- [exercise-03 — Build the Conversational Eval Harness (Turn, Barge-In, Latency)](exercises/exercise-03-build-the-conversational-eval-harness-turn-barge-in-latency.md) (3 h)
- [exercise-04 — Build the Content Eval Harness Wired to CI](exercises/exercise-04-build-the-content-eval-harness-judge-as-llm-wired-to-ci.md) (2 h)
- [exercise-05 — Ship Online Eval with Implicit Feedback and Drift](exercises/exercise-05-ship-online-eval-with-implicit-feedback-and-drift.md) (2 h)
- [exercise-06 — Instrument OpenTelemetry Voice Attributes and WebRTC Stats](exercises/exercise-06-instrument-opentelemetry-voice-attributes-and-webrtc-stats.md) (1 h)

## Structure

- `01-…md` … `07-…md` — lecture chapters.
- `exercises/` — per-exercise prompts.
- `labs/` — long-form hands-on labs (not required for this module; exercises 01–06 together constitute the hands-on build and compose into the full release-assurance packet).
- `quizzes/` — knowledge checks.
- `resources.md` — primary-source references.

## How to use this module

Read the chapters in order. Chapters 1–3 build the three audio-shape eval harnesses — ASR, TTS, conversational. Chapter 4 adds content-level judge-as-LLM graders and wires everything into the CI merge gate. Chapter 5 extends offline eval to online eval on production traffic. Chapter 6 is the OTel + WebRTC instrumentation that unifies the signal plane. Chapter 7 names the role boundaries so you know what you hand off to the two eval-engineer roles adjacent to this one.

The exercises build a single evaluation surface that composes. Exercise-01 produces the ASR packet; exercise-02 the TTS packet; exercise-03 the conversational packet; exercise-04 wires all three plus the content packet into a CI gate that merges-or-blocks a PR; exercise-05 ships the online + drift + incident-fixture loop; exercise-06 unifies the whole thing on a single OTel-based observability plane. By the end you have a reproducible release-assurance packet and a live production telemetry stack, both discoverable through standards (OTel GenAI semconv + the documented `voice.*` namespace).

A rough pacing: one evening on Chapter 1 and exercise-01; one evening on Chapter 2 and exercise-02; one evening on Chapter 3 and exercise-03; a short session on Chapter 4 and exercise-04 (the harness code is largely written by then; this is wiring); one evening on Chapter 5 and exercise-05; a short session on Chapter 6 and exercise-06; a reflective read of Chapter 7 to lock in the role boundaries before you file the first release-assurance packet.

Prerequisite modules: [mod-102](../mod-102-speech-pipeline-foundations/README.md), [mod-103](../mod-103-stt-production-engineering/README.md), [mod-104](../mod-104-tts-production-engineering/README.md), [mod-105](../mod-105-realtime-speech-to-speech/README.md), [mod-106](../mod-106-vad-turn-taking-barge-in/README.md), [mod-107](../mod-107-telephony-webrtc-transport/README.md), [mod-108](../mod-108-safety-compliance-voice/README.md), [mod-109](../mod-109-latency-budget-architecture/README.md), [mod-110](../mod-110-multimodal-input-design/README.md), [mod-111](../mod-111-voice-agent-orchestration/README.md).

This is the final technical module of the Voice & Multimodal Agent Engineer track. The projects — the capstone end-to-end voice agent, the multimodal agent, the eval + observability bundle, and the speech-native architectural brief — assemble what every module in the track produced.
