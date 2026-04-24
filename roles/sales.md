# Sales Manager — Role Preset

Pre-filled defaults for a Sales Manager / Director / VP role. `/brief-setup` layers your specifics on top of this.

## Suggested VIP tiers
- CEO / Founder
- CRO / VP Sales
- Key strategic customers (top 3 accounts)

## Suggested watch-channel types
- Your sales team channel (where reps update on deals)
- Pipeline / forecast review channel
- Exec / leadership channel
- Any customer-escalation channel (CS often pings sales on renewals at risk)

## Suggested team members to DM-scan
- Your direct-report AEs and SDRs
- Sales engineers you work with
- Your sales ops partner

## Integration defaults
- calendar: on (required)
- gmail: on — weight client-domain emails higher than internal
- zoom: on — prospect/customer calls are your primary intel source
- jira: off (most sales orgs don't use it) — enable if you track deal-level implementation in Jira
- hubspot: on — deal-stage changes and rep-owned notes are core signal

## What the briefs prioritize for you
- **Morning**: deals with stage changes in last 24h, customer emails awaiting reply, prospect call next-steps
- **Poll**: new meeting assets from prospect/customer calls, escalations from CS, VIP pings
- **EOD**: deals progressed, at-risk deals, tomorrow's top 3 focus meetings

## Recommended poll interval
60 min — sales is responsive but not frantic. Bump to 30 min during quarter-end crunch.

## What work offers tend to look like for sales
- "Add a note to HubSpot deal X recapping today's discovery call"
- "Draft a follow-up Slack reply to @VP Sales on the ACME forecast ask"
- "Add a task: send MSA redlines to legal by Friday"

## Explicit cautions
HubSpot actions are restricted to read-only notes. The plugin will NEVER change deal stage, amount, owner, or close date. If you try to approve an offer that implies a stage change, the poll will abort it with a warning.
