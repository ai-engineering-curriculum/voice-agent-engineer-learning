# 05. The Interrupt-Aware State Machine

## Why this chapter

Every orchestration option in the previous chapters has an "interrupts are a primitive" story — Pipecat emits an interrupt frame, LiveKit fires a session callback, Vocode has hooks, managed platforms ship a built-in. The primitive is table stakes. What they don't give you is the **state machine** — the explicit decision, written down, of what the agent is doing in each mode, what event moves it to the next mode, what has to be cancelled and in what order, and what has to be resumed.

If you skip this chapter and rely on the framework's defaults, the agent will almost certainly ship a bug that goes: "the bot stopped mid-sentence, the user asked something, the bot answered the *previous* question while ignoring the new one." That is an interrupt-handling bug. The fix is a state machine, agreed before the code is written, that every lifecycle hook maps to a concrete transition.

## States are not sequential

The first mistake is to model the agent as going through states one at a time: LISTENING, then PROCESSING, then SPEAKING, then back to LISTENING. In the wild, the agent is in several states at once across different sub-agents and different legs of the pipeline.

A more honest picture:

- The **transport leg** is always LISTENING to the user's audio track. It never stops. It has an orthogonal flag — IS_USER_SPEAKING — driven by VAD.
- The **brain leg** is in one of IDLE / THINKING / STREAMING_OUT / INTERRUPTED. THINKING means the LLM is generating but no audio has been sent to the user yet. STREAMING_OUT means TTS audio is in-flight to the user. INTERRUPTED means we've received a cancel event and are tearing down the in-flight generation and audio.
- The **mouth leg** (TTS + transport out) is in one of SILENT / SPEAKING / FLUSHING. FLUSHING is the brief window after a cancel where the TTS buffer is being drained before the agent is truly silent again.
- The **tool leg** (any in-flight tool call) is in one of NONE / RUNNING / RETURNING / CANCELLED. A long-running tool call can outlive a single turn.

The agent's real state is the **cross-product** of these. A stable mental model is: "user speaking + brain streaming out + mouth speaking + tool running" is a valid, common composite state — the user is talking over the bot, the LLM is still generating, the TTS is still playing, and a tool from a prior turn is still running. The orchestration decides what each leg does when a specific event (user_speech_started, llm_done, tool_returned, hangup, transfer) arrives.

## The four machines, drawn out

Writing each leg as its own small state machine keeps the whole system in your head.

### The transport-input machine

Always on while the call is connected. Two substates:

- `TRANSPORT_LISTENING_USER_QUIET` — audio flowing, VAD below threshold.
- `TRANSPORT_LISTENING_USER_SPEAKING` — audio flowing, VAD above threshold.

Transitions: VAD thresholds (`speech_started`, `speech_stopped`) plus, once an end-of-utterance detector fires, an explicit `user_turn_ended` signal that the brain leg subscribes to.

### The brain-leg machine

The brain is one of:

- `BRAIN_IDLE` — no LLM call in flight, waiting for a user turn to end.
- `BRAIN_THINKING` — LLM request sent, no tokens streamed yet (TTFT not reached).
- `BRAIN_STREAMING_OUT` — LLM is streaming tokens; TTS is consuming them.
- `BRAIN_INTERRUPTED` — the brain has received a cancel event; it is draining.
- `BRAIN_FINISHED` — the LLM's final token has been emitted and consumed; the brain is drifting back to idle.

Transitions: `user_turn_ended` moves IDLE → THINKING; first LLM token moves THINKING → STREAMING_OUT; last LLM token or tool-trigger moves STREAMING_OUT → FINISHED; a `user_speech_started_during_brain_active` event moves THINKING / STREAMING_OUT → INTERRUPTED; drain-complete moves INTERRUPTED → IDLE.

### The mouth-leg machine

- `MOUTH_SILENT` — no TTS audio in the output buffer.
- `MOUTH_SPEAKING` — TTS audio is flowing out to the user.
- `MOUTH_FLUSHING` — a cancel arrived; the output buffer is being drained.

Transitions: first TTS audio frame moves SILENT → SPEAKING; last TTS frame or cancel moves SPEAKING → FLUSHING; drain-complete moves FLUSHING → SILENT.

The key subtlety: the mouth leg has its own buffer. When the brain is INTERRUPTED, the mouth may still have audio *already shipped to the client* that is going to play regardless. The interrupt experience the user hears depends on the sum of the server's flush time **and** the network / client's playout buffer. Chapter 6 of mod-106 was about this latency; here, we document which component owns which chunk.

### The tool-leg machine

- `TOOL_NONE` — no tool call in flight.
- `TOOL_RUNNING` — a tool call is executing.
- `TOOL_RETURNING` — the tool returned, its result is being inserted back into the brain context.
- `TOOL_CANCELLED` — the tool was cancelled (because the turn it belonged to was interrupted).

Most tools are short (a database lookup, a quick API call). A few are long (a backend search that takes 2 seconds, a payment authorisation that takes 5 seconds). Long-running tools are the reason this leg has its own state — the user can interrupt, start a new turn, and cause the tool from the prior turn to be irrelevant *or* still needed. See Chapter 6 for the cancel-vs-keep decision and the progress-filler pattern.

## The cancel-and-resume contract

The hardest decision is what cancel *means*, concretely. Spell out a contract:

### On `user_speech_started` while the brain is active:

1. **Immediately** tell the LLM to stop generating. For providers that support streaming cancel (OpenAI, Anthropic, etc.), send the abort signal. For providers that don't, set a local flag that drops incoming tokens.
2. **Immediately** tell the TTS service to stop. For streaming TTS (ElevenLabs, Cartesia), send the cancel. For non-streaming TTS, drop the queued audio.
3. **Flush** the client-bound audio buffer. In a WebRTC transport, this means stopping the output track and sending the latest audio timestamp to the client so it can stop playout.
4. **Decide what to do with in-flight tool calls.** Two defaults, pick one and document which you're choosing:
   - *Cancel aggressively.* Cancel any tool calls initiated as part of the interrupted turn. Safer for side-effecting tools ("book the appointment"). Wasted work for pure-read tools ("look up the customer").
   - *Let them run.* Pure-read tool calls complete; their results are discarded. Side-effecting tool calls complete; their effects are permanent. Riskier for the second case.

The right default is cancel-aggressively-except-for-idempotent-read-tools, which requires tagging each tool at registration with an `idempotent_read` flag. Managed platforms often choose one or the other without exposing the knob; know which one yours does.

5. **Update the conversation context.** The assistant's message was truncated. Record what was actually spoken (not what was generated but not delivered) in the assistant context, plus a marker that the turn was interrupted. This is subtle and matters: on the next turn, the LLM needs to know "I started saying X, got cut off at Y, the user then said Z" — not "I said X-fully and the user then said Z". Pipecat and LiveKit have specific utilities for this; verify against current docs.

6. **Transition** to IDLE on the brain leg once all drains complete.

### On `user_turn_ended` with the brain in IDLE:

1. Send the LLM request with the updated context.
2. Transition to THINKING; start the TTFT clock (mod-109).

### On `tool_returned` during BRAIN_STREAMING_OUT:

1. Pass the tool result back to the LLM as a tool-response message. The LLM continues the turn.
2. If the tool took longer than your progress-filler threshold, you have already emitted a filler phrase (Chapter 6) — the resumed LLM output should not repeat it.

### On `tool_returned` during BRAIN_INTERRUPTED:

1. Discard the tool result if the tool is idempotent-read.
2. If the tool has a side effect, record the effect in the conversation context and inform the user on the next turn ("I did book that appointment, but you interrupted me — did you still want it?"). This is the rare case; most tools are read-only.

## Parallel sub-agents

A production voice agent can decompose into sub-agents — a "router" that routes the turn, a "specialist" that handles a particular intent, an "escalation" sub-agent that handles transfer, and so on. The state machine above applies to each sub-agent independently, but you also need a **parent coordinator** that:

- Tracks which sub-agent currently owns the brain leg.
- Routes transport-level events (user speech, interrupts) to that sub-agent.
- Handles sub-agent handoff: the router finishes, the specialist takes over.
- Serialises mutating operations on shared state (the pinned facts, the conversation context) — two sub-agents editing the context at the same time is a bug source.

A simple pattern: the coordinator holds a single `active_sub_agent_id`. All transport events fan to that sub-agent. The coordinator performs explicit "handoff" transitions when a sub-agent requests one, wiring the next sub-agent into the active slot.

More elaborate patterns exist — e.g., two sub-agents each listening to the same audio and racing to answer, with a tie-breaker — but they are almost always the wrong answer for a voice interface. Keep one owner per turn.

## The event-to-transition table

For a reviewable artefact, the simplest format is a table: event × current state → next state + side effects. A compact starter:

| Event | If brain state is ... | New brain state | Mouth effect | Tool effect |
|---|---|---|---|---|
| `user_turn_ended` | IDLE | THINKING | — | — |
| first LLM token | THINKING | STREAMING_OUT | SILENT → SPEAKING | — |
| last LLM token | STREAMING_OUT | FINISHED → IDLE | SPEAKING → SILENT (natural) | — |
| `user_speech_started` | THINKING | INTERRUPTED | — (nothing to flush yet) | cancel idempotent reads; mark side-effecting tools |
| `user_speech_started` | STREAMING_OUT | INTERRUPTED | SPEAKING → FLUSHING → SILENT | cancel idempotent reads; mark side-effecting tools |
| `user_speech_started` | IDLE | IDLE | — | — |
| `tool_returned` | STREAMING_OUT | STREAMING_OUT | — | RUNNING → RETURNING |
| `tool_returned` | INTERRUPTED (idempotent read) | INTERRUPTED | — | discard |
| `tool_returned` | INTERRUPTED (side-effecting) | INTERRUPTED | — | record + notify on next turn |
| `brain_drain_complete` | INTERRUPTED | IDLE | — | — |
| `handoff_requested` | * | IDLE (on new sub-agent) | — | — |
| `hangup` | * | TERMINATED | — | cancel all |

A table like this, written down and reviewed, is what turns an "interrupts are handled" story into a specification a reviewer can ratify.

## Framework-specific plumbing

Each framework exposes different primitives for the transitions above. A short map (verify names against current docs):

- **Pipecat.** `UserStartedSpeakingFrame` and `InterruptionFrame` propagate upstream and trigger cancellation of downstream processors; `BotStartedSpeakingFrame` / `BotStoppedSpeakingFrame` mark the mouth leg; context aggregators emit updated context after each turn. Your job: subclass or insert a processor at the right point in the pipeline to implement the custom transitions (e.g., tool-side-effect handling).
- **LiveKit Agents.** Session-level callbacks for user speech, agent speech, turn completion, and tool calls. The `Agent` subclass is where the custom cancel-and-resume logic lives.
- **Vocode.** `BaseAgent` hooks for user-started-speaking and bot-finished-speaking; narrower than the two above, so you implement more of the state machine yourself.
- **Managed platforms.** Expose a subset of these as config toggles ("interrupt sensitivity", "tool timeout"). What you can't configure, you don't get to change.

The state machine is the specification; the framework is the implementation surface.

## What goes wrong without one

Three failure modes that are common enough to call out:

- **The apology trap.** The agent gets interrupted, apologises on the next turn ("Sorry, let me try again"), and loses context. Users perceive the agent as both annoying and incompetent. Caused by not updating the conversation context correctly after an interrupt — the LLM is prompted to apologise because it can see it was cut off, but it has no memory of what it was saying.
- **Ghost tool effects.** The user interrupts, the tool call completes and books the appointment anyway, the user never finds out until the calendar notification arrives. Caused by not distinguishing idempotent-read from side-effecting tools in the cancel contract.
- **Half-flushed audio.** The brain cancels, the TTS cancels, but the WebRTC jitter buffer at the client plays out another 300 ms of audio. The user says "cancel", hears "...for you today — ", and concludes the agent doesn't listen. Caused by not flushing at the transport layer in addition to the server-side cancel.

All three are preventable with the state machine in this chapter plus the framework primitives of the chapter it ties into (Pipecat / LiveKit).

## Summary

- Model the agent as **four concurrent state machines** — transport-input, brain, mouth, tool — not one sequential pipeline. The real state is their cross-product.
- Write down the **cancel-and-resume contract** explicitly: what cancels on `user_speech_started`, what updates the context, what happens to in-flight tools.
- Tag each tool as **idempotent read** or **side-effecting** so the cancel path knows what to do with its in-flight result.
- Always **flush at the transport layer** in addition to the server-side TTS cancel; the user hears whatever is in the client buffer regardless of what the brain is doing.
- For multi-sub-agent systems, use a **coordinator with a single active sub-agent** per turn; avoid racing sub-agents.
- The event-to-transition table is the review artefact. Build it before the agent, update it when the agent changes.

Next chapter: streaming tool calls inside a live audio turn — the parallel-tool-call shape, the progress-filler pattern for long-running tools, and MCP as the tool-surface integration path.
