# The contract, in plain steps (any language)

`ncdev spec hostapi` is the authority; this page is the order to build it in.
An app that keeps every step passes `ncdev check`.

## 1. Being started

NativeCluster's host starts your **program** (the file nc-app.json's
`program` names, in the package folder) with:

| Variable | What |
|---|---|
| `LB_APP_SOCKET` | a unix socket path: serve your HTTP here, and nowhere else (no TCP port) |
| `LB_HOST_SOCKET` | a unix socket path: call the host here |
| `LB_SECRETS_STDIN=1` | your secrets arrive on stdin |
| `NC_APP_DATA` | your data folder, on the computer running this copy of the app (each person's own computer, or the main): keep every file here, nowhere else. The app's own folder is read-only once installed (its files must stay what the Mac owner vouched for), so nothing is ever written beside the program: no cache, log or database there |
| `NC_APP_BASE` | the path you are served at (`/`): links you write start with it |
| `NC_APP_SOCKETS` | a short folder of your own (emptied at each start) for unix sockets your own processes use between them: a socket's path holds about 100 bytes, so make them here, not in `NC_APP_DATA`. From host 0.3.5 (`min_host`); absent before |

- Read **stdin** once: one JSON object, `{"app_credential": "...", "ticket_key": "<64 hex>"}`.
  Keep both in memory only; never log or write them.
- If run as `<program> manifest`, print nc-app.json byte for byte and exit.
- The working folder is not yours: find your own files relative to the
  program, or embed them.
- On SIGTERM, finish quickly and exit.

## 2. Serving

- Listen on `LB_APP_SOCKET` (remove a stale file first; mode 0600).
- Put **one ticket check in front of every route** (below). A route
  registered anywhere else answers anyone.
- Answer `GET /healthz` with 200 (`{"status":"ok"}`): the host's readiness check.
- Answer `GET /.nc/whoami` with `{"person": "<the ticket's person, or \"\">"}`,
  nothing else (never an email or a name). It is how `ncdev check` proves
  the ticket check; the host never calls it in normal use.

## 3. The ticket (every request)

Header `X-NC-Ticket: <payload>.<signature>`, both base64url **without
padding**:

1. Split at the first `.`; no dot: refuse.
2. `signature` must decode (strict, no `=`) and equal
   `HMAC-SHA256(ticket_key_bytes, "nativecluster-ticket-v1\n" + payload)`,
   where `payload` is the base64url text as received. Compare in constant time.
3. Decode `payload` (strict) to JSON, an object, with each field of its
   type: `id`, `remote`, `sni`, `person`, `email`, `name`, `session`,
   `label` strings; `iat` an integer (Unix milliseconds); `tls`, `host`,
   `mcp` booleans. A field of the wrong type refuses the ticket; an unknown
   field is fine; `null` reads as empty.
4. `remote` must be non-empty; `iat` within 2 minutes of now, either way
   (exactly 2 minutes is still good).
5. Anything that fails: **403**, before any route runs.

The person is `person` (an id the host made for this app; key everything
by it), or nobody when it is empty. **Nothing else names a person**: no
header, cookie, parameter or body field, ever. `email` and `name` come only
if the manifest asks `people.read` (a stop-and-ask item).

`ncdev spec vectors` has 24 cases with their right answers; read each one
with its key and `now_ms` and check your reading matches before going on.

## 4. Models, through the host

Every call to `LB_HOST_SOCKET` carries `Authorization: Bearer <app_credential>`.

1. **Which model:** `GET /host/v1/models?task=<task>&kind=<chat|embedding|transcription|tts|image|diarization>`
   gives `{"models": [...], "to_get": [...]}`, and for `chat` also
   `"windows": {"<model>": <tokens>}`: the window the host runs each one's
   calls in, prompt and answer together (the smallest of the computers'
   that may run the call, so it fits wherever the call runs; ask again
   when a long session starts, since a computer joining can lower it).
   Size every prompt from it, leaving
   your `max_tokens` of room for the answer, and never assume a window of
   your own; don't send `num_ctx` (the host sets it, and only lowers it to
   what you ask). Call `models[0]`, or any model in `models` (each meets
   the task): to offer a faster or a better one, choose within `models` by
   its `facts` (`speed` here in tokens a second, `rating` with its board;
   models.md, "A faster or a better model"). Empty
   `models`: no model on this Mac may do it now (an optional task's feature
   stays hidden; a required one says it is unavailable, plainly). For a
   task with `min_window`, only models that give that window where they run
   are named, and while none does `"why"` holds words to show instead. `tts`
   (reading aloud) and `image` (painting pictures) answer only where the
   Mac's engine runs such a model, with `"voices"` (tts) or `"sizes"`
   (image) for `models[0]`: take them from there, never a fixed list. Ask
   speech for the language you will read, `&lang=<tag>` (`en`, `en-gb`,
   `de`): the answer then holds only the models and voices that read it,
   and an empty one means no voice reads it yet, so that language's
   reading aloud stays hidden. When `models` is empty and the answer names
   `get` (`{model, bytes, page}`), show a button ("Get the voice model,
   185 MB") that opens `get.page`: NativeCluster's own sheet asks a person
   to download it. Never download a model yourself. Ask only a `response_format` the answer's
   `formats` names (an engine that names none takes `wav`, `pcm`, `aac`).
   Their calls are OpenAI's `POST /v1/audio/speech` `{model, input, voice,
   speed, response_format}` (audio bytes back, with their `Content-Type`,
   `X-Audio-Samples` and `X-Audio-Duration-Ms`) and `POST
   /v1/images/generations` `{model,
   prompt, size, n, response_format: "b64_json"}`, through the line like a
   chat call (`ncdev spec hostapi`, "Speech and pictures"). Write them only
   behind a non-empty answer: `ncdev check`'s fake host answers both empty.
2. **A turn in line:** `POST /host/v1/line/wait` with
   `{"kind":"acquire","lane":"P","tag":{"Person":p,"Task":task,"Model":model},
   "ticket":{"origin":"request","person":p,"request":<the ticket's id>}}`.
   The answer is lines of JSON: `{"position":n}` while you wait, then
   `{"id":grant,"cache_key":k}`, or `{"refused":why}`. The tag's fields are
   strings except `Job`, a boolean (`true` for a job's unit, never the job's
   id); a field of the wrong type is a 400 that names it
   (`bad request: tag.Job is a boolean, not a string`).
3. **The call:** `POST /v1/chat/completions` (or `/v1/embeddings`,
   `/v1/audio/transcriptions`), OpenAI-shaped, with `"model": model`,
   header `X-NC-Grant: <grant>`, and `"cache_key": k` in a chat body. Add
   `"think": false` to a short, bounded chat call (a paragraph's
   translation, a title, an extraction): a model that reasons otherwise
   spends most of the call thinking first. Leave it out where the answer
   needs reasoning. Either way, keep only what follows the last `</think>`
   in an answer: a model told not to think may still plan at the start of
   the answer and end it with a bare `</think>`. With thinking on, give the
   call room: a model can spend all of `max_tokens` deliberating and answer
   empty (gemma4:12b, 5 of 8 short translations at 2048), and the engine
   still says `"finish_reason": "stop"` then. Read an empty answer as cut
   short, never as the model's answer.
   When the answer has a shape (a translation per paragraph, fields to
   extract), send `"response_format": {"type": "json_schema", "json_schema":
   {"name": …, "schema": {…}}}`: the model can then write nothing but that
   JSON, from its first token, so no preamble, notes or planning reach it.
   Every property an object declares comes back, in order; `pattern`,
   lengths and item counts are not held, so still check the answer. Not
   with `tools`, which leave the answer unconstrained.
4. **The grant back, always:** `POST /host/v1/line/release` with
   `{"grant":grant,"notes":{"model":model,"calls":1,"read":<prompt tokens>,"written":<answer tokens>}}`.
   Counts only, never words.
5. A `409` with `X-NC-Stopped` or `X-NC-Lost` (or that trailer at the end
   of a stream) means the line made way for someone: release, and run the
   call again (up to 3 more times). A stream already shown in part ends as
   stopped instead.

Long work a request starts (a whole book's translation) is a **job**. Each
of its calls waits with
`{"kind":"acquire","lane":"B","tag":{"Person":p,"Task":task,"Model":model,"Job":true,"Conversation":<the job's id>},
"ticket":{"origin":"job","from":"request","person":p,"request":<id>}}`;
after your app restarts, the ticket is
`{"origin":"job","from":"resume","person":p}`. `ncdev spec hostapi` shows
one unit whole, from the wait to the release. Keep its progress in
`NC_APP_DATA` so a restart resumes it.

**Tell the host your job's units.** `POST /host/v1/units`
`{"waiting":true}` while any job waits or runs (and `"pause_ends"` for a
paused one that resumes by itself), `{"waiting":false}` once none does:
after every change to your jobs, at start, and on your way out. While work
waits the host keeps the app running, and if the app stopped (a restart of
the host, a crash) the host starts it again so the job goes on with no one
opening the app. Waiting in line also counts as in use: the app is never
stopped for being idle while it waits for a turn.

A unit like that (one wait, one chat call, the release) may run on any of
the group's Macs that holds its exact model loaded: the main, or a Mac lent
to it. Its grant then carries chat calls to that model only; anything else
on it answers `403` "wait again for this call". Units that do not depend on
each other (a book's chapters) can run at once, each with its own
`Conversation` (`<job id>/<chapter>`): run as many as `GET
/host/v1/line/state` says in `seats_for[model]` (1 when absent), never
more, since units taking turns on one seat only re-read each other's
prompts. Units that do depend on each other (a chapter's paragraphs) stay
one after another. Add `"pick":"chat"` to the wait and leave the tag's
`Model` out to have the host choose the model for the task; the answer then
names it (`"model"`), and the call is sent with it.

**A `503` with `Retry-After` means come back shortly.** The host answers a
model call `503` with `Retry-After` (seconds) while it restarts the engine
after a model stopped responding (code `engine_restarting`), or while the
model is busy. Wait the seconds it says and call again, a few times, before
treating the call as failed. A model taken out of service after stopping
twice answers `503` with a longer `Retry-After` naming the model that serves
instead: ask the models route again and use the next model it lists. Never
retry a `400`.

**A failed call stops the job, never fills it.** After the retries above, a
call that still fails (the host or the model refused, timed out, answered
nothing) stops the job where it is, kept and resumable, and says so on the
app's jobs page. Never write stand-in text (the source untranslated, an
error string, an empty paragraph) into the job's output to carry on: a
finished job is one whose every unit the model actually did.

## 5. What it reaches

- The host's socket, its own socket, `NC_APP_DATA`, and unix sockets of its
  own in `NC_APP_SOCKETS` between its own processes. Nothing else on this
  Mac: no `localhost`, no other port, no other model server, and NativeCluster's own
  ports (8484-8487, 8490-8499) on no computer. NativeCluster runs every app
  in a sandbox that enforces this on a Mac: the app reads only its own
  package and writes only `NC_APP_DATA` (its `HOME` and `TMPDIR` point
  there), runs only its package's programs and the system's own tools, and
  from 0.3.7 reaches any other computer itself (below). A read or write
  elsewhere fails with "operation not permitted"; `ncdev check` runs the
  manifest step in the same sandbox. See `ncdev spec`, "The app sandbox".
- From 0.3.7, any server it calls, on the internet or the office network (a
  NAS), on any port, whether or not a `net.*` permission names it: an app's
  developer may call any server they want, with no restriction from
  NativeCluster (`ncdev spec`, "Any server it calls").
  Where the app's information lives is the app's decision. A hosted model's
  service is the app's to call too, and `ncdev check` warns that such
  inference leaves the group's machines: call models through the host to
  keep them there. Say `min_host` 0.3.7 if the app reaches a server
  itself. Before 0.3.7 the rules below held:
- The internet only at hosts a `net.*` permission declares, over https, and
  only while the plane is on. A plane is the app's: on for everyone given
  the app once it is installed, with no switch. An older host (0.3.0) reads
  it as each person's own, off until they turn it on, so the app still asks:
  - **Before each send**, ask `GET /host/v1/grants?person=<the ticket's
    person>` (with the credential) and send only if its `on` list names the
    plane: `{"person": "p_…", "on": ["net.books"]}`. Ask every time, never
    cache it: a plane an older host reads as off stops the next send. A
    firing for a person asks as that person; a send for no one (a firing
    of a schedule kept for no person) sends nothing yet.
  - **As it starts**, read the `off_switch` name from the environment: `0`
    means the plane is off for everyone, whatever the route says.
  - From 0.3.6 the host holds the names: the app's environment has
    `HTTPS_PROXY` (and `NC_EGRESS`) set to its own gate on loopback, the
    only address its sandbox reaches, and the gate opens a tunnel only to a
    host a plane names (`ncdev spec`, "Through the host's gate"). Use a
    client that reads `HTTPS_PROXY` (Python's httpx and requests do; in Go,
    the kit's `appnet.NetClient`; aiohttp needs `trust_env=True`; websocket
    clients take the proxy as an option), and resolve no name yourself.
    Never set `HTTP_PROXY` or `ALL_PROXY`: plain http is the host's socket.
    Say `min_host` 0.3.6: the sandbox holds an app to its gate alone only
    then (one made for an older host keeps port 443 too). Up to 0.3.5 the
    sandbox held a dial to port 443 but not to a name, so which declared
    host the app reached was its promise; `ncdev check` still tests that
    best-effort by answering every item off and failing the app if it
    reaches anything.
- A person's role, before letting them change what affects everyone (a
  setting for the whole group, someone else's work): ask
  `GET /host/v1/people/<the ticket's person>/role` (with the credential):
  `{"person": "p_…", "role": "admin"}`, one of `owner`, `admin`, `member`.
  Show and accept such a change only for `owner` or `admin`, asking each
  time, never caching it. It needs no `people.read`.
- Pages load nothing from outside: vendor every script, style and font.

## 6. Schedules

Work on a timer, a schedule or an event runs on the host's clock, on the
computer the app runs on: a person's own computer for their copy, the main
for the main's. A schedule missed while that computer was off runs once
when it starts again. The app keeps no timer, cron thread or scheduler of
its own: it tells the host what to wake it for, and the host wakes it
(starting it first if it rests). Why the app sets a schedule
is its own business: for a person's request, an agent's own plan, another
firing's work.

- **Declare** each kind in `triggers`
  (`{"id":"reminder","kind":"time","name":"your reminder"}`; `event` for
  "when something happened"), name every one in `compute.background`, and
  set `"min_host": "0.3.5"` or newer (`ncdev check` says so).
- **Set** schedules on the host's socket, with the credential: `POST
  /host/v1/triggers/table/set` `{"person": <the ticket's person>, "list":
  [{"ID": "reminder:<key>", "Trigger": "reminder", "Spec": {...},
  "Enabled": true}]}` replaces that person's list; `"person": ""` is the
  app's own upkeep. `put` `{"person", "trigger": {...}}` sets one row and
  `remove` `{"id"}` deletes one, leaving the rest. `Spec` is one of
  `{"cron":"0 9 * * 1-5"}` (five fields, digits and `* / , -` only),
  `{"hour":8,"minute":30,"weekdays":"1111100"}` (Monday first), `{"every":
  <seconds>}` (at least 60), `{"at": <Unix seconds>}` (once: a reminder),
  or `{"event":"notes.saved","min_gap":<seconds>}`; times are the local
  time of the computer whose clock fires them, and one trigger fires at
  most once a minute. Also `list`,
  `get`, `enable` `{"id","on"}`, `defer` `{"id","at"}`, `fire` `{"id"}`
  (run it now), `emit` `{"event","person"}`, `pause`/`resume`
  `{"person","by":"person"}`, `firings`, `counts`, `last`, `clear`.
- **No words in the host:** an id is the trigger, a colon and a key of
  lowercase hex or digits (`reminder:3f9a`, `reminder:42`); an event is a
  short lowercase name. Keep what each schedule does (the prompt, the
  target) in `NC_APP_DATA`, keyed by its id. A person named is one of the
  app's people, by the id its tickets name them by; pausing for everyone or
  as the owner is the group's administrators' alone (403).
- **Answer each firing:** `POST /nc/v1/wake` comes with the host's own
  ticket (its `host` field true; answer anyone else as a route that does
  not exist) and an `appsdk.Firing` `{"id","app","trigger_id","person",
  "trigger","due"}`. Start the work and answer at once with the jobs it
  started, `{"jobs": ["<job id>", ...]}` (at most a minute, or the host
  hands it over again later). The same firing may come again: answer the
  jobs it already started (keep firing id -> jobs in `NC_APP_DATA`). A
  firing for a trigger you no longer keep answers `{"jobs": []}`.
- **Long work for a firing:** answer `{"jobs": [...], "running": true,
  "lease_secs": 600}` instead, and the host keeps the firing for you: do the
  work, `POST /host/v1/triggers/firings/renew` `{"firing"}` every third of
  the lease, and `POST /host/v1/triggers/firings/done` `{"firing",
  "outcome": "done"}` (or `"failed"`) when it ends. A `410` on a renewal
  means stop (its person was turned off or its work paused). If your app
  crashes or is stopped, the host hands the firing over again (`"attempt"`
  counts how many times before), so resume from what you kept in
  `NC_APP_DATA`. Go: `appsdk.WakeHandler` and `appsdk.RunFiring` do this;
  Python: the kit's `on_wake` and `nativecluster.run_firing`.
- **Loops** (an agent working on a task round after round): declare a
  trigger of kind `loop`, start one for a person with `POST
  /host/v1/triggers/table/loops/start` `{"person", "loop": {"ID":
  "agent:<key>", "Trigger": "agent", "rounds": 50, "gap_secs": 0}}`. Each
  round comes as a firing with its `"iteration"`; answer it with
  `"outcome": "continue"` for another round, `"done"` when the task is
  finished, `"failed"` to give up (in the wake's answer, or in the done of
  a round you took on). A round with no outcome ends the loop. Stop one
  with `loops/stop` `{"id"}`. The host caps rounds and hours only to catch
  a loop that never ends; the line decides how much it gets, behind people
  and taking turns with other apps.
- **Its work waits as the firing:** each wait in line carries
  `{"origin":"firing","firing":<id>,"person":<the firing's person>}`, or a
  job's `{"origin":"job","from":"firing","firing":<id>,"person":...}`, in
  lane `B` (or `I` for work that waits for an idle Mac). It waits behind
  people and takes turns with other apps; the host decides how much it
  gets.
- **While the person's computer sleeps** (from 0.4.13, `min_host`
  `0.4.13`): some things go to the same app on the group's main, the
  computer assumed always on. That is work that must happen on time with
  that computer shut, or shared data the user chose to keep in the main's
  copy (`reference/stop-and-ask.md`, items 4 and 8). `POST
  /host/v1/main/hand` on the host's socket, with the credential, carries
  one JSON value of at most 256 KiB to the main's copy, as one of the
  computer's signed calls. That copy gets it as `POST /nc/v1/hand`, with
  the host's ticket naming the person, and its JSON answer, given within a
  minute, comes back as the answer. For an app made for each person, the
  person is the process's own; for any other app, it is the person the
  computer is registered to. The main's copy keeps what it was handed in
  its own `NC_APP_DATA`, and for a handed job sets its schedule on the
  main's clock. Neither host reads or keeps the body. A refusal is
  `{"code", "words"}` (`words` to show the person). `409` `no_main` means
  this computer is the main or has none: keep the work yourself. `503`
  `away` means the main is out of reach: keep it and hand it over again
  later, since the host keeps no copy. The others are `404`
  `app_not_on_main`, `403` (the main refused the computer or the person),
  and `413` `too_large`. `ncdev spec hostapi`, "Handing the main something
  to keep", has the whole table; in Python, the kit's
  `host().hand_to_main(payload)` raises `Away` and `NoMain`.
- **The beat:** while it runs, an app with triggers gets `POST
  /nc/v1/tick` every 15 seconds, for upkeep that needs no firing (a paused
  unit whose pause ended). Answer it, or answer 404.
