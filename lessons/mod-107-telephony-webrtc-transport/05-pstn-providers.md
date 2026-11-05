# 05. PSTN Ingress / Egress Providers

## Why this chapter

PSTN — the Public Switched Telephone Network — is the thing you don't want to build. Numbering, origination, routing, local-number portability, STIR/SHAKEN caller-ID attestation, 911 compliance, international termination rate negotiation: all of these are regulated carrier concerns, and the price of walking into the regulator's office is "I now operate a carrier." The providers in this chapter exist so you don't have to.

Your job is to **pick the one that fits**, wire it to your agent, and know enough of the compliance surface to not get your numbers blocked.

## The provider landscape

The main voice providers you will encounter:

- **[Twilio Programmable Voice](https://www.twilio.com/docs/voice)** — the market-leading developer-first voice provider. TwiML (XML DSL) + Media Streams (bidirectional µ-law 8 kHz WebSocket) + [ConversationRelay](https://www.twilio.com/docs/voice/conversationrelay) (the 2024-era WebSocket API shaped for LLM-first voice agents, with built-in TTS / STT). SIP trunking available.
- **[Vonage Voice API](https://developer.vonage.com/en/voice/voice-api/overview)** — NCCO (JSON DSL) + [WebSocket connect](https://developer.vonage.com/en/voice/voice-api/concepts/websockets). Comparable developer surface to Twilio, different pricing curve.
- **[Amazon Connect](https://aws.amazon.com/connect/) + [Chime SDK](https://aws.amazon.com/chime/chime-sdk/)** — AWS's contact-center-first voice product. Connect handles IVRs, queues, routing; Chime SDK is the WebRTC + media-streaming layer that voice agents hook into via [Connect Integrations with Amazon Lex](https://docs.aws.amazon.com/connect/latest/adminguide/amazon-lex.html) or [Connect + Kinesis Video Streams](https://docs.aws.amazon.com/connect/latest/adminguide/customer-voice-streams.html) for raw audio.
- **[Azure Communication Services](https://learn.microsoft.com/en-us/azure/communication-services/) + [Azure AI Speech](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/)** — Microsoft's equivalent. Call Automation API for call control, Audio Streaming API for raw audio to the agent.
- **[Plivo](https://www.plivo.com/docs/voice/)** — developer-first, Twilio-shaped API, historically price-competitive on international termination.
- **[Sinch](https://developers.sinch.com/docs/voice/)** — carrier-grade global provider, strong in enterprise.
- **[Zoom Phone](https://developers.zoom.us/docs/zoom-phone/)** — PBX-shaped; typically paired with Zoom's contact-center product rather than used as a raw voice API, but available.

There are also **CPaaS-adjacent providers** (Bandwidth, Telnyx, Signalwire) that lean into SIP trunking with a cleaner developer story than the carriers themselves. These are what you reach for when you want to bring your own media server and treat the provider purely as PSTN interconnect.

## The seven axes of selection

### 1. Latency to region

A call from the UK → US provider → UK user is ~160 ms RTT on the carrier leg alone before you've done anything. Pick a provider with points of presence (PoPs) in your users' region. Twilio, Vonage, Sinch are global; Amazon Connect and Azure Communication map to AWS / Azure regions; Plivo is strongest in India + US; Zoom Phone's PoP map is tied to Zoom's data-center footprint. <!-- needs-research: confirm current PoP coverage per provider at authoring time -->

Practical test: provision a number in the target region, call it from the region, measure one-way audio latency with a timed test tone. Repeat across time of day.

### 2. Origination compliance

If you're placing calls (outbound), you cannot just dial E.164 numbers. The rules:

- **US / Canada: STIR/SHAKEN** (SIP Identity, [RFC 8588](https://datatracker.ietf.org/doc/html/rfc8588); the Telephone Robocall Abuse Criminal Enforcement and Deterrence Act in the US requires it). Your provider attests to the identity of the caller — A-attestation (fully attested, best), B-attestation (partially attested), C-attestation (gateway attested, worst — treated as likely spam by terminating carriers). A-attestation is only available when the originating number is one you fully control; borrowed numbers typically get B or C.
- **US toll-free SMS / voice: [Toll-Free Verification](https://www.twilio.com/docs/messaging/compliance/tollfree-verification)** or the provider's equivalent. For voice, "verification" means your use case and sender identity are registered with the Responsible Organization (RespOrg); unverified toll-free voice is increasingly treated as spam by terminating carriers.
- **US local 10-digit: [A2P 10DLC](https://www.twilio.com/docs/messaging/compliance/a2p-10dlc)** registration, primarily for SMS but relevant because voice-agent numbers often need SMS too.
- **EU: GDPR for call recording consent** (next chapter); carrier KYC requirements per country.
- **India: [TRAI DLT](https://www.plivo.com/docs/sms/getting-started/dlt/)** registration of templates and sender IDs, though this is SMS-side; voice regulations differ.
- **Country-specific caller-ID rules.** Many European carriers strip caller-ID on international originations if the number isn't local. France and Germany now enforce "originating CLI must be a France/Germany number" for domestic deliveries. <!-- needs-research: confirm current CLI enforcement rules per country at authoring time -->

The provider abstracts most of this *if* you register the required paperwork through their portal. Budget days-to-weeks of lead time for toll-free verification and 10DLC on a new deployment.

### 3. Per-minute cost

Order of magnitude (US local, as of recent years — check current pricing):

- Inbound: $0.0085 / min
- Outbound: $0.013 / min
- Toll-free inbound: $0.022 / min

International termination rates vary wildly — $0.02/min for most of Europe, $0.10-$0.50/min for parts of Africa and Central Asia. If you expect meaningful international traffic, compare at least two providers. <!-- needs-research: cite each provider's current rate card at authoring time -->

The arithmetic of **who pays whom** is worth internalising:

- Inbound: the calling party's carrier pays termination to your provider; you pay your provider a per-minute fee that covers that termination plus margin.
- Outbound: you pay your provider, which pays origination + termination fees on your behalf.
- Media Streams / WebSocket / ConversationRelay: often billed *in addition* to voice minutes. Check the provider's audio-streaming line item.

### 4. In-session recording controls

Does the provider support **start/stop recording mid-call** under your control, with per-leg or whole-call granularity? Twilio's [Recording APIs](https://www.twilio.com/docs/voice/api/recording) do; Vonage's [Call Recording](https://developer.vonage.com/en/voice/voice-api/features/recording) does; others vary. You also need:

- **Dual-channel vs. single-channel** — store caller and agent on separate channels so you can run STT independently on each.
- **Encryption at rest** — you own the key or the provider does. Twilio offers provider-side encryption; some enterprise contracts allow customer-managed keys.
- **Retention controls** — PCI, HIPAA, and GDPR all demand different retention stories.

### 5. Programmability shape

How do you tell the provider "pick up, play a greeting, stream audio to my server"?

- **Twilio TwiML** — XML DSL. `<Connect><Stream url="wss://..."/></Connect>` is the shape for Media Streams.
- **Twilio ConversationRelay** — WebSocket-first; sends STT transcripts as JSON events and receives TTS directives. Latency-optimised for agent loops.
- **Vonage NCCO** — JSON DSL. `{"action":"connect","endpoint":[{"type":"websocket", "uri":"wss://..."}]}`.
- **Amazon Connect contact flows** — graphical/JSON workflow; Lex blocks for NLU, invoke-Lambda for custom logic, Kinesis for raw audio.
- **Azure Call Automation** — REST APIs + webhook events. More code, less DSL.
- **SignalWire, Telnyx, etc.** — most lean on Twilio-compatible dialects or vanilla SIP trunking.

For a voice-agent team, the decision is usually: "do we want the provider's TTS/STT integration, or are we bringing our own?" If you're bringing your own, raw bidirectional audio (Media Streams, WebSocket connect, Kinesis) is what you need and ConversationRelay-style bundled integrations are nice-to-have. If you want fewer moving parts, the bundled integrations (ConversationRelay, Amazon Connect + Lex) are faster to ship but tie you to the provider's model choices.

### 6. SIP trunking option

If you want to run your own media server (Chapter 3) and only use the provider for PSTN interconnect:

- Twilio Elastic SIP Trunking, Vonage SIP trunks, Plivo SIP Trunk, Bandwidth, Telnyx, SignalWire — all offer SIP trunks you register in their portal and point your SBC or LiveKit SIP at. You pay only for minutes (and optionally for the trunk itself).
- Trade-off: you take on the SIP ops burden (Chapter 4) in exchange for lower per-minute cost and more flexibility. Reasonable choice once you have a dedicated telephony-ops person.

### 7. Observability and quality telemetry

The provider should expose per-call quality traces:

- Twilio: [Voice Insights](https://www.twilio.com/docs/voice/insights) — per-call MOS, jitter, packet-loss traces.
- Vonage: [Call event logs + webhook events](https://developer.vonage.com/en/voice/voice-api/webhook-reference).
- Amazon Connect: CloudWatch Logs + [Contact Trace Records](https://docs.aws.amazon.com/connect/latest/adminguide/contact-trace-records.html).

The ones you care about are the ones that let you correlate the provider's view of the call with your in-agent view (Chapter 7 — WebRTC stats, mod-112 — observability).

## The quick decision table

| Need | Pick |
|---|---|
| Fast developer start, broad coverage, docs-first | Twilio |
| Twilio-shaped API at a lower international rate | Plivo or Telnyx |
| Already in AWS, want contact-center workflow built in | Amazon Connect + Chime SDK |
| Already in Azure, Microsoft shop | Azure Communication Services |
| Enterprise / carrier-grade, global rollout | Sinch or Bandwidth |
| Want LLM-first integration bundled | Twilio ConversationRelay |
| Want to own the SFU, use the provider as a SIP trunk only | Telnyx, SignalWire, Bandwidth (or any Elastic SIP Trunking SKU) |

<!-- needs-research: validate this decision table against current provider feature parity at authoring time -->

## The provider-abstraction choice, revisited

The decision tree:

1. **Need to ship next week?** → Twilio + ConversationRelay, or Amazon Connect contact flows.
2. **Have an existing stack with STT/TTS/LLM already wired?** → Media Streams or WebSocket connect; keep the stack.
3. **Will you scale to >1 M minutes / month?** → Benchmark two providers on your actual traffic pattern (see Chapter 3 on TURN economics and this chapter's §3 on per-minute cost); the margins compound.
4. **Enterprise with ops staff?** → SIP trunk + your own media server. More work, more control.

No provider is "best" — the decisive axis is usually "which provider does your compliance / procurement org already have a contract with?" Save the technical differentiators for when you have a choice.

## Summary

- Don't build PSTN; pick a provider.
- Latency, origination compliance, per-minute cost, recording controls, programmability, SIP-trunk option, observability — seven axes of comparison.
- US/Canada require STIR/SHAKEN A-attestation to avoid spam flags; toll-free and 10DLC need registration.
- "Bundled" (ConversationRelay, Connect + Lex) ships fast; "raw audio" (Media Streams, WebSocket) is more flexible.
- Benchmark on actual regional latency and actual international termination rates — provider sales material is marketing, not measurement.

Next chapter: DTMF, transfer, warm handoff, and recording controls — the in-call operations your provider exposes.
