# exercise-05: Interrupt-Aware State-Machine Design

**Estimated effort:** 2 hours · **Output:** a written state-machine specification with an event-to-transition table, implemented and verified on one of your pipelines

## Objective

Design the interrupt-aware state machine for your agent explicitly: the four concurrent legs (transport input, brain, mouth, tool), their states, their transitions, and the cancel-and-resume contract. Implement it on the Pipecat or LiveKit Agents pipeline from exercises 01/02. Verify the behaviour with a scripted test harness. Produce a reviewable specification document.

## Prerequisites

- [Chapter 5 — The Interrupt-Aware State Machine](../05-interrupt-aware-state-machine.md).
- A working pipeline from exercise-01 or exercise-02.
- The mod-106 Chapter 6 tail-stop metric you already understand.

## Steps

1. **Write the four leg state machines.** On paper or in a diagram tool. Each leg: states, events, transitions, side effects. Keep each machine small enough that someone else can review it in ~5 minutes.

2. **Write the event-to-transition table.** Use the template in Chapter 5 as a starting point. Fill in the rows for your product's events — including custom ones for your tools, your handoff flow, and your hangup handling.

3. **Decide the cancel contract for tools.** For each tool your agent calls (including any demo tools from exercise-01 or exercise-02): classify as `idempotent_read` or `side_effecting`. Document the cancel behaviour for each class.

4. **Implement the state machine in code.** On your Pipecat or LiveKit Agents pipeline from the previous exercises:
   - Introduce explicit `BrainState`, `MouthState`, `ToolState` enums.
   - Wire the state transitions on the framework's lifecycle hooks (Pipecat frames or LiveKit callbacks).
   - Log every transition with a timestamp.
   - Add a "contract violation" alert — if the pipeline transitions in a way not in your table, log an error.

5. **Write a scripted verification harness.** Drive the pipeline through a sequence of events designed to exercise:
   - A normal turn with no interrupt.
   - An interrupt during BRAIN_THINKING (before any audio played).
   - An interrupt during BRAIN_STREAMING_OUT (mid-sentence).
   - An interrupt during a tool call (both idempotent-read and side-effecting).
   - An interrupt during a filler phrase (anticipating Chapter 6 — you can scaffold this now).
   - A rapid back-to-back interrupt (user interrupts, then interrupts the agent's apology).
   For each scenario, record the state transitions and compare to the specification.

6. **Measure the interrupt tail-stop latency against your specification.** With the new instrumented pipeline, measure `user_speech_started → last_bot_audio_frame_delivered_to_client` p50 and p95. Compare to the values from exercise-02 (if you had them) and to your mod-106 commitment.

7. **Write the specification document.** Suitable for a design review. Structure:
   - The four leg state machines (diagrams or tables).
   - The event-to-transition table.
   - The tool cancel contract (table: tool name → class → cancel behaviour).
   - The verification harness scenarios and the observed pass/fail.
   - The interrupt tail-stop latency measurement.
   - Three known failure modes you have *not* yet handled and what you would do about each.

## Starter guidance

- **Do not try to express the state machine as a single mega-automaton.** Four small concurrent machines are easier to reason about than one 32-state Cartesian-product machine. The composite state is in your head and in logs, not in code.
- **Side-effecting tools are rare but matter.** If your test harness doesn't have at least one side-effecting tool (e.g., `send_sms` or `book_appointment` mocked to log the effect), you'll never exercise the hard case.
- **Log every transition, in production too.** You will thank yourself when the first "the bot cut me off" bug report lands.
- **Treat the contract-violation alert as a regression test.** When it fires in testing, fix the pipeline, not the spec — unless the spec was wrong, in which case update it explicitly.
- **If your framework hides a transition from you, that's a finding.** Document it. For example, Pipecat drains certain queues internally on interrupt; if your log doesn't see those transitions, note it.

## Acceptance criteria

- [ ] Four leg state machines, written and reviewable.
- [ ] A complete event-to-transition table with your product's events.
- [ ] Tool cancel contract documented per tool.
- [ ] Code implementation on your Pipecat or LiveKit Agents pipeline, with transition logging and a contract-violation alert.
- [ ] A verification harness that exercises at least the six scenarios listed in step 5.
- [ ] Interrupt tail-stop latency measurement with the new instrumentation.
- [ ] A specification document, dated, suitable for a design review.
- [ ] Three known failure modes you haven't yet handled, with your proposed remediation for each.

## Stretch goals

- **Multi-sub-agent coordinator.** Add a second sub-agent (e.g., a specialist for a specific intent) and extend the state machine for the sub-agent-handoff transition. Verify that interrupts during a handoff are correctly attributed to the incoming sub-agent.
- **Resumption after interrupt.** When the user interrupts and then says "sorry, keep going", can the agent actually keep going from where it left off? Design the resumption contract — what context is preserved, what is discarded.
- **Chaos test.** Generate random interrupts at random times across a 100-turn scripted conversation. Does any contract violation trigger? Does the apology trap appear?
- **Comparison with managed platforms.** Repeat the verification harness scenarios against a Vapi / Retell agent. Where does the managed platform fail quietly? Document the observations.

## Keep it

The state-machine specification is the design artefact you will cite for every subsequent interrupt-related bug report. Keep it versioned and keep it near the agent code.
