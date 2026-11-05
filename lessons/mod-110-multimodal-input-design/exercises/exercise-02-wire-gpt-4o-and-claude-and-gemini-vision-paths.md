# exercise-02: Wire GPT-4o, Claude, and Gemini Vision Paths

**Estimated effort:** 3 hours · **Output:** a working multi-provider vision client + a latency/cost comparison report

## Objective

Build a small, dependency-light client library that wires image-and-text input into **GPT-4o**, **Claude**, and **Gemini** on a single interface. For a fixed set of test images and questions, measure per-call latency and reported token usage across the three providers. By the end you should be able to call `client.ask("<question>", image=<bytes>, provider=<name>)` and get a comparable result across all three.

## Prerequisites

- [Chapter 2 — Wiring Image and Document Input Across GPT-4o, Claude, and Gemini](../02-wiring-image-and-document-input.md).
- API keys for OpenAI, Anthropic, and Google AI / Vertex. Use test tier where possible to keep spend bounded.
- Python 3.11+ (or a language of your choice; the acceptance criteria are language-agnostic).

## Steps

1. **Define a provider-neutral interface.** Design a small `VisionClient` class with a method like `ask(question: str, image: bytes | str, media_type: str, detail: str | None, cache: bool) -> VisionResponse`. The response should include the text, the raw provider payload (for auditing), and token / cost accounting fields.

2. **Implement the GPT-4o path.** Use the Chat Completions API with an `image_url` content part. Support both inline base64 (`data:` URL) and HTTPS URL inputs. Expose the `detail` parameter (`low`, `high`, `auto`).

3. **Implement the Claude path.** Use the Messages API with an `image` content block. Support both `base64` and `url` sources. Ensure the `media_type` field matches the input content.

4. **Implement the Gemini path.** Use `generateContent` with an inline `inline_data` part for small images. Separately, implement a path that uploads the image via the Files API first and references it by URI — the test harness should exercise both paths.

5. **Build an image-caching layer.** When `cache=True`:
   - For OpenAI, verify (in your writeup) whether the shared prefix will be cached by the automatic prefix-cache mechanism; keep the image early in the prompt and the question late.
   - For Claude, set `cache_control: {"type": "ephemeral"}` on the message blocks containing the image + system prompt. Read the current Anthropic prompt-caching docs to confirm the semantics at the time you build this.
   - For Gemini, use the explicit [cached content](https://ai.google.dev/gemini-api/docs/caching) API; create a cache once and reference it on subsequent calls.
   Record cache hit / miss in the `VisionResponse`.

6. **Assemble a fixed test set.** 10–20 (image, question) pairs covering:
   - A screenshot with small text (tests OCR quality).
   - A diagram or chart (tests spatial reasoning).
   - A photo of a real-world object (tests scene understanding).
   - A document page (tests layout).
   - At least one deliberately ambiguous image (tests refusal shape).
   You may use public images; cite the source. Do **not** use any image containing real PII.

7. **Run the test set across all three providers.** For each call, record:
   - Wall-clock latency (TTFB and total).
   - Provider-reported input / output / image tokens.
   - Text response.
   - Whether the response contained markdown or structural artefacts a TTS pipeline would narrate literally.
   - Whether the response grounded the claim (even without explicit grounding instructions yet).

8. **Repeat the test set with caching enabled.** For each provider, call the same (image, question) 5 times in a row with the same image. Measure TTFB and cost on the first call vs. subsequent calls.

9. **Write a short comparison report.** Two or three pages. Cover:
   - A side-by-side table of p50/p95 latency per provider per image class.
   - Cost per query with and without cache for each provider.
   - Qualitative observations: which provider handled OCR best, which produced speakable text by default, which emitted markdown, where any provider refused appropriately or inappropriately.
   - A recommendation for which provider you would default to for your scenario from exercise-01, and why.

## Starter guidance

- **Keep the interface boring.** The point is to compare *providers*, not to design a vision framework.
- **Retries and transient errors.** Each provider has its own rate-limit and 5xx shapes. Add minimal retry with exponential backoff; make it the same across providers so you compare apples to apples.
- **Dates on everything.** Provider pricing and cache semantics move. Date your measurements and the provider docs you consulted; the comparison will be meaningless a quarter later without dates.
- **Image sizing.** Resize a copy of each test image to a realistic voice-agent upload size (e.g., long side 1024 px, JPEG 80%). Running the test at native DSLR resolution is both unrealistic and expensive.
- **Separate base rates from cache deltas.** On a 5-call cached run, the first call is the uncached baseline; calls 2–5 are the cache behaviour. Report both.

## Acceptance criteria

- [ ] A `VisionClient` with provider-neutral `ask(...)` and three provider implementations (GPT-4o, Claude, Gemini).
- [ ] Both inline-base64 and URL / file-reference inputs supported.
- [ ] Image caching integration: Anthropic `cache_control`, Gemini cached content, and OpenAI prefix-friendly layout. Cache hit / miss recorded per call.
- [ ] A fixed test set of ≥10 (image, question) pairs covering OCR, diagram, photo, document, ambiguous-image cases.
- [ ] Measurements across all three providers for the test set: latency, tokens, response text, markdown presence.
- [ ] Repeated-call measurement with caching enabled; first-call vs. subsequent-call latency and cost recorded.
- [ ] A 2–3-page comparison report with a provider recommendation for your exercise-01 scenario.

## Stretch goals

- **Detail-level sweep on OpenAI.** Measure the same queries at `detail: low`, `detail: high`, and `detail: auto`. Report quality vs. cost. Pick a default.
- **Multi-page PDF path.** Compare Claude's native document block, Gemini's Files-API PDF support, and the "render-each-page-as-an-image" approach for OpenAI. Which is best on your test documents?
- **Provider-neutral cache key.** Compute a stable cache key for `(image, prompt_prefix)` and verify your client gets cache hits across all three providers from the same key.
- **Fault injection.** Simulate a provider outage (return a 503 for every GPT-4o call). Ensure your client falls back to another provider without losing the request.

## Keep it

This client is the building block for exercise-05 (modality routing) and for project-101. Make it configurable (model names, regions, detail defaults) rather than hard-coding — the config will move as providers release new models.
