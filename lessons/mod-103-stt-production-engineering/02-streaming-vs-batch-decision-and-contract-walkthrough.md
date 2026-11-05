# 02. Streaming vs. Batch: The Decision and the Contract

## Why this chapter

Streaming and batch ASR look like two configuration options of the same product. They are not. They are two different products with two different cost structures, two different quality envelopes, two different error modes, and two different integration contracts. Picking the wrong one is the second-most common STT mistake after picking the wrong vendor — usually surfacing as "the voice bot feels sluggish" (batch used where streaming was needed) or "the transcript is subtly wrong and nobody noticed" (streaming partials used where a batch final was needed).

This chapter gives you the decision rule, the latency target you're defending, and the streaming-contract fields you have to actually read before you can design the LLM handoff in [Chapter 3](03-confidence-informed-handoff-into-the-llm.md).

## The decision rule

**Streaming** if any downstream component needs partial transcripts before the utterance ends:

- Interactive voice agents. The LLM, the VAD-based turn-taker, and the barge-in logic all consume partials. The latency target is a **sub-300 ms partial-transcript latency** from the moment a word is spoken to the moment the partial appears. (Module 109 budgets this explicitly; the 300 ms figure is the headroom you need to leave for the LLM, TTS, and transport.)
- Live captioning and live translation UIs.
- Realtime fraud / compliance keyword triggers inside a call.

**Batch** if you need the transcript *after* the audio is complete and accuracy dominates:

- Call-centre post-call analytics, QA scoring, coaching.
- Compliance archival, redaction, discovery.
- Podcast / video transcription, meeting summarisation after the fact.
- Any case where the business can wait seconds-to-minutes for a final.

Batch models are usually a slightly larger / slower / higher-accuracy variant of the same base. The provider can use a two-pass decoder, apply a stronger language model, and commit to a single final transcript with no backtracking. Streaming models give up some of that for responsiveness.

A pattern you will see often in production: **both, on the same call.** The streaming leg drives the agent in realtime; the batch leg runs post-call against a stored audio file and produces the archival transcript. Treat the two outputs as *different artefacts for different consumers*, not as a quality-vs-cost swap.

## The streaming contract

A streaming STT endpoint is a bidirectional socket (WebSocket or gRPC) that you push audio into and that pushes events back. Every provider has subtle differences in field names and shapes; the *concepts* below are universal.

### Interim vs. final results

The core distinction. A streaming endpoint returns two kinds of transcript events:

- **Interim / partial result.** A best-guess transcript of what's been heard so far, published mid-utterance. **It can change.** The next event may extend it, revise it, or replace it entirely as more audio arrives. Interim latency is what the user feels.
- **Final result.** A commitment: this is the provider's final transcript for a segment of audio. **It will not change.** Finals are emitted at endpoints (silence / semantic boundary / max-segment length) or at the end of the stream.

The two together form a stream that looks roughly like:

```
→ audio chunk (160 ms)
← interim { "transcript": "hey can you" }
→ audio chunk (160 ms)
← interim { "transcript": "hey can you help me" }
→ audio chunk (160 ms)
← interim { "transcript": "hey can you help me with" }
→ audio chunk (160 ms)
← interim { "transcript": "hey can you help me with my order" }
→ audio chunk (silence)
← final   { "transcript": "Hey, can you help me with my order?" }
```

Notice what happened at the final: capitalisation, punctuation, and formatting ("hey can" → "Hey, can") typically appear only at the final. Interims are usually raw.

The design error you will see repeatedly: **sending interims to the LLM.** The agent then responds to a half-sentence, or to a sentence that the next interim retracted, and the UX falls apart. The general rule is **act on finals, display interims** — though real pipelines nuance this (Chapter 3 covers early-dispatch patterns and their guardrails).

Three fields on the event object you must read in the provider docs:

- `is_final` / `type: "Transcript"` with a final flag / `message_type: "FinalTranscript"` — **what the provider calls it varies, the semantics are the same.** Deepgram uses `is_final` on the top-level result and `speech_final` for an end-of-utterance signal. AssemblyAI emits distinct `PartialTranscript` and `FinalTranscript` messages. Google STT v2's streaming response has an `is_final` boolean on each `SpeechRecognitionAlternative` result. ([Deepgram streaming docs](https://developers.deepgram.com/docs/live-streaming-audio), [AssemblyAI streaming docs](https://www.assemblyai.com/docs/speech-to-text/universal-streaming), [Google STT v2 streaming docs](https://cloud.google.com/speech-to-text/v2/docs/streaming-recognize).)
- **Stability.** Some providers (Google, Azure) emit a `stability` score on interim results — a prediction of "how likely is this interim to survive unchanged." Low-stability interims are the dangerous ones to act on.
- **End-of-utterance / endpointing signal.** Different from "end of stream." Signals that the provider believes the user has stopped speaking (based on silence duration + model-internal endpointing). Named `speech_final` (Deepgram), `UtteranceEnd` (various), or inferred from a final with a trailing silence. This is the signal the turn-taker consumes (Module 106).

### Word timings

Most streaming providers attach per-word `start` / `end` timestamps (in seconds, usually relative to stream start). You will use them for:

- **Mapping confidence onto tokens** for the LLM handoff in [Chapter 3](03-confidence-informed-handoff-into-the-llm.md).
- **Alignment with VAD events** for turn-taking (Module 106).
- **Highlighting words in a UI** for live captions.
- **Diarisation** — the per-word speaker label, when the provider emits it inline.

Beware: interim word timings are not stable. Words at the tail of an interim often have provisional timings; they shift (sometimes by hundreds of milliseconds) at the final. Use interim timings for UI only; use final timings for persistence.

### Confidence

Per-word or per-utterance confidence is a score in `[0, 1]`. What it *means* varies by provider:

- Often it is a **posterior probability** derived from the model's output distribution. Sometimes it is a **heuristic** combining acoustic and language-model scores.
- It is **not guaranteed to be calibrated** — a provider's "0.80 confidence" does not necessarily correspond to an 80 % correctness rate. You calibrate it against your own data (Chapter 3).
- Some providers emit it only on finals, not interims. Read the docs.

### Formatting, redaction, and the "smart format" layer

Most providers run a post-processing layer that converts raw recognition ("one two three main street") into formatted text ("123 Main Street"). Features in this layer:

- **Punctuation and capitalisation.**
- **Number / date / currency / phone-number formatting.**
- **Profanity filtering.**
- **Redaction** — replacing credit-card numbers, SSNs, names with `[REDACTED]`. Essential for PCI / HIPAA pipelines; see Module 108.
- **Entity detection** — tagging `AGENT` / `CALLER`, named entities, etc.

Smart formatting is almost always only applied to **finals**. If you forget this and parse interim transcripts as if they were formatted, you will index the wrong tokens.

### Error, close, and reconnect semantics

The last piece of the contract, and the piece most tutorials skip:

- **Idle / auth / quota errors** mid-stream. What status code does the provider send? Does it close the socket, or send an error frame and keep the socket open?
- **Reconnection.** If the socket drops after 30 s of a 2-minute call, can you resume? Most providers say *no* — you start a fresh stream, with a fresh ID, with a fresh "warm-up" period. You lose any model state that had built up.
- **Backpressure.** If you send audio faster than the provider processes it (because you're replaying a file, or because you buffered during a hiccup), what happens? Some providers silently drop; some close the socket; some apply gentle back-pressure on the send queue. Test this once per provider.
- **Keep-alive / heartbeat requirements.** Many providers require periodic keep-alive frames during silence to keep the socket open. Twilio Media Streams, LiveKit, and provider SDKs handle this for you; if you're wiring the WebSocket by hand, you own it.

**Draw the contract on paper for each provider you shortlist.** You will be grateful at 2 AM.

## A worked contract walkthrough (shape only)

Here is a sketch of what a streaming client looks like for an arbitrary provider. The specifics vary; the *shape* is universal. Treat this as a mental checklist for every provider SDK you touch.

```python
# Pseudocode — adapt to the specific provider SDK.
async def run_streaming_asr(audio_chunks, on_partial, on_final):
    async with provider.streaming(
        sample_rate=16_000,
        encoding="pcm16",
        language="en-US",
        interim_results=True,
        word_timestamps=True,
        confidence=True,
        smart_format=True,
        punctuate=True,
        diarize=True,
        keyterms=["Rook Insurance", "deductible", "copay"],  # see Chapter 4
        endpointing_ms=400,
    ) as stream:
        async def sender():
            async for chunk in audio_chunks:
                await stream.send_audio(chunk)
            await stream.finish()

        async def receiver():
            async for event in stream.events():
                match event.type:
                    case "interim":
                        on_partial(event.transcript, event.stability, event.words)
                    case "final":
                        on_final(event.transcript, event.words, event.confidence)
                    case "utterance_end":
                        on_final.flush()
                    case "error":
                        raise ProviderError(event)

        await asyncio.gather(sender(), receiver())
```

Three things a reviewer should ask when they read your actual implementation:

- **Who decides when the stream ends?** The user stopped talking (VAD), the provider's endpointer, a hard timeout, or the business ("user hung up")? Which signal do you act on, and in what order of precedence?
- **What happens on an in-flight final that disagrees with the last interim?** (It will happen. The provider's commit is authoritative; whatever you built from interims must be revisable.)
- **What is your reconnect policy?** Drop the call, retry silently, retry with a user-visible "one moment"? Each has consequences.

## Batch mode

Batch is the easier contract: push audio (file upload or URL), poll a job, receive a final transcript with word timings, confidences, and diarisation. The nuances worth knowing:

- **Latency is a function of audio length** — usually 0.1–0.3× real-time on fast providers, higher on more accurate models. A 60-minute call may take 6–20 minutes to transcribe. Design the UX for this.
- **Accuracy is usually the strictly-dominant batch quality.** Providers can afford a two-pass decoder and a stronger language model. If you have ground truth, benchmark streaming and batch separately; the WER gap is usually 1–3 points.
- **Call-analytics product tiers** (AWS Transcribe Call Analytics, Azure's conversation analysis, AssemblyAI's LeMUR-adjacent batch flow) do post-call sentiment, topic, categorisation, and summarisation in the same job. Use these when you'd otherwise build a whole analytics pipeline yourself.
- **Diarisation quality is often higher in batch**, because the provider can use the full audio to cluster speaker embeddings.

### When to run both on one call

A common split:

- **Streaming leg → the agent's realtime turn.** Low-latency partials drive VAD, turn-taking, LLM.
- **Batch leg → the call's final transcript of record.** Run after the call ends; this is what you store, what analytics uses, what compliance reviews. Streaming transcripts are not usually of archival quality.

Design the two as different artefacts with different consumers. Do not try to retrofit the streaming trace into the batch slot; you'll inherit the streaming model's quality ceiling.

## Interpreting "partial-transcript latency"

The number you'll defend to product managers is **time-to-first-partial**: from the first audio sample containing a word, how long until you see that word in a partial transcript. Measure it with a timestamped audio file and a logging sink; don't trust the number on a landing page.

Three latencies to track separately:

- **TTFP (time-to-first-partial).** How quickly does *any* text appear after the user starts speaking? This is the "feels responsive" metric.
- **TTFF (time-to-first-final).** How long after the user stops speaking until the final is committed? This is the "turn handoff" metric — the main input into Module 106 turn-taking.
- **Partial churn.** How often do interim transcripts revise themselves? High churn means the partials are not actionable; low churn means you can show them to the user directly.

A provider that advertises "under 300 ms" without naming which of these numbers it is measuring is marketing. Ask them which latency, measured how, on what hardware, with what endpointing settings, and reproduce on your audio before you commit to anything.

## Common anti-patterns

- **Acting on interim transcripts as if they were finals.** The agent responds mid-sentence or retracts its own context when the next interim replaces words. Fix: default to finals; design explicit early-dispatch patterns (Chapter 3) when the speed-up is worth the risk.
- **Treating batch as a drop-in substitute for streaming.** The batch endpoint has no interim concept; your agent sits silent for 10 s while the full-length job runs, and the user hangs up. The decision rule above exists so you pick correctly up front.
- **Ignoring endpointing defaults.** Default endpointing silence thresholds (often 500–800 ms) are tuned for dictation, not conversation. For agents, you often want more aggressive endpointing (200–400 ms) combined with external VAD. Check the knobs.
- **Streaming with keep-alive disabled.** The socket dies after the first few seconds of silence; your agent logs nothing and you can't reproduce it until you add keepalive. (Most provider SDKs do this; rolled-your-own clients often don't.)
- **No reconnect design.** The one-in-a-thousand mid-call socket drop takes the user's call down. At a million calls a month that's a thousand incidents.

## Summary

- Streaming for interactive voice; batch for post-call analytics; often both on one call for different consumers.
- The streaming contract is interim vs. final, word timings, confidence, formatting, and error/reconnect semantics. Read it for every provider you ship.
- Latency target for streaming: sub-300 ms time-to-first-partial, with TTFF and partial-churn tracked separately.
- Default rule: act on finals, display interims. Early-dispatch on interims is possible but requires the Chapter 3 confidence machinery.

Next chapter: the confidence-informed handoff from STT into the LLM — where the contract above meets your prompt design.
