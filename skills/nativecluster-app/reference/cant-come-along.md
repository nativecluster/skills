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
| Its own sign-in or profiles | the host signs people in | the ticket's `person` |
| Its own administrators | roles are NativeCluster's | ask the host each person's role (`reference/platform.md`) |
| Data on a person's own Mac (`this-mac`) | an app's part on a person's Mac is not built yet | `cluster` storage, or stop and ask |
| A timer, cron or scheduler of its own | the host's one clock runs every schedule, on the main, waking the app | a schedule in the host's table (`reference/contract.md`, "Schedules") |
| A program outside the package (a person's `git`, `npm`, `node`, a compiler) | the sandbox runs only the package's own programs and `/bin`, `/usr/bin`, `/sbin`, `/usr/sbin`; and `/usr/bin/git`, `python3`, `make` and `clang` there are Xcode's shims, which fail inside it | carry what the app runs in the package; a coding agent that needs the person's tools is a tool that runs as the person, which only NativeCluster publishes (stop and ask, item 5) |
| A person's own folders (`~/code`, a project, Documents) | an app reads and writes only its package and `NC_APP_DATA`; no permission picks a folder of the person's | keep the work in `NC_APP_DATA`, or stop and ask (item 5): a coding agent needs the tool kind |

A feature that depends on something here is **off**, not broken: an
optional task hidden until its model comes, or removed with the user's
agreement. Never ship a button that fails.
