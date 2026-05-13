---
name: qa-api-sync
description: "Detects OpenAPI spec drift since the last snapshot — diffs added/changed/removed endpoints, auto-updates the main collection's request bodies/params while preserving test scripts, refreshes affected ticket collections, archives removed endpoints as [DEPRECATED], and rewrites the single project-snapshot file to today's date with a new history entry at the top. Also runs in foundation-only scoped mode (no main collection yet, only ticket snapshots present) — iterates per-ticket snapshots and refreshes affected ticket collections."
---

# API Sync — Detect & Reconcile Spec Drift

You are syncing the project's Postman collection with the latest OpenAPI spec.

Invocation:
- Manual: `/qa-api-sync`
- Scheduled (runs even when Claude Code is closed): `/schedule daily at <user's local time> <user's timezone> /qa-api-sync` — derive the timezone from the project's CLAUDE.md "Today's date and timezone" section if present, else ask the user once.
- Auto-called from `/qa-test-ticket <KEY>` Phase 0 as a prerequisite

---

## Hard invariants (must hold throughout this run)

Re-read these before each diff bucket (ADDED/CHANGED/REMOVED) and before writing the snapshot. If a phase instruction ever seems to contradict one, the invariant wins.

 1. **NEVER renumber existing module folders.** New tags get appended as the next available `NN.`; existing modules keep their indices. Teams reference TC IDs by their current location — shuffling breaks references.
 2. **NEVER modify test scripts** on CHANGED endpoints. Update request body/params/headers only. Test scripts are hand-tuned and must be preserved.
 3. **NEVER delete REMOVED endpoints** — rename to `[DEPRECATED] <original name>`. Preserves history.
 4. **NEVER `--bail`** Newman if you run a verification pass after applying changes. Full run only.
 5. **ALWAYS hard-stop** if the fresh spec has 0 endpoints (don't trust an empty response — likely a transient fetch error). Do not touch the snapshot.
 6. **ALWAYS re-export** local `<project-slug>-collection.json` AND any touched `Jira Tickets/<KEY>/collection.json` files after Postman MCP writes. Newman reads local files.
 7. **ALWAYS apply the full applicable negative matrix** to ADDED endpoints — Happy Path entry + module Positive + all matrix-applicable Negative rows in the correct sub-folders. Never just a happy path.
 8. **ALWAYS patch the shared `<ProjectName> Environment`** for new chained-ID vars. Dual-write to `newman-env.json`. If the env is missing, warn and skip env updates (don't silently create one — user should know).
 9. **ALWAYS write today-dated snapshot** with the newest history entry **prepended** (top of array, not appended). The `endpoints` map is fully replaced with fresh state; the `history` array accumulates.
10. **Mode A** for ADDED positive entries (random + round-trip in module Positive; status + schema in Happy Path), **Mode B** for ADDED negatives (deterministic baseline + matrix mutation).
11. **`/qa-api-sync` does NOT renumber, re-audit, or refactor existing artifacts.** It applies the diff. Coverage gaps on pre-existing endpoints are `/qa-negative-audit`'s job.

If you catch yourself about to violate any of these, stop and explain to the user. Do not silently proceed.

---

## Phase 0 — Prereq checks

### 0a — `.env` must exist with required keys

```bash
test -f .env || (echo "MISSING_ENV" && exit 1)
```

Load `.env`. Required: `OPENAPI_SPEC`, `BASE_URL`. If missing → tell user to run `/qa-api-test-setup` first.

### 0b — Postman plugin connected

`mcp__plugin_postman_postman__getAuthenticatedUser`. If fails → stop, point to `/qa-api-test-setup` Phase 0a.

### 0c — Detect project state and select mode

Sync runs in one of two modes depending on which snapshots exist:

```bash
ls project-snapshot-*.json 2>/dev/null | sort -r
ls "Jira Tickets/*/snapshot-*.json" 2>/dev/null
```

**Mode A — Full sync (project snapshot exists)**:
- Pick the most recent `project-snapshot-*.json` as the baseline.
- Run normal flow: project-level diff + ticket cross-reference + main collection updates + ticket collection refreshes.
- If multiple project snapshots exist (shouldn't happen normally), warn user about the extras and use the most recent.

**Mode B — Foundation-only scoped sync (no project snapshot, but ticket snapshots exist)**:
- Iterate each `Jira Tickets/<KEY>/snapshot-*.json` as a per-ticket baseline.
- Diff each against the fresh spec scoped to its tracked endpoints.
- Refresh affected ticket collections only — no main collection updates (none exists).
- Update each refreshed ticket's snapshot to today's date.
- Phase 4a (ADDED to main), 4b (CHANGED in main), 4c (REMOVED from main) all skip in this mode. Phase 4d (ticket refresh) is the only active branch.
- Phase 6 writes per-ticket snapshots only — no project snapshot.

**Mode C — Nothing to sync against (no project snapshot, no ticket snapshots)**:
- Tell user: `No baseline snapshots found. Run /qa-api-test-setup first (full setup) or run /qa-test-ticket to start with an inline foundation.` Stop.

Remember which mode applies; downstream phases branch on it.

---

## Phase 1 — Fetch fresh spec

### 1a — Load the spec (with Scalar / HTML auto-discovery)

Read `OPENAPI_SPEC` from `.env`. Resolve it to a parseable OpenAPI document using the same fallback chain as `/qa-api-test-setup` Phase 2a:

1. **Local file path** → read with `Read`.
2. **URL returning JSON or YAML** → `curl -s` and parse.
3. **URL returning HTML** (Scalar / Swagger UI / Redoc reference page) → fetch the HTML and try, in order:
   - Scalar embed: `<script id="api-reference" data-url="...">` → fetch that URL.
   - Inline Scalar spec: `<script id="api-reference" type="application/json">…</script>` → parse inline.
   - Redoc embed: `<redoc spec-url="..."></redoc>` → fetch that URL.
   - Swagger UI config (`url:` / `urls:[{url:}]`) inside `<script>` blocks → fetch.
   - Conventional paths on the URL's origin: `/openapi.json`, `/openapi.yaml`, `/swagger.json`, `/api/openapi.json`, `/api-docs`, `/v3/api-docs`. Stop at the first one that parses.
4. **If all auto-discovery fails** → ask the user once for the raw spec URL or local path, then retry from step 1.

Validate the loaded document parses as OpenAPI (has `openapi:`/`swagger:` and a `paths:` object). If it fails to parse → stop and show the error; tell user the spec source is broken.

### 1b — Mandatory endpoint count print (with delta vs snapshot)

Count every `(method, path)` pair under `paths.*` (only the HTTP verb keys: `get`, `post`, `put`, `patch`, `delete`, `head`, `options`, `trace`). Also read `meta.currentTotalEndpoints` and `meta.lastSyncedAt` from the snapshot file.

Print this block — it is mandatory and cannot be skipped:

```
✓ Fresh spec loaded
  Source:          <original URL or file path>
  Resolved spec:   <derived JSON URL if Scalar/HTML, else "same as source">
  Total endpoints today:      <N>
  Total endpoints in snapshot (<snapshot date>): <M>
  Delta vs snapshot:          <signed N - M>   (real diff buckets computed in Phase 2c)
```

**Hard stop if `Total endpoints today = 0`** — print: `Spec parsed but no endpoints found. Aborting — the source is likely wrong or temporarily empty.` and exit without touching the snapshot.

---

## Phase 2 — Compute diff

### 2a — Hash every endpoint in the fresh spec

For each `(method, path)` in the spec, compute a SHA-256 hash of a canonical representation:
```
<method>|<path>|<sorted required body fields>|<sorted required query params>|<auth requirement>|<response 2xx schema keys>
```

Build `freshEndpoints: { "POST /api/v1/orders": "sha256:..." , ... }`.

### 2b — Load snapshot endpoints

Read the snapshot file. Pull out `endpoints: { ... }` from it.

### 2c — Compute 3-bucket diff

Compare the two endpoint maps:

- **Added**: keys in `freshEndpoints` not in snapshot endpoints
- **Removed**: keys in snapshot endpoints not in `freshEndpoints`
- **Changed**: keys in both but with different hashes

For **changed** endpoints, also compute the human-readable diff:
- What body fields were added / removed / made required / made optional
- What query params changed
- What response shape changed
- Whether auth requirement flipped

### 2d — Cross-reference ticket collections

Find ticket collections by scanning `Jira Tickets/*/` subfolders locally:

```bash
ls "Jira Tickets/*/collection.json" 2>/dev/null
```

Each `Jira Tickets/<KEY>/collection.json` corresponds to a Postman collection named `<KEY>: <feature>` in the workspace. Read the local JSON (faster than fetching via MCP) or call `mcp__plugin_postman_postman__getCollection` per collection if the local file is stale.

For each ticket collection, build a list of `(method, path)` it uses. Cross-reference with the **Changed** and **Removed** buckets from Phase 2c (Mode A) OR with the diff scoped to that ticket's snapshot (Mode B):

- If a ticket collection uses a **Changed** endpoint → flag for auto-update
- If a ticket collection uses a **Removed** endpoint → flag with warning

**In Mode B (foundation-only)**: instead of using a project-level diff, diff each `Jira Tickets/<KEY>/snapshot-*.json` directly against the fresh spec. Only the endpoints THIS ticket tracked are compared. Each ticket's drift bucket may be different.

---

## Phase 3 — Report

Print the diff clearly. If **no drift detected** → print:
```
✓ No spec drift since <last-sync-date>. Collection is in sync. Nothing to update.
```
Then update the snapshot file's timestamp (rename to today's date) and exit.

If drift detected, print:
```
API spec drift since <last-sync-date>:

+ ADDED (3):
  POST /api/v1/orders/cancel
  GET  /api/v1/orders/{id}/audit
  POST /api/v1/suppliers/bulk-import

~ CHANGED (2):
  POST /api/v1/orders
    Before: required = [supplierId, items]
    After:  required = [supplierId, items, regionId]
    Diff:   + regionId (required, string)

  PUT  /api/v1/products
    Before: response 200 = { id, name, sku }
    After:  response 200 = { id, name, sku, category }
    Diff:   + category in response

- REMOVED (1):
  DELETE /api/v1/orders/{id}

⚠ Ticket collections affected:
  PROJ-123: Order Submission Flow
    - Uses POST /api/v1/orders (CHANGED) — will auto-update request body
  PROJ-456: Product Edit
    - Uses PUT /api/v1/products (CHANGED) — will auto-update
    - Uses DELETE /api/v1/orders/{id} (REMOVED) — will be archived
```

Ask:
> Proceed with applying these updates? (y/n)

If `n` → exit without changes. If `y` → continue.

---

## Phase 4 — Apply changes

**Mode gate**: in Mode A (full sync), phases 4a, 4b, 4c, 4d all run. In Mode B (foundation-only scoped sync), **only 4d runs** — there's no main collection to add to (4a), change (4b), or remove from (4c). The diff per ticket is computed from each ticket's own snapshot in Phase 2d Mode B branch.

### 4a — ADDED endpoints

For each new endpoint, create requests in the **main project collection** in two places:

**1. The `Happy Path - All Endpoints` top-level folder**, with one valid-data request:
- `folderId`: the `Happy Path - All Endpoints` folder ID (look it up via `mcp__plugin_postman_postman__getCollection`)
- `name`: `<NN>. <op.summary>` where `NN` is the next available Happy Path index (sequential across the existing Happy Path folder, zero-padded). `<op.summary>` resolved per CLAUDE.md § Happy Path folder fallback chain. Example: `42. Cancel Order` (next available index in a Happy Path with 41 existing entries).
- Body: built via a pre-request script using **Mode A** from `.claude/commands/qa-negative-matrix.md` § Body data sources (random constraint-aware values for user-supplied fields, chained env vars for resource references, skip readOnly + sensitive fields). Stash in `{{__sentBody}}` for the assertion phase.
- Tests: **status check + schema validation** (Happy Path smoke profile — no per-field round-trip, that's the per-module Positive entry's job).
  ```js
  pm.test('Status is 2xx', () => pm.expect(pm.response.code).to.be.oneOf([200, 201, 202, 204]));
  let schema = { /* inlined OpenAPI 2xx schema */ };
  pm.test('Response matches schema', () => pm.response.to.have.jsonSchema(schema));
  ```
- If the added endpoint is a login or creates an entity, include the same variable-set block (`pm.collectionVariables.set('accessToken'/'test<Entity>Id', ...)`) so downstream Happy Path requests can chain.
- **Position in the folder** (sync-mini-audit): Do **not** append to the bottom blindly. Run a scoped dependency audit on just this endpoint vs. the existing Happy Path entries (same four signals as `/qa-api-test-setup` Phase 2d). Find its correct dependency slot and insert there using Postman's request-ordering API (`mcp__plugin_postman_postman__transferCollectionRequests` if available, otherwise reorder via `putCollection`). If the new endpoint produces a variable that an existing endpoint needs, insert before that consumer. If it consumes a variable from an existing endpoint, insert after that producer. If it's a true orphan (no producer found anywhere), append at the end and warn the user in the Phase 3 report.

**Module-folder resolution**: match the endpoint's OpenAPI `tag` against existing module folders by name (ignore the `NN.` prefix when matching).
- If the tag matches an existing module → use that module's folder, regardless of where in the `NN.` ordering it sits. **Do not renumber existing modules** — the team may already be referencing TC IDs by their current module index, and shuffling indices mid-project is jarring.
- If the tag is new (no matching module exists) → create a new module folder named `<NN>. <Tag>` where `NN` is the **next available index** after the current highest. E.g. if existing modules are `00. Auth`, `01. Users`, `02. Orders`, a new `Reports` tag becomes `03. Reports`. Then create its `Positive` and `Negative` sub-folders.

**Postman environment sync**: if any ADDED endpoint qualifies as a creatable (POST + 2xx returns an ID — see `/qa-api-test-setup` Phase 6b detection rule), the corresponding `test<Entity>Id` env var must exist in the shared `<ProjectName> Environment`. To wire this up:

1. List existing environments via `mcp__plugin_postman_postman__getEnvironments`, find the one named `<ProjectName> Environment`. Save its ID. If not found, warn the user (`Project env missing — re-run /qa-api-test-setup or create one manually`) and skip env updates.
2. Fetch the current env via `mcp__plugin_postman_postman__getEnvironment`.
3. For each new chained-ID var derived from the ADDED endpoints (e.g. `POST /widgets` → `testWidgetId`), check if it already exists in the env. If not, append it with empty value.
4. Patch the env via `mcp__plugin_postman_postman__patchEnvironment` — pass only the **new** values; existing ones stay untouched.
5. **Also update `<project-dir>/newman-env.json`** locally: read the current file, append the same new vars, write back. Both stay in sync.

If any ADDED endpoint uses a NEW path-param like `{tenantId}` whose producer is in the same batch (matrix detected the dep), the chained var is recorded. If the path-param has no producer (orphan), it gets a placeholder `test<Entity>Id` env entry anyway so the user can wire it manually.

**2. The appropriate module folder**, with **the full applicable negative matrix** from `.claude/commands/qa-negative-matrix.md`. Walk every row, check the condition against the new endpoint's spec, and generate the test in its target Negative sub-folder (`Auth Failures` / `Validation Failures` / `Resource Errors` / `Security Probes`) or mark `n/a`. Also add the **Positive happy path** in `<module>PositiveFolderId` with cross-cutting assertions (response time, schema validation, sensitive-data leak).

If the module folder doesn't yet have the 4-way Negative sub-folders (older project pre-matrix), create them first via `createCollectionFolder`. Don't move existing flat tests — leave them; warn the user to run `/qa-negative-audit --reorganize` if they want them migrated.

Same Setup-fixture rule as `/qa-api-test-setup` Phase 5c: rows like `res-conflict`, `res-wrong-state`, `auth-wrong-role` need fixture state — add the necessary Setup steps if not already present in the module's Positive folder.

Test script style follows `/qa-api-test-setup` Phase 5c rules (let-only, status-first assertion, `pm.collectionVariables.set` for chaining in the main collection). The **Positive happy-path entry in the module folder** uses full **Mode A round-trip** (pre-request random gen → post-response round-trip per field + schema). The **Negative tests** use **Mode B** (valid baseline + matrix-row mutation). Matrix-specific scripts come from the matrix doc.

Use `mcp__plugin_postman_postman__createCollectionRequest` for all folders.

### 4b — CHANGED endpoints

For each changed endpoint:

1. Find the existing request(s) in the main collection at that `(method, path)` — there will typically be multiple matches across folders (one in `Happy Path - All Endpoints`, plus the module's `Positive` happy path, plus one or more `Negative` variants).
2. For each matching request, update the request **body and params** to match the new schema. **Do not touch the test scripts** — those are hand-tuned and must be preserved.
3. Use `mcp__plugin_postman_postman__updateCollectionRequest`.
4. Print before/after for the user, noting which folders were touched.

### 4c — REMOVED endpoints

For each removed endpoint:

1. Find requests in the main collection at that `(method, path)` — including the entry in `Happy Path - All Endpoints` AND all module-folder variants.
2. Rename them to `[DEPRECATED] <original name>` via `updateCollectionRequest` (so the Happy Path entry becomes `[DEPRECATED] <NN>. <op.summary>`, e.g. `[DEPRECATED] 17. Cancel Order`).

### 4d — Ticket-collection refresh

For each ticket collection flagged in Phase 2d:

- For **changed** endpoints used by the ticket collection → update its request body/params in **all** folders where it appears (`Happy Path - All Endpoints`, `Positive`, `Negative`). Same rule as 4b: body yes, test scripts no.
- For **removed** endpoints → rename to `[DEPRECATED] <original name>` in every folder where they appear.

After applying the patch, **re-export the local ticket collection JSON** so Newman picks up the changes:
```bash
curl -s -H "X-API-Key: $POSTMAN_API_KEY" \
  "https://api.getpostman.com/collections/<OWNER_ID>-<TICKET_COLLECTION_ID>" \
  -o "Jira Tickets/<KEY>/collection.json"
```

Also **update the per-ticket snapshot** to today's date — the ticket has been brought back in sync with the spec:
- Rename `Jira Tickets/<KEY>/snapshot-<old-date>.json` → `Jira Tickets/<KEY>/snapshot-<today>.json`
- Recompute endpoint hashes against the fresh spec
- Save

Print which ticket collections were touched.

---

## Phase 5 — Re-export main collection (Mode A only)

In **Mode B (foundation-only)**, skip Phase 5 entirely — there's no main collection to re-export. Per-ticket re-exports are handled in 4d above.

```bash
POSTMAN_API_KEY="<from .claude/settings.local.json or ~/.claude/settings.json>"
curl -s -H "X-API-Key: $POSTMAN_API_KEY" \
  "https://api.getpostman.com/collections/<OWNER_ID>-<COLLECTION_ID>" \
  -o "<project-slug>-collection.json"
```

Ticket-collection re-exports are already handled inline in Phase 4d to `Jira Tickets/<KEY>/collection.json`.

---

## Phase 6 — Update the snapshot file

### 6a — Determine the new filename

Today's date: `YYYY-MM-DD`. New filename: `project-snapshot-<today>.json`.

### 6b — Rename the existing file

If the existing snapshot filename's date is **not today** → rename it to the new filename.
If it **is today** (you ran sync twice today) → keep the same filename, overwrite in place.

```bash
# only if old date != today
mv project-snapshot-<old-date>.json project-snapshot-<today>.json
```

### 6c — Update contents

Open the file. Update structure:

```json
{
  "meta": {
    "specSource": "<URL or file path from .env>",
    "currentTotalEndpoints": <N>,
    "lastSyncedAt": "<ISO timestamp>"
  },
  "history": [
    {
      "syncedAt": "<ISO timestamp of THIS sync>",
      "type": "sync",
      "added":    <N>,
      "changed":  <N>,
      "removed":  <N>,
      "addedList":   ["POST /api/v1/orders/cancel", ...],
      "changedList": [
        { "endpoint": "POST /api/v1/orders", "diff": "+ regionId (required, string)" },
        { "endpoint": "PUT /api/v1/products", "diff": "+ category in response" }
      ],
      "removedList": ["DELETE /api/v1/orders/{id}"],
      "ticketCollectionsTouched": ["PROJ-123", "PROJ-456"]
    },
    <... older history entries pushed below ...>
  ],
  "endpoints": {
    "POST /api/v1/auth/login":   "sha256:...",
    "POST /api/v1/orders":          "sha256:<new-hash>",
    "POST /api/v1/orders/cancel":   "sha256:<new>"
  }
}
```

Rules:
- **Newest history entry goes at index 0** (top of array) so the file opens to the latest sync
- `endpoints` map is fully replaced with the fresh state — removed endpoints disappear from the map (their history stays in the `history` array)
- If `history` exceeds 50 entries, trim the oldest

---

## Phase 7 — Summary

```
API Sync Summary — <today>

Spec source:        <URL or file>
Last sync:          <previous date>
Endpoints in spec:  <N> (was <prev-N>)

Changes:
  + Added:   <N>
  ~ Changed: <N>
  - Removed: <N>

Main collection:    updated (<N> requests added, <N> updated, <N> archived)
Ticket collections touched:
  - PROJ-123 (1 endpoint updated)
  - PROJ-456 (1 updated, 1 archived)

Snapshot file:      project-snapshot-<today>.json
Re-exported JSON:   <project-slug>-collection.json + <ticket>-collection.json files

Next step:
  cd <project-dir> && ./run-tests.sh
```

If anything failed mid-sync (e.g. Postman API rejected an update), report the partial state honestly — don't claim success.

**Coverage suggestion (non-blocking):** If any endpoints were ADDED this sync, append a line to the summary:

```
Coverage suggestion:
  This sync added <N> endpoints with their full negative matrix.
  To verify project-wide negative coverage is still on track,
  run /qa-negative-audit (report-only mode is safe and fast).
```

Do NOT auto-invoke `/qa-negative-audit` from sync — it's a separate, on-demand command. The suggestion is a pointer the user follows when they want a coverage health check.

**Also write a coverage delta report to `collection-run-issues/sync-<today>-coverage.txt`** if any endpoints were ADDED, CHANGED, or REMOVED. Format:

```
Sync coverage delta — <today>
Generated by: /qa-api-sync
Spec source:  <URL or file>

Changes in this sync:
  + Added:   <N>
  ~ Changed: <N>
  - Removed: <N>

Per-endpoint matrix coverage for ADDED endpoints:
  POST /api/v1/orders/cancel   <N>/<M> rows covered
    ✓ auth-no-token        TC15
    ...
    ⊘ sec-rate-limit       skipped

Project total coverage (after sync):
  Total applicable rows: <N>
  Covered:               <N>
  Skipped:               <N>
  Coverage:              <pct>%

Suggestion: run /qa-negative-audit --report-only for the full project view.
```

Lives in `collection-run-issues/` (already gitignored). Create the directory if not present. Skip writing the file if there were no changes (clean sync — nothing to report).
