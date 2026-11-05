# 02. The TTS Eval Harness — P.808, P.835, CMOS, UTMOS, DNSMOS

## Why this chapter

ASR eval (Chapter 1) has a ground truth: the reference transcript. Edit distance is defined; the hard part is sampling, labelling, and stratification. TTS eval has no comparable ground truth. "Did this synthetic voice sound good?" is answerable only by a human — and once you commit to that, you commit to the methodology that makes human judgements comparable across runs, vendors, and model versions.

This chapter maps the standards you actually need. The **ITU-T P.808** recommendation defines the crowdsourced subjective-test protocol that is the modern analogue of the older P.800. **P.835** is the subjective-test method that decomposes quality into signal / background / overall, which matters when a noise-suppressor or codec sits in the chain. **Comparative MOS (CMOS)** with A/B anchors is how you compare two systems head-to-head and get a confidence interval on the preference. **UTMOS** and **DNSMOS** are the automated predictors that make it cheap enough to run every PR and every minute of production — with the caveat that they predict human ratings, they are not the ratings.

The right pattern is: subjective crowdsourced panels at release; automated proxies in CI and in production monitoring; and a documented correlation study that justifies the proxy you use in CI against the subjective panel you ran at release.

## The subjective anchor: ITU-T P.808 crowdsourced MOS

[ITU-T Recommendation P.808](https://www.itu.int/rec/T-REC-P.808) defines a subjective evaluation of speech quality run over a **crowdsourcing platform** (Prolific, Amazon Mechanical Turk, Appen, and similar). Treat P.808 as the modern methodology for Absolute Category Rating (ACR) MOS; P.800 is the older recommendation written for controlled lab conditions, and almost nothing in the voice-agent world is tested in a lab.

What P.808 actually prescribes (consult the recommendation for the full procedure; what follows is the high-level shape, not a substitute for the text):

- **5-point ACR scale.** Excellent (5), Good (4), Fair (3), Poor (2), Bad (1). The raw score aggregated across raters is the **Mean Opinion Score**.
- **Listening environment screening.** Raters perform headphone-detection tests, hearing-threshold tests, and comprehension tests before their ratings count. A P.808 run without rater qualification is not a P.808 run.
- **Attention-check clips** embedded in the batch. Raters who fail attention checks are dropped.
- **Reference and anchor clips.** Known-quality clips provide calibration points so you can post-normalise across rater populations.
- **Minimum rater count per clip.** Spec-prescribed (typically tens of ratings per clip; check the current recommendation text) so the per-clip MOS has a usable CI.
- **Payment at or above local minimum.** This is an ethics and a data-quality concern; underpaid raters produce noisy data.

Microsoft maintains an open-source implementation of the P.808 methodology in [`microsoft/P.808`](https://github.com/microsoft/P.808) that runs on Amazon Mechanical Turk and includes the rater-qualification flows, the ACR template, and the data-cleaning scripts. Use it as a reference implementation; do not re-invent the qualification pipeline.

When you report a MOS, report:

- **Mean and 95% CI** across raters per clip, and across clips per system.
- **Rater n per clip and total n.** A MOS from 10 raters has a very different CI than one from 50.
- **Stratification** by clip type (same slices as ASR eval — acoustic condition, voice persona, call phase).
- **Rater pool provenance** — Prolific vs. MTurk vs. an internal panel. These produce measurably different distributions.

A single aggregate MOS without those four accompaniments is not reportable.

## P.835 — decomposing quality into signal, background, overall

[ITU-T P.835](https://www.itu.int/rec/T-REC-P.835) is the subjective-test method for *noise-suppressed speech*. For a voice agent, this matters whenever the audio path includes a noise suppressor, a bandwidth extender, a codec with aggressive compression, or any processing that trades signal quality for background quality.

P.835 asks raters three questions per clip, on the same 5-point scale:

- **SIG** — signal-only quality. "How was the speech signal itself, ignoring the background?"
- **BAK** — background-only quality. "How intrusive was the background noise, ignoring the speech?" (Higher is better; a quiet background scores high.)
- **OVRL** — overall quality.

Why you cannot replace P.835 with a plain P.808 run when there's processing in the chain: a noise suppressor that produces artefact-ridden speech on a quiet background might score similarly on P.808 to a noisy-but-natural recording. P.835 separates those two systems because the SIG score drops on the first and the BAK score drops on the second. The subjective judgement you actually need — "am I trading signal for background, and how much?" — is only legible in the decomposition.

The same Microsoft [`P.808` repo](https://github.com/microsoft/P.808) includes a P.835 template. The [DNS Challenge](https://github.com/microsoft/DNS-Challenge) series (Interspeech 2020, 2021, 2022; ICASSP 2022, 2023) uses P.835 as its official metric, so if you are evaluating noise suppressors against published numbers, P.835 is the number they're reporting.

Report SIG, BAK, OVRL with CIs, same pattern as P.808 MOS. If you only report OVRL, you have discarded the information that justified running P.835 in the first place.

## CMOS — pairwise preference with A/B anchors

Absolute MOS is good for "is this release's quality within our acceptance band?". It is noisy for "is system A better than system B?" — individual raters anchor on their own absolute scale, and small differences wash out. [ITU-T P.800](https://www.itu.int/rec/T-REC-P.800) describes Comparison Category Rating; **Comparative MOS (CMOS)** is the term widely used in TTS papers, and the practical protocol is:

- **Present clips in A/B pairs** of the same utterance synthesised by two systems (or one real, one synthesised).
- **7-point comparison scale**, e.g., `−3` (A much worse than B) through `0` (equal) through `+3` (A much better than B).
- **Randomise A/B order per rater**. Half the raters get "A first", half "B first". Otherwise order bias dominates.
- **Anchor clips.** Include reference pairs with a known preference (e.g., a clean recording vs. a degraded one) so you can drop raters whose anchor scores are inverted.
- **Report mean CMOS and a 95% CI** across rater-pair observations. A CMOS of +0.1 with a CI that straddles zero is a non-result.

CMOS is the metric that will decide "do we promote the new voice model to default?" in practice, because it has smaller variance for a given rater count than absolute MOS. Google, Microsoft, Meta, and Amazon TTS research papers typically report CMOS against the best prior system. Treat that as the convention you align with.

Report scale, anchor set, rater pool, and n with every CMOS number. Without those, the CI is undefined.

## UTMOS — the CI-friendly automated predictor

Running P.808 on every pull request would cost more than the model. **UTMOS** ([Saeki et al., "UTMOS: UTokyo-SaruLab System for VoiceMOS Challenge 2022"](https://arxiv.org/abs/2204.02152)) is a trained neural predictor of MOS. It won the [VoiceMOS Challenge 2022](https://voicemos-challenge-2022.github.io/), it is published open-source at [`sarulab-speech/UTMOS22`](https://github.com/sarulab-speech/UTMOS22), and it is the most widely-used automatic MOS predictor in current TTS papers.

What UTMOS is doing: a model trained on human MOS ratings (primarily from the VoiceMOS Challenge data, itself derived from the Blizzard Challenge and the VCC speech samples) to predict the MOS a human panel would give. It is a *regression* onto human labels, not a quality metric from first principles.

Two consequences:

- **It is reliable where it was trained.** For English synthetic speech similar in distribution to the VoiceMOS training set, UTMOS correlates well with P.808 MOS. For languages or acoustic conditions it has not seen (narrowband telephony, emotional speech, non-English), the correlation can degrade significantly; you cannot assume UTMOS is a drop-in proxy outside its training distribution.
- **It is a comparison metric, not an absolute one.** The absolute UTMOS number is calibrated against the training-set MOS scale. If your rater pool scores systematically higher or lower than the VoiceMOS pool, the absolute UTMOS value will be biased. Differences between two systems on the same distribution are much more stable than absolute scores.

The right way to use UTMOS:

1. **Run it in CI** on a fixed eval set of synthesised utterances. A regression in UTMOS on the eval set blocks the PR.
2. **Validate quarterly** against a P.808 panel on the same eval set. Compute Pearson / Spearman correlation between UTMOS and MOS at the clip level. If the correlation drops below a documented threshold (typical published numbers are in the 0.7–0.9 range for in-distribution data; validate on your own distribution), UTMOS is no longer trustworthy and the quarterly panel has to carry more weight until the proxy is re-trained or replaced.
3. **Never ship a model on UTMOS alone.** Automated predictors have failure modes that the training distribution hid; a release goes on the subjective panel plus the proxy, not the proxy alone.

Alternatives to UTMOS that you may see in release-assurance packets: **MOSNet** ([Lo et al., 2019](https://arxiv.org/abs/1904.08352)), **NISQA** ([Mittag et al., Interspeech 2021, `gabrielmittag/NISQA`](https://github.com/gabrielmittag/NISQA)), **SQUIM** from Meta ([Kumar et al., ICASSP 2023, included in `torchaudio.prototype.pipelines.SquimObjective`](https://pytorch.org/audio/main/prototype.pipelines.html#squim-objective)). Pick one and document the choice; do not swap predictors without re-running the correlation study.

## DNSMOS — the production-monitoring predictor

[DNSMOS](https://arxiv.org/abs/2010.15258) (and the updated [DNSMOS P.835](https://arxiv.org/abs/2110.01763)) is the DNS-Challenge team's automatic predictor trained to approximate P.835 scores. The current public release predicts SIG / BAK / OVRL as a 3-head model.

Why it's the right tool for *production monitoring* in particular:

- **It is fast and has no human cost.** You can run it on a sample of every call in production.
- **It is P.835-shaped.** The three heads let you monitor degraded-signal and degraded-background failure modes separately. A noise-suppressor that silently falls over in production shows up as a BAK regression; a codec that crushes the voice shows up as a SIG regression.
- **It is a known reference for voice-quality monitoring** across the industry — the DNS Challenge series used DNSMOS as its reporting metric for years.

The caveats are the UTMOS caveats, raised by one level because production distribution is wider than any eval set: DNSMOS was trained on a specific distribution of noise and processing types; your production distribution may differ. Validate against a P.835 panel at launch and periodically; use trend changes (not absolute values) as the primary signal.

Pragmatically:

- Sample ~1% of production calls (sampling rate depends on your volume and privacy posture — mod-108 Chapter 4), run DNSMOS on short windows, emit SIG / BAK / OVRL as metrics on the same observability plane you set up in Chapter 6.
- Alert on bucketed-trend drops, not instantaneous drops. Hour-over-hour and day-over-day are more informative than any single call.
- When an alert fires, you look at the metric *and* at the WER trend from Chapter 1 *and* at the latency trend from Chapter 3 — a BAK drop that coincides with a latency spike is probably a transport-quality issue (packet loss, codec switch), not a TTS regression.

## Intelligibility — the crosscheck against ASR

There is one objective number you can get out of synthetic TTS output cheaply: feed it back into an independently-trained ASR and compute the WER against the input text. If the ASR cannot transcribe your synthetic voice, humans probably can't either. This **ASR-based intelligibility check** is a sanity net, not a quality metric:

- It fails loudly on broken phonemes, silence gaps, or language-identity flips — all failure modes that are catastrophic but easy to miss on a MOS panel that averages them out.
- It passes clips that sound robotic but intelligible — it is not a substitute for MOS.
- Use a different ASR than the one in your production pipeline; otherwise a shared failure mode (e.g., a specific accent model) hides itself.

## Correlation discipline

The release-assurance packet has to defend one claim: "the automated proxy we used in CI tracks the subjective panel we ran at release, on the distribution we care about." That defence is a **correlation study**:

- Run the subjective panel (P.808 or P.835) on N clips.
- Run the proxy (UTMOS / DNSMOS / NISQA / SQUIM) on the same N clips.
- Report Pearson correlation (linear), Spearman correlation (rank), and a scatter plot. Pearson is sensitive to the absolute calibration; Spearman is sensitive only to rank.
- Report per-stratum correlation. The overall correlation can hide the fact that the proxy fails specifically on the stratum your product cares about.

Publish the correlation with every release packet; re-run it on a cadence (quarterly is reasonable for a stable product, after every model change otherwise).

## A reference pipeline

```python
# Illustrative — verify current APIs before using.
import numpy as np
from utmos import Score as UTMOS  # pseudo; use the current loader for your fork
from dnsmos import DNSMOS         # pseudo; the public release ships a model + preprocessor

utmos = UTMOS()      # usually loaded from a checkpoint; see upstream docs
dnsmos = DNSMOS()

def score_utterance(wav, sr):
    return {
        "utmos": float(utmos(wav, sr)),
        "dnsmos_sig": float(dnsmos(wav, sr)["sig"]),
        "dnsmos_bak": float(dnsmos(wav, sr)["bak"]),
        "dnsmos_ovrl": float(dnsmos(wav, sr)["ovrl"]),
    }

def eval_system(system_fn, prompts, strata_fn=None):
    rows = []
    for p in prompts:
        wav, sr = system_fn(p["text"])
        s = score_utterance(wav, sr)
        s["stratum"] = strata_fn(p) if strata_fn else "all"
        s["prompt_id"] = p["id"]
        rows.append(s)
    return rows

def aggregate(rows, key):
    xs = np.array([r[key] for r in rows])
    boot = np.array([
        np.mean(np.random.choice(xs, size=len(xs), replace=True))
        for _ in range(1000)
    ])
    return {
        "mean": float(xs.mean()),
        "ci_lo": float(np.percentile(boot, 2.5)),
        "ci_hi": float(np.percentile(boot, 97.5)),
        "n": len(xs),
    }
```

Pair this with: a quarterly P.808 or P.835 panel on a fixed eval set using the Microsoft `P.808` repo as the implementation, a CMOS A/B panel for any system-vs-system promotion, and the correlation study that connects the three.

## Multimodal note — avatar / lip-sync eval

If your product has an animated avatar (mod-110), add two layers on top of the audio eval:

- **Audio-video sync**, measured in ms of offset. The ITU-R BT.1359-1 recommendation defines detectability thresholds (roughly `+45 ms` audio-ahead to `−125 ms` audio-behind is undetected by most viewers; verify current value in the current recommendation). Report the distribution, not just the mean.
- **Lip-sync quality** via a lip-sync discriminator score (the [SyncNet](https://www.robots.ox.ac.uk/~vgg/software/lipsync/) family is the reference).

Avatar quality is subjective in the same way TTS is; run a short P.808-style subjective panel per release but with the avatar shown, not audio-only.

## Summary

- **P.808** is the modern crowdsourced-MOS recommendation; use the Microsoft `P.808` reference implementation, qualify raters, report MOS with CI, rater pool, stratification, and n.
- **P.835** decomposes quality into **SIG / BAK / OVRL** when there is noise suppression or codec processing in the chain. Report all three.
- **CMOS** with A/B anchors is the right protocol for system-vs-system promotion decisions; it has smaller variance than absolute MOS for a given rater count.
- **UTMOS** is the CI-friendly MOS predictor; **DNSMOS** is the production-monitoring predictor. Both are regressions onto human labels and have in-distribution failure modes.
- Run a **correlation study** between your automated proxy and your subjective panel on every release; gate the proxy on its correlation holding up, not just on its own number.
- **Never ship on the automated proxy alone.** The subjective panel is the anchor; the proxy is the daily-flight instrument.

Next chapter: the conversational eval — turn-taking correctness against ground-truth boundaries, barge-in reaction time, time-to-first-audio, mouth-to-mouth p95 enforcement, and interrupt-recovery correctness, all tied back to the mod-109 budget.
