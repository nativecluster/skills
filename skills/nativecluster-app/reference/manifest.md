# nc-app.json

`ncdev spec schema` has every field; `ncdev spec permissions` every
permission. `ncdev new` writes a working one. The fields:

| Field | What |
|---|---|
| `format` | `"nc-app.v1"` |
| `id` | lowercase letters, digits and `-`, 2 to 32: `"classics"`. It is the app: keep it the same in every version, since an update with another id is refused |
| `name` | what people see, up to 40: `"Modern Classics"`. From NativeCluster 0.4.24 a new version may change it (its Store entry names the new name): Apps, the app's window and its Dock icon show the new name, and its data stays in the folder it had (`NC_APP_DATA`). Older NativeCluster refuses a renaming version until it is on 0.4.24. Read your data folder from `NC_APP_DATA`, never from the name |
| `min_host` | optional: the oldest NativeCluster it runs on |
| `version` | optional: the app's own version, `"0.1.0"`: numbers and dots, optionally a short label after a dash (`"1.3.0-beta.2"`), at most 32. Its card on Apps shows it ("Modern Classics 1.3 · Live version is 1.2") and its Publish names it ("Publish 1.3"). It needs `"min_host": "0.3.1"` or newer, since older NativeCluster refuses the field; `ncdev check` says so. `ncdev new` writes `"0.1.0"`. Raise it for each version you publish |
| `program` | the file in the package that NativeCluster starts |
| `per_person` | optional: `true` for an app made for one person at a time, a personal agent whose one folder holds one person's conversations, memory and accounts (Hermes). NativeCluster runs one process of it for each person who uses it. Each process gets its own data folder, the app's whole data folder as it sees it, which no other person's process can read, and is told its person as it starts (`NC_APP_PERSON`, the id its tickets name them by). Every request and firing it gets is that person's; its waits in line and its schedules name them alone, and a schedule's id must be one no other person's row holds (put the person in it). It needs `"min_host": "0.3.7"` or newer, since older NativeCluster refuses the field. Without it, one process serves all the app's people and tells them apart by the ticket |
| `icon` | optional: the app's icon, a square PNG in the package by its path from this folder (`"assets/icon.png"`). Make it 512 px; 128 to 1024 px and at most 1 MB are accepted, never SVG. It needs `"min_host": "0.3.0.4"` or newer, since older NativeCluster refuses the field; `ncdev check` says so. `ncdev package` carries the file. Without one, NativeCluster draws the name's initials |
| `tasks` | each kind of model work: `{"id","name"}`, and optionally `ability`, `optional`, `min_window`, `min_skills`, `tests`, `models` |
| `permissions` | what it asks for (below) |
| `triggers` | each kind of schedule or event the app keeps: `{"id","kind","name"}`, `kind` `time` (at set times, or every so often), `event` (when the app says something happened) or `loop` (an agent's work, round after round until the app says done); `name` is what people see ("your morning summary"). `[]` for none. Declaring one needs `"min_host": "0.3.5"` or newer and `compute.background` naming it; `ncdev check` says so |
| `storage` | each kind of data: `{"kind","label","place":"cluster"}` |
| `review` | `null` |

## Tasks

- One task per kind of model work, named for what it does for a person:
  `{"id": "translate", "name": "translating a book"}`. The app names the
  task in every model call; NativeCluster picks the model.
- `"ability"`: the usual way to say what a task needs, as a kind of model
  rather than a particular one: `"embedding"` (search), `"transcription"`
  (hearing speech), `"diarization"` (telling a recording's speakers apart:
  optional, with `min_host` the release that adds it), `"vision"` (reading
  pictures), `"completion"`, `"tools"`, `"thinking"`. Any model that can,
  of any family, may do it.
  `"speech"` (reading text aloud) and `"image-generation"` (making
  pictures) are reserved: NativeCluster offers no such model yet, so a task
  stating one matches nothing; make it optional, keep the feature hidden,
  and `ncdev request-model -kind tts` or `-kind image` records the need.
  Leave `ability` out only for a task any chat model may do. **A task
  that is not chat must name its ability**: without one, any model counts
  as doing it, so a reading-aloud task reads as covered by a chat model, the
  app's page says "Any model here", and `/host/v1/models` names a chat
  model for it (acceptance run 1: narrate, cover and find-passages had
  none).
- `"optional": true`: the app works without it, and shows that feature
  only while `GET /host/v1/models?task=<id>` names a model. Use it for any
  feature that waits for a model NativeCluster may not have on the Mac.
- `"min_window": 64000`: the least context window, in tokens, a model
  must give the task on the Mac that runs it (prompt and answer together),
  for an app that fails below it, such as an agent whose instructions and
  tools fill much of the window. NativeCluster sizes each chat model's
  window from that Mac's hardware, so Get installs the app only where a
  model gives it on a Mac of the cluster (today, 64,000 with `"tools"`
  needs a 32 GB Mac), and `GET /host/v1/models` names only such models,
  with `why` in words to show while none does. Chat tasks only. It needs
  `"min_host": "0.4.22"` or newer, since older NativeCluster refuses the
  field. Leave it out when any window will do.
- `min_skills`, `tests`, `models` narrow further which models may do it
  (`ncdev spec models`). Leave them out unless the app truly needs them:
  every one you add can only shrink what reaches you. `tests` need results
  NativeCluster has published, so leave them out for now.

## Permissions

| id | Arguments | Means |
|---|---|---|
| `models.use` | `tasks` | the tasks it calls models for: every task |
| `storage.cluster` | `kinds` | the kinds it keeps: every storage kind |
| `compute.background` | `triggers` | working when no one is using it: every trigger the manifest declares, no more and no fewer |
| `people.read` | | the person's email and name in the ticket, and the group's people listed by the host (`GET /host/v1/group/people`, `min_host` 0.4.24): each one's id, name, email and whether they have the app (stop and ask) |
| `purchase.ask` | | asking for a purchase; it can never charge (stop and ask) |
| `files.pick` | | files a person picks on their Mac. The app's window opens the Mac's file picker for a page's `<input type="file">` and uploads the file to the app without this item, and nothing in NativeCluster reads the item yet (`reference/platform.md`, "The app's window") |
| `connect.local` | `apps` | another app on this Mac (not yet for your own apps) |
| `mic` | | the microphone, in the app's own window: each person turns it on for themselves on the app's About page, and macOS then asks them; while it is off, the window refuses the microphone (`reference/platform.md`, "The app's window") |
| `terminal` | | terminals of its own, a shell or any program run in one (an agent's terminal panel): the group's owner approves it for each app, an update that adds it waits for that OK, and the sandbox lets it open only the terminals it made, never another program's on the Mac; `min_host` 0.4.0 (stop and ask) |
| `net.<plane>` | `hosts`, `what`, `off_switch` | reaching the internet at those hosts (exact names; `*.<name>` for any name under one, for a service whose servers change their names, with `min_host` 0.3.6): `what` says what leaves, in plain words; `off_switch` is an `LB_...` name of the app's own, like `LB_BOOKS`, which the host sets to `0` in the app's environment when the plane is off for everyone (how the app goes by it: `reference/contract.md` §5) |

`net.casesearch`, `net.google` and `net.microsoft` are reserved names, each
person's own call rather than the group's; your app's is a new
name, like `net.books`, and is the group's: on for everyone given the app.

## Storage

Every kind is `"place": "cluster"`. Whatever the word suggests, it means
the app's `NC_APP_DATA` on the computer that runs it: on each person's own
computer for their copy, and on the main for the main's copy. It is not a
folder that every computer shares. Data that people share needs one home
(the main's copy, reached through the hand-over, or a server of the app's
own), which is a stop-and-ask item. `this-mac` and `either` are refused at
install ("storing on This Mac is not supported yet"). Declare every kind
`cluster`, which already keeps it on the computer the app runs on. If the
app keeps something outside `NC_APP_DATA` today (an older install's
folder), stop and ask.
