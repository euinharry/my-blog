---
title: "第16讲：Linux环境变量"
date: 2026-09-11T09:15:00+08:00
draft: false
description: "在Linux中，Shell变量分为两大类：局部变量和环境变量。"
series: ["Linux 入门"]
series_order: 12
categories: ["技术笔记"]
tags: ["Linux", "环境变量"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P17)
> **标题**：第16讲 -- Linux环境变量
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. 环境变量基本概念

在Linux中，Shell变量分为两大类：**局部变量**和**环境变量**。

### 1.1 局部变量（普通Shell变量）

- 使用 `=` 直接定义，例如 `ABC=123`
- **只能在当前Shell进程中访问**
- 子进程**无法**访问父进程的局部变量
- 不同终端/进程之间默认不能互相访问变量

### 1.2 环境变量

- 使用 `export` 命令导出后，变量的作用域扩展到**子进程**
- 当前进程的**子进程可以访问**父进程的环境变量
- 但**孙子进程不能直接访问祖先进程**的变量（只在父子之间传递）

### 1.3 核心区别

| 类型 | 定义方式 | 作用范围 |
|------|---------|---------|
| **局部变量** | `VAR=value` | 仅当前Shell进程 |
| **环境变量** | `export VAR` | 当前进程 + 所有子进程 |

> **关键理解**：export 的本质是让变量**沿着进程树"向下传承"**，而不是让所有进程共享。

---

## 2. 常见系统环境变量

Linux系统预定义了许多环境变量，理解它们是使用Linux的基础。

### 2.1 常见环境变量速查表

| 变量名 | 含义 | 典型值示例 | 说明 |
|--------|------|-----------|------|
| **PATH** | 命令搜索路径 | `/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin` | 执行命令时按此路径查找可执行文件 |
| **HOME** | 当前用户家目录 | `/home/username` | `cd ~` 或 `cd` 默认跳转的目录 |
| **SHELL** | 当前Shell路径 | `/bin/bash` | 当前登录使用的Shell程序 |
| **USER** | 当前用户名 | `username` | 等同于 `whoami` 命令的输出 |
| **LANG** | 系统语言编码 | `zh_CN.UTF-8` 或 `en_US.UTF-8` | 影响终端显示语言和字符编码 |
| **PWD** | 当前工作目录 | `/home/username/work` | `pwd` 命令读取的就是这个变量 |
| **OLDPWD** | 上一个工作目录 | `/home/username` | `cd -` 跳转时读取的变量 |
| **TERM** | 终端类型 | `xterm-256color` | 告诉程序当前终端的显示能力 |
| **LD_LIBRARY_PATH** | 动态库搜索路径 | `/usr/local/lib:/opt/lib` | 程序运行时查找共享库(.so)的路径 |
| **LOGNAME** | 登录用户名 | `username` | 与 USER 类似，但反映登录时的用户名 |
| **HOSTNAME** | 主机名 | `my-server` | 当前机器的主机名 |
| **PS1** | 主提示符 | `[\u@\h \W]\$ ` | 定义终端命令提示符的格式 |
| **EDITOR** | 默认文本编辑器 | `/usr/bin/vim` | 部分命令（如 visudo）使用的编辑器 |

### 2.2 实操示例

```bash
# 查看各个常见环境变量
echo $HOME            # 输出：/home/username
echo $SHELL           # 输出：/bin/bash
echo $USER            # 输出：当前用户名
echo $LANG            # 输出：zh_CN.UTF-8
echo $PWD             # 输出：当前所在目录
echo $OLDPWD          # 输出：上一次所在的目录
echo $TERM            # 输出：xterm-256color
echo $PS1             # 输出：[\u@\h \W]\$  （提示符格式定义）

# 查看当前 PATH
echo $PATH
# 输出示例：
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games
```

---

## 3. 查看环境变量的命令

Linux提供了三个常用命令来查看变量，它们各有侧重。

### 3.1 env 命令

```bash
# 查看所有环境变量（格式化输出）
env

# 查看指定环境变量（方式一：结合 grep）
env | grep PATH

# 查看指定环境变量（方式二：使用 echo，效率更高）
echo $PATH
```

`env` 只显示**环境变量**（被 export 过的），不显示普通局部变量。

### 3.2 printenv 命令

```bash
# 查看所有环境变量（与 env 类似）
printenv

# 查看指定环境变量（直接指定名称，不需要 $ 前缀）
printenv PATH          # 查看 PATH
printenv HOME          # 查看 HOME
printenv SHELL         # 查看 SHELL

# 对比：env 需要配合 grep，printenv 可以直接传参
env | grep PATH        # 绕一点
printenv PATH          # 直接
```

`printenv` 与 `env` 功能几乎相同，但 `printenv` 支持直接传参查看单个变量。

### 3.3 set 命令

```bash
# 查看所有变量（包括环境变量 + 局部变量 + Shell函数）
set

# 输出量很大，建议结合 less 分页查看
set | less

# 配合 grep 过滤
set | grep MY_VAR
```

`set` 显示范围最广：**环境变量 + 局部变量 + Shell函数** 全部列出。

### 3.4 三命令对比总结

| 命令 | 环境变量 | 局部变量 | Shell函数 | 查看单个变量 |
|------|---------|---------|----------|------------|
| **env** | 显示 | 不显示 | 不显示 | 需 grep |
| **printenv** | 显示 | 不显示 | 不显示 | 直接传参 |
| **set** | 显示 | 显示 | 显示 | 需 grep |
| **export** | 显示 | 不显示 | 不显示 | 需 grep（不推荐） |

> **建议**：查看环境变量用 `printenv` 或 `echo $VAR`；排查变量问题用 `set` 可看到全局。

---

## 4. export 命令 -- 导出环境变量

### 4.1 基本用法

```bash
# 方式1：先定义局部变量，再导出
ABC=123
export ABC

# 方式2：一步到位（推荐）
export ABC=123
```

### 4.2 实验验证（父子进程）

通过"父子进程"实验来直观理解：

```bash
# ===== 步骤1：定义局部变量 =====
ABC=123

# ===== 步骤2：查看当前进程中的变量 =====
echo $ABC          # 输出：123 (成功)

# ===== 步骤3：启动子进程 =====
bash               # 进入一个新的子Shell进程

# ===== 步骤4：在子进程中尝试访问 =====
echo $ABC          # 输出：空值 (失败，子进程访问不到父进程的局部变量)

# ===== 步骤5：退出子进程 =====
exit               # 回到父进程

# ===== 步骤6：导出为环境变量 =====
export ABC

# ===== 步骤7：再次启动子进程 =====
bash

# ===== 步骤8：验证子进程能否访问 =====
echo $ABC          # 输出：123 (成功，导出后，子进程可以访问)
exit
```

### 4.3 查看已导出的环境变量

```bash
# 查看所有环境变量
env                # 或
export             # 或
printenv
```

### 4.4 export -n -- 取消导出

```bash
# 将环境变量"降级"为局部变量（不再向子进程传递）
export ABC=123     # 先导出
export -n ABC      # 取消导出：ABC 仍存在，但子进程访问不到了

# 验证
bash -c 'echo "子进程中的ABC: $ABC"'   # 输出：空值（已取消导出）
echo $ABC                               # 输出：123（当前进程仍有值）
```

`export -n` 和 `unset` 的区别：

| 命令 | 效果 |
|------|------|
| `export -n VAR` | 取消导出，变量仍在当前进程（变为局部变量） |
| `unset VAR` | 彻底删除变量，当前进程中也访问不到了 |

### 4.5 declare -x -- 等效于 export

```bash
# 以下两条命令效果完全相同
export MY_VAR=hello
declare -x MY_VAR=hello

# declare -x 也可以取消导出
declare +x MY_VAR      # 等效于 export -n MY_VAR

# 查看所有用 declare 定义的变量（含属性标记）
declare -p

# 查看特定变量的属性（含属性标记）
declare -p PATH
# 输出示例：declare -x PATH="/usr/local/bin:/usr/bin:/bin"
#          其中 -x 表示已导出
```

> **说明**：`declare -x` 与 `export` 在底层是同一机制，`export` 本质上是 `declare -x` 的简化语法。`declare +x` 等效于 `export -n`。

---

## 5. PATH 环境变量专题

PATH 是最重要的环境变量之一，它决定了你在终端中输入命令时，系统去哪里找对应的可执行文件。

### 5.1 PATH 的结构和工作原理

PATH 的值是一个**以冒号 `:` 分隔**的目录列表：

```bash
echo $PATH
# 输出示例：
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games
```

当你输入一条命令（如 `ls`）时，Shell 会：

1. 检查命令是否是内置命令（如 `cd`、`echo`）
2. 检查命令是否是 alias 别名
3. **按 PATH 中的顺序**，从左到右依次在每个目录中查找同名可执行文件
4. 找到第一个匹配的就执行；全部找不到则报错 `command not found`

### 5.2 PATH 搜索顺序（从左到右）

```bash
# 假设 PATH 的值为：
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# 输入 ls 命令时，Shell 按以下顺序查找：
# 1. /usr/local/sbin/ls  -> 找不到
# 2. /usr/local/bin/ls   -> 找不到
# 3. /usr/sbin/ls        -> 找不到
# 4. /usr/bin/ls         -> 找到了，执行！
#
# 后面的 /sbin/ls 和 /bin/ls 不会再被检查
```

> **关键**：PATH 的先后顺序很重要。如果两个目录中都有同名命令，**排在前面的优先**。

### 5.3 which 命令 -- 查找命令的实际位置

`which` 利用 PATH 来告诉你某个命令的完整路径：

```bash
which ls           # 输出：/usr/bin/ls
which python3      # 输出：/usr/bin/python3
which which        # 输出：/usr/bin/which  （which 自己也是一个命令）
which vim          # 输出：/usr/bin/vim

# 如果命令不在 PATH 中
which mycommand    # 输出：空（没有输出）

# 查看所有匹配项（而非第一个）
which -a python3   # 可能输出多个路径
```

补充命令 `type` 也可以显示命令的来源：

```bash
type ls            # 输出：ls is aliased to `ls --color=auto'  （可能是别名）
type cd            # 输出：cd is a shell builtin                 （内置命令）
type python3       # 输出：python3 is /usr/bin/python3            （外部命令）
```

### 5.4 添加自定义路径到 PATH

```bash
# 临时添加（关闭终端后失效）
export PATH=$PATH:/my/custom/path

# 添加到 PATH 末尾（优先级低，兜底用）
export PATH=$PATH:/home/user/mybin

# 添加到 PATH 开头（优先级高，优先生效）
export PATH=/home/user/mybin:$PATH
```

**实操示例：将自己写的脚本目录加入 PATH**：

```bash
# 步骤1：创建目录
mkdir -p ~/mybin

# 步骤2：编写一个简单脚本
cat > ~/mybin/hello << 'EOF'
#!/bin/bash
echo "Hello from my custom command!"
EOF

# 步骤3：赋予执行权限
chmod +x ~/mybin/hello

# 步骤4：添加到 PATH
export PATH=$PATH:~/mybin

# 步骤5：验证（可以直接调用，不需要 ./ 前缀）
hello
# 输出：Hello from my custom command!

# 步骤6：确认位置
which hello
# 输出：/home/username/mybin/hello
```

### 5.5 持久化 PATH 配置

临时 `export` 在关闭终端后就失效了。要让 PATH 永久生效，需要写入配置文件：

```bash
# 写入 ~/.bashrc（推荐，当前用户生效）
echo 'export PATH=$PATH:$HOME/mybin' >> ~/.bashrc

# 立即生效（不关终端）
source ~/.bashrc
# 或
. ~/.bashrc

# 写入 /etc/bash.bashrc（全局所有用户生效，需要 sudo）
sudo sh -c 'echo "export PATH=\$PATH:/opt/shared/bin" >> /etc/bash.bashrc'
```

### 5.6 PATH 配置最佳实践

```bash
# 推荐：优先添加到开头（确保自己的版本优先生效）
export PATH=$HOME/mybin:$PATH

# 推荐：避免重复添加（先检查再添加）
if [[ ":$PATH:" != *":$HOME/mybin:"* ]]; then
    export PATH="$HOME/mybin:$PATH"
fi

# 风险提醒：谨慎使用，以下写法有安全风险
# export PATH=.:$PATH   # 把当前目录放在 PATH 开头 -- 危险！容易被恶意程序劫持
```

---

## 6. 不同进程间共享环境变量

### 6.1 export 的局限性

通过 `export` 只能让**子进程**访问环境变量，但：

- **不同终端**（不同进程树）之间仍然**不能互相访问**
- 在一个终端 export 的变量，切换到另一个终端窗口就看不到了

### 6.2 解决方案：Shell配置文件

要让所有进程（包括不同终端）都能访问某个变量，需要将变量定义写入 **Shell配置文件** 中。

Shell 在启动时会自动加载配置文件，从而实现"所有进程共享变量"。

---

## 7. Shell配置文件总览（以 bash 为例）

### 7.1 配置文件一览

| 文件 | 作用域 | 执行时机 |
|------|--------|---------|
| `/etc/profile` | **全局**（所有用户） | 用户**登录**时执行（仅一次） |
| `/etc/bash.bashrc` | **全局**（所有用户） | **新开终端**时执行 |
| `~/.profile` | **当前用户** | 用户**登录**时执行（仅一次） |
| `~/.bashrc` | **当前用户** | **新开终端**时执行 |
| `/etc/profile.d/*.sh` | **全局** | 被 `/etc/profile` 调用（登录时） |
| `~/.bash_profile` | **当前用户** | 用户**登录**时执行（优先级高于 ~/.profile） |

> **说明**：`/etc/` 下的配置文件对所有用户生效；`~/`（用户家目录）下的仅对当前用户生效。

### 7.2 登录 vs 新开终端

| 操作 | 触发条件 | 执行的配置文件 |
|------|---------|--------------|
| **登录** | 输入用户名密码登录系统 / SSH登录 | `/etc/profile` -> `~/.bash_profile`（或 `~/.profile`） |
| **新开终端** | 在桌面环境中打开新的终端窗口 | `/etc/bash.bashrc` -> `~/.bashrc` |

### 7.3 /etc/profile.d/ 目录 -- 模块化配置

`/etc/profile.d/` 是一个**存放独立配置脚本**的目录，它不是直接执行，而是被 `/etc/profile` 通过以下方式加载：

```bash
# /etc/profile 中的典型代码片段
if [ -d /etc/profile.d ]; then
    for i in /etc/profile.d/*.sh; do
        if [ -r $i ]; then
            . $i      # 使用 source 方式逐个执行
        fi
    done
fi
```

**优点**：
- 各软件可以独立维护自己的环境变量配置，不需要修改 `/etc/profile` 主文件
- 卸载软件时只需删除对应的 `.sh` 文件
- 比直接修改 `/etc/profile` 更清晰、更安全

**常见示例**：

```bash
# 查看 profile.d 目录内容
ls /etc/profile.d/
# 示例输出：
#   vim.sh          -> 配置 vim 相关环境
#   bash_completion.sh -> 启用命令自动补全
#   gawk.sh         -> 配置 gawk
#   snapd.sh        -> Snap 包管理器的环境
```

### 7.4 source 命令 -- 重新加载配置

修改配置文件后，不需要注销重新登录，用 `source` 即可让配置立即生效：

```bash
# 编辑配置文件
vim ~/.bashrc

# 使修改立即生效（三种等效写法）
source ~/.bashrc
. ~/.bashrc
exec bash           # 替换当前Shell为新Shell（相当于重新登录）
```

> `source` 和 `.` 完全等价，`.` 是 POSIX 标准写法，`source` 是 bash 的别名。

### 7.5 不同 Shell 的配置文件差异

不同 Shell 使用不同的配置文件，移植配置时需要注意：

| 功能 | Bash | Zsh | Fish |
|------|------|-----|------|
| 登录时执行 | `~/.bash_profile` 或 `~/.profile` | `~/.zprofile` | 不支持此模式 |
| 每次开终端 | `~/.bashrc` | `~/.zshrc` | `~/.config/fish/config.fish` |
| 全局配置 | `/etc/bash.bashrc` | `/etc/zsh/zshrc` | `/etc/fish/config.fish` |
| 退出时执行 | `~/.bash_logout` | `~/.zlogout` | 不支持 |
| 导出变量语法 | `export VAR=val` | `export VAR=val` | `set -x VAR val` |
| 添加PATH | `export PATH=$PATH:...` | `export PATH=$PATH:...` | `fish_add_path /path` |

> **当前常见的是哪个 Shell？**
> ```bash
> echo $SHELL       # 查看当前使用的 Shell
> cat /etc/shells   # 查看系统安装的所有 Shell
> ```

---

## 8. 配置文件执行顺序

### 8.1 完整执行流程

```
用户登录系统（login shell）
  |
  +-- /etc/profile（全局 -- 登录时执行）
        |
        +-- /etc/bash.bashrc（全局 bashrc，被 /etc/profile 调用）
        |
        +-- /etc/profile.d/*.sh（遍历执行所有 .sh 脚本）
        |
        +-- 查找用户级 profile（按优先级）：
              |
              +-- ~/.bash_profile（优先级最高，如果存在）
              |
              +-- ~/.bash_login（如果上面不存在）
              |
              +-- ~/.profile（兜底，如果上面都不存在）
                    |
                    +-- ~/.bashrc（被用户 profile 调用）
```

**新开终端（non-login shell）**：

```
打开新终端窗口（non-login shell）
  |
  +-- /etc/bash.bashrc（全局 bashrc）
  |
  +-- ~/.bashrc（用户 bashrc）
```

### 8.2 关键调用关系

| 调用者 | 被调用者 | 说明 |
|--------|---------|------|
| `/etc/profile` | `/etc/bash.bashrc` | 全局 profile 调用全局 bashrc |
| `/etc/profile` | `/etc/profile.d/*.sh` | 遍历加载所有全局模块脚本 |
| `~/.profile`（或 `~/.bash_profile`） | `~/.bashrc` | 用户 profile 调用用户 bashrc |

### 8.3 优先级总结

| 优先级 | 配置文件 | 覆盖关系 |
|--------|---------|---------|
| 1 | 终端中手动 `export` | 最高，但仅当前会话 |
| 2 | `~/.bashrc` | 覆盖系统级配置 |
| 3 | `~/.profile` / `~/.bash_profile` | 覆盖系统级配置 |
| 4 | `/etc/bash.bashrc` | 系统默认 |
| 5 | `/etc/profile` | 系统默认（最先执行） |

> **记忆技巧**：
> - **profile** = 登录时执行一次（配置用户环境）
> - **bashrc** = 每次开终端都执行（配置终端环境）
> - profile 内部会调用 bashrc，所以定义在 bashrc 中的变量在登录时也能生效
> - **后执行的覆盖先执行的**，用户级覆盖全局级

---

## 9. 配置文件的作用范围

### 9.1 选择策略速查表

| 需求 | 定义位置 | 示例 |
|------|---------|------|
| **所有用户 + 所有进程** 共享变量 | `/etc/bash.bashrc` | 系统级环境变量 |
| **当前用户 + 所有进程** 共享变量 | `~/.bashrc` | 个人常用环境变量 |
| 当前用户仅登录时生效 | `~/.profile` | 登录欢迎信息 |
| 临时测试用 | 直接在终端 `export` | 临时调试 |

### 9.2 实验1：`/etc/bash.bashrc`（全局）

```bash
# ===== 步骤1：编辑全局配置文件 =====
sudo vim /etc/bash.bashrc

# 在文件末尾添加以下内容：
export CD=12345

# ===== 步骤2：新开终端（任何用户均可） =====
# 打开新的终端窗口
echo $CD          # 输出：12345 (成功)

# ===== 步骤3：切换到其他用户验证 =====
su xiaoming       # 切换到用户 xiaoming
echo $CD          # 输出：12345 (成功，其他用户也能访问)
```

> **结论**：`/etc/bash.bashrc` 中定义的变量，系统所有用户都能访问。

### 9.3 实验2：`~/.bashrc`（当前用户）

```bash
# ===== 步骤1：编辑当前用户的配置文件 =====
vim ~/.bashrc

# 在文件末尾添加以下内容：
export GG=66

# ===== 步骤2：新开终端 =====
# 打开新的终端窗口（以当前用户身份）
echo $GG          # 输出：66 (成功)

# ===== 步骤3：切换到其他用户验证 =====
su xiaoming
echo $GG          # 输出：空值 (失败，其他用户不能访问)
```

> **结论**：`~/.bashrc` 中定义的变量只在当前用户下有效，其他用户无法访问。

---

## 10. Shell脚本启动方式对变量的影响

Shell脚本有 **4种启动方式**，分为两大类。

### 10.1 方式分类

| 类别 | 启动方式 | 命令 |
|------|---------|------|
| **第一类：当前进程执行** | source 方式 | `source 脚本.sh` |
| | 点命令方式 | `. 脚本.sh` |
| **第二类：子进程执行** | bash 方式 | `bash 脚本.sh` |
| | 直接执行方式 | `./脚本.sh`（需执行权限） |

### 10.2 两类方式的本质区别

| 类别 | 执行机制 | 脚本中的变量能否被当前进程访问？ |
|------|---------|--------------------------------|
| **当前进程执行** | 脚本在当前Shell进程中执行 | (能) |
| **子进程执行** | 脚本在子Shell进程中执行 | (不能) |

### 10.3 实验演示

**准备测试脚本**：

```bash
#!/bin/bash
# test.sh
A=11
B=22
```

**四种启动方式的对比实验**：

```bash
# ==============================
# 第一类：当前进程执行（变量可被访问）
# ==============================

# 方式1：source 命令
source test.sh
echo $A           # 输出：11 (成功)
echo $B           # 输出：22 (成功)

# 清理变量
unset A B

# 方式2：. 点命令（source 的简写）
. test.sh
echo $A           # 输出：11 (成功)
echo $B           # 输出：22 (成功)

# 清理变量
unset A B

# ==============================
# 第二类：子进程执行（变量不可被访问）
# ==============================

# 方式3：bash 命令
bash test.sh
echo $A           # 输出：空值 (失败)
echo $B           # 输出：空值 (失败)

# ==============================
# 方式4：./ 直接执行（需先赋予执行权限）
chmod +x test.sh
./test.sh
echo $A           # 输出：空值 (失败)
echo $B           # 输出：空值 (失败)
```

### 10.4 总结对比

| 启动方式 | 命令 | 语法 | 前置要求 | 变量可访问？ |
|----------|------|------|---------|------------|
| source | `source test.sh` | 标准写法 | 无 | 可以 |
| 点命令 | `. test.sh` | source 的简写 | 无 | 可以 |
| bash | `bash test.sh` | 显式指定解释器 | 无 | 不可以 |
| 直接执行 | `./test.sh` | 直接运行脚本 | 需要 `chmod +x` | 不可以 |

> **核心结论**：
> - **source / .**  = 在当前进程中执行 -> 变量保留在当前Shell
> - **bash / ./** = 在新的子进程中执行 -> 变量随子进程结束而消失
>
> 这个区别是理解 Shell 环境变量与进程关系的关键。

---

## 11. 进程与环境变量

### 11.1 进程树（pstree）

Linux 中所有进程构成一棵树，环境变量沿这棵树向下传递。

```bash
# 查看当前 Shell 的进程树
pstree -p $$

# 输出示例（简化）：
# systemd(1)---sshd(1234)---sshd(5678)---bash(5679)---pstree(9999)
#                                              ^
#                                         当前 Shell
# $$
# 表示当前 Shell 进程的 PID

# 查看完整的进程树
pstree -a          # 显示命令行参数
pstree -u          # 显示用户切换

# 查看特定用户的进程树
pstree username
```

**进程树示例图解**：

```
init(1)
  +-- sshd(100)
        +-- sshd(200)           <- SSH 登录连接
              +-- bash(201)     <- 用户 A 的 login shell
                    +-- vim(300)   <- 子进程：继承 bash(201) 的环境变量
                    +-- bash(301)  <- 手动启动的子 Shell
                          +-- python3(400) <- 孙子进程
```

环境变量沿进程树**向下**传递：bash(201) 的 export 变量 -> bash(301) 可以访问 -> python3(400) 也可以访问。

### 11.2 env -i -- 清除环境变量启动命令

有时需要在一个干净的环境中运行程序，排除所有环境变量的干扰：

```bash
# 查看正常环境
env | wc -l          # 输出：40+ 个环境变量

# 清除所有环境变量后启动命令
env -i bash          # 启动一个"干净"的 Shell

# 在干净环境中验证
env
# 输出：几乎为空

# 只保留需要的变量
env -i PATH=/usr/bin:/bin HOME=$HOME bash

# 实用场景：测试程序在最小环境下的行为
env -i /path/to/program
```

### 11.3 set -a -- 自动导出所有变量

```bash
# 开启自动导出模式（aloexport）
set -a

# 此后所有定义的变量自动导出
MY_VAR=hello        # 不需要 export，自动成为环境变量
OTHER=world

# 验证
bash -c 'echo $MY_VAR $OTHER'   # 输出：hello world (成功)

# 关闭自动导出模式
set +a

# 关闭后定义的变量恢复为普通局部变量
NEW_VAR=test
bash -c 'echo $NEW_VAR'         # 输出：空值 (失败)
```

> **使用场景**：当需要在一个脚本中定义大量环境变量时，`set -a` 可以减少重复写 `export` 的次数。
>
> **注意**：`set -a` 是 bash 特性，不是 POSIX 标准，在 `#!/bin/sh` 脚本中可能不可用。

---

## 12. 补充实用内容

### 12.1 alias -- 命令别名

别名本身不是环境变量，但通常配置在 `~/.bashrc` 中，与环境变量配合使用。

```bash
# 定义别名
alias ll='ls -alF'
alias la='ls -A'
alias ..='cd ..'
alias ...='cd ../..'

# 查看所有别名
alias

# 查看特定别名
alias ll

# 删除别名
unalias ll

# 临时绕过别名（使用原始命令）
\ls              # 反斜杠前缀绕过别名
command ls       # command 前缀绕过别名
```

**常用别名配置示例（写入 ~/.bashrc）**：

```bash
# 安全类
alias rm='rm -i'            # 删除前确认
alias cp='cp -i'            # 覆盖前确认
alias mv='mv -i'            # 移动/覆盖前确认

# 便捷类
alias ll='ls -alF'
alias grep='grep --color=auto'
alias df='df -h'            # 人类可读的磁盘使用量
alias free='free -h'        # 人类可读的内存使用量

# Git 快捷命令
alias gs='git status'
alias ga='git add'
alias gc='git commit'
alias gp='git push'
```

### 12.2 .env 文件 -- 项目级环境变量

`.env` 文件是**现代开发中常见**的环境变量管理方式，通常用于存储敏感或项目特有的配置。

**`.env` 文件的特点**：

- 是一个纯文本文件，格式为 `KEY=VALUE`
- 通常放在项目根目录
- **不会被 Git 提交**（添加到 `.gitignore`）
- 不同环境（开发、测试、生产）使用不同的 `.env` 文件

```bash
# .env 文件示例
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
API_KEY=sk-abc123def456
DEBUG=true
PORT=3000
```

**在 Shell 脚本中加载 .env 文件**：

```bash
# 方式1：使用 source（仅适用于没有空格和特殊字符的简单格式）
set -a                      # 开启自动导出
source .env                 # 加载 .env 文件
set +a                      # 关闭自动导出

# 方式2：使用 export $(...) 语法
export $(cat .env | xargs)

# 方式3：逐行读取（最安全，处理注释和空格）
while IFS='=' read -r key value; do
    # 跳过空行和注释
    [[ -z "$key" || "$key" =~ ^# ]] && continue
    export "$key=$value"
done < .env
```

> **安全提醒**：`.env` 文件通常包含密码和 API 密钥，务必确保 `.gitignore` 中已添加 `.env`。

### 12.3 完整实操：添加自定义命令到 PATH

本节演示从一个脚本到全局可用的完整流程：

```bash
# ===== 步骤1：创建个人命令目录 =====
mkdir -p ~/mybin

# ===== 步骤2：编写自定义命令脚本 =====
cat > ~/mybin/backup-code << 'SCRIPT'
#!/bin/bash
# backup-code: 快速备份当前目录到带时间戳的 tar.gz
if [ $# -eq 0 ]; then
    echo "用法: backup-code <备份名称>"
    exit 1
fi
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
FILENAME="${1}_${TIMESTAMP}.tar.gz"
tar -czf "$FILENAME" .
echo "备份完成: $FILENAME"
SCRIPT

# ===== 步骤3：赋予执行权限 =====
chmod +x ~/mybin/backup-code

# ===== 步骤4：添加到 PATH（写入 ~/.bashrc） =====
cat >> ~/.bashrc << 'EOF'
# 添加个人命令目录到 PATH
export PATH=$HOME/mybin:$PATH
EOF

# ===== 步骤5：重新加载配置 =====
source ~/.bashrc

# ===== 步骤6：验证 =====
which backup-code
# 输出：/home/username/mybin/backup-code

# ===== 步骤7：使用 =====
mkdir test-dir && cd test-dir
touch a.txt b.txt
backup-code myproject
# 输出：备份完成: myproject_20260723_153000.tar.gz
ls *.tar.gz
# 输出：myproject_20260723_153000.tar.gz
```

---

## 13. 知识总结

### 13.1 核心概念回顾

```
局部变量 --export--> 环境变量 --子进程继承--> 沿进程树向下传递
                              |
配置文件中 export --自动加载--> 所有终端/进程都可以访问
```

### 13.2 环境变量相关命令速查

| 命令 | 用途 |
|------|------|
| `echo $VAR` | 查看单个变量值 |
| `printenv VAR` | 查看单个环境变量 |
| `printenv` / `env` | 查看所有环境变量 |
| `set` | 查看所有变量（含局部变量和函数） |
| `export VAR=val` | 导出环境变量 |
| `export -n VAR` | 取消导出（保留为局部变量） |
| `declare -x VAR=val` | 等效于 export |
| `declare +x VAR` | 等效于 export -n |
| `unset VAR` | 彻底删除变量 |
| `which CMD` | 查找命令在 PATH 中的位置 |
| `type CMD` | 显示命令类型（别名/内置/外部） |
| `env -i CMD` | 在干净环境中执行命令 |
| `set -a` | 开启自动导出模式 |
| `source FILE` / `. FILE` | 在当前Shell中加载文件 |
| `pstree -p $$` | 查看当前进程树 |

### 13.3 配置文件速查

| 配置文件 | 作用范围 | 时机 |
|---------|---------|------|
| `/etc/profile` | 全局 | 登录时 |
| `/etc/bash.bashrc` | 全局 | 每次开终端 |
| `/etc/profile.d/*.sh` | 全局（模块化） | 登录时（被 /etc/profile 调用） |
| `~/.bash_profile` | 用户 | 登录时（优先级高） |
| `~/.profile` | 用户 | 登录时（兜底） |
| `~/.bashrc` | 用户 | 每次开终端 |

### 13.4 脚本执行方式速查

| 方式 | 变量是否保留 | 使用场景 |
|------|------------|---------|
| `source script.sh` / `. script.sh` | 保留 | 加载配置文件、设置环境 |
| `bash script.sh` / `./script.sh` | 不保留 | 执行独立任务 |

### 13.5 实战建议

| 场景 | 推荐做法 |
|------|---------|
| 个人常用环境变量 | 写入 `~/.bashrc` |
| 团队/系统共享环境变量 | 写入 `/etc/bash.bashrc` |
| 软件独立配置 | 写入 `/etc/profile.d/软件名.sh` |
| 项目敏感变量 | 使用 `.env` 文件（不提交Git） |
| 一次性临时变量 | 直接终端 `export` |
| 让脚本定义的变量在当前终端生效 | 用 `source` 或 `.` 执行脚本 |
| 自定义命令 | 创建 `~/mybin/`，加入 PATH |
| 需要大面积导出变量 | 使用 `set -a` |

---

*笔记生成日期：2026-07-23 | 转写：Whisper tiny | 优化补充：2026-07-23*
