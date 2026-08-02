---
name: registration-health-check
description: Check attendee registration pace, status breakdown, and profile completeness for a vFairs event, and flag whether registration is on track against a target. Use this skill whenever the user asks how registration is going, wants a registration pace/health check, asks to compare registrations against a goal or target, or asks about incomplete/inactive attendee registrations for an event. Requires the vFairs MCP connector (get_events, get_attendees).
---

# Registration Health Check

Checks attendee registration health for a vFairs event: current count, pace over time, status breakdown, and profile-completeness issues. Optionally compares against a target the user provides and flags whether the event is on/off track.

Scope: **attendees only** (not exhibitors or speakers — those would need a separate check).

## When to use this skill

Trigger on requests like:
- "How's registration going for [event]?"
- "Registration health check for [event]"
- "Are we on track to hit 1,000 registrations by June 1?"
- "How many incomplete or inactive registrations do we have?"
- "Registration pace for [event] vs last month"

## Output format — read this before doing anything else

**This skill is table and charts first. Every response must contain at least two markdown tables and a chart if possible** Organizers scan a ranked list; they don't read paragraphs describing which sessions did well.

Rules:
- Never write a statistic in prose that could be a table row. If a sentence contains more than two numbers, it should have been a table.
- Find opportunities to create charts when possible. This could be bar charts, pie chart or something else.
- Prose is for the verdict and the recommendation only — the "so what," not the "what."
- Minimum tables per response: **Status Summary** (Step 2) and **Registration Pace** (Step 2). Add **Target Tracking** (Step 4) whenever a target was given, and **Data Quality** (Step 3) whenever issues are found.
- Put the headline verdict in one or two sentences *above* the first table, then the tables, then a short "What to do next" list.
- Use bold status markers in table cells so organizers can scan: **On track** / **At risk** / **Behind**.

The exact table shapes are specified in each step below. Use them.

## Workflow

### Step 1 — Resolve the event

Always ask which event this is for if it isn't already named in the request — do not default to "most recent" or "all events." Use `get_events` (filter by `name`) to resolve the name to an `app_id`. If more than one event matches, list the matches **as a table** (Event Name | Start Date | Type) and ask the user to pick one.

Also capture the event's start date — it's needed for the "days until event" context in Step 4.

### Step 2 — Pull the core numbers

Use `get_attendees` scoped to the resolved `app_id`.

**Total registered + status breakdown:**
```
select="status, COUNT(*) AS count"
group_by="status"
```

Render as the **Status Summary** table:

| Status | Count | % of Total |
|---|---:|---:|
| Active | 842 | 89% |
| Inactive | 74 | 8% |
| Waiting | 31 | 3% |
| **Total** | **947** | **100%** |

**Registration pace:**
```
select="toDate(registered_at) AS day, COUNT(*) AS count"
group_by="toDate(registered_at)"
order_by="day"
```

Render as the **Registration Pace** table. Bucket by week if the event has a long registration window (more than ~3 weeks of data), by day if short — a 90-row daily table is unreadable. Include a running total and a change indicator:

| Week | New Registrations | Running Total | Change |
|---|---:|---:|---|
| Apr 28 – May 4 | 118 | 402 | — |
| May 5 – 11 | 173 | 575 | ▲ +47% |
| May 12 – 18 | 210 | 785 | ▲ +21% |
| May 19 – 25 | 162 | 947 | ▼ −23% |

If the most recent bucket is partial (the week isn't over yet), label it clearly as partial so a drop isn't misread as a stall.

### Step 3 — Check profile completeness

An attendee counts as an **incomplete registration** if either:
- `status_label` is `"inactive"` (the attendee `status` field, not `user_reg_status`), **or**
- Key registration-form fields are missing.

For the missing-fields check:
```
select="id, first_name, last_name, username, status, user_details"
filters=[{"field": "status", "op": "eq", "value": 0}]
```

Also pull a small sample of `user_details` from active registrants to see which form fields the event actually uses. If the user hasn't said which fields are "key" (e.g. company, job title, phone), default to flagging records where more than half of the event's registration-form fields are blank — and state that assumption in one line under the table.

Render as the **Data Quality** table (skip only if there are zero issues — in that case say so in one line):

| Issue | Count | % of Registrants | Why it matters |
|---|---:|---:|---|
| Inactive status | 74 | 8% | Won't receive event comms or be able to log in |
| Missing company name | 38 | 4% | Weakens exhibitor lead quality |
| Missing job title | 22 | 2% | Limits session targeting |

If the user wants specifics, follow with a named list of affected attendees (names only, **never raw `id` values**).

### Step 4 — Compare against target (if given)

Only flag "on track" / "at risk" if the user supplied a target in their prompt (a number, and ideally a date). If no target was given, present the numbers and trend without a health verdict — do not invent a target or assume one from a prior event. Instead, offer once: "If you tell me your registration target and deadline, I can project whether you'll hit it."

If a target + date is given, render the **Target Tracking** table:

| Metric | Value |
|---|---:|
| Target | 1,200 by Jun 15 |
| Registered today | 947 |
| Gap to target | 253 |
| Days remaining | 21 |
| Required pace | 12.0 /day |
| Current pace (last 7 days) | 23.1 /day |
| Projected final | ~1,432 |
| **Status** | **On track** |

Method: days remaining until the target date; average daily pace from the most recent 7 days (not the full window — early-campaign spikes distort the average); projection = `current_count + (recent_daily_pace × days_remaining)`.

Thresholds:
- **On track** — projection meets or exceeds target
- **At risk** — projection lands within 10% below target
- **Behind** — projection falls more than 10% short

Always state the projection is an estimate based on recent pace, not a guarantee — registration typically spikes in the final week and after email sends, so a "Behind" flag early on is a prompt to act, not a forecast of failure.

### Step 5 — Close with recommendations

After the tables, add a short **What to do next** list — 2 to 4 concrete actions tied to what the data actually showed. This is the part organizers act on. Examples:
- Pace dropped this week → suggest a reminder email or social push
- High inactive count → suggest checking whether confirmation emails are landing in spam
- Missing company/job title → suggest making those fields required, or a profile-completion nudge to affected registrants
- Behind on target → state the daily pace now required to close the gap

Only recommend things the data supports. Don't pad the list to reach four items.

## Notes

- Never display raw `id`/`app_id`/`booth_id` values to the user — refer to attendees and events by name.
- `status` (0/1/2 → inactive/active/waiting) and `user_reg_status` (registered/cancelled/pending/etc.) are different fields — this skill's "inactive" flag uses `status`, not `user_reg_status`. If the user asks about cancelled/pending/rejected registrations specifically, that's `user_reg_status`, not this skill's default completeness check.
- If `get_attendees` returns a `warnings` field (e.g. a requested form-field label didn't match anything for this event), surface it to the user rather than silently dropping it.
- If the event has very few registrants (under ~20), skip the pace table and projection — the numbers are too small to trend meaningfully. Say so plainly instead of showing a noisy chart of single-digit rows.