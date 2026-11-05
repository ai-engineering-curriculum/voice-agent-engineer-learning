# 03. The Confidence-Informed Handoff into the LLM

## Why this chapter

Once the STT emits a final transcript, you hand a string of text to an LLM. If you hand it the text alone, you have thrown away the signal that **the STT was unsure**. The LLM then confabulates — it rationalises "fifty four dollars" and "five four dollars" and "fifty fore dollars" into whatever the prompt context plausibly makes them, and you never learn that the acoustic signal said something else.

The confidence-informed handoff is the discipline of keeping some of the STT's uncertainty alive into the LLM, so that the agent can ask the right clarifying question instead of guessing. It is where almost all the preventable "the bot heard me wrong" bugs get fixed.

This chapter covers four handoff patterns and the decision rule for when to abandon the pipeline altogether in favour of a speech-LLM.

## What the STT actually gives you

Every production STT provider returns at least one of:

- A **per-word confidence** in `[0, 1]` on the final transcript.
- A **per-segment / per-utterance confidence**.
- An **N-best list** of alternative transcripts, each with a score.
- Sometimes **lattice / word-graph** output (richer than N-best, mostly in open-weights territory — Kaldi, NeMo).

Confidences are not calibrated out of the box. Before you can threshold on them, calibrate — see the Calibration section below.

The other fact worth stating: **a word-level confidence is a very weak proxy for "is this sentence right."** A single low-confidence word often means the sentence is salvageable with a clarification; three low-confidence words in a row often mean the whole acoustic signal was bad and nothing in the transcript is trustworthy. Design the handoff patterns to reason on both scopes.

## Pattern 1 — Reject and ask again

The simplest and often most effective pattern: if the transcript is below a confidence threshold, do not send it to the LLM. Instead, emit a clarification turn to the user.

```python
MIN_UTTERANCE_CONF = 0.65   # calibrated — see below

def handoff(transcript, word_confidences):
    utterance_conf = min(word_confidences)  # or geometric mean, or weighted
    if utterance_conf < MIN_UTTERANCE_CONF:
        return speak("Sorry, I didn't catch that — could you say it again?")
    return llm.respond(transcript)
```

Design choices to notice:

- **Aggregate the per-word confidences before thresholding.** `min`, geometric mean, or a learned aggregator are all defensible; the right choice depends on your data. `min` is pessimistic and the easiest to start with.
- **Rate-limit the clarification turn.** Two "I didn't catch that" messages in a row make the user hang up. If a second ask fails, fall back to a human or a scripted next step; don't ask a third time.
- **Log every rejection with the audio.** These are the primary source of the retraining data you'll pass to `fine-tuning-engineer` in [Chapter 4](04-domain-adaptation-vocab-keyterms-finetune.md).

When to reach for this pattern: short utterances, low-ambient-noise channels, tasks where a user repeat is cheap (one or two words of input). It is the floor every handoff should start from.

## Pattern 2 — Confidence-weighted token masking (hints into the prompt)

Instead of rejecting the whole utterance, forward it to the LLM **with explicit markers around uncertain spans**. The LLM gets to see which words it should not over-trust.

```
User said (STT confidence annotations):
"I'd like to transfer <?FIVE FORTY?|0.42> dollars to <?SAVINGS?|0.58>"

If any annotated span is critical to the task, ask a targeted clarification
for just that span, not the whole utterance.
```

The LLM is told, via the system prompt, that `<?…?|conf>` wraps words the STT is uncertain about and should trigger a targeted clarification if the span is load-bearing for the task. This is dramatically better than the reject-everything pattern for long utterances where most of the sentence was heard correctly.

A pragmatic variant: pass the **N-best list** for just the uncertain spans. The LLM can pick among the alternatives when context makes one strictly more plausible ("transfer five forty dollars to savings" and "transfer 540 dollars to savings" are the same; "transfer 5:40 dollars" isn't an amount).

This pattern is where provider output fields earn their keep — if the provider doesn't emit per-word confidence or N-best, you are locked out of it. Factor this into the selection in [Chapter 1](01-stt-provider-landscape-and-scoring.md).

## Pattern 3 — Homophone disambiguation hints

Many ASR errors are **homophones** or near-homophones that are only resolvable from context: "to / too / two", "there / their", "fore / four / for", proper names ("Mark / Marc / Marque"). These are usually *not* flagged as low confidence by the STT, because acoustically the model heard what it heard and is certain about that.

Three approaches:

- **LLM post-correction prompt.** A system prompt that tells the LLM "the transcript came from a streaming ASR; correct homophones only when the correction unambiguously fits the context, and never silently alter proper nouns or numbers." Cheap; works well when the LLM is strong and the context is rich.
- **Phonetic rewrite pass.** Feed the LLM the ARPAbet / IPA pronunciation of each content word along with the surface text. Lets the model reason about alternatives it would otherwise never consider. Expensive on tokens; only used in domains where it demonstrably pays off.
- **Domain-constrained decoding at the STT.** Push a keyterm / phrase list of the names and phrases that *matter* for your domain into the STT itself — see [Chapter 4](04-domain-adaptation-vocab-keyterms-finetune.md). This is the strictly-better option for domain-specific homophones; it fixes the problem upstream.

Order of operations: **fix it at the STT first (Pattern 3c), then at the LLM post-correction level (Pattern 3a), then by exposing N-best to the LLM (Pattern 2).** Not the other way around — every layer of post-correction is a layer of latency and a surface for new errors.

## Pattern 4 — Acoustic-feature-aware reranking

A heavier pattern, used when the above aren't enough and you can afford extra compute.

Idea: ask the LLM for its top-K candidate interpretations of the user's intent. Score each candidate against the acoustic evidence (either the N-best list from the STT, or a direct pass through an audio-text alignment model) and pick the one that is **both** linguistically plausible and acoustically compatible.

Flavours you'll see:

- **LLM + N-best rerank.** The LLM is prompted with the top-5 ASR transcripts and asked to pick the one most consistent with the dialogue state and the business constraints. Implementable today with any provider that emits N-best.
- **Audio-text rescoring with a cross-encoder.** The transcript is passed to a smaller model (e.g. a Whisper encoder + a classification head) along with the audio; the model emits a "do these match" probability. This becomes an additional feature for the LLM's rerank. Open-weights territory; non-trivial to maintain.
- **Lattice rescoring.** For open-weights ASR stacks that emit a word lattice (Kaldi, NeMo), a domain-specific LM can be applied post-decode. This is strictly `nlp-engineer` territory at L30; name it as an escalation option, not a build.

Acoustic reranking is a niche pattern. Reach for it when you have measured that Patterns 1–3 leave money on the table, and when the extra compute cost per turn is acceptable.

## When to skip the pipeline and defer to a speech-LLM

All four patterns above add machinery to preserve acoustic information through a text bottleneck. There is a different architectural choice: **remove the bottleneck.** Feed audio directly into an audio-native LLM (OpenAI `gpt-4o-realtime`, Google Gemini audio, Kyutai Moshi, Qwen2-Audio), which produces the response without ever committing to a single text transcription.

The speech-LLM path has real advantages:

- **No text-bottleneck information loss.** Prosody, pauses, hesitation markers, non-lexical sounds (laughter, sighs, groans) survive into the model's reasoning.
- **No confidence-handoff gymnastics.** Uncertainty is handled natively by the model.
- **Lower end-to-end latency** in practice — you've eliminated the STT ↔ LLM round trip.

And real costs:

- **Opacity.** There is no human-readable transcript to log, redact, QA, or train evaluators against. Many compliance regimes assume a transcript exists; several regulators have had to be educated on this.
- **Provider lock-in.** The audio-native model and the business logic are tied together; you can't swap STT and LLM independently.
- **Multilingual and accent coverage is still closing the gap with best-in-class ASR**, especially outside of top-tier languages. Measure before you commit.
- **Cost.** Audio tokens are not free, and providers price audio input at a premium.

The decision rule:

- **Text-STT pipeline** if you need auditability, if you need the transcript as an archival artefact, if the task crosses many languages with uneven coverage, if your LLM needs to be swapped independently of the STT, or if the compliance envelope demands it.
- **Speech-LLM** if latency dominates, if the task is high-prosody (empathy-sensitive, emotion-sensitive, pause-sensitive — like therapy triage, sales qualification, consumer coaching), and if the provider's audio coverage matches your users.
- **Both, side by side,** for the first 90 days of a speech-LLM deployment, so you can produce a text transcript via a parallel STT leg for compliance and QA while the speech-LLM runs the agent. This is the pattern most prudent production teams actually ship.

Module 105 goes into speech-LLM internals; this chapter cares that you **name the choice explicitly** in your handoff design.

## Confidence calibration

Before you threshold on confidence, calibrate. The methodology:

1. Collect ≥500 utterances with ground-truth transcripts from your domain.
2. Run them through the STT. For each word, record the provider's confidence and whether the word was correct against ground truth.
3. Bucket confidences into 10 bins (`[0.0, 0.1)`, `[0.1, 0.2)`, …, `[0.9, 1.0]`). Compute the empirical correctness rate in each bin.
4. Plot the **reliability diagram**: x = reported confidence, y = empirical correctness. A perfectly calibrated STT lies on the diagonal.
5. Summarise with **Expected Calibration Error (ECE)** — the weighted mean distance from the diagonal across bins. ECE above ~0.1 is poor; below ~0.03 is good.

If the STT is miscalibrated, apply **Platt scaling** (fit a logistic on the raw confidence) or **isotonic regression** (non-parametric, more flexible, needs more data). Both are one-liners in `scikit-learn`. The corrected confidence is what you threshold on.

Methodology depth beyond this — temperature scaling for multi-class problems, calibration under class imbalance, group-wise calibration across accents / languages — is `model-evaluation-engineer` (L30) territory. See [Chapter 7](07-boundary-to-nlp-eval-and-training-engineers.md).

## A worked handoff pipeline

Shape of the function that ties the patterns together in a production agent.

```python
def stt_to_llm_handoff(stt_final, dialogue_state):
    """
    stt_final: provider final-transcript object with word timings, per-word
               confidence, and (optionally) N-best alternatives.
    """
    words = stt_final.words
    confs = calibrated_confidences([w.confidence for w in words])
    utterance_conf = geometric_mean(confs)

    # Pattern 1: hard reject if the utterance as a whole is too uncertain,
    # and we haven't just asked them to repeat.
    if utterance_conf < 0.55 and dialogue_state.last_turn != "clarify":
        return clarify("Sorry, I didn't catch that — could you say it again?")

    # Pattern 2: annotate uncertain spans for the LLM.
    annotated = annotate_uncertain_spans(words, confs, threshold=0.70)

    # Pattern 3a: tell the LLM in the system prompt that homophone correction
    # is allowed only when context makes it unambiguous.
    llm_input = render_prompt(annotated, dialogue_state)

    # Pattern 2 (N-best variant): if the STT emits N-best, also pass the top 3
    # alternatives for spans that are both low-confidence and semantically
    # load-bearing (numbers, names, nouns tagged as entities).
    if stt_final.nbest:
        llm_input = attach_nbest_for_load_bearing_spans(
            llm_input, stt_final.nbest, words, confs
        )

    return llm.respond(llm_input)
```

Three decisions a reviewer should ask about:

- **The utterance-reject threshold and the word-annotation threshold are different numbers.** The reject is a hard business decision; the annotation is a dial.
- **"Load-bearing" is domain-specific.** In a banking agent, numbers and account names are load-bearing; in a scheduling agent, dates and times. Make the list explicit in config, not inferred.
- **The clarify-rate-limit lives in `dialogue_state`**, not in the handoff function. The handoff function is pure: given the same state and transcript, same decision.

## Common anti-patterns

- **Thresholding on raw provider confidence.** Without calibration, your 0.80 threshold means something different in English vs. Spanish, in quiet vs. noisy audio, in the first second of a call vs. the fiftieth. Calibrate per language and per deployment.
- **Clarifying too aggressively.** Users abandon agents that ask "could you say that again" twice. Rate-limit; fall through to a safer path (human handoff, scripted question) on the second miss.
- **Ignoring the N-best or confidence fields** because the first draft of the pipeline didn't need them. Picking a provider that doesn't emit them traps you in Pattern 1 forever.
- **Pattern-sprawl.** Don't layer all four patterns in parallel — compound complexity, compound latency, hard to debug. Pick the minimum that moves your metric.
- **Fixing homophones at the LLM instead of at the STT.** If "deductible" always comes back as "deducted bill" in your transcripts, that's a keyterm-prompting problem, not an LLM problem. Fix it upstream (Chapter 4).

## Summary

- Four handoff patterns: hard reject, confidence-weighted annotation, homophone hints (prefer upstream over LLM-level correction), acoustic-feature reranking. Stack from simple to complex only as your metric demands.
- Speech-LLMs are the architectural alternative. The decision rule is auditability + language coverage vs. latency + prosody preservation.
- Calibrate confidence before you threshold on it. ECE on your own corpus, Platt or isotonic for the correction.
- The handoff function is the boundary where the STT contract (Chapter 2) meets the LLM prompt. Keep it explicit, small, and testable.

Next chapter: shipping domain adaptation — custom vocab, keyterm prompting, lexicons, and when to escalate to a fine-tune — against the WER delta you can defend.
