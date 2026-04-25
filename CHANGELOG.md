# Changelog

All notable changes to this plugin are documented here. Newest version first.

To preview what changed before updating, read the entry below for the version you're about to install. To compare two specific versions, visit:

> `https://github.com/williamhscholl/daily-workflow-briefs/compare/<from-version>...<to-version>`

This file follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) conventions.

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
