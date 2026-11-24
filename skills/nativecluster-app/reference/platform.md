# What NativeCluster already does (never rebuild it)

An app on NativeCluster is a part of something larger: NativeCluster runs
the group's Macs, its people, its models and its network, and every app
shares them. Much of what a stand-alone app builds for itself is already
done, once, for every app. Rebuilding any of it makes a second, weaker copy
that disagrees with the real one. Remove it from the app, and do not ask the
user whether to keep it: it is not the app's to decide.

| NativeCluster does | So the app |
|---|---|
| **People and sign-in.** Accounts, passwords, sign-in, sessions, invitations, who is in the group, and which people get which app (Members). Every request's ticket names who is asking, by an id the host made for this app alone; a ticket with no person is someone not signed in. | Has no accounts, logins, profile pickers, user lists, invitations, "who are you" screens or user or device headers (`X-User-ID`, `X-Device-ID`). It keys each person's data by the ticket's person. It needs a person's name or email only to show who did something, and then asks for `people.read`. |
| **Roles.** Who is the group's owner, an administrator or a member is NativeCluster's, set in NativeCluster. The host tells an app each of its people's role: `GET /host/v1/people/<the ticket's person>/role` answers `owner`, `admin` or `member`, fresh every time (`reference/contract.md` §5). | **Never makes its own administrators**: not the first person to open it, not a code, a password or a list in a file. **A setting only an administrator may change stays hidden** from anyone the host does not answer `owner` or `admin` for, and the app asks again when the change arrives. |
| **Schedules.** The clock of the computer the app runs on: the app's schedules, events and loops in the host's table, each firing handed to the app (`POST /nc/v1/wake`), the app started first if it rests, a firing missed while the computer was off caught up once, and every model call of the work in the one line behind people, fairly between apps. Work that must happen while a person's computer sleeps goes to the same app on the group's main, the computer assumed always on, through the hand-over (`POST /host/v1/main/hand`). | Has no timer, cron, scheduler thread or heartbeat of its own. It sets its schedules in the host's table, for its own people and its own reasons, keeps what each one does in `NC_APP_DATA`, and answers each firing (`reference/contract.md`, "Schedules"). |
| **Running the app.** On each computer the app runs on, it starts the program when someone opens the app, waits for `/healthz`, stops it when it has been idle (SIGTERM), starts it again when it is needed (`woken` on stdin), and runs the new version after an update. | Has no supervisor, keep-alive, watchdog, health-check timer or "server status" page. Repairs a timer used to do (rebuilding an index, resetting work left "running" by a crash) run at startup, or where the data is read. Long work is a job that resumes. |
| **The address and the network.** Each app has its own address, behind NativeCluster's TLS and its door: on a person's own computer it answers that computer alone, and on the main the office network too. Its pages are served at `NC_APP_BASE`. | Has no host, port, TLS or CORS settings. Its links start with `NC_APP_BASE`. |
| **Models.** The catalog, downloads, the engine, the waiting line (the main's, which runs each call on one of the group's inference computers), and each person's and each app's fair share of a busy Mac. | Has no model settings: no provider choice, model picker, API key or "local or cloud" switch. It names tasks; the host picks the model. |
| **Permissions.** The sheet before install lists what the app asks for, its internet hosts among them; they are on for everyone given the app, with no switch to turn on or off. | Has no "allow the internet" toggle of its own. Before each send it asks the host whether the plane is on (`reference/contract.md` §5). |
| **Logs and activity.** The app's stdout and stderr go to NativeCluster's own log on the computer it runs on, for whoever runs that Mac. Each model call is recorded as metadata only (which app, when, how long), never a word of it. | Prints its diagnostics to stdout or stderr, never what a person wrote, read or asked. Has **no log page** and no log viewer: in a group it would show everyone's activity to everyone. A jobs page showing progress and errors is the app's own and may stay. |
| **Installing and updating.** NativeCluster installs the package on each computer whose person downloads it, and runs the version the group's Mac owner allowed. | Never updates itself, downloads code or checks for updates. |
| **Where data lives.** One folder for the app, `NC_APP_DATA`, on the computer that runs it: each person's copy keeps that person's data on their own computer, and the main's copy keeps its data on the main. A shared folder every copy opens like a disk is planned, not built. | Keeps everything there. Data that people share needs one home: the main's copy, reached through the hand-over, or a server of the app's own. What one person's data shows to another is the app's choice (stop and ask). |

**The app's window.** Each app opens in a window of its own on the person's
computer. In that window, a page's file input (`<input type="file">`)
opens the Mac's file picker. The file the person picks is uploaded to the
app like any upload, and the app keeps it in `NC_APP_DATA`; it never reads
the person's folders. The upload needs no permission, and nothing in
NativeCluster reads the `files.pick` item yet. The microphone works in that
window once the manifest asks for `mic` and the person turns that item on
for themselves, on the app's About page in NativeCluster; macOS then asks
them. Without the item, or with it off, the window refuses the microphone.

Linking a person's account to a service: see
`reference/linking-accounts.md`.

What NativeCluster does not do yet for your own apps, and so the app can't
either: a storage place other than `cluster` (`this-mac`, `either`), or
reaching another app on the Mac (`connect.local`).
`reference/cant-come-along.md` says what to do instead.
