---
name: nativecluster-app
version: 0.1.0
description: |
  Turn an app, in any language, into a NativeCluster app: one that runs on a
  group's own Macs, reaches models only through NativeCluster's host, knows
  people only by the host's ticket, and keeps its data on the group's main.
  Use when the user says "make this a NativeCluster app", "port this app to
  NativeCluster", "build an app for NativeCluster", "use the nativecluster
  skill", "package this for my group", or asks how an app reaches
  NativeCluster's models. Iron Law: never invent an answer to a stop-and-ask
  item; never run or download a model yourself; reach nothing on this Mac
  but the host's socket.
---

# Make a NativeCluster app (read -> map -> ask -> manifest -> adapt -> check -> try -> package)

NativeCluster runs a group's apps on its own Macs. Every model call goes to
the group's own models through NativeCluster's **host**, so nothing a person
writes reaches a hosted model; the host signs every request with a **ticket**
that names the person; an app keeps its data in the folder the host gives it.
An app may be written in any language. What makes it a NativeCluster app is
that it keeps **the contract**, and `ncdev check` proves it does.

You need `ncdev`, NativeCluster's tool for building apps, which comes with
NativeCluster (0.4.19 and later): `ncdev` on the PATH once the person has
chosen **Install the nativecluster command…** in NativeCluster's menu, else
`/Applications/NativeCluster.app/Contents/Resources/ncdev`. Everything exact
comes from it, never from memory:

    ncdev spec hostapi       the contract: sockets, ticket, routes, wire shapes
    ncdev spec vectors       how the ticket must be read, case by case
    ncdev spec schema        nc-app.json's fields
    ncdev spec permissions   every permission an app may ask for
    ncdev models <dir>       which models NativeCluster offers for each task
    ncdev check <dir>        the contract, proven (static and running)
    ncdev package <dir>      the package to install
    ncdev run <dir>          the app behind a NativeCluster of its own, to try it
    ncdev install <dir>      install it on this Mac (local), and read NativeCluster's answer

Read `reference/contract.md` and `reference/platform.md` (what NativeCluster
already does, so the app never rebuilds it) first; the other reference files
are for the phase that names them.

**This skill and `ncdev` come with NativeCluster, and match its version.**
NativeCluster puts this skill in the coding agent's skills folder and brings
each update's version, so use the `ncdev` of the NativeCluster on this Mac,
never a copy from elsewhere: an older one carries an older contract (a port
once read one from before 0.3.7's open internet, and built the wrong app).

## A cluster app, or a tool that runs as the person

What decides it is **where each piece of data is stored**, not whose it is.
A coding agent may run outside the sandbox because the person's own laptop
holds the code, and only its model calls go to the inference computers,
which hold no copy of it; a computer that did hold the code, beside other
people's data, would have to sandbox it:

- Data a person's work keeps **on a cluster computer** (the main, an
  inference computer), beside other people's, belongs to a **cluster app**,
  which runs in the sandbox, at least for those parts. This skill builds
  that kind.
- Data kept **only on the person's own device**, where only model calls
  leave it (NativeCluster keeps none of their words), can belong to a
  **tool that runs as the person**, outside the sandbox, as a coding agent
  must (it needs the person's folders and developer tools).

**A tool that runs as the person is NativeCluster's alone to publish.**
The kind exists since 0.4.6 (`"kind": "tool"`, `ncdev spec models`, "A
tool: `kind`"; Open Code is one): a program signed with NativeCluster's Developer
ID that runs as its person in their own terminal, never in the sandbox and
never run by the host, its model calls a tool session at the local address
that Members checks on every call. Only NativeCluster publishes one, so
this skill never makes one, and never presents a tool as something a port
can ship itself. When an app's data would all stay on the person's device,
put it to the user as a decision (Phase 4): hand it to NativeCluster to
publish as a tool, or make it a cluster app with what that keeps out
(`reference/cant-come-along.md`).

## Iron Law

1. **Never invent an answer to a stop-and-ask item** (`reference/stop-and-ask.md`).
   Ask the user, all of them at once, and wait. A guess that ships is worse
   than a question.
2. **Never run or download a model yourself.** No model runtime in the app,
   no weights in the package, no model hub, no hosted model, and no call
   straight to an engine on this Mac (nor reading its key). Models come from
   the host; one NativeCluster does not offer is asked for with
   `ncdev request-model`, and the feature waits for it. Testing with real
   models is not running one yourself: do it under `ncdev run` (Phase 8),
   which is the approved way, and is required.
3. **Reach nothing on this Mac but the host's socket.** The app may call
   any server it needs, on the internet or the office network, with no
   restriction from NativeCluster on which; naming its hosts in a `net.*`
   permission shows them on the sheet before install. It never reaches this
   Mac's own services (no `localhost`, no engine) or NativeCluster's ports,
   and its pages load no asset from outside.
4. **Never rebuild what NativeCluster does** (`reference/platform.md`): its
   people and sign-in, its roles (an app never makes its own
   administrators), running the app, its models, its logs. Remove the app's
   own copy instead of asking whether to keep it.

Never weaken a check to pass it, and never edit `ncdev`'s findings away: fix
what the finding says, or stop and ask.

## Phase 0 — Prepare

- Confirm `ncdev` runs (`ncdev help`), and read `ncdev spec hostapi` and
  `reference/contract.md` whole.
- Work in the app's own repository. Do not change anything outside it.
- **A large upstream (an agent, an IDE, anything you don't own) gets a port
  repository of its own**: the port's files there (manifest, launcher,
  adapters, packaging), and the upstream built from a pinned commit, never
  the port's files mixed into a fork. `ncdev check` reads every file of the
  folder it is given, so a port made inside its upstream's fork fails on the
  upstream's own source, with hundreds of findings that are not the port's.
- Prefer **Go** for a new or rewritten app: one binary, no runtime to ship,
  and `ncdev new` writes a working example to copy from. Another language
  works if the package carries everything the program needs to run
  (`reference/languages.md`).

## Phase 1 — Read the app

List every part of the app and what it does, in a table: its pages, its
server and routes, where it keeps data, every model it uses (and for what),
every outside service it calls, background or long work, how it knows who a
person is, and anything it downloads at run time. Read the code, not the
README: what runs is what matters.

Give the table a column **stored where: the cluster, or only this device**,
filled for every piece of data the app keeps (a person's files, their
history, settings, caches, an index). It decides what kind the app is (above)
and which parts keep the sandbox.

## Phase 2 — Map each part

For each row, write the NativeCluster way (`reference/mapping.md`), "NativeCluster
does it" (`reference/platform.md`), or "can't come along"
(`reference/cant-come-along.md`). Keep the table; it is the plan, and the
report at the end.

## Phase 3 — Models

- Each model use becomes a **task** in the manifest (a short id and a plain
  name: "translate", "summarize"). The app names tasks, never models.
- Run `ncdev models <dir>` and use what NativeCluster offers: a chat model
  for text, `embedding` for vectors, `transcription` for speech to text.
- Only if no offered model can do a task, file a request with the evidence:
  `ncdev request-model <dir> -task t -kind k -why "..." -tried "..." -meanwhile hidden`
  (`reference/models.md`). For a kind NativeCluster serves, "tried" is
  required: which offered models you tried and how they fell short.
- A feature that may lack its model on some Mac is an **optional task**
  (`"optional": true`): before offering it, the app asks
  `GET /host/v1/models?task=<it>`, and while `models` is empty the feature
  stays hidden, never broken and never an error message.
- Until a requested model comes, the feature is off (hidden, or shown as
  coming later), and the rest of the app works. Nothing ever downloads from
  outside NativeCluster's catalog.

## Phase 4 — Stop and ask

Collect every decision on `reference/stop-and-ask.md` that this app raises,
and ask the user **all at once**, each with the options and your
recommendation. Do not go on with a part whose decision is open; go on with
the rest. Confirm with the user **every row whose data you marked "only this
device"**: any data stored on the cluster keeps those parts sandboxed.

## Phase 5 — Write nc-app.json

From the map and the answers (`reference/manifest.md`; `ncdev spec schema`).
Storage is `cluster` only and `review` is null. `triggers` lists the
schedules and events the app keeps (`reference/manifest.md`), with
`compute.background` naming every one and `"min_host": "0.3.5"` or newer.

## Phase 6 — Adapt the app

Keep the contract (`reference/contract.md`), part by part:

1. Serve HTTP on the unix socket `LB_APP_SOCKET` names; never a TCP port.
2. Read the secrets from stdin; answer `<program> manifest` with nc-app.json.
3. Put **one** ticket check in front of **every** route; 403 without a valid
   ticket. Answer `GET /healthz` and `GET /.nc/whoami` behind it.
4. Take the person from the ticket and from nothing else; key every person's
   data by the ticket's person id.
5. Call models through the host: which model, a turn in line, the call with
   the grant, the grant back.
6. Keep every file in `NC_APP_DATA`.
7. Honour a plane's off switch in the environment, which an operator may
   set to `0` (`reference/contract.md` §5).
8. Offer an optional task's feature only while the host names a model for
   it (`ncdev new`'s "dictate" shows how).
9. Remove what can't come along; turn off features waiting on a model or a
   decision.

## Phase 7 — Check until clean

Run `ncdev check <dir>` and fix each finding as it says, until it reports 0
errors. Read every warning and either fix it or say in the report why it
stands. The running half is best-effort (it is not a sandbox): it does not
excuse anything the static half or this skill forbids.

`ncdev check` cannot see inside a compiled program: it skips the app's
program and any file over 4 MiB, so a Bun, Deno or Go binary, or a bundle
of an app's whole JavaScript, is never read. A clean check of a compiled
app proves nothing about what is compiled in: read its dependency list
(`package.json`, the bundler's metafile, `go.mod`) yourself, against
`reference/cant-come-along.md`, and say in the report what you read. Its
list of hosted-model clients also lacks the Vercel AI SDK's providers
(`@ai-sdk/openai`, `@ai-sdk/anthropic`, `@ai-sdk/openai-compatible` and the
rest): treat each as a hosted model's client.

## Phase 8 — Try it with real models (required)

`ncdev check` proves the contract, never the answers: its practice host
answers every model call with fixed words. Before packaging, run the app
with the real models:

    ncdev run <dir>

It starts a NativeCluster of its own on this Mac (the installed
NativeCluster's host and engine, alone, in a scratch folder, with this
Mac's models linked in: nothing of the user's NativeCluster is touched),
installs the app as its Mac owner would, and prints where to sign in and
open it. Then, signed in, use **every feature that calls a model, end to
end**, on the app's real material: each chat or search, and each job,
started and left to run its units to the end. Judge the answers with the user (`reference/models.md`,
"Prompts written for another model"), and fix what fails before packaging.
This is the approved way to test with models; never call an engine
directly instead. `-fake-models` (no model runs, every call answered at
once) is only for trying the pages fast or on a Mac with no models, and
never stands in for this phase; `-watch` installs again on every change.
Ctrl-C stops it and removes the folder (`-keep` keeps it). If this Mac has
no NativeCluster or no models, say so plainly in the report: the features
were not tried.

## Phase 9 — Package, install it on this Mac, and the two ends

`ncdev package <dir>` builds, checks and zips the package. NativeCluster
unpacks a package to at most 8 GB in at most 200,000 files and folders and
refuses a bigger one; Hermes, the largest app so far, is about 480 MB in
16,000. Then:

- **Install it on this Mac yourself:** `ncdev install <dir>` packages the
  app and puts it, whole, into NativeCluster's testing folder
  (`~/NativeCluster/Testing`). The NativeCluster app on this Mac installs it
  as a local app, for this Mac's owner only, and answers in
  `<id>.result.json` beside it, which `ncdev install` waits for and prints:
  `ok`, `app_id`, `sha256` (the package it took), `address` (where the app
  opens in NativeCluster's window on this Mac), or `error` (why not, in
  words). Never ask the user to upload the zip; install it this way. Only
  the group's owners, administrators and Developers can install apps of
  their own: if the result says so, tell the user and stop there, never
  work around it. The folder is off on each Mac until the user turns it on
  there (from 0.4.15), in NativeCluster's own window: Settings, Account,
  This Mac, Install apps dropped in the Testing folder. If the result says
  the Testing folder is off, tell the user where to turn it on and wait for
  them, then install again; never try to turn it on yourself. If the user's
  NativeCluster has no testing folder yet (before 0.3.1), say so and stop
  at the package. `-watch` installs again
  on every change, keeping the app's data. Installed, the app is local,
  shown as Testing under Development on the Apps page: nobody else has it.
- **Give it to the group:** publishing stays a person's. On the Apps page,
  the user presses Publish under Development, and the main's Mac owner OKs each
  new app and each new version. Never publish for the user.
- **Submit to NativeCluster (later):** only NativeCluster publishes apps to
  everyone. Send the package and `nc-model-requests.json` to whoever asked
  for the app; submitting is not built yet.

## Done — report back

- The Phase 1-2 table: each part, its NativeCluster way, or why it can't come along.
- The decisions the user made, and any still open.
- The model requests filed (task, kind, why).
- `ncdev check`'s final output, and the package's path and sha256.
- `ncdev install`'s result: installed on this Mac (its address), or why not.
- What you tried under `ncdev run` with real models, feature by feature,
  and how each did (or that it could not be tried, and why).
- Anything you are unsure of, said plainly.
