# devcontainer

Sandboxed environment for running Claude Code against this project. Adapted from [Anthropic's reference devcontainer](https://github.com/anthropics/claude-code/tree/main/.devcontainer).

## Files

Three files in `.devcontainer/`:

- **`devcontainer.json`** — workspace bind-mounted at `/workspace`, persistent volumes for shell history and `~/.claude` config (so auth survives rebuilds), `NET_ADMIN`/`NET_RAW` for the firewall, runs as non-root `node`.
- **`Dockerfile`** — Node 20 base + reference toolchain, plus `ripgrep`, `tree`, `htop`, `tmux`, `python3`/`pip`/`venv`, `build-essential`. Claude Code preinstalled globally.
- **`init-firewall.sh`** — default-deny outbound, allow-list for npm, GitHub, Anthropic API, VS Code marketplace, plus PyPI. Runs on every container start via `postStartCommand`.

## Adaptations vs. the reference

- `TZ` default is `Etc/UTC` instead of `America/Los_Angeles` (sensible for a remote Linux box; override with `TZ=...` in your shell env).
- Extra shell tools: `ripgrep`, `tree`, `htop`, `tmux`, `python3`/`pip`/`venv`, `build-essential`.
- `pypi.org` + `files.pythonhosted.org` added to the firewall allow-list.

## Workflow: VS Code Remote-SSH → Dev Container

This nesting works automatically — no JSON changes needed. The Dev Containers extension runs on the SSH host and uses Docker there.

1. On the remote Linux box: install Docker, make sure your user is in the `docker` group (`sudo usermod -aG docker $USER`, then re-login).
2. From local VS Code: install the **Dev Containers** and **Remote - SSH** extensions.
3. Connect to the box via Remote-SSH, open this project folder (on the remote).
4. Command Palette → **Dev Containers: Reopen in Container**. First build takes ~3–5 min.
5. Open a terminal in VS Code (`` Ctrl+` ``) → run `claude`. Auth is preserved across rebuilds via the named volume.

### Things that "just work" via the Dev Containers extension

You don't need to wire these up:

- SSH agent forwarding from your laptop → SSH host → container, so `git push` over SSH uses your local key.
- `~/.gitconfig` `user.name`/`user.email` is copied in automatically.
- `known_hosts` is propagated.

## Caveats

- Run `claude --dangerously-skip-permissions` only inside this container — that's the whole point of the firewall + isolation. Per Anthropic's docs: a malicious project inside the container could still exfiltrate anything reachable *from* the container (including Claude credentials in the named volume), so still only use it for trusted repos.
- If Claude needs to reach a domain not on the allow-list (private package registry, internal docs, etc.), add it to the resolution loop in `init-firewall.sh`.
- The bind mount uses `${localWorkspaceFolder}`, which resolves to the path on the SSH host — only that single folder is exposed to the container. Nothing else from the Linux box's filesystem is visible.

## Where Claude credentials are stored

Inside the container, Claude Code writes to `/home/node/.claude/`:

- `.credentials.json` — auth tokens
- `settings.json`, `projects/`, `todos/`, etc.

That path is a **Docker named volume** (`claude-code-config-${devcontainerId}`), not a bind mount. On the SSH host it lives at:

```
/var/lib/docker/volumes/claude-code-config-<devcontainerId>/_data/.credentials.json
```

Owned by root on the host (the Docker daemon writes it), readable by the `node` user inside the container. Implications:

- **Persists across rebuilds** — you only authenticate once per devcontainer.
- **Plaintext on Linux** — no keychain integration. Anyone with root on the SSH host (or any container that mounts the same volume) can read the token.
- **Survives container deletion** until you explicitly `docker volume rm claude-code-config-<id>`.
- **Not visible to `/workspace`** — different mount, so a malicious project file can't `cat` the credentials via the workspace path. But code running as the `node` user inside the container *can* read them directly. That's the threat model behind the `--dangerously-skip-permissions` warning.

## Exiting the dev container

In VS Code: Command Palette (Cmd/Ctrl+Shift+P) → **"Dev Containers: Reopen Folder Locally"** (or "Reopen Folder in SSH" if you came in via Remote-SSH). That detaches the window and reopens the same folder on the host.

Other options:

- Click the green **`><` remote indicator** in the bottom-left corner → pick "Reopen Folder Locally" / "Close Remote Connection".
- Just **close the VS Code window** — the container keeps running in the background but you're disconnected.

The container itself isn't deleted when you exit; it's stopped (or left running) and reused next time you reopen. To fully tear it down:

- Command Palette → **"Dev Containers: Rebuild Container"** — wipes and recreates the container.
- From a host shell: `docker ps` + `docker rm -f <id>` removes it. Named volumes (including the credentials volume) survive unless you also `docker volume rm <name>`.
