---
name: eod-brief
description: End-of-day brief. Summarizes meetings attended, Zoom next-steps classified by owner, key Slack threads, and tomorrow's top-3 focus (or Monday's if today is Friday). Posts ONE message to the user's Slack self-DM. Standalone — does NOT depend on the morning brief existing. Does NOT post work offers (those would sit unactioned overnight; offers happen on morning + poll only).
---

# EOD Brief

You are running the user's end-of-day brief. Load their config first.

## Step 1 — Load config

Read `~/.claude/daily-workflow-briefs/config.md`. Same structure as the morning brief. If missing, tell the user to run `/briefs:setup` first and stop.

Also read `$tasks_file` if it exists (for dedup against existing commitments).

## Step 2 — Determine context

- Compute `today_local` and `today_dow` (day-of-week 0=Mon..6=Sun) in `config.timezone`.
- `is_friday = (today_dow == 4)` — used in Step 4 for next-day framing and the weekend send-off.
- Lookback window: since start of today in `config.timezone`, up to now. Override with `config.eod_lookback_hours` if set.

## Step 3 — Gather signals in parallel

### 3a. Meetings attended today (Google Calendar MCP)
Today's events that already started before now. For each:
- Time range (normalized to `config.timezone`)
- Title + key attendees
- Flag declined/OOO events — skip, user wasn't there

Pair with Zoom next-steps (step 3b) to annotate what came out of each meeting.

### 3b. Zoom meeting assets (Zoom MCP — primary source)

For each calendar event that ended before now: `search_meetings` with `q` = topic keywords, narrow `from`/`to` to the meeting window.

**IGNORE `has_summary_permission` in the search response** — it's about sharing, not access.

For each match with `has_summary: true`: `get_meeting_assets` with the UUID. Real gate is `meeting_summary.has_permission` in the asset response.

Extract ONLY `meeting_summary.summary_plain_text`. Never read transcripts (150k+ chars).

Parse the `Next steps` block. Classify each action:
- **User** (match config display_names — first/last/full) → `💼 Zoom next steps (mine)`
- **Team** (from `config.team`) → `🧑‍🤝‍🧑 Team next steps (watching)`
- Others → skip unless blocker on user/team deliverable

Use `Quick recap` to annotate each meeting in `📅 Meetings attended`.

### 3c. Gmail — Jira "work due" digests from today (if jira integration on)

`search_threads` with query `subject:"you have work due in Jira" newer_than:1d`.

For each hit, `get_thread` FULL_CONTENT and extract tickets with due dates. Surface overdue or due-tomorrow tickets not already handled as candidates for **Tomorrow's top 3** or an EOD heads-up in `💬 Key threads + DMs`. Dedup against tasks.md.

### 3d. Slack — last 12h

Same channel-ID + ts-filter approach as morning brief. Channels from `config.watch_channels`, DMs via team user IDs as channel IDs from `config.team`.

For each new signal, classify:
- **Decision made / commitment given** → EOD summary
- **Escalation / flag for tomorrow** → tomorrow's focus
- **Informational** → skip

### 3e. Saved messages — new today

`slack_search_public_and_private` with `from:<@{slack_user_id}> is:saved after:<today_start_YYYY-MM-DD>`.

Dedup against tasks.md: if a save's URL or Jira key already appears on the board, skip.

### 3f. Jira — last 8h (if jira integration on, lighter than morning)

Only check last ~8h for:
- Status changes on tickets the user cares about (Jira keys from tasks.md Notes/URLs)
- Tickets moved into "PMs"/"Needs PM Review"/"Selected for Fix" since the morning
- New comments on tickets where user is assigned or @-mentioned

Same JQL guardrails: quote reserved words, cap `maxResults: 10`, explicit fields list. 3–5 items max.

### 3g. HubSpot — last 8h (if hubspot integration on)

Deals the user owns with activity today: stage changes, new notes, meetings logged. Read-only summary.

### 3h. Additional integrations (from `config.additional_integrations`, last 8h)

For each entry `<mcp_name>: <description>`:
1. Try to invoke the corresponding MCP. If not connected, log "skipped" and continue.
2. Use the description as guidance. Apply heuristic: items where the user is owner/assignee/mentioned, last 8h, narrow to "what changed today".
3. Surface 1–3 items per integration as a section labeled with the integration name (e.g. `📞 Intercom`, `💼 Salesforce`).
4. **Read-only.** Never offer write actions for these in v1.

## Step 4 — Synthesize "Tomorrow's top 3 focus"

Look ahead. What are the 3 things the user MUST prioritize NEXT WORKING DAY? Criteria:
- Deadlines tomorrow (or Monday if Friday) or the day after
- Commitments made today (from next-steps + Slack replies)
- Leftovers from today's decisions that didn't close
- Blockers surfaced above

Each: verb-first action + why it matters + when. Use `finalize / ship / submit / reply to / unblock` — not "think about".

**If `is_friday` is true:**
- Header reads `🧠 Monday's top 3` (not "Tomorrow's").
- Resolve "tomorrow" / "next day" deadlines as Monday's date.
- Include weekend-context items only if they're truly time-sensitive (a customer escalation that could spiral over the weekend, an oncall handoff). Most weekend items should defer.

## Step 5 — Compose and post the main brief

Post ONE message to `config.slack_self_dm`. Format:

```
🌇 *EOD Brief — [Weekday], [Month Day]*
[One sentence — how the day went. Theme: decisions / execution / meeting-heavy / shipped X]

📅 *Meetings attended*
• HH:MM–HH:MM: Title — one-line outcome

💼 *Zoom next steps (mine)*
• [From: Meeting Title] Action text

🧑‍🤝‍🧑 *Team next steps (watching)*
• [Owner] Action text — [From: Meeting Title]

💬 *Key threads + DMs*
• [Channel or person] — decision/commitment in one line

💾 *New saves* (not yet on the board)
• Sender — summary

📋 *Jira Signals (last 8h)*
• KEY-123: title (status) — why it matters

[Additional integration sections, e.g.]
📞 *Intercom*
• [conversation or update]

💼 *Salesforce*
• [opportunity or update]

🧠 *<Tomorrow's | Monday's> top 3*
1. [Verb + action] — [why / stakes / deadline]
2. ...
3. ...
```

**If `is_friday` is true, append a weekend send-off line at the very bottom (after Top 3):**
```
☀️ Have a great weekend! See you Monday.
```

Pick a phrase from this rotation so it doesn't feel rote: "Have a great weekend!", "Enjoy the weekend.", "Catch you Monday.", "Out 'til Monday — rest up.", "Weekend's earned. See you Monday."

Length: target under 2500 chars, cap 3500.

If any MCP errored, append `_Signals skipped: [list]_` at the bottom (above the weekend line if Friday).

If the day is empty (no meetings, no signals), post a short "Light day — nothing flagged. Rest up." (and append the weekend line if Friday) then stop.

## Step 6 — DO NOT post work offers from EOD

The poll runs 8am–4pm weekdays. EOD posts at 3:30pm by default. Any work offers posted at EOD would have at most one poll cycle to be approved (often zero — most users won't reply between 3:30pm and the last poll). Unapproved offers then sit overnight until next morning's brief, which already produces fresh offers based on overnight signals.

**For v1: EOD does NOT emit work offers.** Surface action-required items in the Tomorrow's Top 3 section instead — the user reviews them tomorrow morning when they have time to act.

If a delegable ask appears in afternoon signals, the next morning brief will pick it up (overnight signal scan covers the previous afternoon).

## Hard constraints

- Post only to `config.slack_self_dm`. Never post to any other channel.
- Do NOT modify tasks.md — that's the poll's job.
- EOD runs standalone. Do NOT require the morning brief to have fired.
- If the user is OOO (calendar shows all-day OOO), post "OOO today — no EOD brief" and stop.
- Weekends: skip entirely unless `config.run_on_weekends: true`.
- Report back a ≤200-word summary of sections included, skipped, and errors.
