# 05. Online Eval — Sampled Judge Scores, Implicit Feedback, Drift Detection, and Fixture Back-Feed

## Why this chapter

The CI gate from Chapter 4 proves a release did not regress against a *fixed* eval set. The fixed eval set is frozen; production is not. New customer segments onboard, product prompts get edited, vendor models get silently updated, acoustic conditions shift when callers move indoors in winter. The release-assurance packet goes stale the minute it ships. The online eval is how you keep watching, after the gate has been passed.

This chapter covers four things: sampled judge-as-LLM on live traffic, the implicit user-feedback signals that proxy for quality when a judge is too expensive, drift detection on the metrics the earlier chapters produced, and the regression-fixture back-feed that turns every incident into a permanent eval-set entry so the same bug cannot regress twice.

## Sampled judge-as-LLM on production traffic

The Chapter 4 judges are already written. Running them on 100% of production calls is possible but wasteful — judge-as-LLM costs scale linearly with traffic. The right pattern is **stratified sampling**: sample a known fraction of each stratum so no segment is invisible, then run the judge on the sample.

Stratification for sampling is not the same as stratification for reporting. For sampling you want to over-represent the rare and risky — low-confidence turns, escalated calls, long calls, new-customer segments — so the judge signal is strong where it matters. The pattern:

- **Base sample rate.** 1%–5% of calls for high-volume products. This is the "background rate" that keeps the overall score updating.
- **Boost rare segments.** Over-sample new flows, new locales, high-stakes verticals (healthcare, finance), and any flow where the deployment is still early.
- **Trigger-based sampling.** Any call where the ASR confidence dropped below a threshold, the user said "agent" / "representative" / "human", the latency SLO was violated, or an escalation happened — always sample. Those are the calls most likely to be scoreable.
- **Weight on aggregation.** Each sampled call has a weight equal to `1 / sample_probability`. When you compute the overall mean, apply weights so the top-line number reflects traffic composition, not sample composition.

Report the **per-stratum online score** (not just the overall), with the **effective sample count** (`sum(weights)^2 / sum(weights^2)`) so a reader can tell that the "bank-fraud flow at 2am" number is from eight calls, not eight thousand.

The judge prompts, rubrics, and calibration from Chapter 4 reuse here. The transcript source is the production ASR (there is no gold reference in online eval), and the usual ASR-confusion caveat applies — flag low-confidence transcripts into a separate review queue instead of trusting the judge on noisy input.

### Privacy and consent

Online eval reads production call transcripts. That is in scope for the mod-108 consent record. Some constraints that commonly bite:

- **Judge model geography.** If you process EU calls with a judge model served from a US region, you are doing an international transfer. Confirm with legal / DPO.
- **Human-review queues.** Anything you escalate for human review is a secondary processing purpose. Make sure your consent and privacy notice cover it, or use a separate consent flow.
- **Retention.** Your retention clock (mod-108 Chapter 4) applies to the judge-processed transcripts too. Do not keep judge outputs longer than the underlying transcript.

Document the online-eval data flow in the mod-108 DPIA input. "We run an LLM-as-judge on 3% of traffic, scoring policy compliance, with outputs retained for 30 days" is the level of specificity the DPIA needs.

## Implicit user-feedback signals

Judge-as-LLM is expensive. Implicit feedback — behavioural signals that the user leaked about their experience — is cheap and covers 100% of traffic. The ones that work for voice:

- **Call abandonment.** Caller hangs up before the flow completes. Report by call phase; abandonment in the opening is a different failure than abandonment in resolution.
- **Repeated question.** The user restates the same question within a short window. ASR + semantic-dedup on user turns gives you this signal. A high repeated-question rate means the agent's answer was unclear or the TTS was unintelligible.
- **Hold time / silence.** Long silence from the user after an agent turn. Can mean "confused", can mean "writing it down"; the distribution shift matters more than any single call.
- **Escalation to human.** The user asked for a human (or triggered the deterministic escalation rules). Rate and distribution per flow, per segment.
- **Post-call CSAT / NPS.** If you have a post-call survey (SMS, email, IVR), the response rate is low but the signal is direct. Join to the call ID; segment by same strata.
- **Customer-service follow-up.** The caller came back within N days with a related issue. Harder to measure (requires tying calls to CRM records), but the single strongest proxy for "we failed them".

None of these is a direct quality metric. They are **correlates** of quality that you watch as a bundle. A regression across multiple implicit signals at once is a much stronger signal than any one alone.

A reasonable dashboard surfaces six to eight implicit signals, each with p50 / p95 / trend, each stratified. The judge score is the top-line; the implicit signals are the explanatory variables. See the Dialpad-and-similar public writing on contact-centre KPIs for the shape of the industry dashboards; the exact set of signals is product-specific.

### Instrumenting without poisoning

Two traps to avoid:

- **Treating implicit signals as labels.** Repeated-question is a weak proxy, not a label; training a judge model on "call is bad if user repeats" produces a biased judge. Use implicit signals for monitoring and routing, not as ground truth.
- **Over-fitting to the signal.** If the only metric that goes to the business is "deflection rate", an agent that *refuses to escalate* will optimise it. Metric design is adversarial; stress-test the KPI by asking "what cheap move would game this?" and instrument the countermeasure.

## Drift detection on WER, MOS, and latency

The release-assurance packet is a snapshot. Drift is the slow movement from that snapshot — a 0.5-point WER rise over three weeks, a 50 ms p95 latency creep, a seasonal MOS shift when new acoustic conditions enter the distribution.

Three drift detectors that cover the common failure modes:

### Univariate statistical control

For each key metric (WER on a canary set, DNSMOS trend, mouth-to-mouth p95, deflection rate):

- **Rolling window.** Compare the last N hours or N calls against the last 2N or 7N. Report the delta and a bootstrap-derived CI on the delta.
- **Cumulative sum (CUSUM) chart.** Classic SPC technique — accumulates signed deviations from the target; alerts on the cumulative exceeding a threshold. See [Page, "Continuous Inspection Schemes", Biometrika 1954](https://doi.org/10.1093/biomet/41.1-2.100) for the original; many stats packages implement it.
- **Exponentially-weighted moving average (EWMA).** Smooths the metric and alerts on moving-average excursions.

Univariate detectors catch "WER went up"; they do not catch "WER stayed the same but the input distribution shifted". For that, you need distribution drift detection.

### Distribution drift — on inputs and on outputs

A voice-agent's inputs change: new accents, new call topics, new background acoustic environments. If the input distribution drifts beyond what the eval set covers, the eval set is no longer representative and the CI gate's guarantee weakens.

Standard tools for distribution drift on features (acoustic SNR estimate, codec distribution, call-length distribution, topic-cluster distribution):

- **Kolmogorov–Smirnov two-sample test** for continuous features.
- **Chi-squared or Jensen–Shannon divergence** for categorical features.
- **Maximum Mean Discrepancy (MMD)** on learned embeddings for higher-dimensional features (e.g., audio embeddings, text embeddings of user turns).

Evidently AI's [Evidently](https://github.com/evidentlyai/evidently) and Alibi Detect ([`SeldonIO/alibi-detect`](https://github.com/SeldonIO/alibi-detect)) are open-source reference implementations. Pick one, encode the drift tests, run them on the same cadence as the metrics.

Output-side drift matters too: if the agent's distribution of turn lengths, tool-call types, or disclosed-topics shifts abruptly, something upstream of you changed (new prompt, vendor model version flip). Watch outputs the same way.

### Alert design

Drift alerts need to be actionable, or they get muted. Rules that work:

- **Tier by business impact.** A 0.3-point WER drift in `bank-fraud` flow alerts the on-call; a 0.3-point WER drift in `product-info` flow goes to a weekly review.
- **Correlate before paging.** Fuse the signals (latency *and* repeated-question up, WER flat) — a single-signal page is often a false alarm; a correlated page is almost always real.
- **Require a run-book entry.** An alert without a documented first step for the on-call is a muted alert in waiting. The run-book links to the incident fixture capture (next section).

## Regression-fixture back-feed from every incident

Every production incident is evidence of an eval-set gap: the CI gate should have caught the regression and didn't. The regression-fixture back-feed is the discipline that closes the gap.

The procedure:

1. **Incident is declared.** On-call captures the conversation, the trace, the audio.
2. **Reproduce in the eval harness.** Add the input (audio + context) to a `regressions/` folder in the eval set. Confirm the current build fails the expected behaviour.
3. **Author the expected behaviour.** The test case has an expected output — reference transcript, mandatory policy adherence, latency budget, whatever the incident required. Review it with the ops / policy owner.
4. **Fix the regression.** Code / prompt / data change that makes the test pass.
5. **Leave the test in place forever.** Even after the fix, the test stays in the eval set so the same bug cannot regress. A fixture only comes out of the eval set with a documented reason (feature removed, flow retired, duplicate case).

Two consequences:

- **The eval set grows over time.** This is good; it tracks the real failure modes of your product. CI runtime grows with it — plan for budget expansion.
- **The eval set gets opinionated.** Fixtures encode operational decisions: "in this case we must re-read the disclosure", "in this case we must escalate", "in this case the TTFA ceiling is 400 ms because the caller was in an IVR with a hard timeout". Treat the eval-set changes as product changes, with the same review discipline as code.

A discipline worth adopting: write an **incident eval digest** per incident. One paragraph: "incident, cause, test that would have caught it, test we added". Attach it to the incident retrospective. Over time the digest file is a roadmap for the eval harness and a very powerful artefact when the next vendor review asks "why do you trust your eval?"

## A reference data flow

```
Production traffic
   │
   ├── 100%: transport, WebRTC stats, ASR confidence, timing spans → OTel (Ch 6)
   │
   ├── sample(stratified, triggered): transcripts → judge-as-LLM → scores
   │
   └── 100%: implicit signals (abandon, repeat, escalate, CSAT) → metrics

OTel + judge scores + implicit signals
   │
   ├── Rolling / CUSUM / EWMA controllers → drift alerts
   │
   ├── Distribution drift (KS, MMD) on inputs and outputs → drift alerts
   │
   └── Dashboards (per-stratum, per-flow, per-cohort)

Incident
   │
   ├── Capture audio + trace + expected behaviour → regressions/
   │
   └── Fix → add test → test stays forever
```

Every arrow here is observable on the same pane of glass; Chapter 6 is how you build that pane.

## A note on A/B rollouts

Online eval is the measurement layer for safe rollouts. Any change that passes the CI gate goes out behind a feature flag or routing weight, not to 100% traffic.

- **Shadow.** New build receives traffic in parallel but does not respond. Measure on the shadow's outputs; no customer impact. Best for infra / model changes with no behavioural surface.
- **Canary.** New build gets 1%, 5%, 25%, 100% over hours or days. Measure on the canary cohort; compare to the control cohort. Statistically-significant regression on the canary cohort rolls back.
- **Regional canary.** New build goes to one region first; other regions carry the control. Preserves per-region independence; useful when latency / network / vendor region matters.

The eval metrics you watch during rollout are the same as online eval; the comparison is canary-vs-control rather than now-vs-last-week. The rollback trigger is pre-committed: a per-metric threshold (p95 regression > X, policy compliance < Y) automatically rolls back without a human in the loop. Human-override rollback is also allowed; both must be instrumented.

## Summary

- **Sample judge-as-LLM at a known rate** — stratified, with trigger-based oversampling on risky segments; weight on aggregation so the top-line reflects traffic composition.
- **Watch implicit feedback signals** as a bundle: abandonment, repeated-question, hold time, escalation, CSAT. None is a label; correlated regressions are the actionable signal.
- **Detect drift** on univariate metrics (rolling delta, CUSUM, EWMA) and on input/output distributions (KS, Jensen–Shannon, MMD). Alert when business impact is high; correlate before paging.
- **Back-feed every incident into the regression fixture set.** The eval set grows; so does your confidence. A test only comes out with a documented reason.
- **Roll out behind flags** — shadow / canary / regional canary — with pre-committed metric thresholds for automatic rollback.

Next chapter: instrumenting the whole system with OpenTelemetry GenAI semantic conventions and voice-specific attributes, and wiring the data into Langfuse / LangSmith / Arize Phoenix / Weave / Braintrust, with WebRTC stats on the same plane.
