# exercise-05: Deploy ASVspoof Countermeasures and C2PA Watermarking

**Estimated effort:** 2 hours · **Output:** a working ingress spoof-detection script + a working TTS output-watermarking script + a written design note chaining AudioSeal to a C2PA manifest.

## Objective

Translate [Chapter 6](../06-spoofing-deepfake-provenance.md) into running code for both directions: an inbound anti-spoof pipeline that scores arbitrary audio with a published ASVspoof-era countermeasure model, and an outbound TTS provenance pipeline that applies an AudioSeal watermark and attaches a C2PA manifest. The code can live in a sandbox — no production deployment required — but it must actually run.

## Prerequisites

- [Chapter 6 — Anti-Spoofing, Deepfake Detection, and Audio Provenance](../06-spoofing-deepfake-provenance.md).
- Python 3.10+, `torch`, `torchaudio`, `numpy`. Ability to `pip install` from PyPI.
- A GPU is helpful but not required; the models in Part A and Part B run on CPU at acceptable speeds for a demo.
- A clean sandbox directory. No PHI, no real caller audio.

## Steps

### Part A — Inbound spoof detection (45 minutes)

1. Create a project directory `voice-provenance/`. Install dependencies:
   ```bash
   pip install torch torchaudio numpy soundfile
   pip install speechbrain       # for a pretrained speaker-verification baseline
   ```
2. Fetch at least six audio samples for testing:
   - Two *bonafide* speech samples from a public corpus (e.g., [LibriTTS](https://www.openslr.org/60/) sample downloads, or a Common Voice export).
   - Two *TTS-generated* speech samples from a recent open TTS model (any accessible TTS; the point is that these are non-bonafide).
   - Two *recorded-then-replayed* samples (record your own voice, play it on a phone speaker, re-record it) — proxies for replay attacks.
3. Author `asv_spoof.py` that:
   - Loads a published anti-spoof model (recommended: AASIST or a RawNet3 baseline). You may use the open implementations at [github.com/clovaai/aasist](https://github.com/clovaai/aasist) (MIT) or equivalent. If model weights are not immediately downloadable, substitute a smaller classifier trained quickly on the samples above; document the substitution.
   - Loads a pretrained speaker-verification model via SpeechBrain (`speechbrain.pretrained.EncoderClassifier.from_hparams("speechbrain/spkrec-ecapa-voxceleb")`).
   - Accepts a path to an audio file and emits:
     ```json
     {
       "file": "sample1.wav",
       "bonafide_score": 0.82,
       "speaker_embedding_sha": "...",
       "verdict": "bonafide|spoof|uncertain",
       "threshold_used": 0.5
     }
     ```
4. Run `asv_spoof.py` on each of your six samples. Print a 2×6 table (score, verdict) and compute the per-class error rate.
5. Write a two-paragraph discussion in `NOTES.md` on what failed and why. Common findings: bonafide samples with recording-channel artefacts score low; TTS samples from the newest models score higher than older ones.

### Part B — Outbound watermarking with AudioSeal (30 minutes)

6. Install AudioSeal:
   ```bash
   pip install audioseal
   ```
7. Author `watermark.py` that:
   - Loads the AudioSeal generator and detector (`audioseal.AudioSeal.load_generator("audioseal_wm_16bits")` and `load_detector("audioseal_detector_16bits")`; verify current public API in the repo README at [github.com/facebookresearch/audioseal](https://github.com/facebookresearch/audioseal)).
   - Accepts a plain WAV path and a 16-bit integer payload (`agent_id` encoded as uint16).
   - Emits a watermarked WAV alongside the original.
   - Then runs the detector on the watermarked WAV and verifies the payload round-trips.
8. Run it on each of your bonafide samples; confirm the watermarked audio sounds indistinguishable from the original on casual listening and the detector recovers the payload with high confidence.
9. Test codec robustness: encode the watermarked WAV to Opus at 16 kbps and back (`ffmpeg -i wm.wav -c:a libopus -b:a 16k wm.opus && ffmpeg -i wm.opus wm-rt.wav`), run the detector. Record the detection score. Repeat with G.711 μ-law (8 kHz, 8-bit). Produce a 3-row table.

### Part C — C2PA manifest attachment (30 minutes)

10. Install `c2patool` (Rust binary; prebuilt releases at [github.com/contentauth/c2patool/releases](https://github.com/contentauth/c2patool/releases)).
11. Generate a self-signed claim-signer certificate for the exercise (see the c2patool README for the minimal `openssl req` incantation). Store it in `signer.cert` and `signer.key`.
12. Write a C2PA manifest JSON that includes:
    - `claim_generator`: your exercise name and version.
    - `assertions`: `c2pa.actions` with `c2pa.created`; `c2pa.ai_models` listing the (fictional) TTS model name + version; a custom assertion `com.deskflow.watermark_payload` carrying the integer payload you used in Part B.
    - `ingredients`: empty (no upstream content).
13. Apply the manifest to your watermarked WAV with `c2patool`:
    ```bash
    c2patool wm.wav -m manifest.json -o wm-signed.wav
    ```
14. Verify with `c2patool wm-signed.wav`. Confirm the signature chain validates.
15. Chain to the watermark: in your `NOTES.md`, document the pattern — the watermark payload is a *pointer* (specifically: the first 16 bits of the SHA-256 of the manifest). On receiving a stripped audio clip, the detector recovers 16 bits; the first 16 bits of any candidate manifest's SHA-256 can be compared, and the match points to the manifest for full provenance lookup.

### Part D — Design note (15 minutes)

16. In `NOTES.md`, write a one-page design note that assembles the outbound-provenance pipeline from the pieces you just built:
    - The TTS output is encoded at 48 kHz PCM.
    - AudioSeal injects the watermark whose payload references the manifest.
    - The C2PA manifest is attached to the WAV container.
    - The container is encoded to Opus for transport.
    - The logging layer records (call_sid, agent_id, payload, manifest_hash, sent_at).
17. Include a short section on the *limits*: what happens when the container is lost (watermark survives, manifest doesn't); what happens when the codec is lossier than you benchmarked (watermark degrades); what happens when the attacker re-synthesises with a different model (both are lost — you cannot prove *that* attack with provenance alone).

## Starter guidance

- The AASIST reference implementation requires specific input preprocessing (16 kHz mono, specific length windows); the repo README is authoritative. Follow it exactly.
- AudioSeal's public API as of early 2024 includes `.get_watermark(audio, message)` and `.detect_watermark(audio)` methods; check the current README for any signature changes.
- The c2patool JSON manifest schema is documented in the tool README. Use the v2 schema if available.
- If you are offline and cannot download any model weights, you can still complete the design and code skeleton of Parts A and B; the detector / verifier scores are not themselves important deliverables — the working pipeline is.
- The Opus encoding bitrate for telephony is typically 12–24 kbps depending on provider; benchmark at the actual negotiated bitrate for your target CPaaS.

## Acceptance criteria

- [ ] `asv_spoof.py` runs and emits the JSON report for each of six samples.
- [ ] A 2×6 table of (score, verdict) per sample and a per-class error rate.
- [ ] `watermark.py` runs end-to-end on bonafide audio and round-trips the payload.
- [ ] A 3-row codec-robustness table (raw / Opus 16k / G.711).
- [ ] A signed C2PA manifest attached to a watermarked audio file, verified by `c2patool`.
- [ ] A `NOTES.md` covering (a) spoof-detection failure cases, (b) the watermark-to-manifest chain, (c) the outbound-provenance design note with limits.

## Stretch goals

- **STIR/SHAKEN fusion.** Add a mock `stir_attestation` input to `asv_spoof.py`; implement the fusion logic from Chapter 6 that combines CM score, SV score, and attestation class into a single risk score; produce a 3×3 decision table.
- **Pindrop / Resemble Detect integration.** If you have API access to a commercial deepfake-detection service, add it as a third signal in the fusion and compare outcomes on your six-sample set.
- **Adversarial robustness.** Generate a watermarked sample, then apply common edits (tempo change ±5%, pitch shift ±1 semitone, 2 s crop) and measure detection score. Produce a chart.
- **Key rotation.** Add a second `signer-v2.cert`; sign a new manifest; verify both. Document the rotation procedure you would use in production.
- **Per-call linkage.** Encode a 32-bit call sequence number in the AudioSeal payload and demonstrate that the detector output can be matched back to a specific call-log row.

## Keep it

The watermark + manifest pattern will reappear in multimodal provenance tasks (mod-110) and in observability tasks (mod-112). Keep the `voice-provenance/` directory in your personal reference; it's the closest thing in this track to a reproducible provenance baseline.
