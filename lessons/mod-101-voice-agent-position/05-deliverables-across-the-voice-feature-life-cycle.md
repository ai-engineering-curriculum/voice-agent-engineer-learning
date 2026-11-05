# 05. Deliverables Across the Voice-Feature Life Cycle

## Why this chapter

Role definitions are empty until you can name the artefacts produced. A voice-feature life cycle has eight deliverables that the voice-agent engineer is on the hook for, in roughly this order:

1. Architecture brief
2. Latency budget
3. Speech-pipeline design
4. Telephony integration doc
5. Safety / compliance review
6. Eval bundle
7. Launch-gate decision
8. Operations runbook

The rest of this module's exercises and most of the subsequent modules are organised around producing versions of these documents. This chapter is the reference card for what each one is for, who reads it, and what "done" looks like.

## 1. Architecture brief

**Purpose.** Convey the design to the rest of engineering, product, and (where needed) legal in enough detail that a reader can disagree with a specific decision rather than wave at the whole thing.

**Reader.** Peer engineers, FDE, engineering manager, security review, the applied-AI engineer who owns the text-LLM feature behind the voice one.

**Minimum contents.**

- The problem and user story in one paragraph.
- A single architecture diagram with the components labelled: client, transport, ASR, VAD/turn-taker, LLM, TTS, orchestration, tools, observability.
- The buy-vs-build calls — which providers for ASR / TTS / LLM, which agent framework, which transport.
- The cascaded-vs-speech-to-speech decision, with a one-paragraph rationale.
- The known risks and open questions with named owners.

**Done when.** A peer who was not in the room can read it and tell you what they'd argue against.

## 2. Latency budget

**Purpose.** Pre-commit the end-to-end latency target per component so the team doesn't discover at launch that they've overspent.

**Reader.** You, inference engineering, FDE, engineering manager.

**Minimum contents.**

- A target end-to-end response time (user utterance end → agent audio start). State the percentile (P50 and P95 at minimum).
- A per-component allocation: endpointing delay, STT finalisation, LLM TTFT, TTS TTFB, transport jitter buffer, miscellaneous.
- Measurement methodology: how each slice is instrumented, in which traces, with which clock.
- A P95/P99 story — not just the median — because conversational feel is dominated by the tail.

**Done when.** You can point at any slow call and attribute the overage to exactly one component.

## 3. Speech-pipeline design

**Purpose.** Specify the code-level composition of the pipeline so that a new engineer joining the project can reproduce it.

**Reader.** Peer engineers on the voice team.

**Minimum contents.**

- Audio format conventions (sample rate, encoding, frame size) at each hop.
- The streaming contracts (what producers emit, what consumers expect, how backpressure and cancellation are signalled).
- The barge-in behaviour — what cancels what, when, and how race conditions are handled.
- The fallback behaviour — provider timeouts, network reconnections, degraded-quality modes.
- The tool-use integration — how long-running tools are surfaced to the user while running.

**Done when.** A new engineer can read this doc, read the code, and tell you which code chunk implements which paragraph.

## 4. Telephony integration doc

**Purpose.** Pin down the CPaaS / SIP / WebRTC edge so the carrier side and the LLM side can be debugged independently.

**Reader.** Telephony / CPaaS specialist (internal or vendor support), SRE, FDE.

**Minimum contents.**

- Call-flow diagram covering inbound and outbound.
- The bridge contract — Twilio Media Streams / ConversationRelay, Vonage WebSocket, LiveKit SIP ingress, or a direct SIP gateway.
- Codec negotiation (Opus for WebRTC, µ-law / PCMU for PSTN, transcoding points).
- DTMF handling, transfer behaviour, warm transfer vs. cold transfer.
- STIR/SHAKEN attestation if outbound; recording-consent prompts if inbound.
- Observable fields per call: carrier, jitter, packet loss, codec in use, SIP response codes.

**Done when.** A carrier-side support engineer can read it and give useful advice on a specific failure without a live screenshare.

## 5. Safety / compliance review

**Purpose.** Document that the product satisfies the applicable rules before launch, not after an incident.

**Reader.** Security, legal, compliance, agentic-safety engineer, FDE on regulated customers.

**Minimum contents (the standard checklist to tailor per feature).**

- Recording / disclosure — per-jurisdiction two-party vs. one-party consent; recording-notice prompt wording; retention policy.
- PII handling — redaction in transcripts and logs; data-residency; sub-processor list.
- Regulated-content flows — HIPAA for clinical, TCPA for outbound US calls, GDPR for EU voice data, PCI for payment flows (pause-and-resume or DTMF-only capture).
- Voice-cloning consent — explicit opt-in for synthesising a specific person's voice; impersonation refusal rules.
- Dangerous-capability guardrails — what the agent must refuse; how the refusal is logged; how it sounds (polite, firm, routable to a human).
- Red-team summary — the attacks tried, the ones that landed, and the mitigations shipped.

**Done when.** Legal and security sign off in writing; the agentic-safety engineer is on the distribution list.

## 6. Eval bundle

**Purpose.** Prove the pipeline is good enough before launch and measurably regressing after.

**Reader.** You, eval engineer, engineering manager, the product team deciding whether to ship.

**Minimum contents.**

- Dataset manifests — recorded calls with accents, noise, cell-network degradation; synthetic hard cases; red-team transcripts.
- Metrics — streaming WER / CER on your corpus; TTS AB or MOS where feasible; latency distributions; barge-in false-positive / false-negative rates; task-completion rate; refusal correctness.
- Judges — rubrics for conversational quality; whether a judge is text-only (reads the transcript) or multimodal (listens to the synthesised audio); calibration against human raters.
- Baselines — the last-known-good pipeline and its numbers.
- Regression gates — which thresholds block a launch.

**Done when.** The CI run on a PR produces numbers that an eval-engineer peer would sign off as decision-grade.

## 7. Launch-gate decision

**Purpose.** A single document that records the launch decision and its rationale.

**Reader.** Engineering manager, product, exec sponsor, future-you during the postmortem.

**Minimum contents.**

- The scorecard from the eval bundle against the pre-committed gates.
- The safety / compliance sign-off status.
- The rollout plan — percentage ramp, cohort, rollback trigger.
- The on-call plan — who, what hours, what runbook.
- The out-of-scope list — what this launch is *not* promising.

**Done when.** The decision is written down before the launch, not reconstructed after.

## 8. Operations runbook

**Purpose.** What oncall does at 3 a.m. when something is wrong.

**Reader.** Oncall engineers (you, your peers, the FDE handling a specific customer incident, the SRE on a cross-cutting outage).

**Minimum contents.**

- Dashboards — latency, error, concurrency, carrier-side metrics, per-provider breakdowns.
- The common failure modes with signatures: provider down, carrier down, LLM slow, TTS timed out, VAD stuck open, phone number unreachable.
- Per-failure first-response steps — the specific CLI commands, dashboards to open, config toggles, and feature flags.
- Escalation table — who to page for which failure.
- Postmortem template — what you want from the next incident write-up.

**Done when.** A peer who has never been on this product's oncall can run a Sev-2 to resolution using only the runbook.

## Lifecycle view

```
discovery    design           build              launch                 operate
───────── → ──────────── → ─────────────── → ──────────────────── → ─────────────
 brief   latency budget    pipeline design     eval bundle            runbook
         architecture       telephony doc       safety review          postmortems
         decisions          observability       launch-gate decision
```

Each deliverable corresponds to a decision that cannot be meaningfully deferred to the next stage without later rework.

## How this module's work maps forward

Subsequent modules in this track produce in-depth versions of these deliverables:

- `mod-102-speech-pipeline-foundations` and `mod-103-stt-production-engineering` / `mod-104-tts-production-engineering` — speech-pipeline design.
- `mod-105-realtime-speech-to-speech` — the cascaded-vs-S2S architectural decision.
- `mod-106-vad-turn-taking-barge-in` — the hardest parts of the pipeline design.
- `mod-107-telephony-webrtc-transport` — the telephony integration doc.
- `mod-108-safety-compliance-voice` — the safety / compliance review.
- `mod-109-latency-budget-architecture` — the latency budget, deepened.
- `mod-110-multimodal-input-design` — multimodal extensions to the architecture brief.
- `mod-111-voice-agent-orchestration` — the orchestration layer of the pipeline design.
- `mod-112-voice-multimodal-eval-and-observability` — the eval bundle and the runbook's observability section.
- `project-104-speech-native-architectural-brief` — the architecture-brief capstone.

## Summary

- Eight deliverables span the voice-feature life cycle: architecture brief, latency budget, pipeline design, telephony integration doc, safety / compliance review, eval bundle, launch-gate decision, operations runbook.
- Each has a named reader, a minimum table of contents, and a definition of "done" that is not optional.
- The remaining modules in the track produce deep versions of these same artefacts. Module 101 is where you learn the shape; the rest of the curriculum is where you fill it out.
