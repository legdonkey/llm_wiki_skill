# 官方文档中文翻译（私有参考）

upstream 官方文档的中文译文，仅作团队参考，**不回推 upstream**。

## 翻译范围（白名单）

本清单由 privatize-fork 阶段2「翻译范围选择框」扫描本项目文档后确认，是增量翻译的**唯一依据**。
增删翻译范围改这里即可（不要改 skill 流程文件），重跑 skill 会照此刷新。

**翻译（白名单）**：

- `README.md`
- `api-reference.md`
- `examples.md`
- `SKILL.md`

**显式排除（黑名单，不翻）**：

- （无）

## 命名约定

- 译文文件名：`<原文件名去扩展>.zh.md`，例如 `README.zh.md`。
- 译文头部加 frontmatter，标注译自哪个源文件、哪个 upstream 版本：

```yaml
---
source: README.md
source_version: 0
translated_at: 2026-06-25
---
```

> 本项目 upstream 无正式 tag，`source_version` 暂用 `0` 占位；upstream 开始打 tag 后改为真实版本号。

## 防过期

译文会随 upstream 原文更新而过时。升级 upstream 后：

1. 对比每个译文的 `source_version` 与新基线版本。
2. 若源文件在新版本有改动（下面命令输出非空），重译并更新 frontmatter：

```bash
git diff --stat upstream/main..main -- <source 路径>
```
