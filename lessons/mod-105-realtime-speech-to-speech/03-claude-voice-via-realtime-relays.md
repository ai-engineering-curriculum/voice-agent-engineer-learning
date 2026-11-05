# 03. Claude for Voice: Realtime Relays, Partner Stacks, and a Swappable LLM Leg

## Why this chapter

Chapters 1 and 2 taught you two first-party speech-native sockets — OpenAI's Realtime API and Google's Gemini Live. The third frontier lab, Anthropic, does not ship a first-party realtime audio WebSocket at the time of writing. Claude is a text-in / text-out model with excellent reasoning, strong tool use, and the lowest refusal cost in many compliance-sensitive domains — but you cannot hand it a PCM stream and get PCM back. If you want Claude on the LLM leg of a voice agent, you wire it into one of the **realtime-relay stacks** that bolt an STT leg, a TTS leg, and barge-in orchestration around a text LLM.

This is not a limitation of Claude; it is a pipeline-shape decision. For many deployments it is the right one, because the trade-off that buys you Claude's reasoning quality also buys you logging, auditability, deterministic voice selection, and the freedom to pick each leg's vendor independently. The engineering lesson of this chapter is **design the pipeline so the LLM leg is a config change** — Claude today, GPT tomorrow when a Realtime feature matters more, Gemini for the camera-aware feature, with no rewrite.

## The current Anthropic stance

Verify against the live docs at integration time; the stance is evolving and this curriculum will lag a point release. As of this writing:

- **No first-party realtime audio WebSocket from Anthropic.** The [Anthropic API](https://docs.anthropic.com/en/api/) is a messages / text + tool-use surface. Claude accepts text, images, and documents as input; it does not accept audio input or emit audio output in the realtime sense.
- **Audio input is available in some batch contexts** (file uploads, non-realtime transcription workflows) depending on the model and SDK release. Treat this as a "batch understanding" affordance, not a voice-agent affordance.
- **Anthropic publishes partner integrations for voice** — the recommended path to Claude-for-voice is through a relay stack that owns the STT + TTS + transport + barge-in surface, with Claude called as the text LLM leg.

The practical consequence is that **"use Claude for voice" means "pick a realtime-relay stack and configure Claude as its LLM leg."** The three stacks you need to know by name are the ones Anthropic itself references in its partner docs plus the broader voice-agent ecosystem:

1. **Twilio ConversationRelay** (telephony-native, hosted).
2. **LiveKit Agents** with its Anthropic plugin (WebRTC-first, open-source + hosted).
3. **Pipecat** with its Anthropic plugin (open-source framework, bring-your-own-deployment).

<!-- needs-research: confirm Anthropic's current recommended voice-partner list at author time; the roster moves -->

## The realtime-relay pattern

A realtime relay is a service that owns the audio socket and orchestrates a pipeline behind it. Concretely:

```
┌────────┐   audio  ┌──────────┐   text   ┌──────┐   text   ┌──────────┐   audio   ┌────────┐
│  user  │ ───────▶ │   STT    │ ───────▶ │  LLM │ ───────▶ │   TTS    │ ────────▶ │  user  │
│ (PSTN  │          │ (stream) │          │(text)│          │ (stream) │           │ (PSTN  │
│ /WebRTC│ ◀─────── └──────────┘          │      │          └──────────┘ ◀──────── │ /WebRTC│
│)       │   audio                        │      │                       audio     │)       │
└────────┘                                └──────┘                                 └────────┘
     ▲                                        │                                         ▲
     │  barge-in event                        │ function calls / tool use               │
     └────────────────────────────────────────┴─────────────────────────────────────────┘
                              orchestrated by the relay
```

The relay owns:

- The user-facing audio transport (SIP / RTP for PSTN; WebRTC / SFU for in-app).
- The streaming STT contract (Module 103) and its endpointing.
- The LLM call — a text prompt that includes conversation history plus the fresh STT transcript, with tool schemas attached.
- The streaming TTS contract (Module 104) and the player pipeline.
- The barge-in choreography (Module 106) — when the user starts speaking, cancel in-flight TTS, truncate history, hand the relayed audio back to STT.

Your job on top of the relay is to:

- Author the system prompt and tool schemas.
- Implement the tools your agent needs.
- Set the voice, model, and endpointing config.
- Wire logging, metrics, and the safety mitigations (Module 108).

Everything about barge-in, audio chunking, reconnection, and jitter is the relay's problem. This is the trade — you give up the end-to-end reasoning-to-speech of a speech-native model, and you buy back explicit access to every text token the LLM saw and every text token it produced. For regulated deployments, that visibility is often non-negotiable.

## The three stacks, briefly

### Twilio ConversationRelay

- **Docs:** [twilio.com/docs/voice/twiml/connect/conversationrelay](https://www.twilio.com/docs/voice/twiml/connect/conversationrelay).
- **What it is:** a Twilio-hosted voice-agent runtime. The ingress is PSTN (via Twilio's SIP + voice infra); the egress is Twilio's audio playback over the call leg. The LLM leg is configured by URL — Twilio opens a WebSocket to your service with transcripts and tool-call affordances, you reply with text and tool-call requests; Twilio handles STT, TTS (via its own or partner TTS), and barge-in on the audio side.
- **Why it matters here:** zero infrastructure for PSTN ingress; the fastest path from an Anthropic text prompt to a working inbound/outbound voice agent on a real phone number.
- **Trade-off:** Twilio owns the STT and TTS vendor relationships; you configure which, but you do not run them.

### LiveKit Agents

- **Docs:** [docs.livekit.io/agents](https://docs.livekit.io/agents/). Anthropic plugin: the [`livekit-agents` package](https://github.com/livekit/agents) with an `anthropic` provider for the LLM leg.
- **What it is:** an open-source voice-agent framework built on LiveKit's WebRTC SFU. Supports STT, LLM, and TTS as pluggable providers; the agent is a Python process that joins a LiveKit room as a "participant," listens to the user's audio track, and publishes its own.
- **Why it matters here:** WebRTC-first, so it is the natural pick for in-browser / in-app voice; the plugin model means you can combine Deepgram STT + Claude LLM + ElevenLabs TTS in a one-line config; it also supports a PSTN gateway for telephony ingress; it ships barge-in and turn-taking primitives.
- **Trade-off:** you run the agent process (or use LiveKit Cloud); the operational surface is larger than ConversationRelay's.

### Pipecat

- **Docs:** [docs.pipecat.ai](https://docs.pipecat.ai/). Anthropic service: [Pipecat Anthropic LLM service](https://docs.pipecat.ai/server/services/llm/anthropic) in the `pipecat-ai` package.
- **What it is:** an open-source Python framework for building voice-agent pipelines as a composition of "frame processors" — each piece of the pipeline (VAD, STT, LLM, TTS, playback, transport) is a node that produces and consumes audio or text frames.
- **Why it matters here:** the frame-level composition makes it the most inspectable of the three — you can log every frame, replay them, and swap any single node without touching the others. The Anthropic plugin is the LLM leg.
- **Trade-off:** more plumbing than the hosted options; you own the deployment.

The three are not strictly interchangeable. ConversationRelay is the PSTN-first managed pick; LiveKit Agents is the WebRTC-native framework pick; Pipecat is the maximum-flexibility bring-your-own pick. Matrix them against the Module 107 transport decision — the relay choice is often implied by the transport choice.

## Design the LLM leg as swappable

The architectural win of this chapter is that **all three stacks use text as the lingua franca on the LLM leg**. That lets you abstract the LLM behind a thin interface and swap Claude ⇄ GPT ⇄ Gemini as a config change, not a rewrite.

What swappable looks like, concretely:

- **One interface, many providers.** A single function the pipeline calls:
  ```python
  reply_text, tool_calls = await llm.complete(
      system=SYSTEM_PROMPT,
      messages=conversation,         # [{role, content}, …]
      tools=TOOL_SCHEMAS,
      temperature=0.7,
      stream=True,                   # yields text deltas
  )
  ```
  Each provider (Anthropic Messages API, OpenAI Chat Completions, Gemini `generateContent`) gets an adapter that implements this interface. All three stacks above are built around this abstraction already — LiveKit's `LLM` base class and Pipecat's `LLMService` are the canonical examples.

- **Tool schemas in JSON Schema, tool_use responses in a canonical shape.** All three providers use JSON-Schema-shaped tool declarations. Keep them in one source of truth; serialise to each provider's dialect inside the adapter. Normalise tool-call events to a shared shape so the orchestration layer doesn't care which provider called the tool.

- **Prompt caching per provider, not per pipeline.** Anthropic's prompt caching uses `cache_control` markers on specific message blocks; OpenAI's is automatic prefix-based; Gemini's has its own idiom. Each adapter owns the caching dialect; the pipeline just hands over `system + messages + tools` and trusts the adapter to cache correctly.

- **Latency parity, not perfect portability.** Model families differ in TTFB and tokens-per-second. When you swap Claude for GPT, verify the pipeline still hits its latency budget; in some cases the right answer is "Claude for the chatty turn, GPT for the tool-heavy turn," orchestrated by a routing layer above the adapters.

- **Model version pinning.** Every provider has a `model` string; expose it in your config so a canary / rollout can hold a specific version while the next tier moves forward. Anthropic model IDs change on a predictable cadence; see the Anthropic model-migration patterns (`claude-api` skill) for how to shift between 4.6 → 4.7 → 4.8 without surprises. <!-- needs-research: confirm current Claude model IDs and any voice-capable surface at author time -->

The point is not that you will actually swap providers weekly. The point is that the architecture **lets you**, so you can price-shop, feature-shop, and compliance-shop without a quarter of migration work. Teams that ship without this abstraction find themselves locked into whichever provider they prototyped on; the lock is not about code, it is about conversation-history format drift and tool-call shape divergence that quietly spreads through the codebase.

## When Claude-in-a-relay is the right pick

Pick this pattern when any of these are true:

- **You need auditable, inspectable text at every hop.** Regulated industries (finance, healthcare, government) often require that every token in and out of the LLM be logged in plain text. A speech-native stack hides the model's "thinking" inside a single audio channel; a relay stack logs the transcript the LLM saw and the text it produced.
- **You want deterministic, licensed voice selection.** TTS providers (ElevenLabs, Cartesia, Azure CNV, Polly Brand Voice) give you specific voice IDs with licence terms. Speech-native models give you the voices their vendor ships — no clone, no brand voice, no legal certainty about who owns the voice.
- **You need language coverage the speech-native models lack.** Deepgram, Azure, and Google STT together cover languages the Realtime and Live stacks do not; the TTS side has similar coverage breadth.
- **You want Claude's reasoning quality** — long-context tool use, structured outputs, careful refusals on regulated prompts, agentic workflows — more than you want sub-200 ms spoken response latency.
- **Compliance posture.** Anthropic's HIPAA eligibility, BAA availability, and the ability to run on third-party inference (AWS Bedrock, Google Vertex AI) can make Claude the compliant option where a speech-native stack is not.

Do **not** pick this pattern when:

- **Every millisecond of latency matters.** A pipeline (STT 150 ms + LLM 400 ms + TTS 150 ms) is unavoidably slower than a speech-native (300–500 ms total). If your product dies below 400 ms end-to-end, you probably want speech-native on the chatty turns.
- **You need emotional / non-verbal cues.** Speech-native models can hear "the user is crying" or "the user is whispering" directly; a text-mediated pipeline loses those cues in the STT transcript. If your product depends on paralinguistic signal, speech-native is the right call.
- **The deployment is a demo / prototype where speed of build matters more than compliance.** The relay stack is more plumbing; speech-native is faster to stand up.

Chapter 4 is the full decision drill on pipeline vs. speech-native vs. hybrid. This chapter's narrower claim: **when you've decided on a pipeline, Claude-via-relay is a strong default.**

## Hybrid shapes

The two patterns compose. Common hybrid shapes:

- **Chatty turns on speech-native, tool-heavy turns on pipeline.** Route by turn type. The user says "hello" → Realtime or Gemini Live answers in 300 ms. The user says "cancel my order and refund the shipping" → the orchestrator detects a tool-heavy turn, hands off to a Claude-driven pipeline that plans the function calls deterministically, narrates the result. The user does not know the difference; the orchestrator does.
- **Speech-native for in-app, pipeline for PSTN.** WebRTC apps run Gemini Live; phone calls route to ConversationRelay with Claude. Different ingress, different stack, same personality (you share the system prompt and tool schemas).
- **Speech-native for input, pipeline for output.** Rare but useful: use a speech-native model as a "better STT" that captures paralinguistic cues, emit its transcription (plus cue tags) to a text LLM that reasons and emits text, then synthesise with a standalone TTS. You buy the paralinguistic input lane without committing to the speech-native output voice.

Hybrids pay for themselves when the deployment has genuinely different turn shapes. They do not pay for themselves when the deployment is uniform — then pick one pattern cleanly.

## A minimal walkthrough (LiveKit Agents, Python, pseudo-code)

The agent process code shape. See the current [LiveKit Agents docs](https://docs.livekit.io/agents/) for the authoritative API.

```python
from livekit import agents
from livekit.plugins import anthropic, deepgram, elevenlabs, silero

SYSTEM = "You are a concise voice assistant..."

async def entrypoint(ctx: agents.JobContext):
    await ctx.connect()

    agent = agents.VoicePipelineAgent(
        vad=silero.VAD.load(),                    # Module 106
        stt=deepgram.STT(model="nova-3"),         # Module 103
        llm=anthropic.LLM(model="claude-sonnet-4-6"),  # verify current ID
        tts=elevenlabs.TTS(voice="rachel", model="eleven_flash_v2"),  # Module 104
        chat_ctx=agents.ChatContext.default(SYSTEM),
    )

    agent.start(ctx.room)
    await agent.say("Hello, how can I help?")
```

That is the entire LLM-leg wiring. Swap `anthropic.LLM` for `openai.LLM` or `google.LLM`; the rest of the pipeline is unchanged. This is the swappable-LLM-leg claim made concrete.

Pipecat's and ConversationRelay's wiring are different in syntax but identical in architectural shape — a pipeline composed of named components, with the LLM one of them, each a thin adapter around a provider SDK.

## Prompt caching with Claude in a voice loop

Anthropic's prompt caching is the latency and cost lever that matters most for Claude-in-voice. The rules:

- **Explicit `cache_control` markers.** Unlike OpenAI's automatic prefix cache, Anthropic's cache is opt-in per message block. Mark the system prompt, the tool schemas, and any stable context block with `cache_control: {type: "ephemeral"}` (verify current field at integration time).
- **Cache-eligible = stable prefix.** A block is cacheable if its content and position are byte-identical to a previous request. Any per-turn mutation (a timestamp, a user name injected into the system prompt) defeats the cache.
- **The running conversation is cached as it grows.** Each new turn appends to the conversation; the previous turns are cached as a prefix.
- **Measure the cache-hit rate.** The Anthropic response includes `cache_read_input_tokens` and `cache_creation_input_tokens` — log these per call. A cache-hit rate under 80% in a long voice session means your prefix is leaking variables.

See the `claude-api` skill for the detailed cache ergonomics and the migration patterns between Claude versions; this chapter's level of detail is enough to design the pipeline, not to tune it.

## Common anti-patterns

- **Treating "Anthropic has no realtime API" as a blocker.** It is a design input, not a wall. The relay stacks are production-grade.
- **Hardcoding `anthropic.Anthropic()` into pipeline code.** You've just baked in the lock-in you were trying to avoid. Always go through the adapter.
- **Ignoring prompt caching on the Anthropic leg.** Without it, a long voice session pays full input cost every turn — latency and bill both suffer.
- **Expecting speech-native latency from a pipeline.** The honest TTFB floor on a well-tuned STT → LLM → TTS pipeline is roughly STT 150 ms + LLM first-token 300–500 ms + TTS 150 ms = 600–800 ms. Below that you are optimising physics; above that you should re-measure.
- **Relying on STT for paralinguistic signal.** STT returns words. "The user is crying" is not in the transcript. If your product needs that signal and you've picked a pipeline, you need a separate affect / prosody model.
- **Pinning to the frontier Claude variant on every turn.** The latency penalty is real. For short, chatty turns, Haiku is often the right pick; use Sonnet or Opus on turns where the reasoning quality matters.

## Summary

- Claude has no first-party realtime audio socket. You use it for voice by picking a realtime-relay stack and wiring Claude as the LLM leg: Twilio ConversationRelay (PSTN-first managed), LiveKit Agents (WebRTC-first open-source), or Pipecat (framework-maximum bring-your-own).
- All three stacks use text as the LLM's wire format. Design the LLM leg as an adapter interface so you can swap Claude ⇄ GPT ⇄ Gemini without rewriting the pipeline.
- Prefer this shape when audit / inspectability / voice-IP / language-coverage / Claude-reasoning-quality matter more than the lowest achievable spoken-response latency.
- Hybrid patterns (speech-native for chatty, pipeline for tool-heavy; speech-native for in-app, pipeline for PSTN; etc.) compose the two cleanly when the deployment genuinely has different turn shapes.
- Use Anthropic's prompt caching deliberately. Mark the stable prefix; keep per-turn variables out of it; measure the cache-hit rate.

Next chapter: the decision drill itself — pipeline vs. speech-native vs. hybrid — scored against latency, cost, interrupt behaviour, voice selection, logging, safety, and language coverage.
