# mod-109-latency-budget-architecture: Real-Time Latency Budget Architecture

**Estimated effort:** 14 hours · **Role:** Voice & Multimodal Agent Engineer · **Level:** L35

## What this module is for

A voice agent either feels like a conversation or it doesn't. The single metric that decides which one is the **turn gap** — the time between the user's end-of-utterance and the first byte of the agent's audio reaching their ear. This module is about making a defensible commitment to that number, instrumenting every leg of the pipeline so you can keep the commitment, architecting for parallelism so the commitment is achievable, placing the deployment so geography doesn't ruin it, and enforcing the whole thing as a CI gate.

This is the budget half of the pipeline; mod-106 covers the turn-taking decisions around it, mod-112 covers the broader eval harness that this gate plugs into, and mod-107 covers the transport underneath.

## Learning objectives

- Decompose the mouth-to-mouth latency budget against ITU-T G.114's one-way-delay thresholds (<150 ms unnoticeable, 150–400 ms usable, >400 ms degrading) — mic capture + encode + network ingress + VAD + ASR + LLM + TTS + playout — and set a realistic conversational-feel target (p95 user-pause-to-bot-first-audio under ~800 ms end-to-end).
- Measure and attribute latency per leg — ASR TTFB, ASR final, LLM TTFT and TPOT, TTS TTFB, network RTT — using WebRTC stats + OpenTelemetry spans (and the GenAI semantic conventions) to attribute every millisecond of the turn gap.
- Design the streaming graph for maximum parallelism — start TTS synthesis on the first LLM token, cancel downstream work on interrupt, cache repeat prefixes with prompt caching, warm pools for low-latency routes.
- Pick region + edge deployment — inference colocated with the media server, same-region provider endpoints, private peering where it earns its keep (AWS Direct Connect, Azure ExpressRoute, Google Cloud Interconnect), and understand the latency cost of a cross-region hop.
- Enforce the budget as an engineering gate — synthetic callers driving live traffic, a p95-turn-gap gate in CI, runtime kill-switches and fallbacks when the budget breaches.
- Cite the sharp boundary to the `ai-infra-inference-engineer` (peer, L35) on serving-layer tail-latency tuning (streaming batching for audio, GPU allocation for concurrent realtime sessions, speculative decoding on the LLM leg); own the architectural budget at this level.

## Lecture chapters

1. [01 — The Mouth-to-Mouth Budget](01-the-mouth-to-mouth-budget.md)
2. [02 — Measuring and Attributing Per-Leg Latency](02-measuring-and-attributing-per-leg-latency.md)
3. [03 — Streaming Graph for Maximum Parallelism](03-streaming-graph-for-parallelism.md)
4. [04 — Region, Edge, and Private Peering](04-region-edge-and-private-peering.md)
5. [05 — Enforcing the Budget as an Engineering Gate](05-enforcing-the-budget-as-a-gate.md)
6. [06 — The Boundary to the AI Inference Engineer](06-the-boundary-to-the-inference-engineer.md)

## Exercises

- [exercise-01 — Mouth-to-Mouth Budget Breakdown for a Reference Pipeline](exercises/exercise-01-mouth-to-mouth-budget-breakdown-for-a-reference-pipeline.md) (3 h)
- [exercise-02 — Per-Leg Measurement with OpenTelemetry Spans](exercises/exercise-02-per-leg-measurement-with-opentelemetry-spans.md) (3 h)
- [exercise-03 — Streaming Graph with TTS Start on First LLM Token](exercises/exercise-03-streaming-graph-with-tts-start-on-first-llm-token.md) (3 h)
- [exercise-04 — Region, Edge, and Private Peering Decision](exercises/exercise-04-region-edge-and-private-peering-decision.md) (2 h)
- [exercise-05 — Ship a Latency Budget CI Gate](exercises/exercise-05-ship-a-latency-budget-ci-gate.md) (2 h)

## Structure

- `01-…md` … `06-…md` — lecture chapters.
- `exercises/` — per-exercise prompts.
- `labs/` — long-form hands-on labs (not required for this module; exercise-02, 03, and 05 together constitute the hands-on work).
- `quizzes/` — knowledge checks.
- `resources.md` — primary-source references.

## How to use this module

Read the chapters top-to-bottom first — they build on each other and the exercises assume the whole stack. Then work the exercises in order. Exercises 01 and 04 are paper deliverables (the budget and the placement decision); exercises 02, 03, and 05 are hands-on and build a working instrumented prototype you will reuse in project-101 and reference in project-104's architectural brief.

A rough pacing suggestion: one evening on chapters 1–3 and exercise-01; one evening on chapter 4 and exercise-04; one evening on exercise-02 (instrumentation) and exercise-03 (streaming graph); one evening on chapters 5–6 and exercise-05 (CI gate + fallback).

Prerequisite modules: [mod-102](../mod-102-speech-pipeline-foundations/README.md), [mod-103](../mod-103-stt-production-engineering/README.md), [mod-104](../mod-104-tts-production-engineering/README.md), [mod-106](../mod-106-vad-turn-taking-barge-in/README.md), [mod-107](../mod-107-telephony-webrtc-transport/README.md).

Next module: [mod-110-multimodal-input-design](../mod-110-multimodal-input-design/README.md).
