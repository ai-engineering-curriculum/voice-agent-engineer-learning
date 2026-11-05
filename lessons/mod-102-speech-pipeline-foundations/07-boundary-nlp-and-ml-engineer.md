# 07. The Boundary with `nlp-engineer` and `ml-engineer`

## Why this chapter

Chapters 1 – 6 are the voice-agent engineer's **working depth** on the pipeline: enough to read a spectrogram, pick a codec, tune a resampler, and reason about a token rate. That's the L35 bar.

There is a research-and-training side of the same discipline — deep DSP, self-supervised speech representation learning, codec training, ASR / TTS model training — that is **not yours**. It belongs to:

- **`nlp-engineer` (L30)** — the AI-engineering-curriculum role that owns applied NLP / speech-model-level work: model selection, fine-tuning for a domain, representation extraction, evaluation of model quality on a corpus.
- **`ml-engineer` (L20)** — the broader ML foundation role that owns training-pipeline infrastructure, dataset curation, loss design, distributed training, hyperparameter sweeps.

This chapter does two jobs:

1. **Names the research-side concepts** (wav2vec 2.0, HuBERT, MMS, the DSP stack behind Mel spectrograms, codec training objectives) at the level you need them to *read a paper* and *collaborate with an NLP engineer* — but not to reimplement.
2. **Draws the boundary** — when you handle the question yourself, when you escalate, what to hand across.

Treat this chapter as the deferral contract for the deep-speech-ML side of your role.

## What belongs to you — the pipeline-engineering working depth

Everything in Chapters 1 – 6: sample rates, bit depth, feature extraction *using* standard libraries, buffering, codec selection and configuration, AEC/NS deployment, neural-codec integration as a consumer. You read spectrograms; you don't design the Mel filter bank. You call `MelSpectrogram`; you don't derive it. You integrate EnCodec; you don't train one.

Rule of thumb: **if the task is "pick a library, wire it into the pipeline, and reason about its runtime cost," it is yours.** If the task is "evaluate whether the library's model is appropriate for this domain, and if not, train a better one," escalate.

## What belongs to `nlp-engineer` (L30)

The applied-NLP engineer owns **the speech model as a research object** — choosing it, fine-tuning it, evaluating it, swapping it. Three recurring artefacts:

### Self-supervised speech representations

These are the pretext-trained models that modern ASR / speaker-ID / speech-LM systems are built on. You should know them by name, read papers that cite them, and know when they're relevant — but you don't reimplement them.

- **wav2vec 2.0** ([Baevski et al., 2020](https://arxiv.org/abs/2006.11477)). Convolutional feature encoder + transformer, trained with a contrastive loss over quantised targets. Produces a 768- or 1024-dim representation per 20 ms frame. The base of a large family of downstream ASR models (fine-tune a CTC head on labelled data → an ASR).
- **HuBERT** ([Hsu et al., 2021](https://arxiv.org/abs/2106.07447)). Hidden-unit BERT for speech: clusters MFCC features as pseudo-labels, trains a masked-prediction objective, iterates. Similar output shape to wav2vec 2.0 but typically better downstream performance.
- **WavLM** ([Chen et al., 2021](https://arxiv.org/abs/2110.13900)). HuBERT variant trained with additional denoising and multi-speaker scenarios. Used as the semantic-feature source distilled into Mimi's first codebook (Chapter 6).
- **MMS (Massively Multilingual Speech)** ([Pratap et al., 2023](https://arxiv.org/abs/2305.13516)). Meta's wav2vec-2-based models covering 1 100+ languages for ASR and 1 400+ for TTS. The go-to open-weights model for low-resource languages.
- **Whisper** ([Radford et al., 2022](https://arxiv.org/abs/2212.04356)). Supervised encoder-decoder trained on 680k hours of multilingual audio; not a self-supervised model but the dominant open ASR baseline.

What an NLP engineer does with these that you do not: fine-tune a HuBERT-base on 30 hours of medical-dictation audio to beat the off-the-shelf ASR on acronyms; probe a wav2vec 2.0 for phoneme representations; cluster MMS embeddings to build a speaker-ID retriever.

What you do: pick a provider or model that is *good enough* off the shelf, and if it isn't, write up the WER/CER gap, the domain corpus, and the acceptance threshold, and hand it to the NLP engineer.

### ASR / TTS model fine-tuning

If a provider ASR is bad at your domain:

1. **First mitigation — provider-level features.** Custom vocabulary, keyword boost, prompt biasing. You own this; it's pipeline configuration.
2. **Second mitigation — post-hoc correction with an LLM pass.** Still yours.
3. **Third option — fine-tune or train.** Escalate to `nlp-engineer` (if the chosen base model is a fine-tunable open-weights one) or to `training-pipeline-engineer` (per module 101 chapter 4) if the scope is larger.

The handoff artefacts (same shape as module 101 chapter 4's deferral contract):

- Domain corpus (recordings + transcripts) with licensing and privacy status cleared.
- Current error analysis: top 20 confusions, substitution/deletion/insertion rates, examples.
- Target metric: "95 % WER on this held-out test set" or similar.
- Current pipeline integration point so the trained model can be swapped in.

### Judge / evaluation at the audio level

Evaluating "did the TTS actually sound right" or "is this transcript accurate for this domain" at scale uses techniques (reference-based MOS prediction, UTMOS, DNSMOS, listener preference studies) that overlap with eval-engineer and NLP-engineer craft. You own voice-specific test-set design (Chapter 2 of Module 101); they own methodology depth.

## What belongs to `ml-engineer` (L20)

The ML-engineer role in this ecosystem is the broader foundation of training-infrastructure and model-plumbing work. In speech specifically:

### Training a codec, a vocoder, or an ASR model from scratch

If the business genuinely needs a proprietary ASR model or a from-scratch neural codec — say for an under-served language, an extreme-accuracy medical dictation system, or a differentiating TTS voice — that is a multi-month training project:

- Dataset collection, cleaning, speaker balancing, VAD-based segmentation.
- Loss functions: CTC, RNN-T, attention-encoder-decoder for ASR; adversarial + reconstruction + commitment losses for neural codecs.
- Distributed training (tens to hundreds of GPUs, often).
- Model compression: distillation, pruning, quantisation for serving.

You're almost never the person doing this. If you *are*, you're operating outside the L35 voice-agent-engineer scope, probably in the role of a tech lead on a speech-models team.

### Core DSP math

The stuff behind the libraries:

- STFT and the Mel filter bank — why log-mel is approximately perceptually uniform; the Fourier uncertainty trade-off.
- Linear prediction and formants — how classical vocoders reconstructed speech from a few parameters.
- Pitch estimation algorithms — YIN, PYIN, CREPE, SWIPE; why each fails on noisy or creaky-voice inputs.
- Griffin-Lim and neural-vocoder phase reconstruction — why magnitude spectrograms are not enough to recover audio.
- Beamforming, microphone-array processing, blind source separation — if you ever get into multi-mic scenarios.

You should be able to read a paper that uses these terms and understand what's being said at a high level. You should not expect to hand-derive them, and you should not block yourself from shipping on "I don't understand the derivation of the Mel scale." The library implements it; you measure whether it works on your audio.

## A decision tree for who does the work

For any speech-related question that lands on your desk:

1. **Is it a pipeline-integration or configuration question?** (Codec choice, chunk size, resampler library, AEC placement, provider swap, latency measurement.) → You.
2. **Is it a model-selection question with off-the-shelf options?** (Which Whisper variant for this language? Which TTS voice for this brand?) → You, with the NLP engineer available for second opinion if the field is adversarial.
3. **Does the question require fine-tuning an existing model on your domain data?** → Escalate to `nlp-engineer`.
4. **Does the question require training a new model from scratch, or redesigning a loss / architecture?** → Escalate to `ml-engineer` / training-pipeline engineer.
5. **Does the question require a novel representation (new neural codec, new speech-SSL target)?** → Research, not engineering. If the business funds it, this is an NLP-engineer / research-engineer project; if it doesn't, pick an off-the-shelf alternative and move on.

## Reading list to stay literate, not block yourself on

Six papers you should have at least skimmed so that NLP engineers can talk to you without translating:

- wav2vec 2.0 ([Baevski et al., 2020](https://arxiv.org/abs/2006.11477))
- HuBERT ([Hsu et al., 2021](https://arxiv.org/abs/2106.07447))
- Whisper ([Radford et al., 2022](https://arxiv.org/abs/2212.04356))
- EnCodec ([Défossez et al., 2022](https://arxiv.org/abs/2210.13438))
- SoundStream ([Zeghidour et al., 2021](https://arxiv.org/abs/2107.03312))
- Moshi / Mimi ([Défossez et al., 2024](https://arxiv.org/abs/2410.00037))

Read the **abstract, the architecture figure, and the results table**. If your product uses the model or codec, come back and read the ablations. You do not need to read the loss-function appendices.

## Summary

- The voice-agent engineer's working depth stops at pipeline integration, configuration, and consumption of pre-trained speech models.
- Fine-tuning ASR / TTS / neural codecs for a domain is `nlp-engineer` work.
- Training speech models from scratch or redesigning DSP primitives is `ml-engineer` / training-pipeline-engineer work.
- You should be literate enough in wav2vec 2.0, HuBERT, Whisper, EnCodec, SoundStream, and Mimi to collaborate across the boundary — not to reimplement.
- The pattern for handoffs is the same as the module 101 deferral contract: problem statement, current mitigations, concrete reproducer, target threshold, what stays yours.

This closes the content block for `mod-102`. Proceed to the exercises (sample-rate tour, feature-extraction drill, ring-buffer build, codec decision drill, RNNoise/Krisp deployment, and a neural-codec walk) and then on to `mod-103-stt-production-engineering`.
