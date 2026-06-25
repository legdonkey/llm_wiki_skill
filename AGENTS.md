# 私有维护规范

本仓库是 `nashsu/llm_wiki_skill` 的私有 fork。维护时务必遵循私有规范，**先读 [`private/README.md`](private/README.md)**（远端约定、分支、升级流程、私有版本号）。

三条红线：

1. **私有内容尽量隔离**：新增私有文件一律放 `private/`（本指针段除外）；不往 upstream 维护的文件/目录里新建私有文件。
2. **必改上游的登记入账**：确需直接改 upstream 文件（如本仓库的 `README.md`/`SKILL.md`/`api-reference.md`/`examples.md`），必须登记到 [`private/CHANGES-REGISTRY.md`](private/CHANGES-REGISTRY.md) 并定好升级冲突策略。
3. **绝不推 upstream**：`upstream` 远端 push 已设为 `DISABLED`，私有改动只推 `origin`。

每做一处私有定制，登记到 `private/CHANGES-REGISTRY.md`；升级 upstream 前读 `private/README.md` 的升级流程。中文参考译文见 `private/translations/`。
