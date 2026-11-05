# 04. Audio Codecs: Opus, G.711, PCM16

## Why this chapter

Three codecs will handle ~95 % of the production audio you'll ship through: **Opus** for WebRTC and modern realtime transport, **G.711 µ-law / A-law** for the PSTN edge, and **PCM16** for internal uncompressed paths and recording.

Picking the wrong codec for a leg isn't a quality choice — it's a *the-call-doesn't-work* choice. SIP carriers don't negotiate Opus; browsers don't natively decode µ-law. This chapter gives you the one-page codec decision matrix and the handful of knobs (bitrate, FEC, DTX, PLC) that actually matter at L35.

## PCM16 — the baseline you compare against

PCM16 (16-bit signed linear PCM, little-endian, interleaved) is **not a codec** — it is just samples. No compression, no framing, no decoder state. At 16 kHz mono it's **256 kbps**; at 48 kHz stereo it's **1.536 Mbps**. You use it:

- **Inside the pipeline**, as the canonical format every component speaks (Chapter 3).
- **For recording / archiving** when you must not throw away any quality.
- **To the ASR provider over a WebSocket**, if the provider doesn't require Opus.
- **As the output of the TTS provider** to feed a mixer or a re-encoder.

PCM16 has no quality or framing parameters to set. You decide the sample rate and channel count, and that's it. The only mistake you can make is endian or channel-order confusion.

## Opus — the default for anything real-time over IP

Opus is defined by [RFC 6716](https://datatracker.ietf.org/doc/html/rfc6716) (2012, Xiph.Org / Mozilla / Skype engineers). It is the mandatory-to-implement codec for WebRTC and the overwhelming default for realtime voice on the open internet.

Properties you'll rely on:

- **Royalty-free and open.** You can ship it anywhere.
- **Low latency.** Frame sizes 2.5 / 5 / 10 / 20 / 40 / 60 / 80 / 100 / 120 ms (the standard choice is 20 ms); the encoder's algorithmic delay is small — RFC 6716 §2.1.4 specifies a 5 ms look-ahead, i.e. 26.5 ms total for a 20 ms frame.
- **Variable bitrate from 6 kbps to 510 kbps.**
- **Covers both speech and music.** Has an internal SILK mode (speech-optimised, based on Skype's SILK codec) and CELT mode (music-oriented), and can auto-switch or operate as a hybrid.
- **Operates at internal sample rates 8 / 12 / 16 / 24 / 48 kHz** ("narrowband / medium / wideband / super-wideband / fullband"). WebRTC always runs Opus at 48 kHz on the wire, even when the actual content is speech sampled at 16 kHz — the endpoint resamples.

### The three knobs that matter

#### Bitrate

In WebRTC, the Opus bitrate is negotiated during SDP and adapted at runtime by congestion control. You can constrain it with `maxaveragebitrate` in the SDP fmtp line. Typical production ranges:

| Content | Bitrate | Comment |
|---|---|---|
| Speech-only, narrowband OK | 12 – 24 kbps | Phone-quality speech, lower than G.711 |
| Speech, wideband | 24 – 40 kbps | Clearly better than PSTN; the default sweet spot |
| Speech + music / stereo call | 64 – 128 kbps | Needed only if the audio is actually music |
| Very constrained link | 6 – 12 kbps | Still intelligible speech; quality degrades fast |

For a voice agent: start at **~32 kbps mono**, revisit only if you see call-quality complaints or bandwidth pressure. The RFC lists the full table in §2.1.1; marketing comparisons usually cite 24 – 32 kbps as "transparent" for speech.

#### FEC (Forward Error Correction) — `useinbandfec=1`

Opus supports in-band FEC: a lower-bitrate copy of the previous frame is embedded in the current frame, so when a packet is lost the decoder can recover from the *next* packet without silence. Enable it when:

- The link is lossy (mobile network, PSTN edge, long-haul).
- You've budgeted the extra bitrate (~10–25 % overhead).

Enable with `useinbandfec=1` in the SDP fmtp, plus telling the encoder a target packet-loss-percentage. WebRTC turns this on automatically when it sees loss; your application-layer Opus (over a WebSocket, for example) won't unless you ask.

#### DTX (Discontinuous Transmission) — `usedtx=1`

DTX stops transmitting during silence and sends tiny "comfort noise" descriptors instead. Saves bandwidth; the far-end plays synthesised quiet noise so the line doesn't sound dead. Enable when:

- Bandwidth matters.
- You are **not** also running a VAD-driven turn-taker that needs byte-level timing of silence.

The gotcha: if your downstream logic treats "no packets arriving" as "far-end is silent," DTX is redundant. If it treats it as "connection lost," DTX actively breaks you. Decide once and document it.

### Opus in code

With `opuslib` (Python bindings for `libopus`):

```python
import opuslib

FRAME_MS = 20
SR = 48_000
CH = 1
FRAME_SAMPLES = FRAME_MS * SR // 1000  # 960

enc = opuslib.Encoder(SR, CH, opuslib.APPLICATION_VOIP)
enc.bitrate = 32_000
enc.vbr = True
# enc.inband_fec = True     # if you've enabled FEC in SDP

# pcm_frame must be exactly FRAME_SAMPLES int16 samples
opus_bytes = enc.encode(pcm_frame.tobytes(), FRAME_SAMPLES)

dec = opuslib.Decoder(SR, CH)
pcm = dec.decode(opus_bytes, FRAME_SAMPLES)
```

The one thing to watch: **you must call the encoder with exactly one frame's worth of samples at a time.** Variable frames (Anti-pattern 1 in Chapter 3) cause either `OPUS_BAD_ARG` or quality degradation.

## G.711 — the PSTN edge

G.711 is the ITU-T telephony codec from 1972 (!), defined in [ITU-T Recommendation G.711](https://www.itu.int/rec/T-REC-G.711). It is **8-bit logarithmic encoding of an 8 kHz signal**, giving a fixed **64 kbps** bitstream. Two variants:

- **µ-law (PCMU, RTP payload type 0)** — used in North America and Japan.
- **A-law (PCMA, RTP payload type 8)** — used in Europe and most of the rest of the world.

Properties:

- **Universal on PSTN and SIP.** Every telephony stack speaks it. If your Twilio or Vonage SIP trunk is the transport, you will exchange µ-law or A-law packets.
- **Fixed rate, no knobs.** No bitrate choice, no FEC, no DTX (in the base codec).
- **Narrowband only — 300 Hz to 3.4 kHz effective.** Sibilants and brightness are gone. Expect WER to be 1.5–3× worse than wideband 16 kHz for the same utterance, on most ASR providers. <!-- needs-research: cite a provider's own narrowband-vs-wideband WER comparison if available -->
- **No decoder state.** Each 8-bit sample is independent. Great for tolerating packet loss (you lose 20 ms, not a buffer's worth); terrible for compression.

When you'll touch it:

- **Any PSTN call leg.** Twilio Media Streams deliver µ-law 8 kHz packets by default.
- **Legacy IVR integrations.** G.711 is the baseline that every PBX can negotiate.
- **Transcoding between a SIP leg and an internal 16 kHz pipeline.** You'll decode µ-law to PCM16, upsample to 16 kHz, run ASR; then downsample TTS output from 24 kHz to 8 kHz and re-encode as µ-law for the far-end.

### G.711 in code

Python's standard library has it:

```python
import audioop

# PSTN inbound: 8-bit µ-law @ 8 kHz  →  int16 PCM @ 8 kHz  →  int16 PCM @ 16 kHz
pcm8 = audioop.ulaw2lin(mulaw_bytes, 2)                 # 2 = 16-bit output
pcm16_8k = np.frombuffer(pcm8, dtype=np.int16)
pcm16_16k = resampler_8_to_16(pcm16_8k)                 # use torchaudio or soxr

# Egress to PSTN: TTS @ 24 kHz → 8 kHz → µ-law
pcm16_8k = resampler_24_to_8(tts_pcm16_24k)
mulaw_bytes = audioop.lin2ulaw(pcm16_8k.tobytes(), 2)
```

The three things that go wrong:

- **Confusing µ-law and A-law.** The SDP tells you which one; if you get it wrong, the far-end hears loud white noise.
- **Forgetting to resample.** If you put 16 kHz PCM16 through `lin2ulaw`, you get *samples* in µ-law, but at 16 kHz where the carrier expects 8 kHz — played back at half speed.
- **Clipping during downsample.** The 24 kHz → 8 kHz step needs a low-pass filter *before* decimation, or you'll get aliasing. Any decent resampler library handles this; writing your own with `::2` indexing does not.

## G.722 — honourable mention

G.722 ([ITU-T G.722](https://www.itu.int/rec/T-REC-G.722)) is a 50 Hz – 7 kHz **wideband** codec at 64 kbps that some SIP endpoints and corporate PBXes negotiate when both sides support it. You will occasionally see it on VoIP legs where the SIP ALG chose it in preference to G.711. It behaves roughly like a slightly-lower-quality 16 kHz PCM for your pipeline purposes.

## Picking the codec at each seam

The one-page decision matrix:

| Leg / seam | Codec | Why |
|---|---|---|
| Browser microphone capture (WebRTC) | Opus @ 48 kHz | Mandatory-to-implement in WebRTC; negotiated automatically |
| Browser speaker playback (WebRTC) | Opus @ 48 kHz | Same |
| Native mobile app ↔ media server | Opus or raw PCM16 over WebSocket | Opus if bandwidth-constrained; PCM16 if you control both ends |
| PSTN inbound (Twilio, carriers) | G.711 µ-law @ 8 kHz | PSTN standard |
| PSTN outbound | G.711 µ-law @ 8 kHz | PSTN standard |
| Internal ASR WebSocket | PCM16 @ 16 kHz, or Opus if the provider mandates | Provider contract |
| Internal TTS → mixer | PCM16 @ TTS native rate (often 22.05 / 24 kHz) | Preserve pre-encoder quality |
| Media server ↔ worker (SFU recording path) | Opus or PCM16 | Depends on media server capability |
| Recording / archive | PCM16 or FLAC (lossless) | Don't throw away quality you won't get back |

The asymmetry to internalise: **the PSTN leg and the WebRTC leg use different codecs, so a voice agent that spans both always has a transcoder in the middle.** That transcoder is a latency contributor, a point of failure, and often where quality is lost.

## FEC vs. PLC vs. DTX — don't confuse them

- **FEC** (forward error correction): add redundancy so a lost packet can be reconstructed. Costs bandwidth.
- **PLC** (packet-loss concealment): no redundancy; when a packet is lost, the decoder *synthesises* a plausible waveform to fill the gap. Costs nothing on the wire; quality is limited. All real codecs have some form of PLC.
- **DTX** (discontinuous transmission): don't send packets during silence. Saves bandwidth; needs comfort noise at the far-end to avoid dead-line perception.

Default stance for a production voice agent: **PLC always on (implicit), FEC on under loss, DTX off unless bandwidth is the binding constraint and nothing downstream needs byte-accurate silence timing.**

## Summary

- Three codecs cover 95 % of what you'll ship: PCM16 inside the pipeline, Opus over WebRTC, G.711 over PSTN.
- Opus knobs: bitrate (~32 kbps mono is a reasonable default), FEC (on under loss), DTX (off unless bandwidth matters).
- G.711 has no knobs — only the µ-law / A-law variant to pick, and the discipline to always resample 8 ↔ 16 kHz correctly at the seam.
- The PSTN leg and the WebRTC leg use different codecs, so span-the-two voice agents always include a transcoder. Put it in your diagram.

Next chapter: noise suppression and echo cancellation — WebRTC AEC3, RNNoise, Krisp — and where to place them in the pipeline.
