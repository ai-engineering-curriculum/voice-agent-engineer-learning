# Resources for mod-112-voice-multimodal-eval-and-observability

Primary sources only. Eval methodology, speech benchmarks, and observability platforms all move; class names, dataset versions, and semconv attribute names can change between releases. Date every citation that depends on a specific attribute name, a dataset version, or a vendor feature.

## ASR evaluation (Chapter 1)

### Metrics

- **jiwer** (Python reference implementation of WER / CER / MER) — [github.com/jitsi/jiwer](https://github.com/jitsi/jiwer).
- **torchmetrics `WordErrorRate`** — [lightning.ai/docs/torchmetrics/stable/text/word_error_rate.html](https://lightning.ai/docs/torchmetrics/stable/text/word_error_rate.html).
- **Morris, Maier, Green 2004 — "From WER and RIL to MER and WIL"** — the MER / WIL definition. Available via the ISCA Archive; search the paper title.
- **OpenAI Whisper text normalizers** — reference implementation in [`openai-whisper`](https://github.com/openai/whisper) under `whisper.normalizers`.

### Datasets

- **FLEURS** — [huggingface.co/datasets/google/fleurs](https://huggingface.co/datasets/google/fleurs); paper: [Conneau et al. 2022, "FLEURS: Few-shot Learning Evaluation of Universal Representations of Speech"](https://arxiv.org/abs/2205.12446).
- **LibriSpeech** — [openslr.org/12](https://www.openslr.org/12); paper: [Panayotov et al. 2015, "Librispeech: an ASR corpus based on public domain audio books"](https://www.danielpovey.com/files/2015_icassp_librispeech.pdf).
- **TED-LIUM 3** — [openslr.org/51](https://www.openslr.org/51/); paper: [Hernandez et al. 2018, "TED-LIUM 3"](https://arxiv.org/abs/1805.04699).
- **FLoRes-101** (sentence source for FLEURS) — [ai.meta.com/research/flores](https://ai.meta.com/research/flores/).

### Inter-rater reliability and confidence calibration

- **Cohen 1960, "A Coefficient of Agreement for Nominal Scales"** — Cohen's kappa definition; widely reprinted.
- **Fleiss 1971, "Measuring nominal scale agreement among many raters"** — Fleiss's kappa.
- **Landis & Koch 1977, "The Measurement of Observer Agreement for Categorical Data"** — Biometrics; kappa interpretation rules of thumb (heuristics only).
- **Guo et al. 2017, "On Calibration of Modern Neural Networks"** — [arxiv.org/abs/1706.04599](https://arxiv.org/abs/1706.04599); reliability diagrams and ECE.
- **`netcal`** — [github.com/EFS-OpenSource/calibration-framework](https://github.com/EFS-OpenSource/calibration-framework) — calibration metrics and plots.
- **Clopper–Pearson binomial confidence interval** — [en.wikipedia.org/wiki/Binomial_proportion_confidence_interval](https://en.wikipedia.org/wiki/Binomial_proportion_confidence_interval); pointer-only, verify with a primary stats text.

## TTS evaluation (Chapter 2)

### Standards

- **ITU-T P.808** — [itu.int/rec/T-REC-P.808](https://www.itu.int/rec/T-REC-P.808). Crowdsourced subjective-speech-quality protocol.
- **ITU-T P.835** — [itu.int/rec/T-REC-P.835](https://www.itu.int/rec/T-REC-P.835). SIG / BAK / OVRL method for noise-suppressed speech.
- **ITU-T P.800** — [itu.int/rec/T-REC-P.800](https://www.itu.int/rec/T-REC-P.800). The older ACR / CCR methodologies, still the reference for the vocabulary (MOS, CMOS).
- **ITU-R BT.1359-1** — [itu.int/rec/R-REC-BT.1359](https://www.itu.int/rec/R-REC-BT.1359). Audio-video synchronisation detectability (multimodal use, mod-110).

### Reference implementations

- **Microsoft P.808 repo** — [github.com/microsoft/P.808](https://github.com/microsoft/P.808). P.808 and P.835 templates on MTurk + data cleaning scripts.
- **DNS Challenge** — [github.com/microsoft/DNS-Challenge](https://github.com/microsoft/DNS-Challenge). Benchmarks and models used for noise suppression; uses P.835 as its metric.

### Automated predictors

- **UTMOS** — [github.com/sarulab-speech/UTMOS22](https://github.com/sarulab-speech/UTMOS22); paper: [Saeki et al. 2022, "UTMOS: UTokyo-SaruLab System for VoiceMOS Challenge 2022"](https://arxiv.org/abs/2204.02152).
- **VoiceMOS Challenge 2022** (training-data distribution) — [voicemos-challenge-2022.github.io](https://voicemos-challenge-2022.github.io/).
- **DNSMOS** — [arxiv.org/abs/2010.15258](https://arxiv.org/abs/2010.15258); DNSMOS P.835 extension: [arxiv.org/abs/2110.01763](https://arxiv.org/abs/2110.01763). Reference models in the DNS Challenge repo.
- **MOSNet** — [arxiv.org/abs/1904.08352](https://arxiv.org/abs/1904.08352).
- **NISQA** — [github.com/gabrielmittag/NISQA](https://github.com/gabrielmittag/NISQA); Interspeech 2021.
- **SQUIM (Meta / Facebook)** — in [`torchaudio.prototype.pipelines.SquimObjective`](https://pytorch.org/audio/main/prototype.pipelines.html#squim-objective); ICASSP 2023.

### Lip-sync (multimodal)

- **SyncNet** — [robots.ox.ac.uk/~vgg/software/lipsync](https://www.robots.ox.ac.uk/~vgg/software/lipsync/) and associated papers (Chung & Zisserman).

## Conversational evaluation (Chapter 3)

### Forced alignment / ground-truth boundaries

- **Montreal Forced Aligner (MFA)** — [montreal-forced-aligner.readthedocs.io](https://montreal-forced-aligner.readthedocs.io/).
- **WhisperX** — [github.com/m-bain/whisperX](https://github.com/m-bain/whisperX). Word-timestamped alignment layered on Whisper.

### Latency budget (inherited from mod-109)

- Cross-reference to [mod-109 — Real-Time Latency Budget Architecture](../mod-109-latency-budget-architecture/README.md); the budget commitment and per-leg attribution are the pass/fail anchor for Chapter 3.

### Turn-taking / VAD / barge-in (inherited from mod-106)

- Cross-reference to [mod-106 — VAD, Turn-Taking, Barge-In](../mod-106-vad-turn-taking-barge-in/README.md); the primitives that Chapter 3 measures.

## Content eval and judge-as-LLM (Chapter 4)

- **Zheng et al. 2023, "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena"** — [arxiv.org/abs/2306.05685](https://arxiv.org/abs/2306.05685). Canonical LLM-as-judge paper with bias measurements.
- **G-Eval** — [arxiv.org/abs/2303.16634](https://arxiv.org/abs/2303.16634). Chain-of-thought LLM evaluator for NLG.
- **Arena-Hard** — [github.com/lmarena/arena-hard-auto](https://github.com/lmarena/arena-hard-auto). Auto-judge framework using Chatbot Arena preferences.
- **MT-Bench** — included in Zheng et al.; [github.com/lm-sys/FastChat](https://github.com/lm-sys/FastChat) tree.

## Online eval, drift, and feedback (Chapter 5)

- **Evidently AI** — [github.com/evidentlyai/evidently](https://github.com/evidentlyai/evidently). Drift tests, data quality, monitoring.
- **Alibi Detect** — [github.com/SeldonIO/alibi-detect](https://github.com/SeldonIO/alibi-detect). Statistical drift detectors (KS, MMD, LSDD, etc.).
- **Page 1954, "Continuous Inspection Schemes"** — [doi.org/10.1093/biomet/41.1-2.100](https://doi.org/10.1093/biomet/41.1-2.100). The original CUSUM paper.
- Cross-reference to [mod-108 — Safety & Compliance for Voice](../mod-108-safety-compliance-voice/README.md) for the consent / retention constraints on online eval data.

## OpenTelemetry and WebRTC stats (Chapter 6)

### OpenTelemetry

- **OTel Semantic Conventions for Generative AI** — [opentelemetry.io/docs/specs/semconv/gen-ai](https://opentelemetry.io/docs/specs/semconv/gen-ai/). The `gen_ai.*` attributes. Draft at authoring time; check current version.
- **OTel Python SDK** — [opentelemetry.io/docs/languages/python](https://opentelemetry.io/docs/languages/python/).
- **OTel Node SDK** — [opentelemetry.io/docs/languages/js](https://opentelemetry.io/docs/languages/js/).
- **OTel trace propagation** — [opentelemetry.io/docs/concepts/context-propagation](https://opentelemetry.io/docs/concepts/context-propagation/).

### WebRTC stats

- **W3C WebRTC Statistics** — [w3.org/TR/webrtc-stats](https://www.w3.org/TR/webrtc-stats/). Authoritative on `getStats()` dictionary objects.
- **RFC 3550 (RTP)** — [datatracker.ietf.org/doc/html/rfc3550](https://datatracker.ietf.org/doc/html/rfc3550). Interarrival jitter definition.
- **W3C WebRTC** — [w3.org/TR/webrtc](https://www.w3.org/TR/webrtc/). The main spec.
- Cross-reference to [mod-109 Chapter 2 — Measuring and Attributing Per-Leg Latency](../mod-109-latency-budget-architecture/02-measuring-and-attributing-per-leg-latency.md).

## Evaluation / observability platforms (Chapter 6)

Each entry is positional — verify current OTel ingestion support, audio handling, and compliance posture against the vendor's own docs.

- **Langfuse** — [langfuse.com](https://langfuse.com/); repo: [github.com/langfuse/langfuse](https://github.com/langfuse/langfuse).
- **LangSmith** — [smith.langchain.com](https://smith.langchain.com/); docs: [docs.smith.langchain.com](https://docs.smith.langchain.com/).
- **Arize Phoenix** — [phoenix.arize.com](https://phoenix.arize.com/); repo: [github.com/Arize-ai/phoenix](https://github.com/Arize-ai/phoenix). OpenInference spec: [github.com/Arize-ai/openinference](https://github.com/Arize-ai/openinference).
- **Weights & Biases Weave** — [wandb.ai/site/weave](https://wandb.ai/site/weave).
- **Braintrust** — [braintrust.dev](https://www.braintrust.dev/).

## Role boundaries (Chapter 7)

- **AI Engineering — AI Eval Engineer (L30)** — [`ai-eval-engineer-learning`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning). Peer role on deep eval methodology.
- **AI Governance — AI Evaluation Engineer (L35)** — [`ai-evaluation-engineer-learning`](https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning). Same name, different org; release-assurance / regulator-facing scope.
- **Agentic Safety Engineer (frontier dangerous-capability evals)** — referenced in mod-108 Chapter 8.

## Citation hygiene

- **Semconv attribute names move.** The GenAI semconv is a living document; pin the version string you adopted on every root span.
- **Dataset versions move.** FLEURS, LibriSpeech, and TED-LIUM 3 all have multiple releases; pin the version you evaluated against in the release-assurance packet.
- **Vendor MOS proxies are regressions onto human labels.** UTMOS / DNSMOS / NISQA / SQUIM each have in-distribution failure modes; a correlation study on *your* distribution is the only defence.
- **"Realtime" and "live" are not SLOs.** Mouth-to-mouth p95 is measured in your region on your traffic; verify before citing any vendor's published number.
- **Crowdsource ethics.** P.808 implies a minimum-wage floor and informed rater qualification; document the payment rate and the rater pool in every subjective-panel citation.
- **Judge calibration data has a shelf life.** Re-validate kappa whenever the judge model, the judge prompt, or the transcript source changes.
