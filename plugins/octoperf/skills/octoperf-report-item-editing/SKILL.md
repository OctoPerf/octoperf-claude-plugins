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
- `type` — the `SampleType` (`BASIC`, `WEB`, `LOAD`, `NUMBER_COUNTER`, …).
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

## Notes

- The `subSampleType` tag carries the former Hit/Container distinction inside `BASIC`
  (values `Hit` / `Container`) — it is a closed whitelist, not an open ES value.
- Value checks for open-ended keys (`region`, `injectorId`, `actionPath`,
  `virtualUserName`) are made against the values actually indexed for the report's bench
  results — a value absent from Elasticsearch is rejected.
- `dependsOn`: include the parent filter (e.g. `region`) whenever you set a dependent
  one (e.g. `injectorId`), or validation fails.
