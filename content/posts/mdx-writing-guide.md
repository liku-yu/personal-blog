---
title: 用 Markdown 写文章：Void 主题写作指南
description: 在 Hugo + Void 主题下，文章支持哪些扩展语法？Frontmatter、提示框、代码高亮、KaTeX 一网打尽。
date: 2026-08-10
tags:
  - Markdown
  - 博客
---

**Hugo + Void 主题**的写法主体仍是 Markdown，但这个主题额外提供了目录导航、代码高亮、KaTeX 数学公式、暗色模式等特性。

## 如何写一篇文章

在 `content/posts/` 下新建 `my-post.md`，文件开头填写 frontmatter：

```yaml
---
title: "文章标题"
date: 2026-09-07
tags: ["TypeScript", "前端"]
---
```

支持的可选字段：

- `description` — 摘要，显示在列表卡片
- `tags` — 标签（自动生成标签页）
- `language` — 单篇语言覆盖（如给中文博客里的英文文章标 `language: "en"`）
- `numbered: true` — 章节自动编号（1 / 1.1 / 1.1.1）

## 代码高亮

Void 使用 Hugo 内置的 Chroma 高亮，代码块带语言标签与复制按钮，暗/亮模式自动切换配色：

```python
def greet(name: str) -> str:
    return f"Hello, {name}!"
```

## 提示框（Callout）

支持短代码与栅栏两种写法：

{{< callout title="Note" type="info" >}}
这是 **info** 类型的提示框。
{{< /callout >}}

```callout {.tip title="Tip"}
这是 **tip** 类型。技术文章中推荐这么做时用它最合适。
```

```callout {.warning title="Caution"}
这是 **warning** 类型。提醒读者注意的坑、边界情况。
```

```callout {.danger title="Danger"}
这是 **danger** 类型，用来警告严重的错误或高风险操作。
```

支持的 `type`：`info`（默认）、`tip`、`warning`、`danger`、`neutral`。

## 数学公式（KaTeX）

KaTeX 在**构建时**渲染（无需客户端 JS），支持 `$…$` 行内与 `$$…$$` 块级：

行内公式 $E = mc^2$，以及块级公式：

$$
\int_{-\infty}^{\infty} e^{-x^2} dx = \sqrt{\pi}
$$

甚至可以在 callout / 表格里内嵌公式：

{{< callout title="定义" type="neutral" >}}
状态-动作价值函数：
$$ v_\pi(s_t, a_t) = \mathbb{E}_\pi\left[ \sum_{i=t}^T \gamma^{i-t} r_i \right] $$
{{< /callout >}}

> **注意**：块级公式要换行写，前后留空行；公式里含 `\{` 等花括号时用 callout 的栅栏语法（避免误解析）：
> `​​​```callout {.neutral title="定义"}` 把 `title` 写进属性即可。

## 目录（TOC）

文章存在标题时自动生成目录：宽屏下显示在右侧栏，窄屏下折叠在文章顶部，带滚动高亮。

## 写作约定

| 项目 | 约定 |
| --- | --- |
| 语言 | 中文为主，代码注释用英文 |
| 格式 | `.md`（支持 GFM 表格、代码高亮、KaTeX） |
| 代码块 | 标注语言名，构建时自动高亮 |
| 图片 | 使用完整 URL，或放在 `static/` 下用 `/xxx.png` 引用 |
| 更新 | 记得同步更新 `date` 字段 |

想新增内容？直接在 `content/posts/` 新建 `.md` 文件即可。