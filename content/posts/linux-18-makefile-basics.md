---
title: "第30讲：Makefile基础"
date: 2026-09-11T09:09:00+08:00
draft: false
description: "gcc hello.c -o hello"
series: ["Linux 入门"]
series_order: 18
categories: ["技术笔记"]
tags: ["Linux", "Makefile"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P31)
> **标题**：第30讲 — Makefile基础
> **时长**：12分38秒
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. 为什么需要Makefile？

### 1.1 单文件编译

```bash
gcc hello.c -o hello
```

一个源文件时，直接在终端用 gcc 命令就可以完成编译。

### 1.2 多文件编译的问题

当项目文件增多时：

```bash
gcc a.c b.c c.c d.c e.c -o program
```

| 问题 | 说明 |
|------|------|
| **输入繁琐** | 每次要输入所有文件名，易出错 |
| **重复编译** | 即使只改了一个文件，也要重新编译所有文件 |
| **浪费时间** | 项目越大，浪费越严重 |

### 1.3 一个实际的多文件项目场景

假设你有一个简单的项目，目录结构如下：

```
myproject/
├── main.c          # 主程序
├── math_utils.c    # 数学工具函数
├── math_utils.h    # 数学工具头文件
├── string_utils.c  # 字符串工具函数
├── string_utils.h  # 字符串工具头文件
├── logger.c        # 日志模块
└── logger.h        # 日志头文件
```

**手动编译的方式**（每次改一个文件都要重复）：

```bash
# 没有 Makefile 时，每次都要输入完整命令：
gcc -c main.c -o main.o
gcc -c math_utils.c -o math_utils.o
gcc -c string_utils.c -o string_utils.o
gcc -c logger.c -o logger.o
gcc main.o math_utils.o string_utils.o logger.o -o myprogram

# 如果只改了一行 math_utils.c，上面的命令仍然要全部执行一遍
# 项目有几十个文件时，这种方式根本不可行
```

### 1.4 Makefile 方案：只编译改动过的文件

有了 Makefile 之后，同样的项目只需要：

```bash
# 第一次编译：
make
# 输出：编译所有 .c 文件，链接生成 myprogram

# 修改了 math_utils.c 后再次编译：
make
# 输出：只重新编译 math_utils.c，其他文件跳过

# 这就是"增量编译"的核心价值
```

**类比说明**：如果把编译项目比作做一桌菜，没有 Makefile 就像每次有人来吃饭都把所有菜重新做一遍，哪怕只少了一双筷子。有了 Makefile 就像有个聪明的厨师长：他看一眼厨房，知道哪些菜已经做好了不需要重做，只把需要重做的菜再做一遍。

> **核心思想**：Makefile 的价值在于**增量编译** —— 只编译那些源代码发生了变化的文件。

---

## 2. Make工具与Makefile的关系

### 2.1 各自职责

| 工具 | 作用 |
|------|------|
| **Make工具** | ① 找出修改过的文件 → ② 根据依赖关系找出受影响文件 → ③ 按规则逐个编译 |
| **Makefile文件** | 记录文件之间的**依赖关系**和**编译规则** |

### 2.2 工作流程

```
修改源代码
    ↓
Make工具检查文件修改时间
    ↓
读取Makefile中的依赖关系
    ↓
找出需要重新编译的文件
    ↓
按Makefile中的规则逐个编译
    ↓
只编译受影响的部分，避免全量编译
```

> **核心思想**：Make + Makefile = 自动化的增量编译

### 2.3 增量编译原理：文件时间戳比较

Make 工具判断一个文件是否需要重新编译，依据的是文件的**修改时间戳（mtime）**。

**核心规则**：如果目标文件（如 `.o`）比它的所有依赖文件（如 `.c` 和 `.h`）都新，就不需要重新编译；否则就需要。

**实操演示**：

```bash
# 创建测试文件
cat > hello.c << 'EOF'
#include <stdio.h>
int main() {
    printf("Hello\n");
    return 0;
}
EOF

# 编译生成可执行文件
gcc hello.c -o hello

# 查看文件时间戳
ls -l hello.c hello
# 输出示例：
# -rw-r--r-- 1 user user 102 Jul 24 10:00 hello.c
# -rwxr-xr-x 1 user user 16K Jul 24 10:01 hello    # 比 hello.c 新

# 此时如果执行 make，make 会检查：
#   hello 的修改时间（10:01）> hello.c 的修改时间（10:00）
#   结论：hello 是最新的，不需要重新编译 ✓

# 修改 hello.c
touch hello.c    # 模拟修改，更新了 hello.c 的时间戳

# 再次查看
ls -l hello.c hello
# 现在 hello.c 的时间戳（10:05）> hello 的时间戳（10:01）
# make 会判断：hello 过期了，需要重新编译！
```

**Makefile 中的写法**：

```makefile
# 规则格式：目标: 依赖
#          (Tab) 执行命令
hello: hello.c
	gcc hello.c -o hello
```

当执行 `make` 时，make 工具会：
1. 比较 `hello` 和 `hello.c` 的时间戳
2. 如果 `hello.c` 比 `hello` 新（或 `hello` 不存在），执行 `gcc hello.c -o hello`
3. 如果 `hello` 已经比 `hello.c` 新，跳过，输出 `make: `hello` is up to date.`

---

## 3. Makefile 基本规则格式详解

### 3.1 规则的三要素

每一条 Makefile 规则由三部分组成：

| 要素 | 说明 | 示例 |
|------|------|------|
| **target（目标）** | 要生成的文件名（通常是可执行文件或 .o 文件） | `hello`，`main.o` |
| **prerequisites（依赖）** | 生成目标所需要的文件列表 | `hello.c`，`main.c math_utils.h` |
| **recipe（命令）** | 生成目标所要执行的 shell 命令 | `gcc hello.c -o hello` |

**标准格式**：

```makefile
target: prerequisites
	recipe
```

**关键规则**：
- `target` 和 `:` 之间没有空格，`:` 后面有一个空格
- `recipe` 前面必须是一个 **Tab 字符**（不是空格），这是 Makefile 最经典的坑
- 一个目标可以有多个依赖，用空格分隔
- 一个目标可以有多行 recipe，每条 recipe 前面都要有 Tab

### 3.2 完整的 Makefile 示例

下面是一个多文件项目的完整 Makefile 示例：

```makefile
# 这是注释，用 # 开头
# Makefile 文件名：Makefile 或 makefile 或 GNUmakefile

# 每条规则格式：目标: 依赖
#               (Tab)命令

# 最终目标：可执行文件 myprogram
myprogram: main.o math_utils.o string_utils.o logger.o
	gcc main.o math_utils.o string_utils.o logger.o -o myprogram

# 每个 .o 文件的编译规则
# 它们同时依赖对应的 .c 文件和 .h 头文件
main.o: main.c math_utils.h string_utils.h logger.h
	gcc -c main.c -o main.o

math_utils.o: math_utils.c math_utils.h
	gcc -c math_utils.c -o math_utils.o

string_utils.o: string_utils.c string_utils.h
	gcc -c string_utils.c -o string_utils.o

logger.o: logger.c logger.h
	gcc -c logger.c -o logger.o
```

**这个 Makefile 的工作逻辑**：

1. 执行 `make`，默认处理第一个目标 `myprogram`
2. make 检查 `myprogram` 依赖 `main.o math_utils.o string_utils.o logger.o`
3. 分别检查每个 `.o` 文件是否存在且比其依赖文件更新
4. 如果某个 `.o` 过期，执行对应的编译命令
5. 所有 `.o` 就绪后，执行链接命令生成 `myprogram`

### 3.3 规则的执行顺序

```makefile
# 执行 make 时，默认只构建第一个目标
# 假设有以下 Makefile：

all: program    # 第一个目标，make 默认执行这个

program: a.o b.o
	gcc a.o b.o -o program

a.o: a.c
	gcc -c a.c -o a.o

b.o: b.c
	gcc -c b.c -o b.o

clean:          # 这个目标不会自动执行
	rm -f program a.o b.o
```

```bash
# 构建项目（默认执行第一个目标 all）
make

# 清理编译产物（显式指定目标 clean）
make clean

# 只编译 a.o（指定中间目标）
make a.o
```

**类比说明**：可以把 Makefile 想象成一份施工图纸。`target` 是你要建造的东西（一栋楼），`prerequisites` 是建筑材料（砖、水泥），`recipe` 是施工步骤（砌墙、浇筑）。make 工具就是一个聪明的工头，他看一眼图纸就知道哪些材料准备好了，哪些还需要加工，然后只安排必要的工序。

---

## 4. 学习Makefile的必要性

### 4.1 工程能力的体现

| 对比 | 初级程序员 | 中高级程序员 |
|------|-----------|-------------|
| 关注点 | 实现功能 | 工程管理能力 |
| 工具使用 | 依赖IDE | 掌控编译链接全过程 |
| 项目理解 | 会写代码 | 能分析开源项目 |

### 4.2 嵌入式Linux的特点

- **没有IDE** 集成开发环境
- 程序所有控制权在开发者手上
- 必须深入了解：
  - 底层编译过程
  - 链接原理
  - 库加载机制

### 4.3 后续课程需要

以后的课程涉及：
- **u-boot移植**
- **内核移植**
- **其他开源项目**

如果不熟悉Makefile，连如何分析项目都不知道。

### 4.4 知识层次图

```
┌─────────────────────────────────────┐
│     项目（u-boot/内核/开源项目）       │
├─────────────────────────────────────┤
│          Makefile 管理               │
│   （编译 / 链接 / 库加载）             │
├─────────────────────────────────────┤
│       底层知识（编译/链接/库）         │
└─────────────────────────────────────┘
```

> **背景解释**：Linux 内核、u-boot、busybox 这些嵌入式核心项目全部使用 Makefile 管理编译。打开任何一个这样的项目，第一眼看到的一定是 Makefile（通常是顶层的 `Makefile` 加上各个子目录中的 `Makefile` 或 `Kbuild`）。不会读 Makefile，就等于看不懂这些项目的"总控开关"。

---

## 5. 如何学习Makefile

### 5.1 核心思想

> **Makefile管理项目，本质就是管理文件之间的依赖关系**

- 所有语法都是为了更好地解决**依赖问题**
- 不要死记硬背语法
- 多思考"这个语法为了解决什么依赖问题"

### 5.2 掌握标准

> 当你的大脑能像Make工具一样准确解析Makefile时，你就是Makefile高手。

标准：脑海中非常清晰地知道 **目标与依赖之间的关系**。

### 5.3 学习路线建议

| 阶段 | 学习内容 | 练习目标 |
|------|---------|---------|
| 入门 | 基本规则格式（目标:依赖） | 能写出单文件的 Makefile |
| 进阶 | 变量、自动变量、隐含规则 | 能用变量简化重复内容 |
| 实战 | 条件判断、函数、多目录管理 | 能阅读 u-boot/内核的 Makefile |
| 精通 | 嵌套 Make、子目录递归、交叉编译集成 | 能独立管理大型嵌入式项目 |

> **初学者提示**：不要一开始就试图理解 Linux 内核那几千行的 Makefile。先从 10 行的简单 Makefile 开始，理解了"依赖关系"这个核心概念后，再逐步接触变量、函数等高级特性。

---

## 6. Makefile语法框架总览

### 6.1 思维导图框架

```
Makefile
├── ① 基本语法（核心）
│     └── 定义目标与依赖关系的格式
├── ② 变量
│     └── 记录特定信息，避免重复输入
├── ③ 条件判断
│     └── 控制编译器编译不同内容
├── ④ 头文件依赖
│     └── 管理头文件的依赖关系
├── ⑤ 隐含规则（模式规则）
│     └── 减少Makefile编写工作量
├── ⑥ 自动变量
│     └── 配合模式规则使用
└── ⑦ 函数
      └── Makefile内置的丰富功能
```

### 6.2 各语法的作用

| 语法 | 出现背景 | 解决的问题 |
|------|---------|-----------|
| **基本语法** | 需要定义目标与依赖 | 核心，建立依赖关系 |
| **变量** | 避免重复输入长字符串 | 提高可维护性 |
| **条件判断** | 需要不同平台不同编译方式 | 提高灵活性 |
| **头文件依赖** | 头文件修改也要重新编译 | 完善依赖管理 |
| **隐含规则** | 基本语法写起来太繁琐 | 简化编写 |
| **自动变量** | 配合模式规则使用 | 进一步简化 |
| **函数** | 需要复用常见功能 | 避免自己实现 |

### 6.3 核心思想

> **基本语法是核心！其他所有语法都是为基本语法服务的。**

| 阶段 | 方式 | 特点 |
|------|------|------|
| 基础实现 | 只用基本语法 | 可以实现功能，但冗余、难维护 |
| 进阶优化 | 加入变量/规则/函数 | 更优雅、简洁、可复用 |

这与编程思想一致：不仅要实现功能，还要兼顾**可读性、可移植性、可维护性**。

### 6.4 各语法特性示例

下面用同一个项目（编译 `main.c` 和 `utils.c` 生成 `program`）来对比不同语法的效果。

#### ① 基本语法（最原始版本）

```makefile
program: main.o utils.o
	gcc main.o utils.o -o program

main.o: main.c utils.h
	gcc -c main.c -o main.o

utils.o: utils.c utils.h
	gcc -c utils.c -o utils.o

clean:
	rm -f program main.o utils.o
```

**问题分析**：当文件数量增多时（比如 50 个 `.c` 文件），你需要为每个文件写一条几乎完全相同的规则，`main.c` 和 `utils.c` 在规则中出现了多次，修改起来很痛苦。

#### ② 引入变量

```makefile
# 用变量避免重复
CC       = gcc
CFLAGS   = -Wall -g
TARGET   = program
OBJS     = main.o utils.o

$(TARGET): $(OBJS)
	$(CC) $(OBJS) -o $(TARGET)

main.o: main.c utils.h
	$(CC) $(CFLAGS) -c main.c -o main.o

utils.o: utils.c utils.h
	$(CC) $(CFLAGS) -c utils.c -o utils.o

clean:
	rm -f $(TARGET) $(OBJS)
```

**改进点**：如果将来要换编译器（比如换成 `arm-linux-gnueabihf-gcc`），只需要改 `CC` 变量那一行，所有用到的地方自动生效。

#### ③ 引入隐含规则 + 自动变量

```makefile
CC       = gcc
CFLAGS   = -Wall -g
TARGET   = program
OBJS     = main.o utils.o

$(TARGET): $(OBJS)
	$(CC) $^ -o $@        # $^ = 所有依赖, $@ = 目标名

# 模式规则：%.o 匹配所有 .o 文件，依赖同名的 .c 文件
%.o: %.c utils.h
	$(CC) $(CFLAGS) -c $< -o $@   # $< = 第一个依赖（.c 文件）

clean:
	rm -f $(TARGET) $(OBJS)
```

**改进点**：50 个 `.c` 文件也只需要一条 `%.o: %.c` 模式规则，不再需要为每个文件单独写编译规则。

| 自动变量 | 含义 | 示例（`target: a.c b.c`） |
|---------|------|--------------------------|
| `$@` | 当前目标名 | `target` |
| `$<` | 第一个依赖文件 | `a.c` |
| `$^` | 所有依赖文件（去重） | `a.c b.c` |
| `$?` | 所有比目标新的依赖 | 只有修改过的文件 |
| `$*` | 模式匹配的 stem 部分 | 对于 `%.o: %.c`，匹配到 `foo` |

#### ④ 引入条件判断

```makefile
# 根据平台选择不同的编译选项
CC = gcc

ifeq ($(PLATFORM), arm)
    CC = arm-linux-gnueabihf-gcc
    CFLAGS = -march=armv7-a
else
    CFLAGS = -Wall -g
endif

TARGET = program
OBJS   = main.o utils.o

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) $^ -o $@

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@
```

```bash
# 在 PC 上编译：
make

# 交叉编译给 ARM 板子：
make PLATFORM=arm
```

#### ⑤ 引入函数

```makefile
# 自动查找当前目录下所有 .c 文件
SRC_DIR  = .
SRCS     = $(wildcard $(SRC_DIR)/*.c)
OBJS     = $(patsubst %.c, %.o, $(SRCS))

TARGET = program

$(TARGET): $(OBJS)
	gcc $^ -o $@

%.o: %.c
	gcc -c $< -o $@

clean:
	rm -f $(TARGET) $(OBJS)
```

**关键函数说明**：

| 函数 | 格式 | 作用 | 示例 |
|------|------|------|------|
| `wildcard` | `$(wildcard pattern)` | 查找匹配的文件 | `$(wildcard *.c)` 返回 `main.c utils.c` |
| `patsubst` | `$(patsubst pat,rep,text)` | 模式替换 | `$(patsubst %.c,%.o,main.c utils.c)` 返回 `main.o utils.o` |
| `notdir` | `$(notdir names)` | 去掉路径，只留文件名 | `$(notdir src/main.c)` 返回 `main.c` |
| `dir` | `$(dir names)` | 只保留目录部分 | `$(dir src/main.c)` 返回 `src/` |

### 6.5 进阶版本：生产级别的 Makefile 模板

```makefile
# ============================================================
# 通用 Makefile 模板 - 适用于中小型 C 项目
# ============================================================

# 编译器设置
CC       = gcc
CFLAGS   = -Wall -Wextra -g -O2
LDFLAGS  =

# 目标和源文件
TARGET   = program
SRC_DIR  = src
OBJ_DIR  = build
SRCS     = $(wildcard $(SRC_DIR)/*.c)
OBJS     = $(patsubst $(SRC_DIR)/%.c, $(OBJ_DIR)/%.o, $(SRCS))

# 默认目标
all: $(TARGET)

# 链接
$(TARGET): $(OBJS)
	@echo "Linking $@ ..."
	$(CC) $(OBJS) $(LDFLAGS) -o $@
	@echo "Build complete: $@"

# 编译（自动创建 build 目录）
$(OBJ_DIR)/%.o: $(SRC_DIR)/%.c | $(OBJ_DIR)
	@echo "Compiling $< ..."
	$(CC) $(CFLAGS) -c $< -o $@

# 确保 build 目录存在
$(OBJ_DIR):
	mkdir -p $(OBJ_DIR)

# 清理
clean:
	@echo "Cleaning..."
	rm -rf $(OBJ_DIR) $(TARGET)

# 声明伪目标（避免和同名文件冲突）
.PHONY: all clean
```

```bash
# 使用这个模板
mkdir -p src build
echo `int main() { return 0; }` > src/main.c
make              # 编译
make clean        # 清理
make clean all    # 清理后重新编译
```

---

## 7. make 命令常用操作

### 7.1 基本用法

```bash
# 执行默认目标（Makefile 中的第一个目标）
make

# 执行指定目标
make clean
make all

# 指定 Makefile 文件名（默认按顺序查找：GNUmakefile, makefile, Makefile）
make -f mybuild.mk

# 在指定目录执行
make -C /path/to/project
```

### 7.2 常用命令行选项

| 选项 | 作用 | 示例 |
|------|------|------|
| `-f file` | 指定 Makefile 文件名 | `make -f build.mk` |
| `-C dir` | 切换到指定目录执行 | `make -C src/` |
| `-j N` | 并行编译，使用 N 个进程 | `make -j4`（4核并行） |
| `-k` | 出错后继续编译其他文件（不立即停止） | `make -k` |
| `-n` | 只打印要执行的命令，不真正执行（dry-run） | `make -n` |
| `-s` | 静默模式，不打印执行的命令 | `make -s` |
| `-B` | 强制重新编译所有目标 | `make -B` |

### 7.3 调试技巧

```bash
# 查看 make 会执行哪些命令（不实际执行）——调试 Makefile 第一招
make -n
# 输出示例：
# gcc -c main.c -o main.o
# gcc -c utils.c -o utils.o
# gcc main.o utils.o -o program

# 打印 Makefile 中变量的值 —— 调试第二招
make -p | grep "CC "
# 或使用 info 函数在 Makefile 中打印：
#   $(info CC = $(CC))
#   $(info OBJS = $(OBJS))

# 查看隐式规则数据库
make -p | less

# 显示 make 的版本和内置变量
make --version

# 强制重建（忽略时间戳检查，适用于怀疑时间戳有问题时）
make -B

# 详细模式：打印每条规则的决策过程
make --debug=v
```

### 7.4 常见操作场景

```bash
# 场景1：清理后重新编译
make clean && make

# 场景2：多核并行编译（4核）
make -j4

# 场景3：只编译不链接（生成 .o 文件）
make main.o utils.o

# 场景4：查看但不执行
make -n clean

# 场景5：交叉编译（传递变量给 make）
make CC=arm-linux-gnueabihf-gcc
```

---

## 8. 伪目标 .PHONY 详解

### 8.1 什么是伪目标

有些目标并不是要生成的文件，而是代表一个动作（如清理、安装）。这些目标称为"伪目标（phony target）"。

### 8.2 为什么需要 .PHONY

**问题场景**：假设你的 Makefile 中有一个 `clean` 目标用于删除编译产物。如果项目目录下恰好有一个名为 `clean` 的文件，make 会认为 `clean` 目标已经是最新的（因为文件存在且没有过期），从而跳过清理操作。

```bash
# 模拟问题
echo "oops" > clean
make clean
# 输出：make: `clean` is up to date.
# clean 目标没有执行！因为有个叫 clean 的文件存在
```

**解决方案**：用 `.PHONY` 声明伪目标。

```makefile
.PHONY: clean all install

clean:
	rm -f *.o program

all: program

install:
	cp program /usr/local/bin/
```

```bash
# 即使有 clean 文件，也能正常执行
touch clean
make clean
# 输出：rm -f *.o program
# 正确执行了！
```

### 8.3 常见的伪目标

| 伪目标 | 作用 |
|--------|------|
| `all` | 构建所有目标（通常作为第一个目标） |
| `clean` | 删除所有编译产物 |
| `install` | 安装程序到系统目录 |
| `distclean` | 深度清理，包括配置文件 |
| `test` | 运行测试 |
| `help` | 打印帮助信息 |

---

## 9. 初学者常见错误与注意事项

### 9.1 Tab 缩进问题（最常见）

**错误示例**：

```makefile
# 错误：recipe 前面用了空格而不是 Tab
program: main.c
    gcc main.c -o program    # 这行前面是 4 个空格，报错！

# 正确：recipe 前面必须是 Tab 字符
program: main.c
	gcc main.c -o program    # 这行前面是 1 个 Tab 字符
```

**错误信息**：

```
Makefile:2: *** missing separator.  Stop.
```

**遇到这个问题怎么办？**

1. 确保编辑器没有把 Tab 自动转成空格
2. Vim 用户：`set noexpandtab` 确保按 Tab 插入的是 Tab 字符
3. VS Code 用户：底部状态栏选择 "Indent Using Tabs"
4. 查看已写好的 Makefile 中是否是 Tab：`cat -A Makefile`（Tab 显示为 `^I`）

```bash
# 检查 Makefile 中是否有空格代替 Tab 的问题
cat -A Makefile | head -5
# 正确的输出应该类似：
# program: main.c$
# ^Igcc main.c -o program$    ← ^I 表示 Tab，正确
```

### 9.2 文件名大小写问题

```bash
# Linux 文件名区分大小写！
# Makefile, makefile, GNUmakefile 是三个不同的文件

# make 查找 Makefile 的优先级（从高到低）：
# 1. GNUmakefile
# 2. makefile
# 3. Makefile

# 建议统一使用 Makefile（首字母大写），这是最通用的命名
```

### 9.3 依赖关系遗漏

```makefile
# 错误：只依赖 .c 文件，没有包含头文件依赖
main.o: main.c
	gcc -c main.c -o main.o

# 如果修改了 utils.h，main.o 不会被重新编译！

# 正确：同时声明 .c 和 .h 依赖
main.o: main.c utils.h common.h
	gcc -c main.c -o main.o
```

### 9.4 命令前面加 @ 的作用

```makefile
# 默认情况下，make 会打印它执行的每条命令
program: main.o
	gcc main.o -o program
# 执行时输出:
# gcc main.o -o program     ← make 默认会回显命令

# 在命令前面加 @，可以禁止回显
program: main.o
	@echo "Building program..."
	@gcc main.o -o program
# 执行时输出:
# Building program...       ← 只输出 echo 的内容，命令本身不显示
```

### 9.5 常见错误速查表

| 错误信息 | 原因 | 解决办法 |
|---------|------|---------|
| `missing separator` | 命令行前用了空格而非 Tab | 把空格换成 Tab |
| `No rule to make target `xxx`` | 目标不存在且没有规则生成 | 检查文件名是否正确 |
| `Nothing to be done for `xxx`` | 目标已是最新（或伪目标未声明） | 用 `make -B` 强制重建，或确认是否需要 `.PHONY` |
| `make: *** No targets. Stop.` | Makefile 为空或没有 .PHONY 目标且无默认目标 | 检查 Makefile 文件内容 |

---

## 10. 第一个 Makefile 实操

下面是一个完整的、可以直接复制的入门练习。

### 10.1 准备工作

```bash
# 创建项目目录
mkdir ~/makefile_demo
cd ~/makefile_demo

# 创建 main.c
cat > main.c << 'EOF'
#include <stdio.h>
#include "calc.h"

int main() {
    int a = 10, b = 5;
    printf("%d + %d = %d\n", a, b, add(a, b));
    printf("%d - %d = %d\n", a, b, sub(a, b));
    return 0;
}
EOF

# 创建 calc.c
cat > calc.c << 'EOF'
#include "calc.h"

int add(int a, int b) {
    return a + b;
}

int sub(int a, int b) {
    return a - b;
}
EOF

# 创建 calc.h
cat > calc.h << 'EOF'
#ifndef CALC_H
#define CALC_H

int add(int a, int b);
int sub(int a, int b);

#endif
EOF
```

### 10.2 编写 Makefile

```bash
# 创建 Makefile
cat > Makefile << 'EOF'
# 变量定义
CC      = gcc
CFLAGS  = -Wall -g
TARGET  = demo
OBJS    = main.o calc.o

# 默认目标：生成可执行文件
$(TARGET): $(OBJS)
	$(CC) $(OBJS) -o $(TARGET)
	@echo "Build success: ./$(TARGET)"

# 编译 main.o（依赖 main.c 和 calc.h）
main.o: main.c calc.h
	$(CC) $(CFLAGS) -c main.c -o main.o

# 编译 calc.o
calc.o: calc.c calc.h
	$(CC) $(CFLAGS) -c calc.c -o calc.o

# 清理
clean:
	rm -f $(TARGET) $(OBJS)
	@echo "Clean done"

# 声明伪目标
.PHONY: clean
EOF
```

### 10.3 运行和测试

```bash
# 第一步：查看项目文件
ls
# 输出：calc.c  calc.h  main.c  Makefile

# 第二步：编译
make
# 输出：
# gcc -Wall -g -c main.c -o main.o
# gcc -Wall -g -c calc.c -o calc.o
# gcc main.o calc.o -o demo
# Build success: ./demo

# 第三步：运行程序
./demo
# 输出：
# 10 + 5 = 15
# 10 - 5 = 5

# 第四步：验证增量编译
make
# 输出：make: `demo` is up to date.
# 没有任何文件被重新编译！

# 第五步：修改一个文件后重新编译
touch calc.c
make
# 输出：
# gcc -Wall -g -c calc.c -o calc.o
# gcc main.o calc.o -o demo
# Build success: ./demo
# 只重新编译了 calc.o，main.o 跳过了！

# 第六步：清理
make clean
# 输出：
# rm -f demo main.o calc.o
# Clean done
```

### 10.4 实验：验证依赖关系

```bash
# 实验1：修改头文件后会发生什么？
touch calc.h
make
# 输出：main.o 和 calc.o 都会重新编译（因为两者都依赖 calc.h）
# 这就是"头文件依赖"的作用！

# 实验2：删除中间文件后会发生什么？
rm main.o
make
# 输出：只重新编译 main.o，calc.o 不重新编译
# 因为 calc.o 已经是最新的了

# 实验3：使用 -n 查看将要执行的操作
make -n clean
# 输出：rm -f demo main.o calc.o
# -n 选项让你看到 make 会执行什么，但不会真的执行
```

---

## 本讲要点总结

| 问题 | 答案 |
|------|------|
| Makefile是什么？ | 记录依赖关系和规则的文本文件 |
| Make工具做什么？ | 根据Makefile自动增量编译 |
| 为什么必须学？ | 嵌入式没有IDE，体现工程能力 |
| 怎么学？ | 抓住"依赖关系"这个核心 |
| 核心语法是什么？ | 基本语法（目标: 依赖） |
| 其他语法的作用？ | 让Makefile更优雅、简洁、可维护 |

### 扩展要点

| 概念 | 说明 |
|------|------|
| 增量编译原理 | make 通过比较文件时间戳判断是否需要重新编译 |
| 规则三要素 | target（目标）、prerequisites（依赖）、recipe（命令） |
| 变量 | 用 `$(VAR)` 引用，避免重复，提高可维护性 |
| 隐式规则 | 用 `%.o: %.c` 模式规则代替为每个文件单独写规则 |
| .PHONY | 声明伪目标，防止与同名文件冲突 |
| Tab 缩进 | Makefile 命令行前必须用 Tab，不能用空格 |
| 调试技巧 | `make -n` 预览命令，`make -p` 查看变量，`cat -A` 检查 Tab |

---

> **下讲预告**：P32（第31讲-Makefile的主要语法）将进行实际操作演示。

*笔记生成日期：2026-07-22 | 转写：Whisper tiny | 优化日期：2026-07-24*
