# Design SLA profiles — pass/fail bands on a test's own metrics

Define what "too slow" and "too many errors" mean for a Virtual User, so a run
raises alarms instead of leaving someone to eyeball the report. Read this before
creating or editing an SLA profile.

## Two different things are called SLA

Get this right first — the tools do not overlap.

| | **Design SLA profile** (this skill) | **SLA monitor** (`create_sla_monitor`) |
|---|---|---|
| Scope | attached to a **Virtual User** | created on a **project** |
| Metrics | the ones **you** pick, with **your** bands | a fixed default set of per-request counters |
| Set up | `create_sla_profile` → `attach_sla_profile_to_virtual_user` | one call, pick an UP agent |
| Use it when | you have a target to enforce ("checkout under 1s") | you want per-request SLA counters with no thinking |

They can coexist on the same project. `create_sla_monitor` is documented in
`octoperf://skills/monitoring`; everything below is about profiles.

Both show up under **Pass Criteria** on the scenario page, next to the thresholds of
the project's other monitors — what a passed run means, in one place.
`octoperf://skills/scenario-composition` says what that step covers and how to read
the criteria a test already has.

## Mental model

A profile is a **named set of threshold groups**. One group = one metric plus
the graded bands that fire on it:

```
SLAProfile "Checkout SLA"
└── SLAMonitorThreshold  metric = RESPONSE_TIME_AVG
    ├── Threshold  WARNING   value > 1000 ms   for 10 samples
    └── Threshold  CRITICAL  value > 3000 ms   for 3 samples
```

A band carries three things:

- **`severity`** — `PASSED`, `WARNING` or `CRITICAL`. `PASSED` is not decoration:
  some metrics (Apdex, hits rate) declare the *good* band explicitly.
- **`value`** — a range: `lowerEndpoint` / `upperEndpoint` with `OPEN`
  (exclusive) or `CLOSED` (inclusive) bound types. **An omitted endpoint means
  unbounded** — `{lowerEndpoint: 5, lowerBoundType: "CLOSED"}` is "5 or more".
- **`time`** — how long it must hold before firing, one of two shapes:
  `{"@type":"OccurrenceCondition","times":10}` (10 samples in a row) or
  `{"@type":"DurationCondition","duration":30000}` (**milliseconds**, not
  seconds).

Bands within a group should not overlap, and a group with **zero** bands is
legal — it means "I care about this metric, bands to be defined".

## Nothing happens until it is attached

A profile on its own is inert. The chain is:

1. `attach_sla_profile_to_virtual_user` adds an `SLAProfileAction` to the VU's
   action tree.
2. On every run of a scenario using that VU, the backend turns the attached
   profiles into an **SLA monitor connection per load generator**, with one
   counter per threshold group.
3. A breached band raises a `ThresholdAlarm`, readable with
   `get_report_threshold_alarms`.

So a profile that raises nothing is usually a profile nobody attached. One
profile can be attached to several VUs; a VU can carry several profiles.

## Create, then patch

Do **not** try to author a threshold tree in one shot.

1. **`create_sla_profile(projectId, name, metrics)`** — name the metrics, the
   server fills each with the same defaults the OctoPerf web UI applies. Those
   defaults carry real domain knowledge: Apdex bands come from the Apdex
   specification, connect-time bands from MDN's network-throttling table. A
   metric with no default is still added, with no band.
2. **`get_sla_profile(slaProfileId)`** — read what you got, and use the indexes
   to compute JSON Pointer paths.
3. **`patch_sla_profile(slaProfileId, patch)`** — retune. RFC 6902, same
   mechanics as `patch_virtual_user`: the server re-deserialises the patched
   JSON and rejects an invalid shape.

`list_sla_threshold_defaults(metrics)` shows the bands **without** creating
anything — reach for it when the user asks "what would you suggest for response
time?" and you want to discuss numbers first. Always pass `metrics`: the whole
table spans every metric that has defaults and is large.

Read `octoperf://schema/sla-profile` before writing a patch that adds a node.
It is also served as plain JSON at `/mcp/public/schema/sla-profile.json`.

### Patch recipes

Paths are `/thresholds/<group>/thresholds/<band>/...`:

```jsonc
// retune a warning band
{"op":"replace","path":"/thresholds/0/thresholds/0/value/range/lowerEndpoint","value":800}

// make a band fire faster (3 samples instead of 10)
{"op":"replace","path":"/thresholds/0/thresholds/0/time/times","value":3}

// switch a band from "N samples" to "held for 30s"
{"op":"replace","path":"/thresholds/0/thresholds/0/time",
 "value":{"@type":"DurationCondition","duration":30000}}

// add a band to a group that has none
{"op":"add","path":"/thresholds/2/thresholds/0",
 "value":{"@type":"Threshold","id":"","name":"Too slow","severity":"WARNING",
          "time":{"@type":"OccurrenceCondition","times":5},
          "value":{"@type":"RangeCondition",
                   "range":{"lowerEndpoint":2000,"lowerBoundType":"OPEN"}}}}

// drop a whole metric group
{"op":"remove","path":"/thresholds/1"}
```

Leave `"id":""` on a node you create — the backend assigns it.

## Manage

- **`list_sla_profiles(projectId)`** — id, name, the `metrics` each profile
  constrains, a `thresholdCount`, and a `url` deep-link. Bands are omitted; call
  `get_sla_profile` for those.
- **`attach_sla_profile_to_virtual_user`** / **`detach_…`** — both return the
  profile ids the VU carries afterwards. Attaching twice is a no-op; detaching
  something absent is a no-op. Detaching leaves the profile alive for other VUs.
- **`delete_sla_profile`** — DESTRUCTIVE; confirm first. **Detach it everywhere
  first**: a VU keeping an action that points at a deleted profile is a broken
  reference.

## Pitfalls

- **Confusing the two SLAs.** `create_sla_monitor` does not read your profiles,
  and a profile does not need a monitor. Picking the wrong one silently gives
  the user something they did not ask for.
- **Creating a profile and stopping there.** Without
  `attach_sla_profile_to_virtual_user` it will never fire. Say so when you
  report a created profile.
- **`DurationCondition` in seconds.** It is milliseconds. `30` means 30 ms, not
  30 seconds.
- **Reading a band as a plain number.** `> 1000` and `>= 1000` differ by the
  bound type, and a missing endpoint means unbounded, not zero. Report a band
  the way it reads, not as a single figure.
- **Authoring bands from scratch.** `create_sla_profile` already gives sourced
  values. Inventing "response time critical at 5s" throws that away.
- **Expecting names in alarms.** `get_report_threshold_alarms` returns
  `thresholdId` / `counterId` / `connectionId` — ids, no names. Cross-reference
  with the profile if the user needs to know which band fired.
- **Deleting before detaching.** See above.

## Typical workflows

- **"Fail the test if checkout is slower than 1s"**: `create_sla_profile` with
  `RESPONSE_TIME_AVG` → `patch_sla_profile` to move the warning band to 1000 and
  the critical one where the user wants → `attach_sla_profile_to_virtual_user`
  on the checkout VU.
- **"What SLA should I put on my API?"**: `list_sla_threshold_defaults` for the
  metrics that matter → discuss the numbers → create with the agreed metrics →
  patch what the user changed.
- **"Why did my run raise alarms?"**: `get_report_threshold_alarms` →
  `list_sla_profiles` on the project and `get_sla_profile` to map each
  `thresholdId` back to a named band → explain which target was missed and for
  how long.
- **"Reuse this SLA on another VU"**: `list_sla_profiles` to find it →
  `attach_sla_profile_to_virtual_user` on the second VU. Do not duplicate the
  profile.
