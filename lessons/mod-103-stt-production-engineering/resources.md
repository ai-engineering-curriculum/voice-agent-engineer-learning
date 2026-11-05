# Resources for mod-103-stt-production-engineering

Primary sources only. Cite these in your work; reach for secondary tutorials only when the primary source is unreadable.

## Provider documentation

### Open-weights / self-hostable

- **OpenAI Whisper** — paper: [Radford et al., 2022](https://arxiv.org/abs/2212.04356); reference implementation: [github.com/openai/whisper](https://github.com/openai/whisper); fast inference: [SYSTRAN/faster-whisper](https://github.com/SYSTRAN/faster-whisper); word timings + diarisation: [m-bain/whisperX](https://github.com/m-bain/whisperX). Model cards on Hugging Face for [`openai/whisper-large-v3`](https://huggingface.co/openai/whisper-large-v3) and [`openai/whisper-large-v3-turbo`](https://huggingface.co/openai/whisper-large-v3-turbo).
- **NVIDIA NeMo (Canary, Parakeet, Conformer)** — [nvidia-nemo on Hugging Face](https://huggingface.co/nvidia); [NeMo docs](https://docs.nvidia.com/nemo-framework/user-guide/latest/).
- **Meta MMS (Massively Multilingual Speech)** — paper: [Pratap et al., 2023](https://arxiv.org/abs/2305.13516); model card: [`facebook/mms-1b-all`](https://huggingface.co/facebook/mms-1b-all).
- **Meta SeamlessM4T v2** — paper: [SeamlessM4T (Seamless Communication team, 2023)](https://arxiv.org/abs/2308.11596); model: [`facebook/seamless-m4t-v2-large`](https://huggingface.co/facebook/seamless-m4t-v2-large); project page: [ai.meta.com/research/seamless-communication](https://ai.meta.com/research/seamless-communication/).

### Managed APIs

- **OpenAI STT** — [Speech-to-text API docs](https://platform.openai.com/docs/guides/speech-to-text). Current endpoints include `whisper-1`, `gpt-4o-transcribe`, `gpt-4o-mini-transcribe`, and realtime audio inputs via the Realtime API.
- **Deepgram** — [developers.deepgram.com/docs](https://developers.deepgram.com/docs). Nova-3 model details; [Streaming API reference](https://developers.deepgram.com/reference/streaming); [Keyterm Prompting docs](https://developers.deepgram.com/docs/keyterm).
- **AssemblyAI** — [www.assemblyai.com/docs](https://www.assemblyai.com/docs). [Universal-Streaming](https://www.assemblyai.com/docs/speech-to-text/universal-streaming); [Pre-recorded word boost](https://www.assemblyai.com/docs/speech-to-text/pre-recorded-audio/word-boost).
- **Rev AI** — [docs.rev.ai](https://docs.rev.ai/). Streaming and async APIs.
- **Speechmatics** — [docs.speechmatics.com](https://docs.speechmatics.com/). Ursa (standard) and enhanced models; on-prem deployment docs.
- **Microsoft Azure AI Speech** — [learn.microsoft.com/azure/ai-services/speech-service](https://learn.microsoft.com/azure/ai-services/speech-service/). Streaming recognition, phrase lists, Custom Speech ([overview](https://learn.microsoft.com/azure/ai-services/speech-service/custom-speech-overview)).
- **Google Cloud Speech-to-Text v2** — [cloud.google.com/speech-to-text/v2/docs](https://cloud.google.com/speech-to-text/v2/docs). Chirp 2 / Chirp 3 model family; [streaming recognize](https://cloud.google.com/speech-to-text/v2/docs/streaming-recognize); [model adaptation](https://cloud.google.com/speech-to-text/v2/docs/adaptation-model).
- **AWS Transcribe** — [docs.aws.amazon.com/transcribe](https://docs.aws.amazon.com/transcribe/). [Streaming transcription](https://docs.aws.amazon.com/transcribe/latest/dg/streaming.html); [Call Analytics](https://docs.aws.amazon.com/transcribe/latest/dg/call-analytics.html); [Custom vocabulary](https://docs.aws.amazon.com/transcribe/latest/dg/custom-vocabulary.html); [Custom Language Models](https://docs.aws.amazon.com/transcribe/latest/dg/custom-language-models.html).

### Audio-native LLMs (speech-LM alternative)

- **OpenAI Realtime API** — [platform.openai.com/docs/guides/realtime](https://platform.openai.com/docs/guides/realtime). `gpt-4o-realtime` audio input and output.
- **Google Gemini audio input** — [ai.google.dev/gemini-api/docs/audio](https://ai.google.dev/gemini-api/docs/audio).
- **Anthropic Claude audio input** — [docs.anthropic.com](https://docs.anthropic.com/) (feature availability varies by model and release).
- **Moshi / Mimi (Kyutai)** — paper: [Défossez et al., 2024](https://arxiv.org/abs/2410.00037); code: [github.com/kyutai-labs/moshi](https://github.com/kyutai-labs/moshi).
- **Qwen2-Audio** — model: [`Qwen/Qwen2-Audio-7B-Instruct`](https://huggingface.co/Qwen/Qwen2-Audio-7B-Instruct); paper: [Chu et al., 2024](https://arxiv.org/abs/2407.10759).

## Benchmarks and corpora

### English

- **LibriSpeech** — [openslr.org/12](https://www.openslr.org/12/); paper: [Panayotov et al., 2015](https://www.danielpovey.com/files/2015_icassp_librispeech.pdf). `dev-clean` / `test-clean` + `dev-other` / `test-other` splits.
- **TED-LIUM 3** — [openslr.org/51](https://www.openslr.org/51/); [Hernandez et al., 2018](https://arxiv.org/abs/1805.04699).
- **GigaSpeech** — [SpeechColab/GigaSpeech](https://github.com/SpeechColab/GigaSpeech); [Chen et al., 2021](https://arxiv.org/abs/2106.06909).
- **Earnings-22** — [revdotcom/speech-datasets](https://github.com/revdotcom/speech-datasets); [Del Rio et al., 2022](https://arxiv.org/abs/2203.15591). Noisy finance-domain audio.
- **AMI meeting corpus** — [groups.inf.ed.ac.uk/ami](http://groups.inf.ed.ac.uk/ami/corpus/). Multi-party meeting recordings; useful for diarisation benchmarking.

### Multilingual

- **FLEURS** — paper: [Conneau et al., 2022](https://arxiv.org/abs/2205.12446); dataset: [`google/fleurs`](https://huggingface.co/datasets/google/fleurs). 102 languages, parallel.
- **Multilingual LibriSpeech (MLS)** — paper: [Pratap et al., 2020](https://arxiv.org/abs/2012.03411); [openslr.org/94](https://www.openslr.org/94/).
- **Common Voice (Mozilla)** — [commonvoice.mozilla.org](https://commonvoice.mozilla.org/). Multilingual crowdsourced.
- **CoVoST 2** — [github.com/facebookresearch/covost](https://github.com/facebookresearch/covost); [Wang et al., 2020](https://arxiv.org/abs/2007.10310). 21-to-English and English-to-15 speech translation; usable for ASR.
- **VoxPopuli** — [github.com/facebookresearch/voxpopuli](https://github.com/facebookresearch/voxpopuli); [Wang et al., 2021](https://arxiv.org/abs/2101.00390). European Parliament multilingual speech.

### Code-switching

- **SEAME** — Mandarin-English code-switching corpus; LDC-licensed. See [Dong-Chul Lyu et al., 2010](https://www.isca-archive.org/interspeech_2010/lyu10_interspeech.pdf).
- **ArzEn** — Egyptian Arabic–English code-switching; dataset at [github.com/munakhoury/ArzEn](https://github.com/munakhoury/ArzEn) and successors. <!-- needs-research: confirm the current canonical ArzEn distribution and licence at author time -->
- **MERLIon CCS Challenge 2023** — English–Mandarin code-switching; organiser's resources. <!-- needs-research: confirm current MERLIon artefact URL at author time -->

## Core ASR research (reading list, not reimplement)

### Architectures

- **Conformer** — [Gulati et al., 2020](https://arxiv.org/abs/2005.08100). Convolution-augmented transformer.
- **Transformer Transducer** — [Zhang et al., 2020](https://arxiv.org/abs/2002.02562). RNN-T with transformer encoder.
- **RNN-T** — [Graves, 2012](https://arxiv.org/abs/1211.3711). The streaming-friendly decoder.
- **CTC** — [Graves et al., ICML 2006](https://www.cs.toronto.edu/~graves/icml_2006.pdf). Connectionist Temporal Classification.
- **Whisper** — [Radford et al., 2022](https://arxiv.org/abs/2212.04356).

### Self-supervised speech representations

- **wav2vec 2.0** — [Baevski et al., 2020](https://arxiv.org/abs/2006.11477).
- **HuBERT** — [Hsu et al., 2021](https://arxiv.org/abs/2106.07447).
- **WavLM** — [Chen et al., 2021](https://arxiv.org/abs/2110.13900).
- **MMS** — [Pratap et al., 2023](https://arxiv.org/abs/2305.13516).

### Evaluation and fairness

- **Comparing human and machine errors in conversational speech** — [Stolcke and Droppo, 2017](https://arxiv.org/abs/1708.08615). Human WER floors and model-error typology.
- **Racial disparities in automated speech recognition** — [Koenecke et al., 2020](https://www.pnas.org/doi/10.1073/pnas.1915768117). Required reading on accent / sociolect stratification.
- **MER and WIL metrics** — [Morris, Maier, and Green, 2004](https://www.isca-archive.org/interspeech_2004/morris04_interspeech.pdf).
- **A review of automated speech recognition evaluation** — [Errattahi et al., 2018 "Automatic Speech Recognition Errors Detection and Correction: A Review"](https://www.sciencedirect.com/science/article/pii/S1877050918302187). Overview of WER / CER / stratified evaluation practice.
- **Calibration of modern neural networks** — [Guo et al., 2017](https://arxiv.org/abs/1706.04599). Foundational paper on calibration and temperature scaling.
- **Expected Calibration Error** — [Naeini, Cooper, Hauskrecht, 2015](https://ojs.aaai.org/index.php/AAAI/article/view/9602). Original ECE formulation.
- **Platt scaling** — [Platt, 1999, "Probabilistic Outputs for Support Vector Machines"](https://www.cs.colorado.edu/~mozer/Teaching/syllabi/6622/papers/Platt1999.pdf).
- **Isotonic regression for calibration** — [Zadrozny and Elkan, 2002, "Transforming Classifier Scores into Accurate Multiclass Probability Estimates"](https://dl.acm.org/doi/10.1145/775047.775151).

## Tooling

### Scoring

- **jiwer** — [github.com/jitsi/jiwer](https://github.com/jitsi/jiwer). WER / CER / MER / WIL in Python with normalisation pipelines.
- **torchmetrics.text** — [torchmetrics.readthedocs.io](https://torchmetrics.readthedocs.io/). `WordErrorRate`, `CharErrorRate`.
- **NIST SCLITE** — [github.com/usnistgov/SCTK](https://github.com/usnistgov/SCTK). Reference alignment and stratified reporting toolkit.
- **Whisper normalisers** — [openai/whisper/normalizers](https://github.com/openai/whisper/tree/main/whisper/normalizers). English and multilingual text normalisers; de facto community standard.

### Calibration

- **scikit-learn** — [`sklearn.calibration.CalibratedClassifierCV`](https://scikit-learn.org/stable/modules/generated/sklearn.calibration.CalibratedClassifierCV.html); [`sklearn.isotonic.IsotonicRegression`](https://scikit-learn.org/stable/modules/generated/sklearn.isotonic.IsotonicRegression.html).

### VAD + diarisation

- **webrtcvad** — [github.com/wiseman/py-webrtcvad](https://github.com/wiseman/py-webrtcvad). The WebRTC VAD as a Python wrapper.
- **Silero VAD** — [github.com/snakers4/silero-vad](https://github.com/snakers4/silero-vad). Neural-net VAD; current open-weights default.
- **pyannote.audio** — [github.com/pyannote/pyannote-audio](https://github.com/pyannote/pyannote-audio). Diarisation, VAD, overlap detection; [Bredin et al., 2020](https://arxiv.org/abs/1911.01255).

### Streaming client stacks

- **LiveKit Agents** — [docs.livekit.io/agents](https://docs.livekit.io/agents/). STT plugin surface and voice-agent primitives.
- **Pipecat** — [docs.pipecat.ai](https://docs.pipecat.ai/). Audio-frame-centric pipeline framework; STT adapters.
- **Twilio Media Streams** — [twilio.com/docs/voice/media-streams](https://www.twilio.com/docs/voice/media-streams). PSTN audio ingestion.

## Compliance and deployment references

- **HIPAA / Business Associate Agreements** — [hhs.gov/hipaa](https://www.hhs.gov/hipaa/). Required when transcripts or audio contain PHI.
- **PCI-DSS** — [pcisecuritystandards.org](https://www.pcisecuritystandards.org/). Required when audio contains cardholder data.
- **GDPR** — [gdpr.eu](https://gdpr.eu/). Required for EU-user audio and transcripts.
- **SOC 2** — [AICPA Trust Services Criteria](https://us.aicpa.org/interestareas/frc/assuranceadvisoryservices/aicpasoc2report). The baseline for most enterprise contracts.
- **FedRAMP** — [fedramp.gov](https://www.fedramp.gov/). US federal deployment posture.

## Reference tracks on the ladder (for the boundary chapter)

- **nlp-engineer (L30)** — the AI-engineering-curriculum role that owns ASR model architecture internals, pretraining, representation extraction. <!-- needs-research: confirm the canonical nlp-engineer-learning repo URL at author time -->
- **model-evaluation-engineer (L30)** — owns benchmark methodology, significance testing, calibration-at-depth, judge design. <!-- needs-research: confirm the canonical model-evaluation-engineer-learning repo URL at author time -->
- **fine-tuning-engineer (L30)** — owns ASR fine-tuning as a modelling task. <!-- needs-research: confirm the canonical fine-tuning-engineer-learning repo URL at author time -->
- **training-pipeline-engineer (L35)** — owns training infrastructure, continuous training cadences. <!-- needs-research: confirm the canonical training-pipeline-engineer-learning repo URL at author time -->

## Citation hygiene

- Provider pricing, latency, WER, and feature pages change frequently. When a specific number matters, cite the dated page and archive your evidence.
- Marketing WER numbers are not peer-reviewed. Treat as marketing until reproduced on your own corpus — the methodology is in [Chapter 5](05-wer-benchmarking-librispeech-and-in-domain.md).
- Open-weights model performance claims should cite the model card plus the paper. The model card is the artefact that ships; the paper is the methodology behind it.
- Compliance certifications have expiration dates. A provider "with HIPAA" that last renewed its BAA two years ago is a red flag. Verify at contract time, not at scorecard time.
