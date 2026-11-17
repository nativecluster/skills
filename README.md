# NativeCluster skills

Skills that teach a coding agent to set up and use [NativeCluster](https://www.nativecluster.com), and to
build apps for it. NativeCluster is a free Mac app that runs open AI models on a team's own Macs and gives
every person, app and agent one OpenAI-compatible endpoint, with sign-in and roles. Macs with Apple silicon,
macOS 13 or later.

## Install NativeCluster

```sh
curl -fsSL https://nativecluster.com/install.sh | sh
```

or with Homebrew:

```sh
brew install --cask nativecluster/tap/nativecluster
```

Then open NativeCluster and sign in. On a Mac running it, coding tools and code reach the cluster's models
at `http://localhost:8487/v1`, the OpenAI shapes:

```sh
nativecluster env                       # writes the base URL and a key of your own; source the file it prints
nativecluster run gemma4 "say hi"       # one answer from your cluster, in the terminal
```

## The skills

| Skill | What it does |
|---|---|
| [`nativecluster`](skills/nativecluster/SKILL.md) | Installs and sets up NativeCluster, points a project's model calls at the cluster, and explains its errors. |
| [`nativecluster-app`](skills/nativecluster-app/SKILL.md) | Turns an app, in any language, into a NativeCluster app that runs on a group's own Macs. |

NativeCluster puts both in `~/.claude/skills` when it installs, so a Mac running it already has them.
These are the same files, byte for byte, as the release named in `.claude-plugin/plugin.json`.

## What these skills do on your Mac

They are instructions for your agent, with no code of their own: no hooks, no MCP servers, no scripts.
What they tell the agent to run, fetch or send:

- **Install NativeCluster, only with your OK.** The install line above downloads NativeCluster from
  updates.nativecluster.com, and checks its SHA-256, our team's signature and Apple's notarization
  before it copies anything. It never uses sudo. Homebrew, with your OK, installs the same release.
- **The `nativecluster` and `ncdev` commands on your Mac.** They talk to NativeCluster on this Mac, which
  reaches our servers only for what its [Privacy Policy](https://www.nativecluster.com/privacy) lists,
  such as signing in, model downloads and updates.
- **Model calls to `http://localhost:8487/v1`,** NativeCluster's local address. They run on your own
  Macs, never on a hosted AI service, and never reach us.
- **NativeCluster's docs,** read from www.nativecluster.com.
- **`nativecluster-app` works on the app you ask it to port,** in your folder, and runs `ncdev` to check,
  package and install it on this Mac.

They leave to you: installing NativeCluster, signing in (in your browser), the few steps the Mac confirms
with Touch ID or its password, and choosing each model to download, having seen its size and the folder it
goes to. They never ask for a password, a PIN or a key, and never print one.

### As a Claude Code plugin

```
/plugin marketplace add nativecluster/skills
/plugin install nativecluster@nativecluster
```

### Any other agent

Copy a folder from `skills/` into the folder your agent reads skills from.

## In your project's AGENTS.md

So every agent that works on a project uses your cluster rather than a hosted API:

```md
## AI models

This project's model calls go to our own Macs through NativeCluster, never to a hosted AI service.

- Code reads `OPENAI_BASE_URL` and `OPENAI_API_KEY` from the environment. On a Mac running NativeCluster, run `nativecluster env` and source the file it prints: the base URL is `http://localhost:8487/v1`.
- Never hard-code a key, and never print, paste, log or commit anything starting `nck_` or `nct_`.
- `nativecluster models list` shows the cluster's models. Never download a model the person didn't choose, having seen its size and the folder it goes to.
- NativeCluster's docs: https://www.nativecluster.com/llms.txt
```

## More

- Docs: https://www.nativecluster.com/docs (for agents: https://www.nativecluster.com/llms.txt)
- Python kit for apps: `pip install nativecluster`
- Questions: https://discord.gg/wpuRp3y8j

The skills are licensed under Apache-2.0 (`LICENSE`). NativeCluster itself is a free app, not open source.
