# Engineering — Role Preset

Pre-filled defaults for someone in engineering (engineer, tech lead, EM, director, VP Eng). `/briefs:setup` layers your specifics on top.

## Suggested VIP tiers
- CEO / Founder
- CTO / VP Eng (if above you)
- Head of Product (eng-product partnership)
- Head of Security / Infra (depending on domain)

## Suggested watch-channel types
- Your team's channel (where engineers discuss day-to-day)
- Releases / deploys channel
- Incidents / oncall channel
- Eng-leadership channel
- PR review channel (if your org has one)

## Suggested team members / collaborators to DM-scan
- Engineers on your team
- Your tech lead(s)
- Your partner PM

## Integration defaults
- calendar: on (required)
- gmail: on — less central for eng but PR reviews + vendor emails land here
- zoom: on — planning, 1:1s, incident retros
- jira: on (engineering-specific projects)
- hubspot: off
- additional: GitHub if your org has a Claude Code MCP for it; Linear if you use Linear instead of Jira

## What the briefs prioritize for you
- **Morning**: open PRs awaiting your review, incidents from overnight, sprint burn-down, 1:1s today
- **Poll**: new PR reviews requested, new incidents, Jira status changes on critical tickets
- **EOD**: sprint progress, incidents resolved, tomorrow's top 3 (often: unblock / review / ship). On Friday: weekend send-off + Monday's top 3.

## Recommended poll interval
60 min — or 30 min during release week or active incident.

## What work offers tend to look like for engineering
- "Add a comment to ENG-1234 — 'Reviewed the design, LGTM — please also add a deprecation warning'"
- "Update Confluence runbook 'Oncall Rotation' — add this week's rotation changes"
- "Draft a Slack reply to your CEO on the architecture decision question"
- "Mark task X done in tasks.md"

## Explicit cautions
- The plugin does NOT integrate with GitHub PRs natively in v1 — add it as an additional integration if your org has a GitHub MCP. PR signals also come through Slack/Gmail notifications if your team pipes them there.
- Jira actions are limited to adding comments. No status changes, no assignee changes.
