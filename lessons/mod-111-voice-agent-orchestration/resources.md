# Resources for mod-111-voice-agent-orchestration

Primary sources only. Voice-agent orchestration frameworks and managed platforms move quickly — class names, pricing, and compliance posture can change between quarters. Date every citation that depends on a specific class name, pricing number, or compliance commitment.

## Open-source orchestration frameworks

### Pipecat

- **Docs** — [docs.pipecat.ai](https://docs.pipecat.ai/). Processor pipeline, transports, service integrations.
- **Repository** — [github.com/pipecat-ai/pipecat](https://github.com/pipecat-ai/pipecat). Source, examples, and the `src/pipecat/services/` tree is the authoritative list of first-class service integrations.
- **Pipecat Cloud** — [daily.co/pipecat-cloud](https://www.daily.co/products/pipecat-cloud) (verify current URL at access date). Managed runtime for Pipecat agents from the maintainer.
- **Example applications** — the `examples/` directory of the Pipecat repo carries canonical pipelines for Daily / LiveKit / Twilio transports and for OpenAI Realtime / Gemini Live.

### LiveKit Agents

- **Docs** — [docs.livekit.io/agents](https://docs.livekit.io/agents/). Workers, sessions, the unified `AgentSession` API, dispatch rules.
- **Python repo** — [github.com/livekit/agents](https://github.com/livekit/agents). Plugin catalogue is under `livekit-plugins/`.
- **Node repo** — [github.com/livekit/agents-js](https://github.com/livekit/agents-js). Node/TypeScript runtime.
- **LiveKit Cloud** — [livekit.io/cloud](https://livekit.io/cloud) (verify current URL). Managed media server + agent deployment.
- **LiveKit server (self-hosted)** — [github.com/livekit/livekit](https://github.com/livekit/livekit). Open-source media server you deploy yourself.

### Vocode

- **Docs** — [docs.vocode.dev](https://docs.vocode.dev/). `StreamingConversation`, transcribers, synthesizers, agents, input/output devices.
- **Repository** — [github.com/vocodedev/vocode-core](https://github.com/vocodedev/vocode-core). Open-source core.
- **Hosted / managed offering** — [vocode.dev](https://vocode.dev/). Verify current availability.

## Managed voice-agent platforms

Treat each entry as positional — check the current docs page for pricing, features, SLAs, and compliance documents at your access date.

- **Retell AI** — [retellai.com](https://www.retellai.com/) · [docs.retellai.com](https://docs.retellai.com/). Conversation Flow builder + programmable Agents.
- **Vapi** — [vapi.ai](https://vapi.ai/) · [docs.vapi.ai](https://docs.vapi.ai/). Assistants, squads, transfers.
- **Bland AI** — [bland.ai](https://www.bland.ai/) · [docs.bland.ai](https://docs.bland.ai/). Pathways and outbound dispatch.
- **Hamming** — [hamming.ai](https://hamming.ai/). Voice-agent testing / eval; complements rather than replaces an orchestration framework.
- **Phonely** — [phonely.ai](https://www.phonely.ai/). Receptionist-focused voice AI.
- **Observe.AI** — [observe.ai](https://www.observe.ai/). Contact-centre AI with voice-agent features.

## Realtime / speech-to-speech surfaces used inside the orchestration layer

- **OpenAI Realtime API** — [platform.openai.com/docs/guides/realtime](https://platform.openai.com/docs/guides/realtime). WebRTC/WebSocket session, turn detection, tool calling, server VAD.
- **OpenAI function calling** — [platform.openai.com/docs/guides/function-calling](https://platform.openai.com/docs/guides/function-calling). Parallel tool-call shape and tool-call-id stitching.
- **Google Gemini Live** — [ai.google.dev/gemini-api/docs/live](https://ai.google.dev/gemini-api/docs/live). Bidirectional multimodal streaming.
- **Google Gemini function calling** — [ai.google.dev/gemini-api/docs/function-calling](https://ai.google.dev/gemini-api/docs/function-calling). Parallel calling, response handling.
- **Anthropic tool use** — [docs.claude.com/en/docs/agents-and-tools/tool-use/overview](https://docs.claude.com/en/docs/agents-and-tools/tool-use/overview). Multiple `tool_use` content blocks per response; `tool_use_id` stitching.
- **Hume EVI** — [dev.hume.ai](https://dev.hume.ai/). Empathic voice interface and its SDKs.

## Prompt / context caching (relevant to Chapter 7)

- **OpenAI Prompt Caching** — [platform.openai.com/docs/guides/prompt-caching](https://platform.openai.com/docs/guides/prompt-caching). Automatic prefix caching; keep stable parts of the system prompt first.
- **Anthropic Prompt Caching** — [docs.claude.com/en/docs/build-with-claude/prompt-caching](https://docs.claude.com/en/docs/build-with-claude/prompt-caching). `cache_control: ephemeral`; TTL semantics vary — check current docs at access date.
- **Gemini Context Caching** — [ai.google.dev/gemini-api/docs/caching](https://ai.google.dev/gemini-api/docs/caching). Explicit cached content object.

## Model Context Protocol (MCP)

- **Specification** — [modelcontextprotocol.io](https://modelcontextprotocol.io/). Authoritative spec and introductory docs.
- **Reference servers** — [github.com/modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers). Working server implementations for common tool surfaces.
- **SDKs** — [github.com/modelcontextprotocol](https://github.com/modelcontextprotocol). Python, TypeScript, and other client/server SDKs.

## Observability

- **OpenTelemetry Semantic Conventions for Generative AI** — [opentelemetry.io/docs/specs/semconv/gen-ai](https://opentelemetry.io/docs/specs/semconv/gen-ai/). The `gen_ai.*` attributes used for the per-leg spans in exercises 01 and 02.
- Cross-reference to [mod-112 — Voice and Multimodal Eval and Observability](../mod-112-voice-multimodal-eval-and-observability/README.md).

## Transport and telephony (referenced from mod-107 and used in the exercises)

- **Daily** — [docs.daily.co](https://docs.daily.co/). WebRTC rooms used as the Pipecat default transport.
- **LiveKit** — [docs.livekit.io](https://docs.livekit.io/). WebRTC media server, SIP, and the Agents framework.
- **Twilio Programmable Voice** — [twilio.com/docs/voice](https://www.twilio.com/docs/voice). PSTN entry point used by Pipecat's and Vocode's Twilio connectors.
- **Vonage Voice API** — [developer.vonage.com/en/voice](https://developer.vonage.com/en/voice). Alternative PSTN provider supported by Vocode.

## Compliance and consent (inherited from mod-108)

- Cross-reference to [mod-108 — Safety and Compliance for Voice](../mod-108-safety-compliance-voice/README.md) for GDPR, HIPAA, PCI-DSS, and consent framing.
- **Model Context Protocol Security Considerations** — see the MCP spec's security section. MCP servers are the right chokepoint for per-call authorisation and audit; do not bypass them in the orchestration layer.

## Latency and architecture (inherited from mod-109)

- Cross-reference to [mod-109 — Real-Time Latency Budget Architecture](../mod-109-latency-budget-architecture/README.md) for the budget commitment, streaming-graph patterns, and region / peering design that this module's orchestration choices land on top of.

## Multimodal (inherited from mod-110)

- Cross-reference to [mod-110 — Multimodal Input Design](../mod-110-multimodal-input-design/README.md) for the modality-interleaving contract that the orchestration framework has to carry.

## Role / ladder references (within this curriculum ecosystem)

- [Voice & Multimodal Agent Engineer (L35) — this track](../../README.md).
- [AI Infra Inference Engineer (L35)](https://github.com/AI-Infra-Curriculum/ai-infra-inference-engineer-learning) — peer role on serving-layer tail-latency tuning; relevant when the orchestration layer's latency ceiling is set by the inference stack underneath.
- [Applied AI Engineer (L30)](https://github.com/ai-engineering-curriculum/applied-ai-engineer-learning) — peer role; tool-surface and feature-engineering boundary.

## Citation hygiene

- **Framework class names move.** LiveKit consolidated `VoicePipelineAgent` and `MultimodalAgent` into a unified `AgentSession`; Pipecat has renamed internal processors between major versions. Date every citation that depends on a specific class name.
- **Managed-platform pricing is dated by definition.** Record the per-minute rate and the date you saw it.
- **"Realtime" and "live" are not SLOs.** Provider latency properties are measurable but not guaranteed; measure in your region on your traffic before quoting a number.
- **Compliance claims require signed documents.** A platform's marketing page listing "HIPAA compliant" is not a BAA. Cite the signed agreement when it matters.
- **MCP is still stabilising.** Features, auth semantics, and discovery flow continue to evolve; pin the spec version you built against.
