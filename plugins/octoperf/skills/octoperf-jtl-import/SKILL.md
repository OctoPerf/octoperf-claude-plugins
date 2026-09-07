---
name: octoperf-jtl-import
description: Use when someone has JMeter result files and wants OctoPerf's analysis of them rather than a new run. Triggers on "import my JTL", "analyse these JMeter results", "I ran JMeter from the command line / in CI, can you build a report", "results.jtl", "turn my .jtl into a report". Covers what to check before uploading, how to zip, and how to read the report that comes out. Requires the OctoPerf MCP server.
---

# OctoPerf — Importing JMeter JTL results

`import_jtl_report` mints a presigned URL; you POST a zip of CSV JTL
files to it and OctoPerf builds a bench result and a report out of the
samples. The bytes never go through the MCP server, so a large archive
is fine — up to 200 MB.

Most of what makes an import good or useless happens **before** the
POST. Work in this order.

## 1. Check the format before anything else

**The archive must be a `.zip`.** Nothing else is unpacked — a bare
`.jtl`, a `.tar.gz`, a `.7z` all come back as
`only .zip files are accepted`.

**The JTL files inside must be CSV, not XML.** Look at the first line:
if it starts with `<?xml`, stop and say so. There is no converter — the
test has to be re-run with `jmeter.save.saveservice.output_format=csv`,
which is the default, so an XML file means someone changed it on
purpose.

**Ten columns are mandatory**, and the import refuses the file by name
if any is missing — `results.jtl has missing column(s) => [SENT_BYTES]`:

`timeStamp`, `label`, `URL`, `Latency`, `Connect`, `elapsed`,
`responseCode`, `allThreads`, `bytes`, `sentBytes`

A JMeter run left at its default `saveservice` settings writes all ten,
so a plain `jmeter -n -t plan.jmx -l results.jtl` is enough. Read the
header line anyway and name what is missing — it is faster than a failed
import.

## 2. The two settings that quietly ruin a report

Both pass the mandatory-column check. Neither is visible in the result.

**`timestamp_format` set to a date pattern.** OctoPerf reads `timeStamp`
as epoch milliseconds. JMeter's default is `ms`, but
`jmeter.properties` ships a commented
`jmeter.save.saveservice.timestamp_format=yyyy/MM/dd HH:mm:ss.SSS`
that teams enable to make the file readable by eye. With it, the import
fails on the first line. Check that the first data row's `timeStamp` is
a long, not a date.

**`sample_count` left at its default `false`.** This is the one that
costs the most, because nothing about it looks wrong. OctoPerf counts a
sample as failed when `ErrorCount` is above zero **or** when
`failureMessage` is non-empty — it never reads JMeter's `success`
column. `ErrorCount` is only written when
`jmeter.save.saveservice.sample_count=true`, which is off by default.
So on a default JTL, an HTTP 500 that no assertion caught is imported
as a **success**, and the report shows a 0% error rate over a run that
failed.

Tell the user before importing. Either re-run with
`-Jjmeter.save.saveservice.sample_count=true`, or accept that only
assertion failures will show — and say which of the two the report they
are about to read reflects.

## 3. Zip it

- **One JTL file per thread group.** Every entry in the archive that
  yields samples becomes one OctoPerf user profile, in archive order.
  Merging two thread groups into one file merges their load curves.
- **Samplers are aggregated by `label` across entries**, which is what
  makes a per-thread-group split safe: the same request appearing in
  two files stays one action.
- **Name the archive after the test.** The name, minus its extension,
  becomes the name of the scenario *and* of the report —
  `checkout-soak-2026-08.zip` reads better in a report list than
  `results.zip`.
- **Put nothing else in the zip.** Every entry is parsed as a JTL; a
  `jmeter.log`, a `.jmx` or a `report/` folder from an HTML dashboard
  will fail the header check.
- **One zip is one report.** Unrelated runs in the same archive produce
  one incoherent report, not several.

## 4. Getting the files from the user

There is nothing to configure — you fetch the bytes yourself and POST
them.

- **With a filesystem and a shell** (Claude Code, CLI agents): ask which
  directory holds the results, zip the JTL files there, and POST with
  your shell. This is the only route that works for a large archive.
- **In a hosted code interpreter** (claude.ai web / desktop): the user
  attaches the archive to the conversation and you POST it from the
  interpreter. The attachment ceiling is far below OctoPerf's 200 MB —
  fine for a smoke test, not for a soak run. Say so early rather than
  after a failed upload; a user in that situation is better served by
  running the import from a terminal.

## 5. Import and follow it

1. `import_jtl_report(projectId)` → a presigned `url`, valid ~5 minutes,
   single-use.
2. POST the zip as `multipart/form-data`, one part named `file`,
   Content-Type `application/zip`, the archive's name in
   Content-Disposition.
3. The response is `{taskId, benchResultId, reportName}` — the import is
   already running.
4. Poll `get_task_result(taskId)` every 2–3 seconds. A big archive takes
   minutes; see `octoperf-async-polling` for the cadence.
5. On `SUCCESS`, call `list_bench_reports_by_project(projectId)` and take
   the report whose `benchResultIds` contains `benchResultId`. Hand the
   user its `url`.

On `FAILED`, the `message` carries the backend error. The two you will
actually see are the missing-column message and the `.zip` refusal,
both naming the file at fault.

## 6. Say what the report does not contain

A JTL is a record of what the injector measured, not of the test. Set
expectations before the user goes looking:

- **No monitoring.** No CPU, no memory, no database counters — a JTL
  holds none, so the monitoring items of the report stay empty.
- **No HTTP method, no Content-Type.** The format does not record them,
  so requests are identified by their `label` and `URL` alone.
- **The Virtual User tree is reconstructed, and approximately.** Actions
  are rebuilt from labels; a row whose `URL` is the literal `null` is
  read as a container, which is how transaction controllers reappear.
  Treat the tree as a reading aid, not as a script to re-run.
- **The user load is inferred from `allThreads`** and simplified — the
  curve keeps its shape, not its every point, and it has no data for
  the ramp-up before the first sample or after the last one.
- **The report is a standard OctoPerf report** on the run it just
  created: every `get_report_*_values` tool works on it, and it can be
  compared against a real OctoPerf run like any other.
