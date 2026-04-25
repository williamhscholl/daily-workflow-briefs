---
description: Show the user's current daily-briefs config, scheduled times, last-run stats, and how to use the plugin. Plain English.
---

You are showing the user their current daily-briefs setup and how to use it. Plain English like a colleague walking them through. No JSON, no YAML.

## Step 1 — Check if plugin is configured

Read `~/.claude/daily-workflow-briefs/config.md`.

**If missing:** tell the user "Looks like you haven't run setup yet. Run `/briefs:setup` to get started." Stop.

## Step 2 — Show their current setup

Parse config.md and display a readable summary:

```
📋 Your daily-briefs config

👤 Identity
• Name: <display_names[0]>
• Slack user: <slack_user_id>
• Self-DM (where briefs post): <slack_self_dm>
• Timezone: <timezone>

⏰ Schedule
• Morning brief: <morning_time> <timezone>, weekdays
• Poll: every <poll_interval_minutes> minutes, 8am–4pm
• EOD brief: <eod_time> <timezone>, weekdays
• Weekends: <on/off>

👁 Watching
• <N> Slack channels: <list>
• <N> team members (DM-scanned): <names>
• <N> VIPs (email): <names>

🔌 Integrations
• Slack, Google Calendar, Gmail, Zoom: required (always on)
• Jira: <on / specific projects / off>
• HubSpot: <on / off>
• Additional: <list any from additional_integrations or "none">

🤝 Work-offer preview: <summary | full>
📄 Tasks file: <tasks_file path or 'disabled'>
```

## Step 3 — Show recent run stats

Read `~/.claude/daily-workflow-briefs/.poll-log.jsonl` (tail the last ~5 lines).

```
📊 Recent activity
• Last poll run: <timestamp, e.g. "2h ago">
• Approvals processed last 24h: <N>
• Offers emitted last 24h: <N>
• Task changes last 24h: <N>
```

If `.poll-log.jsonl` doesn't exist or is empty: "No poll runs logged yet — this is a fresh install. First poll fires on your next scheduled cycle."

## Step 4 — Show what the user can do

```
🛠 What you can do

In Slack (reply to the brief thread):
• `apply 1` — approve offer #1
• `skip 2` — dismiss offer #2
• `show 3` — preview offer #3 before approving
• `edit 1: <new text>` — revise the draft before applying
• Natural language: "mark X done", "add a task to <goal>: …", "move X to Friday"

In Claude Code:
• `/briefs:run morning` — trigger morning brief now
• `/briefs:run poll` — process your Slack approvals now (don't wait for scheduled poll)
• `/briefs:run eod` — trigger EOD brief now
• `/briefs:config` — change a config field via chat ("change morning to 8am", "add Salesforce as an integration")
• `/briefs:setup` — re-run the full setup wizard

Or just talk to Claude naturally — "run my morning brief", "what's in my brief config?", "add Sid to my team", etc. Slash commands are shortcuts, not requirements.

Direct file edit:
• Open `~/.claude/daily-workflow-briefs/config.md` in any editor for quick changes.
```

## Step 5 — Offer next actions

End with a short question: "Anything you'd like to change or run right now?"

Listen for natural answers:
- "Run my morning brief" → invoke `morning-brief` skill
- "Change my timezone to X" → run the `/briefs:config` logic
- "Show my latest brief" → fetch the most recent brief from Slack self-DM
- etc.

## Conversational rules

- Plain English throughout. No JSON, no YAML, no cron strings in output.
- If something in the config looks broken (e.g. slack_user_id missing, timezone unparseable), flag it gently: "Heads up — I see <issue>. Want to fix it now?"
- Under 400 words total unless the user asks for more detail.
