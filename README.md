# pi-workbench

An openhost app that gives you tabbed in-browser terminals, preinstalled
[pi](https://github.com/earendil-works/pi-mono) (a terminal coding agent), and
a cloned copy of the openhost repo. A fork of `claude-workbench`
(`claude-code-container`) with the same UI, but running **pi + OpenRouter** by
default instead of Claude Code.

## What's inside the container

- `@earendil-works/pi-coding-agent` (npm, pinned `0.84.1`, requires Node >= 22;
  the image installs Node 22 from NodeSource). Run `pi` in any terminal to
  start it.
- Python 3 + git + the usual tools (gh, glab, uv, the `oh` openhost CLI).
- A clone of `https://github.com/imbue-openhost/openhost` placed at
  `~/openhost` on first container start (override with `OPENHOST_REPO_URL` or
  `OPENHOST_DIR` env vars).
- A pi skill at `~/.pi/agent/skills/openhost/` that points pi at the curated
  docs in the local openhost clone.

## Model selection

On a fresh persistent HOME the workbench seeds `~/.pi/agent/settings.json` so
pi picks a sensible default model (pi 0.84+ fetches its model catalog at
runtime, so no extra model definitions are needed):

- `OPENROUTER_API_KEY` present → `openrouter` / `openrouter/auto`
- otherwise `ZAI_API_KEY` present → `zai` / `glm-5.2`
- neither → left unconfigured, so pi prompts for `/login` on first run

Both defaults are in pi 0.84.1's catalog (`openrouter/auto`, and `glm-5.2` on
the `zai` provider). Override them with `PI_DEFAULT_PROVIDER` and
`PI_DEFAULT_MODEL` env vars (or edit `~/.pi/agent/settings.json` after first
start). The seed only writes when the file doesn't exist, so manual config is
never clobbered.

## Authentication

pi's auth is whatever the user sets up inside the terminal — an API key in the
environment, or an interactive `/login`. As a convenience, if the `secrets-v2`
app is installed the workbench fetches `OPENROUTER_API_KEY` (and `ZAI_API_KEY`)
on first PTY launch and exports them into every new terminal's environment.
This is best-effort — if the secrets app isn't around the terminal still works,
you just set the key yourself.

`HOME` lives on the app's persistent data dir (`/data/app_data/pi-workbench/home`),
so `/login`, the openhost clone, and shell history all survive container
redeploys. The workbench's own prompt, aliases, and PATH fixups live at
`/etc/profile.d/workbench.sh` (sourced by both login and non-login interactive
bash), so `~/.bashrc` and `~/.bash_profile` are entirely yours — anything you
write there sticks around and is never overwritten by image updates.

## The UI

`GET /` serves a tabbed xterm.js page. Each tab opens its own WebSocket to
`/terminal/ws`, which bridges to a PTY running `bash -l` inside the container.
New tabs start `pi` automatically.

## Prefilling a pi session (preview)

There's a stub for the eventual "open a pi session with this context" flow:

```
POST /api/sessions
  { "prompt": "fix this 503", "context": "<app logs / request info>" }
  -> { "id": "<token>", "url": "/?session=<token>" }
```

Opening the returned URL launches a new tab that runs `pi` with the combined
context + prompt as its initial message. The session id is consumed on first
use.

## The `open-workspace` service

pi-workbench is a **provider** of the `open-workspace` openhost service: *"here
is a repo at a commit — send me to a place where a person can work on it."* The
contract is defined in this repo under [`services/open-workspace/`](services/open-workspace/)
and is implementation-neutral, so a future provider (a cloud IDE, Cursor,
PyCharm…) can satisfy it without any caller changing.

```
POST /open-workspace          (form or JSON body, or query params)
GET  /open-workspace          (query params)
  repo=<clone-url>&ref=<commit|tag|branch>
  -> 303 redirect to /?session=<token>
```

GET is accepted in addition to the canonical POST as a workaround for the
openhost router's login bounce: an unauthenticated POST gets `302`'d to
`/login?next=…`, and a browser following that demotes the eventual return hop
to GET (only HTTP `307`/`308` preserve method). Accepting GET means the
post-login landing still resolves instead of `405`-ing.

- `repo` (required) — an `https://`, `http://`, `ssh://`, or `git@…` clone URL.
  Other transports (e.g. `ext::`, `file://`) are rejected.
- `ref` (required) — a commit, tag, or branch identifying the exact code.

The endpoint clones `repo` at `ref` and 303-redirects you into a terminal
sitting in that checkout. Status codes follow the contract: `400` for a
missing/malformed `repo` or `ref`, `404` when the repo or a named ref doesn't
exist, `403` when the repo is private and the workbench has no authorization to
reach it, and `5xx` for internal errors.

The clone lands at `$HOME/<repo-name>`. Opening the same repo again **reuses**
that directory rather than clobbering it: it fetches, and if the working tree
has uncommitted changes it asks — right in the terminal — whether to commit
them to a `workbench-wip-…` branch, stash them, drop them, or keep them as-is
and stop. Only once the tree is clean does it check out the requested ref.

### Private repos

To open a private repo the workbench mints a short-lived, `repo`-scoped GitHub
token via the openhost `oauth` service — the same flow openhost itself uses to
clone private repos — injects it into the clone/fetch URL transiently, and
strips it from the remote afterward so the token is never persisted on disk.

## Running locally without openhost

```
pip install quart hypercorn
python3 server.py
```

Then open http://localhost:5000.
