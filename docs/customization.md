# Customization

Once `/briefs:setup` has generated your `~/.claude/daily-workflow-briefs/config.md`, you can edit it directly to tune the plugin's behavior. This document covers the non-obvious knobs.

---

## Quick edits via Claude Code (no file editing)

You can ask Claude in chat — natural language works:
- "Add #escalations to my brief watch channels"
- "Remove Sid from my team list"
- "Change my poll interval to 30 min"
- "Turn off Jira in my brief config"
- "Add Salesforce as an integration"
- "Switch work-offer preview to full"

Claude will read the config, apply the edit, and confirm. Slash command equivalent: `/briefs:config <your request>`.

---

## The config file

Lives at `~/.claude/daily-workflow-briefs/config.md`. Sections:

### Identity
```yaml
display_names: [Sam, Samuel, Sam Lee]
slack_user_id: U09CAAGER97
slack_self_dm: D09CAAHDF3K
email: sam@company.com
timezone: America/Los_Angeles
```

`display_names` — include every variant the Zoom recap might use for your name. If Zoom transcribes you as "Samuel" but your team calls you "Sam", list both. Missing variants cause next-steps to miss your assignments.

### Watch channels
```yaml
watch_channels:
  - C03FEGLDB6J: #engineering
  - C04FL97MMBJ: #leadership
```

Use channel IDs (not names) — Slack's `search_channels` is unreliable on the MCP so the plugin works directly with IDs.

### Team
```yaml
team:
  - Sid Patel: U06D1LW0DV3
  - Riya Chen: U098F0GGV1D
```

Each team entry gets their DM history scanned every poll. More entries = more MCP calls per run = more tokens. Prune aggressively — only include the people whose DMs you genuinely need scanned.

### VIPs
```yaml
vips:
  - CEO Alex: alex@company.com
  - Head of Eng Jordan: jordan@company.com
```

VIPs drive Gmail priority. Threads from VIPs always surface; threads from others only surface if action-required.

### Built-in integrations
```yaml
integrations:
  calendar: on
  gmail: on
  zoom: on
  jira: projects:[AI,OPS,"INT",CLA]
  hubspot: off
```

Jira variants:
- `jira: on` — all projects your Atlassian account can see
- `jira: off` — skip Jira entirely
- `jira: projects:[KEY1,KEY2,...]` — limit to specific project keys. Quote reserved words (`"INT"`, `"OR"`, etc.).

HubSpot: `on` or `off`. Always read-mostly — notes only.

### Additional integrations
```yaml
additional_integrations:
  - intercom: conversations assigned to me last 12h
  - salesforce: open opportunities I own with stage changes today
  - zendesk: tickets where I'm CC'd
```

Each entry is `<mcp_name>: <what-to-check description>`. The brief skills iterate this list, call the corresponding MCP, and surface findings as a new section in the brief. **Read-only in v1** — no work offers from additional integrations.

For each name to work, the corresponding Claude Code MCP must be connected (see [docs/integrations.md](integrations.md)).

### Schedule
```yaml
morning_time: 07:30
eod_time: 15:30
poll_interval_minutes: 60
run_on_weekends: false
```

Custom poll intervals: set `poll_interval_minutes` to any value (e.g. 45). The scheduled-tasks cron will be approximated. For true custom schedules, edit the scheduled task directly via the scheduled-tasks MCP.

### Tasks
```yaml
tasks_file: ~/.claude/daily-workflow-briefs/tasks.md
```

To disable task management entirely: `tasks_file: none`. The overdue-tasks section will be omitted from briefs; the poll won't edit any file.

### Behavior
```yaml
work_offer_preview: summary    # or 'full'
eod_lookback_hours: 12
```

Knobs:
- `work_offer_preview` — `summary` (one-line offers, type `show 1` to expand) or `full` (full preview text inline)
- `eod_lookback_hours` — how far back the EOD brief looks (default 12)
- `morning_lookback_hours_weekday` — default 12 (Tue–Fri)
- `morning_lookback_hours_monday` — default 72 (captures weekend)

---

## Disabling a specific signal type

In `config.md`, add a `disabled_signals` section:

```yaml
disabled_signals:
  - saved_messages
  - hubspot_deals
```

Known signal keys: `calendar`, `zoom_meetings`, `gmail_vips`, `gmail_team`, `gmail_jira_digests`, `gmail_jira_notifications`, `slack_watch_channels`, `slack_dms`, `saved_messages`, `jira_activity`, `jira_pm_review`, `hubspot_deals`, `tasks_overdue`, `additional_integrations`.

---

## Changing the work-offer allowlist

The default allowlist is intentionally conservative. If you want to enable additional actions (e.g. Jira status changes), you'd need to edit the installed `skills/brief-poll/SKILL.md` and add a new `action_type` branch.

**Don't do this casually.** The whole point of the allowlist is preventing expensive mistakes. If you fork the plugin and broaden the allowlist, you own the risk.

---

## Changing the cron schedules

The scheduled tasks registered during setup can be edited via the scheduled-tasks MCP:

```
List my scheduled tasks
```

Find the three named `morning-brief`, `brief-poll`, `eod-brief`. Update via:

```
Update scheduled task "brief-poll" to run every 45 minutes weekdays 8am–4pm
```

Standard cron syntax supported. Or use `/briefs:config change my poll interval to 45 minutes` for a guided update.

---

## Multi-workspace users

If you work across multiple Slack workspaces or multiple Jira tenants: v1 supports only one of each. Pick the primary. If there's demand, multi-tenant config is a reasonable v2 extension.

---

## Troubleshooting

**The brief didn't fire at the expected time.**
Check the scheduled task is registered and enabled: `List my scheduled tasks`. If missing, re-run `/briefs:setup`. If disabled, enable via the scheduled-tasks MCP.

**The poll isn't executing my `apply 1` replies.**
Check:
1. Are you replying IN THE THREAD (not as a new message)? The poll reads thread replies only.
2. Is the offer still active? Offers expire after 18 hours — check `~/.claude/daily-workflow-briefs/.offers.jsonl` for the offer's `status`.
3. Did the poll run between your reply and now? Check `~/.claude/daily-workflow-briefs/.poll-log.jsonl` for the latest entry. If no poll has run, trigger manually: `/briefs:run poll`.

**Offers are numbered wrong.**
Each brief or poll post starts numbering at 1 scoped to that post. If you reply `apply 1` in a thread with multiple poll posts, the plugin disambiguates by most-recent offer-emission timestamp. To be safe, include the offer text: `apply 1: Add comment to CLA-718`.
