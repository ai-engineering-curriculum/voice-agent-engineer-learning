# 02. LiveKit Agents — Entrypoints, Sessions, and the Lifecycle

## Why this chapter

[LiveKit](https://livekit.io/) started life as an open-source WebRTC stack and is one of the two media servers that the mod-107 transport analysis treats as a serious option. If your transport is already LiveKit, your orchestration layer probably should be too — the handoff between the media server and the agent runtime stops being a cross-process concern and becomes a library call. That is what **LiveKit Agents** gives you.

Unlike Pipecat, LiveKit Agents exposes both a **Python** and a **Node.js/TypeScript** runtime, which matters when the rest of your product is on Node or when you need the agent to share code with a browser client. It also ships as managed infrastructure (**LiveKit Cloud**) for teams who don't want to operate the media server or the agent fleet themselves.

This chapter covers the mental model — entrypoints, rooms, sessions, and the two canonical agent patterns — and the lifecycle hooks you will reach for in exercise-02. We do not re-explain the component choices (STT/TTS/LLM/VAD/realtime); those are the same decisions as in mod-103 through mod-106 and LiveKit has plugins for the same vendors.

## The mental model: entrypoint → room → session → agent

Where Pipecat's organising abstraction is the *pipeline*, LiveKit Agents' is the *session*. The vocabulary is:

- **Worker.** A long-running process (your agent binary) that registers with LiveKit and advertises its capability to take jobs.
- **Job.** A request from LiveKit's server to a worker: "a room needs an agent — handle it". The worker's **dispatch rules** decide which rooms it picks up.
- **Entrypoint.** The function the worker runs for each job. It receives a `JobContext` with the room, the participant list, and the metadata.
- **Room.** The LiveKit room the agent joins as a participant. Audio tracks from users flow in; the agent's audio track flows out.
- **Session.** The agent's per-call state object (an `AgentSession` in current Python; `multimodal.MultimodalAgent` / `pipeline.VoicePipelineAgent` in older APIs). It owns the STT/LLM/TTS plugins, the turn detection, the conversation state, and the event callbacks.
- **Agent.** The behaviour class attached to the session: instructions, tools, lifecycle hooks (`on_enter`, `on_message`, `on_user_turn_completed`, etc.).

Check the [current LiveKit Agents docs](https://docs.livekit.io/agents/) for the exact class names at your target date — the framework consolidated the pipeline / multimodal split into a unified `AgentSession` and is still evolving. The mental model is stable; the names move.

## The two canonical patterns

Just like in Pipecat, you either build a **cascaded pipeline** (STT → LLM → TTS) or run a **realtime model** (OpenAI Realtime, Gemini Live) end-to-end. LiveKit calls the first `VoicePipelineAgent` (or the new unified `AgentSession` configured with STT+LLM+TTS plugins) and the second `MultimodalAgent` / `RealtimeAgent` (an `AgentSession` configured with a realtime LLM plugin).

### Cascaded — `VoicePipelineAgent` / `AgentSession` with STT + LLM + TTS

```python
# Illustrative — verify class names against current LiveKit Agents docs.
from livekit.agents import JobContext, AgentSession, Agent, WorkerOptions, cli
from livekit.plugins import deepgram, openai, cartesia, silero


class SupportAgent(Agent):
    def __init__(self):
        super().__init__(
            instructions=(
                "You are a brief, voice-first support agent. "
                "Reply in one or two sentences unless the user asks for detail."
            ),
        )

    async def on_enter(self):
        # Greet the user the moment the agent is in the room.
        await self.session.say("Hi! What can I help with today?")


async def entrypoint(ctx: JobContext):
    await ctx.connect()

    session = AgentSession(
        vad=silero.VAD.load(),
        stt=deepgram.STT(model="nova-3"),
        llm=openai.LLM(model="gpt-4o-mini"),
        tts=cartesia.TTS(voice="248be419-c632-4f23-adf1-5324ed7dbf1d"),
    )

    await session.start(agent=SupportAgent(), room=ctx.room)


if __name__ == "__main__":
    cli.run_app(WorkerOptions(entrypoint_fnc=entrypoint))
```

Reading it as a worker program:

- The worker registers with LiveKit (via `cli.run_app`), receives jobs, and invokes `entrypoint` with a `JobContext` per job.
- `ctx.connect()` joins the room as a participant.
- The `AgentSession` wires the plugins (VAD, STT, LLM, TTS). The plugins are drop-in — swapping `deepgram.STT` for `assemblyai.STT` is a one-line change.
- The `Agent` subclass carries behaviour: the system instructions, any `function_tool`s you register, and lifecycle callbacks like `on_enter`.
- `session.start(...)` starts the audio loop and the turn-detection state machine.

### Realtime — `MultimodalAgent` / `AgentSession` with `realtime.RealtimeModel`

```python
from livekit.agents import JobContext, AgentSession, Agent, WorkerOptions, cli
from livekit.plugins import openai


class SupportAgent(Agent):
    def __init__(self):
        super().__init__(
            instructions="You are a brief, voice-first support agent.",
        )


async def entrypoint(ctx: JobContext):
    await ctx.connect()

    session = AgentSession(
        llm=openai.realtime.RealtimeModel(
            voice="alloy",
            turn_detection={"type": "server_vad"},
        ),
    )
    await session.start(agent=SupportAgent(), room=ctx.room)


if __name__ == "__main__":
    cli.run_app(WorkerOptions(entrypoint_fnc=entrypoint))
```

The `AgentSession` now has no STT or TTS plugin — the realtime model is the whole brain. Server-side VAD lives inside the realtime service; you can still pass a client-side `vad=` for UX signals like barge-in indicator LEDs but the realtime model owns turn detection.

Gemini Live is the same shape with a different plugin (`google.beta.realtime.RealtimeModel` or similar — again, check current docs).

## The lifecycle hooks you actually write

A production agent is less about the plumbing and more about *what runs at each lifecycle event*. These are the hooks you will spend time in:

### Pre-connect — before `ctx.connect()`

Load per-call state that does not depend on the room: feature flags, session metadata (`ctx.job.metadata` often carries a JSON blob from the dispatch rule — customer ID, campaign ID, prior-conversation summary), warmed-up LLM context from the warm-handoff pipeline (Chapter 7). This is also the right place to *refuse* a job — raising from the entrypoint before you connect is cheap; disconnecting after connect is wasteful.

### Agent `on_enter` — the agent is in the room, ready to speak

Greet the user, play the hold music if you need to warm the LLM, or wait for the user to speak first. Many products get the opening-line UX wrong — the agent speaking first feels pushy in some contexts, mute in others. Match it to the archetype (inbound support bot: greet; outbound sales bot: greet and disclose automated caller per mod-108; concierge waiting for a hand-raise: silent until spoken to).

### `on_user_turn_completed` — the user's turn ended

Pipecat emits a frame; LiveKit emits a callback. This is where you decide whether the LLM should be invoked, whether to pin a new fact (Chapter 7), whether to route to a different sub-agent. Middleware you want to run on every user turn (PII redaction, sentiment check, consent audit) lives here.

### Tool / function callbacks — the model wants to call a tool

The `Agent` base class supports decorated tool methods (`@function_tool` or similar — verify against current docs). These run in the agent's event loop and must not block the audio path. Chapter 6 is dedicated to the streaming-tool-call contract; the LiveKit-specific detail is where the tool runs (in the agent process, not in the LLM).

### Interrupt hooks — the user spoke over the bot

LiveKit's turn-detection and VAD trigger an interrupt event. The session cancels the in-flight LLM/TTS calls (just like Pipecat); you often want to hook this to log the truncation, roll back any side effects that haven't committed yet, and update the assistant's context so the LLM "knows" the user interrupted. Chapter 5 models this as a state transition.

### `on_message` / transcript events — for logging and side-channel UI

User and assistant messages (final transcripts, LLM responses) are emitted as events on the session. You use them to drive the UI (live captions), to stream to your observability stack (mod-112), and to persist to the conversation store. This is not where you should put business logic — business logic belongs in tools and lifecycle hooks — but it is the right place for logging and UI mirroring.

## Dispatch rules: how a room gets an agent

LiveKit doesn't guess which worker handles which room. You configure **dispatch rules** on the server (or with explicit API calls) to route rooms to workers. Common shapes:

- **Match-all.** Every room gets an agent of a given type. Default for a single-product deployment.
- **By metadata.** The room (or the SIP participant) carries metadata; the dispatch rule matches on it. Used for multi-tenant deployments where `tenant=acme` rooms route to the Acme agent variant.
- **By room name pattern.** Workers subscribe to a prefix (`support-*`, `sales-*`, `transfer-*`). Used for routing warm-handoff transfers to a specialist agent (Chapter 7).

The agent code itself is otherwise identical; dispatch is a config-time concern. It belongs in the deployment manifest, not in the agent class.

## Deployment: LiveKit Cloud vs. self-hosted

Both runtime targets exist and the agent code is the same across them.

- **LiveKit Cloud.** Managed media server, managed worker fleet, managed TURN relays. You push your agent (or a container image) and LiveKit handles scaling, autoscale, and the media routing. Right call when you don't want to run WebRTC infrastructure and your data-handling posture (mod-108) is compatible with the Cloud deployment.
- **Self-hosted on your own clusters.** Run `livekit-server` and your worker fleet on your infra. Right call when you need data residency, private peering (mod-109 Chapter 4), or a tighter integration with your existing media / contact-centre stack. The code is identical; the deployment manifests change.

A third pattern is **LiveKit Cloud media + self-hosted workers** — the media server is managed but the agent runs on your compute, connecting as a worker. Useful when the sensitive data (PHI, PCI) is in the agent's conversation state but the media path can reasonably cross a managed boundary. Verify against current LiveKit Cloud networking docs before committing to this pattern.

## Python vs. Node.js — when the runtime matters

LiveKit ships Agents SDKs for both Python and Node/TypeScript ([livekit/agents](https://github.com/livekit/agents) for Python, [livekit/agents-js](https://github.com/livekit/agents-js) for Node). The Python runtime is the reference implementation and tracks features first; the Node runtime exists for teams where Python would be the odd language in the stack.

Pick Node when:

- Your whole backend is Node, and operating a Python worker fleet would be the one Python thing in production.
- The agent needs to share code or types with a React/Next.js client (schema libraries, shared tool definitions, etc.).
- Your DX conventions (lint, deploy, CI) are Node-first and you do not want to duplicate them for one service.

Pick Python when:

- You need the newest plugins the day they ship.
- You're integrating with ML tooling (local Whisper, custom VAD, embeddings) that lives in Python anyway.
- Everyone else on the voice team is already in Python.

Either way, the mental model from this chapter applies.

## When LiveKit Agents is the right call (and when it isn't)

LiveKit Agents is a strong default when:

- Your transport (mod-107) is already LiveKit — self-hosted or Cloud.
- You want **Node.js** as a first-class agent runtime option.
- You want the **unified `AgentSession` model** that treats cascaded and realtime pipelines symmetrically.
- A managed deployment target (LiveKit Cloud) matters to your operational posture.
- You're building browser-first voice apps and want the client/server pieces of LiveKit to come from the same project.

LiveKit Agents is a weaker fit when:

- Your transport is Twilio-PSTN-only and you have no browser or LiveKit footprint — Pipecat's Twilio transport can be a simpler fit.
- You need a provider-agnostic, maximum-flexibility processor pipeline — Pipecat's model is more explicitly composable at the frame level.
- Your product is actually a thin wrapper around a managed voice-agent product (Chapter 4).

## Summary

- The core model is **worker → entrypoint → room → session → agent**: a worker process picks up a job, the entrypoint runs per job, the agent session wires STT/LLM/TTS plugins (or a realtime model), and the Agent subclass holds behaviour.
- Both pipelines are expressible: `AgentSession` with `vad + stt + llm + tts` for cascaded, and `AgentSession` with a `realtime.RealtimeModel` for end-to-end speech-to-speech.
- Production code lives in the **lifecycle hooks**: pre-connect, `on_enter`, `on_user_turn_completed`, tool callbacks, interrupt hooks, and transcript events.
- **Dispatch rules** decide which rooms route to which worker; it's a config concern, not an agent-code concern.
- Deployment targets include **LiveKit Cloud** and **self-hosted**, with the same agent code. Node.js and Python are both first-class runtimes.
- Pick LiveKit Agents when the transport is already LiveKit or when you need Node; pick Pipecat (Chapter 1) when your constraint is maximum vendor flexibility in Python.

Next chapter: Vocode — the minimal compare point that keeps the two heavyweights honest and shows what a lighter-weight custom pipeline looks like.
