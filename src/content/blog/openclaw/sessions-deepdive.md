---
title: "Sessions 走读（Deep Dive）"
description: "代码位置：`/root/github/openclaw/src/sessions/*`"
pubDate: 2026-02-26T17:00:00Z
---

# Sessions 走读（Deep Dive）

> 代码位置：`/root/github/openclaw/src/sessions/*`

## 1. Sessions 的定位

Session 是 OpenClaw 的“对话与状态容器”。它解决的问题包括：

- 同一个人/同一个群/同一个 thread 的消息要落在同一个上下文里
- 不同渠道/不同群聊要隔离（避免串台）
- 对 session 应用策略：模型选择、verbose/thinking、send policy、group activation 等

在 Gateway/Agent 运行时，sessionKey 是串联一切的“主键”。

## 2. Session Key 与类型推断

`src/sessions/session-key-utils.ts` 提供：

- `parseAgentSessionKey(...)`
- `deriveSessionChatType(sessionKey)` → `direct|group|channel|unknown`
- `isCronSessionKey/isSubagentSessionKey/isAcpSessionKey` 等判定
- `resolveThreadParentSessionKey(...)`（推断：thread/session 关系）

这套工具让系统能从 key 本身推断“这是什么类型的会话”。

## 3. Send Policy（是否允许发送）

`src/sessions/send-policy.ts`：

- `normalizeSendPolicy(raw)`
- `deriveSendPolicyDecision({ sessionKey, entry, chatType })` → `allow|deny`

它会：

- 先从 sessionKey 推断 chatType
- 再结合 config 的 SessionEntry（来自 `config/sessions`）做匹配

> 这是重要安全点：即使 agent 生成了回复，也可能因为 send policy 被拒绝投递。

## 4. Overrides：model / verbose / thinking

从文件名可见：

- `model-overrides.ts`：对 session entry 应用模型覆盖
- `level-overrides.ts`：对 verbose/thinking（或类似层级）应用覆盖

这些通常对应：

- /model /think /verbose 等命令对 session 生效
- gateway 的 `sessions.patch` 方法更新 session 运行时状态

## 5. Transcript events

`transcript-events.ts`：

- 用监听器集合发布 transcript 更新事件

这通常用于：

- UI 更新
- 增量写盘/索引
- 触发记忆/检索（推断）

