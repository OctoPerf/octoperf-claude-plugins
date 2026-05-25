---
name: octoperf-export-bench-report-pdf
description: Use when the user asks to "export the report as PDF", "print the bench report", "get a PDF of report X", "share a PDF with stakeholders", or any variation that calls for a static artefact of an OctoPerf benchReport. Walks the LLM through the three-step async chain (submit print task → poll → download presigned URL). Requires the OctoPerf MCP server to be connected.
---

# OctoPerf — Export a benchReport as PDF

OctoPerf prints reports server-side with a headless Playwright render.
The work is async: the MCP tool submits the task, the LLM polls until
it settles, then pulls the PDF off the report's first bench result.

## When this applies

The user wants a static, shareable artefact of an existing bench
report — a stakeholder review, an attachment to a ticket, an offline
archive. If the user just wants to *read* metrics, use the report
value tools (`get_report_summary_values`, `get_report_table_values`,
…) instead. The PDF is for hand-off, not analysis.

## Prerequisites

- A `benchReportId` (find it with `list_bench_reports_by_project`).
- The underlying bench run is terminal (FINISHED / ABORTED / ERROR).
  Printing a report whose run is still RUNNING is allowed but the
  metrics inside the PDF will be partial.

## The three-step chain

### 1. Submit the print task

```
export_bench_report_pdf(benchReportId)
# returns { benchReportId, taskId }
```

The tool returns immediately with a `taskId`. Defaults are sensible
(portrait A4, empty cover page, en-US locale, UTC timezone) — no extra
parameters needed for the common case.

### 2. Poll until the task settles

```
get_task_result(taskId)
# returns { taskId, status, message }
```

`status` cycles through `PENDING` while Playwright is rendering. Stop on:

- `SUCCESS` → the PDF has been uploaded to the report's first bench
  result. Continue with step 3.
- `FAILED` → `message` carries the backend stack trace. Surface it to
  the user verbatim and stop.

Typical render time is 10–30 seconds; expect more on large reports
(many widgets, long trend series). For the polling cadence and the
sleep-between-polls discipline, read `octoperf-async-polling` — the
PDF task is in its table (3s between polls, ~5 min outer deadline).

### 3. Locate the PDF and hand the user a download URL

The PDF lands on the *first* benchResult of the report
(`report.benchResultIds[0]`). For SIMPLE reports that's the only one;
for TREND reports, only the anchor result carries the file.

```
get_bench_report(benchReportId)
# read .benchResultIds[0]

list_bench_result_files(benchResultId)
# find the newest `.pdf` entry — filename is derived from the report
# name with non-word chars replaced by `_`, e.g. `My_Load_Test.pdf`

download_bench_result_file(benchResultId, filename)
# returns { url, method: "GET", expiresAt, instructions }
```

Hand `url` to the user. The single-use token is consumed on the first
GET and the URL expires in ~5 minutes — re-call
`download_bench_result_file` if the user needs a fresh link.

## Gotchas

- **TREND / COMPARISON reports**: the PDF is attached to the
  *anchor* bench result only, not to every result in the trend.
  `list_bench_result_files` on a non-anchor benchResult won't see it.
- **Re-exporting**: re-running `export_bench_report_pdf` produces a
  new file with the same sanitized filename — the previous PDF is
  overwritten on the storage layer.
- **Filename collisions**: two reports with the same name (after
  `\W+ → _` sanitisation) would land on the same filename if they
  share a benchResult. In practice reports rarely share a benchResult
  outside of trend/comparison setups, so this is mostly a curiosity.
- **Don't busy-loop on `get_task_result`.** See
  `octoperf-async-polling` — insert a bounded `Bash sleep` (3s for
  the PDF task) between every `get_task_result` call.

## See also

- `octoperf-async-polling` — full polling cadence + terminal conditions for `get_task_result`.
- `octoperf-bench-reports` — when the user wants the same metrics interactively rather than as a PDF.
