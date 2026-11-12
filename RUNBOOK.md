# RUNBOOK — anthropic-openai-convertion

OpenAI `/v1/chat/completions` 前端 → Anthropic `/v1/messages` 后端的转换 shim。serverless 形制（Cloudflare/Vercel/Netlify）+ 可本地 node/docker 跑。

## 当前状态

| 项 | 值 |
|---|---|
| 分支 / HEAD | `main` @ `c8202fb` — `fix(usage): OpenRouter/OpenAI usage shape` |
| 远程 | `origin` = `gsmainagent/anthropic-openai-convertion`，已推送，无未推提交 |
| CI | GitHub Actions `docker`（push→main）：`c8202fb` success；镜像 `ghcr.io/gsmainagent/anthropic-openai-convertion:latest` |
| 测试 | `npm test` → `node test/smoke.mjs`，44 条断言（含 usage 三桶、流式 `include_usage` 一致性） |

## usage 口径（2026-11-12 定，勿改回）

对齐 OpenAI / **OpenRouter** Chat Completions 语义：

- `prompt_tokens = input_tokens + cache_read_input_tokens + cache_creation_input_tokens`（Anthropic 原生 `input_tokens` **不含**缓存，这里加回来）
- `total_tokens = prompt_tokens + completion_tokens`
- `prompt_tokens_details.cached_tokens` = 缓存读；`.cache_write_tokens` = 缓存写（恒存在，无缓存时为 0）
- 流式：usage 只在客户端带 `stream_options.include_usage` 时于尾部发一条 `choices: []` 的 chunk；累计的是**原始** Anthropic usage 字段（`state.rawUsage`），否则 `message_delta` 会把 `prompt_tokens` 覆盖成 0

字段名依据：OpenAI 官方 OpenAPI spec 的 `CompletionUsage`（`cache_write_tokens`），以及 OpenRouter usage-accounting / prompt-caching 文档实例。

## 请求侧注意

- OpenAI 形状里的 `cache_control` **会被丢掉**（`collectSystem` 把 system 拍平成字符串，`toUserBlocks` 只保留 `type`/`text`）。要触发提示缓存必须走逃生舱 `extra_body.anthropic`，例如把带 `cache_control` 的 system block 数组塞进去（它 verbatim 合并并覆盖 `system`）。
- `stream_options` 由 shim 本地消费，不会转发给 Anthropic（Anthropic 不接受）。

## 环境变量（只列名字，值不落库）

`ANTHROPIC_BASE_URL`（默认 api.anthropic.com，可指订阅型网关）、`ANTHROPIC_MODELS_URL`、`ANTHROPIC_AUTH_STYLE`（x-api-key | bearer | both）、`ANTHROPIC_VERSION`、`DEFAULT_MAX_TOKENS`、`DEFAULT_MODEL`、`ENABLED_MODELS`（`/v1/models` 白名单，支持 glob）。密钥一律经 `Authorization: Bearer` 或 `x-api-key` 入站，不写死。

## 本地调试

`node test/smoke.mjs` 会自起 mock 上游。若在有 env-proxy 的 shell 里跑，务必 `NO_PROXY='*' no_proxy='*'`，否则 Node 的 `fetch()` 打 127.0.0.1 会被塞进出网代理，表现为 `fetch failed`，容易被误判成代码回归。
