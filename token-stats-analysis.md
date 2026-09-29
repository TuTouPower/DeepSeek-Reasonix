# Reasonix Token 统计全链路分析

## 一、来源 — API 响应

DeepSeek API 返回的 `usage` 对象包含四个字段，在 `src/client.ts` 的 `Usage.fromApi()` 中解析：

| API 字段 | 映射到 | 含义 |
|---|---|---|
| `prompt_tokens` | `promptTokens` | 输入 token |
| `completion_tokens` | `completionTokens` | 输出 token |
| `prompt_cache_hit_tokens` | `promptCacheHitTokens` | 缓存命中 token |
| `prompt_cache_miss_tokens` | `promptCacheMissTokens` | 缓存未命中 token |

如果 API 没返回 `miss` 字段，会反推：`max(0, promptTokens - cacheHitTokens)`。

---

## 二、流式管道 — 最后一个带 usage 的分块胜出

流式调用 (`DeepSeekClient.stream()`) 中，每个 SSE 帧都解析。只有**最后一个**带 `usage` 的 chunk 会被 `streamModelResponse()` (`src/loop/streaming.ts:73`) 捕获并返回。

非流式 (`chat()`) 直接从 `data.usage` 提取，路径一致。

---

## 三、缓存诊断 — 自研前缀哈希推断 miss 原因

这是 Reasonix 自己做的，不依赖 API。核心在：

- **`cacheShapeForRequest()`** (`src/loop.ts:618`) — 对每次请求计算 systemHash、toolsHash、fewShotsHash 三个前缀哈希
- **`cacheDiagnosticsForUsage()`** (`src/loop.ts:628`) — 与上一轮比较，生成 `prefixChangeReasons`，能识别 7 种原因：

| 原因 | 含义 |
|---|---|
| `no-miss` | 无缓存未命中 |
| `cold-start` | 首轮请求 |
| `system-prompt-changed` | 系统提示变化 |
| `tool-list-changed` | 工具列表变化 |
| `tool-schema-or-order-changed` | 工具 schema/顺序变化 |
| `mcp-tool-hot-add` | MCP 工具热添加 |
| `memory-or-skill-changed` | 记忆/技能变化 |
| `unknown` | 其他 |

命中率通过 `hit / (hit + miss)` 计算。

---

## 四、消费端 — 四条并行通道

每次助理轮次完成后，`CacheFirstLoop.step()` (`src/loop.ts:1010`) 调用 `this.stats.record()` 将 Usage 分发到：

| 通道 | 文件 | 内容 |
|---|---|---|
| **用量 JSONL** | `src/telemetry/usage.ts` | `UsageRecord`：promptTokens、completionTokens、cacheHitTokens、cacheMissTokens、costUsd。写 `~/.reasonix/usage.jsonl` |
| **会话统计** | `src/telemetry/stats.ts` | `SessionStats` 累加每轮 `TurnStats`（含 cacheHitRatio、cacheDiagnostics），上限 200 轮后折叠 |
| **Transcript JSONL** | `src/transcript/log.ts` | `TranscriptRecord` 含 usage、cost、model、prefixHash、cacheDiagnostic |
| **内核事件日志** | `src/core/eventize.ts` | `ModelFinalEvent` 携带 usage + costUsd，写 `*.events.jsonl` |

---

## 五、成本计算

`costUsd()` (`src/telemetry/stats.ts:9-18`)：

```
成本 = (cacheHitTokens × 缓存命中价 + cacheMissTokens × 缓存未命中价 + completionTokens × 输出价) / 1_000_000
```

| 模型 | 缓存命中 | 缓存未命中 | 输出 |
|---|---|---|---|
| flash | 2.8¢/M | 14¢/M | 28¢/M |
| pro | 3.625¢/M | 43.5¢/M | 87¢/M |

还提供 `cacheSavingsUsd()` 计算缓存节省了多少费用。

---

## 六、关键文件索引

| 文件 | 职责 |
|---|---|
| `src/client.ts:50-60` | `Usage.fromApi()` — API 响应 → 结构化 Usage |
| `src/client.ts:397-403` | 流式 SSE 帧解析 + usage 提取 |
| `src/loop.ts:618-647` | 前缀哈希计算 + 缓存诊断 |
| `src/loop.ts:1010-1015` | 每轮 stats.record() 摄入点 |
| `src/telemetry/stats.ts` | SessionStats 聚合 + 成本计算 |
| `src/telemetry/usage.ts` | UsageRecord 写入 usage.jsonl |
| `src/telemetry/cache-diagnostics.ts` | 缓存诊断逻辑 |
| `src/transcript/log.ts` | Transcript 写入 |
| `src/core/eventize.ts` | 事件日志写入 |
