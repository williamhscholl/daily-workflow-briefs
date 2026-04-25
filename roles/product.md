# Product — Role Preset

Pre-filled defaults for someone in product (PM, lead PM, group PM, head of product). `/briefs:setup` layers your specifics on top.

## Suggested VIP tiers
- CEO / Founder
- Head of Engineering / CTO
- VP Product (if you have one above you)
- Head of CS (product-CS feedback loop)

## Suggested watch-channel types
- Your product team channel (where PMs discuss work)
- Engineering channel (sprint updates, releases)
- Cross-functional channel with CS + Sales
- Exec / leadership channel
- Any channel where customer feedback lands (often #feedback, #customer-voice)

## Suggested team members / collaborators to DM-scan
- PMs you work with
- QA lead
- Technical writer (docs partner)
- Engineering leads you work with most
- Design lead

## Integration defaults
- calendar: on (required)
- gmail: on — PRD reviews, Confluence notifications, Jira digests
- zoom: on — planning meetings, 1:1s, customer discovery calls
- jira: on (all projects) — this is your primary signal source
- hubspot: off (most product folks don't need it)

## What the briefs prioritize for you
- **Morning**: Jira tickets needing PM review, overdue roadmap items, meetings today with context, Zoom next-steps from yesterday's meetings
- **Poll**: new Jira tickets escalated to PM status, new meeting assets, new PRD comments
- **EOD**: decisions made, tickets moved, tomorrow's top 3 (often: ship / unblock / decide). On Friday: weekend send-off + Monday's top 3.

## Recommended poll interval
60 min — product cadence is usually not urgent hour-to-hour. 30 min if you're in a launch week.

## What work offers tend to look like for product
- "Update Confluence PRD 'Calendar Revamp' — append the sync direction decision from today's call"
- "Add a comment to AI-621 — 'Kill-switch deployed tonight per Jordan's confirmation'"
- "Draft a Slack reply to your CEO on the Product Strategy question"
- "Mark task X done in tasks.md"

## Jira quirks
- The plugin quotes reserved JQL words automatically (`"INT"`, `"OR"`, etc.).
- It caps result counts to avoid 50k+ char responses.
- If you have >6 project keys, consider scoping to your most-active 3–4 in `/briefs:setup`.

## Explicit cautions
Jira actions are restricted to adding comments. The plugin never changes status, assignee, or priority — those require you to act via the Atlassian UI or MCP directly.
