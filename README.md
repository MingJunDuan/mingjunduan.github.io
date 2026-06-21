## MingJunDuan的博客

基于 Jekyll + GitHub Pages 搭建的个人技术博客。

## 本地启动

```bash
cd mingjunduan.github.io
bundle exec jekyll serve
```

访问 http://localhost:4000 预览。

---

## 功能

| 功能 | 说明 |
|------|------|
| 侧边栏目录 | 左侧固定，列出全部文章标题，点击跳转 |
| 分页 | 每页 5 篇，底部显示页码 |
| Giscus 评论 | 基于 GitHub Discussions，需登录 GitHub 账号评论 |
| 不蒜子统计 | 顶栏显示全站访问量，文章底部显示每篇阅读量 |
| 代码块复制 | 鼠标移入代码块显示复制按钮，点击复制到剪贴板 |
| 回到顶部 | 滚动超过一屏后右下角出现 ↑ 按钮，点击平滑回顶部 |

---

## 添加新博客

只需要创建文件、写 Frontmatter、写正文、提交推送 4 步，**上述功能全部自动生效**。

### 第 1 步：创建文件

在 `_posts/` 目录下新建 Markdown 文件，文件名格式必须是：`YYYY-MM-DD-标题.md`

```
_posts/2026-06-21-新文章的标题.md
```

### 第 2 步：编写 Frontmatter

文件头部写入以下内容（layout、title、author 三行必须有）：

```yaml
---
layout: post
title: "你的文章标题"
author: "MingJunDuan"
---
```

### 第 3 步：用 Markdown 编写正文

```markdown
# 开头

正文内容，支持标准 Markdown 语法...

## 小标题

内容...

- 列表项
- 列表项

![图片描述](https://github.com/MingJunDuan/mingjunduan.github.io/raw/main/images/xxx.png)
```

### 第 4 步：本地预览 & 发布

```bash
# === 本地预览 ===
bundle exec jekyll serve
# 打开 http://localhost:4000 检查效果

# === 提交 & 发布 ===
git add _posts/2026-06-21-新文章的标题.md
git commit -m "新文章：XXX"
git push origin main
```

### 功能如何自动生效

| 功能 | 原理 | 操作 |
|------|------|------|
| **侧边栏** | 模板遍历 `site.posts` 全部文章，新文章自动出现在列表顶部 | 无需操作 |
| **分页** | `jekyll-paginate` 按每页 5 篇自动计算总页数 | 无需操作 |
| **评论** | Giscus 按 URL pathname 自动匹配 Discussion，新文章首次有人评论时自动创建 | 无需操作 |
| **阅读量** | 不蒜子按页面 URL 独立计数，新页面首次访问时自动开始统计 | 无需操作 |

Push 后 GitHub Pages 自动构建，1-2 分钟后在 https://mingjunduan.github.io 即可看到新文章。
