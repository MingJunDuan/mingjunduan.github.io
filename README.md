## MingJunDuan的博客

基于 Jekyll + GitHub Pages 搭建的个人技术博客。

## 本地启动

```bash
bundle exec jekyll serve
```

访问 http://localhost:4000 预览。

## 添加新博客

### 1. 创建文件

在 `_posts/` 目录下新建 Markdown 文件，文件名格式：`YYYY-MM-DD-标题.md`

```
_posts/2026-06-21-看书读感-《XXX》.md
```

### 2. 编写 Frontmatter

文件头部添加：

```yaml
---
layout: post
title: "你的文章标题"
author: "MingJunDuan"
---
```

### 3. 用 Markdown 编写正文

```markdown
# 开头

正文内容...

## 小标题

...

![图片描述](https://github.com/MingJunDuan/mingjunduan.github.io/raw/main/images/xxx.png)
```

### 4. 本地预览 & 发布

```bash
# 本地预览
bundle exec jekyll serve

# 提交发布
git add _posts/2026-06-21-看书读感-《XXX》.md
git commit -m "新文章：XXX"
git push origin main
```

Push 后 GitHub Pages 自动构建，几分钟后在 `mingjunduan.github.io` 即可看到新文章。
