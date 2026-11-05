# exercise-03: Gemini Multimodal Live with Continuous Video and Screen Share

**Estimated effort:** 3 hours · **Output:** a working Gemini Live prototype with throttled video + a frame-sampling / consent design writeup

## Objective

Build a minimal **Gemini Multimodal Live** prototype that accepts mic audio and a video source (screen share *or* webcam), applies a frame-sampling policy, respects a working consent / revoke flow, and produces audio responses grounded in what the agent has recently seen. The point is not a polished UI — it is to make the frame-sampling and consent trade-offs from Chapter 3 concrete.

## Prerequisites

- [Chapter 3 — Continuous Video and Screen Share via Gemini Multimodal Live](../03-continuous-video-and-screen-share.md).
- Access to Google AI Studio or Vertex AI with the current Gemini Live model.
- A browser or desktop environment with microphone and `getDisplayMedia` / `getUserMedia` support, or a server-side harness that can feed audio and periodic images into the Live API.
- Working voice I/O (reuse whatever you built in mod-102 / mod-105 if you have it).

## Steps

1. **Read the current Gemini Live API docs.** The semantics for session setup, audio-in format, `video_in` cadence, interrupts, and session termination are provider-specific and evolve. Date your notes.

2. **Stand up a bidirectional Live session.** Open the WebSocket, send a configuration message with your system prompt and tool list, send audio frames at the required sample rate, read audio-out back, and play it. Confirm you can have a voice-only conversation end-to-end.

3. **Add a video source.** Pick screen share (`getDisplayMedia`) or webcam (`getUserMedia`). Add a frame sampler in your client / media server that pulls a frame from the video track on a configurable cadence. Encode each frame as JPEG at a modest quality (start with 75% quality, long side 1024 px) and send it on the Live session's video channel.

4. **Implement three frame-sampling policies behind a flag:**
   - **Fixed cadence** — one frame every N seconds (configurable).
   - **Event-driven** — hash each candidate frame (perceptual hash or simple downsampled-SHA), send only if the distance from the last-sent frame exceeds a threshold.
   - **Tool-triggered** — a `look_now()` tool the model can call; your orchestrator fulfils it by capturing a fresh frame and injecting it as a video frame on the session.
   Default to event-driven with a 20 s heartbeat ceiling and `look_now` available.

5. **Build a consent / revoke flow.** Before vision starts, show an explicit consent prompt that is separate from any voice consent. While vision is live, expose a one-click "stop sharing" control that:
   - Immediately stops sampling.
   - Tells the Live session that the video source is gone (either by stopping the video send or by signalling via the session's control channel).
   - Switches the agent to voice-only mode gracefully (the agent should not keep referring to "what I see on your screen").
   Verify the revoke by sharing a screen, saying "describe what you can see", revoking mid-answer, and asking the same question again.

6. **Add a content-aware frame gate.** Before any frame leaves the client, apply at least one content check:
   - A regex match against OCR'd text for 13–16-digit numbers (credit card numbers).
   - A URL allow/block list for the currently-focused tab (screen-share only).
   - Any domain-specific check appropriate to your exercise-01 scenario.
   When the gate fires, either drop the frame entirely or blur / crop the offending region. Log the decision (hash + timestamp + rule) without storing the frame bytes.

7. **Exercise barge-in with vision.** Mid-answer, interrupt the agent with voice. Confirm:
   - Audio-out stops locally within ~100 ms.
   - Video-in stream *keeps flowing* — the agent still has context for the next turn.
   - The agent's next response reflects the interruption.

8. **Instrument frame telemetry.** For each sent frame, log:
   - The sampling policy that produced it (cadence, event, tool).
   - Frame hash.
   - Any gate decision.
   - Session-local timestamp.
   For each session, aggregate: total frames sent, per-policy breakdown, dropped frames by gate rule.

9. **Write a short design writeup.** 1–2 pages. Cover:
   - The sampling policy you defaulted to and the parameters you chose, with justification.
   - The consent flow and the revoke test result.
   - At least one failure mode you discovered while running the prototype that the written chapter did not cover.
   - What you would change to deploy to real users, including any privacy artefacts (DPIA, consent copy, retention policy) that are not technical.

## Starter guidance

- **Keep the UI ugly.** Spending time on CSS is not the point. A single-page web app with a `share screen` button, a `stop` button, and a transcript log is sufficient.
- **Measure frame cost.** Even if you cannot see per-frame billing in real time, you can count frames × resolution × model to estimate cost. Add this to your telemetry so the sampling policy's cost impact is visible.
- **Do not store frames.** Default to in-memory frame handling with explicit discard; store only hashes + timestamps + gate decisions in your audit log. Treat this as the production default from day one.
- **Barge-in is not just audio.** Chapter 3's degradation modes section is where this exercise goes next; if you ship this to users, each of those modes needs a tested path.
- **Clock the session end.** Gemini Live sessions have time limits; design your prototype to re-open a session gracefully rather than falling off a cliff.

## Acceptance criteria

- [ ] A working Gemini Live session accepting mic audio + at least one video source.
- [ ] Frame-sampling implemented for all three policies (fixed cadence, event-driven, tool-triggered) behind a flag.
- [ ] Consent prompt distinct from voice consent + a revoke control with documented behaviour.
- [ ] A content-aware frame gate with at least one specific rule, logged without frame bytes.
- [ ] Barge-in test documented: audio stops, video keeps flowing, next turn reflects interruption.
- [ ] Per-session telemetry: frames sent, per-policy breakdown, drops by gate rule.
- [ ] A 1–2-page design writeup.

## Stretch goals

- **Server-side key-frame index.** Store a hash → frame map server-side (not in the Live session) and let the agent reference "the slide you showed 2 minutes ago". Measure how often this reference is used.
- **Dual-camera scenarios.** For the field-technician archetype, support webcam + document-camera together. Decide which stream to sample, how often, and how to reference them in grounding.
- **Comparative run on OpenAI Realtime (if video input is available).** Side-by-side test of Gemini Live vs. OpenAI Realtime on the same scenario; report deltas on latency, cost, and streaming behaviour. <!-- needs-research: verify current OpenAI Realtime video-input availability at access date -->
- **Automated consent audit.** Build a small reviewer tool that replays the telemetry and flags any session where a frame was sent before explicit consent was granted. This is the kind of audit a privacy team will want.

## Keep it

This prototype is the basis for project-102 (multimodal voice agent). The frame-sampler, consent flow, and telemetry are the parts you will reuse directly; the Live-session wrapper may be replaced if you change provider, but the design around it is portable.
