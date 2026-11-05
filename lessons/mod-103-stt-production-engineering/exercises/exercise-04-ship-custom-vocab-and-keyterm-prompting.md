# exercise-04: Ship Custom Vocab and Keyterm Prompting

**Estimated effort:** 2.5 hours · **Output:** a versioned keyterm list + a before/after WER report + an A/B tooling sketch

## Objective

Climb the first two rungs of Chapter 4's adaptation ladder: ship a **custom vocabulary** and a **keyterm-boosted list**, and measure the WER delta you can defend. You will not run a fine-tune — this exercise is specifically about the cheap, fast, provider-feature adaptations that solve 80 % of domain-vocabulary problems without an escalation to `fine-tuning-engineer`.

You will finish with a keyterm list in version control, a measured WER delta on the terms that matter, and the beginnings of a per-term A/B pipeline.

## Prerequisites

- [Chapter 4 — Domain Adaptation](../04-domain-adaptation-vocab-keyterms-finetune.md).
- A streaming or batch STT provider that supports custom vocabulary or keyterm prompting. Deepgram (Nova-3 keyterm), AssemblyAI (`word_boost`), Google STT v2 (`phrase_sets` with boost), Azure Speech (phrase lists), AWS Transcribe (custom vocabulary), and Whisper's `initial_prompt` are all acceptable. Pick one you have a working client for.
- 30–100 utterances of your own audio containing **domain vocabulary** — product names, brand names, acronyms, drug names, proper nouns, code identifiers, whatever is domain-specific to the deployment you picked in exercise-01. Transcribed with ground truth.
- Python 3.10+ with `jiwer` for scoring.

## Steps

### Part A — Choose a domain and seed the term list (30 minutes)

1. Fix a **domain** in one sentence. ("US pharmacy refill assistant." "German banking customer support." "Field-service technician dictation for industrial elevators.")
2. Write a first-pass keyterm list of 20–60 terms. Include:
   - Brand names, product names, SKU prefixes.
   - Domain-specific vocabulary (medications, tools, parts, policy types).
   - Proper nouns that come up in your audio (user names, agent names, store names).
   - Technical acronyms the ASR will otherwise spell out letter-by-letter.
   - **Do not include** common English nouns, verbs, or filler words.
3. Commit the list to a file (`keyterms.v1.yaml` or similar) in version control, with a one-paragraph header: what domain, what date, what provider, any per-term boost weights you're setting, and your rationale.

### Part B — Measure the baseline (30 minutes)

4. Run your audio through the STT with **no custom vocabulary / keyterm config**. Capture the hypothesis transcripts.
5. Score with `jiwer`. Use a documented normaliser (`whisper.normalizers.EnglishTextNormalizer` for English; its multilingual counterpart otherwise). Compute:
   - Overall WER with a bootstrap 95 % CI.
   - **Per-term recall** — for each term in your keyterm list, how often does the hypothesis contain it when the reference does? This is the number you expect to move.
   - **Per-term precision** — same question, reversed; sanity check that the term isn't already over-firing.
6. Record the top 10 confusions in the baseline (which word does the ASR consistently produce instead of your intended term?). `jiwer`'s alignment output supports this directly.

### Part C — Ship the keyterm list and re-measure (45 minutes)

7. Re-run the audio through the STT **with your keyterm list** applied. Same audio, same model, same everything else. Capture the hypothesis transcripts.
8. Re-score: overall WER (with CI), per-term recall, per-term precision.
9. Produce the delta table:

   | Term | Baseline recall | Keyterm recall | Baseline precision | Keyterm precision | Baseline → Keyterm WER (local) | Verdict |
   |---|---|---|---|---|---|---|
   | …  | | | | | | helps / neutral / hurts |

10. **Flag the regressions.** A keyterm that improves recall but tanks precision (the ASR now hears your brand everywhere) is a problem. A keyterm that doesn't move its recall at all suggests it's not pronounced the way you listed it — a phonetic lexicon might help (Chapter 4's Technique 3, stretch goal).

### Part D — Build the A/B tooling sketch (30 minutes)

11. Write a short script that, given two keyterm-list versions, runs the same audio through both and emits the delta table. No infrastructure — just `python score.py keyterms.v1.yaml keyterms.v2.yaml audio_dir/`.
12. Use it to produce a v2 of your list, applying three edits:
    - Remove any term with a precision regression.
    - Boost any underperforming term more aggressively (if the provider supports per-term weights).
    - Add 2–3 new terms you learned from the baseline confusions.
13. Produce the v1 → v2 delta table. Commit `keyterms.v2.yaml` with a changelog summarising the three edits and their justifications.

## Starter guidance

- Keep the keyterm list **small** — 50 terms is usually plenty, 500 is almost always too many. More ≠ better.
- For Google STT, start boost around 10 (the docs' suggested range is 0–20); for AWS Transcribe, start at MEDIUM. Over-boosting is the single most common mistake.
- For Whisper, the equivalent of keyterm prompting is the `initial_prompt` parameter — a short piece of text containing the terms. Treat it as a hint, not a weighting; it behaves differently from the commercial providers' explicit boosts.
- Normalise your reference transcripts before scoring. The baseline will look "better" than it is if trailing punctuation differs.
- Per-term recall / precision needs the aligned output from `jiwer`, not just the overall WER. See `jiwer.process_words` for the alignment data.
- Resist the urge to tune on your test set. If you want multiple iterations (v1 → v2 → v3), hold out a frozen eval set for the final number.

## Acceptance criteria

- [ ] Domain stated in one sentence; 20–60-term keyterm list v1 committed with header.
- [ ] Baseline WER + per-term recall / precision measured on the frozen audio, with bootstrap CIs on WER.
- [ ] Top-10 baseline confusions captured.
- [ ] Keyterm-applied WER + per-term recall / precision measured on the same audio.
- [ ] Delta table showing per-term helps / neutral / hurts classification.
- [ ] v2 of the list committed with changelog + v1→v2 delta.
- [ ] A/B script that takes two list versions and emits the delta — reusable for future iterations.

## Stretch goals

- **Add a phonetic lexicon rung.** For 3–5 terms where keyterm prompting alone did not move recall, provide a pronunciation hint (IPA or ARPAbet, or the provider-specific `SoundsLike` syntax). Measure the additional lift. Which terms are helped most by phonetic hints?
- **Measure latency delta.** Some providers decode slightly slower when a large keyterm list is attached. Measure p50 / p95 first-partial and endpoint latencies for v0, v1, v2 and plot them. Did any rung cost you more than it was worth?
- **Build a KPI translation.** For your three biggest-impact terms, estimate the business-KPI consequences of the recall improvement — fewer clarifications, lower AHT, fewer transfers. The method is in Chapter 5; this exercise is a dress rehearsal.
- **Draft the fine-tune hand-off packet** (even without sending it). If your WER after v2 still misses target, write the handoff artefact Chapter 4 specifies: problem statement, corpus characterisation, acceptance criteria, pipeline integration. Having to write this will often reveal that you're one more keyterm iteration away from the target.

## Keep it

The versioned keyterm files + the A/B tooling + the delta tables become part of the pipeline's living config. Every quarterly review of the agent's error analysis closes with an updated keyterm list and a new delta row. Treat the list like a dataset — reviewable, diff-able, rollback-able.
