# 01. Pipecat — Processor Pipelines, Transports, and First-Class Integrations

## Why this chapter

By the time you reach this module you can already defend the component choices for a voice agent: STT vendor (mod-103), TTS vendor (mod-104), VAD and turn-taking strategy (mod-106), transport (mod-107), latency budget (mod-109), and safety posture (mod-108). You still have to wire them together. The orchestration layer is where your design turns into a running process — where "ASR → LLM → TTS with interrupts" becomes an actual stream graph that you can deploy, test, and own in production.

Pipecat is a reasonable default for that layer. It is open-source, Python-first, actively developed by [Daily](https://www.daily.co/), and is built around an explicit **processor pipeline** that maps cleanly onto the mental model you built in mod-102 and mod-109. Many of the vendor-specific choices you already made (Deepgram STT, Cartesia or ElevenLabs TTS, OpenAI Realtime, Gemini Live, Hume EVI) have first-class Pipecat processors, so picking Pipecat does not re-litigate the component decisions — it ratifies them.

This chapter is about Pipecat as an *orchestration framework*: what the pipeline abstraction gives you, how transports plug in, how the standard VAD + STT + context + LLM + TTS pipeline is assembled, and where end-of-utterance detection lives in the graph. Later chapters compare LiveKit Agents and Vocode against the same conceptual model.

## The processor pipeline model

Pipecat models the agent as a directed sequence of **frame processors**. Audio, text, control events, and tool results all flow through the pipeline as typed **frames**; each processor subscribes to the frame types it cares about, transforms or emits new frames, and passes them along. The upstream direction carries user-generated events (audio in, transcriptions, LLM requests). The downstream direction carries system-generated events (LLM tokens, TTS audio, playback). This bi-directional frame model is important for interrupts, which we come back to in Chapter 5.

Three practical properties fall out of the model:

- **Processors are composable units, not stages of a sequential script.** You can insert a new processor — a profanity filter, a usage logger, a semantic cache lookup — without rewriting the pipeline body. The graph stays declarative.
- **Latency is attributable per processor.** Because every frame crosses every processor, you can tag frames with timestamps and emit OpenTelemetry spans per processor (mod-109 Chapter 2). The pipeline *is* your latency-budget harness.
- **Interrupts are frames, not special cases.** When VAD detects the user speaking over the bot, Pipecat emits an interrupt frame that cancels downstream TTS and flushes queued output. Interrupt handling is a pipeline concern, not a bespoke hack in each component.

See the [Pipecat architecture docs](https://docs.pipecat.ai/) for the current frame taxonomy and the processor base classes.

## The transport abstraction

Pipecat separates **where audio enters and leaves** from **what the pipeline does with it**. A transport is a processor at both ends of the pipeline that owns the audio I/O and the signalling for a given platform. Switching transports typically does not require changing the body of your pipeline.

Transports Pipecat ships (see the [transport reference](https://docs.pipecat.ai/) for the current list):

- **Daily** — the native WebRTC transport from Pipecat's maintainer. Rooms, participants, data messages, server-side participant for the bot. Good default for a browser-based prototype.
- **LiveKit** — Pipecat can run as a LiveKit participant, which is useful when the rest of your stack is on LiveKit (see Chapter 2).
- **Twilio** — plugs into Twilio Programmable Voice via Media Streams, so the same pipeline handles PSTN calls. The Twilio transport handles the G.711 ↔ linear PCM conversion and the websocket signalling.
- **WebSocket** — generic WebSocket server/client that carries audio frames. Useful for custom front-ends and for integrating with SIP gateways that speak a bespoke protocol.
- **Local** — microphone/speaker on the machine running the pipeline. Only useful for development, but *always* useful for development — the ability to run the exact production pipeline on your laptop is the single highest-leverage thing a framework can give you.

The decoupling means your pipeline body (VAD → STT → LLM → TTS) is the same whether the user is in a browser (Daily/LiveKit transport), on a phone (Twilio transport), or sitting in front of your laptop (Local transport). The transport is where mod-107's decisions land.

## First-class integrations for the components you already picked

The reason Pipecat is a reasonable default rather than a reasonable toy is the breadth of first-class service integrations. For the choices this role has already made in earlier modules, Pipecat ships processors for:

- **STT**: Deepgram, AssemblyAI, Azure Speech, Google Cloud STT, OpenAI Whisper, Groq, Fireworks, local models (faster-whisper, etc.). mod-103 covered the selection; Pipecat provides the processor so the integration is not bespoke.
- **TTS**: ElevenLabs, Cartesia, Deepgram, Azure, Google, PlayHT, Rime, LMNT, OpenAI. mod-104 covered the selection; same pattern.
- **LLM**: OpenAI, Anthropic, Google (Gemini), Groq, Fireworks, Azure OpenAI, Ollama, and any OpenAI-API-compatible endpoint.
- **End-to-end speech-to-speech**: OpenAI Realtime, Gemini Multimodal Live, Hume EVI. These are not "an LLM processor" — they are a single processor that replaces the STT + LLM + TTS trio with a bidirectional model. Covered in mod-105; the pipeline-level implications are in the examples below.
- **VAD**: Silero VAD (the default), WebRTC VAD, and a `SmartTurn`-style end-of-utterance detector. mod-106 covered the choice; Pipecat exposes it as a pipeline processor.

The current, authoritative list of integrations is in the [Pipecat services documentation](https://docs.pipecat.ai/) and in the `pipecat-ai/pipecat` repository under `src/pipecat/services/`. Treat the list as snapshot-dated; new integrations ship frequently.

## The canonical cascaded pipeline

The pipeline you will build in exercise-01 — and the pipeline most Pipecat production agents resemble — is a cascaded ASR → LLM → TTS graph. The pseudocode below shows the shape; verify against the current Pipecat docs before copying, because class names and keyword arguments evolve.

```python
# Illustrative; check current Pipecat docs for exact imports and arguments.
from pipecat.pipeline.pipeline import Pipeline
from pipecat.pipeline.runner import PipelineRunner
from pipecat.pipeline.task import PipelineTask

from pipecat.transports.services.daily import DailyTransport, DailyParams
from pipecat.services.deepgram import DeepgramSTTService
from pipecat.services.openai import OpenAILLMService
from pipecat.services.cartesia import CartesiaTTSService
from pipecat.audio.vad.silero import SileroVADAnalyzer
from pipecat.processors.aggregators.openai_llm_context import (
    OpenAILLMContext,
    OpenAILLMContextAggregator,
)


async def run_agent():
    transport = DailyTransport(
        room_url=DAILY_ROOM_URL,
        token=DAILY_TOKEN,
        bot_name="support-bot",
        params=DailyParams(
            audio_in_enabled=True,
            audio_out_enabled=True,
            vad_enabled=True,
            vad_analyzer=SileroVADAnalyzer(),
            transcription_enabled=False,  # we're using Deepgram, not Daily's built-in
        ),
    )

    stt = DeepgramSTTService(api_key=DEEPGRAM_KEY)
    llm = OpenAILLMService(api_key=OPENAI_KEY, model="gpt-4o-mini")
    tts = CartesiaTTSService(api_key=CARTESIA_KEY, voice_id=VOICE_ID)

    system_prompt = {
        "role": "system",
        "content": "You are a brief, voice-first support agent. Reply in one or two sentences.",
    }
    context = OpenAILLMContext(messages=[system_prompt])
    context_aggregator = OpenAILLMContextAggregator(context)

    pipeline = Pipeline([
        transport.input(),           # audio frames from the user
        stt,                         # audio → transcription frames
        context_aggregator.user(),   # user text → append to context
        llm,                         # context → streaming LLM tokens
        tts,                         # tokens → audio frames
        transport.output(),          # audio frames to the user
        context_aggregator.assistant(),  # final assistant text → append to context
    ])

    task = PipelineTask(pipeline)
    runner = PipelineRunner()
    await runner.run(task)
```

Five details are worth sitting with, because each one carries a design decision from an earlier module into the orchestration layer.

### VAD lives on the transport

The VAD runs inside `transport.input()` rather than as its own processor. This matters because VAD output (speech-started / speech-stopped events) is the signal the pipeline uses to decide when the user's turn ended — and more importantly when the user has started talking *over* the bot. Keeping VAD close to the raw audio makes those events as fresh as possible (mod-109's latency attribution).

### STT → context aggregator → LLM is where turn assembly happens

The STT service emits transcription frames; the **user-side context aggregator** collects interim and final transcripts and, on utterance end, appends the final user message to the conversation context. The LLM processor consumes the context on an LLM-request frame and emits LLM-response frames (streaming tokens). The **assistant-side context aggregator** collects the LLM's streamed response and appends it to the context on turn end.

This split — a user aggregator before the LLM and an assistant aggregator after — is Pipecat's way of expressing the fact that the conversation is a shared piece of state updated from both sides. The context aggregators are the right place to plug in **pinned facts** and **rolling summaries** (Chapter 7).

### TTS runs in parallel with the LLM

The TTS processor starts speaking on the first LLM token, not when the LLM is done. This is the single most important pipeline property for perceived latency — a point we hammered in mod-109 Chapter 3. In Pipecat, it falls out of the frame-streaming model automatically: the LLM processor emits token frames as they arrive from the provider, the TTS processor consumes partial text, calls the TTS provider in streaming mode, and emits audio frames. You do not write this plumbing; you benefit from it by choosing a streaming TTS service.

### End-of-utterance detection is pluggable

Silero VAD gives you voice-activity thresholds. On top of that, Pipecat supports endpointing processors — including an end-of-utterance model like `SmartTurn` — that decide whether the user has finished talking or is just pausing (mod-106 Chapter 4). The endpointing processor sits between the transport input and the STT (or can be fused with VAD) and emits the "turn ended" frame that triggers LLM invocation. The right configuration depends on the latency target you committed to in mod-109.

### Interrupt handling is a pipeline primitive

When the user starts speaking while the bot is in its own audio turn, the transport emits an interrupt frame upstream. Pipecat cancels the in-flight LLM and TTS calls and flushes the queued audio. The user never hears the tail of what the bot was about to say. Chapter 5 of this module designs the state machine behind that cancel-and-resume contract; here, the point is that the primitive exists at the framework layer.

## A realtime (speech-to-speech) pipeline

If you picked a realtime model (OpenAI Realtime, Gemini Live, or Hume EVI) in mod-105, the pipeline shape collapses. The STT, the LLM, and the TTS are *one* bidirectional service — the realtime model processes audio in and emits audio out directly. Pipecat reflects this with dedicated realtime processors.

```python
# Illustrative.
from pipecat.services.openai_realtime import OpenAIRealtimeBetaLLMService

realtime = OpenAIRealtimeBetaLLMService(
    api_key=OPENAI_KEY,
    session_properties={
        "modalities": ["audio", "text"],
        "voice": "alloy",
        "turn_detection": {"type": "server_vad"},
    },
)

pipeline = Pipeline([
    transport.input(),
    realtime,            # audio in → audio out, STT+LLM+TTS fused
    transport.output(),
    # context aggregation differs here — the realtime service manages its own
    # conversation state server-side; you still want a local copy for logging
    # and warm-handoff (Chapter 7).
])
```

Two properties change:

- **Turn detection often moves to the server.** OpenAI Realtime has a `server_vad` turn-detection mode; you still *can* run a client-side VAD for UX cues, but the model decides when the user's turn is over.
- **Tool calls happen inside the realtime session.** The realtime service emits tool-call frames upstream, your pipeline fulfills them, and the result flows back into the session. The tool-call shape is the same contract we design in Chapter 6, but the plumbing is different.

The decision between the cascaded pipeline and the realtime pipeline was made in mod-105; Pipecat makes both expressible with the same framework so you can run both in production without a second framework commitment.

## When Pipecat is the right call (and when it isn't)

Pipecat is a strong default when:

- You need an **open-source, Python** orchestration layer you can run anywhere.
- You want to **mix and match** STT, LLM, and TTS vendors without writing adapters for each.
- You want the **same pipeline** to run on Daily, LiveKit, Twilio, and a local dev machine.
- You expect to **evolve the pipeline** — adding new processors (logging, filters, memory, routing) over time.
- Your team is comfortable owning the deployment (containerising the Python process, scaling worker instances, wiring to a media server).

Pipecat is a weaker fit when:

- You need a Node.js/TypeScript-native stack — Pipecat has a JS SDK for the client side but the agent runtime is Python. LiveKit Agents has a Node runtime (Chapter 2).
- You want a **managed hosted runtime** out of the box. Pipecat gives you the agent; you still run it. (Pipecat Cloud exists but is separate; evaluate current availability and SLAs when you choose.)
- Your whole workload is already on LiveKit and you want tight coupling with LiveKit Agents' lifecycle.
- The agent is a thin wrapper around a single managed voice-agent product (Retell, Vapi, Bland) — then the orchestration layer is theirs, not yours. See Chapter 4.

## Summary

- Pipecat models the agent as a **processor pipeline** of typed frame transformers; the pipeline body is explicit, inspectable, and mapped one-to-one onto the latency budget and interrupt model you built in earlier modules.
- The **transport abstraction** decouples the pipeline body from Daily / LiveKit / Twilio / WebSocket / local audio, so the same code runs on browser, phone, and laptop.
- First-class processors exist for the STT, TTS, LLM, VAD, endpointing, and speech-to-speech services you already selected in mod-103 through mod-106.
- The canonical pipeline is `transport → VAD → STT → context aggregator → LLM → TTS → transport`, with an assistant-side aggregator closing the loop. Streaming TTS on the first LLM token is automatic; interrupts are first-class frames.
- The speech-to-speech pipeline shape collapses the middle three processors into a single bidirectional realtime service; server-side VAD and in-session tool calls are the main structural differences.
- Pick Pipecat when you want an open-source Python runtime with maximum vendor flexibility; look elsewhere (Chapter 2 for LiveKit Agents, Chapter 4 for managed platforms) when your constraints push you in a different direction.

Next chapter: LiveKit Agents — the entrypoint / room / session model, the `VoicePipelineAgent` and `RealtimeAgent` patterns, and the pre-connect / on-message / on-interrupt lifecycle.
