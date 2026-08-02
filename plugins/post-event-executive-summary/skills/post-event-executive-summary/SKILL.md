---
name: post-event-executive-summary
description: Stakeholder-ready, one-page wrap-up of a vFairs event — turnout, top sessions, exhibitor performance, and survey sentiment — in headline numbers only, no data dump. Use this skill whenever the user asks for an executive summary, event recap, wrap-up report, or "how did the event go" for leadership/stakeholders. Requires the vFairs MCP connector (get_events, get_attendees, get_sessions, get_booths, get_attendee_activity_data, get_survey_result).
---

# Post-Event Executive Summary

Composes a one-page, stakeholder-ready wrap-up of a vFairs event: turnout, top sessions, exhibitor highlights, and survey sentiment. This is the condensed, leadership-facing view — for a full breakdown of any one area, point the user at the dedicated skill (`registration-health-check`, `session-attendance-analysis`, `exhibitor-engagement-report`, `at-risk-attendee-identifier`, `survey-insights-sentiment-digest`, `speaker-performance-recap`) instead of dumping that level of detail here.

Scope: **headline numbers only.** This skill deliberately does not reproduce full rankings, per-question survey breakdowns, or long attendee lists — that's what the underlying skills are for. If the user wants that depth after seeing the summary, point them at the relevant skill by name rather than expanding this one.

## When to use this skill

Trigger on requests like:
- "Give me an executive summary for [event]"
- "How did [event] go overall?"
- "Wrap-up report for [event] for leadership"
- "One-pager on [event]'s results"
- "Recap [event] for the stakeholder call"

## Output format — read this before doing anything else

**This is the one skill in this set where "keep it short" overrides "table for every stat."** Executives want headline numbers and a verdict, not a data dump. Rules:
- Cap every table at **5 rows** (top 5, not the full list). If there are more than 5 noteworthy items, say "see `<skill-name>` for the full breakdown" rather than expanding the table.
- Maximum **4 tables total** in the response: Headline KPIs, Top Sessions, Top Exhibitors, and (only if survey data exists) Survey Sentiment.
- Open with a 3–5 sentence **Executive Summary** paragraph — the headline story in plain language — *before* any table. This is the one skill where prose leads, not tables.
- Everything below the opening paragraph should be scannable in under a minute. No sub-bullets explaining what a table already shows.
- Use bold status markers sparingly, only in the Headline KPIs table: **Strong** / **Mixed** / **Needs attention**.

## Workflow

### Step 1 — Resolve the event and check timing

Always ask which event this is for if it isn't already named in the request — never default to "most recent" or "all events." Use `get_events` (filter by `name`) to resolve to an `app_id`.

Check the event's end date against today. If the event **hasn't ended yet**, say so plainly and ask whether they want a mid-event snapshot anyway (framed as preliminary) or want to wait — a headline "final" summary before the event closes will misrepresent turnout and exhibitor numbers that are still climbing. If they want the preliminary version, label every number in the output "as of today" rather than presenting it as final.

### Step 2 — Turnout (condensed registration-health-check + at-risk-attendee-identifier)

```
get_attendees(select="COUNT(*) AS total", app_ids=[app_id])
get_attendees(select="status, COUNT(*) AS count", group_by="status", app_ids=[app_id])
```
Engagement: reuse the never-logged-in / engaged split logic from `at-risk-attendee-identifier` at a summary level only — you need the **engaged count**, not the full segmented list:
```
get_attendee_activity_data(
  select="access_log_id, COUNT(*) AS events",
  activity_type=["booth_visit","session_attendance"],
  app_ids=[app_id],
  group_by="access_log_id",
  limit=500
)
```
Compute `engaged = COUNT(DISTINCT access_log_id) from that result that also appears in the active-attendee ID set` (same set-difference caution as `at-risk-attendee-identifier` — some `access_log_id`s belong to non-attendee identities; don't trust the raw distinct count without checking membership). `Engagement Rate = engaged / total registered`.

### Step 3 — Top sessions (condensed session-attendance-analysis)

```
get_sessions(query="", app_ids=[app_id])
get_attendee_activity_data(
  select="content_id, COUNT(DISTINCT access_log_id) AS unique_attendees",
  activity_type="session_attendance",
  app_ids=[app_id],
  group_by="content_id",
  order_by="unique_attendees DESC",
  limit=500
)
```
Join on `content_id` → session `i` field, take the **top 5** by unique attendees. Also note the total session count and how many had zero attendance (`get_sessions(..., zero_engagement=true)`), as a single count — not a listed table.

### Step 4 — Exhibitor highlights (condensed exhibitor-engagement-report)

```
get_booths(query="", app_ids=[app_id])
get_attendee_activity_data(
  select="content_id, COUNT(DISTINCT access_log_id) AS unique_visitors",
  activity_type="booth_visit",
  app_ids=[app_id],
  group_by="content_id",
  order_by="unique_visitors DESC",
  limit=500
)
```
Join the same way as sessions (zero-traffic booths are absent from the aggregate, not zero rows — cross-check with `get_booths(..., zero_engagement=true)`). Take the **top 5** booths by unique visitors, and report the zero-engagement booth count as a single number, not a listed table.

### Step 5 — Survey sentiment (condensed survey-insights-sentiment-digest, best-effort)

First check for a single, clearly-named "overall event" survey (a `survey_title` like "[Event Name] Survey" distinct from per-session feedback surveys). If one exists and has a StarRating question, use its average directly.

Most events, though, only run **per-session feedback surveys** (the same rubric — e.g. "The session overall was...", "Would you recommend this session?" — repeated once per session, not one event-wide survey). In that common case, there is no single "overall" question to point at — don't group by `question_id`/`question_text` and pick one row, since that only surfaces one session's number. Instead compute one whole-event average across every StarRating response, ungrouped:
```
get_survey_result(
  select="AVG(answer_text) AS avg_rating, COUNT(*) AS responses",
  filters=[{"field":"question_type","op":"eq","value":"StarRating"}],
  app_ids=[app_id]
)
```
Label this plainly as "average session rating across all feedback surveys," not "overall event satisfaction" — it's a real, useful number, but it's an average of session-level ratings, not a response to a dedicated event-wide question, and the exec summary shouldn't blur that distinction.

If there's no StarRating data at all, check for a Radio-style satisfaction question instead; if neither exists, state plainly that no sentiment data is available for this event rather than omitting the section silently — an executive summary should say "we don't have this" rather than just leaving a gap unexplained. Skip open-ended theme extraction entirely here — that's `survey-insights-sentiment-digest`'s job, not this skill's.

### Step 6 — Render the summary

**Executive Summary** (prose, 3–5 sentences, above everything else) — the headline story: how big was turnout, how engaged were people, what stood out (best session, best exhibitor, any concern), and the overall verdict in plain language. Example shape: *"[Event] drew 1,240 registered attendees, 78% of whom engaged with at least one session or booth — a solid rate for a hybrid event. The Opening Keynote was the clear draw at 620 unique attendees. Exhibitor traffic was more uneven: 4 of 22 booths saw zero visitors. Overall satisfaction (survey avg 4.2/5) was strong, though open-ended themes point to platform navigation as a recurring complaint — see the survey digest for detail."*

**Headline KPIs** table:

| Metric | Value | Status |
|---|---:|---|
| Registered attendees | 1,240 | — |
| Engagement rate | 78% | **Strong** |
| Sessions held | 46 (2 zero-attendance) | — |
| Top session attendance | 620 | — |
| Booths | 22 (4 zero-engagement) | **Needs attention** |
| Overall satisfaction | 4.2 / 5 (312 responses) | **Strong** |

**Top Sessions** table (max 5 rows):

| Session | Unique Attendees |
|---|---:|
| Opening Keynote | 620 |
| AI in Practice Workshop | 410 |

**Top Exhibitors** table (max 5 rows):

| Booth | Unique Visitors |
|---|---:|
| Acme Corp | 380 |
| Globex | 290 |

**Survey Sentiment** — one line or a 1–2 row table if there's exactly an overall rating plus one notable sub-rating; skip entirely (with the one-line explanation from Step 5) if no data exists.

### Step 7 — Close with recommendations

After the tables, add a short **What to do next** list — 2 to 4 concrete actions, pitched at a stakeholder-decision level (renew/expand/fix), not operational minutiae. Examples:
- Strong overall turnout and satisfaction → supports a renewal/expansion case for next year
- Exhibitor traffic concentrated in a few booths with several at zero → floor design or promotion may need rework before selling next year's packages
- Survey theme pointing at a specific pain point → a concrete, low-cost fix to commit to publicly
- If any headline number required a caveat (mid-event snapshot, no survey data, etc.) → repeat that caveat here so it isn't lost if this summary is forwarded on its own

Only recommend things the data supports. Don't pad the list to reach four items.

## Notes

- Never display raw `id` / `app_id` / `booth_id` / `session_id` / `question_id` values — refer to everything by name.
- This skill composes queries from several other skills at a summary level — it does not invoke them. If the user wants the full ranking, full survey theme digest, or full at-risk segmentation, tell them which skill to ask for by name rather than reproducing that depth here.
- If turnout, session, exhibitor, or survey data is missing or unavailable for this event, say so plainly in the relevant section rather than omitting it without explanation — a stakeholder reading this later needs to know what wasn't measurable, not just see a shorter report.
- If the event hasn't ended yet, every number in the response must be labeled as a snapshot, not a final result.
