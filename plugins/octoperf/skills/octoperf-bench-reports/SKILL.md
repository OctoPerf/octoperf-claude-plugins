---
name: octoperf-bench-reports
description: Use when reading or interpreting an OctoPerf bench report — picking the right `get_report_*_values` tool for a given report item, understanding the difference between flat, trend and comparison reports, reusing a report's shape through a report template, decoding semantic gotchas (Hits vs Hits CONTAINER, 304 cache hits skewing throughput, Playwright per-step row types, etc.). Triggers on "what's the right tool for this report item", "explain this metric", "how do I read this trend report", "compare these two runs", "save this report as a template", "apply my template to this report", "what does parallelRunsSupported mean", "why is the Network row 24ms while page.goto is 364ms", "DELTA computeType". Complements `octoperf-scenario-diagnosis` — that skill walks the diagnosis workflow, this one is the report item-by-report item reading guide. Requires the OctoPerf MCP server.
---

# OctoPerf — Reading bench reports

A `BenchReport` is a polymorphic document. Its `items` array carries
20+ report item types (charts, tables, top-N, insights, …), each backed by
its own `get_report_*_values` tool. This skill maps every report item you
can encounter to the right tool, calls out the **semantic gotchas**
that have repeatedly tripped LLMs, and explains the trend-report
architecture.

> **Editing** a report item's metrics/filters (not just reading them)? See
> `octoperf://skills/report-item-editing` — it covers the allow-select
> workflow (`get_report_item_allow_select` → `get_report_tag_keys` /
> `get_report_tag_values` → `validate_report_item` → `patch_bench_report`).

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

To read any report item, always start with:

```
mcp__octoperf__get_bench_report(reportId)
```

then dispatch on each `items[i]["@type"]` per the table below.

## Before reading: the run's data must be up to date

A bench result holds its samples in the shape the current analysis
queries expect only once its data has been updated. A run recorded
before the last data model change still holds the previous shape, and
the metric queries never look there: every report item answers **empty** —
not "old numbers", not an error. Zeros read as a finding, which is how
an outdated run becomes a fabricated diagnosis.

So the report tools refuse rather than answer empty:

| tool | on data that is not up to date |
|---|---|
| the 12 `get_report_*_values`, plus `get_report_insights` / `_errors` / `_threshold_alarms` / `_textual_monitors` | **refuse**, naming the runs concerned — per report item, so only the runs that report item actually reads |
| `get_report_tag_keys` / `get_report_tag_values` | **refuse** — an empty tag set reads as "this run has no action, no region" |
| `export_bench_report_pdf` | **refuse** — a PDF of empty report items is a document the user forwards |
| `list_bench_reports_by_project` | answers, carrying `reportIdsNeedingDataUpdate` / `reportIdsBeingUpdated` |
| `create_trend_report_by_name` / `_by_tags` / `_by_creation_date` | answer, carrying the project's runs the trend will not be able to plot |
| `create_comparison_report` | answers, carrying the compared runs whose column would read empty |
| `get_report_item_allow_select`, `validate_report_item`, `patch_bench_report`, `get_bench_report` | answer as usual — editing a report item reads its schema, not samples |
| the seven report-template tools | answer as usual — a template is a layout, it holds no samples |

When a read refuses:

1. `get_report_data_status(reportId)` — sorts the report's runs into
   `upToDate` / `needsUpdate` / `updating` / `unknown` (`unknown` = the
   run could not be read at all, usually deleted).
2. **Ask the user first.** Updating rewrites the run's stored data,
   takes minutes on a large run, and is not reversible.
3. `update_report_data(reportId)` — launches one update per run that
   needs it and returns a `taskId` for each.
4. Poll each task with `get_task_result` (see
   `octoperf-async-polling`), then re-read the report item.

A task that ends FAILED on a `N problem(s) met while migrating…`
message updated what it could and left the rest behind: the run then
counts as up to date while the report items that depended on the missing
part still read empty. Pass that message to the user verbatim — it
names what was left out — and do not re-run the update expecting a
different outcome.

Two rules when talking to the user:

- A refusal is not "the test recorded nothing" — say the report's data
  needs updating first, and name the runs.
- Never call it a *migration*. The report's data is being **updated**.

## Report item → tool mapping

For every report item type that's reachable from MCP:

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
| `InsightsReportItem`                 | `get_report_insights`                                 | `Set<Insight>` (severity + value + drill-in report item)                        |
| `ErrorsReportItem`                   | `get_report_errors`                                   | `List<BenchError>` (per-sample failures)                                   |
| `ThresholdAlarmReportItem`           | `get_report_threshold_alarms`                         | `List<ThresholdAlarm>` (per-breach)                                        |
| `TextualMonitorReportItem`           | `get_report_textual_monitors`                         | `List<TextualCounterValue>` (string-valued monitor samples)                |
| `LoadGeneratorsChartReportItem`      | `list_bench_load_generators`                          | `List<BenchLoadGenerator>` — chart is derived from this                    |
| `LoadGeneratorsTreeReportItem`       | `list_bench_load_generators`                          | Same source as the chart — tree is just a different rendering              |
| `TextReportItem`                     | *(no tool — descriptive markdown)*                    | n/a — `item.description` carries the markdown                              |
| `SynopsisReportItem`                 | *(no tool — scenario metadata)*                       | n/a — render the synopsis section in the UI for the user                   |
| `TrendConfigReportItem`              | *(no tool — read `configs`)*                          | n/a — the selectors live in the report's `TrendReportConfig`               |
| `MonitorsTableReportItem`            | `get_report_monitors_table_values`                    | One row per monitor connection of the run: `monitorType`, `counterPaths`, `alarmCount`, worst `severity` |

Two follow-up tools to keep in mind:

- After `get_report_errors`, drill into a specific failed sample with `fetch_bench_error_http(benchResultId, actionId, timestamp)` — returns the full request + response of that one breach. Both tools hand back text: a Playwright failure is stored with the colors its runner produced — in `errorMessage` and, since the body *is* the error there, in the response body too — and the two tools drop them. The web UI renders them instead, so a screenshot shows colors the tool never returns.
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

### Metric subtypes — not every sub-count is on every report item

For the full per-report item allow-list, see the
[hit-metrics availability table](https://doc.octoperf.com/analysis/edit-bench-report/performance-metrics/#hit-metrics-availability)
in the public doc. The recurring picks that trip up an LLM:

- `Hits` (`Total` / `Total Successful` / `Rate` / `% Successful`) and `Errors` (`Total` / `Rate` / `% Error`) are accepted on Line, Summary, Table/Tree, Bar, Area. `Top` excludes `Rate` for both; `Percentiles` accepts only `Total` + `Rate` for Hits and only `Total` + `Rate` for Errors (no `% Error`).
- `Errors % Error` is on a **0..100 scale**, so Insight thresholds expressed as integers in 0..100 compare to it directly.
- `Median` (`RESPONSE_TIME_MEDIAN`) is on **Summary / Table / Tree / Bar** only — not on Line, Top, Percentiles or Area.
- The discrete percentiles `RESPONSE_TIME_PERCENTILE_80 / 90 / 95 / 99` live on **Line / Summary / Top / Table / Tree / Bar / Area**. The `PercentilesChartReportItem` report item plots a *continuous percentile curve* from a base metric (Response Time, Latency, …) and does **not** accept these discrete percentile sub-counts as metrics — picking one for a Percentiles report item is a mismatch.
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

### Quirks by report item type

- `StackedChartReportItem` accepts **exactly one** metric — the toggle is `mode: ABSOLUTE | PERCENT`. Multi-metric stacked configs are not representable.
- `AreaRangeChartReportItem`: `referenceType = HISTOGRAM` (time-varying reference) or `SUMMARY` (constant average); `rangeType = RAW` (both metrics share a unit) or `PERCENTAGE` (mixed-unit comparison). Wrong combinations return a meaningless curve, not an error.
- `InsightsReportItem` emits a "not enough data" notice when the run has **<50 VUs or <20 minutes** — insights on shorter/smaller runs can be ignored or hedged.
- Insight thresholds are **percentages in 0..100** (not 0..1) — they govern the severity bucket (Passed / Info / Warn / Error). The same heuristic value can map to a different severity depending on the per-config thresholds.
- `StatisticTableReportItem` / `StatisticTreeReportItem`: if the source VU has `downloadResources=true`, every HTTP request action produces **two rows** — the request itself plus a `.resources` row aggregating all embedded assets. The `.resources` row's hit count = total embedded sub-requests, not iterations.
- `ErrorsReportItem` (`get_report_errors`): on SaaS the result is capped at **2 rows per `(loadGenerator, request, responseCode)` triple** — counts are exact but the returned sample list is a quota-limited subset (on-prem can override).
- `ErrorsReportItem` covers 3 trigger types: 4XX/5XX, engine-level Java exception (response code `-1`, header `HTTP/1.1 -1 - UNKNOWN`), and failed `ResponseAssertion`. Only the `assertions[]` field on a `BenchError` distinguishes assertion failures from non-2XX.
- `TextualMonitorReportItem`: filter is **monitor-connection-only** (no metric / location filters). An empty result means the connection emitted no string-valued counters this run.
- `MonitorsTableReportItem` lists the run's own copy of the monitor connections — SLA profiles included, load generator hosts and JVMs excluded since the report charts those in `LoadGeneratorsChartReportItem` items. Its `connectionName` / `counterPaths` are tag values, ready to use as `SingleTermFilter` terms on a `MONITORING` metric. An empty `counterPaths` means the connection was configured but sampled nothing.
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
`AreaRangeChartReportItem` report item linked from its `inspect`. They're
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

Three report item types behave differently in a trend context:

- `TrendConfigReportItem` — read-only display of the selectors. Use the report's `configs` directly.
- `StatisticTableReportItem` with **`computeType: "DELTA"`** — the table shows the delta of each metric between the anchor and each matched run. Negative = improvement, positive = regression. Use `get_report_table_values` as usual; the diff math happens server-side.
- `BarChartReportItem` titled *"Latest vs Reference Summary"* — one bar per matched run for each metric. Same `get_report_summary_values` call, just more values returned.

The other report items (LineChart, AreaRange, Pie, …) work the same; they
just plot the anchor by default.

## Comparison reports

A comparison report holds **2 to 4 bench results** labelled `Run A` /
`B` / `C` / `D` by default in report item legends. Unlike trend
reports, the result list is static — it's the snapshot in
`benchResultIds`, not a selector re-evaluated on read. To follow one
scenario over time, that's a trend report; to put a handful of named
runs side by side, it's this.

**Creating one:** `create_comparison_report(benchResultIds, names?)`.
The backend lays out the whole report from its own template — the
shared report items (user loads, statistics summary, comparison
table, response-code donut, response-time chart) each carry **one
metric per run**, followed by a detail section per run. Nothing has
to be duplicated by hand afterwards.

- **`benchResultIds` is the column order.** Index 0 is the first
  column. The Compare page sorts oldest-first before creating, which
  is the convention to follow unless the user says otherwise.
- **`names` labels the columns positionally** — one per run, or leave
  it out for the automatic `Run A` … `Run D`. The labels live in the
  report's `ComparisonReportConfig`, never on the bench results
  themselves, so renaming a column changes this report only.
- Every run must be **over** (FINISHED or ABORTED; a run in ERROR
  produced nothing) and **in the same project** — the report is filed
  under the project of the first run.
- Adding a fifth run later is not a thing: create another report.

**Reading one:** the `TextReportItem` descriptions of a comparison
report carry `$RESULTS$` and `$RESULT(0)$` markers. They are
placeholders the **web UI** substitutes at render time — through MCP
they come back raw. Resolve them yourself from `benchResultIds` (or
just describe the section) rather than showing a user a `$RESULT(0)$`.
The same goes for `$COMPANY_NAME$`, `$REPORT_NAME$`, `$REPORT_DATE$`,
`$PROJECT_NAME$` and `$SLA_PROFILES(resultId)$`. A PDF export renders
through the UI, so it shows the resolved text.

## Report templates

A **report template** is a saved report layout — the report items and
settings of a report, with the run they were read from taken out — so
that the next run's report can be given the same shape in one call. It
is a **workspace** entity: every project of the workspace sees the same
list. In the UI: **Analysis → Report Templates**.

| Tool | What it does |
|---|---|
| `list_report_templates(projectId)` | the templates of that project's workspace — id, name, description, `itemCount`, url |
| `get_report_template(projectId, templateId)` | the template in full: its `items` and `configs`, the "Report Contents" of the UI |
| `create_report_template_from_report(reportId, name?, description?)` | saves an existing report as a template; the report is untouched |
| `patch_report_template(projectId, templateId, patch)` | RFC 6902 JSON Patch over the template — its report items, its per-metric colours and Apdex, its settings |
| `apply_report_template(reportId, templateId)` | replaces a report's items with the template's, repointed at that report's run |
| `update_report_template(projectId, templateId, name?, description?)` | renames it |
| `delete_report_template(templateId)` | drops the layout; the reports built from it keep their items |

A template is also what a run's report can be built from in the first
place: `run_scenario(scenarioId, templateId)` and
`schedule_scenario_once` / `_cron` with a `templateId` give the new run
the template's shape instead of the default layout — the Template
picker of the launch page and of the scheduler. That is one call
instead of run-then-apply, and it is the only way for a scheduled job,
which fires with nobody there to apply anything.

**A template is not a copy of the report.** On the way in, filters that
only mean something inside the run the report was built on are
stripped: the load generator that served a request (`injectorId`), the
region it ran from (`region`), and the monitoring connections the run
created for itself (the load generators' own host and JVM
connections). A metric whose filters all disappear is dropped, and a
report item left without a single metric is dropped with it. That is
why `create_report_template_from_report` answers with `sourceItemCount`
and `runSpecificFilterFields` next to the template's own `itemCount` —
say what did not make it in rather than presenting the template as
faithful.

**Applying one replaces the report's items**, and the previous layout
is not recoverable. Only the report's own time range survives; the
template's `${PROJECT_NAME}`, `${REPORT_NAME}`, `${PROJECT_DESCRIPTION}`
and `${REPORT_DESCRIPTION}` placeholders are filled in on the way.
`apply_report_template` refuses on a report that compares several runs:
the mechanism repoints every report item at `benchResultIds[0]`, so a
comparison report would come back with a single column.

**Editing a template.** `patch_report_template` writes the same fields
the Report Templates page edits. Where presentation lives:

- **a metric's colour** — a `SingleColorReportConfig` in *that metric's*
  own `configs`, e.g. `/items/2/metrics/1/configs/-` with
  `{"@type":"SingleColorReportConfig","color":"#CC5500"}`;
- **a metric's Apdex** — an `ApdexReportConfig` in the same place;
- **the template-wide settings** — the root `/configs`.

Two things it will not do. It refuses a patch that changes `id` or
`workspaceId`. And, unlike `patch_bench_report`, it cannot check a
metric against a run's allow-select descriptor — a template has no run
to resolve one from, so an impossible metric surfaces only once the
template is applied. When the change is about *which* metric rather
than how it looks, shape a real report first and re-extract.

**Where templates fit the workflow.** The natural loop is: shape one
report the way the user wants it → save it as a template → give it to
the next runs, either up front (`run_scenario(templateId)`,
`schedule_scenario_*(templateId)`) or after the fact
(`apply_report_template`). `create_report_template_from_report` is the
only way to create one — there is no writing a template from scratch.

## Report configuration caveats

- `Time range` filtering applies to **simple reports only** (not
  comparison reports), and only after the run is `FINISHED`. Applying
  a time range to a running test is a no-op.
- The per-report caps `maxPercentiles` / `maxColumns` / `maxPies` /
  `maxStatistics` / `maxLines` are enforced when **adding** metrics;
  report items that pre-date the cap keep their over-the-limit metric lists.

## Pitfalls

- **Don't read zeros as a result.** A report item that refuses because its run's data is not up to date is not a run without traffic. See [Before reading](#before-reading-the-runs-data-must-be-up-to-date) — and never present the two as the same thing.
- **Don't call a `get_report_*_values` tool with the wrong item type.** Each tool checks the `@type` and rejects unrelated report items with `IllegalArgumentException`. The error message points at the right tool.
- **Don't read a half-finished bench's report.** If `get_bench_result(benchResultId).state` is not yet `FINISHED`, the values are partial. Label the read as preliminary and offer to come back once it is. `get_bench_status` returns a 0-100 progress percentage, so it can tell you *how far* the run has got but never *that it is done*.
- **Don't ignore `Trust caveats` from `octoperf-scenario-diagnosis`.** Load-generator overload underestimates response times; cache hits skew global numbers. Both surface in the bench report but the numbers don't carry the warning themselves.
- **A monitors table row counts alarms, it does not describe them.** `get_report_monitors_table_values` says how many alarms a connection raised and the worst severity among them; which threshold fired, when and for how long comes from `get_report_threshold_alarms` on the report's `ThresholdAlarmReportItem`.
- **Don't hand a user a `$RESULT(0)$`.** The markers in a comparison report's texts are resolved by the web UI, not by the tools. Substitute them from `benchResultIds` or describe the section instead.
- **A template applied is a layout overwritten.** `apply_report_template` replaces the target report's items for good; there is no undo and no previous version to read back. Say so before calling it.
- **A `ThresholdAlarm` with 0-duration is an instantaneous breach.** Non-zero duration means a sustained one. Treat isolated 0-duration alarms as noise; clusters (multiple within seconds) are signal.
- **Playwright `Results Tree` rows can show negative response times.** `page.waitForTimeout` and actions inside nested `for` / `if` blocks can produce them in corner cases — don't propagate a negative value as an anomaly upstream.

## See also

- `octoperf-scenario-diagnosis` — workflow for diagnosing a poor run (this skill is the reading guide; scenario-diagnosis is the action plan).
- `octoperf-validation-triage` — when the report shows the VU itself is failing.
- OctoPerf bench reports docs: <https://doc.octoperf.com/analysis/bench-reports/>
