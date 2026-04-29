---
name: morning-brief
description: Pulls calendar, Zoom meeting summaries, Gmail, Slack signals, and (optional) Jira for the user's morning brief. Posts ONE message to their Slack self-DM with today's meetings, overdue tasks, meeting next-steps, monitoring signals, and top-3 decisions. Follows the user's config at ~/.claude/daily-workflow-briefs/config.md.
---

# Morning Brief

You are running the user's morning brief. Load their config first — it defines everything user-specific (Slack IDs, channels, team, integrations, role).

## Step 1 — Load config

Read `~/.claude/daily-workflow-briefs/config.md`. It contains:
- `slack_user_id` — the user's Slack user ID (U...)
- `slack_self_dm` — the channel ID to post to (D...)
- `watch_channels` — list of `CID: #name` entries
- `team` — list of `Name: UID` entries (for DM scanning)
- `vips` — list of `Name: email@domain` entries (always surface their emails)
- `integrations` — flags: `jira: on|off|projects:[KEY1,KEY2]`, `hubspot: on|off`
- `timezone` — default `America/Los_Angeles` (normalize ALL displayed times to this)
- `tasks_file` — path to the user's tasks.md (default `~/.claude/daily-workflow-briefs/tasks.md`)

If the config file doesn't exist, tell the user to run `/brief-setup` first and stop.

Also read `$tasks_file` if it exists — source of truth for "overdue" section.

## Step 1.5 — Process leftover replies from yesterday's EOD brief (carryover)

The `brief-poll` skill stops running before EOD posts (8am–4pm window). Any approvals the user typed into the EOD thread between EOD posting (~3:30pm) and yesterday's last poll won't have been processed by the time you wake up. This step closes that gap so nothing approved overnight gets lost.

Run this BEFORE Step 2 (lookback) and Step 3 (signal gathering) so any task changes are reflected in today's `🔴 Overdue` and `⏳ Owed to You` sections.

**Workflow:**

1. **Find yesterday's EOD message in the user's self-DM.** Use `slack_read_channel` on `config.slack_self_dm` and look for the most recent bot message containing `🌇` (EOD emoji) posted before today's morning-brief run time.
   - Standard lookback: 24 hours.
   - **On Mondays / day-after-OOO**: extend the lookback to 96 hours so weekend / holiday EOD replies don't get lost. Detect Mondays via `config.timezone`; detect post-OOO by scanning calendar for an OOO event in the last 1–3 days.
   - If no EOD message found within the lookback, **skip Step 1.5 entirely** (nothing to carry over).

2. **Read the EOD thread.** Call `slack_read_thread` on the EOD message's `ts`. Filter replies to the user's `slack_user_id` only (skip bot messages — including any `✅ Tasks updated` confirmations from prior carryover runs).

3. **Skip already-processed replies.** Read `~/.claude/daily-workflow-briefs/.poll-log.jsonl`. Look for any prior log entry with `brief_thread_ts` matching this EOD thread's `ts` AND `source: eod_carryover` (or `source: poll` from yesterday's afternoon polls that touched this thread). Replies with `ts` ≤ the latest such processed-timestamp are already handled — skip them. Process only newer replies.

4. **Apply the same parsing logic as `brief-poll` Steps 3–5.** Match `accept N` / `apply N` / `skip N` / `show more N` / `edit N: ...` / `accept all` against `.offers.jsonl` entries scoped to this EOD thread's `brief_thread_ts`. Also match natural-language patterns (`mark X done`, `add a task to <goal>`, `move Y to Friday`, etc.). Strip conversational filler before matching. Execute approved offers through the allowlist (Confluence edit, Jira comment, Slack draft, tasks.md write, HubSpot note).

5. **Append a poll-log entry tagged `source: eod_carryover`** so this thread doesn't get re-processed if the morning brief ever runs twice in a day:

   ```json
   {"ts": "<ISO timestamp>", "brief_thread_ts": "<EOD ts>", "source": "eod_carryover", "replies_processed": <N>, "tasks_added": <N>, "tasks_modified": <N>, "offers_applied": <N>, "offers_skipped": <N>, "offers_edited": <N>, "errors": <N>, "notes": "<one-line summary>"}
   ```

6. **Surface the results at the TOP of today's morning brief output** as the leading section (see Step 7's template — `📥 *Carried over from yesterday's EOD*`). Same formatting conventions as the poll's confirmation message: brief, plain-English, list of changes grouped by type (Applied / Edited / Skipped / Tasks updated / Errors).

7. **If nothing was processed**, skip the carryover section in the brief output entirely. Don't render "no carryover" — that's noise.

**Edge cases:**
- **EOD didn't post offers yesterday** — possible if Step 6 of EOD found zero delegable asks. The thread may still have natural-language replies ("mark X done") — process those normally; just no `accept N` entries to look up.
- **Mid-execution failure** — if applying one offer errors, mark it `errored` in `.offers.jsonl` and continue with the rest. Surface the error in the carryover section.
- **EOD thread is missing** (user deleted it, or it's stale beyond the lookback) — skip silently.
- **More than 10 approvals in carryover** — apply the first 10 and defer the rest with `⏸ 10 applied — N more pending; will continue next poll`. Same rate-limit pattern the regular poll uses.

## Step 2 — Resolve lookback window

Times are in `config.timezone` throughout.
- **Tuesday–Friday**: last 12 hours
- **Monday**: last 72 hours (captures weekend)
- **Day after a holiday**: extend by the holiday span (check calendar for OOO events)

## Step 3 — Gather signals in parallel

### 3a. Calendar (Google Calendar MCP — required)
Today's events. For each: time range (normalized to `config.timezone`), title, key attendees (match against `config.team` + `config.vips` — if none of those, summarize as "external"), Zoom/meet link. Flag conflicts and high-stakes meetings. Skip optional/tentative standups unless they're the only thing.

**Normalize ALL times to `config.timezone`** — calendar events may come back in mixed TZs (different attendees' zones).

### 3b. Meeting transcriber — PRIMARY source for meeting content

Read `config.meeting_transcriber`. Branch on `type`:

#### `type: zoom` (default — most common)

For every meeting that happened in the lookback window, pull assets DIRECTLY from Zoom, NOT from Gmail. The Gmail MCP's `get_thread` silently drops body content on Zoom's HTML-only emails — verified broken.

**Workflow:**
1. List completed meetings from the calendar.
2. For each: `search_meetings` with `q` = meeting topic keywords, `from`/`to` narrowed around the actual meeting time (UTC).
3. **IGNORE `has_summary_permission` in the search response.** It refers to sharing with others, NOT the user's access. It's false by default on nearly every meeting. Do NOT gate on it.
4. For each match where `has_summary: true`: call `get_meeting_assets` with the `meeting_uuid`. The real access flag is `meeting_summary.has_permission` inside the asset response. If that's false, skip with "content unavailable".
5. Extract ONLY `meeting_summary.summary_plain_text`. DO NOT read `recording.transcripts` — can be 150k+ chars per meeting.
6. Parse the `Next steps` block — each line is `<name>: <action>`. Classify by owner:
   - **User** (match against display_names — first/last/full) → `💼 Meeting Next Steps — Mine`
   - **Team** (match against `config.team`) → `🧑‍🤝‍🧑 Meeting Next Steps — Team`
   - Others → skip unless blocker on user/team deliverable
7. If `has_summary: false`, fall back to Gmail email subject line only (no body) — flag as "content unavailable".

#### `type: granola | otter | fireflies | fathom | loom` (Gmail-based)

These tools email meeting recaps. Use the Gmail MCP with `config.meeting_transcriber.gmail_query` directly (it's already a Gmail search filter, e.g. `from:noreply@granola.ai newer_than:1d`).

**Workflow:**
1. `search_threads` with the configured `gmail_query`.
2. For each matching thread (within the lookback window, dedup by message ID):
   - `get_thread` with `FULL_CONTENT` to read the body.
   - Parse the body for "Quick recap" / "Summary" sections — most transcribers structure their emails this way.
   - Parse for a "Next steps" / "Action items" block. Each item typically starts with a name or @-mention.
   - Classify next-steps the same way as Zoom: user → mine, team → watching, others → skip.
3. Surface 1–2 sentences from the recap per meeting in `📅 Today's Meetings` (annotate the calendar entry).
4. If the email body is malformed or doesn't have parseable recap/next-steps, fall back to subject + first paragraph and flag as "structured parse failed — raw summary".

**If `gmail_query` returns zero results in the lookback window:** that's normal — no meetings ended in this window. Skip silently.

#### `type: none`

Skip Step 3b entirely. Don't surface a "Meeting Next Steps" section. Briefs still post calendar (Step 3a) and other signals.

#### `type: custom`

Look at `config.meeting_transcriber.source`:
- If `mcp`: invoke the MCP whose name matches `config.meeting_transcriber.mcp_name` (or fall back to `type` field). Use `description` as guidance. Apply the same owner-classification logic if the response includes structured next-steps.
- If `gmail`: use `gmail_query` exactly like the named-Gmail variants above.

### 3c. Gmail (Gmail MCP — required)

**VIPs (from `config.vips`):** Surface any thread in the window where a VIP is sender or recipient. Sender > recipient priority. Use email-based search (e.g. `from:vip1@domain.com OR from:vip2@domain.com newer_than:1d`).

**Team (from `config.team`):** Surface threads that need user action or indicate a blocker. Skip FYI/informational.

**Jira "work due" digests** (if `integrations.jira != off`): Run `search_threads` with query `subject:"you have work due in Jira" newer_than:2d`. For each hit, call `get_thread` with `FULL_CONTENT` to extract ticket blocks (key, title, due date, status). Surface due-today or overdue tickets under `📋 Jira Signals` labeled "(due digest)".

**Jira comment notifications** (if `integrations.jira != off`): Don't call `get_thread` on these — subject line has the ticket key. Group into a single line per ticket.

**Skip entirely:** promotional, automated calendar invites (already on calendar), broad distribution lists, status-page alerts, weekly digests, marketing.

If Gmail MCP errors (auth, rate limit), note it at the bottom and continue.

### 3d. Slack (Slack MCP — required)

**Watch channels (from `config.watch_channels`):** Use `slack_read_channel` WITHOUT the `oldest` parameter (silently drops everything on this MCP). Fetch recent messages (default limit=100 is fine) and filter client-side by each message's `ts` (Unix epoch seconds) against the lookback window. For each channel, surface @-mentions of the user's Slack ID, replies to their threads, items clearly relating to the user's active work.

**DMs from team (from `config.team`):** For each team member's user ID, call `slack_read_channel` passing the USER ID as `channel_id` (the tool supports this — it reads DM history). Do NOT pass `oldest` — filter client-side by `ts`.

Surface messages they sent to the user in the window that are action-requiring (asks, decisions, escalations). Skip FYI.

**Saved messages:** `slack_search_public_and_private` with `from:<@{slack_user_id}> is:saved after:YYYY-MM-DD` where `after` = lookback start. Without the date filter the search returns months-old stale items. Cross-reference with tasks.md — if a save already maps to a task, skip.

### 3e. Jira (Atlassian MCP — optional, based on `integrations.jira`)

If `integrations.jira` is `off`, skip this entire section.

If `integrations.jira` is `on` (all projects): use `searchJiraIssuesUsingJql` with a broad JQL covering all the user's projects and `updated >= -1d`.

If `integrations.jira` is a project list like `projects:[AI,WAT,INT]`: scope JQL to those project keys.

**Hard constraints to avoid context blowups:**
- `maxResults: 15` max
- Explicit `fields: ["summary","status","priority","assignee","reporter","updated","labels"]` — some tickets include a `renderedFields.description` blob that inflates responses to 50k+ chars.
- **Quote reserved JQL words**: `"INT"`, `"OR"`, `"AND"`, `"NOT"`, etc. — without quotes the query errors.
- If a single response exceeds ~30k chars, truncate each ticket's text fields to ~300 chars in memory before synthesizing.

Run two queries:
1. User-involved (`assignee = currentUser() OR reporter = currentUser() OR text ~ "{user_email}"`) — last 24h, ordered by updated DESC.
2. PM review queue (`status in ("PMs", "Needs PM Review", "Selected For Fix")`) — last 48h, cap `maxResults: 10`.

Surface: status changes, new comments tagging the user, blockers escalated from CS/QA.

### 3f. HubSpot (optional, based on `integrations.hubspot`)

If `integrations.hubspot` is `on`: scan last 24h for deals where the user is owner. Surface deal-stage changes, new notes, meetings logged. **Read-only in the morning brief** — any offers to update HubSpot deals require explicit confirmation (handled by brief-poll, not here).

### 3g. Additional integrations (from `config.additional_integrations`)

Each entry is a structured record with `name`, `source`, and either a `description` (for MCP) or a `gmail_query` + `description` (for Gmail-fallback). Some legacy configs use the simple `name: description` form — treat those as `source: mcp` implicitly.

For each entry:

**Branch on `source`:**

- **`source: mcp`** (or unspecified): Try to invoke the MCP whose name matches. If not connected, log `skipped: <name> (MCP not connected)` and continue. Use `description` as guidance for what to check. Default heuristic: items where the user is owner/assignee/mentioned, last 24h.

- **`source: gmail`**: Use the Gmail MCP with the `gmail_query` field directly as the search filter (e.g. `from:noreply@granola.ai newer_than:1d`). For each matching thread, read the subject + first paragraph (or first 500 chars of body if short). Surface as a brief summary. The `description` field guides what to extract. Common case: meeting transcribers (Granola/Otter/Fireflies/Fathom) → extract "Quick recap" / "Next steps" sections from the email body if present.

Surface 1–3 items per integration as a section labeled with the integration name (e.g. `📞 Intercom (last 24h)`, `🎙 Granola — recent meetings`).

**Read-only.** Never offer write actions for additional integrations in v1, regardless of source or what the underlying MCP supports.

If both an MCP-source entry's MCP fails AND a fallback isn't configured, log clearly: `skipped: <name> — MCP not connected and no Gmail fallback. Use /briefs:config to add a gmail_query if the tool emails you.`

### 3h. Tasks: yours-overdue + owed-to-you (from `$tasks_file`)

Parse tasks.md. Apply the **yours-to-do vs. yours-to-watch** classification (see the `tasks` skill, Step 1.5 — same logic):

- **Yours to do**: task `Owner:` includes the user (display_names, "You", "Will", "Me", first-name match).
- **Yours to watch**: `Owner:` does NOT include the user, AND (parent goal heading is `## MONITORING:` OR task priority is `[MONITORING]`).
- The owner check trumps the goal-level marker — a `## MONITORING:` goal can contain a Will-owned task that goes in the *yours-to-do* bucket.

Produce TWO outputs:

**Yours overdue** — yours-to-do tasks where due < today in `config.timezone`. Grouped by goal. Limit ~10, Critical > High > Medium. Include a one-line total ("X overdue across Y goals").

**Owed to you** — yours-to-watch tasks where due < today (overdue) OR due = today. Grouped by owner. Within owner: overdue first (most overdue → least), then due-today. Limit ~7. Include a one-line total ("X items across Y people").

If `$tasks_file` doesn't exist or is empty, skip both. If only one bucket has content, render only that one.

## Step 4 — Synthesize "Top 3 Decisions Today"

Three most consequential actions TODAY. Decisions can come from EITHER bucket:
- **Yours-to-do** items: deadline in next 24–48h, blocker surfaced in signals, leadership/customer/eng waiting on the user
- **Yours-to-watch** items: critically overdue (e.g. "chase Heghine on FullStory slides — overdue 10 days") or blocking your own work

Don't always pick three from one bucket — mix when both have urgent items. A "chase X" or "escalate Y" decision is just as valid as a "ship Z" decision when someone's been sitting on a deliverable.

Each decision: one verb-first action + the reason + the deadline. Use `reply to / approve / decide between / ship / escalate / chase / nudge` — not "think about X".

## Step 5 — Synthesize "🤝 Work I can do for you"

Scan the collected signals for **explicit asks or action items the user could delegate to this agent**. Candidate sources:
- Zoom next-steps owned by the user ("Will: update the Q2 Confluence page")
- Slack messages the user sent indicating intent ("I need to update the MCP KB article")
- Slack messages to the user containing a specific ask ("@user can you add X to the Confluence page?")
- Email threads where the user committed to an action in the reply

For each candidate, check against the **allowlist of actions this plugin can take** (see Step 6 below). If the ask maps to an allowlisted action, formulate an offer. If not, skip it (or surface as a regular task in `tasks.md`).

Offers go in the FIRST REPLY to the brief thread (Step 8), not in the main brief.

## Step 6 — Allowlist of actions for work offers

Only offer actions on this allowlist. If an ask doesn't map to one, do not offer it.

**Allowed (with confirmation):**
- Edit/append to a Confluence page (with a text diff preview before applying)
- Add a comment to a Jira ticket (with preview)
- Draft a Slack reply (always draft — never auto-send; user copies/pastes or the execution step drafts-in-thread)
- Add, update, or mark-done a task in the user's tasks.md
- Add a read-only note to a HubSpot deal (if `integrations.hubspot: on`) — NEVER change deal stage, owner, or amount

**Explicitly blocked:**
- Sending any email
- Changing Jira status, assignee, or priority
- Changing HubSpot deal stage, amount, owner, or close date
- Deleting anything anywhere
- Creating calendar events with external attendees

When in doubt, don't offer. Surface as a task instead so the user drives it.

## Step 7 — Compose and post the main brief

Post ONE message to `config.slack_self_dm` via `slack_send_message`. Skip sections that have no items — don't write "nothing to report". Format:

```
🌤 *Daily Brief — [Weekday], [Month Day]*
[One sentence: theme of the day]

📥 *Carried over from yesterday's EOD*
• ✓ Applied: <action> — <link to target>
• ✏️ Edited: <action> (still awaiting `accept N`)
• ⏭ Skipped: <action>
• ➕ Tasks added: <N> — see <goal> for details
• 🚨 Errored: <action> — <reason>
(Section omitted entirely if no replies were carried over.)

📅 *Today's Meetings*
• HH:MM–HH:MM: Title (key attendees)

🔴 *Overdue*
• Goal → [PRIORITY] Task (was due Month Day)

⏳ *Owed to You*
• Owner — [PRIORITY] Task (overdue N days, Goal)
• Owner — [PRIORITY] Task (due today, Goal)

📝 *Meeting Next Steps — Mine*
• [From: Meeting Title] Action text

🧑‍🤝‍🧑 *Meeting Next Steps — Team*
• [Owner] Action text — [From: Meeting Title]

👁 *Monitoring Signals*
• [Channel/source] — one-line summary with a <link|short label>

📋 *Jira Signals (last 24h)*
• KEY-123: Title (status) — why it matters

💾 *Saved Messages* (new since last brief)
• Sender — summary

🧠 *Top 3 Decisions Today*
1. [Verb + specific action] — [why / stakes / deadline]
2. ...
3. ...
```

Length: target 1500–2500 chars, cap 4000.

If any MCP errored, append one italic line at the bottom: `_Signals skipped: [list]_`.

If the day is genuinely empty (weekend, holiday, no signals, no meetings), post a short "Light day — nothing flagged. Signals checked: [list]." instead.

## Step 8 — Post work offers as FIRST REPLY to the brief thread

If Step 5 produced any offers, post them as a THREAD REPLY to the brief message (use `thread_ts` from Step 7's message) via `slack_send_message`. This is deliberate: the offers start the conversation, signaling to the user that the thread is the approval channel.

If Step 5 produced zero offers, skip this step entirely.

**Format depends on `config.work_offer_preview` (default `summary`):**

If `summary`:
```
🤝 *Work I can do for you* — reply here to approve

1. [Short verb + target] — [one-line context]
2. [Short verb + target] — [one-line context]
3. [Short verb + target] — [one-line context]

Reply in your own tone — `accept`, `edit`, `skip`, or `show more` on any of the numbered suggestions above. I'll follow through on the next poll automatically.
⚡ Need it sooner? Reply here, then open Claude Code and run `/briefs:run poll` — it'll process your reply within seconds instead of waiting for the cron.
```

If `full`:
```
🤝 *Work I can do for you* — reply here to approve

*1. [Short verb + target]*
> [Full preview text / diff inline — what Claude will apply if approved]

*2. [Short verb + target]*
> [Full preview text / diff]

Reply in your own tone — `accept`, `edit`, or `skip` any of the suggestions above. I'll follow through on the next poll automatically.
⚡ Need it sooner? Reply here, then open Claude Code and run `/briefs:run poll`.
```

**Write each offer to the offers ledger** so the poll can find them by number:

Append a JSON line to `~/.claude/daily-workflow-briefs/.offers.jsonl` for each offer:
```json
{"ts": "<ISO timestamp>", "brief_thread_ts": "<parent message ts>", "offer_number": 1, "offer_id": "uuid-or-hash", "action_type": "confluence_edit|jira_comment|slack_draft|task_add|task_done|task_update|hubspot_note", "target": "<url or identifier>", "preview": "<the full text/diff Claude would apply>", "source_signal": "<where this came from>", "status": "offered"}
```

The `offer_number` resets to 1 for each new brief or poll post (scoped by `brief_thread_ts` + post time). The poll's reply-parser will use `brief_thread_ts` + `offer_number` to look up the full details.

## Hard constraints

- Post only to `config.slack_self_dm`.
- Do NOT modify tasks.md from this skill. (Tasks changes happen via the poll's approval loop.)
- Dedupe across sections — don't list the same thread under Slack AND Monitoring.
- If the user is OOO (all-day OOO event on calendar), post a short "OOO today — no brief" and stop.
- Report back a ≤200-word summary of what was included, what was skipped, and any errors so the user can spot issues quickly.
