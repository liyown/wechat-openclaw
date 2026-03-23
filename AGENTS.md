# Repository Guidelines

## Project Structure & Module Organization
This repository is a small TypeScript OpenClaw plugin for WeCom/WebSocket access. The main entrypoint is [`index.ts`](/Users/liuyaowen/Workspace/javascript/weixin-access/index.ts), which registers plugin metadata, config, status, and gateway behavior. Shared runtime and reporting helpers live in `common/`. WebSocket-specific transport, adapters, and queue handling live in `websocket/`. Plugin manifest data is in `openclaw.plugin.json`. Build output should go to `dist/` and must not be committed.

## Build, Test, and Development Commands
There are no npm scripts yet, so use direct TypeScript commands from the repo root:

- `npm install`: install dependencies.
- `npx tsc -p tsconfig.json`: type-check and compile declarations into `dist/`.
- `npm pack`: create a package tarball to verify publishable contents.

Run these before opening a PR. If you add new tooling, wire it into `package.json` scripts instead of relying on ad hoc local commands.

## Coding Style & Naming Conventions
Use TypeScript with ES modules, strict typing, and explicit imports ending in `.js` for local runtime paths. Match the existing style: 2-space indentation, single quotes, semicolons omitted, and small focused modules. Use `camelCase` for functions and variables, `PascalCase` for types/interfaces, and kebab-case filenames for multiword modules such as `message-handler.ts`.

Keep comments brief and only where the message flow or plugin lifecycle is not obvious. Prefer extending existing modules in `common/` or `websocket/` over adding new top-level files.

## Testing Guidelines
No automated test framework is configured yet. Until one is added, treat `npx tsc -p tsconfig.json` as the minimum validation gate and manually exercise the affected OpenClaw/WebSocket flow. When adding tests, place them under `tests/` or alongside the module as `*.test.ts`, and cover message parsing, queue behavior, and connection-state transitions first.

## Commit & Pull Request Guidelines
The repository has no commit history yet, so use Conventional Commit style going forward, for example `feat: add websocket retry backoff` or `fix: guard empty prompt payloads`. Keep PRs small and include:

- a short description of behavior changes
- linked issue or task ID when available
- manual verification steps
- screenshots or logs when the change affects runtime status or message handling

## Security & Configuration Tips
Do not commit real tokens, `wsUrl` secrets, or account identifiers. Keep credentials in local OpenClaw configuration, and redact sensitive values from logs and PR screenshots.
