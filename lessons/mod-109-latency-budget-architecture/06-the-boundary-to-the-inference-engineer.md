# 06. The Boundary to the AI Inference Engineer

## Why this chapter

Sooner or later, the budget gets stuck on a leg that is not yours to fix. The LLM TTFT is bimodal because the provider is running a batch scheduler that occasionally parks small requests behind larger ones. The TTS TTFB has a 300 ms floor because the model server's streaming batcher is tuned for throughput, not latency. The voice-cloning TTS you shipped runs out of GPU under load and new sessions wait.

All of those are serving-layer problems. The `ai-infra-inference-engineer` (L35, peer role) owns them. You own the budget around them.

This chapter is the sharp boundary: what you escalate, what you keep, and how to package a request to the inference team so they can act on it.

## The handoff, in one paragraph

> You own **mouth-to-mouth latency budget** across the whole voice loop. You do not own **GPU scheduling, request batching, speculative decoding, model-server tail latency, or hardware allocation**. Your job is to measure the serving layer's contribution to your budget, prove it, and request specific fixes. Their job is to deliver those fixes under their own SLOs. The rest of this chapter expands the handoff.

## What lives inside the model server

The ai-infra-inference-engineer's domain, in the subset that matters for voice:

### Streaming batching for audio

Modern ASR and TTS servers batch requests for GPU efficiency. Batching trades per-request tail latency for throughput: a request can wait a few milliseconds for a batch slot to fill, especially under load. For voice workloads, this can appear in your budget as **TTS TTFB variance**: your p50 is clean but p95 has a long tail.

What the inference engineer can do:

- Tune the batcher for **latency-bounded batching** (fill up to N ms, not up to M requests).
- Deploy a **separate, latency-optimised server tier** for interactive voice traffic, isolated from bulk batch work.
- Switch to **continuous / inflight batching** for LLM serving, where new requests join an ongoing batch without blocking (vLLM, TGI, SGLang, TensorRT-LLM all support this).

What you can do:

- Measure TTS TTFB p95 under *concurrent* load, not just single-stream.
- Tag the serving tier you need in the request (where the provider exposes it) or negotiate a dedicated endpoint.

### GPU allocation for concurrent realtime sessions

A voice agent is not just a model call; it's a *session* that holds resources for the duration of a call. A handful of concurrent sessions is easy; a few thousand is an infra problem.

Specifically:

- **LLM**: KV-cache per session. If you run a long call with many turns and a large context, the KV cache grows. Multi-tenant servers evict based on policy; a cold cache after eviction is a TTFT spike.
- **Streaming ASR**: a persistent session per caller. Server has to hold model state per session.
- **Streaming TTS**: usually stateless per request, but the server has to keep the vocal-model memory in GPU memory. Under load, scheduling matters.

The inference engineer owns the capacity plan and the hardware provisioning. You own:

- Forecasting **concurrent session count**, not QPS. A voice product with 100 QPS but 30-minute average call length has to hold ~180k concurrent sessions.
- Communicating the **p95 and p99** of session duration, not just the mean. A long-tailed duration distribution multiplies capacity requirements.

### Speculative decoding on the LLM leg

Speculative decoding (draft-model-and-verify, Medusa-style multi-head heads, lookahead decoding, EAGLE, etc.) is a server-side technique that produces multiple tokens per step, lowering TPOT without changing output distribution. Provider adoption is uneven.

What this looks like in your budget:

- **TPOT drops**, sometimes dramatically. This matters for voice because TPOT is what determines whether TTS can keep pace with the LLM.
- **TTFT may be unaffected** or even slightly hurt (speculation adds a little startup cost). The headline turn-gap metric may not move.
- The win is **more headroom for longer utterances** — you can make the agent say more without the TTS falling behind.

You cannot deploy speculative decoding. You can request it on provider feedback channels and you can prefer provider endpoints that have shipped it. The inference engineer can deploy it on self-hosted serving stacks.

### Model-server tail latency tuning

The remaining catch-all. Queueing policies, autoscaling behaviour, graceful shutdown during deployment, request admission control, backpressure. All of it lives inside the server. All of it can quietly add 100–500 ms to your p95.

Signals that this is where your breach lives:

- Turn-gap p50 is fine, p95 or p99 is bad, and **no single leg explains it** — the tail is spread across all legs, which usually means a shared resource (GPU, scheduler) is contending.
- Latency spikes correlate with **deployment** or **scaling** events on the provider side.
- The breach is **time-of-day dependent** — users in the provider's peak hour see worse numbers.

When those signals show up, you are at the escalation boundary.

## What you own on the budget side of the line

To be explicit — these never leave your lap, no matter how bad the serving layer is:

- **The end-to-end budget target** (p95 turn gap), as a product commitment.
- **The streaming graph architecture** — overlap, cancellation, prompt caching, warm pools (Chapter 3).
- **Region and edge placement** (Chapter 4).
- **Per-leg instrumentation and attribution** (Chapter 2).
- **The CI gate and runtime fallbacks** (Chapter 5).
- **Choosing which provider / model you depend on** — if a provider's serving layer is a chronic drag, moving to a better one is a choice you make.
- **Communicating the problem upward** — architectural briefs, incident reports, provider feedback.

The last two are the tricky ones. A voice-agent engineer who thinks "our LLM provider is slow and that's not my problem" is a voice-agent engineer whose product ships slow.

## How to package an escalation

When you escalate to the inference team — or file a provider ticket — the thing that distinguishes an engineer who gets taken seriously from one who doesn't is the shape of the request.

The template that works:

1. **The metric**: which per-leg metric is breached, by how much, for how long, on which traffic slice.
2. **The attribution**: the per-leg timeline from your traces showing the breach is in their domain, not yours.
3. **The reproducer**: ideally a synthetic scenario (Chapter 5) that reproduces the breach reliably in staging.
4. **The hypothesis**: your best guess at the cause (batching tail, KV-cache eviction, cross-region failover), with the evidence.
5. **The ask**: a specific action — tier isolation, a batcher-config change, a capacity bump, speculative decoding — framed as "what you would do if you were us".
6. **The budget impact**: how much of the top-line budget the fix recovers, and what the user-facing consequence is if it is not fixed.

Do not file a ticket that reads "our voice agent is slow". File a ticket that reads "p95 TTFT on the EU-West endpoint for `claude-sonnet-4-6` has regressed from 320 ms to 540 ms since release X on timestamp Y; attached trace shows the batcher queue time is the delta; reproduces in staging with scenario Z; we're 120 ms over our product's 800 ms turn-gap target as a direct result; please consider latency-tier isolation or a batcher config change."

The inference engineer will thank you. They have the same problem you do, from the other side.

## What the inference engineer owes you back

The other half of the contract, so you know what to ask for:

- **Transparent SLOs** on serving-layer latency (TTFT p95/p99, TPOT, TTS TTFB, availability).
- **A dashboard** you can reach that exposes those SLOs in real time.
- **Warning on changes** — model deploys, batcher changes, hardware reconfiguration — that could materially change your numbers.
- **A clear escalation path** when the SLO is breached.

If the inference team you depend on is not providing those, your architectural brief has a risk section to fill in.

## A one-line summary of the whole module

If the preceding five chapters made the budget visible and defensible, this one makes it **negotiable**:

> You own the budget. The inference engineer owns the slice of the budget that lives inside the model server. Measure your side, escalate theirs, and never confuse the two.

That boundary is the thing that keeps you from (a) wasting weeks tuning parameters in a server you don't run, or (b) blaming the serving layer for a sequential pipeline design that no amount of GPU could save.

## Summary

- **Inside the model server**: streaming batching, GPU allocation for concurrent sessions, speculative decoding, tail-latency tuning. Owned by the ai-infra-inference-engineer.
- **Across the voice loop**: budget definition, streaming graph, region / edge placement, instrumentation, CI gate, runtime fallbacks. Owned by you.
- **Escalate well**: metric + attribution + reproducer + hypothesis + ask + budget impact. The shape of the request determines whether it gets acted on.
- **Negotiate the contract**: SLOs, dashboards, change notifications, escalation paths. Add these to your architectural brief's risk section if your serving dependency doesn't provide them.

End of module. Next up: [mod-110-multimodal-input-design](../mod-110-multimodal-input-design/README.md).
