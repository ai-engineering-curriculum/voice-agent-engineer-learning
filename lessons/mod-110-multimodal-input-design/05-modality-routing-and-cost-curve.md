# 05. The Modality-Routing Heuristic and Cost Curve

## Why this chapter

Multimodal turns are expensive. A text-only LLM call for a short question is a few cents at most; a vision-enabled turn on a high-resolution image can be 10–100× that depending on the provider, the detail setting, and the number of images. Over a multi-turn session with sticky visual context, the cost gap widens further.

The architectural response is **modality routing**: decide per turn whether this turn actually *needs* vision, and route accordingly. A well-tuned router runs most turns through the cheap text-only path and reaches for the vision path only when there is a signal that it will matter. This chapter is the design and the arithmetic of that router.

## Why "always-on vision" is almost never right

Three failure modes of the always-on vision architecture:

- **Cost blowout.** On a 20-minute screen-share session at 1 frame every 3 seconds with a vision model, you are paying 400 frames × per-frame vision cost per session. For a product with thousands of sessions a day, that is the top line of the cost sheet. The math is worse if your sticky-visual-context policy also re-sends the frame on every voice turn.
- **Latency hit.** A vision-enabled LLM turn typically has a higher TTFT than the equivalent text-only turn — the model has to process the image tokens before it can emit the first output token. For a mod-109-shaped turn-gap budget (p95 < 800 ms), unnecessary vision calls on text-answerable turns use latency headroom you don't have.
- **Attention dilution.** On turns where vision was irrelevant, the model's attention still gets consumed by the image. In practice this often produces answers that pivot to describing the image instead of answering the user's question. ("I see a screenshot of a login page. The user is asking…" — the model is answering a different question than the user asked.)

If you can keep ≥70% of turns on a text-only path with no quality loss, that is the dominant cost and latency lever for the whole product.

## The upgrade signals

The router lives in the orchestration layer and decides, for each incoming turn, whether to:

- **Answer text-only** — ignore attached images, do not attach screen-share frames, skip vision models entirely.
- **Answer with attached image(s)** — use the user's latest attached image or the last N frames from the live session.

A reasonable set of signals, roughly ordered by how reliable they are:

### 1. Explicit deictic references in the utterance

The user says "look at this", "on the screen", "in the image", "this error", "this page", "the chart", "right there". These are the strongest signal that the turn is about the visual context. A regex / lightweight classifier on the ASR final transcript is more than enough; a small LLM call is overkill.

### 2. Tool-call-driven artefact

A tool the agent called in a prior turn returned an image (a chart, a map, a rendered diagram). The next turn is likely to reference it; attach it.

### 3. Low ASR confidence with recent image / frame

If the ASR is uncertain about the user's utterance and there is a recent image in the session, the image may disambiguate — a voice agent answering a question about a product the user is holding up to the camera, for instance, can reason from the image even when the mic feed is bad.

This is model-specific: some vision-plus-audio models (Gemini Live) handle this natively if you feed both tracks. For cascaded pipelines, the orchestrator has to make the call.

### 4. Domain / state-driven heuristics

If the user is in a known state that always benefits from vision — mid-onboarding, inside a troubleshooting flow, on a page where user-generated content is the subject — attach by default.

### 5. "I don't know, I need to look"

The model's first attempt at a text-only answer is low-confidence or refuses; escalate to a vision-enabled retry. This is a safety net, not a primary mechanism — it doubles the LLM calls on turns that trigger it, so it is only cheaper than always-on vision if the trigger rate is low.

## Routing policy, as a cascade

A minimal production router:

```python
def choose_modality(turn_ctx):
    utterance = turn_ctx.asr_final
    if deictic_phrase(utterance):
        return "vision+text"
    if turn_ctx.last_tool_call_returned_image_within_s(30):
        return "vision+text"
    if turn_ctx.asr_confidence < 0.6 and turn_ctx.recent_image_within_s(60):
        return "vision+text"
    if turn_ctx.user_state in VISION_HEAVY_STATES:
        return "vision+text"
    return "text_only"
```

The order matters: cheap deterministic checks first, model-involving checks last. Build the routing policy from your actual traffic — look at what kinds of turns you see, measure what fraction match each signal, and tune thresholds against the eval harness in Chapter 6.

## The cost curve

The reason routing is worth doing is that the cost shape of a multimodal product is dominated by two variables:

- **p = fraction of turns that use vision**
- **k = per-vision-turn cost premium over a text-only turn**

Total per-session cost ≈ `turns_per_session × text_cost × (1 + p × (k − 1))`. Two illustrative limits:

- **Always-on vision** (p=1): cost ≈ `turns × text_cost × k`. If k ≈ 20 (reasonable for a high-detail frame), you are paying 20× the text baseline.
- **Well-routed** (p=0.15, k=20): cost ≈ `turns × text_cost × 3.85`. Still ~4× text baseline, but 5× cheaper than always-on.

The exact numbers are provider-dependent and change over time. The *shape* does not. Routing is a knob that bends a per-session cost line from "steep" to "manageable".

## Cost attribution: measuring, not guessing

A cost model is useless if you cannot check it against reality. Instrument the orchestrator to tag every LLM call with:

- `turn.id`
- `modality.chosen` (`text_only` vs. `vision+text`)
- `vision.images_attached`
- `vision.detail_level` (if provider supports it)
- `vision.cache_hit` (if using prompt / context caching)
- `provider.model`
- `usage.input_tokens`, `usage.output_tokens`, `usage.image_tokens` (if reported separately)

Roll these up in your observability stack (mod-112 covers the eval side; this is the cost side). The questions you want to answer on demand:

- "What's the cost per session this week, split by modality path?"
- "What fraction of vision-upgraded turns were triggered by each signal?"
- "What's the cache hit rate on image prompt caching?"
- "Which user states / cohorts are driving vision-turn spend?"

Without these, the router's settings are guesses and drift silently.

## Trading off the router against the eval

A permissive router (upgrade to vision eagerly) is safer in quality but expensive. A strict router (upgrade only on strong signals) is cheap but risks missing turns that *should* have been vision-enabled. The right balance depends on your product; Chapter 6 covers how to measure the balance. The dial you have:

- **Raise the bar for upgrade** (require explicit deictic phrases) → p goes down, cost goes down, miss-rate goes up.
- **Lower the bar for upgrade** (default on in vision-heavy states) → p goes up, cost goes up, miss-rate goes down.

In practice, teams iterate by watching:

- **Turn-gap p95 with vision on** — if vision upgrades push you over budget, tighten the router.
- **Human-eval "did the agent see what it should have" rate** — if this drops below your target, loosen the router.

## Caching is still your friend

Routing is one axis; image caching (mod-110 Chapter 2) is the other. A sticky-visual-context policy with prompt caching on the image can make a vision-upgraded turn *cheaper than a text-only turn without caching* on the tokenisation cost, because the vision tokens come from the cache hit. Always report cache-hit metrics alongside the routing metrics; they interact.

## A worked-example rollup

A hypothetical consumer support agent with 10 turns per session, text baseline $0.001/turn, vision premium k=20:

| Architecture | p (vision rate) | Cache hit rate | Cost/session |
|---|---|---|---|
| Always-on, no cache | 1.0 | 0% | $0.20 |
| Sticky context, no cache | 1.0 | 0% | $0.20 |
| Sticky context, with cache | 1.0 | 80% | ~$0.05 |
| Deictic routing, no cache | 0.15 | 0% | $0.038 |
| Deictic routing + cache for sticky | 0.15 | 80% | ~$0.013 |

Numbers are illustrative. The point is the **ratio**, not the absolute cost: the three levers (routing, caching, detail level) can collectively bring cost per session down 10–20× with no quality loss on routinely-answerable turns.

## Summary

- Default to **text-only**, upgrade to vision on specific signals: deictic references, tool-returned artefacts, low ASR confidence with recent image, vision-heavy user states, and low-confidence text-only answers as a safety net.
- The cost curve is dominated by `p × k` (vision rate × per-vision premium). Routing moves `p`; detail level and caching move `k`.
- **Instrument everything**: tag every LLM call with modality, image count, detail, cache hits, and tokens. Answer cost-per-session questions by SQL, not by guessing.
- Tune the router against the eval harness (Chapter 6). A loose router is safer in quality; a strict router is cheaper. There is no free knob.

Next chapter: evaluating multimodal turns — benchmark baselines, pairwise in-domain human eval, and measuring vision-grounding accuracy.
