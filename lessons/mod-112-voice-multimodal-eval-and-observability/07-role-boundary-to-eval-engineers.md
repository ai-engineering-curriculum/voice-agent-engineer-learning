# 07. Role Boundary — What You Own vs. `ai-eval-engineer` (L30) vs. `ai-evaluation-engineer` (L35)

## Why this chapter

Voice evaluation is not one role's job. This module trains you to build and run the voice-specific evaluation harnesses because those harnesses live inside your product's stack and change on your release cadence. Two adjacent roles are doing related work, and the cleanest way to not duplicate or drop obligations is to name the boundaries up front.

This chapter names three roles and the artefacts that flow between them.

## The three roles

### `voice-agent-engineer` (L35) — this track

You own the voice-specific harnesses: ASR eval (Chapter 1), TTS eval (Chapter 2), conversational eval (Chapter 3), content eval wired to CI (Chapter 4), online eval + drift (Chapter 5), OTel-plus-WebRTC instrumentation (Chapter 6). The deliverables ship with the voice agent; they fail when you push a bad build; they live in your repo.

What's distinctive about your eval work vs. the generic LLM-app eval work:

- The artefacts are **audio-first**. You cannot write an LLM eval for a voice agent without an ASR step and a TTS step; those steps have their own error surfaces.
- The budget is **sub-second**. The content eval has to coexist with the mod-109 latency budget; a judge-as-LLM that would run in 3 s per turn cannot run synchronously on the call path, which changes the design.
- The failure modes are **voice-specific**. Hallucinated TTS phonemes, barge-in races, codec-induced quality drops — these are not in a generic LLM-app eval playbook.

The module owns this work. The adjacent roles support it.

### `ai-eval-engineer` (L30) — Engineering family

This is the deep-eval-methodology slice. A practitioner at this level owns:

- Eval-dataset curation and labelling pipelines (annotation vendor management, IRR, label versioning).
- Judge-as-LLM design and calibration (rubric engineering, bias measurement, calibration against humans).
- Statistical methods for eval reporting (bootstrapping, CIs, effect-size estimation, significance testing).
- Dataset / experiment platforms (Weights & Biases, Langfuse, LangSmith, Arize Phoenix, Braintrust).
- Generic LLM-app eval harnesses that apply across many products in the org.

You consume from them:

- The **house judge-as-LLM patterns** — rubrics, few-shot example conventions, calibration-study templates. Adapt them to your voice-specific dimensions (policy compliance, disclosure re-read, intelligibility).
- **Statistical infrastructure** — the bootstrap library, the significance-test plumbing, the dataset-versioning tool. Do not re-implement these in the voice stack.
- **Judge-model recommendations** — "we use `claude-sonnet-4-6` as the house judge; here is why" is their decision, applied to your rubrics.
- **Calibration data infrastructure** — the pipeline that gets human labels from an annotation vendor, double-annotated, with IRR tracking. You bring voice-specific label tasks to it; they run the pipeline.

You hand to them:

- **Voice-specific eval dimensions** — policy-compliance-with-disclosure-re-read, intelligibility-at-telephony-codec, barge-in-recovery — written as rubrics they can calibrate against humans using the shared pipeline.
- **Your gold-set definitions** — the Chapter 1 stratified gold set schema is theirs to help you version and label.
- **Judge-as-LLM failure cases** from your calibration study, so the house patterns incorporate what breaks on voice transcripts.

The seam is narrow: *you* know what matters in a voice call; *they* know how to turn "what matters" into a reliable grader at the scale of a company's eval surface. Expect to collaborate frequently; expect occasional disagreement about where a rubric belongs (shared vs. voice-specific).

Reference peer role in the AI Engineering curriculum: [`ai-eval-engineer-learning`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning).

### `ai-evaluation-engineer` (L35) — Governance family

This is the **release-assurance** slice, and it is a different role in a different family than the one above. The governance-side AI Evaluation Engineer owns:

- **Model cards and system cards** — the public / regulator-facing description of what the voice agent is, what it was tested on, how it performs, and where it fails.
- **Release-assurance packets** — the signed, versioned document that says "this release passed the following gates on the following evidence". Your Chapter 1–4 outputs feed into this packet; they are not the packet.
- **Pre-release safety and bias assessments** — the regulator-reviewable framing of the stratified results you produce. Where your gold set reports worst-bucket WER, they take that number and write "bias risk assessed; mitigations in place for buckets exceeding the threshold" in a form a DPA will accept.
- **Red-team + dangerous-capability eval coordination** — the governance-side runbook for what happens when a frontier model's capability evals (an `agentic-safety-engineer`'s work, mod-108 Chapter 7) interact with this release.
- **Regulator liaison** — the person who takes the eval evidence to an EU AI Act audit, an FDA submission, or an HHS OCR inquiry.

You consume from them:

- **The release-assurance template** — the schema the packet has to fit so a regulator or an internal governance committee can review it.
- **Mandatory evidence requirements** — "every release must include stratified bias assessment, incident log, judge calibration report, and a drift-detector status snapshot". You produce; they format and sign.
- **Governance sign-off** — the packet gate before deployment; a release does not go out if the packet is incomplete.

You hand to them:

- **The four packets from Chapters 1–4** — ASR, TTS, conversational, content — with CIs, stratifications, baselines, deltas. Formatted and filed as the template requires.
- **The online eval snapshot** from Chapter 5 — latest drift-detector state, implicit feedback trends, incident log.
- **The incident fixtures from Chapter 5** — a changelog of the regression tests added in the last release window, each tied to the incident that motivated it.
- **Component versions from Chapter 6** — exact ASR/TTS/LLM model versions, provider region, judge model + prompt version, telemetry schema version.

Two roles with "evaluation" in the title; one lives inside your product team's release cadence, the other lives inside the governance organisation's audit cadence. The first makes the eval harness; the second makes the harness's output defensible to an outsider.

Reference peer role in the AI Governance curriculum: the governance-family AI Evaluation Engineer (see [`ai-evaluation-engineer-learning`](https://github.com/ai-governance-curriculum/ai-evaluation-engineer-learning) — same name, different org, higher-level responsibilities around release assurance).

## The handoff artefact: the release-assurance packet

The packet is the single artefact that flows across all three roles. A workable schema:

```yaml
release_assurance_packet:
  release_id: "voice-agent-2026-11-05-r3"
  build_commit: "<git sha>"
  components:
    asr:
      provider: "deepgram"
      model: "nova-3"
      version: "2026-10"
    tts:
      provider: "cartesia"
      model: "sonic-2"
      version: "2026-09"
    llm:
      provider: "openai"
      model: "gpt-4o-2024-08-06"
    judge:
      provider: "anthropic"
      model: "claude-sonnet-4-6"
      prompt_version: "v12"

  asr_packet:                 # from Chapter 1
    normaliser: "whisper.EnglishTextNormalizer v2025-07"
    gold_set_version: "support-en-v8"
    fleurs:   { wer_per_lang: { ... } }
    librispeech: { test_clean: { wer, ci }, test_other: { wer, ci } }
    tedlium3: { test: { wer, ci } }
    in_domain: { overall: { wer, ci }, strata: { ... }, irr: { ... } }
    calibration: { ece, reliability_diagram_uri }

  tts_packet:                 # from Chapter 2
    subjective:
      p808_mos: { mean, ci, rater_pool, n }
      p835: { sig, bak, ovrl, each with ci }
      cmos_vs_last_release: { mean, ci, order_randomised: true }
    automated:
      utmos: { mean, ci }
      dnsmos: { sig, bak, ovrl }
    correlation_study: { pearson, spearman, date, stratified }

  conversational_packet:      # from Chapter 3
    turn_taking: { feot_rate, meot_rate, boundary_err_ms_p95, strata }
    barge_in: { reaction_ms_p50, p95, p99, frames_after_cancel }
    ttfa: { p50, p95, p99, per_leg_breakdown }
    mouth_to_mouth: { p95, budget, pass }
    interrupt_recovery: { scenarios_run, pass_count, failures: [] }

  content_packet:             # from Chapter 4
    judge_calibration:
      policy:       { kappa, n, date }
      groundedness: { kappa, n, date }
      pii:          { kappa, n, date }
      tone:         { kappa, n, date }
    scores:
      policy:       { mean, ci, strata }
      groundedness: { mean, ci, strata }
      pii_leak_rate:{ value, ci }
    task_correctness: { per_flow_pass_rate: { ... }, gold_script_version: "v6" }
    deflection_rate: { value, baseline, delta }
    ci_gate: { config_version, hard: [ ... ], soft: [ ... ] }

  online_eval:                # from Chapter 5
    sampling: { base_rate, triggered: [ ... ] }
    drift_state: { latest_alerts: [], snapshot_date }
    incident_fixtures_added: [ { id, incident_link, fixture_description } ]

  observability:              # from Chapter 6
    semconv_version: "opentelemetry.gen_ai/draft@..."
    voice_attrs_namespace_version: "voice.v2"
    dashboards: [ { name, uri } ]

  governance:
    signed_by: [ "voice-agent-engineer", "ai-evaluation-engineer" ]
    signed_date: "2026-11-05"
    bias_assessment_status: "passed_with_mitigations"
    incident_log_snapshot: "<uri>"
```

The schema will vary by company; the spine (components, four packets, online eval, observability, governance sign-off) is stable. This is the artefact you produce every release; the ai-evaluation-engineer (governance) files it; the ai-eval-engineer (eval methodology) consumes the judge-calibration and dataset-version sections for their own scorecards.

## What this module does NOT cover

- **Generic judge-as-LLM research and tooling.** The field advances fast ([G-Eval](https://arxiv.org/abs/2303.16634), [MT-Bench](https://arxiv.org/abs/2306.05685), [Arena-Hard](https://github.com/lmarena/arena-hard-auto)); the ai-eval-engineer tracks the frontier and ships patterns you reuse.
- **Regulator-facing documentation style.** The ai-evaluation-engineer (governance) writes model cards, DPIA appendices, and audit-response letters. You produce the evidence.
- **Dangerous-capability / frontier-model evals.** The agentic-safety-engineer boundary, called out in mod-108 Chapter 8.
- **Serving-infrastructure latency tuning.** Mouth-to-mouth p95 is yours; the SLO under the inference endpoint is the AI Infra Inference Engineer's (mod-109 Chapter 6).

## Summary

- The **voice-agent-engineer** owns the voice-specific eval harnesses (Chapters 1–6). The harnesses live in the product repo and gate the release.
- The **ai-eval-engineer (L30, Engineering family)** owns deep eval methodology — judge rubrics and calibration, statistical reporting, dataset platforms, cross-product eval patterns. You consume their patterns and hand back voice-specific dimensions.
- The **ai-evaluation-engineer (L35, Governance family)** owns the release-assurance packet, model / system cards, and regulator liaison. You hand them the four packets (ASR, TTS, conversational, content) plus the online eval and observability sections; they sign off.
- The **release-assurance packet** is the one artefact that flows across all three roles; it is versioned, reviewable, and the gate on deployment.

The module ends here. The exercises below operationalise every chapter: build each harness (exercises 01–03), wire the CI gate (exercise 04), ship online eval and drift (exercise 05), and instrument the whole thing with OTel + WebRTC stats on one plane (exercise 06).
