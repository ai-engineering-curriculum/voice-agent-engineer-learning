# exercise-06: Streaming Tool Calls and Progress Fillers

**Estimated effort:** 2 hours · **Output:** parallel-tool-dispatch + progress-filler extensions to your pipeline, with measurements before and after

## Objective

Extend the Pipecat or LiveKit Agents pipeline from exercise-01 / exercise-02 so that tool calls do not block the audio turn. Implement parallel tool dispatch for independent tools. Implement the progress-filler pattern for tools classified as "slow". Measure the user-perceived latency before and after. Optionally wire an MCP client for a shared tool catalogue.

## Prerequisites

- [Chapter 6 — Streaming Tool Calls and Progress Fillers](../06-streaming-tool-calls-and-progress-fillers.md).
- The state machine from exercise-05 — its tool-leg states are directly used here.
- A working pipeline from exercise-01 or exercise-02.

## Steps

1. **Define a small tool catalogue.** At least four tools. Mix shapes:
   - `get_current_time()` — fast, idempotent read.
   - `lookup_order(order_id)` — fast, idempotent read, 100–300 ms simulated.
   - `price_estimate(product_id, options)` — slow, idempotent read, 1500 ms simulated.
   - `book_appointment(slot)` — slow, side-effecting, 2000 ms simulated.
   Record for each tool the p50 latency, the classification (fast/slow), and the idempotency class (idempotent_read/side_effecting).

2. **Implement sequential baseline.** First wire all four tools with naive sequential dispatch. Measure p95 user-perceived latency on a scripted turn that uses two tools (e.g., `lookup_order` + `price_estimate`).

3. **Implement parallel tool dispatch.** When the LLM emits multiple tool calls in a single assistant message, dispatch them with `asyncio.gather` (or equivalent). Stitch the results back by tool-call id per the provider's convention (OpenAI `tool_calls[].id`, Anthropic `tool_use_id`, Gemini `functionCall` name). Verify your concurrency doesn't accidentally serialise on a shared lock.

4. **Measure parallel vs. sequential.** Same scripted turn. Record p95. The improvement should scale roughly with the slowest tool vs. the sum of tools.

5. **Implement progress fillers.** Pick model-driven *or* orchestrator-driven (Chapter 6). If model-driven: add a system-prompt rule ("if about to call a tool from the slow list, say a short acknowledgement first"). If orchestrator-driven: implement the filler dispatcher that speaks an in-flight filler within ~300 ms of dispatching a slow tool.
   - Build a small filler catalogue keyed on tool name.
   - Confirm the filler does not block the tool dispatch (Chapter 5's mouth vs. tool orthogonality).
   - Confirm the LLM continuation after the tool result does not repeat the filler phrase.

6. **Interrupt during a filler.** Drive a test where the user interrupts while the filler is playing. Verify: the filler stops, the tool is cancelled (if idempotent-read) or marked (if side-effecting), the pipeline returns to IDLE cleanly. This is the state machine from exercise-05 being tested end-to-end.

7. **Measure filler vs. no-filler.** Same 2-tool scripted turn where one tool is slow. Record:
   - Time to *first audio* after the user's utterance (TTFA).
   - Total silent duration during the turn.
   - Subjective "feels like a real conversation" score (1–5) from 3 people who weren't involved in building this.

8. **Optional: MCP tool surface.** If you have an internal tool server (or can spin up a trivial MCP server for the exercise), wire an MCP client into your pipeline. Register MCP-exposed tools with the LLM. Measure the extra latency per tool call. Decide (and document) which tools belong on MCP vs. local.

9. **Write a measurements note.** Half a page: sequential vs. parallel vs. parallel-plus-filler latency table, filler/no-filler subjective score, any interrupt-during-filler issues, and the MCP latency if you did the optional step.

## Starter guidance

- **Classify tools before you code.** The classification *is* the design. If you later discover that `price_estimate` is sometimes 100 ms (cached) and sometimes 3000 ms (uncached), that's a sign it needs two entries or a dynamic classifier, not that the classification idea is wrong.
- **Make the filler non-blocking.** `session.say(filler, wait_until_finished=False)` or equivalent. If you await the filler, you've defeated the pattern — the user hears the filler, then silence, then the response.
- **Keep the filler catalogue small.** 2–3 phrases per tool is enough. Cycle them. Users notice repetition fast on 5-minute calls.
- **Start without MCP.** Add it only after the local tools work. MCP adds a network hop and new error modes; isolate the variables.
- **Don't speak fillers for fast tools.** If you filler a 200-ms tool, you've just made the turn 400 ms *slower*.

## Acceptance criteria

- [ ] A tool catalogue of at least four tools covering the four (fast, slow) × (idempotent-read, side-effecting) corners.
- [ ] Sequential baseline measurement (p95 of a 2-tool turn).
- [ ] Parallel-dispatch implementation with p95 measurement; demonstrated improvement over sequential.
- [ ] Progress-filler implementation (model- or orchestrator-driven); filler-to-content ratio stays below 30% on your test turns.
- [ ] Interrupt-during-filler test with clean recovery.
- [ ] Filler vs. no-filler measurement including subjective rating from 3 non-authors.
- [ ] A half-page measurements note, dated.

## Stretch goals

- **Dynamic filler classification.** Make the classifier runtime-adaptive — if a tool starts missing its latency budget, auto-promote it to the slow list for the session. Record a hysteresis so one slow call doesn't flap the classification.
- **MCP integration.** Full wiring with a real MCP server (internal or a sample from the [MCP registry](https://modelcontextprotocol.io/)), including auth. Document the latency delta and the tools that make sense through MCP.
- **Multi-turn tool caching.** Implement a session-level cache for idempotent-read tools (e.g., `lookup_order(order_id)` result cached for 30 seconds). Measure the cost / latency benefit on multi-turn conversations that reference the same entity.
- **Managed-platform comparison.** Run the same tool-heavy conversation on the Vapi / Retell agent from exercise-04 and see how its progress-filler behaviour compares. What is it doing on a 2-second tool call?

## Keep it

The parallel-dispatch + filler extension is the thing that makes your pipeline feel like a real agent on tool-heavy turns. Keep it on; it's the baseline for project-101.
