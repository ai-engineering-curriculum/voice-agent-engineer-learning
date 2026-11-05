# 04. Multimodal Grounding and Refusal Shapes

## Why this chapter

A text-only RAG agent that claims "the manual says X" is expected to cite the passage. A multimodal agent that claims "the error at the top right of your screen is X" is expected to cite the *image region* that justifies the claim. Without that, the user cannot verify the agent is actually looking at the right thing; worse, the agent can produce a confidently wrong answer that *looks* grounded and has no way for a reviewer to falsify.

This chapter is about the two sides of multimodal verifiability: **grounding** (how the model shows what it reasoned over) and **refusal** (what the agent does when vision is ambiguous or absent).

## What "grounding" means in a multimodal turn

A grounded multimodal claim has three parts:

1. The **claim** — what the agent asserts. ("The error at the top of the page is 'invalid session token'.")
2. The **region** — where in the image / frame the evidence lives. (A bounding box, a crop, a pixel coordinate, an element selector.)
3. The **evidence excerpt** — the pixels, OCR'd text, or structural description that supports the claim.

The model might produce all three natively (modern vision models can emit bounding boxes with reasonable accuracy), or the orchestration layer might *verify* the claim by cross-checking against OCR / a DOM snapshot / a second model. Both are legitimate; neither is free.

## Representations of a grounding region

Four useful representations, listed in roughly increasing specificity:

### 1. Bounding box

A `[x0, y0, x1, y1]` rectangle, usually in normalized `[0, 1]` coordinates or absolute pixel coordinates. Produced by:

- Prompting the vision model to emit boxes alongside its answer (all three major providers support this via prompt instruction; quality varies).
- Running a separate detection / OCR pipeline (Tesseract, [Amazon Textract](https://aws.amazon.com/textract/), [Google Document AI](https://cloud.google.com/document-ai), [Azure Document Intelligence](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/)) that returns boxes natively, then matching them to the model's claim.

Bounding boxes are good enough for most display-overlay use cases ("highlight where I'm looking") and bad for narration ("the error at coordinates 124, 302" is not speakable; translate to "the error at the top right" before narrating).

### 2. Cropped excerpt

A rendered sub-image of the region the model cared about. Useful for UI rendering (show the user "here's what I saw") and for human eval (the eval reviewer sees exactly what the model saw, not the full frame).

Produce by cropping around the bounding box with a small margin. Keep the crops in your session memory alongside the model's response.

### 3. OCR'd text passed as evidence

If the claim is about text (an error message, a price, a column header), OCR the relevant region and store the extracted text as structured evidence. Then the agent's claim can be checked against the OCR'd text programmatically — "did the string `'invalid session token'` actually appear in the OCR?" is a trivial substring check, and a model that hallucinates a quote is caught immediately.

This is the single highest-leverage multimodal verification technique for document-heavy and screen-share workflows.

### 4. Structural / DOM-level references

For screen-share where the client can expose the DOM (or an accessibility tree), the grounding reference can be an element selector or an AXTree node — far more durable than pixel coordinates (page reflows, resolution changes). If you have DOM access, prefer it; if you don't, bounding boxes + OCR.

## Prompting for grounded output

The model does not emit grounding unless you ask for it. A prompt sketch for a screen-aware agent:

```
You can see the user's screen. When you answer, include a `grounding` field
in your response that explains, in machine-readable form, exactly what you
used to justify the answer. The format is:

{
  "spoken_response": "...",
  "grounding": [
    {
      "claim": "the error message at the top right is 'invalid session token'",
      "region": {"type": "bbox_normalized", "box": [0.78, 0.04, 0.98, 0.10]},
      "evidence": {"type": "text", "text": "invalid session token"}
    }
  ]
}

If you cannot ground a claim — for example, the screen is blurry, the text is
too small to read, or the region you need to see is not currently visible —
say so in `spoken_response` and set `grounding: []`. Do not invent regions
or text you did not actually see.
```

Three design notes:

- Separate **spoken** output from **grounded** output. The TTS pipeline consumes `spoken_response`; the UI and the audit log consume `grounding`.
- The orchestrator should **verify** at least one claim in `grounding` against the OCR / DOM before trusting the whole response. If verification fails, treat it as a low-confidence turn (see refusal shapes below).
- Never make `grounding` optional in the schema. "Optional grounding" collapses into "no grounding" within a release.

## Confidence and verification

The model's confidence in a region is not the same as the model's confidence in a claim. Two useful signals:

- **Cross-check OCR.** If the model claims a specific quoted string is visible, extract the quoted string and search the OCR of the same region. Mismatch → low confidence.
- **Self-consistency pass.** Ask a second, cheaper model or a second instance of the same model to independently answer the same question with the same image. Mismatch → uncertain.

Either is cheap enough to run on turns that are consequential (user asked "is it safe to click this?") and too expensive to run on every turn. Make this a policy, not a hidden default.

## Refusal shapes when vision is ambiguous

The wrong answer is worse than no answer. Four refusal shapes cover almost all ambiguous-vision cases; pick the right one per situation:

### "Ask for a better image"

The user sent an image; the model cannot do its job on it because of quality issues (blur, crop cuts off the subject, glare on a screen, upside-down document).

Template: *"I can see what looks like [brief description], but [specific issue] is making it hard to tell. Could you [specific fix: 'retake the photo with better lighting', 'send the image rotated upright', 'send a closer shot of the error message']?"*

Why it works: it acknowledges what the model saw, identifies the specific defect, and asks for a concrete fix. "I can't read this image" without a specific ask forces the user to guess what fix to try.

### "Ask for the thing you need"

The user asked a question that requires vision, no image is attached, and the last image attached is irrelevant or stale.

Template: *"To answer that, I'd need to see [specific thing: 'the error message you mentioned', 'the top of your screen where the account name shows', 'the label on the back of the device']. Could you [action: 'share your screen', 'take a photo of X']?"*

Avoid generic "could you send a picture"; specify what exactly you need.

### "I'm not sure, here's what I can tell"

The model has a low-confidence answer. Don't refuse outright; acknowledge the uncertainty, give the best guess, and offer verification.

Template: *"I think [claim], but I'm not fully sure — [specific uncertainty source]. Can you confirm [specific thing the user can verify]?"*

### Modality-specific policy refusal

The image's content falls into a policy-restricted category: medical interpretation, legal interpretation of a document, identity document verification, screen-share content containing passwords or payment details.

Template follows your policy, but structurally: *"I can see [generic acknowledgement] but I'm not able to [specific thing this policy forbids]. [Where the user can get help instead.]"*

These are the same categories you handle in text-only safety reviews (mod-108) — the vision path is not a bypass.

## The silent-failure mode to design against

The failure case that breaks user trust the hardest is **confident-and-wrong grounding**: the model emits a plausible bounding box on a plausible-sounding claim, the UI overlays a box on the user's image, and the box is on the wrong thing or the quoted text is a hallucination.

Three defences:

- **Always verify grounded quotes against OCR.** If the model says `"the price is $19.99"` and `"$19.99"` is not in the OCR of the frame, reject the claim.
- **Always render the crop next to the agent's claim in the UI**, so the user can see at a glance whether the model looked at the right thing. The human visual system is very fast at "that's not the error I meant".
- **Log every grounded claim** with its region, its OCR'd evidence, and the model's confidence. The eval harness (Chapter 6) replays these to measure grounding accuracy.

## Worked example: the "support screenshot" turn

User uploads a screenshot and asks "why is my login failing?"

A grounded response:

```json
{
  "spoken_response": "The error at the top of the screen says your session token is invalid. That usually means the app signed you out in the background — try signing in again from the start.",
  "grounding": [
    {
      "claim": "the error at the top of the screen is 'invalid session token'",
      "region": {"type": "bbox_normalized", "box": [0.78, 0.04, 0.98, 0.10]},
      "evidence": {"type": "text", "text": "invalid session token", "source": "ocr+model"}
    },
    {
      "claim": "'invalid session token' typically means the app signed the user out in the background",
      "region": null,
      "evidence": {"type": "model_knowledge"}
    }
  ]
}
```

The orchestrator:

1. Verifies the OCR of the bbox contains the quoted string → pass.
2. Renders the crop next to the spoken response in the UI.
3. Logs the grounding record for eval.

If the OCR check had failed, the orchestrator would downgrade the turn and switch to a "I'm not sure, here's what I can tell" refusal shape.

## Where this slots into the broader agent

Grounding is an **orchestration-layer responsibility**, not a model responsibility. The model can emit bounding boxes and quoted text; it cannot verify them against its own input. Your orchestrator is the only place that owns the truth of "what did the user actually send, and did the model's claim match it."

This is also the layer that mod-112 (eval & observability) and mod-108 (safety & compliance) hook into. A grounded-claims log is both a safety artefact and an eval dataset.

## Summary

- A grounded multimodal claim has three parts: **claim**, **region**, **evidence**. Any missing part is a design bug.
- Prefer **DOM / accessibility-tree references** when available, **bounding boxes + cropped excerpts** otherwise, and always try to **cross-check OCR** for quoted text.
- Prompt explicitly for structured `grounding` output; separate `spoken_response` (for TTS) from `grounding` (for UI, audit, eval).
- Define **four refusal shapes** — "ask for a better image", "ask for the thing you need", "I'm not sure, here's what I can tell", and "policy refusal" — and route per situation.
- The worst failure mode is **confident-and-wrong grounding**; defend against it with OCR verification, UI crops next to claims, and a durable grounded-claims log.

Next chapter: the modality-routing heuristic — when to run vision at all, and the cost curve you are actually on.
