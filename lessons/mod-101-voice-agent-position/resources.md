# Resources for mod-101-voice-agent-position

Primary sources only. Use these to verify claims and to anchor your landscape read in official documentation rather than secondary coverage.

## Landscape — vendor / lab documentation

### Voice-AI pure-plays

- **Deepgram** — [docs.deepgram.com](https://developers.deepgram.com/docs/). Streaming ASR API, Nova model family, domain-adaptation features.
- **AssemblyAI** — [www.assemblyai.com/docs](https://www.assemblyai.com/docs/). Universal-Streaming and async STT.
- **ElevenLabs** — [elevenlabs.io/docs](https://elevenlabs.io/docs). TTS and Conversational AI product.
- **Cartesia** — [docs.cartesia.ai](https://docs.cartesia.ai/). Sonic streaming TTS.
- **LMNT** — [docs.lmnt.com](https://docs.lmnt.com/). Streaming TTS.
- **Hume AI** — [dev.hume.ai](https://dev.hume.ai/). Empathic Voice Interface (EVI) and expression measurement.

### Realtime-voice orchestration platforms

- **LiveKit Agents** — [docs.livekit.io/agents](https://docs.livekit.io/agents/). WebRTC infrastructure and the `livekit-agents` Python / Node framework (STT / TTS / LLM plugins, VAD, turn detection).
- **Pipecat** — [github.com/pipecat-ai/pipecat](https://github.com/pipecat-ai/pipecat). Daily's open-source voice and multimodal agent framework.
- **Daily** — [docs.daily.co](https://docs.daily.co/). WebRTC infrastructure provider.
- **Vocode** — [docs.vocode.dev](https://docs.vocode.dev/). Open-source voice-LLM library and hosted platform.
- **Retell AI** — [docs.retellai.com](https://docs.retellai.com/).
- **Vapi** — [docs.vapi.ai](https://docs.vapi.ai/).
- **Bland AI** — [docs.bland.ai](https://docs.bland.ai/).

### Telephony providers with voice-AI surfaces

- **Twilio Programmable Voice** — [www.twilio.com/docs/voice](https://www.twilio.com/docs/voice). Media Streams and ConversationRelay.
- **Vonage Voice API** — [developer.vonage.com/en/voice/voice-api/overview](https://developer.vonage.com/en/voice/voice-api/overview). WebSocket audio streaming.
- **Zoom Developer Platform** — [developers.zoom.us](https://developers.zoom.us/). Zoom Phone and Contact Center APIs.

### Frontier labs with realtime voice stacks

- **OpenAI Realtime API** — [platform.openai.com/docs/guides/realtime](https://platform.openai.com/docs/guides/realtime). WebRTC and WebSocket transports, native speech-to-speech models.
- **Google Gemini Live API** — [ai.google.dev/gemini-api/docs/live](https://ai.google.dev/gemini-api/docs/live). Bidirectional streaming multimodal API.
- **Google Cloud Speech-to-Text / Text-to-Speech** — [cloud.google.com/speech-to-text](https://cloud.google.com/speech-to-text), [cloud.google.com/text-to-speech](https://cloud.google.com/text-to-speech).
- **Anthropic Claude API** — [docs.claude.com/en/api/overview](https://docs.claude.com/en/api/overview). Claude is used inside voice agents via partner relays (ElevenLabs Conversational AI, LiveKit integrations, Pipecat); check current docs for any first-party realtime voice surface.
- **Meta AI research — Seamless Communication** — [github.com/facebookresearch/seamless_communication](https://github.com/facebookresearch/seamless_communication). SeamlessM4T multilingual speech translation.
- **Kyutai — Moshi** — [github.com/kyutai-labs/moshi](https://github.com/kyutai-labs/moshi), paper [arxiv.org/abs/2410.00037](https://arxiv.org/abs/2410.00037). Open-weight speech-to-speech foundation model and Mimi audio codec.

### Applied-voice product companies

- **Sierra** — [sierra.ai](https://sierra.ai/).
- **Decagon** — [decagon.ai](https://decagon.ai/).
- **Cresta** — [cresta.com](https://cresta.com/).
- **PolyAI** — [poly.ai](https://poly.ai/).
- **Observe.AI** — [observe.ai](https://www.observe.ai/).
- **Replicant** — [replicant.com](https://www.replicant.com/).

### Vertical-voice applications

- **Suki** — [suki.ai](https://www.suki.ai/). Clinical ambient AI assistant.
- **Microsoft / Nuance DAX** — [www.nuance.com/healthcare/dragon-ai-clinical-solutions/dax-copilot.html](https://www.nuance.com/healthcare/dragon-ai-clinical-solutions/dax-copilot.html). Dragon Ambient eXperience Copilot.
- **Abridge** — [abridge.com](https://www.abridge.com/).
- **Ambience Healthcare** — [ambiencehealthcare.com](https://www.ambiencehealthcare.com/).
- **Gong** — [gong.io](https://www.gong.io/). Sales conversation intelligence.
- **Rilla** — [rilla.com](https://www.rilla.com/).
- **Sybill** — [sybill.ai](https://www.sybill.ai/).

## Standards and protocols

- **WebRTC** — [W3C WebRTC 1.0 specification](https://www.w3.org/TR/webrtc/); [MDN WebRTC API overview](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API).
- **Opus audio codec** — [RFC 6716](https://datatracker.ietf.org/doc/html/rfc6716).
- **µ-law / A-law companding (G.711)** — [ITU-T G.711](https://www.itu.int/rec/T-REC-G.711).
- **SIP** — [RFC 3261](https://datatracker.ietf.org/doc/html/rfc3261).
- **E.164 numbering plan** — [ITU-T E.164](https://www.itu.int/rec/T-REC-E.164).
- **STIR/SHAKEN caller-ID attestation** — [RFC 8224](https://datatracker.ietf.org/doc/html/rfc8224), [RFC 8225](https://datatracker.ietf.org/doc/html/rfc8225), [RFC 8226](https://datatracker.ietf.org/doc/html/rfc8226); [FCC STIR/SHAKEN overview](https://www.fcc.gov/call-authentication).

## Compliance references

- **HIPAA** — [HHS HIPAA for Professionals](https://www.hhs.gov/hipaa/for-professionals/index.html). Relevant for clinical voice applications.
- **TCPA** — [FCC Telephone Consumer Protection Act rules](https://www.fcc.gov/general/telemarketing-and-robocalls). Relevant for US outbound voice.
- **GDPR** — [gdpr.eu](https://gdpr.eu/) and the [EDPB guidance pages](https://www.edpb.europa.eu/edpb_en). Relevant for EU voice data; voice can constitute biometric data.
- **Biometric anti-spoofing / presentation-attack detection** — [ISO/IEC 30107 family](https://www.iso.org/standard/79520.html).
- **State-level recording consent (US)** — jurisdiction-dependent; consult your legal counsel. Starting point: [National Conference of State Legislatures — Electronic Surveillance Laws](https://www.ncsl.org/technology-and-communication/electronic-surveillance-laws).

## Role / ladder references (within this curriculum ecosystem)

- [Applied AI Engineer (L30)](https://github.com/ai-engineering-curriculum/applied-ai-engineer-learning)
- [Agentic AI Engineer (L30)](https://github.com/ai-engineering-curriculum/agentic-ai-engineer-learning)
- [AI Evaluation Engineer (L30)](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning)
- [RAG & Retrieval Engineer (L30)](https://github.com/ai-engineering-curriculum/rag-engineer-learning)
- [Forward Deployed AI Engineer (L35)](https://github.com/ai-engineering-curriculum/forward-deployed-engineer-learning)
- [AI Inference Engineer (L35)](https://github.com/AI-Infra-Curriculum/ai-infra-inference-engineer-learning)
- [Senior Agentic AI Engineer (L40)](https://github.com/ai-engineering-curriculum/senior-agentic-ai-engineer-learning)
- [Agent Platform Engineer](https://github.com/ai-engineering-curriculum/agent-platform-engineer-learning)

## Notes on citation hygiene

- Vendor product pages change frequently. When you cite a specific feature, link to the dated docs or archive the page.
- Benchmarks (WER, MOS, TTFT) in vendor blog posts are not peer-reviewed. Treat them as marketing until you can reproduce them on your own corpus.
- "Realtime" has no universal threshold. In this module we use ~800 ms – 1.5 s end-to-end response as a conversational-feel range, which matches order-of-magnitude figures cited across the orchestration platforms above — not a hard SLA.
