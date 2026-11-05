# 05. HIPAA, PCI DSS v4.0, and Regulated-Vertical Content Obligations

## Why this chapter

Chapters 1–4 covered horizontal regulation. This chapter covers vertical regulation that applies the moment a voice agent touches a clinical workflow, a payment card, or a lending / insurance / investment conversation. Each vertical imposes two kinds of obligation on the voice engineer:

- **Infrastructure-shaped obligations** — who you can use as a sub-processor, how data is encrypted, how long it is retained.
- **Content-shaped obligations** — what the agent is *allowed to say*, which constraints land on the prompt, the tool interfaces, and the recording pipeline.

The third category — sector-specific disclosure (TILA/Reg Z for lending, Reg B for credit, FINRA Rule 2210 for securities, FTC CARS Rule for auto finance, HIPAA Privacy for clinical) — doesn't bind the voice agent's existence but does bind what it is permitted to speak. This chapter covers HIPAA and PCI in depth and surveys the content-obligations layer.

## HIPAA — what it actually requires

HIPAA (the [Health Insurance Portability and Accountability Act of 1996](https://www.hhs.gov/hipaa/)) governs **Protected Health Information (PHI)** held by **covered entities** (health plans, healthcare clearinghouses, most healthcare providers) and their **business associates** (vendors that handle PHI on behalf of covered entities). PHI includes any individually identifiable health information — spoken words included.

For a voice agent:

- A **clinical-scribing agent** that records patient-clinician encounters processes PHI from the moment the microphone turns on. It is a business associate of the clinician / health system.
- A **patient-facing voice agent** (appointment scheduling, triage, post-discharge follow-up) handles PHI the moment the caller names a symptom, a medication, a provider, or a date of service.

HIPAA has three operative rules that the voice engineer needs to track:

1. **Privacy Rule** ([45 CFR 164.500–534](https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164)) — permissible uses and disclosures, minimum-necessary principle, patient rights (access, amendment, accounting of disclosures).
2. **Security Rule** ([45 CFR 164.302–318](https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164/subpart-C)) — administrative, physical, and technical safeguards for electronic PHI (ePHI). The HHS proposed a significant update in December 2024 to tighten the technical safeguards; final rule pending at authoring time. <!-- needs-research: confirm whether the 2024 NPRM has been finalized and incorporate the final technical safeguards -->
3. **Breach Notification Rule** ([45 CFR 164.400–414](https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164/subpart-D)) — 60-day notification to subjects + HHS for breaches of unsecured PHI.

### The Business Associate Agreement (BAA)

If your voice-agent architecture handles PHI, every sub-processor that sees PHI must have a signed BAA. The common chain:

- **Model providers.** OpenAI, Anthropic, Google, AWS, Azure all now offer HIPAA-eligible services *with* a BAA — but only on specific SKUs (Azure OpenAI with the Enterprise agreement, OpenAI Enterprise, AWS Bedrock on HIPAA-eligible regions). Sign the BAA; verify the SKU.
- **STT / TTS providers.** AssemblyAI, Deepgram, Speechmatics, Azure AI Speech, Google Cloud Speech-to-Text — each has a HIPAA option; some have it as the default enterprise tier, some as an add-on.
- **Telephony / CPaaS.** Twilio, Vonage, Amazon Connect, Zoom Phone all sign BAAs on enterprise tiers.
- **Recording storage.** S3 + server-side encryption + bucket policy; or your provider's recording vault.

The BAA requires the processor to apply the Security Rule's safeguards, notify breaches, and allow termination for cause. Without a BAA in place, the processor cannot lawfully touch PHI.

### Minimum necessary

The Privacy Rule's minimum-necessary principle ([45 CFR 164.502(b)](https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164/subpart-E/section-164.502)) requires that uses and disclosures of PHI be limited to the minimum necessary for the purpose. For a voice agent, this translates to:

- **Prompt scoping.** The system prompt discloses the minimum PHI necessary for the task. Don't paste a full chart into a context window for a scheduling task.
- **Tool access scoping.** Tools that read PHI must gate by purpose: a scheduling tool returns only appointment data, not diagnosis.
- **Transcript redaction.** Downstream analytics on transcripts (QA, training) must redact identifiers unless the full transcript is strictly necessary and consent is layered.

### The Security Rule's technical safeguards

The Security Rule's technical safeguards (164.312) relevant to voice:

- **Access control (164.312(a))** — unique user identification, emergency access procedure, automatic logoff, encryption and decryption.
- **Audit controls (164.312(b))** — record and examine activity in systems that contain or use ePHI.
- **Integrity (164.312(c))** — mechanisms to authenticate ePHI (hashes, versioning).
- **Person or entity authentication (164.312(d))** — verify a person seeking access is who they claim.
- **Transmission security (164.312(e))** — integrity and encryption during transmission.

The 2024 NPRM would add specific requirements for MFA, encryption at rest, and vulnerability management; check the status before standing up a new HIPAA-scoped service. <!-- needs-research: confirm final form of 2024 Security Rule NPRM -->

### HHS guidance on AI-assisted scribing

HHS Office of Civil Rights has issued informal guidance on AI tools processing PHI (notably in the context of ambient-scribing products). Core expectations:

- The AI tool is a business associate; BAA required.
- Training on PHI requires patient authorization *or* a de-identification process meeting the Safe Harbor (164.514(b)(2)) or Expert Determination (164.514(b)(1)) standard.
- The de-identified output must not be re-identified upstream.

A clinical scribing agent that *does not* send transcripts back to the model provider for training (the majority of enterprise-tier contracts) is in scope. A consumer tier that reserves the right to train on recordings is a HIPAA red flag, regardless of the posted BAA.

## PCI DSS v4.0 — payments in a voice flow

[PCI DSS v4.0.1](https://www.pcisecuritystandards.org/document_library/?category=pcidss) is the current effective version (the v4.0.1 revision released June 2024). It governs the handling of cardholder data: Primary Account Number (PAN), cardholder name, expiration date, service code, and the sensitive-authentication-data category (CAV2/CVC2/CVV2/CID, full track data, PIN/PIN block).

The principle that drives every voice-agent design decision: **minimize scope.** Every system that processes, stores, or transmits cardholder data falls into the Cardholder Data Environment (CDE) and inherits the full 300-page control catalog. Every system that *doesn't* is out of scope. "Descoping" voice-channel card capture means the recording pipeline, the STT provider, the LLM, and the storage layer *never* touch the PAN.

### What to do during a payment moment

At the point the caller is about to speak a card number, the agent has three practical descoping options:

1. **Provider-native PCI-safe capture.** Twilio `<Pay>` ([twilio.com/docs/voice/api/payment-resource](https://www.twilio.com/docs/voice/api/payment-resource)), Amazon Connect secure input blocks, Genesys Secure Pause. The provider intercepts the DTMF or speech, communicates directly with a PCI-DSS-Level-1 payment tokenizer, returns a token; the recording and the agent never see the PAN.
2. **Pause recording + DTMF-only capture.** The agent instructs the caller to "enter your card number on your keypad"; the DTMF digits are stripped from the audio recording *and* from the live media stream before they reach STT. Twilio Media Streams has a configurable DTMF-masking option; verify it's on.
3. **Agent-side transfer to an IVR.** The voice agent hands the call off to a hardened IVR (or to a human in a PCI-scoped room) specifically for the payment segment, and resumes after.

Pure-speech capture ("say your card number") is almost never PCI-descoped — the audio, the STT output, and the LLM context all touch the PAN. Avoid it.

### The in-scope set — if you can't avoid it

If some portion of your flow *must* be in scope (e.g., call-centre human agents have to type the number from caller speech), the controls apply:

- **Requirement 3** — don't store sensitive authentication data post-authorization; if PAN is stored, it must be rendered unreadable (truncation, hashing, strong encryption with key management).
- **Requirement 4** — strong cryptography on public-network transmission.
- **Requirement 8** — MFA for all access to the CDE.
- **Requirement 10** — log all access; review daily.
- **Requirement 12** — formal information security policy; incident response.

PCI DSS v4.0 added explicit requirements around **authenticated scripts in e-commerce payment pages** (6.4.3), **MFA for all non-console admin access** (8.5), and **targeted risk analysis** (12.3.1). Voice-flow analogues: the IVR counts as the "payment page"; the recording-admin interface counts as non-console admin.

### "Pause recording" as a load-bearing primitive

From Chapter 2, the pause-recording primitive is shared between state recording consent and PCI DSS. Verify three things before a PCI audit:

1. The pause command actually pauses — not merely marks the recording for later redaction.
2. The STT stream is also gated — if the live transcript is piped to STT during the paused window, the PAN leaks there.
3. The LLM context does not receive the PAN — it should receive `{{card_token}}` only.

A cheap red-team: have an auditor speak a dummy PAN during a paused segment and grep the entire audit chain (recording, STT output, LLM context, provider logs) for the digits. Any hit is a scope failure.

## Content obligations by vertical

Separately from infrastructure, verticals impose content obligations on the agent's *speech*. A partial survey:

### Healthcare (HIPAA + FDA)

- **No diagnosis / treatment recommendation without the clinician in the loop.** A triage agent that says "you have strep throat, take amoxicillin" is practicing medicine. Appropriate language: "That sounds consistent with a sore-throat infection — let me book you an appointment with Dr. X." The line is "I will do X for you" vs. "You should do X."
- **FDA Software as a Medical Device (SaMD) classification** — if the agent's output drives a diagnostic or therapeutic decision, SaMD rules can attach. Most voice agents are below the SaMD threshold; some triage products are above.
- **42 CFR Part 2** — substance-use-disorder records have stricter protections than general PHI. Treat substance-use discussions with an additional consent layer.

### Payments and finance (TILA, Reg Z, Reg B, FDCPA)

- **Truth in Lending Act / Reg Z** ([12 CFR 1026](https://www.consumerfinance.gov/rules-policy/regulations/1026/)) — specific disclosures when discussing credit terms (APR, finance charge, amount financed, total of payments).
- **Equal Credit Opportunity Act / Reg B** ([12 CFR 1002](https://www.consumerfinance.gov/rules-policy/regulations/1002/)) — adverse-action notice for credit denials.
- **Fair Debt Collection Practices Act** ([15 U.S.C. § 1692](https://www.law.cornell.edu/uscode/text/15/chapter-41/subchapter-V)) — mini-Miranda ("this is an attempt to collect a debt..."), permitted call times, do-not-call-at-work on request. CFPB's Regulation F ([12 CFR 1006](https://www.consumerfinance.gov/rules-policy/regulations/1006/)) tightens the rules further.
- **Reg CC, GLBA** for bank disclosures.

For voice agents in these verticals, the model's system prompt should include the required disclosures as *mandatory openings or closings* — not optional. The disclosure text must be reviewed by counsel and kept as the agent's canonical opening, reinforced by output-validation that the agent uttered it before ending the call.

### Securities (FINRA)

- **FINRA Rule 2210** — communications with the public must be fair and balanced, not misleading, with specific content requirements. A voice agent discussing securities is a "communication" subject to these rules.
- **SEC Marketing Rule** ([17 CFR 275.206(4)-1](https://www.ecfr.gov/current/title-17/chapter-II/part-275)) — for investment advisers; prohibited claims, testimonials, and performance presentations.
- **Call recording retention** — books-and-records rules require retention for 3–6 years with WORM-equivalent protection.

### Automotive and consumer finance (FTC CARS Rule)

The FTC's [CARS Rule](https://www.ftc.gov/legal-library/browse/rules/combating-auto-retail-scams-trade-regulation-rule), effective 2024, prohibits certain misrepresentations and requires specific disclosures in auto sales and finance; a voice agent handling auto-sales calls must observe these.

### Education (FERPA)

The Family Educational Rights and Privacy Act ([20 U.S.C. § 1232g](https://www2.ed.gov/policy/gen/guid/fpco/ferpa/index.html)) governs student education records. A voice agent in an EdTech scenario handling student records inherits FERPA's parental-consent and record-amendment rights.

## Putting it together — a decision tree

```
Is the agent processing PHI?
  Yes → BAA with every processor; Security Rule technical safeguards; minimum-necessary prompt; no training on PHI.

Is the agent capturing payment card data?
  Yes → provider-native PCI-safe capture OR pause-recording with DTMF descope;
        verify pause stops STT and LLM context; otherwise full PCI DSS v4.0 controls.

Is the agent discussing credit, collection, securities, or regulated products?
  Yes → mandatory disclosures encoded as agent-prompt openings/closings; counsel review;
        output validation confirms disclosure was uttered; retention per vertical rules.

Is the agent speaking with a child?
  Yes → COPPA for US, GDPR Article 8 for EU, FERPA for education; parental-consent plumbing.
```

## Summary

- HIPAA: BAA with every processor, Security Rule technical safeguards, minimum-necessary prompts, no training on PHI without authorisation.
- PCI DSS v4.0.1: descope by design — provider-native capture (Twilio `<Pay>`), pause-recording with DTMF descope, or human-agent transfer. Pure-speech card capture is almost never descoped.
- The pause-recording primitive gates BOTH the state recording-consent requirement and PCI descope; verify it gates STT and LLM context too.
- Content obligations are vertical: TILA, Reg B, FDCPA, FINRA, CARS, FERPA each shape what the agent is allowed to say. Encode disclosures as mandatory agent openings/closings with output validation.
- Many sector rules carry books-and-records retention (3–7 years) with integrity controls; design storage for WORM-equivalent protection where needed.

Next chapter: on the attack side — anti-spoofing, deepfake detection, and provenance stamping.
