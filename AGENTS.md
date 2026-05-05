This folder contains the source for a Skilled Agent originally built for the Valet runtime. Changes should follow the Skilled Agent open standard.

## Setup

### Connectors

- **google-forms**: The Google Forms API, OAuth-authenticated. The agent uses it to list forms (when `VOLUNTEER_FORM_ID` is unset), read responses on the intake form, and parse fields for skills, availability, and preferred team. Read-only — the agent never writes back to Google Forms. Add it from the catalog at the org level so other Forms-powered agents can share it.

### Channels

- **slack** (slack): The agent's per-agent Slack bot. Listens for @mentions and replies in-thread, posts volunteer cards to whichever channels the bot has been invited to, and (on confirmation) DMs team leads. Slack writes use the auto-injected outbound Slack connector.
- **heartbeat** (heartbeat): Wakes the agent once a day to sweep the volunteer intake form for new responses since the last watermark in MEMORY.md, parse them, score team matches, and post a card. Declared inline in `valet.yaml`, so it's created automatically by the dashboard setup flow.

### Secrets

This agent uses the OAuth variant of Google Forms, so no API token is needed at the org or agent level. The OAuth grant happens in the dashboard setup flow when you connect Google.

### External Setup

1. Complete the Google OAuth grant in the dashboard setup flow. The granted account must have read access to the volunteer intake form.
2. Set the agent's env vars so it knows what to watch and how to route:
   - `FORM_ID` (or `VOLUNTEER_FORM_ID`) — the Google Form id of the intake form. If unset, the agent auto-discovers the first form whose title contains `Volunteer`, `Sign Up`, `Signup`, or `Intake`.
   - `TEAM_NEEDS` *(optional, recommended)* — a JSON map of team name → keyword list describing what each team needs help with. Example: `{"Build Crew":["paint","carpentry","saturday","lifting"],"Tutoring":["math","reading","ESL","kids"],"Outreach":["spanish","events","social media"]}`. The agent scores each volunteer against these keywords for routing. A simple `Team=keyword|keyword2,Other Team=...` form is also accepted.
   - `TEAMS` *(optional)* — fallback if `TEAM_NEEDS` is unset: comma-separated team labels (e.g. `Build Crew, Tutoring, Outreach, Accessibility Testing`). The team labels themselves are used as the keyword set. If both are unset, the agent derives team labels from Slack channel names prefixed `team-`, `vol-`, `volunteers-`, or `nonprofit-`.
   - `TEAM_DAYS` *(optional)* — JSON or `Team=Mon|Wed|Sat`-style map giving the days each team most needs help. Adds a +1 match score when the volunteer's availability covers one of those days.
   - `LEAD_DMS` *(optional)* — `Team=SlackUserId` map (e.g. `Build Crew=U012ABCDE,Tutoring=U045FGHIJ`). Required only if you want the agent to be able to DM team leads on confirmation.
3. **Per-team channel routing** (recommended): invite the agent's Slack bot to **one channel per team** (e.g. `#team-build-crew`, `#vol-tutoring`, `#volunteers-outreach`). The agent derives the team label from each channel's name and posts each new volunteer card to **only the matching team's channel(s)** — not to every channel it's been invited to. If a volunteer doesn't match any team, the card falls back to **all** invited channels with a `Needs review — no team match` tag.
4. If you only have one volunteers channel, that's fine too — invite the bot there and every volunteer card lands in that one channel. Smart routing only kicks in when multiple team-named channels are invited.
5. Invite the bot to any additional channels (e.g. an ops channel) where teammates should be able to @mention it for ad-hoc questions like *"any new volunteers for the Saturday build?"*. Q&A replies are routed back to the originating channel/thread; they're not affected by smart routing.
6. The first heartbeat fires within 5 minutes of deploy. The first run only seeds MEMORY.md with the most-recent existing response — no card is posted because the backlog is intentionally not surfaced (cold-start rule). Subsequent heartbeats post cards for forward-looking responses only.

## Customizing

- **Change the heartbeat interval**: edit the `every` value on the `heartbeat` channel in `valet.yaml` (e.g. `1h`, `12h`), then redeploy. The default `24h` routes new volunteers once a day; faster intervals mean faster routing but more Forms API traffic.
- **Tune the `TEAM_NEEDS` map**: this is the main routing knob. Add the keywords your teams actually use ("saturday", "spanish", "carpentry") so volunteers get matched on real signal rather than the team label alone. Update it as your needs change — no redeploy needed if your runtime hot-reloads env vars; otherwise redeploy.
- **Tune team-match heuristics**: SOUL.md's *Phase 4: Match to teams* defines the scoring (preferred-team match `+3`, keyword overlap `+1`, availability-day overlap `+1`). Edit that section to make the agent's suggestions louder, quieter, or more domain-specific.
- **Tune the smart-routing fallback**: SOUL.md's *Where to post* defines the fallback rule (no team match → post to all invited channels with a `Needs review` tag). If you'd rather hold unmatched volunteers in a single triage channel, name that channel `#team-needs-review` (or similar) and add it to `TEAM_NEEDS` so the matcher routes there explicitly.
- **Watch a different form**: change the `FORM_ID` (or `VOLUNTEER_FORM_ID`) env var. The next heartbeat re-resolves the form and re-seeds the watermark for the new form.
- **Enable lead DMs**: set `LEAD_DMS` to the team-to-Slack-user-id map. Without it, the agent only posts cards to channels and answers questions in-thread; the DM-the-lead path is disabled by design.
