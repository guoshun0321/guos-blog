---
title: "3. 项目的主要包/模块（Major Packages）"
description: "本节回答：仓库中哪些目录/包最关键，它们各自负责什么。"
pubDate: 2026-02-26T12:00:00Z
---

# 3. 项目的主要包/模块（Major Packages）

本节回答：仓库中哪些目录/包最关键，它们各自负责什么。

## 3.1 根包（workspace root: `.`）

- 包名：`openclaw`
- workspace：pnpm
- 运行时语言：TypeScript（构建到 `dist/`）
- bin：`openclaw.mjs`

核心代码都在 `src/`，并在发布时包含：`dist/`、`docs/`、`extensions/`、`skills/` 等（见根 `package.json#files`）。

## 3.2 `src/` 核心模块（按目录）

下面是 `src/` 下的重要子系统（从目录可见）：

- `src/gateway/`：Gateway 服务端核心（控制平面）
- `src/cli/`：CLI 命令实现（gateway/doctor/models/memory/channels/nodes/...）
- `src/agents/`：agent runtime（主循环、工具调用对接、会话绑定等）
- `src/sessions/`：会话管理、状态、策略
- `src/channels/`：渠道抽象 + 可能的内置渠道实现
- `src/plugins/`：插件系统
- `src/plugin-sdk/`：插件 SDK（对外 exports）
- `src/browser/`：浏览器控制工具
- `src/canvas-host/`：Canvas host 与 A2UI（构建脚本里有 bundle）
- `src/nodes*` / `src/node-host/`：设备节点连接与命令
- `src/memory/`：记忆/检索相关（向量库/embedding provider 可能在 extensions 里）
- `src/cron/`：cron jobs 与定时唤醒
- `src/routing/`：路由（不同 agent / session / channel 的路由规则）
- `src/security/`、`src/pairing/`：安全/配对
- `src/web/`：Web surface（控制 UI / WebChat）

> 你要抓主干，优先读：`src/gateway/*`、`src/agents/*`、`src/sessions/*`、`src/channels/*`、`src/plugins/*`、`src/cli/program/*`。

## 3.3 `extensions/*`（扩展包）

本仓库 `extensions/` 非常关键，覆盖：

### 渠道类扩展（举例）

- `extensions/telegram`
- `extensions/discord`
- `extensions/slack`
- `extensions/whatsapp`
- `extensions/signal`
- `extensions/googlechat`
- `extensions/imessage` / `extensions/bluebubbles`
- `extensions/msteams`
- `extensions/matrix`
- `extensions/irc`
- `extensions/line`
- `extensions/twitch`
- `extensions/feishu`

### 能力类/基础设施扩展（举例）

- `extensions/memory-core`、`extensions/memory-lancedb`：记忆与向量库实现（推断）
- `extensions/diagnostics-otel`：可观测性/otel（推断）
- `extensions/thread-ownership`：线程/会话归属管理（推断）
- `extensions/llm-task`：LLM 任务编排（推断）
- `extensions/copilot-proxy`：代理/转发（推断）
- 各种 `*-auth`：如 `google-gemini-cli-auth`、`qwen-portal-auth` 等（OAuth/portal 辅助，推断）

> 建议做法：把 `extensions/*` 当作“可插拔模块库”，先从你关心的渠道（比如 telegram/discord）或能力（memory）入手。

## 3.4 `ui/`（前端/控制台）

`ui/package.json` 表明这是独立 package（细节需再读 ui 的 package.json 与 src）。

## 3.5 `packages/*`

当前可见：

- `packages/clawdbot`
- `packages/moltbot`

它们很可能是相关 bot/运行时封装或示例产品（需要进一步阅读各自 package.json/README）。

## 3.6 `apps/*`

- `apps/android`
- `apps/ios`
- `apps/macos`
- `apps/shared`

用于 companion apps 与 node 能力（camera/screen/canvas/voice wake 等）。
