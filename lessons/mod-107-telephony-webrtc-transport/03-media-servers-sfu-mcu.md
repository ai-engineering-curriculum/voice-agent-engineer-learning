# 03. WebRTC Media Servers: SFU vs. MCU, Cascading, and TURN Economics

## Why this chapter

Two browsers can WebRTC each other peer-to-peer. The moment you have a third participant, a recording requirement, a server-side bot, or a bridge to telephony, you need a media server in the middle. For voice agents, the "third participant" is the agent itself — a server-side process that needs to receive the user's audio and send TTS back — so a media server is table stakes, not optional.

This chapter is about choosing and operating one.

## SFU vs. MCU

Two topologies dominate.

**SFU (Selective Forwarding Unit).** Each endpoint sends one encoded stream up; the SFU fans out copies to each receiver. The SFU does not decode, re-encode, or mix — it chooses which packets to forward and drops the rest. Modern WebRTC (LiveKit, Daily, Jitsi, Janus in routing mode, Pion-based custom servers) is SFU by default.

- **CPU per participant: cheap** (forward packets, maintain congestion-control feedback).
- **Bandwidth: N× on the server** (one upstream in, N-1 downstreams out per participant for an N-way call).
- **Quality: as good as the sender** — each receiver decodes the original encoder's output.
- **End-to-end encryption possible** — the SFU is forwarding encrypted SRTP; with SRTP double-encryption (Insertable Streams), it can forward packets it cannot decrypt.

**MCU (Multipoint Control Unit).** The server decodes every stream, mixes them into one, and encodes once per receiver. Legacy videoconferencing (Polycom, Lifesize) used MCUs.

- **CPU: expensive.** Every participant is a decode + encode.
- **Bandwidth: 1 downstream per receiver**, regardless of N.
- **Quality: capped by the server's mix.**
- **No E2E encryption** — the server has the plaintext.

For voice agents, SFU is almost always the right answer:

- Calls are usually 2 participants + 1 bot — fan-out is small, bandwidth isn't the bottleneck.
- The agent needs raw per-participant audio for ASR; a mixed down-mix is a *loss* of information, not a gain.
- Server CPU budget is better spent on the agent's model calls than on audio transcoding.

The exception: if you need to deliver a **single mixed stream** to a PSTN leg (one caller hears a mix of bot + other humans in the room), you need a mixer somewhere. LiveKit, Daily, and Pion all offer mixing APIs that run only when invoked; they do not mix by default.

## Media-server options

### LiveKit

- **[livekit.io](https://livekit.io/)** — open source (Apache 2) with a hosted cloud. Go server, Agents framework in Python / Node.
- **Why it shows up in voice-agent stacks:** [LiveKit Agents](https://docs.livekit.io/agents/) ships a Python framework that wires STT → LLM → TTS on top of a LiveKit room; the room is both the WebRTC transport *and* the control plane the agent participates in.
- **Native SIP ingress / egress:** [LiveKit SIP](https://docs.livekit.io/sip/) terminates SIP calls as participants in a room.
- **Deployment story:** regional clusters you register with the control plane; clients connect to the nearest; rooms are home-region unless you enable [room cascading](https://docs.livekit.io/home/cloud/cloud-architecture/) (cloud) or set up federation (self-hosted). <!-- needs-research: verify current cascading / federation terminology against latest LiveKit docs -->

### Daily

- **[daily.co](https://daily.co/)** — commercial SFU with a browser SDK and the open-source [Pipecat](https://github.com/pipecat-ai/pipecat) framework for voice agents.
- Pipecat is the counterpart to LiveKit Agents — Python-first, frame-oriented pipeline.
- Daily's SFU is managed; you pay per participant-minute and do not operate the servers.

### Jitsi

- **[jitsi.org](https://jitsi.org/)** — open source (Apache 2), Java-based (Jicofo + JVB). Historically video-conferencing-first.
- Standard choice if you want a self-hostable SFU for a product that is mostly multi-party video. Less idiomatic for one-to-one voice bots, but perfectly workable.

### Janus

- **[janus.conf.meetecho.com](https://janus.conf.meetecho.com/)** — open source (GPLv3), C-based, plugin-oriented.
- Flexible — plugins for SIP, SFU routing, recording, streaming — but closer to a toolkit than a product. You wire the plugins you need.

### Pion

- **[pion.ly](https://pion.ly/)** — pure-Go WebRTC implementation. Not an SFU per se; it is the library you build an SFU on. Used by [ion-sfu](https://github.com/pion/ion-sfu), [Galene](https://galene.org/), [LiveKit](https://github.com/livekit/livekit) internals, and many bespoke production servers.
- The right choice when you need WebRTC inside a Go service and don't want to pay for a full product stack.

### aiortc

- **[aiortc.readthedocs.io](https://aiortc.readthedocs.io/)** — pure-Python WebRTC implementation, async-first. Not a server product; the library you use when you want a Python service to behave like a WebRTC peer. Useful for quick experiments and for the server-side peer in a lab; production SFUs are not built on aiortc.

## Cascading and regional deployment

A naive deployment puts one SFU in one region, say `us-east-1`. Latency from Tokyo to that SFU is ~160 ms one way, so a round trip is ~320 ms and a Tokyo user talking to a US agent sees a conversational latency floor well over half a second before you count ASR / LLM / TTS.

Two patterns to push the SFU closer to the user:

### Regional home-ing

Deploy an SFU cluster per region (`us-east-1`, `eu-west-1`, `ap-northeast-1`, ...). Each room is assigned to one cluster, typically the region nearest the first joiner or explicitly pinned by the application. Participants far from the home region eat the WAN latency.

Good for **voice agents where everyone (user + agent worker) is in the same region**. Simple. All participant-to-participant routing is one hop.

### SFU cascading / federation

Deploy an SFU per region *and* let SFUs forward media to each other on behalf of their local participants. A user in Tokyo and an agent worker in us-east don't both cross the Pacific; the Tokyo SFU and the US SFU exchange one copy per stream.

- LiveKit Cloud calls this [cascaded routing / multi-home rooms](https://docs.livekit.io/home/cloud/cloud-architecture/).
- Daily, Agora, Vonage Video, Zoom all have equivalents under different names.
- Self-hosted Jitsi supports octo (meshed JVBs); self-hosted LiveKit supports regions via explicit configuration.

The latency math: a well-cascaded deployment typically holds one-way server-contributed latency to the tens-of-ms range for the regional leg and the inter-region leg each, rather than the ~160 ms WAN RTT you'd eat with a single-region SFU.

For voice agents the two decisions are: (1) **where does the agent worker run?** (ideally the same region as the user — pin it from the inbound-call region) and (2) **are you using a hosted SFU with cascading built in, or are you running it yourself?** The second decision is a staffing decision as much as a technical one.

## TURN economics

Chapter 1 introduced TURN. Here's what it actually costs:

- **Bandwidth.** 32 kbps Opus mono, both directions, for one hour of call ≈ 29 MB in and 29 MB out = 58 MB per call-hour relayed. Egress from cloud is typically $0.08–$0.12 / GB; call it ~$0.005 per relayed call-hour. <!-- needs-research: cite current major-cloud egress pricing at authoring time -->
- **Concurrency.** Hosted TURN services (Twilio, Xirsys, Metered) bill either per GB or per concurrent session. Rates vary by provider; compare against self-hosted `coturn` plus your cloud egress.
- **TCP/TLS fallback cost.** TURN over UDP is cheap; TURN over TCP on 443 is more expensive because the server holds TCP state. TURN over TLS on 443 is required for users behind corporate firewalls that only allow HTTPS.
- **Geographic placement.** A TURN server across the WAN from the user adds latency just like an SFU does. Deploy TURN per region.

Order-of-magnitude rule for sizing: assume **10–25 % of calls will relay through TURN** on the open internet, higher on enterprise or mobile networks. For a deployment doing 100k call-hours / month, that's 10k–25k relayed hours × 58 MB ≈ 580 GB–1.45 TB of relayed audio traffic, ~$50–$175 at typical egress prices, and bandwidth charges can swamp that for video. <!-- needs-research: cite a public measurement of TURN usage share from a current WebRTC provider -->

## Recording, transcription, and the server-side bot seam

A media server is also usually where you implement:

- **Recording.** The SFU terminates SRTP and writes audio (or the raw packets) to object storage. With E2E-encrypted SFUs, recording is impossible without participant consent to decrypt.
- **Server-side STT.** The agent either joins the room as a participant (LiveKit Agents, Daily / Pipecat) and consumes raw audio frames, or the media server ships audio to STT over an internal WebSocket.
- **Server-side TTS.** The agent produces PCM / Opus audio frames and injects them into the room as its own track.
- **PSTN bridge.** The SIP gateway (Chapter 4) joins the room as a participant on behalf of the phone caller. From the agent's perspective, the SIP caller looks like any other room participant.

The question "where does my bot live?" is usually answered "as a participant in the media server's room," not "as a WebSocket beside it." Framework-first (LiveKit Agents, Pipecat) makes this the default path.

## The five-question checklist for picking a media server

1. **Open source or managed?** Managed (LiveKit Cloud, Daily, Agora, Vonage) buys you ops. Open source (self-hosted LiveKit, Jitsi, Janus, Pion-based) buys you cost control and customisation at the price of ops staff.
2. **Does it have native SIP ingress?** If you need PSTN, the SFU should either terminate SIP directly (LiveKit SIP, Janus SIP plugin) or there must be a well-documented bridge.
3. **Does it support regional deployment with cascading?** Hosted: usually yes. Self-hosted: check.
4. **What are the agent-framework hooks?** LiveKit Agents, Pipecat on Daily, custom code on Janus/Pion. The hook is what makes the SFU usable from the voice-agent code rather than usable only from a browser.
5. **What is the pricing model?** Per-participant-minute (most hosted), per-GB (TURN-heavy), flat server (self-hosted). The economics flip around 10–50 kMAU, depending on vendor. <!-- needs-research: confirm current pricing tiers against vendor pages at authoring time -->

## Summary

- SFU is the default topology for voice agents; MCU is almost never what you want.
- LiveKit + Agents and Daily + Pipecat are the two dominant open-friendly stacks; Jitsi, Janus, Pion are alternatives with different trade-offs.
- Regional deployment keeps user ↔ SFU RTT low; cascading keeps user ↔ remote-participant latency low across regions.
- TURN costs bandwidth proportional to call minutes; budget ~10–25 % relay share by default.
- The agent lives inside the media server as a participant, not beside it.

Next chapter: SIP for traditional telephony endpoints and the SIP↔WebRTC bridge.
