# Resources for mod-107-telephony-webrtc-transport

Primary sources only. Cite these in your work; reach for secondary tutorials only when the primary source is unreadable.

## WebRTC and Media Capture (Chapters 1, 2, 7)

### W3C specifications

- **WebRTC 1.0: Real-Time Communication Between Browsers** — [W3C Recommendation](https://www.w3.org/TR/webrtc/). The JavaScript API surface — `RTCPeerConnection`, `RTCDataChannel`, `RTCRtpSender`, `RTCRtpTransceiver`.
- **Media Capture and Streams** — [W3C Candidate Recommendation](https://www.w3.org/TR/mediacapture-streams/). `getUserMedia`, `MediaStreamTrack`, device constraints.
- **Web Audio API** — [W3C Recommendation](https://www.w3.org/TR/webaudio/). `AudioWorklet`, `AnalyserNode`, graph-based audio processing.
- **Identifiers for WebRTC's Statistics API (webrtc-stats)** — [W3C Working Draft](https://www.w3.org/TR/webrtc-stats/). The authoritative reference for every `getStats()` field.
- **Media Capture Transforms** — [W3C Working Draft](https://www.w3.org/TR/mediacapture-transform/). Access to decoded audio/video frames in a `MediaStreamTrack`.

### IETF RFCs cited in Chapters 1 and 2

- **SDP (Session Description Protocol)** — [RFC 8866](https://datatracker.ietf.org/doc/html/rfc8866). Obsoletes RFC 4566. Line-by-line syntax reference.
- **Offer/Answer Model with SDP** — [RFC 3264](https://datatracker.ietf.org/doc/html/rfc3264). The offer/answer semantics you munge in exercise 02.
- **RTP: A Transport Protocol for Real-Time Applications** — [RFC 3550](https://datatracker.ietf.org/doc/html/rfc3550). RTP header fields, RTCP, sender reports, receiver reports.
- **RTP Profile for Audio and Video Conferences** — [RFC 3551](https://datatracker.ietf.org/doc/html/rfc3551). Static payload type assignments (PT 0 = PCMU, PT 8 = PCMA, PT 9 = G.722).
- **Opus audio codec** — [RFC 6716](https://datatracker.ietf.org/doc/html/rfc6716). Internal modes (SILK/CELT), frame sizes, bitrate ranges.
- **RTP Payload Format for Opus** — [RFC 7587](https://datatracker.ietf.org/doc/html/rfc7587). SDP `fmtp` parameters: `useinbandfec`, `usedtx`, `minptime`, `maxaveragebitrate`, `stereo`.
- **SRTP (Secure Real-time Transport Protocol)** — [RFC 3711](https://datatracker.ietf.org/doc/html/rfc3711). SRTP key derivation and encryption.
- **DTLS Extension to Establish Keys for the SRTP Protocol** — [RFC 5764](https://datatracker.ietf.org/doc/html/rfc5764). How WebRTC bootstraps SRTP keys over DTLS on the media 5-tuple.
- **ICE (Interactive Connectivity Establishment)** — [RFC 8445](https://datatracker.ietf.org/doc/html/rfc8445). Candidate gathering, pair priority, connectivity checks.
- **STUN (Session Traversal Utilities for NAT)** — [RFC 8489](https://datatracker.ietf.org/doc/html/rfc8489). The server-reflexive discovery protocol.
- **TURN (Traversal Using Relays around NAT)** — [RFC 8656](https://datatracker.ietf.org/doc/html/rfc8656). The relay fallback.
- **Trickle ICE** — [RFC 8838](https://datatracker.ietf.org/doc/html/rfc8838). Streaming candidates to the peer as they are gathered.
- **A Framework for Session Negotiation (unified-plan)** — [RFC 8829](https://datatracker.ietf.org/doc/html/rfc8829). The current WebRTC-specific JSEP semantics.
- **Client-to-Mixer Audio Level Indication** — [RFC 6464](https://datatracker.ietf.org/doc/html/rfc6464). RTP header extension for per-packet audio level; `inbound-rtp.audioLevel`.

### MDN and vendor reference

- **MDN WebRTC API** — [developer.mozilla.org/en-US/docs/Web/API/WebRTC_API](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API). First-stop reference for the browser API surface.
- **MDN Media Capture and Streams API** — [developer.mozilla.org/en-US/docs/Web/API/Media_Capture_and_Streams_API](https://developer.mozilla.org/en-US/docs/Web/API/Media_Capture_and_Streams_API).
- **webrtcHacks** — [webrtchacks.com](https://webrtchacks.com/). Practitioner blog covering real-world WebRTC edge cases. Secondary source; cross-check against the RFC.

## Media servers (Chapter 3)

- **LiveKit** — [livekit.io](https://livekit.io/); docs [docs.livekit.io](https://docs.livekit.io/). Open-source Apache 2 SFU + Agents framework. Source: [github.com/livekit/livekit](https://github.com/livekit/livekit).
- **LiveKit Agents** — [docs.livekit.io/agents](https://docs.livekit.io/agents/). Python / Node framework for server-side voice agents joining a LiveKit room.
- **LiveKit SIP** — [docs.livekit.io/sip](https://docs.livekit.io/sip/). Native SIP ingress/egress to a LiveKit room.
- **Daily** — [daily.co](https://www.daily.co/); docs [docs.daily.co](https://docs.daily.co/). Managed SFU.
- **Pipecat** — [github.com/pipecat-ai/pipecat](https://github.com/pipecat-ai/pipecat); docs [docs.pipecat.ai](https://docs.pipecat.ai/). Frame-oriented voice-agent framework; Daily is the canonical transport.
- **Jitsi** — [jitsi.org](https://jitsi.org/). Open-source Apache 2 SFU; JVB + Jicofo + Jigasi (SIP gateway).
- **Janus** — [janus.conf.meetecho.com](https://janus.conf.meetecho.com/). Open-source (GPLv3) general-purpose WebRTC gateway with SIP, SFU, streaming, recording plugins.
- **Pion** — [pion.ly](https://pion.ly/); source [github.com/pion/webrtc](https://github.com/pion/webrtc). Pure-Go WebRTC library used in LiveKit, ion-sfu, and bespoke servers.
- **aiortc** — [aiortc.readthedocs.io](https://aiortc.readthedocs.io/); source [github.com/aiortc/aiortc](https://github.com/aiortc/aiortc). Pure-Python WebRTC implementation.
- **coturn** — [github.com/coturn/coturn](https://github.com/coturn/coturn). The reference open-source TURN server.
- **REST API for TURN (Uberti draft)** — [draft-uberti-behave-turn-rest-00](https://datatracker.ietf.org/doc/html/draft-uberti-behave-turn-rest-00). The de-facto HMAC-based short-lived TURN credential pattern.

## SIP (Chapter 4)

### IETF RFCs

- **SIP (Session Initiation Protocol)** — [RFC 3261](https://datatracker.ietf.org/doc/html/rfc3261). The core SIP spec.
- **SIP Response for Session Transfer** — [RFC 3515](https://datatracker.ietf.org/doc/html/rfc3515) (REFER method) + [RFC 5589](https://datatracker.ietf.org/doc/html/rfc5589) (call-transfer-explicit).
- **SIP INFO Method** — [RFC 2976](https://datatracker.ietf.org/doc/html/rfc2976) (obsolete) / [RFC 6086](https://datatracker.ietf.org/doc/html/rfc6086) (current).
- **SIP over WebSocket** — [RFC 7118](https://datatracker.ietf.org/doc/html/rfc7118). The transport the browser uses to speak SIP.
- **SIP Digest Authentication** — [RFC 7616](https://datatracker.ietf.org/doc/html/rfc7616). The modernised HTTP Digest spec that SIP reuses.
- **SIP Identity (STIR)** — [RFC 8224](https://datatracker.ietf.org/doc/html/rfc8224), [RFC 8225](https://datatracker.ietf.org/doc/html/rfc8225), [RFC 8588](https://datatracker.ietf.org/doc/html/rfc8588) (PASSporT extension for SHAKEN).

### Softphones and test endpoints

- **Linphone** — [linphone.org](https://www.linphone.org/). Open-source softphone; multi-platform.
- **Zoiper** — [zoiper.com](https://www.zoiper.com/). Commercial softphone with free tier.
- **MicroSIP** — [microsip.org](https://www.microsip.org/). Windows softphone built on pjsip.
- **pjsip / pjsua** — [pjsip.org](https://www.pjsip.org/); docs [docs.pjsip.org](https://docs.pjsip.org/). Reference C SIP stack with a CLI (`pjsua`) suitable for test calls.
- **sngrep** — [github.com/irontec/sngrep](https://github.com/irontec/sngrep). The SIP ladder-diagram tool you install on an SBC.
- **Wireshark** — [wireshark.org](https://www.wireshark.org/). Full packet capture with SIP and RTP dissectors.

### Soft-switches and PBXes

- **Asterisk** — [asterisk.org](https://www.asterisk.org/); source [github.com/asterisk/asterisk](https://github.com/asterisk/asterisk). Open-source PBX with `chan_pjsip`.
- **FreeSWITCH** — [signalwire.com/freeswitch](https://signalwire.com/freeswitch); source [github.com/signalwire/freeswitch](https://github.com/signalwire/freeswitch).
- **Kamailio** — [kamailio.org](https://www.kamailio.org/). Open-source SIP server / proxy, often as the signalling front door for Asterisk or FreeSWITCH.

## PSTN providers (Chapter 5)

### Hyperscale and CPaaS

- **Twilio Programmable Voice** — [twilio.com/docs/voice](https://www.twilio.com/docs/voice). TwiML reference, REST APIs.
- **Twilio Media Streams** — [twilio.com/docs/voice/media-streams](https://www.twilio.com/docs/voice/media-streams). Bidirectional µ-law 8 kHz WebSocket spec.
- **Twilio ConversationRelay** — [twilio.com/docs/voice/conversationrelay](https://www.twilio.com/docs/voice/conversationrelay). WebSocket-first API for LLM-driven voice agents.
- **Twilio Voice Insights** — [twilio.com/docs/voice/insights](https://www.twilio.com/docs/voice/insights). Per-call MOS, jitter, loss telemetry.
- **Twilio STIR/SHAKEN** — [twilio.com/docs/voice/trusted-calling-with-shakenstir](https://www.twilio.com/docs/voice/trusted-calling-with-shakenstir). Attestation levels and provider implementation.
- **Twilio Elastic SIP Trunking** — [twilio.com/docs/sip-trunking](https://www.twilio.com/docs/sip-trunking). SIP interconnect to PSTN without TwiML orchestration.
- **Vonage Voice API** — [developer.vonage.com/en/voice/voice-api/overview](https://developer.vonage.com/en/voice/voice-api/overview). NCCO (JSON DSL), WebSocket connect, REST APIs.
- **Amazon Connect** — [docs.aws.amazon.com/connect](https://docs.aws.amazon.com/connect/). Contact-centre-first voice product.
- **Amazon Chime SDK** — [docs.aws.amazon.com/chime-sdk](https://docs.aws.amazon.com/chime-sdk/). The media layer under Connect.
- **Azure Communication Services** — [learn.microsoft.com/en-us/azure/communication-services](https://learn.microsoft.com/en-us/azure/communication-services/). Microsoft's CPaaS; Call Automation + Audio Streaming for voice agents.
- **Azure AI Speech** — [learn.microsoft.com/en-us/azure/ai-services/speech-service](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/). The paired STT/TTS layer.
- **Plivo** — [plivo.com/docs/voice](https://www.plivo.com/docs/voice/). Twilio-shaped alternative.
- **Sinch** — [developers.sinch.com/docs/voice](https://developers.sinch.com/docs/voice/). Carrier-grade global provider.
- **Zoom Phone** — [developers.zoom.us/docs/zoom-phone](https://developers.zoom.us/docs/zoom-phone/). PBX-shaped.

### SIP-trunk-focused providers

- **Bandwidth** — [dev.bandwidth.com](https://dev.bandwidth.com/). US-first carrier with developer APIs.
- **Telnyx** — [developers.telnyx.com](https://developers.telnyx.com/). Developer-first carrier with price-competitive origination.
- **SignalWire** — [developer.signalwire.com](https://developer.signalwire.com/). Twilio-compatible API with SIP trunking.

### Compliance surfaces

- **STIR/SHAKEN overview (FCC)** — [fcc.gov/call-authentication](https://www.fcc.gov/call-authentication). US regulator's framing.
- **A2P 10DLC (The Campaign Registry)** — [campaignregistry.com](https://www.campaignregistry.com/). The US 10-digit-long-code registration ecosystem (SMS-oriented, voice-adjacent).
- **Toll-Free Verification** — [twilio.com/docs/messaging/compliance/tollfree-verification](https://www.twilio.com/docs/messaging/compliance/tollfree-verification). Twilio's user-facing registration guide.
- **GDPR (official text)** — [gdpr-info.eu](https://gdpr-info.eu/). The EU General Data Protection Regulation; recording consent lives in Articles 6 and 7.
- **PCI DSS** — [pcisecuritystandards.org](https://www.pcisecuritystandards.org/). Standards for payment-card data handling; voice-recording implications in section 3.
- **HIPAA (HHS)** — [hhs.gov/hipaa](https://www.hhs.gov/hipaa/). US healthcare data regulation.

## DTMF and in-call operations (Chapter 6)

- **RTP Payload for DTMF (telephone-event)** — [RFC 4733](https://datatracker.ietf.org/doc/html/rfc4733). Obsoletes RFC 2833. The in-band RTP event format WebRTC emits.
- **Twilio DTMF handling** — [twilio.com/docs/voice/media-streams/websocket-messages](https://www.twilio.com/docs/voice/media-streams/websocket-messages) (Media Streams DTMF event); [twilio.com/docs/voice/twiml/gather](https://www.twilio.com/docs/voice/twiml/gather) (TwiML `<Gather>`).
- **Twilio Call recording** — [twilio.com/docs/voice/api/recording](https://www.twilio.com/docs/voice/api/recording).
- **Twilio `<Pay>`** — [twilio.com/docs/voice/api/payment-resource](https://www.twilio.com/docs/voice/api/payment-resource). PCI-safe card capture.
- **Twilio webhooks security** — [twilio.com/docs/usage/webhooks/webhooks-security](https://www.twilio.com/docs/usage/webhooks/webhooks-security). Signature verification.
- **Vonage Call Recording** — [developer.vonage.com/en/voice/voice-api/features/recording](https://developer.vonage.com/en/voice/voice-api/features/recording).
- **Amazon Connect secure IVR input** — [docs.aws.amazon.com/connect/latest/adminguide/secure-input.html](https://docs.aws.amazon.com/connect/latest/adminguide/secure-input.html).

## WebRTC stats and observability (Chapter 7)

- **W3C WebRTC Stats** — [W3C spec](https://www.w3.org/TR/webrtc-stats/). (Also listed above.)
- **chrome://webrtc-internals docs (Chrome developer site)** — [chromium.googlesource.com/external/webrtc/+/HEAD/g3doc/native-code/webrtc-internals.md](https://chromium.googlesource.com/external/webrtc/+/HEAD/g3doc/native-code/webrtc-internals.md).
- **about:webrtc (Firefox docs)** — [firefox-source-docs.mozilla.org/dom/media/webrtc/Debugging.html](https://firefox-source-docs.mozilla.org/dom/media/webrtc/Debugging.html). <!-- needs-research: confirm current canonical URL for Firefox about:webrtc documentation at authoring time -->
- **Prometheus** — [prometheus.io/docs](https://prometheus.io/docs/).
- **OpenTelemetry** — [opentelemetry.io/docs](https://opentelemetry.io/docs/).
- **Grafana** — [grafana.com/docs](https://grafana.com/docs/).
- **Linux `tc netem` reference** — [man7.org/linux/man-pages/man8/tc-netem.8.html](https://www.man7.org/linux/man-pages/man8/tc-netem.8.html). Impairment injection for Part C of exercise 07.

## Books and long-form

- **"WebRTC: APIs and RTCWEB Protocols of the HTML5 Real-Time Web"** — Alan B. Johnston, Daniel C. Burnett. 2012-era but still readable as an orientation piece.
- **"SIP: Understanding the Session Initiation Protocol"** — Alan B. Johnston. Fourth edition is the standard reference text if you need more SIP depth than Chapter 4 and the RFCs give you.

## Citation hygiene

- Vendor pricing pages, PoP maps, STIR/SHAKEN rules, and recording-consent laws change often. When a specific number matters, cite the dated page or the paper, and archive your evidence.
- RFCs and ITU recommendations are authoritative for protocol capability claims. Vendor blogs summarise; they do not define.
- MOS / jitter / loss numbers in provider marketing material are not peer-reviewed. Treat them as marketing until you reproduce them on your own traffic (which is what exercises 05 and 07 ask you to do).
