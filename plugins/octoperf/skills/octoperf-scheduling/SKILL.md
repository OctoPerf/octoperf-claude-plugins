---
name: octoperf-scheduling
description: Use when scheduling an OctoPerf scenario to run at a specific time (one-shot) or on a recurring cadence (cron), or when listing / pausing / resuming / deleting an existing schedule. Triggers on "schedule the scenario for tomorrow morning", "run this every weekday at 8am", "every night at midnight", "pause the cron job", "delete the schedule", "show scheduled jobs". Covers the unusual cron format (Unix 5-field UTC, NOT Quartz), the timezone conversion gymnastics, the pre-flight rule (a misconfigured scenario will fire failing runs forever until disabled), and the full job lifecycle. Requires the OctoPerf MCP server.
---

# OctoPerf — Scheduling scenarios

OctoPerf can fire a load-test scenario on a schedule — either once at
a future moment, or recurrently via a cron expression. **Every fire
consumes credits exactly like a manual `run_scenario`**, so a daily
cron drains the subscription every day until you pause or delete it.

## When this applies

- The user wants a scenario to run automatically — overnight batch,
  weekly soak test, regression run after a release tag, etc.
- The user is asking about an existing scheduled job — show it, pause
  it, change its cadence, delete it.

## The pitfalls — read this BEFORE invoking any schedule tool

### 1. The cron format is Unix 5-field, not Quartz, and it's in UTC

OctoPerf's cron parser is `UnixCronTrigger` — **5 fields, no seconds,
evaluated in UTC** on the server. Quartz expressions like
`0 0 3 * * ?` (6 fields) are rejected with an opaque HTTP 400.

The five fields are:
```
minute  hour  day-of-month  month  day-of-week
0-59    0-23  1-31           1-12   0-7 (0 and 7 are Sunday)
```

Use `*` for "any value". Comma lists (`1,15`), ranges (`1-5`) and step
values (`*/15`) work. Day-of-week uses Unix conventions (1=Monday in
the documented examples).

### 2. UTC, not local time — convert explicitly

A user saying "every night at midnight" almost never means UTC. You
have to convert their local time to UTC before computing the
expression. Examples for Europe/Paris:

| User's local cadence              | DST (CEST = UTC+2) | Winter (CET = UTC+1) |
|-----------------------------------|--------------------|----------------------|
| Daily at midnight Paris           | `0 22 * * *`       | `0 23 * * *`         |
| Daily at 09:00 Paris              | `0 7 * * *`        | `0 8 * * *`          |
| Weekdays at 08:30 Paris           | `30 6 * * 1-5`     | `30 7 * * 1-5`       |
| Monthly on the 1st at 03:00 Paris | `0 1 1 * *`        | `0 2 1 * *`          |

**Daylight Saving Time is a real concern**: a daily Paris cron set in
summer will fire one hour off in winter (and vice versa). The user
either accepts that drift, or you advise picking a less DST-sensitive
slot (e.g. anchor on UTC and tell them when it fires in local time).

When in doubt, ask the user which fixed-UTC slot they want, or use a
single-fire `schedule_scenario_once` with an ISO-8601 offset which is
unambiguous.

### 3. Pre-flight FIRST — a misconfigured cron fires failing runs forever

Before `schedule_scenario_*`, call `get_scenario_matching_plans` on
the scenario. If it returns an empty list, the scheduled run will
fail at startup *every single fire* until someone notices and disables the job.

```
mcp__octoperf__get_scenario_matching_plans(scenarioId)
```

Empty result → flag to the user, run `list_active_subscriptions` to
explain which cap is binding, **do not schedule**.

A scheduled cron is also more dangerous than a manual `run_scenario`
because:
- The failure is asynchronous (the user might not see it for days)
- The credit cost compounds (1/day × 30 days = 30 credits for nothing)
- The orchestration log is the only diagnostic when the run errored
  before producing samples (`list_bench_docker_logs`)

## Steps

### 1. Decide one-shot vs recurring

- **One-shot** (`schedule_scenario_once`) — for "run X at this specific
  moment". Takes an ISO-8601 datetime with timezone offset
  (`2026-06-15T03:00:00+02:00`) — unambiguous, no timezone math.
- **Recurring** (`schedule_scenario_cron`) — for "every N days /
  hours / Monday morning / etc.". Requires the Unix 5-field UTC cron.

### 2. Build the trigger value

For one-shot:
- Take the user's local datetime, write it ISO-8601 with the
  appropriate offset for that wall-clock date (mind DST).
- Example: "tomorrow at 6 PM Paris time" on 2026-05-15 = `2026-05-15T18:00:00+02:00`.

For recurring:
- Convert the local time of day to UTC for the current DST state.
- Surface the DST caveat to the user if their cadence crosses DST
  transitions.

### 3. Pre-flight + schedule

```
mcp__octoperf__get_scenario_matching_plans(scenarioId)   # MUST be non-empty
mcp__octoperf__schedule_scenario_once(scenarioId, runAt, name)
# or
mcp__octoperf__schedule_scenario_cron(scenarioId, expression, name)
```

Always pass an explicit `name` — empty names make the scheduler view
in the OctoPerf UI hard to read.

### 4. Confirm the next fire time

Both tools return `nextRun` (epoch-ms). Render it back to the user in
their local time so they can sanity-check ("nextRun = 2026-05-15T00:00
Paris ✓" — not "1778796000000"). If it lands on the wrong instant,
the UTC conversion is off — fix the cron and re-schedule.

### 5. Manage existing jobs

```
mcp__octoperf__list_scheduled_jobs_by_project(projectId)
mcp__octoperf__enable_scheduled_job(jobId)    # re-arms credit-consuming runs
mcp__octoperf__disable_scheduled_job(jobId)   # safe pause, reversible
mcp__octoperf__delete_scheduled_job(jobId)    # destructive — prefer disable
```

The list returns every job in the project — including the disabled
ones. Group by `enabled` when reporting to the user.

### 6. Recovery — when a scheduled run silently failed

A scheduled fire that errored before producing samples won't show up
in `list_bench_reports_by_project` like a real run. To audit:

- `list_scheduled_jobs_by_project(projectId)` → check each job's
  `nextRun` — if a daily cron's nextRun is in the past, something is
  off (the trigger may be misconfigured).
- For each suspect job, get the recent benchResults of the scenario
  (`list_bench_reports_by_project` filtered on the scenario's id) and
  check their state via `get_bench_result`. ERROR-state runs without
  metrics = the fire happened but the scenario didn't start.
- If you find an ERROR run, follow the `octoperf-scenario-diagnosis`
  step 0 ("did the run even start?") — `list_bench_docker_logs` on
  the bench result id is the next step.

## Pitfalls

- **Don't pass a Quartz expression** — it'll be rejected with HTTP 400. Use Unix 5-field.
- **Don't forget UTC** — every cron expression is evaluated in UTC. A cron typed in local time is silently wrong.
- **Don't schedule a scenario you haven't pre-flighted** — the cost compounds with the cadence.
- **Don't delete to pause** — `delete_scheduled_job` is irreversible. Use `disable_scheduled_job` and keep the config around.
- **Don't ignore DST** — a cron that's right in May will be one hour off in November (and vice versa). Mention it to the user when relevant.

## See also

- `octoperf-scenario-diagnosis` — when a scheduled run produced bad metrics or errored before sampling.
- OctoPerf scheduler docs: <https://doc.octoperf.com/runtime/scheduler/>
