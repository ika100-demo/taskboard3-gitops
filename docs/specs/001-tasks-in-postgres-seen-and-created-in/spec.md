---
spec_id: 001-tasks-in-postgres-seen-and-created-in
title: Tasks in Postgres, seen and created in the web UI
status: done
priority: P1
shape: gitops-app
---

# 001 — Tasks in Postgres, seen and created in the web UI

## Problem

The Task API can create and list tasks, but it keeps them in process memory. Every restart or redeploy (which happens on every merge to main in dev) deletes all tasks, and two running instances would each show a different list. The web app is still the starter page, so a person who is not using API tools has no way to see or add tasks. This spec makes tasks durable in a Postgres database that belongs to the system, and gives people a page in the web app where they can see the task list and add a task. The task contract of the API (fields, validation, order) stays as it is.

## Stories

- As a taskboard user, I want the tasks I created to still be there after the system was restarted or redeployed, so that I can rely on the list.
  **Repos:** taskboard3-gitops, taskboard3-api
- As a taskboard user, I want to open the web app and see all tasks with their due dates, so that I know what has to be done without using API tools.
  **Repos:** taskboard3-gitops, taskboard3-web, taskboard3-api
- As a taskboard user, I want to add a task with a title and an optional due date in the web app, so that I can record something to do in a few seconds.
  **Repos:** taskboard3-gitops, taskboard3-web, taskboard3-api
- As a taskboard user, I want a clear message when tasks cannot be loaded or saved, so that I know my input did not get lost silently.
  **Repos:** taskboard3-web, taskboard3-api

## Acceptance criteria

Durable storage (API):

- **AC-001.1** Given tasks A, B and C were created through `POST /tasks`, when the API is restarted or redeployed, then `GET /tasks` still returns A, B and C in creation order, each with the same `id`, `title` and `due_date` as its create response.
- **AC-001.2** Given the API runs as more than one instance in an environment, when a task is created through one instance, then `GET /tasks` answered by any instance includes that task.
- **AC-001.3** Given the database is in use, when clients call `POST /tasks` and `GET /tasks`, then requests, responses, validation errors and list order are exactly as in the API's spec 001 (title required, trimmed, 1 to 200 characters; optional `due_date` as `YYYY-MM-DD`, past dates accepted; `201` with `{id, title, due_date}`; `422` on invalid input; list in creation order).
- **AC-001.4** Given the database cannot be reached, when a client sends `POST /tasks` or `GET /tasks`, then the response is `503` with a JSON error body, no task is lost or half-created, and once the database is reachable again both requests work without restarting the API.
- **AC-001.5** Given the database cannot be reached, when the API's readiness endpoint is called, then it reports not ready; the health endpoint keeps reporting healthy.

Seeing tasks (web):

- **AC-001.6** Given tasks exist, when a person opens the web app's root URL, then the page shows every task's title and due date, in creation order (oldest first), the same set of tasks that `GET /tasks` returns.
- **AC-001.7** Given a task without a due date, when it is shown in the list, then it shows the title and no date (no placeholder such as "null" or "Invalid date").
- **AC-001.8** Given a task with a due date, when it is shown in the list, then the date is shown as the same calendar day that the API returns (for `2026-12-31`: 31 December 2026, never shifted by a time zone), formatted as `YYYY-MM-DD`.
- **AC-001.9** Given no task exists, when a person opens the web app's root URL, then the page shows the text "No tasks yet." instead of an empty list.
- **AC-001.10** Given the API or the database is unavailable, when a person opens the web app's root URL, then the page still loads and shows the text "Tasks could not be loaded. Please try again later." instead of the list, and no error page or stack trace is shown.

Creating tasks (web):

- **AC-001.11** Given the task page, when a person enters a title, optionally picks a due date and submits the form, then the new task appears at the end of the list without the person reloading the page, the form is emptied, and the task is also returned by `GET /tasks` and still shown after a reload.
- **AC-001.12** Given the task page, when a person submits the form with an empty or whitespace-only title, then no task is created and the text "Please enter a title." is shown next to the title field.
- **AC-001.13** Given the task page, when a person enters a title of more than 200 characters, then the task is not created and the text "The title can be at most 200 characters." is shown next to the title field; a title of exactly 200 characters is accepted.
- **AC-001.14** Given a title with leading or trailing spaces, when it is submitted in the web app, then the list shows the trimmed title, the same as the API returns.
- **AC-001.15** Given the API or the database is unavailable, when a person submits a valid form, then the text "The task could not be saved. Please try again." is shown, the entered title and due date stay in the form, and no task appears in the list.
- **AC-001.16** Given a task created through the API directly, when a person opens or reloads the web app's root URL, then that task is shown in the list.

## Non-goals

- Editing, completing, deleting or reordering tasks, in the API or the web app.
- Filtering, searching, sorting other than creation order, and pagination.
- Users, sign-in and per-user task lists; for now anyone who can reach the web app can see and add every task.
- Live updates of the list when someone else adds a task (a reload shows it).
- Moving the tasks held in memory by the currently running API into the database; they are lost once, on the switch.
- Changes to the API's task contract (new fields, new endpoints, other error codes for invalid input).
- Running the components and the database in staging or prod; this spec is delivered in dev. Each environment will have its own database when promoted.
- Database backups and restore; they come in a separate spec before the system is promoted to staging or prod.
- An upper limit on the number of tasks.

## Open questions

- ~~Confirm: tasks in the list are shown oldest first (creation order, as the API returns them).~~ Answered: yes, see AC-001.6 and AC-001.11.
- ~~How should a due date be displayed in the web app?~~ Answered: as `YYYY-MM-DD`, exactly as the API returns it, no locale formatting, see AC-001.8.
- ~~Confirm the texts users see.~~ Answered: yes, in English, exactly as in AC-001.9, AC-001.10, AC-001.12, AC-001.13 and AC-001.15.
- ~~Confirm: when the database is down the API answers `503` and reports not ready, but stays healthy.~~ Answered: yes, see AC-001.4 and AC-001.5.
- ~~Confirm: the tasks currently held in the running API's memory may be lost on the switch to the database.~~ Answered: yes, the loss of the dev demo data is accepted, see Non-goals.
- ~~Should database backups be part of this spec?~~ Answered: no, a separate spec before promoting to staging or prod, see Non-goals.
- ~~Is the web app open to anyone who can reach it, with no sign-in?~~ Answered: yes, for now, see Non-goals.

## Changelog

- 2026-11-05 created
- 2026-11-05 amended: all open questions answered as suggested (oldest-first order, `YYYY-MM-DD` dates, the five English UI texts, `503` and not ready while the database is down, in-memory tasks may be lost, no backups, no sign-in); criteria unchanged, non-goals on backups and sign-in made explicit
- 2026-11-05 approved
- 2026-11-06 building
- 2026-11-06 done
