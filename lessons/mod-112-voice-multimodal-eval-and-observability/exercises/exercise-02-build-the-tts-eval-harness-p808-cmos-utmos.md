# exercise-02: Build the TTS Eval Harness — P.808 MOS, CMOS A/B, UTMOS + DNSMOS Proxies

**Estimated effort:** 3 hours · **Output:** a TTS eval harness, a dry-run P.808 panel plan, a CMOS A/B test design, a running UTMOS/DNSMOS CI job, and a TTS release-assurance packet

## Objective

Build the TTS eval surface: author a P.808 crowdsourced-MOS panel (with the Microsoft `P.808` reference implementation) and a CMOS A/B protocol you *could* run, and operationalise UTMOS as the CI-friendly proxy and DNSMOS as the production-monitoring proxy. You will not necessarily *run* a live paid crowdsource panel for the exercise — a dry-run with 10 in-house raters is enough — but you must produce the panel design (clip set, rubric, rater qualification, randomisation, n), the correlation study linking the proxy to the subjective panel, and the release-assurance TTS packet.

## Prerequisites

- [Chapter 2 — The TTS Eval Harness](../02-tts-eval-harness.md).
- mod-104 (TTS production engineering) for the current TTS vendor / model in use.
- A streaming TTS integration from mod-104 or mod-111 that can synthesise arbitrary text to wav.
- Python 3.11+. Recommended libraries: a current open-source UTMOS release (e.g., [`sarulab-speech/UTMOS22`](https://github.com/sarulab-speech/UTMOS22) or a maintained fork), a DNSMOS release ([Microsoft DNS Challenge](https://github.com/microsoft/DNS-Challenge)'s reference model), `numpy`, `scipy`, `matplotlib`.
- A reader of the current [ITU-T P.808](https://www.itu.int/rec/T-REC-P.808) and [P.835](https://www.itu.int/rec/T-REC-P.835) recommendations. The text is authoritative; do not substitute blog summaries.
- A crowdsourcing-platform account (Prolific or Mechanical Turk) for the dry-run — optional for the exercise if budget is a constraint; document the deferral if so.

## Steps

1. **Define the TTS eval clip set.** A frozen set of 100–200 utterances covering your product's distribution: short conversational replies, numeric read-backs, policy disclosures, long multi-sentence responses, locale/dialect variants if applicable. Version it (`tts-eval-v1`).

2. **Synthesise the clip set with your production TTS** at the exact streaming configuration you ship. Save WAVs plus per-clip metadata (text, synth model, voice ID, sample rate, codec, synthesis latency).

3. **Design the P.808 panel.** Using the Microsoft [`microsoft/P.808`](https://github.com/microsoft/P.808) reference implementation, prepare the ACR template with your clip set. Define: rater pool (Prolific / MTurk / internal), rater qualification flow (headphone check, hearing threshold, comprehension), attention-check clips, reference anchors, per-clip rater count target, payment per task at or above local minimum wage. Document as a one-pager plan.

4. **Dry-run the panel with ≥10 in-house raters.** Walk ≥10 people through the ACR UI on a laptop. Collect raw scores. Produce the standard P.808 output: per-clip MOS, per-system MOS, 95% CI, rater n. Flag that this is a dry-run; production decisions require the real panel.

5. **Design the CMOS A/B protocol.** Pair this release's TTS with the prior release (same utterances, randomised A/B order per rater, 7-point comparison scale, anchor pairs with known preference). Document the protocol even if you only dry-run it.

6. **Wire UTMOS into CI.** Install an open-source UTMOS release. Build a CI job that: reads the frozen clip set, synthesises each with the current TTS build, scores with UTMOS, compares against the baseline manifest, fails the job on regression. Document the regression threshold and how it was chosen (noise band on the baseline eval set).

7. **Wire DNSMOS for production monitoring.** Install a DNSMOS release. Build a service that: receives sampled TTS audio from production (Chapter 5 sampling policy), runs DNSMOS SIG / BAK / OVRL, emits as OTel metrics named per Chapter 6 (`voice.tts.mos_proxy.*`). For the exercise, run against a canned batch of production-shape audio and verify the metrics flow.

8. **Correlation study.** Run UTMOS *and* DNSMOS on the same clip set the dry-run P.808 panel saw. Report Pearson and Spearman correlation between the proxy and the panel. Scatter plots with per-stratum colouring. Document the result in the packet.

9. **Emit the release-assurance TTS packet.** YAML/JSON with: clip-set version, subjective panel results (MOS, CI, rater pool, n) marked as dry-run or production, CMOS design (and results if run), UTMOS CI threshold + current value, DNSMOS production baseline + current value, correlation study results, caveats where a proxy is used outside its calibrated distribution.

## Starter guidance

- **Run the Microsoft `P.808` repo locally before you commit to a crowdsource vendor.** The template, the qualification flow, and the data cleaning are already written. Re-implementing this is not a good use of exercise time.
- **The dry-run panel will be biased.** In-house raters know what the system does and anchor their scale weirdly; the real crowdsource panel is where the numbers come from. Use the dry-run to validate the pipeline, not the model.
- **Automated proxies are comparison tools.** UTMOS 4.0 → 3.8 absolute may not mean the audio got worse — the proxy's scale is not a product truth. Changes on your own fixed set are the signal. Document this prominently in the packet.
- **DNSMOS P.835 heads are conceptually distinct from P.835-the-standard.** The DNSMOS model predicts the three P.835 subjective scores; you still run a P.835 subjective panel periodically if noise-suppression quality matters to your product.
- **Keep the clip set content-balanced across your real call distribution.** A clip set that is 80% short greetings will not catch a long-response regression.
- **Legal / ethics for the crowdsource panel.** Underpaid raters are a quality problem *and* an ethics issue; set the hourly rate at or above local minimum wage and document it.

## Acceptance criteria

- [ ] A versioned TTS eval clip set with metadata, synthesised through the production TTS build.
- [ ] A P.808 panel plan (clip set, rubric, rater qualification, anchors, randomisation, n, payment), runnable on Microsoft `P.808`.
- [ ] A dry-run P.808 result with ≥10 raters, per-clip MOS + CI + rater n.
- [ ] A CMOS A/B protocol documented (clip pairs, scale, order randomisation, anchors), run or deferred.
- [ ] A UTMOS CI job that regresses on a fixed clip set and fails a PR on regression, with a documented threshold.
- [ ] A DNSMOS production-monitoring service emitting SIG / BAK / OVRL as OTel metrics on a sampled-production stream.
- [ ] A correlation study (Pearson + Spearman + scatter) connecting the proxy to the dry-run subjective panel.
- [ ] A machine-readable `tts_packet.yaml` / `tts_packet.json`.

## Stretch goals

- **Run the real P.808 panel.** Pay for a 50-rater Prolific / MTurk run on the clip set. Replace the dry-run numbers in the packet.
- **Add a P.835 panel.** Required if there is noise suppression or aggressive codec compression in the chain.
- **Intelligibility crosscheck.** Pipe the synthesised audio into a *different* ASR than your production one; compute WER vs. the input text. Report alongside MOS.
- **Lip-sync + AV-sync metric** for a multimodal avatar (mod-110). Measure audio-video offset distribution and run a SyncNet-style discriminator.
- **Second-voice CMOS.** Use CMOS to compare two different voices from your TTS vendor on the same clip set. The protocol is reusable for every future voice-model promotion.

## Keep it

The packet feeds exercise-04's CI gate. The UTMOS CI job is the exact pattern the gate wraps. The DNSMOS service is the first producer of a Chapter 6 OTel metric; keep it running.
