# 02. Audio Features with librosa and torchaudio

## Why this chapter

You don't need to be a DSP researcher to ship voice agents. You do need to be able to **look at an audio file and say what's wrong with it** — because "the ASR returned garbage" and "the TTS sounds dead" are, nine times out of ten, diagnosable at the feature level in two minutes.

The five features in this chapter are the engineer's working toolkit:

- **Waveform** — the raw amplitude-over-time plot. Answers: is there any signal? Is it clipping? Is it mostly silence?
- **Spectrogram** (log-mel) — energy over frequency over time. Answers: is the audio band-limited (telephony, downsampled)? Is there a hum, a hiss, a vocoder artefact?
- **MFCCs** — compact spectral features, the historical input to many ASR systems. Answers: do two clips have the "same" speech content at a cheap-to-compare level?
- **Pitch (f0)** — fundamental frequency over voiced regions. Answers: is the speaker male/female/child typical? Is the TTS flat (monotone pitch), or does it move?
- **Energy / RMS** — signal loudness over time. Answers: is the audio too quiet for ASR? Is the level consistent, or does it duck at the boundaries where VAD should fire?

Two libraries dominate the engineer's workflow — `librosa` for interactive analysis and notebook-style debugging, `torchaudio` for pipeline-grade GPU-batchable feature extraction. You should be fluent in both.

> Scope note. This chapter stays at the **engineering working depth** — enough to debug. The deeper DSP math (Mel scale derivation, LPC vs. cepstral analysis, pitch-estimation algorithms like PYIN) is `nlp-engineer` and `ml-engineer` territory; see [Chapter 7](07-boundary-nlp-and-ml-engineer.md).

## librosa vs. torchaudio: when to use which

| | `librosa` | `torchaudio` |
|---|---|---|
| Primary purpose | Audio analysis, music info retrieval, notebook DSP | Deep learning audio pipelines |
| Core dtype | `numpy.ndarray` float32 | `torch.Tensor` float32 (CPU or GPU) |
| Streaming? | No — assumes an in-memory signal | Partial (functional ops operate on tensors you can batch/stream) |
| Resampler | `librosa.resample` (soxr by default) | `torchaudio.transforms.Resample` (sinc interpolation, batchable) |
| MFCC | `librosa.feature.mfcc` | `torchaudio.transforms.MFCC` |
| Mel spectrogram | `librosa.feature.melspectrogram` | `torchaudio.transforms.MelSpectrogram` |
| Load a `.wav` | `librosa.load` (resamples to 22.05 kHz by default — surprising!) | `torchaudio.load` (keeps native rate) |

The friction points that bite:

- `librosa.load(..., sr=None)` — **always pass `sr=None`** unless you want librosa to resample for you. The default `sr=22050` is a music-IR legacy and will quietly re-rate every file you load.
- `torchaudio.load` returns `(waveform, sample_rate)` where `waveform` is shape `(channels, samples)`. Transpose to match any downstream library that expects `(samples, channels)`.
- Feature outputs are often 2-D `(n_features, n_frames)` and the frame index ↔ time mapping is `t = frame_idx * hop_length / sr`. Hold on to `hop_length` religiously.

## Waveform — the first thing to look at

If something is wrong and you don't have a hypothesis, plot the waveform first. It takes 15 seconds and rules out half the failure modes.

```python
import librosa, numpy as np, matplotlib.pyplot as plt

y, sr = librosa.load("suspicious.wav", sr=None, mono=True)
t = np.arange(len(y)) / sr
plt.figure(figsize=(12, 3))
plt.plot(t, y, linewidth=0.5)
plt.xlabel("time (s)"); plt.ylabel("amplitude")
plt.title(f"waveform, sr={sr} Hz, {len(y)/sr:.2f}s, peak={np.max(np.abs(y)):.3f}")
plt.show()
```

Things that jump out of a waveform plot:

- **Clipping.** The signal is pinned at `±1.0` for stretches. ASR will degrade; TTS re-encoded through a clipping stage sounds crunchy.
- **Silence.** The signal is near zero for the whole file. Common with "forgot to open the mic" bugs.
- **DC offset.** The signal is centred well above or below zero. Rare, but breaks some codecs.
- **Flatline near end.** Common streaming-capture artefact — the recorder filled the buffer and padded with zeros.
- **Clicks and pops.** Discontinuities — usually a resampler or a cross-thread drift (see [Chapter 3](03-buffering-and-chunking-for-streaming.md)).
- **Amplitude envelope vs. expected speech.** Speech has a characteristic 2–5 Hz envelope — syllables. A flat high-level signal is probably music or hold tone; a brief spike inside silence is a cough or a door slam.

## Spectrogram — where frequencies live over time

The log-mel spectrogram is the engineer's workhorse. It's what every modern ASR model is implicitly consuming (as the input layer or something close to it), and it makes visible almost every "weird audio" bug at a glance.

```python
import librosa, librosa.display, matplotlib.pyplot as plt

y, sr = librosa.load("clip.wav", sr=None)
M = librosa.feature.melspectrogram(
    y=y, sr=sr, n_fft=512, hop_length=160, n_mels=80,  # 10 ms hop at 16 kHz
    fmin=0, fmax=sr // 2,
)
M_db = librosa.power_to_db(M, ref=np.max)

librosa.display.specshow(M_db, x_axis="time", y_axis="mel", sr=sr, hop_length=160)
plt.colorbar(format="%+2.0f dB"); plt.title("log-mel spectrogram")
plt.show()
```

What to look for:

- **Hard cutoff at 4 kHz.** The clip has been through a telephone leg. Anything above 4 kHz is zero; sibilance is gone. If ASR is doing badly, that's often why.
- **Hard cutoff at 8 kHz.** The clip was downsampled to 16 kHz at some point. Expected for ASR input.
- **Vertical bars.** Transient events — clicks, pops, hard consonants.
- **Horizontal line at a single frequency.** A hum. 50 Hz (Europe) or 60 Hz (US) means AC line noise. 1 kHz or 440 Hz hum means a test tone leaked in.
- **Smeared, low-contrast spectrum in voiced regions.** Likely over-aggressive noise suppression wiped speech harmonics along with noise.
- **Empty spectrogram for voiced regions.** Audio is silence; check capture.

The `n_fft`, `hop_length`, `n_mels` parameters:

- `n_fft` — the FFT window size. 512 at 16 kHz = 32 ms analysis window. 1024 at 16 kHz = 64 ms. Larger window = finer frequency resolution, coarser time resolution.
- `hop_length` — samples between successive windows. 160 at 16 kHz = 10 ms frame hop, matching what most ASR frontends use.
- `n_mels` — number of Mel bins. 40, 64, 80, 128 are typical. 80 is the de-facto standard for ASR training.

For `torchaudio` the equivalent is `torchaudio.transforms.MelSpectrogram`, which returns a `torch.Tensor` and can run on GPU:

```python
import torch, torchaudio

waveform, sr = torchaudio.load("clip.wav")       # (channels, samples)
if waveform.size(0) > 1:
    waveform = waveform.mean(dim=0, keepdim=True)

mel = torchaudio.transforms.MelSpectrogram(
    sample_rate=sr, n_fft=512, hop_length=160, n_mels=80,
)(waveform)                                      # (1, 80, n_frames)
mel_db = torchaudio.functional.amplitude_to_DB(mel, multiplier=10.0, amin=1e-10, db_multiplier=1.0)
```

## MFCCs — the cheap spectral fingerprint

MFCCs (Mel-Frequency Cepstral Coefficients) are a 13–40-coefficient-per-frame compression of the mel spectrogram. They're the historical ASR input and still a useful lightweight representation for speaker ID, voice activity heuristics, and quick similarity checks.

```python
mfcc = librosa.feature.mfcc(y=y, sr=sr, n_mfcc=13, n_fft=512, hop_length=160)
print(mfcc.shape)  # (13, n_frames)
```

As a voice-agent engineer, you will almost never train on raw MFCCs — modern ASR models consume log-mel directly or raw waveforms. You will read them for **debugging speaker/content similarity**: a quick MFCC + cosine distance between two clips tells you whether a user's "hello" and your TTS's "hello" are plausibly the same content/speaker.

## Pitch (f0) — the dead-TTS detector

Fundamental frequency, `f0`, is the pitch of voiced speech — the rate at which the vocal folds vibrate. It's the single most informative feature for diagnosing "the TTS sounds robotic or dead" bugs.

```python
f0, voiced_flag, voiced_probs = librosa.pyin(
    y, fmin=librosa.note_to_hz("C2"), fmax=librosa.note_to_hz("C7"), sr=sr
)
# f0 is NaN on unvoiced frames.
voiced_f0 = f0[~np.isnan(f0)]
print(f"f0 mean  = {np.nanmean(f0):.1f} Hz")
print(f"f0 stdev = {np.nanstd(f0):.1f} Hz")
```

Typical ranges:

| Speaker | f0 range (Hz, relaxed speech) |
|---|---|
| Adult male | ~85 – 180 |
| Adult female | ~165 – 255 |
| Child | ~250 – 400 |

What to look for:

- **Monotone TTS.** `np.nanstd(f0)` is near zero — the model is producing flat pitch. Usually a prosody model regressed; test with a reference TTS clip to confirm.
- **f0 outside 60–400 Hz.** You're reading pitch off noise, not voice. Something upstream has stripped speech content.
- **Sudden jumps.** TTS phrase boundaries are often associated with pitch resets. Natural speech has smooth f0 contour; splice artefacts show up as vertical jumps.
- **Mismatch against expected speaker.** If you're running a voice-clone and f0 landed in a different register, your reference speaker was mis-conditioned.

Pitch estimators have their own quirks — `pyin` is slow but accurate; CREPE is a learned alternative; `torchaudio.functional.detect_pitch_frequency` is quick but coarse. For debugging, slow-and-accurate is the right trade.

## Energy / RMS — the level check

Signal level dictates whether ASR hears you and whether VAD fires. Measure RMS (root-mean-square) in short windows, convert to dBFS, and plot.

```python
hop = 160  # 10 ms at 16 kHz
rms = librosa.feature.rms(y=y, frame_length=512, hop_length=hop)[0]
rms_db = 20 * np.log10(np.maximum(rms, 1e-10))
```

Rules of thumb:

- **Target speech RMS: around −20 dBFS.** Dialogue mastered for broadcast sits around there.
- **Below −45 dBFS:** too quiet. ASR will miss short words; VAD will miss onsets.
- **Above −3 dBFS or clipping:** too hot. Compressors or ASR models trained on normal levels may misfire.

A quick normalisation pass with `pyloudnorm` (ITU-R BS.1770 loudness) brings mixed-level corpora to a consistent reference — cleaner than peak-normalising, because peak normalisation is dominated by a single loud sample.

## Debugging recipes

### Recipe 1 — "the ASR is returning empty strings"

1. Plot waveform. If it's silence, your capture layer is broken — stop and fix that.
2. If there's signal, check peak and RMS. If RMS is below −45 dBFS, boost gain and retry.
3. Plot log-mel. If there's a hard cutoff at 4 kHz, you're on a telephony leg — confirm the ASR provider knows the source is 8 kHz narrowband, or upsample+re-flag as wideband (not the same as real wideband — don't claim quality you don't have; just unblock the provider).
4. Check sample rate against what you're telling the provider. A WAV header claiming 16 kHz where the samples are actually 8 kHz plays as half-speed garbage; the ASR will return something weird or nothing.

### Recipe 2 — "the TTS sounds flat / dead / robotic"

1. Load the output. Plot log-mel. Look for clean harmonics in voiced regions; a smeared blob means the vocoder produced junk.
2. Estimate f0. If `nanstd(f0)` is near zero, the prosody model lost its contour.
3. Compare against a known-good reference synthesised with the same voice. If both are flat, the voice itself is the issue; if only the new one is flat, the request parameters changed (temperature? style tag? ssml?).
4. Listen through a mono reference monitor. A lot of "flat" complaints are a stereo-vs-mono rendering issue, not a model issue.

### Recipe 3 — "the barge-in isn't firing"

1. Plot waveform and RMS envelope of the far-end audio. If the user utterance RMS peaks are lower than the VAD threshold, barge-in literally can't see them.
2. Check whether echo cancellation (Chapter 5) is attenuating the user voice along with the TTS echo.
3. Verify the audio reaching the VAD is 16 kHz PCM16 mono, not the raw 48 kHz WebRTC stream.

## Summary

- Learn to read five features: waveform, log-mel spectrogram, MFCCs, f0, RMS.
- `librosa` for interactive debugging; `torchaudio` for pipeline / GPU work.
- Clipping and sample-rate mismatch show up in the waveform. Band limits and noise show up in the spectrogram. Dead prosody shows up in f0 variance. Level issues show up in RMS.
- The debugging stance: form a hypothesis, extract the feature that confirms or refutes it, iterate. Do not stare at the audio and listen 20 times — plot it.

Next chapter: the buffering layer — frames, ring buffers, and the chunking rules that keep a streaming pipeline healthy.
