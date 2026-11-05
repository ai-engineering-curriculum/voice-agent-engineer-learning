# exercise-04: Opus and G.711 Codec Decision Drill

**Estimated effort:** 1 hour · **Output:** a filled codec-decision matrix for three concrete architectures + a short SDP-reading note

## Objective

Make the codec decisions from [Chapter 4](../04-codecs-opus-g711-pcm16.md) concrete by filling out the codec-and-knobs matrix for three real voice-agent architectures, and by reading enough SDP to recognise what a WebRTC negotiation actually looks like.

This is a paper exercise. No code required; the point is to be able to answer "what codec at which seam, with which knobs" without hedging.

## Prerequisites

- [Chapter 4 — Audio Codecs: Opus, G.711, PCM16](../04-codecs-opus-g711-pcm16.md).
- Familiarity with [RFC 6716 (Opus)](https://datatracker.ietf.org/doc/html/rfc6716) — Section 2 is enough.
- Loose familiarity with SDP (`m=audio`, `a=rtpmap`, `a=fmtp` lines). If you've never seen SDP, read [MDN: SDP in WebRTC](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Protocols#session_description_protocol_sdp) for 10 minutes first.

## Steps

### Part A — Three architectures (30 minutes)

For each of the three architectures below, fill out a table with the following columns at *each leg* (where a leg is a hop between two components):

| Leg | Direction | Codec | Sample rate | Channels | Knobs (FEC / DTX / bitrate) | Why |

**Architecture 1 — Browser-to-agent (WebRTC native).** User on a laptop with a browser running your voice-agent SPA; your media server (LiveKit or Pipecat on Daily) terminates WebRTC; ASR via Deepgram WebSocket at 16 kHz PCM; LLM; TTS via ElevenLabs at 24 kHz; mix back to the browser.

**Architecture 2 — PSTN inbound (Twilio Media Streams).** Caller dials an E.164 number provisioned on Twilio; Twilio establishes a WebSocket to your server and streams µ-law 8 kHz PCM in both directions; same ASR / LLM / TTS stack as Architecture 1; audio returns through the same WebSocket.

**Architecture 3 — Hybrid: mobile app (native WebRTC) with a telephony side-car.** User on an iOS app with a native WebRTC SDK; the agent can hand off mid-conversation to a human agent reachable only via a SIP phone (so the second leg is a PSTN call bridge). The voice stack runs the ASR/LLM/TTS loop with the user *before* the handoff.

For each architecture, you should end up with 6–10 legs. Include the ingress, the internal processing seams, and the egress.

### Part B — Pick the knobs (15 minutes)

4. For each Opus leg in your three architectures, decide:

   - **Target bitrate** (give a number, not a range).
   - **FEC** — on or off, and under what packet-loss assumption.
   - **DTX** — on or off, and what downstream component the decision is coupled to.
   - **Application mode** (`OPUS_APPLICATION_VOIP` vs. `OPUS_APPLICATION_AUDIO`) — justify.

5. For each G.711 leg, note µ-law vs. A-law and the region/carrier implication.

### Part C — SDP reading (15 minutes)

6. Below is a simplified Opus offer in an SDP answer. Read it and answer four questions underneath:

   ```
   m=audio 49170 UDP/TLS/RTP/SAVPF 111 103 0 8
   a=rtpmap:111 opus/48000/2
   a=fmtp:111 minptime=10;useinbandfec=1;usedtx=0;stereo=0;maxaveragebitrate=32000
   a=rtpmap:103 isac/16000
   a=rtpmap:0 PCMU/8000
   a=rtpmap:8 PCMA/8000
   ```

   - (a) Which codecs did this endpoint offer, in preference order?
   - (b) What Opus configuration is being requested? (FEC / DTX / stereo / bitrate ceiling)
   - (c) What does `minptime=10` constrain?
   - (d) If the peer accepts only PCMA, which codec will the session use, and what sample rate will the Opus-negotiated 48 kHz path never see?

   Write your answers in two or three sentences each.

## Starter guidance

- The transcoder between Opus@48k and µ-law@8k is a *component*, not a seam — it has its own ingress leg (Opus@48k PCM in), an internal resample, and an egress leg (µ-law@8k out). Document it as such.
- In Architecture 3, the handoff is a non-trivial moment: the agent's audio no longer needs to reach the user, and the user's audio no longer needs to reach the agent. Your codec table should reflect which legs tear down and which stand up.
- Opus's `maxaveragebitrate` is a ceiling, not a target. The encoder may run below it under VBR; congestion control may push it lower. Don't treat it as a point value.
- When PSTN is involved, the line "the carrier owns echo" means: do not add AEC server-side. Reflect that in your "why" column.

## Acceptance criteria

- [ ] Three architecture tables, each with 6–10 legs, columns filled.
- [ ] Opus bitrate / FEC / DTX / application-mode decisions explicit for every Opus leg.
- [ ] µ-law vs. A-law choice justified for every G.711 leg.
- [ ] Four answers to the SDP-reading questions, demonstrating you can map an `fmtp` line to a codec decision.
- [ ] A one-paragraph self-check: "which of my three architectures is most expensive in CPU, and why" (the transcoder legs are a hint).

## Stretch goals

- **Add a fourth architecture: speech-to-speech provider.** User on a browser (WebRTC), server bridges to OpenAI Realtime over a provider WebSocket. What codec is used on the provider leg today, and which knobs does the provider expose? Check current docs. <!-- needs-research: current OpenAI Realtime transport-level codec defaults and knobs at the time of the exercise -->
- **Draw the transcoder in detail.** For Architecture 2's return path (TTS → µ-law out), diagram the three internal stages: decode-from-24 kHz, downsample-to-8 kHz, re-encode-to-µ-law. Where in the stage does clipping become a risk? Where does low-pass filtering have to happen?
- **Negotiate against yourself.** Write a minimal SDP offer that your Architecture 1 server would send, listing Opus with FEC and DTX. Then write an answer as the browser would send it. Compare `m=audio` lines; note which attributes survive the negotiation.

## Keep it

These tables are reusable as the "codec" row in future module-101 Chapter-5 pipeline-design documents. Pin the three tables as reference; you'll copy them into real architecture briefs with only minor edits.
