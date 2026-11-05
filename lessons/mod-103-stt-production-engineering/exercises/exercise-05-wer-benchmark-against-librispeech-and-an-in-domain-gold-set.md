# exercise-05: WER Benchmark — LibriSpeech + In-Domain Gold Set

**Estimated effort:** 3 hours · **Output:** a reproducible benchmark script + a mini in-domain gold set + a two-tab report (LibriSpeech floor, in-domain truth)

## Objective

Run the honest benchmark. Score at least two STT providers on **LibriSpeech test-clean** (the floor) and on a **small in-domain gold set you build yourself** (the real test). Report WER with bootstrap CIs, a per-stratum breakdown on the in-domain set, and a two-paragraph translation to a business KPI.

You will finish with a reusable scoring script + the in-domain gold set + the discipline that keeps you from citing WER numbers you cannot defend.

## Prerequisites

- [Chapter 5 — Benchmarking WER / CER / MER](../05-wer-benchmarking-librispeech-and-in-domain.md).
- Python 3.10+ with `jiwer`, `datasets` (Hugging Face), `openai-whisper` (for the normaliser), `numpy`, `matplotlib`.
- Two STT providers you can run against audio. One can be self-hosted Whisper / Whisper-turbo + faster-whisper (free); the other should be a managed provider (Deepgram free tier, AssemblyAI, Google STT).
- 30–60 minutes of your own audio (or audio you have rights to transcribe) for the in-domain gold set.
- Enough time to transcribe or review ~15–30 minutes of audio carefully — this exercise's in-domain set lives or dies by the transcripts.

## Steps

### Part A — The LibriSpeech floor (45 minutes)

1. Download **LibriSpeech test-clean** (a subset is fine; the full set is 5.4 hours, ~2 620 utterances). The `librispeech_asr` dataset on Hugging Face is the easiest path.
2. Write `score_asr.py`: given a dataset iterable yielding `(audio, reference_text)`, a provider adapter, and a normaliser, compute:
   - Overall WER with a bootstrap 95 % CI (resample utterance indices with replacement, 1 000 bootstrap draws, report the 2.5th and 97.5th percentiles).
   - Overall CER.
   - Overall MER.
   - A per-utterance WER distribution (for the plot in step 4).
3. Run both providers on test-clean through `score_asr.py`. Use `whisper.normalizers.EnglishTextNormalizer` for both hypothesis and reference. Save the hypothesis transcripts + the scored output to disk; you will want them again.
4. Plot a histogram of per-utterance WER, overlaid for both providers. Note the mode, the tail, and any outlier utterances.

### Part B — Build the in-domain gold set (75 minutes)

5. Pick 50–100 utterances from your domain audio (the same audio you used in exercises 2–4 is fine). Aim for diversity: short confirmations, long open-ended turns, numeric content, proper nouns, hesitant speech, clean speech. 15–30 minutes of total audio is a reasonable target.
6. **Transcribe with an inter-rater protocol.** Either:
   - (Full protocol.) Two independent transcribers; third resolver on disagreements; compute transcriber-vs-transcriber WER as the noise floor; proceed only if under ~2 %.
   - (Minimum protocol.) You transcribe first. Wait 24 hours. Transcribe the same audio again without looking at your first pass. Compute first-pass-vs-second-pass WER. This is a weak noise floor but it's honest.
7. Write a one-page style guide (same spec you'd hand to a transcriber): punctuation, digits vs. words, filler handling, inaudible markers, PII redaction. Pin the Whisper normaliser version. Commit the style guide and the transcripts to version control.
8. **Stratify** the gold set. For each utterance, label:
   - Speaker accent cluster (your best judgment; two or three labels).
   - Audio channel (phone / in-app / studio / etc.).
   - Utterance type (greeting / numeric / name / open-ended / confirmation).
   - Noise level (quiet / moderate / loud — eyeball or RMS).
9. Save the gold set as a structured artefact (JSONL with `audio_path`, `reference`, strata, transcriber IDs).

### Part C — Score the in-domain set (30 minutes)

10. Run both providers on the in-domain gold set through `score_asr.py`.
11. Produce a **two-tab report** (Markdown or notebook):
    - **Tab 1 — LibriSpeech floor.** WER / CER / MER, 95 % CI, per-utterance histogram.
    - **Tab 2 — In-domain truth.** WER / CER / MER with CI, **plus per-stratum WER tables**: by accent, by channel, by utterance type. Call out any stratum where a provider dramatically differs from its pooled number.
12. Compare: where does the ranking between providers change between LibriSpeech and your gold set? That delta is the exercise's main lesson.

### Part D — The business-KPI translation (30 minutes)

13. Pick one plausible business KPI for your domain (clarification rate, first-contact resolution, average handling time, deflection rate). Make a **back-of-the-envelope** translation:
    - For one point of WER improvement, estimate how many additional utterances are transcribed correctly in your 50k minutes / month deployment.
    - Of those, estimate what fraction land on the KPI you picked (e.g., "half of the newly-correct utterances were content words that the LLM would have had to clarify otherwise → half × 50 % clarification rate reduction per corrected utterance").
    - Translate to a dollar or operational-metric figure.
14. Write a two-paragraph recommendation: on the LibriSpeech floor, Provider X wins; on the in-domain gold set, Provider Y wins by Z points, which translates to roughly `${dollars}` or `{percent}` KPI movement. Which would you ship? Under what phasing (if any)?

## Starter guidance

- The HuggingFace `datasets` library can stream LibriSpeech without downloading the whole thing — useful for the quick run. `load_dataset("librispeech_asr", split="test.clean", streaming=True)`.
- For the bootstrap CI: resample the **list of per-utterance errors**, not individual words. Word-level resampling ignores the per-utterance correlation structure.
- For inter-rater agreement: compute WER with transcriber A as reference and transcriber B as hypothesis. The number is symmetric in interpretation even though WER itself isn't (approximately).
- Resist the urge to score a provider "in general." Each run cites the provider, the model version, the date, the audio corpus, the normaliser version. Future-you will thank you.
- The two-tab report is the output your product org can argue with. Don't bury the in-domain numbers behind the LibriSpeech numbers.

## Acceptance criteria

- [ ] `score_asr.py` is a reusable script; runs against any dataset iterable and provider adapter, emits WER/CER/MER + bootstrap CIs + per-utterance distributions.
- [ ] LibriSpeech test-clean scored for both providers with the standard normaliser.
- [ ] In-domain gold set of 50–100 utterances, versioned, with a style guide and strata labels committed.
- [ ] Inter-rater noise floor measured (full or minimum protocol).
- [ ] In-domain WER / CER / MER with bootstrap CIs; per-stratum breakdowns by accent, channel, and utterance type.
- [ ] Observed ranking change between LibriSpeech and in-domain is called out explicitly.
- [ ] Two-paragraph business-KPI translation + a defensible recommendation with phasing.

## Stretch goals

- **Significance test.** For the in-domain head-to-head, run a McNemar test on per-utterance correct/wrong and a paired bootstrap on WER. Report the p-value, and reflect: how confidently can you distinguish the two providers at this sample size? (The honest answer is often "not very" at 100 utterances — which is itself a lesson.)
- **Per-language slice.** If any utterances in your gold set are in a second language, report per-language numbers. The multilingual picture from Chapter 6 lives here in miniature.
- **Add a "with keyterm list" column.** Re-score the in-domain set with your exercise-04 keyterm list applied. Does the ranking between providers change again? (It often does — vendors differ in how well their keyterm mechanism works.)
- **Add a fine-tune baseline estimate.** For the strata where both providers miss the target, estimate what a fine-tune could plausibly achieve, using the published WER numbers for fine-tuned Whisper on similar domains. Compare to the keyterm delta and the business-KPI translation. Does the fine-tune pay for itself?
- **Submit the in-domain gold set to code review.** Have a peer review your style guide and transcribe five utterances blindly. Compare their transcripts to yours. This is the most expensive step and the one that will upgrade the gold set the fastest.

## Keep it

The `score_asr.py` script + the versioned in-domain gold set are the production-grade artefacts this exercise produces. They go into the Module 112 observability loop (continuous sampling against the gold set → WER drift alerts) and into every subsequent provider-selection or fine-tune evaluation. Give them a real home in the repo; this is not throwaway code.
