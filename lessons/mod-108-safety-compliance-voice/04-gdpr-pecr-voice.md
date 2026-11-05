# 04. GDPR, UK PECR, and Ofcom for Voice

## Why this chapter

The AI Act governs what the system *is allowed to do*. The GDPR governs what you *do with the data the system produces or consumes*. For a voice agent, that data is the raw audio, the derived transcript, any voice embedding or speaker ID, and every downstream artefact — tool-call logs, CRM updates, outbound emails. Every one of those is "personal data" under GDPR, and voice-plus-biometric-features is "special category" under Article 9.

Separately, in the UK (and, in a similar shape, across the EU member states where ePrivacy Directive transpositions govern unsolicited marketing calls), the **Privacy and Electronic Communications Regulations 2003 (PECR)** add a layer *on top of* UK GDPR for electronic marketing and automated calls. Ofcom enforces the silent-and-abandoned-call rules; the ICO enforces PECR consent.

This chapter gives the engineer-shaped read. GDPR deep interpretation belongs to a DPO + counsel; your job is to make sure the data flows, retention clocks, and DSAR primitives are in place.

## GDPR Article 6 — lawful basis

[Article 6](https://gdpr-info.eu/art-6-gdpr/) gives six lawful bases for processing. For a voice agent, the realistically-available ones are:

- **6(1)(a) Consent** — the subject opts in to the call being recorded and to the voice being processed.
- **6(1)(b) Contract** — processing is necessary to perform a contract the subject requested (e.g., the subject calls support, consents implicitly to the support call being handled).
- **6(1)(c) Legal obligation** — processing is required by law (e.g., financial record-keeping).
- **6(1)(f) Legitimate interests** — the controller's legitimate interests, balanced against the subject's rights. Historically the busiest basis, and the one that fails most often under scrutiny.

For *inbound* support calls, (b) and (f) are typical. For *outbound* marketing calls, consent (a) is effectively required — and under PECR (below), *explicit* consent is required for automated calls regardless.

Pick one lawful basis per processing purpose; document it; don't switch bases mid-flow without re-grounding.

## Article 7 — conditions of consent

[Article 7](https://gdpr-info.eu/art-7-gdpr/) requires that consent be freely given, specific, informed, and unambiguous (Article 4(11) definition, reiterated in Article 7's conditions). Key engineer-shaped mechanics:

- **Demonstrability** — the controller must be able to demonstrate that consent was given (Article 7(1)). Immutable, timestamped consent records.
- **Separability** — consent for each processing purpose must be distinguishable from other matters (Article 7(2)). One checkbox for "I agree to the TOS *and* authorise AI voice calls" fails.
- **Withdrawability** — it must be as easy to withdraw as to give (Article 7(3)). A one-click in-UI toggle *and* a spoken "stop" in the call must both work.
- **Not conditional on service** — consent is not freely given if accepting consent is required to use the service (Article 7(4)), unless the consent is strictly necessary to provide that specific service.

## Article 9 — special categories (biometric and health)

[Article 9](https://gdpr-info.eu/art-9-gdpr/) prohibits processing of special categories of personal data unless one of the enumerated conditions applies. Two voice-relevant categories:

- **Biometric data for the purpose of uniquely identifying a natural person** (Article 4(14) definition). *The key qualifier is "for the purpose of uniquely identifying."* A voice recording per se is not biometric data under Article 9 unless it is being used — or the system is designed to use it — for unique identification. A voiceprint for authentication *is* Article 9 data. A diarized audio file where the embeddings are discarded at hangup *arguably is not*.
- **Data concerning health** — PHI in a clinical-scribing voice flow is Article 9 data regardless of the biometric question.

Conditions for processing Article 9 data most relevant to voice:

- **9(2)(a)** — explicit consent (higher bar than Article 6 consent; the consent must specifically reference the special-category processing).
- **9(2)(h)** — processing for the purposes of preventive or occupational medicine, diagnosis, treatment, management of health systems, under a medical professional's responsibility.

Operationally, voice biometric authentication at scale in the EU almost always rides on 9(2)(a) explicit consent. The enrollment flow therefore needs a separately-recorded, demonstrable, specific-to-biometrics consent act — not an inherited TOS acceptance.

## Article 22 — automated decision-making

[Article 22](https://gdpr-info.eu/art-22-gdpr/) gives the data subject the right not to be subject to decisions based *solely* on automated processing which produce legal or similarly significant effects. Exceptions apply (contract necessity, EU/member-state law authorisation, explicit consent).

For voice agents: a fully autonomous agent that denies a loan, cancels an insurance policy, or triggers a debt-collection action on its own call is a 22-event. The engineering mitigation is the *human-in-the-loop breakpoint* — the agent proposes the action, a human reviews within SLA, the human confirms or overrides. Logs of the human review satisfy both Article 22 and the AI Act's Article 14 human-oversight obligation.

The ECJ's *SCHUFA* decision (C-634/21, December 2023) read "solely" broadly: a scoring step that materially drives a human decision can still be Article 22 if the human's role is rubber-stamping. Design the human review as a real check, not a formality.

## Article 32 — security of processing

[Article 32](https://gdpr-info.eu/art-32-gdpr/) requires technical and organisational measures appropriate to the risk. For voice:

- **Pseudonymisation and encryption** — audio at rest encrypted; transcripts with PII pseudonymised in analytical stores; voice embeddings in a separate key-protected store.
- **Confidentiality, integrity, availability, resilience** — the voice-agent stack has single points of failure the regulation would care about; document them.
- **Regular testing** — DR, pentesting; see Chapter 7 for the voice-adversarial threat model that feeds this.
- **Processor contracts** — every third party (CPaaS, STT, TTS, LLM) is a processor; sign a GDPR-compliant DPA (Article 28). Verify sub-processor transparency clauses; voice is a chain of five or six processors.

## Article 35 — DPIA

[Article 35](https://gdpr-info.eu/art-35-gdpr/) requires a Data Protection Impact Assessment where processing is "likely to result in a high risk." A production voice agent almost always triggers at least one of the Article 35(3) explicit triggers:

- **(a)** systematic and extensive evaluation based on automated processing — most voice agents do routing or triage decisions.
- **(b)** large-scale processing of special categories (biometric, health).
- **(c)** large-scale systematic monitoring of a publicly accessible area — not usually voice, but the pattern.

The EDPB's "when a DPIA is required" guidelines list nine criteria; two matter: evaluation/scoring and innovative technology. A voice-LLM-agent system hits both. Treat a DPIA as mandatory and author it before launch.

A DPIA for a voice agent should document:

1. Systematic description of the processing operations and purposes (the voice-agent architecture).
2. Assessment of necessity and proportionality (why voice? why this scope?).
3. Assessment of risks to data subjects (unauthorised access, deepfake impersonation, model inversion of voiceprints, downstream LLM data leakage).
4. Measures to address those risks (encryption, access control, watermarking, audit trail).

Many national DPAs publish templates; the UK ICO's [DPIA template](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/accountability-and-governance/data-protection-impact-assessments-dpias/) is a usable starting point for both UK and EU purposes.

## Cross-border transfers

Voice audio frequently moves from the EU to a non-EU model provider (OpenAI US, Google US, Anthropic US, AWS US). GDPR Articles 44–50 govern. The practical options:

- **Adequacy decision** — the EU-US Data Privacy Framework ([Commission Implementing Decision 2023/1795](https://eur-lex.europa.eu/eli/dec_impl/2023/1795/oj)) restores adequacy for US transfers *to DPF-certified recipients*. Check the [Data Privacy Framework List](https://www.dataprivacyframework.gov/list) for each sub-processor.
- **Standard Contractual Clauses** (SCCs) — the June 2021 clauses.
- **Explicit consent** for the specific transfer (Article 49(1)(a)) — not a scalable basis for recurring voice traffic.

If any sub-processor is not DPF-certified and SCCs apply, you must also run a **Transfer Impact Assessment** per *Schrems II* and the EDPB's recommendations. Audit this at launch and re-check every six months — DPF certifications lapse.

Region-pinning the audio helps: Whisper, AssemblyAI, Deepgram, OpenAI, Azure OpenAI Service, and Google Cloud all now offer EU-region endpoints for voice. Prefer those to avoid the transfer apparatus entirely.

## UK PECR — unsolicited marketing calls

The UK's [Privacy and Electronic Communications Regulations 2003](https://www.legislation.gov.uk/uksi/2003/2426/contents) govern electronic marketing. The ICO enforces. Key regulations for voice:

- **Regulation 19 — Automated marketing calls (recorded messages).** Prohibits automated calls for direct-marketing purposes unless the recipient has given *prior consent* that specifically covers the type of marketing and the caller. Consent must be "notified" — the caller must be identifiable.
- **Regulation 21 — Unsolicited calls for direct-marketing.** Prohibits calls where the subscriber has told the caller not to call (internal opt-out), *or* has registered with the Telephone Preference Service (TPS) and the caller has no overriding prior consent.
- **Regulation 21A — Calling-line identification display.** Any caller must present a valid, dialable number.

For a voice agent dialling into the UK:

- The FCC's 2024 AI-voice ruling is not UK law, but Regulation 19 already caught AI-driven automated calls. The ICO [guidance updated for AI](https://ico.org.uk/for-organisations/direct-marketing-and-privacy-and-electronic-communications/) confirms the same reading.
- Register the calling numbers with the TPS/CTPS before dialing and check the lists at call time.
- The ICO has fined AI-driven call campaigns at seven-figure levels (Cityline Communications, DM Design Bedrooms, among others). <!-- needs-research: cite current ICO enforcement register at time of use for recent AI-call actions -->

## Ofcom silent and abandoned call rules

Ofcom (the UK communications regulator) enforces its [Persistent Misuse policy](https://www.ofcom.org.uk/siteassets/resources/documents/consultations/category-2-1-less-than-6-weeks/104-56-days/misuse-statement/persistent-misuse-of-electronic-networks.pdf) for automated-dialling. Key numbers:

- **Abandonment rate ≤ 3%** per campaign per 24-hour period (same shape as the FCC rule).
- **"Silent call" ban** — if an automated dialler connects faster than the called party picks up, and no agent/system speaks immediately, this counts as a silent call and is directly actionable.
- **Information message within 2 seconds** of connection on any abandoned call: the caller's name, the purpose, a dedicated non-premium return number, and the information that the call will not be followed up.
- **Internal DNC** — a subscriber who asks not to be called must be suppressed for at least 30 days (longer per internal policy).
- **Call-recording for consent** — documented evidence of consent is expected.

Ofcom can impose penalties up to £2,000,000. The engineer-shaped translation is near-identical to Chapter 1's dialler gate: abandonment-rate instrumentation, 2-second time-to-first-utterance, immediate identification message, number registration.

## Data-subject rights — the DSAR flow for voice

GDPR Chapter III (Articles 12–23) gives subjects rights of access, rectification, erasure, restriction, portability, and objection. For a voice-agent system, the DSAR plumbing needs to resolve:

1. **Subject identification** — given a person's assertion of identity, find all their call records. Store a stable `subject_id` across calls and index by (phone number, email, account ID).
2. **Audio retrieval** — return the actual recordings on Article 15 access requests. Keep them for the retention window, and in a format the subject can play (MP3/Opus containerised).
3. **Transcript retrieval** — return every downstream transcript, including the LLM tool-call audit log.
4. **Voiceprint retrieval + erasure** — if a voiceprint exists, return the derived features (or the fact of their existence) on access, and delete on erasure. Deletion must cascade to any model fine-tune that used the voiceprint.
5. **Pseudonymised analytics erasure** — if PII was pseudonymised, document the mapping and support subject-request-driven deletion of the mapping.
6. **Retention clocks** — each artefact has a retention schedule; on request, destroy early; on expiry, destroy on schedule.

Most teams under-design here. The DSAR primitive is a small service: one endpoint `/dsar/subject/{id}` that returns a zip of audio + transcripts + metadata + voiceprint status, a second endpoint `/dsar/subject/{id}/erase` that fans out to every storage system. The first time an EU consumer sends a subject access request, you want this to already exist.

## Children — Article 8 and the age question

If your voice agent could speak with a child under 16 (13 in some member states that have opted lower), Article 8 requires parental consent for processing based on consent. Clinical paediatric scribing, education tooling, and consumer-smart-speaker integrations are all exposed.

Verifying age by voice alone is not reliable and can itself be a biometric-categorisation issue (Article 5(1)(g) territory). Design flows that *assume* a caller may be a child and refuse to process accordingly — e.g., require account authentication for sensitive flows, with the account carrying the age attestation.

## Summary

- Lawful basis (Article 6): usually contract or consent for voice agents; legitimate interests is high-risk-of-challenge.
- Consent (Article 7): freely given, specific, informed, unambiguous, demonstrable, separable, as easy to withdraw as to give.
- Biometric voice data for identification → Article 9 special-category → needs explicit consent or another 9(2) condition.
- Article 22 → human-in-the-loop for high-impact autonomous decisions.
- Article 32 → encryption, pseudonymisation, resilience, tested security.
- Article 35 → a DPIA is effectively mandatory for a production voice agent.
- UK PECR Regulation 19 → automated calls require prior consent, with identifiable caller.
- Ofcom silent/abandoned rules → 3% abandon cap, 2-second information message, number registration.
- DSAR primitive: one API, one erase endpoint, retention clocks on every artefact.

Next chapter: US sector-specific law — HIPAA for healthcare voice, PCI DSS v4.0 for payments inside a voice flow.
