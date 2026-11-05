# exercise-06: Instrument the Voice Stack — OTel GenAI Semconv + Voice Attributes + WebRTC Stats on One Plane

**Estimated effort:** 1 hour · **Output:** a vendor-neutral OTel instrumentation that emits GenAI semconv + the `voice.*` attribute namespace, WebRTC `getStats()` sidecar, four working dashboards, and a chosen observability vendor

## Objective

Instrument the voice agent so every turn produces a span tree whose events, attributes, and correlated metrics let a reader reconstruct the call. Adopt the OpenTelemetry GenAI semantic conventions for LLM spans, extend with the documented `voice.*` namespace from Chapter 6 for voice-specific fields, and stream WebRTC `getStats()` as OTel metrics on the same correlation keys (`voice.call.id`, `voice.turn.id`). Choose and configure one observability platform (Langfuse / LangSmith / Arize Phoenix / Weave / Braintrust) on evidence — keep the emission vendor-neutral so the choice stays reversible.

## Prerequisites

- [Chapter 6 — OpenTelemetry, Voice Attributes, and WebRTC Stats](../06-otel-voice-attrs-webrtc-stats.md).
- A pipeline from mod-111 with at least the shape `transport → VAD → ASR → orchestrator → LLM → TTS → transport`.
- Current OpenTelemetry Python / Node SDK plus the GenAI semantic-convention attribute names (verify against the live spec at your date).
- A WebRTC transport (Daily / LiveKit / Twilio Media Streams — mod-107). If WebSocket-only, substitute application-level pings / timestamps per Chapter 6.
- An observability platform account (self-hosted Langfuse or Phoenix is fine; cloud accounts are fine too).

## Steps

1. **Pin the semconv and namespace versions.** Vendor the GenAI semantic-convention attribute list at the version you adopt. Write a `conventions.md` in your telemetry repo documenting the `voice.*` namespace (table from Chapter 6). Both versions are strings you emit on every root span.

2. **Build the span tree.** Implement the per-turn span tree from Chapter 6: `call` → `turn` → leg spans for `transport.ingest`, `vad`, `asr.recognize`, `orchestrate`, `gen_ai.chat`, `tts.synthesize`, `transport.egress`. Use OTel span events for sub-span boundary timestamps (`asr.first_partial`, `llm.first_token`, `tts.first_byte`, etc.).

3. **Attribute enforcement.** Write a lint / validator that fails the build if a `turn` span is emitted without the required `voice.*` attributes. Enforce in CI.

4. **WebRTC stats sidecar.** A loop that polls `RTCPeerConnection.getStats()` every 1–5 seconds, extracts RTT / jitter / packet loss / playout delay / concealment events, emits as OTel metrics with labels `voice.call.id`, `voice.turn.id` (latest active turn), `speaker` (`user` / `agent`). If on WebSocket transport, emit the same metric names from ping / framing-counter derivation and note the substitution.

5. **Trace-context propagation.** Verify that spans emitted by the transport layer, the orchestrator, and any provider SDKs land in the same trace. OTel auto-propagation across HTTP + WebSocket + gRPC. Fix any cross-process gaps.

6. **Pick and wire the observability platform.** Short evidence-based comparison: OTel support, audio-blob support, CI-gate programmatic API, compliance posture for your product (HIPAA / GDPR / data residency), cognitive-load cost. Document the pick in a one-paragraph decision note. Point the OTel exporter at it.

7. **Build the four dashboards.** In the chosen platform:
   - **Per-call timeline** — pick a `call.id`, see spans + events + audio clips per turn.
   - **Per-flow quality** — p95 mouth-to-mouth, deflection, judge scores, implicit feedback.
   - **Per-stratum drift** — Chapter 5 drift detector state per stratum.
   - **Release rollout** — control vs. canary metric panels.

8. **Prove correlation.** Pick a real call, pivot from the mouth-to-mouth metric panel → to the turn with the worst mouth-to-mouth → to its `tts.synthesize` span → to the WebRTC jitter / RTT metrics at that timestamp. If any hop breaks, instrumentation is incomplete; fix.

## Starter guidance

- **GenAI semconv is draft.** Verify the exact attribute names at your access date; do not hand-code names that are about to be renamed. Pin in `conventions.md`.
- **Do not ship free-text `turn.id` fields.** A stable string generated at the start of the turn by the orchestrator and threaded through every leg. Two components inventing their own IDs is the most common correlation break.
- **Events, not child spans, for ms-scale boundaries.** `tts.first_byte` is an event attached to the `tts.synthesize` span. Modelling it as a sub-span adds overhead that dominates the leg's actual duration.
- **Stratification attributes change mid-call.** Attach them to the turn, not the call.
- **Audio in traces is expensive.** Store audio in object storage with a stable URL, attach the URL as an attribute, not the bytes themselves.
- **Vendor-neutral emission beats a lock-in.** Emit OTel; let the platform ingest OTel. If the platform's SDK requires calls to proprietary APIs, you have accepted lock-in — make that choice consciously.

## Acceptance criteria

- [ ] `conventions.md` documenting the adopted GenAI semconv version and the `voice.*` namespace.
- [ ] A per-turn span tree with the Chapter 6 shape, including events on each leg boundary.
- [ ] A CI validator that fails on missing required `voice.*` attributes on root `turn` spans.
- [ ] A WebRTC stats sidecar emitting RTT / jitter / loss / playout delay as OTel metrics with correlation labels.
- [ ] Trace propagation verified across transport → orchestrator → LLM leg.
- [ ] A chosen observability platform wired via OTel with a documented decision note.
- [ ] The four dashboards live and linkable from the repo.
- [ ] A reproducible pivot from a dashboard panel to the exact turn's span and its correlated transport metric.

## Stretch goals

- **OpenInference alignment.** If you picked Phoenix / Arize, align span attribute names with [OpenInference](https://github.com/Arize-ai/openinference) to get richer out-of-box dashboards. Document the mapping between your `voice.*` namespace and OpenInference where they overlap.
- **Second-platform parallel export.** Export OTel to two platforms simultaneously for one week to prove the vendor-neutrality claim.
- **Audio attachment pattern.** Prototype an audio-playback panel in the chosen platform (embed a signed URL to the TTS clip for any turn). Decide whether the platform can host the UX you want.
- **Cost attribution.** Attach `gen_ai.usage.*` tokens to the call-level span so cost rollups are available per call / per flow.
- **Trace sampling policy.** For high-volume calls, sample traces (while keeping metrics at 100%); verify the sampling policy preserves the per-turn information you actually need for incident debugging.

## Keep it

This is the backbone every other exercise plugs into. ASR, TTS, conversational, and content metrics all land here as attributes on the right span. Online eval (exercise-05) reads from this plane; the CI gate (exercise-04) attaches the baseline against artefacts that reference this plane's trace IDs. If the plane is broken, everything above it is harder to debug.
