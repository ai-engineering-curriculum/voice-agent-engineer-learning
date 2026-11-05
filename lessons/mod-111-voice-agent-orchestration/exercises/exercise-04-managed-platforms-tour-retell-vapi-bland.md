# exercise-04: Managed Platforms Tour — Retell, Vapi, Bland

**Estimated effort:** 2 hours · **Output:** a filled build-vs-buy checklist plus a one-page recommendation

## Objective

Spin up the same simple voice agent (a small appointment-booking or customer-support bot — your pick) on at least two managed voice-agent platforms. Fill in the build-vs-buy checklist from Chapter 4 using your own measurements and findings. Produce a written recommendation that compares the managed options against your own-pipeline builds from exercises 01 and 02.

This is primarily a *measurement and procurement* exercise. You will spend more time filling in the checklist than you will writing code.

## Prerequisites

- [Chapter 4 — Managed Voice-Agent Platforms: The Survey](../04-managed-platforms-survey.md).
- Completed exercise-01 and exercise-02 — the baseline cost / latency / control numbers for your own stack.
- Trial accounts on at least two of: [Retell AI](https://www.retellai.com/), [Vapi](https://vapi.ai/), [Bland AI](https://www.bland.ai/). (Many offer free credit.)
- A fixed scenario — a 5-turn scripted conversation (user says X, bot does Y, tool call, user says Z, bot wraps up). Write this down before touching any platform.

## Steps

1. **Pin the scenario.** On one page: the use case (appointment booking, inbound support, outbound qualification), the opening message, the three tools the bot can call (give each a mock implementation), the expected conversation length, the expected volume at steady state, and the compliance posture inherited from mod-108.

2. **Spin up the agent on platform A.** For each of Retell, Vapi, or Bland that you selected:
   - Create the agent/assistant via UI or API — use the API path if you want reproducibility; the UI path is faster for a first pass.
   - Wire the three tools (webhook or function definitions per the platform's docs).
   - Configure voice, STT engine, LLM, interrupt sensitivity, timeouts.
   - Record what you could configure and what you couldn't.

3. **Repeat on platform B.** Same scenario, same tools, same voice/style if the platform allows it.

4. **Run the scripted conversation on both platforms.** Drive 10 conversations through each platform (via their test console, a demo call button, or a scripted SIP driver — whatever each platform offers). Record:
   - p50 and p95 end-to-end turn gap (as perceived by the client).
   - Interrupt behaviour (does the bot stop fast when you talk over it?).
   - Tool-call latency as seen by the user (the time from user utterance to the bot's response that uses the tool result).
   - Whether the bot emitted markdown, unspoken punctuation, or any TTS artefacts.
   - Any errors or retries the platform surfaced.

5. **Compliance walkthrough per platform.** For each: locate the DPA, BAA (if relevant to your posture), PCI posture, data-residency options, model-training opt-out, subprocessor list, retention / deletion API. Note which items are available immediately, which require a sales call, and which are not offered. Date this.

6. **Cost walkthrough per platform.** Compute the 12-month cost at your expected volume. Separate the platform per-minute rate from the LLM/STT/TTS passthrough costs. Compare to the 12-month cost of the own-pipeline from exercise-01 / exercise-02 (fully loaded: engineering time to operate, infra bill, your sub-vendor contracts).

7. **Fill in the Chapter 4 checklist** with one column per platform plus a column for your own-pipeline build.

8. **Write the recommendation.** One page. Opening line is the answer ("We recommend own-pipeline on LiveKit Agents" / "We recommend Vapi for inbound + own-pipeline for outbound" / etc.). The paragraphs that follow tie each row of the checklist to that answer. Pre-commit the exit conditions that would reopen the decision (Chapter 8).

## Starter guidance

- **Do not drift the scenario between platforms.** If you reconfigure the bot differently on each one, you're not comparing platforms, you're comparing your taste in prompts.
- **Measure in the same region.** All platforms perform worse cross-region; a cross-region tail is a region story, not a platform story.
- **Confirm the compliance claims in writing.** A platform's marketing page is not the DPA. If you cannot find a signed document, mark that row as "not demonstrated" — not as "yes".
- **Do not pay for an enterprise plan to complete this exercise.** The free / startup tier is enough to produce defensible measurements. Where a feature is enterprise-only, note that it is enterprise-only in the checklist.
- **The comparison is only valid at today's date.** Date every row. In three months, half the numbers will be stale.

## Acceptance criteria

- [ ] A pinned scenario document (one page) that both platforms (and the own-pipeline baseline) implement identically.
- [ ] Working agents on at least two of Retell, Vapi, Bland, each driving the scripted conversation successfully.
- [ ] A completed Chapter 4 checklist with dated numbers across cost, latency, control, compliance, and in-house ML lanes.
- [ ] Interrupt behaviour measured (subjective + a rough tail-stop estimate) per platform.
- [ ] A one-page recommendation with the chosen archetype (or hybrid), tied to specific checklist rows.
- [ ] Pre-committed exit conditions for re-opening the decision.

## Stretch goals

- **Add a contact-centre AI (Observe.AI or Phonely) to the comparison.** These usually require a sales call to try; email them and note the procurement timeline as a datapoint.
- **Fault-injection test.** Deliberately introduce a tool failure (return a 500 to the webhook) on each platform. Does the bot handle it gracefully? Does the user hear a reasonable sentence or a "I'm having trouble" loop?
- **Volume sensitivity analysis.** Re-compute the 12-month cost at 0.5× and 2× your volume forecast. Does the archetype choice change? At what volume does it flip?
- **Latency across tools.** Add a deliberately slow mock tool (3 seconds). Measure how each platform handles it — does it speak a filler? Does it go silent? Does it timeout?

## Keep it

The checklist and the recommendation are the actual artefacts of a build-vs-buy decision. Keep them in a form you would hand to a reviewer without reformatting. The recommendation is the input to Chapter 8's end-to-end defence.
