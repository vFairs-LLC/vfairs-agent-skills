---
name: survey-insights-sentiment-digest
description: Aggregate vFairs survey, quiz, and poll responses into structured breakdowns and themed digests — ratings distributions, quiz accuracy, and recurring themes from open-ended answers — and flag low-satisfaction areas. Use this skill whenever the user asks about survey results, attendee feedback/sentiment, satisfaction ratings, quiz scores, or poll results for an event. Requires the vFairs MCP connector (get_events, get_survey_result, get_attendees).
---

# Survey Insights & Sentiment Digest

Aggregates a vFairs event's survey, quiz, and poll responses into a readable digest: rating/choice distributions for structured questions, accuracy for quiz questions, and recurring themes for open-ended (free-text) questions. Flags low-satisfaction areas so organizers know where to follow up.

Scope: **survey_result data only**. This tool has no `session_id`/`speaker_id`/`booth_id` field, so a survey response can't be automatically linked to a specific session or speaker unless the survey/question naming makes that link explicit (see Step 3). Don't invent that link.

## When to use this skill

Trigger on requests like:
- "What did attendees think of [event]?"
- "Survey results / feedback digest for [event]"
- "What's our satisfaction rating for [event]?"
- "How did people score on the quiz?"
- "What are people saying in the open-ended feedback?"
- "Any low-satisfaction areas we should worry about?"

## Output format — read this before doing anything else

**This skill is table and charts first. Every response must contain at least two markdown tables and a chart if possible.** Organizers scan a ranked list; they don't read paragraphs describing what each rating question showed.

Rules:
- Never write a statistic in prose that could be a table row.
- Find opportunities to create charts when possible — a bar chart of rating distribution, or a pie chart of choice-question splits, works well here.
- Prose is for the verdict, the theme summaries, and the recommendation only — the "so what," not the "what."
- Minimum tables per response: the **Survey Overview** table (Step 2) and the **Question Summary** table (Step 3). Add the **Quiz Accuracy** table (Step 4) whenever a quiz is present, and the **Open-Ended Themes** table (Step 5) whenever free-text questions have responses.
- Put the headline verdict in one or two sentences *above* the first table, then the tables, then a short "What to do next" list.
- Use bold status markers in table cells: **Strong** / **Mixed** / **Low satisfaction**.

## Workflow

### Step 1 — Resolve the event

Always ask which event this is for if it isn't already named in the request — never default to "most recent" or "all events." Use `get_events` (filter by `name`) to resolve to an `app_id`. If more than one event matches, list them as a table (Event Name | Start Date | Type) and ask the user to pick one.

### Step 2 — Survey overview

Scope everything to the resolved `app_id` via `app_ids=[app_id]` on `get_survey_result`.

```
get_survey_result(
  select="survey_id, survey_title, survey_type, COUNT(DISTINCT question_id) AS questions, COUNT(DISTINCT user_id) AS respondents",
  group_by="survey_id, survey_title, survey_type",
  app_ids=[app_id]
)
```

**Gotcha:** `survey_type` is not in the allowed `filters` field list, even though it's selectable/groupable — you cannot filter rows by survey type directly. Use this overview call to learn which `survey_id`s are which type (survey / quiz / chat_survey / poll), then scope later calls by `survey_id` (which *is* filterable) when you need to isolate one.

Render the **Survey Overview** table:

| Survey | Type | Questions | Respondents |
|---|---|---:|---:|
| Post-Event Feedback | survey | 8 | 412 |
| Sponsor Trivia | quiz | 5 | 201 |
| Which track interests you most? | poll | 1 | 630 |

**Verify the respondent count before trusting it.** `COUNT(DISTINCT user_id)` silently undercounts on surveys that accept anonymous or public-link submissions (e.g. a QR-code survey not gated behind login) — in live testing, one event's general post-event survey showed `COUNT(DISTINCT user_id) = 2` for a survey that actually had 63 real submissions, because most anonymous responses shared one placeholder `user_id`. Cross-check by pulling `COUNT(*)` for a single non-CheckBox question in that survey (CheckBox rows can multiply per respondent, so don't use it for this check):
```
get_survey_result(select="question_id, COUNT(*) AS total_rows", group_by="question_id", filters=[{"field":"survey_id","op":"eq","value":survey_id}], app_ids=[app_id])
```
If `total_rows` for a representative question is much higher than the `COUNT(DISTINCT user_id)` figure, the survey has anonymous/shared-identity submissions — report the per-question row count as the respondent figure instead (label it "responses," not "unique respondents," since you can no longer confirm uniqueness), and note the discrepancy once rather than re-deriving it per table. Session-scoped surveys tied to a logged-in registration flow generally don't have this problem — check before assuming it does.

If the user named a specific survey, filter the rest of the workflow to that `survey_id`/`survey_title`. Otherwise cover all surveys for the event, grouped by survey where it aids readability.

For response-rate context, pull total registered attendees: `get_attendees(select="COUNT(*) AS total", app_ids=[app_id])`. State plainly that this is an approximate response rate — surveys aren't always scoped to attendees only (speakers/exhibitors can have their own forms too), so treat it as directional, not exact.

### Step 3 — Structured questions (Radio / CheckBox / DropDown / StarRating)

```
get_survey_result(
  select="question_id, question_text, question_type, answer_text, COUNT(*) AS count",
  group_by="question_id, question_text, question_type, answer_text",
  order_by="question_id, count DESC",
  filters=[{"field":"question_type","op":"in","value":["Radio","CheckBox","DropDown","StarRating"]}],
  app_ids=[app_id],
  limit=500
)
```
Exclude `Blank Paragraph` question_type — that's instructional/display text, not a real question, and will otherwise pollute the digest.

For **StarRating** questions specifically, also pull the numeric average:
```
get_survey_result(
  select="question_id, question_text, AVG(answer_text) AS avg_rating, COUNT(*) AS responses",
  group_by="question_id, question_text",
  filters=[{"field":"question_type","op":"eq","value":"StarRating"}],
  app_ids=[app_id]
)
```
If `AVG()` on `answer_text` errors (some environments won't implicitly cast a text column), fall back to pulling the raw per-answer distribution from the first call and computing the weighted average yourself from the option/count pairs.

Render the **Question Summary** table (one row per question):

| Question | Type | Respondents | Top Answer | % | Signal |
|---|---|---:|---|---:|---|
| How would you rate the event overall? | StarRating (avg 4.1/5) | 428 | 5 stars | 49% | **Strong** |
| Was the registration process easy? | Radio | 390 | Yes | 88% | **Strong** |
| How did you hear about us? | CheckBox | 401 | Email | 61% | — |
| Rate the exhibit hall experience | StarRating (avg 2.8/5) | 210 | 2 stars | 34% | **Low satisfaction** |

Signal heuristic (state this inline as an assumption, not a fixed target):
- **StarRating** — avg at/above 4.0 (of 5) is **Strong**; 3.0–3.9 is **Mixed**; below 3.0 is **Low satisfaction**. If the event uses a different star scale, adjust proportionally and say so.
- **Radio/CheckBox/DropDown that reads as a satisfaction/agreement scale** (options like "Poor/Fair/Good/Excellent" or "Disagree/Agree") — flag **Low satisfaction** if the two most-negative options together exceed ~25% of responses. This is a judgment call on the option wording, not a fixed rule — read the actual option text rather than assuming a scale exists.
- Questions with no clear rating/satisfaction framing (demographics, "how did you hear about us," etc.) get no signal marker — don't force sentiment onto neutral data.

If any question is flagged **Low satisfaction**, break out its full answer distribution into its own small table right below the summary, so the organizer sees exactly where the drop-off is:

| Rating | Count | % |
|---|---:|---:|
| 1 star | 42 | 20% |
| 2 star | 60 | 29% |
| 3 star | 55 | 26% |
| 4 star | 38 | 18% |
| 5 star | 15 | 7% |

### Step 4 — Quiz accuracy (if a quiz survey is present)

For each `survey_id` identified as `quiz` in Step 2:
```
get_survey_result(
  select="question_id, question_text, SUM(is_correct) AS correct, COUNT(*) AS total",
  group_by="question_id, question_text",
  filters=[{"field":"survey_id","op":"eq","value":quiz_survey_id}, {"field":"is_correct","op":"is_not_null","value":null}],
  app_ids=[app_id]
)
```
`is_correct` is NULL for free-text answers (not applicable to scoring) — the `is_not_null` filter keeps the accuracy math clean.

Render the **Quiz Accuracy** table:

| Question | Correct | Total | Accuracy |
|---|---:|---:|---:|
| What year was vFairs founded? | 178 | 201 | 89% |
| Which package includes lead retrieval? | 94 | 201 | 47% |

Call out in prose the single lowest-accuracy question — that's the one worth a follow-up (content gap, confusing wording, or a genuinely hard question), not a ranked list of all of them.

### Step 5 — Open-ended themes (TextField questions)

There's no theme-extraction tool — this is qualitative synthesis you do by reading the responses, not a database aggregation. Say so plainly if asked how the themes were derived.

**Skip PII-collection fields before theme-extracting.** Not every `TextField` question is open-ended feedback — event surveys commonly include a raffle/giveaway entry field ("include your email address for a chance to win...") that is technically `TextField` but contains only names, emails, or phone numbers. In live testing, one event's survey had exactly this — a raffle-entry email field mixed in among real feedback questions. Skim each `TextField` question's `question_text` before pulling its answers; skip (don't theme, don't quote) any question whose text is about collecting contact info for a drawing/follow-up rather than asking for an opinion. If unsure whether a field is feedback or contact-collection, sample a few `answer_text` values first — email addresses and bare names are the tell.

For each remaining (genuinely open-ended) `TextField` question with responses:
```
get_survey_result(
  select="question_id, question_text, answer_text, date_answered",
  filters=[{"field":"question_type","op":"eq","value":"TextField"}, {"field":"question_id","op":"eq","value":question_id}],
  order_by="date_answered DESC",
  app_ids=[app_id],
  limit=200
)
```
If a question has more than ~200 responses, this is a sample, not the full set — say so explicitly ("themes below are drawn from the 200 most recent of N responses") rather than presenting it as exhaustive.

Read through the sample and group answers into 3–6 recurring themes. Render the **Open-Ended Themes** table:

| Theme | Approx. Mentions | Example Quote |
|---|---:|---|
| Wanted more networking time | ~34 | "Would've loved a dedicated networking hour between sessions." |
| Praised the platform's ease of use | ~51 | "So much easier to navigate than last year's platform." |
| Audio/video issues in sessions | ~19 | "Kept freezing during the keynote." |

Label counts as approximate (`~N`), since theme assignment is a judgment call, not an exact query. Never fabricate a theme with no supporting quotes in the sample. Don't attribute a specific quote to a named respondent in the output by default — treat open-ended answers as pooled/anonymous unless the user explicitly asks for a specific attendee's response and has independent reason to know they answered.

### Step 6 — Close with recommendations

After the tables, add a short **What to do next** list — 2 to 4 concrete actions tied to what the data actually showed. Examples:
- A **Low satisfaction** StarRating question → name the specific area (exhibit hall, platform, catering) and suggest it as a priority fix for the next event
- Low quiz accuracy on a specific question → may indicate a content gap worth addressing in pre-event materials, not just a quiz-difficulty issue
- A recurring open-ended theme (e.g., "wanted more networking time") → concrete, attendee-sourced input for next year's agenda design
- Low response rate relative to registered attendees → consider a shorter survey or an incentive next time, since sentiment on a small sample is less reliable

Only recommend things the data supports. Don't pad the list to reach four items.

## Notes

- Never display raw `survey_id` / `question_id` / `user_id` values to the user — refer to surveys and questions by their title/text.
- `survey_type` can be selected and grouped but **not** filtered via `filters` — scope by `survey_id` instead once you know which survey is which type from the Step 2 overview.
- There is no field linking a survey response to a specific session, speaker, or booth. A link only exists if the survey/question naming makes it explicit (e.g., a survey titled "Feedback: AI Trends Keynote"). Don't assume a generic "Rate this session" question applies to a particular session unless the survey is scoped to it by name.
- `get_survey_result` defaults to a 100-row limit (max 500) — raise it for full distributions and paginate with `offset` if a question has more responses than one page returns.
- Open-ended theme counts are approximate and based on a read-through sample, not an exact query — always say so, and never present a theme count with false precision (e.g., say "~30", not "31").
- If `get_survey_result` returns very few respondents for a question (under ~15), skip the distribution table/chart for that question and report the raw numbers plainly instead — a pie chart of single-digit counts is noise, not signal.
- `question_text` can include inline HTML (e.g. `<br>`) and a bilingual (e.g. English/Spanish) combined label — display it cleanly (strip obvious markup) but don't drop the second language, some events' respondent base skews toward it.
- Never surface a respondent's raw contact info (email/phone/name typed into a "for a raffle drawing" style field) in a themed quote or anywhere in the output — treat those `TextField` questions as out of scope for sentiment analysis entirely, per Step 5.
