# 05. Voice Cloning: Consent, Provenance, and Knowing When to Say No

## Why this chapter

Modern TTS stacks can clone a human voice from thirty seconds of audio, zero-shot, no fine-tune. The technology is capability-complete; the industry around it is not. Voice cloning is the one TTS capability where **the engineering question is not the hardest question** — the consent flow, the provenance trail, the licensing surface, and the applicable law are harder, and the engineer who ships cloning without clearing them is the engineer whose product makes the news for the wrong reason.

This chapter is the voice-agent engineer's working depth on cloning: the two canonical cloning modes, the consent-flow shape, the provenance artefacts that a legal review will ask for, the US / UK / EU regimes you need to know by name, and the decision rule for saying no. It is not legal advice; it is the engineering surface that lets legal counsel do their job.

## IVC vs. PVC — the two canonical cloning modes

### Instant Voice Clone (IVC)

- **Input:** 30 seconds to ~3 minutes of reference audio from the target speaker.
- **Mechanism:** The TTS model ingests the reference at inference time (zero-shot) or produces a voice-embedding (one-shot fine-tune of a lightweight adapter). No gradient updates to the base model.
- **Output:** A voice-ID the client can reference in synthesis calls.
- **Quality:** Good-to-great; depending on vendor and reference-audio quality, listeners often cannot distinguish IVC output from the source speaker on short clips.
- **Available at:** ElevenLabs (Instant Voice Cloning), Resemble (Instant), Cartesia, PlayHT, Coqui XTTS v2 (open-weights, zero-shot).
- **Classic use case:** rapid prototyping, user-controlled voice-cloning in consumer apps, voice preservation for medical voice loss (where the user is the subject and consent is self-evident).

### Professional Voice Clone (PVC)

- **Input:** Studio-quality recordings, 30 minutes to several hours, often scripted to cover phonetic diversity. Plus a signed consent / licence agreement.
- **Mechanism:** A deeper fine-tune — sometimes a full voice-adapter, sometimes a model-level fine-tune on the vendor's base. Usually gated behind a vendor review process that verifies identity and consent documentation.
- **Output:** A voice-ID the vendor binds to a customer account, often with licence terms attached (per-use royalty, geographic restrictions, revocation clauses).
- **Quality:** The best available. Preserves character across emotional registers, long passages, less-common phonemes.
- **Available at:** ElevenLabs (Professional Voice Cloning), Resemble (Professional), Microsoft Azure (Custom Neural Voice — with a notably stricter vetting process), AWS Polly (Brand Voice — gated enterprise-only).
- **Classic use case:** branded enterprise voice, licensed celebrity / voice-actor collaboration, long-form narration.

### The decision rule

Use **IVC** when:

- The subject is the user themselves and consent is self-evident (voice preservation, personalisation).
- The subject is an **employee / contractor** who has explicitly consented in writing for this specific use, duration, and geography.
- The subject is a **licensed voice actor** who has agreed a short-form clone for a specific product scope.
- You can accept the vendor's weaker vetting (most IVC flows do not verify that the submitted audio is of the person clicking the consent box).

Use **PVC** when:

- The use is commercial and ongoing — a brand voice, a flagship product persona.
- The subject is a known public figure, voice actor, or employee whose identity the vendor should verify.
- You need the quality floor PVC offers and the licence terms that accompany it.
- Compliance requires an auditable consent + licensing trail that the vendor will provide.

**Do not clone** when:

- The subject has not clearly consented, including deceased people and "public figures whose voices are on YouTube."
- The subject has consented for one use and the deployment is a different use.
- The jurisdiction's legal framework (below) prohibits the deployment regardless of consent.
- The product is designed to deceive — impersonation, scam calls, political manipulation, adult content without identity-release documentation.

The L35 engineer's job is not to write the policy; it is to **refuse to ship the pipeline that would violate it**. More on that stance at the end.

## The consent flow

The vendor consent flow and the compliance consent flow are two different things, and the vendor's does not substitute for yours. Design yours like this.

### The consent record — what has to be in it

1. **Identity of the speaker.** Full legal name, contact information, government-ID verification (for PVC — some vendors require it; where they don't, your product should).
2. **Identity of the requesting party.** Who is asking to clone. Who at the company owns the voice-identity asset.
3. **Scope of use.** What the voice will say — product category, example scripts, prohibited content. Where it will say it — geographies, platforms. For how long — term and renewal terms. In what languages — the IVC / PVC can usually generate languages the speaker never spoke; the consent should name whether that is in or out of scope.
4. **Explicit statement of what cloning is.** "You are agreeing that an AI model will be trained / conditioned on your recorded voice and used to generate synthetic speech in your voice saying words you did not say, controlled by [company], for the purposes of [X]." Users click past "terms"; this requires a specific affirmative.
5. **Revocation right and mechanism.** The speaker can revoke consent. What does revocation cause? (Deletion of voice-ID, cessation of synthesis, takedown of in-flight audio, model retraining if applicable.) How long does revocation take to propagate?
6. **Jurisdiction and governing law.** The consent is enforced under some jurisdiction's law; name it.
7. **Signature and witness.** Electronic signature is fine in most places; pair it with an audio consent recording (the speaker reads a standard consent script aloud) because audio of the person consenting is harder to repudiate than a clicked checkbox.
8. **Timestamp and non-repudiable storage.** Hash the record, store it with the audio in append-only storage, keep an audit log.

### The audio consent recording

The speaker reads a standardised consent script aloud, and you store that audio alongside the written consent:

> I, [full name], on [date], give [company name] permission to create a synthetic model of my voice using the recordings I have provided. I understand this model will be used to generate speech in my voice for the purposes of [specific use case]. I have the right to revoke this consent at any time, after which [company name] agrees to cease synthesis within [time window] and delete the model from active storage within [time window]. My signed written consent sets out the full terms.

Several reasons to require this on top of a written form:

- The vendor's consent flow may not require it; yours should.
- A verbatim audio record is forensically stronger than a form.
- It is the first place a bad-faith request fails (a sock-puppet account cannot read the script in the voice they claim to represent).

### The vendor's piece

For PVC, the vendor will typically:

- Require their own consent / release form, on top of yours.
- Verify identity (government ID, selfie, sometimes a live video session).
- Ingest the audio in a monitored environment.
- Attach licence terms to the resulting voice-ID — per-month subscription, per-use royalty, geographic limits.

For IVC, vendors traditionally rely on a click-through "I have the right to clone this voice" declaration. **This is not sufficient for commercial deployment of a non-self voice.** Your consent flow has to do the work the vendor skipped.

## Provenance

Even with a clean consent trail, you have to be able to prove at audit time:

- Which voice-ID was used for which synthesis call.
- Which consent record (and audio recording) authorises that voice-ID.
- When that consent was granted, when it expires, whether it has been revoked.
- Which model revision produced the audio (the same voice-ID on an updated base can sound noticeably different).

The provenance artefacts that matter:

1. **Voice-ID → consent record mapping.** A table in your voice-identity service that answers "for voice-ID X, where is the consent?" in one query.
2. **Synthesis-call logs.** Every TTS call records voice-ID, model revision, timestamp, text, and the output audio hash. Retention set by compliance (often 1–7 years depending on jurisdiction and content type).
3. **Audio watermarking.** Several vendors watermark generated audio — Resemble's provenance marker (PerTH), ElevenLabs' AI Speech Classifier for detection, Microsoft's content-provenance initiative via C2PA. Watermarks are not forensic-grade (they can be stripped) but they are useful evidence that the output came from a specific vendor, and they help detect misuse post-leak. Enable where available; verify the specification at author time because this space moves.
4. **C2PA metadata** where the output is distributed as a file. The [Content Authenticity Initiative (C2PA)](https://c2pa.org/) standard provides a signed manifest attached to media; use it where your downstream tooling supports it.
5. **A takedown workflow.** If a voice is misused — the consented speaker says "someone is using my clone to say things I did not consent to" — you need a playbook: identify the voice-ID, disable it in the TTS service, propagate revocation to vendors, notify downstream consumers, retain evidence. Write this before you ship; do not improvise it.

### A worked provenance record (shape)

```json
{
  "voice_id": "rook-prod-brand-01",
  "clone_mode": "pvc",
  "vendor": "elevenlabs",
  "vendor_voice_id": "ElevenLabs_abc123",
  "model_revision": "eleven_multilingual_v2_r4",
  "speaker": {
    "legal_name": "[REDACTED]",
    "contact_hash": "...",
    "identity_verified_at": "2026-01-14T00:00:00Z",
    "identity_verification_method": "vendor+internal-video-call"
  },
  "consent": {
    "written_signed_at": "2026-01-14T15:22:00Z",
    "written_consent_hash": "sha256:...",
    "audio_consent_s3_uri": "s3://rook-consent-vault/voice/2026/01/...wav",
    "audio_consent_hash": "sha256:...",
    "scope_use_cases": ["customer-support", "ivr"],
    "scope_prohibited": ["political-messaging", "adult-content", "paid-promotions-outside-rook-products"],
    "scope_geographies": ["US", "CA"],
    "scope_languages": ["en-US", "en-CA"],
    "term_start": "2026-01-14",
    "term_end": "2028-01-14",
    "revocation": {
      "revocable_on_notice": true,
      "cessation_sla_hours": 48,
      "model_deletion_sla_hours": 168
    },
    "governing_law": "State of Illinois, USA"
  },
  "watermark": {
    "scheme": "perth",
    "enabled": true
  }
}
```

The point is that this exists and is queryable. If your legal counsel cannot run one query and produce the consent audio for voice-ID X, your provenance story will not survive an audit.

## The legal regimes you need to know by name

Not an exhaustive survey; the regimes most likely to apply to a voice-agent deployment in 2026. **This is not legal advice** — reach for counsel before you ship — but you should be able to name them at the standup.

### United States

- **Illinois Biometric Information Privacy Act (BIPA).** [740 ILCS 14/](https://www.ilga.gov/legislation/ilcs/ilcs3.asp?ActID=3004). Treats voice as biometric information in Illinois. Requires informed written consent before collection, a public retention policy, prohibits sale, imposes statutory damages per violation. Multi-million-dollar class actions have been filed and settled under BIPA for voice collection by call centres, smart speakers, and voice-authentication products. If any part of your deployment touches an Illinois resident's voice, this applies.
- **Texas CUBI** (Tex. Bus. & Com. Code § 503.001) and Washington state biometric statutes are similar in spirit; no private right of action in Texas (enforcement is by the Attorney General), which changes the litigation posture but not the engineering burden.
- **NO FAKES Act** — the Nurture Originals, Foster Art, and Keep Entertainment Safe Act — a bipartisan federal bill creating a federal right of publicity over voice and likeness, targeting unauthorised AI-generated voice and video replicas. <!-- needs-research: confirm current enactment status, effective date, and takedown-process specifics at author time — the bill text has moved in session; verify against congress.gov before relying on specifics --> As of this writing it is in federal legislative process; its practical effect is to add a federal cause of action on top of state right-of-publicity law (California Civil Code § 3344, New York Civil Rights Law §§ 50–51, Tennessee ELVIS Act). You should assume *some* federal voice-and-likeness regime will exist in your deployment horizon and build consent flows that would satisfy it.
- **FTC guidance on AI impersonation.** The FTC's impersonation rules apply to AI-generated voice when used in deceptive commerce. Scammers using cloned voices are an active FTC enforcement area.
- **Tennessee ELVIS Act (Ensuring Likeness, Voice, and Image Security Act, 2024).** State-level voice right of publicity; a template other states are copying.

### United Kingdom

- **UK Data (Use and Access) Act (DUAA 2025).** <!-- needs-research: confirm the exact title, enactment date, and scope at author time — this is the current UK data-protection reform package; verify text and commencement against legislation.gov.uk --> Updates the UK GDPR regime for AI and data-use cases. Treat voice recordings as personal data (and biometric data when used for identification) under UK GDPR as updated; special category data if used for authentication. Lawful basis for cloning is almost always consent; legitimate interest is a weaker footing for biometric processing.
- **UK GDPR + ICO guidance.** Even without a specific cloning statute, the general UK GDPR requires lawful basis, purpose limitation, data-minimisation, and the data subject's rights (access, erasure). A cloned voice model trained on a person's voice is derived personal data; erasure requests may require model deletion or retraining.

### European Union

- **EU AI Act.** Articles on biometric systems and on deepfake-labelling obligations apply directly to voice synthesis of real people. Deepfakes (content generated or manipulated to resemble a real person) must be labelled as artificially generated. Voice cloning falls inside this.
- **GDPR.** As with UK, voice is personal data; voice used for identification is biometric / special-category data under Article 9. Consent must meet the GDPR standard (freely given, specific, informed, unambiguous, written if the controller is relying on consent for special-category processing).

### Across jurisdictions

- **Right of publicity / personality rights.** Varies by jurisdiction; often runs post-mortem. Cloning a deceased person's voice without their estate's authorisation is a live liability in multiple jurisdictions.
- **Platform policy.** If you distribute via an app store, a telephony carrier, or a cloud compute provider, their TOS will have voice-cloning provisions on top of the law.

### The engineering implication

You will not clear all this by reading statutes. You clear it by:

1. Building the consent flow and provenance artefacts above.
2. Making legal review a required step before any non-self voice clone ships to production.
3. Documenting the jurisdictional scope of every voice-ID so counsel can tell you where it is deployable.
4. Having a kill switch per voice-ID.

## When to say no

Three scenarios worth rehearsing.

- **"Clone this celebrity for a parody."** The user is well-intentioned, the vendor's IVC flow allows it, the audio is public. Say no. US right-of-publicity law, the NO FAKES Act in its draft forms, and the EU AI Act all treat this as a specific class of risk, parody fair-use arguments notwithstanding — and your product is not the test case you want to make.
- **"Clone the CEO's voice for the earnings-call voicemail."** Internal, employee, no commercial harm. Still require the full consent flow — written + audio consent recording + scope document. The CEO is an employee with a complicated power dynamic over anyone handling the request; the consent flow protects both of you.
- **"We'll let users clone any voice they upload."** Consumer-facing IVC with a click-through. Say no to the open-ended version. Say yes to self-only cloning verified by liveness challenge (the user reads a prompt that their claimed source audio could not have pre-recorded, and the match is verified). Say yes to employee / contractor / licensed-actor cloning via admin flow with the full consent record. Do not ship an open "upload anyone's voice" surface.

The L35 bar is: **your engineering escalates the request to legal, in time, with the right artefacts, with a recommendation.** The L40+ `agentic-safety-engineer` owns the frontier methodology on this — detecting synthetic-voice misuse at scale, designing adversarial test suites, voicing the vendor-selection policy at the org level. You compose their outputs; you do not reinvent them. See [Chapter 7](07-boundary-to-training-pipeline-and-agentic-safety.md) for the boundary.

## Common anti-patterns

- **The vendor's click-through is the consent record.** It is not. The vendor's click-through protects the vendor; your consent record protects you and your speaker.
- **Cloning without an audit trail.** You cannot answer "which consent authorises this voice-ID" six months in. Build the mapping from day one.
- **No revocation mechanism.** The speaker revokes; your system has no story. Design the kill switch before the first clone.
- **Watermark-only provenance.** Watermarks are evidence, not authorisation. A watermark on content that was generated without consent is still non-consensual content.
- **Treating "public figures" as fair game.** Public figures retain voice-identity rights. "Their voice is on the internet" is not consent.
- **Shipping IVC in a consumer app without liveness.** Any user can upload a 30-second clip of anyone. The product is now a free impersonation tool.
- **Legal review as a formality.** If legal is CCed on the PR, you did not do a legal review. Legal review includes reading the consent form, listening to the audio consent recording, confirming jurisdiction, and sign-off in writing.

## Summary

- Two cloning modes: IVC for low-friction, user-or-employee self-cloning with a strong consent flow; PVC for brand / commercial voice with vendor identity verification and licence terms.
- The consent flow is yours to design, not the vendor's. Written + audio-recorded + scoped + revocable + jurisdictionally-grounded.
- Provenance is a queryable artefact: voice-ID → consent record, synthesis logs, watermarks where available, C2PA where supported, a documented takedown workflow.
- Know the regimes by name: BIPA, NO FAKES Act (draft / emerging), state right-of-publicity statutes, UK DUAA + UK GDPR, EU AI Act, GDPR.
- Saying no is part of the job. The L35 engineer ships what legal has signed off on, with the provenance artefacts that make the signoff meaningful.

Next chapter: modern generative TTS architectures at a working level — VITS, VoiceBox, XTTS, Bark — enough to pick, not to re-implement.
