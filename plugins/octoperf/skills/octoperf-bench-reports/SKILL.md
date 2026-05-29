---
name: octoperf-bench-reports
description: Use when reading or interpreting an OctoPerf bench report — picking the right `get_report_*_values` tool for a given widget, understanding the difference between flat and trend reports, decoding semantic gotchas (Hits vs Hits CONTAINER, 304 cache hits skewing throughput, Playwright per-step row types, etc.). Triggers on "what's the right tool for this widget", "explain this metric", "how do I read this trend report", "what does parallelRunsSupported mean", "why is the Network row 24ms while page.goto is 364ms", "DELTA computeType". Complements `octoperf-scenario-diagnosis` — that skill walks the diagnosis workflow, this one is the widget-by-widget reading guide. Requires the OctoPerf MCP server.
---

# OctoPerf — Reading bench reports

A `BenchReport` is a polymorphic document. Its `items` array carries
20+ widget types (charts, tables, top-N, insights, …), each backed by
its own `get_report_*_values` tool. This skill maps every widget you
can encounter to the right tool, calls out the **semantic gotchas**
that have repeatedly tripped LLMs, and explains the trend-report
architecture.

## The BenchReport shape — one quick anchor

```
BenchReport {
  id, projectId, name, benchResultIds,         // — the runs the report aggregates
  configs:  [ApdexReportConfig | TrendReportConfig | ...],  // global settings
  items:    [polymorphic BenchReportItem...]   // — what's visible on the page
}
```

- A **regular report** has 1 entry in `benchResultIds` (the run it
  was generated for) and items that pull values from that run.
- A **trend report** has 1 entry too (the *reference* anchor) and a
  `TrendReportConfig` in `configs` whose `selectors` are
  re-evaluated **dynamically at read time** to pull in other matching
  runs. See [Trend reports](#trend-reports) below.

To read any widget, always start with:

```
mcp__octoperf__get_bench_report(reportId)
```

then dispatch on each `items[i]["@type"]` per the table below.

## Widget → tool mapping

For every widget type that's reachable from MCP:

| `@type`                              | Tool                                                  | Returns                                                                    |
|--------------------------------------|-------------------------------------------------------|----------------------------------------------------------------------------|
| `SummaryReportItem`                  | `get_report_summary_values`                           | `List<Double>` aligned with `item.metrics[i].id`                           |
| `BarChartReportItem`                 | `get_report_summary_values` *(same shape as Summary)* | `List<Double>` aligned with `item.metrics[i].id`                           |
| `StatisticTableReportItem`           | `get_report_table_values`                             | `List<TableEntry>` (`actionId` → `values`)                                 |
| `StatisticTreeReportItem`            | `get_report_tree_values`                              | `List<TreeEntry>` (`virtualUserId` + `actionId` → `values`) — per-VU split |
| `TopReportItem`                      | `get_report_top_values`                               | `TopResult` (top-N actionIds + per-action curve)                           |
| `PieChartReportItem`                 | `get_report_pie_values`                               | `List<Map<String, Long>>` (one map per benchResult, label → count)         |
| `LineChartReportItem`                | `get_report_line_chart_values`                        | `List<List<GraphPoint>>` (one series per metric, `(x=epoch-ms, y)`)        |
| `PercentilesChartReportItem`         | `get_report_line_chart_values`                        | Same shape — percentile curve                                              |
| `StackedChartReportItem`             | `get_report_stacked_chart_values`                     | `List<MapGraphPoint>` (`x` + per-series map)                               |
| `AreaRangeChartReportItem`           | `get_report_area_range_values`                        | `AreaRangeResult` (`curve` vs `reference`, `rmse`)                         |
| `InsightsReportItem`                 | `get_report_insights`                                 | `Set<Insight>` (severity + value + drill-in widget)                        |
| `ErrorsReportItem`                   | `get_report_errors`                                   | `List<BenchError>` (per-sample failures)                                   |
| `ThresholdAlarmReportItem`           | `get_report_threshold_alarms`                         | `List<ThresholdAlarm>` (per-breach)                                        |
| `TextualMonitorReportItem`           | `get_report_textual_monitors`                         | `List<TextualCounterValue>` (string-valued monitor samples)                |
| `LoadGeneratorsChartReportItem`      | `list_bench_load_generators`                          | `List<BenchLoadGenerator>` — chart is derived from this                    |
| `LoadGeneratorsTreeReportItem`       | `list_bench_load_generators`                          | Same source as the chart — tree is just a different rendering              |
| `TextReportItem`                     | *(no tool — descriptive markdown)*                    | n/a — `item.description` carries the markdown                              |
| `SynopsisReportItem`                 | *(no tool — scenario metadata)*                       | n/a — render the synopsis section in the UI for the user                   |
| `TrendConfigReportItem`              | *(no tool — read `configs`)*                          | n/a — the selectors live in the report's `TrendReportConfig`               |
| `MonitorsTableReportItem`            | **❌ no MCP tool**                                     | UI only — list of monitor connections with threshold-alarm counts          |

Two follow-up tools to keep in mind:

- After `get_report_errors`, drill into a specific failed sample with `fetch_bench_error_http(benchResultId, actionId, timestamp)` — returns the full request + response of that one breach.
- For non-text bench-result artefacts (Playwright `trace.zip`, screenshots, HAR), `download_bench_result_file(benchResultId, filename)` returns a presigned GET URL (single-use, ~5 min) — fetch the bytes directly with your code interpreter. `read_bench_result_file_lines` only handles text.

## Semantic gotchas

A field-collected list of values that *look* like one thing but mean
another. Each cost an LLM debug cycle in the past — surface them to
the user when reading the data:

### `Hits` vs `Hits (CONTAINER)`

- `Hits` (and its rates `Hits/s`, `Hits successful total`) count **HTTP samplers only**.
- `Hits (CONTAINER)` counts **everything else** — containers, logic actions (Loop / If / While), JMeter plugins, the VU root container.

When `get_report_top_values` returns a top-by-avg-RT where the highest
row is the **VU's root container** (no parent in the action tree),
the value is the *whole iteration's wall-clock* — including thinktime.
Ignore the container row when looking for slow *real* actions.

### Metric subtypes — not every sub-count is on every widget

For the full per-widget allow-list, see the
[hit-metrics availability table](https://doc.octoperf.com/analysis/edit-bench-report/performance-metrics/#hit-metrics-availability)
in the public doc. The recurring picks that trip up an LLM:

- `Hits` (`Total` / `Total Successful` / `Rate` / `% Successful`) and `Errors` (`Total` / `Rate` / `% Error`) are accepted on Line, Summary, Table/Tree, Bar, Area. `Top` excludes `Rate` for both; `Percentiles` accepts only `Total` + `Rate` for Hits and only `Total` + `Rate` for Errors (no `% Error`).
- `Errors % Error` is on a **0..100 scale**, so Insight thresholds expressed as integers in 0..100 compare to it directly.
- `Median` (`RESPONSE_TIME_MEDIAN`) is on **Summary / Table / Tree / Bar** only — not on Line, Top, Percentiles or Area.
- The discrete percentiles `RESPONSE_TIME_PERCENTILE_80 / 90 / 95 / 99` live on **Line / Summary / Top / Table / Tree / Bar / Area**. The `PercentilesChartReportItem` widget plots a *continuous percentile curve* from a base metric (Response Time, Latency, …) and does **not** accept these discrete percentile sub-counts as metrics — picking one for a Percentiles widget is a mismatch.
- `Apdex` is defined on `Response Time / Connect Time / Latency` only, on Line, Summary, Table, Tree, Bar, Area — never on Top or Percentiles. It requires `satisfying` + `tolerating` thresholds, falling back to the global `ApdexReportConfig` on the report when unset on the metric.
- `Network Time = Response Time − Latency` — pre-computed server-side; the value is real even if no `Latency` curve appears in the report. No `StdDev` or `Apdex` variant exists.
- `Received Data` only supports `Total` and `Rate`; `Sent Data` adds `Average / Min / Max / StdDev / Total / Rate`. Asking for `Received Data Average` returns nothing.
- `UserLoad` is a **monitor sample** (not a hit metric). It shows up as the load-curve overlay on Line / Bar / Area charts but isn't selectable through the same picker as hit metrics.
- `HTTP methods` / `HTTP response codes` / `Media types count` / `Media types throughput` only appear on `PieChartReportItem` and `StackedChartReportItem` — they're not in the hit-metrics availability table.

### Cache hits (304s) skew global numbers

JMeter's `CacheManager` is on by default. On any VU that revisits
the same URL within a session, the server returns HTTP **304 Not
Modified** and JMeter records the sample. The response time +
throughput then reflect a *cache check*, not real load on the SUT.

→ If `get_report_pie_values` on the response-codes pie shows more
than ~40% 304s, **flag it** when summarising: the visible numbers
are an optimistic floor; the real SUT cost lives in the 200 samples.

### Playwright per-step row types (Statistic*Tree*)

The same VU can emit many row types in `get_report_tree_values`,
keyed by `actionId` with a JSON-encoded suffix:

| `type` (in the label suffix) | What it measures                                           |
|------------------------------|------------------------------------------------------------|
| (bare actionId, no suffix)   | Wall-clock per spec iteration — source of truth for UX     |
| `GROUP` (`label="Actions"`)  | Sum of all `ACTION` durations                              |
| `GROUP` (`label="Network"`)  | Cumulative time in HTTP requests per iteration             |
| `HOOK` (`Before/After Hooks`)| Playwright setup / teardown                                |
| `ACTION` (`page.X(...)`)     | Single Playwright command duration                         |
| `EXPECT` (`expect.X(...)`)   | Single assertion duration                                  |
| `NETWORK` (`<host>`)         | Aggregate of every HTTP request the browser made           |
| `NAVIGATION`                 | DOM ready / load timing                                    |

**Cardinal rule: don't sum types — they overlap because Playwright is
async.** If `Network` GROUP says 2.5 s and `Actions` GROUP says 1.5 s,
the per-iteration wall-clock is **not** 4 s. Read the bare actionId
row for the true wall-clock.

### Widget-specific quirks

- `StackedChartReportItem` accepts **exactly one** metric — the toggle is `mode: ABSOLUTE | PERCENT`. Multi-metric stacked configs are not representable.
- `AreaRangeChartReportItem`: `referenceType = HISTOGRAM` (time-varying reference) or `SUMMARY` (constant average); `rangeType = RAW` (both metrics share a unit) or `PERCENTAGE` (mixed-unit comparison). Wrong combinations return a meaningless curve, not an error.
- `InsightsReportItem` emits a "not enough data" notice when the run has **<50 VUs or <20 minutes** — insights on shorter/smaller runs can be ignored or hedged.
- Insight thresholds are **percentages in 0..100** (not 0..1) — they govern the severity bucket (Passed / Info / Warn / Error). The same heuristic value can map to a different severity depending on the per-config thresholds.
- `StatisticTableReportItem` / `StatisticTreeReportItem`: if the source VU has `downloadResources=true`, every HTTP request action produces **two rows** — the request itself plus a `.resources` row aggregating all embedded assets. The `.resources` row's hit count = total embedded sub-requests, not iterations.
- `ErrorsReportItem` (`get_report_errors`): on SaaS the result is capped at **2 rows per `(loadGenerator, request, responseCode)` triple** — counts are exact but the returned sample list is a quota-limited subset (on-prem can override).
- `ErrorsReportItem` covers 3 trigger types: 4XX/5XX, engine-level Java exception (response code `-1`, header `HTTP/1.1 -1 - UNKNOWN`), and failed `ResponseAssertion`. Only the `assertions[]` field on a `BenchError` distinguishes assertion failures from non-2XX.
- `TextualMonitorReportItem`: filter is **monitor-connection-only** (no metric / location filters). An empty result means the connection emitted no string-valued counters this run.
- `MonitorsTableReportItem` excludes load generators (LGs live in `LoadGeneratorsTreeReportItem`). It's also UI-only — no MCP read tool.
- `LoadGeneratorsChartReportItem` (hosts: `monitorType=HOST`; JVMs: `monitorType=JVM`) plots a **fixed metric set as max across all LGs**, not per-LG. Hosts: `%CPU`, `%Mem`, `%SegRetrans`, `Received MB/s`. JVMs: heap %, G1 young/old count + time.

### `parallelRunsSupported` in `ScenarioMatchingPlan`

When `get_scenario_matching_plans` returns plans with
`parallelRunsSupported`, that integer is the number of **simultaneous
instances of the scenario** the plan can host (typically 1 — only
matters with `maxTestsPerRun > 1`). It is **not** "max users the plan
will allocate". Any non-empty result means the run is launchable as
configured.

### KO matrix can be overridden by a ResponseAssertion

`get_report_errors` will return KO samples that look like 200/200 in
the recorded matrix — that's a `ResponseAssertion` firing on the
body. Check `assertions` on the `BenchError` before assuming HTTP
mismatch. Useful pointer:
[octoperf-validation-triage](octoperf-validation-triage) for the
full KO/OK matrix + assertion override.

### Insight `value` ≡ AreaRange `rmse`

When an `InsightsReportItem` fires with severity ERROR/WARN, its
`value` is **the same number** as the `rmse` of the
`AreaRangeChartReportItem` widget linked from its `inspect`. They're
the same heuristic exposed twice. Don't fetch both unless you want
to render the curve+reference visually.

## Trend reports

A trend report compares the **anchor** benchResult (the one in
`benchResultIds[0]`) against a **dynamically-resolved** list of other
benchResults from the project. The matching is defined by the
`TrendReportConfig` inside the report's `configs`:

```
{
  "@type": "TrendReportConfig",
  "selectors": [{
    "@type": "TrendReportNameSelector" | "TrendReportTagsSelector" | "TrendReportCreationDateSelector",
    ...
  }],
  "shownResults": 20
}
```

Three selector types correspond to the three creation tools:

- `create_trend_report_by_name` — `TrendReportNameSelector` matching the scenario name (EQUALS / CONTAINS / STARTS_WITH / ENDS_WITH, with `_IGNORECASE` variants).
- `create_trend_report_by_tags` — `TrendReportTagsSelector` matching the tag intersection on bench results.
- `create_trend_report_by_creation_date` — `TrendReportCreationDateSelector` matching a `[fromMs, toMs]` window.

**The list is recomputed on every report read.** A run created after
the trend's `created` timestamp **will** appear on the next read if
it matches the selector — you don't need to recreate the report.

**Caps and the Reference Test.** A trend report holds at most **25
matched results** plus one **Reference Test** that cannot be
unselected. The Reference Test is preserved past the project's
default 100-result retention cap, and deleting it is blocked while a
trend report still uses it. Manual labels (Trend Manual Selection)
override the auto-generated bench names and live on the trend config,
not on the bench results themselves.

### What changes in a trend report's items

Three widget types behave differently in a trend context:

- `TrendConfigReportItem` — read-only display of the selectors. Use the report's `configs` directly.
- `StatisticTableReportItem` with **`computeType: "DELTA"`** — the table shows the delta of each metric between the anchor and each matched run. Negative = improvement, positive = regression. Use `get_report_table_values` as usual; the diff math happens server-side.
- `BarChartReportItem` titled *"Latest vs Reference Summary"* — one bar per matched run for each metric. Same `get_report_summary_values` call, just more values returned.

The other widgets (LineChart, AreaRange, Pie, …) work the same; they
just plot the anchor by default.

## Comparison reports

A comparison report holds **2 to 4 bench results** labelled `A` / `B`
/ `C` / `D` by default in widget legends. Labels are renamed in the
report's `configuration`, not on the bench results. Unlike trend
reports, the result list is static — it's the snapshot in
`benchResultIds`, not a selector re-evaluated on read.

## Report configuration caveats

- `Time range` filtering applies to **simple reports only** (not
  comparison reports), and only after the run is `FINISHED`. Applying
  a time range to a running test is a no-op.
- The per-report caps `maxPercentiles` / `maxColumns` / `maxPies` /
  `maxStatistics` / `maxLines` are enforced when **adding** metrics;
  widgets that pre-date the cap keep their over-the-limit metric lists.

## Pitfalls

- **Don't call a `get_report_*_values` tool with the wrong item type.** Each tool checks the `@type` and rejects unrelated widgets with `IllegalArgumentException`. The error message points at the right tool.
- **Don't read a half-finished bench's report.** If `get_bench_status(benchResultId) < 1.0`, the values are partial. Label the read as preliminary and offer to come back when state is FINISHED.
- **Don't ignore `Trust caveats` from `octoperf-scenario-diagnosis`.** Load-generator overload underestimates response times; cache hits skew global numbers. Both surface in the bench report but the numbers don't carry the warning themselves.
- **MonitorsTableReportItem has no MCP read tool.** If a report has one and the user asks to see the data, point them to the OctoPerf UI report URL. Don't fabricate values.
- **A `ThresholdAlarm` with 0-duration is an instantaneous breach.** Non-zero duration means a sustained one. Treat isolated 0-duration alarms as noise; clusters (multiple within seconds) are signal.
- **Playwright `Results Tree` rows can show negative response times.** `page.waitForTimeout` and actions inside nested `for` / `if` blocks can produce them in corner cases — don't propagate a negative value as an anomaly upstream.

## See also

- `octoperf-scenario-diagnosis` — workflow for diagnosing a poor run (this skill is the reading guide; scenario-diagnosis is the action plan).
- `octoperf-validation-triage` — when the report shows the VU itself is failing.
- OctoPerf bench reports docs: <https://doc.octoperf.com/analysis/bench-reports/>
