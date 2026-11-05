# 06. Multilingual, Code-Switching, and Accent Scenarios

## Why this chapter

Every provider has a "100+ languages supported" badge on their landing page. The number is marketing. The real question is *which* 100, with what quality, and on what kinds of audio. A voice agent that works beautifully in English and mangles the same user's name when they say it with its correct pronunciation is a worse product than an English-only agent that admits it only speaks English.

This chapter gives you the picture: how to read a multilingual ASR benchmark honestly, how code-switching breaks most stacks, how to handle accents the provider under-serves, and when a local per-language ASR beats a general-purpose foundation model.

## The FLEURS per-language picture

[FLEURS (Conneau et al., 2022)](https://arxiv.org/abs/2205.12446) covers 102 languages, with the same source sentences translated and read in each. The per-language WERs it produces are the first comparable multilingual ASR numbers in the field.

When you read a FLEURS table for a provider, three questions:

1. **Is there a clean "tier 1" group and a long tail?** Almost always yes. Whisper's own numbers in Appendix D of the [Whisper paper](https://arxiv.org/abs/2212.04356) show single-digit WER on about a dozen languages and WERs above 50 % on dozens more. Every other provider has roughly the same shape; the tier membership differs.
2. **Are your user languages in tier 1?** If English, Spanish, French, German, Mandarin, Japanese, Portuguese, Italian, Dutch, Korean — probably yes across most providers. If Khmer, Yoruba, Odia, Haitian Creole — probably no, and the provider's badge is misleading.
3. **What does the provider's per-language FLEURS number predict for your conversational audio?** FLEURS is read speech in a studio setting. Conversational and telephony audio is usually 2–4× worse WER. If FLEURS gives you 15 %, expect 30–60 % on real agent audio.

A reasonable shortlisting rule for multilingual agents:

- **Tier 1 for your product** (any language you commit to a user you speak): provider must have a published FLEURS or equivalent WER better than ~15 % *and* you must reproduce it on your in-domain gold set.
- **Tier 2** (supported with caveats): FLEURS WER in the 15–30 % range; reserved for low-stakes or confirmation-only tasks.
- **Unsupported** (the agent politely admits this): FLEURS WER above 30 % or no published number. Route to a human.

## Code-switching

**Code-switching** is a single speaker mixing two or more languages within a single conversation or utterance: a Spanish-English speaker ordering "una coffee, please", a Mandarin-English speaker in Singapore saying "我 can help you with that, 对不对?", a Hindi-English speaker in India narrating a technical problem.

Most production ASR stacks are **one language per session** — you declare the language at stream start and the model is optimised for that language. A code-switched utterance then falls off a cliff: the English half gets transcribed, the Spanish half becomes gibberish, or vice versa.

### Where code-switching gets handled

- **Language auto-detection at session start.** First few seconds of audio → detected language → the "right" monolingual model. Doesn't handle *intra-utterance* switching.
- **Streaming language ID + model switching.** Harder: detect the switch mid-stream and hand off to a different model. Real-world latency implications are significant; few production stacks do this cleanly.
- **Multilingual single-model ASR.** A single model trained on multi-language data. Whisper, MMS, Google Chirp, Meta SeamlessM4T. These handle code-switching *better* than monolingual-switched systems — they can decode a bilingual utterance in one pass — but the per-language WER is often slightly worse than a dedicated monolingual model.
- **Phonetic-space tolerance.** Some vendors can mix a small set of English loanwords ("iPhone", "WhatsApp", "email") into non-English transcripts, even on monolingual models. Not full code-switching, but it covers the most common real-world case.

### Design decisions

- **Check your users' actual code-switching rate.** Sample calls and label the fraction that are code-switched. A 1 % rate is usually safe to ignore with a graceful-degradation fallback; a 20 % rate is a core product requirement that drives the provider decision.
- **If code-switching matters, prefer a multilingual single-model ASR**, not a language-switching pipeline. SeamlessM4T, Chirp, and Whisper large-v3 are the three families to shortlist.
- **Keyterm lists work across languages.** Boost English loanwords in the keyterm list for the non-English language — it closes a lot of the gap cheaply.
- **The LLM downstream often cleans up residual code-switching errors** — a bilingual LLM can repair a code-switched transcript more reliably than it can repair a monolingual transcription of the same utterance. Factor this in when choosing between a monolingual ASR + LLM-correction pipeline vs. a multilingual ASR.

## The SeamlessM4T option

[Meta's SeamlessM4T (and v2)](https://ai.meta.com/research/seamless-communication/) is a single open-weights model for speech-to-text, text-to-speech, speech-to-speech translation, and more, across ~100 languages.

For STT specifically:

- **Strengths.** Broad language coverage; unified model (one deployment, many languages); strong on many non-English languages; translation built in.
- **Weaknesses.** Large (billions of parameters) so inference latency / cost is nontrivial on CPU; English WER is competitive but not always best-in-class; streaming use cases are limited vs. a streaming-optimised Deepgram or AssemblyAI.
- **When to reach for it.** Multilingual batch pipelines; low-resource languages where no vendor has coverage; cases where you also need S2S translation and want one model to manage rather than three.
- **When not to.** Single-language interactive agents where a streaming-specialised provider will beat it on latency; English-dominant products where Whisper-turbo or Deepgram are cheaper and faster.

Open-weights, Apache-2.0 licensed. If you're self-hosting multilingual STT and you don't already have a deeper research reason to pick something else, this is the default shortlist entry.

## Accent handling

Accent is the *within-language* axis of multilingual quality. A stock English ASR trained predominantly on North American English can catastrophically underperform on:

- British, Irish, Scottish, Australian, New Zealand, South African English — the "other native" accents.
- Indian, Singaporean, Nigerian, Filipino English — accents of English as a second or co-official language.
- Non-native-speaker English of all kinds.
- Regional US accents under-represented in training data (Appalachian, Rural Southern, African American English — see [Koenecke et al., 2020 — "Racial disparities in automated speech recognition"](https://www.pnas.org/doi/10.1073/pnas.1915768117) for a rigorous demonstration of the gap).

The Koenecke et al. paper is required reading. Measured against identical transcripts, five major commercial ASR systems had WERs roughly twice as high for Black speakers as for White speakers. The gap is real, measurable, and material; it is also fixable with disciplined evaluation and provider selection.

### What to do about accents

1. **Measure before you commit.** Your in-domain gold set from [Chapter 5](05-wer-benchmarking-librispeech-and-in-domain.md) must be stratified by accent cluster relevant to your users. Report per-stratum WER, not just the global number.
2. **Shortlist providers that advertise accent robustness.** Speechmatics has historically positioned on this; Google Chirp was trained with explicit multilingual + accent diversity; Whisper's broad training data gives it baseline resilience.
3. **Keyterm lists for domain + accent.** Proper nouns pronounced with non-US phonetics are the first and worst casualties; phonetic hints (Chapter 4) help.
4. **Fine-tune where the accent underservice is systematic.** If 30 % of your users speak an accent that gives 2× the global WER, that's a `fine-tuning-engineer` escalation. Hand across the stratified gold set and the acceptance criteria.
5. **Communicate failure honestly.** If an accent is below the acceptance threshold and you can't fix it in this quarter, degrade gracefully — a human handoff beats a wrong-transcription loop.

### Dialect vs. accent vs. sociolect

Three concepts you should be able to use correctly in a design doc:

- **Accent** — phonetic differences in pronunciation of the same language variant.
- **Dialect** — accent plus vocabulary and grammatical differences (e.g., "rubbish" vs. "garbage"; "reckon" vs. "think").
- **Sociolect** — language variation along social dimensions (age, class, community), of which African American English is the most-studied example.

ASR is sensitive to all three. A model trained predominantly on a single sociolect of English will systematically mis-transcribe other sociolects even when the audio quality is identical. Treat sociolect as a first-class stratification axis in the gold set.

## Local per-language ASR vs. foundation model

The architectural choice: one big multilingual model, or a per-language monolingual ASR for each supported language.

### Advantages of a foundation (multilingual) model

- **One deployment, many languages.** Simpler ops; one model to monitor, one fleet to scale.
- **Zero-shot coverage** on languages you didn't commit to but a user occasionally arrives speaking.
- **Code-switching handled natively.**
- **Transfer gains** — training data in a high-resource language can help a lower-resource one via shared representations.

### Advantages of a monolingual per-language ASR

- **Peak accuracy on the target language.** The model is optimised for one acoustic / linguistic space.
- **Smaller, faster models per language.** Lower latency, lower cost per minute.
- **Language-specific tuning.** Keyterm lists, phonetic lexicons, and fine-tunes are per-language tuned; a multilingual model's tuning surface is coarser.
- **Independent evolution.** Upgrading the French model doesn't risk the English numbers.

### The decision rule

- **Interactive agents in a known language with latency constraints:** per-language monolingual ASR (or a single multilingual ASR the provider has specifically tuned for your tier-1 languages). Deepgram's and AssemblyAI's streaming models are effectively monolingual-per-session and fast; Chirp and Whisper are multilingual. Pick based on your language mix + latency profile.
- **Batch analytics across many languages:** foundation model. Whisper large-v3, SeamlessM4T. One pipeline, universal coverage.
- **Multi-accent English-only product:** English monolingual, but choose a provider with explicit accent-diversity training and verify on your stratified gold set.
- **Products that must handle a long tail of user languages:** foundation model as the floor; per-language monolingual as an upgrade once a language reaches the volume that justifies the ops.

Document the choice in the latency budget (Module 109) and the observability design (Module 112). Multilingual monitoring is per-language, not pooled.

## When speech translation is the right answer (and when it isn't)

A related option: **transcribe the user's utterance in their language and translate it to a canonical language for the LLM** (usually English), then translate the LLM's response back. Speech translation is where [SeamlessM4T](https://arxiv.org/abs/2308.11596) and the wider Meta speech-translation stack land.

Advantages:

- Keeps the LLM and the business-logic layer single-language.
- Reduces the LLM evaluation surface to one language.

Disadvantages:

- **Each translation step is a lossy pass.** Nuance, hedging, and modal verbs are the first things to go. Disambiguation becomes harder for the LLM.
- **Prosody is lost twice** — once in STT, once in the TTS translation back.
- **Latency compounds.** Four model passes instead of two.
- **Evaluation of the pipeline requires bilingual evaluators**, which raises the cost of your gold set.

The honest rule: translate only for batch-analytics or low-latency-unimportant use cases. For interactive agents where the user speaks a language you have a decent ASR for, run the whole pipeline in that language — it is almost always strictly better.

## Handing the problem across to `nlp-engineer`

Some multilingual questions are not pipeline-engineering questions:

- "The provider's Welsh WER is 45 %; can we fine-tune MMS on 20 hours of Welsh contact-centre audio and beat it?" → `fine-tuning-engineer` + `nlp-engineer`.
- "Our Arabic transcription is handling MSA well but failing on dialect — can we train a dialect-aware model?" → `nlp-engineer`, with the dialect gold set as the handoff artefact.
- "We need a speech model for a language with no commercial coverage." → research-adjacent; probably not a project a voice-agent engineer should own alone.

The handoff contract from Chapter 4 applies here too: problem, corpus, acceptance criteria, integration. Flag the data-availability constraint explicitly — low-resource languages are often constrained by what audio you can legally collect and annotate.

## Common anti-patterns

- **Reporting a single global WER** on a multilingual product. The headline hides the catastrophe on tier-3 languages. Always per-language.
- **One-language-per-session assumption** on code-switching audiences. The feature "works" until a user says "podría transferirme a a representative, please" and the stack falls over.
- **Ignoring accent stratification.** A product that works for native North American English speakers and fails for everyone else is both a quality issue and an equity issue; name it in your reports.
- **Over-reaching on language coverage in marketing.** If the agent cannot reliably do Khmer, do not advertise that it can. The reputational cost of a mis-advertised language is much higher than the cost of a polite "I only speak English, Spanish, and Mandarin" opening.
- **Translating when you could transcribe natively.** Lossy; latency-costly; evaluation-costly. Translate only where you must.

## Summary

- Read multilingual provider claims through FLEURS — comparable per-language numbers; tier your supported languages explicitly.
- Code-switching needs a multilingual single-model ASR; monolingual-switched pipelines fail on intra-utterance mixing.
- Accent is within-language diversity; stratify the gold set and measure per cluster. The Koenecke et al. result is not a bug, it's a baseline you are accountable to improve on.
- Per-language monolingual ASR wins on interactive latency; foundation multilingual models win on coverage and ops simplicity. Pick per use case.
- Deep multilingual modelling work (new language training, dialect-aware models) is `nlp-engineer` / `fine-tuning-engineer` territory; the handoff contract from Chapter 4 applies.

Next chapter: the boundary with `nlp-engineer`, `model-evaluation-engineer`, and `training-pipeline-engineer`, and the deferral contracts that go with them.
