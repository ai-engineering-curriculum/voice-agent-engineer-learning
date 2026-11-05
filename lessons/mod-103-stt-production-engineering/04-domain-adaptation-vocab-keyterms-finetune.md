# 04. Domain Adaptation — Custom Vocab, Keyterm Prompting, Lexicons, and Fine-Tune

## Why this chapter

Every production voice agent has words the generic ASR will get wrong: your product name, the names of SKUs, acronyms, clinical terms, policy codes, place names. A stock ASR trained on web-scraped audio has never heard "Entresto" or "deductible" or your company's brand pronunciation; it will transcribe them phonetically, confidently, and wrongly. The LLM then rationalises the wrong text and the user hears "I'm not sure what you mean" from an agent that misheard their drug name.

Domain adaptation is the discipline of fixing this — with the cheapest tool that works, measured by the WER delta you can defend. The hierarchy is: **custom vocab → keyterm prompting / boost → phonetic lexicon → model fine-tune**, each step more powerful, more expensive, and more rigid than the last. Climb only as far as the metric demands.

## The adaptation hierarchy

In order of cost and lead-in time:

| Technique | What it does | Where it lives | Typical lift | Cost to ship |
|---|---|---|---|---|
| Custom vocabulary / phrase list | Adds terms to the decoder's lexicon; raises their prior | Request param on every stream | Small-to-medium on the listed terms | Minutes |
| Keyterm prompting / boost | Boosts the acoustic/LM score of specified terms, often with a weight | Request param on every stream | Medium-to-large on the listed terms | Hours |
| Phonetic lexicon | Tells the ASR how a term is pronounced (IPA / ARPAbet) | Custom-model-level in some vendors | Large on hard-to-pronounce terms | Day |
| Custom-model / Fine-tune | Retrains the final layer(s) or the full stack on your labelled audio | Model artefact hosted at the vendor or self-hosted | Transforms WER on the domain | Weeks |

Important caveat: specific lift numbers are **domain- and vendor-dependent**, and vendor marketing pages routinely overstate them. Measure on your own data. Chapter 5 is the methodology.

## Technique 1 — Custom vocabulary / phrase lists

The baseline mechanism. Most providers expose a parameter like:

- Deepgram — `keywords=[]` legacy param, `keyterm=[]` on Nova-3. ([Keyterm Prompting docs](https://developers.deepgram.com/docs/keyterm).)
- AssemblyAI — `word_boost=[]` on async; equivalent on streaming. ([AssemblyAI word boost](https://www.assemblyai.com/docs/speech-to-text/pre-recorded-audio/word-boost).)
- Google STT v2 — `SpeechAdaptation` with `phrase_sets`, `custom_classes`, and a `boost` value per phrase. ([Google STT model adaptation](https://cloud.google.com/speech-to-text/v2/docs/adaptation-model).)
- Azure Speech — Phrase lists at session scope; Custom Speech for the heavier version. ([Azure phrase lists](https://learn.microsoft.com/azure/ai-services/speech-service/improve-accuracy-phrase-list).)
- AWS Transcribe — `VocabularyName` referencing a vocab resource; `VocabularyFilterName` for exclusion. ([AWS Transcribe custom vocabulary](https://docs.aws.amazon.com/transcribe/latest/dg/custom-vocabulary.html).)

Shape of what you submit:

```python
# Pseudocode — adjust to provider SDK
keyterms = [
    "Entresto",            # brand drug name
    "Lisinopril",          # generic drug name
    "deductible",          # a word the ASR often mangles
    "Rook Insurance",      # proper noun (brand)
    "Policy 71-A",         # alphanumeric
]
asr.stream(audio, keyterms=keyterms, ...)
```

Three design rules:

- **Keep the list small.** Hundreds of terms, not thousands. Longer lists dilute the prior and often *hurt* WER on common words.
- **Include the full form, not fragments.** "Rook Insurance" beats "Rook" alone — unigram boosts over-fire.
- **Avoid boosting words that are also common nouns in general speech.** Boosting "cat" because you sell a product called CAT will steal probability from "cat" in every utterance about pets.

Custom vocab is **per-session** on most providers — you push it on stream creation. Dynamic lists (e.g., the caller's name added at session start) are natural here.

## Technique 2 — Keyterm prompting / boost

A step up from a flat list: each term carries a **boost weight** indicating how aggressively to raise its score. Weights are provider-specific; Google STT uses a 0–20 scale, AWS Transcribe uses `BoostLevel` HIGH/MEDIUM/LOW, Deepgram Nova-3's `keyterm` is a prompt-style list without an explicit weight.

Rules of thumb:

- **Start low.** Over-boosting a term creates false positives — the ASR hears "boost" everywhere. Google's docs recommend starting around 10 and tuning down.
- **Per-term A/B testing.** Not every term benefits equally. Some terms improve the moment they're listed; others take significant boost; others cause regressions. Measure (Chapter 5).
- **Context phrases beat single words.** "Rook Insurance policy number" is more effective than "policy" alone because it anchors the acoustic context.
- **Use custom classes for structured text.** Google's `custom_classes` let you declare an open-ended slot ("ORDER_NUMBER = any 8-digit integer with hyphens") and the decoder applies the pattern. AWS Transcribe has a similar mechanism via custom vocabulary filter types.

Operationally, keyterm lists need **version control and A/B tooling**. A list with 200 terms, half of which help and half of which hurt, is normal. Treat the list as a code artefact — PR review, test deltas, rollback.

## Technique 3 — Phonetic lexicons

For words with non-standard pronunciations, telling the ASR the pronunciation is worth more than any boost.

Supported where:

- Azure Custom Speech — pronunciation guidance via a `<pronunciation>` tag in training data.
- AWS Transcribe custom vocabulary — `SoundsLike` column with phonetic hints in a vendor-specific syntax, plus an `IPA` column for IPA transcriptions. ([AWS Transcribe custom vocab reference](https://docs.aws.amazon.com/transcribe/latest/dg/custom-vocabulary-create.html).)
- Google STT — `custom_classes` can reference pronunciation hints in model-adaptation configurations.
- Self-hosted Whisper / NeMo — lexicon augmentation is possible but is `nlp-engineer` territory, not pipeline configuration.

Use cases where this tier earns its keep:

- Pharma product names with non-English phonetics.
- Brand names with deliberate mispronunciations ("Hyundai", "Porsche").
- Names of people on your team / customers whose names the ASR will not correctly English-ify.
- Technical acronyms the ASR will otherwise spell out letter-by-letter.

The cost: phonetic transcriptions are not natural for engineers to write, so you either need a linguist's help or an IPA conversion tool. They also need maintenance when the word's pronunciation evolves in your product.

## Technique 4 — Fine-tune / custom model

The nuclear option, and the one where the chapter starts drawing boundaries.

What fine-tuning is: retraining some or all of the ASR model's parameters on your labelled audio to bake in the domain. What you get in return: a step-change in WER on the domain, in exchange for a hosting footprint, an evaluation pipeline, and a training cadence.

Where it's available as a managed product:

- **Azure Custom Speech** — upload audio + transcripts, train, host. ([Azure Custom Speech](https://learn.microsoft.com/azure/ai-services/speech-service/custom-speech-overview).)
- **Google Cloud STT model adaptation at scale** — beyond phrase sets, the broader model-tuning surface. <!-- needs-research: confirm current managed-fine-tune surface for Google STT v2 Chirp at author time; the offerings here have shifted meaningfully between releases -->
- **AWS Transcribe Custom Language Models (CLM)** — text-only corpus to adapt the decoder's language model; no audio training. ([AWS Transcribe CLM](https://docs.aws.amazon.com/transcribe/latest/dg/custom-language-models.html).)
- **AssemblyAI custom model training** — a vendor-managed service for fine-tuning on your data. <!-- needs-research: AssemblyAI custom-training availability and SKU names at author time -->
- **Deepgram** — the current product tier for domain fine-tuning. <!-- needs-research: Deepgram custom-model offering at author time; feature set has evolved across Nova generations -->
- **Self-hosted Whisper / NeMo / wav2vec-2** — DIY on your own infrastructure.

When to reach for a fine-tune:

- Keyterm prompting and lexicons have been exhausted and WER on the domain gold set still misses the target.
- The domain has **structural** acoustic mismatch, not just vocabulary mismatch: e.g., very noisy audio, far-field capture, consistent accent cluster the generic model under-serves, non-standard dialectal pronunciations.
- You have at least **tens of hours** of transcribed in-domain audio (rule of thumb; vendor minimums vary from 1 hour for text-only adaptation to 100+ hours for full acoustic fine-tune).

## The data contract: what you hand to `fine-tuning-engineer` or `training-pipeline-engineer`

Fine-tuning, when it's more than a managed-service "upload audio + transcripts" button, is a peer handoff. The roles you work with:

- **`fine-tuning-engineer`** (L30, in the sibling `fine-tuning-engineer-learning` curriculum) — owns the fine-tune: selecting the base model, configuring the training run, hyperparameter sweeping, evaluation against your acceptance threshold, shipping the artefact. <!-- needs-research: confirm canonical fine-tuning-engineer-learning repo URL at author time -->
- **`training-pipeline-engineer`** (L35, sibling curriculum) — owns the training infrastructure: distributed training, dataset ETL, checkpoint management, evaluation harness, continuous training cadences. <!-- needs-research: confirm canonical training-pipeline-engineer-learning repo URL at author time -->

Rule: if it's a one-off fine-tune against an existing base model, it's usually `fine-tuning-engineer`. If it's a continuous training loop that runs against streaming new labelled data on your own infra, it's `training-pipeline-engineer`.

The handoff artefact — a one-pager, versioned, reviewed:

### 1. Problem statement

- Current provider / model + baseline WER on the in-domain gold set.
- The specific error classes that matter (e.g. "drug-name substitutions represent 34 % of errors and 70 % of user-visible failures").
- Business cost of each error class — tied to a KPI in Chapter 5.
- Target WER on each class, plus a date.

### 2. Domain corpus

- **Hours of transcribed audio.** Split into train / dev / test, with the test set frozen *before* fine-tuning starts.
- **Source audio provenance.** How was it captured, from whom, with what consent. This is also the compliance-boundary artefact — see Module 108.
- **Transcription quality.** Who transcribed (in-house human, vendor service, partial auto-transcription + human review), with what style guide (punctuation conventions, capitalisation, number formatting).
- **Inter-rater reliability.** If multiple transcribers — WER agreement between them on a sampled subset. Chapter 5 specifies the method.
- **Metadata.** Language, dialect, speaker demographics (where ethically collected), audio SNR distribution, channel conditions (phone / web / in-person), speaker count per clip.

### 3. Acceptance criteria

- **WER target** on the frozen test set.
- **Per-class WER target** where it matters (drug names, policy codes, numbers).
- **No-regression target** on a held-out generic-English set (e.g. LibriSpeech test-clean), so the model doesn't over-fit to the domain.
- **Latency target.** A fine-tuned model that is 30 % slower than the baseline is a *regression* for streaming agents. Specify the latency budget up front.
- **Deployment posture.** Hosted at vendor, on-prem, both. Hosting cost ceiling.

### 4. Current pipeline integration

- Which provider / endpoint the fine-tuned model replaces.
- Which streaming-contract fields the pipeline consumes (word timings? confidence? diarisation?) — the fine-tuned model must emit a superset.
- Rollout plan: shadow mode → canary → full roll — the evaluation harness has to support each stage.

Hand this document across, not Slack messages. The quality of the fine-tune is bounded by the quality of this spec.

## What stays yours after the fine-tune lands

Even with a fine-tuned model, the pipeline-engineering work remains yours:

- **Serving the model** (self-hosted) — GPU scheduling, inference runtime, autoscaling. Shared with `ml-platform-engineer` for the fleet side. <!-- needs-research: confirm canonical ml-platform-engineer role in curriculum at author time -->
- **Observability** on the fine-tuned model — WER drift, confidence drift, latency drift. Module 112.
- **Re-tuning triggers** — when WER drifts past a threshold or new vocabulary appears, you schedule the next fine-tune cycle. This is where `training-pipeline-engineer`'s continuous-training work meshes with your observability loop.
- **Keyterm lists and prompting** still apply on top of a fine-tuned model — they're cheaper than another fine-tune and compose with it.

## Measuring the lift honestly

The temptation after a fine-tune or a new keyterm list is to report a single WER number. Resist. Report:

- **WER on the domain gold set, before and after**, with a bootstrap 95 % confidence interval.
- **WER on a held-out generic set** — unchanged is good; worse is a regression.
- **Error-class breakdown** — how many of the previously-wrong proper nouns are now right? Any new regressions in common-English coverage?
- **Latency delta** — fine-tuned models often grow; keyterm lists sometimes slow decoding slightly on some vendors.
- **Business-KPI delta** — cost per handled call, deflection rate, escalation rate. Chapter 5 ties the WER numbers to these.

A fine-tune that improves WER from 11 % to 8 % on paper but raises first-partial latency from 250 ms to 450 ms is a *loss* for an interactive agent. The honest write-up names this.

## Common anti-patterns

- **Stuffing the keyterm list.** 5 000 terms, including common English nouns, boosted to the ceiling. Decoder drifts toward the boosted vocabulary; WER on general utterances degrades. Keep the list surgical.
- **Jumping straight to fine-tune.** Every provider has a free-ish keyterm / phrase mechanism; try that first. A 30-minute keyterm experiment can save a three-month fine-tune.
- **No frozen test set.** Fine-tuning engineers who report WER on data they trained on will hit a dramatically different number in production. Freeze the test set *before* the fine-tune begins.
- **Ignoring the no-regression set.** A fine-tuned ASR that perfectly transcribes your proprietary acronyms and can no longer handle ordinary English on a cold call is a product disaster. Always evaluate against a generic held-out set.
- **Owning the fine-tune training infra yourself** at L35. Unless you are also the training-pipeline engineer for the team, this is scope creep. Hand across; stay accountable for the integration and the metric.

## Summary

- Climb the adaptation ladder from phrase lists to fine-tune, picking the highest-leverage rung at each step.
- Keyterm lists are version-controlled artefacts with per-term A/B evidence, not a dumping ground.
- Fine-tuning is a peer handoff to `fine-tuning-engineer` or `training-pipeline-engineer` with a strict data contract: problem, corpus, acceptance criteria, integration.
- Measure lift on the domain *and* on a generic no-regression set, with latency and business KPIs attached. A WER win that costs you 200 ms of latency is often a net loss.

Next chapter: how to run the WER benchmark honestly — LibriSpeech as a baseline, in-domain gold sets as the real metric, and the business-KPI translation that lets you defend the number upwards.
