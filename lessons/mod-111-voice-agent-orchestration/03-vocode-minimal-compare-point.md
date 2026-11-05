# 03. Vocode — The Minimal Compare Point

## Why this chapter

Pipecat and LiveKit Agents are the two framework-sized choices. [Vocode](https://github.com/vocodedev/vocode-core) (open-source core) is a third option that deserves a chapter not because it is likely to be your production answer — it probably isn't, at this level of role — but because it is the simplest explicit model of "conversation ↔ transcriber ↔ synthesiser ↔ I/O devices", and seeing that model next to the two heavyweights sharpens what Pipecat and LiveKit actually buy you.

Think of Vocode as the *compare point*: a minimal, hackable Python library in the same problem space that lets you reason about what you give up when you drop to a lighter framework, and what you give up when you pick up a heavier one.

## The Vocode mental model

Vocode (the core library) exposes four named components that together make a conversation:

- **Transcriber** — streaming STT (Deepgram, AssemblyAI, Azure, Whisper, etc.).
- **Agent** — the "brain": an LLM (OpenAI, Anthropic, …), a RAG agent, a function-calling agent, or a custom `BaseAgent` subclass.
- **Synthesiser** — streaming TTS (ElevenLabs, PlayHT, Azure, Google, Cartesia, …).
- **Input / output device** — where audio comes from and goes to: microphone/speaker (local), Twilio (phone), Vonage, a WebSocket, or a browser-side client.

A `StreamingConversation` object glues the four together. The key simplification versus Pipecat: there is no generic "frame pipeline" — the four named components talk to each other in a fixed topology, and extension points are subclasses, not arbitrary processor insertion.

```python
# Illustrative — check current Vocode docs for exact class names and args.
from vocode.streaming.streaming_conversation import StreamingConversation
from vocode.streaming.input_device.microphone_input import MicrophoneInput
from vocode.streaming.output_device.speaker_output import SpeakerOutput
from vocode.streaming.transcriber.deepgram_transcriber import DeepgramTranscriber
from vocode.streaming.agent.chat_gpt_agent import ChatGPTAgent
from vocode.streaming.synthesizer.cartesia_synthesizer import CartesiaSynthesizer


async def run_local_agent():
    mic, speaker = MicrophoneInput.from_default_device(), SpeakerOutput.from_default_device()

    conversation = StreamingConversation(
        input_device=mic,
        output_device=speaker,
        transcriber=DeepgramTranscriber(DeepgramTranscriberConfig(
            sampling_rate=mic.sampling_rate,
            audio_encoding=mic.audio_encoding,
            api_key=DEEPGRAM_KEY,
            model="nova-3",
        )),
        agent=ChatGPTAgent(ChatGPTAgentConfig(
            initial_message=BaseMessage(text="Hi — what can I help with?"),
            prompt_preamble="You are a brief, voice-first support agent.",
            model_name="gpt-4o-mini",
        )),
        synthesizer=CartesiaSynthesizer(CartesiaSynthesizerConfig(
            voice_id=VOICE_ID,
            api_key=CARTESIA_KEY,
        )),
    )

    await conversation.start()
```

That's it. Four components, one call, you have a voice loop on your laptop. Swap `MicrophoneInput` + `SpeakerOutput` for a Twilio phone-call connector and the same four components drive a phone call.

## What you gain by going minimal

### A short mental loop

The whole agent runtime fits in your head. When you have to debug a problem — latency on the agent leg, an interrupt landing late, a TTS cut-off — you can trace from input device all the way to output device without climbing an abstraction. For *learning*, this is extremely valuable. For *shipping a complex multi-sub-agent system with a plug-in ecosystem*, it stops helping.

### Easy custom agents

Vocode's `BaseAgent` is explicitly designed to be subclassed. If your "LLM" is actually a state machine, a bespoke pathway engine, or a chain-of-Markovs toy for a specific product, Vocode happily lets you drop it in — because it doesn't ask you to speak a pipeline DSL, just to implement the abstract methods.

### Low framework overhead

Vocode is a Python library you pip-install, not an ecosystem you adopt. If your product is really a small voice-enabled feature inside a larger app and you don't want a new runtime to operate, Vocode can stay out of your way.

## What you give up versus Pipecat / LiveKit Agents

### Fewer first-class integrations, and less uniform across them

Pipecat and LiveKit have active plugin ecosystems with first-class support for the breadth of STT/TTS/LLM/realtime vendors covered in mod-103 through mod-105. Vocode supports a solid set but not an exhaustive one, and it does not treat realtime speech-to-speech as a *first-class* pattern the way Pipecat and LiveKit now do.

### No processor pipeline for middleware

If you want to insert a redaction processor, a semantic cache, a mid-pipeline sentiment classifier, or a usage logger between two stages, Vocode forces you into subclassing one of the four components and doing it there. That's fine until you have three cross-cutting middlewares; at four, Pipecat's processor pipeline earns its keep.

### Transport coverage is narrower

Vocode has connectors for Twilio and Vonage telephony and for local I/O; it does not match LiveKit's or Pipecat's breadth of WebRTC transports. For a product whose roadmap includes browser-based voice sessions on WebRTC, Vocode leaves more of the plumbing to you.

### Less convergent interrupt / turn-taking defaults

Pipecat's frame-level interrupt and LiveKit's unified session both have an opinionated default for cancel-on-interrupt. Vocode exposes the hooks and has reasonable defaults, but you spend more time choosing endpointing settings and tuning interrupt behaviour yourself. mod-106 tells you *what* to configure; Pipecat / LiveKit *have* those configurations in-framework; Vocode exposes the knobs and expects you to turn them.

### Smaller community and less deployment infra

There is no "Vocode Cloud" in the same sense as LiveKit Cloud. Daily markets Pipecat Cloud separately. Vocode has a hosted offering ([Vocode Cloud](https://vocode.dev/)) with its own feature set; evaluate it against your requirements the same way you'd evaluate any managed platform (Chapter 4). The point is that Vocode-the-library's deployment story is "run this Python process", not "we also ship a managed runtime of the same thing".

## When Vocode is the right call

- You are building a **learning-focused** prototype and want a one-file, minimal-ceremony stack you can read end to end (exercise-03 is precisely this).
- You need a **custom `Agent`** that isn't an LLM — a scripted dialog, a bespoke pathway engine, a deterministic voice menu — and you want the framework to stay out of the way.
- Your voice feature is **small, bounded, and inside a larger app** where adopting Pipecat or LiveKit would be a big framework commitment for a small benefit.
- You're working a bake-off and want a quick point of comparison alongside your Pipecat and LiveKit Agents prototypes.

## When it isn't

- You're building a **production**, multi-sub-agent, long-running, observable voice-agent product. Pick Pipecat or LiveKit Agents (or a managed platform).
- You need **first-class realtime speech-to-speech** support as a drop-in option.
- You need **Node.js** as a runtime.
- You need **broad WebRTC transport support** and tight integration with a media server you control.

## Compare points, side by side

| Concern | Pipecat | LiveKit Agents | Vocode |
|---|---|---|---|
| Organising abstraction | Processor pipeline of frames | Worker → session → Agent | Fixed 4-component conversation |
| Runtime languages | Python | Python + Node/TS | Python |
| Realtime (speech-to-speech) first-class | Yes | Yes | Supported; not the primary idiom |
| Transports | Daily / LiveKit / Twilio / WebSocket / local | LiveKit (native) | Twilio / Vonage / local / WebSocket |
| Managed runtime | Pipecat Cloud (separate) | LiveKit Cloud | Vocode Cloud |
| Extensibility shape | Insert a processor anywhere | Lifecycle hooks + tools | Subclass one of four components |
| Fit for a small, bounded voice feature | Overkill | Overkill | Natural fit |
| Fit for a multi-agent, multi-transport production | Natural fit | Natural fit | Strained |

This is a compare table, not a scorecard. Vocode "losing" more rows is not a verdict — it is the price of being minimal. You want it in your toolbox precisely because the other two aren't.

## Summary

- Vocode models the agent as a fixed **4-component conversation**: input device, transcriber, agent, synthesiser, output device. Simple, explicit, and the smallest readable example of the mod-102 pipeline in library form.
- You gain a short mental loop and easy custom agents; you give up breadth of integrations, in-framework interrupt/middleware primitives, broader transport coverage, and community size.
- Use Vocode for **learning**, **custom non-LLM agents**, and **small embedded voice features**; reach for Pipecat or LiveKit Agents for production.
- Keeping all three in your toolbox means you can match the framework to the shape of the problem, instead of defending your favourite one against every problem.

Next chapter: managed voice-agent platforms (Retell, Vapi, Bland, Hamming, Phonely, Observe.AI) — the fourth option, where the orchestration layer isn't code you write at all.
