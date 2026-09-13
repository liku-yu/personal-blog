# 我的技术博客

🌐 **在线访问 → [https://liku-yu.github.io/personal-blog/](https://liku-yu.github.io/personal-blog/)**

[![Deploy to GitHub Pages](https://github.com/liku-yu/personal-blog/actions/workflows/deploy.yml/badge.svg)](https://github.com/liku-yu/personal-blog/actions/workflows/deploy.yml)
[![Hugo](https://img.shields.io/badge/Hugo-extended-FF4088?logo=hugo&logoColor=white)](https://gohugo.io/)
[![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-v4-38BDF8?logo=tailwindcss&logoColor=white)](https://tailwindcss.com/)
[![Pagefind](https://img.shields.io/badge/Search-Pagefind-2ea44f)](https://pagefind.app/)

基于 **Hugo + hugo-theme-void** 的纯静态博客（Tailwind CSS v4 技术栈）。

## 常用命令

```bash
npm install     # 安装依赖（Tailwind CLI + Pagefind，首次及 node_modules 被清理后需要）
npm run dev     # 本地开发：http://localhost:1313（hugo server）
npm run build   # 完整构建：hugo + 生成 Pagefind 搜索索引 → public/
npm run search  # 只重新生成搜索索引（构建后执行）
hugo new posts/xxx.md   # 新建文章草稿（draft: true）
```

## 写一篇文章

在 `content/posts/` 下新建 `my-post.md`：

```markdown
---
title: "文章标题"
date: 2026-09-05
tags: ["TypeScript", "前端"]
description: 一句话摘要
---
正文内容…
```

可选字段：`language`（单篇语言）、`numbered: true`（章节编号）、`featured: true`（置顶）。

- 文件名即链接：`my-post.md` → `/posts/my-post/`
- 草稿 `draft: true` 不会出现在站点，`npm run dev` 时加 `-D` 可预览
- **搜索**：构建完成后用 `npm run search` 生成索引，全站文章（含中文）即可在 `/search/` 页面实时搜索

## 目录结构

```
content/
├── _index.md          # 首页内容
├── about.md           # 关于页（个人简介 + 社交链接）
├── search.md          # 搜索页（layout: search，Pagefind 挂载点）
└── posts/             # ★ 文章
hugo.toml              # 站点主配置（params、菜单、安全规则、KaTeX）
layouts/               # ★ 站点级主题覆盖（勿删）
├── _default/baseof.html   # 布局修复：partialCached/Defer 冲突 + 首页垂直居中
├── _default/home.html     # 首页定制（头像/社交链接/导航居中、首屏不滚动）
├── about/single.html      # 关于页定制（头象放大、社交图标、去 Disqus）
└── partials/
    ├── head.html          # 修复主题在 Hugo 0.165 下的 partialCached 冲突
    └── footer.html        # 精简页脚（仅版权 + 访问量）
themes/void              # 主题（git submodule，勿直接改）
archetypes/              # 文章模板
package.json             # Tailwind CLI + Pagefind 依赖
```

## 自定义

- **博客名 / 简介**：`hugo.toml` 的 `title`、`[params].description`
- **导航菜单**：`[[menus.main]]`（identifier 匹配 i18n 键：nav_home/nav_posts/nav_tags/nav_about）
- **社交链接**：`[params].social`（github / cnb / email），首页头像下与关于页显示图标
- **头像**：`[params].avatar.url`（可换成本地图片放 `static/`）
- **代码高亮**：`[markup.highlight]`，style 由主题 Chroma 亮/暗双主题自动切换
- **KaTeX**：`$…$` / `$$…$$`，构建时渲染（`[markup.goldmark.extensions.passthrough]`）

> ⚠️ 修改主题样式请用 `layouts/` 覆盖（与主题布局同名即可优先生效），不要改 `themes/void/` 内的文件。

## 部署

### GitHub Pages（已配置自动部署）

已提供 `.github/workflows/deploy.yml`，推送到 `master` 自动构建并发布：

1. 仓库 **Settings → Pages → Build and deployment → Source** 选择 **GitHub Actions**
2. 之后每次 `git push` 会自动构建（Hugo + Pagefind）并部署
3. 访问 `https://liku-yu.github.io/personal-blog/`

工作流通过 `HUGO_BASEURL` 环境变量注入子路径地址，本地 `hugo.toml` 仍保持 `localhost`。

### 其他静态托管

```bash
npm run build   # 输出 public/（含搜索索引）
```

把 `public/` 目录丢到任意静态托管即可（Vercel / Netlify / Nginx…）。
若部署在子路径，构建时用 `HUGO_BASEURL=https://your.site/sub/ npm run build` 覆盖 baseURL。

## 技术栈

Hugo · hugo-theme-void · Tailwind CSS v4 · Chroma 高亮 · KaTeX（构建时渲染）· Pagefind 静态搜索（WASM）· 自托管字体