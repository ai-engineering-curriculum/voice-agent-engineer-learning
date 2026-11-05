# 06. OpenTelemetry GenAI Semantic Conventions + Voice-Specific Attributes + WebRTC Stats

## Why this chapter

Chapters 1–5 produce numbers. This chapter wires those numbers into a single pane of glass so that when a call goes wrong you can pivot from "the p95 latency alert fired" to "turn 7 of call xyz took 1.4 s because TTS TTFB was 900 ms on the Cartesia leg and the upstream jitter was 40 ms" in a few clicks, not a few days.

The standards you will align with are the [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/) for the LLM-call shape, OpenTelemetry's general trace and metric conventions for the per-processor spans, and the [W3C WebRTC `RTCPeerConnection.getStats()` API](https://www.w3.org/TR/webrtc-stats/) for the transport-side counters. Voice-specific attributes — turn ID, speaker ID, modality, TTFB per leg, barge-in reaction time, WER-of-turn, MOS proxy per turn — are added on top of the GenAI conventions, because the GenAI conventions do not yet (at authoring time) describe voice-specific fields.

The chapter closes on the evaluation / observability platform choice. Langfuse, LangSmith, Arize Phoenix, Weave, and Braintrust each consume OTel; which one you pick is a product decision, not a protocol decision. Keep the instrumentation vendor-neutral so the choice stays yours.

## The GenAI semantic conventions — what you adopt and what you add

The OpenTelemetry GenAI semconv (see the [current draft](https://opentelemetry.io/docs/specs/semconv/gen-ai/) at your access date; it is actively evolving) defines spans and attributes for model calls. The attributes most relevant to the voice stack:

- `gen_ai.system` — the provider (`openai`, `anthropic`, `google`, `azure`, `aws.bedrock`, etc.).
- `gen_ai.request.model` — the requested model name.
- `gen_ai.response.model` — the actual model that served the response (vendors substitute).
- `gen_ai.usage.input_tokens` / `gen_ai.usage.output_tokens` — token counts for billing and TPOT math.
- `gen_ai.operation.name` — `chat`, `text_completion`, `embeddings`, etc.
- Streaming attributes — the current draft includes a time-to-first-token attribute; verify the exact name in the version of the spec you adopt. <!-- needs-research: confirm final TTFT attribute name in current gen_ai semconv -->

Adopt these verbatim for the LLM span. Do not invent `llm.ttft` on your own if the spec has settled on a name.

What the GenAI semconv does not cover: ASR, TTS, VAD, endpointing, transport. Those are covered by a **voice-specific attribute convention** you define and document in-repo. A reasonable namespace:

| Attribute | Type | Description |
|---|---|---|
| `voice.turn.id` | string | Stable ID for the user+agent turn pair. The root span of the turn. |
| `voice.call.id` | string | Stable ID for the whole conversation (call / session). |
| `voice.speaker.id` | string | Speaker role: `user` / `agent` (expand to speaker IDs in multi-party). |
| `voice.modality` | string | `audio`, `text`, `image`, etc. (multi-modal, mod-110). |
| `voice.asr.provider` | string | ASR vendor. |
| `voice.asr.model` | string | ASR model/version. |
| `voice.asr.first_partial_ms` | int | TTFB on first partial transcript, from first audio frame. |
| `voice.asr.final_latency_ms` | int | Endpoint-to-final-transcript latency. |
| `voice.asr.wer` | float | WER of the turn against gold, when gold is available (offline eval). |
| `voice.asr.confidence` | float | ASR's reported utterance-level confidence. |
| `voice.tts.provider` | string | TTS vendor. |
| `voice.tts.model` | string | TTS model/version. |
| `voice.tts.first_byte_ms` | int | First audio byte latency from the TTS leg's first-token in. |
| `voice.tts.mos_proxy` | float | DNSMOS or UTMOS value for this turn's TTS audio (Chapter 2). |
| `voice.turn.ttfa_ms` | int | Time-to-first-audio for this turn (Chapter 3). |
| `voice.turn.mouth_to_mouth_ms` | int | Mouth-to-mouth for this turn, if measurable. |
| `voice.barge_in.detected` | bool | Did a barge-in happen in this turn? |
| `voice.barge_in.reaction_ms` | int | Barge-in reaction time, if applicable. |
| `voice.transport.rtt_ms` | int | Current RTT from WebRTC stats or WebSocket ping. |
| `voice.transport.jitter_ms` | int | Jitter from inbound-rtp. |
| `voice.transport.packet_loss` | float | Packet loss fraction. |
| `voice.transport.codec` | string | Negotiated codec (e.g., `opus`, `g711-ulaw`). |
| `voice.vad.endpoint_decision_ms` | int | Latency from last speech frame to endpoint fire. |
| `voice.content.policy_pass` | bool | Policy-compliance judge result for this turn (online eval). |
| `voice.content.pii_hits` | int | PII regex hit count. |

Document this namespace once, in a `conventions.md` in your telemetry repo, and require it on every span. Enforce via a linter or an OTel instrumentation library; without enforcement, half of your spans will miss attributes.

## The span tree per turn

A turn's trace tree should let a reader reconstruct everything at a glance:

```
call (root span — call.id, speaker, start/end)
└── turn (turn.id, modality=audio, strata annotations)
    ├── transport.ingest  (bytes in, RTT, jitter, packet-loss, codec)
    ├── vad  (events: speech_start, speech_end; voice.vad.endpoint_decision_ms)
    ├── asr.recognize  (voice.asr.*, events: first_partial, final)
    ├── orchestrate
    │   ├── prompt.build  (prompt token count, cache hit rate)
    │   ├── retrieval  (optional — ragas-style spans)
    │   └── gen_ai.chat  (gen_ai.* attributes + events: llm.first_token, llm.last_token)
    │       └── tool.call  (if any — one child span per call)
    ├── tts.synthesize  (voice.tts.*, events: first_byte, last_byte)
    ├── transport.egress  (bytes out, playout delay)
    └── voice.content.*  (post-call content eval attributes attached on close)
```

Design rules that save you later:

- **One root `call` span per session.** Everything else is a child. `turn` spans are children of `call`.
- **Events, not child spans, for sub-millisecond boundaries.** Span start/end overhead can dominate when the leg is fast; use `span.add_event("llm.first_token")` and derive durations in the backend.
- **Attach stratification attributes to the turn, not the call.** Acoustic condition, call phase, and codec can change mid-call; strata should move with the turn.
- **Set the trace context across processes.** When a call crosses from the edge (WebRTC server) to the orchestrator to the LLM proxy, use OTel propagation so the spans end up in the same trace.
- **Give every turn a stable business key.** `voice.turn.id` must be the same string everywhere (orchestrator logs, judge-as-LLM outputs, incident captures). Do not let two components invent their own ID.

## WebRTC stats on the same plane

WebRTC's `getStats()` is the transport-side ground truth. The relevant objects (see the [W3C WebRTC Stats spec](https://www.w3.org/TR/webrtc-stats/) and mod-109 Chapter 2):

- `inbound-rtp` / `outbound-rtp` — `jitter`, `packetsLost`, `bytesReceived`, `framesDecoded`, `audioLevel`.
- `candidate-pair` — `currentRoundTripTime`, `totalRoundTripTime`, `responsesReceived`.
- `remote-inbound-rtp` — `roundTripTime` (RTCP-derived).
- `media-source` / `media-playout` — `totalPlayoutDelay`, `totalSamplesCount`, `concealmentEvents`.

Collect `getStats()` on a fixed cadence (1–5 s) and emit the deltas as OTel **metrics** (not spans — span-per-stats-sample would be noisy). Suggested metric names:

- `voice.transport.rtt_ms` (gauge)
- `voice.transport.jitter_ms` (gauge)
- `voice.transport.packets_lost` (counter)
- `voice.transport.playout_delay_ms` (gauge)
- `voice.transport.concealment_events` (counter)

Attach `call.id`, `speaker` (`user` / `agent`), and the current `turn.id` as labels so a dashboard query can slice per turn. The trace and the metrics are correlated by the shared IDs, not by a cross-join on timestamps.

If the transport is a WebSocket (telephony Media Streams, direct provider connections), you do not get WebRTC stats. Substitute application-level pings, ingress/egress timestamps, and framing-counter-derived loss, and emit under the same metric names. The dashboard does not need to know which transport produced the number.

## Wiring into evaluation / observability platforms

The platforms you will see in this space differ on whether they are *trace-first* (OTel ingestion, span-level inspection) or *experiment-first* (eval-run tracking, dataset-vs-model-vs-prompt grid). The voice stack usually needs both; the question is which one you build *on* and which one you build *around*.

A snapshot at authoring time (verify each vendor's docs for current OTel support and feature set):

### Langfuse ([langfuse.com](https://langfuse.com/))

- Open-source, self-hostable ([`langfuse/langfuse`](https://github.com/langfuse/langfuse)) and cloud.
- Trace-first; the span tree is first-class. Supports GenAI semconv ingestion and arbitrary custom attributes.
- Judge-as-LLM "scores" attach to traces; dataset-based evals are supported.
- OTel ingestion is documented; verify current adapter.

### LangSmith ([smith.langchain.com](https://smith.langchain.com/))

- Commercial, from LangChain.
- Trace-first with strong LangChain integration; non-LangChain spans supported.
- Datasets, experiments, and eval runs are first-class.
- OTel ingestion via the LangSmith SDK or via an OTel exporter; verify current support.

### Arize Phoenix ([phoenix.arize.com](https://phoenix.arize.com/))

- Open-source ([`Arize-ai/phoenix`](https://github.com/Arize-ai/phoenix)) from Arize AI. Also available as a managed Arize platform.
- Trace-first with explicit [OpenInference](https://github.com/Arize-ai/openinference) spec on top of OTel.
- Strong for distribution drift and embedding-based analysis (useful for Chapter 5's input drift).

### Weave ([wandb.ai/site/weave](https://wandb.ai/site/weave))

- From Weights & Biases.
- Experiment-first, with tracing support.
- Integrates with W&B's existing training-experiment surface, useful if the same team runs model-training experiments.

### Braintrust ([braintrust.dev](https://www.braintrust.dev/))

- Commercial.
- Eval-first — strong on dataset-vs-model grids, offline experimentation, and judge-as-LLM workflows.
- Trace ingestion supported; verify OTel coverage.

### Deciding between them

For a voice-agent product the usable criteria:

- **Does it consume OTel natively?** If yes, you can switch vendors without re-instrumenting. If no, you are writing adapter code that locks you in.
- **Does it store audio or audio references?** Traces are more useful when a reader can play the TTS clip from the trace. Some platforms store blob payloads; others require you to attach URLs to artefacts in your own object store.
- **Can it drive the CI gate?** The gate from Chapter 4 reads eval metrics; platforms differ on whether they expose a programmatic API for "did the eval run pass".
- **Does the compliance posture work?** On-prem / self-hosted matters for HIPAA deployments; EU data-residency matters for GDPR. Check against the mod-108 compliance envelope.
- **How does your team already work?** If the team is deep in LangChain, LangSmith is low-friction; if the team already runs on W&B for training, Weave is low-friction. Cognitive-load cost is real.

Pick one and invest in it; keep the instrumentation vendor-neutral by emitting OTel spans with standard attributes. The vendor is where you *view*; the OTel pipe is what you *own*.

## The dashboards this plane makes possible

With the spans + metrics + judge scores all on one plane, four dashboards are now cheap:

1. **Per-call timeline.** Pick a `call.id`, see every turn's span tree, hover to see per-leg timings, play the audio clip for any turn, read the judge score, read the implicit-feedback flags. The "debug one bad call in five minutes" dashboard.
2. **Per-flow quality.** For each named flow (support, order-taking, triage, etc.), p50/p95 mouth-to-mouth, deflection, judge score, implicit feedback. The "is the product working" dashboard.
3. **Per-stratum drift.** The Chapter 5 drift detectors' current state, per stratum, with an alert status and a run-book link.
4. **Release-rollout board.** Side-by-side control vs. canary metrics for whatever is currently rolling out. Rollback status visible.

All four read the same OTel + metrics store. Nothing in them is voice-vendor-specific.

## Example instrumentation (Python-ish pseudocode)

```python
# Illustrative. Verify current OTel API and GenAI semconv attribute names.
from opentelemetry import trace, metrics
from opentelemetry.trace import SpanKind

tracer = trace.get_tracer("voice-agent")
meter = metrics.get_meter("voice-agent")
rtt_gauge = meter.create_gauge("voice.transport.rtt_ms")
jitter_gauge = meter.create_gauge("voice.transport.jitter_ms")

async def handle_turn(call_id, turn_id, user_audio, strata):
    with tracer.start_as_current_span(
        "turn",
        attributes={
            "voice.call.id": call_id,
            "voice.turn.id": turn_id,
            "voice.modality": "audio",
            "voice.call.phase": strata.phase,
            "voice.acoustic.condition": strata.acoustic,
            "voice.transport.codec": strata.codec,
        },
    ) as turn_span:
        # ASR
        with tracer.start_as_current_span("asr.recognize") as asr_span:
            asr_span.set_attribute("voice.asr.provider", "deepgram")
            asr_span.set_attribute("voice.asr.model", "nova-3")
            result = await asr.stream(user_audio, span=asr_span)
            asr_span.set_attribute("voice.asr.first_partial_ms", result.first_partial_ms)
            asr_span.set_attribute("voice.asr.final_latency_ms", result.final_latency_ms)
            asr_span.set_attribute("voice.asr.confidence", result.confidence)

        # Orchestrate + LLM
        with tracer.start_as_current_span("gen_ai.chat") as llm_span:
            llm_span.set_attribute("gen_ai.system", "openai")
            llm_span.set_attribute("gen_ai.request.model", "gpt-4o-mini")
            llm_span.set_attribute("gen_ai.operation.name", "chat")
            tokens = llm.stream(result.final_text)
            first = await tokens.first()
            llm_span.add_event("llm.first_token")

            # TTS, started on the first token
            with tracer.start_as_current_span("tts.synthesize") as tts_span:
                tts_span.set_attribute("voice.tts.provider", "cartesia")
                tts_span.set_attribute("voice.tts.model", "sonic-2")
                audio = tts.stream(stream_after(first, tokens))
                first_byte = await audio.first()
                tts_span.add_event("tts.first_byte")
                await playout(audio, span=tts_span)
                tts_span.add_event("tts.last_byte")

            llm_span.add_event("llm.last_token")

        # Post-call content eval attached on close
        turn_span.set_attribute("voice.turn.ttfa_ms", measured_ttfa)
        turn_span.set_attribute("voice.turn.mouth_to_mouth_ms", measured_m2m)

# WebRTC stats emitted as metrics on a cadence
async def poll_stats(peer_connection, call_id):
    while True:
        stats = await peer_connection.getStats()
        rtt_ms = stats["candidate-pair"]["currentRoundTripTime"] * 1000
        jitter_ms = stats["inbound-rtp"]["jitter"] * 1000
        rtt_gauge.set(rtt_ms, {"voice.call.id": call_id, "speaker": "user"})
        jitter_gauge.set(jitter_ms, {"voice.call.id": call_id, "speaker": "user"})
        await asyncio.sleep(2)
```

The point of the pseudocode is that every data item from Chapters 1–5 ends up on the same trace / metric pane, keyed by the same `voice.call.id` and `voice.turn.id`. That is the entire deliverable of this chapter.

## Summary

- Adopt the **OpenTelemetry GenAI semantic conventions** for every LLM call; verify the current spec version and attribute names at your access date.
- Extend with a documented **voice-specific attribute namespace** (`voice.*`) covering turn ID, call ID, speaker, modality, ASR/TTS provider+model, per-leg timings, WER / MOS-proxy per turn, barge-in, transport metrics, and content-eval flags.
- Shape the per-turn **span tree** with one root `call` span, child `turn` spans, and leg-level spans for `transport.ingest / vad / asr.recognize / orchestrate / gen_ai.chat / tts.synthesize / transport.egress`. Use events for sub-millisecond boundaries.
- Emit **WebRTC `getStats()`** as OTel metrics on a 1–5 s cadence, keyed by `voice.call.id` and `voice.turn.id`, so trace and metrics line up.
- **Wire into one evaluation / observability platform** — Langfuse, LangSmith, Arize Phoenix, Weave, or Braintrust — on evidence (OTel support, audio support, CI-gate API, compliance posture, team cognitive load). Keep the instrumentation vendor-neutral.
- Four dashboards fall out automatically: per-call timeline, per-flow quality, per-stratum drift, release rollout.

Next chapter: the role boundary — what this module owns vs. what the `ai-eval-engineer` (L30) and the `ai-evaluation-engineer` (L35, governance family) own, and what the release-assurance packet looks like at the handoff.
