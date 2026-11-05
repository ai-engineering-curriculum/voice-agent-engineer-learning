# exercise-02: Role Scope and Deferral Contract Diagram

**Estimated effort:** 2 hours · **Output:** one scope diagram + one filled-out deferral contract

## Objective

Make the L35 scope of the voice-agent engineer concrete. Produce (a) a scope diagram showing what this role owns across the eight voice-feature-lifecycle deliverables (Chapter 5) and where the lines touch other roles on the ladder (Chapter 2), and (b) a filled-out deferral contract (Chapter 4) for a concrete escalation scenario.

The point is to practise drawing lines in ink, not in pencil.

## Prerequisites

- [Chapter 2: Position on the Ladder](../02-position-on-the-ladder.md).
- [Chapter 4: The Generalist Stance and Escalation](../04-generalist-stance-and-escalation.md).
- [Chapter 5: Deliverables Across the Voice-Feature Life Cycle](../05-deliverables-across-the-voice-feature-life-cycle.md).

## Steps

### Part A — Scope diagram (60 minutes)

1. Draw a 2-D grid:
   - **Rows** — the eight voice-feature-lifecycle deliverables (architecture brief, latency budget, pipeline design, telephony integration doc, safety / compliance review, eval bundle, launch-gate decision, operations runbook).
   - **Columns** — the roles from Chapter 2 that touch this work (`voice-agent-engineer` (you), `applied-ai-engineer`, `agentic-ai-engineer`, `ai-eval-engineer`, `forward-deployed-engineer`, `ai-infra-inference-engineer`, `senior-agentic-ai-engineer`, `ai-engineering-manager`). You may add `agentic-safety-engineer`, `training-pipeline-engineer`, or `security-learning` if your scenario calls for it.

2. Fill each cell with **one of**: `O` (owns), `C` (contributes), `R` (reviews), `I` (informed), or blank. Voice-agent-engineer should be `O` on most rows, with other roles contributing in known patterns. (This is a RACI-style matrix; use your own letters if preferred.)

3. **Annotate** the three cells where your organisation (real or imagined) is most likely to get the boundary wrong, and write one line explaining the failure mode.

### Part B — Deferral contract (45 minutes)

4. Pick **one** of the five escalation scenarios from Chapter 4 (tail-latency, judge methodology, ASR domain adaptation, audio-adversarial, dangerous-capability).

5. Write the deferral contract — the table in Chapter 4 — as if you were about to send it today. Fill all six fields: problem statement, what you've tried, minimal reproducer, budget / threshold, what stays yours, what's theirs.

6. Make the scenario **specific**. Concrete enough that a peer reading only the contract could start work:
   - A real product (yours, a former employer's, or a plausible fictional one — but *concrete*).
   - A real metric that has moved (even if invented numbers — label them as such).
   - A reproducer that another engineer could run or inspect.

### Part C — Peer review loop (15 minutes)

7. Trade the scope diagram and the deferral contract with a peer. Ask them to find:
   - One cell in the scope diagram that they would move from one role to another, and why.
   - One field in the deferral contract that is too vague to action.

## Starter guidance

- The eval bundle row is almost always contested between `voice-agent-engineer` and `ai-eval-engineer`; the honest answer is "you own voice-specific dimensions, they own methodology" — but turn that into cell letters.
- The telephony integration doc is contested between `voice-agent-engineer` and (where applicable) a dedicated telephony specialist in a contact-centre org. State the local answer.
- For the deferral contract, prefer a scenario close to one you've seen in reality — concrete details beat polish.

## Acceptance criteria

- [ ] A 2-D scope diagram covering **all eight** deliverables and at least **six** roles.
- [ ] Each voice-agent-engineer cell is labelled (owner, contributor, reviewer, informed) with a reason that is not just "because it's on the role charter".
- [ ] Three annotated cells where the boundary is at risk of being mis-drawn, each with a one-line failure mode.
- [ ] A filled-out deferral contract with all six fields completed and specific enough to action.
- [ ] A peer-review note identifying one cell to move and one field to tighten.

## Stretch goals

- Add a row for "incident response": who owns the Sev-1 on a dropped-call storm? A fraudulent voice-cloning complaint? A clinical-dictation mistranscription? Draw the escalation chain explicitly.
- Produce a second deferral contract for the escalation scenario you find hardest to call — then explain why it's hardest.

## Keep it

The scope diagram is a direct input into Module 112's observability ownership matrix and the project-104 architectural-brief capstone. The deferral contract template is reused in every subsequent module's lab wrap-up.
