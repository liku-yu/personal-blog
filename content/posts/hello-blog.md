---
title: 你好，这是我的技术博客
description: 这个博客的定位、写作约定，以及如何发布一篇新文章。
date: 2026-07-30
tags:
  - 博客
---

欢迎来到我的技术博客 👋

这个博客主要记录我在写代码过程中的**技术笔记、踩坑记录和思考**。内容以技术文章为主，偶尔会有工具和效率相关的分享。

## 如何发布新文章

操作非常简单，一共三步：

1. 在 `content/posts/` 目录下新建一个 `.md` 文件（文件名就是文章链接，例如 `hello-world.md` → `/posts/hello-world/`）
2. 在文件开头填写元信息（frontmatter）
3. 用 Markdown 写正文，执行 `hugo server` 本地预览，构建后自动发布

> 想了解本主题支持的 Markdown 扩展能力（frontmatter 字段、提示框短代码等），看这篇：[用 Markdown 写文章：Void 主题写作指南](/posts/mdx-writing-guide/)。

## 现有内容一览

博客目前主要覆盖这些方向，欢迎按需阅读：

- **前端基础**：[CSS Grid 常用布局速查表](/posts/css-grid-cheatsheet/) — 响应式卡片墙、居中、整页骨架
- **React**：[深入理解 useEffect 的依赖数组](/posts/react-use-effect/) — 闭包陷阱、竞态、exhaustive-deps
- **TypeScript**：[用 TypeScript 写一个可即时取消的防抖函数](/posts/typescript-debounce/) — 泛型与 cancel/flush
- **搜索原理**：[Pagefind 静态搜索技术解析](/posts/pagefind-technical/) — 倒排索引与 WASM 检索

下面是一个文章模板：

```yaml
---
title: 文章标题
date: 2026-09-05
tags: [TypeScript, 前端]
description: 一句话摘要，会显示在列表卡片上
---
```

## 写作约定

| 项目 | 约定 |
| --- | --- |
| 语言 | 中文为主，代码注释用英文 |
| 格式 | `.md`（支持 GFM 表格、代码高亮、KaTeX 数学公式） |
| 代码块 | 标注语言名，构建时自动高亮 |
| 图片 | 使用完整 URL，或放在 `static/` 下用 `/xxx.png` 引用 |
| 更新 | 记得同步更新 `date` 字段 |

祝你写作愉快 🚀