# mod-110-multimodal-input-design: Multimodal Input Design — Vision, Screen Share, and Video in a Voice Agent Turn

**Estimated effort:** 14 hours · **Role:** Voice & Multimodal Agent Engineer · **Level:** L35

## What this module is for

A voice agent that also accepts images, documents, camera frames, or screen share is not a voice pipeline with vision bolted on. It is a **multimodal session** where voice is one of several concurrent input tracks, each with its own arrival pattern, cost profile, latency floor, and privacy envelope. The goal of this module is to make the design decisions around that session **explicit and reviewable** — the modality contract, the vendor wiring, the frame-rate and routing policy, the grounding scheme, and the eval harness — so that implementation choices flow from a written design rather than from whichever provider SDK had the clearest example.

This module sits between mod-109 (the latency budget you inherit for any turn) and mod-111 (the orchestration layer that the modality decisions plug into). It also feeds the eval story that mod-112 formalises.

## Learning objectives

- Design the modality-interleaving contract for a voice agent that also consumes vision — user speaks + shows an image (uploaded photo, document scan, camera frame, screen share) or continuous video; recognise when the right UX is still voice-first with vision on demand vs. true multimodal.
- Wire image / document input via GPT-4o vision, Claude vision, Gemini vision — pick the entry point (Chat / Messages / `generateContent` inline, Realtime API tool call with image argument, Gemini Live with `video_in`) and manage image caching for cost / context-length.
- Wire continuous video + screen share via Gemini Multimodal Live — architect the session, decide frame-rate throttling, decide what the bot 'sees' every N seconds vs. on-event vs. tool-triggered; manage the privacy / consent envelope distinct from voice consent.
- Design multimodal grounding — when the LLM should cite the image region or screen area it reasoned over (bounding boxes, cropped excerpts, OCR'd text passed as evidence); design the refusal shape when vision is ambiguous.
- Build the modality-routing heuristic — cheap text-first path for the N% of turns where vision is unnecessary, upgrade to vision on signal (user mentions "look at this"; tool call returns an image artefact; low ASR confidence may benefit from a recent image); watch the cost curve.
- Evaluate multimodal turns — MMMU + MM-Vet + LLaVA-OneVision benchmarks for the model baseline; pairwise human eval on in-domain multimodal turns; measure vision-grounding accuracy (did the answer actually use the image?).
- Cite the sharp boundary to `applied-ai-engineer` (L30, peer) on in-house image-input feature engineering and to `nlp-engineer` (L30, peer) on deep multimodal-model research.

## Lecture chapters

1. [01 — The Modality Interleaving Contract](01-the-modality-interleaving-contract.md)
2. [02 — Wiring Image and Document Input Across GPT-4o, Claude, and Gemini](02-wiring-image-and-document-input.md)
3. [03 — Continuous Video and Screen Share via Gemini Multimodal Live](03-continuous-video-and-screen-share.md)
4. [04 — Multimodal Grounding and Refusal Shapes](04-multimodal-grounding-and-refusal.md)
5. [05 — The Modality-Routing Heuristic and Cost Curve](05-modality-routing-and-cost-curve.md)
6. [06 — Evaluating Multimodal Turns](06-evaluating-multimodal-turns.md)
7. [07 — The Boundary to Applied-AI and NLP Engineering](07-boundary-to-applied-and-nlp.md)

## Exercises

- [exercise-01 — Modality Interleaving Contract Design](exercises/exercise-01-modality-interleaving-contract-design.md) (2 h)
- [exercise-02 — Wire GPT-4o, Claude, and Gemini Vision Paths](exercises/exercise-02-wire-gpt-4o-and-claude-and-gemini-vision-paths.md) (3 h)
- [exercise-03 — Gemini Multimodal Live with Continuous Video and Screen Share](exercises/exercise-03-gemini-multimodal-live-with-continuous-video-and-screen-share.md) (3 h)
- [exercise-04 — Multimodal Grounding with Citation (Bounding Boxes or OCR)](exercises/exercise-04-multimodal-grounding-with-citation-bounding-boxes-or-ocr.md) (2 h)
- [exercise-05 — Modality Routing Heuristic with Cost Attribution](exercises/exercise-05-modality-routing-heuristic-with-cost-attribution.md) (2 h)
- [exercise-06 — Multimodal Benchmark Walkthrough — MMMU, MM-Vet, Grounding Accuracy](exercises/exercise-06-multimodal-benchmark-mmmu-and-mmvet-walkthrough.md) (1 h)

## Structure

- `01-…md` … `07-…md` — lecture chapters.
- `exercises/` — per-exercise prompts.
- `labs/` — long-form hands-on labs (not required for this module; exercises 02–06 together constitute the hands-on work and build on each other).
- `quizzes/` — knowledge checks.
- `resources.md` — primary-source references.

## How to use this module

Read the chapters top-to-bottom first — each one assumes the contract defined in Chapter 1 and the wiring patterns in Chapter 2. Then work the exercises in order: exercise-01 is the paper contract that anchors everything else; exercises 02–05 are hands-on and build a working multimodal prototype; exercise-06 is the eval pass that plugs into mod-112.

A rough pacing suggestion: one evening on Chapters 1–2 and exercise-01; one evening on exercise-02 (the provider wiring); one evening on Chapter 3 and exercise-03 (Gemini Live + consent); one evening on Chapters 4–5 and exercises 04–05 (grounding + routing); one short session on Chapters 6–7 and exercise-06 (eval + boundary).

Prerequisite modules: [mod-102](../mod-102-speech-pipeline-foundations/README.md), [mod-105](../mod-105-realtime-speech-to-speech/README.md), [mod-108](../mod-108-safety-compliance-voice/README.md), [mod-109](../mod-109-latency-budget-architecture/README.md).

Next module: [mod-111-voice-agent-orchestration](../mod-111-voice-agent-orchestration/README.md).
