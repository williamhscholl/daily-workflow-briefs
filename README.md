# Daily Workflow Briefs

**A Claude Code plugin that runs your day via Slack.** Three automated briefs post to your Slack self-DM:

- 🌤 **Morning brief** — today's meetings, overdue work, meeting next-steps, top-3 decisions
- 🔄 **Throughout-the-day poll** — processes your thread replies, applies approved work, surfaces new signals
- 🌇 **EOD brief** — meetings attended, tomorrow's top-3 focus

You pick the exact times (and timezone) during setup. Defaults are 7:20am / 3:30pm weekdays, poll every hour.

Plus a `🤝 Work I can do for you` section that shows up as the first reply to each brief. You approve with `apply 1` / `skip 2` / `edit 1: …` and the plugin executes on the next poll (or immediately if you trigger a manual run).

---

## Quick start

Three steps. Allow 10 minutes. You need a terminal open for step 1; everything else happens inside Claude Code.

### Step 1 — Install Claude Code, then the plugin

**If you don't have Claude Code installed yet:**

Mac / Linux (open Terminal):
```bash
curl -fsSL https://claude.ai/install.sh | bash
```

Windows (open **PowerShell**, not Git Bash):
```powershell
irm https://claude.ai/install.ps1 | iex
```

> ⚠ Claude Code (the CLI) is a **separate product from the Claude Desktop chat app**. Having the chat app installed is not enough.

After install, close and reopen your terminal. Verify:
```bash
claude --version
```
You should see a version number like `2.1.119`. If you see `command not found`, follow the path-fix hint the installer printed (usually `echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc`).

**Install the plugin** (one line):
```bash
claude plugin marketplace add williamhscholl/daily-workflow-briefs && claude plugin install daily-workflow-briefs@daily-workflow-briefs
```

On Windows PowerShell, run them as two separate lines:
```powershell
claude plugin marketplace add williamhscholl/daily-workflow-briefs
claude plugin install daily-workflow-briefs@daily-workflow-briefs
```

### Step 2 — Connect your MCPs

The briefs read from these tools via Claude Code MCPs:

**Required:** Slack, Google Calendar, Gmail, Zoom
**Optional:** Atlassian (Jira + Confluence), HubSpot

If you don't have them connected already, open the **Claude Code app** (the desktop app — download from [claude.com](https://claude.com) if you don't have it) and connect each one via Settings → MCP Servers. See [docs/integrations.md](docs/integrations.md) for details.

### Step 3 — Run setup

Open the Claude Code desktop app. In any chat window, type:

```
/daily-workflow-briefs:brief-setup
```

Claude will walk you through configuration in chat:
- Pick a role preset (Sales / CS / Product / Engineering / Custom)
- Enter your Slack user ID + self-DM channel (help text tells you where to find them)
- Paste your team members' Slack IDs
- List your VIPs (emails)
- Choose your brief times and timezone
- Pick a poll interval
- Confirm

You can also use this same command from a terminal by running `claude` first. Either environment works — desktop app is easier if you're not a terminal person.

### Done

Briefs start firing on the schedule you picked. First one lands on the next weekday at your chosen morning time.

---

## Commands reference

Every command is namespaced under the plugin name. **All commands need the `daily-workflow-briefs:` prefix.**

| Command | What it does |
|---------|--------------|
| `/daily-workflow-briefs:brief-setup` | Interactive setup wizard. Run first. Also use to re-configure. |
| `/daily-workflow-briefs:brief-run morning` | Run the morning brief now (useful for testing) |
| `/daily-workflow-briefs:brief-run eod` | Run the EOD brief now |
| `/daily-workflow-briefs:brief-run poll` | Run the poll now — **use this after approving offers in Slack** to apply them instantly instead of waiting for the scheduled poll |
| `/daily-workflow-briefs:brief-help` | Show your current config + scheduled times + usage tips |
| `/daily-workflow-briefs:brief-config` | Edit a single field without re-running the full wizard (e.g. "change my morning time to 8am", "add #new-channel to my watch list") |

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
⚡ Need it sooner? Reply, then open Claude Code and run /daily-workflow-briefs:brief-run poll
```

You reply in Slack (on your phone, from bed, wherever). The next poll reads your reply and executes approved items. If you want it to happen *right now*, open Claude Code and run `/daily-workflow-briefs:brief-run poll` — the poll runs immediately and processes your reply within seconds.

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

## Roles

`/daily-workflow-briefs:brief-setup` pre-fills defaults based on your role:

- [Sales](roles/sales.md) — HubSpot-centric, deal-stage signal, customer call next-steps
- [Customer Success](roles/cs.md) — Jira bug-ticket flow, escalations, customer account signal
- [Product](roles/product.md) — Jira-heavy, PRD/Confluence editing, PM team coordination
- [Engineering](roles/engineering.md) — Jira + Slack PR notifications, incident signal
- [Custom / Other](roles/_blank.md) — blank slate, you define everything

Each role file documents what signals get prioritized, typical work offers, and cautions. Read yours before running setup.

---

## Updating the plugin

When new features ship to this repo, update with:

```bash
claude plugin marketplace update daily-workflow-briefs
claude plugin update daily-workflow-briefs@daily-workflow-briefs
```

Your config and tasks.md are preserved — only the plugin logic refreshes.

---

## Where files live

- **Config**: `~/.claude/daily-workflow-briefs/config.md` — human-readable, editable any time
- **Tasks**: `~/.claude/daily-workflow-briefs/tasks.md` (default; configurable)
- **Ledger**: `~/.claude/daily-workflow-briefs/.offers.jsonl` — tracks which offers were applied/skipped
- **Run log**: `~/.claude/daily-workflow-briefs/.poll-log.jsonl` — one line per poll run

---

## Uninstall

From any terminal:
```bash
claude plugin uninstall daily-workflow-briefs
```

The three scheduled cron tasks will stop firing. Config and tasks.md stay in `~/.claude/daily-workflow-briefs/` — delete that directory manually if you want a clean wipe.

---

## FAQ

**Does this send email / DMs automatically?**
No. The plugin never sends anything outside your self-DM. Slack replies are drafted only — you copy/paste to send.

**What if I approve an offer by mistake?**
Every execution is logged with a timestamp and the exact text applied, and the confirmation shows up in-thread so you see it immediately. To undo: revert manually (edit the Confluence page back, delete the Jira comment). The plugin doesn't keep a "revert" command in v1.

**Can I use this if I don't want the task-tracking part?**
Yes. In `/daily-workflow-briefs:brief-setup`, skip the tasks.md question and set `tasks_file: none`. The overdue-tasks section will be omitted from briefs.

**What's the token cost?**
Roughly $0.15–$0.60/day on Sonnet depending on your poll interval. Morning + EOD are fixed cost (~5k tokens each); poll cost scales with interval.

**Does this work with Codex / Gemini / other agents?**
Not yet. v1 is Claude Code only.

**Why do slash commands need the `daily-workflow-briefs:` prefix?**
That's how Claude Code namespaces plugin commands — it prevents collisions between plugins. Once you type `/daily` autocomplete usually kicks in, so it's less typing than it looks.

---

## Contributing / issues

File issues at https://github.com/williamhscholl/daily-workflow-briefs/issues.

## License

MIT
