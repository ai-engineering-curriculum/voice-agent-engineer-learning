# 02. Reading and Mutating an SDP Offer / Answer

## Why this chapter

SDP is the one layer of WebRTC that is actually plain text. When a call misbehaves — the wrong codec got negotiated, FEC isn't on, the `sendrecv` you expected is `recvonly`, Opus is running at 16 kbps and sounding terrible — the SDP is the scene of the crime. You can read it, grep it, and, when the SDK you're using forces you to, munge it.

This chapter teaches you to read an offer/answer end to end and gives you the small set of mutations that are reasonable to do in production.

## SDP, line by line

SDP is defined in [RFC 8866](https://datatracker.ietf.org/doc/html/rfc8866) (which obsoletes RFC 4566). Every SDP is a line-oriented text blob where each line starts with a one-letter key:

```
v=0
o=- 4611732775322794380 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0
a=msid-semantic: WMS stream0
m=audio 9 UDP/TLS/RTP/SAVPF 111 103 104 9 0 8 106 105 13 110 112 113 126
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:F7g1
a=ice-pwd:x9cml/YzichV2+XlhiMu8g
a=fingerprint:sha-256 ...
a=setup:actpass
a=mid:0
a=sendrecv
a=rtcp-mux
a=rtpmap:111 opus/48000/2
a=fmtp:111 minptime=10;useinbandfec=1
a=rtpmap:103 ISAC/16000
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
```

| Line | Meaning |
|---|---|
| `v=0` | SDP version |
| `o=` | Origin — session ID, version, address. Mostly cosmetic for WebRTC |
| `s=`, `t=` | Session name, timing. Opaque for WebRTC |
| `a=group:BUNDLE 0` | All media streams in the group `0` share one UDP 5-tuple (saves ICE work) |
| `m=audio 9 UDP/TLS/RTP/SAVPF 111 103 ...` | **Audio media line.** Protocol is DTLS-SRTP (SAVPF). Payload types are listed in preference order. |
| `c=` | Connection address placeholder (actual address is signalled via ICE) |
| `a=ice-ufrag`, `a=ice-pwd` | ICE session credentials |
| `a=fingerprint` | DTLS certificate fingerprint that the peer will check |
| `a=setup:actpass` | DTLS role — "act or pass, I don't care" |
| `a=mid:0` | Media-stream ID that matches the `BUNDLE` group |
| `a=sendrecv` | Direction — we both send and receive |
| `a=rtcp-mux` | RTP and RTCP share one port |
| `a=rtpmap:111 opus/48000/2` | Payload type 111 is Opus at 48 kHz, 2 channels (encoder capability, not forced stereo) |
| `a=fmtp:111 ...` | Format parameters for payload type 111 |
| `a=rtpmap:0 PCMU/8000` | Payload type 0 is µ-law |
| `a=rtpmap:8 PCMA/8000` | Payload type 8 is A-law |

The one unintuitive part: the `m=` line is **the preference list for this session**, and the `a=rtpmap:` lines merely define what each number means. The peer compares the two preference lists (offer and answer), picks the highest match, and both sides commit to it. That's the "negotiation" in offer/answer.

## Offer/answer and the direction attributes

An offer/answer round-trip ([RFC 3264](https://datatracker.ietf.org/doc/html/rfc3264)):

1. One side builds an SDP **offer**. It lists every codec and direction it is willing to speak.
2. The other side builds an **answer** — the subset it will actually use.
3. Both sides reconfigure to match the answer.

Direction attributes on an `m=` line:

- `a=sendrecv` — I will both send and receive this media.
- `a=sendonly` — I will send but not receive. Common for TTS-only playout.
- `a=recvonly` — I will receive but not send. Common for a listen-only room.
- `a=inactive` — I will do neither. Used to hold a slot open for later renegotiation.

The direction is **per-direction**: if the offer says `sendonly`, a valid answer says `recvonly`, not `sendonly`. Mis-mirroring a direction is one of the two or three most common hand-rolled SDP bugs (silent calls where both sides think *they* are the sender).

## The Opus `fmtp` line — the knobs that matter

For Opus, `fmtp` is where the knobs live. The full list is in [RFC 7587](https://datatracker.ietf.org/doc/html/rfc7587) (RTP payload format for Opus); the ones you will actually touch:

| Parameter | Effect | Typical value |
|---|---|---|
| `minptime` | Minimum packetisation time in ms | `10` (so the peer may send 10 ms packets; browsers default to 20) |
| `maxaveragebitrate` | Ceiling (bps) the encoder may run at | `32000` for mono speech, `64000` for wideband |
| `useinbandfec` | Request in-band Forward Error Correction | `1` under loss, `0` otherwise |
| `usedtx` | Request Discontinuous Transmission | `0` for voice-agent (see mod-102 Ch. 4) |
| `stereo` | Request stereo decoder | `0` for mono voice |
| `sprop-stereo` | Advertise sender capability | `0` for mono voice |
| `cbr` | Constant bitrate | `0` (VBR default) |
| `maxplaybackrate` | Receiver-side resampling ceiling | Rarely set — defaults to 48 kHz |

A production Opus `fmtp` for a voice agent typically reads:

```
a=fmtp:111 minptime=10;useinbandfec=1;usedtx=0;stereo=0;sprop-stereo=0;maxaveragebitrate=32000
```

That line captures: "mono, 32 kbps ceiling, FEC on for loss resilience, DTX off so silence timing is preserved for the turn-taker, allow 10 ms packets if the peer needs lower latency."

## Reading an SDP in code

You don't need a library to answer most questions; grep works. But for production, a real parser is cleaner:

```python
# `sdp-transform` (Python port of the Node library) or `aiortc`'s built-in parser.
from aiortc.sdp import SessionDescription

sdp_text = pc.localDescription.sdp  # whatever you got
sd = SessionDescription.parse(sdp_text)
for m in sd.media:
    print(m.kind, m.port, m.direction, m.profile)
    for pt, rt in m.rtp.items():
        print(' ', pt, rt.name, rt.clockRate, rt.parameters)
```

Node / browser:

```javascript
import sdpTransform from 'sdp-transform';   // npm i sdp-transform
const parsed = sdpTransform.parse(pc.localDescription.sdp);
for (const media of parsed.media) {
  console.log(media.type, media.direction, media.rtp.map(r => r.codec));
  for (const fmtp of media.fmtp) console.log(fmtp.payload, fmtp.config);
}
// Mutate, then write back:
const rewritten = sdpTransform.write(parsed);
await pc.setLocalDescription({ type: 'offer', sdp: rewritten });
```

## When is it OK to munge SDP?

The browser's position is "don't munge, use the real API": prefer `RTCRtpSender.setParameters()`, codec preferences via `setCodecPreferences()`, and the `getCapabilities()` surface. In practice you still mutate SDP for two legitimate reasons:

1. **Forcing specific Opus parameters that the API doesn't expose.** `useinbandfec=1`, `maxaveragebitrate` — browser APIs have been slow to add these, and SDP munging is the canonical workaround.
2. **Interop with a non-browser peer** (SIP gateway, SFU) that sends SDP you have to rewrite to match your server's expectations.

The things you should **not** munge:

- `a=fingerprint`, `a=ice-*`, `a=setup` — rewriting these breaks the DTLS handshake.
- `a=group:BUNDLE` — rewriting breaks multi-stream bundling.
- `a=msid` or `a=ssrc` — rewriting these decorrelates the track from the application-level stream ID.
- The order and completeness of the `m=` lines — the browser assumes the `m=` sections you pass back in `setLocalDescription` match the ones it generated.

## A small mutation example — enabling FEC

The pre-ChromeAPI pattern to force FEC was:

```javascript
const offer = await pc.createOffer();
const mutated = offer.sdp.replace(
  /(a=fmtp:\d+ .*)/g,
  (line) => line.includes('useinbandfec') ? line : `${line};useinbandfec=1`
);
await pc.setLocalDescription({ type: 'offer', sdp: mutated });
```

This is the sort of regex surgery you will see in older WebRTC codebases. For anything more structured, use a parser; for anything new, prefer the real API when it's available.

## A small mutation example — pinning a codec

Say the browser offers `opus 111, isac 103, pcmu 0, pcma 8` and your SFU only wants Opus. Pre-`setCodecPreferences`, you'd rewrite the `m=` line to just `111` and strip the other `rtpmap` / `fmtp` lines. Modern code:

```javascript
const transceiver = pc.addTransceiver('audio');
const { codecs } = RTCRtpSender.getCapabilities('audio');
transceiver.setCodecPreferences(codecs.filter(c => c.mimeType === 'audio/opus'));
```

That emits SDP with only Opus; no string surgery.

## The negotiation gotchas that bite people

- **"My SDP looks right in the offer but the fmtp I set is missing in the answer."** Many peers echo only the parameters they chose. The ground truth is the **answer**, not the offer.
- **"The call connects but there's no audio."** Nine times out of ten this is a direction mismatch (both sides `sendonly`, or one side `inactive`). Print `a=sendrecv`-style lines first.
- **"Audio sounds bad after a while."** Congestion-control dropped the bitrate below your `minptime`-implied floor, or the browser disabled FEC when loss dropped. The runtime stats (Chapter 7) will show this.
- **"My server-side peer offers G.722 and the browser chooses it instead of Opus."** Browsers prefer the first codec in the offer; if your server lists G.722 before Opus in `m=`, you'll get G.722. Reorder.
- **"My call works in Chrome but not Firefox."** Historically the two browsers differ in how aggressively they normalise SDP (BUNDLE, unified-plan, msid semantics). Current browsers converge on unified-plan; older code that assumed plan-b still breaks. <!-- needs-research: confirm whether any production browser as of the current Chromium / Firefox release still runs plan-b by default; last known removal was Chrome M93 -->

## Summary

- SDP is a line-oriented text format; every `m=` block is a media stream with a preference list of codecs.
- The `fmtp` line is where Opus knobs live — `useinbandfec`, `usedtx`, `maxaveragebitrate`, `stereo`.
- Direction attributes (`sendrecv` / `sendonly` / `recvonly` / `inactive`) are per-direction and must mirror in the answer.
- Use `setCodecPreferences()` and `setParameters()` when you can; fall back to parsed-SDP mutation when the API doesn't expose the knob.
- Never rewrite the ICE, fingerprint, or BUNDLE lines.

Next chapter: operating a media server (LiveKit, Daily, Jitsi, Janus, Pion) and the SFU vs. MCU trade.
