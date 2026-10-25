---
name: octoperf-k6-scripting
description: Use when writing, importing, editing, validating or exporting a K6 Virtual User on OctoPerf. Triggers on "import my K6 script", "run my k6 test on OctoPerf", "write a K6 script for this journey", "use a CSV in my K6 script", "read a project variable from K6", "why does my K6 validation fail", "create the scenario from my K6 options", "run this VU locally with k6 run". Covers the Virtual User as its set of script files (relative imports, no npm install, no bundler), the reserved names `octoperf.js` and `__octoperf_`, where the project variables surface (`__ENV` and the generated `./octoperf.js`), the SHARED / PRIVATE split of CSV files across load generators, the load the script declares turned into a scenario with `create_scenario_from_k6_options`, and the write, check and validate workflow. Requires the OctoPerf MCP server.
---

# K6 scripting on OctoPerf — files, variables, validation

A K6 Virtual User **is** its set of script files. OctoPerf ships them into an
`octoperf/k6` container, generates a `./octoperf.js` module next to them, and runs
`k6 run <entrypoint>` there. Read this before writing, importing or editing a K6
script — it carries what a `.js` file cannot guess: where the project's variables
show up, when a CSV is split across injectors and when each one gets it whole, and
which file names are reserved.

## The Virtual User is its files

- `import_k6_virtual_user` mints a presigned URL; POST **every** file of the VU in
  that one request, one `files` part each, the part filename being the name the
  scripts import it under (`script.js`, `helpers/login.js`). Each part becomes one
  `K6ScriptAction`. Chain into `describe_virtual_user` for the id and the UI link.
- Relative imports (`./helpers.js`) resolve by file name, `https://` imports are
  fetched. **No npm install, no bundler, no `node_modules`** — anything else must
  be vendored as a file of the VU.
- Editing afterwards: `get_virtual_user` to read the tree, then
  `patch_virtual_user` on the `K6ScriptAction` children (`fileName`, `script`).
- Running it locally: `export_virtual_user` downloads the VU as a ZIP that runs with
  `k6 run` as is — the scripts, a local `octoperf.js` with its data module, and the
  CSV files of the variables they use. Constants and secrets are never exported:
  pass them with `k6 run -e NAME=value`. Bring the edited scripts back with
  `patch_virtual_user`, or `import_k6_virtual_user` for a new VU: POST the `.js` /
  `.ts` scripts only — the import leaves out `octoperf.js` and `__octoperf_*`, and
  refuses `README.md` and the CSV files. A changed CSV goes back through
  `upload_project_file`. Locally, `row()` reads the whole CSV: the split across load
  generators only happens on OctoPerf.
- The entrypoint is `entrypointFileName` — on the scenario's `K6UserProfileEngine`
  and as a parameter of `validate_virtual_user`. Empty means the VU's first script.

## Reserved names

The generated module is `octoperf.js` at the root, and the prefix `__octoperf_` is
reserved for everything else the platform writes beside the scripts. A user script
taking either name is rejected by `sanity_check_virtual_user` as an ERROR — rename
it, the import will not silently win the collision.

The other file-name rules, same check: a relative path with no `..`, no backslash
and no space, ending in `.js` or `.ts`; names unique across the VU; a name no
project file already carries, since the scripts are unzipped over the project files
on the load generator; at least one script in the VU.

## Variables — `__ENV` and `./octoperf.js`

**`__ENV.<NAME>`** holds every **constant** and **secret** variable of the project,
by variable name — the K6 counterpart of `process.env.<name>` in a Playwright VU.
Nothing else of the container environment is visible to the script.

Everything else comes from the generated module:

```js
import { run, vars, random, csv, row } from './octoperf.js';
```

| Export | Shape | Notes |
|---|---|---|
| `run` | `{benchResultId, regionId, injectorId, virtualUserName, userProfileIndex}` | identity of the run and of **this** injector |
| `vars` | object | the same constants and secrets as `__ENV`, as data |
| `random(name)` | a new integer in `[min, max]` at **every** call | zero-padded when the variable's output format is a run of `0`s (the JMeter `DecimalFormat` subset) |
| `csv(name)` | `SharedArray` of row objects, keyed by the CSV variable's column names — or by the file's header line when no names are configured | built once at init, from what this container received: its slice (`SHARED`) or the whole file (`PRIVATE`) |
| `row(name)` | one row object | `csv(name)[exec.scenario.iterationInTest % rows.length]` |

## CSV files: split (SHARED) or whole (PRIVATE)

The `scope` of the CSV variable decides what each container of the run receives:

| Scope | Each container gets | Pick it for |
|---|---|---|
| `SHARED` | its own slice: rows **disjoint across injectors** | unique test data — accounts, orders, anything two injectors must never send twice |
| `PRIVATE` | the **whole file** | reference data every injector must walk whole — a catalog, a list of pages, the categories a journey loops over |

A slice keeps the header line when the file has one. A `SHARED` file holding fewer
rows than there are containers is not split either: every container gets it whole.

`csv(name)` returns the rows this container received, and `row(name)` hands out
one distinct row per iteration within that container, whatever the scope — the
natural replacement for JMeter's per-iteration CSV variable. The cursor is the
container's iteration count, not one per virtual user: under `PRIVATE`, every
container walks the same rows in the same order, starting from the first one.
A script walking the whole reference list in one iteration loops over `csv(name)`.

**A CSV variable counts as used when the script mentions its name in quotes** —
`row('customers')` or `csv("customers")`. That literal is what triggers the split
and the download, so never build the name at run time: `csv('cust' + 'omers')` or
a name read from a loop variable leaves the file absent from the container.

## Files opened at init are project files

The project files sit beside the scripts, and `open('invoice.pdf', 'b')` reads one. The load generator runs the
init code twice: first `k6 archive`, which packs the files the init code opens, then `k6 run` on that archive,
whose file system is what archiving packed. Both see the same `__ENV`, so a name built from a variable —
`` open(`batch-${__ENV.BATCH}.json`) ``, a loop up to `__ENV.BATCH_FILES` — works, provided every file it names is a
project file: one the init code does not open at archive time is not in the archive, and `open()` at run time
fails with `file does not exist`.

## Not provided, on purpose (JMeter parity)

- **Counter variables** — use `__VU`, `__ITER` or `k6/execution`
  (`exec.scenario.iterationInTest`, `exec.vu.idInTest`).
- **List variables** — list variables are not exposed to K6 scripts; use a CSV
  variable for tabular data.
- **`recycleOnEOF` / `stopThreadOnEOF`** — a `SharedArray` has no cursor to run off;
  `row()` wraps with the modulo and never ends an iteration.
- **No masking** of a secret the script prints itself — `console.log(vars.PASSWORD)`
  lands in the K6 log as typed. Don't.
- **Sharding is native**: OctoPerf passes `--execution-segment` per container.
  Never shard by hand off `__VU`, `run.injectorId` or the CSV length.

The load is the scenario's too: OctoPerf drops `vus`, `duration`, `iterations`,
`stages` and `scenarios` from the script's exported `options` and generates its own
executor from the profile's load shape. Keep thresholds and HTTP options there;
setting a VU count is a no-op. A threshold's `abortOnFail` is cleared unless the
K6 profile sets `abortOnFail`: the threshold is still evaluated and reported, and
the test runs its whole load.

## Importing a script that declares its own load

`import_k6_virtual_user` stores the `options` as written, and the run drops the load half of
them. So right after the import, turn that half into the scenario with
`create_scenario_from_k6_options` — the backend converts it, never guess it by hand:

1. **With `k6` installed**, run on the user's machine, where the script belongs:
   ```
   k6 inspect --execution-requirements <entrypoint>
   ```
   and pass the JSON it prints on stdout as `options`, untouched, with the entrypoint's file name as
   `entrypointFileName` — the profiles run that script, whose exported functions `exec` names. The flag is not
   optional:
   it makes K6 derive `scenarios` from the `vus` / `duration` / `iterations` / `stages`
   shortcuts, and `scenarios` is the only field the tool reads. Without it, a script using the
   shortcuts comes back as `NO_SCENARIOS`.
2. **Without `k6`**, transcribe the script's `options` into that shape yourself — one entry per
   scenario under `scenarios`, the shortcuts written as the scenario K6 derives from them:
   `vus` + `duration` → `{"executor": "constant-vus", "vus": …, "duration": …}`, `stages` →
   `{"executor": "ramping-vus", "startVUs": <vus>, "stages": […]}` (`startVUs` only when the script sets
   `vus`), `vus` + `iterations` → `{"executor": "shared-iterations", "vus": …, "iterations": …}`. Durations
   are strings (`"30s"`, `"1m30s"`); leave out what the script leaves to the K6 default — no count below zero.

Each K6 scenario becomes one profile named after it, running its `exec` function with its `env`
and `gracefulStop`, on the `providerId` and `location` you pass: the workspace's own from
`list_docker_providers_by_workspace`, or a public one from `list_public_docker_providers`.
Then report every `unconverted`
entry, by `kind`; its `outcome` says what became of the scenario — `NOT_CONVERTED` (no profile),
`APPROXIMATED` or `IGNORED` (the profile runs, without that setting):

| `kind` | What happened | What to tell the user |
|---|---|---|
| `NO_SCENARIOS` | nothing was read | re-run `k6 inspect` **with** `--execution-requirements` |
| `OPEN_MODEL` | no profile for this scenario | `constant-arrival-rate` / `ramping-arrival-rate` target a rate, an OctoPerf profile a number of concurrent users. Name the rate and let them pick a user count — then `create_scenario_ramp_up` or `patch_scenario`. Never turn a rate into users silently |
| `UNKNOWN_EXECUTOR` | no profile for this scenario | the executor does not exist in K6 2.x (`externally-controlled` among them): ask what load they meant |
| `SHARED_ITERATIONS` | approximated | K6 shares one stock of iterations, a fast VU taking more; the profile gives each VU the stock divided by the VUs, rounded up. Say it as an approximation |
| `START_TIME_WITH_ITERATIONS` | `startTime` dropped | an iteration-based profile starts with the test |
| `GRACEFUL_RAMP_DOWN` | dropped | OctoPerf stops ramped-down VUs at once, so the test lasts what the scenario announces |
| `SCENARIO_TAGS` | dropped | the samples carry OctoPerf's tags |

A scenario without `gracefulStop` or `gracefulRampDown` gets OctoPerf's zero rather than K6's 30
seconds, silently: only an explicit non-zero `gracefulRampDown` is reported. `octoperf://skills/scenario-composition` covers editing the profiles
afterwards.

Say the rest plainly, once: the script's `vus`, `duration`, `iterations`, `stages` and
`scenarios` are ignored at run time — `sanity_check_virtual_user` warns about the same thing — so
after the import the scenario is the only place the load lives.

## Script skeleton

```js
import http from 'k6/http';
import { check } from 'k6';
import { row, random, vars, run } from './octoperf.js';

export default function () {
  const customer = row('customers');           // {id, email} from this injector's slice
  const res = http.post(`${__ENV.BASE_URL}/login`, {
    user: customer.email,
    password: __ENV.PASSWORD,
    order: random('orderId'),
  });
  check(res, { 'logged in': r => r.status === 200 });
}
```

## Workflow — write, check, validate

1. `list_variables(projectId)` — read the names you are allowed to reference, the
   CSV column names, and the min/max and output format of the random ones. A name
   you invent resolves to nothing at run time.
2. Missing data? `upload_project_file` then `create_csv_variable` on it
   (`create_constant_variable` / `create_secret_variable` for an `__ENV` entry).
3. Write the files, `import_k6_virtual_user`, POST them all in one request,
   `describe_virtual_user` for the id and the deep-link.
4. `sanity_check_virtual_user` — zero credits. Fix every ERROR (reserved name,
   extension, duplicate file name, a project file of the same name, a Counter
   variable read from the script) before spending a run on it.
5. `validate_virtual_user` with `entrypointFileName`, then poll
   `get_virtual_user_validation` (cadence: `octoperf://skills/async-polling`).
   Read `get_virtual_user_validation_index` first: each `check()` is a row
   `check: <name>` of its own, failed when the check was false. Then the K6
   stdout/stderr, with `list_bench_result_files` +
   `read_bench_result_file_lines` — a script that fails to import `./octoperf.js`,
   or asks for a CSV it never quoted, says so there.
6. Clean? Put it in a scenario as a K6 user profile —
   `octoperf://skills/scenario-composition` for the `K6UserProfileEngine` block.

## Notes

- One iteration = one `export default function` call. Init code (module scope,
  `csv()`) runs once per VU; keep the HTTP calls out of it.
- A K6 VU carries no OctoPerf action tree: correlation rules, extractors and
  assertions do not apply to it. Correlate in JavaScript, and assert with
  `check()`: a failed check fails its own row of the validation index.
- A K6 script that throws is diagnosed from its own log, not from an action tree:
  `list_bench_result_files` + `read_bench_result_file_lines` on the validation
  `benchResultId` is the first thing to read.
