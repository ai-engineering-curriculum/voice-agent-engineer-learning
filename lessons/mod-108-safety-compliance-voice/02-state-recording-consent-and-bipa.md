# 02. State Call-Recording Consent and the Illinois BIPA Voiceprint Rule

## Why this chapter

Chapter 1 governed *placing* the call. This chapter governs *recording* it, and separately, *doing speaker ID or emotion analysis on the recording*. The two obligations layer: you can lawfully be on the call under TCPA and still unlawfully record it under a state wiretap statute; you can lawfully record it under state law and still unlawfully extract a voiceprint under the Illinois Biometric Information Privacy Act (BIPA).

Recording consent is state-level in the United States. The federal wiretap statute ([18 U.S.C. § 2511](https://www.law.cornell.edu/uscode/text/18/2511)) sets a floor: recording is lawful when *one party* to the conversation has consented. Most states track that one-party baseline. A minority of states ("two-party" or "all-party" states) require every party on the call to consent. For a multi-state outbound or inbound voice agent, the compliance surface collapses to: **you must get all-party consent for every call that touches an all-party state.**

BIPA ([740 ILCS 14](https://www.ilga.gov/legislation/ilcs/ilcs3.asp?ActID=3004&ChapterID=57)) is adjacent but distinct. It treats a *voiceprint* as a biometric identifier. Extracting one for speaker ID, voice-cloning, or biometric authentication requires written informed consent regardless of the recording question, forbids sale, and demands a public retention schedule. BIPA has a private right of action with statutory damages (\$1,000 per negligent violation, \$5,000 per intentional) and has produced nine-figure settlements in the past five years.

## The one-party / two-party map

The federal baseline is one-party. State wiretap statutes layer on top. The following nine states are typically described as all-party or two-party for *audio* conversations (some have nuances for in-person vs. telephone, or expectation-of-privacy carve-outs):

| State | Statute | Notes |
|---|---|---|
| California | [Cal. Penal Code § 632](https://leginfo.legislature.ca.gov/faces/codes_displaySection.xhtml?sectionNum=632.&lawCode=PEN) | "Confidential communication" standard; applies regardless of where the other party is located if California party. Flanagan v. Flipboard line of cases. |
| Florida | [Fla. Stat. § 934.03](http://www.leg.state.fl.us/Statutes/index.cfm?App_mode=Display_Statute&URL=0900-0999/0934/Sections/0934.03.html) | All-party consent; criminal penalty up to 3rd-degree felony. |
| Illinois | [720 ILCS 5/14-2](https://www.ilga.gov/legislation/ilcs/fulltext.asp?DocName=072000050K14-2) | Two-party (as revised after *People v. Clark*, 2014). Plus BIPA on the biometric side. |
| Maryland | [Md. Code, Cts. & Jud. Proc. § 10-402](https://mgaleg.maryland.gov/mgawebsite/Laws/StatuteText?article=gcj&section=10-402) | Two-party for private conversations; standard set in *Fearnow v. C&P Telephone*. |
| Massachusetts | [Mass. Gen. Laws ch. 272 § 99](https://malegislature.gov/Laws/GeneralLaws/PartIV/TitleI/Chapter272/Section99) | All-party *secret* recording prohibited — if the recording is open and announced, consent is implied. |
| Montana | [Mont. Code § 45-8-213](https://leg.mt.gov/bills/mca/title_0450/chapter_0080/part_0020/section_0130/0450-0080-0020-0130.html) | All-party consent. |
| Nevada | [Nev. Rev. Stat. § 200.620](https://www.leg.state.nv.us/nrs/nrs-200.html) | Statute reads two-party; *Lane v. Allstate*, 114 Nev. 1176 (1998) construed telephone recordings as needing all-party consent. |
| New Hampshire | [N.H. Rev. Stat. § 570-A:2](https://www.gencourt.state.nh.us/rsa/html/lix/570-a/570-a-2.htm) | All-party consent. |
| Pennsylvania | [18 Pa. Cons. Stat. § 5704](https://www.legis.state.pa.us/cfdocs/legis/LI/consCheck.cfm?txtType=HTM&ttl=18&div=0&chpt=57) | All-party consent except for enumerated exceptions. |
| Washington | [RCW 9.73.030](https://app.leg.wa.gov/RCW/default.aspx?cite=9.73.030) | All-party consent. |

(The learning objective lists nine; the table above adds Maryland, which is also widely cited. Connecticut's *civil* statute is all-party, but its criminal statute is one-party — check the matter type. Verify the current state statute text before any flow goes live. <!-- needs-research: confirm the list against the most recent NCSL or Reporters Committee survey at the time of use -->)

**The practical rule for a multi-state voice agent:** capture consent *on the call* from every party. The classic opening is:

> "This call may be recorded for quality and training purposes. If you'd prefer not to be recorded, please let me know and I'll turn off recording."

That phrasing both satisfies disclosure (so one-party states are covered by the agent-side consent) and seeks the other party's consent (so all-party states are covered). Pair it with a "pause recording" command that the agent can invoke when the caller says no — otherwise the consent capture is meaningless.

## The engineering gate

Recording consent for a voice agent is a four-step gate:

1. **Geofence the call.** Determine the state of each party. For the agent side, you know (your call centre's jurisdiction). For the caller side, infer from the called-party or calling-party number (area code → likely state) *and* from any explicit location the caller gives. Area-code-only is a weak signal; mobile numbers travel.
2. **Play the disclosure prompt before recording starts.** The recording on-off state is a provider primitive (Twilio `<Record>`, Vonage `startRecording`, LiveKit room-recording toggle). Default = OFF; turn on only after the disclosure plays.
3. **Listen for refusal.** If the caller says "no, don't record," "please don't record me," "I don't consent to recording," or similar, keep recording OFF for the rest of the call. Store `recording_consent: false` on the call record.
4. **Keep the disclosure immutable.** Store the exact TTS audio (or the LiveKit room audio) of the consent prompt alongside the call transcript. If the call is ever disputed, you want to replay the exact words the caller heard.

Store per call:

```json
{
  "call_sid": "CA...",
  "caller_state_inferred": "CA",
  "agent_state": "NY",
  "recording_jurisdiction": "all_party",
  "disclosure_prompt_id": "disc-v3-2026-01",
  "disclosure_prompt_hash": "sha256:...",
  "disclosure_played_at": "2026-01-15T14:33:21Z",
  "recording_consent": true,
  "recording_started_at": "2026-01-15T14:33:24Z",
  "recording_url": "s3://.../rec.mp3"
}
```

## BIPA — what counts as a voiceprint

BIPA defines a **biometric identifier** as "a retina or iris scan, fingerprint, voiceprint, or scan of hand or face geometry" (740 ILCS 14/10). A **biometric information** means "any information, regardless of how it is captured, converted, stored, or shared, based on an individual's biometric identifier used to identify an individual."

Voice-agent operations that *do* produce biometric identifiers:

- **Voice biometric authentication** — a bank enrolls a customer's voice; subsequent calls match a voice embedding to the stored one.
- **Speaker ID / diarization that persists** — if you store a speaker embedding keyed to the caller and reuse it to identify them on later calls.
- **Voice cloning for an individual** — enrolling the user's voice to synthesise it later (Resemble, ElevenLabs enrolled voices, OpenAI Voice Engine).

Voice-agent operations that *do not* produce biometric identifiers (as typically scoped):

- **Transient diarization within a single call** (session-scoped embeddings that are deleted at hangup).
- **Open STT** — Whisper, Deepgram, AssemblyAI, Google Speech-to-Text ordinarily do not derive voiceprints. (If the provider offers a *speaker-identification* feature, that feature is in scope; the plain transcription is not.)

The BIPA obligations (740 ILCS 14/15):

- **§ 15(a)** — Develop and publish a *written policy* establishing a retention schedule and guidelines for permanently destroying biometric data. Default: destroy when the initial purpose is satisfied or within 3 years of the last interaction, whichever comes first.
- **§ 15(b)** — Before collection, inform the subject in writing (a) that a biometric identifier is being collected or stored, (b) the specific purpose, (c) the length of term; and receive a written release.
- **§ 15(c)** — Do not sell, lease, trade, or otherwise profit from biometric data.
- **§ 15(d)** — Do not disclose except with consent, to complete a transaction the subject requested, as required by law, or pursuant to a warrant.
- **§ 15(e)** — Store, transmit, and protect using reasonable care in the industry, and with a standard at least as protective as that used for other confidential information.

The engineer-shaped translation:

- Your voice-authentication enrollment page must present the three § 15(b) disclosures and require a *written release* — a signed consent, not an audio one. (Illinois *Rogers v. BNSF*, N.D. Ill. 2019, treated electronic sign-ins with explicit acknowledgment as sufficient; voice-only consent is risky.)
- Your retention policy must be public (post it; a dead link counts as not-posted under BIPA).
- The voiceprint must be stored separately from account data, encrypted at rest, and *never* leave Illinois / US jurisdictions in a way that would frustrate deletion.
- A "delete me" request destroys the voiceprint and any derived features.
- Do not share the voiceprint with model training without an additional, specific consent.

## BIPA litigation posture

BIPA has a private right of action (§ 20), statutory damages (\$1,000 negligent, \$5,000 intentional), and — per *Cothron v. White Castle*, 216 Ill. 2d 1 (2023) — damages can accrue *per scan*, not per enrollment, which has driven the nine-figure settlement outcomes. The SB 2979 amendment (signed August 2024) narrowed *Cothron* by capping damages at one per person per violation type. <!-- needs-research: confirm current statutory damages posture after SB 2979 at time of use -->

Most voice-agent teams handle BIPA by **avoiding voiceprints entirely** unless the product requires voice biometric auth. If you can design the call flow with a knowledge-based or SMS-OTP second factor instead of voice-print auth, do that. The voiceprint buys you one UX win (no extra factor) at the cost of a BIPA-shaped surface and the matching retention / consent flows.

## California CCPA / CPRA — the biometric-sensitive bucket

California's California Privacy Rights Act (CPRA, [Cal. Civ. Code § 1798.140](https://leginfo.legislature.ca.gov/faces/codes_displayText.xhtml?lawCode=CIV&division=3.&title=1.81.5.&part=4.&chapter=&article=)) lists "biometric information" as *sensitive personal information* and grants consumers the right to limit its use. Unlike BIPA, CPRA does not have a private right of action for biometric violations outside a data breach, but it does trigger:

- **Right to limit** processing of sensitive personal information to disclosed purposes.
- **Notice at collection** including the categories of sensitive PI collected.
- **Risk assessment** obligation (regulations finalized by CPPA; see [cppa.ca.gov](https://cppa.ca.gov/)).

Texas CUBI ([Tex. Bus. & Com. Code § 503.001](https://statutes.capitol.texas.gov/Docs/BC/htm/BC.503.htm)) and Washington's biometric statute ([RCW 19.375](https://app.leg.wa.gov/rcw/default.aspx?cite=19.375)) impose similar duties without a private right of action; the attorney general enforces.

Operationally, this means a voice-agent that uses voiceprints should have *one* consent and retention regime that satisfies the strictest jurisdiction on the map (currently BIPA) and apply it everywhere. Trying to run different regimes per state is more legal risk than it is worth.

## A "pause recording" primitive

For any flow where the caller may be asked for a SSN, PAN, PHI, or any other descope-able field, you need a *pause recording* primitive that:

1. Sends a provider-level command to stop writing the recording file.
2. Confirms in the response that recording actually stopped — not merely queued.
3. Resumes recording after the sensitive segment.
4. Produces a single contiguous recording with a gap (not multiple fragments) — some providers emit multiple files, which breaks downstream workflows.
5. Writes an audit log entry naming who/what paused and resumed, with timestamps.

Provider primitives:

- **Twilio** — `<Record recordingStatusCallback>` + `pauseRecording` / `resumeRecording` REST actions, or `<Pay>` for a PCI-safe card capture that never enters recording ([twilio.com/docs/voice/api/payment-resource](https://www.twilio.com/docs/voice/api/payment-resource)).
- **Vonage** — `recording:action` NCCO with pause support ([developer.vonage.com/en/voice/voice-api/features/recording](https://developer.vonage.com/en/voice/voice-api/features/recording)).
- **Amazon Connect** — `UpdateContactAttributes` with secure-input blocks ([docs.aws.amazon.com/connect/latest/adminguide/secure-input.html](https://docs.aws.amazon.com/connect/latest/adminguide/secure-input.html)).
- **Google Cloud Contact Center AI** — DTMF-only input blocks. <!-- needs-research: current API surface for descope on CCAI -->

The pause primitive is the join point between this chapter and PCI DSS (Chapter 5).

## Summary

- Federal wiretap law is one-party; nine-plus states are all-party. Design for the strictest jurisdiction and play a consent disclosure on every call.
- Keep the disclosure audio, timestamp, and caller response immutable — these are the evidentiary record when the call is disputed.
- BIPA treats a voiceprint as biometric data: written consent, published retention schedule, no sale, encrypted storage, no derivative sharing without further consent. Avoid voiceprints unless the product needs them.
- CPRA / CUBI / Washington extend biometric obligations beyond Illinois; design to the strictest regime.
- A "pause recording" primitive is non-optional for any flow that touches sensitive data; wire it to the provider API, verify the acknowledgement, and audit it.

Next chapter: on the EU side, the questions are *whether the system is allowed at all* (Article 5) and *whether disclosure is mandatory* (Article 50).
