---
spec_id: 001-persistent-tasks-in-postgres-with-a-web
title: Persistent tasks in Postgres with a web UI
status: approved
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks in Postgres with a web UI

## Problem

The taskboard4 Task API can create and list tasks, but it keeps them in process memory: every restart or redeploy wipes them, and the API cannot run more than one instance without each instance seeing different tasks. The only way to use tasks today is to call the API by hand; the web app still shows the template starter page and does not talk to the API. People who want to record what needs doing therefore have nowhere durable to put it and no screen to work in. This feature keeps tasks in a Postgres database owned by the system, so they survive restarts and are shared by every API instance, and lets people see and create tasks in the web app.

## Stories

As a person using taskboard4, I want the tasks I create to still be there after the system restarts or is redeployed, so that I can rely on it as my record of what needs doing.
**Repos:** taskboard4-api, taskboard4-gitops

As a person using taskboard4, I want to open the web app and see every task with its due date, so that I know what needs doing without calling the API by hand.
**Repos:** taskboard4-web, taskboard4-api, taskboard4-gitops

As a person using taskboard4, I want to create a task with a title and an optional due date in the web app, so that I can record new work where I see it.
**Repos:** taskboard4-web, taskboard4-api

## Acceptance criteria

- **AC-001.1** Given tasks "A" (no due date) and "B" (due `2026-12-31`) were created through `POST /tasks`, when every instance of the Task API is restarted and a caller then sends `GET /tasks`, then the response is `200` with both tasks, oldest first, each with the same `id`, `title` and `due_date` as returned when it was created.
- **AC-001.2** Given the Task API runs as two instances (as it does in dev), when a caller creates a task through `POST /tasks` and then sends `GET /tasks` ten times, then every one of the ten responses contains that task, whichever instance answers.
- **AC-001.3** Given the persistent storage, when callers use `POST /tasks` and `GET /tasks`, then the existing request and response contract is unchanged: `201` with `{id, title, due_date}`, titles trimmed and 1 to 200 characters, `due_date` a `YYYY-MM-DD` calendar date or `null` (past dates accepted), `422` naming the offending field with no task stored, and `GET /tasks` a bare JSON array oldest first.
- **AC-001.4** Given the database is unreachable, when a caller sends `POST /tasks` with a valid body or sends `GET /tasks`, then the response is `503` within 10 seconds, and no task is created.
- **AC-001.5** Given the database is unreachable, when the API's readiness endpoint is checked, then it reports not ready; and given the database becomes reachable again, when a caller sends `GET /tasks`, then the response is `200` with all tasks created before the outage, without the API being restarted.
- **AC-001.6** Given tasks "A", "B" and "C" were created in that order (through the web app or the API), when a person opens the web app's root URL, then the page lists the three tasks in creation order, oldest first, each showing its title and its due date as stored in the form `YYYY-MM-DD` (for example "2026-12-31"), or "No due date" when it has none.
- **AC-001.7** Given no task exists, when a person opens the web app's root URL, then the page shows the message "No tasks yet" and the form to create a task.
- **AC-001.8** Given the web app's root URL is open, when a person enters the title "Buy milk", leaves the due date empty and submits, then "Buy milk" appears in the list with "No due date" without the person reloading the page, the form is cleared, and `GET /tasks` on the API returns a task titled "Buy milk" with `due_date` `null`.
- **AC-001.9** Given the web app's root URL is open, when a person enters the title "File taxes", picks the due date 31 December 2026 and submits, then the task appears in the list with the due date "2026-12-31" and `GET /tasks` returns it with `due_date` `"2026-12-31"`; a due date in the past is accepted the same way.
- **AC-001.10** Given the web app's root URL is open, when a person submits with an empty title or a title of only spaces, then the message "Title is required" is shown next to the title field and no task is created; when a person types a title, then the field accepts at most 200 characters.
- **AC-001.11** Given the Task API is unreachable or answers with an error, when a person opens the web app's root URL, then the page still loads and shows "Tasks could not be loaded. Please try again later." instead of the list.
- **AC-001.12** Given the Task API is unreachable or answers with an error, when a person submits a valid task, then the page shows "The task could not be saved. Please try again.", keeps the entered title and due date in the form, and no task is added to the list.

## Non-goals

- Keeping the tasks that exist in the in-memory API at the time of the switch: they are lost once, as on any restart today.
- Reading a single task, editing, completing or deleting tasks, in the API or the web app.
- Filtering, searching, sorting options or paginating the list.
- Users, sign-in or per-user task lists: everyone sees and creates every task.
- Due times or time zones: a due date is a calendar date only.
- Fields beyond `id`, `title` and `due_date`.
- Live updates: a task created elsewhere appears in an open web page only after the page is reloaded.
- Changing whether the Task API stays reachable from outside the system.
- Running in staging or prod: this spec covers dev; promotion follows the usual pin policy.
- Database backups: a separate spec, before staging or prod run the database.
- The template starter page: the root page is replaced by the task list and create form.

## Open questions

- ~~How should the API answer when the database is unreachable?~~ Answered: `503` within 10 seconds, readiness reports not ready, recovery without a restart, see AC-001.4 and AC-001.5.
- ~~Should the Task API run as two instances in dev?~~ Answered: yes, two instances in dev, see AC-001.2.
- ~~Should the database be backed up in this spec?~~ Answered: no, a separate spec before staging or prod, see Non-goals.
- ~~In which order should the web app list tasks?~~ Answered: oldest first, same as the API, see AC-001.6.
- ~~How should the web app show a due date?~~ Answered: `YYYY-MM-DD` as stored, see AC-001.6 and AC-001.9.
- ~~Confirm the user-facing wording.~~ Answered: as written, see AC-001.6, AC-001.7, AC-001.10, AC-001.11 and AC-001.12.
- ~~Confirm: the title field stops input at 200 characters.~~ Answered: yes, see AC-001.10.
- ~~Confirm: the task list and create form are on the web app's root page.~~ Answered: yes, replacing the starter page, see AC-001.6 to AC-001.12 and Non-goals.

## Changelog

- 2026-11-06 created
- 2026-11-06 amended: answered all open questions; AC-001.2 states two API instances in dev; AC-001.6 and AC-001.9 show due dates as `YYYY-MM-DD`; database backups and the starter page added to Non-goals.
- 2026-11-06 approved
