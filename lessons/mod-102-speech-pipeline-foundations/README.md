# mod-102-speech-pipeline-foundations: Audio and Speech Foundations for Engineers

**Estimated effort:** 12 hours · **Role:** Voice & Multimodal Agent Engineer · **Level:** L35

## What this module is for

This is the floor. Every voice agent you build in the rest of the track rides on the four numbers (sample rate, bit depth, channels, codec), the five debug features (waveform, log-mel, MFCCs, pitch, RMS), a chunking layer, a codec decision at each seam, a cleanup layer, and — increasingly — a neural audio codec sitting between a speech-LM and the wire.

The goal of this module is not to make you a DSP researcher. It is to give you the **engineer's working depth**: enough to design and debug the pipeline, choose configuration knobs with reasons, and know exactly when to escalate to `nlp-engineer` or `ml-engineer`.

## Learning objectives

- Reason about sample rates (8 kHz telephony / 16 kHz streaming ASR / 24 kHz TTS output / 48 kHz WebRTC), bit depth, mono vs. stereo, PCM vs. compressed — and know which mismatch breaks which link in a voice pipeline.
- Extract and read standard audio features — waveform, log-mel spectrogram, MFCCs, pitch (f0), energy — enough to debug a bad ASR run or a dead TTS pipeline, using librosa and torchaudio.
- Design the audio buffering layer — ring buffers, chunk sizes (10 ms / 20 ms / 40 ms frames), sample-rate conversion, resampling trade-offs — for streaming pipelines; recognise the common anti-patterns (variable-frame concatenation, late-resample, cross-thread drift).
- Choose and configure the audio codec — Opus (RFC 6716) for WebRTC and modern telephony, G.711 (µ-law / A-law) for PSTN, PCM16 for loss-free capture, with the bitrate / FEC / DTX knobs that matter.
- Deploy real-time noise suppression and echo cancellation — WebRTC AEC3, RNNoise, Krisp SDK — with the acoustic-echo and background-voice failure modes each targets; decide placement (client vs. media server vs. provider edge).
- Understand neural audio codecs (EnCodec, SoundStream, Kyutai Mimi) as the substrate for speech-token LLMs — why 12.5 Hz tokenisation matters for realtime, what the latency vs. quality trade-off is, and where open-weights codecs land vs. commercial.
- Cite the boundary to `nlp-engineer` (level 30) and `ml-engineer` (level 20) on the deep DSP / self-supervised-speech-representation (wav2vec 2.0, HuBERT, MMS) research side — this module covers the engineer's working depth.

## Lecture chapters

1. [01 — Sample Rates, Bit Depth, Channels, and PCM vs. Compressed](01-sample-rates-bit-depth-and-pcm.md)
2. [02 — Audio Features with librosa and torchaudio](02-audio-features-with-librosa-and-torchaudio.md)
3. [03 — Buffering and Chunking for Streaming Pipelines](03-buffering-and-chunking-for-streaming.md)
4. [04 — Audio Codecs: Opus, G.711, PCM16](04-codecs-opus-g711-pcm16.md)
5. [05 — Noise Suppression and Echo Cancellation](05-noise-suppression-and-echo-cancellation.md)
6. [06 — Neural Audio Codecs: EnCodec, SoundStream, Mimi](06-neural-audio-codecs.md)
7. [07 — The Boundary with `nlp-engineer` and `ml-engineer`](07-boundary-nlp-and-ml-engineer.md)

## Exercises

- [exercise-01 — Sample Rates, Codecs, and Bit Depth Tour](exercises/exercise-01-sample-rates-codecs-and-bit-depth-tour.md) (2 h)
- [exercise-02 — Feature Extraction with librosa and torchaudio](exercises/exercise-02-feature-extraction-with-librosa-and-torchaudio.md) (2 h)
- [exercise-03 — Ring Buffer and Chunking Layer for Streaming](exercises/exercise-03-ring-buffer-and-chunking-layer-for-streaming.md) (2 h)
- [exercise-04 — Opus and G.711 Codec Decision Drill](exercises/exercise-04-opus-and-g711-codec-decision-drill.md) (1 h)
- [exercise-05 — Deploy RNNoise (and Evaluate Krisp) in a Pipeline](exercises/exercise-05-deploy-rnnoise-and-krisp-in-a-pipeline.md) (2 h)
- [exercise-06 — Walk a Neural Audio Codec: EnCodec or Mimi](exercises/exercise-06-walk-a-neural-audio-codec-encodec-or-mimi.md) (2 h)

## Structure

- `01-…md` … `07-…md` — lecture chapters.
- `exercises/` — per-exercise prompts.
- `labs/` — long-form hands-on labs (none in this module; subsequent modules do the longer-form work).
- `quizzes/` — knowledge checks.
- `resources.md` — primary-source references for everything cited in this module.

## How to use this module

Read the chapters top to bottom. Chapters 1–3 build the mental model of a stream of audio samples travelling through a pipeline; chapters 4–5 attach codec and cleanup decisions to the seams; chapter 6 looks ahead to speech-LM substrates; chapter 7 draws the boundary with other roles.

The exercises are **best done after the matching chapter rather than at the end**. Exercise 1 pairs with Chapter 1, exercise 2 with Chapter 2, etc. Exercises 1–3 build artefacts (reference corpus, feature plotter, ring buffer) you will reuse in modules 103 (STT), 104 (TTS), 106 (VAD/turn-taking), and 112 (observability).

Prerequisite: [mod-101-voice-agent-position](../mod-101-voice-agent-position/README.md).

Next module: [mod-103-stt-production-engineering](../mod-103-stt-production-engineering/README.md).
