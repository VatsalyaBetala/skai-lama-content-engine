# Skai Lama Content Engine

An AI-orchestrated content marketing pipeline built on Claude and n8n. Designed to replace the weekly manual process of signal gathering, content planning, keyword research, and first-draft generation with a sequential multi-agent system that preserves human judgment at every gate.

This is not a content automation tool. It is a system that does the mechanical work so that the human decisions — what to write, what angle to take, what to kill — are made with better information and less friction.

---

## What it does

Every Monday, the engine:

1. Pulls signals from Slack channels, Notion pages, Google Drive meeting notes, and Crisp support conversations
2. Synthesises those signals into a weekly digest with named entities, specific stats, and content opportunities
3. Proposes a content calendar with briefs, angles, and publish dates
4. Runs keyword research and SERP analysis for each approved calendar item
5. Generates a first draft using the research brief and a style guide

At each layer, a human approves or rejects before the next layer runs. The pipeline does not proceed without explicit approval. This is intentional — the system is opinionated about where AI judgment ends and human judgment begins.

---

## Architecture

```
Signal Intelligence (parallel subflows)
    Slack · Notion · Google Drive · Crisp · GitHub
            ↓
Layer 1 — Signal Digest Agent
    Claude Sonnet 4 synthesises signals into a structured digest
    Human approval gate (Notion status → Slack notification)
            ↓
Layer 2 — Calendar Agent
    Claude Sonnet 4 proposes calendar items with briefs and angles
    Human approval gate
            ↓
Layer 3 — Research Agent
    Claude Haiku + Ahrefs MCP + web search
    Keyword research, SERP analysis, ICP pain points, content gap
    Human approval gate
            ↓
Layer 4 — Generation Agent
    Claude Sonnet 4 writes first draft against style guide
    Self-critique pass included
    Delivered to PMM for review
```

Each layer writes a `## For [Next Layer]` handoff section to its Notion output page. This section is machine-readable creative direction — it tells the next agent what to preserve, what to find, and what not to do. It is not for humans.

---

## Why it is built this way

**Sequential, not parallel.** Each layer depends on the output of the previous one. Running them in parallel would mean L3 research happens without knowing which calendar items were approved, and L4 generation happens without research. The pipeline is deliberately linear.

**Approval gates are load-bearing.** The system is only as good as the judgment applied at each gate. A PMM who rubber-stamps everything will get mediocre output. A PMM who reads the brief, adjusts the angle, and rejects weak calendar items will get genuinely useful drafts. The gates are not checkpoints — they are the mechanism by which human context enters the system.

**Prompts are in Notion, not in n8n.** Every agent prompt is fetched at runtime from a Notion Prompt Library. This means prompt iteration does not require touching the workflow. The person who improves the prompts does not need to understand n8n.

**Haiku for handoffs, Sonnet for generation.** The agent handoff sections between layers are written by Claude Haiku. They are structured, templated, and do not require reasoning — just extraction and formatting. Sonnet is reserved for the tasks that require synthesis: the digest, the calendar, the draft.

**The style guide is the highest-leverage input.** The system will produce generic output until the style guide is real. A placeholder style guide produces placeholder prose. This is the most important thing to build after the pipeline is stable.

---

## Notion workspace

The engine is backed by six Notion databases:

| Database | Purpose |
|---|---|
| Signal Digests | L1 output — weekly digest pages |
| Content Calendar | L2 output — proposed and approved items |
| Research Briefs | L3 output — keyword data, SERP analysis, content gap |
| Drafts | L4 output — first drafts awaiting PMM review |
| Prompt Library | Runtime prompt storage for all agents |
| Engine Run Log | Execution history with status, layer completion, error messages |

---

## Credentials required

| Service | Used for |
|---|---|
| Notion API | Reading and writing all database pages |
| Anthropic API | All Claude API calls (Sonnet 4 and Haiku) |
| Slack API | Approval notifications and signal ingestion |
| Ahrefs API | Keyword research and SERP analysis via MCP |
| Google Drive OAuth2 | Meeting notes ingestion |
| Crisp API | Support conversation signal extraction |

---

## Workflows

| File | Description |
|---|---|
| `main_content_engine.json` | Core pipeline — L1 through L4 with approval gates |
| `logging_nodes.json` | Parallel logging branches writing to Engine Run Log |
| `subflow_slack.json` | Slack signal ingestion and summarisation |
| `subflow_notion.json` | Notion signal extraction from recently updated pages |
| `subflow_google_drive.json` | Meeting notes distillation from Google Drive |
| `subflow_crisp.json` | Support conversation signal extraction (Easy Bundles inbox) |
| `subflow_github_pr_signals.json` | GitHub PR signal collector |

---

## Setup

See [docs/setup.md](docs/setup.md) for step-by-step instructions.

---

## Operating the engine

See [docs/operating_manual.md](docs/operating_manual.md) for how to run the pipeline, what the approval gates require, what bad output looks like, and what to do when things break.

---

## What is not built yet

- Style guide (placeholder exists, needs content)
- Mantle and Mixpanel signal subflows
- Multi-inbox Crisp coverage (currently Easy Bundles only)
- RAG layer for internal data injection
- Automated publishing to CMS
- Revenue attribution from published content
