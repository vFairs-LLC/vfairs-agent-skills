---
name: speaker-performance-recap
description: Per-speaker performance report for a vFairs event — sessions presented, attendance drawn, and ratings/sentiment where survey data supports it — for speaker relations and future invite decisions. Use this skill whenever the user asks how a speaker/presenter performed, wants a speaker recap or leaderboard, asks which speakers drew the most attendance, or is deciding who to re-invite. Requires the vFairs MCP connector (get_events, get_speakers, get_attendee_activity_data, get_survey_result).
---

# Speaker Performance Recap

Builds a per-speaker performance report for a vFairs event: which sessions they presented, how much attendance those sessions drew, and — only where the survey data actually supports it — a satisfaction signal. Built for speaker relations and future-invite decisions, not for public leaderboards.

Scope: **attendance is the primary signal.** Rating data is best-effort and frequently unavailable — see Step 4. Never fabricate a rating for a speaker when none exists in the data.

## When to use this skill

Trigger on requests like:
- "How did our speakers perform at [event]?"
- "Speaker recap / report for [event]"
- "Which speakers drew the biggest audiences?"
- "Should we re-invite [speaker] next year?"
- "Give me a leaderboard of speaker attendance"

## Output format — read this before doing anything else

**This skill is table and charts first. Every response must contain at least two markdown tables and a chart if possible.** Organizers scan a ranked list; they don't read paragraphs describing how each speaker did.

Rules:
- Never write a statistic in prose that could be a table row.
- Find opportunities to create charts when possible — a bar chart of speakers by total attendance works well here.
- Prose is for the verdict and the recommendation only — the "so what," not the "what."
- Minimum tables per response: the **Speaker Performance** table (Step 5). Add the **Session Detail** table (Step 5) whenever a speaker presented more than one session, and a **No Session Assigned** table (Step 2) whenever the roster includes speakers with no session.
- Put the headline verdict in one or two sentences *above* the first table, then the tables, then a short "What to do next" list.
- Use bold status markers in table cells: **Strong draw** / **On par** / **Underattended** / **No data**.

## Workflow

### Step 1 — Resolve the event

Always ask which event this is for if it isn't already named in the request — never default to "most recent" or "all events." Use `get_events` (filter by `name`) to resolve to an `app_id`. If more than one event matches, list them as a table (Event Name | Start Date | Type) and ask the user to pick one.

### Step 2 — Pull speakers and their sessions

```
get_speakers(select="id, first_name, last_name, user_details", app_ids=[app_id], limit=500)
```
Check the `total` field and paginate with `offset` if the event has more speakers than one page returned.

Each speaker row includes a `sessions` list (`session_id`, `session_name`) — this is the speaker-to-session map; no separate `get_sessions` call is needed unless the user wants session descriptions/dates too.

If the user named one specific speaker, filter to them (by name match) after pulling — `get_speakers` filters don't include a name-substring match beyond exact `first_name`/`last_name`, so pull the relevant slice and match client-side if needed, or use `webinar_title` if the user instead named a session.

**Speakers with an empty `sessions` list** never presented — pull these into a small **No Session Assigned** table (Speaker | Registered) rather than silently dropping them; they have no performance data to report, which is itself useful for invite-decision context (a speaker who registered but never got scheduled).

**Test/duplicate accounts caveat:** speaker rosters can include leftover test or duplicate entries (names literally containing "Test," or the same person registered twice). Skim names before presenting the final report and flag anything that looks like test data rather than silently including it in the ranking.

### Step 3 — Pull session attendance

Collect the full set of unique `session_id`s across all speakers' `sessions` lists, then:
```
get_attendee_activity_data(
  select="content_id, COUNT(DISTINCT access_log_id) AS unique_attendees, COUNT(*) AS total_visits",
  activity_type="session_attendance",
  app_ids=[app_id],
  content_ids=[<the collected session_ids>],
  group_by="content_id",
  limit=500
)
```
If the row count returned equals the `limit`, raise it or paginate — don't silently truncate.

**Sessions with literally zero attendance won't appear in this result at all** (no row, not a zero row) — treat any session_id from Step 2 missing from this result as 0 unique attendees, same as `session-attendance-analysis` handles it.

**Total registered attendees** (for a "% of registered attendees" reach column):
```
get_attendees(select="COUNT(*) AS total", app_ids=[app_id])
```

### Step 4 — Ratings (best-effort, likely unavailable)

The vFairs survey tool (`get_survey_result`) has no `session_id` or `speaker_id` field, so a rating can only be linked to a specific speaker if the survey/question naming makes that link explicit — e.g. a survey titled "Feedback: [Speaker Name]" or "Rate [Session Title]," or a question whose text names the session/speaker directly.

1. Pull a light survey overview scoped to the event: `get_survey_result(select="survey_id, survey_title, question_id, question_text, question_type", group_by="survey_id, survey_title, question_id, question_text, question_type", app_ids=[app_id])`.
2. Scan `survey_title`/`question_text` for a match against each speaker's name or their session name(s) from Step 2.
3. Where a match exists and the question is `StarRating` (or a clear Radio rating scale), pull the average: `get_survey_result(select="AVG(answer_text) AS avg_rating, COUNT(*) AS responses", filters=[{"field":"question_id","op":"eq","value":matched_question_id}], app_ids=[app_id])`.
4. If nothing in the survey data names sessions or speakers individually, **say so plainly once** and skip the rating column entirely rather than guessing or reusing an unrelated "overall event" rating as a stand-in for a specific speaker. Don't repeat this caveat per speaker — state it once, up front.

### Step 5 — Render the recap

Headline verdict (one or two sentences) above the table, e.g. *"Jane Kim drew the largest audience at 438 unique attendees across 2 sessions; 3 of 18 speakers presented to fewer than 20 people."*

**Speaker Performance** table, sorted by total unique attendance descending:

| Speaker | Sessions | Total Unique Attendance | Avg per Session | % of Registered | Rating | Status |
|---|---:|---:|---:|---:|---:|---|
| Jane Kim | 2 | 438 | 219 | 46% | 4.7/5 | **Strong draw** |
| Marcus Lee | 1 | 210 | 210 | 22% | — | **On par** |
| Priya Shah | 1 | 18 | 18 | 2% | — | **Underattended** |

`Total Unique Attendance` sums unique attendees across a speaker's sessions — note this is **not** deduplicated across sessions (the same attendee attending two of a speaker's talks counts twice), which is a reasonable proxy for total reach but not unique-person reach; say so if a speaker has multiple sessions.

Leave `Rating` as `—` (not `0` or blank guesswork) for any speaker with no linked survey data, per Step 4.

Status heuristic (state this inline as an assumption, not a fixed target):
- **Strong draw** — total attendance at or above 1.5× the event's median per-speaker attendance
- **On par** — within 0.5×–1.5× of the median
- **Underattended** — below 0.5× the median
- **No data** — speaker had no session assigned (from Step 2's separate table), not scored here

If any speaker presented more than one session, add a **Session Detail** table so the per-session split isn't hidden inside the sum:

| Speaker | Session | Unique Attendees |
|---|---|---:|
| Jane Kim | Opening Keynote | 320 |
| Jane Kim | AI in Practice Workshop | 118 |

Skip ranking nuance if the event has very few speakers (under ~5) — report the raw numbers plainly.

### Step 6 — Close with recommendations

After the tables, add a short **What to do next** list — 2 to 4 concrete actions tied to what the data actually showed. This skill exists to inform speaker relations and invite decisions, so keep it concrete. Examples:
- Top-drawing speakers → strong candidates for re-invitation or a bigger/keynote slot next time
- A speaker with high rating but modest attendance (if rating data exists) → may indicate a promotion/scheduling gap, not a content problem — worth a better slot rather than dropping them
- Underattended speakers clustered in the same time slot → may be a scheduling conflict, not a draw problem (cross-check with `session-attendance-analysis` for parallel-slot conflicts)
- Speakers with no session assigned → follow up before next event on whether they should be scheduled or removed from the roster

Only recommend things the data supports. Don't pad the list to reach four items.

## Notes

- Never display raw `id` / `app_id` / `session_id` / `question_id` values to the user — refer to speakers and sessions by name/title.
- `get_speakers` defaults to a 50-row page and caps at 500 — check `total` and paginate with `offset` for a full-roster recap.
- Sessions with zero attendance are absent from the `get_attendee_activity_data` aggregate entirely, not returned as a zero-count row — treat any session missing from the join as 0.
- Ratings are only available when the survey/question naming explicitly names a session or speaker — there is no structural link in the data. State this limitation once if no such data exists rather than presenting a "Rating" column full of `—` without explanation.
- Test/duplicate speaker accounts are a known source of noise in the roster — call this out if any names look like test data rather than including them silently in the ranking.
- This is a **post-event** (or late-event) skill — attendance data won't be meaningful before a speaker's session has actually happened; if the event hasn't started, say so and suggest `pre-event-readiness-audit` instead.
