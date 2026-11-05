# exercise-05: Modality Routing Heuristic with Cost Attribution

**Estimated effort:** 2 hours · **Output:** a modality router + a per-session cost dashboard

## Objective

Implement a **modality router** that decides, per turn, whether to call the vision-enabled path or a text-only path. Instrument it so you can attribute per-session cost to the routing decision and the caching outcome. Then sweep the router's thresholds against a replay of real or synthetic traffic and show the cost / miss-rate trade-off.

## Prerequisites

- [Chapter 5 — The Modality-Routing Heuristic and Cost Curve](../05-modality-routing-and-cost-curve.md).
- The `VisionClient` from exercise-02 (or equivalent provider wiring).
- A dataset of turns to replay. A reasonable minimum is 200 turns with a mix of "clearly needs vision" / "clearly does not" / "borderline" annotations. You can synthesise this from real product logs (with consent) or by scripting plausible user utterances; just document provenance.

## Steps

1. **Define the router interface.** A function `choose_modality(turn_ctx) -> "text_only" | "vision+text"` taking a context object with at least:
   - `asr_final` (string transcript)
   - `asr_confidence` (float)
   - `attached_images` (list of image references with timestamps)
   - `last_tool_calls` (list with timestamps and whether any returned an image)
   - `user_state` (optional)

2. **Implement the signals as a cascade** following Chapter 5:
   - Deictic phrase match against a configurable list. Start simple ("look at", "on the screen", "this error", "in the photo", "this picture", "that chart", "see this", "right there", "here"); add regional / domain terms as needed.
   - Tool-returned image within the last N seconds (configurable).
   - Low ASR confidence with a recent image within M seconds (two configurable thresholds).
   - User state in a `VISION_HEAVY_STATES` set.
   Keep the cascade order explicit so you can reason about which signal fired.

3. **Add a "retry with vision on refusal" safety net (optional).** If the router chose `text_only` and the model's response matches a configurable low-confidence / refusal pattern, and an image is available, retry on the vision path. Log that this retry fired.

4. **Instrument per-turn cost accounting.** For every LLM call, record:
   - `modality.chosen` (text vs. vision+text), including whether it was the initial choice or a retry.
   - `vision.images_attached` count and total image size.
   - `vision.detail_level`.
   - `vision.cache_hit` (true / false / mixed).
   - Provider-reported token counts and image tokens.
   - Billed cost estimate (compute from provider rates; date your rate sheet).
   - Routing signal that fired (or `none` for the default text path).

5. **Replay the dataset through the router.** For each of your 200+ turns, run the pipeline end-to-end and record the cost-accounting fields.

6. **Compute the trade-off curves.**
   - Plot **cost per session vs. vision-rate** `p` across different router configurations: strict (deictic only), medium (deictic + tool-returned), permissive (all signals), always-on vision, always text-only.
   - For each configuration, compute the **miss rate** — the fraction of turns labelled "clearly needs vision" where the router chose `text_only` without a successful retry.
   - For each configuration, compute the **over-shoot rate** — the fraction of turns labelled "clearly does not need vision" where the router chose `vision+text`.

7. **Pick a default.** On your trade-off curve, choose a configuration and justify it. Prefer a configuration that keeps miss-rate below a documented target (e.g., < 5%) while minimising vision-rate `p`.

8. **Dashboard it.** Build a small dashboard — SQL + a one-page visualisation, a spreadsheet, or a Grafana panel — that answers:
   - Per-session cost split by modality and by cache outcome.
   - Per-signal breakdown of what triggered vision upgrades.
   - Cache-hit rate for vision turns.
   - Trend over time (even if your replay is one batch, future production traffic will populate this).

9. **Write a short writeup.** 1–2 pages. Cover:
   - The dataset's provenance and labelling methodology.
   - The trade-off curve.
   - The default configuration you chose and the specific miss-rate / over-shoot numbers it achieves.
   - The two or three signals that drove most of the routing decisions, and the ones that were essentially dead weight.
   - What you would measure next if you had another week.

## Starter guidance

- **Deictic phrases are not universal.** If your product is non-English or domain-specific, the Chapter 5 phrase list is a starting point, not a solution. Add terms from your actual traffic.
- **Beware cost estimates from provider marketing numbers.** Use published per-token rates, not rounded "typical" numbers. The ratio between providers is more stable than the absolute cost, so compare ratios when the absolute numbers are noisy.
- **Attribution is the point.** Even a mediocre router with excellent attribution is more useful than a sophisticated router whose decisions you cannot explain.
- **Cache-hit rate for vision turns will drive your actual cost** more than router thresholds, once routing is tuned. Make this metric prominent on the dashboard.
- **Label 200 turns yourself if you have to.** The dataset is the bottleneck on getting a real trade-off curve; a borrowed benchmark will not reflect your product's routing needs.

## Acceptance criteria

- [ ] A `choose_modality(turn_ctx)` router implementing the deictic / tool / confidence / state cascade.
- [ ] Per-turn cost accounting: modality chosen, images, detail, cache hit, tokens, estimated cost, signal that fired.
- [ ] Replay through a dataset of ≥200 turns with routing + cost logged per turn.
- [ ] Trade-off curves for at least five router configurations (strict, medium, permissive, always-on, always-off) with per-session cost and miss / over-shoot rates.
- [ ] A chosen default configuration with numeric justification.
- [ ] A dashboard answering per-session cost / signal breakdown / cache-hit questions.
- [ ] A 1–2-page writeup covering provenance, trade-off, chosen default, and next measurements.

## Stretch goals

- **ML router.** Train a small classifier (logistic regression or a tiny transformer) on your labelled dataset to predict whether a turn needs vision. Compare against the rule-based cascade for miss-rate and over-shoot. Only ship if it is clearly better.
- **Dynamic detail level.** Once the router has chosen vision, add a second decision — `detail: low` vs. `detail: high` — based on whether the user's question mentions small text or detail. Measure cost savings on low-detail turns.
- **Cost / latency joint curve.** Plot vision-rate vs. p95 turn gap (not just cost). Chapter 5 claims vision upgrades cost latency; verify or refute on your traffic.
- **Live-traffic shadow test.** Run the router in shadow mode in production (log the decision but keep always-on vision live). Compare the shadow's decisions against the live behaviour; identify misses the offline replay could not predict.

## Keep it

The router is a project-101 integration point. Keep it configurable at runtime (feature flags per signal) rather than hard-coded — the thresholds will change as your traffic mix shifts.
