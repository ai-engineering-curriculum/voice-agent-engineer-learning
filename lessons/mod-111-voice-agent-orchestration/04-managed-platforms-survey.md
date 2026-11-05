# 04. Managed Voice-Agent Platforms — The Survey

## Why this chapter

Chapters 1–3 cover the frameworks you run yourself. There is a fourth option: pay a vendor to run the orchestration layer for you. That category — **managed voice-agent platforms** — has matured fast, and for a non-trivial fraction of real products the right engineering answer is "use one of these and stop". Your job as a voice-agent engineer is to know the shape of the category, know the specific platforms well enough to defend a build-vs-buy call, and to know what you give up by handing the orchestration layer to a vendor.

This chapter is a survey, not a shopping list. The specific products move: new ones launch, old ones acquire each other, pricing structures and feature matrices change month to month. What you should take from the chapter is the **vendor patterns** and the **decision criteria**. The headline products are placeholders for the pattern.

## The three main patterns

Within "managed voice-agent platform", three product shapes recur. Any given vendor usually leans on one of them.

### Pattern A — Conversation-flow / pathway builders

The platform ships a **visual designer** for the dialog, a library of pre-built blocks (collect info, call an API, transfer the call, hang up), and a managed voice stack underneath (STT + LLM + TTS + telephony). You draw the conversation; the platform runs it.

- The **unit of design** is a graph or state machine, not a system prompt.
- Non-engineers can author flows; engineers review and plug in the tool / API blocks.
- Highest fit for scripted, domain-constrained conversations: appointment booking, voicemail triage, outbound qualification, lead routing.

Examples: **Retell AI** (Conversation Flow), **Bland AI** (Pathways).

### Pattern B — Assistant orchestration

The platform exposes a programmable "assistant" (an LLM, a voice, a tool catalogue, a set of lifecycle configs) and a dispatch layer. You create assistants via API, kick off calls, handle transfers, chain assistants together (one takes the opening, another takes a specialist leg). The platform runs the voice loop; you write the prompts, tools, and routing.

- The **unit of design** is still prompts and tools, closer to what you'd do on Pipecat or LiveKit — but the voice loop, telephony, interrupt handling, and VAD are theirs.
- Fit for teams who want the behaviour of a code-level agent but do not want to run the stack.
- Often support **squads / handoffs** (multiple assistants per call) and **transfer workflows**.

Examples: **Vapi** (assistants + squads + transfer), **Retell AI** (programmable Agents), **Bland AI** (custom LLM).

### Pattern C — Contact-centre AI, with voice-agent features inside it

Platforms whose primary product is contact-centre tooling (quality monitoring, agent assist, workforce management) and who have added voice agents and/or conversation intelligence as a feature. Buyer is a VP of Support or a COO, not a platform team.

- Voice-agent features are **part of a larger product** — QA, ingestion of post-call transcripts, agent coaching, compliance scorecards.
- Deep integration with CCaaS platforms (Genesys, Five9, Amazon Connect, Zoom CC).
- You buy them when the agent is one feature in a wider contact-centre modernisation, not the headline deliverable.

Examples: **Observe.AI**, **Phonely** (contact-centre-focused voice-AI), **Hamming** (eval / testing specialist for voice-agent and contact-centre pipelines — a valuable adjacency, especially feeding mod-112).

## The headline products — what each one is for

The point of naming specific vendors is to give you concrete shapes to think about. Check their current docs before quoting features; everything below is positional, not canonical.

### Retell AI — [retellai.com](https://www.retellai.com/)

- **Conversation Flow** visual designer (Pattern A) and a programmable Agent SDK (Pattern B) under one roof.
- Built-in voice library covering multiple vendors.
- Batch outbound dispatch for calling many numbers.
- Transfer, SIP integration, custom tools.
- Pricing per minute; LLM calls billable separately in some configs.

Natural fit when: you want the flow builder for the bulk of the dialog but need to drop into code occasionally for the hard bits.

### Vapi — [vapi.ai](https://vapi.ai/)

- Assistants + **squads** (multiple assistants per call, with explicit transfer semantics) — Pattern B.
- Programmable via REST and SDKs; webhook-based tool calls.
- Supports your own LLM endpoint for teams who want to keep the brain in-house.
- Built-in telephony (BYO-carrier supported).
- Price per minute, plus LLM / STT / TTS passthrough depending on config.

Natural fit when: you want to control prompts and tools in code, but let the platform own voice-loop, VAD, interrupts, and telephony.

### Bland AI — [bland.ai](https://www.bland.ai/)

- **Pathways** (visual + code hybrid; Pattern A).
- Managed voice stack with their own in-house models.
- Heavy on outbound dispatch at scale (phone-call campaigns).
- Custom-LLM option for enterprise deployments.

Natural fit when: your product is high-volume outbound (collections, scheduling, outbound sales) and you want a vendor whose core use case matches yours.

### Hamming — [hamming.ai](https://hamming.ai/)

- Not an agent *runtime*; an **eval and testing** platform for voice agents (Pattern C, adjacency).
- Automated test calls, scenario simulation, regression detection.
- Pairs with Pipecat/LiveKit/managed platforms; it is a complement, not a substitute.

Natural fit when: you are running your own stack and need an eval harness, or you are shipping a managed platform agent and want third-party QA. Also interesting for mod-112.

### Phonely — [phonely.ai](https://www.phonely.ai/)

- Contact-centre-oriented voice-AI platform (Pattern C).
- Focus on receptionist / inbound / overflow workloads.
- Integration with CCaaS products.

Natural fit when: you're building a receptionist or inbound-overflow agent for a non-tech buyer.

### Observe.AI — [observe.ai](https://www.observe.ai/)

- Contact-centre AI with QA, agent assist, and more recently voice-agent capabilities (Pattern C).
- Deep CCaaS integrations.
- Enterprise contracts and procurement motion.

Natural fit when: the voice agent is one tile in a broader contact-centre modernisation program.

## Build-vs-buy decision criteria

The right decision depends on five dimensions. Be honest with yourself on each.

### 1. Cost

Managed platforms typically charge **per minute** of conversation, often with LLM/STT/TTS passthrough added on. At low volume, that's cheaper than running your own stack. At high volume, it crosses over — a product doing millions of minutes a month usually finds the per-minute price larger than the fully-loaded cost of running Pipecat or LiveKit on your own infrastructure.

The honest exercise: project the per-minute rate times your 12-month volume forecast; put the all-in cost of your own stack next to it (engineering time to build and maintain, infra bills, incident hours). If the numbers are close, cost is not the deciding axis. If they're an order of magnitude apart, cost is deciding.

### 2. Control

What can you change about the agent?

- Prompt text — all platforms let you.
- Tool definitions — all platforms let you.
- The LLM model — some do, some don't.
- The TTS voice or STT vendor — usually some, often limited to a fixed list.
- The interrupt and turn-detection behaviour — rarely exposed in detail.
- The audio processing stack (AGC, noise suppression, echo cancellation) — almost never.
- The transport and media routing — platform's.
- The exact tool-call invocation shape and error handling — partially.

Build when the agent needs fine-grained control that the platform does not expose and you cannot tolerate waiting on their roadmap. Buy when the control the platform exposes is enough.

### 3. Compliance

mod-108 commits you to a compliance posture. For a managed platform to be usable, it must support **that** posture — not a generically "secure" one.

Specifics to put on paper before you sign:

- **Data processing agreement (DPA)** aligned with your legal basis (GDPR).
- **Business associate agreement (BAA)** if PHI is in scope (HIPAA).
- **PCI-DSS** scope reduction — does the platform pause STT on card-number utterances? Does it redact in the transcript?
- **Data residency** — can you pin processing to a specific region?
- **Model-training opt-out** — can you guarantee your voice data is not used to train their models?
- **Subprocessor list** — which vendors (model providers, cloud regions, voice services) does the platform in turn depend on, and have you reviewed each?
- **Call recording and retention** — per your policy, and on your schedule, with your deletion API.

Build when any of these is "no" and there is no path to "yes" in a timeline you can live with. Buy when the answer is "yes, documented, in writing" — not "we're working on it".

### 4. Latency

mod-109 gave you a budget. A managed platform runs on their media routing and their model choices; their p95 turn gap may or may not fit inside your budget. Three things to actually measure, not trust the marketing page:

- End-to-end p50 and p95 from your users' regions, on your traffic shape.
- Tail behaviour under load (not just a quiet weekday demo).
- Behaviour on an interrupt (how fast does the TTS stop when the user speaks over the bot?).

Build when the measured platform latency breaches your budget and there's no provisioned-capacity / region-pinning lever that fixes it. Buy when the platform meets the budget today and has a credible story for maintaining it.

### 5. In-house ML and differentiation

What is your team actually *best at*? If your competitive edge is the quality of the voice (custom cloned voices), the quality of the ASR on your in-domain audio, or a bespoke LLM fine-tuned on your product data, a managed platform typically does not let you use those assets. You end up with the platform's models, which wastes your advantage.

Build when your differentiation is in the ML stack. Buy when the differentiation is elsewhere (product UX, integrations, go-to-market).

## Hybrid patterns

The decision isn't always all-or-nothing.

- **Managed platform for inbound, own stack for outbound at scale.** Different traffic shapes, different economics, different control needs. Perfectly reasonable to split.
- **Managed platform at the front, own stack for the specialist leg.** The front door is a Retell / Vapi assistant; on escalation it warm-hands-off (Chapter 7) to a LiveKit or Pipecat agent you run.
- **Own stack with an eval provider on top.** You run Pipecat, Hamming runs the test harness.
- **Managed platform with your own LLM.** Vapi / Bland / others support custom LLM endpoints — you own the brain, they own the voice loop.

Write the hybrid shape down. "We use Vapi for inbound but our outbound campaigns run on a self-hosted LiveKit fleet" is a defensible architecture. "We kind of use Vapi and also have some LiveKit code in a branch" is not.

## The checklist to bring to the decision meeting

When you make the build-vs-buy call formally, bring this one-page checklist (adapt the numbers to your product):

| Criterion | Build-ourselves answer | Vendor-X answer | Decisive? |
|---|---|---|---|
| Projected 12-month minutes | | | |
| Projected 12-month $ cost | | | |
| Required control: which knobs? | | | |
| Required compliance: DPA, BAA, PCI, residency | | | |
| Required p95 turn-gap (mod-109) | | | |
| Required interrupt tail-stop latency (mod-106) | | | |
| In-house ML assets we'd be shelving | | | |
| Team available to operate: FTE-weeks/yr | | | |
| Lock-in exit cost | | | |
| Roadmap items blocked on vendor | | | |

The ratification is that every row has an owner and a dated answer. The common failure is that half the rows are "we'll figure it out later" — at which point you cannot defend the choice to anyone, including your future self.

## Summary

- Managed voice-agent platforms come in three patterns: **conversation-flow builders**, **assistant-orchestration platforms**, and **contact-centre AI products** with voice-agent features.
- Headline products — Retell, Vapi, Bland, Hamming, Phonely, Observe.AI — are illustrative; check current docs and SLAs before quoting features.
- Decide build vs. buy against **five axes**: cost, control, compliance, latency, and in-house ML differentiation.
- **Hybrid patterns** (managed for inbound, own stack for outbound; managed front door with a self-hosted specialist leg; own stack with eval vendor) are often the right answer; write the split down.
- The deliverable of the decision is a one-page checklist with a dated answer for every row — not a general sense of preference.

Next chapter: the interrupt-aware state machine — regardless of framework or platform, this is the design artefact that makes the cancel-and-resume contract real.
