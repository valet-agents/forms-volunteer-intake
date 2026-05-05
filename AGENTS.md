This folder contains the source for a Skilled Agent originally built for the Valet runtime. Changes should follow the Skilled Agent open standard.

## Setup

### Connectors

- **google-forms**: The Google Forms API, OAuth-authenticated. The agent uses it to list forms (when `VOLUNTEER_FORM_ID` is unset), read responses on the intake form, and parse fields for skills, availability, and preferred team. Read-only — the agent never writes back to Google Forms. Add it from the catalog at the org level so other Forms-powered agents can share it.

### Channels

- **slack** (slack): The agent's per-agent Slack bot. Listens for @mentions and replies in-thread, posts volunteer cards to whichever channels the bot has been invited to, and (on confirmation) DMs team leads. Slack writes use the auto-injected outbound Slack connector.
- **heartbeat** (heartbeat): Wakes the agent every 5 minutes to sweep the volunteer intake form for new responses since the last watermark in MEMORY.md, parse them, score team matches, and post a card. Declared inline in `valet.yaml`, so it's created automatically by the dashboard setup flow.

### Secrets

This agent uses the OAuth variant of Google Forms, so no API token is needed at the org or agent level. The OAuth grant happens in the dashboard setup flow when you connect Google.

### External Setup

1. Complete the Google OAuth grant in the dashboard setup flow. The granted account must have read access to the volunteer intake form.
2. Set the agent's env vars so it knows what to watch and how to route:
   - `VOLUNTEER_FORM_ID` — the Google Form id of the intake form. If unset, the agent auto-discovers the first form whose title contains `Volunteer`, `Sign Up`, `Signup`, or `Intake`.
   - `TEAMS` — comma-separated team labels the agent should match against (e.g. `Build Crew, Tutoring, Outreach, Accessibility Testing`). If unset, the agent derives team labels from Slack channel names prefixed `team-`, `vol-`, or `volunteers-`.
   - `TEAM_DAYS` *(optional)* — JSON or `Team=Mon|Wed|Sat`-style map giving the days each team most needs help. Adds a +1 match score when the volunteer's availability covers one of those days.
   - `LEAD_DMS` *(optional)* — `Team=SlackUserId` map (e.g. `Build Crew=U012ABCDE,Tutoring=U045FGHIJ`). Required only if you want the agent to be able to DM team leads on confirmation.
3. After deploy, invite the agent's Slack bot to whichever channel(s) you want new-volunteer cards in. The agent posts to every channel it's a member of. If the bot has not been invited anywhere, the card is sent as a DM to the workspace install user with a one-line nudge to invite it somewhere.
4. Invite the bot to any additional channels where teammates should be able to @mention it for ad-hoc volunteer questions (e.g. an ops channel for "any new volunteers for the Saturday build?" follow-ups).
5. The first heartbeat fires within 5 minutes of deploy. The first run only seeds MEMORY.md with the most-recent existing response — no card is posted because the backlog is intentionally not surfaced. Subsequent heartbeats post cards for forward-looking responses only.

## Customizing

- **Change the heartbeat interval**: edit the `every` value on the `heartbeat` channel in `valet.yaml` (e.g. `2m`, `15m`), then redeploy. Faster intervals mean faster routing but more Forms API traffic.
- **Tune team-match heuristics**: SOUL.md's *Phase 4: Match to teams* defines the scoring (preferred-team match `+3`, skill keyword overlap `+1`, availability-day overlap `+1`). Edit that section — and the small synonym map it references — to make the agent's suggestions louder, quieter, or more domain-specific.
- **Watch a different form**: change the `VOLUNTEER_FORM_ID` env var. The next heartbeat re-resolves the form and re-seeds the watermark for the new form.
- **Enable lead DMs**: set `LEAD_DMS` to the team-to-Slack-user-id map. Without it, the agent only posts cards to channels and answers questions in-thread; the DM-the-lead path is disabled by design.
