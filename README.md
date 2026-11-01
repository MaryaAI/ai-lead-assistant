# AI lead assistant — Propwise take-home

An AI assistant for a real-estate CRM: it reads a lead's WhatsApp or email message, works out what
they want, recommends listings from the agent's own agency, and drafts a follow-up task that is only
saved once the agent approves it.

**This repository contains only the code I wrote.** The CRM it plugs into — the leads, tasks and
properties API, the Next.js and Expo apps, the seed data — was provided as a starting point by the
company running the assessment. That scaffold and the assessment brief are theirs, not mine to
publish, so they are not here. Consequently **this repo is a code showcase and will not build or run
on its own**: the assistant module imports the scaffold's entities and services.

## What is here

| | |
|---|---|
| `api/src/assistant/` | The whole feature — 35 files: extraction, policy, agent, guardrails, routes |
| `api/evals/` | A 17-case eval suite against the real model, with a harness, runner, report and a saved trace per case |
| `api/test/` | The end-to-end specs for the assistant routes |
| `web/src/`, `mobile/src/` | The assistant panel and its cards, on both platforms |
| `NOTES.md` | The decisions I would defend, and an honest list of what is weak or missing |
| `PLAN.md` | The phase-by-phase plan, with an amendment for every deliberate deviation and every bug found |
| `docs/ARCHITECTURE_AND_PLAN.md` | The design document the plan derives from (about half Arabic) |

Not included, besides the scaffold: the assessment brief, and the project-instructions file that
quoted its rules and API contract verbatim.

## How it works

Seven stages. The split is the point: **the model reads language, code decides every fact and owns
the only write.**

1. **Context** — the lead, the analysed message, the last 10 messages, the agency's areas.
2. **Extract** (model, structured output) — requirements, intent, follow-up timing. Every field must
   quote the message it came from, and code checks the quote is really there.
3. **Merge** (code) — extracted fields overlay the previous state, field by field.
4. **Route** (code) — `search`, `property_question`, `change_follow_up`, `rent`, `clarify`, `closed`.
   Three of those never call the model at all.
5. **Agent** (model + read-only tools) — `search_properties` and `get_property`. The agency id is
   not a tool parameter; it lives in a closure, so the model cannot ask about another agency.
6. **Guardrails** (code) — every recommendation is re-fetched from the database and re-checked: it
   exists, belongs to this agency, is available, appeared in this run's own tool output, and meets
   the hard constraints.
7. **Propose** (code) — one follow-up task, pending approval. Approving is the only code path in the
   feature that creates a task.

A follow-up chat turn becomes a **new run** carrying a parent pointer, so requirements accumulate
without the previous proposal being lost or silently duplicated.

## The parts I would talk about

**There is no write tool.** Two read-only tools and nothing else, so there is nothing for an
injected instruction to execute.

**Extraction is evidence-gated**, which is what makes "a message is data, not instructions"
enforceable rather than aspirational: text can argue for anything, but it cannot produce a
requirement the lead did not state. A review found one field — the flag that closes a lead — reaching
the router ungated, which meant injected text could suppress the follow-up. That is fixed and tested.

**Dates anchor on whose words they were.** A lead's "tomorrow" is the day after *their message* was
sent, in the agent's timezone, even when that lands on a weekend. The agent's own "move it to
Tuesday" anchors on now. All date maths goes through luxon, never a local `Date` method, because the
API is graded with `TZ=UTC`.

**Requirements record what the lead said; matching is allowed to be kinder.** "Around 2M" is stored
as 2,000,000 and matched 5% higher, so a recommendation can sit just above the reported budget —
deliberately, with the real ceiling disclosed in the response rather than hidden.

`NOTES.md` has the full list, including the things that are still weak.

## Tests

233 across the three apps: 124 API unit, 66 API end-to-end against real Postgres with a mocked
model, 27 web, 16 mobile — plus the 17 eval cases against the real model, covering extraction,
matching, no-suitable, ambiguous dates, prompt injection, tool failure, tool choice and
no-action-without-approval.

The eval report in `api/evals/REPORT.md` shows 17/17, but it predates the last round of fixes and
could not be regenerated: the assessment API key ran out of credit. `NOTES.md` says so plainly
rather than leaving a stale green result looking current.
