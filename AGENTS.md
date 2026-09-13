# Blog 项目指南

基于 **Hugo + hugo-theme-void** 的纯静态博客（Tailwind CSS v4）。

## 要点

- 内容在 `content/posts/`，Markdown 编写，文件名即 URL slug；首页内容在 `content/_index.md`；关于页 `content/about.md`；搜索页 `content/search.md`
- 构建：`npm run build`（hugo + Pagefind 搜索索引，输出 `public/`，需要站点根可解析 tailwindcss CLI）；开发：`npm run dev`（hugo server）
- 站点配置在 `hugo.toml`（`[params]`、`[[menus.main]]`、`[markup]`、`[security.exec]` 放行 tailwindcss）
- `layouts/` 是站点级主题覆盖：`baseof.html`/`home.html`/`about/single.html`/`partials/footer.html` 修复主题兼容性并定制首页/关于页/页脚，勿删
- 不要直接改 `themes/` 里的文件（submodule），覆盖放站点 `layouts/`
- 主题文档见 `themes/void/README.md`