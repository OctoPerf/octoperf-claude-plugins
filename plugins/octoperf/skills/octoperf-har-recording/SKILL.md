---
name: octoperf-har-recording
description: Use when a Virtual User has to be built from real browser traffic — "record my application", "capture a HAR", "build a VU from what I do in the browser", "import this .har", "record the checkout journey". Covers driving the recording yourself with Playwright (the only route that names the containers), what to tell a user recording by hand in Chrome / Firefox / Fiddler / Charles, and how to import either. Requires the OctoPerf MCP server.
---

# OctoPerf — Recording a HAR and importing it

A HAR is a list of HTTP exchanges. `import_har_virtual_user` turns it into a
Virtual User whose actions are grouped into **containers** — the steps a
reader sees, and the unit a report measures. Everything below is about
getting those containers right, because a Virtual User whose containers are
called `/api/v2/cart/items` is one nobody can read or maintain.

## 1. Decide who records

**If you can drive a browser** (Playwright available in your environment),
record it yourself. It is the only route where the containers get real names,
and it costs the user nothing but a description of the journey.

**If you cannot**, the user records by hand and you import the file — go to
section 5 and tell them which recorder to use.

## 2. Why an automated recording needs a container timeline

OctoPerf has three ways to build containers. Two of them are automatic and
**both collapse on a scripted recording** — this is measured, not assumed:

| Strategy | How it groups | On a Playwright recording |
|---|---|---|
| `PAGE_REF` | one container per `page` in the HAR | Playwright writes **one `page` per `Page` object**, not one per navigation → **a single container** holding the whole session |
| `THINKTIME` | splits on a gap above 3 s | a script never pauses that long → **a single container**, named after a URL path |
| container timeline | the names you supply, with their timestamps | one container per step, named as you named it |

So the timeline is not a refinement. It is what makes a scripted recording
worth importing at all. And it is exactly what a script can produce and a
human cannot: the script knows the wall-clock time of every step.

## 3. Record with Playwright

Two artefacts from one run: the HAR, and a timeline naming each step.

```js
const context = await browser.newContext({
  recordHar: {path: 'session.har', content: 'embed', mode: 'full'}
});

const entries = [];
const step = async (name, fn) => {
  entries.push({type: 'ENABLED', name, startTimestamp: Date.now()});
  await fn();
};

const page = await context.newPage();
await step('Home', () => page.goto('https://shop.example/'));
await step('Search shoes', async () => { … });
await step('Checkout', async () => { … });

await context.close();                       // flushes the HAR
fs.writeFileSync('timeline.json', JSON.stringify({entries}));
```

Five things decide the result:

- **Stamp the first entry before the first navigation.** Traffic older than
  the first timeline entry lands in no container, and depending on the platform
  version it costs the whole import rather than that one request — so treat
  this as a rule, not a precaution. Push the first entry before `newPage()` if
  anything at all may load on its own.
- **`content: 'embed'`, never `'attach'`.** `attach` writes a **zip**, which
  the import refuses with a raw Jackson 400. `embed` inlines the response
  bodies — on **both** Chromium and Firefox, so the engine no longer decides
  whether you get them.
- **Name steps after the business action**, not the URL: `Checkout`, not
  `/actions/order.action`. These names become the container names, and they
  are what the report will show.
- **`type: 'DISABLED'`** for a segment to import but not replay — a one-off
  sign-in, an analytics burst. It becomes a disabled container: visible, inert.
- **`Date.now()` needs no conversion.** It is the same clock as the HAR's
  `startedDateTime`; the backend matches each request into
  `[entry, next entry)` directly.

## 4. Import it

1. `import_har_virtual_user(projectId)` → a presigned `url`, valid ~5 minutes,
   single-use. Set the options here — see the table below.
2. POST `multipart/form-data` with **two parts**: `file` (the HAR,
   `application/json`) and `containerTimeline` (the timeline JSON as a
   string).
3. The response is a raw VirtualUser. Read its `id` and call
   `describe_virtual_user` for the compact listing and the UI deep-link.
4. **Read the container names back.** One container holding everything means
   the timeline did not take — check that its first entry really precedes the
   first request.

| Option | What it does |
|---|---|
| `adBlocking` | `ENABLED` by default: drops requests matching ad/tracker blocklists. Disable to keep everything. |
| `resources` | `REMOVE` drops images / CSS / JS and lets the runtime fetch them; `KEEP_ALL` keeps every request as its own action — heavier Virtual User. |
| `thinktime` | `THINKTIMES` puts the recorded pauses on each request; `DELAYS` turns them into delays between containers. |
| `maxThinktimeMs` | caps a pause, so a coffee break in the middle of the recording does not become a 4-minute wait. |
| `virtualUserId` | merges into an existing VU instead of creating one. |

Then `validate_virtual_user` before trusting it: a recording replays only as
well as its dynamic values survive, and `octoperf-auto-correlation` is what
fixes the ones that do not.

## 5. When the user records by hand

Say which recorder and what it implies, rather than leaving them to find out
after the import.

| Recorder | Containers come from | Watch out |
|---|---|---|
| **Firefox** | think-time gaps above 3 s | Response bodies are included. Truncated? raise `devtools.netmonitor.requestBodyLimit` / `responseBodyLimit` in `about:config` — `0` and `-1` do **not** mean unlimited |
| **Chrome / Edge / Opera / Safari** | the `pages` the browser writes | **Responses are not saved** when `preserve log` is on. Either click each response before exporting, or use Firefox |
| **Fiddler** | think-time gaps | HTTPS needs "Decrypt HTTPS Traffic" + trusting its root certificate. Set `prefs set fiddler.importexport.HTTPArchiveJSON.MaxTextBodyLength 10000000`, or bodies are dropped. Export as **HTTPArchive v1.2** |
| **Charles** | think-time gaps | Enable SSL Proxying with a `*` host entry, trust its root certificate |

Response bodies are not a detail: variable extractors and correlation rules
read them, so a HAR without them gives a Virtual User that replays a recorded
session verbatim and breaks on the first token.

**The manual container timeline** — OctoPerf's own, in the web UI — is the
same mechanism as section 3, driven by hand: the user names each step in a
popup while browsing. It is worth suggesting for a journey with clear steps,
with one warning: **it lives in the browser and is lost on refresh or on
leaving the page**, so the import has to be finished in the same sitting.
