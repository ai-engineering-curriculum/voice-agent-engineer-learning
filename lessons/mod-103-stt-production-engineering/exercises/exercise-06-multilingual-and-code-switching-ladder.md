# exercise-06: Multilingual and Code-Switching Ladder

**Estimated effort:** 2.5 hours · **Output:** a FLEURS per-language scoring run + a code-switching stress test + a tier-and-policy document

## Objective

Produce a defensible multilingual language-support policy for your voice agent. By the end you will have:

1. **Per-language WER numbers** on FLEURS for at least one open-weights multilingual model (Whisper large-v3 or SeamlessM4T) and at least one managed multilingual provider.
2. A **small code-switching stress test** showing how each model handles a bilingual utterance.
3. A **language-support tier document** classifying candidate languages into "tier 1 / tier 2 / unsupported" against measurable criteria.

This is the Chapter 6 skill applied to concrete evidence, not a vendor-claim summary.

## Prerequisites

- [Chapter 6 — Multilingual, Code-Switching, and Accent Scenarios](../06-multilingual-code-switching-and-accents.md).
- Python 3.10+ with `datasets`, `transformers`, `torch`, `jiwer`, `openai-whisper` (for the multilingual normaliser).
- Hugging Face access to [`google/fleurs`](https://huggingface.co/datasets/google/fleurs).
- GPU recommended for Whisper / SeamlessM4T (CPU is possible but slow — budget accordingly).
- Developer access to a managed provider that supports 10+ languages. Google STT v2 Chirp, Azure Speech, and Speechmatics Ursa are all defensible choices.

## Steps

### Part A — Pick 5 languages (15 minutes)

1. Pick **5 languages relevant to a realistic deployment**. Include:
   - 1 "tier 1" language you expect every vendor to do well on (English, Spanish, French, German, or Mandarin).
   - 1 "mid-tier" language with moderate commercial coverage (e.g. Portuguese, Polish, Turkish, Vietnamese).
   - 1 "lower-tier" or under-served language (e.g. Welsh, Swahili, Bengali, Tagalog) — a language where you expect material quality differences across vendors.
   - 2 more of your choice, driven by the deployment you picked in exercise-01.
2. For each language, write a one-line note: why it's in scope, estimated user volume, which stratum your users fall into.

### Part B — Score FLEURS per language (60 minutes)

3. Download FLEURS test splits for your 5 languages. The dataset has ~350 utterances per language; subsample 100 per language to keep runtime reasonable.
4. Run **model A: open-weights Whisper large-v3** (or Whisper-turbo; name which). For each of the 5 languages, compute WER (with bootstrap CI) + CER, using the multilingual Whisper normaliser.
5. Run **model B: a managed multilingual provider** on the same utterances. Match the model version where possible.
6. Produce a per-language table:

   | Language | FLEURS WER (Whisper) | FLEURS CER (Whisper) | FLEURS WER (Managed) | FLEURS CER (Managed) | Delta (W − M) |
   |---|---|---|---|---|---|

7. Note each language's tier against Chapter 6's shortlisting rule (WER < 15 % → tier-1; 15–30 % → tier-2; > 30 % → unsupported).

### Part C — Code-switching stress test (45 minutes)

8. Prepare 10 code-switched utterances. These can be:
   - Self-recorded (easiest): record yourself saying 10 short sentences that mix two languages. E.g. "I'd like to transfer cincuenta euros to my savings account." A bilingual colleague is a huge help here.
   - Sampled from a real code-switching dataset. The SEAME corpus (Mandarin-English) and the ArzEn corpus (Arabic-English) are the usual references. <!-- needs-research: confirm current, licensable code-switching corpora at author time; some are research-only -->
9. Score each utterance with:
   - **Model A (Whisper)** with language set to `auto` (let it detect).
   - **Model A (Whisper)** with language pinned to the dominant language.
   - **Model B (Managed)** with the provider's multilingual / auto-detect mode if available; monolingual if not.
10. Compute per-utterance WER against your ground-truth bilingual transcript. (Scoring code-switched WER is itself subtle — see "Starter guidance" below.) Note which errors are **language misidentification** vs. **word substitution**.
11. Produce one qualitative observation per model: what fails, and why?

### Part D — Write the tier-and-policy document (30 minutes)

12. For each of your 5 languages, decide the tier and justify it with evidence:

    | Language | Tier | FLEURS evidence | Code-switch behaviour | Recommended provider | Policy caveat |
    |---|---|---|---|---|---|

13. Add three policy decisions to the document:
    - **Which languages the agent commits to serving.** Advertised; held to SLA.
    - **Which the agent handles with a caveat.** ("Supported in beta; may require clarification.")
    - **Which the agent will not serve** — and what the graceful degradation is (polite refusal + human handoff).
14. Add one paragraph on **accent policy** inside your tier-1 languages: how will you stratify in-domain evaluation (Chapter 5) to detect accent-based disparities? Cite [Koenecke et al. 2020](https://www.pnas.org/doi/10.1073/pnas.1915768117) as the floor.

## Starter guidance

- The FLEURS language codes are 2-letter + script (e.g. `en_us`, `cmn_hans_cn`). Check the dataset card on Hugging Face for the exact codes for your target languages.
- Whisper's multilingual normaliser is `whisper.normalizers.BasicTextNormalizer`. For specific languages it does less than the English normaliser does; document what it does and doesn't do.
- For **code-switching WER**, there is no universal convention. The common approach: concatenate the bilingual reference as plain text and score as if monolingual. Alternatives — scoring each language's sub-string separately, scoring with language-tagged tokens — exist but add complexity. Pick one and document it.
- Latency measurement matters even in a WER-focused exercise. Note which languages your managed provider was dramatically slower on. A tier-1 language at 400 ms is tier-1; at 2000 ms it may not be.
- Bilingual utterance generation: a small LLM can produce a code-switching script you then read aloud. Keep it natural — "I need to check my saldo" is realistic; "I need to analizar the cosmological expansion coefficient of my milkshake" is not.

## Acceptance criteria

- [ ] 5 languages picked, with one-line rationale each.
- [ ] FLEURS per-language WER + CER for both models, with bootstrap CIs on WER.
- [ ] Per-language tier assignment against Chapter 6's shortlisting rule.
- [ ] Code-switching stress test: 10 utterances, scored under 3 configurations, with language-misidentification vs. substitution errors noted.
- [ ] Tier-and-policy document: tier table + three policy decisions + accent-stratification plan.
- [ ] Evidence-based justification for each tier assignment (no vendor marketing numbers).

## Stretch goals

- **Add a third model: MMS or SeamlessM4T**. These are the open-weights tier for broad language coverage; a 102-language FLEURS run on either is a tour de force. Compare to the commercial and Whisper numbers.
- **Measure accent stratification.** Within your tier-1 language, score an accent-stratified subset of Common Voice (or your in-domain audio from exercise-05 if you have accent labels). Reproduce the Koenecke et al. phenomenon on your own data: do you see a WER gap across accent clusters?
- **Design a language-detection layer.** Build a tiny classifier (or use Whisper's own LID output) that routes incoming audio to the right language-pinned pipeline. Measure its error rate. What happens on code-switched utterances?
- **Draft the handoff to `fine-tuning-engineer`** for the under-served language(s) in your 5. Write the data-contract spec from Chapter 4: hours needed, acceptance criterion, deployment posture. Having to write it will reveal whether the language is actually one escalation away or ten.
- **Add a translation-pipeline comparison.** For one tier-2 or tier-3 language, run the pipeline two ways: (a) native-language STT → native-language LLM; (b) native-language STT → translate to English → English LLM → translate back. Measure quality on a few representative tasks. This is the architectural question Chapter 6 names, made tangible.

## Keep it

The FLEURS per-language scoring script becomes a regression test you run on every provider version bump — it catches silent coverage changes that vendor release notes don't highlight. The tier-and-policy document is a living artefact; refresh quarterly as you add users, add languages, or detect drift.
