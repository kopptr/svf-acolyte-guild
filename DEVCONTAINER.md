# Development Container

This project uses the [Dev Containers spec](https://containers.dev/) to give
every contributor an identical, isolated environment with the
[Claude Code](https://code.claude.com/docs) CLI pre-installed. The
configuration lives in [`.devcontainer/`](.devcontainer/):

| File | Purpose |
| --- | --- |
| `devcontainer.json` | Container settings: base image build, features, mounts, VS Code integration |
| `Dockerfile` | Base image and any OS packages/tools you add for this project |

It's built and run with **[Podman](https://podman.io/)**, not Docker. Nothing
here talks to a Docker daemon or `/var/run/docker.sock`.

## Prerequisites

- **Podman** installed (`podman --version`). On Linux, rootless mode is the
  default and what this config assumes.
- Rootless podman needs subordinate UID/GID ranges configured for your user
  (usually already true on Fedora/Debian/Ubuntu out of the box). Check with:
  ```sh
  podman unshare cat /proc/self/uid_map
  ```
- **For VS Code:** the
  [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)
  extension.
- **For the CLI (no VS Code):** Node.js on the host, then:
  ```sh
  npm install -g @devcontainers/cli
  ```

## Option A: VS Code

VS Code's Dev Containers extension drives Docker by default. Point it at
`podman` instead, either per-project (`.vscode/settings.json`, not tracked by
this template) or in your user settings:

```json
{
  "dev.containers.dockerPath": "podman"
}
```

Then open this folder in VS Code and run **Dev Containers: Reopen in
Container** from the Command Palette (`Ctrl+Shift+P`). VS Code builds the
image, starts the container, and installs the Claude Code extension
automatically (from the `ghcr.io/anthropics/devcontainer-features/claude-code`
feature in `devcontainer.json`).

## Option B: CLI only

From the repository root:

```sh
# Build and start the container
devcontainer up --workspace-folder . --docker-path podman

# Open a shell inside it
devcontainer exec --workspace-folder . --docker-path podman bash
```

Re-run `devcontainer up` after changing `devcontainer.json` or the
`Dockerfile` to rebuild.

## First run: signing in to Claude Code

Inside the container:

```sh
claude
```

Follow the browser authentication prompt. If the browser callback can't reach
the container (common when port forwarding doesn't cooperate), copy the code
shown in the browser and paste it at the `Paste code here if prompted` prompt.

Your credentials, settings, and session history persist across container
rebuilds because `~/.claude` is mounted from a named Podman volume
(`claude-code-config-<id>` in `devcontainer.json`), not baked into the image
or read from the host — so you only need to sign in once per project, and the
image never contains your token.

## Customizing for your project

- **OS packages / language runtimes:** add `RUN apt-get install ...` (or a
  language-specific devcontainer feature) to `Dockerfile` or the `features`
  block in `devcontainer.json`. Browse available features at
  [containers.dev/features](https://containers.dev/features).
- **VS Code extensions:** add IDs under
  `customizations.vscode.extensions` in `devcontainer.json`. Don't add the
  Claude Code extension itself — the `claude-code` feature installs it
  automatically when the container is opened in VS Code or Codespaces.
- **Project-scoped MCP servers:** define them in a `.mcp.json` at the repo
  root (tracked in git) rather than per-user `~/.claude` config, so the whole
  team gets them. Install any binaries the servers need in the `Dockerfile`.
- **Team-wide Claude Code settings:** commit shared permissions, hooks, or
  env config to `.claude/settings.json` (tracked). Personal overrides go in
  `.claude/settings.local.json`, which is gitignored.
- **Baseline Claude Code policy for this container:** uncomment the
  `containerEnv` block in `devcontainer.json` to disable non-essential
  network traffic (telemetry/error reporting) and/or stop the CLI from
  auto-updating in place.
- **Org-enforced policy that engineers can't override by editing the repo:**
  copy a `managed-settings.json` into `/etc/claude-code/` from the
  `Dockerfile` — Claude Code applies it at the highest precedence in the
  [settings hierarchy](https://code.claude.com/docs/en/settings#how-scopes-interact).
  For policy that must survive even if someone edits the Dockerfile, use
  [server-managed settings](https://code.claude.com/docs/en/server-managed-settings)
  instead. See [Set up Claude Code for your organization](https://code.claude.com/docs/en/admin-setup)
  for the full list of controls (permission rules, tool restrictions, MCP
  allowlists).

## Podman-specific notes

- **`--userns=keep-id`** (in `runArgs`) maps the container's non-root user to
  the *host* user that invoked `podman`, so files you create inside the
  container land in your workspace owned by you, not by an arbitrary
  subordinate UID. This only lines up cleanly if your host user's UID is
  `1000` (the default first user on most single-user Fedora/Debian/Ubuntu
  installs). If `id -u` on your host isn't `1000`, either drop `keep-id` and
  accept subordinate-UID ownership, or add a matching `userUid`/`userGid` to
  the `ghcr.io/devcontainers/features/common-utils` feature.
- **`relabel=private`** on the workspace and any bind mounts is podman's
  `--mount` equivalent of the `:Z` suffix, needed on SELinux-enforcing hosts
  (Fedora, RHEL, CentOS) so the container is actually allowed to read/write
  the bind-mounted directory. If you add your own bind mounts, add
  `relabel=private` (or `relabel=shared`/`:z` if multiple containers need the
  same path) or you'll hit `Permission denied` inside the container.
- These flags are podman-specific and will error under plain Docker; remove
  them if you ever switch container engines.

## Secrets and per-clone environment variables

`.devcontainer/.env` holds environment variables for *your own* clone —
API keys, cloud credentials, anything a tool inside the container needs
that shouldn't be shared with the team or committed to git:

- **`.devcontainer/.env.example`** is the tracked template: comments and
  commented-out `KEY=` examples, no real values.
- **`.devcontainer/.env`** is gitignored (`.gitignore` already has `.env`,
  `.env.*`, and `!.env.example` patterns) and is created automatically —
  empty except for the template's comments — the first time you build the
  container, via `initializeCommand` in `devcontainer.json`. It's never
  overwritten if it already exists, so your values are safe across
  rebuilds.
- Fill in whatever you need in your local `.env`, then rebuild the
  container (see above) — `--env-file` in `runArgs` loads it into the
  container's environment at creation time. Just restarting an
  already-created container won't pick up changes; it needs the same
  rebuild step as any other `devcontainer.json`/`Dockerfile` change.

This is different from the `containerEnv` block further up: `containerEnv`
is tracked in git and meant for non-secret, team-wide settings everyone
should get. `.env` is personal and per-clone — never put a real secret in
`containerEnv` or anywhere else that's tracked.

## Security notes

- Don't bind-mount host secrets (`~/.ssh`, cloud credential files, etc.) into
  the container. Prefer repository-scoped or short-lived tokens passed via
  `containerEnv`, and for cloud LLM providers (Bedrock, Vertex, Foundry) pass
  credentials as environment variables rather than mounting files — see the
  [Claude Code devcontainer docs](https://code.claude.com/docs/en/devcontainer)
  for provider-specific instructions.
- The dev container isolates Claude Code from your host, but a container
  process can still modify anything in the bind-mounted workspace (which is
  your real repo) and reach the network. If you run `claude` with
  `--dangerously-skip-permissions`, only do it against trusted repositories,
  and consider adding an egress firewall (see the reference `init-firewall.sh`
  in [anthropics/claude-code](https://github.com/anthropics/claude-code/tree/main/.devcontainer))
  if you need stronger guarantees. `--dangerously-skip-permissions` also
  requires a non-root `remoteUser`, which this template already sets.

## Troubleshooting

- **`Permission denied` on workspace files:** almost always the SELinux label
  issue above — confirm `relabel=private` is present on the mount, or check
  `getenforce` isn't blocking podman.
- **Files created in the container are owned by a strange UID on the host:**
  see the `--userns=keep-id` note above.
- **`claude` command not found:** the `claude-code` feature failed to install
  Node.js/npm. Add `ghcr.io/devcontainers/features/node:1` explicitly (already
  included in this template) or install Node in the `Dockerfile`.
- **Podman machine (macOS/Windows only):** if `podman info` fails with a
  connection error, run `podman machine init && podman machine start` first.
