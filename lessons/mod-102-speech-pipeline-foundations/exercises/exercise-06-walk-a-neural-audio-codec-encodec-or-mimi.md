# exercise-06: Walk a Neural Audio Codec — EnCodec or Mimi

**Estimated effort:** 2 hours · **Output:** encoded token stream + reconstruction + a one-page comparison write-up

## Objective

Make the neural-codec concepts from [Chapter 6](../06-neural-audio-codecs.md) tangible by running a pre-trained codec end-to-end: encode speech to discrete tokens, inspect the token stream, decode back to audio, and measure the latency + bitrate + reconstruction quality.

You will finish able to answer "would this codec work in a realtime speech-LM?" from first principles — token rate, codebook count, decoder latency, open-weight availability — rather than from marketing.

## Prerequisites

- [Chapter 6 — Neural Audio Codecs](../06-neural-audio-codecs.md).
- Python 3.10+ with `torch`, `torchaudio`, `numpy`, `librosa`, `soundfile`.
- One of:
  - **EnCodec** via `transformers` (`EncodecModel`) or the `encodec` pip package. Open-weights, MIT-licensed. The 24 kHz speech checkpoint is the easier starting point.
  - **Mimi** via the [`moshi`](https://github.com/kyutai-labs/moshi) package (Kyutai). Open-weights, Apache-2.0. Weights are on Hugging Face.
- A short speech clip (~5–10 seconds, single speaker).

## Steps

### Part A — Encode (30 minutes)

1. Load your speech clip at the codec's native sample rate (24 kHz for both EnCodec-speech and Mimi — resample if needed).
2. Run the encoder. Capture the output tokens and record:
   - **Shape of the token tensor.** Expect roughly `(K, T)` where K is the number of codebooks and T is the number of frames.
   - **Number of unique token values** per codebook (each codebook has a fixed size; 1024 is typical).
   - **Token rate in frames per second** — compute it as `T / duration_seconds`. Compare against the codec's advertised rate.
   - **Equivalent bitrate** = frame_rate × K × log₂(codebook_size).
3. Plot a 2-D heatmap of the token IDs across time and codebook index. The top codebook (the semantic one for Mimi, the coarsest for EnCodec) should look much more "slow-varying" than the bottom codebooks.

### Part B — Decode (30 minutes)

4. Run the decoder on the full token stream. Save the reconstructed audio.
5. Compute these quality measures against the original clip:
   - **Mel-cepstral distortion (MCD)** or at minimum a log-mel MSE — cheaper and tells you most of what you need.
   - **Perceptual listening test** — listen through headphones to the original and the reconstructed, and write 1–2 sentences about what you lost (sibilance? room sound? pitch?).
6. **Simulate quantisation stress.** Zero out the bottom half of the codebooks (set those token streams to the codebook's silence / padding ID, or drop them if the codec API allows) and re-decode. Compare quality. This tells you how much each additional codebook contributes.

### Part C — Latency (30 minutes)

7. Measure per-frame encoder and decoder latency on your hardware:
   - Encoder: time to encode one frame of audio given existing state (streaming-style). If the API is non-streaming, time the full-clip encode and divide.
   - Decoder: time to decode one frame of tokens into audio.
8. Compare the decoder latency to the codec's frame period (80 ms for Mimi; ~13 ms for EnCodec at 75 Hz). **Decoder latency ≤ frame period is the condition for realtime playback.** Record whether your hardware meets it.

### Part D — The write-up (30 minutes)

9. Produce a one-page comparison with these sections:

   - **Name of the codec and version** (checkpoint, release date).
   - **Three numbers:** frame rate, codebook count × size, equivalent bitrate.
   - **Realtime verdict:** "On this hardware (${device}), decoder runs at ${x} ms per 80 ms frame, which is / is not realtime-feasible for a speech-to-speech LM."
   - **Reconstruction quality notes:** what's preserved, what's lost; one sentence on pitch, sibilance, room sound.
   - **Open-weights status and license.**
   - **If used as a speech-LM substrate, what would the LM's token-generation rate have to be?** (frame_rate × K tokens/s.)

## Starter guidance

- With `transformers`, EnCodec is loaded as `EncodecModel.from_pretrained("facebook/encodec_24khz")`. The call returns `audio_codes` (the token tensor) and `audio_scales`.
- For Mimi, see the `moshi` package README. Loading is `from moshi.models.loaders import get_mimi; mimi = get_mimi(...)`. Streaming encode and decode are supported.
- Both models prefer float32 input in `[-1, 1]`, shape `(batch, channels, samples)`. Convert PCM16 before encoding.
- Running on CPU is OK for this exercise; GPU is nice for the latency measurements (and more honest for a realtime claim).
- For the "zero out codebooks" stress test, be mindful: dropping the top (semantic) codebook destroys the audio completely; dropping the bottom codebooks mostly affects fine detail. This asymmetry *is* the result you want to see.

## Acceptance criteria

- [ ] Encode + decode runs end-to-end, saves the reconstructed audio file.
- [ ] Token-tensor shape, frame rate, codebook count, and bitrate reported and sanity-checked against the codec's advertised specs.
- [ ] Heatmap plot of tokens across time and codebook.
- [ ] Quality measurement (MCD or log-mel MSE) and a listening-test note.
- [ ] Codebook-drop stress test showing graceful (semantic preserved) vs. catastrophic (semantic dropped) degradation.
- [ ] Encoder + decoder per-frame latency measured on your hardware, with the realtime verdict.
- [ ] One-page comparison write-up, grounded in your measurements.

## Stretch goals

- **Run both EnCodec and Mimi on the same clip and compare.** Same audio in, two reconstructions out. Which is more faithful? Which uses fewer tokens? Which is realtime-feasible on your hardware?
- **Streaming encode / decode with Mimi.** Use the streaming API; measure incremental latency per frame as the clip is fed in. Produce a time series: input frame index → encoder output latency.
- **Round-trip robustness.** Encode, drop N % of frames (simulate packet loss), decode. At what loss rate does the audio become unintelligible? Compare to Opus PLC behaviour (from exercise-04) at similar loss rates.
- **Measure context efficiency.** If a 2k-token LM context holds 160 s of Mimi tokens (8 codebooks × 12.5 Hz × 160 s = 16 000 tokens, which actually *exceeds* 2k — confirm the arithmetic in the Moshi paper and reconcile with how Moshi handles context). What does a 7-codebook-per-frame speech stream cost per minute of dialogue in LM tokens?

## Keep it

Your encode/decode script and the single-page comparison are referenceable going forward. In Module 105 (realtime speech-to-speech) you will revisit these codecs with a fuller architectural lens; the measurements you took here are the floor for those decisions.
