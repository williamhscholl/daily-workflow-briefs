---
name: brief-poll
description: Periodic brief poll. Reads replies to today's brief threads, executes approved work offers (apply/skip/edit), updates tasks.md from natural-language replies, scans new signals since last poll, and emits fresh work offers as a thread reply. Runs on the interval configured in the user's config.
---

# Brief Poll

You run on an interval (default 1 hour, configurable in `config.md`). Your job is four-fold:

1. **Process approvals** — read thread replies to today's brief(s), find `apply N` / `skip N` / `edit N: ...` commands, execute approved offers, post confirmations in-thread.
2. **Natural-language task updates** — parse free-form replies for task changes (add, done, reprioritize, reassign).
3. **Scan new signals** — Zoom meetings ended, Jira "work due" digests, new Slack signals from team/VIPs — since the last poll run.
4. **Emit new offers** — if new signals produce new delegable asks, post them as a new numbered list in the thread.

## Step 1 — Load context

- `~/.claude/daily-workflow-briefs/config.md`
- `$tasks_file` (from config; default `~/.claude/daily-workflow-briefs/tasks.md`)
- `~/.claude/daily-workflow-briefs/.offers.jsonl` (ledger; create if missing)
- `~/.claude/daily-workflow-briefs/.poll-log.jsonl` (run log; create if missing)

If config missing → tell user to run `/brief-setup` and stop.

## Step 2 — Exit early if weekend/OOO

- If weekend and `config.run_on_weekends: false` → exit.
- If today's calendar has an all-day OOO → exit.

## Step 3 — Find active brief threads

"Active" = posted in `config.slack_self_dm` within the last 18 hours by the user or the bot, starting with 🌤 (morning) or 🌇 (EOD), that have at least one unresolved offer in `.offers.jsonl`, OR have replies from the user since the last poll.

Use `slack_read_channel` on `config.slack_self_dm` WITHOUT `oldest` (the param is broken on this MCP — filter client-side by `ts`). Limit 50 recent messages is sufficient.

For each active brief thread found, call `slack_read_thread` to get replies.

## Step 4 — Parse user replies

Filter thread replies to those from `config.slack_user_id` (skip bot messages from prior polls).

For each reply, parse it in this order — first match wins, but a single reply can contain multiple directives on separate lines:

### 4a. Offer approval commands

Regex patterns (case-insensitive, leading whitespace ok):
- `^apply\s+(\d+)(?:\s+(.*))?$` — approve offer N (optional extra text = note to append to the action)
- `^skip\s+(\d+)$` — dismiss offer N
- `^show\s+(\d+)$` — reply in-thread with full preview of offer N (don't execute; just show)
- `^edit\s+(\d+):\s*(.+)$` — replace offer N's `preview` text with the provided string, mark as `edited`, wait for explicit `apply N` before executing
- `^apply\s+all$` — approve every currently-offered item in this thread (confirm first with a reply before executing)

For `apply`, `edit`, `skip`, `show`: look up the offer by `brief_thread_ts` + `offer_number` in `.offers.jsonl`. Only match offers with status `offered` or `edited` (skip `applied` / `skipped` / `expired`).

### 4b. Natural-language task commands (applied to `$tasks_file`)

Patterns:
- **Completion**: `mark X done` / `X is done` / `finished X` / `completed X` → find matching task, toggle `- [ ]` → `- [x]`. If multiple matches or ambiguous, DON'T guess — flag under `Flagged for your review` in the confirmation.
- **New task**: `add task: X` / `new task X` / `add to [goal]: X` — parse owner/priority/due if given ("assign to Y", "label critical", "due tomorrow"). Resolve relative dates ("tomorrow", "next Monday") to ISO YYYY-MM-DD in `config.timezone`.
- **Modification**: `move X to Y` / `change X due to Y` / `bump X to critical` — update the specific field.
- **For me**: `for me: X` or a bare action with a deadline → assume Owner = user, Priority = HIGH unless stated.

### 4c. Context reference

If the user pastes a Slack link or email subject with instructions like `add context to [goal]`: fetch the source (Slack via `slack_read_thread`, Gmail via `get_thread`), extract relevant info, append to the target goal's Context block in `$tasks_file`.

### 4d. Ambiguity handling

If a reply is ambiguous (unclear which task, unclear goal, multiple matches), prefer the most likely interpretation AND add it to a `Flagged for your review` list in the confirmation. Never silently guess.

## Step 5 — Execute approved offers

For each offer where parsing identified `apply N`:

1. Look up the offer in `.offers.jsonl`.
2. Branch on `action_type`:

### action_type: `confluence_edit`
- Call `getConfluencePage` on `target` (page ID or URL) to fetch current content.
- Apply the change described in `preview` (append, replace section, etc.).
- Call `updateConfluencePage` with new content.
- Capture the new version number.
- Confirmation payload: `✓ Updated Confluence page <target> — [summary of change]. View: <page URL>`.

### action_type: `jira_comment`
- Call `addCommentToJiraIssue` with the ticket key from `target` and `preview` as the comment body.
- Confirmation payload: `✓ Added comment to <TICKET-KEY>: "[first 80 chars]..." — <ticket URL>`.

### action_type: `slack_draft`
- **Never auto-send.** Instead, create the draft via `slack_send_message_draft` (if tool supports it) targeting the original channel — but do NOT deliver.
- If draft creation isn't supported, just post the drafted text in the BRIEF thread (not the target channel) with a `📝 Drafted reply for <channel> — copy/paste to send:` prefix.
- Confirmation payload: `✓ Drafted Slack reply for <channel> — see thread above for the draft text`.

### action_type: `task_add` / `task_done` / `task_update`
- Edit `$tasks_file` directly. Respect existing format: `- [ ] [PRIORITY] Task text | Owner: X | Due: YYYY-MM-DD | Note: N`.
- Preserve task order within sections unless the reorder is explicit.
- Never delete unless user said so.
- If user edited tasks.md directly since last poll (detect via file mtime vs last poll timestamp in `.poll-log.jsonl`), treat user's edits as authoritative — do NOT revert.
- Confirmation payload: `✓ Added/Completed/Updated task "<task text>" in goal <goal>`.

### action_type: `hubspot_note`
- Call the HubSpot MCP's note-creation tool on the deal ID from `target` with `preview` as the note body.
- **NEVER change stage, owner, amount, or close date** — if the preview text implies that, abort and post `⚠ Offer N aborted — HubSpot action out of allowed scope. This plugin only adds notes.`
- Confirmation payload: `✓ Added note to HubSpot deal <deal name> — "[first 80 chars]..."`.

3. Update `.offers.jsonl`: flip status from `offered` → `applied`, add `applied_ts`, add `confirmation`, add `result` (URL/ID of what changed).

### Execution guardrails (apply to ALL action types)

- **Every execution must show the exact diff/text applied.** Post it as part of the confirmation.
- **Catch exceptions from MCP calls.** If a call fails, mark the offer status `errored` (not `applied`), log the error, include in the confirmation with a 🚨 icon.
- **Idempotency**: before executing, check if the target already has the change (e.g. comment text already present on the Jira ticket, task already marked done in tasks.md). If yes, skip with status `noop` and a `⏭ Already done — skipped` message.
- **Rate limit**: if more than 10 approved offers in a single poll run, execute the first 10 and defer the rest to the next poll with a message `⏸ 10 applied this cycle — N more pending; will continue next poll`.

## Step 6 — Scan new signals (since last poll run)

Window = `config.poll_interval` + 5 min overlap (so nothing falls between polls).

Run in parallel (same techniques as morning brief — see that skill for MCP details and quirks):

- **Zoom new meetings**: `search_meetings` with `from` = (last poll ts) and `to` = now UTC. For each with `has_summary: true`, `get_meeting_assets` and parse next-steps. Classify by owner (user / team / skip).
- **Gmail Jira "work due" digests**: `search_threads` with `subject:"you have work due in Jira" newer_than:1d`. For each hit: get full content, extract ticket blocks, dedup against `tasks.md`, add new ones as monitoring tasks under matching goal.
- **Slack watch channels**: for each in `config.watch_channels`, `slack_read_channel` without `oldest`, filter by ts.
- **Slack DMs from team**: for each in `config.team`, `slack_read_channel` with user ID as channel_id, filter by ts.
- **Saved messages**: `slack_search_public_and_private` with `from:<@{user_id}> is:saved after:<yesterday>`.

### 6a. Dedup (apply aggressively)

- If signal references a Jira key or URL already in `tasks.md` → skip.
- If signal text fuzzy-matches an existing task (>70% similarity) → skip.
- If signal was already flagged in a prior brief (check recent brief messages for same link/text) → skip.
- **Err on the side of NOT adding.** A missed signal resurfaces next run; clutter is harder to undo.

### 6b. Classify each new signal

- **Action-required for user** → candidate for a new work offer (Step 7) OR a new task in tasks.md (if not delegable to this plugin).
- **Action for user's team** → new `[MONITORING]` task under matching goal. Owner = named person.
- **Informational** → skip entirely.

## Step 7 — Emit new work offers (if any)

For signals that produced delegable asks matching the action allowlist (see morning-brief Step 6), formulate offers and post them as a thread reply to the most recent active brief thread.

**Number offers starting at 1 for this poll cycle** (NOT continuing from morning brief's numbering). Scope is the poll's own post — users' replies to this poll message reference these new numbers.

Format (same as morning brief):
```
🤝 *Work I can do for you* — reply here to approve

1. [Short verb + target] — [one-line context]
2. ...

Reply with `apply 1` / `skip 2` / `show 3` (to preview) / `edit 1: <new text>`.
I'll pick up your reply on the next poll. ⚡ Need it sooner? Reply, then run `/brief-run poll` in Claude Code.
```

Append corresponding entries to `.offers.jsonl` with this poll's `brief_thread_ts` + new `offer_number` sequence.

**If no new offers and no executions in Steps 5–6, and no natural-language task changes: DO NOT POST ANYTHING.** Silent polls are expected and fine.

## Step 8 — Post consolidated confirmation (if anything happened)

If ANY of the following occurred, post a thread reply to the relevant brief thread. Consolidate into ONE message (not one per action):

```
✅ *Poll check — <HH:MM> — here's what changed*

*Applied*
• ✓ [confirmation for offer 1] — <link>
• ✓ [confirmation for offer 2] — <link>

*Drafted* (ready for your review)
• 📝 [what was drafted + where to find it]

*Task board*
• ➕ Added: <task> (<goal>)
• ✅ Done: <task> (<goal>)
• ✏️ Updated: <task> (<goal> — what changed)

*Errors* (if any)
• 🚨 [what failed, why]

*Flagged for your review* (ambiguous — correct me next cycle)
• [what you said → what I guessed]

*New signals since last poll*
• [Summary of new monitoring items added to tasks.md, if any]
```

**Plain English only.** Do NOT reference internal pattern codes, offer action_types, or ledger fields. Describe what changed as a colleague would.

Cap the message at 2500 chars. If longer, collapse with `…plus N more`.

## Step 9 — State log

Append one line to `.poll-log.jsonl`:
```json
{"ts": "<ISO>", "brief_threads_found": N, "approvals_processed": N, "offers_executed": N, "task_changes": N, "new_offers_posted": N, "new_signals": N, "errors": [...], "posted_summary": true|false}
```

Used for the "detect user's direct edits" check next run, and for debugging.

## Hard constraints

- Never modify `tasks.md` without also posting a confirmation.
- Never post a confirmation if nothing actually happened.
- Cap Slack summary at ~2500 chars.
- Respect the user's explicit words. "critical" → CRITICAL, "tomorrow" → tomorrow's date.
- Don't change goal-level structure (rename, reorder, add goals) unless the user explicitly asked.
- Plain English in user-facing output. No internal jargon (no "Pattern A", no "action_type: task_add").
- Every executed action must show what was applied (diff / comment text / task text) — the user should be able to catch mistakes immediately.
- If an MCP errors mid-execution, mark `errored` and continue with the others. Don't let one failure block the rest.
- Report back a ≤200-word debug summary after the run.
