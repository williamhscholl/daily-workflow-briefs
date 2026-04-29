# Daily Workflow Briefs

### Your work changes all day. Your task list should too.

*An automatic task manager that lives in Slack.*

## Automatically organizes, prioritizes, and updates your tasks

Your work changes all day — Slack threads, meeting decisions, email asks, calendar shifts. Your task list should reflect those changes without you typing them in.

Briefs reads your signals throughout the day and updates your tasks automatically: what you owe, what you're monitoring, what you need to brush up on. All in Slack. No new app.

Three skills run on a schedule:

- 🌤 **Morning** — what to focus on today + top 3 decisions
- 🌇 **End of day** — recap of today + what's up next for tomorrow
- 🔄 **Periodic checker** — scans Slack and other signals throughout the day to update tasks

You pick the times during setup. Defaults: 7:30am / 3:30pm weekdays, checker every hour.

---

## What makes Briefs different

### Briefs offers to do work for you

Briefs doesn't just tell you what's on your plate — it offers to handle it. Each morning brief and periodic check ends with concrete actions Claude can take: edit a Confluence page, comment on a Jira ticket, draft a Slack reply, mark a task done. Approve from your phone in your own tone — `accept 1`, `skip 2`, `edit 3: <new text>`, `show more 1`. Nothing happens without your explicit approval.

### Work is organized into todo and monitoring

Briefs distinguishes between what *you owe* and what's *owed to you*. Action items are tasks. Things you're watching — direct reports' deliveries, customer escalations someone else is fixing, items waiting on confirmation — show up as monitoring items, with their owners and due dates tracked separately. Ask "what should I prioritize today?" and you get both: 🎯 your work + 👁 what others owe you today.

---

## How your work is organized

Two levels: **goals** (projects/objectives like "Q2 Roadmap" or "Hire QA Lead") and **tasks** (the discrete to-dos under them). Every task has a priority, owner, optional due date, and note. The third dimension — **monitoring** — applies at either level: a `## MONITORING:` goal or a `[MONITORING]` task tag flags work where someone else is driving and you're keeping tabs. All stored as plain markdown in `tasks.md`.

```markdown
## GOAL: Q2 Product Roadmap
**Owner:** You | **Priority:** Critical | **Due:** 2026-04-21
- [ ] [CRITICAL]   Review meeting transcript — clean up Q2 doc | Owner: You
- [ ] [HIGH]       Meet with tech leads on estimations | Owner: You + John
- [ ] [MONITORING] Alex: deliver ENG-122 fix | Owner: Alex

## MONITORING: Session Replay Evaluation
**Owner:** Helen | **Priority:** High
- [ ] [HIGH]       Helen: comparison slides for review | Owner: Helen
```

> Don't want the task layer? Set `tasks_file: none` and the briefs run as pure summarizers.

---

## How the pieces fit together

```
                  ┌────────────────────────────────────┐
                  │            config.md               │
                  │   identity • channels • schedule   │
                  │   team • VIPs • integrations       │
                  │   (read by every skill below)      │
                  └─────────────────┬──────────────────┘
                                    │
                  ┌─────────────────▼──────────────────┐
                  │             tasks.md               │
                  │    goals → tasks → monitoring      │
                  │   (your work — read AND written    │
                  │    by the briefs as you approve)   │
                  └─────────────────┬──────────────────┘
                                    │
       ┌────────────────────────────┼────────────────────────────┐
       ▼                            ▼                            ▼
┌──────────────────┐       ┌──────────────────┐       ┌──────────────────┐
│      BRIEFS      │       │       SYNC       │       │     ON-DEMAND    │
│  → Slack self-DM │       │  → Slack self-DM │       │  → Claude chat   │
├──────────────────┤       ├──────────────────┤       ├──────────────────┤
│ morning-brief    │       │ brief-poll       │       │ tasks            │
│  today's agenda  │       │  syncs your      │       │                  │
│  7:30 am         │       │  Slack replies   │       │  "top three?"    │
│                  │       │  into tasks.md   │       │  "overdue?"      │
│ eod-brief        │       │  hourly 8a–4p    │       │  "waiting on?"   │
│  day recap +     │       │                  │       │                  │
│  tomorrow's plan │       │                  │       │                  │
│  3:30 pm         │       │                  │       │                  │
└──────────────────┘       └──────────────────┘       └──────────────────┘
```

The four skills share `config.md` (your settings) and `tasks.md` (your work) as foundation files.

**Briefs** push state out to you — morning agenda, end-of-day recap, tomorrow's top 3 — on a fixed schedule. The morning brief also runs a "carryover" pass first: it reads any approvals you posted to last night's EOD thread and applies them before producing today's brief, so nothing accepted overnight gets lost.

**Sync** is the inverse direction. The poll reads your thread replies and converts them into edits on `tasks.md`. It picks up both **explicit approvals** (`accept 1`, `skip 2`, `edit 1: …`, `show more 3`) and **natural-language updates** (`mark X done`, `add a task to <goal>`, `move Y to Friday`). Posts a `✅ Tasks updated` confirmation in-thread.

**On-demand** is `tasks` — fires when you ask in Claude Code chat. Reads `tasks.md` directly, no Slack involvement.

Slash commands (`/briefs:setup`, `/briefs:run`, `/briefs:help`, `/briefs:config`, `/briefs:monitoring`) are manual entry points — useful for testing, ad-hoc runs, or alternatives to natural language.

---

## Quick start

Three steps. ~10 minutes.

### 1. Install Claude Code, then the plugin

If you don't have Claude Code yet:

```bash
# Mac / Linux (Terminal)
curl -fsSL https://claude.ai/install.sh | bash

# Windows (PowerShell, NOT Git Bash)
irm https://claude.ai/install.ps1 | iex
```

> ⚠ Claude Code is a separate product from the Claude Desktop chat app. Restart your terminal after install, then verify with `claude --version`.

Install the plugin:

```bash
claude plugin marketplace add williamhscholl/daily-workflow-briefs && claude plugin install briefs@briefs
```

(On PowerShell <7, run as two separate lines — `&&` requires PowerShell 7+.)

### 2. Connect MCPs

Open the Claude Code app → Settings → MCP Servers.

- **Required**: Slack, Google Calendar, Gmail
- **Pick a meeting transcriber**: Zoom (default) / Granola / Otter / Fireflies / Fathom / Loom / None
- **Optional**: Atlassian (Jira + Confluence), HubSpot
- **Or any MCP**: Salesforce, Intercom, Zendesk, Linear, GitHub, Notion, Asana — read-only signal source

See [docs/integrations.md](docs/integrations.md) for details.

### 3. Run setup

In Claude Code chat:

```
/briefs:setup
```

The wizard walks you through identity, channels, team, VIPs, transcriber, schedule, and preferences. Briefs start firing on your schedule — first one lands the next weekday morning.

---

## Commands

Slash commands are namespaced `briefs:`. Plain English works too — both invoke the same skills.

| Command | Plain language | What it does |
|---|---|---|
| `/briefs:setup` | "set up my briefs" | Interactive setup wizard |
| `/briefs:run morning` | "run my morning brief" | Trigger morning brief now |
| `/briefs:run eod` | "run my EOD brief" | Trigger EOD brief now |
| `/briefs:run poll` | "process my Slack replies" | Process pending approvals immediately |
| `/briefs:help` | "what's in my brief config?" | Show config + schedule + recent runs |
| `/briefs:config` | "change my morning to 8am" | Edit a single config field via chat |
| `/briefs:monitoring` | "what does Sid owe me?" | Show what's owed to you, grouped by owner |

Config edits work without slash commands too: "change my poll interval to 90 minutes", "add Riya to my team", "switch transcriber to Granola".

### Ask about your tasks

The `tasks` skill auto-fires when you ask about your work. Just type the question:

| You ask | What you get |
|---|---|
| "What should I prioritize today?" | 🎯 your work today + 👁 owed to you today |
| "What's overdue?" | 🔴 yours overdue + ⏳ others' overdue |
| "What am I waiting on?" | All watching items, grouped by owner |
| "What does [name] owe me?" | Watching items filtered to that owner |
| "What's on my plate this week?" | Tasks due in next 7 days |
| "What did I get done this week?" | Completed tasks since Monday |
| "Show me everything for [goal]" | All tasks in that goal |
| "How's my workload?" | Counts + top owners + overdue snapshot |
| "Find the task about [topic]" | Full-text search |

Modifications work the same way: "mark X done", "add a task to <goal>: …", "move X to Friday", "bump Y to critical".

---

## What Briefs can do on your behalf

When you `apply N` an offer, Briefs can take exactly these actions:

- ✅ Edit / append to a Confluence page (with diff preview)
- ✅ Add a comment to a Jira ticket (with preview)
- ✅ Draft a Slack reply — *drafted only, never auto-sent*
- ✅ Add / update / mark-done a task in your `tasks.md`
- ✅ Add a read-only note to a HubSpot deal
- ❌ Send email
- ❌ Change Jira status / assignee / priority
- ❌ Change HubSpot deal stage / amount / owner / close date
- ❌ Anything destructive (delete, close, archive)
- ❌ Write actions on additional integrations (read-only in v1)

Every execution shows the exact text/diff applied, in-thread, so you can catch mistakes immediately.

**Preview detail:** during setup, pick `summary` (one-line offers; type `show 1` to expand) or `full` (full preview inline). Change later via `/briefs:config switch preview to full`.

> The EOD brief posts offers, but they're processed differently than morning/poll offers — the next morning's brief reads any replies you posted overnight and applies them as a "carryover" pass before producing the morning brief itself. Confirmations show up at the top of the morning brief in a `📥 Carried over from yesterday's EOD` section. The EOD offer message tells you this explicitly: "I'll pick up your reply in tomorrow's morning brief and follow through then."

---

## Roles

`/briefs:setup` pre-fills defaults based on your role:

- [Sales](roles/sales.md)
- [Customer Success](roles/cs.md)
- [Product](roles/product.md)
- [Engineering](roles/engineering.md)
- [Custom / Other](roles/_blank.md)

Each role file documents what signals get prioritized, typical work offers, and cautions. Read yours before running setup.

---

## Updating

Updates are manual — nothing auto-updates. Run when you want to:

```bash
claude plugin marketplace update briefs
claude plugin update briefs@briefs
```

Your config and `tasks.md` are preserved.

**Before updating:** read [CHANGELOG.md](CHANGELOG.md) (newest first), or compare your version to main:
```
https://github.com/williamhscholl/daily-workflow-briefs/compare/v0.6.0...main
```

If a new version breaks something, roll back by reinstalling the previous tag:

```bash
claude plugin uninstall briefs
git clone --branch v0.6.0 https://github.com/williamhscholl/daily-workflow-briefs ~/briefs-prev
claude plugin marketplace add ~/briefs-prev
claude plugin install briefs@briefs
```

Then file an issue at [github.com/williamhscholl/daily-workflow-briefs/issues](https://github.com/williamhscholl/daily-workflow-briefs/issues).

---

## Privacy & data

**Everything you care about stays on your machine.** No telemetry, no phone-home, no external logging. The plugin author cannot see your data.

- **Config, tasks.md, offers ledger, poll log**: all in `~/.claude/daily-workflow-briefs/`.
- **Slack / Gmail / Calendar / Zoom / Jira / HubSpot**: flow through *your* OAuth tokens via *your* MCP connections — never through the plugin author.
- **Briefs post only to your Slack self-DM** (a DM with yourself; no one else sees it).
- **Anthropic API**: signals get sent for Claude to reason over — same data path as any Claude Code session. See [Anthropic's privacy policy](https://www.anthropic.com/legal/privacy).
- **Open-source trust**: like any GitHub plugin, `update` pulls code from this repo. Updates are manual (read the changelog first); the plugin is text-only markdown (auditable by skimming); pin to a tag if you don't want changes.

For SOC 2 / HIPAA / GDPR compliance: this plugin adds nothing beyond what Claude Code already does — talk to your security team about Claude Code first.

---

## Where files live

- **Config**: `~/.claude/daily-workflow-briefs/config.md` — human-readable, editable any time
- **Tasks**: `~/.claude/daily-workflow-briefs/tasks.md` (default; configurable)
- **Ledger**: `~/.claude/daily-workflow-briefs/.offers.jsonl` — tracks which offers were applied/skipped
- **Run log**: `~/.claude/daily-workflow-briefs/.poll-log.jsonl` — one line per poll run

---

## Uninstall

```bash
claude plugin uninstall briefs
```

The three scheduled cron tasks stop firing. Config and `tasks.md` stay in `~/.claude/daily-workflow-briefs/` — delete that directory manually for a clean wipe.

---

## FAQ

**Does this send email / DMs automatically?**  
No. Briefs only post to your self-DM. Slack replies are drafted only — you copy/paste.

**What if I approve an offer by mistake?**  
Every execution is logged with a timestamp and the exact text applied. To undo: revert manually (edit the Confluence page, delete the Jira comment). No "revert" command in v1.

**Can I use this without the task-tracking?**  
Yes. Set `tasks_file: none` in setup. The overdue section drops; everything else still works.

**What's the token cost?**  
~$0.15–$0.60/day on Sonnet depending on poll interval. Morning + EOD are fixed (~5k tokens each); poll scales with interval.

**Can I add my own tools (Salesforce, Intercom, Zendesk, etc.)?**  
Yes. Either via a Claude Code MCP, or — for tools that email you summaries (Granola, Otter, Fireflies, Fathom) — via a Gmail search filter. Both paths are read-only in v1. See [docs/integrations.md](docs/integrations.md#additional-integrations-any-tool).

**Other agents (Codex, Gemini)?**  
v1 is Claude Code only.

---

## Contributing / issues

File issues at [github.com/williamhscholl/daily-workflow-briefs/issues](https://github.com/williamhscholl/daily-workflow-briefs/issues).

## License

MIT
