# 01. TCPA, the FCC's AI-Voice Ruling, and Outbound-Call Consent

## Why this chapter

Anytime your agent *places* a call — rings a phone, leaves a voicemail, auto-dials a lead list — a 1991 statute and a 2024 FCC ruling land on top of your architecture. The Telephone Consumer Protection Act (TCPA, [47 U.S.C. § 227](https://www.law.cornell.edu/uscode/text/47/227)) is the single most-litigated consumer-protection statute in the United States; private plaintiffs can recover \$500 per violation and \$1,500 per *willful* violation, with no cap. A single poorly-gated outbound campaign has produced nine-figure class settlements more than once.

On February 8, 2024 the FCC issued a Declaratory Ruling ([FCC 24-17](https://docs.fcc.gov/public/attachments/FCC-24-17A1.pdf)) that classifies calls using AI-generated voices as "artificial" under the TCPA. The practical effect: every outbound voice-agent call now requires the *higher* tier of consent, even when it would have been an ordinary human-driven call a year earlier.

This chapter is enough TCPA/FCC working knowledge to design an outbound-call consent flow that counsel can sign. The chapter does not cover marketing-SMS (10DLC, A2P), the Do-Not-Call list for SMS, or the state TCPA mini-statutes (Florida FTSA, Washington CEMA, Oklahoma TCPA) beyond pointing at them.

## The statute in one page

TCPA's two operative prohibitions for voice (summarised from 47 U.S.C. § 227(b)):

1. **Automated-dialler + artificial / prerecorded voice to a cell phone** → requires *prior express consent* for non-marketing, *prior express written consent* for marketing. "Automated telephone dialing system" (ATDS) is the statutory term; the Supreme Court narrowed it in *Facebook v. Duguid*, 592 U.S. 395 (2021) to systems that use a random or sequential number generator. Most modern voice-agent dialers no longer meet that ATDS definition — but the artificial-voice prong catches them regardless.
2. **Artificial or prerecorded voice to a residential landline** → requires *prior express written consent* for marketing; narrow exemptions for non-marketing informational calls.

Add two near-neighbors from the FCC's TCPA rules (47 CFR 64.1200):

3. **National Do Not Call Registry** (telemarketing only) — honor within 31 days of registration.
4. **Internal (company-specific) DNC list** — every caller must maintain one and honor requests indefinitely.

Then the FCC's February 2024 ruling layers on:

5. **AI-generated voices count as "artificial"** under § 227(b). Every outbound call that uses a TTS-generated voice is in scope; the mere fact that the words were composed in real-time by an LLM does not make the voice "live."

The ruling did *not* create a new prohibition. It clarified that an existing prohibition applies to a technology that didn't exist in 1991.

## "Prior express written consent" — what actually counts

Prior express written consent is defined at 47 CFR 64.1200(f)(9). The four requirements (paraphrased from the regulation):

- A written agreement, signed by the consumer. Electronic signatures compliant with the E-SIGN Act are valid.
- Clear and conspicuous disclosure that the consumer **authorises the seller to deliver, or cause to be delivered, advertisements or telemarketing messages** using an automatic telephone dialing system or an artificial or prerecorded voice.
- Disclosure that the consumer **is not required to sign the agreement** (directly or indirectly), or agree to enter it as a condition of purchasing any property, goods, or services.
- Identification of the specific phone number the consumer is authorising to be called.

The engineer-shaped takeaway: a buried "I agree to the Terms" checkbox does not meet this bar. A compliant flow is two deliberate affirmances — one for the service, one for the automated-call permission — with the authorising phone number captured and stored *immutably alongside* the consent timestamp and the exact disclosure text as shown.

Store at least:

```json
{
  "consumer_id": "...",
  "phone_number": "+15551234567",
  "consent_version": "2026-01-12",
  "consent_text_hash": "sha256:...",
  "consent_text_rendered": "By checking this box ...",
  "granted_at": "2026-01-15T14:33:21Z",
  "method": "web_form",
  "ip_address": "198.51.100.5",
  "user_agent": "...",
  "revoked_at": null
}
```

The hash and the rendered text matter because the FCC's 2024 "1:1 consent" rule (currently stayed; see note below) and the state FTSA-family statutes allow the plaintiff to demand the *exact text* the consumer saw.

> **Status note on 1:1 consent.** The FCC's *Closing the Lead Generator Loophole* rule (89 FR 5098, published January 25, 2024) would have required consent to be given to *one seller at a time*, not a broad lead-generator "network of partners." The rule was vacated by the Eleventh Circuit in *Insurance Marketing Coalition v. FCC* in January 2025, so the pre-2024 broader-consent-per-opt-in regime remains in effect at federal level as of the time this module is authored. Verify current status at [FCC Docket 02-278](https://www.fcc.gov/ecfs/search/search-filings/results?limit=25&proceedings_name=02-278) before building new flows. <!-- needs-research: confirm docket status and any successor rulemaking at the time of use -->

## Revocation

Consumers may revoke consent *in any reasonable manner* (47 CFR 64.1200(a)(10), finalized in [FCC 24-24](https://docs.fcc.gov/public/attachments/FCC-24-24A1.pdf), the February 2024 revocation rule). The engineer implication:

- Honor "stop," "quit," "end," "revoke," "opt out," "unsubscribe" — and anything a reasonable person would understand — spoken *in the call itself*. This means the agent's live transcript must be scanned for revocation phrases on every user turn, and a trigger must flip the consent record to `revoked_at`.
- Honor revocation **across channels**. If the consumer texts STOP to an unrelated SMS number, that revokes voice consent unless the consent was explicitly scoped. Store consents keyed to *phone number*, not *channel*.
- Honor revocation within **10 business days**. The FCC tightened this to a hard deadline in the 2024 revocation order.

## Automated dialler safeguards

Even outside the strict ATDS definition, the FCC's telemarketing rules (47 CFR 64.1200) impose:

- **Abandonment rate ≤ 3%** per 30-day campaign (64.1200(a)(7)(i)). An "abandoned" call is one where no live agent (or compliant artificial voice) is available within 2 seconds of the called party's greeting.
- **Monitoring and recording** of abandonment rate; per-campaign measurement.
- **Identification message** when an answering machine is reached, with name + callback number.
- **Pre-recorded message requirement**: automated calls must identify the caller's business name at the start and include a callback number during or after the message (47 CFR 64.1200(b)(1)-(2)).

For a voice agent, "abandonment rate" is usually zero by construction — the agent *is* the live respondent. But if your architecture places the call before the model is ready to talk (TTS warm-up, STT connection, LLM session), you can drift above 3% silently. Instrument the time-to-first-utterance.

## Caller ID — STIR/SHAKEN and SIP identity

The FCC's STIR/SHAKEN rules (47 CFR 64.6300 et seq.) require voice service providers to authenticate caller-ID information for IP-originated calls. Attestation levels:

- **A (Full attestation)** — provider authenticated the caller and knows the caller is entitled to use the calling number.
- **B (Partial attestation)** — provider authenticated the caller but cannot verify the right to use the number.
- **C (Gateway attestation)** — provider is only a transit point; cannot authenticate.

Your carrier/CPaaS passes a `verstat` parameter back; a good provider will refuse to originate your traffic at A-attestation unless you've registered your calling numbers with them. This matters because downstream carriers may block or label C-attestation calls as "spam likely."

The FCC also requires caller ID to display the **calling party's name or number accurately** (47 CFR 64.1601). Spoofing is permitted only when the displayed number is one the caller is authorised to use. A voice agent dialling on behalf of an enterprise must show either the enterprise's number or a number the enterprise owns, not the CPaaS's numeric pool at random.

## Reassigned Numbers Database (RND)

Phone numbers in the US are recycled constantly; a plaintiff who received a call intended for the previous holder of that number has a TCPA claim unless you checked the FCC's Reassigned Numbers Database. The RND is operated by Somos under FCC contract; the API is at [reassigned.us](https://www.reassigned.us/). The safe harbour at 47 CFR 64.1200(a)(4)(iv): querying the RND within the 15 days preceding the call and receiving a "no reassignment" response insulates the caller from liability if the number was in fact reassigned.

Operational pattern:

1. At the moment consent is captured, record `consent_valid_from`.
2. Before each outbound call, call the RND API with the number and `consent_valid_from`. If the RND reports a reassignment date *after* `consent_valid_from`, do not place the call; suppress the number.
3. Cache the response by `(number, consent_valid_from)` for up to 15 days per safe-harbour rules.

The query is \$0.0002 per number at wholesale as of 2024; the cost of one TCPA class settlement pays for roughly 10 billion queries. <!-- needs-research: confirm current per-query fee at the time of authoring the agent -->

## Call labelling

Separately from STIR/SHAKEN, the major US carriers run their own analytics layers that tag calls as "Spam Likely," "Scam," or "Telemarketer." The three big labellers:

- **First Orion** — used by T-Mobile, US Cellular, CSPire.
- **Hiya** — used by AT&T, Samsung Smart Call.
- **TNS (Transaction Network Services)** — used by Verizon.

A voice agent that generates outbound calls at any volume will get labelled unless you register the numbers and the campaign. Each labeller offers a registration portal (free for "Branded Caller ID" basics, paid tiers for richer display). You will also want to register through the **Industry Traceback Group** (ITG) framework if you place high volumes, and check your numbers through [free-caller-registry.com](https://www.freecallerregistry.com/) — an industry portal that fans out to the labellers.

Labelling is a *reputation signal*, not a legal one. But if your branded-caller-ID effort fails and your number shows as "Scam Likely" in the called party's phone, answer rates collapse and the deal with your buyer of the product fails before the TCPA argument ever matters.

## Putting it together — an outbound-agent flow

Design the following gates in the dialler, in order:

1. **Consent gate.** Fetch the consent record. If no record, no call. If `revoked_at` is set, no call. If the consent was captured for a different *channel* or a different *purpose* than this campaign, no call.
2. **DNC gate.** Join the number against: the National DNC Registry (Federal Trade Commission's [Do Not Call Registry](https://www.donotcall.gov/); the API is for telemarketers who have paid the Subscription Account), your internal DNC list, and any state-specific lists that apply to the campaign's targeted states.
3. **RND gate.** Call the Reassigned Numbers Database. Suppress if reassigned.
4. **Hours gate.** 47 CFR 64.1200(c) restricts telemarketing calls to 8 a.m. – 9 p.m. in the *called party's local time*. Infer local time from area code (reasonable per the regulation) or from the consumer's stored address.
5. **Attestation + label gate.** Confirm carrier originates at A-attestation; confirm the number is not currently labelled "Scam Likely" (optional, but cheap to check).
6. **Place the call.** Open the SIP leg. As soon as the called party picks up, the agent must speak (within 2 seconds, see abandonment rule) and must identify: (a) the business name on whose behalf the call is placed, (b) the fact that this is an AI-generated voice if that is material (see Chapter 3 for the EU-side Article 50 disclosure obligation; US TCPA does not currently mandate AI-disclosure, but several state bills introduced in 2024-2025 do — Utah SB 149, California AB 2013 adjacent).
7. **Live revocation monitor.** Every user turn's transcript passes through a revocation classifier. On hit, mark consent `revoked_at`, confirm revocation in the agent's reply ("Understood, I'll remove you from our list"), and end the call.

This is a lot of gates. Nearly all of them exist today in off-the-shelf outbound-dialler SaaS (Five9, Genesys, Vonage, Twilio Flex). A ground-up voice agent typically wraps a provider that offers these as features; verify each one is on, don't assume.

## What this chapter did not cover

- State TCPA mini-statutes — Florida Telephone Solicitation Act (§ 501.059), Washington Commercial Electronic Mail Act, Oklahoma TCPA, New York TCPA — several of which are significantly stricter than the federal rule. If you dial into those states, add a state-specific gate.
- Marketing-SMS (10DLC, Toll-Free Verification, A2P registration). The Campaign Registry governs that; it's a separate ecosystem.
- International outbound dialling — UK PECR and Ofcom rules are in Chapter 4; dialling into Canada invokes CRTC's unsolicited-telecommunications rules, which this module does not cover.
- The AI-disclosure question. There is no federal TCPA disclosure requirement that the voice is AI *per se* — that obligation lives in the EU AI Act Article 50 (Chapter 3), and in a handful of state bills still landing. <!-- needs-research: track state AI-disclosure bills before authoring a US-only call-opening -->

## Summary

- TCPA + the FCC's February 2024 ruling: every outbound voice-agent call uses an "artificial voice" and requires the higher consent tier.
- Prior express written consent is a four-element contract (signed, clear disclosure, not required to buy, specific number). Store the exact rendered text and a hash.
- Revocation can happen on any channel in any reasonable manner; honor within 10 business days; scan the live transcript for revocation phrases.
- Six-gate dialler: consent → DNC → RND → hours → attestation/label → place.
- Abandonment ≤ 3%, business ID at start, callback number, STIR/SHAKEN A-attestation, call-labeller registration.

Next chapter: once the call is up, the *recording* of it is a separate regulated artefact — state by state, with Illinois BIPA adding voice-biometric obligations on top.
