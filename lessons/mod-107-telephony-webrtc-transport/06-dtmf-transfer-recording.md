# 06. DTMF, Transfer, Warm Handoff, and Recording Controls

## Why this chapter

The single-call happy path — user dials in, bot answers, bot talks, user hangs up — is not the only path. Production voice agents need to recognise keypad input ("press 1 for sales"), transfer to a human agent, recognise when to escalate, start and stop recording with consent, and surface call-lifecycle events to the rest of the system. All of these are **in-call operations** that live on top of the transport we built in Chapters 1–5.

This chapter walks each operation, names the three or four ways it is commonly implemented, and gives you the decision rubric for picking one.

## DTMF — Dual-Tone Multi-Frequency

DTMF is the "touch tones" from telephony: pairs of audio frequencies that encode the digits 0–9, `*`, `#`, A–D. For a voice agent, DTMF is how a user navigates menus, enters an account number, or confirms a payment code — even when speech recognition is live.

Three transports, which you may hear interchangeably:

### In-band RTP events — RFC 4733

[RFC 4733](https://datatracker.ietf.org/doc/html/rfc4733) (obsoletes RFC 2833) defines a dedicated RTP payload type for DTMF events. Instead of the actual tones (which would survive codec transcoding badly), the sender emits a telephony-event RTP packet with the digit, volume, and duration encoded as a small binary payload.

```
RTP payload for a DTMF '5' event (RFC 4733 §2.3):
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     event '5' |E|R| volume  |          duration             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- The payload type is negotiated in SDP as `a=rtpmap:<pt> telephone-event/8000` (the clock rate matches the RTP media clock; for Opus WebRTC it's 48000).
- **This is what WebRTC uses.** The browser's `RTCDTMFSender` emits RFC 4733 events, not actual tones.

### SIP INFO — RFC 2976 / RFC 6086

Some SIP endpoints send DTMF out-of-band via a SIP `INFO` request:

```
INFO sip:agent@example.com SIP/2.0
Content-Type: application/dtmf-relay

Signal=5
Duration=250
```

This bypasses the media plane entirely. It's historically common in enterprise PBX deployments and still shows up on legacy trunks. Less reliable than RFC 4733 for mid-call feedback latency, more reliable for trunks that mangle in-band events.

### Provider API events

When you're using a managed provider (Chapter 5), DTMF is usually delivered to your application as a webhook or a WebSocket event, regardless of which of the above the carrier uses on the wire:

- **Twilio Media Streams** delivers DTMF as a JSON event over the same WebSocket as the audio.
- **Twilio ConversationRelay** has explicit DTMF events in its event stream.
- **Vonage WebSocket connect** emits DTMF events via webhooks (`eventUrl`).
- **LiveKit SIP** surfaces DTMF events through the SDK's participant API.

The provider abstracts away the distinction between RFC 4733 and SIP INFO. Your agent reads events; the provider handles the plumbing.

### Decision rubric

- **Browser WebRTC client:** use `RTCDTMFSender`. Always RFC 4733 on the wire.
- **SIP trunk you own:** negotiate RFC 4733 in SDP; fall back to SIP INFO only if the carrier requires it. Log which was used per call so when something breaks you don't have to guess.
- **Managed provider:** read the event stream; don't worry about the wire format.
- **ASR-based DTMF ("the user said five")** is a trap. Pure keypad input is one in-band RTP event; pure speech-to-digit is a three-step ASR-plus-prompt-plus-post-processing round trip with higher latency and lower precision. Use ASR only when the keypad isn't available.

## Call transfer

The user wants to talk to a human (or a different bot). Three mechanisms, in rough order of flexibility:

### REFER — RFC 3515

[RFC 3515](https://datatracker.ietf.org/doc/html/rfc3515) and [RFC 5589](https://datatracker.ietf.org/doc/html/rfc5589) (the call-transfer-explicit spec) define SIP's `REFER` method:

```
REFER sip:caller@example.com SIP/2.0
Refer-To: <sip:supervisor@example.com>
```

The receiving endpoint, on seeing REFER, issues a new INVITE to the Refer-To URI and reports progress back via SIP NOTIFY messages. Classic **blind transfer** (hang up as soon as the new call starts) and **attended transfer** (dial, confer with the supervisor, then hang up) are both REFER-driven, with different flows.

REFER is the "correct" SIP mechanism for transfer. In practice, two things complicate it:

- Not every endpoint supports REFER properly; carrier SBCs sometimes strip or mishandle it.
- The in-dialog identity preservation across REFER is subtle — the original caller sees a new SIP dialog, and your call-logging needs to stitch the before and after together via `Replaces` or correlation metadata.

### Provider-flavored transfer

Managed providers wrap transfer in a higher-level API:

- **Twilio `<Dial>`**: during a TwiML flow, you redirect to a TwiML Bin that returns `<Response><Dial>+15552225555</Dial></Response>`. Twilio bridges the legs and reports status via `action`-URL callbacks.
- **Twilio REST `Calls.update(url=...)`**: mid-call, you POST to Twilio asking it to redirect the call to a new TwiML URL, which can be a `<Dial>` block.
- **Vonage NCCO `connect` action**: `[{"action":"connect","endpoint":[{"type":"phone","number":"15552225555"}]}]`. Updated mid-call via the NCCO transfer URL.
- **Amazon Connect contact flows**: use "Transfer to flow" or "Transfer to queue" blocks; the flow itself is a state machine.
- **LiveKit SIP**: add a new SIP participant to the room (the human agent). The transport layer doesn't change — the room gains or loses participants.

### REST-driven handoff

The provider-flavored transfer is actually one specific case of a REST-driven handoff. In general, **the agent's "transfer" is a tool call that invokes the provider's API to add a new participant or redirect a leg**. For voice agents built on LLMs with tool use, that means:

1. LLM emits a `transfer_to_human` tool call with a `reason` and optional `summary`.
2. The agent's backend validates the reason (is this user even transferable? is a human available?).
3. The backend calls the provider's API to redirect or dial out.
4. The agent announces the transfer and goes silent.
5. The provider reports the leg's final status via webhook; the backend logs it.

### Blind vs. warm vs. attended

Terminology that will come up:

- **Blind transfer** — the agent dials the destination and hangs up as soon as the new leg rings. The caller may hear ringing, then the destination. Simplest, worst UX.
- **Warm transfer** — the agent stays on the call, speaks to the destination (summarising the issue), then bridges the caller in or hangs up.
- **Attended transfer** — the agent dials the destination privately, confirms the destination can take the call, then bridges.

For a voice agent handing off to a human, **warm transfer** is almost always the right UX — the human picks up, hears "This is Bot. I have Alice on the line; she's trying to reset her password but needs manual verification," then the bot drops and Alice is on the line. This is implemented as:

1. Put caller on hold or play hold-music (`<Play>` loop).
2. Dial the destination as a *separate* leg to a different endpoint than the caller.
3. Produce a bot-generated summary on the destination leg via TTS.
4. On human acknowledgement (DTMF or voice), bridge the caller leg to the destination leg.
5. Drop the bot's participation.

LiveKit, Pipecat, and provider abstractions all have patterns for this. It is not one line of code; it is a flow with explicit state machine.

## Call recording and consent

Recording is where the compliance axis meets the engineering axis. The rules vary dramatically by jurisdiction:

- **US federal "one-party consent"** — recording legal if any participant (including the recorder) consents. Permissive floor.
- **US "two-party consent" states** (California, Florida, Illinois, Maryland, Massachusetts, Montana, Nevada, New Hampshire, Pennsylvania, Washington) — all parties must be notified before recording; in some states, affirmative consent is required.
- **EU: GDPR** — call recording is "processing of personal data." You need a lawful basis (consent, legitimate interest, contract) and must disclose the recording, the purpose, and the retention period.
- **UK: UK GDPR + PECR** — essentially the same shape as EU GDPR.
- **PCI DSS** — if the call captures cardholder data, specific DSS requirements apply (don't record the CVV, encrypt stored recordings, restrict access).
- **HIPAA** — if the call captures Protected Health Information, BAAs are needed with every vendor that stores the recording.
- <!-- needs-research: confirm current status of recording-consent laws in each jurisdiction at authoring time; this area changes with court rulings -->

The engineering implications:

### Recording start / stop under agent control

Provider APIs expose record-on-demand:

- **Twilio `<Record>` TwiML, `Recordings.create()` REST API, or mid-call `<Start><Record/>` and `<Stop><Record/>` directives.**
- **Vonage `startRecording` and `stopRecording` NCCO actions.**
- **Amazon Connect "Set recording and analytics behavior" block in a contact flow.**
- **LiveKit Egress service** — start recording a room or an individual track via API; can target cloud storage directly.

### The consent-gate pattern

The responsible pattern:

1. **Pre-call disclosure.** IVR plays "This call may be recorded for quality and training."
2. **Wait for consent or opt-out.** User presses 1 to accept, 2 to decline, or waits for N seconds.
3. **If declined, don't record.** Or route to a non-recording queue.
4. **If accepted, start recording and emit a tamper-evident consent log.** The consent log is what your legal team hands over if the recording's admissibility is challenged.
5. **Allow mid-call pause** when the user says "I'm going to read out my credit card." The agent's PCI handoff tool call stops recording, collects the data on a non-recorded sub-leg, and resumes.

### Technical pause for PCI

If your agent accepts credit cards by voice, you cannot record the card data. The common pattern:

1. User signals "I'd like to pay now."
2. Agent calls `pause_recording()` tool.
3. Agent transfers card entry to a specialised PCI endpoint (usually DTMF digits captured by the provider; e.g., [Twilio `<Pay>`](https://www.twilio.com/docs/voice/api/payment-resource) or [Amazon Connect secure IVR](https://docs.aws.amazon.com/connect/latest/adminguide/secure-input.html)).
4. Agent resumes recording after confirmation.

Do **not** attempt to post-process a recording to redact a 16-digit sequence. PCI DSS treats "recorded then redacted" differently from "never recorded." The regulators care about the storage step.

### Storage, retention, and access

- **Encrypt at rest.** Provider-side encryption by default; customer-managed keys if you're regulated.
- **Retention tied to policy.** Seven years is common for finance; "as long as needed for quality review" with periodic purge is common for general support. Pick a policy and automate the purge.
- **Access logging.** Who downloaded a recording? When? Audit.
- **Subject access requests.** GDPR gives data subjects the right to a copy of their recordings; you need a mechanism to retrieve and deliver one.

## Status and webhook callbacks

Every provider sends lifecycle events to your webhook endpoints:

- **Twilio**: `initiated`, `ringing`, `answered`, `completed`, `busy`, `no-answer`, `canceled`, `failed` on the call resource; recording events on the recording resource.
- **Vonage**: similar vocabulary via `eventUrl`.
- **Amazon Connect**: contact events via Amazon EventBridge.
- **Azure**: Event Grid webhook events.

Two engineering concerns:

### Idempotency and retry

Provider webhooks are **at-least-once**. The same event can be delivered twice if your handler is slow or returns a non-2xx. Your handler must be idempotent — key off the provider's `CallSid` / `UUID` and the event name; store processed events; drop duplicates.

### Signature verification

Every provider signs webhook payloads. **Verify the signature.** Attackers love webhook endpoints that trust whatever POSTs to them. Twilio uses `X-Twilio-Signature`; Vonage uses `Authorization: Bearer <JWT>`; Amazon uses standard AWS Signature v4.

### Status to log

The minimum fields to log per call:

- `call_id` (provider's identifier)
- `started_at`, `answered_at`, `ended_at`
- `direction` (inbound / outbound)
- `caller_id` (E.164, if unmasked)
- `destination` (E.164)
- `stir_shaken_attestation` (A/B/C/none, for inbound)
- `terminated_by` (caller / bot / system)
- `recording_sid` (if recorded)
- `consent_timestamp` (if consent-gated)

These feed the observability platform (mod-112) and the compliance evidence trail.

## Summary

- DTMF has three transports (RFC 4733, SIP INFO, provider API); WebRTC always uses RFC 4733.
- Call transfer has three shapes (REFER, provider `<Dial>`-style, REST-driven handoff); warm transfer is the human-agent UX you almost always want.
- Recording is a compliance axis before it is a technical one; implement consent gates, PCI pauses, and audit-logged access.
- Webhooks are at-least-once and must be signature-verified.

Next chapter: reading WebRTC `getStats()` output to triage a degrading call live.
