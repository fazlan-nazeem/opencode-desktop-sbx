# opencode-desktop

> **Experimental.** This kit depends on `--display`, which is an experimental
> sbx feature gated behind a feature flag. See [Prerequisites](#prerequisites).

A mixin that installs [OpenCode Desktop](https://opencode.ai) (the Electron
build of OpenCode's GUI client) from its GitHub release `.deb` and launches it
as a native window on the host desktop. Unlike an editor mixin, OpenCode
Desktop is the full OpenCode agent — it can run sessions on its own, wired to
whatever model provider you configure inside the app.

## Prerequisites

**sbx v0.39.0 or later** is required. Earlier versions set `WAYLAND_DISPLAY`
container-wide in a way that caused the clipboard bridge to clobber the
compositor socket, so Electron apps never mapped a window.

OpenCode Desktop renders to the host compositor via `--display`, which
requires enabling a feature flag:

```console
$ sbx settings set platform.allowExperimentalFeatures true
$ sbx settings set feature.sandbox-display true
```

## Usage

The natural pairing is the built-in `opencode` base agent — terminal TUI and
desktop GUI are the same underlying tool, sharing the workspace:

```console
$ sbx run opencode \
    --display \
    --kit "git+https://github.com/docker/sbx-kits-contrib.git#dir=opencode-desktop" \
    ~/my-project
```

The OpenCode Desktop window appears on your host desktop. The `opencode`
terminal session runs concurrently in the same sandbox — both share the
workspace at the path you passed.

You can use any agent that accepts `--kit`, though — the desktop app runs
independently of whichever terminal agent shares the sandbox:

```console
$ sbx run claude --display --kit "git+https://github.com/docker/sbx-kits-contrib.git#dir=opencode-desktop" ~/my-project
$ sbx run shell --display --kit "git+https://github.com/docker/sbx-kits-contrib.git#dir=opencode-desktop" ~/my-project
```

## First install

On the first `sbx run`/`sbx create` with this kit, the OpenCode Desktop
`.deb` (~120 MB) is downloaded from the project's latest GitHub release and
installed with `apt`, which also pulls its runtime library dependencies
(`libgtk-3-0`, `libnss3`, `libsecret-1-0`, etc.) from the Ubuntu mirrors. This
takes roughly 1–2 minutes depending on network speed. Subsequent starts reuse
the installed binary from the sandbox's persistent overlay.

## Model provider access

OpenCode Desktop is a full agent, not just an editor — it needs outbound
access to whichever model provider you sign in with **inside the app**, and
it reads its config from the same place the CLI does
(`~/.config/opencode/opencode.json`, confirmed from strings in the desktop
app's bundled `app.asar` — it's the same OpenCode core, just an Electron
shell around it).

**Paired with the built-in `opencode` base agent, this is already covered.**
That agent's own `spec.yaml` allows egress to Anthropic, OpenAI, GitHub
Copilot, Google, Groq, OpenRouter, and xAI, and wires proxy-managed
credential injection for each (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, etc.).
Network policy and credential injection are sandbox-wide, not per-process —
they're unioned in from every kit composing the sandbox — so the desktop
app picks up the same allowed hosts and injected credentials the terminal
`opencode` session gets, with nothing extra to configure.

**Paired with a different base agent** (`claude`, `shell`, ...), none of
that comes along, so you'll need to allow whichever provider you pick
yourself:

```console
$ sbx policy allow network api.anthropic.com          # Anthropic
$ sbx policy allow network api.openai.com              # OpenAI
$ sbx policy allow network github.com,api.github.com   # GitHub Copilot
$ sbx policy allow network opencode.ai,api.opencode.ai # OpenCode Zen
```

If sign-in or chat requests fail, `sbx policy log <sandbox-name>` shows the
exact host that was blocked.

## How the display surface works

`--display` provisions a Wayland socket inside the microVM and connects it to
the host compositor. OpenCode Desktop is started with
`--ozone-platform=wayland`, so its window is a first-class Wayland surface on
the host — resize, focus, and clipboard work as expected.

The startup command runs as root and immediately drops to uid 1000 via
`setpriv` before exec'ing the app. This keeps the kit portable across base
templates that use different usernames at uid 1000 (e.g. `agent` in the
standard `docker/sandbox-templates`).

`--no-sandbox` is passed because Electron's setuid sandbox helper needs
privileges (or working user namespaces) that an unprivileged container
doesn't reliably grant; the sandbox container itself is the security
boundary here, same as every other Electron-based kit in this repo (see
[`vscode/`](../vscode)).

## Troubleshooting

**Window doesn't appear**

Check that you passed `--display` to `sbx run`/`sbx create` and that
`feature.sandbox-display` is enabled:

```console
$ sbx settings get feature.sandbox-display
```

Also check that sbx is v0.39.0 or later (`sbx version`).

**Window was closed — how to reopen it**

The launch wrapper is on PATH inside the sandbox. Run it from any terminal
session in the sandbox:

```console
$ opencode-desktop &
```

**Sign-in or chat requests fail**

See [Model provider access](#model-provider-access) above — this kit doesn't
pre-allow any specific model provider's network egress.
