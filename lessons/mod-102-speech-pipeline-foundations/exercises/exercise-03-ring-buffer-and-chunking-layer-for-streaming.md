# exercise-03: Ring Buffer and Chunking Layer for Streaming

**Estimated effort:** 2 hours · **Output:** a working `AudioRingBuffer` with tests + a drift-detection experiment

## Objective

Build the one data structure every streaming voice pipeline needs — a fixed-capacity audio ring buffer — and prove it survives the three anti-patterns from [Chapter 3](../03-buffering-and-chunking-for-streaming.md): variable-frame concatenation, late resample, and cross-thread drift.

The implementation is almost beside the point. The real deliverable is the **test suite**, because that's what you'll reuse in the next six modules when the next streaming pipeline you touch misbehaves at the seams.

## Prerequisites

- [Chapter 3 — Buffering and Chunking for Streaming Pipelines](../03-buffering-and-chunking-for-streaming.md).
- Python 3.10+ with `numpy`, `pytest`, and a resampler library (`torchaudio`, `soxr`, or `scipy`).

## Steps

### Part A — Build the ring buffer (45 minutes)

1. Implement a single-producer / single-consumer ring buffer for PCM16 mono audio. API:

   ```python
   class AudioRingBuffer:
       def __init__(self, capacity_samples: int): ...
       def write(self, samples: np.ndarray) -> int: ...  # returns samples written
       def read(self, n: int) -> np.ndarray | None: ...  # None if < n samples available
       def available(self) -> int: ...                   # samples currently buffered
       def clear(self) -> None: ...
   ```

   Non-negotiable properties:

   - **Fixed capacity.** The underlying array is allocated in `__init__` and never grown.
   - **Exact-size reads.** `read(320)` returns 320 samples or `None`.
   - **Drop oldest on overflow.** If a write would exceed capacity, the oldest samples are discarded. The write succeeds.
   - **Thread-safe.** Protect the pointers and fill counter with a lock (standard library `threading.Lock` is enough).
   - **No allocation in the hot path.** `write` and `read` do not create new arrays beyond the output buffer itself.

2. You may copy or adapt the sketch in Chapter 3. You must be able to explain every line.

### Part B — Build the test suite (45 minutes)

3. Write `pytest` tests for:

   - **Round-trip correctness.** Write N samples, read N samples, assert equality.
   - **Wrap-around.** Fill the buffer across a wrap boundary; confirm reads return the right samples in the right order.
   - **Underflow.** `read(n)` when `available() < n` returns `None` and does not advance pointers.
   - **Overflow (drop-oldest).** Write capacity+k samples; confirm that the oldest k samples are dropped and the newest `capacity` samples are retained in order.
   - **Exact-frame feed.** Simulate a jittery producer that writes between 50 and 500 samples at a time; a consumer that pulls exactly 320 samples per step; and confirm that every consumer read gets exactly 320 samples (or `None` when starving).

### Part C — The drift experiment (30 minutes)

4. Simulate cross-thread drift (Anti-pattern 3 in Chapter 3):

   - Spawn two threads. Thread A ("producer") pretends to be a 16 000 Hz capture source: every 10 ms it writes 160 samples. Thread B ("consumer") pretends to be a 15 998 Hz downstream consumer: every 10 ms it reads 159.98 samples on average (you can approximate by reading 160 nine times out of ten and 159 one time out of ten).
   - Run for 60 simulated seconds. Log `available()` over time.
   - Plot the buffer occupancy curve. It should drift upward (producer running faster than consumer) until it hits capacity, then plateau as drop-oldest kicks in. **If your plot is flat, you have a bug.**

5. **Reverse the drift direction** (consumer faster than producer) and plot again. Expected: occupancy trends toward zero, consumer gets `None` increasingly often.

6. Write a one-paragraph note on *where* in a real voice pipeline each direction of drift would come from (hint: mic clock vs. system clock; two SIP legs on different carriers; TTS provider over WebSocket vs. local Opus encoder).

## Starter guidance

- The sketch in Chapter 3 compiles and runs; do not retype it, do type through it line by line.
- For the jittery-producer test, use `numpy.random.default_rng` for reproducibility; seed the generator.
- For the drift experiment, you do not need real timing — use a monotonic simulated clock. Real `time.sleep` is unnecessary and makes the test slow.
- Don't use `queue.Queue` as a shortcut. It allocates per-write and doesn't have the exact-size-read semantic you need.

## Acceptance criteria

- [ ] Working `AudioRingBuffer` with all five methods implemented.
- [ ] Passing `pytest` suite covering round-trip, wrap-around, underflow, overflow-drop-oldest, and exact-frame-feed semantics.
- [ ] Drift experiment reproduced in both directions with plots showing the expected occupancy curves.
- [ ] One-paragraph note mapping each drift direction to a real pipeline scenario.
- [ ] Explanation (one paragraph) of why you chose single-lock / lock-free / `collections.deque` / whatever — not just "that's what the chapter said."

## Stretch goals

- **Re-chunker.** Build an `ExactFrameChunker(ring, frame_samples)` wrapper that exposes an iterator yielding one exact-size frame at a time. This is the shape most downstream components actually want.
- **Streaming resampler integration.** Put a `torchaudio.transforms.Resample` between a 48 kHz producer and a 16 kHz consumer that both share one ring buffer. Verify the output rate matches a reference 440 Hz sine's expected pitch.
- **Multi-consumer ring.** Allow N consumers to each have their own read pointer over the same buffer (so a VAD and a recorder can both tail the same capture). Decide: do you track per-consumer slow-reader starvation separately? How?
- **Benchmark.** Measure per-frame `write`+`read` latency at a 10 ms cadence on your machine. Is the lock cheap enough to run inside a WebRTC audio callback? (Hint: on most systems, yes; but measure.)

## Keep it

The `AudioRingBuffer` + test suite is the artefact you'll port into every subsequent lab. The drift experiment is also a reference — when a production pipeline exhibits "works for 30 seconds, then crackles," your first debug step is "plot buffer occupancy over time" and the shape should look familiar.
