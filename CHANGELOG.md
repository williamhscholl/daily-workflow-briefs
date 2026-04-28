# Changelog

All notable changes to this plugin are documented here. Newest version first.

To preview what changed before updating, read the entry below for the version you're about to install. To compare two specific versions, visit:

> `https://github.com/williamhscholl/daily-workflow-briefs/compare/<from-version>...<to-version>`

This file follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) conventions.

---

## [v0.7.5] — 2026-04-28

### Added
- **Slack-tagging flow for setup wizard.** Setup Steps 4 and 5 (watch channels + DM-scan team members) collapse into a single new Step 4 that uses Slack's autocomplete for both. The wizard posts a DM to your self-DM channel asking you to reply with `@-mentions` of teammates and `#-mentions` of channels — mixed however you want. You then return to Claude Code, type `done`, and the wizard parses the resolved IDs out of the Slack message format (`<@U...>` for users, `<#C...|name>` for channels). Channel display names are extracted automatically; teammate names are fetched via `slack_read_user_profile`. Eliminates 5–15 manual ID lookups for a typical user.

### Why
The old flow was the highest-friction part of setup. Users had to click each channel → "Copy link" → grab the `C...` part of the URL → paste back into Claude → repeat ~6 times → then do the same dance for teammate user IDs. Slack already resolves `@-mentions` and `#-mentions` to fully-qualified IDs in its message format — the plugin just had to ask for them in Slack instead of asking the user to manually look them up.

### Edge cases handled
- **Deactivated users** — `slack_read_user_profile` 404s; logged and skipped.
- **External-shared channels** — parse normally, kept.
- **Bot users** — flagged for user confirmation before keeping (default no).
- **Timeout** — 3-minute wait between DM post and `done`; wizard prompts "still there?" if no reply.
- **Empty reply** — wizard catches and asks user to reply in the thread first.
- **Slack MCP unavailable** — automatic fallback to manual paste flow (Path b).

### Changed
- Step renumbering: old Steps 6–18 are now 5–17 (channels + team merged into Step 4). Body cross-references updated:
  - Step 6 (Built-in integrations) "verify in Step 12" → "Step 13" (was a pre-existing bug; Step 12 was Work-offer preview, not MCP check)
  - Step 7 (Meeting transcriber) "verify in Step 14" → "Step 13"
  - Step 13 (MCP connection check) "any from Step 9" → "Step 8"

### Migration from v0.7.4
- New users see the Slack-tagging flow during `/briefs:setup`.
- Existing users — your config is already written, no action needed. To take advantage of the new flow on a re-setup, run `/briefs:setup` (overwrites config) and pick path (a) when offered.

---

## [v0.7.4] — 2026-04-28

### Fixed (real bugs surfaced from setup-wizard cold-test)
- **EOD default time is now derived from morning brief time, not hardcoded to 3:30pm.** Old behavior: every user got `default 3:30pm` regardless of when their morning brief fires. So a user setting morning=11:30am got an EOD suggestion of 3:30pm — only 4 hours later, missing the back half of their work day. New behavior: suggested EOD = `morning_time + 8 hours` (capped at 22:00). User can still type a custom override. The wizard also rejects EOD times within an hour of morning ("only 30 min between them — pick a later time").
- **Poll cron window is now derived from morning + EOD times, not hardcoded to 8am–3pm.** Old behavior: every user's poll ran `8-15` regardless of their actual brief schedule. So a user with morning at 11:30am and EOD at 7:30pm got a poll window that ended *4 hours before their first brief landed* and never overlapped with their actual work day. New behavior: poll cron computes `start_hour` = first whole hour after morning, `end_hour` = last whole hour before EOD. Edge case (morning too close to EOD) falls back to inclusive window with a warning.

### Why these slipped through
Both defaults were copy-pasted from the original 7:30am/3:30pm reference user (the author). v0.1.0 shipped without anyone testing the wizard against alternate schedules — when Will cold-tested with a Brazil-timezone account using an 11:30am morning, both bugs surfaced immediately.

### Migration from v0.7.3
- **New users** see the corrected defaults from setup.
- **Existing users** are unaffected — their cron is already written and live. To pick up the new defaults, re-run `/briefs:setup` (overwrites config) or edit `poll_interval_minutes` and the cron schedule via `mcp__scheduled-tasks` directly.

### Where the fix lives
[`commands/setup.md`](commands/setup.md) Step 10 (EOD time) and Step 16 (poll cron computation).

---

## [v0.7.3] — 2026-04-28

### Changed
- **README rewritten top-to-bottom for tighter, PM-style structure.** Reduced from 414 lines to 312 (~25%). Same content, less meandering.

### New top-of-README structure
1. **What it does** — two value-props lead, no preamble: (1) organizes and prioritizes your work from Slack/email/calendar/meeting notes/Jira, (2) updates work automatically throughout the day. Tagline: "No todo app to learn. No manually updating anything. No forgetting a new due date or a decision made in a meeting."
2. **What makes Briefs different** — two named differentiators with explicit headings:
   - *Briefs offers to do work for you* (the approval-loop pitch, surfaced upfront instead of buried halfway down)
   - *Work is organized into todo and monitoring* (the monitoring distinction, surfaced upfront instead of in a sub-section)
3. **How your work is organized** — collapsed three previous sub-sections (Goals, Tasks, MONITORING) into one paragraph and one example codeblock. Schema is still complete; verbosity is gone.
4. **How the pieces fit together** — diagram unchanged from v0.7.2.
5. **Quick start, Commands, Allowlist, Roles, Updating, Privacy, FAQ** — all kept, all tightened.

### What was cut
- Verbose explainer paragraphs that re-stated what the bullets and tables already said.
- Redundant "approval loop" section (now subsumed into the up-front differentiator + a leaner allowlist section).
- Step-by-step setup-wizard walkthrough (the wizard already prompts for these — duplicating the list in README adds no value).
- Privacy section consolidated from 4 sub-sections to 1 bullet list.
- Updating section consolidated from 3 sub-sections to 1 flow.

### What was kept verbatim
- The architecture diagram (added in v0.7.2)
- The `tasks.md` schema example
- The Commands table and the Ask-about-your-tasks query table
- The allowlist (✅ / ❌ table of permitted actions)

### Migration from v0.7.2
No action required — README-only docs change.

---

## [v0.7.2] — 2026-04-28

### Added
- **README "How the pieces fit together" architecture diagram.** New sub-section under "How your work is organized" with an ASCII flowchart showing the foundation layer (`config.md` → `tasks.md`) feeding three skill categories: BRIEFS (push state to Slack), SYNC (poll reads your replies, edits tasks.md), ON-DEMAND (the `tasks` chat skill). Followed by a tight three-paragraph explainer covering the implicit-vs-explicit reply-handling nuance and how slash commands fit in.

### Why
The README defined the data model (v0.6.2/v0.6.3) but didn't show the *flow* — readers had to infer the relationship between the four skills, the two foundation files, and the two surfaces (Slack vs. chat). The diagram makes the trust boundary obvious: file-based state is the single source of truth; skills are stateless workers reading/writing those files.

### Migration from v0.7.1
No action required — README-only docs change.

---

## [v0.7.1] — 2026-04-28

### Changed
- **Confirmation label**: `✅ Goals updated` → `✅ Tasks updated` in `brief-poll` skill output. The data model still has goals (parents) and tasks (children) — but the user-facing artifact people see updated *is* the task list. Confirmation messages now reflect that. Same change cascades through:
  - `skills/eod-brief/SKILL.md` — "New saves (not yet on the board)" → "New saves (not yet in tasks)"; dedup wording
  - `README.md` — example confirmation in the "How your work is organized" round-trip paragraph
- **Why this matters in practice:** The confirmation message used to be the *only* thing first-time users saw post-approval. "Board" was opaque (rejected in v0.6.2). "Goals updated" was technically correct but mismatched the unit of action — a single `apply 3` typically marks ONE task done, not a whole goal. "Tasks" matches what the user just did.

### Migration from v0.7.0
No action required. Next poll emits the new label automatically. Historical "Goals updated" / "Board updated" messages in your past brief threads stay as they were posted — the EOD brief's task-changes-today scanner already understands all three forms (Board / Goals / Tasks).

---

## [v0.7.0] — 2026-04-27

**Theme:** make MONITORING a first-class part of the daily decision-driving flow.

Until now, MONITORING was a documentation concept (added in v0.6.3) that the skills handled emergently — `[MONITORING]` tasks dropped out of priority queries only because they weren't tagged Critical/High/Medium/Low. v0.7.0 makes the contract explicit and surfaces what's owed to you alongside what you owe.

### Added
- **Tasks skill: explicit yours-to-do vs. yours-to-watch classification.** New "Step 1.5" in [`skills/tasks/SKILL.md`](skills/tasks/SKILL.md) defines the rule once and reuses it across queries:
  - **Yours to do** — task `Owner:` includes the user
  - **Yours to watch** — `Owner:` is someone else AND (parent goal is `## MONITORING:` OR priority is `[MONITORING]`)
  - The owner check trumps the goal-level marker — a `## MONITORING:` goal with `Owner: <you>` task is in your action list, not your watching list.
- **New `Monitoring / waiting on / owed to you` query type** in the tasks skill. Triggers: "what am I waiting on?", "what am I watching?", "what's owed to me [today/this week]?", "what does [name] owe me?", "show me monitoring", "monitoring". Output: grouped by owner, overdue first, then by earliest due date.
- **`/briefs:monitoring` slash command.** Thin wrapper over the new query type. Argument support: `<name>` (filter to that owner), `today`, `this week`, or empty (all).
- **Parallel sections in priority queries.** "What should I prioritize today?", "what's overdue?", and "what's on my plate this week?" now return TWO sections side-by-side:
  - 🎯 **Your work** — yours-to-do tasks
  - 👁 **Owed to you / watching** — yours-to-watch tasks
  - Either section is omitted silently if empty (no "nothing to watch" noise).
- **Morning brief `⏳ Owed to You` section.** The morning brief now includes a dedicated section for watching items that are overdue or due today, between `🔴 Overdue` and `📝 Meeting Next Steps — Mine`. Grouped by owner, overdue-first within owner.
- **Top 3 Decisions can include "chase" or "escalate" actions** when a critical watching item is overdue. The morning brief's Top 3 logic now considers both buckets — a "chase Heghine on FullStory slides — overdue 10 days" decision is just as valid as a "ship Q2 doc cleanup" decision.
- **Workload summary now reports both buckets.** "How's my workload?" returns 🎯 yours-to-do counts (priority, top goals, overdue, due-this-week) AND 👁 yours-to-watch counts (top owners, overdue, due-this-week).

### Changed
- `commands/help.md` lists the new monitoring queries and the `/briefs:monitoring` slash command.
- README's "Ask about your tasks" table updated with the new queries and the parallel-section behavior. New cross-link to the MONITORING data-model section.

### Why
v0.6.3's README claimed MONITORING items "drop out of priority queries automatically" — but that was emergent behavior, not enforced contract. A user customizing their tasks skill could have regressed it. v0.7.0 makes the rule explicit (Step 1.5) and adds the missing flip side: critically overdue watching items deserve to surface in the daily flow, not just in opt-in queries. If Heghine has owed you slides for 10 days, your morning brief should say so.

### Migration from v0.6.x
No action required. The classification rule is parsing-only — your existing `tasks.md` works as-is. The new queries and slash command are additive. Output format changes are visible the next time you run a priority query or the morning brief.

If you want to verify the new behavior:
- Ask Claude: "what am I waiting on?"
- Run: `/briefs:monitoring`
- Run: `/briefs:run morning` (look for the new ⏳ Owed to You section)

---

## [v0.6.3] — 2026-04-27

### Added
- **README "MONITORING" section.** New sub-section under "How your work is organized" documenting the third concept in the data model: how to mark goals or tasks you're watching — direct reports, partner teams, items waiting on someone else's confirmation/approval/delivery — vs. work you actively own. Worked example shows both levels (goal-level `## MONITORING:` and task-level `[MONITORING]` priority tag) and explains how MONITORING items behave in priority queries.

### Migration from v0.6.2
No action required — text-only documentation.

---

## [v0.6.2] — 2026-04-27

### Added
- **README "How your work is organized" section.** New top-level section before Quick Start that defines what a *goal* and *task* are, shows a worked example of the `tasks.md` schema (goal metadata + tasks with priority/owner/due/note), and explains the round-trip between Slack approvals and the file. Customization docs now link here from the `tasks_file` config block.

### Changed
- **README opening rewritten to lead with both halves of the product.** New tagline: "A self-maintaining task list, delivered through Slack." The pitch now explicitly calls out (1) goals/tasks organization and (2) automatic daily refresh as separable value props, instead of foregrounding only the briefs.
- **Poll confirmation label**: `✅ Board updated` → `✅ Goals updated` in `brief-poll` skill output. End users (especially first-timers) had no shared definition of "board" — `Goals` matches the new README terminology.

### Migration from v0.6.1
No action required — text-only changes. Next poll will emit "Goals updated" automatically.

---

## [v0.6.1] — 2026-04-25

### Changed
- README docs polish: new 3-column commands table (slash command / plain-language equivalent / what it does) and dedicated task-query examples table for the `tasks` skill. No behavior changes.

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
