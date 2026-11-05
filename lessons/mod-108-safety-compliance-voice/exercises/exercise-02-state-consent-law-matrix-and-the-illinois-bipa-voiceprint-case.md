# exercise-02: State Consent-Law Matrix and the Illinois BIPA Voiceprint Case

**Estimated effort:** 2 hours · **Output:** a state-by-state recording-consent matrix + a BIPA enrollment-flow brief + a pause-recording primitive specification.

## Objective

Translate [Chapter 2](../02-state-recording-consent-and-bipa.md) into (a) an engineering-usable state recording-consent policy table, (b) a BIPA-compliant voice-biometric enrollment design, and (c) a pause-recording primitive specification that works end-to-end across the recording file, the STT stream, and the LLM context. Paper deliverable; no running system required.

## Scenario

You are shipping an inbound voice agent for a fictional fintech, "TrueCredit." The agent handles support calls, authenticates returning callers with a voice-biometric check (enrollment is optional for premium members), and may take payments when callers want to pay down a balance. Callers come from all 50 US states.

## Prerequisites

- [Chapter 2 — State Call-Recording Consent and the Illinois BIPA Voiceprint Rule](../02-state-recording-consent-and-bipa.md).
- Primary sources: statutes in the Chapter 2 table (CA Penal § 632, FL § 934.03, IL 720 ILCS 5/14-2, MA ch. 272 § 99, MT § 45-8-213, NV NRS 200.620, NH RSA 570-A:2, PA 18 Pa.C.S. § 5704, WA RCW 9.73.030, MD CJP § 10-402) and [BIPA 740 ILCS 14](https://www.ilga.gov/legislation/ilcs/ilcs3.asp?ActID=3004&ChapterID=57).
- Secondary: NCSL or Reporters Committee for Freedom of the Press state-by-state recording-law summary for a sanity check, but cite the statutes themselves in your deliverable. <!-- needs-research: confirm NCSL/RCFP survey is current at time of use -->

## Steps

### Part A — The state recording-consent matrix (45 minutes)

1. Produce a table with one row per US state + DC with columns:
   - `state`
   - `statute_citation`
   - `regime` — one-party / two-party / all-party / mixed-with-exception
   - `applies_to_phone_calls` — yes / no / unclear
   - `applies_to_in_person` — yes / no / unclear
   - `criminal_penalty` — misdemeanor / felony / none
   - `private_right_of_action` — yes / no
   - `notes` — any caselaw or carve-out that matters (e.g., Massachusetts' "secret" qualifier; Nevada's statute vs. *Lane* caselaw).
2. For every state you classify as two-party or all-party, write the *disclosure prompt* the agent should play at call start. All states should receive a single uniform prompt — "we will play the two-party prompt everywhere so we never get it wrong" is a defensible position. State one.
3. Identify three states where you believe the statute is *not* a strict all-party rule but your policy should treat them as such because the caselaw or regulator guidance makes the strict reading safer. Justify in one sentence each.
4. Include citations with full statute names and public URLs. Do not paraphrase — quote the operative language.

### Part B — The BIPA-compliant enrollment flow (45 minutes)

5. Design the voice-biometric enrollment flow for the "premium voice auth" feature. Specify:
   - Enrollment UI — written disclosure text that meets 740 ILCS 14/15(b): identification of biometric collection, specific purpose, length of term, and the written release mechanism.
   - Where consent is captured (web form, mobile app) — explicitly *not* voice-only.
   - Storage: what is the artefact? (embedding, not raw audio; encrypted at rest; key separate from account DB).
   - Retention schedule and public policy link.
   - "Delete my voice" primitive: the API endpoint, what it destroys, confirmations to user.
6. Write the retention policy text that must be publicly posted per § 15(a). Include a specific retention window ("destroyed when initial purpose is satisfied OR 3 years after last interaction, whichever first") and a destruction method ("cryptographic erasure of the embedding key").
7. Enumerate three *other* places in the TrueCredit stack where voiceprint-derived data might leak (training sets, analytics, support-ticket attachments) and specify a control for each.
8. Note which jurisdictions beyond Illinois would *also* view the voiceprint as sensitive biometric data (California CPRA, Texas CUBI, Washington biometric statute). Recommend applying the BIPA-grade controls everywhere.

### Part C — The pause-recording primitive (30 minutes)

9. Produce a specification for a `pause_recording(reason, resume_hint_seconds)` primitive. Specify:
   - The provider-level call (e.g., Twilio `pauseRecording` REST; LiveKit room-recording toggle; Vonage NCCO action). Choose one provider for concreteness.
   - The STT-stream gating mechanism (how you prevent the sensitive window from going to STT).
   - The LLM-context gating mechanism (what the LLM sees during the paused window — ideally a placeholder token).
   - The audit-log entry format: who/what/when/why.
   - The confirmation path: how the primitive verifies the pause *actually happened* before proceeding.
   - The resume path: including the edge case where the caller spoke the sensitive data and the agent's STT already queued it.
10. Design the red-team test that proves the primitive works. Specify:
    - A test fixture: an agent conversation with a dummy sensitive data moment (a fake 16-digit card number).
    - A grep pass across the recording file, the STT output, the LLM context, and the provider log.
    - A pass/fail criterion: no appearance of the digits anywhere downstream.

## Starter guidance

- Several state statutes exempt calls where notice is given at the start. The MA § 99 "secret" qualifier is the classic. Read the actual statute language rather than secondary summaries — the carve-outs matter.
- Nevada's statute at NRS 200.620 reads as two-party *for telephone* calls after the *Lane v. Allstate* interpretation; the statute text alone would read differently. This is a classic caselaw-trumps-text situation; cite both.
- For BIPA, the Supreme Court of Illinois cases *Rosenbach v. Six Flags* (2019), *McDonald v. Symphony Bronzeville Park* (2022), and *Cothron v. White Castle* (2023), and the SB 2979 amendment (signed August 2024) are the key developments. Cite them in your enrollment-flow brief. <!-- needs-research: confirm SB 2979 effective date and the current statutory-damages posture at time of use -->
- For the pause-recording primitive, consult [Twilio's recording API](https://www.twilio.com/docs/voice/api/recording) and the `pauseRecording` action; verify the provider's stated behaviour (does it stop *and* strip, or just stop writing from this moment?).

## Acceptance criteria

- [ ] A 51-row (50 states + DC) consent-regime table with citations.
- [ ] The uniform two-party disclosure prompt copy, deployable everywhere.
- [ ] Three state-level "strict-reading because caselaw" justifications.
- [ ] A BIPA enrollment flow with § 15(b) disclosure text, written-release mechanism, encrypted embedding storage, and a destroy endpoint.
- [ ] A publicly-postable retention-policy paragraph.
- [ ] Three stack-wide voiceprint-leakage locations with a control for each.
- [ ] A `pause_recording` primitive spec with provider call, STT gating, LLM gating, audit log, confirmation path, resume path.
- [ ] A red-team test spec with fixture, grep pass, and pass/fail criterion.

## Stretch goals

- **DSAR flow.** Design the data-subject-access flow for an Illinois resident requesting (a) all their voiceprints and (b) deletion of them. Include the response-time SLA and the format in which the embedding (or confirmation of its existence) is returned.
- **BIPA-safe fallback.** For callers who refuse voice biometric enrollment, design a fallback authentication path that does not weaken security. Knowledge-based auth + OTP-to-phone is the common pattern; critique its weaknesses (SIM swap).
- **CPRA / CUBI parity.** Draft one-pagers that could be given to the California and Texas DPAs if asked. Show that the BIPA-grade controls satisfy both regimes.
- **Cross-recording consent.** Design the flow where the caller is recording *their own* side of the call too (common for consumer complaints). How does your consent model account for being on a recorded-by-both-sides call?

## Keep it

The state-consent matrix and the BIPA enrollment pattern are reusable across every voice product you will ship. The pause-recording primitive is foundational for PCI descope in exercise 05 and onward.
