# 01. Sample Rates, Bit Depth, Channels, and PCM vs. Compressed

## Why this chapter

Every voice bug you'll ship this quarter will route back through four numbers: **sample rate, bit depth, channel count, and container/codec**. Get them wrong and your ASR returns empty strings, your TTS sounds like a chipmunk, your WebRTC renderer plays silence, or your telephony call-leg returns pink noise.

The skill is boring but load-bearing. The components in a voice pipeline — microphone capture, resampler, VAD, STT provider, LLM, TTS provider, mixer, codec, transport, far-end playback — each have an implicit contract about what audio they accept and emit. Most "the agent went silent after the handoff" tickets are a mismatch at one of those contract boundaries.

This chapter gives you the four numbers, the common sample-rate tiers, and the smallest set of rules you need to not break the pipeline at a seam.

## The four numbers

A raw digital audio stream is fully described by:

1. **Sample rate** — how many amplitude samples per second, measured in Hz. 16 000 Hz = 16 kHz.
2. **Bit depth** — how many bits encode each sample. 8, 16, 24, 32 are the usual choices. 16-bit signed little-endian ("PCM16LE") is the default across speech work; float32 appears inside ML frameworks.
3. **Channels** — 1 (mono) or 2 (stereo) for the audio you'll see in this track. Phone audio is always mono; browser microphone capture is mono in practice even when the API says stereo.
4. **Container / codec** — raw PCM (no framing), WAV (PCM + a 44-byte header), FLAC, Opus-in-Ogg, Opus-in-WebM, µ-law/A-law in a SIP RTP stream, etc. "Codec" is *how* the samples are compressed on the wire; "container" is *how they're framed* so a decoder can find them.

If any one of those four is wrong at a boundary, the stream downstream is wrong.

## Why these specific sample rates

Human speech energy is almost entirely below 8 kHz, with intelligibility carried in the 300 Hz – 3.4 kHz band (the classic "telephone bandwidth"). By the Nyquist–Shannon sampling theorem you need a sample rate at least twice the highest frequency you want to represent — so 8 kHz sampling covers up to 4 kHz, which is why telephony has been able to use it for a century.

Different stages of a modern voice stack sit at different sample-rate tiers because they were optimised at different times for different trade-offs:

| Tier | Where you see it | Why that rate |
|---|---|---|
| **8 kHz** | PSTN / landline / SIP trunk audio, G.711 µ-law or A-law | ITU-T telephony standards since the 1970s; preserves the intelligibility band only. |
| **16 kHz** | Streaming ASR default (Whisper, Deepgram, AssemblyAI, Google STT, Azure STT) | "Wideband" speech — the full energy range of voiced speech, no music. All production ASR models are trained here. |
| **24 kHz** | Many neural-TTS output streams (ElevenLabs, OpenAI TTS historically, several open-weights vocoders) | Enough headroom to render sibilants and consonants cleanly; cheaper than 48 kHz on the vocoder side. |
| **44.1 / 48 kHz** | WebRTC default, browser `getUserMedia`, music, consumer audio files | 48 kHz is the WebRTC RTP clock rate and the native Opus internal rate; 44.1 kHz is CD-audio legacy. |

Two consequences you will keep running into:

- **ASR wants 16 kHz, everything else gives you something different.** The browser hands you 48 kHz; the phone hands you 8 kHz; your TTS returns 22.05 or 24 kHz. You will resample.
- **TTS output is often 22.05 or 24 kHz, but your transport wants 48 kHz (Opus/WebRTC) or 8 kHz (telephony).** Again, you will resample — this time downstream of the model.

> *Sibilance* is the hissy, high-frequency energy in sounds like /s/, /ʃ/ ("sh"), /z/, and /tʃ/ ("ch"). It sits roughly in the 4–10 kHz band, which is why cleanly rendered sibilants are the first thing that dies when you downsample 24 kHz TTS to 8 kHz for a phone call — and why cheap TTS sounds "lispy" over a landline.

The one rate you should treat as fixed is 16 kHz for ASR input. If a provider lists additional supported rates, prefer the native rate they trained on; the "we'll resample internally" option exists for convenience, not for quality.

## Bit depth

16-bit signed PCM covers roughly 96 dB of dynamic range, which is more than speech ever needs. You will see:

- **PCM16** (16-bit signed little-endian, interleaved for multi-channel) — the default on-wire format for raw audio in every voice pipeline.
- **PCM24 / PCM32** — studio captures; occasional ingest format. ASR providers accept this and downconvert.
- **float32** in range `[-1.0, 1.0]` — the internal format of `librosa`, `torchaudio`, `numpy` DSP, ML models.
- **µ-law / A-law (G.711)** — 8-bit **logarithmic** encoding. Not linear PCM: a non-linear curve maps the 14- or 13-bit linear sample range into 8 bits. Over PSTN you'll receive 8 kHz / 8-bit µ-law ("PCMU", RTP payload type 0) in North America and Japan, A-law ("PCMA", payload type 8) in Europe.

Converting between PCM16 and float32 is trivial but error-prone — the two directions of the conversion have been the source of many "why is the audio clipping" incidents:

```python
import numpy as np

# int16 ∈ [-32768, 32767]  →  float32 ∈ roughly [-1.0, 1.0]
def pcm16_to_float32(x: np.ndarray) -> np.ndarray:
    assert x.dtype == np.int16
    return (x.astype(np.float32) / 32768.0)

# float32 ∈ [-1.0, 1.0]  →  int16  (clip anything outside the range)
def float32_to_pcm16(x: np.ndarray) -> np.ndarray:
    assert x.dtype == np.float32
    return np.clip(x * 32768.0, -32768, 32767).astype(np.int16)
```

The `32768.0` divisor (not `32767.0`) is the de-facto convention — it's asymmetric but matches what every audio library you'll meet does. µ-law ↔ PCM16 conversion is a non-trivial curve; use `audioop.ulaw2lin` from the Python standard library, or your provider SDK, rather than writing it yourself.

## Mono vs. stereo

Assume **mono** for every speech pipeline you'll build in this track, unless you're explicitly doing:

- **Stereo call capture for diarisation** — some contact-centre vendors record the agent on the left channel and the caller on the right so downstream diarisation can split them trivially. Your ASR provider either takes two mono streams (one per channel) or you demux the stereo file yourself.
- **Binaural or spatial output** for a product with a spatial audio story. Rare at L35 voice-agent scope.

The anti-pattern to recognise: **a stream labelled stereo but containing the same waveform on both channels**, usually because a provider or SDK duplicated a mono stream into an "interleaved stereo" format. Most ASR providers accept it; most VADs and feature extractors silently take only the left channel. If your voice activity detection suddenly behaves differently after a transport change, check channel count.

## PCM vs. compressed: the three questions

Raw PCM has no framing, no codec, no header — just interleaved samples. On the wire, you usually want framing (so decoders can find packet boundaries) and compression (so you fit in a bitrate budget). Three questions to classify any audio blob:

1. **Container or raw?** A `.wav` file is a 44-byte header + PCM body. An RTP payload is framed per-packet. A WebSocket chunk is often raw PCM with no framing at all — the frame boundary is the WebSocket message boundary.
2. **Compressed or not?** PCM16 is uncompressed (and unambiguously ~256 kbps at 16 kHz / 16-bit mono). Opus, G.711 µ-law, G.722, AAC, MP3, FLAC are compressed, each with its own trade-offs.
3. **Streaming or file?** Streaming codecs (Opus, G.711) carry one frame per packet and recover from loss. File codecs (MP3, AAC in MP4) carry a full bitstream with headers at the start. Don't try to stream a `.mp3` file into an ASR WebSocket; you'll ship headers the decoder can't resync on.

Codec choices are covered in depth in [Chapter 4](04-codecs-opus-g711-pcm16.md). For this chapter, all you need: know what you have and know what the next hop wants.

## Where mismatches break the pipeline

Here is the minimum mental model for debugging "the audio is broken somewhere." Each link in the pipeline has an input contract and an output contract.

```
mic / phone leg
   │  48 kHz float32 stereo  (browser)
   │  8 kHz µ-law mono       (PSTN)
   ▼
capture layer
   │  → resample to 16 kHz PCM16 mono
   ▼
VAD + ring buffer
   │  → 16 kHz PCM16 mono, 10 or 20 ms frames
   ▼
ASR provider
   │  → text + word timings
   ▼
LLM
   │  → text
   ▼
TTS provider
   │  → 24 kHz PCM16 mono (or Opus frames at 48 kHz)
   ▼
mixer / codec
   │  → resample + re-encode to transport rate
   ▼
transport
   │  Opus @ 48 kHz for WebRTC,  µ-law @ 8 kHz for SIP
   ▼
far-end playback
```

The failure modes you'll see at each seam:

| Seam | Mismatch | Symptom |
|---|---|---|
| capture → VAD | Forgot to resample from 48 kHz to 16 kHz | VAD triggers constantly ("everything is speech") or never triggers; ASR receives 3× sped-up audio and returns gibberish |
| capture → ASR | Sent float32 where provider wants PCM16 | Empty or garbage transcript; some providers reject with 4xx, some silently return "" |
| capture → ASR | Sent stereo where provider wants mono | ASR takes left channel only → silently halved audio if source was wrong |
| ASR → LLM | Partial-vs-final confusion | LLM responds to speculative partials; see Module 103 on endpointing |
| TTS → mixer | TTS at 24 kHz, mixer expects 48 kHz | TTS plays back pitch-shifted up an octave (treating 24 k as 48 k = 2× playback) |
| TTS → mixer | TTS at 24 kHz, mixer expects 8 kHz | TTS plays slowed and aliased |
| mixer → transport (SIP) | Forgot to downsample to 8 kHz / µ-law | SIP leg plays silence or a loud buzz — the carrier drops the packet |
| mixer → transport (WebRTC) | Encoding Opus at 8 kHz "narrowband" when 48 kHz was expected | Audio plays but sounds like a 1998 cellphone |

If you remember one pattern from this chapter, make it the pitch-shift one. **Playback speed is proportional to sample-rate mismatch.** Playing 24 kHz audio into a 48 kHz sink without resampling plays it at 2× speed (and roughly an octave higher). Playing 48 kHz audio into a 16 kHz sink plays it at 1/3 speed. This is the Rosetta Stone of audio debugging: when a voice sounds wrong, measure the pitch shift, infer the rate error, trace the resampler.

## A minimum reproducer of each mismatch

Keep this snippet in your toolbox. It generates a 440 Hz sine tone — a reference pitch — in a few common formats, so you can play them through your pipeline and hear the mismatch directly.

```python
import numpy as np, soundfile as sf

def write_tone(path, sr, bit_depth, seconds=1.0, freq=440.0, channels=1):
    t = np.arange(int(sr * seconds), dtype=np.float32) / sr
    wave = 0.3 * np.sin(2 * np.pi * freq * t).astype(np.float32)
    if channels == 2:
        wave = np.stack([wave, wave], axis=1)
    subtype = {16: "PCM_16", 24: "PCM_24", 32: "PCM_32"}[bit_depth]
    sf.write(path, wave, sr, subtype=subtype)

write_tone("tone_8k_16b.wav",  sr=8_000,  bit_depth=16)
write_tone("tone_16k_16b.wav", sr=16_000, bit_depth=16)
write_tone("tone_24k_16b.wav", sr=24_000, bit_depth=16)
write_tone("tone_48k_16b.wav", sr=48_000, bit_depth=16)
```

Play the 24 kHz file through a pipeline that assumes 48 kHz — the 440 Hz reference tone comes out at 880 Hz. That's the diagnostic.

## Summary

- Four numbers describe any speech-pipeline audio stream: sample rate, bit depth, channels, container/codec.
- Four tiers dominate: 8 kHz telephony, 16 kHz streaming ASR, 24 kHz TTS output, 48 kHz WebRTC.
- 16-bit signed PCM is the default wire format; float32 is the in-process ML format; µ-law is the PSTN wire format.
- Most mismatches manifest as pitch shift (wrong sample rate), silence (wrong channel), or clipping/noise (wrong bit depth or codec).
- When an audio stream sounds wrong, measure pitch against a reference tone and work backwards to the resampler.

Next chapter: how to extract and read the standard audio features (waveform, log-mel, MFCC, pitch, energy) so that "the audio is broken somewhere" becomes "the audio is broken *here*."
