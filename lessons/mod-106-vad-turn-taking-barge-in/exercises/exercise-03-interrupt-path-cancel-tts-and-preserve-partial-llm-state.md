# exercise-03: Interrupt Path — Cancel TTS and Preserve Partial LLM State

**Estimated effort:** 3 hours · **Output:** a working barge-in path in a minimal voice agent, a measured p95 interrupt reaction time, and a short write-up on the state-preservation design.

## Objective

Implement the Chapter 3 barge-in path end-to-end in a minimal voice agent you control. Hit the five stages: detect, classify (stub for now), cancel (LLM + TTS), flush (player), truncate (conversation state). Measure p95 interrupt reaction time against a corpus of barge-in clips; iterate until you meet the <200 ms budget on at least one deployment pattern.

The point is to confront the stages that look simple on the lecture page but have sharp edges in code: stopping the TTS stream cleanly, flushing the player without clicks, computing `audio_played_ms` correctly, and truncating conversation state so the next turn doesn't reference phantom content. You will fail the first test run, probably on the playback flush. That is the point.

## Prerequisites

- [Chapter 3 — The Interrupt Path](../03-barge-in-interrupt-path.md).
- Exercise 1: you have `StreamingVAD`.
- Access to **one** pipeline stack. Either:
  - **Pipeline path.** A streaming LLM (any; `gpt-4o-mini`, `claude-haiku`, `gemini-flash`) + a streaming TTS (ElevenLabs, OpenAI TTS, Deepgram TTS, Rime). You own the socket and the player.
  - **Speech-native path.** OpenAI Realtime (`turn_detection: null`, manual commit mode). Vendor owns generation; you own the player and the cancel semantics.
- Python 3.10+ with `asyncio`. A terminal and a way to play audio (e.g., `sounddevice`) OR a WAV sink file.

## Requirements

### Part A — stand up a minimal voice agent

Build a single-turn agent that:

1. Listens for user audio via `StreamingVAD`.
2. On `speech_ended`, commits a turn: finalises transcript, calls LLM, streams TTS, plays back.
3. Supports a single `on_interrupt()` signal that triggers the cancel path.

This is scaffolding, not product. One test utterance is enough: "tell me a 30-second story about a lighthouse" gets a long bot response you can interrupt mid-stream. Keep the rest of the loop stubbed.

### Part B — the interruptible player

Implement a player with a cancellable queue:

```python
class InterruptiblePlayer:
    def __init__(self, sample_rate: int = 24000):
        self.sample_rate = sample_rate
        self.queue: asyncio.Queue[bytes] = asyncio.Queue()
        self.played_bytes: int = 0
        self._stopping = asyncio.Event()

    async def feed(self, pcm_chunk: bytes):
        if self._stopping.is_set(): return
        await self.queue.put(pcm_chunk)

    async def run(self, sink):
        while not self._stopping.is_set():
            chunk = await self.queue.get()
            # Apply a short fade-out if stopping during this chunk.
            await sink.write(chunk)
            self.played_bytes += len(chunk)

    def stop(self) -> int:
        self._stopping.set()
        self._drain_queue()
        # Return ms of audio played before stop.
        return (self.played_bytes // 2) * 1000 // self.sample_rate  # PCM16 mono
```

- Clearing `self.queue` on stop is mandatory; without it, the player drains buffered audio and the "cancel" is invisible.
- A 20–50 ms fade-out on the trailing frame avoids a click at the cut; a hard cut sounds like a bug.
- `played_bytes` must track bytes actually written to the sink, not bytes enqueued.

### Part C — the cancel path

Wire `on_interrupt()`:

**Pipeline variant:**

```python
async def on_interrupt():
    t_detect = perf_counter_ms()
    await llm_stream.aclose()          # stop generating new tokens
    await tts_stream.aclose()           # stop synthesising new audio
    played_ms = player.stop()           # flush player; record played_ms
    truncate_last_turn_history(played_ms)  # Stage 5
    t_stopped = perf_counter_ms()
    log_interrupt(detect=t_detect, stopped=t_stopped, played_ms=played_ms)
```

**Realtime variant:**

```python
async def on_interrupt():
    t_detect = perf_counter_ms()
    await ws.send(json.dumps({"type": "response.cancel"}))
    played_ms = player.stop()
    await ws.send(json.dumps({
        "type": "conversation.item.truncate",
        "item_id": last_response_item_id,
        "content_index": 0,
        "audio_end_ms": played_ms,
    }))
    t_stopped = perf_counter_ms()
    log_interrupt(detect=t_detect, stopped=t_stopped, played_ms=played_ms)
```

**The cancel must be parallel.** `asyncio.gather` the LLM close, the TTS close, and the player stop — not serial. Serial costs you the three operations in sum; parallel costs you the max.

### Part D — truncate conversation state

**Pipeline:** rewrite the running `history[-1]` to the truncated prefix. Use TTS word-timings if your vendor exposes them (ElevenLabs returns character-level timings on its websocket; OpenAI TTS does not stream timings per character but you can approximate from the audio position). If no timings, assume ~15 characters per second as a constant speaking-rate prior and truncate the response text at `played_ms * 15 / 1000` characters.

**Realtime:** the `conversation.item.truncate` event is the full truncation; no client-side history to rewrite because the server owns it.

Append a `[interrupted]` marker to the truncated prefix so the LLM on the next turn knows it was interrupted. Prompt the model (in its system instructions) to acknowledge the interruption gracefully on the next turn.

### Part E — build a barge-in corpus

Record (or synthesise) **at least 10 clips** of the following shape: a prompt the bot starts answering, and a user utterance that begins at a known `t_barge_in_ms` into the bot's response. Each clip:

- Input: the prompt text, so the pipeline generates a predictable response.
- Audio: the user's barge-in utterance with the precise onset time labelled.
- Variants:
  - Very early barge-in (bot has just started speaking).
  - Mid-response barge-in (user cuts in 3 s into a 15 s response).
  - Late barge-in (bot almost done; user adds a quick correction).

You can synthesise barge-in audio directly or record it; realistic is better but either works for this exercise.

### Part F — measure

For each clip, run the pipeline end-to-end:

1. Play the prompt (text, not audio).
2. Pipeline generates response; starts synthesising; player starts playing.
3. At the labelled barge-in time, feed the user's audio frames into `StreamingVAD`.
4. The VAD fires `speech_started`; `on_interrupt()` triggers.
5. Record timing.

Compute:

- **Reaction time** = `t_stopped - t_user_onset`. This is the Chapter 5 Metric 5.
- **Stages' contribution.** `detect_ms` (VAD fire minus user onset), `cancel_ms` (operations complete minus VAD fire), `flush_ms` (player stopped minus cancel sent).

Report p50 and p95 reaction time across the corpus. Target: p95 < 200 ms. Also report the stage contributions; name the longest stage.

### Part G — write-up

Answer:

- What is your p50 and p95 reaction time? Break down by stage.
- Where did the longest stage end up — VAD latency, cancel protocol, or player flush?
- Run one clip without `conversation.item.truncate` (or the pipeline equivalent). Describe the misbehaviour on the next turn.
- Run one clip without the player fade-out. Describe what the audio cut sounds like.
- On your pipeline, is it feasible to resume the bot's utterance from the interrupt point (Chapter 3 §Preserving partial state) rather than starting over? Sketch the design even if you don't build it.

## Starter guidance

- **AEC.** If your test rig is a headset, AEC is implicit (no feedback path). If it is a speakerphone or two-device loopback, run AEC upstream or the bot will interrupt itself every time. For this exercise, headset is fine.
- **Parallelism.** `asyncio.gather(llm.aclose(), tts.aclose())` is a one-liner; use it. Serial `await` chains add real latency.
- **Logging.** Log every stage with `perf_counter_ms()` timestamps. Wall-clock on `time.time()` is too coarse.
- **Measuring at the device vs. server.** Where feasible, timestamp the moment the player actually stops writing to the sink, not the moment the cancel is scheduled. Chapter 5 is explicit about this.
- **Don't trust a single run.** Barge-in timing has variance; run each clip at least three times and aggregate.
- **Keep the prompt deterministic.** `temperature=0` on the LLM so the TTS stream has roughly the same length each run; variance in run-to-run response length confuses the measurement.

## Acceptance criteria

- [ ] Working single-turn voice agent on either pipeline or Realtime stack.
- [ ] `InterruptiblePlayer` that stops, flushes, fades, and reports `played_ms`.
- [ ] `on_interrupt()` that cancels LLM (if pipeline), cancels TTS, flushes player, and truncates conversation state. Cancel ops run in parallel.
- [ ] Barge-in corpus of ≥ 10 clips with labelled onset times.
- [ ] Measurement harness that runs each clip, logs stage timings, and emits p50 + p95 reaction time.
- [ ] One-page write-up answering the five questions above.
- [ ] Sanity demo: a clip where the user interrupts "tell me a story" and asks "wait — what's the weather?"; show that (a) the bot stops within 300 ms and (b) the LLM on the next turn does not reference content that was cut off.

## Stretch goals

- **Compare pipeline and Realtime stacks.** Run the same corpus through both and compare reaction time + resume quality. Which stage shifts between them?
- **Resume-from-interrupt.** Implement "resume the last turn from where it was cut, incorporating the user's correction," using the Chapter 3 §Preserving partial state approach. Measure: does it feel better than starting over?
- **Realtime without `conversation.item.truncate`.** Reproduce the failure mode (bot references content the user never heard). Log the transcript of the next turn.
- **Stress test: rapid barge-in.** Interrupt within 100 ms of the bot starting to speak. Does the pipeline handle a barge-in that arrives before the first `audio.delta`?
- **Measurement at the device.** If your sink is `sounddevice` or an audio library with callback timing, record the actual sink-stopped timestamp and compare with the cancel-scheduled timestamp. Quantify the player-flush latency.

## Non-goals

- The backchannel classifier. This exercise assumes any user speech during bot-speaks is a barge-in. Exercise 4 adds the classifier.
- Full observability (dashboards, alerting). A log file and a script are enough.
- Deploying to production. This is a bench harness.

## Keep it

The `InterruptiblePlayer`, the cancel-path code, and the barge-in corpus are reusable across the rest of the module and the module's capstone project. Keep them in a shared `utils/` directory; Exercises 4 and 5 will import from it.
