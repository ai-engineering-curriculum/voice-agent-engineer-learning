# exercise-01: Deploy Silero VAD and Tune Thresholds Against a Noisy Corpus

**Estimated effort:** 2 hours · **Output:** a Silero VAD wrapper, a labelled mini-corpus, a threshold-sweep table, and a short write-up justifying the chosen operating point.

## Objective

Ship a working Silero VAD at the pipeline edge and tune the three knobs that matter — threshold, silence duration (hangover), and minimum speech + prefix padding — against a mixed-condition corpus. By the end, you should be able to name your operating point in one sentence ("threshold 0.55, silence-duration 500 ms, min-speech 250 ms, prefix 300 ms on headset audio; threshold 0.65, silence-duration 650 ms on noisy telephony") and point at the numbers that justify it.

The point is not to pick the globally best VAD — it is to walk the tuning loop once end-to-end so the knob surface is familiar and the failure modes are legible in a packet trace.

## Prerequisites

- [Chapter 1 — VAD at the pipeline edge](../01-vad-at-the-pipeline-edge.md).
- Python 3.10+, `torch`, `torchaudio`, optionally `onnxruntime`.
- A terminal and a way to play audio clips back.
- Module 102 Chapter 1 (sample rates, PCM) familiarity.
- A few minutes of the LibriSpeech `dev-clean` split ([openslr.org/12](https://www.openslr.org/12/)) OR equivalent clean speech you can legally use. You will mix this against noise.

Optional: a noise corpus. [MUSAN](https://www.openslr.org/17/) (Snyder, Chen, Povey 2015) ships music + speech + noise and is the default background-mixing corpus. [DEMAND](https://zenodo.org/record/1227121) is an alternative with realistic environmental recordings.

## Requirements

### Part A — stand up a streaming Silero VAD wrapper

1. Pull the latest Silero VAD model from [github.com/snakers4/silero-vad](https://github.com/snakers4/silero-vad) via `torch.hub` or the ONNX distribution.
2. Build a Python class `StreamingVAD` that accepts PCM16 chunks (16 kHz mono) and emits an event per state change: `speech_started(ts_ms)`, `speech_ended(ts_ms, duration_ms, mean_prob)`. Internally it runs the model's iterator on 30 ms frames and manages threshold, silence duration, minimum speech, and prefix padding.

   ```python
   vad = StreamingVAD(
       threshold=0.5,
       silence_duration_ms=500,
       min_speech_duration_ms=250,
       speech_pad_ms=300,
   )
   async for pcm_chunk in mic_stream:
       for event in vad.step(pcm_chunk):
           handle(event)
   ```

3. Write a small CLI harness: `python vad_harness.py path/to/clip.wav` prints the stream of events with wall-clock-relative timestamps.

### Part B — build a mini mixed-condition corpus

Construct a corpus of **at least 30 clips, totalling ≥ 10 minutes**, stratified roughly evenly across:

- **Clean** — LibriSpeech-like studio speech, SNR > 20 dB.
- **Moderate noise** — speech mixed with 10–15 dB SNR ambient noise (cafe, office, keyboard).
- **Heavy noise** — speech mixed with 0–5 dB SNR ambient (street, car, strong background chatter).
- **Hard cases, at least five** — one of each:
  1. A short utterance (< 500 ms, e.g., "yes", "no").
  2. A mid-utterance pause ≥ 700 ms ("so I was thinking — three things").
  3. A non-speech vocalisation (laugh / cough).
  4. A quiet-starting utterance (breathy "yes" starting below the main speech energy).
  5. A speakerphone leak — echo of a bot voice leaking back into the mic (simulate by playing a TTS clip at low level in the same file).

For each clip, annotate a lightweight JSONL with (at least) the ground-truth `speech_start_ms`, `speech_end_ms`, and `class` (clean / moderate / heavy / hard). Two labellers if you have them; otherwise one labeller, and acknowledge the limitation in the write-up.

### Part C — threshold sweep

Sweep the three knobs independently (one axis at a time, holding the others at the defaults from Chapter 1):

- **Threshold:** 0.3, 0.4, 0.5, 0.6, 0.7.
- **Silence duration (ms):** 200, 350, 500, 700, 1000.
- **Minimum speech (ms):** 100, 250, 500. Hold prefix padding at 300.

For each sweep point, compute (per clip + aggregated per class):

- **False trigger rate** — speech-started events outside any labelled speech span, per minute of audio.
- **Missed trigger rate** — labelled speech spans with no speech-started event within 300 ms of the labelled onset.
- **Onset clip** — mean | event_ts − labelled_onset | for correctly-triggered spans.
- **Offset error** — mean (event_ts − labelled_offset) for correctly-triggered spans; positive is "declared end too late," negative is "clipped the tail."
- **Non-speech trigger rate** — speech-started events on the hard-case clips 3 (laugh / cough) specifically.

Emit a results table as Markdown or CSV. One row per (knob value × class), one column per metric.

### Part D — pick the operating point

Pick a shipping configuration **for each deployment class** (clean headset vs. noisy telephony vs. hard-cases-tolerant). Write one or two sentences per class justifying the pick from the sweep numbers.

Specifically:

- The clean-headset config should prioritise low missed-trigger rate and low onset-clip.
- The noisy-telephony config should prioritise low false-trigger rate (bandwidth and STT-bill cost).
- The hard-cases-tolerant config should push `silence_duration_ms` higher to protect against mid-utterance pauses; name the trade-off against bot-response gap.

### Part E — compare against WebRTC VAD (optional baseline)

Run the same corpus through [py-webrtcvad](https://github.com/wiseman/py-webrtcvad) at aggressiveness modes 1 and 2. Report the same five metrics. One paragraph: on which clip classes is WebRTC VAD close to Silero, and on which does it visibly lose?

This is scoped as optional because the main learning is Silero tuning; the WebRTC comparison is the "is-the-upgrade-worth-it" evidence you want for a design review.

## Starter guidance

- **Use 16 kHz mono PCM16 throughout.** Silero supports 8 kHz but 16 kHz is the default and simpler.
- **Chunk size.** Silero's design window is 30 ms (512 samples at 16 kHz). Feed the model in exact multiples. Your streaming wrapper can accept arbitrary chunks and internally re-chunk.
- **Prefix padding is backfill.** Keep a rolling buffer of the last 500 ms of audio. When VAD declares `speech_started`, "backfill" by emitting the event with a `ts_ms` corresponding to `now - speech_pad_ms` — not `now`. This is what prevents the first phoneme being clipped downstream.
- **Noise mixing.** [`torchaudio`'s `add_noise` or a manual `signal * 1.0 + noise * gain` works fine. Measure SNR with `signal_rms / noise_rms`.] Keep the mixing code in your repo so the corpus is reproducible.
- **Measurement window.** Match events to labels within a 300 ms tolerance. Document the tolerance.
- **Logging.** Dump the per-frame probability stream when debugging a specific clip; it makes failure modes legible.
- **Iterate on one knob at a time.** Threshold first; then silence duration; then min speech + padding. Compound sweeps are not the goal of this exercise.

## Acceptance criteria

- [ ] `StreamingVAD` class accepts streaming PCM16 and emits `speech_started` / `speech_ended` with wall-clock-relative timestamps.
- [ ] Corpus of ≥ 30 clips, ≥ 10 min total, stratified across clean / moderate / heavy / hard, with a JSONL label file.
- [ ] Threshold-sweep table covering at least 5 × 5 threshold × silence-duration combinations, with the five metrics above, per class and overall.
- [ ] One-page write-up:
  - Chosen operating point per deployment class, one sentence each with a pointer to the table row that justifies it.
  - A short discussion of which hard case each config handles acceptably and which it does not.
  - One paragraph on the false-trigger failure mode on the hard-case clip 3 (laugh / cough). Does any threshold suppress it, or does it require a downstream non-speech classifier?
- [ ] Reproducible: a `README.md` in your exercise directory with "how to run" steps.

## Stretch goals

- **Rolling-noise-floor hangover.** Replace the fixed `silence_duration_ms` with a hangover that scales with the recent-noise-floor estimate. Compare false-trigger rate on heavy-noise clips.
- **Non-speech classifier.** Train a small MFCC-based logistic regression on labelled laugh / cough / breath segments. Gate `speech_started` on `p(non-speech) < 0.5`. Report the delta on hard-case clip 3.
- **Compare with vendor server VAD.** Feed the same clips through OpenAI Realtime (`turn_detection.type = "server_vad"` with the parameters from the config object). Log the server's `input_audio_buffer.speech_started` / `.speech_stopped` events; compare against your Silero numbers on the matched clips. Note: this costs vendor minutes.
- **Multilingual sanity check.** Run a handful of non-English clips (any FLEURS language, say Spanish or Japanese) through the same operating points. Record whether the thresholds hold or need language-specific tuning.

## Non-goals

- Reaching production-perfection. The tuning exercise is a walk, not a destination.
- Building the EOU layer. That is Exercise 2. Silence-based endpointing is enough for this exercise.
- Shipping a full pipeline. A WAV in and an event stream out is enough.

## Keep it

The `StreamingVAD` class, the mini corpus, and the sweep harness are reusable. Exercise 5 (the metric suite) will reuse the corpus as a seed; the barge-in exercise (Exercise 3) will plug `StreamingVAD` into the mic-side of the pipeline. Commit the harness; do not throw it away when the write-up is done.
