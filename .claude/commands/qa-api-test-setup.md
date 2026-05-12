---
name: qa-api-test-setup
description: "First-time API test suite setup — verifies plugins (Postman/Atlassian/Newman), discovers or creates .env, lists Postman workspaces, builds the main collection from an OpenAPI spec with auto-generated happy + negative tests per endpoint, writes a Newman run script, and creates the initial api-snapshot file used by /qa-api-sync to track future spec drift."
---

# API Test Suite Setup

You are setting up a fully automated API testing suite using Postman + Newman for a new project. Follow every step in order. Do not skip steps.

This command is for **first-time project setup only**. For per-ticket test collections on an already-set-up project, use `/qa-test-ticket <KEY>` instead.

---

## Preflight — Already-set-up guard

Run this BEFORE Phase 0. It prevents accidental re-setup that would create duplicate Postman collections and overwrite local files.

### Detect setup artifacts in the current working directory

Check for the presence of each of the following:

| Signal | Detection |
|---|---|
| `.env` file with required keys | `test -f .env` and the keys `BASE_URL`, `OPENAPI_SPEC`, `AUTH_TYPE` all present |
| Main collection JSON | `ls *-collection.json 2>/dev/null` matches at least one file (excluding ticket-style names like `proj123-collection.json`) |
| Snapshot file | `ls api-snapshot-*.json 2>/dev/null` matches at least one file |
| Newman env | `test -f newman-env.json` |
| Run script | `test -f run-tests.sh` |

Count how many signals are present.

### Decision

**If 3 or more signals are present** → treat the project as already set up. Stop and tell the user:

> This project appears to be already set up:
>   - `<list each detected artifact>`
>
> `/qa-api-test-setup` is for first-time setup only. To work with an existing project:
>
>   - New endpoints added by devs?     -> `/qa-api-sync`
>   - New Jira ticket to test?         -> `/qa-test-ticket <KEY>`
>   - Just want to run the full suite? -> `./run-tests.sh`
>
> If you really want to re-setup from scratch (this will create a duplicate collection in Postman and overwrite local files), confirm by typing exactly:
>
>   `force re-setup`
>
> Otherwise I will stop here.

Wait for the user's reply.

- If they type exactly `force re-setup` → proceed to Phase 0, but first warn: `Force re-setup confirmed. Existing local files will be overwritten and a duplicate Postman collection will be created. Continuing.`
- If they type anything else (including "yes", "y", "ok", "go ahead") → STOP. Do not proceed. The escape hatch requires the exact phrase to avoid foot-gun mistypes.

**If fewer than 3 signals are present** → the project is not set up (or only partially set up). Proceed to Phase 0 normally. If 1-2 signals are present, mention them in passing so the user knows what was detected:

> Detected some existing artifacts (`<list>`) but project is not fully set up. Continuing with setup. Existing files in `.env` will be preserved where possible.

---

## Phase 0 — Plugin & tool readiness (silent prereqs)

Do these BEFORE asking the user anything.

### 0a — Postman plugin

Call `mcp__plugin_postman_postman__getAuthenticatedUser`.

- **Success** → note username + user ID. Tell user: `Postman connected as: <username>`. Continue to 0b.
- **Fails (tool not found / auth error)** → stop and tell the user:

> Postman plugin is not connected. Fix this first.
>
> 1. Generate a Postman API key at https://go.postman.co/settings/me/api-keys (name it "Claude Code", copy — starts with `PMAK-`).
> 2. Decide **where to put the key**. Two valid locations — pick one (you don't need both):
>
>    **(a) Project-local (recommended for cloners)** — `<project>/.claude/settings.local.json` (gitignored, never committed). Per-project; great if you work on multiple Postman accounts/workspaces.
>    ```bash
>    cp .claude/settings.local.json.example .claude/settings.local.json
>    # then edit .claude/settings.local.json and paste your PMAK
>    ```
>    If `.claude/settings.local.json` already exists (e.g. with permissions), merge an `env` block into it instead of overwriting:
>    ```json
>    {
>      "permissions": { ... existing ... },
>      "env": { "POSTMAN_API_KEY": "PMAK-your-key-here" }
>    }
>    ```
>
>    **(b) Global (one-time-per-machine)** — `~/.claude/settings.json`. Shared across every project on this laptop.
>    ```json
>    "env": { "POSTMAN_API_KEY": "PMAK-your-key-here" }
>    ```
>
>    Local wins if both are set (Claude Code's settings cascade auto-merges).
>
> 3. Either way, ensure `"postman@claude-plugins-official": true` is in `enabledPlugins` in `~/.claude/settings.json`.
> 4. Restart Claude Code, then re-run `/qa-api-test-setup`.

Stop until the user confirms it's working.

### 0b — Atlassian plugin (Confluence + Jira)

Call `mcp__plugin_atlassian_atlassian__atlassianUserInfo`.

- **Success** → tell user: `Atlassian connected as: <name> (<email>)`. Continue to 0c.
- **Fails** → ask the user this question and wait for their choice:

> To pull API docs from Confluence and (later) test directly from Jira tickets, the Atlassian plugin needs to be authorized. **This is recommended.** Without it, you can still paste ticket/doc content here manually when needed.
>
> **(a) Authorize now (recommended)** — I'll trigger the auth flow; click the link Claude Code shows you, approve in browser, then re-run.
> **(b) Skip / use manual paste mode** — we'll continue; you'll paste Confluence content if needed.
>
> Which? (a/b)

- If **(a)** → tell them to ensure `"atlassian@claude-plugins-official": true` is in `enabledPlugins` in `~/.claude/settings.json`, then re-run. The first plugin call will surface the OAuth URL.
- If **(b)** → note "manual paste mode for Atlassian" and continue. Confluence step later becomes a paste prompt.

### 0c — Newman

```bash
which newman && newman --version
```

If missing:
```bash
npm install -g newman newman-reporter-htmlextra
```

Confirm install before continuing.

---

## Phase 1 — `.env` discovery and project info

### 1a — Check for `.env` in the current working directory

```bash
test -f .env && echo "EXISTS" || echo "MISSING"
```

### 1b — If `.env` EXISTS

Read it and print a checklist of what's present vs missing. The required keys are:

| Key | Used for |
|---|---|
| `BASE_URL` | API server root |
| `OPENAPI_SPEC` | Path or URL to OpenAPI JSON/YAML |
| `HEALTH_PATH` | Reachability check path used by run-tests.sh |
| `LOGIN_EMAIL` | Test account email (if JWT auth) |
| `LOGIN_PASSWORD` | Test account password (if JWT auth) |
| `PLATFORM_API_KEY` | Platform-level API key (if API-key auth) |
| `JIRA_PROJECT_KEY` | Optional; auto-populated on first `/qa-test-ticket` call |

**Auto-detected keys (NOT required in `.env`):** `AUTH_TYPE`, `LOGIN_PATH`, `TOKEN_JSON_PATH` are auto-detected from the OpenAPI spec during Phase 1d below. If the user has manually set any of these in `.env` as overrides, the override wins.

Print:
```
.env discovery:
  ✓ BASE_URL = <value>
  ✓ LOGIN_EMAIL = <value>
  ✗ OPENAPI_SPEC (missing)
  ✗ HEALTH_PATH (missing)
  (AUTH_TYPE, LOGIN_PATH, TOKEN_JSON_PATH will be auto-detected from spec)
  ...
```

Ask the user ONLY for the missing required items, plus things `.env` can't carry: project name + Postman workspace + (optional) Confluence space.

### 1c — If `.env` is MISSING

Ask the user for everything below, one logical group at a time:

1. **Project name** (for collection name + filenames, e.g. `MyProject`)
2. **API base URL** (e.g. `https://api.example.com`)
3. **OpenAPI spec source** — ask: "Do you have a URL for the API docs (Swagger, Scalar, Redoc, Confluence), or a local JSON/YAML file path?"
   - URL preferred. Fetch and validate (see Phase 2 for Scalar/HTML auto-discovery).
   - File path is the fallback.
4. **Health check path** — a reachable endpoint for run-tests.sh to ping (e.g. `/api/v1/health`). If the user is unsure, suggest the simplest GET endpoint from the spec.
5. **Login credentials** — email + password (only needed if the spec has any JWT-protected endpoints; auto-detected in 1d).
6. **Platform API key** — only needed if the spec has any apiKey-protected endpoints; auto-detected in 1d.
7. **Postman workspace** — call `mcp__plugin_postman_postman__getWorkspaces`, show as numbered list:
   ```
   Which Postman workspace should I create the collection in?
     1. My Workspace (personal)
     2. Team Workspace 1
     3. Project Workspace
   ```
   User picks a number. **Do NOT save workspace ID to `.env`** — always list fresh on each run.
8. **Confluence space key** (optional) — only if Atlassian was connected in 0b. Skip otherwise.

**Do not** ask about AUTH_TYPE, LOGIN_PATH, or TOKEN_JSON_PATH. Those get auto-detected from the spec in Phase 1d.

After all answers collected (and after Phase 1d completes), **write `.env` to the project directory** with the persistent values. Confirm to user: `Created .env at <path>`.

### 1d — Auto-detect auth shape from the OpenAPI spec

After the spec is loaded (Phase 2 logic can be pulled forward to here, or this can run as the first step of Phase 2 and feed back into Phase 1c's `.env` write), derive three values automatically. Show the detected values and ask the user to confirm before continuing.

**Step 1 - Detect `AUTH_TYPE`:**

Inspect `components.securitySchemes` in the spec:
- If a scheme with `type: http` and `scheme: bearer` is present → `AUTH_TYPE=jwt`
- If a scheme with `type: apiKey` is present → `AUTH_TYPE=apikey`
- If both → `AUTH_TYPE=both`
- If neither (rare; spec has no auth) → ask the user

Allow override: if `.env` already has `AUTH_TYPE` set, use it instead and skip detection.

**Step 2 - Detect `LOGIN_PATH`** (only if `AUTH_TYPE` includes `jwt`):

Scan operations in the spec. Score each candidate:
- Operations tagged `Authentication`, `Auth`, `Login`, `Session`, `Token`, or similar → high score
- `POST` operations with paths matching `*/login`, `*/signin`, `*/auth/login`, `*/auth/token`, `*/oauth/token` → high score
- Operations whose `summary` or `operationId` contains `login`, `signin`, `authenticate` → medium score

Pick the highest-scoring candidate. If there is a tie or zero matches, ask the user to pick from the candidates.

Allow override: if `.env` has `LOGIN_PATH` set, use it.

**Step 3 - Detect `TOKEN_JSON_PATH`** (only if `AUTH_TYPE` includes `jwt`):

Look up the login operation's `200`/`201` response schema. Recursively walk the schema's properties, scoring each leaf string property:
- Property name in `{accessToken, access_token, jwt, token, idToken, id_token, authToken}` → high score
- Property name contains "token" but not "refresh" → medium score

Build the dotted path from the response root to the highest-scoring property (e.g. `payload.data.tokens.accessToken`).

Allow override: if `.env` has `TOKEN_JSON_PATH` set, use it.

**Step 4 - Show the user what was detected:**

```
Auto-detected from OpenAPI spec:
  AUTH_TYPE       = <detected value>
  LOGIN_PATH      = <detected value>     (only shown if JWT)
  TOKEN_JSON_PATH = <detected value>     (only shown if JWT)

Are these correct? (y/n/edit-one)
```

- `y` → use as-is. Do NOT write these to `.env`. They are recomputed each setup run.
- `n` → ask user to provide all three manually. If they provide values, write them to `.env` as explicit overrides.
- `edit-one` → ask which key to override, take that single value from the user, write only that key to `.env` as an override.

**Step 5 - Verify with one live login call (JWT only):**

If `AUTH_TYPE` includes `jwt` AND login credentials are available in `.env`:
```bash
curl -s -X POST "<BASE_URL><LOGIN_PATH>" \
  -H "Content-Type: application/json" \
  -d '{"email":"<LOGIN_EMAIL>","password":"<LOGIN_PASSWORD>"}'
```

Parse the response. Walk the detected `TOKEN_JSON_PATH` against the actual response. If the value found is a non-empty string longer than 20 characters, the path verifies. Confirm: `Live login verified - token extracted at TOKEN_JSON_PATH.`

If it fails (path returns null/undefined/short string), tell the user:
```
The detected TOKEN_JSON_PATH (<path>) did not return a valid token from the live login.
Actual response: <truncated response body>
Please tell me the correct dotted path to the access token in this response.
```

Wait for the user's correct path. Write it to `.env` as an explicit override.

If live verification cannot run (no creds, server unreachable), skip it and warn: `Live login verification skipped - the agent will assume detected TOKEN_JSON_PATH is correct. If the suite fails with login errors, edit TOKEN_JSON_PATH in .env.`

---

## Phase 2 — Read the OpenAPI spec

### 2a — Load the spec (with Scalar / HTML auto-discovery)

The source from Phase 1c step 3 is either a URL or a local file path. Resolve it to a parseable OpenAPI JSON/YAML document using this fallback chain:

1. **Local file path** → read directly with `Read`. Done.
2. **URL returning JSON or YAML** → fetch with `curl -s` and parse. Done.
3. **URL returning HTML** (Scalar / Swagger UI / Redoc reference page) → fetch the HTML and try, in order:
   - Look for a Scalar embed: `<script id="api-reference" data-url="..."` → fetch that URL.
   - Look for an inline Scalar spec: `<script id="api-reference" type="application/json">…</script>` → parse the inline content.
   - Look for a Redoc embed: `<redoc spec-url="..."></redoc>` → fetch that URL.
   - Look for a Swagger UI config: `url: "..."` or `urls: [{ url: "..." }]` inside `<script>` blocks → fetch.
   - Try common conventional paths against the URL's origin: `/openapi.json`, `/openapi.yaml`, `/swagger.json`, `/api/openapi.json`, `/api-docs`, `/v3/api-docs`. Stop at the first one that parses as OpenAPI.
4. **If all auto-discovery fails** → ask the user once: `This looks like a Scalar/Swagger UI page but I could not locate the underlying OpenAPI JSON. Paste the raw spec URL (or a local file path).` Use their answer and retry from step 1.

Validate the loaded document parses as OpenAPI (has `openapi:` or `swagger:` top-level key + a `paths:` object). If it does not parse → stop and show the parse error.

### 2b — Mandatory endpoint count print

Before anything else, count every `(method, path)` pair in `paths.*` (skip `parameters`, `summary`, `description` at path level — only count actual HTTP verb keys: `get`, `post`, `put`, `patch`, `delete`, `head`, `options`, `trace`). Print this block to the user — it is mandatory and cannot be skipped:

```
✓ OpenAPI spec loaded
  Source:          <original URL or file path>
  Resolved spec:   <derived JSON URL if Scalar/HTML, else "same as source">
  OpenAPI version: <openapi/swagger version>
  Total endpoints: <N>
  Modules (tags):  <M>
    00. <Tag1> (<count>)
    01. <Tag2> (<count>)
    ...
    (untagged: <count>)
  Auth schemes:    <comma-separated from components.securitySchemes>
```

**Hard stop if `Total endpoints = 0`** — print: `Spec parsed but no endpoints found. The source is likely wrong or the paths object is empty. Aborting.` and exit.

### 2c — Extract details

Now extract for planning:

- All **endpoints grouped by tag** (each tag = a module/folder)
- Per endpoint: method, path, summary, required body fields, query params, response schema
- **Auth requirements** per endpoint (bearerAuth vs apiKey vs none)
- Enum values for any constrained fields

Scan the spec completely before planning.

---

## Phase 3 — Read Confluence (if available)

Only if Atlassian connected AND user provided a space key in 1b/1c:

1. `mcp__plugin_atlassian_atlassian__getConfluenceSpaces` — find the space
2. `mcp__plugin_atlassian_atlassian__getPagesInConfluenceSpace` — list pages
3. `mcp__plugin_atlassian_atlassian__getConfluencePage` — read pages mentioning "API", "onboarding", "test cases", "endpoints", or module names
4. Extract business rules, expected error codes, field constraints not in the spec

If user chose manual paste mode in 0b and gave Confluence content: use the pasted text instead.

Combine with the spec for richer assertions.

---

## Phase 4 — Plan and confirm

Decide the structure using the **nested module / test-type pattern**:

### Folder hierarchy

```
<ProjectName> API — Automated Tests
├── Happy Path - All Endpoints
│   ├── POST /auth/login
│   ├── GET  /orders
│   ├── POST /orders
│   └── ... one valid-data request per endpoint in the spec
├── 00. <Module1>
│   ├── Positive
│   │   ├── TC01: <happy path> (200)
│   │   └── TC02: <happy path 2> (201)
│   └── Negative
│       ├── TC03: <auth fail> (401)
│       └── TC04: <validation fail> (400)
├── 01. <Module2>
│   ├── Positive
│   │   └── TC01: <happy path> (200)
│   └── Negative
│       └── TC02: <auth fail> (401)
...
```

### Rules

- **Happy Path - All Endpoints** is a top-level folder, always created first, alongside (not replacing) the module folders. It contains exactly one valid-data request per `(method, path)` in the spec — no negatives. Purpose: a fast smoke pass across the entire API in one folder. See Phase 5b/5c for build details.
- **Top-level module folders** = one per API **tag** (from the OpenAPI spec's `tags` field), named `<NN>. <TagName>` where `NN` is zero-padded order in the spec. If the spec has no tags, ask the user how to group (by path prefix, by HTTP method, or one big folder).
- **Sub-folders** = always two under each module: `Positive` and `Negative`. Always create both, even if one starts with a single test.
- **TC numbering** = **resets per module**. TC01 starts fresh inside `00. <Module1>`, then again at TC01 inside `01. <Module2>`. Within a module, numbering is continuous across Positive then Negative (Positive ends at TC04, Negative starts at TC05). Happy Path requests are **not** TC-numbered — they are named `<METHOD> <path>`.
- **Minimum coverage per endpoint** = appears in **Happy Path - All Endpoints** AND has at least one happy path in its module's `Positive` AND at least one auth-failure negative (no auth → 401) in its module's `Negative`. Add more negatives where the spec or context warrants:
  - Validation failure (missing required field → 400)
  - Not found (non-existent ID → 404)
  - Conflict (duplicate where applicable → 409)
  - Wrong state (state transitions where applicable → 422)
- **Collection variables set by certain requests** = note explicitly which Positive tests set `accessToken` (login), `test<Entity>Id` (creates), etc.

### Print the plan

```
Plan for <ProjectName> API

Total endpoints in spec: <N>
Modules detected from spec tags: <M>

Happy Path - All Endpoints (<N>):
  POST /auth/login
  GET  /orders
  POST /orders
  ...

00. <Module1>
  Positive (<N>):
    TC01: <description> (200)
    TC02: <description> (201)
  Negative (<N>):
    TC03: <description> (401)
    TC04: <description> (400)

01. <Module2>
  Positive (<N>):
    TC01: <description> (200)
  Negative (<N>):
    TC02: <description> (401)

...

Total folders: 1 (Happy Path) + <M> modules × 2 sub-folders each = <T>
Total requests: <N> (Happy Path) + <N> (module Positive + Negative) = <T>

Variable-setting requests:
  - Happy Path > POST /auth/login → sets accessToken, refreshToken
  - 00. <Module1> > Positive > TC01: Login → sets accessToken, refreshToken
  - 02. <Module3> > Positive > TC01: Create Product → sets testProductId
  - ...
```

Ask:

> Does this look right, or any changes?

**Wait for confirmation before building.**

---

## Phase 5 — Build the Postman collection

### 5a — Create the collection

`mcp__plugin_postman_postman__createCollection`:
- `name`: `<ProjectName> API — Automated Tests`
- `description`: `Automated test suite for <ProjectName>. Base URL: <baseUrl>`
- `auth`: bearer using `{{accessToken}}` at collection level
- `variable`: `baseUrl`, `accessToken`, `refreshToken` (if JWT), `platformApiKey` (if API key), one `test<Entity>Id` per creatable resource
- `workspace`: chosen workspace ID

Save the returned collection ID.

### 5b — Create folders

**Step 1 — Happy Path - All Endpoints (created first, top-level):**

Call `mcp__plugin_postman_postman__createCollectionFolder` with `name`: `Happy Path - All Endpoints` and no `parentFolderId`. Save the returned ID as `happyPathFolderId`. This is the first folder so it sits at the top of the collection.

**Step 2 — Module folders (two levels deep):**

For each module from Phase 4's plan:

1. Create the top-level module folder via `mcp__plugin_postman_postman__createCollectionFolder` with `name`: `<NN>. <ModuleName>`. Save the returned folder ID as `<module>ModuleFolderId`.
2. Create the `Positive` sub-folder under it via the same MCP call but with `parentFolderId: <module>ModuleFolderId`. Save the returned ID as `<module>PositiveFolderId`.
3. Create the `Negative` sub-folder the same way. Save ID as `<module>NegativeFolderId`.

After all folders are created, you should have a map like:
```
{
  "Happy Path - All Endpoints": "<happyPathFolderId>",
  "00. Module1": {
    moduleFolderId:   "...",
    positiveFolderId: "...",
    negativeFolderId: "..."
  },
  "01. Module2": {
    ...
  }
}
```

### 5c — Create requests

Build in two passes.

**Pass 1 — Happy Path - All Endpoints (build first):**

For every `(method, path)` in the spec, create one request via `mcp__plugin_postman_postman__createCollectionRequest`:
- `folderId`: `happyPathFolderId`
- `name`: `<METHOD> <path>` exactly (e.g. `POST /auth/login`, `GET /orders/{id}`). No `HP` prefix, no `TC` prefix, no status suffix.
- `method`, `url` from spec — host as `{{baseUrl}}`. Path params like `{id}` resolve to chained variables like `{{testOrderId}}` if a prior request creates that entity; otherwise leave as-is for the user to wire up.
- `header`: `Content-Type: application/json`; for API-key endpoints add `Authorization: ApiKey {{platformApiKey}}` and set `auth.type = "noauth"`
- `body`: realistic valid data drawn from the spec schema + examples. Use `{{$timestamp}}` for unique fields.
- `events` test script — **status-only assertion**:
  ```js
  pm.test('Status is 2xx', () => {
    pm.expect(pm.response.code).to.be.oneOf([200, 201, 202, 204]);
  });
  ```
  No deep schema checks — that's what the per-module Positive folder is for. Happy Path is the smoke pass.
- **Variable chaining still applies**: if the endpoint is the login or creates an entity, include the same `pm.collectionVariables.set(...)` block as Pass 2 below so downstream Happy Path requests can use the IDs/tokens. The login endpoint MUST appear first in Happy Path (reorder if needed) so `accessToken` is set before any other request runs.

**Pass 2 — Module Positive / Negative requests:**

For each request in each module folder, route it to the correct sub-folder by test type. Use `mcp__plugin_postman_postman__createCollectionRequest`:
- `folderId`: the **sub-folder** ID (`<module>PositiveFolderId` or `<module>NegativeFolderId`), NOT the module folder ID
- `method`, `url` from spec — host as `{{baseUrl}}`
- `header`: `Content-Type: application/json`; for API-key endpoints add `Authorization: ApiKey {{platformApiKey}}` and set `auth.type = "noauth"`
- `body`: realistic test data from the spec; use `{{$timestamp}}` for unique fields (email, codes, names)
- `events` test script — follow rules below

**Test script rules:**
- Assert status code first: `pm.test('Status is 200', () => pm.response.to.have.status(200));`
- Use `let` at top scope, never `const` — Newman throws `SyntaxError: Identifier already declared`
- Login → save tokens:
  ```js
  const d = pm.response.json().payload?.data;
  if (d) {
    pm.collectionVariables.set('accessToken', d.tokens.accessToken);
    pm.collectionVariables.set('refreshToken', d.tokens.refreshToken);
  }
  ```
  Adjust the JSON path to match the actual response shape (use `TOKEN_JSON_PATH` from `.env`).
- Create endpoints → save new ID:
  ```js
  const d = pm.response.json().payload?.data;
  if (d?.id) pm.collectionVariables.set('testEntityId', d.id);
  ```
- List endpoints → save first item's ID:
  ```js
  let items = pm.response.json().page?.data;
  if (items?.length > 0) pm.collectionVariables.set('testEntityId', items[0].id);
  ```
- Negative tests → `pm.expect(pm.response.text()).to.include('ERROR_CODE');`

---

## Phase 6 — Export collection + create environment file

### 6a — Export collection locally

```bash
POSTMAN_API_KEY="<from .claude/settings.local.json or ~/.claude/settings.json>"
curl -s -H "X-API-Key: $POSTMAN_API_KEY" \
  "https://api.getpostman.com/collections/<OWNER_ID>-<COLLECTION_ID>" \
  -o "<project-dir>/<project-slug>-collection.json"
```

Owner ID = user ID from `getAuthenticatedUser`. Collection ID = from `createCollection` response.

### 6b — Newman environment file

Write `<project-dir>/newman-env.json`:
```json
{
  "id": "<project-slug>-env",
  "name": "<ProjectName> Environment",
  "values": [
    { "key": "baseUrl", "value": "<baseUrl>", "enabled": true },
    { "key": "accessToken", "value": "", "enabled": true },
    { "key": "refreshToken", "value": "", "enabled": true },
    { "key": "platformApiKey", "value": "<apiKeyValue>", "enabled": true }
  ]
}
```

Add one entry per collection variable.

---

## Phase 7 — Generate run-tests.sh and generate-issues.py from templates

Two files get written to the project root by reading templates from `.claude/templates/` and substituting placeholders.

### 7a — Write run-tests.sh

Read `.claude/templates/run-tests.sh`. Replace these placeholders with values from `.env` and the user's earlier answers:

| Placeholder | Substitute with |
|---|---|
| `__PROJECT_NAME__` | Project name from Phase 1 (e.g. `MyProject`) |
| `__PROJECT_SLUG__` | Lowercased hyphen-free version (e.g. `myproject`) |
| `__BASE_URL__` | `BASE_URL` from `.env` |
| `__HEALTH_PATH__` | A reachable endpoint path for the health-check (e.g. `/api/v1/health` or the simplest GET in the spec). If unclear, ask the user once. |
| `__HEALTH_AUTH_HEADER__` | Leave empty (`""`) for public health endpoints. If the health endpoint requires auth, build the header string (e.g. `Authorization: ApiKey {{platformApiKey}}` only if user confirms) - usually empty. |

Write the filled content to `<project-dir>/run-tests.sh` and `chmod +x` it.

### 7b — Write generate-issues.py

Read `.claude/templates/generate-issues.py`. This template has NO placeholders - it reads project info from CLI args at runtime (passed by `run-tests.sh`). Write it verbatim to `<project-dir>/generate-issues.py`. Do not chmod, it is invoked via `python3`.

This script parses the Newman JSON results and produces a human-readable `.txt` report classifying:
- Status mismatch issues (wrong HTTP code, with security/regression context)
- Message quality issues (vague error messages, with guidance on what good ones look like)

Outputs go to `collection-run-issues/<project-slug>-issues-<timestamp>.txt`.

### 7c — Verify both files exist

```bash
test -f run-tests.sh && test -f generate-issues.py && echo "Both helper scripts written."
```

If either file is missing, stop and tell the user which template failed.

---

## Phase 8 — Smoke test

```bash
curl -s --max-time 5 <baseUrl>/<any-endpoint> -w "\nHTTP_STATUS:%{http_code}"
```

If reachable → `cd <project-dir> && ./run-tests.sh`.

Watch for:
- Login returning 200 and saving `accessToken` ✓
- Downstream requests no longer 401 ✓
- `SyntaxError: Identifier already declared` → a `const` slipped into top scope; change to `let`
- `JSONError` → response wasn't JSON; check status + server logs

### 8a — Auto-fix loop (if any tests failed)

If the smoke run had any failures, apply the same auto-fix matrix used by `/qa-test-ticket` Phase 6 before reporting to the user. The agent's job is to leave only genuine bugs and env issues in the failure list, not to hand off every minor issue.

**Two non-negotiable rules in this phase:**

1. **Never `--bail` the Newman run.** The first run must complete fully so the HTML report shows every failure - bailing leaves an incomplete report that's worse than no report. Auto-fix iterations also run to completion.
2. **Cloud + file must stay in sync.** Every `updateCollectionRequest` MCP call must be followed by a re-export of the local `<project-slug>-collection.json` so the next Newman iteration reads the patched state. See the curl in 8b.

For each failed request:
1. Classify (spec drift / stale accessToken / setup timing / `{{$timestamp}}` missing / `const` syntax / transient 5xx / HTML error page / real bug / env / account / flaky)
2. Apply the corresponding fix via Postman MCP (see qa-test-ticket.md Phase 6b for the full action table)
3. Re-export the local collection JSON (see 8b)
4. Re-run only the affected requests (no `--bail`)
5. Cap at 3 iterations

### 8b — Re-export local collection JSON after each patch

```bash
POSTMAN_API_KEY="<from .claude/settings.local.json or ~/.claude/settings.json>"
curl -s -H "X-API-Key: $POSTMAN_API_KEY" \
  "https://api.getpostman.com/collections/<OWNER_ID>-<COLLECTION_ID>" \
  -o "<project-slug>-collection.json"
```

Without this re-export, Newman keeps running the OLD local JSON and your patches will have no effect on the next iteration. Mandatory after every patch.

After auto-fix:
- Print a clear "Auto-fix log" showing what was changed in Postman cloud AND that the local JSON has been re-exported.
- Anything still failing in a fixable category after 3 iterations is treated as "stuck" and surfaced to the user.
- Anything in a non-fixable category (real bug / env / account / flaky) is surfaced as-is.

The full debugging playbook (Phase 11) is still available for the user to consult after auto-fix gives up. The Jira bug-logging flow lives in `/qa-test-ticket` Phase 9, not here - first-time setup should not log bugs (the suite is meant to validate the API works at all, not raise tickets on day one).

---

## Phase 9 — Create initial spec snapshot

After the collection is built and the smoke test has run, create the baseline snapshot file used by `/qa-api-sync` to detect future drift.

### 9a — Hash every endpoint in the spec

For each `(method, path)` in the OpenAPI spec, compute a SHA-256 hash of:
```
<method>|<path>|<sorted required body fields>|<sorted required query params>|<auth requirement>|<response 2xx schema keys>
```

### 9b — Write the snapshot file

Today's date: `YYYY-MM-DD`. Write to project root: `api-snapshot-<today>.json`:

```json
{
  "meta": {
    "specSource": "<URL or file path from .env>",
    "currentTotalEndpoints": <N>,
    "lastSyncedAt": "<ISO timestamp>"
  },
  "history": [
    {
      "syncedAt": "<ISO timestamp>",
      "type": "initial-snapshot",
      "added":   <N>,
      "changed": 0,
      "removed": 0,
      "addedList":   ["POST /api/v1/auth/login", "GET /api/v1/products", "..."],
      "changedList": [],
      "removedList": [],
      "ticketCollectionsTouched": []
    }
  ],
  "endpoints": {
    "POST /api/v1/auth/login": "sha256:...",
    "GET /api/v1/products":    "sha256:...",
    ...
  }
}
```

Confirm to user: `Initial snapshot written: api-snapshot-<today>.json (<N> endpoints baselined).`

From now on, `/qa-api-sync` will diff against this file when developers push new/changed endpoints.

---

## Phase 10 — Summary

```
Postman plugin:   connected as <username>
Atlassian plugin: connected as <name> / manual-paste mode
Newman:           v<version>

Collection:       <name>  (ID: <id>)
Workspace:        <workspace name>
Folders:          <N>
Total requests:   <N>
Passed:           <N>
Failed:           <N>
Report:           <path>/newman-reports/<file>.html

.env written:     <path>/.env
Spec snapshot:    <path>/api-snapshot-<today>.json  (<N> endpoints baselined)

Next steps:
  - Run /qa-api-sync daily, or schedule it: /schedule daily at 2:30 PM BDT /qa-api-sync
  - Run /qa-test-ticket <KEY> when a new Jira ticket needs scoped tests

Fixes needed (if any):
  - <list failing tests + reason>
```

If server was unreachable → tell user everything is built and to run `./run-tests.sh` when server is back.

---

## Phase 11 — Debugging failed tests (playbook)

When the smoke test (Phase 8) reports any failed test, or when the user runs `./run-tests.sh` later and tests fail, do NOT auto-raise a Jira ticket. Walk through this triage methodically. This playbook is the canonical reference for all three agents (`/qa-test-ticket` and `/qa-api-sync` point back to it).

### 11a — Classify first (before any "fix")

Every failure falls into one of five categories. Identifying the category narrows the fix-path immediately:

| Category | Signal | Where the fix lives |
|---|---|---|
| **Real bug** | Server returns wrong data or 500-class error, manual curl reproduces it | Raise a Jira ticket (after curl repro) |
| **Spec drift** | Test expected one shape, response has a new/missing/renamed field | Run `/qa-api-sync` - the test isn't broken, the API moved |
| **Test setup issue** | A precondition wasn't met (wrong state, missing entity) | Fix the Setup folder; downstream tests will pass once state is right |
| **Environment issue** | Server unreachable, VPN dropped, DNS, expired account | Not a bug. Document and retry. |
| **Flaky test** | Passes 3 of 5 runs without code changes | Log it, do not silently retry. Investigate root cause (timing, race, shared state) |

### 11b — Read the HTML report properly

The report is at `newman-reports/<project>-report-<timestamp>.html`. Open it in the browser. The order of investigation:

1. **"Failed Tests" tab at the top** - jump straight to failures, skip the passes
2. **Click into the failed request** - you get four tabs:
   - **Request** - method, URL with rendered variables, headers, body actually sent
   - **Response** - status code, headers, full response body
   - **Test results** - which `pm.test(...)` assertion failed and why
   - **Console** - any `console.log(...)` output from the test script
3. **Note the timestamp** - if the team has server logs, cross-reference

The single biggest mistake is judging a failure by status code alone. Always read the response body - APIs often return `{ "error": "Order must be APPROVED before submission", "code": "ORDER_NOT_APPROVED" }` which tells you exactly what went wrong.

### 11c — Reproduce with curl (PROJECT RULE - never skip)

This step is non-negotiable per project standards: before raising any ticket at any priority, the failing request must be reproduced manually with curl and its full response body read.

Copy the request from the HTML report verbatim and run:

```bash
curl -X <METHOD> -v \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token-from-report>" \
  -d '<body-from-report>' \
  "<full-url-from-report>"
```

Interpret the result:

- **Curl succeeds (2xx) but suite said failure** -> test setup issue (state was different when curl ran vs when suite ran) - check Setup folder
- **Curl fails the same way as the suite** -> likely a real bug OR spec drift. Read the error message; if it mentions a field/shape change, run `/qa-api-sync` and re-test
- **Curl returns a clearly informative error** ("validation failed: regionId is required") -> spec drift, run `/qa-api-sync`
- **Curl returns 500 with a vague error or stack trace** -> real bug, raise a ticket with the curl command + full response in the description

### 11d — Common HTTP failure patterns

| Status | Pattern | Most likely cause |
|---|---|---|
| **401 on every request** | Cascade after login failed | Check the first Login request - it didn't save `accessToken`. Look at its response body. Common causes: wrong creds, login JSON path wrong (check `TOKEN_JSON_PATH` in `.env`), account locked |
| **401 on one specific endpoint** | Cascade is fine elsewhere | Endpoint may need a special role/permission. Check ticket for "admin user" or similar role hints |
| **400 (missing required field)** | Endpoint that worked yesterday | Spec drift - the API added a required field. Run `/qa-api-sync` |
| **404 on Get/Update/Delete with `{{testEntityId}}`** | Earlier List/Create save script failed | Check the upstream request's test script - the ID chaining didn't work. Look for empty `testEntityId` in the environment |
| **409 (conflict / duplicate)** | Happens on re-runs only | A field that should be unique isn't using `{{$timestamp}}`. Find it and add the placeholder |
| **422 (unprocessable / wrong state)** | A state transition is illegal | Setup didn't build the expected precondition state. E.g. trying to approve an entity that's still DRAFT |
| **429 (rate limit)** | Suite ran fine before, suddenly failing | Bump `--delay-request` from 300 to 1000+ ms |
| **500 (server error)** | Vague stack trace in response | Real backend bug. Reproduce with curl, capture the request + response, raise a Jira ticket |
| **Timeout / ECONNRESET** | Hit-or-miss failures | Server cold start, slow infra. Bump `--timeout-request` to 30000. If consistent, check server health |
| **JSONError in test script** | Suite log says "JSONError" | Response wasn't JSON. Probably an HTML error page from a load balancer or proxy. Check the raw response body |
| **`SyntaxError: Identifier already declared`** | Newman startup crash | A `const` slipped into top scope of a test script. Find it and change to `let` |

### 11e — Newman flags for deeper digging (manual human use only)

When the HTML report is not enough, the human can re-run Newman with debugging flags. **The agent's auto-fix phases never use `--bail`** - these flags are for the human, after auto-fix and the full report exist.

```bash
# Run only one folder
newman run <project>-collection.json -e newman-env.json --folder "00. Authentication"

# Run only a single named request
newman run <project>-collection.json -e newman-env.json --folder "TC11: Submit Order - Already Submitted"

# Verbose: full request + response bodies in console
newman run <project>-collection.json -e newman-env.json --verbose

# Stop at first failure (HUMAN TRIAGE ONLY - useful when chasing a single cascading bug;
# never use this for the full suite, you want the report to be complete)
newman run <project>-collection.json -e newman-env.json --bail

# Increase per-request timeout for slow staging servers
newman run <project>-collection.json -e newman-env.json --timeout-request 30000

# Slow down requests if hitting rate limits
newman run <project>-collection.json -e newman-env.json --delay-request 1000

# Combine: verbose + single failing folder - the "give me everything about this one failure" combo
# (notice: no --bail - the folder is already scoped, and you want every failure in it)
newman run <project>-collection.json -e newman-env.json \
  --folder "<folder>" --verbose
```

**Rule of thumb for `--bail`:** Only use it when you are narrowing in on ONE specific failing request and you want to stop the moment it fails so you can grab the verbose output. For everything else - full suite runs, folder runs, auto-fix re-runs - do not bail.

### 11f — Decision tree: what to do next

```
Test failed
│
├── Did the server respond at all?
│   ├── No  → environment issue (VPN/DNS/server down) → retry, don't raise ticket
│   └── Yes → continue
│
├── Does the response shape match what the test expected?
│   ├── No  → spec drift → /qa-api-sync → re-run
│   └── Yes → continue
│
├── Reproduce with curl. Does curl fail the same way?
│   ├── No  → test setup issue (state was different) → fix Setup folder
│   └── Yes → continue
│
├── Is the response body informative (clear error message)?
│   ├── Yes → fix the cause it points to (often a precondition or input)
│   └── No  (vague 500) → real bug → raise Jira ticket
│
└── Always include in the ticket: full curl command, full response body,
    timestamp, suite name (<KEY>-XXX), and a one-line summary of expected vs actual
```

### 11g — What NOT to do

- Do NOT modify a test to make it pass without understanding why it failed. That's how you hide real bugs.
- Do NOT immediately raise a ticket without curl repro - per project rule, all tickets need manual reproduction.
- Do NOT blame the test framework first. Newman/Postman bugs are rare; spec drift and state issues are common.
- Do NOT ignore intermittent failures. A test that fails 1 in 10 times is hiding a real timing/race/state issue and will eventually fail in production.
- Do NOT delete failing tests. If a test is genuinely no longer relevant, archive it (rename to `[ARCHIVED] <name>`) so the history stays visible.

### 11h — When the failure is actually correct behavior

Some "failures" are the test correctly catching a known limitation:

- Negative tests SHOULD fail with their expected error code - if `TC11: Submit Order - Already Submitted (400)` returns 400, that's a pass, not a fail. Make sure you're reading the report correctly.
- A status assertion that says "expected 200 got 201" might be a spec drift where the API now correctly returns 201 for creates - update the assertion, not the request.

---

## Reliability notes

- **Login must run first** — it sets `accessToken` for all downstream requests
- **`{{$timestamp}}` for unique fields** prevents 409 on re-run
- **ID chaining** — List requests save first item's ID; Get/Update/Delete use `{{testEntityId}}`
- **Never `const` at test script top scope** — use `let`
- **API-key endpoints** — set `auth.type = "noauth"` + manual `Authorization: ApiKey {{platformApiKey}}` header to override collection-level bearer auth
- **Never write `POSTMAN_API_KEY` to project `.env`** — it lives in `.claude/settings.local.json` (project-local, gitignored) or `~/.claude/settings.json` (global). Local wins if both set.
- **Never write workspace ID to `.env`** — always list fresh
- **Never `--bail` the Newman run** — initial run + auto-fix iterations must complete fully so the HTML report captures every failure in one pass. `--bail` is reserved for human-driven single-bug triage in Phase 11e.
- **Cloud + local JSON stay in sync** — every Postman MCP `updateCollectionRequest` call must be immediately followed by re-exporting the local `<project-slug>-collection.json` (the curl in Phase 8b). Otherwise Newman re-runs the old state and your patches do nothing.
