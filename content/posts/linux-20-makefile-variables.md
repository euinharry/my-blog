---
title: "第33讲：Makefile的变量"
date: 2026-09-11T09:07:00+08:00
draft: false
description: "默认情况下，make 工具查找当前目录的 Makefile 文件。"
series: ["Linux 入门"]
series_order: 20
categories: ["技术笔记"]
tags: ["Linux", "Makefile"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P34)
> **标题**：第33讲 — Makefile的变量
> **时长**：20分27秒
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. 指定Makefile文件名

### 1.1 基本用法

默认情况下，make 工具查找当前目录的 `Makefile` 文件。

如果使用其他文件名，需要用 `-f` 选项指定：

```bash
make -f MakefileTest
```

### 1.2 为什么需要 `-f` 选项

在实际项目中，有时需要多个 Makefile 共存：

- 不同构建目标用不同的 Makefile（如 `Makefile.debug` 和 `Makefile.release`）
- Makefile 模板/生成文件不能直接命名为 `Makefile`
- 测试不同版本的 Makefile 规则

make 查找 Makefile 的优先级顺序是：`GNUmakefile` > `makefile` > `Makefile`。如果这三个都不存在，就必须用 `-f` 显式指定。

**实操示例**：使用非默认名称的 Makefile 构建项目

```bash
# 创建一个非默认名称的 Makefile
cat > mybuild.mk << 'EOF'
hello:
	@echo "Hello from mybuild.mk"
EOF

# 用 -f 指定它
make -f mybuild.mk hello
# 输出：Hello from mybuild.mk

# 不加 -f 会失败（因为没有 Makefile）
make hello
# 输出：make: *** No rule to make target 'hello'.  Stop.

# 同时也可以指定目标
make -f mybuild.mk clean
```

**常见错误**：新手经常忘记加 `-f`，make 报 "No targets specified and no makefile found"。解决方案就是检查当前目录的文件名，然后用 `-f` 指定。

### 1.3 使用 `-C` 切换到子目录

通常在项目根目录通过 `-C` 切换到子目录执行 Makefile：

```bash
# 切换到 subdir 目录，然后找 Makefile 执行
make -C subdir

# 等价于：
cd subdir && make

# 结合 -f 使用：
make -C subdir -f custom.mk target_name
```

---

## 2. 系统变量（内置变量）

### 2.1 常用系统变量

Makefile 内置了一些系统变量，代表常用的工具。这些变量都有默认值，但可以在 Makefile 或命令行中覆盖。

| 变量 | 含义 | 默认值 |
|------|------|--------|
| `CC` | C编译器 | `cc` |
| `CXX` | C++编译器 | `g++` |
| `AS` | 汇编器 | `as` |
| `AR` | 静态库打包工具 | `ar` |
| `MAKE` | Make工具本身 | `make` |
| `RM` | 删除命令 | `rm -f` |

```makefile
# 测试系统变量
test:
	@echo "CC   = $(CC)"
	@echo "CXX  = $(CXX)"
	@echo "AS   = $(AS)"
	@echo "AR   = $(AR)"
	@echo "MAKE = $(MAKE)"
	@echo "RM   = $(RM)"
```

执行：

```bash
make -f MakefileTest test
# 输出：
# CC   = cc
# CXX  = g++
# AS   = as
# AR   = ar
# MAKE = make
# RM   = rm -f
```

### 2.2 预置的编译选项变量

除了工具变量，Make 还预置了编译选项变量。它们在隐含规则中自动使用，但初始值为空——需要你显式设置才有内容。

| 变量 | 含义 | 传递给 |
|------|------|--------|
| `CFLAGS` | C 编译选项 | `$(CC)` 编译 `.c` 时 |
| `CXXFLAGS` | C++ 编译选项 | `$(CXX)` 编译 `.cpp` 时 |
| `LDFLAGS` | 链接选项（如 `-L` 库路径） | 链接阶段 |
| `LDLIBS` | 链接的库（如 `-lm`） | 链接阶段 |

**背景解释**：为什么 Make 要预先定义这些变量？因为 GNU Make 有一套"隐含规则"。比如当你写 `foo: foo.o` 而没有写规则时，make 会自动调用 `$(CC) $(LDFLAGS) foo.o $(LDLIBS) -o foo`。设置这些变量就可以在不写显式规则的情况下控制构建行为。

**类比说明**：系统变量就像模板中的占位符。模板已经写好了 "把 $(CC) 放在这里"，你只需在上方定义 `CC = arm-linux-gnueabihf-gcc`，整个模板的编译器就切换了。

```makefile
# 使用预置编译选项变量的典型 Makefile 开头
CC      = gcc
CFLAGS  = -Wall -g -O2
LDFLAGS = -L./lib
LDLIBS  = -lm

# 隐含规则自动使用上述变量
# 即使你不写规则，make 也能编译简单的 C 程序
```

### 2.3 命令行覆盖系统变量

系统变量的一个重要特性是可以在命令行传递，覆盖 Makefile 中的值。这是嵌入式开发中切换工具链的常用手段。

```bash
# 临时换成交叉编译器（ARM）
make CC=arm-linux-gnueabihf-gcc

# 开启调试符号
make CFLAGS="-g -O0"

# 多个变量同时覆盖
make CC=clang CFLAGS="-Wall -Wextra"
```

> 命令行传入的变量优先级高于 Makefile 中的赋值。如果你希望 Makefile 中的值不被命令行覆盖，可以使用 `override` 指令（见第6节）。

---

## 3. 自定义变量

Makefile 支持四种赋值方式。理解它们的区别是写好 Makefile 的基础。

### 3.1 延时赋值 `=`（Lazy Set / Recursively Expanded）

**特点**：使用时才计算值，存储的是"表达式"而非"值"。

```makefile
A = 123    # 延时赋值
B = $(A)   # B 此时不取值，只是记住了 "A" 这个引用
A = 456    # 修改 A

test:
	@echo "B = $(B)"   # 输出：B = 456 （取的是 A 的最新值）
```

> `B = $(A)` 只是记录了"A"这个引用，真正取值是在 `$(B)` 被展开的时候。

**更直观的示例**：延时赋值像 Excel 中的公式引用。当 `B1 = A1` 时，改 A1 的值会自动影响 B1。

```makefile
# 深入理解：延时赋值的嵌套行为
BAR = $(FOO)         # BAR 记住了 "FOO" 这个名字
FOO = hello
# 此时 $(BAR) 展开为 "hello"

FOO = world
# 此时 $(BAR) 展开为 "world"   (每次展开都重新取值)

test_nested:
	@echo "FOO = $(FOO)"          # world
	@echo "BAR = $(BAR)"          # world
```

**初学者常见错误**：延时赋值可能引发无限递归！

```makefile
# 错误示例：自己引用自己
VAR = $(VAR) suffix   # 展开时陷入死循环

test:
	@echo $(VAR)
# 执行结果：make 报错
# Makefile:X: *** Recursive variable 'VAR' references itself (eventually).  Stop.
```

> make 有递归检测机制，遇到自我引用会直接终止，不会真的无限循环。

### 3.2 立即赋值 `:=`（Immediate Set / Simply Expanded）

**特点**：定义时立即计算值，后续修改变量来源不影响已赋值的变量。

```makefile
A = 123    # 延时赋值
B := $(A)  # 立即赋值，此时 B = 123
A = 456    # 修改 A，不影响 B

test:
	@echo "B = $(B)"   # 输出：B = 123 （定义时已固定）
```

> `:=` 在定义时就展开并固定值，后续修改不影响。

**类比说明**：`:=` 像摄影，按下快门那一刻就固定了画面；`=` 像实时监控，始终显示当前状态。

**`:=` 的实际应用**：使用 shell 函数的返回值。

```makefile
# 使用 := 获取当前目录（只执行一次）
CUR_DIR := $(shell pwd)

# 如果用 = ，每次引用都会执行 pwd 命令
# BAD_DIR = $(shell pwd)   # 每次展开都执行 shell，浪费性能

show_dir:
	@echo "Current dir: $(CUR_DIR)"
```

```makefile
# 另一个典型场景：获取 git 版本号
GIT_HASH := $(shell git rev-parse --short HEAD 2>/dev/null || echo "unknown")

build:
	@echo "Building version: $(GIT_HASH)"
```

### 3.3 延时 vs 立即：进阶对比

下面的例子展示了延时赋值的一个重要特性：它可以引用**后面**定义的变量。

```makefile
# = 可以引用后面定义的变量
MAIN_DEPS = main.o $(EXTRA_DEPS)   # 允许 EXTRA_DEPS 在后面定义
EXTRA_DEPS = math.o utils.o
# $(MAIN_DEPS) 展开为 "main.o math.o utils.o"

# := 不能引用后面定义的变量（展开时还不存在）
MAIN_DEPS2 := main.o $(EXTRA_DEPS)   # EXTRA_DEPS 此时为空
EXTRA_DEPS = math.o utils.o
# $(MAIN_DEPS2) 展开为 "main.o"（丢失了 math.o utils.o）
```

**何时用 `=`，何时用 `:=`**：

| 场景 | 推荐 | 原因 |
|------|------|------|
| 引用后面定义的变量 | `=` | `:=` 取不到值 |
| 调用 shell 获取值 | `:=` | 避免每次展开都执行 shell |
| 值固定不变 | `:=` | 性能更好，语义清晰 |
| 拼接最终目标列表 | `=` | 可以逐步追加依赖 |
| 一般情况（无特殊需求） | `:=` | 更安全，不会意外被后续赋值影响 |

### 3.4 条件赋值 `?=`（Conditional Set）

**特点**：只有当变量为空时才赋值。如果变量已有值（来自环境变量或之前的赋值），则什么都不做。

```makefile
A ?= 123   # A 为空，赋值成功
A ?= 456   # A 已存在，赋值无效

test:
	@echo "A = $(A)"   # 输出：A = 123
```

**`?=` 的设计意图**：设置"可被覆盖的默认值"。最经典的用途是允许用户在命令行自定义编译器，如果没有指定就用默认的。

```bash
# 用户不指定 CC，Makefile 里 CC ?= gcc 生效 → 用 gcc
make

# 用户指定了 CC，Makefile 里 CC ?= gcc 不生效 → 用 clang
make CC=clang
```

```makefile
# 典型用法：给编译选项设置默认值，但允许外部覆盖
CC      ?= gcc
CFLAGS  ?= -Wall -O2
PREFIX  ?= /usr/local

# 这样用户可以通过以下方式定制：
# make CC=arm-linux-gnueabihf-gcc PREFIX=/opt/myapp
```

> `?=` 只会检查变量是否"未定义或值为空"。如果变量已在环境中设为空字符串 `""`，`?=` 也会认为它"有值"而不覆盖。这是少数需要注意的边界情况。

### 3.5 追加赋值 `+=`（Append）

**特点**：在原有值后面追加，不覆盖。如果变量从未定义过，`+=` 等价于 `=`（延时赋值）。

```makefile
A = 123
A += 456   # 追加

test:
	@echo "A = $(A)"   # 输出：A = 123 456
```

**`+=` 与不同基础赋值的配合**：

```makefile
# 情况1：基础是 =  → 追加后保持延时特性
A = hello
A += world
# A 仍然是延时展开

# 情况2：基础是 := → 追加后保持立即特性
B := hello
B += world
# B 仍然是立即展开

# 情况3：从未定义 → += 视为 =
C += hello   # 等价于 C = hello
```

**实操示例**：逐步构建编译选项。

```makefile
# 构建 CFLAGS：逐步添加选项
CFLAGS  = -Wall
CFLAGS += -g
CFLAGS += -O2
CFLAGS += -I./include
# 最终 CFLAGS = -Wall -g -O2 -I./include

# 构建对象文件列表
OBJS  = main.o
OBJS += sub.o
OBJS += math.o
OBJS += utils.o
# 最终 OBJS = main.o sub.o math.o utils.o

build:
	gcc $(CFLAGS) -o app $(OBJS)
```

### 3.6 四种赋值方式总对比

| 方式 | 运算符 | 赋值时机 | 未定义时行为 | 适用场景 |
|------|--------|---------|-------------|---------|
| 延时赋值 | `=` | 使用时才展开 | 创建新变量 | 需要引用后续定义的变量 |
| 立即赋值 | `:=` | 定义时立即展开 | 创建新变量 | 值固定不变；调用 shell |
| 条件赋值 | `?=` | 仅当变量为空时 | 赋值（变量为空） | 设置可被环境变量覆盖的默认值 |
| 追加赋值 | `+=` | 追加到原有值后 | 等价于 `=` | 逐步构建变量内容 |

---

## 4. 自动变量（Automatic Variables）

自动变量在 Makefile 规则的命令部分自动获取值，不需要手动赋值。它们是 Makefile 中最常用的变量。

### 4.1 核心自动变量

| 变量 | 含义 | 助记 |
|------|------|------|
| `$<` | **第一个**依赖文件 | `<` 像第一个（左尖括号在最左） |
| `$^` | **所有**依赖文件（去重） | `^` 像伞，可以盖住所有 |
| `$@` | **目标**名称 | `@` 像@某人（目标） |
| `$?` | **比目标新的**依赖文件 | `?` 像问号，问"哪些变了？" |
| `$*` | 模式规则中 % 匹配的部分（**主干**） | `*` 通配符，匹配的那部分 |

**为什么需要自动变量？** 如果没有它们，每次写规则都要手动把目标名和依赖名抄一遍。文件一多，抄错一个字母就出问题。自动变量让规则变得"匿名"——不论依赖叫什么，规则都能正确工作。

### 4.2 逐个详解

#### `$<` — 第一个依赖

```makefile
# 编译 .c → .o 时，.c 文件就是第一个依赖
%.o: %.c
	gcc -c $< -o $@
# 等价于：gcc -c foo.c -o foo.o （第一个依赖 = foo.c）

# 多依赖场景：$< 只取第一个
app: main.o lib.o util.o
	gcc -o $@ $^       # 全部依赖
	@echo "First dep: $<"   # 只输出 main.o
```

#### `$^` — 所有依赖（去重）

```makefile
# $^ 包含所有依赖，自动去除重复项
app: main.o math.o main.o   # main.o 出现两次
	gcc -o $@ $^
# 等价于：gcc -o app main.o math.o   (自动去重)
```

> 去重特性很重要：如果因为 include 导致同一个 `.o` 文件在依赖列表中出现两次，链接时不会重复引用。

#### `$@` — 目标名

```makefile
# $@ 在命令行中代表当前规则的目标
app: main.o
	gcc -o $@ $^
# 等价于：gcc -o app main.o

# 多个目标的规则中，$@ 分别代表每个目标
all: target_a target_b target_c
	echo "Building $@..."
# 这不是你想象的行为——make 只为 all 构建依赖，$@ 始终是 all
```

#### `$?` — 比目标新的依赖

```makefile
# 只重新编译"变化了"的文件
libmylib.a: a.o b.o c.o
	ar r $@ $?   # 只更新那些比 libmylib.a 新的 .o 文件
```

**背景解释**：`$?` 用于增量构建。当只有部分 `.o` 文件重新编译后，`ar` 命令只替换那些变化的 `.o`，而不重新打包整个库。在大型项目中这可以显著节省时间。

#### `$*` — 模式匹配的"主干"

```makefile
# 模式规则中 % 匹配的部分
%.o: %.c
	gcc -c $< -o $@
# 当用于 foo.o 时：$* = foo, $@ = foo.o, $< = foo.c
```

### 4.3 扩展自动变量

GNU Make 还支持一些带修饰符的自动变量：

| 变量 | 含义 | 示例（目标为 `dir/sub/foo.o`） |
|------|------|------|
| `$(@D)` | 目标文件的**目录**部分 | `dir/sub` |
| `$(@F)` | 目标文件的**文件名**部分 | `foo.o` |
| `$(<D)` | 第一个依赖的目录部分 | 同上逻辑 |
| `$(<F)` | 第一个依赖的文件名部分 | 同上逻辑 |
| `$(^D)` | 所有依赖的目录列表 | - |
| `$(^F)` | 所有依赖的文件名列表 | - |
| `$%` | 静态模式规则中的目标成员名 | 特定场景 |
| `$+` | 所有依赖（**包含**重复项） | 与 `$^` 的区别是不去重 |

```makefile
# $(@D) 和 $(@F) 的实际应用
# 目标文件在子目录中
build/subdir/app.bin: src/main.c
	mkdir -p $(@D)              # 创建 build/subdir 目录
	gcc -o $@ $<
```

### 4.4 自动变量完整示例

```makefile
# 综合展示所有常用自动变量
CC      = gcc
CFLAGS  = -Wall
TARGET  = app
OBJS    = main.o sub.o math.o

$(TARGET): $(OBJS)
	@echo "目标:        $@"       # app
	@echo "所有依赖:    $^"       # main.o sub.o math.o
	@echo "第一个依赖:  $<"       # main.o
	@echo "变化的依赖:  $?"       # 变化的那些 .o 文件
	@echo "目标目录:    $(@D)"    # .
	@echo "目标文件名:  $(@F)"    # app
	$(CC) -o $@ $^

%.o: %.c
	@echo "编译: $< → $@  (主干: $*)"
	$(CC) $(CFLAGS) -c $< -o $@

.PHONY: clean
clean:
	rm -f $(TARGET) $(OBJS)
```

**记忆口诀**（对应键盘上的符号位置）：

```
$<  →  左尖括号  →  第一个（它在最左边）
$^  →  上尖括号  →  所有（它盖住上面全部）
$@  →  at 符号   →  目标（@某人 = 指定目标）
$?  →  问号      →  哪些变了？
$*  →  星号      →  通配符匹配的主干
```

---

## 5. 用变量优化Makefile

### 5.1 优化前（硬编码，不通用）

```makefile
app3: main.o sub.o
	gcc -o app3 main.o sub.o

main.o: main.c
	gcc -c main.c -o main.o

sub.o: sub.c
	gcc -c sub.c -o sub.o

.PHONY: clean
clean:
	rm -f app3 main.o sub.o
```

**问题诊断**：
- 编译器 `gcc` 写了 3 遍，换交叉编译器需要改 3 处
- 目标 `app3` 写了 4 遍，改名需要改 4 处
- `.o` 文件列表在依赖和 clean 中各写一遍，容易不一致
- 新增 `.c` 文件需要新增一整条规则（3 行代码）

### 5.2 优化第一步：引入变量 + 模式规则

```makefile
CC = gcc
TARGET = app3
OBJS = main.o sub.o

$(TARGET): $(OBJS)
	$(CC) -o $@ $^

%.o: %.c
	$(CC) -c $< -o $@

.PHONY: clean
clean:
	rm -f $(TARGET) $(OBJS)
```

**优化效果**：新增 `.c` 文件只需在 `OBJS` 后面加一行 `OBJS += newfile.o`，不需要写新的编译规则。

### 5.3 优化第二步：添加编译选项和自动发现源文件

```makefile
# 编译器与选项
CC      = gcc
CFLAGS  = -Wall -g -O2
LDFLAGS =
LDLIBS  =

# 目标与源文件
TARGET  = app3
SRCS   := $(wildcard *.c)        # 自动发现所有 .c 文件
OBJS   := $(SRCS:.c=.o)          # 对应替换为 .o 列表

# 链接
$(TARGET): $(OBJS)
	$(CC) $(LDFLAGS) -o $@ $^ $(LDLIBS)

# 编译（模式规则）
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

# 生成依赖关系
depend:
	$(CC) -MM $(SRCS) > .depend

-include .depend

.PHONY: clean
clean:
	rm -f $(TARGET) $(OBJS)
```

**逐步拆解**：

| 步骤 | 代码 | 作用 |
|------|------|------|
| 1 | `SRCS := $(wildcard *.c)` | 用 `wildcard` 函数自动列出当前目录所有 `.c` 文件 |
| 2 | `OBJS := $(SRCS:.c=.o)` | 用替换引用把 `.c` 后缀全部换成 `.o` |
| 3 | `%.o: %.c` | 模式规则，匹配任意 `.o` → `.c` 的编译关系 |
| 4 | `-include .depend` | 自动处理头文件依赖（`-` 表示文件不存在时不报错） |

### 5.4 优化第三步：支持多文件项目（完整模板）

```makefile
# ===== 项目配置 =====
TARGET   = app
BUILD_DIR = build

# ===== 编译器与选项 =====
CC       = gcc
CFLAGS   = -Wall -g -O2 -I./include
LDFLAGS  =
LDLIBS   = -lm

# ===== 源文件自动发现 =====
SRCS    := $(wildcard src/*.c)
OBJS    := $(patsubst src/%.c, $(BUILD_DIR)/%.o, $(SRCS))

# ===== 构建规则 =====
$(TARGET): $(OBJS)
	@echo "Linking $@..."
	$(CC) $(LDFLAGS) -o $@ $^ $(LDLIBS)
	@echo "Build complete: $(TARGET)"

$(BUILD_DIR)/%.o: src/%.c
	@mkdir -p $(@D)
	$(CC) $(CFLAGS) -c $< -o $@

# ===== 工具规则 =====
.PHONY: clean distclean
clean:
	rm -rf $(BUILD_DIR) $(TARGET)

distclean: clean
	rm -f .depend
```

这个模板的特点：
- `wildcard` 自动发现 `src/` 下所有 `.c` 文件
- `patsubst` 将 `src/xxx.c` 转换为 `build/xxx.o`，保持目录结构
- `mkdir -p $(@D)` 确保输出目录存在
- 新增 `.c` 文件后无需修改 Makefile，自动纳入编译

### 5.5 优化效果总结

| 对比维度 | 优化前 | 优化后 |
|------|--------|--------|
| 修改编译器 | 改多处 `gcc` | 改1处 `CC` 变量 |
| 修改目标名 | 改多处 `app3` | 改1处 `TARGET` |
| 增减源文件 | 改多处 `.o` 列表并写新规则 | 改1处 `OBJS`（或自动发现） |
| 新增 `.c` 文件 | 需新增3行规则 | 自动纳入（模式规则） |
| 切换编译选项 | 每处手动改 | 改 `CFLAGS` 一处 |
| 可重用性 | 仅当前项目 | 复制到其他项目也只需改配置区 |

---

## 6. 变量进阶用法

### 6.1 变量引用语法

Makefile 支持两种变量引用写法，功能相同：

```makefile
# 两种写法等价
VAR = hello
result1 = $(VAR)    # 推荐：不容易与 shell 变量混淆
result2 = ${VAR}    # 也可以用

# 嵌套引用需要加 $$ 转义
# 单 $ 会被 make 解释，双 $$ 传递一个 $ 给 shell
show:
	@echo "shell PID: $$(pidof make)"   # $$ 传递 $ 给 shell
```

### 6.2 override 指令：阻止命令行覆盖

命令行传入的变量默认会覆盖 Makefile 中的赋值。如果需要锁定某个值，使用 `override`：

```makefile
# 普通赋值可能被覆盖
CFLAGS = -Wall

# 加上 override 后，命令行无法覆盖
override CFLAGS += -DDEBUG   # 无论命令行传什么，都会追加 -DDEBUG

# 也可以直接覆盖：
override CC = gcc
```

```bash
# 即使命令行指定，override 的值仍会追加
make CFLAGS="-O2"
# 最终 CFLAGS = -O2 -DDEBUG   (override 追加的部分保留了)
```

### 6.3 目标特定变量（Target-specific Variables）

只在特定目标及其依赖的规则中生效的变量。

```makefile
# 全局编译选项
CFLAGS = -Wall -O2

# debug 目标的专属编译选项
debug: CFLAGS = -Wall -g -O0   # 仅在 debug 及其依赖中覆盖
debug: app

# release 目标的专属编译选项
release: CFLAGS = -Wall -O3 -DNDEBUG
release: app

app: main.o sub.o
	gcc $(CFLAGS) -o $@ $^

# make debug  → CFLAGS = -Wall -g -O0
# make release → CFLAGS = -Wall -O3 -DNDEBUG
```

```makefile
# 更复杂的示例：不同子目录用不同的包含路径
CFLAGS = -Wall

# tool/ 下的文件编译时自动添加 -Itool/include
tool/%.o: CFLAGS += -Itool/include

# test/ 下的文件编译时自动添加 -Itest/include
test/%.o: CFLAGS += -Itest/include
```

### 6.4 export：传递变量给子 make

默认情况下，Makefile 中定义的变量不会传递给 `$(MAKE)` 调用的子进程。使用 `export` 来传递。

```makefile
# 导出单个变量
export CC
export CFLAGS = -Wall

# 导出所有变量（谨慎使用）
export

# 不导出（逆向操作）
unexport SECRET_VAR

# 子目录构建时自动继承导出的变量
submodule:
	$(MAKE) -C subdir
```

**背景解释**：为什么默认不传递变量？因为大项目的不同子模块可能需要不同的编译选项。如果全部传递，一个子模块的配置会污染另一个。显式 `export` 让你控制哪些变量共享。

### 6.5 变量的条件赋值与默认值模式

```makefile
# 模式1：用 ?= 设置默认值
PREFIX ?= /usr/local

# 模式2：手动检查 origin 函数
ifeq ($(origin CC), undefined)
    CC = gcc
endif
# origin 函数能区分：未定义 / 环境变量 / Makefile / 命令行 / override

# 模式3：允许命令行覆盖，但 Makefile 提供默认
ifeq ($(origin CC), default)
    CC = gcc   # 当 CC 是默认值（来自内置规则）时覆盖
endif
```

### 6.6 特殊变量

| 变量 | 含义 |
|------|------|
| `.VARIABLES` | 列出所有已定义的变量 |
| `MAKEFLAGS` | 传递给子 make 的标志 |
| `MAKECMDGOALS` | 命令行指定的目标列表 |
| `CURDIR` | 当前目录的绝对路径 |
| `MAKEFILE_LIST` | 已读取的 Makefile 列表 |

```makefile
# 调试：查看所有已定义的变量
list_vars:
	@echo $(.VARIABLES)

# 查看当前目标
debug_goals:
	@echo "Goals: $(MAKECMDGOALS)"
```

---

## 7. 常见错误与调试技巧

### 7.1 错误1：递归变量自引用

```makefile
# 错误写法
VAR = $(VAR) suffix    # 展开时死循环

# 正确写法（用 := 或 +=）
VAR := suffix          # 直接赋值
VAR += more            # 追加
```

### 7.2 错误2：变量值末尾的空格

```makefile
# 注意：等号后面的空格会被保留！
DIR = /usr/local/      # 末尾无空格 ✓
DIR = /usr/local/      # 末尾有空格（看不见但存在）⚠

# 在依赖路径中会出错
# $(DIR)/bin  →  /usr/local/ /bin  （多了一个空格）
```

### 7.3 错误3：命令中的变量与 shell 变量混淆

```makefile
# Makefile 变量用 $() 展开
NAME = world
hello:
	echo $(NAME)           # make 展开 → echo world ✓

# shell 变量在 Makefile 中需要 $$ 转义
list:
	for f in *.c; do \
		echo "file: $$f"; \    # $$f 传递给 shell 变成 $f
	done

# 错误示例：单 $ 被 make 当成自己的变量
list_bad:
	for f in *.c; do echo $f; done   # $f 被 make 解释，出错
```

### 7.4 调试技巧

```makefile
# 技巧1：打印变量值
debug:
	@echo "CC      = [$(CC)]"
	@echo "CFLAGS  = [$(CFLAGS)]"
	@echo "OBJS    = [$(OBJS)]"

# 技巧2：查看变量来源（origin 函数）
debug_origin:
	@echo "CC origin: $(origin CC)"
	# 输出可能是：default / environment / file / command line / override / automatic / undefined

# 技巧3：查看变量内容（flavor 函数）
debug_flavor:
	@echo "CFLAGS flavor: $(flavor CFLAGS)"
	# 输出：simple (= 立即) 或 recursive (= 延时)

# 技巧4：make -p 打印所有规则和变量
# 在命令行执行：
# make -p -f /dev/null    → 查看内置规则和变量
# make -p                 → 查看当前 Makefile 的所有规则和变量

# 技巧5：make -n 只打印命令不执行（dry run）
# make -n    → 预览将要执行的命令
```

---

## 8. 本讲总结

### 8.1 变量类型速查

| 类型 | 语法 | 说明 |
|------|------|------|
| 系统变量 | `$(CC)`, `$(AS)`, `$(MAKE)`, `$(CXX)` | 内置的默认工具 |
| 延时赋值 | `=` | 使用时才展开 |
| 立即赋值 | `:=` | 定义时立即展开 |
| 条件赋值 | `?=` | 仅空时赋值 |
| 追加赋值 | `+=` | 追加不覆盖 |
| 自动变量 | `$<`, `$^`, `$@`, `$?`, `$*` | 规则中自动取值 |

### 8.2 选择指南

| 你想做什么 | 用哪个 |
|------------|--------|
| 设置允许命令行覆盖的默认值 | `?=` |
| 调用 shell 获取动态值 | `:=` |
| 引用后面才定义的变量 | `=` |
| 逐步拼接编译选项 | `+=` |
| 写编译规则，不写死文件名 | `$<`, `$@`, `$^` |
| 锁定变量不让命令行覆盖 | `override` |
| 不同构建目标用不同编译选项 | 目标特定变量 |

### 8.3 最佳实践

1. **编译器/选项用变量**：`CC`, `CFLAGS` 等，方便切换工具链
2. **文件和目录用变量**：`TARGET`, `OBJS`, `SRCS` 集中管理
3. **用 `:=` 作为默认赋值方式**：除非需要延时展开
4. **用 `?=` 给所有变量设默认值**：允许环境变量覆盖
5. **规则里用自动变量**：`$<`, `$@`, `$^` 让规则通用
6. **善用函数**：`wildcard`, `patsubst`, `shell` 减少手动维护

---

*笔记生成日期：2026-07-22 | 转写：Whisper tiny*
*优化日期：2026-07-24 | 补充：变量进阶用法、常见错误、调试技巧、完整示例*
