# 08. Role Boundaries and Handoffs

## Why this chapter

Safety and compliance for a voice agent is not one role's job. It is at least four, often more, cooperating under a shared compliance envelope. Knowing where *your* seat ends is as valuable as knowing what sits in it. This chapter names the boundaries this module respects and the artefacts you hand off at each edge.

## The adjacent roles

### `ai-risk-engineer` (Governance family, L25)

This is the risk-engineering slice: harm model, red-team data, risk register, DPIA / FRIA authoring, bias audit, incident post-mortems, regulatory filings. An AI Risk Engineer usually owns the *governance artefacts*; you own the *stack that makes the artefacts defensible*.

Hand off:

- **Threat model (Chapter 7 output)** — structured document of threats × mitigations. Feeds the risk register.
- **DPIA technical inputs** — system diagram, data flows, sub-processor list, retention schedules.
- **Incident evidence** — immutable consent records, call recordings, audit logs.

Consume from them:

- Approved risk register entries you must address before launch.
- Fundamental Rights Impact Assessment (FRIA) outputs that constrain the agent's deployment scope.
- Bias-audit findings that may invalidate certain feature choices (e.g., gender-routing).

### `security-learning` and `ai-infra-security` (Security family, L35)

This is the platform-scale MLSec slice: securing training pipelines, model-serving infrastructure, cross-tenant isolation at the fleet level, secret management across model endpoints, supply-chain attestation of model weights.

Hand off:

- **Voice-specific threat items** that intersect with the fleet (audio-domain adversarials, cross-tenant memory leakage in a shared LLM runtime, model-theft detection across many customers).
- **Component inventory** — exact STT/TTS/LLM provider versions, with provenance.

Consume from them:

- Platform-level controls (KMS, VPC, network policy, OPA / Cedar authz on tool invocations).
- Red-team findings from fleet-wide exercises you may not run yourself.
- Supply-chain policies (which models and which versions are approved).

### `agentic-safety-engineer` (L40)

This is the frontier dangerous-capability eval slice: pre-deployment safety evaluations for highly-capable models, catastrophic-risk scenarios, autonomous-action tripwires. For most production voice agents you will not interact with this role; a voice agent is *not* a frontier system. But two scenarios bring it in:

- You are integrating a frontier general-purpose model (next-generation Realtime / Live) where dangerous-capability evals live upstream and you deploy with those evals as a binding input.
- Your voice agent crosses an autonomy threshold (long-horizon, multi-call persistent memory, high-value autonomous action) that pulls it into agentic-safety scope.

Hand off:

- Autonomy diagrams and tool-permission matrices.
- Any evidence the agent can take high-stakes irreversible actions without human review.

### DPO / legal / compliance counsel

Not an engineering role but the entity that turns the engineer-shaped artefacts into legally-defensible positions. Hand off the compliance envelope (Chapters 1–5 content filled in for your specific deployment). Consume their final sign-off on consent text, prompt openings, disclosure phrasing, and retention windows.

### `nlp-engineer` and `deep-learning-engineer` (adjacent IC roles)

Research-shaped work: training a better anti-spoof model on your data, adversarial training against audio-domain attacks, fine-tuning a speaker-verification model on your tenant distribution. You consume models these roles produce; they consume evaluation harnesses your module 112 work produces.

## The boundaries this module does NOT cover

- **Legal interpretation.** Statutory analysis is counsel's. You brief; they decide.
- **Policy authoring.** The company's AI Acceptable Use Policy lives in legal + security + comms. You implement against it.
- **Red-team operations at fleet scale.** Pentesting campaigns across customers are an ai-infra-security function.
- **Model-side alignment research.** Fine-tuning for refusal behaviour, constitutional AI, RLHF — those are nlp-engineer / deep-learning-engineer work.
- **Frontier-model release gating.** If a new model's dangerous-capability evals have not cleared, that is agentic-safety-engineer's call, not yours.

## The deliverables this module *does* own

Everything here lives in the voice stack and ships with your release:

1. **The outbound-consent flow** (Chapter 1) — consent record schema, UI, revocation pipeline, dialler gates.
2. **The recording-consent plumbing** (Chapter 2) — disclosure prompt, pause primitive, jurisdictional policy table.
3. **The EU AI Act integration points** (Chapter 3) — Article 50 opening disclosure, synthetic-audio watermark, human-oversight breakpoints.
4. **The GDPR plumbing** (Chapter 4) — DSAR API, retention clocks, DPIA technical inputs, cross-border region pinning.
5. **The vertical-specific pipeline** (Chapter 5) — BAA-covered providers for HIPAA; descoped payment flow for PCI; mandatory-disclosure encoding for Reg Z / FDCPA / FINRA.
6. **The provenance pipeline** (Chapter 6) — AudioSeal, C2PA manifest signer, inbound spoof-detection chain.
7. **The threat model and red-team harness** (Chapter 7) — living document, quarterly review cadence, automated regression tests for known attack classes.

## A one-page compliance checklist for launch

```
[ ] TCPA consent record schema deployed; 4-element written consent captured; revocation pipeline wired.
[ ] Dialler six-gate wired: consent → DNC → RND → hours → attestation → place.
[ ] STIR/SHAKEN A-attestation confirmed; call-labeller registrations active.
[ ] State recording-consent prompt played at call start; pause-recording primitive tested end-to-end.
[ ] BIPA-shape voiceprint collection only if required; separate written consent + retention policy published.
[ ] EU AI Act Article 50 opening disclosure encoded; AudioSeal watermark applied to TTS output; C2PA manifest signer deployed.
[ ] GDPR DSAR endpoint live; retention clocks running; cross-border transfers audited.
[ ] HIPAA: BAA signed with every sub-processor (if PHI in scope).
[ ] PCI: pause-recording verified to gate STT + LLM context; no PAN in training data.
[ ] Threat model (Chapter 7) authored; quarterly review scheduled; red-team suite runs in CI.
[ ] Risk register entries from ai-risk-engineer closed.
[ ] Legal sign-off on consent text, call opening, disclosure phrasing, retention windows.
```

Each item in this list is covered in one of Chapters 1–7. If any is red at launch time, the agent should not ship.

## Summary

- Compliance for a voice agent is a four-role minimum: voice-agent-engineer (you), ai-risk-engineer, ai-infra-security, legal/DPO; sometimes agentic-safety-engineer at the frontier edge.
- You own the voice-stack plumbing: consent records, recording primitives, Article 50 disclosures, provenance stamps, threat model, DSAR API.
- You hand off governance artefacts, threat items, and component inventories to adjacent roles; consume their findings as binding constraints.
- Launch gate: the eleven-item compliance checklist above; each item lives in a prior chapter of this module.

The module ends here. Exercises paired with Chapters 1, 2, 3, 6, and 7 give you hands-on practice with each major deliverable; the labs from the earlier mods (recording in mod-107, evaluation in mod-112) carry the ongoing operation of these obligations.
