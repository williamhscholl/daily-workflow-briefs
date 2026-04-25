# MCP Integrations Guide

Daily Workflow Briefs needs Claude Code MCPs connected to pull signals from your tools. This document lists each MCP, how to add it, and what data the plugin uses from it.

---

## Required MCPs

### Slack
- **What the plugin uses**: posting briefs to your self-DM, reading channel history, reading DM history (by passing user IDs as channel IDs), reading thread replies, scanning saved messages.
- **Setup**: Claude Code → Settings → MCP Servers → add the official Slack MCP. You'll need to OAuth into your Slack workspace.
- **Permissions needed**: read public channels, read private channels you're in, read DMs, send messages to your self-DM.

### Google Calendar
- **What the plugin uses**: today's events, attendees, Zoom links, OOO detection.
- **Setup**: Claude Code → Settings → MCP Servers → add the Google Calendar MCP. OAuth with your Google Workspace account.
- **Permissions needed**: read events on your primary calendar.

### Gmail
- **What the plugin uses**: searching threads for VIP/team activity, parsing Jira "work due" digest emails.
- **Setup**: Claude Code → Settings → MCP Servers → add the Gmail MCP. OAuth with your Google Workspace account.
- **Permissions needed**: read threads, read message bodies (where available — see known issues below).
- **Known issue**: the Gmail MCP's `get_thread` silently drops body content on Zoom's "Meeting assets for..." emails (HTML-only). The plugin works around this by going directly to Zoom MCP for meeting content.

### Zoom (only required if you pick Zoom as your meeting transcriber during setup)
- **What the plugin uses**: meeting summaries (`summary_plain_text`), meeting next-steps extracted from the summary.
- **Setup**: Claude Code → Settings → MCP Servers → add the Zoom MCP. OAuth with your Zoom account.
- **Permissions needed**: search meetings, read meeting assets (summaries).
- **Known quirk**: `search_meetings` returns a `has_summary_permission` field that looks like an access flag but is actually about sharing with others. The plugin ignores it and checks `meeting_summary.has_permission` in the asset response instead.
- **Alternative**: if you use Granola, Otter, Fireflies, Fathom, or Loom instead of Zoom for transcripts, pick that during `/briefs:setup` Step 8 — the plugin will read those tools' recap emails via your Gmail MCP instead, no Zoom MCP required.

---

## Optional MCPs

### Atlassian (Jira + Confluence)
- **Enable during `/briefs:setup`** if you use Jira or Confluence.
- **What the plugin uses**:
  - Jira: searching issues (`searchJiraIssuesUsingJql`), reading ticket details, adding comments (only — never status/assignee/priority changes).
  - Confluence: reading pages (`getConfluencePage`), updating pages (`updateConfluencePage`) for approved offers.
- **Setup**: Claude Code → Settings → MCP Servers → add the Atlassian MCP. OAuth with your Atlassian account.
- **JQL quirks** (the plugin handles these automatically):
  - Reserved words (`INT`, `OR`, `AND`, `NOT`) must be quoted.
  - `maxResults` capped at 15, explicit `fields` list required, or responses blow up to 50k+ chars.
- **Scoping**: during setup you can limit to specific project keys (e.g. `[AI, OPS, CLA]`) if your workspace has dozens of projects.

### HubSpot
- **Enable during `/briefs:setup`** if you track deals in HubSpot.
- **What the plugin uses**:
  - Read: deals you own, deal-stage changes in last 24h, notes, meetings logged.
  - Write: add read-only notes to deals.
- **Setup**: Claude Code → Settings → MCP Servers → add the HubSpot MCP. Connect your HubSpot account.
- **Hard restrictions**: the plugin NEVER changes deal stage, amount, owner, or close date. If you try to approve an offer that implies one of these, the poll aborts it with a warning. These are intentional guardrails to prevent accidental pipeline corruption.

---

---

## Additional integrations (any tool)

The plugin can pull read-only signals from tools beyond the built-ins via two paths:

### Path A — MCP (preferred when available)
If you have a Claude Code MCP connected for the tool, the plugin calls it directly. Examples:

- **Salesforce** — opportunities, contacts, account activity
- **Intercom** — conversations assigned to you, recent customer messages
- **Zendesk** — tickets you own / are CC'd on / are watching
- **Linear** — issues assigned to you with status changes
- **GitHub** — PRs awaiting your review (if your org has a GitHub MCP connected)
- **Notion** — pages you've recently edited or are mentioned in
- **Asana** — tasks assigned to you with deadline updates

### Path B — Gmail (fallback when no MCP is connected)
For tools that email you summaries, the plugin can search your Gmail using the Gmail MCP you already have. This is the fallback path when an MCP isn't available or installed.

For meeting-transcriber emails specifically, the brief skills extract "Quick recap" and "Next steps" sections from the email body — same pattern as the built-in Zoom integration. Next steps get classified by owner (you vs your team) just like Zoom.

### Meeting transcribers

The four major Zoom alternatives all have official MCPs now. **MCP is preferred** (real-time, structured data); Gmail is the fallback when an MCP isn't available.

| Tool | Official MCP | Install link | Gmail fallback |
|------|------|------|------|
| **Zoom** | ✓ (built-in) | Claude Code Settings → MCP Servers → Zoom | `from:no-reply@zoom.us subject:"Meeting assets" newer_than:1d` |
| **Granola** | ✓ | [granola.ai/blog/granola-mcp](https://www.granola.ai/blog/granola-mcp) | `from:noreply@granola.ai newer_than:1d` |
| **Otter.ai** | ✓ | [help.otter.ai → Otter MCP Server](https://help.otter.ai/hc/en-us/articles/35287607569687-Otter-MCP-Server) | `from:noreply@otter.ai newer_than:1d` |
| **Fireflies.ai** | ✓ | [docs.fireflies.ai → MCP configuration](https://docs.fireflies.ai/getting-started/mcp-configuration) | `from:noreply@fireflies.ai newer_than:1d` |
| **Fathom** | ✓ (via Composio) | [composio.dev/toolkits/fathom](https://composio.dev/toolkits/fathom) | `from:no-reply@fathom.video newer_than:1d` |
| **Loom** | — | (no public MCP) | `from:no-reply@loom.com subject:"Recap" newer_than:1d` |

If you install an MCP later, switch with `/briefs:config switch transcriber source to mcp`. To switch back to Gmail: `/briefs:config switch transcriber source to gmail`.

### How to add one

1. **For MCP-backed tools:** connect the MCP in Claude Code Settings → MCP Servers first.
2. Run `/briefs:config add <tool> as integration: <description>` (MCP path) or `/briefs:config add <tool> via gmail: <pattern>` (Gmail path). Or include them during `/briefs:setup` Step 8.
3. The brief skills iterate this list each run, call the right MCP, and surface findings as a new section in the brief.

If you say `add granola as integration` and don't have a Granola MCP, the wizard will ask whether to set up the Gmail-fallback form instead.

### Hard restrictions
- **Read-only in v1.** The plugin won't offer write actions for additional integrations regardless of source.
- If you want write actions on a specific tool, file an issue requesting first-class support — that requires designing the safe-action allowlist for that tool.

---

## Checking what's connected

Run this in Claude Code any time:

```
Which MCPs do I have connected?
```

Claude will list them. Compare against the required + optional lists above.

---

## Troubleshooting

**Brief posts "Signals skipped: gmail" at the bottom.**
The Gmail MCP errored mid-run (often auth expiry). Reconnect via Settings → MCP Servers → Gmail → re-auth. The other signals still came through; next brief will be complete.

**Zoom meeting content says "content unavailable — not invited / no access".**
You weren't in the meeting and haven't been granted access to its summary. This is correct behavior — the plugin won't hallucinate content for meetings it can't read.

**Jira responses cause context blowups.**
Shouldn't happen — the plugin caps `maxResults` and `fields`. If you see it, file an issue with the JQL that failed.

**HubSpot offer aborted with "out of allowed scope".**
You tried to approve an action that would change deal stage / owner / amount / close date. The plugin blocked it. Execute that change via the HubSpot UI directly.
