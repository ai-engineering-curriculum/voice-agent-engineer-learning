# 01. WebRTC End-to-End: From `getUserMedia` to Opus-over-SRTP

## Why this chapter

WebRTC is the transport most voice agents use from a browser or a mobile app. It is also the transport that hides the most behind one `RTCPeerConnection` object. If the call cannot connect, cannot find audio, gets dropped by a corporate firewall, or quietly re-keys mid-session, you need a mental model of *every* layer — media capture, codec, RTP, DTLS, SRTP, ICE, STUN, TURN — because the browser's error messages assume you already have one.

This chapter walks the stack top-to-bottom, naming each specification so you can go to the primary source when debugging, and ends with a working minimal client pattern.

## The spec stack

WebRTC is the integration of several specs the IETF and W3C wrote in parallel:

| Layer | Spec | What it defines |
|---|---|---|
| JavaScript API | [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/) | `RTCPeerConnection`, `RTCDataChannel`, SDP munging surface |
| Media capture | [W3C Media Capture and Streams](https://www.w3.org/TR/mediacapture-streams/) | `getUserMedia`, `MediaStreamTrack`, device constraints |
| Audio graph (client-side) | [W3C Web Audio API](https://www.w3.org/TR/webaudio/) | AudioWorklets, analyser nodes, resampling |
| Session negotiation | [RFC 8866 (SDP)](https://datatracker.ietf.org/doc/html/rfc8866) + [RFC 3264 (offer/answer)](https://datatracker.ietf.org/doc/html/rfc3264) | What codecs, what addresses, who sends to whom |
| Media transport | [RFC 3550 (RTP)](https://datatracker.ietf.org/doc/html/rfc3550) + [RFC 3551 (payload types)](https://datatracker.ietf.org/doc/html/rfc3551) | Framing audio into UDP packets with timestamps and sequence numbers |
| Security | [RFC 5764 (DTLS-SRTP)](https://datatracker.ietf.org/doc/html/rfc5764) + [RFC 3711 (SRTP)](https://datatracker.ietf.org/doc/html/rfc3711) | TLS handshake over UDP, then encrypted RTP |
| Connectivity | [RFC 8445 (ICE)](https://datatracker.ietf.org/doc/html/rfc8445) + [RFC 8489 (STUN)](https://datatracker.ietf.org/doc/html/rfc8489) + [RFC 8656 (TURN)](https://datatracker.ietf.org/doc/html/rfc8656) | NAT and firewall traversal |
| Audio codec | [RFC 6716 (Opus)](https://datatracker.ietf.org/doc/html/rfc6716) | Mandatory-to-implement audio codec in WebRTC |

The browser does all of this for you behind `new RTCPeerConnection()`. Your job is to know which of these layers is the one failing when something goes wrong.

## The permission flow: `getUserMedia`

Audio capture starts in JavaScript with a call that triggers a browser permission prompt the user has to click through:

```javascript
const stream = await navigator.mediaDevices.getUserMedia({
  audio: {
    sampleRate: 48000,
    channelCount: 1,
    echoCancellation: true,
    noiseSuppression: true,
    autoGainControl: true,
  },
  video: false,
});
```

Three things to internalise:

- **The prompt is per-origin, per-device, per-session.** Grant on `https://app.example.com` does not carry to `https://admin.example.com`. The grant persists per the browser's permission policy, not your code's.
- **HTTPS is mandatory** (localhost is the only exemption). `getUserMedia` on `http://` returns `NotAllowedError` with no prompt.
- **The constraints are hints, not demands.** The browser picks the closest-matching device configuration it can. Read the actual configuration back with `stream.getAudioTracks()[0].getSettings()` — the number you asked for may not be the number you got, which matters when you plumb the track into Web Audio at a specific sample rate.

The common failure modes:

| Error | What it means |
|---|---|
| `NotAllowedError` | User (or policy) blocked the prompt |
| `NotFoundError` | No matching device (headless CI, no mic installed) |
| `NotReadableError` | Device busy — another app or tab holds the mic |
| `OverconstrainedError` | Constraints are satisfiable individually but not together (rare) |
| `SecurityError` | HTTP page, or iframe without `allow="microphone"` |

## RTP and Opus on the wire

Once connected, the browser encodes each 20 ms of microphone PCM into one Opus packet and wraps it in an RTP header:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|V=2|P|X|  CC   |M|     PT      |       sequence number         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           timestamp                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           synchronization source (SSRC) identifier            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Opus payload                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

(adapted from [RFC 3550](https://datatracker.ietf.org/doc/html/rfc3550) §5.1).

Field by field, the ones you'll see in logs:

- **Payload type (PT)** — identifies the codec. For Opus in WebRTC it's dynamic (negotiated in SDP, typically 111). Static types are listed in [RFC 3551](https://datatracker.ietf.org/doc/html/rfc3551) — PT 0 is PCMU (µ-law), PT 8 is PCMA (A-law), PT 9 is G.722.
- **Sequence number** — monotonic per source; gaps mean packet loss, out-of-order delivery, or wrap-around.
- **Timestamp** — RTP clock rate for Opus in WebRTC is always 48000 Hz, so a 20 ms packet advances the timestamp by 960 regardless of the actual Opus internal sample rate.
- **SSRC** — the stable per-stream identifier that lets you correlate RTP with RTCP and with `getStats()` reports (Chapter 7).

Opus in WebRTC always runs at 48 kHz on the wire and always declares "stereo-capable" (even for mono), per [RFC 7587 (RTP payload format for Opus)](https://datatracker.ietf.org/doc/html/rfc7587). The actual mono/stereo decision is in the SDP `fmtp` line (Chapter 2).

## Security: DTLS-SRTP, always

WebRTC has no unencrypted mode. All media is SRTP. The SRTP keys are derived from a DTLS handshake that runs over the same UDP 5-tuple as the media itself, per [RFC 5764](https://datatracker.ietf.org/doc/html/rfc5764).

Order of events:

1. ICE picks a 5-tuple (source IP:port → destination IP:port, UDP).
2. DTLS handshake runs over that tuple. Each side authenticates with a self-signed certificate; the fingerprint of that certificate was already announced in SDP, so a MITM would have to match it.
3. The DTLS master secret is exported to feed SRTP key derivation.
4. All subsequent packets on that tuple are SRTP (media) or SCTP-over-DTLS (data channels).

What this means operationally:

- You cannot `tcpdump` a WebRTC call and read it. The SRTP payload is encrypted. You can see RTP headers, SSRCs, packet rates — enough for connectivity triage — not the audio.
- There is no "disable encryption for debugging" switch. If you want to inspect audio, use `getStats()`, add server-side recording via a media server that terminates SRTP (LiveKit, Pion, Janus), or use the browser's `chrome://webrtc-internals` dump (Chapter 7).
- Certificate fingerprints matter. If a signalling-path attacker can rewrite the SDP (replace the fingerprint), they can intercept — which is why the signalling channel itself must be authenticated end-to-end, typically TLS + bearer token.

## Connectivity: ICE, STUN, TURN

Two peers behind NAT cannot just send UDP to each other. [ICE (RFC 8445)](https://datatracker.ietf.org/doc/html/rfc8445) is the procedure that finds a working path by gathering **candidates** and trying each in order.

### Candidate types

- **Host** — the local IP:port of your network interface. Works if both peers are on the same LAN.
- **Server-reflexive (srflx)** — the public IP:port your NAT maps you to, discovered by asking a STUN server [RFC 8489](https://datatracker.ietf.org/doc/html/rfc8489). Works if your NAT is cone-like and the remote side can reach that public port directly.
- **Peer-reflexive (prflx)** — a reflexive candidate the remote side learns about during connectivity checks; bookkeeping, you won't configure it.
- **Relayed (relay)** — an allocation on a TURN server [RFC 8656](https://datatracker.ietf.org/doc/html/rfc8656) that forwards packets between the two peers. The fallback that always works (as long as both peers can reach the TURN server).

The browser gathers candidates for all enabled types, sorts them by priority (host > srflx > relay), and exchanges them with the peer through your signalling channel. Each pair of local/remote candidates is probed with STUN binding requests until one succeeds.

### STUN vs. TURN

- **STUN** tells you "what's my public IP:port?" and nothing else. One request / one response. Cheap — a public STUN server costs essentially nothing to operate.
- **TURN** allocates a server-side UDP port, and *every media packet* between the two peers is relayed through it. Expensive — it uses bandwidth proportional to your call minutes.

The economics: a one-hour WebRTC call with Opus at 32 kbps mono in both directions carries roughly `3600 × 32000 × 2 / 8 ≈ 29 MB` of audio each way, or ~58 MB per call through a TURN server. For a large deployment, this is real money. Hosted TURN services bill either by GB or by concurrent sessions; self-hosted `coturn` is cheaper per byte but costs operational attention.

### When does TURN actually get used?

TURN is engaged when neither host nor srflx candidates can reach the peer. The usual triggers:

- **Symmetric NAT** on one side — the NAT maps the same internal port to a different external port per destination, so the srflx candidate you advertised is useless to the peer.
- **Corporate firewalls** that block outbound UDP entirely (TURN over TCP/443 or TURN over TLS/443 is the workaround — same spec, different transport).
- **Carrier-grade NAT** on cellular — the mobile device doesn't have a reachable srflx.

In practice, expect [10–20 %] of your calls to relay through TURN on the open internet, and 100 % behind enterprise networks that forbid outbound UDP. <!-- needs-research: cite a current measured figure from a public WebRTC deployment (Daily, LiveKit, Twilio report blog posts) -->

### Trickle ICE

The modern pattern is **trickle ICE** [RFC 8838](https://datatracker.ietf.org/doc/html/rfc8838): send the SDP offer as soon as it's ready, then stream each candidate to the peer as it's gathered rather than waiting for all candidates. Setup time drops by hundreds of milliseconds. Every production client should trickle; most SDKs do by default.

## The minimal browser client

Below is the shape of a browser WebRTC client that captures the microphone, offers to a server, and renders the server's reply audio. It is intentionally small; real clients add reconnect logic, device-change handling, and stats collection.

```javascript
const pc = new RTCPeerConnection({
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },
    {
      urls: 'turns:turn.example.com:443?transport=tcp',
      username: '<short-lived-user>',
      credential: '<short-lived-credential>',
    },
  ],
});

// 1. Capture mic.
const localStream = await navigator.mediaDevices.getUserMedia({ audio: true });
localStream.getAudioTracks().forEach(t => pc.addTrack(t, localStream));

// 2. Render whatever the server sends back.
const audioEl = document.querySelector('audio#remote');
pc.ontrack = (ev) => { audioEl.srcObject = ev.streams[0]; };

// 3. Trickle candidates to the server as they arrive.
pc.onicecandidate = (ev) => {
  if (ev.candidate) signaling.send({ type: 'ice', candidate: ev.candidate });
};

// 4. Offer / answer.
const offer = await pc.createOffer();
await pc.setLocalDescription(offer);
signaling.send({ type: 'offer', sdp: offer.sdp });

signaling.on('answer', async ({ sdp }) => {
  await pc.setRemoteDescription({ type: 'answer', sdp });
});
signaling.on('ice', async ({ candidate }) => {
  await pc.addIceCandidate(candidate);
});
```

The server side — whether LiveKit, Daily, Pion, aiortc, or your own — does the symmetric half. See Chapter 3 for the media-server shape.

### Credentials for TURN

Never embed a long-lived TURN password in client code. The standard pattern is **time-limited HMAC credentials** per [REST API for TURN](https://datatracker.ietf.org/doc/html/draft-uberti-behave-turn-rest-00): the server mints a `username` of the form `<expiry-unix-ts>:<user-id>` and a `credential` that is `HMAC-SHA1(username, shared-secret)` base64-encoded. The TURN server verifies in constant time, no shared database. The user has at most the credential's TTL (minutes) to abuse the account before the key expires.

## The common failure modes, named

- **"ICE failed"** — ICE gathered no working pair. Almost always a firewall: peer cannot reach your STUN/TURN or vice versa. Check `chrome://webrtc-internals` for which candidates were gathered and which pairs were tried.
- **"ICE disconnected" mid-call** — the current pair stopped working. Device went to sleep, network hopped (Wi-Fi → cellular), NAT mapping expired. ICE restart is the recovery.
- **"DTLS failed"** — connectivity succeeded but the DTLS handshake didn't. Almost always clock skew or a broken fingerprint in the SDP.
- **One-way audio** — ICE succeeded, DTLS succeeded, but packets only flow one way. A firewall is doing stateful UDP filtering and only allows the direction that was initiated first. TURN fixes it.
- **No audio at all** — the common surprise: the `MediaStreamTrack` is attached but muted. Check `track.muted` and `track.enabled`. Also check that the browser tab has focus; some browsers silently mute background-tab capture under aggressive power-saving policies.
- **Silence from one browser version to another** — the Opus `fmtp` lines don't agree (one side offered `stereo=1`, the other offered `stereo=0`, and the negotiation picked mono; or a `maxaveragebitrate` was set so low the audio is unintelligible). Chapter 2.

## Summary

- WebRTC is seven specs in a trenchcoat: W3C JS APIs, SDP, RTP, DTLS/SRTP, ICE/STUN/TURN, Opus.
- `getUserMedia` is the entrypoint; HTTPS is mandatory; constraints are hints.
- Media on the wire is always SRTP; you cannot tcpdump it.
- ICE finds a path; TURN is the fallback that costs bandwidth; trickle to cut setup time.
- The failure taxonomy is small and worth memorising: ICE failed, DTLS failed, one-way audio, silence from fmtp mismatch.

Next chapter: SDP offer/answer — the one piece of WebRTC that is actually readable text.
