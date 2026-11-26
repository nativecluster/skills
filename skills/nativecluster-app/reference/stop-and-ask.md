# Stop and ask

These are the user's decisions, never yours. Collect every one the app
raises, ask them **all at once**, each with the options and your
recommendation, and wait. Go on with the parts no open decision touches.

1. **A feature whose model NativeCluster does not offer** (text to speech,
   images, alignment, anything `ncdev models` lacks): hide it until a
   model comes (an optional task plus a model request), show it as coming
   later, or drop it?
2. **Where content comes from, and whether it may be shipped**: a
   download's license or terms, whose name may appear, content that may
   not be shared where the app is sold.
3. **People's names** (`people.read`): does the app need a person's email or
   name, or the list of the group's people (to pick someone to share
   with), or is its own id for them enough? Recommend no unless the app
   shows who did something or lets a person choose someone.
4. **Sharing between people**: is anything one person makes visible to
   another? Each person's copy of the app runs on their own computer and
   keeps their data there, so by default every person's data is their own.
   Data people share needs one home that every copy reaches. Ask which
   home, and give the options built today:
   - **the main's copy of the app**, which keeps the data in its
     `NC_APP_DATA` on the main and decides who may see what. Each person's
     copy reaches it through the hand-over (`POST /host/v1/main/hand`,
     `min_host` 0.4.13). Each call carries one JSON value of at most
     256 KiB, so a large file goes in pieces. The main must run the app,
     must answer within a minute, and must be in reach: while it is away,
     the copy keeps the change and sends it again later. For an app not
     made for each person, the main's copy gets the hand-over as the person
     the computer is registered to (`reference/contract.md`, "Schedules");
   - **a server of the app's own**, which the app may call;
   - **no sharing**: each person's data stays on their own computer.

   A shared folder that every copy opens like a disk (synced storage) is
   planned, not built; never build on it. If the answer is a setting, it
   is an administrator's, and administrators are NativeCluster's
   (`reference/platform.md`): never ask how the app should pick them.
5. **Where each piece of data is stored** (Phase 1's "stored where"
   column): confirm every row you marked **only this device**. It stays on
   the person's own computer, in their copy's `NC_APP_DATA`, and no other
   person's copy sees it. Every row marked the cluster needs its home from
   item 4. Data stored on the cluster belongs to a cluster app and keeps
   the sandbox. Data only on the person's own device, with only model calls
   leaving it, fits a cluster app too. It could instead belong to a tool
   that runs as the person, if the app needs the person's own folders or
   developer tools (SKILL.md, "A cluster app, or a tool that runs as the
   person"). Only NativeCluster publishes that kind (`"kind": "tool"`,
   since 0.4.6). When every row is "only this device" and the app needs
   those, ask whether to hand it to NativeCluster to publish as a tool, or
   to make a cluster app now, with what the sandbox keeps out
   (`reference/cant-come-along.md`: no folders of the person's, no
   developer tools). Recommend the tool for an app whose point
   is the person's own files or tools (a coding agent, an IDE). The
   `this-mac` storage setting is not supported either.
6. **Removing a feature** for any reason above.
7. **A tool with a strong license** (GPL FFmpeg and the like) shipped in the
   package.
8. **Work that must happen while the person's computer sleeps** (a
   morning summary, a reminder sent at 7): a schedule fires on the computer
   the app runs on, and one missed while that computer was off runs once
   when it starts again. Ask whether to leave it so, or to hand the job to
   the same app on the main, which keeps it and runs it on time with the
   computer shut. A handed job travels through the hand-over
   (`reference/contract.md`, "Schedules"), so the main must run the app,
   and the job carries what it needs (its prompt, the files given it)
   inside the hand-over's limit.
9. **Data from an older install** (the app's own folder in the person's
   home or Library, a database beside the program, a file another version
   wrote): the app never reads, copies or moves it, and never asks for a
   way around the sandbox to do so. Ask whether to start fresh, or to have
   the person bring it in through the app's window: an export from the
   older install that they pick and upload, kept in `NC_APP_DATA`
   (`reference/platform.md`, "The app's window"). Recommend the upload if
   the old data matters to people. Never write a step that reaches outside
   `NC_APP_DATA`.
10. **Linking a person's account to a service**: see
    `reference/linking-accounts.md`.
11. **Anything you are unsure is allowed.** Ask; don't decide.

Work on a timer, a schedule or an event is not a question: the app keeps
its schedules in the host's table, for whatever reason it has, and the
host's clock runs them on the computer the app runs on
(`reference/contract.md`, "Schedules"). Only where the work must run while
that computer sleeps is a question (item 8).

Never ask whether the app may reach the internet: that is fine. Declare
each host in a `net.*` permission, saying in `what` exactly what leaves the
Mac (the words of a search, an id), and go on; only item 2 asks anything of
a download.

Never ask whether to keep what NativeCluster already does (sign-in, roles,
running the app, health checks, logs: `reference/platform.md`). Remove it
and say so in the plan.
