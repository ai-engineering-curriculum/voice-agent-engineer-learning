# 04. Expressive vs. Neutral TTS and Voice Selection UX

## Why this chapter

By 2026, "neural TTS" is table stakes. Every managed vendor produces voices that are intelligible, well-paced on common English, and comfortable in a short clip. The split that matters now is **expressive vs. neutral** — does the voice modulate emotion, emphasis, questions, laughter, hesitations, and tempo the way a human would, or does it deliver every sentence at the same affective baseline?

Expressiveness is not universally better. A bank-balance read-out in a cheerful upbeat voice is uncanny; a hospital triage line that laughs through an urgent message is a trust failure. The right setting depends on the product, the brand, the use case, and the user population. This chapter is the decision framework — what each category of voice is good for, how to pick, and how to manage voice-identity UX so the user doesn't notice three different speakers in one five-turn conversation.

## The expressive / neutral split

### Neutral TTS

The classical neural-TTS output. Clear, consistent, broadcast-grade; no laughter, no sighs, limited emotional range. Prosody is appropriate but **unmodulated** — the voice delivers information cleanly and does not editorialise.

Representative:

- **AWS Polly Neural**, **Google Chirp 3 HD** (classical variants), **Azure Neural Voices** (standard, without `mstts:express-as` style).
- **OpenAI `tts-1`** — fast, neutral, no steering.
- Historical ElevenLabs Multilingual v2 with default style settings.

Good for:

- Account-balance read-outs, order-status updates, policy disclosures, IVR menus.
- Any context where expressiveness could read as inappropriate or manipulative (health, bereavement, legal, debt collection).
- High-volume synthesis where cost and consistency matter more than personality.
- Multilingual products where neutral delivery is the only consistent surface across locales.

### Expressive / conversational TTS

The current generation of voices that can laugh, sigh, interject, emphasise, question-rise, pause for effect, and deliver non-neutral emotions. Either prompted ("cheerful", "sympathetic", "urgent") or inferred ("empathic" voices like Hume EVI read the user's voice and adjust).

Representative:

- **ElevenLabs Conversational** — the current expressiveness reference for a general-purpose conversational voice; voice-identity preservation across emotional ranges.
- **Hume EVI** — not just an expressive TTS; a full voice-agent stack with a prosody model that *infers* the user's affect from their voice and conditions the output accordingly.
- **Cartesia Sonic Turbo** / **Sonic 2** — low-latency expressive voices with explicit affect / emotion control.
- **OpenAI `gpt-4o-mini-tts`** — steerable by natural-language style instructions ("say this in a calm sympathetic tone").
- **AWS Polly Generative** — the current Polly flagship with broader expressive range than Neural.
- **Azure Neural Voices with `<mstts:express-as>`** — the long-standing enterprise expressive surface; style catalogue includes `cheerful`, `sad`, `empathetic`, `newscast`, `customerservice`, etc., per voice.

Good for:

- Customer support, where the voice conveying sympathy at the right moment is the difference between "robot" and "representative."
- Consumer-facing assistants and companions where the voice *is* the brand.
- Drama / interactive fiction / game NPCs.
- Any turn where emotional register matters more than throughput.

Bad for:

- Contexts where expressed emotion is inappropriate (medical dosage instructions, legal advisement).
- Accessibility read-outs where users expect consistent, flat delivery.
- Anywhere "sincere empathy from a machine" would feel manipulative to your user population.

### The in-between: steerable expressive

The modern norm. `gpt-4o-mini-tts`, Azure's `<mstts:express-as>`, ElevenLabs voice-settings (stability / style), Cartesia's emotion controls — each gives you a single voice whose expressive register you steer per utterance. A sympathetic register for the apology turn; a neutral register for the balance read-out; a cheerful register for the greeting. Same voice, different delivery.

This is usually the right default for a modern voice agent: **one voice, context-conditional expressiveness.**

## The A/B drill

Do not pick expressive vs. neutral from the vendor's demo page. Pick from an A/B test on your text with your listeners. The drill:

1. **Sample 20–30 utterances from your production LLM output** — real responses to real inputs, covering the registers your agent actually produces (greeting, task completion, error, apology, delay, hand-off).
2. **Render each in both expressive and neutral variants** on 2–3 shortlisted vendors. Keep voice identity, locale, and SSML constant.
3. **Pair-wise listening test** with 10–30 listeners drawn from your target user population — not engineers, not UX researchers who already know the brief. Use ITU-T P.808 as the methodology spec; crowdsource via Prolific or UserTesting if you don't have an in-house listener pool.
4. **For each pair, ask three questions:**
   - Which voice would you prefer to receive this response from?
   - Which sounds more trustworthy?
   - Which sounds more appropriate for the content?
5. **Stratify by register.** Expressive wins on apologies and greetings. Neutral wins on numbers and disclosures. If you do not stratify, you will miss the per-register answer and pick a global winner that is wrong in half your turns.
6. **Report preference with CIs.** 15 listeners × 25 pairs is not a lot; use clustered bootstrap over listener-pair preferences to get a defensible preference rate.

The output is not "expressive beat neutral 60–40." The output is "**for apology turns, expressive beat neutral 72% (CI 65–78%). For balance read-outs, neutral beat expressive 68% (CI 60–75%).**" That is a decision.

## Voice selection UX

A voice agent speaks to the user. That voice becomes part of the product's identity. Three UX questions that production teams get wrong:

### Does the user pick the voice, or do you?

The three patterns:

- **Fixed brand voice.** One voice, chosen by you, used for all users. The voice is part of the brand; users learn to associate it with the product. Best when brand consistency dominates (banking, government, airline) or the voice-selection UX is a distraction.
- **User-selectable from a curated set.** A small catalogue (3–6 voices) with clear personas (friendly / professional / warm / crisp), gender variety, maybe locale variety. User picks at onboarding or in settings. Best for consumer assistants where personal fit matters (fitness, productivity, companion apps).
- **Dynamic / inferred.** The agent picks the voice based on context — locale, user demographics, product tier. Rarely a good idea; feels inconsistent and often falls into bias-coded assumptions ("we gave the female voice to the beauty queries").

Default to **fixed brand voice** unless there is a clear reason to offer choice. One well-chosen voice beats a mediocre selector every time.

### Can the voice change between turns of the same conversation?

**No.** The user has enrolled a mental model of who they are talking to. Changing voice mid-conversation reads as "I was transferred" or "the system is confused." Stick with the same voice for the duration of the session.

Three places where teams accidentally change the voice:

- **Fallback paths.** Primary TTS fails; fallback uses a different voice. The user hears the switch and loses trust. Choose the fallback voice to be as close as possible; if you can't match it, consider degrading gracefully (a quick "one moment" in the fallback voice, then back) rather than hiding the change.
- **Locale shifts.** The agent opens in `en-US`, the user asks something in Spanish, the agent switches to `es-MX`. If the `es-MX` voice is a different identity, the user notices. If the same voice-ID is available across locales (ElevenLabs Multilingual, Azure's multilingual voices), use it; if not, acknowledge the switch verbally the first time.
- **Model upgrades.** Vendor releases a new model revision with subtly different voice characteristics. Pin voice-ID + model-revision explicitly; do not accept automatic model upgrades without a listening pass.

### Voice-identity consistency across turns

Even on the same vendor, same voice-ID, you can get drift. Known sources:

- **Random seed, where exposed.** ElevenLabs, Cartesia, and others expose a seed. If you reseed per turn, voice character fluctuates (pitch, cadence shifts). Pin a seed per session, not per turn.
- **Style / stability parameters.** ElevenLabs' stability control (0–1) and style control trade consistency for expressiveness. Lower stability = more expressive but less consistent; higher stability = flatter but more consistent. Tune this in combination with the register decision above, not independently.
- **Emotion / style switching.** Switching Azure `<mstts:express-as style>` between turns audibly changes the voice. Change styles within a turn (where the engine blends), or warm-up the next style with a short neutral bridge — not at an abrupt turn boundary.
- **SSML prosody overrides.** Heavy `<prosody>` on some turns and not others makes the voice feel like two different people. Apply prosody consistently or don't apply it.

**Design test:** record three consecutive agent turns in a realistic conversation; listen; ask yourself "does this sound like one person speaking?" If no, debug.

## Localisation and voice-catalogue management

Two patterns for multilingual deployments:

- **One voice-ID per locale, consciously matched.** You select the voice for each locale separately, aiming for a consistent "character" across locales (approximate age, gender, warmth, register). The user hears a different voice when they switch languages, but each locale feels right on its own. Cheaper, broader vendor support.
- **One multilingual voice-ID across locales.** ElevenLabs Multilingual v2, Azure's multilingual voices, and a few others provide a single voice-ID that speaks many languages with consistent identity. More expensive, narrower vendor list; the identity consistency is often worth it for truly code-switched or multi-locale users.

Either way: **the voice catalogue is a curated asset.** Not every voice in every locale from every vendor. The voices your product uses, with per-voice metadata (gender, age-coded, register, locale, licence terms, provenance — see [Chapter 5](05-voice-cloning-consent-and-provenance.md)), under version control.

## The branding conversation you will have

A product manager asks "can we make the voice sound more premium / warmer / more trustworthy?" This is an actual request, not a bike-shed. Four moves in your repertoire:

1. **Pick a different voice-ID.** Cheapest, usually best. Modern catalogues have the range; pick one that matches the brief and A/B test it.
2. **Change the register per turn with SSML or style controls.** If the product wants to sound "warmer on apologies," add `<mstts:express-as style="empathetic">` or steer `gpt-4o-mini-tts` with natural-language style per turn.
3. **Change the pacing.** A slightly slower rate, a touch more pause after commas, feels more thoughtful. `<prosody rate="95%">` and `<break time="150ms"/>` at sentence boundaries get you there.
4. **Clone a specific voice.** A professional voice actor, with consent and a licence, for the brand. This is the Chapter 5 territory — expensive, legally fraught, high upside when it's right. Reach for it last.

Resist the move to "we need a more expressive engine" when the actual fix is a different voice selection or a tuned register per turn. Switching engine is a quarter of work; switching voice-ID is an afternoon.

## Common anti-patterns

- **Picking a voice on marketing demos.** Vendors demo on short, well-written prompts in favourable registers. Your text includes six-digit account numbers, formal legalese, and the word "unfortunately." Demo on your text.
- **Expressive everywhere.** A sympathetic voice delivering "your card was declined" is correct; the same sympathetic voice delivering "your balance is $4,217.63" is weird. Stratify by register.
- **User-selectable voice as a six-item grid.** Analysis paralysis. Three to four distinct-feeling voices beat eight similar-feeling ones.
- **Voice-ID upgrade with no listening pass.** Vendor announces "we've improved voice X"; you upgrade in production; users complain the agent "sounds different today." Pin voice-ID + model-revision; opt into upgrades with a listening pass.
- **A brand voice that was never A/B tested against your users.** The CEO loves it; the users don't. The voice is part of the product; validate it like any other product decision.
- **Reusing the same voice-ID across two products with different brand contracts.** The user hears "the X voice" on your consumer app and then on your enterprise product; the identity bleeds across. Separate catalogues per product.
- **Changing voice on fallback.** The user notices. Design the fallback for identity-similarity, not just latency.

## Summary

- The TTS split that matters now is expressive vs. neutral. Neutral for information-forward turns; expressive for empathy-forward turns; steerable-expressive is the usual default.
- Pick with an A/B test on your text with your users, stratified by register, reported with CIs.
- Voice selection UX: fixed brand voice by default; curated small catalogue if users should pick; never dynamic / inferred. Do not change voice mid-conversation.
- Voice-identity consistency across turns is a design discipline: pin seeds, keep style parameters stable, don't abruptly switch expressiveness, re-test on vendor model revisions.
- The voice catalogue is a curated asset with provenance and licence metadata, not a loose config.

Next chapter: voice cloning — IVC vs. PVC, the consent flow, provenance artefacts, and the legal regimes that constrain what you can ship.
