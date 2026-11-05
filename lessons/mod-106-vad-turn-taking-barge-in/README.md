# mod-106-vad-turn-taking-barge-in: Voice Activity Detection, Turn-Taking, and Barge-In Engineering

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 14 hours

## Learning objectives

- Deploy VAD at the pipeline edge — Silero VAD (de facto open-source baseline), WebRTC VAD (energy + GMM, historical compare-point), server-side-VAD contracts from OpenAI Realtime / Gemini Live — with the threshold / hangover / minimum-speech knobs that matter and the failure modes they expose
- Implement end-of-utterance (EOU) detection beyond pure VAD — the LiveKit open-weights EOU model, semantic end-pointing via a small LLM classifier on the running transcript, and the Voice Activity Projection (VAP) family of self-supervised turn-prediction models (Ekstedt & Skantze) — so the bot stops predicting 'the user is done' from silence alone
- Design the interrupt-handling path (barge-in) — user-interrupts-bot (cancel in-flight TTS, truncate the LLM generation, preserve partial state so the response can pick up), bot-interrupts-user (back-channeling, 'go ahead' cues, non-speech-interrupt rules); target <200 ms interrupt reaction
- Manage overlap and backchannelling — when the user says 'mm-hm' or 'right' mid-sentence, the bot should NOT treat it as a turn grab; when the user says 'wait, no' mid-sentence, the bot MUST; design the classifier that distinguishes
- Author the turn-taking metric suite — turn-taking correctness vs. a ground-truth turn boundary, false-turn-grab rate, missed-end-of-utterance rate, median + p95 bot-response gap, p95 user-interrupt reaction time — tying metrics back to mod-112 evaluation harness
- Read the turn-taking literature at engineer altitude — Skantze's 2021 survey, VAP's 2022 ICASSP paper, the LiveKit turn-detector methodology — enough to defend the VAD / EOU / barge-in choice to a reviewer; deep conversational-ML research is linked to `nlp-engineer` peer

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
