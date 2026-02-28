---
title: "Canvas / A2UI 走读（Deep Dive）"
description: "代码位置："
pubDate: 2026-02-27
---

# Canvas / A2UI 走读（Deep Dive）

> 代码位置：
>
> - Canvas host：`/root/github/openclaw/src/canvas-host/*`
> - Gateway HTTP/upgrade 对 canvas 的集成：`/root/github/openclaw/src/gateway/server-http.ts`

## 1. Canvas 的定位

Canvas 是 OpenClaw 的“可视化/交互表面”：

- Gateway 提供 canvas host（静态资源 + websocket live reload）
- Node 端（iOS/Android/macOS app）用 webview 呈现 canvas
- 通过 A2UI（Agent-to-UI）把 UI 组件树推送到 canvas
- 用户在 canvas 上的交互（actions）回传给 agent/gateway

## 2. Canvas host 的能力（`src/canvas-host/server.ts`）

Canvas host 可以以两种方式提供：

- `createCanvasHostHandler(...)`：创建 handler（可被 Gateway 复用/嵌入）
- `startCanvasHost(...)`：独立启动一个 http server 承载 canvas

### 2.1 rootDir 与默认页面

- 默认 rootDir：`<stateDir>/canvas`
- 若没有 `index.html`，会自动写入一个默认页面（内含 demo buttons 与 action bridge 检测）

### 2.2 live reload

- 默认开启 live reload
- 通过 chokidar watch rootDir，变更后向 websocket clients 广播 `reload`

### 2.3 路由

- `CANVAS_HOST_PATH`（basePath 下静态文件）
- `CANVAS_WS_PATH`（live reload websocket）
- `handleA2uiHttpRequest`（A2UI 相关路由）

## 3. Gateway 对 Canvas 的鉴权集成（`src/gateway/server-http.ts`）

Gateway 在处理 HTTP 请求时会识别 canvas path：

- A2UI、canvas host 静态资源、canvas ws

并对 canvas path 进行 `authorizeCanvasRequest(...)`：

- 本地直连请求直接通过
- bearer token（走 gateway auth）通过
- 或者 **canvas capability**：只要存在一个已授权的 node ws client 持有该 capability（并在 TTL 内）即可

这允许 node webview 通过“能力票据”访问 canvas，而不需要 gateway token。

## 4. 建议的进一步走读点

1) `src/canvas-host/a2ui.ts`
- A2UI 的 HTTP contract 与 push/reset 行为

2) `src/gateway/canvas-capability.ts`
- capability 的生成/TTL/绑定逻辑

