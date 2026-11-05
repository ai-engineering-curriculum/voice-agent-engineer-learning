# 01. The TTS Provider Landscape and the TTFB-First Scoring Decision

## Why this chapter

Module 103 argued that the STT pick sets your WER floor, your latency floor, your language coverage, and your compliance posture. The TTS pick does the same from the other end of the pipeline — it sets your **time-to-first-audio-byte (TTFB)** floor, your naturalness ceiling, your voice-identity envelope, your SSML ergonomics, and the second half of your unit economics. Where the STT decision is dominated by accuracy, the TTS decision is dominated by latency: the user cannot hear what has not yet been synthesised, and the thing they hear first is "silence after my question." Every millisecond spent choosing the wrong architecture is a millisecond the agent feels dead.

This chapter teaches you how to score the TTS landscape **for your deployment**, with TTFB as the latency primitive and naturalness, expressiveness, SSML support, language coverage, cloning, cost, and compliance as the other axes. It does not pick a vendor — pricing and model versions move faster than any curriculum can — but it teaches you to run the decision in a way that survives contact with production.

## The current landscape

Treat this list as the set you'll usually shortlist from. Current as of late 2025; dates every claim to a provider doc page you should re-verify at author time.

### Managed, latency-first (voice-agent-shaped)

- **Cartesia Sonic family** — [docs.cartesia.ai](https://docs.cartesia.ai/). The Sonic, Sonic Turbo, and Sonic 2 line is explicitly engineered for voice agents, with Cartesia advertising **sub-100 ms model latency** for Sonic 2 and WebSocket + SSE streaming contracts. Sonic 2 is the current latency reference for managed TTS; verify the exact TTFB claim and the sampling-rate options at integration time.
- **ElevenLabs** — [elevenlabs.io/docs](https://elevenlabs.io/docs). The voice-cloning-forward vendor; four model families relevant here: **Flash v2** (lowest-latency, English), **Turbo v2.5** (low-latency multilingual), **Multilingual v2** (quality-first, non-streaming-strong), and **Conversational** (expressive, backs ElevenLabs' Conversational AI product). The WebSocket streaming endpoint is the real voice-agent contract; the HTTP endpoints are quality-first.
- **OpenAI TTS** — [platform.openai.com/docs/guides/text-to-speech](https://platform.openai.com/docs/guides/text-to-speech). Three model surfaces: `tts-1` (lower-latency), `tts-1-hd` (higher-quality), and `gpt-4o-mini-tts` (steerable — you can prompt the voice's affect, pacing, and style in natural language). The Realtime API (`gpt-4o-realtime`, Module 105) is a separate path that ships audio out directly.
- **PlayHT** — [docs.play.ht](https://docs.play.ht/). **Play 3.0** for low-latency streaming; **Play Dialog** for multi-speaker conversational synthesis. WebSocket streaming native.
- **LMNT** — [docs.lmnt.com](https://docs.lmnt.com/). Low-latency streaming TTS with a voice-cloning layer; smaller vendor with a sharp latency focus.
- **Hume EVI** — [dev.hume.ai](https://dev.hume.ai/). Empathic Voice Interface: a full voice-agent runtime (STT + LLM + TTS + prosody model) whose selling point is **prosodic expressiveness driven by inferred affect**. When you buy EVI you are buying the whole stack as one unit; evaluating its TTS in isolation is possible but partial.

### Managed, enterprise / cloud-vendor

- **Google Cloud Text-to-Speech — Chirp 3 HD voices** — [cloud.google.com/text-to-speech/docs](https://cloud.google.com/text-to-speech/docs). The Chirp 3 HD line is Google's current high-quality neural voice family; streaming via gRPC. SSML support with a Google subset.
- **Microsoft Azure AI Speech — Neural Voices** — [learn.microsoft.com/azure/ai-services/speech-service/text-to-speech](https://learn.microsoft.com/azure/ai-services/speech-service/text-to-speech). Deep SSML support (including Microsoft-specific extensions for `mstts:express-as` style, roles, prosody), a large voice catalogue, Custom Neural Voice (CNV) for professional clones, and on-prem container deployment. The go-to enterprise choice for SSML-heavy work.
- **AWS Polly** — [docs.aws.amazon.com/polly](https://docs.aws.amazon.com/polly/). Three engines matter: **Neural** (the baseline), **Long-Form** (for narration / podcast-length audio), and **Generative** (Polly's current expressive flagship). SSML support with Polly's subset. Streaming via `SynthesizeSpeech` with the stream response.
- **Resemble AI** — [docs.resemble.ai](https://docs.resemble.ai/). Voice-cloning-forward with an explicit provenance / watermarking stance (see [Chapter 5](05-voice-cloning-consent-and-provenance.md)).

### Open-weights / self-hostable

Covered at architecture level in [Chapter 6](06-generative-tts-architectures-vits-voicebox-xtts-bark.md); named here for the shortlist:

- **Coqui XTTS v2** — [`coqui/XTTS-v2`](https://huggingface.co/coqui/XTTS-v2). Zero-shot multilingual voice cloning from a short reference clip. Coqui the company has wound down, but the weights remain available.
- **VITS / VITS2 / StyleTTS 2** — the end-to-end open-weights family; practical when you want fully local inference and you can live without the managed polish.
- **Bark (Suno AI)** — [`suno/bark`](https://huggingface.co/suno/bark). Audio-language-model-style generation; expressive, slow, not realtime.
- **Parler-TTS** — [`parler-tts/parler-tts-large-v1`](https://huggingface.co/parler-tts/parler-tts-large-v1). Text-conditioned voice style ("a female speaker with a slightly low-pitched, calm voice").
- **F5-TTS / E2 TTS** — [`SWivid/F5-TTS`](https://huggingface.co/SWivid/F5-TTS). Flow-matching zero-shot cloning; current open-weights SOTA by many informal comparisons.

### Audio-native / speech-to-speech alternative

As with STT, there is a path that **skips the TTS step entirely**: an audio-native LLM that emits speech directly. Covered in Module 105. The models in scope are OpenAI's Realtime (`gpt-4o-realtime`), Google's Gemini Live, Kyutai Moshi, and Hume EVI's end-to-end mode. They are not strictly a TTS choice — they are a pipeline-shape choice — but any TTS scorecard that does not name them as the alternative is incomplete.

## The eight axes you score on

Score providers against these **for your deployment**. Generic "best TTS" claims mean nothing; each axis's weight depends on your product shape and compliance envelope.

1. **TTFB — time-to-first-audio-byte.** The core latency metric. Measured from "last token of the LLM response arrives at the TTS client" to "first PCM byte reaches the audio sink." On voice agents, anything above ~400 ms begins to feel sluggish; below 200 ms feels snappy. TTFB is the number that matters, **not total synthesis time** — the player can consume audio as fast as it arrives, so you only ever wait for the first byte. [Chapter 2](02-streaming-tts-contract-and-player-pipeline.md) is dedicated to this.
2. **Naturalness — MOS / CMOS.** Mean Opinion Score (0–5, higher is better) and Comparative MOS (pairwise preference) are the subjective quality primitives. Every vendor claims SOTA on MOS. The number that matters is MOS **on your text** with **your voice** rated by **your listeners**; marketing MOS on neutral prompts does not transfer. ITU-T Rec. P.808 is the standard methodology for crowdsourced MOS collection.
3. **Expressive range.** How well does the voice handle emotion, questions, emphasis, pacing, interjections, laughter, whispering? Expressive providers (ElevenLabs Conversational, Hume EVI, Cartesia Sonic Turbo, Polly Generative, `gpt-4o-mini-tts`) differ meaningfully from neutral providers (classical Azure Neural Voices, Polly Neural, older Google WaveNet). [Chapter 4](04-expressive-vs-neutral-tts-and-voice-selection-ux.md) walks the decision.
4. **SSML support — subset / extensions.** W3C SSML 1.1 is the standard; **nobody implements it faithfully**. Each provider ships a subset plus extensions. If you are writing prosody-rich speech, the subset you can rely on matters more than the full specification. [Chapter 3](03-ssml-prosody-and-phonetic-lexicons.md) enumerates the diffs.
5. **Language and locale coverage.** How many languages, which locales, what accents per language, and which voices per locale. "Spanish" means nothing without `es-ES`, `es-MX`, `es-US`, `es-AR` disaggregation.
6. **Voice cloning — instant vs. professional.** Instant voice clone (IVC) from a 30-second to 3-minute sample; professional voice clone (PVC) from a studio session with consented identity verification. [Chapter 5](05-voice-cloning-consent-and-provenance.md) is dedicated to this because the legal surface alone can take a quarter to clear.
7. **Cost.** Billed per character (ElevenLabs, PlayHT, Cartesia, OpenAI), per audio-second generated (Polly Generative), or per 1M characters (Google, Azure). Model the steady state honestly: average-response-length-in-characters × turns-per-call × calls-per-day. Expressive and cloning tiers are usually billed higher than neutral.
8. **Compliance / deployment posture.** On-prem container (Azure, some open-weights), VPC / dedicated (several enterprise tiers), managed only (most consumer-grade vendors). HIPAA BAA, PCI, SOC 2, FedRAMP, GDPR regional residency. For healthcare, finance, government, or EU-regulated work, this axis dominates.

A ninth axis that doesn't belong on the scorecard but will blow up the project if ignored: **voice-identity provenance and takedown workflow**. A provider whose voice catalogue is partially the output of other customers' uploaded clips, with no audit trail you can present to legal, is a liability. More in [Chapter 5](05-voice-cloning-consent-and-provenance.md).

## The deployment-posture decision tree

Run compliance first; quality only distinguishes between providers that cleared the filter.

1. **Is the generated audio or its text regulated?** (PHI responses under HIPAA; account-balance or card-last-4 under PCI; EU-user data under GDPR with no adequacy decision for the vendor's region.) If yes, you need either on-prem / self-hosted TTS, a VPC-dedicated managed deployment, or a cloud provider with the specific compliance certification *and a signed BAA / DPA* that names the TTS service.
2. **Are you cloning a named person's voice?** (Celebrity, employee, customer.) If yes, you cannot ship without the consent flow and provenance artefacts from [Chapter 5](05-voice-cloning-consent-and-provenance.md). Some jurisdictions (Illinois BIPA, the US NO FAKES Act, the UK Data (Use and Access) Act) will constrain this regardless of vendor posture.
3. **Is the latency ceiling below what a managed vendor advertises end-to-end?** (Sub-200 ms TTFB for a region where the managed vendor only has one PoP, 300 ms RTT away.) If yes, either colocate in the vendor's region or self-host.
4. **Does the product need voices in locales no managed vendor covers well?** If yes, either shortlist a locale-specialist vendor or self-host an open-weights model with a fine-tune.
5. **Otherwise** — managed is almost always the right default. Running a TTS fleet (GPU inference, voice-asset caching, autoscaling, continuous eval, watermark / provenance logging) is substantial, and for most products a vendor whose entire company is that fleet will beat you on both quality and ops.

Document which row of this tree your project lives in. It is the piece of context most often missing from TTS write-ups.

## Scorecard template

Flat table. No weighted sums — weights are a lie and the point of the matrix is to make trade-offs legible, not to produce a single number.

| Axis | Weight for this deployment | Cartesia Sonic 2 | ElevenLabs (Flash v2 / Multilingual v2) | OpenAI (`tts-1` / `gpt-4o-mini-tts`) | Google Chirp 3 HD | Azure Neural | Polly (Neural / Generative) | PlayHT (Play 3.0) | Hume EVI | Self-host (XTTS / F5-TTS) |
|---|---|---|---|---|---|---|---|---|---|---|
| TTFB (p50 / p95) | High | | | | | | | | | |
| TTFB (regional PoPs you can use) | | | | | | | | | | |
| Naturalness — MOS on your text | High | | | | | | | | | |
| Expressive range (emotion, laughter, emphasis) | | | | | | | | | | |
| SSML subset you need (phoneme, break, prosody, say-as, lexicon) | High | | | | | | | | | |
| Phoneme / pronunciation-lexicon support | | | | | | | | | | |
| Language + locale coverage required | High | | | | | | | | | |
| Voice catalogue size (and regeneration stability) | | | | | | | | | | |
| Voice cloning — IVC / PVC / neither | | | | | | | | | | |
| Streaming contract (WS / SSE / chunked HTTP / SDK only) | | | | | | | | | | |
| Barge-in support (interrupt + mid-stream cancel) | High | | | | | | | | | |
| Cost per 1M characters (streaming) | | | | | | | | | | |
| Compliance certifications you require | Blocking | | | | | | | | | |
| Deployment postures (SaaS / VPC / on-prem) | Blocking | | | | | | | | | |
| Provenance / watermark posture | | | | | | | | | | |
| SDK languages / stability | | | | | | | | | | |

Two columns are **blocking**, not scoring: compliance and deployment posture. If a provider fails those for your project, mark the row red and move on — their MOS does not matter.

## How to actually fill it in

You need three data sources to populate the matrix without lying to yourself:

- **Provider documentation.** The deterministic facts: supported languages, deployment postures, pricing pages, feature matrices, SSML tag lists. Archive the page as of your read date; these pages change.
- **Your own TTFB and MOS benchmark.** Script every shortlisted provider's streaming endpoint from your production region, with real text drawn from your LLM's typical responses, measure TTFB p50 / p95, and run a small MOS panel (10–20 listeners, P.808-style ABX) on 20–30 utterances. The methodology shape is in [Chapter 2](02-streaming-tts-contract-and-player-pipeline.md) (for TTFB) and [Chapter 4](04-expressive-vs-neutral-tts-and-voice-selection-ux.md) (for the A/B).
- **A trial / dev-account integration.** Half a day wiring each shortlisted provider's streaming API. You will learn more from reading its WebSocket error messages than from any blog post.

Three disciplines that make the output trustworthy:

- **Dated evidence.** Each cell links to the vendor doc page with a date, or to your benchmark run ID. "Cartesia Sonic 2 is sub-100 ms" without a dated link rots.
- **Negative cells.** If a provider does not support a feature you need (SSML `<phoneme>` on a specific locale, mid-stream cancel, a specific locale) the cell says "no" — not blank. Blank reads as "I didn't check."
- **A one-page recommendation at the end.** Primary provider, under what assumptions, for which phases. "ElevenLabs Flash v2 for the English-language MVP; migrate the EU-regulated workload to Azure Neural Voices in VPC once we have >100 calls/day there; keep Cartesia Sonic 2 warm as the latency-tie-breaker alternative." A matrix without a recommendation is a research exercise, not a decision.

## A worked frame — the three axes most teams get wrong

Teams I've watched pick TTS tend to get three things wrong. Spot them in your own matrix:

- **MOS without TTFB.** The team picks the highest-MOS vendor, discovers in production that its p95 TTFB is 900 ms, and the agent feels dead. TTFB is the floor; MOS differentiates vendors that cleared the floor.
- **"Naturalness" as a scalar.** Naturalness has three sub-axes: segmental quality (does each phoneme sound right), prosodic appropriateness (do questions rise, lists cadence, emphasis land), and voice identity (does it sound like a consistent person turn after turn). Most providers are strong on one and weak on another. Rate them separately.
- **SSML support as a boolean.** Provider says "supports SSML." Provider in fact supports `<break>` and `<prosody>` but not `<phoneme>`, not `<lexicon>`, and silently drops unknown tags. If your drug-name / product-name / acronym layer depends on `<phoneme>`, you have a problem. [Chapter 3](03-ssml-prosody-and-phonetic-lexicons.md) is the drill.

## Common anti-patterns

- **"We'll support everyone via an abstraction layer."** Same mistake as in STT, worse in TTS. TTS vendor contracts diverge on streaming protocol, chunking cadence, SSML subset, voice-ID lifecycle, and barge-in semantics. An abstraction that papers over this costs you both the per-provider optimisation surface *and* your ability to reason about player behaviour. A thin adapter per vendor, called from one place, is the usual sweet spot.
- **Picking on one-line prompts.** "Hello, how can I help you today?" sounds great on every provider. Your real responses include six-digit numbers, URLs, product SKUs, dollar amounts, hesitations, and apology phrasing. Benchmark on your text.
- **Ignoring barge-in.** If the user interrupts the agent mid-sentence, can you cancel the in-flight TTS synthesis? Can you stop the audio at a word boundary? Some providers support mid-stream cancel; some only let the stream drain. This is a Module 106 requirement that lands on the TTS contract in this module — do not defer it.
- **Benchmarking TTFB from your laptop.** Your laptop is not your region. Measure from the production egress path, during business hours, against the vendor PoP you will actually use.
- **Picking the cheapest provider for the MVP and committing to the migration cost "later."** Switching TTS providers after you have voice-identity consistency built into UX, SSML tuned per voice, pronunciation lexicons shipped, and a barge-in behaviour your users depend on is roughly a quarter of engineering work. Pick like you're going to stay.

## Summary

- The shortlist is: Cartesia, ElevenLabs, OpenAI, PlayHT, LMNT for latency-first managed; Google Chirp 3 HD, Azure Neural, AWS Polly for enterprise managed; Resemble for provenance-forward cloning; XTTS / F5-TTS / Parler / Bark for open-weights; Hume EVI as end-to-end; audio-native speech-to-speech as the "skip TTS" alternative (Module 105).
- Score providers on TTFB, MOS on your text, expressive range, SSML subset, locale coverage, voice cloning posture, cost, and compliance — with SDK-stability and provenance-workflow as tie-breakers.
- Compliance and deployment posture are blocking filters, not scoring axes. Apply them first.
- The output is a scorecard **and** a one-page recommendation tied to a phase, not a weighted sum pretending to be objective.

Next chapter: the streaming TTS contract, and how to design a player pipeline where TTFB — not total synthesis time — is the latency driver.
