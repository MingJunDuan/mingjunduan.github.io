# CLAUDE.md — MingJunDuan 的博客

## 项目定位

基于 **Jekyll 3.8 + GitHub Pages** 的个人技术博客。主题 "Rain" 是从零手写的简洁 Jekyll 主题，非第三方模板。仓库 `MingJunDuan/mingjunduan.github.io`，发布地址 `https://mingjunduan.github.io`。

## 技术栈

| 层 | 技术 |
|---|---|
| 静态生成 | Jekyll 3.8, kramdown (Markdown) |
| 样式 | SCSS (`_sass/rain/`)，编译为 `assets/main.css` |
| 模板 | Liquid (`_layouts/`, `_includes/`) |
| 评论 | Giscus (GitHub Discussions) |
| 统计 | 不蒜子 (Busuanzi) |
| 分页 | jekyll-paginate (每页 5 篇) |
| 图片压缩 | ImgBot (自动压缩) + GitHub Actions (自动合并 PR) |
| 部署 | GitHub Pages (push main 自动构建) |

## 目录结构

```
├── _config.yml          # Jekyll 配置（站点、分页、Giscus、社交链接）
├── _layouts/
│   ├── default.html     # 全局框架：顶栏 + 侧边栏 + 内容区 + 页脚 + 回到顶部 + 代码复制
│   ├── home.html        # 首页（文章列表 + 分页）
│   └── post.html        # 文章详情（标题、正文、阅读量、相关文章、Giscus 评论）
├── _includes/
│   └── head.html        # <head>：meta、CSS、Font Awesome、favicon、不蒜子脚本
├── _sass/rain/
│   ├── rain.scss         # 入口，import 下面所有文件
│   ├── _variables.scss   # 颜色、字体变量
│   ├── _base.scss        # 基础重置
│   ├── _layout.scss      # 顶栏、wrapper(flex)、侧边栏、内容区、响应式
│   ├── _sidebar.scss     # 文章列表样式
│   ├── _code.scss        # 代码块 + 复制按钮样式
│   ├── _post.scss        # 文章页样式
│   ├── _syntax.scss      # 代码高亮
│   ├── _backtotop.scss   # 回到顶部按钮
│   ├── _pagination.scss  # 分页样式
│   └── _catalogue.scss   # 首页文章卡片样式
├── _posts/               # Markdown 博客文章（文件名 YYYY-MM-DD-标题.md）
├── _site/                # Jekyll 构建输出（gitignore）
├── assets/               # 编译后的 CSS、favicon
├── images/               # 博客图片（引用路径 /images/xxx.png）
├── index.html            # 首页入口
├── rain.gemspec          # 主题 gem 定义（Rain 0.1.0）
├── Gemfile               # Ruby 依赖（源 ruby-china）
├── .github/workflows/
│   └── auto-merge-imgbot.yml  # ImgBot PR 自动合并
└── README.md             # 项目说明 + 新博客操作步骤
```

## 核心架构

### 布局链
```
post.html  →  default.html  →  head.html
  (extends)      (extends)       (include)

home.html  →  default.html  →  head.html
```

`default.html` 是唯一的全局框架，所有页面通过 Frontmatter `layout: default` 继承它。`{{ content }}` 被替换为子布局的 HTML。

### 左侧边栏
- 在 `default.html` 中用 `{% for post in site.posts %}` 遍历全部文章生成链接列表
- 固定在 260px 宽度，移动端（≤768px）变为顶部横向排列
- 新文章创建后**自动出现**，无需任何配置

### 布局结构 (default.html)
```
┌──────────────────────────────┐
│  顶栏：博客标题 + 社交图标 + 全站UV  │
├────────┬─────────────────────┤
│ 侧边栏  │  内容区              │
│ 文章列表 │  {{ content }}     │
│        │  页脚 ©             │
└────────┴─────────────────────┘
        右下角：回到顶部 ↑（fixed）
```

### 代码块复制
- 在 `default.html` 底部用纯 JS 实现
- 给所有 `<pre>` 包裹 `<div class="code-block">` 并注入复制按钮
- 点击用 `navigator.clipboard.writeText()` 复制，2 秒后恢复
- 样式在 `_code.scss`：按钮默认 `opacity: 0`，hover 代码块时显示

### 回到顶部
- 同样在 `default.html` 底部 JS 实现
- 滚动超过 400px 显示，平滑滚动到顶部

### Giscus 评论
- 在 `post.html` 底部加载
- 配置在 `_config.yml` 的 `giscus` 字段
- `mapping: pathname` 按 URL 路径匹配 Discussion，新文章首次有人评论时自动创建

### 不蒜子统计
- 全站访问量：`default.html` 顶栏显示 `busuanzi_value_site_uv`
- 文章阅读量：`post.html` 底部显示 `busuanzi_value_page_pv`
- 脚本在 `head.html` 中异步加载

### 分页
- `jekyll-paginate` 插件，每页 5 篇
- `home.html` 用 `paginator.posts` 遍历当前页文章
- 底部显示全部页码 + 上一页/下一页链接

### 图片压缩 & 自动合并
- 安装 ImgBot 后，push 图片到 `images/` 目录时自动提 PR 压缩
- `.github/workflows/auto-merge-imgbot.yml` 检测到 PR 作者为 `imgbot` 时，自动 `gh pr merge --auto --squash`
- 全程零操作：push 图片 → ImgBot 提 PR → Actions 自动合并

## 日常操作

### 添加新博客
1. 在 `_posts/` 下创建 `YYYY-MM-DD-标题.md`
2. 写 Frontmatter（`layout: post`, `title`, `author` 三行必须有）
3. 写 Markdown 正文
4. `bundle exec jekyll serve` 本地预览 → `git commit && git push`

所有功能（侧边栏、分页、评论、阅读量）**零配置自动生效**。

### 修改样式
- 编辑 `_sass/rain/_xxx.scss`，Jekyll 自动重编译 `assets/main.css`
- 颜色/字体变量在 `_variables.scss`

### 修改布局
- 全局框架：`_layouts/default.html`
- 首页：`_layouts/home.html`
- 文章页：`_layouts/post.html`
- `<head>`：`_includes/head.html`

### 本地启动
```bash
bundle exec jekyll serve
# → http://localhost:4000
```

## Git 状态（截至 2026-06-22）

本地领先 origin/main 10 个 commit（含 ImgBot、CLAUDE.md 等），`git push origin main` 推送全部。

## 注意事项

- **Jekyll 版本**：`~> 3.8.3`，非 Jekyll 4.x，API 有差异
- **Gem 源**：使用 `gems.ruby-china.com`（国内镜像），非 rubygems.org
- **_site/**：构建产物，已 gitignore，不要手动编辑
- **.sass-cache/**：SCSS 编译缓存，已 gitignore
- **图片引用路径**：`/images/xxx.png`（不是相对路径），GitHub Pages 上完整 URL 为 `https://github.com/MingJunDuan/mingjunduan.github.io/raw/main/images/xxx.png`
- **Permalink**：`/:year-:month-:day/:title`，如 `/2026-06-21/文章标题`
- **jekyll-paginate**：官方插件，GitHub Pages 原生支持，无需额外配置
- **Rain 主题**：此主题是这个项目的私有实现，不是发布的 Ruby gem，所有源码在 `_sass/` 和 `_layouts/` 中直接修改
- **Git push 报错**：如果 push 时报 `workflow scope` 错误，说明 Personal Access Token 缺少 `workflow` 权限。到 https://github.com/settings/tokens 给 token 勾选 `workflow` scope 即可
