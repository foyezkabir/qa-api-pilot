# Project context for Claude Code

This project is an automated API testing suite for REST APIs, built on Postman + Newman, driven by three custom slash commands defined in `.claude/commands/`. When you operate in this project, treat these commands and rules as authoritative.

## The custom commands

| Command | Use when |
|---|---|
| `/qa-api-test-setup` | First-time project setup. Run once to build the main collection from an OpenAPI spec. Has a preflight guard - refuses to re-run on an already-set-up project unless the user types `force re-setup`. Main collection uses **Happy Path folder + nested module folders + Positive/Negative-with-4-sub-folder layout**. |
| `/qa-test-ticket <JIRA-KEY>` | A new Jira ticket needs scoped API tests. Auto-runs `/qa-api-sync` first, hard-stops on missing credentials, builds a standalone `<KEY>: <feature>` collection with **flat Setup / Happy Path - All Endpoints / Positive / Negative folders** (no module sub-grouping — tickets are scoped to one feature). Happy Path is ordered by a dependency audit scoped to the ticket's endpoints. |
| `/qa-api-sync` | The OpenAPI spec changed and the collection is out of date. Diffs spec vs snapshot, auto-updates the main collection's request bodies/params (preserves test scripts), refreshes affected ticket collections, archives removed endpoints as `[DEPRECATED]`. Scheduled daily at 2:30 PM BDT via `/schedule`. |
| `/qa-negative-audit` | On-demand negative-coverage audit. Walks every endpoint, evaluates the negative matrix, reports gaps per endpoint, and offers to fill them in module-wise batches with a checkpoint file. Run when sync flagged many additions, or periodically to catch coverage drift. **Distinct from `/qa-api-sync`** — sync = spec drift, audit = coverage drift. |

Full details are in `.claude/commands/qa-api-test-setup.md`, `qa-test-ticket.md`, `qa-api-sync.md`, `qa-negative-audit.md`. Two shared reference docs are sourced by all four commands: `qa-negative-matrix.md` (the negative test matrix + body-data Mode A/B) and `qa-debugging-playbook.md` (canonical debugging playbook). Workflow walk-throughs are in `README.md`.

## Hard rules (project conventions - never violate)

### Bugs and tickets

- **Always reproduce manually with curl + read the full response body BEFORE raising any Jira ticket at any priority level.** This is non-negotiable. The `/qa-test-ticket` Phase 9 enforces this in its verification gate. No agent should auto-create a Jira ticket without a successful curl repro first.
- **Search Jira for duplicates** before creating a new bug. Add comments to existing tickets when matches are found rather than spawning duplicates.

### Test execution

- **Never `--bail` the Newman run.** The initial suite run and every auto-fix iteration must complete fully so the HTML report captures every failure in one pass. `--bail` is only acceptable for manual human triage of one specific request — see `.claude/commands/qa-debugging-playbook.md` § 5 for the rule of thumb.
- **3-iteration cap on auto-fix.** Do not loop forever. If 3 rounds of auto-fix do not resolve a failure, hand off to the human.
- **Patch test setup only, never assertions.** If an assertion says "expected 200" and the response is 201, the fix is to verify the spec changed (run `/qa-api-sync`), not to silently change the assertion. Changing assertions hides real bugs.
- **Never modify the system under test.** Auto-fix only changes how we call the API, never the API itself.

### Postman cloud ↔ local JSON sync

- **Whenever the agent patches a request via Postman MCP (`updateCollectionRequest`), immediately re-export the local collection JSON.** Newman reads the local file, so an out-of-sync file means patches have no effect on the next run. Both files (`qa-api-test-setup.md` Phase 8b and `qa-test-ticket.md` Phase 6d) define the exact curl command.

### `.env` conventions

- **Required runtime keys**: `BASE_URL`, `OPENAPI_SPEC`, `HEALTH_PATH`, `LOGIN_EMAIL`, `LOGIN_PASSWORD`, `PLATFORM_API_KEY` (if apiKey auth), `JIRA_PROJECT_KEY` (auto-populated on first ticket).
- **Auto-detected (NOT in `.env` by default)**: `AUTH_TYPE`, `LOGIN_PATH`, `TOKEN_JSON_PATH` are derived from the OpenAPI spec during `/qa-api-test-setup` Phase 1d. The detected values are baked into the generated collection. They only appear in `.env` if the user manually overrode them when the spec was wrong or incomplete.
- **`POSTMAN_API_KEY` (PMAK-...) lives under `env.POSTMAN_API_KEY` in either `.claude/settings.local.json` (project-local, gitignored) OR `~/.claude/settings.json` (global, user-wide). NEVER in project `.env`.** Both are valid; pick one per machine/project. Claude Code's settings cascade auto-merges them, with **local winning if both are set**. Local is recommended for fresh cloners (matches the `.env` mental model, gitignored, per-project key possible); global is the one-time-per-machine option for users with a single Postman account across projects. A `.claude/settings.local.json.example` ships with the repo — cloners copy it to `.claude/settings.local.json` and paste their PMAK. The two `.claude/` directories are different scopes: global (`~/.claude/`) holds user-wide settings; project-local (`<project>/.claude/`) holds project agents, templates, and (optionally) project-local credentials.
- **Workspace IDs are NEVER stored in `.env`.** Always list workspaces fresh via `getWorkspaces` and let the user pick.
- **One-off entity IDs do NOT go in `.env`.** Only persistent credentials and API keys.
- **`JIRA_PROJECT_KEY` is auto-extracted** from the first `/qa-test-ticket <KEY>` call (e.g. `PROJ-123` → `PROJ`) and saved automatically to `.env`. Used by the bug-logging gate.

### Test scripts

- **Never use `const` at top scope in Postman test scripts.** Newman throws `SyntaxError: Identifier already declared` because the same script gets executed multiple times in one run. Use `let` everywhere at top scope.
- **Ticket-scoped collections use `pm.environment.set(...)`** for variable chaining, not `pm.collectionVariables.set(...)`. The main project-level collection uses `pm.collectionVariables`. Do not mix the two scopes within one collection.
- **Use Mode A random generators** (`{{$randomEmail}}`, `{{$randomUUID}}`, `{{$randomFullName}}`, `{{$randomInt}}`, etc.) for unique fields (email, codes, names) to prevent 409 conflicts on re-run. See `.claude/commands/qa-negative-matrix.md` § Body data sources for the full source chain.

### Spec sources

- **Accept Scalar / Swagger UI / Redoc reference pages as spec sources, not just raw OpenAPI JSON/YAML.** When the user provides a URL that returns HTML, both `/qa-api-test-setup` (Phase 2a) and `/qa-api-sync` (Phase 1a) must auto-discover the underlying spec by looking at: Scalar embed (`<script id="api-reference" data-url="...">`), inline Scalar (`<script id="api-reference" type="application/json">`), Redoc embed (`<redoc spec-url="...">`), Swagger UI config (`url:`/`urls:` inside script blocks), and conventional paths (`/openapi.json`, `/openapi.yaml`, `/swagger.json`, `/api/openapi.json`, `/api-docs`, `/v3/api-docs`). If discovery fails, ask the user once for the raw spec URL.
- **Mandatory endpoint count print after loading any spec.** After a successful spec load in `/qa-api-test-setup` Phase 2b and `/qa-api-sync` Phase 1b, print a fixed summary block to the user (source, resolved spec URL, OpenAPI version, total endpoints, per-tag module counts, auth schemes — and for sync, delta vs snapshot). This is non-skippable. **Hard stop if total endpoints = 0.**

### Artifact ownership (three buckets, strict separation)

Every project file and Postman cloud artifact has exactly one owner. Crossing ownership lines is forbidden.

| Bucket | Artifacts | Created by | Modified by |
|---|---|---|---|
| **Shared infrastructure** | `.env`, `newman-env.json`, `run-tests.sh`, `generate-issues.py`, shared Postman environment in cloud | `/qa-api-test-setup` OR inline foundation in `/qa-test-ticket` | Either of the above, plus `/qa-api-sync` (patches env for new chained vars) |
| **Setup-owned** | `<project-slug>-collection.json`, `project-snapshot-*.json`, main Postman collection in cloud | `/qa-api-test-setup` only | `/qa-api-sync` (updates on drift), `/qa-negative-audit` (adds tests) |
| **Ticket-owned** | `Jira Tickets/<KEY>/collection.json`, `Jira Tickets/<KEY>/snapshot-*.json`, ticket Postman collection in cloud | `/qa-test-ticket` only | `/qa-test-ticket` (auto-fix), `/qa-api-sync` (refresh on drift) |

**Three valid project states**, detectable by file presence:
- **Fully fresh** — none of the bucket-1/2/3 artifacts exist.
- **Foundation-only** — bucket 1 (shared infra) exists. Bucket 2 (setup) does not. Bucket 3 (tickets) may or may not.
- **Fully set up** — all three buckets present.

Detection rules used by agents:
- `/qa-api-test-setup` preflight counts ONLY setup-owned signals (bucket 2). Shared-infra-only state → proceed normally, reuse shared infra, create main collection. Setup-owned signals present → block with force-re-setup gate.
- `/qa-test-ticket` Phase 0a: missing `.env` → 3-way prompt (inline foundation / full setup / cancel). Phase 0d: missing project snapshot → scoped sync mode (iterate per-ticket snapshots) or skip sync (no snapshots at all).
- `/qa-api-sync`: missing project snapshot → Mode B foundation-only scoped sync over ticket snapshots. No snapshots at all → stop.
- `/qa-negative-audit`: missing main collection → stop with "foundation-only project, run /qa-api-test-setup first".

### Postman Environment (shared across all collections in a project)

- **One Postman Environment per project**, named `<ProjectName> Environment`, created in the project workspace during `/qa-api-test-setup` Phase 6c. **Shared by the main collection AND every ticket collection** — one env per project, not one env per collection.
- The same env data is written in two places: the cloud Postman Environment (created via `createEnvironment`) AND the local `newman-env.json` file (used by Newman). Both must stay in sync — never edit one without the other.
- **Universal vars** are always present: `baseUrl`, `accessToken`, `refreshToken`, `platformApiKey`, `expiredAccessToken`, `tamperedAccessToken`, `altUserAccessToken`, `nonExistentId`, `RESPONSE_TIME_SLA_MS`, `INCLUDE_RATE_LIMIT_TESTS`.
- **Chained-ID vars** (`test<Entity>Id`) are **auto-detected from the spec** during setup — one per detected creatable POST endpoint. Names are never hardcoded; they're derived from the path's terminal resource segment (`POST /widgets` → `testWidgetId`).
- `/qa-api-sync` and `/qa-test-ticket` patch new chained-ID vars into the shared env via `patchEnvironment` AND update `newman-env.json`. Existing vars are never touched.
- `/qa-negative-audit` verifies the env exists at Phase 1; offers to recreate it if missing.
- Ticket test scripts use `pm.environment.set(...)` (not `pm.collectionVariables.set(...)`) so they read/write to this shared env. The main project collection still uses `pm.collectionVariables` for its own scope — both can coexist; do not mix them within a single collection.
- The user can manually run any collection in the Postman GUI by selecting `<ProjectName> Environment` from the env dropdown. Tokens auto-populate when Login runs. This is the key UX win — without the cloud env, manual Postman runs require the user to construct env vars by hand.

### Body data generation (Mode A / Mode B)

- **POST / PUT / PATCH bodies are NEVER hardcoded across endpoints.** They're built from one canonical source chain documented in `.claude/commands/qa-negative-matrix.md` § Body data sources.
- **Two modes**:
  - **Mode A (positive tests)** — pre-request script builds random constraint-aware values for user-supplied fields, chained env vars for resource references (`{{testCustomerId}}` etc.), skips read-only fields (`id`, `createdAt`) and sensitive fields (`password`). Stashes the sent body in `{{__sentBody}}`. Post-response script asserts **per-field round-trip** (every user-supplied field echoed back) + status + schema + cross-cutting (sensitive-leak, response-time). Catches the "API silently dropped my field" regression class.
  - **Mode B (negative tests)** — start from a **deterministic valid baseline body** (first enum value, spec example, type-default — no randomness; negatives must be reproducible) and apply the matrix row's **mutation** (e.g. omit one required field; swap a field's type; inject `' OR '1'='1`). Assert the row's expected error code + error-body shape + no-stack-trace + sensitive-leak. **No round-trip** — the request was supposed to fail.
- **Happy Path - All Endpoints folder uses Mode A's body gen BUT a lightweight assertion profile**: status + schema only, no per-field round-trip. Round-trip rigor lives in per-module `Positive` folders. Happy Path = "is it alive AND shaped correctly?" in two cheap assertions.
- **GET / DELETE / HEAD / OPTIONS need no body.** Path params come from chained env vars. Query params from spec `default` / `enum[0]` / random within constraints.
- **Fallback for ungenerable fields**: if no source applies (no enum, no example, no default, no format, no pattern, no chained match), use a generic placeholder (`"test-string"`, `0`, `false`) and print a warning at end-of-build listing the affected fields. Better visible fallback than silent constraint violations.

### Negative coverage matrix

- **Every endpoint MUST be evaluated against the full negative matrix** defined in `.claude/commands/qa-negative-matrix.md`. The agents do not "decide mentally" what negatives are worth writing — they walk every matrix row, evaluate the condition, and either generate the test or mark the row `n/a (condition not met)`. No row is silently skipped.
- **The matrix has 24 rows** organized into 4 negative categories (`Auth Failures`, `Validation Failures`, `Resource Errors`, `Security Probes`) plus 6 cross-cutting assertions added INTO existing test scripts (`xcut-sensitive-leak`, `xcut-error-body-shape`, `xcut-no-stack-trace`, `xcut-response-time`, `xcut-schema-validation`, `xcut-idempotency`).
- **Main collection: 4-way `Negative` sub-folder split.** Every module's `Negative` folder has `Auth Failures` / `Validation Failures` / `Resource Errors` / `Security Probes` sub-folders, always created upfront even if empty. Sub-folder names describe the **category of failure** held inside, applied to the parent module's endpoints (e.g. `Orders > Negative > Auth Failures` = auth-failure tests for Orders endpoints, NOT tests of the Auth module). TC numbering is continuous across all four (single counter per module).
- **Ticket collections: flat `Negative`.** Same matrix applied, but the 4 categories share one flat folder since ticket surface area is small.
- **Rate-limit tests are destructive and opt-in.** Gated by `INCLUDE_RATE_LIMIT_TESTS=true` in `.env`. Default `false`. Asked once during `/qa-api-test-setup` Phase 1. The audit reports skipped rate-limit rows as `skipped (INCLUDE_RATE_LIMIT_TESTS=false)` so the gap stays visible.
- **Coverage = `covered / (covered + missing)`**, with `n/a` and `skipped` excluded from the denominator. A "100% coverage" project means every applicable row has a test, not necessarily that every matrix row was generated.
- **`/qa-negative-audit` is the canonical way to check coverage.** Sync may suggest running it after big spec changes; manual runs are encouraged periodically (weekly / monthly) to catch hand-edit drift.
- **Matrix is the contract.** When a test-script template or payload needs to change globally, edit `qa-negative-matrix.md`. All four commands re-read it on their next run.
- **Audit edits are additive only.** `/qa-negative-audit` never modifies or deletes existing tests; it only adds missing ones. Renumbering or removing requires explicit human action.

### Dependency-aware ordering (audit pass)

- **Before building any collection, run a dependency audit over the in-scope endpoints** (whole spec for `/qa-api-test-setup`, ticket-scoped endpoints for `/qa-test-ticket`). Build a DAG from four signals: (a) `security: [bearerAuth]` → depends on login chain; (b) path params like `{userId}` → depends on a producer (a `POST` whose response carries `userId`); (c) request body referencing another resource ID → depends on producer of that ID; (d) verb-within-resource order: `POST` → `GET /list` → `GET /<id>` → `PUT/PATCH` → `DELETE`. Topological sort the DAG.
- **The audit drives two orderings**: (1) the order of requests inside `Happy Path - All Endpoints`, (2) the `NN.` index of top-level module folders in the main collection. Inside each module's `Positive` / `Negative` sub-folders, order is unchanged — TC numbering follows the existing rules, no reorder.
- **Orphans**: if a dependency cannot be auto-resolved (e.g. `{auditLogId}` has no producer in the spec), the endpoint is placed at the end of its folder with a `{{test<Entity>Id}}` placeholder, and listed in the plan's orphan block for the user to wire up manually. Never block the build for orphans.
- **One confirm gate, not two**: the audited order is folded into the existing Phase 4 plan (setup) / Phase 3 plan (ticket). User responds `y` / `n` / `edit`. No separate audit-confirm step.
- **The plan must render reasoning inline.** For every Happy Path entry and every module in the printed plan, show what it needs (with the producer's flow-index for citation) and what it produces — e.g. `05. Create Order — needs {{accessToken}} (#02); produces {{testOrderId}}`. For modules: `02. Profile (was 07 in spec) — needs user created in 01. Users`. For orphans: a `reason:` line explaining why the producer couldn't be auto-resolved. This lets the user audit the agent's reasoning before approving, not just the order. The Phase 2d audit must capture `needs` / `needsFrom` / `produces` / `reason` per entry to make this rendering possible.
- **`/qa-api-sync` does not re-audit existing endpoints or renumber modules** — only newly ADDED endpoints get audited against the existing Happy Path order and inserted at their correct dependency slot. Renumbering existing modules mid-project would scramble TC IDs the team is already referencing.

### Happy Path folder

- **`Happy Path - All Endpoints`** is the first top-level folder in every collection (main and ticket). It holds exactly one valid-data request per `(method, path)` in scope.
- **Naming**: `<NN>. <friendly name>` where `<NN>` is the audited-flow index (zero-padded) and `<friendly name>` comes from the OpenAPI op's `summary` field. Example: `02. Create Supplier`. No `TC` prefix, no status suffix on Happy Path entries.
  - **Friendly-name fallback chain** (deterministic, agents must follow): (1) op's `summary` field if present → use it (e.g. `"Create a new supplier"` → `Create Supplier`); (2) else op's `operationId` converted to Title Case (e.g. `getOrderById` → `Get Order By Id`, `list_user_orders` → `List User Orders`); (3) else fall back to `<METHOD> <path>` (e.g. `POST /orders`). Never block on missing names.
  - **Index scope**: `NN` is sequential across the entire Happy Path folder (not reset per module). Zero-padded to **2 digits** by default (`01.`, `02.`, …, `99.`). Use **3 digits** when total endpoints in Happy Path > 100 (`001.`, `002.`, …, `227.`).
  - **Duplicate summaries**: if two endpoints share the same summary (e.g. spec has two `List products` ops for v1 and v2), disambiguate by appending the HTTP method, the last path segment, or a version tag — whatever makes them distinct. E.g. `15. List Products (v1)`, `16. List Products (v2)`.
  - **Matrix-row test naming** (Negative folder): the agent assembles each negative test's name as `TC<NN>: <op.summary> - <row description> (<status>)`. Example: `TC07: Login - No Auth Token (401)`. Matrix doc rows contain only the `<row description> (<status>)` part; the `<op.summary> - ` prefix is added at build time.
  - **Internal identity vs display**: lookup keys, snapshot keys, request `method:`/`url:` fields stay as the actual HTTP method and path (`POST /api/v1/orders`). Only the test's display `name:` uses the friendly format. Identity (machine-readable) and display (human-readable) are deliberately separate.
- **Test scripts**: status-only assertion (`pm.expect(pm.response.code).to.be.oneOf([200, 201, 202, 204])`). No deep schema checks — those live in module Positive folders.
- **Variable chaining**: in the main collection, the login request in Happy Path also sets `accessToken` (so it must be the first Happy Path request). In ticket collections, variable chaining stays in Setup — Happy Path doesn't `pm.environment.set(...)`.
- **Endpoint coverage rule**: every spec endpoint must appear in Happy Path AND in its module's Positive (main collection) or in the ticket's Positive (ticket collection). Happy Path is the smoke pass; Positive is the assertion-rich pass.

## Collection structures (two different shapes)

**Main collection** (`/qa-api-test-setup`) - Happy Path folder + module-grouped Positive/Negative:

```
<ProjectName> API — Automated Tests
├── Happy Path - All Endpoints           ← top-level, runs first
│   ├── 01. Register User
│   ├── 02. Login
│   ├── 03. List Orders
│   ├── 04. Create Order
│   └── ... one valid-data request per (method, path), named <NN>. <op.summary>
├── 00. <ModuleFromSpecTag>
│   ├── Positive
│   │   └── TC01: ... (200)
│   └── Negative
│       └── TC02: ... (401)
└── 01. <NextModule>
    ├── Positive
    └── Negative
```

- `Happy Path - All Endpoints` sits **alongside** the module folders (not replacing them). It contains one valid-data request per spec endpoint, named `<NN>. <op.summary>` (sequential index + friendly name; see the Happy Path naming rule for the full fallback chain and the duplicate-summary handling). Status-only assertions. Login goes here too (first request) so `accessToken` is set before downstream Happy Path calls.
- Module names come from OpenAPI spec `tags` (never hardcoded).
- Module `NN.` index reflects **audited dependency-flow order** from `/qa-api-test-setup` Phase 2d, not spec-tag order. Auth/signup modules float to the top; downstream modules get later indices. `/qa-api-sync` does NOT renumber existing modules — new tags added during sync get appended as the next available `NN.`.
- TC numbering **resets per module**; continuous across Positive → Negative within one module. Happy Path requests are not TC-numbered.
- Both sub-folders always exist per module, even if one has a single test.
- Every endpoint gets: one entry in Happy Path + at least one happy path in its module's Positive + at least one no-auth → 401 negative.

**Ticket collection** (`/qa-test-ticket`) - flat, four folders:

```
<KEY>: <feature name>
├── Setup
├── Happy Path - All Endpoints           ← scoped to ticket's endpoints
├── Positive
└── Negative
```

- No module sub-grouping (one ticket = one feature).
- `Happy Path - All Endpoints` here is scoped to only the endpoints the ticket touches. Same naming rule: `<NN>. <op.summary>` (sequential within the ticket's Happy Path folder). Status-only assertions. Variable chaining (`pm.environment.set`) stays in Setup — Happy Path doesn't set vars.
- Folder order in the collection: Setup → Happy Path → Positive → Negative (matches run order).
- Continuous TC numbering across Positive → Negative (no reset). Happy Path requests are not TC-numbered.
- Full auth-matrix negatives (no/expired/tampered/malformed token).

When building/updating collections via Postman MCP, always create the sub-folders explicitly via `createCollectionFolder` with `parentFolderId` set. Do not route requests directly under module folders for the main collection - they must go into the `Positive` or `Negative` sub-folder.

## Project file layout

```
your-project/
├── .env                                # API base URL, creds, spec source, JIRA_PROJECT_KEY
├── .env.example                        # Template
├── .gitignore                          # Tells git which generated files to skip
├── README.md, CLAUDE.md                # Docs
├── .claude/
│   ├── commands/                       # Agent definitions + shared references
│   │   ├── qa-api-test-setup.md
│   │   ├── qa-test-ticket.md
│   │   ├── qa-api-sync.md
│   │   ├── qa-negative-audit.md
│   │   ├── qa-negative-matrix.md       # shared matrix — sourced by all 4 commands
│   │   └── qa-debugging-playbook.md    # shared debugging playbook — referenced by setup + ticket + audit
│   ├── templates/                      # run-tests.sh + generate-issues.py templates
│   ├── settings.local.json             # project-local Postman API key (gitignored)
│   └── settings.local.json.example     # template for cloners
├── run-tests.sh                        # GENERATED from template by /qa-api-test-setup
├── generate-issues.py                  # GENERATED from template by /qa-api-test-setup
├── newman-env.json                     # GENERATED by /qa-api-test-setup; mirror of the cloud Postman Environment "<ProjectName> Environment"
├── <project-slug>-collection.json      # GENERATED main Postman collection export
├── (ticket collections moved to Jira Tickets/<KEY>/ subfolders — see above)
├── project-snapshot-YYYY-MM-DD.json    # GENERATED setup-owned drift baseline (project-wide)
├── Jira Tickets/                       # GENERATED ticket-owned subfolder tree
│   ├── PROJ-123/
│   │   ├── collection.json             # Ticket Postman collection export
│   │   └── snapshot-YYYY-MM-DD.json   # Per-ticket scoped snapshot (only this ticket's endpoints)
│   └── PROJ-456/...
├── .audit-progress.json                # GENERATED (transient) — checkpoint for resumable /qa-negative-audit; auto-deleted on full completion
├── newman-reports/                     # GENERATED HTML + JSON per run
└── collection-run-issues/              # GENERATED parsed issues .txt per run + coverage-*.txt reports from /qa-negative-audit --report-only
```

## Template flow (`.claude/templates/` → project root)

The `.claude/templates/` folder holds two blueprint files that `/qa-api-test-setup` Phase 7 reads ONCE during setup and writes to the project root:

| Template (in subfolder) | Generated (in root) | Substitution at write time |
|---|---|---|
| `.claude/templates/run-tests.sh` | `run-tests.sh` (chmod +x) | `__PROJECT_NAME__`, `__PROJECT_SLUG__`, `__BASE_URL__`, `__HEALTH_PATH__`, `__HEALTH_AUTH_HEADER__` replaced from `.env` |
| `.claude/templates/generate-issues.py` | `generate-issues.py` | No placeholders - generic. Reads project info from CLI args at runtime (passed by run-tests.sh). |

Key invariants:

- Templates are read ONCE per project, during setup only. Never touched at runtime.
- Newman / `run-tests.sh` at runtime never look at the templates folder - they use the generated files in root.
- Editing a template after setup does NOT propagate to an already-set-up project. To pick up template changes, the user must re-run `/qa-api-test-setup force re-setup` (or manually copy the new content).
- If a generated file goes missing, re-running `/qa-api-test-setup` will detect partial setup and regenerate from the templates.

When asked to change how `run-tests.sh` or `generate-issues.py` behaves for FUTURE projects, edit the template in `.claude/templates/`. When asked to change behavior for the CURRENT project right now, edit the root file directly AND update the template to match (so future re-runs stay aligned).

## What `./run-tests.sh` produces (and why generate-issues.py matters)

Each run writes three outputs:
1. **HTML report** in `newman-reports/` - the visual one.
2. **JSON results** in `newman-reports/` - raw Newman output, consumed by generate-issues.py.
3. **Issues report** in `collection-run-issues/<slug>-issues-<timestamp>.txt` - human-readable text classifying failures into:
   - **Status mismatch** (wrong HTTP code with security/regression context)
   - **Message quality** (vague error messages with guidance on improvement)

When investigating a failure, prefer reading the issues report first - it has already classified the failure type and explained why each issue matters. Drop into the HTML report for visual cross-reference.

The `generate-issues.py` script is generated from `.claude/templates/generate-issues.py` at setup time and takes `<timestamp> <results.json> <output-dir> <project-slug> <project-name>` as args. It is invoked by `run-tests.sh` automatically.

## When something fails

Three stages, in this order:

1. **Auto-fix** (the agent attempts to fix what it can):
   - Spec drift → call `/qa-api-sync` → re-run
   - Stale `accessToken` → re-run Login → re-run downstream
   - Stale chained IDs → re-run Setup folder → re-run dependents
   - 409 conflict on unique field → patch body with Mode A random value (`{{$randomEmail}}`, `{{$randomUUID}}`, etc.) → re-run
   - `const` SyntaxError → patch script `const` → `let` → re-run
   - Transient 5xx / HTML error page → retry once after wait
2. **Manual debugging** (`.claude/commands/qa-debugging-playbook.md` - canonical playbook, shared by all four agents):
   - Classify (real bug / spec drift / test setup / env / flaky)
   - Read the HTML report properly (4 tabs per failed request)
   - Curl repro (PROJECT RULE)
   - Match against common HTTP patterns
   - Newman flags for deeper digging
   - Decision tree
3. **Jira bug logging** (`/qa-test-ticket` Phase 9 only - never in setup):
   - Verification gate: curl repro must succeed
   - Dedupe check
   - Priority prompt (default Medium)
   - Structured 6-section ticket: Context / What we tested / Expected / Actual / Reproduction / Test metadata
   - Linked to parent ticket via "relates to"

## When Claude is asked to make changes

- **Edit the agent definition files** in `.claude/commands/` for any agent behavior changes - they are the source of truth.
- **Keep README.md and CLAUDE.md in sync** when agent flows change. README is user-facing prose; CLAUDE.md is the AI-facing summary.
- **Do not delete `project-snapshot-*.json` or `Jira Tickets/<KEY>/snapshot-*.json`** without warning the user — they hold the drift baselines (project-wide and per-ticket respectively).
- **Do not delete `Jira Tickets/<KEY>/` folders** without warning the user - each contains the ticket's collection AND its per-ticket snapshot.

## Today's date and timezone

Bangladesh Standard Time (UTC+6). All `/schedule` commands use BDT. The daily auto-sync runs at 14:30 BDT.

## Pointer for more detail

- User-facing docs and workflows: `README.md`
- Agent internals: `.claude/commands/qa-*.md`
- Project memory (auto-loaded by Claude harness): `~/.claude/projects/<auto-generated-project-id>/memory/` (the project ID is derived from your local project path - Claude Code creates and manages this folder automatically the first time you open the project)
