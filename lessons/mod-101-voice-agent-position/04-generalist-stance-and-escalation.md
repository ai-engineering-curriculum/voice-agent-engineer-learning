# 04. The Generalist Stance: When You Own, When You Escalate

## Why this chapter

The voice-agent engineer is a technical generalist across a *narrow but deep* stack: ASR, VAD, LLM, TTS, orchestration, telephony edge, evals, observability. "Generalist" here does not mean "does a bit of everything casually." It means **you own the stack end-to-end in the normal case**, and you know exactly when to stop and bring in a peer specialist.

Mis-calling the generalist boundary has two failure modes, both expensive:

1. **Over-escalating.** You hand off a problem that was yours to solve, the specialist gets interrupted for something out of their scope, and the feedback loop on the voice stack slows to a crawl.
2. **Under-escalating.** You spend three weeks reinventing something a peer could have solved in an afternoon, and the final answer is worse because the peer's whole career is spent on that one thing.

This chapter gives you the decision rule.

## The default stance: you own the loop

Start every new voice feature assuming you own it end-to-end. Specifically, in the normal case you own:

- **Pipeline design** — ASR → VAD → LLM → TTS composition; speech-to-speech vs. cascaded; transport choice.
- **Latency budget** — allocating a target end-to-end response time across the components, and defending it.
- **Telephony / WebRTC edge** — the integration with CPaaS (Twilio, Vonage), SIP trunks, WebRTC rooms (LiveKit, Daily).
- **Voice-specific evals** — streaming WER on your own calls, TTS quality AB, latency distributions, barge-in metrics.
- **Observability** — tracing a single turn from audio-in to audio-out, structured logs per component, error budgets.
- **Safety and compliance review** — recording consent, PII redaction in transcripts, voice cloning consent, dangerous-capability refusal in the dialogue.
- **Runbook** — what oncall does when calls start dropping, when the TTS provider goes down, when the LLM is slow.

The expectation at L35 is that for 80 % of voice-feature work, no specialist peer is needed. You *are* the specialist.

## The five escalation scenarios

The remaining 20 % divides into roughly five recognisable patterns. For each one, learn the shape and the handoff.

### 1. Serving-layer tail latency — escalate to `ai-infra-inference-engineer`

**Shape.** Your end-to-end P50 is fine but P95 and P99 are blowing the budget. You've traced it to time-to-first-token or inter-token latency on a hosted LLM or STT endpoint, not to your orchestration.

**Why you don't fix it yourself.** The inference engineer owns batching, scheduling, speculative decoding, and provider negotiations. If the fix is "move to a different serving tier", "enable prefix caching", "split the Triton deployment", or "renegotiate the SLA", that is their craft and their lever.

**The handoff.**

- Give them a reproducible trace — timestamps per component, request IDs, enough samples to see the tail.
- Give them a budget — "we need P95 TTFT under X ms for this voice loop to feel conversational."
- Keep owning the loop behaviour while they fix the server: if they ship a provider swap, you own revalidating the full voice pipeline on top.

### 2. Judge-methodology question — escalate to `ai-eval-engineer`

**Shape.** You have a voice eval running and the results are suspicious — judges seem to prefer longer responses, scores correlate with voice choice rather than content, or you can't get inter-rater agreement above chance.

**Why you don't fix it yourself.** Calibrating judges, designing against reward hacking, picking the right pairwise vs. absolute methodology, and running power analyses are a whole discipline. You know enough to notice the problem, not enough to redesign the harness.

**The handoff.** Show the eval-engineer the scoring artefacts, sample disagreements, and your hypothesis about what's going wrong. Own the voice-specific dimensions they fold back into the fixed design.

### 3. ASR domain adaptation — escalate to `training-pipeline-engineer`

**Shape.** The provider's off-the-shelf ASR is good at general speech but bad at your domain — medical terminology, legal jargon, code names, brand phonetics, a specific accent your callers have.

Mitigations you can try first: custom vocabulary / word boost at the provider level, in-prompt LM biasing where available, post-hoc correction with an LLM pass. If those are not enough and the provider supports domain fine-tuning (or you have an open-weights model), that is a training job.

**Why you don't fix it yourself.** Datasets, labelling, train/val/test splits, hyperparameter sweeps, catastrophic-forgetting checks — a training pipeline engineer's whole discipline.

**The handoff.** Supply the domain corpus (recordings + transcripts), the current error analysis, and the acceptance threshold. Own the pipeline reintegration and the A/B rollout.

### 4. Audio-adversarial / voice-cloning threat — escalate to the `security-learning` track and related security specialists

**Shape.** Your voice agent is being probed with audio adversarial examples, deepfake voices are impersonating customers against your voice-authenticated flows, or a red team is embedding prompt injections in TTS-rendered audio.

**Why you don't fix it yourself.** Audio adversarial robustness research, deepfake detection, STIR/SHAKEN-based caller attestation, biometric anti-spoofing (ISO/IEC 30107 PAD) are specialist fields. You own the mitigations in the pipeline (add a liveness check, downgrade to human review, disable voice-only authentication for high-risk actions), you do not own the research programme.

**The handoff.** Document the attack surface, expected blast radius, and what you can and cannot change in the pipeline. The security specialist drives threat-model depth; you own the implementation.

### 5. Frontier dangerous-capability concern — escalate to `agentic-safety-engineer`

**Shape.** Someone demonstrates that your voice agent, in combination with the tools you've given it, can be socially engineered into something downstream that looks like a serious safety concern — misleading medical advice at scale, financial fraud assistance, instructions for restricted activities, or convincing impersonation that could enable larger-scale fraud.

**Why you don't fix it yourself.** Deciding where the line is for a capability, writing the organisation-level policy, and running pre-deployment dangerous-capability evaluations are not voice-pipeline work. The agentic-safety engineer owns the programme; you own the per-feature guardrail that enforces its output.

**The handoff.** Minimum artefacts to bring: the capability description, representative transcripts (not just worst-case; include the common case too), the current guardrail, and the business ask that pushed capability expansion in the first place.

## The deferral contract

For each escalation scenario, the shape of the handoff is the same:

| Field | What it says |
|---|---|
| **Problem statement** | One sentence, measurable |
| **What you've tried** | So the specialist doesn't repeat it |
| **Minimal reproducer** | Trace, transcript, dataset snippet |
| **Budget / threshold** | What "fixed" looks like |
| **What stays yours** | Reintegration, evals, rollout |
| **What's theirs** | The root-cause fix, the methodology, the policy |

Writing this contract down at the time of escalation prevents the specialist from feeling dumped on, and prevents you from losing track of a re-emerging problem.

## Warning signs you're mis-calling the boundary

**Signs you're under-escalating:**

- You are reading papers for a problem that has a known specialist in your org.
- The fix you are considering requires changing someone else's artefact (model weights, serving config, policy doc).
- You've been at it for a week with no measurable progress and no plan for a measurable next step.

**Signs you're over-escalating:**

- The problem is "my pipeline latency is bad" and you've not yet traced which component is slow.
- You're asking the specialist to make a product decision (ship / don't-ship, which voice, which prompt).
- You've not written down a minimal reproducer because "they can read my code."

## Summary

- Default stance: the voice-agent engineer owns the voice stack end-to-end.
- Five escalation patterns — serving-layer tail latency, judge methodology, ASR domain adaptation, audio-adversarial threats, dangerous-capability concerns — have known handoff targets.
- The deferral contract (problem, prior attempts, reproducer, budget, what stays yours, what's theirs) is the artefact that keeps the handoff healthy.

Next chapter: the concrete deliverables you own across a voice-feature life cycle.
