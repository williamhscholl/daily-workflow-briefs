# Daily Workflow Briefs

**A Claude Code plugin that runs your day via Slack.** Three automated briefs post to your Slack self-DM:

- 🌤 **Morning brief** (7:20am weekdays) — today's meetings, overdue work, meeting next-steps, top-3 decisions
- 🔄 **Throughout-the-day poll** (every 1hr by default) — processes your thread replies, applies approved work, surfaces new signals
- 🌇 **EOD brief** (3:30pm weekdays) — meetings attended, tomorrow's top-3 focus

Plus a `🤝 Work I can do for you` section that shows up as the first reply to each brief. You approve with `apply 1` / `skip 2` / `edit 1: …` and the plugin executes on the next poll (or immediately if you run `/brief-run poll`).

---

## Install

One command from any terminal:

```bash
claude plugin marketplace add williamhscholl/daily-workflow-briefs && claude plugin install daily-workflow-briefs@daily-workflow-briefs
```

Then configure — in Claude Code (desktop or CLI, either works):

```
/brief-setup
```

That's it. Claude walks you through the setup in chat — no file editing needed. Pick a role (Sales / CS / Product / Engineering / Custom), paste a few Slack IDs, choose a poll interval. Takes about 5 minutes if your MCPs are already connected.

<details>
<summary>Alternative: install from inside a Claude Code session</summary>

If you're already in a Claude Code REPL and prefer slash commands:

```
/plugin marketplace add williamhscholl/daily-workflow-briefs
/plugin install daily-workflow-briefs@daily-workflow-briefs
```

Equivalent to the shell command above.
</details>

---

## What you need before running setup

These MCPs must be connected to your Claude Code:

**Required** (briefs won't work without these):
- Slack
- Google Calendar
- Gmail
- Zoom

**Optional** (enable during setup):
- Atlassian (Jira / Confluence)
- HubSpot

See [docs/integrations.md](docs/integrations.md) for MCP setup instructions.

---

## The approval loop (the thing that makes this different)

Every brief ends with a thread reply like this:

```
🤝 Work I can do for you — reply here to approve

1. Update Confluence "Q2 Roadmap" — append calendar decision context
2. Add comment to PUL-244 — status-check note to Sebastian
3. Draft Slack reply to @CEO on forecast question

Reply with `apply 1` / `skip 2` / `show 3` (to preview) / `edit 1: <new text>`.
I'll pick up your reply on the next poll.
⚡ Need it sooner? Reply, then run `/brief-run poll` in Claude Code.
```

You reply in Slack (on your phone, from bed, wherever). The next poll reads your reply and executes approved items. If you want it to happen *right now*, open Claude Code and run `/brief-run poll` — the poll runs immediately and processes your reply within seconds.

**The allowlist** — actions this plugin can take on your behalf:
- ✅ Edit / append to a Confluence page (with diff preview)
- ✅ Add a comment to a Jira ticket (with preview)
- ✅ Draft a Slack reply — drafted only, never auto-sent
- ✅ Add / update / mark-done a task in your tasks.md
- ✅ Add a read-only note to a HubSpot deal
- ❌ Send email
- ❌ Change Jira status / assignee / priority
- ❌ Change HubSpot deal stage / amount / owner
- ❌ Delete anything

Every execution shows the exact text/diff that was applied, so you can catch mistakes immediately.

---

## Daily flow

| Time | What happens |
|------|--------------|
| 7:20am Mon–Fri | Morning brief posts to your self-DM |
| +30 seconds | Work offers post as the first thread reply |
| Anytime morning | You reply `apply 1` / etc. in thread |
| Every Xh (default 1h) | Poll runs, executes your approvals, scans new signals, posts any new offers |
| 3:30pm Mon–Fri | EOD brief posts; fresh offers as first reply |
| Whenever | You can trigger manually: `/brief-run morning`, `/brief-run poll`, `/brief-run eod` |

---

## Roles

`/brief-setup` pre-fills defaults based on your role. Currently supported:

- [Sales](roles/sales.md) — HubSpot-centric, deal-stage signal, customer call next-steps
- [Customer Success](roles/cs.md) — Jira bug-ticket flow, escalations, customer account signal
- [Product](roles/product.md) — Jira-heavy, PRD/Confluence editing, PM team coordination
- [Engineering](roles/engineering.md) — Jira + Slack PR notifications, incident signal
- [Custom / Other](roles/_blank.md) — blank slate, you define everything

Each role file documents what signals get prioritized, typical work offers, and cautions. Read yours before running setup.

---

## Where files live

- **Config**: `~/.claude/daily-workflow-briefs/config.md` — human-readable, editable any time
- **Tasks**: `~/.claude/daily-workflow-briefs/tasks.md` (default; configurable)
- **Ledger**: `~/.claude/daily-workflow-briefs/.offers.jsonl` — tracks which offers were applied/skipped (internal)
- **Run log**: `~/.claude/daily-workflow-briefs/.poll-log.jsonl` — one line per poll run (internal)

---

## Uninstall

```
/plugin uninstall daily-workflow-briefs
```

The three scheduled cron tasks will stop firing. Config and tasks.md stay in `~/.claude/daily-workflow-briefs/` — delete that directory manually if you want a clean wipe.

---

## Customization

- Edit `~/.claude/daily-workflow-briefs/config.md` directly for quick changes (add a channel, add a team member).
- Re-run `/brief-setup` for a guided re-configure.
- See [docs/customization.md](docs/customization.md) for advanced tweaks (custom cron, custom poll intervals, skipping specific signal types).

---

## FAQ

**Does this send email / DMs automatically?**
No. The plugin never sends anything outside your self-DM. Slack replies are drafted only — you copy/paste to send.

**What if I approve an offer by mistake?**
Every execution is logged with a timestamp and the exact text applied, and the confirmation shows up in-thread so you see it immediately. To undo: revert manually (edit the Confluence page back, delete the Jira comment). The plugin doesn't keep a "revert" command in v1.

**Can I use this if I don't want the task-tracking part?**
Yes. In `/brief-setup`, skip the tasks.md question and set `tasks_file: none`. The overdue-tasks section will be omitted from briefs.

**What's the token cost?**
Roughly $0.15–$0.60/day on Sonnet depending on your poll interval. Morning + EOD are fixed cost (~5k tokens each); poll cost scales with interval.

**Does this work with Codex / Gemini / other agents?**
Not yet. v1 is Claude Code only.

---

## Contributing / issues

File issues at https://github.com/williamhscholl/daily-workflow-briefs/issues.

## License

MIT
