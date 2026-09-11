---
title: "第24讲：Git项目管理与获取"
date: 2026-09-11T09:14:00+08:00
draft: false
description: "2005年，Linux 内核社区使用的商业版本控制工具 BitKeeper 收回了免费使用权。Linus Torvalds 一怒之下，花了大约两周时间用…"
series: ["Linux 入门"]
series_order: 13
categories: ["技术笔记"]
tags: ["Linux", "Git"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P25)
> **标题**：第24讲 — Git项目管理与获取
> **时长**：12分29秒
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. Git简介

### 1.1 Git 的诞生历史

2005年，Linux 内核社区使用的商业版本控制工具 BitKeeper 收回了免费使用权。Linus Torvalds 一怒之下，花了大约两周时间用 C 语言写出了 Git 的第一个版本，用于管理 Linux 内核这个当时全球最大的开源项目。

为什么叫 Git？Linus 自己解释说："I'm an egotistical bastard, so I name all my projects after myself. First 'Linux', now 'git'."（Git 在英式俚语里意为"饭桶、傻瓜"）

> **核心事实**：Git 由 Linus Torvalds 于 2005 年创建，初衷是为 Linux 内核开发提供一套高速、分布式的版本控制系统。

### 1.2 Git 的优势

| 特性 | 说明 |
|------|------|
| **分布式** | 每个开发者电脑上都有完整的代码仓库和历史，不依赖中央服务器 |
| **速度快** | 大部分操作在本地完成，无需网络，毫秒级响应 |
| **数据完整性** | 所有文件通过 SHA-1 哈希校验，内容一旦存储几乎不可能被篡改 |
| **分支轻量** | 创建、切换、合并分支非常快，鼓励频繁使用分支 |
| **暂存区设计** | 可精确控制哪些改动进入下一次提交 |
| **开源免费** | GPL v2 协议，完全自由 |

### 1.3 Git 的三个区域

Git 的工作流程围绕三个逻辑区域展开：

```
工作区 (Working Directory)
    |
    |  git add
    v
暂存区 (Staging Area / Index)
    |
    |  git commit
    v
本地仓库 (Local Repository)
    |
    |  git push  (到远程)
    v
远程仓库 (Remote Repository)
```

| 区域 | 说明 | 对应操作 |
|------|------|----------|
| **工作区** | 你正在编辑的目录，肉眼可见的文件 | 创建/编辑/删除文件 |
| **暂存区** | 临时存放即将提交的改动 | `git add` 将文件加入暂存区 |
| **本地仓库** | 存储完整版本历史（`.git` 目录） | `git commit` 将暂存内容提交到仓库 |

### 1.4 Git 的四种文件状态

一个文件在 Git 管理下的生命周期：

```
未跟踪 (Untracked)
    |
    |  git add
    v
已暂存 (Staged)
    |
    |  git commit
    v
已提交 (Committed)
    |
    |  修改文件
    v
已修改 (Modified)
    |
    |  git add
    v
已暂存 (Staged)  ...（循环）
```

| 状态 | 含义 | 对应场景 |
|------|------|----------|
| **未跟踪 (Untracked)** | Git 不知道这个文件的存在 | 新建了一个文件，还没执行过 `git add` |
| **已暂存 (Staged)** | 文件已加入暂存区，等待提交 | 执行了 `git add` 之后 |
| **已修改 (Modified)** | 已跟踪的文件被改动了，但未暂存 | 修改了已提交过的文件，还没 `git add` |
| **已提交 (Committed)** | 文件已安全存入本地仓库 | 执行了 `git commit` 之后 |

### 1.5 本地仓库 vs 远程仓库

| 类型 | 说明 | 特点 |
|------|------|------|
| **本地仓库** | 在自己电脑上管理项目 | 只能自己使用，别人无法访问 |
| **远程仓库** | 托管在服务器上 | 团队共享，多人协作 |

### 1.6 GitHub / Gitee

- **GitHub**（国外）：全球最大的代码托管平台，开源项目首选
- **Gitee（码云）**（国内）：国内代码托管平台，访问速度更快

两者本质都是用 Git 工具搭建的**代码托管服务器**，功能类似，都是基于 Git 协议的 Web 界面。

---

## 2. 安装Git

### 2.1 Windows系统

从 Git 官网下载安装包：
```
https://git-scm.com/
```
双击安装，按默认配置一路 Next 即可。安装完成后右键菜单里会出现 "Git Bash Here"，这是一个类 Linux 的终端环境。

安装完成后，打开 Git Bash，验证安装：
```bash
git --version
# 输出示例：git version 2.42.0.windows.1
```

### 2.2 Ubuntu/Debian系统

```bash
# 1. 更新软件源
sudo apt update

# 2. 安装Git
sudo apt install git
```

验证安装：
```bash
git --version
# 输出示例：git version 2.40.0
```

---

## 3. 配置Git（首次使用必做）

安装 Git 后，首先要配置你的身份信息。这些信息会记录在每一次提交中，方便追溯。

### 3.1 配置用户名和邮箱

```bash
# 配置全局用户名（所有仓库通用）
git config --global user.name "你的名字"

# 配置全局邮箱
git config --global user.email "your_email@example.com"

# 示例
git config --global user.name "张三"
git config --global user.email "zhangsan@foxmail.com"
```

> **注意**：`--global` 表示全局配置，对当前用户的所有仓库生效。去掉 `--global` 则只对当前仓库有效。

### 3.2 配置默认编辑器

Git 需要你输入提交信息时，会打开一个文本编辑器。可以自定义：

```bash
# 设置为 vim（推荐，Linux 环境默认）
git config --global core.editor vim

# 设置为 nano
git config --global core.editor nano

# Windows 下设置为 VS Code
git config --global core.editor "code --wait"

# Windows 下设置为 Notepad++
git config --global core.editor "'C:/Program Files/Notepad++/notepad++.exe' -multiInst -notabbar -nosession -noPlugin"
```

### 3.3 查看当前配置

```bash
# 查看所有配置项
git config --list

# 查看全局配置
git config --global --list

# 查看某个具体配置项的值
git config user.name
git config user.email
```

配置信息存储在三个层级（优先级从高到低）：

| 层级 | 作用范围 | 配置文件位置 |
|------|----------|-------------|
| **local** | 当前仓库 | `.git/config` |
| **global** | 当前用户所有仓库 | `~/.gitconfig` (C:\Users\用户名\.gitconfig) |
| **system** | 本机所有用户 | `/etc/gitconfig` |

### 3.4 配置命令别名（提升效率）

```bash
# 设置别名
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --all"

# 使用别名
git st       # 等价于 git status
git co main  # 等价于 git checkout main
git lg       # 等价于 git log --oneline --graph --all
```

---

## 4. Git 基础操作

### 4.1 初始化仓库 — git init

在任何一个目录下，执行 `git init` 即可将该目录变成 Git 管理的仓库：

```bash
# 创建新目录
mkdir my_project
cd my_project

# 初始化 Git 仓库
git init
# 输出：Initialized empty Git repository in /path/to/my_project/.git/

# 查看生成的 .git 隐藏目录
ls -la
# 看到 .git 目录即表示初始化成功
```

执行 `git init` 后，该目录下会生成一个 `.git` 隐藏目录，里面存储了 Git 的所有版本控制数据。**不要手动修改 `.git` 目录下的任何内容。**

### 4.2 暂存文件 — git add

```bash
# 暂存单个文件
git add filename.c

# 暂存多个指定文件
git add file1.c file2.h

# 暂存当前目录下所有改动
git add .

# 暂存整个仓库的所有改动
git add -A   # 或 git add --all
```

常见场景演示：

```bash
# 1. 创建一个新文件
echo "# My Project" > README.md

# 2. 查看状态（此时 README.md 是"未跟踪"状态）
git status

# 3. 暂存文件
git add README.md

# 4. 再次查看状态（README.md 变为"已暂存"状态）
git status
```

### 4.3 提交 — git commit

将暂存区的内容永久保存到本地仓库：

```bash
# 提交并附带简要说明（最常用）
git commit -m "添加 README 文件"

# 提交时自动暂存所有已跟踪文件的修改（跳过 git add）
git commit -am "修改了 main.c 的初始化逻辑"

# 提交并打开编辑器，编写详细的提交说明
git commit
```

> **提交信息规范建议**：用简短的一句话描述做了什么改动。中文或英文均可，但要清晰、有意义。避免写"改了点东西"、"fix"等模糊信息。

### 4.4 查看状态 — git status

`git status` 是日常开发中最常用的命令之一，告诉你当前仓库的状态：

```bash
git status
```

输出示例：
```
On branch main
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   README.md

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
        modified:   main.c

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        temp.log
```

`git status` 的简洁模式：
```bash
git status -s   # 或 git status --short

# 输出：
# A  README.md      (A = Added，新增已暂存)
#  M main.c           (右边M = Modified，已修改但未暂存)
# ?? temp.log         (?? = Untracked，未跟踪)
```

### 4.5 查看提交历史 — git log

```bash
# 查看完整提交日志
git log

# 简洁模式，每次提交一行
git log --oneline

# 图形化显示分支合并历史
git log --oneline --graph

# 显示所有分支的历史
git log --oneline --graph --all

# 限制显示最近 N 条
git log --oneline -5
```

> **小技巧**：可以设置别名 `git config --global alias.lg "log --oneline --graph --all"`，之后只需敲 `git lg` 即可。

### 4.6 查看差异 — git diff

```bash
# 查看工作区与暂存区的差异（即还没 git add 的改动）
git diff

# 查看暂存区与最新提交的差异（即 git add 了还没 git commit 的）
git diff --staged   # 或 git diff --cached

# 查看工作区与最新提交的差异（跳过暂存区）
git diff HEAD

# 查看两个提交之间的差异
git diff commit1 commit2

# 查看某个文件的改动
git diff -- filename.c
```

---

## 5. 下载项目资料 — git clone

### 5.1 克隆远程仓库

```bash
# 创建一个目录存放项目
mkdir my_project
cd my_project

# 克隆远程仓库
git clone https://github.com/用户名/仓库名.git
```

克隆完成后，会生成一个与仓库同名的子目录，里面包含了完整的代码和历史记录。

常用选项：
```bash
# 克隆到指定目录名（而非默认的仓库名）
git clone https://github.com/user/repo.git my-custom-dir

# 只克隆最近一次提交的历史（深度为1），加快下载速度
git clone --depth 1 https://github.com/user/repo.git

# 克隆指定分支
git clone -b develop https://github.com/user/repo.git
```

### 5.2 GitHub vs Gitee 速度对比

```bash
# GitHub 克隆（国外服务器，速度可能较慢）
git clone https://github.com/xxx/project.git

# Gitee 克隆（国内服务器，速度快）
git clone https://gitee.com/xxx/project.git
```

> **建议**：如果 GitHub 克隆速度慢，可以换用 Gitee 上的镜像仓库。

### 5.3 直接下载zip包

在 GitHub/Gitee 仓库页面点击 **Download ZIP** 可直接下载源码压缩包。

> **缺点**：没有版本历史记录，无法跟踪修改，无法与远程仓库同步更新。

---

## 6. 仓库更新 — git pull 与 git fetch

### 6.1 git pull（拉取并合并）

当远程仓库有新内容时，使用以下命令拉取最新版本：

```bash
git pull
```

如果已经是最新版本，会提示：
```
Already up to date.
```

`git pull` 实际上是两个操作的组合：
```
git pull = git fetch + git merge
```
即：先从远程下载新数据，再将远程分支合并到当前本地分支。

### 6.2 git fetch（仅拉取，不合并）

```bash
# 从远程仓库下载所有更新，但不合并
git fetch

# 从指定的远程仓库下载
git fetch origin

# 下载后可以查看远程分支的变化
git log origin/main   # 查看远程 main 分支的提交历史
```

### 6.3 pull vs fetch 对比

| 命令 | 行为 | 使用场景 |
|------|------|----------|
| `git pull` | 下载远程数据 + 自动合并 | 信任远程更新，直接同步 |
| `git fetch` | 只下载远程数据，不合并 | 想先查看远程有什么变化，再决定要不要合并 |

推荐工作流：
```bash
# 1. 先查看远程变化（安全）
git fetch origin

# 2. 查看远程分支和本地的差异
git log main..origin/main   # 远程比本地多了哪些提交

# 3. 确认无误后再合并
git merge origin/main
```

---

## 7. 远程仓库管理

### 7.1 关联远程仓库 — git remote add

如果你有一个本地仓库，想把它推送到远程，需要先关联：

```bash
# 为本地仓库添加一个远程仓库，取名为 origin（约定俗成的名称）
git remote add origin https://github.com/用户名/仓库名.git

# 示例
git remote add origin https://github.com/zhangsan/my-linux-driver.git
```

### 7.2 查看远程仓库 — git remote -v

```bash
# 查看已关联的远程仓库信息
git remote -v

# 输出示例：
# origin  https://github.com/zhangsan/my-linux-driver.git (fetch)
# origin  https://github.com/zhangsan/my-linux-driver.git (push)
```

### 7.3 推送代码 — git push

```bash
# 将本地 main 分支推送到远程 origin
git push origin main

# 首次推送（设置上游分支，之后可以简写为 git push）
git push -u origin main

# 设置上游后，之后的推送只需：
git push
```

### 7.4 删除和重命名远程仓库

```bash
# 重命名远程仓库（将 origin 改为 upstream）
git remote rename origin upstream

# 删除远程仓库关联
git remote remove origin
```

---

## 8. 分支管理

分支是 Git 最强大的特性之一。你可以在不影响主线代码的情况下开发新功能、修复 bug，等确认无误后再合并回去。

### 8.1 分支概念示意图

```
    main (主分支)
      |
      *---*---*---*---*---* (最新提交)
          |
          *---*---* (feature-login 功能分支)
          |
          *---* (fix-bug-101 修复分支)
```

### 8.2 查看分支 — git branch

```bash
# 查看本地分支（当前分支前有 * 号标记）
git branch

# 查看远程分支
git branch -r

# 查看所有分支（本地 + 远程）
git branch -a
```

### 8.3 创建并切换分支

```bash
# 方法一：git checkout（传统方式）
git checkout -b feature-login
# 等价于：
# git branch feature-login    创建分支
# git checkout feature-login   切换分支

# 方法二：git switch（Git 2.23+，语义更清晰，推荐）
git switch -c feature-login
# 等价于：
# git switch feature-login    切换到已有分支
```

切换回主分支：
```bash
git checkout main      # 传统方式
git switch main        # 新方式（推荐）
```

### 8.4 合并分支 — git merge

```bash
# 1. 先切换到目标分支（比如 main）
git switch main

# 2. 将 feature-login 分支合并到当前分支
git merge feature-login

# 合并过程可能产生冲突，需要手动解决后：
git add .
git commit -m "解决合并冲突"
```

分支合并的三种情况：

| 情况 | 说明 | 合并方式 |
|------|------|----------|
| **快进合并 (Fast-forward)** | main 没有新提交，直接把指针前移 | 自动完成，无额外提交 |
| **三方合并 (3-way merge)** | 两个分支都有各自的提交 | 自动创建一次合并提交 |
| **冲突 (Conflict)** | 两个分支修改了同一文件的同一位置 | 需手动解决冲突再提交 |

### 8.5 删除分支

```bash
# 删除已合并的分支
git branch -d feature-login

# 强制删除（即使未合并）
git branch -D feature-login

# 删除远程分支
git push origin --delete feature-login
```

### 8.6 分支管理最佳实践

```
main
  |
  +-- develop (开发分支，日常开发在这里)
  |     |
  |     +-- feature-a (功能分支)
  |     +-- feature-b (功能分支)
  |     +-- fix-issue-42 (修复分支)
  |
  +-- release (发布分支)
```

常见分支命名：
| 分支类型 | 命名示例 | 用途 |
|----------|----------|------|
| 主分支 | `main` / `master` | 稳定版本，随时可发布 |
| 开发分支 | `develop` | 日常开发集成分支 |
| 功能分支 | `feature/xxx` | 开发新功能 |
| 修复分支 | `fix/xxx` | 修复 bug |
| 发布分支 | `release/v1.0` | 准备发布版本 |

---

## 9. 撤销操作

### 9.1 撤销工作区修改 — git restore

```bash
# 撤销某个文件的工作区修改（回到最近一次 git add 或 git commit 的状态）
git restore filename.c

# 撤销所有工作区修改
git restore .

# 从暂存区移出（即"撤销 git add"，但保留工作区的修改）
git restore --staged filename.c
```

### 9.2 撤销提交 — git reset

`git reset` 用于回退到历史某个版本，有三种模式：

```bash
# 查看提交历史，找到要回退到的 commit 哈希
git log --oneline

# --soft：回退提交，但保留暂存区和工作区的修改
# （相当于"撤销 commit，不撤销 add"）
git reset --soft HEAD~1       # 回退最近 1 次提交
git reset --soft abc1234      # 回退到指定提交

# --mixed（默认）：回退提交和暂存区，但保留工作区的修改
# （相当于"撤销 commit 和 add，但不改文件"）
git reset HEAD~1
git reset --mixed abc1234

# --hard：彻底回退，丢弃所有修改（危险操作！）
git reset --hard HEAD~1
git reset --hard abc1234
```

三种 reset 模式对比：

| 模式 | 回退提交 | 回退暂存区 | 回退工作区 | 危险程度 |
|------|:--------:|:----------:|:----------:|:--------:|
| `--soft` | 是 | 否 | 否 | 低 |
| `--mixed` (默认) | 是 | 是 | 否 | 中 |
| `--hard` | 是 | 是 | 是 | **高** |

### 9.3 安全撤销 — git revert

`git revert` 创建一次新的提交来"反做"某次历史提交，不会修改历史。多人协作时强烈推荐用 revert 而非 reset：

```bash
# 撤销最近一次提交（生成一次新提交）
git revert HEAD

# 撤销指定的提交
git revert abc1234
```

| 操作 | 适用场景 | 是否改写历史 |
|------|----------|:------------:|
| `git reset` | 本地的提交，还没推送 | 是 |
| `git revert` | 已推送到远程的提交，多人协作 | 否 |

---

## 10. 忽略文件 — .gitignore

### 10.1 什么是 .gitignore

某些文件不需要纳入版本控制（编译产物、临时文件、密码配置文件等），可以把这些文件的规则写入项目根目录下的 `.gitignore` 文件，Git 会自动忽略它们。

### 10.2 .gitignore 语法

```bash
# 注释：以 # 开头
# ===============================

# 忽略特定文件
secret.txt

# 忽略所有 .o 文件（编译中间产物）
*.o

# 忽略所有 .log 文件
*.log

# 忽略特定目录（路径后的 / 表示目录）
build/
temp/
output/

# 忽略所有 .a 文件，但 libfoo.a 除外
*.a
!libfoo.a

# 只忽略根目录下的 TODO 文件，不忽略子目录的
/TODO

# 忽略 doc 目录下所有 .pdf 文件
doc/**/*.pdf
```

### 10.3 嵌入式开发常见 .gitignore 示例

```bash
# 编译产物
*.o
*.ko
*.mod.c
*.mod
*.symvers
*.order
*.bin
*.elf
*.hex

# 编辑器/IDE
.vscode/
.idea/
*.swp
*.swo
*~

# 系统文件
.DS_Store
Thumbs.db

# 编译输出目录
build/
output/

# 密码/密钥（绝对不能提交）
*.key
*.pem
config/private.conf
```

> **注意**：`.gitignore` 只能忽略**未跟踪**的文件。如果文件已经被 Git 跟踪，需要先执行 `git rm --cached 文件名` 移除跟踪。

---

## 11. 实战工作流

### 11.1 单人开发基本流程

```
  新建仓库
     |
     v
  git init (或 git clone)
     |
     v
  写代码，修改文件
     |
     v
  git add .          <!-- 暂存改动 -->
     |
     v
  git commit -m "..." <!-- 提交到本地仓库 -->
     |
     v
  git push            <!-- 推送到远程仓库 -->
     |
     (循环)
```

示例：
```bash
# 初始化或克隆
git init   # 或 git clone xxx

# 写一些代码后...
git add main.c
git commit -m "实现 GPIO 初始化函数"

# 继续修改...
git add -A
git commit -m "添加 UART 驱动框架"

# 查看历史
git log --oneline

# 推送到远程（如果有的话）
git push
```

### 11.2 协作开发基本流程（Fork + PR 模型）

这是 GitHub/Gitee 上最常见的开源协作模式：

```
1. Fork 原仓库（在 GitHub 网页上操作，复制一份到自己名下）

2. Clone 自己的仓库到本地
   git clone https://github.com/你的用户名/仓库名.git
   cd 仓库名

3. 添加上游仓库（方便同步更新）
   git remote add upstream https://github.com/原作者/仓库名.git
   git remote -v     <!-- 确认 origin 和 upstream 都配好了 -->

4. 创建功能分支
   git switch -c feature-my-uart

5. 写代码 + 提交
   git add .
   git commit -m "添加 UART DMA 模式驱动"

6. 推送到自己的远程仓库
   git push -u origin feature-my-uart

7. 在 GitHub/Gitee 网页上创建 Pull Request (PR)
   从 "你的仓库/feature-my-uart" 向 "原仓库/main" 发起 PR

8. 等待代码审查，根据反馈修改后继续 push，PR 会自动更新

9. 合并后清理本地分支
   git switch main
   git branch -d feature-my-uart
```

### 11.3 同步上游更新

在协作过程中，原仓库可能有新的提交，需要同步到本地：

```bash
# 1. 从上游仓库下载更新
git fetch upstream

# 2. 切回 main 分支
git switch main

# 3. 合并上游 main
git merge upstream/main

# 4. 推送到自己的 origin（可选）
git push origin main
```

---

## 12. 项目文件结构

克隆完成后，进入项目目录查看：

```bash
ls
cd 项目目录
```

重点关注 `basecode` 目录，里面包含：
- 工具配置文件
- 源代码文件

---

## 13. Git学习资源推荐

| 资源 | 说明 |
|------|------|
| [廖雪峰的Git教程](https://www.liaoxuefeng.com/wiki/896043488029600) | 通俗易懂，案例丰富 |
| [Git官方中文参考手册](https://git-scm.com/book/zh/v2) | 内容详细全面，适合深入学习 |

---

## 14. 替代方案：通过野火大学堂下载

如果不习惯使用 Git，也可通过**野火大学堂**（Embedfire University）工具下载项目资料：

1. 打开野火大学堂
2. 等待资料同步更新
3. 导航：**Linux系列开发 -> i.MX6ULL开发 -> 基本资料**
4. 找到项目资料进行下载

下载后可在 Windows 本地找到 `basecode` 目录中的文件。

---

## 本讲总结

### 核心命令速查

| 分类 | 命令 | 说明 |
|------|------|------|
| **配置** | `git config --global user.name "..."` | 设置用户名 |
| **配置** | `git config --global user.email "..."` | 设置邮箱 |
| **配置** | `git config --list` | 查看所有配置 |
| **初始化** | `git init` | 初始化本地仓库 |
| **暂存** | `git add .` / `git add filename` | 将改动加入暂存区 |
| **提交** | `git commit -m "消息"` | 提交到本地仓库 |
| **状态** | `git status` | 查看仓库当前状态 |
| **历史** | `git log --oneline --graph` | 查看提交历史 |
| **差异** | `git diff` | 查看文件改动 |
| **克隆** | `git clone <仓库地址>` | 克隆远程仓库到本地 |
| **拉取** | `git pull` | 拉取远程更新并合并 |
| **拉取** | `git fetch` | 只下载远程更新，不合并 |
| **推送** | `git push` | 推送本地提交到远程 |
| **远程** | `git remote add origin <地址>` | 关联远程仓库 |
| **远程** | `git remote -v` | 查看远程仓库信息 |
| **分支** | `git branch` | 查看分支列表 |
| **分支** | `git switch -c 分支名` | 创建并切换分支 |
| **分支** | `git merge 分支名` | 合并分支 |
| **分支** | `git branch -d 分支名` | 删除分支 |
| **撤销** | `git restore filename` | 撤销工作区修改 |
| **撤销** | `git restore --staged filename` | 取消暂存 |
| **撤销** | `git reset --soft HEAD~1` | 撤销提交（保留改动） |
| **撤销** | `git revert <commit>` | 安全撤销某次提交 |
| **忽略** | `.gitignore` | 配置忽略文件规则 |

### 关键概念

| 概念 | 说明 |
|------|------|
| **三个区域** | 工作区 -> 暂存区 -> 本地仓库 |
| **四种状态** | 未跟踪、已修改、已暂存、已提交 |
| **分布式** | 每台机器都有完整仓库 |
| **GitHub vs Gitee** | GitHub 国外速度快，Gitee 国内速度快 |

---

*笔记生成日期：2026-07-22 | 转写：Whisper tiny | 优化补充：2026-07-23*
