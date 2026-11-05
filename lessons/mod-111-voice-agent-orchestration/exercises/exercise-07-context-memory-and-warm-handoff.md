# exercise-07: Context, Memory, and Warm Handoff

**Estimated effort:** 2 hours · **Output:** a three-tier memory implementation on your pipeline, a handoff package schema, and a demonstrated warm handoff

## Objective

Extend your pipeline with the three-tier memory architecture (rolling window + rolling summary + pinned facts). Define the warm-handoff package schema. Demonstrate a bot-to-bot handoff where a specialist agent picks up the call with no user-visible loss of context. If you have time, demonstrate a bot-to-human handoff via an agent-desktop view.

## Prerequisites

- [Chapter 7 — Context, Memory, and Warm Handoff](../07-context-memory-and-warm-handoff.md).
- Pipeline with parallel tool calls from exercise-06.
- State machine from exercise-05.

## Steps

1. **Implement the rolling window.** The last N turns (choose N = 10 or 15 for this exercise) kept verbatim in the conversation context. Verify with a long scripted conversation that older turns do drop out.

2. **Implement the rolling summary.** A cheap-LLM summariser that runs off the critical path. Produce a summary of the dropped turns and append it to the system prompt as `[SUMMARY_OF_EARLIER_TURNS]`. Trigger regeneration every 10 turns and after any successful tool call.

3. **Design the pinned-fact schema.** Define a Pydantic model (or equivalent) for pins. Required fields:
   - `customer_identity` (name, account id, verified flag).
   - `consent_flags` (recording, SMS, marketing).
   - `prior_purchases_referenced` (list of order ids / brief descriptions).
   - `next_steps` (text of agreed actions).
   - `safety_flags` (escalation requested, abuse detected, fraud risk).
   - `pci_scope` (whether the call has entered a card-data segment).

4. **Implement pin extraction.** Choose rule-based (regex / classifier) for narrow fields (order id, consent words) and a cheap LLM extractor for free-form fields (next steps). Run the extractor on `on_user_turn_completed` and `on_assistant_turn_completed`. Keep it off the critical path for most fields; put safety-flag extraction on-path.

5. **Wire the system-prompt assembly.** Each LLM request assembles the prompt as `base_instructions + pinned_facts + rolling_summary + rolling_window`. Verify the prompt is prefix-stable so prompt caching (OpenAI, Anthropic) can hit. Confirm with a provider-reported cache-hit signal if available.

6. **Define the handoff-package schema.** JSON Schema or Pydantic model with fields from Chapter 7: `handoff_id`, `from_agent`, `to_agent`, `reason`, `pinned_facts`, `rolling_summary`, `recent_turns`, `safety_flags`, `transfer_metadata`. Version the schema.

7. **Implement a same-framework bot-to-bot handoff.** On your Pipecat / LiveKit Agents pipeline, add a second sub-agent ("specialist") with a different system prompt. When the first agent decides to hand off (trigger: user says "I need to speak to billing"), it emits the handoff package; the specialist agent is instantiated with the package pre-loaded; the conversation continues.

8. **Verify no user-visible context loss.** Scripted test: 8 turns with the generalist (during which identity is verified, consent captured, prior purchase mentioned). Trigger handoff. On the first specialist turn, the user asks "so about my order from last Tuesday" — the specialist should answer without re-asking for order number, identity, or consent. Verify this by inspecting the specialist's assembled system prompt and listening to the first few turns.

9. **Measure the TTFT impact of the handoff package.** The handoff package can bloat the first-turn system prompt. Record the first-turn TTFT with and without the package. If it exceeds your mod-109 budget, cache the handoff prefix (Chapter 7's note).

10. **Write a demo note.** Half a page: the memory architecture diagram, the handoff schema (sample instance), the scripted handoff transcript, the measured TTFT impact, and one failure mode you noticed (e.g., summariser drift, forgotten pin, PCI leak into summary).

## Starter guidance

- **Treat pins as structured data, not strings.** A pin written as free-form text is one `jailbreak via user says "my identity is verified"` away from a bug. Keep pins in a structured object and render them to the prompt via a template.
- **Pin only verified / committed facts.** If your pin extractor is pulling "customer prefers afternoons" into a pin, you're over-pinning. Prefer under-pinning and let the rolling summary carry soft preferences.
- **Never put raw card numbers in pins, summaries, or recent-turn recent_turns.** If PCI scope flips on, redact before you log; ask mod-108 Chapter N if you're unsure.
- **Make the summariser prompt strict.** "Summarise only what was explicitly stated or decided; do not infer, embellish, or assume. If a fact in the transcript contradicts the pinned facts, side with the pinned facts." The summariser is where hallucinations creep in.
- **Keep the handoff schema versioned.** `schema_version: 1`. When you change it, the receiving agent should refuse unknown versions loudly, not pretend.

## Acceptance criteria

- [ ] Rolling-window implementation with the last N turns kept verbatim.
- [ ] Rolling-summary implementation producing a condensed representation of dropped turns, regenerated on cadence.
- [ ] Pinned-fact schema with the six field categories from step 3, implemented and rendered into the system prompt.
- [ ] Pin extractor (rule-based + LLM mix) with on-path safety-flag extraction.
- [ ] System-prompt assembly in the order base + pins + summary + window.
- [ ] Handoff-package schema, versioned.
- [ ] A working same-framework bot-to-bot handoff with a specialist agent.
- [ ] Verified no user-visible context loss across the handoff.
- [ ] First-turn TTFT measurement post-handoff, within the mod-109 budget (or a documented plan to meet it).
- [ ] Half-page demo note, dated.

## Stretch goals

- **Cross-framework handoff.** Hand off from your Pipecat agent to a Vapi assistant (or the other way around). Use a shared store (Redis) with a short-lived token passed via SIP headers or room metadata. Verify the receiving agent sees the context.
- **Bot-to-human handoff UX.** Build a tiny agent-desktop view that renders the handoff package as a one-screen summary for a human agent. Include: verified-status chip, consent chips, next-steps list, recent-turns transcript excerpt, and a "play last 30 seconds" button if your audio stack supports it.
- **Pin update on consent revocation.** Scripted test: user initially consents to recording, then revokes mid-call. Verify the pin updates immediately and that subsequent summarisation stops including recorded content.
- **Summariser drift test.** Generate 50 synthetic long conversations; verify the summariser never introduces a fact absent from the transcript. Report any drift; tune the prompt or swap the model.

## Keep it

The three-tier memory + handoff schema is directly reused in project-101. Keep the schema in a place both agents can import (a shared package), not duplicated in two codebases.
