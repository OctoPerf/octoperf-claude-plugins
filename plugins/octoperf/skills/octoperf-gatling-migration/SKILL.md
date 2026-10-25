---
name: octoperf-gatling-migration
description: 'Use when migrating Gatling simulations (Java, Scala or Kotlin) to OctoPerf. Triggers on "migrate from Gatling", "convert my Gatling simulation", "import a Gatling project", "can OctoPerf run my Gatling scripts", "we''re leaving Gatling". Decides first whether the project is worth migrating, then rewrites each simulation as a K6 Virtual User and its injection profile as an OctoPerf scenario, and treats by hand what has no K6 counterpart: open-model injection, session code, XPath, embedded resources, non-HTTP protocols, libraries the project imports. Requires the OctoPerf MCP server.'
---

# OctoPerf — Gatling migration

OctoPerf does not run Gatling. A Gatling simulation becomes a **K6 Virtual User**: you rewrite its scenario in
JavaScript, and its injection profile becomes an OctoPerf scenario. The DSLs are close — a request, a check, a
feeder, a loop each have a K6 counterpart — so most of the rewrite is mechanical, and the customer gains what Gatling
OSS keeps back: connect time, TLS handshake, TTFB and bytes per request.

`analyze_gatling_project` reads the project without writing anything and tells you what does **not** cross over on
its own, by source and line. There is no Gatling import tool: you write the K6 files, import them with
`import_k6_virtual_user`, turn the load into a scenario with `create_scenario_from_k6_options`, and validate.

**Read `octoperf://skills/k6-scripting` first.** It holds what a K6 script on OctoPerf may and may not do — `__ENV`
and `./octoperf.js`, CSV variables split per injector, reserved file names. This skill says how to get there from
Gatling; that one says where "there" is.

The conversion is **one-shot**: the K6 Virtual User becomes an OctoPerf asset and the Gatling project goes on living
elsewhere. Say so before starting — re-running the conversion after the customer edits their simulation would
overwrite whatever was finished by hand.

## 1. Decide before you convert

Zip the Gatling project folder whole — simulations, the helpers they import, `resources/` with feeders and body
templates, any `.proto` — call `analyze_gatling_project`, and POST the zip to the URL it returns. Nothing is
written to the project. Read `portability` first:

| Verdict | What it means | What to do |
|---|---|---|
| `MECHANICAL` | Plain HTTP, closed load, nothing to translate by hand | Convert; expect it to validate nearly as-is |
| `ASSISTED` | Everything has a K6 counterpart, some of it to write by hand: session code, open injection, checks K6 lacks | Convert, then work the `unconverted` list |
| `MOSTLY_MANUAL` | A protocol with no K6 extension (JMS, MQTT, AMQP), or a library the upload does not carry | **Say so before converting** |
| `EMPTY` | No HTTP request, and nothing left by hand but the shape of the load (injection, throttling, sequence, global assertions) | **Say so**: the upload is missing its simulations |

`MOSTLY_MANUAL` is not a harder `ASSISTED`. A simulation that signs its requests with the customer's own crypto
library, or sends JMS messages, has a part no rewrite of the DSL reaches. Name the `EXTERNAL_DEPENDENCY` and
`NON_HTTP_PROTOCOL` entries and let the user decide.

Then read the rest of the answer:

- `languages` — Java, Scala or Kotlin. Scala projects older than Gatling 3.7 write the Expression Language as
  `${name}`, newer ones as `#{name}`; both convert the same way.
- `simulations` — one entry per concrete `Simulation` class, with the requests and the **injection steps** of its
  source, of the abstract base it extends and of the helpers they import — a project often builds its scenarios and
  its load in helper objects. This is the load you will rebuild in §2; the `inventory` counts the whole project.
- `inventory` — requests, checks, extractors (`saveAs`), feeders, pauses, loops, conditions, session functions. They
  count what the sources **write**, not what a run sends: a helper method building a request —
  `req(name, url)` — is one request however many times the scenario calls it. Count the requests in the scenario.
- `unconverted` — every construct with no K6 counterpart, with `source`, `line` and the line itself in `excerpt`
  (line `0` for an import). `octoperf://schema/gatling-assessment` is the vocabulary.

A project holding several simulations is several Virtual Users. Ask which ones to migrate before rewriting all of
them — example projects often repeat one journey with a request more or less, and the fullest one covers the rest.

## 2. Rebuild the load first

The load decides the shape of the script, so read it before writing a line of JavaScript. OctoPerf runs a
**closed** load — a number of concurrent virtual users — and K6 runs it with an executor OctoPerf generates from the
scenario (`octoperf://skills/k6-scripting`). Gatling's injection steps split in two:

| Gatling | Model | K6 scenario (`options.scenarios`) |
|---|---|---|
| `constantConcurrentUsers(n).during(d)` | closed | `{"executor": "constant-vus", "vus": n, "duration": d}` |
| `rampConcurrentUsers(a).to(b).during(d)` | closed | `{"executor": "ramping-vus", "startVUs": a, "stages": [{"duration": d, "target": b}]}` |
| `atOnceUsers(n)`, `rampUsers(n).during(d)` | **open** | depends on the scenario — see below |
| `constantUsersPerSec(r)`, `rampUsersPerSec(a).to(b)`, `stressPeakUsers(n)`, `incrementUsersPerSec(…)` | **open** | depends on the scenario — see below |
| `nothingFor(d)` | — | a stage holding the current target for `d`, or a `startTime` on the next scenario |

**An open step injects users that run the scenario once and leave.** Whether that is a closed load in disguise
depends on the scenario, not on the step — which is why the assessment reports every open step as
`OPEN_MODEL_INJECTION` and leaves the call to you:

1. **The scenario loops for a time** — `during(5.minutes) { … }`, `forever { … }`, `asLongAs(…) { … }` wrapping the
   whole chain. Each injected user then *is* a virtual user for the length of the loop: `atOnceUsers(n)` is `n`
   concurrent users, `rampUsers(n).during(r)` a ramp to `n` over `r`. Convert to `ramping-vus`, the loop becoming
   the K6 iteration (the body of the exported function) and its length the scenario's duration.
2. **The scenario is one journey and ends** — the true open model: a rate of arrivals. K6 has
   `constant-arrival-rate` / `ramping-arrival-rate`, but **OctoPerf does not run them yet** —
   `create_scenario_from_k6_options` reports them `OPEN_MODEL` and creates no profile. Rebuild it as a closed load
   by Little's law: concurrent users ≈ arrival rate × duration of one journey, each user looping the journey. Take
   the duration from a validation run (§5), not from a guess, and **tell the user this is an approximation** — a
   closed load slows its arrivals down when the system slows down, an open one does not. Read the rate off the
   step: `rampUsers(n).during(d)` spreads `n` arrivals evenly, a rate of `n / d`. `atOnceUsers(n)` is a burst, and
   its closed twin is exact: `{"executor": "per-vu-iterations", "vus": n, "iterations": 1}`. And when the journey
   outlasts the ramp — `rampUsers(100).during(60 s)` over a 97 s journey — nobody has left when the last user
   arrives: the load is a closed ramp to `n` over the ramp, held for the rest of one journey, with the `gracefulStop`
   of a journey, so each VU runs its one journey — `stages: [{60s → 100}, {37s → 100}]`, `gracefulStop: "97s"`.

Never turn a rate into a user count silently. Write the numbers in the report: the Gatling step, the journey
duration you measured, the user count you derived. Then check them against each other: the closed load must run
about as many journeys as Gatling injects users. `constantUsersPerSec(3).during(5 s)` injects 15 users; with a
journey of 2.3 s measured by the validation, 3 × 2.3 ≈ 7 users looping for 5 s run 7 × 5 / 2.3 ≈ 15 journeys —
`{"executor": "constant-vus", "vus": 7, "duration": "5s"}`.

**A short journey needs pacing.** Rounded up to a whole user, Little's law overshoots whenever rate × journey is
below one or two: `constantUsersPerSec(10)` over a one-request journey of a few milliseconds is 0.05 users, one user
looping without a pause sends hundreds of requests a second, not ten. Take `ceil(rate × journey)` users and pace each
iteration to `users / rate` seconds — sleep what is left of it after the journey — with the pace in a constant
variable (`PACE_SECONDS`) the user can tune. Read the rate back in the validation log: `http_reqs` per second is the
arrival rate the closed load reproduces — and `iterations` per second the journeys, when a journey sends several.

**The journey has to be measured against the system under test.** Validated against a mock or a staging server
answering in a millisecond, a journey looks far shorter than it runs in production, and the user count derived from
it far too small: a paced user cannot keep its pace once one journey outlasts it. Keep a margin — twice the users,
each paced twice as long, gives the same rate — and say which target the duration was measured on.

An `injectOpen` of several steps runs them one after the other: one K6 scenario per step, all running the same
function, each with a `startTime` at the end of the step before — `atOnceUsers` takes no time, `nothingFor(d)` is
only that delay. Set each scenario's `gracefulStop` to the journey duration: a Gatling user injected before the end
finishes its journey, a K6 iteration still running at the end is cut unless the stop leaves it time.

A `setUp` built over a list — `urls.map { url -> scn.inject(…).protocols(http.baseUrl(url)) }`, one population per
server — is one K6 scenario per entry, all running the same function, each passing its URL in the scenario's `env`
(`"env": {"PELIAS_URL": "…"}`), which the script reads as `__ENV`.

A load read from properties — `simulation.users.count = 1` — is the author's default, often a smoke test: convert it,
and ask for the numbers of the run the user has in mind.

`andThen` chains scenarios: give each later one a `startTime` equal to the end of the one before
(`SEQUENTIAL_SCENARIOS`). `maxDuration(d)` caps the run: make it the scenarios' duration.

Each Gatling `scenario(…)` of the `setUp` becomes one K6 scenario, each running its own exported function through
`exec` — one script, one function per Gatling scenario.

## 3. Rewrite the scenario

One K6 file per simulation, one exported function per Gatling scenario, helpers as further `.js` files the entry
script imports. The platform generates `./octoperf.js` beside them.

**Export a `default` function too**, calling the main scenario's: a validation runs the default export, where the
scenario's profiles run the named functions through `exec`.

```js
export function PetStoreSimulation() { … }        // exec of the K6 scenario, one per Gatling scenario
export default function () { PetStoreSimulation(); }  // what validate_virtual_user runs
```

| Gatling | K6 |
|---|---|
| `http.baseUrl("https://shop")` | a constant variable: `create_constant_variable(BASE_URL)`, read as `__ENV.BASE_URL` |
| `http.header(…)`, `acceptHeader`, `userAgentHeader`, `headers(Map)` | a shared `params.headers` object passed to every call — a setting written twice keeps the last one, and a `baseUrl` ending on `/` joins a path opening on one without doubling it |
| `http.check(status().is(200))` on the protocol | applies to every request: a small `request()` helper that sends and checks |
| `System.getProperty("baseUrl", …)`, `System.getenv(…)`, a Typesafe `ConfigFactory` | the settings of the run: constant variables (`__ENV`), secret variables for credentials; a load setting goes to the scenario, a threshold to `options.thresholds` |
| `http("Home").get("/")` | `http.get(\`${BASE}/\`, { ...params, tags: { name: 'Home' } })` — the `name` tag is the request name in the report |
| `http("Page #{page} / #{total}")` — a name built from the session | a fixed `name` (`'Page n / total'`): every distinct name is a line of its own in the report |
| `.basicAuth(user, password)` | `Authorization: \`Basic ${encoding.b64encode(\`${user}:${password}\`)}\`` from `k6/encoding`, the password in a secret variable |
| `.post(…).body(StringBody("…")).asJson` | `http.post(url, JSON.stringify(…), { headers: { 'Content-Type': 'application/json', Accept: 'application/json' } })` — `asJson` sets both |
| `.queryParam("origin.city", "besko")`, `.queryParamMap(…)` | the query string of the URL, each value through `encodeURIComponent` |
| `ElFileBody("create.json")`, `RawFileBody`, `PebbleFileBody` | upload the file with `upload_project_file`, read it once at init with `open('create.json')`, substitute its EL in the script — `template.replaceAll('${id}', id)` in plain quotes, since a template literal would read `${id}` as its own |
| a body a DTO builds — `toJson(CourtHouseRequestBody.builder()…build())` through Jackson | the object in JavaScript under its `@JsonProperty` names (`courthouse_name`), then `JSON.stringify`; `@JsonInclude(NON_NULL)` leaves the nulls out, `ALWAYS` keeps them |
| a helper building headers with a `switch` — `getHeaders(26)` | read every `case` it runs through: one without `break` falls into the next, whose headers it adds or replaces |
| a GraphQL query in a `StringBody` — `{"query": "mutation {\n …}"}` | the query in a template literal, sent as `JSON.stringify({ query, variables })`; a `200` carrying `errors` is a failure: check `r.json('errors') === undefined` |
| `formParamMap(map).asFormUrlEncoded()`, `formParamSeq(Seq(("k", "v")))` | the pairs as the body object: `http.post(url, { grant_type: 'password', … })` encodes it and sets the content type |
| `formParam`, `formUpload("file", "invoice.pdf")` | a form object; `http.file(open('invoice.pdf', 'b'), 'invoice.pdf')` for the upload |
| `asMultipartForm()` with `RawFileBodyPart("file", "data/ncsp.csv")`, `StringBodyPart("request", "…")` | the same form object, one key per part: `{ file: http.file(ncsp, 'ncsp.csv'), request: '…' }`, the file read once at init with `open('ncsp.csv', 'b')` and uploaded as a project file — opened in binary, it is sent whole and is no CSV variable to declare |
| `.authorizationHeader("Bearer " + token)` on the protocol | a secret variable holding the token, in the shared headers |
| an abstract simulation holding the protocol and the `setUp`, a subclass holding the scenario | one K6 file: the protocol and the load come from the base, the journey from the subclass |
| `check(status().is(200))` | `check(res, { 'Home 200': r => r.status === 200 })` |
| `check(status().in(200, 404))` | the same check, and `responseCallback: http.expectedStatuses(200, 404)` on that request — otherwise the 404 Gatling accepts counts in `http_req_failed` and trips its threshold |
| a request with no status check | Gatling checks it anyway — a status of 200 to 209 or 304 — and fails the request otherwise: write that check, or the failure only shows in `http_req_failed` |
| `jsonPath("$.id").exists()` | `check(res, { 'id': r => r.json('id') !== undefined })` |
| `jsonPath("$..errors").notExists()` — recursive descent | a walk of `res.json()` for the key, at any depth: GJSON has no `..` |
| `jsonPath("$.token").saveAs("token")` | `const token = res.json('token')` — K6 selects with GJSON (`data.0.id`), not JSONPath (`$.data[0].id`) |
| `jmesPath(…)` | the same, rewritten as a GJSON path or as plain JavaScript over `res.json()` |
| `regex("id=(\\d+)").saveAs("id")` | `const id = (res.body.match(/id=(\d+)/) || [])[1]` |
| `regex(…).findAll`, `.findRandom` | `[...res.body.matchAll(/…/g)].map(m => m[1])`, then one drawn at random |
| `css("a:contains('MacBook Pro')", "href")` — a link found by its text | `parseHTML(res.body).find('a').toArray().find(a => a.text().includes(name)).attr('href')`; the name from the feeder goes in the JavaScript, not in the selector |
| `css("input[name=csrf]", "value").saveAs("csrf")` | `parseHTML(res.body).find('input[name=csrf]').attr('value')` from `k6/html` |
| `headerRegex("Location", "(.*)").saveAs("url")`, then `.get("#{url}")` | `res.headers.Location`; Gatling puts a URL without a scheme after the `baseUrl`, so do the same: `url.startsWith('http') ? url : \`${BASE}${url}\`` |
| `header("Location").saveAs(…)`, `bodyString`, `substring(…)` | `res.headers['Location']`, `res.body`, `res.body.includes(…)` |
| `#{token}`, `${token}` | the JavaScript variable holding it — a template literal |
| `#{randomUuid()}`, `#{currentTimeMillis()}`, `#{items.random()}` | `crypto.randomUUID()` (the global Web Crypto API), `Date.now()`, `items[Math.floor(Math.random() * items.length)]` |
| `feed(csv("users.csv"))` | a CSV variable (`upload_project_file` + `create_csv_variable`), `row('users')` at the point of the `feed` |
| `pause(2)`, `pause(1, 3)` | `sleep(2)`, `sleep(1 + Math.random() * 2)` |
| `pace(10)` | measure the iteration and `sleep` what is left of the ten seconds |
| `group("Checkout") { … }` | `group('Checkout', () => { … })` |
| `repeat(5) { … }`, `foreach(list, "item") { … }` | a `for` loop |
| `asLongAs(cond) { … }`, `during(d) { … }` inside the scenario | a `while` loop — `during` over a clock you read with `Date.now()` |
| `doIf`, `doIfOrElse`, `doSwitch` | `if`, `switch` |
| `randomSwitch(60.0 -> a, 40.0 -> b)`, `uniformRandomSwitch`, `roundRobinSwitch` | a weighted `Math.random()` draw, an even one, `exec.scenario.iterationInTest % n` |
| `exitHereIfFailed`, `exitBlockOnFail { … }` | `if (!check(…)) return;` — ends this iteration, as the Gatling user ends |
| `before { … }` / `after { … }` | `export function setup()` / `export function teardown(data)` — once **per load generator**: a run on several recreates what `setup()` creates on each, so keep a one-off preparation (creating a collection, loading a dataset) out of the test when the run spreads |
| `session.set("page", 1)`, a value kept across the loop | a module-scope variable: per VU, it survives from one iteration to the next |

**An extraction that finds nothing fails the Gatling request**, and the chain goes on with an unresolved EL. Check
the extraction in K6 and end the iteration on a miss — `if (!check(ids, { 'productId found': i => i.length > 0 }))
return;` — rather than sending the next request with `undefined` in it.

**A failed check is not a failed request in K6.** Gatling marks a request KO when one of its checks fails; K6 records
the check apart and the request stays green. OctoPerf reports both — errors from the requests, assertion failures
from the checks — so a failing `status().is(200)` shows up as an assertion failure, not an error. Say it once when you
present the result.

### Feeders

`csv`, `tsv` and `ssv` feeders become CSV variables: upload the file with `upload_project_file`, declare it with
`create_csv_variable`, and read it with `row('<variable name>')` — quote the name literally, or the file is never
downloaded to the injector. A Gatling feeder file opens on its header line: pass the columns as `names` and set
`ignoreFirstLine`, so the header is not read as a row. The strategies:

| Gatling | K6 on OctoPerf |
|---|---|
| `.queue` (the default), `.circular` | `row(name)` — one row per iteration within the injector, wrapping at the end: `.queue` never runs out, where Gatling would stop |
| `.random` | `const rows = csv(name)` at init, `rows[Math.floor(Math.random() * rows.length)]` |
| `.shuffle` | `row(name)` over a shuffled copy of the file — or `.random`, which the user rarely tells apart |

Pick the `scope` of the CSV variable from what the feeder holds. A feeder handing each user a record of its own —
accounts, orders — is unique test data: `SHARED` (the default), which splits the file between the injectors. A
feeder walked whole — `foreach` over its records, `readRecords` turned into a list, the categories or pages every
user browses — is reference data: `PRIVATE`, which sends the whole file to every injector, and the script loops
over `csv(name)`. Never inline such a file as a constant array in the script: the data leaves the project, and the
next edit of the file no longer reaches the test.

A feeder over files the run finds on disk — an iterator over a directory, each record naming a file that
`RawFileBody("#{file}")` sends — reads files the upload does not carry: ask for them, upload them as project files
named on a pattern (`batch-1.json` … `batch-<n>.json`), and open them at init in a loop up to a constant variable —
`octoperf://skills/k6-scripting`, files opened at init. Each file sent once, by users sharing the iterator, is
`shared-iterations` over the file count.

A feeder whose file is named at run time — `csv("users-${getenv("ENV")}.csv")` — reads one file per environment: pick
the one of the target, and ask for it when the project does not carry it; a CSV variable names a fixed file.

`Random(seed)` drawing values from a seed of the feeder repeats the same draws for the same seed; `Math.random()` cannot be
seeded. Write a small seeded generator (mulberry32 is ten lines) when the reproducibility matters, and say that the
sequence is not the JVM's.

A counter generated in code — `generateSequence(1L) { it + 1 }`, `Iterator.from(1)` — is
`exec.scenario.iterationInTest + 1` from `k6/execution`: unique on one load generator; add `run.injectorId` from
`./octoperf.js` when several share the scenario. A random one (`Iterator.continually(Map("id" -> UUID…))`) is
`crypto.randomUUID()` in the iteration.

A `jsonFile` feeder becomes a `SharedArray` over `JSON.parse(open('users.json'))`. A `redisFeeder` reads through
`k6/x/redis`, a `jdbcFeeder` to Postgres or MySQL through `k6/x/sql` (the K6 binary carries both drivers); anything
generated in code is `CUSTOM_CODE`.

### Protocols the K6 binary carries

| Gatling | K6 module |
|---|---|
| `ws("…").connect(…)`, `sendText`, `await(…)(check)` | `k6/websockets` |
| `grpc(…)` (phiSgr/gatling-grpc) | `k6/net/grpc` — `client.load` over the `.proto`, or `client.connect(addr, { reflect: true })` |
| JDBC (galax-io plugin), `jdbcFeeder` | `k6/x/sql` with `k6/x/sql/driver/postgres` or `k6/x/sql/driver/mysql` |
| Redis | `k6/x/redis` |
| Kafka (galax-io plugin) | `k6/x/kafka` |

Anything else — SSE, JMS, MQTT, AMQP — is not in the binary: see `SSE` and `NON_HTTP_PROTOCOL`.

The plugins' constructs, each checked on the K6 binary against a real broker, database and server:

| Gatling plugin | K6 |
|---|---|
| `kafka("…").send(key, value, headers)` | `new Producer({ brokers, topic }).produce({ messages: [{ key, value, headers: { 'test-header': 'test_value' } }] })`, key and value through `registry.serialize({ data, schemaType: SCHEMA_TYPE_STRING })` |
| `.requestReply().requestTopic(a).replyTopic(b)` | produce on `a`, then `new Consumer({ brokers, topic: b, partition: 0, offset, maxWait })` and keep the message whose key is the one sent — the plugin matches a reply by key; `.check(jsonPath(…))` reads `JSON.parse(registry.deserialize({ data: reply.value, schemaType: SCHEMA_TYPE_STRING }))` |
| the topics a test writes to | created in `setup()` with `AdminClient.createTopic` when the broker creates none on demand |
| `jdbc("…").rawSql(…)`, `.insertInto(table, cols).values(Map)` | `db.exec(sql)`, `db.exec('INSERT … VALUES ($1, $2)', …)` — Postgres placeholders, in the column order |
| `.queryP("… WHERE ID = {id}").params(Map.of("id", 20))` | `db.query('… WHERE ID = $1', 20)`: each named parameter becomes a positional one |
| `.call("PROC").params(…)` | `db.query('SELECT PROC($1, $2)', …)` for a function, `CALL PROC(…)` for a procedure |
| `.batch(insertInto…, update…)` | the statements between `db.exec('BEGIN')` and `db.exec('COMMIT')` |
| `.check(simpleCheck(NonEmpty), allResults().saveAs(…))` | the rows `db.query` returns: `rows.length > 0` |
| `grpc("…").rpc(Service.getMethod()).payload(…)` | `client.invoke('package.Service/Method', message, { metadata, tags: { name } })`, the message in the JSON names of the `.proto` |
| `.check(statusCode().is(Status.Code.NOT_FOUND))` | `r.status === grpc.StatusNotFound` |
| `.check(extract(m -> m.getData()).saveAs(…))` | `res.message.data` |
| `.headerEL(key, "#{token}")` on the protocol | the `metadata` of every `invoke` |
| `.target(otherChannel)` | a second `grpc.Client`, connected to that address |

A plugin's own examples often start an in-process server in the simulation — `TestServer.startServer(8080)` in the
initializer — or run on an embedded database (`jdbc:h2:mem:`): they test the plugin, not a system. Migrate the
calls, and point them at the service the user actually tests.

## 4. Work the `unconverted` list

Every entry names a source and a line. Read the line in the project before acting: the `excerpt` is the line, not
the construct around it.

### `OPEN_MODEL_INJECTION` — decide the model before the numbers

An open injection step. §2 is the whole answer: a scenario that loops converts to a number of users, one that ends
converts by Little's law, with a measured journey duration, stated to the user as an approximation.

### `THROTTLING` — a ceiling, not a target

`throttle(reachRps(100).in(10), holdFor(1.minute))` targets a throughput. K6 on OctoPerf has no target, but the K6
profile has a ceiling: the `rps` setting of the profile's K6 engine (`octoperf://skills/scenario-composition`),
applied **per injector**. Divide the Gatling figure by the number of injectors, and say that a ceiling slows the
virtual users down rather than adding any.

### `SEQUENTIAL_SCENARIOS` — `andThen`

One scenario starts when another ends. Give the later one a `startTime` equal to the duration of the earlier: K6
starts it then, and `create_scenario_from_k6_options` turns it into the profile's delay.

### `ASSERTION` — thresholds, or an SLA

`assertions(global().responseTime().percentile3().lt(800), global().failedRequests().percent().lt(1.0))` judges the
whole run. Two places can hold it: K6 `thresholds` in the script's `options` — kept at run time, and shown in the
report — `{ http_req_duration: ['p(95)<800'], http_req_failed: ['rate<0.01'] }` — `successfulRequests().percent().gt(95.0)` is
`http_req_failed: ['rate<0.05']`, `responseTime().max().lt(2000)` and `mean()` are `'max<2000'` and `'avg<…'`; or an OctoPerf SLA profile
(`octoperf://skills/sla`) when the user wants it on the report and in trends. `forAll` judges each request on its own
where `global` judges them together: a threshold per request name — `'http_req_duration{name:get user}': ['p(95)<1000']`
— or the global one as an approximation, said as such. Gatling's `percentile3` is the
configured third percentile, 95 by default — check `gatling.conf` before writing `p(95)`.

### `RENDEZ_VOUS` — no barrier in K6

K6 virtual users never wait for each other. Leave it out and say so; if the synchronised burst was the point of the
test, a step in the load (§2) is the nearest equivalent.

### `TRY_MAX` — a retry loop

`tryMax(3) { … }` repeats a block until it passes. Write the loop: run the block, stop on success, give up after
three tries. Keep the attempts inside the iteration.

### `HTML_RESOURCES` — K6 fetches no embedded resource

`inferHtmlResources` and `resources(…)` fetch a page's images, scripts and styles. K6 fetches nothing it is not told
to. Either leave them out — say that the page weights and the CDN are then out of the test — or list the resources
the recording shows and fetch them with `http.batch` after the page. On a protocol serving an API — JSON, GraphQL —
`inferHtmlResources` fetches nothing, often left over from a recorder template: drop it, nothing is lost. The same
holds for `inferHtmlResources(BlackList(".*\.js.*", ".*\.css.*", …), WhiteList())` excluding every static type: what
is left are frames, rarely there.

`resources(…)` listing requests of the application itself — the API calls a single-page front end fires together
when a screen opens — is no static content: never drop it. Send the request and its resources in one
`http.batch([main, ...resources])`, which runs them in parallel as Gatling does; a value one of them extracts is
there for the next step, not for its siblings.

### `XPATH_CHECK` — no XPath in K6

Rewrite the check against the text: `parseHTML(res.body).find(…)` when the document is HTML, a regular expression
over `res.body` for a SOAP envelope. Keep the same thing checked, not the same expression.

### `SSE` — not in the binary yet

Server-sent events need an extension the OctoPerf binary does not carry (OctoPerf/octoperf#3466). Leave the stream
out and say it.

### `NON_HTTP_PROTOCOL` — JMS, MQTT, AMQP

No extension of the K6 binary speaks these. The requests around them convert; the messages do not. Report the count
per protocol and let the user decide whether a test without them still answers their question.

### `JDBC_DRIVER` — the database is not Postgres or MySQL, or nothing says which

The K6 binary carries the Postgres and MySQL drivers of `k6/x/sql`. The entry means the call names another database,
or passes its URL through a variable the scan cannot read — `AppConfig.DB_URL`. Find the URL in the project's
configuration: Postgres or MySQL, it converts; anything else, the feeder has to become a CSV exported once from that
database.

### `GRPC_WITHOUT_PROTO` — K6 needs the service description

The upload carries no `.proto`. K6 can still call the service if the server answers **reflection**
(`client.connect(addr, { reflect: true })`) — ask the user, or ask for the `.proto` files and upload them beside the
script.

### `CUSTOM_CODE` — the Java, Scala or Kotlin the simulation runs itself

A function over the session (`exec(session -> …)`, `pause(session -> …)`, `doIf(session -> …)`), a method reference
handed to `exec`, a feeder generated in code, an Expression given as a function — `StringBody(s -> …)`,
`.post(s -> base + s.getString("id"))` — which is the JavaScript expression building that value in the iteration. Read it and rewrite it in JavaScript — most are a few lines: a counter,
a date, a value computed from another. What it reads from the session is a variable of the iteration, or a
module-scope variable if it outlives it.

A field of the simulation that the functions write — `private var bearerToken`, a token cached for five minutes
— is shared by every Gatling user of the JVM. K6 virtual users share no memory: keep it at module scope, one copy
per VU (each VU then fetches its own token), or compute it once in `setup()` and hand it to every VU when it never
has to change during the run. Say which, since the number of calls to that endpoint changes.

Credentials written in the sources — a client secret, a password in an `object Configuration` — go to secret
variables. Ask the user for the values of the target rather than copying those of the repository.

The session is immutable: a function changes it only through the session it **returns**. Read the return, not every
`set` — in `exec { it.set("total", "#{total}"); it.set("page", 1) }` the first `set` is thrown away and `total` keeps
what `saveAs` stored; rewriting both as assignments changes what the script does. A `repeat { … }` or `doIf { … }`
taking a function reads its bound from the session: a loop bounded by a value of the response, usually.

### `EXTERNAL_DEPENDENCY` — a library the upload does not carry

An import from outside Gatling, the JVM languages and the project — a Jackson `ObjectMapper`, a signature library, a
company SDK. Find what the simulation does with it. JSON handling is plain JavaScript; a hash or an HMAC is
`k6/crypto`; a company's own signing scheme has to be reimplemented in JavaScript, or the request precomputed. This is
the entry that most often makes a project `MOSTLY_MANUAL`: say what the library does before promising a date.

A library that **encodes the payload** — Avro records built from generated schema classes, a Protobuf message sent
over HTTP — needs the schemas themselves (`.avsc`, `.proto`), which the upload rarely carries: ask for them. Avro's
JSON encoding can be written by hand from the schema; its binary encoding has no K6 module for an HTTP body
(`k6/x/kafka` carries Avro for the Kafka protocol only).

## 5. Import, scenario, validation

1. **Variables first.** `list_variables`, then create what the script reads: `create_constant_variable` for the base
   URL and plain settings, `create_secret_variable` for credentials, `upload_project_file` + `create_csv_variable` for
   each feeder, `upload_project_file` for each body template.
2. **Import.** `import_k6_virtual_user`, every file of the Virtual User in one request, then `describe_virtual_user`
   for the id and the link.
3. **Check.** `sanity_check_virtual_user` — zero credits. Fix every ERROR: a reserved name, a file named like a
   project file (the body templates are project files), a CSV name built at run time.
4. **Validate.** `validate_virtual_user`, poll `get_virtual_user_validation`, and read the K6 log with
   `list_bench_result_files` + `read_bench_result_file_lines` when a request fails. `octoperf://skills/validation-triage`
   groups the failures. The first run on a real project is rarely green: a GJSON path written as JSONPath, a CSV
   column named differently, a header the protocol block set once for every request.
5. **Scenario.** `create_scenario_from_k6_options` with the script's `options` in the shape of
   `k6 inspect --execution-requirements` — `{"scenarios": {…}}`, one entry per K6 scenario of §2 — the entrypoint's
   file name, and the provider and location to run on — the workspace's own from `list_docker_providers_by_workspace`,
   or a public one from `list_public_docker_providers`. Report every `unconverted` entry it returns
   (`octoperf://skills/k6-scripting`): an `OPEN_MODEL` there means an arrival-rate executor slipped through §2.

Present the result as **converted and validated**, never as converted: the requests that replayed green, the checks
that passed, and the list of what was left out or approximated — the open injection rebuilt as users, the
resources not fetched, the assertions moved to thresholds. Do not run the scenario until the Virtual User validates
clean — `run_scenario` burns credits.

## Related skills

- `octoperf-k6-scripting` — what a K6 script on OctoPerf can reach: `__ENV`, `./octoperf.js`, CSV variables, reserved
  names. Read it before writing the first file.
- `octoperf-scenario-composition` — editing the profiles `create_scenario_from_k6_options` created, the K6 engine
  settings (`rps`, `exec`, graceful stop).
- `octoperf-validation-triage` — the first validation will need it.
- `octoperf-sla` — for the Gatling assertions moved to an SLA profile.
