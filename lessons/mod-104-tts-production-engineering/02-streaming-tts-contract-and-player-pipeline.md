# 02. The Streaming TTS Contract and the Player Pipeline

## Why this chapter

Streaming TTS is the second place a voice agent's latency budget lives or dies. The user has finished speaking, the LLM has committed its response, and the only thing standing between the agent and the first audible syllable is the TTS request plus its playback pipeline. Get the contract right and the user hears the agent start speaking within 150–300 ms. Get it wrong — synthesise the whole utterance, wait for the WAV to finish, then begin playback — and you have added two full seconds of dead air that no amount of model quality will fix.

This chapter is the mechanics of that contract: HTTP chunked / Server-Sent Events / WebSocket / SDK-abstracted; how to design the player pipeline so that **TTFB** (time-to-first-audio-byte) is the latency driver, not the total synthesis time; what the knobs are on the streaming endpoint; and the barge-in surface Module 106 will consume.

## The latency identity you are optimising

Write it on a sticky note. Every TTS architectural decision follows from this:

```
perceived_agent_latency = LLM_last_token_time
                        + TTS_request_setup
                        + TTS_time_to_first_audio_byte
                        + player_queue_depth
                        + transport_latency
```

The term the user feels is `TTS_time_to_first_audio_byte` (TTFB). Everything after the first byte is overlapped with playback — the mouth is already moving, so the engine can keep synthesising while the speaker drains buffered audio. Total synthesis time matters only in two cases:

1. The synthesis rate is **slower than realtime** (the engine produces one second of audio in more than one second). You will underrun the player and stutter. Any production-grade streaming TTS runs faster than realtime; verify this for your vendor on your text.
2. You need to cancel mid-utterance (barge-in, Module 106). Then the cost of a slow engine is the audio that was already synthesised and discarded.

Everything else — "our TTS is 2× realtime vs. their 4× realtime" — is throughput, not latency.

The implication: **optimise for TTFB first, and never wait for the full synthesis before starting playback.** Any pipeline shape that violates this is broken; see the anti-patterns at the end.

## The streaming transport surface

Vendors expose streaming TTS over one of four surfaces. Each has a different feel from the client side.

### HTTP chunked transfer / `Transfer-Encoding: chunked`

The simplest and most portable. You POST text; the vendor begins returning the audio body in HTTP chunks (`Transfer-Encoding: chunked`) as it is synthesised. You read bytes off the response stream and feed them into the player. The audio container is usually PCM, MP3, or Opus — pick PCM for the voice-agent path because the player doesn't have to decode.

Shape:

```
POST /v1/audio/speech
Accept: audio/pcm;rate=24000
Body: { "text": "...", "voice": "...", "format": "pcm_24000" }

→ HTTP/1.1 200
   Transfer-Encoding: chunked
   Content-Type: audio/pcm;rate=24000
← [first PCM chunk — ~20–80 ms of audio]
← [next PCM chunk]
← ...
```

TTFB here is essentially **the vendor's internal pipeline latency + the network round-trip**. No WebSocket handshake tax; no SSE framing overhead. Supported by OpenAI (`audio/speech` endpoint with `stream=True`), Azure (via the streaming SDK and the REST endpoint), Google Cloud TTS (streaming over gRPC bidi; the HTTP endpoint is batch-only).

### Server-Sent Events (SSE)

A text-framed streaming channel over plain HTTP. The vendor sends `data: { ... }` lines; each line typically carries base64-encoded audio plus a metadata field (e.g. the character range it covers). Used by some vendors where metadata interleaving matters — character timings, alignment, or speech-mark events.

### WebSocket bidirectional

The richest contract. A full-duplex socket; the client streams text in (even token-by-token, as the LLM emits it), the vendor streams audio out. Supports:

- **Text streaming into the TTS** — you don't need to wait for the LLM to finish before synthesis begins. Push tokens as they arrive; the vendor buffers until it has a synthesis-worthy phrase and begins emitting audio. This is the pattern that gets you the lowest end-to-end latency, because the TTS is pre-warming while the LLM is still generating.
- **Mid-stream cancel.** Send a `flush` or `clear` message to abort the current synthesis. Essential for barge-in (Module 106). Without it, you have to wait for the in-flight synthesis to drain before the agent can speak again.
- **Character-level alignment.** The vendor returns timing events — "characters 10–14 are at audio-time 420 ms" — useful for highlighting words in a UI or for aligning facial animation on a multimodal avatar.
- **Multiple utterances per socket.** Reuse the connection across turns to amortise the handshake tax; the first utterance pays the TLS + WS upgrade cost, subsequent utterances see the lower steady-state TTFB.

Supported natively by ElevenLabs (`/v1/text-to-speech/{voice_id}/stream-input`), Cartesia, PlayHT, and LMNT. WebSocket is the voice-agent contract.

### SDK / provider-only

Some vendors expose a client library that wraps one of the above transports and hides the details. Fine for prototyping; treacherous for production. Two questions you must answer before shipping on an SDK:

- Which transport is underneath? (If it's a WebSocket, what are the reconnect semantics? If it's chunked HTTP, does the SDK buffer in memory, defeating streaming?)
- Can you bypass the SDK when needed? (For load tests, latency measurement, or debugging. If the SDK is the only path, you don't have a production-grade client; you have a wrapper.)

## The TTFB budget

A plausible budget for a production voice agent, target TTFB 250 ms:

| Component | Target | Where it comes from |
|---|---|---|
| LLM-to-TTS RPC setup | <10 ms | In-cluster call or warm HTTP/2 to the TTS endpoint |
| TTS text ingestion | <20 ms | Vendor parses, splits into synthesis units, routes to GPU |
| First inference step | 50–150 ms | Model generates the first ~100 ms of audio |
| First-chunk emission + network | 30–80 ms | Vendor serialises PCM, writes to socket, bytes cross network |
| Client receive + hand to player | <10 ms | Framework overhead |
| Player prebuffer | 0–50 ms | **Set deliberately.** More prebuffer = more TTFB but less underrun risk |
| **Total TTFB** | **~150–300 ms** | |

Numbers vary by vendor, region, model, and text length. This is the shape, not the specification. For the exact numbers that apply to your deployment, run the exercise in [`exercise-02`](exercises/exercise-02-wire-streaming-tts-with-chunked-playout.md).

Three decisions bite you if you get them wrong:

- **The prebuffer size.** Zero prebuffer → lowest TTFB → maximum underrun risk. Too much prebuffer → smooth playback but a noticeable delay. 20–50 ms is usually the sweet spot for voice agents; 100+ ms is appropriate for music or narration. Measure under realistic network jitter, not on a wired desk.
- **Streaming text into the TTS while the LLM is still generating.** If you wait for the LLM's `[DONE]` before even opening the TTS socket, you have added the full LLM tail (hundreds of ms on long responses) onto TTFB. Start streaming text into the TTS as soon as the first complete phrase ("Sure, one moment —") is available.
- **Phrase chunking.** Some TTS vendors want you to flush at sentence boundaries; others want punctuation-triggered flushes; some let you stream raw. Over-chunking causes prosodic seams between utterances; under-chunking buffers the first chunk and increases TTFB. The vendor-specific knob is worth reading.

## The player pipeline

The client half of the contract. Shape-wise:

```
                                               ┌──────────────────────┐
LLM tokens  ──►  phrase buffer  ──►  TTS WS  ──►   audio frame queue  ──►  audio sink (speaker / RTP)
                                               └──────────────────────┘
                                                        ▲
                                                        │
                                                        └── barge-in / cancel
```

Design rules:

1. **The audio sink runs on a dedicated cadence.** A ring buffer or SharedArrayBuffer (web), a PortAudio callback (native), or an RTP packetiser (telephony / WebRTC). It drains at exactly the output sample rate and does not care who fills it.
2. **The frame queue is bounded and small.** A deep queue defeats streaming; it adds queueing delay on top of TTFB. Target 20–80 ms of buffered audio.
3. **The decoder is lazy.** If you pick PCM (recommended for voice agents), there's nothing to decode; bytes go straight into the frame queue. If you pick MP3 / Opus for bandwidth, you pay per-chunk decode latency — measure it.
4. **Barge-in flushes everything.** On a cancel signal (user started talking mid-agent-turn), you must: (a) tell the TTS vendor to stop synthesising, (b) drop everything in the frame queue, and (c) stop the audio sink at the next frame boundary. Letting the frame queue drain is "the agent keeps talking over the user for 800 ms"; your users will complain.
5. **The sample rate is fixed end-to-end.** A sample-rate mismatch between the vendor (24 kHz) and the sink (8 kHz for PSTN, 48 kHz for WebRTC) is handled by a resampler, which costs CPU and introduces latency. Match rates where you can; resample at ingestion where you can't.

### A worked player skeleton (shape only)

```python
# Pseudocode — adapt to the specific SDK / audio framework.
async def speak(text_stream, voice, on_boundary):
    async with tts.stream(
        voice=voice,
        format="pcm_24000",
        sample_rate=24_000,
        enable_alignment=True,     # character-level timing events
        auto_mode=False,           # flush on our schedule, not the vendor's
    ) as session:
        async def sender():
            buf = ""
            async for token in text_stream:
                buf += token
                if is_flushable(buf):             # phrase boundary heuristic
                    await session.send_text(buf)
                    buf = ""
            if buf:
                await session.send_text(buf)
            await session.flush()

        async def receiver():
            async for event in session.events():
                match event.type:
                    case "audio":
                        audio_queue.put(event.pcm)         # <=80 ms of audio
                    case "alignment":
                        on_boundary(event.character_index, event.audio_time_ms)
                    case "error":
                        raise TTSError(event)
                    case "end_of_stream":
                        audio_queue.put_sentinel()
                        return

        async def canceller():
            await barge_in_event.wait()
            await session.cancel()
            audio_queue.flush()
            audio_sink.stop_next_frame()

        await asyncio.gather(sender(), receiver(), canceller())
```

Three things a reviewer should ask when reading your actual implementation:

- **Who owns the sample-rate clock?** The audio sink should. Everything else is pushing into it on its own cadence.
- **What happens on an in-flight TTS error?** You are mid-utterance, synthesis dies. Can you fail over to a backup voice? Fall back to a cached "sorry, one moment" phrase? Let the silence land and move on? Decide before it happens.
- **What is the reconnect policy?** Each turn opens a new WS session, or you reuse? If you reuse, how do you detect a socket that has silently died (the vendor dropped it but didn't send a FIN)?

## Measuring TTFB honestly

Three numbers to track separately:

- **TTFB-wall** — `wall_clock(first PCM byte received) - wall_clock(first text byte sent)`. The one that hits the user.
- **TTFB-synth** — the vendor's internal processing time, excluding network. Some vendors emit this as a response header; others require a probe where you co-locate the client with the vendor PoP.
- **Realtime factor (RTF)** — total audio-seconds generated / total wall-clock-seconds. Below 1.0 means the engine is slower than realtime and your player **will** underrun. RTF is a throughput number, not a latency number, but it bounds the shape.

Four discipline rules for measurement:

- **Measure from your region, in business hours**, not from your laptop at 11 PM. The vendor PoP routing and the region-pair RTT change the numbers.
- **Use realistic text.** Short "hello" probes measure the wrong thing — the vendor has silently different code paths for one-sentence prompts vs. multi-sentence paragraphs.
- **Stratify by output length.** TTFB on a 10-character "Yes." is not the same as TTFB on a 400-character apology. Report p50 and p95 per length bucket.
- **Repeat enough times for a defensible CI.** 100+ runs per bucket, cold and warm (socket-reused) separately. Vendor warm-up effects can be 2–3× the steady-state TTFB.

## The barge-in contract

Barge-in is Module 106's home, but the TTS contract has to carry it. The fields to verify per provider:

- **Mid-stream cancel.** Can the client tell the vendor "stop synthesising this utterance, now"? Over what signal? (A `cancel` WS message, closing the stream, an HTTP `DELETE`.) What's the vendor's p95 latency from cancel to actual-stop-of-billing?
- **Word / sentence boundary stop.** Can you ask the vendor to stop at the next word or sentence boundary instead of mid-phoneme? Some providers expose this; most do not. Mid-phoneme stops sound like a glitch.
- **Partial-utterance credit.** When you cancel mid-utterance, do you pay for the characters you requested, the characters synthesised, or the audio seconds generated? Affects unit economics on high-barge-in deployments.
- **Alignment events.** If you need to know "which character did the agent reach before the user interrupted," the vendor must emit alignment events. Without them, resuming context ("sorry, let me finish — as I was saying…") is guesswork.

Design the barge-in cancel path **before** you ship. Teams who ship TTS without barge-in and bolt it on later rediscover the full streaming contract the hard way.

## Common anti-patterns

- **Synthesise the whole utterance, then play.** The classic. Six seconds of text produce six seconds of silence before the agent speaks. If you see `wav = tts.synthesise(text); player.play(wav)` in a voice-agent codebase, that is the bug — regardless of how fast the synthesis is.
- **A large prebuffer to mask underrun.** You've hidden a slow engine by adding 200 ms of latency. You've optimised the wrong variable. Fix the engine (or the chunk cadence) and shrink the prebuffer.
- **Flushing on every token.** The TTS vendor synthesises three characters, then three more, then three more. Prosody dies; seams appear every 60 ms. Flush on phrase / sentence boundaries.
- **No cancel on barge-in.** The user interrupts; the agent keeps talking for another 800 ms because the frame queue is draining. User hangs up.
- **One socket per utterance, no reuse.** TLS + WS handshake on every turn. Measurable TTFB regression at scale. Reuse the session across turns on the same conversation.
- **MP3 or Opus on the voice-agent path.** You're paying a per-chunk decode cost to save a bit of bandwidth that doesn't matter at voice-agent volumes. Use PCM.
- **Not testing with jitter.** Your office wifi is not your users' LTE. Synthetic packet loss and latency jitter uncover underrun bugs the desk setup hides.

## Summary

- The latency variable is TTFB. Everything after the first audio byte is overlapped with playback.
- Pick a streaming transport that matches the use case: HTTP chunked for simple one-shot streaming, WebSocket bidirectional for voice agents (text-in-stream, mid-stream cancel, alignment events).
- Design the player pipeline around a bounded frame queue with a dedicated audio sink; keep decoding lazy; match sample rates end-to-end.
- Stream text into the TTS as the LLM generates it, don't wait for `[DONE]`. Flush on phrase boundaries, not tokens.
- Barge-in is a first-class requirement on the TTS contract; ship with cancel from day one.
- Measure TTFB with realistic text, from your region, in business hours, stratified by length.

Next chapter: SSML prosody and phonetic lexicons — how to make the voice say things the way you want, including the words it has never heard.
