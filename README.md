# QA API Pilot

A reusable scaffold for setting up automated API testing on any REST API, built on Postman + Newman and driven by three custom Claude Code agents. Clone it as the starting point for a new QA project, run the setup command, and you have a working test suite. The agents adapt to whatever OpenAPI spec, base URL, and credentials you give them — nothing is hardcoded to a specific project.

## What this is

A self-contained QA workflow that:

1. Reads an OpenAPI spec and builds a full Postman collection with happy + negative tests
2. Creates per-ticket Postman collections scoped to individual Jira tickets
3. Detects when developers ship new or changed endpoints and auto-updates the collections

All three pieces are slash commands that live in `.claude/commands/` and are invoked from inside Claude Code.

## Prerequisites

Before running any of the agents, make sure your machine has:

1. **Claude Code** installed and signed in.
2. **Newman** installed globally (`npm install -g newman newman-reporter-htmlextra`) — used by `run-tests.sh` to execute the collections.
3. **`python3`** available on the PATH — used by `generate-issues.py` to produce the human-readable failure report.
4. The following **Claude Code plugins enabled in your global `~/.claude/settings.json`**:
   - `postman@claude-plugins-official` — **required**. Used by all four agents to talk to your Postman workspace.
   - `atlassian@claude-plugins-official` — **recommended**. Lets `/qa-test-ticket` pull Jira tickets directly. If skipped, you'll need to paste ticket content manually.
5. A **Postman API key** (PMAK-…), either in `~/.claude/settings.json` (global) or `<project>/.claude/settings.local.json` (per-project). See the [`.env` keys section](#env-keys) for both options.

If any plugin is missing when you run `/qa-api-test-setup`, Phase 0 will stop and tell you exactly which one to enable and how. You won't get silent failures partway through.

## The agents

| Command | Purpose | When to run |
|---|---|---|
| `/qa-api-test-setup` | First-time project setup. Builds the main collection from scratch. | Once, when starting on a new API project. |
| `/qa-test-ticket <JIRA-KEY>` | Builds a per-ticket test collection (e.g. PROJ-123). | Every time a new Jira ticket needs scoped tests. |
| `/qa-api-sync` | Detects spec drift, updates collections, refreshes snapshot. | Daily (via `/schedule`), before every ticket build (auto), or manually. |
| `/qa-negative-audit` | Walks every endpoint, evaluates the negative matrix, reports gaps, optionally fills them in module-wise batches with a resume checkpoint. | On-demand: after big sync changes, periodically to catch hand-edit drift, or anytime you want a coverage health check. |

---

## Full workflow

### Day 1: First-time setup

Before running anything, make sure your Postman API key is reachable. **Two options — pick one** (see the [`.env` keys section](#env-keys) for full detail):

- **Project-local (recommended for cloners):** `cp .claude/settings.local.json.example .claude/settings.local.json` and paste your `PMAK-...` key into it. Gitignored.
- **Global:** add `"env": { "POSTMAN_API_KEY": "PMAK-..." }` to `~/.claude/settings.json`. One-time per machine.

If both are set, the local file wins.

Inside the project directory, run:
```
/qa-api-test-setup
```

It will:
1. Verify the Postman, Atlassian, and Newman plugins are connected.
2. Read your `.env` (or create one if missing).
3. Ask for missing pieces: project name, OpenAPI spec source, auth type, login credentials, Postman workspace.
4. Read the OpenAPI spec — accepts raw OpenAPI JSON/YAML URLs **or Scalar / Swagger UI / Redoc reference pages** (auto-discovers the underlying spec). After loading, prints a mandatory summary block: source, OpenAPI version, **total endpoint count**, per-tag module counts, auth schemes.
5. **Run a dependency audit** to figure out the right execution flow — auth endpoints (register, login) run first, endpoints that need a `{{testOrderId}}` run after the `POST /orders` that produces it, etc. Spec-tag order is ignored. Endpoints whose dependencies can't be resolved are flagged as orphans and placed at the end with `{{test<Entity>Id}}` placeholders.
6. Plan the collection structure with the audited flow order baked in, ask you to confirm (`y` / `n` / `edit`).
7. Build the main Postman collection: a top-level `Happy Path - All Endpoints` smoke folder in dependency order, plus one folder per module (`NN.` index reflects flow order, not spec-tag order) with `Positive` + `Negative` sub-folders.
8. Export the collection JSON locally.
9. **Create a shared Postman Environment** in your workspace, named `<ProjectName> Environment`, with all the universal vars (`baseUrl`, `accessToken`, `refreshToken`, etc.) AND auto-detected chained-ID vars (`testUserId`, `testOrderId`, or whatever your spec defines). This is what lets you select the env in Postman GUI and run any request manually — tokens auto-populate when Login runs.
10. Create `newman-env.json` (local mirror of the cloud env for Newman) and `run-tests.sh`.
11. Write the initial `api-snapshot-YYYY-MM-DD.json` baseline.

**About the negative coverage approach.** The setup applies a **24-row negative test matrix** (`.claude/commands/qa-negative-matrix.md`) to every endpoint — not just the basic "no-auth + missing-field + 404" trio. The matrix covers `Auth Failures` (no/expired/tampered/malformed token, wrong-role), `Validation Failures` (missing required, wrong type, out-of-range, regex mismatch, enum out-of-range), `Resource Errors` (not-found, wrong-state, conflict), and `Security Probes` (SQL injection, XSS reflection, path traversal, CORS, optional rate-limit). Plus cross-cutting assertions (response time, schema validation, sensitive-data leak, error body structure, no stack trace, idempotency) added INTO every test. The agent walks every matrix row per endpoint and either generates the test or marks it `n/a (condition not met)` — nothing is mentally skipped. Rate-limit tests are destructive and opt-in via `INCLUDE_RATE_LIMIT_TESTS=true` in `.env` (default `false`, asked once during setup). For big projects, builds happen module-wise with a continue prompt every module if `>20` endpoints total.

### Day 2 onwards: Spec stays in sync automatically

Schedule the daily sync once (see the Scheduling section below):
```
/schedule daily at 2:30 PM BDT /qa-api-sync
```

From this point on, the agent runs every day at 2:30 PM Bangladesh time and keeps your main collection up to date with whatever the dev team ships. Like the setup command, it auto-discovers the spec from Scalar/Swagger UI/Redoc pages if needed and prints the mandatory endpoint-count block (including delta vs the snapshot) before applying any changes.

### When a new Jira ticket lands

Run:
```
/qa-test-ticket PROJ-123
```

It will:
1. Run `/qa-api-sync` first to make sure the spec is current.
2. Pull the ticket from Jira (or accept pasted content if Atlassian plugin is not connected).
3. Hard-stop if any credentials it needs are not in `.env`, and ask you to provide them.
4. **Run a dependency audit** scoped to just the endpoints this ticket touches, ordering them into a valid run flow (Setup-created entities count as producers).
5. Plan a Setup / Happy Path - All Endpoints / Positive / Negative folder structure with the audited flow baked into Happy Path, ask you to confirm.
6. Build a standalone collection named `PROJ-123: <feature name>` in the same workspace.
7. Run Newman, produce an HTML report.

---

## Command details

### `/qa-api-test-setup`

Think of this as the "first-day setup" agent. When you start QA work on a brand new API project, this is the only thing you need to run to bootstrap everything from zero. It handles every annoying piece of test-suite setup that nobody wants to do by hand: it reads the OpenAPI spec, asks you a handful of short questions (only the ones it cannot figure out on its own), and then goes off and builds a fully functioning Postman collection with login, happy-path tests, and negative tests for every endpoint. It also writes you a `.env` file, a Newman run script, and a baseline snapshot file that the sync agent uses later to detect drift. After it finishes, you have a runnable test suite that produces HTML reports - no manual Postman clicking required.

**What you can do with it:**
- Bootstrap a fresh API project end-to-end with one command.
- Auto-generate happy + negative tests for every endpoint in your OpenAPI spec.
- Let it figure out auth requirements, request bodies, response schemas, enum values - all from the spec.
- Walk away with a runnable Newman suite that produces a styled HTML report.
- Resume an interrupted setup - if `.env` already exists, it reuses what is there and only asks for what is missing.
- Safely re-run it without fear of overwriting an existing project (the preflight guard stops you unless you type `force re-setup`).

First-time project setup only. Do not re-run on an already-set-up project.

Built-in guard: before doing anything, it scans the project directory for setup artifacts (`.env`, main collection JSON, `api-snapshot-*.json`, `newman-env.json`, `run-tests.sh`).

- If 3 or more of these exist -> the agent stops and tells you to use `/qa-api-sync`, `/qa-test-ticket`, or `./run-tests.sh` instead. To override (rarely needed), type exactly `force re-setup` to proceed and overwrite everything.
- If only `.env` exists (or 1-2 partial artifacts) -> it proceeds normally, reuses the existing `.env`, and only asks for what is missing. Useful when an earlier setup got interrupted halfway.
- If nothing exists -> fresh setup, asks every question.

Inputs it asks for (only if not already in `.env`):
- Project name
- OpenAPI spec source (URL or local file)
- API base URL
- Auth type (JWT / API key / both)
- Login email + password
- Postman workspace (picked from a live list)
- Confluence space key (optional)

Outputs:
- Main Postman collection in your chosen workspace
- `<project>-collection.json` (local export)
- `newman-env.json` (Newman environment file)
- `run-tests.sh` (run script, chmod +x)
- `generate-issues.py` (Newman-result parser)
- `.env` (created if missing)
- `api-snapshot-<today>.json` (baseline for future drift detection)

### Main-collection folder structure

The main collection has a top-level **`Happy Path - All Endpoints`** folder (a quick smoke pass across every spec endpoint with valid data) followed by **module-grouped + Positive/Negative** folders. Module names come from the OpenAPI spec's `tags` field - whatever the dev team tagged endpoints with becomes a top-level folder.

```
<ProjectName> API — Automated Tests
├── Happy Path - All Endpoints     # runs first; one valid-data request per endpoint, audited flow order
│   ├── POST /auth/login
│   ├── GET  /orders
│   ├── POST /orders
│   └── ...                          # named <METHOD> <path>, status-only assertions
├── 00. <Module1>           # NN. index = audited dependency-flow order, not spec-tag order
│   ├── Positive
│   │   ├── TC01: <happy path> (200)
│   │   └── TC02: <another happy path> (201)
│   └── Negative                                    # 4-way sub-folder split, continuous TC numbering across all four
│       ├── Auth Failures                           # matrix rows: no-auth, expired/tampered/malformed token, wrong-role
│       │   ├── TC03: <no auth> (401)
│       │   ├── TC04: <expired token> (401)
│       │   ├── TC05: <tampered token> (401)
│       │   ├── TC06: <malformed token> (401)
│       │   └── TC07: <wrong role> (403)
│       ├── Validation Failures                     # missing required, wrong type, out-of-range, regex, enum
│       │   ├── TC08: <missing required field> (400)
│       │   ├── TC09: <wrong type> (400)
│       │   └── TC10: <out of range> (400)
│       ├── Resource Errors                         # not-found, wrong-state, conflict
│       │   ├── TC11: <non-existent ID> (404)
│       │   └── TC12: <duplicate> (409)
│       └── Security Probes                         # SQL injection, XSS, path traversal, CORS, opt-in rate-limit
│           ├── TC13: <SQL injection probe> (rejected)
│           ├── TC14: <XSS probe> (not reflected)
│           └── TC15: <path traversal> (rejected)
├── 01. <Module2>
│   ├── Positive
│   │   └── TC01: <happy path> (200)
│   └── Negative
│       ├── Auth Failures
│       │   └── TC02: <no auth> (401)
│       ├── Validation Failures
│       ├── Resource Errors
│       └── Security Probes
└── 02. <Module3>
    ├── Positive
    │   └── ...
    └── Negative
        ├── Auth Failures
        ├── Validation Failures
        ├── Resource Errors
        └── Security Probes
```

> **Note:** every module's `Negative` folder ALWAYS has all 4 sub-folders (`Auth Failures`, `Validation Failures`, `Resource Errors`, `Security Probes`), even if some are empty for that module. Keeps the layout visually consistent so users always know where to look. The sub-folder name describes the **category of failure** the tests inside cover, applied to the **parent module's** endpoints — so `Orders > Negative > Auth Failures` holds the auth-failure tests *for Orders endpoints* (not tests of the Auth module).
> **TC numbering** is continuous across all 4 Negative sub-folders within a module — a single counter from where Positive ended.
> **Cross-cutting assertions** (response time SLA, schema validation, sensitive-data leak, error body structure, no-stack-trace) are added INTO existing test scripts as additional `pm.test(...)` blocks, not as separate test cases.

Key properties:
- **`Happy Path - All Endpoints`** is a fast smoke pass. One valid-data request per `(method, path)`, named `<METHOD> <path>` exactly. Status-only assertion (`2xx`). Requests are ordered by the **dependency audit** (`POST /auth/login` runs before `GET /profile`, `POST /orders` runs before `GET /orders/{id}`, etc.), not by spec order. This folder is in addition to — not replacing — the module folders.
- Module names are NOT hardcoded - they come from your OpenAPI spec's `tags`. For an e-commerce API you might see `Products`, `Orders`, `Customers`. For a CMS API: `Content`, `Users`, `Media`. The agent reads the spec and uses what's there.
- **Module `NN.` index reflects audited flow order, not spec order.** Auth/signup modules float to `00.`/`01.`, modules that only depend on others get later indices. `/qa-api-sync` does not renumber existing modules — new tags added later get appended with the next available index.
- If the spec has no tags, the agent asks you how to group endpoints.
- **TC numbering resets per module.** Each module starts at TC01. Numbers are continuous across Positive → Negative within a module (e.g. Positive ends at TC04, Negative starts at TC05). Happy Path requests are not TC-numbered.
- **Every endpoint gets at least one negative test** (no-auth → 401) in its module's Negative sub-folder. Additional negatives (400/404/409/422) added where the spec or context warrants.
- **Both sub-folders always exist** per module, even if one starts with a single test - keeps the structure visually consistent across modules.

Compare with ticket collections (`/qa-test-ticket`), which use a flat **Setup / Happy Path / Positive / Negative** layout scoped to one ticket's endpoints.

### `/qa-test-ticket <JIRA-KEY>`

This is the per-ticket testing agent. Every time the Jira board lights up with a new feature, bug, or story that touches the API, you run this once with the ticket key and it builds you a dedicated test collection just for that ticket. It reads the ticket directly from Jira (or accepts pasted content if the Atlassian plugin is not connected), figures out which endpoints are in scope, plans the Setup/Positive/Negative scenarios, hard-stops to ask you for any credentials it needs but does not have, then constructs a standalone collection inside Postman that maps 1:1 to the ticket's acceptance criteria. Each ticket gets its own collection named like `PROJ-123: <Feature Name>`, so your main test suite stays clean and your per-ticket work stays organized.

**What you can do with it:**
- Test a single Jira ticket end-to-end without polluting the main collection.
- Auto-generate happy-path tests for every acceptance criterion in the ticket.
- Get the full auth-matrix (no token / expired / tampered / malformed) negative tests for every endpoint the ticket touches.
- Pull the ticket directly from Jira via the Atlassian plugin, or paste the ticket content manually if the plugin is not authorized.
- Have the agent block-and-ask when the ticket needs credentials/IDs/data you do not have - no silent failures, no half-built collections.
- Re-test the same ticket later - its collection stays in Postman until you delete it, and gets auto-refreshed by `/qa-api-sync` if the endpoints it uses change.

**What's a Jira ticket key?**

A Jira ticket key is the short identifier Jira assigns to every issue. It is shaped like `PROJECT-NUMBER`:
- `PROJ-123` - project prefix `PROJ`, ticket number `123`
- `ACME-42` - project prefix `ACME`, ticket number `42`
- `WEB_UI-7` - project prefix `WEB_UI`, ticket number `7`

You can find it in two places:
- The **URL bar** of any Jira ticket: `https://yourcompany.atlassian.net/browse/PROJ-123` - the part after `/browse/` is the key.
- The **top-left of the ticket page** in Jira, displayed next to the ticket title.

**Both forms work as input:**
- `/qa-test-ticket PROJ-123` (just the key)
- `/qa-test-ticket https://yourcompany.atlassian.net/browse/PROJ-123` (full URL - the agent extracts the key automatically)
- `/qa-test-ticket https://yourcompany.atlassian.net/browse/PROJ-123?focusedCommentId=9999` (URL with extra query params is fine too)

The agent confirms the parsed key back to you before doing anything else (`Working on ticket: PROJ-123 (parsed from ...)`).

Per-ticket collection. Invocation example: `/qa-test-ticket PROJ-123`.

### Ticket-collection folder structure

Every ticket collection is a **standalone, flat four-folder collection** - no module sub-grouping, because one ticket is scoped to one feature. Sub-grouping by endpoint would only add noise.

```
<KEY>: <feature name>                # e.g. PROJ-123: Order Submission Flow
├── Setup
│   ├── 1. Login
│   ├── 2. Fetch <Entity> ID
│   ├── 3. Create <Entity>
│   └── ...                           # scaled to whatever the ticket needs
├── Happy Path - All Endpoints       # scoped to the endpoints this ticket touches, audited order
│   ├── POST /api/v1/orders
│   ├── GET  /api/v1/orders/{id}
│   └── ...                           # named <METHOD> <path>, status-only assertions
├── Positive
│   ├── TC01: <happy-path AC scenario> (200/201)
│   ├── TC02: <next AC scenario> (200)
│   ├── TC02b: <verification step tied to TC02> (200)
│   └── ...
└── Negative                                       # flat folder; full matrix applied, continuous TC
    ├── TC<N>:   <no auth> (401)                   # Auth rows from the matrix
    ├── TC<N+1>: <expired token> (401)
    ├── TC<N+2>: <tampered token> (401)
    ├── TC<N+3>: <malformed token> (401)
    ├── TC<N+4>: <wrong role> (403)
    ├── TC<N+5>: <missing required field> (400)    # Validation rows
    ├── TC<N+6>: <wrong type> (400)
    ├── TC<N+7>: <out of range> (400)
    ├── TC<N+8>: <non-existent ID> (404)            # Resource rows
    ├── TC<N+9>: <wrong state transition> (422)
    ├── TC<N+10>: <duplicate> (409)
    ├── TC<N+11>: <SQL injection probe> (rejected)  # Security rows
    ├── TC<N+12>: <XSS probe> (not reflected)
    └── TC<N+13>: <path traversal> (rejected)
```

Key properties:
- **Standalone collection, not a fork** of the main one. Lives independently in Postman.
- **Named `<KEY>: <feature name>`** (e.g. `PROJ-123: Order Submission Flow`).
- **Flat four folders only, in run order**: `Setup`, `Happy Path - All Endpoints`, `Positive`, `Negative`. No module sub-folders, and no sub-folders within `Negative` (unlike the main collection's 4-way split — ticket scope is small enough that flat is more navigable).
- **Happy Path - All Endpoints** is scoped to only the endpoints this ticket touches — one valid-data request per `(method, path)`, named `<METHOD> <path>` exactly, status-only assertions. Variable chaining (`pm.environment.set`) stays in Setup; Happy Path doesn't set vars.
- **Setup is sized to the ticket** - a simple ticket may have 1-3 setup items; a complex feature may need 20+ (login → fetch existing IDs → create prerequisite entities → build state).
- **Continuous TC numbering** across `Positive` then `Negative` (no reset). E.g. Positive ends at TC04, Negative starts at TC05. Happy Path requests are not TC-numbered. Within `Negative`, the matrix-driven order is Auth Failures first, then Validation Failures, Resource Errors, Security Probes — all in one continuous TC sequence.
- **Full negative matrix** applied to every endpoint in scope (auth-matrix, validation, resource, security rows — see `.claude/commands/qa-negative-matrix.md`). Rows whose conditions don't apply are marked `n/a` and skipped. Rate-limit probe is opt-in via `INCLUDE_RATE_LIMIT_TESTS=true` in `.env`.
- **Verification suffixes** (`TC02b`, `TC02c`) for follow-up checks tied to a primary case.

Compare with the **main collection** (`/qa-api-test-setup`), which uses a `Happy Path - All Endpoints + Module → Positive/Negative` structure because it covers every endpoint across every module of the API.

Credential gate: if the ticket needs any creds, IDs, or test data not in `.env`, the agent stops and asks you to provide them. It suggests asking the dev team, checking API docs, or checking the linked Confluence page.

### `/qa-api-sync`

This is the "spec watcher." Your API is a moving target - developers ship new endpoints, change required fields, deprecate old routes - and without this agent your tests would silently rot until something breaks in production. This agent compares the current OpenAPI spec against the last snapshot it took, figures out exactly what changed, and updates your Postman collections to match. New endpoints get auto-generated test sets. Changed endpoints get their request bodies and params refreshed while your hand-written test assertions are preserved untouched. Removed endpoints get archived (renamed to `[DEPRECATED]`) instead of being deleted, so you keep the history. And any ticket-specific collections that reference changed endpoints get refreshed too, so per-ticket collections stay aligned with the latest API even when you have not touched that ticket in weeks.

**What you can do with it:**
- Detect every new endpoint the dev team shipped since the last sync.
- See exactly what changed in each modified endpoint, with a before/after diff (e.g. "new required field: regionId").
- Auto-update request bodies and params without ever touching your custom test scripts.
- Archive deprecated endpoints as `[DEPRECATED] <name>` so the history stays visible in Postman.
- Refresh every affected ticket collection in the same pass (no manual hunting through `<KEY>-XXX` collections).
- Schedule it to run daily/hourly/whatever cadence you want so you never wake up to stale tests (see Scheduling section below).
- Run it on demand before any ticket work, or rely on `/qa-test-ticket` to auto-run it as its first step.

Reconciliation between the OpenAPI spec and your Postman collections.

What it detects:
- Added endpoints (in spec but not in collection)
- Changed endpoints (request body, params, response, or auth requirement changed)
- Removed endpoints (in collection but no longer in spec)

What it does:
- Adds new endpoints to the main collection with the standard test set (happy + 401 + 400, plus 404 / 422 / 409 where applicable)
- Updates request body/params for changed endpoints, preserving the hand-written test scripts
- Renames removed endpoints to `[DEPRECATED] <name>` instead of deleting them
- Refreshes any ticket collections (`<KEY>-XXX`) that use the changed/removed endpoints
- Renames the snapshot file to today's date and adds a new entry at the top of the `history` array

Invocation modes:
- Manual: `/qa-api-sync`
- Scheduled (runs even when Claude Code is closed): `/schedule daily at 2:30 PM BDT /qa-api-sync`
- Auto-called as Phase 0 of `/qa-test-ticket <KEY>`

### `/qa-negative-audit`

The on-demand **coverage health check**. Sync watches for spec drift; audit watches for *coverage* drift — the gap between what you have and what the negative matrix says you should have. Run when:
- A sync brought in many new endpoints and you want to confirm their matrices were fully generated.
- Teammates have been hand-editing collections and you suspect tests got removed or skipped.
- You want a periodic (weekly / monthly) coverage report.
- You're seeing the suggestion `Run /qa-negative-audit to close coverage gaps?` at the end of a sync.

What it does:
- Walks every endpoint in the spec.
- Evaluates every row of the negative matrix (`.claude/commands/qa-negative-matrix.md`) against the endpoint's spec shape.
- Cross-references against the actual tests in the collection.
- Prints a coverage report per endpoint and per module: `✓ covered (TC<NN>)`, `✗ missing`, `n/a (condition not met)`, `skipped (env flag)`.
- Offers to fill the gaps in **module-wise batches**. Each module is a checkpoint — interrupt with Ctrl+C and the next `/qa-negative-audit` resumes from the checkpoint file (`.audit-progress.json`).

Flags:
- `(no flag)` — default audit + fill flow
- `--report-only` — just print the gap report, change nothing
- `set` — fill one module's gaps, then stop and ask
- `--verbose` — show every row including covered + n/a
- `--reorganize` — one-shot pass that moves loose tests at the root of `Negative` into the 4-way sub-folders (only needed for older collections built before the matrix existed)
- `--include-rate-limit` — override `INCLUDE_RATE_LIMIT_TESTS=false` for one run (use with care — destructive)

The audit **never modifies or deletes existing tests**. It only adds missing ones. Renumbering or removing requires explicit human action. This makes it safe to run on any collection — worst case, it does nothing because everything is already covered.

---

## Scheduling deep-dive

The `/schedule` skill is a built-in Claude Code feature that runs slash commands on a cron-style timer, even when Claude Code is not open on your machine.

### One-time setup (do this once, manually by yourself)

The first time, you have to register the schedule yourself. Open any Claude Code session inside this project and type this exactly:

```
/schedule daily at 2:30 PM BDT /qa-api-sync
```

Hit enter. The `/schedule` skill will confirm the schedule was created and give you back a schedule ID. That is it - you are done. No further action from you.

### What happens from the next day onwards (automatic)

Once that one-time command is registered, the system takes over:

- Every day at 2:30 PM Bangladesh time, `/qa-api-sync` fires by itself.
- It runs even when Claude Code is closed, your laptop is shut, or you are on leave.
- It pulls the latest OpenAPI spec, diffs it against the snapshot, updates the main collection, refreshes any affected ticket collections, archives removed endpoints, and rewrites the snapshot file with today's date.
- When you next open Claude Code, you can see what the run did via `/schedule list` (it shows the latest run output for each scheduled agent).

You only need to repeat the setup command if you want to change the time, change the cadence, or delete the schedule - see the sections below for that.

### `/schedule` examples (pick the cadence that fits)

1. **Once a day at 2:30 PM BDT** (the standard, recommended for most teams):
   ```
   /schedule daily at 2:30 PM BDT /qa-api-sync
   ```

2. **Twice a day - morning standup + end of dev day** (good when devs ship multiple times daily):
   ```
   /schedule daily at 9 AM and 5 PM BDT /qa-api-sync
   ```

3. **Weekdays only, skip weekends** (no point running on Sat/Sun if no one is shipping):
   ```
   /schedule weekdays at 10 AM BDT /qa-api-sync
   ```

4. **Hourly during work hours** (intense sprint, lots of API churn):
   ```
   /schedule hourly 9 AM to 6 PM BDT on weekdays /qa-api-sync
   ```

5. **Weekly Monday morning** (slow-moving APIs, light cadence):
   ```
   /schedule every Monday at 9 AM BDT /qa-api-sync
   ```

You can also combine `/schedule` with other commands - for example, auto-running tests after every sync:
```
/schedule daily at 2:30 PM BDT ./run-tests.sh
```

### Manual run (any time)

If you want to force a sync right now without waiting for the schedule:
```
/qa-api-sync
```

Useful when you know the dev team just shipped something and you want fresh tests immediately.

### How to check your active schedules

In any Claude Code session:
```
/schedule list
```

You will see all your scheduled agents, their IDs, their cron, and recent run history. Look for the entry pointing at `/qa-api-sync` to confirm it is active.

### How to check what the daily sync did

Each scheduled run leaves output in the `/schedule` history. To see what happened on the latest run:
```
/schedule list
```
then pick the entry. You will see the summary the agent printed (added / changed / removed counts, ticket collections touched, snapshot filename).

You can also confirm it ran by looking at the snapshot file in the project root - the date in the filename should be today's date, and the top entry of the `history` array should match the latest sync.

### How to change the time

```
/schedule update <id> daily at 9:00 AM BDT /qa-api-sync
```

Replace `<id>` with the schedule ID shown by `/schedule list`. The new time applies starting from the next scheduled run.

### How to delete the schedule

```
/schedule delete <id>
```

After this, the daily auto-sync stops. You can still run `/qa-api-sync` manually any time, and `/qa-test-ticket` will still auto-call it.

### Where to find the `/schedule` command

It is a built-in Claude Code skill, already available in your environment. You do not install it, you just type it in any Claude Code chat input. If you ever forget the syntax, type `/schedule` alone and it will guide you.

### When to use `/schedule` vs `/loop`

| Tool | Runs when CC is closed? | Use for |
|---|---|---|
| `/schedule` | Yes | Long-term recurring jobs (daily syncs, hourly checks). |
| `/loop` | No, only inside an open session | Short-term polling (watching a deploy, retrying a build). |

For the daily sync, always use `/schedule`. `/loop` would only work while you sit with Claude Code open all day.

---

## Project files

The structure below shows the canonical layout after running `/qa-api-test-setup` once. Files with `<placeholders>` are generated based on what you tell the agents during setup - they vary per project.

```
your-project/
├── .env                                   # YOUR project credentials & config
├── .env.example                           # Template you copy to .env
├── .gitignore                             # Tells git which generated files to skip
├── README.md                              # User-facing docs
├── CLAUDE.md                              # Auto-loaded AI context
├── .claude/
│   ├── commands/                          # The three agent definitions
│   │   ├── qa-api-test-setup.md
│   │   ├── qa-test-ticket.md
│   │   └── qa-api-sync.md
│   └── templates/                         # Template files the agent writes from
│       ├── run-tests.sh                   # template with __PROJECT_NAME__/__BASE_URL__ placeholders
│       └── generate-issues.py             # generic - takes project info as CLI args
├── run-tests.sh                           # GENERATED. One-shot script: pings the server, runs the full Newman suite, generates the HTML report, runs generate-issues.py, opens the report.
├── generate-issues.py                     # GENERATED. Parses Newman JSON results into a human-readable issues report (status mismatches + message quality).
├── newman-env.json                        # GENERATED. Newman environment variables
├── <openapi-spec>.json                    # OpenAPI spec (only if stored locally; can be a URL in .env instead)
├── api-snapshot-YYYY-MM-DD.json           # GENERATED. Drift baseline, updated by /qa-api-sync
├── <project-slug>-collection.json         # GENERATED. Main Postman collection export
├── <jira-key>-collection.json             # GENERATED. One per Jira ticket built
├── newman-reports/                        # GENERATED. HTML test reports + Newman JSON (one set per run)
└── collection-run-issues/                 # GENERATED. Parsed issues reports (one .txt per run)
```

Naming conventions:
- `<project-slug>` is a lowercase, hyphen-free version of your project name (chosen during `/qa-api-test-setup`, e.g. `myproject`).
- `<jira-key>` is the lowercased ticket key (e.g. `PROJ-123` becomes `proj123`).
- `<openapi-spec>` is whatever you named your spec file - the agents do not care about the filename, only the path/URL stored in `.env`.

## How the template flow works

`.claude/templates/` holds blueprint files that the agent reads ONCE during setup, fills in with your project-specific values, and writes to the project root as live, runnable scripts.

```
.claude/templates/run-tests.sh           .claude/templates/generate-issues.py
  (has placeholders                        (generic, takes CLI args)
   like __PROJECT_NAME__)                    
        |                                          |
        v                                          v
[/qa-api-test-setup Phase 7 reads the file,
 substitutes placeholders from .env, writes to root]
        |                                          |
        v                                          v
./run-tests.sh                            ./generate-issues.py
  (generated, project-specific,             (generated verbatim)
   chmod +x)
```

Important properties:

- **Templates are read ONCE, during setup only.** After Phase 7 finishes, the agent never opens them again. The generated files in root are what `./run-tests.sh` and Newman actually use at runtime.
- **Newman never touches the templates folder.** It only reads `<project>-collection.json` and runs the generated `run-tests.sh` from root.
- **If you ever edit a template, the change does NOT propagate** to an already-set-up project. You would need to re-run setup (with `force re-setup`) or manually copy the new template content into root.
- **If you delete root's `run-tests.sh` or `generate-issues.py` by accident**, just re-run `/qa-api-test-setup` (it will detect partial setup and re-generate them).
- **Why the subfolder?** To keep filenames clean - both the template and the generated file are named `run-tests.sh`, so they need to live in different folders. The template is the blueprint, the root file is the active script.

## What gets generated by `./run-tests.sh`

Three outputs per run, all written under `newman-reports/` and `collection-run-issues/`:

1. **HTML report** (`newman-reports/<project>-report-<timestamp>.html`) - styled report with every request, response, and assertion. Auto-opens in your browser. Best for visual inspection.
2. **JSON results** (`newman-reports/<project>-results-<timestamp>.json`) - raw Newman output. Used as input by `generate-issues.py`. You usually do not read this directly.
3. **Issues report** (`collection-run-issues/<project>-issues-<timestamp>.txt`) - plain text human-readable report classifying failures into two categories:
   - **Status mismatch** - wrong HTTP status code with contextual reasoning (e.g. "Expected 401 but got 200. The API accepted a potentially dangerous payload (XSS or SQL injection) without sanitising or rejecting it. This is a security vulnerability.")
   - **Message quality** - vague error messages with guidance on what good ones look like (e.g. names the affected field, explains the reason, tells the user what input is accepted)

The issues report is what to read when investigating bug candidates - it does the categorization work for you before you reach for the HTML report.

## .env keys

Populated by `/qa-api-test-setup`, used by all three agents at runtime.

### Required keys

| Key | Purpose |
|---|---|
| `BASE_URL` | API server root (e.g. `https://api.example.com`) |
| `OPENAPI_SPEC` | URL or path to the OpenAPI doc |
| `HEALTH_PATH` | Path of a reachable endpoint used by `run-tests.sh` to ping the server before running the suite |
| `LOGIN_EMAIL` | Test account email (used at runtime by the Login request) |
| `LOGIN_PASSWORD` | Test account password (used at runtime by the Login request) |
| `PLATFORM_API_KEY` | Platform-level API key (only if any endpoints use apiKey auth) |
| `JIRA_PROJECT_KEY` | Jira project key (e.g. `PROJ`). Auto-extracted from the first `/qa-test-ticket <KEY>` call. |

### What `HEALTH_PATH` is for

Before Newman runs the suite, `run-tests.sh` pings `$BASE_URL$HEALTH_PATH` with a plain `GET`. The check decides whether to even start the suite:

- **2xx / 3xx / 4xx response** → server is alive → run the suite
- **5xx or connection refused** → server is down or misconfigured → abort with a clear message instead of pretending 600 tests "failed"

The goal is fail-fast on a dead environment, not a deep functional check. The endpoint only needs to **respond** to a GET — it doesn't need to return 200.

**Picking a value (in order of preference):**

1. **A dedicated health endpoint** if one exists in your API: `/health`, `/api/v1/health`, `/healthz`, `/status`, `/ping`. Always returns 200. Cleanest.
2. **Any cheap public GET** from the spec — list of regions, currencies, categories, etc. Returns 200 without auth.
3. **The login endpoint** (`LOGIN_PATH`). A GET on a POST-only path usually returns 405 Method Not Allowed, which counts as "alive". Acceptable fallback when 1 and 2 don't exist.

**How to check if your API has option 1:**

```bash
source .env
# search the spec for any health-style path
curl -s "$OPENAPI_SPEC" | grep -iE '"(/[^"]*(health|status|ping|live|ready)[^"]*)"'

# or probe the common conventions directly
for p in /health /api/health /api/v1/health /healthz /livez /readyz /status /ping; do
  code=$(curl -s -o /dev/null -w "%{http_code}" "$BASE_URL$p")
  echo "$code  $p"
done
```

Anything returning `200` is a good `HEALTH_PATH`.

**What to avoid:**

- Endpoints that need a request body (POST/PUT) — `run-tests.sh` sends GET only
- Endpoints that hit third-party services or do heavy DB queries — slow, can flap
- Endpoints whose path includes user-specific IDs — won't work for an anonymous ping

### Auto-detected from the OpenAPI spec (NOT required in .env)

The agent reads these from the spec during setup. They live inside the generated Postman collection after build, not in `.env`. Only put them in `.env` if you need to override the agent's detection.

| Key | What gets auto-detected |
|---|---|
| `AUTH_TYPE` | From `components.securitySchemes` (jwt / apikey / both) |
| `LOGIN_PATH` | From tagged auth operations + path heuristics (`*/login`, `*/auth/login`, `*/token`) |
| `TOKEN_JSON_PATH` | By walking the login endpoint's response schema for `accessToken`/`access_token`/`token` properties, then verifying with one live login call |

If the detection is wrong, uncomment and edit these in `.env` to force-override:
```
AUTH_TYPE=jwt
LOGIN_PATH=/api/v1/auth/login
TOKEN_JSON_PATH=payload.data.tokens.accessToken
```

The Postman API key (`PMAK-...`) does NOT live in `.env`. It lives under `env.POSTMAN_API_KEY` in a Claude Code settings file. **Two valid locations — pick one:**

**Option A — Project-local (recommended for cloners)**

`<project-root>/.claude/settings.local.json` — gitignored, per-project. Best if you work with multiple Postman accounts/workspaces, or just want everything for this project in one place. The repo ships a template with both your PMAK slot AND the pre-approved permission list (so you don't get spammed with permission prompts on the first run):

```bash
cp .claude/settings.local.json.example .claude/settings.local.json
# then edit .claude/settings.local.json and paste your PMAK
```

The pre-approved `permissions.allow` block whitelists every Postman MCP tool (`mcp__plugin_postman_postman__*`), `curl`, `python3`, `node`, `newman run`, and the project's `run-tests.sh`. If you'd rather grant each one interactively, delete the `permissions` block from your copy.

If `.claude/settings.local.json` already exists in your clone (e.g. you've added your own custom permissions), merge an `env` key into it rather than overwriting:
```json
{
  "permissions": { ... existing ... },
  "env": { "POSTMAN_API_KEY": "PMAK-your-key-here" }
}
```

**Option B — Global (one-time-per-machine)**

`~/.claude/settings.json` — your home-directory Claude Code config, shared across every project on your machine. Best if you use one Postman account for all QA work.

```json
{
  "env": { "POSTMAN_API_KEY": "PMAK-your-key-here" }
}
```

**Both at once?** Claude Code's settings cascade merges them automatically. **The local file wins if both define `POSTMAN_API_KEY`** — useful when you want a project-specific override of a global default.

To make the file-location distinction concrete:
- **Global config**: `~/.claude/settings.json` (i.e. `/Users/<your-username>/.claude/settings.json` on macOS, `/home/<your-username>/.claude/settings.json` on Linux, `C:\Users\<your-username>\.claude\settings.json` on Windows). Lives in your home directory, shared across every project on your machine.
- **Project-local config**: `<project-root>/.claude/settings.local.json` (e.g. `/Users/<your-username>/Documents/your-project/.claude/settings.local.json`). Lives inside this repo's `.claude/` folder, gitignored. The `.claude/commands/` and `.claude/templates/` folders next to it are the committed agent + template definitions shared with your team.

Same folder name (`.claude/`), two completely different scopes.

## Running tests

After collection is built, run:
```
./run-tests.sh
```

The script:
1. Pings the API to confirm the server is reachable.
2. Runs Newman with the main collection + environment.
3. Writes an HTML report to `newman-reports/<project>-report-<timestamp>.html`.
4. Opens the report in the browser.

For ticket runs, `/qa-test-ticket` handles the Newman invocation itself and outputs to `newman-reports/<KEY>-report-<timestamp>.html`.

## When tests fail (auto-fix first, then debug, then log to Jira)

The agents do NOT immediately hand failures back to you. They go through three stages:

### Stage 1 - Auto-fix (the agent attempts to fix what it can)

After the suite runs, if there are any failures, `/qa-test-ticket` Phase 6 and `/qa-api-test-setup` Phase 8a kick in. The agent classifies each failure and tries to fix the fixable categories automatically:

- **Spec drift** -> calls `/qa-api-sync`, refreshes request bodies/params, re-runs affected tests.
- **Stale `accessToken`** -> re-runs Login, re-runs downstream failed tests.
- **Stale chained IDs** -> re-runs Setup folder, re-runs dependents.
- **Setup chain timing race** -> adds delay, re-runs.
- **409 conflict on a unique field** -> patches the body to use `{{$timestamp}}`, re-runs.
- **`SyntaxError: Identifier already declared`** -> patches `const` to `let` in the script, re-runs.
- **Transient 5xx / HTML error page** -> retries once after a short wait.

Cap: 3 iterations max. The agent prints a clear "Auto-fix log" showing what was changed and what recovered. No silent fixes - you always see what got patched.

What auto-fix WILL NOT do:
- Modify the system under test (the API itself).
- Change assertions to make a failing test pass (that would hide bugs).
- Loop forever - 3 iterations and it stops, leaving the rest for you.
- Bail the Newman run on the first failure. The full suite always runs to completion so the HTML report captures every failure in one pass. `--bail` is reserved for manual single-bug triage by a human (Phase 11e).

Two guarantees you can rely on:
- **Initial Newman run always completes.** No bail. The HTML report shows you every failure, not just the first one.
- **Postman cloud and local JSON stay in sync.** Every patch the agent makes to a Postman request is immediately re-exported to the local collection JSON. You will never see Postman and the local file disagree - they always reflect the same state.

### Stage 2 - Manual debugging (the playbook for what auto-fix could not solve)

For failures that survived auto-fix, the agent surfaces the Phase 11 debugging playbook (canonical reference in `qa-api-test-setup.md`):

1. **Classify the failure** as one of five categories: real bug, spec drift, test setup issue, environment issue, or flaky test. The category points to the fix-path.
2. **Read the HTML report properly** - jump to the "Failed Tests" tab, click into the failed request, read the Request/Response/Test results/Console tabs. Never judge by status code alone - read the full response body.
3. **Reproduce with curl** before doing anything else. This is a hard project rule: no ticket gets raised without a manual curl reproduction. Copy the exact request from the report and run it; the result tells you whether it's a real bug, spec drift, or setup issue.
4. **Match against common HTTP patterns** - 401 cascade after login fails, 422 from wrong state, 409 from missing `{{$timestamp}}`, 400 from spec drift (run `/qa-api-sync`), etc. Each has a known fix-path.
5. **Use Newman flags for deeper digging** - `--verbose --bail --folder "<name>"` lets you focus on one failing request with full bodies in the console.
6. **Decision tree at the end** - based on classification, either fix the cause, run `/qa-api-sync`, or proceed to Stage 3 below.

For ticket-collection failures specifically, `qa-test-ticket.md` Phase 8 lists 6 things to check first (Setup chain, credential gate creds, state preconditions, scope mixing, timestamp placement, spec drift) before falling through to the generic playbook.

### Stage 3 - Log confirmed bugs to Jira (verification-gated)

Only triggered for `/qa-test-ticket` (not first-time setup) and only AFTER Stage 1 + Stage 2 have not resolved the failure. The agent never auto-files tickets - the user has to opt in.

When invoked, Phase 9 walks through this gate for each remaining failure:

1. **Curl reproduction** - the project rule. The agent runs curl with the exact request and reads the full response. If curl returns 2xx the failure was NOT real (probably flaky or state-poisoned), the candidate is dropped, no ticket gets filed.
2. **Dedupe check** - searches Jira for existing bug tickets in the same project that match the endpoint or error code. If matches found, the agent offers to add a comment to the existing ticket instead of creating a duplicate.
3. **Priority prompt** - defaults to Medium, user can override per batch.
4. **Structured ticket creation** - uses a 6-section format: Context / What we tested / Expected / Actual / Reproduction (curl) / Test run metadata. Linked to the parent ticket via "relates to". Saved automatically to Jira via the Atlassian plugin.
5. **Confirmation report** - the agent prints the new ticket key and URL.

The Jira project key is auto-extracted from the first ticket key you ever run (e.g. `PROJ-123` -> `PROJ`) and saved to `.env` as `JIRA_PROJECT_KEY` so subsequent runs do not ask again.


## Common scenarios

### "The dev team just deployed new endpoints, my tests are stale"
Run `/qa-api-sync` manually. The next scheduled run would also catch it, but you do not have to wait.

### "I want to start testing a new Jira ticket"
Run `/qa-test-ticket <KEY>`. It auto-syncs first, then builds the ticket collection.

### "My test failed, I need to verify it is a real bug"
Reproduce manually with curl and read the full response body before raising a ticket. (Project rule.)

### "I want to pause the daily sync for a week"
`/schedule delete <id>` to stop, then re-create the schedule when ready.

### "I forgot what creds the test account uses"
Check `.env`.

### "My snapshot file got corrupted"
Delete it, then re-run `/qa-api-test-setup` Phase 9 only. Or delete the snapshot file entirely and the next `/qa-api-sync` will treat the current spec as the new baseline (you lose the drift history but nothing else breaks).

### "I accidentally ran /qa-api-test-setup on an already-set-up project"
The preflight guard will stop and refuse to proceed. It lists what it detected and tells you which command you actually wanted (`/qa-api-sync` for spec updates, `/qa-test-ticket` for ticket work, `./run-tests.sh` to just run the suite). To override (only if you genuinely want a fresh duplicate collection), type the exact phrase `force re-setup` when prompted - generic `y/n` is intentionally not accepted.

### "My previous /qa-api-test-setup got interrupted halfway, .env exists but nothing else"
Just re-run `/qa-api-test-setup`. With only `.env` present (1 signal), the preflight lets it through. Phase 1 reads the existing `.env`, only asks for missing keys, and continues from where you left off.
