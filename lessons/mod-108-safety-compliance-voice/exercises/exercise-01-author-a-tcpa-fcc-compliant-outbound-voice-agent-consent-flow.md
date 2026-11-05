# exercise-01: Author a TCPA/FCC-Compliant Outbound Voice-Agent Consent Flow

**Estimated effort:** 2 hours · **Output:** a filled consent-record schema + consent-capture UI copy + a dialler-gate flow diagram + a revocation-plumbing design.

## Objective

Translate [Chapter 1](../01-tcpa-fcc-outbound-consent.md) into a deployable consent flow for a hypothetical outbound voice-agent product. You will author (a) the consent-record data structure and audit trail, (b) the consent-capture UI copy that meets the 47 CFR 64.1200(f)(9) four-element bar, (c) the six-gate dialler flow, and (d) the revocation pipeline that catches both channel-local and cross-channel revocations. Deliverables are paper / Markdown / pseudocode; this exercise does not require a working dialler.

## Scenario

You are shipping an outbound voice agent for a (fictional) subscription auto-insurance company, "SafeRateInsure." The agent calls leads collected from the company's web funnel to walk them through a rate quote and, if interested, to initiate a policy. Leads opt in on the web form; some arrive from a partner lead-generator network. The campaign runs in all 50 US states plus DC.

## Prerequisites

- [Chapter 1 — TCPA, the FCC's AI-Voice Ruling, and Outbound-Call Consent](../01-tcpa-fcc-outbound-consent.md).
- Primary sources: [47 CFR 64.1200](https://www.ecfr.gov/current/title-47/chapter-I/subchapter-B/part-64/subpart-L/section-64.1200), [FCC 24-17 (Feb 2024 AI-voice ruling)](https://docs.fcc.gov/public/attachments/FCC-24-17A1.pdf), [FCC 24-24 (revocation rule)](https://docs.fcc.gov/public/attachments/FCC-24-24A1.pdf).
- Familiarity with JSON Schema or an equivalent data-modelling notation.

## Steps

### Part A — Consent-record schema (30 minutes)

1. Produce a JSON Schema (or equivalent) for a `consent_record`. At a minimum, capture:
   - Subject identity (id, name, email).
   - The phone number authorized (E.164).
   - The *exact rendered consent text* the user saw, the version, and a SHA-256 hash of that text.
   - Timestamps: `granted_at`, `revoked_at`.
   - Capture method (web form, SMS reply, oral) and metadata (IP, user agent, session ID).
   - The *scope* of consent: channel (voice / SMS / email), purpose (informational / marketing / post-sale), and seller identity (SafeRateInsure vs. partner).
   - Audit linkage: a stable `consent_id`, `batch_id`, `lead_source`.
2. Include a `revocation_events` sub-array that captures every revocation attempt with timestamp, channel, phrase that triggered it, and confirmed-by metadata.
3. Write a one-paragraph justification for each field tying it to a specific 47 CFR 64.1200 clause or to the FCC's demonstrability requirement.

### Part B — Consent-capture UI copy (30 minutes)

4. Draft the consent-capture UI copy for the web funnel. Produce two variants:
   - **Variant A: pure SafeRateInsure consent** — the lead consents to receive calls from SafeRateInsure only.
   - **Variant B: SafeRateInsure plus a *named* list of partner sellers.** Each named seller is listed by legal name with a separate per-seller checkbox.
5. For each variant, write the full disclosure text and list which of the four 64.1200(f)(9) elements each sentence satisfies:
   - Signed written agreement.
   - Clear and conspicuous disclosure authorising artificial-voice calls using an automatic dialing system.
   - Non-conditional on purchase.
   - Identifies the specific number.
6. Call out at least two patterns that would *fail* the bar (buried text, pre-checked checkbox, "I agree to the TOS *and* calls" combined checkbox). Explain why each fails.

### Part C — The six-gate dialler (30 minutes)

7. Draw or describe the six-gate dialler flow from Chapter 1 as pseudocode or a flow diagram:
   1. Consent gate.
   2. DNC gate (Federal Do Not Call, internal DNC, state DNC lists).
   3. Reassigned Numbers Database gate.
   4. Local-time hours gate (8 AM – 9 PM called party local time).
   5. Attestation + call-label gate (STIR/SHAKEN A, no "Scam Likely" label).
   6. Place the call.
8. For each gate, specify:
   - The input it consumes (what field of `consent_record` or what external API).
   - The decision it makes (allow / suppress / defer).
   - The failure audit log entry it emits.
   - The expected latency budget (ms).
9. Describe the fallback when the RND API times out. Default should be *suppress*, with a documented exception path that requires manual review.

### Part D — Revocation plumbing (30 minutes)

10. Design the revocation path. Specify:
    - An on-call live-transcript classifier that detects revocation phrases (`stop`, `quit`, `end`, `remove me`, `don't call me`, `opt out`, `unsubscribe`, variants). List at least 20 canonical phrases you would seed the classifier with.
    - An action on hit: confirm revocation in the agent reply (write the exact confirmation copy), set `revoked_at`, add a `revocation_events` entry.
    - A cross-channel propagator that fans out to any other channel keyed to the same phone number (SMS, email).
    - A regulatory deadline check: the revocation must be honored across the platform within 10 business days (FCC 24-24).
11. Specify the UI-side revocation mechanism (one-click toggle on the account page, no phone-tree-only revocation).

## Starter guidance

- Use a real statutory reading. The four elements are at [47 CFR 64.1200(f)(9)](https://www.ecfr.gov/current/title-47/chapter-I/subchapter-B/part-64/subpart-L/section-64.1200). Quote the regulation in your justification; do not paraphrase.
- The 1:1 consent rule (89 FR 5098) was vacated in *Insurance Marketing Coalition v. FCC* (11th Cir. Jan 2025); assume the pre-1:1 regime is in force. Variant B is therefore allowed *if the partner list is specific and the consent text is clear about each seller*. Verify current state before building. <!-- needs-research: confirm current regulatory status at the time of use -->
- For the RND, assume the FCC-contracted service at [reassigned.us](https://www.reassigned.us/) with the 15-day safe-harbour cache window.
- For the "Scam Likely" labeller check, assume you query three labellers (First Orion, Hiya, TNS) via a reputation-aggregator provider.
- Local time inference from area code is defensible under FCC guidance; mobile portability makes it imperfect. Note the limitation.

## Acceptance criteria

- [ ] A JSON Schema (or equivalent) for `consent_record` with every field justified against a specific clause.
- [ ] Consent-capture UI copy for both Variant A and Variant B, with per-sentence mapping to 64.1200(f)(9) elements.
- [ ] A list of at least two failing patterns with the specific reason each fails.
- [ ] A six-gate dialler flow with input, decision, failure-log, and latency budget for each gate.
- [ ] A fallback policy for RND API timeout; defaults to suppress.
- [ ] A revocation path specification: 20 seed phrases, confirmation copy, cross-channel propagation, 10-business-day SLA check, UI-side mechanism.
- [ ] A one-paragraph self-assessment: of the above, which items are *technical* obligations and which are *contractual* (legal review); what would you escalate to counsel before launch?

## Stretch goals

- **State mini-statutes.** Pick two state TCPA mini-statutes (Florida FTSA, Washington CEMA) and flag which of your dialler gates would need state-specific behaviour. Produce a short table of deltas.
- **AI-disclosure.** Draft an AI-disclosure sentence that would satisfy both EU AI Act Article 50 and any US state bill you can cite. Position it in the call opening alongside the business-identification message.
- **Reassigned-Number safe harbor instrumentation.** Design the logging that would prove in litigation that you queried the RND within 15 days before the call and received a no-reassignment response.
- **Partner attestation.** For Variant B, draft a contractual clause that partner lead-generators must sign attesting they captured consent on the SafeRateInsure name. Outline the audit trail the partner must provide.

## Keep it

The consent-record schema is a reusable template across any outbound-call product you will ship. Keep it in a `templates/` folder in your personal reference.
