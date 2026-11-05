# mod-104-tts-production-engineering: Text-to-Speech (TTS) Production Engineering

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 14 hours

## Learning objectives

- Select and wire the TTS provider — OpenAI TTS (tts-1, tts-1-hd, gpt-4o-mini-tts), ElevenLabs (Flash v2, Turbo v2.5, Multilingual v2, Conversational), Cartesia Sonic family (Sonic, Sonic Turbo, Sonic 2 at sub-100 ms TTFB), PlayHT (Play 3.0, Play Dialog), LMNT, Resemble, Google Chirp3 HD, Azure Neural Voices, AWS Polly (Neural + Long-Form + Generative), Hume EVI — scoring against time-to-first-audio-byte (TTFB), naturalness (MOS / CMOS), expressive range, SSML support, language coverage, voice-cloning capability, cost, and compliance posture
- Author the streaming TTS contract — HTTP chunked / WebSocket streaming / provider SDK; design the player pipeline so TTFB is the latency-driver, not the total synthesis time
- Write SSML (W3C Speech Synthesis Markup Language 1.1) for prosody, pauses, emphasis, phonemic pronunciation, breaks, and interpret-as; know which providers subset / extend SSML; ship a phonetic lexicon for domain vocabulary (product names, drug names, acronyms)
- Decide between expressive / conversational TTS (ElevenLabs Conversational, Hume EVI, Cartesia Sonic Turbo) and neutral TTS for the deployment, keeping voice consistency across turns; manage voice selection UX
- Walk the voice-cloning stack end-to-end — instant voice clone vs. professional voice clone, consent-flow requirements, provenance (ElevenLabs voice IDs, Resemble provenance), legal implications (Illinois BIPA, NO FAKES Act, UK Data (Use and Access) Act) — and know when to say no
- Understand modern generative TTS architectures at a working level — VITS end-to-end, VoiceBox flow-matching, XTTS zero-shot cloning, Bark audio-language-modelling — enough to pick the right provider / open-weights model, not to re-implement
- Cite the boundary to `training-pipeline-engineer` (level 35) on TTS voice-cloning fine-tunes and to `agentic-safety-engineer` (level 40) on frontier voice-safety methodology

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
