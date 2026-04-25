# Sales — Role Preset

Pre-filled defaults for someone in sales (rep, AE, sales lead, sales ops, etc.). `/briefs:setup` layers your specifics on top.

## Suggested VIP tiers
- CEO / Founder
- CRO / VP Sales
- Key strategic customers (top 3 accounts)

## Suggested watch-channel types
- Your sales team channel (where reps update on deals)
- Pipeline / forecast review channel
- Exec / leadership channel
- Any customer-escalation channel (CS often pings sales on renewals at risk)

## Suggested team members / collaborators to DM-scan
- Direct-report AEs and SDRs
- Sales engineers you work with
- Your sales ops partner

## Integration defaults
- calendar: on (required)
- gmail: on — weight client-domain emails higher than internal
- zoom: on — prospect/customer calls are your primary intel source
- jira: off (most sales orgs don't use it) — enable if you track deal-level implementation in Jira
- hubspot: on — deal-stage changes and rep-owned notes are core signal
- additional: Salesforce works well here too — add it in setup with a description like "open opportunities I own with stage changes today"

## What the briefs prioritize for you
- **Morning**: deals with stage changes in last 24h, customer emails awaiting reply, prospect call next-steps
- **Poll**: new meeting assets from prospect/customer calls, escalations from CS, VIP pings
- **EOD**: deals progressed, at-risk deals, tomorrow's top-3 focus meetings (or Monday's if it's Friday)

## Recommended poll interval
60 min — sales is responsive but not frantic. Bump to 30 min during quarter-end crunch.

## What work offers tend to look like for sales
- "Add a note to HubSpot deal X recapping today's discovery call"
- "Draft a follow-up Slack reply to your VP Sales on the forecast ask"
- "Add a task: send MSA redlines to legal by Friday"

## Explicit cautions
HubSpot actions are restricted to read-only notes. The plugin NEVER changes deal stage, amount, owner, or close date. If you try to approve an offer that implies a stage change, the poll aborts it with a warning. Same restriction applies to any additional integrations like Salesforce.
