---
name: octoperf-loadrunner-migration
description: Use when migrating a LoadRunner script or Controller scenario to OctoPerf. Triggers on "migrate from LoadRunner", "import my VuGen script", "convert a .usr script", "import an .lrs scenario", "we're leaving Micro Focus/OpenText LoadRunner", "can OctoPerf read a LoadRunner script". Decides first whether the script is worth migrating, imports what converts mechanically — including the load shape a .lrs describes — then finishes by hand what has no OctoPerf equivalent: hand-written C, conditions reading C locals, recorded cookies, parameterised servers, load generators, non-HTTP protocols. Requires the OctoPerf MCP server.
---

# OctoPerf — LoadRunner migration

`import_loadrunner_virtual_user` does the mechanical part: the requests —
`web_rest` included, whose headers are written inside the call rather
than before it — the transactions, the extractors and checks, the
headers, the think times, the `.prm` parameters with their `.dat` files uploaded, and the
parameters the script writes with `lr_save_string` and `lr_save_datetime`,
and the credentials `web_set_user` puts on a server. Given a provider it
also reads the `.lrs` — each Controller group becomes a user profile
carrying the load its schedule describes. It deliberately
stops short of what has no OctoPerf equivalent, and hands back everything
it left undone in the response's `unconverted` list, some of it as a
**named marker** planted in the tree. This skill is what turns that list
into a test that runs.

Work in this order. Skipping the first step is how migrations get
committed to and then abandoned.

## 1. Decide before you import

Call `analyze_loadrunner_project` and read `portability` before anything
else:

| Verdict | What it means | What to do |
|---|---|---|
| `MECHANICAL` | Plain HTTP throughout | Import; expect it to run nearly as-is |
| `ASSISTED` | HTTP plus C helpers or control flow | Import, then work the `unconverted` list |
| `MOSTLY_MANUAL` | The C outweighs the requests | **Say so before importing** |

`MOSTLY_MANUAL` is not a difficulty rating, it is a different script. A
VuGen `.c` is C: an author is free to build a whole framework in it, and
plenty have. In the reference corpus of 276 published scripts, 120 come
out `MOSTLY_MANUAL`, and they are not near misses — they are scripts
whose calls are `getenv`, `sprintf`, `strtok` and the author's own
`wi_*` helpers, branching on a `rc` returned by another of their own
functions. Nothing in that converts, because nothing in it is a request.

**A script can be `QTWeb` and send nothing at all.** Check `inventory`,
not the protocol: a script with 99 calls and no pages is a framework.
Importing it produces a tree of markers and no traffic, which reads as a
broken importer and is not one.

Then read `protocols`. A `TruClientWeb`, `DevWeb`, `Web2UI`, `SapGui`,
`RTE` or `RDP` script is reported rather than converted, and the import
creates no Virtual User for it at all — see `UNSUPPORTED_PROTOCOL`
below. Tell the user the count and let them decide; importing first and
discovering it later wastes their time and yours.

The per-script `scripts` breakdown matters as much as the verdict. An
upload can be `MOSTLY_MANUAL` overall while three of its six scripts are
plain HTTP. Migrate those and leave the rest.

`recordingLevel` on each script tells you what the tree will look like
before you see it. `HTML` means VuGen recorded user actions — one step
per click, resources implicit, and every URL recovered from the `data/`
snapshots. `HTTP` means it recorded requests — every asset its own step.
The same journey recorded both ways gives very different trees, and
neither is wrong.

Finally read `unconverted`, which the assessment carries too. Nothing
has been planted at this point, so the `marker` of every entry is empty;
the `kind`, the `script` and the `line` are what identify the piece.
`octoperf://schema/loadrunner-assessment` is the full vocabulary — every
kind, and the facts each carries.

## 2. Import

`import_loadrunner_virtual_user` takes the **whole script folder**
zipped. Not the `Action.c`.

`data/` is not optional. A script recorded at the HTML level carries no
URLs of its own: `web_link("Text=FI-SW-01", "Snapshot=t71.inf")` says
"click the thing labelled FI-SW-01 on whatever page you are on", and the
only record of which request that produced is `data/t71.inf`. Without
it, every click imports as a request against the entry page — the script
appears to convert and replays as one page fetched six times.

Zip the folder with its `.usr`, its `.c` sources, `default.cfg`,
`default.usp`, the `.prm` and the `.dat` files it names, and `data/`.
Leave out `result*/`, `data/http.db/` (VuGen's snapshot database — the
largest in the corpus holds 1630 files an import never reads) and the
replay logs. `pre_cci.c` and `combined_*.c` are dropped by the reader
whether you send them or not: the first declares every LoadRunner
function the script *could* call, so counting it turns nine real calls
into a hundred and five.

Several script folders in one archive are fine — each becomes its own
Virtual Users.

**Send the `.lrs` too, and pass a provider.** A Controller scenario
becomes an OctoPerf scenario only when `providerId` is given: each group
becomes a user profile carrying the load its schedule describes, and a
profile has to say where it runs. Without a provider the `.lrs` is read
for the assessment and no scenario is created — which is the right call
when you only want the traffic.

The provider is yours to choose. A `.lrs` names its load generators by
hostname, and a hostname says nothing about a region — see
`LOAD_GENERATOR`.

## 3. Work the `unconverted` list

`unconverted` is the whole of the manual work — the import merges into
it what conversion hit and what it never touches, so it is the only list
you need.

An entry is data, not prose. It carries a `kind` — which construct it
is — the `script` and the `action` it was in, the `line` of the `.c` it
sat on, and for some kinds a `facts` object holding the values the
importer had. **This skill is where the explanation lives**: find the
section named after the `kind` below and it tells you what happened and
what to do; the `facts` tell you with what.

The `line` is what makes an entry actionable. A customer holding a
two-thousand-line `Action.c` needs it, and the `script` alongside it —
a project of six scripts has six `Action.c` files.

`facts` is polymorphic and names its own shape in a `@type` property, so
read `@type` before the fields. An entry with no `facts` at all is a
kind whose section needs no values from the project: the section alone
is the instruction.

Some entries also name a `marker`, and a marker is a promise: a node of
**exactly** that name is in the tree. Find it with `get_virtual_user`,
rewrite it with `patch_virtual_user` against `octoperf://schema/vu`.

An empty `marker` is the ordinary case, and it says something too: the
work is not a step in a tree at all. A variable to fill, a server to
point at an environment, a parameter a `.prm` declares, an iteration
count that belongs to a user profile — searching a tree for those wastes
a pass over it. Work those from the `kind`, the `script` and the `line`.
The kinds that do plant one are `CUSTOM_C_CODE`,
`UNSUPPORTED_FUNCTION`, the protocol kinds, `CONVERSION_FAILED`,
`MISSING_SNAPSHOT`, `LOOP_BOUND_NOT_LITERAL`, `RENDEZVOUS_POLICY` and
`SCRIPT_SEQUENCE` — the last one naming a Virtual User rather than an
action.

Work from the list, not from the tree. A `[LoadRunner] …` name is not by
itself something to fix. If a name is not in `unconverted`, leave it.

Take them in this order; the later ones are cheap and the first is
sometimes a reason to stop.

### `UNSUPPORTED_PROTOCOL` — a different script, not a harder one

The `.usr` declares an `ActiveTypes` that is not HTTP. **The rule is a
one-name allowlist**: `QTWeb`, LoadRunner's token for HTTP/HTML, is
converted and every other token is reported here — including one nobody
has seen before, since a token that travelled silently would be a script
converted on a guess. `function` holds whichever it was. The corpus's are
`TruClientWeb`, `TruClient2Web`, `TC4Mobile`, `Web2UI`, `Web2UIMob`,
`DevWeb`, `RTE`, `RDP`, `SapGui`, `General-Java`, `WS_SOAP` and
`SAP_Web`, but do not read that as the list to match against.

**No Virtual User is created for that script.** The importer does not
enumerate its calls either: fifteen TruClient scripts in the corpus
would have contributed 372 `truclient_step` entries, and a report of 372
lines that all say the same thing hides the ones that matter.

There is nothing to translate. A TruClient or Web2UI script drives a
real browser through recorded UI steps; a DevWeb one is JavaScript. The
traffic has to be recorded again against the live application — with
OctoPerf's own recorder, or as a HAR imported through
`import_har_virtual_user`.

An `ActiveTypes` the assessment reports as empty is not this. A script
declaring no protocol is walked and converted on the strength of its
calls, because an absent `ActiveTypes` says nothing about what a script
does. All seven such scripts in the corpus turned out to be UFT or
protobuf with no `web_` call at all — which is what walking them
established.

### `SOCKET_PROTOCOL`, `SAP_PROTOCOL`, `RDP_PROTOCOL`, `TERMINAL_PROTOCOL` — the same, one call at a time

A script that declared a convertible protocol, or none, and then calls
`lrs_*` (WinSock), `sapgui_*`, `rdp_*`, `ctrx_*` (Citrix) or `te_*`
(terminal emulation). That is a mixed script: its HTTP half converts and
this half does not.

`function` names the call and `line` places it. Decide per script
whether the HTTP half is a test on its own. A journey that logs in over
SAP GUI and then browses over HTTP is not — the browsing depends on a
session the converted half cannot open.

### `CUSTOM_C_CODE` — the C is the script, not decoration

A call whose name carries none of LoadRunner's own prefixes. Either the
C library — `sprintf` is 108 call sites in the corpus, beside `malloc`,
`strtok`, `atoi`, `fopen` — or a helper the author wrote:
`wi_start_transaction`, `mystrcat`, `login`, `get_google_access_token`.

494 entries on the corpus, and they are the difference between a
recording and a framework. Read them together rather than one at a
time, and ask what the C was **for**:

- **It computed a value later sent.** `sprintf(url, "%s/item/%d", base,
  i)` feeding a request. Rebuild the value as an OctoPerf variable, or
  as a `JSR223Action` if it needs arithmetic. The request itself will
  already be in the tree, pointing at whatever literal the recording
  held.
- **It wrapped web calls.** `wi_generic_html_get` and its like are the
  author's own request functions. Their bodies are in the script's `.h`
  or in an action the `.usr` does not list, so the importer never saw
  them and no request came out. Find the body, read what it sends, and
  build the request by hand — or re-record the journey, which is usually
  faster.
- **It logged, timed or counted.** `lr_output_message` and its family
  are dropped silently and never appear here. What does appear —
  `lr_user_data_point`, `wi_startPrintingInfo` — is instrumentation, and
  OctoPerf measures its own. Delete the marker.
- **It paused.** `sleep(...)`, 7 sites on the corpus and not one of them
  a plain literal — the arguments are C locals, so there is no duration
  to carry over. A Delay action is the replacement and you choose the
  number. Two things to know before you do: `sleep` takes
  **milliseconds on a Windows load generator and seconds on a Linux
  one**, and the archive never says which the script was written for, so
  a mechanical conversion would be a factor of 1000 decided by a guess.
  And unlike `lr_think_time` it is not think time — `[ThinkTime]
  Options=NOTHINK` does not switch it off, which is usually why the
  author reached for it.

A script whose `unconverted` is all `CUSTOM_C_CODE` and whose tree holds
no request is the framework case from step 1. Say so rather than
patching sixty markers.

### `UNSUPPORTED_FUNCTION` — read the function, most are one line

A real `web_*` or `lr_*` function with no converter yet. 50 distinct
functions across the corpus; these are the ones you will actually meet:

| Function | What it did | What to do |
|---|---|---|
| `lr_user_data_point` | A custom metric | Delete; OctoPerf measures its own |
| `web_js_run` | Ran JavaScript in the page | Not HTTP. Either the value it computed is correlatable, or the journey needs re-recording |
| `lr_get_attrib_string` | Read a command-line attribute | A variable holding what the runtime passed |
| `web_reg_async_attributes`, `web_stop_async` | Long-polling or server push | Neither is request/response. Model the polling with a Loop container, or drop it |
| `web_websocket_connect`, `web_websocket_send` | A WebSocket session | OctoPerf has WebSocket support; rebuild the session by hand |
| `web_stream_open`, `web_stream_play`, `web_stream_close` | Played a media stream | Read `Protocol=` on the open. `HTTP` is a progressive download and rebuilds mechanically: one GET on its `URL`, then a Delay of `PlayingDuration` seconds so the journey keeps its tempo. Close and stop convert to nothing. The GET pulls the whole media at once where the player spread it across the play — same bytes, different rate. Any other protocol is not HTTP; leave it reported |
| `lr_mysql_*`, `lr_xml_get_values` | A database query, XML reading | Not traffic. Usually setup that belongs outside the test |
| `lrvtc_*`, `vtc_*` | LoadRunner's Virtual Table Server | A shared value store. A shared CSV variable is the nearest thing; a counter if it was minting ids |

Anything not in that table: read the function name, look it up if you do
not know it, and ask whether it produced traffic. If it did not, delete
the marker.

### `THINK_TIME_NOT_LITERAL` — the pause is gone, and it was real

`lr_think_time(RThinkTime)`. The duration is a C variable, so there is
no number to convert and none is invented: an invented pause lowers the
throughput the test was written to produce. No Delay is built, and the
entry is here because the silence cuts the other way — the converted
user runs faster than the script did.

`facts` is a `LoadRunnerThinkTimeFacts` holding the duration as the
script wrote it. That text is where to start: find where the variable
was filled — a runtime attribute, a `rand()`, a value read from a file —
and decide what the test was meant to wait for. Then add a Delay with
that number.

**Check `default.cfg` before you do.** `[ThinkTime]` states whether think
time runs at all and by what factor, and both apply to what you add:
`Factor=1.5` on a two second pause is three seconds. A script with
`Options=NOTHINK` never reports this kind — there the configuration
removes every pause, the literal ones included, and adding one back
would be a test the customer never ran.

7 sites on the corpus, against 725 think times that name a number.

### `CONDITION_NOT_SUPPORTED` — the guard is gone, the branch is not

The construct converted and its condition did not. **The steps are in
the tree**: an `if` whose guard could not be read becomes an If
container on `true` with the `else` beside it on `false`, so the branch
the script guarded still fires and the alternative does not. A `while`
gets a blank condition, which ends the loop on its first failing request
rather than never.

A chain of `else if` stacks rather than flattens: each `if` after the
first lands inside the preceding `else`, so it sits under a container on
`false` and cannot fire whatever its own guard says. **Only the first
branch of a chain runs.** Re-expressing one condition is not enough
there — the cascade has to be rebuilt as a whole, and until it is, the
branches below the first are unreachable rather than merely unguarded.
91 `else if` across the corpus, in 10 files.

`facts` is a `LoadRunnerConditionFacts` holding the raw C.

Three C shapes convert, and all three read a LoadRunner parameter:

```c
strcmp(lr_eval_string("{status}"), "OK") == 0
atoi(lr_eval_string("{count}")) >= 1
strstr(lr_eval_string("{body}"), "token") != NULL
```

Everything else reads a C local, and that is why it cannot convert
rather than a gap to be filled. The corpus's four most common
unconverted guards say it plainly:

- `rc != LR_PASS` (48 sites) — `rc` is never set from a `web_*` call.
  It comes from the author's own functions: `wi_end_transaction`,
  `WT3_SignIn`. It means "my helper failed", and nothing on the OctoPerf
  side stands for that.
- `iVerbosity >= 2` (43) — a debug switch, set from a constant or a
  command-line parameter. Decide whether the debug branch should run at
  all; usually it should not, so delete that container.
- `stricmp("X", LPCSTR_RunType) == FOUND` (28) — a run-mode switch on a
  C local read at startup. If the value comes from a parameter, rewrite
  the condition against the variable; if it comes from the command line,
  decide the mode now and delete the branch that does not run.
- `strcmp(Day, "Monday") == 0` — a local filled earlier in the C.

For each entry, decide which branch the test should take, then either
delete the container that must not run, or give the surviving one a real
condition with `patch_virtual_user`. Leaving both at `true`/`false` runs
one path forever, which is a decision by default rather than a bug — but
make it deliberately.

### `LOOP_BOUND_NOT_LITERAL` — decide the count

A `for` whose iteration count is not a literal. The body is in a plain
container that runs **once**, and `function` holds the loop header —
`i = 0; i < iCount; i++`.

Inventing a number would silently change how much load the script
generates, which is why it was not. Read the header, find what bounds
it, and either put the count on a Loop container or accept the single
pass. A loop bounded by a retry counter usually should run once.

52 of the corpus's 83 `for` loops land here, so this is common and
usually cheap.

### `CONTROL_FLOW` — a fall-through to widen

A `switch` case that falls through carrying a body of its own. The case
before it would need its body copied into every case beneath, which the
importer does not do.

Rare: one `switch` in 276 corpus scripts. `function` names the case.
Either copy the body down, or widen the guard of the case below to cover
both labels — the guards are plain expressions on the switch subject and
`patch_virtual_user` can rewrite them.

An empty fall-through — `case 1: case 2:` — is already folded into one
guard and never appears here.

### `PARAMETERISED_SERVER` — one server edit fixes the whole script

The server a request points at was written with parameters, and not all
of them survived. `facts` is a `LoadRunnerServerFacts` with the recorded
`url` and the `scheme` and `port` the import had to fill in.

Two causes, both common:

- **The whole origin was one parameter.** `"URL={URL_BASE}/actions/Catalog.action"`
  — 602 of the corpus's 5013 recorded URLs. There is no scheme or host
  to read, so the origin was recovered from what the parameter replaced:
  VuGen keeps `OriginalValue`, the literal the parameter stood in for.
  The request now points at the environment the script was **recorded
  against**, where the script pointed at whichever one it was given.
- **The scheme or the port was its own parameter.**
  `"URL={SCHEME}://{HOST}:{PORT}/catalog"` — five sites. A parameterised
  *host* converts faithfully, because a server hostname is substituted
  at replay; a scheme and a port cannot, since an OctoPerf server holds
  one of two protocols and a number.

Either way the fix is one edit, not one per request. Find the server
with `list_http_servers_by_project`, point it at the environment under
test with `update_http_server`, and every request on it follows. Check
`scheme` in the facts against what you expect — HTTPS is the guess, and
a plain-text service will refuse it.

### `RECORDED_COOKIE` — add back the one the application needs

`web_add_cookie` is the third most frequent call of the corpus — 1489
sites — and what it carries is a browser's jar as it stood on the day of
the recording. `function` holds the cookie's name.

None of them is replayed, and that is deliberate. The corpus's cookies
are `userCart` holding a JSON document beside `_ga`, `_gid`,
`AMCV_…@AdobeOrg` and an `mbox=session#…` stamped 2021. Handing every
virtual user of a five-hundred-user test the same analytics identity and
the same expired session is not the load the test claims to produce, so
the virtual user starts with the empty jar OctoPerf gives a new visitor.

One exception, already handled: a cookie whose **value** carries a
parameter — `web_add_cookie("JSESSIONID={jsessionid}")` — is a value the
script correlated out of a response, so it is state the script computes
rather than state it remembers, and it is sent. Those do not appear
here. A parameterised `DOMAIN` does not count: that says where the
cookie applies, not that its value was correlated.

So: for each name, decide whether the application needs it.

- **A session cookie the server sets itself** — `JSESSIONID`,
  `ASP.NET_SessionId`, `PHPSESSID` — needs nothing. OctoPerf's cookie
  manager receives it from the response and stores it, which is what a
  real browser does.
- **A consent, locale or feature flag** — `cookieConsent`, `lang`,
  `ab_variant` — needs adding back, as a `Cookie` header on the first
  request. Take the value from the entry's line in the `.c`.
- **Third-party analytics** — `_ga`, `_gid`, `_gat`, `mbox`, `ak_bmsc` —
  leave out. Replaying an Akamai bot-management token from 500 users is
  worse than not sending it.

`web_cleanup_cookies` is in neither the list nor the tree, and that is
correct rather than a gap: JMeter manages its own cookie store and a
virtual user starts each iteration with an empty jar. A script that
called it deliberately already has what it asked for.

### `MISSING_SNAPSHOT` — the click has no URL to recover

A `web_link` or `web_image` whose `Snapshot=` names a `data/tNN.inf` the
archive does not hold. The step is a named marker rather than a request,
because there is genuinely nothing to fetch.

Almost always a packaging mistake: re-zip the folder **with** its
`data/`. If the snapshots really are gone, find what the click did in
the application and build the request by hand — `function` names the
function and `line` places the call, whose `Text=` or `Src=` says what
was clicked.

### `MISSING_DATA_FILE` — upload it

A `.dat` a `.prm` names, which the archive did not carry or the storage
would not take. `facts` is an `ImportFileFacts` with the filename.

The CSV variable was created either way, so every reference to it
resolves the moment the file exists. Upload it with
`upload_project_file` under **exactly** that name.

Two of the corpus's 314 tables name an absolute Windows path in
`TableLocation` — a shared table living outside the script folder. Those
were never in the archive and never will be; get the file from whoever
ran the test.

### `MISSING_SCRIPT` — the scenario arrived without its scripts

A `.lrs` group points at its script by absolute Windows path —
`C:\\Users\\…\\aos-web-html\\aos-web-html.usr` — so an upload holding the
scenario alone carries none of them. `function` names the script and
`facts` is an `ImportFileFacts` holding the same name.

**This is why an import can come out empty.** A group whose script is not
in the archive has nothing to run and makes no user profile; a scenario
whose every group is in that state makes no scenario. Without this entry
the result is a project with no Virtual User, no scenario, and an
`unconverted` list that says nothing — which reads as a conversion that
found nothing to do.

Named once per script however many groups run it, since uploading it
settles all of them: four groups running one journey from four cities is
one file to find.

What to do: zip the Controller folder **and** the script folders together
and import again. The scripts are usually beside the `.lrs` on the
machine that ran the test — 33 of the corpus's 36 scenarios have theirs
in the same repository, even though the path written in the file points
somewhere else entirely.

### `PARAMETER_TYPE` — the parameter has nothing to become

A `.prm` parameter with no OctoPerf variable to map onto. `function`
holds its name. Four types and one function reach here:

| Type | What to do |
|---|---|
| `Time`, `Date` | A `JSR223Action` writing the format you need, or a constant if the test does not care |
| `Userid` | JMeter's own thread number. A `JSR223Action` doing `vars.put("id", ctx.getThreadNum())` |
| `Group` | The user profile's name. Usually a constant per profile |
| `Output` | A parameter a LoadRunner function filled in. Find which, and correlate it |

A `SaveCount` on a `web_reg_find` also lands here: the check converted,
but OctoPerf states whether a pattern matched and not how many times.
If the count mattered, a regexp extractor with match number `-1` puts
the count in `${name_matchNr}`.

A `Custom` parameter appears here only when it held nothing. One with a
value became a constant variable, which is what makes the
`{SCHEME}`/`{URL}`/`{PORT}` scripts work.

### `SCRIPT_SEQUENCE` — three trees, one script

Not a defect and nothing to repair. A LoadRunner script runs three
sequences and an OctoPerf Virtual User is only the repeated one, so
`vuser_init` and `vuser_end` each became a Virtual User of their own,
named `<script> - vuser_init` and `<script> - vuser_end`.

It is reported because nothing else says so: without this entry the
three trees look like three unrelated Virtual Users.

**If you imported with a provider, this is already done.** The scenario
conversion attaches `vuser_init` as each profile's **Set Up** and
`vuser_end` as its **Tear Down**, so the two extra Virtual Users are
wired rather than loose. Read the entry to know they exist and check the
session below; there is nothing to move.

Without a provider there is no profile to attach them to. Put the init on
the user profile's **Set Up** and the end on its **Tear Down** yourself —
`octoperf-scenario-composition` covers how. Keeping the init inside the
repeated body instead would sign in again on every iteration, which is a
different test.

Then check the session. A Set Up runs in a thread group of its own, so
cookies and variables it acquires do not reach the iterations. If
`vuser_init` signed in and the actions depend on that session, either
move the sign-in into the repeated body or correlate the session
properly — `octoperf-auto-correlation`.

### `ITERATION_COUNT` — two counts, pick one

`default.cfg` and `default.usp` disagree about how many times the
actions run. `facts` is a `LoadRunnerIterationFacts` with both numbers:
`runLogic` from the `.usp`, `settings` from the `.cfg`.

VuGen keeps them in step; a hand-edited script does not have to.
Resolving it would be picking one at random, so both are handed over.
Ask which the test ran with, then set it on the user profile of the
scenario — not on the Virtual User, which has no iteration count of its
own.

Usually you can leave it alone. A Controller schedule that states a
duration keeps its Vusers iterating until the duration elapses whatever
the script's count says, so an imported profile is bounded by its
plateau and not by iterations. The count matters when the schedule runs
until the iterations finish instead — see `SCHEDULE_DURATION`.

An upload with no scenario has nowhere to put the number at all, which
is the other reason to read this entry.

### `RENDEZVOUS_POLICY` — the barrier is there, confirm the numbers

`lr_rendezvous` converted: a rendezvous action is in the tree under the
name the script gave it. What is **not** in the script is how many users
it waits for and for how long — those live in the Controller.

So a policy was chosen: every user, released after thirty seconds,
across the whole test. `facts` is an `ImportRendezVousFacts` stating
exactly that. Ask whoever configured the Controller and correct it if
those are not the numbers, with `patch_virtual_user`.

A rendezvous also needs enough concurrent users to be reachable. A
barrier waiting for 100% of users in a scenario ramping up one user at a
time releases only on the timeout, which is a test that measures the
timeout.

### `SCENARIO_TYPE` — the scenario states its size in a third way

Two ways are read. One enumerates its Vusers, so a group of ten is ten
entries under `{GroupChief}` and the count is how many there are. The
other states a percentage per group, and the total comes from the
schedule. 34 of the 36 corpus scenarios are the first, 2 are the second.

Anything else lands here, and a goal-oriented scenario is the case to
expect: it has no Vuser count at all, adjusting one at run time until it
reaches a target hit rate or response time. OctoPerf has no load shape
that chases a target, so there is nothing to convert and nothing was
guessed.

Ask what the scenario was aiming for, then express it as a load a
Vuser count can produce: read the last report the customer has, take the
throughput the target implies, and pick a plateau that reaches it. Say
plainly that this is a translation and not a copy — a goal-oriented run
and a fixed-Vuser run do not fail the same way.

### `SCHEDULED_START` — the run was held back

The Controller waits a delay, or a wall-clock date and time, before its
first Vuser starts. The imported scenario starts as soon as it is run.

Nothing to repair in the scenario. If the delay was there to line the run
up with something — a batch window, a nightly job, another team's test —
reproduce it with `schedule_scenario_once` or
`schedule_scenario_cron` instead, where a start time belongs.

### `SCHEDULE_DURATION` — it ran until the iterations finished

The schedule states no duration: it runs until every Vuser has played
the iterations its script asks for, and stops. How long that takes is a
property of the target, not of the scenario, so there is no number to
convert. The profile ramps up and holds for no time.

Two things to set, both on the user profile:

- the **iteration count**, from the script's `default.usp`
  (`RunLogicNumOfIterations`) — see `ITERATION_COUNT`;
- a **plateau long enough** that the iterations finish inside it. Take
  the iteration count times the slowest iteration the customer has
  measured, and add margin. Too short truncates the test; too long is
  harmless, because the profile stops when its iterations are done.

If nobody knows how long an iteration takes, run one Vuser for one
iteration with `validate_virtual_user`, read the duration, and multiply.

### `LOAD_GENERATOR` — the load came from somewhere named

A group runs on a machine the scenario names rather than on the
Controller's own. The name is in `function`.

This is the one thing an import cannot carry over. An OctoPerf scenario
chooses a provider and a region; a hostname says nothing about either,
and `lg-paris-01` may or may not be in Paris. Every group of the
reference corpus runs on `localhost`, and two of its scenarios declare
generators beside it — so what you are looking at is the customer's own
topology.

Ask where each named generator actually sits, then pick the matching
region per user profile with `update_scenario`. `list_public_docker_providers`
and `list_docker_providers_by_workspace` give what is available. If the
generators were all in one place, one region for the whole scenario is
the faithful answer.

### `ITERATION_PACING` — the pace is stated the other way round

`facts` is a `LoadRunnerPacingFacts` with the `type` VuGen wrote and its
bounds in seconds.

An OctoPerf pacing is a **minimum iteration duration**: "a new iteration
every 60 seconds", counted from the previous start. That is what
LoadRunner's `Const` pace means too, and it crosses over untouched.

`After` does not. "60 seconds **after** the previous iteration ends"
adds 60 to whatever the iteration took, so under load it produces fewer
iterations per hour than a 60-second interval does — and the gap widens
exactly when the target slows down, which is when the measurement
matters. Converting it as a pacing would run the test slower than the
customer asked for and say nothing about it, so it was left out.

Two ways to settle it, and the choice belongs to whoever owns the test:

- **Keep the think-time shape**: add a delay of that many seconds as the
  last action of the Virtual User's repeated body. Exact, and the pause
  shows up in the tree where a reader will find it.
- **Keep the rate**: set a pacing of the pause plus the iteration
  duration the customer measures. Simpler, and it holds the throughput
  steady rather than the pause.

A `type` that is neither `Asap`, `Const` nor `After` is a pace this
importer has no sample of. VuGen offers random variants and writes their
bounds — `RunLogicRandomPaceMin`, `RunLogicAfterPaceMax` — in every file
whatever the choice, so the bounds in `facts` may be defaults rather than
the numbers in force. Open `default.usp` and read the block before
setting anything.

### `SAVED_PARAMETER` — the parameter is there, the value is not

`lr_save_string` writes a value into a LoadRunner parameter, and it is
the most used function of the reference corpus — 319 sites.
`lr_param_sprintf` writes one too, out of a printf format and its
arguments. Most of them now convert on their own: a parameter written
once with a value the script holds becomes a **constant variable**, and
every reference to it was already being rewritten, so nothing is left to
do. This entry is the rest.

`facts` is a `LoadRunnerSaveFacts` with the parameter name and **every
value the script writes to it**. That list is the whole of what you need,
and its being empty is itself information.

Three cases, told apart by that list.

**Several values.** The script assigns the parameter at different points
— the commonest shape is a transaction name set before each step, one
corpus script writing twenty-one different values into `pTransName`. A
constant standing for one of the twenty-one would run and look right,
which is why there is none.

Set it where the script sets it, with a JSR223 **pre-processor** on the
request that reads it:

```groovy
vars.put("pTransName", "WT3_T07_SignIn")
```

`vars` is the virtual user's own variables, so each user gets its own
value — which is what a LoadRunner parameter is. Do **not** reach for
`props`: those are global to the whole test, and one user's assignment
would land on all of them. The
[JSR223 samples](https://api.octoperf.com/doc/design/edit-virtual-user/action-types/jsr223-actions/jsr223-samples/)
page has the shapes for reading, incrementing and replacing a variable.

Two things to watch. A value carrying a quote or a backslash needs
escaping for Groovy — the corpus holds literals with embedded `\"` — and
a JSR223 node per assignment is a node a reader has to scroll past, so if
the same value is set twenty-one times in a row, ask whether the
parameter was doing any work at all.

**One value, and it is empty.** The value came from somewhere the import
does not follow: a C local, a `sprintf` buffer, a database cell read by
`lrd_*`. 86 of the corpus sites are these. The value is genuinely not in
the script, so read the surrounding C — the `CUSTOM_C_CODE` entry for the
same lines usually explains it — and decide what the parameter should
hold.

**A format the import cannot fill.** A `lr_param_sprintf` whose value is
in the script converts like any other write — `"%s %s"` over two
`lr_eval_string` arguments is the text between the conversions with the
parameters in place. Two shapes land here instead. A **numeric
conversion** (`%d`, `%02d`, `%.02x`) formats something the script
computes at replay: a counter, a digest byte, a `time(0)`. And a format
**reading the parameter it writes** — `lr_param_sprintf("Login", "%s%s",
lr_eval_string("{Login}"), …)` — is the script appending to itself, which
no constant can do. For the second, a JSR223 pre-processor reading the
old value and putting the new one back is the faithful answer:

```groovy
vars.put("Login", vars.get("Login") + vars.get("RandomLetter"))
```

One shape lands here and reads oddly: a name `lr_save_datetime` stamps
with **several different formats**, as one corpus script does with
`Today` — `%A`, then `%B %d %Y`, then `%j`. It is reported as a
reassignment rather than as a date because that is what it is: the
parameter holds a different string at different points. Read the formats
out of the script and treat each point separately, using
`DATE_PARAMETER`'s table of what a strftime format becomes.

**A name the `.prm` already declares.** Left to the `.prm` on purpose:
its declaration may read a data file of a thousand rows, and replacing
that with the one value a save happens to write would drop the file
without saying so. Decide which the test meant. If the save was
overriding the file for one journey, a JSR223 assignment on that journey
is the faithful answer; if the file is dead, delete the variable and keep
the constant.

### `STEP_NAME_PARAMETER` — every transaction has the same name

`lr_start_transaction(lr_eval_string("{pTransName}"))` names its
transaction through a parameter. A container's name is fixed when the
test is built, not when it runs, so **no run-time mechanism can rename
it** — a JSR223 assignment will not help here, and neither will
anything else. Every transaction converted from that call carries the
same name, and a report adds up business steps that have nothing to do
with each other.

12 of the corpus's 1880 transaction sites. One resolved by itself: a
parameter written with a single value everywhere became that value. The
rest kept the parameter's name, which at least reads as what it is, and
`facts` carries the values the script writes.

Rename the containers from that list. Order matters and the list is in
the order the script writes them, which is usually the order of the
journey — but check against the requests inside each container before
committing, because 10 of those 12 write the parameter in one file and
open the transaction in another, and the sequence is the caller's.

If the list is empty, the names are not in the script at all. Read the
requests in each container and name it after what it does; a transaction
called `pTransName` in a report is worse than one called
`Search flights`.

### `SLA_RULE` — the thresholds are there, attach them

The scenario was given service-level rules. 4 of the corpus's 36 were;
the other 32 carry the SLA document and nothing in it, so an entry here
means somebody actually set one.

**An entry with `facts`** is the rules that converted. `facts` is a
`LoadRunnerSlaFacts` listing the **SLA profiles created**, each named
after the transaction its rules watch — `Full-Process`, `Trx01-URL`. They
exist in the project and are **attached to nothing**, on purpose: a
profile is checked wherever it sits in the tree, and which container that
is depends on how the script was converted. Attach each profile to the
container of the same name. If the container is called something else
because the transaction name did not survive — see `STEP_NAME_PARAMETER`
— settle that first, or you will attach the checkout's threshold to the
search.

What the conversion did with the numbers, so you can check it: LoadRunner
states an SLA ceiling in **seconds** and OctoPerf holds milliseconds, so
13 becomes 13000, and the alarm is the range **above** it. Severity is
critical, because a LoadRunner SLA passes or fails and a warning would
soften a verdict the customer set. The condition is one occurrence: a
single measurement past the ceiling fails it.

**An entry with no `facts`** is a rule that did not convert, named by the
measurement it asked for. Three reasons:

- a **measurement** with no OctoPerf metric behind it — the throughput
  ones are the case to expect, and guessing a unit would arm the alarm by
  a factor nobody would notice;
- a **percentile** we do not compute. 80, 90, 95 and 99 convert; a
  scenario asking for a fiftieth is reported rather than served the
  ninetieth, because the number is the whole point of the rule;
- a **ceiling per range of running Vusers**. LoadRunner's complex-load
  rules let a threshold change as the load climbs, and an OctoPerf
  threshold does not vary with load. Pick the band that matters — usually
  the one covering the load the test actually reaches — and set it with
  `create_sla_monitor`.

### `WAN_EMULATION` — one of the four numbers crossed over

The Controller put an emulated network between a group and the server.
Every `.lrs` carries LoadRunner's whole library of locations — some
thirty, from `3G Busy` to `China to US EC` — so an entry here means a
group actually sat on one and the scenario had emulation switched on.

`facts` is a `LoadRunnerWanFacts` with the location's name, the group,
and all four numbers.

**What converted**: `downKbps`. The user profile is throttled to it — the
generated test limits JMeter's socket read rate — and the profile carries
the location's name, so it reads `3G Busy` rather than a number somebody
has to recognise.

**What did not**: the upload rate, the latency and the packet loss. There
is nothing on a profile to delay a packet by or to drop one with, and
folding them into the bandwidth would slow the test by an amount nobody
asked for — 200 milliseconds of latency is not a bandwidth.

So the question to put to whoever owns the test is whether a download
limit alone still measures what they were measuring. Two common answers:

- **Latency was the point.** A journey of forty small requests over a
  200 ms link spends eight seconds on round trips and almost nothing on
  bandwidth. Throttling the bytes reproduces none of that. Say so, and
  either accept that the test now measures the server rather than the
  link, or move the latency into the think times — which is a different
  test and should be labelled as one.
- **Bandwidth was the point.** A journey downloading images and bundles
  over 384 kbps is shaped by the rate, and the conversion is faithful
  enough. Check the page-asset behaviour while you are there — see the
  `downloadResources` note above — because a converted page may fetch a
  different set of assets and the rate limit makes that visible.

Packet loss has no equivalent at all and no approximation worth
proposing. If the test existed to measure behaviour under loss, say
plainly that the migration does not carry it.

### `DATE_PARAMETER` — the date converted, read what it became

`lr_save_datetime` fills a parameter with a formatted date, and most of them
convert on their own: strftime and a Java date pattern name the same
fields, and "now" is a JMeter function. `"%Y-%m-%d"` with `DATE_NOW`
becomes a constant variable holding `${__time(yyyy-MM-dd,)}`, and
`DATE_NOW + ONE_DAY` becomes `${__timeShift(MMMM dd yyyy,,P1D,,)}`. 16 of
the corpus's 44 calls cross over with nothing left to do and produce no
entry at all.

This is the rest, and `facts` — a `LoadRunnerDateFacts` — tells the three
cases apart by which of its fields are filled.

**`expression` filled, and the format holds a time of day.** It did
convert; the entry is here because the two tools compute it at different
moments. VuGen worked the value out at each call, so a script stamping
`"%H:%M:%S"` inside its Action got a fresh clock on every iteration.
OctoPerf's constant variable is a JMeter *user-defined variable*: the
function in it is evaluated once per virtual user, so iteration 2 reuses
iteration 1's time.

Whether that matters is a question about the test, not about the
conversion. If the stamp was decoration — a display field, a log line —
leave it. If the target keys on it, or rejects a replayed timestamp, move
it to a JSR223 **pre-processor** on the request that reads it:

```groovy
vars.put("sTime", new java.text.SimpleDateFormat("HH:mm:ss").format(new Date()))
```

The [JSR223 samples](https://api.octoperf.com/doc/design/edit-virtual-user/action-types/jsr223-actions/jsr223-samples/)
page has the shapes. A date — `%Y-%m-%d`, `%A`, `%B %d %Y` — never needs
this: it is the same all run.

**`pattern` filled and `expression` empty.** The format translated and the
offset did not: the script works it out at replay, as in
`DATE_NOW + ONE_DAY * atoi(lr_eval_string("{p_DaysFuture}"))`. 14 of the
corpus's calls are these, and they are the interesting ones — a date N
days out where N is itself a parameter. That is expressible:

```
${__timeShift(yyyy-MM-dd,,P${p_DaysFuture}D,,)}
```

Read the C to find what the offset really is, then write the shift. Watch
for two parameters added together — `__timeShift` takes one duration, so a
sum needs a JSR223 or a single variable holding the total.

**Both empty.** The format holds a token no date pattern states. `%X` is
the locale's own idea of a time and `%c` its idea of a whole date — there
is no pattern that means "whatever this machine calls a time". Decide what
the target actually expects and write a fixed pattern for it.

### `TRANSPORT_OPTION` — the script configured the handshake

`web_set_sockets_option` sets things about the connection rather than
about a request. Three options across the corpus's 343 call sites, and
seven of every ten ask for what a converted virtual user already does —
so they are not reported at all and you will not see them here.

`facts` is a `LoadRunnerTransportFacts` with the option and its value.
What to do depends entirely on which:

**`SSL_VERSION` set to a version.** The script pins TLS 1.1, 1.2 or
"TLS". A converted virtual user negotiates, which is right far more often
than a pin — a pin usually outlived the reason it was added. Try without
it first. If the target genuinely refuses to negotiate, set
`https.default.protocol` in the user profile's JMeter properties;
`SSL_VERSION=AUTO` is the value that needed nothing and never appears in
a report.

**`TLS_SNI` set to `0`.** The script asks the load generator *not* to
send the server name during the handshake. Java has sent it by default
for years and OctoPerf does not switch it off, so a converted script
sends it. This matters only against a server that presents a different
certificate when it sees a name — check the certificate the target
returns before assuming it is harmless. `TLS_SNI=1` is the other value,
and it needed nothing.

**`INITIAL_AUTH` set to `BASIC`.** The script sends its credentials
without waiting for the challenge. OctoPerf sends a server authorization
the same way, so there is nothing to add for the *timing* — what carries
the credentials is `web_set_user` in the same script, which the import
converts on its own. If that script reports a `SERVER_AUTHORIZATION`,
read the next section; if it does not, the authorization is already on
the server and this entry needs nothing beyond checking that it is.

### `SERVER_AUTHORIZATION` — the credentials are on the server, minus one piece

`web_set_user("user", password, "host:port")` becomes an **authorization
on the HTTP server** rather than an action, because in OctoPerf
credentials belong to the server and not to a request. The import builds
it and reports what it could not finish.

`facts` is a `LoadRunnerAuthorizationFacts` with the `username`, the
`domain` (empty unless the script wrote `domain\user`, which is the NTLM
form and imports as an NTLM authorization), the `target` exactly as the
call wrote it, and the `server` the import resolved — **empty when it
could not resolve one**. Two things send an entry here, and the `server`
field tells you which:

**A password the script masked.** `lr_unmask("60907177e7c411aa6aea54")`
is VuGen's own obfuscation, not an encoding, and the value cannot be read
back outside LoadRunner — 16 of the corpus's 17 `web_set_user` calls use
it. The server, the username and the domain are all in clear and are
already set; only the password is empty. Ask whoever owns the test for
it, then type it into the authorization on the server named in `server`.
Put it in a **secret variable** if the project has one and reference it,
rather than in clear on the server.

**A target the script parameterised.** `"{URL}"` or
`"{host_nimbusserver_aos_com_8002}"` names a host through a parameter,
so no server could be built and `server` is empty — the authorization
was not created at all. Resolve the parameter to the host it stands for
(the `.prm` file, or the `PARAMETERISED_SERVER` entries for the same
script), find or create that HTTP server, and add the authorization to
it by hand with the username and domain from `facts`.

### `CONVERSION_FAILED` — the importer broke on this one

A converter threw. `facts` is an `ImportFailureFacts` with the
exception, and the call is a named marker in the tree so nothing else
was lost.

A VuGen body is C: a hand-edited argument list, a macro that expanded to
something unexpected. Read the `.c` at `line`, build the request by hand
if it was one, and **report it** — a `CONVERSION_FAILED` is a gap on our
side, not in the script.

## 4. Check what came through, not just what did not

These convert silently and are worth verifying before you hand the
project over. Nothing reports them, so nothing will remind you.

**Servers the browser talked to, not the application.** A recording made
in an everyday browser carries the traffic that browser sent for itself:
extension updates, profile sync, safebrowsing, captive-portal checks.
They convert like any other request, so a converted user sends them for
real — from every load generator, for the whole run. `web-login-logout`
imports with five servers and one of them is the application: the other
four are Chrome. Across the corpus, 678 URLs reach 28 such hosts in 47 of
its 222 script folders, led by `detectportal.firefox.com` at 198 and
`firefox.settings.services.mozilla.com` at 166.

So read the project's **server list before the action tree**: a hostname
nobody in the room recognises is one of these. Then work in that order —
**the requests first, the servers after**. `delete_unused_http_servers`
only removes a server nothing points at, so running it on a freshly
imported project deletes nothing at all: every one of those hosts is held
by the request that recorded it. In `web-login-logout` each of the four
is held by exactly one. Delete those four requests and the same call then
takes all four servers.

Look hardest at the ones sitting *inside* a transaction. This script
fetches a Chrome plugin manifest inside `Username` and posts a Chrome sync
status inside `Login`, where they inflate response times somebody will
later read as the application's.

**The load shape.** A Controller schedule is an ordered list of
actions, and the profile it became is a reading of them rather than a
copy of a named shape. One start, one hold and a way down became a
ramp-up: `Start 50 Vusers, 5 every 10 seconds` is ten batches, so the
ramp is 100 seconds and not the 50 you get by multiplying the wrong pair.
A schedule that starts Vusers **more than once** — five `Start 10` in a
row — became a drawn curve instead, because nothing in a Controller
schedule makes the steps equal and a named staircase would need them to
be. Open the profile and check the curve climbs where you expect.

**Which schedule was read.** A `.lrs` always carries both a global
schedule and one per group, and the per-group ones go stale: 65 of the
corpus's 92 state a Vuser count the group does not have, a group of ten
routinely claiming one. The import reads the global schedule when the
scenario has one and the per-group schedules only otherwise. If the load
looks a tenth of what you expected, that is the mistake to rule out — but
rule it out in the `.lrs`, not in the imported profile.

**How the Vusers were split.** Under a global schedule every group ramps
over the *scenario's* ramp duration, not its own: 30 of 50 Vusers still
take the 100 seconds the scenario takes to start all 50, because the
Controller issues its batches across the groups. A group holding one
Vuser out of a hundred keeps that Vuser — a share is never rounded down
to nobody.

**Parameters the script writes.** A `lr_save_string` with a literal
became a constant variable of that name, so a request referring to
`{TransName}` resolves where before it resolved to nothing. Check the
variable list against the names your requests use: a reference with no
variable behind it sends the literal `${TransName}` to the server, which
a validation run shows as a puzzling 404 rather than as a missing
variable.

**Random picks out of an extracted list.** `lr_paramarr_random("productId")`
became a **second extractor** beside the one that fills the array — same
expression, its own name, and OctoPerf's match number for "one occurrence
at random". Beside rather than instead, because a script routinely reads
the whole array as well. If you see two extractors on one request with
identical expressions, that is why; deleting the one with match number
`-1` breaks whatever reads the full list.

**What a capture reads.** VuGen scopes an extractor and a check with
`Scope=` or `Search=`, and the two spellings mean the same thing.
`Cookies` and `Headers` became an extractor on the response **headers** —
a cookie reaches a client in a `Set-Cookie` header, so a session
correlated with `"Scope=Cookies"` reads them. `Body`, and a call naming
no scope at all, became one on the body. `All` searched both, which an
OctoPerf extractor cannot, and lands on the **body**. So a value
correlated out of a header with `Scope=All` is the one capture to
re-point by hand: the tree and the expression both look right and the
variable comes back empty, several steps before anything fails.

**Content types.** `EncType=` is where VuGen keeps a request's content
type — 1564 call sites in the corpus against four that set a
`Content-Type` header themselves. An OctoPerf post body knows two
shapes, form encoding and multipart, so an `application/json` or
`text/xml` body travels as a body plus a `Content-Type` header. If a
POST is refused as a form, check that header survived your edits.

**Page assets.** The importer follows the script's own structure, so the
recording level is visible in the tree. An `EXTRARES` run is a page
*property* and became `downloadResources` on one request — OctoPerf
re-reads the response and fetches what it names, which is not
byte-for-byte the recorded list. A `Resource=1` call is a *call* and
became a request of its own. So a HTML-mode recording gives a compact
tree that fetches slightly different assets, and a URL-mode one gives an
explicit tree that fetches exactly the recorded set. Neither is wrong;
know which you have.

**URLs recovered from snapshots.** In an HTML-mode script every
`web_link` and `web_image` got its URL from `data/tNN.inf`, and that URL
is the request the click produced — including anything the recording
carried in it. A path like
`/actions/Catalog.action;jsessionid=FFEA949193A1ECEF837C64920ABBAB83`
is a session id baked into the path, faithful to the recording and
useless on replay. Those are what `octoperf-auto-correlation` is for,
and they will not be in `unconverted`.

**Headers that last.** `web_add_header` applies to the next request and
`web_add_auto_header` to every request until reverted. Both crossed
over, but they are easy to confuse when reading the result: an
`Authorization` on every request of a script that set it for one is the
symptom.

**Think times.** A `lr_think_time` converted to the pause it takes at
**replay**, not the number it names. `default.cfg` decides: `[ThinkTime]
Options=NOTHINK` switches think time off for the whole script, and a
factor multiplies what is left. A script full of `lr_think_time(7)` that
imported with no delays at all is a script whose own configuration
switched them off. Check the `.cfg` before assuming the import lost
them.

**Parameters that share a file.** Two `.prm` parameters reading the same
`.dat` with `SelectNextRow="Same line as …"` became **one** CSV variable
with two columns, which is what keeps a login paired with its password.
The column names come from the parameters, never from the file's header
line — a script references `{AOS_URL}` while its `.prm` says
`ColumnName="Col 1"` and the file's header says `AOS_URL`. If a variable
resolves to the wrong column, that is where to look.

**Unique parameters.** `SelectNextRow="Unique"` is the only selection
that guarantees a value goes to one virtual user and no other, so it
became a **shared** CSV variable; everything else became one private to
each user. Getting that backwards is how a hundred users log in as the
same account.

## 5. Validate

Run `validate_virtual_user` before anything else. On a script of any
size the first validation will be red, and that is expected: use
`octoperf-validation-triage` to group the failures, and
`octoperf-auto-correlation` for the session tokens VuGen correlated with
its own `web_reg_save_param` calls and for the ones it never correlated
at all.

A LoadRunner script that replayed green in VuGen can still fail here for
a reason that is neither tool's fault: the recording is old. Check the
dates in the recorded cookies and paths before hunting a conversion bug.

Do not run a scenario until one Virtual User validates clean —
`run_scenario` burns credits.

## Related skills

- `octoperf-validation-triage` — the first validation run will need it.
- `octoperf-auto-correlation` — for the session ids baked into recorded paths.
- `octoperf-scenario-composition` — for the Set Up / Tear Down the sequences become, and the load shape.
- `octoperf-bench-reports` — for reading the SLA profiles the import created once a run has data.
