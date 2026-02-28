---
title: "Nodes（设备节点）走读（Deep Dive）"
description: "代码位置："
pubDate: 2026-02-26T22:00:00Z
---

# Nodes（设备节点）走读（Deep Dive）

> 代码位置：
>
> - Gateway 侧：`/root/github/openclaw/src/gateway/server-methods/nodes.ts`、`/root/github/openclaw/src/gateway/node-registry.ts`、`/root/github/openclaw/src/gateway/server-node-subscriptions.ts` 等
> - Node host（节点执行端）：`/root/github/openclaw/src/node-host/*`

## 1. Nodes 的定位

Nodes 是 OpenClaw 的“远端能力执行体”（通常是 macOS/iOS/Android 设备），可以执行：

- camera snap/clip
- screen record
- canvas（A2UI）
- location.get
- system.run（在 node 上执行命令，受权限与审批控制）
- notifications

Gateway 通过 `node.invoke` 把命令发到 node，node 通过 `node.invoke.result` 回传结果。

## 2. node.invoke 的“协议形态”（从 node-host/invoke.ts 侧观察）

`src/node-host/invoke.ts` 定义了 Node invoke payload：

```ts
export type NodeInvokeRequestPayload = {
  id: string;
  nodeId: string;
  command: string;
  paramsJSON?: string | null;
  timeoutMs?: number | null;
  idempotencyKey?: string | null;
};
```

关键点：

- `command` 是字符串（如 `system.run`、`browser.proxy`、`system.which`）
- 参数以 `paramsJSON` 传递（字符串 JSON）
- 结果通过 GatewayClient 请求 `node.invoke.result` 返回
- Node 也会发 `node.event`（例如 exec.finished）

## 3. Node host 支持的命令（node 端）

从 `handleInvoke(...)` 分支可见，node-host 当前至少支持：

- `system.execApprovals.get` / `system.execApprovals.set`
  - 用于下发/更新 exec approvals 文件（含 socket token，但会 redact）
  - 有 baseHash 防并发写冲突
- `system.which`
  - 在 node 侧找可执行文件路径
- `browser.proxy`
  - 代理浏览器相关命令（具体看 `invoke-browser.ts`）
- `system.run`
  - 执行命令：
    - 有 output cap（200k）与超时 kill
    - 有 env sanitize（阻止 PATH override）
    - 有 exec security/ask（deny/allowlist/full + off/on-miss/always）
    - 支持 macOS app exec host socket 转发（更强隔离/权限）

其他命令会返回 `UNAVAILABLE: command not supported`。

## 4. system.run 的安全设计（node-host 侧）

亮点集中在 `system.run`：

- `sanitizeHostExecEnv({ blockPathOverrides: true })`：限制环境变量注入风险
- `ExecApprovals` 文件：
  - 可配置安全级别（deny/allowlist/full）
  - 可配置 ask 策略（off/on-miss/always）
- 可通过 socket 把执行请求交给 macOS app（`requestExecHostViaSocket`）
  - 环境变量 `OPENCLAW_NODE_EXEC_HOST=app` 可强制

## 5. 与 Gateway 的关系

- Gateway 会维护 `NodeRegistry`（当前在线节点）
- 通过 `node.list/node.describe` 暴露给 CLI/UI
- 通过 `node.invoke` 把调用请求发给 node
- 通过 subscriptions 把 node events 推送到相关 session（`nodeSendToSession/nodeSendToAllSubscribed`）

> 下一步想更深入：建议读 `src/gateway/server-methods/nodes.ts` 与 `src/gateway/node-registry.ts`，把 “WS method → registry → node connection” 的路径画出来。

