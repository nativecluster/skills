# Any language

The contract is HTTP on unix sockets, one HMAC, and JSON: every language
has them. `ncdev check` holds the result to the same contract whatever it
is written in.

## Go (recommended)

One binary, no runtime to ship. `ncdev new <dir>` writes a complete
example (`main.go`): copy its ticket check, its socket server and its host
client, and `ncdev package` builds and zips it.

## Python

Use NativeCluster's Python kit (Python 3.9 or newer, nothing but Python
itself). `ncdev new <dir> -lang python` writes a whole app on it that
passes `ncdev check`, the kit beside it as `vendor/nativecluster` with
`vendor` on `sys.path`: start from it, or copy the kit's folder from there
into an existing app.

- **The main:** `nativecluster.run_wsgi("<path to nc-app.json>", app,
  on_wake=on_firing)` for a WSGI app (Flask, Django, a function), or
  `run_asgi` for an ASGI one (FastAPI, Starlette; it runs uvicorn, which
  the package carries). It answers `<program> manifest`, reads the stdin
  secrets, puts the ticket check in front of every route, answers
  `/healthz`, `/.nc/whoami` and (with `on_wake`) the host's firings, serves
  on `LB_APP_SOCKET` (0600, never a TCP port) and leaves on SIGTERM.
- **The person:** `nativecluster.person_of(request)` (the ASGI scope, the
  WSGI environ, or the framework's request), and nothing else.
- **Models:** `nativecluster.models().chat("<task>", messages)`,
  `.chat_stream`, `.embed`, `.transcribe`, `.choices`, `.window`. Each waits
  in line as whatever started it: the gate sets the request's origin, a
  firing's handler runs with the firing's. Work moved to a thread of its
  own carries it with `nativecluster.bind(fn)`, or names it with
  `acting_for(for_job(person, request=...))`; an agent's turn its person
  waits for says `for_job(person, request=..., waits=True)` (their lane
  while the host lets the request's turn last). A framework that runs its
  work on threads and pools of its own takes it along with
  `nativecluster.carry_origin_into_threads()` once at start, and
  `nativecluster.otherwise_acting_for(fn, conversation)` names what a call
  made with no origin set is for: fn() asked at every call (the turn its
  person last started, say), and the conversation it continues, which
  keeps that conversation's reading warm.
- **OpenAI's library** stays, through the kit's transport:
  `OpenAI(base_url=transport.BASE_URL, api_key="nativecluster",
  http_client=transport.http_client())` (`from nativecluster import
  transport`; `async_http_client()` for `AsyncOpenAI`). The call's `model`
  is the task's id, or `<task>/<model>` for a model a person picked from
  the task's `models().choices(task)` (a model menu; one the host no
  longer names gives way to the task's first); every call waits in line
  and gives its grant back, a call the line stopped runs again, and a
  prompt longer than the window is OpenAI's `context_length_exceeded`.
  To stop a call from another thread (its person closed the window while
  the agent's thread waits in line or reads the answer), call
  `transport.cancel(client)`: the wait leaves the line, the model stops
  writing, and the call raises `transport.Cancelled` (OpenAI's
  `APIConnectionError`); the client takes no model call after, so make a
  new one.
  `ncdev check` refuses `openai` only where the kit's transport does not
  carry it.
- **Its own processes** (an agent's gateway, workers, a code kernel) talk
  over unix sockets in `nativecluster.sockets_dir()`, a short folder of
  the app's own; `asyncio` and `socket.socketpair()` work as they are.
- **Schedules:** `nativecluster.host()`: `put_schedule(person,
  schedule("reminder:3f9a", "reminder", cron("0 9 * * 1-5")))`,
  `start_loop`, `post_units`, `net_on`, `role`. Long work for a firing:
  answer `WakeAnswer(jobs=[...], running=True)` and run it with
  `run_firing(host(), firing, work)`.
- **The package must carry its own Python** and every dependency, so the
  program runs on a Mac with nothing installed: a launcher script as the
  program, beside a bundled Python runtime and the app's code
  (`ncdev package -include <folder>` for each). Weigh that against
  rewriting in Go, and say which you chose. The app's folder is read-only
  when it runs: the host sets `PYTHONDONTWRITEBYTECODE=1`; never write
  `__pycache__` or anything else beside the code (`ncdev check` fails if
  `<program> manifest` does). The example's launcher (`notes`) answers
  `manifest` with `cat`, so that step needs no Python, and runs Python with
  `-I -B`. A standalone CPython (an `install_only` build of
  python-build-standalone) runs in the app sandbox as `python/` in the
  package: copy it with its links followed (`cp -RL`), since a package holds
  no link, and install the app's dependencies into it. `ncdev check` reads
  a Python the package carries as a dependency: its installed packages'
  names (a model runtime or a hosted model's client among them still
  fails), never its code.

## Node.js and TypeScript

There is **no JavaScript kit** yet: the app does what the Python kit does
itself, from `ncdev spec hostapi`.

- `server.listen(process.env.LB_APP_SOCKET)` with `http.createServer`
  (Express: `app.listen(path)`).
- `crypto.createHmac("sha256", key)` and `crypto.timingSafeEqual`.
- `http.request({ socketPath: process.env.LB_HOST_SOCKET, ... })`.
- **Every model call waits in the one line**, in three steps on
  `LB_HOST_SOCKET` (`ncdev spec hostapi`, "The waiting line", has an
  example):
  1. `POST /host/v1/line/wait` with `{"kind": "acquire", "lane", "tag",
     "ticket"}`: the answer is NDJSON, `{"position": n}` while it waits,
     then `{"id", "cache_key"}`; the `id` is the grant.
  2. The call itself, `POST /v1/chat/completions` (or embeddings,
     transcriptions) with `X-NC-Grant: <grant>` and the `cache_key`.
  3. `POST /host/v1/line/release` with `{"grant", "notes"}`, always, the
     call failed or not.

  A `409` with `X-NC-Stopped` or `X-NC-Lost` means the line stopped the
  call: wait again and run it again. A `410` "stopped by an administrator"
  is an ending: show it as stopped, never retry.
- **An agent that makes its own model calls** (an OpenAI-shaped client, the
  Vercel AI SDK's `@ai-sdk/openai-compatible`) keeps them by being given a
  `fetch` that does the three steps around each call it makes: the base URL
  is the host, the model is the task's id. That is what the Python kit's
  transport does for OpenAI's library: read
  `vendor/nativecluster/transport.py`, which `ncdev new -lang python <dir>`
  writes beside its example, and do the same, its retries and its context
  errors included.
- Like Python, the package carries its own Node (or a compiled Bun or Deno
  program) and `node_modules`. `ncdev check` cannot read inside a compiled
  program (SKILL.md, Phase 7): read its dependencies yourself.

## Every language

- One check in front of every route, then `/healthz` and `/.nc/whoami`
  behind it.
- An app with schedules answers `POST /nc/v1/wake` behind the same check,
  for the host's own ticket only (its `host` field true), and may answer
  `POST /nc/v1/tick` (`reference/contract.md`, "Schedules"). Long work for
  a firing answers `"running": true`, renews and says done: a background
  thread (Python) or a promise (Node) beside the server, the progress in
  `NC_APP_DATA`.
- Read `ncdev spec vectors` with your ticket code and compare every case.
- Run `ncdev check` early and often: it starts the program as the host does.
