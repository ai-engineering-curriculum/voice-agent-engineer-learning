# 04. Pipeline vs. Speech-Native vs. Hybrid: The Architectural Decision

## Why this chapter

The previous three chapters taught you three of the stacks you have a choice between: OpenAI Realtime (speech-native), Gemini Live (speech-native with vision), Claude-via-relay (pipeline). This chapter is the decision itself — when to pick each shape, scored against the axes that actually matter, with the compliance filter applied first and the latency / cost / quality trade-offs made explicit.

The decision is not "which vendor." It is **which architecture**. Vendor is downstream — once you've chosen pipeline, you still pick STT, LLM, TTS vendors (Modules 103 and 104 are those decisions); once you've chosen speech-native, you still pick between OpenAI Realtime and Gemini Live. The architecture is the invariant; the vendor list under it rotates.

The frame of the chapter mirrors the TTS and STT provider-scoring frames of Modules 103 and 104: run compliance first, then score the architectures on the axes, then write a one-page recommendation. Do not produce a weighted sum — weights are a lie and the point of the matrix is to make trade-offs legible.

## The three shapes, restated in one paragraph each

**Pipeline (STT → LLM → TTS).** Three independent streaming services, each the best at its job, wired together through text. STT emits a transcript; the LLM reasons on text, calls tools, writes a reply; TTS synthesises the reply as audio. Latency is the sum of three; audit log is the three transcripts in plain text; voice selection is a TTS config choice. Modules 103 and 104 are the per-leg detail.

**Speech-native (Realtime / Gemini Live / Moshi / Qwen2-Audio).** A single multimodal model that accepts audio in and emits audio out directly. No intermediate text transcript (optionally, a server-side transcription for logging). Latency is one model's end-to-end response time; the model can hear paralinguistic signal (whispering, crying, laughing) and emit it in kind; voice selection is limited to the vendor's shipped voices.

**Hybrid.** The pipeline and the speech-native stack run side by side; an orchestrator routes turns to one or the other based on turn shape. Chatty turns go to the speech-native stack for latency. Tool-heavy, regulated, or multi-step turns go to the pipeline for auditability and reasoning. The user hears a unified agent; the architecture picks the right tool per turn.

## The seven axes to score

The weights depend on your deployment. Keep each axis independent; do not compress them into a single number.

### 1. Latency (TTFB and perceived response)

Rough shapes, verified by your own measurement, not vendor claims:

| Architecture | Typical TTFB | Floor (well-tuned) | Ceiling (expect worse under load) |
|---|---|---|---|
| Pipeline (STT→LLM→TTS, tuned) | 600–900 ms | ~500 ms | 1.5–2 s at p95 |
| Speech-native (Realtime / Gemini Live) | 300–700 ms | ~250–300 ms | 1 s at p95 |
| Speech-native full-duplex (Moshi, see Chapter 5) | 100–300 ms | sub-200 ms | depends on self-host hardware |
| Hybrid (routed) | Pipeline on tool turns, speech-native on chat turns; per-turn shape from either column |

**Why the gap.** In a pipeline, three distinct network RTTs stack: mic → STT, STT → LLM, LLM → TTS; plus LLM time-to-first-token plus TTS time-to-first-audio-byte. A speech-native model runs one RTT and one end-to-end inference. You do not beat physics by moving the components closer; the only way to close the gap is to collapse the components.

**When this axis wins.** Consumer companions, virtual assistants, in-game voice, any product where "the agent feels alive" is the product. Anything below ~400 ms end-to-end implies speech-native or full-duplex.

**When this axis loses.** Telephony IVR (users tolerate 800 ms on a phone call — they expect it), regulated workflows (where the user is reading a form aloud and expects a considered response), any workflow where the model is doing multi-step tool use anyway.

Module 109 is the full latency-budget treatment. For this chapter, hold the rough shapes.

### 2. Cost

**Pipeline.** Each leg bills independently. Current rough shape (verify every provider at integration time):

- STT: a few dollars per thousand hours of audio for streaming (Deepgram, AssemblyAI; Whisper via OpenAI is per-minute).
- LLM: per-token input + output; voice sessions are chatty so output tokens dominate.
- TTS: per character or per 1M characters; expressive and cloning tiers are more expensive.

**Speech-native.** Billed per **audio-second** on input and output, typically with distinct rates. This is the right unit — the model is computing on audio. It can be dramatically cheaper than a pipeline for short, high-volume interactions (no three-vendor overhead) or dramatically more expensive for long sessions because audio tokens are substantially more expensive per "equivalent" word than text tokens. Measure, don't assume.

**Hybrid.** Both bills; the orchestrator's routing decides the mix.

**When this axis wins.** B2C at scale; any product where the unit economics live or die at per-turn cost. Model it honestly: average turn length × turns per session × sessions per day, at steady state.

### 3. Interrupt behaviour and barge-in

**Pipeline.** You own barge-in completely. Your VAD decides when the user is speaking again; you cancel the TTS stream, truncate history, hand audio back to STT. Latency to silence depends on your TTS's mid-stream cancel support (some vendors drain; some cut at the frame boundary).

**Speech-native.** The vendor owns server-side VAD. You trust it, or you run in manual mode (Chapters 1 and 2 both document the manual path). When it works, interrupts are very snappy (sub-100 ms perceived). When the server's VAD mis-classifies (background noise, overlapping speech), you have fewer levers.

**Full-duplex (Moshi).** The model streams audio with the user, not after; it can hear itself and the user simultaneously and respond inside the user's utterance ("mhm," short acknowledgments, mid-sentence clarifications). This is a distinct interaction model, not just lower latency. Chapter 5.

**When this axis wins.** Any conversational product where users interrupt the agent frequently — companion apps, customer service, tutoring. Measuring "interrupts per 100 turns" on a pilot tells you how much this axis matters.

### 4. Voice-selection freedom and voice-IP

**Pipeline.** Any TTS you want. Instant voice clones, professional voice clones (brand voice), licensed voice actors, open-weights voices self-hosted. You own the IP of the voice (subject to Module 104's consent-flow surface).

**Speech-native.** The vendor's shipped voices. Typically six to a dozen pre-trained voices per vendor, with no cloning path. You cannot have "our brand voice" on a speech-native stack today; you can only pick the one that sounds closest.

**When this axis wins.** B2C with a brand voice; characters / personas tied to specific voices; licensed celebrity partnerships; voice preservation; any accessibility use case that needs a specific voice.

**When this axis loses.** Early-stage product where the voice just needs to be "good"; internal tools; anything where the voice is an implementation detail.

### 5. Logging, auditability, and inspectability

**Pipeline.** Every word the LLM saw and emitted is in plain text. Three transcript streams (user STT, LLM input + output, TTS text) make for clean logs; SIEM integration is straightforward; a compliance officer can read exactly what happened.

**Speech-native.** Logs are audio-shaped. The transcript the vendor returns is a best-effort automatic transcription; it is not literally what the model heard, and it is not the only thing the model attended to (prosody, filler words, timing). You can still log the audio and the vendor's transcript, but you cannot say "here is the exact input to the model" with the same confidence a pipeline gives.

**When this axis wins.** Regulated industries (finance, healthcare, public sector), workflows where disputes require a forensic record, workflows where RAG citations must be auditable. For these the pipeline is often the only compliant choice; the speech-native stack would require vendor affidavits a vendor may not provide.

**When this axis loses.** Consumer products; internal tools where logs are for debugging, not compliance.

### 6. Safety mitigations and refusal behaviour

**Pipeline.** You can inject a safety classifier at any seam: on the STT transcript (detect intent to self-harm; route to a specialist policy), between LLM and TTS (filter the response), in the TTS itself (add a watermark, refuse certain voices). The pipeline is a set of chokepoints you can intervene at.

**Speech-native.** The vendor's safety stack is between you and the model. OpenAI and Google both publish system cards that describe the mitigations shipped in Realtime and Live; Chapter 6 reads those in depth. You cannot insert a classifier between the user's audio and the model; you can only post-filter the output audio (if the vendor exposes the transcript in time) and use the vendor's refusal behaviour as-is.

**When this axis wins.** Any deployment where a specific safety behaviour (self-harm referral protocol; regulated-advice refusal; crisis handoff; emotion-inference prohibition under EU AI Act Article 5(1)(f)) must be guaranteed. The pipeline gives you the knobs; the speech-native stack gives you the vendor's defaults.

### 7. Language and locale coverage

**Pipeline.** The union of your STT's languages, your LLM's languages, and your TTS's languages. STT and TTS typically cover more languages than speech-native models; open-weights STT (Whisper, MMS, SeamlessM4T) extends coverage further. You can mix providers to cover a long-tail locale.

**Speech-native.** The vendor's supported-languages list, which is typically a subset of the pipeline union — even for "multilingual" speech-native models, specific voices or specific tool-calling behaviours may be weaker in non-English locales. Verify on your actual locales; "supports Spanish" without disaggregating `es-MX` from `es-ES` is a marketing claim.

**When this axis wins.** Any deployment with a long-tail of locales (healthcare in the US has 30+ regularly-spoken languages; EU deployments span dozens).

## The scorecard template

Flat table. Score against **your deployment**. Blocking filters first; axes second; recommendation third.

| Axis | Weight for this deployment | Pipeline (STT→LLM→TTS, Claude on the LLM leg) | Speech-native (OpenAI Realtime) | Speech-native (Gemini Live) | Full-duplex speech-native (Moshi, self-host) | Hybrid (route by turn shape) |
|---|---|---|---|---|---|---|
| Compliance posture (BAA, GDPR, FedRAMP) | **Blocking** | | | | | |
| Regional data residency | **Blocking** | | | | | |
| Required languages / locales | **Blocking** | | | | | |
| TTFB target (p50) | High | | | | | |
| TTFB target (p95) | High | | | | | |
| Cost per turn (steady state) | | | | | | |
| Interrupt latency ceiling | | | | | | |
| Voice selection / brand voice | | | | | | |
| Paralinguistic signal needed | | | | | | |
| Auditability / inspectability | | | | | | |
| Mid-session config flexibility | | | | | | |
| Tool-call density per turn | | | | | | |
| Video / screen-share in loop | | | | | | |
| Grounding (search, RAG) | | | | | | |
| Safety mitigation controls | | | | | | |
| Operational complexity (surfaces you run) | | | | | | |
| Vendor lock-in / exit cost | | | | | | |

Three cells are blocking (not scoring): compliance posture, regional residency, required locales. A stack that fails those is out, regardless of its TTFB or price.

## The decision tree

Before the scorecard, run the compliance tree to prune:

1. **Is the audio or its content regulated** (PHI under HIPAA; PCI; EU data under GDPR without an adequacy decision for the vendor's region)? If yes, the stack must have the specific compliance posture and a signed BAA / DPA. Many speech-native vendors are HIPAA-eligible; Gemini Live via Vertex AI offers EU residency; verify the exact posture today. If no stack qualifies, you are on self-hosted open-weights (Chapter 7).
2. **Does the product need video / screen-share in the voice turn?** If yes, Gemini Live (speech-native) or a custom pipeline with a vision model are your options. The Realtime API and most pipelines are audio-only.
3. **Is the latency ceiling below the pipeline floor (~600 ms)?** If yes, speech-native is forced. If sub-300 ms is a hard requirement, consider full-duplex (Moshi) with a self-host cost.
4. **Does the deployment need a brand voice / voice IP / cloned voice?** If yes, pipeline is forced — speech-native vendors do not clone today.
5. **Is tool density high (more than one tool call per turn on average)?** If yes, pipeline or hybrid; speech-native handles tool calls, but is weaker than a text LLM at multi-step tool orchestration.
6. **Does regulatory posture require inspectable text logs** (every word in and out)? If yes, pipeline. Speech-native vendors may provide transcripts, but they are not literal inputs to the model.
7. **Otherwise** — speech-native is the usual fast path for consumer / internal / prototype work; pipeline is the usual fast path for enterprise / regulated / branded work; hybrid is the usual fast path when the deployment has both shapes and the orchestration overhead is justified.

Document which row of this tree your project lives in. It is the piece of context most often missing from architecture write-ups.

## A worked frame — three deployments, three different answers

### Deployment A — telephony IVR for an EU health-insurance renewal line

Compliance posture: HIPAA-equivalent (GDPR + national health laws), audio must stay in the EU, every word in and out logged, consent capture on call start.

- **Speech-native?** Gemini Live via Vertex AI (EU region) is possible for the compliance posture, but inspectability of inputs to the model is weaker than a pipeline. Realtime API's compliance posture is weaker for EU residency.
- **Pipeline?** Deepgram (EU residency) → Claude via AWS Bedrock Frankfurt → Azure Neural Voice (EU residency) is the clean answer. Every leg is EU-residency, HIPAA-eligible, and inspectable.
- **Hybrid?** Overkill for a mostly-IVR workflow.
- **Pick: pipeline.** Recommendation: Deepgram Nova-3 → Claude Sonnet via Bedrock → Azure Neural Voice (German + English), ConversationRelay or Pipecat on the orchestration leg.

### Deployment B — consumer companion app, in-app voice (WebRTC), brand voice

Compliance: none regulated; GDPR-aware for EU users. Latency target: 400 ms TTFB p95. Voice: a specific ElevenLabs Professional Voice Clone, the brand persona.

- **Speech-native?** Fast and cheap, but the brand voice is a hard requirement — speech-native cannot clone. Out.
- **Pipeline?** Deepgram → Claude → ElevenLabs Flash v2 is around 700 ms TTFB p95, over the target.
- **Hybrid?** Possible: speech-native for the chatty turns (user hears a near-match voice) and pipeline for the branded reads. But users will notice the voice swap mid-conversation; this fails the brand-voice requirement.
- **Pick: pipeline, optimise latency aggressively, accept a 600-ms floor.** The brand voice is a dealbreaker above all other axes.

### Deployment C — kitchen-camera voice assistant, English, consumer

Compliance: none. Latency target: 500 ms. The user shows the camera an ingredient and says "what can I make with this?" Continuous video needed.

- **Speech-native (Gemini Live)?** Yes — continuous video + audio in a single turn is the capability it owns, and the latency fits.
- **Pipeline?** Would need a separate vision model inline; cost and complexity go up; latency floor pushes past 1 s.
- **Hybrid?** Unnecessary for a single-shape deployment.
- **Pick: Gemini Live, speech-native.**

Three deployments, three different answers. The point of the matrix is to see why.

## How to actually fill in the matrix

Same disciplines as the TTS and STT matrices:

- **Dated evidence.** Each cell links to the vendor doc page with a date you read it, or to your benchmark run ID. Vendor landscapes rot.
- **Measured numbers.** TTFB and cost cells are measured in your deployment's network region on your actual utterances. Do not take vendor marketing.
- **Negative cells.** If a stack does not support a feature you need, the cell says "no" — not blank. Blank reads as "didn't check."
- **A one-page recommendation.** Primary architecture, under what assumptions. Fallback architecture and the specific axis that would trigger the switch. POC plan: 2–3 things you cannot resolve from docs + prototypes. Open risks.

## Common anti-patterns

- **Picking speech-native because it is new.** The newer stack is not always the right one. Score on axes, not on recency.
- **Picking pipeline because it feels more professional.** Three-component pipelines have real latency and operational costs. Score on axes, not on aesthetics.
- **Picking hybrid without turn-type data.** Hybrid pays off only when there are genuinely distinct turn shapes and the orchestration cost is less than the win. Measure; do not hope.
- **"Speech-native to prototype, pipeline later."** The two have different UX implications (brand voice, paralinguistic signal, inspectability); switching after launch is expensive. Pick for the destination, not for the prototype.
- **Ignoring Module 106 (barge-in) in the architecture decision.** Interrupt behaviour differs qualitatively between the architectures; the Module 106 chapter on barge-in is in-scope input for this decision.
- **Scoring on a weighted sum.** Weights are a lie; different stakeholders (ops, legal, product) have different weights, and the single-number ranking hides the trade.

## Summary

- There are three architectural shapes: pipeline, speech-native, hybrid. The decision is architecture-first; vendor is downstream.
- Score against: latency, cost, interrupt behaviour, voice-selection freedom, logging / inspectability, safety mitigations, language coverage, plus blocking filters for compliance, residency, and locale support.
- Run the compliance tree first; it eliminates stacks before anything else matters.
- Expect pipeline TTFB around 600–900 ms, speech-native around 300–700 ms, full-duplex around 100–300 ms. These are the floors — your deployment will measure differently.
- Brand voice, inspectable text logs, and dense tool calling push toward pipeline. Live video, sub-400-ms latency, paralinguistic signal push toward speech-native.
- Produce a scorecard **and** a one-page recommendation tied to specific assumptions. Do not produce a weighted sum.

Next chapter: the half-duplex / full-duplex distinction underneath the speech-native option, and why Moshi, Advanced Voice, and Gemini Live represent three distinct points on the spectrum with different downstream implications for turn-taking, latency, and consent.
