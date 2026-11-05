# 04. Overlap and Backchannelling: The Backchannel-vs-Turn-Grab Classifier

## Why this chapter

Chapter 3 ended with a cliffhanger: when the user speaks while the bot is speaking, Stage 2 has to decide whether it is a turn-grab (cancel everything, listen) or a backchannel (keep speaking; the user is just signalling they are following along). Getting this wrong in either direction is a visible defect:

- **False turn-grab** (classify "mm-hm" as a barge-in): the bot cuts itself off every time the user shows they are listening. Users get frustrated and stop acknowledging.
- **Missed turn-grab** (classify "wait, no" as a backchannel): the bot plows through while the user is trying to correct it. Users have to repeat themselves, loudly.

This chapter is the design of the classifier that decides. It covers (a) what backchannels and turn-grabs look like in the audio and the transcript, (b) the three deployment shapes — heuristic, trained classifier, LLM prompt — and when each fits, (c) training and evaluation, including how to build a labelled dataset from your own call logs, and (d) the integration points with Chapters 1–3.

## The phenomenon: what you're classifying

Linguists call them **backchannels** or **continuers**: short vocalisations a listener produces to signal attention without claiming the floor. They are ubiquitous in conversational speech — the Switchboard corpus statistics put the rate at roughly one every few seconds. Examples:

- Lexical: `mm-hm`, `uh-huh`, `yeah`, `right`, `okay`, `sure`, `oh`, `wow`, `I see`.
- Non-lexical: short laughs, aspirations, tongue clicks.
- Short questions that are not real questions: `oh really?`, `is it?` (phatic).

**Turn-grabs** are everything else — the user is taking the floor. Examples:

- Corrections: `wait, no`, `actually that's wrong`, `hold on`.
- Redirections: `can we talk about—`, `let me ask something different`.
- New information: `oh, I forgot to mention—`.
- Full-sentence answers to a question still being asked.

The classification is not purely linguistic — prosody and duration matter:

- A backchannel is typically **short** (<700 ms), **flat intonation**, and **low energy** relative to the speaker's turn-taking utterances.
- A turn-grab is typically **>400 ms**, with **rising energy**, and often starts with a **discourse marker** (`wait`, `actually`, `so`, `but`, `oh`) or a **first-person pronoun** (`I think`, `can I`).

These are tendencies, not laws. "Oh really?" can be either. "Wait" can be a backchannel of surprise or the start of a turn-grab. The classifier has to resolve the ambiguity under latency pressure.

**Reading.** Skantze's 2021 survey ([Computer Speech & Language 67](https://doi.org/10.1016/j.csl.2020.101178)) is the shortest route to the linguistic framing; the backchannel section surveys the acoustic and lexical cues the research community has characterised.

## The three deployment shapes

### Shape 1: the heuristic classifier

The simplest thing that works, and the thing most production voice agents ship first. No model; a scoring function over three signals:

```python
def classify_backchannel_vs_turn_grab(user_audio_ms, partial_transcript, energy_rel):
    """Return 'backchannel', 'turn_grab', or 'defer' (not enough info yet)."""
    # 1. Duration gate: very short utterances are nearly always backchannels.
    if user_audio_ms < 400:
        return "defer"  # wait another 200 ms and re-classify

    # 2. Lexical gate: if the stabilised transcript matches a backchannel vocabulary, likely backchannel.
    BACKCHANNELS = {
        "mm", "mm-hm", "uh-huh", "yeah", "yep", "yes", "right", "okay",
        "ok", "sure", "oh", "wow", "i see", "got it", "hm",
    }
    text = partial_transcript.strip().lower().rstrip(".?!,")
    if user_audio_ms < 900 and text in BACKCHANNELS:
        return "backchannel"

    # 3. Prosodic gate: low energy + short duration + flat pitch → backchannel.
    if user_audio_ms < 900 and energy_rel < 0.5:
        return "backchannel"

    # 4. Start-of-turn markers.
    STARTERS = ("wait", "actually", "no", "stop", "hold on", "sorry", "hang on")
    if any(text.startswith(s) for s in STARTERS):
        return "turn_grab"

    # 5. Longer utterance → turn-grab.
    if user_audio_ms >= 900:
        return "turn_grab"

    return "defer"
```

Operational notes:

- **Duration is the strongest single signal.** A 1200 ms "mm-hm" does not exist; a 1200 ms utterance is a turn-grab regardless of transcript content.
- **The vocabulary is per-language and per-domain.** English backchannels and Japanese backchannels and Mandarin backchannels overlap partially; verify. Within English, a technical-support domain has a different backchannel lexicon ("got it," "understood") than a therapy domain ("mm-hm," "and then").
- **`defer` is a legitimate output.** For the first few hundred ms of user audio, you don't have enough signal. Hold off the cancel decision and classify again in 200 ms. Chapter 3's "cancel in parallel with classifier" pattern means the bot has already stopped speaking; `defer` just delays the "resume vs. truly interrupt" decision.

Why this works well enough to ship as v1: backchannels and turn-grabs are **duration-separable** in practice. The ambiguous mid-range (400–900 ms, generic-word content) is small, and the cost of misclassifying in that range is low — the bot cancels for a moment and resumes.

### Shape 2: a trained small classifier

When the heuristic misclassifies in your domain more than you can afford, promote to a trained classifier. The inputs are the same (audio features + transcript); the model is the thing being trained.

**Model architecture.** Something small. Three options worth naming:

- **Logistic regression / gradient-boosted trees over hand-crafted features.** Features: duration, mean energy, pitch mean & variance, zero-crossing rate, first N MFCCs, transcript word count, BOW over a backchannel vocabulary. Cheap to train, interpretable, 5–20 ms inference. Often enough.
- **A small audio+text transformer.** Two embedding paths (audio frames + ASR tokens) concatenated into a two-class head. Few tens of thousands of params. Needs ~10–50 h of labelled conversational data.
- **Repurpose VAP.** The Voice Activity Projection model (Chapter 2) already produces a "P(overlap) + P(hold vs. grab)" signal. In a VAP-deployed stack, the overlap-vs-grab head is derivable from the projection distribution without a separate classifier.

**Training data.** The dataset you want: 1–3 s windows of user audio centred on a VAD-detected onset during bot-speaking, with a binary label `backchannel` vs. `turn_grab`. You can source windows from:

- Your own call logs — if you have them, this is the correct answer. Sampling policy: every detected barge-in from the last N months, labelled by a human against the rule "if the agent should have kept talking, label backchannel; if the agent should have stopped, label turn_grab."
- Switchboard Dialog Act annotations ([Jurafsky et al.](http://compprag.christopherpotts.net/swda.html), [LDC97S62](https://catalog.ldc.upenn.edu/LDC97S62)) have a `b` (backchannel) tag family; useful as a general-English baseline though not calibrated to your UX.
- Fisher English ([LDC2004S13](https://catalog.ldc.upenn.edu/LDC2004S13), [LDC2005S13](https://catalog.ldc.upenn.edu/LDC2005S13)) telephony conversation; richer audio coverage, per-turn transcripts but not per-event backchannel labels.
- CallHome ([LDC97S42](https://catalog.ldc.upenn.edu/LDC97S42)) telephone conversational speech, English and other languages.

**Important:** labels are UX-defined, not phoneticist-defined. A backchannel in your product is "an utterance that, had the bot kept talking, would have been the right behaviour." A turn-grab is "an utterance that, had the bot kept talking, would have been a bug." Inter-rater reliability on this definition is good when the criterion is operational, poor when it is linguistic. Write the operational criterion into the label guide.

**Evaluation.** Hold out a corpus of real barge-in events from your own logs, labelled as above. Report:

- Precision on `turn_grab` (how often a predicted turn-grab was really a turn-grab).
- Recall on `turn_grab` (how often a real turn-grab was caught).
- False-turn-grab rate (Chapter 5): predicted turn-grabs that were backchannels, normalised per minute of call.
- Latency: p50, p95 of inference time; include the audio-window fetch in the budget.

A trained classifier only earns the engineering cost if it beats the heuristic on the UX metric (not just on classification accuracy). Measure both; ship if the UX metric moves.

### Shape 3: a small LLM prompt on the partial transcript

The quick-and-dirty escape hatch:

```jsonc
{
  "model": "gpt-4o-mini",
  "temperature": 0,
  "max_tokens": 1,
  "system": "You are deciding whether a listener's short utterance is a backchannel (a brief acknowledgment while the speaker continues) or a turn-grab (the listener taking the floor). Reply exactly one character: 'B' for backchannel, 'T' for turn-grab.",
  "user": "SPEAKER: {bot_utterance_so_far}\nLISTENER: {user_partial_transcript}"
}
```

Latency: 100–300 ms per call. Cost: one small-model call per detected barge-in; usually acceptable. Strength: inherits an LLM's language and discourse priors; works in multiple languages with no training data.

Weakness: text-only. The LLM does not see prosody, duration, or energy; it classifies on words. For many backchannels (short prosodically-flat "mm" or "right") the text alone is what matters; for ambiguous content ("wait" at the start of a turn vs. "wait" as a backchannel of surprise), the LLM can be wrong exactly where the acoustic signal is informative.

Pattern: use the LLM as the second-stage classifier after a duration gate. If `user_audio_ms < 400`, defer. If `400 ≤ user_audio_ms ≤ 900` and the transcript is lexically ambiguous, call the LLM. Otherwise fall back to the heuristic.

## The combined decision loop

Putting Chapters 1–3 together with this classifier, the decision loop while the bot is speaking:

```
on_frame(user_audio_frame):
    vad_decision = vad.step(user_audio_frame)
    if not vad_decision.speech_started:
        return
    # Speech started mid-bot-speaks. Start cancelling.
    schedule_cancel()  # parallel; Chapter 3 Stage 3

    # Classify.
    label = classify(
        user_audio_ms=vad_decision.speech_ms,
        partial_transcript=stt.partial(),
        energy_rel=vad_decision.energy_rel,
    )
    if label == "turn_grab":
        complete_cancel()   # proceed through Stages 4–5
    elif label == "backchannel":
        abort_cancel()      # resume bot speech; emit a short acknowledgment if the player supports it
    else:  # defer
        pass  # re-evaluate on next frame
```

Note what this does **not** do:

- **It does not wait for the classifier before cancelling.** The cancel is scheduled on VAD speech-started, in parallel. If the classifier later says "backchannel," `abort_cancel` undoes the cancel if it hasn't completed yet; if the cancel already landed, the bot restarts its last sentence.
- **It does not fail hard on `defer`.** `defer` is the safe state; the classifier re-runs every frame.

### "Abort cancel" — the hard part

If the cancel has already propagated to TTS (the TTS stream is closed, the player is flushed) and the classifier then says "backchannel," you cannot un-cancel. The bot already stopped. The right UX is for the bot to say something like "— so, as I was saying," picking up from the last sensible pause boundary. The LLM can do this if you prompt it to "resume your previous response, acknowledging a short interjection from the user," passing it the truncated response and the user's interjection.

Two engineering choices:

1. **Short deliberate cancel delay.** Introduce a 50–100 ms grace period between "cancel scheduled" and "cancel executed." If the classifier returns `backchannel` inside the grace period, the cancel is aborted cleanly. Costs 50–100 ms in the barge-in latency budget; worth it if your backchannel rate is high.
2. **Eager cancel, graceful resume.** Cancel immediately; on `backchannel` after the cancel, resume via the LLM "pick up from an interjection" prompt. Latency-friendly but resume quality depends on the LLM.

Pick on the interrupt-reaction-time budget. Grace-period is correct when latency allows; eager-with-resume is correct when you must ship <200 ms barge-in reaction.

## The non-English story

The backchannel inventory varies by language. Partial list:

- **Japanese.** Aizuchi (`はい` *hai*, `うん` *un*, `ええ` *ee*, `そうですね` *soo desu ne*) — famously frequent, nearly continuous.
- **Mandarin.** `嗯` *ǹg*, `对` *duì*, `好` *hǎo*.
- **Spanish.** `sí`, `claro`, `vale`, `ajá`.

If your deployment is multilingual, the heuristic vocabulary and the trained classifier's data must be multilingual. The LLM-as-classifier generalises more readily across languages since the model has seen each; the trained small classifier does not.

Separately, the **rate** of backchannelling varies across cultures. In Japanese conversation, a listener who does not aizuchi is heard as not listening; the bot in a Japanese product may itself need to backchannel (Chapter 3's bot-interrupts-user section). In Nordic cultures, backchannels are sparser; frequent bot-side backchannels can feel pushy. Your choice of when the bot backchannels is a product-UX decision, not just an engineering default.

## Observability

Log, for every detected user-utterance-during-bot-speaking:

- `user_audio_ms` at classification time.
- `partial_transcript` at classification time.
- Classifier output and confidence.
- Post-event decision (cancel completed, cancel aborted, resume mode).
- User's next turn (how they responded to the bot's decision — did they repeat themselves, did they continue naturally?). This is your ground truth for the Chapter 5 metrics.

A dashboard of these fields over a week will tell you your false-turn-grab rate, your missed-turn-grab rate, and where in the audio-duration × transcript-ambiguity plane your classifier fails. That is what you re-tune against.

## Common anti-patterns

- **Treating any user audio during bot-speaks as a turn-grab.** Cheap; catastrophic for backchannel-heavy audiences. Users feel the bot stepping on itself.
- **Treating any user audio during bot-speaks as a backchannel.** The symmetric failure: users cannot interrupt. Equally obvious; less commonly shipped by accident.
- **Blocking cancel on classifier.** The classifier cost arrives in the barge-in budget. Cancel first; resume if the classifier says "backchannel."
- **Classifying on the raw ASR stream.** Partials rewrite; your classifier runs twice on the same audio with different text. Classify on stabilised partials (Chapter 2).
- **A heuristic vocabulary that doesn't fit your product.** The default English list (`mm-hm`, `yeah`, `right`) misses domain acknowledgments (`understood`, `confirmed`, `copy that`). Audit against your logs.
- **No per-event log row.** You cannot diagnose or re-tune without the triple (duration, transcript, classifier output).
- **Treating it as a pure acoustic problem.** Short-utterance backchannel detection is dominated by duration + lexical signal; adding a complex acoustic model does not always help.
- **Ignoring language and culture.** The classifier that ships in English may be wrong in Japanese or Mandarin out of the box.

## Summary

- The backchannel-vs-turn-grab classifier is the thing that decides, mid-barge-in, whether the user is really taking the floor or just saying "mm-hm." False turn-grabs cut the bot off for nothing; missed turn-grabs trample real interrupts.
- Three deployment shapes: heuristic (duration + lexical + prosody — ships first and usually well enough), trained small classifier (earned when heuristic misses in your domain), small-LLM prompt (easy to ship, good across languages, text-only).
- Duration is the single strongest signal. A 1200 ms "mm-hm" does not exist. Prefer a duration gate before any model inference.
- The integration pattern (Chapter 3): start the cancel on VAD speech-started, run the classifier in parallel, cancel if `turn_grab`, abort or resume if `backchannel`.
- Labels are UX-defined, not phoneticist-defined. Train on "would the bot have been right to keep talking?" labels sourced from your own call logs.
- The backchannel inventory and rate vary by language. Multilingual deployments need multilingual vocabularies and data; the LLM-as-classifier generalises more readily than a trained model.
- Log the triple (duration, partial transcript, classifier output) for every event. Chapter 5's metric suite reads from this log.

Next chapter: the turn-taking metric suite — turn-taking correctness vs. ground-truth boundaries, false-turn-grab rate, missed-end-of-utterance rate, median + p95 bot-response gap, p95 user-interrupt reaction time, tied back to the Module 112 evaluation harness.
