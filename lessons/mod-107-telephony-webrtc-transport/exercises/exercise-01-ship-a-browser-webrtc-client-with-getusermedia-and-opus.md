# exercise-01: Ship a Browser WebRTC Client with `getUserMedia` and Opus

**Estimated effort:** 3 hours · **Output:** a working browser page + a server-side peer that echoes your voice back, plus a short write-up of the ICE / DTLS / SRTP you observed

## Objective

Build the smallest possible end-to-end WebRTC path — browser mic in, server peer, server echoes audio back to the browser — and verify every layer from [Chapter 1](../01-webrtc-end-to-end.md) actually did what the spec says it did.

You will finish this exercise able to point at any one line in a real WebRTC trace and say "that's the ICE candidate" / "that's the DTLS fingerprint" / "that's the Opus payload type," and to recognise the common failure modes (`NotAllowedError`, one-way audio, ICE stuck).

## Prerequisites

- [Chapter 1 — WebRTC End-to-End](../01-webrtc-end-to-end.md).
- Node.js 20+ **or** Python 3.11+ for the server. If Python, `pip install aiortc`; if Node, use the `wrtc` package or run a small LiveKit room.
- A modern Chrome or Firefox. Chrome is recommended because `chrome://webrtc-internals` is the richest introspection tool.
- Serve the client over `https://localhost` (self-signed TLS or `mkcert`) or plain `http://localhost` — `getUserMedia` requires one of these.

## Steps

### Part A — Build the browser client (45 minutes)

1. Create a single-page app with:
   - An "**Allow microphone**" button that calls `navigator.mediaDevices.getUserMedia({ audio: true })`.
   - A status pane that logs each `RTCPeerConnection` event: `onicecandidate`, `oniceconnectionstatechange`, `onconnectionstatechange`, `onicegatheringstatechange`, `onsignalingstatechange`, `ontrack`.
   - An `<audio id="remote" autoplay>` element that plays back whatever the server sends.
   - A **"Start call"** button that creates an offer, sends it to the server, applies the answer, and trickles ICE candidates.

2. Use **plain WebSocket** for signalling. Keep it dumb: `{ type: "offer" | "answer" | "ice", ... }`.

3. Capture the `audioLevel` and `audioLevel`-related settings from the first `MediaStreamTrack` and log the `getSettings()` result alongside what you asked for. Note any differences between requested and actual sample rate.

### Part B — Build the server peer (45 minutes)

4. Build a minimal server-side peer. Two paths:

   - **Python / aiortc:** implement a `RTCPeerConnection` that accepts the browser's offer, adds an echo track that reads inbound audio and writes it back out (aiortc ships `MediaRelay`), and replies with an answer.
   - **LiveKit room (easier for most learners):** run `livekit-server --dev` locally; join the room with both the browser (via the LiveKit JS SDK) and a server-side worker (via `livekit-agents` or `livekit-server-sdk-python`) that subscribes to the browser's track and republishes it.

5. Point the browser at your server's WebSocket signalling (or at the LiveKit room URL). Confirm the "echo" works: you speak, the server returns the audio, you hear yourself.

### Part C — Observe the layers (60 minutes)

6. Open `chrome://webrtc-internals` and drill into the active peer connection. Find:
   - The full offer and answer SDP.
   - The **DTLS fingerprint** on both sides and confirm they match what each side sent in SDP.
   - The ICE candidate pairs — specifically, the **selected pair**. Record its `localCandidateType` and `remoteCandidateType` (expect `host/host` for localhost).
   - The `inbound-rtp` and `outbound-rtp` stats; confirm the codec is `audio/opus`.

7. Dump a `chrome://webrtc-internals` JSON via the "Create Dump" button. Save it as `part-c-dump.json`.

8. Force a TURN relay. Stand up a local `coturn` (or a hosted one with short-lived credentials), configure the client with it as the *only* `iceServer`, block UDP egress to force TCP/TLS fallback (or use a `turns:` URL on 443), and verify the selected pair is now `relay/relay`.

### Part D — Break it deliberately (30 minutes)

9. Reproduce three failure modes and capture what the browser surfaces:
   - **`NotAllowedError`** — refuse the mic permission; show the exact error text.
   - **ICE failed** — point the client at a nonexistent STUN/TURN server; wait for the connection state to transition to `failed`. Note how long it took.
   - **One-way audio** — remove the `pc.ontrack` handler on the client (or silence the server's outbound track). Confirm packets *are* flowing on the server side but nothing is heard.

10. For each, write down: the user-visible symptom, the event (or lack of event) in your status pane, and the field in `chrome://webrtc-internals` that confirms the diagnosis.

## Starter guidance

- The simplest browser-side signalling library is nothing — a raw `WebSocket` and a JSON string. Don't reach for Socket.IO or signalling frameworks for this exercise.
- `RTCPeerConnection` gathers its candidates asynchronously. Don't block on `createOffer()`'s return — set the local description, then trickle candidates as they arrive.
- If you go the LiveKit route, follow the [Agents quickstart](https://docs.livekit.io/agents/quickstart/) — the simplest "republish an inbound track" is a few lines.
- For aiortc, [the `server.py` example](https://github.com/aiortc/aiortc/tree/main/examples/server) is a reasonable starting template; strip it down to audio-only.
- Use `openssl rand -hex 16` to generate TURN shared secrets if you self-host `coturn`.

## Acceptance criteria

- [ ] Browser → server echo works: you hear your voice back with roundtrip <200 ms on localhost.
- [ ] `chrome://webrtc-internals` dump saved; you can point at the offer, answer, fingerprint, and selected ICE pair in it.
- [ ] Codec confirmed as Opus at 48 kHz in the `codec` stat; `fmtp` line captured.
- [ ] TURN-relayed call demonstrated with `localCandidateType = relay` in the selected pair.
- [ ] Three failure-mode captures: `NotAllowedError`, ICE failed, one-way audio — each with one paragraph explaining what the browser surfaced and what field confirmed the diagnosis.
- [ ] One-page write-up: for each layer in Chapter 1's "spec stack" table, cite the specific line or field in your trace that proves you saw it (SDP line for RTP payload type, DTLS fingerprint line, ICE candidate for connectivity, Opus rtpmap for codec, etc.).

## Stretch goals

- **Replace the echo with a Web Audio-driven effect.** Pipe the inbound audio through an `AudioWorklet` that applies a low-pass filter before playback. Confirm the processing happens after decode, not before.
- **Add trickle-ICE timing.** Instrument the client to log the timestamps of `onicecandidate` and compare total time-to-first-media between trickle-on and trickle-off (remove the local candidates and attach them all at once via `setLocalDescription`). Expect >100 ms difference.
- **Add a stats collector** that logs `candidate-pair.currentRoundTripTime` and `inbound-rtp.jitter` once per second to the browser console. This is the seed for exercise 07.
- **Test on real cellular.** Open the hosted page on your phone over LTE (not Wi-Fi). Record the selected candidate pair type — expect `relay/*` if you're behind carrier-grade NAT.
- **Compare Chrome and Firefox.** Run the same client in both. Compare the generated SDP: unified-plan differences, `extmap` lines, codec preferences. Write down any differences in a short note.

## Keep it

The browser client + server peer is the smallest reproducible WebRTC testbed you'll own. You will reuse it in every subsequent exercise — add SDP munging in exercise 02, swap the SFU for LiveKit in exercise 03, add stats collection in exercise 07. Keep the repo around.
