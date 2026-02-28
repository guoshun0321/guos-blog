---
title: "Tools（工具系统）走读（Deep Dive）"
description: "代码位置分散在："
pubDate: 2026-02-26T19:00:00Z
---

# Tools（工具系统）走读（Deep Dive）

> 代码位置分散在：
>
> - Gateway methods：`/root/github/openclaw/src/gateway/server-methods/*`（tools.catalog、exec approvals、browser、nodes、send 等）
> - Agent tools：`/root/github/openclaw/src/agents/*`（bash-tools、process、channel-tools 等）
> - Browser/Canvas/Node-host：各自目录

## 1. Tools 的定位

OpenClaw 的“工具”不是单一模块，而是一套贯穿 Gateway+Agent 的机制：

- Agent 在模型输出中产生 tool call
- Gateway/Agent runtime 执行 tool（可能在 host、sandbox、node、browser、canvas 等执行域）
- 返回结果给模型继续推理
- 同时把 tool 事件广播给 UI/客户端（可观察性）

## 2. 从 Gateway 的方法表看 Tools 的“控制面”入口

`src/gateway/server-methods-list.ts` 里明确有：

- `tools.catalog`：工具目录/清单
- `exec.approvals.*`：exec 审批/策略
- `node.invoke`：节点工具入口
- `browser.request`：浏览器工具入口
- `send`：出站发送

此外，HTTP 侧还有：

- `handleToolsInvokeHttpRequest`（`server-http.ts`）
  - 这是一个“通过 HTTP 调用工具”的面向集成/回归的入口（常用于 cron/hook/外部系统）

## 3. 执行域（Execution Host）的多样性

从本仓库结构与 node-host 的实现可以看出，OpenClaw 工具有多个执行域：

- **host**：Gateway 所在机器直接执行（最常见）
- **sandbox**：非 main session 可在 docker sandbox 里执行（安全隔离）
- **node**：设备节点执行（camera/screen/system.run）
- **browser**：受控浏览器进程执行（Playwright/CDP）
- **canvas**：渲染/交互由 canvas host 与 node webview 承载

## 4. 安全控制是工具系统的一等公民

可以看到多处安全控制：

- Gateway method scopes/roles（`server-methods.ts`）
- control-plane 写操作 rate limit（config.apply/config.patch/update.run）
- exec approvals（host/node 都有）
- HTTP endpoint 的 gateway auth（Control UI、channels api 等）
- canvas capability（为 node webview 提供最小权限访问）

## 5. 下一步建议（想真正“走读清楚工具系统”）

若要把工具系统做成“可操作的心智模型”，建议继续补充两块：

1. `src/gateway/server-methods/tools-catalog.ts`
   - tools.catalog 输出的结构、是否含动态工具、权限说明
2. `src/agents/*bash-tools*` 与 `src/agents/tools/*`
   - 模型 tool call 是怎么被解析/执行/回填

