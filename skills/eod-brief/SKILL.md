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

### 3b. Meeting transcriber (primary source for meeting content)

Read `config.meeting_transcriber`. Branch on `type`:

#### `type: zoom`

For each calendar event that ended before now: `search_meetings` with `q` = topic keywords, narrow `from`/`to` to the meeting window.

**IGNORE `has_summary_permission` in the search response** — it's about sharing, not access.

For each match with `has_summary: true`: `get_meeting_assets` with the UUID. Real gate is `meeting_summary.has_permission` in the asset response.

Extract ONLY `meeting_summary.summary_plain_text`. Never read transcripts (150k+ chars).

Parse the `Next steps` block. Classify each action:
- **User** (match `display_names` — first/last/full) → `💼 Meeting next steps (mine)`
- **Team** (from `config.team`) → `🧑‍🤝‍🧑 Team next steps (watching)`
- Others → skip unless blocker on user/team deliverable

Use `Quick recap` to annotate each meeting in `📅 Meetings attended`.

#### `type: granola | otter | fireflies | fathom | loom` (Gmail-based)

Use the Gmail MCP with `config.meeting_transcriber.gmail_query` (a Gmail search filter like `from:noreply@granola.ai newer_than:1d`).

For each matching thread today:
- `get_thread` with `FULL_CONTENT`.
- Parse body for "Quick recap" / "Summary" sections (annotate `📅 Meetings attended`).
- Parse for "Next steps" / "Action items" block. Classify each item by owner same as Zoom — user under `💼 Meeting next steps (mine)`, team under `🧑‍🤝‍🧑 Team next steps (watching)`, others skipped.
- Fall back to subject + first paragraph if structured parse fails.

#### `type: none`
Skip Step 3b entirely. Briefs still post calendar (Step 3a) and other signals.

#### `type: custom`
Branch on `source` — `mcp` uses `mcp_name`, `gmail` uses `gmail_query` — same logic as morning-brief Step 3b.

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

Dedup against tasks.md: if a save's URL or Jira key already appears in tasks, skip.

### 3f. Jira — last 8h (if jira integration on, lighter than morning)

Only check last ~8h for:
- Status changes on tickets the user cares about (Jira keys from tasks.md Notes/URLs)
- Tickets moved into "PMs"/"Needs PM Review"/"Selected for Fix" since the morning
- New comments on tickets where user is assigned or @-mentioned

Same JQL guardrails: quote reserved words, cap `maxResults: 10`, explicit fields list. 3–5 items max.

### 3g. HubSpot — last 8h (if hubspot integration on)

Deals the user owns with activity today: stage changes, new notes, meetings logged. Read-only summary.

### 3h. Additional integrations (from `config.additional_integrations`, last 8h)

Each entry has `name`, `source`, and either `description` (MCP) or `gmail_query` + `description` (Gmail). Legacy `name: description` form is treated as `source: mcp` implicitly.

For each entry:

- **`source: mcp`**: try the MCP whose name matches. If not connected, skip with `skipped: <name> (MCP not connected)`. Use the description as guidance. Heuristic: items where the user is owner/assignee/mentioned, last 8h.

- **`source: gmail`**: use the Gmail MCP with `gmail_query` directly. For each matching thread today, extract subject + first paragraph (or first 500 chars). For meeting-transcriber emails specifically (Granola/Otter/Fireflies/Fathom), look for "Quick recap" or "Next steps" sections and extract them. Classify any "Next steps" by owner (user vs team) the same way Zoom next-steps are classified — surface user's items under `💼 <name> next steps (mine)` and team items under `🧑‍🤝‍🧑 <name> next steps (watching)`.

Surface 1–3 items per integration. **Read-only.** Never offer write actions for these in v1.

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

💾 *New saves* (not yet in tasks)
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

## Step 6 — Synthesize "🤝 Work I can do for you"

Same allowlist and same logic as the morning brief's Step 5–6. Scan today's afternoon signals (post-noon meeting next-steps, late-day Slack threads, Gmail asks the user committed to in a reply) for delegable actions matching the allowlist.

**Allowlist (same as morning brief):**
- ✅ Edit/append a Confluence page (with diff preview)
- ✅ Add a comment to a Jira ticket (with preview)
- ✅ Draft a Slack reply (drafted only, never auto-sent)
- ✅ Add / update / mark-done a task in `tasks.md`
- ✅ Add a read-only note to a HubSpot deal
- ❌ Send email • Change Jira status/assignee/priority • Change HubSpot deal stage/amount/owner • Anything destructive • Write actions on additional integrations (read-only in v1)

If zero offers, skip Step 7 (don't post an empty offers reply).

## Step 7 — Post work offers as FIRST REPLY to the EOD thread

If Step 6 produced offers, post them as a thread reply to the EOD message (use the `thread_ts` from the message you posted in Step 5). Format mirrors the morning brief, with one critical difference: the closing line points at *tomorrow's morning brief*, not "the next poll" — because the poll doesn't run overnight.

**Format depends on `config.work_offer_preview` (default `summary`):**

If `summary`:
```
🤝 *Work I can do for you* — reply here to approve

1. [Short verb + target] — [one-line context]
2. [Short verb + target] — [one-line context]
3. [Short verb + target] — [one-line context]

Reply in your own tone — `accept`, `edit`, `skip`, or `show more` on any of the numbered suggestions above. I'll pick up your reply in tomorrow's morning brief and follow through then.
```

If `full`:
```
🤝 *Work I can do for you* — reply here to approve

*1. [Short verb + target]*
> [Full preview text / diff inline]

*2. [Short verb + target]*
> [Full preview text / diff]

Reply in your own tone — `accept`, `edit`, or `skip` any of the suggestions above. I'll pick up your reply in tomorrow's morning brief and follow through then.
```

**Write each offer to the offers ledger** so tomorrow's morning brief can find them by number. Append a JSON line to `~/.claude/daily-workflow-briefs/.offers.jsonl`:

```json
{"ts": "<ISO timestamp>", "brief_thread_ts": "<EOD message ts>", "offer_number": 1, "offer_id": "uuid-or-hash", "action_type": "confluence_edit|jira_comment|slack_draft|task_add|task_done|task_update|hubspot_note", "target": "<url or identifier>", "preview": "<full text/diff>", "source_signal": "<where this came from>", "status": "offered"}
```

The `offer_number` resets to 1 for this EOD thread (scoped by `brief_thread_ts`). Tomorrow's morning brief Step 1.5 (the EOD carryover) reads `.offers.jsonl` plus the EOD thread replies and processes approvals before producing today's brief.

### Why EOD offers no longer "sit unactioned overnight"

This was the rationale for removing EOD offers in v0.2.0: the poll didn't run after EOD, so any offer would just sit there until the next morning. v0.8.0 fixes this by adding **morning-brief Step 1.5** — a carryover pass that runs BEFORE the morning's signal gathering, reads yesterday's EOD thread for `accept` / `edit` / `skip` replies, and processes them through the same execution path the `brief-poll` skill uses. The user gets confirmations at the top of the morning brief (`📥 Carried over from yesterday's EOD`) so nothing approved overnight gets lost.

## Hard constraints

- Post only to `config.slack_self_dm`. Never post to any other channel.
- Do NOT modify tasks.md — that's the poll's job.
- EOD runs standalone. Do NOT require the morning brief to have fired.
- If the user is OOO (calendar shows all-day OOO), post "OOO today — no EOD brief" and stop.
- Weekends: skip entirely unless `config.run_on_weekends: true`.
- Report back a ≤200-word summary of sections included, skipped, and errors.
