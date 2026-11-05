# exercise-04: Multimodal Grounding with Citation (Bounding Boxes or OCR)

**Estimated effort:** 2 hours · **Output:** a grounded-vision response layer + a grounded-claims audit log

## Objective

Add a **grounded response contract** to the vision client from exercise-02. Every vision-based claim the model emits must come with a bounding-box region and verifiable evidence (OCR text or a cross-check), stored in a structured audit log. You will also implement at least one of the four refusal shapes from Chapter 4 and demonstrate that it fires on an appropriately ambiguous image.

## Prerequisites

- [Chapter 4 — Multimodal Grounding and Refusal Shapes](../04-multimodal-grounding-and-refusal.md).
- The `VisionClient` from exercise-02 (or a provider of your choice with image input).
- An OCR path — Tesseract, [Amazon Textract](https://aws.amazon.com/textract/), [Google Document AI](https://cloud.google.com/document-ai), [Azure Document Intelligence](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/), or the provider's own text-extraction output.

## Steps

1. **Update the system prompt to require grounded output.** Instruct the model to return JSON with `spoken_response` (plain prose for TTS) and `grounding` (array of `{claim, region, evidence}` records). Use the shape from Chapter 4. Pick one bounding-box convention — e.g., normalised `[x0, y0, x1, y1]` in `[0, 1]` — and document it.

2. **Parse and validate the model output.** Reject responses that are not valid JSON, do not contain `spoken_response`, or do not contain a `grounding` array. On parse failure, down-grade to a refusal response rather than passing garbage to TTS.

3. **Add an OCR cross-check.** For each grounding record whose evidence is a quoted text string, OCR the image region specified by the bounding box (plus a small margin). Compute a fuzzy match between the model's quoted text and the OCR output (e.g., Levenshtein ratio ≥ 0.8). Mark the record as `verified: true` if the match passes, `verified: false` otherwise.

4. **Build the grounded-claims audit log.** For every vision turn, log:
   - `turn_id`
   - `image_hash` (perceptual or SHA-256)
   - `question`
   - `model`
   - `spoken_response`
   - Each grounding record, including `region`, `evidence`, and `verified` flag.
   - A `trust_score` aggregating the verified flags.
   Store this as structured records (JSONL, a SQLite table, or your preferred). The log must not retain raw image bytes by default (store hashes); allow opt-in retention for specific debugging turns.

5. **Implement one refusal shape.** Choose **"ask for a better image"** or **"I'm not sure, here's what I can tell"**. Add the policy: when the model's `grounding` array is empty, when every grounding record fails verification, or when the model emits an explicit low-confidence signal, substitute the real response with a refusal built from a template. The refusal must name the specific issue, not just say "I can't read that".

6. **Demonstrate end-to-end on three test cases.**
   - **A clear screenshot with an identifiable error message.** Expected: grounded answer, OCR verification passes.
   - **A blurry photo of a product label.** Expected: refusal with specific ask.
   - **A chart that the model summarises in prose.** Expected: grounded claim with a non-text region (bounding box covering the chart) and evidence recorded as `"model_description"` (since no OCR applies). Decide whether this counts as verified; justify.

7. **Render the grounding for a human reviewer.** Write a tiny HTML page that, for each audited turn, shows the original image with the bounding boxes overlaid, the spoken response, and the per-record verification status. This is the artefact a human reviewer would inspect during eval (exercise-06).

8. **Write a short writeup.** One page. Cover:
   - How often the OCR cross-check caught a hallucinated quote (across your three test cases plus any extras you ran).
   - The specific refusal template you chose and the issues you named.
   - Where the grounding model's bounding boxes were visibly wrong, and what you would do about it.
   - One scenario where your current verification would miss a hallucination (think adversarially).

## Starter guidance

- **Models vary in bbox quality.** Current frontier vision models produce usable bounding boxes, but the quality is noticeably below their text accuracy. Expect to see boxes that are approximately in the right place but off by 10–20% of the image dimension. Your reviewer UI makes this visible instead of invisible.
- **Do not require structured output in free-form user conversations.** If you force every response into JSON, the model's prose quality drops. Scope grounded-JSON mode to the vision turn specifically, or use the provider's structured-output / JSON-mode feature where available.
- **Fuzzy matching, not exact.** OCR errors and model rendering differences both introduce character-level noise. A character-level ratio is more useful than exact match. Tune the threshold on your test set.
- **Non-text evidence is still evidence.** If the claim is about a diagram, the "evidence" can be the cropped image region itself — logged, reviewable, not OCR-verifiable but still auditable.
- **The audit log is where this exercise pays off.** The prompt and schema are trivial; the log is the production artefact.

## Acceptance criteria

- [ ] System prompt requires `{spoken_response, grounding[]}` output.
- [ ] Parser rejects malformed output and routes to a refusal.
- [ ] OCR cross-check on every text-quoted grounding record with a fuzzy-match threshold.
- [ ] Structured audit log of grounded claims (no raw image bytes by default).
- [ ] At least one refusal shape implemented with a specific issue named in the response.
- [ ] Three demonstration cases run end-to-end (clear screenshot, blurry label, chart).
- [ ] An HTML reviewer page overlaying bounding boxes on the original image.
- [ ] A one-page writeup including an adversarial scenario the current verification would miss.

## Stretch goals

- **Second refusal shape.** Add "ask for the thing you need" for turns where the user's question requires vision but no image is attached or the latest image is stale.
- **Self-consistency pass.** On consequential turns, re-ask a second cheaper model with the same image and claim; compare answers; downgrade on disagreement.
- **DOM-based grounding.** If your exercise-03 prototype had screen share, swap bbox grounding for DOM / accessibility-tree grounding on that path. Compare precision against bbox grounding on the same questions.
- **Quote-contamination benchmark.** Build a tiny benchmark of (image, question) pairs where the correct answer is present in the image and nowhere else, and where a hallucinated plausible alternative is tempting. Measure how often each provider hallucinates the alternative.

## Keep it

The audit log is the input to the eval harness in exercise-06. Keep the schema stable; the sooner you have real logged claims, the sooner you can iterate on routing (exercise-05) against measured grounding accuracy rather than vibes.
