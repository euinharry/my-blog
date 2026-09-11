---
title: "第13讲：Shell脚本编程（上）"
date: 2026-09-11T09:18:00+08:00
draft: false
description: "Shell脚本 = 用砖头搭建起来的房子"
series: ["Linux 入门"]
series_order: 9
categories: ["技术笔记"]
tags: ["Linux", "Shell", "脚本"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P14)
> **标题**：第13讲 ,  Shell脚本编程（上）
> **时长**：15分23秒
> **转写方式**：Whisper tiny (CPU) 语音识别
> **笔记补充**：在原视频内容基础上，额外补充了变量、条件判断、循环、函数、调试等核心 Shell 编程知识点及实操示例

---

## 1. Shell脚本简介

### 1.1 Shell脚本是什么？

> **Shell脚本 = Shell命令按照一定语法组成的程序文件**

类比理解：
```
Shell命令 = 一块一块的砖头
Shell脚本 = 用砖头搭建起来的房子
```

### 1.2 类比Windows批处理

Linux的Shell脚本类似于Windows的**批处理文件（.bat）**：
- 包含一条或多条命令
- 执行时按顺序逐条运行
- 可简化日常开发中的重复性工作

### 1.3 Shell脚本的功能

通过组合各种Shell命令，Shell脚本可以完成丰富的功能：
- 启动/停止软件
- 系统性能监控
- 系统日志分析
- 自动化任务

---

## 2. Shell命令的本质

### 2.1 内建命令 vs 外部命令

| 类型 | 说明 | 示例 | 特点 |
|------|------|------|------|
| **内建命令** | Shell自身实现的命令 | cd, pwd, echo, type | 启动时加载到内存，运行速度快 |
| **外部命令** | 由其他应用程序实现的命令 | ifconfig, ls, grep | 需从硬盘加载到内存再执行，每次执行需创建子进程 |

> 常见的内建命令还包括：`export`, `read`, `source`, `alias`, `unalias`, `exit`, `history` 等。可以用 `type 命令名` 来判断一个命令是内建还是外部。

### 2.2 判断命令类型 ,  type命令

```bash
type cd         # 输出：cd 是 shell 内建
type pwd        # 输出：pwd 是 shell 内建
type ifconfig   # 输出：ifconfig 是 /sbin/ifconfig（外部命令）
type ls         # ls 是 `ls --color=auto' 的别名（同时也是外部命令）
```

### 2.3 Shell查找外部命令的机制

**问题**：执行外部命令时只指定了名字，Shell如何找到对应的应用程序？

**两种可能方案**：
1. 遍历文件系统所有目录 ,  效率太低（不可行）
2. **通过PATH环境变量** ,  Shell预设了命令搜索路径（实际方案）

### 2.4 PATH环境变量

```bash
echo $PATH
# 输出示例：/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:...
```

- `PATH` 中记录了多个目录路径，用 `:` 分隔
- 执行外部命令时，Shell按 `PATH` 中的顺序逐个目录查找
- 找到则创建新进程执行；找不到则报错 `"command not found"`

### 2.5 命令执行流程总结

```
用户在终端输入命令
        |
Shell先去内存查找【内建命令】
   |--- 找到 -> 直接执行（不创建子进程）
   |--- 没找到 -> 去PATH环境变量中逐目录查找【外部命令】
         |--- 找到对应程序 -> 创建子进程执行
         |--- 没找到 -> 报错 "command not found"
```

---

## 3. 小实验：将C程序变成Shell命令

### 实验步骤

```c
// 1. 编写C程序 hello.c
#include <stdio.h>
int main() {
    printf("hello world\n");
    return 0;
}
```

```bash
# 2. 编译成可执行文件
gcc hello.c -o hello

# 3. 验证运行
./hello     # 输出：hello world

# 4. 复制到PATH中的目录
sudo cp hello /usr/bin/

# 5. 现在hello就是一个Shell命令了！
hello       # 输出：hello world
```

### 实验原理

将可执行文件放到 `/usr/bin/`（PATH 中已有的目录）后，Shell 在查找外部命令时就能找到 `hello` 这个"新命令"。本质上，所谓的"外部命令"就是一个存放在 `PATH` 目录中的可执行文件。

---

## 4. 编译型语言 vs 解析型语言

| 对比维度 | 编译型语言（C语言） | 解析型语言（Shell） |
|----------|-------------------|---------------------|
| **执行方式** | 需编译器编译成二进制机器码后执行 | 由解析器逐行解析执行，无需提前编译 |
| **工具** | gcc / clang 等编译器 | Shell解析器（bash / sh / dash 等） |
| **运行速度** | 快（直接执行机器码） | 较慢（多了一层解析过程） |
| **代表语言** | C、C++、Go、Rust | Shell、Python、JavaScript、Perl |

示例对比：
```bash
# C语言过程：编写 -> 编译 -> 执行（三步）
gcc hello.c -o hello   # 第一步：编译源文件生成二进制
./hello                # 第二步：执行二进制文件

# Shell脚本过程：编写 -> 执行（两步）
chmod +x hello.sh      # 第一步：添加执行权限（仅一次）
./hello.sh             # 第二步：直接执行（由bash解析）
```

---

## 5. Shell解析器

### 5.1 常见Shell解析器列表

记录在 `/etc/shells` 文件中：

```bash
cat /etc/shells
# 输出示例：
# /bin/sh
# /bin/bash
# /bin/rbash
# /bin/dash
# /usr/bin/tmux   （某些系统可能包含）
```

### 5.2 各解析器简介

| 解析器 | 全称 | 说明 |
|--------|------|------|
| **sh** | Bourne Shell | Unix 最早的 Shell，功能最基础 |
| **bash** | Bourne Again Shell | sh 的增强版，Linux 最常用，功能最丰富 |
| **dash** | Debian Almquist Shell | 轻量级，启动快，Ubuntu/Debian 的 `/bin/sh` 默认指向它 |
| **zsh** | Z Shell | 高度可定制，macOS 默认 Shell，支持插件系统 |
| **rbash** | Restricted Bash | bash 的受限版本，限制部分危险操作 |

### 5.3 Ubuntu默认解析器

Ubuntu系统默认使用 **bash**（Bourne Again Shell）。交互式终端的默认Shell也是bash。

---

## 6. 第一个Shell脚本

### 6.1 编写脚本

```bash
#!/bin/bash           # 第1行：Shebang，指定解析器为bash
echo "hello world"    # 脚本内容：输出字符串到终端
```

### 6.2 Shebang（#!）详解

- `#!` 必须写在脚本的**第一行**，前面不能有空行或空格
- `#!` 后的路径指定了执行此脚本使用的解析器
- 常见写法：
  - `#!/bin/bash` ,  用bash解析（推荐，功能最全）
  - `#!/bin/sh` ,  用sh解析（兼容性最好）
  - `#!/usr/bin/env bash` ,  从PATH中查找bash（可移植性更好，但不推荐在系统脚本中使用）
  - `#!/usr/bin/python3` ,  用Python解析（Python脚本）

### 6.3 添加执行权限

```bash
chmod +x hello.sh     # 推荐：仅添加执行权限（等同 chmod u+x）
# 或
chmod 755 hello.sh    # 更精确：rwxr-xr-x
ls -l hello.sh        # 确认权限位
# 输出示例：-rwxr-xr-x 1 user user 32 Jul 22 10:00 hello.sh
```

> **安全提示**：避免使用 `chmod 777`，这会赋予所有用户完全权限，存在安全隐患。生产环境中建议使用 `chmod 755`。

### 6.4 四种启动方式

| 序号 | 方式 | 命令 | 说明 |
|------|------|------|------|
| 1 | 直接运行 | `./hello.sh` | 需有执行权限，创建子进程执行 |
| 2 | 指定解释器 | `bash hello.sh` | 无需执行权限，创建子进程执行 |
| 3 | source命令 | `source hello.sh` | 在当前Shell进程中执行，不创建子进程 |
| 4 | 点命令 | `. hello.sh` | source的简写形式，功能完全相同 |

### 6.5 四种方式的本质区别

这四种方式的核心区别在于**是否创建子进程**，这直接影响了**环境变量的作用范围**：

```
方式1和方式2：创建子Shell进程执行
    脚本运行在子进程中 -> 脚本中export的变量在脚本结束后消失
    父Shell不受脚本中变量修改的影响

方式3和方式4：在当前Shell进程中执行
    脚本运行在当前Shell中 -> 脚本中的变量修改会保留在当前Shell
    常用于加载环境变量配置（如 source ~/.bashrc）
```

> **实战技巧**：修改了 `.bashrc` 或 `/etc/profile` 后，用 `source ~/.bashrc` 让配置立即生效。

---

## 7. 本讲要点总结

1. **Shell脚本** = 命令按语法组合的程序文件
2. **内建命令**（cd/pwd/echo）速度快，**外部命令**（ifconfig/ls/grep）通过PATH查找
3. **PATH环境变量** 记录了外部命令的搜索路径
4. C程序放入 `/usr/bin/` 即可变成Shell命令（本质是放在PATH目录中的可执行文件）
5. C = 编译型语言（编译器 -> 二进制），Shell = 解析型语言（解析器直接执行）
6. **Shebang** `#!/bin/bash` 指定脚本解析器，必须写在第一行
7. Shell脚本需加 `chmod +x` 执行权限，有4种启动方式（直接运行/指定解释器/source/点命令）

---

## 8. Shell变量

### 8.1 变量定义

Shell中变量定义**等号两边不能有空格**，这是初学者最容易犯的错误。

```bash
# 正确：等号两边无空格
name="Linux"
count=10
pi=3.14

# 错误：等号两边有空格会导致语法错误
name = "Linux"    # Shell会认为 name 是一个命令
count= 10         # 同样会报错
```

### 8.2 变量引用

```bash
name="shell"
echo $name        # 输出：shell
echo ${name}      # 输出：shell（花括号方式，更明确）

# 花括号的必要场景：变量名与后续字符连写时
echo "hello${name}world"    # 输出：helloshellworld
echo "hello$nameworld"      # 输出：hello（Shell会找名为nameworld的变量，找不到输出空）
```

### 8.3 引号的区别

| 引号类型 | 示例 | 说明 |
|----------|------|------|
| **双引号** `"` | `"$name"` | 允许变量替换和命令替换 |
| **单引号** `'` | `'$name'` | 所有字符原样输出，不解析变量 |
| **反引号** `` ` `` | `` `date` `` | 命令替换（旧语法，不推荐） |
| **$()** | `$(date)` | 命令替换（新语法，推荐） |

```bash
name="Linux"
echo "Hello $name"     # 输出：Hello Linux
echo 'Hello $name'     # 输出：Hello $name（变量未展开）
echo "Today is $(date +%Y-%m-%d)"   # 输出：Today is 2026-07-22
```

### 8.4 只读变量和删除变量

```bash
name="original"
readonly name          # 设为只读，后续不可修改
# name="new"           # 报错：name: readonly variable

age=18
unset age              # 删除变量
echo $age              # 输出空（变量已不存在）
```

### 8.5 常见环境变量

| 变量 | 含义 | 示例值 |
|------|------|--------|
| `$HOME` | 当前用户主目录 | `/home/user` |
| `$USER` | 当前用户名 | `root` 或 `ubuntu` |
| `$PATH` | 命令搜索路径 | `/usr/local/sbin:/usr/bin:...` |
| `$SHELL` | 当前Shell路径 | `/bin/bash` |
| `$PWD` | 当前工作目录 | `/home/user/project` |
| `$OLDPWD` | 上一个工作目录 | `/home/user` |

```bash
echo "当前用户：$USER"
echo "主目录：$HOME"
echo "当前Shell：$SHELL"
```

### 8.6 通过export导出环境变量

```bash
# 局部变量：仅在当前Shell可见
my_var="hello"

# 导出为环境变量：当前Shell及所有子进程可见
export my_var="hello"

# 简写方式
export PATH=$PATH:/new/path
```

---

## 9. 特殊变量与位置参数

### 9.1 位置参数

| 变量 | 含义 |
|------|------|
| `$0` | 脚本自身的名称 |
| `$1` | 第一个参数 |
| `$2` | 第二个参数 |
| `$3`~`$9` | 第三到第九个参数 |
| `${10}` | 第十个参数（必须加花括号） |

```bash
#!/bin/bash
# 脚本名：greet.sh
echo "脚本名称：$0"
echo "第一个参数：$1"
echo "第二个参数：$2"
```

运行效果：
```bash
./greet.sh Alice Bob
# 输出：
# 脚本名称：./greet.sh
# 第一个参数：Alice
# 第二个参数：Bob
```

### 9.2 参数数量与参数列表

| 变量 | 含义 |
|------|------|
| `$#` | 传递给脚本的参数个数 |
| `$@` | 所有参数列表，每个参数独立（推荐使用） |
| `$*` | 所有参数列表，合并为一个字符串 |
| `"$@"` | 每个参数独立加引号（最安全） |
| `"$*"` | 所有参数合并为一个带引号的字符串 |

> `$@` 和 `$*` 的区别在带引号时最为关键：`"$@"` 保留参数边界，`"$*"` 将所有参数当作一个整体。

```bash
#!/bin/bash
# 脚本名：params.sh
echo "参数个数：$#"
echo "所有参数(@)：$@"

# 遍历每个参数
for arg in "$@"; do
    echo "参数：$arg"
done
```

### 9.3 进程相关特殊变量

| 变量 | 含义 |
|------|------|
| `$?` | 上一条命令的退出状态码（0=成功，非0=失败） |
| `$$` | 当前Shell进程的PID |
| `$!` | 后台运行的最后一个进程的PID |
| `$-` | 当前Shell的选项标志 |

```bash
ls /nonexist
echo $?       # 输出：2（非0，表示命令执行失败）

ls /
echo $?       # 输出：0（命令执行成功）

echo "当前进程PID：$$"
```

### 9.4 退出状态码约定

| 状态码 | 含义 |
|--------|------|
| `0` | 命令执行成功 |
| `1` | 一般性错误 |
| `2` | 误用Shell命令 |
| `126` | 命令不可执行（权限问题） |
| `127` | 命令未找到 |
| `128+n` | 命令被信号n终止 |

```bash
#!/bin/bash
# 脚本中使用 exit 返回状态码
if [ $# -lt 1 ]; then
    echo "用法：$0 <文件名>"
    exit 1          # 参数不足，返回错误码1
fi
echo "处理文件：$1"
exit 0              # 正常结束，返回0
```

---

## 10. 条件判断

### 10.1 test命令与 [ ] 语法

Shell中条件判断有两种等价写法：

```bash
# 写法1：test 命令
test -f /etc/passwd && echo "文件存在"

# 写法2：[ ] 语法（推荐，可读性更好）
[ -f /etc/passwd ] && echo "文件存在"
```

> **注意**：`[` 是一个命令，不是括号语法。`[` 和 `]` 两边必须有空格！`[$a` 会报错。

### 10.2 数值比较

| 运算符 | 含义 | 示例 |
|--------|------|------|
| `-eq` | 等于 (equal) | `[ "$a" -eq "$b" ]` |
| `-ne` | 不等于 (not equal) | `[ "$a" -ne "$b" ]` |
| `-gt` | 大于 (greater than) | `[ "$a" -gt "$b" ]` |
| `-lt` | 小于 (less than) | `[ "$a" -lt "$b" ]` |
| `-ge` | 大于等于 (greater equal) | `[ "$a" -ge "$b" ]` |
| `-le` | 小于等于 (less equal) | `[ "$a" -le "$b" ]` |

```bash
count=10
if [ "$count" -gt 5 ]; then
    echo "count大于5"
fi
```

### 10.3 字符串比较

| 运算符 | 含义 | 示例 |
|--------|------|------|
| `=` | 字符串相等 | `[ "$a" = "$b" ]` |
| `!=` | 字符串不相等 | `[ "$a" != "$b" ]` |
| `-z` | 字符串长度为0（空串） | `[ -z "$a" ]` |
| `-n` | 字符串长度不为0（非空） | `[ -n "$a" ]` |

```bash
name=""
if [ -z "$name" ]; then
    echo "变量name为空"
fi
```

> **最佳实践**：比较字符串时始终**用双引号包裹变量**，防止变量为空时语法错误。

### 10.4 文件测试

| 运算符 | 含义 |
|--------|------|
| `-f` | 是否为普通文件 |
| `-d` | 是否为目录 |
| `-e` | 文件是否存在（不区分类型） |
| `-r` | 文件是否可读 |
| `-w` | 文件是否可写 |
| `-x` | 文件是否可执行 |
| `-s` | 文件是否非空（大小>0） |
| `-L` | 是否为符号链接 |

```bash
file="/etc/passwd"
if [ -f "$file" ]; then
    echo "$file 是一个普通文件"
elif [ ! -e "$file" ]; then
    echo "$file 不存在"
fi
```

### 10.5 逻辑运算

```bash
# AND：-a（test语法）或 &&
[ "$a" -gt 5 -a "$b" -lt 10 ]            # test语法
[ "$a" -gt 5 ] && [ "$b" -lt 10 ]        # 更推荐：用 && 连接

# OR：-o（test语法）或 ||
[ "$a" -eq 0 -o "$b" -eq 0 ]             # test语法
[ "$a" -eq 0 ] || [ "$b" -eq 0 ]         # 更推荐：用 || 连接

# NOT：!
[ ! -f "/tmp/log" ] && echo "文件不存在"
```

### 10.6 if/elif/else 完整结构

```bash
#!/bin/bash
# 完整的条件判断示例

score=$1

if [ -z "$score" ]; then
    echo "用法：$0 <分数>"
    exit 1
elif [ "$score" -ge 90 ]; then
    echo "等级：优秀"
elif [ "$score" -ge 80 ]; then
    echo "等级：良好"
elif [ "$score" -ge 60 ]; then
    echo "等级：及格"
else
    echo "等级：不及格"
fi
```

### 10.7 [[ ]] 增强条件判断（bash专用）

```bash
# bash的 [[ ]] 比 [ ] 更强大、更安全
# 支持 && || 直接在内部使用
# 支持 =~ 正则匹配
# 无需对变量加引号防空格

str="hello world"
if [[ "$str" == hello* ]]; then
    echo "以hello开头"
fi

# 正则匹配
if [[ "$str" =~ ^he[l]{2}o ]]; then
    echo "匹配正则"
fi
```

### 10.8 case 多分支语句

```bash
#!/bin/bash
# case语句示例：根据用户输入执行不同操作

read -p "请输入选项(start/stop/restart/status): " action

case "$action" in
    start)
        echo "启动服务..."
        ;;
    stop)
        echo "停止服务..."
        ;;
    restart)
        echo "重启服务..."
        ;;
    status)
        echo "检查服务状态..."
        ;;
    *)
        echo "用法：$0 {start|stop|restart|status}"
        exit 1
        ;;
esac
```

---

## 11. 循环

### 11.1 for循环（遍历列表）

```bash
#!/bin/bash
# 方式1：遍历列表
for fruit in apple banana orange; do
    echo "水果：$fruit"
done

# 方式2：遍历命令输出
for user in $(cat /etc/passwd | cut -d: -f1); do
    echo "用户：$user"
done

# 方式3：遍历文件通配符
for file in /tmp/*.log; do
    echo "找到日志文件：$file"
done
```

### 11.2 for循环（C语言风格）

```bash
#!/bin/bash
# 经典的三段式for循环
for (( i=1; i<=5; i++ )); do
    echo "第 $i 次循环"
done

# 输出：
# 第 1 次循环
# 第 2 次循环
# 第 3 次循环
# 第 4 次循环
# 第 5 次循环
```

### 11.3 while循环

```bash
#!/bin/bash
# 计数器式while循环
count=1
while [ "$count" -le 5 ]; do
    echo "count = $count"
    count=$((count + 1))
done

# 逐行读取文件
while IFS= read -r line; do
    echo "行内容：$line"
done < /etc/hosts
```

> `IFS=` 防止去除行首空白，`-r` 防止反斜杠转义被解释。

### 11.4 until循环

```bash
#!/bin/bash
# until：条件为假时执行，条件为真时退出（与while相反）
count=1
until [ "$count" -gt 5 ]; do
    echo "count = $count"
    count=$((count + 1))
done
```

### 11.5 循环控制：break 和 continue

```bash
#!/bin/bash
# break：跳出整个循环
for i in {1..10}; do
    if [ "$i" -eq 5 ]; then
        echo "遇到5，跳出循环"
        break
    fi
    echo "数字：$i"
done

# continue：跳过本次循环，继续下一次
for i in {1..5}; do
    if [ "$i" -eq 3 ]; then
        continue
    fi
    echo "数字：$i"        # 不会输出3
done
```

---

## 12. 用户输入 ,  read命令

### 12.1 基本用法

```bash
#!/bin/bash
echo "请输入你的名字："
read name
echo "你好，$name！"
```

### 12.2 常用选项

| 选项 | 说明 | 示例 |
|------|------|------|
| `-p` | 显示提示信息 | `read -p "输入：" name` |
| `-s` | 静默模式（不显示输入内容，用于密码） | `read -s -p "密码：" pass` |
| `-t` | 超时时间（秒），超时后自动跳过 | `read -t 5 -p "5秒内输入：" name` |
| `-n` | 限制读取字符数 | `read -n 1 -p "按任意键继续..."` |
| `-a` | 将输入存入数组 | `read -a arr` |
| `-r` | 不解释反斜杠转义（推荐始终使用） | `read -r line` |

### 12.3 综合示例

```bash
#!/bin/bash
# 模拟登录提示

read -p "用户名：" username
read -s -p "密码：" password
echo                          # read -s后手动换行
echo ""

if [ "$username" = "admin" ] && [ "$password" = "123456" ]; then
    echo "登录成功！"
else
    echo "用户名或密码错误"
fi
```

---

## 13. 函数

### 13.1 函数定义

```bash
# 方式1：function 关键字（传统写法）
function hello {
    echo "Hello, $1"
}

# 方式2：函数名+括号（POSIX兼容，更推荐）
hello() {
    echo "Hello, $1"
}
```

### 13.2 函数调用与参数传递

```bash
#!/bin/bash
# 函数内部使用 $1 $2 ... 获取参数（不是脚本级的 $1）
greet() {
    echo "你好，$1！今天日期是 $(date +%Y-%m-%d)"
}

# 调用函数，传递参数
greet "小明"
greet "小红"

# 输出：
# 你好，小明！今天日期是 2026-07-22
# 你好，小红！今天日期是 2026-07-22
```

### 13.3 函数返回值

```bash
#!/bin/bash
# Shell函数通过 echo 返回字符串结果，通过 return 返回状态码（0-255）

# 方式1：用 echo 返回计算结果
add() {
    echo $(( $1 + $2 ))
}

result=$(add 5 3)
echo "5 + 3 = $result"

# 方式2：用 return 返回状态码
file_exists() {
    [ -f "$1" ] && return 0 || return 1
}

if file_exists "/etc/passwd"; then
    echo "文件存在"
fi
```

> Shell函数**不能**像C/Python那样直接 `return 计算结果`。`return` 只能返回 0-255 的状态码。需要返回计算结果时，使用 `echo` 配合 `$(...)` 命令替换获取。

### 13.4 局部变量

```bash
#!/bin/bash
my_func() {
    local name="局部变量"     # local声明为函数内局部变量
    age=18                    # 没有local则为全局变量
    echo "函数内：name=$name"
}

my_func
echo "函数外：name=$name"     # 输出空（局部变量在函数外不可见）
echo "函数外：age=$age"       # 输出 18（全局变量函数外可见）
```

---

## 14. 脚本调试

### 14.1 bash -x（执行跟踪）

```bash
# 执行脚本时显示每条被执行的命令
bash -x script.sh

# 输出示例（每行以 + 开头）：
# + name=Linux
# + echo 'Hello Linux'
# Hello Linux
```

### 14.2 set -x 和 set +x（局部调试）

```bash
#!/bin/bash
# 在脚本中使用 set -x 开启调试，set +x 关闭调试

echo "这行不会被跟踪"

set -x              # 开启调试
name="Linux"
echo "Hello $name"  # 这行会被跟踪显示
set +x              # 关闭调试

echo "这行也不会被跟踪"
```

### 14.3 其他调试选项

| 命令/选项 | 说明 |
|-----------|------|
| `bash -n script.sh` | 仅检查语法错误，不执行脚本 |
| `bash -v script.sh` | 显示读入的每一行（verbose模式） |
| `bash -xv script.sh` | 同时使用 -x 和 -v |
| `set -e` | 任何命令出错（退出码非0）时立即退出脚本 |
| `set -u` | 使用未定义变量时报错退出 |
| `set -o pipefail` | 管道中任一命令失败则整个管道失败 |

```bash
#!/bin/bash
# 常用调试头：在脚本开头加上以下三行
set -e              # 遇错即停
set -u              # 未定义变量报错
set -o pipefail     # 管道错误不隐藏
```

---

## 15. 综合脚本示例

### 15.1 系统信息监控脚本

```bash
#!/bin/bash
# sysinfo.sh ,  快速查看系统关键信息

echo "==================== 系统信息 ===================="
echo "主机名：$(hostname)"
echo "内核版本：$(uname -r)"
echo "系统运行时间：$(uptime -p)"
echo ""

echo "==================== CPU信息 ===================="
echo "CPU型号：$(grep 'model name' /proc/cpuinfo | head -1 | cut -d: -f2)"
echo "CPU核心数：$(nproc)"
echo ""

echo "==================== 内存信息 ===================="
free -h

echo ""
echo "==================== 磁盘信息 ===================="
df -h /

echo ""
echo "==================== 网络信息 ===================="
ip addr show | grep -E 'inet ' | grep -v 127.0.0.1
```

### 15.2 简易文件备份脚本

```bash
#!/bin/bash
# backup.sh ,  备份指定目录到 /backup/ 下

set -e

# ========== 配置区域 ==========
SOURCE_DIR="${1:-/home/user/data}"       # 默认备份源
BACKUP_DIR="/backup"
RETENTION_DAYS=7                         # 保留最近7天的备份
# =============================

# 检查源目录
if [ ! -d "$SOURCE_DIR" ]; then
    echo "错误：源目录 $SOURCE_DIR 不存在"
    exit 1
fi

# 创建备份目录
mkdir -p "$BACKUP_DIR"

# 生成带时间戳的备份文件名
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/backup_$(basename "$SOURCE_DIR")_$TIMESTAMP.tar.gz"

# 执行备份
echo "正在备份 $SOURCE_DIR -> $BACKUP_FILE ..."
tar -czf "$BACKUP_FILE" -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

if [ $? -eq 0 ]; then
    echo "备份完成：$BACKUP_FILE"
    echo "备份大小：$(du -h "$BACKUP_FILE" | cut -f1)"
else
    echo "备份失败！"
    exit 1
fi

# 清理过期备份
echo "清理 ${RETENTION_DAYS} 天前的旧备份..."
find "$BACKUP_DIR" -name "backup_*.tar.gz" -mtime +$RETENTION_DAYS -delete
echo "清理完成"

exit 0
```

### 15.3 批量创建用户脚本

```bash
#!/bin/bash
# create_users.sh ,  从文本文件批量创建用户

USERLIST="${1:-users.txt}"

if [ ! -f "$USERLIST" ]; then
    echo "错误：用户列表文件 $USERLIST 不存在"
    echo "文件格式：每行一个用户名"
    exit 1
fi

# 需要root权限
if [ "$EUID" -ne 0 ]; then
    echo "请使用 root 权限运行此脚本"
    exit 1
fi

while read -r username; do
    # 跳过空行和注释行
    [ -z "$username" ] && continue
    [[ "$username" == \#* ]] && continue

    # 检查用户是否已存在
    if id "$username" &>/dev/null; then
        echo "用户 $username 已存在，跳过"
        continue
    fi

    # 创建用户（同时创建同名组和主目录）
    useradd -m "$username"
    echo "用户 $username 创建成功"
    echo "$username:password123" | chpasswd

    # 强制首次登录修改密码
    chage -d 0 "$username"
done < "$USERLIST"

echo "所有用户创建完毕"
```

---

## 16. Shell脚本中的常用技巧

### 16.1 命令行参数解析

```bash
#!/bin/bash
# 使用 getopts 解析命令行选项

while getopts "hvf:" opt; do
    case "$opt" in
        h) echo "帮助信息"; exit 0 ;;
        v) echo "详细模式开启"; VERBOSE=1 ;;
        f) echo "文件参数：$OPTARG"; FILE="$OPTARG" ;;
        ?) echo "未知选项"; exit 1 ;;
    esac
done
```

### 16.2 颜色输出

```bash
#!/bin/bash
# ANSI颜色码
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'          # No Color (重置)

echo -e "${GREEN}[成功]${NC} 操作完成"
echo -e "${RED}[错误]${NC} 操作失败"
echo -e "${YELLOW}[警告]${NC} 磁盘空间不足"
```

### 16.3 路径相关的字符串操作

```bash
filepath="/home/user/docs/report.txt"

dirname=$(dirname "$filepath")       # /home/user/docs
basename=$(basename "$filepath")     # report.txt
filename="${basename%.*}"            # report（去掉扩展名）
extension="${basename##*.}"          # txt（去掉文件名部分）
```

### 16.4 trap信号捕获

```bash
#!/bin/bash
# trap 用于捕获信号，确保脚本退出前完成清理工作

cleanup() {
    echo "正在清理临时文件..."
    rm -f /tmp/mytemp_*.txt
    echo "清理完成，脚本退出"
}

# 捕获 EXIT（正常退出）、INT（Ctrl+C）、TERM（kill）信号
trap cleanup EXIT INT TERM

# 脚本主体
echo "脚本运行中（PID: $$），按 Ctrl+C 测试清理逻辑..."
sleep 60
```

---

## 17. Shell脚本编码规范建议

1. **Shebang** 必须写在第一行：`#!/bin/bash`
2. **变量**：使用 `${var}` 而非 `$var`，养成好习惯
3. **引号**：变量引用始终加双引号 `"$var"`，避免空值导致的错误
4. **错误处理**：脚本开头添加 `set -euo pipefail`
5. **缩进**：统一使用4个空格，不混用Tab
6. **注释**：关键逻辑和复杂命令必须注释
7. **函数**：功能模块化，单一职责
8. **退出码**：明确返回0（成功）或非0（失败）
9. **路径**：使用绝对路径或 `$(dirname "$0")` 获取脚本所在目录
10. **临时文件**：使用 `mktemp` 创建，用 `trap` 确保清理

---

*笔记生成日期：2026-07-22 | 转写：Whisper tiny | 笔记优化补充：Sisyphus-Junior*
