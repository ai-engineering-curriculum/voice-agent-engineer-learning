# 07. Reading WebRTC Stats and Triaging a Degrading Call Live

## Why this chapter

A call that was fine five minutes ago is now choppy. The user complains; the bot sounds like it's talking through a jittery mobile connection; your TTS is starting to arrive after the next user turn has begun. This is where **WebRTC statistics** earn their keep — a stream of numeric reports emitted by every `RTCPeerConnection` that tells you, in real time, what the transport is doing. Reading them is a learnable skill; wiring them into the observability platform (mod-112) is a one-time setup that pays back every incident.

## The `getStats()` API

The source of truth is [W3C WebRTC Statistics Identifiers](https://www.w3.org/TR/webrtc-stats/). From JavaScript:

```javascript
const stats = await pc.getStats();
stats.forEach((report) => {
  console.log(report.type, report);
});
```

`pc.getStats()` returns a `Map<string, RTCStatsReport>`. Each entry has a `type` field. The types you will actually look at:

| Type | What it tells you |
|---|---|
| `inbound-rtp` | Packets you received from the peer (per SSRC) |
| `outbound-rtp` | Packets you sent to the peer (per SSRC) |
| `remote-inbound-rtp` | What the peer reported about packets they received *from you* — this is how you learn *their* loss / jitter |
| `remote-outbound-rtp` | What the peer reported about packets they sent to you |
| `candidate-pair` | The ICE pair currently in use (RTT, state, bytes sent/received) |
| `local-candidate` / `remote-candidate` | The individual ICE candidates |
| `transport` | DTLS / SCTP transport-level info |
| `codec` | Which codec and its configuration is actually in use |
| `media-source` | The microphone input (audio levels pre-encoder) |

## The four numbers you look at first

For a degrading voice call, four fields drive the triage:

### 1. Round-Trip Time (RTT) — `candidate-pair.currentRoundTripTime`

The RTT between you and the peer over the active ICE pair, in seconds. Updated continuously via STUN binding keepalives (and RTCP, see below).

- **<50 ms**: local LAN or same-region SFU. Excellent.
- **50–150 ms**: cross-region or decent broadband + VPN. Fine.
- **150–300 ms**: trans-continental, or congested path. Noticeably slower turn-taking but conversational.
- **>300 ms**: trans-oceanic without cascading, or seriously congested. Barge-in feels broken.

RTT is a **lower bound** on turn-taking latency. If `currentRoundTripTime` is 300 ms, your end-to-end bot response gap can never be below 300 ms regardless of how fast ASR + LLM + TTS runs.

### 2. Jitter — `inbound-rtp.jitter`

Statistical variance of packet inter-arrival times, in seconds, computed per [RFC 3550](https://datatracker.ietf.org/doc/html/rfc3550) §6.4.1.

- **<10 ms**: healthy.
- **10–30 ms**: noticeable; the jitter buffer eats most of it; may hear occasional stutters.
- **>30 ms**: degraded; the buffer grows (adding latency) or packets get dropped (adding loss).

Jitter above 30 ms on a call you expect to be smooth almost always means a congested link or a device-side power-save event. For mobile, it is often the cellular modem pausing to hand off between towers.

### 3. Packet loss — `inbound-rtp.packetsLost` and `remote-inbound-rtp.packetsLost`

Two viewpoints:

- `inbound-rtp.packetsLost` — packets *you* didn't receive from the peer.
- `remote-inbound-rtp.packetsLost` — packets *the peer* didn't receive from you (reported back via RTCP).

For a one-way complaint ("the agent sounds bad to me but I sound fine to the agent"), the two viewpoints diverge:

- Loss on **your** inbound → the peer's upstream is dropping packets or the network between them and you is lossy. You can't fix this at your end; you can only ask Opus to recover more (FEC, PLC) via SDP tweaks.
- Loss on the **peer's** inbound → your upstream is lossy or your network outbound is congested.

Rule of thumb: >1% loss is audible; >3% loss sounds broken.

### 4. Remote-inbound delta — the one people forget

`remote-inbound-rtp` is reported via RTCP Sender Reports and comes with a **round-trip-time estimate from the peer's perspective** and a delay-since-last-SR. The two viewpoints should match — if your local `currentRoundTripTime` disagrees significantly with the peer's `remote-inbound-rtp.roundTripTime`, you have asymmetric congestion (one path is much slower than the other).

## A minimal in-browser stats collector

```javascript
async function collectStats(pc) {
  const stats = await pc.getStats();
  const snapshot = {};
  stats.forEach((r) => {
    if (r.type === 'candidate-pair' && r.state === 'succeeded' && r.nominated) {
      snapshot.rtt = r.currentRoundTripTime;
      snapshot.availableOutgoingBitrate = r.availableOutgoingBitrate;
      snapshot.bytesSent = r.bytesSent;
      snapshot.bytesReceived = r.bytesReceived;
    }
    if (r.type === 'inbound-rtp' && r.kind === 'audio') {
      snapshot.inbound = {
        jitter: r.jitter,
        packetsLost: r.packetsLost,
        packetsReceived: r.packetsReceived,
        totalAudioEnergy: r.totalAudioEnergy,
        audioLevel: r.audioLevel,
        concealedSamples: r.concealedSamples,
        concealmentEvents: r.concealmentEvents,
      };
    }
    if (r.type === 'outbound-rtp' && r.kind === 'audio') {
      snapshot.outbound = {
        packetsSent: r.packetsSent,
        bytesSent: r.bytesSent,
        targetBitrate: r.targetBitrate,
      };
    }
    if (r.type === 'remote-inbound-rtp' && r.kind === 'audio') {
      snapshot.remoteInbound = {
        jitter: r.jitter,
        packetsLost: r.packetsLost,
        roundTripTime: r.roundTripTime,
        fractionLost: r.fractionLost,
      };
    }
    if (r.type === 'codec' && r.mimeType?.startsWith('audio/')) {
      snapshot.codec = {
        mimeType: r.mimeType,
        payloadType: r.payloadType,
        clockRate: r.clockRate,
        sdpFmtpLine: r.sdpFmtpLine,
      };
    }
  });
  return snapshot;
}

setInterval(async () => {
  const s = await collectStats(pc);
  ingestToBackend(s);
}, 2000);
```

Two-second cadence is a reasonable default. Faster (500 ms) if you want to drive real-time UI; slower (5 s) if you're only logging for post-hoc analysis.

## Computing the deltas that matter

Counts (`packetsSent`, `packetsLost`, `bytesSent`, etc.) are **cumulative**. To get a rate, subtract a prior snapshot:

```javascript
const packetsPerSec = (curr.outbound.packetsSent - prev.outbound.packetsSent) / dtSeconds;
const lossRate = (curr.inbound.packetsLost - prev.inbound.packetsLost)
               / Math.max(1, (curr.inbound.packetsReceived - prev.inbound.packetsReceived));
const kbps = ((curr.outbound.bytesSent - prev.outbound.bytesSent) * 8) / 1000 / dtSeconds;
```

Opus at 32 kbps with 20 ms packets is 50 packets/second and ~32 kbps. If your observed rate is radically different from the SDP-advertised `maxaveragebitrate`, congestion-control has intervened or the encoder downgraded.

## The two concealment metrics

`concealedSamples` and `concealmentEvents` on `inbound-rtp` tell you how much audio the decoder *synthesised* to cover missing packets (PLC, see mod-102 Chapter 4).

- A rising `concealmentEvents` counter is a smoking gun for inbound loss or jitter — even if `packetsLost` hasn't moved, the jitter buffer may have declared packets late.
- `concealedSamples / totalSamplesReceived` gives a "fraction of audio the user heard that wasn't real audio." >1% is noticeable; >5% sounds broken.

## The audio-level metrics

`media-source.audioLevel` and `inbound-rtp.audioLevel` are **moving-window loudness in [0,1]** per [RFC 6464](https://datatracker.ietf.org/doc/html/rfc6464). Useful for:

- **"Is the mic actually picking up anything?"** If `media-source.audioLevel` is 0, the user isn't being captured — mic muted, browser permission revoked, or a hardware issue.
- **"Is the far end actually talking?"** If `inbound-rtp.audioLevel` is 0 but packets are arriving, the peer is sending silence frames (DTX, or a dead microphone upstream).

## A debugging checklist

A call just went bad. Walk through:

1. **Is `candidate-pair.state == 'succeeded'`?** If no — ICE problem. Jump to Chapter 1.
2. **Is the active pair `relay/relay`?** You're going through TURN; expected if ICE fell back, but check if it *shouldn't* be (should be `host/host` for same-LAN).
3. **What is `currentRoundTripTime`?** Compare to the baseline for this region.
4. **Is `jitter` rising?** Yes → congestion or wifi handoff.
5. **Is `packetsLost` rising?** Yes → compare `inbound-rtp` vs `remote-inbound-rtp` to find the direction.
6. **Is `concealmentEvents` rising?** Yes → the user is *perceiving* loss or late packets.
7. **Did the bitrate drop?** Check `outbound-rtp.targetBitrate` and `candidate-pair.availableOutgoingBitrate`. A drop from 32 kbps to 8 kbps is a bandwidth-estimator reaction.
8. **Is the codec what you expect?** `codec.mimeType` should be `audio/opus`; if it says `audio/PCMA`, you fell back to A-law — Chapter 2 negotiation went wrong.
9. **Compare client view to server view.** The media server has its own stats surface; mismatches reveal which half of the path is broken.

## `chrome://webrtc-internals`

The browser debugging tool: open `chrome://webrtc-internals` in a new tab while a call is live. You get a per-peerconnection view with:

- Graphs of every numeric stat over time.
- The full SDP offer and answer.
- The ICE candidate pairs, including which were tried and rejected.
- The createOffer/createAnswer legalese and errors.

Hit **"Create Dump"** to save a JSON blob you can share in an incident post-mortem. This is the single most useful WebRTC debugging artefact in Chrome. Firefox's equivalent is `about:webrtc`. <!-- needs-research: confirm current path and features of `chrome://webrtc-internals` and `about:webrtc` against latest releases at authoring time -->

## Server-side stats

The browser is one of two endpoints. Your media server has its own view:

- **LiveKit** exposes [Analytics API](https://docs.livekit.io/home/analytics/) and emits per-participant quality webhooks. <!-- needs-research: verify current analytics API path and semantics against latest LiveKit docs -->
- **Daily** exposes per-call quality metrics via its API.
- **Pion / aiortc**: expose their own stats interfaces at the library level.
- **Janus** has an `admin` HTTP API with per-session packet counters.

The useful pattern: **collect both ends, ship to one place**. If you only have the client view, you miss server-side loss; if you only have the server view, you miss client-side device issues.

## Integrating with the observability platform

mod-112 covers the full picture. The minimum integration from here:

1. **Each call emits a `call_id`-tagged stream of stats snapshots** at the chosen cadence.
2. **A metrics pipeline (Prometheus / OpenTelemetry / Datadog) aggregates** by region, provider, codec, etc.
3. **Alerting fires** on p95 loss > 3%, p95 jitter > 50 ms, p95 RTT > 300 ms *per region*, rolling window.
4. **A trace per call** correlates the stats stream with the ASR / LLM / TTS traces, so a slow call has a single `call_id` you can pivot across.
5. **chrome://webrtc-internals dumps** are archived when a user reports a bad call, along with the transcript. Store 30 days by default.

The thing to push back on: **dashboards that aggregate without tagging regions, providers, or codec are useless**. A global p50 jitter number is noise. The question is always "what changed and where?" — region, provider, codec, time.

## Summary

- `pc.getStats()` is the live stream of transport metrics; `inbound-rtp`, `outbound-rtp`, `remote-inbound-rtp`, `candidate-pair` are the entries you read.
- Four numbers: RTT, jitter, packet loss (both directions), concealment events.
- Deltas matter more than cumulative counts — snapshot every 2s and diff.
- `chrome://webrtc-internals` and `about:webrtc` are the browser debugging tools of record.
- Ship both client and server stats to the observability platform, tagged by region and provider.

Next module: [mod-108-safety-compliance-voice](../mod-108-safety-compliance-voice/) covers the compliance surface the recording decisions from Chapter 6 opened up.
