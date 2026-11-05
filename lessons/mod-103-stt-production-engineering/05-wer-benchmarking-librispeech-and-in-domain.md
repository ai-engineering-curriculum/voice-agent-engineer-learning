# 05. Benchmarking WER / CER / MER — LibriSpeech, TED-LIUM 3, FLEURS, and Your Own Gold Set

## Why this chapter

Every decision in the previous chapters routes back through a number. Which provider is better? Is the keyterm list helping? Is the fine-tune worth the latency it cost? You cannot answer any of these without an honest benchmark, and most published STT benchmarks are not honest about what they are measuring.

This chapter teaches the engineer's working depth on benchmarking: what WER actually computes, why CER and MER exist, which standard corpora to use for the floor, how to construct an in-domain gold set that won't lie to you, and how to translate the numbers into the KPIs your product owner cares about. The deeper statistical machinery — significance testing between models, per-stratum calibration analysis, sample-size planning — belongs to `model-evaluation-engineer` and is named in [Chapter 7](07-boundary-to-nlp-eval-and-training-engineers.md).

## The metrics

### Word Error Rate (WER)

The standard metric. Edit distance on words, normalised:

```
WER = (S + D + I) / N
```

where, comparing the hypothesis transcript to the reference:
- **S** = substitutions (hyp has a different word than ref at an aligned position)
- **D** = deletions (ref word absent from hyp)
- **I** = insertions (hyp word absent from ref)
- **N** = number of words in the reference

A WER of 0 is perfect. A WER of 1.0 means the hypothesis is as wrong as if it had been empty. WER above 1 is possible: a hypothesis that inserts more wrong words than the reference contains will report >1.0.

Note: *WER is not symmetric.* Swap hypothesis and reference and you get a different number. The provider's transcript is always the hypothesis.

### Character Error Rate (CER)

Same formula at the character level. Useful when:

- **Languages without word boundaries** — Chinese, Japanese, Thai. WER requires a word tokenisation that doesn't naturally exist; CER sidesteps it.
- **Rich-morphology languages** — Finnish, Turkish, German compounds — where a single-character-off error costs you a whole wrong "word" under WER. CER gives a smoother signal.
- **Comparing models that differ in tokenisation conventions.** The CER is invariant.

CER on English runs roughly 0.3–0.5× the WER — a 10 % WER corresponds to roughly 3–5 % CER on typical text.

### Match Error Rate (MER) and Word Information Lost (WIL)

Two less-common metrics worth knowing by name ([Morris, Maier, and Green, 2004 — "From WER and RIL to MER and WIL"](https://www.isca-archive.org/interspeech_2004/morris04_interspeech.pdf)):

- **MER = (S + D + I) / (S + D + I + C)** — matches rather than reference length. `C` is the count of correct words. MER is bounded in `[0, 1]`, which WER is not, making it more comparable across experiments.
- **WIL** captures how much *information* was lost rather than how many words were wrong; useful when you care about downstream NLU rather than exact transcription.

In practice, teams report WER plus occasionally CER (for multilingual work). Report MER when you need a bounded metric or when comparing across short vs. long utterances.

### What WER does *not* tell you

- **Where the errors are.** A 5 % WER with every error on content words is worse for your agent than 10 % WER with errors on filler words.
- **Which errors cost business money.** "Fifty four" → "five four" is one substitution, worth one WER point, worth a wrong payment in a banking agent.
- **Whether the errors are systematic or random.** 10 % WER evenly distributed is different from 10 % WER concentrated on your top-20 brand terms.

Always stratify. A single number is a conversation-starter, not a conclusion.

## Text normalisation: the quiet source of 2–5 WER points

Before you compute WER, the hypothesis and reference must be **normalised** — put into a common surface form so that trivial differences don't register as errors.

The standard set of normalisation rules:

- Lowercase everything.
- Remove or standardise punctuation.
- Expand contractions ("don't" → "do not") *or* leave them — pick one and stick to it.
- Convert digits to words consistently ("54" ↔ "fifty four") per your reference style.
- Collapse whitespace.
- Strip hesitation and filler markers ("um", "uh") — *or* keep them; pick one.
- Spell out acronyms uniformly.

Whisper ships an opinionated normaliser — the [`whisper.normalizers.EnglishTextNormalizer`](https://github.com/openai/whisper/blob/main/whisper/normalizers/english.py) and its basic multilingual counterpart. They are the de-facto defaults in the open-source community, and are what most papers use when reporting WER on standard benchmarks. Using them is a reasonable default; **document the version and configuration**.

Why it matters: the difference between "normalised WER" and "raw WER" on the same model and corpus is routinely 2–5 points. Compare apples to apples; cite the normaliser with its version.

## Standard English benchmarks

### LibriSpeech

[OpenSLR 12](https://www.openslr.org/12/). 1 000 hours of read audiobook English, derived from LibriVox. The *de facto* floor for English ASR benchmarking.

Splits:

- `dev-clean` / `test-clean` — clean, close-talk, read audiobook speech. A model should get WER < 5 % here; anything higher is a provider you should not ship.
- `dev-other` / `test-other` — harder samples (accented speakers, more audio variability). A better stress test; still read speech.

What LibriSpeech tells you: "this ASR can transcribe clean read English of an audiobook flavour." That is a necessary floor, not a sufficient test for voice agents. **Users do not read audiobooks on phone calls.** A model that passes LibriSpeech and fails your agent's gold set is a common outcome; the reverse is also real but less common.

### TED-LIUM 3

[Rousseau et al., LREC 2014 (TED-LIUM); TED-LIUM 3 2018](https://www.openslr.org/51/). ~450 hours of TED talks with transcripts. Closer to conversational than LibriSpeech but still prepared speech by professional speakers.

Use TED-LIUM 3 as a step between LibriSpeech and your agent gold set — more vocabulary variety, more spontaneous phrasing, reasonable audio conditions.

### GigaSpeech and Earnings-22

Two heavier benchmarks worth knowing:

- **GigaSpeech** — [SpeechColab 10 000-hour corpus](https://github.com/SpeechColab/GigaSpeech), multi-domain (podcasts, YouTube, audiobooks). Closer to a mixed-domain model-evaluation surface.
- **Earnings-22** — [`revdotcom/speech-datasets` earnings-22](https://github.com/revdotcom/speech-datasets). Noisy conference-call audio in the finance domain. A useful proxy for "hard business audio" without being your actual domain.

These are tests `model-evaluation-engineer` or `nlp-engineer` will reach for in deeper comparisons. For your L35 engineering work, LibriSpeech + TED-LIUM 3 + your own gold set is almost always enough.

## Multilingual benchmarks: FLEURS, Multilingual LibriSpeech, Common Voice

### FLEURS

[Conneau et al., 2022 — FLEURS: Few-shot Learning Evaluation of Universal Representations of Speech](https://arxiv.org/abs/2205.12446). 102-language parallel ASR corpus derived from FLoRes-101 (a translated Wikipedia-sentence dataset).

Why FLEURS matters: it is the first multilingual ASR benchmark where results are **comparable across languages** — the same sentences, translated and read in each language. Publishing a per-language WER chart from FLEURS tells you honestly where a model's language coverage is strong and where it's catastrophic.

Use FLEURS when you need to answer "which of my languages is this model actually good at." See [Chapter 6](06-multilingual-code-switching-and-accents.md).

### Multilingual LibriSpeech (MLS)

[Pratap et al., 2020](https://arxiv.org/abs/2012.03411). 50k+ hours in 8 languages, derived from LibriVox (same flavour as English LibriSpeech, in more languages). Good for European-language ASR baselining.

### Common Voice

[commonvoice.mozilla.org](https://commonvoice.mozilla.org/). Crowdsourced multi-speaker read speech in 100+ languages. Quality is uneven but coverage is broad; useful for stress-testing accent diversity where a benchmark corpus doesn't exist.

## The in-domain gold set

LibriSpeech tells you the model works on audiobooks. Your in-domain gold set tells you whether the model works on your users. Building it is one of the most under-glamoured and highest-leverage pieces of work an L35 voice engineer does.

### Size and composition

- **At least 2–5 hours of audio.** 1 000+ utterances is a reasonable floor; 5 000+ makes per-stratum numbers trustworthy.
- **Sampled from production, not studio.** Real callers, real phones, real noise, real accents. If you cannot ship production audio out (compliance), sample within a secure environment.
- **Stratified** — so you can report per-stratum WER:
  - Dialect / accent clusters relevant to your users.
  - Audio-source channel (PSTN / WebRTC / SIP / mobile vs. desktop).
  - Utterance type (open-ended user turn / numeric response / name / confirmation / transfer-intent).
  - Noise level (SNR bins).
  - Language, if multilingual.

### Transcription: inter-rater reliability

A gold set is worth exactly what its transcripts are worth. Transcription protocol:

1. **Style guide.** Write a one-page spec: punctuation, capitalisation, digit vs. word, filler handling, inaudible markers, PII redaction. Pin the Whisper-flavoured normaliser you'll use at scoring time so transcribers can match it.
2. **Two independent transcribers per sample.** They do not see each other's work.
3. **A third resolver** for disagreements — either a senior transcriber or a reviewing engineer.
4. **Inter-rater agreement measurement.** Compute WER between transcriber A and transcriber B on each sample (treat A as reference, B as hypothesis). The pooled number is your transcription noise floor; no ASR can meaningfully beat it.
5. **Reject the gold set if inter-rater WER is above ~2 %.** Clean the style guide; re-run; iterate.

The inter-rater floor is the honest ceiling on model quality on this corpus. If your ASR reports 3 % WER against a gold set with 5 % inter-rater disagreement, the ASR might actually be *better than the gold*.

Industry benchmarks on transcription quality: a professional human transcriber on clean audio achieves roughly 5 % WER against a reference; ~4 % is near the physical floor ([Stolcke and Droppo, 2017 — "Comparing Human and Machine Errors in Conversational Speech Transcription"](https://arxiv.org/abs/1708.08615)). Your gold set's floor will depend on your audio quality and transcriber training.

### Freezing and refreshing

- **Freeze the gold set version.** Every WER you report cites the gold-set version hash. When you add new audio, version-bump; don't mutate the existing set.
- **Refresh quarterly.** Production audio drifts (new users, new accents, new topics). An unrefreshed gold set reports regressions that aren't real and misses regressions that are.
- **Keep a holdout**. If you have 5 hours of gold, keep 20 % as a "red team" set that is never used in day-to-day decisions and is checked only before major releases.

## Tooling

Where to get the computation done without rolling your own:

- **`jiwer`** ([github.com/jitsi/jiwer](https://github.com/jitsi/jiwer)) — the de-facto Python library for WER/CER/MER/WIL + text normalisation. Use this.
- **`torchmetrics.text.WordErrorRate` / `CharErrorRate`** — PyTorch's built-in versions, if you're already in a PyTorch eval pipeline.
- **Whisper's normaliser** — bundled with [`openai/whisper`](https://github.com/openai/whisper); importable as `whisper.normalizers`.
- **NIST SCLITE** — [github.com/usnistgov/SCTK](https://github.com/usnistgov/SCTK). The old reference implementation, with sophisticated alignment visualisation and stratified reporting. Overkill for day-to-day work; worth knowing for deep investigations.

A minimal `jiwer`-based scoring loop:

```python
import jiwer
from whisper.normalizers import EnglishTextNormalizer

norm = EnglishTextNormalizer()

def score_asr(gold_set):
    refs, hyps = [], []
    for sample in gold_set:
        refs.append(norm(sample.reference))
        hyps.append(norm(sample.hypothesis))
    return {
        "wer": jiwer.wer(refs, hyps),
        "cer": jiwer.cer(refs, hyps),
        "mer": jiwer.mer(refs, hyps),
    }
```

Add bootstrap confidence intervals for the WER by resampling the sample list with replacement; plot the distribution. A reported WER without a CI is a point estimate masquerading as a conclusion.

## Translating WER to business KPIs

The number product owners actually care about is not WER. It is **cost per handled call**, **deflection rate** (% of calls the agent resolves without escalation), **escalation rate**, **customer satisfaction** (CSAT / NPS), **average handling time (AHT)**, and **first-contact resolution (FCR)**.

The job of this translation is to turn a WER delta into a business-KPI delta your product org can decide against.

### The translation

A useful mental model, tightened up with instrumentation:

1. **Instrument the call.** Tag every call with the ASR WER (against a sampled transcript), the number of clarifying turns, the escalation outcome, AHT, resolution flag.
2. **Regress KPI on WER across a sample of calls.** The regression coefficient is a first-order estimate of "how much does 1 point of WER cost us." It will be noisy; a bootstrap CI gives the honest uncertainty.
3. **Stratify the regression.** WER on numbers vs. names vs. filler matters very differently. The LLM handoff design (Chapter 3) is the dominant moderator; a 10 % WER with a well-calibrated reject-and-ask pattern can be better than 5 % WER with a naive pass-through.

### Common translations you'll quote

Treat these as the *shapes* of the business argument; always reproduce the numbers on your own data before quoting them to a stakeholder.

- "A 2-point WER improvement on the drug-name stratum reduces our clarify-rate on refill calls from 28 % to 19 %. Each clarify adds ~18 s of AHT; this is worth roughly ${X} per 1 000 calls."
- "A 30 % reduction in number-substitution errors moves our FCR on billing calls from 71 % to 75 %. That's roughly ${Y} in reduced callbacks per week."
- "WER on first-utterance greetings turns out not to correlate with any downstream KPI; we can accept a higher WER there without product consequence."

The third bullet is the one that saves you the most engineering time: the pieces of WER that *don't* matter are as important to identify as the pieces that do.

## The KPI → WER feedback loop

The business-KPI translation runs both directions. KPIs drift; WER is the diagnostic.

- **Monitor WER and KPI jointly.** A sudden drop in deflection rate without a WER regression points you to the LLM or orchestration layer; a WER regression with no KPI drop suggests your handoff absorbs the error (good) or you're not measuring the right KPI (bad).
- **Alert on WER drift.** A rolling-window WER against a continuously sampled gold set is the primary ASR health signal. Module 112 covers the observability design.
- **Loop into retraining.** WER drift past threshold fires the retraining / re-tuning cycle from Chapter 4.

## Common anti-patterns

- **Reporting WER on the training split.** Classic overfitting-report pattern. Cite the test-split name and the dataset version.
- **Reporting WER without a normaliser.** The number is uninterpretable. Cite the normaliser.
- **Benchmarking only on LibriSpeech.** It's a floor, not a gold set. See the opening of this chapter.
- **One-rater gold sets.** Inter-rater disagreement is high enough that single-rater gold sets routinely report WERs lower than the human floor. You cannot trust the number.
- **Reporting a weighted global WER only.** Stratify by call segment / accent / utterance type; the headline number masks the diagnostic signal.
- **No bootstrap CI.** "We went from 8.3 % to 7.9 %" over a 500-utterance sample might be within sampling noise. Report the interval.
- **No business-KPI translation.** The provider pitch deck will be all WER and no dollars. Your scorecard, in the other direction, needs the translation to defend decisions upwards.

## Summary

- WER is the reference metric; CER and MER have well-defined roles; always pair with text normalisation and bootstrap CIs.
- LibriSpeech / TED-LIUM 3 are floors; your in-domain gold set is the real test.
- Gold sets live or die by inter-rater reliability; two independent transcribers plus a resolver, with a measured transcription-noise floor.
- Translate WER to KPIs upstream; monitor both and let WER drift trigger retraining.
- Deeper statistical methodology — hypothesis testing between models, per-stratum power analysis — belongs to `model-evaluation-engineer`. See [Chapter 7](07-boundary-to-nlp-eval-and-training-engineers.md).

Next chapter: multilingual, code-switching, and accent scenarios — the FLEURS per-language picture, SeamlessM4T, and the local-vs-foundation-model trade-off.
