# exercise-04: Backchannel vs. Turn-Grab Classifier

**Estimated effort:** 2.5 hours · **Output:** two classifier implementations (heuristic + LLM), a labelled mini-corpus of backchannels and turn-grabs, and an integration test showing the barge-in path aborts correctly on backchannels.

## Objective

Design, train (lightly), and evaluate the classifier that decides, mid-barge-in, whether the user said "mm-hm" (don't cancel) or "wait, no" (do cancel). Integrate it into the Exercise 3 barge-in path and show that false-turn-grabs drop without introducing missed-turn-grabs.

The point: ship the simplest thing that works (heuristic), measure its errors, promote to a model only where the errors live. Shipping a trained classifier where a 15-line heuristic suffices is the expensive mistake this exercise is designed to prevent.

## Prerequisites

- [Chapter 4 — The Backchannel-vs-Turn-Grab Classifier](../04-backchannel-vs-turn-grab-classifier.md).
- Exercise 3 completed: you have an interruptible player, an `on_interrupt()` path, and a barge-in corpus.
- A streaming ASR that can emit partial transcripts fast enough to be useful < 500 ms after speech start (Deepgram, AssemblyAI, Google STT, `gpt-4o-mini-transcribe`, or `faster-whisper` streaming).
- Access to a small LLM for the LLM-classifier variant.

## Requirements

### Part A — build the backchannel vs. turn-grab corpus

Collect **at least 40 short clips** (0.3–2.0 s each) centred on a user utterance that would occur during bot-speaks. Stratify roughly:

- **Lexical backchannels (10 clips).** "mm-hm", "yeah", "right", "okay", "sure", "uh-huh", "oh", "got it", "I see", "hm". Vary speaker voice and prosody.
- **Prosodic backchannels (5 clips).** Short laughs, small aspirations, a soft "oh" of mild surprise.
- **Ambiguous short utterances (5 clips).** "wait" (as backchannel of surprise), "really?" (phatic), "oh really?" — labels are judgement calls; record your rationale.
- **Clear turn-grabs (15 clips).** "wait, no," "stop," "actually," "hold on," "that's wrong," "can I ask something," longer corrections ("no, I meant on Tuesday").
- **Edge cases (5 clips).** 700–900 ms "mm-hmmm" (long backchannel), 400 ms "no" (short turn-grab), speakerphone-leak clips with the bot's voice bleeding through.

Label each clip with:

- `audio_duration_ms` (ground truth; what a 1-shot classifier would see at time-of-decision varies — see Part C).
- `transcript` (the ASR output you'd expect; ok to transcribe by hand for this corpus).
- `label` ∈ {`backchannel`, `turn_grab`}.
- `label_rationale` (one sentence; why you labelled it that way).

Two labellers if available; inter-rater agreement should land above 0.8 on this binary. If labellers disagree on more than 15 % of clips, the labelling guide needs to be sharpened before you move on.

### Part B — implement the heuristic classifier

Code Chapter 4's heuristic:

```python
BACKCHANNELS = { "mm", "mm-hm", "uh-huh", "yeah", "yep", "yes", "right", "okay",
                 "ok", "sure", "oh", "wow", "i see", "got it", "hm" }
STARTERS = ("wait", "actually", "no", "stop", "hold on", "sorry", "hang on")

def classify(user_audio_ms: int, partial_transcript: str, energy_rel: float) -> str:
    if user_audio_ms < 400: return "defer"
    text = partial_transcript.strip().lower().rstrip(".?!,")
    if user_audio_ms < 900 and text in BACKCHANNELS: return "backchannel"
    if user_audio_ms < 900 and energy_rel < 0.5: return "backchannel"
    if any(text.startswith(s) for s in STARTERS): return "turn_grab"
    if user_audio_ms >= 900: return "turn_grab"
    return "defer"
```

Tune the constants against your corpus's training split.

### Part C — simulate time-of-decision

Here is the gotcha: the classifier does not see the full clip; it sees the clip as it exists `t_decision` ms after speech-start, with the transcript as it is at that moment. Simulate this:

- For each clip, at `t_decision ∈ {300, 500, 700, 1000}` ms from speech-start, compute what the classifier would see: `audio_duration_ms = t_decision`, `partial_transcript = <ASR output up to t_decision>`.
- You may stub the "ASR output" by running your streaming ASR on the clip and timestamping each partial, then querying at `t_decision`. Or (for speed) hand-generate a plausible partial ("mm-" at 300 ms, "mm-hm" at 500 ms, etc.).
- Record the classifier output at each `t_decision`.

Report: at what `t_decision` does the heuristic reach a confident classification for each clip? That is the signal for how fast the barge-in path can commit to its decision.

### Part D — LLM classifier variant

Implement the small-LLM variant from Chapter 4:

```python
SYSTEM = """You are deciding whether a listener's short utterance is a backchannel (a brief acknowledgment while the speaker continues) or a turn-grab (the listener taking the floor). Reply exactly one character: 'B' for backchannel, 'T' for turn-grab."""

def classify_llm(bot_utterance: str, user_partial: str) -> tuple[str, float]:
    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        temperature=0,
        max_tokens=1,
        logprobs=True,
        top_logprobs=2,
        messages=[
            {"role": "system", "content": SYSTEM},
            {"role": "user", "content": f"SPEAKER: {bot_utterance}\nLISTENER: {user_partial}"},
        ],
    )
    # Derive label + confidence from logprobs.
```

Run against the same corpus and the same `t_decision` sweep.

### Part E — integration test with Exercise 3

Reuse the Exercise 3 barge-in path. Modify `on_interrupt()`:

- Start the cancel immediately (parallel with classifier).
- Call `classify(user_audio_ms, partial_transcript, energy_rel)` as the user audio accumulates.
- On classifier `turn_grab`: let the cancel complete.
- On classifier `backchannel` **within a grace window of 100 ms** of the first cancel operation: abort the cancel (TTS resumes from where it was).
- On classifier `backchannel` **after** the cancel has completed: emit an LLM "pick up from an interjection" resume turn (Chapter 3 §Preserving partial state).

Run the full barge-in corpus from Exercise 3 (turn-grabs) and the backchannel corpus (part A). Measure:

- **False-turn-grab count.** Backchannel clips that incorrectly triggered a completed cancel.
- **Missed-turn-grab count.** Turn-grab clips that incorrectly aborted the cancel.
- **Reaction-time delta.** Did adding the classifier push p95 reaction time above 200 ms?

### Part F — write-up

Answer:

- Which clip categories did the heuristic get right? Which did it get wrong?
- At what `t_decision` is the heuristic confident enough to commit? The LLM?
- Did the LLM close any of the heuristic's errors? Which ones?
- Would you ship the heuristic alone, the LLM alone, or a cascade (heuristic first, LLM as fallback on `defer`)?
- What is the per-session cost of the LLM-classifier path, extrapolated to 1 M sessions / month at your pipeline's barge-in rate?

## Starter guidance

- **Duration first.** The duration signal does most of the work on clean speech. If your heuristic needs more than ~30 lines to be competitive, something is wrong.
- **Energy relative.** Normalise the user's utterance energy against the recent noise floor OR against the bot's own speech energy. Absolute dB readings are device-dependent and will not generalise.
- **Label edge cases explicitly.** "wait" and "really?" are the hard cases; your rationale on these is what tells future-you what you meant.
- **Measure the cost, not just the accuracy.** 98 % accuracy at 300 ms of extra latency is a worse product than 90 % accuracy at 20 ms. Report both.
- **Evaluate at the operational time-of-decision, not on full clips.** Clip-at-rest accuracy flatters the classifier. Partial-transcript-at-decision-time accuracy is what ships.
- **Start with your Exercise 3 code for the resume path.** `[interrupted]` markers, truncation, and the LLM pick-up-turn prompt are already there.

## Acceptance criteria

- [ ] Corpus of ≥ 40 clips, stratified across lexical / prosodic / ambiguous / turn-grab / edge, with labels + rationale.
- [ ] Heuristic classifier matching Chapter 4's sketch, tuned on a training split.
- [ ] Small-LLM classifier with logprob-derived confidence.
- [ ] Time-of-decision sweep at 300, 500, 700, 1000 ms for both variants.
- [ ] Integration test with Exercise 3 barge-in path on both corpora; false-turn-grab and missed-turn-grab counts recorded.
- [ ] One-page write-up answering the five questions above.
- [ ] Reaction-time measurement after classifier integration; delta vs. Exercise 3 baseline.

## Stretch goals

- **Trained small classifier.** Train a logistic regression over hand-crafted features (duration, energy, pitch mean/variance, first-N MFCCs, BOW over backchannel vocab). Compare against the heuristic on a held-out split. Note sample-size constraints; a 40-clip corpus is barely enough to train on.
- **VAP output as a feature.** If you have a VAP deployment (research-grade from Exercise 2 stretch or Chapter 2 §VAP), use its "P(overlap)" or "P(turn-shift)" head as a feature for the classifier. Note whether it moves the numbers.
- **Multilingual.** Repeat Part A for Spanish or Japanese. Backchannel vocabularies are different; measure whether the LLM generalises and the heuristic does not.
- **Streaming classifier.** Call the classifier incrementally — once at 300 ms, again at 500 ms, 700 ms, 1000 ms — and let the barge-in path act on the first confident decision. Report the latency vs. correctness trade-off.
- **Observability.** Instrument the Exercise 3 pipeline to log the classifier triple (duration, transcript, output) per event. Dump a day's worth of session logs and chart the decision-time distribution.

## Non-goals

- Shipping a production-quality trained classifier. 40 clips is a bench corpus. A production classifier needs thousands of clips from your own data.
- Evaluating backchannel-emission by the bot. This exercise is receive-side only.
- Replacing the Chapter 3 barge-in path. The classifier is an addition to it, not a replacement.

## Keep it

The corpus, the heuristic, and the LLM classifier are reusable. Exercise 5 will treat false-turn-grab-on-backchannel as one of its metric stratifications; the labelled corpus is the ground truth for that stratification.
