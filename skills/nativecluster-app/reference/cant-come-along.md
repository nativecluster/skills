# What can't come along (and what to do instead)

`ncdev check` finds these; each finding says what to fix.

| Can't come along | Why | Instead |
|---|---|---|
| A model runtime in the app (torch, transformers, onnxruntime, llama.cpp, MLX, a local model server, Kokoro, diffusers, Whisper) | models run in NativeCluster's engine, in one line, for everyone | a task through the host; `request-model` if nothing offered fits |
| Model weights in the package (`.gguf`, `.safetensors`, `.onnx`, `.pt`, `.bin` weights) | same | same |
| A hosted model's client (OpenAI, Anthropic, Gemini, Cohere, Mistral, Groq): a warning, not a refusal | an app may call any service, but this inference leaves the group's machines | a task through the host keeps it on them |
| A model hub (Hugging Face, or a model server's own registry) | an app never downloads a model | `request-model` |
| A TCP port, `localhost`, another service on the Mac | the app answers only the host, and reaches only the host | `LB_APP_SOCKET`, `LB_HOST_SOCKET`; between the app's own processes, unix sockets in `NC_APP_SOCKETS` |
| A call to this Mac's own services (`localhost`, an engine, NativeCluster's ports) | the sandbox lets an app reach this Mac only through the host's socket | reach models and the group through `LB_HOST_SOCKET`; any other computer is the app's to call |
| Outside page assets | a page loads nothing from outside | vendor them |
| Its own sign-in as WHO the person is (its login page, its user table as identity) | the host signs people in | the ticket's `person` |
| An account at the app's own service, or a linked account (OAuth), that its connectors go through | allowed: an app whose connectors go through a service may ask its person to sign up there or link the account (an agent asking for its maker's account is the common case) | keep it, keyed by the ticket's `person`; link without a loopback listener (`reference/linking-accounts.md`), tokens in the host's secrets store |
| Its own administrators | roles are NativeCluster's | ask the host each person's role (`reference/platform.md`) |
| A storage place of `this-mac` or `either` | refused at install ("storing on This Mac is not supported yet") | `cluster` storage, which is already the app's `NC_APP_DATA` on the computer it runs on |
| An older install's data (a folder in the person's home or Library, a database beside the program) | the sandbox reads and writes nothing outside the package and `NC_APP_DATA` | never read, copy or move it from the app; stop and ask (item 9) |
| A timer, cron or scheduler of its own | the host's clock, on the computer the app runs on, runs every schedule and wakes the app; work for while that computer sleeps is handed to the main | a schedule in the host's table (`reference/contract.md`, "Schedules") |
| A program outside the package (a person's `git`, `npm`, `node`, a compiler) | the sandbox runs only the package's own programs and `/bin`, `/usr/bin`, `/sbin`, `/usr/sbin`; and `/usr/bin/git`, `python3`, `make` and `clang` there are Xcode's shims, which fail inside it | carry what the app runs in the package; a coding agent that needs the person's tools is a tool that runs as the person, which only NativeCluster publishes (stop and ask, item 5) |
| A person's own folders (`~/code`, a project, Documents) | an app reads and writes only its package and `NC_APP_DATA`; no permission picks a folder of the person's | keep the work in `NC_APP_DATA` (a person may upload files they pick in the app's window: `reference/platform.md`), or stop and ask (item 5): a coding agent needs the tool kind |

A feature that depends on something here is **off**, not broken: an
optional task hidden until its model comes, or removed with the user's
agreement. Never ship a button that fails.
