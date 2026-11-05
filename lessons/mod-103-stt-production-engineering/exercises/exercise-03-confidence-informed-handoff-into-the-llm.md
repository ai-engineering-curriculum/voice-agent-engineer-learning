# exercise-03: Confidence-Informed Handoff into the LLM

**Estimated effort:** 3 hours · **Output:** a calibration study + a handoff function + an A/B evaluation

## Objective

Build the confidence-informed STT → LLM handoff from Chapter 3 against real data. By the end you will have:

1. A **reliability diagram and ECE** for your provider's confidence scores on your audio.
2. A **calibrator** (Platt or isotonic) that corrects those scores.
3. A **handoff function** implementing at least two of Chapter 3's patterns, composed, with clean rate-limiting.
4. A **comparative A/B evaluation** against a naive pass-through baseline on your audio.

This is the exercise that turns "the ASR is sometimes wrong" from a product complaint into a measurable, tunable system.

## Prerequisites

- [Chapter 3 — The Confidence-Informed Handoff into the LLM](../03-confidence-informed-handoff-into-the-llm.md).
- The streaming client from [exercise-02](exercise-02-streaming-vs-batch-decision-and-contract-walkthrough.md), or an equivalent setup that produces per-word confidences.
- 50–200 utterances with ground-truth transcripts. These can be the in-domain gold set you'll build for exercise-05, a subset of LibriSpeech test-clean, or a labelled slice of your own recordings. Larger is better for calibration; smaller is fine for a first pass.
- Python 3.10+ with `numpy`, `scikit-learn` (for Platt / isotonic), `matplotlib`, and `jiwer`.
- Access to an LLM (any provider; `claude-haiku-4-5` or an equivalent fast model works well).

## Steps

### Part A — Measure baseline calibration (45 minutes)

1. Run your audio through the STT and capture, per word: provider confidence, word text, word timings, and whether the word matches the reference transcript at that position. (For "position" alignment, use `jiwer`'s alignment output or equivalent — align the hypothesis to the reference, then propagate the match/no-match label down to each word.)
2. Compute the **reliability diagram** with 10 bins. Plot it; include the diagonal (perfect calibration) and the bin counts.
3. Compute the **Expected Calibration Error (ECE)**. Write down whether it's under 0.03 (good), under 0.10 (acceptable), or worse (poor).
4. Stratify the diagram: separate reliability lines for (a) content words vs. filler words; (b) numbers vs. non-numbers; (c) proper nouns vs. common nouns if you can label them. Note any stratum that's materially worse than the pooled number.

### Part B — Fit a calibrator (30 minutes)

5. Split your labelled data 70 / 30 train / test.
6. On the train split, fit two calibrators:
   - **Platt scaling** — `sklearn.linear_model.LogisticRegression` on `(raw_confidence, correct_label)`. The sigmoid output is your corrected confidence.
   - **Isotonic regression** — `sklearn.isotonic.IsotonicRegression`. Same inputs; non-parametric.
7. Evaluate both calibrators on the test split: ECE, Brier score, and the re-drawn reliability diagram. Pick the better one and justify the choice in one sentence.
8. Save the fitted calibrator as a file (`joblib` or `pickle`). The handoff function in Part C loads it.

### Part C — Implement the handoff function (45 minutes)

9. Implement a `stt_to_llm_handoff(stt_final, dialogue_state)` function with the Chapter 3 shape. At minimum include:
   - **Pattern 1** (reject-and-ask-again) with a calibrated utterance-level threshold. Rate-limit: no more than one consecutive clarification request.
   - **Pattern 2** (confidence-weighted token masking) that wraps low-confidence spans in a provider-agnostic marker and documents the marker in the system prompt.
   - The function must be **pure**: given the same `stt_final` and `dialogue_state`, it returns the same decision. Side effects (calling the LLM, calling TTS) live outside.
10. Write a short system prompt that tells the LLM how to interpret the uncertainty markers, including the rule that a load-bearing low-confidence span triggers a targeted clarification. "Load-bearing" is domain-specific — define your criteria (numbers, named entities, etc.) explicitly in the prompt.
11. Unit-test the handoff on at least six crafted inputs:
    - A high-confidence transcript (passes through).
    - A low-confidence utterance with no prior clarification (rejects).
    - A low-confidence utterance after a just-rejected utterance (falls through — does not re-ask).
    - A high-confidence transcript with one low-confidence number word (masked, LLM asked to confirm).
    - A high-confidence transcript with one low-confidence filler word (passes through; filler is not load-bearing).
    - An N-best alternative case (if your provider emits N-best) where the top-1 looks implausible and the top-3 contains the likely correct version.

### Part D — A/B evaluate (60 minutes)

12. Build a tiny evaluation harness: for each labelled utterance, run three variants and record the LLM's response plus whether the response correctly understood the intent (binary; labelled by you, or by a rubric).
    - **Baseline:** pass raw transcript to LLM, no confidence information.
    - **Pattern 1 only:** reject-and-ask-again with calibrated threshold.
    - **Pattern 1 + Pattern 2 (full handoff).**
13. Compute, for each variant:
    - Rate of correct intent capture.
    - Rate of clarify turns issued.
    - Rate of silent LLM confabulations (transcript was wrong, agent responded as if it were right, user was not asked to repeat).
14. Produce a two-row-by-three-column table reporting the three metrics for the three variants, with bootstrap CIs on at least the headline "correct intent capture" row.
15. Write a 1–2 paragraph conclusion. Where does Pattern 2 earn its keep? Where is Pattern 1 alone sufficient? Is there a stratum where the handoff backfires?

## Starter guidance

- The `jiwer` alignment output includes the operation (`equal`, `substitute`, `insert`, `delete`) for each pair; use `equal` to label correctness per word.
- For Platt vs. isotonic: Platt is a strong default for small data (<1k samples). Isotonic is better for larger samples (~3k+) because it fits a non-parametric curve. If you have 100 utterances, pick Platt.
- When implementing Pattern 2, use a marker format that's unambiguous and short — `<?word|0.42?>` or similar. Document it in the system prompt. Avoid markdown or HTML markers; they interfere with other formatting.
- Rate-limiting is in `dialogue_state`, not the handoff function. The handoff function reads `dialogue_state.last_turn`; something upstream of it writes to that state.
- For Part D, if you don't want to hand-label LLM responses, use an LLM-as-judge: a second LLM call asking "does this response correctly capture the user's intent `{intent}`?" This is noisier than human labels — document it.

## Acceptance criteria

- [ ] Reliability diagram + ECE for raw provider confidences on your audio.
- [ ] Platt and isotonic calibrators trained, evaluated, and the better one selected with justification.
- [ ] Pure `stt_to_llm_handoff` function implementing Patterns 1 and 2, with rate-limiting via `dialogue_state`.
- [ ] System prompt that documents the uncertainty markers and the domain-specific "load-bearing" rule.
- [ ] Six unit tests covering the handoff decision space.
- [ ] A/B evaluation: three variants × three metrics, table with bootstrap CIs, written conclusion.

## Stretch goals

- **Add Pattern 3 (homophone hints).** For a domain-relevant list of homophones your users produce, implement an LLM post-correction prompt with explicit rules. Compare to a version that pushes those homophones through the STT's keyterm list instead (preview of exercise-04).
- **Add Pattern 4 (N-best rerank).** If your provider emits N-best, add a stage where the LLM picks among the top-K given the dialogue context. Measure whether it helps on low-confidence utterances.
- **Stratified calibration.** Fit a separate calibrator per accent / per channel / per audio quality band; compare the handoff behaviour to a single-calibrator baseline. Where does per-stratum calibration earn its keep?
- **The speech-LLM comparison.** Run the same audio through an audio-native LLM (OpenAI `gpt-4o-audio-preview`, Google Gemini audio, or similar) and compare on the same three metrics. Where does the speech-LLM beat the text pipeline? Where does it lose? This is the preview of the architectural choice Chapter 3 names.

## Keep it

The calibration script and the handoff function become reusable modules. The calibrator in particular is a per-deployment artefact — you refit it when you change provider, change language, change audio-channel mix, or when drift alerts in Module 112 fire. Treat it as a tracked model, not a script output.
