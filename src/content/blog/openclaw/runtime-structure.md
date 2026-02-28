---
title: "1. 项目运行时结构（Runtime Structure）"
description: "本节回答：OpenClaw 在“跑起来”时，进程/入口/模块如何拼装；有哪些关键运行时组件；它们如何连接。"
pubDate: 2026-02-27
---

# 1. 项目运行时结构（Runtime Structure）

本节回答：OpenClaw 在“跑起来”时，进程/入口/模块如何拼装；有哪些关键运行时组件；它们如何连接。

## 1.1 入口与启动链路（CLI → Runtime）

仓库根 `package.json` 显示：

- 包名：`openclaw`
- bin：`openclaw -> openclaw.mjs`
- 构建产物入口：`dist/index.js` / `dist/entry.js`

关键入口文件：

- `openclaw.mjs`：bootstrap，尝试 `import ./dist/entry.js`（或 `.mjs`）
- `src/entry.ts`：真正的 CLI entry，做环境归一化、必要时 respawn Node 以屏蔽 ExperimentalWarning，然后 `import("./cli/run-main.js")`
- `src/cli/run-main.ts`：加载 dotenv、runtime guard、路由 CLI、注册 commander program、再解析并执行命令

### 启动链路图

```mermaid
flowchart TD
  A[用户执行: openclaw ...] --> B[openclaw.mjs (bootstrap)]
  B -->|import| C[dist/entry.js]
  C --> D[src/entry.ts]
  D --> E{是否需要 respawn?\n--disable-warning=ExperimentalWarning}
  E -->|是| F[spawn node child\n并 attach bridge]
  E -->|否| G[src/cli/run-main.ts runCli(argv)]
  G --> H[tryRouteCli (部分命令可直接路由/短路)]
  H -->|未路由| I[buildProgram + registerCore/subcli + plugin CLI]
  I --> J[commander parseAsync 执行子命令]
```

> 备注：开发模式常用 `pnpm dev` / `pnpm openclaw ...`，脚本 `scripts/run-node.mjs` 会在 `dist` 过期时自动触发 `tsdown` 编译，再运行 `openclaw.mjs`。

## 1.2 运行时“控制平面”与连接面

从 README 的架构描述（以及 `src/gateway/`、`src/web/`、`src/nodes/` 等目录）可以总结：

- **Gateway**：核心常驻服务（WebSocket 控制平面），通常绑定 `127.0.0.1:18789`
- **CLI**：通过 Gateway 协议（WS/RPC）进行控制、查询状态、发送消息、运行 agent 等
- **Web 控制面 / WebChat**：由 Gateway 提供 web surface
- **Channels**：WhatsApp/Telegram/Slack/Discord/Signal/iMessage 等消息渠道接入（核心在 `src/channels/` + `extensions/*`）
- **Nodes（设备节点）**：iOS/Android/macOS 节点，可执行 camera/screen/canvas/location/system.run 等（`src/node-host/`、`src/nodes` / `apps/*`）
- **Browser/Canvas 工具面**：自动化浏览器与画布渲染（`src/browser/`、`src/canvas-host/`）

### 运行时组件关系图（宏观）

```mermaid
flowchart LR
  subgraph Surfaces[外部/用户触达面]
    WA[WhatsApp]
    TG[Telegram]
    DC[Discord]
    SL[Slack]
    SIG[Signal]
    WEB[WebChat]
    IOS[iOS/Android/macOS Node]
  end

  subgraph Gateway[OpenClaw Gateway\n(control plane)]
    WS[WebSocket / RPC]
    SES[Sessions]
    AG[Agents]
    TL[Tools: exec/browser/canvas/nodes/message/...]
    CFG[Config + Secrets + Policy]
    PLG[Plugins/Extensions]
  end

  CLI[openclaw CLI] --> WS
  WEB --> WS
  WA --> WS
  TG --> WS
  DC --> WS
  SL --> WS
  SIG --> WS
  IOS --> WS

  WS --> SES
  WS --> AG
  WS --> TL
  WS --> CFG
  WS --> PLG
```

## 1.3 开发/构建运行时

仓库使用 pnpm workspace：

- `pnpm-workspace.yaml`：`., ui, packages/*, extensions/*`
- `scripts/run-node.mjs`：开发运行时如果 `dist` 过期会自动 build（`tsdown exec tsdown --no-clean`）

关键脚本（根 `package.json`）：

- `pnpm dev` / `pnpm start`：走 `node scripts/run-node.mjs`
- `pnpm gateway:watch`：watch gateway（热重载）
- `pnpm build`：构建 dist，同时 bundle canvas a2ui、生成 plugin-sdk dts、写 build info 等

> 这意味着运行时有两种典型形态：
> 1) **from source**：tsx/tsdown 直接跑 TS（开发调试）；
> 2) **installed package**：跑 `dist/`（发布形态）。
