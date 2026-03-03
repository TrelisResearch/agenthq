# Agent HQ

Browser-based control plane for managing coding agents (Claude Code, Codex CLI, Cursor Agent, etc.) across multiple environments.

<img width="3451" height="1990" alt="image" src="https://github.com/user-attachments/assets/8a917cbb-2fc0-49b8-8a0f-71a63710bde6" />

\> [check video demo](https://x.com/RickLamers/status/2018084764285382851)

## Quick Start

### Prerequisites

- Node.js 22+ (with `--experimental-sqlite` support — see note below)
- pnpm 9+
- Go 1.23+
- At least one agent CLI installed (e.g. `claude`, `codex`, `cursor-agent`)

> **Node.js `node:sqlite` note:** The server uses the built-in `node:sqlite` module. On Node.js 22–23 you must pass `--experimental-sqlite` via the `NODE_OPTIONS` env var (see below). Node.js 24+ includes it without a flag.

### Setup

```bash
# Install dependencies
pnpm install

# Build all packages (shared, server, web)
pnpm build

# Build daemon
cd daemon && go build -o agenthq-daemon ./cmd/agenthq-daemon && cd ..
```

### Running Locally

You need three things running: the **server**, the **web UI**, and the **daemon**.

#### 1. Configure a daemon auth token

The daemon authenticates to the server with a shared token. Create the config file in your workspace:

```bash
WORKSPACE=~/my-repos   # directory containing your git repos

mkdir -p "$WORKSPACE/.agenthq-meta"
echo '{"daemonAuthToken":"my-secret-token"}' > "$WORKSPACE/.agenthq-meta/config.json"
```

Or pass it as an env var to the server: `AGENTHQ_DAEMON_AUTH_TOKEN=my-secret-token`.

#### 2. Start the server

```bash
NODE_OPTIONS="--experimental-sqlite" \
AGENTHQ_WORKSPACE=~/my-repos \
AGENTHQ_DEFAULT_USERNAME=admin \
AGENTHQ_DEFAULT_PASSWORD=changeme \
pnpm --filter @agenthq/server dev
```

#### 3. Start the web UI

```bash
pnpm --filter @agenthq/web dev
```

#### 4. Start the daemon

```bash
AGENTHQ_AUTH_TOKEN=my-secret-token \
./daemon/agenthq-daemon --workspace ~/my-repos
```

The `--workspace` flag tells the daemon where to scan for git repositories. Each top-level directory containing a `.git` folder will appear as a repo in the UI.

> **Launching from inside Claude Code?** The daemon inherits `CLAUDECODE` and `CLAUDE_CODE_ENTRYPOINT` env vars, which prevent nested Claude Code sessions. Strip them:
> ```bash
> env -u CLAUDECODE -u CLAUDE_CODE_ENTRYPOINT \
>   AGENTHQ_AUTH_TOKEN=my-secret-token \
>   ./daemon/agenthq-daemon --workspace ~/my-repos
> ```

#### 5. Open the UI

Go to http://localhost:5173, log in with the credentials you set, select a repo, create a worktree, and spawn an agent.

### Using the Makefile (Linux)

The root Makefile provides convenience commands but uses Linux-specific tools (`ss`, `fuser`) and won't work on macOS without modification.

```bash
make start WORKSPACE=~/my-repos
make status
make tail-logs
make stop
```

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Browser Client                               │
│           React + Vite + shadcn + xterm.js                          │
└────────────────────────────────────────┬────────────────────────────┘
                                         │ WebSocket
                                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          agenthq-server                             │
│                    Node + Fastify + TypeScript                      │
└───────────────────┬─────────────────────────────────────────────────┘
                    │ WebSocket
                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                          agenthq-daemon                              │
│                               Go                                     │
│                     PTY spawning, worktree mgmt                      │
└──────────────────────────────────────────────────────────────────────┘
```

## Project Structure

```
agenthq/
├── packages/
│   ├── server/         # @agenthq/server - Fastify API + WebSocket
│   ├── web/            # @agenthq/web - React UI
│   └── shared/         # @agenthq/shared - Protocol types
├── daemon/             # Go daemon binary
├── package.json        # Root workspace config
└── pnpm-workspace.yaml
```

## Environment Variables

### Server

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `AGENTHQ_WORKSPACE` | Yes | — | Path to directory containing your git repos |
| `AGENTHQ_PORT` | No | `3000` | Server port |
| `AGENTHQ_DAEMON_AUTH_TOKEN` | No | — | Shared secret for daemon auth (alternative to config file) |
| `AGENTHQ_DEFAULT_USERNAME` | No | — | Seed a default login user on startup |
| `AGENTHQ_DEFAULT_PASSWORD` | No | — | Password for the default user |
| `NODE_OPTIONS` | No | — | Set to `--experimental-sqlite` on Node.js 22–23 |

### Daemon

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `AGENTHQ_AUTH_TOKEN` | Yes* | — | Shared secret matching the server's daemon auth token |
| `AGENTHQ_SERVER_URL` | No | `ws://localhost:3000/ws/daemon` | WebSocket URL to connect to |
| `AGENTHQ_ENV_ID` | No | auto-generated | Environment ID |

\* Required when the server has a daemon auth token configured (which it should).

The daemon also accepts a `--workspace` flag:

```bash
./agenthq-daemon --workspace /path/to/repos
```

## Supported Agents

| Agent | Command | Description |
|-------|---------|-------------|
| Claude Code | `claude` | Anthropic coding agent |
| Codex CLI | `codex` | OpenAI coding agent |
| Cursor Agent | `cursor-agent` | Cursor coding agent |
| Kimi CLI | `kimi` | Moonshot coding agent |
| Droid CLI | `droid` | Factory AI coding agent |
| Terminal | `bash` | Plain shell |

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `ERR_UNKNOWN_BUILTIN_MODULE: node:sqlite` | Set `NODE_OPTIONS="--experimental-sqlite"` or upgrade to Node.js 24+ |
| `Daemon connection rejected: no daemon auth token configured` | Set `AGENTHQ_DAEMON_AUTH_TOKEN` env var on server, or create `.agenthq-meta/config.json` in workspace |
| `Invalid auth token` | Ensure `AGENTHQ_AUTH_TOKEN` (daemon) matches `AGENTHQ_DAEMON_AUTH_TOKEN` (server) |
| `Claude Code cannot be launched inside another Claude Code session` | Launch daemon with `env -u CLAUDECODE -u CLAUDE_CODE_ENTRYPOINT` |
| `No workspace configured, returning empty repos list` | Pass `--workspace /path/to/repos` when starting the daemon |
| Makefile commands fail on macOS | Use the manual commands instead — the Makefile uses Linux-specific tools (`ss`, `fuser`) |

## License

MIT
