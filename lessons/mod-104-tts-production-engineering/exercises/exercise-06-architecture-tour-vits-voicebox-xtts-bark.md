# exercise-06: Architecture Tour — VITS, VoiceBox, XTTS, Bark

**Estimated effort:** 1.5 hours · **Output:** a one-page architecture write-up per family (4 pages total) + a vendor-mapping note + at least one hands-on self-host sample

## Objective

Internalise Chapter 6 to the point where you can read a vendor's "our new voice is based on flow-matching with in-context conditioning" announcement and know exactly what they mean, what the trade-offs are, and what questions to ask next. You will produce short architectural write-ups for VITS, VoiceBox, XTTS, and Bark, map each to the vendors most likely to be using it, and run one open-weights model locally to anchor the reading.

Not a research exercise. A literacy exercise, with one hands-on anchor.

## Prerequisites

- [Chapter 6 — Modern Generative TTS Architectures at a Working Level](../06-generative-tts-architectures-vits-voicebox-xtts-bark.md).
- The papers listed in Chapter 6's reading list; you will read abstracts + architecture figures, not full papers.
- Python 3.10+, a machine with at least one GPU (CUDA preferred; MPS acceptable; CPU-only will work on the smaller models but slowly).

## Steps

### Part A — Read and summarise (40 minutes)

1. For each of the four families — VITS (+ VITS2 / StyleTTS 2), VoiceBox, XTTS, Bark — read the abstract and architecture figure(s) of the primary paper. Chapter 6 lists them.
2. Produce a one-page write-up per family (4 × 1 page) answering:
   - **What problem does this architecture solve that the prior art did not?**
   - **What is the acoustic representation?** (Mel-spec, codec tokens, continuous latent.)
   - **How is generation structured?** (AR, NAR, flow-matching, diffusion.)
   - **What does "cloning" look like on this architecture?** (Fine-tune only, zero-shot from reference, speaker embedding.)
   - **What are the failure modes you would expect in production?** (Long-form divergence, length mis-handling, artefacts at low bitrate, etc.)
   - **What managed vendors would plausibly be using this family?** Speculative but informed; cite where the vendor has acknowledged it.
3. Each write-up should be readable by an engineer with no prior TTS background in ten minutes. Explain acronyms the first time you use them.

### Part B — Map to the current landscape (15 minutes)

4. Produce a 4-row × N-column table: rows are the four families, columns are managed vendors you've evaluated (from exercise-01). For each cell, note "likely", "unlikely", "unknown", with a sentence of justification.
5. Flag where a vendor's public communications contradict their likely architecture. Vendors rarely publish; your mapping is informed speculation, not forensic claim. Say so explicitly.

### Part C — Run one open-weights model (30 minutes)

6. Pick one open-weights model from Chapter 6's shortlist:
   - **XTTS v2** for zero-shot multilingual cloning (via `coqui-tts` or the HF transformers path).
   - **StyleTTS 2** for a single-speaker high-quality baseline.
   - **F5-TTS** for flow-matching zero-shot cloning (the open-weights VoiceBox-adjacent family).
   - **Piper** for a tiny on-device baseline.
7. Install, download the model, and synthesise 3–5 sample utterances. Save the outputs.
8. Measure:
   - Time-to-first-audio-byte for the shortest utterance (cold and warm).
   - Realtime factor on a longer utterance.
   - GPU / RAM footprint (approximate).
9. Pick one Chapter 6 claim about the model's family and verify-or-disprove it on your sample. Examples:
   - Claim: "VITS is realtime-capable." Did you get RTF > 1 on your hardware?
   - Claim: "XTTS zero-shot cloning works from 6 seconds of reference." Did it preserve the reference voice recognisably?
   - Claim: "AR models can hallucinate on long inputs." Did you observe repetition / skipping on a 30-second input?

### Part D — Decide what you would ship (15 minutes)

10. For each of these three hypothetical briefs, pick the family you would ship and justify in 2–3 sentences:
    - A 50-language voice-preservation app for ALS patients (moderate latency acceptable).
    - A realtime consumer voice agent with one branded voice (sub-300 ms TTFB required).
    - A creative tool that generates character voices with laughter and emotion (no realtime requirement).

## Starter guidance

- **Don't read the ablations.** You are reading for recognition, not implementation. Abstract + architecture figure + results table in that order.
- **Chapter 6 is the map; the papers are the territory.** If Chapter 6's description contradicts the paper's description, trust the paper and let the author know.
- **The install may fight you.** `coqui-tts` vs. recent transformers / torch versions is a known pain point; pin versions. If an install consumes more than 20 minutes, switch to Piper or a tiny-VITS path — the point is to see one inference, not fight dependencies.
- **Document your hardware.** TTFB on an H100 and an M2 Mac are not the same number; your numbers are only meaningful with the hardware noted.
- **Vendor-mapping is informed speculation.** Most managed vendors do not disclose architecture. "Likely X because they market Y" is a valid cell content; "Definitely X" is not.

## Acceptance criteria

- [ ] Four one-page architecture write-ups (VITS-family, VoiceBox, XTTS, Bark) answering the six questions each.
- [ ] Vendor-mapping table with justifications and uncertainty flags.
- [ ] One open-weights model installed and run; 3–5 sample utterances captured; TTFB, RTF, and footprint noted.
- [ ] One Chapter 6 claim verified or disproven empirically on the model you ran.
- [ ] Three-brief shipping decision with 2–3-sentence rationales.
- [ ] Hardware documented alongside the numbers.

## Stretch goals

- **Second model.** Run a second open-weights model from a different family. Compare on the same text; note the qualitative differences (expressiveness, cloning fidelity, speed, stability).
- **Compare to a managed vendor** on the same text. Is the open-weights gap closing on your utterances?
- **Fine-tune a VITS / StyleTTS 2 on a tiny custom voice.** 10 minutes of audio, a LoRA or an adapter if the recipe supports it. The goal is not quality — it is to walk the training surface so you know what the `training-pipeline-engineer` conversation looks like from your side.
- **Write a brief for `fine-tuning-engineer` or `training-pipeline-engineer`.** For your product's brief, sketch the data spec + acceptance criteria you would hand over to request a self-hosted voice fine-tune. This is the Chapter 7 handoff shape; exercise-05 is the consent half, this is the data half.
- **Read the EnCodec paper abstract and look at the vector-quantisation figure.** Enough to know what "codec tokens" means when you encounter it in audio-LM-style systems. This also prepares you for Module 105.

## Keep it

The four write-ups are a reference for every time a vendor announces a new voice line with architecture claims. Update them when the papers are superseded (VoiceBox → something newer; XTTS → a non-dormant alternative). Over a year, this file becomes the briefest readable reference you have on the moving TTS architecture landscape.
