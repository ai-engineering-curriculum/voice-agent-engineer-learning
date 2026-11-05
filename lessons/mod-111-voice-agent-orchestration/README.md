# mod-111-voice-agent-orchestration: Realtime Voice-Agent Orchestration Frameworks

**Estimated effort:** 16 hours · **Role:** Voice & Multimodal Agent Engineer · **Level:** L35

## What this module is for

By the time you reach this module you can defend every component choice for a voice agent in isolation — STT (mod-103), TTS (mod-104), realtime speech-to-speech (mod-105), VAD and turn-taking (mod-106), transport (mod-107), safety and compliance (mod-108), latency budget (mod-109), multimodal input (mod-110). This module is where those choices get *assembled* into a running agent. The orchestration layer is the piece that turns a collection of vendor contracts into a process you can deploy, test, and operate.

There are four real options at this level: build on **Pipecat**, build on **LiveKit Agents**, build on **Vocode** for lightweight / custom pipelines, or buy a **managed voice-agent platform** (Retell, Vapi, Bland, or a contact-centre AI vendor). The module teaches each option well enough to pick between them on evidence, then teaches the orchestration-level design work (interrupt state machine, streaming tool calls, long-call memory, warm handoff) that is required of *any* choice. It closes with a build-vs-buy defence that ties every decision back to the mod-108 posture, the mod-109 budget, and the mod-112 eval bar.

## Learning objectives

- Build on **Pipecat** — processor pipeline + transport abstraction (Daily / LiveKit / Twilio / WebSocket / local) + first-class integrations for Deepgram / ElevenLabs / Cartesia / OpenAI Realtime / Gemini Live / Hume EVI; author a pipeline with VAD + STT + context aggregator + LLM + TTS + end-of-utterance detection.
- Build on **LiveKit Agents (Python + Node)** — entrypoint + room + session + cascaded `AgentSession` and realtime patterns; author pre-connect + on-message + on-interrupt lifecycle; deploy to LiveKit Cloud or self-hosted.
- Build on **Vocode (Python)** — conversation + transcriber + synthesiser + input/output devices; use for lightweight custom pipelines.
- Survey **managed voice-agent platforms** — Retell (Conversation Flow + voice library + batch dispatch), Vapi (squads + call transfers + assistants), Bland (pathways + dispatch + custom LLM), Hamming + Phonely + Observe.AI — with build-vs-buy decision criteria.
- Design the **interrupt-aware state machine** — states are not sequential; the agent is in LISTENING / PROCESSING / SPEAKING / INTERRUPTED simultaneously across different legs; design the cancel-and-resume contract.
- Author **tool / function calling inside a streaming audio turn** — the parallel-tool-call shape; the progress-filler pattern for long-running tools; integrate MCP for shared enterprise tool catalogues.
- Manage **context length + memory across a long call** — rolling window + rolling summary + pinned facts; warm-handoff context packaging for the next agent.
- Decide **build vs. buy end-to-end** — frontier Realtime API (highest latency ceiling, least control) vs. own pipeline on Pipecat / LiveKit (medium latency, highest control) vs. managed platform (lowest effort, platform-dependent control + cost); defend the choice against the mod-109 budget, the mod-108 compliance posture, and the mod-112 eval bar.

## Lecture chapters

1. [01 — Pipecat: Processor Pipelines, Transports, and First-Class Integrations](01-pipecat-pipeline-and-processors.md)
2. [02 — LiveKit Agents: Entrypoints, Sessions, and the Lifecycle](02-livekit-agents-sessions-and-lifecycle.md)
3. [03 — Vocode: The Minimal Compare Point](03-vocode-minimal-compare-point.md)
4. [04 — Managed Voice-Agent Platforms: The Survey](04-managed-platforms-survey.md)
5. [05 — The Interrupt-Aware State Machine](05-interrupt-aware-state-machine.md)
6. [06 — Streaming Tool Calls and Progress Fillers](06-streaming-tool-calls-and-progress-fillers.md)
7. [07 — Context, Memory, and Warm Handoff](07-context-memory-and-warm-handoff.md)
8. [08 — Build vs. Buy, End to End](08-build-vs-buy-end-to-end.md)

## Exercises

- [exercise-01 — Ship a Pipecat Pipeline with VAD + STT + LLM + TTS](exercises/exercise-01-ship-a-pipecat-pipeline-with-vad-stt-llm-tts.md) (3 h)
- [exercise-02 — Ship a LiveKit Agent with AgentSession and Interrupts](exercises/exercise-02-ship-a-livekit-agent-with-voicepipelineagent-and-interrupts.md) (3 h)
- [exercise-03 — Walk a Vocode Pipeline as the Minimal Compare Point](exercises/exercise-03-walk-vocode-pipeline-as-the-minimal-compare-point.md) (1 h)
- [exercise-04 — Managed Platforms Tour (Retell, Vapi, Bland)](exercises/exercise-04-managed-platforms-tour-retell-vapi-bland.md) (2 h)
- [exercise-05 — Interrupt-Aware State-Machine Design](exercises/exercise-05-interrupt-aware-state-machine-design.md) (2 h)
- [exercise-06 — Streaming Tool Calls and Progress Fillers](exercises/exercise-06-streaming-tool-calls-and-progress-fillers.md) (2 h)
- [exercise-07 — Context, Memory, and Warm Handoff](exercises/exercise-07-context-memory-and-warm-handoff.md) (2 h)

## Structure

- `01-…md` … `08-…md` — lecture chapters.
- `exercises/` — per-exercise prompts.
- `labs/` — long-form hands-on labs (not required for this module; exercises 01–07 together constitute the hands-on build and build on each other).
- `quizzes/` — knowledge checks.
- `resources.md` — primary-source references.

## How to use this module

Read the chapters in order. Chapters 1–4 survey the four orchestration options at the level you need to choose between them on evidence. Chapters 5–7 then teach the orchestration-level design work (interrupt contract, streaming tools, memory, warm handoff) that *every* option forces you to own at least part of. Chapter 8 is where the pieces resolve into a build-vs-buy defence.

The exercises build a single reference agent that evolves across the module. Exercise-01 (Pipecat) and exercise-02 (LiveKit Agents) produce two real pipelines you can measure against each other. Exercise-03 (Vocode) is the minimal compare point and exercise-04 (managed platforms tour) is the procurement-style measurement pass. Exercises 05–07 extend whichever pipeline you preferred from 01/02 with the interrupt state machine, streaming tool calls + fillers, and the memory + handoff plumbing.

A rough pacing: one evening on Chapter 1 and exercise-01; one evening on Chapter 2 and exercise-02; a short session on Chapter 3 and exercise-03; one evening on Chapter 4 and exercise-04; one evening on Chapter 5 and exercise-05; one evening on Chapter 6 and exercise-06; one evening on Chapter 7 and exercise-07; a shorter session on Chapter 8 to write the build-vs-buy defence.

Prerequisite modules: [mod-102](../mod-102-speech-pipeline-foundations/README.md), [mod-103](../mod-103-stt-production-engineering/README.md), [mod-104](../mod-104-tts-production-engineering/README.md), [mod-105](../mod-105-realtime-speech-to-speech/README.md), [mod-106](../mod-106-vad-turn-taking-barge-in/README.md), [mod-107](../mod-107-telephony-webrtc-transport/README.md), [mod-108](../mod-108-safety-compliance-voice/README.md), [mod-109](../mod-109-latency-budget-architecture/README.md), [mod-110](../mod-110-multimodal-input-design/README.md).

Next module: [mod-112 — Voice and Multimodal Eval and Observability](../mod-112-voice-multimodal-eval-and-observability/README.md).
