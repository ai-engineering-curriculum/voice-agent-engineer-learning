# exercise-01: Build the ASR Eval Harness — FLEURS + LibriSpeech + TED-LIUM 3 + In-Domain Gold

**Estimated effort:** 3 hours · **Output:** a runnable ASR eval harness, a versioned in-domain gold set skeleton, and a release-assurance ASR packet for the current STT tier

## Objective

Build a reproducible ASR eval harness that produces WER / CER / MER against the three public anchor corpora and an in-domain gold set, with inter-rater reliability on the gold labels, a confidence-calibration plot, and stratified results by acoustic condition and speaker demographic. The harness has to emit a machine-readable release-assurance ASR packet that the Chapter 4 CI gate (exercise-04) will later consume.

This is the first of three harness-build exercises; keep the output discoverable to exercises 04, 05, 06.

## Prerequisites

- [Chapter 1 — The ASR Eval Harness](../01-asr-eval-harness.md).
- mod-103 (STT production engineering) for the current STT vendor / model in use.
- A working ASR integration from mod-103 or mod-111 that you can call synchronously or stream audio into.
- Python 3.11+. Recommended libraries: [`jiwer`](https://github.com/jitsi/jiwer), [`datasets`](https://huggingface.co/docs/datasets), [`torchmetrics`](https://lightning.ai/docs/torchmetrics/), [`openai-whisper`](https://github.com/openai/whisper) for the English text normaliser, `scipy` / `numpy` for bootstrap CIs, `matplotlib` for the reliability diagram.
- Access to a small sample (20–200 utterances) of in-domain production audio with written consent to use for eval purposes. If you do not have that sample at your organisation, construct a synthetic in-domain approximation and note the limitation in the output packet.

## Steps

1. **Pin a normaliser.** Vendor the Whisper `EnglishTextNormalizer` (or the multilingual variants for non-English) into your repo at a specific version. Document the normaliser + version in the eval config. Every WER number downstream is computed with this normaliser applied identically to reference and hypothesis.

2. **Wire the three public anchors.** Load FLEURS (five to ten locales that match your deployment footprint), LibriSpeech (`test-clean` and `test-other`), and TED-LIUM 3 (`test`) via `datasets.load_dataset`. For each, run your STT on the audio, apply the normaliser, compute WER / CER / MER with `jiwer`.

3. **Build the in-domain gold set skeleton.** Define the sampling design (strata = acoustic condition × speaker demographic × call phase × utterance-length bucket × codec) in a config file. Capture the sample. Double-annotate at least a pilot slice; record each annotator's transcript and the adjudicated final label. Version the gold set (`support-en-v1`); freeze it.

4. **Compute inter-rater reliability on the gold pilot.** For each utterance with two annotations: exact-match agreement and annotator-A-vs-annotator-B WER. Report the distribution of annotator-WER. For any categorical annotation fields (acoustic bucket, speaker role), report Cohen's or Fleiss's kappa. Do *not* publish the gold set as release-ready if the IRR is worse than the smallest regression you want to detect.

5. **Stratify and bootstrap.** On the in-domain set, compute WER / CER / MER per stratum. Bootstrap 95% CIs (resample utterances with replacement, 1000 replicates). Produce a table that calls out the worst bucket.

6. **Confidence-calibration plot.** If your STT emits per-word confidence, build the reliability diagram (10 bins, bin-confidence vs. bin-accuracy after word alignment). If only utterance-level confidence is available, calibrate at the utterance level. Compute Expected Calibration Error (ECE). Save the plot to the packet.

7. **Emit the release-assurance ASR packet.** A YAML or JSON file with: normaliser version, public-anchor WERs with CIs, gold-set version and sampling design, overall + per-stratum in-domain WER / CER / MER with CIs, IRR, reliability diagram URI + ECE, worst-bucket call-out. This is the artefact exercise-04's CI gate will consume.

## Starter guidance

- **Normaliser consistency is the #1 trap.** If you compute LibriSpeech WER with one normaliser and your in-domain WER with another, you are comparing apples and oranges. Lock it in step 1 and never vary it inside a release.
- **FLEURS per-language is the right unit.** An average across five languages hides which one is broken. Report per-language WER separately.
- **Do not label the gold set yourself.** You are both the author of the harness and a trained listener; your labels are biased toward the easy cases. Use at least two independent annotators who are not on the agent build team.
- **Legal clearance for the gold set is a hard precondition.** If your organisation does not yet have consent-scope for retaining customer audio for eval purposes, do that paperwork before the exercise, not during. Document any gap and defer to a synthetic set until cleared.
- **Keep the harness a library, not a script.** Everyone else's exercise in this module calls into yours. A `pip install` able harness with a single `run_eval(system, dataset)` entry point beats a 400-line script every time.
- **ECE needs enough samples per bin.** If a bin has fewer than ~30 samples, do not report its calibration — note it and keep collecting.

## Acceptance criteria

- [ ] A versioned, runnable eval harness that computes WER / CER / MER with a documented normaliser and `jiwer` (or equivalent).
- [ ] FLEURS (per-language, ≥5 locales), LibriSpeech (`test-clean` + `test-other`), TED-LIUM 3 (`test`) numbers in the packet, each with a bootstrap 95% CI.
- [ ] A versioned in-domain gold set (schema + sampling design + at least a pilot slice labelled), with the frozen version string in the packet.
- [ ] IRR reported — annotator-vs-annotator WER distribution and kappa for categorical fields.
- [ ] Per-stratum in-domain WER / CER / MER table with worst-bucket call-out, each with CIs.
- [ ] Reliability diagram (plot) and ECE scalar in the packet.
- [ ] A machine-readable `asr_packet.yaml` / `asr_packet.json` following the Chapter 7 schema shape; discoverable by exercise-04.

## Stretch goals

- **Expand to a second language.** If your product serves a second locale, build the second-language gold set and re-run the harness. You will discover the normaliser story is messier in non-English.
- **Automate label collection.** Wire the gold set to a label-review tool (Label Studio, Argilla, or a cloud annotation vendor) so new samples flow through the same pipeline weekly.
- **Compare two STT vendors.** Run the same harness against a second STT and emit a side-by-side packet. Use bootstrap CIs on the delta.
- **Add a hallucination-specific metric.** For every utterance, record whether the hypothesis has words not present in the reference (Whisper's classic failure mode on silence); report rate and distribution.

## Keep it

The packet from this exercise is the input to exercise-04's CI gate. The harness itself is reused by exercise-05 (online eval, running the same WER math against sampled production audio) and exercise-06 (instrumenting the per-turn WER attribute on OTel spans).
