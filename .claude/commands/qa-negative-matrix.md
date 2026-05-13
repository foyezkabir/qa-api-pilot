---
name: qa-negative-matrix
description: "Shared reference for the negative test matrix. Sourced by /qa-api-test-setup, /qa-test-ticket, /qa-api-sync, and /qa-negative-audit. Defines the 24 negative-coverage rows, the condition under which each row applies, the expected status, the Postman test script template, and the target sub-folder."
---

# Negative Test Matrix — canonical reference

This file is the **single source of truth** for what negative tests the agents generate. The four commands read from this matrix:

| Command | When it applies the matrix |
|---|---|
| `/qa-api-test-setup` | Phase 5c — first-time build, applies matrix per endpoint, distributes into sub-folders |
| `/qa-test-ticket` | Phase 4d — applies matrix scoped to the ticket's endpoints, keeps flat Negative folder |
| `/qa-api-sync` | Phase 4a — when an ADDED endpoint comes in, generates the full applicable matrix for it |
| `/qa-negative-audit` | The whole reason it exists — diffs current coverage against this matrix, fills gaps |

**Coverage rule (hard):** every endpoint that meets a row's condition MUST have a test for that row. The agent does not "decide mentally" what negatives are worth writing — it walks every row, checks the condition, and either writes the test or marks the row as `n/a (condition not met)`. No row is silently skipped.

---

## The matrix

Each row below has: a stable `id` the agents use in the coverage report, the **condition** under which the row applies, the **expected status / behavior**, the **Postman test script template**, and the **target sub-folder** in the main collection (ticket collections are flat).

**Resolved test name format**: the agent assembles each negative test's display name as:

```
TC<NN>: <op.summary> - <row description> (<status>)
```

Example: row pattern `No Auth (401)` applied to `POST /api/v1/auth/login` (summary: `Login`) → `TC07: Login - No Auth (401)`.

The `<op.summary>` prefix is added at build time (resolved per the fallback chain in CLAUDE.md § Happy Path folder: summary → Title-Cased operationId → `<METHOD> <path>` last resort). Rows below contain only the `<row description> (<status>)` portion — no method, no path, no leading dash.

### Auth Failures sub-folder

| id | Condition | Test name pattern | Expected | Test script |
|---|---|---|---|---|
| `auth-no-token` | Endpoint has `security` defined and uses bearer or apiKey | `No Auth (401)` | 401 | `pm.test('No auth rejected', () => pm.response.to.have.status(401));` — request has no `Authorization` header |
| `auth-expired-token` | Endpoint uses JWT bearer | `Expired Token (401)` | 401 | Same as above but `Authorization: Bearer <a-known-expired-JWT>`. Use `{{expiredAccessToken}}` env var (preseed with a JWT well past its exp). |
| `auth-tampered-token` | Endpoint uses JWT bearer | `Tampered Token (401)` | 401 | `Authorization: Bearer <current accessToken with one char flipped in payload>`. Use `{{tamperedAccessToken}}`. |
| `auth-malformed-token` | Endpoint uses JWT bearer | `Malformed Token (401)` | 401 | `Authorization: Bearer not.a.real.jwt` |
| `auth-wrong-role` | Endpoint requires a specific role OR resource is owned by a user (path param like `{userId}`, `{tenantId}`) | `Wrong Role (403)` | 403 | Use a `{{altUserAccessToken}}` for a different user/role. Skip if no role discriminator is detectable from spec; flag as `n/a (no role info in spec)` in audit. |

### Validation Failures sub-folder

| id | Condition | Test name pattern | Expected | Test script |
|---|---|---|---|---|
| `val-missing-required` | Endpoint has a body with required fields | `Missing <field> (400)` | 400 | Body omits one required field per test (one test per required field). |
| `val-wrong-type` | Body has a typed field (string/integer/boolean) | `<field> Wrong Type (400)` | 400 | Send a string where an integer is expected, an integer where a string is expected, etc. One test per field with a swappable type. |
| `val-out-of-range` | Body has min/max/minLength/maxLength constraints | `<field> Out of Range (400)` | 400 | Send `max + 1` or `min - 1`, `''` for `minLength > 0`, very-long string for `maxLength`. |
| `val-regex-mismatch` | Body has a `pattern` (regex) constraint, e.g. email/phone | `<field> Bad Format (400)` | 400 | Send a value that violates the regex (e.g. `notanemail` for an email field). |
| `val-enum-out-of-range` | Body or query has `enum` constraint | `<field> Bad Enum (400)` | 400 | Send a value not in the enum list. |

### Resource Errors sub-folder

| id | Condition | Test name pattern | Expected | Test script |
|---|---|---|---|---|
| `res-not-found` | Endpoint has a path param like `{id}`, `{userId}` | `Non-existent <param> (404)` | 404 | Use a well-formed but non-existent ID (`{{nonExistentId}}` env var, e.g. `00000000-0000-0000-0000-000000000000`). |
| `res-wrong-state` | Spec mentions state transitions OR endpoint name contains lifecycle verb (`/submit`, `/approve`, `/cancel`, `/publish`) | `Wrong State (422)` | 422 | Call lifecycle endpoint on a resource that is already in a terminal state. Setup must create a resource in the "wrong" state for this test. |
| `res-conflict` | Endpoint creates a resource with uniqueness constraint (e.g. unique email) | `Duplicate (409)` | 409 | First call succeeds (in Setup), second call with same unique value → 409. |

### Security Probes sub-folder

| id | Condition | Test name pattern | Expected | Test script |
|---|---|---|---|---|
| `sec-sql-injection` | Body or query has any string field | `SQL Injection Probe (rejected)` | 400/422 or sanitised | Payload: `' OR '1'='1`, `'; DROP TABLE users; --`, `1 UNION SELECT NULL`. One test per likely-SQL-bound field (names, emails, IDs as strings). Assert response code is in `[400, 422]` AND response body does NOT contain SQL error fragments (`syntax error`, `unterminated string`, `psql`, etc.). |
| `sec-xss-reflection` | Body or query has any string field | `XSS Probe (not reflected)` | 4xx or sanitised | Payload: `<script>alert(1)</script>`, `<img src=x onerror=alert(1)>`. Assert response body does NOT contain the literal `<script>` or `onerror=` (i.e. value was either rejected or escaped). |
| `sec-path-traversal` | Endpoint has a path parameter OR a file-related query param | `Path Traversal Probe (rejected)` | 400/404 | Payload in path param: `../../etc/passwd`, `..%2F..%2Fetc%2Fpasswd`. Assert non-2xx response and response body does not contain `/etc/passwd` content. |
| `sec-rate-limit` | `INCLUDE_RATE_LIMIT_TESTS=true` in `.env` AND endpoint has documented rate limit (or applies to all endpoints if no spec info) | `Rate Limit (429)` | 429 | Pre-request script loops N rapid calls (default 100) within 1 second; final request asserts `pm.response.code === 429`. **Destructive — skipped by default.** |
| `sec-cors-preflight` | All endpoints (one CORS test per endpoint, or one per origin) | `CORS Preflight` | response allows only declared origins | Send `OPTIONS` with `Origin: https://evil.example.com`. Assert response does NOT include `Access-Control-Allow-Origin: *` or `Access-Control-Allow-Origin: https://evil.example.com`. |

---

## Body data sources (where field values come from)

Test bodies for `POST`, `PUT`, `PATCH` need actual field values. The agent uses **two modes** depending on what's being built:

### Mode A — Random gen + round-trip assertion (positive tests only)

Used for **Happy Path - All Endpoints** (the smoke pass — but see exception below) AND **per-module Positive tests**.

**Pre-request script** generates random values respecting OpenAPI constraints for each user-supplied field, sends the request, then **post-response script** asserts the response echoed back the same values per field. Catches "API silently dropped my field" regressions that spec-example-driven tests miss.

For each field in the request body, follow this priority chain when picking the value:

| # | Source | When it wins |
|---|---|---|
| 1 | **Chained env var** (`{{testCustomerId}}`, `{{accessToken}}`) | Field is a reference to an existing resource — must be a real ID, never random |
| 2 | **Ticket-specific value** (`/qa-test-ticket` only) | The Jira AC mentions a concrete value (e.g. "submit with amount 500") |
| 3 | **Postman dynamic random** matching field type/format | `{{$randomEmail}}` for `format: email`, `{{$randomUUID}}` for `format: uuid`, `{{$randomFullName}}`, `{{$randomLoremSentence}}`, `{{$randomInt}}`, etc. — constrained-aware where the spec gives `minLength` / `maxLength` / `minimum` / `maximum` / `pattern` |
| 4 | **`enum` first value or random pick** | Field constrained to enum — pick randomly from the list |
| 5 | **Spec `example` / `default`** | Last-resort fallback when no random gen is sensible (rare; usually only for fields that MUST be a specific value, like a known correlation ID) |

**Skip these fields** — do not send them, do not assert echo:
- Server-generated read-only fields (`id`, `createdAt`, `updatedAt`, `slug`, etc. — typically marked `readOnly: true` in the spec)
- Sensitive fields the API should NOT echo (`password`, `apiKey`, etc.) — assert their **absence** from response instead, as part of `xcut-sensitive-leak`

**Pre-request script (per endpoint, generated at build time):**
```js
// Build the body using the source chain
let sentBody = {
  customerId: pm.environment.get("testCustomerId"),                  // source 1: chained
  amount:     500,                                                    // source 2: from ticket (if applicable)
  notes:      pm.variables.replaceIn("{{$randomLoremSentence}}"),     // source 3: random string
  priority:   ["LOW","MEDIUM","HIGH"][Math.floor(Math.random() * 3)], // source 4: enum random pick
  // Skipped fields: id, createdAt (readOnly), password (sensitive)
};

// Stash for post-response assertion
pm.environment.set("__sentBody", JSON.stringify(sentBody));

// Inject into request
pm.request.body.raw = JSON.stringify(sentBody);
```

**Post-response script (round-trip assertions per user-supplied field):**
```js
pm.test("Status is 2xx", () => pm.expect(pm.response.code).to.be.oneOf([200, 201, 202, 204]));

let sent = JSON.parse(pm.environment.get("__sentBody"));
let got  = pm.response.json().payload?.data;  // adjust JSON path per project

// Round-trip each user-supplied field
for (let k of Object.keys(sent)) {
  pm.test(`${k} round-trip`, () => pm.expect(got[k]).to.eql(sent[k]));
}

// Server-generated fields: assert presence + format, not value
pm.test("id present", () => pm.expect(got.id).to.be.a("string"));
pm.test("createdAt is ISO-8601", () => pm.expect(got.createdAt).to.match(/^\d{4}-\d{2}-\d{2}T/));

// Chain new ID for downstream requests
if (got.id) pm.environment.set("testOrderId", got.id);
```

**Exception — Happy Path - All Endpoints**: keep this folder fast and noise-free. Use Mode A's pre-request gen + the chain, but for the assertions phase: **status check + schema validation only**, no per-field round-trip. (Round-trip rigor lives in the per-module Positive folder.)

### Mode B — Deterministic source + mutation (negative tests only)

Used for **Negative sub-folders** (`Auth Failures`, `Validation Failures`, `Resource Errors`, `Security Probes`).

Negatives need **known bad values** — random won't produce `' OR '1'='1` or `quantity = max + 1`. So:

1. Build the **valid baseline body** using Mode A's source chain (without the random gen — just take the first valid value per field: enum first item, spec example, type-default).
2. Apply the **matrix row's mutation** to that body (e.g. drop one required field for `val-missing-required`, swap one field's type for `val-wrong-type`, inject `' OR '1'='1` into one string field for `sec-sql-injection`).
3. Assert the response code matches the matrix row's expected status. No round-trip assertion (the request was supposed to fail).

The mutation rules per row are defined in the matrix table above. Each row that produces multiple tests (e.g. `val-missing-required` loops over required fields) generates the variants from the same baseline body, mutating one field per variant.

### What happens for GET / DELETE / HEAD / OPTIONS

No body needed. Path params come from the chained env (source 1). Query params either constants from the spec's `default` / `enum[0]` or random within constraints (Mode A).

### Fallback when no source applies

Rare edge case — a field with no `enum`, no `example`, no `default`, no `format`, no `pattern`, no chained env match, no type-default applicable.

The agent uses a generic placeholder (`"test-string"`, `0`, `false`) and **prints a warning at the end of build**: `N fields fell back to generic placeholders — review collection-run-issues/<slug>-coverage-<ts>.txt § Fallback Placeholders and fix manually before running suite.` Better to flag explicitly than to silently produce a value that might violate a constraint and tank a happy-path test.

---

## Cross-cutting assertions (not separate tests — added INTO existing tests)

These do not get their own test cases. They are extra `pm.test(...)` blocks added to the test scripts of **existing positive and negative tests**.

| id | Where it goes | Assertion |
|---|---|---|
| `xcut-sensitive-leak` | Every response (positive + negative) | `pm.test('No sensitive data in response', () => { let body = pm.response.text(); pm.expect(body).to.not.match(/"password"\s*:\s*"[^"]+"/i); pm.expect(body).to.not.match(/"apiKey"\s*:\s*"[^"]+"/i); pm.expect(body).to.not.match(/-----BEGIN [A-Z ]+PRIVATE KEY-----/); });` |
| `xcut-error-body-shape` | Every negative test (4xx/5xx) | `pm.test('Error body has structure', () => { let b = pm.response.json(); pm.expect(b).to.have.any.keys('code', 'errorCode', 'error'); pm.expect(b).to.have.any.keys('message', 'detail', 'description'); });` — Tolerant of field name variants. |
| `xcut-no-stack-trace` | Every 5xx response | `pm.test('No stack trace leaked', () => { let body = pm.response.text(); pm.expect(body).to.not.include(' at '); pm.expect(body).to.not.match(/Traceback \(most recent call last\)/); pm.expect(body).to.not.match(/Exception in thread/); });` |
| `xcut-response-time` | Every test (positive + negative) | `pm.test('Response time under SLA', () => { pm.expect(pm.response.responseTime).to.be.below(parseInt(pm.environment.get('RESPONSE_TIME_SLA_MS') || '500')); });` Tier-classify: < 200 ms = Excellent, 200–500 = Good, 500–1000 = Acceptable, 1000–3000 = Slow, > 3000 = Critical. |
| `xcut-schema-validation` | **Happy Path** + every per-module Positive test (2xx response) | `pm.test('Response matches schema', () => { let schema = <inlined OpenAPI 2xx response schema>; pm.response.to.have.jsonSchema(schema); });` Use `tv4` (built into Postman). Schema is inlined at build time from the spec's `responses.<2xx>.content.application/json.schema`. **Happy Path uses schema validation but no per-field round-trip** — that's the lightweight smoke-pass profile. Per-module Positive uses both. |
| `xcut-idempotency` | PUT, DELETE, HEAD, OPTIONS (and GET implicitly) | A second test (suffixed `b`) immediately after the primary, calling the same endpoint again, asserting the response is consistent: same body for PUT/GET, 404 (or same idempotent result) for DELETE on already-deleted resource. |

---

## Sub-folder distribution (main collection only)

Inside each module's `Negative` folder, the four sub-folders are created upfront, even if some end up empty:

```
00. Auth
└── Negative
    ├── Auth Failures        ← rows: auth-no-token, auth-expired-token, auth-tampered-token, auth-malformed-token, auth-wrong-role
    ├── Validation Failures  ← rows: val-missing-required, val-wrong-type, val-out-of-range, val-regex-mismatch, val-enum-out-of-range
    ├── Resource Errors      ← rows: res-not-found, res-wrong-state, res-conflict
    └── Security Probes      ← rows: sec-sql-injection, sec-xss-reflection, sec-path-traversal, sec-rate-limit, sec-cors-preflight
```

Each sub-folder name describes the **category of negative test** stored in it (the failure type), applied to the **parent module's endpoints**. So `01. Orders > Negative > Auth Failures` holds the auth-failure tests *for Orders endpoints* — not tests of the Auth module.

TC numbering is continuous across all four sub-folders. If `Negative > Auth Failures` ends at TC09, `Negative > Validation Failures` starts at TC10 — single counter per module.

Ticket collections keep `Negative` flat (no sub-folders) since the test count is small.

---

## Audit / coverage reporting

When `/qa-negative-audit` runs, it walks every endpoint and produces a report keyed by matrix `id`:

```
POST /api/v1/orders
  ✓ auth-no-token        TC07
  ✓ auth-expired-token   TC08
  ✓ auth-tampered-token  TC09
  ✓ auth-malformed-token TC10
  ✗ auth-wrong-role      missing
  ✓ val-missing-required TC11 (supplierId), TC12 (items)
  ✗ val-wrong-type       missing
  ✗ val-out-of-range     missing
  ✗ val-regex-mismatch   n/a (no regex constraints in spec)
  ✗ val-enum-out-of-range missing
  ✓ res-conflict         TC14
  ✗ sec-sql-injection    missing
  ✗ sec-xss-reflection   missing
  ✗ sec-rate-limit       skipped (INCLUDE_RATE_LIMIT_TESTS=false)
  Coverage: 6/13 applicable rows
```

Rows marked `n/a` count as covered (condition not met). Rows marked `skipped` do not count against coverage but are listed so the gap is visible. Rows marked `missing` count against coverage.

---

## Implementation notes for the agents

1. **Generate one test per applicable field, not one test per row.** `val-missing-required` for an endpoint with 5 required body fields produces 5 tests (one per omitted field), not 1.
2. **Setup dependencies.** Tests like `res-conflict` and `res-wrong-state` depend on Setup creating a fixture in the right state. The agent inserts the necessary Setup steps automatically when the matrix demands them.
3. **Env vars used by the matrix (must exist in `newman-env.json`):**
   - `expiredAccessToken` — a known-expired JWT. Seeded during setup (the agent generates one with `exp` set to past).
   - `tamperedAccessToken` — current `accessToken` with one char of the payload section flipped. Re-derived per Newman run via a pre-request script in the Setup folder.
   - `altUserAccessToken` — an access token for a different user/role. Asked for once during setup if `auth-wrong-role` is going to apply anywhere.
   - `nonExistentId` — fixed value, e.g. `00000000-0000-0000-0000-000000000000`.
   - `RESPONSE_TIME_SLA_MS` — default `500`, overridable per project.
   - `INCLUDE_RATE_LIMIT_TESTS` — `true` / `false`. Asked once during setup.
4. **The matrix is the contract.** When a row's test script needs to change globally (e.g. a new XSS payload), edit it HERE. All commands then re-read this file on their next run.
