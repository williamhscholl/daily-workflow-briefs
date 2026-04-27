# Daily Workflow Briefs

**A Claude Code plugin that runs your day via Slack.** Three automated briefs post to your Slack self-DM:

- 🌤 **Morning brief** — today's meetings, overdue work, meeting next-steps, top-3 decisions
- 🔄 **Throughout-the-day poll** — processes your thread replies, applies approved work, surfaces new signals
- 🌇 **EOD brief** — meetings attended, tomorrow's top-3 focus (or Monday's, with a weekend send-off if it's Friday)

You pick the exact times and timezone during setup (defaults: 7:30am / 3:30pm weekdays, poll every hour).

Plus a `🤝 Work I can do for you` section that shows up as the first reply to the morning brief and each poll. You approve with `apply 1` / `skip 2` / `edit 1: …` and the plugin executes on the next poll (or instantly if you trigger one manually).

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
You should see a version number (e.g. `2.1.119`). If you see `command not found`, follow the path-fix hint the installer printed (usually `echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc`).

**Install the plugin** (one line, from any terminal):
```bash
claude plugin marketplace add williamhscholl/daily-workflow-briefs && claude plugin install briefs@briefs
```

On Windows PowerShell, run as two separate lines (`&&` requires PowerShell 7+):
```powershell
claude plugin marketplace add williamhscholl/daily-workflow-briefs
claude plugin install briefs@briefs
```

### Step 2 — Connect your MCPs

The briefs read from these tools via Claude Code MCPs:

**Required:** Slack, Google Calendar, Gmail
**Meeting transcriber** (you pick one during setup): Zoom (default — uses Zoom MCP), or Granola / Otter / Fireflies / Fathom / Loom (uses Gmail to read their emailed recaps), or None
**Optional:** Atlassian (Jira + Confluence), HubSpot
**Anything else:** Salesforce, Intercom, Zendesk, Linear, GitHub, Notion, Asana — any MCP you can connect to Claude Code can be added as a read-only signal source during setup. Or, for tools that don't have an MCP but email you summaries, the plugin can watch Gmail for those emails instead. See [docs/integrations.md](docs/integrations.md#additional-integrations-any-tool).

If you don't have them connected already, open the **Claude Code app** (the desktop app — download from [claude.com](https://claude.com) if you don't have it) and connect each via Settings → MCP Servers. See [docs/integrations.md](docs/integrations.md) for details.

### Step 3 — Run setup

Open the Claude Code desktop app. In any chat, type:

```
/briefs:setup
```

Claude walks you through configuration in chat:
- Pick a role preset (Sales / CS / Product / Engineering / Custom)
- Enter your Slack user ID + self-DM channel (help text tells you where to find them)
- Paste your team members' Slack IDs
- List your VIPs (emails)
- **Pick your meeting transcriber** (Zoom default; or Granola / Otter / Fireflies / Fathom / Loom; or None)
- **List any other tools you want briefs to scan** (Salesforce, Intercom, Zendesk, etc.)
- Pick your morning + EOD times and timezone
- Pick a poll interval
- Pick work-offer preview detail (summary vs full)
- Confirm

You can also type `claude` in a terminal to start a CLI session and run the same command. Either environment works.

### Done

Briefs start firing on your schedule. First one lands the next weekday at your chosen morning time.

---

## Commands reference

Every plugin command is namespaced under `briefs:`. Slash commands and plain English both work — use whichever feels natural.

| Command | Plain language equivalent | What it does |
|---------|--------------------------|--------------|
| `/briefs:setup` | "set up my briefs" | Interactive setup wizard. Run first. Also use to re-configure. |
| `/briefs:run morning` | "run my morning brief" | Run the morning brief now (useful for testing) |
| `/briefs:run eod` | "run my EOD brief" | Run the EOD brief now |
| `/briefs:run poll` | "run my poll now" / "process my Slack replies" | Run the poll now — **use this after approving offers in Slack** to apply them instantly instead of waiting for the scheduled poll |
| `/briefs:help` | "what's in my brief config?" / "how do I use this?" | Show your current config + scheduled times + recent run stats + usage tips |
| `/briefs:config` | "change my morning to 8am" / "add Sid to my team" | Edit a single config field via chat |

You can also make config changes without a slash command at all:
- "change my poll interval to 90 minutes"
- "add Zendesk as an integration: tickets I'm watching"
- "switch my meeting transcriber to Granola"
- "add Riya to my team"

### Ask about your tasks (any time, in Claude Code)

The plugin includes a `tasks` skill that auto-fires when you ask about your work. No slash command — just type the question:

| You say | What you get |
|---------|-------------|
| "What should I prioritize today?" | Top 3 by deadline + who's blocked on you |
| "What's overdue?" | All overdue tasks grouped by goal |
| "What do I need to do next?" | Next 3 by priority |
| "What's on my plate this week?" | Tasks due in the next 7 days |
| "What did I get done this week?" | Completed tasks since Monday |
| "Show me everything for [goal name]" | All open + completed tasks for that goal |
| "How's my workload?" | Snapshot: counts by priority, busiest goals, overdue |
| "What's [name] working on?" | Tasks filtered by owner — great for 1:1 prep |
| "Find the task about [topic]" | Full-text search across tasks + notes |

You can also modify tasks in plain English: "mark X done", "add a task to [goal]: …", "move X to Friday", "bump Y to critical". Same patterns the brief poll uses for Slack thread replies — they work in Claude Code too.

---

## The approval loop (the thing that makes this different)

The morning brief and each throughout-the-day poll end with a thread reply like this:

```
🤝 Work I can do for you — reply here to approve

1. Update Confluence "Product Strategy" — append decision context from yesterday's call
2. Add comment to CLA-718 — status-check note to Sid
3. Draft Slack reply to your CEO on forecast question

Reply with apply 1 / skip 2 / show 3 (to preview) / edit 1: <new text>.
I'll pick up your reply on the next poll.
⚡ Need it sooner? Reply, then run /briefs:run poll in Claude Code.
```

You reply in Slack (on your phone, from bed, wherever). The next poll reads your reply and executes approved items. If you want it to happen *right now*, open Claude Code and run `/briefs:run poll` — the poll runs immediately and processes your reply within seconds.

> Note: the EOD brief does NOT post work offers. The poll only runs 8am–4pm, so any EOD offer would sit unactioned overnight. Action-required items from afternoon meetings show up in next morning's offers instead.

**The allowlist** — actions this plugin can take on your behalf:
- ✅ Edit / append to a Confluence page (with diff preview)
- ✅ Add a comment to a Jira ticket (with preview)
- ✅ Draft a Slack reply — drafted only, never auto-sent
- ✅ Add / update / mark-done a task in your tasks.md
- ✅ Add a read-only note to a HubSpot deal
- ❌ Send email
- ❌ Change Jira status / assignee / priority
- ❌ Change HubSpot deal stage / amount / owner
- ❌ Anything destructive — delete, close, archive
- ❌ Write actions on additional integrations (Salesforce, Intercom, etc.) — read-only in v1

Every execution shows the exact text/diff that was applied, so you can catch mistakes immediately.

**Preview detail:** during setup you pick `summary` (one-line offers, type `show 1` in Slack to expand the full text) or `full` (full preview inline in every offer message). Change later with `/briefs:config switch preview to full`.

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

## Updating the plugin

**Updates are 100% manual — nothing auto-updates.** You stay on whatever version you installed until you explicitly run an update command. If you never want to update, you never have to.

### Preview what's in a new version BEFORE updating

1. Open the [CHANGELOG.md](CHANGELOG.md) — every version's changes are documented there, newest first.
2. Or compare your installed version to `main` directly:
   ```
   https://github.com/williamhscholl/daily-workflow-briefs/compare/v0.2.0...main
   ```
   Replace `v0.2.0` with whatever version you currently have. GitHub will show every line that changed.
3. Or browse the commits at [github.com/williamhscholl/daily-workflow-briefs/commits/main](https://github.com/williamhscholl/daily-workflow-briefs/commits/main). Each commit message describes the change.

If you don't like what you see, just don't update. Your installed version keeps working forever.

### Update commands

When you decide to update:

```bash
claude plugin marketplace update briefs
claude plugin update briefs@briefs
```

Your config and tasks.md are preserved — only the plugin logic refreshes. After updating, check [CHANGELOG.md](CHANGELOG.md) for any migration steps the new version requires.

### If a new version breaks something for you

Roll back by reinstalling the previous tagged release:
```bash
claude plugin uninstall briefs
claude plugin marketplace remove briefs
git clone --branch v0.2.0 https://github.com/williamhscholl/daily-workflow-briefs ~/briefs-v0.2.0
claude plugin marketplace add ~/briefs-v0.2.0
claude plugin install briefs@briefs
```

(Replace `v0.2.0` with whichever earlier version you want.) Then file an issue at [github.com/williamhscholl/daily-workflow-briefs/issues](https://github.com/williamhscholl/daily-workflow-briefs/issues) describing what broke.

---

## Privacy & data

**Everything you care about stays on your machine.** This plugin doesn't phone home, doesn't send telemetry, doesn't log to any external service. The plugin author (and anyone else) cannot see your data.

### What stays local
- **Your config**: `~/.claude/daily-workflow-briefs/config.md` — Slack IDs, team list, VIP emails, integration settings.
- **Your tasks.md**: same directory. Never leaves your machine unless you put it on a synced drive yourself.
- **The offers ledger and poll log**: same directory. Internal bookkeeping only.

### What flows through your own connections
- **Slack content** flows through your own Slack OAuth token via the Slack MCP you connected to Claude Code. The plugin author has no access to it.
- **Gmail / Google Calendar / Zoom / Jira / HubSpot / additional integrations**: same — your own OAuth tokens, your own MCP connections, your own data.
- **Briefs post to your Slack self-DM only** (a DM with yourself — no one else can see it).

### What goes to Anthropic's API
This is a Claude Code plugin, so when a brief runs, the signals it gathers (your Slack messages, Gmail threads, Zoom summaries, etc.) get sent to Anthropic's API for Claude to reason over. That's how Claude Code works — the same data path applies to any Claude Code session you've ever run. [Anthropic's data-handling policy](https://www.anthropic.com/legal/privacy) covers this. The plugin doesn't send anything *additional* to Anthropic beyond what Claude Code itself would send.

### Open-source trust model
Like any open-source plugin, when you install or update, you pull and execute code from this GitHub repo. If the author (or someone with write access) ever pushed something malicious, an `update` command would propagate it to your machine. This is the standard trust model for `npm`, `pip`, GitHub Actions, etc. Mitigations:
- Updates are manual (see above) — you choose when, after reading the changelog.
- The plugin code is text-only (markdown skill specs and a JSON manifest) — no compiled binaries, no shell scripts. Easy to audit by skimming the files.
- You can pin to a tagged version and never update if you want.

If you have specific compliance requirements (SOC 2, HIPAA, GDPR data residency rules), this plugin doesn't add anything beyond what Claude Code itself does — talk to your security team about whether Claude Code is approved in your environment first.

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
claude plugin uninstall briefs
```

The three scheduled cron tasks will stop firing. Config and tasks.md stay in `~/.claude/daily-workflow-briefs/` — delete that directory manually if you want a clean wipe.

---

## FAQ

**Does this send email / DMs automatically?**
No. The plugin never sends anything outside your self-DM. Slack replies are drafted only — you copy/paste to send.

**What if I approve an offer by mistake?**
Every execution is logged with a timestamp and the exact text applied, and the confirmation shows up in-thread so you see it immediately. To undo: revert manually (edit the Confluence page back, delete the Jira comment). The plugin doesn't keep a "revert" command in v1.

**Can I use this if I don't want the task-tracking part?**
Yes. In `/briefs:setup`, set tasks file to `none`. The overdue-tasks section will be omitted from briefs.

**What's the token cost?**
Roughly $0.15–$0.60/day on Sonnet depending on your poll interval. Morning + EOD are fixed cost (~5k tokens each); poll cost scales with interval.

**Can I add tools beyond the built-in MCPs (Salesforce, Intercom, Zendesk, etc.)?**
Yes. Two paths: (a) if you have a Claude Code MCP for the tool, the plugin calls it directly. (b) If no MCP exists but the tool emails you summaries (e.g. Granola, Otter, Fireflies, Fathom for meeting transcripts), the plugin searches Gmail for those emails using a pattern you provide. Both paths are read-only — no work offers from additional integrations in v1. See [docs/integrations.md](docs/integrations.md#additional-integrations-any-tool) for details and example patterns.

**Does this work with Codex / Gemini / other agents?**
Not yet. v1 is Claude Code only.

---

## Contributing / issues

File issues at https://github.com/williamhscholl/daily-workflow-briefs/issues.

## License

MIT
