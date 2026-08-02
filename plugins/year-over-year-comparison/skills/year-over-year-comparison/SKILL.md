---
name: year-over-year-comparison
description: Compare KPIs (turnout, engagement, session attendance, exhibitor traffic, survey sentiment) across two or more instances of a vFairs event — e.g. this year vs. last year — with explicit handling for events of different sizes or types. Use this skill whenever the user asks how an event compares to a prior year, wants year-over-year or historical trend numbers, or asks whether an event grew or shrank vs. a past instance. Requires the vFairs MCP connector (get_events, get_attendees, get_sessions, get_booths, get_attendee_activity_data, get_survey_result).
---

# Year-over-Year Event Comparison

Compares KPIs across two or more instances of a vFairs event (e.g. "Annual Conference 2025" vs. "Annual Conference 2026"). Nothing in the MCP tools aggregates historical KPIs automatically — every metric here is pulled fresh per event and assembled into a side-by-side comparison.

Scope: **comparison, not a full recap of any single event.** For a deep dive on one instance, point the user at `post-event-executive-summary` or the relevant single-topic skill instead of expanding every table here.

## When to use this skill

Trigger on requests like:
- "How did [event] do this year compared to last year?"
- "Year-over-year comparison for [event]"
- "Did attendance grow vs. our last conference?"
- "Compare [Event 2025] and [Event 2026]"
- "Are we trending up or down across our last 3 events?"

## Output format — read this before doing anything else

**This skill is table and charts first. Every response must contain at least two markdown tables and a chart if possible.** Organizers scan a comparison table; they don't read paragraphs describing each metric in turn.

Rules:
- Never write a statistic in prose that could be a table row.
- Find opportunities to create charts when possible — a grouped bar chart (metric × event) or a simple trend line across 3+ events works well here.
- Prose is for the verdict and the recommendation only — the "so what," not the "what."
- Minimum tables per response: the **Event Overview** table (Step 1) and the **Year-over-Year Comparison** table (Step 4).
- Put the headline verdict in one or two sentences *above* the first table, then the tables, then a short "What to do next" list.
- Use bold status markers in table cells: **Up** / **Flat** / **Down**. Favor these on rate-based rows; use them cautiously on raw-count rows if event sizes/types differ (see Step 3).

## Workflow

### Step 1 — Resolve the events to compare

Always ask which events to compare if fewer than two are clearly named — never assume "this year vs. last year" from a single event name. If the user names one recurring event (e.g., "Annual Conference"), use `get_events(name=["Annual Conference"])` — it OR's partial matches, so this one call surfaces every year's instance — and show the matches as a table so the user picks which specific instances to compare:

**Event Overview** table:

| Event Name | Start Date | Type |
|---|---|---|
| Annual Conference 2025 | Apr 14, 2025 | Virtual Conference |
| Annual Conference 2026 | Apr 20, 2026 | Virtual Conference |

Require at least 2 selected events before proceeding. Order every subsequent table chronologically (oldest first, left to right or top to bottom) so trend direction reads naturally.

### Step 2 — Pull KPIs per event

Loop this block once per resolved `app_id` — every call below is scoped to a single event at a time (`app_ids=[app_id]`), since the comparison needs each event's numbers isolated, not summed together.

**Turnout:**
```
get_attendees(select="COUNT(*) AS total", app_ids=[app_id])
```

**Engagement rate** (reuse the `at-risk-attendee-identifier` logic at summary level):
```
get_attendee_activity_data(
  select="access_log_id, COUNT(*) AS events",
  activity_type=["booth_visit","session_attendance"],
  app_ids=[app_id],
  group_by="access_log_id",
  limit=500
)
```
Compute `engaged / total registered`, checking engaged IDs against the actual attendee ID set (the same non-attendee-identity caution documented in `at-risk-attendee-identifier` applies here).

**Session attendance:**
```
get_sessions(query="", app_ids=[app_id])
get_attendee_activity_data(
  select="content_id, COUNT(DISTINCT access_log_id) AS unique_attendees",
  activity_type="session_attendance", app_ids=[app_id], group_by="content_id", limit=500
)
```
Compute session count and avg unique attendees per session (treat sessions missing from the aggregate as 0, same join caveat as `session-attendance-analysis`).

**Exhibitor traffic:**
```
get_booths(query="", app_ids=[app_id])
get_attendee_activity_data(
  select="content_id, COUNT(DISTINCT access_log_id) AS unique_visitors",
  activity_type="booth_visit", app_ids=[app_id], group_by="content_id", limit=500
)
```
Compute booth count and avg unique visitors per booth (same zero-row join caveat as `exhibitor-engagement-report`).

**Survey sentiment (best-effort):**
```
get_survey_result(
  select="AVG(answer_text) AS avg_rating, COUNT(*) AS responses",
  filters=[{"field":"question_type","op":"eq","value":"StarRating"}],
  app_ids=[app_id]
)
```
This is deliberately ungrouped — most events run per-session feedback surveys (the same rating rubric repeated once per session) rather than one event-wide survey, so grouping by question would only surface one session's number. The ungrouped average is a "typical session rating across the event" metric; label it that way in the table, not "overall satisfaction," unless the event genuinely has one dedicated event-wide rating question. If an event has no StarRating survey data, leave that cell `—` rather than treating it as a 0 or omitting the row for other events.

If any call for any event returns row counts at the `limit`, raise it (max 500) or paginate — a truncated year would silently understate that year's numbers and distort the comparison.

### Step 3 — Check comparability before verdict

Before rendering the comparison, check each event's `app_type` from Step 1 and its raw attendee count from Step 2.

- **If `app_type` differs across events** (e.g., "Virtual Career Fair" vs. "Virtual Conference"), say so explicitly and lead the verdict with rate-based metrics (engagement %, avg attendance per session, avg visitors per booth, satisfaction avg) rather than raw counts — a career fair and a conference draw fundamentally different behavior, and a raw headcount comparison would mislead.
- **If registered attendee counts differ by more than ~2×**, flag the size gap in the verdict too, for the same reason — a smaller event's raw session/booth counts aren't directly comparable to a larger one's, even if the type matches. Rate-based metrics normalize for this; raw counts don't.
- Never silently present a raw-count comparison as if the events were equivalent when either check trips — one sentence stating the mismatch is enough, don't belabor it.

### Step 4 — Render the comparison

Headline verdict (one or two sentences) above the table, e.g. *"Annual Conference 2026 grew turnout 18% over 2025, with engagement rate holding steady at ~76% — growth came from more registrants, not better activation."*

**Year-over-Year Comparison** table — one column per event, ordered chronologically, plus a change column when comparing exactly 2 events:

| Metric | 2025 | 2026 | Change |
|---|---:|---:|---|
| Registered attendees | 1,050 | 1,240 | ▲ +18% |
| Engagement rate | 74% | 76% | ▲ +2pp |
| Sessions held | 40 | 46 | ▲ +6 |
| Avg attendance per session | 142 | 148 | ▲ +4% |
| Booths | 20 | 22 | ▲ +2 |
| Avg unique visitors per booth | 118 | 96 | ▼ −19% |
| Overall satisfaction | 4.0 / 5 | 4.2 / 5 | ▲ +0.2 |

For 3+ events, drop the single `Change` column and instead add one **Trend** column at the far right marking each metric ▲/▼/— based on the most recent step (last event vs. second-to-last), while the full history stays visible across the other columns so the reader isn't losing older data to save space.

Status thresholds for bold markers (state this is a heuristic, not a target the user set):
- **Up** — metric improved by more than ~5%
- **Flat** — within ±5%
- **Down** — declined by more than ~5%

Apply percentage-point (`pp`) change, not percent change, to metrics that are already percentages (e.g., engagement rate) — a move from 74% to 76% is "+2pp," not "+2.7%."

Skip the trend/verdict framing on any single metric where Step 3's comparability check applies — report the numbers plainly for that row instead of a confident ▲/▼.

### Step 5 — Close with recommendations

After the tables, add a short **What to do next** list — 2 to 4 concrete actions tied to what the data actually showed. Examples:
- Turnout up but engagement rate flat/down → growth came from marketing, not experience — worth investigating what changed in the platform/agenda before the next cycle
- A metric down significantly (e.g., booth traffic) → cross-reference with `exhibitor-engagement-report` for this year specifically to find which booths drove the decline
- Satisfaction trending down across multiple years → treat as a priority even if turnout is growing, since it's a leading indicator
- Comparing 3+ years and a metric has been declining every cycle → call this out distinctly from a single-year dip, since a multi-year trend needs a different fix than a one-off

Only recommend things the data supports. Don't pad the list to reach four items.

## Notes

- Never display raw `id` / `app_id` / `booth_id` / `session_id` values — refer to events, sessions, and booths by name.
- Nothing in the MCP tools aggregates historical KPIs automatically — every metric is computed fresh per event with the same per-event query pattern used by the single-event skills (`registration-health-check`, `session-attendance-analysis`, `exhibitor-engagement-report`, `survey-insights-sentiment-digest`). This skill's job is the side-by-side assembly and the size/type comparability check, not new data sources.
- Comparing events of different `app_type` or very different attendee counts is a known distortion risk — always check Step 3 before presenting a confident verdict, and favor rate-based metrics over raw counts when either applies.
- If any per-event metric is missing (no survey data, no booths that year, etc.), show `—` for that event/metric rather than a 0 or blank — a 0 implies the event had it and it was zero, which is a different claim than "not measured."
- `get_events(name=[...])` accepts a list and OR's partial matches — pass every year's naming variant in one call rather than calling it once per year, if the user hasn't already given exact event names.
