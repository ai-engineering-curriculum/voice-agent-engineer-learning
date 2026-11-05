# exercise-02: Streaming vs. Batch Decision + Streaming-Contract Walkthrough

**Estimated effort:** 2.5 hours · **Output:** a small working streaming client + an annotated event log + a short contract comparison

## Objective

Internalise the Chapter 2 streaming contract by **wiring a real streaming STT client end-to-end** against at least one provider, logging every event the provider emits, and annotating it with the Chapter 2 concepts: interim vs. final, stability, word timings, confidence, endpointing, keep-alive, reconnect.

You will finish able to reach for any provider's streaming SDK and know in under an hour which contract fields it supports, which have provider-specific quirks, and where the sharp edges are.

## Prerequisites

- [Chapter 2 — Streaming vs. Batch: The Decision and the Contract](../02-streaming-vs-batch-decision-and-contract-walkthrough.md).
- A developer account with at least one streaming provider that has a generous free tier (Deepgram, AssemblyAI, and Google Cloud all qualify at the time of writing). A second provider is strongly recommended for Part D.
- Python 3.10+ with the provider's SDK + `numpy` + `soundfile`.
- A ~30–60 second audio sample containing a mix of speech types: short utterances, a longer run-on sentence, a sentence with a few hesitations, a sentence with numbers or an address (something the provider might format).

## Steps

### Part A — Decide the mode (15 minutes)

1. Write a two-paragraph decision note for your chosen audio: **streaming or batch?** Why? Which of Chapter 2's decision-rule rows applies? If you chose "both," name what each leg is for.
2. If your audio is prepared / recorded (you're testing with a WAV file), simulate streaming by sending it in realtime-cadence chunks (e.g. 100 ms per chunk, with `asyncio.sleep(0.1)` between sends). Resist the temptation to flood the socket at once — you are testing a streaming contract, not throughput.

### Part B — Wire the streaming client (60 minutes)

3. Build a minimal client that:
   - Opens the streaming session with appropriate parameters (sample rate, encoding, language, interim results on, word timestamps on, confidence on, smart format / punctuate on, diarisation on if the provider supports it).
   - Sends audio in 100–200 ms chunks at real-time cadence.
   - **Logs every event the provider emits** to a JSONL file. Include the provider's raw event object plus a timestamp in ms since stream start.
   - Prints interims and finals to stdout as they arrive, with a distinct glyph for each.

4. Verify the client works end-to-end by running it against your audio and confirming:
   - You see interim events appearing throughout the utterance.
   - You see finals appearing at pauses or utterance ends.
   - Finals carry punctuation / capitalisation that interims do not.
   - Word timings on finals are stable (re-run; same word → same `start`/`end` within a few ms).

### Part C — Annotate the event log (45 minutes)

5. Open your JSONL log and, for each unique event type, write an annotation:
   - **Field map.** Which field in this provider's event corresponds to Chapter 2's: `is_final`, `stability`, `word_timings`, `word_confidence`, `utterance_end`, `error`?
   - **Deviations.** Anything the provider does that Chapter 2 did not predict — e.g., a `speech_final` that fires before the full final arrives; a separate "model-switched" event; batched word-time arrays per segment.
   - **Latency numbers.** Compute TTFP, TTFF, and partial churn for this run. Partial churn = `(count of interim events whose text differs from the previous interim) / (total interim count)`.

6. Produce three measurements:
   - **p50 and p95 TTFP** across utterances in your audio.
   - **p50 and p95 TTFF** across utterances.
   - **A plot** (ASCII or image) showing: y-axis word index; x-axis time; one line per interim, one for the final. The "convergence" pattern should be visible.

### Part D — Compare a second provider (30 minutes, strongly recommended)

7. Point the same client (or a minimal fork) at a second streaming provider with the same audio. Capture the same event log, annotated.
8. Produce a side-by-side comparison on the Chapter 2 axes:
   - Interim / final event shapes.
   - Field names for word timings and confidence.
   - Stability support (yes / no / partial).
   - Endpointing behaviour on your audio (which trailing silence duration triggers a final? Is there a tunable?).
   - Reconnect behaviour: kill the socket mid-stream manually; does the SDK auto-reconnect? Does your transcript resume or restart?
   - Keep-alive behaviour: pause audio for 20s; does the socket close?
9. Pick one quirk you did not predict from the documentation and note it. (There will be at least one.)

## Starter guidance

- Keep the client under ~150 lines of Python. The point is to exercise the contract, not to build an abstraction.
- Use the provider SDK, not a raw WebSocket, for the first build. Switch to raw WS only if the SDK hides the contract details you need.
- Log *raw* events, not pretty-printed ones. You want to see the actual field names and shapes the provider uses — that's what makes the annotation exercise meaningful.
- If your provider has multiple models (base, enhanced, Nova-3 etc.), pick the production-recommended one. Don't tune it; the goal is to see the default contract.
- For the reconnect test, use `kill` on the network layer — close the TCP socket, don't just stop the sender. SDK auto-reconnect behaviour varies by provider and version.

## Acceptance criteria

- [ ] Streaming vs. batch decision note, with the applicable Chapter 2 rule cited.
- [ ] Working streaming client that logs all provider events to JSONL.
- [ ] Annotated field map: provider event fields → Chapter 2 concepts, with any deviations flagged.
- [ ] Measured TTFP (p50, p95), TTFF (p50, p95), and partial churn across the audio.
- [ ] Interim-convergence plot (shows interims refining into a final over time).
- [ ] At least one documented provider quirk that was not predictable from the docs.
- [ ] (Strongly recommended) Side-by-side second-provider comparison on the same axes.

## Stretch goals

- **Add a VAD-based turn-taker to your client.** Use `webrtcvad` or Silero VAD to flag "end of turn" independently of the provider's endpointing. Log both signals side by side. Which comes first? By how much? (This is the Module 106 preview.)
- **Build a transcript-diff visualiser.** For each pair of consecutive interims, show the diff (words added / removed / changed) with colour. Use it to visualise partial churn — the pattern is the primary signal for whether interims are actionable.
- **Test the provider's N-best or alternatives surface.** For each final, if the provider emits N-best, log the top 3 and their scores. Now hold up your hand: how often does the top-1 differ from the final committed transcript? (Should be rare, but non-zero.)
- **Drive the client from a Twilio Media Streams or LiveKit audio source** instead of a file. You now have the beginnings of a production voice-agent streaming pipeline — one that modules 106–109 will extend.

## Keep it

The JSONL event logs and the annotated field maps are the start of your personal provider-contract reference. Every future provider evaluation is one of these runs away from a go/no-go decision. Keep them under version control; they age well.
