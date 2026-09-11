---
title: "把 Hugo 博客部署到 Cloudflare Pages"
date: 2026-09-05
draft: false
description: "免费、自动 HTTPS、支持自定义域名，推送到 GitHub 就自动发布。"
categories: ["技术笔记"]
tags: ["Cloudflare", "部署", "Hugo"]
---

Cloudflare Pages 是目前对个人博客最友好的免费托管：全球 CDN、自动 HTTPS、每次 `git push` 自动重新构建。

<!--more-->

## 部署三步走

### 1. 把代码推到 GitHub

```bash
git add .
git commit -m "init blog"
git push
```

### 2. 在 Cloudflare Pages 里连接这个仓库

1. 打开 <https://dash.cloudflare.com/>，左侧选 **Workers & Pages** → **Create** → **Pages** → **Connect to Git**
2. 授权 GitHub，选中你的博客仓库
3. 构建配置填：

| 配置项 | 值 |
| --- | --- |
| Framework preset | `Hugo` |
| Build command | `hugo --gc --minify` |
| Build output directory | `public` |

4. 在 **Environment variables** 里加一条：`HUGO_VERSION` = `0.166.0`

5. 点 **Save and Deploy**，等 1～2 分钟

### 3. 改 baseURL

部署完成后你会得到一个网址，类似 `https://my-blog.pages.dev`。把这个网址填回 `hugo.toml` 的 `baseURL`，再提交一次，样式和链接才会完全正确。

## 自定义域名

在 Pages 项目的 **Custom domains** 里添加你的域名，按提示把 DNS 交给 Cloudflare 托管即可，HTTPS 证书会自动签发。

## 常见问题

- **样式全丢了 / 页面排版混乱** → 99% 是 `baseURL` 不对。
- **站内搜索没结果** → 确认 `hugo.toml` 里 `[outputs] home = ["HTML", "RSS", "JSON"]` 没被删掉。
- **改了文章线上没变化** → 去 Pages 的 Deployments 看构建日志，或者确认文章 `draft` 是不是 `true`。
- **构建失败提示找不到 hugo** → 检查 `HUGO_VERSION` 环境变量有没有设置。