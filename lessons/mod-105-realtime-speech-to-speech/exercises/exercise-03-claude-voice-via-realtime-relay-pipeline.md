# exercise-03: Claude Voice via a Realtime-Relay Pipeline

**Estimated effort:** 3 hours · **Output:** a working voice-agent pipeline with Claude on the LLM leg, a one-page write-up of the LLM-leg swap abstraction, and a swap proof — a diff that migrates the same pipeline from Claude to GPT (or Gemini) in config, not code.

## Objective

Build a voice agent with **Claude as the text LLM leg** of a pipeline (STT → Claude → TTS), through one of the realtime-relay stacks. Make the LLM leg a thin adapter interface; prove the design by swapping it to another provider with a config change. By the end you will have:

- A working, end-to-end voice agent you can talk to.
- The LLM leg expressed as a pluggable adapter with Claude today and at least one other provider wired behind the same interface.
- A write-up explaining what the adapter has to normalise (tool-call shape, prompt-caching, conversation history, streaming deltas) so the swap is clean.

This is the exercise that proves Chapter 3's core architectural claim: swapping the LLM leg can be a config change. If you cannot make it a config change, you have not finished.

## Prerequisites

- [Chapter 3 — Claude for voice: realtime relays, partner stacks, and a swappable LLM leg](../03-claude-voice-via-realtime-relays.md).
- An Anthropic API key, plus one of: an OpenAI API key, a Gemini API key (for the swap half).
- One STT and one TTS account you already have working from Modules 103 and 104 — Deepgram + ElevenLabs is a reasonable default.
- One of the three relay stacks:
  - **LiveKit Agents** (`pip install livekit-agents[all]` and a local LiveKit SFU, or LiveKit Cloud sandbox).
  - **Pipecat** (`pip install pipecat-ai`) with a WebRTC or local-audio transport.
  - **Twilio ConversationRelay** (requires a Twilio account and a public URL, e.g., via `ngrok`). Only pick this if PSTN is actually the ingress you care about; it is the slowest path to a demo.

Use your existing comfort. For this exercise, LiveKit Agents or Pipecat is the recommended path.

## Requirements

Build one program that stands up the pipeline and demonstrates the swap.

### Core pipeline

1. **Transport.** Pick WebRTC (browser → agent) or a local audio I/O (mic + speakers through the framework's debug transport). Document the pick at the top of the write-up.

2. **STT leg.** Deepgram Nova-3 streaming with voice-agent-shaped settings (endpointing + interim results + keyterms). Do not use a batch STT; this exercise requires streaming.

3. **LLM leg — the adapter.** Define an interface:
   ```python
   class LLMAdapter(Protocol):
       async def stream(
           self,
           system: str,
           messages: list[dict],
           tools: list[dict],
           temperature: float,
       ) -> AsyncIterator[LLMEvent]: ...
   ```
   where `LLMEvent` is a normalised event type covering: `text_delta`, `tool_call_start`, `tool_call_delta`, `tool_call_done`, `end`, `error`. Keep the normalisation as the point — no provider-specific shapes leak out.

4. **Claude adapter.** Implement the adapter against the Anthropic Messages API (streaming). Include:
   - `cache_control` on the system prompt and tool schemas (prompt caching is the latency win).
   - Streaming text deltas.
   - `tool_use` events normalised to the adapter's event shape.
   - Model version pinned in config (e.g., `claude-sonnet-4-6` — verify current; see the `claude-api` skill for the model-migration pattern).

5. **TTS leg.** ElevenLabs Flash v2 or Cartesia Sonic Turbo streaming. Configure for a specific voice.

6. **Barge-in.** Enable the framework's built-in barge-in. Verify by interrupting the agent mid-sentence and confirming (a) TTS stops within ~300 ms, (b) the conversation history reflects only what the user heard. Log the event timeline.

7. **A simple tool.** `get_current_time` (same as earlier exercises) declared at the orchestration layer, available to whichever LLM is active. Confirm Claude calls it correctly.

8. **Smoke test.** Hold a short conversation: "hello" → "what time is it" → "thanks, goodbye." Confirm the agent sounds coherent.

### The swap

9. **Second adapter.** Implement the same `LLMAdapter` interface for **one** other provider: OpenAI Chat Completions (streaming) or Google Gemini (streaming). Normalise the tool-call events, the streaming text, and the context formatting.

10. **Config-driven selection.** Make the active LLM configurable by a single string in config:
    ```yaml
    llm:
      provider: anthropic           # or openai, or google
      model: claude-sonnet-4-6
      temperature: 0.7
    ```
    Prove that switching providers is a config change — no edits to the pipeline composition, no edits to the tool definitions, no edits to the STT or TTS code.

11. **Diff artefact.** Run the smoke test on both providers. Capture a short conversation transcript from each. Commit the config file as `config.anthropic.yaml` and `config.openai.yaml` (or similar). The diff between them should be only `provider:` and `model:` plus optionally `temperature:`.

### Prompt caching sanity check (Anthropic leg)

12. **Measure.** Run three successive sessions with a long, stable system prompt (≥ 2000 tokens — pad with a persona description and a tool catalogue). Log the Anthropic `usage` fields `cache_read_input_tokens` and `cache_creation_input_tokens` on each response.

13. **Break the cache.** Inject a per-session timestamp into the system prompt. Re-run three sessions; observe the cache miss. Discuss in the write-up.

## Starter guidance

- **Normalise tool-call shapes.** Anthropic uses `tool_use` + `tool_result`; OpenAI uses `tool_calls` with a `function` sub-object; Gemini uses `function_call` / `function_response`. Pick a canonical shape; translate inside each adapter.
- **Streaming deltas are not identical.** Anthropic emits `content_block_delta` events with explicit start/stop markers; OpenAI emits `delta` fragments with a `finish_reason` only at the end. Your normalised event type has to carry enough info for the pipeline to feed TTS in both cases.
- **System-prompt placement differs.** Anthropic takes `system` as a top-level field; OpenAI treats it as a message with `role: "system"`; Gemini uses `system_instruction`. The adapter owns the placement.
- **Cache_control is Anthropic-specific.** OpenAI's cache is automatic; Gemini's has its own idiom. Each adapter owns caching in its dialect.
- **Choose one framework and own it.** Do not split the exercise across two frameworks; the mental overhead will swamp the point.
- **Model IDs rotate.** Verify current Claude, GPT, and Gemini IDs at integration time. Pin versions in the config, not in code.
- **Log normalised events separately from provider events.** You will debug both the normalisation and the pipeline; being able to see "what the pipeline saw" vs. "what the provider sent" is essential.

## Acceptance criteria

- [ ] Working pipeline with STT + Claude + TTS + barge-in, demoable via a short live conversation.
- [ ] `LLMAdapter` interface defined, with normalised event types.
- [ ] Anthropic adapter implemented, with prompt caching on.
- [ ] A second adapter (OpenAI or Gemini) implemented against the same interface.
- [ ] Config-driven swap: `config.anthropic.yaml` vs `config.<other>.yaml` differ only in provider / model / temperature (plus any necessary per-provider credentials). No code changes to switch.
- [ ] Short conversation transcript captured on both providers for the same utterances.
- [ ] Prompt-cache-read tokens logged on three successive Anthropic sessions; cache miss reproduced by breaking the stable prefix.
- [ ] Barge-in event timeline logged for one interrupt (user speech → TTS cut → history truncate).
- [ ] One-page write-up answering:
  - What are the three biggest things the adapter had to normalise?
  - What is the latency delta between Claude and the other provider on your pipeline, measured on five turns each?
  - What does "swap at a config change" actually buy you — give one concrete scenario where you would exercise it in production.
  - Where do the three frameworks (ConversationRelay, LiveKit Agents, Pipecat) differ, from your experience with the one you picked?

## Stretch goals

- **Three-adapter proof.** Wire all three providers (Anthropic + OpenAI + Gemini). Make the config choose; demo all three.
- **Routing layer.** Add a tiny router above the adapter that chooses the provider by heuristic — e.g., "if the user asked a tool-heavy question, route to Claude; else Realtime-shaped turn, route to GPT." Measure the aggregate latency win.
- **Model-version migration.** Use the `claude-api` skill's migration pattern to migrate your Anthropic adapter from one Claude version to the next (e.g., Sonnet 4.6 → 4.7 when the next release ships). Capture the diff; verify prompt caching still hits.
- **Deploy once on each of two frameworks.** Compare their barge-in choreography, their STT-plug surface, their logging hooks.
- **Add a second tool.** One that takes arguments and returns a larger result (e.g., `look_up_order(order_id)` returning 2 KB of JSON). Measure the TTFB impact on the next audio response after the tool returns.

## Non-goals

- Shipping a production UI.
- Implementing STT or TTS yourself; use the vendor you already wired in Modules 103 / 104.
- Covering PSTN unless that is the ingress you actually care about (use WebRTC or local audio for this exercise).
- Implementing a PVC / IVC voice-clone flow (that is Module 104's territory).

## Keep it

The `LLMAdapter` interface and its two adapters are a durable artefact. Every subsequent project that uses one of the realtime-relay stacks can either consume this adapter directly (if the framework's abstractions are thin) or borrow its normalisation. The design — one interface, many providers, config-driven selection — is the architectural invariant Chapter 3 argued for; this exercise is the proof.
