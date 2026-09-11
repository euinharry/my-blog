---
title: "第36讲：Makefile的常用函数"
date: 2026-09-11T09:06:00+08:00
draft: false
description: "Make 工具有非常丰富的内置函数，可以在官方文档中查阅："
series: ["Linux 入门"]
series_order: 21
categories: ["技术笔记"]
tags: ["Linux", "Makefile"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P37)
> **标题**：第36讲 --- Makefile的常用函数
> **时长**：约25分钟
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. 背景介绍

### 1.1 官方文档

Make 工具有非常丰富的内置函数，可以在官方文档中查阅：

| 文档 | 版本 | 说明 |
|------|------|------|
| GNU Make 官方手册 | 4.3 | 最新英文版 |
| GNU Make 中文手册 | 3.81 | 适合英文阅读困难者（章节编号一致） |

**第八章**专门介绍 Makefile 的函数。

### 1.2 本讲重点

从众多函数中挑选**使用频率最高的4个**，熟练掌握即可应对绝大部分项目需求。

### 1.3 函数调用的通用语法

所有 Makefile 内置函数的调用形式都是统一的：

```makefile
$(函数名 参数1, 参数2, 参数3, ...)
# 或者用花括号
${函数名 参数1, 参数2, 参数3, ...}
```

**关键规则**：

- 函数名与第一个参数之间用**空格**分隔
- 参数之间用**逗号**分隔
- 参数可以包含空格（函数会按逗号拆分，不会按空格拆分）

```makefile
# 正确：三个参数
$(patsubst %.c, %.o, foo.c bar.c)

# 错误：逗号位置不对
$(patsubst,%.c,%.o,foo.c bar.c)   # 多了逗号

# 正确：参数本身可以含空格
$(patsubst %.c, %.o, foo.c bar.c baz.c)
#                    ^^^^^^^^^^^^^^^^ 这是一个参数（包含空格）
```

### 1.4 为什么需要 Makefile 函数

在大型项目中，手动列举所有源文件既不现实也容易出错。Makefile 函数解决了几个核心问题：

- **自动化**：不需要每次新增源文件都手动修改 Makefile
- **可维护性**：目录结构调整后，只需修改变量即可
- **可移植性**：一套规则适用于不同模块、不同项目

**类比说明**：如果把 Makefile 比作一个工厂的流水线指令书，那函数就是这条流水线上的自动化机器人。没有函数，你需要手动告诉每个工人该搬哪个箱子；有了函数，机器人自动扫描仓库、找到所有箱子、搬运到指定位置。

---

## 2. 四个核心函数

### 2.1 `$(patsubst pattern,replacement,text)` --- 模式替换函数

**作用**：将 `text` 中匹配 `pattern` 的单词替换为 `replacement`。

**基本示例**：

```makefile
# 将 foo.c bar.c 替换为 foo.o bar.o
OBJS = $(patsubst %.c, %.o, foo.c bar.c)
# OBJS = foo.o bar.o
```

**参数说明**：

| 参数 | 说明 |
|------|------|
| `pattern` | 匹配模式（含通配符 `%`） |
| `replacement` | 替换模式（`%` 匹配相同部分） |
| `text` | 由若干单词组成的文本 |

**验证**：

```makefile
test:
	@echo $(patsubst %.c, %.o, foo.c bar.c)
# 输出: foo.o bar.o
```

#### patsubst 深入理解

**`%` 的工作原理**：`%` 在 `pattern` 中匹配任意字符串（包括空字符串），同一个 `%` 在 `replacement` 中会被替换为匹配到的那段字符串。

```makefile
# % 匹配了 "main"，然后用 "main" 替换 replacement 中的 %
$(patsubst src/%.c, build/%.o, src/main.c)
# 结果: build/main.o

# 匹配多个文件
$(patsubst src/%.c, build/%.o, src/main.c src/init.c src/util.c)
# 结果: build/main.o build/init.o build/util.o
```

**`%` 在 pattern 中的位置**：`%` 可以出现在任意位置，不限于开头或结尾。

```makefile
# 给文件名添加前缀
$(patsubst %, lib%, math.c string.c)
# 结果: libmath.c libstring.c

# 替换目录中间的部分
$(patsubst app/v%/src, lib/v%/obj, app/v1/src app/v2/src)
# 结果: lib/v1/obj lib/v2/obj
```

#### patsubst 与 subst 的区别

Make 还提供了 `$(subst from,to,text)` 函数，做的是**字面量替换**而非模式匹配：

```makefile
# subst：精确字符串替换（不支持 % 通配符）
$(subst .c,.o, foo.c bar.c)
# 结果: foo.o bar.o

# patsubst：模式匹配替换（支持 % 通配符）
$(patsubst %.c, %.o, foo.c bar.c)
# 结果: foo.o bar.o
```

| 对比项 | `subst` | `patsubst` |
|--------|---------|------------|
| 匹配方式 | 精确字符串匹配 | 通配符 `%` 模式匹配 |
| 适用场景 | 简单替换（如路径分隔符） | 后缀替换、路径重组 |
| 性能 | 稍快 | 略慢（需模式解析） |
| 典型用法 | `$(subst /,\,path)` | `$(patsubst %.c,%.o,$(SRCS))` |

```makefile
# 典型用例对比

# 场景一：统一路径分隔符（Windows ↔ Linux）
WIN_PATH = $(subst /,\,src/module/file.c)    # src\module\file.c

# 场景二：替换文件后缀
OBJS = $(patsubst %.c, %.o, $(SRCS))          # 只能用 patsubst
```

**初学易错点**：`subst` 不支持 `%`，如果你写了 `$(subst %.c, %.o, foo.c)`，会直接去匹配字面量 `%.c`，结果是找不到任何匹配，原样返回。

---

### 2.2 `$(notdir names...)` --- 取文件名函数

**作用**：去掉路径中的目录部分，只保留文件名。

**基本示例**：

```makefile
NAME = $(notdir src/main.c)
# NAME = main.c
```

**演示**：

```makefile
test:
	@echo $(notdir src/main.c src/sub.c)
# 输出: main.c sub.c
```

常用于将带路径的源码名转换为纯文件名。

#### notdir 深入理解

**notdir 只去掉目录部分，不修改文件名本身**：

```makefile
# 多层嵌套路径
$(notdir a/b/c/d/file.txt)       # file.txt

# 多个路径同时处理
$(notdir src/main.c lib/util.c include/config.h)
# 结果: main.c util.c config.h

# 如果已经是纯文件名，不做任何改变
$(notdir main.c)                  # main.c
```

**notdir 在实际项目中的典型用法**：

```makefile
# 场景：源码在子目录中，但 .o 文件要统一放到 build/ 目录
SRCS = $(wildcard src/*.c lib/*.c)
# SRCS = src/main.c src/init.c lib/util.c

# 去掉路径，只剩文件名
OBJ_NAMES = $(notdir $(SRCS))
# OBJ_NAMES = main.c init.c util.c

# 再替换后缀并加目录前缀
OBJS = $(patsubst %.c, build/%.o, $(OBJ_NAMES))
# OBJS = build/main.o build/init.o build/util.o
```

#### notdir 与其他路径函数的对比

Make 提供了一组处理路径和文件名的函数，各司其职：

| 函数 | 输入 | 输出 | 说明 |
|------|------|------|------|
| `$(notdir ...)` | `src/main.c` | `main.c` | 取文件名，去掉所有目录 |
| `$(dir ...)` | `src/main.c` | `src/` | 取目录部分（保留末尾 `/`） |
| `$(basename ...)` | `src/main.c` | `src/main` | 去掉后缀 |
| `$(suffix ...)` | `src/main.c` | `.c` | 取后缀 |

```makefile
# 实际操作示例
PATH = src/app/main.c

$(notdir $(PATH))     # main.c       ← 只要文件名
$(dir $(PATH))        # src/app/     ← 只要目录（注意末尾的 /）
$(basename $(PATH))   # src/app/main ← 去掉后缀
$(suffix $(PATH))     # .c           ← 只要后缀
```

---

### 2.3 `$(wildcard pattern)` --- 通配符函数

**作用**：展开通配符，获取当前目录下所有匹配的文件名列表。

**基本示例**：

```makefile
SRC_FILES = $(wildcard *.c)
# 假设当前目录有 main.c sub.c，则 SRC_FILES = main.c sub.c
```

**使用场景**：

- 自动扫描目录下的所有源文件
- 无需手动列举每个文件

#### wildcard 深入理解

**wildcard 与 shell 通配符的关键区别**：

在 Makefile 中，直接写 `*.c` 不会自动展开（和 shell 不同），必须用 `$(wildcard *.c)` 才能获取文件列表。

```makefile
# 错误写法：Makefile 中直接写通配符不会展开
SRCS = *.c          # SRCS 的值是字面量 "*.c"，不是文件列表！

# 正确写法：必须用 wildcard 函数
SRCS = $(wildcard *.c)   # SRCS = main.c util.c ...

# 为什么？因为 Make 处理变量赋值时不会展开通配符
# 通配符展开是 shell 的特性，make 需要显式调用 wildcard 函数
```

**通配符支持的语法**：

```makefile
# * 匹配任意字符串（不含路径分隔符）
$(wildcard *.c)           # main.c util.c

# ? 匹配单个字符
$(wildcard file?.txt)     # file1.txt fileA.txt（不匹配 file10.txt）

# [字符集] 匹配指定范围内的字符
$(wildcard [a-z]*.c)      # 以小写字母开头的 .c 文件

# 子目录中的通配符
$(wildcard src/*.c)       # src/main.c src/util.c
$(wildcard */*.c)         # module1/mid.c module2/main.c（一级子目录）
```

**wildcard 的一个经典陷阱：延迟展开**：

```makefile
# 陷阱：在规则中重复调用 wildcard
FILES = $(wildcard *.c)    # 此时扫描目录

all:
	@echo $(FILES)

# 问题：如果在 make 运行期间新增了 .c 文件，FILES 不会更新
# 因为 wildcard 在变量赋值时执行，而不是在规则执行时

# 解决方法：在规则内部直接调用 wildcard
all:
	@echo $(wildcard *.c)   # 每次执行规则时重新扫描
```

#### 递归扫描多个子目录

```makefile
# 一级子目录
SRCS = $(wildcard */*.c)
# module1/mid.c module2/main.c

# 两级子目录
SRCS = $(wildcard */*/*.c)
# module1/sub/file.c

# 配合 foreach 递归多级目录（见 2.4 节）
```

---

### 2.4 `$(foreach var,list,text)` --- 循环函数

**作用**：遍历 `list` 中的每个单词，赋值给 `var`，执行 `text`，将所有结果拼接。

**语法**：

```makefile
$(foreach var, list, text)
```

**基本示例**：

```makefile
DIRS = module1 module2
SRCS = $(foreach dir, $(DIRS), $(wildcard $(dir)/*.c))
# 等价于：wildcard module1/*.c + wildcard module2/*.c
# 返回两个目录下所有 .c 文件
```

**执行流程**：

```
第1次遍历：var=module1 → 执行 wildcard module1/*.c
第2次遍历：var=module2 → 执行 wildcard module2/*.c
结果拼接：.../module1/*.c .../module2/*.c
```

**常用组合**：`foreach` + `wildcard` → 遍历多个子目录收集源文件。

#### foreach 深入理解

**foreach 的返回值拼接方式**：

```makefile
# 每次迭代的结果用空格拼接
$(foreach n, 1 2 3, $(n).txt)
# 结果: 1.txt 2.txt 3.txt
```

**foreach 与 shell 循环的类比**：

```bash
# shell 中的 for 循环（bash）
for dir in module1 module2; do
    ls $dir/*.c
done

# Makefile 中的 foreach（功能类似，但写法不同）
$(foreach dir, module1 module2, $(wildcard $(dir)/*.c))
```

**类比说明**：可以把 `foreach` 理解为工厂里的一条传送带。`list` 是传送带上的物品（单词），`var` 是工人的手（每次拿起一件物品），`text` 是工人对物品做的操作。所有处理完的物品按顺序堆放在一起，就是最终结果。

**嵌套 foreach**：

```makefile
# 二维遍历：模块 × 文件类型
MODULES = core net ui
TYPES = .c .h

FILES = $(foreach m, $(MODULES), \
           $(foreach t, $(TYPES), $(m)/*$(t)))
# 结果: core/*.c core/*.h net/*.c net/*.h ui/*.c ui/*.h
```

**foreach 的常见用法模式**：

```makefile
# 模式1：遍历目录列表收集文件
SRCS = $(foreach dir, $(SRC_DIRS), $(wildcard $(dir)/*.c))

# 模式2：为每个模块生成编译规则
RULES = $(foreach mod, $(MODULES), $(mod).rule)

# 模式3：批量添加前缀
INCLUDES = $(foreach dir, $(INC_DIRS), -I$(dir))
# INCLUDES = -Iinclude -Isrc -Ilib
# 注意：这里 -I 和 $(dir) 之间没有空格，这是 gcc 能接受的格式
# gcc 同时支持 -I dir 和 -Idir 两种写法
```

**`-I` 前缀的细节说明**：

```makefile
INC_DIRS = include src lib

# 方式A：addprefix 函数（推荐）
CFLAGS += $(addprefix -I, $(INC_DIRS))
# 结果: -Iinclude -Isrc -Ilib

# 方式B：foreach 手动拼接
CFLAGS += $(foreach d, $(INC_DIRS), -I$(d))
# 结果: -Iinclude -Isrc -Ilib

# 两种方式结果相同。gcc 接受 -Iinclude（无空格）和 -I include（有空格）
```

#### 常见错误：foreach 中变量展开时机

```makefile
# 错误示例：试图在 foreach 中修改变量
# 不要这样写！foreach 中的变量展开在 Make 解析阶段，不是运行时

# 正确：foreach 只用于生成文本，不要用于"累加"操作
NAMES = $(foreach n, 1 2 3, file_$(n))
# NAMES = file_1 file_2 file_3
```

---

## 3. 更多常用函数

掌握了上面四个核心函数后，下面这些函数能在特定场景下大幅简化 Makefile 的编写。

### 3.1 文本处理类函数

#### `$(filter pattern..., text)` --- 过滤保留

只保留 `text` 中匹配 `pattern` 的单词。

```makefile
# 只保留 .c 文件
SRCS = $(filter %.c, main.c util.h config.c readme.txt)
# SRCS = main.c config.c

# 只保留特定目录下的文件
SRC_FILES = $(filter src/%, src/main.c lib/util.c src/init.c)
# SRC_FILES = src/main.c src/init.c
```

#### `$(filter-out pattern..., text)` --- 过滤排除

排除匹配 `pattern` 的单词，保留其余部分。

```makefile
# 排除 .o 文件，获取其他文件列表
NON_OBJ = $(filter-out %.o, main.o util.c data.txt app.o)
# NON_OBJ = util.c data.txt

# 实用场景：从所有文件中排除不需要编译的文件
ALL_SRCS = $(wildcard *.c)
EXCLUDE = test_main.c skip_this.c
SRCS = $(filter-out $(EXCLUDE), $(ALL_SRCS))
```

#### `$(sort list)` --- 排序并去重

```makefile
# 排序 + 去重
$(sort c b a b c)
# 结果: a b c

# 实用场景：去重头文件搜索路径
INC_DIRS = $(sort include lib include src lib)
# INC_DIRS = include lib src
# 去掉了重复的 include 和 lib，按字母排序
```

#### `$(word n,text)` --- 取第 n 个单词

```makefile
$(word 2, apple banana cherry)
# 结果: banana

$(word 1, apple banana cherry)
# 结果: apple

# 索引超出范围返回空字符串（不是错误）
$(word 10, a b c)
# 结果: （空）
```

#### `$(wordlist s,e,text)` --- 取单词范围

```makefile
$(wordlist 2, 4, a b c d e f)
# 结果: b c d

# 起点大于终点 → 返回空
$(wordlist 4, 2, a b c d e f)
# 结果: （空）

# 对应 Python 中的 text.split()[s-1:e]
```

#### `$(words text)` / `$(firstword text)` / `$(lastword text)`

```makefile
TEXT = apple banana cherry date

$(words $(TEXT))       # 4（单词数量）
$(firstword $(TEXT))   # apple
$(lastword $(TEXT))    # date
```

### 3.2 文件名处理类函数

#### `$(dir names...)` --- 取目录部分

```makefile
$(dir src/main.c)
# 结果: src/

$(dir src/foo/bar.txt lib/util.h)
# 结果: src/foo/ lib/

# 注意：返回值末尾有 /
```

#### `$(basename names...)` / `$(suffix names...)`

```makefile
# 去掉后缀
$(basename src/main.c)
# 结果: src/main

# 取后缀
$(suffix src/main.c)
# 结果: .c

# 多个文件
$(basename a.tar.gz b.txt)
# 结果: a.tar b

$(suffix a.tar.gz b.txt)
# 结果: .gz .txt
# 注意：suffix 只取最后一个 . 之后的部分，不是 .tar.gz
```

#### `$(addsuffix suffix, names...)` / `$(addprefix prefix, names...)`

```makefile
# 添加后缀
$(addsuffix .o, main util)
# 结果: main.o util.o

# 添加前缀
$(addprefix build/, main.o util.o)
# 结果: build/main.o build/util.o

# 与 patsubst 配合使用的典型模式
OBJS = $(addprefix $(BUILD_DIR)/, $(patsubst %.c, %.o, $(notdir $(SRCS))))
# 先替换后缀 .c→.o，再加 build/ 前缀
```

#### `$(join list1, list2)` --- 逐项拼接

```makefile
$(join a b c, .c .h .o)
# 结果: a.c b.h c.o

# 两个列表长度不同时，多出的部分原样保留
$(join a b, 1 2 3 4)
# 结果: a1 b2 3 4
```

### 3.3 Shell 交互与流程控制

#### `$(shell command)` --- 执行 Shell 命令

```makefile
# 获取当前日期
BUILD_DATE = $(shell date +%Y-%m-%d)

# 获取 git 版本号
GIT_VER = $(shell git describe --tags --always 2>/dev/null || echo "unknown")

# 自动检测操作系统
UNAME_S = $(shell uname -s)
ifeq ($(UNAME_S), Linux)
    PLATFORM = linux
else ifeq ($(UNAME_S), Darwin)
    PLATFORM = macos
endif
```

**注意事项**：`$(shell ...)` 在 Makefile 解析阶段执行，不是在规则执行阶段。如果把 `$(shell ...)` 放在变量定义中，它只执行一次（在 make 启动时）。

#### `$(error text...)` / `$(warning text...)` / `$(info text...)`

```makefile
# info：打印信息（不中断）
$(info === 开始编译，目标平台: $(PLATFORM) ===)

# warning：打印警告（不中断）
ifndef CC
    $(warning CC 未定义，使用默认编译器 gcc)
    CC = gcc
endif

# error：打印错误并中断 make 执行
ifndef CROSS_COMPILE
    $(error 交叉编译前缀 CROSS_COMPILE 未定义！)
endif
```

#### `$(call variable, param1, param2, ...)` --- 调用自定义函数

Makefile 没有真正的"函数"关键字，但可以通过 `define` + `call` 模拟：

```makefile
# 定义一个"函数"：将源文件编译为目标文件
define compile_module
    $(CC) $(CFLAGS) -c $(1) -o $(2)
endef

# 调用该"函数"
module.o: module.c
	$(call compile_module, $<, $@)
# 展开为: gcc -Wall -c module.c -o module.o
```

在 `call` 内部，`$(1)`, `$(2)`, `$(3)` ... 分别对应第1、2、3个参数。

```makefile
# 更实用的例子：为每个模块生成编译规则
define MODULE_TEMPLATE
$(1)_OBJS = $$(patsubst %.c, build/%.o, $$(notdir $$(wildcard $(1)/*.c)))
$(1)_target: $$($(1)_OBJS)
	$$(CC) -o $$@ $$^
endef

MODULES = core net ui
$(foreach m, $(MODULES), $(eval $(call MODULE_TEMPLATE, $(m))))
```

**注意**：`define` 体内使用 `$$` 代替 `$`，是因为 `call` 在展开时会处理一次 `$`，如果不加双写，内部的变量引用会被提前展开。

#### `$(eval text)` --- 动态生成 Makefile 规则

`eval` 可以将一段文本作为 Makefile 语法解析执行，常用于批量生成规则：

```makefile
# 动态为每个 .c 文件生成编译规则
SRCS = main.c util.c
define GEN_RULE
$(patsubst %.c, %.o, $(1)): $(1)
	$(CC) -c $$< -o $$@
endef

$(foreach src, $(SRCS), $(eval $(call GEN_RULE, $(src))))

# 展开后等价于：
# main.o: main.c
# 	$(CC) -c $< -o $@
# util.o: util.c
# 	$(CC) -c $< -o $@
```

### 3.4 函数速查表

#### 按类别分组

**文本/模式匹配**：

| 函数 | 语法 | 作用 | 示例 |
|------|------|------|------|
| `subst` | `$(subst from,to,text)` | 字面量替换 | `$(subst .c,.o,main.c)` |
| `patsubst` | `$(patsubst pat,rep,text)` | 模式替换 | `$(patsubst %.c,%.o,$(SRCS))` |
| `filter` | `$(filter pat...,text)` | 保留匹配项 | `$(filter %.c,$(FILES))` |
| `filter-out` | `$(filter-out pat...,text)` | 排除匹配项 | `$(filter-out %.o,$(FILES))` |
| `sort` | `$(sort list)` | 排序去重 | `$(sort $(INC_DIRS))` |
| `strip` | `$(strip string)` | 去除首尾空格，合并中间空格 | `$(strip  a   b  )` → `a b` |

**单词操作**：

| 函数 | 语法 | 作用 |
|------|------|------|
| `word` | `$(word n,text)` | 取第 n 个单词 |
| `wordlist` | `$(wordlist s,e,text)` | 取第 s 到 e 个单词 |
| `words` | `$(words text)` | 统计单词数量 |
| `firstword` | `$(firstword text)` | 取第 1 个单词 |
| `lastword` | `$(lastword text)` | 取最后 1 个单词 |

**文件名/路径**：

| 函数 | 语法 | 作用 |
|------|------|------|
| `notdir` | `$(notdir names...)` | 取文件名，去掉路径 |
| `dir` | `$(dir names...)` | 取目录部分 |
| `basename` | `$(basename names...)` | 去掉后缀 |
| `suffix` | `$(suffix names...)` | 取后缀 |
| `addsuffix` | `$(addsuffix suf,names...)` | 添加后缀 |
| `addprefix` | `$(addprefix pre,names...)` | 添加前缀 |
| `join` | `$(join list1,list2)` | 逐项拼接 |
| `wildcard` | `$(wildcard pattern)` | 展开通配符 |
| `realpath` | `$(realpath names...)` | 解析为绝对路径 |
| `abspath` | `$(abspath names...)` | 转换为绝对路径（不解析符号链接） |

**流程控制**：

| 函数 | 语法 | 作用 |
|------|------|------|
| `foreach` | `$(foreach var,list,text)` | 循环遍历 |
| `call` | `$(call var,param,...)` | 调用自定义函数 |
| `eval` | `$(eval text)` | 动态生成 Makefile 语句 |
| `shell` | `$(shell command)` | 执行 shell 命令 |
| `error` | `$(error text...)` | 中断并报错 |
| `warning` | `$(warning text...)` | 打印警告 |
| `info` | `$(info text...)` | 打印信息 |

---

## 4. 实战：用函数优化项目 Makefile

### 4.1 项目目录结构（优化后）

```
app3/
├── Makefile
├── module1/
│   └── mid.c
├── module2/
│   └── main.c
└── build/          ← 自动生成，存放编译产物
    ├── app3        ← 可执行文件
    ├── mid.o
    └── main.o
```

### 4.2 优化目标

| 目标 | 实现方式 |
|------|---------|
| 源码分模块存放 | 每个模块一个子目录 |
| 自动扫描源文件 | `foreach` + `wildcard` |
| 路径 → 文件名转换 | `notdir` |
| 后缀替换 `.c` → `.o` | `patsubst` |
| 输出目录隔离 | `BUILD_DIR` 变量 |
| 自动创建输出目录 | 伪目标 `create_build` |

### 4.3 最终 Makefile

**注意**：以下 Makefile 在原始笔记的基础上补充了 `VPATH` 声明，这是实际使用中**必需**的。原始笔记中缺少这行，导致 make 无法在子目录中找到 `.c` 源文件。

```makefile
CC = gcc
TARGET = app3

# 源码目录列表
SRC_DIR = module1 module2

# 构建输出目录
BUILD_DIR = build

# ★ 关键：告诉 make 在哪些目录中搜索源文件
# 没有这行，模式规则中的 %.c 无法在子目录中找到源文件
VPATH = $(SRC_DIR)

# 遍历所有源码目录，收集所有 .c 文件
SRCS = $(foreach dir, $(SRC_DIR), $(wildcard $(dir)/*.c))

# 去除路径，只保留文件名，再替换 .c → .o
OBJS = $(patsubst %.c, $(BUILD_DIR)/%.o, $(notdir $(SRCS)))

# 指定头文件搜索路径
INC_DIR = $(SRC_DIR)

# 默认目标
$(BUILD_DIR)/$(TARGET): $(OBJS) | $(BUILD_DIR)
	$(CC) -o $@ $^

# 模式规则：编译 .c → .o（make 会通过 VPATH 找到源文件）
$(BUILD_DIR)/%.o: %.c | $(BUILD_DIR)
	$(CC) -c $(addprefix -I, $(INC_DIR)) $< -o $@

# 创建输出目录（伪目标）
$(BUILD_DIR):
	mkdir -p $@

.PHONY: clean
clean:
	rm -rf $(BUILD_DIR)
```

### 4.4 编译验证

```bash
$ make
mkdir -p build
cc -c -I module1 -I module2 module1/mid.c -o build/mid.o
cc -c -I module1 -I module2 module2/main.c -o build/main.o
cc -o build/app3 build/mid.o build/main.o

$ ./build/app3
Hello World!
```

### 4.5 逐步拆解：每个函数在 Makefile 中的角色

以下逐行分析每个函数是如何参与构建过程的。

#### 第1步：扫描源文件（foreach + wildcard）

```makefile
SRC_DIR = module1 module2
SRCS = $(foreach dir, $(SRC_DIR), $(wildcard $(dir)/*.c))
```

**拆解**：

```
foreach 第1次: dir=module1 → wildcard module1/*.c → module1/mid.c
foreach 第2次: dir=module2 → wildcard module2/*.c → module2/main.c
拼接结果: SRCS = module1/mid.c module2/main.c
```

**为什么用 foreach + wildcard 而不是 $(wildcard */*.c)**？

- `$(wildcard */*.c)` 会匹配**所有**一级子目录下的 `.c` 文件
- 无法选择性排除不需要编译的目录（如 `test/`、`backup/`）
- `foreach` + 明确的 `SRC_DIR` 列表让你**精确控制**哪些目录参与编译

#### 第2步：路径转文件名（notdir）

```makefile
# 输入
SRCS = module1/mid.c module2/main.c

# 处理
$(notdir $(SRCS))
# 输出: mid.c main.c
```

这一步去掉了所有目录前缀，为后续统一加 `build/` 前缀做准备。

#### 第3步：后缀替换（patsubst）

```makefile
OBJS = $(patsubst %.c, $(BUILD_DIR)/%.o, $(notdir $(SRCS)))
# 等价于:
# OBJS = $(patsubst %.c, build/%.o, mid.c main.c)
# 结果: build/mid.o build/main.o
```

**为什么分成 notdir + patsubst 两步，而不是一步**？

```makefile
# 不能这样写（错误示范）：
OBJS = $(patsubst %.c, build/%.o, $(SRCS))
# 结果: build/module1/mid.o build/module2/main.o
# 问题：obj 文件嵌套在 build/module1/、build/module2/ 下，不是扁平结构
```

两步法的优势：所有 `.o` 文件统一放在 `build/` 根目录下，结构清晰。

#### 第4步：VPATH 的作用

```makefile
VPATH = $(SRC_DIR)   # VPATH = module1 module2
```

在模式规则 `$(BUILD_DIR)/%.o: %.c` 中：

- 目标：`build/mid.o` → 提取 stem = `mid`
- 前提：`mid.c`
- make 在当前目录找不到 `mid.c`，转而搜索 VPATH 中的目录
- 在 `module1/` 中找到 `mid.c` → 成功匹配

```makefile
# VPATH 的工作原理示意
#  build/mid.o 需要 mid.c
#  → 当前目录: mid.c? 不存在
#  → VPATH[0]=module1: module1/mid.c? 找到！
#  → $< 自动变为 module1/mid.c（含完整路径）
```

#### 第5步：order-only 前提（`|` 管道符）

```makefile
$(BUILD_DIR)/$(TARGET): $(OBJS) | $(BUILD_DIR)
#                               ^
#                          order-only 前提
```

管道符 `|` 后面的 `$(BUILD_DIR)` 是 **order-only 前提**：

- 只要 `build/` 目录存在，就不会因为它比目标新而重新编译
- 避免了每次 `mkdir` 后都触发重新链接的问题

### 4.6 优化效果对比

| 对比项 | 优化前 | 优化后 |
|--------|--------|--------|
| 源文件管理 | 全放当前目录 | 分模块子目录 |
| 添加新模块 | 改 Makefile 多处 | 加 1 个目录名到 `SRC_DIR` |
| 添加新文件 | 手动加到 `OBJS` | 自动扫描 |
| 编译产物 | 散落在源码目录 | 统一在 `build/` |
| 可重用性 | 低 | 高（复制即可用） |

---

## 5. 进阶实战场景

### 5.1 场景一：自动生成依赖文件（`.d` 文件）

当修改头文件时，make 需要知道哪些 `.c` 文件包含了该头文件，才能正确重新编译。手动维护依赖关系不现实，用 gcc 的 `-MMD` 选项可以自动生成 `.d` 依赖文件。

```makefile
CC = gcc
CFLAGS = -Wall -MMD -MP       # -MMD: 生成 .d 依赖文件；-MP: 为每个头文件生成空规则
BUILD_DIR = build

SRCS = $(wildcard *.c)
OBJS = $(patsubst %.c, $(BUILD_DIR)/%.o, $(SRCS))
DEPS = $(patsubst %.c, $(BUILD_DIR)/%.d, $(SRCS))   # 对应的 .d 文件列表

$(BUILD_DIR)/app: $(OBJS)
	$(CC) -o $@ $^

# -MMD 会自动生成 build/main.d 文件，内容类似：
# build/main.o: main.c header.h common.h

$(BUILD_DIR)/%.o: %.c | $(BUILD_DIR)
	$(CC) $(CFLAGS) -c $< -o $@

$(BUILD_DIR):
	mkdir -p $@

# ★ 关键：引入自动生成的 .d 文件
# -include 表示文件不存在也不报错
-include $(DEPS)

.PHONY: clean
clean:
	rm -rf $(BUILD_DIR)
```

**依赖文件 `.d` 的内容示例**：

```makefile
# build/main.d 的内容（由 gcc -MMD 自动生成）
build/main.o: main.c common.h types.h config.h
```

有了这些 `.d` 文件，修改 `common.h` 后，make 就知道应该重新编译 `main.c`。

### 5.2 场景二：多级子目录递归扫描

当项目有嵌套子目录时（如 `core/net/`、`core/io/`），需要递归扫描所有层级：

```makefile
# 方案A：配合 shell find 命令
SRCS = $(shell find src -name "*.c")
# 注意：shell 函数在 make 启动时执行，后续新增文件不会被检测到

# 方案B：使用 wildcard 嵌套（需要知道层级深度）
SRCS = $(wildcard src/*.c src/*/*.c src/*/*/*.c)
# 缺点：层级深度固定，不够灵活

# 方案C：递归 wildcard 函数（推荐，纯 Makefile 实现）
rwildcard = $(foreach d, $(wildcard $(1:=/*)), \
              $(call rwildcard, $d, $(2)) $(filter $(2), $d))
# 用法
SRCS = $(call rwildcard, src, *.c)
```

**方案C 的工作原理**（以 `rwildcard src *.c` 为例）：

```
第1层: 扫描 src/* → 得到 src/core, src/net
      对每个子目录递归: rwildcard src/core *.c
第2层: 扫描 src/core/* → 得到 src/core/main.c, src/core/io
      src/core/main.c 匹配 *.c → 收集
      对 io 递归: rwildcard src/core/io *.c
第3层: 扫描 src/core/io/* → 得到 src/core/io/serial.c
      匹配 *.c → 收集
最终: src/core/main.c src/core/io/serial.c src/net/...
```

### 5.3 场景三：区分编译目标和主机工具

交叉编译场景中，部分工具需要在**主机**（x86）上运行（如代码生成器），目标代码需要在**目标机**（ARM）上运行：

```makefile
# 交叉编译工具链
CROSS_COMPILE = arm-linux-gnueabihf-
CC = $(CROSS_COMPILE)gcc

# 主机编译器（用于生成主机工具）
HOST_CC = gcc

BUILD_DIR = build
HOST_BUILD_DIR = host_build

# 目标机源码和主机工具源码分开
TARGET_SRCS = $(wildcard src/*.c)
HOST_SRCS = $(wildcard tools/*.c)

# 目标机 .o 文件
TARGET_OBJS = $(patsubst src/%.c, $(BUILD_DIR)/%.o, $(TARGET_SRCS))

# 主机工具 .o 文件
HOST_OBJS = $(patsubst tools/%.c, $(HOST_BUILD_DIR)/%.o, $(HOST_SRCS))

# ★ 主机工具使用主机编译器
$(HOST_BUILD_DIR)/codegen: $(HOST_OBJS) | $(HOST_BUILD_DIR)
	$(HOST_CC) -o $@ $^

# 主机工具 .o 编译规则
$(HOST_BUILD_DIR)/%.o: tools/%.c | $(HOST_BUILD_DIR)
	$(HOST_CC) -c $< -o $@

# 目标机程序使用交叉编译器
$(BUILD_DIR)/app: $(TARGET_OBJS) | $(BUILD_DIR)
	$(CC) -o $@ $^

$(BUILD_DIR)/%.o: src/%.c | $(BUILD_DIR)
	$(CC) -c $< -o $@

# 目标程序依赖 codegen（构建时运行代码生成器）
src/generated.c: $(HOST_BUILD_DIR)/codegen
	$(HOST_BUILD_DIR)/codegen -o $@

$(BUILD_DIR) $(HOST_BUILD_DIR):
	mkdir -p $@

.PHONY: clean
clean:
	rm -rf $(BUILD_DIR) $(HOST_BUILD_DIR)
```

---

## 6. 常见错误与排错指南

### 6.1 VPATH 缺失导致找不到源文件

**症状**：

```
make: *** No rule to make target 'mid.c', needed by 'build/mid.o'.  Stop.
```

**原因**：模式规则 `$(BUILD_DIR)/%.o: %.c` 需要的 `%.c` 在子目录中，但 make 不知道去哪里找。

**解决**：添加 `VPATH = $(SRC_DIR)` 或使用 `vpath` 指令：

```makefile
# 方式A：全局 VPATH
VPATH = $(SRC_DIR)

# 方式B：按模式指定搜索路径（更精确）
vpath %.c $(SRC_DIR)
vpath %.h $(INC_DIR)
```

### 6.2 patsubst 与 subst 的混淆

```makefile
# 错误：试图用 subst 做模式替换
OBJS = $(subst %.c, %.o, $(SRCS))
# subst 不支持 % 通配符！它会查找字面量 "%.c"，找不到则原样返回

# 正确：用 patsubst
OBJS = $(patsubst %.c, %.o, $(SRCS))
```

**记忆方法**：名称中的 `pat` = `pattern`（模式），有 `pat` 的才支持 `%`。

### 6.3 foreach 中的变量展开时机

```makefile
# 这是一个常见但容易困惑的写法
SRC_DIR = module1 module2
SRCS = $(foreach dir, $(SRC_DIR), $(wildcard $(dir)/*.c))

# 在 foreach 执行时：
# 1. $(dir) 在每次迭代中替换为当前单词（module1, module2）
# 2. $(wildcard module1/*.c) 被执行
# 3. 结果用空格拼接

# 注意：foreach 中的变量（var 参数）不需要用 $() 包裹定义
# 正确: $(foreach dir, ...
# 错误: $($foreach dir, ...
```

### 6.4 wildcard 在赋值时的"快照"特性

```makefile
# 问题场景
SRCS = $(wildcard *.c)    # ← wildcard 在这一行执行

# ... 后来用脚本生成了新的 .c 文件

all: $(SRCS)
    # SRCS 还是旧的文件列表，新文件被忽略了！
```

**解决方法1**：在规则中调用 wildcard：

```makefile
all:
	@echo $(wildcard *.c)   # 每次 make all 都重新扫描
```

**解决方法2**：使用 `:=` 延迟特定场景的判断逻辑（但 wildcard 仍只在展开时执行一次）。

### 6.5 空格处理陷阱

Make 用空格分割单词，有时不小心引入的多余空格会导致问题：

```makefile
# 陷阱1：变量末尾的额外空格
SRC_DIR = module1 module2    # 假设这里不小心多了一个空格

# $(foreach dir, $(SRC_DIR), ...) 可能迭代出一个空单词！

# 解决方法：
SRC_DIR = $(strip module1 module2)   # strip 去掉首尾和重复空格
```

另外注意 `foreach` 的第一个参数（变量名）和第二个参数（列表）之间是**逗号**，不是空格：

```makefile
$(foreach dir,$(DIRS),$(wildcard $(dir)/*.c))
#            ^-- 这个逗号是参数分隔符
#                 dir 和 $(DIRS) 之间没有空格也不要紧
```

---

## 7. 本讲总结

### 7.1 核心函数速记

| 函数 | 语法 | 作用 | 典型场景 |
|------|------|------|---------|
| `patsubst` | `$(patsubst %.c, %.o, text)` | 模式替换（如 `.c` → `.o`） | 生成目标文件列表 |
| `notdir` | `$(notdir path/file.c)` | 取文件名（去掉路径） | 统一处理带路径的源文件 |
| `wildcard` | `$(wildcard *.c)` | 展开通配符 | 自动扫描源文件 |
| `foreach` | `$(foreach v, list, text)` | 遍历列表，拼接结果 | 遍历多个目录/模块 |

### 7.2 典型组合（一行写出自动化构建列表）

```makefile
# 自动收集多个子目录下的所有 .c 文件
SRCS = $(foreach dir, $(SRC_DIR), $(wildcard $(dir)/*.c))

# 自动生成 .o 目标列表（去掉路径 + 替换后缀 + 加输出目录前缀）
OBJS = $(patsubst %.c, build/%.o, $(notdir $(SRCS)))
```

### 7.3 完整 Makefile 模板（可直接复制使用）

```makefile
CC = gcc
TARGET = app
SRC_DIR = src module1 module2    # 修改为你的源码目录
BUILD_DIR = build

VPATH = $(SRC_DIR)

SRCS = $(foreach dir, $(SRC_DIR), $(wildcard $(dir)/*.c))
OBJS = $(patsubst %.c, $(BUILD_DIR)/%.o, $(notdir $(SRCS)))
DEPS = $(patsubst %.c, $(BUILD_DIR)/%.d, $(notdir $(SRCS)))

CFLAGS = -Wall -MMD -MP $(addprefix -I, $(SRC_DIR))

$(BUILD_DIR)/$(TARGET): $(OBJS) | $(BUILD_DIR)
	$(CC) -o $@ $^

$(BUILD_DIR)/%.o: %.c | $(BUILD_DIR)
	$(CC) $(CFLAGS) -c $< -o $@

$(BUILD_DIR):
	mkdir -p $@

-include $(DEPS)

.PHONY: clean
clean:
	rm -rf $(BUILD_DIR)
```

### 7.4 学习建议

- **先掌握四个核心函数**：`patsubst`、`notdir`、`wildcard`、`foreach`。这4个函数覆盖了90%的实际使用场景。
- **遇到需求再查文档**：不必一次记住所有函数。知道有哪些类别（文本处理、文件名处理、流程控制），需要时翻阅第八章即可。
- **多写多调试验证**：Makefile 的调试可以用 `$(info ...)` 打印中间变量值，也可以用 `make -p` 查看所有规则和变量。

```makefile
# 调试技巧：在 Makefile 中插入这行来查看某个变量的值
$(info DEBUG: SRCS = $(SRCS))
$(info DEBUG: OBJS = $(OBJS))
# 运行 make 时会打印在终端
```

---

*笔记生成日期：2026-07-22 | 转写：Whisper tiny*
*优化日期：2026-07-24 | 新增：第3章（更多常用函数）、第5章（进阶实战场景）、第6章（常见错误与排错指南）*
