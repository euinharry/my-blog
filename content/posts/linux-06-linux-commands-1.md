---
title: "第8讲：使用Linux命令（上）"
date: 2026-09-11T09:21:00+08:00
draft: false
description: "用户输入命令 -> Shell解析命令 -> 调用应用程序"
series: ["Linux 入门"]
series_order: 6
categories: ["技术笔记"]
tags: ["Linux", "命令行"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P9)
> **标题**：第8讲 -- 使用Linux命令（上）
> **时长**：25分43秒
> **转写方式**：Whisper tiny (CPU) 语音识别

---

[TOC]



## 1. 什么是 Shell？

### 1.1 Shell 的本质

> Shell 是 Linux 系统中的一个**应用程序**，功能是作为**用户与Linux内核沟通的桥梁**。

### 1.2 Shell 的工作流程

```
用户输入命令 -> Shell解析命令 -> 调用应用程序
-> 应用程序通过系统调用API使用内核服务
-> 内核执行完毕返回结果 -> Shell显示结果
```

### 1.3 Shell 的三大核心功能

| 功能 | 说明 |
|------|------|
| 1. 接收命令 | 对外接收用户输入的所有命令 |
| 2. 传递执行 | 通过系统调用把应用程序传递给内核运行 |
| 3. 呈现结果 | 将内核的运行结果显示给用户 |

### 1.4 Shell vs 图形化界面

| 对比项 | Shell | 图形化界面 |
|--------|-------|-----------|
| 操作方式 | 键盘输入命令 | 鼠标点击图标 |
| 学习成本 | 较高（需记忆命令） | 较低 |
| 功能覆盖 | 几乎控制所有程序 | 仅限有GUI的程序 |
| 灵活性 | 可通过选项精细控制 | 受限 |

> **结论**：Shell 学习成本虽高，但功能远比图形界面强大。

### 1.5 常见的 Shell 类型

Linux 下有多种 Shell 可供选择，它们的共同点是都能解释执行用户命令，但在交互体验、脚本语法上有差异。

| Shell | 全称 | 特点 |
|-------|------|------|
| **sh** | Bourne Shell | 最早的 Unix Shell，功能基础，几乎所有 Unix/Linux 都自带 |
| **bash** | Bourne Again Shell | sh 的增强版，Linux 默认 Shell，兼容 sh 语法，功能丰富 |
| **zsh** | Z Shell | 兼容 bash，支持更强大的自动补全、主题插件（oh-my-zsh） |
| **fish** | Friendly Interactive Shell | 开箱即用的友好 Shell，语法高亮、自动建议，但不兼容 bash 脚本 |
| **dash** | Debian Almquist Shell | 轻量级 sh 实现，启动快，Debian/Ubuntu 中 /bin/sh 实际指向 dash |
| **csh/tcsh** | C Shell / TENEX C Shell | 语法类似 C 语言，主要在 BSD 系统使用 |

#### 查看当前使用的 Shell

```bash
# 方法1：查看环境变量 SHELL（显示当前用户的默认 Shell）
echo $SHELL
# 输出示例：/bin/bash

# 方法2：查看当前进程的 Shell
ps -p $$
# 输出示例：
#   PID TTY          TIME CMD
#  1234 pts/0    00:00:00 bash

# 方法3：查看 /etc/passwd 中当前用户配置的 Shell
grep "^$(whoami):" /etc/passwd | cut -d: -f7
```

#### 查看系统支持的所有 Shell

```bash
cat /etc/shells
# 输出示例：
# /bin/sh
# /bin/bash
# /usr/bin/bash
# /bin/rbash
# /bin/dash
# /usr/bin/dash
# /usr/bin/tmux
# /usr/bin/screen
```

只有列在 `/etc/shells` 中的程序才能被 `chsh`（change shell）命令设为用户的登录 Shell。

### 1.6 Bash 常用操作技巧

#### Tab 自动补全

按一次 Tab 键：如果唯一匹配，自动补全；如果不唯一，没有任何反应。
按两次 Tab 键：列出所有可能的匹配项。

```bash
# 示例：输入 cd /h 然后按 Tab，自动补全为 cd /home/
# 示例：输入 ls D 然后按两次 Tab，列出所有以 D 开头的文件
Desktop/  Documents/  Downloads/
```

Tab 补全不仅适用于文件名和目录，也可以补全命令名、变量名、用户名等。

#### 历史命令

```bash
history          # 查看所有历史命令（默认保存最近1000条）
history 10       # 查看最近10条历史命令

!n               # 执行历史记录中编号为 n 的命令（如 !520 执行第520条）
!!               # 执行上一条命令（等价于按上箭头再回车）
!$               # 引用上一条命令的最后一个参数

# 示例：
mkdir /tmp/test_dir
cd !$            # 等价于 cd /tmp/test_dir

!ls              # 执行最近一条以 ls 开头的命令
```

#### 反向搜索历史命令

按 `Ctrl + r`，然后输入关键词，可以模糊搜索历史命令。再次按 `Ctrl + r` 继续向前搜索，按 `Enter` 执行匹配的命令，按 `Ctrl + g` 退出搜索。

```
(reverse-i-search)`echo': echo "Hello World"
```

#### 命令别名

```bash
alias                    # 查看当前所有别名

# 定义别名
alias ll='ls -alF'       # 最常用的别名，显示详细信息并标记文件类型
alias la='ls -A'         # 列出所有文件（含隐藏文件，但不含 . 和 ..）
alias grep='grep --color=auto'   # grep 搜索结果高亮显示
alias rm='rm -i'         # 删除前确认（防止误删）

# 取消别名
unalias ll
```

别名只在当前 Shell 会话有效。如果需要永久生效，将别名写入 `~/.bashrc` 或 `~/.bash_aliases` 文件。

```bash
# 编辑 ~/.bashrc 并添加：
echo "alias ll='ls -alF'" >> ~/.bashrc
source ~/.bashrc          # 立即生效，无需重新登录
```

---

## 2. 命令格式与帮助系统

### 2.1 通用命令格式

```
命令 [选项] [参数]
```

- **命令**：要执行的程序名称
- **选项**：对命令进行第一层设置（如 `-a`, `-l`）
  - 短选项：单个字母，前面加 `-`，如 `ls -l -a` 可合并为 `ls -la`
  - 长选项：完整单词，前面加 `--`，如 `ls --all`
- **参数**：更深层设置（如文件名、目录路径）

```bash
# 示例：选项和参数的区别
ls -l /home          # -l 是选项（控制显示格式），/home 是参数（要操作的对象）
grep -n "error" log  # -n 是选项（显示行号），"error" 和 log 是参数
```

### 2.2 三种帮助查询方式

Linux 提供三层帮助体系，从简到深：

| 方式 | 命令格式 | 适用场景 |
|------|----------|----------|
| `--help` | `命令 --help` | 快速查看选项列表，适合已知命令名但忘记选项 |
| `help` | `help 命令` | 仅用于 bash 内置命令（如 cd、echo、alias） |
| `man` | `man 命令` | 最详细的官方手册，包含所有用法、返回值、相关文件 |

```bash
# --help：查看外部命令的简要帮助
ls --help            # 列出 ls 的所有选项

# help：查看 Shell 内置命令的帮助
help cd              # 查看 cd 命令的帮助（cd 是内置命令，没有 --help）
help echo            # 查看 echo 的帮助

# man：查看详细手册
man ls               # ls 的完整手册，包括所有选项、退出状态、作者、BUG等
```

**注意**：`help` 命令只能用于 bash 内置命令。对于外部命令（如 ls、cp、grep），使用 `命令 --help` 或 `man 命令`。

辨别内置命令和外部命令：

```bash
type cd              # 输出：cd is a shell builtin
type ls              # 输出：ls is aliased to `ls --color=auto'
which ls             # 输出：/usr/bin/ls（外部命令的路径）
```

### 2.3 info 命令

`info` 是比 man 更详细的文档系统（GNU Texinfo 格式），支持超链接式导航。

```bash
info ls              # 以超文本格式查看 ls 手册
info coreutils       # 查看 coreutils 整体文档（包含 ls、cp、mv 等）
```

info 阅读器快捷键：

| 快捷键 | 功能 |
|--------|------|
| `n` | 跳转到下一个节点 |
| `p` | 跳转到上一个节点 |
| `u` | 跳转到上层节点 |
| `Enter` | 进入光标所在的链接 |
| `q` | 退出 info |

**建议**：初学者优先使用 `man`，遇到 GNU 工具（如 gcc、bash、coreutils）时可以尝试 `info` 获取更深入的解释。

### 2.4 man 手册的 9 章结构

| 章节 | 内容 | 典型查询 |
|------|------|----------|
| 第1章 | 可执行程序与 Shell 命令 | `man 1 ls` |
| 第2章 | 系统调用（内核提供的函数） | `man 2 open` |
| 第3章 | 库函数调用（C 标准库等） | `man 3 printf` |
| 第4章 | 特殊文件（/dev 下的设备文件） | `man 4 tty` |
| 第5章 | 文件格式与配置文件 | `man 5 passwd` |
| 第6章 | 游戏 | `man 6 sl` |
| 第7章 | 杂项（宏包、协议等） | `man 7 signal` |
| 第8章 | 系统管理命令（root 权限） | `man 8 mount` |
| 第9章 | 内核例程 | `man 9 printk` |

指定章节查询：

```bash
man 3 printf         # 查 C 库函数 printf（格式化输出）
man 1 printf         # 查 Shell 命令 printf（Shell 格式化输出）
man 5 passwd         # 查 /etc/passwd 文件的格式说明
man 1 passwd         # 查 passwd 命令（修改密码）
```

如果不指定章节号，默认从第1章开始查找。如果有同名条目在多个章节，man 显示找到的第一个。

### 2.5 man 手册的搜索功能

```bash
man -k keyword       # 在所有手册的 NAME 节中搜索关键词（等价于 apropos 命令）
man -k print         # 搜索所有名称或描述中包含 "print" 的手册页
man -f ls            # 精确搜索命令名（等价于 whatis 命令），显示在哪些章节出现

# 示例：
man -k sort
# 输出：
# sort (1)             - sort lines of text files
# tsort (1)            - perform topological sort
# ...

man -f printf
# 输出：
# printf (1)           - format and print data
# printf (3)           - formatted output conversion
```

### 2.6 man 手册内部快捷键

在 man 手册阅读过程中：

| 快捷键 | 功能 |
|--------|------|
| `Space` 或 `f` | 向下翻一页（Forward） |
| `b` | 向上翻一页（Backward） |
| `Enter` 或 `j` 或 `↓` | 向下滚动一行 |
| `k` 或 `↑` | 向上滚动一行 |
| `g` | 跳到手册开头 |
| `G` | 跳到手册末尾 |
| `/关键词` | 向下搜索关键词，按 `n` 跳到下一个匹配，按 `N` 跳到上一个 |
| `?关键词` | 向上搜索关键词 |
| `q` | 退出 man 手册 |

---

## 3. 目录操作类命令

### 3.1 ls -- 列出目录内容

```bash
ls                  # 列出当前目录内容（不含隐藏文件）
ls -a               # 包含隐藏文件（以 . 开头的文件）
ls -A               # 包含隐藏文件，但不显示 .（当前目录）和 ..（上级目录）
ls -l               # 详细信息格式（权限、链接数、所有者、大小、修改时间、文件名）
ls -lh              # 详细信息 + 人类可读的文件大小（如 1.5K、4.0M）
ls -la              # 组合选项：详细信息 + 隐藏文件
ls -lS              # 按文件大小从大到小排序（S = Sort by size）
ls -lt              # 按修改时间从新到旧排序（t = time）
ls -lr              # 反向排序（r = reverse），常与 -t 或 -S 组合
ls -ltr             # 按修改时间从旧到新（最常用的组合之一）
ls -R               # 递归列出所有子目录内容
ls -F               # 在目录名后加 /，可执行文件后加 *，链接后加 @
ls -i               # 显示 inode 号
ls --color=auto     # 按文件类型着色（通常已在 ~/.bashrc 中通过别名默认启用）
```

详细信息的字段含义（`ls -l` 的输出）：

```
drwxr-xr-x 2 fire fire 4096 Jul 22 10:30 Desktop
|  |  |  |  |    |     |    |         |
|  |  |  |  |    |     |    |         +-- 文件名
|  |  |  |  |    |     |    +-- 修改时间（月 日 时间）
|  |  |  |  |    |     +-- 文件大小（字节）
|  |  |  |  |    +-- 所属组
|  |  |  |  +-- 所有者
|  |  |  +-- 硬链接数
|  |  +-- 其他用户权限
|  +-- 组权限
+-- 文件类型 + 所有者权限（d=目录, -=普通文件, l=链接）
```

### 通配符在 ls 等命令中的使用

通配符（Globbing）是 Shell 提供的文件名匹配机制，适用于 `ls`、`cp`、`mv`、`rm` 等几乎所有操作文件的命令。

| 通配符 | 含义 | 示例 |
|--------|------|------|
| `*` | 匹配任意长度任意字符 | `ls *.txt` 列出所有 .txt 文件 |
| `?` | 匹配单个任意字符 | `ls file?.txt` 匹配 file1.txt、fileA.txt |
| `[abc]` | 匹配方括号内任意一个字符 | `ls [abc]*.txt` 匹配 a.txt、b.txt、c.txt |
| `[a-z]` | 匹配指定范围内的任意一个字符 | `ls [0-9]*.txt` 匹配以数字开头的 .txt 文件 |
| `[!abc]` | 匹配不在方括号内的任意一个字符 | `ls [!0-9]*.txt` 匹配不以数字开头的 .txt 文件 |
| `{a,b,c}` | 花括号展开（不是严格意义上的通配符） | `ls *.{txt,md}` 匹配 .txt 和 .md 文件 |

```bash
# 实操示例
ls *.c              # 列出所有 C 源文件
ls report?.pdf      # 列出 report1.pdf、report2.pdf 等（单个字符）
ls [A-Z]*.log       # 列出以大写字母开头的 .log 文件
ls *.{py,sh}        # 列出所有 .py 和 .sh 文件
```

### 3.2 cd -- 切换目录

```bash
cd /                # 到根目录
cd ..               # 返回上级目录
cd ../..            # 返回上两级目录
cd ~                # 回到家目录（等价于 cd $HOME 或 cd）
cd -                # 切换到上一个工作目录（在两边来回切换非常实用）
cd /home/fire/doc   # 到指定绝对路径
```

**提示**：`cd -` 是最常用的技巧之一，在两个目录之间快速来回切换。

### 3.3 pwd -- 显示当前路径

```bash
pwd                 # 显示当前工作目录的完整路径
# 输出示例：/home/fire

pwd -P              # 如果用符号链接进入了某个目录，-P 显示真实的物理路径
# 示例：/var/run 是 /run 的符号链接，pwd 显示 /var/run，pwd -P 显示 /run
```

### 3.4 mkdir -- 创建目录

```bash
mkdir mydir         # 创建单个目录
mkdir -p a/b/c/d    # 递归创建多级目录（如果中间目录不存在，自动创建）
mkdir -p /home/fire/projects/myapp/src   # 一键创建完整目录树
mkdir -m 755 mydir  # 创建目录的同时指定权限（755 = rwxr-xr-x）
```

> **建议**：在家目录（`~`）下创建练习用的目录，避免权限问题。

```bash
# 实操示例：创建项目目录结构
mkdir -p ~/myproject/{src,docs,tests,config}
# 等价于分别执行：
# mkdir ~/myproject/src
# mkdir ~/myproject/docs
# mkdir ~/myproject/tests
# mkdir ~/myproject/config
```

### 3.5 rmdir -- 删除空目录

```bash
rmdir emptydir       # 只能删除空目录，如果目录非空会报错
rmdir -p a/b/c       # 递归删除路径上的所有空目录
```

> **注意**：`rmdir` 只能删除空目录。非空目录用 `rm -r`（见 4.5 节）。

### 3.6 mv -- 重命名 / 移动

```bash
mv old.txt new.txt          # 重命名文件
mv file.txt /tmp/           # 将文件移动到 /tmp 目录
mv dir1 /home/user/backup/  # 将整个目录移动到其他位置
mv -i src.txt dest.txt      # 如果目标已存在，提示是否覆盖
mv -v file.txt /tmp/        # 显示移动过程的详细信息
```

### 3.7 tree -- 以树状图显示目录结构

`tree` 命令需要安装（通常不是预装的）：

```bash
# Ubuntu/Debian
sudo apt install tree

# 基本用法
tree                # 以树状图显示当前目录结构
tree /home/fire     # 显示指定目录的树状结构
tree -L 2           # 只显示2层深度（-L 指定层级）
tree -d             # 只显示目录，不显示文件
tree -a             # 包含隐藏文件
tree -h             # 同时显示文件大小（human-readable）
tree -f             # 显示每个文件的完整路径

# 实操示例
mkdir -p ~/demo/{src,include,lib,tests}
touch ~/demo/src/main.c
touch ~/demo/include/header.h
tree ~/demo
# 输出：
# /home/fire/demo
# |-- include
# |   `-- header.h
# |-- lib
# |-- src
# |   `-- main.c
# `-- tests
```

---

## 4. 文件操作类命令

### 4.1 touch -- 创建空文件 / 更新文件时间戳

```bash
touch newfile.txt            # 创建一个空文件（如果不存在）
touch existing.txt           # 如果文件已存在，更新其访问时间和修改时间为当前时间
touch -t 202501011200 a.txt  # 将文件时间戳设置为指定时间（2025-01-01 12:00）

# 实操示例：批量创建多个文件
touch file{1..5}.txt
# 等效于：touch file1.txt file2.txt file3.txt file4.txt file5.txt
```

### 4.2 查看文件内容的命令族

Linux 提供了多种查看文本文件内容的方式，各有适用场景。

#### cat -- 连接并显示文件内容

```bash
cat file.txt             # 显示文件全部内容
cat -n file.txt          # 显示行号
cat -b file.txt          # 对非空行显示行号
cat -A file.txt          # 显示所有不可见字符（制表符显示为^I，行尾显示$）
cat file1.txt file2.txt  # 依次显示多个文件的内容（拼接）
cat file1.txt file2.txt > merged.txt  # 将多个文件合并输出到新文件
```

#### more -- 分页查看（只能向下翻）

```bash
more large_file.txt      # 分页显示，按 Space 翻页，按 Enter 翻一行，按 q 退出
```

#### less -- 分页查看（可上下翻页，推荐使用）

```bash
less large_file.txt      # 分页显示，支持上下翻页和搜索
```

less 内部快捷键：

| 快捷键 | 功能 |
|--------|------|
| `Space` / `f` | 向下翻一页 |
| `b` | 向上翻一页 |
| `j` / `↓` | 向下滚动一行 |
| `k` / `↑` | 向上滚动一行 |
| `g` | 跳到文件开头 |
| `G` | 跳到文件末尾 |
| `/关键词` | 向下搜索 |
| `?关键词` | 向上搜索 |
| `n` | 下一个搜索结果 |
| `N` | 上一个搜索结果 |
| `q` | 退出 |

**less vs more**：less 功能更强大（"less is more"），支持回翻、搜索高亮、大文件不一次性加载到内存等。日常使用推荐 `less`。

#### head -- 查看文件开头

```bash
head file.txt            # 默认显示前 10 行
head -n 20 file.txt      # 显示前 20 行
head -n -5 file.txt      # 显示除最后 5 行外的所有行
```

#### tail -- 查看文件末尾

```bash
tail file.txt            # 默认显示最后 10 行
tail -n 20 file.txt      # 显示最后 20 行
tail -n +5 file.txt      # 从第 5 行开始显示到末尾
```

#### tail -f -- 实时跟踪文件变化（最有用的功能之一）

```bash
tail -f /var/log/syslog       # 实时查看系统日志的新增内容（Ctrl+C 退出）
tail -F /var/log/app.log      # -F 与 -f 类似，但文件轮转后会自动重新打开
```

**典型场景**：在终端 A 中用 `tail -f` 监控日志，在终端 B 中操作程序，可以实时看到日志输出。

```bash
# 实操示例：终端1
tail -f /var/log/syslog

# 终端2
logger "test log message"
# 此时终端1立即显示新加入的日志行
```

#### 各命令适用场景总结

| 命令 | 适用场景 |
|------|----------|
| `cat` | 查看短文件，或拼接多个文件 |
| `less` | 查看长文件，需要上下翻页和搜索 |
| `more` | 老旧系统上的简单分页（功能弱于 less） |
| `head` | 只看文件开头几行（如查看配置文件头部注释） |
| `tail` | 只看文件末尾几行 |
| `tail -f` | 实时监控日志文件 |

### 4.3 echo -- 输出字符串与重定向

```bash
echo "Hello World"                   # 终端显示：Hello World
echo -e "Line1\nLine2"               # -e 启用转义字符：\n 换行，\t 制表符
echo -e "Col1\tCol2\tCol3"           # 输出带制表符分隔的内容
echo -n "No newline"                 # -n 不在末尾添加换行符
echo $PATH                           # 输出环境变量的值
echo "当前用户: $(whoami)"           # 在字符串中嵌入命令执行结果
```

常用转义字符（需要 `-e` 参数）：

| 转义字符 | 含义 |
|----------|------|
| `\n` | 换行（newline） |
| `\t` | 制表符（tab） |
| `\\` | 反斜杠本身 |
| `\"` | 双引号 |
| `\b` | 退格 |

```bash
# 实操示例
echo -e "名称\t数量\t价格\n苹果\t3\t15元\n香蕉\t5\t10元"
# 输出：
# 名称    数量    价格
# 苹果    3       15元
# 香蕉    5       10元
```

### 4.4 重定向详解

重定向是将命令的输出（或输入）导向到文件或其他命令的机制。

#### 标准输入 / 输出 / 错误

Linux 中每个进程有三个默认的数据流：

| 文件描述符 | 名称 | 默认目标 | 说明 |
|-----------|------|---------|------|
| 0 | stdin（标准输入） | 键盘 | 命令读取数据的来源 |
| 1 | stdout（标准输出） | 终端 | 命令正常输出的目标 |
| 2 | stderr（标准错误） | 终端 | 命令错误输出的目标 |

#### 输出重定向

```bash
# > 覆盖重定向（先清空文件，再写入）
echo "Hello" > file.txt         # 将 Hello 写入 file.txt（覆盖原内容）

# >> 追加重定向（在文件末尾添加）
echo "World" >> file.txt        # 将 World 追加到 file.txt 末尾

# 2> 错误重定向（只捕获错误输出）
ls /nonexistent 2> error.log    # 错误信息写入 error.log，正常输出仍显示在终端

# 2>> 追加错误重定向
ls /nonexistent 2>> error.log   # 错误信息追加到 error.log

# &> 同时重定向标准输出和错误输出
ls /tmp /nonexistent &> all.log # 正常输出和错误输出都写入 all.log

# > /dev/null 丢弃输出
ls > /dev/null                  # 正常输出被丢弃
ls 2> /dev/null                 # 错误输出被丢弃
ls &> /dev/null                 # 所有输出被丢弃（静默模式）
```

**注意**：`>` 会先清空目标文件。如果目标文件是一个重要的已有文件，内容将丢失。

```bash
# 实操示例：区分 stdout 和 stderr
ls /home /nonexistent > ok.txt 2> err.txt
# ok.txt 中包含：/home 目录下的文件列表
# err.txt 中包含：ls: cannot access '/nonexistent': No such file or directory
```

### 4.5 管道符 | 详解

管道符 `|` 将前一个命令的**标准输出**作为后一个命令的**标准输入**。

```
命令A | 命令B | 命令C ...
```

```bash
# 基础用法
ls -l | grep ".txt"           # 从 ls 输出中过滤出 .txt 文件

# 多级管道
cat access.log | grep "ERROR" | cut -d' ' -f1 | sort | uniq -c | sort -rn
# 分解说明：
# cat access.log          - 读取日志文件
# grep "ERROR"            - 过滤出含 ERROR 的行
# cut -d' ' -f1           - 提取每行第一个字段（通常是 IP）
# sort                    - 排序（uniq 需要排序后的输入）
# uniq -c                 - 统计每个 IP 出现次数
# sort -rn                - 按次数从大到小排序

# 管道和重定向结合
ls -l | grep ".txt" > txt_files.txt   # 将管道结果保存到文件
```

**管道 vs 重定向的本质区别**：

| | 管道 `|` | 重定向 `>` |
|--|---------|-----------|
| 连接对象 | 命令与命令 | 命令与文件 |
| 数据流向 | 标准输出 -> 标准输入 | 标准输出 -> 文件 |

### 4.6 tee 命令 -- 同时输出到终端和文件

`tee` 从标准输入读取数据，同时输出到标准输出（终端）和一个或多个文件。

```bash
# 基本用法
ls -l | tee output.txt       # 在终端显示的同时保存到 output.txt

# -a 追加模式（不覆盖）
echo "追加内容" | tee -a output.txt

# 多个文件
ls -l | tee file1.txt file2.txt   # 同时保存到两个文件

# 典型场景：既想看实时输出，又想保留日志
./compile.sh 2>&1 | tee build.log  # 编译输出同时显示在屏幕并存入 build.log
```

### 4.7 wc -- 文本统计（Word Count）

```bash
wc file.txt            # 输出：行数 单词数 字符数 文件名
wc -l file.txt         # 仅统计行数
wc -w file.txt         # 仅统计单词数
wc -c file.txt         # 仅统计字符数（字节数）
wc -m file.txt         # 仅统计字符数（考虑多字节字符，如中文）
wc -L file.txt         # 显示最长行的长度

# 管道中使用
ls -1 | wc -l          # 统计当前目录下有多少个文件/目录
cat file.txt | wc -w   # 统计文件中的单词数
```

### 4.8 rm -- 删除文件或目录

```bash
rm file.txt            # 删除文件
rm -r dirname          # 递归删除非空目录（含所有子文件/目录）
rm -i file.txt         # 交互式删除，每删除一个文件都要求确认
rm -I file.txt         # 删除3个以上文件时提示一次确认（比 -i 温和）
rm -v file.txt         # 显示删除过程（verbose，删除什么就显示什么）
rm -f file.txt         # 强制删除，不提示（force，即使文件不存在也不报错）
rm -rf dirname         # 强制递归删除（极度危险！没有确认，不可恢复）
```

**rm 与 rmdir 的区别**：

| 命令 | 适用范围 | 注意事项 |
|------|---------|---------|
| `rmdir` | 只能删除**空**目录 | 安全，有保护机制 |
| `rm -r` | 可递归删除**非空**目录 | 需谨慎使用 |
| `rm -rf` | 强制递归删除 | 极度危险，不可恢复 |

> **警告**：永远不要执行 `sudo rm -rf /` 或 `sudo rm -rf /*`。这会删除整个系统，没有回收站。

```bash
# 实操示例
mkdir -p ~/demo/rm_test/subdir
touch ~/demo/rm_test/file1.txt
touch ~/demo/rm_test/subdir/file2.txt

# 尝试用 rmdir（失败，因为目录非空）
rmdir ~/demo/rm_test
# 错误：rmdir: failed to remove 'rm_test': Directory not empty

# 用 rm -r（成功）
rm -r ~/demo/rm_test
# 整个 rm_test 目录被删除
```

### 4.9 ln -- 创建链接文件（预览）

`ln` 命令用于创建链接，分为硬链接（hard link）和符号链接（symbolic link，也叫软链接）。

```bash
# 硬链接：同一个文件的不同入口，共享同一个 inode
ln original.txt hardlink.txt

# 符号链接（软链接）：指向目标文件的快捷方式，有独立的 inode
ln -s original.txt symlink.txt
ln -s /usr/local/bin/node /usr/bin/node   # 创建软链接到 PATH 路径

# 查看链接
ls -l                 # 软链接显示为 lrwxrwxrwx，并显示箭头指向目标
readlink symlink.txt  # 查看软链接指向的目标
```

| 对比 | 硬链接 | 符号链接 |
|------|--------|---------|
| inode | 与目标相同 | 独立 inode |
| 跨文件系统 | 不可以 | 可以 |
| 链接目录 | 不可以 | 可以 |
| 目标删除后 | 链接仍可用（数据还在） | 链接断裂（变成死链接） |

更详细的链接讲解见中篇笔记。这里先了解基本概念即可。

---

## 5. 命令速查总表

### 5.1 Shell 与帮助

| 类别 | 命令 | 功能 |
|------|------|------|
| Shell | `echo $SHELL` | 查看当前 Shell |
| Shell | `cat /etc/shells` | 查看系统支持的 Shell |
| Shell | `alias` | 查看/设置命令别名 |
| Shell | `history` | 查看历史命令 |
| Shell | `!n` / `!!` | 执行指定历史命令 |
| Shell | `type 命令` | 查看命令类型（内置/外部/别名） |
| 帮助 | `man 命令` | 查看详细手册 |
| 帮助 | `命令 --help` | 查看简要用法 |
| 帮助 | `help 命令` | 查看 Shell 内置命令帮助 |
| 帮助 | `info 命令` | 查看 info 格式文档（GNU 工具） |
| 帮助 | `man -k 关键词` | 搜索手册页 |
| 帮助 | `man -f 命令` | 精确查询手册位置 |

### 5.2 目录操作

| 命令 | 常用选项 | 功能 |
|------|---------|------|
| `ls` | `-a` `-l` `-lh` `-S` `-t` `-r` `-R` | 列出目录内容 |
| `cd` | `-`（回上一个目录） | 切换目录 |
| `pwd` | `-P`（物理路径） | 显示当前路径 |
| `mkdir` | `-p` `-m` | 创建目录 |
| `rmdir` | `-p` | 删除空目录 |
| `mv` | `-i` `-v` | 重命名 / 移动 |
| `tree` | `-L N` `-d` `-a` | 树状显示目录结构 |

### 5.3 文件操作

| 命令 | 常用选项 | 功能 |
|------|---------|------|
| `touch` | `-t` | 创建空文件 / 更新时间戳 |
| `cat` | `-n` `-b` `-A` | 显示文件全部内容 |
| `less` | - | 分页查看（推荐） |
| `more` | - | 分页查看（简单） |
| `head` | `-n N` | 查看文件开头 |
| `tail` | `-n N` `-f` | 查看文件末尾 / 实时跟踪 |
| `echo` | `-e` `-n` | 输出字符串 |
| `>` | - | 覆盖重定向 |
| `>>` | - | 追加重定向 |
| `2>` | - | 错误重定向 |
| `&>` | - | 同时重定向 stdout 和 stderr |
| `|` | - | 管道（连接命令） |
| `tee` | `-a` | 同时输出到终端和文件 |
| `wc` | `-l` `-w` `-c` `-m` | 文本统计 |
| `rm` | `-r` `-f` `-i` `-v` | 删除文件或目录 |
| `ln` | `-s` | 创建链接文件 |

---

## 6. 本讲要点

1. **Shell** = 用户与内核的桥梁，功能：接收命令 -> 传递执行 -> 呈现结果
2. **Shell 类型**：bash（默认）、zsh、fish、sh、dash 等，通过 `echo $SHELL` 查看当前 Shell，`cat /etc/shells` 查看系统支持的所有 Shell
3. **Tab 补全**：按一次尝试补全，按两次列出所有匹配项
4. **历史命令**：`history` 查看，`!n` 执行第 n 条，`!!` 执行上一条，`Ctrl+r` 反向搜索
5. **命令别名**：`alias ll='ls -alF'`，永久保存写入 `~/.bashrc`
6. **帮助系统**三层：`--help`（快速）、`help`（内置命令）、`man`（详细手册）
7. **man 手册**分 9 章，可用 `man N 函数名` 指定章节，`man -k` 搜索手册
8. **man 内部快捷键**：`Space/f` 翻页、`b` 回翻、`/` 搜索、`q` 退出
9. **命令通用格式**：`命令 [选项] [参数]`
10. **隐藏文件**以 `.` 开头，`ls -a` 才能看到
11. **ls 常用选项**：`-lh`（人类可读大小）、`-S`（按大小排序）、`-t`（按时间排序）、`-r`（反向）
12. **通配符**：`*`（任意字符）、`?`（单个字符）、`[]`（字符集），适用于大多数文件操作命令
13. **重定向**：`>` 覆盖，`>>` 追加，`2>` 错误，`&>` 全部，`/dev/null` 丢弃
14. **管道 `|`**：连接命令，前一个命令的 stdout 变为后一个命令的 stdin
15. **tee**：同时输出到终端和文件
16. **查看文件**：短文件用 `cat`，长文件用 `less`，看开头用 `head`，看末尾用 `tail`，实时监控用 `tail -f`
17. **删除目录**：空目录用 `rmdir`，非空用 `rm -r`
18. **链接**：`ln` 创建硬链接，`ln -s` 创建符号链接（软链接）

> 下节预告：继续讲解网络操作、权限管理、进程管理等命令

---

*笔记生成日期：2026-07-22 | 转写：Whisper tiny*
*优化日期：2026-07-22*
