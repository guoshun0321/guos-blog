---
title: "Gateway 走读（Deep Dive）"
description: "代码位置：`/root/github/openclaw/src/gateway/*`"
pubDate: 2026-02-27
---

# Gateway 走读（Deep Dive）

> 代码位置：`/root/github/openclaw/src/gateway/*`
>
> 本文基于静态阅读与目录结构梳理，目标是帮你快速建立“Gateway 是如何启动、如何组织子系统、如何对外提供 WS/HTTP 控制面”的心智模型。

## 1. Gateway 的定位

Gateway 是 OpenClaw 的**常驻控制平面**（control plane），统一承载：

- WebSocket/RPC 控制面（CLI/Web 控制台/Nodes/工具调用都通过它）
- HTTP endpoints（Control UI + 可选 OpenAI ChatCompletions/OpenResponses 兼容 API）
- sessions / agent 运行 / 工具调度 / 渠道接入 / 插件扩展
- 安全（auth、rate limit、origin、pairing/policy）、运维（health、reload、discovery、tailscale exposure）

核心入口函数：

- `src/gateway/server.impl.ts` → `startGatewayServer(port, opts)`

CLI 启动入口：

- `src/cli/gateway-cli/run.ts` → `runGatewayLoop(... startGatewayServer ...)`

## 2. 从 CLI 到 Gateway：启动与安全护栏

### 2.1 CLI 参数解析（`src/cli/gateway-cli/run.ts`）

该命令处理：

- `--port`：端口（默认 18789 或 config）
- `--bind`：`loopback|lan|tailnet|auto|custom`
- `--auth`/`--token`/`--password`：鉴权模式与密钥
- `--tailscale`：`off|serve|funnel`
- `--force`：必要时杀掉占用端口的进程
- `--dev/--reset`：开发模式便捷配置

关键护栏：

- 当 `bind != loopback` 且没有共享密钥（token/password）时，会**拒绝启动**（除非 `trusted-proxy`）。
- 当 gateway 配置缺失或 `gateway.mode != local` 时，会要求修复配置或显式 `--allow-unconfigured`。

### 2.2 进入 run loop

`runGatewayLoop` 负责：

- 抢占/锁定端口（避免多实例）
- 捕获/打印更友好的启动错误（如端口占用、service 已运行等）

## 3. `startGatewayServer` 的启动编排（`src/gateway/server.impl.ts`）

`startGatewayServer` 可以理解为一个“**组装器**”：它把配置、插件、WS/HTTP server、channels、nodes、cron、sidecars 等拼成一个可运行的 Gateway，并返回一个 `{ close() }` 句柄。

### 3.1 启动阶段总览

```mermaid
flowchart TD
  A[readConfigFileSnapshot + migrateLegacy] --> B[applyPluginAutoEnable]
  B --> C[ensureGatewayStartupAuth]
  C --> D[loadGatewayPlugins + channel methods]
  D --> E[resolveGatewayRuntimeConfig]
  E --> F[createGatewayRuntimeState]
  F --> G[init nodes/cron/channels/discovery]
  G --> H[attachGatewayWsHandlers]
  H --> I[start sidecars + config reloader]
  I --> J[return GatewayServer.close()]
```

### 3.2 配置与启动前校验

- 读取 config snapshot：`readConfigFileSnapshot()`
- legacy config 自动迁移：`migrateLegacyConfig()` + `writeConfigFile()`
- 自动启用 plugins：`applyPluginAutoEnable()`
- 确保 auth 就绪：`ensureGatewayStartupAuth({ persist: true })`
  - token 缺失时可能会生成并持久化到 `gateway.auth.token`

### 3.3 插件与方法表

- `listGatewayMethods()`：得到基础方法清单
- `loadGatewayPlugins(...)`：加载 plugins，返回
  - `pluginRegistry`（含 `gatewayHandlers` 等）
  - `baseGatewayMethods`
- `listChannelPlugins()`：收集 channel plugins
  - 额外把 `plugin.gatewayMethods` 合并到 gateway methods

> 结果：Gateway 的 RPC 方法清单不是写死的，而是**核心 + 插件 + channel 插件**合成。

### 3.4 运行时配置解析

`resolveGatewayRuntimeConfig(...)` 决定：

- `bindHost`（loopback/lan/tailnet/auto/custom 的最终落地地址）
- `controlUiEnabled`、`controlUiBasePath`、`controlUiRoot`（Control UI 静态资源根）
- HTTP endpoints 是否开启：
  - `openAiChatCompletionsEnabled` (`POST /v1/chat/completions`)
  - `openResponsesEnabled` (`POST /v1/responses`)
- `resolvedAuth`、`tailscaleMode` 等

并可选启用 auth rate limit（若 config 明确配置了 `gateway.auth.rateLimit`）。

### 3.5 构建 Gateway Runtime State

`createGatewayRuntimeState(...)` 返回大量“核心运行时对象”，典型包括：

- HTTP server(s) + WSS
- `clients`（连接的 client 集合）
- `broadcast` / `broadcastToConnIds`
- `chatRunState` / `chatAbortControllers`（chat/agent run 的状态机与中断）
- `dedupe`（去重）
- `toolEventRecipients`（工具事件订阅方）
- `canvasHost`（若启用）

可以把它看成：Gateway 的“内核容器”。

### 3.6 子系统装配：nodes / cron / channels / discovery

- Nodes：
  - `NodeRegistry` 管理节点连接
  - `createNodeSubscriptionManager()` 管理 session 对 node events 的订阅
- Cron：`buildGatewayCronService({ cfg, deps, broadcast })`
- Channels：`createChannelManager({ loadConfig, channelLogs, channelRuntimeEnvs })`
- Discovery：`startGatewayDiscovery(...)`（mDNS + wide-area 可选）
- Maintenance timers：`startGatewayMaintenanceTimers(...)`（presence/health/cleanup）
- Heartbeat：`startHeartbeatRunner({ cfg })`
- Channel health monitor：`startChannelHealthMonitor(...)`
- Delivery recovery：恢复崩溃前未完成的 outbound deliveries

### 3.7 attach WS handlers（关键步骤）

`attachGatewayWsHandlers({ ... })` 把 Gateway 变成可用的 WS/RPC 服务。

它接收：

- `gatewayMethods`：允许的方法集合
- `events`：Gateway 广播事件类型集合
- `extraHandlers`：插件 handlers + exec approval handlers
- `context`：一个“全家桶”上下文（cron、channels、nodes、health、wizard、dedupe、chat-run 状态等）

> 这一步可以理解为：**把“方法表 + handler 实现 + 上下文依赖”绑定到 WSS 上**。

### 3.8 Sidecars、热重载与关闭

- Sidecars：`startGatewaySidecars(...)`
  - 启动 channels
  - 可能启动 browser control server
  - 初始化 plugin services
- 配置热重载：`startGatewayConfigReloader(...)`
  - 可 hot reload 的直接 apply
  - 不可的触发 restart 请求
- 关闭：`createGatewayCloseHandler(...)` 统一 stop
  - 先跑 `gateway_stop` hooks
  - stop discovery/tailscale/canvas/channels/cron/heartbeat/timers/ws/http...

## 4. Gateway 的“宏观运行时结构图”

```mermaid
flowchart TB
  subgraph CP[Control Plane]
    WS[WS Server (wss)\nattachGatewayWsHandlers]
    HTTP[HTTP Server(s)\nControl UI + API endpoints]
    AUTH[Auth + Rate limit + Origin checks]
  end

  subgraph CORE[Runtime Core]
    STATE[createGatewayRuntimeState\nclients/broadcast/chatRun/dedupe]
    METHODS[coreGatewayHandlers\n+ pluginRegistry.gatewayHandlers\n+ channel methods]
  end

  subgraph SUBSYS[Subsystems]
    CH[ChannelManager]
    NODES[NodeRegistry + Subscriptions]
    CRON[Cron Service]
    DISC[Discovery]
    TS[Tailscale Exposure]
    BROW[Browser control]
    CANVAS[Canvas host]
    RELOAD[Config reloader]
  end

  WS --> METHODS
  METHODS --> STATE
  STATE --> CH
  STATE --> NODES
  STATE --> CRON
  STATE --> BROW
  STATE --> CANVAS
  STATE --> RELOAD
  STATE --> DISC
  STATE --> TS
  HTTP --> METHODS
  AUTH --> WS
  AUTH --> HTTP
```

## 5. 建议的进一步走读点（下一步）

如果你要继续深入 Gateway 的“具体行为”，建议优先看：

1. `src/gateway/server-ws-runtime.ts`：WS 层的连接、鉴权、dispatch、日志
2. `src/gateway/server-methods.ts` + `src/gateway/server-methods/*`：RPC 方法实现
3. `src/gateway/server-http.ts`：HTTP endpoints（Control UI + OpenAI/OpenResponses）
4. `src/gateway/server-plugins.ts`：插件如何加载并注入 handlers/methods
5. `src/gateway/server-runtime-config.ts`：bind/auth/tailscale/controlUi 的解析策略
