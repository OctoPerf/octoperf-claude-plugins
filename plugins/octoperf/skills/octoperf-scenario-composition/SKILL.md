---
name: octoperf-scenario-composition
description: Use when building or reshaping an OctoPerf scenario — its user profiles, load shapes, engines, set-up/tear-down and pass criteria. Triggers on "create a scenario with 100 users", "ramp up over 10 minutes", "add a user profile", "run this VU from another location", "set pass criteria on the test". Covers create-then-patch, load shapes, one engine per Virtual User type, why set-up/tear-down are not user profiles, pass criteria and re-pointing a profile. Requires the OctoPerf MCP server.
---

# Scenario composition — user profiles, load shapes, engines

A scenario is a list of **user profiles**. One profile = one Virtual User, played on
one provider location, under one load shape, through one engine. Read this before
building or reshaping one; `octoperf://schema/scenario` carries the exact field
list, this page carries the parts a schema cannot say — including what says the run
passed, which lives nowhere in that schema.

## Create, then patch

Never hand-write a whole profile. Create the scenario with a `create_scenario_*`
shortcut using a VU of the engine type you want — the backend fills a complete
engine block — then change what you need with `patch_scenario` (RFC 6902) or
`update_user_profile`.

An engine block written from scratch has to get `bandwidth`, `browser.cache`,
`browser.cookies`, `dns`, `jtl` and `properties` all right, and the round-trip
validation rejects an incomplete one.

## Load shapes

Four shapes, **every duration in milliseconds**, and the shortcut that creates each:

| `@type` | Fields | Shortcut |
|---|---|---|
| `UserProfileLoadRampUp` | `delayMs`, `rampUpMs`, `plateauMs`, `plateauVus` | `create_scenario_ramp_up` |
| `UserProfileLoadRampUpThenRampDown` | the same + `rampDownMs` | `create_scenario_ramp_up_down` |
| `UserProfileLoadAscendingStairs` | the same + `stepCount` | `create_scenario_stairs` |
| `UserProfileLoadCustom` | `points`: a list of `ThreadGroupPoint` | none — patch only |

The shortcuts take **seconds** and multiply; the persisted document is in
milliseconds, and the scenario's **Load** step draws the same shapes as a curve.
`rampUpMs: 0` collapses to an instant constant load. Each shortcut applies the same
shape to every profile it creates: differentiating them is a patch.

## One engine per Virtual User type

The engine block is not free-form — it follows the type of the VU the profile plays:

| VU type | Engine | Carries |
|---|---|---|
| `JMETER` | `JmeterUserProfileEngine` | `settings`, `bandwidth`, `browser`, `dns`, `jtl`, `properties` |
| `WEB_DRIVER` | `SeleniumUserProfileEngine` | `settings`, `bandwidth` |
| `PLAYWRIGHT` | `PlaywrightUserProfileEngine` | `settings`, `device`, `timeoutMs`, `baseURL`, `slowMoMs` |
| `K6` | `K6UserProfileEngine` | `entrypointFileName` (empty = the first script of the VU), `exec` (the exported function of that script the profile runs, empty = its `default` export) and the `k6 run` flags OctoPerf imposes: `insecureSkipTlsVerify`, `noConnectionReuse`, `gracefulStopMs` (absent = 0, not K6's 30 s: the iterations still running when the load ends are interrupted), `noThresholds`, `throwOnError`, `abortOnFail` (off by default: OctoPerf clears the `abortOnFail` of the script thresholds, which stay evaluated and reported; on, a failing one stops K6 on its load generator, the others going on), `maxRedirects`, `userAgent`, `rps`, `batch`, `batchPerHost`, `dns` (`ttl`, `select`, `policy`), `blockHostnames`, `blacklistIps`, `env` (`-e` variables), `uploadCsvOutput` (uploads the raw `--out csv` file with the logs of a standard run, off by default), `memory.capturePercent` (share of the container, 1 to 50, the HTTP capture previews may hold; absent = the platform default, and the budget stays within the platform bounds); no JMeter `settings` block at all, the load is the scenario's |
| `FRAGMENTS` | none | a fragment is reusable content, it cannot be loaded on its own |

A K6 profile's `entrypointFileName` names a file of the VU, and the script itself
reads the project's variables through `__ENV` and its generated `./octoperf.js` —
`octoperf://skills/k6-scripting` before touching one.

So DNS overrides and JMeter properties simply do not exist on a Playwright profile,
and a device is meaningless on a JMeter one. `update_user_profile` swaps the whole
block when you re-point a profile to a VU of another type — which resets that
profile's engine settings to the defaults of the new type.

The three JMeter-based engines share `JmeterEngineSettings`: `thinkTime`, `errorHandling`,
`externalLiveReporting`, and the `setUp` / `tearDown` below.

## Set Up and Tear Down are not user profiles

**Set Up** runs a Virtual User before the profile's own users, **Tear Down** after.
Everything else about them is identical — same options, same meaning, same panel;
only the moment differs.

Both are **options of an existing profile**, not extra entries in `userProfiles`:
they live in `engine.settings.setUp` / `.tearDown` and carry their *own* Virtual
User, which has to be a **JMeter** one even when the profile itself is Playwright.

**The time they take is taken out of the total testing time** — both of them — so
set a Lifetime rather than letting the VU run as long as it wants.

The options, in the order the Options → Set Up and Options → Tear Down panels
present them:

| Option | Values | Field |
|---|---|---|
| Virtual User | a JMeter VU of the project | `virtualUserId` |
| Scope | Local / Global | `scope`: `LOCAL` / `GLOBAL` |
| Error handling | Continue / Start next VU iteration / Stop VU / Stop test / Stop test now | `onSampleError` |
| Number of concurrent users | One / Same number as main VU | `numberOfThreads`: `1` / `-1` |
| Ramp-up period in s. | only with *Same number as main VU* | `rampupPeriodSec` |
| VU iteration behavior | Start from scratch / Keep cache and cookies | `sameUserOnEachIteration`: `false` / `true` |
| VU End | VU duration / Max iterations | `loopCount`: `-1` / the count |
| Lifetime | VU End / Duration in s. | `pacing`: absent / `{"min": 0, "max": <s>, "unit": "SECONDS"}` |

A single run of a single user, before or after the test:

```json
{"virtualUserId": "…", "scope": "LOCAL", "onSampleError": "CONTINUE",
 "numberOfThreads": 1, "rampupPeriodSec": 0, "sameUserOnEachIteration": false,
 "loopCount": 1, "pacing": null}
```

Both are absent by default; `null` removes one.

## Pass Criteria say what a passed run means

The scenario page carries a **Pass Criteria** step, and it holds three lines — its own
words on the left:

| Line | What it counts |
|---|---|
| *All SLA profiles used in the Virtual Users* | the design SLA profiles attached to the VUs this scenario plays |
| *SLA thresholds applied to aggregated test metrics (e.g. response time, error rate)* | the project's **enabled** monitors of type `SLA` |
| *Thresholds on server, database, or system-level metrics* | every other **enabled** monitor of the project |

**None of it is stored on the scenario.** There is no field to patch and nothing to
create alongside a user profile: a scenario has pass criteria because the VUs it plays
carry SLA profiles, and because the project it belongs to has monitors switched on.
Re-pointing a profile to another VU changes them; enabling a monitor changes them for
every scenario of that project at once.

Worth naming **once** while a test is composed, and dropping the moment the user is not
interested: plenty of tests legitimately have none, and a run without pass criteria is
read by eye rather than failed automatically. It is not a step to enforce — never create
a profile or a monitor to fill the section in.

When the user does want one, which half they mean decides where to read next:

- bands on the test's own metrics — response time, errors, Apdex → `octoperf://skills/sla`
- the machines behind it — hosts, databases, web and app servers → `octoperf://skills/monitoring`

### Reading the ones a test already has

No tool returns them in one call. This is the derivation, and it is the one the web UI
performs to draw that step:

1. `get_scenario(scenarioId)` → the distinct `userProfiles[].virtualUserId`.
2. `get_virtual_user` on each → an `SLAProfileAction` in the action tree names a profile
   the VU carries, in its `profileId`. `get_sla_profile` then expands its bands.
3. `list_monitors_by_project(projectId)` → keep the **enabled** ones. Those of type `SLA`
   answer the second line, all the others the third.
4. `get_monitor_counters(monitorId)` → which of its counters actually carry a threshold.
   A monitor collects whether or not anything grades it, so an enabled monitor with no
   threshold anywhere watches without ever judging: it belongs to the report, not to the
   pass criteria.

After a run, the breaches themselves come from `get_report_threshold_alarms` on the
report's `ThresholdAlarmReportItem`, and `get_report_monitors_table_values` says which
connection raised how many and how badly.

## Re-pointing a profile

`update_user_profile` changes what a profile plays and where — the scenario's **VU**
and **Location** steps — and nothing else. A provider or a location the workspace
cannot use is refused, and the refusal names the ones it can. The workspace runs on its
own providers, `list_docker_providers_by_workspace`, and on the public ones,
`list_public_docker_providers`.

Everything else is `patch_scenario` against the profile's path, with
`octoperf://schema/scenario` as the source of truth for the shape.

## Check the scenario before it runs

Once composed, call `sanity_check_scenario`. It runs nothing and costs no credit, and it
catches what would otherwise fail at the launch, each issue under its `rule`:
- Virtual Users: missing or outside the project, set-up and tear-down included
  (`VIRTUAL_USER_MISSING`, `VIRTUAL_USER_OUT_OF_PROJECT`), a Fragments Virtual User, an
  engine that does not match its Virtual User, a K6 script with no entrypoint.
- Providers and locations: a provider missing, unreachable or disabled, a region the
  provider does not offer, too few dedicated IPs (`DEDICATED_IPS_SHORT`, a WARNING), a
  private server a cloud location cannot reach, IP spoofing outside an on-premise location.
- On-premise capacity, the load generators placed on the agents `UP` the way a launch does:
  no agent in the location (`AGENT_MISSING`), not enough memory even on idle agents
  (`CAPACITY_EXCEEDED`), agents busy with other tests right now (`CAPACITY_BUSY`, a WARNING),
  agents sharing one machine, whose memory is then counted several times
  (`AGENTS_SHARING_A_MACHINE`, a WARNING), a memory per agent declared so low that the load
  starts more load generators than the agents need, or under half the memory of the smallest
  agent (`MEMORY_UNDER_DECLARED`, an INFO).
- Load: no Virtual User at its peak, a load or a set-up / tear-down that never ends,
  iterations past the watchdog, a memory per Virtual User out of range.
- Subscriptions: none of the user's subscriptions can run the scenario
  (`SUBSCRIPTION_MISSING`), each one with the limit that stops it.
- Settings: a live reporting listener or a DNS server with no address, user profiles
  duplicated or sharing a name.

Every issue names the user profile to fix, by id for `update_user_profile` and by name for
the user. An ERROR blocks the launch: `run_scenario` and every scheduled fire refuse the
scenario until it is fixed; a WARNING or an INFO never does.
