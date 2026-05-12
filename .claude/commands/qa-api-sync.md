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

Read `OPENAPI_SPEC` from `.env`. It is either a URL or a local file path.

- **URL** → fetch with `curl -s` and parse JSON/YAML
- **File path** → read with `Read` tool

Validate it's a parseable OpenAPI document. If it fails to parse → stop and tell user the spec source is broken; show the error.

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

For each new endpoint, create a request in the **main project collection** in the appropriate folder (matched by the endpoint's OpenAPI `tag`).

Use the standard test set:

| Test type | Always or conditional | Purpose |
|---|---|---|
| **Happy path** (2xx) | Always | Baseline functional check |
| **No auth → 401** | Always (unless `security: []` on endpoint) | Auth wall |
| **Missing required field → 400** | If endpoint has a body with required fields | Validation |
| **Non-existent ID → 404** | If endpoint has a path param like `{id}` | Not-found |
| **Wrong-state → 422** | If spec mentions state transitions | Lifecycle |
| **Conflict → 409** | If endpoint creates uniqueness-constrained resources | Duplicate |

Test script style follows `/qa-api-test-setup` Phase 5c rules (let-only, status-first assertion, `{{$timestamp}}` for unique fields, `pm.collectionVariables.set` for chaining in the main collection).

Use `mcp__plugin_postman_postman__createCollectionRequest`.

### 4b — CHANGED endpoints

For each changed endpoint:

1. Find the existing request(s) in the main collection at that `(method, path)` — could be more than one (happy + negative variants)
2. For each matching request, update the request **body and params** to match the new schema. **Do not touch the test scripts** — those are hand-tuned and must be preserved.
3. Use `mcp__plugin_postman_postman__updateCollectionRequest`.
4. Print before/after for the user.

### 4c — REMOVED endpoints

For each removed endpoint:

1. Find requests in the main collection at that `(method, path)`
2. Rename them to `[DEPRECATED] <original name>` via `updateCollectionRequest`
3. Do **not** delete — preserves history in case the endpoint comes back or you need to reference the old test data

### 4d — Ticket-collection refresh

For each ticket collection flagged in Phase 2d:

- For **changed** endpoints used by the ticket collection → update its request body/params (same rule as 4b: body yes, test scripts no)
- For **removed** endpoints → rename to `[DEPRECATED]`

Print which ticket collections were touched.

---

## Phase 5 — Re-export main collection

```bash
POSTMAN_API_KEY="<from ~/.claude/settings.json>"
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
