---
name: session-attendance-analysis
description: Rank vFairs event sessions by actual attendance, flag zero-attendance and underattended sessions, and surface scheduling conflicts where popular sessions ran in the same time slot. Use this skill whenever the user asks which sessions were most/least popular, wants a session attendance breakdown, asks which sessions nobody attended, or asks whether sessions overlapped/competed for attendees. Requires the vFairs MCP connector (get_events, get_sessions, get_attendee_activity_data, get_attendees).
---

# Session Attendance & Popularity Analysis

Ranks a vFairs event's sessions by actual attendance (from `visitor_log`), flags sessions with little or no attendance, and surfaces time slots where multiple sessions competed for the same audience.

Scope: **actual attendance only**. The vFairs MCP tools do not expose a per-session RSVP/registration count or a room-capacity field, so this skill cannot report "registered vs. attended" or true venue overcrowding unless the user supplies a capacity/target themselves. State that limitation rather than inventing numbers — see Step 3.

## When to use this skill

Trigger on requests like:
- "Which sessions were most popular at [event]?"
- "Session attendance breakdown for [event]"
- "Which sessions did nobody show up to?"
- "Were any sessions overcrowded / underattended?"
- "Did any sessions overlap and compete for attendees?"

## Output format — read this before doing anything else

**This skill is table and charts first. Every response must contain at least two markdown tables and a chart if possible** Organizers scan a ranked list; they don't read paragraphs describing which sessions did well.

Rules:
- Never write a statistic in prose that could be a table row.
- Find opportunities to create charts when possible. This could be bar charts, pie chart or something else.
- Prose is for the verdict and the recommendation only — the "so what," not the "what."
- Minimum tables per response: the **Popularity Ranking** table (Step 3). Add the **Scheduling Conflicts** table (Step 4) whenever overlapping high-attendance slots are found, and a **Zero Attendance** table (Step 3) whenever any sessions have no attendance at all.
- Put the headline verdict in one or two sentences *above* the first table, then the tables, then a short "What to do next" list.
- Use bold status markers in table cells: **Popular** / **On par** / **Underattended** / **Zero attendance**.

## Workflow

### Step 1 — Resolve the event

Always ask which event this is for if it isn't already named in the request — never default to "most recent" or "all events." Use `get_events` (filter by `name`) to resolve to an `app_id`. If more than one event matches, list them as a table (Event Name | Start Date | Type) and ask the user to pick one.

### Step 2 — Pull sessions and attendance data

Scope everything to the resolved `app_id`.

**All sessions** (id, title, start, end, track/day label):
```
get_sessions(query="", app_ids=[app_id])
```
No pagination needed — it returns the full list directly.

**Actual attendance per session:**
```
get_attendee_activity_data(
  select="content_id, COUNT(DISTINCT access_log_id) AS unique_attendees, COUNT(*) AS total_visits",
  activity_type="session_attendance",
  app_ids=[app_id],
  group_by="content_id",
  order_by="unique_attendees DESC",
  limit=500
)
```
If the row count returned equals the `limit`, raise `limit` or paginate with `offset` — don't silently truncate a large event.

**Sessions with literally zero attendance won't appear in this result at all** (no row, not a zero row). Join the attendance rows onto the full session list from `get_sessions` by matching `content_id` to the session's `i` field, and treat any session missing from the join as 0 unique attendees. Cross-check with `get_sessions(..., zero_engagement=true)`, which returns the same set directly — use it to confirm your join rather than trusting the join alone.

**The reverse mismatch also happens:** some `content_id`s in the attendance data won't match any session in the current `get_sessions` list at all — leftover visitor-log rows from a deleted, renamed, or superseded session. Drop these from the ranking (there's no title to show), but don't just silently discard them — mention in one line that N attendance rows referenced sessions no longer in the current agenda, so the organizer knows the totals exclude them rather than assuming the data is complete.

**Total registered attendees** (for a "% of registered attendees" popularity column):
```
get_attendees(select="COUNT(*) AS total", app_ids=[app_id])
```

### Step 3 — Popularity ranking

Render the **Popularity Ranking** table, sorted by unique attendees descending:

| Session | Unique Attendees | % of Registered | Total Visits | Status |
|---|---:|---:|---:|---|
| Opening General Session | 521 | 65% | 1,027 | **Popular** |
| DC Update General Session | 441 | 55% | 589 | **Popular** |
| MSIX – New and Updated Resources | 168 | 21% | 220 | **On par** |
| Regional Breakout: Northwest | 25 | 3% | 61 | **Underattended** |
| Closing Remarks | 0 | 0% | 0 | **Zero attendance** |

`Total Visits` counts every check-in row (`COUNT(*)`), which can exceed unique attendees if people re-entered or rewatched — don't conflate the two columns.

Status heuristic (state this inline as an assumption, not a fixed target — the user hasn't set one):
- **Popular** — unique attendees at or above 1.5× the event's median session attendance
- **On par** — within 0.5×–1.5× of the median
- **Underattended** — below 0.5× the median, but above zero
- **Zero attendance** — no attendance rows at all

If the user has given a room capacity or a registration/RSVP target for specific sessions, use that instead of the median heuristic and say so — that's a stronger signal than a relative ranking. Otherwise, do not invent a capacity number; this skill only knows who actually showed up, not how many seats a room had.

If any sessions have zero attendance, pull them into a separate **Zero Attendance** table (Session | Track/Day) so they're not buried in a long ranking.

Skip trending/ranking nuance if the event has very few sessions (under ~5) — just report the raw numbers plainly.

### Step 4 — Scheduling conflicts

Group sessions by their `s` (start) value — sessions sharing an identical start timestamp ran in parallel (the same slot had multiple concurrent tracks). This event's data showed 7–8 sessions per slot on multi-track days; don't assume single-track scheduling.

**Data caveat:** the `e` (end time) field in this data has shown values that appear earlier than `s` on the same calendar-date label (e.g. start `15:30`, end `01:30`) — likely a timezone or day-wrap formatting quirk, not a real 22-hour session. Don't trust `e` for precise overlap-window math; use matching `s` values as the reliable signal for "these ran at the same time," not computed start/end overlap.

For each slot with 2+ sessions, check whether **two or more** of them individually rank in the **top quartile of attendance across all sessions** (not just "Popular," which is too loose a bar here — on a multi-track agenda, most breakout slots will contain at least one "Popular" and one "On par" session simply because there are always several tracks running at once, and flagging every one of those as a "conflict" buries the real signal in noise). Top-quartile-vs-top-quartile is a much stronger indicator that two genuine audience draws actually split attendance.

Render the **Scheduling Conflicts** table only when at least one such slot exists:

| Time Slot | Competing Sessions | Attendance |
|---|---|---:|
| Apr 28, 11:30 AM | MSIX – New and Updated Resources | 168 |
| | Interstate Coordination Roundtable | 143 |

Note in prose which of the two likely pulled the larger draw and that the other's numbers may be depressed by the overlap rather than by low interest — that distinction changes what an organizer should do about it (reschedule vs. drop).

If no slot has two qualifying sessions, skip the table and say so in one line.

### Step 5 — Close with recommendations

After the tables, add a short **What to do next** list — 2 to 4 concrete actions tied to what the data actually showed. Examples:
- Zero-attendance sessions → check whether they were cancelled, mis-promoted, or simply didn't fit attendee interests before scheduling a repeat
- Underattended sessions clustered in one track/day → may indicate a scheduling or promotion gap for that day, not a content problem
- A popular session overlapped with another popular one → consider separating them next time, or recording/replaying the lower-traffic one
- High total-visits-to-unique-attendee ratio on one session → may indicate a recording or resource worth promoting further, since people are returning to it

Only recommend things the data supports. Don't pad the list to reach four items.

## Notes

- Never display raw `id` / `app_id` / `content_id` values to the user — refer to sessions by title.
- There is no per-session registration/RSVP count or room-capacity field in the current MCP tools. This skill reports actual attendance and relative popularity only — it cannot say a session was "overcrowded" against a real limit unless the user supplies one.
- The `tr` (track) field on a session is often just a day label ("Monday, April 27"), not a named parallel track — don't assume it distinguishes concurrent tracks; use matching start times for that instead.
- `get_attendee_activity_data` defaults to a 20-row limit — always raise it (e.g. 500) for a full-event ranking, and check whether the returned row count suggests more pages exist.
- Sessions with zero attendance are absent from the aggregate query entirely, not returned as a zero-count row — you must join against the full session list (or cross-check with `zero_engagement=true`) to catch them.
