---
title: "第9讲：使用Linux命令行（中）"
date: 2026-09-11T09:20:00+08:00
draft: false
description: "在理解链接命令之前，有必要先了解 Linux 文件系统的核心概念：inode。"
series: ["Linux 入门"]
series_order: 7
categories: ["技术笔记"]
tags: ["Linux", "命令行"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P10)
> **标题**：第9讲 — 使用Linux命令（中）
> **时长**：约23分
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. 前置知识：inode 与文件系统基础

在理解链接命令之前，有必要先了解 Linux 文件系统的核心概念：**inode**。

### 1.1 什么是 inode？

Linux 文件系统将文件分为两部分存储：

| 组成部分 | 存储内容 | 说明 |
|----------|----------|------|
| **inode（索引节点）** | 文件元数据 | 权限、所有者、大小、时间戳、数据块指针等 |
| **数据块（data block）** | 文件实际内容 | 文件中存放的真实数据 |

每个文件在创建时都会分配一个唯一的 inode 编号，这个编号是文件的"身份证号"。文件名只是指向 inode 的一个标签，两者是分离的。

### 1.2 查看 inode 信息

```bash
# 查看文件的 inode 编号
ls -i 文件名
ls -li 文件名          # 同时显示 inode 和详细属性

# 查看文件的完整元数据
stat 文件名
```

`stat` 命令输出示例：
```
  文件：123.txt
  大小：42        	块：8          IO 块：4096   普通文件
设备：801h/2049d	Inode：1049088     硬链接：1
权限：(0644/-rw-r--r--)  Uid：( 1000/   laoli)   Gid：( 1000/   laoli)
最近访问：2026-07-22 10:30:00
最近更改：2026-07-22 10:25:00
最近改动：2026-07-22 10:25:00
创建时间：-
```

`stat` 输出的三个时间戳含义：

| 时间戳 | 缩写 | 含义 |
|--------|------|------|
| 最近访问 (Access) | atime | 文件最后一次被读取的时间 |
| 最近更改 (Modify) | mtime | 文件内容最后一次被修改的时间 |
| 最近改动 (Change) | ctime | 文件元数据（权限、所有者等）最后一次变更的时间 |

---

## 2. ln — 创建链接文件

ln 命令用于为文件创建链接，分为**硬链接**和**软链接（符号链接）**两种。

### 2.1 硬链接（Hard Link）

```bash
ln 原文件 链接文件
```

#### 底层原理

硬链接的本质是：**在目录中创建一个新的文件名，指向同一个 inode**。

```
原始文件         创建硬链接后
+---------+     +---------+
| 文件名A  |     | 文件名A  |----+
| 123.txt |     | 123.txt |    |
+----+----+     +---------+    |   +---------+
     |                         +-->| inode   |
     |              +---------+|   | #12345  |--> 数据块
     |              | 文件名B  ||   +---------+
     +--------------| 456     |+
                    +---------+
```

inode 内部有一个**链接计数器（link count）**，每多一个文件名指向它，计数就 +1。当计数降为 0 时，文件才真正被删除（数据块被释放）。

可以用 `ls -l` 第二列查看链接计数，或直接用 `stat` 查看。

#### 实操示例：用 inode 验证硬链接

```bash
# 创建测试文件
echo "hello world" > 123.txt

# 查看 inode 编号
ls -i 123.txt
# 输出：1049088 123.txt

# 创建硬链接
ln 123.txt 456

# 再次查看 inode（两者相同！）
ls -i 123.txt 456
# 输出：
# 1049088 123.txt
# 1049088 456            <- 同一个 inode！

# 查看链接计数
ls -l 123.txt 456
# 第二列数字变为 2（之前是 1）
stat 123.txt | grep 链接
# 输出：硬链接：2

# 删除原文件后硬链接仍可访问
rm 123.txt
cat 456                   # hello world  <- 内容还存在！
ls -li 456                # inode 不变，链接计数降为 1
```

#### 硬链接的限制

1. **不能跨文件系统**：inode 编号只在单个文件系统内唯一，不同分区可能使用相同的 inode 编号，所以硬链接只能在同一个分区内创建。
2. **不能链接目录**：防止产生循环引用，导致文件系统遍历时陷入死循环。Linux 内核直接禁止了对目录创建硬链接（root 也不行）。`.` 和 `..` 是唯一的例外，由系统自动维护。
3. **不适用某些文件系统**：部分非 Unix 风格的文件系统（如 FAT32、exFAT）不支持硬链接。

### 2.2 软链接 / 符号链接（Symbolic Link）

```bash
ln -s 原文件 链接文件
```

#### 底层原理

软链接是一个**独立的文件**，拥有自己的 inode 和数据块。它的数据块中存放的是目标文件的**路径字符串**。

```
软链接文件
+---------+     +---------+
| 文件名   |---->| inode   |--> 数据块："456"（路径字符串）
| 789     |     | #67890  |
+---------+     +---------+
```

因为软链接存的是路径而不是 inode 引用，所以：
- 可以跨文件系统
- 可以链接目录
- 原文件删除后软链接断裂（就像 Windows 快捷方式指向不存在的目标）

#### 实操示例

```bash
# 创建软链接
ln -s 456 789

# 查看区别
ls -li 456 789
# 输出：
# 1049088 -rw-r--r-- 2 laoli laoli 12 Jul 22 10:30 456     <- 普通文件
# 1049090 lrwxrwxrwx 1 laoli laoli  3 Jul 22 10:31 789 -> 456  <- 软链接

# inode 不同！789 有自己的 inode
# 文件类型为 l（link），权限显示 lrwxrwxrwx

# 软链接的大小 = 目标路径的字符数
# 这里 "456" 是 3 个字符，所以大小为 3

# 删除目标文件后软链接断裂
rm 456
cat 789
# 报错：cat: 789: 没有那个文件或目录
ls -l 789
# 显示：789 -> 456（红色闪烁，表示断裂的软链接）
```

#### 软链接的"穿透"特性

```bash
mkdir dir1
echo "content" > dir1/file.txt
ln -s dir1 link_to_dir

cd link_to_dir          # 可以像进入真实目录一样进入软链接
cat file.txt            # 读取到的是 dir1/file.txt
ls ..                   # 显示的是 link_to_dir 的上级目录

cd -P link_to_dir       # -P 选项：进入"物理"目录（追踪到真实路径）
cd -L link_to_dir       # -L 选项：停留在"逻辑"目录（默认行为）
```

### 2.3 两种链接的对比

| 特性 | 硬链接 | 软链接 |
|------|--------|--------|
| 本质 | 同一个 inode 的多个文件名 | 独立文件，存有目标路径 |
| inode | 与目标相同 | 与目标不同 |
| `ls -l` 显示 | 与普通文件相同 | `lrwxrwxrwx` + `->` 指向关系 |
| `ls -i` 显示 | inode 编号相同 | inode 编号不同 |
| 删除原文件 | 仍可访问 | 失效（断裂） |
| 跨文件系统 | 不支持 | 支持 |
| 链接目录 | 不支持 | 支持 |
| 链接计数 | 会递增 | 不递增 |
| 占用空间 | 仅占目录项 | 占一个 inode + 路径字符串空间 |

---

## 3. cp — 复制文件或目录

### 3.1 基本用法

```bash
cp 源文件 目标文件         # 复制文件
cp -r 源目录 目标目录       # 递归复制目录（-r 或 -R）
```

示例：
```bash
cp 123.txt test            # 复制文件
cp -r bfdir bfdir_backup   # 复制整个目录
```

### 3.2 常用选项详解

| 选项 | 全称 | 功能 | 示例 |
|------|------|------|------|
| `-i` | interactive | 覆盖前询问确认 | `cp -i a.txt b.txt` |
| `-u` | update | 仅在源文件更新时才复制（跳过同名旧文件） | `cp -u *.txt backup/` |
| `-p` | preserve | 保留权限、时间戳、所有者等属性 | `cp -p a.txt b.txt` |
| `-a` | archive | 归档模式（等于 `-dR --preserve=all`），递归+保留所有属性+跟随软链接 | `cp -a src/ dst/` |
| `-v` | verbose | 显示复制过程 | `cp -v *.txt backup/` |
| `-f` | force | 强制覆盖（目标不能打开时先删除） | `cp -f a.txt b.txt` |
| `-l` | link | 不复制内容，改为创建硬链接 | `cp -l a.txt b.txt` |
| `-n` | no-clobber | 不覆盖已存在的文件 | `cp -n *.txt backup/` |

#### 实操示例

```bash
# 交互式复制（防止误覆盖）
cp -i important.txt backup/
# cp: 是否覆盖 'backup/important.txt'？ y

# 增量备份（只复制有变化的文件）
cp -ru source/ backup/

# 保留所有属性的复制
cp -p config.ini config.ini.bak
ls -l config.ini*       # 时间戳和权限完全一致

# 归档模式（最常用的备份方式）
cp -a /etc/nginx/ /backup/nginx-$(date +%Y%m%d)/
# 完整保留目录结构、权限、软链接等所有属性
```

### 3.3 复制目录时源路径结尾 `/` 的区别

```bash
# 情况1：源路径没有结尾 /
cp -r dir1 dir2
# 如果 dir2 不存在 -> 创建 dir2，并把 dir1 的内容复制进去
# 如果 dir2 已存在 -> 把 dir1 整个目录复制到 dir2 里面（变成 dir2/dir1）

# 情况2：源路径有结尾 / 或用 /*
cp -r dir1/ dir2
# 如果 dir2 不存在 -> 创建 dir2，并把 dir1 目录下的内容复制进去
# 如果 dir2 已存在 -> 把 dir1 目录下的内容复制到 dir2 里面（不创建 dir1 子目录）
```

理解这个区别在编写脚本复制文件时至关重要，避免目录结构出问题。

---

## 4. tar — 打包与解包

> tar 最初的设计用途是 **T**ape **AR**chiver（磁带归档器）。它只负责将多个文件合并成一个归档文件，不负责压缩。压缩需要结合其他工具（如 gzip、bzip2、xz）。

### 4.1 基本语法

```bash
tar [选项] 包名.tar 文件/目录...
```

核心选项：

| 选项 | 含义 | 记忆 |
|------|------|------|
| `-c` | create，创建打包 | **C**reate |
| `-x` | extract，解包 | E**x**tract |
| `-t` | list，查看包内容 | Lis**t** |
| `-v` | verbose，显示详细信息 | **V**erbose |
| `-f` | file，指定归档文件名 | **F**ile |

### 4.2 基本操作示例

```bash
# 打包
tar -cf backup.tar bfdir bug/

# 查看包内容
tar -tvf backup.tar
# -rw-r--r-- laoli/laoli  1024 2026-07-22 10:30 bfdir/config.txt
# drwxr-xr-x laoli/laoli     0 2026-07-22 10:30 bfdir/subdir/
# ...（列出包内所有文件的属性）

# 解包到当前目录
tar -xf backup.tar

# 解包到指定目录（-C 选项）
tar -xf backup.tar -C /tmp/restore/
```

### 4.3 tar 配合压缩工具

tar 通过 `-z`、`-j`、`-J` 选项内置了对常见压缩工具的支持：

| 选项 | 压缩工具 | 扩展名 | 压缩比 | 速度 |
|------|----------|--------|--------|------|
| `-z` | gzip | `.tar.gz` / `.tgz` | 中 | 快 |
| `-j` | bzip2 | `.tar.bz2` | 高 | 慢 |
| `-J` | xz | `.tar.xz` | 最高 | 最慢 |

```bash
# 打包并用 gzip 压缩（最常用）
tar -czf backup.tar.gz dir/

# 打包并用 bzip2 压缩（压缩率更高）
tar -cjf backup.tar.bz2 dir/

# 打包并用 xz 压缩（压缩率最高）
tar -cJf backup.tar.xz dir/

# 查看压缩包内容（不解压）
tar -tzf backup.tar.gz
tar -tjf backup.tar.bz2

# 解压压缩包
tar -xzf backup.tar.gz
tar -xjf backup.tar.bz2
tar -xJf backup.tar.xz
```

### 4.4 完整流程示例

```bash
# 第一步：打包压缩
tar -czf myproject-$(date +%Y%m%d).tar.gz ~/project/

# 第二步：查看压缩包内容（确认是否包含需要的文件）
tar -tzf myproject-20260722.tar.gz

# 第三步：传输到其他机器后，解压到指定目录
ssh user@remote "mkdir -p /opt/myproject"
scp myproject-20260722.tar.gz user@remote:/tmp/
ssh user@remote "tar -xzf /tmp/myproject-20260722.tar.gz -C /opt/myproject/"

# 第四步：验证解压结果
ssh user@remote "ls -la /opt/myproject/"
```

### 4.5 其他实用选项

```bash
# 打包时排除某些文件（--exclude）
tar -czf backup.tar.gz dir/ --exclude="*.log" --exclude="node_modules"

# 追加文件到已有包（-r）
tar -rf backup.tar newfile.txt

# 从包中删除文件（--delete）
tar --delete -f backup.tar unwanted.txt

# 解压前先预览（-t）确认没有路径冲突
tar -tzf unknown.tar.gz | head -20
```

---

## 5. find — 查找文件

find 是 Linux 下最强大的文件查找工具，根据文件名、类型、大小、时间等各种条件递归搜索目录树。

### 5.1 基本语法

```bash
find [搜索路径] [查找条件] [操作]
```

### 5.2 按文件名查找

```bash
# 精确匹配
find / -name "123.txt"

# 通配符匹配（使用引号防止 Shell 展开）
find . -name "*.txt"
find /etc -name "*.conf"

# 忽略大小写
find . -iname "README*"
```

### 5.3 按文件类型查找（`-type`）

| 类型标识 | 含义 |
|----------|------|
| `f` | 普通文件 (regular file) |
| `d` | 目录 (directory) |
| `l` | 符号链接 (symbolic link) |
| `b` | 块设备 (block device) |
| `c` | 字符设备 (character device) |
| `p` | 管道 (named pipe / FIFO) |
| `s` | 套接字 (socket) |

```bash
# 查找目录
find /home -type d -name "log"

# 查找所有软链接
find /usr/lib -type l

# 查找所有普通 .py 文件
find . -type f -name "*.py"

# 查找空文件或空目录
find . -type f -empty
find . -type d -empty
```

### 5.4 按大小查找（`-size`）

```bash
# 精确匹配 n 个块（512字节），用 c 表示字节
find . -size 1024c           # 正好 1024 字节

# 大于 100MB 的文件
find / -type f -size +100M

# 小于 1KB 的文件
find . -type f -size -1k

# 查找 10MB 到 100MB 的文件
find . -type f -size +10M -size -100M

# 查找大于 1GB 的大文件（排查磁盘空间）
find /home -type f -size +1G
```

大小单位：`c`（字节）、`k`（KB）、`M`（MB）、`G`（GB）

### 5.5 按时间查找

| 选项 | 含义 | 检查的时间戳 |
|------|------|------------|
| `-mtime n` | n 天前修改过（内容） | mtime |
| `-mmin n` | n 分钟前修改过（内容） | mtime |
| `-atime n` | n 天前访问过 | atime |
| `-amin n` | n 分钟前访问过 | atime |
| `-ctime n` | n 天前元数据变更过 | ctime |
| `-cmin n` | n 分钟前元数据变更过 | ctime |

符号含义：`+n`（n 天/分钟之前）、`n`（恰好 n 天/分钟前）、`-n`（n 天/分钟之内）

```bash
# 最近 7 天内修改过的文件
find . -type f -mtime -7

# 60 分钟前修改过的文件
find . -type f -mmin +60

# 最近 24 小时内修改过的 .log 文件
find /var/log -type f -name "*.log" -mtime -1

# 超过 30 天未访问的文件（可能是垃圾文件）
find /tmp -type f -atime +30
```

### 5.6 组合条件

```bash
# AND（默认）：多个条件同时满足
find . -name "*.txt" -size +1M

# OR：使用 -o
find . -name "*.jpg" -o -name "*.png"

# NOT：使用 ! 或 -not
find . -type f ! -name "*.txt"

# 复杂组合（用括号，需转义）
find . \( -name "*.py" -o -name "*.sh" \) -mtime -7
```

### 5.7 find 配合 `-exec` 执行操作

`-exec` 选项允许对搜索到的每个文件执行指定的命令。语法为：
```bash
find ... -exec 命令 {} \;
# {}   = 匹配到的文件路径（占位符）
# \;   = 命令结束标记
```

```bash
# 删除所有 .tmp 临时文件
find . -type f -name "*.tmp" -exec rm {} \;

# 修改一批文件的权限
find . -type f -name "*.sh" -exec chmod +x {} \;

# 把查到的文件移动到另一个目录
find . -type f -name "*.log" -exec mv {} /backup/logs/ \;

# 使用 + 结尾（批量处理，效率更高）
# \; 每个文件执行一次命令，+ 把多个文件合并为一次
find . -type f -name "*.tmp" -exec rm {} +

# -exec 配合确认（-ok 会在执行前询问）
find . -type f -name "*.bak" -ok rm {} \;
# < rm ... ./config.bak > ? y
```

### 5.8 find 配合 xargs

当文件数量极大时，`-exec {} +` 可能受命令行长度限制。此时用 `xargs` 更可靠：

```bash
# find + xargs（默认以空白分隔，文件名有空格会出问题）
find . -type f -name "*.tmp" | xargs rm

# 推荐：使用 -print0 和 -0 处理特殊文件名
find . -type f -name "*.tmp" -print0 | xargs -0 rm

# 与 grep 配合：在所有 .c 文件中搜索关键字
find . -type f -name "*.c" -print0 | xargs -0 grep "main("

# 限制每批处理数量（-n）
find . -type f -name "*.txt" -print0 | xargs -0 -n 100 gzip
```

### 5.9 find 实用技巧

```bash
# 查找并列出文件大小（前 10 大文件）
find . -type f -exec ls -lh {} \; | sort -k5 -hr | head -10

# 统计各类文件数量
find . -type f -name "*.py" | wc -l

# 查找最近修改的 5 个文件
find . -type f -printf '%T@ %p\n' | sort -n | tail -5
```

---

## 6. grep — 文本搜索

grep（**G**lobal **R**egular **E**xpression **P**rint）用于在文本中搜索匹配正则表达式的行。

### 6.1 基本用法

```bash
grep [选项] "模式" 文件...

# 基本搜索
grep "error" app.log              # 在 app.log 中查找含 "error" 的行

# 搜索时显示行号
grep -n "error" app.log           # 输出：23:ERROR: Connection refused

# 高亮匹配（部分系统默认开启）
grep --color "error" app.log
```

### 6.2 正则表达式基础

grep 默认使用**基本正则表达式（BRE）**，`-E` 启用**扩展正则（ERE）**。

#### 基础元字符

| 符号 | 含义 | 示例 | 匹配结果 |
|------|------|------|----------|
| `^` | 行首 | `grep "^Error" log` | 以 "Error" 开头的行 |
| `$` | 行尾 | `grep "done$" log` | 以 "done" 结尾的行 |
| `.` | 任意单个字符 | `grep "h.t" words` | hot, hat, hit... |
| `*` | 前一个字符重复 0 次或多次 | `grep "ab*c"` | ac, abc, abbc... |
| `[]` | 字符集（任选其一） | `grep "[Hh]ello"` | Hello 或 hello |
| `[^]` | 否定字符集 | `grep "[^0-9]" file` | 不含数字的行 |
| `\` | 转义字符 | `grep "\.$" file` | 以句号结尾的行 |

#### 扩展正则（grep -E / egrep）

| 符号 | 含义 | 示例 |
|------|------|------|
| `+` | 前一个字符至少出现 1 次 | `[0-9]+` 匹配一个或多个数字 |
| `?` | 前一个字符出现 0 或 1 次 | `colou?r` 匹配 color 和 colour |
| `{n}` | 恰好出现 n 次 | `[0-9]{3}` 匹配三位数字 |
| `{n,}` | 至少出现 n 次 | `[0-9]{3,}` 匹配三位及以上数字 |
| `{n,m}` | 出现 n 到 m 次 | `[0-9]{2,4}` 匹配二到四位数字 |
| `|` | 或（alternation） | `error|fail` 匹配 error 或 fail |
| `()` | 分组 | `(error|fail)ed` 匹配 errored 或 failed |

```bash
# 查找 IP 地址（简化版）
grep -E "[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}" log

# 查找函数定义（C 语言）
grep -E "^[a-zA-Z_][a-zA-Z0-9_]*\s+\w+\(" *.c

# 查找以 # 开头或以 ; 开头的注释行
grep -E "^(#|;)" config.ini
```

### 6.3 常用选项一览

| 选项 | 含义 | 示例 |
|------|------|------|
| `-i` | 忽略大小写 (ignore case) | `grep -i "error" log` |
| `-v` | 反向匹配（排除匹配行） | `grep -v "^#" config` |
| `-r` / `-R` | 递归搜索目录 | `grep -r "TODO" ~/project/` |
| `-c` | 只显示匹配行数量 (count) | `grep -c "ERROR" log` |
| `-l` | 只显示包含匹配的文件名 (list) | `grep -rl "import sys" .` |
| `-L` | 只显示不包含匹配的文件名 | `grep -rL "TODO" .` |
| `-n` | 显示行号 (line number) | `grep -n "def main" *.py` |
| `-w` | 匹配整个单词 (word) | `grep -w "time" code.py` |
| `-A n` | 同时显示匹配行后 n 行 (After) | `grep -A 3 "ERROR" log` |
| `-B n` | 同时显示匹配行前 n 行 (Before) | `grep -B 2 "ERROR" log` |
| `-C n` | 同时显示匹配行前后各 n 行 | `grep -C 5 "Exception" log` |
| `-E` | 使用扩展正则表达式 | `grep -E "err|warn" log` |
| `-F` | 固定字符串匹配（不解析正则） | `grep -F "[ERROR]" log` |
| `-o` | 只输出匹配部分 (only matching) | `grep -o "[0-9]\+" file` |
| `-H` | 显示文件名（多文件搜索默认）| `grep -H "import" *.py` |
| `--color` | 高亮匹配内容 | `grep --color "main" code.c` |
| `-m n` | 匹配 n 行后停止 | `grep -m 5 "ERROR" log` |

### 6.4 实操示例

```bash
# 忽略大小写搜索
grep -i "warning" app.log

# 递归搜索整个项目
grep -r "config_dir" ~/project/

# 反向匹配：排除注释行和空行
grep -v "^#" config.ini | grep -v "^$"

# 统计出现次数
grep -c "ERROR" /var/log/syslog

# 查找哪些文件引用了某个模块（只列文件名）
grep -rl "import re" ~/project/

# 查看错误上下文（前后各 3 行）
grep -C 3 "FATAL" /var/log/app.log

# 只提取匹配的内容（配合管道）
grep -o "user_[0-9]\+" app.log | sort | uniq

# 查找不包含 error 的行
grep -v "error" build.log
```

### 6.5 grep 配合管道（最常用场景）

```bash
# 筛选进程
ps aux | grep nginx

# 筛选日志
dmesg | grep -i error
journalctl -xe | grep -i fail

# 排除 grep 自身
ps aux | grep nginx | grep -v grep
# 或者：ps aux | grep "[n]ginx"    （正则技巧，不会匹配自身）

# 查找大文件中的关键行
cat huge_file.txt | grep "keyword"

# 组合使用
history | grep "ssh" | tail -5
```

### 6.6 grep 的三种变体：grep / egrep / fgrep

| 命令 | 等价写法 | 正则模式 | 适用场景 |
|------|----------|----------|----------|
| `grep` | `grep -G` | 基本正则（BRE） | 一般搜索 |
| `egrep` | `grep -E` | 扩展正则（ERE） | `+`、`?`、`|`、`()` 等不需要转义 |
| `fgrep` | `grep -F` | 无（固定字符串） | 搜索包含正则元字符的原义字符串 |

```bash
# egrep：扩展正则更方便
egrep "error|fail|panic" app.log
# 等价于
grep -E "error|fail|panic" app.log

# fgrep：搜索含有 [ ] . * 等特殊字符的字符串
fgrep "[ERROR]" app.log     # 把 [ERROR] 当普通字符串，不是字符集
# 等价于
grep -F "[ERROR]" app.log

# 如果不用 fgrep，这些字符需要转义
grep "\[ERROR\]" app.log    # 麻烦且易错
```

---

## 7. 用户管理类命令

### 7.1 sudo — 临时获取 root 权限

```bash
sudo 命令    # 以 root 权限执行命令
```

用于普通用户临时获取 root 权限执行管理操作。sudo 会记录所有执行的命令到日志中，比直接使用 root 更安全。

### 7.2 su — 切换用户

```bash
su 用户名         # 切换到指定用户（需要目标用户密码）
su - 用户名       # 切换并加载目标用户的环境变量（推荐）
su                # 不加用户名，默认切换到 root
```

`su` 与 `su -` 的区别：`su -` 是 login shell，会模拟完整的登录过程，加载目标用户的 `.bashrc`、`.profile` 等环境配置，路径和工作目录也会切换过去。

### 7.3 创建用户：useradd vs adduser

| 对比 | useradd | adduser |
|------|---------|---------|
| 特点 | 仅创建最基础的用户 | 交互式创建，自动完成所有设置 |
| 密码 | 不设置密码 | 提示输入密码 |
| 家目录 | 不自动创建 | 自动创建 `/home/用户名` |
| 所属系统 | 所有 Linux 发行版通用 | Debian/Ubuntu 系的友好封装 |
| 推荐度 | 脚本化操作用 | 手动创建用户用 |

#### useradd 完整选项

> `useradd` 功能强大，通过选项可以精确控制用户创建的每一个细节，适合脚本化批量操作。

```bash
sudo useradd [选项] 用户名
```

| 选项 | 含义 | 示例 |
|------|------|------|
| `-m` | 自动创建家目录 | `useradd -m zhangsan` |
| `-d 路径` | 指定家目录路径 | `useradd -d /opt/homedir -m zhangsan` |
| `-s Shell` | 指定登录 Shell | `useradd -s /bin/zsh zhangsan` |
| `-g 组` | 指定主组 | `useradd -g developers zhangsan` |
| `-G 组1,组2` | 指定附加组 | `useradd -G sudo,docker zhangsan` |
| `-u UID` | 指定用户 ID | `useradd -u 1500 zhangsan` |
| `-e 日期` | 账号过期日期 | `useradd -e 2026-12-31 zhangsan` |
| `-r` | 创建系统用户 | `useradd -r -s /usr/sbin/nologin myservice` |
| `-c "注释"` | 添加用户描述（GECOS） | `useradd -c "张三,开发部" zhangsan` |
| `-p 密码` | 设置加密密码 | `useradd -p $(openssl passwd -1 pwd) zhangsan` |

```bash
# 创建普通用户（常用组合）
sudo useradd -m -s /bin/bash zhangsan
sudo passwd zhangsan              # 为新用户设置密码

# 创建只能运行服务的系统用户（不能登录）
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/myservice myservice

# 创建用户并加入多个组
sudo useradd -m -G sudo,docker,developers zhangsan

# 交互式创建（推荐日常使用）
sudo adduser zhangsan
```

### 7.4 usermod — 修改用户属性

```bash
sudo usermod [选项] 用户名
```

| 选项 | 含义 | 示例 |
|------|------|------|
| `-l 新用户名` | 修改用户名 (login) | `usermod -l newname oldname` |
| `-d 新目录` | 修改家目录 | `usermod -d /new/home -m user` |
| `-s Shell` | 修改登录 Shell | `usermod -s /bin/zsh zhangsan` |
| `-g 组` | 修改主组 | `usermod -g developers zhangsan` |
| `-G 组` | 设置附加组（覆盖！） | `usermod -G sudo zhangsan` |
| `-aG 组` | 追加到附加组（推荐！） | `usermod -aG docker zhangsan` |
| `-L` | 锁定账号 (Lock) | `usermod -L zhangsan` |
| `-U` | 解锁账号 (Unlock) | `usermod -U zhangsan` |
| `-e 日期` | 设置过期日期 | `usermod -e 2026-12-31 zhangsan` |
| `-u UID` | 修改用户 ID | `usermod -u 2000 zhangsan` |

```bash
# 给用户追加 sudo 权限（不覆盖已有组）
sudo usermod -aG sudo zhangsan

# 将用户改名为新名字
sudo usermod -l newname oldname

# 暂时锁定账号（禁止登录）
sudo usermod -L zhangsan

# 解锁
sudo usermod -U zhangsan

# 修改用户的默认 Shell
sudo usermod -s /usr/bin/zsh zhangsan
```

**注意**：`-G` 是覆盖式设置，会把用户从现在的附加组中移除，只保留指定的组。**务必用 `-aG` 来追加组！**

### 7.5 passwd — 密码管理

```bash
passwd [选项] [用户名]    # 不加用户名则修改自己的密码
```

| 选项 | 含义 |
|------|------|
| `-l` | 锁定用户密码（禁止登录，比 usermod -L 更底层） |
| `-u` | 解锁用户密码 |
| `-d` | 删除密码（允许无密码登录，危险！） |
| `-S` | 显示密码状态（Status） |
| `-e` | 强制用户下次登录时修改密码（expire） |
| `-n 天数` | 设置密码最短使用天数 |
| `-x 天数` | 设置密码最长使用天数（过期天数） |
| `-w 天数` | 密码过期前多少天开始警告 |
| `-i 天数` | 密码过期后多少天锁定账号 |

```bash
# 查看密码状态
sudo passwd -S zhangsan
# 输出：zhangsan P 07/22/2026 0 99999 7 -1
#       ^用户名  ^状态  ^最后修改   ^最短 ^最长 ^警告 ^不活跃

# 强制用户下次登录修改密码
sudo passwd -e zhangsan

# 设置密码 90 天过期
sudo passwd -x 90 zhangsan

# 锁定/解锁密码
sudo passwd -l zhangsan     # 锁定
sudo passwd -u zhangsan     # 解锁
```

### 7.6 chsh — 更改登录 Shell

```bash
# 查看当前 Shell
echo $SHELL

# 查看系统可用的 Shell
cat /etc/shells

# 更改当前用户的登录 Shell
chsh -s /bin/zsh

# 更改指定用户的 Shell（需 root）
sudo chsh -s /bin/bash zhangsan
```

### 7.7 删除用户：userdel vs deluser

```bash
# userdel（通用）
sudo userdel 用户名          # 只删除用户
sudo userdel -r 用户名       # 同时删除家目录和邮件池

# deluser（Debian/Ubuntu 友好封装）
sudo deluser 用户名           # 删除用户
sudo deluser --remove-home 用户名   # 同时删除家目录
sudo deluser 用户名 组名      # 将用户从指定组中移除
```

### 7.8 用户组管理

```bash
# 添加组
sudo groupadd 组名           # 基础版
sudo addgroup 组名           # Debian/Ubuntu 封装版

# 删除组
sudo groupdel 组名
sudo delgroup 组名

# 查看用户所属的组
groups 用户名                # 列出用户的所有组
id 用户名                    # 同时显示 UID、GID 和所有组 ID

# 修改组属性
sudo groupmod -n 新组名 旧组名   # 重命名组
sudo groupmod -g 2000 组名      # 修改 GID
```

### 7.9 为用户添加 sudo 权限

```bash
# 方法1：加入 sudo 组（Debian/Ubuntu 常用）
sudo adduser 用户名 sudo
# 或
sudo usermod -aG sudo 用户名

# 方法2：加入 wheel 组（CentOS/RHEL 常用）
sudo usermod -aG wheel 用户名

# 方法3：编辑 /etc/sudoers（需要 visudo，不要直接编辑）
sudo visudo
# 添加一行：用户名 ALL=(ALL:ALL) ALL

# 方法4：在 /etc/sudoers.d/ 下创建独立文件（推荐）
echo "用户名 ALL=(ALL:ALL) ALL" | sudo tee /etc/sudoers.d/用户名
sudo chmod 440 /etc/sudoers.d/用户名
```

### 7.10 查看用户信息的命令

```bash
# who：查看当前登录的用户
who
# 输出：laoli   tty1     Jul 22 09:30
#       zhangsan pts/0  Jul 22 10:15 (192.168.1.100)

# whoami：当前有效用户名
whoami

# id：显示用户和组 ID 信息
id                  # 当前用户
id zhangsan         # 指定用户
# 输出：uid=1001(zhangsan) gid=1001(zhangsan) groups=1001(zhangsan),27(sudo),999(docker)

# last：查看最近的登录记录
last                # 所有用户的登录记录
last zhangsan       # 指定用户
last -n 10          # 最近 10 条记录

# lastb：查看登录失败的记录（需要 root）
sudo lastb

# w：显示谁在登录以及在做什么
w

# users：简洁地列出登录用户
users
```

---

## 8. 补充工具命令

### 8.1 file — 查看文件类型

file 通过读取文件的 magic number（魔数）来判断文件类型，不依赖文件扩展名。

```bash
# 基本用法
file 文件名

# 示例
file /bin/ls
# 输出：/bin/ls: ELF 64-bit LSB executable, x86-64, dynamically linked...

file script.sh
# 输出：script.sh: Bourne-Again shell script, ASCII text executable

file unknown.bin
# 输出：unknown.bin: gzip compressed data

file image.png
# 输出：image.png: PNG image data, 1920 x 1080, 8-bit/color RGBA

# 查看目录类型
file /home
# 输出：/home: directory

# 批量查看
file /bin/* | head -5
```

### 8.2 sort — 排序

```bash
sort [选项] 文件

# 基本排序（按字母顺序）
sort names.txt

# 数字排序
sort -n numbers.txt

# 反向排序
sort -r file.txt

# 按指定列排序（-k）
ls -l | sort -k5 -n          # 按第 5 列（文件大小）数字排序

# 去重排序
sort -u file.txt             # 排序并去除重复行

# 忽略大小写
sort -f file.txt

# 检查文件是否已排序
sort -c file.txt

# 常用组合
du -sh * | sort -hr          # 按大小排序目录/文件
ps aux | sort -k3 -rn | head # 按 CPU 使用率排序进程
```

### 8.3 uniq — 去重

> 注意：uniq 只能去除**相邻的**重复行，通常配合 sort 使用。

```bash
# 统计每行出现次数
sort file.txt | uniq -c

# 只显示重复的行
sort file.txt | uniq -d

# 只显示不重复的行
sort file.txt | uniq -u

# 典型用法：日志分析
grep "ERROR" app.log | awk '{print $5}' | sort | uniq -c | sort -rn
# 统计各种 ERROR 的出现次数，从多到少排列

# 找出重复的 IP 地址
awk '{print $1}' access.log | sort | uniq -d
```

### 8.4 cut — 截取文本列

```bash
cut [选项] 文件

# 按字符截取（-c）
echo "abcdefghij" | cut -c 3-5       # 输出：cde
echo "abcdefghij" | cut -c 1,3,5     # 输出：ace

# 按分隔符截取（-d 指定分隔符，-f 指定字段）
cut -d ':' -f 1 /etc/passwd          # 截取用户名（第 1 列）
cut -d ':' -f 1,7 /etc/passwd        # 截取用户名和 Shell

# 截取范围
cut -d ',' -f 1-3 data.csv           # 第 1 到第 3 列
cut -d ',' -f 2- data.csv            # 第 2 列到结尾

# 实用示例
# 查看所有用户的 Shell
cut -d ':' -f 1,7 /etc/passwd | sort

# 从 CSV 提取指定列
cut -d ',' -f 2,4 employees.csv

# 配合管道
ls -l | tr -s ' ' | cut -d ' ' -f 5,9  # 提取文件大小和文件名
```

---

## 9. 命令速查总表

### 链接与文件操作

| 命令 | 功能 | 示例 |
|------|------|------|
| `ls -i` | 查看 inode 编号 | `ls -i file.txt` |
| `stat` | 查看文件元数据 | `stat file.txt` |
| `file` | 查看文件类型 | `file unknown.bin` |
| `ln` | 创建硬链接 | `ln a.txt b.txt` |
| `ln -s` | 创建软链接 | `ln -s a.txt b.txt` |
| `cp` | 复制文件 | `cp src dst` |
| `cp -r` | 复制目录 | `cp -r dir1 dir2` |
| `cp -a` | 归档复制 | `cp -a src/ backup/` |
| `cp -i` | 交互式复制 | `cp -i a.txt b.txt` |
| `cp -u` | 增量复制 | `cp -u src/ backup/` |
| `cp -p` | 保留属性 | `cp -p a.txt b.txt` |

### 打包与压缩

| 命令 | 功能 | 示例 |
|------|------|------|
| `tar -cf` | 打包 | `tar -cf a.tar dir/` |
| `tar -tvf` | 查看包内容 | `tar -tvf a.tar` |
| `tar -xf` | 解包 | `tar -xf a.tar` |
| `tar -xf -C` | 解包到指定目录 | `tar -xf a.tar -C /dest/` |
| `tar -czf` | gzip 压缩打包 | `tar -czf a.tar.gz dir/` |
| `tar -cjf` | bzip2 压缩打包 | `tar -cjf a.tar.bz2 dir/` |
| `tar -cJf` | xz 压缩打包 | `tar -cJf a.tar.xz dir/` |
| `tar -tzf` | 查看压缩包 | `tar -tzf a.tar.gz` |

### 查找与搜索

| 命令 | 功能 | 示例 |
|------|------|------|
| `find -name` | 按文件名查找 | `find / -name "*.txt"` |
| `find -type f` | 按文件类型 | `find . -type f -name "*.py"` |
| `find -size` | 按文件大小 | `find . -size +100M` |
| `find -mtime` | 按修改时间 | `find . -mtime -7` |
| `find -exec` | 查找并执行 | `find . -name "*.tmp" -exec rm {} \;` |
| `find \| xargs` | 查找并批处理 | `find . -name "*.log" -print0 \| xargs -0 rm` |
| `grep` | 文本搜索 | `grep "error" file.txt` |
| `grep -i` | 忽略大小写 | `grep -i "error" log` |
| `grep -r` | 递归搜索 | `grep -r "TODO" ~/project/` |
| `grep -v` | 反向匹配 | `grep -v "^#" config` |
| `grep -c` | 计数 | `grep -c "ERROR" log` |
| `grep -l` | 只列文件名 | `grep -rl "import re" .` |
| `grep -n` | 显示行号 | `grep -n "def" *.py` |
| `grep -E` | 扩展正则 | `grep -E "err\|warn" log` |
| `grep -F` | 固定字符串 | `grep -F "[ERROR]" log` |
| `sort` | 排序 | `sort -n file.txt` |
| `uniq -c` | 去重并统计 | `sort file.txt \| uniq -c` |
| `cut -d -f` | 截取文本列 | `cut -d ':' -f 1 /etc/passwd` |

### 用户管理

| 命令 | 功能 | 说明 |
|------|------|------|
| `sudo` | 临时提权 | `sudo command` |
| `su` | 切换用户 | `su - username`（- 表示加载环境） |
| `useradd` | 添加用户 | `useradd -m -s /bin/bash user`（脚本化） |
| `adduser` | 添加用户 | 交互式，建家目录+设密码，推荐日常用 |
| `usermod` | 修改用户 | `usermod -aG sudo user`（追加组） |
| `passwd` | 密码管理 | `passwd -S user`（查看状态） |
| `chsh` | 更改 Shell | `chsh -s /bin/zsh` |
| `userdel` | 删除用户 | `userdel -r user`（同时删家目录） |
| `deluser` | 删除用户 | `deluser --remove-home user` |
| `groupadd` | 添加组 | `groupadd 组名` |
| `groupdel` | 删除组 | `groupdel 组名` |
| `groups` | 查看用户组 | `groups user` |
| `id` | 显示 UID/GID | `id user` |
| `who` | 当前登录用户 | `who` |
| `whoami` | 当前用户名 | `whoami` |
| `last` | 登录记录 | `last -n 10` |
| `w` | 谁在做什么 | `w` |

---

*笔记优化日期：2026-07-22 | 原始转写：Whisper tiny*
