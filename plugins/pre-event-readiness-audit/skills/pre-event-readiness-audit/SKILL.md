---
name: pre-event-readiness-audit
description: Cross-check sessions, speakers, and booths for missing info before a vFairs event goes live — sessions with no description, speakers with no bio or photo, booths with no rep assigned. Use this skill whenever the user asks if an event is "ready to go live," wants a pre-event checklist, asks what content is missing, or asks for a readiness score before launch. Requires the vFairs MCP connector (get_events, get_sessions, get_speakers, get_booths, get_exhibitors).
---

# Pre-Event Readiness Audit

Checks whether a vFairs event's content is ready to go live: session details, speaker profiles, and booth/exhibitor setup. Produces a pass/fail checklist per item and an overall readiness score.

Scope: **content completeness**, not engagement or attendance — this skill never looks at visitor/traffic data. For engagement questions, that's a different skill.

## When to use this skill

Trigger on requests like:
- "Is [event] ready to go live?"
- "Pre-event readiness check for [event]"
- "What's missing before launch?"
- "Which sessions/speakers/booths need attention before the event?"
- "Give me a readiness score for [event]"

## Output format — read this before doing anything else

**This skill is table and charts first. Every response must contain at least two markdown tables and a chart if possible** Organizers scan a ranked list; they don't read paragraphs describing which sessions did well.

Rules:
- Never write a statistic in prose that could be a table row.
- Find opportunities to create charts when possible. This could be bar charts, pie chart or something else.
- Prose is for the verdict and the recommendation only — the "so what," not the "what."
- Minimum tables per response: the **Readiness Scorecard** (Step 6) and at least one category **Detail** table (Steps 3–5) for whichever category has issues. If a category has zero issues, say so in one line instead of an empty table — don't render an empty table.
- Put the headline verdict in one or two sentences *above* the first table, then the tables, then a short "What to do next" list.
- Use bold status markers in table cells: **Pass** / **Needs attention** / **Missing**.

## Workflow

### Step 1 — Resolve the event

Always ask which event this is for if it isn't already named in the request — never default to "most recent" or "all events." Use `get_events` (filter by `name`) to resolve to an `app_id`. If more than one event matches, list them as a table (Event Name | Start Date | Type) and ask the user to pick one.

Capture the event's start date — a readiness gap matters more with 3 days to go than with 6 weeks to go, so mention days-until-launch in the verdict.

### Step 2 — Pull the raw data

Scope everything to the resolved `app_id`.

- `get_sessions(query="", app_ids=[app_id])` — returns all sessions directly; no pagination.
- `get_speakers(select="id, first_name, last_name, user_details", app_ids=[app_id], limit=500)` — paginate with `offset` if `total` exceeds the returned count.
- `get_booths(query="", app_ids=[app_id])` — returns all booths directly; no pagination.
- `get_exhibitors(select="id, booth_id", app_ids=[app_id], limit=500)` — used only to check which booths have at least one rep assigned; paginate the same way as speakers.

Before scoring, sample 5–10 `user_details` entries from the speaker pull to learn the event's actual form field labels — the bio/photo field is a registration-form field and its exact label (`"Bio"`, `"Speaker Bio"`, `"About"`, etc.) varies per event. Match on the closest label rather than assuming an exact name, and state which label you matched in the notes if it isn't an obvious match.

**Placeholder-text check (applies to session and booth descriptions alike):** a length check alone isn't enough — real events have shipped descriptions that clear 40 characters but are still unusable filler: "Lorem Ipsum" boilerplate, or gibberish/keyboard-mash strings (low character variety, no real words, e.g. "jfdrgjfg ditrifitiitiiiiiiii"). Treat a description as placeholder text, not a pass, if it contains "lorem ipsum" (case-insensitive) or reads as random characters rather than real words. This is a judgment call on the text itself, not a regex — apply it the way a person skimming the field would.

### Step 3 — Session readiness

For each session, check:

| Check | Pass condition |
|---|---|
| Description present | `d` (description) is non-blank, more than ~40 characters, and not obvious placeholder text (see below) |
| Schedule set | Both start (`s`) and end (`e`) times are present |
| Track/day assigned | `tr` (track) is non-empty |
| Speaker assigned | At least one speaker's session list (from Step 2's speaker pull) includes this session |

A session is **ready** only if it passes all four. Render the **Session Detail** table listing only sessions with at least one issue (never the full session list if most are fine):

| Session | Missing |
|---|---|
| Partner Onboarding Overview | No description, no speaker assigned |
| Closing Remarks | No track/day assigned |

If every session passes, say so in one line and skip the table.

### Step 4 — Speaker readiness

For each speaker, check:

| Check | Pass condition |
|---|---|
| Bio present | The matched bio field is non-blank and more than a few words |
| Photo present | The matched profile-picture field is non-blank |
| Session assigned | The speaker's `sessions` list (from `get_speakers`) is non-empty |

A speaker is **ready** only if it passes all three. Render the **Speaker Detail** table listing only speakers with issues:

| Speaker | Missing |
|---|---|
| Greg Contreras | No bio, no photo |
| Sarah Johnson | No session assigned |

If every speaker passes, say so in one line and skip the table.

**Caveat to check for and mention if found:** event rosters can include leftover test or duplicate accounts (e.g. names literally containing "Test," or the same person registered twice under different emails). These will show up as "no session assigned" and inflate the issue count without being a real gap. Skim names before presenting the final list and flag anything that looks like test data rather than silently including it — don't guess which ones to drop, just call it out so the organizer can confirm.

### Step 5 — Booth readiness

For each booth, check:

| Check | Pass condition |
|---|---|
| Description present | `d` (description) is non-blank, more than ~40 characters, and not obvious placeholder text (see below) |
| Rep assigned | At least one exhibitor row from Step 2's `get_exhibitors` pull has a matching `booth_id` |

A booth is **ready** only if it passes both. Render the **Booth Detail** table listing only booths with issues:

| Booth | Missing |
|---|---|
| Booth B | No description, no rep assigned |
| UnidosUS | No rep assigned |

If every booth passes, say so in one line and skip the table.

**Known gap — state this once, don't re-derive it each time:** the vFairs tools don't expose a booth "materials/collateral uploaded" field. This skill checks description and rep assignment only; it cannot verify whether a booth has brochures, videos, or documents attached. If the user specifically asks about materials, say this isn't currently checkable via the connector rather than guessing.

### Step 6 — Overall readiness score

Render the **Readiness Scorecard** — one row per category plus an overall row. Category score = (items passing all checks) ÷ (total items in that category). Overall score = the simple average of the three category scores (not weighted by item count) — this keeps a small booth count from being drowned out by a large session count.

| Category | Items Checked | Passing | Score | Status |
|---|---:|---:|---:|---|
| Sessions | 56 | 51 | 91% | **Pass** |
| Speakers | 84 | 68 | 81% | **Needs attention** |
| Booths | 6 | 4 | 67% | **Needs attention** |
| **Overall** | **146** | **123** | **80%** | **Needs attention** |

Status thresholds (state this is a heuristic, not a target the user set):
- **Pass** — 90%+ of items in the category are fully ready
- **Needs attention** — 75–89%
- **Missing** — under 75%

Never invent a launch/go-live deadline if the user didn't give one — just report against days-until-start captured in Step 1.

### Step 7 — Close with recommendations

After the tables, add a short **What to do next** list — 2 to 4 concrete actions tied to what the data actually showed. Examples:
- Low speaker score driven mostly by missing photos → suggest a one-line reminder email to affected speakers with a photo-upload link
- Booths with no rep assigned → these won't have anyone to answer booth-chat questions live; flag for outreach first, before description gaps
- Sessions missing a track → scheduling/agenda display may break or mis-group; fix before the agenda page is finalized
- Tight timeline (event starts in under a week) and score below "Pass" → prioritize speaker/session gaps over booth cosmetic gaps, since those affect the printed/published agenda

Only recommend things the data supports. Don't pad the list to reach four items.

## Notes

- Never display raw `id` / `app_id` / `booth_id` / `session_id` values to the user — refer to sessions, speakers, and booths by name/title.
- Speaker bio/photo field labels are event-specific registration-form fields, not fixed columns — always sample before scoring, and don't assume a label that doesn't exist in this event's data.
- `get_speakers` and `get_exhibitors` default to a 50-row page and cap at 500 — check the `total` field and paginate with `offset` if the event has more records than one page returned.
- Test/duplicate accounts are a known source of false positives in speaker and booth data — call this out when flagging "no session assigned" or "no rep assigned" issues rather than presenting the count as if every row is a real gap.
- If the event has very few sessions, speakers, or booths in a category (under ~5), still show the detail table if there are issues, but don't dress up a 2-item category with a false sense of precision in the score.
