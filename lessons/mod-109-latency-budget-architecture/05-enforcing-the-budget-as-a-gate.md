# 05. Enforcing the Budget as an Engineering Gate

## Why this chapter

A budget that lives in a design document is marketing. A budget that lives in CI is engineering. The difference — and the only difference — is that one of them fails the build.

Chapters 1–4 define the budget, measure it, and architect toward it. This chapter is about keeping it. The three pieces that make a budget operational:

1. **Synthetic callers** that drive representative traffic on every build and in continuous staging.
2. **A budget gate** in CI — a p95 turn-gap check that blocks a merge when the number regresses.
3. **Kill-switches and fallbacks** at runtime — when the budget breaches in production, the pipeline degrades gracefully rather than letting the user suffer.

Everything in this chapter lives at the boundary with the eval-and-observability work in mod-112, which is where the broader eval harness lives. Here you own the latency-budget half of it.

## Synthetic callers

You cannot gate on p95 turn gap from live traffic alone — live traffic is a function of users, time of day, and chance. You need a reproducible, bounded, representative source of calls.

A synthetic caller is a program that:

1. **Connects as a client would** — WebRTC, SIP, or your platform's native transport.
2. **Plays audio of a scripted utterance** — real recorded speech, not TTS, to avoid testing your TTS against your TTS.
3. **Measures the response** — captures the agent's audio, timestamps first byte, and emits the per-leg metrics from Chapter 2.
4. **Scores** against the budget and against any output-quality checks the eval harness provides.

A few design notes:

- **Use recorded human audio.** A library of ~20–200 utterances covering your product's canonical turns (greetings, confirmations, interruptions, numbers, named entities, barge-in). The reason to avoid TTS-playing-into-ASR is that it over-reports ASR quality and under-reports latency variance on real microphones.
- **Include noise and codec degradation.** Mix in background noise, apply a telephony codec (G.711 or Opus at low bitrate) to a subset, and simulate packet loss on another subset. These are not edge cases — they are the modal call.
- **Run both single-turn and multi-turn flows.** A one-shot "hi, what's the weather" doesn't exercise session state, prompt caching, or tool calls. Your scenario set should cover both.
- **Use existing frameworks when they fit.** LiveKit's synthetic agent harness and Pipecat's test utilities are both suited to this, and some orchestration platforms ship a loadrunner specifically for driving voice sessions. Build your own only when a specific scenario isn't supported.

## The CI gate

Once you can drive synthetic calls, the gate itself is a short list of checks:

- **Correctness gate** — the agent's text output and spoken output pass the eval checks from mod-112 (grounded in the knowledge base, policy-compliant, etc.). This is not latency-specific but is upstream of it.
- **Budget gate** — the p95 turn gap across the synthetic run is under the committed target (default 800 ms), and the p99 is under a looser second threshold (default 1.2 s).
- **Per-leg gate** — no single leg's p95 is more than X% over the per-leg budget from Chapter 1. This catches "overall budget is fine because one leg got faster and one got slower", which usually means a provider changed something.

What to do when the gate fails:

- **Fail the build.** The build is the enforcement point. There is no stronger signal.
- **Attach a per-leg breakdown to the failure.** A build failure that says "p95 turn gap regressed from 720 ms to 890 ms" is useless; "p95 ASR final regressed from 240 ms to 410 ms, no change in any other leg" tells the engineer where to look.
- **Fail softly for non-blocking metrics.** p99 regressions and per-leg regressions on non-critical legs can emit a warning without failing. Do not add ambiguity to the blocking metric.

### Where the gate runs

CI runs are cheap, short-lived, and predictable. Production is none of those. The pragmatic pattern:

- **In CI** — a short synthetic run (1–5 minutes, 10–50 scenarios) against a representative environment (ideally, a staging pod backed by the same provider endpoints as production).
- **In continuous staging** — a longer-running synthetic fleet that drives traffic 24/7 and emits the same metrics to the production observability stack. Regressions there are a leading indicator for live-traffic regressions.
- **In production** — the same metric is derived from real user traffic, with the same dashboards and alerts. Production is where you catch the things CI's synthetic scenarios missed.

The three environments measure the same metric three ways. That is intentional. If any of the three disagrees with the others, the discrepancy itself is a signal.

## Runtime enforcement: kill-switches and fallbacks

Even with CI gates, production will blow the budget. Providers have bad days. A deployment goes to the wrong region. Prompt-cache contents evict on a long call. The runtime enforcement is about *failing gracefully*, not about preventing the failure.

Three patterns that work:

### Pattern 1: Faster TTS voice

If TTS TTFB is the breach, fall back to a lower-quality but faster TTS voice. This is the cheapest pattern: a different voice config in the same provider, or a provider-switch to a lower-latency alternative. The user notices quality; they do not notice that the pipeline rerouted.

Implementation sketch:

```python
def choose_tts():
    if recent_p95("tts.ttfb") > ttfb_budget * 1.5:
        return TTS_FAST_VOICE
    return TTS_DEFAULT_VOICE
```

Decisions based on a rolling window of recent observations, not on the single most recent turn.

### Pattern 2: Smaller / faster LLM

If LLM TTFT is the breach, fall back to a smaller or lower-latency model. The quality cost is real — users may notice weaker responses — but a measurable degradation in quality is strictly better than a dead-air outage.

Tier the fallback. For example:

- **Tier 1**: your production model (Claude Sonnet, GPT-4o, Gemini 2.5, whatever you shipped with).
- **Tier 2**: a faster model with the same quality bar on the common cases (a Haiku-class, or a smaller GPT, or a smaller Gemini).
- **Tier 3**: a scripted fallback ("I'm having trouble, let me take a note and get back to you") that doesn't call an LLM at all.

Each tier has to be covered in CI so the fallback isn't itself broken.

### Pattern 3: Shed load

If the breach is caused by volume — a spike in concurrent sessions — the correct response is not to degrade every call. It's to **reject new calls** (telephony: busy signal; WebRTC: connection declined) while keeping in-flight calls at full quality. Users prefer "call back in a minute" to "all calls got bad".

This is politically harder because product and ops teams don't like rejecting calls. Make the call rejection *observable* and *rare* and tie it to a specific capacity signal (provider rate limit, media-server CPU, concurrent-session ceiling). Do not use it as a stealth cost-control mechanism.

### A note on what *not* to do

- **Do not silently truncate LLM output** when TPOT falls behind. The user will hear a cut-off sentence.
- **Do not cache a previous turn's audio** as a fallback. It is almost always wrong and very, very obvious.
- **Do not swap TTS mid-utterance.** The voice change mid-sentence is worse than any plausible latency gain.

The fallback principle: degrade to the *next complete pipeline*, not to a corrupted version of the current one.

## The runtime budget dashboard

Three charts, pinned to oncall's wall:

- **Turn gap p50 / p95 / p99** over time, with the committed target overlaid as a horizontal line.
- **Per-leg stacked p95** — every leg's p95 contribution to the turn gap, stacked, so a breach is immediately attributable.
- **Fallback-activation count** — how often each tier of fallback fired in the last N minutes. Any sustained non-zero rate is a pager-grade signal.

If an engineer can't tell within 60 seconds which leg is causing a turn-gap alert, the instrumentation (Chapter 2) is not complete.

## Keeping the budget alive

The budget changes. New models ship; new regions open; the product adds tool calls; customer traffic shifts geography. Do not treat the numbers in Chapter 1 as permanent.

A quarterly ritual:

1. Re-measure the production budget per-leg.
2. Compare against the committed per-leg budget.
3. Negotiate updates where provider improvements have made a leg cheaper (take the budget back) or where product scope has made a leg more expensive (push some back into other legs or renegotiate the target).
4. Push the updated budget into the CI gate.
5. Document the change in the architectural brief for the team's next decision.

If the budget drifts without any re-measurement, it has silently become a lie. The ritual is cheap insurance.

## Summary

- **Synthetic callers** drive reproducible traffic, using recorded human audio with noise/codec/loss injection.
- **The CI gate** blocks regressions on **p95 turn gap** and **per-leg** metrics; a failure attaches a per-leg breakdown.
- Three environments measure the same metric: CI, continuous staging, production. Disagreements between them are themselves signals.
- **Runtime fallbacks**: faster TTS, smaller LLM, load shedding. Degrade to the next *complete pipeline*, not to a corrupted version of the current one.
- Re-measure the budget quarterly and update the gate; a budget that doesn't drift is one that nobody is watching.

Next chapter: the sharp boundary to the AI Inference Engineer — what lives inside the model server vs. in your architectural budget.
