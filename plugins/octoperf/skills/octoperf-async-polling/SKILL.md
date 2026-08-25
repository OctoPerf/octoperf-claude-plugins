---
name: octoperf-async-polling
description: Use whenever an OctoPerf operation runs asynchronously and the LLM has to wait for it to settle — `validate_virtual_user`, `run_scenario`, `export_bench_report_pdf`, the async correlation tasks behind `apply_correlations_to_virtual_user`, or any tool that returns a `taskId` / `benchResultId` instead of the final result. Defines the cadence, the terminal conditions, and the anti-patterns so the LLM does not tight-loop the MCP server or sleep blindly for the full expected duration.
---

# OctoPerf — Polling async operations reliably

Several OctoPerf MCP tools kick off work that runs for seconds to
hours and return immediately with a handle (`benchResultId`,
`taskId`, …). The actual result lives behind a second "get-status"
tool that you poll until terminal. This skill is the **single source
of truth** for how to do that polling without burning the context
window or missing an early failure.

## When this applies

You started one of:

| Starter tool                                       | Returns                               | Status tool to poll                                       |
|----------------------------------------------------|---------------------------------------|-----------------------------------------------------------|
| `validate_virtual_user`                            | `benchResultId`                       | `get_virtual_user_validation` (or `get_bench_result`)     |
| `run_scenario`                                     | `benchReportId` + `benchResultIds[]`  | `get_bench_result` (state) and/or `get_bench_status` (%)  |
| `export_bench_report_pdf`                          | `taskId`                              | `get_task_result`                                         |
| `update_report_data`                               | one `taskId` per run to update        | `get_task_result`                                         |
| Async correlation task (auto-correlate workflow)   | `taskId`                              | `get_task_result`                                         |

If your call returned the final answer (no `taskId` / no
`benchResultId`), you're not in async territory — skip this skill.

## The pacing rule

**Pace the polls.** Each MCP roundtrip takes 100–500 ms of
context. Calling the status tool in tight succession over a 10-min
test would consume 100+ tool turns with identical responses. Put a
delay of `N` between polls — the *how* depends on your harness (see
"How to insert the delay" below).

Pick `N` from the expected duration of the work:

| Expected duration              | Sleep between polls (`N`) |
|--------------------------------|---------------------------|
| Validation run (~30s)          | 5s                        |
| 1-min smoke scenario           | 5–10s                     |
| 5-min scenario                 | 30s                       |
| 10-min scenario                | 60s                       |
| 30-min + soak / load test      | 60s (cap)                 |
| PDF export task (10–60s)       | 3s                        |
| Async correlation (5–60s)      | 3s                        |

Rule of thumb: `N ≈ expected_duration / 10`, clamped to `[3s, 60s]`.
The cap matters — there is no benefit to sleeping longer than 60s
even on a multi-hour run, because a hard cap keeps the LLM
responsive if the run aborts early (ERROR, manual stop).

### How to insert the delay (harness-dependent)

The cadence table is *what* — the *how* depends on which agent harness
is driving this server:

- **Harnesses that allow a bounded blocking sleep** (most CLI agents):
  insert a bounded `Bash sleep N` (or equivalent) between two MCP
  calls. Do **not** chain multiple short sleeps to fake a longer one.
- **Claude Code blocks *all* blocking sleeps used as a wait** — even a
  bounded `sleep 60` between two MCP calls is intercepted (it suggests
  `Monitor` or `run_in_background` instead). Use its native
  recurring-prompt mechanism: `/loop <interval> <poll-prompt>` (or
  `CronCreate` directly) re-enters the poll on a wall-clock interval,
  so the cadence table maps straight to the `/loop` interval (`60s`
  rounds to cron's 1-min minimum). Each fire re-invokes the MCP status
  tool; stop the loop with `CronDelete` once the state is terminal.

**Event-watchers don't help here.** The run state lives *behind an MCP
status tool*, not in any file, log line, or local process. So
shell-level watchers (Claude Code's `Monitor`, `tail -f`,
`inotifywait`, a `run_in_background` until-loop) cannot observe it
without calling the platform REST API directly — which needs the OAuth
token the MCP server holds. The wake mechanism must re-call the MCP
status tool, which is exactly what a recurring prompt does.

## The polling loop

Pseudocode:

```
start    = start_async_tool(...)        # returns id (taskId / benchResultId)
handle   = start.benchResultId | start.taskId
duration = estimated_run_seconds        # from the scenario / task

while True:
    status = get_status_tool(handle)
    if is_terminal(status):
        break
    wait N                              # bounded sleep, or harness scheduler — see below
```

### Terminal conditions (per status tool)

- `get_virtual_user_validation` → `finished == true`. The `state` field
  is one of `CREATED / PENDING / SCALING / PREPARING / INITIALIZING /
  RUNNING / FINISHED / ABORTED / ERROR`; treat the last three as
  terminal.
- `get_bench_result` → `state ∈ {FINISHED, ABORTED, ERROR}`. Same
  state machine as validation (validation runs reuse the bench
  pipeline). This is the **canonical** terminal check for a load run.
- `get_bench_status` → returns a numeric progress value. **Do not use
  this for terminal detection.** It returns elapsed-as-percent (0 →
  ~100) for in-flight runs and is useful for "show progress to the
  user" updates, but it does not flip to a sentinel value when the
  run reaches ABORTED / ERROR. Always cross-check with
  `get_bench_result.state`.
- `get_task_result` → `status ∈ {SUCCESS, FAILED}`. `PENDING` is the
  in-flight value. The `message` field carries the backend error on
  `FAILED` — surface it verbatim and stop.

### After the loop

The status tool already tells you the outcome (FINISHED vs ABORTED vs
ERROR, SUCCESS vs FAILED). Branch from there:

- **Terminal-success path** (`FINISHED` / `SUCCESS`) → proceed to the
  workflow-specific next step (read report, fetch URL, …).
- **Terminal-failure path** (`ABORTED` / `ERROR` / `FAILED`) → do
  not retry blindly. Pull the explanation from the matching skill:
  - `octoperf-validation-triage` (`benchResultId` failed validation)
  - `octoperf-scenario-diagnosis` (`benchResultId` failed under load)
  - `octoperf-export-bench-report-pdf` (PDF task failed)

## Estimating `expected_duration`

For a **scenario run**, read the scenario before launch:
`get_scenario(scenarioId)` exposes `userProfiles[].loadStrategy` —
sum `rampUpSec + holdForSec + rampDownSec` for the longest profile
and add ~30s of provisioning/teardown headroom. That sum is your
`expected_duration`; feed it into the table above.

For a **validation run**, default to 30s — the platform caps a single
VU iteration at well under a minute. Bump to 60s for Playwright VUs
(real browsers are slower).

For a **PDF export** / **correlation task**, you don't know the
duration upfront. Use the table's `3s` cadence with a generous outer
deadline (~5 min for PDF, ~2 min for correlation); if the task is
still PENDING past that, surface to the user.

For a **report data update** (`update_report_data`), the duration
scales with how many samples the run holds — seconds for a smoke test,
minutes for a long high-load run. Poll every `10s` with a ~15 min
deadline, and tell the user what is happening rather than going quiet:
the report items stay unreadable until each task reaches SUCCESS. The runs
being updated are listed by `get_report_data_status`; see
`octoperf-bench-reports` for when a read refuses in the first place.

## Anti-patterns

- **Tight-looping the status tool with no delay.** Burns 1 tool turn
  every few hundred ms on identical responses; for a 10-min run that
  is 1000+ wasted turns. Always pace the polls — a bounded sleep where
  the harness allows it, otherwise the harness's recurring-prompt
  scheduler (Claude Code: `/loop`).
- **Sleeping for the full expected duration in one shot.** You miss
  early `ERROR` / `ABORTED` transitions (e.g. capacity exhausted at
  startup, image pull failure) and the user waits the whole run for a
  failure that surfaced in the first 10 seconds.
- **Using `get_bench_status` as the terminal check.** It returns a
  progress percentage, not a terminal state — a run that ABORTs at
  50% will leave `get_bench_status` stuck near 50 forever. Always
  cross-check with `get_bench_result.state` for the actual outcome.
- **Polling without first checking the start tool returned a usable
  id.** `validate_virtual_user` / `run_scenario` can return a 4xx
  before any work starts (no matching plan, expired auth, …); polling
  a missing id just produces noise.
- **Polling after the resource is already terminal.** Once you have
  `state=FINISHED`, stop calling the status tool — the next call
  burns context for the same value.

## See also

- `octoperf-validation-triage` — what to do once a validation run reaches a terminal state with failures.
- `octoperf-scenario-diagnosis` — what to do once a scenario run reaches a terminal state and the metrics look bad.
- `octoperf-auto-correlation` — uses async correlation tasks; the polling here applies to their `taskId`.
- `octoperf-export-bench-report-pdf` — uses async print tasks; the polling here applies to its `taskId`.
