# 02. Wiring Image and Document Input Across GPT-4o, Claude, and Gemini

## Why this chapter

Every major LLM provider exposes vision through a slightly different surface. GPT-4o and Claude accept images inline on a standard chat/messages request; Gemini has both a chat-shaped API and a bidirectional Live surface; OpenAI's Realtime API has its own rules about how an image gets pulled into an active voice session. The differences are not cosmetic — they change your transport, your prompt layout, and how you cache repeated images across turns.

This chapter picks the right entry point per provider, explains the image-caching trade-off, and gives you the small set of wiring decisions that make multimodal turns affordable and low-latency.

## Pick the entry point first

For a voice-first agent, you already have a transport for voice (WebRTC or WebSocket) and a separate request path for LLM calls. Where images attach depends on *where* the vision call happens. Four plausible entry points:

### 1. Vision through the regular LLM call

The user attaches an image; your orchestrator receives it over HTTPS (multipart or presigned URL); when it builds the next turn's LLM request, it inlines the image on a standard chat-completions-style endpoint. This is the common case.

- **GPT-4o** — [OpenAI Chat Completions API](https://platform.openai.com/docs/api-reference/chat) with an `image_url` content part per image, either as an `https://...` URL or an inline `data:` URL (base64).
- **Claude** — [Anthropic Messages API](https://docs.claude.com/en/api/messages) with an `image` content block per image, using either a URL source or a base64 source.
- **Gemini** — [Gemini API `generateContent`](https://ai.google.dev/api/generate-content) with an `inline_data` or `file_data` part per image.

This is the right choice when the voice loop is cascaded (ASR → LLM → TTS) and the LLM call is a separate request from the voice transport.

### 2. Vision through a Realtime / speech-to-speech surface tool call

If you are using the OpenAI Realtime API or similar for the voice loop, the LLM call isn't a separate HTTP request — it's embedded in the live session. To get an image into the model, the cleanest path is a **tool call that returns an image**. The agent calls a tool like `look_at_user_image(image_id="...")`, your server fulfils the tool call by fetching the image and returning it in the tool result content, and the realtime model consumes it in the next model turn.

OpenAI's Realtime API lets conversation items include image content parts, so the tool-result message carries the image inline. Check the current [Realtime API docs](https://platform.openai.com/docs/guides/realtime) for the exact content-part shape — this surface is evolving. <!-- needs-research: confirm whether current Realtime API accepts image_url parts directly on user messages or only via tool results -->

### 3. Vision through a native multimodal live surface

Gemini's **Multimodal Live API** accepts audio and video on the same bidirectional session; images or video frames arrive on the `video_in` stream and the model sees them without a separate HTTP round-trip. This is covered in detail in Chapter 3 for continuous streams; for a single image, the still-image API (option 1) is usually simpler.

### 4. Vision as a pre-processing step, text to the LLM

OCR the image to text, caption it with a cheap vision model, or run an extractor; pass the *text summary* to the main LLM. The main LLM never sees pixels. This is cheap, fast, and brittle — it works for structured docs and known templates and fails for anything that requires the model to actually reason about the image. Keep this option on the shelf for very-high-volume, narrow-domain use cases.

The right mix is almost always option 1 for user-attached images on a cascaded pipeline, option 2 for a realtime voice loop, option 3 for durable video context, and option 4 only where the image structure is known in advance and tightly bounded.

## Shapes of the request, side by side

The same conceptual payload — "here is a question and here is an image, answer it" — looks like this across the three providers. All three accept both URL and inline-base64 forms; the examples below show the common patterns. Always check current docs before copying anything into production.

### GPT-4o (OpenAI Chat Completions)

```python
response = openai.chat.completions.create(
    model="gpt-4o",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "What error is this screenshot showing?"},
                {"type": "image_url", "image_url": {"url": "https://cdn.example.com/screenshot.png"}},
            ],
        }
    ],
)
```

`image_url.detail` can be `"low"`, `"high"`, or `"auto"`. `"low"` uses a fixed, small number of image tokens regardless of input resolution; `"high"` uses more tokens for higher-resolution analysis. See OpenAI's [Vision guide](https://platform.openai.com/docs/guides/vision) for the current token-pricing formulas. The right choice depends on whether the task needs fine-grained detail (OCR of small text, inspecting a complex diagram) or just gross scene understanding.

### Claude (Anthropic Messages API)

```python
response = anthropic.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "What error is this screenshot showing?"},
                {
                    "type": "image",
                    "source": {
                        "type": "base64",
                        "media_type": "image/png",
                        "data": base64_encoded_png,
                    },
                },
            ],
        }
    ],
)
```

Claude accepts `base64` and `url` sources for images; `source.type` disambiguates. The [Claude vision guide](https://docs.claude.com/en/docs/build-with-claude/vision) documents the supported media types and image token math. Claude also supports PDFs via the [document content block](https://docs.claude.com/en/docs/build-with-claude/pdf-support), which is important for document-heavy workflows where you want the model to see both the rendered page and the extracted text.

### Gemini (`generateContent`)

```python
import google.generativeai as genai

model = genai.GenerativeModel("gemini-2.5-pro")
response = model.generate_content([
    "What error is this screenshot showing?",
    {"mime_type": "image/png", "data": png_bytes},
])
```

The Python SDK accepts a list of parts; strings become text parts, dicts with `mime_type` + `data` become inline binary. For larger or reused images, upload via the [Files API](https://ai.google.dev/gemini-api/docs/files) and pass a `file_data` reference instead. Gemini's image handling is covered in the [vision docs](https://ai.google.dev/gemini-api/docs/vision).

## Image caching: the cost / latency lever

A naive implementation re-uploads the same image on every turn that it's attached to. For a sticky-visual-context policy (Chapter 1), this is pathological: a 2 MB screenshot that persists for 20 turns is 40 MB of upload bandwidth and 20× the per-image token cost. Three mitigations, roughly in order of adoption:

### Provider-side image references

All three providers accept an image reference that is cheaper than inlining raw bytes:

- **OpenAI** — upload the image once via the Files API (or via a presigned URL) and reference it by URL / file ID on subsequent calls. The model still re-tokenises the image on each call, but your upload cost and request-size latency go away.
- **Anthropic** — Claude currently does not have a long-lived "attach once, refer later" file reference for images on standard Messages requests in the way OpenAI does. For repeat usage, see **prompt caching** below. <!-- needs-research: confirm whether Claude has shipped a dedicated image-file-reference API by the current date -->
- **Google** — [Gemini Files API](https://ai.google.dev/gemini-api/docs/files) stores files for ~48 hours and returns a URI usable in subsequent `generateContent` calls. Supports images, video, audio, and PDFs.

Use these for images that persist across multiple turns of the same session.

### Prompt caching including image blocks

LLM prompt caching (covered for text in mod-109 Chapter 3) extends to image blocks:

- **Anthropic prompt caching** marks blocks with `cache_control: {"type": "ephemeral"}`; image blocks are cacheable. The documented cache lifetime is ~5 minutes by default; see [prompt caching docs](https://docs.claude.com/en/docs/build-with-claude/prompt-caching). An image placed in the cached prefix and referenced by a short text question is a large TTFT and cost win. <!-- needs-research: confirm current Anthropic cache TTLs and image cacheability as of access date -->
- **OpenAI prompt caching** is automatic for sufficiently-long prefixes (see [OpenAI caching docs](https://platform.openai.com/docs/guides/prompt-caching)). Keep images early in the prompt so the prefix is cacheable.
- **Google Gemini context caching** is explicit: you create a [cached content object](https://ai.google.dev/gemini-api/docs/caching) with the image(s) and prefix, then reference the cache on subsequent calls. Minimum cache sizes apply.

For a sticky-visual-context policy, image prompt caching is the single highest-impact budget move. The vision-processing cost of a cached image is a fraction of a cold one, and the TTFT shift is in the hundreds of milliseconds.

### Downscale and pre-process before the LLM

Not every image needs full resolution. If your use case does not require reading small text or examining fine detail:

- Resize to the provider's recommended "low detail" size (OpenAI documents the pixel threshold that triggers `low` vs. `high` modes; Gemini scales large images server-side; Claude has its own recommendations).
- Strip EXIF and unnecessary metadata.
- Convert to a compressed format (WebP or JPEG at modest quality) unless the task requires lossless.

The savings are in both bandwidth and image tokens. The risk is that a resolution that "works most of the time" fails on the one turn where OCR of a small error code mattered — hence Chapter 6's eval pass.

## Response shape considerations for voice

Vision answers come back as text, but the text has to be spoken. Two consequences for your prompt design:

- **Markdown is not speakable.** If the vision model outputs `**bold**` or ``` ``` ``` code blocks ``` ``` ``` to format its answer, your TTS will narrate the asterisks. Either instruct the model to answer in plain prose, or strip markup in the speakable-token filter (mod-109 Chapter 3).
- **Bounding boxes and coordinates are not speakable either.** If the model emits "the error is at (124, 302) on the screen", the user cannot act on that. Chapter 4 covers how to represent spatial grounding in both a *speakable* form and a *UI-renderable* form at the same time.

Give the model a system instruction that makes voice-first output the default and tool-flavoured grounding information a side-channel.

## A decision table

For the common vision tasks you will wire, this is a reasonable default:

| Task | Entry point | Caching strategy |
|---|---|---|
| User sends one screenshot, asks a question | Chat/Messages inline, low or high detail based on task | None (one-shot) |
| User attaches an image, agent discusses it over 5+ turns | Chat/Messages, provider-side file reference | Prompt caching of the prefix including the image |
| Agent calls a tool that returns an image (e.g., "render this chart") | Tool-return content part | Provider cache if the artefact is referenced across turns |
| Realtime voice session, user shows one image mid-call | Realtime tool call returning image content | Not cached (one-shot in the live session) |
| User uploads a multi-page PDF | Native document block (Claude) / Files API (Gemini) / page-by-page images (OpenAI) | Prompt / context cache of the prefix |
| Continuous video or screen share | Not this chapter — see Chapter 3 (Gemini Live) | Not this chapter |

## Summary

- The right **entry point** depends on your voice transport: standard chat APIs for cascaded pipelines, Realtime-API tool-call content for live sessions, Gemini Live for continuous streams.
- All three providers accept inline base64 and remote URLs; **Gemini Files API** and **OpenAI file references** make re-use across turns cheap; **Claude currently relies on prompt caching** for the same win.
- **Image caching** via prompt caching (Anthropic, OpenAI) or context caching (Gemini) is the biggest single lever on multimodal cost and TTFT for sticky-visual-context policies.
- Plan for the **voice response shape** at prompt-design time — plain prose for TTS, grounding details as structured side-channel data.

Next chapter: wiring continuous video and screen share via Gemini Multimodal Live — the session architecture, frame-rate throttling, and the privacy envelope.
