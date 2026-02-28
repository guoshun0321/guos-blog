---
title: "Gateway 协议与方法表走读（WS/HTTP）"
description: "代码位置："
pubDate: 2026-02-27
---

# Gateway 协议与方法表走读（WS/HTTP）

> 代码位置：
>
> - WS runtime：`/root/github/openclaw/src/gateway/server-ws-runtime.ts`、`/root/github/openclaw/src/gateway/server/ws-connection.ts`
> - 方法分发：`/root/github/openclaw/src/gateway/server-methods.ts`
> - 方法清单：`/root/github/openclaw/src/gateway/server-methods-list.ts`
> - HTTP server：`/root/github/openclaw/src/gateway/server-http.ts`

本文重点：**Gateway 对外暴露了哪些方法/事件？鉴权/授权怎么做？WS 与 HTTP 如何共存？**

## 1. Gateway Methods（方法表）= “控制平面 API”

`server-methods-list.ts` 定义了 `BASE_METHODS`，包含核心控制面能力：

- health / status
- logs.tail
- config.get/set/apply/patch/schema
- tts.* / voicewake.*
- sessions.*
- skills.* / agents.*
- cron.*
- node.* / device.*
- exec approvals
- agent / agent.wait / chat.history/chat.send/chat.abort
- browser.request

并且：

- `listGatewayMethods()` 会把 `listChannelPlugins()` 的 `plugin.gatewayMethods` 合并进来

因此：

> **Gateway 的方法表 = Core methods + Channel plugins methods（动态注入）**。

## 2. 核心 handler 分发：`handleGatewayRequest`

`src/gateway/server-methods.ts` 提供：

- `coreGatewayHandlers`：把各个模块 handlers 合并（agent/sessions/nodes/config/...）
- `handleGatewayRequest({ req, respond, client, context, extraHandlers })`

关键逻辑：

1) **authorizeGatewayMethod**：

- 按 client.connect.role（默认 operator）解析 role
- `isRoleAuthorizedForMethod(role, method)`：role-policy
- `authorizeOperatorScopesForMethod(method, scopes)`：method-scopes
- `ADMIN_SCOPE` 可 bypass
- role === node 直接放行（node 作为第一等公民，但能力边界由 handler 自己再控制）

2) **control-plane write rate limit**：

- 对 `config.apply/config.patch/update.run` 这类写操作做“3 per 60s”的速率限制

3) 查找 handler：

- 优先 `extraHandlers[req.method]`（插件/扩展注入）
- 否则 `coreGatewayHandlers[req.method]`

4) 执行 handler

> 结论：Gateway 的方法分发是一个“**可扩展的路由器**”，允许插件覆盖或增加方法实现。

## 3. WS runtime：attach handlers 到连接层

`src/gateway/server-ws-runtime.ts` 本身很薄：

- `attachGatewayWsHandlers(...)` 只是把参数转交给 `attachGatewayWsConnectionHandler(...)`
- `attachGatewayWsConnectionHandler` 才是 WS 握手、鉴权、连接生命周期、请求帧处理的核心

> 建议后续深入：直接读 `src/gateway/server/ws-connection.ts` 与 `src/gateway/protocol/*`。

## 4. Gateway Events（广播事件）

`server-methods-list.ts` 也定义了 `GATEWAY_EVENTS`：

- connect.challenge
- agent/chat/presence/tick/shutdown/health/heartbeat/cron
- node pairing / invoke
- device pairing
- voicewake.changed
- exec approval requested/resolved
- update.available

这说明：

- WS 连接不仅是 request/response，也承担 **事件总线** 的作用
- Gateway 内部大量子系统会 `broadcast(event, payload, { dropIfSlow })`

## 5. HTTP server：Control UI + API endpoints + hooks + canvas

`src/gateway/server-http.ts` 创建 HTTP/HTTPS server，并在 `handleRequest` 里按顺序路由：

1) 设置默认安全 header（HSTS 可选）
2) 跳过 upgrade=websocket 请求（由 upgrade handler 处理）
3) `handleHooksRequest`（gateway hooks：/hooks/...）
4) `handleToolsInvokeHttpRequest`（工具 HTTP invoke）
5) `handleSlackHttpRequest`（Slack 特殊 HTTP）
6) `handlePluginRequest`（插件 HTTP 路由；channels 路由默认 gateway-auth 保护）
7) OpenResponses endpoint（可选）
8) OpenAI chat/completions endpoint（可选）
9) Canvas host/A2UI（需要 canvas capability 或 gateway auth / local direct）
10) Control UI 静态资源 + avatar

### 5.1 Canvas 的鉴权（亮点）

Canvas 的授权是“双通道”的：

- 本地直连（loopback/可信代理）可直接通过
- 或者 bearer token（走 authorizeHttpGatewayConnect）
- 或者 **canvas capability**：
  - capability 来自 scoped URL
  - 只有当存在“已授权的 node WS client”持有相同 capability 且未过期才允许
  - 并且 capability 采用 sliding TTL（持续使用会延长）

这让 Canvas 能“只对配对节点可用”，而不必把 gateway token 暴露给 webview。

### 5.2 Upgrade handler（WebSocket 共存）

`attachGatewayUpgradeHandler` 在 httpServer 的 `upgrade` 事件里：

- 优先让 canvasHost 处理 `CANVAS_WS_PATH`
- 否则 upgrade 到 gateway 的 wss

## 6. 建议的进一步走读路线（更细）

要把 Gateway 协议彻底吃透，下一步建议：

1. `src/gateway/server/ws-connection.ts`：WS handshake / connect params / auth challenge
2. `src/gateway/protocol/*`：消息帧格式、错误码、client info
3. `src/gateway/server-methods/*`：逐个方法的语义与实现（尤其 node.invoke、tools、agent、chat）
4. `src/gateway/auth.ts`：HTTP/WS 的 auth 细节一致性
