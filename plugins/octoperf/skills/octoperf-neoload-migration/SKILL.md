---
name: octoperf-neoload-migration
description: Use when migrating a NeoLoad project to OctoPerf. Triggers on "migrate from NeoLoad", "import my NeoLoad project", "convert a .nlp project", "we're leaving Neotys/Tricentis NeoLoad", "can OctoPerf read config.zip". Decides first whether the project is worth migrating, imports what converts mechanically, then finishes by hand the constructs that have no OctoPerf equivalent — protocol plugin requests, NeoLoad JavaScript, Advanced Actions, try/catch, fork, list and password variables, XPath assertions, monitors. Requires the OctoPerf MCP server.
---

# OctoPerf — NeoLoad migration

`import_neoload_virtual_user` does the mechanical part: it rebuilds the
Virtual Users, HTTP servers, variables, data files and scenarios. It
deliberately stops short of what has no OctoPerf equivalent, and hands
back everything it left undone in the response's `unconverted` list —
usually as a **named marker** planted in the tree. This skill is what
turns that list into a test that runs.

Work in this order. Skipping the first step is how migrations get
committed to and then abandoned.

## 1. Decide before you import

Call `analyze_neoload_project` and read `portability` before anything
else:

| Verdict | What it means | What to do |
|---|---|---|
| `MECHANICAL` | Plain HTTP throughout | Import; expect it to run nearly as-is |
| `ASSISTED` | HTTP plus scripts or control flow | Import, then work the `unconverted` list |
| `MOSTLY_MANUAL` | Protocol plugins outnumber HTTP requests | **Say so before importing** |

`MOSTLY_MANUAL` is not a difficulty rating, it is a different project.
A NeoLoad plugin request (Oracle Forms, AMF, GWT, Siebel, RTMP,
Hessian) carries a proprietary dialog encoded in base64, not an HTTP
body — there is nothing to translate, and no tool on either side can
invent it. The traffic has to be recorded again against the live
application. Tell the user the count from `plugins` and let them decide;
importing first and discovering it later wastes their time and yours.

The per-user `virtualUsers` breakdown matters just as much: a project
can be `MOSTLY_MANUAL` overall while three of its ten users are pure
HTTP. Migrate those, and leave the rest.

A verdict of `MOSTLY_MANUAL` on a Virtual User with **no requests at
all** is an Advanced Action, not a protocol plugin: those weigh in the
verdict too, because an APM connector or a Selenium launcher converts no
better than an AMF dialog does. `customActions` on the breakdown says how
many.

Then read `unconverted`, which the assessment carries as well. Nothing has
been planted anywhere at this point, so the `marker` of every entry is
empty and the `facts` are what identify the element.

The elements of the tree come with a `NeoLoadElementFacts` naming one — an
Advanced Action by the plugin it calls rather than by the name the step was
given. That is enough to say whether the step is a launcher to drop or a
helper worth porting, which are very different answers to give before
promising anything. What an import never touches carries its own shape
instead: a `MONITOR` a `NeoLoadMonitorFacts` with the host and its counters,
an `SLA_PROFILE` a `NeoLoadSlaProfileFacts` with what NeoLoad measured. Read
`@type` first, as everywhere else in the list. The same list names the
`ENCRYPTED_VARIABLE` passwords you will have to ask for.

## 2. Import

`import_neoload_virtual_user` takes the whole project folder zipped, or
its `config.zip` alone. **Prefer the full folder** — `config.zip` holds
only the three XML documents, so a bare upload arrives without recorded
request bodies, without the JavaScript sources, and without the CSV and
TXT files backing the file variables. You will be reconstructing all of
them by hand otherwise.

Pass `providerId` to bring the scenarios in. Without it the Virtual
Users are created alone: a NeoLoad scenario names load generators by
hostname, which says nothing about an OctoPerf provider, so there is
nothing sensible to default to.

The iteration pacing goes with them. NeoLoad paces the Actions container
of the Virtual User; OctoPerf paces the user profile that runs it, so a
project imported without `providerId` loses the pacing along with the
scenarios — and the users then loop as fast as the server answers, which
is a different test. Import again with a provider rather than adding the
pacing by hand.

## 3. Work the `unconverted` list

`unconverted` is the whole of the manual work — the import merges into
it what conversion hit and what it never touches, so it is the only list
you need.

An entry is data, not prose. It carries a `kind` — which construct it
is — and, for most kinds, a `facts` object holding the values the
importer had: a plugin name, a counter, an expression, a timeout. **This
skill is where the explanation lives**: find the section named after the
`kind` below and it tells you what happened and what to do; the `facts`
tell you with what. Nothing in the response asks to be read as English,
and nothing in it should be parsed as such.

`facts` is polymorphic and names its own shape in a `@type` property, so
one `kind` can hand over different shapes — a `GENERATED_VARIABLE` is a
`NeoLoadSqlFacts` when NeoLoad filled it from a database and a
`NeoLoadQueueFacts` when it drew from a queue. Read `@type` before the
fields. An entry with no `facts` at all is a kind whose section needs no
values from the project: the section alone is the instruction.

Most entries also name a `marker` — the exact action name planted in the
tree — and the `virtualUser` holding it. Find it with
`get_virtual_user`, rewrite it with `patch_virtual_user` against
`octoperf://schema/vu`.

An entry with an empty `marker` had nothing planted for it, because it
was never inside a Virtual User to begin with: a monitored host, an SLA
profile. Those are rebuilt with the typed tools rather than patched — or,
for an SLA profile, already exist and are waiting to be pointed at what
they measure.

Work from the list, not from the tree. A `[NeoLoad] …` name is not by
itself something to fix — `[NeoLoad] start next iteration` and
`[NeoLoad] stop virtual user` are working Groovy actions the converter
named that way. If a name is not in `unconverted`, leave it.

Take them in this order; the later ones are cheap and the first is
sometimes a reason to stop.

### `HTTP_PUSH` and `MEDIA_STREAM` — the call is there, the pacing is not

Three NeoLoad actions are a request plus something done with the answer.
All three keep the request, so the server is still called and the URL
still exercised; what is reported is the part that was the point.

- **Polled** (`pushType` = `POLLING`). NeoLoad asked again and again and
  cut messages out of each answer. This one rebuilds: wrap the request in
  a loop with a pause and add a regular expression extractor.
  `messageSplitter` is the very expression NeoLoad split the messages
  with — reuse it verbatim rather than writing one.
- **Streamed** (any other `pushType`). NeoLoad held one answer open and
  read messages as they arrived. This one does not rebuild: a JMeter
  sampler reads a response to its end before handing back, so a stream
  the server keeps open stops the Virtual User instead of feeding it.
  Poll an endpoint if the application offers one; otherwise leave it out
  and say so.
- **Played** (`MEDIA_STREAM`, a `NeoLoadMediaFacts`). A media stream
  paced to a bitrate, buffering `bufferSeconds` ahead and stopping where a
  viewer would — `durationStrategy` `PLAY_ALL` means it played to the end,
  anything else means it stopped after `durationSeconds`. The request now
  pulls the whole file as fast as the link allows — the same bytes in a
  fraction of the time, which a CDN answers quite differently. If the test
  was about delivery, rebuild the pace with ranged requests and pauses
  between them.

A polled or streamed action can also hold steps of its own in NeoLoad —
what it ran on each message it cut out. Those are not brought over: there
is nowhere for them to go until the loop around the request exists, and
planting them beside the request would run them once instead of once per
message. Open the action in the NeoLoad project before rebuilding a poll,
and put them inside the loop you build.

Ask before rebuilding a stream or a media action: both are cheap to
leave out and expensive to fake, and a test that was never about
delivery does not need either.

### `SOAP_ENVELOPE` — paste the message in

A SOAP call generated from a WSDL arrives as an HTTP POST with the right
server, path, `Content-Type` and `SOAPAction`, and its envelope
assembled from the operation's parameter tree — NeoLoad describes the
message rather than storing it, so the import builds it.

An entry means the description was not enough: an `rpc` or `encoded`
operation, or one carrying a SOAP header block, whose message needs the
schema the WSDL held and the project does not carry. The request travels
with its address and headers, so what is missing is only the body. Take
the envelope from the WSDL `facts.wsdl` names, or from a capture of the
original call, and paste it into the request. `version`, `style` and
`use` say which shape the message had — an `rpc` style or an `encoded`
use is why it could not be assembled.

Two things worth knowing whichever way it arrived:

- **A recorded SOAP call is a plain request.** A proxy sees a POST with
  an XML body, so it converts like any other and none of this applies.
- **A 200 is not a success.** A SOAP fault comes back in the body, often
  with a perfectly ordinary status. Add a response assertion on
  something the answer contains — the result element, not the status.

### `PLUGIN_REQUEST` — re-record, do not repair

`facts.plugin` names the protocol — FORMS, AMF, GWT, Siebel, RTMP,
Hessian — and `facts.transport` what it spoke over, which decides what
came with the marker.

Over **plain HTTP**, the marker is a real request with the right method,
path, server, headers, extractors and assertions, and **no body**. That
skeleton is worth keeping as documentation of the sequence, but it will
not replay.

The same plugins also speak over WebSockets, raw sockets and server
pushes, and those leave a named marker holding nothing — there is no
request to rebuild a skeleton from. The answer is the same either way.

Do not attempt to reconstruct the payload from the base64 dialog. Tell
the user this protocol needs a fresh recording, and point them at the
HAR import for the parts that speak HTTP.

### `JAVASCRIPT_ACTION` — translate the API, keep the logic

The script came over verbatim inside a `JSR223Action`, and it does not
run: it is written against the NeoLoad JavaScript API. Translate:

| NeoLoad | JSR223 (Groovy) |
|---|---|
| `context.variableManager.getValue("X")` | `vars.get("X")` |
| `context.variableManager.setValue("X", v)` | `vars.put("X", v)` |
| `logger.debug(msg)` / `logger.error(msg)` | `log.debug(msg)` / `log.error(msg)` |
| `context.currentVirtualUser.name` | `ctx.getThreadGroup().getName()` |
| `new com.acme.Whatever()` from the project's `lib` | nothing — the jar is not there |

Switch the action's `language` to `groovy` once translated; the
imported one is set to `javascript` only because that is what the
source was.

A script calling into a Java class the project shipped in its `lib`
folder cannot be salvaged as-is. Read what it does — it is usually
logging or a small helper — and rewrite the behaviour rather than
porting the class.

The script does run before you touch it — the load generators carry a
JavaScript engine — which is why the failure is a `ReferenceError` rather
than a refusal to start. `ReferenceError: "X" is not defined` is the
signature of this whole section: `X` is either a NeoLoad API object the
table above renames, or a function from one of the libraries below.

`facts.libraries` names the `.js` files the project brought along.
NeoLoad loads every library into the engine and no script names the one
it uses, so a call the source does not define may come from any of them
rather than from the NeoLoad API — worth knowing before hunting for an
equivalent that never existed. Each file travels in the archive under
`scripts/`, so read it and fold what the script needs into the
`JSR223Action`. NeoLoad's own `neoload-<version>.js` is never listed:
nobody wrote it and nobody has to port it. An empty list makes the
NeoLoad API the only suspect.

### `TRY_CATCH` — re-express the recovery

The try branch is a plain container and needs nothing. The catch branch
sits beside it under a marker, and its steps now run **unconditionally**
— which is wrong, and is why it is reported.

`catchesErrors` and `catchesAssertions` say what entered the branch, and
they are re-expressed differently: an error branch is rebuilt from the
preceding requests' error handling set to continue, an assertion branch
from the assertion's own result. Either way, extract the failure into a
variable and wrap the former catch steps in an `IfContainerAction`
testing it. If the catch branch only logged, delete it.

### `FORK` — it runs in parallel, check where it landed

The branches keep running at the same time: the fork becomes a **Parallel
Controller**, and the Virtual User pulls the plugin by naming it. So this
entry is not about something lost, it is about three ways the result
degrades without failing.

- **Position.** The controller behaves only as the **first child** of its
  container. Open the Virtual User and look at where it sits; if steps
  precede it under the same parent, move it or wrap it.
- **Think times.** They make it report incoherent response times, and the
  delays NeoLoad puts between the steps of a branch become exactly that.
  Move them outside the controller, or drop them if the branches were
  meant to fire together.
- **Variables**, when `facts.copiedVariables` is true. NeoLoad gave each
  branch its own; the controller runs them on the Virtual User's, so a
  branch now sees what another writes. Where the branches were meant to be
  independent, give each one variables of its own name. False means
  NeoLoad shared them too, and there is nothing to do.

Response times are reported per branch rather than for the fork as a
whole — `PARENT_SAMPLE` is off, since a migration wants to know which
branch got slower.

### `CUSTOM_ACTION` — find out what the plugin did

A NeoLoad Advanced Action: the project calls a plugin by name and passes
it parameters, the code sits in a jar on the NeoLoad installation. There
is nothing to translate, so `facts.plugin` names it and
`facts.parameters` holds what it was called with. Read the plugin first,
because the three families end very differently:

- **a launcher** — `Java Action`, the Selenium and Tosca runners: it
  drove something outside the protocol layer. Nothing in OctoPerf runs
  it. Ask whether the step mattered to the load, and either drop the
  container or rebuild the work it did as HTTP requests;
- **a monitoring connector** — `DynatraceMonitoringAction`,
  `AppDynamics`, `NewRelic`: recreate it as an OctoPerf monitor with the
  matching typed tool (`create_newrelic_monitor`, and see
  `octoperf-monitoring`), not as an action inside the Virtual User;
- **a protocol or utility helper** — anything computing a value, signing
  a payload, calling an API: usually a `JSR223Action`, using
  `facts.parameters` as its inputs.

`facts.encrypted` names the parameters NeoLoad holds under a key of its
own. They appear in `parameters` with an empty value, which is not the
same as a parameter passed empty — ask the user for those, and leave the
genuinely empty ones alone.

### `ENCRYPTED_VARIABLE` — type the password back in

NeoLoad encrypts password variables with a key held by its own
installation, so the value cannot travel. The variable does: it is
already in the project as an **empty secret variable** under its
original name, so every `${…}` reference still resolves. Ask the user
for the value and set it with `create_secret_variable`. Nothing else
needs touching — leave the name alone, the project points at it.

No facts: the `marker` is the variable name, and that is all there is.

### `CONTAINER_EXECUTION` — the draw survives, the odds do not

NeoLoad plays a container one of three ways, and two of them are a
JMeter controller exactly, so they convert without a word:

| NeoLoad | becomes | reported |
|---|---|---|
| plays all elements in order | a Container | no |
| plays all elements in a random order | a Random Order Controller | no |
| draws **one** element | a Random Container | no |
| draws **several** of many | a Container, in order | yes |
| weights the draw | drawn evenly | yes |

So an entry here means one of the last two, and the facts say which: a
non-empty `drawn` is the draw of several, a non-empty `weighted` is the
weighting. Both can be set on the same container.

**A draw of several** has no counterpart: JMeter draws one element or
plays them all, never three of seven. Every element now runs once, in
order, which moves the load mix without failing anything — no validation
and no run would show it. Rebuild it if the mix matters: a Random
Controller holding sub-containers, or an `IfContainerAction` per element
over a random variable.

**A weighted draw** converts to an even one. `weighted` lists the
elements that carried a weight of their own, each as an `element` and its
`weight`; the ones NeoLoad left at the default are not listed. Rebuild
the ratio with `create_random_variable` over 1-100 and one
`IfContainerAction` per branch, sized to its weight — three-to-one
becomes `<= 25` and `> 25`.

Do not reach for a `LoopContainerAction` here. NeoLoad weights are odds
in a draw, not repetition counts.

### `SWITCH_FALL_THROUGH` — one condition to widen, at the end of the switch

A NeoLoad switch becomes one `IfContainerAction` per case, gathered under
a container named after the switch, and a last one testing that no case
matched — the only way to say *otherwise* with no switch to fall out of.
The switch value is compared as a string to each case value, which is
what NeoLoad does with them.

A NeoLoad switch is **fall-through**: a case whose Break flag is cleared
lets the case below it run as well. That converts on its own, because the
flags are read at import time — the case below accepts its own value
*and* the values of the cases falling into it, so it runs on exactly the
passes NeoLoad ran it on:

```
case Case 1   "${role}" == "Case 1"                            <- Break cleared
case Case 2   "${role}" == "Case 1" || "${role}" == "Case 2"
default       "${role}" != "Case 1" && "${role}" != "Case 2"
```

An entry here is the one arrangement that does not settle itself: the
**last** case has no case below it to fall into, and nothing states
whether NeoLoad then runs the default. It does not here — the default
keeps testing that no case matched.

`caseValue` names the case. If the project relied on the default running
after it, remove that value's `!=` term from the default's condition so
both run on it. If it relied on nothing after it, there is nothing to do:
the tree already says so.

A case switched off in NeoLoad arrives switched off, and the default's
condition does not mention it — a case that does not run matches nothing,
so the default answers for its value. Nothing to do there either; it is
worth knowing only because a disabled `IfContainerAction` in the middle
of a switch is not a leftover.

### `RENDEZVOUS_POLICY` — confirm the numbers, the barrier is already there

The meeting point converted: OctoPerf has a rendezvous action, and every
step naming the same NeoLoad `rendezVousName` landed on the same one.
Nothing to rebuild.

What is reported is the release policy, because NeoLoad does not put it in
the project — how many users the rendezvous waits for, and for how long,
are set in the Controller. The import chose **every user, released after
30 s, across the whole test**, which is what a NeoLoad rendezvous does by
default. Ask what the run being migrated actually used and patch the
`RendezVousAction` if it differs; a barrier that releases at the wrong
threshold changes the test without failing it.

### `SERVER_CREDENTIALS` — type the password back in, on the server

A NeoLoad server that authenticates. Both sides hang credentials off the
server, so the scheme and the user came over onto the OctoPerf HTTP
server; the password did not, being encrypted with NeoLoad's own key.

The `marker` is the user name; `facts` names the `server`, the `scheme`
and the `user`. Ask for the password and set it on the server — this is
not a variable, so `create_secret_variable` is the wrong tool: patch the
server's authorization instead. Until then every request to that host replays
unauthenticated and fails as a 401, which is worth knowing before you
read that as a correlation problem.

A scheme other than Basic is reported the same way but carried no
further: nothing was attached, so the authentication has to be rebuilt
whole.

### `GENERATED_VARIABLE` — rebuild it where it is used

A value NeoLoad recomputed as the test ran, with no OctoPerf variable
able to hold it. Three shapes of facts land under this one kind, so read
`@type` first:

- **`NeoLoadGeneratedVariableFacts`** — a random string, a random UUID, a
  date. `source` says which, and `function` **is the JMeter function that
  replaces it**, already written: take it from there rather than looking
  one up in the table below. `script` carries the source when NeoLoad
  computed the value with JavaScript, which is then rewritten as a
  `JSR223Action` the way the reported scripts are.
- **`NeoLoadSqlFacts`** — NeoLoad filled it from a database. `database`
  is the address it read from and `query` the statement. The archive
  carries neither the rows nor the password: export the rows once and
  upload them as a CSV variable, or query at run time with a JDBC sampler
  if they have to be fresh. An empty `database` means the project named
  no connection — ask where the rows came from.
- **`NeoLoadQueueFacts`** — values passed between Virtual Users through a
  queue holding `size` entries and waiting `pollTimeoutMs` for one.
  OctoPerf has the same queue, shared across every load generator: add a
  `PollQueueAction` on queue `name` before each place that read
  `${name}`, and a `PutQueueAction` wherever the test fed it. Poll into a
  variable of that same name and the references already written keep
  working. Do not reach for a CSV variable, which hands rows out but
  never takes any back.

For the first shape, put the replacement **in the request that uses the
value**, not in a project variable: a project variable is a JMeter User
Defined Variable, evaluated once when the thread starts, which would
freeze a timestamp NeoLoad refreshed on every iteration. Inline in the
request, these are evaluated per sample — and are what `function`
already holds:

| NeoLoad drew | Function |
|---|---|
| a random string | `${__RandomString(10,abcdefghijklmnopqrstuvwxyz)}` |
| a random UUID | `${__UUID()}` |
| a random UUID, upper case | `${__changeCase(${__UUID()},UPPER,)}` |
| the current date | `${__time(dd/MM/yyyy,)}` |
| a date walking forward | `${__timeShift(...)}` |

OctoPerf's own random variable is numeric, which is why none of these
convert to one — do not try to substitute it.

### `WAIT_UNTIL` — make it wait, not spin

A NeoLoad wait converts to the loop that says the same thing: a while
container running **while the condition does not hold**, named
`[NeoLoad] wait until …`. The condition itself came over whole.

**It arrives switched off, and leaving it that way is safe.** A NeoLoad wait
holds no steps of its own — the grammar gives it a condition and nothing
else — so the loop comes over with an empty body, and an empty while loop
runs until the test ends. Enable it once you have put the two things below
inside it, not before.

Two things have to go inside it, and without them the loop is wrong
rather than approximate:

- **Something that re-reads the value on each pass** — the request, the
  Poll Queue action or the JSR223 that made the value change. A while
  container re-evaluates its condition but nothing else, so on its own it
  spins on what the expression already evaluated to.
- **A pause**, or the loop burns a core doing nothing.

And the timeout has no equivalent: NeoLoad gives up after
`facts.timeoutSeconds`, a JMeter loop never does. Add a counter and leave
the loop when it runs out, or the users wait as long as the test lasts.

The condition is wrapped in `${__groovy(…)}` and has to stay wrapped: a While
controller does not evaluate its condition, it ends the loop only when the
rendered text is the string `false`. A bare comparison renders to something
like `"done" != "done"`, which is not that string, so an unwrapped loop never
ends however well its body is filled in.

Waits usually come with the queues and forks around them — NeoLoad pairs
them, since what a user waits for is generally what another produces.

### `PREDEFINED_VARIABLE` — the name resolves to nothing, decide what it meant

NeoLoad defines thirteen variables itself, all named `NL-…`, plus a
`${<loop>_counter}` inside every loop. Nothing in the project declares
them, so nothing was created for them here — and an unresolved `${…}`
travels as its own text: a switch on one matches no case, a path built
from one is requested verbatim, and neither fails.

Two are rewritten before you see them and are never reported:

| NeoLoad | becomes |
|---|---|
| `${NL-VirtualUserId}` | `${__threadNum}` |
| `${<loop>_counter}` | `${__intSum(${__jm__<loop>__idx},1)}` |

The second is the loop's iteration number. JMeter publishes the same
index for a Loop, While or ForEach controller under
`__jm__<controller name>__idx` and counts it from zero where NeoLoad
counts from one, hence the sum. It is rewritten only for the loops the
project declares: a counter variable of your own called
`something_counter` is a variable like any other.

An entry here is one of the rest. `facts.reference` is the name without
its `${…}` — search the tree for it, the reference is still there —
and `facts.closest` is the expression that says the same thing where one
does:

| `reference` | what it held | `closest` |
|---|---|---|
| `NL-UserPathName` | name of the User Path | `${__threadGroupName}` — check what the run calls the population |
| `NL-HttpRequestName`, `NL-HttpRequestPageName`, `NL-HttpRequestTransactionName` | the request, page or transaction being played | `${__samplerName}`, which only resolves inside the sampler's own children |
| `NL-HttpRequestId`, `NL-TestResultId`, `NL-ScenarioName`, `NL-ProjectName`, `NL-ZoneName`, `NL-ControllerIp`, `NL-CustomResources`, `NL-TestStatus` | the run, not the traffic | nothing: a JMeter script is told none of them |
| `<name>_counter` naming no loop | the iteration counter of a container | nothing published: the closest is JMeter's own thread iteration, `${__groovy(vars.getIteration())}` — check its base against your first pass |

Where `closest` is empty, ask what the value was for before rebuilding
it. Half of them are only ever used to label something — a header, a
logged line, a filename — and a constant variable holding your own
label is a truer answer than a function reaching for a name JMeter does
not have.

### `SERIALIZED_VARIABLE` — re-enter the values

NeoLoad stores list variable rows as a Java-serialized blob, which the
importer refuses to deserialize — reading an arbitrary object graph out
of an upload is how a file import becomes remote code execution.
`facts.columns` carries the column names, which is the shape to re-enter.
Ask the user for the values, or pull them from the running NeoLoad
project, then create a CSV variable with `create_csv_variable`.

### `MISSING_DATA_FILE` — upload it

Something in the project reads a file the archive did not carry, and
`upload_project_file` puts it back under the name `facts.filename` gives.
Two cases reach here:

- **a CSV variable** pointing at a filename that never came. Almost
  always the consequence of uploading `config.zip` instead of the
  project folder;
- **a file a multipart request posted**. NeoLoad keeps what a recording
  uploaded under `recorded-resources/`, which an uploaded project folder
  carries and NeoLoad's own export does not — so this reaches the list only
  for a project exported rather than zipped. The file part is already in the
  request under the right name, so uploading a file under that name is all
  it takes. Ask what the endpoint expects — an image, a CSV — because the
  original bytes are gone.

### `BINARY_BODY` — rebuild it from the endpoint

A recorded body made of bytes no text body can carry: a protocol buffer,
an uploaded image, a compressed payload. The importer refuses to force
it through a decoder, which would post a body full of replacement
characters and fail against the server without saying why — so the
request comes over **with no body at all**. `facts.bytes` gives the size
of what was dropped, which is the first thing to weigh: a handful of
bytes is a protocol buffer worth skipping, a megabyte is an upload worth
rebuilding.

A binary frame inside a **WebSocket** message lands here too, with no
facts: the payload is left empty rather than decoded. Load it from a file
on the Single Write sampler, or rebuild it from what the endpoint
expects.

There is no way back from the recording alone. Ask what the endpoint
takes. A protobuf usually means the request is worth skipping rather
than replaying; an upload is better rebuilt as a multipart request with
`upload_project_file`.

### `WEBSOCKET_PUSH` — the connection works, nothing reads it

The connection and the messages sent on it convert on their own: a
NeoLoad channel becomes a Luminis **WebSocket Open** sampler, and each
message a **WebSocket Single Write**, payload and correlated variables
included. What does not convert is what NeoLoad never wrote down.

**Reads.** NeoLoad records what the server pushed into a file the
channel points at, not into actions — `facts.pushedMessages` is how many
it captured, and `facts.connection` names the channel. Zero means the
recording caught none, not that the server sends none. A replay that only
writes never consumes an answer, so add a **WebSocket Single Read**
wherever the script needs one: after a request whose reply it uses, and
once per pushed message it
should wait for. `patch_virtual_user` plants them; the samplers reuse
the open connection, so leave their `server`, `port` and `path` empty
and `createNewConnection` false, exactly as the writes have them.

**The close.** NeoLoad ends the connection with its container rather
than with an action, so nothing marks it. Add a **WebSocket Close** at
the end if the connection should not outlive the iteration — harmless
to omit for a single-iteration test, a leak across a long run.

Do not invent how many reads there should be. Ask what the script is
meant to wait for: a chat client draining every push and a trading
client reading one quote per order are the same recording and different
tests.

### `MONITOR` — recreate the host, then match the selection

Reported, never created, and not for want of trying: a monitor's counter
tree is computed by the load generator rather than the backend, and
creating one probes the target and refuses to persist a monitor it
cannot collect from. An import runs far from the monitored machine, long
after the test, with no agent named anywhere in the project. You are the
one who can do this, because you can ask.

The facts carry everything the project knew:

```json
{
  "@type": "NeoLoadMonitorFacts",
  "monitor": "MySQL", "connector": "mysql", "noEquivalent": false,
  "host": "db.example.com", "port": "3306", "login": "root",
  "intervalMs": 5000, "passwordEncrypted": true,
  "counters": [
    {"name": "Connections/ aborted clients", "unit": "connections/s",
     "alarms": [{"minimum": "5.0", "maximum": "", "duration": "30",
                 "durationKind": "SECONDS", "severity": "WARNING",
                 "comment": "High number of aborted connections"}]},
    {"name": "IO Requests/ bytes sent", "unit": "kbytes/s", "alarms": []}
  ]
}
```

Counter names hold slashes and spaces, and an alarm is a range, a
duration and a grade rather than a phrase — which is why both travel
whole. Never reassemble either from text.

Work it in three steps, and stop after the first if the user has no
agent for that host — a monitor cannot be created ahead of one.

1. **Create it** with the tool matching `monitor`: `create_mysql_monitor`,
   `create_linux_monitor`, `create_apache_httpd_monitor`,
   `create_tomcat_monitor`, `create_prometheus_monitor`,
   `create_generic_jmx_monitor`. Ask for the credentials when
   `passwordEncrypted` is true — the archive never carries a usable one.
   Creation fails outright on an unreachable target, which is the answer,
   not an obstacle: fix the host or the credentials before going on.
2. **Match the selection** with `update_monitor_counters` against
   `counters[].name`. Creation selects the type's defaults, which are not
   the same set NeoLoad had.
3. **Set only the thresholds that differ**, from `counters[].alarms`,
   with `set_monitor_thresholds`. An empty `minimum` or `maximum` is an
   open bound — `minimum: "5.0"` with an empty maximum is "over 5.0";
   `durationKind` is `SECONDS` or `OCCURRENCES`. Check first with
   `get_monitor_counters`: our stock thresholds are frequently NeoLoad's
   own, down to the numbers — both warn over five aborted MySQL
   connections a second across thirty seconds — and re-setting what is
   already there earns nothing.

An empty `monitor` means there is nothing to create, and `noEquivalent`
says which of the two cases it is. **True** — a connector we know and
have no monitor for, `dynatrace` or `datadog`: say so and move on, do not
substitute a different monitor. **False** — a connector the importer does
not know: `connector` is NeoLoad's own name for it, so look for a
matching `create_*_monitor` yourself and tell the user when there is
none.

NeoLoad's own controller statistics — user load, throughput — travel in
a connection block too, and are deliberately left out of this list:
OctoPerf reports them natively.

### `SLA_PROFILE` — the profile exists, point it at something

The import creates the profile and its thresholds; what it does not do
is decide what they measure. OctoPerf scopes an SLA by **where an
`SLAProfileAction` sits in the Virtual User tree**, so planting one is a
measurement decision rather than a mechanical step — which is why it is
left to you.

The facts name the `profile`, how many `metrics` it watches and how many
`alarms` those come to — one metric graded warning and critical is one
metric and two alarms — and `boundTo`, the elements NeoLoad measured it
on.

NeoLoad binds a profile to an element rather than to a population, so
those names are containers and Virtual Users. Add the action **inside**
each named container to measure what the original measured; add it once
at the end of a Virtual User to measure the whole of it. `patch_virtual_user`
plants it, and `get_sla_profile` / `list_sla_profiles` find the id.

A name matching a Virtual User is a binding on one of the three sequences
of a user path. Those carry no name of their own in NeoLoad and read as the
Virtual User they became here, so what was measured is the whole of the
sequence rather than a container inside it.

When `boundTo` is **empty**, the binding was switched off on every
element. That is worth repeating to the user
rather than fixing silently: the original checked nothing, so anywhere
you attach it is new behaviour, not restored behaviour. It is the common
case — across fourteen public NeoLoad projects, not one has a working
binding.

### `SLA_THRESHOLD` — state this one by hand

The rest of its profile converted; this line did not, because no
OctoPerf metric states it without guessing. `facts.profile` names the
profile it belongs to and `facts.identifier` the metric NeoLoad asked
for. Two cases land here:

- **throughput** — we hold bytes per second and NeoLoad's unit could not
  be established from any real project. Ask which unit the original used,
  then add `THROUGHPUT_RATE` or `THROUGHPUT_TOTAL`.
- **a percentile we do not compute** — NeoLoad names the one it wants and
  we hold the 80th, 90th, 95th and 99th. A project asking for anything
  else lands here rather than being served the nearest one. Ask what to
  substitute; do not pick for them.

Add them with `patch_sla_profile` to the profile that was already
created, rather than creating a second one. `list_sla_threshold_defaults`
gives sensible bands if the original value turns out to be unusable.

**Everything else already converted, in milliseconds.** NeoLoad states a
response time in seconds and OctoPerf holds milliseconds, so the import
scales them; do not scale them again. Percentiles included: the import
reads the one NeoLoad named and falls back to the 90th, which is
NeoLoad's own default, when the project names none.

### `XPATH_ASSERTION` — the node is checked, the regular expression is not

NeoLoad validates a response against a single XPath node, and the check
crosses over whole: OctoPerf's own model holds no XPath assertion, but
JMeter ships one, so it travels as a generic action the way the URL
rewriting modifier does. A plain pattern narrows the expression —
`(//title)[contains(., 'Yours')]` — which is what NeoLoad reads a node and
a pattern as, and an empty pattern narrows nothing, being the existence
test it is on both sides. Nothing is reported for either.

An entry here is the one half that cannot travel: a **regular expression**
on the node, which NeoLoad writes under `assertion-plugin-response`. XPath
1.0 has no such predicate, so the node is checked for its presence and
`facts.expression` hands over the expression that was checking its
content. Two ways to finish it, and the first is usually enough:

- fold a plain equivalent into the expression by hand —
  `(//status)[contains(., 'OK')]` where the original matched `OK|Ok`;
- or add an `XPath1VariableExtractor` with the same expression and work
  from the variable, when what matters is the value rather than a pass.

A negated check reports the same way and matters more: until it is
finished, the request passes on responses the original rejected.

### `STOP_VIRTUAL_USER` — the stop works, the restart is gone

Reported only for the variant that asked NeoLoad to start a replacement
user. The stop itself came over and runs: the marker
`[NeoLoad] stop virtual user` is a working `JSR223Action` calling
`ctx.getThread().stop()`, so **leave it alone**. What has no equivalent
is the second half — JMeter takes its thread count from the load policy,
not from a step in the script.

Ask what the replacement was for. Keeping the user count up during the
run is a load shape (see `octoperf-scenario-composition`), not an action.

No facts: NeoLoad records nothing about the replacement user beyond
having asked for one.

### `USER_PATH_SEQUENCE` — check whether the setup still shares a session

A NeoLoad user path runs three sequences, and the project spells out what
each is for: the init holds *"elements executed once when the Virtual
User starts"*, the actions *"elements repeated until the Virtual User
stops"*, the end *"elements executed before the Virtual User stops"*.

An OctoPerf Virtual User is only the repeated part, so the import splits
the user path in three. `Foo` keeps the repeated sequence — the name the
population points at — and `Foo - Init` and `Foo - End` become Virtual
Users of their own, attached to the profile as its **setUp** and
**tearDown**. Both are reported whatever happened, because two things
about them are decisions rather than translations.

```json
{"@type": "NeoLoadSequenceFacts", "sequence": "init", "virtualUser": "Foo - Init"}
```

**The session is the one to check first.** A setUp is a thread group of
its own, so its cookies and its JMeter variables never reach the
iterations. If the Init only warms a cache or writes a fixture row, that
is fine and there is nothing to do. **If it signs in, the login is now
useless** — the Actions run in a different thread, with an empty cookie
jar, and every request replays unauthenticated.

When the session matters, undo the wiring and keep the sequence inside
the user's own thread:

1. Clear `setUp` on the profile (`patch_virtual_user` is not the tool
   here — the setting lives on the scenario's user profile, so
   `patch_scenario`).
2. In `Foo`, at the very top, add a **Once Only Controller** holding a
   **`LinkAction`** pointing at `Foo - Init`.

That runs the Init on the first iteration of each thread — same thread,
same session, once per user, which is exactly what NeoLoad did. The
`LinkAction` keeps the steps in their own Virtual User rather than
copying them, so `Foo - Init` stays the one place to edit them.

There is no equivalent for the End: JMeter has no "last iteration only"
controller per thread. A tearDown is the closest thing, and it breaks the
session the same way — so if the End had to run in the user's session
(a logout that must carry the cookie), say so and leave it out rather
than replay it somewhere it means nothing.

**How many threads run the setUp is the second decision.** The import
writes one thread, one loop. NeoLoad ran the Init once per virtual user,
so if the Init provisions per-user data, the count has to match the
users the profile ramps to — no population states it, and guessing high
doubles the fixture data.

### `UNKNOWN_ELEMENT` — read the tag

An empty container named after a NeoLoad element the importer does not
know. No facts: `tag` is the whole of what is known.

`variable-modifier-action` is the one to expect, and it becomes a one-line
`JSR223Action` calling `vars.put`. A tag with a converter of its own never
reaches here — `wait-until-action` has one, and reports itself under
`WAIT_UNTIL` above — so an entry naming a tag this skill covers elsewhere
means the converter threw rather than that the tag is unknown; read
`CONVERSION_FAILED` in that case.

Inside an `assertions` block the same marker appears as a **disabled**
`ResponseAssertion` named `[NeoLoad] <tag> to rebuild`.
Two of those have a known meaning and no OctoPerf equivalent, because
OctoPerf checks them per run rather than per request:

- `assertion-duration` — the response had to arrive within `duration`
  milliseconds;
- `assertion-size` — the response size had to satisfy `operator` against
  `size`.

Recreate the first as an SLA profile on response time (see
`octoperf-sla`) and delete the marker. For anything else under
`assertions`, nothing in the marker says what was being checked: ask
what the request is supposed to return, then rewrite or delete it.

### `CONVERSION_FAILED` — the importer broke on this one

Not a construct without an equivalent: a converter claimed the element
and then threw. The import carried on rather than failing the project,
which is why one bad element does not cost you the other nine hundred.

What is left behind depends on what failed. An **action** leaves an empty
container named `[NeoLoad] <tag> failed to convert`, like any other marker,
so you have the spot. A **variable** leaves nothing — there is no tree to
plant it in — and every `${…}` reference to it stays pointing at a name
nothing defines. That second case is the one you cannot afford to skim
past: nothing in the tree will remind you of it.

`facts.failure` carries the exception. Read it before assuming the project
is at fault: a malformed project and a shape the importer does not expect
look the same from here, and the second is a bug worth reporting rather
than working around.

The `marker` names the element that failed — for a variable, the name
every `${…}` reference still points at, because the references were
rewritten before the conversion broke. Rebuild it with the matching typed
tool (`create_csv_variable`, `create_constant_variable`, …) under that
exact name and they resolve again.

### `CORRELATION_CANDIDATE` — the rule the project asks for and never states

The only entry that reports nothing broken. NeoLoad extracted a value from
a response and then left the recorded copy of that same value in clear on
other requests — its own correlation injected the variable in some places
and not others. Those requests replay a value from the recording session.

Nothing is created for it on purpose: a rule rewrites every match in every
Virtual User of the project, which is a decision about the test rather than
a translation.

The facts are read, not guessed. `expression` is NeoLoad's own, which is
known to capture the value; `extractFrom` is where it was found to match
when the expression was run against the recording; `occurrences` counts the
requests still carrying the literal and `targets` names the parts of them
it sits in, in the words `create_correlation_rule` takes.

```json
{"@type": "NeoLoadCorrelationFacts",
 "variable": "SAP_JSESSIONID", "expression": "set-cookie: JSESSIONID=([^;]+)",
 "extractFrom": "HEADERS", "targets": ["HEADER", "PATH"],
 "occurrences": 4, "virtualUsers": ["PetStore"]}
```

Create it with `create_correlation_rule` under `variable` — reusing the
`expression` and `extractFrom` as they come — then `apply_correlations_to_virtual_user`
on each Virtual User `virtualUsers` names, and poll the task. The engine
plants the extractor on the request whose recorded response holds the value
and rewrites the literal wherever it sits, so nothing has to be patched by
hand.

A value under six characters is never proposed, dynamic or not: a category
name of four characters occurs across a tree for reasons that have nothing
to do with a session, and a rule rewriting all of them breaks more than it
fixes.

**And the list is precise rather than complete.** It reports what NeoLoad
itself correlated; a value NeoLoad never correlated is invisible to it and
stays hard-coded — the `_sourcePage` and `__fp` tokens of a Stripes
application are the usual example. Those surface at validation, which is
where `octoperf-auto-correlation` takes over.

## 4. Check what came through, not just what did not

These convert silently and are worth verifying before you hand the
project over. Nothing reports them, so nothing will remind you.

**Extraction scopes.** NeoLoad runs an expression over the whole response,
headers and body at once, while an OctoPerf extractor reads one or the
other — so the scope is settled by running the expression against what the
recording holds. `set-cookie: JSESSIONID=([^;]+)` lands on the headers,
`OAUTH_CLIENT_SECRET:"(.*?)"` on the body although it reads like a header,
and an expression matching neither reads the body. An extractor whose
variable stays at its default through a whole run is worth checking here
first.

**Failing on a missed extraction.** NeoLoad fails a request whose
extraction finds nothing, which is a tick box on the extractor
(`assertionOnNoMatch`) rather than an element. OctoPerf says it with an
assertion, so the pair crosses over together, named
`[NeoLoad] <variable> found` and stated in the query language the extractor
reads: a `ResponseAssertion` on the extractor's own expression for a
regular expression, a `JsonAssertion` on its path with nothing to validate
for a JSONPath, an XPath assertion on its expression for an XPath.
Deleting one as a duplicate of its extractor restores a silent pass.

The query language is what matters here. NeoLoad keeps the expression of
every mode an extractor was ever switched to, so a JSONPath extractor
carries a `regExp` of `(.*)` beside its path — a check built on that
matches anything and can never fail, which reads as a control and is
none.

**Pauses.** A NeoLoad delay carries its own value, and a Virtual User can
overrule it from its runtime parameters — "Override think time", with a
`+/- X %` spread and a percentage factor. Both come over: a delay set to
draw between two values becomes a uniform random think time, an override
replaces the lot. What does **not** come over is a think time NeoLoad
resolved outside the project. If the imported delays look wrong, open the
`delay-action` in the source and compare against the `thinktime-policy`
on its Virtual User before assuming the import lost something — a
`duration="0"` under policy `0` really is a zero pause on both sides.

A think time reading `${__jexl3(${PAUSE_READ} * 130 / 100)}` is that
percentage factor applied to a pause held in a variable: the value is
only known at run time, so the multiplication travels with it. Leave the
arithmetic in integers if you edit it — `* 1.3` evaluates to a decimal,
which the JMeter timer answers with no pause at all rather than an error.

**Session ids carried in the URL.** A NeoLoad server can be set to rewrite
the session id into the path — `;jsessionid=…` rather than a cookie. That
becomes an `URLRewritingModifier` leading the Virtual User, named
`URL rewriting (<parameter>)`, because JMeter scopes it by position and it
has to reach every request below. **Leave it first.** Moving it under a
container, or deleting it as an unfamiliar element, silently restores the
bug it exists to prevent: the recorded ids stay hard-coded in the paths
and the replay works against a session that died with the recording,
which looks exactly like a correlation failure.

NeoLoad scopes that setting to one server and OctoPerf cannot, so a
project whose servers rewrite different parameters gets one modifier each,
all applying throughout. Each only touches the parameter it was given, so
they coexist — but it is worth knowing before you wonder why there are two.

**Multipart requests.** The parts come over one for one, and the
recorded `Content-Type` is dropped on purpose: it carries the boundary
of the browser that recorded it, while the engine rebuilds the parts
behind a boundary of its own. Nothing to repair — but if the server
insists on a charset or a vendor content type, add it back **without**
the boundary.

**Assertions that covered a whole Virtual User.** NeoLoad lets an
`assertions` block hang off a Virtual User or a container, where it is
checked against every request underneath. Those arrive as a
`ResponseAssertion` sitting first in that Virtual User or container
rather than inside a request, which keeps the same reach: an assertion
applies to every request below it. Two things are worth a look — a
container the user later moves takes its assertion along, and a global
assertion on a body fragment (`200`, a page title) now also runs against
the static resources of the page, which may not carry it.

**JSON assertions.** A NeoLoad JSONPath validation checks that the node
**contains** the value; the OctoPerf JSON assertion compares the node
**against** it. They agree when the path points at exactly the value
NeoLoad was looking for, and disagree when the node holds more than
that fragment — where NeoLoad passed, the import now fails. Turn those
into a regular expression: set `regex` on the assertion and widen the
value to `.*fragment.*`.

**Column references.** NeoLoad reads a file variable column as
`${VarName.Column}`; OctoPerf exposes each column as a variable of its
own, so every reference was rewritten to `${Column}`. When two data
files declare a column with the same name, those references now collide.
List the variables with `list_variables` and check for duplicate column
names across CSV variables — if there are any, rename the columns and
patch the references.

Columns named `col_0`, `col_1`, … are not what an import made of them:
that is NeoLoad's own default naming, written when it re-reads a data file
whose first line it is told not to use as a header. It renames the columns
and leaves every `${VarName.Column}` reference pointing at the old names,
so the source project is already broken — in the Controller as much as
here. The imported requests then ask for `${username}` while the variable
offers `${col_0}`, which is faithful rather than wrong. Rename the columns
on the OctoPerf variable, or fix the source and import again; do not
rewrite the requests to the `col_*` names, which says nothing to a reader.

**Column counts.** `sanity_check_virtual_user` may report
`N lines in file 'X.csv' have inconsistent column counts (expected: 13)`
on an imported file variable. Check the source before treating it as an
import defect: NeoLoad addresses columns by position and yields empty
past the end of a line, so a project can declare thirteen columns against
a two-column file and never complain. Where that is what happened, the
check is telling you the truth about the NeoLoad project — the eleven
other `${…}` references produced nothing there either. Say so rather than
trimming the declaration, which would hide it and break the references
already rewritten against those names.

**Init and End sequences.** They are not in the Virtual User you are
reading: each became one of its own, wired to the profile as a setUp and
a tearDown. `USER_PATH_SEQUENCE` above is where they are, and the session
question it raises is the one to settle before trusting a login.

## 5. Validate

Run `validate_virtual_user` before anything else. On a project of any
size the first validation will be red, and that is expected: use
`octoperf-validation-triage` to group the failures, and
`octoperf-auto-correlation` for the session tokens NeoLoad correlated
with its own extractors and OctoPerf now needs its own rules for.

Do not run a scenario until one Virtual User validates clean —
`run_scenario` burns credits.

## Related skills

- `octoperf-validation-triage` — the first validation run will need it.
- `octoperf-auto-correlation` — for the dynamic values that break on replay.
- `octoperf-scenario-composition` — for Set Up / Tear Down and load shapes.
- `octoperf-monitoring` — for rebuilding the monitored hosts.
- `octoperf-sla` — for the thresholds.
