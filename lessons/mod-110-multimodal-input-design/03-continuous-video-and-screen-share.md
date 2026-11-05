# 03. Continuous Video and Screen Share via Gemini Multimodal Live

## Why this chapter

A true-multimodal agent — the archetype where the user is screen-sharing or pointing a camera at a workbench — needs *continuous* vision, not a one-shot image per turn. At the same time, feeding the model a full video stream at native frame rate is wasteful: the user says one thing per few seconds, the agent only needs a representative frame per decision, and you are paying per-frame vision costs either way.

This chapter is specifically about **Gemini Multimodal Live** — the surface that today most directly accepts a live audio-plus-video stream over a single bidirectional connection — and the architectural decisions around it: how to shape the session, how to throttle frames, what "the bot sees" means in a streaming world, and how the privacy envelope for video is different from (and more demanding than) the one for voice.

> The patterns here apply to any live multimodal surface with similar semantics. At time of writing, Gemini Live is the most-documented option; OpenAI's Realtime API is adding video input; other providers are shipping similar surfaces. Verify the current capability of the specific API you depend on before architecting on it. <!-- needs-research: refresh provider-support matrix for streaming video-in on live LLM APIs at access date -->

## The Gemini Live session, in brief

Gemini's [Multimodal Live API](https://ai.google.dev/gemini-api/docs/live) is a bidirectional WebSocket. A session sends:

- **Audio in** (16-bit PCM at a defined sample rate),
- **Video in** (periodic image frames, typically PNG or JPEG),
- **Text in** (optional; mostly for system prompts and tool configuration).

And the server sends back:

- **Audio out** (synthesised speech),
- **Text out** (streaming text, for transcription or display),
- **Tool calls** (for function-calling flows),
- **Session-control messages**.

The session persists for minutes; the model sees a continuous audio track and the latest frames you have sent. Crucially, **you choose when to send a video frame** — the API does not require 30 fps. This is the whole latency/cost architecture.

## Frame-rate throttling: the only decision that matters

A naive implementation forwards every browser-decoded video frame (24–30 fps) to the API. That is both ruinously expensive and produces no better answers — the model does not need 30 frames per second to know what's on a slide. The architectural question is **how often to sample a frame**, and three policies cover almost all real usage.

### Policy 1 — fixed cadence

Send one frame every N seconds, regardless of what's happening. Simple, predictable, auditable.

- **1 frame / 1 s** — around the low end for a true "the agent is watching" feel. High cost; use only when every second of visual change matters.
- **1 frame / 3–5 s** — reasonable default for a co-pilot that reasons about screen content at conversational tempo.
- **1 frame / 10–30 s** — enough for a slow-changing surface (a long-form document, a static scene) where the agent mostly needs *recent* context.

Rule of thumb: pick the cadence against the *rate the surface actually changes*, not against the voice turn rate.

### Policy 2 — event-driven

Send a frame when something changed enough to be worth looking at. "Changed enough" is measured locally on your client / media server before the frame goes anywhere:

- **Pixel-difference threshold.** Hash each frame (perceptual hash) and send only if the hash distance exceeds a threshold. Cheap, resilient to noise.
- **Scene-change detection.** Use any off-the-shelf shot-change detector; send on change.
- **Application-level events.** On screen share, hook into whatever your client emits (window focus change, URL change in a shared browser tab, cursor idle → active transitions).

Event-driven policies cut cost 10× or more compared to fixed-cadence at the price of more code and a tuning problem (the threshold).

### Policy 3 — tool-triggered

The agent asks to see a frame when it needs one. The model has a `look_at_screen()` tool; the tool returns the latest frame; the model proceeds. This gives the *model* control of its own sampling rate. It is the most selective — many turns never call the tool — and it works well when:

- The agent is mostly voice-first and only occasionally needs vision confirmation.
- The user's turns are explicit ("look at the error on the top right").
- Pay-per-frame cost is high relative to per-turn cost.

Downside: the model has to be good enough to ask. In practice, mixing tool-triggered with a slow fixed cadence ("I always see a frame every 15 s, but I can ask for one on demand") is often the best spot.

### A hybrid default

For most true-multimodal agents, a reasonable default is **event-driven with a 15–30 s heartbeat ceiling and a `look_now` tool call for explicit requests**. You capture important changes, you bound the worst-case by the heartbeat, and the agent can self-serve if it wants a specific moment.

## Frame resolution and format

Per-frame cost is dominated by image tokens. Lower-resolution frames cost fewer tokens; the trade-off is detail. Gemini documents recommended preprocessing in the [vision](https://ai.google.dev/gemini-api/docs/vision) and Live API docs — check the current guidance before deciding on target resolution. A reasonable starting point:

- **JPEG at ~75% quality**, long side 768–1024 px for screen-share content (text-heavy).
- **JPEG at ~60% quality**, long side 512–768 px for camera-pointed-at-world scenes.

Trade quality down until your eval (Chapter 6) tells you it hurts.

## "What the bot sees right now" — the context window of frames

Even with throttling, over a 20-minute session you accumulate tens to hundreds of frames in the session context. Two questions:

- **How many of them does the model keep in its working context at any moment?** Provider-defined; check the current Live API docs for the sliding-window semantics. You cannot assume the model sees every historical frame. <!-- needs-research: confirm current Gemini Live context-retention semantics for historical video frames -->
- **Which frames should you *try* to keep in view?** Three useful classes:
  1. The most recent few frames.
  2. One frame near each user turn (captures what the user was looking at when they spoke).
  3. Agent-annotated key frames: when the agent or the user says "remember this screen", explicitly keep that frame around.

You can implement #3 by storing the frame in your own session memory and re-attaching it on subsequent turns via the Files API reference, rather than hoping the Live session retains it. This is the same pattern as RAG, applied to vision.

## Barge-in, cancellation, and the streaming graph (still applies)

Everything from mod-109 Chapter 3 still applies. The user interrupts; you cut audio playback; you cancel the model's current response; the video stream *keeps flowing* (because the agent still wants context for the next turn). Two Live-specific wrinkles:

- Gemini Live has a session-level "interrupt" semantic — check the current API docs for the current activityStart / activityEnd / barge-in message types. Send the right one; do not try to simulate barge-in by opening and closing the socket.
- The audio-out stream from the server will keep flowing for a moment after you ask it to stop; buffer locally so you can cut playback at your client without waiting for the server ack.

## The privacy / consent envelope

Continuous video is categorically more invasive than voice. The consent model has to be explicit, visible, and revocable *per modality*, not aggregated with voice consent.

### A minimum policy

- **Separate consent gestures.** Users must affirmatively enable screen share or camera; voice consent is not enough. Reflect this in both the UI and your audit log.
- **A visible indicator while vision is live.** A recording indicator on the user's screen or device — like native screen-share and camera-on indicators on every major OS — is the norm, not a nicety.
- **One-click revoke.** The user can stop screen share at any time, and the rest of the session continues in voice-only mode. The agent must handle that transition gracefully (see "Degradation modes" below).

### Content-aware frame dropping

A screen-share frame can contain things you do not want to send to a third-party LLM:

- Passwords (visible in password managers' autofill UI, in sudo prompts).
- Credit card numbers (checkout pages).
- Patient information or other PII (healthcare portals, email threads).
- Content that is not yours to transmit (another person's video window picked up on share).

Patterns that work:

- **Client-side masking.** Before frames leave the user's device, redact regions. On the web, [`CropTarget`](https://developer.mozilla.org/en-US/docs/Web/API/CropTarget) and the screen-share picker's `displaySurface` constraint limit what the user can share in the first place.
- **Content-classifier gate.** Run a small local or server-side classifier for high-risk content (OCR + regex for 16-digit numbers, URL allowlists / blocklists) and drop the frame or blur regions before it hits the LLM.
- **Audit-log the frame hash, not the frame.** You often cannot retain the frame bytes for compliance. Logging a perceptual hash + timestamp preserves debuggability without retaining content.

### Retention

Frames and the derived model outputs are both subject to retention rules. Even if your voice retention is 30 days, screen-share frames may need to be retained **zero days** — processed in-session and discarded. Design for this at ingestion, not later.

### GDPR / HIPAA / PCI-DSS implications

- **GDPR.** Video is personal data; consent must be specific, and the lawful basis must be documented. Subject access requests cover the video if retained.
- **HIPAA.** If the surface can contain PHI, the entire session may be a PHI processing environment; your BAA with the model provider must cover video.
- **PCI-DSS.** Cardholder data in screen share is a compliance landmine. Prefer *prevent* (crop / masking) over *detect-and-drop*.

Mod-108 covers the broader safety and compliance picture; this chapter's job is to make sure vision is not a silent exception to those policies.

## Degradation modes

The agent must behave correctly when vision drops out:

- **User revokes screen share mid-session.** Agent continues in voice-only, explicitly acknowledges the transition if asked.
- **Camera fails or network bandwidth collapses.** Frame rate drops to the heartbeat ceiling; if no frame has arrived in N seconds and the current turn needs vision, agent asks for a re-check.
- **Content-classifier drops a frame.** Agent either operates without that frame (and says so if asked) or triggers a soft refusal.
- **Model confidence on the latest frame is low** (e.g., motion blur, low light). Chapter 4 covers the refusal shape.

Each of these should be a documented state, not an unhandled exception.

## A reference architecture sketch

```
┌─ User device ────────────────────────────────┐
│                                              │
│  Microphone ──► (voice track)                │
│                                              │
│  getDisplayMedia / getUserMedia               │
│      │                                       │
│      ▼                                       │
│  Frame sampler ──► Pixel-diff / scene change │
│      │                                       │
│      ▼                                       │
│  Content-classifier gate                     │
│      │                                       │
│      ▼ (safe frames + voice)                 │
└──────┼───────────────────────────────────────┘
       │
       ▼
┌─ Media server ───────────────────────────────┐
│  Session orchestrator                        │
│    ├── Gemini Live session (audio, video, text)
│    ├── Tool-call executor                    │
│    └── Session memory (key-frame index via   │
│        Files API)                            │
└──────┼───────────────────────────────────────┘
       │
       ▼
   Audio back to user, plus optional text / UI updates
```

The parts worth emphasising:

- Frame sampling and content classification happen **before** the frame is on the wire. Once Google (or any third party) has the bytes, you cannot un-send them.
- The session orchestrator keeps its own **key-frame index** that is independent of what the Live session happens to retain. This buys you durable multi-turn vision grounding (Chapter 4).
- Tool calls are still the control plane for "take a close look at this specific region" — the model asking for a high-detail crop, for example, is a tool call, not a resend of the whole frame.

## Summary

- **Pick a frame-rate policy** — fixed cadence, event-driven, or tool-triggered — and size it against how fast the *surface* changes, not the voice turn rate.
- A reasonable default: **event-driven with a 15–30 s heartbeat ceiling and a `look_now` tool call** for on-demand vision.
- Keep **"what the bot sees"** under your control: sample frames, resize them, and maintain your own key-frame index via the provider's Files API rather than trusting the Live session to retain history.
- The **privacy envelope** for video is separate from and stricter than voice: explicit consent, visible indicators, one-click revoke, content-aware frame dropping, short retention.
- Define **degradation modes** for every way vision can drop out — the agent still has to behave.

Next chapter: multimodal grounding — how to make the model cite the image region it reasoned over, and the refusal shape when vision is ambiguous.
