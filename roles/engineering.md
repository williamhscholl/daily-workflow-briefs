# Engineering Manager — Role Preset

Pre-filled defaults for an Engineering Manager / Director / VP Eng role. `/brief-setup` layers your specifics on top of this.

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

## Suggested team members to DM-scan
- Your direct-report engineers
- Your tech lead(s)
- Your partner PM

## Integration defaults
- calendar: on (required)
- gmail: on — less central for eng but PR reviews + vendor emails land here
- zoom: on — planning, 1:1s, incident retros
- jira: on (engineering-specific projects)
- hubspot: off

## What the briefs prioritize for you
- **Morning**: open PRs awaiting your review, incidents from overnight, sprint burn-down, 1:1s today
- **Poll**: new PR reviews requested, new incidents, JIRA status changes on critical tickets
- **EOD**: sprint progress, incidents resolved, tomorrow's top 3 (often: unblock / review / ship)

## Recommended poll interval
60 min — or 30 min during release week or active incident.

## What work offers tend to look like for engineering
- "Add a comment to ENG-1234 — 'Reviewed the design, LGTM — please also add a deprecation warning'"
- "Update Confluence runbook 'Oncall Rotation' — add this week's rotation changes"
- "Draft a Slack reply to @CEO on the architecture decision question"
- "Mark task X done in tasks.md"

## Explicit cautions
- The plugin does NOT integrate with GitHub PRs directly (v1). PR signals come from Slack/Gmail notifications if your team pipes them there. If you want first-class GitHub integration, file an issue.
- Jira actions are limited to adding comments. No status changes, no assignee changes.
