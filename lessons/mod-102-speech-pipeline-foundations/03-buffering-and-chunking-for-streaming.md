# 03. Buffering and Chunking for Streaming Pipelines

## Why this chapter

A streaming voice pipeline is a dataflow graph of components that each prefer to work on a specific-size chunk of audio: Opus wants 20 ms frames, WebRTC VAD wants 10 / 20 / 30 ms frames, ASR WebSockets want whatever you decide to send them (often 100 ms–1 s at a time), TTS returns variable-sized chunks, codecs want power-of-two-sized buffers for the FFT. **You are the one who makes them agree.**

The buffering layer is not glamorous, and most tutorials skip it. It is also where almost every intermittent, hard-to-reproduce voice bug lives — crackle, drift, delayed barge-in, "sometimes the ASR returns stale partials." Learn the shape once, and the rest of the pipeline stops being flaky.

This chapter covers:

- Frame sizes you'll actually see (10 / 20 / 40 ms) and why.
- The ring buffer: the one data structure every streaming pipeline ends up re-inventing.
- Resampling: where to do it, how much it costs, which library to trust.
- The common anti-patterns — variable-frame concatenation, late-resample, cross-thread drift — and how to recognise them in a trace.

## Frame sizes: 10 / 20 / 40 ms is not a style choice

Every codec and every voice-processing component defines a **frame** — a fixed-length block of samples that is the unit of work. The sizes are not arbitrary:

| Frame size | Where it comes from | Where you'll see it |
|---|---|---|
| **10 ms** | WebRTC native frame size; shortest Opus frame | WebRTC `MediaStreamTrack` callbacks, WebRTC VAD, AEC3 processing step |
| **20 ms** | Opus default, RTP packetisation default | Opus-over-WebRTC, SIP/RTP, most VADs, streaming ASR chunks |
| **30 ms** | Opus supported mode | WebRTC VAD third option; some ASR SDKs |
| **40 ms / 60 ms** | Opus supported modes | Rare; telephony with high jitter tolerance |
| **80 ms** | Pipecat default audio frame size for pipelines <!-- needs-research: confirm current default in pipecat-ai repo at author time --> | Internal processor pipelines batching more work per frame |
| **~100 ms – 1 s** | ASR WebSocket message batching | Deepgram / AssemblyAI / provider streaming endpoints — dictated by server-side chunking |

The numbers 10, 20, 40 are not random. Opus defines frames of 2.5, 5, 10, 20, 40, 60, 80, 100, 120 ms (see RFC 6716 §2.1.4). WebRTC VAD accepts only 10, 20, or 30 ms. Every other voice component is downstream of one of those decisions, so you will see those sizes repeatedly.

Three operational numbers follow from a frame size at a given sample rate:

- **Samples per frame** = `frame_ms * sample_rate / 1000`. At 16 kHz, 20 ms = 320 samples.
- **Bytes per frame** (PCM16 mono) = samples × 2. 320 samples = 640 bytes.
- **Frame latency floor.** The pipeline cannot respond faster than one frame hop; a 20 ms frame means +20 ms minimum per component.

If you ever see frames of 7 ms, 13 ms, or 47.5 ms coming out of your pipeline, something concatenated two frames and split them in the middle. That's the first anti-pattern.

## The ring buffer

Every streaming component inside your pipeline will, at some point, want "the last N samples" or "exactly 320 samples now, no more, no less." The right data structure is a **ring buffer**: a fixed-size circular byte array with a read pointer and a write pointer.

```
   write ptr →
   ┌─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┐
   │ │ │█│█│█│█│█│█│ │ │ │ │   fixed-size buffer, wraps around
   └─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┘
            ↑ read ptr
```

Properties that matter in practice:

- **Bounded memory.** You decide the size at construction; it never grows. A 5-second buffer at 16 kHz / PCM16 is 160 KB — set it once, forget.
- **Constant-time writes and reads.** No allocation in the hot path.
- **Natural "pull N samples" interface.** Downstream components ask for the exact chunk they need; the buffer doesn't care if producer and consumer disagree on frame size.
- **Overflow policy.** If the consumer is too slow, the buffer either blocks or drops. Realtime voice chooses drop (the oldest audio), because blocking the producer would stall the capture thread and create audible gaps.

A minimal implementation for PCM audio in Python:

```python
import numpy as np
from threading import Lock

class AudioRingBuffer:
    """Fixed-capacity ring buffer over a 1-D int16 numpy array.
    Single-producer, single-consumer; a lock is still useful because reads
    and writes often come from different threads (capture callback, worker)."""

    def __init__(self, capacity_samples: int):
        self._buf = np.zeros(capacity_samples, dtype=np.int16)
        self._cap = capacity_samples
        self._read = 0
        self._write = 0
        self._filled = 0
        self._lock = Lock()

    def write(self, samples: np.ndarray) -> int:
        """Write as many samples as will fit; drop the oldest on overflow."""
        assert samples.dtype == np.int16
        with self._lock:
            n = len(samples)
            if n > self._cap:
                samples = samples[-self._cap:]
                n = self._cap
            # overflow: advance read ptr forward, discarding oldest audio
            if self._filled + n > self._cap:
                drop = self._filled + n - self._cap
                self._read = (self._read + drop) % self._cap
                self._filled -= drop
            first = min(n, self._cap - self._write)
            self._buf[self._write:self._write + first] = samples[:first]
            self._buf[:n - first] = samples[first:]
            self._write = (self._write + n) % self._cap
            self._filled += n
            return n

    def read(self, n: int) -> np.ndarray | None:
        """Return exactly n samples, or None if not enough are buffered."""
        with self._lock:
            if self._filled < n:
                return None
            out = np.empty(n, dtype=np.int16)
            first = min(n, self._cap - self._read)
            out[:first] = self._buf[self._read:self._read + first]
            out[first:] = self._buf[:n - first]
            self._read = (self._read + n) % self._cap
            self._filled -= n
            return out

    def available(self) -> int:
        with self._lock:
            return self._filled
```

The design choices to notice:

- **int16 throughout.** Don't mix PCM16 and float32 inside the buffer; convert at the boundaries.
- **Drop oldest on overflow, not newest.** In a conversation, the useful audio is "what the user just said," not "what they said seven seconds ago."
- **Exact-size reads.** `read(320)` returns 320 samples or nothing. Downstream components get the chunk they asked for or wait for more data.

You will almost never implement this yourself in production — `torchaudio`, `av`, `sounddevice`, or your framework's `AudioFrame` will cover it — but you need the mental model to debug when they misbehave.

## Resampling

Resampling is the operation that converts a stream from one sample rate to another — e.g., 48 kHz WebRTC → 16 kHz for ASR. It is *not* free; done badly, it introduces aliasing (false frequencies) that degrades ASR WER and TTS quality.

Three implementations you'll meet:

- **Linear interpolation.** Fast, terrible quality. Never use for speech.
- **Polyphase / sinc interpolation.** The industry-standard quality. `torchaudio.transforms.Resample`, `scipy.signal.resample_poly`, `libsoxr`.
- **soxr (`python-soxr`).** The high-quality reference. `librosa.resample` uses it by default when installed.

A typical resample call:

```python
# librosa
y16 = librosa.resample(y48, orig_sr=48_000, target_sr=16_000, res_type="soxr_hq")

# torchaudio
import torchaudio
resampler = torchaudio.transforms.Resample(
    orig_freq=48_000, new_freq=16_000, resampling_method="sinc_interp_kaiser",
)
y16 = resampler(y48_tensor)
```

Rules of thumb:

- **Downsample before expensive work, upsample at the last moment.** A 48 kHz stream processed as 48 kHz through VAD, ring buffer, and ASR wastes 3× the memory and CPU. Resample to 16 kHz as soon as it enters your domain.
- **Resample once per hop.** Don't resample 48 → 16 → 24 → 16 to feed four different components. Keep a canonical rate inside the pipeline and resample on egress.
- **Rate ratios matter.** 48 kHz → 16 kHz is a clean 3:1 integer ratio — fast and artefact-free. 48 kHz → 22.05 kHz is 160:147 — requires more taps, more CPU, more care. Prefer integer ratios where you have a choice.
- **For realtime: use a streaming resampler.** A one-shot `librosa.resample` on a growing buffer re-does work every call. `torchaudio.transforms.Resample` is an `nn.Module` with internal state; `python-soxr` has a streaming `ResampleStream`.

The asymmetry to remember: **upsampling cannot reconstruct what wasn't there.** If an 8 kHz telephone leg is upsampled to 16 kHz for ASR, the audio above 4 kHz is zero — no sibilance will appear. A provider that advertises "wideband ASR" on 8 kHz input is doing marketing, not DSP.

## Three anti-patterns

### Anti-pattern 1 — Variable-frame concatenation

**Symptom.** Opus encoder barks `OPUS_BAD_ARG`; VAD occasionally returns nonsense; ASR partial transcripts land with misaligned word timings.

**Cause.** You read "whatever's available" from your ring buffer and shove it to the next component. The chunks come out at 7 ms, then 23 ms, then 15 ms.

**Fix.** The ring buffer's `read(n)` must return **exactly `n` samples or nothing**. Downstream components get a fixed-frame-size feed. If upstream didn't produce enough yet, wait.

```python
# WRONG: variable chunks
while True:
    data = capture.read()       # returns whatever just arrived
    vad.feed(data)              # VAD expects 20 ms

# RIGHT: fixed-frame feed
FRAME = 320  # 20 ms at 16 kHz
while True:
    chunk = ring.read(FRAME)
    if chunk is None:
        sleep_until_more_data()
        continue
    vad.feed(chunk)
```

### Anti-pattern 2 — Late resample

**Symptom.** Perceived CPU cost higher than it should be. P95 latency jitter on ASR partials. Pitch-shifted or buzzy output at the end of a call.

**Cause.** You pass raw 48 kHz through VAD, ring buffer, and feature extraction, and resample only at the moment you call the ASR WebSocket. Three times the work; three times the memory; three times the chance to drop a packet on a context switch.

**Fix.** Resample at the pipeline **ingress**. Your canonical in-pipeline rate is 16 kHz PCM16 mono. Only the capture-and-resample stage sees the native rate. Downstream components are all 16 kHz. Resample *out* to 48 kHz or 8 kHz only inside the egress encoder.

```
┌─────────────┐   ┌──────────┐   ┌─────┐   ┌──────┐   ┌─────┐   ┌────────────┐
│ mic / phone │ → │ resample │ → │ VAD │ → │ ring │ → │ ASR │   │  canonical │
│ 48k / 8k    │   │  → 16k   │   │     │   │      │   │     │   │   16 kHz   │
└─────────────┘   └──────────┘   └─────┘   └──────┘   └─────┘   └────────────┘
```

### Anti-pattern 3 — Cross-thread drift

**Symptom.** Audio works for the first 30 seconds of a call, then starts crackling or echoing. The TTS and the ASR are both running but slightly out of sync with real time.

**Cause.** Two threads (or two devices) each claim to be running at 16 kHz, but their actual sample clocks differ by a few parts per million. Over seconds, their notions of "now" diverge. The ring buffer overflows (one side runs fast) or underflows (one side runs slow).

**Fix (options, in order of simplicity).**

- **Single clock.** Drive the whole pipeline from one hardware or system clock. Use the audio driver callback as your time source; don't mix a software timer with a hardware timer.
- **Adaptive resampling.** If two devices really do have independent clocks (microphone and speaker on different sound cards; two SIP legs), measure the drift and apply a tiny resampling correction (e.g., 16 000.3 → 16 000.0 effective Hz). Serious media servers do this transparently; cheap ones don't.
- **PLC (packet-loss concealment) and jitter buffers.** WebRTC and most SIP stacks have these; your job is to use them, not reinvent them. If you build your own transport over WebSocket, you inherit the responsibility.

Drift is the single most common "it worked on my laptop but fails in production" bug in voice. Your laptop has one audio clock. Production has a Twilio clock, your SFU's clock, your worker's clock, and your TTS provider's clock. Design for drift from day one.

## Picking your chunk sizes

A workable default stack for a 16 kHz production ASR pipeline:

| Stage | Chunk size | Rationale |
|---|---|---|
| Capture callback | 10 ms (160 samples) | Smallest useful unit; matches WebRTC native cadence |
| Ring buffer capacity | 5 s (80 000 samples, 160 KB) | Long enough to hold a single utterance; short enough to bound memory |
| VAD input | 20 ms (320 samples) | WebRTC VAD native; most feature extractors align here |
| ASR send chunk | 100 – 200 ms | Balance of network efficiency and end-of-speech responsiveness |
| TTS receive chunk | whatever the provider sends | Buffer into a ring; re-chunk to transport frame size on egress |
| Egress encode frame | 20 ms Opus / 20 ms µ-law | Match the transport's native cadence |

Document these numbers in the latency budget (per module 101, Chapter 5). "We send 100 ms chunks to the ASR" is a concrete decision; someone 18 months from now needs to be able to find it.

## Summary

- Frame sizes come from Opus, WebRTC, and ASR providers — 10, 20, 40 ms dominate. Match them.
- Ring buffers are the universal glue. Exact-size reads; drop oldest on overflow; one dtype.
- Resample once, at the ingress edge; keep a canonical in-pipeline rate.
- Variable-frame concatenation, late resample, and cross-thread drift are the three anti-patterns that explain most mysterious streaming bugs.

Next chapter: the codecs — Opus, G.711, PCM16 — and the knobs you'll actually have to set.
