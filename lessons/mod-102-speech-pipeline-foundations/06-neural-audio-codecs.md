# 06. Neural Audio Codecs — EnCodec, SoundStream, Mimi

## Why this chapter

Opus, G.711, and PCM16 cover transport on the open internet (Chapter 4). A different family of codecs has emerged for a different job: **turning audio into a sequence of discrete tokens that an LLM can consume.**

The modern speech-to-speech models (OpenAI Realtime, Google Gemini Live, Moshi, Zephyr-style speech LMs) are not operating on raw waveforms. They operate on **discrete audio tokens** produced by a learned codec — the audio-equivalent of BPE subword tokens for text. A neural audio codec is what maps 48 kHz waveforms down to a stream of ~12.5–75 discrete tokens per second that an LLM backbone can process the same way it processes text.

This chapter is the voice-agent-engineer's working view of three codecs — Meta's EnCodec, Google's SoundStream, Kyutai's Mimi — and the three facts that shape every integration decision:

1. **What a neural codec is.** Encoder → vector-quantised discrete tokens → decoder, trained end-to-end on real audio.
2. **Why token rate matters.** For realtime speech-to-speech, the codec's **tokens-per-second** sets the LLM's bandwidth requirement and the achievable latency floor.
3. **Where open-weights codecs land vs. commercial / closed ones.** Which choices are actually available to you.

You do not need to *train* a neural audio codec at L35 (that's `nlp-engineer` and `ml-engineer` territory; see [Chapter 7](07-boundary-nlp-and-ml-engineer.md)). You do need to be able to pick the right pre-trained one, integrate it, and reason about its latency / quality trade-off.

## The pattern: tokeniser for audio

All three codecs share a shape:

```
raw waveform (48 kHz float)
    │
    ▼
┌─────────────────┐
│   CNN encoder   │   downsamples to a low-rate latent stream
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ residual vector │   each latent frame → multiple discrete token IDs
│  quantisation   │   (RVQ: K stacked codebooks of size 1024)
│    (RVQ)        │
└────────┬────────┘
         │  discrete tokens @ ~12.5 – 75 Hz × K codebooks
         ▼
┌─────────────────┐
│   CNN decoder   │   generates waveform from tokens
└────────┬────────┘
         │
         ▼
reconstructed waveform
```

Key terminology:

- **Frame rate** — how many *time steps* per second the encoder emits. For a 48 kHz input and a 1920× downsampling factor, that's 25 Hz. For Mimi with a 3840× factor (at 24 kHz input), that's 12.5 Hz. Lower frame rate → fewer tokens per second → easier for an LLM to generate.
- **Codebook depth (K)** — how many stacked vector quantisation codebooks. 4, 8, or 32 are common. Deeper stacks give better reconstruction but multiply the token count per time step.
- **Tokens per second** = frame rate × K. This is the actual bandwidth the LLM has to generate.
- **Bitrate** = tokens-per-second × log₂(codebook size). Mimi at 12.5 Hz × 8 codebooks × 10 bits = 1 000 bits/s ≈ 1 kbps. EnCodec at 75 Hz × 8 × 10 = 6 kbps. The codec papers report this as "kbps-equivalent."

## The three codecs

### EnCodec (Meta, 2022)

[EnCodec](https://arxiv.org/abs/2210.13438) ("High Fidelity Neural Audio Compression") was one of the first widely-used neural audio codecs. Meta released weights and a reference implementation.

Key facts:

- Convolutional encoder/decoder with residual vector quantisation.
- Supports 24 kHz (speech-focused) and 48 kHz (general-audio) checkpoints.
- Operates at **75 Hz** frame rate with up to **32 codebooks** of 1024 entries each; typical inference uses 4–8 codebooks.
- Target bitrates in the paper range from 1.5 kbps to 24 kbps.
- Available in `transformers` (`EncodecModel`) and the `encodec` Python package.

Where EnCodec is used:

- **MusicGen** (Meta) uses the 32 kHz music variant as the token backbone.
- **Audiobox / Voicebox** (Meta) use EnCodec-like tokenisation.
- Many open-source speech-LM research releases build on EnCodec.

### SoundStream (Google, 2021)

[SoundStream](https://arxiv.org/abs/2107.03312) is Google's neural audio codec; the architectural ancestor of the above. The specific weights are not open-sourced in a widely-used form — SoundStream is best thought of as an *architecture*, with the practically available descendants being products like the Lyra codec and speech tokens inside AudioLM / SPEAR-TTS / VALL-E-style research.

For a voice-agent engineer: you'll see SoundStream referenced in papers as the token backbone, but you'll rarely instantiate it directly. The open-weight codec you'll actually run will be EnCodec or Mimi.

### Mimi (Kyutai, 2024)

[Mimi](https://arxiv.org/abs/2410.00037) is the neural audio codec inside Kyutai's Moshi speech-to-speech model, released open-weights with the Moshi model itself.

Key facts (from the Moshi paper):

- **12.5 Hz frame rate** at 24 kHz input — one acoustic token every 80 ms per codebook.
- **8 codebooks** → 100 tokens/s, i.e. **1.1 kbps** equivalent.
- The first codebook is a **semantic** token (distilled from WavLM features) with an explicit information-theoretic objective; the remaining 7 are **acoustic** residual codebooks.
- Decoder is causal and streamable, so one can generate speech incrementally.

The 12.5 Hz rate is the headline number: at 100 tokens/second, speech is cheap enough to be generated interleaved with text tokens by a single LLM backbone — which is exactly what Moshi does. Compare to EnCodec at 75 Hz × 8 = 600 tokens/s; at 7B-parameter LM scale the latter is impractical for realtime.

### A quick comparison

| Codec | Sample rate | Frame rate | Typical K | Tokens/s | Equivalent bitrate |
|---|---|---|---|---|---|
| EnCodec (speech) | 24 kHz | 75 Hz | 4–8 | 300–600 | 3–6 kbps |
| EnCodec (general) | 48 kHz | 75 Hz | 4–32 | 300–2400 | 3–24 kbps |
| Mimi | 24 kHz | 12.5 Hz | 8 | 100 | 1.1 kbps |

<!-- needs-research: cross-check the Mimi comparison table against the Moshi tech report Table 1 at the time of authorship -->

## Why 12.5 Hz matters for realtime

Three things get easier when a codec runs at 12.5 Hz rather than 75 Hz:

**1. The LLM can keep up at a conversational rate.**
Moshi's text backbone generates audio tokens at the codec's frame rate. At 12.5 Hz × 8 codebooks, that's 100 tokens/s — well within the token-emission rate of a modern 7B model on a single H100-class GPU. At 75 Hz × 8 = 600 tokens/s, the same model is latency-bound.

**2. The attention context stretches further.**
A 2 048-token context window holds **163 seconds** of audio at 12.5 Hz, versus only **27 seconds** at 75 Hz. Long-horizon dialogue memory becomes tractable.

**3. End-to-end latency drops.**
Because speech can be generated one acoustic *frame* at a time with Mimi's causal decoder (80 ms per frame), the minimum time between the model "deciding" to start speaking and the first audio sample reaching the user is bounded by one frame — ~80 ms plus decoder forward pass.

The engineering trade-off: lower frame rates require *larger codebook depth* (more codebooks per frame) to maintain quality. Mimi's 8 codebooks are already a lot for an LM to generate in parallel; some research codecs push to 32, which is prohibitive for most LMs.

## When you'll touch a neural audio codec at L35

Three concrete places:

### 1. Integrating a speech-to-speech model

If you build on OpenAI Realtime or Gemini Live, the codec is hidden inside the provider — you send and receive PCM16 or Opus, and the provider's own audio tokeniser lives on their side. You never see the tokens.

If you build on an open-weight model (Moshi, future entrants), the codec is **part of your deployment surface**. Your pipeline looks like:

```
mic → PCM 24 kHz → Mimi encoder → tokens → LM → tokens → Mimi decoder → PCM 24 kHz → speaker
```

You deploy the codec weights, manage their memory footprint (~40 MB for Mimi), expose a streaming API that takes 80 ms frames in and emits 80 ms frames out, and make sure the encoder and decoder share the same checkpoint. Versions must match across encoder/LM/decoder; a Mimi-2024-10 LM paired with a Mimi-2025-04 decoder produces noise.

### 2. Research-adjacent evaluation

An eval harness that scores TTS quality on a reconstruction task (encode → decode → compare) treats the neural codec as a *reference*. You pick the codec deliberately — EnCodec for compatibility with the research literature, Mimi if you're evaluating against speech-LM baselines.

### 3. Building a cascaded pipeline that *borrows* the codec's semantic tokens

The first codebook in Mimi is a semantic token distilled from WavLM. Some non-realtime architectures use that as a lightweight speech representation for retrieval or classification, side-stepping the full LM. This is a research pattern more than a production one at L35, but worth knowing exists.

## Open-weights vs. commercial

The map, roughly:

| Codec | Open weights? | Primary users | Usable today? |
|---|---|---|---|
| EnCodec | **Yes** (Meta, MIT license) | MusicGen, open research | Yes — `transformers` or `encodec` package |
| SoundStream | Architecture only | Google research, Lyra (closed derivative) | Not directly — study the paper, use Lyra for Google services |
| Mimi | **Yes** (Kyutai, Apache 2.0) | Moshi, Kyutai-stack products | Yes — `moshi` package, PyTorch weights |
| OpenAI Realtime internal tokeniser | No | OpenAI GPT-4o-Realtime | No — access via API only |
| Gemini Live internal tokeniser | No | Google Gemini Live | No — access via API only |

The practical implication: **if you want to run a speech-LM on your own infrastructure today, your codec options are EnCodec or Mimi.** Everything commercial lives behind a provider API that abstracts the codec entirely.

## The latency vs. quality trade-off

You will keep running into this three-way knob:

- **Lower frame rate** → easier for LM, longer context, lower latency floor; but harder to reconstruct fine temporal detail, so more codebooks needed.
- **More codebooks** → better reconstruction; but higher token rate for the LM to generate.
- **Larger codebook size** → better per-code quality; but more bits per token, slower sampling.

Mimi's pick (12.5 Hz × 8 × 1024) is a specific choice for realtime conversational speech. For a non-realtime use case — a music generator or a dubbing pipeline — different choices (75 Hz, more codebooks) are preferable.

You will not retune these as a voice-agent engineer. You *will* read a codec's spec sheet and know whether its choices are compatible with a realtime loop or not.

## Summary

- Neural audio codecs are tokeniser-detokeniser pairs for audio: encoder → RVQ tokens → decoder. They are the substrate for speech-LM models.
- EnCodec (Meta, open-weights) and Mimi (Kyutai, open-weights) are the two codecs you can actually run today. SoundStream is referenced but not directly available.
- **12.5 Hz tokenisation** is what makes realtime speech-to-speech LMs tractable: 100 tokens/s fits on a single-GPU LM budget, 2k context = 160 s of audio, per-frame latency ~80 ms.
- Commercial speech-to-speech providers (OpenAI Realtime, Gemini Live) hide their codec; open-weights models (Moshi) expose it as part of your deployment.

Next chapter: the boundary with `nlp-engineer` and `ml-engineer` — DSP depth and self-supervised speech representations (wav2vec 2.0, HuBERT, MMS) — and where to escalate.
