# PRD: App Replicator (working name)

## Goal
Point a browser at any web app you have a login for. In minutes, get a working Python, Postgres and Redis copy of it, plus a score showing how closely the copy matches the original.

Demo target: a plain CRUD app (contacts or inventory) running locally, so the demo is fast and safe. Built for education and for migrating apps you own or are licensed to move.

## Success criteria for the demo
1. One command starts the proxy and the explorer.
2. User logs in by hand in the opened browser, then presses Enter in the terminal.
3. Explorer runs for 5 to 10 minutes with no input.
4. A generated app starts with one command and serves the same pages and routes.
5. A report shows the replay match percentage and a list of mismatches.

## Users
Developers and students migrating a legacy app (PHP, Perl, Rails, anything) to Python.

## Architecture
Five stages, each reading and writing a shared SQLite file (`recording.db`) and a `spec.json`.

1. **Capture** (mitmproxy addon). Logs every request and response to SQLite. Stores method, URL, headers, body, status, timing, and the id of the current explorer action.
2. **Explore** (Playwright via the proxy). Reuses the logged in browser profile. Per page: screenshot, list links, buttons and forms, then act on them. Fills forms with tagged test data (e.g. `zz_test_<n>`). Logs each action with a timestamp.
3. **Probe** (experiment runner). Runs scripted experiments on each form and each calculated value:
   - create a record, then find where the tagged values appear later
   - edit one field at a time and record what changes
   - delete and record what disappears
   - submit empty, oversized, wrong type and duplicate values
   - vary numeric inputs across 0, 1, 2, 10, 100, negative, decimal, huge
   - repeat actions in different statuses to learn allowed transitions
4. **Interpret** (LLM). Reads the normalised traffic and experiment results and writes `spec.json`: entities, fields, types, relationships, routes, validation rules, formulas, workflows, and a "needs human to confirm" list.
5. **Generate and verify.**
   - Generator turns `spec.json` into a FastAPI app, SQLAlchemy models, an Alembic migration, Jinja templates (or copied static assets for JSON apps), Redis for sessions and a job queue, and seed data from captured responses.
   - Verifier replays every recorded request against the replica, ignores volatile values (timestamps, CSRF tokens, generated ids, session cookies), and diffs the rest.
   - Failures go back to the LLM as failing tests. Loop up to N rounds.

## Normalised record (the common format)
Each interaction becomes one row: action id, request, response, screenshot path, and a type tag: `html`, `json`, `xml`, `redirect`, `download`, or `fragment`. Everything after capture works from this table, so the source language does not matter.

## Scope for version 1
In scope:
- Cookie session apps, one login
- Server rendered HTML and JSON APIs
- Create, read, update, delete, list and search
- Numeric formula inference
- Status workflow inference
- Replay match report

Out of scope for now:
- Rebuilding the front end from screenshots
- Emails, scheduled jobs, file processing, third party API calls (listed under "needs human to confirm")
- Multiple roles and permission inference (version 2)
- WebSockets and streaming

## Safety rails
- Only run against apps you own, are licensed to use, or are migrating.
- Run against a test or staging copy where possible, because the explorer creates and deletes data.
- Allowlist of hosts, a request rate limit, and a dry run mode that records but does not submit.
- Never store real credentials; session cookies stay in the local browser profile.

## Tech stack
Python 3.12, mitmproxy, Playwright, SQLite for recordings, Postgres and Redis for the generated app (Docker Compose), FastAPI, SQLAlchemy, Alembic, Jinja2, Claude API for the interpret and fix steps.

## Repo layout
```
capture/addon.py        mitmproxy addon
explore/crawler.py      Playwright explorer
probe/experiments.py    CRUD, validation and formula probes
interpret/spec.py       LLM spec builder
generate/               templates and code generator
verify/replay.py        replay and diff
demo/target_app/        simple CRUD app to demo against
run_demo.sh             one command start
```

## Build order for Claude Code (fits one sitting)
1. `demo/target_app`: small CRUD app with one calculated field and a status field, so there is something to discover.
2. Capture addon plus SQLite schema.
3. Explorer: login handoff, click and fill loop, screenshots.
4. Probes: create, edit and delete tracing first, then validation, then formulas.
5. Interpret: prompt plus JSON schema for `spec.json`.
6. Generator: models, routes, templates, Compose file.
7. Verifier: replay, normalise, diff, report.
8. `run_demo.sh` and a README with the demo script.

## Risks
- LLM guesses wrong on hidden logic: mitigated by testing guesses on unseen inputs and the replay loop.
- Explorer gets stuck or destroys data: step limits, a staging copy, and a hard stop.
- Dynamic JavaScript pages: Playwright waits for network idle and logs fragment requests.
- Demo time: pre cache LLM results for the demo app so the live run stays under a few minutes.
