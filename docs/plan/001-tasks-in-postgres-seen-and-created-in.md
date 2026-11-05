---
plan_id: 001-tasks-in-postgres-seen-and-created-in
spec: 001-tasks-in-postgres-seen-and-created-in
feature: Tasks stored in the system's Postgres and shown and created in the web app
gitops_app: ika100-demo/taskboard3-gitops
status: draft
repos:
  - id: taskboard3-gitops
    shape: gitops-app
    summary: Postgres for the taskboard3 system, wired to the API, and the API's URL wired to the web app
    acs: [AC-001.1, AC-001.2, AC-001.6, AC-001.11, AC-001.16]
    gitops:
      - {resource: postgres}
      - {uses: postgres, component: taskboard3-api}
      - {env: {TASKS_API_URL: 'http://taskboard3-api'}, component: taskboard3-web}
    depends_on: []
    done: false
  - id: taskboard3-api
    shape: service-python
    summary: Store tasks in Postgres instead of memory, with 503 and not-ready while the database is down
    acs: [AC-001.1, AC-001.2, AC-001.3, AC-001.4, AC-001.5]
    arguments: |
      Builds on the API's spec 001 (in memory): keep the routes, models and validation of
      src/taskboard3_api/tasks/ and replace the in-memory TaskStore behind get_task_store with a
      Postgres store. Connection settings come from DATABASE_URL (PG* are also set); see the
      plan's Contract for the table, ordering, 503 body, readiness and timeouts. The API's
      acceptance tests need a real Postgres (a throwaway one in tests/CI), not a mock.
    depends_on: []
    done: false
  - id: taskboard3-web
    shape: web-nextjs
    summary: Task page at the root URL that lists tasks and adds a task through the Task API
    acs: [AC-001.6, AC-001.7, AC-001.8, AC-001.9, AC-001.10, AC-001.11, AC-001.12, AC-001.13, AC-001.14, AC-001.15, AC-001.16]
    arguments: |
      Replaces the starter page in app/page.tsx. The API is reached only server side, through
      TASKS_API_URL (add it to docs/env-vars.md); the browser never calls the API directly.
      Component and e2e tests stub the API at TASKS_API_URL with the responses of the Contract.
    depends_on: []
    done: false
gitops_pin: []
---

# Plan 001 — Tasks in Postgres, seen and created in the web UI

Three repos, one level: `taskboard3-gitops`, `taskboard3-api` and `taskboard3-web` all have `depends_on: []` and build in parallel against the Contract below. No repo needs another repo's code. The web app builds against the Contract, and the API keeps the contract it already has (AC-001.3). The deploy order is handled by `/system:ship` (spec 144): it runs the gitops operations and pins every build of this spec on `compose/<spec>`, so the Postgres resource, `DATABASE_URL`, `TASKS_API_URL` and the new images reach dev in one PR. The new API pod is never deployed without its database, and the web app never without the API's URL.

No pins: the spec delivers in dev only (staging and prod are non-goals), and dev tracks `main`, so `gitops_pin` is empty.

## taskboard3-gitops — Postgres for the system, wired to the API, and the API's URL wired to the web app

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.2, AC-001.6, AC-001.11, AC-001.16 · **Depends on:** []

The data and the wiring that make the criteria possible in a deployed environment:

- `{resource: postgres}`: the system's database, one per environment (only dev in this spec). It belongs to the system, not to a pod, so tasks survive an API restart or redeploy (AC-001.1). Every API instance reaches the same database (AC-001.2).
- `{uses: postgres, component: taskboard3-api}`: the API gets `DATABASE_URL` and `PG*`.
- `{env: {TASKS_API_URL: 'http://taskboard3-api'}, component: taskboard3-web}`: the web server reaches the API through its in-cluster Service (`taskboard3-api`, port 80). The web app shows exactly what `GET /tasks` returns (AC-001.6, AC-001.16), and a task created in the form reaches the API (AC-001.11).

Both components are already exposed (`expose.host` in `system.yaml`), so no `expose` operation is needed. Replicas stay at the default: AC-001.2 follows from the shared database and is proven in the API's tests with two app instances on one database, not by scaling dev.

## taskboard3-api — Store tasks in Postgres, with 503 and not-ready while the database is down

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2, AC-001.3, AC-001.4, AC-001.5 · **Depends on:** []

The API's spec 001 shipped `POST /tasks` and `GET /tasks` in memory. This repo moves the store to Postgres and leaves the HTTP contract as it is.

- **AC-001.1 / AC-001.2:** tasks live in the `tasks` table (see Contract). The list order comes from a database sequence, not from per-process state, so the order is the same on every instance.
- **AC-001.3:** the existing acceptance tests of spec 001 must keep passing unchanged, now against Postgres.
- **AC-001.4:** connection errors and timeouts become `503` with the body from the Contract. Each create is a single-statement insert in its own transaction, so a failed create leaves no row. The pool reconnects on the next request. The API must start while the database is down (no crash, no `sys.exit`), and it creates the schema on the first successful connection.
- **AC-001.5:** `/ready` checks the database. `/health` never touches it.
- Test concerns: use a real Postgres. Simulate an outage by stopping it or pointing at a closed port, then bring it back and repeat the requests without restarting the app. For AC-001.2, run two app instances (two `create_app()`/TestClients) on the same database. Keep the existing smoke tests: `POST` is `@writes`, `GET /tasks` and `/ready` are read-only.

## taskboard3-web — Task page at the root URL that lists and adds tasks

**Shape:** `web-nextjs` · **Criteria:** AC-001.6 to AC-001.16 · **Depends on:** []

The root route `/` becomes the task page. It is rendered dynamically on every request (no static rendering, no fetch cache), so a reload always shows the current `GET /tasks` result (AC-001.6, AC-001.16).

- The list is rendered on the server from `GET {TASKS_API_URL}/tasks`, in the order the API returns it. The page never re-sorts.
- `due_date` is shown as the exact string from the API (`YYYY-MM-DD`). It is never passed through `Date` or locale formatting (AC-001.8). When it is `null`, nothing is shown for the date (AC-001.7).
- The form (title text field, optional `<input type="date">`) submits through a Next.js Server Action that calls `POST {TASKS_API_URL}/tasks` on the server. On `201`, the returned task is appended to the list shown, and the form is cleared (AC-001.11). The list shows the title **from the response**, which is trimmed (AC-001.14).
- The form validates on the client before calling the API: trim the title, then reject an empty title ("Please enter a title.") or one longer than 200 characters ("The title can be at most 200 characters."). Characters are counted as Unicode code points (`[...title].length`), the same as the API's Python `len`. In both cases nothing is sent (AC-001.12, AC-001.13).
- The exact UI texts are those of AC-001.9, AC-001.10, AC-001.12, AC-001.13 and AC-001.15. The load and save errors are rendered in the page, never through `error.tsx` and never with a stack trace.
- `/api/ready` and `/api/health` must **not** depend on the API: the page has to load while the API is down (AC-001.10).
- Test concerns: unit and component tests stub the API with each response the Contract lists (and with a timeout and a refused connection). Add a read-only `@smoke` test that the root page loads with a list, "No tasks yet." or the load error text, plus a `@smoke @writes` test that creates a `smoke-<uuid>` task through the form and sees it after a reload.

## Contract

Between `taskboard3-web` (caller, server side only) and `taskboard3-api`, plus the API's own database layout and probes. Base URL in the cluster: `TASKS_API_URL=http://taskboard3-api` (no trailing slash). All bodies are JSON (`Content-Type: application/json`).

### Task object

```json
{"id": "3f2c9a8e-...", "title": "Buy milk", "due_date": "2026-12-31"}
```

| Field | Type | Notes |
|---|---|---|
| `id` | string, non-empty | Opaque. A UUID string today; the caller must not parse it. |
| `title` | string | Already trimmed, 1 to 200 characters. |
| `due_date` | string `YYYY-MM-DD` or `null` | A calendar date, no time and no time zone. Callers display it as is. |

No other fields. A caller ignores any extra field it does not know.

### `GET /tasks`

- `200` with a JSON array of Task objects, oldest first (creation order). `[]` when there are none. No pagination.
- `503` when the database is unavailable (see Errors).

### `POST /tasks`

Request body: `{"title": <string>, "due_date": <"YYYY-MM-DD" | null>}`. `due_date` may be omitted (same as `null`). The web app sends `null` when no date is picked, and sends the title as typed (the API trims it).

- `201` with the created Task object.
- `422` on invalid input (FastAPI's validation format, see Errors). The rules are unchanged from the API's spec 001: `title` is required, a string, trimmed, 1 to 200 characters; `due_date` is a valid calendar date in `YYYY-MM-DD` or `null`, and past dates are accepted.
- `503` when the database is unavailable. No task was stored.

### Probes of the API

- `GET /health`: `200 {"status": "ok"}` as long as the process runs. It never checks the database.
- `GET /ready`: `200 {"status": "ready"}` when `SELECT 1` succeeds within 1 s. Otherwise `503 {"status": "not ready"}`.

### Storage (API internal, listed so both sides know where the order comes from)

Table `tasks`, created by the API on startup or on the first successful connection (`CREATE TABLE IF NOT EXISTS`):

| Column | Type | |
|---|---|---|
| `seq` | `bigint generated always as identity` | Primary key. The list is `ORDER BY seq`. Never exposed. |
| `id` | `text not null unique` | The UUID returned as `id`. |
| `title` | `varchar(200) not null` | |
| `due_date` | `date null` | Serialized back as `YYYY-MM-DD`. |

Connection settings: `DATABASE_URL` (set by the gitops `uses: postgres` operation). Tasks that the running in-memory API holds are not migrated (a non-goal).

### Errors

- **`422`** (unchanged, FastAPI default): `{"detail": [{"type": "...", "loc": ["body", "title"], "msg": "...", "input": ...}, ...]}`. `loc` names the field (`title` or `due_date`).
- **`503`** (new): `{"detail": "database unavailable"}`. The API sends it for any database connection error, pool timeout or statement timeout on `/tasks`. Other unexpected server errors stay `500`, also with a JSON `{"detail": ...}` body and no stack trace.
- **How the web app treats responses:**
  - `GET /tasks`: only a `200` with a JSON array counts as success. Any other status (`503`, `500`, `404`, ...), a body that is not a JSON array, a refused connection or a timeout shows "Tasks could not be loaded. Please try again later." instead of the list. The page itself still returns `200`.
  - `POST /tasks`:
    - `201`: append the returned task and clear the form.
    - `422` whose `detail` has an entry with `"title"` in `loc`: show the title message that fits the trimmed input ("Please enter a title." when empty, otherwise "The title can be at most 200 characters."). Normally the client check catches these first.
    - Any other `422`, any other status, a body that is not a Task object, a refused connection or a timeout: show "The task could not be saved. Please try again.", keep the title and date in the form and add nothing to the list.
- The web app never shows response bodies, status codes or stack traces to the user. It logs them on the server.

### Timeouts

| Call | Timeout | When the other side is down or slow |
|---|---|---|
| web server → `GET /tasks` | 5 s total (`AbortSignal.timeout(5000)`) | The page renders with the load error text. No retry inside the request; the user reloads. |
| web server → `POST /tasks` | 5 s total | The save error text, and the form keeps its input. **No automatic retry** (a retry after a timeout could create the task twice). |
| API → Postgres, connect / pool checkout | 2 s | `503 {"detail": "database unavailable"}` |
| API → Postgres, statement (`statement_timeout`) | 5 s | `503`. The insert's transaction is rolled back, so no partial task. |
| API `/ready` → `SELECT 1` | 1 s | `503 {"status": "not ready"}`. Kubernetes takes the pod out of the Service; `/health` stays `200`, so the pod is not restarted. |

The API does not cache connection failures. Each request tries the pool again, so service resumes without a restart once the database is back (AC-001.4). The web's 5 s budget covers the API's worst case (2 s connect plus the query), so a database outage reaches the user as a `503`, not as a web timeout.

## gitops-app PR

The `taskboard3-gitops` entry runs first through `/system:ship` (`cplat resource add postgres`, `cplat component set taskboard3-api --uses postgres`, `cplat component set taskboard3-web --env TASKS_API_URL=http://taskboard3-api`). After that `devbox run render` and `devbox run validate` must be green. There are no pins: only dev is in scope, and dev tracks `main`.

After Argo reconciles dev:

- The Postgres resource is healthy, and the `taskboard3-api` pod has `DATABASE_URL` (check the env names only, never print the value).
- The `taskboard3-web` pod has `TASKS_API_URL=http://taskboard3-api`.
- After the API's PR merges: `/ready` of the API returns `200`. `POST /tasks` and then a restart of the API pod (`kubectl rollout restart`) still show the task in `GET /tasks`.
- After the web PR merges: `http://taskboard3-web.taskboard3-dev.localhost/` shows the tasks of `GET /tasks`. A task added in the form is still shown after a reload.
- Both components' dev smoke tests are green.
