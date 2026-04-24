---
description: 'Interactive setup wizard for Daily Workflow Briefs. Walks the user through config (Slack IDs, team, integrations) in a guided conversation and schedules the three briefs.'
---

You are running the Daily Workflow Briefs setup wizard. Your job: build a config file the user can't easily mess up, then schedule the three brief runs.

Walk the user through setup ONE question at a time, in plain English. Never ask them to edit JSON/YAML directly. If they pick a role preset, pre-fill every field you can from the preset and ONLY ask for fields that can't be defaulted (Slack IDs, team members).

## Step 1 — Detect existing config

Check if `~/.claude/daily-workflow-briefs/config.md` already exists.
- If yes: tell the user you found an existing config and ask: "Re-run full setup (overwrites), update a specific field, or cancel?" Let them choose.
- If no: proceed to Step 2.

Create the directory if missing: `mkdir -p ~/.claude/daily-workflow-briefs`

## Step 2 — Role preset

Ask: "What's your role? This pre-fills sensible defaults. Options: (1) Sales, (2) Customer Success, (3) Product, (4) Engineering, (5) Custom / other."

Read the matching file from the plugin directory: `roles/sales.md`, `roles/cs.md`, `roles/product.md`, `roles/engineering.md`. If Custom, use `roles/_blank.md` (all fields empty).

The role file has suggested VIP tiers, watch-channel naming conventions, integration defaults, etc. Use it as the starting point — you'll layer the user's specifics on top.

## Step 3 — Identity (required)

Ask each of these one at a time. After each, confirm back what you heard before moving on. If they give you something that doesn't match the expected format (e.g. "@will" instead of `U...`), catch it and ask again with an example.

1. **Your name** (what you'd say if a meeting next-step said "<name>: do X"). Ask for first name, last name, and any nicknames the Zoom recap might use (e.g. "William" if their name is "Will"). Store ALL variants — meeting owner-matching uses all of them.

2. **Your Slack user ID** (starts with `U`). Help text: "In Slack, click your profile → three dots → 'Copy member ID'."

3. **Your Slack self-DM channel ID** (starts with `D`). Help text: "Message yourself in Slack, click the channel name at the top, 'Copy link' → the D... part at the end of the URL. This is where every brief will post."

4. **Your work email** (used for Jira `text ~ "email"` searches and to cross-check Gmail VIPs).

5. **Timezone** (default from role preset, usually `America/Los_Angeles`). Confirm or override.

## Step 4 — Watch channels

Ask: "What Slack channels should I watch for @-mentions of you and topics you care about? Paste them one at a time as `CID name` — e.g. `C03FEGLDB6J #engineering`. Type `done` when finished."

The role preset suggests channel types (e.g. Sales role suggests pipeline/exec/team). Remind them to include at least: their team channel, their leadership channel, any customer-facing channel they monitor.

To get a channel ID: click the channel → channel name at top → "Copy link" → the `C...` part.

## Step 5 — Team members

Ask: "Who's on your team? I'll DM-scan them every brief for asks directed at you. Paste one per line as `Name: UID` — e.g. `Hugo Salomon: U06D1LW0DV3`. Type `done` when finished."

Confirm the count when they're done ("Got 6 team members.").

## Step 6 — VIPs

Ask: "Any VIPs whose emails and messages always deserve attention? (Your boss, a key client, an executive who pings you often.) Paste as `Name: email@domain` — one per line. Type `done` when finished."

Pre-fill common tiers from the role preset (Sales: CEO/CRO; Product: CEO/Head of Eng; etc.) — just ask the user to fill in the names/emails.

## Step 7 — Integrations

Ask each in order:

**Calendar**: assumed ON (required). Confirm "Google Calendar MCP is connected? (I'll check in Step 10.)"

**Gmail**: assumed ON (required). Same check.

**Zoom**: assumed ON (required). Same check.

**Jira**: ask "Do you use Jira? (yes-all / yes-specific-projects / no)". If specific: "Which project keys? e.g. `AI, WAT, INT, PLA, TEL, PUL`." Remind them: "If any project key is a JQL reserved word (common: `INT`, `OR`, `AND`, `NOT`), I'll quote it automatically."

**HubSpot**: ask "Do you use HubSpot for deals? (yes / no)". If yes, clarify: "I'll add read-only deal summaries to briefs and offer to add notes to deals — NEVER change deal stage, amount, or owner. OK?"

## Step 8 — Poll interval

Ask: "How often should the throughout-the-day poll run? It checks thread replies, executes approved offers, and scans new signals. More often = more responsive + more tokens. Options:

- 30 min (~14 runs/day — highest responsiveness, highest cost)
- 1 hour (~7 runs/day — **recommended**)
- 90 min (~5 runs/day)
- 2 hours (~4 runs/day)
- Custom (type minutes as a number)

Rough token cost per day: 30min ≈ 40–60k tokens, 1hr ≈ 20–30k, 90min ≈ 15k, 2hr ≈ 12k. On Sonnet that's roughly $0.60 / $0.30 / $0.20 / $0.15 per day respectively (varies with activity)."

## Step 9 — Tasks file location

Ask: "Where should I keep your tasks.md? (default `~/.claude/daily-workflow-briefs/tasks.md`, or paste your own path.)"

If the default doesn't exist yet, tell them: "I'll create it from a starter template so the morning brief has something to read." Copy `templates/tasks.md.starter` from the plugin dir to their chosen path.

## Step 10 — MCP connection check

Check which MCPs are connected (by trying a minimal call or reading `~/.claude/settings.json`):
- ✓ Required: Slack, Gmail, Google Calendar, Zoom
- Optional based on config: Atlassian/Jira, HubSpot

For any missing MCP, tell the user which one and link to `docs/integrations.md` for setup instructions. Warn them briefs will fail if required MCPs aren't connected by the time the cron fires.

## Step 11 — Write config

Write `~/.claude/daily-workflow-briefs/config.md` in this format. Keep it human-readable — the user may edit it later.

```markdown
# Daily Workflow Briefs — Config

_Generated by `/brief-setup` on <date>. Edit this file directly or re-run `/brief-setup` any time._

## Identity
display_names: [Will, William, Will Scholl, William Scholl]
slack_user_id: U09CAAGER97
slack_self_dm: D09CAAHDF3K
email: will@company.com
timezone: America/Los_Angeles

## Watch channels
- C03FEGLDB6J: #engineering
- C04FL97MMBJ: #leadership

## Team
- Hugo Salomon: U06D1LW0DV3
- Raphael Carvalho: U098F0GGV1D

## VIPs (always surface)
- CEO Anoosh: anoosh@company.com
- Head of Eng Sohrab: sohrab@company.com

## Integrations
calendar: on
gmail: on
zoom: on
jira: projects:[AI,WAT,"INT",PLA,TEL,PUL]
hubspot: off

## Poll
poll_interval_minutes: 60
run_on_weekends: false

## Tasks
tasks_file: ~/.claude/daily-workflow-briefs/tasks.md

## Behavior
eod_lookback_hours: 12
```

## Step 12 — Schedule the three briefs

Use `mcp__scheduled-tasks__create_scheduled_task` three times (confirm with the user before firing each):

1. **morning-brief** — cron `20 7 * * 1-5` in their timezone (7:20am Mon–Fri). Runs the `morning-brief` skill.
2. **brief-poll** — cron based on their poll interval:
   - 30 min: `*/30 8-15 * * 1-5`
   - 60 min: `0 8-15 * * 1-5`
   - 90 min: `0,30 8-15 * * 1-5` (approximate; 90-min true interval needs custom)
   - 120 min: `0 8,10,12,14 * * 1-5`
   - Custom: ask them for the cron string.
3. **eod-brief** — cron `30 15 * * 1-5` (3:30pm Mon–Fri).

## Step 13 — Offer a dry run

Ask: "Want me to run the morning brief once right now as a dry run? It'll post to your self-DM."

If yes: invoke the `morning-brief` skill immediately.

## Step 14 — Wrap up

Show a short summary:
- ✓ Config at `~/.claude/daily-workflow-briefs/config.md`
- ✓ Three briefs scheduled (list times)
- ✓ Tasks file at `<path>`
- ✓ MCPs connected: <list>
- ⚠ MCPs missing (if any): <list + link to docs/integrations.md>
- Next scheduled run: <the earliest of the three>

Tell them: "Edit `~/.claude/daily-workflow-briefs/config.md` any time to change things. Re-run `/brief-setup` for a guided edit. Use `/brief-run morning | poll | eod` to trigger manually."

## Conversational rules

- ONE question at a time. Never dump a wall of fields.
- Always confirm what you heard back before moving on.
- Catch malformed input (e.g. `@user` instead of `U...`) with a gentle correction and example.
- Let the user say `skip` or `later` for any optional field — use sensible defaults.
- At the end, show them the full config file content so they can sanity-check before you write it.
