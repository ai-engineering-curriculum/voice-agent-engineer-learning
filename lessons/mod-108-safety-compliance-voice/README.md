# mod-108-safety-compliance-voice: Voice-Channel Safety, Compliance, and Provenance

**Estimated effort:** 12 hours · **Role:** Voice & Multimodal Agent Engineer · **Level:** L35

## What this module is for

A production voice agent is a regulated artefact the moment it touches a telephone number, a microphone, or a user's biometric signal. The TCPA cares about who called whom and under what consent; the FCC's February 2024 ruling folded AI-generated voices into the "artificial" bucket overnight; the EU AI Act prohibits whole classes of voice inference and demands deepfake disclosure under Article 50; GDPR treats a voiceprint as special-category biometric data; HIPAA pulls clinical-scribing agents into a Business Associate relationship; PCI DSS wants the agent's recording *paused* when a card number is spoken. On the adversarial side, a voice agent inherits every LLM-era attack (prompt injection, tool abuse, sensitive-information disclosure) plus audio-domain attacks that don't exist anywhere else in the stack (DolphinAttack ultrasonics, CommanderSong audio-hidden commands, voice-clone impersonation of an executive on a SIP call).

This module gives you the **engineer-shaped** reading of those regulations and threats: enough to design a defensible outbound-call consent flow, map a recording feature against state consent law, mark up a TTS output with C2PA provenance, wire ASVspoof-era countermeasures into the inbound-call verification path, and threat-model the agent against OWASP + MITRE ATLAS. Deep legal interpretation belongs to counsel; policy-shop work belongs to `ai-risk-engineer`; platform MLSec belongs to `ai-infra-security` and `security-learning`; frontier dangerous-capability eval belongs to `agentic-safety-engineer`. Your job is to know where the boundaries are and to build the mitigations that live in the voice stack itself.

## Learning objectives

- Author a defensible outbound-voice-agent consent flow under US TCPA + the FCC February 2024 ruling classifying AI-generated voice as 'artificial' — express prior written consent, automated-dialer safeguards, DNC + internal-DNC + reassigned-numbers handling, call-labelling / caller-ID.
- Map US state call-recording consent laws — one-party (federal baseline + most states) vs. two-party / all-party states (California, Florida, Illinois, Massachusetts, Montana, Nevada, New Hampshire, Pennsylvania, Washington) — and the Illinois BIPA obligation around voiceprints as biometric data (informed written consent, retention schedule, no sale).
- Interpret the EU AI Act voice-specific obligations — Article 5(1)(f) prohibition on emotion-inference AI in workplace / education, Article 5(1)(g) prohibition on biometric-categorisation inferring sensitive attributes, Article 50 transparency obligation for synthetic audio (deepfake) and chatbot disclosure, Annex III high-risk categories for employment / credit / education / law-enforcement voice systems.
- Apply GDPR to voice — voice as biometric and often special-category data; Article 6 lawful basis, Article 7 consent, Article 9 special-category conditions, Article 22 automated-decision-making, Article 32 security of processing, Article 35 DPIA when deploying a voice agent at scale; UK PECR + Ofcom guidance where relevant.
- Handle HIPAA (clinical-scribing + patient-facing voice agents), PCI DSS v4.0 (payment steps in a voice flow — descoping with 'pause recording' or DTMF capture), and the content obligations of regulated verticals.
- Deploy anti-spoofing + deepfake-detection — ASVspoof methodology, speaker-verification (VoxCeleb-trained models), audio watermarking (Meta AudioSeal, SilentCipher), C2PA Content Credentials for synthetic audio, Resemble Detect for deepfake detection; chain the mitigation into the agent's incoming-call verification path and the outgoing-TTS provenance stamp.
- Threat-model the voice agent against OWASP LLM Top 10 + OWASP Agentic AI Top 10 + MITRE ATLAS — voice-channel manifestations: prompt injection via transcribed content (hostile ASR output), audio-domain adversarial commands (DolphinAttack-style ultrasonic, CommanderSong-style hidden in music), tool abuse mid-turn in a telephony session, non-human identity in SIP, model theft of a cloned voice, sensitive-information disclosure in TTS.
- Cite the boundary to `ai-risk-engineer` (Governance family, L25) on the risk-engineering slice, to `security-learning` / `ai-infra-security` (L35) on platform-scale MLSec, and to `agentic-safety-engineer` (L40) on frontier dangerous-capability eval.

## Lecture chapters

1. [01 — TCPA, the FCC's AI-Voice Ruling, and Outbound-Call Consent](01-tcpa-fcc-outbound-consent.md)
2. [02 — State Call-Recording Consent and the Illinois BIPA Voiceprint Rule](02-state-recording-consent-and-bipa.md)
3. [03 — The EU AI Act for Voice Systems](03-eu-ai-act-voice.md)
4. [04 — GDPR, UK PECR, and Ofcom for Voice](04-gdpr-pecr-voice.md)
5. [05 — HIPAA, PCI DSS v4.0, and Regulated-Vertical Content Obligations](05-hipaa-pci-regulated-verticals.md)
6. [06 — Anti-Spoofing, Deepfake Detection, and Audio Provenance](06-spoofing-deepfake-provenance.md)
7. [07 — Voice Adversarial Threat Model (OWASP + MITRE ATLAS)](07-voice-adversarial-threat-model.md)
8. [08 — Role Boundaries and Handoffs](08-role-boundaries.md)

## Exercises

- [exercise-01 — Author a TCPA/FCC-Compliant Outbound Voice-Agent Consent Flow](exercises/exercise-01-author-a-tcpa-fcc-compliant-outbound-voice-agent-consent-flow.md) (2 h)
- [exercise-02 — State Consent-Law Matrix and the Illinois BIPA Voiceprint Case](exercises/exercise-02-state-consent-law-matrix-and-the-illinois-bipa-voiceprint-case.md) (2 h)
- [exercise-03 — EU AI Act Article 5 and 50 Mapping for Voice Agents](exercises/exercise-03-eu-ai-act-article-5-and-50-mapping-for-voice-agents.md) (2 h)
- [exercise-04 — Voice Adversarial Threat Model: OWASP LLM + Agentic + MITRE ATLAS](exercises/exercise-04-voice-adversarial-threat-model-owasp-llm-agentic-and-mitre-atlas.md) (3 h)
- [exercise-05 — Deploy ASVspoof Countermeasures and C2PA Watermarking](exercises/exercise-05-deploy-asvspoof-countermeasures-and-c2pa-watermarking.md) (2 h)

## Structure

- `01-…md` … `08-…md` — lecture chapters.
- `exercises/` — per-exercise prompts.
- `labs/` — long-form hands-on labs (none in this module; the lab-style work lives in exercises 04 and 05).
- `quizzes/` — knowledge checks.
- `resources.md` — primary-source references for every standard, regulation, paper, and tool cited.

## How to use this module

Chapters 1–5 are the compliance stack, in roughly outside-in order: outbound dialling law (1), recording + biometric law (2), EU model regulation (3), European data-protection + unsolicited-calls law (4), US sector-specific law (5). Chapter 6 shifts to the provenance / anti-spoofing primitives you deploy at the media layer; Chapter 7 is the adversarial threat model that drives your hardening backlog; Chapter 8 names the roles you escalate to.

Do the exercises paired with the chapters: 1↔Ch1, 2↔Ch2, 3↔Ch3, 4↔Ch7, 5↔Ch6. Most of the work is paper or policy-writing; exercises 4 and 5 add a running codebase.

Prerequisites: [mod-107-telephony-webrtc-transport](../mod-107-telephony-webrtc-transport/README.md) (recording and DTMF primitives) and [mod-103-stt-production-engineering](../mod-103-stt-production-engineering/README.md) (transcript handling, the surface where prompt-injection-via-ASR lives).

Next module: [mod-109-latency-budget-architecture](../mod-109-latency-budget-architecture/README.md) — once the compliance envelope is fixed, the latency budget has to fit inside it (mandatory disclosures, pause-recording round-trips, DPIA-driven region pinning all consume ms).

## Caveat

Nothing in this module is legal advice. The chapters cite primary sources — statutes, regulations, FCC rulings, EU Official Journal entries, standards documents — and give you the engineer-shaped reading of them. When a specific flow has to go into production, your legal counsel owns the final interpretation. This module teaches you to brief counsel precisely, not to replace them.
