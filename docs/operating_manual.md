# Operating Manual

This document is for whoever runs the content engine week to week. It assumes you have read the architecture document and understand the basic pipeline structure.

---

## Weekly cadence

The engine runs automatically every Monday at 9am IST via a schedule trigger. It can also be triggered manually at any time using the Manual Trigger node in n8n.

A full pipeline run, from signal collection to draft delivery, takes between 4 and 8 hours depending on approval turnaround. The pipeline does not time out waiting for approval at L1, it polls every hour and abandons after 8 hours, sending a timeout notification to Slack.

---

## What you do at each gate

### Gate 1, Signal Digest (after L1)

You receive a Slack message with a link to the Signal Digest in Notion.

Open the digest. Read it. You are checking:
- Are the signals specific? Named merchants, specific numbers, direct quotes. Generic signals ("merchants are frustrated") indicate a problem with the Signal Intelligence subflows.
- Are the three content opportunities grounded in actual signals? Each opportunity should trace directly to a named signal in the digest.
- Are products correctly assigned? A Kite signal should not generate an Easy Bundles content opportunity.

If the digest is good: change Status to `Approved`. The pipeline will pick this up in the next poll (within 1 hour) and proceed to L2.

If the digest is weak: do not approve. Investigate which signal source produced thin output. Common causes: Slack channels had low activity, Notion Summariser found nothing recently updated, Crisp had no new conversations.

You can edit the digest before approving. L2 reads the full page content, so edits you make will be reflected in the calendar.

### Gate 2, Content Calendar (after L2)

You receive a Slack message with a link to the Content Calendar.

L2 typically proposes 3-6 calendar items. For each item, open the page and read the Brief and Angle sections. You are checking:
- Is the brief specific to Skai Lama's position, or could any competitor write this?
- Is the angle grounded in internal data or a named merchant story?
- Is the keyword realistic, not too broad, not zero volume?
- Is the publish date sensible?

Change Status to `Approved` for items you want to proceed. You can approve multiple items, L3 will run research for each one.

Items you do not approve stay as `Proposed` and will not proceed. You can approve them later if you change your mind, but you will need to manually trigger L3 for them.

You can edit the Brief, Angle, or keyword on any item before approving. L3 reads these fields when building its research context.

### Gate 3, Research Briefs (after L3)

You receive a Slack message with a link to Research Briefs.

For each brief, check:
- Is the recommended keyword realistic? Look at the volume and difficulty numbers.
- Is the content gap specific? "No content on X" is specific. "Competitors lack depth" is not.
- Is the recommended angle still the one from L2, or has research changed it? If research surfaced a better angle, decide whether to keep the original or use the new one.
- Are the ICP pain points verbatim quotes? If they are paraphrases, the Reddit research was thin.

Change Status to `Approved` for briefs ready to proceed to draft generation.

### Gate 4, Draft delivery (after L4)

You receive a Slack message with the draft title, word count, and PMM notes.

The PMM notes are written by the generation agent itself, they list the top 3 things that need human attention before publish. Read these first. Common flags: data points that need verification, named merchant examples that need confirmation, CTAs that need real links.

Read the Self-critique section at the bottom of the draft. The agent will have flagged voice issues, product naming errors, and claims it was uncertain about.

The draft is a first draft. It is not publish-ready. It is a starting point that has done the structural and research work so you can focus on voice, accuracy, and editorial judgment.

---

## What good output looks like

**Signal Digest:** Every signal has a named entity or specific number. The three content opportunities each have a paragraph-length angle that references internal data no competitor can access. The footer lists every source used.

**Calendar items:** The Brief answers all four questions (what it is, who it's for, what they'll do after reading, why it ranks). The Angle is one paragraph with a specific insight, not a generic claim about being "first" or "data-driven".

**Research Brief:** The content gap is one specific thing that is missing from all SERP results. The recommended keyword has volume between 100 and 2000 and difficulty below 60 (for most pieces). ICP pain points include at least 2-3 verbatim quotes.

**Draft:** Opens with a data point or merchant outcome, not a setup paragraph. Every claim is traceable to the research brief. CTA is outcome-first ("See how merchants benchmark their bundles" not "Try Easy Bundles").

---

## What bad output looks like and why it happens

**Generic signals without named entities**
Cause: Slack channels had low signal volume that week, or the Slack Summariser prompt is not specific enough.
Fix: Edit the digest to remove generic signals. Check Slack Summariser output directly to see what it ingested.

**Calendar items that any competitor could write**
Cause: The L2 prompt is not enforcing the internal data requirement strictly enough, or the Signal Digest did not contain strong enough internal signals.
Fix: Edit the Angle before approving. Add the specific internal data point.

**Research brief with zero-volume keywords**
Cause: The L3 agent chose the exact phrase from the calendar item brief rather than finding the actual search term.
Fix: Edit the keyword in the research brief before approving. The L4 generation will use whatever is in the brief.

**Draft that opens with a generic paragraph**
Cause: Style guide is a placeholder. The generation agent defaults to generic prose when the style guide does not specify an opening pattern.
Fix: Add specific opening rules to the style guide: "Always open with a specific merchant outcome or a data point. Never open with a scene-setting paragraph."

---

## When things break

**Pipeline stalled, no Slack notification after triggering**
Check the n8n execution log. Signal Intelligence subflows are the most common failure point, a Slack API rate limit or a Crisp authentication error will silently fail.

**L1 approval timed out**
The pipeline sent a timeout Slack notification and stopped. To continue: approve the digest in Notion manually, then trigger the workflow again from the `L2: Fetch Signal Digest Content` node (use the n8n "execute from node" feature).

**L2 loop created duplicate calendar items**
A previous run created items that were not cleaned up. Delete the duplicates in Notion before re-running. The loop creates one Notion page per calendar item, if it runs twice, you get duplicates.

**L3 research brief has empty keyword data**
The Ahrefs API call failed or the keyword returned no results. The agent should have pivoted to a shorter keyword, if it did not, the brief will have zero volume. Edit the keyword in the brief to a known-volume term before approving for L4.

**L4 draft is truncated**
The Notion page creation hit the 2000-character limit on a rich text block. The draft content is written as a single paragraph block, if the draft exceeds 2000 characters (it usually does), the content is cut off. This is a known issue. Fix: split the draft into multiple paragraph blocks in the `L4: Create Draft in Notion` node.

---

## Prompt iteration

All prompts live in the Notion Prompt Library. To improve a prompt:

1. Open the Prompt Library page in Notion
2. Find the relevant section (Layer 1, Layer 2, etc.)
3. Edit the prompt text directly
4. The next pipeline run will use the updated prompt automatically

Do not edit prompts in n8n. The workflow fetches prompts at runtime, anything hardcoded in n8n will be overwritten the next time someone edits the Prompt Library.

When you change a prompt, note what you changed and why in a comment on the Notion page. Prompt quality degrades silently, you will not know a change made things worse until you see two or three weeks of output.

---

## Adding a new signal source

Each signal source is a standalone n8n subflow that returns a structured summary object. The object must have at minimum:

```json
{
  "summary": "string, distilled overview",
  "signals": "string, formatted signal list",
  "source": "string, source identifier"
}
```

Wire the subflow into `Run: Signal Intelligence` as a parallel branch. Merge its output into the context object in `L1: Build Claude Context` alongside the existing Slack, Notion, and Google Drive summaries.

The Signal Intelligence layer is intentionally designed to be extended without modifying the core pipeline.
