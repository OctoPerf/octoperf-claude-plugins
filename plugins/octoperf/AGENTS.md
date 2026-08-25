# OctoPerf MCP — Agent guide

This file tells any CLI agent (Codex CLI, Aider, Cursor, Continue.dev,
Claude Code, …) how to drive the **OctoPerf MCP server** in this project.
Drop it at your project root and your agent will pick it up automatically.

OctoPerf is a load-testing platform. The MCP server exposes its REST API
as MCP tools so an agent can: import Virtual Users (HAR / JMX / Postman /
Playwright / WebDriver / URLs), edit them (variables, HTTP
servers, correlation rules), validate them functionally, run load
scenarios, and read back metrics — all without leaving the chat.

## Connect once

The server speaks OAuth 2.1 + PKCE + DCR. There is **no API key**; every
tool call carries a short-lived JWT minted from your user identity.

**Claude Code — quickest path: the official plugin.** It bundles the MCP
server registration, this `AGENTS.md`, and the sixteen workflow skills in
one install:

```text
/plugin marketplace add OctoPerf/octoperf-claude-plugins
/plugin install octoperf@octoperf
```

For everything else, register the MCP endpoint manually:

| Client                 | Setup                                                                                                  |
|------------------------|--------------------------------------------------------------------------------------------------------|
| Claude.ai              | Settings → Connectors → Add Custom Connector → `https://api.octoperf.com/mcp`                          |
| Claude Code (manual)   | `claude mcp add octoperf --transport http https://api.octoperf.com/mcp`                                |
| Claude Desktop         | Add the MCP server URL `https://api.octoperf.com/mcp` in `claude_desktop_config.json`                  |
| Cursor / Continue.dev  | Add the MCP server URL `https://api.octoperf.com/mcp` in the MCP config; browser opens for OAuth login |
| MCP Inspector          | `npx @modelcontextprotocol/inspector https://api.octoperf.com/mcp` (Streamable HTTP)                   |

For self-hosted Enterprise, swap the host for your instance (e.g.
`https://octoperf.example.com/mcp`).

Revoke at any time from **Account → Connected applications** on OctoPerf UI.

### Claude.ai web: allow the OctoPerf host for presigned URLs

Tools that move file bytes — `upload_project_file`,
`download_project_file`, `download_bench_result_file`, the PDF leg of
`export_bench_report_pdf`, every `import_*_virtual_user` /
`upload_jmx_virtual_user` that consumes a file (HAR, JMX, Postman,
Playwright), and any Playwright trace / HAR / JTL pull — return a
short-lived **presigned URL** pointing at the OctoPerf REST API (same
origin as the MCP server: `https://api.octoperf.com`, or your
self-hosted host). On Claude.ai web, the OAuth trust granted
when connecting the MCP server does **not** authorize Claude's sandbox
to reach that host: a workspace admin must add `octoperf.com` (or
`*.octoperf.com` to also cover an on-prem deployment) under
**Claude.ai → Organization Settings (Admin only) → Capabilities →
Domain allowlist → Additional allowed domains**. The dropdown can
stay on **None** (no preset list needed); the manual entry alone is
enough. The setting is org-scoped: members inherit it, no per-user
action required.

This limitation only applies to the Claude.ai web (and Desktop)
clients. Claude Code CLI, Cursor, Continue.dev and MCP Inspector
execute the fetch themselves and are not gated by this allowlist.

## Tool catalogue

All tools are also exposed as same-named MCP prompts (slash commands
`/mcp__octoperf__<tool>` in clients that support them).

### Discovery

**Start every session with `get_current_user`**, then open your first reply with
one line naming the account and whether it administers the platform — "You are
connected to OctoPerf as alice@corp.com (platform administrator)" — before
answering what was asked. Once per session, in the user's language. Several
OctoPerf servers (local, preprod, SaaS) can be registered at once and nothing
else distinguishes them — and the same account is `ADMIN` on one workspace and
`VIEWER` on the next, so this is also how you know what you may do before you
attempt it.

| Tool                                  | Purpose                                                |
|---------------------------------------|--------------------------------------------------------|
| `get_current_user`                    | Who this session is connected as: account id, `username` (the e-mail the member tools address it by), `platformAdmin`, and every workspace with the role held there |
| `list_workspaces`                     | Workspaces the user is a member of, each with the `role` the connected account holds on it |
| `list_projects_by_workspace`          | DESIGN projects in a workspace                         |
| `list_scenarios_by_project`           | STANDARD-mode scenarios of a project                   |
| `list_virtual_users`                  | Virtual Users (VUs) of a project                       |
| `list_docker_providers_by_workspace`  | Load-generator providers of the workspace (id, type, driver, locations)  |
| `list_public_docker_providers`        | Shared OctoPerf Cloud providers                        |

### Workspace & project

A workspace scopes **members, load-generator providers and their agents, and
notifications**; a project scopes **virtual users, scenarios, monitors and
reports**. **Read `octoperf://skills/workspace`** before adding someone to a
workspace or changing what they can do.

| Tool                  | Purpose                                                |
|-----------------------|--------------------------------------------------------|
| `create_workspace`    | New workspace — its own members and load-generator providers. **Not deletable from here, and not self-service**: removal goes through the platform administrator (on-premise) or an OctoPerf support request (SaaS). Create one only for a separate set of members or load generators; to merely group work, `create_project` in an existing workspace |
| `update_workspace`    | Rename / re-describe a workspace (Admin role required) — the name is what every member sees in the switcher; membership, projects, providers and notifications untouched |
| `list_workspace_members` | Who has access to a workspace and with which role — members are named by the e-mail their account is registered under |
| `get_current_user`    | Which account the session is authenticated as, and its role on each workspace — the way to know whether a member change is about to hit yourself |
| `add_workspace_member` | Give an **existing** OctoPerf account access — **not an invitation**, an unknown address is refused and nothing is e-mailed; arrives as VIEWER unless a role is passed |
| `set_workspace_member_role` | **Destructive** — replace a member's access with ADMIN / TESTER / TEST_EXECUTOR / VIEWER; a workspace keeps at least one admin, and a user stays admin of at least one workspace |
| `remove_workspace_member` | **Destructive** — take a member's access away (account and other workspaces untouched); the last admin cannot be removed |
| `create_project`      | New DESIGN project in a workspace                      |
| `update_project`      | Rename / re-describe / re-tag an existing project      |
| `delete_project`      | **Destructive** — the widest delete here: the project goes, and with it every Virtual User, scenario, variable, correlation rule, HTTP server, file, monitor, SLA profile, scheduled job, and every past bench result and report of that project. No restore. Guarded: the project's exact current name must be passed as `confirmProjectName`, so read it from `list_projects_by_workspace` and have the user confirm that name — two projects of one workspace often look alike |

Names are **80 characters** at most and descriptions **2000** — the bounds the web
forms enforce. Past them the entity's own edit form is invalid the moment it opens,
so the write is refused rather than truncated.

### On-premise providers & agents

Run load generators on **your own machines**. A provider *supplies* load
generators **through agents** you install — each agent starts load generators on
demand; the provider itself runs none. It is a localized load **capacity** (VUs
per location); an agent is one Docker container per machine — **one agent per
machine** (a single agent runs many load generators; stacking agents does not
add capacity). On-premise is chosen mostly for **reach**: agent machines must be
able to hit the app under test (internal/private targets the Cloud can't) and
must **not** be the machine hosting that app. Read
`octoperf://skills/onpremise-agent` before using these.

| Tool                              | Purpose                                                                                     |
|-----------------------------------|---------------------------------------------------------------------------------------------|
| `create_onpremise_docker_provider`| New PRIVATE on-premise provider with one or more named locations (name + lat/long)          |
| `add_provider_location`           | Add a location to an existing provider                                                       |
| `update_provider_location`        | Rename / move a location — **renaming orphans its agents** (returns `orphanedAgents`)        |
| `remove_provider_location`        | **Destructive** — drop a location; its agents are orphaned                                   |
| `get_onpremise_agent_command`     | The `docker run` command to install an agent on a machine for a provider location           |
| `list_provider_agents`            | Agents of a provider (state, host) + duplicate-on-host detection + per-agent uninstall cmd   |
| `get_provider_usage`              | Current capacity & load per location (agents + memory in use) + per-machine memory config    |

### Virtual User — read & inspect

| Tool                                       | Purpose                                                                 |
|--------------------------------------------|-------------------------------------------------------------------------|
| `get_virtual_user`                         | Full action tree (heavy — polymorphic children, headers, postData, …)   |
| `describe_virtual_user`                    | Compact `VirtualUserListing` (id, name, tags, timestamps, `url` deep-link) — chain after a presigned import to surface the UI link |
| `sanity_check_virtual_user`                | Static design checks — non-destructive, no credit cost                  |
| `get_virtual_user_validation_index`        | Per-action validation failure summary                                   |
| `get_validation_failure_detail`            | The four HTTP entities of one failing action (sent/recv request/response) |
| `fetch_validation_http_body`               | Pull a single HTTP body recorded during validation                      |
| `get_virtual_user_validation`              | Latest validation run state on a VU                                     |

### Virtual User — import

Imports split in two shapes:

- **File-backed imports** (`import_har_virtual_user`,
  `import_postman_virtual_user`, `import_playwright_virtual_user`,
  `upload_jmx_virtual_user`) mint a short-lived presigned upload URL.
  The client POSTs the file directly to the OctoPerf REST host (bypassing
  the MCP server for the bytes); the REST response is a raw
  `VirtualUser` (HAR / Postman / Playwright) or a Virtual User Action
  array (JMX), neither of which carries a `url` deep-link. **Always
  chain into `describe_virtual_user`** with each returned `id` to obtain
  the compact listing and the link to the Virtual User page.
- **In-process imports** (`import_urls_virtual_user`,
  `import_webdriver_virtual_user`) accept their input as a tool
  parameter (URL list) and return a `VirtualUserListing` directly —
  no follow-up call needed.

| Tool                              | Shape    | Source                                                                                   |
|-----------------------------------|----------|------------------------------------------------------------------------------------------|
| `import_har_virtual_user`         | upload   | HAR capture                                                                              |
| `import_postman_virtual_user`     | upload   | Postman v2.1 JSON                                                                        |
| `import_playwright_virtual_user`  | upload   | Single `.spec.ts` file; add helpers / `package.json` afterwards via `patch_virtual_user` |
| `upload_jmx_virtual_user`         | upload   | JMeter `.jmx` (creates one or more VUs)                                                  |
| `analyze_neoload_project`         | upload   | NeoLoad archive → portability assessment, writes nothing. Run it **before** importing    |
| `import_neoload_virtual_user`     | upload   | NeoLoad project folder or its `config.zip` (VUs + servers + variables + scenarios)       |
| `analyze_loadrunner_project`      | upload   | Zipped VuGen script folder → portability assessment, writes nothing. Ship `data/` with it |
| `import_loadrunner_virtual_user`  | upload   | Zipped VuGen script folder + its `.lrs` (VUs + servers + variables + `.dat` + scenarios)  |
| `import_urls_virtual_user`        | in-proc  | URL list → HTTP VU                                                                       |
| `import_webdriver_virtual_user`   | in-proc  | URL list → browser VU                                                                    |
| `update_virtual_user`             | —        | Edit metadata (name/description/tags); tree untouched                                    |
| `backup_virtual_user`             | —        | Duplicate a VU + tag it `backup` before a risky change (no VU versioning in OctoPerf)    |
| `patch_virtual_user`              | —        | Edit the action tree via RFC 6902 JSON Patch                                             |
| `delete_virtual_user`             | —        | **Destructive** — drops the tree                                                         |

**Read `octoperf://skills/neoload-migration` before importing a NeoLoad
project, and `octoperf://skills/loadrunner-migration` before importing a
LoadRunner script.** The shape of what those two answer with is
`octoperf://schema/neoload-assessment` and
`octoperf://schema/loadrunner-assessment` — the upload bypasses the MCP server
for both the bytes and the response, so neither appears in a tool's
`outputSchema` and the schema resource is where the vocabulary lives. The import is best-effort by design: it converts what has an
OctoPerf equivalent and hands back everything else in `unconverted`,
each entry carrying a `kind`, the `marker` planted in the tree where there
is one, and — for most kinds — a `facts` object holding the values to act
on. No entry carries prose: the sentence saying what happened, and what to
do about it, is the skill's section for that `kind`. What to *do*
about each kind — which of the three container modes rebuild, why a
streamed request cannot, where an SLA profile has to be attached, which
counters to select on a recreated monitor — lives in the skill and
nowhere else. Without it you have the findings and not the playbook.

#### When the upload is too big

Presigned uploads stream the bytes directly to the OctoPerf REST host
in a single `multipart/form-data` POST — they are bounded only by the
backend's own request-size limit, not by the MCP tool-call token budget.
If a client environment still can't perform the POST (no network access,
sandbox restrictions, file too big for the backend, …), don't fail the
task: tell the user to upload the file manually through the OctoPerf
web UI. Discover the project URL via `list_projects_by_workspace` and
point them at it — the UI's import / files pages do the same upload
server-side.

### Variables (parameterization)

| Tool                       | Purpose                                            |
|----------------------------|----------------------------------------------------|
| `list_variables`           | List all variables (with usage)                    |
| `create_constant_variable` | Same value on every read                           |
| `create_csv_variable`      | Row-by-row from an uploaded CSV                    |
| `create_counter_variable`  | Auto-incremented integer                           |
| `create_random_variable`   | Random value within constraints                    |
| `create_secret_variable`   | Plaintext stored encrypted, redacted in UI/logs    |
| `delete_variable`          | **Destructive**                                    |

### HTTP servers

| Tool                          | Purpose                                                |
|-------------------------------|--------------------------------------------------------|
| `list_http_servers_by_project`| baseUrl + timeouts + IP-spoofing flag                  |
| `list_http_server_usages`     | Which VUs reference a given server                     |
| `update_http_server`          | Edit baseUrl / timeouts / IP spoofing                  |
| `delete_http_server`          | **Destructive**                                        |
| `delete_unused_http_servers`  | **Destructive** — bulk cleanup                         |

### Project files (CSV, fixtures…)

| Tool                       | Purpose                                                |
|----------------------------|--------------------------------------------------------|
| `list_project_files`       | Files attached to a project                            |
| `read_project_file_lines`  | Slice of a file by line range                          |
| `upload_project_file`      | Add / overwrite a file (e.g. CSV used by a variable)   |
| `download_project_file`    | Presigned GET URL (single-use, ~5 min) to pull a file  |
| `delete_project_file`      | **Destructive**                                        |

### Correlation (dynamic-value extraction)

| Tool                                   | Purpose                                                                |
|----------------------------------------|------------------------------------------------------------------------|
| `list_correlation_frameworks`          | Built-in presets (SAML, OAuth, .NET, Java, Token, AzureAD, …)          |
| `add_correlation_framework_to_project` | Apply a preset to a VU                                                 |
| `list_correlation_rules`               | Project-defined regex rules                                            |
| `create_correlation_rule`              | New regex rule                                                         |
| `delete_correlation_rule`              | **Destructive**                                                        |
| `apply_correlations_to_virtual_user`   | Async re-walk the VU and rewrite extractors/usages — returns a task id |

### Design SLA profiles

Pass/fail bands on a Virtual User's own metrics (response time, errors, Apdex, …). A profile is inert until it is attached to a VU; on every run the attached profiles become an SLA monitor connection whose breaches surface through `get_report_threshold_alarms`. Attached, they are one of the three lines the scenario page shows under **Pass Criteria**. **Unrelated to `create_sla_monitor`**, which is a project-scoped monitor with a fixed counter set. **Read `octoperf://skills/sla` before creating or editing one.** Create-then-patch: `create_sla_profile` pre-fills the same sourced defaults as the web UI, then retune with `patch_sla_profile`.

| Tool                                    | Purpose                                                                                          |
|-----------------------------------------|--------------------------------------------------------------------------------------------------|
| `list_sla_profiles`                     | Profiles of a project — metrics constrained, threshold count, UI deep-link (bands omitted)        |
| `get_sla_profile`                       | Full profile: every metric group and its graded bands                                            |
| `list_sla_threshold_defaults`           | Suggested bands per metric, without creating anything (pass `metrics` — the full table is large) |
| `create_sla_profile`                    | New profile pre-filled with the default bands of the named metrics                                |
| `patch_sla_profile`                     | **Destructive** — RFC 6902 patch: retune a band, add/remove a metric group                        |
| `delete_sla_profile`                    | **Destructive** — detach it everywhere first                                                      |
| `attach_sla_profile_to_virtual_user`    | Bind the profile to a VU (idempotent) — nothing fires before this                                  |
| `detach_sla_profile_from_virtual_user`  | **Destructive** — unbind; the profile stays available to other VUs                                 |

### Runtime — scenarios

A scenario says what runs and under which load. What says the run **passed** is not on
it: the scenario page shows that as **Pass Criteria**, computed from the SLA profiles the
Virtual Users it plays carry and from the project's enabled monitors. Worth naming once
while composing a test — never worth forcing. `octoperf://skills/scenario-composition`
covers the step and how to read the criteria a test already has.

| Tool                             | Purpose                                                                                                                                         |
|----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------|
| `create_scenario_ramp_up`        | Scenario with ramp-up + hold load (VUs + provider + one-or-more regions round-robin)                                                            |
| `create_scenario_ramp_up_down`   | Scenario with ramp-up + plateau + ramp-down (for soak tests with controlled wind-down)                                                          |
| `create_scenario_stairs`         | Scenario with ascending stairs (N discrete steps to target users — for capacity-finding tests)                                                  |
| `get_scenario`                   | Full Scenario (metadata + userProfiles with load shapes + engine settings)                                                                      |
| `update_scenario`                | Edit metadata (name/description/tags); userProfiles untouched                                                                                   |
| `patch_scenario`                 | Edit the full scenario via RFC 6902 JSON Patch                                                                                                  |
| `update_user_profile` | **Destructive** — re-point one user profile: its Virtual User, provider or location. Swapping to a VU of another type swaps the engine block and resets that profile's engine settings |
| `delete_scenario`                | **Destructive** — drops the scenario configuration                                                                                              |
| `schedule_scenario_once`         | **Destructive (credits on fire)** — schedule a one-shot scenario run at a given ISO-8601 datetime                                               |
| `schedule_scenario_cron`         | **Destructive (credits on each fire)** — schedule recurring runs from a Unix 5-field cron expression evaluated in UTC (NOT Quartz / no seconds) |
| `list_scheduled_jobs_by_project` | List all scheduled jobs of a project — id, scenarioId, name, trigger description, enabled flag, nextRun                                         |
| `enable_scheduled_job`           | Re-arm a paused job (**re-arms a destructive action**)                                                                                          |
| `disable_scheduled_job`          | Pause a job without deleting — re-enable later via `enable_scheduled_job`                                                                       |
| `delete_scheduled_job`           | **Destructive** — drop the schedule entry (past runs stay in history). Prefer `disable_scheduled_job` for a reversible pause                    |

**Load-shape vocabulary** — when the user describes a load using OctoPerf's UI terms, map to:

| UI term                     | Shape                             | Tool                                            |
|-----------------------------|-----------------------------------|-------------------------------------------------|
| **Smooth**                  | ramp-up → plateau → ramp-down     | `create_scenario_ramp_up_down`                  |
| **Sustained**               | ramp-up → plateau (no ramp-down)  | `create_scenario_ramp_up`                       |
| **Stress**                  | N discrete steps up to peak       | `create_scenario_stairs`                        |
| **Custom** (point-by-point) | arbitrary inflection points       | `patch_scenario` against `userProfiles[i].load` |

WebDriver-based virtual users are **capped at 1 concurrent user per UserProfile** in the engine (real-browser CPU contention). Don't propose a ramp to N users on a WEB_DRIVER VU — split across N UserProfiles instead.
Prefer Playwright-based virtual users over WebDriver-based virtual users.

### Runtime — validate & run

| Tool                          | Purpose                                                                                                                                                                                                                                                                                                                                               |
|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `validate_virtual_user`       | Starts a functional check (1 user, N iterations). Returns `(benchResultId, state)`. No credit cost.                                                                                                                                                                                                                                                   |
| `get_scenario_matching_plans` | **PRE-FLIGHT** — list the account's subscriptions / plans that can host the scenario, with per-plan caps (concurrency, real-browser, duration) and the actual VU count each plan would allocate. Call before `run_scenario` to avoid burning credits on an under-sized plan.                                                                          |
| `list_active_subscriptions`   | Account-level inspection: every usable subscription on the account (status `active` / `trialing` / `cancel_at_period_end`) with full plan caps (maxConcurrentUsers, maxRealBrowserUsers, maxProfilesPerScenario, maxTestDurationSec, remainingTests). Use when `get_scenario_matching_plans` returned empty to explain which cap blocks the scenario. |
| `run_scenario`                | **Destructive (credits)** — starts a real load test. Returns the bench result id.                                                                                                                                                                                                                                                                     |
| `get_bench_status`            | Progress `[0.0, 1.0]` for a running test                                                                                                                                                                                                                                                                                                              |

### Runtime — bench results

| Tool                                  | Purpose                                                                                                                                                                                               |
|---------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `get_bench_result`                    | Full bench result entity (canonical state lookup — CREATED/PENDING/…/RUNNING/FINISHED/ABORTED/ERROR)                                                                                                  |
| `stop_bench_result`                   | **Destructive** — aborts a running bench (terminal state = ABORTED)                                                                                                                                   |
| `list_bench_docker_logs`              | Docker container logs of the bench launch (same panel the web UI streams) — diagnose ERROR / stuck PREPARING / INITIALIZING runs                                                                      |
| `list_bench_result_files`             | JMeter logs + JTL / HAR / screenshots / attachments stored against a benchResultId (also covers validation runs)                                                                                      |
| `read_bench_result_file_lines`        | Read a contiguous range of lines from one of those files (gzip-transparent; binary files return garbage)                                                                                              |
| `download_bench_result_file`          | Mint a presigned GET URL (single-use, ~5 min) to pull one bench-result file directly — Playwright `trace.zip`, screenshots (`.png`), HAR archives, any binary artefact the line reader can't surface. |

**Bench-result state machine** — drives when log files / report metrics become available:

```
CREATED → PENDING → SCALING → PREPARING → INITIALIZING ─┬─→ ERROR     (terminal — provisioning failed)
                                                         └─→ RUNNING → ┬─→ FINISHED (planned end)
                                                                       └─→ ABORTED (external stop)
```

- Log files (`list_bench_result_files`) and the default report's metrics are populated only once the state is terminal (FINISHED / ABORTED / ERROR). Polling earlier returns an empty set.
- Validation logs are erased **after 7 days** or when the user leaves the design screen — don't promise to read old validation logs.
- `get_bench_status` returns progress `[0.0, 1.0]` while RUNNING; `get_bench_result` returns the canonical state at any time.
- Docker launch logs (`list_bench_docker_logs`) are the right read when the run is **stuck in PREPARING / INITIALIZING** or ended in **state=ERROR** before JMeter ever started — they show provider quota errors, image pull failures, missing project files, agent boot crashes. Available from the moment the bench is scheduled; no terminal-state wait.

### Runtime — monitoring

Monitors watch an external target (OS / DB / web-server / JMX / Prometheus / New Relic / SLA) during a run, through an **UP agent that can reach it**. Monitors are project-scoped, agents workspace-scoped. **Read `octoperf://skills/monitoring` before creating or configuring one.** Create-then-configure: a `create_*` tool makes the monitor with sensible default counters; refine with the selection tools (mandatory for e.g. a Tomcat webapp). Create / check / discovery contact the agent, so they take a few seconds.

| Tool | Purpose |
|------|---------|
| `list_monitors_by_project` / `get_monitor` | List / inspect monitors (secret-free: type, agent, target, enabled, tags) |
| `list_workspace_agents` | Workspace agents with `state` — pick an `UP` one that reaches the target (project↔workspace gap: pass the project's workspace) |
| `create_linux_monitor` | Linux host over SSH (password or private key) |
| `create_postgres_monitor` / `create_mysql_monitor` / `create_oracle_monitor` | Database over JDBC (url + credentials). **Not SQL Server** — see the JMX row |
| `create_mongodb_monitor` | MongoDB (host / port / database, optional credentials) |
| `create_nginx_monitor` / `create_apache_httpd_monitor` / `create_lighttpd_monitor` / `create_prometheus_monitor` | Web-server status page / Prometheus `/metrics` url (optional credentials) |
| `create_generic_jmx_monitor` / `create_tomcat_monitor` / `create_jmeter_monitor` / `create_iis_monitor` / `create_windows_monitor` / `create_sqlserver_monitor` | JMX endpoint (service url, or host + JMX-RMI port). IIS / Windows / **SQL Server** read Windows perf counters through the OctoPerf Windows/JMX bridge on the target (typically port 1099) — SQL Server is *not* JDBC |
| `create_newrelic_monitor` | New Relic APM (REST API url + api key) |
| `create_sla_monitor` | Self-contained monitor of the test's own per-request SLA metrics (independent of Design SLA profiles) |
| `check_monitor_connection` | Synchronous (waits for the agent, up to ~60s) — asks the agent to open the connection; returns `{reachable, message}` (reachable=true ⇒ reached + creds OK, else the error) |
| `list_monitor_applications` | Applications the monitor can collect for (Tomcat webapps, Linux/Windows/IIS disks / NICs / processes, SQL Server databases / locks / caches); empty for JDBC DB / HTTP / Prometheus / generic JMX / SLA |
| `preview_monitor_counters` | Counter tree flattened to slash-joined paths + `selectedByDefault` (type default) + `selected` (this monitor's current selection — read back / incremental edit); narrow with `"*"` patterns, capped at 200 paths then `groups` summarizes the families (pass application names to include their counters) |
| `update_monitor_counters` | **Destructive** — replace collected counters by path or `"*"` pattern (a folder keeps its whole subtree, a pattern keeps a whole family); a counter that stays selected keeps its thresholds, folder, renamed name and edited description; fails rather than emptying the monitor when nothing matches; omit paths to reset to defaults |
| `organize_monitor_counters` | File the collected counters into named folders — `groups=["<folder> => <path or pattern>", …]`, first match wins, moved counters keep their thresholds; rearranges only, never re-selects and never contacts the agent |
| `set_monitor_thresholds` | Hang alert thresholds on collected counters — JSON array of `{counters, name (≤ 100 chars), severity, one of above/below/between, occurrences\|seconds}`; the range is the state to warn about, same-named thresholds are replaced; never re-selects, never reshapes, never contacts the agent |
| `get_monitor_counters` | Read side of the three above — what the monitor **actually collects**, each counter under its curated path with name, description, unit and thresholds (in the shape `set_monitor_thresholds` takes, so re-sendable to tune one); narrow with `"*"` patterns, capped at 200; never contacts the agent, and empty ⇒ silent no-op |
| `update_monitor` | **Destructive** — rename / enable-disable / retag / re-point to another agent (does not touch counters) |
| `delete_monitor` | **Destructive** — remove the monitor (past runs' recorded values unaffected) |

### Analysis — bench reports

| Tool                                   | Purpose                                                                                                                                                                                                                                                                         |
|----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `list_bench_reports_by_project`        | Every report of a project (metadata + url) — start here to find reportIds                                                                                                                                                                                                       |
| `get_bench_report`                     | Full report — polymorphic `items` (charts / tables / top / summary / …) + per-benchResult configs                                                                                                                                                                               |
| `update_bench_report`                  | Partial metadata edit — name / description / tags (items + configs untouched)                                                                                                                                                                                                   |
| `patch_bench_report`                   | RFC 6902 JSON Patch over the full BenchReport entity (use `octoperf://schema/bench-report`). Metric-bearing items are validated against their allow-select descriptor as a pre-flight — an invalid metric `type`/`id`/filter is rejected with the violations.                    |
| `delete_bench_report`                  | **Destructive** — drops the report, leaves benchResults intact                                                                                                                                                                                                                  |
| `create_trend_report_by_tags`          | TREND report anchored on one benchResult, other points picked by tag intersection on bench results                                                                                                                                                                              |
| `create_trend_report_by_name`          | TREND report anchored on one benchResult, other points picked by scenario-name match (EQUALS / CONTAINS / STARTS_WITH / ENDS_WITH, case-sensitive or not)                                                                                                                       |
| `create_trend_report_by_creation_date` | TREND report anchored on one benchResult, other points picked by created-at window (fromMs / toMs epoch-ms, either bound optional)                                                                                                                                              |
| `create_comparison_report`             | COMPARISON report over 2 to 4 finished runs of one project — the backend lays out the report items with one metric per run. `benchResultIds` is the column order; optional `names` labels the columns instead of the automatic Run A / B / C / D. Static list: adding a run means another report. |
| `export_bench_report_pdf`              | Submit an async task that renders the report as a PDF (headless Playwright print). Returns a `taskId` to poll with `get_task_result`; on SUCCESS, the PDF is attached to the report's first benchResult — pull it via `list_bench_result_files` + `download_bench_result_file`. |
| `get_report_data_status`               | Whether the report's runs hold their data in the format report queries read — sorts them into `upToDate` / `needsUpdate` / `updating` / `unknown`. A run in the previous format answers every metric query empty, so the value tools refuse rather than return zeros.             |
| `update_report_data`                   | **Destructive** — rewrites a run's stored data into the current format so the report becomes readable. One async `taskId` per run to poll with `get_task_result`. Ask the user first: it can take minutes on a large run. See `octoperf://skills/bench-reports`.                  |

### Analysis — report templates

A template is a saved report layout, reusable on any later run. It belongs to the **workspace**, so every project of that workspace shares the same list. A run can also be built from one up front — `run_scenario` and `schedule_scenario_once` / `_cron` take an optional `templateId`. See `octoperf://skills/bench-reports`.

| Tool | Purpose |
|------|---------|
| `list_report_templates`              | Templates of the project's workspace — id, name, description, `itemCount`, url. Template ids are readable nowhere else. |
| `get_report_template`                | The template in full — its polymorphic `items` and its `configs`, what the page shows as "Report Contents". Required upstream of `patch_report_template`. |
| `patch_report_template`              | **Destructive** — RFC 6902 JSON Patch over the template (report items, per-metric colour / Apdex in each metric's own `configs`, template-wide `configs`). Schema: `octoperf://schema/bench-report-template`. Refuses a patch changing `id` / `workspaceId`; metrics are NOT allow-select validated (a template has no run). |
| `create_report_template_from_report` | Save an existing report as a template. The report is untouched. Filters naming the run's own infrastructure (`region`, `injectorId`, the run's own monitoring connections) are stripped and can take their report item with them — the answer carries `sourceItemCount` + `runSpecificFilterFields` to say what was lost. |
| `apply_report_template`              | **Destructive** — replaces a report's items with the template's, repointed at that report's run. No undo. Refused on a report comparing several runs (it would collapse to the first column) and on a template of another workspace. |
| `update_report_template`             | **Destructive** — rename / re-describe. The report items come from the source report, so re-extract to change them. |
| `delete_report_template`             | **Destructive** — drops the layout; the reports built from it keep their items. |

### Analysis — report item values

After `get_bench_report`, dispatch by the item's `@type` to read its aggregated values:

| Tool                                 | For item `@type`                                                                              | Returns                                                                              |
|--------------------------------------|-----------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| `get_report_table_values`            | `StatisticTableReportItem`                                                                    | `List<TableEntry>`                                                                   |
| `get_report_tree_values`             | `StatisticTreeReportItem` (per-VU breakdown for hybrid scenarios)                             | `List<TreeEntry>`                                                                    |
| `get_report_top_values`              | `TopReportItem` (with optional from/to window)                                                | `TopResult`                                                                          |
| `get_report_pie_values`              | `PieChartReportItem`                                                                          | `List<Map<String, Long>>`                                                            |
| `get_report_line_chart_values`       | `LineChartReportItem` / `PercentilesChartReportItem` (with from/to)                           | `List<List<GraphPoint>>`                                                             |
| `get_report_stacked_chart_values`    | `StackedChartReportItem`                                                                      | `List<MapGraphPoint>`                                                                |
| `get_report_area_range_values`       | `AreaRangeChartReportItem`                                                                    | `AreaRangeResult` (curve + ref + rmse)                                               |
| `get_report_summary_values`          | `SummaryReportItem` / `BarChartReportItem` (with optional from/to)                            | `List<Double>` aligned with `item.metrics`                                           |
| `get_report_insights`                | `InsightsReportItem` (with optional from/to window)                                           | `Set<Insight>` (id + level + value + drill-in report items)                               |
| `get_report_errors`                  | `ErrorsReportItem` (with optional from/to window)                                             | `List<BenchError>` (per-sample failures)                                             |
| `fetch_bench_error_http`             | One `BenchError` (`benchResultId` + `actionId` + `timestamp`)                                 | `(HttpRequestEntity, HttpResponseEntity)` of the failing sample                      |
| `get_report_threshold_alarms`        | `ThresholdAlarmReportItem` (with optional from/to window)                                     | `List<ThresholdAlarm>` (per-breach: severity, threshold, observed)                   |
| `get_report_textual_monitors`        | `TextualMonitorReportItem`                                                                    | `List<TextualCounterValue>` (string-valued monitor samples)                          |
| `list_bench_load_generators`         | `LoadGeneratorsChartReportItem` / `LoadGeneratorsTreeReportItem` — both pull from same source | `List<BenchLoadGenerator>` (one per LG container: region, host, VU count, start/end) |

### Analysis — report item editing

To change a report item's metrics/filters, resolve what's selectable, discover values, validate, then patch. See `octoperf://skills/report-item-editing`.

| Tool                             | Purpose                                                                                                                                              |
|----------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| `get_report_item_allow_select`   | Resolve an item's allowed `sampleTypes`, `metricIdsBySampleType`, and per-tag-key `tagKeyConstraints` (offered/required, allowedValues, single, dependsOn) |
| `get_report_tag_keys`            | Tag keys (filter dimensions) indexed for a report's bench results + sample type                                                                       |
| `get_report_tag_values`          | Distinct values per tag key, optionally scoped by already-selected filters (e.g. `injectorId` values for a chosen `region`)                           |
| `validate_report_item`           | Validate an item's metrics against its allow-select descriptor → `ReportItemValidation` (`valid` + `violations`). `patch_bench_report` runs this too. |

**Metric name semantics** — when reading report item metrics from any
`get_report_*_values` tool:

- `Hits` (and related rates: `Hits/s`, `Hits successful total`) count **HTTP/S samplers only**.
- `Hits (CONTAINER)` counts **everything else** — containers, logic actions (Loop / If / While …), JMeter plugins.
- For end-user actions, use `Hits`; for total VU activity (e.g. assessing whether a Loop completed), use `Hits (CONTAINER)`.

### Async tasks

| Tool              | Purpose                                            |
|-------------------|----------------------------------------------------|
| `get_task_result` | Poll any async OctoPerf task (e.g. correlation)    |

### Notifications

The **runtime alerting layer**: a notification fires a channel when a test run
reaches an event (started / ended / passed / failed / error), optionally
narrowed by filters. It pairs with `run_scenario` / `schedule_scenario_*`, and
`list_notifications_matching_scenario` previews which notifications a scenario's
runs will fire. One typed create/update tool per channel. Secrets (tokens /
webhook URLs / header values) are **write-only** — never returned, and must be
resent on update. The JIRA channel is multi-step: run the four `*_jira_*`
lookup tools (check → boards → statuses → transitions) to resolve the board /
status / transition ids, then `create_jira_notification`. Read
`octoperf://skills/notifications` first.

| Tool                                    | Purpose                                                                                     |
|-----------------------------------------|---------------------------------------------------------------------------------------------|
| `list_notifications_by_workspace`       | Notifications of a workspace (channel, events, filters summary, non-secret target)          |
| `get_notification`                      | One notification with its non-secret `details`                                              |
| `list_notifications_matching_scenario`  | Which notifications a given scenario's runs would fire                                       |
| `create_email_notification`             | Email channel (optional PDF report attachment)                                              |
| `create_slack_notification` / `_teams_` / `_google_chat_` / `_webex_` / `_http_` | Chat / webhook channels          |
| `create_jira_notification`              | JIRA channel — comments on / transitions board issues on test events (multi-step, see below) |
| `check_jira_notification_connection`    | Validate a JIRA connection (open-world) — do this before the JIRA lookups                    |
| `list_jira_notification_boards`         | JIRA boards (+ their project) reachable with the connection (open-world)                     |
| `list_jira_notification_statuses`       | Statuses of a JIRA project — pick the source "To do" status (open-world)                     |
| `list_jira_notification_transitions`    | Transitions of a JIRA board — pick the pass/fail/error target columns (open-world)           |
| `update_<channel>_notification`         | **Destructive** — overwrite a notification (resend the write-only secret)                    |
| `delete_notification`                   | **Destructive** — permanently delete a notification                                          |
| `test_notification`                     | Send a real test message to the channel (open-world; rate-limited 1 / 5s / user)            |

## Resources

The server publishes MCP resources alongside its tools. Load them via
`resources/read` whenever you need the matching context — schemas are
mandatory before writing a `patch_*` payload, skills are the deeper
playbook a workflow points at, and `templates/agents-md` is this very
file (so an agent can drop it at the user's project root).

### Templates

| URI                                | Mime               | Purpose                                                                                                 |
|------------------------------------|--------------------|---------------------------------------------------------------------------------------------------------|
| `octoperf://templates/agents-md`   | `text/markdown`    | This `AGENTS.md` — drop it at the user's project root to brief any CLI agent on how to drive the server |

### Skills (playbooks)

Read these as a system / context message before tackling the matching
workflow — they carry the failure catalogues and classification tables
that the workflow TL;DRs omit.

| URI                                            | Mime            | Purpose                                                                                                                                                                                                                                          |
|------------------------------------------------|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `octoperf://skills/auto-correlation`           | `text/markdown` | Playbook: fix a VU whose validation fails on dynamic values (CSRF tokens, sessions, signed URLs)                                                                                                                                                 |
| `octoperf://skills/validation-triage`          | `text/markdown` | Playbook: triage a VU with many validation failures, group by root cause, fix one per group                                                                                                                                                      |
| `octoperf://skills/scenario-diagnosis`         | `text/markdown` | Playbook: investigate a poor / failing scenario run (global metrics → drill-in → verdict)                                                                                                                                                        |
| `octoperf://skills/bench-reports`              | `text/markdown` | Reading guide: report item-by-report item mapping to the right `get_report_*_values` tool, semantic gotchas (Hits vs CONTAINER, trend report DELTA, Playwright row types)                                                                                  |
| `octoperf://skills/report-item-editing`        | `text/markdown` | Editing guide: allow-select-driven workflow to change a report item's metrics/filters (`get_report_item_allow_select` → `get_report_tag_keys`/`get_report_tag_values` → `validate_report_item` → `patch_bench_report`)                               |
| `octoperf://skills/real-browser-probe`         | `text/markdown` | Playbook: compose a hybrid scenario (N×JMeter for load + 1×Playwright probe) for user-perceived metrics during a bench                                                                                                                           |
| `octoperf://skills/scheduling`                 | `text/markdown` | Playbook: schedule a scenario one-shot or cron — Unix 5-field UTC format (NOT Quartz), timezone conversion, pause/resume/delete lifecycle                                                                                                        |
| `octoperf://skills/export-bench-report-pdf`    | `text/markdown` | Playbook: export a benchReport as PDF via the async print task (submit → poll `get_task_result` → download from the first benchResult)                                                                                                           |
| `octoperf://skills/async-polling`              | `text/markdown` | Reference: how to poll OctoPerf async ops (validate, run, export, correlate) — sleep cadence `expected_duration / 10` clamped `[3s, 60s]`, terminal conditions per status tool, anti-patterns (tight loop, `get_bench_status` as terminal check) |
| `octoperf://skills/onpremise-agent`            | `text/markdown` | Playbook: run load tests from your own machines — model a provider as a localized VU capacity, size it, create it, install one agent per machine, manage locations/agents; the one-agent-per-machine rule and the rename/remove-location → reinstall flow |
| `octoperf://skills/workspace`                  | `text/markdown` | Playbook: what a workspace scopes (members, providers/agents, notifications) vs what a project scopes; members by e-mail and the four roles; the two last-admin rules; adding a member is not an invitation; creating is free, deleting goes through the platform admin or support |
| `octoperf://skills/notifications`              | `text/markdown` | Playbook: manage workspace notifications — pick channel (email/Slack/Teams/Google Chat/Webex/HTTP/JIRA) → events → filters → test; write-only secrets, resend on update; JIRA multi-step lookup flow                                             |
| `octoperf://skills/monitoring`                 | `text/markdown` | Playbook: monitor servers during a run — pick an UP agent that reaches the target, create a monitor per type, then configure counters/applications (create-then-configure: `list_monitor_applications` → `preview_monitor_counters` → `update_monitor_counters`, select by path or `"*"` pattern); synchronous connection check; the self-contained SLA monitor |
| `octoperf://skills/scenario-composition`      | `text/markdown` | Playbook: build or reshape a scenario's userProfiles — the four load shapes and their millisecond fields, one engine per VU type, setUp/tearDown as profile settings, create-then-patch; plus **Pass Criteria** — what says the run passed, why none of it is stored on the scenario, and the derivation that reads the ones a test already has |
| `octoperf://skills/loadrunner-migration`       | `text/markdown` | Playbook: migrate a LoadRunner VuGen script and its Controller scenario — read the portability verdict before committing (a script can be `QTWeb` and send nothing at all: its calls are C), import, then work the `unconverted` list kind by kind (hand-written C, conditions reading C locals, recorded cookies, parameterised servers, loop bounds, load generators, the pace between iterations, the parameters and dates a script writes, the credentials on a server, the SLA profiles to attach, the emulated network a group sat on). **Mandatory before importing**: the entries say what happened, the skill says what to do |
| `octoperf://skills/neoload-migration`          | `text/markdown` | Playbook: migrate a NeoLoad project — read the portability verdict before committing, import, then work the `unconverted` list kind by kind (plugin dialogs, NeoLoad JavaScript, try/catch, waits, pushed and played requests, SOAP envelopes, container draws, monitors, SLA profiles). **Mandatory before importing**: the entries say what happened, the skill says what to do |
| `octoperf://skills/sla`                        | `text/markdown` | Playbook: Design SLA profiles — separating them from `create_sla_monitor`, the threshold-group model (severity, OPEN/CLOSED range bounds, occurrence-count vs millisecond hold), create-then-patch with sourced defaults, JSON Patch recipes, and the attach step without which nothing fires |

### JSON Schemas (mandatory before any `patch_*`)

Every schema is JSON Schema 2020-12 generated from the matching Java
class. The `oneOf` branches list every polymorphic subtype with its
`@type` discriminator and required fields — the LLM must keep those
intact when building a JSON Patch, otherwise the server's round-trip
Jackson validation rejects the payload. Rejection messages are
actionable: they identify the failing operation (index, op kind,
path) and the JSON pointer of the schema mismatch, with a specific
hint pointing back to the matching `octoperf://schema/*` resource
when the `@type` discriminator is invalid or missing — read the
error and retry with the same patch corrected.

> **Booleans default to `false` when adding a node.** When a `patch_*`
> op *adds* a node (e.g. an action into `children`), any boolean field
> you omit deserializes to `false` — and the round-trip validation does
> **not** flag it. So an action added without `"enabled": true` is
> created **disabled** and silently does nothing at run time. Always set
> `"enabled": true` explicitly on inserted actions (the JSON field is
> `enabled`, not `isEnabled`), and set any display / behaviour flags you
> actually want rather than relying on defaults.

| URI                                       | Mime               | Use it before                                                                                                                                                                    |
|-------------------------------------------|--------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `octoperf://schema/vu`                    | `application/json` | `patch_virtual_user` — full polymorphic Action / Extractor / Assertion / PostProcessor tree                                                                                      |
| `octoperf://schema/scenario`              | `application/json` | `patch_scenario` — `userProfiles` with polymorphic load shapes + engine settings                                                                                                 |
| `octoperf://schema/bench-report`          | `application/json` | `patch_bench_report` — polymorphic report items in `items` + per-benchResult `configs`                                                                                                |
| `octoperf://schema/variables`             | `application/json` | Constructing a polymorphic `Variable` payload manually (Constant / Counter / Random / CSV / Secret / List)                                                                       |
| `octoperf://schema/correlation-rules`     | `application/json` | Constructing a `CorrelationRule` payload (nested polymorphic extractor + InjectionRule) — `create_correlation_rule` takes typed params, so only needed for advanced custom rules |
| `octoperf://schema/injection-rules`       | `application/json` | The 8 `InjectionRule` subtypes (header name/value, query-param name/value, post-param name/value, request path, post body) — sub-schema of `correlation-rules`                   |

### HTTP fallback (clients that can't read MCP resources)

Some hosts (e.g. the claude.ai web client) call tools but don't read
MCP resources. Every schema above is therefore **also served as plain
JSON over HTTP**, unauthenticated, under `/mcp/public/schema/`. The
path mirrors the resource URI — `octoperf://schema/vu` →
`https://api.octoperf.com/mcp/public/schema/vu.json` (swap the origin
for a self-hosted server). Bodies are byte-identical to the MCP
resource.

| HTTP URL                                                       | Mirrors                               |
|----------------------------------------------------------------|---------------------------------------|
| `https://api.octoperf.com/mcp/public/schema/vu.json`           | `octoperf://schema/vu`                |
| `https://api.octoperf.com/mcp/public/schema/scenario.json`     | `octoperf://schema/scenario`          |
| `https://api.octoperf.com/mcp/public/schema/bench-report.json` | `octoperf://schema/bench-report`      |
| `https://api.octoperf.com/mcp/public/schema/variables.json`    | `octoperf://schema/variables`         |
| `https://api.octoperf.com/mcp/public/schema/correlation-rules.json` | `octoperf://schema/correlation-rules` |
| `https://api.octoperf.com/mcp/public/schema/injection-rules.json`   | `octoperf://schema/injection-rules`   |
| `https://api.octoperf.com/mcp/public/schema/sla-profile.json`  | `octoperf://schema/sla-profile`       |

## Recommended workflows

Each workflow is a quick-start TL;DR. For deeper playbooks (failure
catalogues, classification tables, log signatures, …) read the matching
MCP skill resource via `resources/read` — the server publishes them at
`octoperf://skills/*`:

| MCP resource URI                       | When to load                                                                                                                                 |
|----------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| `octoperf://skills/workspace`          | About to add someone to a workspace, change what they can do, or create a workspace                                                          |
| `octoperf://skills/auto-correlation`   | A VU validation fails on session / token / signed-URL replay (workflow 2)                                                                    |
| `octoperf://skills/validation-triage`  | A VU validation has many failures and you need to triage by root cause (workflow 3)                                                          |
| `octoperf://skills/scenario-diagnosis` | A scenario run produced bad / failing metrics and the user wants to know why (workflow 5)                                                    |
| `octoperf://skills/sla`                | Creating or editing an SLA profile, or explaining which SLA band a run breached                                                               |
| `octoperf://skills/async-polling`      | About to wait on a `taskId` or `benchResultId` (load test, validation, PDF, correlation) — defines the sleep cadence and terminal conditions |

### 1. Import → validate → fix → run

1. `import_*_virtual_user` (HAR, JMX, Postman, …) → returns the new VU id.
2. `sanity_check_virtual_user` → catches static issues with zero credit cost. Fix anything flagged.
3. `validate_virtual_user` against a Docker provider/location → starts a 1-user functional run.
4. Poll `get_virtual_user_validation` until `finished=true` (see `octoperf://skills/async-polling` for cadence — ~5s between calls).
5. If failures: `get_virtual_user_validation_index` lists the failing actions; for each, `get_validation_failure_detail` shows the four HTTP entities so you can diagnose. Apply correlation rules or edit variables / servers as needed.
6. Loop steps 3-5 until clean, then `run_scenario` and poll `get_bench_result` until `state ∈ {FINISHED, ABORTED, ERROR}` (use `get_bench_status` for progress display only; see `octoperf://skills/async-polling`); once FINISHED, pull the default report with `get_bench_report` and read its SummaryReportItem via `get_report_summary_values`.

### 2. Auto-correlation a HAR-imported VU

> Full playbook: `octoperf://skills/auto-correlation`.

Replays often fail because session tokens, CSRF inputs, or signed URLs
captured during recording are stale on the next run. Use this recipe:

1. Validate the VU; confirm failures look like correlation issues (`401`, `403`, signature mismatches, server-side state errors).
2. `list_correlation_frameworks` → pick the framework matching the target stack (SAML / OAuth / Token / …).
3. `add_correlation_framework_to_project` → submits an async task; poll with `get_task_result`.
4. Re-validate. For values the preset doesn't catch, write a regex rule with `create_correlation_rule`, then `apply_correlations_to_virtual_user`.

### 3. Triage validation failures across a VU

> Full playbook: `octoperf://skills/validation-triage` — includes the KO/OK matrix, the sanity-check message catalogue, and the engine-level-failure branch (read `jmeter.log` when there are no HTTP entities to read).

When validation has many failures, don't read them all serially:

1. `get_virtual_user_validation_index` → action-by-action summary (HTTP code, error type).
2. Group by error category (auth, missing variable, server error, body mismatch).
3. Drill into the *first* representative of each group with `get_validation_failure_detail` and `fetch_validation_http_body`.
4. Apply one fix (a correlation rule, a variable, an HTTP server timeout) → re-validate → confirm the whole group cleared.

### 4. Parameterize from a CSV

1. `upload_project_file` → upload the CSV.
2. `create_csv_variable` referencing that file, with the column names.
3. Re-import / edit the VU to bind action fields to `${variableName}` references.
4. Validate.

### 5. Diagnose a failing / underperforming scenario run

> Full playbook: `octoperf://skills/scenario-diagnosis` — includes the global-metrics classification table, jmeter.log signature catalogue (planned stop vs killed by infra vs stall abort), per-sample network-failure signatures (SSLException / SocketTimeoutException / …), the smoke-vs-load error-rate heuristic, and the load-generator-overload trust caveat.

1. `get_bench_status(benchResultId)` → confirm the run is terminal (FINISHED / ABORTED / ERROR) before reading metrics.
2. Find the report tied to the run via `list_bench_reports_by_project(projectId)` filtered on `benchResultIds`.
3. `get_bench_report(reportId)` → locate the SummaryReportItem and call `get_report_summary_values` for the global numbers.
4. Classify the run (high errors / climbing p95 / flat throughput / stopped early) → drill into the matching report item (`get_report_top_values` / `get_report_errors` / `get_report_insights`) or read `jmeter.log` via `list_bench_result_files` + `read_bench_result_file_lines`.
5. Surface a verdict + 2-3 metric snippets + a single next step.

## Safety conventions

- Tools marked **Destructive** in this guide consume credits or delete data. Always confirm with the user (PR-style summary, expected impact) before invoking them. Name the account the work will be done as — `get_current_user` — in that confirmation.
- `run_scenario` is the most expensive — it triggers a real load test on the configured providers. Never call it as part of an exploratory chain; only when the user explicitly asks to run. Call `get_scenario_matching_plans` first as a pre-flight: an empty list means no plan can host the scenario (use `list_active_subscriptions` to see which cap is binding); a non-empty list confirms the run is launchable as configured.
- `validate_virtual_user` does not consume credits, but it still spins up a load generator and takes time — batch fixes and re-validate once, not after every micro-edit.
- Imports overwrite nothing: each `import_*` creates a *new* VU. Use `update_*` / `delete_*` if you want to edit in place.

## Conventions

- IDs are opaque strings. Never construct them by hand; always read them from a `list_*` or a previous tool result.
- Rights are per workspace, not per account: read the `role` on the workspace you are about to write to rather than assuming the one you hold elsewhere. `CUSTOM` is a fine-grained access only the web UI edits — assume nothing from it.
- Listings always include a `url` deep-link to the OctoPerf UI. Render it when you summarize a list for the user.
- Most read tools are safe and idempotent. Prefer `list_*` / `get_*` before any mutation.
- Long-running operations (`apply_correlations_to_virtual_user`, validation, scenario run) all return ids you can poll with the matching `get_*_status` / `get_task_result` tool. Don't busy-loop; poll on a sensible interval and let the user know when it completes.

## See also

- OctoPerf support: <mailto:support@octoperf.com>
- OctoPerf docs: <https://api.octoperf.com/doc>
- OctoPerf blog: <https://blog.octoperf.com>
- OctoPerf tutorials: <https://www.iorad.com/help-center/159034?roleId=7490>
- Claude Code plugin marketplace: <https://github.com/OctoPerf/octoperf-claude-plugins>
