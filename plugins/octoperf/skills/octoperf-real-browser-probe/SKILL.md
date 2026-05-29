---
name: octoperf-real-browser-probe
description: Use when the user wants to run a real-browser probe alongside a JMeter HTTP load test to capture user-perceived metrics (page load time, render time, JS execution, Core Web Vitals) while JMeter generates the bulk HTTP load. Triggers on "real browser monitoring during load test", "EUM probe", "playwright probe", "synthetic monitor during bench", "convert my JMeter VU to Playwright", "RealBrowser user", "TruClient equivalent", "hybrid load test (HTTP + browser)". Walks the LLM through JMeter→Playwright VU conversion (direct translation or codegen capture) and hybrid scenario composition (N×JMeter for load + 1×Playwright probe for UX measurement). Requires the OctoPerf MCP server.
---

# OctoPerf — Real-browser probe alongside JMeter load

## The pattern

JMeter HTTP virtual users are great for server-side metrics (response
time, throughput, error rate) but they don't reflect what a real user
*perceives*: page load, JS execution, rendering, layout shifts, Core
Web Vitals. A **real-browser probe** is a single Playwright VU that
runs the same user journey through an actual Chromium during the load
test — measuring UX while the JMeter pool keeps the server busy.

Commercial equivalents (the LLM will recognise these terms): NeoLoad
*RealBrowser User*, LoadRunner *TruClient*, Gatling *Browser User*,
k6 *browser* module, BlazeMeter *Real Browser Users*. The pattern is
also known as **synthetic browser probe** or **end-user experience
monitoring during load test**.

## When this applies

The user has:

- A working OctoPerf JMeter Virtual User (validated clean) that they
  want to load-test.
- The need for **user-perceived metrics** during the test, not just
  server-side HTTP metrics — typically because the front-end is
  JS-heavy (SPA, React/Vue/Angular) and HTTP timings miss the
  client-side render cost.
- Optionally, an SLA framed around browser metrics (e.g. "TTFB < 200 ms
  AND LCP < 2.5 s under 50 concurrent users").

If the user just wants HTTP load (no UX measurement) → stick to
`run_scenario` on the JMeter VU. If the user wants *only* browser
testing without a load backdrop → just create the Playwright VU
without the JMeter one.

## Steps

### 1. Decide: direct translation vs codegen capture

Read the source JMeter VU's action tree:

```
mcp__octoperf__get_virtual_user(virtualUserId)
```

Classify the flow:

| Flow shape                                                                   | Path                                                                       |
|------------------------------------------------------------------------------|----------------------------------------------------------------------------|
| Linear navigation + simple forms (≤ 20 HTTP actions)                         | Step 2 (direct translation)                                                |
| Auth widgets, OAuth popups, file uploads, iframe-heavy SPA, JS-driven canvas | Step 2-bis (`playwright codegen`)                                          |
| Mixed: most simple, a few complex steps                                      | Direct translation, leave `// TODO codegen` markers for the complex blocks |

Direct translation is fast (no manual capture) but **brittle on
dynamic UIs**: a single-page app that doesn't change the URL between
clicks will defeat `page.goto(...)`. When in doubt, codegen.

### 2. Direct translation (JMeter → Playwright spec)

Walk the JMeter action tree and emit one Playwright statement per
action. Translation rules:

| JMeter element                                | Playwright equivalent                                                      |
|-----------------------------------------------|----------------------------------------------------------------------------|
| `HttpRequestAction` GET                       | `await page.goto(url)`                                                     |
| `HttpRequestAction` POST (form-urlencoded)    | `page.fill('input[name="x"]', value)` + `page.click('input[type=submit]')` |
| `HttpRequestAction` POST (JSON / multipart)   | `page.evaluate(() => fetch(...))` OR rebuild via real DOM interactions     |
| `${variable}` (JMeter CSV/Counter/Random)     | JS variable, `process.env.X`, or Playwright fixture                        |
| `ResponseAssertion` (BODY contains/not)       | `await expect(page.locator('body')).toContainText(...)`                    |
| `ThinktimeConstant`                           | `await page.waitForTimeout(ms)` (use sparingly)                            |
| `JSESSIONID` correlation rule                 | **Delete** — the browser handles cookies natively                          |
| Stripes `_sourcePage` / `__fp` correlation    | **Delete** — the browser submits the live form                             |
| `__VIEWSTATE` correlation                     | **Delete** — same                                                          |
| `LoopContainerAction` (N iterations)          | `for (let i = 0; i < N; i++) { ... }`                                      |
| `IfContainerAction` (condition)               | `if (cond) { ... }`                                                        |

**Most correlation rules become noise** in the Playwright translation
— the real browser submits the live form, sends the real cookies, and
echoes the real hidden inputs. Strip them. Keep only correlations that
extract a value the *user* visibly types (rare).

Minimal Playwright project layout — **keep it config-less**:

```
playwright-probe/
├── package.json
└── tests/
    └── user-journey.spec.ts
```

Why no `playwright.config.ts`? OctoPerf's importer treats every `.ts` /
`.js` file as a `PlaywrightSpecAction`, **including the config file**.
The engine then invokes the config as a test sampler at run time —
which fails (no `test()` defined) and inflates the failure count.

Disabling the config-as-spec (via `patch_virtual_user`,
`/children/<i>/enabled = false`) is a workaround, but it has a
second-order consequence: the config file is no longer written to
disk at run time → `use.baseURL` is lost → `page.goto('/')` becomes
`Cannot navigate to invalid URL`. The cleanest fix is to **skip the
config entirely** and put what you need in the spec:

- Use **absolute URLs** everywhere: `page.goto('https://target/path')`, never `page.goto('/path')`.
- Set headless / timeouts via `test.use({ ... })` at the top of the spec, not via `playwright.config.ts`.

Example spec (translated from a petstore-like JMeter VU):

```ts
import { test, expect } from '@playwright/test';

test('petstore browse + login probe', async ({ page }) => {
  const username = process.env.PETSTORE_USER ?? 'j2ee';
  const password = process.env.PETSTORE_PASSWORD ?? 'j2ee';

  await page.goto('https://petstore.octoperf.com/');
  await page.goto('https://petstore.octoperf.com/actions/Catalog.action?viewCategory=&categoryId=FISH');
  await page.click('a[href*="productId=FI-SW-01"]');
  await page.click('a[href*="itemId=EST-1"]');
  await page.click('a[href*="addItemToCart"]');

  await page.goto('https://petstore.octoperf.com/actions/Account.action?signonForm=');
  await page.fill('input[name="username"]', username);
  await page.fill('input[name="password"]', password);
  await page.click('input[name="signon"]');

  await expect(page.locator('body')).not.toContainText('Signon failed');
});
```

Drop the spec under `tests/`. Skip `playwright.config.ts` (see the
config-less layout note above) — put any per-test config via
`test.use({ headless: true, ... })` inside the spec itself.

### 2-bis. Codegen capture (alternative)

When direct translation would be brittle (heavy SPA, OAuth popup,
custom widget), capture the journey live with Playwright codegen:

```sh
npx playwright codegen https://target-url.com
```

The user performs the journey in the launched browser; codegen emits
the corresponding spec file. Save it under `tests/user-journey.spec.ts`
in the project layout above. **Do not** edit the codegen output to
re-add correlations or cookies — the browser handles them natively.

Codegen output may include brittle selectors (`xpath=...`,
auto-generated names); review and replace with stable selectors
(`data-testid`, `role=`) before committing.

### 3. Import the Playwright VU

The MCP server accepts a **single `.spec.ts` file** and packages it
into a VU. Helpers, fixtures and any `package.json` are added in a
second step via `patch_virtual_user`.

`import_playwright_virtual_user` is a presigned upload tool: it mints
a short-lived URL and returns it; the client then POSTs the spec
directly to the OctoPerf REST host (bypassing the MCP server for the
bytes). The REST response is a raw `VirtualUser` JSON without a UI
deep-link — chain into `describe_virtual_user` to get the compact
listing.

```
upload = mcp__octoperf__import_playwright_virtual_user(projectId)
# POST the spec bytes to `upload.url` as multipart/form-data, single
# part named `file`, Content-Type `text/plain; charset=utf-8`. Read
# the returned VU's `id`, then:
mcp__octoperf__describe_virtual_user(virtualUserId)
```

If the client can't perform the direct POST (no network, sandbox
restrictions, file too big for the backend), tell the user to upload
the spec through the OctoPerf web UI rather than failing the task.

### 4. Validate the Playwright VU standalone

```
mcp__octoperf__validate_virtual_user(projectId, virtualUserId,
                                     providerId, location,
                                     iterations=1)
mcp__octoperf__get_virtual_user_validation(projectId, virtualUserId)
```

Poll until terminal. Playwright validation is slower than JMeter
(launching Chromium = ~10 s warmup). The validation captures a HAR
file and screenshots under the run's `benchResultId` — list them with
`list_bench_result_files`.

**Debugging the failure: read the trace.zip, not just the log.** The
JMeter docker log only sees the JMeter-side wrapper around `npx
playwright test`; the actual error (selector miss, timeout, navigation
abort) is in the Playwright **trace.zip** named
`<actionId-prefix>-<hash>-<test-name>-<browser>.trace.zip`. Pull it
with the binary-aware tool:

```
mcp__octoperf__download_bench_result_file(benchResultId, traceFilename)
# returns { url, method: "GET", expiresAt, instructions }
```

GET `url` directly with your code interpreter (single-use token,
valid ~5 minutes), then unzip — `trace.trace` (newline-delimited JSON
of every action) and the screenshots inside give you the per-step
view. If the token has been consumed or expired, call the tool again
to mint a fresh URL.

Common Playwright-specific failures:

- `Timeout 30000ms exceeded waiting for selector` — selector is wrong; codegen-captured `xpath=...` are common culprits.
- `Target page closed` — popup window swallowed the action; add `context.on('page', ...)` handling.
- `Browser was not launched` — Playwright deps missing in the load-generator image; the agent log will name the missing OS package.
- `Cannot navigate to invalid URL` on a `page.goto('/...')` — the spec relies on `use.baseURL` from a `playwright.config.ts` that wasn't applied (either you skipped the config per the layout note, or it was disabled to avoid the spec-import trap). Fix: use absolute URLs in the spec.
- `playwright.config.ts` imported as a spec → run-time failure on a "no tests in this file" sampler. Fixable via `patch_virtual_user` (`/children/<i>/enabled = false` on the config's `PlaywrightSpecAction`), but **disabling the config also strips `baseURL` / timeouts at run time** — relative URLs in the spec will then break (see above). Better: import the project without a config in the first place.

### 5. Compose the hybrid scenario

The hybrid pattern uses **two UserProfiles** in one scenario:

- UserProfile A: the JMeter VU, ramped to N concurrent users for load.
- UserProfile B: the Playwright VU, **pinned to 1 user** (real
  browsers are CPU-heavy; the OctoPerf engine caps Playwright /
  WebDriver UserProfiles at 1 concurrent user per profile).

If you want **multiple** browser probes (e.g. one per region) use
**multiple UserProfiles**, each with 1 Playwright user, not one
profile scaled to N.

Create the base scenario with the JMeter VU first:

```
mcp__octoperf__create_scenario_ramp_up(
  scenarioName='hybrid-load',
  ...)
```

Then patch the scenario to add the Playwright UserProfile. Read the
scenario schema for the userProfiles structure
(`octoperf://schema/scenario`) and use `patch_scenario` to append a
second UserProfile with the Playwright VU id and a constant 1-user
load shape. The Playwright UserProfile's load should be
**simultaneous** with the JMeter one (same start/end times) so the
probe measures UX *during* the load, not before or after.

### 6. Pre-flight: verify a plan can host the run

Before `run_scenario`, check that the user's subscriptions can sustain
the hybrid scenario (the real-browser cap is its own dimension —
e.g. a plan with `maxConcurrentUsers=100` but `maxRealBrowserUsers=0`
won't run the probe):

```
mcp__octoperf__get_scenario_matching_plans(scenarioId)
```

Non-empty list = the scenario is launchable as-is on the listed plans.
Empty list = no plan can host the run; flag the user and stop.

When the result is empty, call `list_active_subscriptions` to surface
all usable plans and their caps — typical hybrid-blockers are
`maxRealBrowserUsers=0` (basic plans reject any Playwright profile)
or `maxProfilesPerScenario<3` (free/trial plans reject the 3-profile
mix). Report the binding cap to the user so they can adjust the
scenario or upgrade.

### 7. Run and read the metrics side-by-side

```
mcp__octoperf__run_scenario(scenarioId)
mcp__octoperf__get_bench_status(benchResultId)   # poll until 1.0
```

In the resulting `get_bench_report`, both VUs show up — but their
metrics tell different stories:

- **JMeter VU metrics**: server response times, throughput, error
  rate. Read p95 / p99 from `get_report_summary_values`. This is the
  load story.
- **Playwright VU metrics**: per-action timings include browser-side
  cost (DOM, paint, JS). The action labelled `page.goto(...)` reports
  the full *user-perceived* time, not just TTFB. Compare its p95
  against the JMeter equivalent for the same URL — the delta is the
  client-side cost (rendering, JS execution, blocking resources).

For Core Web Vitals (LCP, FID, CLS), the basic Playwright VU doesn't
capture them — they need a custom spec using `page.evaluate(() =>
performance.getEntriesByType('navigation'))` or a `web-vitals`
package import. Mention this to the user if they ask for those metrics
specifically.

## Pitfalls

- **Never scale a Playwright UserProfile beyond 1 user.** The engine
  enforces it, but the LLM might still ask. Real browsers consume
  ~250 MB / ~1 vCPU each — at scale they bottleneck the load
  generator and skew *everyone's* timings (cross-contamination with
  the JMeter pool).
- **Don't port correlation rules to the Playwright spec.** The
  browser handles cookies, hidden form inputs, CSRF tokens natively.
  Carrying over JMeter regex rules is dead weight at best, a source of
  bugs at worst.
- **Don't `page.waitForTimeout(thinktime_ms)` blindly.** JMeter's
  thinktime simulates user pause; Playwright's `waitForTimeout` does
  the same but inflates the end-to-end probe duration. Prefer
  `page.waitForLoadState('networkidle')` for navigation, `waitForTimeout`
  only when you want to mimic user dwell time.
- **Don't compare Playwright p95 with JMeter p95 directly without
  caveat.** Browser-side timings include DOM build, JS execution,
  paint — JMeter's don't. The difference *is* the value, but it has
  to be framed correctly when reported to the user.
- **Don't run the Playwright probe on an under-provisioned load
  generator.** Chromium needs CPU + memory headroom; on a saturated
  LG, the probe's own measurements become noise. Pin the probe to a
  dedicated LG or use a cloud provider with enough resources.
- **Don't ship a `playwright.config.ts` in the imported project.**
  OctoPerf imports every `.ts` / `.js` as a `PlaywrightSpecAction`,
  so the config becomes a no-test sampler that fails at run time.
  Disabling that sampler then drops `use.baseURL` and timeouts —
  every relative URL in the real spec breaks with `Cannot navigate
  to invalid URL`. Two traps, one root cause: skip the config file
  entirely and use absolute URLs + per-spec `test.use({ ... })`.

## See also

- `octoperf-validation-triage` — when the Playwright VU itself fails
  to validate.
- `octoperf-scenario-diagnosis` — when the hybrid run produced bad
  metrics and you need to tell whether it's the load, the probe, or
  the target.
- Playwright docs: <https://playwright.dev/>
- OctoPerf Playwright VU docs: <https://doc.octoperf.com/design/virtual-users/playwright/>
