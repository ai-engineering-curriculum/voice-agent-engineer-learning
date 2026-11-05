# 01. The ASR Eval Harness — WER, CER, MER on FLEURS + LibriSpeech + TED-LIUM 3 + Your Gold Set

## Why this chapter

Every voice agent has an ASR tier at its upstream edge. In mod-103 you picked a vendor and tuned it; in this module you have to prove, on demand, that the ASR tier is still good enough to meet the product's error budget, and that a vendor swap or a model-version bump has not quietly regressed it. "Still good enough" is not an opinion — it is a number produced by a reproducible harness run against a documented set of audio with reference transcripts. The ASR eval harness is the earliest, cheapest, most mechanical eval you will build in this module, and it is the one your CI gate (Chapter 4) relies on to block a regression.

This chapter defines the metrics, the public datasets you anchor against, the in-domain gold set you build yourself, how you measure inter-rater reliability on your own labels, how you read a confidence-calibration plot, and how you stratify results so a single average does not hide a demographic-specific failure.

## The three metrics you need, and what each one is lying about

### Word Error Rate (WER)

WER is the Levenshtein edit distance between the reference transcript and the hypothesis, normalised by the number of reference words:

```
WER = (S + D + I) / N
```

where `S` is substitutions, `D` is deletions, `I` is insertions, and `N` is the number of words in the reference. The canonical Python implementations are [`jiwer`](https://github.com/jitsi/jiwer) and [`torchmetrics`](https://lightning.ai/docs/torchmetrics/stable/text/word_error_rate.html); both agree on the formula.

Three well-known traps:

- **Normalisation changes the number.** Casing, punctuation, numbers ("twenty three" vs "23"), contractions ("don't" vs "do not") all count unless you normalise both sides first. The OpenAI Whisper paper describes an "English text normalizer" used for its public WER numbers; the `whisper.normalizers` package in the [`openai-whisper`](https://github.com/openai/whisper) repository is a reasonable reference implementation. Pick one normaliser and apply it identically to reference and hypothesis across every run. Document which normaliser you used; a WER number without that attribution is not comparable to anyone else's.
- **Insertions can push WER above 100%.** A hallucinating model that emits long filler text can drive WER arbitrarily high. WER is bounded below by 0 but has no upper bound.
- **WER weights every word equally.** A deletion of "not" is the same cost as a deletion of "the". For most products "not" matters a lot more. WER is a diagnostic, not an outcome metric — content eval (Chapter 4) is where meaning-preservation lives.

### Character Error Rate (CER)

CER is the same edit distance, applied at the character level, normalised by reference character count. For languages without reliable word boundaries (Mandarin, Japanese, Thai) CER is the primary metric and WER is often not meaningful. For English, CER is a tie-breaker when WER is close — a system that gets one character wrong per word (plural vs singular, "a" vs "an") has a lower CER than a system that fully substitutes a word. Both `jiwer` and `torchmetrics` expose CER directly.

### Match Error Rate (MER)

MER normalises by the length of the *longer* of reference and hypothesis rather than only the reference:

```
MER = (S + D + I) / (N + I)
```

MER is bounded on `[0, 1]`, which makes it stable in the presence of hallucinated insertions. If you have a model that hallucinates when there is no speech (Whisper historically does; see the [Whisper GitHub issues](https://github.com/openai/whisper/issues) for the long history of this failure mode), WER can balloon into meaningless numbers while MER stays bounded. Report both. MER is defined in Morris, Maier, and Green's 2004 paper *"From WER and RIL to MER and WIL"*; `jiwer` computes it directly.

Pick a primary metric per language (WER for most European languages, CER for CJK), and report MER alongside as the hallucination-robust companion.

## The public corpora you anchor against

Public corpora are useful because they are *comparable* — any vendor, any researcher, any model can report a WER against FLEURS test-clean and that number can be cross-checked. They are useful because the labels are frozen, which means your harness's WER number only moves when the ASR system moves. They are *not* the final gate; your in-domain gold set is. But a model that is off the public-corpus curve is one you cannot defend.

### FLEURS (Few-shot Learning Evaluation of Universal Representations of Speech)

[FLEURS](https://huggingface.co/datasets/google/fleurs) is Google's 102-language parallel speech dataset built from the [FLoRes-101](https://ai.meta.com/research/flores/) translation benchmark — the same 2000-ish sentences read by native speakers in each language. See the FLEURS paper: [Conneau et al., "FLEURS: Few-shot Learning Evaluation of Universal Representations of Speech"](https://arxiv.org/abs/2205.12446). Properties that make it the right multilingual anchor:

- **Parallel** — the same utterances in every language. If a model fails on a sentence in one language and succeeds in another, you can attribute the difference to the acoustic model, not to the content.
- **Multi-speaker per language** — reduces speaker overfitting as a confound.
- **Studio-quality** — low noise, close-mic. FLEURS tests *what the model can transcribe when the audio is clean*; a model that fails FLEURS is not going to recover when the user is in a car.
- **Hugging Face hub distribution** — loadable via `datasets.load_dataset("google/fleurs", "all")` (or a specific locale) with the splits already defined.

You do not need all 102 languages. Pick the five to ten that match your product's deployment footprint and run the harness per language. Report per-language WER; the average across languages hides which ones are broken.

### LibriSpeech

[LibriSpeech](https://www.openslr.org/12) is the long-running English read-speech benchmark from LibriVox audiobooks, with four standard test splits: `train-clean-100`, `train-clean-360`, `dev-clean` / `dev-other`, and `test-clean` / `test-other`. See [Panayotov et al., "Librispeech: an ASR corpus based on public domain audio books"](https://www.danielpovey.com/files/2015_icassp_librispeech.pdf). Properties:

- **Clean-English anchor.** `test-clean` is the one every published ASR number has been compared against for a decade. If your vendor's `test-clean` WER number has changed, so has something upstream of you.
- **Noise contrast built in.** `test-other` is the harder split. The ratio `WER(test-other) / WER(test-clean)` is a crude but useful robustness proxy.
- **Read speech only.** LibriSpeech is read audiobook narration. It does not contain conversational speech, back-channels, or disfluencies. A system that is great on LibriSpeech and bad on your call-centre audio is possible and in fact common.

### TED-LIUM 3

[TED-LIUM 3](https://www.openslr.org/51/) is the conference-English benchmark built from TED talks; see [Hernandez et al., "TED-LIUM 3: twice as much data and corpus repartition for experiments on speaker adaptation"](https://arxiv.org/abs/1805.04699). Properties:

- **Monologue, planned.** TED talks are planned but not fully scripted; they contain disfluencies, speaker-specific prosody, and domain vocabulary.
- **Longer form.** Utterances and sessions are longer than LibriSpeech's. Useful for probing endpointing behaviour (which bleeds over into mod-106).
- **Accent diversity.** Speakers come from many L1 backgrounds.

Together, FLEURS (multilingual, studio), LibriSpeech (clean English, read), and TED-LIUM 3 (planned conference English, longer-form) form a three-point anchor. If your in-domain WER is suspicious, one of these three will usually tell you whether it's the model or your labels.

## The in-domain gold set — the only eval that actually matters

The public corpora catch catastrophic regressions. They do not tell you whether your product works, because none of them look like your product's audio. Your in-domain gold set does, and it is the only eval that goes into the Chapter 4 CI gate as a hard pass/fail.

Author the gold set like a dataset, not like a one-off:

- **Sample from production audio distribution.** Pull a stratified random sample from the last N days of real calls. Stratify by acoustic condition (quiet office / street / in-car / call-centre), by call outcome (successful / abandoned / escalated), by speaker demographic you can legally record (region / language variant), and by call duration bucket. Document the sampling design; without it you can't tell whether your gold set drifted.
- **Size it to the error bar you want.** A gold set of 500 utterances at a measured WER of 10% has a 95% confidence interval roughly ±2.6 percentage points on the proportion of errored words (binomial). If you want to detect a 1-point WER regression you need thousands of utterances. There is no cheap shortcut; small gold sets produce noisy gates. The [Clopper–Pearson interval](https://en.wikipedia.org/wiki/Binomial_proportion_confidence_interval) or a non-parametric bootstrap over utterances are the two common ways to put a CI on a WER number.
- **Double-annotate.** Every utterance gets labelled by at least two annotators independently, then adjudicated. The disagreement data is what drives inter-rater reliability (next section).
- **Freeze on release.** Once a gold-set version is used in a release-assurance report, freeze it. New labels go into the next version. If you re-label v1 after the fact, you have invalidated every historical comparison that used v1.
- **Legal / consent.** Audio has to be in scope for your consent record (mod-108 Chapter 2). A gold set built from recordings the user did not consent to being retained for this purpose is a compliance incident, not a dataset. Confirm with legal before you build it.

## Inter-rater reliability on the gold labels

If you cannot trust your labels you cannot trust the WER you compute against them. Inter-rater reliability (IRR) is the standard way to report "how much do the humans agree before adjudication". For transcription the typical reporting is:

- **Exact-match agreement rate** — what fraction of utterances had identical post-normalisation transcripts from both annotators.
- **WER between annotators** — treat annotator A's transcript as the reference and annotator B's as the hypothesis; compute WER. This is the one everyone intuitively understands. A gold set where annotators disagree at 5% WER is a gold set where you cannot measure a model that is better than 5% WER.
- **Cohen's or Fleiss's kappa for categorical labels.** If you also label categorical fields (speaker ID, acoustic-condition bucket), use kappa, not raw agreement — raw agreement is inflated by class imbalance. See Cohen 1960 and Fleiss 1971; many stats packages implement both (`sklearn.metrics.cohen_kappa_score`, `statsmodels.stats.inter_rater.fleiss_kappa`). Kappa does not apply to free-text transcripts; use the annotator-WER for those.

The conventional rules of thumb published by Landis & Koch in 1977 (`< 0.20` poor, `0.21-0.40` fair, `0.41-0.60` moderate, `0.61-0.80` substantial, `0.81-1.00` almost perfect) are widely cited; they are heuristics, not laws. The right target is that annotator disagreement is small relative to the regression size you want to detect.

Report the IRR number next to the WER number, every time. A WER of 8% against a gold set where annotators disagreed at 6% is a different story from a WER of 8% against a gold set where annotators disagreed at 1%.

## Confidence calibration

Most streaming ASR vendors emit per-word or per-utterance confidence scores. A model is **well-calibrated** if, for the set of words it reported at 90% confidence, roughly 90% are actually correct.

Why you care, in a voice-agent context specifically:

- **Downstream decisions depend on confidence.** If your orchestrator reads a transaction amount off a user's speech, a low-confidence word should trigger a confirmation turn, not an attempted charge. That contract only works if the confidence number means what it claims.
- **Confidence is how you route to human review.** Online eval (Chapter 5) lets you escalate low-confidence turns to a judge-as-LLM or to a human reviewer; miscalibration throws the escalation budget away on confident mistakes.

The standard calibration plot is a **reliability diagram**: bin the model's predictions by confidence (e.g., 10 bins from 0.0 to 1.0), compute the actual accuracy per bin, plot bin-confidence (x) vs. bin-accuracy (y). A perfectly calibrated model follows `y = x`. The summary scalar is the **Expected Calibration Error (ECE)** — the weighted average of `|confidence − accuracy|` across bins (see [Guo et al., "On Calibration of Modern Neural Networks"](https://arxiv.org/abs/1706.04599) for the standard definition; `torchmetrics.CalibrationError` and `netcal` are two reasonable implementations).

For ASR the common convention is to compute calibration on **word-level** predictions — for each word in the hypothesis, is it correct against the reference (after alignment) and what confidence did the model emit — because the orchestrator's decisions are usually per-word. If your vendor only emits an utterance-level confidence, calibrate on utterances; document which you did.

Read the diagram:

- **Flat line below `y = x`** — the model is *underconfident*. Rare in production ASR.
- **Curve above `y = x` at high confidence** — the model is *overconfident* at the top end. This is the common and dangerous failure, because the confident-wrong words are the ones your orchestrator will act on.
- **Noisy bins in the high-confidence regime** — your sample is too small in that bin; refuse to publish ECE until you have per-bin counts above a sensible floor.

A model you cannot calibrate to the confidence contract you need is one you cannot ship into a product that depends on that contract.

## Stratification — the single number is the enemy

A whole-corpus WER of 7% averages two realities: WER of 4% on quiet office audio and WER of 15% on in-car audio. The average is useless; the stratified table is actionable. At minimum, stratify by:

- **Acoustic condition** — quiet office, street / outdoors, in-car, call-centre (narrowband, often G.711 8 kHz), noisy consumer. The label comes from the metadata you captured on the utterance (room type, codec, SNR estimate). If you don't have the metadata, you cannot stratify, and the fix is upstream of this module.
- **Speaker demographic** — gender, age bucket, L1 / accent, region. Record only what you have a legal basis to record (mod-108 Chapter 4); do not infer demographic from audio without explicit consent. Report WER per bucket; publish the IRR *per bucket* too, because annotators are often better on their own L1.
- **Call phase** — greeting / discovery / resolution / closing. Error rates are often very different across phases because the vocabulary and the acoustic context differ.
- **Utterance length bucket** — short (< 3 s), medium (3–10 s), long (> 10 s). Short utterances are disproportionately slot-filling turns where errors propagate worst.
- **Codec / sample-rate bucket** — the Twilio-delivered 8 kHz G.711 leg (mod-107) is a very different acoustic beast from a 48 kHz WebRTC leg.

Report the full table, highlight the worst bucket, and gate on the worst-bucket WER as well as the average. A model that is great on average and terrible on a demographic you serve is not shippable.

## A reference pipeline

A workable implementation that you can drop into a repo:

```python
# Illustrative — verify class names and dataset splits at the dataset's current docs.
import jiwer
from datasets import load_dataset
from whisper.normalizers import EnglishTextNormalizer

NORM = EnglishTextNormalizer()

def wer_cer_mer(refs, hyps):
    refs_n = [NORM(r) for r in refs]
    hyps_n = [NORM(h) for h in hyps]
    return {
        "wer": jiwer.wer(refs_n, hyps_n),
        "cer": jiwer.cer(refs_n, hyps_n),
        "mer": jiwer.mer(refs_n, hyps_n),
        "n_utts": len(refs_n),
    }

def run_eval(asr_fn, dataset_name, config, split, strata_fn=None):
    ds = load_dataset(dataset_name, config, split=split)
    rows = []
    for item in ds:
        hyp = asr_fn(item["audio"]["array"], item["audio"]["sampling_rate"])
        row = {
            "ref": item["transcription"],
            "hyp": hyp,
            "stratum": strata_fn(item) if strata_fn else "all",
        }
        rows.append(row)
    # overall
    overall = wer_cer_mer([r["ref"] for r in rows], [r["hyp"] for r in rows])
    # per stratum
    strata = {}
    for s in set(r["stratum"] for r in rows):
        s_rows = [r for r in rows if r["stratum"] == s]
        strata[s] = wer_cer_mer([r["ref"] for r in s_rows], [r["hyp"] for r in s_rows])
    return {"overall": overall, "strata": strata}
```

Bootstrap a 95% confidence interval by resampling utterances with replacement (e.g., 1000 bootstrap replicates) and reporting the 2.5th / 97.5th percentiles. Without a CI on the number, a 0.5-point difference means nothing.

## What belongs in the release-assurance packet

For every ASR tier that ships (new vendor, new model version, new routing rule), the packet that goes to the ai-evaluation-engineer (Chapter 7) must include:

- WER / CER / MER on `FLEURS` (per language), `LibriSpeech test-clean` / `test-other`, `TED-LIUM 3 test`, and the in-domain gold set, with 95% CIs.
- The normaliser you used, pinned.
- The gold-set version, with the sampling design, the size, and the IRR between annotators.
- A reliability diagram and ECE on your gold set.
- The stratified table with all buckets filled in and the worst bucket called out.
- The comparison to the previous version — delta on each line, with the CI, so "improved" means "improved beyond the noise band".

That packet is the artefact the next three chapters build on. Chapters 2 (TTS) and 3 (conversational) produce their own packets; Chapter 4 wires all of them into a single CI gate.

## Summary

- Measure **WER** as your primary metric for most languages, **CER** as a tie-breaker (primary for CJK), and **MER** as the hallucination-robust companion. Use one normaliser consistently; otherwise your numbers do not compare.
- Anchor against **FLEURS** (multilingual studio), **LibriSpeech** (clean English read), and **TED-LIUM 3** (planned conference English). They catch catastrophic regressions; they do not replace in-domain evidence.
- Build a **versioned in-domain gold set** sampled stratified from production, double-annotated, sized for the error bar you want, and legally in scope.
- Report **inter-rater reliability** (annotator-vs-annotator WER; kappa for categorical fields) next to every WER number. Disagreement bounds the smallest regression you can detect.
- Produce **reliability diagrams and ECE** for the model's confidence signal. Overconfidence in the top bins is the dangerous failure mode in a voice product that routes on confidence.
- **Stratify every number** by acoustic condition, speaker demographic, call phase, utterance length, and codec. Gate on the worst-bucket WER, not the average.

Next chapter: the TTS side — ITU-T P.808 crowdsourced MOS, P.835 signal / background / overall MOS, CMOS, and the automated proxies (UTMOS, DNSMOS) that make this cheap enough to run in CI.
