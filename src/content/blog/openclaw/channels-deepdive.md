---
title: "Channels 走读（Deep Dive）"
description: "代码位置："
pubDate: 2026-02-27
---

# Channels 走读（Deep Dive）

> 代码位置：
>
> - 核心：`/root/github/openclaw/src/channels/*`
> - 渠道插件注册表：`/root/github/openclaw/src/channels/plugins/*`
> - 具体渠道实现多在：`/root/github/openclaw/extensions/*`（如 telegram/discord/slack/whatsapp 等）

## 1. Channels 的定位

Channels 是 OpenClaw 与外部消息平台之间的适配层：

- 把外部平台的 **入站消息** 转成 OpenClaw 的统一事件/消息格式
- 把 OpenClaw 的 **出站消息**（文本/媒体/动作）发回对应平台
- 提供渠道特有能力：
  - typing indicator
  - reactions / threads
  - 目录（directory peers/groups）
  - 登录/设备配对（某些渠道）

## 2. Channel Plugins：从插件系统注册渠道

`src/channels/plugins/index.ts` 说明了一个关键事实：

- **渠道插件不是静态写死**，而是由插件 loader 把“channel 插件”注册到 active plugin registry 里。

关键 API：

- `listChannelPlugins(): ChannelPlugin[]`
  - 从 `requireActivePluginRegistry().channels` 取出插件
  - 去重（按 id）
  - 按 `CHAT_CHANNEL_ORDER` + meta.order 排序
- `getChannelPlugin(id): ChannelPlugin | undefined`
- `normalizeChannelId(raw): ChannelId | null`（输入归一化委托给 `src/channels/registry.ts`）

### 2.1 渠道插件加载为何放在 plugins 系统里？

因为渠道常常“很重”：

- 需要 web login、长连接、SDK 依赖
- 可能有后台监控/健康检查

所以 `src/channels/plugins/index.ts` 特别注释：

> 共享路径（reply flow、command auth、sandbox explain）应依赖 `src/channels/dock.ts`，避免在热路径里加载重插件。

这体现了一个设计原则：

- **轻逻辑（路由/规则/格式）在 core**
- **重适配器（真正对接外部渠道）在 plugins/extensions**

## 3. 核心能力（从目录观察）

`src/channels/` 下能看到这些核心模块（摘取一些最有代表性的）：

- allowlist/allow-from：安全与允许来源匹配
- mention-gating / command-gating：群聊 mention 才响应、命令权限
- sender identity / labels：把不同平台的 sender 统一成可用于 session key 的身份
- reply-prefix：不同平台的回复前缀策略
- dock：渠道对接/出站适配（注释中强烈提示它是“共享路径”的依赖）
- plugins/*：渠道插件框架、目录配置、outbound adapters

## 4. 与 Gateway 的关系（重要）

Gateway 启动时会：

- 通过 `createChannelManager(...)` 创建 channel manager
- 通过 channel plugins 提供的 `gatewayMethods` 把渠道相关 RPC 方法并入 gateway methods
- 启动 channels（由 `startGatewaySidecars(...)` 内部调用 `startChannels` 等）

并且 Gateway 有 `startChannelHealthMonitor(...)` 定期检查渠道健康。

