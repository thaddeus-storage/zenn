---
name: zenn
description: Zenn 仓库内容的创建、编辑、审查、预览与发布规范。Use when working with Zenn articles, books, images, Front Matter, Zenn Markdown, scheduled publishing, or GitHub deployment.
---

# Zenn

以 [Zenn Manual](https://zenn.dev/manual) 为准。遇到不确定或可能变化的部署规则时，先查官方文档。

## 内容

- 文章放在 `articles/<slug>.md`；书放在 `books/<slug>/`，包含 `config.yaml` 和章节 Markdown。
- 不修改已发布内容的 slug。新 slug 使用小写字母、数字、`-`、`_`，长度为 12–50。
- Web 导出的既有书继续保留 `allow_override: true`。Scrap 不参与 GitHub 同步。
- 图片放在根目录 `images/`，以 `/images/...` 引用；单文件不超过 3 MB，仅使用 PNG、JPEG、GIF 或 WebP。
- 写作或推敲日文技术内容时，同时使用 `japanese-tech-writing`。

## Zenn Markdown

- 正文见出し优先从 `##` 开始。
- 代码块指定语言；用 `` ```lang:filename `` 显示文件名，用 `` ```diff lang `` 显示差异。
- 使用 `:::message`、`:::message alert` 和 `:::details 标题` 表示提示、警告和折叠内容。
- 使用 `[^label]` 或 `^[注释]` 添加脚注；单独一行 URL 或 `@[card](URL)` 生成链接卡片。
- 支持 KaTeX、外部嵌入和 Mermaid；Mermaid 单块不超过 2,000 字符。

## 发布与验证

- 文章 Front Matter 使用 `title`、`emoji`、`type`、`topics` 和 `published`。
- 文章预约发布设置 `published: true` 与 `published_at: YYYY-MM-DD hh:mm`；时区为 JST。书不使用 `published_at`。
- 优先运行 `npx zenn preview` 检查渲染；项目未安装 Zenn CLI 时，先报告并请求设置权限。
- 修改书籍时验证 YAML 和全部章节引用；修改图片时验证路径、扩展名和大小。
- 只有用户明确要求时才提交、推送、发布或删除。删除 Zenn 内容时同时处理 Dashboard 和仓库文件。
