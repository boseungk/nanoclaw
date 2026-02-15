# NanoClaw

Personal assistant orchestrator with WhatsApp I/O and container-isolated agent runtimes.
See `README.md` for setup and `docs/REQUIREMENTS.md` for architecture decisions.

## Quick Context

- Single Node.js host process.
- Messages are stored in SQLite and processed by per-group isolated containers.
- Runtime is selectable:
  - `AGENT_RUNTIME=claude` (default)
  - `AGENT_RUNTIME=codex`
- Each group has isolated workspace, IPC namespace, and runtime session state.

## Runtime And Auth

- Claude runtime:
  - Uses Claude Agent SDK in `container/agent-runner/src/index.ts`.
  - Auth via `CLAUDE_CODE_OAUTH_TOKEN` or `ANTHROPIC_API_KEY`.
- Codex runtime:
  - Uses Codex CLI (`codex exec` / `codex exec resume`) in `container/agent-runner/src/index.ts`.
  - Preferred auth is OAuth session on host:
    - `codex login --device-auth`
    - Host `~/.codex/auth.json` is synced into each group session directory automatically.
  - API key fallback via `OPENAI_API_KEY`.

## Key Files

| File | Purpose |
|------|---------|
| `src/index.ts` | Main orchestrator (loops, queue integration, runtime dispatch) |
| `src/container-runner.ts` | Container mounts, secrets/auth sync, process lifecycle |
| `src/task-scheduler.ts` | Due task polling and scheduled agent execution |
| `src/ipc.ts` | MCP IPC watcher and task/group control actions |
| `src/db.ts` | SQLite schema and data access (`runtime_sessions` included) |
| `src/config.ts` | Runtime/config env constants (`AGENT_RUNTIME`, `CODEX_MODEL`, etc.) |
| `src/channels/whatsapp.ts` | WhatsApp auth and inbound/outbound transport |
| `container/agent-runner/src/index.ts` | In-container runtime adapter (Claude/Codex) |
| `container/agent-runner/src/ipc-mcp-stdio.ts` | MCP stdio server for host IPC bridge |
| `container/skills/agent-browser/SKILL.md` | Browser skill synced into runtime sessions |

## Skills

| Skill | When to Use |
|-------|-------------|
| `/setup` | First-time install, auth, service bootstrap |
| `/customize` | Add channels/integrations, behavior changes |
| `/debug` | Runtime/container/log troubleshooting |

## Development Workflow

Run commands directly; do not ask the user to run them.

```bash
npm run dev
npm run typecheck
npm test
npm run build
cd container/agent-runner && npm run build
./container/build.sh
```

## Completion Checks

Before reporting completion for code changes, run:

```bash
npm run typecheck
npm test
npm run build
cd container/agent-runner && npm run build
```

## Service Management

- macOS (launchd):

```bash
launchctl load ~/Library/LaunchAgents/com.nanoclaw.plist
launchctl unload ~/Library/LaunchAgents/com.nanoclaw.plist
```

- Linux: use your process supervisor (`systemd`, `pm2`, or container runtime). No `launchctl`.

## Container Build Cache Notes

Apple Container build cache can retain stale context for `COPY` layers.
If rebuilds do not reflect source changes, force-clean builder state:

```bash
container builder stop && container builder rm && container builder start
./container/build.sh
```

Verify rebuilt source is present:

```bash
container run -i --rm --entrypoint wc nanoclaw-agent:latest -l /app/src/index.ts
```
