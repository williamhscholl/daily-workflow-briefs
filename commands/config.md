---
description: Edit a single field in the daily-briefs config without re-running the full setup wizard. Accepts natural language ('change my morning to 8am', 'add Sid to team', 'turn off jira', 'add Salesforce as integration', 'switch to full preview').
argument-hint: Describe the change (or leave blank to browse fields)
---

You are editing the user's daily-briefs config. Their request is: `$ARGUMENTS`

## Step 1 — Load the config

Read `~/.claude/daily-workflow-briefs/config.md`.

If missing: tell the user "Run `/briefs:setup` first to create your config." Stop.

## Step 2 — Parse the user's request

If `$ARGUMENTS` is empty: ask "What would you like to change? Examples:
- `change my morning brief to 8am`
- `add Sid (U06D1LW0DV3) to my team`
- `remove Riya from my team`
- `add C0ABC123 #new-channel to watch list`
- `change my poll interval to 30 minutes`
- `turn off hubspot`
- `turn on jira for projects AI, OPS, CLA`
- `change my timezone to Europe/Lisbon`
- `disable weekend runs`
- `add Salesforce as integration: open opportunities I own`
- `remove Intercom integration`
- `switch work-offer preview to full` (or `summary`)
- `change tasks file to ~/notes/tasks.md` (or `disable task tracking`)"

Wait for their answer.

## Step 3 — Classify the request

Map the natural-language request to a specific field change. Recognized patterns:

### Time changes
- `change morning (brief/time) to <time>` → update `morning_time`
- `change eod (brief/time) to <time>` → update `eod_time`
- Resolve: `8am` → `08:00`, `3:30pm` → `15:30`. Store 24-hour.

### Timezone
- `change (my) timezone to <tz>` → update `timezone`. Accept IANA (`Europe/Lisbon`) or aliases (`PT` → `America/Los_Angeles`, `ET` → `America/New_York`, `Lisbon` → `Europe/Lisbon`).

### Poll interval
- `change poll (interval) to <N> (min|hr)` → update `poll_interval_minutes`
- Warn on unusual values: "30 min uses ~40k tokens/day vs 60 min at ~20k. Confirm?"

### Team members
- `add <Name> (<UID>) to team` → append to `team`
- `remove <Name> from team` → delete entry
- If UID missing: "What's <Name>'s Slack user ID? (Profile → 'Copy member ID' — starts with `U`.)"

### Watch channels
- `add <CID> <#name> to watch list` → append to `watch_channels`
- `remove <#name> from watch list` → delete entry

### VIPs
- `add <Name> <email> as VIP` → append to `vips`
- `remove <Name> from VIPs` → delete entry

### Built-in integrations (Jira / HubSpot)
- `turn on/off jira` → flip `integrations.jira`
- `turn on jira for projects <KEYS>` → set to `projects:[KEYS]`. Auto-quote reserved JQL words (`"INT"`, `"OR"`, etc.).
- `turn on/off hubspot` → flip `integrations.hubspot`

### Additional integrations (Salesforce, Intercom, Zendesk, Granola, Otter, etc.)

Each integration has two possible source types: **MCP** (when the user has a Claude Code MCP for the tool) or **Gmail** (when the tool emails the user, like meeting transcribers).

**MCP form:**
- `add <name> as integration: <description>` → append `name: <name>, source: mcp, description: <text>`.
- `add <name>` (no description) → append with default description "items relevant to me, last 24h".

**Gmail form (for tools without MCPs but that email summaries):**
- `add <name> via gmail: <gmail-query>` → append `name: <name>, source: gmail, gmail_query: <query>, description: <inferred>`.
- Common patterns the user might give:
  - Granola: `from:noreply@granola.ai newer_than:1d`
  - Otter: `from:noreply@otter.ai newer_than:1d`
  - Fireflies: `from:noreply@fireflies.ai newer_than:1d`
  - Fathom: `from:no-reply@fathom.video newer_than:1d`
- If the user says `add granola` (no source specified), check whether they have a Granola MCP connected. If yes → MCP form. If not → ask "I don't see a Granola MCP. Granola emails meeting recaps — should I watch your Gmail for them? If yes, what address does Granola send from? (Default: `noreply@granola.ai`)" Then build the Gmail form.

**Other operations:**
- `remove <name> integration` → delete matching entry by `name`.
- `change <name> description to <text>` → update the description string.
- `change <name> gmail query to <query>` → update the gmail_query (Gmail-source entries only).
- `switch <name> to gmail (with <query>)` / `switch <name> to mcp` → flip the source for an existing entry.

Tell the user: "Additional integrations are read-only signals only. The plugin won't offer write actions for them in v1, regardless of source."

### Weekend behavior
- `enable/disable weekend runs` → flip `run_on_weekends`

### Work-offer preview detail
- `switch (work-offer/offer) preview to full` → set `work_offer_preview: full`
- `switch (work-offer/offer) preview to summary` → set `work_offer_preview: summary`

### Tasks file
- `change tasks file to <path>` → update `tasks_file`
- `disable task tracking` → set `tasks_file: none`

### Anything else (fallback)
- If the request doesn't match a known pattern, ask for clarification with 2–3 concrete example phrasings.

## Step 4 — Confirm before writing

Show the user exactly what you're about to change:

```
I'll update your config:
• [field]: <old> → <new>

Save? (yes / no / cancel)
```

For destructive changes (removing a team member, disabling tasks, removing an integration), require explicit `yes` before writing.

## Step 5 — Write the change

Edit `~/.claude/daily-workflow-briefs/config.md` directly. Preserve all other fields, all formatting, and any user comments (lines starting with `#`). Only touch the line(s) you're changing.

## Step 6 — If the change affects scheduling, update cron

If the change was to `morning_time`, `eod_time`, `poll_interval_minutes`, `timezone`, or `run_on_weekends`: scheduled tasks need updating.

1. List current scheduled tasks to find `morning-brief`, `brief-poll`, `eod-brief`.
2. For each affected one, call `mcp__scheduled-tasks__update_scheduled_task` with the new cron.
3. Build the cron in the user's timezone (normalize to UTC if the MCP requires UTC).

Cron templates:
- Morning: `<min> <hour> * * 1-5` weekdays-only, or `<min> <hour> * * *` if `run_on_weekends`.
- EOD: same pattern.
- Poll:
  - 30 min: `*/30 8-15 * * 1-5`
  - 60 min: `0 8-15 * * 1-5`
  - 90 min: `0,30 8-15 * * 1-5` (approximation)
  - 120 min: `0 8,10,12,14 * * 1-5`
  - Custom: compute from user minutes.

Confirm each cron update: "Updated the morning-brief cron to fire at 8:00am your timezone."

## Step 7 — Offer next step

```
✓ Saved. Your config now reads:
• <changed field>: <new value>

Anything else? Run `/briefs:help` to see your full config.
```

## Conversational rules

- One field at a time. If the user asks to change multiple things in one prompt, handle sequentially with confirmation after each.
- Always show what you're changing BEFORE writing, and confirm after.
- Never modify fields the user didn't mention.
- Plain English — no JSON, no cron strings shown in output.
- If you can't cleanly classify the request, ask for clarification with 2–3 concrete example phrasings.
