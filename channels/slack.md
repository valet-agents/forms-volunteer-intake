# Slack Message Received

The Slack event payload is appended directly after these
instructions in the user message. Parse it inline — do not fetch,
list, or search for the payload elsewhere. Do NOT use tools to
read the payload.

## Quick Filter — Exit Early If Not Relevant

Before doing anything else, check whether this message is worth
responding to. **Stop immediately and take no action** if ANY of
these are true:

- The message is from a bot (check for `bot_id` or
  `subtype: "bot_message"` in the payload).
- The message is from yourself.
- The message is a channel join/leave, topic change, pin, or other
  system event (any non-empty `subtype` that isn't a real user
  message).
- The message body, after stripping your @mention, is empty or
  just a greeting / thank-you / emoji.
- You're not @mentioned and the message isn't in a thread you
  already replied in.

If you are unsure whether the message is relevant, err on the side
of NOT responding.

## Scope

Extract the `channel` and `ts` (or `thread_ts`) from the payload.
All replies MUST go to this channel and thread. Do not read or
act on messages from other channels or threads.

## Steps

1. Extract `channel`, `ts`, `thread_ts` (if present), `user`, and
   `text` from the event payload.
2. Apply the Quick Filter above. If the message fails the filter,
   **stop here — do nothing**.
3. Strip your @mention token from `text` to get the raw question.
4. Decide if this is a **read** (a question about the volunteer
   form, sign-ups, skills, or availability) or a **write** (asks
   you to DM a team lead with a volunteer card — the only write
   action this agent supports). When ambiguous, treat it as a
   read and ask one clarifying question.
5. For reads: pick the smallest set of `google-forms` queries
   that answer the question. Format the result per the SOUL
   "Read-only questions" guidance — short, masked-email bullets
   with the form-response link. Cap suggested teams to 2 in any
   card you compose.
6. For writes (DM a team lead): resolve the lead from the
   `LEAD_DMS` env var. If unresolvable, say so in-thread and
   stop. Otherwise restate the proposed DM in one line and wait
   for an explicit confirmation (👍, "yes", "go", "do it") in
   the same thread before executing. Reply with `Sent.` and the
   lead's `<@USERID>` only after executing.
7. Reply in the thread using `thread_ts` if present, otherwise
   `ts`. One reply per mention.
