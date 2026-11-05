# 03. SSML Prosody and Phonetic Lexicons

## Why this chapter

The TTS engine is a function from text to audio. Text is a wildly under-specified input. "I owe you $45.17 by 11/2" could be read as a sentence or a phone number; "Dr. Jones lives on Dr. Avenue" has two different `Dr.` expansions; "the patient is on tirzepatide" is the moment a healthcare agent either sounds like a professional or sounds like a tourist trying their first drug-name reading. The engine will guess. Sometimes it will guess well; on domain vocabulary it will usually guess wrong.

Your two levers are:

1. **SSML** — the W3C Speech Synthesis Markup Language, a declarative annotation layer on top of the text that lets you control prosody, pauses, emphasis, pronunciation, and interpret-as rules.
2. **A phonetic lexicon** — a curated list of your domain's problem words with explicit pronunciations in IPA (International Phonetic Alphabet) or X-SAMPA, loaded into the synthesis context either inline (`<phoneme>`) or as an external PLS (Pronunciation Lexicon Specification) document.

This chapter teaches you how to use both — W3C SSML 1.1 as the reference, provider subsets as the reality, and a shippable lexicon as the artefact that turns correct pronunciation from an engineering wish into a maintained asset.

## SSML 1.1 at a working level

SSML 1.1 is specified in [W3C Speech Synthesis Markup Language Version 1.1](https://www.w3.org/TR/speech-synthesis11/). You do not need to read it top to bottom. You need to know these elements and what each actually controls.

### The envelope

Every SSML document is wrapped in a `<speak>` element with an `xml:lang` attribute:

```xml
<speak version="1.1" xml:lang="en-US">
  Hello, this is the agent. How can I help you today?
</speak>
```

Some providers accept the body text without the outer `<speak>`; some require it. When in doubt, include it — it costs nothing and prevents one class of "tags were silently ignored" failure.

### Prosody — `<prosody>`

Controls rate, pitch, and volume on an inline span:

```xml
<prosody rate="slow" pitch="-2st" volume="-3dB">
  Your account number is three-one-seven-four.
</prosody>
```

- `rate` — `x-slow` / `slow` / `medium` / `fast` / `x-fast` / percentage (`80%`).
- `pitch` — relative (`+2st`, `-5%`) or named (`low`, `medium`, `high`, `x-high`).
- `volume` — `silent` / `x-soft` / `soft` / `medium` / `loud` / `x-loud` / dB offset (`-6dB`).

Use `<prosody>` sparingly. Over-use produces a voice that sounds mechanically poked; the modern expressive engines in [Chapter 4](04-expressive-vs-neutral-tts-and-voice-selection-ux.md) are usually better at natural prosody than a hand-tuned `<prosody>` block. Reach for it when you have a specific delivery requirement — a number read slow for clarity, an apology read quietly — and let the engine own the rest.

### Pauses — `<break>`

Inserts silence:

```xml
Please hold<break time="500ms"/> while I look that up.
```

- `time` — `250ms`, `1s`, etc.
- `strength` — `none` / `x-weak` / `weak` / `medium` / `strong` / `x-strong`. Loosely corresponds to pause durations the engine chooses; use `time` for exactness.

Common pattern: `<break time="200ms"/>` before a delivered piece of information ("Your balance is <break time="200ms"/> four hundred and twenty-two dollars") to give the user a cue to attend.

### Emphasis — `<emphasis>`

```xml
This is <emphasis level="strong">urgent</emphasis>.
```

Levels: `reduced` / `moderate` / `strong`. Engines implement this with pitch, rate, and volume tweaks — the specific sound differs per voice.

### Pronunciation — `<phoneme>`

The most important tag for domain vocabulary. Overrides the engine's pronunciation with an explicit phonetic transcription:

```xml
Please take your <phoneme alphabet="ipa" ph="tɪrˈzɛpətaɪd">tirzepatide</phoneme> once weekly.
```

- `alphabet` — `ipa` (International Phonetic Alphabet) or `x-sampa` (ASCII-safe version of IPA). Some providers accept vendor-specific phoneme sets.
- `ph` — the transcription.

**This is the tag your drug-name / product-name / acronym layer depends on.** Verify per-provider that it is supported for the voice and locale you use; some providers support `<phoneme>` for a subset of their voice catalogue.

### Interpret-as — `<say-as>`

Tells the engine how to read a token that could be interpreted multiple ways:

```xml
Call me at <say-as interpret-as="telephone">4155550198</say-as>.
Your order ships on <say-as interpret-as="date" format="mdy">11/02/2026</say-as>.
The code is <say-as interpret-as="characters">ABC123</say-as>.
Your balance is <say-as interpret-as="currency">$1,234.56</say-as>.
```

Standard values include `cardinal`, `ordinal`, `digits`, `fraction`, `unit`, `date`, `time`, `telephone`, `address`, `expletive`, `verbatim` / `spell-out`, `characters`. Provider subsets vary; some accept `interpret-as="currency"` with a `language` attribute, others want the currency marker in the text directly. Read the docs.

**Rule of thumb: if the token is a number, a date, a time, a phone number, a currency amount, an address, or an acronym, wrap it in `<say-as>`.** The engine will usually get the common case right, but it will get the long tail wrong — six-digit sequences read as two three-digit sequences, dates in the wrong locale order, currency misparsed in the wrong currency.

### Lexicon — `<lexicon>`

References an external PLS (Pronunciation Lexicon Specification) document:

```xml
<speak version="1.1" xml:lang="en-US">
  <lexicon uri="https://example.com/lexicons/medical.pls" type="application/pls+xml"/>
  Please take your tirzepatide once weekly.
</speak>
```

The engine applies the lexicon's pronunciations as substitutions. Provider support is **inconsistent**: Azure and Polly support external `<lexicon>` for most voices; OpenAI, ElevenLabs, Cartesia, and several others do not — they expect inline `<phoneme>` or a vendor-specific inline mechanism. [Chapter 5 of the W3C PLS spec](https://www.w3.org/TR/pronunciation-lexicon/) is the reference; it is readable in an hour.

### Audio — `<audio>`

Splices pre-recorded audio:

```xml
Your call has been accepted.
<audio src="https://example.com/ding.wav">[ding]</audio>
A representative will join shortly.
```

Useful for voicemail stingers, hold music snippets, and the "we've updated this prompt — here's the human-recorded greeting we baked" case. The inner text is a fallback if the audio cannot be fetched.

### What SSML 1.1 has that you will rarely use

- `<voice>` — switch voice mid-document. Supported inconsistently; usually easier to synthesise separate clips and splice.
- `<lang>` — switch language mid-document. Supported more broadly than `<voice>`; essential for code-switched utterances in multilingual agents (see Module 103 Chapter 6 for the ASR side).
- `<p>` and `<s>` — paragraph and sentence markers. Mostly handled by the engine's internal sentence segmentation; setting them helps on text that is already hand-formatted.
- `<mark>` — a named timepoint in the audio stream, surfaced via alignment events. Useful for synchronising animations or triggering downstream actions ("at the end of the greeting, start the hold music").

## Provider SSML subsets — the reality

No vendor implements SSML 1.1 completely. The subsets below are the shape; read the dated vendor doc at integration time.

- **Azure AI Speech** — the broadest SSML coverage of the major vendors. Full prosody, break, emphasis, phoneme (IPA + SAPI + UPS), say-as, external lexicon, mark, voice, lang. Plus the Microsoft extension namespace `mstts:` for `<mstts:express-as style="cheerful">`, `<mstts:silence>`, `<mstts:viseme>`, and `<mstts:ttsembedding>`. ([Azure SSML reference](https://learn.microsoft.com/azure/ai-services/speech-service/speech-synthesis-markup).)
- **AWS Polly** — strong SSML support including prosody, break, emphasis, phoneme (IPA + X-SAMPA), say-as, pause, and Amazon extensions like `<amazon:effect name="whispered">`, `<amazon:auto-breaths>`, and `<amazon:domain name="news">` on some voices. Generative Polly voices accept a narrower SSML surface than Neural voices; verify per engine. ([Polly SSML tags](https://docs.aws.amazon.com/polly/latest/dg/supportedtags.html).)
- **Google Cloud TTS** — supports prosody, break, say-as, mark, audio, phoneme (IPA + X-SAMPA), emphasis, sub, voice-specific tags. Chirp 3 HD voice behaviour on SSML can differ from the classical WaveNet / Neural2 line; read the per-voice notes. ([Google Cloud TTS SSML](https://cloud.google.com/text-to-speech/docs/ssml).)
- **ElevenLabs** — historically a reduced SSML surface; the Flash and Turbo models interpret a limited tag set (`<break>` reliably, `<phoneme>` on some voices and locales). Expressive delivery is more usefully steered via text prompting and the ElevenLabs "voice settings" (stability, similarity, style) than via SSML. Verify current support against the docs at integration time.
- **OpenAI TTS** — `tts-1` / `tts-1-hd` have minimal SSML support. `gpt-4o-mini-tts` takes **natural-language style instructions** instead — "say this in a cheerful customer-service voice, slowly, with a short pause before the balance" — which is closer in spirit to SSML than to a tag set, but different in substance. Treat it as a different ergonomic surface, not a drop-in SSML implementation.
- **Cartesia / PlayHT / LMNT** — subset support for `<break>` and `<phoneme>`; vendor-specific control over emotion / affect via API parameters rather than SSML. Check each vendor's doc for the tags their current model actually honours; silent tag drops are common.

Two consequences:

- **Do not rely on an SSML tag without testing it.** Submit a synthesis request with and without the tag; listen for the difference; if they sound identical, the tag is being silently dropped.
- **Portable SSML is a smaller surface than full SSML.** For a multi-vendor pipeline (primary + fallback), restrict yourself to the intersection: `<break>`, `<say-as interpret-as>` on the common values, `<phoneme ipa>`, `<emphasis>`. Treat `<prosody>`, `<voice>`, `<mstts:express-as>`, and external `<lexicon>` as vendor-specific extensions.

## The phonetic lexicon as a shippable artefact

The "drug names, product names, acronyms" layer is the single highest-leverage SSML investment for most voice agents. It is not a one-off — it is a maintained asset.

### Scoping the lexicon

Start with four categories:

- **Product / brand names.** Your SaaS, your service, your SKU IDs. The TTS has never heard them; it will guess by grapheme-to-phoneme rules.
- **Industry vocabulary.** Drug names (healthcare), legal terms (legal tech), part numbers (field service), financial instruments (fintech).
- **Acronyms.** Should each be read as letters (`EU` = "ee-you") or as a word (`NATO` = "nay-toe")? Default engine heuristics guess, often wrong.
- **Proper nouns in your user base.** Common customer names, city names, street names in your service area. Important for confirmation turns ("I'm sending this to Siobhán at…").

### Authoring entries

For each entry you need five fields:

| Field | Example | Note |
|---|---|---|
| grapheme | `tirzepatide` | The surface form the LLM will emit |
| phoneme (IPA) | `tɪrˈzɛpətaɪd` | The target pronunciation |
| locale | `en-US` | A word's pronunciation is locale-specific |
| part-of-speech / disambiguator | `n` / "drug" | For homographs ("live" verb vs. adjective) |
| source / sign-off | "Dr. Carter, 2026-02-14" | Who approved this pronunciation and when |

The last column is the discipline. A lexicon without provenance is a guessbook; a lexicon with provenance is an auditable asset.

Representation choice:

- **IPA** is the standard, linguist-readable, and what `<phoneme alphabet="ipa">` wants. Requires a tool that can enter Unicode IPA; a cheatsheet and an online IPA keyboard are enough.
- **X-SAMPA** is ASCII-safe IPA; easier in some toolchains, less readable.
- **Vendor-specific phoneme sets** (Microsoft UPS, SAPI, AT&T) may be more accurate on that vendor's voice but are not portable.

**Pick IPA unless you have a specific reason not to.** It travels across vendors, which the proprietary sets do not.

### Shape of a PLS document

```xml
<?xml version="1.0" encoding="UTF-8"?>
<lexicon version="1.0"
      xmlns="http://www.w3.org/2005/01/pronunciation-lexicon"
      alphabet="ipa" xml:lang="en-US">
  <lexeme>
    <grapheme>tirzepatide</grapheme>
    <phoneme>tɪrˈzɛpətaɪd</phoneme>
  </lexeme>
  <lexeme>
    <grapheme>EU</grapheme>
    <alias>European Union</alias>
  </lexeme>
  <lexeme>
    <grapheme>NATO</grapheme>
    <phoneme>ˈneɪtoʊ</phoneme>
  </lexeme>
  <lexeme>
    <grapheme>Rook Insurance</grapheme>
    <phoneme>rʊk ɪnˈʃʊrəns</phoneme>
  </lexeme>
</lexicon>
```

`<alias>` is the "read-as" form — useful for acronyms that should be expanded ("EU" → "European Union") rather than pronounced as a word.

### Deployment

- **Vendor-managed PLS (Azure, Polly).** Upload the PLS document to the vendor's lexicon store; reference it in requests by name or by `<lexicon uri>`. Pros: no per-request overhead; cons: vendor-locked format.
- **Inline `<phoneme>` injection.** Pre-process the LLM response to wrap known-problematic grapheme sequences in `<phoneme>` tags using a lookup against the lexicon. Pros: works with any vendor that supports `<phoneme>`; cons: adds a pre-processing pass and you own the lookup logic.
- **Both.** Vendor-managed PLS as the primary surface for providers that support it; inline `<phoneme>` injection as the fallback path for providers that don't. This is the usual production shape.

### Lifecycle

- **Who owns the lexicon.** A single file in your voice-agent repo, reviewed on PR, with a required reviewer who can read IPA. Not a config spread across three vendor consoles.
- **How entries get in.** A listening session against the agent's current output on in-domain text catches the real problems. Build a "bad-pronunciation report" channel — PMs, subject-matter experts, and users can submit audio clips of bad renders with the correct pronunciation. The maintainer promotes them into the lexicon.
- **When entries get retired.** When the engine or model revision subsumes them (test the entries you wrote six months ago against the current model; if the engine now says it correctly without your override, drop the entry). Dead entries rot the lexicon.

## A worked SSML example — the shape of production text

The LLM emits a plain-text response. A pre-processing layer converts it to SSML for the TTS. For a healthcare agent, that layer might turn:

> Your refill for tirzepatide, order number A-4017, is scheduled for 11/02. Please confirm your date of birth: 01/14/1962.

into:

```xml
<speak version="1.1" xml:lang="en-US">
  <lexicon uri="https://rook.example.com/lexicons/health-v4.pls" type="application/pls+xml"/>
  Your refill for
  <phoneme alphabet="ipa" ph="tɪrˈzɛpətaɪd">tirzepatide</phoneme>,
  order number <say-as interpret-as="characters">A-4017</say-as>,
  is scheduled for <say-as interpret-as="date" format="md">11/02</say-as>.
  <break time="300ms"/>
  Please confirm your date of birth:
  <say-as interpret-as="date" format="mdy">01/14/1962</say-as>.
</speak>
```

Notice the choices:

- Drug name wrapped in `<phoneme>`, with the PLS loaded for coverage of anything else in the medical vocabulary.
- Order number read character-by-character (not "a dash four thousand seventeen").
- Dates wrapped in `<say-as interpret-as="date">` with the format so the engine knows this is month/day rather than day/month.
- A short pause before the confirmation prompt, giving the user time to attend.
- No `<prosody>` — the base voice's prosody is usually better than a hand-tuned override.

**The pre-processing layer is a first-class piece of infrastructure.** Hand-authoring SSML every turn does not scale; letting the engine guess every turn does not work. The layer lives between the LLM and the TTS, is unit-testable, and is where the lexicon, the `<say-as>` heuristics, and any provider-specific tag quirks live.

## Common anti-patterns

- **Hand-authoring SSML in prompts.** You cannot ask the LLM to emit well-formed SSML reliably; it will emit invalid XML, drop closing tags, and inline tags inside words. Emit plain text from the LLM and transform it in a dedicated pass.
- **Tags that the vendor silently drops.** Your `<prosody rate="slow">` on ElevenLabs Flash reads at normal speed because Flash doesn't honour it. Test every tag you use per vendor.
- **IPA transcriptions sourced from Wiktionary.** Wiktionary IPA is often British English where you want General American, or wrong for a specific sense. Treat Wiktionary as a hint, confirm with an audio pronunciation you trust, and document the source in the lexicon.
- **Lexicon entries that fight the engine's good guesses.** You added `tirzepatide` in February; the vendor updated their G2P model in April and now gets it right; your override is now applying a subtly-wrong 2026-era pronunciation. Re-test entries on model revisions.
- **One lexicon for all locales.** `tomato` is `təˈmeɪtoʊ` in en-US and `təˈmɑːtoʊ` in en-GB. One PLS per locale, or per-entry locale tags within one PLS.
- **Treating `<say-as interpret-as="characters">` as free.** Character-by-character reads sound stilted on long strings. For a 10-digit confirmation code, consider `interpret-as="digits"` plus manual `<break>` grouping (`<say-as interpret-as="digits">415</say-as> <break time="100ms"/> <say-as interpret-as="digits">555</say-as>...`).

## Summary

- SSML 1.1 is the standard; provider subsets are the reality. The portable core is `<break>`, `<say-as>` on common values, `<phoneme>` in IPA, `<emphasis>`.
- A pre-processing layer between the LLM and the TTS converts plain-text to SSML. The lexicon lives there; `<say-as>` heuristics live there; vendor-specific quirks live there.
- A phonetic lexicon is a maintained asset: entries with grapheme, IPA phoneme, locale, disambiguator, and provenance; reviewed on PR; lifecycle-managed as the engine evolves.
- Test every SSML tag against every vendor you ship on. Silent tag drops are the default failure mode.
- Prefer the engine's prosody over hand-tuned `<prosody>` on modern expressive voices.

Next chapter: expressive vs. neutral TTS, voice selection, and keeping voice identity consistent across turns.
