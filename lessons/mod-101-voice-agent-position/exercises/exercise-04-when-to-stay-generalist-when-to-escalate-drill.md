# exercise-04: When to Stay Generalist, When to Escalate — Drill

**Estimated effort:** 2 hours · **Output:** a decision log over six vignettes + one written reflection

## Objective

Rehearse the generalist / escalation call (Chapter 4) on six voice-engineering vignettes. For each one, write the decision — own it or escalate — and the reasoning, under the same time pressure you would face in a weekly team review.

By the end you should have an internal reflex, not a lookup table.

## Prerequisites

- [Chapter 4: The Generalist Stance and Escalation](../04-generalist-stance-and-escalation.md).
- Ideally [Exercise 02](exercise-02-role-scope-and-deferral-contract-diagram.md) so the deferral-contract shape is fresh.

## Format

For each vignette:

1. **Set a timer for 10 minutes.** No research, no reading outside the chapter. First instinct, then sanity-check.
2. Write a short decision log entry with four fields:
   - **Call** — "own" or "escalate to <role>".
   - **Why** — two to three sentences.
   - **First action** — what you do in the next hour.
   - **Signal to re-decide** — the measurement or event that would flip your call.
3. Move on. You get ten minutes, including writing.

Then — after all six — spend 30 minutes revisiting and revising, and write a 300-word reflection on which calls you'd now revise and why.

## The vignettes

### Vignette 1 — Tail-latency mystery

Your end-to-end P50 response time is 780 ms, well under your 1.1 s budget. But P95 is at 2.4 s and your users are complaining. You traced a sample of slow calls and the LLM provider's TTFT is clearly bimodal — most are fast, a long tail is slow. Your traces are clean; the orchestration code is not implicated.

### Vignette 2 — Weird eval results

Your voice eval harness uses an LLM-as-judge on transcripts. On the last three weeks of runs, the judge prefers the configuration with the slowest TTS voice. You think the judge is biased by response length (the slower voice synthesises longer transcripts because of its pacing), but you aren't sure. Your eval-engineer peer is on vacation this week.

### Vignette 3 — Medical-term accuracy

A clinical customer's pilot fails because the ASR misrecognises paediatric cardiology drug names. You've tried the provider's word-boost feature; it reduces the error rate but not to clinical threshold. The provider supports domain fine-tuning via an API, but you've never trained an ASR model.

### Vignette 4 — Deepfake complaint

A customer reports that a scammer called their elderly parent using what sounded like your voice-agent product's distinctive voice, claiming to be from the customer's bank. You have no evidence your product was used — the voice could have been cloned externally — but the customer's legal team wants an answer in 48 hours.

### Vignette 5 — Barge-in regression

Your last release shipped a new VAD implementation. Across your eval set, the agent is now interrupting users roughly twice as often as before. The eval result is clean; you think your silence-threshold constants were tuned for the old VAD and need to be recalibrated. No one else on the team has looked at this yet.

### Vignette 6 — Capability creep

Product wants the voice agent to be able to "explain" medications it pulls from the patient's EHR, including interactions. Your current guardrail refuses any medical advice. Legal is cautiously open. Product is pushing hard because a competitor claims to do this.

## Starter guidance

- For each vignette, resist the temptation to pick a *comfortable* answer. Note it when the first-instinct call feels uncomfortable.
- The "signal to re-decide" field is the most important. A call without a flip-signal is a call you never revisit.

## Acceptance criteria

- [ ] Six decision-log entries, one per vignette, each with the four fields.
- [ ] At least **two** entries labelled "own" and at least **two** labelled "escalate" — if all six are one or the other, you're likely mis-calling.
- [ ] The 300-word reflection names at least one call you'd revise on second reading and explains what changed.
- [ ] No vignette left as "it depends" without a stated condition that resolves it.

## Stretch goals

- Convert one of the "escalate" calls into a filled-out deferral contract (the Chapter 4 table). Compare it with the one you produced in [Exercise 02](exercise-02-role-scope-and-deferral-contract-diagram.md).
- Rewrite one of the vignettes to make the opposite call defensible — what fact would have to change? What does that tell you about the boundary?

## Keep it

The decision-log format and the vignette set reappear in later modules' incident-response drills and in `project-104-speech-native-architectural-brief`, where you make the own-vs-escalate call for an architecture you own.
