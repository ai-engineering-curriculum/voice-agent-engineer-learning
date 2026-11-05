# 02. Measuring and Attributing Per-Leg Latency

## Why this chapter

A budget is a pre-commitment; measurement is how you keep yourself honest about it. The first time someone says "the bot feels slow" you need to answer "which leg?" in minutes, not days. That means every leg of the pipeline from Chapter 1 has to emit signals you can line up on a single timeline — with a shared trace ID — and attribute every millisecond of the turn gap to exactly one span.

This chapter is the instrumentation backbone for everything that follows.

## The per-leg metrics you actually need

Across the pipeline, a short list of measurements explains almost all of the turn gap:

- **ASR TTFB (time-to-first-partial)** — from first audio frame carrying speech to the ASR's first partial transcript. A proxy for ASR's streaming readiness, not a direct contributor to turn gap, but it lets you distinguish "ASR is slow at partials" from "ASR is slow at finalisation".
- **ASR final latency** — from end-of-utterance (endpoint detected) to final transcript emitted. This *is* a direct contributor to the turn gap and often one of the largest discretionary legs.
- **LLM TTFT (time-to-first-token)** — from prompt sent to first output token received. Vendor-reported, often has a long tail.
- **LLM TPOT (time-per-output-token)** — the per-token cadence after TTFT; drives whether TTS can keep up without starving.
- **TTS TTFB (time-to-first-audio-byte)** — from first LLM token (not final text) to first TTS audio chunk emitted. The critical pivot that converts LLM streaming into playable audio.
- **Network RTT** — round-trip between the client and the media server, from WebRTC stats or your WebSocket ping. Appears twice in the turn gap, once for the trailing audio frames before endpointing and once for the first synthesised audio.
- **Jitter and packet loss** — not directly a latency number, but losses force retransmit or concealment that lengthens the jitter buffer; you must watch both.

A reasonable rule: if a leg doesn't have a p50 / p95 / p99 you can look up in the last hour, it is uninstrumented and you cannot defend the budget.

## WebRTC stats: the transport side

The W3C WebRTC specification exposes a `getStats()` API that returns a dictionary of statistics objects per RTP stream. The ones that matter for latency work:

- **`inbound-rtp` / `outbound-rtp`** — per-stream counters (`packetsReceived`, `packetsLost`, `jitter`, `bytesReceived`, `framesDecoded`). The `jitter` field is the interarrival jitter from RFC 3550.
- **`candidate-pair`** — once ICE has selected a pair, `currentRoundTripTime` gives you the smoothed RTT and `totalRoundTripTime` / `responsesReceived` give you a reconstructable mean.
- **`remote-inbound-rtp`** — RTCP receiver-report data about the stream the *other* endpoint is receiving; includes a `roundTripTime` derived from RTCP SR/RR timestamps.
- **`media-source` / `media-playout`** — on modern implementations, the playout side exposes `totalPlayoutDelay` and `totalSamplesCount`, which together give you an average client-side delay.

Collect `getStats()` on a fixed cadence (every 1–5 seconds) during a call and store the time series. Do not try to compute a single turn-level RTT by subtracting timestamps; use the RTCP-derived `currentRoundTripTime` and treat it as a slowly-varying property of the connection.

If you are running over a WebSocket transport (telephony media streams, direct provider connections), you don't get WebRTC stats. Substitute:

- Application-level ping / pong at a known cadence (e.g., every 5 s).
- Timestamping at ingress and acknowledgement at the far end.
- Packet-loss inference from RTP sequence gaps if the payload wraps RTP, otherwise from your own framing counters.

## OpenTelemetry: the pipeline side

Everything above the transport — ASR, LLM, TTS, orchestration — is HTTP-ish request / streaming traffic that OpenTelemetry already knows how to instrument. Use distributed tracing, not just metrics, because you need to reconstruct a single turn's timeline across processes.

Two conventions matter here:

1. **OpenTelemetry Semantic Conventions for Generative AI** (`gen_ai.*`) define spans and attributes for LLM calls. Common attributes you should emit: `gen_ai.system`, `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`. For streaming responses, record **time-to-first-token** as a span event or a separate attribute (`gen_ai.server.time_to_first_token` is in current draft conventions; check the current version of the spec before adopting a specific attribute name). <!-- needs-research: confirm final attribute name for TTFT in gen_ai semconv at author time -->
2. **Span events for leg boundaries** — within a top-level `turn` span, attach timestamped events for each boundary: `audio.first_frame`, `vad.speech_end`, `asr.first_partial`, `asr.final`, `llm.first_token`, `llm.last_token`, `tts.first_byte`, `tts.last_byte`, `playout.first_byte`. From those events you can derive every per-leg metric without having to open child spans for sub-second phases where span overhead would dominate.

A sensible span hierarchy for a single turn:

```
turn (root)
├── asr.recognize
│    ├── event: audio.first_frame_in_utterance
│    ├── event: vad.speech_end
│    ├── event: asr.first_partial
│    └── event: asr.final
├── orchestrate
│    ├── prompt.build
│    ├── retrieval (optional)
│    └── llm.generate
│          ├── event: llm.first_token
│          └── event: llm.last_token
└── tts.synthesize
     ├── event: tts.first_byte
     └── event: tts.last_byte
```

Give every span the same `turn.id` attribute and link child spans to the root via the trace context, not via ad-hoc IDs.

## Clock discipline

Latency numbers are useless if you compared clocks that weren't synchronised.

- **Inside one process**, use a monotonic clock (`time.monotonic()` in Python, `performance.now()` in a browser, `CLOCK_MONOTONIC` on Linux). Never use wall clock for elapsed-time math because NTP adjustments can move it backwards.
- **Across processes**, OpenTelemetry's `Span` stores start/end times as wall-clock timestamps. That is acceptable for *attribution*, because you are aligning events within a few hundred milliseconds of each other on machines that are NTP-synced. It is not acceptable for *duration* math on the same machine — use the monotonic clock inside one process and let the trace backend align wall-clock timestamps across processes.
- **Client / server skew** is real. A browser's `Date.now()` can drift seconds from a server's clock. For user-side timestamps that have to line up with server-side timestamps, either (a) carry the browser's `performance.now()` offset from a known handshake moment, or (b) do all duration math on the server side and treat the client as "emitted an event at round-trip/2 before we saw it", with an explicit lower bound.

## The one-table summary

For every leg of the pipeline from Chapter 1, you should be able to fill in this table on demand:

| Leg | Signal source | p50 / p95 / p99 | Alert threshold |
|---|---|---|---|
| Transport (RTT) | WebRTC `candidate-pair` stats | measured | matches region/edge SLO |
| Jitter / loss | WebRTC `inbound-rtp` | measured | codec-dependent |
| VAD endpoint | OTel event delta (`speech_end` − `last_speech_frame`) | measured | product-tuned |
| ASR partial | OTel event (`first_partial` − `audio.first_frame`) | measured | provider SLO |
| ASR final | OTel event (`asr.final` − `speech_end`) | measured | provider SLO + your budget |
| Orchestration | OTel `orchestrate` span | measured | your budget |
| LLM TTFT | OTel `llm.generate` + `first_token` event | measured | provider SLO |
| LLM TPOT | derived (`last_token` − `first_token`) / output_tokens | measured | model-dependent |
| TTS TTFB | OTel event (`tts.first_byte` − `llm.first_token`) | measured | provider SLO |
| Turn gap (end-to-end) | derived | measured | **product target** (e.g., p95 < 800 ms) |

If you can't fill this in for the last week of production traffic, that's the first thing to fix.

## Example instrumentation (Python-ish pseudocode)

Not real code, but enough to make the attribution concrete:

```python
tracer = trace.get_tracer("voice-agent")

async def handle_turn(audio_stream):
    with tracer.start_as_current_span("turn") as turn_span:
        turn_span.set_attribute("turn.id", turn_id())

        with tracer.start_as_current_span("asr.recognize") as asr_span:
            asr_span.add_event("audio.first_frame")
            final_text = await stream_asr(audio_stream, span=asr_span)
            # stream_asr adds asr.first_partial, vad.speech_end, asr.final events

        with tracer.start_as_current_span("llm.generate") as llm_span:
            llm_span.set_attribute("gen_ai.system", "openai")
            llm_span.set_attribute("gen_ai.request.model", "gpt-4o")
            token_iter = llm.stream(final_text)
            first_token = await token_iter.first()
            llm_span.add_event("llm.first_token")

            # Pass the token_iter straight into TTS — do not wait for last_token.
            with tracer.start_as_current_span("tts.synthesize") as tts_span:
                audio_iter = tts.stream(token_iter_from(first_token, token_iter))
                first_byte = await audio_iter.first()
                tts_span.add_event("tts.first_byte")
                await playout(audio_iter, span=tts_span)
                tts_span.add_event("tts.last_byte")
            llm_span.add_event("llm.last_token")
```

The important bits:

- `turn` is the root. Every leg hangs off it so a trace backend can reconstruct the turn gap in one query.
- TTS synthesis starts after `llm.first_token`, not after `llm.last_token`. This is the single most impactful architectural decision in the entire budget — Chapter 3 is dedicated to it.
- Each leg's timing is derived from span events, not from span start/end, because span start/end measure *the leg's work*, not *the user-perceptible boundaries* inside it.

## Smell tests before trusting your dashboards

- **A dashboard that shows only means is broken.** Minimum shape is p50 / p95 / p99.
- **A dashboard that doesn't line up legs on the same time axis is broken.** You cannot attribute the turn gap without a per-turn timeline.
- **A dashboard with no turn-gap metric at all is broken.** The product target is the top-line number; all other metrics are explanatory variables.
- **A dashboard sampled from a single caller is broken.** Collect from live traffic or from a synthetic-caller fleet (Chapter 5).

## Summary

- Measure and attribute **ASR TTFB / final, LLM TTFT / TPOT, TTS TTFB, and transport RTT** per turn.
- Use **WebRTC `getStats()`** (or a WebSocket equivalent) for transport metrics.
- Use **OpenTelemetry**, including the **GenAI semantic conventions** for LLM spans, with **span events** for leg boundaries hung off a single `turn` root.
- Use monotonic clocks for intra-process duration math; let NTP-synced wall clocks align across processes.
- Fill in the p50/p95/p99 table for the last week's traffic — if you can't, the rest of the module cannot improve a number you don't have.

Next chapter: designing the streaming graph so that TTS synthesis starts on the first LLM token, not the last sentence.
