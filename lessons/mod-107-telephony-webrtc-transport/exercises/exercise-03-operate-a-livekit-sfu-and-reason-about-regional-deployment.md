# exercise-03: Operate a LiveKit SFU and Reason About Regional Deployment

**Estimated effort:** 3 hours · **Output:** a running LiveKit room with a browser + agent participant, a captured latency measurement, and a two-page design brief for regional deployment

## Objective

Translate [Chapter 3](../03-media-servers-sfu-mcu.md) into muscle memory by **running a real SFU**, joining it from a browser and a server-side agent, and then **designing** the regional / cascading deployment you would use for a hypothetical multi-region product. The exercise is deliberately half operational, half architectural — in production you will spend time on both.

## Prerequisites

- [Chapter 3 — WebRTC Media Servers](../03-media-servers-sfu-mcu.md).
- Docker + docker compose, or the [LiveKit CLI](https://docs.livekit.io/home/cli/cli-setup/) + `livekit-server`.
- Python 3.11+ with `pip install livekit livekit-agents` (or Node + `@livekit/agents` if you prefer).
- A LiveKit Cloud account is **optional** — the local `--dev` mode is enough for the operational part.

## Steps

### Part A — Run a LiveKit server locally (30 minutes)

1. Start `livekit-server` in dev mode:
   ```bash
   livekit-server --dev
   ```
   Note the API key / secret it prints. Do **not** use dev credentials in production.

2. Join the server with the [LiveKit Meet example page](https://meet.livekit.io/) pointed at your local URL. Open two browser windows. Confirm audio flows both ways.

3. In one window, open `chrome://webrtc-internals`. Record:
   - The selected ICE candidate pair. Expect `host/host` (localhost).
   - The transport protocol. Expect UDP.
   - The codec negotiated. Expect Opus at 48 kHz.
   - The SFU's SSRC.

### Part B — Join as a server-side agent (45 minutes)

4. Build a small agent worker that joins the same room as a participant, subscribes to the browser's audio track, and prints basic frame statistics (frames received per second, sample rate, duration). The [LiveKit Agents quickstart](https://docs.livekit.io/agents/quickstart/) is a starting template; strip it down to `on_audio_frame` printing.

5. Confirm from the browser's perspective that the agent is an additional participant in the room. Confirm from the agent's perspective it is receiving audio frames from the browser.

6. Add a second server-side task to the agent: generate a 440 Hz sine wave as a `RemoteAudioTrack`-equivalent and publish it back to the room. The browser should now hear the sine wave in addition to the other browser participant.

7. In the browser, use `getStats()` to confirm two `inbound-rtp` audio streams — one from each remote participant (the other browser + the agent). Capture the SSRCs and `audioLevel` per stream.

### Part C — Force a TURN relay (30 minutes)

8. Reconfigure the browser's `RTCConfiguration` to point at a TURN server you control (local `coturn` is fine) and set `iceTransportPolicy: 'relay'` to force relay. Rejoin the room.

9. Capture the new selected ICE pair (expect `relay/relay`), the extra latency (compare `currentRoundTripTime` to the direct-path baseline), and the total bytes relayed in a 60-second call.

10. Compute: at this byte rate, what would 10,000 call-minutes / month of relay cost at $0.10/GB egress? Show the arithmetic.

### Part D — Regional deployment design brief (75 minutes)

This is a paper exercise. Produce a two-page design brief for the following scenario:

> **Scenario.** Your voice-agent product has three target regions: US east coast, Western Europe, and Southeast Asia. You expect ~70 % of traffic in US east, ~20 % in EU, ~10 % in SEA. Peak concurrency is 1,000 calls. Each call has one browser user + one agent worker. The agent worker runs a GPU-less LLM-API-calling process, so it can be anywhere there is a compute region. ASR + TTS are provider APIs, both multi-region.

Answer, in prose + diagrams:

11. **Where do you deploy SFUs?** One cluster per region, or one global cluster? Justify with Chapter 3's latency math.

12. **Where do you place agent workers for each call?** Pin to the user's region, pin to a central region, follow-the-user-with-handoff? Justify.

13. **Do you need cascading?** Give the specific user-to-user scenarios (bot stays silent while two participants talk) where cascading matters, and the voice-agent-specific scenarios where it probably doesn't.

14. **Pick your SFU product.** LiveKit Cloud with multi-region built in, self-hosted LiveKit with regional clusters, Daily, or something else. Justify against ops staffing (you have 1 full-time SRE who also owns the LLM-backend reliability work).

15. **Pick your TURN strategy.** Hosted (Twilio NTS, Xirsys, Metered) or self-hosted `coturn`? What goes in each region?

16. **Draw the data-flow diagram** for a single call in each of the three regions. Call out which hops cross a WAN and which stay in-region.

17. **Estimate the bandwidth and cost at peak.** 1,000 concurrent calls × 32 kbps × 2 directions × assumed TURN share (give a number). What's the monthly egress bill at $0.08–$0.12/GB? Show the arithmetic.

## Starter guidance

- `livekit-server --dev` is a single-node mode; it is enough for the operational parts but cannot demonstrate cascading. The design-brief part is explicitly about what you *would* run in production.
- The agent worker joining the room pattern is documented in [LiveKit Agents docs](https://docs.livekit.io/agents/). Pipecat + Daily is a parallel implementation if you prefer that stack; the brief should pick one and defend it.
- For the TURN cost math, assume 32 kbps Opus mono bidirectional; 1 call-hour ≈ 29 MB per direction ≈ 58 MB total. Verify this against your actual bytesSent measurement from Part C.
- The design brief does not need to be beautiful — it needs to be defendable. A reviewer should be able to read it in 10 minutes and either agree with your choices or ask specific follow-ups.

## Acceptance criteria

- [ ] Local LiveKit server running; two browsers in a room exchanging audio; `chrome://webrtc-internals` snapshot captured.
- [ ] Server-side agent joined as a participant; audio frames received and logged; sine wave published back and audible in the browser.
- [ ] TURN-relayed call demonstrated; `relay/relay` ICE pair recorded; bytes relayed measured.
- [ ] Monthly relay-cost arithmetic for 10k call-minutes / month at a specified egress rate.
- [ ] Two-page design brief answering Q11–Q17 with prose + at least one diagram.
- [ ] A bibliography line for each product / standard cited in the brief — LiveKit docs page, RFCs, cloud pricing pages.

## Stretch goals

- **Compare LiveKit vs. Daily operationally.** Join the same two browsers to a Daily room via the Daily SDK and the LiveKit room via the LiveKit SDK; compare the SDP, the `getStats()` output, and the dashboards. Write one page on what you would prefer and why.
- **Simulate loss.** Use `tc netem` on a Linux host to add 3 % packet loss between the browser and the SFU. Observe how LiveKit's audio congestion-control reacts (bitrate drops, FEC kicks in). Capture before/after bitrate curves.
- **Deploy to a real second region.** Rent a $5 VPS in a different continent; run `livekit-server` there; add `--rtc.turn_servers` pointing at your `coturn`. Join from both regions; measure actual observed latency.
- **Audit the dev-mode credentials.** Grep the LiveKit source for the dev JWT secret; confirm it is a fixed value. Write a one-paragraph "why you must rotate this before production."

## Keep it

The deployment brief is a reusable artefact. The next time you spin up a voice-agent product and someone asks "which region do we host in?", your answer is already written — update the traffic shares and the economics, keep the reasoning.
