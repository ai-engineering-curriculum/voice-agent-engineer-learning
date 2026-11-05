# 07. The Boundary with `nlp-engineer`, `model-evaluation-engineer`, and `training-pipeline-engineer`

## Why this chapter

Chapters 1 – 6 are the voice-agent engineer's **working depth** on STT: pick the provider, read the streaming contract, design the handoff, ship domain adaptation, benchmark honestly, handle multilingual. That is the L35 bar.

There is a research-and-training side of the same discipline — deep ASR architecture, pretraining theory, statistical evaluation methodology, continuous training pipelines — that is **not yours**. It belongs to three sibling roles. This chapter names them, draws the boundary, and gives you the deferral contracts.

## The three sibling roles

- **`nlp-engineer` (L30)** — in the sibling [`nlp-engineer-learning`](https://github.com/ai-engineering-curriculum) curriculum. <!-- needs-research: confirm canonical repo URL at author time --> Owns applied-NLP / speech-model-level work: ASR model architecture internals, representation extraction, model-selection depth, domain fine-tuning (as a modelling task, not a Chapter 4 "upload-and-click" product).
- **`model-evaluation-engineer` (L30)** — in the sibling [`model-evaluation-engineer-learning`](https://github.com/ai-engineering-curriculum) curriculum. <!-- needs-research: confirm canonical repo URL at author time --> Owns benchmark methodology: significance testing, per-stratum calibration, sample-size planning, adversarial evaluation, LLM-as-judge rigor.
- **`training-pipeline-engineer` (L35)** — in the sibling [`training-pipeline-engineer-learning`](https://github.com/ai-engineering-curriculum) curriculum. <!-- needs-research: confirm canonical repo URL at author time --> Owns training infrastructure: distributed training, dataset ETL, continuous training cadences, checkpoint / artefact management. In the fine-tune case, this is the peer who stands up the pipeline that runs the fine-tune, versus the `nlp-engineer` / `fine-tuning-engineer` peer who designs the fine-tune itself.

## The rule of thumb

**If the task is "pick a model, wire it into the pipeline, measure it, and reason about its runtime cost," it is yours.** If the task is "evaluate whether the model is architecturally appropriate, redesign the model, design the statistical methodology for the evaluation, or stand up a training pipeline for it," escalate.

## What belongs to `nlp-engineer` — the model internals

You don't need to reimplement the models you select; you do need to be literate enough to talk to the engineer who does. The names and papers below are the floor.

### ASR model architectures

- **Conformer** — [Gulati et al., 2020](https://arxiv.org/abs/2005.08100). Convolution-augmented transformer. The de-facto architecture for streaming ASR; the backbone of NeMo Parakeet, and many commercial systems. Combines a convolutional block (local patterns) with a self-attention block (global patterns) in each layer.
- **Whisper** — [Radford et al., 2022](https://arxiv.org/abs/2212.04356). Encoder-decoder transformer trained on 680k hours of weakly-labelled multilingual audio. Supervised, not self-supervised; the dominant open ASR baseline.
- **RNN-T / Transducer** — [Graves, 2012](https://arxiv.org/abs/1211.3711). The streaming-friendly architecture that most production streaming ASR systems use under the hood. A frame-synchronous decoder that avoids the latency of attention-based decoders.
- **CTC** — [Graves et al., 2006](https://www.cs.toronto.edu/~graves/icml_2006.pdf). Connectionist Temporal Classification, the loss function that lets you train frame-level outputs without explicit alignment. The decoder head on many fine-tuned wav2vec 2.0 ASR systems.
- **wav2vec 2.0** — [Baevski et al., 2020](https://arxiv.org/abs/2006.11477). The self-supervised pretraining recipe behind a large family of downstream ASR models. Learn by name.
- **HuBERT** — [Hsu et al., 2021](https://arxiv.org/abs/2106.07447). The other dominant SSL recipe for speech. Masked-prediction over quantised features.

What an NLP engineer does with these: fine-tune a Conformer on 100 hours of medical audio; probe a wav2vec 2.0 for phoneme representations; design an RNN-T with a custom predictor for a specific latency target. What you do: pick which of their outputs to use and integrate it.

### Pretraining and transfer

Three concepts you should be literate in:

- **Pretraining-then-fine-tuning.** The wav2vec 2.0 / HuBERT / WavLM paradigm: SSL on unlabelled audio produces a representation; a small supervised head produces an ASR. `nlp-engineer` owns both stages for a serious custom build.
- **Multitask training.** Whisper's recipe: a single encoder-decoder trained on ASR + speech translation + language ID. Gives you one artefact with multiple capabilities, at a model-size cost.
- **Low-resource transfer.** MMS and SeamlessM4T demonstrate that joint training across hundreds of languages improves low-resource accuracy. The theory and recipe design are `nlp-engineer` territory.

### Model selection depth

If you're stuck between "Whisper large-v3-turbo, fine-tuned" and "a Conformer trained from scratch on our domain" — and the choice is strategic, not operational — the architectural call is `nlp-engineer`'s. You bring the data contract (Chapter 4), the acceptance criteria, and the production constraints; they bring the model-architecture judgment.

## What belongs to `model-evaluation-engineer` — the evaluation methodology

Chapter 5 gave you the engineer's working depth on WER benchmarking. The deeper methodology belongs here.

### Statistical significance between models

You can report WER with bootstrap CIs — that is your floor. The next question — "is the 0.3-point WER improvement statistically meaningful, controlling for the per-utterance correlation structure?" — needs:

- **Matched-pair tests** (McNemar on utterance-level correct/wrong; signed-rank / sign tests on per-utterance WER).
- **Clustered bootstrap** respecting the per-speaker correlation (utterances from the same speaker are not independent).
- **Multiple-comparison correction** when you're A/B-testing many provider combinations.

These are `model-evaluation-engineer` tools, not voice-engineer tools. You should know they exist and ask for them when the decision is material.

### Calibration at depth

Chapter 3 gave you ECE and Platt / isotonic. The deeper territory:

- **Multiclass calibration** — temperature scaling across many categories.
- **Class-conditional calibration** — a model miscalibrated differently on numbers vs. names needs per-class corrections.
- **Group calibration** — does the confidence calibration hold across accent clusters, languages, noise levels? If not, the handoff thresholds you set in Chapter 3 have different meanings per group, which is itself a quality bug.

Don't re-derive these. Hand the data across.

### LLM-as-judge in the ASR eval loop

Modern evaluation pipelines use an LLM as a judge: "given the reference transcript and the ASR hypothesis, rate the semantic equivalence." Useful for domains where WER is too noisy a proxy (medical dictation, where paraphrase is acceptable; conversational Q&A, where a hesitation inserted into the transcript matters less than a wrong product name).

The design of a reliable judge — prompt engineering, calibration against humans, bias detection — is `model-evaluation-engineer` depth. You use the artefact; they build it.

### Sample-size planning for in-domain gold sets

Chapter 5 said "at least 2–5 hours, 1 000+ utterances." That's a heuristic. The real answer depends on:

- The variance of per-utterance WER in your domain.
- The effect size you need to detect (1 WER point? 0.3?).
- The stratification you need to report (if you want per-accent power, you need samples per accent, not pooled).

Ask `model-evaluation-engineer` to run the power analysis when you need defensible numbers on a small gold set.

## What belongs to `training-pipeline-engineer` — the fine-tune infrastructure

Chapter 4 said: if the WER gap is too big for keyterm lists, escalate to a fine-tune. The peer who stands up the training infrastructure is the `training-pipeline-engineer`.

### What that includes

- **Dataset ETL.** Taking your raw contact-centre recordings + transcripts, segmenting them, aligning them, filtering low-quality samples, producing a reproducible training dataset.
- **Training run management.** GPU scheduling, checkpointing, hyperparameter sweeps, experiment tracking (Weights & Biases, MLflow), reproducibility.
- **Continuous training.** The cadenced retraining that keeps a fine-tuned model fresh as your domain drifts. The observability loop you own (Module 112) is the trigger; the training infrastructure they own is the responder.
- **Model artefact lifecycle.** Versioning, canary deployment, rollback, serving-infrastructure handoff.

### What stays yours after the fine-tune

Even once the fine-tune pipeline is standing:

- **The training-data spec.** Chapter 4's data contract — hours, provenance, stratification, acceptance criteria — is *yours* to write. The training pipeline consumes that spec.
- **Integration and metric ownership.** You wire the fine-tuned model into your voice pipeline, you monitor its WER and latency, you decide when to roll back or re-tune.
- **Observability signals.** Module 112's WER-drift alerts fire the retraining cadence.
- **Keyterm / phrase-list configuration** on top of the fine-tuned model — that remains a pipeline-engineering decision.

## What *doesn't* belong to anyone on the ladder below you

A clarification worth making: the handoff contracts above are not "hand this off because it's beneath you." They are "hand this off because this role has the depth, and your job is to compose their output." The quality of the fine-tune, the honesty of the eval, the stability of the training pipeline — all depend on a tight handoff artefact. **Your job is to make the handoff legible.**

Three habits that make you a peer, not a customer, to these roles:

1. **Write the acceptance criteria before the work starts.** "95 % of drug-name-stratum utterances have WER < 10 % on the frozen test set vX." Not "better than today."
2. **Provide a reproducer.** A specific audio clip + current transcript + target transcript, with a `jiwer` invocation that shows the current error. The receiving engineer should be able to run one command and see the problem.
3. **Own the integration side.** If their improved model needs a new field in the streaming contract, you add it. If their fine-tune needs a different confidence normalisation, you wire it. They should not be reaching into your pipeline; you should be reaching into theirs with structured hand-offs.

## The decision tree

For any ASR-related question that lands on your desk:

1. **Is it a pipeline-integration or configuration question?** (Provider swap, streaming contract, keyterm list, handoff threshold, observability.) → You. Chapters 1 – 4, 6.
2. **Is it a measurement question with standard tools?** (WER on a gold set, per-stratum breakdown, A/B latency measurement.) → You. Chapter 5.
3. **Is it a measurement question requiring methodology depth?** (Statistical significance of a WER delta, calibration across groups, LLM-as-judge design.) → `model-evaluation-engineer`.
4. **Is it an "existing-model is close but not quite" question?** (Tune a keyterm list, run a provider custom-speech fine-tune.) → You, or co-owned with `fine-tuning-engineer` / `nlp-engineer` depending on depth.
5. **Is it a "new model needed" question?** (Train an architecturally different ASR, redesign a loss, support a non-covered language.) → `nlp-engineer` + `training-pipeline-engineer`.
6. **Is it a "we need a training pipeline that runs continuously" question?** → `training-pipeline-engineer`.

## Reading list to stay literate

Six papers at the top of the ASR reading list. Read the abstract, the architecture figure, and the results table; come back to ablations if the model ships in your product.

- Conformer — [Gulati et al., 2020](https://arxiv.org/abs/2005.08100)
- Whisper — [Radford et al., 2022](https://arxiv.org/abs/2212.04356)
- wav2vec 2.0 — [Baevski et al., 2020](https://arxiv.org/abs/2006.11477)
- HuBERT — [Hsu et al., 2021](https://arxiv.org/abs/2106.07447)
- MMS — [Pratap et al., 2023](https://arxiv.org/abs/2305.13516)
- FLEURS — [Conneau et al., 2022](https://arxiv.org/abs/2205.12446)

Plus one on evaluation that you should actually read in full — the Koenecke et al. paper from Chapter 6 ([2020](https://www.pnas.org/doi/10.1073/pnas.1915768117)) — because it demonstrates the stakes of stratified evaluation in a way the WER-with-bootstrap language never quite does.

## Summary

- Deep ASR architecture internals (Conformer, wav2vec 2.0 pretraining) and model-selection-at-depth belong to `nlp-engineer`.
- Statistical evaluation methodology — hypothesis testing between models, per-group calibration, judge design — belongs to `model-evaluation-engineer`.
- Training-infrastructure for ASR fine-tunes — ETL, distributed training, continuous training — belongs to `training-pipeline-engineer`.
- Your role is to compose their outputs with the pipeline-engineering judgment Chapters 1 – 6 built. The handoff contract — problem, corpus, acceptance criteria, integration — is yours to write, and its quality bounds the output.
- Be a peer by making the handoff legible: acceptance criteria before work, reproducers with evidence, and integration-side ownership once the model lands.

This closes the content block for `mod-103`. Proceed to the exercises (provider scorecard, streaming-contract walkthrough, confidence handoff, keyterm shipping, WER benchmark, multilingual ladder) and then on to [`mod-104-tts-production-engineering`](../mod-104-tts-production-engineering/README.md).
