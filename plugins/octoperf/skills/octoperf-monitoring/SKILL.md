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
- The values and alarms show up in the bench **report** (live and final): add
  monitoring report items in the UI, or read them over MCP with
  `get_report_textual_monitors` (textual monitor values) and
  `get_report_threshold_alarms` (fired alarms); numeric monitor curves come
  through the generic `get_report_*_values` metric tools.

So the useful loop is: create + enable a monitor → run the scenario →
read its counters/alarms in the report.

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

Creation auto-selects reasonable **default** counters — enough for OS/DB/SLA
monitors. For others you must refine the selection, especially **application
selection which is mandatory for some types**: a Tomcat monitor collects nothing
useful until you pick the **webapp(s)** to watch. Applications are the disks /
network interfaces / processes of a Linux host, or the webapps of a Tomcat
server, …

Configure any existing monitor (this even works post-creation, which the web UI
does not offer) with three tools:

1. **`list_monitor_applications(monitorId)`** → the applications the monitor can
   collect counters for (`{type, name}`): a Tomcat server's webapps, a Linux
   host's disks / NICs / processes, a Windows or IIS host's CPUs / disks / NICs /
   processes, a SQL Server's databases / locks / plan caches. Empty for the types
   with no application step (JDBC databases, HTTP, Prometheus, generic JMX, SLA).
   The list can be large (e.g. every process on a host) — filter it to what the
   user actually wants.
2. **`preview_monitor_counters(monitorId, applications?)`** → the full counter
   tree flattened to **slash-joined paths**
   (`"Network Interfaces / eno1 / Sent Bytes"`) each with `selectedByDefault`
   (the type's default) and `selected` (what this monitor collects **now** — use
   it to read back the current selection, or to make an incremental change:
   re-send the still-`selected` paths plus/minus your edit). Pass the application
   names you chose to include their counters (required to see e.g. a Tomcat
   webapp's or a specific NIC's counters).
3. **`update_monitor_counters(monitorId, applications?, counterPaths?)`** →
   replace the collected counters with the chosen paths. **A folder path keeps
   its whole subtree** (including thresholds), so `"Network Interfaces / eno1"`
   keeps every eno1 counter. Omit `counterPaths` to reset to the type's default
   selection. DESTRUCTIVE — it replaces the current selection.

Select by **path** (names), not by id — the backend regenerates counter ids on
every discovery. Because selection is a flat named list, choose by *intent*
("keep CPU %, memory %, and all eno1 counters") rather than trying to walk a
tree. Note process names are not unique (many `java`) — selecting one by name
includes all of them.

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
