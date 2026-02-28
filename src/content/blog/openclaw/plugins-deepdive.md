---
title: "Plugins 走读（Deep Dive）"
description: "代码位置：`/root/github/openclaw/src/plugins/*`"
pubDate: 2026-02-26T24:00:00Z
---

# Plugins 走读（Deep Dive）

> 代码位置：`/root/github/openclaw/src/plugins/*`
>
> 目标：解释 OpenClaw 如何发现/加载插件，插件能提供哪些能力（gateway handlers、channels、tools、hooks、CLI commands、services），以及安全/可控性设计。

## 1. 插件在 OpenClaw 里的角色

插件是 OpenClaw 的“扩展单元”，可以向系统注入：

- **Gateway handlers**（WS/RPC 方法实现或增强）
- **Gateway methods 列表**（告诉控制平面有哪些方法可调用）
- **Channels**（Telegram/Discord/Slack/WhatsApp… 这种渠道插件）
- **Tools**（新增工具给 agent 调用）
- **Hooks**（gateway_start/gateway_stop、before/after tool call、LLM hooks 等）
- **CLI commands**（在 CLI 上注册额外子命令）
- **HTTP handlers**（拓展 HTTP 路由）
- **Services**（插件内部运行的后台服务句柄）

## 2. 插件加载入口：`loadOpenClawPlugins`（`src/plugins/loader.ts`）

核心函数：

- `loadOpenClawPlugins(options)`

它做的事情可以概括为：

1. 读取并规范化 plugins 配置（enabled / allow / loadPaths / entries / slots）
2. 发现插件候选（workspace + extraPaths）
3. 加载 manifest registry（解析每个插件的元信息与 config schema）
4. 安全与可控性检查（allowlist、provenance、path safety）
5. 用 `jiti` 加载 TS/JS 插件入口
6. 执行插件 `register/activate`（同步注册）
7. 汇总到 `PluginRegistry` 并缓存
8. 初始化全局 hook runner（`initializeGlobalHookRunner`）

### 2.1 发现：`discoverOpenClawPlugins`

`discoverOpenClawPlugins({ workspaceDir, extraPaths })` 会返回 candidates 列表。

候选来源通常包括：

- bundled 插件（随发行版/仓库自带）
- workspace / extensions 下的插件
- 配置的额外 loadPaths

### 2.2 Manifest registry：`loadPluginManifestRegistry`

manifest 记录会提供：

- 插件 id / name / description / version
- rootDir / source / origin（bundled/local 等）
- **config schema**（非常关键）

在 loader 中：如果插件缺失 config schema，会被标为 error 并拒绝加载。

### 2.3 allowlist 与 provenance（重要安全点）

loader 里有两个非常“安全工程化”的设计：

- **plugins.allow**：如果 allow 为空且发现 non-bundled plugins，会打印警告
- **provenance tracking**：检查插件是否能被 config 的 installs/loadPaths 追踪
  - 如果是“未追踪的本地代码”，会记为 warn diagnostic，并提醒你 pin trust

### 2.4 path safety

loader 会检查：

- 插件 entry path 必须在插件 rootDir 内（防止路径逃逸）

### 2.5 使用 jiti 加载（支持 TS）

loader 用 `createJiti(import.meta.url, ...)` 来加载插件，扩展名支持很全：

- `.ts/.tsx/.mts/.cts/...` + `.js/.mjs/.cjs` + `.json`

并且做了 alias：

- `openclaw/plugin-sdk` → 指向本仓库的 `src/plugin-sdk` 或 `dist/plugin-sdk`
- `openclaw/plugin-sdk/account-id` 同理

这允许插件代码用稳定的 SDK import 路径。

### 2.6 插件导出契约（register/activate）

loader 用 `resolvePluginModuleExport` 兼容多种导出方式：

- default export 是 function → 当作 register
- default export 是 object → 读 `register` 或 `activate`

> 注意：如果 `register` 返回 Promise，会被警告：**async registration is ignored**。

## 3. PluginRegistry（运行时注册表）

loader 最终构建 `PluginRegistry`，并通过：

- `setActivePluginRegistry(registry, cacheKey)`

把它设为“当前活跃插件注册表”。随后 `initializeGlobalHookRunner(registry)` 初始化全局 hooks。

> Channel 插件的查找也依赖这个 active registry（见 `src/channels/plugins/index.ts`）。

## 4. memory slot 选择（插件槽位）

loader 里有“slots.memory”的逻辑：

- 多个 memory 类型插件时，只能选一个作为当前 memory backend
- `resolveMemorySlotDecision(...)` 负责仲裁

这是一种“插件槽位”机制：避免同类插件并存时冲突。

## 5. 与 Gateway 的关系

Gateway 在启动时会：

- 调用 `loadGatewayPlugins(...)`（位于 `src/gateway/server-plugins.ts`）
- 该函数内部会使用插件系统加载 registry
- 插件提供的 `gatewayHandlers` 会在 `attachGatewayWsHandlers(...)` 时被合并进 `extraHandlers`

也就是说：

- **Gateway 的 RPC 能力 = core handlers + plugin handlers + channel plugin methods**。

