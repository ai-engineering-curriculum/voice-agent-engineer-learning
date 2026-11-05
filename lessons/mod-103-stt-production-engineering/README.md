# mod-103-stt-production-engineering: Speech-to-Text (ASR / STT) Production Engineering

**Estimated effort:** 16 hours · **Role:** Voice & Multimodal Agent Engineer · **Level:** L35

## What this module is for

Picking the STT is the single highest-leverage decision in the voice pipeline. It sets your WER floor, your latency floor, your language coverage, your compliance posture, and about half of your unit economics. Every other module in this track inherits the choice.

This module is the engineer's working depth on STT: how to score providers against the deployment, how to read a streaming contract, how to design the LLM handoff so STT uncertainty doesn't turn into LLM confabulation, how to ship domain adaptation at the cheapest rung that works, how to benchmark honestly, and how to handle multilingual + accent realities. It also draws the boundary to `nlp-engineer`, `model-evaluation-engineer`, and `training-pipeline-engineer` — the three sibling roles that own the parts that are not yours.

## Learning objectives

- Select and wire the STT provider for the deployment — Whisper (hosted vs. self-hosted whisper-large-v3-turbo + faster-whisper), Deepgram Nova-3 (streaming + diarization + keyterm prompting), AssemblyAI Universal-2, Rev AI, Speechmatics Ursa (multilingual), Microsoft Azure AI Speech, Google Cloud STT v2 (Chirp universal ASR), AWS Transcribe + Call Analytics — scoring against latency, WER, language coverage, diarization quality, confidence calibration, cost, and the compliance posture (on-prem / VPC / managed).
- Author the streaming vs. batch decision — streaming for interactive voice (sub-300 ms partial-transcript target), batch for offline analytics; interpret streaming-provider contracts (interim vs. final results, word timings, confidence).
- Design the confidence-informed handoff into the LLM — reject-and-ask-again, confidence-weighted token masking, homophone disambiguation hints, acoustic-feature-aware reranking; decide when to defer the acoustic signal into the LLM via a speech-LLM instead.
- Ship domain adaptation — custom vocabulary, keyterm prompting / boost lists, phonetic lexicons, model customisation / fine-tune (request to `fine-tuning-engineer` or `training-pipeline-engineer` peer with the data contract) — against the WER delta you can defend.
- Benchmark WER / CER / MER against standard corpora (LibriSpeech, TED-LIUM 3, FLEURS multilingual) and against your in-domain gold set; author inter-rater-reliability-checked transcription for the in-domain set; align WER numbers to business-KPI delta (cost per handled call, deflection rate, escalation rate).
- Handle multilingual + code-switching + accent scenarios — the FLEURS per-language picture, the SeamlessM4T option for speech-to-text in 100 languages, and the when-to-use-a-local-ASR-model vs. a general-purpose foundation model.
- Cite the boundary to `nlp-engineer` (level 30) on deep ASR-model-architecture internals (Conformer, wav2vec 2.0 pretraining), to `model-evaluation-engineer` (level 30) on benchmark / statistical-methodology depth, and to `training-pipeline-engineer` (level 35) on ASR fine-tune pipelines.

## Lecture chapters

1. [01 — The STT Provider Landscape and the Scoring Decision](01-stt-provider-landscape-and-scoring.md)
2. [02 — Streaming vs. Batch: The Decision and the Contract](02-streaming-vs-batch-decision-and-contract-walkthrough.md)
3. [03 — The Confidence-Informed Handoff into the LLM](03-confidence-informed-handoff-into-the-llm.md)
4. [04 — Domain Adaptation: Custom Vocab, Keyterm Prompting, Lexicons, and Fine-Tune](04-domain-adaptation-vocab-keyterms-finetune.md)
5. [05 — Benchmarking WER / CER / MER — LibriSpeech, TED-LIUM 3, FLEURS, and Your Own Gold Set](05-wer-benchmarking-librispeech-and-in-domain.md)
6. [06 — Multilingual, Code-Switching, and Accent Scenarios](06-multilingual-code-switching-and-accents.md)
7. [07 — The Boundary with `nlp-engineer`, `model-evaluation-engineer`, and `training-pipeline-engineer`](07-boundary-to-nlp-eval-and-training-engineers.md)

## Exercises

- [exercise-01 — STT Provider Decision Matrix](exercises/exercise-01-stt-provider-decision-matrix.md) (2 h)
- [exercise-02 — Streaming vs. Batch Decision + Streaming-Contract Walkthrough](exercises/exercise-02-streaming-vs-batch-decision-and-contract-walkthrough.md) (2.5 h)
- [exercise-03 — Confidence-Informed Handoff into the LLM](exercises/exercise-03-confidence-informed-handoff-into-the-llm.md) (3 h)
- [exercise-04 — Ship Custom Vocab and Keyterm Prompting](exercises/exercise-04-ship-custom-vocab-and-keyterm-prompting.md) (2.5 h)
- [exercise-05 — WER Benchmark: LibriSpeech + In-Domain Gold Set](exercises/exercise-05-wer-benchmark-against-librispeech-and-an-in-domain-gold-set.md) (3 h)
- [exercise-06 — Multilingual and Code-Switching Ladder](exercises/exercise-06-multilingual-and-code-switching-ladder.md) (2.5 h)

## Structure

- `01-…md` … `07-…md` — lecture chapters.
- `exercises/` — per-exercise prompts.
- `labs/` — long-form hands-on labs (none in this module; the exercises are already hands-on, and Module 112 / the capstone projects do the longer-form integration work).
- `quizzes/` — knowledge checks.
- `resources.md` — primary-source references for everything cited in this module.

## How to use this module

Read the chapters in order. Chapter 1 forces the provider-selection decision up front; Chapter 2 nails down the streaming contract that selection commits you to. Chapters 3 and 4 are the two pipeline-engineering disciplines that make the pick pay off — handoff design and domain adaptation. Chapter 5 is the measurement backbone that disciplines the whole module and feeds Module 112's observability work. Chapter 6 handles the multilingual reality. Chapter 7 is the boundary chapter — who owns what, and what the handoff artefacts look like.

The exercises are best done after the matching chapter rather than at the end. Exercise 1 pairs with Chapter 1, exercise 2 with Chapter 2, etc. Exercise 5 (the WER benchmark) produces artefacts — a reusable scoring script and an in-domain gold set — that you will reuse across the rest of the track. Prioritise that one if time is short.

Prerequisite: [mod-102-speech-pipeline-foundations](../mod-102-speech-pipeline-foundations/README.md).

Next module: [mod-104-tts-production-engineering](../mod-104-tts-production-engineering/README.md).
