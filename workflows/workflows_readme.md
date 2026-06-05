# Workflows

Export each workflow from n8n (⋯ menu → Download) and place the JSON files here.

| Filename | Workflow name in n8n |
|---|---|
| `main_content_engine.json` | Main: Content Engine |
| `logging_nodes.json` | Content Engine - Logging Nodes |
| `subflow_slack.json` | Subflow: Slack |
| `subflow_notion.json` | Subflow: Notion |
| `subflow_google_drive.json` | Subflow: Google Drive |
| `subflow_crisp.json` | Subflow: Crisp |
| `subflow_github_pr_signals.json` | Subflow: GitHub PR Signals |

JSON files are not committed to this repository to avoid leaking credential IDs and internal Notion page IDs. Export from your n8n instance directly.

If you are setting this up from scratch, import in the order listed - subflows first, main workflow last.
