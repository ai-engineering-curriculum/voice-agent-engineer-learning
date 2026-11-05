# exercise-05: Deploy RNNoise (and Evaluate Krisp) in a Pipeline

**Estimated effort:** 2 hours · **Output:** a before/after audio comparison, WER numbers, and a placement decision write-up

## Objective

Operationalise the cleanup-layer decisions from [Chapter 5](../05-noise-suppression-and-echo-cancellation.md) by actually deploying noise suppression on a noisy voice corpus, measuring the ASR impact, and deciding where in the pipeline it should live.

Krisp is a commercial SDK; this exercise assumes you don't have a licence. The spirit is: **run RNNoise end-to-end, read Krisp's docs to know what it would add, and write up the decision.** If you do have Krisp access, substitute it into Steps 3 and 4.

## Prerequisites

- [Chapter 5 — Noise Suppression and Echo Cancellation](../05-noise-suppression-and-echo-cancellation.md).
- Python 3.10+ with `numpy`, `librosa`, `soundfile`, and one of:
  - [`rnnoise-wrapper`](https://pypi.org/project/rnnoise-wrapper/) or similar Python binding;
  - a compiled `rnnoise_demo` binary from the [RNNoise source repo](https://github.com/xiph/rnnoise);
  - the RNNoise WASM build if you prefer running it client-side.
- An ASR provider with a free tier or a local Whisper model (`openai-whisper` or `faster-whisper`).
- Noisy speech corpus: ~20 clips, 3–10 seconds each, mixed from a clean-speech source (e.g., LibriSpeech dev-clean) and a noise source (e.g., [DNS Challenge noise set](https://github.com/microsoft/DNS-Challenge), or just real recordings at a café / near a HVAC vent).

## Steps

### Part A — Baseline (30 minutes)

1. Build the noisy corpus. For each of 20 clean clips, mix in a different noise segment at an SNR of about 5–10 dB. Keep the mixing script; you'll use it again.
2. Transcribe all 20 noisy clips with your chosen ASR. Record WER against the clean transcripts. Do the same for the 20 clean clips as a ceiling baseline.
3. Produce a small table: clean WER (ceiling), noisy WER (floor without cleanup).

### Part B — Run RNNoise (45 minutes)

4. Apply RNNoise to each noisy clip. RNNoise expects **48 kHz float32 mono** PCM frames of 480 samples (10 ms). Resample up to 48 kHz if your clips are at 16 kHz; resample back after.
5. Re-transcribe. Record the RNNoise WER. Compare against the clean and noisy baselines.
6. For three specific clips (pick: steadiest noise, most non-stationary noise, loudest overall), make a side-by-side plot: noisy log-mel spectrogram vs. RNNoise-cleaned log-mel spectrogram. Note where the cleanup helped and where it smeared voiced content (Chapter 2's "over-suppression" signature).

### Part C — Placement decision (30 minutes)

7. In your three Chapter-4 architectures (browser-to-agent, PSTN inbound, hybrid-with-handoff), decide **where** you would run RNNoise. For each architecture, state the placement (client, provider edge, media server, pre-ASR worker) and the trade-off you accepted (CPU cost, bandwidth saved, over-suppression risk).

8. Read [Krisp's overview](https://krisp.ai/) (or a current comparable commercial doc) and note down:

   - One thing Krisp claims to do that RNNoise cannot (specifically **background-voice cancellation**, not just noise).
   - Two scenarios in your user base where that difference would actually bite ASR, if any.
   - One scenario where paying for Krisp would be **gratuitous** (RNNoise does enough).

### Part D — Double-suppression trap (15 minutes)

9. Apply RNNoise **twice** to the same clip (output of first pass → input of second pass). Transcribe. Plot the spectrogram. Confirm Chapter 5's "underwater" signature. Write one sentence on what you would check next time someone reports "the audio sounds watery" in production.

## Starter guidance

- RNNoise is strictly 48 kHz. Do not try to feed 16 kHz input "to save a resample" — the model was trained at 48 kHz and will do the wrong thing.
- The RNNoise reference binary processes one `.raw` 16-bit PCM file. For Python integration, the `rnnoise-wrapper` package or a `ctypes`-bound compiled lib is easier than piping through `subprocess`.
- When mixing noise into clean speech, measure the SNR you're actually hitting rather than trusting a scaling factor; `20 * log10(rms_speech / rms_noise)` after mixing.
- Pick an ASR that can be re-run cheaply on 20 clips. `faster-whisper` with `small.en` is usually enough and runs on CPU.
- If Krisp access is available (free trial, or an employer-licensed SDK), swap it in for one of your architectures and repeat the WER measurement. Report your own numbers rather than Krisp's marketing ones.

## Acceptance criteria

- [ ] 20-clip noisy corpus assembled with documented SNRs.
- [ ] WER table: clean / noisy / RNNoise-cleaned / (optional) Krisp-cleaned.
- [ ] Spectrogram comparison for at least three clips with annotated improvements and side-effects.
- [ ] Explicit RNNoise placement decision for each of the three Chapter-4 architectures, with trade-offs stated.
- [ ] Krisp-vs-RNNoise note: one scenario each for "buy," "don't buy," and "needs testing."
- [ ] Double-suppression reproduced and documented.

## Stretch goals

- **AEC experiment.** Build a crude simulated acoustic echo: play a TTS clip through your speakers, record it on your microphone, and attempt to cancel it using an open-source AEC library (e.g. the Speex DSP library's AEC, or [WebRTC APM Python bindings](https://github.com/signalaudio/webrtc-audio-processing)). You'll need the reference signal (what you played) synchronised to the mic signal. Document the latency and what happens when they drift.
- **Target-speaker extraction research review.** Read one recent paper on target-speaker extraction (e.g. from Interspeech 2023/2024) and note how it is evaluated vs. RNNoise-style metrics (SDR, PESQ, WER). Where would this fit in your pipeline, compared to Krisp?
- **Latency and CPU budget.** Measure RNNoise's per-frame inference time on your target hardware (laptop, cloud VM CPU, mobile if you can). Is it under 1 ms per 10 ms frame? If not, what does that mean for a 200-concurrent-call server?

## Keep it

The before/after comparison and WER delta is the shape of the evidence you'll present when proposing a noise-suppression change in production. Save the experiment harness.
