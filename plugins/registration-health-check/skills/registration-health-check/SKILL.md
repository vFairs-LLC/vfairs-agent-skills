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

## Workflow

### Step 1 — Resolve the event

Always ask which event this is for if it isn't already named in the request — do not default to "most recent" or "all events." Use `get_events` (filter by `name`) to resolve the name to an `app_id`. If more than one event matches, list the matches and ask the user to pick one.

### Step 2 — Pull the core numbers

Use `get_attendees` scoped to the resolved `app_id`.

**Total registered + status breakdown:**
```
select="status, COUNT(*) AS count"
group_by="status"
```
This returns active/inactive/waiting counts (via `status_label`). Sum for the total.

**Registration pace (daily or weekly trend):**
```
select="toDate(registered_at) AS day, COUNT(*) AS count"
group_by="toDate(registered_at)"
order_by="day"
```
Use this to describe the trend (accelerating, flat, stalling) and, if the user gave a target, to project whether current pace will reach it by the target date.

### Step 3 — Check profile completeness

Per this skill's definition, an attendee counts as an **incomplete registration** if either:
- `status_label` is `"inactive"` (the attendee `status` field, not `user_reg_status`), **or**
- Key registration-form fields are missing.

For the missing-fields check:
```
select="id, first_name, last_name, username, status, user_details"
filters=[{"field": "status", "op": "eq", "value": 0}]
```
Also pull a small sample of `user_details` from active registrants to see which form fields the event actually uses. If the user hasn't told you which fields are "key" (e.g. company, job title, phone), ask once, or default to flagging records where more than half of the event's registration-form fields are blank. Don't guess silently on a large customer-facing number — state the assumption you used.

Report: count of inactive-status attendees, count of active attendees with missing key fields, and a couple of concrete examples if helpful (name + which fields are missing) — but never expose raw `id` values to the user.

### Step 4 — Compare against target (if given)

Only flag "on track" / "at risk" if the user supplied a target in their prompt (a number, and ideally a date). If no target was given, present the numbers and trend without a health verdict — do not invent a target or assume one from a prior event.

If a target + date is given:
- Compute days remaining until the target date.
- Compute the average daily registration pace from Step 2.
- Project: `current_count + (avg_daily_pace * days_remaining)` vs. target.
- Flag as **On track**, **At risk**, or **Behind** based on whether the projection meets the target, is within ~10% of it, or falls meaningfully short.

State the projection as an estimate based on recent pace, not a guarantee.

### Step 5 — Output

**Always give a chat summary first:**
- Total registered, status breakdown
- Pace trend in plain language (e.g. "averaging ~40 new registrations/day over the last week, up from ~25/day the week before")
- Target comparison verdict, if applicable
- Headline completeness issue (e.g. "38 attendees have inactive status; 12 active attendees are missing company name or job title")

**Then offer a detailed report** (only build it if the user asks for it, or if the summary surfaced enough detail that a table is clearly useful): a markdown table breaking down registrations by day/week, the full list of incomplete profiles (names, not ids), and the target-projection math.

## Notes

- Never display raw `id`/`app_id`/`booth_id` values to the user — refer to attendees and events by name.
- `status` (0/1/2 → inactive/active/waiting) and `user_reg_status` (registered/cancelled/pending/etc.) are different fields — this skill's "inactive" flag uses `status`, not `user_reg_status`. If the user asks about cancelled/pending/rejected registrations specifically, that's `user_reg_status`, not this skill's default completeness check.
- If `get_attendees` returns a `warnings` field (e.g. a requested form-field label didn't match anything for this event), surface that to the user rather than silently dropping it.