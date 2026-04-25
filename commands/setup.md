---
description: Interactive setup wizard for daily Slack briefs. Walks the user through config (Slack IDs, team, integrations, brief times) in a guided conversation and schedules the three briefs.
---

You are running the daily-briefs setup wizard. Your job: build a config file the user can't easily mess up, then schedule the three brief runs.

Walk the user through setup ONE question at a time, in plain English. Never ask them to edit JSON/YAML directly. If they pick a role preset, pre-fill every field you can from the preset and ONLY ask for fields that can't be defaulted.

## Step 1 — Detect existing config

Check if `~/.claude/daily-workflow-briefs/config.md` already exists.
- If yes: tell the user you found an existing config and ask: "Re-run full setup (overwrites), update a specific field, or cancel?" If "update a specific field", invoke the `/briefs:config` flow instead.
- If no: proceed to Step 2.

Create the directory if missing: `mkdir -p ~/.claude/daily-workflow-briefs`

## Step 2 — Role preset

Ask: "What's your role? This pre-fills sensible defaults. Options: (1) Sales, (2) Customer Success, (3) Product, (4) Engineering, (5) Custom / other."

Read the matching file from the plugin directory: `roles/sales.md`, `roles/cs.md`, `roles/product.md`, `roles/engineering.md`. If Custom, use `roles/_blank.md`.

The role file has suggested VIP tiers, watch-channel naming conventions, integration defaults. Use it as the starting point — you'll layer the user's specifics on top.

## Step 3 — Identity (required)

Ask each one at a time. After each, confirm back what you heard before moving on. If the user gives something that doesn't match the expected format (e.g. `@name` instead of `U...`), catch it and ask again with an example.

1. **Your name** (what you'd say if a meeting next-step said "<name>: do X"). Ask for first name, last name, and any nicknames the Zoom recap might use (e.g. "Sam" if their name is "Samuel"). Store ALL variants — meeting owner-matching uses all of them.

2. **Your Slack user ID** (starts with `U`). Help text: "In Slack, click your profile → three dots → 'Copy member ID'."

3. **Your Slack self-DM channel ID** (starts with `D`). Help text: "Message yourself in Slack, click the channel name at the top, 'Copy link' → the D... part at the end of the URL. This is where every brief will post."

4. **Your work email** (used for Jira `text ~ "email"` searches and Gmail VIP cross-checks).

5. **Timezone** (default from role preset, usually `America/Los_Angeles`). Confirm or override. Accept IANA names (`Europe/Lisbon`) or common aliases (`PT`, `ET`, `Lisbon`, etc.).

## Step 4 — Watch channels

Ask: "What Slack channels should I watch for @-mentions of you and topics you care about? Paste them one at a time as `CID name` — e.g. `C03FEGLDB6J #engineering`. Type `done` when finished."

Remind them to include at least: their team channel, their leadership channel, any customer-facing channel they monitor.

To get a channel ID: click the channel → channel name at top → "Copy link" → the `C...` part.

## Step 5 — Team members

Ask: "Who's on your team? I'll DM-scan them every brief for asks directed at you. Paste one per line as `Name: UID` — e.g. `Sam Lee: U06D1LW0DV3`. Type `done` when finished."

Confirm the count when they're done ("Got 6 team members.").

## Step 6 — VIPs

Ask: "Any VIPs whose emails and messages always deserve attention? (Your boss, a key client, an executive who pings you often.) Paste as `Name: email@domain` — one per line. Type `done` when finished."

Pre-fill common tiers from the role preset (Sales: CEO/CRO; Product: CEO/Head of Eng; etc.) — just ask the user to fill in the names/emails.

## Step 7 — Built-in integrations

Ask each in order:

**Slack**: required, always on. Confirm "Slack MCP is connected? (I'll verify in Step 12.)"

**Google Calendar**: required, always on. Same check.

**Gmail**: required, always on. Same check.

**Jira**: ask "Do you use Jira? (yes-all / yes-specific-projects / no)". If specific: "Which project keys? e.g. `AI, OPS, CLA`." Remind them: "If any project key is a JQL reserved word (common: `INT`, `OR`, `AND`, `NOT`), I'll quote it automatically."

**HubSpot**: ask "Do you use HubSpot for deals? (yes / no)". If yes: "I'll add read-only deal summaries to briefs and offer to add notes to deals — NEVER change deal stage, amount, or owner. OK?"

## Step 8 — Meeting transcriber

This is one of the highest-signal sources for daily briefs — meeting recaps tell you who committed to what. Almost everyone uses one of these.

Ask: "What do you use to transcribe and summarize your meetings?

1. **Zoom** (default — most common)
2. **Granola**
3. **Otter.ai**
4. **Fireflies.ai**
5. **Fathom**
6. **Loom**
7. **None** — I don't use a transcriber
8. **Custom** — different tool; I'll tell you how to reach it

Type a number (default 1)."

For each tool with an official MCP (Zoom, Granola, Otter, Fireflies, Fathom), prefer the MCP path. Ask:

> "Do you have the **<Tool> MCP** installed in Claude Code? (yes / no / not sure)
>
> If no or not sure, install link: `<link>` — it's a 5-minute setup. Or I can fall back to reading <Tool>'s recap emails from Gmail (lower fidelity but works without an MCP)."

If they say **yes** → set `source: mcp`.
If they say **no** and want Gmail fallback → set `source: gmail` with the default `gmail_query` for that tool.
If they say **install it**, give them the link, ask them to come back when ready.

Loom doesn't have a public MCP yet, so Loom is Gmail-only.

### Configuration matrix

| Choice | type | preferred source | MCP install link | Gmail fallback query |
|--------|------|------|------|------|
| 1. Zoom | `zoom` | `mcp` | (built-in Zoom MCP — Claude Code Settings → MCP Servers) | `from:no-reply@zoom.us subject:"Meeting assets" newer_than:1d` |
| 2. Granola | `granola` | `mcp` | https://www.granola.ai/blog/granola-mcp | `from:noreply@granola.ai newer_than:1d` |
| 3. Otter | `otter` | `mcp` | https://help.otter.ai/hc/en-us/articles/35287607569687-Otter-MCP-Server | `from:noreply@otter.ai newer_than:1d` |
| 4. Fireflies | `fireflies` | `mcp` | https://docs.fireflies.ai/getting-started/mcp-configuration | `from:noreply@fireflies.ai newer_than:1d` |
| 5. Fathom | `fathom` | `mcp` | https://composio.dev/toolkits/fathom | `from:no-reply@fathom.video newer_than:1d` |
| 6. Loom | `loom` | `gmail` (no MCP) | (n/a) | `from:no-reply@loom.com subject:"Recap" newer_than:1d` |
| 7. None | `none` | (n/a) | (n/a) | (n/a) |
| 8. Custom | `custom` | (ask) | (ask) | (ask if gmail) |

For Zoom (option 1): tell the user "I'll use the Zoom MCP to pull `summary_plain_text` from each completed meeting. Make sure your Zoom MCP is connected (I'll verify in Step 14). If you don't have it, fallback Gmail pattern is `from:no-reply@zoom.us subject:\"Meeting assets\" newer_than:1d`, but that's lower-fidelity than the MCP — Zoom MCP strongly recommended."

For options 2–5: prefer MCP. Show the install link. Offer Gmail fallback as an explicit second choice.

For option 6 (Loom): only Gmail available. Set `source: gmail` automatically with the Loom pattern.

For option 7 (None): tell the user "Got it — meeting next-steps will be skipped. You can add a transcriber later via `/briefs:config`."

For option 8 (Custom): ask tool name, source (MCP or Gmail), and the corresponding details (MCP name + what-to-check description, or Gmail filter + description).

**Confirm the choice and pattern back to the user before continuing.**

## Step 9 — Additional integrations (optional)

Ask: "Any other tools you want briefs to scan? Examples:
- Tools with a Claude Code MCP — Salesforce, Intercom, Zendesk, Linear, GitHub, Notion, Asana
- Tools that email you summaries — Granola (meeting transcriber), Otter, Fireflies, Fathom, Loom

Type `done` when finished, or `skip` if none."

For each one the user names, ask which path applies:

> "How does the plugin reach `<tool>`?
> 1. Via an MCP I have connected (preferred — works for any current state, real-time)
> 2. Via Gmail (the tool emails me; I'll watch for messages matching a pattern)
> 3. I'm not sure — let me check"

### Path 1 (MCP)
Get a one-line description of what to check, e.g.:
- `salesforce: open opportunities I own with stage changes today`
- `intercom: conversations assigned to me in last 12h`
- `zendesk: tickets where I'm CC'd or watching`

Store as: `name: <tool>, source: mcp, description: <text>`.

### Path 2 (Gmail fallback)
For email-based tools, ask:
- "What email address does `<tool>` send from? (Look at a recent recap email.)"
- "What time window? Default: last 24h."
- "Anything to call out? Default: just summarize the email subject + first paragraph."

Build a Gmail search query. Common patterns:
- Granola: `from:noreply@granola.ai newer_than:1d`
- Otter: `from:noreply@otter.ai newer_than:1d`
- Fireflies: `from:noreply@fireflies.ai newer_than:1d`
- Fathom: `from:no-reply@fathom.video newer_than:1d`
- Loom recap: `from:no-reply@loom.com subject:"Recap" newer_than:1d`

Confirm the pattern with the user before saving. Store as: `name: <tool>, source: gmail, gmail_query: <pattern>, description: <text>`.

### Path 3 (uncertain)
Tell the user: "Open Claude Code → Settings → MCP Servers and search for `<tool>`. If you find it, connect it and pick option 1. If not, check whether the tool emails you summaries — pick option 2 with the sender's email. If neither, skip for now."

### Restrictions for all additional integrations
- **Read-only in v1.** No work offers from these tools regardless of source.
- **Confirm at the end:** show the full list (name, source, description/query) so the user can sanity-check before continuing.

## Step 10 — Brief times

Ask each in order, confirming what you heard back:

1. **Morning brief time**: ask
   > "When should your morning brief fire? (Heads up: scheduled tasks only fire while your computer is awake — pick a time you're usually at your laptop.)
   > 1. **7:30am** — early bird
   > 2. **9:30am** — morning person
   > 3. **11:30am** — sleeping in
   > 4. **Custom** — type a time
   >
   > Pick a number (no default — ask the user to choose so they don't end up with someone else's morning routine)."
   
   For options 1–3, store the corresponding 24-hour value (`07:30`, `09:30`, `11:30`). For option 4, accept formats like `7:45am`, `08:15`, `10am` and convert to 24-hour.

2. **EOD brief time**: "What time should your EOD brief fire? (default 3:30pm your local timezone)." Accept any format like `3:30pm`, `15:30`, `4pm`. Store as 24-hour.

3. **Weekend runs**: "Skip weekends? (default yes — most people don't want weekend briefs.)" Yes → weekdays only. No → include Saturday + Sunday.

## Step 11 — Poll interval

Ask: "How often should the throughout-the-day poll run? It checks thread replies, executes approved offers, and scans new signals. More often = more responsive + more tokens. Options:

- 30 min (~14 runs/day — highest responsiveness, highest cost)
- 1 hour (~7 runs/day — **recommended**)
- 90 min (~5 runs/day)
- 2 hours (~4 runs/day)
- Custom (type minutes as a number)

Rough token cost per day: 30min ≈ 40–60k tokens, 1hr ≈ 20–30k, 90min ≈ 15k, 2hr ≈ 12k. On Sonnet that's roughly $0.60 / $0.30 / $0.20 / $0.15 per day respectively (varies with activity)."

## Step 12 — Work-offer preview detail

Ask: "When the briefs offer to do work for you, how much detail do you want in the offer message?

- **Summary** (default — recommended): short one-line label per offer, type `show 1` in Slack to expand.
- **Full**: full preview text/diff inline. More visual, longer messages.

Pick one."

Store as `work_offer_preview: summary` or `work_offer_preview: full`.

## Step 13 — Tasks file location

Ask: "Where should I keep your tasks.md? (default `~/.claude/daily-workflow-briefs/tasks.md`, or paste your own path. To skip task tracking entirely, say `none`.)"

If the default doesn't exist yet, tell them: "I'll create it from a starter template so the morning brief has something to read." Copy `templates/tasks.md.starter` from the plugin dir to their chosen path.

## Step 14 — MCP connection check

Check which MCPs are connected (by trying minimal calls or reading `~/.claude/settings.json`):
- ✓ Required: Slack, Gmail, Google Calendar
- ✓ Required if `meeting_transcriber.type: zoom`: Zoom MCP
- Optional based on config: Atlassian/Jira, HubSpot
- Additional: any from Step 9

For any missing MCP, tell the user which one and link to `docs/integrations.md` for setup instructions. Warn them briefs will fail or have reduced signal if required MCPs aren't connected by the time the cron fires.

## Step 15 — Write config

Write `~/.claude/daily-workflow-briefs/config.md` in this format. Keep it human-readable — the user may edit it later.

```markdown
# Daily Workflow Briefs — Config

_Generated by `/briefs:setup` on <date>. Edit this file directly or run `/briefs:config` for guided changes._

## Identity
display_names: [Sam, Samuel, Sam Lee]
slack_user_id: U09CAAGER97
slack_self_dm: D09CAAHDF3K
email: sam@company.com
timezone: America/Los_Angeles

## Watch channels
- C03FEGLDB6J: #engineering
- C04FL97MMBJ: #leadership

## Team
- Sid Patel: U06D1LW0DV3
- Riya Chen: U098F0GGV1D

## VIPs (always surface)
- CEO Alex: alex@company.com
- Head of Eng Jordan: jordan@company.com

## Integrations
calendar: on
gmail: on
jira: projects:[AI,OPS,"INT",CLA]
hubspot: off

## Meeting transcriber
type: zoom              # zoom | granola | otter | fireflies | fathom | loom | none | custom
source: mcp             # mcp | gmail
# gmail_query: from:noreply@granola.ai newer_than:1d   # only when source: gmail

## Additional integrations
- name: intercom
  source: mcp
  description: conversations assigned to me last 12h
- name: salesforce
  source: mcp
  description: open opportunities I own with stage changes

## Schedule
morning_time: 07:30
eod_time: 15:30
poll_interval_minutes: 60
run_on_weekends: false

## Behavior
work_offer_preview: summary
eod_lookback_hours: 12

## Tasks
tasks_file: ~/.claude/daily-workflow-briefs/tasks.md
```

## Step 16 — Schedule the three briefs

Use `mcp__scheduled-tasks__create_scheduled_task` three times (confirm with the user before firing each). Use the user's chosen `morning_time`, `eod_time`, `poll_interval_minutes`, and `timezone`. If `run_on_weekends` is true, use `* * *` day-of-week instead of `1-5`.

1. **morning-brief** — cron `<morning_min> <morning_hour> * * <dow>` in their timezone. Runs the `morning-brief` skill.
2. **brief-poll** — cron based on poll interval:
   - 30 min: `*/30 8-15 * * 1-5`
   - 60 min: `0 8-15 * * 1-5`
   - 90 min: `0,30 8-15 * * 1-5` (approximate)
   - 120 min: `0 8,10,12,14 * * 1-5`
   - Custom: ask for the cron string.
3. **eod-brief** — cron `<eod_min> <eod_hour> * * <dow>`. Runs the `eod-brief` skill.

## Step 17 — Offer a dry run

Ask: "Want me to run the morning brief once right now as a dry run? It'll post to your self-DM."

If yes: invoke the `morning-brief` skill immediately.

## Step 18 — Wrap up

Show a short summary:
- ✓ Config at `~/.claude/daily-workflow-briefs/config.md`
- ✓ Three briefs scheduled (list times in user's timezone)
- ✓ Tasks file at `<path>` (or "task tracking off")
- ✓ MCPs connected: <list>
- ⚠ MCPs missing (if any): <list + link to docs/integrations.md>
- Next scheduled run: <the earliest of the three>

Tell them: "Edit `~/.claude/daily-workflow-briefs/config.md` directly any time. Run `/briefs:config` for guided changes. Use `/briefs:run morning | poll | eod` to trigger manually. Run `/briefs:help` for current state + all options."

## Conversational rules

- ONE question at a time. Never dump a wall of fields.
- Always confirm what you heard back before moving on.
- Catch malformed input (e.g. `@user` instead of `U...`) with a gentle correction and example.
- Let the user say `skip` or `later` for any optional field — use sensible defaults.
- At the end, show them the full config file content so they can sanity-check before you write it.
