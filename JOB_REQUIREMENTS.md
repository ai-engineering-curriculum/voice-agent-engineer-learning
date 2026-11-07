# Voice & Multimodal Agent Engineer — Job Requirements

> **Cycle:** 2026-11 · **Window:** 2026-08-09 → 2026-11-07 (90 days) · **Sample:** 31 postings (23 in-window, 8 just out-of-window retained for breadth)
>
> Raw normalized postings live in [`.aicg/job-requirements.json`](./.aicg/job-requirements.json). The curriculum-plan delta for this cycle lives in [`.aicg/curriculum-plan-delta.json`](./.aicg/curriculum-plan-delta.json). Both files are regenerated on every research run; this document is the human-readable index.

## Summary

Every requirement theme that cleared the **0.30 frequency threshold** maps cleanly onto an existing authored module (`mod-101` through `mod-112`). The 2026-11 cycle proposes **zero curriculum additions** — the market sample confirms the shape of the curriculum rather than extending it. See the [continuity bias note](./.aicg/curriculum-plan-delta.json) for the rationale.

Watchlist items (below threshold but worth tracking): MCP server authorship in voice-AI platforms (0.10, trending), speech-to-speech / speech-native end-to-end (0.16, waiting on the pipeline-vs-realtime split), and multimodal vision in voice turns (0.03, a notable gap against the role title — separate proposal if it persists two more cycles).

## Role positioning (`req-01`, frequency 1.0)

Every posting in the sample is a hands-on specialist IC who builds production realtime voice or multimodal agents end-to-end. The role shows up under multiple titles: *Voice AI Engineer* (Pallet, timepay.ai, HiringBlaze, NameCoach, Carbon), *AI Engineer, Agentic Voice* (Dialpad, 100MS), *Conversational AI Engineer* (TTEC Digital, Blooming Health), *Speech Software Engineer* (ASAPP), *ML Engineer, Voice & AI* (Freed, Hugging Face), *AI Agent Engineer* (CollectWise, Observe.AI, Prodigal), *(Founding) Forward Deployed Engineer* (Regal.ai, Supermove, Phonely, Prodigal), *Software Engineer, Agent Experience* (AssemblyAI), *Product Engineer* (Hamming, Asendia).

- **Curriculum owner:** [`mod-101-voice-agent-position`](./lessons/mod-101-voice-agent-position/README.md)
- **Evidence (selected):** ASAPP Speech Software Engineer, timepay.ai Voice AI Engineer, CollectWise AI Agent Engineer, Carbon Technology Senior Voice AI Engineer, Weya AI AI Engineer — Voice & Agentic Systems, Hugging Face Senior ML Engineer — Voice Agents.

## Audio & speech foundations (`req-02`, frequency 0.26)

Below threshold but structurally load-bearing. Explicit sample-rate / codec / buffer / noise-suppression fluency is cited by ASAPP ("Strong grasp of audio fundamentals, including sample rates, codecs (Opus, G.711), buffering, packet loss, and jitter"), Hugging Face ("Audio pipeline knowledge: VAD, echo cancellation, jitter buffers, barge-in and turn detection"), timepay.ai ("Strong understanding of real-time systems: WebSockets, WebRTC, audio streaming, codecs and latency"), and Carbon Technology ("Built the full voice stack: STT, LLM reasoning integration, TTS, VAD, turn-taking, and tool calls").

- **Curriculum owner:** [`mod-102-speech-pipeline-foundations`](./lessons/mod-102-speech-pipeline-foundations/README.md)
- **Takeaway:** keep the module; frequency is lower than STT/TTS because postings assume the foundation and cite the higher-layer behavior.

## Speech-to-text production engineering (`req-03`, frequency 0.42)

Cited in 13 of 31 postings. Providers named directly: Deepgram (Weya AI, Carbon Technology, Hugging Face), AssemblyAI (own role at AssemblyAI), Whisper / WhisperX (DeepScribe, Freed), Azure / Google STT (Weya AI). Carbon Technology asks for "Built the full voice stack: STT, LLM reasoning integration, TTS, VAD, turn-taking, and tool calls." ASAPP wants "WER/CER analysis, latency tracking" built into the eval harness. Weya AI requires "hands-on work on STT/TTS providers (e.g. Deepgram, ElevenLabs, Azure, Google) and latency tuning."

- **Curriculum owner:** [`mod-103-stt-production-engineering`](./lessons/mod-103-stt-production-engineering/README.md)
- **Watchlist (below threshold):** Multilingual Indic / Arabic STT surfaced in 3 postings (Weya AI, timepay.ai, HiringBlaze) at 0.10 frequency. Already covered by FLEURS + SeamlessM4T references in `mod-103`; next cycle, consider flagging Indic-language failure-mode examples specifically.

## Text-to-speech production engineering (`req-04`, frequency 0.32)

Cited in 10 of 31 postings. ElevenLabs (Weya AI, Carbon), Cartesia (Carbon), Dialpad in-house ("Own the integration and optimization of multiple TTS vendor APIs behind a unified interface with failover"), AWS Polly / Azure neural voices (Weya AI, Observe.AI). Dialpad surfaces the SSML / prosody craft explicitly: "Apply your knowledge of phonetics and sociolinguistics to format TTS input for maximum naturalness."

- **Curriculum owner:** [`mod-104-tts-production-engineering`](./lessons/mod-104-tts-production-engineering/README.md)
- **Note:** voice-cloning consent surfaced in 2 postings (Dialpad: "voice attributes to the product UI"; NameCoach: "MCP-style integrations for voice AI platforms" + pronunciation-model work). Covered under `mod-104` + `mod-108` (consent / provenance pairing).

## Real-time speech-to-speech & speech-native models (`req-05`, frequency 0.16)

5 of 31 postings explicitly reference speech-to-speech models, native OpenAI Realtime, or Gemini Live: Hugging Face ("architectural ownership of large parts of speech-to-speech: pipeline design, latency budget"), Blooming Health (OpenAI, Anthropic APIs for realtime), TTEC Digital (GECX + Dialogflow + "streaming, low-latency audio-to-audio (A2A) voice agents"), and HiringBlaze (speech-to-speech in Pipecat/LiveKit context).

- **Curriculum owner:** [`mod-105-realtime-speech-to-speech`](./lessons/mod-105-realtime-speech-to-speech/README.md)
- **Watchlist:** pipeline (STT→LLM→TTS) still dominates the sample over end-to-end speech-native. If `req-05` crosses 0.30 next cycle, deepen `mod-105` with a dedicated speech-native-production exercise; not actionable this cycle.

## VAD, turn-taking, and barge-in engineering (`req-06`, frequency 0.32)

Surfaced in 10 of 31 postings (union of VAD and barge-in / turn-taking mentions). timepay.ai: "Handle the hard parts of live conversation: voice activity detection, turn-taking, barge-in, silence, background noise and call transfers." Weya AI: "voice activity detection, barge-in/interruption handling, and turn-taking." CollectWise: "Optimize ASR, STT, TTS, turn-taking, interruption handling, and state transitions." Hugging Face explicitly asks for "VAD, echo cancellation, jitter buffers, barge-in and turn detection." 100MS: "Tune latency, turn-taking, and conversational naturalness for voice AI systems."

- **Curriculum owner:** [`mod-106-vad-turn-taking-barge-in`](./lessons/mod-106-vad-turn-taking-barge-in/README.md)

## Telephony & WebRTC transport engineering (`req-07`, frequency 0.45)

Strongest transport-layer signal in the sample: 14 of 31 postings mention WebRTC, SIP, Twilio, or a specific media server (LiveKit, Pipecat). AgentPhone: "Build and scale core telephony infrastructure: SIP trunking, real-time voice (LiveKit), call routing, and media handling." Carbon Technology: "Experience with SIP/WebRTC and telephony platforms (Twilio) and LiveKit or equivalent." Weya AI: "Integrate and maintain telephony providers (e.g. Twilio, Plivo, Exotel, Vonage, SIP trunks): inbound/outbound calls, call transfer, recordings, DTMF, and media streaming." timepay.ai: "Telephony: SIP, Twilio, Exotel, Plivo or similar; Asterisk or FreeSWITCH." CollectWise: "Experience with SIP trunking, telephony infrastructure, production voice systems."

- **Curriculum owner:** [`mod-107-telephony-webrtc-transport`](./lessons/mod-107-telephony-webrtc-transport/README.md)

## Voice-channel safety, compliance & provenance (`req-08`, frequency 0.32)

10 of 31 postings surface a safety / compliance expectation: HIPAA (DeepScribe ×2, 100MS, Blooming Health — the healthcare-vertical cluster), TCPA / consent (Weya AI: "consent, call recording rules, PII handling, audit trails"; AgentPhone: "A2P 10DLC, TCR registration, KYC"), BFSI / regulated-financial (HiringBlaze, Weya), voice-cloning consent (Dialpad, NameCoach). The EU AI Act and PCI DSS surfaced zero explicit mentions — unsurprising for a US-and-India-heavy sample; `mod-108` covers them for completeness because enterprise RFPs still require them.

- **Curriculum owner:** [`mod-108-safety-compliance-voice`](./lessons/mod-108-safety-compliance-voice/README.md)
- **Cross-track links:** deep MLSec is linked out to [`ai-infra-security-learning`](../ai-infra-security-learning/) / [`security-learning`](../security-learning/); deep risk-engineering to [`ai-risk-engineer-learning`](../ai-risk-engineer-learning/); regulator-facing packet composition to [`ai-evaluation-engineer-learning`](../ai-evaluation-engineer-learning/) (level 35, Governance family).

## Real-time latency budget architecture (`req-09`, frequency 0.32)

10 of 31 postings explicitly target a latency budget. Asendia AI: "real-time phone interviews with sub-300ms latency." 100MS: "Tune latency, turn-taking, and conversational naturalness." ASAPP: "Balance latency vs. quality tradeoffs in streaming speech pipelines to maintain real-time responsiveness." Dialpad: "Minimize time-to-first-audio and end-to-end latency across the ASR → LLM → TTS pipeline." TTEC Digital: "streaming, low-latency audio-to-audio (A2A) voice agents." Weya AI: "latency per turn, call success rates, cost per call, and alerts that fire before a client notices." Hugging Face: "latency budget, and the reliability of the realtime loop."

- **Curriculum owner:** [`mod-109-latency-budget-architecture`](./lessons/mod-109-latency-budget-architecture/README.md)

## Multimodal input design (`req-10`, frequency 0.03)

**Notable gap signal against the role title.** Only 1 of 31 postings explicitly mentioned multimodal vision input in a voice-agent turn (Hugging Face's speech-to-speech role, which is bounded to audio-mostly). The market is overwhelmingly voice-only in 2026-11. The multimodal half of the role title is held by the frontier-realtime APIs (Gemini Live, GPT-4o Realtime) more than by current hiring signals.

- **Curriculum owner:** [`mod-110-multimodal-input-design`](./lessons/mod-110-multimodal-input-design/README.md)
- **Action:** per the continuity bias, keep the module. Removal is out of scope for the standard research packet. If frequency remains ≤0.10 after two more cycles, open a separate proposal to demote `mod-110` and reconsider the "Multimodal" half of the role title.

## Voice-agent orchestration frameworks (`req-11`, frequency 0.48)

15 of 31 postings cite a voice-agent orchestration framework or integration surface. LiveKit (CollectWise, timepay.ai, Weya AI, Carbon, HiringBlaze, AgentPhone, Hugging Face — 7 postings, the dominant framework). Pipecat (timepay.ai, HiringBlaze, Hugging Face). Vapi (Weya AI). The sample also surfaced orchestrator-adjacent in-house work (Dialpad's TTS vendor-failover layer, Phonely's Python state system, telli's distributed call scheduling). Managed platforms (Retell AI, Bland AI, Vocode) appear by name only through adjacent coverage — the market has materially consolidated onto LiveKit + Pipecat as the open-source default plus whatever the frontier Realtime API provides directly.

- **Curriculum owner:** [`mod-111-voice-agent-orchestration`](./lessons/mod-111-voice-agent-orchestration/README.md)
- **Watchlist (below threshold):** MCP server authorship by voice-AI platforms at 0.10 — AssemblyAI builds an agent-skill library and extends an MCP server; AgentPhone ships an MCP server alongside telephony; NameCoach builds MCP-style integrations. If this crosses 0.30 next cycle, author a `mod-111` exercise on authoring voice-platform MCP servers (session lifecycle, tool contracts, realtime vs. async tool surface).

## Voice + multimodal evaluation & observability (`req-12`, frequency 0.58)

Highest-frequency single requirement in the sample — 18 of 31 postings cite evals, regression testing, observability, or eval-gated deployment. ASAPP: "Define and implement speech quality evaluation frameworks, including WER/CER analysis, latency tracking, and perceptual TTS metrics." Weya AI: "automated scoring of agent conversations, prompt regression testing." Hamming AI (an entire voice-agent-QA product company): "voice-agent QA: test creation, scenario management, evaluation results." Asendia AI: "Run evals and reliability work: audit agents across a live fleet, catch regressions." 100MS: "Run systematic LLM evaluations, track regressions, and ensure models meet quality bars." AgentPhone: "Set the engineering foundations (testing, CI/CD, observability, on-call)." Weya AI again: "Instrument everything: latency per turn, call success rates, cost per call."

- **Curriculum owner:** [`mod-112-voice-multimodal-eval-and-observability`](./lessons/mod-112-voice-multimodal-eval-and-observability/README.md)
- **Cross-track link:** application-layer eval methodology at depth is owned by [`ai-eval-engineer-learning`](../ai-eval-engineer-learning/) (level 30); release-assurance methodology is owned by [`ai-evaluation-engineer-learning`](../ai-evaluation-engineer-learning/) (level 35, Governance family).

## Peer / lower-level requirements (linked out; this track does not teach at depth)

| Req | Theme | Freq | Primary owner | External resources |
|---|---|---|---|---|
| `req-13` | Inference serving depth for audio workloads — streaming batching, KV-cache for long turns, GPU per-stream budget, quantisation | 0.06 | [`ai-infra-inference-engineer`](../ai-infra-inference-engineer-learning/) (level 35) | [vLLM](https://github.com/vllm-project/vllm), [TGI](https://github.com/huggingface/text-generation-inference), [SGLang](https://github.com/sgl-project/sglang), [WhisperX](https://github.com/m-bain/whisperX) |
| `req-14` | Release-assurance methodology / regulator-facing packets for voice | 0.03 | [`ai-evaluation-engineer`](../ai-evaluation-engineer-learning/) (level 35) | [NIST AI RMF 1.0 + GenAI Profile](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-1.pdf), [EU AI Act Regulation 2024/1689](https://eur-lex.europa.eu/eli/reg/2024/1689/oj) |
| `req-15` | Deep ML/AI security at platform scale including audio-domain adversarial defence | 0.03 | [`ai-infra-security`](../ai-infra-security-learning/) / [`security-learning`](../security-learning/) (level 35) | [MITRE ATLAS](https://atlas.mitre.org/), [OWASP LLM Top 10 (2025)](https://owasp.org/www-project-top-10-for-large-language-model-applications/), [NIST AI 100-2 (2025)](https://csrc.nist.gov/publications/detail/nistir/8269/draft) |
| `req-16` | Applied LLM feature engineering craft (prompts, structured outputs, caching, text-LLM streaming, text-level cost/latency SLOs) | 0.71 | [`applied-ai-engineer`](../applied-ai-engineer-learning/) (level 30) | [Chip Huyen — AI Engineering](https://www.oreilly.com/library/view/ai-engineering/9781098166298/), [Eugene Yan — LLM Patterns](https://eugeneyan.com/writing/llm-patterns/) |
| `req-17` | Deep agent-framework internals (LangGraph, OpenAI Agents SDK, Claude Agent SDK, MCP, tool-calling patterns) | 0.42 | [`agentic-ai-engineer`](../agentic-ai-engineer-learning/) (level 30) | [Anthropic — Building effective agents](https://www.anthropic.com/research/building-effective-agents), [MCP spec](https://modelcontextprotocol.io/) |
| `req-18` | Application-layer evaluation methodology (traces, judges, RAG eval, eval-gated CI/CD, online eval) | 0.55 | [`ai-eval-engineer`](../ai-eval-engineer-learning/) (level 30) | [Langfuse](https://langfuse.com/docs), [LangSmith](https://docs.smith.langchain.com/), [Arize Phoenix](https://docs.arize.com/phoenix), [Braintrust](https://www.braintrust.dev/docs), [Weave](https://wandb.ai/site/weave) |
| `req-19` | Fine-tune workflows for voice — ASR domain adaptation, voice-clone fine-tunes, speech-LLM fine-tunes | 0.13 | [`fine-tuning-engineer`](../fine-tuning-engineer-learning/) / [`training-pipeline-engineer`](../training-pipeline-engineer-learning/) (level 30 / 35) | [XTTS](https://arxiv.org/abs/2406.04904), [openai/whisper](https://github.com/openai/whisper) |
| `req-20` | Build leadership, shared-infra authorship, multi-agent architecture, management scope | 0.00 | [`senior-agentic-ai-engineer`](../senior-agentic-ai-engineer-learning/) / [`ai-engineering-manager`](../ai-engineering-manager-learning/) / [`agentic-systems-architect`](../agentic-systems-architect-learning/) (level 40–48) | — (filtered out by strict IC filter) |
| `req-21` | Production Python + TypeScript, async + streaming + WebSocket + WebRTC, FastAPI/Node, tests, CI, Docker, IaC, SQL | 0.90 | [`ai-infra-junior-engineer`](../ai-infra-junior-engineer-learning/) (level 10) | See [`PREREQUISITES.md`](./PREREQUISITES.md) |
| `req-22` | Classical ML + PyTorch + Hugging Face Transformers fluency at applied depth | 0.19 | [`ml-engineer`](../ml-engineer-learning/) (level 20) | See [`PREREQUISITES.md`](./PREREQUISITES.md) |

## Emerging themes below the 0.30 threshold (watchlist, not actionable this cycle)

See [`emerging_themes_below_threshold`](./.aicg/job-requirements.json) in the raw file for the structured list. Summary:

- **MCP server authorship by voice-AI platforms** (0.10) — AssemblyAI, AgentPhone, NameCoach. If ≥0.30 next cycle, author a `mod-111` exercise.
- **Multilingual voice — Indic, Arabic, Hinglish** (0.10) — Weya AI, timepay.ai, HiringBlaze. Already covered by FLEURS + Speechmatics references in `mod-103`; consider surfacing Indic failure-mode examples specifically.
- **Voice-agent QA / regression tooling** (0.03) — Hamming AI (an entire company around this). Covered by `mod-112`.
- **Voice model fine-tuning requested by the voice-agent engineer** (0.13) — DeepScribe, Freed, Carbon, 100MS. The when-to-fine-tune decision stays with this track; the training work is linked out to [`fine-tuning-engineer`](../fine-tuning-engineer-learning/) / [`training-pipeline-engineer`](../training-pipeline-engineer-learning/).
- **Call-center-grade production scale** (0.06) — ASAPP, CollectWise. Covered by `mod-107` / `mod-109`.
- **WhatsApp / omnichannel agent extension** (0.10) — Weya AI, timepay.ai, AgentPhone. Chat-channel craft is owned by [`applied-ai-engineer`](../applied-ai-engineer-learning/); this track owns the voice-channel-specific pieces only.

## Salary signal (partial — 23 of 31 postings disclose)

US IC Voice & Multimodal Agent Engineer base (n=16): p25 ≈ $135K, median ≈ $170K, p75 ≈ $215K, p90 ≈ $250K. Frontier-adjacent Senior IC (Hugging Face, Carbon, Blooming Health, Freed; n=4): $170K–$270K base + equity. India (n=3): ₹20L–₹50L. Berlin (n=2): €75K–€160K. Vancouver (Dialpad, n=1): CAD $161.5K–$191.5K.

Full per-posting verbatim ranges in [`salary_summary.ranges_verbatim`](./.aicg/job-requirements.json).

## Ownership rule

Primary coverage is assigned to the lowest-level role where a skill is genuinely required. Higher-level tracks link to the owner unless additional depth, architectural context, or leadership scope is required. See [`.aicg/job-requirements.json#ownership_rule`](./.aicg/job-requirements.json) for the full role-boundary narrative.

## How this file was produced

WebSearch + WebFetch fan-out across 10+ job boards (Greenhouse, Lever, Ashby, Workable, Wellfound, YC Companies, Built In, Himalayas, Workday, direct careers pages) and 60+ employer categories (voice-AI pure-plays, realtime orchestration platforms, telephony providers, frontier labs, applied-voice product cos, contact-centre AI, in-vehicle / smart home). Postings normalized into `.aicg/job-requirements.json`. Frequencies computed across the full 31-posting sample. Excluded-posting ledger (30+ URLs including Deepgram, Decagon, LiveKit, Sierra, Simple AI, Cresta, ElevenLabs, Cartesia, OpenAI Applied Voice, xAI Voice MTS — all JS-rendered pages the WebFetch pipeline could not parse) stored in [`.aicg/job-requirements.json#excluded_postings`](./.aicg/job-requirements.json).

See [`.aicg/research/runs/voice-agent-engineer/raw/postings.json`](./.aicg/research/runs/voice-agent-engineer/raw/postings.json) for the raw subagent output (postings + excluded + search log).
