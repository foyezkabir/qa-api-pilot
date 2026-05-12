# Project context for Claude Code

This project is an automated API testing suite for REST APIs, built on Postman + Newman, driven by three custom slash commands defined in `.claude/commands/`. When you operate in this project, treat these commands and rules as authoritative.

## The three custom commands

| Command | Use when |
|---|---|
| `/qa-api-test-setup` | First-time project setup. Run once to build the main collection from an OpenAPI spec. Has a preflight guard - refuses to re-run on an already-set-up project unless the user types `force re-setup`. Main collection uses **nested module folders + Positive/Negative sub-folders** (module names come from OpenAPI spec tags). |
| `/qa-test-ticket <JIRA-KEY>` | A new Jira ticket needs scoped API tests. Auto-runs `/qa-api-sync` first, hard-stops on missing credentials, builds a standalone `<KEY>: <feature>` collection with **flat Setup/Positive/Negative folders** (no module sub-grouping - tickets are scoped to one feature). |
| `/qa-api-sync` | The OpenAPI spec changed and the collection is out of date. Diffs spec vs snapshot, auto-updates the main collection's request bodies/params (preserves test scripts), refreshes affected ticket collections, archives removed endpoints as `[DEPRECATED]`. Scheduled daily at 2:30 PM BDT via `/schedule`. |

Full details are in `.claude/commands/qa-api-test-setup.md`, `qa-test-ticket.md`, `qa-api-sync.md`. Workflow walk-throughs are in `README.md`.

## Hard rules (project conventions - never violate)

### Bugs and tickets

- **Always reproduce manually with curl + read the full response body BEFORE raising any Jira ticket at any priority level.** This is non-negotiable. The `/qa-test-ticket` Phase 9 enforces this in its verification gate. No agent should auto-create a Jira ticket without a successful curl repro first.
- **Search Jira for duplicates** before creating a new bug. Add comments to existing tickets when matches are found rather than spawning duplicates.

### Test execution

- **Never `--bail` the Newman run.** The initial suite run and every auto-fix iteration must complete fully so the HTML report captures every failure in one pass. `--bail` is only acceptable in Phase 11e (manual human triage of one specific request).
- **3-iteration cap on auto-fix.** Do not loop forever. If 3 rounds of auto-fix do not resolve a failure, hand off to the human.
- **Patch test setup only, never assertions.** If an assertion says "expected 200" and the response is 201, the fix is to verify the spec changed (run `/qa-api-sync`), not to silently change the assertion. Changing assertions hides real bugs.
- **Never modify the system under test.** Auto-fix only changes how we call the API, never the API itself.

### Postman cloud ↔ local JSON sync

- **Whenever the agent patches a request via Postman MCP (`updateCollectionRequest`), immediately re-export the local collection JSON.** Newman reads the local file, so an out-of-sync file means patches have no effect on the next run. Both files (`qa-api-test-setup.md` Phase 8b and `qa-test-ticket.md` Phase 6d) define the exact curl command.

### `.env` conventions

- **Required runtime keys**: `BASE_URL`, `OPENAPI_SPEC`, `HEALTH_PATH`, `LOGIN_EMAIL`, `LOGIN_PASSWORD`, `PLATFORM_API_KEY` (if apiKey auth), `JIRA_PROJECT_KEY` (auto-populated on first ticket).
- **Auto-detected (NOT in `.env` by default)**: `AUTH_TYPE`, `LOGIN_PATH`, `TOKEN_JSON_PATH` are derived from the OpenAPI spec during `/qa-api-test-setup` Phase 1d. The detected values are baked into the generated collection. They only appear in `.env` if the user manually overrode them when the spec was wrong or incomplete.
- **`POSTMAN_API_KEY` (PMAK-...) lives in the GLOBAL Claude Code config (`~/.claude/settings.json` in the user's home directory, NOT in the project-local `.claude/` folder), under `env.POSTMAN_API_KEY`. NEVER in project `.env`.** It is a Claude Code plugin credential shared across every project on the user's machine. The two `.claude/` directories are different scopes - global (`~/.claude/`) holds plugin credentials and global settings; project-local (`<project>/.claude/`) holds the agents and templates for this specific project.
- **Workspace IDs are NEVER stored in `.env`.** Always list workspaces fresh via `getWorkspaces` and let the user pick.
- **One-off entity IDs do NOT go in `.env`.** Only persistent credentials and API keys.
- **`JIRA_PROJECT_KEY` is auto-extracted** from the first `/qa-test-ticket <KEY>` call (e.g. `PROJ-123` → `PROJ`) and saved automatically to `.env`. Used by the bug-logging gate.

### Test scripts

- **Never use `const` at top scope in Postman test scripts.** Newman throws `SyntaxError: Identifier already declared` because the same script gets executed multiple times in one run. Use `let` everywhere at top scope.
- **Ticket-scoped collections use `pm.environment.set(...)`** for variable chaining, not `pm.collectionVariables.set(...)`. The main project-level collection uses `pm.collectionVariables`. Do not mix the two scopes within one collection.
- **Use `{{$timestamp}}` for unique fields** (email, codes, names) to prevent 409 conflicts on re-run.

## Collection structures (two different shapes)

**Main collection** (`/qa-api-test-setup`) - module-grouped, two levels deep:

```
<ProjectName> API — Automated Tests
├── 00. <ModuleFromSpecTag>
│   ├── Positive
│   │   └── TC01: ... (200)
│   └── Negative
│       └── TC02: ... (401)
└── 01. <NextModule>
    ├── Positive
    └── Negative
```

- Module names come from OpenAPI spec `tags` (never hardcoded).
- TC numbering **resets per module**; continuous across Positive → Negative within one module.
- Both sub-folders always exist per module, even if one has a single test.
- Every endpoint gets at least one no-auth → 401 negative.

**Ticket collection** (`/qa-test-ticket`) - flat, three folders:

```
<KEY>: <feature name>
├── Setup
├── Positive
└── Negative
```

- No module sub-grouping (one ticket = one feature).
- Continuous TC numbering across Positive → Negative (no reset).
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
│   ├── commands/                       # The three agent definitions
│   └── templates/                      # run-tests.sh + generate-issues.py templates
├── run-tests.sh                        # GENERATED from template by /qa-api-test-setup
├── generate-issues.py                  # GENERATED from template by /qa-api-test-setup
├── newman-env.json                     # GENERATED by /qa-api-test-setup
├── <project-slug>-collection.json      # GENERATED main Postman collection export
├── <jira-key>-collection.json          # GENERATED one per Jira ticket built
├── api-snapshot-YYYY-MM-DD.json        # GENERATED drift baseline
├── newman-reports/                     # GENERATED HTML + JSON per run
└── collection-run-issues/              # GENERATED parsed issues .txt per run
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
   - 409 conflict on unique field → patch body to add `{{$timestamp}}` → re-run
   - `const` SyntaxError → patch script `const` → `let` → re-run
   - Transient 5xx / HTML error page → retry once after wait
2. **Manual debugging** (`qa-api-test-setup.md` Phase 11 - canonical playbook):
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
- **Do not delete `api-snapshot-*.json`** without warning the user - it loses drift history.
- **Do not delete `<jira-key>-collection.json`** files - they hold per-ticket test history.

## Today's date and timezone

Bangladesh Standard Time (UTC+6). All `/schedule` commands use BDT. The daily auto-sync runs at 14:30 BDT.

## Pointer for more detail

- User-facing docs and workflows: `README.md`
- Agent internals: `.claude/commands/qa-*.md`
- Project memory (auto-loaded by Claude harness): `~/.claude/projects/<auto-generated-project-id>/memory/` (the project ID is derived from your local project path - Claude Code creates and manages this folder automatically the first time you open the project)
