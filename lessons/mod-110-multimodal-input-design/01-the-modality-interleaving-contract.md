# 01. The Modality Interleaving Contract

## Why this chapter

A voice agent that *also* accepts images, documents, camera frames, or screen share is no longer a voice pipeline — it is a **multimodal session** with voice as one of several concurrent input tracks. The temptation is to glue a vision model onto the existing voice pipeline and call it done. The problem is that the voice loop has a sub-second turn-gap budget (mod-109), and adding a 500 ms vision call on every turn blows straight through it. Worse, the user is now operating in two modalities at once — what they *say* and what they *show* may or may not be related, may arrive in different orders, and may each warrant a different response policy.

The design artefact that makes all of this tractable is a **modality-interleaving contract**: an explicit statement, agreed before you write any code, of which modalities the agent accepts, how it decides which one matters for the current turn, and what shape of response each modality can produce. This chapter defines that contract. Subsequent chapters wire each modality into real providers.

## Voice-first-with-vision-on-demand vs. true multimodal

The first design decision is the *UX shape*. There are two useful archetypes, and almost every real product sits closer to one than the other.

### Voice-first with vision on demand

The agent is fundamentally a voice interface. The user talks; the agent talks back. Occasionally the user attaches an image ("look at this error message", "what is this plant"), and the agent has to switch into a vision-enabled turn — look at the image, reason about it, respond in voice. The image is a **tool call**, not a continuous stream.

Signals that this is your shape:

- Vision use is rare (a few percent of turns, up to maybe ~30% in support-ish domains).
- Most turns can be answered without pixels at all.
- The user is on a phone-call-shaped device: hands and eyes busy, occasional attachments.
- Latency budget is tight; you cannot afford a vision call on every turn.

### True multimodal / vision-grounded conversation

The agent is looking at *something* for the duration of the session — a shared screen, a camera feed, a document. Every turn is grounded in what's on screen *right now*. The user may not even mention the visual context explicitly; the agent is expected to reference it.

Signals that this is your shape:

- The session has a durable "surface" (screen share, document on the table, camera pointed at a workbench).
- Turns are often about "this", "that", "here" — deictic references to the visual context.
- The agent is a co-worker on a specific artefact, not a conversationalist with occasional show-and-tell.
- Vision access is the whole point; the voice channel is the *control* channel.

### Two different contracts, not one

Collapsing these into one architecture is the single most expensive mistake in multimodal voice design. The voice-first design must *decide* whether to pull in vision for a given turn; the true-multimodal design must *throttle* the vision stream so it doesn't bankrupt you. They have different latency budgets, different cost curves, different privacy envelopes, and different failure modes. Pick which one you are building, write it down, and let every subsequent decision flow from that.

If your product spans both — e.g., a support agent that is usually voice-only but can escalate into screen share — architect them as **two modes** with an explicit transition, not as a continuum.

## The four pieces of a modality-interleaving contract

Regardless of which archetype you picked, a usable contract has four pieces. If any of them is implicit, your implementation will have a bug that nobody can describe clearly enough to fix.

### 1. The modality manifest

List every input modality the agent accepts, with its arrival pattern:

| Modality | Arrival pattern | Who initiates | Typical size |
|---|---|---|---|
| Voice | Streaming, continuous | User | ~16 kbps opus / 64 kbps G.711 |
| Still image (uploaded photo, screenshot) | Discrete, one-shot | User (sometimes tool) | 100 KB – few MB |
| Scanned document (PDF, multi-page image) | Discrete, one-shot | User | A few MB – tens of MB |
| Camera frame (single capture) | Discrete, triggered | User or agent tool | ~200 KB |
| Camera stream (continuous video in) | Streaming | User | ~1–3 Mbps |
| Screen share | Streaming or per-frame polled | User | Variable |
| Tool-returned image artefact | Discrete, mid-turn | Agent tool call | Small to medium |

For *each* modality your product supports, the manifest commits you to its arrival pattern, which drives the transport choice (Realtime API tool call for discrete images, Gemini Live `video_in` for streaming frames, etc. — see Chapters 2 and 3).

### 2. The interleaving policy

Given that voice and vision can arrive together, out of order, or independently, you have to decide how turns are assembled. Four reasonable policies, listed from least to most tightly coupled:

- **Modality-independent turns.** Each image upload or screen-share tick is its own turn; voice turns are separate. The LLM sees one modality per call. Cheapest, least contextual.
- **Sticky visual context.** The most recent image (or screen frame) is attached to every voice turn until the user replaces it or the session times out. Common for support chatbots.
- **Deictic binding.** The agent decides per-turn whether to attach the visual context, triggered by linguistic cues ("look at this", "on the screen", "in the photo") or by ASR confidence signals. Most selective, most logic.
- **Fully multimodal turn.** Every turn is a multimodal LLM call with the latest visual frame(s) attached. Default for true-multimodal / screen-share agents.

The choice is not stylistic — it is a direct input into cost, latency, and context length. Chapter 5 is dedicated to making this choice quantitatively.

### 3. The ordering guarantees

When a user says "is this correct?" at timestamp *t*, and an image arrives at timestamp *t + 200 ms*, which one does the agent process first? Three plausible answers:

- **Wait for the image, then process the voice turn.** Blocking join; the agent delays the turn to collect the referenced image. Small latency hit, clearly-grounded answer.
- **Process voice immediately, image arrives in the next turn.** Streaming behaviour; the agent answers "which one?", image lands, user retries. Lower latency, higher friction.
- **Process voice immediately but attach the image once it lands, allow the agent to self-correct.** Only works with a streaming LLM that can retract/amend; mostly not worth the complexity.

Pick one and document it. The alternative is that the ordering is determined by whichever race happens to win, and bug reports become unreproducible.

### 4. The modality-aware response contract

The agent's response is still voice (usually). But what the agent is *allowed to say* depends on which modalities fed the turn:

- If the response claims something about an image, is it required to cite the image region or OCR'd text? (See Chapter 4.)
- If the vision model is unsure, what does the agent say? ("I can see something that looks like X but I'm not confident" vs. "Could you reposition the camera so I can see Y clearly".)
- If the user asked a question that *requires* vision and no image was attached, does the agent ask for one, assume one, or refuse?
- Are there modality-specific **refusal shapes** (medical image interpretation, personal-document PII, screen-share content containing passwords)?

Write these as a short policy table. The policy sits with the system prompt and the orchestration layer, not with the vision model.

## A worked example: a support agent

To make the contract concrete, here is a condensed example for a consumer-electronics support agent.

```yaml
archetype: voice-first-with-vision-on-demand
modalities:
  voice:
    arrival: streaming
    transport: webrtc_opus
  image_upload:
    arrival: discrete
    transport: https_multipart
    max_size_mb: 20
    accepted_mime: [image/jpeg, image/png, image/webp, application/pdf]
  tool_returned_image:
    arrival: discrete_midturn
    transport: internal_tool_response
interleaving_policy: deictic_binding
ordering: wait_for_image_up_to_1500ms_then_proceed
vision_attachment_triggers:
  - utterance_matches: ["look at", "on the screen", "in the photo", "this error"]
  - asr_confidence_below: 0.6_with_recent_image
  - tool_call_returns: image
response_contract:
  grounded_claims_must_cite: true
  cite_shape: [bounding_box, ocr_excerpt]
  on_low_vision_confidence: ask_for_better_image
  refusals:
    - domain: medical_image_interpretation
      response_template: medical_refusal_v1
    - domain: contains_credit_card_number
      response_template: pii_redaction_v1
```

This isn't a library format — it's a design artefact. The point is that every decision above is *explicit* and reviewable before a single API call is wired.

## What goes wrong without a contract

Three failure modes you will reliably hit if you skip this chapter.

- **Silent modality dropping.** The user uploads a photo at the exact moment a VAD endpoint fires. The agent answers the voice turn without the image. The user's next turn is "did you see the photo?", and now you have a conversation about the agent's failure rather than about the user's problem.
- **Vision cost blowouts.** With a fully-multimodal-turn default and no throttling, a 20-minute screen-share session can run 1,200 turns × one high-resolution frame per turn through a vision model. Chapter 5 shows the arithmetic; the headline is that this can be a 10–100× cost multiplier vs. voice-only.
- **Un-redactable screen share.** A support agent is screen-sharing with a user, the user's bank website scrolls into view, the agent now has an obligation (from your policy, from GDPR, from PCI-DSS) to not store that frame, and you built no mechanism to drop frames by content. The privacy envelope for continuous video is not the same as for voice; Chapter 3 is dedicated to it.

All three are preventable with a written contract reviewed by product + privacy + engineering before implementation.

## Summary

- Decide the archetype first — **voice-first with vision on demand** vs. **true multimodal / vision-grounded** — and let the rest of the design flow from that choice.
- Write a **modality-interleaving contract** with four pieces: the modality manifest, the interleaving policy, the ordering guarantees, and the modality-aware response contract.
- The hardest decision is the **interleaving policy** (modality-independent turns, sticky visual context, deictic binding, fully multimodal); it drives cost and latency more than any single provider choice.
- Make the contract reviewable: a one-page document before a single wire is soldered. Chapters 2–5 wire each piece.

Next chapter: how to actually wire image and document input into GPT-4o, Claude, and Gemini — picking the right entry point per provider and managing image caching.
