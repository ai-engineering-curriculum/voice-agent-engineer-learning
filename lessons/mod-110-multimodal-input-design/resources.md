# Resources for mod-110-multimodal-input-design

Primary sources only. Use these to verify provider shapes, benchmark methodology, and privacy / compliance requirements against official documentation rather than blog posts. Dates matter in this module — provider APIs, cache semantics, and model capabilities move quarterly; cite your access date when you build on any specific capability.

## Provider documentation: image / document vision APIs

### OpenAI (GPT-4o family)

- **Vision guide** — [platform.openai.com/docs/guides/vision](https://platform.openai.com/docs/guides/vision). Image input in chat completions, URL vs. inline, detail levels, token math.
- **Chat Completions API reference** — [platform.openai.com/docs/api-reference/chat](https://platform.openai.com/docs/api-reference/chat). The `image_url` content part.
- **Realtime API** — [platform.openai.com/docs/guides/realtime](https://platform.openai.com/docs/guides/realtime). Voice-loop surface; check for current image / vision input support in sessions.
- **Prompt caching** — [platform.openai.com/docs/guides/prompt-caching](https://platform.openai.com/docs/guides/prompt-caching). Automatic prefix caching behaviour.
- **Files API** — [platform.openai.com/docs/api-reference/files](https://platform.openai.com/docs/api-reference/files). Storing images for re-use.

### Anthropic (Claude vision)

- **Vision guide** — [docs.claude.com/en/docs/build-with-claude/vision](https://docs.claude.com/en/docs/build-with-claude/vision). Image blocks, base64 vs. URL sources, image token math.
- **Messages API** — [docs.claude.com/en/api/messages](https://docs.claude.com/en/api/messages). Request and response shape.
- **PDF / document support** — [docs.claude.com/en/docs/build-with-claude/pdf-support](https://docs.claude.com/en/docs/build-with-claude/pdf-support). Multi-page document block.
- **Prompt caching** — [docs.claude.com/en/docs/build-with-claude/prompt-caching](https://docs.claude.com/en/docs/build-with-claude/prompt-caching). `cache_control: ephemeral` semantics and image cacheability. <!-- needs-research: confirm current cache TTLs and image cacheability at access date -->

### Google (Gemini vision + Gemini Live)

- **Gemini vision** — [ai.google.dev/gemini-api/docs/vision](https://ai.google.dev/gemini-api/docs/vision). Image inputs, `inline_data` vs. `file_data`.
- **`generateContent` reference** — [ai.google.dev/api/generate-content](https://ai.google.dev/api/generate-content). Request / response shape.
- **Files API** — [ai.google.dev/gemini-api/docs/files](https://ai.google.dev/gemini-api/docs/files). Upload and reference files for ~48 hours.
- **Context caching** — [ai.google.dev/gemini-api/docs/caching](https://ai.google.dev/gemini-api/docs/caching). Explicit cached-content API.
- **Multimodal Live API** — [ai.google.dev/gemini-api/docs/live](https://ai.google.dev/gemini-api/docs/live). Bidirectional WebSocket, audio-in, video-in, tool calling.
- **Vertex AI Gemini** — [cloud.google.com/vertex-ai/generative-ai/docs/multimodal/overview](https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/overview). Enterprise / data-residency path for Gemini multimodal.

### Azure OpenAI

- **Azure OpenAI Service** — [learn.microsoft.com/en-us/azure/ai-services/openai](https://learn.microsoft.com/en-us/azure/ai-services/openai/). Regional deployment and GPT-4o vision via Azure. Useful when data-residency drives the deployment choice.

### Azure / Google / AWS document AI (applied-AI-side)

- **Azure Document Intelligence** — [learn.microsoft.com/en-us/azure/ai-services/document-intelligence](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/). Prebuilt and custom document extraction with bounding boxes.
- **Google Document AI** — [cloud.google.com/document-ai](https://cloud.google.com/document-ai). OCR + structured extraction.
- **AWS Textract** — [aws.amazon.com/textract](https://aws.amazon.com/textract/). OCR, forms, tables with bounding boxes.
- **Tesseract OCR** — [github.com/tesseract-ocr/tesseract](https://github.com/tesseract-ocr/tesseract). Open-source OCR; useful for the audit-log cross-check in exercise-04.

## Browser / client APIs for screen share and camera

- **W3C Media Capture and Streams** — [w3.org/TR/mediacapture-streams](https://www.w3.org/TR/mediacapture-streams/). `getUserMedia()` and related.
- **W3C Screen Capture** — [w3.org/TR/screen-capture](https://www.w3.org/TR/screen-capture/). `getDisplayMedia()` for screen share.
- **W3C MediaStreamTrack Insertable Streams** — [w3.org/TR/mediacapture-transform](https://www.w3.org/TR/mediacapture-transform/). For in-browser frame processing.
- **MDN — `getDisplayMedia`** — [developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getDisplayMedia](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getDisplayMedia).
- **MDN — `CropTarget` and element-level capture** — [developer.mozilla.org/en-US/docs/Web/API/CropTarget](https://developer.mozilla.org/en-US/docs/Web/API/CropTarget). Limits what the user can share to a specific element.

## Multimodal benchmarks

- **MMMU (Massive Multi-discipline Multimodal Understanding and Reasoning)** — [mmmu-benchmark.github.io](https://mmmu-benchmark.github.io/), [arXiv:2311.16502](https://arxiv.org/abs/2311.16502), [Hugging Face dataset](https://huggingface.co/datasets/MMMU/MMMU). College-level multimodal reasoning across six disciplines.
- **MMMU-Pro** — [arXiv:2409.02813](https://arxiv.org/abs/2409.02813). Harder, vision-only follow-up.
- **MM-Vet** — [github.com/yuweihao/MM-Vet](https://github.com/yuweihao/MM-Vet), [arXiv:2308.02490](https://arxiv.org/abs/2308.02490). Integrated multimodal capabilities (recognition, OCR, knowledge, language, spatial, math) with an LLM-as-judge grading protocol.
- **MM-Vet v2** — [github.com/yuweihao/MM-Vet-v2](https://github.com/yuweihao/MM-Vet-v2), [arXiv:2408.00765](https://arxiv.org/abs/2408.00765). Expanded capability coverage.
- **LLaVA-OneVision** — [arXiv:2408.03326](https://arxiv.org/abs/2408.03326), [llava-vl.github.io](https://llava-vl.github.io/). Model family + evaluation harness across single-image, multi-image, and video tasks.
- **ScienceQA** — [scienceqa.github.io](https://scienceqa.github.io/). Multimodal science question answering; complementary to MMMU for science domains.
- **ChartQA** — [github.com/vis-nlp/ChartQA](https://github.com/vis-nlp/ChartQA). Chart understanding and question answering.
- **DocVQA** — [rrc.cvc.uab.es/?ch=17](https://rrc.cvc.uab.es/?ch=17). Document visual question answering.

## Open-weight multimodal models (for self-hosted / NLP-engineer boundary reference)

- **LLaVA (Large Language and Vision Assistant)** — [github.com/haotian-liu/LLaVA](https://github.com/haotian-liu/LLaVA), [arXiv:2304.08485](https://arxiv.org/abs/2304.08485). Foundational open multimodal model family.
- **LLaVA-OneVision** — [arXiv:2408.03326](https://arxiv.org/abs/2408.03326). Multi-image and video extensions.
- **Qwen-VL / Qwen2-VL** — [github.com/QwenLM/Qwen-VL](https://github.com/QwenLM/Qwen-VL), [arXiv:2409.12191](https://arxiv.org/abs/2409.12191). Open-weight multimodal from Alibaba.
- **InternVL** — [github.com/OpenGVLab/InternVL](https://github.com/OpenGVLab/InternVL). Open-weight multimodal from OpenGVLab.
- **PaliGemma** — [ai.google.dev/gemma/docs/paligemma](https://ai.google.dev/gemma/docs/paligemma). Open-weight vision-language model from Google.
- **Idefics** — [huggingface.co/HuggingFaceM4](https://huggingface.co/HuggingFaceM4). Open-weight multimodal from Hugging Face.

## Realtime / multimodal live surfaces

- **OpenAI Realtime API** — [platform.openai.com/docs/guides/realtime](https://platform.openai.com/docs/guides/realtime). Voice-loop surface with tool-call support.
- **Google Gemini Live** — [ai.google.dev/gemini-api/docs/live](https://ai.google.dev/gemini-api/docs/live). Bidirectional multimodal (audio + video + text) streaming.
- **Pipecat** — [docs.pipecat.ai](https://docs.pipecat.ai/), [github.com/pipecat-ai/pipecat](https://github.com/pipecat-ai/pipecat). Processor-pipeline framework with vision-input processors.
- **LiveKit Agents** — [docs.livekit.io/agents](https://docs.livekit.io/agents/). Streaming agent framework that supports multimodal input processors.

## Privacy, consent, and compliance

- **W3C Permissions API** — [w3.org/TR/permissions](https://www.w3.org/TR/permissions/). Browser-level permission model for camera / microphone.
- **GDPR (Regulation (EU) 2016/679)** — [gdpr-info.eu](https://gdpr-info.eu/). Lawful basis, specific consent, retention requirements. Applies to images of people and screen content containing personal data.
- **HIPAA Security Rule** — [hhs.gov/hipaa](https://www.hhs.gov/hipaa/for-professionals/security/index.html). Covers PHI in multimodal content; BAAs with model providers.
- **PCI-DSS v4.0** — [pcisecuritystandards.org](https://www.pcisecuritystandards.org/document_library/). Cardholder-data handling; strongly implicated when screen share can capture payment pages.
- **NIST SP 800-66 (HIPAA Security Rule implementation)** — [csrc.nist.gov/pubs/sp/800/66/r2/final](https://csrc.nist.gov/pubs/sp/800/66/r2/final). Operational guidance for PHI protection.
- Cross-reference to [mod-108](../mod-108-safety-compliance-voice/README.md) for the broader safety and compliance framing that vision inherits.

## Eval & observability

- Cross-reference to [mod-112 — Voice and Multimodal Eval and Observability](../mod-112-voice-multimodal-eval-and-observability/README.md) for the broader eval harness that this module's exercises plug into.
- **OpenTelemetry Semantic Conventions for Generative AI** — [opentelemetry.io/docs/specs/semconv/gen-ai](https://opentelemetry.io/docs/specs/semconv/gen-ai/). The `gen_ai.*` attribute namespace; useful for tagging multimodal LLM calls on top of the base conventions.

## Role / ladder references (within this curriculum ecosystem)

- [Voice & Multimodal Agent Engineer (L35) — this track](../../README.md).
- [Applied AI Engineer (L30)](https://github.com/ai-engineering-curriculum/applied-ai-engineer-learning) — peer role; Chapter 7 defines the boundary for in-house image / document feature engineering.
- [NLP Engineer (L30)](https://github.com/ai-engineering-curriculum/nlp-engineer-learning) — peer role; Chapter 7 defines the boundary for deep multimodal-model research.
- [AI Infra Inference Engineer (L35)](https://github.com/AI-Infra-Curriculum/ai-infra-inference-engineer-learning) — peer role on serving-layer tuning; relevant when multimodal inference latency is a serving problem (mod-109 Chapter 6).

## Citation hygiene

- **Provider docs change monthly.** Date every citation that depends on a specific API shape, cache TTL, or pricing number.
- **Benchmark numbers are only comparable within the same version.** MMMU v1 and MMMU-Pro are different benchmarks; MM-Vet and MM-Vet v2 are different; record which version and judge you used.
- **"Realtime" and "live" are not SLOs.** When a provider ships a "live" or "realtime" multimodal API, the latency properties are measurable but not guaranteed; measure in your region on your traffic before quoting a number.
- **Privacy and compliance are not features.** Ticking a GDPR / HIPAA / PCI box in a vendor comparison matrix is not the same as holding a Data Processing Agreement or a Business Associate Agreement; cite the agreement, not the matrix.
- **Model identifiers are not model versions.** A provider's `gpt-4o` or `claude-sonnet-4` points at a specific snapshot that can change; record the precise version string you tested against for benchmark results.
