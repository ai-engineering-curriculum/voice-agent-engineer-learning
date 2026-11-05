# 02. Position on the AI Engineering Ladder

## Why this chapter

The Voice & Multimodal Agent Engineer is **level 35** on the AI Engineering ladder. Levels 30 are the entry-level applied roles; level 35 is where you own a product surface end-to-end; level 40 is senior engineering or management. "Realtime streaming audio" is the thing this role adds to the levels below and hands up to the levels above.

Use this chapter as a boundary map: when another engineer in the org is doing text-LLM work that looks suspiciously similar to your voice work, this chapter tells you which parts are theirs and which are yours.

## The AI Engineering ladder, visualised

```
L40  senior-agentic-ai-engineer          ai-engineering-manager
       │                                   │
       │ inherits upward                   │ inherits upward
       │                                   │
L35  voice-agent-engineer       forward-deployed-engineer     ai-infra-inference-engineer
       │ (you are here)               │                           │
       │                              │                           │
       │  adds realtime streaming audio to the stack below        │
       │                                                          │
L30  applied-ai-engineer    agentic-ai-engineer    ai-eval-engineer    rag-engineer
```

Levels are coarse. Treat them as scope-of-ownership markers, not as seniority gates.

## What L30 roles own, and what voice adds

### vs. `applied-ai-engineer` (L30)

The applied AI engineer owns **text-LLM feature engineering**: prompts, context assembly, tool definitions, retries, model routing, output structure, product-level evals. The voice-agent engineer **inherits all of that** and adds:

- A **streaming event model** — audio frames at ~20 ms cadence, partial ASR transcripts, LLM tokens, TTS audio chunks — rather than one request / one response.
- A **turn-taking contract** — VAD, endpointing, barge-in, silence handling — that doesn't exist in text chat because keystrokes are an unambiguous turn signal.
- A **codec and transport story** — Opus over WebRTC, µ-law / PCMU over SIP, raw PCM over WebSocket — that an applied AI engineer never has to reason about.

Rule of thumb: if a feature works unchanged in a text chat widget, it belongs to applied-ai-engineer. The moment the design mentions "while the user is speaking…" or "barge-in" or "time-to-first-audio", it belongs to the voice-agent engineer.

### vs. `agentic-ai-engineer` (L30)

The agentic engineer owns **agent-framework internals**: tool-use loops, planning, memory, multi-step execution, failure recovery, cost accounting. The voice-agent engineer **consumes those frameworks** and adds:

- **Latency-constrained orchestration** — the agent loop has to produce a first audible token in roughly the time a human expects a response in conversation, not when it's done thinking.
- **Interruptible execution** — user barge-in has to cancel in-flight LLM generation and TTS synthesis cleanly, which most text-mode agent loops don't natively model.
- **Audio-aware tool choice** — "read this out loud" vs. "show this on the screen" is a modality decision the agent loop has to make, not a formatter decision after the fact.

Rule of thumb: if the agent loop can run as a batch job with no human on the other end, it belongs to agentic-ai-engineer. If it has to feel like a conversation, it belongs here.

### vs. `ai-eval-engineer` (L30)

The eval engineer owns **evaluation methodology**: dataset construction, judge design, calibration, metric selection, guarding against reward hacking. The voice-agent engineer inherits that methodology and adds:

- **Voice-specific metrics** — WER / CER for ASR; MOS or AB preference for TTS; end-to-end conversational latency distributions; barge-in false-positive rate.
- **Audio-grounded test sets** — recorded calls with noise, accents, crosstalk, cell-network degradation; not just text transcripts.
- **Multimodal judge design** — a judge that listens to the agent's TTS output, not just reads its text, catches "the words are right but the prosody is wrong."

Rule of thumb: an ai-eval-engineer designs the evaluation harness; the voice-agent engineer owns the voice-specific dimensions that go into it. For a deep judge-methodology question, you escalate (see Chapter 4).

## What L35 peers own, and where the boundary sits

### vs. `forward-deployed-engineer` (L35)

The FDE owns **customer-embedded delivery**: being onsite / onchannel with a specific enterprise customer, integrating against their systems, closing the gap between generic product and specific workflow. The voice-agent engineer owns **the voice pipeline itself** — the stack of ASR, VAD, LLM, TTS, orchestration, transport, and evals — regardless of which customer is using it.

In a typical deployment:

- The FDE translates "our call-centre runs on Genesys and our CRM is Salesforce" into integration requirements.
- The voice-agent engineer translates those requirements into the pipeline: STT provider choice, latency budget, TTS voice selection, telephony edge, fallback behaviour.

They work together constantly. The FDE's deliverable is a working customer; the voice-agent engineer's deliverable is the voice stack that makes that customer work.

### vs. `ai-infra-inference-engineer` (L35)

The inference engineer owns **the serving layer**: GPU scheduling, batching, speculative decoding, tail-latency reduction, cost-per-token at the model level. The voice-agent engineer **consumes** the serving layer — usually as a hosted API from a lab or provider, occasionally as a self-hosted Triton / vLLM deployment the infra team runs.

The sharp boundary:

- Latency at the **endpoint-to-first-token** level inside a model server belongs to inference engineering.
- Latency across the **full voice loop** (user-utterance-end → agent-first-audible-syllable) belongs to this role. You own the budget that the serving layer has to fit inside.

If you discover your P95 is bad because the LLM provider's tail latency is bad, your job is to measure it, prove it, and escalate; their job is to fix the server.

## What L40 inherits upward

### `senior-agentic-ai-engineer` (L40)

Owns multi-agent systems, cross-product agent platforms, long-horizon planning, formal evaluation programmes. A voice-agent engineer who scales up tends to grow into this role when voice becomes one modality among several in a larger agent platform rather than the whole product.

### `ai-engineering-manager` (L40)

Owns hiring, team shape, roadmap, cross-functional coordination. The voice-agent engineer feeds this role with (1) architectural briefs that make buy-vs-build decisions tractable, (2) eval bundles that let the manager defend a launch decision to the rest of the org, (3) runbooks that mean oncall isn't the manager's weekend.

## A one-page handoff table

| Edge | What they own | What you own | Where the line sits |
|---|---|---|---|
| applied-ai-engineer | Text LLM feature work | Realtime audio loop around that work | The moment "streaming audio" enters the design |
| agentic-ai-engineer | Agent loops, tools, memory | Latency- and interruption-aware orchestration | Whether a human waits in realtime |
| ai-eval-engineer | Eval methodology | Voice-specific test sets and metrics | Judge design is theirs; audio dimensions are yours |
| forward-deployed-engineer | The customer's workflow | The voice pipeline reused across customers | Customer-specific vs. product-wide |
| ai-infra-inference-engineer | Model server internals | Full voice-loop latency budget | Inside the model vs. across the pipeline |
| senior-agentic-ai-engineer (L40) | Multi-agent / platform | Single voice product surface | Scope of the agent platform |
| ai-engineering-manager (L40) | Team, roadmap, headcount | Technical deliverables that justify decisions | People vs. systems |

## Summary

- Voice-agent engineer = **L35**, with ownership of the realtime voice pipeline end-to-end.
- Relative to **L30** roles: you inherit their text-LLM, agent-framework, and eval foundations and add the streaming-audio, turn-taking, and voice-eval layers.
- Relative to **L35** peers: FDE is customer-facing where you are product-facing; inference engineer is server-internal where you are pipeline-wide.
- **L40** inherits either multi-agent breadth (senior-agentic) or team ownership (manager); either way, the voice-pipeline mastery you build here is what gets delegated down.

Next chapter: adjacent non-engineering roles in the voice ecosystem, and where the boundaries are intentionally sharp.
