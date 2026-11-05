# 07. Open-Weights Speech Foundation Models: Moshi, Spirit LM, Qwen2-Audio, SeamlessM4T

## Why this chapter

The managed speech-native stacks (OpenAI Realtime, Gemini Live) are the fastest path to a shipping voice agent. They are not the only path, and they are not the right path for every deployment. Four pressures push a team toward open-weights speech foundation models:

1. **Compliance.** Air-gapped deployment, on-prem inference, or region-locked data where even a VPC-hosted managed vendor won't clear.
2. **Cost at scale.** At sufficient volume, the per-minute managed rate crosses the break-even with running the inference yourself.
3. **Capability vendors do not ship.** Full-duplex co-articulation (Moshi); joint text-and-speech decoding in one model (Spirit LM); audio **understanding** without synthesis (Qwen2-Audio); speech-to-speech translation at the matrix scale of 100+ languages (SeamlessM4T).
4. **Research cadence.** A team that needs to fine-tune on domain audio, inspect the model, or publish papers has a hard ceiling on what managed APIs allow.

This chapter is the working survey: what each open-weights family does, where it came from (the research heritage you should know by name), and the decision rule for when open-weights is a **defensible** compliance / cost choice — not merely a possible one.

The chapter deliberately stays architectural, not implementation. A full fine-tuning pipeline for Moshi or a deployment stack for Qwen2-Audio would eat the chapter and the module. The level of detail here is "pick the right model; know why; know the trade-off."

## The research heritage you should know

Modern open-weights speech foundation models are built on three threads that trace back through the late 2010s and early 2020s. Hold these; every model in the survey is a descendant.

### AudioLM and the neural-audio-codec paradigm

- **AudioLM** — [Borsos et al., 2022](https://arxiv.org/abs/2209.03143). Google Research's reframing of audio generation as a language-modelling problem on **discrete audio tokens** produced by a neural codec. The AudioLM paper demonstrated long-horizon coherent audio generation (speech, piano) from short prompts.
- **SoundStream** — [Zeghidour et al., 2021](https://arxiv.org/abs/2107.03312). The neural audio codec AudioLM used. Encodes 24 kHz audio into discrete tokens at bitrates as low as 3 kbit/s while remaining perceptually intelligible. The architecture (RVQ — residual vector quantisation) is the direct ancestor of every codec in use today.
- **EnCodec** — [Défossez et al., 2022](https://arxiv.org/abs/2210.13438). Meta's neural codec; RVQ-based, bitrate-tunable, the codec behind MusicGen and AudioGen.
- **DAC (Descript Audio Codec)** — [Kumar et al., 2023](https://arxiv.org/abs/2306.06546). Higher-fidelity RVQ codec; often the preferred option for modern speech-LM work.
- **Mimi** — [Kyutai, 2024](https://arxiv.org/abs/2410.00037). Kyutai's neural codec paired with Moshi; ≈ 1.1 kbit/s, 12.5 Hz frame rate, specifically designed for low-latency speech-LM use.

**The architectural invariant.** Every open-weights speech-LM below either produces or consumes **discrete audio tokens** from one of these codecs. Understanding the codec–LM pairing is understanding the model.

### Self-supervised speech representations

- **wav2vec 2.0** — [Baevski et al., 2020](https://arxiv.org/abs/2006.11477).
- **HuBERT** — [Hsu et al., 2021](https://arxiv.org/abs/2106.07447).
- **WavLM** — [Chen et al., 2021](https://arxiv.org/abs/2110.13900).

These are not speech-LMs; they are **encoders**. They produce dense continuous embeddings of speech, used as features inside downstream models. Spirit LM uses HuBERT-style representations under the hood; Qwen2-Audio uses Whisper's encoder; SeamlessM4T's speech side inherits from this family.

### Speech translation and the SeamlessM4T lineage

- **SeamlessM4T v2** — [Barrault et al., 2023](https://arxiv.org/abs/2312.05187). Meta's speech-to-speech translation model, descending from mBART, wav2vec 2.0, and the UnitY2 architecture.

Translation sits one step adjacent to voice-agent work — a global voice agent is often a translation agent under the hood — and SeamlessM4T is the open-weights reference.

## The survey

Six open-weights families worth knowing by name.

### 1. Kyutai Moshi + Mimi

- **Paper:** [Défossez et al., 2024](https://arxiv.org/abs/2410.00037).
- **Code:** [github.com/kyutai-labs/moshi](https://github.com/kyutai-labs/moshi).
- **What it is:** a **full-duplex speech-language model**. Audio tokens in, audio tokens out, at 12.5 Hz on parallel streams. Chapter 5 is the architectural read; this chapter is the "when do you pick it" read.
- **Modalities:** audio in, audio out. The hidden inner-monologue text stream is a scaffold, not an output.
- **Strengths:** sub-200-ms response latency; native backchannels and co-articulated turn-taking; open weights, fully inspectable; MIT-license-equivalent terms (verify at integration time).
- **Limitations:** English-first (multilingual variants exist but coverage is thin); the model is not a strong tool-caller; voice selection is baked into the model.
- **When to pick:** research, demos, deployments that need full-duplex and can self-host; any workflow where audio data cannot leave your infrastructure.

### 2. Meta Spirit LM

- **Paper:** [Nguyen et al., 2024](https://arxiv.org/abs/2402.05755).
- **Project page:** [speechbot.github.io/spiritlm](https://speechbot.github.io/spiritlm/).
- **What it is:** a **cross-modal speech-and-text language model**. Spirit LM interleaves text tokens and speech-unit tokens (HuBERT-derived) in a single sequence; the model can switch between text and speech at any point in the sequence.
- **Modalities:** text + speech bidirectional. You can prompt in text and get a speech continuation; prompt in speech and get a text continuation; or mix.
- **Strengths:** the cleanest demonstration that speech and text can share a token space; preserves expressive prosody (the "Spirit LM Expressive" variant).
- **Limitations:** research-grade; not a production voice-agent model. Smaller model sizes; weaker tool use.
- **When to pick:** when you need the specific capability of treating speech and text symmetrically — e.g., a transcription → reasoning → synthesis flow inside a single model, or a joint text/speech reasoning task.

### 3. Qwen2-Audio

- **Paper:** [Chu et al., 2024](https://arxiv.org/abs/2407.10759).
- **Model:** [`Qwen/Qwen2-Audio-7B-Instruct`](https://huggingface.co/Qwen/Qwen2-Audio-7B-Instruct).
- **What it is:** an **audio-understanding LLM**. Audio in, text out. Qwen2-Audio accepts speech, music, and environmental audio as input (via a Whisper-based audio encoder) and reasons about it in text.
- **Modalities:** audio in, text out. It does **not** synthesise audio; that is a separate stage.
- **Strengths:** strong audio understanding for a 7B-size model; can handle speech + non-speech audio in one call; permissive licence (verify at integration time); cheap to self-host.
- **Limitations:** no audio output — Qwen2-Audio is a listening model, not a speaking one. Pair with a TTS leg for full agent loop.
- **When to pick:** as a replacement for the STT leg in a pipeline, where "understand what the user said including non-speech context" matters; as an audio ingest for an agentic workflow where the model's reply does not need to be spoken; as the input half of a hybrid speech-native-ish architecture.

### 4. SeamlessM4T v2

- **Paper:** [Barrault et al., 2023](https://arxiv.org/abs/2312.05187).
- **Model:** [`facebook/seamless-m4t-v2-large`](https://huggingface.co/facebook/seamless-m4t-v2-large).
- **Project page:** [ai.meta.com/research/seamless-communication](https://ai.meta.com/research/seamless-communication/).
- **What it is:** a **unified speech / text translation and transcription model**. Four task heads: speech-to-speech translation, speech-to-text translation, text-to-speech translation, text-to-text translation. Trained on 100+ source languages and 36+ speech-output languages.
- **Modalities:** speech and text in both directions, across languages.
- **Strengths:** the open-weights reference for speech-to-speech translation; preserves speaker characteristics across languages (SeamlessExpressive variant); production-ready licence.
- **Limitations:** not a conversational agent — it translates, it does not reason about tools; latency is high compared to a dedicated translation stack.
- **When to pick:** multilingual voice-agent deployments where translation is part of the loop; globalised customer-service agents; accessibility applications; any workflow where "the user and the model can speak different languages" is the product.

### 5. Whisper and Whisper-derived speech encoders

- **Paper:** [Radford et al., 2022](https://arxiv.org/abs/2212.04356).
- **Model family:** [`openai/whisper-large-v3`](https://huggingface.co/openai/whisper-large-v3), [`openai/whisper-large-v3-turbo`](https://huggingface.co/openai/whisper-large-v3-turbo).

Covered end-to-end in Module 103 as the open-weights STT reference. Named here because: (a) Whisper's audio encoder is the input lane of Qwen2-Audio and several other audio-LLM designs; (b) a self-hosted Whisper + open-weights LLM + self-hosted TTS is a complete open-weights pipeline, which is often the right answer when "fully open-weights" is the compliance requirement.

### 6. Meta MMS and the long-tail-language story

- **Paper:** [Pratap et al., 2023](https://arxiv.org/abs/2305.13516).
- **Model:** [`facebook/mms-1b-all`](https://huggingface.co/facebook/mms-1b-all).
- **What it is:** Massively Multilingual Speech — ASR, TTS, and language-identification models covering ~1,000+ languages.
- **When to pick:** when your deployment has an endangered-language, low-resource-language, or specific-dialect requirement that managed vendors do not cover. MMS is the canonical reach model.

## The decision rule: when is open-weights defensible?

The right question is not "is open-weights ever worth it" (it is) but "is open-weights worth it for **this** deployment." Four tests.

### Test 1 — Compliance forces you off managed

Hard forcings:

- Air-gapped deployment (defence, intelligence, nuclear, classified pharma research).
- Data cannot leave jurisdiction and no managed vendor has a compliant presence (specific EU regulated sectors; certain Chinese data regimes; some Indian data regimes).
- Specific regulatory language that prohibits "transfer of personal data to third parties" with no vendor exception (some healthcare contexts in regulated geographies).
- Procurement rules that require full model inspectability.

If any of these apply, open-weights is forced. The engineering question becomes "which open-weights stack," not "open-weights or managed."

### Test 2 — Cost at scale crosses the break-even

Rough shape for the managed vs. self-host economics:

- A managed speech-native stack bills around tens of cents per minute of audio in and out combined (verify current rates).
- A self-hosted Moshi or Qwen2-Audio + TTS pipeline requires GPU capacity for the concurrent sessions (plausibly one A100 / H100 per few concurrent sessions, depending on model and optimisation).

The crossover is roughly in the **tens of thousands of hours per month** range. Below that, managed is cheaper on total cost of ownership (hardware, DevOps, on-call, model updates). Above that, self-host starts to win — assuming you have the ops team. Do the math on your own volume before committing.

### Test 3 — Capability vendors do not ship

Capabilities only available open-weights:

- **Full-duplex co-articulated speech** (Moshi). Managed vendors do not ship this today.
- **Cross-modal speech/text unified** (Spirit LM). Managed stacks offer either/or, not symmetric.
- **Speech understanding of non-speech audio** (Qwen2-Audio). Managed stacks assume audio input is speech.
- **100+ language speech-to-speech translation** (SeamlessM4T). Managed translation is text-mediated or language-limited.
- **Reach into endangered languages** (MMS).

If the deployment depends on one of these, open-weights is the only path.

### Test 4 — You can afford the ops surface

Open-weights means you own:

- GPU capacity planning and autoscaling.
- Inference optimisation (quantisation, batching, continuous batching, KV-cache management).
- Model updates (watching for new model releases, rebuilding training / deployment pipelines).
- On-call for model inference incidents (CUDA OOM at 3 am).
- Compliance posture for the inference layer (HIPAA for your own containers if that is the posture).
- Eval harness, benchmark cadence, and drift detection.

If your team does not have a dedicated ML-infra function, this is substantial. Do not pick open-weights because the model is free; the operational cost is not.

### The decision rule in one sentence

**Pick open-weights speech foundation models when compliance forces it, when you have the volume to amortise the ops surface, or when the specific capability is only available open-weights — and never because open-weights sounds appealing in isolation.**

## Composing an open-weights pipeline

A production-grade open-weights voice agent is usually one of three compositions.

### Composition A — full pipeline, all open-weights

- STT: Whisper-large-v3-turbo, or NVIDIA Parakeet, or MMS for long-tail languages.
- LLM: a strong open-weights text LLM (Llama 3.x, Qwen2.5, Mistral) with your own prompt caching and tool use.
- TTS: XTTS v2, F5-TTS, Parler-TTS, or Bark.
- Transport + relay: LiveKit (self-hosted) or Pipecat.

Entirely inside your infrastructure. Compliance-clean; engineering-heavy.

### Composition B — self-hosted speech-native

- Moshi for the conversational core (self-hosted).
- Optional orchestration layer on top for tool routing (since Moshi is weak at tools).

Lower-latency than Composition A. Appropriate when the product is conversational-first and tool use is light.

### Composition C — hybrid open + managed

- Self-hosted Whisper or Qwen2-Audio for ingress (compliance over the input audio).
- Managed LLM (where the text can legally leave your infra) — Claude via Bedrock, GPT via Azure, etc.
- Managed or self-hosted TTS.

Common when input-side data residency is the binding constraint and the output side is less sensitive.

## Licence posture

Each model's licence matters for commercial deployment. Read them, verify at author time. Rough shape as of this writing:

- **Moshi:** MIT-licensed codebase, weights with permissive terms. Commercial use allowed.
- **Spirit LM:** research-only licence — not permitted for commercial use. <!-- needs-research: confirm Spirit LM's current licence; Meta research licences evolve -->
- **Qwen2-Audio:** Apache 2.0-like permissive licence (verify current terms).
- **SeamlessM4T:** CC-BY-NC for the research release of certain variants; commercial-permissible for others. Read the specific model card.
- **Whisper:** MIT-licensed; fully commercial.
- **MMS:** CC-BY-NC for certain components; CC-BY-SA for others. Component-specific.

A permissive text LLM licence does not transfer. Each component of your open-weights pipeline has its own licence and its own posture; the pipeline inherits the strictest.

## Common anti-patterns

- **Open-weights as a vibes decision.** The model being open does not make the deployment economic. Run the four tests.
- **Underestimating the ops surface.** The hardest part of running Moshi in production is not Moshi; it is the inference autoscaling, the eval harness, and the 3-am pages.
- **Mixing licences without reading them.** Research-only components in a commercial pipeline is an infringement waiting to happen.
- **Treating a research-grade model as production.** Spirit LM is beautiful; it is not a product. Qwen2-Audio is close. Moshi is production-grade for a specific shape of deployment.
- **Missing the quality gap.** Open-weights text LLMs are close to frontier managed models in 2025; open-weights speech-LMs are a year or two behind the frontier. Measure before committing.
- **No eval harness.** Managed vendors evaluate continuously on your behalf; self-hosting means you evaluate. Module 112 is the full eval treatment; name it here as a dependency of any open-weights story.

## Summary

- Open-weights speech foundation models cluster into four kinds: **audio-language-models** (Moshi, Spirit LM, Qwen2-Audio, Whisper-derived audio LLMs); **neural audio codecs** (SoundStream, EnCodec, Mimi, DAC) that produce the token vocabulary; **speech translation models** (SeamlessM4T); **long-tail-reach ASR / TTS** (Meta MMS).
- Heritage threads: AudioLM / SoundStream (discrete-token audio generation), wav2vec 2.0 / HuBERT (self-supervised speech encoders), SeamlessM4T (open-weights speech translation).
- The right-to-pick tests: compliance forces it; cost at scale forces it; a specific capability only open-weights ships; you can afford the ops surface. Pass at least one — pass the fourth — and open-weights is the right call.
- Three canonical compositions: full open-weights pipeline, self-hosted speech-native (Moshi-centric), hybrid (open ingress, managed middle, mixed output).
- Licences are per-component. The pipeline inherits the strictest.
- Open-weights voice-agent quality in 2025 lags managed frontier by one to two years; measure before committing.

## End of module

This module has taught you three stacks (Realtime, Gemini Live, Claude-via-relay), one architecture decision (pipeline vs. speech-native vs. hybrid), one duplex spectrum (half vs. full), one safety discussion (vendor system cards vs. EU AI Act), and one open-weights survey.

The downstream modules take this foundation and stack specific capabilities on top: Module 106 (turn-taking and barge-in), Module 107 (telephony and WebRTC transport), Module 108 (safety, consent, and regulatory compliance in depth), Module 109 (latency budget and the end-to-end timing diagram), Module 110 (multimodal input design including the video lane Gemini Live opens up), Module 111 (voice-agent orchestration — stitching tool use, routing, and persona), Module 112 (evaluation and observability).

When a design decision from one of those modules depends on the architectural choice made here — half-duplex vs. full-duplex changing the turn-taking model, speech-native vs. pipeline changing the latency budget shape, vendor System Card vs. open-weights changing the safety composition — it is this module's chapters you cite.
