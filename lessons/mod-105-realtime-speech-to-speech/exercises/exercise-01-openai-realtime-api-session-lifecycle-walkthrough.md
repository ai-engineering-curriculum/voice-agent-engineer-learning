# exercise-01: OpenAI Realtime API — Session Lifecycle Walkthrough

**Estimated effort:** 3 hours · **Output:** a working, instrumented Realtime client (WebSocket or WebRTC) plus a lifecycle log showing every event and a short write-up of what you observed.

## Objective

Walk the OpenAI Realtime API's session lifecycle end-to-end **by hand**, so that every event in the grammar from Chapter 1 — `session.update`, `input_audio_buffer.append`, `input_audio_buffer.commit`, `response.create`, `response.audio.delta`, `response.function_call_arguments.*`, `conversation.item.truncate`, `response.cancel`, and friends — is something you have sent or received at least once and can reason about. No agent framework, no vendor SDK except as a thin wrapper around the socket.

The point is not to ship a product; the point is to understand the contract well enough that when a production bug surfaces ("the user says the agent is hearing itself," "interrupts leave the model's history out of sync," "the agent is silent for 2 s between the tool call and the follow-up"), you can diagnose from first principles.

## Prerequisites

- [Chapter 1 — The OpenAI Realtime API: session lifecycle, transports, and the voice loop](../01-openai-realtime-api-session-lifecycle.md).
- An OpenAI API key with Realtime access (verify your account can see `gpt-4o-realtime-*` in the model list).
- Python 3.10+ or Node 20+; the `websockets` library (or `ws`). Do not use the LiveKit / Pipecat / OpenAI agent-framework abstractions for this exercise — you learn nothing by letting them manage the socket.
- A microphone + speakers, or a prerecorded PCM file you can loop as "mic input" plus a WAV file you append audio frames to as "speaker output."
- A terminal where you can see per-event logs in time order.

## Requirements

Build one program that does the following, on either the WebSocket or the WebRTC transport (pick one; preference WebSocket because it is easier to log).

### Core walk

1. **Connect and configure.** Open the socket. Send a `session.update` that sets:
   - `modalities: ["text", "audio"]`,
   - `voice`: any voice,
   - `input_audio_format: "pcm16"`, `output_audio_format: "pcm16"`,
   - `input_audio_transcription.model: "whisper-1"`,
   - `turn_detection.type: "server_vad"` with `silence_duration_ms: 500`,
   - a short `instructions` string,
   - one function tool (see §Tool).

   Log the `session.updated` echo. Confirm the resolved config matches what you sent (look for silently-ignored fields).

2. **Append audio.** Stream real or prerecorded PCM16 chunks via `input_audio_buffer.append` at ~100 ms of audio per event. Log each append with a wall-clock timestamp and the chunk size.

3. **Let server VAD commit the turn.** Observe the server commit the buffer and begin a response. Log every event received: `input_audio_buffer.speech_started`, `.speech_stopped`, `.committed`, `response.created`, `response.output_item.added`, `response.audio_transcript.delta`, `response.audio.delta` (log sizes, not payloads), `response.output_item.done`, `response.done`.

4. **Play audio.** Write the received audio deltas to a WAV file or pipe to speakers. Verify the sample rate matches your `output_audio_format`.

5. **Also run manual commit.** In a second session (or by reconfiguring mid-session), set `turn_detection: null` and commit a turn manually via `input_audio_buffer.commit` + `response.create`. Confirm the model responds. This is the push-to-talk path.

### Tool

6. **Declare a tool.** A function named `get_current_time` with no arguments that returns an ISO-8601 timestamp. In your system prompt, instruct the model to use the tool when asked about the time.

7. **Call the tool.** In a turn, ask "what time is it?" Watch the stream of `response.function_call_arguments.delta` events, then `.done`. Log the `call_id`.

8. **Return the result.** Post a `conversation.item.create` with `type: "function_call_output"`, matching `call_id`, and the ISO timestamp. Then send `response.create`. Confirm the model speaks the time.

9. **Cover the gap.** While the tool runs, send a short filler `response.create` with `modalities: ["audio"]` and an instruction like "briefly say 'one moment'." Compare the user-perceived gap with and without the filler.

### Interrupt

10. **Deliberately interrupt.** Start a long response (ask the model to "count slowly from one to twenty with short pauses between numbers"). While it is speaking, start sending user audio again (if using server VAD) or send `response.cancel` (if manual). Observe the `response.done` with `status: "cancelled"`.

11. **Truncate the history.** After the interrupt, send a `conversation.item.truncate` for the model's last item with an `audio_end_ms` reflecting what the user actually heard. On the next turn, confirm the model does not reference content it generated after the interrupt (ask it "what was the last number you said?" — the correct answer is the last number the user heard, not the last number it generated).

### Prompt caching sanity check

12. **Measure the cache.** Keep the same `instructions` string and `tools` config across three successive sessions. Log the `usage` payload on each `response.done`. Observe `cached_tokens` (or the current field name) on the second and third sessions. Change one character in the `instructions`; observe the cache miss. Discuss in the write-up what the delta implies about stable-prefix design.

## Starter guidance

- **Chunk size.** Start with 100 ms of 24 kHz PCM16 per `input_audio_buffer.append` and let WAV concatenation be your audio sink. You can tune later.
- **Event log format.** One line per event, with wall-clock millis, event type, and the salient fields (e.g., chunk size, text delta, voice, call_id). Do not dump the raw base64 — it is noise.
- **Logging reads from a queue.** Your event loop reads events into a queue; a logger thread formats them. If you log inline you will block the socket read.
- **Playback timing.** Write the audio-delta bytes into a WAV file in order; do not try real-time playback on the first pass. Real-time playback is a distraction from the lifecycle.
- **One concern per iteration.** Do the plain happy-path first. Then layer tool call. Then layer interrupt. Then layer manual-commit. Do not try to build the full lifecycle in one pass.
- **Verify the model ID.** The `gpt-4o-realtime-preview-*` model names rotate; check the current list in [platform.openai.com/docs/models](https://platform.openai.com/docs/models) before you connect.
- **The `OpenAI-Beta: realtime=v1` header is required.** Missing it is the most common "my socket connects and immediately closes with 401" cause.

## Acceptance criteria

- [ ] Working client in Python or Node that connects, configures, streams audio, receives audio, and closes cleanly.
- [ ] One end-to-end event log, in time order, covering the happy path with server VAD.
- [ ] One end-to-end event log for the manual-commit path.
- [ ] One end-to-end event log for the tool-call path, including the filler and the real response.
- [ ] One end-to-end event log for the interrupt path, showing `response.cancel` or VAD-driven cancellation + `conversation.item.truncate`.
- [ ] Prompt-cache usage logged across three sessions, with the cached-tokens field named and the before/after of a small `instructions` change recorded.
- [ ] One-page write-up answering:
  - What is the shape of a `response.done` payload (fields, usage structure)?
  - How does the server behave when you send `input_audio_buffer.commit` with a buffer below the vendor's minimum audio length?
  - What happens if you forget `conversation.item.truncate` after an interrupt — reproduce once and describe the misbehaviour on the next turn.
  - How long is the real p50 TTFB (from the last `append` before the VAD commit to the first `audio.delta`)? Measure on at least 10 turns.

## Stretch goals

- **Repeat on WebRTC.** Mint an ephemeral token via `POST /v1/realtime/sessions`; have the browser (or a Python WebRTC client) do the SDP exchange; wire the same lifecycle on the data channel. Compare perceived latency.
- **Semantic VAD.** Reconfigure with `turn_detection.type: "semantic_vad"` and run the same walk. Record qualitative differences on utterances that contain mid-sentence pauses ("so I was thinking — three things").
- **Second voice swap.** Send a `session.update` mid-session changing the voice. Confirm whether the swap takes effect on the current turn or the next.
- **Audio stream consistency.** Intentionally send an append of the wrong sample rate (e.g., 16 kHz PCM when configured for 24 kHz). Describe the failure mode — does the server error, does it silently resample, does audio play back at the wrong pitch?
- **Cold vs. warm connection TTFB.** Measure the TTFB of the first turn on a fresh WebSocket versus the second turn on the same socket. Record the delta; discuss what it implies for session-reuse design.

## Non-goals

- Shipping a UI. A terminal + a WAV file is enough.
- Barge-in choreography beyond the one interrupt exercise above — Module 106 is where that lands.
- Deploying to production. Handling reconnect, backoff, and rate-limits is out of scope here.

## Keep it

Your event-log scripts become the artefact you reach for when debugging a Realtime API bug in the wild: a reproducible, event-level view of the socket. Keep the Python / Node file; every subsequent Realtime bug you hit will be some variant of "this field did not do what I thought it did," and a 20-minute re-run of this walkthrough against the current API tells you.
