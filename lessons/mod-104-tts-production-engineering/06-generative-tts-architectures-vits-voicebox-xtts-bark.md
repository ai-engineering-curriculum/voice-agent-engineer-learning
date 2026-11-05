# 06. Modern Generative TTS Architectures at a Working Level

## Why this chapter

You are not a TTS researcher. You will not be re-implementing a mel-spectrogram vocoder. You are, however, picking between a vendor whose voice is a VITS-descendant and a vendor whose voice is a VoiceBox-descendant, and two vendors will be opaque about which architecture backs their model. The reason to be literate at architecture-level is the same reason it mattered on the ASR side: **the architecture explains the trade-offs you observe at integration time** — zero-shot cloning quality, expressiveness controllability, latency, length-handling, long-form stability — and the category tells you what to test for.

This chapter gives you the four families you should be able to speak about by name — VITS, VoiceBox, XTTS, Bark — plus the enabling pieces (vocoders, codec LMs, flow-matching, diffusion) and a decision frame for picking a managed or open-weights family for your deployment. Enough to pick well; not enough to retrain.

## The three-stage mental model

Most modern neural TTS systems fit a three-stage pipeline, even when the stages are fused into one trained model:

1. **Text processing.** Grapheme-to-phoneme (G2P), prosody prediction, speaker / style conditioning.
2. **Acoustic model.** Produces an intermediate acoustic representation — historically a mel-spectrogram; more recently a neural audio codec's discrete tokens.
3. **Vocoder.** Turns the intermediate representation into waveform (PCM audio samples).

The categorical splits that matter:

- **Two-stage vs. end-to-end.** Classical two-stage (Tacotron 2 + HiFi-GAN) vs. end-to-end (VITS, VITS2, StyleTTS 2) that learn text-to-waveform jointly.
- **Mel-spectrogram vs. neural-codec target.** Mel + vocoder (VITS, XTTS, most production stacks) vs. discrete-codec tokens + codec decoder (VoiceBox, Bark, and the current audio-LM generation). Codec targets enable audio-LM-style generation and discrete-token tricks (editing, prompt-conditioning).
- **Autoregressive vs. non-autoregressive generation.** AR models (Tacotron 2, Bark) produce the output step-by-step; NAR / flow / diffusion models (FastSpeech 2, VITS, VoiceBox, F5-TTS) produce it in parallel. NAR is faster and more stable on long inputs; AR models can be more expressive but hallucinate (repeat, skip, run away).

Keep this frame in mind as the four families below land in it.

## VITS (and VITS2, StyleTTS 2) — the end-to-end mainline

**Paper:** [Kim, Kong, and Son, 2021, "Conditional Variational Autoencoder with Adversarial Learning for End-to-End Text-to-Speech"](https://arxiv.org/abs/2106.06103).

VITS trains one model from text to waveform. The architecture combines:

- A **conditional variational autoencoder** on the acoustic side (learns a latent prosody space).
- A **normalising-flow prior** over the latent space (so the text-conditioned distribution of prosody is expressive enough to produce human-like variation).
- A **monotonic alignment search** between text and audio frames (replaces explicit duration predictors).
- An **adversarial / GAN** training signal from a HiFi-GAN-style waveform discriminator.

End-to-end means no separate vocoder; the model emits waveform directly from text. In practice the inference is fast and the output quality is strong. VITS2 ([Kong et al., 2023](https://arxiv.org/abs/2307.16430)) tightens the architecture. StyleTTS 2 ([Li et al., 2023](https://arxiv.org/abs/2306.07691)) layers a style-diffusion pipeline on top and is the open-weights quality reference for single-speaker TTS in 2024-2026.

What this matters for:

- **Picking an open-weights single-speaker TTS.** VITS / VITS2 / StyleTTS 2 is the mainline family — the one with the most forks, the best community tooling, and the most finetune recipes.
- **Expected limitations.** VITS-family models do **not** do zero-shot cloning well on their own; cloning typically requires fine-tuning on target-speaker data. For zero-shot cloning on open weights, skip to XTTS / F5-TTS.
- **Latency.** Low. VITS variants are fast enough to run realtime on a single GPU and in some cases on CPU.
- **Expressiveness.** Prosody comes from the latent prior; expressive control is less direct than codec-LM-based systems. StyleTTS 2 improves this via the diffusion-based style sampler.

## XTTS (Coqui) — zero-shot multilingual voice cloning

**Model / paper:** [`coqui/XTTS-v2`](https://huggingface.co/coqui/XTTS-v2); [Casanova et al., 2024, "XTTS: a Massively Multilingual Zero-Shot Text-to-Speech Model"](https://arxiv.org/abs/2406.04904).

XTTS is the open-weights zero-shot voice-cloning reference that most practitioners have shipped at least once. Architecture: a GPT-style autoregressive language model operating over discrete audio tokens (produced by a VQ-VAE codec), plus a conditioning path that ingests a short speaker reference clip and extracts a speaker embedding. Output: codec tokens decoded to waveform by a HiFi-GAN-style vocoder.

What this matters for:

- **Zero-shot cloning on open weights.** A 6–30 second reference clip, 16+ languages, no fine-tune required. Quality is below ElevenLabs IVC but in the "good enough for prototypes" range.
- **Multilingual with voice preservation.** The same reference voice reads multiple languages — one of the three or four systems in 2024-2026 that does this credibly on open weights.
- **AR-induced instability on long inputs.** Being autoregressive, XTTS can repeat, skip, or diverge on long sentences. Production use requires text chunking.
- **Latency.** Moderate; not a realtime voice-agent choice in default configuration, though community optimisations exist.
- **Licensing.** Coqui the company has wound down; the weights are still available under the Coqui Public Model Licence, which has commercial-use restrictions worth reading before shipping.

## VoiceBox — non-autoregressive, flow-matching, in-context

**Paper:** [Le et al., 2023 (Meta), "Voicebox: Text-Guided Multilingual Universal Speech Generation at Scale"](https://arxiv.org/abs/2306.15687).

VoiceBox demonstrated that non-autoregressive, flow-matching-based generation can produce state-of-the-art TTS without the stability costs of AR models. Architecture: a transformer trained with **conditional flow matching** on a continuous audio representation; conditioning is "in-context" — you provide a short context audio + text, and VoiceBox fills in the rest.

What this matters for:

- **The current open-paper SOTA direction for TTS.** Not released as open weights by Meta; the methodology has been reproduced in open-weights work ([Matcha-TTS](https://arxiv.org/abs/2309.03199), [E2 TTS](https://arxiv.org/abs/2406.18009), [F5-TTS](https://arxiv.org/abs/2410.06885)) which you can self-host.
- **Zero-shot cloning without autoregression.** Flow matching lets VoiceBox-style models produce stable, expressive, natural-sounding output at non-trivial length without the AR hallucination modes.
- **Faster and more stable than AR.** The parallel generation is both lower-latency and less prone to run-away.
- **Reads to engineers as "diffusion-like."** Flow matching is not diffusion but is closely related; mental model "learns a velocity field in a continuous space that transports noise into audio" is enough.

If a managed vendor's new voice line markets "flow-matching" or "non-autoregressive," this is the architectural family they are claiming.

## Bark — audio-language-modelling as TTS

**Code / model:** [`suno/bark`](https://huggingface.co/suno/bark) by Suno AI.

Bark is the clearest instance of **audio-language modelling** as TTS. Rather than mel-spectrograms, Bark operates on discrete audio codec tokens (specifically, EnCodec's codebook). The architecture is three transformer stages that successively produce semantic tokens, coarse acoustic tokens, and fine acoustic tokens, which then decode to waveform.

What this matters for:

- **Expressive, non-speech sounds.** Bark emits laughter, sighs, music, sound effects, and non-verbal cues because the audio-LM paradigm learns everything in the audio domain, not just phonemes. For a conversational agent whose brief includes natural non-verbal expressiveness, this is a strong fit.
- **Prompt-conditional style.** Natural-language prompts can steer the delivery.
- **Slow, not realtime.** Bark is not a voice-agent production choice. It is a creative-tool choice.
- **The template for everything that followed.** The audio-LM pattern — EnCodec tokens + transformer LM over them + decoder back to waveform — underlies many of the current generation of models, including on the audio-native-LLM side (Module 105).

## The vocoder, the codec, and the "what are we generating" axis

Two pieces of context that explain a lot of vendor marketing:

### Mel-spectrograms + neural vocoders

The classical path. The acoustic model emits a mel-spectrogram; a vocoder — **HiFi-GAN** ([Kong et al., 2020](https://arxiv.org/abs/2010.05646)), **BigVGAN** ([Lee et al., 2022](https://arxiv.org/abs/2206.04658)), **Vocos**, **WaveGrad** — converts the mel to waveform. Fast, well-understood, loses some high-frequency detail. Most managed vendors from 2020-2023 are in this regime.

### Neural audio codecs + token generation

The modern path. **EnCodec** ([Défossez et al., 2022](https://arxiv.org/abs/2210.13438)), **SoundStream** ([Zeghidour et al., 2021](https://arxiv.org/abs/2107.03312)), **Mimi** ([Kyutai Moshi paper, 2024](https://arxiv.org/abs/2410.00037)), **DAC** ([Kumar et al., 2023](https://arxiv.org/abs/2306.06546)) turn waveform into **discrete tokens**; an LM-style generator produces token sequences; the codec decoder reconstructs audio. This is the unification point with LLMs — audio becomes a token stream the same way text does, which is why the audio-native LLMs in Module 105 (Moshi, `gpt-4o-realtime`, Gemini Live) work at all.

### Why you care

- Vendor marketing that mentions "codec", "discrete tokens", or "audio language model" is in the second regime. Expect audio-LM-like tricks (prompt conditioning, expressive non-speech, editing).
- Vendor marketing that mentions "HiFi-GAN", "mel", or "neural vocoder" is the first regime. Expect classical TTS trade-offs — fast, stable, less expressive in non-speech ways.
- The codec itself is sometimes the bottleneck for audio quality. EnCodec at low bitrates reconstructs well on speech but can leave artefacts; DAC handles higher-fidelity music too.

## A worked decision frame

Three tracks a voice-agent engineer picks between on an open-weights / self-host question.

### Track 1 — Managed, no open-weights involvement

Most product teams. Pick a vendor per Chapter 1; don't care about the architecture beyond what the vendor tells you. Verify the four axes that matter per Chapter 1 and ship.

**The architecture knowledge from this chapter is mostly diagnostic** — it explains why a vendor's long-form stability is good or bad, why their expressiveness is natural or mechanical, why their latency is what it is. You use it to debug and to negotiate with vendors, not to choose weights.

### Track 2 — Open-weights for compliance / cost / control

Self-hosted. Pick by use case:

- **Single-speaker, high-quality, realtime-ish, one voice you fine-tune from a studio session.** StyleTTS 2 or VITS2. Mature forks, good community tooling.
- **Multilingual zero-shot cloning for prototypes, non-realtime acceptable.** XTTS v2 or F5-TTS. F5-TTS if you want the flow-matching family; XTTS if you want the broader ecosystem.
- **Expressive, non-speech sounds, non-realtime.** Bark or Parler-TTS.
- **Low-latency, low-resource, on-device.** Smaller VITS variants, Piper (`rhasspy/piper`), or the smaller Coqui models.

Expect a six-figure annual eng cost (one engineer's attention, ~20%) to run self-hosted TTS well. If that is less than your managed spend at scale, self-hosting pays; if not, managed.

### Track 3 — Audio-native / speech-to-speech alternative (Module 105)

If you are considering skipping the explicit TTS step entirely, the models in scope (Moshi, `gpt-4o-realtime`, Gemini Live) sit atop the audio-codec + audio-LM paradigm above. Learning the paradigm here prepares you for Module 105's discussion of when that path is the better architectural call.

## The right level of depth for the L35 bar

Six papers you should know by name, enough to read an abstract and a figure. This is the floor to be literate at. Do not read these to re-implement; read them to recognise.

- **Tacotron 2** — [Shen et al., 2018](https://arxiv.org/abs/1712.05884). The paper that mel-spectrogram-into-vocoder TTS became production on.
- **FastSpeech 2** — [Ren et al., 2021](https://arxiv.org/abs/2006.04558). Non-autoregressive TTS with explicit duration prediction; the template behind many production vendor stacks.
- **VITS / VITS2 / StyleTTS 2** — [Kim 2021](https://arxiv.org/abs/2106.06103), [Kong 2023](https://arxiv.org/abs/2307.16430), [Li 2023](https://arxiv.org/abs/2306.07691). The end-to-end mainline.
- **VoiceBox** — [Le et al., 2023](https://arxiv.org/abs/2306.15687). Flow-matching in-context TTS.
- **XTTS** — [Casanova et al., 2024](https://arxiv.org/abs/2406.04904). Zero-shot multilingual cloning on open weights.
- **Bark + EnCodec** — [EnCodec paper (Défossez 2022)](https://arxiv.org/abs/2210.13438) + the Bark [repo](https://github.com/suno-ai/bark). The audio-LM paradigm as TTS.

And one on the codec side you will encounter: **SoundStream** ([Zeghidour 2021](https://arxiv.org/abs/2107.03312)) and **DAC** ([Kumar 2023](https://arxiv.org/abs/2306.06546)) as the current references on neural audio codecs.

## Common anti-patterns

- **Picking open-weights for a production voice agent because "self-hosted is cheaper."** It isn't, at reasonable scale, once you account for the engineering time. Rerun the math honestly.
- **Choosing an AR model (XTTS, Bark) for a realtime streaming voice agent.** The inference profile does not fit. Flow-matching (F5-TTS) or VITS-family is a better match.
- **Retraining a vendor model you do not have weights for.** You cannot. You can at most fine-tune a vendor's PVC pipeline on their terms. Full-model retraining is the `training-pipeline-engineer` conversation ([Chapter 7](07-boundary-to-training-pipeline-and-agentic-safety.md)).
- **Treating "mel-spectrogram" and "codec tokens" as the same intermediate.** They are not. The downstream capabilities and artefacts differ substantially.
- **Building on an archived open-weights model.** Coqui XTTS is shipping-safe but the upstream is dormant; you own all bug fixes. Factor this into the choice.

## Summary

- Four families to be literate at: VITS (end-to-end mainline), XTTS (zero-shot multilingual cloning), VoiceBox (flow-matching non-AR), Bark (audio-LM with non-speech expressiveness).
- The three-stage pipeline (text → acoustic → waveform) is still the mental model; modern systems fuse or replace stages but you can still map them to it.
- The mel-vocoder vs. codec-token split explains most vendor trade-offs on expressiveness, latency, and audio-LM-adjacent features.
- Three tracks: managed (most teams); open-weights when compliance / cost / control justifies the eng cost; audio-native / speech-to-speech as the "skip TTS" alternative (Module 105).
- Six papers + two codecs are the reading list to stay literate. Read abstracts and figures; do not re-implement.

Next chapter: the boundary with `training-pipeline-engineer` (voice-clone fine-tune infrastructure) and `agentic-safety-engineer` (frontier voice-safety methodology).
