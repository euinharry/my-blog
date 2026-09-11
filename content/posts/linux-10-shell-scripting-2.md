---
title: "第14讲：Shell脚本编程（中）"
date: 2026-09-11T09:17:00+08:00
draft: false
description: "Shell变量有三种定义方式，各有不同的行为。"
series: ["Linux 入门"]
series_order: 10
categories: ["技术笔记"]
tags: ["Linux", "Shell", "脚本"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P15)
> **标题**：第14讲 — Shell脚本编程（中）
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. Shell变量的定义

Shell变量有三种定义方式，各有不同的行为。

| 方式 | 示例 | 特点 |
|------|------|------|
| **直接赋值** | `var=value` | 等号和值之间不能有空格；值不能包含空格或特殊字符 |
| **单引号** | `var='value'` | 原样输出，**不会解析**内部的变量引用（如 `$VAR`） |
| **双引号** | `var="value"` | **会解析**内部的变量引用（变量替换） |

### 1.1 变量命名规则

Shell 变量命名必须遵循以下规则：

| 规则 | 说明 | 正确示例 | 错误示例 |
|------|------|----------|----------|
| 字符组成 | 只能由字母、数字、下划线组成 | `my_var`、`name2` | `my-var`、`a@b` |
| 首字符限制 | 不能以数字开头 | `file1`、`_count` | `1file`、`2name` |
| 区分大小写 | `Name` 和 `name` 是两个不同的变量 | — | — |
| 关键字限制 | 不能使用 Shell 保留关键字 | `my_if` | `if`、`for`、`case` |

```bash
# 正确命名
user_name="zhangsan"
_file_count=10
dirPath1="/home/user"

# 错误命名
1st_name="zhangsan"     # 不能以数字开头
my-var="hello"           # 不能包含连字符（会被解析为减法）
user name="zhangsan"     # 不能包含空格
```

### 1.2 三种定义方式的对比演示

```bash
name="野火"

# 方式1：直接赋值（不能有空格和特殊字符）
var1=hello

# 方式2：单引号 — 原样输出
var2='hello $name'      # 输出：hello $name（$name 不会被替换）

# 方式3：双引号 — 变量替换
var3="hello $name"      # 输出：hello 野火（$name 会被替换为变量的值）
```

> **关键区别**：单引号里的 `$VAR` 保持原样输出；双引号里的 `$VAR` 会被替换为其实际值。

### 1.3 readonly — 定义只读变量

`readonly` 用于将变量设置为只读，设置后该变量不可被修改或删除。

```bash
readonly var="固定的值"

# 或者先定义再设置只读
name="野火"
readonly name

# 尝试修改只读变量会报错
name="新值"
# 输出：bash: name: readonly variable
```

```bash
# 实用场景：定义不可改变的常量
readonly PI=3.14159
readonly MAX_RETRY=3
readonly BASE_DIR="/opt/myapp"
```

### 1.4 declare / typeset — 声明变量属性

`declare` 和 `typeset` 是等价的命令，用于声明变量的类型和属性，控制变量的行为。`typeset` 是旧式写法，`declare` 是现代推荐写法。

| 选项 | 含义 | 示例 |
|------|------|------|
| `-i` | 声明为**整型**变量，赋值时自动进行算术求值 | `declare -i num=5+3` → `echo $num` 输出 `8` |
| `-r` | 声明为**只读**变量（等同于 `readonly`） | `declare -r CONST=100` |
| `-a` | 声明为**普通数组**（索引数组） | `declare -a arr=(a b c)` |
| `-A` | 声明为**关联数组**（键值对，类似字典） | `declare -A map; map[name]=zhangsan` |
| `-x` | 声明为**环境变量**（等同于 `export`） | `declare -x PATH="/usr/local/bin:$PATH"` |
| `-l` | 将值转为**小写**存储 | `declare -l lower="HELLO"` → `hello` |
| `-u` | 将值转为**大写**存储 | `declare -u upper="hello"` → `HELLO` |

```bash
# 整型变量：自动计算
declare -i result=10+5
echo $result            # 15（自动完成算术运算）

# 整型变量的好处：赋值字符串时自动转为 0
declare -i num
num="hello"
echo $num               # 0（字符串被转换为 0）

# 只读变量
declare -r VERSION="2.0"
VERSION="3.0"           # 报错：bash: VERSION: readonly variable

# 查看变量的属性和值
declare -p result       # 输出：declare -i result="15"

# 查看所有已声明的变量（包括类型属性）
declare
```

> **提示**：`declare -p 变量名` 可以查看变量的完整声明信息，包括类型属性和当前值，是调试时的常用手段。

---

## 2. Shell变量的使用

### 2.1 两种引用方式

| 方式 | 语法 | 建议 |
|------|------|------|
| 直接引用 | `$变量名` | 简单场景可用 |
| 加花括号 | `${变量名}` | **推荐使用**，可明确变量边界 |

### 2.2 为什么需要花括号？

```bash
str="hello"

echo $str          # hello
echo $strBBB       # 空（Shell 认为整个 $strBBB 是变量名，但该变量不存在）
echo ${str}BBB     # helloBBB 正确：花括号明确了变量名是 str
```

> **规则**：花括号 `${}` 用于明确变量的边界，避免变量名和后续字符粘连导致歧义。

### 2.3 变量扩展语法：设置默认值

这一组语法在变量**未定义或为空**时提供默认值，非常适合写健壮的脚本。

| 语法 | 含义 | 说明 |
|------|------|------|
| `${var:-default}` | 如果 var 未定义或为空，**使用** default 作为值 | 最常用，不改变 var 本身 |
| `${var:=default}` | 如果 var 未定义或为空，**赋值** default 给 var 并返回 | 会修改 var 的值 |
| `${var:?err_msg}` | 如果 var 未定义或为空，**报错**并显示 err_msg | 用于必填参数的校验 |
| `${var:+alt_val}` | 如果 var 已定义且非空，**使用** alt_val | 条件替换 |

```bash
# ${var:-default} — 提供默认值（不改原变量）
dir_name="${1:-/tmp}"          # 如果有传参就用，否则默认 /tmp
echo "目录：$dir_name"

name=""
greeting="Hello, ${name:-World}"   # name 为空，使用 World
echo $greeting               # Hello, World
echo $name                   # 仍然是空（name 本身未被修改）

# ${var:=default} — 赋值默认值（会修改原变量）
unset count
echo ${count:=0}             # 0（count 原本未定义，赋值为 0）
echo $count                  # 0（count 已被设置）

# ${var:?err_msg} — 必填校验，未定义则直接报错退出
filename="${1:?错误：缺少文件名参数}"
# 如果脚本没有传参数，输出：
# bash: 1: 错误：缺少文件名参数

# ${var:+alt_val} — 条件替换
DEBUG="true"
mode="${DEBUG:+debug mode}"  # DEBUG 非空，mode 被设置为 "debug mode"
echo $mode                   # debug mode
```

### 2.4 字符串操作

Shell 提供了丰富的字符串操作语法，都使用花括号 `${}` 包裹。

#### 2.4.1 获取字符串长度

```bash
str="hello world"
echo ${#str}         # 11（包含空格）
```

#### 2.4.2 截取子串

```bash
str="hello world"

echo ${str:0:5}      # hello（从索引 0 开始，取 5 个字符）
echo ${str:6}        # world（从索引 6 开始，取到末尾）

# 从右边数（推荐用括号包裹负数索引）
echo ${str:(-5)}     # world（从倒数第 5 个到末尾）
echo ${str:(-5):3}   # wor（从倒数第 5 个，取 3 个字符）
```

#### 2.4.3 前缀删除：`#` 和 `##`

`#` 用于从字符串**开头**删除匹配的最短/最长内容。

```bash
path="/home/user/config.txt"

echo ${path#*/}      # home/user/config.txt（删除最短匹配：/ → 删除第一个 /）
echo ${path##*/}     # config.txt（删除最长匹配：*/ → 删除到最后一个 /）

# 实用场景：提取文件名和路径
filepath="/var/log/app.log"
filename=${filepath##*/}    # app.log（提取文件名）
dirname=${filepath%/*}      # /var/log（提取目录名）
```

#### 2.4.4 后缀删除：`%` 和 `%%`

`%` 用于从字符串**末尾**删除匹配的最短/最长内容。

```bash
filename="archive.tar.gz"

echo ${filename%.*}      # archive.tar（删除最短 .* 匹配 → 删掉 .gz）
echo ${filename%%.*}     # archive（删除最长 .* 匹配 → 删掉 .tar.gz）

# 实用场景：提取文件扩展名和主名
file="backup.tar.gz"
ext=${file##*.}          # gz（扩展名）
basename=${file%%.*}     # backup（去掉所有扩展名）
```

#### 2.4.5 字符串替换：`/` 和 `//`

```bash
str="hello world, hello shell"

echo ${str/hello/hi}     # hi world, hello shell（只替换第一个）
echo ${str//hello/hi}    # hi world, hi shell（替换所有）

# 替换前缀：以某内容开头的
echo ${str/#hello/HI}    # HI world, hello shell（只替换开头匹配的）

# 替换后缀：以某内容结尾的
echo ${str/%shell/SHELL} # hello world, hello SHELL（只替换结尾匹配的）
```

#### 2.4.6 全部操作速查表

| 语法 | 含义 | 示例 |
|------|------|------|
| `${#var}` | 字符串长度 | `${#str}` → `11` |
| `${var:offset:length}` | 截取子串 | `${str:0:5}` → `hello` |
| `${var#pattern}` | 删除最短前缀匹配 | `${path#*/}` |
| `${var##pattern}` | 删除最长前缀匹配 | `${path##*/}` |
| `${var%pattern}` | 删除最短后缀匹配 | `${file%.*}` |
| `${var%%pattern}` | 删除最长后缀匹配 | `${file%%.*}` |
| `${var/old/new}` | 替换第一个匹配 | `${str/hello/hi}` |
| `${var//old/new}` | 替换所有匹配 | `${str//hello/hi}` |
| `${var/#old/new}` | 替换前缀匹配 | `${str/#hello/HI}` |
| `${var/%old/new}` | 替换后缀匹配 | `${str/%shell/SHELL}` |

---

## 3. 命令结果赋值给变量

将命令的执行结果保存到变量中，有两种写法：

### 3.1 反引号法（旧式）

```bash
var=`pwd`
echo $var    # 输出当前工作目录，如 /home/user
```

### 3.2 `$()` 法（推荐）

```bash
var=$(pwd)
echo $var    # 输出当前工作目录，如 /home/user
```

| 方式 | 语法 | 推荐度 | 说明 |
|------|------|--------|------|
| 反引号 | `` `command` `` | 旧式写法，支持嵌套但可读性差 |
| `$()` | `$(command)` | 现代推荐写法，支持嵌套，可读性好 |

```bash
# 实用示例：获取当前日期
today=$(date +%Y-%m-%d)
echo "今天是：$today"

# 嵌套命令示例
result=$(ls $(pwd))   # $() 方式嵌套清晰
```

---

## 4. 删除变量

```bash
unset 变量名
```

示例：
```bash
name="hello"
echo $name       # hello
unset name
echo $name       # 空（变量已被删除）
```

---

## 5. 特殊变量

Shell提供了一系列预定义的特殊变量，用于获取脚本运行时的上下文信息。

| 变量 | 含义 | 示例说明 |
|------|------|----------|
| `$0` | 当前脚本的文件名 | `./test.sh` → `$0` 为 `./test.sh` |
| `$1` ~ `$9` | 脚本的第1~9个参数 | `./test.sh a b c` → `$1` = a, `$2` = b, `$3` = c |
| `${10}` ~ `${n}` | 脚本的第10个及以后的参数 | 超过 9 的参数必须用 `${}` 包裹，如 `${10}` |
| `$#` | 传递给脚本的参数个数 | `./test.sh a b c` → `$#` = 3 |
| `$*` | 所有参数（作为一个字符串） | `./test.sh a b c` → `$*` = "a b c" |
| `$@` | 所有参数（作为列表） | 与 `$*` 在 for 循环中有重要区别（后续讲解） |
| `$?` | 上一条命令的退出状态码 / 函数返回值 | 0 表示成功，非 0 表示失败 |
| `$$` | 当前 Shell 进程的 PID | 如 `12345` |
| `$!` | 最后一个后台进程的 PID | 结合 `&` 使用 |
| `$-` | 当前 Shell 的选项标志 | 如 `himBHs` |

### 5.1 演示脚本

```bash
#!/bin/bash
# 保存为 special_var.sh
echo "脚本名：$0"
echo "第1个参数：$1"
echo "第2个参数：$2"
echo "参数个数：$#"
echo "所有参数（字符串）：$*"
echo "当前进程PID：$$"
```

运行：
```bash
./special_var.sh hello world
# 脚本名：./special_var.sh
# 第1个参数：hello
# 第2个参数：world
# 参数个数：2
# 所有参数（字符串）：hello world
# 当前进程PID：5678
```

### 5.2 shift 命令 — 移动位置参数

`shift` 用于将位置参数整体向左移动一位：`$2` 变成 `$1`，`$3` 变成 `$2`，依此类推，原来的 `$1` 被丢弃。`shift N` 表示一次移动 N 位。

```bash
#!/bin/bash
# 演示 shift
echo "处理前——参数个数：$#"
echo "第一个参数：$1"

shift               # 整体左移一位
echo "处理后——参数个数：$#"
echo "第一个参数（原$2）：$1"
```

运行：
```bash
./test.sh a b c d
# 处理前——参数个数：4
# 第一个参数：a
# 处理后——参数个数：3
# 第一个参数（原$2）：b
```

**实用场景**：逐条处理传入的参数，每次 shift 后剩下的都是待处理的参数。

```bash
#!/bin/bash
# 处理命令行选项（case 语句在下一讲详讲）
while [ $# -gt 0 ]; do
    case "$1" in
        -f) file="$2"; shift 2 ;;   # 取 -f 后面的文件名，跳过两个参数
        -v) verbose=true; shift 1 ;; # 标记 verbose，只跳一个
        *) echo "未知选项：$1"; shift 1 ;;
    esac
done
```

---

## 6. 字符串拼接

Shell中字符串拼接非常简单——**直接并排放即可**，无需加号或其他连接符。

```bash
name="野火"
greeting="hello ${name} 欢迎学习Linux"
echo $greeting     # hello 野火 欢迎学习Linux

# 也可以直接写在一起
var="hello"
result="${var} world"
echo $result       # hello world
```

> **要点**：Shell 中字符串拼接不需要任何连接符，直接将变量和字符串写在一起就行。

---

## 7. read命令 — 读取键盘输入

`read` 命令用于从标准输入（键盘）读取用户输入并存入变量。

### 7.1 基本用法

```bash
read var              # 等待用户输入，按回车后存入变量 var
echo "你输入的是：$var"
```

### 7.2 带提示信息的 read（-p 参数）

```bash
read -p "请输入你的名字：" name
echo "你好，${name}！"
```

运行效果：
```
请输入你的名字：张三
你好，张三！
```

### 7.3 常用选项

| 选项 | 说明 | 示例 |
|------|------|------|
| `-p` | 显示提示信息 | `read -p "请输入：" var` |
| `-t` | 设置超时时间（秒） | `read -t 5 var`（5秒后超时，返回非 0 退出码） |
| `-s` | 静默输入（不显示输入内容） | `read -s -p "密码：" pass` |
| `-n` | 限制输入字符数 | `read -n 1 var`（只读1个字符，无需按回车） |
| `-a` | 将输入按空格拆分后存入**数组** | `read -a arr` → 输入 `a b c` → `echo ${arr[0]}` 输出 `a` |
| `-r` | 不处理反斜杠转义 | `read -r line`（推荐用于读取文件，避免 `\` 被吃掉） |
| `-d` | 指定分隔符（默认为换行） | `read -d ':' var`（读到 `:` 为止） |

### 7.4 一次读取多个变量

```bash
# 用户输入会被按空格拆分，依次赋给多个变量
read -p "请输入姓名和年龄：" name age
echo "姓名：$name, 年龄：$age"
```

运行：
```
请输入姓名和年龄：张三 25
姓名：张三, 年龄：25
```

### 7.5 读取密码（静默模式）

```bash
read -s -p "请输入密码：" password
echo          # 输出一个换行（-s 模式不会自动换行）
echo "密码已输入（${#password} 位）"
```

### 7.6 while read line — 逐行读取文件

这是 `read` 最常用的场景之一：将整个文件内容逐行读入处理。

```bash
#!/bin/bash
# 方式1：管道方式（最常用，但变量修改在子 Shell 中会丢失）
cat /etc/passwd | while read -r line; do
    echo "行内容：$line"
done

# 方式2：重定向方式（推荐，避免子 Shell 问题）
while read -r line; do
    echo "行内容：$line"
done < /etc/passwd

# 方式3：按特定分隔符读取（如读取 CSV 文件）
while IFS=',' read -r col1 col2 col3; do
    echo "列1: $col1, 列2: $col2, 列3: $col3"
done < data.csv
```

> **注意**：推荐使用方式2（重定向方式），因为管道方式会在子 Shell 中执行 `while` 循环，循环内的变量修改在循环结束后会丢失。使用 `-r` 参数可防止反斜杠被解释为转义字符。

### 7.7 printf — 格式化输出

`printf` 提供了类似 C 语言 `printf()` 的格式化输出能力，比 `echo` 更精确，适合需要对齐、控制格式的复杂输出场景。

```bash
# 基本语法：printf "格式字符串" 参数1 参数2 ...
printf "姓名：%-10s 年龄：%3d\n" "张三" 25
# 输出：姓名：张三         年龄： 25

# 与 echo 对比
echo "姓名：$name 年龄：$age"             # 简单，但格式不可控
printf "姓名：%-10s 年龄：%03d\n" "$name" "$age"   # 可精确控制列宽、补零、对齐
```

**常用格式占位符**：

| 占位符 | 含义 | 示例 |
|--------|------|------|
| `%s` | 字符串 | `printf "%s" "hello"` |
| `%d` | 十进制整数 | `printf "%d" 42` |
| `%f` | 浮点数 | `printf "%.2f" 3.14159` → `3.14` |
| `%x` | 十六进制 | `printf "%x" 255` → `ff` |
| `%o` | 八进制 | `printf "%o" 8` → `10` |
| `\n` | 换行 | `printf "line1\nline2\n"` |
| `\t` | 制表符 | `printf "col1\tcol2\n"` |

**宽度与对齐**：

```bash
# %10s：宽度为 10，右对齐
# %-10s：宽度为 10，左对齐（负号表示左对齐）
# %03d：宽度为 3，不足补零

printf "|%-10s|%10s|\n" "左对齐" "右对齐"
# 输出：|左对齐       |      右对齐|

printf "编号：%03d\n" 7
# 输出：编号：007
```

**实用场景**：格式化输出表格

```bash
#!/bin/bash
# 打印格式化的表格标题
printf "%-4s %-10s %-8s %-8s\n" "ID" "姓名" "科目" "分数"
printf "%-4s %-10s %-8s %-8s\n" "---" "----" "----" "----"
printf "%-4d %-10s %-8s %-8.1f\n" 1 "张三" "数学" 92.5
printf "%-4d %-10s %-8s %-8.1f\n" 2 "李四" "语文" 88.0
```

输出：
```
ID   姓名         科目     分数
---  ----         ----     ----
1    张三         数学     92.5
2    李四         语文     88.0
```

---

## 8. 数学运算 — `$(( ))`

Shell 默认将所有变量视为**字符串**。要进行数学运算，必须使用 `$(( ))` 语法。

### 8.1 基本语法

```bash
a=5
b=3

sum=$((a + b))       # 注意：$(( )) 内部的变量可以省略 $ 符号
echo $sum             # 8
```

### 8.2 支持的运算符

| 运算符 | 含义 | 示例 | 结果 |
|--------|------|------|------|
| `+` | 加法 | `$((5 + 3))` | 8 |
| `-` | 减法 | `$((5 - 3))` | 2 |
| `*` | 乘法 | `$((5 * 3))` | 15 |
| `/` | 除法（整除） | `$((5 / 2))` | 2 |
| `%` | 取余 | `$((5 % 2))` | 1 |
| `**` | 幂运算 | `$((2 ** 3))` | 8 |
| `++` | 自增 | `$((a++))` | — |
| `--` | 自减 | `$((a--))` | — |

### 8.3 注意事项

```bash
# 正确写法
echo $((a + b))

# 也可以
echo $(($a + $b))

# 错误：不用 $(( )) 的话，结果是字符串拼接
echo a+b      # 输出：a+b（字面量，不是运算）

# 错误：普通括号无效
echo (a+b)    # 语法错误
```

> **核心要点**：
> - 运算符前后推荐加空格分隔
> - 必须使用 `$(( ))` 语法才能获取数学运算结果
> - `$(( ))` 内部的变量 `$` 可以省略

### 8.4 (( )) — 用于条件判断（无需 $ 前缀）

在条件判断环境中（如 `if`、`while`），使用 `(( ))` 且不加 `$` 前缀。它的退出状态码由表达式结果决定：非 0 为真（成功），0 为假（失败）。

```bash
a=5; b=3

# (( )) 作为条件测试
(( a > b )) && echo "a 大于 b"
(( a == b )) || echo "a 不等于 b"

# 在 if 中使用（下一讲详讲）
if (( a > b )); then
    echo "a is greater"
fi

# 支持 C 风格的复合赋值和自增自减
(( count++ ))
(( sum += a + b ))
```

### 8.5 let 命令 — 算术赋值

`let` 命令用于执行算术表达式并将其结果赋给变量。功能与 `(( ))` 类似，但 `let` 不需要 `$` 前缀来获取变量的值。

```bash
a=5; b=3

# let 基本用法
let result=a+b
echo $result           # 8

# 多条表达式（逗号分隔）
let x=10 y=20 z=x+y
echo $z                # 30

# 自增自减
let count=0
let count++
echo $count            # 1

# 复合赋值（注意 let 中 * 需要转义或加引号，否则会被当作通配符）
a=5
let "a *= 3"
echo $a                # 15

# 对比：let vs (( ))
let result=a+b         # let 后直接写表达式
result=$((a + b))      # $(( )) 需要 $ 来获取值，赋值给变量
(( result = a + b ))   # (( )) 可以直接赋值（内部不需要 $）
```

**三种方式的对照**：

| 方式 | 语法 | 适用场景 |
|------|------|----------|
| `let` | `let result=a+b` | 简单赋值，可多条表达式 |
| `$(( ))` | `result=$((a + b))` | 嵌入字符串或复杂表达式 |
| `(( ))` | `(( result = a + b ))` | 条件判断，复合赋值 |

### 8.6 expr 命令 — 传统算术运算

`expr` 是一个外部命令，也能完成算术运算，但写法较为繁琐，已逐渐被 `$(( ))` 取代。

```bash
# expr 基本用法（注意：运算符和操作数之间必须有空格）
result=$(expr 5 + 3)
echo $result           # 8

# 乘法需要用反斜杠转义 *（防止被 Shell 当作通配符）
result=$(expr 5 \* 3)
echo $result           # 15

# expr 的缺点：必须有空格、乘法必须转义、只支持整数
result=$(expr 5+3)     # 输出字面量 "5+3"
result=$(expr 5 * 3)   # Shell 会把 * 展开为当前目录的文件名
```

**expr vs $(( )) 对比**：

| 特性 | `expr` | `$(( ))` |
|------|--------|----------|
| 空格要求 | 运算符两边**必须**有空格 | 可选，推荐加空格 |
| 乘法转义 | 必须用 `\*` | 不需要转义 |
| 执行方式 | 外部命令，启动子进程 | Shell 内建，性能好 |
| 浮点数 | 不支持 | 不支持 |
| 推荐度 | （仅兼容老脚本时使用） | （推荐） |

> **建议**：新脚本一律使用 `$(( ))`。只有维护老旧脚本或遇到不兼容的场景时才用 `expr`。

### 8.7 bc 命令 — 浮点数运算

`bc`（Basic Calculator）是一个支持任意精度浮点数运算的命令行计算器。Shell 的 `$(( ))` 只支持整数，遇到浮点数需求时就需要 `bc`。

```bash
# 基本用法：管道传表达式给 bc
echo "5.5 + 3.2" | bc
# 输出：8.7

echo "10 / 3" | bc
# 输出：3（默认只保留整数）

# scale 设置小数位数
echo "scale=2; 10 / 3" | bc
# 输出：3.33

echo "scale=4; 22 / 7" | bc
# 输出：3.1428

# 多条表达式（分号分隔）
echo "scale=2; a=3.14; b=2; a * b" | bc
# 输出：6.28

# 变量赋值
result=$(echo "scale=2; 100 / 3" | bc)
echo "结果：$result"    # 结果：33.33

# 使用 here-document 传复杂表达式
result=$(bc << EOF
scale=4
x = 3.14159
y = 2.71828
x + y
EOF
)
echo $result            # 5.8598
```

**bc 支持的常见运算**：

```bash
# 基础四则运算
echo "scale=2; 10.5 + 3.2" | bc    # 13.7
echo "scale=2; 10.5 - 3.2" | bc    # 7.3
echo "scale=2; 10.5 * 3.2" | bc    # 33.60
echo "scale=2; 10.5 / 3.2" | bc    # 3.28

# 幂运算
echo "2 ^ 10" | bc                  # 1024

# 平方根
echo "scale=4; sqrt(2)" | bc        # 1.4142

# 数学库（-l 参数启用标准数学库）
echo "scale=4; s(3.14159/2)" | bc -l    # 正弦 sin(pi/2)
echo "scale=4; l(2.71828)" | bc -l      # 自然对数 ln(e)
```

> **要点**：`bc` 是 Shell 进行浮点数运算的唯一可靠方式。需要小数计算时，直接用 `echo "表达式" | bc` 即可。

---

## 9. 逻辑运算 — `&&` 和 `||`

逻辑运算符用于根据前一个命令的执行结果，决定是否执行后一个命令。

### 9.1 逻辑与 `&&`

> **规则**：前一个命令**成功**（退出码为 0）才执行后一个命令。

```bash
# 创建目录成功后才进入该目录
mkdir test && cd test

# 如果目录已存在（mkdir 失败），则不执行 cd
```

### 9.2 逻辑或 `||`

> **规则**：前一个命令**失败**（退出码非 0）才执行后一个命令。

```bash
# 尝试进入目录，失败则创建该目录
cd test || mkdir test

# 如果目录不存在（cd 失败），则创建它
```

### 9.3 组合使用

```bash
# 实用示例：确保目录存在并进入
cd test || mkdir test && cd test

# 等价于：如果 cd test 失败 → 创建 test 目录 → 再进入
```

### 9.4 退出状态码 `$?`

每个命令执行后都会返回一个退出状态码，存放在 `$?` 中：

```bash
ls /tmp
echo $?      # 0 = 成功

ls /nonexistent
echo $?      # 2 = 失败（目录不存在）
```

| 值 | 含义 |
|----|------|
| `0` | 命令执行成功 |
| 非 `0` | 命令执行失败（不同数字代表不同错误原因） |

---

## 10. 数组

Shell 支持两种数组：**索引数组**（数字索引）和**关联数组**（字符串键，类似其他语言的字典/Map）。

### 10.1 定义索引数组

```bash
# 方式1：括号直接定义
fruits=(apple banana orange grape)

# 方式2：逐个赋值
fruits[0]=apple
fruits[1]=banana
fruits[2]=orange
fruits[3]=grape

# 方式3：declare 声明
declare -a names=(zhangsan lisi wangwu)

# 混合赋值（索引可以不连续）
arr=([0]=10 [5]=50 [8]=80)
```

### 10.2 访问数组元素

```bash
fruits=(apple banana orange grape)

echo ${fruits[0]}        # apple（访问第 1 个元素，索引从 0 开始）
echo ${fruits[1]}        # banana

echo ${fruits[@]}        # apple banana orange grape（所有元素，作为独立元素展开）
echo ${fruits[*]}        # apple banana orange grape（所有元素，作为一个字符串）

echo ${#fruits[@]}       # 4（数组长度——元素个数）

echo ${!fruits[@]}       # 0 1 2 3（获取所有键/索引）
```

### 10.3 遍历数组

```bash
#!/bin/bash
fruits=(apple banana orange grape)

# 方式1：遍历值
for item in "${fruits[@]}"; do
    echo "水果：$item"
done

# 方式2：遍历索引
for i in "${!fruits[@]}"; do
    echo "索引 $i = ${fruits[$i]}"
done

# 方式3：C 风格 for 循环（下一讲详讲）
for ((i=0; i<${#fruits[@]}; i++)); do
    echo "第 $i 个：${fruits[$i]}"
done
```

### 10.4 数组操作

```bash
fruits=(apple banana orange grape)

# 追加元素
fruits+=(mango)                     # fruits 变为 5 个元素
fruits+=("pineapple" "peach")       # 一次追加多个

# 修改元素
fruits[1]=berry                     # banana 变成 berry

# 切片（取出部分元素）
echo ${fruits[@]:1:2}               # banana orange（从索引 1 开始取 2 个）

# 删除元素
unset fruits[2]                     # 删除 orange（索引 2 变成空位，数组长度不变）
unset fruits                        # 删除整个数组
```

### 10.5 关联数组（键值对）

关联数组使用字符串作为键，需要用 `declare -A` 声明。

```bash
# 声明关联数组（必须用 declare -A）
declare -A user

# 赋值
user[name]=zhangsan
user[age]=25
user[city]=beijing

# 访问
echo ${user[name]}          # zhangsan
echo ${user[age]}           # 25

# 遍历键
for key in "${!user[@]}"; do
    echo "$key = ${user[$key]}"
done

# 输出：
# name = zhangsan
# age = 25
# city = beijing

# 批量定义
declare -A colors=([red]="#FF0000" [green]="#00FF00" [blue]="#0000FF")
echo ${colors[red]}         # #FF0000
```

### 10.6 $* 和 $@ 在数组中的区别

`$*` 和 `$@` 的区别同样适用于数组的 `[*]` 和 `[@]` 展开：

```bash
fruits=(apple "big banana" orange)

# [*] 将所有元素当做一个字符串
for item in "${fruits[*]}"; do
    echo "-> $item"
done
# 输出（一次循环）：-> apple big banana orange

# [@] 将每个元素独立展开（加引号保护空格）
for item in "${fruits[@]}"; do
    echo "-> $item"
done
# 输出（三次循环）：
# -> apple
# -> big banana
# -> orange
```

> **建议**：遍历数组时**始终使用** `"${arr[@]}"`（带双引号），它能正确处理元素中含空格的情况。

---

## 11. eval 命令 — 二次解析

`eval` 命令将其参数作为 Shell 命令执行，相当于对参数进行**两次扫描解析**：第一次完成常规的变量替换，第二次将替换后的结果作为命令再次解析执行。

### 11.1 基本用法

```bash
cmd="echo hello world"
eval $cmd          # 输出：hello world
# 等价于直接执行：echo hello world
```

### 11.2 为什么需要 eval？

当变量中包含 Shell 特殊语法（如重定向、管道、变量名嵌套）时，直接执行不会解析这些语法，`eval` 能做两次解析。

```bash
# 场景1：变量中包含重定向符
cmd="echo hello > /tmp/out.txt"
$cmd                # 输出：hello > /tmp/out.txt（> 被当作字面输出）
eval $cmd           # 真正执行了重定向，hello 被写入 /tmp/out.txt

# 场景2：间接变量引用（用变量的值作为另一个变量名）
prefix="APP"
APP_NAME="my_application"

# 想要获取 APP_NAME 的值，但只知道 prefix 变量
var_name="${prefix}_NAME"      # var_name = "APP_NAME"
echo ${!var_name}               # my_application（间接引用，Bash 4.0+）
# 或者用 eval：
eval "echo \$${var_name}"       # my_application
```

### 11.3 eval 的风险

**eval 会将字符串当作代码执行，存在安全风险**。如果参数中包含未经过滤的用户输入，可能导致命令注入。

```bash
# 危险示例：用户输入包含恶意命令
read -p "输入文件名：" filename
# 用户输入：file.txt; rm -rf /
eval "cat $filename"
# 被执行为：cat file.txt; rm -rf /（两条命令都会执行！）
```

> **原则**：能用 `${!var}`（间接引用）、数组、函数等替代方案解决的，不要用 `eval`。如果必须使用，确保参数经过严格校验。

---

## 12. exec 命令 — 替换当前进程

`exec` 用于**用指定命令替换当前 Shell 进程**。执行后，当前进程的 PID 不变，但 Shell 本身被替换为指定的命令，原 Shell 不复存在。

### 12.1 基本用法

```bash
#!/bin/bash
echo "脚本 PID：$$"
echo "即将被 exec 替换..."

exec /bin/ls -la /tmp
# 从这里开始，当前 Shell 已经被 ls 替换
# 下面的代码永远不会被执行
echo "这行不会输出"
```

运行：
```bash
./test.sh
# 脚本 PID：12345
# 即将被 exec 替换...
# （显示 /tmp 目录内容）
# 注意：echo "这行不会输出" 确实不会执行
```

### 12.2 exec 重定向文件描述符

`exec` 还有一个重要用途：在当前 Shell 中**重定向文件描述符**（不替换进程）。

```bash
# 将标准输出重定向到文件（之后所有 echo 的输出都写入文件）
exec > /tmp/output.txt
echo "这条会写入文件"
echo "这条也会写入文件"
# 恢复标准输出：exec > /dev/tty

# 打开文件描述符 3 并绑定到文件
exec 3> /tmp/log.txt
echo "写入 fd 3" >&3
exec 3>&-          # 关闭 fd 3

# 从文件描述符读取
exec 3< /etc/hostname
read -r hostname <&3
exec 3<&-          # 关闭 fd 3
echo "$hostname"
```

**常用模式**：

```bash
# 保存原始 stdout，临时重定向到文件
exec 3>&1                     # 将 fd 3 指向原始 stdout
exec > /tmp/script.log        # 所有输出转到日志
echo "这条写入日志"
exec 1>&3                     # 恢复原始 stdout
exec 3>&-                     # 关闭 fd 3
echo "这条回到终端显示"
```

---

## 13. 条件检测（预告）

条件检测（`if-else`、`test` 命令等）将在**第15讲 — Shell脚本编程（下）**中详细讲解。

---

## 14. 本讲要点总结

### 变量
1. **变量定义**三种方式：直接赋值、单引号（不解析变量）、双引号（解析变量）
2. **命名规则**：字母/数字/下划线，不能数字开头，区分大小写
3. `readonly` / `declare -r` 定义只读变量
4. `declare -i` 定义整型变量，`-a` 索引数组，`-A` 关联数组

### 变量使用
5. 引用变量推荐用 `${变量名}` 避免边界歧义
6. **变量扩展**：`${var:-默认值}` 提供默认值；`${var:=默认值}` 赋值默认值；`${var:?错误信息}` 必填校验
7. **字符串操作**：`${#var}` 取长度，`${var#pat}` 删前缀，`${var%pat}` 删后缀，`${var/old/new}` 替换

### 输入输出
8. 命令结果赋值推荐用 `$(command)` 而非反引号
9. `read -p` 读取键盘输入并显示提示；`read -s` 静默读密码
10. `while read -r line` 逐行读取文件内容
11. `printf` 提供 C 语言风格格式化输出，比 `echo` 更精确

### 运算
12. **数学运算**：`$(( ))` 进行整数运算（推荐）；`let` 也可用但少见；`expr` 老旧不推荐
13. **浮点数运算**：使用 `bc` 命令（`echo "scale=2; 10/3" | bc`）
14. `(( ))` 可用于条件判断环境，支持自增自减等 C 风格运算

### 逻辑与进程
15. `&&` 前成功才执行后，`||` 前失败才执行后
16. **特殊变量**：`$0`(脚本名)、`$1`~`$9`(参数)、`$#`(参数个数)、`$?`(退出码)、`$$`(PID)
17. `shift` 移动位置参数，常用于逐条处理命令行参数
18. `eval` 二次解析字符串为命令，功能强大但需注意安全风险
19. `exec` 替换当前进程，或仅重定向文件描述符

### 数组
20. **数组定义**：`arr=(val1 val2 val3)`，访问用 `${arr[0]}` / `${arr[@]}`
21. **关联数组**：`declare -A` 声明，用字符串作为键
22. `unset` 删除变量或数组元素

### 预告
23. 条件检测（if/else等）将在下一讲展开

---

*笔记生成日期：2026-07-22 | 转写：Whisper tiny | 优化日期：2026-07-23*
