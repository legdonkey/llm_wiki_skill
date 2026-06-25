---
source: SKILL.md
source_version: 0
translated_at: 2026-06-25
---

# LLM Wiki 本地 API 技能

## 描述

查询用户的 LLM Wiki 知识库（位于 127.0.0.1:19828 的 LLM Wiki 桌面应用——**不是** Obsidian、Notion、Apple Notes、Logseq 或任何其他 PKM 工具）。**仅当**用户明确提到 LLM Wiki、说"我的 wiki"、"我的 知识库 / 知识库 / knowledge base"，或问类似"我的 wiki 里关于 X 是怎么说的"、"读一下 wiki 页面 Y"、"显示我的 wiki 图谱 / 知识图谱"、"在我的 LLM Wiki 项目里搜索"、"重新扫描我的 wiki 源 / 重新索引"，或按 ID 指名某个 wiki 项目时才触发。**不要**在通用的"搜索我的笔记"、"在我的 notebook 里找"、"看看我的 Obsidian"等情况下触发——那些属于用户可能安装的其他工具。本技能覆盖针对正在运行的 LLM Wiki 桌面应用的 wiki 页面搜索、文件列表、内容读取、知识图谱导航和源重扫描。除源重扫描外均为只读。

通过 LLM Wiki 应用内置的 HTTP API，与用户本地运行的该应用通信。这是一套**标准 JSON API**——直接用你环境里已有的任意 HTTP 工具调用它（`curl`、`fetch`、`requests`、`http` 中间件等）。无需安装客户端库，也无需学习 SDK。

把这个 wiki 当作用户一直在维护的**私有、结构化知识库**：页面以 `wiki/**.md` 形式存在，原始文档放在 `raw/sources/` 下，wikilinks（双链）构成一张图谱。

## 何时触发

**仅当**用户明显是在专门指 **LLM Wiki** 时才触发——通过应用名、通过 `wiki` 的说法，或通过 `知识库` 的说法。具体而言：

- 以"我的 **wiki** / 我的 **knowledge base** / 我的**知识库** / **LLM Wiki** 里关于 X 是怎么说的"形式提问
- 要求"在 **我的 wiki** / **LLM Wiki** 项目 / 我的**知识库** 里搜索 X"
- 按文件名主干（stem）或标题引用某个 **wiki 页面**，想要阅读或交叉链接
- 索要 **wiki 图谱 / 知识图谱 / wiki 概览 / wiki 结构**
- 刚在 LLM Wiki **源文件夹**下添加或编辑了文件，想重新运行摄取 / **重新索引**
- 说"用 **我的 wiki** 作为上下文" / "把答案落到 **我的 wiki** 上" / "查一下 **我的 LLM Wiki**"
- 指名某个 wiki 项目（按 ID、按绝对路径，或按 `current`）

**不要在用户说以下内容时触发：**

- "搜索 **我的笔记**"且没有进一步限定——很可能是 Obsidian / Apple Notes / Notion / Logseq / Bear 等
- "在 **我的 notebook** 里找"——很可能是 Jupyter / OneNote / Notability
- "看看 **我的 Obsidian / Notion / Roam / Logseq 库**"——明确是另一个工具
- "查一下 **我的 Anki / Readwise / Pocket**"——不同工具
- "搜索 **我的文件 / 我的 Documents 文件夹**"——通用文件系统，不是 wiki
- 世界通用知识、时事，或任何用户明显想从公开网络获取的内容

当不确定用户指的是哪个知识工具时，要问：*"你是特指你的 LLM Wiki，还是另一个工具？"*——不要在可能是 Obsidian 库的对象上悄悄调用 LLM Wiki API。

## 快速开始

整套 API 都是纯 HTTP + JSON。最快的路径：

```bash
BASE=http://127.0.0.1:19828
TOKEN="${LLM_WIKI_API_TOKEN:-<paste-from-Settings>}"

# 1. probe state — no auth needed
curl -s $BASE/api/v1/health

# 2. list projects
curl -s -H "Authorization: Bearer $TOKEN" $BASE/api/v1/projects

# 3. search
curl -s -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"query":"rope embedding","topK":5}' \
  $BASE/api/v1/projects/current/search

# 4. read a page
curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE/api/v1/projects/current/files/content?path=wiki/concepts/rope.md"
```

如果你在写 TypeScript / JavaScript：

```ts
const res = await fetch("http://127.0.0.1:19828/api/v1/projects/current/search", {
  method: "POST",
  headers: { "Authorization": `Bearer ${process.env.LLM_WIKI_API_TOKEN}`, "Content-Type": "application/json" },
  body: JSON.stringify({ query: "rope embedding", topK: 5 }),
})
const { results } = await res.json()
```

Python 的形态完全一样——`urllib.request`、`requests`、`httpx`，用你手头已有的任意一个就行。**不要安装任何新东西。**

## 鉴权模型

该 API **仅限本机（localhost）**。token 来自以下之一：

1. `LLM_WIKI_API_TOKEN` 环境变量（若设置，则覆盖 UI 里的设置）
2. 用户通过 Settings → API Server 保存的 `apiConfig.token`
3. `allowUnauthenticated: true` 模式（无需 token；少见，仅用户主动选择开启）

务必先检查 `/api/v1/health`——它返回 `{ enabled, authConfigured, allowUnauthenticated, tokenSource }`。**如果 `authConfigured: false && allowUnauthenticated: false`，请让用户打开 `Settings → API Server → Generate new token`**。在鉴权配置好之前不要继续。

发送 token 有三种等价方式：

```
Authorization: Bearer <token>          # preferred
X-LLM-Wiki-Token: <token>              # alternative header
?token=<urlencoded-token>              # query param — last resort, leaks into logs
```

**绝不记录或回显 token。绝不把它放进任何用户能在你输出里看到的 URL**（Referer / shell 历史 / 日志都会泄露它）。

## 标准工作流

当用户说"在我的 wiki 里查一下"时：

1. **解析项目**（见下文 [项目解析](#项目解析)）。
2. **搜索**：`POST /api/v1/projects/{id}/search`，带 `{ query, topK: 5..10 }` → 排序后的命中结果（`path`、`title`、`snippet`、`score`、`titleMatch`，可选的 `vectorScore`、`images`）。检查 `response.mode` 以了解混合检索是否生效。
3. **读取靠前的命中**：对每个有希望的命中，用 `GET /api/v1/projects/{id}/files/content?path=...` 获取完整 markdown。或者在搜索时传 `includeContent: true` 以避免多一次往返。
4. **引用 + 作答**：基于读到的页面综合出有据可依的答案。**引出你用到的每个页面的 `path`**，以便用户核实并在应用内跳转。

### 解读 score

`score` 字段的量纲取决于 `mode`：

- **`mode: "keyword"`**——可累加的关键词得分。文件名精确命中约为 200；标题中的短语命中约 50+；按 token 词袋命中落在个位数。把低于最高结果约 5% 的都视为低置信度。
- **`mode: "hybrid"` 或 `"vector"`**——RRF（Reciprocal Rank Fusion，倒数排名融合）得分，通常在 **0.015–0.035** 区间。绝对数值很小；重要的是相对排序。若需要，可用每条结果的 `vectorScore`（原始余弦值 0–1）来判断"embedding 匹配有多强"。

不要跨 mode 套用固定的得分阈值。按 `score` 降序排序，并依赖相对差距。

### 项目解析

每个面向项目的端点中的 `{id}` 接受**四种形式**：

| 形式 | 何时使用 | 示例 |
|---|---|---|
| `current`（字面量） | "我的 wiki / 我的知识库 / 这个项目 / 这个 wiki" 的默认值。用户指的是桌面 UI 里当前打开的那个。 | `/api/v1/projects/current/search` |
| UUID | 用户粘贴了项目 ID，或你之前已把某个名称解析为 ID 并想复用它。 | `/api/v1/projects/a0e90b29-fcf3-4364-9502-8bd1272de820/files` |
| 绝对文件系统路径（URL 编码） | 用户指名了路径（如 `~/notes/research`）。当用户有多个名称相近的项目时很有用。 | `/api/v1/projects/%2FUsers%2Fme%2Fwiki%2Fresearch/files` |
| 项目名称 | **不直接支持。** 你必须先 `GET /api/v1/projects`，按 `name` 找到匹配，然后使用该项目的 `id`。 |

针对用户所说内容的**决策树**：

```
"my wiki" / "my 知识库" / "this wiki" / "this project" / unspecified
    → use `current`

"my Research project" / "in Reading"
    → GET /api/v1/projects
    → name-match (case-insensitive substring on `name`)
    → use the resulting `id`
    → if 0 matches: tell the user, list available names, fall back to `current` only if they confirm
    → if 2+ matches: ask the user to disambiguate, quoting both names + paths

"the project at /Users/me/foo"
    → URL-encode the path, use directly
    → if the API returns 404, the project isn't registered — list and let user pick

"project a0e90b29-…"
    → use the UUID literally
```

在本次对话余下过程中缓存解析出的 `id`——没必要每次调用都重新 `GET /projects`。但如果用户在对话中途切换上下文（"现在去我的 Reading 项目里找"），就重新解析。

当用户没说用哪个项目时，**默认用 `current`**，并提一次：*"在你当前活动的项目（Research Notes）里查找……"*。这样可以避免跨项目的意外。

针对图谱 / 交叉引用类问题：

- `GET /api/v1/projects/{id}/graph?limit=200` → `{ nodes: [{id, label, nodeType, path, linkCount}], edges: [{source, target, weight}] }`
- 用 `?q=term`（对 id/label 做子串匹配，不区分大小写）和 `?nodeType=entity|concept|...` 过滤

针对"我加了新文档"类请求：

- `POST /api/v1/projects/{id}/sources/rescan` → 返回 `{ queue: { tasks }, changedTasks: [...] }`。告诉用户有多少文件发生了变化。实际摄取通过桌面端队列异步运行。

## 端点契约（v1）

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/v1/health` | 无需鉴权。返回 `{ ok, status, version, enabled, authRequired, authConfigured, allowUnauthenticated, tokenSource }`。 |
| GET | `/api/v1/projects` | 列出项目。每项：`{ id, name, path, current }`。 |
| GET | `/api/v1/projects/{id}/files?root=wiki\|sources\|all&recursive=true&maxFiles=2000` | `{ name, path, isDir, size, children }` 构成的树。上限 10000 个节点（超出返回 413）。 |
| GET | `/api/v1/projects/{id}/files/content?path=wiki/foo.md` | 仅限文本文件（md/mdx/txt/json/yaml/yml/csv/html/htm/xml/rtf/log）。最大 2 MB。二进制返回 415，超大返回 413，越界路径返回 403。 |
| POST | `/api/v1/projects/{id}/search` | 请求体：`{ "query": "...", "topK": 10, "includeContent": false }`。当用户在 Settings 中配置了 embeddings 时使用**混合检索（关键词 + 向量）**；否则回退为纯关键词。响应携带 `mode: "keyword" \| "vector" \| "hybrid"`，以及 `tokenHits` / `vectorHits` 和每条结果的 `vectorScore`。空 query → 400。 |
| GET | `/api/v1/projects/{id}/graph?q=&nodeType=&limit=200` | 来自 `wiki/*.md` 的 wikilinks 图谱。limit 被钳制到 1000。 |
| POST | `/api/v1/projects/{id}/sources/rescan` | 使用用户的 Source Watch 配置触发后端重扫描。返回重扫描后的队列 + 实际发生变化的任务。 |
| POST | `/api/v1/projects/{id}/chat` | **501**——v1 中未实现。不要调用。 |

`{id}` 接受 UUID、绝对文件系统路径（URL 编码）或字面字符串 `current`。

## 错误处理

始终把状态码当作契约来对待：

| 状态码 | 含义 | 该怎么做 |
|---|---|---|
| 200 | OK | 双保险地用 `body.ok === true` 校验；负载就在同一个对象里。 |
| 400 | Bad request | 显示 `body.error`。常见：空 `query`、非法 `?root=`、请求体过大。 |
| 401 | Unauthorized | token 缺失/错误。让用户在 Settings → API Server 设置/重新生成。 |
| 403 | Forbidden | 路径穿越或越界（如 `../app-state.json`）。不要对同一路径重试。 |
| 404 | Not found | 未知项目 id 或未知路由。遇到未知项目时，先列出项目以便恢复。 |
| 405 | Method not allowed | HTTP 动词用错了。 |
| 413 | Payload too large | 文件 > 2 MB、文件树 > maxFiles，或请求体 > 1 MB。建议缩小范围。 |
| 415 | Unsupported media | 二进制或非 UTF-8 文件内容。API 仅支持文本。 |
| 429 | Too many requests | 限流（全局 120 req/sec）。退避 ≥1 秒。 |
| 500 | Internal error | 记录 + 报告；不要死循环。 |
| 501 | Not implemented | `/chat` 占位。不要重试。 |
| 503 | Service unavailable | 两种情况：API 被关闭（`error` 含 "disabled"）；在途请求达到上限（64）（"busy"）。退避 ≥2s。 |

如果 HTTP 调用本身失败（连接被拒 / ENOTFOUND）：说明桌面应用**没在运行**。告诉用户："启动 LLM Wiki，然后重试。"

## 礼仪

- **引用路径。** 当你用 wiki 内容作答时，要指名页面：`(from wiki/concepts/rope.md)`。用户用这些来核实并在应用内跳转。
- **默认保持只读。** 只有 `sources/rescan` 会改动状态；其余都是读取。不要臆造写入端点——v1 里没有它们。
- **不要在未被要求时倾倒整页内容。** 通常片段 + 路径就够了。只有在推理确实需要时才拉取完整内容。
- **尊重项目边界。** 当前项目是用户的活动上下文。不要悄悄切换项目。
- **遵守限流。** 120 req/sec 对顺序工作绰绰有余，但并行读取页面可能瞬时逼近上限。在 API 允许处做批处理（搜索时用 `includeContent: true` 可避免 N+1 次读取）。
- **绝不泄露 token。** 请求头是安全的；query 参数和你自己的输出文本不安全。

## 另见

- `api-reference.md`——完整的端点形态，含请求 / 响应示例
- `examples.md`——常见对话模式映射到直接的 `curl` / `fetch` 序列
- `README.md`——面向人的安装说明（token 生成、端口冲突、故障排查）
