---
title: "第31讲：Makefile主要语法"
date: 2026-09-11T09:08:00+08:00
draft: false
description: "在实际的C/C++项目中，可能有几十上百个源文件。每次修改代码后手动输入一长串编译命令，不仅效率低下，而且容易出错。比如："
series: ["Linux 入门"]
series_order: 19
categories: ["技术笔记"]
tags: ["Linux", "Makefile"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P32)
> **标题**：第31讲 — Makefile主要语法
> **时长**：10分54秒
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. Makefile 三要素

### 1.1 背景解释：为什么要用 Makefile？

在实际的C/C++项目中，可能有几十上百个源文件。每次修改代码后手动输入一长串编译命令，不仅效率低下，而且容易出错。比如：

```bash
# 手动编译一个多文件项目（累死人！）
gcc -o app main.c utils.c network.c database.c -I./include -lpthread -lm
```

Makefile 的目的就是：**你只需要输入 `make` 一个命令，它自动帮你做所有事。**

类比说明：Makefile 就像一个"菜谱"：
- 你要做一道菜（目标）
- 需要食材（依赖）
- 按照步骤烹饪（命令）

### 1.2 三要素定义

| 要素 | 说明 | 类比 |
|------|------|------|
| **目标（Target）** | 要生成的文件或要执行的动作 | 做出来的菜 |
| **依赖（Prerequisites）** | 目标所依赖的文件或其他目标 | 做菜需要的食材 |
| **命令（Commands）** | 生成目标需要执行的Shell命令 | 做菜的步骤（切、炒、煮） |

### 1.3 三要素的书写格式

```makefile
目标: 依赖1 依赖2 依赖3 ...
<TAB>命令1
<TAB>命令2
```

**重要规则**：
- 目标和依赖在同一行，用 `:` 分隔
- **命令前必须加 TAB 制表符**（空格不行，这是Makefile的特定语法要求）
- 命令可以有多条
- 每个命令在独立的子Shell中执行（默认行为）

### 1.4 示例

```makefile
all: target_b target_c
	@echo "all"

target_b:
	@echo "target_b"

target_c:
	@echo "target_c"
```

**`@` 符号的作用**：命令前的 `@` 表示**不回显命令本身**，只显示命令的输出。去掉 `@` 的话，你会看到：

```bash
# 不加 @ 的输出：
echo "target_b"
target_b

# 加了 @ 的输出：
target_b
```

### 1.5 实操示例：一个简单的 C 编译例子

```makefile
# 目标: hello (可执行文件)
# 依赖: hello.c (源文件)
# 命令: gcc 编译
hello: hello.c
	gcc -o hello hello.c
```

执行 `make hello` 时，Make 会检查：
- 如果 `hello` 不存在 → 执行 `gcc -o hello hello.c`
- 如果 `hello` 存在且比 `hello.c` 新 → 跳过（已是最新）
- 如果 `hello.c` 被修改过（比 `hello` 新） → 重新编译

---

## 2. Make 工具的工作流程

### 2.1 执行步骤

当在终端输入 `make` 时：

```
① make 读取当前目录的 Makefile 文件
    ↓
② 找到 Makefile 中的第一个目标（默认目标）
    ↓
③ 检查该目标的依赖是否存在 / 是否已更新
    ↓
④ 如果依赖不存在或已过时 → 先执行依赖的命令
    ↓
⑤ 所有依赖执行完毕 → 执行目标自己的命令
```

### 2.2 示例执行过程

对于上面的 Makefile 示例，执行 `make` 的输出：

```
target_b      ← 先执行依赖 target_b 的命令
target_c      ← 再执行依赖 target_c 的命令
all           ← 最后执行目标 all 的命令
```

### 2.3 指定执行特定目标

```bash
make target_b    # 只执行 target_b 目标
make target_c    # 只执行 target_c 目标
```

### 2.4 深度解读：Make 如何判断"是否需要重新编译"？

这是 Make 最核心的智能之处。Make 通过**文件时间戳**来判断：

| 情况 | Make 的判断 | 是否执行命令 |
|------|------------|-------------|
| 目标文件不存在 | 必须生成 | 执行 |
| 目标比所有依赖都新 | 已是最新，无需重新生成 | 不执行 |
| 任意依赖比目标新 | 依赖有更新，需要重新生成 | 执行 |

**为什么这样做？** 这叫做"增量编译"。一个大型项目可能有上千个文件，每次全量编译可能要几分钟甚至几十分钟。Make 只重新编译那些**被修改过的文件及其依赖者**，其他不动的文件直接跳过。

类比说明：就像写论文，你只修改了第三章，不需要重新打印整本论文，只需要重新打印第三章和目录（如果有影响的话）。

### 2.5 实操示例：多文件项目的依赖链

假设项目结构如下：

```bash
main.c       # 包含 main() 函数
utils.c      # 工具函数实现
utils.h      # 工具函数声明
```

对应的 Makefile：

```makefile
app: main.o utils.o
	gcc -o app main.o utils.o

main.o: main.c utils.h
	gcc -c main.c -o main.o

utils.o: utils.c utils.h
	gcc -c utils.c -o utils.o
```

执行 `make` 时的依赖解析过程：

```
app 依赖 main.o 和 utils.o
  ├── main.o 依赖 main.c 和 utils.h
  │    如果 main.c 或 utils.h 被修改 → 重新编译 main.o
  └── utils.o 依赖 utils.c 和 utils.h
       如果 utils.c 或 utils.h 被修改 → 重新编译 utils.o

只有当 main.o 和 utils.o 都准备好后，才链接成 app
```

当只修改了 `main.c`：
- `utils.o` 比 `utils.c` 和 `utils.h` 都新 → **跳过编译 utils.o**
- `main.o` 比 `main.c` 旧 → **重新编译 main.o**
- `app` 比 `main.o` 旧 → **重新链接 app**

---

## 3. 目标文件存在时的问题

### 3.1 问题描述

如果当前目录下存在与目标同名的文件，Make 会认为该目标已经是最新的，**不会执行命令**。

```bash
# 当前目录有 target_b 文件
touch target_b
make target_b
# 输出：make: 'target_b' is up to date.
# 不会执行 echo 命令！
```

### 3.2 原因

Make 判断目标是否需要重建的依据：
- 如果目标文件**不存在** → 执行命令
- 如果目标文件**存在且比依赖新** → 不执行（认为已是最新）

### 3.3 常见错误：命名冲突

初学者容易犯的错误：目标名和项目中的文件名重名。

```makefile
# 错误示例
clean: 
	rm -f *.o app

# 如果当前目录有一个叫 "clean" 的文件，make clean 不会执行！
# 输出：make: 'clean' is up to date.
```

**解决方法**：把这类目标声明为伪目标（见第4章）。

### 3.4 自查清单

| 问题 | 如何排查 |
|------|---------|
| `make` 什么都不输出 | 检查当前目录是否已有与第一个目标同名的文件 |
| `make clean` 不执行 | 检查是否已存在名为 `clean` 的文件 |
| `make XXX` 提示 up to date | 目标文件已经存在，且依赖没有变化 |

解决方法：`rm <文件名>` 删除同名文件后重试，或者直接使用 `.PHONY`。

---

## 4. 伪目标（.PHONY）

### 4.1 作用

声明一个目标是**伪目标**，告诉 Make 它不是一个真正的文件。

```makefile
.PHONY: target_b
```

### 4.2 使用伪目标的效果

添加 `.PHONY` 后，无论是否存在同名文件，每次执行都会运行命令。

```makefile
.PHONY: target_b target_c all

all: target_b target_c
	@echo "all"

target_b:
	@echo "target_b"

target_c:
	@echo "target_c"
```

### 4.3 常用场景

| 场景 | 说明 | 示例 |
|------|------|------|
| `clean` | 清理编译产物，不是文件 | `make clean` 删除所有 .o 和可执行文件 |
| `all` | 默认编译所有目标 | 作为 Makefile 第一个目标 |
| `install` | 安装编译好的程序 | `make install` 复制到系统目录 |
| `distclean` | 彻底清理（包括配置文件） | `make distclean` |
| `test` | 运行测试 | `make test` |
| 非文件目标 | 任何不产生对应文件的目标 | `make help` 显示帮助信息 |

### 4.4 深入理解：.PHONY 的实现原理

当 Make 看到一个 `.PHONY` 声明的目标时，它会**跳过文件时间戳检查**，直接执行该目标的命令。

类比说明：
- 普通目标：你到冰箱找牛奶，如果牛奶还有（文件存在），就不买了。
- `.PHONY` 目标：你妈让你去买牛奶，不管冰箱里有没有，都去买！它不是一个"物品"，它是一个"动作"。

### 4.5 完整示例：标准项目 Makefile 中的常见伪目标

```makefile
.PHONY: all clean install distclean test

# 默认目标：编译整个项目
all: app

# 编译生成可执行文件
app: main.o utils.o
	gcc -o app main.o utils.o

main.o: main.c utils.h
	gcc -c main.c -o main.o

utils.o: utils.c utils.h
	gcc -c utils.c -o utils.o

# 清理编译产物
clean:
	rm -f *.o app

# 安装到系统目录
install:
	cp app /usr/local/bin/

# 彻底清理
distclean: clean
	rm -f config.h Makefile

# 运行测试
test:
	./app --test
```

---

## 5. 完整示例

### 5.1 项目结构

```
Makefile实验/
└── Makefile
```

### 5.2 Makefile 内容

```makefile
.PHONY: all target_b target_c

# 默认目标
all: target_b target_c
	@echo "==== all done ===="

target_b:
	@echo "target_b"

target_c:
	@echo "target_c"
```

### 5.3 执行结果

```bash
$ make
target_b
target_c
==== all done ====

$ make target_b
target_b

$ make target_c
target_c
```

---

## 6. 变量与自动变量

### 6.1 背景解释：为什么要用变量？

在之前的 C 编译示例中，你会发现 `gcc`、`main.o`、`utils.o` 等名字反复出现。如果项目有几十个文件，修改编译器或文件名会非常痛苦。

变量可以把**重复的东西定义一次，到处使用**。

类比说明：变量就像一个"快捷标签"。你把长地址存到 GPS 的"家"里，以后只需要点"家"，不用每次都输入完整的地址。

### 6.2 自定义变量

```makefile
# 定义变量（类似 Shell 变量赋值）
CC      = gcc
CFLAGS  = -Wall -g -O2
OBJS    = main.o utils.o network.o
TARGET  = app

# 使用变量：$(变量名) 或 ${变量名}
$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $(TARGET) $(OBJS)

main.o: main.c utils.h
	$(CC) $(CFLAGS) -c main.c -o main.o

utils.o: utils.c utils.h
	$(CC) $(CFLAGS) -c utils.c -o utils.o
```

### 6.3 变量赋值方式对比

| 赋值符号 | 说明 | 示例 |
|---------|------|------|
| `=` | 递归展开赋值（用到时才展开） | `A = $(B)` 之后改 B，A 也跟着变 |
| `:=` | 立即展开赋值（赋值时就确定值） | `A := $(B)` 之后改 B，A 不变 |
| `?=` | 如果变量未定义才赋值 | `CC ?= gcc` 只在 CC 为空时生效 |
| `+=` | 追加内容 | `CFLAGS += -O2` 在原有值后追加 |

### 6.4 关键区别：`=` vs `:=`

```makefile
# = 递归展开（容易出问题）
VAR1 = $(VAR2)        # VAR1 现在还不知道 VAR2 的值
VAR2 = hello
# 最终 VAR1 = hello   （用到时再取 VAR2 的最终值）

# := 立即展开（更安全）
VAR3 := $(VAR4)       # 此时 VAR4 还没定义，VAR3 = 空
VAR4 = hello
# 最终 VAR3 = 空      （赋值时就确定了）

# 结果：VAR1 = hello，VAR3 = 空
```

**建议**：一般用 `:=` 更安全，避免循环引用和意外行为。

### 6.5 自动变量（核心概念）

这是 Makefile 中最常用的变量，不需要自己定义，Make 自动根据当前规则设置。

| 自动变量 | 含义 | 使用场景 |
|---------|------|---------|
| `$@` | 当前规则的目标文件名 | 命令中使用目标名 |
| `$<` | 第一个依赖文件名 | 编译 .c → .o 时引用源文件 |
| `$^` | 所有依赖文件名（去重） | 链接时列出所有 .o 文件 |
| `$?` | 所有比目标新的依赖文件 | 只处理有更新的依赖 |
| `$*` | 目标文件名去掉后缀的部分 | 模式规则中提取文件名 |

### 6.6 实操示例：用自动变量重写编译规则

**改写前**（手动写文件名，又臭又长）：

```makefile
app: main.o utils.o network.o
	gcc -o app main.o utils.o network.o

main.o: main.c utils.h
	gcc -c main.c -o main.o

utils.o: utils.c utils.h
	gcc -c utils.c -o utils.o

network.o: network.c network.h
	gcc -c network.c -o network.o
```

**改写后**（用自动变量，简洁通用）：

```makefile
CC      := gcc
CFLAGS  := -Wall -g
OBJS    := main.o utils.o network.o
TARGET  := app

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^
	#          $@ = app
	#          $^ = main.o utils.o network.o

main.o: main.c utils.h
	$(CC) $(CFLAGS) -c $< -o $@
	#          $< = main.c（第一个依赖）
	#          $@ = main.o

utils.o: utils.c utils.h
	$(CC) $(CFLAGS) -c $< -o $@

network.o: network.c network.h
	$(CC) $(CFLAGS) -c $< -o $@
```

---

## 7. 模式规则（Pattern Rules）

### 7.1 背景解释：为什么要用模式规则？

上一节的改写中，每个 `.o` 文件都配了一条几乎一模一样的规则，只有文件名不同。如果有 50 个源文件，就要写 50 条规则，这太蠢了。

模式规则就是：**一条规则，匹配所有同类文件**。

### 7.2 语法

```makefile
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@
```

- `%` 是通配符，表示"任意字符串"
- `%.o: %.c` 表示：任何 `.o` 文件都依赖于同名的 `.c` 文件

### 7.3 实操示例：用模式规则大幅简化 Makefile

```makefile
CC      := gcc
CFLAGS  := -Wall -g
OBJS    := main.o utils.o network.o
TARGET  := app

# 链接规则
$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^

# 一条模式规则替代所有 .c → .o 的编译规则
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

# 但是！模式规则没有处理头文件依赖
# 解决方法：手动声明 .o 对 .h 的依赖
main.o: utils.h
network.o: network.h utils.h
```

这条 `%.o: %.c` 规则会自动处理 `main.o`、`utils.o`、`network.o` 和任何未来的 `.o` 文件。

### 7.4 自动生成头文件依赖

手动维护 `.h` 依赖容易遗漏。GCC 可以自动生成依赖信息：

```makefile
# 使用 -MMD -MP 自动生成 .d 依赖文件
CFLAGS  := -Wall -g -MMD -MP

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

# 包含自动生成的依赖文件
DEPS := $(OBJS:.o=.d)
-include $(DEPS)
```

这样每次编译 `.c` 文件时，GCC 会自动生成一个 `.d` 文件，记录该源文件依赖了哪些头文件。下次编译时 Make 会读这些 `.d` 文件，自动知道依赖关系。

---

## 8. 常用函数

### 8.1 背景解释

Makefile 提供内置函数来处理字符串、文件名、条件判断等。函数语法：

```makefile
$(函数名 参数1, 参数2, ...)
# 或
${函数名 参数1, 参数2, ...}
```

### 8.2 文件查找函数 `wildcard`

自动查找匹配通配符的文件：

```makefile
# 获取当前目录下所有 .c 文件
SRCS := $(wildcard *.c)

# 获取 src 目录下所有 .c 文件
SRCS := $(wildcard src/*.c)
```

### 8.3 字符串替换函数 `patsubst`

模式替换字符串：

```makefile
# 把 SRCS 中所有 .c 替换为 .o
OBJS := $(patsubst %.c, %.o, $(SRCS))

# 等价于
OBJS := $(SRCS:.c=.o)
```

### 8.4 常用函数速查表

| 函数 | 语法 | 说明 | 示例 |
|------|------|------|------|
| `wildcard` | `$(wildcard 模式)` | 查找匹配文件 | `$(wildcard *.c)` |
| `patsubst` | `$(patsubst 模式,替换,文本)` | 模式替换 | `$(patsubst %.c,%.o,$(SRCS))` |
| `notdir` | `$(notdir 路径列表)` | 去掉路径，只留文件名 | `$(notdir src/main.c)` → `main.c` |
| `dir` | `$(dir 路径列表)` | 只取目录部分 | `$(dir src/main.c)` → `src/` |
| `basename` | `$(basename 文件名)` | 去掉后缀 | `$(basename main.c)` → `main` |
| `addsuffix` | `$(addsuffix 后缀,文本)` | 加后缀 | `$(addsuffix .o, main utils)` → `main.o utils.o` |
| `addprefix` | `$(addprefix 前缀,文本)` | 加前缀 | `$(addprefix obj/, main.o)` → `obj/main.o` |
| `shell` | `$(shell 命令)` | 执行Shell命令 | `$(shell uname -r)` |
| `foreach` | `$(foreach 变量,列表,操作)` | 遍历列表 | `$(foreach f,$(SRCS),$(f).bak)` |

### 8.5 实操示例：自动发现源文件的 Makefile

不需要手动列出源文件，让 Makefile 自动找到所有 `.c` 文件：

```makefile
CC      := gcc
CFLAGS  := -Wall -g -O2

# 自动查找所有 .c 文件
SRCS    := $(wildcard *.c)
# 生成对应的 .o 文件名
OBJS    := $(SRCS:.c=.o)
# 最终目标
TARGET  := app

.PHONY: all clean

all: $(TARGET)

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) -o $@ $^

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(TARGET)
```

这个 Makefile 放到任何纯 C 项目目录里，不需要修改就能编译所有 `.c` 文件。

---

## 9. 常见错误与排查

### 9.1 用空格代替 TAB

```makefile
# 错误！命令前用了空格
hello: hello.c
    gcc -o hello hello.c
```

**错误信息**：`Makefile:2: *** missing separator.  Stop.`

解决方法：把命令前的空格换成 TAB 键。大多数编辑器（VSCode、Vim）可以设置 Makefile 文件类型自动使用 TAB。

### 9.2 忘记声明 .PHONY

```makefile
# 如果当前目录有名为 clean 的文件，make clean 永远不执行
clean:
	rm -f *.o app
```

解决方法：`.PHONY: clean all`

**经验法则**：所有不生成同名文件的目标，都要声明为 `.PHONY`。

### 9.3 变量引用忘记括号

```makefile
# 错误
CFLAGS  = -Wall
app: main.c
	gcc $CFLAGS -o app main.c   # $C 被解释为变量 C，FLAGS 是普通文本
```

正确写法：`$(CFLAGS)` 或 `${CFLAGS}`

### 9.4 依赖声明不完整

```makefile
# 错误：只依赖 .c，没声明头文件依赖
main.o: main.c
	gcc -c main.c -o main.o

# 如果修改了 utils.h，make 不知道要重新编译 main.o
```

解决方法：手动添加头文件依赖，或使用 GCC 的 `-MMD -MP` 自动生成（见 7.4 节）。

### 9.5 `=` 递归展开导致的死循环

```makefile
VAR1 = $(VAR2)
VAR2 = $(VAR1)
# Make 检测到循环引用会报错，但更隐蔽的循环可能让 Make 卡住
```

解决方法：用 `:=` 代替 `=`。

### 9.6 常见错误速查表

| 错误信息 | 原因 | 解决方法 |
|---------|------|---------|
| `missing separator. Stop.` | 命令前用了空格而非 TAB | 换成 TAB |
| `Nothing to be done for 'xxx'` | 目标已是最新，或依赖未正确声明 | 检查依赖关系，touch 源文件重试 |
| `No rule to make target 'xxx'` | 缺少生成目标的规则 | 添加对应的规则 |
| `*** commands commence before first target` | 命令写在目标行之前了 | 检查 Makefile 开头是否有多余内容 |
| `Circular dependency` | 循环依赖 | 重新设计依赖关系 |

---

## 10. 实战：一个完整的 C 项目 Makefile

### 10.1 项目结构

```
myproject/
├── Makefile
├── src/
│   ├── main.c
│   ├── utils.c
│   └── network.c
├── include/
│   ├── utils.h
│   └── network.h
├── obj/            (编译产物)
└── bin/            (可执行文件)
```

### 10.2 完整 Makefile

```makefile
# ========== 编译器配置 ==========
CC       := gcc
CFLAGS   := -Wall -Wextra -g -O2 -MMD -MP
LDFLAGS  := -lpthread -lm

# ========== 目录 ==========
SRC_DIR  := src
INC_DIR  := include
OBJ_DIR  := obj
BIN_DIR  := bin

# ========== 自动查找源文件 ==========
SRCS     := $(wildcard $(SRC_DIR)/*.c)
OBJS     := $(patsubst $(SRC_DIR)/%.c, $(OBJ_DIR)/%.o, $(SRCS))
DEPS     := $(OBJS:.o=.d)
TARGET   := $(BIN_DIR)/app

# ========== 伪目标 ==========
.PHONY: all clean distclean

# ========== 默认目标 ==========
all: $(TARGET)

# ========== 链接 ==========
$(TARGET): $(OBJS)
	@mkdir -p $(BIN_DIR)
	$(CC) $(CFLAGS) -o $@ $^ $(LDFLAGS)
	@echo "==== Build complete: $(TARGET) ===="

# ========== 编译 ==========
$(OBJ_DIR)/%.o: $(SRC_DIR)/%.c
	@mkdir -p $(OBJ_DIR)
	$(CC) $(CFLAGS) -I$(INC_DIR) -c $< -o $@

# ========== 自动依赖 ==========
-include $(DEPS)

# ========== 清理 ==========
clean:
	rm -rf $(OBJ_DIR) $(BIN_DIR)

distclean: clean
	rm -f *~ *.bak
```

### 10.3 使用方法

```bash
# 编译
make

# 查看 Makefile 中的变量值（调试用）
make -p | grep SRCS

# 清理
make clean

# 并行编译（利用多核 CPU）
make -j4
```

---

## 本讲要点总结

| 概念 | 要点 |
|------|------|
| **三要素** | 目标 : 依赖 → 命令（TAB开头） |
| **Make工作流程** | 读取Makefile → 找默认目标 → 解析依赖 → 执行命令 |
| **文件存在问题** | 目标文件存在时不执行命令 |
| **.PHONY** | 声明伪目标，每次都执行命令 |
| **命令前** | 必须用 TAB，不能用空格 |
| **变量** | 用 `:=` 赋值，`$(VAR)` 引用，减少重复 |
| **自动变量** | `$@`(目标), `$<`(第一个依赖), `$^`(所有依赖) |
| **模式规则** | `%.o: %.c` 一条规则匹配所有同类文件 |
| **函数** | `wildcard` 查文件, `patsubst` 替换, `notdir` 取文件名 |
| **自动依赖** | GCC `-MMD -MP` 自动追踪头文件变化 |

---

*笔记生成日期：2026-07-22 | 转写：Whisper tiny*
*优化日期：2026-07-24 | 补充：变量、自动变量、模式规则、常用函数、常见错误、实战项目*
