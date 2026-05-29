---
name: octoperf-auto-correlation
description: Use when an OctoPerf Virtual User imported from a HAR/Postman/JMX recording fails its validation run because dynamic values (session tokens, CSRF, signed URLs, anti-forgery inputs, auth challenges) captured at recording time are stale on replay. Triggers on requests for "auto-correlation", "correlate the VU", "fix replay errors", "401/403 on replay after import", "tokens don't match", "signature mismatch in load test". Walks the LLM through framework preset selection, async polling, and regex-rule fallback. Requires the OctoPerf MCP server to be connected.
---

# OctoPerf — Auto-correlation workflow

When a recorded Virtual User (VU) fails its replay because dynamic
server-issued values become stale, **correlation** is the fix: extract
the new value from a response and inject it into the next request. This
skill drives that loop through the MCP tools without manually editing
the VU.

## When this applies

Apply this skill in one of two modes.

**Reactive** — when **all** of:

- The VU was imported (`import_har_virtual_user`, `upload_jmx_virtual_user`, `import_postman_virtual_user`, …) — not hand-written.
- A `validate_virtual_user` run finished with failures.
- The failures look like dynamic-value mismatches: HTTP 401/403 on the second+ request, server errors about "invalid token", "expired session", "CSRF mismatch", "signature verification failed", "nonce reused", or response bodies referencing IDs/keys that don't match what the request sent.

**Proactive** — validation is clean but a `run_scenario` is planned.
A 1-user validation often masks correlation gaps because JMeter's
`CookieManager` auto-handles session cookies — under concurrent load
the same VU can break (per-session token binding, race conditions,
WAF heuristics). Walk the VU with `get_virtual_user` and look for
hardcoded server-generated values still in place: long base64 / hex
strings, session ids in URL paths (`;jsessionid=...`), hidden form
tokens (`_sourcePage`, `__fp`, `__VIEWSTATE`, …), JWTs in headers.
Apply step 2 (framework presets) to harden, then re-validate.

If failures are about request *bodies* (wrong JSON shape), missing
variables, HTTP server config (wrong timeout / baseUrl), or 5xx server
errors unrelated to identity — this is **not** an auto-correlation
problem; use the validation-triage skill instead.

## Steps

### 0. Snapshot the VU before you rewrite it

Correlation rewrites the VU's action tree **in place**, and OctoPerf
has **no VU versioning** — `apply_correlations_to_virtual_user` and a
bad custom rule are hard to undo. Take a one-call backup first:

```
mcp__octoperf__backup_virtual_user(virtualUserId, label="pre-correlation")
```

This duplicates the VU in the same project (full action tree, recorded
bodies excluded) and tags the copy `backup` + `pre-correlation` so it's
easy to find. If a framework or rule mangles the tree, discard the
working VU and re-correlate from the copy instead of re-importing from
scratch. Skip it only when the VU is itself a throwaway import you can
trivially re-create.

### 1. Confirm the diagnosis on one failing request

Don't skip this. Correlation rewrites your VU; you want to be sure the
fix matches the symptom.

```
mcp__octoperf__get_virtual_user_validation_index(virtualUserId)
```

Pick the *first* failing action whose error category looks like
"identity / state". Then:

```
mcp__octoperf__get_validation_failure_detail(virtualUserId, actionId)
```

This returns the four HTTP entities: the request as **sent**, the
request as **received**, the response as **sent**, the response as
**received**. Look for a value that appears in the previous response
but is missing/stale in this request. Common shapes:

- A `Set-Cookie` header on response N that doesn't show up on request N+1.
- A hidden form input `<input name="__RequestVerificationToken" value="…">` in HTML, whose recorded value is sent back as a form field on the next POST.
- A JSON field `{ "sessionId": "…" }` echoed back in subsequent request bodies or query strings.
- An `Authorization: Bearer <jwt>` recorded at capture time, where the JWT's `exp` is now in the past.

If you spot one, you've confirmed correlation is the right fix.

### 2. Try a framework preset first

Frameworks are curated rule packs that catch the standard correlation
patterns of a given stack. They're the cheapest fix.

```
mcp__octoperf__list_correlation_frameworks()
```

Pick the one matching the target system. The lists below are the
exact extractor names each preset registers — match them against what
you see in the recorded traffic.

- **SAML** — federated SSO. Catches the request / response payloads
  (`SAMLRequest`, `SAMLResponse`), the signature material
  (`Signature`, `SignatureValue`, `DigestValue`, `X509Certificate`),
  assertion metadata (`AssertionID`, `nameid-format`, `IssueInstant`,
  `NotBefore`, `NotOnOrAfter`, `AuthenticationInstant`,
  `correlationId`), the context tokens (`ctx`, `wctx`), and the
  Azure-AD-flavored attributes that often live inside the assertion
  (`objectIdentifier`, `tenantId`, `puuid`, `name`,
  `authnmethodsreferences`). 22 extractors total.
- **OAuth** — OAuth 1.0a-style flows. Catches `oauth_token`,
  `oauth_state`, `oauth_timestamp` (plus the `…1` re-extracted
  variants for double-occurrence in the trace). For OAuth 2.0 with
  PKCE / authorization-code, use **AzureAD** if the IdP is Microsoft,
  otherwise build per-VU rules with `create_correlation_rule`.
- **Token** — generic CSRF / session tokens not tied to a stack.
  Catches `x-csrf-token`, `csrfToken`, `token`, `authtoken`, `pk`
  (with `…1` / `…2` re-extracted variants), and `ReturnURL` echoes.
  9 extractors total. **Note:** the ASP.NET anti-forgery
  `__RequestVerificationToken` is in **.NET**, not here.
- **.NET** — ASP.NET WebForms. Catches `__VIEWSTATE`,
  `__VIEWSTATEGENERATOR`, `__EVENTVALIDATION`, `__PREVIOUSPAGE`,
  `__REQUESTID`, and the anti-forgery `__RequestVerificationToken`,
  with `__VIEWSTATE1` / `__EVENTVALIDATION1` re-extracted variants for
  multi-form pages.
- **Java** — Java-server session bootstrap. Catches `sessionId` only.
  For JSF (`javax.faces.ViewState`) or Spring Security (`_csrf`),
  build per-VU rules with `create_correlation_rule` — they are *not*
  bundled here. **Session cookies** (`JSESSIONID`, `PHPSESSID`,
  `ASPSESSIONID`, …) are auto-handled by JMeter's `CookieManager`
  (it re-injects `Set-Cookie` → `Cookie` for you) — a custom rule is
  only needed when the value also appears in URL paths
  (`;jsessionid=...` URL rewriting fallback), in `Referer` headers,
  or echoed in request bodies.
- **AzureAD** — Microsoft sign-in (`login.microsoftonline.com`,
  `login.live.com`) and AAD-issued OAuth 2.0 / OIDC. Catches
  `Azure_canary`, `Azure_ctx`, `Azure_sFT`, `Azure_sessionId`,
  `Azure_state` (with `Azure_state1` / `Azure_state2` variants),
  `Azure_token`, `Azure_client_id`, `Azure_code`,
  `Azure_code_challenge`, `Azure_nonce`, `Azure_session_state`.
  13 extractors total.

**Stacks not bundled** — Stripes (`_sourcePage`, `__fp`), Wicket
(`wicket:interface`, `_wicket-...`), Tapestry (`t:formdata`), JSF
(`javax.faces.ViewState`), Spring Security (`_csrf`), or any custom
in-house framework. No preset matches these. Skip step 2 entirely
and go straight to step 4 (custom regex rules).

Frameworks compose: applying **SAML** then **Token** to the same
project is safe — `add_correlation_framework_to_project` dedupes structurally
(it normalises `id` / `userId` / `projectId` away and skips any rule
already present), so re-applying or stacking only creates the rules
that are missing. When in doubt, apply the one with the heaviest
match first (SAML or AzureAD) then layer Token / .NET on top.

Then apply it. The preset is applied to a **project**, not a single
VU — it bulk-creates the missing rules in the project's rule library:

```
mcp__octoperf__add_correlation_framework_to_project(projectId, frameworkId)
```

This is **synchronous** — it returns the list of rules that were
actually created (already-present rules are silently skipped). No
task to poll.

The newly-created project rules are not yet wired into the failing
VU's action tree. Walk the VU once to materialise extractors /
injections from the rule library:

```
mcp__octoperf__apply_correlations_to_virtual_user(projectId, virtualUserId)
```

### 3. Re-validate

```
mcp__octoperf__validate_virtual_user(projectId, virtualUserId, providerId, location, iterations=1)
mcp__octoperf__get_virtual_user_validation(projectId, virtualUserId)  # poll until finished=true
```

Use `octoperf-async-polling` for the polling cadence — validation
runs are short (~30s) so a 5s sleep between `get_virtual_user_validation`
calls is plenty.

If failures drop to zero — done. If they shrink but some remain, the
preset caught most patterns and the rest need a custom rule (next
step). If they're unchanged or worse, the preset was the wrong fit —
revisit step 1 with a different framework.

### 4. Custom regex rule for what the preset missed

For each remaining failure, re-read the failure detail (`get_validation_failure_detail`)
and locate the dynamic value's *source* (the response it should be
extracted from) and *target* (where it should be injected).

```
mcp__octoperf__create_correlation_rule(projectId, name, regex, ...)
```

Rule design tips:

- The regex should capture **exactly one group** — the dynamic value.
- Anchor against stable surrounding text. `name="csrf" value="([^"]+)"` is more robust than just `([a-f0-9]{64})`.
- Test the regex mentally against the recorded response body in the failure detail — if it doesn't match there, it won't match at replay either.
- Name the rule for the stack it covers, e.g. `myapp-csrf-token`, not `rule1`.
- **Avoid variable names starting with `__`** (double underscore) — they collide with JMeter's function syntax `${__functionName(args)}`. For example, the Stripes form field `__fp` should be captured into a variable named `fp` (or `flashParam`); the POST parameter name stays `__fp` (server-side requirement), only the JMeter variable is renamed.

**Picking `injectionTargets`** — match each *target* (where the
captured value should reappear in subsequent requests) to one of:

| Where the value appears next                           | `injectionTargets`                    |
|--------------------------------------------------------|---------------------------------------|
| Hidden form input on HTML POST                         | `POST_PARAM`                          |
| Header value (`Authorization: Bearer`, `X-CSRF-Token`) | `HEADER`                              |
| URL path segment (`;jsessionid=...`, REST id)          | `PATH` (+ `HEADER` if also in Cookie) |
| Query string (signed URL, `?state=...`)     c          | `QUERY_PARAM`                         |
| Body of JSON / XML request                             | `BODY`                                |

Multiple targets are valid: a JSESSIONID that's URL-rewritten **and**
echoed in `Referer` headers needs `["PATH", "HEADER"]`.

Then re-walk the VU to apply the new rule:

```
mcp__octoperf__apply_correlations_to_virtual_user(projectId, virtualUserId)
mcp__octoperf__get_task_result(taskId)  # poll — see octoperf-async-polling (3s cadence)
```

Before burning a validation cycle, optionally confirm the rule
actually matched by reading the VU tree and grepping for
`${variableName}`:

```
mcp__octoperf__get_virtual_user(virtualUserId)
```

If the variable doesn't appear at the expected injection sites, the
regex didn't match the recorded response — fix it before re-validating.
(`get_virtual_user` can exceed the LLM token budget on large VUs; in
that case strip the VU first or delegate the scan to a sub-agent.)

### 5. Iterate, then stop

Loop steps 3-4 until validation is clean. Two stop conditions:

- **Clean:** zero failures on a 2-3 iteration validation run. Surface the result to the user and offer to `run_scenario`.
- **Plateau:** if the same failure persists after two rule attempts, the issue likely isn't correlation. Stop, summarize what was found, and hand back to the user.

## Pitfalls

- **Don't run a load test to debug correlation.** `validate_virtual_user` is the right tool; it's a 1-user run with full HTTP capture. `run_scenario` is destructive (consumes credits) and gives you metrics, not request/response bodies.
- **Frameworks DO compose** — `add_correlation_framework_to_project` dedupes structurally (normalises `id`/`userId`/`projectId`, then set-skips already-present rules), so applying SAML *and* Token only creates the missing rules. Stacking is the intended way to cover multi-stack flows; you don't have to pick exactly one.
- **Watch for credentials in rule regexes.** A rule that captures a literal password is a leak. Always capture *generated* values, never input ones.
- **Correlation can't fix a wrong variable.** If the value the user originally supplied (login, account id, …) is wrong for the target environment, no rule will recover it — the value is wrong at recording time. Surface that to the user.

## See also

- `octoperf-validation-triage` — when validation has many failures of mixed kinds.
- `octoperf-async-polling` — sleep cadence for `get_virtual_user_validation` and `get_task_result`.
- OctoPerf correlation docs: <https://doc.octoperf.com/design/correlations/>
