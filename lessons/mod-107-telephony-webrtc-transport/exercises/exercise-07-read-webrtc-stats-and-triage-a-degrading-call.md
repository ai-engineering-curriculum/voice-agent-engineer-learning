# exercise-07: Read WebRTC Stats and Triage a Degrading Call

**Estimated effort:** 2 hours · **Output:** a live stats collector, three triage case studies against deliberately-induced impairments, and a prometheus/otel-ready metrics pipeline

## Objective

Operationalise [Chapter 7](../07-webrtc-stats-triage.md) by (a) building a continuous `getStats()` collector on top of the exercise-01 testbed, (b) inducing specific transport failures and triaging them from the stats stream alone, and (c) shipping the collector's output to a metrics pipeline that could plug into mod-112 observability tomorrow.

## Prerequisites

- [Chapter 7 — Reading WebRTC Stats](../07-webrtc-stats-triage.md).
- The browser + server-peer testbed from [exercise 01](exercise-01-ship-a-browser-webrtc-client-with-getusermedia-and-opus.md).
- Linux with `sudo` for `tc netem` network impairments. (macOS has `dnctl`; Windows + WSL works for the client side; the impairment host should be Linux.)
- Prometheus + Grafana locally (or Grafana Cloud free tier), or any OpenTelemetry-compatible collector.

## Steps

### Part A — Build the stats collector (45 minutes)

1. On the browser client from exercise 01, add a `setInterval(collectStats, 2000)` that:
   - Calls `pc.getStats()`.
   - Extracts the Chapter-7 fields: `candidate-pair.currentRoundTripTime`, `candidate-pair.availableOutgoingBitrate`, `inbound-rtp.jitter`, `inbound-rtp.packetsLost`, `inbound-rtp.packetsReceived`, `inbound-rtp.concealmentEvents`, `inbound-rtp.concealedSamples`, `inbound-rtp.totalSamplesReceived`, `inbound-rtp.audioLevel`, `outbound-rtp.packetsSent`, `outbound-rtp.bytesSent`, `remote-inbound-rtp.jitter`, `remote-inbound-rtp.packetsLost`, `remote-inbound-rtp.fractionLost`, `remote-inbound-rtp.roundTripTime`, `codec.mimeType`, `media-source.audioLevel`.
   - Computes **deltas** against the prior snapshot: packets-per-second, kbps sent/received, loss rate.
   - POSTs the snapshot to a `/stats` endpoint on your server, tagged with `call_id`.

2. On the server side, add a `/stats` endpoint that stores each snapshot with its `call_id` and timestamp. A time-series DB (InfluxDB), Prometheus Pushgateway, or just JSON lines on disk is enough.

3. Expose a `/metrics` endpoint in Prometheus text format with the key gauges and counters:
   - `voice_call_rtt_seconds{call_id="..."}`
   - `voice_call_jitter_seconds{direction="inbound|outbound",call_id="..."}`
   - `voice_call_packets_lost_total{direction="...",call_id="..."}`
   - `voice_call_concealment_events_total{call_id="..."}`
   - `voice_call_target_bitrate_bps{direction="outbound",call_id="..."}`

4. Scrape it with Prometheus; render the gauges in a Grafana dashboard. Panels: RTT line, jitter line, loss-rate line, concealment-event rate, bitrate line.

### Part B — Baseline measurement (15 minutes)

5. Do a 2-minute "known good" call on localhost. Save the Grafana dashboard JSON and a screenshot. This is your baseline; everything in Part C is a deviation from it.

6. Record baseline values for:
   - RTT (expect <5 ms)
   - Jitter (expect <2 ms)
   - Loss (expect 0)
   - Concealment events per second (expect 0)
   - Outbound bitrate (expect ~32 kbps for mono voice)

### Part C — Induce three impairments and triage (45 minutes)

Run `tc netem` on the host between your client and your server peer to simulate real network conditions. For each impairment, run a 60-second call, capture the stats, and write a 2-paragraph triage: what the Chapter-7 fields showed, which fields diverged first, and what the user's audio sounded like.

7. **Impairment C.1 — Added latency.** `sudo tc qdisc add dev <iface> root netem delay 200ms`. Expectations: RTT jumps by ~400 ms (round trip), jitter unchanged, loss unchanged, concealment events unchanged. Confirm or refute.

8. **Impairment C.2 — Packet loss.** `tc qdisc change dev <iface> root netem loss 3%`. Expectations: `packetsLost` rises; `concealmentEvents` rises; outbound bitrate may drop (congestion control); FEC (if enabled in exercise 02) may change the recovery picture. Confirm or refute; note whether the loss is visible more on `inbound-rtp` or `remote-inbound-rtp`.

9. **Impairment C.3 — Jitter.** `tc qdisc change dev <iface> root netem delay 50ms 30ms distribution normal`. Expectations: inbound jitter rises to ~30 ms; concealment events start. Confirm or refute.

10. Remove the impairment (`sudo tc qdisc del dev <iface> root`) and confirm the metrics recover.

### Part D — Observability integration (15 minutes)

11. Add **per-region tags** to your metrics: `region="local"`, `provider="direct"` for the current setup. Document the extension points for later adding `region="eu-west"`, `provider="livekit-cloud"`, etc.

12. Add **alerting rules** in Prometheus:
    - p95 RTT > 300 ms over 5 minutes → page.
    - p95 jitter > 50 ms over 5 minutes → warn.
    - loss rate > 3 % over 1 minute → page.
    - concealment event rate > 5 Hz over 5 minutes → warn.

13. Trigger each alert by re-running the matching impairment. Confirm the alert fires and recovers.

## Starter guidance

- The `getStats()` API returns a `Map`; iterate with `stats.forEach((report, id) => { ... })`. Do **not** rely on cross-browser field names for everything — some fields are Chrome-only (`totalSamplesReceived`, `concealmentEvents` are standard; `qpSum` is video-only; browsers evolve).
- `tc netem` only affects the interface you specify. Apply it on the host where your server runs so the impairment affects both directions evenly.
- Prometheus textfile exposition format: `# HELP ...`, `# TYPE ...`, `metric_name{labels} value`. A library (`prom-client` for Node, `prometheus_client` for Python) is easier than hand-rolling.
- `chrome://webrtc-internals` is still the fastest ad-hoc introspection tool — your Prometheus dashboard is for history and alerting, not for the first 5 minutes of triage.
- `audioLevel` is a per-packet moving window; it's not a signal-level dB in the DSP sense. Use it for "is there any speech at all," not for fine-grained loudness work.

## Acceptance criteria

- [ ] Stats collector running every 2 s; snapshots POSTed with `call_id`; stored durably.
- [ ] Prometheus `/metrics` endpoint exposing the five gauges; scraped; rendered in Grafana.
- [ ] Baseline measurements for a healthy call captured with screenshot.
- [ ] Three impairment case studies (added latency, 3% loss, jittered delay) with 2-paragraph triage each; which stat field named the problem first.
- [ ] Alerting rules in Prometheus for p95 RTT / jitter / loss / concealment, each verified by induced impairment.
- [ ] One-page "runbook" for an on-call engineer: "call is degrading, user is complaining" — step-by-step walk through the Chapter-7 checklist using *your* dashboard panels.

## Stretch goals

- **Correlate with server-side stats.** Collect the server peer's view (aiortc's `getStats()` equivalent, or LiveKit's analytics API) and overlay on the same dashboard. Which direction's view exposes a mismatch first when you induce asymmetric loss?
- **Add a client-side EL-level recording trigger.** When concealment events spike over threshold, automatically save a `chrome://webrtc-internals`-style dump plus the last 30 seconds of inbound audio. This is the "post-mortem artefact" pattern.
- **Add OpenTelemetry traces.** Each call is a trace; each `getStats()` snapshot is a span attribute. Correlate the stats trace with the ASR / LLM / TTS traces from exercise 05 so one `call_id` pivots across all telemetry.
- **Degrade gracefully.** When loss > 5 %, emit a `connection_quality:bad` event the UI can show. When loss recovers, emit `connection_quality:good`. Show the state transitions in the UI.
- **Run the same measurement against Twilio.** Place a call to your Twilio number from a browser (via Twilio Client SDK); compare the latency and loss stats to your LiveKit direct path. Where does Twilio add latency compared to a direct SFU?

## Keep it

The stats collector + alerting rules + runbook triple is the seed of your production voice observability. Carry it forward into mod-112 — that module adds the ASR / LLM / TTS traces on top, so this piece is the transport layer of the full telemetry stack.
