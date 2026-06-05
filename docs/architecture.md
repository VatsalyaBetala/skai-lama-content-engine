# Architecture

## Overview

The engine is a five-layer sequential pipeline. Each layer is a Claude API call with tool access, preceded by a context-building step and followed by a Notion write and a human approval gate.

The pre-processing layer (Signal Intelligence) runs before L1 and reduces raw signal sources to structured summaries. This exists to prevent context bloat , passing 200 raw Slack messages to Claude is worse than passing a 500-word distillation of those messages.

---

## Layer structure

### Signal Intelligence

Four parallel subflows run before L1 and feed into a single context object:

- **Slack Summariser** , reads 7 internal channels, extracts signals by type (shipped feature, customer signal, pain point, commercial signal), distils with Claude Haiku
- **Notion Summariser** , queries recently updated Notion pages and PRDs, extracts product context
- **Google Drive Distiller** , fetches Google Meet transcripts from the past 7 days, distils with Claude Haiku
- **Crisp Distiller** , fetches support conversations from Crisp, extracts merchant pain points verbatim, groups by theme and frequency

Each subflow outputs a structured summary object. These are merged into a single `signalDigestContext` string before L1.

### Layer 1 , Signal Digest

**Model:** Claude Sonnet 4  
**Input:** merged signal context from all four subflows  
**Output:** structured weekly digest page in Notion (Signal Digests DB)

The L1 prompt enforces specificity rules that are non-negotiable: every signal must include a named entity, a specific number, or a direct quote. Generic signals are not permitted. This is enforced through the prompt, not through post-processing.

After writing the digest, a Haiku call writes a `## For Layer 2 , Calendar Agent` section to the bottom of the page. This section tells L2 which signals are worth building content around, which to ignore, and which product each signal belongs to. Product assignment errors (Kite signals assigned to Easy Bundles) are caught here before they propagate.

**Approval gate:** Slack notification with digest URL. PMM changes Notion status to `Approved`. Workflow polls every hour, times out after 8 hours.

### Layer 2 , Calendar Agent

**Model:** Claude Sonnet 4  
**Input:** full digest content fetched from Notion, plus L1 handoff section  
**Output:** calendar item pages in Content Calendar DB, one per proposed piece

Each calendar item page contains:
- Structured properties (title, format, ICP, funnel stage, keyword, publish date, opportunity score)
- A `Brief` section (3-5 sentences: what the piece is, who it's for, what they'll do after reading, why it ranks)
- An `Angle` section (one paragraph: the specific insight that makes this piece impossible for a competitor to replicate)
- A `Next Step` section (one action item with named owner and deadline)

The calendar agent runs inside a loop , one Notion page per calendar item, created sequentially. A merge node after the HTTP POST recombines the Notion response with the upstream payload fields (brief, angle, keyword) that the Notion API does not return.

After each page is created, a Haiku call writes a `## For Layer 3 , Research Agent` section. This tells L3 what makes the piece irreplicable, what to find in SERP and Reddit, what conditions would kill the piece, and what not to do.

**Approval gate:** Slack notification. PMM changes status of items they want to proceed to `Approved`. L3 only runs on approved items.

### Layer 3 , Research Agent

**Model:** Claude Haiku with Ahrefs MCP and web search tools  
**Input:** approved calendar item pages (properties + body blocks including L2 handoff)  
**Output:** research brief pages in Research Briefs DB

The research agent fetches the calendar item page body to extract the brief, angle, and L2 handoff context before running. This is important , the agent needs to know what angle to defend and what SERP gap to find evidence for, not just the target keyword.

Research tasks per item:
1. `keywords-explorer-overview` for target keyword and 3-5 related keywords
2. `serp-overview` for the highest-opportunity keyword
3. Web search for ICP pain points on Reddit and community forums
4. Gap analysis against SERP results

If SERP is dominated by DR 90+ pages, the agent finds a long-tail variant. If the target keyword returns no volume, the agent pivots to a related keyword with measurable demand.

Output is a JSON object. The agent is instructed to return only the JSON with no preamble. A parse node strips any markdown fences Claude occasionally wraps around JSON despite being instructed not to.

**Approval gate:** Slack notification with brief count. PMM reviews each brief and changes status to `Approved`.

### Layer 4 , Generation Agent

**Model:** Claude Sonnet 4  
**Input:** approved research brief (full page content fetched from Notion) + style guide  
**Output:** draft pages in Drafts DB

The generation agent reads the full research brief body, not just the properties. This includes SERP analysis, ICP pain points, content gap, and recommended structure. It also fetches the style guide page and applies voice principles, banned phrases, and product naming rules.

Draft rules:
- Open with a data point or specific merchant outcome , never a generic setup paragraph
- Every claim must be grounded in the research brief or Signal Digest , no invented stats
- CTA at end: outcome-first, not product-first

The agent adds a `## Self-critique` section at the bottom of every draft listing: what is working, what needs PMM attention (data sign-offs, named examples, voice calibration), and any product naming errors found and corrected.

---

## Agent handoff architecture

The handoff sections between layers are the mechanism by which context is preserved across the pipeline without requiring each agent to re-read everything. Each handoff is:

- Written by Claude Haiku (not Sonnet , it's templated extraction, not reasoning)
- Appended to the output page as Notion blocks after the main content
- Read by the next agent as part of its input context
- Not intended for human reading

The L1 handoff tells L2 which signals to prioritise. The L2 handoff tells L3 what to find and what not to do. The L3 handoff tells L4 what research confirmed, what changed, and what the piece must accomplish.

Without this architecture, each agent would only have access to the structured properties of the previous layer's output , not the reasoning behind it.

---

## Data flow

```
L1: Set Config
    runId, database IDs, Slack channel IDs, Notion page IDs
    ↓
L1: Fetch Prompt Library
    All prompts fetched at runtime from Notion
    ↓
Run: Signal Intelligence
    Parallel subflows → merged context object
    ↓
L1: Build Claude Context
    Config + prompts + signal context → userMessage
    ↓
L1: Claude API
    → L1: Parse Digest Response
    → Notion: Signal Digest Payload
    → L1: Create Signal Digest in Notion
    → L1: Build Agent Handoff Context
    → L1: Claude , Write Agent Handoff (Haiku)
    → L1: Append Handoff to Signal Digest
    → L1: Store Digest Page ID
    → Slack: Notification
    → [approval gate , poll every hour, timeout 8 hours]
    ↓
L2: Fetch Signal Digest Content
    → L2: Build Calendar Context
    → L2: Claude API , Calendar Agent
    → L2: Parse Calendar Items
    → L2: Loop Over Calendar Items
        → L2: Build Calendar Payload
        → L2: Create Calendar Entry in Notion
        → L2: Merge Payload into Response
        → L2: Build Agent Handoff Context
        → L2: Claude , Write Agent Handoff (Haiku)
        → L2: Parse Agent Handoff
        → L2: Append Handoff to Calendar Item
    → L2: Notify Slack , Calendar Ready
    → [approval gate]
    ↓
L3: Split Calendar Items
    → L3: Fetch Calendar Content (blocks , brief, angle, handoff)
    → L3: Loop Over Calendar Items
        → L3: Parse Calendar Content
        → L3: Prepare Agent Input
        → L3: Research Agent (Haiku + Ahrefs MCP + web search)
        → L3: Parse Agent Output
        → L3: Build Brief Payload
        → L3: Create Research Brief in Notion
        → L3: Update Calendar Item Status
    → L3: Summarise Brief Results
    → L3: Notify Slack , Briefs Ready
    → [approval gate]
    ↓
L3: Get Approved Briefs
    → L3: Split Out
    → L3: Fetch Brief Content
    → L3: Parse Brief Content
    → L4: Fetch Style Guide from Notion
    → L4: Build Generation Context
    → L4: Claude API , Generation Agent
    → L4: Parse Draft Response
    → L4: Create Draft in Notion
    → L4: Update Calendar Item , In Draft
    → L4: Notify Slack , Draft Ready for PMM
```

---

## Known issues and limitations

**Item lineage across HTTP boundaries.** n8n's HTTP Request node (typeVersion 4.2) does not support "Include Input in Output". When a loop iteration creates a Notion page, the response contains only the Notion page object , not the upstream fields (brief, angle, keyword) that were sent as the request body. This is solved with a `Merge Payload into Response` Code node immediately after the HTTP call that reads `$('upstream-node').item.json` to recombine the fields.

**`.first()` vs `.item` in loops.** Inside a `splitInBatches` loop, `$('node').first().json` always returns iteration 0's data regardless of which iteration is running. `$('node').item.json` returns the current iteration's data. All Code nodes inside loops use `.item`. All Code nodes that reference global config (outside the loop) use `.first()`.

**Prompt fetching.** The Prompt Library fetch reads blocks from a Notion page and matches them to prompt keys by heading text. If a heading is renamed in Notion, the corresponding prompt will not be found and the layer will fall back to a hardcoded default. Heading names in Notion must match exactly: `Layer 1`, `Layer 2`, `Layer 3`, `Layer 4`.

**Claude JSON wrapping.** Claude occasionally wraps JSON output in markdown fences (` ```json ``` `) even when explicitly instructed not to. All parse nodes strip these before `JSON.parse()`.

**Crisp coverage.** The Crisp subflow currently covers one inbox (Easy Bundles). Each additional product inbox requires a separate website token and a cloned subflow with the correct `website_id`.
