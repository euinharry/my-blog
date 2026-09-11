# 我的博客

一个开箱即用的中文个人博客，基于 **Hugo + PaperMod**，部署在 **Cloudflare Pages**。

自带：分类、标签、标签云、按时间归档、Fuse.js 站内搜索、深色模式、文章目录、代码高亮与一键复制、RSS。

---

## 一、部署到 Cloudflare Pages（三步）

### 第 1 步：把代码推到 GitHub

```bash
git remote add origin https://github.com/<你的用户名>/<仓库名>.git
git branch -M main
git push -u origin main
```

### 第 2 步：在 Cloudflare 里连接仓库

1. 打开 <https://dash.cloudflare.com/>，左侧 **Workers & Pages**
2. 点 **Create** → 选 **Pages** 标签 → **Connect to Git**
3. 授权 GitHub，选中这个博客仓库
4. 构建配置按下面填（**这一步千万别填错**）：

   | 配置项 | 填什么 |
   | --- | --- |
   | Framework preset | `Hugo` |
   | Build command | `hugo --gc --minify` |
   | Build output directory | `public` |
   | Root directory | 留空 |

5. 展开 **Environment variables (advanced)**，添加一条：

   | 变量名 | 值 |
   | --- | --- |
   | `HUGO_VERSION` | `0.166.0` |

   > ⚠️ **这条必须加。** Cloudflare 默认装的 Hugo 是很多年前的老版本，本项目用到了新版本的配置和模板语法，不加这一条会构建失败。

6. 点 **Save and Deploy**，等 1～2 分钟。

### 第 3 步：把网址填回配置

部署成功后会得到一个网址，形如 `https://<项目名>.pages.dev`。

打开 `hugo.toml`，把第一处的 `baseURL` 改成这个网址：

```toml
baseURL = "https://<项目名>.pages.dev/"
```

提交并推送：

```bash
git commit -am "set baseURL" && git push
```

Cloudflare 会重新构建。

> **为什么一定要改？** 页面里的 CSS、图片、链接地址都是根据 `baseURL` 生成的。地址不对，线上会出现「样式全丢、排版错乱」。

### 绑定自己的域名（可选）

在 Pages 项目的 **Custom domains** 里添加你的域名，按提示把域名的 DNS 交给 Cloudflare 托管即可，HTTPS 证书会自动签发。绑定完记得把 `baseURL` 也换成自定义域名。

---

## 二、改成你自己的信息

全站配置都在 **`hugo.toml`** 一个文件里，需要改的地方都带了 `★★★` 注释：

| 改什么 | 在 `hugo.toml` 里找 |
| --- | --- |
| 站点网址 | `baseURL` |
| 博客名（显示在左上角、页脚、浏览器标题） | `title` |
| 首页自我介绍 | `[params.homeInfoParams]` |
| 作者名（显示在文章头上） | `[params]` 下的 `author` |
| 网站描述（SEO 用） | `[params]` 下的 `description` |
| 社交图标 | `[[params.socialIcons]]`，不需要就整段删掉 |
| 每页文章数 | `[pagination]` 下的 `pagerSize` |
| 菜单元 | 文件末尾的 `[[menu.main]]` |

「关于」页面的正文在 `content/about.md`，直接改就行。

### 社交图标支持的名字

`github` `email` `bilibili` `zhihu` `juejin` `douban` `wechat` `qq` `x` `youtube` `telegram` `rss` `stackoverflow` 等等，写错名字图标不会显示。

---

## 三、写一篇新文章

```bash
# 新建（文件名会成为网址的一部分，建议用英文或拼音）
hugo new content posts/my-post.md
```

然后编辑 `content/posts/my-post.md`：

```yaml
---
title: "文章标题"
date: 2026-09-11
draft: false                # true = 草稿，不会发布
description: "列表页显示的摘要"
categories: ["技术笔记"]      # 分类
tags: ["Hugo", "教程"]        # 标签
---

正文从这里开始。

在正文里插入一行 <!--more-->，它上面的内容会作为列表页摘要。
```

写完推送到 GitHub，Cloudflare 会自动重新构建上线。

### 几个容易踩的坑

- **文件名别用中文**，网址会变成一长串 `%E6%8A%80`，不好看也不好分享。
- **记得把 `draft` 改成 `false`**，否则线上看不到。
- **日期别写未来时间**，Hugo 默认不构建「未来」的文章。
- **图片放到 `static/images/`**，用 `/images/xxx.png` 引用，最不容易出错。

---

## 四、本地预览（可选）

本地预览需要先装 Hugo（**extended 版**）：

1. 打开 <https://github.com/gohugoio/hugo/releases>
2. 下载 `hugo_extended_0.166.0_windows-amd64.zip`
3. 解压出 `hugo.exe`，放到任意目录，并把这个目录加进系统环境变量 `PATH`
4. 打开终端验证：`hugo version`，应显示 `... +extended ...`

然后在本目录下执行：

```bash
# 本地预览，改文件浏览器会自动刷新；-D 表示连草稿一起显示
hugo server -D

# 正式构建（和 Cloudflare 上跑的命令一致）
hugo --gc --minify
```

浏览器打开 <http://localhost:1313> 即可。

> 不装 Hugo 也完全不影响线上部署——Cloudflare 会在云端帮你构建。

---

## 五、目录结构

```text
.
├── hugo.toml                 # ★ 全站配置，99% 的修改都在这里
├── content/                  # 你写的文章
│   ├── posts/                #   博客文章（首页只显示这个目录）
│   ├── about.md              #   「关于」页面
│   ├── archives.md           #   「归档」页面
│   └── search.md             #   「搜索」页面
├── assets/css/extended/
│   └── custom.css            # 中文阅读体验微调（字体、行距、宽度）
├── static/                   # 原样拷到网站根目录：favicon、图片
├── themes/PaperMod/          # 主题（已内置在仓库里，无需额外安装）
└── public/                   # 构建产物，不提交到 git
```

主题是**直接内置**在仓库里的（没有用 git submodule），所以 Cloudflare Pages 拉下来就能直接构建，不需要额外的网络请求，也不会出现「主题没拉下来导致构建失败」的问题。

---

## 六、常见问题

**Q：线上样式全丢了，排版乱成一团？**
A：99% 是 `baseURL` 没改对。见「第 3 步」。

**Q：站内搜索搜不出结果？**
A：检查 `hugo.toml` 里 `[outputs]` 的 `home = ["HTML", "RSS", "JSON"]` 有没有被删掉，`JSON` 就是搜索索引。

**Q：Cloudflare 构建失败，日志里说找不到 hugo 或者版本太低？**
A：检查环境变量 `HUGO_VERSION` 是否为 `0.166.0`。

**Q：改了文章但线上没变化？**
A：去 Pages 的 **Deployments** 看构建日志；确认文章的 `draft` 是不是 `true`。

**Q：构建时出现两条 `deprecated` 警告，要紧吗？**
A：不要紧。那是主题模板用了 Hugo 的两个旧字段名，只是提醒，不影响功能。

---

## 附：本项目使用的版本

- Hugo `0.166.0`（extended）
- PaperMod 主题 `master` 分支，commit `d3768854d00ad003b0a8dbdba254ce9224377a01`

> 说明：PaperMod 官方最新的 release 标签是 `v8.0`（2024 年 11 月），它和现在的新版 Hugo 已不兼容，所以这里用的是官方仍在维护的 `master`，并固定到具体 commit，保证构建结果稳定可复现。

favicon 是自动生成的占位图，想换成自己的，直接替换 `static/` 下的
`favicon.ico`、`favicon-16x16.png`、`favicon-32x32.png`、`apple-touch-icon.png`、`safari-pinned-tab.svg` 即可。

---

主题 [PaperMod](https://github.com/adityatelange/hugo-PaperMod) 基于 MIT 协议，许可证见 `themes/PaperMod/LICENSE`。