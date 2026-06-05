# Setup

## Prerequisites

- n8n instance (self-hosted or cloud). Tested on n8n cloud.
- Notion workspace with the Content Engine databases (see below)
- Anthropic API key
- Slack bot token with read access to relevant channels
- Ahrefs API key
- Google Drive OAuth2 credentials
- Crisp account with website token per inbox

---

## Step 1, Notion workspace

The engine requires six databases. If you are setting up from scratch, create each database with the properties listed. If you are working from the Skai Lama workspace, the IDs are already set in `L1: Set Config`.

### Signal Digests DB
Properties: `Digest Title` (title), `Status` (select: Ready for Review / Approved), `Week Of` (date), `Content Opportunities` (number), `Key Signals` (text), `Products Covered` (multi-select)

### Content Calendar DB
Properties: `Title` (title), `Status` (select: Proposed / Approved / In Research / In Draft / Published), `Format` (select), `ICP` (select), `Funnel Stage` (select), `Business Goal` (select), `Target Keyword` (text), `Signal Source` (text), `Opportunity Score` (number), `Publish Date` (date), `Product` (multi-select), `Research Brief` (relation → Research Briefs DB)

### Research Briefs DB
Properties: `Brief Title` (title), `Status` (select: Ready for review / Approved), `Target Keyword` (text), `ICP` (select), `Product` (multi-select), `Signal Source` (text), `Recommended Angle` (text), `Content Gap` (text), `Monthly Search Volume` (number), `Keyword Difficulty` (number), `Calendar Item` (relation → Content Calendar DB)

### Drafts DB
Properties: `Title` (title), `Status` (select: PMM Review / Approved / Published), `Format` (select), `Product` (multi-select), `Word Count` (number), `Publish Date` (date), `PMM Notes` (text), `Research Brief` (relation), `Calendar Item` (relation)

### Prompt Library
A standard Notion page (not a database). Each prompt is stored under a `## Heading` with the prompt text in a code block or paragraph immediately below. Required headings: `Layer 1`, `Layer 2`, `Layer 3`, `Layer 4`, `Crisp Distiller`.

### Engine Run Log DB
Properties: `Run Name` (title), `Run ID` (text), `Status` (select: Running / Success / Partial / Failed), `Triggered By` (select: Scheduled / Manual), `Layers Completed` (multi-select), `Run Date` (date), `Briefs Created` (number), `Drafts Created` (number), `Calendar Entries Created` (number), `Signals Collected` (number), `Error Message` (text), `Error Node` (text), `Notes` (text)

---

## Step 2, n8n credentials

Create the following credentials in n8n Settings → Credentials:

**Notion API**, predefined credential type `notionApi`
- Integration token from Notion Settings → Integrations
- Share all six databases with the integration

**Anthropic API**, Header Auth
- Header name: `x-api-key`
- Header value: your Anthropic API key
- Name it exactly `Anthropic API` or update the credential reference in all HTTP nodes

**Slack API**, Header Auth
- Header name: `Authorization`
- Header value: `Bearer xoxb-your-bot-token`
- Bot must have `channels:history`, `channels:read` scopes

**Ahrefs API**, Header Auth
- Header name: `Authorization`
- Header value: `Bearer your-ahrefs-key`
- Used via MCP client node in L3

**Google Drive**, OAuth2 (built-in n8n credential type)
- Authorise with the Google account that owns the meeting notes Drive folder

**Crisp API**, Header Auth
- Header name: `Authorization`
- Header value: `Basic base64(token_identifier:token_key)`
- Generate token from Crisp Settings → Workspace Settings → Advanced configuration
- One credential per inbox (each Crisp workspace generates its own token)

---

## Step 3, Update L1: Set Config

Open `Main: Content Engine` → find `L1: Set Config` → update these values:

```
promptLibraryPageId    ID of your Prompt Library page (no dashes)
signalDigestsDb        ID of your Signal Digests database
calendarDb             ID of your Content Calendar database
briefsDb               ID of your Research Briefs database
draftsDb               ID of your Drafts database
styleGuidePageId       ID of your Style Guide page
slackMarketingChannel  Slack channel ID for approval notifications
```

Channel IDs for additional Slack signal sources are also set here. To find a Slack channel ID: right-click the channel → View channel details → copy the ID from the bottom of the modal.

---

## Step 4, Import workflows

In n8n, go to Workflows → Import → upload each JSON file in this order:

1. `subflow_slack.json`
2. `subflow_notion.json`
3. `subflow_google_drive.json`
4. `subflow_crisp.json`
5. `subflow_github_pr_signals.json`
6. `logging_nodes.json`
7. `main_content_engine.json`, import last, as it references the subflow IDs

After importing the main workflow, open `Run: Signal Intelligence` (the Execute Workflow node) and verify the workflow ID points to your imported Signal Intelligence workflow.

---

## Step 5, Verify the Prompt Library

Open your Prompt Library Notion page. Confirm it contains sections with these exact heading names:
- `Layer 1`
- `Layer 2`
- `Layer 3`
- `Layer 4`
- `Crisp Distiller`

Each section must have the prompt text in either a code block or a paragraph of 200+ characters immediately below the heading. The `L1: Extract All Prompts` node matches by heading text, if the heading name differs, the prompt will not be found.

---

## Step 6, Test run

Trigger the workflow manually (Manual Trigger node). Watch the execution:

1. Signal Intelligence subflows should complete and return a `signalDigestContext` string
2. L1 Claude API call should return a structured digest with three content opportunities
3. A Notion page should appear in Signal Digests DB
4. A Slack message should appear in your marketing channel

If step 1 returns empty context, check Slack API permissions and verify the channel IDs in config.

If step 3 fails with a Notion validation error, check that all database property names in `Notion: Signal Digest Payload` match your actual database schema exactly, Notion property names are case-sensitive.

---

## Troubleshooting

**`invalid_session` from Crisp API**, the token key was not saved correctly. Regenerate the website token in Crisp and update the credential. Crisp only shows the token key once at generation time.

**Prompts not found**, `L1: Extract All Prompts` logs which prompt keys were found. Check the n8n execution log for this node. Heading names in your Prompt Library must match exactly.

**Loop producing duplicate entries**, a node inside the loop is using `.first()` instead of `.item`. All Code nodes inside `splitInBatches` loops must use `$('node-name').item.json` for iteration-specific data.

**Notion PATCH returning 404**, the `calendarPageId` being used for the handoff append is incorrect. Check `L2: Merge Payload into Response` output to confirm the `id` field is a valid Notion page UUID.

**Claude returning JSON wrapped in backticks**, the parse node should strip these. If it does not, add `.replace(/\`\`\`json|\`\`\`/g, '').trim()` before `JSON.parse()`.
