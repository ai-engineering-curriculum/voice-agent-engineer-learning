# 08. Build vs. Buy, End to End

## Why this chapter

Every chapter so far has been a *piece* of the orchestration decision — Pipecat, LiveKit Agents, Vocode, managed platforms, the interrupt state machine, tool calls, memory. This chapter is where those pieces resolve into a single architectural commitment that you defend in a design review. The audience is your product counterpart, your platform-security reviewer, and your director — the people who will ask "why this and not the other thing", and expect the answer to tie to the budget, the compliance posture, and the eval bar you have already committed to.

The three deployment archetypes are familiar by now; this chapter puts them side by side on the axes that matter and gives you the shape of the defence.

## The three archetypes, named

- **Managed voice-agent platform.** Retell, Vapi, Bland, or a contact-centre AI product running the voice loop for you. You own the prompts, tools, and some config; they own the orchestration layer, the voice stack, the transport, and the operational burden.
- **Own pipeline on an open-source framework.** Pipecat or LiveKit Agents running on your own compute, with explicit choices of STT, LLM, TTS, transport, and plugins. You own the orchestration and all the operational burden; you also own all the knobs.
- **Frontier Realtime API directly.** OpenAI Realtime, Gemini Live, or Hume EVI called straight from a thin orchestration layer. The model provider owns the speech-to-speech loop; you own session management, tool dispatch, transport adaptation, and the control plane around it. The "frontier" position here is about the audio quality and tight audio-first integration; the latency ceiling and control surface are weaker than a well-tuned own-pipeline for anything beyond the simplest shape.

A fourth, honest archetype is the **hybrid** — a managed platform at the front door, an own-pipeline specialist on the back leg, both using MCP for shared tools. Chapter 4 covered the shape; it's the right answer surprisingly often.

## The axes you defend against

### 1. Latency, against the mod-109 budget

You committed in mod-109 to a p95 turn-gap target (typically ~800 ms end-to-end for conversational feel). Each archetype has a different latency profile:

- **Frontier Realtime API** — the fused speech-to-speech model is fast on the inner loop, but you have no control over the provider's inference region, no ability to colocate with your media server, and no way to tune the interrupt flush. In practice, the p95 is set by the provider and often sits above your target in all but their best-supported regions. Lowest control, hardest to improve.
- **Own pipeline on Pipecat / LiveKit** — you pick every leg: STT in-region, LLM endpoint in-region, TTS streaming, transport colocated with the media server. If you miss the budget, you have a lever for every leg (mod-109 Chapter 3). Highest ceiling once tuned, biggest tuning cost up front.
- **Managed voice-agent platform** — the vendor has their own pipeline tuned to their own target. Often meets a conversational-feel budget on their happy path; often misses it on your odd traffic shape (odd region, odd codec, high-load tails). Needs measurement.

The defence for your choice: a measurement, in your region, on your traffic, at your p95. "Vendor X documents 700 ms" is not the defence; "we measured 740 ms p95 in us-east on our traffic in week of 2026-11-01" is.

### 2. Compliance, against the mod-108 posture

mod-108 committed you to a specific posture: lawful basis under GDPR, HIPAA coverage if PHI is in scope, PCI-DSS behaviour around card data, consent and retention policies, breach-notification obligations, model-training opt-out. Against each archetype:

- **Frontier Realtime API** — the provider is a sub-processor; you need a DPA, you need their current data-handling terms in writing, you need region pinning if your residency rules require it, you need their BAA if PHI is involved (not all providers offer one). Consent-driven audio stop and PCI pausing have to be built on top; the provider may or may not expose the hooks.
- **Own pipeline** — every sub-processor is your explicit choice; the posture is the sum of the DPAs you signed for Deepgram, OpenAI, ElevenLabs, your cloud provider, etc. Most control, most paperwork.
- **Managed platform** — one DPA with the platform, which is the *fewest contracts* but also the *most concentrated risk*. A single vendor outage or compliance change is your whole product's problem. Check what sub-processors the platform uses and that the chain of DPAs reaches your regulators.

The defence for your choice: a reviewed, dated list of agreements signed per sub-processor, and a specific mapping from your policy (e.g., "all EU user audio must stay in the EU") to the configuration that enforces it ("Vendor X EU region; LLM provider EU endpoint; TTS EU region; transport EU-only media server").

### 3. Eval bar, against mod-112

mod-112 will formalise your eval harness (end-to-end conversational quality, interrupt handling, grounding quality, safety, speech quality). Against each archetype:

- **Frontier Realtime API** — eval is end-to-end; you see the user, the model, and the response, but the intermediate steps (ASR output, model reasoning) are opaque. Debugging eval failures is harder because you cannot inspect the leg that failed.
- **Own pipeline** — eval can be per-leg: ASR WER on your traffic, LLM task quality, TTS MOS, interrupt tail-stop latency, end-to-end latency. Each leg has an explicit metric and an explicit owner. Diagnosable, more infrastructure.
- **Managed platform** — eval is mostly end-to-end (the platform may expose some leg-level metrics, or may not). If the eval fails, you escalate to the vendor; you cannot fix it yourself. Third-party eval vendors (Hamming et al.) help here but they test externally.

The defence for your choice: a concrete eval plan that your archetype supports. "We measure p95 latency, WER, MOS, safety-eval pass rate, and three in-domain task metrics; on archetype X we have the following access to each metric: ...". If the archetype can't support your eval plan, the archetype is the wrong one.

### 4. Cost, honestly

Cost is often the loudest argument and the easiest to over-sell. A fair version:

- **Managed platform** — cheapest at low volume (no FTE time, no infra); crosses over to expensive at high volume (the per-minute rate times a lot of minutes is a lot). Also opaque: pricing changes without your input.
- **Own pipeline** — cheapest at high volume (you control every sub-vendor contract); expensive at low volume because FTE and infra are mostly fixed costs. Predictable.
- **Frontier Realtime API** — midpoint; the model is per-token-per-minute, your orchestration is cheap but has a fixed FTE baseline.

The defence: a 12-month forecasted cost with the per-minute rates and FTE time annualised, on your volume forecast. The number is a point estimate; include a sensitivity to volume (half and 2× your forecast).

### 5. Control, specifically

"We need more control" is the vaguest defence anyone gives. Make it concrete by listing *which knob*:

- A custom voice (ElevenLabs IVC trained on your brand voice actor) that managed platforms don't support.
- Per-tenant language model (a fine-tuned LLM per customer) that only an own-pipeline can host.
- An ultra-low tail-stop interrupt (sub-200 ms) that the provider doesn't expose.
- A specific on-device inference step for a privacy-sensitive leg.
- An MCP integration to an internal tool server that the managed platform doesn't support.

If none of these apply, your "control" argument is weak and you probably should buy.

### 6. In-house ML, honestly

If your team's edge is the ML — STT in your domain, TTS cloned voices, a fine-tuned LLM on your call corpus — a managed platform wastes it. If your team's edge is the product, the integrations, or the go-to-market, your ML is usually a commodity and a managed platform is not wasting anything.

Be honest about which team you are.

## The archetype-selection grid

A small decision grid you can reuse. For each dimension, score each archetype on a 1–3 scale (3 is best for your specific product):

| Dimension | Frontier Realtime | Own pipeline | Managed platform | Hybrid |
|---|---|---|---|---|
| Hits mod-109 p95 target on our region/volume |   |   |   |   |
| Supports our mod-108 posture (DPA, BAA, PCI) |   |   |   |   |
| Supports our mod-112 eval plan |   |   |   |   |
| 12-month cost on forecast volume |   |   |   |   |
| Exposes the knobs we listed |   |   |   |   |
| Lets us use our in-house ML advantage |   |   |   |   |
| Operational burden fits our team size |   |   |   |   |
| Lock-in exit cost is acceptable |   |   |   |   |

The grid isn't a scoring algorithm — it's a *conversation shape*. The archetype with the most 3s is usually right; but the archetype with the most 3s and a 1 in compliance is wrong, because a 1 in compliance is a deal-breaker, not an average.

## The defence, structured

A well-formed design-review defence has four parts. Most reviewers want this in a single page.

### Part 1 — the committed targets

Re-state, don't rediscuss: "our mod-109 target is p95 end-to-end < 800 ms, our mod-108 posture is HIPAA + state-level consent, our mod-112 eval bar is <5% regression on interrupt tail stop over 30 days." If anyone wants to re-open any of these, this is the design review for the *architecture*, not for the targets.

### Part 2 — the measured candidates

For each archetype you considered, a short section:

- What specifically you tested (prototype, scope, duration, traffic).
- The measurements against your targets.
- The qualitative finding about control and compliance.
- The cost estimate.

A quick row like "Vapi, inbound support traffic, 2 weeks, 500 calls, p95 820 ms, HIPAA BAA available but DPA addendum pending, 12-month cost projection $X" is enough. The reviewers care that you *measured*, not that you *picked*.

### Part 3 — the decision

The chosen archetype. One line, then one paragraph of justification tied to the measurements in Part 2. No marketing language.

### Part 4 — the exits

Two things:

- **The exit criteria** — "if any of these three conditions becomes true, we revisit the decision: (1) p95 exceeds 1000 ms for three consecutive days; (2) the vendor announces a compliance change; (3) our volume doubles beyond forecast". Pre-committed exit conditions turn a one-time decision into a reviewable one.
- **The exit cost** — what it would take to move. "To exit Vapi for our own pipeline we would need ~6 engineering weeks for the migration plus a 2-month shadow traffic period." Reviewers trust architects who already know the exit cost.

## A note on timing — the decision is not permanent

Orchestration choices have shorter half-lives than you think. Managed platforms add and remove features; open-source frameworks consolidate; realtime models from different providers take turns being the best on a given axis. A 12-month-old architectural decision deserves a quick re-evaluation on every renewal cycle (annual, typically) — not a full re-design, but at least a "did anything move?" review.

The flip side: if your organisation routinely re-opens architectural decisions every two months, you will ship nothing. The point of writing the decision, the targets, and the exit criteria down is so you can say "no, we are not re-opening this until an exit condition is hit."

## Interactions with the earlier modules — a cross-reference

Pulling the threads together:

- **mod-102 speech pipeline foundations** gave you the mental model this chapter puts into product form.
- **mod-103 STT / mod-104 TTS / mod-105 realtime / mod-106 VAD** picked the components; this chapter picks who assembles them.
- **mod-107 telephony / WebRTC** picked the transport; the orchestration layer picks the transport *abstraction* on top.
- **mod-108 safety / compliance** is where the posture lives; this chapter is where the posture turns into a vendor-sub-processor diagram.
- **mod-109 latency** is where the budget lives; this chapter is where the budget turns into a committed p95 on a specific archetype.
- **mod-110 multimodal** is where the modality contract lives; the chosen orchestration framework has to carry the vision channel too.
- **mod-112 voice / multimodal eval and observability** will pick up where this chapter leaves off — once the architecture is picked, mod-112 instruments it.

## Summary

- The three archetypes are **frontier realtime**, **own pipeline on Pipecat / LiveKit**, and **managed voice-agent platform** — with **hybrids** a frequent fourth option.
- Defend the choice on six axes: **latency vs. mod-109**, **compliance vs. mod-108**, **eval vs. mod-112**, **cost**, **control (specific knobs)**, and **in-house ML**.
- Use an **archetype-selection grid** that scores each option on each axis, with compliance treated as a veto, not an average.
- Write the defence in four parts: **committed targets**, **measured candidates**, **the decision with justification**, and **exit criteria + exit cost**.
- Re-evaluate on a yearly cadence; don't re-open mid-cycle without an exit condition firing.
- The decision is boring on purpose. Boring architectural decisions age better than clever ones.

End of module. Next module is [mod-112 — Voice and Multimodal Eval and Observability](../mod-112-voice-multimodal-eval-and-observability/README.md), which instruments whatever you chose here.
