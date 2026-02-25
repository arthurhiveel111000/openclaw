# Telegram Webhook（挂载到 Gateway HTTP）使用说明

本文档说明本次在 OpenClaw Gateway 内新增的 Telegram webhook 接入方式：**不单独起一个 Telegram webhook HTTP server**，而是把 webhook 处理器挂载到 Gateway 的 HTTP 请求链路里。

默认对外暴露路径：`POST /telegram/webhook`（可用环境变量覆盖）。

## 前置条件

- 你已经在 @BotFather 创建了 bot，拿到 token。
- Gateway 有一个 Telegram 可访问到的公网 HTTPS 地址（或通过反代/隧道暴露）。

## 配置

### 环境变量

- `TELEGRAM_BOT_TOKEN`
  - Telegram bot token。
- `TELEGRAM_WEBHOOK_SECRET`
  - webhook 的 secret token（Telegram 会在请求头携带，用于校验）。
  - **必须设置**，否则 Gateway 不会挂载 webhook。
- `TELEGRAM_WEBHOOK_PATH`
  - Gateway 上 webhook 的 HTTP path。
  - 默认：`/telegram/webhook`

示例（参考 `.env.example`）：

```dotenv
TELEGRAM_BOT_TOKEN=123456:ABCDEF...
TELEGRAM_WEBHOOK_SECRET=change-me-to-a-long-random-string
TELEGRAM_WEBHOOK_PATH=/telegram/webhook
```

## 启动 Gateway 并确认 webhook 已挂载

启动：

```bash
openclaw gateway --port 18789 --verbose
```

日志里出现下面类似内容，表示已经挂载成功：

- `telegram webhook mounted at /telegram/webhook`

如果没有出现：

- 检查是否同时满足 `TELEGRAM_BOT_TOKEN` 和 `TELEGRAM_WEBHOOK_SECRET`。
- 检查是否处于 minimal test gateway（测试环境会跳过挂载）。

## 在 Telegram 侧设置 webhook

你需要把 Telegram webhook URL 指向你的 Gateway 公网地址 + `TELEGRAM_WEBHOOK_PATH`，并设置 `secret_token`。

例如你的公网地址是 `https://example.com`，path 是 `/telegram/webhook`：

```bash
curl -s \
  -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/setWebhook" \
  -d "url=https://example.com${TELEGRAM_WEBHOOK_PATH}" \
  -d "secret_token=${TELEGRAM_WEBHOOK_SECRET}"
```

查询 Telegram 当前 webhook 配置：

```bash
curl -s "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getWebhookInfo"
```

## 验证

- 给 bot 发一条消息（DM 或群里触发）。
- 观察 Gateway 日志是否有 webhook 收到/处理的记录。

## 常见问题排查

- **返回 404**
  - Gateway 没有把该路径注册为 Telegram webhook。
  - 确认 `TELEGRAM_WEBHOOK_PATH` 与 Telegram `setWebhook` 的 url path 完全一致。

- **Telegram 回调失败 / 401 / 403**
  - secret token 不匹配。
  - 确认 `TELEGRAM_WEBHOOK_SECRET` 与 `setWebhook secret_token` 完全一致。

- **一直没有收到 webhook**
  - 确认公网 URL 可达（HTTPS、证书有效）。
  - 确认你的网关端口/反代规则把该 path 转发到了 Gateway。
  - 跟踪日志：

```bash
openclaw logs --follow
```

## 相关代码位置（便于二次开发）

- 挂载到 Gateway 请求链路：`src/gateway/server-http.ts`
- Gateway 启动时注册/关闭时清理：`src/gateway/server.impl.ts`
- Telegram webhook handler（可挂载的 handler）：`src/telegram/webhook.ts`
- Telegram HTTP handler registry：`src/telegram/http/registry.ts`
