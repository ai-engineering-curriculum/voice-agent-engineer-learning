# exercise-02: Google Gemini Live — Session Lifecycle Walkthrough

**Estimated effort:** 3 hours · **Output:** a working, instrumented Gemini Live client that exercises audio + video + tool-calling + search grounding, plus a short write-up comparing lifecycle details against the OpenAI Realtime walk from exercise-01.

## Objective

Walk the Gemini Live API's session lifecycle the same way you walked OpenAI's in exercise-01, with the addition of two capabilities the Realtime API does not expose today: **continuous video / screen-share frames inside a voice turn**, and **Google Search grounding** as a first-class tool. By the end you will have an instrumented client, event logs for each lifecycle phase, and a comparison matrix that names the specific places the two stacks' grammars diverge.

The point is not to decide which stack is "better." The point is that the architectural decisions in later modules (barge-in, latency budget, multimodal input) depend on knowing these differences by heart, not by reading.

## Prerequisites

- [Chapter 2 — The Gemini Live API: bidirectional streaming, native audio, and continuous video](../02-gemini-live-api-session-lifecycle.md).
- Exercise-01 completed (so the comparison has something to compare to).
- A Gemini API key from [aistudio.google.com](https://aistudio.google.com/) with Live-API access, or a Vertex AI project with the Live-API endpoints enabled.
- Python 3.10+. The `google-genai` SDK is the current reference client; you may also roll your own WebSocket client against the wire protocol if you want to see every byte.
- A microphone + speakers, or a prerecorded PCM file + WAV sink. A webcam or a screen-capture source (JPEG frames at ~1 fps is enough).

## Requirements

Build one program that exercises each phase of the lifecycle at least once.

### Core walk

1. **Connect and send `setup`.** Configure:
   - `response_modalities: ["AUDIO"]`,
   - a `speech_config` with one of the current prebuilt voice names,
   - a short `system_instruction`,
   - one function tool (see §Tool),
   - `realtime_input_config.automatic_activity_detection` (default behaviour).

   Log the `setup_complete` echo.

2. **Append audio.** Send PCM16 at **16 kHz, mono** in `realtime_input.media_chunks[].data` with `mime_type: "audio/pcm;rate=16000"`. Chunk at ~100 ms.

3. **Let server VAD commit the turn.** Observe server behaviour: `server_content` events with `input_transcription` (opt in), `output_transcription`, `model_turn.parts[].inline_data` (audio at **24 kHz**), and `turn_complete`.

4. **Play audio.** Write the 24 kHz output to a WAV file or pipe to speakers. Confirm the sample-rate mismatch between input (16k) and output (24k) is handled correctly by your sink.

5. **Manual turn boundaries.** Reconnect with `realtime_input_config.automatic_activity_detection: { "disabled": true }` and mark turn boundaries explicitly with `activity_start` / `activity_end`. Confirm the same contract works.

### Video / screen-share

6. **Continuous video.** Add a second producer that captures a webcam or screen frame at **1–2 fps** as JPEG and sends each frame in `realtime_input.media_chunks` alongside audio. The frames interleave with audio on the wire; the model consumes both.

7. **Ask the model about what it sees.** In one turn, speak "what do you see on screen right now?" and confirm the model's spoken response references content that actually appeared in the last frame window. Log the input-transcription and output-transcription alongside the frame timestamps.

8. **Frame cadence trade-off.** Re-run the same query at 0.5 fps and at 4 fps. Record qualitative differences: does the model's response feel more or less responsive? Does TTFB change? Note that model attention to the video lane is probabilistic — some frames are not attended to.

### Tool and grounding

9. **Declare a function tool.** `get_current_time` with no arguments, same as exercise-01. Also declare the built-in `google_search` tool in `setup.tools` — this is the grounding path.

10. **Function tool call.** Ask "what time is it?" Capture the `tool_call` event; execute; send `tool_response` with the matching `id`; confirm the model speaks the time.

11. **Grounded answer.** Ask "what's the weather like in Berlin right now?" Observe the model use `google_search` grounding. Capture the `grounding_metadata` on the response and log the citation URLs.

### Interrupt

12. **Interrupt.** Start a long response (count slowly from one to twenty with pauses). Start speaking again; observe the server emit `server_content` with `interrupted: true`. Flush your playback buffer; confirm the model listens again cleanly.

13. **Manual interrupt.** In manual-VAD mode, pre-empt by sending `activity_start` while the model is still speaking. Observe behaviour; compare with the server-VAD path.

### Session duration + resumption

14. **Bump against the duration ceiling.** Let the session run — e.g., 10–15 minutes of alternating turns. Note what the vendor's documented ceiling is for the model you chose; note whether you hit it. If the API exposes a session-resumption field, enable it; otherwise, design and document the reconnect handoff you would ship.

## Write-up: the comparison matrix

Produce a short matrix comparing Realtime and Gemini Live on these dimensions, with observations from your own two walks:

| Dimension | OpenAI Realtime (exercise-01) | Gemini Live (this exercise) |
|---|---|---|
| Config change mid-session? | | |
| Input audio sample rate | | |
| Output audio sample rate | | |
| Automatic VAD knobs | | |
| Manual turn boundaries field | | |
| Tool-call event shape | | |
| Native video / screen input | | |
| Search grounding | | |
| Interrupt signal | | |
| Session duration ceiling | | |
| SDK maturity (Python) | | |

Add bullets at the bottom for "gotchas that cost me more than 15 minutes."

## Starter guidance

- **Sample rates differ in and out.** 16 kHz in, 24 kHz out. If your player thinks everything is 24 kHz you will get chipmunks on your own audio and correct playback of the model's. Test with both separately.
- **1–2 fps video is the starting point.** Higher costs latency and tokens; lower makes the model miss changes. Do not start at 30 fps.
- **Configuration is one-shot.** You cannot change `response_modalities` or `voice_config` mid-session. Reconnect for any such change.
- **Live-API model IDs rotate.** Verify current model names and which variants support audio output at [ai.google.dev/gemini-api/docs/live](https://ai.google.dev/gemini-api/docs/live).
- **Frame encoding.** JPEG quality ~75 is usually fine; quality 95 wastes bandwidth. Downscale to ~1280 wide for screen-share (so text is readable); smaller for face / scene video.
- **Grounding attribution.** `grounding_metadata` includes citations; your UI (or your spoken response) should reference them when the grounded content is user-visible. This is a safety and transparency obligation (Module 108).

## Acceptance criteria

- [ ] Working Python client (SDK- or raw-WebSocket-based) that connects, sends `setup`, exchanges audio, and closes cleanly.
- [ ] Event log for the happy-path turn with server VAD and audio output.
- [ ] Event log for the manual-VAD turn.
- [ ] Event log for a turn with continuous video and a spoken question about the frames.
- [ ] Event log for a turn with a function tool call and a `tool_response`.
- [ ] Event log for a turn with `google_search` grounding, with citation URLs captured.
- [ ] Event log for an interrupted turn, showing `interrupted: true` and the playback-flush client-side behaviour.
- [ ] Short comparison matrix as described above, with at least one line of empirical observation per cell.
- [ ] One-page write-up answering:
  - What is the practical cadence trade-off you observed at 0.5 fps vs. 1–2 fps vs. 4 fps?
  - How does the interrupt experience feel compared to Realtime — qualitatively and in terms of time-to-silence?
  - What is one feature of Gemini Live's lifecycle that would change your Chapter 4 architectural decision for one of your real deployments?

## Stretch goals

- **Screen-share + OCR check.** Point your screen capture at an app with legible text; ask the model to read a specific line. Measure accuracy as frame resolution and cadence change.
- **Multi-turn with session resumption.** If the SDK / API exposes session resumption or context compression, enable it and reconnect at the ceiling. Keep a running "did the model remember X" test.
- **Audio-only input, text-only output.** Rerun the happy path with `response_modalities: ["TEXT"]`. Measure the latency-to-first-text-token vs. latency-to-first-audio-delta.
- **Vertex AI deployment.** Run the same client against the Vertex AI endpoint with a service-account auth. Compare behaviour; note any differences in model availability or latency.
- **Cross-stack A/B.** Give the same system prompt and tool catalog to Realtime and Live; run identical user utterances against both; produce transcripts side by side.

## Non-goals

- Production-grade reconnection, authentication, or UI.
- Comparing voice-naturalness or choosing between the two for production (Chapter 4's job).
- Running Gemini Live on a mobile device (that is a Module 107 concern).

## Keep it

Your Gemini Live client + the comparison matrix are a durable reference. When a future deployment decision asks "which socket for this ingress shape," the matrix tells you the plumbing differences without having to re-read docs.
