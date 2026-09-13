---
title: Pagefind 静态搜索技术解析：纯前端也能做的全文搜索
description: 深入讲解 Pagefind 的原理——构建时建索引、运行时 WASM 检索，让静态博客无需后端也能拥有强大的全文搜索。
date: 2026-09-08
tags:
  - Pagefind
  - Hugo
  - 搜索
---

本博客的搜索功能就是基于 **Pagefind** 实现的。这篇文章讲讲它为什么能在纯静态站点上提供全文搜索，以及它的工作原理。

## 为什么静态博客需要搜索

静态博客（Hugo、Hexo、Jekyll 等）没有后端、没有数据库，传统的站内搜索需要：

- 服务端分词 + 倒排索引（如 Elasticsearch）
- 或者第三方案例服务（如 Algolia，需付费账号）

Pagefind 的答案是：**把搜索索引在构建时生成好，运行时在浏览器本地完成检索**。这既保留了静态部署的简单，又能提供接近实时的搜索体验。

## 工作原理总览

```text
构建阶段（Node CLI）
┌────────────────────────────────────────────┐
│  npx pagefind --site public                │
│  ① 爬取 public/ 下所有 HTML                │
│  ② 提取文本 → 分词 → 建立倒排索引           │
│  ③ 压缩输出到 public/pagefind/             │
└────────────────────────────────────────────┘
                │ 产物（纯静态文件）
                ▼
运行时阶段（浏览器）
┌────────────────────────────────────────────┐
│  ① 检测 <html lang> → 加载对应语言索引      │
│  ② 用户输入 → Intl.Segmenter 分词          │
│  ③ 按需加载索引块 → WASM 计算相关度         │
│  ④ 加载片段 → 生成高亮摘要 → 渲染结果       │
└────────────────────────────────────────────┘
```

## 构建阶段：索引怎么生成

Pagefind 的 CLI 会爬取构建产物目录，对每个 HTML 页面：

1. **提取正文**：去除导航、页脚等重复内容（通过 `data-pagefind-body` / `data-pagefind-ignore` 标记控制权重区域）
2. **分词**：按语言规则切词，中英文都能处理
3. **建立倒排索引**：记录「词 → 出现在哪些页面 + 位置」
4. **压缩输出**：索引、片段、语言元数据分别压缩存储

### 倒排索引长什么样

倒排索引（inverted index）是搜索引擎的核心数据结构——反向的目录：**给定一个词，立刻查到它出现在哪些文档的哪些位置**：

```json
{
  "防抖":   [{ "doc": 3, "pos": [12, 58], "weight": 0.42 }],
  "依赖":   [{ "doc": 4, "pos": [3, 27] }],
  "grid":   [{ "doc": 5, "pos": [0, 40, 99] }]
}
```

相比正排（文档 → 词）需要扫描全库，倒排索引让查询复杂度降到只查相关条目，这也是 Pagefind 能秒回的原因之一。

### 控制哪些内容参与索引

默认整页都参与，但可以用标记颗粒化控制：

```html
<!-- data-pagefind-body：只索引本区域（正文），导航/页脚被忽略 -->
<div data-pagefind-body>
  <article>这里的内容会进入搜索</article>
</div>

<!-- data-pagefind-ignore：显式排除某块 -->
<footer data-pagefind-ignore>版权信息不参与搜索</footer>
```

本站主题的正文区域就带了 `data-pagefind-body`，所以搜索只命中文章内容，不会把导航菜单、页脚版权都搜出来。

生成的核心文件（都在 `public/pagefind/` 下）：

| 文件 | 作用 |
| --- | --- |
| `pagefind-entry.json` | 入口元数据（版本、语言、页面数） |
| `index/zh_*.pf_index` | 倒排索引块（词 → 文档） |
| `fragment/zh_*.pf_fragment` | 每个页面的内容片段（用于生成摘要） |
| `wasm.*.pagefind` | 搜索内核（Rust 编译的 WebAssembly） |
| `pagefind.js` / `pagefind-worker.js` | 运行时桥接 + Web Worker |

## 运行时：浏览器怎么搜

### 1. 语言自动检测

```javascript
// 从 <html lang> 读取语言
const langCode = document.querySelector("html").getAttribute("lang");
// 加载对应语言的索引和分词元数据
```

本站 `<html lang="zh">`，Pagefind 自动选择中文索引。

### 2. 分词：Intl.Segmenter

中文没有空格分词，Pagefind 利用浏览器内置的 `Intl.Segmenter` 按语言切词：

```javascript
const segmenter = new Intl.Segmenter("zh", { granularity: "word" });
for (const { segment: word } of segmenter.segment("防抖函数")) {
  console.log(word); // 逐个切出的词
}
```

> 这就是为什么中文文章能精确搜到「防抖」「依赖数组」等关键词。

### 3. 按需加载索引块

Pagefind 不会把整个索引都塞进浏览器，而是**先问 WASM 内核哪些块包含了查询词，再只加载那些块**：

```javascript
// 向 WASM 请求：哪些索引块包含这些词
const index_array = backend.request_indexes(ptr, term);
// 按返回的哈希只加载需要的块
index_array.forEach(hash => loadChunk(hash));
```

### 4. 相关度排序：BM25

检索核心是 **BM25 算法**（信息检索领域的经典排序模型），公式如下：

$$
\text{score}(D,Q) = \sum_{i=1}^{n} \text{IDF}(q_i) \cdot \frac{f(q_i, D) \cdot (k_1 + 1)}{f(q_i, D) + k_1 \cdot (1 - b + b \cdot \frac{|D|}{\text{avgdl}})}
$$

其中：

- $f(q_i, D)$ — 词 $q_i$ 在文档 $D$ 中出现的频率
- $|D|$ — 文档长度，$\text{avgdl}$ — 平均文档长度
- $k_1, b$ — 调节参数（词频饱和度、长度归一化）
- $\text{IDF}(q_i)$ — 逆文档频率（越稀有的词权重越高）

直观理解：**关键词出现越多、文档越短、词越稀有** → 得分越高。Pagefind 的 WASM 内核里就是这套计算。

### 5. 摘要与高亮

搜索到结果后，Pagefind 从 `fragment` 里加载页面片段，**选择关键词最密集的段落**作为摘要，并自动包上 `<mark>` 高亮：

```html
<p>
  防抖（debounce）是前端高频场景的基础工具：**在事件停止触发一段时间后…
  <mark>防抖</mark>把"连续触发"折叠成"一次执行"…
</p>
```

## 在本站的接入

本站通过 npm 脚本串联构建与索引：

```bash
npm run build   # = hugo + pagefind --site public
```

搜索页（`content/search.md`）用主题自带的模板挂载 UI：

```html
<link rel="stylesheet" href="pagefind/pagefind-ui.css">
<div id="search"></div>
<script src="pagefind/pagefind-ui.js"></script>
<script>
  new PagefindUI({
    element: "#search",
    showSubResults: true,
    translations: { placeholder: "搜索..." }
  });
</script>
```

## 优缺点

{{< callout title="优点" type="tip" >}}
- **零后端**：纯静态文件即可部署，GitHub Pages / Vercel / Nginx 都行
- **隐私好**：搜索在浏览器本地完成，不把查询发给第三方（Algolia 会）
- **无费用**：不依赖付费搜索服务
- **速度快**：索引分块按需加载，WASM 内核高效
{{< /callout >}}

{{< callout title="注意" type="warning" >}}
- **构建多一步**：每次要重新生成索引（已打包进 `npm run build`）
- **中文无词干还原**：不支持英文那样搜索复数/时态变形（对中文无影响）
- **无服务端统计**：看不到搜索词统计（需要自行埋点）
{{< /callout >}}

## 小结

Pagefind 的思路很巧妙：**把"建索引"这步重活放在构建时，把"检索"放在浏览器端的 WASM 里**。静态站点因此也能拥有带相关度排序、高亮摘要的全文搜索，且部署、隐私、成本都比第三方搜索服务更有优势。

如果你也想在自己的 Hugo / 静态站里加搜索，试试 `npm i -D pagefind && npx pagefind --site public`，配上本文前面的配置即可。