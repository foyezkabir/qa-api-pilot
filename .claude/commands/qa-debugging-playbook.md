---
name: qa-debugging-playbook
description: "Canonical reference for debugging failed tests across all QA agents. Not a command — a shared playbook the user (and agents) consult when /qa-api-test-setup smoke run, /qa-test-ticket Phase 6 auto-fix, or any manual ./run-tests.sh run produces failures. Covers: classify-first triage, how to read the HTML report, curl reproduction (mandatory before Jira), common HTTP failure patterns, Newman flags for deeper digging, decision tree, what-not-to-do, and the 'failure is actually correct behaviour' cases."
---

# Debugging failed tests (shared playbook)

This file is the canonical reference for all four QA agents. When the smoke test reports any failed test, or when `./run-tests.sh` later reports failures, do NOT auto-raise a Jira ticket. Walk through this triage methodically.

Cross-referenced by:
- `/qa-api-test-setup` Phase 8 (smoke test) and Phase 10 (summary remaining failures)
- `/qa-test-ticket` Phase 6 (auto-debug + auto-fix) and Phase 8 (manual debugging tips)
- `/qa-api-sync` (when a sync apply fails partway)

---

## 1. Classify first (before any "fix")

Every failure falls into one of five categories. Identifying the category narrows the fix-path immediately:

| Category | Signal | Where the fix lives |
|---|---|---|
| **Real bug** | Server returns wrong data or 500-class error, manual curl reproduces it | Raise a Jira ticket (after curl repro) |
| **Spec drift** | Test expected one shape, response has a new/missing/renamed field | Run `/qa-api-sync` — the test isn't broken, the API moved |
| **Test setup issue** | A precondition wasn't met (wrong state, missing entity) | Fix the Setup folder; downstream tests will pass once state is right |
| **Environment issue** | Server unreachable, VPN dropped, DNS, expired account | Not a bug. Document and retry. |
| **Flaky test** | Passes 3 of 5 runs without code changes | Log it, do not silently retry. Investigate root cause (timing, race, shared state) |

---

## 2. Read the HTML report properly

The report is at `newman-reports/<project>-report-<timestamp>.html`. Open it in the browser. Order of investigation:

1. **"Failed Tests" tab at the top** — jump straight to failures, skip the passes.
2. **Click into the failed request** — you get four tabs:
   - **Request** — method, URL with rendered variables, headers, body actually sent.
   - **Response** — status code, headers, full response body.
   - **Test results** — which `pm.test(...)` assertion failed and why.
   - **Console** — any `console.log(...)` output from the test script.
3. **Note the timestamp** — if the team has server logs, cross-reference.

The single biggest mistake is judging a failure by status code alone. Always read the response body — APIs often return `{ "error": "Order must be APPROVED before submission", "code": "ORDER_NOT_APPROVED" }` which tells you exactly what went wrong.

---

## 3. Reproduce with curl (PROJECT RULE — never skip)

Non-negotiable per project standards: before raising any ticket at any priority, the failing request must be reproduced manually with curl and its full response body read.

Copy the request from the HTML report verbatim and run:

```bash
curl -X <METHOD> -v \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token-from-report>" \
  -d '<body-from-report>' \
  "<full-url-from-report>"
```

Interpret the result:

- **Curl succeeds (2xx) but suite said failure** → test setup issue (state was different when curl ran vs when suite ran) — check Setup folder.
- **Curl fails the same way as the suite** → likely a real bug OR spec drift. Read the error message; if it mentions a field/shape change, run `/qa-api-sync` and re-test.
- **Curl returns a clearly informative error** ("validation failed: regionId is required") → spec drift, run `/qa-api-sync`.
- **Curl returns 500 with a vague error or stack trace** → real bug, raise a ticket with the curl command + full response in the description.

---

## 4. Common HTTP failure patterns

| Status | Pattern | Most likely cause |
|---|---|---|
| **401 on every request** | Cascade after login failed | Check the first Login request — it didn't save `accessToken`. Look at its response body. Common: wrong creds, login JSON path wrong (check `TOKEN_JSON_PATH` in `.env`), account locked |
| **401 on one specific endpoint** | Cascade is fine elsewhere | Endpoint may need a special role/permission. Check ticket for "admin user" or similar role hints |
| **400 (missing required field)** | Endpoint that worked yesterday | Spec drift — the API added a required field. Run `/qa-api-sync` |
| **404 on Get/Update/Delete with `{{testEntityId}}`** | Earlier List/Create save script failed | Check the upstream request's test script — the ID chaining didn't work. Look for empty `testEntityId` in the environment |
| **409 (conflict / duplicate)** | Happens on re-runs only | A field that should be unique isn't randomly generated per run. Use `{{$randomEmail}}` / `{{$randomUUID}}` / `{{$randomInt}}` (Mode A) so each run uses fresh data |
| **422 (unprocessable / wrong state)** | A state transition is illegal | Setup didn't build the expected precondition state. E.g. trying to approve an entity that's still DRAFT |
| **429 (rate limit)** | Suite ran fine before, suddenly failing | Bump `--delay-request` from 300 to 1000+ ms |
| **500 (server error)** | Vague stack trace in response | Real backend bug. Reproduce with curl, capture the request + response, raise a Jira ticket |
| **Timeout / ECONNRESET** | Hit-or-miss failures | Server cold start, slow infra. Bump `--timeout-request` to 30000. If consistent, check server health |
| **JSONError in test script** | Suite log says "JSONError" | Response wasn't JSON. Probably an HTML error page from a load balancer or proxy. Check the raw response body |
| **`SyntaxError: Identifier already declared`** | Newman startup crash | A `const` slipped into top scope of a test script. Find it and change to `let` |
| **Round-trip assertion fails** (Mode A) | Status 2xx but `expected X got undefined` on a field | API silently dropped a field. Real bug — raise ticket. The round-trip caught what status-only would miss |

---

## 5. Newman flags for deeper digging (manual human use only)

When the HTML report is not enough, re-run Newman with debugging flags. **The agent's auto-fix phases never use `--bail`** — these flags are for the human after auto-fix and the full report exist.

```bash
# Run only one folder
newman run <project>-collection.json -e newman-env.json --folder "00. Authentication"

# Run only a single named request
newman run <project>-collection.json -e newman-env.json --folder "TC11: Submit Order - Already Submitted"

# Verbose: full request + response bodies in console
newman run <project>-collection.json -e newman-env.json --verbose

# Stop at first failure (HUMAN TRIAGE ONLY — useful when chasing a single cascading bug;
# never use this for the full suite, you want the report to be complete)
newman run <project>-collection.json -e newman-env.json --bail

# Increase per-request timeout for slow staging servers
newman run <project>-collection.json -e newman-env.json --timeout-request 30000

# Slow down requests if hitting rate limits
newman run <project>-collection.json -e newman-env.json --delay-request 1000

# Combine: verbose + single failing folder — the "give me everything about this one failure" combo
# (notice: no --bail — the folder is already scoped, and you want every failure in it)
newman run <project>-collection.json -e newman-env.json \
  --folder "<folder>" --verbose
```

**Rule of thumb for `--bail`:** Only use it when narrowing in on ONE specific failing request and you want to stop the moment it fails so you can grab verbose output. For everything else — full suite runs, folder runs, auto-fix re-runs — do not bail.

---

## 6. Decision tree: what to do next

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

---

## 7. What NOT to do

- Do NOT modify a test to make it pass without understanding why it failed. That's how you hide real bugs.
- Do NOT immediately raise a ticket without curl repro — per project rule, all tickets need manual reproduction.
- Do NOT blame the test framework first. Newman/Postman bugs are rare; spec drift and state issues are common.
- Do NOT ignore intermittent failures. A test that fails 1 in 10 times is hiding a real timing/race/state issue and will eventually fail in production.
- Do NOT delete failing tests. If a test is genuinely no longer relevant, archive it (rename to `[ARCHIVED] <name>`) so the history stays visible.

---

## 8. When the failure is actually correct behaviour

Some "failures" are the test correctly catching a known limitation:

- **Negative tests SHOULD fail with their expected error code** — if `TC11: Submit Order - Already Submitted (400)` returns 400, that's a pass, not a fail. Make sure you're reading the report correctly.
- A status assertion that says "expected 200 got 201" might be **spec drift** where the API now correctly returns 201 for creates — update the spec/snapshot via `/qa-api-sync`, not the assertion (changing assertions hides real bugs).
