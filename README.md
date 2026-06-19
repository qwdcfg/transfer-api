# unlimited.surf Transfer API Worker

中文 | [English](#english)

这是一个 Cloudflare Worker 中转适配器，用于把 `https://unlimited.surf` 转换成 OpenAI 兼容的 `/v1/*` 接口，以及 Anthropic/Claude Code 兼容的 `/v1/messages` 和 `/anthropic/*` 接口。

## 功能概览

- OpenAI 兼容：`/v1/chat/completions`、`/v1/responses`、`/v1/models`、`/v1/files`。
- Anthropic 兼容：`/v1/messages`、`/v1/models`、`/anthropic/v1/messages`、`/anthropic/v1/models`。
- unlimited.surf 原始接口代理：`/api/*` 会直接转发到上游。
- Web Search：映射到上游 `POST /api/search`。
- Merge AI：映射到上游 `POST /api/merge`。
- Files：上传/提取映射到上游 `POST /api/attachments/extract`。
- Agent Setup、Codex、MCP：提供说明和配置入口，分别是 `/v1/setup`、`/v1/codex`、`/v1/mcp`。

注意：MCP server 仍然需要在本地 agent、IDE 或 Claude Code/Codex 环境里运行。这个 Worker 只提供模型 API 端点，不会在 Cloudflare 边缘侧读取或修改你的本地文件。

## 通过 GitHub 关联 Cloudflare 自动部署

可以把本项目推送到 GitHub 私有仓库，然后让 Cloudflare 在每次提交后自动部署。

### 1. 推送到 GitHub

创建 GitHub 仓库并推送本项目。不要把 unlimited.surf API key 提交进仓库。

### 2. 在 Cloudflare 连接 GitHub 仓库

1. 打开 Cloudflare Dashboard。
2. 进入 `Workers & Pages`。
3. 点击 `Create`。
4. 选择 `Import a repository` 或 `Connect to Git`。
5. 按提示授权 Cloudflare 访问 GitHub。
6. 选择这个项目所在的 GitHub 仓库。
7. Root directory 填 `/`，除非你把项目放在仓库子目录里。

### 3. 配置构建和部署参数

Cloudflare 部署表单里可以使用：

```text
Framework preset: None
Build command: npm install
Deploy command: npx wrangler deploy
Root directory: /
Wrangler config: wrangler.toml
```

如果 Cloudflare 只提供一个命令输入框，可以填：

```bash
npm install && npx wrangler deploy
```

### 4. 添加 Cloudflare Secret

在 Worker 设置里添加 Secret：

```text
UNLIMITED_SURF_API_KEY=<你的 unlimited.surf key>
```

通常位置是：

```text
Workers & Pages -> 你的 Worker -> Settings -> Variables -> Secrets
```

请只把 key 放到 Secret 里，不要写进 `wrangler.toml`、`README.md` 或任何 GitHub 文件。

### 5. 部署后验证

部署完成后打开：

```text
https://<your-worker>.workers.dev/health
```

看到 JSON 里有 `"ok": true` 即表示 Worker 正常运行。

测试 OpenAI 模型列表：

```bash
curl https://<your-worker>.workers.dev/v1/models
```

测试 Anthropic/Agent 配置说明：

```bash
curl https://<your-worker>.workers.dev/v1/setup
```

## 本地 Wrangler 手动部署

也可以在本地直接部署：

```powershell
npm install -g wrangler
wrangler login
wrangler secret put UNLIMITED_SURF_API_KEY
wrangler deploy
```

`wrangler secret put` 时输入你的 unlimited.surf key。也可以不配置 Secret，而是在每次请求里传：`Authorization: Bearer <key>` 或 `x-api-key: <key>`。

## OpenAI 兼容接口

Base URL：

```text
https://<your-worker>.workers.dev/v1
```

支持接口：

- `GET /v1/models`
- `POST /v1/chat/completions`
- `POST /v1/responses`
- `POST /v1/search`
- `POST /v1/merge`
- `GET /v1/key`、`GET /v1/usage`
- `POST /v1/files`
- `POST /v1/files/extract`、`POST /v1/attachments/extract`
- `GET /v1/setup`、`GET /v1/codex`、`GET /v1/mcp`

示例：

```bash
curl https://<your-worker>.workers.dev/v1/chat/completions \
  -H "Authorization: Bearer <key>" \
  -H "Content-Type: application/json" \
  -d '{"model":"gateway-gpt-5","messages":[{"role":"user","content":"Hello"}],"stream":true}'
```

## Anthropic / Claude Code 兼容接口

Base URL：

```text
https://<your-worker>.workers.dev
```

支持接口：

- `POST /v1/messages`
- `GET /v1/models`
- `POST /anthropic/v1/messages`
- `GET /anthropic/v1/models`
- `GET /v1/setup`、`GET /v1/codex`、`GET /v1/mcp`

Claude Code PowerShell 示例：

```powershell
$env:ANTHROPIC_BASE_URL = "https://<your-worker>.workers.dev"
$env:ANTHROPIC_AUTH_TOKEN = "<key>"
$env:ANTHROPIC_API_KEY = "<key>"
$env:ANTHROPIC_MODEL = "claude-opus-4-7-20260101"
claude
```

## 功能映射

- Chat 映射到上游 `POST /api/chat`。
- Web Search 在调用 `/v1/search`、传入 `web_search_options`、传入 `query`，或包含 web search tool 时映射到 `POST /api/search`。
- Merge AI 在调用 `/v1/merge`、传入 `merge: true`，或传入 2 个以上 `models` 时映射到 `POST /api/merge`。
- Models 映射到上游 `GET /api/models`，如果上游不可用会返回内置 fallback 模型列表。
- Files 映射到上游 `POST /api/attachments/extract`。如果需要持久化文件，需要额外接入 Cloudflare KV 或 R2。
- Codex、Agent Setup、MCP 是配置说明入口；MCP 工具执行仍然发生在本地 agent/IDE 里。
- Embeddings、audio、images 会返回 `501`，因为当前提供的 unlimited.surf 文档中没有这些原生接口。

## 原始上游代理

任何 `/api/*` 请求都会被转发到 unlimited.surf，并自动带上配置好的 key，因此你仍然可以通过 Worker 使用原始接口。

---

## English

This is a Cloudflare Worker adapter for `https://unlimited.surf`. It exposes OpenAI-compatible `/v1/*` routes and Anthropic/Claude Code-compatible `/v1/messages` plus `/anthropic/*` aliases.

## Features

- OpenAI-compatible routes: `/v1/chat/completions`, `/v1/responses`, `/v1/models`, `/v1/files`.
- Anthropic-compatible routes: `/v1/messages`, `/v1/models`, `/anthropic/v1/messages`, `/anthropic/v1/models`.
- Raw upstream proxy: `/api/*` forwards directly to unlimited.surf.
- Web Search maps to upstream `POST /api/search`.
- Merge AI maps to upstream `POST /api/merge`.
- Files extraction maps to upstream `POST /api/attachments/extract`.
- Agent Setup, Codex, and MCP info endpoints are available at `/v1/setup`, `/v1/codex`, and `/v1/mcp`.

MCP servers still run inside your local agent, IDE, Claude Code, or Codex environment. This Worker only provides the model API endpoint and does not read or modify local files from Cloudflare.

## Deploy from GitHub with Cloudflare

Push this project to a GitHub repository, then let Cloudflare deploy it automatically on each commit.

### 1. Push to GitHub

Create a GitHub repository and push this project. Do not commit your unlimited.surf API key.

### 2. Connect the repository in Cloudflare

1. Open the Cloudflare Dashboard.
2. Go to `Workers & Pages`.
3. Click `Create`.
4. Choose `Import a repository` or `Connect to Git`.
5. Authorize Cloudflare to access GitHub if prompted.
6. Select the repository containing this project.
7. Set Root directory to `/` unless the project is in a subdirectory.

### 3. Build and deploy settings

Use these values:

```text
Framework preset: None
Build command: npm install
Deploy command: npx wrangler deploy
Root directory: /
Wrangler config: wrangler.toml
```

If Cloudflare shows only one command field, use:

```bash
npm install && npx wrangler deploy
```

### 4. Add the Cloudflare Secret

Add this secret in the Worker settings:

```text
UNLIMITED_SURF_API_KEY=<your unlimited.surf key>
```

The usual location is:

```text
Workers & Pages -> your Worker -> Settings -> Variables -> Secrets
```

Keep the key in Secrets only. Do not put it in `wrangler.toml`, `README.md`, or GitHub files.

### 5. Verify the deployment

Open:

```text
https://<your-worker>.workers.dev/health
```

You should see JSON with `"ok": true`.

Test OpenAI-compatible models:

```bash
curl https://<your-worker>.workers.dev/v1/models
```

Test Anthropic/agent setup docs:

```bash
curl https://<your-worker>.workers.dev/v1/setup
```

## Manual deploy with Wrangler

You can also deploy from your local machine:

```powershell
npm install -g wrangler
wrangler login
wrangler secret put UNLIMITED_SURF_API_KEY
wrangler deploy
```

Enter your unlimited.surf key when `wrangler secret put` prompts for it. You can also skip the secret and pass a key per request with `Authorization: Bearer <key>` or `x-api-key: <key>`.

## OpenAI-compatible routes

Base URL:

```text
https://<your-worker>.workers.dev/v1
```

Supported routes:

- `GET /v1/models`
- `POST /v1/chat/completions`
- `POST /v1/responses`
- `POST /v1/search`
- `POST /v1/merge`
- `GET /v1/key`, `GET /v1/usage`
- `POST /v1/files`
- `POST /v1/files/extract`, `POST /v1/attachments/extract`
- `GET /v1/setup`, `GET /v1/codex`, `GET /v1/mcp`

Example:

```bash
curl https://<your-worker>.workers.dev/v1/chat/completions \
  -H "Authorization: Bearer <key>" \
  -H "Content-Type: application/json" \
  -d '{"model":"gateway-gpt-5","messages":[{"role":"user","content":"Hello"}],"stream":true}'
```

## Anthropic / Claude Code-compatible routes

Base URL:

```text
https://<your-worker>.workers.dev
```

Supported routes:

- `POST /v1/messages`
- `GET /v1/models`
- `POST /anthropic/v1/messages`
- `GET /anthropic/v1/models`
- `GET /v1/setup`, `GET /v1/codex`, `GET /v1/mcp`

Claude Code PowerShell example:

```powershell
$env:ANTHROPIC_BASE_URL = "https://<your-worker>.workers.dev"
$env:ANTHROPIC_AUTH_TOKEN = "<key>"
$env:ANTHROPIC_API_KEY = "<key>"
$env:ANTHROPIC_MODEL = "claude-opus-4-7-20260101"
claude
```

## Feature mapping

- Chat maps to upstream `POST /api/chat`.
- Web Search maps to upstream `POST /api/search` when you call `/v1/search`, pass `web_search_options`, pass `query`, or include a web search tool.
- Merge AI maps to upstream `POST /api/merge` when you call `/v1/merge`, pass `merge: true`, or pass `models` with 2+ model IDs.
- Models maps to upstream `GET /api/models` with a fallback catalog if the upstream call fails.
- Files maps upload/extract requests to upstream `POST /api/attachments/extract`; persistent file storage requires adding KV or R2.
- Codex, Agent Setup, and MCP are setup/info endpoints. MCP tool execution remains local to the client agent or IDE.
- Embeddings, audio, and images return `501` because the provided unlimited.surf docs do not expose those native APIs.

## Raw upstream proxy

Any `/api/*` request is forwarded to unlimited.surf with the configured key, so the original API remains available through the Worker.
