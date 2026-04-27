---
name: tasks
description: Answers conversational questions about the user's tasks, priorities, workload, AND what's owed to them by teammates (monitoring/waiting-on items). Auto-fires when the user asks "what should I prioritize today?", "what's overdue?", "what do I need to do next?", "what's on my plate this week?", "what did I get done?", "show me everything for [goal]", "how's my workload?", "what am I waiting on?", "what am I watching?", "what does [name] owe me?", "what is owed to me today/this week?", or anything related to their personal task management. Reads the user's tasks.md managed by the briefs plugin. Answers in plain English, no JSON.
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

## Step 1.5 — Classify every task: yours-to-do vs. yours-to-watch

Every task in tasks.md falls into one of three buckets. Apply this classification once when you parse the file; reuse it across every query below.

1. **Yours to do** — task `Owner:` field includes the user (matches `display_names` from config, OR the literals "You" / "Will" / "Me" / first-name match). These are items *you* execute personally.

2. **Yours to watch** — task `Owner:` does NOT include the user, AND ONE OR BOTH of:
   - Parent goal heading is `## MONITORING:` (instead of `## GOAL:`), OR
   - Task priority tag is `[MONITORING]`
   
   These are items you're tracking — direct reports' deliveries, teammate work you need to react to, customer escalations someone else is fixing, items waiting on confirmation/approval/reply.

3. **Neither** — task `Owner:` is someone else AND parent is `## GOAL:` AND priority is `[CRITICAL]/[HIGH]/[MEDIUM]/[LOW]`. Usually means the task is stale or mis-tagged (someone else's work parked in a goal you own without the `[MONITORING]` tag). Don't surface in default views. Only return when the user explicitly asks "what's [name] working on?" or "find the task about X".

**Critical rule:** the owner check trumps the goal-level marker. A `## MONITORING: FullStory Replacement` goal can contain a task with `Owner: Will` — that task is **yours to do**, not yours to watch. The `## MONITORING:` heading is a default for that goal's tasks; per-task `Owner:` overrides it.

**Tag-level** `[MONITORING]` priority is the explicit per-task signal. A task with `Owner: Sebastian | [MONITORING]` inside a regular `## GOAL:` is a watching item.

## Step 2 — Classify the user's question

Match the user's intent to one of these patterns. Pick the closest fit; if ambiguous, ask one clarifying question.

### Priority / Top 3 / "what should I prioritize"
Keywords: "prioritize", "what should I do", "what's most important", "top X", "focus today", "what do I need to do next", "what's most urgent"

→ Return TWO parallel sections — your action items AND what you're watching today. Both drive daily decisions.

**🎯 Your work today** (yours-to-do bucket):
- Critical-priority items with a deadline in the next 48h
- Items where someone's blocked on the user
- Items overdue (Critical + High)

3–5 items, verb-first action + reason + deadline. If fewer than 3 qualifying, say so: "Looks light today — only 2 things stand out: …"

**👁 Owed to you / watching today** (yours-to-watch bucket):
- Watching items overdue (any priority/tag)
- Watching items due today
- Watching items due in the next 48h

Group by owner. Within each owner, order: overdue first, then earliest due date. Show as: `<Owner>: <task> — due <date>` or `<Owner>: <task> — overdue <N> days`. Cap at 5–7 items unless the user asks for more.

If no watching items qualify: omit the section silently (don't print "nothing to watch" — that's noise).

### Overdue
Keywords: "overdue", "past due", "behind on", "what's late"

→ Return TWO parallel sections.

**🔴 Yours overdue** (yours-to-do bucket): tasks where `Due:` is before today, grouped by goal. Order: Critical → High → Medium → Low. Include the due date and how many days overdue.

**⏳ Owed to you (overdue)** (yours-to-watch bucket): tasks owned by others where `Due:` is before today. Group by owner. Within owner, sort by days overdue (most overdue first). Format: `<Owner>: <task> — overdue <N> days (<goal>)`.

If both sections are empty: "Nothing overdue. ✓"
If only one section is empty: omit that section (don't print "nothing yours overdue" if you have plenty owed to you).

### Monitoring / waiting on / owed to you
Keywords: "what am I waiting on", "what am I watching", "what's owed to me", "what is owed to me today", "what is owed to me this week", "what does [name] owe me", "show me monitoring", "monitoring"

Also fires from the `/briefs:monitoring` slash command.

→ Return ONLY yours-to-watch items (no parallel "your work" section — this query is explicitly about what you're tracking).

Default scope: all open watching items. Narrow if the user implied a window:
- "today" / "owed to me today" → due today or overdue
- "this week" / "owed to me this week" → due in the next 7 days or overdue
- "what does Sid owe me?" → filter to `Owner: Sid` (or display variants)

**Output structure** — group by owner, sort owners by total-overdue-count desc:

```
👁 You're waiting on:

🧑 <Owner Name> (<N> items)
  • [PRIORITY] <task> — overdue <N> days (<goal>)
  • [PRIORITY] <task> — due <date> (<goal>)
  ...

🧑 <Other Owner> (<N> items)
  • ...
```

Within each owner: overdue first (most overdue → least), then upcoming (earliest due → latest). Show priority badge in `[BRACKETS]`. Show goal in parens for context. Cap at 15 items unless user asks for more.

If no watching items: "Nothing owed to you right now. ✓"

If a watching item has no `Due:` field: include it under that owner with "no due date" in place of the date — but rank below items with dates.

### Upcoming / this week / by deadline
Keywords: "this week", "next 7 days", "upcoming", "what's due", "deadlines", "what's coming up"

→ Return TWO parallel sections.

**🎯 Your work** (yours-to-do): tasks with `Due:` in the implied window (default 7 days; respect "today", "this week", "by Friday"). Group by due date. Show priority and owner.

**👁 You're watching for** (yours-to-watch): same window, same `Due:` filter, but yours-to-watch tasks. Group by owner. Show priority and due date. If a goal-level `## MONITORING:` has no due dates on its tasks, include only those with explicit `Due:`.

Omit either section if empty.

### Completed / what got done
Keywords: "done", "completed", "finished", "what did I ship", "this week's wins", "yesterday's progress"

→ Find `- [x]` checkbox tasks. If the file uses Notes with completion dates (e.g. `Note: Done Apr 22`), use those. Group by date. Default window: last 7 days unless the user specified.

### By goal / project / topic
Keywords: "show me everything for X", "what's open on X", "what's the status of [goal]", "everything related to X"

→ Filter to the matching goal section. Include open tasks (priority, owner, due) and completed ones (collapsed: "12 done"). If multiple goals match, ask which.

### Workload summary
Keywords: "how's my workload", "summary", "overview", "what's the state of things", "give me a snapshot"

→ Compute and return TWO blocks (yours-to-do, then yours-to-watch):

**🎯 Your work**
- Total open tasks (yours-to-do bucket)
- Counts by priority (Critical / High / Medium / Low)
- Top 3 goals by open-task count
- Overdue count
- Due this week count

**👁 What you're watching**
- Total open watching items
- Top 3 owners by item count ("Heghine: 5, Sebastian: 3, Marcelo: 2")
- Overdue count
- Due this week count

End with a one-line gut-check covering both: "You're heaviest on <goal> and waiting on <name> for <N> items" or "Calendar's clear-ish — manageable on both fronts."

### By owner
Keywords: "what's [name] working on", "what's blocking on [name]", "everything for [name]"

→ Filter to tasks where `Owner: <name>` matches (any task, regardless of bucket). Show open tasks with priority and due, grouped by goal. Mark watching items with a `👁` glyph for clarity. Helpful for 1:1 prep — shows everything that person is doing for you AND with you.

Note: "what am I waiting on from [name]?" or "what does [name] owe me?" routes to the **Monitoring / waiting on** query above (filtered to that owner). The distinction:
- "What's [name] working on?" → all their tasks (yours and theirs combined)
- "What does [name] owe me?" → only watching items owned by [name]

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
- After monitoring/waiting-on: "Want me to draft a nudge for any of these? Just say `draft a poke to <Owner> about <task>`."

Don't add a follow-up if the user's question was a simple lookup ("is there a task for X?") and you answered it.

## Hard constraints

- Read-only by default. Only modify tasks.md when the user explicitly asks (Modification pattern above).
- If the user's question references a goal/task that doesn't exist, say so honestly: "I don't see anything matching '<term>' in your tasks. Want me to add it?"
- Plain English. Never dump raw markdown lines from tasks.md verbatim — parse and reformat for readability.
- If reading tasks.md returns >2000 lines, parse efficiently — don't blow context on tasks the user didn't ask about. Filter first, then render.
- Respect the user's timezone (`config.timezone`) for all date comparisons. "Today", "tomorrow", "this Friday" resolve in their TZ, not UTC.
