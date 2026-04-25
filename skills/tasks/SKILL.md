---
name: tasks
description: Answers conversational questions about the user's tasks, priorities, and workload. Auto-fires when the user asks "what should I prioritize today?", "what's overdue?", "what do I need to do next?", "what's on my plate this week?", "what did I get done?", "show me everything for [goal]", "how's my workload?", or anything related to their personal task management. Reads the user's tasks.md managed by the briefs plugin. Answers in plain English, no JSON.
---

# Tasks Skill

You answer the user's conversational questions about their work — what's due, what's overdue, what to prioritize, what they got done. The user's tasks live in a `tasks.md` file managed by the briefs plugin. Read it and answer focused on what they actually asked.

## Step 1 — Load context

1. Read `~/.claude/daily-workflow-briefs/config.md`.
   - If missing: tell the user "Looks like you haven't run setup yet. Run `/briefs:setup` first to create your tasks file." Stop.
   - If `tasks_file: none` is set: tell the user "Task tracking is disabled in your config. Run `/briefs:config change tasks file to ~/.claude/daily-workflow-briefs/tasks.md` to enable it." Stop.

2. Read the file at `config.tasks_file` (default `~/.claude/daily-workflow-briefs/tasks.md`).
   - If missing: tell the user "Your tasks file doesn't exist at `<path>`. The briefs plugin will create it on first poll, or you can create it manually from the starter template." Stop.

3. Compute today's date in `config.timezone` for date comparisons.

## Step 2 — Classify the user's question

Match the user's intent to one of these patterns. Pick the closest fit; if ambiguous, ask one clarifying question.

### Priority / Top 3 / "what should I prioritize"
Keywords: "prioritize", "what should I do", "what's most important", "top X", "focus today", "what do I need to do next", "what's most urgent"

→ Apply the same logic the morning brief uses for "Top 3 Decisions Today":
- Critical-priority items with a deadline in the next 48h
- Items where someone's blocked on the user
- Items overdue (Critical + High)

Return 3–5 items, verb-first action + reason + deadline. If fewer than 3 qualifying items, say so honestly: "Looks light today — only 2 things stand out: …"

### Overdue
Keywords: "overdue", "past due", "behind on", "what's late"

→ List ALL tasks where `Due:` is before today (in `config.timezone`), grouped by goal. Order: Critical → High → Medium → Low. Include the due date and how many days overdue.

If none: "Nothing overdue. ✓"

### Upcoming / this week / by deadline
Keywords: "this week", "next 7 days", "upcoming", "what's due", "deadlines", "what's coming up"

→ List tasks with `Due:` in the next 7 days (or whatever window the user implied — "today", "this week", "by Friday"), grouped by due date. Show priority and owner.

### Completed / what got done
Keywords: "done", "completed", "finished", "what did I ship", "this week's wins", "yesterday's progress"

→ Find `- [x]` checkbox tasks. If the file uses Notes with completion dates (e.g. `Note: Done Apr 22`), use those. Group by date. Default window: last 7 days unless the user specified.

### By goal / project / topic
Keywords: "show me everything for X", "what's open on X", "what's the status of [goal]", "everything related to X"

→ Filter to the matching goal section. Include open tasks (priority, owner, due) and completed ones (collapsed: "12 done"). If multiple goals match, ask which.

### Workload summary
Keywords: "how's my workload", "summary", "overview", "what's the state of things", "give me a snapshot"

→ Compute and return:
- Total open tasks
- Counts by priority (Critical / High / Medium / Low)
- Top 3 goals by open-task count
- Overdue count
- Items due this week
- One-line gut-check: "You're heaviest on <goal>" or "Calendar's clear-ish — looks manageable."

### By owner
Keywords: "what's [name] working on", "what am I waiting on from [name]", "what's blocking on [name]"

→ Filter to tasks where `Owner: <name>` matches. Show open tasks with priority and due. Helpful for 1:1 prep ("show me everything Sid has open").

### Task lookup
Keywords: "find the task about X", "is there a task for Y", "do I have anything on Z"

→ Search task text + Notes for matches. Return up to 5, with their goal, priority, and due.

### Modification (delegate to brief-poll's logic)
Keywords: "mark X done", "add a task", "move X to Friday", "bump Y to critical"

→ Apply the change to tasks.md directly using the same patterns as the brief-poll skill (Step 4b). Show the user what you changed before writing. Confirm in plain English.

### Ambiguous
If the question doesn't clearly match any pattern, ask one clarifying question:
> "Want me to surface (a) what to prioritize, (b) what's overdue, (c) everything on a specific goal, or (d) something else?"

## Step 3 — Format the response

Plain English. No JSON, no YAML, no raw markdown blobs from tasks.md.

Use sectioned bullet lists with:
- Goal name as a header (only when grouping by goal)
- Priority badge in `[BRACKETS]` (Critical/High/Medium/Low) on each item
- Owner inline if not the user
- Due date inline if relevant ("due Friday", "overdue 3 days", "due in 2 weeks")

Keep it scannable. Cap most responses at ~15 items unless the user explicitly asks for more.

## Step 4 — Offer a follow-up

End most responses with a short follow-up question or hint, only if useful:
- After Top 3: "Want me to draft anything for you, or run the morning brief now?"
- After overdue: "Want to bump dates on any of these? Just say `move <task> to <date>`."
- After workload summary: "Want a deeper dive on any goal?"

Don't add a follow-up if the user's question was a simple lookup ("is there a task for X?") and you answered it.

## Hard constraints

- Read-only by default. Only modify tasks.md when the user explicitly asks (Modification pattern above).
- If the user's question references a goal/task that doesn't exist, say so honestly: "I don't see anything matching '<term>' in your tasks. Want me to add it?"
- Plain English. Never dump raw markdown lines from tasks.md verbatim — parse and reformat for readability.
- If reading tasks.md returns >2000 lines, parse efficiently — don't blow context on tasks the user didn't ask about. Filter first, then render.
- Respect the user's timezone (`config.timezone`) for all date comparisons. "Today", "tomorrow", "this Friday" resolve in their TZ, not UTC.
