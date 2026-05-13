---
name: qa-negative-audit
description: "Coverage audit for negative tests. Walks every endpoint, evaluates the negative matrix (.claude/commands/qa-negative-matrix.md), reports gaps per endpoint, and offers to fill them. Module-wise batched generation with a checkpoint file so an interrupted audit resumes where it left off. Run on-demand whenever the project's negative coverage needs a top-up; distinct from /qa-api-sync (which is about spec drift)."
---

# Negative Coverage Audit

You are running a **coverage audit** against the project's main Postman collection. The audit walks every endpoint in the OpenAPI spec, evaluates each row of the negative matrix (`.claude/commands/qa-negative-matrix.md`), and reports — and optionally fills — gaps.

This command does NOT detect spec drift (that's `/qa-api-sync`). It assumes the spec and collection are in sync at the request level and asks: **do we have full negative coverage on each endpoint?**

When to run:
- After `/qa-api-test-setup` if the initial build was interrupted before negatives finished.
- After `/qa-api-sync` flags many ADDED endpoints and you want to verify their negative coverage filled correctly.
- Periodically (weekly, monthly) to catch coverage drift — when teammates patch collections by hand and accidentally remove tests.
- Anytime you see a sync output line `Run /qa-negative-audit to close coverage gaps?`.

---

## Hard invariants (must hold throughout this run)

Re-read these before each module batch and at the start of every Phase 5 fill iteration. If a phase instruction ever seems to contradict one, the invariant wins.

 1. **NEVER modify or delete existing tests.** The audit is additive-only. If an existing test looks wrong, flag it in the report — let the user decide. Silent edits hide regressions.
 2. **NEVER renumber existing TC numbers.** New tests get the next available TC number after the existing maximum. Renumbering scrambles references everyone has memorized.
 3. **NEVER touch the spec.** This command never modifies the OpenAPI source.
 4. **NEVER generate rate-limit tests** if `INCLUDE_RATE_LIMIT_TESTS=false` in `.env`, unless `--include-rate-limit` flag is explicitly passed for this run. Mark as `skipped (env flag)` in the report so the gap stays visible.
 5. **NEVER overwrite a cross-cutting assertion block** that already exists in a test script. Detect existing blocks and skip — idempotent.
 6. **NEVER `--bail`** if you run a Newman verification pass after generation. Full run only.
 7. **ALWAYS walk every matrix row** per endpoint. Each row resolves to one of: `✓ covered (TC<NN>)`, `✗ missing`, `n/a (condition not met)`, `skipped (env flag)`. No mental skipping.
 8. **ALWAYS write checkpoint** to `.audit-progress.json` after each module completes — so the next invocation resumes from where it left off if interrupted.
 9. **ALWAYS re-export** local `<project-slug>-collection.json` after fill batches. Newman reads the local file.
10. **ALWAYS write the coverage report file** to `collection-run-issues/<slug>-coverage-<timestamp>.txt` — regardless of `--report-only` vs full-fill mode.
11. **ALWAYS verify the shared `<ProjectName> Environment` exists** at Phase 0b. If missing, warn the user and offer to recreate it before continuing.

If the checkpoint disagrees with what's actually in the collection (e.g. user deleted tests manually), trust the live collection and update the checkpoint, but warn the user about the discrepancy.

If you catch yourself about to violate any invariant, stop and explain to the user. Do not silently proceed.

---

## Phase 0 — Prereqs

### 0a — Postman plugin

`mcp__plugin_postman_postman__getAuthenticatedUser`. If fails → stop, point to `/qa-api-test-setup` Phase 0a.

### 0b — Project state

Load `.env`. Required keys: `BASE_URL`, `OPENAPI_SPEC`. If missing → tell user to run `/qa-api-test-setup` first.

Check the main collection exists. Locate `<project-slug>-collection.json` in the project root. If missing → check if this is foundation-only state (shared infra present + `Jira Tickets/` folder may exist but no main collection):

```bash
if [ -f .env ] && [ -d "Jira Tickets" ] && [ ! -f <project-slug>-collection.json ]; then
  echo "Foundation-only project — main suite not built yet."
fi
```

If foundation-only → stop and tell user: `Foundation-only project detected. /qa-negative-audit needs the main collection to audit against. Run /qa-api-test-setup first (it will reuse your existing shared infrastructure).` and exit.

If neither main collection nor shared infra → stop with: `No main collection found. Run /qa-api-test-setup first.`

Check the negative matrix reference exists: `.claude/commands/qa-negative-matrix.md`. If missing → stop and tell user the matrix doc is the contract; nothing to audit against without it.

Check the shared Postman environment exists. Call `mcp__plugin_postman_postman__getEnvironments` and look for `<ProjectName> Environment`. If missing, warn the user:
```
Shared Postman environment "<ProjectName> Environment" not found.
This means Postman GUI runs and matrix-based negative tests (which use vars like
{{expiredAccessToken}}, {{nonExistentId}}) will not work until the env is recreated.

Recreate it now from the setup spec (y/n)?
```
- `y` → call `mcp__plugin_postman_postman__createEnvironment` with the universal-var list from `/qa-api-test-setup` Phase 6b. Save its ID.
- `n` → continue, but mark in the audit report that env-dependent rows can't be verified.

### 0c — Resume from checkpoint?

If `.audit-progress.json` exists in the project root:

```
Existing audit checkpoint found from <startedAt>.
  Completed modules: <list>
  In progress:       <currentModule> (endpoint <N> of <M>)
  Endpoints covered: <N> of <total>

Resume from checkpoint? (y/n)
  y — pick up where the previous audit left off
  n — discard checkpoint and start over
```

If `y` → load checkpoint, skip to Phase 5 with the saved state.
If `n` → delete checkpoint, continue to Phase 1.

---

## Phase 1 — Inventory

### 1a — Load endpoints from the spec

Read the spec (same logic as `/qa-api-test-setup` Phase 2a — Scalar/HTML auto-discovery if needed). Print the mandatory count block (same format as setup Phase 2b). Hard-stop if total endpoints = 0.

### 1b — Load existing tests from the collection

Fetch the collection via `mcp__plugin_postman_postman__getCollection`. Parse the folder tree:

- Find `Happy Path - All Endpoints` (record its endpoint set, for cross-reference).
- For each module folder (`<NN>. <ModuleName>`), find:
  - `Positive` and its requests
  - `Negative` and its sub-folders: `Auth Failures`, `Validation Failures`, `Resource Errors`, `Security Probes` (any missing sub-folder = creates a gap automatically; record so Phase 5 creates them). For older collections that still have the short names (`Auth`, `Validation`, `Resource`, `Security`) or a flat layout, run `/qa-negative-audit --reorganize` first.

Build an in-memory index keyed by `<METHOD> <path>`:

```
existingTests = {
  "POST /api/v1/orders": {
    happyPath:  { id: "...", name: "POST /api/v1/orders" },
    positive:   [{ id, name, ... }],
    negative:   { auth: [...], validation: [...], resource: [...], security: [...] }
  },
  ...
}
```

For each test, extract its matrix-row `id` if the test name matches a known pattern (e.g. `... - No Auth (401)` → `auth-no-token`). Fall back to fuzzy name matching for tests written manually.

### 1c — Evaluate the matrix per endpoint

For each endpoint in the spec:

1. Walk every matrix row from `qa-negative-matrix.md`.
2. Evaluate the row's **condition** against the endpoint's spec (has body? has path param? has security? RBAC mentioned? etc.).
3. Mark the row as one of:
   - `✓ covered (TC<NN>)` — a matching test exists
   - `✗ missing` — condition applies but no test exists
   - `n/a (reason)` — condition does not apply
   - `skipped (reason)` — applies but env flag forbids (e.g. `sec-rate-limit` with `INCLUDE_RATE_LIMIT_TESTS=false`)
4. Compute per-endpoint coverage: `covered / (covered + missing)`. `n/a` and `skipped` rows do not count against denominators.

---

## Phase 2 — Gap report

Print the report, ordered by module. For each endpoint, show only:
- Rows that are missing
- Rows that are skipped (visibility)
- The coverage fraction

Hide `covered` and `n/a` rows for brevity unless the user passes `--verbose` (then show all).

Example:

```
Negative coverage audit — <Project Name>
Snapshot: <project-snapshot-YYYY-MM-DD.json>
Total endpoints: 160
Project coverage: 67% (438 covered / 654 applicable, 23 skipped)

══════════════════════════════════════════════════════════════════════════
00. Auth   (12 endpoints, 92% coverage)
──────────────────────────────────────────────────────────────────────────
  POST /api/v1/auth/login   8/9 applicable rows covered
    ✗ sec-sql-injection      missing
    ⊘ sec-rate-limit         skipped (INCLUDE_RATE_LIMIT_TESTS=false)

  POST /api/v1/auth/register   6/9 applicable rows covered
    ✗ val-wrong-type         missing
    ✗ val-out-of-range       missing
    ✗ sec-xss-reflection     missing
    ⊘ sec-rate-limit         skipped

══════════════════════════════════════════════════════════════════════════
01. Users   (15 endpoints, 41% coverage)
──────────────────────────────────────────────────────────────────────────
  GET /api/v1/users/{userId}   3/8 applicable rows covered
    ✗ auth-wrong-role        missing
    ✗ res-not-found          missing
    ✗ sec-path-traversal     missing
    ✗ sec-sql-injection      missing
    ✗ sec-xss-reflection     missing
    ⊘ sec-rate-limit         skipped

  ... (13 more endpoints)

══════════════════════════════════════════════════════════════════════════

Summary by module:
  00. Auth        92% ✓
  01. Users       41% — 84 gaps across 13 endpoints
  02. Profile     78%
  03. Orders      55% — 47 gaps across 9 endpoints
  04. Reports     100% ✓
  05. Admin       12% — 71 gaps across 6 endpoints

Total gaps to fill: 202 tests across 32 endpoints
Estimated batches: 7 (module-wise) | Estimated time: 25 min
```

---

## Phase 3 — Confirm gate

```
Generate the missing tests in batches of 1 module at a time?
  y    — generate everything (will prompt after each module)
  set  — generate one set (one module, then stop and ask)
  no   — exit, report only, change nothing
  edit — pick specific endpoints / specific rows to fill
```

- `y` → proceed to Phase 5 with full-run mode.
- `set` → proceed to Phase 5 with single-set mode (one module, then exit).
- `no` → write the report to `collection-run-issues/<slug>-coverage-<timestamp>.txt` and exit.
- `edit` → ask which modules/endpoints/rows to include; build a filtered work list; proceed to Phase 5.

---

## Phase 4 — Sub-folder normalization

Before filling gaps, ensure every module has the 4-way Negative sub-folder structure (`Auth Failures` / `Validation Failures` / `Resource Errors` / `Security Probes`). For each module that's missing any sub-folder:

1. Use `mcp__plugin_postman_postman__createCollectionFolder` with `parentFolderId: <module>NegativeFolderId` to create the missing sub-folder.
2. If `Negative` currently has tests directly in it (legacy flat layout), **do not move them automatically** — leave them and warn the user: `<module> > Negative has N tests at the root level. Run /qa-negative-audit --reorganize to move them into sub-folders, or move manually.`

The `--reorganize` mode is a separate one-shot pass that infers each loose test's row `id` from its name and moves it into the correct sub-folder. Not part of the default audit flow.

---

## Phase 5 — Generate missing tests (module-wise, checkpointed)

### 5a — Loop control

Iterate modules in their existing `NN.` order (do NOT re-audit module ordering — that's setup's job). For each module:

1. Print: `Working on <NN>. <ModuleName> — <N> gaps in <M> endpoints...`
2. For each endpoint in the module (in audited flow order from setup), for each missing row, generate the test (Phase 5b).
3. After the module completes, **write checkpoint**:
   ```json
   {
     "command": "/qa-negative-audit",
     "startedAt": "<ISO>",
     "completedModules": ["00. Auth", "01. Users", "02. Profile"],
     "currentModule": null,
     "currentEndpointInModule": 0,
     "totalEndpointsCovered": 27,
     "totalEndpoints": 160,
     "matrixVersion": "<hash of qa-negative-matrix.md>"
   }
   ```
4. If `totalEndpoints > 20` AND mode is not single-set, ask: `<NN>. <ModuleName> done — N gaps filled. Continue to <NN+1>. <NextModule>? (y/n)`. `n` → stop, checkpoint preserved.
5. If mode is single-set → stop after this module's first batch.
6. For modules with > 10 endpoints, break into sub-batches of 10 endpoints. Same checkpoint logic per sub-batch.

### 5b — Generate one test (the matrix call)

For a missing row `<row-id>` on endpoint `<METHOD> <path>` (identity key; not the display name):

1. Look up the row in `.claude/commands/qa-negative-matrix.md` — get the row description, expected status, test script template, and target sub-folder.
2. Resolve `<op.summary>` for the endpoint per the fallback chain in CLAUDE.md § Happy Path folder (summary → Title-Cased operationId → `<METHOD> <path>`). Substitute `<field>` placeholders inside the row description if applicable.
3. Determine the next available TC number for this module (continuous across all four Negative sub-folders).
4. Build the request:
   - `folderId`: the target sub-folder (`<module>Negative<Subfolder>FolderId`)
   - `name`: `TC<NN>: <op.summary> - <row description> (<status>)` — example: `TC07: Login - No Auth (401)`. The display name is human-readable. NEVER use `<METHOD> <path>` in the display name unless it's the fallback case.
   - `method`, `url`, headers: copied from the existing happy-path request for this endpoint (these stay as actual HTTP method and path — `POST`, `/api/v1/auth/login` — for the actual HTTP request; only the `name:` is humanized), with the row's mutation applied (e.g. for `auth-no-token` → remove `Authorization` header; for `val-missing-required` → omit the field from body)
   - `body`: same as happy-path, with mutation
   - `events`: test script from the matrix row template
5. Call `mcp__plugin_postman_postman__createCollectionRequest`.
6. If the row generates multiple tests (e.g. `val-missing-required` for an endpoint with 5 required fields produces 5 tests, one per omitted field), loop within step 4.

### 5c — Setup dependencies

Some rows need Setup-state fixtures:
- `res-conflict` needs a resource already created with the same unique value.
- `res-wrong-state` needs a resource in a terminal/non-target state.
- `auth-wrong-role` needs an alternate-user/role access token.

For each such row generated, check whether the necessary Setup request already exists in the collection. If not:
- Add the Setup request to the **module's Positive folder** (or to a dedicated `Setup` request inside the module if the existing Setup pattern uses one — match existing convention).
- Set the appropriate collection variable so the negative test can reference it.
- Note this in the audit log.

If the alt-user token isn't available and no `{{altUserAccessToken}}` env var exists, ask the user once during Phase 3 confirm to provide one (or skip `auth-wrong-role` for the run).

### 5d — Cross-cutting assertions

After all the row-based negatives are generated for a batch, apply the cross-cutting assertions from the matrix (`xcut-sensitive-leak`, `xcut-error-body-shape`, `xcut-no-stack-trace`, `xcut-response-time`, `xcut-schema-validation`, `xcut-idempotency`):

- For each existing positive AND negative test in this batch's modules, fetch the request via `getCollectionRequest`, append the cross-cutting assertions to the test script, and update via `updateCollectionRequest`.
- If a test already has an identical cross-cutting block (e.g. from a previous audit run), skip — idempotent.

---

## Phase 6 — Re-export the local collection JSON

After all batches complete (or after the user stops mid-flow):

```bash
POSTMAN_API_KEY="<from .claude/settings.local.json or ~/.claude/settings.json>"
curl -s -H "X-API-Key: $POSTMAN_API_KEY" \
  "https://api.getpostman.com/collections/<OWNER_ID>-<COLLECTION_ID>" \
  -o "<project-slug>-collection.json"
```

---

## Phase 7 — Summary + cleanup

Print:

```
Audit complete.

Tests generated this run:     <N>
Endpoints touched:             <N>
Modules completed:             <N> / <M>
Cross-cutting assertions added: <N>

Project coverage:
  Before: 67% (438 / 654)
  After:  93% (607 / 654)
  Skipped (rate-limit, etc.): 23

Remaining gaps:
  - <module>: <N> tests still missing (run /qa-negative-audit again to fill, or run /qa-negative-audit set to do one module)

Local collection JSON:  <project-slug>-collection.json (re-exported)
Audit report:           collection-run-issues/<slug>-coverage-<timestamp>.txt
```

**Always write the audit report to `collection-run-issues/<project-slug>-coverage-<YYYYMMDD-HHMMSS>.txt`** — not just on `--report-only` runs. After a fill-run the report shows before-and-after state per endpoint (which rows were missing, which got filled, which remain). After a `--report-only` run the report is the gap inventory only. Create `collection-run-issues/` if it doesn't exist.

Delete `.audit-progress.json` if the full run completed (no remaining work for any module). Keep it if the user stopped mid-flow so the next invocation can resume.

---

## Flags

| Flag | Behavior |
|---|---|
| `(no flag)` | Default audit — report + offer to fill in module-wise batches |
| `--report-only` | Generate the gap report, write it to `collection-run-issues/`, exit. Same as answering `no` at Phase 3. |
| `set` (argument, not flag) | Single-set mode — fill one module of gaps then stop. Useful when user wants to chip away over time. |
| `--verbose` | Show all rows in the report (covered, n/a, skipped, missing), not just gaps. |
| `--reorganize` | One-shot pass: move loose tests at the root of `Negative` into the 4-way sub-folders based on test name. Skips matrix-driven generation. |
| `--include-rate-limit` | Override `INCLUDE_RATE_LIMIT_TESTS=false` from `.env` for this run only. Use with caution — destructive on the API. |
