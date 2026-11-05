# 01. The Voice & Multimodal Agent Engineer Landscape

## Why this chapter

Before you can locate the role on an org chart, you need the map of companies and products that actually hire for it. "Voice-agent engineer" is a recent job title — it crystallised roughly 2023–2025, as streaming LLMs, realtime audio APIs, and WebRTC-native agent frameworks converged. The companies below are the ones generating the demand: they ship realtime audio products where latency, turn-taking, and multimodal fusion are first-class concerns rather than demo-ware.

Use this chapter to recognise the six shapes of employer and the three archetype questions each one asks in an interview loop.

## Six categories of employer

### 1. Voice-AI pure-plays (ASR / TTS / audio-first labs)

These companies sell speech models as API products. The voice-agent engineer at a pure-play either builds the developer-facing agent frameworks on top of the house models, or runs the reference applications that prove the models in production.

| Company | Core product | What this role tends to own |
|---|---|---|
| **Deepgram** | Streaming ASR (Nova family), speech-to-text API | Reference voice-agent integrations; SDK ergonomics; latency benchmarks |
| **AssemblyAI** | Streaming and async ASR; Universal-Streaming | Developer examples; realtime conversation templates |
| **ElevenLabs** | Low-latency TTS, voice cloning, Conversational AI product | End-to-end agent orchestration; voice-UX defaults |
| **Cartesia** | Sonic family TTS, state-space audio models | Realtime streaming TTS integrations; low-latency reference clients |
| **LMNT** | Streaming TTS | Reference voice-agent stacks; latency-oriented sample code |
| **Hume** | Empathic voice models (EVI), expression measurement | Voice interface for emotion-aware agents |

### 2. Realtime-voice orchestration platforms

These frameworks and platforms exist specifically to glue ASR, TTS, LLM, and transport into a running voice agent. They are the libraries you reach for before writing your own event loop.

- **LiveKit** — WebRTC stack with an open-source `livekit-agents` Python / Node framework that supplies session plumbing, VAD, turn detection, and plugin adapters for the major ASR / TTS / LLM providers.
- **Pipecat / Daily** — `pipecat-ai` is Daily's open-source framework for building voice and multimodal AI agents as processor pipelines; Daily runs the underlying WebRTC infrastructure.
- **Vocode** — Open-source library and hosted platform for building voice-based LLM applications.
- **Retell AI**, **Vapi**, **Bland AI** — "voice-agent-as-a-service" platforms where you configure STT provider, LLM, prompts, tools, and telephony and get a dialable number back.

If a job description mentions "Pipecat pipelines", "LiveKit Agents", or "`VoicePipelineAgent`", this is the category.

### 3. Telephony providers with voice-AI surfaces

The traditional CPaaS vendors have retrofitted realtime audio streaming onto their phone-number infrastructure. A voice-agent engineer here is usually bridging PSTN / SIP to a modern LLM stack and worrying about carrier-grade reliability.

- **Twilio** — Programmable Voice with Media Streams (bidirectional audio over WebSocket) and ConversationRelay for direct integration with voice agents.
- **Vonage** — Voice API with WebSocket audio streaming.
- **Zoom** — Zoom Phone and Contact Center APIs; realtime audio for AI companions.

### 4. Frontier labs with realtime voice stacks

The model labs that ship their own first-party realtime voice interfaces. Interview loops here look more like applied research + systems engineering than CRUD-API work.

- **OpenAI** — Realtime API (WebRTC and WebSocket transports) with native speech-to-speech models; ChatGPT Advanced Voice Mode.
- **Google** — Gemini Live API with bidirectional streaming; Google Cloud Speech-to-Text and Text-to-Speech.
- **Anthropic** — Claude's realtime voice exposure has historically been through partner relays (e.g. ElevenLabs Conversational AI, LiveKit integrations) rather than a first-party audio-in/audio-out surface. <!-- needs-research: confirm current state of any first-party Anthropic realtime voice surface at author time -->
- **Meta** — SeamlessM4T (multilingual speech translation), AudioCraft; audio research shipping into consumer products.
- **Kyutai** — French research lab that released Moshi, a speech-to-speech foundation model, and the Mimi audio codec, both open-weight.

### 5. Applied-voice product companies (CX, contact centre, agent-ops)

Companies whose product is a voice agent or a voice-powered copilot for human agents. Interview loops emphasise latency under real call conditions, grounding in a customer's knowledge base, barge-in behaviour, and compliance.

- **Sierra** — Conversational AI agents for customer experience.
- **Decagon** — AI agents for customer support, including voice channels.
- **Cresta** — Realtime agent assist and coaching for contact centres.
- **PolyAI** — Voice assistants for enterprise contact centres.
- **Observe.AI** — Contact-centre intelligence with realtime agent assist.
- **Replicant** — Autonomous voice agents for contact centres.

### 6. Vertical-voice applications

Where the voice agent is the product inside a specific industry's workflow — and where regulation and domain accuracy dominate.

- **Clinical ambient documentation** — **Suki**, **Nuance DAX (Microsoft)**, **Abridge**, **Ambience Healthcare**. Ambient listening during a clinician–patient visit produces a structured note that flows into the EHR.
- **Sales-call intelligence** — **Rilla**, **Sybill**, **Gong**. Record-and-analyse (and increasingly live-coach) sales conversations.

## Three archetype interview questions by category

Different categories test different muscles. If you match a job description to one of the six boxes, you can anticipate the loop:

- **Pure-plays and frontier labs** — "Here is our streaming ASR / TTS / realtime API. Walk me through a reference voice agent on top of it. Where do you spend your latency budget?"
- **Orchestration platforms and telephony vendors** — "Design the pipeline for a user calling a phone number and ending up in a conversation with a Claude- or GPT-powered agent, including barge-in, DTMF handoff, and silent-call teardown."
- **Applied-voice and vertical companies** — "A customer complains the bot interrupts them, misses medical terminology, and ‘sounds robotic'. Which of STT, VAD, LLM, TTS, or orchestration do you suspect first, how do you measure it, and what's the eval you add before shipping a fix?"

The role is the same. The centre of gravity shifts.

## Why this role separated from "applied AI engineer" now

A short list of the tech changes that made this a distinct discipline:

1. **Streaming LLMs are cheap and fast enough** that token-by-token output can drive incremental TTS synthesis. Before that, a text LLM was always a blocking turn.
2. **Streaming ASR providers finalise partials quickly enough** (hundreds of milliseconds) that endpointing can run on live transcripts.
3. **WebRTC-native agent frameworks** (LiveKit Agents, Pipecat) replaced ad-hoc WebSocket plumbing, and SIP-to-WebRTC bridges in CPaaS platforms (Twilio Media Streams, Vonage WebSocket) made telephony integrations routine rather than bespoke.
4. **Speech-to-speech foundation models** (OpenAI Realtime, Gemini Live, Moshi) started replacing cascaded STT→LLM→TTS pipelines for some workloads — which created a new architectural choice to be made.
5. **Compliance and safety requirements** (HIPAA for clinical voice, TCPA for outbound calling in the US, GDPR for voice biometrics in the EU, state-level recording-consent laws) are concrete enough that a specialist is now cheaper than ad-hoc legal reviews.

None of those individually require a new role. Stacked, they do.

## Summary

- Six categories of employer hire for this role: voice-AI pure-plays, realtime-voice orchestration platforms, telephony providers, frontier labs, applied-voice product companies, and vertical-voice applications.
- The centre of gravity shifts by category — pure-plays care about model performance under their own APIs, orchestration platforms about pipeline composition, applied-voice companies about domain accuracy and compliance.
- The role separated from the generic "applied AI engineer" because streaming LLMs, streaming ASR, WebRTC-native agent frameworks, speech-to-speech models, and voice-specific compliance stacked up at the same time.

Next chapter: where this role sits on the AI Engineering level ladder.
