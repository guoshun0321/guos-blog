---
title: "Cron 走读（Deep Dive）"
description: "代码位置：`/root/github/openclaw/src/cron/*`"
pubDate: 2026-02-26T23:00:00Z
---

# Cron 走读（Deep Dive）

> 代码位置：`/root/github/openclaw/src/cron/*`

## 1. Cron 的定位

Cron 子系统负责：

- 定时触发“唤醒/运行”
- 管理 cron jobs 的增删改查、调度、持久化
- 支持 one-shot / periodic / catch-up 等行为（从 test 文件能看出覆盖很广）

## 2. CronService API（`src/cron/service.ts`）

`CronService` 封装了对外 API：

- `start()` / `stop()` / `status()`
- `list()` / `listPage()`
- `add()` / `update()` / `remove()`
- `run(id, mode)`：`mode: due|force`
- `wake({ mode: now|next-heartbeat, text })`

实现细节由 `service/ops.ts` 驱动，状态由 `service/state.ts` 持有。

## 3. 与 Gateway 的关系

Gateway 启动时会：

- `buildGatewayCronService({ cfg, deps, broadcast })`
- 然后在非 minimal test 模式下 `cron.start()`

并且把 cron 相关能力通过 WS handlers 暴露给 CLI/UI（在 attachGatewayWsHandlers 的 context 里能看到 `cron` 与 `cronStorePath` 被注入）。

## 4. 可靠性与行为覆盖（从测试文件观察）

`src/cron/service/*` 下大量 regression 测试表明：

- 系统非常重视：
  - 防重复定时器
  - 重启 catch-up
  - delivered status 持久化
  - 列表/分页行为
  - top-of-hour stagger

