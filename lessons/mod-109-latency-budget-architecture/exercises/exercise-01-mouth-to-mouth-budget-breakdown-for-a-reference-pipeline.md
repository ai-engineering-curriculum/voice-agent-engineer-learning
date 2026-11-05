# exercise-01: Mouth-to-Mouth Budget Breakdown for a Reference Pipeline

**Estimated effort:** 3 hours · **Output:** an annotated budget table + a one-page architectural brief

## Objective

Produce a defensible, per-leg latency budget for a specific voice-agent pipeline you design. The deliverable is the artefact a Voice & Multimodal Agent Engineer is on the hook for at the "design phase" of a new voice feature — before the first line of code ships.

You are *not* hitting the budget yet. You are committing to it on paper.

## Prerequisites

- [Chapter 1 — The Mouth-to-Mouth Budget](../01-the-mouth-to-mouth-budget.md).
- A rough understanding of G.114's one-way-delay thresholds.
- A chosen target product: pick one of the three reference scenarios below (or propose your own and justify it).

## Reference scenarios

Pick one. The scenario decides the user population, transport, and constraints — every budget below flows from it.

1. **Consumer mobile voice app.** US-based users, WebRTC transport, same-region media server, cascaded STT→LLM→TTS, p95 turn-gap target 800 ms.
2. **Inbound customer-service phone bot.** PSTN callers routed via Twilio Media Streams to a WebSocket orchestrator, G.711 codec, cascaded STT→LLM→TTS, p95 turn-gap target 1000 ms (telephony conventions are more forgiving).
3. **EU clinical ambient scribe**, EU data residency, EU-region media server, EU-region LLM/TTS providers where available, p95 turn-gap target 900 ms (clinical users tolerate slightly longer pauses in exchange for accuracy).

## Steps

1. **Draw the pipeline.** Produce a boxes-and-arrows diagram of the end-to-end path from microphone to the user's ear. Every component from Chapter 1 should appear. Label which components run on the user's device, which on your infrastructure, and which on a third-party provider.

2. **Fill the budget table.** For the scenario you chose, fill in a p95 budget table with columns `leg`, `owner (device / your infra / provider)`, `p95 budget (ms)`, `source / justification`. Legs to cover, at minimum:
   - Mic capture + device encode
   - Network ingress
   - Media-server jitter buffer + decode
   - VAD endpoint detection
   - ASR partial (optional, as a floor on ASR streaming readiness)
   - ASR final
   - Orchestration (prompt build, any retrieval, tool routing)
   - LLM TTFT
   - TTS TTFB
   - Network egress
   - Client decode + playout

3. **Justify each number.** For every row, cite a source: a provider SLO, a published benchmark (dated), your own prior measurement, or a reasoned estimate explicitly marked as such. "Feels right" is not a source. Numbers you can't source should be tagged with a specific research action to take before committing the budget.

4. **Sum the budget.** Compute the naive-sequential sum and the overlap-aware sum (assume TTS starts on first LLM token, so LLM TPOT doesn't appear on the critical path). If the overlap-aware sum exceeds the product target, you must either (a) renegotiate the target, (b) tighten a specific leg and explain how, or (c) propose an architectural change (region move, provider swap, model swap). Document which you chose and why.

5. **Call out your top three risks.** For each, write one paragraph: what could cause the leg to exceed its p95, how you would detect it, and what the fallback is.

6. **Write the architectural brief.** One page of prose, the deliverable you would hand to an engineering manager or a product lead to approve the design before implementation starts. Cover: the scenario, the target, the budget table (inline or linked), the top risks, and the top three decisions you are asking the reviewer to ratify.

## Starter guidance

- **Reach for provider documentation, not blog posts.** If a provider doesn't publish an SLO, note that as a risk.
- **Use p95, not p50.** A budget built from p50s will fail in production.
- **Keep the budget small enough that it isn't the whole document.** The reader wants to know your choices, not to admire your table.
- **The "orchestration" leg is small but easy to blow.** Prompt assembly, retrieval, policy checks, tool routing can all accumulate; budget for 10–40 ms and aggressively profile what you spend.
- **If a leg is owned by the user's device or by the user's carrier**, you still have to budget for it. You can't fix it, but you can plan for it.

## Acceptance criteria

- [ ] One of the three reference scenarios is chosen (or a custom scenario is specified with a one-paragraph justification).
- [ ] A pipeline diagram covering device / your infra / provider boundaries.
- [ ] A per-leg p95 budget table with legs for mic, ingress, jitter, VAD, ASR partial + final, orchestration, LLM TTFT, TTS TTFB, egress, playout.
- [ ] Every row has a sourced justification or an explicitly tagged research action.
- [ ] Both naive-sequential and overlap-aware sums are computed and compared to the target.
- [ ] Top three risks are documented with detection and fallback plans.
- [ ] One-page architectural brief suitable to hand to a reviewer.

## Stretch goals

- **Second scenario in parallel.** Do the same work for a *different* reference scenario and compare the budgets side-by-side. The exercise is to see which legs change materially and why.
- **Sensitivity analysis.** If every provider SLO you cited drifted by +20% at the p95, which legs would breach the budget first? Which fallbacks (Chapter 5) would you activate?
- **Speech-to-speech model alternative.** Re-do the overlap-aware budget assuming you replaced the cascaded STT→LLM→TTS with a native speech-to-speech model (OpenAI Realtime, Gemini Live, Moshi). Which legs merge or disappear, and which new unknowns appear?

## Keep it

The budget table and brief are the template you will reuse in the project-104 architectural-brief capstone. Save them in a form you can hand to a reviewer without reformatting.
