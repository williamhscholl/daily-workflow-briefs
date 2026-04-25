# Changelog

All notable changes to this plugin are documented here. Newest version first.

To preview what changed before updating, read the entry below for the version you're about to install. To compare two specific versions, visit:

> `https://github.com/williamhscholl/daily-workflow-briefs/compare/<from-version>...<to-version>`

This file follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) conventions.

---

## [v0.6.0] — 2026-04-24

### Added
- **`tasks` skill** — auto-fires in Claude Code when the user asks about their work. Reads tasks.md and answers conversationally. Handles:
  - Priority queries: "What should I prioritize today?" / "What do I need to do next?"
  - Status queries: "What's overdue?" / "What did I get done this week?" / "What's on my plate this week?"
  - Goal queries: "Show me everything for [goal]" / "How's my workload?"
  - Owner queries: "What's [name] working on?" (filter by Owner field)
  - Lookups: "Find the task about [topic]"
  - Modifications: "Mark X done" / "Add a task to [goal]: …" / "Move X to Friday" / "Bump Y to critical" — same patterns the brief poll uses for Slack thread replies, now usable directly in Claude Code chat.
- README now lists task-query examples in a dedicated "Ask about your tasks" section.
- `/briefs:help` includes task-query examples in its "What you can do" output.

### Why
Users were already asking task questions naturally in Claude Code, but without a dedicated skill, Claude had to guess where the tasks file lived and how to interpret the question. The new skill triggers reliably on task-related natural language and answers focused on the user's actual work — same level of polish as the morning brief's Top 3 Decisions section, but on-demand.

### Migration from v0.5.x
No action required — additive feature. The skill auto-activates when the user asks task-related questions in Claude Code. No config changes, no slash commands to learn.

---

## [v0.5.0] — 2026-04-24

### Added
- **Official MCPs documented for Granola, Otter, Fireflies, and Fathom.** Setup wizard now defaults each one to MCP source (preferred) and asks whether the user has the MCP installed before falling back to Gmail.
- MCP install links surface inline during setup:
  - Granola: https://www.granola.ai/blog/granola-mcp
  - Otter: https://help.otter.ai/hc/en-us/articles/35287607569687-Otter-MCP-Server
  - Fireflies: https://docs.fireflies.ai/getting-started/mcp-configuration
  - Fathom (via Composio): https://composio.dev/toolkits/fathom
- Loom remains Gmail-only (no public MCP available).
- Zoom now has a documented Gmail fallback pattern (`from:no-reply@zoom.us subject:"Meeting assets" newer_than:1d`) for users who can't or don't want to connect the Zoom MCP — though Zoom MCP is still strongly recommended for fidelity.

### Changed
- Setup wizard's transcriber step asks "Do you have the [Tool] MCP installed?" before assuming Gmail. Users who say yes get the MCP path; users who say no get the install link plus an offer to use Gmail fallback in the meantime.
- `docs/integrations.md` now has a unified table for all transcribers showing MCP availability, install links, and Gmail fallback patterns.

### Migration from v0.4.x
No action required. Existing configs continue to work exactly as before. To switch a transcriber from Gmail-fallback to MCP after installing the MCP: `/briefs:config switch transcriber source to mcp`.

---

## [v0.4.0] — 2026-04-24

### Added
- **Meeting transcriber as a first-class config field.** During setup, the wizard now asks "What do you use to transcribe and summarize meetings?" with Zoom as default (option 1) and Granola, Otter, Fireflies, Fathom, Loom as alternatives. Each option auto-configures the correct source (Zoom MCP vs Gmail search) and email pattern.
- New `meeting_transcriber` block in config with `type`, `source`, and optional `gmail_query` fields.
- `type: none` option for users who don't use a transcriber — skips meeting next-steps in briefs entirely without breaking other signal-gathering.
- `type: custom` option for in-house or unsupported transcribers.

### Changed
- Zoom is no longer auto-required. The Zoom MCP is only required if the user picks Zoom as their transcriber. Slack, Calendar, and Gmail remain required for everyone.
- Brief skills (morning, EOD, poll) now read `meeting_transcriber` and branch on `type`. Zoom-specific code path is preserved (with all the `has_summary_permission` workarounds intact); Gmail-based transcribers get a separate parsing path that extracts recap + next-steps from email bodies.
- `/briefs:config change meeting transcriber to <name>` for natural-language updates.

### Migration from v0.3.x
No action required if you use Zoom — `meeting_transcriber` defaults to Zoom-MCP behavior when the field is absent.

To switch transcribers later: run `/briefs:config change meeting transcriber to granola` (or any other supported tool).

---

## [v0.3.0] — 2026-04-24

### Added
- **Gmail-fallback for additional integrations.** Tools that don't have a Claude Code MCP but email you summaries (Granola, Otter, Fireflies, Fathom, Loom) can now be added as integrations using a Gmail search filter. The brief skills search Gmail for matching emails and extract content (recap + next steps for meeting transcribers).
- New config format for additional integrations supports `source: mcp` (preferred) or `source: gmail` (with `gmail_query` field). Legacy `name: description` entries continue to work as MCP-source.
- `/briefs:setup` Step 8 now asks which path applies for each integration (MCP vs Gmail) and helps build the Gmail filter using known patterns for common tools.
- `/briefs:config` accepts `add <tool> via gmail: <pattern>` and `switch <tool> to gmail/mcp` for source changes.
- Documentation: example Gmail patterns table for common tools in [docs/integrations.md](docs/integrations.md#path-b-gmail-when-no-mcp-exists).

### Changed
- Brief skills (morning, EOD, poll) now branch on `source` for additional integrations and try both paths cleanly.

### Migration from v0.2.x
No action required. Existing additional-integration entries (`name: description` form) still work — they're treated as `source: mcp` implicitly. To add Gmail-backed tools, use `/briefs:config add granola via gmail: from:noreply@granola.ai newer_than:1d` or re-run `/briefs:setup`.

---

## [v0.2.0] — 2026-04-24

**Migration required from v0.1.x.** Plugin renamed from `daily-workflow-briefs` to `briefs`. See [README → Updating](README.md#updating-the-plugin) for the migration command.

### Added
- **Additional integrations**: register any MCP you have connected to Claude Code (Salesforce, Intercom, Zendesk, Linear, GitHub, Notion, Asana, etc.) during setup with a description of what to check. Brief skills surface read-only signals from each. Read-only — no work offers from these tools in v1.
- **Work-offer preview toggle**: choose `summary` (one-line offers, type `show 1` to expand) or `full` (full preview text inline). Pick during setup, change later via `/briefs:config`.
- **Friday EOD weekend send-off**: when EOD runs on a Friday in your timezone, "Tomorrow's top 3" becomes "Monday's top 3" and a brief weekend send-off line is appended.
- **`/briefs:help`**: shows your current config, schedule, recent run stats, and usage tips. Plain English — no JSON.
- **`/briefs:config`**: edit a single config field via natural language ("change my morning to 8am", "add Sid to team", "switch preview to full") without re-running the full setup wizard.

### Changed
- **Plugin renamed**: `daily-workflow-briefs` → `briefs`. Slash commands now `/briefs:setup`, `/briefs:run`, `/briefs:help`, `/briefs:config` (saves ~20 chars per invocation).
- **Default brief times**: morning at 7:30am (was 7:20am), EOD still 3:30pm.
- **Removed "manager" framing**: descriptions and role files now read as "anyone in this role" rather than "managers only".
- **Anonymized example data**: README, role files, and docs now use generic example names/IDs (Sid, Riya, Sam, CLA-718, Product Strategy) instead of the author's real workplace data.

### Removed
- **EOD work-offers**: EOD posts at 3:30pm but the poll runs 8am–4pm — any EOD offer would sit unactioned overnight. Action-required afternoon items now surface in next morning's offers instead. EOD focuses on summary + Tomorrow's Top 3 only.

### Migration steps for v0.1.x users
```bash
claude plugin uninstall daily-workflow-briefs
claude plugin marketplace remove daily-workflow-briefs
claude plugin marketplace add williamhscholl/daily-workflow-briefs
claude plugin install briefs@briefs
```
Your config and tasks.md are preserved (they live at `~/.claude/daily-workflow-briefs/...` — internal directory unchanged). After install, run `/briefs:config add Salesforce as integration: <description>` to wire up additional integrations, or `/briefs:config switch preview to summary` to set your preview detail.

---

## [v0.1.0] — 2026-04-24

Initial release.

### Added
- Three Claude Code skills: `morning-brief`, `eod-brief`, `brief-poll`.
- Three slash commands: `/daily-workflow-briefs:brief-setup`, `/daily-workflow-briefs:brief-run`, `/daily-workflow-briefs:brief-config`, `/daily-workflow-briefs:brief-help`.
- Five role presets: Sales, Customer Success, Product, Engineering, Custom.
- Work-offer approval loop: morning + poll briefs end with a thread reply listing offers; user approves with `apply N` / `skip N` / `edit N: …` in Slack; poll executes on next run.
- Allowlist of safe actions: Confluence edits, Jira comments, Slack drafts (never auto-sent), tasks.md writes, HubSpot read-only notes.
- Hard guardrails: never sends email, never changes Jira status/assignee/priority, never changes HubSpot deal stage/amount/owner/close-date, never deletes anything.
- Hard-won MCP quirk handling baked in: Zoom `has_summary_permission` red-herring, Slack `oldest` parameter broken (filter client-side), JQL reserved-word quoting, Gmail body-drop on Zoom emails.
