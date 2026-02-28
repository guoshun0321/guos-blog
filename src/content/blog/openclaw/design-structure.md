---
title: "2. 项目设计结构（Design Structure）"
description: "本节回答：代码如何分层、模块边界是什么、核心抽象是什么。"
pubDate: 2026-02-27
---

# 2. 项目设计结构（Design Structure）

本节回答：代码如何分层、模块边界是什么、核心抽象是什么。

> 说明：这里基于目录结构与少量关键入口文件（`src/entry.ts`、`src/cli/run-main.ts` 等）推断高层设计。更细节的“类/接口”级别设计，需要再深入阅读 `src/gateway/*`、`src/agents/*`、`src/sessions/*`、`src/channels/*`、`src/plugins/*` 关键实现文件。

## 2.1 单仓（Monorepo）与分区

顶层分区（重要目录）：

- `src/`：OpenClaw 核心（Gateway、CLI、agent runtime、session、tools、channels 等）
- `extensions/`：大量渠道/能力扩展（也可能包含 auth portal、memory provider 等）
- `skills/`：技能包（OpenClaw skill 平台的一部分，发布时会带上）
- `ui/`：Web/UI 侧（独立 package）
- `packages/`：额外包（如 `clawdbot`、`moltbot`）
- `apps/`：iOS/Android/macOS 以及 shared 代码
- `docs/`：项目文档站点内容

## 2.2 典型分层（建议心智模型）

可以用“控制平面 + 适配器 + 运行时”的视角理解：

1. **控制平面（Gateway）**
   - 负责：会话管理、工具调用、鉴权/策略、路由、统一协议（WS/RPC）
2. **适配器层（Channels / Nodes / Browser / Canvas 等）**
   - 将外部世界（消息渠道、设备能力、浏览器）抽象成统一事件/命令
3. **Agent 运行时（Agents / Sessions / Memory）**
   - 处理输入消息 → 调用模型 → 输出消息/动作
   - 通过 tools 执行外部动作
4. **CLI/UI（操作面）**
   - CLI 是主要控制入口；Web 控制台/WebChat 是可选操作面

### 设计分层图

```mermaid
flowchart TB
  subgraph ControlPlane[控制平面]
    GW[Gateway WS/RPC]
    ROUTE[Routing/Policy]
    CONF[Config/Secrets]
    SESS[Session Store]
  end

  subgraph Runtime[Agent 运行时]
    AG[Agents]
    MEM[Memory]
    TOOLS[Tool Registry]
    MODELS[Model Providers + Failover]
  end

  subgraph Adapters[适配器/集成]
    CH[Channels\n(WhatsApp/Telegram/Slack/...)]
    NODES[Device Nodes\n(camera/screen/canvas/...)]
    BROWSER[Browser Control]
    CANVAS[Canvas Host + A2UI]
    PLUGINS[Plugins/Extensions]
  end

  subgraph Surfaces[操作面]
    CLI[CLI]
    WEB[Web UI/WebChat]
    APPS[iOS/Android/macOS Apps]
  end

  Surfaces --> ControlPlane
  Adapters --> ControlPlane
  ControlPlane <--> Runtime
  Runtime --> Adapters
```

## 2.3 插件/扩展思想

从目录 `src/plugins/`、`src/plugin-sdk/`、`extensions/*`、以及 `package.json` 的 `exports`（`./plugin-sdk`）可以推断：

- 核心提供 **Plugin SDK**（供扩展编写/编译时类型）
- 扩展通常位于 `extensions/*`，一些是渠道（discord/telegram/...），一些是能力模块（memory-xxx、thread-ownership、copilot-proxy 等）
- CLI 在解析命令时会“注册插件 CLI commands”（见 `src/cli/run-main.ts`：`registerPluginCliCommands(program, loadConfig())`）

> 这对应一种设计：核心保持稳定 API + 协议，扩展通过统一接口挂载（运行时加载、CLI 注册、配置驱动启用）。

## 2.4 安全与隔离（设计要点）

从 README 可以看到 OpenClaw 的安全设计倾向：

- **DM pairing / allowlist**：未知用户默认走配对流程
- **非 main 会话可 sandbox**：通过 Docker sandbox 隔离执行工具（bash/browser/nodes 等）
- **工具级权限**：不同 surface/session 允许的工具集不同

这些通常会体现在：

- `src/security/`、`src/sandbox/`（或相关目录）
- `src/pairing/`（配对/批准）
- `src/sessions/`（会话策略、group activation、sendPolicy 等）
