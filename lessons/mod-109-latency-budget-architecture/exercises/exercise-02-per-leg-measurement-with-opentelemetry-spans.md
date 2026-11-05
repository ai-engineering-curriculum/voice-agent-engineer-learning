# exercise-02: Per-Leg Measurement with OpenTelemetry Spans

**Estimated effort:** 3 hours · **Output:** an instrumented voice-agent reference app + a per-turn trace visualisation

## Objective

Take an existing minimal voice-agent (yours from prior modules, or a reference app from `livekit-agents`, `pipecat`, or `vocode`) and instrument it so that **every leg of the turn gap** is attributable in a trace backend. By the end you should be able to open a single trace and read off ASR TTFB, ASR final, LLM TTFT, LLM TPOT, TTS TTFB, and transport RTT without doing arithmetic in your head.

## Prerequisites

- [Chapter 2 — Measuring and Attributing Per-Leg Latency](../02-measuring-and-attributing-per-leg-latency.md).
- A running voice-agent prototype in any framework. If you don't have one, start from `livekit-agents`' minimal pipeline example or `pipecat`'s quickstart.
- An OpenTelemetry-compatible trace backend: Jaeger, Tempo, Honeycomb, Datadog, SigNoz, or any OTLP-capable collector will do. Local Jaeger-in-Docker is sufficient.

## Steps

1. **Define the span hierarchy.** On paper first, list the spans you will emit per turn: a root `turn` span, children for `asr.recognize`, `orchestrate`, `llm.generate`, `tts.synthesize`, and any tool calls. For each child, list the **span events** you will emit at leg boundaries (`audio.first_frame`, `vad.speech_end`, `asr.first_partial`, `asr.final`, `llm.first_token`, `llm.last_token`, `tts.first_byte`, `tts.last_byte`, `playout.first_byte`).

2. **Adopt the GenAI semantic conventions.** On LLM spans, set the required `gen_ai.system`, `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`. Check the current version of the OTel GenAI semantic conventions before committing a specific attribute name for TTFT; the spec is evolving.

3. **Wire up a tracer and an exporter.** Use the official OpenTelemetry SDK for your language. Export via OTLP to your local collector / backend.

4. **Instrument the ASR leg.** Open an `asr.recognize` span when audio starts flowing to ASR; emit `audio.first_frame` as a span event on the first frame carrying speech; emit `asr.first_partial` on the first partial transcript; emit `vad.speech_end` when VAD declares end-of-utterance; emit `asr.final` when the final transcript is returned. Close the span on `asr.final`.

5. **Instrument the orchestration leg.** A short `orchestrate` span wrapping prompt build, retrieval (if any), and tool routing. Inside, use span events or sub-spans for anything that might itself blow a budget (a retrieval call, a policy check).

6. **Instrument the LLM leg.** Open an `llm.generate` span before you send the request; emit `llm.first_token` on the first token received; emit `llm.last_token` on completion; close the span. Set the gen_ai attributes above.

7. **Instrument the TTS leg.** Open a `tts.synthesize` span when the first token flows into the TTS; emit `tts.first_byte` when the first audio chunk comes out; emit `tts.last_byte` on completion; close the span. Explicitly do *not* wait for `llm.last_token` before starting TTS (Chapter 3).

8. **Capture transport RTT.** From WebRTC's `getStats()` (or your WebSocket ping), poll the RTT every 1–5 seconds during the call and set the most recent value as an attribute on each `turn` span (`transport.rtt_ms`). This is per-turn attribution, not per-call.

9. **Add a `turn.id` attribute** to every span in the turn. Set it from the root; propagate via the context so children inherit.

10. **Drive at least five turns of real traffic through the pipeline.** Mix single-turn and multi-turn flows, include at least one barge-in, and at least one long-utterance turn (10+ seconds of input or 50+ tokens of output).

11. **Open a trace in the backend and compute the turn gap.** From a single trace, extract:
    - ASR final − VAD speech end = ASR final latency.
    - TTS first byte − VAD speech end = *turn gap* (approximately, net of the client-side playout delay).
    - TTS first byte − LLM first token = TTS TTFB.
    - LLM first token − ASR final = LLM TTFT + orchestration overhead.
    - (LLM last token − LLM first token) / output tokens = TPOT.

    Record the values. Confirm by eye that the timeline in the trace backend matches your mental model of the pipeline.

12. **Write a short attribution writeup.** Three or four paragraphs: which leg in your pipeline is the largest p95 contributor, which leg has the most variance, and what you would change if you had to cut 150 ms from the turn gap tomorrow.

## Starter guidance

- **Do not use span start / end timestamps** for leg attribution. Use **span events**. Span durations measure the leg's own work; events measure the user-perceptible boundaries.
- Use a **monotonic clock** inside one process; let wall-clock timestamps align across processes via NTP.
- If you can, run the whole thing in Docker Compose with Jaeger — the trace waterfall is easier to read than any spreadsheet.
- OTel's Python / Node / Go SDKs each have idiomatic patterns for propagating context into async streams; follow the SDK's docs rather than threading context manually.
- If you're using `livekit-agents` or `pipecat`, both frameworks have patterns for OTel integration — adopt those rather than writing a parallel tracer.

## Acceptance criteria

- [ ] A written span-hierarchy design with the events listed above.
- [ ] Code emitting `turn`, `asr.recognize`, `orchestrate`, `llm.generate`, `tts.synthesize` spans with the correct parent/child relationships.
- [ ] Span events at `audio.first_frame`, `vad.speech_end`, `asr.first_partial`, `asr.final`, `llm.first_token`, `llm.last_token`, `tts.first_byte`, `tts.last_byte`, `playout.first_byte`.
- [ ] Gen_ai semantic-convention attributes set on the LLM span.
- [ ] Transport RTT polled from WebRTC `getStats()` or a WebSocket equivalent and attached to the turn span.
- [ ] Traces visible in a backend; one trace exported as a screenshot or JSON for review.
- [ ] A derived per-turn metric table for at least five turns: ASR final, LLM TTFT, LLM TPOT, TTS TTFB, transport RTT, turn gap.
- [ ] A short attribution writeup naming the biggest contributor and biggest-variance leg.

## Stretch goals

- **Compare two providers.** Repeat the measurement with a second LLM or TTS provider. Report the p95 delta per leg and the change in turn gap.
- **Dashboards, not just traces.** Build a per-leg p50/p95/p99 dashboard on top of your trace data. Pin it next to the trace waterfall.
- **Clock-skew probe.** Add a synthetic ping that measures client-server wall-clock skew at session start, log the number, and factor it into your turn-gap derivation.
- **Attribute cold vs. warm connections.** Tag the first turn of each call separately; measure how much the TTFT improves on warm connections.

## Keep it

The instrumentation is a prerequisite for exercises 03, 05, and for the project-101 end-to-end capstone. Leave it on in your prototype; the data gets richer the longer you leave it running.
