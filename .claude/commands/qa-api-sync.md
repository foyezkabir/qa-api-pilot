---
name: qa-api-sync
description: "Detects OpenAPI spec drift since the last snapshot — diffs added/changed/removed endpoints, auto-updates the main collection's request bodies/params while preserving test scripts, refreshes affected ticket collections, archives removed endpoints as [DEPRECATED], and rewrites the single api-snapshot file to today's date with a new history entry at the top."
---

# API Sync — Detect & Reconcile Spec Drift

You are syncing the project's Postman collection with the latest OpenAPI spec. The spec changes over time as developers ship new endpoints, modify existing ones, or deprecate them. This command detects that drift and updates the collection automatically.

This command only runs in a project already set up via `/qa-api-test-setup`. It requires `.env`, an existing main collection, and a snapshot file in the project root.

Invocation:
- Manual: `/qa-api-sync`
- Scheduled (runs even when Claude Code is closed): `/schedule daily at 2:30 PM BDT /qa-api-sync`
- Auto-called from `/qa-test-ticket <KEY>` Phase 0 as a prerequisite

---

## Phase 0 — Prereq checks

### 0a — `.env` must exist with required keys

```bash
test -f .env || (echo "MISSING_ENV" && exit 1)
```

Load `.env`. Required: `OPENAPI_SPEC`, `BASE_URL`. If missing → tell user to run `/qa-api-test-setup` first.

### 0b — Postman plugin connected

`mcp__plugin_postman_postman__getAuthenticatedUser`. If fails → stop, point to `/qa-api-test-setup` Phase 0a.

### 0c — Find the snapshot file

```bash
ls api-snapshot-*.json 2>/dev/null | sort -r
```

- **No file found** → tell user: `No snapshot file. Run /qa-api-test-setup first to establish a baseline.` Stop.
- **Exactly one** → use it.
- **Multiple** → use the one with the most recent date in its filename. Warn user about the extras:
  > Found multiple snapshot files: <list>. Using the most recent (<filename>). Consider deleting the older ones.

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

Count every `(method, path)` pair under `paths.*` (only the HTTP verb keys: `get`, `post`, `put`, `patch`, `delete`, `head`, `options`, `trace`). Also read `meta.currentTotalEndpoints` and `meta.lastSyncedAt` from the snapshot file (Phase 2b loads the snapshot — pull these values forward here, or read the snapshot file once now).

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

The exact `+added / -removed / changed` bucket counts come from Phase 2c. The delta line above is just `today - snapshot`, useful as an early signal.

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

List all collections in the project's workspace via `mcp__plugin_postman_postman__getCollections`. Find ones named like `<KEY>: <feature>` (e.g. `PROJ-123: Order Submission Flow`).

For each ticket collection, fetch its requests (`mcp__plugin_postman_postman__getCollection`). Build a list of `(method, path)` it uses. Cross-reference with the **Changed** and **Removed** buckets:

- If a ticket collection uses a **Changed** endpoint → flag for auto-update
- If a ticket collection uses a **Removed** endpoint → flag with warning

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

### 4a — ADDED endpoints

For each new endpoint, create requests in the **main project collection** in two places:

**1. The `Happy Path - All Endpoints` top-level folder**, with one valid-data request:
- `folderId`: the `Happy Path - All Endpoints` folder ID (look it up via `mcp__plugin_postman_postman__getCollection`)
- `name`: `<METHOD> <path>` exactly (e.g. `POST /api/v1/orders/cancel`)
- Body: valid data from the spec. Use `{{$timestamp}}` for unique fields.
- Test: status-only — `pm.expect(pm.response.code).to.be.oneOf([200, 201, 202, 204]);`
- If the added endpoint is a login or creates an entity, include the same variable-set block (`pm.collectionVariables.set('accessToken'/'test<Entity>Id', ...)`) so downstream Happy Path requests can chain.
- **Position in the folder** (sync-mini-audit): Do **not** append to the bottom blindly. Run a scoped dependency audit on just this endpoint vs. the existing Happy Path entries (same four signals as `/qa-api-test-setup` Phase 2d). Find its correct dependency slot and insert there using Postman's request-ordering API (`mcp__plugin_postman_postman__transferCollectionRequests` if available, otherwise reorder via `putCollection`). If the new endpoint produces a variable that an existing endpoint needs, insert before that consumer. If it consumes a variable from an existing endpoint, insert after that producer. If it's a true orphan (no producer found anywhere), append at the end and warn the user in the Phase 3 report.

**Module-folder resolution**: match the endpoint's OpenAPI `tag` against existing module folders by name (ignore the `NN.` prefix when matching).
- If the tag matches an existing module → use that module's folder, regardless of where in the `NN.` ordering it sits. **Do not renumber existing modules** — the team may already be referencing TC IDs by their current module index, and shuffling indices mid-project is jarring.
- If the tag is new (no matching module exists) → create a new module folder named `<NN>. <Tag>` where `NN` is the **next available index** after the current highest. E.g. if existing modules are `00. Auth`, `01. Users`, `02. Orders`, a new `Reports` tag becomes `03. Reports`. Then create its `Positive` and `Negative` sub-folders.

**2. The appropriate module folder**, with the standard module-level test set:

| Test type | Always or conditional | Purpose |
|---|---|---|
| **Happy path** (2xx) | Always | Baseline functional check |
| **No auth → 401** | Always (unless `security: []` on endpoint) | Auth wall |
| **Missing required field → 400** | If endpoint has a body with required fields | Validation |
| **Non-existent ID → 404** | If endpoint has a path param like `{id}` | Not-found |
| **Wrong-state → 422** | If spec mentions state transitions | Lifecycle |
| **Conflict → 409** | If endpoint creates uniqueness-constrained resources | Duplicate |

Test script style follows `/qa-api-test-setup` Phase 5c rules (let-only, status-first assertion, `{{$timestamp}}` for unique fields, `pm.collectionVariables.set` for chaining in the main collection).

Use `mcp__plugin_postman_postman__createCollectionRequest` for both folders.

### 4b — CHANGED endpoints

For each changed endpoint:

1. Find the existing request(s) in the main collection at that `(method, path)` — there will typically be multiple matches across folders (one in `Happy Path - All Endpoints`, plus the module's `Positive` happy path, plus one or more `Negative` variants).
2. For each matching request, update the request **body and params** to match the new schema. **Do not touch the test scripts** — those are hand-tuned and must be preserved.
3. Use `mcp__plugin_postman_postman__updateCollectionRequest`.
4. Print before/after for the user, noting which folders were touched.

### 4c — REMOVED endpoints

For each removed endpoint:

1. Find requests in the main collection at that `(method, path)` — including the entry in `Happy Path - All Endpoints` AND all module-folder variants.
2. Rename them to `[DEPRECATED] <original name>` via `updateCollectionRequest` (so the Happy Path entry becomes `[DEPRECATED] <METHOD> <path>`).
3. Do **not** delete — preserves history in case the endpoint comes back or you need to reference the old test data.

### 4d — Ticket-collection refresh

For each ticket collection flagged in Phase 2d:

- For **changed** endpoints used by the ticket collection → update its request body/params in **all** folders where it appears (`Happy Path - All Endpoints`, `Positive`, `Negative`). Same rule as 4b: body yes, test scripts no.
- For **removed** endpoints → rename to `[DEPRECATED] <original name>` in every folder where they appear.

Print which ticket collections were touched.

---

## Phase 5 — Re-export main collection

```bash
POSTMAN_API_KEY="<from .claude/settings.local.json or ~/.claude/settings.json>"
curl -s -H "X-API-Key: $POSTMAN_API_KEY" \
  "https://api.getpostman.com/collections/<OWNER_ID>-<COLLECTION_ID>" \
  -o "<project-slug>-collection.json"
```

Also re-export any ticket collections that were touched in 4d → overwrites their local JSON.

---

## Phase 6 — Update the snapshot file

### 6a — Determine the new filename

Today's date: `YYYY-MM-DD`. New filename: `api-snapshot-<today>.json`.

### 6b — Rename the existing file

If the existing snapshot filename's date is **not today** → rename it to the new filename.
If it **is today** (you ran sync twice today) → keep the same filename, overwrite in place.

```bash
# only if old date != today
mv api-snapshot-<old-date>.json api-snapshot-<today>.json
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
- If `history` exceeds 50 entries, trim the oldest (you don't need a year of daily syncs)

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

Snapshot file:      api-snapshot-<today>.json
Re-exported JSON:   <project-slug>-collection.json + <ticket>-collection.json files

Next step:
  cd <project-dir> && ./run-tests.sh
```

If anything failed mid-sync (e.g. Postman API rejected an update), report the partial state honestly — don't claim success.

---

## Rules

- **One snapshot file only**, renamed to today's date each successful sync
- **Newest history entry at the top** of the `history` array
- **Body/params update on CHANGED endpoints, test scripts preserved**
- **REMOVED endpoints are archived (renamed `[DEPRECATED] ...`), never deleted**
- **Ticket collections auto-update** for any changed/removed endpoints they reference
- **History array trimmed at 50 entries** to keep file size sane
- **If no drift detected**, still rename the snapshot file to today's date so `lastSync` is current
- **Never edit test scripts in CHANGED updates** — only body/url/params/headers
