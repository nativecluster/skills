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
   name, or is its own id for them enough? Recommend no unless the app
   shows who did something.
4. **Sharing between people**: is anything one person makes visible to
   another? Today every person's data is their own. If the answer is a
   setting, it is an administrator's, and administrators are NativeCluster's
   (`reference/platform.md`): never ask how the app should pick them.
5. **Where each piece of data is stored** (Phase 1's "stored where"
   column): confirm every row you marked **only this device**. Data stored
   on the cluster (the main, an inference computer) belongs to a cluster
   app and keeps the sandbox; data only on the person's own device, with
   only model calls leaving it, could belong to a tool that runs as the
   person (SKILL.md, "A cluster app, or a tool that runs as the person").
   Only NativeCluster publishes that kind (`"kind": "tool"`, since
   0.4.6): when every row is "only this device", ask whether to hand it to
   NativeCluster to publish as a tool, or make a cluster app now, with what
   the sandbox keeps out (`reference/cant-come-along.md`: no folders of the
   person's, no developer tools). Recommend the tool for an app whose point
   is the person's own files or tools (a coding agent, an IDE). The
   `this-mac` storage setting is not supported either.
6. **Removing a feature** for any reason above.
7. **A tool with a strong license** (GPL FFmpeg and the like) shipped in the
   package.
8. **Anything you are unsure is allowed.** Ask; don't decide.

Work on a timer, a schedule or an event is not a question: the app keeps
its schedules in the host's table, for whatever reason it has, and the
host's clock runs them (`reference/contract.md`, "Schedules").

Never ask whether the app may reach the internet: that is fine. Declare
each host in a `net.*` permission, saying in `what` exactly what leaves the
Mac (the words of a search, an id), and go on; only item 2 asks anything of
a download.

Never ask whether to keep what NativeCluster already does (sign-in, roles,
running the app, health checks, logs: `reference/platform.md`). Remove it
and say so in the plan.
