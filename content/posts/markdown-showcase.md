---
title: "Markdown 语法与主题功能展示"
date: 2026-08-15
draft: false
description: "标题、列表、代码块、表格、引用、目录……一篇文章看全 PaperMod 的排版效果。"
categories: ["技术笔记"]
tags: ["Markdown", "Hugo", "排版"]
---

这篇文章用来展示排版效果：标题、列表、代码块、表格、引用分别长什么样。

<!--more-->

## 二级标题

### 三级标题

#### 四级标题

正文里可以**加粗**、*斜体*、~~删除线~~、`行内代码`，也可以放[链接](https://gohugo.io/)。

## 列表

无序列表：

- 第一项
- 第二项
  - 嵌套一项
  - 嵌套又一项

有序列表：

1. 先这样
2. 再那样
3. 最后这样

## 引用

> 好的排版是看不见的。
> 它只是让你读得更顺一点。

## 代码块

```python
from pathlib import Path

def count_lines(path: Path) -> int:
    """统计文件行数"""
    with path.open(encoding="utf-8") as f:
        return sum(1 for _ in f)

if __name__ == "__main__":
    print(count_lines(Path("hugo.toml")))
```

```toml
baseURL = "https://example.com/"
title = "我的博客"
theme = "PaperMod"
```

## 表格

| 功能 | 说明 | 是否需要配置 |
| --- | --- | --- |
| 分类 | 一级归类，比如「技术笔记」 | 写 front matter 即可 |
| 标签 | 多维度标记 | 写 front matter 即可 |
| 归档 | 按时间列出全部文章 | 已内置 |
| 站内搜索 | 基于 Fuse.js，纯前端 | 已内置 |
| 深色模式 | 右上角按钮切换 | 已内置 |

## 图片

把图片放进 `static/images/` 目录，然后这样引用：

```markdown
![图片说明](/images/example.png)
```

## 目录（TOC）

只要文章里出现了二级标题，右上角就会自动生成目录。方法是在 front matter 里加 `showToc: true`，或者保持站点默认开启。