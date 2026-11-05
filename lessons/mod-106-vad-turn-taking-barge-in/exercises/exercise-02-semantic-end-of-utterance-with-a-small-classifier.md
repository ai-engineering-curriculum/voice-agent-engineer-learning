# exercise-02: Semantic End-of-Utterance with a Small Classifier

**Estimated effort:** 3 hours · **Output:** a semantic EOU classifier, a variable-hangover commit loop, and an A/B evaluation against silence-only endpointing on your corpus.

## Objective

Build the EOU layer that sits on top of Silero (Exercise 1). Specifically: author a small-LLM EOU classifier, integrate it into a variable-hangover commit loop, and show that on utterances containing mid-utterance pauses the hybrid pipeline interrupts the user less often than silence-only endpointing — without blowing up the median bot-response gap.

The pattern you will build is the Chapter 2 reference: silence duration is the hard floor; the EOU probability modulates the hangover; a short, confident EOU fires the turn quickly, a weak EOU holds the turn open for a full second.

## Prerequisites

- [Chapter 2 — End-of-Utterance Detection Beyond VAD](../02-end-of-utterance-detection-beyond-vad.md).
- Exercise 1 completed: you have a `StreamingVAD` and a corpus.
- API access to one small-latency LLM (OpenAI `gpt-4o-mini`, Anthropic `claude-haiku-*`, Google `gemini-flash-*`, or an equivalent). The classifier will be a one-token call; cost per exercise is dollars, not tens of dollars.
- A streaming ASR (Deepgram, AssemblyAI, Google STT, OpenAI `gpt-4o-mini-transcribe`, or self-hosted `faster-whisper` with a partial-emit loop). Either is fine; the ASR emits partial transcripts and the EOU consumes them.

## Requirements

### Part A — expand the corpus with mid-utterance pauses

Exercise 1's corpus covers VAD cases. Extend it with EOU-targeted cases. Add **at least 20 clips, each ≥ 4 s long**, covering:

- **Mid-utterance thinking pause (≥ 500 ms).** "So I was thinking — [pause] — three options come to mind." Label the ground-truth turn end at the end of the complete utterance, not at the pause.
- **False-starts.** "I want to — no, actually, could you — let me think." Label the ground-truth turn end at the end.
- **Trailing hesitation.** "I need to book a flight for tomorrow, um…" Label two candidate ends: the first at "tomorrow," the second at the end of "um." Record which your pipeline fires on; both can be defensible.
- **Clear ends.** "Book a flight to Tokyo." Clean, prosodically-clear endings. These are the fast-fire cases.
- **Backchannel-like.** A clip with a short "mm-hm" mid-turn by the speaker themselves (rare but worth having).

Document the labelling policy in the corpus README.

### Part B — implement the EOU classifier

Build `EOUClassifier.classify(partial_transcript: str) -> float` returning `P(end of turn)`.

Implement two variants:

**Variant 1: small-LLM classifier.**

```python
PROMPT = """You decide whether a short transcript is a plausible complete utterance a conversational speaker might pause on, or mid-utterance. Output exactly one character: 'Y' if the speaker could reasonably be done, 'N' if they are clearly in the middle of an utterance.

Transcript:
{transcript}"""

def classify(transcript: str) -> float:
    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        temperature=0,
        max_tokens=1,
        logprobs=True,
        top_logprobs=2,
        messages=[{"role": "user", "content": PROMPT.format(transcript=transcript)}],
    )
    # Convert the Y/N token logprobs into a calibrated probability.
    return _logprob_to_p_eou(resp)
```

Convert the top-logprob output into a single `P(Y) / (P(Y) + P(N))` probability. Use `temperature=0`, `max_tokens=1`, and enable logprobs. Cache aggressively: if the pipeline polls the classifier every 200 ms during silence, the same partial transcript may be classified multiple times — memoise on the transcript string.

**Variant 2 (baseline): rule-based.** A handful of hand-written patterns that return a probability:

- Transcript ends with `. ! ?` and a token count ≥ 3 → `P(EOU) = 0.8`.
- Transcript ends with a common discourse continuation token (`and`, `so`, `but`, `because`, `if`, `um`, `uh`, `like`, `that`, `the`) → `P(EOU) = 0.1`.
- Transcript word count < 2 → `P(EOU) = 0.2`.
- Otherwise → `P(EOU) = 0.5`.

You will compare the two.

### Part C — the variable-hangover commit loop

Combine `StreamingVAD` from Exercise 1 with the classifier:

```python
def should_commit_turn(silence_ms: int, p_eou: float) -> bool:
    if silence_ms < 200: return False
    if p_eou >= 0.8 and silence_ms >= 200: return True
    if p_eou >= 0.5 and silence_ms >= 500: return True
    if silence_ms >= 1200: return True
    return False
```

The policy has three bands; tune the thresholds against the corpus (part D).

Operational details:

- Only query the classifier when VAD declares silence has started. Not every frame.
- Classify on the stabilised transcript — words that have not been revised for the last 200 ms of ASR output.
- If the ASR has not emitted any words yet (user has not spoken enough for partials), skip classification and fall back to the silence floor (`1200 ms`).

### Part D — A/B against silence-only endpointing

Build a replay harness: takes a clip from the corpus, feeds audio into the pipeline at realtime rate, logs the commit events. Run each clip under three configurations:

1. **Silence-only.** Fixed `silence_duration_ms = 500` (the Chapter 1 default).
2. **Rule-based EOU.** Variable-hangover loop with Variant 2 classifier.
3. **Small-LLM EOU.** Variable-hangover loop with Variant 1 classifier.

For each configuration, compute on the whole corpus:

- **False-turn-grab count.** Commits that fired during a mid-utterance pause (not at the labelled turn-end).
- **Missed-EOU count.** Clips where no commit fired within 1500 ms of the labelled turn-end.
- **Median + p95 commit gap.** Commit timestamp minus labelled turn-end, over correctly-fired commits.
- **Classifier calls per clip.** Cost proxy.

Report the numbers in a 3-row comparison table.

### Part E — a one-page write-up

Answer:

- Did the small-LLM EOU reduce false-turn-grabs on the mid-utterance-pause clips?
- What was the median latency cost compared with silence-only?
- Did the rule-based variant close most of the gap? Where did it still miss?
- At current pricing, what is the per-session cost of the LLM EOU in your pipeline? Extrapolate to 1 M sessions / month.
- One failure mode you did not anticipate before the eval.

## Starter guidance

- **Cache the classifier. Keep the pre-canned prompt stable.** Prompt caching will recover most of the cost of repeat calls across sessions; verify the cached-tokens counter on your provider.
- **Guard the input length.** Truncate the transcript to the last ~200 characters before the classifier call. Longer is more expensive and rarely more accurate.
- **Stabilise the suffix.** Call the classifier on the stabilised partials, not on every ASR event. Deepgram's `is_final`, AssemblyAI's `final`, or a simple "last word unchanged for 200 ms" heuristic all work.
- **Realtime replay rate matters.** The commit-gap number is wrong if you replay faster than realtime; partial-ASR timing changes.
- **Log the per-frame inputs that drove the commit.** `vad_silence_ms`, `p_eou`, `partial_transcript`. When you disagree with the pipeline's decision later, the log is what tells you why.
- **Don't trust a single run.** Classifier latency has variance; run each config at least twice and average.

## Acceptance criteria

- [ ] Corpus extended with ≥ 20 EOU-targeted clips, each labelled with ground-truth turn-end position.
- [ ] `EOUClassifier` implementation in both variants (rule-based + small-LLM), with the small-LLM variant producing a calibrated `[0, 1]` probability from logprobs.
- [ ] Variable-hangover commit loop combining VAD silence with EOU probability, matching the policy sketch above.
- [ ] Replay harness that runs a clip through the pipeline at realtime rate and logs the commit events, for all three configurations.
- [ ] 3-row comparison table on the corpus, covering false-turn-grab count, missed-EOU count, median and p95 commit gap, classifier calls per clip.
- [ ] One-page write-up answering the five questions above.

## Stretch goals

- **LiveKit turn-detector integration.** Install the LiveKit Agents SDK, enable the open-weights turn-detector, and run it as a fourth configuration. Compare the three metrics; comment on latency (local-model vs. LLM) and quality.
- **OpenAI Realtime `semantic_vad`.** Compare against `turn_detection.type: "semantic_vad"` end-to-end. (Different stack; you will need to wire the clip into a Realtime session.) Note the observability gap — you cannot see the server's probability, only the decision.
- **Calibration diagnostic.** Plot the small-LLM classifier's output probability against the empirical "fraction of these calls followed by a true turn-end." Is the model's P(EOU) well-calibrated? If not, apply isotonic regression (`sklearn.isotonic.IsotonicRegression`) to recalibrate.
- **Multilingual.** Run the same pipeline on a handful of non-English clips (Japanese or Spanish). Does the small-LLM classifier work out of the box? Does the rule-based variant need language-specific tokens?

## Non-goals

- Barge-in. That is Exercise 3.
- Backchannel classification. That is Exercise 4.
- Training a dedicated turn-end model from scratch. Use the LLM or the rules; the training pipeline is `training-pipeline-engineer` scope.

## Keep it

The replay harness and the labelled EOU corpus are the two artefacts Exercise 5 will build on. Keep both in the repo; treat the harness as a production-adjacent tool, not a throwaway script.
