---
title: "OpenClaw 项目技术架构分析"
description: "| 层次 | 技术 |"
pubDate: 2026-02-26T10:00:00Z
---

# OpenClaw 项目技术架构分析

## 一、技术栈

### 语言与运行时

| 层次 | 技术 |
|------|------|
| 后端核心 | TypeScript 5.9.3（ESM strict mode），Node.js ≥22.12.0 |
| 脚本执行 | Bun（首选 TS 执行器），tsx 4.21.0 |
| macOS / iOS | Swift 5.10+，SwiftUI + UIKit |
| Android | Kotlin，Jetpack Compose，JVM 17 |

### 后端框架与核心库

| 分类 | 库 | 版本 | 用途 |
|------|-----|------|------|
| HTTP | Express | 5.2.1 | Control UI、OpenAI 兼容端点 |
| WebSocket | ws | 8.19.0 | Gateway 主通信协议 |
| CLI | Commander.js | 14.0.3 | CLI 应用框架 |
| Schema | TypeBox | 0.34.48 | 类型安全 JSON Schema（协议定义）|
| 验证 | Zod | 4.3.6 | 运行时数据验证 |
| 配置格式 | JSON5 | 2.2.3 | `~/.openclaw/openclaw.json` |
| 任务调度 | croner | 10.0.1 | Cron 自动化 |
| 服务发现 | @homebridge/ciao | 1.3.5 | mDNS/Bonjour LAN 发现 |

### AI Agent

| 库 | 版本 | 用途 |
|-----|------|------|
| @mariozechner/pi-agent-core | 0.55.0 | Pi Agent 推理引擎 |
| @mariozechner/pi-ai | 0.55.0 | 通用对话 Agent |
| @mariozechner/pi-coding-agent | 0.55.0 | 代码生成专用 Agent |
| @mariozechner/pi-tui | 0.55.0 | TUI 交互 |
| @agentclientprotocol/sdk | 0.14.1 | Agent Client Protocol |
| @aws-sdk/client-bedrock | ^3.997.0 | AWS Bedrock 模型接入 |

### 消息通道（内置）

| 平台 | 库 |
|------|----|
| Telegram | grammy 1.40.0，@grammyjs/runner |
| Discord | @discordjs/voice 0.19.0，discord-api-types |
| Slack | @slack/bolt 4.6.0，@slack/web-api |
| WhatsApp | @whiskeysockets/baileys 7.0.0-rc.9 |
| LINE | @line/bot-sdk 10.6.0 |
| Feishu/Lark | @larksuiteoapi/node-sdk 1.59.0 |
| iMessage | BlueBubbles 客户端（HTTP 集成）|
| Signal | signal-cli（外部二进制）|

消息通道扩展（`extensions/` 下独立包）：msteams、matrix、zalo、zalouser、googlechat、mattermost、irc、twitch、synology-chat、tlon、nextcloud-talk 等。

### 媒体处理

| 库 | 用途 |
|-----|------|
| sharp 0.34.5 | 图像缩放/格式转换 |
| pdfjs-dist 5.4.624 | PDF 渲染与文本提取 |
| ffmpeg（外部二进制）| 音视频转码 |
| playwright-core 1.58.2 | 浏览器自动化 |
| node-edge-tts 1.2.10 | 文本转语音 |
| @mozilla/readability | 网页正文提取 |
| @napi-rs/canvas | 图形渲染（可选对等依赖）|

### 数据存储

| 技术 | 用途 |
|------|------|
| 文件系统（JSON5）| 主配置 `~/.openclaw/openclaw.json` |
| JSONL 文件 | 会话历史 `~/.openclaw/agents/<id>/sessions/*.jsonl` |
| sqlite-vec 0.1.7-alpha.2 | SQLite 向量扩展，用于 RAG 记忆 |
| LanceDB（memory-lancedb 扩展）| 向量数据库，记忆检索 |

### 前端（Web UI）

| 技术 | 版本 | 用途 |
|------|------|------|
| Lit | 3.3.2 | Web Components 框架 |
| @lit/context | 1.1.6 | 依赖注入 |
| @lit-labs/signals | 0.2.0 | 响应式状态 |
| Vite | 7.3.1 | 构建工具 |
| DOMPurify | 3.3.1 | XSS 防护 |
| marked | 17.0.3 | Markdown 渲染 |
| @noble/ed25519 | 3.0.0 | Ed25519 签名（设备认证）|

### 构建与工具链

| 工具 | 版本 | 用途 |
|------|------|------|
| pnpm | 10.23.0 | Monorepo 包管理 |
| tsdown | 0.20.3 | TypeScript 打包 |
| oxfmt | 0.35.0 | 代码格式化（Rust 实现）|
| oxlint | 1.50.0 | 类型感知 Lint（Rust 实现）|
| vitest | 4.0.18 | 单元 & 集成测试 |
| SwiftFormat / SwiftLint | — | Swift 代码质量 |

---

## 二、整体部署图

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  外部消息平台（云端）                                                  │
  │  Telegram API │ Discord API │ Slack API │ WhatsApp Web │ Signal CLI  │
  │  iMessage/BlueBubbles │ LINE │ Feishu │ Teams │ Matrix │ Zalo …      │
  └────────────────────────────┬─────────────────────────────────────────┘
                               │  各平台协议（HTTPS / WebSocket / CLI）
                               │
  ┌────────────────────────────▼─────────────────────────────────────────┐
  │                     用户本地设备（默认）                               │
  │                                                                       │
  │  ┌─────────────────────────────────────────────────────────────────┐ │
  │  │              Gateway 控制平面                                    │ │
  │  │        ws://127.0.0.1:18789  /  http://127.0.0.1:18789          │ │
  │  │                                                                  │ │
  │  │  ┌─────────────┐  ┌───────────────┐  ┌──────────────────────┐  │ │
  │  │  │ WS Server   │  │ HTTP Server   │  │ 消息通道驱动层        │  │ │
  │  │  │ (客户端/节  │  │ Control UI    │  │ src/channels/        │  │ │
  │  │  │  点连接)    │  │ /v1/chat/...  │  │ extensions/*/        │  │ │
  │  │  └──────┬──────┘  └───────────────┘  └──────────┬───────────┘  │ │
  │  │         │                                        │               │ │
  │  │  ┌──────▼──────────────────────────────────────▼───────────┐   │ │
  │  │  │                    路由与会话管理                         │   │ │
  │  │  │   src/routing/  ←→  src/gateway/  ←→  src/channels/dock │   │ │
  │  │  │   binding: channel/account/peer → agentId + sessionKey   │   │ │
  │  │  └─────────────────────────────┬────────────────────────────┘   │ │
  │  │                                │                                  │ │
  │  │  ┌─────────────────────────────▼────────────────────────────┐   │ │
  │  │  │               Pi Agent（RPC 模式）                        │   │ │
  │  │  │   pi-agent-core / pi-ai / pi-coding-agent                │   │ │
  │  │  │   工具：bash、file I/O、browser、system-run …            │   │ │
  │  │  └─────────────────────────────┬────────────────────────────┘   │ │
  │  │                                │ HTTPS                           │ │
  │  │  ┌─────────────────────────────▼────────────────────────────┐   │ │
  │  │  │              LLM 模型提供商（云端 API）                   │   │ │
  │  │  │  Anthropic Claude │ OpenAI │ Google Gemini │ OpenRouter   │   │ │
  │  │  │  AWS Bedrock │ VolcEngine │ Qianfan │ LM Studio（本地）  │   │ │
  │  │  └──────────────────────────────────────────────────────────┘   │ │
  │  │                                                                  │ │
  │  │  ┌──────────────────────────────────────────────────────────┐   │ │
  │  │  │              本地持久化                                   │   │ │
  │  │  │  ~/.openclaw/openclaw.json（配置）                       │   │ │
  │  │  │  ~/.openclaw/agents/<id>/sessions/*.jsonl（会话历史）    │   │ │
  │  │  │  ~/.openclaw/credentials/（凭证）                        │   │ │
  │  │  │  SQLite + sqlite-vec / LanceDB（RAG 向量记忆）           │   │ │
  │  │  └──────────────────────────────────────────────────────────┘   │ │
  │  └─────────────────────────────────────────────────────────────────┘ │
  │                                                                       │
  │  ┌──────────────────────────────────────────────────────────────┐    │
  │  │            Gateway 客户端 / 节点（均通过 WS 连接 Gateway）    │    │
  │  │                                                                │    │
  │  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐  │    │
  │  │  │  CLI 工具    │  │ macOS 菜单栏 │  │    Web UI (Lit)    │  │    │
  │  │  │  openclaw    │  │  App (Swift) │  │  Control UI        │  │    │
  │  │  │  gateway/    │  │  Voice Wake  │  │  WebChat           │  │    │
  │  │  │  agent/send  │  │  Canvas 显示 │  │  Canvas (A2UI)     │  │    │
  │  │  └──────────────┘  └──────────────┘  └────────────────────┘  │    │
  │  │                                                                │    │
  │  │  ┌──────────────┐  ┌──────────────┐                           │    │
  │  │  │  iOS Node    │  │ Android Node │                           │    │
  │  │  │  (Swift)     │  │ (Kotlin)     │                           │    │
  │  │  │  摄像头/位置 │  │ 屏幕/Talk    │                           │    │
  │  │  │  屏幕/联系人 │  │ Mode/摄像头  │                           │    │
  │  │  └──────────────┘  └──────────────┘                           │    │
  │  └──────────────────────────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────────────────┘
```

---

## 三、模块详解与访问方式

### 3.1 Gateway 对外暴露的接口

**监听地址**（由 `gateway.bind` 配置控制）：

| 绑定模式 | 实际地址 | 适用场景 |
|---------|---------|---------|
| `loopback`（默认）| `127.0.0.1:18789` | 仅本机访问，最安全 |
| `lan` | `0.0.0.0:18789` | 局域网内其他设备访问 |
| `tailnet` | Tailscale IP:18789 | 通过 Tailscale 远程访问 |
| `auto` | 优先 loopback，否则 LAN | 自动检测 |

**HTTP 端点**（同一端口）：

| 路径 | 方法 | 说明 |
|------|------|------|
| `/` | GET | Control UI 主页（SPA） |
| `/__openclaw__/avatar/<agentId>` | GET | Agent 头像 |
| `/__openclaw__/canvas/` | GET | Canvas A2UI 宿主页 |
| `/v1/chat/completions` | POST | OpenAI 兼容聊天接口（需开启）|
| `/v1/responses` | POST | OpenResponses API（需开启）|

**WebSocket 协议**（同一端口，HTTP Upgrade）：

握手帧（客户端首帧）：
```json
{
  "type": "connect",
  "minProtocol": 1,
  "maxProtocol": 1,
  "client": { "id": "...", "version": "...", "platform": "macos", "mode": "client" },
  "auth": { "token": "..." },
  "role": "client"
}
```

服务端响应：
```json
{
  "type": "hello-ok",
  "protocol": 1,
  "server": { "version": "2026.2.24", "connId": "..." },
  "features": { "methods": [...], "events": [...] },
  "snapshot": { "presence": {}, "health": {} }
}
```

后续通信帧格式：
```
Request:  { type:"req", id:"<uuid>", method:"agent", params:{...} }
Response: { type:"res", id:"<uuid>", ok:true, payload:{...} }
Event:    { type:"event", event:"chat", payload:{...}, seq:42 }
```

代表性 RPC 方法：`health`、`status`、`agent`、`send`、`config.get`、`channels.status`、`presence`、`cron.schedule`、`logs.tail`、`exec`

代表性服务端推送事件：`agent`（流式回复）、`chat`（消息）、`presence`、`health`、`heartbeat`、`tick`、`shutdown`、`push`

### 3.2 各客户端如何连接 Gateway

**CLI**（`pnpm openclaw ...`）
- 直接进程内调用 Gateway，或通过 WebSocket 连接本地 Gateway
- `pnpm openclaw gateway:watch` 启动 Gateway 并监听文件变化

**macOS App**（Swift，`apps/macos/`）
- 通过 `OpenClawKit`（共享框架）中的 `GatewayNodeSession`
- 连接 `ws://127.0.0.1:18789`，role = `node`
- 能力标记：`canvas.render`、`voice.wake`、`camera.capture` 等
- 通过菜单栏 UI 提供 Gateway 启停控制

**iOS Node**（Swift，`apps/ios/`）
- 通过 `OpenClawKit` 共享库，相同 WebSocket 协议
- 提供设备能力：摄像头、屏幕录制、位置、日历、联系人
- 通过 Tailscale 可远程连接非本机 Gateway

**Android Node**（Kotlin，`apps/android/`）
- 通过 Ktor WebSocket 客户端 + kotlinx.serialization
- 提供能力：Talk Mode 语音覆盖、屏幕录制、摄像头、SMS（可选）

**Web UI**（Lit，`ui/`）
- 浏览器中通过原生 WebSocket API 连接
- `GatewayBrowserClient` 封装连接、重连、事件订阅逻辑
- 提供 Control UI（配置面板）、WebChat（聊天界面）、Canvas（A2UI 可视化工作区）

**Canvas A2UI**
- Agent 通过 WebSocket 推送 A2UI 协议帧，Gateway 转发给渲染节点
- 渲染节点（macOS/iOS/Web）在沙箱 iframe 中执行 HTML/CSS/JS

### 3.3 消息通道如何接入 Gateway

消息通道驱动（内置于 `src/channels/`，扩展于 `extensions/*/`）通过 Gateway 内部事件总线与核心通信，**不**经过 WebSocket，而是直接进程内调用：

```
外部消息平台
    │  长轮询 / Webhook / WebSocket（各平台协议）
    ▼
通道驱动（Telegram/Discord/Slack…）
    │  内部事件 emit
    ▼
src/channels/dock.ts（消息路由）
    │  根据 binding 规则解析 sessionKey
    ▼
src/routing/resolve-route.ts
    │  确定目标 agentId
    ▼
Pi Agent（RPC 调用）
    │  生成回复
    ▼
通道驱动发送回复 → 外部平台
```

### 3.4 插件/扩展系统

- 扩展以独立 `package.json` workspace 存在于 `extensions/*/`
- 运行时通过 `jiti` 动态加载，路径别名 `openclaw/plugin-sdk` 在运行时解析
- 扩展中 `openclaw` 只能放 `devDependencies` 或 `peerDependencies`，runtime 依赖放 `dependencies`
- 禁止在扩展 `dependencies` 中使用 `workspace:*`（npm install 会报错）

### 3.5 认证机制

| 方式 | 配置 | 适用场景 |
|------|------|---------|
| 无认证（`auth.mode: none`）| 默认 loopback 下 | 本机单用户 |
| Token 认证 | `OPENCLAW_GATEWAY_TOKEN` | 远程连接 |
| 密码认证 | `OPENCLAW_GATEWAY_PASSWORD` | 备选 |
| 设备配对 | Ed25519 公钥签名 nonce | 所有节点首次连接 |

设备配对流程：Gateway 发送 nonce → 客户端用私钥签名 → Gateway 验证公钥 → 颁发短期 `deviceToken`。本地 loopback 连接可配置为自动批准。

---

## 四、数据流汇总

```
用户在 Telegram 发送消息
  → Telegram API 推送 Update 到 grammy 长轮询
  → src/channels/telegram/ 接收，封装为内部 ChatMessage
  → src/channels/dock.ts 路由，查找 binding，得到 agentId + sessionKey
  → src/routing/resolve-route.ts 确认会话
  → Pi Agent (RPC) 调用，注入工具（bash、file、browser…）
  → Pi Agent 调用 Claude/OpenAI API（HTTPS，带 API Key）
  → 模型返回 token 流
  → Pi Agent 执行工具调用（如需）
  → 最终回复写回 src/channels/telegram/
  → grammy 发送消息到 Telegram API
  → 用户收到回复

  同时：
  → 会话历史追加到 ~/.openclaw/agents/<id>/sessions/*.jsonl
  → 若启用记忆：embedding 存入 SQLite/LanceDB
  → WS 客户端（Control UI / macOS App）收到 "agent" 事件推送（实时流）
```
