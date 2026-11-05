# exercise-03: Author SSML Prosody and a Phonetic Lexicon

**Estimated effort:** 2 hours · **Output:** a PLS lexicon with 20+ entries + an SSML pre-processing pass + an A/B listening sample demonstrating the delta

## Objective

Build the SSML + phonetic-lexicon layer for a specific voice-agent domain of your choice. By the end you will have an authored PLS document, a working text → SSML pre-processing pass, and before/after audio demonstrating that your layer actually changes pronunciation and prosody the way you intended.

This is Chapter 3 made concrete. The deliverable is not a lecture — it is a lexicon file, a pre-processor you could ship, and an audible demonstration that the work matters.

## Prerequisites

- [Chapter 3 — SSML Prosody and Phonetic Lexicons](../03-ssml-prosody-and-phonetic-lexicons.md).
- At least one TTS vendor account from exercise-02 **that supports `<phoneme alphabet="ipa">`** on a voice in your target locale. Azure, Polly, and Google Cloud TTS are the safest choices; verify tag honouring per voice before you start.
- Python 3.10+.
- A text editor that can display Unicode IPA. If not, an [IPA keyboard](https://ipa.typeit.org/) in the browser.
- 20–30 utterances from your target domain, drawn from realistic LLM outputs, containing known-problem vocabulary (drug names, product names, acronyms, numbers, dates, amounts).

## Steps

### Part A — Pick a domain and collect the problem words (20 minutes)

1. Pick a domain: healthcare, fintech, legal, logistics, telco, retail. Name a brand (your company or a plausible stand-in).
2. Curate 20 problem words in three categories:
   - **Domain vocabulary** — at least 10 (drug names for health, fund tickers for fintech, equipment models for logistics).
   - **Brand / product names** — at least 5 (your brand, product SKUs, feature names).
   - **Acronyms, dates, amounts** — at least 5 (hard-to-read numeric or date forms, abbreviations like `Dr.`, `St.`, `mg`, `UTC`).
3. Render the 20 utterances with a default voice and **no SSML**. Listen. Mark which ones the engine gets wrong, which are ambiguous, and which are fine.

### Part B — Author the PLS lexicon (40 minutes)

4. Create a `lexicon.pls` with the `application/pls+xml` schema; namespace it to `http://www.w3.org/2005/01/pronunciation-lexicon`.
5. For each problem word you marked wrong or ambiguous, add a `<lexeme>` with:
   - `<grapheme>` — the surface form the LLM will emit.
   - `<phoneme>` — IPA transcription (not `<alias>` unless the pronunciation really is "read this as a different word").
   - Locale (via `xml:lang` on the lexicon root, or per-lexeme if you are mixing locales).
6. Decide IPA for each entry by one of:
   - A linguist / SME reading aloud; you transcribe.
   - A trusted dictionary (Wiktionary with caveats; Cambridge / Merriam-Webster for common words; official brand phonetic guides for product names).
   - The vendor's pronunciation preview tool, iterated.
   
   Document the source for every entry.
7. Add a **provenance column** — who authored this entry, what their source was, when they signed off. This is the discipline Chapter 3 names; make it a column in the comments or a sidecar YAML.

### Part C — Build the text → SSML pre-processor (40 minutes)

8. Write a Python function `text_to_ssml(text: str, locale: str) -> str` that:
   - Wraps the text in `<speak version="1.1" xml:lang="...">`.
   - References the PLS via `<lexicon uri="..."/>` **if your target vendor supports external `<lexicon>`**. If not, implement inline `<phoneme>` injection: scan the text for each lexicon `<grapheme>` and wrap it with `<phoneme alphabet="ipa" ph="...">...</phoneme>`.
   - Applies `<say-as>` tags to recognised patterns: dollar amounts → `interpret-as="currency"`; MM/DD or MM/DD/YYYY → `interpret-as="date"`; phone-number-shaped sequences → `interpret-as="telephone"`; isolated alphanumeric codes → `interpret-as="characters"` or `interpret-as="digits"`.
   - Inserts strategic `<break time="200ms"/>` at natural information boundaries ("Your balance is <break/> …").
   - Does **not** rewrite words that are not in the lexicon or regex. Over-aggressive rewriting is a bug.
9. Add minimal unit tests — one per rule — asserting the input-to-output transformation.

### Part D — A/B listen and measure delta (20 minutes)

10. Render the 20 utterances in three conditions: (a) no SSML, (b) `<say-as>` only, (c) full SSML with lexicon.
11. Listen to each pair. For each utterance, record which condition is best and why.
12. Count the problem-word error rate (fraction of problem words mispronounced) per condition. The lexicon should reduce it meaningfully; if it does not, debug (is the vendor honouring `<phoneme>` on this voice? Is the IPA transcription correct? Is the lexicon being loaded?).
13. Save the three audio versions of 3–5 representative utterances as artefacts.

## Starter guidance

- **Start with the vendor's SSML sandbox** if they have one (Azure, Polly, Google all do). You can iterate on IPA faster there than by rebuilding the client each time.
- **One PLS per locale** — don't mix en-US and en-GB entries in one file. The IPA differs.
- **Test tag honouring empirically.** Submit a synthesis with and without each tag; if the audio is identical, the tag is being silently dropped.
- **Pronunciation is a judgement call.** Two SMEs will disagree. Pick a canonical source and document it; do not rebuild the lexicon every time a new reviewer weighs in.
- **Avoid over-prosody.** `<prosody rate="slow">` on every number sounds robotic. Pick 2–3 places where prosody actually improves comprehension; leave the rest to the engine.
- **Pre-processor failure modes to think through**: SSML injection via user-supplied text (user name contains `<`), malformed regex matches on dates (what if the LLM says "nineteen eighty-four"?), locale leaks (en-US dates in an en-GB session).

## Acceptance criteria

- [ ] A `lexicon.pls` with at least 20 entries across the three categories, each with IPA phoneme and provenance.
- [ ] A `text_to_ssml` pre-processor with at least 6 rules (lexicon injection, currency, date, digits/characters, telephone, break-insertion). Minimal unit tests passing.
- [ ] Rendered audio for the 20 utterances in all three conditions (no SSML / say-as only / full SSML + lexicon).
- [ ] Problem-word error rate measured and reported per condition.
- [ ] Short write-up (half a page): what improved, what did not, which lexicon entries the engine ignored (and why you think so).
- [ ] At least one tag that was silently dropped by the vendor (every vendor does this on something). Document which.

## Stretch goals

- **Portable SSML.** Build the pre-processor to emit two SSML dialects — a portable one (break, phoneme, say-as, emphasis) and a vendor-extended one (e.g., Azure `<mstts:express-as>` for empathetic turns). Switch via config.
- **Second vendor.** Run the same pre-processor output through a second vendor's TTS. Enumerate the tags that worked on one vendor but not the other. Decide which tags to keep in the portable core.
- **Lexicon-management tooling.** Build a small CLI: `lexicon add <grapheme> --ipa <...> --source <...>`, `lexicon list --locale en-US`, `lexicon test <word>` (renders the word through the vendor and plays it). Lexicons without tooling rot; a CLI is the lightweight discipline that keeps them clean.
- **Catch bad lexicon entries with regression tests.** Record expected audio for 5 problem words; run a weekly job that re-synthesises and compares (MFCC distance, or a simple spectrogram hash). Alert on drift.
- **LLM-driven lexicon candidate mining.** Scan a week of LLM outputs for rare words (low frequency in your corpus); flag them for lexicon review. Most production voice agents accumulate problem words faster than the maintainer notices.

## Keep it

The lexicon and the pre-processor are the two artefacts that compound. Every month you discover 3–5 new problem words; add them. Every vendor you switch to, run the pre-processor's tests against. Over a year this is the difference between "the agent sounds smart" and "the agent reads drug names like a tourist."
