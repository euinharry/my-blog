---
title: "Hugo 日常使用笔记：目录、命令、发布流程"
date: 2026-08-28
draft: false
description: "记住这几条命令和几个目录，就能一直用下去了。"
categories: ["技术笔记"]
tags: ["Hugo", "教程"]
---

Hugo 上手最大的门槛其实是「目录在哪、命令是什么」。这篇把常用的都列出来。

<!--more-->

## 目录结构

```text
blogs/
├── hugo.toml                 # 全站配置：站名、菜单、搜索……改这里
├── content/                  # 你写的文章都放这里
│   ├── posts/                # 博客文章
│   ├── about.md              # 「关于」页面
│   ├── archives.md           # 「归档」页面
│   └── search.md             # 「搜索」页面
├── assets/css/extended/      # 自定义 CSS（覆盖主题样式）
│   └── custom.css
├── static/                   # 原样拷贝到网站根目录：图片、favicon
│   └── images/
├── themes/PaperMod/          # 主题（已内置在仓库里，不用额外安装）
└── public/                   # 构建产物，git 里不提交
```

## 常用命令

```bash
# 本地预览，改文件后浏览器自动刷新；-D 表示连草稿一起显示
hugo server -D

# 本地预览并允许局域网其它设备访问（手机上看效果）
hugo server -D --bind 0.0.0.0 --baseURL http://192.168.1.10:1313

# 正式构建（Cloudflare Pages 上用的就是这条）
hugo --gc --minify

# 新建文章
hugo new content posts/文章文件名.md

# 查看版本
hugo version
```

## 文章的 front matter

```yaml
---
title: "文章标题"
date: 2026-08-28
draft: false                # true = 草稿，不会发布
description: "列表页显示的摘要"
categories: ["技术笔记"]      # 分类，可以有多个
tags: ["Hugo", "教程"]        # 标签，可以有多个
showToc: true               # 是否显示目录
---
```

## 几个容易踩的坑

1. **文件名别用中文**，虽然能用，但网址会变成一长串编码，不美观也不好分享。建议用拼音或英文。
2. **改完文章记得 `draft: false`**，否则线上看不到。
3. **日期别写成未来时间**，Hugo 默认不构建「未来」的文章。
4. **图片放 `static/` 下**，用 `/images/xxx.png` 这样的绝对路径引用，最不容易出错。