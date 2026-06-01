---
name: octoperf-validation-triage
description: Use when an OctoPerf Virtual User validation run has produced many failing actions and the user needs to diagnose them efficiently without reading every single failure serially. Triggers on "the validation is red", "lots of errors after import", "VU validation failed, what's wrong", "triage these failures", "why is my virtual user failing". Groups failures by category, drills into one representative per group, and proposes the matching MCP-tool fix. Requires the OctoPerf MCP server.
---

# OctoPerf — Validation-failure triage

When a Virtual User validation finishes with many failing actions,
reading each failure detail one by one wastes context window and time.
This skill groups failures by **root cause**, drills into one
representative per group, and maps each group to the correct fix.

## When this applies

- A `validate_virtual_user` run finished with `finished=true` and at least one failing action.
- The user wants to know *what's wrong*, not just retry blindly.

## Steps

### 1. Get the failure index — no bodies yet

```
mcp__octoperf__get_virtual_user_validation_index(virtualUserId)
```

The index returns one entry per **validated** HTTP action — `actionId`,
`success`/`total` counts, `successTimestamps` and `failedTimestamps`. For
triage, focus on the entries whose `failedTimestamps` is non-empty (the
actions that failed at least one run); entries with only `successTimestamps`
passed every run. **Do not fetch failure details yet** — the index alone is
usually enough to classify.

(A `successTimestamps` entry is also a handle to read a *passing* run's
body: pass it to `fetch_validation_http_body` with
`kind=VALIDATION_RESPONSE` — e.g. to inspect what a Debug-style action
actually captured on a green run.)

OctoPerf's "KO" rule isn't just "non-2XX". A sample is KO when:

| Live response code | Recorded response code | Result                                                           |
|--------------------|------------------------|------------------------------------------------------------------|
| 2XX                | none                   | ✅ OK                                                             |
| 2XX                | 2XX                    | ✅ OK                                                             |
| 2XX                | 3XX                    | ❌ KO (recording expected a redirect, got a body — usually wrong) |
| 3XX                | 3XX                    | ✅ OK                                                             |
| 3XX                | 2XX                    | ❌ KO (recording expected a body, got a redirect)                 |
| Any 4XX / 5XX      | anything               | ❌ KO                                                             |
| Unknown code       | anything               | ❌ KO                                                             |
| Any code           | 4XX / 5XX / unknown    | ❌ KO (recording was already broken; re-record)                   |

This matters when classifying: a "successful" 200 against a recorded 302 is a
real bug, not a false positive — the VU is hitting a different code path than
when it was recorded.

**Caveat — ResponseAssertion overrides the matrix.** A `ResponseAssertion`
attached to the action can mark a sample KO even when the matrix above
says OK (e.g. status 200 against recorded 200, but the body contains
the assertion's pattern — typical for forms that re-render with an
error message on a 200). Check for assertions on the failing action
before assuming an HTTP-level mismatch. The assertion's `type` field
also controls scope:

- `REQUEST_ONLY` — only the parent sample is checked.
- `REQUEST_AND_SUBREQUESTS` — parent **and** embedded resources. Matters when `downloadResources=true`: a 404 on an embedded CSS can fail the parent.
- `SUBREQUESTS_ONLY` — only embedded resources.

### 1b. Partial failures (`success < total`)

When the index entry shows `success < total` (e.g.
`{"success": 2, "total": 3, "failedTimestamps": [...]}`), the action
failed on **some iterations** but passed on others. Common patterns:

- **Bad row in a CSVVariable** — one iteration picks a row that
  doesn't pass server-side validation (e.g. a wrong-password row,
  a malformed account id, a stale token). Cheapest possible fix:
  edit the CSV. Investigate this **before** correlation when the VU
  uses a `CSVVariable`.
- **Race condition** — uncommon in 1-user validation but possible if
  the VU has internal `LoopContainerAction` or shared variables.
- **CSV exhausted with EOF = StopVU** — iterations past the CSV length
  never run; not strictly a failure but `success < total` reflects it.

`failedTimestamps` lists every failed iteration's epoch-ms. Pass the
**exact** timestamp to `get_validation_failure_detail` to pin the
failing iteration instead of getting a random one — critical when
debugging CSV-driven flakiness so you see the bad row, not a good one.

### 2. Group by category

Bucket the failing actions into a small number of categories. The
common ones:

| Category                  | Index signal                                                                                                                                      | Likely fix                                                                                                                                                                              |
|---------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Auth / state**          | 401, 403; "invalid token", "expired", "CSRF"                                                                                                      | Auto-correlation (separate skill)                                                                                                                                                       |
| **Variable / data**       | 400 with body validation errors; "field required", "invalid format"                                                                               | Edit / create variables; check CSV upload                                                                                                                                               |
| **HTTP server config**    | Connection timeout, DNS failure, SSL handshake error, wrong port                                                                                  | `update_http_server` (baseUrl, timeouts, IP spoofing)                                                                                                                                   |
| **Server-side 5xx**       | 500, 502, 503, 504                                                                                                                                | Not the VU's fault — surface to user; check target env                                                                                                                                  |
| **Body mismatch**         | 422; "schema mismatch", "unexpected field"                                                                                                        | Re-import (recording out of date) or edit action body                                                                                                                                   |
| **Assertion failure**     | Status 2XX/3XX matching recorded, but failure message quotes the assertion's pattern (often on a `ResponseAssertion` node attached to the action) | Read `validationResponse.body` to see the matched substring/regex. Either the response is genuinely wrong (fix the upstream cause) or the assertion's pattern is too strict (loosen it) |
| **Missing dependency**    | 404 on resources, signed URLs returning errors                                                                                                    | Run a correlation rule, or the resource genuinely doesn't exist                                                                                                                         |

A useful heuristic: if 80% of failures fall in one category, fix that
first and re-validate before investigating the rest. Most of the long
tail clears once the dominant root cause is resolved.

### 3. Confirm each category against one representative

For each group, pick the **first** failing action and fetch its detail:

```
mcp__octoperf__get_validation_failure_detail(virtualUserId, actionId)
```

The detail returns the four HTTP entities (sent/received request and
response). For very large bodies, you can pull a single one with
`mcp__octoperf__fetch_validation_http_body(...)`. Its `kind` parameter
(`RECORDED_REQUEST` / `RECORDED_RESPONSE` / `VALIDATION_REQUEST` /
`VALIDATION_RESPONSE`) lets you isolate the recorded-vs-replay diff
side-by-side — useful when the request/response is several MB and a
single side fits in your context but not both.

Read each detail with a specific question in mind based on the
category:

- **Auth/state:** is there a token in the *previous* response that should have been re-sent here? → Auto-correlation skill.
- **Variable/data:** does the request body contain a literal value that should be a `${variable}` reference? → `list_variables` → `create_*_variable` → re-import or edit.
- **HTTP server config:** is the request reaching the server at all? Timing out? Hitting the wrong host? → `list_http_servers_by_project` → `update_http_server`.
- **Body mismatch:** does the recorded request body match what the API actually expects today? If not, the recording is stale.
- **Assertion failure:** is the matched pattern (e.g. `"Signon failed"` substring) a symptom of an upstream wrongness (auth failed, validation error re-rendered in the page) or an overly-strict assertion? Fix the upstream cause first; loosen the assertion's pattern only if the response is genuinely correct.
- **Missing dependency:** is the requested resource (`/api/orders/12345`) one that the recording created earlier, but whose id is now stale? → correlation, or use a CSV variable.

### 3b. Inspect live variable state with a DebugAction

When the category is **Auth/state** or **Variable/data**, the decisive
question is usually *"what value did the variable actually hold at this
point in the flow?"* — did an extractor capture nothing (empty), capture
the wrong substring, or capture the right value that the server then
rejected? The HTTP entities only show what was *sent*; they don't show
the variable table.

Insert a `DebugAction` (JMeter's Debug Sampler) right after the action
that *populates* the variable, then re-validate and read its body:

```
mcp__octoperf__patch_virtual_user(virtualUserId, ops=[{
  "op": "add",
  "path": "/children/<index>",
  "value": {
    "@type": "DebugAction",
    "id": "<fresh-uuid>",
    "enabled": true,
    "variablesDisplayed": true
  }
}])
```

Two payload gotchas: the body dumps the variable table **only** when
`variablesDisplayed: true` — omit it and the response comes back empty.
And `DebugAction` requires a non-null `id` (supply a fresh UUID) and has
**no** `name` field — don't add one. `propertiesDisplayed` /
`systemPropertiesDisplayed` stay `false` unless you also want the JMeter
property dump.

A `DebugAction` always succeeds, so it never shows up in
`failedTimestamps` — find its run under `successTimestamps` in the index,
then pull its body:

```
mcp__octoperf__get_virtual_user_validation_index(virtualUserId)      # grab the DebugAction's successTimestamp
mcp__octoperf__fetch_validation_http_body(
  projectId, virtualUserId, debugActionId, timestamp, kind=VALIDATION_RESPONSE)
```

The body is a dump of every JMeter variable in scope at that point:

```
JMeterVariables:
JMeterThread.last_sample_ok=true
slideTitle=Sample Slide Show
slideTitle_matchNr=1
...
```

How to read it:

- **`var=<value>` + `var_matchNr=1`** → the extractor matched once and
  captured the value. The correlation works; if the request still fails,
  the *value* is being rejected (stale, wrong scope), not missing.
- **`var=` empty, or the variable absent entirely** → the extractor
  matched nothing. The regex/JSONPath is wrong or runs against the wrong
  response. → fix the extractor (`patch_virtual_user`) or the correlation rule.
- **`var_matchNr` > 1** → multiple matches; the wrong occurrence may be
  injected downstream. Pin the match number.

Remove the `DebugAction` once diagnosed (`patch_virtual_user` with an
`op: remove`) — it's a diagnostic probe, not part of the script.

### 4. Apply ONE fix, then re-validate

Resist the urge to fix three things at once. Apply the fix that should
clear the largest category, then:

```
mcp__octoperf__validate_virtual_user(projectId, virtualUserId, providerId, location, iterations=1)
mcp__octoperf__get_virtual_user_validation(projectId, virtualUserId)  # poll — see octoperf-async-polling
```

Re-fetch the failure index. Verify:

- **Cleared:** the target category is gone. Move on to the next.
- **Reduced:** some failures of the same category cleared, others didn't. The fix was partial — refine it (e.g. add a more specific correlation rule).
- **Unchanged or worse:** the fix was wrong; revert it (`update_*` or `delete_*`) before trying another.

### 5. Engine-level failure (no index, no HTTP entities)

If `get_virtual_user_validation_index` comes back empty but the run is
still marked failed/aborted, the validation engine itself crashed
(JMeter OOM, missing Playwright dependency, bad locale, …) — there are
no HTTP samples to read. (An empty index now means *no validated
samples at all*: a VU that ran and passed lists every action with only
`successTimestamps`, so a truly empty result points at the engine, not
a clean pass.) The validation run produces a `benchResultId`
(returned by `validate_virtual_user`) which backs the same log storage
as a real bench run, so:

```
mcp__octoperf__list_bench_result_files(benchResultId)
mcp__octoperf__read_bench_result_file_lines(benchResultId, "jmeter.log")
```

surfaces the engine logs (and any Playwright trace / screenshot / HAR
the engine left behind). Read the tail of `jmeter.log` first — startup
errors are usually within the first 50 lines and fatal errors within
the last 50.

**Log retention.** Validation log files are erased **7 days after the
run**, or as soon as the user leaves the design screen. Old validation
runs may no longer have logs — call out the freshness window before
promising a re-read.

For **binary artefacts** (Playwright `trace.zip`, screenshots `.png`,
HAR archives) the line-based reader returns garbage. Use the
binary-aware tool instead:

```
mcp__octoperf__download_bench_result_file(benchResultId, filename)
# returns { url, method: "GET", expiresAt, instructions }
```

GET `url` directly with your code interpreter (single-use token,
valid ~5 minutes) and inspect the bytes locally (e.g. `unzip -p
trace.zip trace.trace`). This is especially valuable for Playwright
VU failures — the JMeter wrapper log only sees the spawn, the actual
selector miss / timeout / navigation abort lives in the trace.

### 6. Stop conditions

Stop and surface to the user when:

- Zero failures → VU is ready. Offer to `run_scenario`.
- All remaining failures are 5xx → it's the target environment, not the VU. Hand back with the list.
- After two rounds of fixes the failure count plateaus → there's a structural issue (wrong recording, target API changed, …). Hand back with the diagnosis.

## Sanity-check output reference

`sanity_check_virtual_user` runs *before* the first validation. Its output is
a flat list of `(level, message)`. ERROR entries block validation, WARNING /
INFO entries don't. Mapping the canonical messages to fixes:

| Level   | Message                                   | What it means                                          | Fix                                                                           |
|---------|-------------------------------------------|--------------------------------------------------------|-------------------------------------------------------------------------------|
| ERROR   | A file is missing for CSV variable        | A CSVVariable points at a file not uploaded            | `upload_project_file`, or `patch_virtual_user` to repoint the variable        |
| ERROR   | CSVVariable has conflicting column names  | Two CSVVariables share a column name                   | Prefix one variable's columns; `patch_virtual_user`                           |
| ERROR   | JSR223Action is empty                     | Empty script generates only noise logs                 | Delete the action or fill the script (`patch_virtual_user`)                   |
| ERROR   | No Server Found                           | A request points at a deleted HTTP server              | `list_http_servers_by_project` → recreate or repoint via `update_http_server` |
| ERROR   | Cyclic Dependency Detected!               | A fragment references itself (directly or indirectly)  | Break the cycle with `patch_virtual_user`                                     |
| WARNING | Clear Cookies before recording …          | Recorded cookies may leak invalid session ids          | Remove `Cookie` headers in the relevant requests                              |
| WARNING | Empty file for CSVVariable                | The uploaded CSV parsed to zero rows                   | Re-upload a properly-encoded UTF-8 file                                       |
| WARNING | End Of Value Policy is 'Stop VU'          | Test will end abruptly when the CSV is exhausted       | Confirm with user; otherwise switch policy to Recycle / Continue              |
| WARNING | file is missing for POST request          | A multipart POST references a file not in `/resources` | `upload_project_file` (no path prefix; OctoPerf adds `/resources/` itself)    |
| INFO    | Host header and server host are differing | Some servers reject mismatched Host headers            | Search-and-replace the Host header if the target rejects                      |
| INFO    | Using a JMeter generic action             | Imported a raw JMeter element; double-check its config | None unless behavior is unexpected — JAR plugins go under `/lib/ext`          |
| INFO    | XXX sec thinktime is high                 | A recorded pause is unusually long                     | Trim the thinktime if the duration would harm the test                        |
| INFO    | xxxxx should have a name                  | Unnamed element renders as "Unnamed" in reports        | Rename via `patch_virtual_user`                                               |
| INFO    | xxxxx is empty                            | Empty controller / logic action that won't execute     | Delete it                                                                     |
| INFO    | HTTP Action has empty query parameter     | Imported a stray query param with no name / value      | Remove the parameter via `patch_virtual_user`                                 |

Apply the ERROR fixes first — validation is blocked until those clear.

## Anti-patterns

- **Don't fetch failure details for every action.** The index is enough to classify; details are for confirmation, not bulk inspection.
- **Don't `run_scenario` to debug.** Validation is the right tool — it's cheap, captures full HTTP. A load test gives you metrics, not bodies.
- **Don't sanity-check after validating.** `sanity_check_virtual_user` is a static check; run it *before* the first validation run, not after. If it would have caught the issue, you wasted a validation cycle.
- **Don't edit the VU silently.** Summarize what fix you're about to apply and confirm with the user before any `delete_*` or destructive change. For irreversible tree edits (`patch_virtual_user`, applying correlations), snapshot first with `backup_virtual_user(virtualUserId, label="pre-triage")` — there's no VU versioning to undo a bad patch.
- **Don't omit `enabled` when adding an action.** When a `patch_virtual_user` op *adds* a node to `children`, every boolean defaults to `false` if absent — so an action added without `"enabled": true` is created **disabled** and silently does nothing at run time, with no validation error. Always set `"enabled": true` explicitly on inserted actions. The JSON field is `enabled`, **not** `isEnabled`.

## See also

- `octoperf-auto-correlation` — for the "auth / state" category.
- `octoperf-scenario-diagnosis` — for diagnosing problems that appear under load but not in validation.
- `octoperf-async-polling` — sleep cadence and terminal conditions for `get_virtual_user_validation` / `get_bench_result`.
