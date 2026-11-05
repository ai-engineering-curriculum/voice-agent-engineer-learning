# 07. The Boundary with `training-pipeline-engineer` and `agentic-safety-engineer`

## Why this chapter

Chapters 1 – 6 are the voice-agent engineer's **working depth** on TTS: pick the vendor, read the streaming contract, author SSML + a lexicon, decide expressive vs. neutral, walk the voice-cloning stack with consent + provenance, and be literate at architecture level. That is the L35 bar.

There is a training-and-safety side of the same discipline — custom voice fine-tunes at scale, continuous training pipelines, frontier voice-safety methodology, adversarial evaluation of synthetic-voice misuse — that is **not yours**. It belongs to two sibling roles. This chapter names them, draws the boundary, and gives you the deferral contracts.

## The two sibling roles

- **`training-pipeline-engineer` (L35)** — in the sibling [`training-pipeline-engineer-learning`](https://github.com/ai-engineering-curriculum) curriculum. <!-- needs-research: confirm canonical repo URL at author time --> Owns training infrastructure: dataset ETL, distributed training, continuous training cadences, checkpoint / artefact management. In the TTS case, this is the peer who stands up the pipeline that runs a professional voice-clone fine-tune or an in-domain acoustic / lexical fine-tune, versus the `fine-tuning-engineer` / `nlp-engineer` peer who designs the fine-tune itself.
- **`agentic-safety-engineer` (L40)** — in the sibling [`agentic-safety-engineer-learning`](https://github.com/ai-engineering-curriculum) curriculum. <!-- needs-research: confirm canonical repo URL at author time --> Owns frontier safety methodology for agentic systems, including the voice-specific subset: synthetic-voice misuse detection at scale, adversarial red-teaming of TTS and cloning systems, provenance-watermark evaluation, deepfake-labelling policy, model-card and system-card authorship for voice products.

## The rule of thumb

**If the task is "pick a model, wire it into the pipeline, measure it, author the consent flow, and reason about its runtime cost," it is yours.** If the task is "stand up the training pipeline for a custom voice-clone", or "design the adversarial evaluation suite for synthetic-voice misuse", escalate.

## What belongs to `training-pipeline-engineer`

Chapter 5 drew the engineering boundary around the consent and provenance workflow for voice cloning. The *infrastructure* for producing a professional voice clone — especially one that goes deeper than a vendor's hosted PVC flow — belongs here.

### What that includes

- **Dataset ETL for a voice fine-tune.** Taking the studio recordings, segmenting, normalising, phoneme-aligning, filtering low-quality takes, deduplicating, stratifying across phonetic coverage, producing a reproducible training dataset.
- **Training run management.** GPU scheduling, checkpointing, hyperparameter sweeps over LoRA ranks or adapter configurations, experiment tracking (Weights & Biases, MLflow), reproducibility.
- **Continuous training.** The cadenced retraining that keeps a fine-tuned voice fresh as the base model evolves or as the domain drifts (new pronunciations, new product names, new accents). The observability loop you own (Module 112) is the trigger; the training infrastructure they own is the responder.
- **Artefact lifecycle.** Versioning the voice-ID × model-revision matrix, canary deployment of a new clone revision, rollback, serving-infrastructure handoff.
- **Self-hosted open-weights TTS fine-tunes.** When Chapter 6 pointed you at XTTS-v2 or StyleTTS 2 as your open-weights base, the fine-tune pipeline for a new speaker is a `training-pipeline-engineer` conversation, not yours.

### What stays yours after the fine-tune

Even once the fine-tune pipeline is standing:

- **The data contract.** Which audio, from which speaker, with what consent record, covering what phonetic diversity, with what acceptance criteria. Chapter 5's consent / provenance artefacts are *yours* to write. The training pipeline consumes that spec.
- **Integration and metric ownership.** You wire the fine-tuned voice into your TTS pipeline, you monitor TTFB and MOS on the new voice, you decide when to roll back.
- **Observability signals.** Module 112's voice-quality-drift alerts fire the retraining cadence.
- **The consent + provenance record.** The training-pipeline-engineer produces a model artefact; the consent that authorises it is yours to produce and bind to the artefact.
- **SSML / lexicon tuning** on top of the fine-tuned voice — remains pipeline engineering.

### Handoff contract

What goes over the wall to `training-pipeline-engineer` when you need a voice-clone fine-tune:

1. **Consent + provenance artefacts.** Per Chapter 5. No training without this.
2. **Reference audio corpus.** Raw files, segmented transcripts, phonetic-coverage audit, known-problem segments (bad quality, non-speech noise, two-speaker artefacts).
3. **Target voice profile.** The persona document — age coding, register, emotional range needed, locales, scripts the voice should handle well.
4. **Acceptance criteria.** Concrete. "MOS ≥ 4.1 on 30 held-out utterances evaluated by 15 listeners; MOS ≥ 4.0 on 20 drug-name utterances specifically; TTFB p95 ≤ 300 ms when served from the production inference endpoint." Not "it should sound good."
5. **Integration spec.** Which TTS serving stack it will deploy into, which voice-ID binding, which SSML tags must be honoured post-fine-tune.
6. **Rollback plan.** Which existing voice-ID the clone replaces or augments; what happens if MOS regresses post-deployment.

## What belongs to `agentic-safety-engineer`

Chapter 5 drew the consent-flow boundary. The *methodology* behind detecting and defending against synthetic-voice misuse at frontier scale belongs here.

### Synthetic-voice misuse detection at scale

Your product may be using TTS; the world is also using TTS, against your product. Scam calls using cloned voices; voice-authentication bypass; fraudulent consent records produced by cloning the consenting party. The surface:

- **Detection models.** Classifiers that tell "is this audio synthetic?" — ElevenLabs AI Speech Classifier, open-weights detectors from academia, Microsoft's Video Authenticator lineage for audio. Methodology — training data curation, generalisation across unseen vendors, adversarial robustness — is `agentic-safety-engineer` depth.
- **Watermark verification.** Resemble's PerTH, provider-specific watermark schemes, C2PA manifests. Verifying a mark is present, intact, and attributable. Designing the watermark policy (which vendors you trust, how you handle unverified audio) is `agentic-safety-engineer`.
- **Liveness challenges.** For voice-authentication paths, challenging the caller with prompts an AI replica would struggle to produce in the live moment (spontaneous content, interactive dialog). The design of these challenges and their anti-spoof evaluation is `agentic-safety-engineer`.

### Adversarial evaluation of your TTS pipeline

Every TTS system has misuse modes. What `agentic-safety-engineer` runs that you do not:

- **Red-teaming of the voice-cloning flow.** Can a user bypass your consent flow? What happens if a user uploads another user's voice with a forged consent form? What is the vendor's response?
- **Adversarial SSML / prompt injection against the TTS.** Can text embedded in LLM responses cause the TTS to say unauthorised things (hidden SSML tags, text-encoded jailbreaks)? The adversarial surface here mirrors prompt-injection against LLMs, with an audio-specific twist.
- **Deepfake-of-your-agent attacks.** An attacker clones your agent's branded voice and uses it elsewhere. Can you detect it? Can you file takedowns? The playbook here is `agentic-safety-engineer`'s.

### Policy authorship

The org-level policy documents that govern your voice product:

- **Voice-product system card.** What the product does, what it does not do, what it will refuse, what risks remain. The template, methodology, and review cadence are `agentic-safety-engineer` territory.
- **Vendor-selection policy at the org level.** Which cloning vendors are approved, which are blocklisted, which jurisdictions you clone in, which content categories you do not clone for. Legal and `agentic-safety-engineer` co-own this.
- **Incident response playbook for voice-specific incidents.** Clone misuse, undetected synthetic audio in a sensitive turn, a watermark-evasion exploit. The playbook owner is `agentic-safety-engineer`, with ops support from you.

### Handoff contract

What goes over the wall to `agentic-safety-engineer` when a voice-specific safety question lands on your desk:

1. **The threat model.** Who the adversary is, what they are trying to achieve, what they have access to. You bring the product shape; they bring the adversary analysis.
2. **The current pipeline surface.** Which vendors, which cloning modes are enabled, which consent flow is in production, which turns are gated and which are not.
3. **The specific incident or near-miss.** Logs, audio samples, adversary behaviour pattern.
4. **The constraints you cannot change.** Product requirements that bound the safety response space.

## The rule-of-thumb decision tree

For any TTS-related question that lands on your desk:

1. **Is it a pipeline-integration or configuration question?** (Vendor swap, streaming contract, SSML tag tuning, lexicon update, voice-ID binding, consent-flow mechanics.) → You. Chapters 1 – 5.
2. **Is it a measurement question with standard tools?** (TTFB on a workload, MOS A/B, naturalness stratified by register.) → You. Chapters 1 – 4.
3. **Is it an architecture-selection question for a self-hosted stack?** (Pick between XTTS and F5-TTS, pick between StyleTTS 2 and a vendor PVC, decide the acoustic-model family.) → You, often with `fine-tuning-engineer` or `nlp-engineer` for the fine-tune-design depth.
4. **Is it a "stand up a continuous voice-fine-tune pipeline" question?** → `training-pipeline-engineer`.
5. **Is it a "what is the adversarial surface" or "what is the frontier methodology" question?** (Synthetic-voice misuse detection at scale, deepfake-labelling policy, watermark evaluation, red-teaming methodology.) → `agentic-safety-engineer`.
6. **Is it a "which jurisdiction's regime applies to this specific deployment" question?** → Legal counsel, with you and `agentic-safety-engineer` providing the technical context and policy posture.

## The habits that make you a peer, not a customer

The same rule from the mod-103 boundary chapter applies here: the quality of the fine-tune, the honesty of the safety review, the stability of the training pipeline all depend on a tight handoff artefact. **Your job is to make the handoff legible.**

1. **Write the acceptance criteria before the work starts.** "MOS ≥ 4.1 on 30 held-out drug-name utterances, evaluated by 15 listeners, on the production-serving path at p95 TTFB ≤ 300 ms." Not "better than today."
2. **Provide reproducers.** A specific voice-ID, a specific input text, a specific expected output characteristic, with a concrete command or notebook that demonstrates the current problem.
3. **Own the integration side.** If the training pipeline produces a model with a different output sample rate, you adapt the player. If the safety pipeline wants a logging field you didn't emit, you emit it. The engineering burden of integrating their outputs is yours.
4. **Propagate consent and provenance into the handoff.** A voice-clone fine-tune request without the Chapter 5 consent + provenance artefacts should not be accepted by the training-pipeline peer; your job is to attach them by default.

## What *doesn't* belong to anyone on the ladder below you

The handoffs above are not "beneath you" tasks. They are tasks that have the depth of a different role. The L35 bar is to **compose their outputs** with the pipeline-engineering judgment Chapters 1 – 6 built, and to carry the integration surface and the production operational burden. In particular:

- **The consent flow is yours.** Legal counsel owns the legal review; `agentic-safety-engineer` owns the frontier threat model; the consent form, the audio consent recording, the provenance record are your product's artefact to produce and maintain.
- **Voice-identity UX is yours.** Chapter 4's voice-selection UX, consistency-across-turns rules, and catalogue management are not a design-team or a safety-team deliverable.
- **Day-to-day lexicon and SSML maintenance is yours.** The training pipeline retrains the base; the lexicon that steers runtime pronunciation remains your working file.
- **TTFB and barge-in budget enforcement is yours.** No other role will own this; the latency floor is a voice-pipeline-engineering discipline.

## Reading list to stay literate

Three artefacts worth reading top-to-bottom to pre-stage the safety conversation:

- **EU AI Act, articles on biometric systems and deepfake labelling.** The regulatory shape that most other jurisdictions will converge toward.
- **A system card for a frontier voice product** — e.g., OpenAI's `gpt-4o` system card section on audio output, ElevenLabs' responsible-AI documentation, Google's voice-content provenance materials. Pattern-match the shape of a system card so that when `agentic-safety-engineer` produces yours, you can review it meaningfully.
- **An academic survey on synthetic-speech detection.** Current references around the ASVspoof challenge series ([Yamagishi et al., multiple editions](https://www.asvspoof.org/)) and recent deepfake-audio detection benchmarks. <!-- needs-research: add the current canonical survey reference at author time -->

## Summary

- Deep training-infrastructure for voice-clone fine-tunes — ETL, distributed training, continuous training — belongs to `training-pipeline-engineer`.
- Frontier voice-safety methodology — synthetic-voice misuse detection, adversarial evaluation, provenance-watermark policy, voice-product system cards — belongs to `agentic-safety-engineer` (L40, one level above you).
- Your role is to compose their outputs with the pipeline-engineering judgment Chapters 1 – 6 built. The handoff contract — consent record, data spec, acceptance criteria, integration spec — is yours to write, and its quality bounds the output.
- The consent flow, the voice-identity UX, the SSML / lexicon maintenance, and the TTFB / barge-in budget stay with you after all the handoffs land.

This closes the content block for `mod-104`. Proceed to the exercises (TTFB-scored scorecard, streaming TTS with chunked playout, SSML + lexicon authoring, expressive vs. neutral A/B, voice-cloning consent and provenance walkthrough, architecture-tour write-up) and then on to [`mod-105-realtime-speech-to-speech`](../mod-105-realtime-speech-to-speech/README.md).
