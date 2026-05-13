---
name: qa-test-ticket
description: "Per-ticket Postman test collection — pulls a Jira ticket (plugin or pasted), runs /qa-api-sync first, hard-stops on missing credentials, then builds a standalone <KEY>: <feature> collection in the project workspace with Setup / Positive / Negative folders, continuous TC numbering, and full auth-matrix negatives."
---

# Ticket-Driven API Test Collection

You are creating a **per-ticket** Postman test collection scoped to a single Jira ticket. The project must already be set up via `/qa-api-test-setup` (i.e. `.env`, main collection, and run script all exist).

Invocation accepts either form:
- A Jira ticket key: `/qa-test-ticket PROJ-123`
- A Jira ticket URL: `/qa-test-ticket https://yourcompany.atlassian.net/browse/PROJ-123`
- URL with query params: `/qa-test-ticket https://yourcompany.atlassian.net/browse/PROJ-123?focusedCommentId=9999`

The output is a **separate, standalone collection** inside the same Postman workspace as the main project, named after the ticket. It is not a fork of the main collection.

---

## Input normalization (parse the argument before Phase 0)

Run this first - extract a clean ticket key from whatever the user passed.

1. If the argument matches the regex `^[A-Z][A-Z0-9_]*-\d+$` → it is already a ticket key (e.g. `PROJ-123`, `ACME-42`, `WEB_UI-7`). Use as-is.
2. If the argument contains `/browse/` → extract the key with regex `/browse/([A-Z][A-Z0-9_]*-\d+)`. Strip any query string or URL fragment after the key.
3. Anything else (e.g. just a number, a free-form string, an unparseable URL) → stop and tell the user:
   > Could not parse a Jira ticket key from `<input>`. Pass either a key like `PROJ-123` or the full ticket URL (e.g. `https://yourcompany.atlassian.net/browse/PROJ-123`).

Confirm the parsed key to the user before continuing:
> Working on ticket: **PROJ-123** (parsed from `<original input>`)

From this point on, use only the extracted key everywhere downstream: collection name, local filename, Jira API calls, `JIRA_PROJECT_KEY` derivation in Phase 9a, ticket linkage, etc.

---

## Phase 0 — Prereq checks (silent)

### 0a — `.env` must exist

```bash
test -f .env || (echo "MISSING" && exit 1)
```

If missing → stop and tell user: `No .env in current directory. Run /qa-api-test-setup first.`

Load `.env`. Required keys: `BASE_URL`, `OPENAPI_SPEC`, `LOGIN_EMAIL`, `LOGIN_PASSWORD`, `AUTH_TYPE`, `TOKEN_JSON_PATH`, `LOGIN_PATH`. If any required key is missing → stop and tell user which keys to add.

### 0b — Postman plugin connected

Call `mcp__plugin_postman_postman__getAuthenticatedUser`. If fails → stop and refer to `/qa-api-test-setup` Phase 0a for setup.

### 0c — Atlassian plugin OR pasted ticket

Call `mcp__plugin_atlassian_atlassian__atlassianUserInfo`.

- **Success** → plugin mode. Proceed.
- **Fails** → ask the user:
  > Atlassian plugin not authorized. To pull the ticket automatically you need to authorize it (recommended).
  >
  > **(a) Authorize now** — enable `"atlassian@claude-plugins-official": true` in `~/.claude/settings.json`, restart, re-run.
  > **(b) Paste ticket content here manually** — I'll work from what you paste.
  >
  > Which? (a/b)

  If (b) → ask user to paste full ticket: summary, description, acceptance criteria, any linked specs. Use that content for Phase 1 instead of the plugin.

### 0d — Spec sync gate (run /qa-api-sync first)

Before building anything for this ticket, ensure the project's main collection reflects the current API spec. Stale spec = stale tests.

1. Check for an existing snapshot file: `ls api-snapshot-*.json 2>/dev/null`.
   - **No snapshot** → tell user: `No api-snapshot-*.json found. Run /qa-api-test-setup first to baseline the spec.` Stop.
2. Run the full `/qa-api-sync` flow internally (Phases 0–7 from `qa-api-sync.md`).
3. If drift was detected and applied → tell the user:
   > Spec drift was detected and applied before building this ticket's collection. The main collection (and any affected ticket collections) are now up to date.
4. If user declined the sync in `/qa-api-sync` Phase 3 → ask:
   > You skipped the sync. The ticket collection may be built against stale request shapes. Continue anyway? (y/n)

   If `n` → stop. If `y` → proceed but warn that test failures may reflect spec drift, not real bugs.

### 0e — Workspace target

Read the main project's collection metadata from local export (`<project-slug>-collection.json` in CWD) → extract collection UID → call `mcp__plugin_postman_postman__getCollection` to find its workspace. Use that workspace ID for the new ticket collection.

If the project's collection file isn't in CWD → ask user which workspace, listing via `getWorkspaces`.

---

## Phase 1 — Pull and analyze the ticket

### 1a — Get ticket data

- **Plugin mode** → `mcp__plugin_atlassian_atlassian__getJiraIssue` with the key. Fetch summary, description, acceptance criteria, comments, linked Confluence pages.
- **Paste mode** → use the content the user pasted.

### 1b — Analyze scope

From the ticket content, identify:

1. **Feature/module name** — for the collection title (e.g. "Order Submission Flow")
2. **Endpoints touched** — match ticket terminology against the OpenAPI spec referenced in `.env`. List exact paths + methods.
3. **Acceptance criteria scenarios** — each AC item becomes one or more positive test cases
4. **State preconditions** — what entities must exist before the feature can be tested (e.g. a complex ticket may need: branch, customer, items, multiple orders in different states)
5. **Specific data values** mentioned in the ticket — SKU codes, amounts, status names, role names

Print the analysis:
```
Ticket: <KEY> — <summary>
Feature: <feature name>
Endpoints in scope: <N>
  POST /api/v1/orders
  GET  /api/v1/orders
  ...
AC scenarios: <N>
Preconditions detected: <list>
Specific data values: <list>
```

---

## Phase 2 — Credential gate (HARD STOP)

Cross-check what the ticket needs against `.env`. Common things tickets may need that aren't already there:

- A specific user role / alternate account (e.g. admin user, partner admin)
- Pre-existing entity IDs the ticket references explicitly (e.g. specific branch ID)
- Third-party API keys / integration tokens
- Feature-flag-gated account credentials
- Specific test data values mentioned in AC

**If anything is needed but not in `.env`**, stop and ask:

> This ticket needs the following that I don't have in `.env`:
>
>   - <thing 1>  (e.g. admin user login email + password)
>   - <thing 2>  (e.g. integration API key for partner sync)
>
> Please provide these. If you don't have them:
>   - Ask the dev team
>   - Check the API documentation
>   - Check the linked Confluence page on the ticket
>
> I cannot proceed without these. Paste the values when you have them.

**Do not build with placeholders. Do not partially build. Wait for full info.**

Once the user provides:
- Ask: "Should I save the persistent ones (creds, API keys) to `.env` for reuse on future tickets? (y/n)"
- If y → append to `.env`. Do NOT save one-off IDs (those are ticket-specific).

---

## Phase 3 — Plan and confirm

Build the plan in **four folders**:

### Happy Path - All Endpoints folder
One valid-data request per `(method, path)` this ticket touches (the endpoints listed in Phase 1's analysis). No negatives, no auth-matrix variants — just a smoke pass across the ticket's surface area with valid data.

Naming: `<METHOD> <path>` exactly (e.g. `POST /api/v1/orders`, `GET /api/v1/orders/{id}`). No `HP` prefix, no `TC` prefix, no status suffix.

The login request belongs in `Setup` (not here) — Happy Path runs after Setup so `{{accessToken}}` is already populated.

**Dependency audit (run before printing the plan):** Apply the same audit logic as `/qa-api-test-setup` Phase 2d, scoped to **only the endpoints this ticket touches**. Build the DAG from the same four signals (auth requirement, path params, body refs, verb-within-resource order). Topological sort to get the audited Happy Path order. Treat anything Setup creates as a producer (e.g. if Setup includes `Create Supplier` which sets `{{testSupplierId}}`, then `GET /suppliers/{supplierId}` has its dependency satisfied and is not an orphan). Anything still unresolved becomes an orphan, placed at the end of Happy Path with a `{{test<Entity>Id}}` placeholder and flagged in the plan.

### Setup folder
- Login first
- Fetch existing IDs (e.g. branch, category) needed for the feature
- Create prerequisite entities (customers, items, etc.)
- Build feature-state precondition data (e.g. multiple orders in different lifecycle states)

Scale this folder to what the ticket actually needs. A simple ticket may have 1–3 Setup items. Complex tickets may have 20+. **Do not add fluff. Do not skip necessary state.**

Use `[GROUP1]`, `[GROUP2]` prefixes for related multi-step flows (e.g. `[ORD1] Create Order`, `[ORD1] Submit Order`, `[ORD1] Approve Order`).

### Positive folder
Every happy-path scenario from the ticket's acceptance criteria.

Naming: `TC<NN>: <scenario> (<expected status>)`
- `TC01: Get Order Prefill (200)`
- `TC02: Submit Order - Full Match → SUBMITTED (201)`
- `TC02b: Verify Order Status After Submission (200)` — use `b/c/d` suffix for verification steps tied to a primary case

### Negative folder
Apply the **full negative matrix** (`.claude/commands/qa-negative-matrix.md`) to every endpoint the ticket touches. Walk every row, check the condition, generate the test or mark `n/a` — same logic as `/qa-api-test-setup` Phase 5c.

Rows that almost always apply (auth-matrix from existing rule):
- `auth-no-token` → 401
- `auth-expired-token` → 401
- `auth-tampered-token` → 401
- `auth-malformed-token` → 401
- `auth-wrong-role` → 403 (if RBAC mentioned in ticket OR endpoint has owner-scoped path param)

Plus the rest of the matrix where applicable:
- `val-*` (missing required, wrong type, out of range, regex mismatch, enum out of range) → 400
- `res-*` (not found, wrong state, conflict) → 404/422/409
- `sec-*` (SQL injection, XSS, path traversal, CORS) → rejected/non-2xx
- `sec-rate-limit` only if `INCLUDE_RATE_LIMIT_TESTS=true` in `.env`

Cross-cutting assertions (`xcut-sensitive-leak`, `xcut-error-body-shape`, `xcut-no-stack-trace`, `xcut-response-time`, `xcut-schema-validation`) added INTO every test script — not as separate tests.

**Folder layout for ticket collections**: keep `Negative` **flat** (no `Auth Failures` / `Validation Failures` / `Resource Errors` / `Security Probes` sub-folders). Ticket surface area is small enough that flat is more navigable. The 4-way split is only for the main collection.

**Numbering is continuous across Positive → Negative**. If Positive ends at TC10b, Negative starts at TC11. Within Negative, the matrix-driven order is: Auth rows first (TC11-TC15), Validation rows next, Resource rows, then Security rows — but it's all one continuous TC sequence in one flat folder.

### Print the plan

```
Plan for <KEY>: <feature name>

Setup (<N>):
  1. Login                                 → sets {{accessToken}}, {{refreshToken}}
  2. Fetch Branch ID                       → sets {{testBranchId}}
  3. Create Supplier                       → sets {{testSupplierId}}
  ...

Happy Path - All Endpoints (<N>, audited flow order, with reasoning):
  1. POST /api/v1/orders                   — needs {{accessToken}} (Setup #1) + {{testBranchId}} (Setup #2); produces {{testOrderId}}
  2. GET  /api/v1/orders/{id}              — needs {{testOrderId}} (#1)
  3. PATCH /api/v1/orders/{id}             — needs {{testOrderId}} (#1)
  ...
  N. POST /api/v1/orders/{id}/audit-log    ⚠ orphan: no producer for {auditLogId}, placed at end

Orphans (placed at end with placeholder vars, fix manually if needed):
  - POST /api/v1/orders/{id}/audit-log    needs: {{testAuditLogId}}   reason: ticket doesn't create audit logs and no Setup producer; wire manually if test requires it

Positive (<N>):
  TC01: <scenario> (200)
  ...

Negative (<N>):
  TC11: <scenario> (4xx)
  ...

Total requests: <N>
```

Ask: `Does this flow look right, or any changes?` Same response options as `/qa-api-test-setup` Phase 4 (`y` / `n` / `edit`). Wait for confirmation.

---

## Phase 4 — Build the ticket collection

### 4a — Locate or create the shared Postman environment

Before creating the collection, ensure the **shared project env** (`<ProjectName> Environment`) exists. Ticket collections read/write tokens and chained IDs from this env, NOT from a per-ticket env (one env per project, shared by main + all ticket collections — keeps the workspace clean and lets tokens carry between collections).

1. Call `mcp__plugin_postman_postman__getEnvironments` for the workspace. Look for an env named `<ProjectName> Environment` (where `<ProjectName>` is derived from `BASE_URL` host or matches the main collection's project name).
2. **If found** → save its ID. Skip to 4b.
3. **If not found** → warn the user:
   ```
   No shared Postman environment found. The main collection from /qa-api-test-setup
   should have created "<ProjectName> Environment". I'll create one now with the
   universal vars (baseUrl, accessToken, etc.) so this ticket can run.
   ```
   Then create it via `mcp__plugin_postman_postman__createEnvironment` using the **same universal-var list** as `/qa-api-test-setup` Phase 6b. Save the returned ID.
4. Scan the ticket's endpoints (from Phase 1) for any new chained-ID vars that aren't yet in the env (e.g. ticket uses a `{{testInvoiceId}}` that the env doesn't have). Patch the env via `mcp__plugin_postman_postman__patchEnvironment` to add the missing keys with empty values.
5. **Also update local `newman-env.json`** to mirror — same dual-write rule as the setup command.

### 4b — Create the collection

`mcp__plugin_postman_postman__createCollection`:
- `name`: `<KEY>: <feature name>` (e.g. `PROJ-123: Order Submission Flow`)
- `description`: `Test collection for <KEY>. <ticket summary>`
- `workspace`: the workspace ID from Phase 0d
- `auth`: bearer using `{{accessToken}}`
- `variable`: keep this **minimal** — only the few collection-scoped vars not in the shared env. Most variable chaining for tickets goes through `pm.environment.set(...)` reading the shared env from 4a, not collection vars. Include here only if absolutely ticket-local (rare).

### 4c — Create the 4 folders

`mcp__plugin_postman_postman__createCollectionFolder` for `Setup`, `Happy Path - All Endpoints`, `Positive`, `Negative` — **in that order** so they appear in run order in the collection. Save IDs.

### 4d — Create requests

For each request, `mcp__plugin_postman_postman__createCollectionRequest`:
- `folderId`: the target folder
- `method` + `url` — host as `{{baseUrl}}`
- `header`: `Content-Type: application/json`; for API-key endpoints add `Authorization: ApiKey {{platformApiKey}}` and `auth.type = "noauth"`
- `body`: realistic data drawn from the ticket's specific values + OpenAPI schema. Use `{{$timestamp}}` for unique fields.
- `events` test script — rules below

**Happy Path - All Endpoints requests**: status-only assertion, valid body, no negatives. Iterate the audited flow list from Phase 3 **in order** so the resulting folder is top-to-bottom runnable.
- `name`: `<METHOD> <path>` exactly (no prefix, no status suffix)
- Test script:
  ```js
  pm.test('Status is 2xx', () => {
    pm.expect(pm.response.code).to.be.oneOf([200, 201, 202, 204]);
  });
  ```
- Path params resolve to `{{test<Entity>Id}}` chained variables populated by Setup. If Setup doesn't create the needed entity, leave the placeholder as-is and document it in the request description.
- Variable-set blocks (`pm.environment.set(...)`) are not added here — chaining stays in Setup. Happy Path is a read-mostly smoke pass; entity-creating endpoints in Happy Path produce throwaway records.

**Test script rules**:
- Top scope uses `let`, never `const` (Newman crashes on `const` re-declaration)
- Use `pm.environment.set(...)` for variable chaining
- Assert status first: `pm.test('Status 200', () => pm.response.to.have.status(200));`
- Login (in Setup):
  ```js
  pm.test('Status 200', () => pm.response.to.have.status(200));
  const tokens = pm.response.json().payload.data.tokens;
  pm.environment.set('accessToken', tokens.accessToken);
  pm.environment.set('refreshToken', tokens.refreshToken);
  ```
  Adjust JSON path using `TOKEN_JSON_PATH` from `.env`.
- Fetch ID (list endpoints in Setup):
  ```js
  pm.test('Fetch <Entity> 200', () => pm.response.to.have.status(200));
  var arr = pm.response.json().page.data;
  pm.test('<Entity> found', () => pm.expect(arr.length).to.be.above(0));
  pm.environment.set('test<Entity>Id', arr[0].id);
  ```
- Create entity (Setup):
  ```js
  pm.test('Create <Entity> 201', () => pm.response.to.have.status(201));
  const d = pm.response.json().payload?.data;
  if (d?.id) pm.environment.set('test<Entity>Id', d.id);
  ```
- Negative auth-matrix tests — use prebuilt token values:
  - Expired: a known-expired JWT (ask user or generate one well-past `exp`)
  - Tampered: take current `{{accessToken}}`, flip a char in payload section
  - Malformed: `Bearer not.a.real.jwt`
- Negative body validation → assert error code:
  ```js
  pm.test('Status 400', () => pm.response.to.have.status(400));
  pm.expect(pm.response.text()).to.include('<ERROR_CODE_FROM_SPEC>');
  ```

---

## Phase 5 — Export + run

### 5a — Export collection

```bash
POSTMAN_API_KEY="<from .claude/settings.local.json or ~/.claude/settings.json>"
curl -s -H "X-API-Key: $POSTMAN_API_KEY" \
  "https://api.getpostman.com/collections/<OWNER_ID>-<COLLECTION_ID>" \
  -o "<project-dir>/<key-lowercase>-collection.json"
```

E.g. `proj123-collection.json`.

### 5b — Run with Newman

Reuse the existing `newman-env.json` from the project. Override the report filename:

```bash
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
REPORT_FILE="newman-reports/<KEY>-report-$TIMESTAMP.html"
mkdir -p newman-reports

newman run "<key-lowercase>-collection.json" \
  --environment "newman-env.json" \
  --reporters cli,htmlextra \
  --reporter-htmlextra-export "$REPORT_FILE" \
  --reporter-htmlextra-title "<KEY> Test Report - $TIMESTAMP" \
  --delay-request 300 \
  --timeout-request 15000

open "$REPORT_FILE" 2>/dev/null || echo "Open: $REPORT_FILE"
```

---

## Phase 6 — Auto-debug + auto-fix

After Newman finishes (the FULL initial run completes — never use `--bail` on the initial run, the HTML report must capture every failure in one pass), do NOT immediately show the summary if there are failures. Iterate up to 3 times to auto-fix what is fixable. The goal is to leave only genuine bugs and environment issues for the human.

**Two hard rules for this phase:**

1. **No bail on the initial Newman run.** The first run must complete fully so the HTML report contains every failure. Auto-fix iterations happen AFTER the report is generated.
2. **Postman cloud and local JSON stay in sync.** Every time auto-fix patches a request via `mcp__plugin_postman_postman__updateCollectionRequest` (cloud update), the local `<KEY>-collection.json` MUST be re-exported immediately so Newman re-runs read the same state. See Phase 6d for the export command.

### 6a — Classify each failure

For each failed test, classify into one of these categories using the failure signal (status code, response body, error message):

| Signal | Category | Auto-fixable? |
|---|---|---|
| Response shape differs from spec (new/missing/renamed field) | Spec drift | Yes |
| Cascading 401s starting mid-suite | Stale `accessToken` | Yes |
| 404 on Get/Update/Delete with `{{testEntityId}}` | Stale chained ID | Yes |
| Setup Create-then-Fetch race condition | Setup timing | Yes |
| 409 conflict on a field that should be unique per run | Missing `{{$timestamp}}` | Yes |
| `SyntaxError: Identifier already declared` | `const` in test script top-scope | Yes |
| 5xx transient (intermittent, no clear error) | Transient infra | Yes (retry once) |
| `JSONError` + response is HTML | Load balancer error page | Yes (retry once after 30s) |
| Consistent 4xx/5xx with informative error message | Real bug | No -> mark as ticket candidate |
| Server unreachable, DNS failure, timeout on every request | Env issue | No -> halt and report |
| Account locked, expired password, missing permission | Account state | No -> ask user for fix |
| Passes some retries, fails others (no code change) | Flaky test | No -> log as flaky |

### 6b — Apply auto-fixes

For each fixable failure, apply the corresponding fix:

| Category | Action |
|---|---|
| Spec drift | Call `/qa-api-sync` -> wait for completion -> re-run affected tests |
| Stale `accessToken` | Re-run the Login request -> re-run all downstream failed tests |
| Stale chained ID | Re-run the Setup folder -> re-run dependent Positive/Negative tests |
| Setup timing | Add `pm.environment.set('delay', 1500)` or `setTimeout` before the Fetch, OR add Newman `--delay-request 1500` for next iteration |
| Missing `{{$timestamp}}` | Update the offending request body via `mcp__plugin_postman_postman__updateCollectionRequest`: insert `{{$timestamp}}` into the conflicting field (e.g. `"email": "test{{$timestamp}}@example.com"`) -> re-run |
| `const` at top scope | Patch the test script via `updateCollectionRequest`: replace top-scope `const ` with `let ` -> re-run |
| Transient 5xx | Re-run the single failing request once with a 2-second delay |
| HTML error page (JSONError) | Wait 30 seconds, re-run the failing request once |

### 6c — Iteration loop

```
iteration = 0
while iteration < 3 AND any-fixable-failures-remain:
  classify all current failures
  apply auto-fixes for fixable ones via Postman MCP (cloud update)
  re-export local <KEY>-collection.json (cloud -> file)
  re-run only the affected tests via Newman (no --bail)
  iteration++
```

Stop conditions:
- All tests pass -> proceed to Phase 7 (Summary)
- 3 iterations reached -> proceed to Phase 7 with remaining failures listed
- Only unfixable categories remain (real bug / env / account / flaky) -> proceed to Phase 7

### 6d — Re-export local collection JSON (cloud -> file sync)

After ANY `updateCollectionRequest` call in this phase, re-export the collection from Postman to the local file so Newman re-runs read the patched state:

```bash
POSTMAN_API_KEY="<from .claude/settings.local.json or ~/.claude/settings.json>"
curl -s -H "X-API-Key: $POSTMAN_API_KEY" \
  "https://api.getpostman.com/collections/<OWNER_ID>-<COLLECTION_ID>" \
  -o "<KEY-lowercase>-collection.json"
```

Without this re-export, Newman will keep running the OLD local JSON and your patches will have no effect on the next iteration. This is non-negotiable.

### 6e — Print what was auto-fixed (transparency, no silent changes)

Before moving on, print a clear log of every auto-fix applied:

```
Auto-fix log (iteration 1):
  ~ Spec drift detected for POST /api/v1/orders (new required field: regionId)
    -> Called /qa-api-sync, body updated, re-ran TC02-TC04 -> now passing
  ~ Stale accessToken on TC15-TC18 (cascade after long Setup)
    -> Re-ran Login, re-ran 4 affected tests -> now passing

Auto-fix log (iteration 2):
  ~ 409 conflict on TC02 - field "supplierCode" not using {{$timestamp}}
    -> Patched request body, re-ran TC02 -> now passing

Auto-fix log (iteration 3): (no fixable failures, stopping)

Final state after auto-fix:
  Passed:           <P+R> (P initial + R recovered via auto-fix)
  Still failing:    <N>
  Ticket candidates (real bugs):  <list>
  Halt-required (env/account):    <list>
  Flaky candidates:               <list>
```

### 6f — Hard rules for auto-fix

1. **Cap at 3 iterations.** Do not loop forever. If failures persist past 3 rounds, hand off to the human.
2. **Patch test setup ONLY, never assertions.** If an assertion expected 200 and the test got 201, the auto-fix is to verify the spec changed and update via `/qa-api-sync` -> not to silently change the assertion. Changing assertions hides real bugs.
3. **Never modify the system under test.** We only modify the way we call the API. The API itself is untouched.
4. **Always print what changed.** Silent fixes are forbidden. The user must be able to see "this field was patched", "this script was changed".
5. **Re-run only affected tests, not the whole suite.** Use Newman `--folder` or per-request invocation. Saves time and avoids state pollution.
6. **Do not retry consistent 4xx/5xx with informative errors.** Those are real bugs. Mark as candidates and stop trying to "fix" them.
7. **Never `--bail` any Newman invocation in this phase.** Both the initial full run AND per-iteration re-runs go to completion. A bailed run leaves an incomplete HTML report which confuses humans during Phase 8.
8. **Always re-export local JSON after every cloud patch.** Phase 6d step is mandatory after every `updateCollectionRequest` call. Postman cloud and local file must match before re-running Newman.

---

## Phase 7 — Summary

```
Ticket:        <KEY> — <summary>
Source:        Atlassian plugin / pasted content
Collection:    <KEY>: <feature name>  (ID: <id>)
Workspace:     <workspace name>
Setup items:   <N>
Positive:      <N>
Negative:      <N>
Total:         <N>

Run result (initial):    <P0> passed, <F0> failed
Auto-fix recovered:      <R>
Final run result:        <P> passed, <F> failed
Report:                  <path>/newman-reports/<KEY>-report-<ts>.html

Credentials added to .env: <list, or "none">

Remaining failures (after auto-fix):
  - <TC>: <category: real-bug / env / account / flaky> — <one-line reason>

Negative coverage:
  Total applicable rows: <N>
  Covered:               <N>
  Skipped (rate-limit):  <N>
  Coverage:              <pct>%
```

**Also write the full coverage report to `collection-run-issues/<KEY>-coverage-<YYYYMMDD-HHMMSS>.txt`** using the same format as `/qa-api-test-setup` (per-endpoint row-by-row breakdown, summary by phase Setup/Happy Path/Positive/Negative). Create `collection-run-issues/` if not present.

If any tests are still failing after Phase 6 (auto-fix) gave up, do NOT auto-raise tickets. The Jira logging gate is Phase 9 - it requires curl reproduction and dedupe checks first, per project memory.

---

## Phase 8 — Manual debugging tips (when auto-fix did not resolve)

For the full debugging playbook (triage tree, HTML report walkthrough, curl reproduction, HTTP failure patterns, Newman flags, decision tree, what-not-to-do), see `qa-api-test-setup.md` Phase 11. The same playbook applies to ticket runs - do not duplicate it here.

Ticket-specific things to check FIRST before falling back to the generic playbook:

1. **Setup folder ran cleanly** - complex tickets depend on a chain of Setup requests (Login -> Fetch IDs -> Create entities -> Build state). If any Setup request failed, every Positive/Negative test downstream is suspect. Always triage Setup failures first - they cascade.

2. **Credentials in the credential gate** - if Phase 2 collected new creds and you saved them to `.env`, verify they actually work. A test using `ADMIN_USER_EMAIL` will 401 silently if the password got typo'd during gating.

3. **State precondition mismatch** - if the ticket says "test approving a PENDING entity" but Setup built the entity in RECEIVED state, every Positive test that expects PENDING will fail. Re-read the Setup script for that specific entity.

4. **`pm.environment.set` vs `pm.collectionVariables.set` mixing** - ticket-scoped collections use `pm.environment.set` for chaining. If a test script accidentally uses `pm.collectionVariables.set`, the value goes to the wrong scope and downstream lookups return undefined.

5. **`{{$timestamp}}` placement** - tickets that re-run often (e.g. on every PR) need timestamp-uniqueness on unique fields (email, code, name). Missing it = 409 on re-run.

6. **Spec drift since collection was built** - if the ticket collection is older than your last `/qa-api-sync` run, the endpoints in it may have changed shape. Run `/qa-api-sync` first - it auto-refreshes affected ticket collections.

If none of the above applies, fall through to the generic playbook in `qa-api-test-setup.md` Phase 11.

---

## Phase 9 — Log confirmed bug to Jira (verification-gated)

Only enter this phase if tests are still failing AFTER:
- Phase 6 auto-fix has exhausted (3 iterations or no fixable categories left)
- Phase 8 manual debugging tips were reviewed by the user (or they confirmed they want to log)

This phase is the gate that respects the project rule: "Always reproduce manually with curl and read full response body before raising a ticket at any priority level." No ticket gets created without curl repro.

### 9a — Resolve Jira project key

Read `JIRA_PROJECT_KEY` from `.env`.
- If present -> use it.
- If missing -> extract from the current ticket key (e.g. `PROJ-123` -> `PROJ`), save to `.env` as `JIRA_PROJECT_KEY=PROJ`, confirm to user: `Saved JIRA_PROJECT_KEY=<KEY> to .env for future runs.`

### 9b — Show the user their options

Present the remaining real-bug-candidates from Phase 6's auto-fix log:

```
Tests still failing after auto-fix + debugging review:

  - TC02: Submit Order - expected 201, got 422 (real-bug-candidate)
  - TC05: Approve Order - expected 200, got 404 (real-bug-candidate)

Options:
  (a) Log all as Jira Bug tickets, linked to <KEY> via "relates to"
       I will curl-repro each one + check Jira for duplicates first.
  (b) Re-verify once more (re-run failing tests with --bail --verbose
       + run a fresh curl for each, then return to this gate)
  (c) Stop - I'll handle these manually.

Pick (a), (b), or (c):
```

Wait for user reply.

### 9c — If (b) re-verify

Re-run the failing tests (no `--bail` — let the run complete fully so the report is whole):
```bash
newman run <KEY>-collection.json -e newman-env.json \
  --folder "<failing folder>" --verbose
```

If you want to focus on ONE specific failing request (not the whole folder), pass `--folder "<exact request name>"` instead. Still no `--bail`.

For each failing test, capture the rendered request from the Newman verbose output and run it through curl. Show the user the curl command and full response for each.

After re-verification, return to the option prompt in 9b. If the user picks (a) this time, proceed.

### 9d — If (a) log as Jira bugs (verification gate runs here)

For each real-bug-candidate, run the following sequence in order. **Skip none.**

**Step 1 - Curl reproduction (REQUIRED by project rule):**
- Extract the exact rendered request from the Newman report (method, URL, headers, body)
- Run via curl, capture full response (status, headers, body) verbatim
- If curl returns a 2xx, the failure was NOT real - remove from candidates list, do not file a ticket
- If curl returns the same failure -> proceed

**Step 2 - Dedupe check:**
- Use `mcp__plugin_atlassian_atlassian__searchJiraIssuesUsingJql` with a JQL that searches the same project for similar bugs. Example:
  ```
  project = <JIRA_PROJECT_KEY> AND issuetype = Bug AND status != Done
  AND (summary ~ "<endpoint path>" OR description ~ "<error code from response>")
  ORDER BY created DESC
  ```
- If matches found -> show them to user:
  ```
  Possible duplicates for "TC02: Submit Order":
    - PROJ-100 (Open) - "Order creation fails with PRECONDITION_NOT_MET" (created 2026-04-21)
    - PROJ-105 (In Progress) - "Order endpoint returns wrong status code" (created 2026-04-30)

  Options:
    (1) Skip - this is already tracked in <key>
    (2) Add a comment to <key> with this run's repro
    (3) File a new bug anyway (you believe it's a different issue)
  ```
  Wait for user choice.
- If no matches -> proceed to step 3.

**Step 3 - Priority:**
Ask the user once per batch:
```
Default priority is Medium. Override? (Low/Medium/High/Critical or Enter to accept):
```

**Step 4 - Create the ticket via `mcp__plugin_atlassian_atlassian__createJiraIssue`:**

Use this format for `description`:

```
## Context
While testing <KEY> (<parent summary>), test case <TC> failed in the <folder> folder.

Test suite: `<KEY>: <feature name>` collection
Run timestamp: <ISO timestamp>
Report file: newman-reports/<KEY>-report-<ts>.html

## What we tested
- Endpoint: `<METHOD> <PATH>`
- Auth: `<Bearer / ApiKey / none>` (logged in as <test account email>)
- Request body:
  ```json
  <body sent, rendered variables>
  ```
- Query params: <list or "none">

## Expected behavior (from spec + AC)
- HTTP <expected status>
- Response shape: <list of expected fields, or full schema>
- Side effects: <state transitions, if any>

Source: OpenAPI spec at `<OPENAPI_SPEC from .env>` and AC item: <quoted from Jira ticket>

## Actual behavior
- HTTP <actual status>
- Response body (verbatim):
  ```json
  <full response from curl>
  ```
- Response headers:
  ```
  <key headers, especially X-Request-ID or similar>
  ```

## Reproduction (curl)
```bash
curl -X <METHOD> -v \
  -H "Content-Type: application/json" \
  -H "Authorization: <auth header, token redacted>" \
  -d '<request body>' \
  '<full URL with rendered variables>'
```

## Test run metadata
- Newman version: <output of `newman --version`>
- Reporter: <atlassian user from atlassianUserInfo>
- Suite collection ID: <collection UID>
- Environment: <BASE_URL from .env>
```

Other Jira fields:
- `project`: `JIRA_PROJECT_KEY` from `.env`
- `summary`: `[QA][<KEY>] <one-line failure summary>`
- `issuetype`: `Bug` (default) - or `Improvement` if the failure is about an ambiguous spec rather than wrong behavior
- `priority`: from step 3

**Step 5 - Link to parent ticket:**

Use `mcp__plugin_atlassian_atlassian__createIssueLink`:
- `inwardIssue`: new bug key
- `outwardIssue`: `<KEY>` (the parent ticket)
- `type`: `Relates` (or `is caused by` if more apt)

**Step 6 - Confirm to user:**

```
Created Bug: <NEW-KEY>
  Title:    [QA][<KEY>] <summary>
  Priority: <chosen>
  Linked to <KEY> via "relates to"
  URL: https://<your-atlassian-domain>/browse/<NEW-KEY>
```

Repeat steps 1-6 for each remaining real-bug-candidate.

### 9e — Phase 9 summary

After all bug-logging finishes, print:

```
Bug logging summary for <KEY>:
  Real bugs confirmed via curl: <N>
  Duplicates found, commented on existing: <list>
  New tickets created: <list with URLs>
  Skipped (curl failed to reproduce): <list - means it was env/flaky, not a real bug>

JIRA_PROJECT_KEY (saved to .env): <KEY>
```

---

## Rules

- **No partial builds.** If credentials/data are missing, stop and ask — never proceed with placeholders.
- **Standalone collection** — not a fork; carry own auth + variables.
- **Continuous TC numbering** across Positive → Negative.
- **Full auth matrix** on every primary endpoint in Negative (no/expired/tampered/malformed).
- **Setup scales to need** — don't add irrelevant prep, don't skip required state.
- **Never `const` at top scope** in test scripts.
- **One-off IDs do NOT go in `.env`** — only persistent creds/keys.
- **`POSTMAN_API_KEY` stays in `.claude/settings.local.json` (project-local, gitignored) or `~/.claude/settings.json` (global)**, never in project `.env`. Local wins if both set.
- **Never `--bail` the Newman run.** The initial run must complete to fill the HTML report with every failure. Auto-fix iterations also run to completion. `--bail` is allowed only in Phase 8's manual debugging tips for human-driven single-bug triage, never as default.
- **Cloud + file must stay in sync.** Any change the agent makes to a request via Postman MCP must be paired with a re-export of the local collection JSON (Phase 6d's curl command). Newman reads the file, so an out-of-sync file = patches ignored on the next run.
