---
title: "Memory 走读（Deep Dive）"
description: "代码位置：`/root/github/openclaw/src/memory/*`"
pubDate: 2026-02-27
---

# Memory 走读（Deep Dive）

> 代码位置：`/root/github/openclaw/src/memory/*`

## 1. Memory 的定位

Memory 子系统负责“长期记忆/检索”，典型包括：

- embedding 生成与批处理
- 向量索引的写入/去重/更新
- 查询：相似度检索、MMR、query expansion、temporal decay
- 可能支持本地/远程 provider（从 `embeddings-remote-*`、`remote-http.ts` 可见）

OpenClaw 的 memory 不只是“把内容塞进向量库”，而是一个较完整的检索工程模块。

## 2. 核心对外入口（`src/memory/index.ts`）

导出：

- `MemoryIndexManager`
- `getMemorySearchManager()`（返回 search manager 或失败原因）

核心类型：

- `MemorySearchManager` / `MemorySearchResult`
- embedding probe result 等

## 3. 组件划分（从文件名观察）

### embeddings

- `embeddings.ts`：embedding provider 统一入口/封装
- `embeddings-openai.ts / embeddings-gemini.ts / embeddings-voyage.ts / embeddings-mistral.ts`
- `embedding-*limits*.ts`：输入/chunk 限制
- `manager-embedding-ops.ts` / `manager-sync-ops.ts`：embedding 写入/同步操作

### search

- `manager-search.ts` / `search-manager.ts`
- `mmr.ts`（最大边际相关性）
- `query-expansion.ts`
- `temporal-decay.ts`

### storage

- `sqlite.ts` / `sqlite-vec.ts`
- `fs-utils.ts`

### batch

- `batch-*.ts`：批处理 embedding / 上传

## 4. 与插件系统的关系（重要）

从 `src/plugins/loader.ts` 可见：

- 存在 **memory slot** 机制
- `record.kind === "memory"` 的插件会参与 slot 仲裁

这意味着 memory backend/实现可能不止一套，且可能由插件选择（比如 `extensions/memory-*`）。

