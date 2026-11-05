# exercise-02: Wire Streaming TTS With Chunked Playout

**Estimated effort:** 3 hours · **Output:** a working streaming TTS client with chunked playout + an annotated TTFB measurement log + a barge-in demo

## Objective

Internalise the Chapter 2 streaming contract by **wiring a real streaming TTS client end-to-end** against at least one provider, with a player pipeline that begins playback on the first audio chunk, supports mid-stream barge-in cancel, and measures TTFB honestly. You will finish able to reach for any provider's streaming surface and have a working prototype in under an hour, with the right observability.

## Prerequisites

- [Chapter 2 — The Streaming TTS Contract and the Player Pipeline](../02-streaming-tts-contract-and-player-pipeline.md).
- A developer account with a streaming-native TTS vendor (Cartesia, ElevenLabs, PlayHT, OpenAI, or Google Cloud TTS).
- Python 3.10+ with the vendor SDK, `numpy`, `soundfile`, and an audio library (`sounddevice`, `pyaudio`, or an equivalent).
- A machine with working speakers or a headset. (An audio-device-less environment will do for the file-output variant; see Starter guidance.)
- A text file with 10–20 representative LLM-output-shaped utterances for your target product (from exercise-01 if you did that one).

## Steps

### Part A — Wire a minimal streaming client (45 minutes)

1. Open a streaming TTS session with:
   - Sample rate matched to your output sink (24 kHz is a common sweet spot).
   - Encoding: **PCM** (not MP3 / Opus). Decode-free path.
   - Format and voice-ID appropriate for your brief.
   - Alignment / character-timing events on if the vendor supports them.
   - `auto_mode=False` or the equivalent: you control when to flush.
2. Send text in a way that **begins synthesis while the input is still streaming** — simulate an LLM by chunking the input text into tokens with a small delay (`asyncio.sleep(0.03)` per chunk) and forwarding to the TTS as you go.
3. On the receive side, push audio chunks into a bounded frame queue (≤80 ms of audio buffered).
4. Drive an audio sink that drains the frame queue at the output sample rate.
5. Log **every event** the vendor emits to a JSONL file with a monotonic timestamp (ms since stream start). Include audio events (size, not payload), alignment events, errors, and end-of-stream.

### Part B — Measure TTFB and realtime factor (30 minutes)

6. For 20 utterances drawn from your text file, measure:
   - **TTFB-wall** — from "first text byte sent to the TTS" to "first audio byte reached the audio sink."
   - **TTFB-synth** — if the vendor exposes an internal latency header, record it too; otherwise, approximate as TTFB-wall minus RTT.
   - **Realtime factor (RTF)** — total audio-seconds produced divided by total wall-clock synthesis time.
   - **Partial-chunk cadence** — the inter-arrival gap between consecutive audio chunks.
7. Report p50 / p95 / p99 for TTFB-wall across utterances. Separate cold-session (new WS per utterance) and warm-session (same WS reused).
8. Plot (ASCII, image, or spreadsheet) the arrival-timeline for one longer utterance: x-axis = wall-clock, y-axis = cumulative audio-seconds-received. The slope should be >1 (faster than realtime); the y-intercept is TTFB.

### Part C — Chunked playout + prebuffer tuning (30 minutes)

9. Run the player with three prebuffer settings — 0 ms, 30 ms, 100 ms — on the same 20-utterance set.
10. For each setting, record:
    - TTFB delta vs. the 0 ms baseline.
    - Number of underruns (frame-queue starvation events).
    - Subjective judgement: does it sound smooth?
11. Add a synthetic jitter profile (`asyncio.sleep(random.uniform(0, 0.08))` between receive events) and re-run. Report the same metrics under jitter.
12. Pick the prebuffer that minimises TTFB while keeping underruns at zero under jitter. Document the choice.

### Part D — Barge-in cancel (45 minutes)

13. Add a cancel path that:
    - Listens for a keyboard event (or a sentinel audio-level event, if you want to simulate VAD).
    - On trigger: issues the vendor's mid-stream cancel, flushes the frame queue, and stops the audio sink at the next frame boundary.
14. Run the client on a long (~10 second) utterance; trigger cancel ~1 second in.
15. Measure **cancel-to-silence latency** — from cancel-sent to audio-actually-stopped. Report p50 / p95 across 10 trials.
16. Verify the vendor **actually stops billing** you for the cancelled synthesis — some vendors drain the in-flight request server-side and charge you anyway. Document the behaviour; if you cannot verify from a client, note it.

### Part E — Comparison (optional, +30 minutes)

17. Point the same client (or a minimal fork) at a second provider with the same text. Capture the same TTFB / RTF / cancel numbers.
18. Produce a side-by-side on the Chapter 2 contract axes:
    - Transport (WS / chunked HTTP / SSE / SDK-only).
    - Audio encoding knobs.
    - Flush-control granularity.
    - Alignment-event support.
    - Mid-stream cancel semantics.
    - Session-reuse behaviour (can you send multiple utterances on one WS? What's the handshake tax saved?).

## Starter guidance

- Keep the client under ~200 lines of Python. The point is to exercise the contract, not to build an abstraction layer.
- Use the vendor SDK for the first build; fall back to raw WebSocket only if the SDK hides fields you need.
- If you lack an audio device, write chunks to a WAV file. You lose the subjective-smoothness check but keep the TTFB and RTF metrics.
- Timestamp with `time.monotonic_ns()`. Wall-clock timestamps are fine for TTFB-wall; monotonic is better for intervals.
- Barge-in: implement cancel on a `Ctrl-C`-style signal first; only wire a VAD trigger if Module 106 overlap is in your comfort zone.
- If your vendor doesn't expose mid-stream cancel, note it — and then rank the vendor lower in your exercise-01 matrix if barge-in matters to your brief.

## Acceptance criteria

- [ ] Working streaming client with chunked playout under ~200 lines.
- [ ] JSONL event log per utterance, with every vendor event captured with timestamps.
- [ ] TTFB-wall p50 / p95 / p99 reported for cold and warm sessions.
- [ ] Realtime factor reported for a longer utterance.
- [ ] Prebuffer tuned empirically; justification documented.
- [ ] Barge-in cancel path implemented; cancel-to-silence latency measured over at least 10 trials.
- [ ] (Optional) Side-by-side second-provider comparison on the Chapter 2 contract axes.
- [ ] A "sharp edges" paragraph: at least one vendor behaviour you did not predict from the docs.

## Stretch goals

- **Drive the client from an LLM stream.** Point an LLM (OpenAI, Anthropic, local) at a prompt; stream its tokens into the TTS as they arrive. Measure end-to-end latency: user-turn-committed → first-TTS-byte-played. This is the production latency path.
- **Pipe into a telephony / WebRTC sink.** Instead of `sounddevice`, emit RTP frames or push into a Twilio Media Stream / LiveKit room. You now have the skeleton of a real voice-agent audio path; Module 107 will build on this.
- **Measure `cancel-to-billing-stop`** using the vendor's usage API. The unit-economics of a chatty agent with frequent barge-in is not what you think without this number.
- **Build a Module 106 preview** — wire Silero VAD on a microphone input, trigger barge-in cancel the moment the user starts speaking over the agent, and measure the user-interrupt-to-agent-silent total latency.
- **Replace PCM with Opus** in one run. Measure the per-chunk decode latency on the client. Observe that it adds to TTFB; decide whether the bandwidth savings matter for your deployment.

## Keep it

The client and the TTFB-probe harness are reusable. Every vendor evaluation is one run away from a defensible number; keep them under version control. The JSONL event logs from the first few vendors are a reference for what streaming TTS actually looks like in production.
