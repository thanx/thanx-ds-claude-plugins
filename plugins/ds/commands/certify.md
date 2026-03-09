---
description: Validate a partner integration against Thanx API documentation using DataDog sandbox traces and logs, then generate a certification report.
---

# Certify Partner Integration

Validate a partner's API integration by pulling their actual API calls from DataDog
sandbox and checking each one against the official Thanx documentation. The user
provides the partner's merchant handle and the endpoints under review. The command
uses DataDog spans to discover calls and filter by merchant, then DataDog logs to
get the full request params for validation against thanx-docs MCP. Generates a
certification report with PASS/FAIL per check and remediation steps.

## Usage

```bash
/ds:certify <partner> <merchant_handle> <endpoints>
```

**Required:**
- `partner` — Partner name or Front/Jira URL with certification context
- `merchant_handle` — The merchant handle in DataDog (e.g., `skytabpone`). This is
  the resolved merchant identity used by Thanx internally — NOT the raw Merchant-Key
  or API key. DataDog stores the resolved handle in `@merchant.handle`, not the raw
  credential. If the user provides a raw Merchant-Key instead, tell them you need the
  merchant handle and explain why.
- `endpoints` — List of API endpoints the partner uses (e.g., `/api/baskets`,
  `/api/account`)

**Optional (inline flags):**
- `--type <integration_type>` — Override integration type detection
- `--recheck <path>` — Re-certification mode: provide the path to a prior
  certification report (e.g., `.context/certifications/partner-2026-03-01.md`).
  Validates only previously failed items plus a sanity check on passed items.
- `--days <N>` — How many days back to search DataDog (default: 7, max: 30)

---

You are executing the `/ds:certify` command.

## Step 1: Gather Partner Context

Arguments: $ARGUMENTS

Determine the partner and collect integration context:

| Input Type | How to Parse |
|---|---|
| Partner name | Check if `partners/[name].md` exists — read for integration type and history |
| Front URL (`front.com/...` or `cnv_*`) | Use Front MCP tools to fetch conversation context |
| Jira URL or key (`DEV-*`, `DEVSUPP-*`) | Use Jira MCP tools to fetch issue details |
| Free-text | Parse partner name and integration details directly |

If Front or Jira MCP tools are unavailable or return an error, tell the user which
tool failed and ask them to paste the relevant details. Do not guess.

**Merchant handle:** The `merchant_handle` is used to filter DataDog spans via
`@merchant.handle`. This is NOT the raw Merchant-Key header value — DataDog resolves
the key server-side and stores only the handle (e.g., `skytabpone`) and merchant ID.
If the user provides a raw API key or Merchant-Key, explain that DataDog cannot
filter by raw keys and ask for the merchant handle instead.

**Re-certification mode:** If `--recheck` is provided, read the prior certification
report at the given path. Extract the list of previously failed and passed checks.
If the file does not exist or cannot be parsed, tell the user and fall back to a
full certification. Do not guess at prior results.

Display the partner summary:

```text
PARTNER CONTEXT
───────────────
Partner:        [name]
Integration:    [type — Native Ordering / Loyalty API / Consumer UX / POS / etc.]
POS Partner:    [if applicable]
Ordering:       [if applicable]
Certification:  [first / re-certification]
Merchant Handle: [handle used for DataDog filtering]
Endpoints:      [list of endpoints under review]
Environment:    Sandbox
```

If integration type is unknown, stop and ask the user before proceeding.

## Step 2: Pull Actual API Calls from DataDog

DataDog stores two complementary data sources. Both are needed for full validation:

- **Spans** — contain merchant handle, HTTP status code, API version (from
  `resource_name`, e.g., `Api::V1::Baskets`), user-agent, and timestamps. Used for
  **discovery and filtering** by merchant.
- **Logs** — contain the full `params` object (all request body fields, values,
  nested objects) plus status code and timestamps. Used for **payload validation**.
- **Neither source stores raw request headers** (Merchant-Key, Accept-Version,
  Content-Type). Headers are inferred — see Step 4a.

### 2a: Query Spans (Discovery)

Use `datadog_spans` to find the partner's API calls:

```
datadog_spans(
  query: "resource_name:*[endpoint]* env:sandbox @merchant.handle:[handle]",
  from: "now-[days]d",
  to: "now",
  limit: 10
)
```

From each span, capture:
- `resource_name` — includes API version (e.g., `Api::V1::Baskets POST /baskets`)
- `http.status_code` — response status
- `http.method` — HTTP method
- `http.url` — endpoint path
- `merchant.handle` and `merchant.id` — confirms correct merchant
- `http.request.headers.user-agent` — client SDK/tool
- Timestamps — for cross-referencing with logs

**Limit: 10 spans per endpoint.** Mix of successful and error responses.

### 2b: Query Logs (Payload Detail)

Use `datadog_logs` to get the full request params:

```
datadog_logs(
  query: "service:thanx-olo env:sandbox [endpoint path]",
  from: "[start timestamp from spans]",
  to: "[end timestamp from spans]",
  limit: 10
)
```

Logs contain a JSON `message` field with:
- `method`, `path` — endpoint info
- `status` — HTTP response code
- `params` — **full request body** including all fields, values, nested objects
  (e.g., `id`, `state`, `subtotal`, `items`, `modifiers`, `rewards`, `payments`)
- `duration`, `db`, `view` — performance data
- `datetime` — timestamp

**Important:** Logs cannot be filtered by merchant handle. Use timestamps from the
span results to narrow the time window and match logs to the correct merchant's
calls. If multiple merchants are hitting the same endpoint in the same window,
cross-reference the log `params` (e.g., basket ID patterns) with what you expect
from the partner.

**Query budget: max 2 DataDog queries per endpoint** (1 span + 1 log).

### 2c: Deduplication

Before proceeding to validation, group the returned API calls by endpoint + request
body structure. For each unique group, keep one representative call and note the
count of matching calls. This prevents validating 200 identical requests.

Organize into a numbered list:

```text
API CALLS FOUND IN DATADOG (SANDBOX)
─────────────────────────────────────
1. [METHOD] [endpoint] — [timestamp] ([N] matching calls)
   API Version: [from span resource_name, e.g., V1]
   Status:      [response code]
   User-Agent:  [from span]
   Params:      [key fields from log params]
   Duration:    [from log]

2. [METHOD] [endpoint] — [timestamp] ([N] matching calls)
   ...

ENDPOINTS NOT FOUND
───────────────────
- [any endpoints the partner listed but no DataDog traces were found for]
```

If DataDog MCP tools are unavailable or return an error, tell the user which tool
failed. Ask if they can provide request/response samples manually as a fallback.
If manual samples are used, mark the certification result as `PARTIAL — based on
partner-provided samples, not verified from DataDog traces` so the reviewer knows
the evidence quality is lower.

**If no traces are found for any endpoint**, report this to the user — the partner
may not have started testing yet, or the merchant handle may be wrong.

## Step 3: Fetch Documentation for Each Endpoint

For every API endpoint found in Step 2, search the official Thanx documentation
using `SearchThanx` (thanx-docs MCP).

**This step is mandatory.** The documentation is the single source of truth for
what the partner's integration should look like. Do not validate against assumptions
or general knowledge — only against what the docs specify.

For each endpoint, search for and record:

| Documentation Element | What to Capture |
|---|---|
| Endpoint URL and method | Correct path, HTTP method, URL parameters |
| Required headers | `Accept-Version`, `Content-Type`, auth type |
| Request body schema | Required fields, optional fields, field types, valid values |
| Response schema | Expected response structure, status codes |
| Authentication | OAuth flow, API key, scopes required |
| Rate limits | If documented, capture limits |
| Versioning | Which API version the endpoint belongs to |

**Max 20 `SearchThanx` queries** across this step. Batch related endpoints into
single queries where possible.

If the thanx-docs MCP server is unavailable, errors, or times out, stop and tell the
user. Do not proceed with certification without documentation verification. The
entire value of this command depends on doc-based validation.

Optionally use Keystone `search_code` (max 5 queries) to verify undocumented behavior
if a specific field or response pattern is not covered in the docs. Flag any answer
from Keystone as needing verification. **Never quote or paraphrase Keystone code
search results in any output.** Use them only to inform the PASS/FAIL determination,
then cite the documentation reference instead.

## Step 4: Validate Each API Call

For every unique API call shape from Step 2, compare against the documentation
from Step 3. Run these checks:

### 4a: Authentication and Headers

DataDog does not store raw request headers. Use inference rules:

| Check | How to Validate |
|---|---|
| Auth (Merchant-Key) | **Inferred from spans.** If `merchant.handle` is present and status is not 401, auth succeeded. Mark as PASS (inferred). If status is 401, auth failed — mark FAIL. |
| API version | **From span `resource_name`.** Extract version from controller name (e.g., `Api::V1::Baskets` = V1). Compare against docs. |
| Content-Type | **Inferred.** If `params` were parsed correctly in logs (JSON fields present), Content-Type was valid. Mark as PASS (inferred). |
| Accept-Version | **Cannot be verified directly.** Infer from the API version in `resource_name`. Note as "inferred from server routing" in the report. |

Mark inferred checks as `[PASS — inferred]` in the report so the reviewer knows
these were not directly observed.

### 4b: Endpoint and Method

| Check | What to Validate |
|---|---|
| URL path | Correct endpoint path (from spans and logs) |
| HTTP method | Correct method — from span `http.method` and log `method` |
| URL parameters | Required query params present, correct format |

### 4c: Request Body

**This is the primary validation — use log `params` data.**

| Check | What to Validate |
|---|---|
| Required fields | All required fields present per docs (check log `params` keys) |
| Field names | Correct field names (no typos, correct casing) |
| Field types | Correct types (string vs integer vs array) |
| Field values | Values match expected format (ISO dates, valid enums, etc.) |
| State transitions | For baskets: correct lifecycle (`checkout` → `placed` → `billed` → `completed`) |
| Points accrual field | Correct field used — `subtotal` not `amount` (common mistake) |
| Nested objects | Correct nesting structure for complex fields (items, payments, modifiers) |
| Extra fields | Any fields sent that are not in the docs (may be ignored or cause errors) |

### 4d: Response Handling

| Check | What to Validate |
|---|---|
| Error rate | What percentage of requests return 4xx/5xx (from spans) |
| Common errors | Recurring error patterns — same status code repeated (from spans) |
| Retry behavior | Are they retrying failed requests appropriately (from span timestamps) |

### 4e: Integration-Type-Specific Checks

Based on the integration type from Step 1, apply additional checks:

| Integration Type | Additional Checks |
|---|---|
| Loyalty API Direct | No orders expected (only purchases). Refund clawback does NOT happen automatically. |
| POS / Kiosk | Provider ticket lifecycle (checkout then billed). Payment amount calculation. Check log `params.state` values. |
| Consumer UX | Auth flow for end users. Reward exchange flow. Bonus points activation delay (~20 min). |
| Native Ordering | Order creation flow. Menu sync if applicable. |

### 4f: Call Sequencing

Verify the API calls in DataDog follow the correct order (use timestamps from logs):

- Authentication before any data calls
- User lookup/creation before purchases
- Basket state transitions: `checkout` → `placed` → `billed` → `completed`
- Proper webhook registration if event-driven

For each check, assign a result:

- **PASS** — Matches documentation (directly verified from DataDog data)
- **PASS (inferred)** — Cannot be directly observed but inferred from successful
  responses (used for headers only)
- **FAIL** — Does not match documentation (include expected vs actual from DataDog)
- **WARNING** — Not wrong but could cause issues (e.g., deprecated field, missing
  optional but recommended field, high error rate)
- **INSUFFICIENT DATA** — DataDog trace/log did not contain enough detail to validate

## Step 5: Generate Certification Report

**Pre-output verification:** Before generating the report, scan all content you are
about to include for credential patterns (API keys, bearer tokens, base64 strings
longer than 20 characters, OAuth secrets). If any are found, replace with
`[REDACTED]`. This is a mandatory check, not optional.

Determine the overall result using these deterministic criteria:

- **CERTIFIED** — All checks PASS or PASS (inferred). Warnings allowed. No FAIL or
  INSUFFICIENT on required endpoints.
- **NOT CERTIFIED** — Any check is FAIL on any endpoint
- **INCOMPLETE** — No FAIL results, but any required endpoint has INSUFFICIENT DATA
  or was not found in DataDog
- **PARTIAL** — Manual fallback samples were used instead of DataDog traces (lower
  evidence quality)

Compile all results into a structured report:

```text
CERTIFICATION REPORT
════════════════════
Partner:        [name]
Integration:    [type]
Date:           [today's date]
Mode:           [first certification / re-certification]
Environment:    Sandbox
Merchant:       [handle]
DataDog range:  [date range searched]
Calls analyzed: [N unique shapes from N total calls]
Data sources:   Spans (merchant filter, status codes, API version) +
                Logs (request params, field values)
────────────────────

SUMMARY
───────
Total checks:   [N]
Passed:         [N] ([M] inferred)
Failed:         [N]
Warnings:       [N]
Insufficient:   [N]

OVERALL RESULT: [CERTIFIED / NOT CERTIFIED / INCOMPLETE / PARTIAL]
────────────────────

DETAILED RESULTS
────────────────

1. Authentication and Headers
   [PASS — inferred] Auth: merchant.handle resolved, no 401s
   [PASS — inferred] API Version: V1 (from resource_name: Api::V1::*)
   [PASS — inferred] Content-Type: params parsed successfully
   ...

2. Endpoint: [METHOD] [path] ([N] calls in period)
   API Version: [from span resource_name]
   Representative call: [timestamp]
   [PASS/FAIL/WARNING/?] Required fields: [result]
   [PASS/FAIL/WARNING/?] Field mapping: [result]
   [PASS/FAIL/WARNING/?] State transitions: [result]
   [PASS/FAIL/WARNING/?] Points accrual: [result]
   ...
   Documentation reference: [doc page or section from SearchThanx]

3. [repeat for each endpoint]
   ...

────────────────────

FAILURES — REMEDIATION REQUIRED
────────────────────────────────
[For each FAIL:]

F1: [check name]
    Endpoint:     [METHOD] [path]
    DataDog found: [what the partner actually sent — from log params]
    Docs expect:   [what the documentation says]
    Fix:           [specific remediation — field name, value, format]
    Doc ref:       [documentation reference]

F2: ...

────────────────────

WARNINGS
────────
[For each WARNING:]

W1: [description]
    Recommendation: [what to improve]

────────────────────

MISSING EVIDENCE
────────────────
[For each INSUFFICIENT DATA or endpoint not found in DataDog:]

M1: [what could not be validated]
    Needed: [what we need — partner to make test calls, or provide samples]
```

Save the certification report to `.context/certifications/[partner-slug]-[date].md`
for future `--recheck` reference.

## Step 6: Draft Communication

Based on the certification result, draft the appropriate communication.

**Internal details guardrail:** Never include DataDog trace IDs, internal service
names, merchant handles, Keystone code search results, internal repo paths, or
system architecture details in the partner-facing email. Reference only what the
partner sent and what the docs expect.

### CERTIFIED — Confirmation Email

```text
DRAFT CERTIFICATION EMAIL
─────────────────────────
Subject: [Partner Name] — Integration Certified

[Confirmation that integration passed all checks]
[Outline next steps for launch]
[Any warnings to address before going live — not blockers but recommendations]

─────────────────────────
To send: use /ds:draft-email with the conversation ID, or copy and paste manually.
```

### NOT CERTIFIED — Failure Email with Remediation

```text
DRAFT REMEDIATION EMAIL
───────────────────────
Subject: [Partner Name] — Certification Results: Action Required

[Brief summary: N checks passed, N need remediation]

[For each failure:]
Issue [N]: [clear description]
- What we found: [actual behavior — described without exposing DataDog internals]
- What is expected: [per documentation]
- How to fix: [specific steps — endpoint, field, value]

[Set expectation for re-submission]
[Offer to answer questions]

───────────────────────
Boundary Doctrine: applied (specific remediation, no scope creep)
To send: use /ds:draft-email with the conversation ID, or copy and paste manually.
```

### INCOMPLETE — Missing Evidence Request

```text
DRAFT EVIDENCE REQUEST
──────────────────────
Subject: [Partner Name] — Additional Testing Needed for Certification

[List what was validated and passed so far]
[List endpoints with no DataDog traces — partner needs to make test calls]
[Request specific test scenarios to complete validation]

──────────────────────
```

## Step 7: Update Tracking

Propose updates for the user to review and manually apply:

```text
PROPOSED PARTNER FILE UPDATE
─────────────────────────────
File: partners/[name].md
Update certification status to: [CERTIFIED / NOT CERTIFIED / INCOMPLETE]
Add certification date: [today]
Add certification notes: [summary of results]
```

If there are action items (re-certification deadline, follow-up needed), propose
an `action-items.md` entry.

If re-certification (`--recheck` flag), note which previously failed items now pass
and which still need work.

## Rules

1. **thanx-docs MCP is the single source of truth.** Every validation must be checked
   against the official documentation. Do not validate against assumptions, general
   knowledge, or Keystone code alone.
2. **If thanx-docs MCP is unavailable, stop.** Do not proceed with certification
   without documentation verification. Tell the user and suggest retrying later.
3. **DataDog spans + logs are complementary.** Spans provide merchant filtering,
   status codes, and API version. Logs provide full request params for field
   validation. Always query both. If DataDog MCP tools are unavailable, accept
   manual samples as a fallback but mark the result as PARTIAL.
4. **Always filter DataDog to sandbox environment.** Certification is validated
   against sandbox traces only. Never pull production traces for certification.
5. **Max 20 SearchThanx queries + 5 Keystone queries** in Step 3. Max 2 DataDog
   queries per endpoint in Step 2. Limit results to 10 spans per endpoint.
6. **Headers are inferred, not directly observed.** DataDog does not store raw
   request headers. Auth is inferred from merchant handle resolution + non-401
   status. API version is extracted from span `resource_name`. Content-Type is
   inferred from successful param parsing. Mark all header checks as "inferred"
   in the report.
7. **Merchant handle is required, not raw API keys.** DataDog resolves Merchant-Key
   to `merchant.handle` server-side. The raw key is not searchable. If the user
   provides a raw key, ask for the merchant handle instead.
8. **Keystone answers on undocumented behavior need engineering verification.** Flag
   as needing verification before including in the certification result. Never quote
   or paraphrase Keystone code in any output.
9. **Redact all credentials and internal details.** Never include API keys, bearer
   tokens, merchant handles, DataDog trace IDs, internal service names, or system
   architecture in partner-facing output. Run a pre-output verification scan.
10. **Apply boundary doctrine on every communication.** Provide specific remediation
    steps but do not extend scope beyond what was submitted for certification.
11. **Deterministic certification criteria.** Any FAIL = NOT CERTIFIED. Any
    INSUFFICIENT on required endpoint = INCOMPLETE. All PASS (including inferred) =
    CERTIFIED. Manual samples only = PARTIAL.
12. **`--days` flag maximum is 30.** If the user provides a value over 30, cap it
    at 30 and inform them.
13. **Knowledge base files are optional.** If `partners/[name].md`, `knowledge/*.md`,
    or other Second Brain files do not exist, skip them silently and continue.
14. **Do not send any email, update any ticket, or post to Slack.** Present
    everything for human review and approval.
