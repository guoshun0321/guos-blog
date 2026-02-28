---
title: "Browser（浏览器控制）走读（Deep Dive）"
description: "代码位置：`/root/github/openclaw/src/browser/*`"
pubDate: 2026-02-26T20:00:00Z
---

# Browser（浏览器控制）走读（Deep Dive）

> 代码位置：`/root/github/openclaw/src/browser/*`

## 1. Browser 子系统的定位

Browser 子系统为 OpenClaw 提供“可控浏览器”能力（类似远程操控 Chrome/Chromium）：

- 打开/切换 tab
- 抓取快照（snapshot）
- 执行动作（click/type/press/drag/select/evaluate 等）
- 截图、PDF、文件上传
- 支持两类模式：
  - openclaw 托管的浏览器 profile
  - 通过 extension relay 接管用户现有 Chrome tab（从文件名看：`extension-relay.*`）

依赖上看：仓库使用 `playwright-core`，并且有很多 `pw-*` 文件（Playwright）。

## 2. 目录结构的关键模块（从文件名推断）

- `cdp.ts` / `cdp.helpers.ts`：CDP（Chrome DevTools Protocol）交互
- `chrome.*`：Chrome 可执行文件发现、profile 装饰、extension manifest 等
- `pw-session.ts`：Playwright session 管理（targetId/page 选择、fallback 等）
- `pw-tools-core.*`：动作/快照/下载/交互等核心实现
- `server.ts` / `server-context.ts` / `routes/*`：Browser control server 的 HTTP API/路由
- `control-auth.ts` / `http-auth.ts` / `csrf.ts`：Browser 控制面的鉴权/CSRF
- `extension-relay.ts`：浏览器扩展 relay（把已有 Chrome tab 的事件/控制权接入）

## 3. 与 Gateway 的关系

Gateway 启动时（`startGatewaySidecars(...)`）会根据配置决定是否启动 browser control server。

然后 agent 的 `browser` tool 调用，会通过 Gateway 转发到 browser server（或直接调用其内部 API），实现“在工具层面可调用”。

> 后续如果你希望我做更精确的走读：我建议下一步读 `src/browser/server.ts`、`src/browser/server-context.ts`、`src/browser/pw-session.ts`、`src/browser/pw-tools-core.*`，把 HTTP 路由与 tool contract 对齐。

