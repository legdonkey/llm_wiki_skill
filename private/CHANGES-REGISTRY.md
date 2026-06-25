# 私有改动台账

记录本 fork 相对 upstream 的私有定制。**只记 git 看不出的东西** —— 「为什么改」和「升级冲突时怎么处理」。
文件清单用 git 命令即可（upstream 无 tag，基线用 `upstream/main`）：

```bash
git diff --stat upstream/main...main
```

每做一处私有定制就在此登记；升级合并冲突时按此表逐条核对，优先保留私有版再并入 upstream 新内容。

| 改动 | 涉及文件/目录 | 为什么 | 升级冲突策略 | 引入版本 |
|------|--------------|--------|--------------|----------|
| 私有维护规范 | `private/README.md` | fork 维护流程，upstream 没有 | 保私有版 | v0-private.1 |
| 私有目录 | `private/` | 台账/规范/译文，与 upstream 隔离 | upstream 无此目录，不冲突 | v0-private.1 |
| 中文参考译文 | `private/translations/*.zh.md` | 团队中文参考，不回推 upstream | upstream 无此目录，不冲突；原文变动后增量重译 | v0-private.1 |
| 指针段 | `CLAUDE.md` / `AGENTS.md` | 让 CC/Codex 自动遵循私有规范 | 保私有段，再并入 upstream 新内容 | v0-private.1 |

## 高冲突文件清单

以下文件最容易和 upstream 撞车，升级合并时重点检查，默认「保私有 + 手动并入 upstream 新内容」：

- `README.md` —— upstream 主介绍文档，更新频繁
- `SKILL.md` —— skill 入口与触发描述，upstream 会调整
- `api-reference.md` —— API 契约文档，随 upstream API 变化
- `examples.md` —— 用法示例，随 upstream 更新
- `CLAUDE.md` / `AGENTS.md` —— 私有指针段所在；upstream 若新增同名文件需手动并段

> 说明：本项目所有 upstream 跟踪文件都是文档（markdown），私有改动应尽量只放 `private/`；确需直接改上述文档时务必回到本表登记。

## 本项目的特殊坑（勘察阶段发现，逐条记录）

- **upstream 无任何正式 tag**：升级基线只能用 `upstream/main`，私有 tag 从 `v0-private.1` 起；待 upstream 开始打 tag 后再切换到真实版本号。
- **无 `.gitignore`**：upstream 未提供忽略规则，新增私有文件不会被误伤，但也无「故意跟踪」标记需注意。
- **无版本号文件**：项目是纯文档型 skill，版本以 upstream git 历史为准。
- **无本地状态文件**（`.obsidian`/`.idea`/`.vscode`）：无需 `skip-worktree` 处理。
