# Resources for mod-109-latency-budget-architecture

Primary sources only. Use these to verify claims, anchor your budget citations, and defend architectural decisions against standards / published provider documentation rather than folklore.

## Standards: transmission and transport

- **ITU-T G.114 — One-way transmission time** — [itu.int/rec/T-REC-G.114](https://www.itu.int/rec/T-REC-G.114). The reference for the 150 / 400 ms thresholds cited in Chapter 1. Note that G.114 is a transmission-time recommendation for voice telephony; it does not define a target for AI-pipeline processing.
- **ITU-T G.107 — The E-model** — [itu.int/rec/T-REC-G.107](https://www.itu.int/rec/T-REC-G.107). Companion to G.114; a computational model for estimating conversational speech quality. Useful background for why one-way delay has the thresholds it has.
- **RFC 3550 — RTP: A Transport Protocol for Real-Time Applications** — [datatracker.ietf.org/doc/html/rfc3550](https://datatracker.ietf.org/doc/html/rfc3550). Source of the `jitter` definition referenced in Chapter 2.
- **RFC 3611 — RTP Control Protocol Extended Reports (RTCP XR)** — [datatracker.ietf.org/doc/html/rfc3611](https://datatracker.ietf.org/doc/html/rfc3611). Carries the voice quality metrics some media servers use for tail-latency reporting.
- **RFC 8866 — SDP: Session Description Protocol** — [datatracker.ietf.org/doc/html/rfc8866](https://datatracker.ietf.org/doc/html/rfc8866). Background for how codec and packetisation negotiation lands on the wire.

## Measurement: WebRTC and OpenTelemetry

- **W3C WebRTC 1.0** — [w3.org/TR/webrtc](https://www.w3.org/TR/webrtc/). Normative spec for the WebRTC API, including `RTCPeerConnection`.
- **W3C WebRTC Statistics API** — [w3.org/TR/webrtc-stats](https://www.w3.org/TR/webrtc-stats/). Defines `getStats()`, the stats objects (`inbound-rtp`, `outbound-rtp`, `candidate-pair`, `remote-inbound-rtp`, `media-playout`), and field semantics.
- **MDN — RTCPeerConnection.getStats()** — [developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/getStats](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/getStats). Developer-facing reference with usage patterns.
- **OpenTelemetry — Specification** — [opentelemetry.io/docs/specs/otel](https://opentelemetry.io/docs/specs/otel/). Tracing, metrics, and context propagation.
- **OpenTelemetry — Semantic Conventions for Generative AI** — [opentelemetry.io/docs/specs/semconv/gen-ai](https://opentelemetry.io/docs/specs/semconv/gen-ai/). Standard attribute names (`gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.*`) and span conventions for LLM calls. Specification is evolving; check the current version before adopting specific attribute names. <!-- needs-research: confirm the current recommended attribute name for LLM TTFT at author time -->
- **OpenTelemetry — Language SDKs** — [opentelemetry.io/docs/languages](https://opentelemetry.io/docs/languages/). Language-specific implementations.

## Streaming inference providers (regional availability, streaming APIs, prompt caching)

Always verify current region availability and feature support against the vendor's own documentation — each of these pages changes often.

### LLM providers

- **OpenAI — Platform** — [platform.openai.com/docs](https://platform.openai.com/docs/). Streaming chat completions, Realtime API, prompt caching documentation.
- **Anthropic — Claude API** — [docs.claude.com](https://docs.claude.com/en/api/overview). Streaming responses and prompt caching reference.
- **Google — Gemini API and Vertex AI** — [ai.google.dev/gemini-api/docs](https://ai.google.dev/gemini-api/docs), [cloud.google.com/vertex-ai/generative-ai](https://cloud.google.com/vertex-ai/generative-ai). Streaming, prompt caching, and regional-endpoint documentation.
- **Azure OpenAI Service** — [learn.microsoft.com/en-us/azure/ai-services/openai](https://learn.microsoft.com/en-us/azure/ai-services/openai/). Regional deployment, data-residency, and SLA guidance.
- **AWS Bedrock** — [docs.aws.amazon.com/bedrock](https://docs.aws.amazon.com/bedrock/). Regional model availability and streaming.

### ASR providers

- **Deepgram — Streaming API** — [developers.deepgram.com/docs/streaming](https://developers.deepgram.com/docs/streaming). Interim results, endpointing, and latency considerations.
- **AssemblyAI — Streaming Speech-to-Text** — [www.assemblyai.com/docs/speech-to-text/streaming](https://www.assemblyai.com/docs/speech-to-text/streaming). Universal-Streaming model latency guidance.
- **Google Cloud Speech-to-Text — Streaming recognition** — [cloud.google.com/speech-to-text/docs/basics](https://cloud.google.com/speech-to-text/docs/basics). Streaming vs. batch APIs.
- **Azure Speech Service — Speech to Text streaming** — [learn.microsoft.com/en-us/azure/ai-services/speech-service/speech-to-text](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/speech-to-text).

### TTS providers

- **ElevenLabs — Streaming TTS** — [elevenlabs.io/docs/api-reference/text-to-speech/stream](https://elevenlabs.io/docs/api-reference/text-to-speech/stream). WebSocket streaming and latency optimisation.
- **Cartesia — Sonic streaming** — [docs.cartesia.ai](https://docs.cartesia.ai/). Streaming TTS and latency SLOs.
- **LMNT — Streaming TTS** — [docs.lmnt.com](https://docs.lmnt.com/). Low-latency streaming synthesis.
- **Google Cloud Text-to-Speech — Streaming** — [cloud.google.com/text-to-speech](https://cloud.google.com/text-to-speech).

### Realtime / speech-to-speech surfaces

- **OpenAI Realtime API** — [platform.openai.com/docs/guides/realtime](https://platform.openai.com/docs/guides/realtime). WebRTC / WebSocket transport, speech-to-speech models.
- **Google Gemini Live API** — [ai.google.dev/gemini-api/docs/live](https://ai.google.dev/gemini-api/docs/live). Bidirectional streaming multimodal API.
- **Kyutai Moshi** — [arxiv.org/abs/2410.00037](https://arxiv.org/abs/2410.00037), [github.com/kyutai-labs/moshi](https://github.com/kyutai-labs/moshi). Open-weight speech-to-speech foundation model with reported latency characteristics; useful as a reference architecture.

## Orchestration frameworks (streaming graph patterns)

- **LiveKit Agents** — [docs.livekit.io/agents](https://docs.livekit.io/agents/). Streaming pipeline, VAD, turn detection, plugin adapters.
- **Pipecat** — [github.com/pipecat-ai/pipecat](https://github.com/pipecat-ai/pipecat), [docs.pipecat.ai](https://docs.pipecat.ai/). Processor-pipeline framework; reference implementation for TTS-on-first-token patterns.
- **Vocode** — [docs.vocode.dev](https://docs.vocode.dev/). Streaming voice pipeline library.

## Region, edge, and private peering

- **AWS Direct Connect** — [aws.amazon.com/directconnect](https://aws.amazon.com/directconnect/) and [docs.aws.amazon.com/directconnect](https://docs.aws.amazon.com/directconnect/). Dedicated network connection into AWS regions.
- **AWS Global Accelerator** — [aws.amazon.com/global-accelerator](https://aws.amazon.com/global-accelerator/). Anycast-based acceleration for internet-facing traffic.
- **Azure ExpressRoute** — [learn.microsoft.com/en-us/azure/expressroute](https://learn.microsoft.com/en-us/azure/expressroute/). Private connectivity into Azure.
- **Google Cloud Interconnect** — [cloud.google.com/network-connectivity/docs/interconnect](https://cloud.google.com/network-connectivity/docs/interconnect). Dedicated and Partner Interconnect products.
- **Cloudflare — Internet latency insights** — [radar.cloudflare.com](https://radar.cloudflare.com/). Useful for a view of public-internet routing and latency trends.
- **Telephony edge PoPs** — vendor documentation: [Twilio Super Network](https://www.twilio.com/en-us/global-infrastructure), [Vonage global infrastructure](https://developer.vonage.com/en/voice/voice-api/overview), [Zoom data centre regions](https://support.zoom.us/hc/en-us/articles/360059254691).

## Serving-layer and inference tuning (boundary to the inference engineer)

- **vLLM** — [docs.vllm.ai](https://docs.vllm.ai/). Continuous batching, PagedAttention, streaming responses, speculative decoding.
- **Text Generation Inference (TGI)** — [github.com/huggingface/text-generation-inference](https://github.com/huggingface/text-generation-inference). Hugging Face's serving stack.
- **NVIDIA TensorRT-LLM** — [github.com/NVIDIA/TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM). Optimised LLM serving on NVIDIA GPUs, including speculative decoding support.
- **SGLang** — [github.com/sgl-project/sglang](https://github.com/sgl-project/sglang). Structured generation + fast serving.
- **NVIDIA Triton Inference Server** — [github.com/triton-inference-server/server](https://github.com/triton-inference-server/server). Request batching, scheduling, and streaming.
- **Speculative decoding (speculative sampling)** — [arxiv.org/abs/2211.17192](https://arxiv.org/abs/2211.17192) (Leviathan et al., Google). Original formulation; see also EAGLE, Medusa, and lookahead-decoding papers for follow-ups.

## Eval and CI integration

- Cross-reference to [mod-112 — Voice & Multimodal Eval and Observability](../mod-112-voice-multimodal-eval-and-observability/README.md) for the broader eval harness that the CI gate from Chapter 5 plugs into.
- **GitHub Actions** — [docs.github.com/actions](https://docs.github.com/en/actions). For the CI wiring in exercise-05 (any other CI is equivalent).
- **OpenTelemetry Collector** — [opentelemetry.io/docs/collector](https://opentelemetry.io/docs/collector/). For local-development and CI trace aggregation.

## Role / ladder references (within this curriculum ecosystem)

- [Voice & Multimodal Agent Engineer (L35) — this track](../../README.md)
- [AI Infra Inference Engineer (L35)](https://github.com/AI-Infra-Curriculum/ai-infra-inference-engineer-learning) — peer role; Chapter 6 defines the boundary.
- [Forward Deployed AI Engineer (L35)](https://github.com/ai-engineering-curriculum/forward-deployed-engineer-learning) — peer role in customer delivery.
- [Agentic AI Engineer (L30)](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning) — feeder role, upstream of the orchestration graph you own.

## Citation hygiene

- **Latency numbers in vendor blog posts are typically p50 marketing numbers.** Treat them as p50, not p95, until you can measure them yourself in your region on your traffic.
- **Region availability changes month-to-month.** Cite vendor pages with an access date; avoid writing "X supports region Y" as a permanent architectural assumption.
- **"Realtime" is not an SLO.** When a vendor ships a "realtime API", read their published latency guidance; "realtime" by itself doesn't imply a specific number.
- **Private-peering latency claims deserve scepticism.** The win is typically consistency (jitter / tail), not raw latency. If a sales deck promises both, ask to see the measurement.
