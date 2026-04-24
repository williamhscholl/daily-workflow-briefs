---
description: Edit a single field in the Daily Workflow Briefs config without re-running the full setup wizard. Accepts natural language ('change my morning time to 8am', 'add Hugo to my team', 'turn off jira', 'add #new-channel to watch list').
argument-hint: Describe the change you want (or leave blank to browse fields)
---

You are editing the user's Daily Workflow Briefs config. Their request is: `$ARGUMENTS`

## Step 1 — Load the config

Read `~/.claude/daily-workflow-briefs/config.md`.

If missing: tell the user "Run `/daily-workflow-briefs:brief-setup` first to create your config." Stop.

## Step 2 — Parse the user's request

If `$ARGUMENTS` is empty: ask the user "What would you like to change? Examples:
- `change my morning brief to 8am`
- `add Hugo (U06D1LW0DV3) to my team`
- `remove Raphael from my team`
- `add C0ABC123 #new-channel to watch list`
- `change my poll interval to 30 minutes`
- `turn off hubspot`
- `turn on jira for projects AI, WAT, PUL`
- `change my timezone to Europe/Lisbon`
- `disable weekend runs`"

Wait for their answer.

## Step 3 — Classify the request

Map the natural-language request to a specific field change. Recognized patterns:

### Time changes
- `change morning (brief/time) to <time>` → update `morning_time` field
- `change eod (brief/time) to <time>` → update `eod_time` field
- Resolve times: `8am` → `08:00`, `3:30pm` → `15:30`. Assume 24-hour internally.

### Timezone
- `change (my) timezone to <tz>` → update `timezone` field. Accept both IANA (`Europe/Lisbon`, `America/Los_Angeles`) and common aliases (`PT` → `America/Los_Angeles`, `ET` → `America/New_York`, `Lisbon` → `Europe/Lisbon`).

### Poll interval
- `change poll (interval) to <N> (min|hr|minutes|hours)` → update `poll_interval_minutes`
- Warn if they pick something unusual: "30 minutes uses ~40k tokens/day vs 60 minutes at ~20k. Confirm?"

### Team members
- `add <Name> (<UID>) to (my) team` → append to `team` list
- `remove <Name> from (my) team` → delete matching entry
- If they don't provide the UID: ask "What's <Name>'s Slack user ID? (Right-click their name in Slack → Copy member ID — starts with `U`.)"

### Watch channels
- `add <CID> (<#name>) to (watch list/channels)` → append to `watch_channels`
- `remove <#name> from (watch list)` → delete matching entry

### VIPs
- `add <Name> (<email>) as (a) VIP` → append to `vips`
- `remove <Name> from VIPs` → delete matching entry

### Integrations
- `turn on/off jira` → flip `integrations.jira` to `on` or `off`
- `turn on jira for projects <KEY1, KEY2>` → set `integrations.jira` to `projects:[KEY1,KEY2]`. Quote reserved JQL words (`"INT"`, `"OR"`, etc.).
- `turn on/off hubspot` → flip `integrations.hubspot`

### Weekend behavior
- `enable/disable weekend runs` → flip `run_on_weekends`

### Tasks file
- `change my tasks file to <path>` → update `tasks_file`
- `disable task tracking` → set `tasks_file: none`

### Anything else (fallback)
- If the request doesn't match a known pattern, ask the user to clarify with examples.

## Step 4 — Confirm before writing

Show the user exactly what you're about to change:

```
I'll update your config:
• [field]: <old value> → <new value>

Save? (yes / no / cancel)
```

For destructive changes (removing a team member, disabling tasks), require explicit `yes` before writing.

## Step 5 — Write the change

Edit `~/.claude/daily-workflow-briefs/config.md` directly. Preserve all other fields, all formatting, and any user comments (lines starting with `#`). Only touch the line(s) you're changing.

## Step 6 — If the change affects scheduling, update cron

If the change was to `morning_time`, `eod_time`, `poll_interval_minutes`, `timezone`, or `run_on_weekends`: the scheduled tasks need updating. Use the scheduled-tasks MCP:

1. List current scheduled tasks to find `morning-brief`, `brief-poll`, `eod-brief`.
2. For each affected one, call `mcp__scheduled-tasks__update_scheduled_task` with the new cron expression.
3. Build the cron expression using the user's timezone (normalize to UTC if the MCP requires UTC).

Cron templates:
- Morning: `<min> <hour> * * 1-5` if weekdays-only, or `<min> <hour> * * *` if run_on_weekends.
- EOD: same pattern.
- Poll:
  - 30 min: `*/30 8-15 * * 1-5`
  - 60 min: `0 8-15 * * 1-5`
  - 90 min: `0,30 8-15 * * 1-5` (approximation — true 90-min needs multiple entries)
  - 120 min: `0 8,10,12,14 * * 1-5`
  - Custom: compute from user's minutes.

Confirm each cron update back to the user: "Updated the <morning-brief> cron to fire at 8:00am your timezone (Europe/Lisbon)."

## Step 7 — Offer next step

End with:
```
✓ Saved. Your config now reads:
• <changed field>: <new value>

Anything else? Run `/daily-workflow-briefs:brief-help` to see your full config.
```

## Conversational rules

- One field at a time. If the user asks to change multiple things in one prompt ("change morning to 8am and add Hugo to team"), handle sequentially with a confirmation after each.
- Always show what you're changing BEFORE writing, and confirm after.
- Never modify fields the user didn't mention.
- Plain English — no JSON, no cron strings shown in output (they're internal).
- If you can't cleanly classify the request, ask for clarification with 2–3 concrete examples of phrasings you'd understand.
