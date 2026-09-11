---
title: "欢迎来到我的博客"
date: 2026-08-01
draft: false
description: "第一篇文章：这里会写什么，以及怎么发一篇新文章。"
categories: ["随笔"]
tags: ["开始", "博客"]
---

这是这个博客的第一篇文章，欢迎你 👋

<!--more-->

## 这里会写什么

- **技术笔记**：踩过的坑、用过的工具、读过的代码
- **读书笔记**：看过的书，和一些零散的想法
- **生活随笔**：随手记点什么

## 怎么发一篇新文章

```bash
# 1. 新建一篇文章（文件名会成为网址的一部分，建议用英文或拼音）
hugo new content posts/my-second-post.md

# 2. 用编辑器打开 content/posts/my-second-post.md，把 draft 改成 false，然后写正文

# 3. 本地预览
hugo server -D

# 4. 推送到 GitHub，Cloudflare Pages 会自动重新构建
git add . && git commit -m "new post" && git push
```

## 文章开头的那段文字

在正文里插入一行 `<!--more-->`，它上面的内容就会成为列表页的摘要，下面的内容要点击进来才能看到。