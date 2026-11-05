# 01. The STT Provider Landscape and the Scoring Decision

## Why this chapter

Picking an STT provider is the single highest-leverage decision in the voice pipeline. It sets your WER floor, your latency floor, your language coverage, your compliance posture, and about half of your unit-economics. Every other module in this track inherits its choice: the LLM handoff design (Module 103 Chapter 3), the latency budget (Module 109), the compliance envelope (Module 108), the evaluation surface (Module 112). Get the pick wrong and you will fight it for the lifetime of the product.

This chapter gives you the current landscape, the axes you score providers on, and the decision patterns that reliably survive contact with production. It does **not** tell you which vendor to buy — pricing and model versions move faster than any curriculum can. It teaches you how to run the decision.

## The current landscape (late 2025)

Treat this list as the set you'll usually shortlist from. You can find others; these are the ones whose products are mature enough to carry a production voice agent.

### Open-weights / self-hostable

- **OpenAI Whisper** — Encoder-decoder transformer trained on 680k hours of weakly labelled multilingual audio ([Radford et al., 2022](https://arxiv.org/abs/2212.04356)). Open weights under MIT. The `large-v3` checkpoint is the quality reference for open ASR; `large-v3-turbo` is a distilled variant with fewer decoder layers (trading a small WER regression for a roughly 4–8× speed-up, depending on hardware and language). Hosted at OpenAI (`gpt-4o-transcribe`, `whisper-1`); self-hostable via [`openai/whisper`](https://github.com/openai/whisper), [`faster-whisper`](https://github.com/SYSTRAN/faster-whisper) (CTranslate2 backend, the usual production choice), and [`WhisperX`](https://github.com/m-bain/whisperX) (word timings + speaker diarisation).
- **NVIDIA NeMo Canary / Parakeet** — Open-weights Conformer / FastConformer-based models from NVIDIA, released on [Hugging Face](https://huggingface.co/nvidia). Strong on English and a small set of other languages; the Parakeet-TDT family targets fast English transcription.
- **Meta MMS (Massively Multilingual Speech)** — [Pratap et al., 2023](https://arxiv.org/abs/2305.13516). 1 100+ ASR languages. The go-to for low-resource languages where no vendor has coverage. Not a streaming-first design.
- **Meta SeamlessM4T v2** — [`facebook/seamless-m4t-v2-large`](https://huggingface.co/facebook/seamless-m4t-v2-large). Speech-to-text in ~100 languages plus speech-to-speech translation. Big, slow, high-quality.

### Managed / API

- **Deepgram Nova-3** — Streaming-first ASR with diarisation and **keyterm prompting**. Focus on sub-300 ms partial latency. ([Deepgram docs](https://developers.deepgram.com/docs).)
- **AssemblyAI** — Universal-2 for batch / high-accuracy work; Universal-Streaming for interactive agents. Diarisation, formatting, redaction, and an auto-chapters stack. ([AssemblyAI docs](https://www.assemblyai.com/docs).)
- **Rev AI** — Historically the human-transcription vendor; their model stack is tuned for American English conversational audio and has a reputation for good handling of overlapping speech. ([Rev AI docs](https://docs.rev.ai/).)
- **Speechmatics Ursa** — Multilingual streaming ASR with a strong accent-robustness positioning. On-prem deployment available. ([Speechmatics docs](https://docs.speechmatics.com/).)
- **Microsoft Azure AI Speech** — Full stack (STT, TTS, translation, custom speech). Deeply integrated with Teams / contact-centre tooling. Fine-tune ("Custom Speech") and phrase-list biasing available. ([Azure Speech docs](https://learn.microsoft.com/azure/ai-services/speech-service/).)
- **Google Cloud Speech-to-Text v2 (Chirp)** — Chirp 2 / Chirp 3 "universal ASR model" family, available as streaming and batch endpoints with speech adaptation ("boost"). ([Google STT docs](https://cloud.google.com/speech-to-text/v2/docs).)
- **AWS Transcribe + Transcribe Call Analytics** — AWS's in-house ASR plus a contact-centre analytics layer (sentiment, categorisation, PII redaction, agent-caller diarisation). ([AWS Transcribe docs](https://docs.aws.amazon.com/transcribe/).)

### Speech-LLM / audio-native models (the alternative)

A separate option: **skip transcription as a discrete step** and feed audio into an audio-native LLM that produces text (or speech) directly. The models that land in a voice-agent scope are OpenAI's `gpt-4o`/`gpt-4o-realtime` audio modes, Google's Gemini audio inputs, Anthropic's Claude audio inputs (where supported), Kyutai's open-weights [Moshi](https://arxiv.org/abs/2410.00037), and the Qwen2-Audio family. We revisit this choice in [Chapter 3](03-confidence-informed-handoff-into-the-llm.md) and again in Module 105. For now: it is a real alternative to the STT-then-LLM pipeline, and provider selection should name it as such.

## The eight axes you score on

Score providers against these axes **for your deployment** — generic "best STT" claims mean nothing because each axis's weight depends on your product.

1. **Latency.** For interactive voice, the number that matters is **first partial transcript latency** (how long after the user starts speaking before you see any text) and **endpointing latency** (how long after the user stops speaking before you see the final). Batch is a different regime; see [Chapter 2](02-streaming-vs-batch-decision-and-contract-walkthrough.md).
2. **WER on your domain.** The provider's advertised WER on LibriSpeech test-clean is a necessary baseline, not a sufficient one. The real number is WER against your own gold set — the method is in [Chapter 5](05-wer-benchmarking-librispeech-and-in-domain.md).
3. **Language and accent coverage.** How many languages, with what WER each; what happens on code-switched utterances; what happens on accents under-represented in the training data. [Chapter 6](06-multilingual-code-switching-and-accents.md) does this properly.
4. **Diarisation quality.** Does the provider separate speakers? How many? With what diarisation error rate (DER)? Does it label the agent vs. the caller on a two-channel call?
5. **Confidence calibration.** Does the provider return a per-word or per-utterance confidence score, and is that score **calibrated** — meaning that "0.70 confidence" actually corresponds to a ~70 % chance of being correct? Miscalibrated confidence poisons the downstream LLM handoff in [Chapter 3](03-confidence-informed-handoff-into-the-llm.md).
6. **Domain-adaptation features.** Custom vocabulary / phrase lists, keyterm prompting / boost, phonetic lexicons, model fine-tune. [Chapter 4](04-domain-adaptation-vocab-keyterms-finetune.md) enumerates them.
7. **Cost.** Usually billed per minute of audio for streaming and per minute (or per request) for batch. Some vendors bill extra for diarisation, PII redaction, or custom-model hosting. Model this honestly: usage × minutes-per-call × calls-per-day.
8. **Compliance / deployment posture.** On-prem, VPC, dedicated instance, standard managed? HIPAA / PCI / SOC 2 / FedRAMP posture? Where does the audio go and for how long? Regions available? Data-retention controls? For healthcare, finance, or EU-regulated work this axis dominates.

A ninth axis that doesn't belong on the scorecard but will blow up the project if you ignore it: **SDK quality and streaming contract stability**. A provider whose Python SDK is the only path and whose streaming protocol has changed three times in a year will cost you more in engineering time than a 10 % WER delta.

## The deployment-posture decision tree

Compliance rules out options before quality distinguishes between them, so run this first.

1. **Is the audio or its transcript regulated?** (PHI under HIPAA; cardholder data under PCI; personal data under GDPR in a region without an adequacy decision; classified / government work.) If yes, you need either an on-prem / self-hosted model, a VPC-dedicated managed deployment, or a cloud provider with the specific compliance certification *and a signed BAA / DPA* covering the service.
2. **Is there a latency / cost ceiling that only self-hosted can meet?** (Example: a call centre handling a million minutes a month, where managed ASR cost dominates everything else.) If yes, self-host and own the ops.
3. **Does the product need coverage of a language or accent that no single managed vendor has?** If yes, either shortlist a multilingual-focused vendor (Speechmatics Ursa, Azure, Google Chirp) or self-host MMS / Whisper for the long tail.
4. **Otherwise** — managed is almost always the right default. The engineering cost of self-hosting an ASR fleet (GPU scheduling, autoscaling, model hot-swap, continuous evaluation, observability) is substantial, and you will under-invest in it relative to a vendor whose entire product is that fleet.

Document which row of this tree your project lives in. It is the single piece of context most often missing from provider-selection write-ups.

## Scorecard template

Use a flat Markdown or spreadsheet template; fight the urge to compute weighted sums. The weights are a lie — the point of the matrix is to make the deltas visible and the trade-offs legible.

| Axis | Weight for this deployment | Whisper-turbo (self-host) | Deepgram Nova-3 | AssemblyAI U-Streaming | Google STT v2 (Chirp) | Azure Speech | AWS Transcribe |
|---|---|---|---|---|---|---|---|
| First-partial latency (p50) | High | | | | | | |
| Endpoint latency (p50) | High | | | | | | |
| WER on in-domain gold set | High | | | | | | |
| WER on LibriSpeech test-clean (baseline) | Medium | | | | | | |
| Diarisation DER (if needed) | | | | | | | |
| Confidence calibration (ECE) | | | | | | | |
| Keyterm / phrase-list support | | | | | | | |
| Fine-tune / Custom-Model support | | | | | | | |
| Language coverage required | High | | | | | | |
| Cost per minute (streaming) | | | | | | | |
| Compliance certifications you require | Blocking | | | | | | |
| Deployment postures (SaaS / VPC / on-prem) | Blocking | | | | | | |
| Streaming contract: interim / final / word-timing / confidence fields | | | | | | | |
| SDK language support | | | | | | | |

Two columns here are **blocking**, not scoring: if a provider fails compliance or deployment posture for your project, they're out before any accuracy number matters. Mark them red, don't rank them.

## How to actually fill it in

You need three data sources to populate the matrix without lying to yourself:

- **Provider documentation** for the deterministic facts: supported languages, deployment postures, pricing pages, feature matrices. Archive what the page said on the date you read it; those pages change.
- **Your own benchmark.** Run every shortlisted provider on the same 10–30 minutes of **your** audio, scored against your own gold transcripts. The methodology is in [Chapter 5](05-wer-benchmarking-librispeech-and-in-domain.md). Published WERs on marketing pages are not evidence.
- **A trial / dev-account integration.** Spend half a day wiring each shortlisted provider's streaming API. You will learn more from reading its WebSocket error messages than from any blog post.

Three disciplines that make the output trustworthy:

- **Dated evidence.** Each cell links to the vendor doc page with a date, or to your benchmark run ID. "Deepgram supports keyterm prompting" without a dated link rots the moment the feature is renamed.
- **Negative cells.** If a provider does not support a language you need, the cell says "no" — not blank. Blank reads as "I didn't check."
- **A one-page recommendation at the end.** Which provider, under what assumptions, for what phases. "Managed Deepgram Nova-3 for the English-language MVP; migrate the EU-regulated workload to Azure Speech in VPC once we have >100 calls/day there; keep Whisper-turbo + faster-whisper warm as the self-host fallback." A matrix without a recommendation is a research exercise, not a decision.

## Common anti-patterns

- **"We'll support everyone via an abstraction layer."** Attractive on paper, usually a mistake. STT provider contracts diverge in ways that are hard to hide — partial-vs-final semantics, word-timing representations, confidence-field shapes, endpointing-behaviour defaults, keyterm-boost semantics. An abstraction that papers over this costs you both the per-provider optimisation surface *and* the ability to reason about pipeline behaviour. A thin adapter per provider, called from one place, is usually the sweet spot.
- **Benchmarking only on LibriSpeech.** LibriSpeech is clean read English audiobook speech recorded with a close-talk mic. Your users are on cell phones in moving cars. LibriSpeech is a floor, not a ceiling.
- **Picking the cheapest provider for the MVP and committing to the migration cost "later."** Switching ASR providers after you have a confidence-calibrated LLM handoff, domain-adapted vocabulary, and an evaluation pipeline that depends on a specific partial-vs-final contract is roughly a quarter of engineering work. Pick like you're going to stay.
- **Treating WER as the only quality number.** A provider at 8 % WER that mis-transcribes dollar amounts ("fifty four" vs "five four") is strictly worse for a billing agent than one at 11 % WER that gets numbers right. Stratify errors by category — see [Chapter 5](05-wer-benchmarking-librispeech-and-in-domain.md) on substitution/deletion/insertion breakdowns.
- **Trusting marketing "state-of-the-art" claims.** Every STT vendor claims SOTA on a benchmark that favours them. The decision rule is: reproduce on your corpus or discard.

## Summary

- The shortlist is: Whisper (hosted or self-hosted) for the open-weights baseline; Deepgram, AssemblyAI, Rev, Speechmatics, Azure, Google Chirp, AWS Transcribe for managed; MMS and SeamlessM4T for the multilingual long tail; speech-LLMs as the "skip STT entirely" alternative.
- Score providers on latency, WER-on-your-domain, language coverage, diarisation, confidence calibration, cost, and compliance posture — with an SDK-stability tie-breaker.
- Compliance and deployment posture are blocking filters, not scoring axes. Apply them first.
- The output is a scorecard **and** a one-page recommendation tied to a phase, not a weighted sum that pretends to be objective.

Next chapter: the streaming vs. batch decision, and how to read the contract a streaming STT provider actually gives you.
