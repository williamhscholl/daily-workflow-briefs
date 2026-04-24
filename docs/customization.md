# Customization

Once `/brief-setup` has generated your `~/.claude/daily-workflow-briefs/config.md`, you can edit it directly to tune the plugin's behavior. This document covers the non-obvious knobs.

---

## Quick edits via Claude Code (no file editing)

You can ask Claude in chat:
- "Add #product-escalations to my brief watch channels"
- "Remove Hugo from my team list"
- "Change my poll interval to 30 min"
- "Turn off Jira in my brief config"

Claude will read the config, apply the edit, and confirm. For anything more structural, edit the file directly.

---

## The config file

Lives at `~/.claude/daily-workflow-briefs/config.md`. Sections:

### Identity
```yaml
display_names: [Will, William, Will Scholl, William Scholl]
slack_user_id: U09CAAGER97
slack_self_dm: D09CAAHDF3K
email: will@company.com
timezone: America/Los_Angeles
```

`display_names` — include every variant the Zoom recap might use for your name. If Zoom transcribes you as "William" but your team calls you "Will", list both. Missing variants cause next-steps to miss your assignments.

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
  - Hugo Salomon: U06D1LW0DV3
  - Raphael Carvalho: U098F0GGV1D
```

Each team member gets their DM history scanned every poll. More team members = more MCP calls per run = more tokens. Prune aggressively — only include direct reports and key collaborators.

### VIPs
```yaml
vips:
  - CEO Anoosh: anoosh@company.com
  - Head of Eng Sohrab: sohrab@company.com
```

VIPs drive Gmail priority. Threads from VIPs always surface; threads from others only surface if action-required.

### Integrations
```yaml
integrations:
  calendar: on
  gmail: on
  zoom: on
  jira: projects:[AI,WAT,"INT",PLA,TEL,PUL]
  hubspot: off
```

Jira variants:
- `jira: on` — all projects your Atlassian account can see
- `jira: off` — skip Jira entirely
- `jira: projects:[KEY1,KEY2,...]` — limit to specific project keys. Quote reserved words (`"INT"`, `"OR"`, etc.).

HubSpot: `on` or `off`. Always read-mostly — notes only.

### Poll
```yaml
poll:
  poll_interval_minutes: 60
  run_on_weekends: false
```

Custom intervals: set `poll_interval_minutes` to any value (e.g. 45). The scheduled-tasks cron will be approximated. For true custom cron schedules, edit the scheduled task directly after setup.

### Tasks
```yaml
tasks:
  tasks_file: ~/.claude/daily-workflow-briefs/tasks.md
```

To disable task management entirely: `tasks_file: none`. The overdue-tasks section will be omitted from briefs; the poll won't edit any file.

### Behavior
```yaml
behavior:
  eod_lookback_hours: 12
```

Knobs:
- `eod_lookback_hours` — how far back the EOD brief looks (default 12).
- `morning_lookback_hours_weekday` — default 12 (Tue–Fri).
- `morning_lookback_hours_monday` — default 72 (captures weekend).

---

## Disabling a specific signal type

In `config.md`, add a `disabled_signals` section:

```yaml
disabled_signals:
  - saved_messages
  - hubspot_deals
```

Known signal keys: `calendar`, `zoom_meetings`, `gmail_vips`, `gmail_team`, `gmail_jira_digests`, `gmail_jira_notifications`, `slack_watch_channels`, `slack_dms`, `saved_messages`, `jira_activity`, `jira_pm_review`, `hubspot_deals`, `tasks_overdue`.

---

## Changing the work-offer allowlist

The default allowlist is intentionally conservative. If you want to enable additional actions (e.g. Jira status changes), you'd need to edit the `skills/brief-poll/SKILL.md` in the installed plugin and add a new `action_type` branch in Step 5.

**Don't do this casually.** The whole point of the allowlist is preventing expensive mistakes. If you fork the plugin and broaden the allowlist, you own the risk.

---

## Changing the cron schedules

The scheduled tasks registered during setup can be edited via the scheduled-tasks MCP:

```
List my scheduled tasks
```

Find the three named `morning-brief`, `brief-poll`, `eod-brief`. Update their cron expressions via:

```
Update scheduled task "brief-poll" to run every 45 minutes weekdays 8am–4pm
```

Standard cron syntax supported.

---

## Multi-workspace users

If you work across multiple Slack workspaces or multiple Jira tenants: v1 supports only one of each. You'd need to pick the primary. If there's demand, multi-tenant config is a reasonable v2 extension.

---

## Troubleshooting

**The brief didn't fire at 7:20am.**
Check the scheduled task is registered and enabled: `List my scheduled tasks`. If it's missing, re-run `/brief-setup`. If it's disabled, enable via the scheduled-tasks MCP.

**The poll isn't executing my `apply 1` replies.**
Check:
1. Are you replying IN THE THREAD (not as a new message)? The poll reads thread replies only.
2. Is the offer still active? Offers expire after 18 hours — check `~/.claude/daily-workflow-briefs/.offers.jsonl` for the offer's `status`.
3. Did the poll run between your reply and now? Check `~/.claude/daily-workflow-briefs/.poll-log.jsonl` for the latest entry. If no poll has run, trigger manually: `/brief-run poll`.

**Offers are numbered wrong / two offers have the same number.**
Each brief or poll post starts offer numbering at 1 scoped to that post. If you reply `apply 1` in a thread with multiple poll posts, the plugin disambiguates by most-recent offer-emission timestamp. To be safe, quote the offer text: `apply 1: Update Confluence Q2 Roadmap`.
