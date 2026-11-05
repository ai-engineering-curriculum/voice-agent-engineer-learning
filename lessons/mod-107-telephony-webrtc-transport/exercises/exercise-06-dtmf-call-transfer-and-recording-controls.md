# exercise-06: DTMF, Call Transfer, and Recording Controls

**Estimated effort:** 2 hours · **Output:** a flow with keypad input, a warm-handoff state machine, a consent-gated recording trigger, and a captured webhook event stream

## Objective

Make [Chapter 6](../06-dtmf-transfer-recording.md) concrete by layering **in-call operations** on top of the Twilio agent you built in exercise 05. By the end you will have: DTMF menu navigation, a warm transfer to a human (simulated), and a consent-gated recording with a working mid-call pause for PCI-like data entry.

## Prerequisites

- [Chapter 6 — DTMF, Transfer, Warm Handoff, and Recording Controls](../06-dtmf-transfer-recording.md).
- The Twilio Media Streams or ConversationRelay path from [exercise 05](exercise-05-wire-twilio-media-streams-and-conversationrelay-to-a-voice-agent.md).
- A second phone number (mobile, SIP endpoint, or a second Twilio number) to act as the "human agent" for the warm-handoff step.
- Object storage (S3, GCS, local disk) for the recording artefact.

## Steps

### Part A — DTMF input (30 minutes)

1. Extend your agent to listen for DTMF events on the Media Streams WebSocket (events are JSON of the form `{"event":"dtmf", "digit":"5", ...}`; see [Twilio Media Streams spec](https://www.twilio.com/docs/voice/media-streams/websocket-messages)).

2. Add a trivial menu:
   - "Press 1 to check your balance."
   - "Press 2 to speak to an agent."
   - "Press * to cancel."

3. On digit `1`, speak the balance (mock value).
   On digit `2`, trigger the warm-handoff flow (Part B).
   On `*`, speak a "cancelled" confirmation.

4. Dial the agent, exercise each key path. Confirm the agent reacts within ~500 ms of the digit (the DTMF event is near-realtime).

5. Log the transport. For each DTMF event, record whether it arrived via Media Streams JSON, SIP INFO (if your provider shows it), or in-band. Twilio Media Streams typically delivers RFC 4733 as JSON — confirm.

### Part B — Warm transfer to a "human agent" (45 minutes)

6. When the user presses `2`, execute the warm-handoff state machine:

   - **State: "putting-user-on-hold"** — speak "Hold on, I'll find someone." Start a music / silence loop on the user leg.
   - **State: "dialing-human"** — originate a new call to the second phone number via Twilio REST (`client.calls.create(...)` with a TwiML URL that answers and connects to a second WebSocket).
   - **State: "briefing-human"** — on the human-agent leg, your bot plays a TTS summary: "You have a caller who was asking about their balance. They now want to talk to a person. Press 1 to accept, 9 to reject."
   - **State: "accepting"** — on DTMF `1` from the human, use Twilio's `<Dial>` or conference APIs to bridge the two legs. The bot drops off.
   - **State: "rejecting"** — on DTMF `9`, return to the user leg, speak "Sorry, no one is available. Please call back later."

7. Document the state machine as a diagram (textual is fine; mermaid is better). For each transition, label which Twilio API call or TwiML verb drives it.

8. Place a call, press `2`, answer the second phone, press `1` to accept, confirm you can talk to the "caller" (yourself on the first leg).

### Part C — Consent-gated recording (30 minutes)

9. At the start of each call, add the consent gate from Chapter 6:
   - Play "This call may be recorded for quality assurance. Press 1 to accept, 2 to decline."
   - On `1`: start recording with `<Start><Record/>` or the REST API; log a `consent_granted_at` timestamp.
   - On `2`: route to a non-recording path; log `consent_denied_at`.

10. During a recorded call, add a PCI-pause:
    - If the user's speech contains "credit card" (match on the transcript), speak "I'll pause the recording. Please enter your card number on the keypad."
    - Pause the recording via REST (`client.calls(sid).recordings(rec_sid).update(status='paused')` or the equivalent for Media Streams flows).
    - Collect 15 DTMF digits (card number + CVV + expiry — fake data for this exercise).
    - Speak "Thank you, resuming." Resume the recording.

11. After the call, fetch the recording via REST, confirm it is **shorter than the full call duration** and the pause window is missing.

### Part D — Webhook event stream (15 minutes)

12. Configure Twilio's `status_callback` on the outbound and inbound numbers to POST to your `/status` endpoint.

13. Log every event delivered during a 2-minute call: `initiated`, `ringing`, `answered`, `in-progress`, `completed`, `recording-completed`, `transfer-*` (if emitted).

14. Verify the **webhook signature** (`X-Twilio-Signature`) using the Twilio SDK's validator. Fail closed on signature mismatch and log an alert.

15. Confirm **idempotency**: deliberately return a 500 for the first `completed` event to force Twilio to retry. Verify your handler sees the duplicate and drops it on the second try.

## Starter guidance

- Twilio REST for mid-call redirect: `client.calls(sid).update(url='https://.../transfer-twiml')`. The URL returns new TwiML.
- For the bridge step, [`<Dial>`](https://www.twilio.com/docs/voice/twiml/dial) is one option; [Twilio Conference](https://www.twilio.com/docs/voice/twiml/conference) is the more flexible one if you want hold music, muting, and attended-transfer flexibility.
- Record pausing: [`Recording.update(status='paused')`](https://www.twilio.com/docs/voice/api/recording-resource). Resume with `status='in-progress'`.
- Webhook signature validation: [`twilio.request_validator.RequestValidator`](https://www.twilio.com/docs/usage/webhooks/webhooks-security) with your account's auth token.
- A real production PCI handoff would use [Twilio `<Pay>`](https://www.twilio.com/docs/voice/api/payment-resource) or a token provider; the DTMF-collect in this exercise is deliberately simpler and still carries card data through your server — don't deploy it as-is.

## Acceptance criteria

- [ ] DTMF menu wired; three key paths (1, 2, *) each produce the correct behaviour; reaction time measured and under 500 ms.
- [ ] Warm-transfer state machine implemented end-to-end; the human agent can accept or reject; on accept, the two legs are bridged and the bot is out of the loop.
- [ ] State-machine diagram (prose or mermaid) with every transition labelled with the Twilio API / TwiML verb that triggers it.
- [ ] Consent-gated recording: accept path records, decline path does not; `consent_granted_at` / `consent_denied_at` logged with call ID.
- [ ] PCI-pause: during a recorded call, recording pauses for the DTMF-collect window; the resulting recording file is demonstrably shorter than the full call and the paused window is absent.
- [ ] Webhook signature verification implemented; mismatch triggers an alert; duplicate `completed` event dropped on retry.
- [ ] Two-paragraph reflection: which part of this exercise would you refuse to ship on day one, and why? (PCI handling is the expected answer; a specialised tool is required.)

## Stretch goals

- **Add a ToolCall abstraction.** Expose `transfer_to_human(reason, summary)`, `pause_recording()`, `resume_recording()`, `hang_up()` as tool calls to your LLM. On each tool invocation, call the matching Twilio API. This is the shape of the real production agent (mod-111 — orchestration).
- **REFER-based transfer.** If you have a SIP trunk available, implement the warm transfer via SIP REFER (Chapter 4) instead of Twilio's `<Dial>`. Capture the `REFER`, `NOTIFY sipfrag`, and new dialog creation.
- **Multi-recording.** Enable **dual-channel** recording: caller on one channel, bot on the other. Run the recording through your STT provider separately per channel and verify the transcripts don't overlap.
- **Consent audit log.** Add an append-only log (hash-chained or Merkle-logged) of consent events. Expose a replay API that can prove "consent was recorded at $timestamp for call $id." This is what a legal subject-access request looks like.
- **Jurisdiction-aware consent.** Look up the caller's region from the E.164 prefix; route US-one-party-consent, US-two-party, and EU GDPR paths differently. What's the minimum IVR disclosure for each?

## Keep it

The warm-transfer state machine and the consent-gate flow are production primitives you will keep rebuilding until you extract a library. Extract it now — a single `AgentCall` class with `transfer()`, `pause_recording()`, `resume_recording()`, `hang_up()` methods ends up as the voice-agent orchestration surface in mod-111.
