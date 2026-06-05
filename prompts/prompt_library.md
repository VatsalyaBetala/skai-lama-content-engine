# Prompt Library

This file documents the prompts used by each layer. The authoritative versions live in the Notion Prompt Library page and are fetched at runtime. This file is a reference copy.

Prompts should be updated in Notion, not here.

---

## Layer 1, Signal Digest Agent

The L1 prompt enforces strict specificity rules. The quality of the entire pipeline depends on the quality of the digest. Generic signals propagate forward.

Key constraints enforced by the prompt:
- Every signal must include a named entity, specific number, or direct quote — no generalisations
- Content opportunities must be fully formed briefs with working titles, not topics
- Every signal must cite its exact source with URL where available
- Three content opportunities exactly — not two, not four

The prompt does not constrain the model's analysis — it constrains what the model is allowed to include. Signals that cannot be named or quantified are excluded entirely.

---

## Layer 2 — Calendar Agent

The L2 prompt instructs the model to read the Signal Digest, extract opportunities, and create calendar items. The critical constraint is the Angle section.

The Angle must:
- Reference the exact signal that triggered it (named merchant, specific stat, or direct quote)
- Explain why no competitor can replicate this piece
- Not use generic claims ("first to market", "data-driven", "comprehensive")

The prompt includes company context for each product: the core insight, the ICP, the internal data advantage. This context is why L2 can produce product-specific angles rather than generic content briefs.

---

## Layer 3 — Research Agent

The L3 prompt instructs the model to use Ahrefs and web search to validate and strengthen the calendar item angle. The model has access to `keywords-explorer-overview`, `serp-overview`, and `web_search` tools.

Key constraints:
- Maximum 8 tool calls total — this prevents the agent from over-researching at the expense of structured output
- Keywords must be 2-4 words — longer phrases return empty Ahrefs results
- If SERP is dominated by DR 90+ pages, find a long-tail variant
- ICP pain points must be verbatim quotes from community research
- Output is JSON only — no preamble, no backticks

The prompt specifies the exact JSON output format. The parse node attempts to extract JSON from the agent output and falls back to field-by-field regex extraction if JSON.parse fails.

---

## Layer 4 — Generation Agent

The L4 prompt instructs the model to write a complete first draft using the research brief and style guide. The model reads both documents before writing.

Key constraints:
- Open with a data point or specific merchant outcome — never a generic setup paragraph
- Every claim must be grounded in the research brief or Signal Digest — no invented stats
- CTA at end: outcome-first, not product-first
- Follow the recommended structure from the research brief exactly
- Self-critique section required at the bottom

The self-critique section is the mechanism by which the model surfaces its own uncertainty. It lists what it was confident about, what needs PMM verification, and any voice or naming errors it caught and corrected.

---

## Layer 1 Handoff — For Calendar Agent

Written by Claude Haiku after L1. Appended to the Signal Digest page. Tells L2:
- Top 3 signals worth building content around, with named entities and internal data
- Signals to ignore this week and why
- Internal data points L2 must use (MUST USE vs NICE TO HAVE)
- Product assignment rules for this week's signals

The most important constraint in this prompt: never assign Kite signals to Easy Bundles. Product assignment errors made at L2 propagate through L3 and L4.

---

## Layer 2 Handoff — For Research Agent

Written by Claude Haiku after each calendar item is created. Appended to the calendar item page. Tells L3:
- What makes this piece irreplicable (the specific internal data)
- What the research agent must find (SERP gaps, Reddit pain points, competitor weaknesses)
- Conditions that would kill the piece
- The angle that must survive — full paragraph, cannot be replaced by research findings
- What L3 must not do

The kill conditions are the most operationally important part. If L3 finds that the kill conditions are met (e.g., a competitor has already published this angle with the same data), it should flag this in the brief rather than proceeding.

---

## Crisp Distiller

Written by Claude Haiku in the Crisp subflow. Takes raw support conversation metadata (topic, last message, merchant name, GMV, app revenue, segments) and returns structured signal output.

Key constraints:
- Name every merchant — never "a merchant"
- verbatimSignal must be the exact topic or last message text, not a paraphrase
- crispSummary must read like a teammate briefing, not a consultant summary
- contentOpportunity must be one specific actionable sentence

The distiller does not fetch full conversation message threads — it works from the conversation metadata (topic and last message) which Crisp provides at the list level. This is sufficient for signal extraction but produces weaker verbatim quotes than full transcript analysis would.
