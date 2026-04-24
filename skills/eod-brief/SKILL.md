---
name: eod-brief
description: End-of-day brief. Summarizes meetings attended, Zoom next-steps classified by owner, key Slack threads, and tomorrow's top-3 focus. Posts ONE message to the user's Slack self-DM. Runs standalone — does NOT depend on the morning brief existing.
---

# EOD Brief

You are running the user's end-of-day brief. Load their config first.

## Step 1 — Load config

Read `~/.claude/daily-workflow-briefs/config.md`. Same structure as the morning brief. If missing, tell the user to run `/brief-setup` first and stop.

Also read `$tasks_file` if it exists (for dedup against existing commitments).

## Step 2 — Lookback window

- Default: since start of today in `config.timezone`, up to now.
- If `config.eod_lookback_hours` is set, use that instead.

## Step 3 — Gather signals in parallel

### 3a. Meetings attended today (Google Calendar MCP)
Today's events that already started before now. For each:
- Time range (normalized to `config.timezone`)
- Title + key attendees
- Flag declined/OOO events — skip, user wasn't there

Pair with Zoom next-steps (step 3b) to annotate what came out of each meeting.

### 3b. Zoom meeting assets (Zoom MCP — primary source)

For each calendar event that ended before now: `search_meetings` with `q` = topic keywords, narrow `from`/`to` to the meeting window.

**IGNORE `has_summary_permission` in the search response** — it's about sharing, not access. False by default on ~every meeting.

For each match with `has_summary: true`: `get_meeting_assets` with the UUID. Real gate is `meeting_summary.has_permission` in the asset response.

Extract ONLY `meeting_summary.summary_plain_text`. Never read transcripts (150k+ chars).

Parse the `Next steps` block. Classify each action:
- **User** (match config display name — first/last/full) → `💼 Zoom next steps (mine)`
- **Team** (from `config.team`) → `🧑‍🤝‍🧑 Team next steps (watching)`
- Others → skip unless blocker on user/team deliverable

Use the `Quick recap` line to annotate each meeting in `📅 Meetings attended`.

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

The morning brief already covered 24h. At EOD, only check last ~8h for:
- Status changes on tickets the user cares about (pull watchlist from `tasks.md` — any Jira keys referenced in Notes/URLs)
- Tickets moved into "PMs"/"Needs PM Review"/"Selected for Fix" since the morning
- New comments on tickets where user is assigned or @-mentioned

Same JQL guardrails: quote reserved words, cap `maxResults: 10`, explicit fields list. 3–5 items max.

### 3g. HubSpot — last 8h (if hubspot integration on)

Deals the user owns with activity today: stage changes, new notes, meetings logged. Read-only summary.

## Step 4 — Synthesize "Tomorrow's top 3 focus"

Look ahead. What are the 3 things the user MUST prioritize tomorrow? Criteria:
- Deadlines tomorrow or day-after
- Commitments made today (from next-steps + Slack replies)
- Leftovers from today's decisions that didn't close
- Blockers surfaced above

Each: verb-first action + why it matters + when. Use `finalize / ship / submit / reply to / unblock` — not "think about".

## Step 5 — Synthesize "🤝 Work I can do for you"

Same process as morning brief — scan signals for delegable asks matching the allowlist (Confluence edits, Jira comments, Slack drafts, task writes, HubSpot notes). At EOD this tends to be smaller than morning since most actions have been handled, but:
- Next-steps from afternoon meetings that haven't been actioned yet
- Slack asks from afternoon threads
- "Write a recap / document X" type asks from meetings

## Step 6 — Compose and post the main brief

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

🧠 *Tomorrow's top 3*
1. [Verb + action] — [why / stakes / deadline]
2. ...
3. ...
```

Length: target under 2500 chars, cap 3500.

If any MCP errored, append `_Signals skipped: [list]_` at the bottom.

If the day is empty (no meetings, no signals), post a short "Light day — nothing flagged. Rest up." and stop.

## Step 7 — Post work offers as FIRST REPLY to the EOD thread

If Step 5 produced offers, post them as a thread reply (same format as morning brief). Use the same `.offers.jsonl` ledger — each offer gets a number scoped to this thread.

```
🤝 *Work I can do for you* — reply here to approve

1. [Short verb + target] — [context]
2. ...

Reply with `apply 1` / `skip 2` / `show 3` (to preview) / `edit 1: <new text>`.
I'll pick up your reply on the next poll. ⚡ Need it sooner? Reply here, then open Claude Code and run `/brief-run poll`.
```

If the next morning brief fires before the user replies, offers carry over — the poll reads the EOD thread too (any thread with unresolved `offered` entries in the ledger within the last 18 hours).

## Hard constraints

- Post only to `config.slack_self_dm`. Never post to any other channel.
- Do NOT modify tasks.md — that's the poll's job.
- EOD runs standalone. Do NOT require the morning brief to have fired.
- If the user is OOO (calendar shows all-day OOO), post "OOO today — no EOD brief" and stop.
- Weekends: skip entirely unless `config.run_on_weekends: true`.
- Report back a ≤200-word summary of sections included, skipped, and errors.
