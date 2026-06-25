# 私有维护规范

本文档记录 `legdonkey/llm_wiki_skill` fork 的远端、分支、升级和私有版本规则。目标是让私有化定制可以长期维护，同时持续跟踪 upstream 的稳定版本。

> 关系速记：`origin` = 我自己的 fork（`legdonkey/llm_wiki_skill`）；`upstream` = 原作者仓库（`nashsu/llm_wiki_skill`），只读、只拉不推。

## 一次性初始化

同事从 `origin`（私有 fork）克隆后，配置 upstream 只读远端 → 禁推（push 设为 `DISABLED`）→ `git fetch upstream --tags`。两种方式：

- **装了 privatize-fork skill**（CC 或 Codex）：直接**重跑该 skill**（幂等，自动补齐 upstream 配置，并增量刷新翻译）。
- **没装 skill**：手动执行等价操作即可——

```bash
git remote add upstream git@github.com:nashsu/llm_wiki_skill.git
git remote set-url --push upstream DISABLED
git fetch upstream --tags
```

## 远端约定

- `origin`: `git@github.com:legdonkey/llm_wiki_skill.git`
- `upstream` fetch: `git@github.com:nashsu/llm_wiki_skill.git`
- `upstream` push: `DISABLED`
- 本地 `main` 跟踪 `origin/main`，`upstream` 只作更新来源，不推私有改动

## 分支约定

标准开发分支名，唯独保留 `upgrade/` 表达「合并 upstream」这一特殊语义。

- `main`: 私有主干，推 `origin/main`
- `feat/<topic>` / `fix/<topic>` / `hotfix/<topic>`: 私有开发
- `upgrade/upstream-<基线>`: 合并 upstream 稳定版

### 历史红线

- 私有 `main` **只 merge，永不 rebase、永不 force-push**。
- 例外：fork 初始化阶段、确认无人 clone 时，允许一次性整理初始提交；一旦有协作者就严守此红线。

## 私有版本号

格式 `v0-private.<N>`。规则：

- 本项目 upstream **暂无正式 tag**，故基线版本记为 `0`（占位）。一旦 upstream 开始打 tag，基线改为对应真实 tag（如 `v1.2.0`），格式同步变为 `v1.2.0-private.<N>`。
- `<N>` 从 1 递增；升级到新 upstream 版本后重置为 1。
- `*-private.*` tag 只推 `origin`，不推 `upstream`。

## upstream 同步与版本判断

- 优先升级到 upstream 打了正式 tag 的版本；本项目当前无 tag，只能跟踪 `upstream/main` 的稳定提交，升级前务必看 diff。
- 升级前按可靠性递增地读：① `CHANGELOG` / release notes（看意图，本项目暂无）② `git log --oneline <旧基线>..upstream/main`（看提交）③ `git diff --stat <旧基线>..upstream/main`（**ground truth**，看动了哪些文件，与高冲突清单交叉比对）。
- 官方版本号体现在：无版本号文件，仅以 upstream 的 git tag / `upstream/main` 提交为准。

## 私有改动登记与目录

- **台账**：每做一处私有定制，登记到 [`CHANGES-REGISTRY.md`](CHANGES-REGISTRY.md)，重点记「为什么改」和「升级冲突策略」（文件清单用 `git diff --stat upstream/main...main` 自动得到）。

### 私有内容存放

```text
private/                 # 私有数据：本规范、台账、译文
private/translations/    # upstream 官方文档的中文参考译文（不回推）
```

upstream 初始化与文档翻译由 privatize-fork skill 内联完成（重跑即可）。让 AI 自动遵循本规范的指针段写在 `CLAUDE.md`（CC）和 `AGENTS.md`（Codex）。

## 升级流程

```bash
git fetch upstream --tags
git checkout -b upgrade/upstream-<基线>
git merge upstream/main    # 参照台账「高冲突文件清单」解决冲突；有正式 tag 后改 merge <tag>
# —— 验证：本项目无自动化测试，靠人工验证（在编辑器/agent 里确认 skill 文档可正常加载与触发）——
git checkout main && git merge upgrade/upstream-<基线>
git tag v0-private.<N>
git push origin main && git push origin v0-private.<N>
```

### 回滚预案

- 升级一律在 `upgrade/` 分支做，验证通过才并回 `main`；并回前记 `git rev-parse main` 备用。
- 私有 tag 一旦推送到 `origin` 不要删除/移动；如需修正，递增 `-private.<N>` 另打新 tag。

## 发布前检查

- `origin` 指自己的 fork，`upstream` push = `DISABLED`
- `main` 跟踪 `origin/main`
- 私有 tag 明确包含 upstream 基线版本（当前为 `0`）
- 工作区已有用户改动未被回滚
