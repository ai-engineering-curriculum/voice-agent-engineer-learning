# exercise-05: Ship a Latency Budget CI Gate

**Estimated effort:** 2 hours · **Output:** a working CI gate + a runtime fallback

## Objective

Convert the latency budget from a design document into an enforcement mechanism. By the end you will have (a) a synthetic-caller harness that drives representative voice-agent traffic in CI, (b) a budget-gate check that fails the build when p95 turn gap regresses, and (c) at least one runtime fallback that activates when a leg breaches in production.

If the budget cannot fail the build, the budget does not exist.

## Prerequisites

- [Chapter 5 — Enforcing the Budget as an Engineering Gate](../05-enforcing-the-budget-as-a-gate.md).
- Exercise-02's instrumentation (per-turn metrics).
- Exercise-03's streaming graph (otherwise your "p95 turn gap" is dominated by a problem you already know about).
- A CI system you can modify: GitHub Actions, GitLab CI, Buildkite, CircleCI — any works.

## Steps

### Part A — Synthetic caller harness

1. **Build a small scenario library.** At minimum, 10 scenarios covering: a short confirmation turn, a medium informational turn, a long multi-sentence turn, a turn with a named entity (a date, phone number, or proper noun), a turn with a barge-in, a turn with a tool call (if your agent has tools), and three domain-specific turns relevant to your target product. Each scenario is a recorded human-audio file (your own voice is fine) plus an expected-behaviour spec (what the agent should say or do).

2. **Build the driver.** A small program that:
   - Connects to your voice agent as a client (WebRTC, WebSocket, or the native transport your agent exposes).
   - Plays the scenario audio.
   - Records the agent's output audio with timestamps.
   - Pulls per-turn metrics from your trace backend (the one you wired in exercise-02) or from a sidecar metric sink.

3. **Introduce realistic degradation on a subset of runs.** Apply a telephony codec (Opus at low bitrate or G.711) to half the audio and simulate packet loss (1–5%) on another quarter. `sox`, `ffmpeg`, and `tc`/`netem` are all usable; your framework may have built-in hooks.

### Part B — Budget gate

4. **Define the committed budget.** Write a `budget.yaml` (or equivalent) that records:
   - The p95 turn-gap target (your product target from exercise-01).
   - The p99 turn-gap target (default: 1.5× p95).
   - Per-leg p95 budgets from exercise-01.

5. **Run the synthetic suite.** Script a CI job that runs the full scenario library against a staging deployment and collects per-turn metrics into a CSV or JSON.

6. **Implement the gate check.** A short script that:
   - Computes p50 / p95 / p99 turn gap and per-leg p95 across the run.
   - Compares against `budget.yaml`.
   - **Fails the build** if p95 turn gap exceeds the committed target by more than a tolerance (e.g., 5%).
   - **Fails the build** if any single leg's p95 exceeds its committed budget by more than a wider tolerance (e.g., 15%).
   - Emits a failure message that includes the per-leg breakdown of where the regression lives.

7. **Wire the gate into CI.** Fail the build on gate failure. Post a PR comment or a job summary with the per-leg breakdown so the engineer who regressed it can read it without hunting through logs.

### Part C — Runtime fallback

8. **Pick one fallback from Chapter 5.** The simplest is TTS-voice downgrade on TTS-TTFB breach. If your product has tiered LLMs, a smaller-LLM fallback is a stronger choice. Pick exactly one — do not try to ship all three in this exercise.

9. **Implement the fallback.** The fallback should:
   - Observe the breaching metric on a rolling window (the last N turns, not the last one).
   - Switch the configuration for *new turns* — do not swap mid-utterance.
   - Emit a span event or log line indicating the fallback activated, with the metric value that triggered it.
   - Revert when the metric recovers for a sustained window.

10. **Exercise the fallback.** Deliberately break the primary path (slow down the TTS by inserting a sleep in a middleware, or throttle via a provider's rate limit, or any other reproducible method) and verify that the fallback activates, the subsequent turns use the fallback, and the turn gap stays within budget at the cost of the quality degradation you chose.

### Part D — Documentation

11. **Write a one-page operational doc.** Cover:
    - What the gate is, what it checks, how to read a failure.
    - Where the budget lives (`budget.yaml`) and how to update it (the quarterly ritual from Chapter 5).
    - What runtime fallbacks are active, how they activate, and what the user-facing symptom is when each fires.
    - What the gate does *not* catch (output-quality regressions, cost regressions, specific provider-side issues) and where those are covered (mod-112).

## Starter guidance

- **The gate is a product policy, not an engineering preference.** Get sign-off on the tolerance (`5%` vs. `10%` on p95) from whoever owns the product target.
- **Do not make the gate flaky.** A gate that fails randomly will be ignored or disabled. If your staging environment has too much variance to support a tight gate, loosen the tolerance *or* increase the scenario count *or* raise the environment's quality — don't let a flaky gate live.
- **Measure runtime fallback cost.** The fallback's activation rate is a signal; if it's always on, the fallback has become the primary path and your budget is a fiction.
- **Keep fallbacks observable.** `fallback.active=true` as a dashboard tile; `fallback.tier=2` as a span attribute. If the fallback is a secret, nobody will investigate why it fired.

## Acceptance criteria

- [ ] Synthetic scenario library with at least 10 scenarios covering the categories listed above.
- [ ] Driver that connects, plays scenario audio, and captures per-turn metrics.
- [ ] Codec and packet-loss injection on a subset of runs.
- [ ] `budget.yaml` (or equivalent) with p95 / p99 turn-gap targets and per-leg budgets.
- [ ] CI job that runs the synthetic suite and executes the gate check.
- [ ] Gate blocks the build on p95-turn-gap regression beyond tolerance and emits a per-leg breakdown.
- [ ] One runtime fallback implemented, observable, and tested against a reproducible breach.
- [ ] One-page operational doc.

## Stretch goals

- **Continuous staging.** Run the synthetic fleet in staging 24/7 and emit the per-leg metrics to the production dashboards. Track the staging-vs-production delta as its own signal.
- **Multiple fallback tiers.** Implement a second fallback tier (e.g., both a TTS voice and a smaller LLM) and a decision policy for when each is activated. Document how the two interact.
- **Load-shedding mode.** Implement the third pattern from Chapter 5: reject new calls when a capacity signal (concurrent sessions, provider rate-limit proximity) crosses a threshold, with a graceful "please try again" response.
- **Gate on cost-per-turn.** Alongside latency, gate on provider cost per completed turn. A latency fix that triples cost is still a regression.

## Keep it

The gate and the fallback are the operational deliverables for a shipping voice-agent product. Keep them wired into your CI and your staging; they are the strongest signal you have between "designed to hit the budget" and "actually hits the budget".
