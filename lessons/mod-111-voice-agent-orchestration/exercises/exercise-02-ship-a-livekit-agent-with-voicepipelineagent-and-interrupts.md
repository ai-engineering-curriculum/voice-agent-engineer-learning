# exercise-02: Ship a LiveKit Agent with VoicePipelineAgent (AgentSession) and Interrupts

**Estimated effort:** 3 hours · **Output:** a working LiveKit Agents worker (cascaded + realtime variants) with explicit interrupt instrumentation

## Objective

Rebuild the pipeline from exercise-01 on [LiveKit Agents](https://docs.livekit.io/agents/). Produce a cascaded `AgentSession` (VAD + STT + LLM + TTS) and a realtime `AgentSession` (OpenAI Realtime or Gemini Live). Wire the lifecycle hooks (pre-connect, `on_enter`, `on_user_turn_completed`, tool callbacks, interrupt hooks). Measure the end-to-end latency and *in particular* the interrupt tail-stop latency — the time from `user_speech_started` to the last audio frame delivered to the client. Deploy to LiveKit Cloud *or* self-host; your choice.

## Prerequisites

- [Chapter 2 — LiveKit Agents: Entrypoints, Sessions, and the Lifecycle](../02-livekit-agents-sessions-and-lifecycle.md).
- Completed exercise-01 so you have a reference pipeline to compare against.
- LiveKit project set up — self-hosted `livekit-server` on your machine, or a LiveKit Cloud project.
- Python 3.11+ (or Node 20+ if you want to run the Node runtime).
- API keys for the same STT/LLM/TTS vendors from exercise-01.

## Steps

1. **Set up the worker skeleton.** Scaffold an agent process per the current [LiveKit Agents quickstart](https://docs.livekit.io/agents/). Wire `cli.run_app(WorkerOptions(entrypoint_fnc=entrypoint))`. Confirm the worker registers with LiveKit and shows up in the dashboard / server logs.

2. **Build the cascaded `AgentSession`.** VAD (`silero.VAD.load()`), STT (Deepgram or similar), LLM (OpenAI/Anthropic), TTS (Cartesia/ElevenLabs). Wire a trivial `Agent` subclass with a short, voice-first system prompt. Verify by joining the room from a browser client (LiveKit's example web client is fine) and conversing.

3. **Implement lifecycle hooks.**
   - **Pre-connect**: read `ctx.job.metadata` for session variables (customer id, scenario tag); log them.
   - **`on_enter`**: speak a short greeting, scenario-appropriate.
   - **`on_user_turn_completed`**: log the final user transcript to your observability stack.
   - **Tool callback**: register a trivial tool (`get_current_time()` or `lookup_demo_order(id)`) so you can exercise the tool-dispatch path. Chapter 6 is where you build this out — for now, just prove the plumbing.
   - **Interrupt hook**: subscribe to the interrupt event and log the transition from THINKING / STREAMING_OUT → INTERRUPTED (Chapter 5).

4. **Build the realtime variant.** A second entrypoint (or a feature-flagged variant of the first) using `openai.realtime.RealtimeModel` or the Gemini Live equivalent. Keep the `Agent` subclass the same if possible. Confirm it runs; note which framework calls still apply when the STT/TTS aren't in the session.

5. **Measure end-to-end latency.** Instrument spans around each leg as in exercise-01, with OTel. Record p50 and p95 end-to-end turn gap for both variants on 20 scripted turns.

6. **Measure interrupt tail-stop latency.** Scripted test: the agent is 2 seconds into a 5-second response; the client plays a pre-recorded "stop" at a known time. Measure the wall-clock gap between the client-recorded "stop" audio and the last bot audio frame heard by the client. Repeat 10 times. Report p50 and p95. This is the metric from mod-106 Chapter 6 made real.

7. **Deploy.** Either (a) run the worker on LiveKit Cloud via their container deployment, or (b) run the worker on a VM / container in a region near your LiveKit server, as a self-hosted deployment. Confirm dispatch rules route rooms to your worker.

8. **Write a short deployment note.** A page covering: the deployment target you picked and why, the dispatch-rule configuration, the per-leg latency, the interrupt tail-stop latency, and any lifecycle hook that behaved differently from what you expected.

## Starter guidance

- **Verify class names against current docs.** The framework consolidated `VoicePipelineAgent` + `MultimodalAgent` into a unified `AgentSession` and class names have moved. Don't fight an old tutorial.
- **Use the LiveKit example web client for testing.** It's a known-good peer; start there before writing your own front end.
- **Add logging on *every* lifecycle hook, not just the one you think you'll use.** When you debug the interrupt flow, you'll want the full event stream.
- **Make the cascaded and realtime variants side-by-side, not sequential.** Running them as two workers with different dispatch rules (match on room name prefix) lets you compare the exact same traffic.
- **Measure in the same region as your worker.** Cross-region tests will tell you a region story, not an orchestration story.

## Acceptance criteria

- [ ] A LiveKit Agents worker that joins a room as a participant, handles a full conversational loop, and emits lifecycle logs for pre-connect, `on_enter`, `on_user_turn_completed`, tool callbacks, and interrupts.
- [ ] A cascaded variant (VAD + STT + LLM + TTS plugins).
- [ ] A realtime variant (OpenAI Realtime or Gemini Live).
- [ ] Per-leg OTel spans with the GenAI semantic conventions.
- [ ] p50 and p95 end-to-end latency over 20 scripted turns, for both variants.
- [ ] p50 and p95 **interrupt tail-stop latency** over 10 scripted barge-ins, for both variants.
- [ ] Deployed to LiveKit Cloud *or* self-hosted, with working dispatch rules.
- [ ] One-page deployment note, dated.

## Stretch goals

- **Node runtime.** Rebuild the cascaded variant in Node/TypeScript with [`livekit/agents-js`](https://github.com/livekit/agents-js). Compare DX and latency against the Python version.
- **Multi-tenant dispatch.** Add room-metadata-based dispatch (`tenant=acme` routes to worker A, `tenant=widgetco` routes to worker B). Prove it with two parallel rooms.
- **Warm-handoff hook.** Add a second `Agent` subclass ("specialist"). On a tool-call-triggered handoff, switch the active Agent and inspect what the framework carries over automatically. Exercise-07 builds this out.
- **Direct comparison to exercise-01.** Put the two measurement tables side by side. Which framework was faster on your traffic? Which was easier to instrument? Which was easier to interrupt-correctly? Capture the observations.

## Keep it

The LiveKit worker is a candidate production shape. If you pick LiveKit Agents for your project-101 capstone, this exercise is the foundation.
