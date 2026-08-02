---
name: exhibitor-engagement-report
description: Rank vFairs event booths/exhibitors by visitor traffic and interaction volume, flag zero-engagement and underperforming booths, and add sponsor-tier context so a quiet platinum sponsor stands out from a quiet bronze one. Use this skill whenever the user asks how exhibitors/booths/sponsors are performing, which booths have low or no traffic, wants an exhibitor engagement report mid-event, or asks about sponsor ROI/visibility. Requires the vFairs MCP connector (get_events, get_booths, get_exhibitors, get_attendee_activity_data).
---

# Exhibitor Engagement Report

Ranks a vFairs event's booths by actual visitor traffic (from `visitor_log`), flags booths with little or no engagement, and — when sponsor-tier data is available — calls out underperforming sponsors whose tier makes the shortfall a bigger problem. Built to be run **mid-event**, while there's still time to intervene, not just as a post-event recap.

Scope: **booth traffic only**. The vFairs MCP tools do not expose booth-level lead counts, chat-message volume, or document-download counts — only visit events from `visitor_log` (unique visitors + total visit events). State that limitation rather than inventing engagement dimensions the data doesn't support.

## When to use this skill

Trigger on requests like:
- "How are our exhibitors/booths doing so far?"
- "Exhibitor engagement report for [event]"
- "Which booths have low or zero traffic?"
- "Are our platinum sponsors getting enough visibility?"
- "Rank booths by visitor traffic"

## Output format — read this before doing anything else

**This skill is table and charts first. Every response must contain at least two markdown tables and a chart if possible.** Organizers scan a ranked list; they don't read paragraphs describing which booths did well.

Rules:
- Never write a statistic in prose that could be a table row.
- Find opportunities to create charts when possible — a bar chart of top/bottom booths by unique visitors works well here.
- Prose is for the verdict and the recommendation only — the "so what," not the "what."
- Minimum tables per response: the **Engagement Ranking** table (Step 3). Add the **Zero Engagement** table (Step 3) whenever any booths have no traffic at all, and the **Sponsor Tier Breakdown** table (Step 4) whenever tier data is available or supplied.
- Put the headline verdict in one or two sentences *above* the first table, then the tables, then a short "What to do next" list.
- Use bold status markers in table cells: **Strong** / **On par** / **Underperforming** / **Zero engagement**.

## Workflow

### Step 1 — Resolve the event

Always ask which event this is for if it isn't already named in the request — never default to "most recent" or "all events." Use `get_events` (filter by `name`) to resolve to an `app_id`. If more than one event matches, list them as a table (Event Name | Start Date | Type) and ask the user to pick one.

### Step 2 — Pull booths and traffic data

Scope everything to the resolved `app_id`.

**All booths** (id, name, description, floor):
```
get_booths(query="", app_ids=[app_id])
```
No pagination needed — it returns the full list directly.

**Actual booth traffic:**
```
get_attendee_activity_data(
  select="content_id, COUNT(DISTINCT access_log_id) AS unique_visitors, COUNT(*) AS total_visits",
  activity_type="booth_visit",
  app_ids=[app_id],
  group_by="content_id",
  order_by="unique_visitors DESC",
  limit=500
)
```
If the row count returned equals the `limit`, raise `limit` or paginate with `offset` — don't silently truncate a large event.

**Booths with literally zero traffic won't appear in this result at all** (no row, not a zero row). Join the traffic rows onto the full booth list by matching `content_id` to the booth's `i` field, and treat any booth missing from the join as 0 unique visitors. Cross-check with `get_booths(..., zero_engagement=true)`, which returns the same set directly — use it to confirm your join rather than trusting the join alone.

**The reverse mismatch also happens:** some `content_id`s in the traffic data won't match any booth in the current `get_booths` list — leftover visitor-log rows from a deleted or renamed booth. Drop these from the ranking (there's no name to show), but mention in one line that N traffic rows referenced booths no longer on the current floor, so the organizer knows the totals exclude them rather than assuming the data is complete.

**Total registered attendees** (denominator for a "% of registered attendees" reach column):
```
get_attendees(select="COUNT(*) AS total", app_ids=[app_id])
```

### Step 3 — Engagement ranking

Render the **Engagement Ranking** table, sorted by unique visitors descending:

| Booth | Unique Visitors | % of Registered Attendees | Total Visits | Status |
|---|---:|---:|---:|---|
| Engaging Hearts & Minds | 438 | 63% | 2,784 | **Strong** |
| UnidosUS | 170 | 24% | 351 | **On par** |
| Test Booth | 4 | 1% | 10 | **Underperforming** |
| Booth B | 0 | 0% | 0 | **Zero engagement** |

`Total Visits` counts every visit event (`COUNT(*)`), which can exceed unique visitors if people returned to a booth — a high total-to-unique ratio is itself a signal (people are coming back, e.g. for a giveaway or live chat window), so call it out rather than treating the two columns as redundant.

Status heuristic (state this inline as an assumption, not a fixed target — the user hasn't set one):
- **Strong** — unique visitors at or above 1.5× the event's median booth traffic
- **On par** — within 0.5×–1.5× of the median
- **Underperforming** — below 0.5× the median, but above zero
- **Zero engagement** — no visit rows at all

If the user has given a specific traffic target per booth (e.g. from a sponsorship contract's guaranteed-visibility clause), use that instead of the median heuristic and say so — that's a stronger, contractually-relevant signal than a relative ranking.

If any booths have zero engagement, pull them into a separate **Zero Engagement** table (Booth | Floor) so they're not buried in a long ranking. This is the most actionable finding mid-event — flag it prominently in the headline verdict, not just in the table.

Skip trending/ranking nuance if the event has very few booths (under ~5) — just report the raw numbers plainly.

### Step 4 — Sponsor-tier context (best-effort)

vFairs has no native booth-tier/package field in `get_booths`. Tier data, when it exists, lives in the exhibitor registration form (BOOTH_REP), captured per exhibitor rep, not per booth. Try to recover it before giving up on tier context:

1. Pull `get_exhibitors(select="id, booth_id, user_details", app_ids=[app_id])`.
2. Scan the `user_details` maps for a label that looks like tier/package (e.g. contains "package", "tier", "sponsor", "level"). Field names vary by event — don't hardcode one label.
3. If found, map each booth to the tier value from its exhibitor rep(s) (if reps on the same booth disagree, note the conflict rather than silently picking one).
4. If nothing resembling a tier field exists in this event's form, **say so plainly** and offer once: "If you tell me which booths are Platinum/Gold/Silver/Bronze (or your sponsorship tiers), I'll flag underperforming ones by tier — that matters more than a raw ranking since a quiet platinum sponsor is a bigger problem than a quiet bronze one." Do not invent tiers or guess from booth name/description.

If tier data is available (recovered or user-supplied), render the **Sponsor Tier Breakdown** table:

| Tier | Booths | Avg Unique Visitors | Underperforming/Zero |
|---|---:|---:|---:|
| Platinum | 3 | 312 | 0 |
| Gold | 5 | 140 | 1 |
| Silver | 8 | 61 | 2 |
| Bronze | 12 | 24 | 3 |

Then call out by name, in prose, any Platinum or Gold booth that landed in **Underperforming** or **Zero engagement** in Step 3 — that combination (high tier, low traffic) is the single most important finding this skill can surface, since it's the one most likely to need a same-day fix (booth-visit push notification, staff outreach, floor-placement check) rather than a post-event note.

### Step 5 — Close with recommendations

After the tables, add a short **What to do next** list — 2 to 4 concrete actions tied to what the data actually showed. Examples:
- Zero-engagement booths → check whether the booth page is actually live/linked from the lobby before assuming it's a promotion problem
- A high-tier sponsor underperforming → flag for same-day outreach (push notification, email blast, or staff-directed traffic) since there's still time to fix it mid-event
- High total-visits-to-unique-visitor ratio on one booth → worth highlighting as a model for other exhibitors (giveaway, live rep, downloadable content)
- Traffic clustered on 2-3 booths while most sit near zero → may indicate a floor/navigation design issue rather than individual exhibitor content quality

Only recommend things the data supports. Don't pad the list to reach four items.

## Notes

- Never display raw `id` / `app_id` / `booth_id` / `content_id` values to the user — refer to booths by name.
- There is no lead-count, chat-volume, or document-download metric in the current MCP tools — this skill reports visit traffic only. If the user asks about lead counts specifically, say that's not available from this data.
- `get_attendee_activity_data` defaults to a 20-row limit — always raise it (e.g. 500) for a full-event ranking, and check whether the returned row count suggests more pages exist.
- Booths with zero engagement are absent from the aggregate query entirely, not returned as a zero-count row — join against the full booth list (or cross-check with `zero_engagement=true`) to catch them.
- Sponsor tier is not a first-class field anywhere in the MCP tools — it's recovered opportunistically from exhibitor registration-form answers, and many events won't have it captured at all. Never fabricate a tier.
- `get_exhibitors` is scoped to booth reps (`user_type_id=3`), not attendees — use it only to look up tier/package answers, not to measure booth traffic.
