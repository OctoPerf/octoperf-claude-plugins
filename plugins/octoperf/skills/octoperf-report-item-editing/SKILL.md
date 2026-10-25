---
name: octoperf-report-item-editing
description: Use when changing what an OctoPerf bench-report item displays — its metrics, filters or curves — rather than reading its values (see octoperf-bench-reports for that). Triggers on "add a metric to this chart", "change the filter of this report item", "show errors per container in the report", "add a new chart to the report", "plot response time against the active users". Covers which metrics each report item type accepts, the edit workflow, adding an item instead of editing one, and plotting a line chart against a metric instead of time. Requires the OctoPerf MCP server.
---

# Editing OctoPerf bench-report items

This skill covers **changing** the metrics/filters of a bench report item (a
`BenchReportItem`) — not reading its values (see `octoperf://skills/bench-reports`
for the read side). A report item's metrics are constrained: only certain sample
types, metric ids and tag filters are valid for a given report item type. The backend
exposes that contract; use it instead of guessing.

## The model

A metric-bearing report item (`StatisticTableReportItem`, `LineChartReportItem`,
`PercentilesChartReportItem`, `SummaryReportItem`, `PieChartReportItem`,
`StatisticTreeReportItem`, `BarChartReportItem`, `AreaRangeChartReportItem`) holds a
list of `ReportItemMetric`:

- `id` — the `MetricId` (e.g. `RESPONSE_TIME_AVG`, `HITS_TOTAL`).
- `type` — the `SampleType` (`BASIC`, `WEB`, `LOAD`, `NUMBER_COUNTER`, `CUSTOM`, …).
- `filters` — a set of `SingleTermFilter{field,term}` / `MultiTermFilter{field,terms}`
  narrowing the query by tag (`region`, `injectorId`, `virtualUserName`, `actionPath`,
  `subSampleType`, …). AND-combined.
- `benchResultId`, `configs`.

Which `type`/`id`/filter is legal depends on the report item type, the report's bench
results and whether it is a trend report. That is the **allow-select descriptor**.

## Workflow

1. **Read the report** — `get_bench_report(reportId)` to see items and their ids/`@type`.
2. **Resolve what's allowed** — `get_report_item_allow_select(reportId, itemId)` returns:
   - `sampleTypes` — allowed sample types for the item.
   - `metricIdsBySampleType` — selectable metric ids per sample type.
   - `tagKeyConstraints` — per tag key: whether it's offered/`required`, `allowedValues`
     (a closed whitelist when non-empty, e.g. `subSampleType` ∈ {Hit, Container}),
     `single` (single vs multi select), and `dependsOn` (parent keys, e.g. `injectorId`
     depends on `region`).
3. **Discover filter values** (for open-ended tag keys):
   - `get_report_tag_keys(reportId, sampleType)` — the filterable tag keys.
   - `get_report_tag_values(reportId, sampleType, tagKeys, filters?)` — the concrete
     values, optionally scoped by already-selected `filters` (e.g. restrict `injectorId`
     values to a chosen `region` by passing `{"region":["eu-west-1"]}`).
4. **Build the metric(s)** using only allowed types/ids/fields/values.
5. **Validate** — `validate_report_item(reportId, itemId)` returns `ReportItemValidation`
   (`valid` + `violations`). Fix every violation before persisting.
6. **Persist** — `patch_bench_report(reportId, patch)` with an RFC 6902 JSON Patch
   (consult `octoperf://schema/bench-report` for the item shape). The server runs the
   same allow-select validation as a **pre-flight** and rejects the patch (with the
   violation messages) if any metric is invalid, so a green `patch_bench_report` means
   the edit was both structurally and semantically valid.

## Adding an item rather than editing one

The same `patch_bench_report` adds an item: `{"op":"add","path":"/items/-","value":{…}}`
with the subtype's `@type` and the fields `octoperf://schema/bench-report` requires. The
server round-trips it through Jackson and runs the same allow-select pre-flight, so a
green call means the new item is readable — `get_report_*_values` on the id it comes back
with returns its points straight away.

A monitoring curve is the common case, and the two filters are what make it one:

```json
{"@type": "LineChartReportItem", "name": "nginx accepts per sec",
 "metrics": [{"id": "MONITORING", "type": "NUMBER_COUNTER", "benchResultId": "…",
   "configs": [],
   "filters": [{"@type": "SingleTermFilter", "field": "connectionName", "term": "nginx local"},
               {"@type": "SingleTermFilter", "field": "counterPath", "term": "Connections⫽Connection Accepts per sec"}]}]}
```

`get_report_monitors_table_values` hands back both terms for every connection of the run,
which is where to get them rather than guessing. `MonitorsTableReportItem`,
`ThresholdAlarmReportItem` and `TextualMonitorReportItem` are added the same way and carry
a `benchResultId` rather than metrics.

A user-defined metric (a K6 `Trend`, `Counter`, `Gauge` or `Rate`, or a Core Web Vital of a
Playwright page: `LCP`, `FCP`, `TTFB`, `INP` in milliseconds, `CLS` as a score) is stored as a
`CUSTOM` sample and charted by the `CUSTOM_*` metric ids (`CUSTOM_AVG`, `CUSTOM_MIN`,
`CUSTOM_MAX`, `CUSTOM_SUM`, `CUSTOM_COUNT`, `CUSTOM_RATE`) in a `LineChartReportItem` or a
`StatisticTableReportItem`. Every `CUSTOM` metric of a line chart needs a `metricName`
filter; `get_report_tag_values(reportId, "CUSTOM", ["metricName"])` lists the names. A Core
Web Vital also carries the `actionPath` of the Playwright action that started the page, so an
optional `actionPath` filter (with its `virtualUserName`) charts one page; K6 metrics carry
none, and such a filter leaves them out. A statistic table needs no `metricName`: it shows one
row per metric name, all actions together.

```json
{"@type": "LineChartReportItem", "name": "checkout duration",
 "metrics": [{"id": "CUSTOM_AVG", "type": "CUSTOM", "benchResultId": "…", "configs": [],
   "filters": [{"@type": "SingleTermFilter", "field": "metricName", "term": "checkout_duration"}]}]}
```

Several names in one `MultiTermFilter` aggregate those metrics into a single curve:
`CUSTOM_SUM`, `CUSTOM_COUNT` and `CUSTOM_RATE` add them up (`orders_web` + `orders_mobile` is
every order), `CUSTOM_AVG` averages all their values, `CUSTOM_MIN` / `CUSTOM_MAX` take the
extreme. That only reads right for metrics of one kind and one unit, so `validate_report_item`
refuses names whose `metricKind` or `metricUnit` differ: chart unrelated metrics as separate
curves instead.

Two `CUSTOM` curves of one chart with different `metricName`s land on the same Y axis whenever
their metric ids share a unit: `CUSTOM_AVG`, `_MIN`, `_MAX` and `_SUM` all draw on the "value"
axis, while `CUSTOM_COUNT` and `CUSTOM_RATE` have axes of their own. A checkout duration in
milliseconds next to a cart size of 3 then flattens the smaller one. Give every such curve after
the first its own axis with an `AxisReportConfig` in its `configs`, as the report editor does when
it detects the conflict:

```json
{"id": "CUSTOM_AVG", "type": "CUSTOM", "benchResultId": "…",
 "configs": [{"@type": "AxisReportConfig", "dedicated": true}],
 "filters": [{"@type": "SingleTermFilter", "field": "metricName", "term": "cart_items"}]}
```

Curves of the same `metricName` (its avg, min and max) keep sharing one axis.

The `metricKind` (`TREND`, `COUNTER`, `GAUGE`, `RATE`) and `metricUnit` (`DEFAULT`, `TIME`
in milliseconds, `DATA` in bytes) tags say how to read the value. Copy the `metricUnit` into a
`{"@type": "UnitReportConfig", "unit": "TIME"}` in the metric's `configs`, as the report editor
does: a `TIME` curve is then drawn in milliseconds on the axis it shares with the response times,
a `DATA` one in bytes, and only `DEFAULT` curves of different names still need the dedicated axis
above.

## Plotting a line chart against a metric instead of time

A `LineChartReportItem` plots its curves against time unless it carries an `xAxis`: then
each curve is drawn against a metric — response time against the active users, say. Set it
with a patch on an existing chart, or in the `value` of a new one:

```json
[{"op": "add", "path": "/items/2/xAxis",
  "value": {"id": "USERLOAD", "type": "LOAD",
            "binning": {"@type": "AutoBinning"}, "spread": "NONE"}}]
```

- `id` and `type` are required. `binning` defaults to `AutoBinning` (the server picks the
  classes); `{"@type": "FixedBinning", "size": 10}` groups the X values by multiples of
  `size` instead. `spread` (`NONE`, `STD`, `MIN_MAX`) only sets the band the UI draws.
- The legal X metrics are in `xAxisMetricIdsBySampleType` of
  `get_report_item_allow_select` — `LOAD` and `BASIC` metrics, a line chart only.
- The X carries no `benchResultId` and no filters: each curve reads its X on its own run,
  narrowed by the curve's filters its sample type knows (`virtualUserName` applies to a
  `LOAD` X, `actionPath` does not).
- `validate_report_item` and the `patch_bench_report` pre-flight report on the `xAxis`
  field when the X is not offered, when it is also one of the curves (same `id` and `type`),
  or when it is `BASIC` while a curve is not — a `BASIC` X takes its Hit/Container
  `subSampleType` from each curve.
- `{"op": "remove", "path": "/items/2/xAxis"}` plots the chart against time again.
- An `xAxis` names its type: `"@type": "MetricXAxis"` or `"@type": "TimeXAxis"`. One
  without `@type` is refused.
- The `size` of a `FixedBinning` is in the unit the X metric is stored in: milliseconds for the
  response time, the latency and the connect time, bytes for a size, users for `USERLOAD`.
  Latencies of about one second in classes of 10 ms are `"size": 10`, not `0.01`.

### A spread band on a chart plotted against time

A time chart draws a band when its `xAxis` is a `TimeXAxis`:

```json
[{"op": "add", "path": "/items/2/xAxis",
  "value": {"@type": "TimeXAxis", "spanIntervals": 3, "spread": "STD"}}]
```

- `spanIntervals` (≥ 1) counts the sampling intervals taken on **each side** of a bucket: 3
  spans 7 buckets, 35 s at the default 5 s sampling. Each point becomes the mean of its span,
  so the curve turns into a moving average while the band is drawn.
- `spread` is `STD` or `MIN_MAX`. `NONE` is refused: a time chart without a band carries no
  `xAxis` at all.

Read the result with `get_report_line_chart_values`; `octoperf://skills/bench-reports`
explains its fields.

## Notes

- The `subSampleType` tag carries the former Hit/Container distinction inside `BASIC`
  (values `Hit` / `Container`) — it is a closed whitelist, not an open ES value.
- Value checks for open-ended keys (`region`, `injectorId`, `actionPath`,
  `virtualUserName`) are made against the values actually indexed for the report's bench
  results — a value absent from Elasticsearch is rejected.
- `dependsOn`: include the parent filter (e.g. `region`) whenever you set a dependent
  one (e.g. `injectorId`), or validation fails.
