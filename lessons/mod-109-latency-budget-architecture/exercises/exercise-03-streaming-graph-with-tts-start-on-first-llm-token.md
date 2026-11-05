# exercise-03: Streaming Graph with TTS Start on First LLM Token

**Estimated effort:** 3 hours · **Output:** two runs of the same prototype (sequential and streaming), with a measured turn-gap delta

## Objective

Convert a sequentially-wired voice-agent prototype — LLM finishes, then TTS starts — into a streaming graph where **TTS synthesis starts on the first LLM token**, and measure the latency improvement. The exercise is deliberately a *before/after*: the number you report at the end has to be defensible against your own measurement harness from exercise-02.

## Prerequisites

- [Chapter 3 — Streaming Graph for Maximum Parallelism](../03-streaming-graph-for-parallelism.md).
- A voice-agent prototype instrumented from exercise-02. If your prototype is already streaming, downgrade it to sequential for the "before" measurement.
- A streaming LLM API (OpenAI, Anthropic, Gemini, or any OpenAI-compatible endpoint) and a streaming TTS API (ElevenLabs, Cartesia, LMNT, or an equivalent) — do not use non-streaming endpoints; this exercise is pointless without them.

## Steps

### Part A — Baseline sequential run

1. **Confirm the pipeline is sequential.** Verify in code and in your traces that TTS synthesis does not start until the LLM has returned all tokens. Add a log line if the trace isn't enough.

2. **Run 20 turns.** Use a mix of utterance lengths: five short responses (10–20 tokens), ten medium (30–80), five long (100+). Capture the per-turn metrics from exercise-02.

3. **Record the p50 / p95 turn gap.** Attribute the gap per leg — pay particular attention to the sum of `LLM TTFT + (LLM TPOT × N)` relative to the total gap.

### Part B — Streaming graph

4. **Pipe LLM tokens into TTS.** Change the orchestrator so that tokens from the LLM stream are forwarded into the TTS stream as they arrive. Do not wait for `llm.last_token`.

5. **Add a phrase buffer.** Buffer incoming tokens until you reach either a sentence boundary (`.`, `?`, `!`), a secondary boundary (`,`, `;`, `:`), or a hard word-count cap (your choice between 4 and 10 words). Measure how the choice of boundary affects TTS first-byte latency: run 10 turns at each of three boundary settings and record.

6. **Add a speakable-token filter.** If your agent emits tool-call JSON, markdown, or any non-speech content, add a parser that strips it from the stream before TTS. If your agent only emits plain speech responses, skip this step but write a one-paragraph note on what you would do.

7. **Add structured cancellation.** Wire the LLM stream, the TTS stream, and audio playout under a single abort / cancellation scope. Trigger it on simulated barge-in (an event you fire 1.5 s into TTS playback) and verify that (a) TTS audio stops at the client within 200 ms, (b) the LLM stream is cancelled server-side within 500 ms, (c) no audio is played after the cancel.

8. **Run the same 20 turns.** Capture the same per-turn metrics.

### Part C — Analysis

9. **Compare**. Produce a table with columns `metric`, `sequential p50`, `sequential p95`, `streaming p50`, `streaming p95`, `delta`. Metrics: turn gap, ASR final, LLM TTFT, LLM TPOT, TTS TTFB.

10. **Explain the deltas.** For each metric, in one sentence, state why it moved (or didn't). If a leg got worse, say so and explain — overlap-aware pipelines sometimes shift costs rather than eliminate them.

11. **Pick the boundary setting.** From Part B step 5, pick the boundary strategy you would ship and defend it in one paragraph. Trade off TTS first-byte latency against prosody quality — if you have access to a listener (yourself, a colleague), include their subjective quality call in the analysis.

12. **Write the kill-switch for barge-in.** Document, in one short page, the structured-cancellation scope you used, the measured barge-in-to-silence latency, and the caveats (e.g., which provider SDKs don't cancel cleanly and how you worked around it).

## Starter guidance

- **The LLM SDK matters.** Some SDKs yield tokens one-per-event; some chunk them. Measure your own.
- **Your TTS provider's streaming contract matters.** Some require a final "end of input" signal; some stream continuously. RTFM the vendor's streaming docs before wiring the pipeline.
- **Do not benchmark against local TTS without warming it.** A cold TTS model loads for seconds; that is not the number you want in your table.
- **Use the same prompts and the same seeds (where supported) for both runs** so that the LLM output length is comparable.
- **Keep the orchestration overhead small.** If your "before" pipeline had significant orchestration latency, factor it out before attributing deltas to the streaming change.
- Resist adding prompt caching in this exercise — it muddies the result. Save it for a later experiment.

## Acceptance criteria

- [ ] A sequential baseline run of 20 turns with recorded per-turn metrics.
- [ ] A streaming graph implementation where TTS consumes LLM tokens incrementally.
- [ ] A phrase buffer with three documented boundary settings and measured first-byte latency for each.
- [ ] A speakable-token filter (or a written note explaining why it wasn't needed).
- [ ] Structured cancellation on barge-in with measured barge-in-to-silence latency.
- [ ] A comparison table across the two runs with per-metric deltas and one-sentence explanations.
- [ ] A one-page kill-switch writeup documenting the cancellation scope and caveats.

## Stretch goals

- **Prompt caching layered on top.** After the streaming graph is in place, enable prompt caching on your LLM provider and re-run. Attribute the additional delta.
- **Warm pools.** Pre-open LLM and TTS connections at call-connect. Measure the first-turn delta.
- **Tool-call streaming.** If your agent uses tools, exercise a turn where the LLM emits a tool call mid-stream. Verify that your speakable-token filter does not leak the tool call into the audio, and that the subsequent response (after the tool returns) continues the streaming graph correctly.
- **Speculative LLM on partial ASR.** As described in Chapter 3, kick off a speculative LLM call on a high-confidence ASR partial; cancel on mismatch. Measure both the best-case gain (confirmed partial) and the cost (cancelled speculative requests).

## Keep it

This is the workhorse architectural move from Chapter 3. The streaming graph and the kill-switch are reused in project-101 and are examined in project-104's architectural brief. Keep the before/after table — it's the single best evidence that the architecture is doing its job.
