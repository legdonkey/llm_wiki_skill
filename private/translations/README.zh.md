---
source: README.md
source_version: 0
translated_at: 2026-06-25
---

# llm-wiki skill

让 AI 智能体（Claude Code、Codex 等）通过内置的 HTTP API 查询用户本地运行的 **LLM Wiki** 桌面应用。

本技能**仅为文档**。该 API 是一套标准的 HTTP+JSON 契约——用 `curl`、`fetch`、`requests`，或你环境中已有的任意 HTTP 工具调用它即可。没有客户端库，没有 SDK，也没有编译步骤。

## Installation

### 推荐方式——通过 `skills` CLI 一行命令安装

```bash
npx skills add https://github.com/nashsu/llm_wiki_skill.git --skill llm-wiki
```

这会从 GitHub 拉取该技能的最新版本，并将其注册到你的智能体运行时。之后想更新到更新的版本时，重新运行同一条命令即可。

### 备选方式——克隆 + 软链接（Claude Code）

如果你更倾向于在本地管理源码并将其链接到 Claude Code 中（技能位于 `~/.claude/skills/`）：

```bash
git clone https://github.com/nashsu/llm_wiki_skill.git
ln -s "$(pwd)/llm_wiki_skill" ~/.claude/skills/llm-wiki
```

### 备选方式——手动放置

该技能仅由三个 markdown 文件构成，别无其他。把文件夹放到任何你的智能体运行时能发现技能的位置（例如 Codex 的 `agents/`、自定义 MCP server 的技能目录）。无依赖，无构建步骤。

## What this skill enables

- "我的 wiki 里关于 X 是怎么说的？" → 混合检索（关键词 + 向量）+ 读取
- "在我的图谱中展示节点 Y 的邻域" → wikilinks 遍历
- "把关于 Z 的页面读给我听" → 文件内容获取
- "我刚加了新文档，重新索引" → 后端重新扫描
- "给我一个 wiki 的结构性概览" → 文件树 + index.md

除 `sources/rescan`（会触发一次内部队列差异比对）外，全部为只读操作。

## Files

| File | Purpose |
|---|---|
| `SKILL.md` | 面向智能体的指令。由 AI 运行时自动加载。 |
| `api-reference.md` | 完整的端点契约（状态码、参数、响应结构）。 |
| `examples.md` | 对话 → API 配方模式。 |
| `README.md` | 本文件——面向人类的设置 / 安装 / 故障排查。 |

没有脚本，没有封装。如果你发现自己想要一个，直接用 `curl` 就好。

## Prerequisites

1. **LLM Wiki 桌面应用已安装并正在运行。** 该 API 仅在应用打开期间绑定到 `127.0.0.1:19828`。如果应用未运行，智能体会看到 `connection refused` 并告知你。

2. **API server 已启用。** 默认：开启。在 `Settings → API Server → "Enable local HTTP API"` 处确认其已勾选。

3. **已配置 token。** 默认：关闭（token 字段为空 → 每个端点都返回 401）。二选一：
   - 在应用中：`Settings → API Server → Generate new token`。复制并妥善保存。
   - 通过环境变量：`export LLM_WIKI_API_TOKEN=...`（覆盖 UI 设置；对智能体进程持续有效）。

   你也可以打开 `Settings → API Server → "Allow access without a token"`，这会完全移除鉴权。**仅限本机——但此时本机上的任意进程都能读取你的 wiki。** 仅在受信任的环境中使用。

## Quick smoke test

> **Windows 用户**：把 `export LLM_WIKI_API_TOKEN=...` 替换为 `$env:LLM_WIKI_API_TOKEN = "..."`（PowerShell）或 `set LLM_WIKI_API_TOKEN=...`（cmd.exe）。在 PowerShell 中用 `curl.exe` 代替 `curl`，以绕过 `Invoke-WebRequest` 别名。反斜杠续行符（`\`）在 PowerShell 中变为反引号（`` ` ``），在 cmd 中变为 `^`。

```bash
export LLM_WIKI_API_TOKEN=...

# 1. Probe — no auth needed
curl -s http://127.0.0.1:19828/api/v1/health

# Expected:
# {
#   "ok": true,
#   "status": "running",
#   "enabled": true,
#   "authConfigured": true,
#   "tokenSource": "env",
#   "allowUnauthenticated": false,
#   ...
# }

# 2. List projects
curl -s -H "Authorization: Bearer $LLM_WIKI_API_TOKEN" \
  http://127.0.0.1:19828/api/v1/projects

# 3. Search the active project
curl -s -H "Authorization: Bearer $LLM_WIKI_API_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"query":"rope","topK":3}' \
  http://127.0.0.1:19828/api/v1/projects/current/search

# 4. Read a page
curl -s -H "Authorization: Bearer $LLM_WIKI_API_TOKEN" \
  "http://127.0.0.1:19828/api/v1/projects/current/files/content?path=wiki/concepts/rope.md"
```

如果你装了 `jq` 且想要格式化的输出，可以在上述任意命令末尾加上 `| jq`。这不是必需的。

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `curl: (7) Failed to connect to 127.0.0.1 port 19828` | 应用未运行 | 启动 LLM Wiki 桌面应用，等待约 2 秒让 API server 完成绑定。 |
| `/health` 中出现 `Status: port_conflict` | 另有进程占用 19828 | 用 `lsof -i :19828` 找出占用者；退出并重启应用。 |
| 每个端点都返回 401 | 未配置 token | Settings → API Server → Generate；或 `export LLM_WIKI_API_TOKEN=...` |
| 每个端点都返回 503 并提示 "disabled" | 总开关处于关闭状态 | Settings → API Server → 勾选 "Enable local HTTP API"。 |
| 每个端点都返回 503 并提示 "busy" | 已达到并发上限（64 并发） | 退避，减少并行请求数量。 |
| 快速请求时出现 429 | 速率限制：全局 120 req/sec | 退避 ≥1 秒。 |
| `files/content` 对 `.pdf` 返回 415 | API 仅支持文本 | 通过桌面 UI 读取源 PDF；仅暴露文本类扩展名。 |
| 即便已在 UI 中清空，`tokenSource` 仍为 `"env"` | 某处设置了 `LLM_WIKI_API_TOKEN` 环境变量 | 执行 `unset LLM_WIKI_API_TOKEN`；或将其值设置为与 UI 中一致。 |

## Security model

- 该 API 仅监听 `127.0.0.1`。它**无法从你网络中的其他主机访问**。
- 基于 token 的鉴权采用常数时间比较（鉴权检查中无计时泄露）。
- 路径穿越在路由处理器处被阻断（`safe_join` 配合规范化路径前缀检查）。
- 文件读取限制在白名单内（`purpose.md`、`schema.md`、`wiki/**`、`raw/sources/**`）；仅限文本扩展名；上限 2 MB。
- 请求体上限 1 MB；并发上限 64；速率限制 120/sec。
- v1 中没有写入端点。`/sources/rescan` 是唯一的变更操作，而且它只读取磁盘 + 排队内部工作。

威胁模型中**不包含**：

- 持有 token 的恶意**本地**进程可以读取你当前项目 wiki + sources 中的一切。请把 token 当作本机机密对待。
- 恶意本地进程可以探测 `/health`（无需鉴权）以发现该 API 的存在。属于信息泄露，但不泄露内容。
- 跨源浏览器页面只有在持有 token 时才能访问该 API。默认 CORS 为 `*`——依赖 token 的保密性。不要把你的 token 粘贴到不受信任的 web 工具里。

## Version

本技能对应 **LLM Wiki API v1**，即应用版本 `0.4.10+` 中所发布的版本。混合检索（关键词 + 向量）已上线；响应中携带 `mode: "keyword" | "vector" | "hybrid"`、`tokenHits`、`vectorHits`，以及每条结果的 `vectorScore`。如果桌面应用的 `/health` 报告了主版本号的提升，请在逐字依赖该契约之前先查看 `api-reference.md` 是否有变动。

## Updating

如果你是通过 `npx skills add` 安装的，重新运行同一条命令即可拉取最新版本：

```bash
npx skills add https://github.com/nashsu/llm_wiki_skill.git --skill llm_wiki_skill
```

如果你是克隆的仓库，在克隆出的文件夹内执行 `git pull` 即可——软链接会自动获取改动。

## Contributing

源码仓库：<https://github.com/nashsu/llm_wiki_skill>

欢迎提交 Issue 和 PR。本技能应当保持**无客户端**——没有脚本，没有 SDK 垫片，没有封装的二进制文件。底层 API 的全部意义就在于任何 HTTP 工具都能调用它；本技能是契约文档，而非运行时。

## License

许可证详情见仓库。
