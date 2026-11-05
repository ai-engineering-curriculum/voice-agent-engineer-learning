# exercise-01: Sample Rates, Codecs, and Bit Depth Tour

**Estimated effort:** 2 hours · **Output:** a short Jupyter notebook (or equivalent script + notes) + a one-page symptom table

## Objective

Internalise the "four numbers" (sample rate, bit depth, channels, container/codec) from [Chapter 1](../01-sample-rates-bit-depth-and-pcm.md) by **deliberately creating** each common mismatch on a reference tone, listening to the result, and writing down the diagnostic signature.

You will finish this exercise able to recognise a sample-rate / bit-depth / channel / codec mismatch within 30 seconds of hearing or seeing one.

## Prerequisites

- [Chapter 1 — Sample Rates, Bit Depth, Channels, and PCM vs. Compressed](../01-sample-rates-bit-depth-and-pcm.md).
- Python 3.10+ with `numpy`, `soundfile`, `librosa`, and either `audioop` (Python 3.12 or older — moved in 3.13) or an equivalent µ-law/A-law library. `ffmpeg` on PATH is helpful.
- A quiet place and headphones.

## Steps

### Part A — Build the reference corpus (30 minutes)

1. Generate a **reference tone corpus** — a 1-second 440 Hz sine wave and a 3-second recording of your own voice saying "test one two three." Save each as:
   - PCM16 mono at 8 kHz, 16 kHz, 24 kHz, 48 kHz (four WAV files per source).
   - PCM24 mono at 16 kHz (one file per source).
   - PCM16 stereo (identical channels) at 16 kHz (one file per source).
   - µ-law 8 kHz (one file per source).
   - PCM16 mono at 16 kHz re-encoded through Opus at 32 kbps and 8 kbps (two files per source; use `ffmpeg -c:a libopus`).

   You should end up with ~20 files total.

2. Verify each file. For each, use `soundfile.info` or `ffprobe` to confirm the sample rate, bit depth, and channel count match what you intended. Log the actual file size and compare to the theoretical one for PCM (sr × bytes/sample × seconds × channels). Note any discrepancies.

### Part B — Create the mismatches deliberately (60 minutes)

3. Write a small script that, given a file, **plays or saves it as if it were a different rate / depth / channel count**. The script should produce:

   - A "pitch-shift" set: take the 24 kHz file and treat it as 48 kHz (save or play as 48 kHz without resampling). Measure the actual pitch of the sine with `librosa.pyin` and confirm it's 880 Hz.
   - A "bit-depth drop" set: convert float32 → PCM16 → PCM8, then back. Capture the audible quantisation noise in the voice sample.
   - A "stereo-mono confusion" set: duplicate a mono file into fake stereo, then also create a stereo file where left is voice and right is silence; run both through your favourite VAD or feature extractor. Record what each library does with the extra channel.
   - A "µ-law without conversion" set: read the µ-law file as if it were 8-bit linear PCM. Describe what you hear.
   - An "Opus-reencoded-to-death" set: Opus at 8 kbps → decode → Opus at 8 kbps → decode, three times. Diff the spectrogram with the original.

4. For each of the above, **listen to the result** and plot its waveform or log-mel spectrogram (Chapter 2 preview — `librosa.feature.melspectrogram` + `librosa.display.specshow` is enough).

### Part C — Write the symptom table (30 minutes)

5. Produce a **one-page symptom table** with three columns: *Mismatch*, *What it sounds like*, *What it looks like on the plot*. Include the five mismatch families above. Keep entries to one or two sentences each.

6. **Add two "real" rows.** Pick two voice files from a provider you've used (your TTS output; an ASR sample from a dataset) and describe what *correct* audio for that source looks like on the same plots, so your future-self has a baseline.

## Starter guidance

- The `soundfile` library's `subtype` parameter controls PCM bit depth (`"PCM_16"`, `"PCM_24"`, `"PCM_32"`, `"FLOAT"`). `audioop.lin2ulaw` / `ulaw2lin` handle µ-law.
- For the pitch-shift demo, do *not* resample — change only the sample-rate metadata. `soundfile.write("out.wav", y, 48000)` on data captured at 24 kHz does exactly this.
- For the Opus round-trip, drive `ffmpeg` from Python with `subprocess`. `ffmpeg -y -i in.wav -c:a libopus -b:a 8k out.opus` then `ffmpeg -y -i out.opus out_decoded.wav`.
- Be careful with headphones; a mis-rated file can be loud. Keep volume low on first play.

## Acceptance criteria

- [ ] Reference corpus of ~20 files, each verified against its intended four-number spec.
- [ ] Each of the five mismatch families reproduced, with saved output files and plots.
- [ ] Pitch-shift mismatch empirically confirmed via measured f0 (should be exactly 2× or 0.5× the reference).
- [ ] Symptom table with **five mismatch rows + two baseline rows**, each with "sounds like" and "looks like on the plot" entries.
- [ ] A one-paragraph reflection at the bottom: which of the mismatches would you expect to see most often in a browser → agent voice loop, and which in a PSTN → agent loop? Justify in terms of where resampling happens in each architecture.

## Stretch goals

- Extend the µ-law work to A-law and confirm by inspection that the two curves are *not* interchangeable (listen to each played as the other).
- Add a channel-order mismatch: write a stereo file with the channels labelled wrong in the WAV header (left/right swapped at write time). Does any downstream library catch it? (Most don't.)
- Automate the whole thing: a pytest-style script that generates the corpus, applies each mismatch, extracts the diagnostic feature, and asserts the expected signature. Keep this script — it's a real regression test for future pipeline work.
- Reproduce the "16 kHz claimed, actually 8 kHz" bug: write a WAV where the header says 16000 but the samples are actually from an 8 kHz source. Run it through your favourite ASR provider's playground. Does the provider complain, or silently return garbage?

## Keep it

The symptom table is a referenceable artefact. Pin it in your notes or your team's runbook. You will walk a future colleague through it the first time they hit a "the audio sounds wrong but I can't tell why" ticket.
