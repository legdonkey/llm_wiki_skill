---
source: api-reference.md
source_version: 0
translated_at: 2026-06-25
---

# LLM Wiki API v1 — 接口参考

Base URL: `http://127.0.0.1:19828`
前缀:   `/api/v1`
默认内容类型: `application/json; charset=utf-8`

除 `/health` 外的所有接口都需要通过以下任一方式进行认证:

```
Authorization: Bearer <token>
X-LLM-Wiki-Token: <token>
?token=<urlencoded-token>
```

---

## GET /api/v1/health

**Auth:** 不需要。

```json
{
  "ok": true,
  "status": "running",
  "version": "0.4.x",
  "enabled": true,
  "authRequired": true,
  "authConfigured": true,
  "allowUnauthenticated": false,
  "tokenSource": "store"
}
```

字段说明:

- `status` — `starting` / `running` / `port_conflict` / `error`
- `enabled` — `false` 表示用户已关闭 API；除 `/health` 外的所有接口都返回 503
- `authRequired` — 当且仅当 `allowUnauthenticated: true` 时为 `false`
- `authConfigured` — 当环境变量 `LLM_WIKI_API_TOKEN` 或 `apiConfig.token` 已设置时为 `true`
- `allowUnauthenticated` — 匿名本地进程模式（罕见；仅在用户主动开启时生效）
- `tokenSource` — `env` / `store` / `none`。若为 `env`，则桌面端 UI 中的 token 字段会**被忽略**。

---

## GET /api/v1/projects

**Auth:** 必需。

```json
{
  "ok": true,
  "projects": [
    {
      "id": "a0e90b29-fcf3-4364-9502-8bd1272de820",
      "name": "Research Notes",
      "path": "/Users/me/wiki-projects/research",
      "current": true
    },
    {
      "id": "...",
      "name": "Reading",
      "path": "/Users/me/wiki-projects/reading",
      "current": false
    }
  ]
}
```

`current` 标志标记的是当前在桌面端 UI 中打开的项目。当用户没有明确指定项目时，使用它。

每个项目维度接口中的 `{id}` 占位符接受:

- 项目 UUID（`p.id`）
- 项目文件系统路径（`p.path`，需 URL 编码）
- 字面量字符串 `current`

### 解析用户口述的项目名

该接口上**没有 `?name=` 过滤参数**，且 `{id}` 也**不**直接接受名称——名称完全在客户端列出所有项目后再解析。

算法:

1. `GET /api/v1/projects` → 返回 `{id, name, path, current}` 数组
2. 对 `name` 做不区分大小写的子串匹配:
   ```js
   const matches = projects.filter(p =>
     p.name.toLowerCase().includes(spokenName.toLowerCase())
   )
   ```
3. 处理匹配数量:
   - **0 个匹配** → 告知用户，列出可用名称并询问。不要悄悄回退到 `current`。
   - **1 个匹配** → 在本次对话后续的所有调用中使用 `matches[0].id`。
   - **2 个及以上匹配** → 请用户消歧（为每个匹配展示 `name` + `path`）。
4. 在本次对话剩余过程中缓存已解析的 `id`。仅当用户切换上下文时才重新列出。

如果用户原样给出了文件系统路径，你可以对其进行 URL 编码并直接作为 `{id}` 传入——无需列表查询:

```bash
# macOS / Linux (bash / zsh)
PROJECT_PATH=$(printf %s "/Users/me/wiki/reading" | jq -sRr @uri)
curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE/api/v1/projects/$PROJECT_PATH/files?root=wiki"
```

```powershell
# Windows (PowerShell)
$projectPath = [System.Uri]::EscapeDataString("C:/Users/me/wiki/reading")
curl.exe -s -H "Authorization: Bearer $env:LLM_WIKI_API_TOKEN" `
  "$env:BASE/api/v1/projects/$projectPath/files?root=wiki"
```

**Windows 路径注意事项** —— 要与桌面应用存储时的形式保持一致，否则会得到 404:

- 使用**正斜杠**（`C:/Users/me/wiki`），而非反斜杠。桌面应用在保存前会将路径规范化为正斜杠。
- 保留用户磁盘上实际的大小写（对 API 的字符串比较来说，`C:/Users/Me/...` ≠ `c:/users/me/...`，尽管 Windows 本身不区分大小写）。
- 盘符后的冒号**必须**进行百分号编码（`%3A`）——它是 URI 的保留分隔符。`EscapeDataString` / `jq @uri` / `encodeURIComponent` 都会替你处理这一点。
- 如果你得到 404，回退到 `GET /api/v1/projects`，在那里找到该项目并使用它的 `id`（UUID）—— UUID 与平台无关，永远不需要编码。

如果该路径未在桌面应用中注册，你会得到 **404** —— 回退到列出并询问。

---

## GET /api/v1/projects/{id}/files

**Auth:** 必需。

查询参数:

| 参数 | 默认值 | 说明 |
|---|---|---|
| `root` | `wiki` | 取值之一：`wiki` / `sources`（别名 `raw`、`raw/sources`）/ `all`。`all` 列出所有公开子树（`purpose.md`、`schema.md`、`wiki/`、`raw/sources/`）。 |
| `recursive` | `true` | `false` → 只列一层。 |
| `maxFiles` | `2000` | 限制在 `[1, 10000]`。超出 → 413。 |

响应:

```json
{
  "ok": true,
  "projectId": "...",
  "root": "wiki",
  "files": [
    {
      "name": "concepts",
      "path": "wiki/concepts",
      "isDir": true,
      "size": null,
      "children": [
        {
          "name": "rope.md",
          "path": "wiki/concepts/rope.md",
          "isDir": false,
          "size": 4321,
          "children": null
        }
      ]
    }
  ],
  "truncated": false
}
```

隐藏文件（点文件）和符号链接会被静默跳过。

---

## GET /api/v1/projects/{id}/files/content

**Auth:** 必需。

查询参数:

| 参数 | 是否必填 | 说明 |
|---|---|---|
| `path` | 是 | 项目相对路径。白名单：`purpose.md`、`schema.md`、`wiki/**`、`raw/sources/**`。不允许点文件路径段。不允许 `..`。 |

响应:

```json
{
  "ok": true,
  "projectId": "...",
  "path": "wiki/concepts/rope.md",
  "content": "---\ntitle: RoPE\n---\n\n# Rotary Position Embedding\n..."
}
```

失败情形:

- `403` — 路径在白名单之外（例如 `../app-state.json`、`.llm-wiki/foo.json`）
- `404` — 文件不存在
- `413` — 文件 > 2 MB
- `415` — 文件为二进制 / 非 UTF-8（例如 PNG、PDF）。请使用桌面端 UI 查看；该 API 仅支持文本。

---

## POST /api/v1/projects/{id}/search

**Auth:** 必需。

请求体:

```json
{
  "query": "rope rotary position embedding",
  "topK": 10,
  "includeContent": false,
  "queryEmbedding": null
}
```

| 字段 | 默认值 | 说明 |
|---|---|---|
| `query` | （必填） | 不能为空。空 / 仅空白字符 → 400。 |
| `topK` | `10` | 限制在 `[1, 50]`。 |
| `includeContent` | `false` | 为 `true` 时，每条命中都会附带 `content`（完整 markdown）。可省去逐页拉取内容的往返。 |
| `queryEmbedding` | `null` | 可选 `number[]`。如果你已预先计算好查询向量（用你自己的模型、离线批处理），在此传入，服务端就会跳过自己的 embed 调用。必须是非空的有限数字数组，否则 → 400。 |

响应:

```json
{
  "ok": true,
  "projectId": "...",
  "mode": "hybrid",
  "tokenHits": 78,
  "vectorHits": 14,
  "note": "Search uses the shared backend retrieval service. When embeddingConfig is enabled, the API automatically includes LanceDB vector results; clients may also pass queryEmbedding explicitly.",
  "results": [
    {
      "path": "wiki/concepts/rope.md",
      "title": "Rotary Position Embedding",
      "snippet": "...inject positional information by rotating Q and K...",
      "titleMatch": true,
      "score": 0.0315136476426799,
      "vectorScore": 0.94,
      "images": [
        { "url": "wiki/media/rope-diagram.png", "alt": "RoPE rotation diagram" }
      ],
      "content": null
    }
  ]
}
```

### 检索模式

服务端会根据当前激活项目是否配置了 embeddings（Settings → Embeddings）**以及**该项目的向量索引是否有数据，自动选择模式:

| `mode` | 触发条件 | 分值量级 |
|---|---|---|
| `"keyword"` | 没有 `embeddingConfig`，或 embedding 获取失败，或向量索引为空。 | 累加式关键词分：文件名完全匹配约 200，标题中短语匹配约 50+，词袋评分为个位数。 |
| `"vector"` | 向量索引返回了命中，但关键词评分没有匹配到任何内容。实践中罕见。 | RRF 排名分，通常为 `1 / (60 + rank)` ≈ `0.015–0.017`。 |
| `"hybrid"` | 关键词和向量两条流水线都产生了命中——启用 embeddings 时的常见情况。 | RRF 合并分：对顶部命中最高可达 `1/61 + 1/61` ≈ `0.0328`。 |

`tokenHits` 是关键词阶段评分的页面数量；`vectorHits` 是 LanceDB 返回的不同页面数量。两者都可能为 0。

### 单条结果字段

| 字段 | 是否总是存在？ | 说明 |
|---|---|---|
| `path` | 是 | 指向 markdown 页面的项目相对路径。 |
| `title` | 是 | 若存在则取 front-matter 的 `title:`；否则取第一个 `# Heading`；否则取文件名并把连字符 → 空格。 |
| `snippet` | 是 | 约 160 字符的窗口。关键词模式下：以页面正文中的查询/锚点词为中心。纯向量匹配时：实际匹配的 chunk 文本，可能以该 chunk 的标题路径作为前缀（例如 `"Section > Detail: chunk text..."`）。 |
| `titleMatch` | 是 | 当某个词或短语命中标题时为 `true`（会提升排名）。 |
| `score` | 是 | 最终排名分。量级见“检索模式”。 |
| `vectorScore` | 可选 | 当页面通过向量索引匹配时的原始向量相似度（≈ 余弦 0–1）。可用于判断“语义匹配有多强”。纯关键词命中时不存在。 |
| `images` | 是 | 在 markdown 中发现的内嵌 `![alt](url)` 引用，按 URL 去重。便于 agent 呈现图示。 |
| `content` | 可选 | 完整 markdown，仅当 `includeContent: true` 时存在。 |

`results` 按 `score` 降序排序。不要**跨模式**比较分值（按构造，关键词分会比 RRF 分大 100 倍以上）。请只依赖同一次响应内的相对排序。

---

## GET /api/v1/projects/{id}/graph

**Auth:** 必需。

查询参数:

| 参数 | 默认值 | 说明 |
|---|---|---|
| `q` | — | 对 `id` 或 `label` 的子串过滤，不区分大小写。 |
| `nodeType` | — | 对 frontmatter 的 `type:` 进行过滤（例如 `entity`、`concept`、`query`、`other`）。 |
| `limit` | `200` | 限制在 `[1, 1000]`。 |

响应:

```json
{
  "ok": true,
  "projectId": "...",
  "nodes": [
    {
      "id": "rope",
      "label": "Rotary Position Embedding",
      "nodeType": "concept",
      "path": "wiki/concepts/rope.md",
      "linkCount": 4
    }
  ],
  "edges": [
    {
      "source": "rope",
      "target": "attention",
      "weight": 1.0
    }
  ]
}
```

边由 `wiki/*.md` 内部的 `[[wikilink]]` 引用推导而来。按无序对 `(source, target)` 去重—— b 中的 `[[a]]` 和 a 中的 `[[b]]` 只产生**一条**边。自环会被丢弃。v1 中 `weight` 恒为 `1.0`。

`linkCount` 是该节点在去重后图中的度。

---

## POST /api/v1/projects/{id}/sources/rescan

**Auth:** 必需。**会修改状态。**

无需请求体。

```json
{
  "ok": true,
  "projectId": "...",
  "result": {
    "queue": {
      "version": 1,
      "tasks": []
    },
    "changedTasks": [
      {
        "id": "...",
        "path": "raw/sources/new-paper.pdf",
        "kind": "created"
      }
    ]
  }
}
```

`changedTasks` 包含本次重扫**实际检测为已变更**的文件（created / modified / deleted）。下游摄取队列会异步处理这些文件——该 API 调用在差异入队后即返回，而非在摄取完成后返回。

会遵循用户的 `sourceWatchConfig`（文件类型过滤、排除目录、最大大小）。如果用户禁用了自动摄取，文件仍会被检测到，但不会进入 LLM 流水线的队列。

---

## POST /api/v1/projects/{id}/chat

**返回 501。** 在 v1 中，Chat / RAG 流水线位于 WebView 内。不要调用。请告知用户使用桌面端聊天 UI。

---

## 限制与防护

| 限制 | 取值 | 效果 |
|---|---|---|
| 请求体大小 | 1 MiB | 超出 → 400。 |
| 文件内容读取 | 2 MiB | 超出 → 413。 |
| 文件树节点数 | 10000 硬上限 | 超出 → 413。 |
| 搜索 `topK` | 最大 50 | 静默裁剪。 |
| 图 `limit` | 最大 1000 | 静默裁剪。 |
| 速率限制 | 120 req/sec（全局） | 429，隐含 `Retry-After` 语义（退避 ≥1s）。 |
| 并发请求 | 64 并发 | 503 "API server is busy" —— 退避 ≥2s。 |

CORS: `Access-Control-Allow-Origin: *`。预检通过 `Access-Control-Max-Age: 600` 缓存 10 分钟。允许的请求头包括 `Authorization`、`X-LLM-Wiki-Token`、`Content-Type`。

非 `GET` / 非 `POST` 方法 → 405。
