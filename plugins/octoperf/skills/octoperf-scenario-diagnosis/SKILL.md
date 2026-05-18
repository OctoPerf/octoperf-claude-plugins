---
name: octoperf-scenario-diagnosis
description: Use when an OctoPerf load-test scenario has completed (or is running) and the user wants to understand why it failed, underperformed, or behaved unexpectedly. Triggers on "the load test failed", "why are response times so high", "high error rate in the scenario", "diagnose this bench", "the run looks bad". Walks the LLM through reading global metrics, narrowing scope, comparing against validation, and surfacing the right next step (re-validate, tune scenario, fix infra). Requires the OctoPerf MCP server and a `benchResultId` to investigate.
---

# OctoPerf — Scenario / bench-result diagnosis

A scenario run produced metrics that look bad — high error rate, high
response times, low throughput, premature stop. This skill walks the
diagnosis: read metrics → narrow down → match the symptom to one of
four root-cause classes → surface the right fix.

## Inputs

You need a `benchResultId` from one of:

- A user-supplied id (often from a Slack / email link they paste).
- The return of `mcp__octoperf__run_scenario(scenarioId)`.
- `mcp__octoperf__list_bench_reports_by_project(projectId)` filtered on `benchResultIds` for the UI deep-link.

If `mcp__octoperf__get_bench_status(benchResultId)` shows progress < 1.0
the test is still running. Either wait and re-check, or surface what
*has* been measured so far with the caveat that it may change.

## Steps

### 0. Did the run even start?

Before reading metrics, confirm the run actually produced samples.
`run_scenario` can fail **before** any HTTP traffic is generated —
infrastructure error, no matching plan, deserialisation issue,
configuration rejected. A diagnosis built on metrics from a run that
never started will mislead the user.

```
mcp__octoperf__get_bench_result(benchResultId)
```

The exhaustive state machine is `CREATED → PENDING → SCALING →
PREPARING → INITIALIZING → (ERROR | RUNNING) → (FINISHED | ABORTED)`.
Any other label is a transport / UI artefact.

- `state = FINISHED` → proceed to step 1.
- `state = ABORTED` → either manual stop or stall-abort; jump to
  the [jmeter.log signature catalogue](#jmeterlog-signature-catalogue).
- `state = ERROR` → the run errored during provisioning or startup,
  no samples to read. Pull the orchestration logs:

  ```
  mcp__octoperf__list_bench_docker_logs(benchResultId)
  ```

  Common ERROR-state causes:

  - *No matching plan / capacity exhausted* → run pre-flight on the
    scenario to see why: `get_scenario_matching_plans(scenarioId)`
    (empty result) + `list_active_subscriptions()` (lists caps).
    The binding cap is usually `maxRealBrowserUsers=0` on basic
    plans rejecting a Playwright UserProfile, or
    `maxProfilesPerScenario` rejecting a multi-VU hybrid.
  - *Image pull / provider not available* → docker log surfaces it.
  - *Validation pre-flight failed* (some on-prem setups force a
    sanity check before run) → handle as a validation issue, hand
    off to `octoperf-validation-triage`.

### 1. Read global metrics first

```
mcp__octoperf__list_bench_reports_by_project(projectId)
# pick the report tied to your benchResultId, then
mcp__octoperf__get_bench_report(reportId)
# locate the SummaryReportItem in the returned items list, then
mcp__octoperf__get_report_summary_values(reportId, summaryItemId)
```

The default report's SummaryReportItem aggregates the test-wide values:
average response time, percentiles (p50/p90/p95/p99), hits per second,
total error rate, error count by type, total transactions, throughput.
**Don't dive into per-action data yet** — the global view tells you
which class of problem you're in.

**Trust caveat — load-generator overload.** Before reading any response
time, check whether the bench report has a `MonitoringAlarmsReportItem`
firing on the load generators (CPU / memory / load average). If it
fires, the response times in the report are **underestimated**: the
load generator itself was the bottleneck, and JMeter's internal timing
becomes unreliable. Surface this as a confidence caveat ("response
times are suspect — LG was overloaded") before drawing conclusions,
and suggest re-running on the cloud or on a larger LG.

**Trust caveat — cache hits skew the global numbers.** JMeter's
`CacheManager` is enabled by default. When a recorded VU hits the
same URL repeatedly (typical on a session that revisits pages), the
server returns HTTP 304 Not Modified and JMeter records the sample —
but the response time / throughput then reflect a *cache check*, not
real load on the SUT. If `get_report_pie_values` on the response-codes
widget shows more than ~40% 304s, flag it: the visible numbers are an
optimistic floor, the real SUT cost lives in the 200 samples. To
diagnose the SUT, filter to status=200 when drilling into per-action
metrics.

**Trust caveat — fail-fast peaks.** When the response-codes pie shows
a peak of errors **correlated with the hit-rate peak**, the server is
failing fast (errors return short, cheap responses). The apparent
throughput spike is illusory — read the error rate **before** the hit
rate when a chart shows a sudden bump.

**LG monitoring caveats.**

- Recommended ceiling per LG: ~1000 hits/sec on a 4-8 CPU LG.
  Persistent CPU alerts above that volume usually mean the test
  exceeds a single LG's headroom — add LGs, don't blame the SUT.
- High CPU **after G1 Old collections start** is heap pressure, not
  CPU starvation. Check `G1 Old / collectionCount` on the LG-JVMs
  widget before recommending more LGs.
- On cloud LGs, `%UsedMemory` alerts essentially never fire (OctoPerf
  pre-provisions). When they fire on an on-prem agent, **another
  process on the host** is the cause — the JVM alone won't trigger it.
- An empty `LoadGeneratorsChartReportItem` (hosts) on an on-prem run
  usually means **IP Spoofing is enabled** on that LG, which disables
  agent monitoring entirely — not "no data".

### 1b. Run the insights heuristics

OctoPerf ships a `InsightsReportItem` in the default report — call
`get_report_insights` and let the platform classify the run for you.
One call returns up to ~15 insights tagged by severity (`ERROR` /
`WARN` / `INFO` / `PASSED`) with the heuristic's numeric value. This
is the fastest path to a classification — often skips the manual
table lookup in step 2.

```
mcp__octoperf__get_report_insights(reportId, insightsItemId)
```

Mapping of common `InsightId`s to root-cause classes (look at the
ones tagged `ERROR` or `WARN` first):

| `InsightId`                       | Severity at fire | What it means                                                                                  | Where to look next                                                       |
|-----------------------------------|------------------|------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------|
| `RESPONSE_TIME_GLOBAL_AVG`        | INFO/WARN/ERROR  | RT drifts from the test's own average. Severity scales with deviation                          | `get_report_area_range_values` on the linked `inspect` widget            |
| `RESPONSE_TIME_STD_DEVIATION`     | INFO/WARN/ERROR  | Wide spread between p50 and p99 — user experience is inconsistent                              | `get_report_line_chart_values` on a PercentilesChartReportItem           |
| `STEP_BY_STEP_RESPONSE_TIME`      | INFO/WARN/ERROR  | One or two actions much slower than the rest — bottleneck on a specific endpoint              | `get_report_top_values` on Top Response Times                            |
| `CONNECT_TIME_VS_RESPONSE_TIME`   | INFO/WARN/ERROR  | Connect time is a large fraction of RT → TLS handshake / no keep-alive / connection pool small | Check HTTP server config: enable keep-alive, increase `resourcesPool`    |
| `LATENCY_VS_RESPONSE_TIME`        | INFO/WARN/ERROR  | Latency (server time to first byte) is the bulk of RT → server-side processing bottleneck     | Surface to user — check SUT thread pool, DB, GC                          |
| `HIT_RATE_INFLEXION_POINT`        | WARN/ERROR       | Hits/s plateaus before the VU count plateaus — SUT reached a soft cap mid-ramp                | Confirm with VU-vs-hits chart; the inflexion x-axis is the saturation point |
| `STEP_BY_STEP_ERRORS`             | INFO/WARN/ERROR  | Errors concentrated on a few actions                                                           | `get_report_top_values` on Top Error Percentages; then drill via `get_report_errors` |
| `PEAK_OF_ERRORS`                  | INFO/WARN/ERROR  | Errors spike at one moment (e.g. ramp inflexion)                                              | `get_report_line_chart_values` (USERLOAD + ERRORS_RATE) — what changed at that time? |
| `OVERALL_ERROR_4XX/5XX/NONE`      | INFO/WARN/ERROR  | Global error rate per family                                                                   | `get_report_pie_values` on the response-codes pie                        |
| `THROUGHPUT_IMAGE_NEW_FORMAT`     | INFO             | Old image formats (JPEG/GIF) dominate                                                          | Optimisation hint, not a perf bottleneck                                 |
| `THROUGHPUT_IMAGE_OPTIMIZE` / `THROUGHPUT_CSS` / `THROUGHPUT_JAVASCRIPT` | INFO | Bandwidth eaten by un-minified or un-compressed static assets                  | Optimisation hint                                                        |
| `THRESHOLD_ALARM`                 | (varies)         | A user-configured `ThresholdAlarmReportItem` fired (e.g. SLA breached)                        | The widget's metric tells you which monitor crossed which threshold      |

Each `Insight` carries a `more` widget (the visual context) and
sometimes an `inspect` widget (the drill-down comparison —
typically an `AreaRangeChartReportItem`). Use the matching
`get_report_*_values` on those to surface evidence for the user.
Note that the **same numeric value** that flagged the insight is
also the `rmse` field of the `AreaRangeChartReportItem` linked from
`inspect` — they're literally the same heuristic, exposed twice.

### 2. Classify the run

Match the metrics against one of these patterns:

| Pattern                                         | Likely class                | Where to look next                                                        |
|-------------------------------------------------|-----------------------------|---------------------------------------------------------------------------|
| High error rate (>5%), low load                 | **Functional regression**   | Validation skill — the VU itself is broken; don't analyze perf            |
| High error rate (>5%), high load                | **System under stress**     | Server-side capacity / config — surface to user, OctoPerf-side fix is rare|
| Low errors, p95 climbing with load              | **Bottleneck (application)**| Per-action / per-server metrics to identify the slow request              |
| Low errors, p95 flat-high, `CONNECT_TIME_VS_RESPONSE_TIME` fires | **Bottleneck (infra: TLS / keep-alive)** | Check the HTTP server config — enable keep-alive, increase resourcesPool. Symptom: connect time = 40%+ of RT |
| Low errors, p95 flat-high, `LATENCY_VS_RESPONSE_TIME` fires | **Bottleneck (server-side processing)** | SUT-side: thread pool, DB pool, GC. Surface to user — outside OctoPerf control |
| Hits/s plateaus before VU plateau (`HIT_RATE_INFLEXION_POINT` WARN+) | **Soft cap (knee point)** | The SUT saturates mid-ramp — capacity is below the configured load. Re-run at the inflexion x-axis VU count to confirm the knee |
| Low errors, flat p95, low throughput            | **Scenario misconfigured**  | User-load profile (`list_scenarios_by_project` → scenario detail in UI)   |
| Errors concentrated on one action               | **Specific endpoint broken**| Re-validate the VU; that action probably already fails functionally       |
| Errors only at start of run                     | **Warmup / cache cold**     | Surface to user; rerun with warmup or longer test                         |
| Errors only at end of run                       | **Resource exhaustion**     | Memory / connection / DB-pool — server side                               |
| Recurring per-minute error pulses               | **Synchronicity artefact** *(not a VU bug)* | Fixed think-times → bursts of same-second requests, or an infra cron (firewall / WAF heartbeat). Randomise thinktime to disambiguate before assuming auth/state issues |
| Test stopped early                              | **Killed or planned stop**  | `jmeter.log` signature distinguishes which — see [jmeter.log signature catalogue](#jmeterlog-signature-catalogue) |

**Smoke-vs-load heuristic.** If a low-VU smoke run (1 user, 10-20
iterations) exists for the same scenario / VU, compare the per-action
error rates:

- **Same error rate on smoke and load** → the failure is independent of
  concurrency. It's a **VU bug** (bad assertion, stale recording, wrong
  variable). Hand off to validation-triage.
- **Higher error rate under load (or new errors appearing)** → the
  application breaks *because* of concurrency. **Server-side issue** —
  check monitoring, surface to the user. Don't edit the VU.

If no smoke baseline exists, propose creating one via
`create_scenario_ramp_up` (users=1, rampUpSec=0) before running at full
scale — that's an order of magnitude cheaper than diagnosing failures
post-hoc.

### 3. Confirm class against the VU's validation

If you classified the run as functional regression or "specific
endpoint broken", **don't try to fix it from metrics**. Validation has
the HTTP-level detail you need.

```
mcp__octoperf__list_virtual_users(projectId)
# Identify the VU(s) used by the scenario from the scenario detail
mcp__octoperf__get_virtual_user_validation(projectId, virtualUserId)
```

If the latest validation is also failing, the VU is broken regardless
of load — hand off to the validation-triage skill. If validation is
clean, the breakage emerged under concurrent load (race condition,
test data exhausted, rate limit hit) — that's a finding to surface to
the user, not a VU-edit task.

### 4. Surface the verdict

End with a clear summary the user can act on:

- **Verdict:** one sentence — "the scenario failed because X".
- **Evidence:** 2-3 metric/HTTP snippets from `get_report_summary_values` / `get_report_*_values` that back the verdict.
- **Next step:** one of:
  - "Re-validate the VU with `validate_virtual_user` — validation is failing too."
  - "Tune the scenario (user load profile, ramp-up) — current settings under-/over-load the target."
  - "Check the target environment — errors are server-side, not VU-side."
  - "Re-run the scenario with longer/shorter duration / different profile — the current run was too short / unsuited to the symptom."

### 5. Open the report for the user

For anything beyond high-level diagnosis (per-percentile graphs,
per-monitor metrics, custom dashboards) the OctoPerf UI report is far
better than another tool call:

```
mcp__octoperf__list_bench_reports_by_project(projectId)
```

Filter the result on `benchResultIds` to find the reports tied to this
run and render their `url` as Markdown links so the user can open them.

## Hybrid scenarios — split per-VU

When a scenario has multiple UserProfiles (e.g. N×JMeter for load +
1×Playwright probe — see `octoperf-real-browser-probe`), the default
report's `StatisticTableReportItem` aggregates across all of them.
Use the tree variant to split by VU:

```
mcp__octoperf__get_report_tree_values(reportId, statisticTreeItemId)
```

Each `TreeEntry` carries a `virtualUserId` — group by it before
reading. Two important caveats when reading a hybrid run:

**1. Don't compare Playwright vs JMeter per-action timings naïvely.**
For the same target URL:
- JMeter measures *server-side HTTP response time* (TTFB + transmission).
- Playwright per-ACTION row measures the *Playwright command*
  (e.g. `page.goto` duration including JS exec + render, *or* just
  `page.click` time = the click + paint, the resulting navigation is
  separate). The numbers reflect different things.
- The browser also **caches** between actions within the same context
  — subsequent visits to the same URL look much faster than JMeter's
  cache-clearing iterations. This is real user behaviour, but it
  doesn't mean the SUT is fast.

The best apples-to-apples are the first homepage visit (Playwright's
`page.goto('/')` order 9 vs JMeter's `GET /` parent) — and even
then, the Playwright probe is 1 VU vs N JMeter VUs, so no contention.

**2. Playwright tree rows have multiple types — learn the
hierarchy.** The Playwright VU rows in the tree are labelled with a
`{label, type}` suffix on the actionId. Types you'll see:

| Type        | What it measures                                                                                |
|-------------|-------------------------------------------------------------------------------------------------|
| (bare id, no suffix) | Wall-clock per spec iteration (the source of truth for user-perceived journey duration)   |
| `GROUP` (label=`Actions`)   | Sum of all `ACTION` durations (overlaps with `Network` since Playwright is async)   |
| `GROUP` (label=`Network`)   | Cumulative time spent in HTTP requests per iteration                                |
| `HOOK` (label=`Before/After Hooks()`) | Playwright setup / teardown overhead                                      |
| `ACTION` (label=`page.X(...)`) | Individual Playwright command duration                                            |
| `EXPECT` (label=`expect.X(...)`) | Individual assertion duration                                                   |
| `NETWORK` (label=`<host>`)  | Aggregate of every individual HTTP request the browser made (hits=total requests)   |
| `NAVIGATION`              | Whole-page nav timing (DOM ready, load)                                               |

**Don't sum types** to compute total time — they overlap (Playwright
is async). The bare actionId row is the wall-clock. If a hybrid
scenario reports `Network` GROUP = 2.5s and `Actions` GROUP = 1.5s,
the real per-iteration time is *not* 4s — the bare row will show
~1.4s (the actual wall-clock).

**`Hits (CONTAINER)` vs `Hits` in TopReportItem.** When
`get_report_top_values` returns the container actionId at the top
(in our case `e3331762-...` for the JMeter VU's root container), the
value is the *whole iteration's elapsed time* — including thinktime
and all sub-actions. That's expected, not an anomaly. To find the
slowest *real* action, ignore the container and any `.resources`
aggregate.

## jmeter.log signature catalogue

**Log retention.** JMeter / Playwright log files are erased **7 days
after the run**, or as soon as the user leaves the design screen.
Zipped logs >200 MB are dropped entirely. Old runs may have no logs
to read — confirm freshness before promising a re-read.

When the test stopped earlier than its planned duration (or finished with the
"no more users running" pattern), pull `jmeter.log` via
`list_bench_result_files` + `read_bench_result_file_lines` and grep for one of
these signatures to tell *why* it ended:

| Log signature                                                                        | Meaning                                                              | What to tell the user                                                                |
|--------------------------------------------------------------------------------------|----------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| `Thread finished: <id>` repeated as threads drain before duration ends               | **End of iterations** — VUs hit their max-iterations policy          | Planned stop. Increase iterations or duration if more load was expected.             |
| `End of file:resources/<csv>.csv detected for CSV DataSet:<name> ... stopThread:true`| **CSV exhausted** — End-of-File policy = Stop VU                     | Planned stop. Either grow the CSV file or change EOF policy to Recycle / Continue.   |
| `Shutdown Test detected by thread: <id>`                                             | **On-Sample-Error policy fired** — at least one sample failed and the policy is Stop VU / Stop test | Policy-driven stop. Check the failing samples to decide whether to relax the policy. |
| `<user>@octoperf.com aborted the test` then `Test status changed: RUNNING => ABORTED`| **Manual abort** by a user in the UI                                  | Operator action. Nothing to fix on the VU side.                                       |
| `WARN Aborting stall test (expected end time:<ts> is past now)`                      | **Stall abort** — batch killed an unresponsive run 20 min past its planned end | The test stopped responding to shutdown signals (long loops / scripts / timeouts). Inspect per-action metrics for the culprit. |
| `o.a.j.JMeter: Command: Shutdown received from /127.0.0.1` (and nothing else)        | **Load generator killed** — container shutdown from outside JMeter   | Infra event (on-premise agent stopped, container removed, OOM). Re-run on the cloud or a healthier agent.  |
| `ERROR: java.lang.RuntimeException: Failed to perform cmdline operation: jmeter-plugins.org` | **Plugin download failure** (on-premise agent without internet)      | Provide the plugin JAR via the project files menu, or disable plugin download in the on-premise infra settings. |

Read the tail of `jmeter.log` first (the last 100 lines usually carry the
shutdown signature), then scan from the top if the cause isn't terminal.

**Per-sample network failure signatures** are a different beast — they show
up in the Errors report item / `get_report_errors` (response code = `-1
UNKNOWN`) rather than as test-stopping events. When the LLM sees these in
the `errorMessage` field of a `BenchError`, the cause is **not** in the VU
and is **not** under OctoPerf's control:

| Java exception in `errorMessage`                                            | Cause                                                       | What to tell the user                                                                                       |
|-----------------------------------------------------------------------------|-------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| `javax.net.ssl.SSLException: Connection reset`                              | TCP connection torn down mid-handshake                      | Target rejected the connection. Most often **DDoS protection / WAF** rate-limiting the LG IPs.              |
| `java.net.SocketTimeoutException: Read timed out`                           | Server accepted but didn't answer within timeout            | Target is overloaded or the action's response timeout is too tight. Check server monitoring first.          |
| `javax.net.ssl.SSLHandshakeException: Remote host terminated the handshake` | Server cut the TLS handshake                                | Almost always **WAF / DDoS protection**. Suggest dedicated IPs or whitelist the LG IP ranges.               |
| `org.apache.http.NoHttpResponseException: <host> failed to respond`         | TCP connection alive but server returned no HTTP response   | Server-side request handling failed silently — often a saturated thread pool.                               |
| `org.apache.http.conn.HttpHostConnectException: ... Connection timed out`   | LG couldn't open a TCP connection within timeout            | Network path issue (firewall, routing, target IP unreachable from the LG region).                            |
| `java.net.NoRouteToHostException: No route to host`                         | LG cannot reach the target at all (no route in routing tbl) | Target requires IP whitelisting — suggest the LG region's IP range or a dedicated IP.                       |

When two or more of these appear under load but **not** in a smoke run, the
diagnosis is almost always *server-side mitigation kicking in* (rate-limit,
WAF, DDoS-protection) rather than a real capacity issue.

## Anti-patterns

- **Don't re-run the scenario to "see if it's flaky"** unless the user explicitly asks. `run_scenario` is destructive (consumes credits) and rarely the right next move during diagnosis.
- **Don't drill into per-action metrics until the global view says you should.** Global metrics narrow the problem class in one call; per-action drilling without that context is expensive (tokens, tool calls).
- **Don't conclude from a half-finished test.** If `get_bench_status` < 1.0, label the read as preliminary and offer to come back.
- **Don't mix "the load test failed" with "the VU is broken".** The right tool is validation, not load. Surface the distinction to the user — they're paying for the credits either way.

## See also

- `octoperf-validation-triage` — when the VU itself is failing.
- `octoperf-auto-correlation` — when failures are session/auth-state related.
- OctoPerf bench reports docs: <https://doc.octoperf.com/analysis/bench-reports/>
