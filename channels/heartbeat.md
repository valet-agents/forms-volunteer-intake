# Volunteer Intake Sweep (Heartbeat)

The heartbeat channel fires once a day. There is no payload
to parse — your job is to find new responses on the volunteer
intake Google Form since the last sweep, parse them, score team
matches, and post a card to Slack.

## What it does

1. Resolve the form per the SOUL **Phase 1** rules: prefer
   `VOLUNTEER_FORM_ID`; otherwise auto-discover the first
   OAuth-listed form whose title contains `Volunteer`, `Sign Up`,
   `Signup`, or `Intake`. Cache the resolved id into MEMORY.md.
2. Read MEMORY.md for `last_seen_response_id` and
   `last_seen_submitted_time`. List form responses via
   `google-forms` sorted by submission time descending and take
   any newer than the watermark, capped at 5 per fire so a
   backlog can't stampede a channel.
3. For each new response (oldest first), parse name, email,
   skills, availability, and preferred team per SOUL **Phase 3**.
   Mask the email before any output: `m***@example.com`.
4. Score team matches per SOUL **Phase 4** using the `TEAM_NEEDS`
   env var (a JSON map of team → keyword list), falling back to
   `TEAMS`, then to Slack channel naming convention. Take the
   **top 2** teams by score. Drop ties at zero. If nothing
   scores above zero, mark the card `Needs human triage` —
   never reject a volunteer.
5. Compose a volunteer card per SOUL **Phase 5** — Slack
   `mrkdwn`, under 1,500 chars, masked email, top 2 suggested
   teams, link to the form response.
6. **Smart routing — post to the matching team's channel(s)
   only**: derive each invited channel's team label from its
   name (strip `team-`/`vol-`/`volunteers-`/`nonprofit-`) and
   send the card only to channels whose label matches one of
   the suggested teams. If zero invited channels match the
   suggested teams, fall back to posting to **every** invited
   channel with a `Needs review — no team match` tag at the
   top of the card. Never broadcast a matched volunteer to
   every invited channel. If a post fails for a particular
   channel, log and continue with the others — do not retry.
7. Update MEMORY.md to the newest response processed (response
   id + `submitted_time`). The next fire reads from there.

## MEMORY.md state shape

The agent persists a small block in MEMORY.md to track what's
been processed. Shape:

```
## forms-volunteer-intake

form_id: 1aBcD...XYZ
last_seen_response_id: ACYDBNh...
last_seen_submitted_time: 2026-05-05T17:42:00.000Z
```

Update this block in place each fire. Do not append a new block
per fire. This is the contract that prevents posting the same
volunteer twice.

## Where to post

Per SOUL **Where to post** (smart routing):

- For matched volunteers, post **only** to invited channels
  whose name resolves to one of the top suggested teams (strip
  `team-`/`vol-`/`volunteers-`/`nonprofit-` prefixes and match
  case-insensitively).
- For unmatched volunteers (no team scored above zero, or no
  invited channel maps to any suggested team), post to **every**
  invited channel with a `Needs review — no team match` tag.
- If the bot is in zero channels, DM the workspace install user
  with the volunteer card and a one-line invite hint that
  mentions per-team channel routing.

## Skip conditions

Skip posting (and stop silently) if any of these are true:

- The form can't be resolved (no `VOLUNTEER_FORM_ID` set and no
  OAuth-listed form whose title contains `Volunteer`, `Sign Up`,
  `Signup`, or `Intake`). Write `form: not_found` to MEMORY.md
  and stop. Do not post.
- This is the first run after deploy (no watermark in
  MEMORY.md). Seed the watermark to the most-recent existing
  response and stop — the backlog is not surfaced.
- Zero responses are newer than the watermark. Stay silent — no
  "no new volunteers" post.
- A response has no answers at all (a partially submitted form
  glitch). Skip that response, log it to MEMORY.md, and advance
  the watermark past it so it isn't retried forever.
