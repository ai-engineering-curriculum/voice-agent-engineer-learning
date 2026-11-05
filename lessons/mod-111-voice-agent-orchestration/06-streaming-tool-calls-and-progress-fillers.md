# 06. Streaming Tool Calls and Progress Fillers

## Why this chapter

Tool calling in a text chat is an engineering problem; tool calling in a live audio turn is a *dialogue* problem. The difference is that in a text chat the user is willing to see "Thinking..." for two seconds while the model calls a tool. On a phone call, two seconds of silence after "let me look that up" is an eternity — the user starts repeating themselves, hangs up, or assumes the bot crashed.

The orchestration challenge is to call tools inside a streaming audio turn **without the tools blocking the TTS**. That requires three things: a parallel-tool-call shape (so the model can emit multiple tool calls in one turn and the pipeline can dispatch them concurrently), a progress-filler pattern (so the user hears *something* while a slow tool runs), and an integration path to tool catalogues you don't own (so an enterprise deployment can reuse the tools surface rather than re-registering every API).

## Why synchronous tool calls fail in voice

Let's do the latency math. A user asks "what's the status of my order?" The model generates a tool call `get_order_status(order_id="…")`. In a classical chat flow:

1. Model generates the tool call. TTFT is ~300 ms.
2. Tool runs. Suppose it takes 1500 ms (DB lookup + cross-service call, totally normal).
3. Tool returns. Model generates the next response. TTFT another ~300 ms.
4. TTS kicks in.

Total silence: ~2100 ms minimum. mod-109's conversational-feel target was p95 end-to-end under ~800 ms. You missed by a factor of 2.6. If the tool call is longer (an external API that averages 3000 ms), you miss by a factor of 4+.

Worse, during those 2100 ms the user hears nothing. There is no audio in the pipeline at all. From the user's perspective, it's indistinguishable from a dropped call.

## The two mitigations, used together

### Parallel tool-call shape

Most tool calls a model makes in a single turn are independent — `get_order_status(id="A")` and `get_customer_profile(id="C")` can run concurrently. All three major LLM providers support emitting **multiple tool calls in one assistant message** that your orchestrator is free to dispatch in parallel:

- **OpenAI** — tool-calling returns a `tool_calls` array on the assistant message; dispatch them concurrently; stitch results back by `tool_call_id`. See the [OpenAI function calling guide](https://platform.openai.com/docs/guides/function-calling).
- **Anthropic** — the Messages API supports multiple `tool_use` content blocks in one response; return multiple `tool_result` blocks keyed by `tool_use_id`. See [Anthropic tool use docs](https://docs.claude.com/en/docs/agents-and-tools/tool-use/overview).
- **Google Gemini** — parallel function calling emits multiple `functionCall` parts in one response. See [Gemini function calling](https://ai.google.dev/gemini-api/docs/function-calling).

The orchestrator pattern is: when the LLM emits N tool calls, kick off N concurrent executions, `await asyncio.gather(...)` (or equivalent), feed all results back as the next model message, and resume generation. The user-perceived latency becomes `max(tool_1, tool_2, ..., tool_N)` instead of `sum(...)`.

```python
# Illustrative pseudocode for a Pipecat / LiveKit function-tool handler.
async def handle_tool_calls(tool_calls):
    async def run(tc):
        try:
            result = await TOOLS[tc.name](**tc.arguments)
            return tc.id, {"status": "ok", "result": result}
        except Exception as e:
            return tc.id, {"status": "error", "message": str(e)}

    results = await asyncio.gather(*(run(tc) for tc in tool_calls))
    return dict(results)  # id -> result; feed back into the model
```

Keep the handler **non-blocking**. Any synchronous I/O in there blocks the event loop; the whole pipeline pauses.

### Progress-filler pattern

Parallel-calling doesn't help if the single longest tool is still slower than the user's silence tolerance (~600–1000 ms on a phone call). The mitigation is to *speak a filler* while the tool is running.

Two design choices to make:

#### Who says the filler?

- **The model.** Instruct the model to say a short acknowledgement before issuing a slow tool call: "Let me look that up for you." The TTS speaks this, then the tool call fires. The model produces the real response once the tool returns.
- **The orchestration layer.** The model emits the tool call directly; your orchestrator detects that the tool is in a "slow" category and *injects* a filler audio frame while the tool runs, without the model's involvement.

Each has a cost. Model-driven fillers are simpler to maintain but require the model to actually follow the instruction, which breaks under prompt drift. Orchestrator-driven fillers are robust but you have to classify tools as slow/fast and manage the filler inventory yourself. Pick one per team; a hybrid is fine but needs a documented split.

#### What does the filler say?

- "One moment while I check..." — generic, works, gets annoying if the user hears it every turn.
- "Let me pull up your account..." — specific, more informative, requires the model (or the orchestrator) to know what it's about to do.
- Short filler phrases rotated at random — "Give me a second...", "Just a moment...", "Let me find that..." — reduces the perception of repetitiveness.
- *No* filler if the tool is fast (< ~500 ms) — speaking a filler for a 300-ms tool is worse than silence, because the filler interrupts the natural pacing of the response.

Rules of thumb that work in practice:

- **Classify every tool** as `fast` (< 500 ms p50, no filler needed) or `slow` (speak a filler within 300 ms of tool dispatch).
- **Cap the filler-to-content ratio.** If every turn has a filler, the agent sounds hedging and incompetent. Aim for < 30% of turns.
- **Tie filler phrasing to the tool.** A tool named `lookup_order` gets "Let me pull up your order..."; a tool named `price_estimate` gets "One moment while I calculate that...". Variety here hides the fact that the agent is stalling.
- **Don't filler on tools your users know are fast.** If `check_time` always returns in < 100 ms, never filler it, even if a filler would technically be allowed.

### What about the TTS keeping going?

The subtle technical point: the TTS pipeline must handle a filler phrase as *not the end of the turn*. After the filler audio plays and the tool returns, the LLM continues generating, the TTS has to pick up seamlessly. If your TTS implementation treats each utterance as a fresh session, there's a noticeable gap between the filler and the content. Streaming TTS services (ElevenLabs, Cartesia) support continuation; verify the behaviour on your stack.

## Example — a filler-driven tool call

```python
# Illustrative — exact hooks differ between Pipecat and LiveKit.

FILLER_CATALOG = {
    "lookup_order": ["Let me pull up your order...", "Looking that up now..."],
    "price_estimate": ["One moment while I calculate that..."],
    "transfer_call": ["Connecting you now, one moment..."],
}

async def dispatch_with_filler(tool_calls, session):
    # Separate slow and fast tools.
    slow = [tc for tc in tool_calls if TOOL_SPEC[tc.name].slow]
    fast = [tc for tc in tool_calls if not TOOL_SPEC[tc.name].slow]

    # For each slow tool, speak a filler right now (orchestrator-driven).
    for tc in slow:
        filler = random.choice(FILLER_CATALOG.get(tc.name, ["One moment..."]))
        await session.say(filler, wait_until_finished=False)  # do not block!

    # Run all tools concurrently.
    async def run(tc):
        return tc.id, await TOOLS[tc.name](**tc.arguments)

    results = dict(await asyncio.gather(*(run(tc) for tc in (slow + fast))))
    return results
```

Important detail: `session.say(filler, wait_until_finished=False)` must not block the tool dispatch. The filler speaks on the mouth leg while the tool runs on the tool leg. Chapter 5's state machine was built for exactly this — the mouth is SPEAKING a filler while the tool leg is RUNNING, and these are orthogonal states.

## When the user interrupts a filler

The filler is audio. It can be interrupted like anything else. The relevant state-machine transitions from Chapter 5:

- Filler is playing (MOUTH_SPEAKING) and user barges in → cancel the filler TTS, flush the client, transition BRAIN back to IDLE.
- Tool is still running — cancel it if idempotent-read, else let it complete and record the side effect.

This is exactly the normal interrupt path; the filler doesn't need special-case cancel handling if the state machine is correct.

## MCP for enterprise tool catalogues

A production voice agent often needs access to the same tools as a text chat agent, a desktop assistant, or an in-IDE copilot: the same CRM lookup, the same billing API, the same calendar operations. Registering these tools in each product's codebase is duplication; the cost compounds as the tool surface grows.

The **Model Context Protocol (MCP)** — Anthropic-originated, now a multi-vendor standard — is a way to publish a tool server that any MCP-aware client (Claude Desktop, Cursor, your voice agent) can discover and call. See the [MCP specification](https://modelcontextprotocol.io/).

For a voice agent on Pipecat or LiveKit, the relevant pattern is:

1. **Enterprise publishes an MCP server** that exposes the shared tool catalogue (CRM queries, calendar operations, payments, etc.) with proper auth, audit, and compliance scoping.
2. **The voice agent runs an MCP client** inside the orchestration layer. The client discovers the available tools, fetches their schemas, and registers them with the LLM.
3. **Tool calls route through MCP** — the LLM emits a tool call; the MCP client forwards it to the server; the server returns the result; the client hands it back to the LLM.

The payoff is that the enterprise tool surface is maintained once; the voice agent, text agent, and IDE agent all consume it the same way. The cost is one extra network hop and a new protocol to understand; for a *single-product* voice agent this is unnecessary overhead. For a voice agent that's one of several surfaces onto a shared enterprise AI stack, MCP is the right path.

Specific guidance:

- Use MCP for the **shared enterprise catalogue**, not for product-local tools. A `check_internal_cache` tool that's specific to the voice agent lives in the agent code; a `lookup_customer` tool that every product on the floor needs lives in the MCP server.
- **Latency budget**: an MCP call is one more network hop. Measure the added TTFB; it is typically small (tens of ms to a same-region MCP server) but is real.
- **Auth and audit**: the MCP server is the right chokepoint for per-call authorisation and audit logging. Do not bypass it in the voice agent "for latency" — you will regret it at your next compliance review.
- **Tool classification carries through**: the `fast/slow` and `idempotent_read/side-effecting` tags from the state machine still apply. Decorate them on the MCP server side (if the protocol supports it at your date) or maintain the mapping in the agent config.

## Streaming structured output that isn't a tool call

An adjacent pattern: the model emits a structured response in the middle of a turn — not a tool call, but e.g. a JSON blob the UI has to render while the audio plays. Realtime surfaces (OpenAI Realtime, Gemini Live) support this via side-channel events. The pattern is the same as a tool call in reverse: the audio plays on the mouth leg, the structured event flies to the UI on a data channel, neither blocks the other. State-machine-wise, nothing new; plumbing-wise, you need a separate data channel to the client (WebRTC data channels, LiveKit data messages, or your framework's side-channel).

## Cost and tail-latency considerations

Parallel tool calling cuts user-perceived latency; it does not cut cost. You still pay for N tool calls. For high-volume agents, two observations are worth making explicit:

- **Cache tool results** when the tool call is on a short-lived cache key (customer-id → profile). A 30-second cache saves huge cost at the LLM level because downstream turns of the same call can reuse it without re-dispatching.
- **Set hard timeouts per tool.** A tool that exceeds its timeout should return a structured error ("the lookup took too long") instead of waiting. The LLM can tell the user and move on. Without timeouts, a stuck external service hangs the agent indefinitely.

## Summary

- Synchronous tool calls do not work in a live audio turn — the user-perceived silence breaks the latency budget from mod-109.
- Use **parallel tool calling**: emit multiple tool calls in one assistant turn, dispatch concurrently, stitch results by ID.
- Use the **progress-filler pattern**: a short filler phrase while a slow tool runs, classified per tool as fast or slow, with phrasing tied to the tool's purpose and a cap on filler-to-content ratio.
- Interrupts work normally during a filler — the state machine from Chapter 5 already handles it; the filler is just audio on the mouth leg.
- Integrate **MCP** when the tool surface is shared across products; keep product-local tools in the agent code. Respect the extra hop in the latency budget and use the MCP server as the auth/audit chokepoint.
- Cache tool results on short-lived keys; set hard per-tool timeouts to keep a stuck backend from hanging the whole call.

Next chapter: managing context length and memory across a long call — rolling summaries, pinned facts, and warm-handoff context packaging for the next agent.
