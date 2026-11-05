# mod-105-realtime-speech-to-speech: Real-Time Speech-to-Speech and Speech-Native Foundation Models

**Estimated effort:** 16 hours

The classical pipeline you built in Modules 102–104 (STT → LLM → TTS) is one of two architectures a voice agent can ship on. This module teaches you the other — speech-native foundation models that accept audio in and emit audio out directly, over a single stateful socket — and how to decide between them. You will learn the OpenAI Realtime API, Google's Gemini Live API, and Anthropic's current partner-relay stance for voice; the duplex spectrum from half-duplex managed vendors to Moshi's full-duplex open-weights reference; the vendor safety stacks (System Cards) and their mapping to the EU AI Act Article 5(1)(f) emotion-inference prohibition; and the open-weights speech foundation models worth knowing by name.

## Learning objectives

- Build on the OpenAI Realtime API — gpt-4o-realtime, gpt-4o-mini-realtime, Advanced Voice — with WebSocket + WebRTC transports, server-side VAD, function calling inside the voice loop, interrupts, prompt caching, session + turn-detection configuration; author the session lifecycle (begin / append-audio / commit / interrupt / end).
- Build on the Google Gemini Live API — bidirectional streaming over WebSocket, native audio input and output, continuous video + screen share in a single turn, tool calling mid-turn, grounding with Google Search; author the session lifecycle for the Gemini Multimodal Live stack.
- Build on Anthropic Claude for voice — current stance (realtime voice via partner realtime-relay stacks: Twilio ConversationRelay, LiveKit Agents partner integration, Pipecat Anthropic-plugin pipelines) rather than a first-party realtime WebSocket; author the pipeline so swapping the LLM leg (Claude vs. GPT vs. Gemini) is a config change, not a rewrite.
- Decide pipeline (STT → LLM → TTS) vs. speech-native (Realtime / Gemini Live / Moshi / Qwen2-Audio) vs. hybrid (speech-native for the chatty turns, pipeline fall-back for tool-heavy turns), scoring against latency, cost, interrupt behaviour, voice-selection freedom, logging / auditability, safety mitigations, and language coverage.
- Reason about half-duplex vs. full-duplex design — the Kyutai Moshi + Mimi architecture (sub-200 ms response, streaming overlap with the user), the OpenAI Advanced Voice half-duplex + VAD stance, the Gemini Live bidirectional model — and the downstream implications for turn-taking (Module 106), latency budget (Module 109), and consent capture (Module 108).
- Read the GPT-4o System Card and the equivalent safety discussion from Google / Anthropic — what voice-cloning safeguards are in place, what emotion-inference boundaries are enforced, and how those map to the EU AI Act Article 5(1)(f) prohibition on workplace / education emotion-inference.
- Survey open-weights speech foundation models — Kyutai Moshi + Mimi, Meta Spirit LM, Qwen2-Audio, Meta SeamlessM4T for speech translation, AudioLM / SoundStream heritage — and know when open-weights is a defensible compliance / cost choice.

## Lecture chapters

1. [01 — The OpenAI Realtime API: session lifecycle, transports, and the voice loop](01-openai-realtime-api-session-lifecycle.md)
2. [02 — The Gemini Live API: bidirectional streaming, native audio, and continuous video](02-gemini-live-api-session-lifecycle.md)
3. [03 — Claude for voice: realtime relays, partner stacks, and a swappable LLM leg](03-claude-voice-via-realtime-relays.md)
4. [04 — Pipeline vs. speech-native vs. hybrid: the architectural decision](04-pipeline-vs-speech-native-vs-hybrid-decision.md)
5. [05 — Half-duplex vs. full-duplex: Moshi, Advanced Voice, and what duplex actually means](05-half-vs-full-duplex-moshi-and-advanced-voice.md)
6. [06 — Reading the System Cards: voice-cloning safeguards, emotion inference, and the EU AI Act](06-safety-system-cards-and-eu-ai-act.md)
7. [07 — Open-weights speech foundation models: Moshi, Spirit LM, Qwen2-Audio, SeamlessM4T](07-open-weights-speech-foundation-models.md)

## Exercises

- [exercise-01 — OpenAI Realtime API session lifecycle walkthrough](exercises/exercise-01-openai-realtime-api-session-lifecycle-walkthrough.md)
- [exercise-02 — Google Gemini Live session lifecycle walkthrough](exercises/exercise-02-google-gemini-live-session-lifecycle-walkthrough.md)
- [exercise-03 — Claude voice via realtime-relay pipeline](exercises/exercise-03-claude-voice-via-realtime-relay-pipeline.md)
- [exercise-04 — Pipeline vs. speech-native decision drill](exercises/exercise-04-pipeline-vs-speech-native-decision-drill.md)
- [exercise-05 — Half- vs. full-duplex architecture read: Moshi and Advanced Voice](exercises/exercise-05-half-vs-full-duplex-architecture-read-moshi-and-advanced-voice.md)
- [exercise-06 — Open-weights speech foundation-model tour: Moshi, Qwen2-Audio, SeamlessM4T](exercises/exercise-06-open-weights-speech-foundation-model-tour-moshi-qwen2audio-seamlessm4t.md)

## Structure

- `01-…md` … `07-…md`: lecture chapters.
- `exercises/`: per-exercise prompts (solutions live in the paired `-solutions` repo).
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.

## How this module fits the track

- **Depends on** Modules 102 (audio foundations), 103 (STT), and 104 (TTS) — the pipeline you'll be comparing against lives there.
- **Feeds** Module 106 (VAD and barge-in — the duplex discussion here sets its vocabulary), Module 107 (telephony and WebRTC transport — the transport choices here and there have to agree), Module 108 (safety and compliance — Chapter 6 here is the speech-native-specific slice), Module 109 (latency budget — Chapter 4's TTFB ranges are its primary input), Module 110 (multimodal input — Gemini Live's video lane is where that module starts), Module 111 (orchestration — the LLM-leg abstraction here is its baseline), Module 112 (eval and observability — the auditability discussion in Chapter 4 is a direct input).
