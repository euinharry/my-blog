---
title: "第15讲：Shell脚本（下）"
date: 2026-09-11T09:16:00+08:00
draft: false
description: "Shell脚本中判断条件是否成立有三种方式："
series: ["Linux 入门"]
series_order: 11
categories: ["技术笔记"]
tags: ["Linux", "Shell", "脚本"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P16)
> **标题**：第15讲 — Shell脚本（下）
> **时长**：31分29秒
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. 条件检测 — test、[ ] 与 [[ ]]

Shell脚本中判断条件是否成立有三种方式：

### 方式一：test 命令

```bash
test 条件表达式
```

### 方式二：[ ] （POSIX 标准，兼容性最好）

```bash
[ 条件表达式 ]
```

> **注意**：`[` 和 `]` 前后必须加空格，否则语法错误。

### 方式三：[[ ]] （bash 扩展，功能更强，推荐在 bash 脚本中使用）

```bash
[[ 条件表达式 ]]
```

> **注意**：`[[ ]]` 是 bash（及 zsh/ksh）的扩展语法，不是 POSIX 标准。如果脚本需要跨 sh/dash 运行，请使用 `[ ]`。如果是纯 bash 环境，推荐 `[[ ]]`。

### 三者对比表

| 特性 | `test` | `[ ]` | `[[ ]]` |
|------|--------|-------|---------|
| 空变量必须加引号 | 是 | 是 | 否（更安全） |
| 支持 `&&` `||` 在内部 | 否（用 `-a` `-o`） | 否（用 `-a` `-o`） | 是 |
| 支持正则 `=~` | 否 | 否 | 是 |
| 支持模式匹配 `==` | 否 | 否 | 是（如 `[[ $a == abc* ]]`） |
| 词拆分/路径展开 | 会执行 | 会执行 | 不会（更安全） |
| POSIX 兼容 | 是 | 是 | 否 |

### 1.1 数值比较选项

| 选项 | 含义 | 示例 |
|------|------|------|
| `-eq` | 等于 (equal) | `[ $a -eq $b ]` |
| `-ne` | 不等于 (not equal) | `[ $a -ne $b ]` |
| `-gt` | 大于 (greater than) | `[ $a -gt $b ]` |
| `-lt` | 小于 (less than) | `[ $a -lt $b ]` |
| `-ge` | 大于等于 (greater or equal) | `[ $a -ge $b ]` |
| `-le` | 小于等于 (less or equal) | `[ $a -le $b ]` |

### 1.2 增强版数值条件：(( ))

`(( ))` 用于数值运算和条件判断，写法更接近 C 语言，比 `[ -gt ]` 之类直观得多：

```bash
a=10
b=20

# 传统写法
[ $a -gt $b ] && echo "a > b"

# (( )) 写法（更简洁）
(( a > b )) && echo "a > b"
(( a < b )) && echo "a < b"
(( a == b )) && echo "a == b"
(( a != b )) && echo "a != b"
(( a >= b )) && echo "a >= b"
(( a <= b )) && echo "a <= b"

# (( )) 也支持复合条件
(( a > 0 && b < 100 )) && echo "条件成立"

# 注意：(( )) 返回 0 表示 true（条件成立），返回 1 表示 false
(( 1 > 0 ))
echo $?    # 输出 0（true）
(( 1 < 0 ))
echo $?    # 输出 1（false）
```

> **对比 while 中的用法**：`while (( i <= 10 ))` 比 `while [ $i -le 10 ]` 更简洁。

### 1.3 字符串判断选项

| 选项 | 含义 | 示例 |
|------|------|------|
| `-z` | 字符串是否为空 | `[ -z "$str" ]` |
| `-n` | 字符串是否非空 | `[ -n "$str" ]` |
| `=` | 字符串是否相等 | `[ "$a" = "$b" ]` |
| `!=` | 字符串是否不等 | `[ "$a" != "$b" ]` |

[[ ]] 中字符串判断更安全，不需要担心变量为空时的错误：

```bash
# [ ] 中变量为空会报错
[ $unset_var = "hello" ]    # 报错：一元运算符预期

# [[ ]] 中安全
[[ $unset_var = "hello" ]]  # 正常执行，结果为 false
```

### 1.4 文件判断选项

| 选项 | 含义 | 示例 |
|------|------|------|
| `-d` | 目录是否存在 | `[ -d "/etc" ]` |
| `-f` | 普通文件是否存在 | `[ -f "/etc/passwd" ]` |
| `-e` | 路径是否存在（文件和目录都算） | `[ -e "/tmp/somefile" ]` |
| `-s` | 文件存在且非空（大小 > 0） | `[ -s "/var/log/syslog" ]` |
| `-r` | 文件是否可读 | `[ -r "/etc/shadow" ]` |
| `-w` | 文件是否可写 | `[ -w "/tmp/test.txt" ]` |
| `-x` | 文件是否可执行 | `[ -x "/usr/bin/python3" ]` |
| `-L` | 是否为符号链接 | `[ -L "/usr/bin/java" ]` |
| `-h` | 同 `-L`，是否为符号链接 | `[ -h "/usr/bin/java" ]` |
| `-b` | 是否为块设备文件 | `[ -b "/dev/sda" ]` |
| `-c` | 是否为字符设备文件 | `[ -c "/dev/tty" ]` |
| `-p` | 是否为管道文件 | `[ -p "/tmp/mypipe" ]` |

**常见组合示例：**

```bash
# 检查文件是否存在且可读
if [ -f "$file" ] && [ -r "$file" ]; then
    cat "$file"
fi

# 检查目录是否存在，不存在则创建
[ -d "$dir" ] || mkdir -p "$dir"
```

### 1.5 逻辑组合

在 `[ ]` 中使用 `-a`（and）、`-o`（or）、`!`（not）组合条件：

| 符号 | 含义 | 示例 |
|------|------|------|
| `-a` | 逻辑与（AND） | `[ $a -gt 0 -a $b -lt 100 ]` |
| `-o` | 逻辑或（OR） | `[ $a -eq 0 -o $a -eq 1 ]` |
| `!` | 逻辑非（NOT） | `[ ! -f "file" ]`（文件不存在时为真）|

```bash
# 示例：使用 -a 和 -o
[ $a -gt 0 -a $a -lt 10 ]    # a > 0 且 a < 10
[ $b -eq 0 -o $b -eq 1 ]     # b == 0 或 b == 1
[ ! -d "/tmp/xxx" ]            # /tmp/xxx 不是目录时为真
```

> **注意**：`-a` 和 `-o` 是 POSIX 标准，但在 `[[ ]]` 中可以直接用 `&&` 和 `||`，更推荐。

在 `[[ ]]` 中直接使用 `&&` 和 `||`：

```bash
[[ $a -gt 0 && $a -lt 10 ]]   # 更清晰
[[ $b -eq 0 || $b -eq 1 ]]
[[ ! -d "/tmp/xxx" ]]
```

### 1.6 [[ ]] 的增强功能：正则匹配 =~

```bash
# 检查变量是否匹配正则表达式
[[ "hello123" =~ ^hello[0-9]+$ ]] && echo "匹配"
[[ "abc@def.com" =~ ^[a-z]+@[a-z]+\.[a-z]+$ ]] && echo "邮箱格式正确"
```

---

## 2. 管道 |

管道可以将多条命令连接，前一条命令的**标准输出**传递给后一条命令作为输入。

```bash
命令1 | 命令2
```

### 示例

```bash
ls | grep "123.txt"      # 列出当前目录，过滤出包含 123.txt 的行
```

### 注意事项

- 管道传递的是**标准输出（stdout）**，不是标准错误（stderr）
- 如果命令1执行失败（输出错误信息），命令2接收到的将是空内容

```bash
ls --bad-option | grep "test"    # grep 不会执行，ls 已报错中断
```

### 2.1 多个管道串联

```bash
# 三个命令串联：ls -> grep -> wc
ls -l /etc | grep "^d" | wc -l
# 解读：列出/etc下所有条目 -> 过滤出目录行（以d开头） -> 统计行数（目录数量）

# 实战：找出当前目录下占用空间最大的5个文件
du -sh * | sort -hr | head -5

# 实战：查看某个进程的 PID
ps aux | grep "nginx" | grep -v grep | awk '{print $2}'
```

### 2.2 合并标准错误与标准输出：2>&1

默认情况下，管道只传递 stdout。如果想同时传递 stderr，需要先合并：

```bash
# 将 stderr 重定向到 stdout（合并后一起走管道）
some_command 2>&1 | grep "error"

# 示例：编译时同时捕获错误和警告
gcc program.c 2>&1 | tee build.log

# 简短写法（bash 4+）
some_command |& grep "error"       # |& 等价于 2>&1 |
```

---

## 3. 重定向补充

### 3.1 Here Document（<< EOF）

将多行文本作为标准输入传递给命令：

```bash
# 基本语法
命令 << 结束标记
第一行内容
第二行内容
...
结束标记

# 示例：给文件写入多行内容
cat << EOF > config.txt
server {
    listen 80;
    server_name example.com;
    root /var/www/html;
}
EOF

# 如果开始标记加引号（'EOF'），则禁用变量替换
cat << 'EOF' > script.sh
echo "这里的 $HOME 不会被展开"
current_user=$(whoami)    # 也不会被执行
EOF
```

### 3.2 Here String（<<<）

将单个字符串直接作为命令的标准输入：

```bash
# 语法：命令 <<< "字符串"

# 示例
grep "root" <<< "hello root user"    # 直接搜索字符串
wc -w <<< "one two three four"       # 输出 4

# 等效于
echo "hello root user" | grep "root"

# 常用于变量内容处理
text="hello world"
grep "world" <<< "$text"
```

---

## 4. if 条件判断

### 4.1 if -- fi

```bash
if [ 条件 ]; then
    # 条件成立时执行
fi
```

### 4.2 if -- else -- fi

```bash
if [ 条件 ]; then
    # 条件成立
else
    # 条件不成立
fi
```

### 4.3 if -- elif -- else -- fi

```bash
if [ 条件1 ]; then
    # 条件1成立
elif [ 条件2 ]; then
    # 条件2成立
else
    # 以上都不成立
fi
```

### 4.4 示例

```bash
read -p "输入a: " a
read -p "输入b: " b

if [ $a -eq $b ]; then
    echo "a等于b"
elif [ $a -gt $b ]; then
    echo "a大于b"
else
    echo "a小于b"
fi
```

### 4.5 简写形式（条件执行的两种写法）

利用 `&&` 和 `||` 的短路特性，简化单行条件判断：

```bash
# 条件成立时执行（AND）
[ -f "/etc/passwd" ] && echo "文件存在"

# 条件不成立时执行（OR）
[ -d "/tmp/work" ] || mkdir -p "/tmp/work"

# 组合使用：模拟 if-else
[ -f "$file" ] && echo "存在" || echo "不存在"

# 更复杂的简写
grep -q "root" /etc/passwd && echo "root用户存在" || echo "root用户不存在"
```

### 4.6 if 中多个条件

```bash
# 方式一：用 && / || 连接多个 [ ] 测试
if [ $a -gt 0 ] && [ $a -lt 100 ]; then
    echo "a 在 0 到 100 之间"
fi

if [ -f "$file" ] && [ -r "$file" ]; then
    echo "文件存在且可读"
fi

# 方式二：用 [[ ]] 内部使用 && / ||
if [[ $a -gt 0 && $a -lt 100 ]]; then
    echo "a 在 0 到 100 之间"
fi

# 方式三：用 (( )) 做数值多条件
if (( a > 0 && a < 100 )); then
    echo "a 在 0 到 100 之间"
fi
```

### 4.7 if 判断命令执行结果

if 后面可以直接放命令，判断命令是否执行成功（返回值为 0）：

```bash
# 判断文件是否包含某文本（-q 静默模式，不输出）
if grep -q "nginx" /etc/passwd; then
    echo "nginx 用户存在"
else
    echo "nginx 用户不存在"
fi

# 判断命令是否成功
if ping -c1 -W1 8.8.8.8 &>/dev/null; then
    echo "网络连通"
else
    echo "网络不通"
fi

# 判断目录是否创建成功
if mkdir /tmp/testdir 2>/dev/null; then
    echo "目录创建成功"
else
    echo "目录已存在或无权限"
fi
```

---

## 5. case 语句

类似C语言的 switch，用于多分支匹配。

```bash
case $变量 in
    值1)
        语句1
        ;;        # 分支结束
    值2)
        语句2
        ;;
    *)            # 通配符，默认分支
        默认语句
        ;;
esac              # case 结束
```

### 5.1 基本示例

```bash
read -p "输入a: " a

case $a in
    1)
        echo "a等于1"
        ;;
    2)
        echo "a等于2"
        ;;
    *)
        echo "a不等于1也不等于2"
        ;;
esac
```

### 5.2 使用通配符和正则模式

```bash
read -p "输入一个字符: " ch

case $ch in
    [0-9])          # 匹配单个数字
        echo "你输入了一个数字"
        ;;
    [a-z])          # 匹配单个小写字母
        echo "你输入了一个小写字母"
        ;;
    [A-Z])          # 匹配单个大写字母
        echo "你输入了一个大写字母"
        ;;
    [abc])          # 匹配 a、b 或 c
        echo "你输入了 a、b 或 c"
        ;;
    *)              # 其他所有情况
        echo "你输入了其他字符"
        ;;
esac
```

### 5.3 使用 | 组合多个模式

```bash
read -p "输入 yes 或 no: " answer

case $answer in
    yes|y|Y|YES|Yes)
        echo "你选择了 是"
        ;;
    no|n|N|NO|No)
        echo "你选择了 否"
        ;;
    *)
        echo "无效输入，请输入 yes 或 no"
        ;;
esac
```

### 5.4 实战：系统服务管理脚本

```bash
#!/bin/bash
# 简单的服务管理脚本

SERVICE_NAME="myapp"

case $1 in
    start)
        echo "正在启动 $SERVICE_NAME ..."
        # systemctl start $SERVICE_NAME
        echo "$SERVICE_NAME 已启动"
        ;;
    stop)
        echo "正在停止 $SERVICE_NAME ..."
        # systemctl stop $SERVICE_NAME
        echo "$SERVICE_NAME 已停止"
        ;;
    restart)
        echo "正在重启 $SERVICE_NAME ..."
        # systemctl restart $SERVICE_NAME
        echo "$SERVICE_NAME 已重启"
        ;;
    status)
        echo "查看 $SERVICE_NAME 状态 ..."
        # systemctl status $SERVICE_NAME
        ;;
    reload)
        echo "重新加载 $SERVICE_NAME 配置 ..."
        # systemctl reload $SERVICE_NAME
        ;;
    *)
        echo "用法: $0 {start|stop|restart|status|reload}"
        exit 1
        ;;
esac
```

### 5.5 实战：交互式选择菜单

```bash
#!/bin/bash

echo "========= 功能菜单 ========="
echo "1) 显示日期时间"
echo "2) 显示当前用户"
echo "3) 显示磁盘使用情况"
echo "4) 退出"
echo "============================"
read -p "请选择 [1-4]: " choice

case $choice in
    1)
        date
        ;;
    2)
        whoami
        ;;
    3)
        df -h
        ;;
    4)
        echo "再见！"
        exit 0
        ;;
    *)
        echo "无效选择，请输入 1-4 之间的数字"
        exit 1
        ;;
esac
```

---

## 6. for 循环

### 6.1 基本用法 -- 遍历列表

```bash
for var in 1 2 3 4 5 6 7 8 9 10; do
    echo $var
done
```

### 6.2 遍历范围

```bash
for var in {1..10}; do
    echo $var
done

# {start..end..step} 带步长（bash 4+）
for var in {2..20..2}; do
    echo $var    # 输出 2, 4, 6, ..., 20（偶数）
done
```

### 6.3 遍历命令执行结果

```bash
for var in $(ls /bin/*sh); do
    echo $var
done

# 注意：如果文件名包含空格，$(ls) 会出问题
# 更推荐用 glob 直接遍历：
for file in /bin/*sh; do
    echo "$file"
done
```

### 6.4 遍历脚本参数

```bash
# 写法一：$*（不加引号，逐个输出）
for var in $*; do
    echo $var
done

# 写法二：$@（不加引号，同上）
for var in $@; do
    echo $var
done
```

### 6.5 C 风格 for 循环

```bash
# 语法：for ((初始化; 条件; 步进)); do ... done

# 从 0 数到 9
for ((i = 0; i < 10; i++)); do
    echo "i = $i"
done

# 从 10 倒数到 1
for ((i = 10; i >= 1; i--)); do
    echo "倒计时: $i"
done

# 步长为 3
for ((i = 0; i <= 30; i += 3)); do
    echo "$i"
done

# C 风格 for 比 {1..10} 更灵活（支持变量、复杂表达式）
count=10
for ((i = 0; i < count * 2; i++)); do
    echo $i
done
```

### 6.6 break 和 continue

```bash
# break：跳出循环
for i in {1..10}; do
    if [ $i -eq 5 ]; then
        echo "遇到 5，退出循环"
        break
    fi
    echo "当前数字: $i"
done
# 输出：1, 2, 3, 4, 遇到 5，退出循环

# continue：跳过本轮，继续下一轮
for i in {1..10}; do
    if [ $((i % 2)) -eq 0 ]; then
        continue    # 跳过偶数
    fi
    echo "奇数: $i"
done

# break N：跳出 N 层循环
for i in {1..5}; do
    for j in {1..5}; do
        if [ $((i * j)) -gt 10 ]; then
            echo "i=$i, j=$j, 乘积 > 10，跳出所有循环"
            break 2    # 跳出两层循环
        fi
    done
done
```

---

## 7. $* 与 $@ 的区别

| 变量 | 不加引号 | 加引号 `"$var"` |
|------|---------|----------------|
| `$*` | 逐个列出所有参数 | **作为一个整体字符串** |
| `$@` | 逐个列出所有参数 | **每个参数独立存在** |

### 对比示例

```bash
# 脚本 test.sh 内容：
for var in "$*"; do
    echo $var
done

for var in "$@"; do
    echo $var
done
```

```bash
# 执行：./test.sh a b c d
# "$*" 输出：a b c d（一行，整体）
# "$@" 输出：a b c d（四行，每行一个）
```

---

## 8. while 循环

### 8.1 基本语法

```bash
i=1
while [ 条件 ]; do
    # 循环体
    i=$((i + 1))
done
```

### 8.2 示例 -- 使用数学运算代替 test 选项

```bash
i=1
while (( i <= 10 )); do
    echo $i
    i=$((i + 1))
done
```

> 使用 `(( ))` 进行数学运算判断，比记忆 test 选项更直观。

### 8.3 示例 -- 使用 [ ] 的传统写法

```bash
i=1
while [ $i -le 10 ]; do
    echo "$i"
    i=$((i + 1))
done
```

### 8.4 逐行读取文件：while read

```bash
# 逐行读取 /etc/passwd
while IFS= read -r line; do
    echo "行内容: $line"
done < /etc/passwd

# 提取每行的用户名（以 : 分隔，取第一列）
while IFS=':' read -r username _; do
    echo "用户: $username"
done < /etc/passwd
```

> **说明**：
> - `IFS=` 防止去除行首行尾空格
> - `read -r` 防止反斜杠转义
> - 这是 Shell 中处理文本的标准方式

```bash
# 实战：读取 CSV 文件
while IFS=',' read -r name age city; do
    echo "姓名: $name, 年龄: $age, 城市: $city"
done < data.csv

# 实战：从命令输出逐行读取
ls -l /bin | while read -r line; do
    echo "$line"
done
```

### 8.5 until 循环 -- 条件为假时执行

`until` 与 `while` 相反：条件为**假**时执行循环体，条件为**真**时退出。

```bash
# 语法
until [ 条件 ]; do
    # 条件为假时执行
done

# 示例：等待某个文件出现
until [ -f "/tmp/ready.flag" ]; do
    echo "等待文件出现..."
    sleep 1
done
echo "文件已出现，继续执行"

# 示例：倒计时
count=10
until (( count <= 0 )); do
    echo "倒计时: $count"
    count=$((count - 1))
done
echo "时间到！"

# 等价于
count=10
while (( count > 0 )); do
    echo "倒计时: $count"
    count=$((count - 1))
done
```

### 8.6 循环控制补充

```bash
# break N：跳出 N 层循环（适用于嵌套 while/for/until）
i=1
while [ $i -le 10 ]; do
    j=1
    while [ $j -le 10 ]; do
        if [ $((i * j)) -gt 50 ]; then
            break 2    # 跳出最外层和内层两个 while
        fi
        j=$((j + 1))
    done
    i=$((i + 1))
done
```

---

## 9. 函数定义与调用

### 9.1 函数定义

```bash
# 写法一（推荐）
function_name() {
    # 函数体
}

# 写法二
function function_name {
    # 函数体
}

# 写法三（不常用，写全了反而多余）
function function_name() {
    # 函数体
}
```

### 9.2 函数调用

直接写函数名即可调用：

```bash
# 定义
myfunc() {
    echo "hello from myfunc"
}

# 调用
myfunc
```

### 9.3 示例

```bash
#!/bin/bash
print_name() {
    echo "embedfire"
}

print_name    # 输出：embedfire
```

### 9.4 函数参数（$1, $2, ...）

函数内部用 `$1`、`$2`、`$#`、`$@` 等接收参数，与脚本参数用法完全一致：

```bash
#!/bin/bash

greet() {
    echo "你好, $1！"
    echo "你传入了 $# 个参数"
    echo "所有参数: $@"
}

# 调用函数并传参
greet "Alice" 25 "北京"
# 输出:
# 你好, Alice！
# 你传入了 3 个参数
# 所有参数: Alice 25 北京
```

> **注意**：函数内的 `$1`、`$2` 是函数的参数，不是脚本的命令行参数。脚本的 `$1` 在函数外才有效。

```bash
#!/bin/bash

add() {
    result=$(( $1 + $2 ))
    echo "$1 + $2 = $result"
}

add 10 20    # 输出：10 + 20 = 30
add 3 7      # 输出：3 + 7 = 10
```

### 9.5 函数返回值：return vs echo

Shell 函数有两种"返回"值的方式，用途完全不同：

| 方式 | 作用 | 取值范围 | 获取方式 | 用途 |
|------|------|---------|---------|------|
| `return N` | 返回退出状态码 | 0~255（0表成功） | `$?` | 表示函数执行成功/失败 |
| `echo` | 输出到 stdout | 任意字符串 | `$(函数)` 或 变量捕获 | 返回函数计算结果 |

```bash
#!/bin/bash

# return：返回状态码（0-255），用 $? 获取
is_number() {
    [[ $1 =~ ^[0-9]+$ ]] && return 0
    return 1
}

is_number "123"
echo "123 是数字吗？$?"    # 输出 0（是）
is_number "abc"
echo "abc 是数字吗？$?"    # 输出 1（否）

# echo：返回字符串结果，用 $() 捕获
add() {
    echo $(( $1 + $2 ))
}

sum=$(add 10 20)
echo "和是: $sum"          # 输出：和是: 30
```

### 9.6 局部变量：local 关键字

函数内部默认使用全局变量（会污染外部命名空间）。用 `local` 声明局部变量：

```bash
#!/bin/bash

name="全局变量"    # 全局变量

func_test() {
    local name="局部变量"    # 局部变量，不影响外部
    echo "函数内部: $name"
}

func_test                  # 输出：函数内部: 局部变量
echo "函数外部: $name"     # 输出：函数外部: 全局变量


# 不用 local 的陷阱
func_bad() {
    name="被改了"           # 直接修改了全局变量
}

func_bad
echo "$name"               # 输出：被改了（预期之外！）
```

> **建议**：函数内部变量一律用 `local` 声明，避免意外修改全局变量。

```bash
#!/bin/bash

# 典型用法：函数内用 local 隔离
process_file() {
    local file="$1"
    local count=0            # 局部变量

    while IFS= read -r line; do
        count=$((count + 1))
    done < "$file"

    echo "总行数: $count"    # 通过 echo 输出结果
}

lines=$(process_file "/etc/passwd")
echo "$lines"
```

### 9.7 函数库：用 source 加载

把常用函数放在一个单独文件中，用 `source`（或 `.`）加载到当前脚本：

```bash
# ============ 文件: mylib.sh（函数库）============
#!/bin/bash

# 日志函数
log_info() {
    echo "[INFO] $(date '+%Y-%m-%d %H:%M:%S') $*"
}

log_error() {
    echo "[ERROR] $(date '+%Y-%m-%d %H:%M:%S') $*" >&2
}

# 工具函数
is_root() {
    [ "$(id -u)" -eq 0 ] && return 0 || return 1
}

check_file() {
    local file="$1"
    if [ -f "$file" ] && [ -r "$file" ]; then
        return 0
    fi
    return 1
}

# ============ 文件: main.sh（主脚本）============
#!/bin/bash

# 加载函数库（source 等价于 .）
source ./mylib.sh
# 或：. ./mylib.sh

# 现在可以直接使用 mylib.sh 中定义的函数
log_info "脚本开始执行"

if is_root; then
    log_info "以 root 用户运行"
else
    log_error "请用 root 用户运行此脚本"
    exit 1
fi

if check_file "/etc/passwd"; then
    log_info "/etc/passwd 存在且可读"
fi
```

---

## 10. 逻辑运算回顾

### 逻辑与 &&

```bash
命令1 && 命令2    # 只有命令1成功，才执行命令2
```

### 逻辑或 ||

```bash
命令1 || 命令2    # 只有命令1失败，才执行命令2
```

---

## 11. 综合实战脚本

### 11.1 系统信息收集脚本

```bash
#!/bin/bash

# 系统信息收集脚本
# 运行后输出当前系统的基本信息

echo "==================== 系统信息收集 ===================="
echo ""

echo "=== 系统基本信息 ==="
echo "主机名:      $(hostname)"
echo "内核版本:    $(uname -r)"
echo "系统类型:    $(uname -s)"
echo "硬件架构:    $(uname -m)"
echo "运行时间:    $(uptime -p | sed 's/up //')"

echo ""
echo "=== CPU 信息 ==="
cpu_model=$(grep "model name" /proc/cpuinfo | head -1 | cut -d: -f2 | xargs)
echo "CPU型号:     $cpu_model"
echo "CPU核心数:   $(nproc)"

echo ""
echo "=== 内存信息 ==="
free -h | head -2

echo ""
echo "=== 磁盘信息 ==="
df -h | head -1
df -h | grep "^/dev" | grep -v "loop\|snap"

echo ""
echo "=== 网络信息 ==="
ip -4 addr show | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | while read -r ip; do
    [ "$ip" != "127.0.0.1" ] && echo "IP地址:      $ip"
done

echo ""
echo "=== 当前登录用户 ==="
who

echo ""
echo "====================================================="
```

### 11.2 文件批量处理脚本

```bash
#!/bin/bash

# 批量文件处理脚本
# 功能：将指定目录下所有 .txt 文件转成 .md，并添加标题

SOURCE_DIR="${1:-.}"      # 默认当前目录
LOG_FILE="batch_rename.log"

# 参数检查
if [ ! -d "$SOURCE_DIR" ]; then
    echo "错误: 目录 '$SOURCE_DIR' 不存在"
    exit 1
fi

count=0

# 遍历所有 .txt 文件
for file in "$SOURCE_DIR"/*.txt; do
    # 检查文件是否存在（避免没有匹配文件时的字面量）
    [ -f "$file" ] || continue

    filename=$(basename "$file" .txt)       # 去掉路径和扩展名
    newfile="$SOURCE_DIR/$filename.md"      # 新文件名

    # 如果目标文件已存在，跳过
    if [ -f "$newfile" ]; then
        echo "[跳过] $newfile 已存在"
        continue
    fi

    # 处理文件：加标题行
    {
        echo "# $filename"
        echo ""
        cat "$file"
    } > "$newfile"

    count=$((count + 1))
    echo "[完成] $file -> $newfile" | tee -a "$LOG_FILE"
done

echo ""
echo "共处理 $count 个文件，日志保存在 $LOG_FILE"
```

### 11.3 交互式菜单脚本（完整版）

```bash
#!/bin/bash

# 交互式系统管理菜单

show_menu() {
    clear
    echo "========================================="
    echo "         系统管理菜单 v1.0               "
    echo "========================================="
    echo " 1. 查看系统信息"
    echo " 2. 查看磁盘使用情况"
    echo " 3. 查看内存使用情况"
    echo " 4. 查看网络连接"
    echo " 5. 查看运行中的进程（前10）"
    echo " 6. 清空日志文件"
    echo " 0. 退出"
    echo "========================================="
}

# 主循环
while true; do
    show_menu
    read -p "请输入选项 [0-6]: " choice

    case $choice in
        1)
            echo ""
            echo "=== 系统信息 ==="
            echo "主机名: $(hostname)"
            echo "内核: $(uname -r)"
            echo "运行时间: $(uptime -p | sed 's/up //')"
            echo ""
            read -p "按 Enter 继续..."
            ;;
        2)
            echo ""
            df -h
            echo ""
            read -p "按 Enter 继续..."
            ;;
        3)
            echo ""
            free -h
            echo ""
            read -p "按 Enter 继续..."
            ;;
        4)
            echo ""
            ss -tuln | head -20
            echo ""
            read -p "按 Enter 继续..."
            ;;
        5)
            echo ""
            ps aux --sort=-%cpu | head -11
            echo ""
            read -p "按 Enter 继续..."
            ;;
        6)
            read -p "确认清空 /tmp 下的日志文件？(y/N): " confirm
            if [[ $confirm =~ ^[Yy]$ ]]; then
                rm -f /tmp/*.log 2>/dev/null
                echo "日志文件已清空"
            else
                echo "已取消"
            fi
            read -p "按 Enter 继续..."
            ;;
        0)
            echo "再见！"
            exit 0
            ;;
        *)
            echo "无效选项，请重试"
            sleep 1
            ;;
    esac
done
```

---

## 12. 总结 -- Shell脚本编程全套内容

### 第13讲 上 -- 基础概念

| 主题 | 要点 |
|------|------|
| Shell脚本本质 | 命令按语法组合的程序文件 |
| 内建 vs 外部 | type命令判断；PATH环境变量 |
| 编译型 vs 解析型 | C需编译，Shell直接解析 |
| 第一个脚本 | Shebang `#!/bin/bash` + chmod + 4种启动方式 |

### 第14讲 中 -- 变量与运算

| 主题 | 要点 |
|------|------|
| 变量定义 | 直接赋值 / 单引号 / 双引号（推荐） |
| 变量使用 | `${var}` 加花括号明确边界 |
| 命令赋值 | `$(cmd)` 推荐 |
| 特殊变量 | `$0`, `$1`~`$9`, `$#`, `$*`, `$@`, `$?`, `$$` |
| 输入 | `read -p "提示" 变量` |
| 数学运算 | `$(( ))` |
| 字符串拼接 | 直接并排放 |

### 第15讲 下 -- 条件判断与流程控制

| 主题 | 要点 |
|------|------|
| 条件检测 | `test` / `[ ]`（POSIX标准）/ `[[ ]]`（bash扩展，推荐） |
| 数值比较 | `-eq`, `-ne`, `-gt`, `-lt`, `-ge`, `-le`；或用 `(( ))` 简写 |
| 字符串判断 | `-z`(空), `-n`(非空), `=`(相等), `!=`(不等) |
| 文件判断 | `-d`(目录), `-f`(普通文件), `-e`(存在), `-s`(非空), `-r`(可读), `-w`(可写), `-x`(可执行), `-L`(符号链接) |
| 逻辑组合 | `[ ]` 中用 `-a` `-o` `!`；`[[ ]]` 中用 `&&` `||` `!` |
| 管道 `|` | stdout 传递给下一条命令；`2>&1` 合并 stderr |
| if 语句 | if-fi / if-else-fi / if-elif-else-fi；简写 `[ cond ] && cmd`；判断命令结果 |
| case 语句 | case-esac, `;;` 结束, `*` 默认；支持通配符 `[0-9]` 和 `|` 组合 |
| for 循环 | 列表 / 范围 `{1..10}` / 命令结果 `$(cmd)` / C风格 `((i=0;i<10;i++))` |
| while 循环 | `while [ ]; do done` / `while (( ))` / `while read line` 逐行读取 |
| until 循环 | `until [ ]; do done`（条件为假时执行） |
| break/continue | `break` 跳出循环；`continue` 跳过本轮；`break N` 跳出 N 层 |
| `$*` vs `$@` | 加引号时：`"$*"`整体 vs `"$@"`独立 |
| 函数 | `func() { }` -> 直接调用；`local` 局部变量；`$1 $2` 接收参数 |
| 返回值 | `return` 返回退出码(0~255)；`echo` 返回字符串结果 |
| 函数库 | 用 `source` (或 `.`) 加载外部函数文件 |
| 重定向 | `<< EOF` (here doc) / `<<<` (here string) |

---

*笔记生成日期：2026-07-22 | 转写：Whisper tiny | 优化日期：2026-07-23*
