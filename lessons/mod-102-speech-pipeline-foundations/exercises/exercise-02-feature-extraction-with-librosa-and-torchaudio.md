# exercise-02: Feature Extraction with librosa and torchaudio

**Estimated effort:** 2 hours · **Output:** a notebook with five plotted features over five clips + a short diagnostic write-up

## Objective

Build the five-feature debugging toolkit from [Chapter 2](../02-audio-features-with-librosa-and-torchaudio.md) as muscle memory. By the end you should be able to:

- Load an audio file and extract waveform, log-mel, MFCCs, pitch (f0), and RMS in under 10 lines.
- Read each plot and state what the audio "is" from the plot alone.
- Match plots to the three debugging recipes in Chapter 2 (empty-ASR, dead-TTS, broken-barge-in).

## Prerequisites

- [Chapter 2 — Audio Features with librosa and torchaudio](../02-audio-features-with-librosa-and-torchaudio.md).
- Python 3.10+ with `librosa`, `torchaudio`, `numpy`, `matplotlib`, `soundfile`.
- At least five short audio clips (see below) — you can reuse the reference corpus from exercise-01 and add more.

## Steps

### Part A — Set up the clip set (20 minutes)

1. Assemble five clips, each 2–5 seconds long:

   - **Clip A — Clean speech.** Your own voice reading a short sentence at a normal level, 16 kHz PCM16 mono.
   - **Clip B — Telephony-band speech.** The same sentence passed through an 8 kHz → 16 kHz round-trip (downsample, then upsample; or a real PSTN recording if you have one).
   - **Clip C — Noisy speech.** The same sentence with a continuous noise source added (HVAC loop, pink noise, or a known noise sample from a public dataset).
   - **Clip D — TTS output.** A synthesised version of the same sentence from your favourite TTS provider (or a free one — Coqui XTTS, Piper, etc.).
   - **Clip E — Clipped / overdriven speech.** The same sentence recorded or scaled such that at least 10 % of samples hit ±1.0.

### Part B — Build the feature plotter (45 minutes)

2. Write a single function `plot_features(path)` that produces a 5-row figure for a given file:

   - Row 1 — Waveform (time on x, amplitude on y). Annotate peak and RMS in the title.
   - Row 2 — Log-mel spectrogram (80 Mel bins, `n_fft=512`, `hop_length=160` for 16 kHz input).
   - Row 3 — MFCCs (13 coefficients, same hop).
   - Row 4 — Pitch (f0) from `librosa.pyin`, with voiced probability ≥ 0.5 overlaid or masked.
   - Row 5 — RMS in dBFS (smoothed with a 20 ms window), with the threshold line at −45 dBFS drawn.

3. **Build a `torchaudio` equivalent** of rows 2 and 3 using `torchaudio.transforms.MelSpectrogram` and `torchaudio.transforms.MFCC`. Confirm the outputs agree with `librosa` to within rounding (expect small differences — document which parameters differ between the two libraries' defaults and which you had to equalise).

### Part C — Read the plots (45 minutes)

4. Run `plot_features` on each of the five clips. For **each clip**, write a short paragraph (3–5 sentences) that answers:

   - What is immediately visible in the plots (hard cutoff, flat f0, clipped waveform, consistent RMS, etc.)?
   - What does that imply the audio is / where in the pipeline it came from / what would happen if it went into an ASR?
   - Which of the three Chapter 2 debugging recipes would you apply if you'd received this clip in a ticket?

5. **Pick one clip and deliberately damage it** (e.g., add aggressive noise suppression that smears harmonics; apply a 3 kHz low-pass; pitch-shift by an octave) and re-run the plots. Write one paragraph on what changed and which feature changed most dramatically.

### Part D — The one-liner cheat sheet (10 minutes)

6. Produce a short reference cheat sheet (part of the notebook or a separate `.md`) with one librosa line per feature and the matching torchaudio call. Keep it under half a page. This is the file you'll paste from for the next six months.

## Starter guidance

- `librosa.load(path, sr=None)` — don't forget `sr=None` or you'll resample everything to 22.05 kHz silently.
- `librosa.pyin` is slow. For a 5-second clip it may take 1–2 seconds; don't panic.
- When building torchaudio equivalents, note: librosa's `n_mels` and `fmax` defaults differ from torchaudio's; set both explicitly on both sides so the comparison is meaningful.
- To match f0 plot time axis with the rest, use `hop_length=160` and convert frame indices to seconds yourself.
- RMS in dBFS: `20 * np.log10(np.maximum(rms, 1e-10))`. The `np.maximum` guard is non-negotiable; log of zero is not fun.

## Acceptance criteria

- [ ] Five clips assembled, each saved with a clear label.
- [ ] `plot_features` function working and producing a 5-row figure for each clip.
- [ ] `torchaudio` log-mel and MFCC implementations agreeing with `librosa` (within documented differences).
- [ ] One short diagnostic paragraph per clip (five total) grounded in the plots, not general knowledge.
- [ ] One "deliberate damage" paragraph showing you can predict which feature will move.
- [ ] A one-page cheat sheet of librosa ↔ torchaudio calls.

## Stretch goals

- Add pitch estimation with **CREPE** ([`torchcrepe`](https://github.com/maxrmorrison/torchcrepe) or the original CREPE repo) alongside PYIN. On the same clip, do the two estimators agree? Where do they diverge — creaky voice, high noise, octave jumps?
- Compute the UTMOS or DNSMOS score for clips A, C, D. Does the model's single-number quality estimate agree with what you see in the plots? Where does it disagree?
- Package `plot_features` as a tiny CLI (`python feature_report.py clip.wav → clip.png`). This becomes the attachment you'll paste into every voice-audio bug report for the rest of your career.

## Keep it

The `plot_features` function and the cheat sheet. You'll reuse both in modules 103 (STT debugging), 104 (TTS quality), 106 (VAD tuning), and 112 (observability).
