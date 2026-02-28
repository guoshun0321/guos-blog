---
title: "CLAUDE.md"
description: "This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository."
pubDate: 2026-02-26
---

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> The codebase lives at `/root/github/openclaw`. The existing `AGENTS.md` (symlinked as `CLAUDE.md`) there covers commit workflow, release procedures, and operational details. This file focuses on architecture and development commands.

---

## Build & Development Commands

**Package manager:** pnpm (v10.23.0). Node 22+ required. Bun is preferred for TypeScript execution.

```bash
pnpm install                  # Install deps
pnpm build                    # Full build (bundles Canvas, runs tsdown, generates protocol schema)
pnpm gateway:watch            # Dev server with auto-reload (skips channel connections by default)
pnpm openclaw ...             # Run CLI from source via Bun
pnpm ui:dev                   # Web UI dev server
```

**Quality checks (run before committing):**
```bash
pnpm check                    # format:check + tsgo + lint (full CI gate)
pnpm tsgo                     # TypeScript type checking only
pnpm format:fix               # Auto-fix formatting (oxfmt)
pnpm lint:fix                 # Auto-fix linting (oxlint --type-aware)
```

**Testing:**
```bash
pnpm test                     # Unit + integration tests (vitest, parallel)
pnpm test:coverage            # With V8 coverage (70% threshold required)
pnpm test:e2e                 # End-to-end tests
OPENCLAW_TEST_PROFILE=low OPENCLAW_TEST_SERIAL_GATEWAY=1 pnpm test  # Low-memory hosts
CLAWDBOT_LIVE_TEST=1 pnpm test:live   # Live tests against real APIs
```

**Commits:** Use `scripts/committer "<msg>" <file...>` (scopes staging). Pre-commit hooks via `prek install`.

---

## Architecture Overview

OpenClaw is a **local-first, multi-channel AI gateway**. A single long-lived WebSocket server (the Gateway) runs on the user's device and bridges 13+ messaging platforms to an AI agent runtime. No user data passes through external servers by default.

### Control flow

```
Messaging channels (WhatsApp, Telegram, Slack, Discord, Signal, Teams, iMessage, ...)
        │
        ▼
  Gateway Control Plane  (ws://127.0.0.1:18789)
  ┌─ Session management      ─ Plugin/hook system
  ├─ Message routing         ─ Media pipeline
  ├─ Cron/webhook automation ─ Memory/RAG
  └─ Agent invocation (Pi agent in RPC mode)
        │
   ┌────┴──────────────┬──────────────┐
   ▼                   ▼              ▼
 CLI tools         Companion apps  Web UI (Lit)
 (gateway, agent,  macOS menu bar  Control UI
  send, config…)   iOS/Android     WebChat
                   nodes           Canvas (A2UI)
```

### Monorepo layout

| Path | Purpose |
|---|---|
| `src/` | Core gateway, CLI, channels, agents, media pipeline |
| `extensions/*/` | Workspace packages: channel plugins (discord, slack, msteams, matrix, zalo…) and feature plugins (voice-call, memory-lancedb, diagnostics-otel…) |
| `packages/*/` | Workspace-internal shared packages |
| `ui/` | Web UI built with Lit + Vite (Control UI, WebChat, Canvas) |
| `apps/macos/` | Swift/SwiftUI menu bar app + voice wake |
| `apps/ios/` | Swift/UIKit node (camera, screen, location) |
| `apps/android/` | Kotlin node (Talk Mode, screen recording) |
| `docs/` | Mintlify docs (docs.openclaw.ai) |
| `dist/` | Compiled output (main entry: `dist/index.js`) |

### Key source modules under `src/`

- **`cli/`** — Commander.js wiring, `createDefaultDeps` DI container, `progress.ts` for spinners
- **`commands/`** — All CLI command implementations
- **`gateway/`** — WebSocket server, session management, routing, hooks, boot logic
- **`channels/`** — Built-in channel drivers: `telegram/`, `discord/`, `slack/`, `signal/`, `imessage/`, `web/` (WhatsApp)
- **`agents/`** — Agent invocation, auth profiles, bash tools
- **`infra/`** — Port management, binary detection, environment setup
- **`media/`** — Image/audio/video pipeline (sharp, ffmpeg, pdfjs)
- **`canvas-host/`** — A2UI (Agent-to-UI) canvas protocol host
- **`terminal/`** — `palette.ts` (shared colors), `table.ts` (ANSI-safe tables)

### Gateway WebSocket protocol

Clients communicate via JSON frames:
- **Request:** `{type:"req", id, method, params}`
- **Response:** `{type:"res", id, ok, payload|error}`
- **Event:** `{type:"event", event, payload, seq?, stateVersion?}`

Handshake: client sends `connect` with device identity → server replies `hello-ok` + presence snapshot. Protocol schema generated to `dist/protocol.schema.json` and Swift models at `apps/macos/Sources/OpenClawProtocol/GatewayModels.swift`.

### Extension/plugin system

Extensions are separate `package.json` workspaces under `extensions/`. Runtime deps must be in `dependencies` (not `devDependencies`). The root `openclaw` package should go in `devDependencies`/`peerDependencies` only—runtime resolves `openclaw/plugin-sdk` via jiti alias. Never use `workspace:*` in a plugin's `dependencies`.

### Configuration

Config file: `~/.openclaw/openclaw.json` (JSON5 format). Key sections: `agent` (model, workspace, sandbox), `gateway` (bind, port, auth), `channels` (per-channel tokens/allowlists), `agents` (multi-agent routing).

---

## Important Conventions

- **TypeScript ESM, strict mode.** No `any`, no `@ts-nocheck`, no prototype mutation for mixins—use explicit class inheritance/composition.
- **File size:** aim for ≤500–700 LOC; split/refactor when it improves clarity.
- **CLI UI:** use `src/cli/progress.ts` for spinners; `src/terminal/palette.ts` for colors (never hardcode); `src/terminal/table.ts` for tabular output.
- **Tool schemas:** no `anyOf`/`oneOf`/`allOf`, no raw `format` property keys, no `Type.Union` in input schemas.
- **Channels:** when refactoring routing, allowlists, or onboarding, update *all* channels—both built-in (`src/channels`) and extensions (`extensions/*`).
- **Docs (Mintlify):** internal links are root-relative with no `.md` extension. Avoid em dashes/apostrophes in headings (breaks anchors).
- **Naming:** **OpenClaw** in headings/docs; `openclaw` for CLI, paths, config keys.
- **Version bumping:** update `package.json`, both iOS `Info.plist` files, both macOS `Info.plist` files, `apps/android/app/build.gradle.kts`, and `docs/install/updating.md`. Do not touch `appcast.xml` unless cutting a Sparkle release.
- **`pnpm.patchedDependencies`** entries must use exact versions (no `^`/`~`). Patching requires explicit approval.
- **Never update** the Carbon dependency.
