# Mapping an app's parts

| The app has | The NativeCluster way |
|---|---|
| A web server on a port (`0.0.0.0:8000`, `app.run(port=...)`, `listen(3000)`) | serve on the unix socket `LB_APP_SOCKET` (`reference/languages.md`) |
| Parts that talk to each other over `127.0.0.1` (a gateway its own workers or command line dial, a control port, a code kernel) | unix sockets in `NC_APP_SOCKETS`, the app's short folder for them; requests on `LB_APP_SOCKET` must carry the host's ticket, so its own parts get a socket of their own there |
| A start script, a virtualenv, `.env` settings | the host starts the program; settings it needs are constants or files in the package; secrets come on stdin |
| Its own accounts, profiles, logins, "who are you" screens | gone: the ticket's `person` is the person; key every person's data by it; a person's name needs `people.read` (stop and ask) |
| Its own admins, roles or "first user is admin" | gone: roles are NativeCluster's (`reference/platform.md`); a setting only an administrator may change is shown and accepted only when the host's role route answers `owner` or `admin` |
| A health check, keep-alive or watchdog on a timer | gone: NativeCluster runs the app; the repairs it made run at startup or where the data is read |
| A log file, log page or log viewer | diagnostics to stdout or stderr (NativeCluster keeps them), never a person's words; the page is removed |
| Model, provider or API-key settings | gone: the host picks the model for each task |
| Files and databases in its folder or home | `NC_APP_DATA` only (a SQLite file there is fine) |
| Data shared between people | per person by default, on each person's own computer; a shared home (the main's copy of the app, reached through the hand-over, or a server of the app's own) is a decision (stop and ask) |
| Model calls | `reference/models.md`: by task, through the host |
| Long work (whole-book jobs, batch processing) | a job: started from a request, calls in lane `B` with a job ticket, progress kept in `NC_APP_DATA`, resumed after a restart |
| Work on a timer, cron or schedule, an event, a heartbeat | a schedule in the host's table, fired on the computer the app runs on, its firing answered at `POST /nc/v1/wake` (`reference/contract.md`, "Schedules"); work that must run while that computer sleeps is handed to the main (stop and ask); the app's own timer, cron thread or scheduler library goes |
| Downloads at run time (books, datasets, assets) | keep them: declare each host in a `net.*` permission; ask only about what the download may be used for (`reference/stop-and-ask.md`, 2) |
| CDN scripts, web fonts, images from outside | vendor them into the package |
| FFmpeg or another tool it shells out to | only if the feature using it stays; it ships inside the package with its license (stop and ask before adding a GPL build) |
| Telemetry, analytics, crash reporting | remove |
| Admin pages, debug routes | removed; an administrator's page waits for NativeCluster's roles, and stays behind the ticket check |
