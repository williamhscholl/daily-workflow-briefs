# Customer Success — Role Preset

Pre-filled defaults for someone in CS (CSM, CS lead, CS ops, renewals, implementation). `/briefs:setup` layers your specifics on top.

## Suggested VIP tiers
- CEO / Founder
- VP CS / Chief Customer Officer
- CRO (renewals overlap)
- Top 3 at-risk or strategic accounts

## Suggested watch-channel types
- Your CSM team channel
- Customer escalations channel (often #cs-escalations or similar)
- Product bug-triage channel (CSMs file a lot of tickets here)
- Exec / leadership channel
- Customer-specific shared Slack Connect channels (if you use them)

## Suggested team members / collaborators to DM-scan
- Direct-report CSMs
- Renewals / implementation lead
- Your product liaison / partner PM

## Integration defaults
- calendar: on (required)
- gmail: on — customer emails and product-filed tickets are high signal
- zoom: on — customer QBRs, EBRs, escalation calls
- jira: on (projects: your bug/feature tracker, plus any CS-specific project like `CS` or `CLA`)
- hubspot: off by default — turn on if your org uses HubSpot for CS account tracking
- additional: Intercom or Zendesk are common — add either in setup with a description like "tickets I'm CC'd on, last 12h"

## What the briefs prioritize for you
- **Morning**: at-risk accounts with activity in last 24h, escalations from eng/product, customer emails awaiting reply
- **Poll**: new meeting assets from customer calls, new bug tickets filed by your team, VIP pings
- **EOD**: customer calls held, escalations closed or pending, tomorrow's customer meetings (or Monday's if it's Friday)

## Recommended poll interval
60 min — balanced. Escalations rarely can't wait an hour. If your team is in a crisis cycle, bump to 30 min.

## What work offers tend to look like for CS
- "Add a comment to CLA-718 — 'Still waiting on Sid's prioritization decision'"
- "Update Confluence page 'Customer Health Dashboard' — add new at-risk signal from today's call with Acme"
- "Draft a Slack reply to your VP CS on the renewal question"
- "Add a task: follow up with Acme on their NPS score by Friday"

## Explicit cautions
The plugin never changes Jira status (it can't close a ticket or move to "Done"). It only adds comments. For status changes, the plugin will draft the change as a task for you to execute manually. Same applies to additional integrations like Intercom/Zendesk — read-only, comments only.
