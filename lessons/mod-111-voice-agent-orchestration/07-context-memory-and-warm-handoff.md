# 07. Context, Memory, and Warm Handoff

## Why this chapter

A short demo agent fits in a single system prompt plus a few turns. A production voice agent does not. A 20-minute customer service call is 50–100 turns long, several minutes of verbatim transcript, half a dozen tool call round-trips, probably a mid-call escalation, and sometimes a handoff to a different agent. If you feed the whole transcript to the LLM every turn, your p95 TTFT climbs past your mod-109 budget, your bill climbs with it, and the quality of the model's responses eventually degrades as the context fills with low-signal chatter.

This chapter is about the three memory operations that a long voice call forces you to implement explicitly: **rolling summaries** to keep the active context small, **pinned facts** to make sure critical state never falls out of the context no matter how much the conversation drifts, and **warm-handoff packaging** so that when a call transfers — bot to bot, bot to human, bot to bot in a different product — the next agent inherits the state without re-interviewing the user.

## The three memory tiers

The architecture most production agents converge on has three distinct tiers of memory, each with a different arrival pattern and a different lifetime.

### Tier 1 — the rolling window

The last N turns, verbatim. Fast to retrieve (it's already in the session's in-memory context), accurate because it's not summarised, but bounded in size because you cap N.

- **Typical size**: last 10–20 turns, or a token cap (e.g., 2000 tokens of transcript).
- **Lifetime**: the current call.
- **What it buys you**: the model has literal access to recent dialog ("you just said...", "a minute ago we agreed...").

### Tier 2 — the rolling summary

A compressed representation of the earlier turns that have fallen out of the rolling window. Produced by a summarisation step (either a cheap LLM call or a rule-based condenser) and appended to the system prompt.

- **Typical size**: 100–300 tokens of prose.
- **Lifetime**: the current call; regenerated periodically.
- **What it buys you**: the model keeps a sense of "what happened earlier in this call" without paying the token cost of the full transcript.

### Tier 3 — pinned facts

Structured, high-value facts extracted from the conversation and injected into the system prompt directly. These never roll out and never get summarised — they are rendered into the context verbatim on every turn.

- **Typical content**: verified customer identity (name, account number), consent flags (recording consent, SMS consent), prior purchases referenced in this call, agreed-upon next steps, red-flag markers (fraud suspected, escalation requested, PCI leak observed).
- **Lifetime**: the current call, or until explicitly unpinned.
- **What it buys you**: guarantees that the model cannot "forget" that the user already verified their identity, or that they said they don't want SMS, or that fraud risk has been flagged.

The three tiers are complementary. The rolling window is for immediate dialog coherence; the rolling summary is for long-call coherence; pinned facts are for correctness on the facts that *must* survive no matter what.

## Context assembly, turn by turn

On each LLM request, assemble the context in this order:

```
system_prompt:
  base_instructions
  current_date_time
  pinned_facts (Tier 3)
  rolling_summary (Tier 2)
rolling_window:  (Tier 1)
  turn_N-k
  ...
  turn_N
```

Two properties fall out of this ordering:

- **Pinned facts beat the summary.** If the summary says "customer may not have verified identity" but the pinned fact says "customer identity verified at turn 5", the pinned fact wins. Pin, then summarise.
- **The cacheable prefix is stable.** The base instructions and pinned facts only change when a fact is added or updated; the rolling summary only changes when it is regenerated. Prompt caching (OpenAI, Anthropic) will hit on a long stable prefix and miss on a chatty tail. Put the stable things first.

Pipecat's and LiveKit's context aggregators are where you plug in the pinning / summarisation hooks — the aggregator runs after each turn, inspects what was said, and updates the pinned facts or triggers a summarisation.

## Pinning facts safely

The temptation is to pin anything the user says. Resist. Pinning is dangerous when:

- The user *lies* or *makes a mistake* and the agent writes the mistake to a permanent part of the context. Pin verified facts, not asserted ones.
- The user *changes their mind* mid-call and the stale fact never updates. Pinning means also unpinning / updating.
- The pin contains PII in violation of your data-handling policy (mod-108). PCI card numbers must never be pinned; even in-memory, they're a compliance risk.

Rules that work:

- **Pin verified or committed facts.** "Identity verified via account-number lookup" (verified), not "user's name is John" (asserted). "Agreed to callback at 3 pm on 2026-11-07" (committed), not "mentioned prefers afternoons".
- **Pin consent flags.** Recording consent, SMS consent, marketing consent. These have audit implications and must survive.
- **Pin safety / escalation flags.** If the agent has decided "this user is in distress and requires human escalation", that flag is pinned so no subsequent turn can ignore it.
- **Expire pins that time out.** A "consent given" pin might be valid for the call; an "about to be transferred" pin has a short useful life and should be removed after the transition.
- **Never pin raw card numbers or other high-sensitivity PII.** Pin a redacted form: "card ending in 1234, type Visa".

The pins are a small structured object you own in your orchestration code. They are serialised into the system prompt with a stable format the model has been trained on via the system prompt — e.g.:

```
Verified facts about this call:
- Customer: Jane Doe, account #12345 (identity verified)
- Consent: recording=granted, SMS=denied
- Prior purchases referenced: order #7890 (two devices)
- Next steps: callback scheduled for 2026-11-07 15:00 PT
```

## Rolling summaries that don't drift

The rolling summary is produced by a summariser — a cheap LLM call (or an in-house tuned model) that condenses the dropped turns into a paragraph.

Two failure modes to design around:

- **Factual drift.** The summariser invents a fact that didn't happen. The agent then references that fact later in the call. Mitigation: the summariser prompt has to say "summarise only what was explicitly stated or decided; do not infer or embellish", and the summariser's output is sanity-checked against a quick keyword-overlap heuristic (if the summary mentions names or numbers not in the transcript, reject it).
- **Pinned-fact contradiction.** The summariser mentions "customer not yet identified" when a pinned fact already says identity was verified. Mitigation: pass the current pinned facts into the summariser prompt and instruct it not to contradict them.

A reasonable cadence is to regenerate the summary every ~10 turns, or every time a tool call completes successfully (since tool calls often change the state of the conversation). Cheaper than regenerating every turn, more responsive than regenerating only once.

## Pinning extraction without a round-trip

Pin extraction can be done by a cheap second-LLM call after each user turn ("based on the last two turns, what facts should be pinned?"), or by rule-based extractors (regex for order numbers, intent classifier for consent, etc.). The right mix depends on your domain:

- For narrow domains (collections, appointment booking) rule-based extractors are highly accurate and fast.
- For open-ended domains (support bot) a small LLM extractor is more robust.

Either way, the extractor runs **off the critical path** — it does not block the next LLM response. If the extractor hasn't finished by the time the next LLM request fires, the pin lands in the context on the turn *after* that. A one-turn lag is acceptable for anything short of safety-critical flags.

Safety-critical flags (escalation, abuse detection, consent revocation) do belong on the critical path — a one-turn lag on "escalate immediately" is one turn of damage.

## Warm handoff — context packaging for the next agent

A warm handoff is any transition where the call continues but the agent changes:

- Bot → human (the usual meaning of warm handoff).
- Bot-A → Bot-B (specialist bot, product-specific bot, or just a retry after the current bot failed).
- Bot → bot in a different product altogether (crossing a managed-platform boundary, for example).

The point of "warm" versus "cold" is that the user should not have to repeat anything. That requires the outgoing agent to produce a context package and the incoming agent to consume it.

### What goes in the package

A handoff package has a predictable shape. Write it as a schema, not free text:

```json
{
  "handoff_id": "hdo_...",
  "from_agent": "support-bot-v3",
  "to_agent": "billing-specialist-bot",
  "to_role": "specialist",
  "reason": "customer requested refund; outside policy I'm allowed to approve",
  "pinned_facts": {
    "customer_id": "...",
    "identity_verified": true,
    "consent": {"recording": true, "sms": false},
    "pci_in_scope": false
  },
  "rolling_summary": "Customer called about order #7890, received damaged item on 2026-11-02, wants a refund. I verified identity and offered a replacement, which they declined.",
  "recent_turns": [
    {"role": "user", "text": "I just want my money back."},
    {"role": "assistant", "text": "I understand. Let me transfer you..."}
  ],
  "safety_flags": [],
  "transfer_metadata": {
    "channel": "sip",
    "sip_refer_target": "+1555...",
    "priority": "high"
  }
}
```

- **Pinned facts and summary** carry the state the next agent needs to continue the conversation.
- **Recent turns** (the Tier 1 window) give the next agent the dialog texture; without them, the specialist agent feels like a cold restart even if it knows all the facts.
- **Reason** is the one-sentence "why I'm handing this to you", which both the incoming human (for a bot-to-human handoff) and the incoming bot system prompt (for a bot-to-bot handoff) consume to prime the first response.
- **Safety flags** carry forward any escalation or risk markers.
- **Transfer metadata** is the plumbing — SIP REFER, channel handoff, authentication token for the next product.

### Serialise it; don't rely on vibes

Define the handoff schema (JSON Schema, Pydantic model, protobuf, whatever your stack uses) and version it. A handoff package parsed by the new agent that has a required field missing should fail loud, not quietly. The packaging quality is the handoff quality.

### Bot-to-bot handoffs on the same framework

Same-stack handoffs (within Pipecat's pipeline, LiveKit's sessions, Vapi's squads) are the easiest case — the orchestrator can pass the package in-process. The framework provides hooks for this:

- **Pipecat** — a handoff processor in the pipeline that cleans up the current sub-agent's context, loads the package into the new sub-agent's context, and continues.
- **LiveKit Agents** — a session-level API to switch the active `Agent` instance, carrying handoff data via metadata.
- **Vapi** — explicit "squad" / transfer-call primitives that carry assistant-to-assistant handoff state.

Verify the exact hook names against current docs; the patterns are stable, the APIs move.

### Bot-to-bot handoffs across frameworks / platforms

Across-stack handoffs need the package to cross a wire. The common patterns:

- **SIP REFER** for telephony — the call moves to a different endpoint; the handoff package travels out-of-band (HTTPS to the receiving system, keyed on the call ID).
- **WebRTC room transfer** — the user is moved to a new room where the new agent is waiting; the handoff package is passed via the room metadata or as a side-channel API call to the new agent's dispatch system.
- **Shared store with a pointer** — the package is written to a Redis / DB with a short-lived token; the token is passed via SIP headers or room metadata; the receiving agent fetches by token.

Prefer the shared-store-plus-token pattern for anything cross-platform. SIP headers truncate; room metadata size varies by provider; a token + store is universal.

### Bot-to-human handoffs

The human agent doesn't want to read JSON. The human agent wants a one-screen view in their agent desktop: who the customer is, verified status, what happened, what the bot tried, any safety flags. The handoff package is consumed by the **agent desktop**, which renders it as a human-readable summary + relevant structured fields.

Two specific design points:

- **Make the summary auditable.** The human agent should see "bot said X, user said Y" for the last few turns, not just the bot's summarisation. Humans distrust summaries, and rightly so.
- **Give the human a replay** of the last ~30 seconds of audio if your product allows it. A quick replay tells the human the emotional tone of the moment, which the written summary does not.

## Interaction with mod-108 and mod-109

The three memory tiers interact with earlier modules:

- **mod-108 (compliance)**: pinned facts can carry consent flags. If the user revokes consent mid-call, the pin flips immediately and the summariser/extractor stops recording new transcript. The handoff package must not leak PII that the user hasn't consented to share with the receiving agent or product.
- **mod-109 (latency)**: summarisation runs off the critical path, but on the first turn of a handed-off call the new agent's TTFT is sensitive to how big the package is — a 10KB JSON blob in the system prompt bloats the TTFT. Cache the handoff prefix.

## Summary

- Use a **three-tier memory architecture**: rolling window (verbatim recent turns), rolling summary (condensed older turns), pinned facts (structured, high-value, long-lived).
- Assemble the context in the order `base instructions → pinned facts → rolling summary → rolling window` so prompt caching has the longest stable prefix.
- Pin **verified / committed / safety** facts; do not pin asserted claims or raw sensitive PII. Pin extraction can run off the critical path for most facts, on-path for safety flags.
- The **rolling summary** is produced by a cheap summariser, regenerated every ~10 turns or on significant events, and sanity-checked for factual drift against the pinned facts.
- **Warm handoff** is context packaging as a schema — pinned facts, summary, recent turns, reason, safety flags, transfer metadata. Define the schema, version it, fail loud on missing required fields.
- Same-framework handoffs use the framework's hooks; cross-framework handoffs should use a shared store plus a token carried in SIP headers or room metadata.
- Bot-to-human handoffs feed an agent-desktop view, not raw JSON; include transcript excerpts and optionally a short audio replay.

Next chapter: putting it all together — build vs. buy end-to-end, with a defence against the mod-109 budget, the mod-108 compliance posture, and the mod-112 eval bar.
