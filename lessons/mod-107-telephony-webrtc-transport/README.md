# mod-107-telephony-webrtc-transport: Telephony, WebRTC, and SIP Transport Engineering

**Estimated effort:** 18 hours · **Role:** Voice & Multimodal Agent Engineer · **Level:** L35

## What this module is for

This is the transport layer. Every voice agent sits on top of a bidirectional audio pipe that is either WebRTC (browsers, mobile apps), SIP (telephony, enterprise), or a provider abstraction over one of those (Twilio Media Streams, Vonage Voice API, Amazon Connect, Azure Communication Services). When that pipe misbehaves — the call doesn't connect, audio goes one-way, the codec negotiates to something you didn't expect, the TURN bill explodes, DTMF stops being delivered, a corporate firewall blocks UDP — your agent's quality falls off a cliff and the LLM-level intelligence is irrelevant.

This module gives you the **engineer's working depth** on all three transports: enough to design a stack from scratch, debug a stuck trunk, triage a degrading call live, pick a provider against the right axes, and know when to escalate to a telephony specialist.

## Learning objectives

- Author WebRTC end-to-end — W3C WebRTC 1.0 + Media Capture and Streams + Web Audio for the client; ICE + STUN + TURN for traversal; DTLS + SRTP for security; RTP ([RFC 3550](https://datatracker.ietf.org/doc/html/rfc3550)) carrying Opus ([RFC 6716](https://datatracker.ietf.org/doc/html/rfc6716)) for audio — against a browser or mobile client, with the `getUserMedia` permission flow.
- Read and modify an SDP offer / answer ([RFC 8866](https://datatracker.ietf.org/doc/html/rfc8866)); interpret the codec negotiation (opus / PCMU / PCMA), bandwidth lines, direction attributes (`sendrecv` / `sendonly` / `recvonly` / `inactive`), and the `a=fmtp` parameters that tune Opus (`useinbandfec`, `usedtx`, `minptime`, `maxaveragebitrate`).
- Operate a WebRTC media server — LiveKit (SFU + Agents), Daily (SFU + Pipecat), Jitsi, Janus, Pion — understanding SFU vs. MCU trade-offs, the cascading / regional-deployment story, and the TURN economics.
- Author SIP ([RFC 3261](https://datatracker.ietf.org/doc/html/rfc3261)) for traditional telephony endpoints — INVITE / 180 Ringing / 200 OK / BYE flow, `Via` / `Route` / `Record-Route`, authentication, registrar + proxy roles — enough to debug a stuck SIP trunk; know when a soft-switch / media gateway is doing the SIP↔WebRTC bridge.
- Wire PSTN ingress / egress via Twilio Programmable Voice + Media Streams + ConversationRelay, Vonage Voice API, Amazon Connect + Chime SDK, Azure Communication Services + Azure AI Speech, Plivo, Sinch, Zoom Phone — picking the provider against latency to region, origination compliance (toll-free verification, STIR / SHAKEN A-attestation), per-minute cost, in-session recording controls.
- Manage DTMF (in-band RTP events [RFC 4733](https://datatracker.ietf.org/doc/html/rfc4733) vs. SIP INFO vs. provider API), call transfer (REFER vs. Twilio `<Dial>`-transfer vs. REST-driven handoff), warm-handoff to human agents, call recording start / stop with consent gating, and status / webhook callbacks.
- Read WebRTC-stats API output — round-trip time (RTT), jitter, packet loss, remote-inbound delta — and triage a stuck or degrading call live; integrate the stream into the observability platform ([mod-112](../mod-112-voice-multimodal-eval-and-observability/)).

## Lecture chapters

1. [01 — WebRTC End-to-End: From `getUserMedia` to Opus-over-SRTP](01-webrtc-end-to-end.md)
2. [02 — Reading and Mutating an SDP Offer / Answer](02-sdp-offer-answer.md)
3. [03 — WebRTC Media Servers: SFU vs. MCU, Cascading, and TURN Economics](03-media-servers-sfu-mcu.md)
4. [04 — SIP for Traditional Telephony Endpoints](04-sip-for-telephony.md)
5. [05 — PSTN Ingress / Egress Providers](05-pstn-providers.md)
6. [06 — DTMF, Transfer, Warm Handoff, and Recording Controls](06-dtmf-transfer-recording.md)
7. [07 — Reading WebRTC Stats and Triaging a Degrading Call Live](07-webrtc-stats-triage.md)

## Exercises

- [exercise-01 — Ship a Browser WebRTC Client with `getUserMedia` and Opus](exercises/exercise-01-ship-a-browser-webrtc-client-with-getusermedia-and-opus.md) (3 h)
- [exercise-02 — Read and Mutate an SDP Offer / Answer](exercises/exercise-02-read-and-mutate-an-sdp-offer-answer.md) (2 h)
- [exercise-03 — Operate a LiveKit SFU and Reason About Regional Deployment](exercises/exercise-03-operate-a-livekit-sfu-and-reason-about-regional-deployment.md) (3 h)
- [exercise-04 — Debug a SIP Trunk with Trace Logs](exercises/exercise-04-debug-a-sip-trunk-with-trace-logs.md) (2 h)
- [exercise-05 — Wire Twilio Media Streams and ConversationRelay to a Voice Agent](exercises/exercise-05-wire-twilio-media-streams-and-conversationrelay-to-a-voice-agent.md) (3 h)
- [exercise-06 — DTMF, Call Transfer, and Recording Controls](exercises/exercise-06-dtmf-call-transfer-and-recording-controls.md) (2 h)
- [exercise-07 — Read WebRTC Stats and Triage a Degrading Call](exercises/exercise-07-read-webrtc-stats-and-triage-a-degrading-call.md) (2 h)

## Structure

- `01-…md` … `07-…md` — lecture chapters.
- `exercises/` — per-exercise prompts.
- `labs/` — long-form hands-on labs (none in this module; the in-depth lab-style work lives in the exercises).
- `quizzes/` — knowledge checks.
- `resources.md` — primary-source references for everything cited in this module.

## How to use this module

Read the chapters top to bottom. Chapters 1–2 build the WebRTC and SDP mental models; Chapter 3 is the media-server layer that a voice agent lives inside; Chapter 4 is the SIP side; Chapter 5 is the provider abstraction over SIP; Chapter 6 is the in-call operations that sit on top; Chapter 7 closes the loop with live debugging.

Exercises pair directly with chapters: 1↔Ch1, 2↔Ch2, and so on. **Do each exercise right after the matching chapter** rather than saving them all for the end — the WebRTC client you build in exercise 01 is what you add stats collection to in exercise 07, and the Twilio wiring in exercise 05 is what you add DTMF and recording to in exercise 06.

Several exercises (03, 05) require accounts with third-party providers (LiveKit Cloud or a self-host; Twilio). If you cannot create those accounts, the acceptance criteria call out which parts become paper-exercises instead.

Prerequisite: [mod-102-speech-pipeline-foundations](../mod-102-speech-pipeline-foundations/README.md) — in particular the codec chapter (Opus / G.711) and the chunking chapter.

Next module: [mod-108-safety-compliance-voice](../mod-108-safety-compliance-voice/README.md) — the consent / recording / PII surface introduced here deepens into the compliance obligations.
