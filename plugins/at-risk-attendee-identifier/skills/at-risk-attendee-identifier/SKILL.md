---
name: at-risk-attendee-identifier
description: Identify registered vFairs attendees with little or no engagement mid-event — segmented into "never logged in" vs. "logged in but didn't visit a booth or session" — so organizers can trigger the right re-engagement action for each group. Use this skill whenever the user asks who hasn't engaged, who's at risk of a no-show, wants a re-engagement or win-back list, or asks which registrants never logged in / never visited a booth or session. Requires the vFairs MCP connector (get_events, get_attendees, get_attendee_activity_data).
---

# At-Risk Attendee Identifier

Finds registered attendees with zero or minimal engagement mid-event, and splits them into two segments that need different fixes:
- **Never logged in** — never accessed the event platform at all.
- **Logged in, no engagement** — logged in but never visited a booth or attended a session.

These are different problems with different fixes: a "never logged in" attendee needs a login-link/reminder nudge; a "logged in, no engagement" attendee needs to be pointed at *something specific* (a booth, a session) because the platform itself isn't the barrier.

Scope: **active, registered attendees only.** Attendees with an inactive registration or a cancelled/rejected `user_reg_status` are a data-quality problem, not an engagement one — that's [`registration-health-check`](../../../registration-health-check/skills/registration-health-check/SKILL.md)'s job, not this skill's. Don't mix the two.

## When to use this skill

Trigger on requests like:
- "Who hasn't engaged with [event] yet?"
- "Which attendees are at risk of no-showing?"
- "Give me a re-engagement / win-back list for [event]"
- "Who registered but never logged in?"
- "Who's logged in but hasn't visited any booths or sessions?"

This skill is built to run **mid-event**, while there's still time to nudge people — flag that upfront if the event hasn't started yet (see Step 2).

## Output format — read this before doing anything else

**This skill is table and charts first. Every response must contain at least two markdown tables and a chart if possible.** Organizers act on a segmented list; they don't read paragraphs describing who's disengaged.

Rules:
- Never write a statistic in prose that could be a table row.
- Find opportunities to create charts when possible — a pie or bar chart of Engaged / Logged In No Engagement / Never Logged In works well here.
- Prose is for the verdict and the recommendation only — the "so what," not the "what."
- Minimum tables per response: the **Engagement Segments** table (Step 3). Add a named list table for a segment whenever the user asks for specifics, or whenever a segment is small enough (under ~30) to list in full usefully.
- Put the headline verdict in one or two sentences *above* the first table, then the tables, then a short "What to do next" list.
- Use bold status markers in table cells: **Never logged in** / **Logged in, no engagement** / **Engaged**.

## Workflow

### Step 1 — Resolve the event

Always ask which event this is for if it isn't already named in the request — never default to "most recent" or "all events." Use `get_events` (filter by `name`) to resolve to an `app_id`. If more than one event matches, list them as a table (Event Name | Start Date | Type) and ask the user to pick one.

### Step 2 — Sanity-check timing

Check the event's start/end dates from Step 1 against today.
- If the event **hasn't started yet**, say so plainly and stop before running the segmentation — zero engagement pre-event is expected, not a signal, and the numbers will just alarm the organizer for no reason. Suggest running this after go-live instead.
- If the event **already ended**, the framing shifts from "nudge them now" to "who to prioritize for post-event follow-up" — say so in the verdict line, but the workflow below still applies.

### Step 3 — Segment attendees

Scope everything to the resolved `app_id` and to **active registrations only**:
```
filters=[{"field": "status", "op": "eq", "value": 1}]
```
(Skip inactive/cancelled registrants — see Scope note above.)

**Total active registered attendees (denominator):**
```
get_attendees(select="COUNT(*) AS total", app_ids=[app_id], filters=[{"field":"status","op":"eq","value":1}])
```

**Never logged in:**
```
get_attendees(
  select="id, first_name, last_name, username, registered_at",
  app_ids=[app_id],
  filters=[{"field":"status","op":"eq","value":1}],
  activity_filter={"activity_type":"web_login","visited":false},
  limit=500
)
```

**Logged in (needed to compute the second segment):**
```
get_attendees(
  select="id, first_name, last_name, username",
  app_ids=[app_id],
  filters=[{"field":"status","op":"eq","value":1}],
  activity_filter={"activity_type":"web_login","visited":true},
  limit=500
)
```

**Anyone with real engagement (booth visit or session attendance, union of both):**
```
get_attendee_activity_data(
  select="access_log_id, COUNT(*) AS events",
  activity_type=["booth_visit", "session_attendance"],
  app_ids=[app_id],
  group_by="access_log_id",
  limit=500
)
```
`access_log_id` is the same identity as an attendee's `id` from `get_attendees` — use it to match rows across the two calls. **Important:** `get_attendee_activity_data` is not scoped to attendees only — `visitor_log` also carries rows for exhibitors, speakers, and staff. In live testing against a real event, ~3% of the returned `access_log_id`s belonged to non-attendee identities not present in either attendee list at all. Don't trust a raw `COUNT(DISTINCT access_log_id)` from this call as "attendees engaged" — always compute the segments as set differences against the known attendee `id` universe (below), which discards those foreign rows automatically.

**Logged in, no engagement** = attendees in the "Logged in" set whose `id` does **not** appear in the engagement set above. Compute this as a set difference in your response — there's no single tool call for it, since `activity_filter` only expresses one activity condition at a time.

**Engaged** count = Total active − Never logged in − Logged in-no-engagement. Derive it this way (don't use the raw engagement-query row count) — the subtraction is naturally immune to the foreign-identity rows described above, since it only ever removes IDs that are actually members of the "Logged in" set.

If any of the `get_attendees`/`get_attendee_activity_data` calls returns exactly `limit` rows, raise the limit (max 500) or paginate with `offset` before computing the segments — otherwise the "never logged in" or "logged in, no engagement" counts will be silently undercounted for a large event. This is not a rare edge case: in testing, both the "logged in" list and the engagement-activity query needed a second page to get accurate totals.

### Step 4 — Render the segmentation

Headline verdict (one or two sentences) above the table, e.g. *"148 of 947 active registrants (16%) haven't engaged at all — 92 never logged in, 56 logged in but never visited a booth or session."*

**Engagement Segments** table:

| Segment | Count | % of Active Registrants | Fix |
|---|---:|---:|---|
| **Engaged** | 799 | 84% | — |
| **Logged in, no engagement** | 56 | 6% | Point them at something specific |
| **Never logged in** | 92 | 10% | Login reminder / access issue |
| **Total active registrants** | **947** | **100%** | |

If the user asked for specifics, or a segment is small (under ~30), add a named list table for that segment:

| Name | Registered | Status |
|---|---|---|
| Jane Doe | May 3 | **Never logged in** |
| John Smith | May 4 | **Never logged in** |

Never display raw `id`/`app_id` values — names only. For a large "never logged in" segment (30+), report the count and offer to export the full list rather than dumping it inline.

Skip segmentation nuance if the active registrant base is very small (under ~20) — report the raw counts plainly instead of a chart.

### Step 5 — Close with recommendations

After the tables, add a short **What to do next** list — 2 to 4 concrete actions tied to what the data actually showed. This skill exists to trigger action, so don't skip this. Examples:
- Large "never logged in" segment → check that confirmation/reminder emails are landing (not spam), and resend a direct login link
- "Logged in, no engagement" segment → don't send a generic "come back" email; point them at a specific booth or session based on their registration-form interests (pair with `exhibitor-engagement-report` or `session-attendance-analysis` to pick what to recommend)
- One segment dominates the other → tailor messaging accordingly rather than sending one blanket re-engagement email to everyone at risk
- Event is close to ending → prioritize outreach to the "logged in, no engagement" segment first — they've already cleared the login barrier, so they're the cheaper win with less time left

Only recommend things the data supports. Don't pad the list to reach four items.

## Notes

- Never display raw `id` / `app_id` values to the user — refer to attendees by name.
- "Active registration" here means `status = 1` (not `user_reg_status`) — see the same distinction called out in `registration-health-check`. This skill deliberately excludes inactive/cancelled/rejected registrants; if the user wants those numbers, point them at `registration-health-check` instead.
- `get_attendees` and `get_attendee_activity_data` default to modest row limits (50 and 20 respectively) — always raise them (up to 500) for a full-event segmentation, and check for truncation on a large event before trusting the counts.
- "Logged in, no engagement" is computed as a set difference in-response (logged-in IDs minus engaged IDs) — there's no single MCP call that expresses "did A but not B and not C." Say so if asked why two calls were needed instead of one.
- If `get_attendees` returns a `warnings` field, surface it rather than silently dropping it.
- This is a mid-event (or post-event follow-up) skill — it isn't meaningful before an event has started; check dates before running it (Step 2).
