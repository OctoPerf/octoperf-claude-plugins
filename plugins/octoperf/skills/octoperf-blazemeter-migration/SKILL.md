---
name: octoperf-blazemeter-migration
description: Use when migrating BlazeMeter tests to OctoPerf. Triggers on "migrate from BlazeMeter", "import my BlazeMeter tests", "we're leaving BlazeMeter", "leaving Perforce", "bzm to octoperf", "convert my Taurus test". Inventories a BlazeMeter account, brings the JMeter and Taurus tests across as Virtual Users, rebuilds the data files and load profiles as OctoPerf scenarios, and reports what BlazeMeter holds that has no OctoPerf equivalent. The agent calls the BlazeMeter API itself with the user's own keys — OctoPerf servers never do. Requires the OctoPerf MCP server.
---

# OctoPerf — BlazeMeter migration

Unlike the NeoLoad and LoadRunner migrations, there is no OctoPerf
importer for BlazeMeter and there does not need to be one. A BlazeMeter
performance test *is* a JMeter `.jmx` plus its data files plus a load
profile — three things OctoPerf already reads. The whole migration is
orchestration, and this skill is the orchestrator.

**You make the BlazeMeter calls, not OctoPerf.** Read
[Before you start](#1-before-you-start) before the first request: this
is a rule about where the credentials live, not a deployment detail.

## 1. Before you start

### The keys stay with the user

BlazeMeter authenticates with an **API key id + secret** pair, sent as
HTTP Basic (`Authorization: Basic base64(id:secret)`). Ask the user for
theirs — Settings → API Keys in the BlazeMeter UI.

- Never echo a key back, never write one into a file you create, never
  put one in a URL you show. If the user points you at a file that
  already holds a key in clear, read it if you must, then say so and
  recommend rotating it.
- Never send a BlazeMeter key to OctoPerf. Nothing in the OctoPerf API
  takes one, and no OctoPerf tool should ever receive one.

OctoPerf's servers must never call `blazemeter.com`. Every BlazeMeter
request in this skill is made **by you**, on behalf of the user, with
the user's keys. What reaches OctoPerf is bytes: a `.jmx`, a `.csv`, a
scenario definition. Never a key, never a BlazeMeter URL for OctoPerf to
fetch.

### Say who is responsible

Before the first call, tell the user, in one sentence: their use of the
BlazeMeter API — including exporting and migrating their own data — is
governed by their own agreement with BlazeMeter/Perforce, and they
should confirm they are permitted to do it. OctoPerf is not affiliated
with BlazeMeter, and BlazeMeter is a trademark of Perforce Software.

Stop immediately, and tell the user, on `401`, `403` or `429`. Do not
retry around an authorization error and do not work around a rate
limit — a suspended account is a worse outcome than a slow migration.

### Claude.ai web needs two allowed domains

Presigned OctoPerf uploads already require `octoperf.com` in the
workspace **Domain allowlist**. This skill also reaches BlazeMeter, so
`blazemeter.com` must be added the same way (Organization Settings →
Capabilities → Domain allowlist). Without it the sandbox cannot reach
`a.blazemeter.com` and every step below fails on a network error, with
no useful message. On Claude Code CLI there is no allowlist.

## 2. Inventory first — write nothing

Skipping this step is how migrations get committed to and then
abandoned. Build the picture before creating anything in OctoPerf.

Base URL `https://a.blazemeter.com`. Paginate everything: pass
`skip` / `limit` with `limit=100` and keep going while the page comes
back full.

| Call | Gives you |
|---|---|
| `POST /api/v4/search` body `{"entity":"workspace","skip":0,"limit":100}` | the workspaces |
| `POST /api/v4/search` body `{"entity":"project","workspaceId":<id>,"skip":0,"limit":100}` | the projects **of that workspace** |
| `GET /api/v4/tests?projectId=<id>&platform=performance&skip=0&limit=100` | the tests |

**Scope the project search by workspace.** Without `workspaceId` the
search answers with the projects of the whole account, and walking that
same list once per workspace re-creates every project inside every
workspace — a cartesian product on any multi-workspace account.

### Read `configuration.scriptType` and give a verdict

Group the tests and tell the user what the account actually holds
before importing anything:

| `scriptType` | Verdict | Why |
|---|---|---|
| `jmeter` | **Mechanical** | The `.jmx` is what OctoPerf's JMeter importer already reads |
| `taurus` | **Assisted** | A YAML wrapper: mechanical when it points at a `.jmx`, hand-built when the requests are inline — see §7 |
| `selenium` | **Manual** | A Java/Python/C# project; OctoPerf runs Playwright or WebDriver VUs, which is a rewrite, not a conversion |
| `gatling`, `locust`, `grinder`, `pbench`, `siege`, `apiritif`, `nose`, `pytest`, `robot` | **Not migrated** | No OctoPerf equivalent — the traffic has to be rebuilt |

Count them and say the ratio out loud. An account that is nine tenths
Gatling is not a difficult migration, it is a different project, and
the user should hear that before you spend an afternoon on it.

Also flag, from the same listing, what a migration never touches:
`platform=functional` tests, multi-tests
(`GET /api/v4/multi-tests?projectId=`), mock services, API monitors,
private locations, historical reports and trends, and team members.
None of them come across. Say so now rather than at the end.

## 3. Create the OctoPerf side

Mirror the BlazeMeter hierarchy: one workspace per workspace, one
`DESIGN` project per project.

Reuse before creating — `list_workspaces` and
`list_projects_by_workspace` first, match on name, and only
`create_workspace` / `create_project` when nothing matches. Ask the
user before creating a second workspace with a name they already have.

## 4. Bring each test across

For one test id:

1. `GET /api/v4/tests/{testId}/files` → a list of `{name, link}`. The
   `link` is **pre-signed**: fetch it with no `Authorization` header at
   all. Sending Basic auth to it is what breaks this step.
2. The JMX is the entry whose `name` equals `configuration.filename`.
   Call `upload_jmx_virtual_user` for the project, POST the `.jmx`
   bytes to the presigned URL it returns, and read the Virtual User
   ids out of the response.
3. Every other file is a data file. `upload_project_file` per file,
   same presigned-POST pattern.
4. `describe_virtual_user` on each new id, to give the user the UI
   deep-link.

If `configuration.filename` names a file that is not in the listing,
the test is broken on the BlazeMeter side. Record it and move on —
do not fail the whole migration on one test.

**Report every failure.** A rejected JMX is easy to lose: nothing else
in the run will mention it, and a migration that reports nothing reads
as a migration that worked. Keep a running list; §10 prints it.

## 5. Shared folders

`configuration.plugins.sharedFolders` names the folders a test pulls
files from. For each one, `GET /api/v4/folders/{folderId}/files` and
upload the contents with `upload_project_file`.

Tell the user what this flattens: a BlazeMeter shared folder lives at
the **workspace** level and is referenced by many tests, while OctoPerf
project files live inside **one project**. Migrating N projects that
share a folder copies that folder N times, and they will drift apart
afterwards. Only migrate the folders actually referenced by a test.

## 6. Test data sets

A test's `dependencies.data` (from `GET /api/v4/tests/{testId}`) is a
BlazeMeter Test Data Model. Read it before acting: it decides which of
three cases you are in, and two of them need no work at all.

### First, look at what the import already did

**A JMeter script usually declares its own CSV.** If the `.jmx` holds a
`CSVDataSet`, `upload_jmx_virtual_user` has already created the matching
OctoPerf variable — with the column names **the script actually
substitutes**. Call `list_project_files` and `list_variables` after §4
and before creating anything.

Creating a second variable over the same file is a real defect, not
untidiness: `sanity_check_virtual_user` reports *"File x.csv is used by
multiple csv variables. This can lead to unpredictable behavior."*

**The `.jmx` wins over the data model.** The two name their columns
independently, and only the script's names are referenced by its
requests. A test whose TDM model maps `user1, pass` while the
`CSVDataSet` declares `variableNames = login,password` resolves
`${login}` and `${password}` at runtime — `${user1}` appears nowhere.
Grep the script before trusting a name from the model.

### Then, the three cases

| `dependencies.data` | What to do |
|---|---|
| absent | Nothing — the data files of §4 are the whole story |
| `datasources[].type == "csv"` naming a file already in `/tests/{id}/files` | **No TDM call.** The CSV is downloadable like any other test file; the model only supplies metadata |
| a generator with no backing file | The data is synthesised at run time — see below |

The middle case is the common one, and the model is worth reading even
when the variable already exists: `targets.*.columnMapping` gives the
column order, `delimiter`, `fileEncoding`, `quoted` and `isHeadless` its
parsing. `isHeadless: true` means **no header row** — pass
`ignoreFirstLine=false`, and do not mistake a first line that looks like
a header for one.

### Only for a real generator: ask first

Materialising synthesised data means calling `tdm.blazemeter.com`. That
is the one call in this migration whose presence in BlazeMeter's
*published* API reference could not be confirmed, and the user's
Perforce agreement requires published APIs only. Offer both, and let the
user choose:

- **Export from the UI** (preferred): the user downloads the data set as
  CSV and hands you the file.
- **Generate via the API**: only if the user says their agreement allows
  it.

Either way, finish with `upload_project_file` then
`create_csv_variable`, and **namespace the variable** — OctoPerf
variables are project-scoped while BlazeMeter data sets are
test-scoped, so two migrated tests using `username` collide silently.
Prefix with the test name and say which VUs now reference a renamed
variable.

## 7. Taurus tests

A `taurus` test's file is a YAML config. Fetch it like any other test
file and read it.

```yaml
execution:
- concurrency: 100
  ramp-up: 1m
  hold-for: 10m
  scenario: my-scenario
scenarios:
  my-scenario:
    script: my-test.jmx      # (a) points at a script
    requests:                # (b) requests declared inline
    - url: https://example.com/login
      method: POST
```

- **(a) `script:` naming a `.jmx`** — mechanical. The JMX is in the same
  test's file listing; take §4 from step 2.
- **(a) `script:` naming anything else** (`.py`, `.scala`, a Selenium
  project) — not migrated, report it.
- **(b) inline `requests:`** — build the VU yourself. For a flat list of
  URLs, `import_urls_virtual_user` with `DO_NOT_CRAWL` is the shortest
  path. Anything with bodies, extractors, assertions or `think-time`
  needs `patch_virtual_user` against `octoperf://schema/vu` afterwards.
  Say how much of it you rebuilt.

The `execution` block is also the load profile — §8 reads it the same
way it reads a JMeter test's `executions`.

## 8. Load profiles become scenarios

Stopping at the design leaves the user with Virtual Users and no way to
run them. The numbers are right there — use them.

A test's `executions[]` array (or `overrideExecutions[]` when the user
overrode the script's own thread groups) carries the load:

```json
{
  "concurrency": 20,
  "rampUp": "1m",
  "holdFor": "19m",
  "steps": 0,
  "locations": { "us-east-1": 20 },
  "executor": "jmeter"
}
```

`rampUp` and `holdFor` are **duration strings** (`"30s"`, `"1m"`,
`"2h"`); the OctoPerf tools take **seconds**. Convert, do not pass the
string through.

| BlazeMeter | OctoPerf tool |
|---|---|
| `steps` absent, `0` or `1` | `create_scenario_ramp_up` — `users`, `rampUpSec`, `holdForSec` |
| `steps` > 1 | `create_scenario_stairs` — the same plus `stepCount` |

BlazeMeter has no ramp-down, so `create_scenario_ramp_up_down` never
comes from a migration — only from a user asking for one.

Two mappings that are not one-to-one, and that you must state rather
than guess:

- **Locations.** BlazeMeter's `us-east-1` is a BlazeMeter region, not an
  OctoPerf one. Call `list_docker_providers_by_workspace`, show the
  user the providers and locations they actually have, and let them
  choose. An unknown location is refused by the tool.
- **Concurrency.** BlazeMeter's `concurrency` is per location and is
  split across them; OctoPerf's `users` applies to **each** user
  profile, so the total is `users × virtualUserIds.size()`. If you
  spread one BlazeMeter test over several OctoPerf locations, divide.
  Show the arithmetic and the resulting total.

`configuration.plugins.thresholds` holds the test's failure criteria.
They do not convert — rebuild them as an OctoPerf SLA profile with
`octoperf-sla` if the user wants them, and list them in the report
either way.

## 9. Validate before declaring victory

An import that has never run is not a migration.

1. `sanity_check_virtual_user` on each new VU — it catches the variable
   collisions from §6 and the missing data files from §4.
2. `validate_virtual_user`. Expect the first run to be red on any real
   project.
3. `octoperf-validation-triage` to group the failures, then
   `octoperf-auto-correlation` for the session tokens and CSRF values
   BlazeMeter's JMeter script correlated with its own extractors.

Do not run a scenario until one Virtual User validates clean —
`run_scenario` burns credits.

## 10. Report what happened

Close with a table, one row per BlazeMeter test. This is the
deliverable.

| Test | Type | Result | Notes |
|---|---|---|---|
| Checkout load | `jmeter` | Migrated | 3 VUs, 2 data files, scenario `Checkout 20u` |
| Search API | `taurus` | Partial | Inline requests rebuilt; 2 assertions dropped |
| Mobile journey | `gatling` | Not migrated | No OctoPerf equivalent — re-record needed |

Then a short section per category left behind, with the count: the
non-JMeter script types, functional tests, multi-tests, mock services,
API monitors, private locations, thresholds, historical reports and
trends, team members and permissions. A user who reads "not migrated"
next to a number can decide what to do; a user who discovers it in
three weeks cannot.

## Related skills

- `octoperf-validation-triage` — the first validation run will need it.
- `octoperf-auto-correlation` — for the dynamic values that break on replay.
- `octoperf-scenario-composition` — for load shapes and Set Up / Tear Down.
- `octoperf-sla` — for rebuilding the BlazeMeter failure criteria.
