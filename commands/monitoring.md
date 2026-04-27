---
description: "Show what's owed to you / what you're watching — yours-to-watch tasks across all goals, grouped by owner. Optional argument: a name (e.g. 'Sid') to filter, or 'today' / 'this week' to narrow the window."
argument-hint: '[name] | today | this week | (empty for all)'
---

You are surfacing the user's monitoring/waiting-on items from their tasks.md. The user's argument is: `$ARGUMENTS`

This is a thin wrapper over the `tasks` skill's "Monitoring / waiting on" query type. Invoke that skill with the user's argument routed as a monitoring query.

## Argument routing

- **Empty** → "what am I waiting on?" (all open watching items, all owners, no time filter)
- **A name** (e.g. `Sid`, `Heghine`, `Sebastian`) → "what does <name> owe me?" (filtered to that owner)
- **`today`** → "what is owed to me today?" (overdue + due today)
- **`this week`** → "what is owed to me this week?" (overdue + due in next 7 days)
- **Anything else** → pass through to the tasks skill as the user's literal question

## Before invoking

Read `~/.claude/daily-workflow-briefs/config.md`. If it doesn't exist or `tasks_file: none`, tell the user to run `/briefs:setup` first (or enable tasks via `/briefs:config`) and stop.

## Output

The tasks skill handles formatting. Don't re-decorate the response — just relay it. End with the same follow-up the tasks skill offers ("Want me to draft a nudge for any of these?").

## Why a slash command exists for this

Most users will type "what am I waiting on?" or "what does Sid owe me?" naturally — the tasks skill auto-fires on those. The slash command is for users who want a predictable trigger (script, keybind, Claude Code from a fresh terminal) and don't want to remember the natural-language phrasings.
