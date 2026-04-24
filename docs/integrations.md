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

### Zoom
- **What the plugin uses**: meeting summaries (`summary_plain_text`), meeting next-steps extracted from the summary.
- **Setup**: Claude Code → Settings → MCP Servers → add the Zoom MCP. OAuth with your Zoom account.
- **Permissions needed**: search meetings, read meeting assets (summaries).
- **Known quirk**: `search_meetings` returns a `has_summary_permission` field that looks like an access flag but is actually about sharing with others. The plugin ignores it and checks `meeting_summary.has_permission` in the asset response instead.

---

## Optional MCPs

### Atlassian (Jira + Confluence)
- **Enable during `/brief-setup`** if you use Jira or Confluence.
- **What the plugin uses**:
  - Jira: searching issues (`searchJiraIssuesUsingJql`), reading ticket details, adding comments (only — never status/assignee/priority changes).
  - Confluence: reading pages (`getConfluencePage`), updating pages (`updateConfluencePage`) for approved offers.
- **Setup**: Claude Code → Settings → MCP Servers → add the Atlassian MCP. OAuth with your Atlassian account.
- **JQL quirks** (the plugin handles these automatically):
  - Reserved words (`INT`, `OR`, `AND`, `NOT`) must be quoted.
  - `maxResults` capped at 15, explicit `fields` list required, or responses blow up to 50k+ chars.
- **Scoping**: during setup you can limit to specific project keys (e.g. `[AI, WAT, INT]`) if your workspace has dozens of projects.

### HubSpot
- **Enable during `/brief-setup`** if you track deals in HubSpot.
- **What the plugin uses**:
  - Read: deals you own, deal-stage changes in last 24h, notes, meetings logged.
  - Write: add read-only notes to deals.
- **Setup**: Claude Code → Settings → MCP Servers → add the HubSpot MCP. Connect your HubSpot account.
- **Hard restrictions**: the plugin NEVER changes deal stage, amount, owner, or close date. If you try to approve an offer that implies one of these, the poll aborts it with a warning. These are intentional guardrails to prevent accidental pipeline corruption.

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
