# Monitoring — watch servers during a load test

Collect infrastructure metrics (OS, database, web/app server, JMX, Prometheus,
New Relic, SLA) **while a scenario runs**, so a slow or failing test can be tied
to what the monitored systems were doing. Read this before creating or
configuring a monitor.

## Mental model

- A **monitor** (`MonitorConnection`) watches **one external target** — a Linux
  host, a database, nginx/Tomcat, a JMX endpoint, … — during runs of a
  **project**'s scenarios. Monitors are **project-scoped**.
- A monitor runs **through an agent**: the agent opens the connection (SSH /
  JDBC / JMX / HTTP) to the target and samples counters. **Agents are
  workspace-scoped** — pick one from the workspace that owns the monitor's
  project.
- **Reachability is the whole point**: the chosen agent must be able to reach
  the target (same network, DNS, ports). For a Linux host the agent often runs
  *on* that host (SSH to `localhost`); for a database it must reach the JDBC
  endpoint. An agent that can't reach the target only produces connection
  errors — that is what `check_monitor_connection` catches.
- Credentials are **write-only**: they are sent when creating a monitor and are
  never returned by `get_monitor` / `list_monitors_by_project`.

## How a monitor fits a run (why it matters)

You don't attach a monitor to a scenario — it's automatic:

- Every **ENABLED** monitor of the project is collected on **every** scenario
  run of that project (disable one with `update_monitor` to exclude it from runs
  without deleting it). There is no per-scenario wiring.
- Most monitors ship with **default counters and default thresholds**; a
  breached threshold raises a **monitoring alarm** during the run. The SLA
  monitor is the same mechanism applied to the test's *own* metrics (response
  time, error rate) — perf alerts while the test runs.
- The values and alarms show up in the bench **report** (live and final). Read them
  with `get_report_monitors_table_values` (one row per connection of the run: its
  counters, its alarm count, the worst severity), `get_report_threshold_alarms`
  (each breach), `get_report_textual_monitors` (string-valued counters), and the
  generic `get_report_*_values` tools for the numeric curves.
- **Adding a monitoring chart to a report is an MCP move too**, not a UI-only one:
  `patch_bench_report` accepts a `LineChartReportItem` whose metric is
  `{id: MONITORING, type: NUMBER_COUNTER}` filtered on `connectionName` and
  `counterPath` — both are tag values `get_report_monitors_table_values` hands back
  ready to use. See `octoperf://skills/report-item-editing`.

So the useful loop is: create + enable a monitor → run the scenario →
read its counters/alarms in the report.

An enabled monitor whose counters carry thresholds is half of what the scenario page
calls **Pass Criteria** — the other half being the SLA profiles its Virtual Users
carry. `octoperf://skills/scenario-composition` says what the step covers and how to
read the ones a test already has.

## Pick an agent first

`list_workspace_agents(workspaceId)` lists the workspace's agents with their
`state`. **Only an `UP` agent can run a monitor.** Remember the project↔workspace
gap: pass the workspace that owns the monitor's project, then keep the UP agents
that can reach your target. (`list_provider_agents(providerId)` is the
provider-scoped equivalent from the on-premise skill.)

## Create a monitor — one tool per type

Each type has its own `create_*_monitor` tool that takes the connection details
and returns the created monitor:

- **`create_linux_monitor`** — SSH host + username + password *or* sshPrivateKey.
- **`create_postgres_monitor` / `create_mysql_monitor` /
  `create_oracle_monitor`** — a JDBC url + credentials. **SQL Server is not in
  this group**: see `create_sqlserver_monitor` below.
- **`create_mongodb_monitor`** — host/port/database (+ optional credentials).
- **`create_nginx_monitor` / `create_apache_httpd_monitor` /
  `create_lighttpd_monitor`** — a status-page url (stub_status / mod_status).
- **`create_prometheus_monitor`** — a `/metrics` url.
- **`create_generic_jmx_monitor`** — a full JMX service url; **`create_tomcat_monitor`**
  — host + JMX-RMI port (+ domain, default `Catalina`); **`create_jmeter_monitor`
  / `create_iis_monitor` / `create_windows_monitor` / `create_sqlserver_monitor`**
  — host + JMX-RMI port. IIS, Windows and **SQL Server** read Windows performance
  counters through the OctoPerf Windows/JMX bridge installed on the target host
  (typically port 1099) — SQL Server is *not* monitored over JDBC.
- **`create_newrelic_monitor`** — New Relic REST API url + api key.
- **`create_sla_monitor`** — a self-contained monitor of the *test's own*
  per-request metrics (times, errors, percentage counters with pass/fail
  thresholds). No target, no settings — just an agent. Independent of the
  Design-side SLA profiles.

Creating contacts the agent to validate the connection and discover a sensible
**default** counter set, so it takes a few seconds and fails with the real
connection error if the agent can't reach the target.

## Configure counters & applications (create-then-configure)

Creation auto-selects reasonable **default** counters for most types — enough for
OS / DB / SLA monitors. Two things can still leave a monitor collecting nothing:

**Four types auto-select NOTHING at all.** `create_prometheus_monitor`,
`create_jmeter_monitor`, `create_newrelic_monitor` and
`create_generic_jmx_monitor` ship no default rules, because their counter set is
defined by the target (arbitrary Prometheus metric names, an arbitrary MBean tree,
a New Relic account's metrics) rather than by the type. The monitor is created
**enabled, reachable, and collecting zero counters** — it simply never appears in
the report. For these four, `preview_monitor_counters` +
`update_monitor_counters` is a **required step**, not a refinement. On a target
exposing hundreds of counters the short path is `preview_monitor_counters` → read
its `groups` → `update_monitor_counters` with a few family patterns
(`"prometheus_tsdb_*"`), no need to enumerate every metric name.

Creation **runs the type's connection check first and persists nothing when it
fails**, so a monitor that exists is one the agent could actually collect from. That
check is stricter than opening a session: a Linux host answers over SSH and still
hands back its 44 standard counters while missing the `sysstat` package it needs to
collect any of them, and that now fails the creation instead of producing a monitor
that never reports.

Read `selectedCounterCount` on the creation result: **zero means the monitor is a
silent no-op.** `discoveredCounterCount` tells you which problem you have — a
non-zero discovery against a zero selection means the selection step is missing,
while a zero discovery means the target answered but exposed nothing usable.

**Application selection is mandatory for some types**: a Tomcat monitor collects
nothing useful until you pick the **webapp(s)** to watch. Applications are the
disks / network interfaces / processes of a Linux host, or the webapps of a Tomcat
server, …

Configure any existing monitor (this even works post-creation, which the web UI
does not offer) with three tools:

1. **`list_monitor_applications(monitorId)`** → the applications the monitor can
   collect counters for (`{type, name}`): a Tomcat server's webapps, a Linux
   host's disks / NICs / processes, a Windows or IIS host's CPUs / disks / NICs /
   processes, a SQL Server's databases / locks / plan caches. Empty for the types
   with no application step (JDBC databases, HTTP, Prometheus, generic JMX, SLA) —
   an empty list means there is no application to pick, **not** that there is
   nothing left to do: Prometheus and generic JMX still need the counter step
   below, more than any other type. The list can be large (e.g. every process on a
   host) — filter it to what the user actually wants.
2. **`preview_monitor_counters(monitorId, applications?, patterns?)`** → the
   counter tree flattened to **slash-joined paths**
   (`"Network Interfaces / eno1 / Sent Bytes"`) each with `selectedByDefault`
   (the type's default) and `selected` (what this monitor collects **now** — use
   it to read back the current selection, or to make an incremental change:
   re-send the still-`selected` paths plus/minus your edit). Pass the application
   names you chose to include their counters (required to see e.g. a Tomcat
   webapp's or a specific NIC's counters). **Narrow a large tree with
   `patterns`** — `"*"` wildcards matched case-insensitively against the whole
   path (`"prometheus_tsdb_*"`, `"*eno1*"`). The list is capped at 200 paths:
   past that `truncated` is true and `groups` returns one ready-to-use pattern
   per counter family with its path count, so a Prometheus endpoint exposing
   hundreds of metrics answers with a handful of families to walk instead of a
   wall of names. The paths the monitor **currently collects are never dropped by
   the cap**, so the `selected` flags always describe the whole selection and an
   incremental edit stays safe on a truncated list.
3. **`update_monitor_counters(monitorId, applications?, counterPaths?)`** →
   replace the collected counters with the chosen paths. **A folder path keeps
   its whole subtree** (including thresholds), so `"Network Interfaces / eno1"`
   keeps every eno1 counter. `counterPaths` accepts the same `"*"` patterns, so
   `["prometheus_tsdb_*", "go_memstats_*"]` selects two whole families without
   echoing every path — and it **fails instead of emptying the monitor** when
   nothing matches. Omit `counterPaths` to reset to the type's default selection.
   DESTRUCTIVE — it replaces the current selection.

   **A counter that stays selected keeps what the agent cannot rediscover**: its
   thresholds, the folder the user filed it under, its renamed name and edited
   description. The counter tree is the user's to organize — discovery only seeds
   it — so re-running this on a monitor someone has curated in the UI is safe.
   Counters you leave out are still removed, with their thresholds: send the
   still-`selected` paths plus/minus your edit, not just your edit.

Select by **path** (names), not by id — the backend regenerates counter ids on
every discovery. Because selection is a flat named list, choose by *intent*
("keep CPU %, memory %, and all eno1 counters") and let a pattern carry that
intent rather than trying to walk a tree path by path. Note process names are not
unique (many `java`) — selecting one by name includes all of them.

## Organize the counters into folders

A selection is a flat list, and on Prometheus or generic JMX that means dozens of
counters side by side — readable by nobody. **`organize_monitor_counters(monitorId,
groups)`** files them into named folders, and this is where you add value the
platform cannot: name a folder for *what it tells the reader* ("CPU & Memory",
"TSDB ingestion", "Scrape health"), not for the metric prefix it happens to share.

Each `groups` entry reads `"<folder name> => <counter path or pattern>"`, the
pattern taking the same `"*"` wildcards. Repeat a folder name to give it several
patterns:

```
["CPU & Memory => process_*",
 "CPU & Memory => go_memstats_*",
 "GC => go_gc_*",
 "Scrape health => prometheus_target_*"]
```

Folders fill in order and a counter joins the first one matching it, so write the
specific patterns before the broad ones — a later catch-all left with nothing is
skipped, while a pattern matching **none** of the collected counters fails the
call before anything moves. A counter no folder matches stays where it is, and a
moved counter keeps its thresholds. A folder name the tree already uses is reused
rather than duplicated.

**A pattern matches a counter by its own name as well as by its full path**, here and
in `set_monitor_thresholds`. Both read the persisted tree, where a counter already
filed into a folder carries that folder
(`"WAL & Disk I/O / prometheus_tsdb_wal_writes_failed_total"`), while discovery only
ever returns the bare `"prometheus_tsdb_wal_writes_failed_total"` — so patterns
written off a preview keep working on a monitor that has since been organized, and
re-running a layout is idempotent. Counter names are not unique (a host running
several `java` processes), and a bare name matches every one of them, exactly as it
would against the flat discovered tree.

It only rearranges what the monitor already collects — it never changes the
selection and **never contacts the agent**, so it is fast. It is still
DESTRUCTIVE on the tree's *shape*: a layout the user arranged in the UI is
replaced by yours, so prefer it right after creating a monitor rather than on one
somebody has already organized.

## Add thresholds

A counter without a threshold is a curve somebody has to eyeball. With one, a run
raises a **threshold alarm** in the report. Most types seed sensible defaults at
discovery (Linux, Postgres, Oracle, MySQL, MongoDB, IIS, Windows, httpd, lighttpd,
SQL Server) — the ones that cannot are exactly Prometheus and generic JMX, because
no table of limits can exist for arbitrary metric names. That is where knowing
what a metric *means* pays off.

**`set_monitor_thresholds(monitorId, thresholds)`** takes a JSON array:

```json
[{"counters": ["process_open_fds"],
  "name": "Descriptors near the limit",
  "severity": "CRITICAL",
  "above": 80,
  "occurrences": 3}]
```

- `counters` — paths or `"*"` patterns, matched by counter name as well as by full
  path (see grouping above), so a folder layout never gets in the way.
- `name` — what the alarm says when it fires, **100 characters at most** (longer
  is rejected). Write it for a reader who did not pick the counters, but keep it a
  label rather than a sentence: it names the alarm in the monitor tree and is
  printed on every report chart carrying it.
- `severity` — `WARNING` or `CRITICAL` (`PASSED` exists for a range that means
  healthy).
- the range, **exactly one** of `above` (at or above), `below` (at or below) or
  `between: [min, max]`. **The range is the state to be warned about**, not the
  healthy one: the alarm holds while the value sits inside it. `above: 80` on a
  percentage means "warn from 80% up".
- `occurrences` (default 1) consecutive collections the value must stay in range,
  or `seconds` instead — not both. A single occurrence alarms on the first
  collected value in range, which is noisy on a spiky counter.

A threshold already carrying the same name on a counter is **replaced**, so
re-sending a set tunes it instead of piling up duplicates. It fails without
writing anything when a pattern matches none of the collected counters. Like
grouping, it changes neither the selection nor the tree's shape, and never
contacts the agent.

Confirm the limits with the user before setting them — a wrong limit is noise in
every future report, and it overwrites a same-named threshold they may have tuned
themselves.

## Read back what a monitor collects

`get_monitor_counters(monitorId, patterns?)` is the **read side** of the three
tools above, which all write into the persisted tree. It returns each collected
counter under its curated path, with its name, description, unit and thresholds —
and each threshold in the shape `set_monitor_thresholds` accepts, so reading one,
changing a number and re-sending it is the way to tune alerting. Never contacts the
agent, so it is fast; capped at 200 counters with `truncated` saying so.

**One round-trip caveat on `between`.** Several types seed a warning range whose
upper bound is *exclusive* — `[4, 8)` for "high", with a separate critical at `8`,
so a value of exactly 8 raises the critical alone. The write shape has no way to
say "exclusive", so re-sending that range as `between: [4, 8]` makes it inclusive
and both alarms then fire at 8. Read it, but leave it alone unless widening it is
what you want.

**Do not reach for `preview_monitor_counters` for this.** The two answer different
questions: preview re-discovers from the agent to show what the target *could*
expose, so it reports the type's default thresholds rather than the user's and
carries no description or unit at all. An empty `get_monitor_counters` is the
plainest possible statement that a monitor is a silent no-op.

## Check a monitor

`check_monitor_connection(monitorId)` asks the agent to open the connection and
authenticate. It is **synchronous** (it waits for the agent, up to ~60s) and
returns `{reachable, message}`: `reachable=true` when the agent reached the
target with valid credentials, otherwise `reachable=false` with the agent's
error in `message`. Use it to troubleshoot a monitor that is not collecting data
(wrong host/port, bad credentials, or the agent cannot reach the target's
network).

## Manage

- **`list_monitors_by_project(projectId)`** / **`get_monitor(monitorId)`** —
  inspect (secret-free: type, agent, target, enabled, tags).
- **`update_monitor(monitorId, …)`** — rename, enable/disable, retag, or
  re-point to another (reachable, UP) agent. Does not touch the counter
  selection — use `update_monitor_counters` for that.
- **`delete_monitor(monitorId)`** — DESTRUCTIVE; confirm first. Past runs'
  recorded values are unaffected.

## Typical workflows

- **Simple OS/DB monitor**: pick a UP agent that reaches the target →
  `create_linux_monitor` / `create_postgres_monitor` → (optionally)
  `check_monitor_connection` → done (defaults are fine).
- **Tomcat / app-server monitor**: `create_tomcat_monitor` (base) →
  `list_monitor_applications` (the webapps) → `preview_monitor_counters` with the
  chosen webapp(s) → `update_monitor_counters` to keep them. Without this the
  monitor watches only the JVM/global counters.
- **Refine a Linux monitor**: create → `list_monitor_applications` (disks / NICs
  / processes) → `preview_monitor_counters(applications=[…])` →
  `update_monitor_counters` with the counter paths to keep.
