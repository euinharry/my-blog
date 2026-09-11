---
title: "第26讲：GCC与Helloworld"
date: 2026-09-11T09:13:00+08:00
draft: false
description: "GCC不仅仅是C编译器，它通过前端（front-end）机制支持多种语言："
series: ["Linux 入门"]
series_order: 14
categories: ["技术笔记"]
tags: ["Linux", "GCC", "编译"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P27)
> **标题**：第26讲 -- GCC与HelloWorld
> **时长**：13分55秒
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. GCC简介

### 1.1 GCC是什么？

> **GCC = GNU Compiler Collection（GNU编译器套件）**

- 由 **GNU组织** 开发的编译器套件，最初仅支持C语言，现已扩展为支持多种语言的编译器集合
- 简单理解为：GCC就是C/C++等语言的**编译工具**
- 注意事项：不同编译器的规则不同，不能混用（如GCC和MSVC的语法扩展有差异）

### 1.2 GCC的发展历史

| 年份 | 事件 |
|------|------|
| **1984** | Richard Stallman（理查德·斯托曼）发起GNU项目 |
| **1987** | Richard Stallman发布GCC第一版（1.0），当时称为 **GNU C Compiler** |
| **1992** | GCC 2.0发布，开始支持C++ |
| **1997** | EGCS（Experimental/Enhanced GNU Compiler System）分支出现，极大加快了GCC发展 |
| **1999** | EGCS被采纳为GCC官方版本，GCC正式更名为 **GNU Compiler Collection** |
| **2001** | GCC 3.0发布，采用全新的优化框架 |
| **2005** | GCC 4.0发布，引入SSA（Static Single Assignment）中间表示 |
| **2015** | GCC 5.0发布，默认C语言标准切换为C11 |
| **至今** | 持续更新中，当前主流版本为GCC 11/12/13/14系列 |

### 1.3 GCC支持的语言

GCC不仅仅是C编译器，它通过**前端（front-end）**机制支持多种语言：

| 命令 | 支持的语言 | 说明 |
|------|-----------|------|
| `gcc` | C | C语言编译器（最常用） |
| `g++` | C++ | C++语言编译器 |
| `gfortran` | Fortran | Fortran语言编译器 |
| `gnat` | Ada | Ada语言编译器 |
| `gcj`（已废弃）| Java | Java编译器（GCC 7起移除） |
| `gccgo` | Go | Go语言编译器 |

> **核心概念**：GCC内部采用"前端/后端"分离架构。前端负责解析不同编程语言并生成统一的中间表示（GIMPLE），后端负责将中间表示优化并生成目标平台的机器码。因此同一套后端可以服务于多种语言前端。

### 1.4 `gcc` 与 `g++` 的区别

| 对比维度 | `gcc` | `g++` |
|---------|-------|-------|
| **默认编译语言** | 根据文件扩展名判断（`.c`当C处理，`.cpp`当C++处理） | 始终按C++编译 |
| **链接时的库** | 不会自动链接C++标准库 | 自动链接C++标准库（`-lstdc++`） |
| **适用场景** | 编译纯C程序 | 编译C++程序或混合编译 |

> **实践经验**：编译 `.c` 文件用 `gcc`，编译 `.cpp` 文件用 `g++`。如果用 `gcc` 编译 C++ 代码并在链接阶段遇到 `undefined reference to std::...` 错误，需要手动加上 `-lstdc++` 链接选项。

### 1.5 GCC全称演变

```
最初：GCC = GNU C Compiler          （仅C语言编译器）
现在：GCC = GNU Compiler Collection  （编译器套件，支持多语言）
```

GNU组织在整个Linux发展历程中贡献巨大，致力于开发**自由软件**来对抗收费软件。

### 1.6 查看GCC版本

```bash
# 查看GCC版本（简洁）
gcc --version

# 查看GCC版本（详细，包含配置选项）
gcc -v
```

**输出解读示例**（Ubuntu 20.04）：

```
gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0
Copyright (C) 2019 Free Software Foundation, Inc.
```

| 字段 | 含义 |
|------|------|
| `gcc` | 编译器名称 |
| `(Ubuntu 9.4.0-1ubuntu1~20.04.2)` | 发行版打包信息：Ubuntu维护的9.4.0版本，适用于20.04 |
| `9.4.0` | 上游GCC版本号（主版本.次版本.修订版本） |
| `Copyright (C) 2019` | FSF版权声明年份 |

---

## 2. 影响Linux的重要组织/技术

| 组织/技术 | 影响 |
|-----------|------|
| **GNU组织** | 开发GCC等自由软件，为Linux提供了编译器工具链。很多人把Linux称为 GNU/Linux |
| **Unix系统** | Linux的前身，其命令行接口被Linux继承 |
| **Minix系统** | 用于教学的操作系统，Linux创始人Linus通过Minix学习操作系统知识 |
| **POSIX接口** | 操作系统统一接口标准，遵循POSIX的系统可轻松移植应用程序，推动了Linux生态发展 |
| **Internet** | 没有互联网就没有Linux今天的发展规模 |

### 2.1 GNU项目背景与GNU/Linux称谓

**GNU项目**（GNU's Not Unix）由Richard Stallman于1984年发起，目标是创建一个完全自由的类Unix操作系统。到1990年代初，GNU项目已经完成了除内核（Hurd）之外的大部分组件，包括：

- GCC（编译器）
- Bash（Shell）
- Emacs（编辑器）
- glibc（C标准库）
- Coreutils（核心工具集，ls、cp、mv等）

1991年Linus Torvalds发布了Linux内核，填补了GNU项目缺失的关键部分。Linux内核结合GNU工具链，构成了完整的操作系统。因此严谨的叫法是 **"GNU/Linux"**，但日常使用中常简称为"Linux"。

### 2.2 GPL协议基本概念

**GPL（GNU General Public License，GNU通用公共许可证）**由Richard Stallman于1989年发布，是自由软件运动的基石。核心原则：

| 原则 | 说明 |
|------|------|
| **自由使用** | 任何人可以自由运行该软件，用于任何目的 |
| **自由学习** | 可以查看和修改源代码（必须提供源码） |
| **自由分发** | 可以复制和分发原始软件 |
| **自由改进** | 可以分发修改后的版本 |
| **Copyleft（著佐权）** | 关键约束：修改后的衍生作品也必须以GPL协议发布 |

> GPL v2：1991年发布，Linux内核采用此版本。
> GPL v3：2007年发布，增加了对专利和硬件限制的保护条款。

> 不要盲目追捧Linux，应全面客观认识各种技术的历史贡献。

---

## 3. GCC编译工具链的组成

### 3.1 编译四阶段总览

C程序的编译过程分为**四个阶段**：

```
源代码(.c) 直到 预处理 直到 编译 直到 汇编 直到 链接 直到 可执行文件
   hello.c     hello.i    hello.s   hello.o    hello
```

GCC编译工具链由两大部分组成：

| 组成部分 | 负责阶段 | 说明 |
|---------|---------|------|
| **GCC编译器** | 预处理 + 编译 | 将C代码转为汇编代码 |
| **Binutils工具集** | 汇编 + 链接 | 将汇编代码转为机器码并链接 |

> GCC命令可以统一调用这两个工具集中的工具。

### 3.2 阶段一：预处理（Preprocessing）

**功能**：处理源代码中以 `#` 开头的预处理器指令。

**主要操作**：
- `#include` -- 头文件展开：将 `<stdio.h>` 等头文件内容直接插入源文件
- `#define` -- 宏替换：将代码中的宏名替换为对应的值或表达式
- `#ifdef / #ifndef / #if / #else / #endif` -- 条件编译：根据条件选择性保留代码
- 删除注释：移除 `//` 和 `/* */` 注释
- 添加行号标记：插入 `#line` 标识以便编译器报错时定位

**实际操作**：

```bash
# 仅执行预处理，查看预处理后的代码
gcc -E hello.c -o hello.i

# 更接近实际编译时的预处理（保留行号信息）
gcc -E hello.c | head -50
```

> 预处理的输出文件通常很大。一个简单的 `printf("hello world")` 经过预处理后，`#include <stdio.h>` 会展开成千上万行。

### 3.3 阶段二：编译（Compilation）

**功能**：将预处理后的C代码翻译为汇编代码。

**主要操作**：
- **词法分析**：将代码拆分为token（关键字、标识符、运算符、字面量等）
- **语法分析**：根据C语言语法构建抽象语法树（AST）
- **语义分析**：检查类型匹配、变量声明、作用域等
- **中间代码生成**：生成平台无关的中间表示（GIMPLE）
- **汇编代码生成**：将中间表示翻译为特定CPU架构的汇编代码

**实际操作**：

```bash
# 仅执行预处理+编译，生成汇编代码
gcc -S hello.c -o hello.s

# 查看生成的汇编代码
cat hello.s
```

**汇编代码示例**（x86-64，简化版）：

```asm
        .section        .rodata
.LC0:
        .string "hello world"
        .text
        .globl  main
        .type   main, @function
main:
        pushq   %rbp
        movq    %rsp, %rbp
        leaq    .LC0(%rip), %rdi
        call    puts@PLT
        movl    $0, %eax
        popq    %rbp
        ret
```

### 3.4 阶段三：汇编（Assembly）

**功能**：将汇编代码翻译为机器码，生成**目标文件**（Object File, `.o`）。

**主要操作**：
- 将汇编助记符（mov、push、call等）翻译为二进制机器指令
- 生成符号表：记录函数名、全局变量名及其地址
- 生成重定位表：标记需要在链接阶段修正的地址引用
- 按目标文件格式（Linux下为ELF）组织数据

**实际操作**：

```bash
# 仅执行预处理+编译+汇编，生成目标文件（不链接）
gcc -c hello.c -o hello.o

# 或从汇编代码生成
gcc -c hello.s -o hello.o

# 查看目标文件信息
file hello.o
# 输出：hello.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
```

### 3.5 阶段四：链接（Linking）

**功能**：将一个或多个目标文件与所需的库文件组合，生成最终的可执行文件。

**主要操作**：
- **符号解析**：将 `printf`、`puts` 等外部函数引用与实际定义关联
- **地址重定位**：修正所有函数和变量的最终内存地址
- **库链接**：链接必要的C运行时库（`libc`）、启动文件（`crt0.o`）等

**两种链接方式**：

| 链接方式 | 操作 | 优点 | 缺点 |
|---------|------|------|------|
| **动态链接**（默认） | 运行时加载共享库（.so） | 可执行文件小；库更新无需重新编译 | 依赖系统库环境；启动略慢 |
| **静态链接** | 将库代码全部打包进可执行文件 | 独立运行，不依赖外部库 | 文件体积大（可能数MB） |

```bash
# 动态链接（默认）
gcc hello.c -o hello_dynamic

# 静态链接
gcc -static hello.c -o hello_static

# 对比文件大小
ls -lh hello_dynamic hello_static
```

**查看可执行文件信息**：

```bash
# 1. 查看文件类型
file hello

# 2. 查看ELF文件头
readelf -h hello

# 3. 查看动态库依赖
ldd hello                    # 显示依赖的.so文件
ldd hello_static             # 输出：not a dynamic executable
```

### 3.6 使用 -v 查看编译阶段的详细调用

```bash
gcc -v hello.c -o hello
```

> `-v` 揭示了GCC并非单一程序，而是协调调用 `cc1`（编译器本体）、`as`（汇编器）、`collect2`（链接器）等多个工具。

---

## 4. 常用GCC编译选项

### 4.1 输出控制选项

| 选项 | 功能 | 示例 |
|------|------|------|
| `-o <file>` | 指定输出文件名 | `gcc hello.c -o hello` |
| `-E` | 仅预处理，输出到标准输出 | `gcc -E hello.c` |
| `-S` | 预处理+编译，生成汇编文件（.s） | `gcc -S hello.c -o hello.s` |
| `-c` | 预处理+编译+汇编，生成目标文件（.o） | `gcc -c hello.c -o hello.o` |
| `-v` | 显示编译过程的详细信息 | `gcc -v hello.c` |

### 4.2 警告与调试选项

| 选项 | 功能 | 说明 |
|------|------|------|
| `-Wall` | 开启几乎所有常用警告 | 编译时必须开启 |
| `-Wextra` | 开启额外警告（-Wall未覆盖的） | 可与-Wall组合使用 |
| `-Werror` | 将警告视为错误 | 严格模式，警告会导致编译失败 |
| `-w` | 禁止所有警告 | 不推荐使用 |
| `-g` | 添加调试信息 | 生成供GDB调试使用的符号信息 |
| `-g0` / `-g1` / `-g2` / `-g3` | 调试信息详细程度 | 数字越大越详细，-g默认为-g2 |

### 4.3 优化选项

| 选项 | 功能 | 编译速度 | 运行速度 | 适用场景 |
|------|------|---------|---------|---------|
| `-O0`（默认） | 不优化 | 最快 | 最慢 | 开发调试阶段 |
| `-O1` | 基础优化 | 较快 | 较快 | 需要一定性能但不希望编译太久 |
| `-O2` | 标准优化（推荐） | 中等 | 快 | 大多数发布场景 |
| `-O3` | 激进优化 | 慢 | 最快 | 性能敏感场景 |
| `-Os` | 优化代码体积 | 中等 | 较快 | 嵌入式/存储受限场景 |
| `-Ofast` | 极致优化（可能不符合标准） | 慢 | 极快 | 科学计算等场景 |

### 4.4 头文件与库路径选项

| 选项 | 功能 | 示例 |
|------|------|------|
| `-I<dir>` | 添加头文件搜索路径 | `gcc -I/home/user/include hello.c` |
| `-L<dir>` | 添加库文件搜索路径 | `gcc -L/home/user/lib hello.c -lmylib` |
| `-l<name>` | 链接库文件（如libm.so） | `gcc hello.c -lm` |
| `-static` | 强制静态链接 | `gcc -static hello.c -o hello` |
| `--sysroot=<dir>` | 指定系统根目录（交叉编译） | `arm-linux-gnueabihf-gcc --sysroot=/sysroot hello.c` |

### 4.5 完整编译选项汇总表

| 类别 | 选项 | 简要说明 |
|------|------|----------|
| **阶段控制** | `-E` | 仅预处理 |
|  | `-S` | 预处理+编译（生成汇编） |
|  | `-c` | 预处理+编译+汇编（生成.o） |
| **输出** | `-o file` | 指定输出文件名 |
| **警告** | `-Wall` | 开启主要警告 |
|  | `-Wextra` | 开启额外警告 |
|  | `-Werror` | 警告即错误 |
|  | `-w` | 禁止所有警告 |
| **调试** | `-g` | 生成调试信息 |
| **优化** | `-O0` | 不优化 |
|  | `-O1` | 基本优化 |
|  | `-O2` | 标准优化 |
|  | `-O3` | 激进优化 |
|  | `-Os` | 优化体积 |
| **路径** | `-I dir` | 头文件搜索路径 |
|  | `-L dir` | 库搜索路径 |
|  | `-l name` | 链接指定库 |
| **链接** | `-static` | 静态链接 |
|  | `-shared` | 生成共享库 |
| **标准** | `-std=c11` | 指定C语言标准 |
| **宏** | `-D name` | 定义宏 |
| **诊断** | `-v` | 显示详细编译过程 |

---

## 5. 在Ubuntu PC上运行第一个HelloWorld

### 5.1 编写代码

```c
#include <stdio.h>

int main() {
    printf("hello world\n");
    return 0;
}
```

### 5.2 编译

```bash
# Ubuntu 18.04 及以上版本自带GCC，无需安装

# 编译 hello.c，生成可执行文件 hello
gcc hello.c -o hello

# 如果不指定 -o，默认生成 a.out
gcc hello.c          # 生成 a.out
```

### 5.3 运行

```bash
# 查看生成的文件
ls -l hello

# 运行（需要有执行权限）
./hello
# 输出：hello world
```

### 5.4 分析可执行文件

```bash
# 查看文件类型
file hello
# 输出示例：hello: ELF 64-bit LSB pie executable, x86-64, ...

# 查看ELF文件头
readelf -h hello

# 查看动态库依赖
ldd hello
# 输出：
#   linux-vdso.so.1 (0x...)
#   libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x...)
#   /lib64/ld-linux-x86-64.so.2 (0x...)
```

---

## 6. 在开发板（Debian系统）上运行HelloWorld

### 6.1 安装GCC

```bash
# 1. 更新软件源
sudo apt update

# 2. 安装GCC
sudo apt install gcc -y

# 3. 查看版本
gcc -v
gcc --version
# 输出：gcc 8.3.0
```

### 6.2 编写、编译、运行

```bash
# 编写
vi hello.c

# 编译
gcc hello.c -o hello

# 运行
./hello
# 输出：hello world
```

### 6.3 关键结论

> GCC不只是在Ubuntu上才能使用，**开发板上的Debian系统也可以使用GCC**。
>
> 区别仅在于它们针对的是**不同的硬件平台**（x86 vs ARM）。

**交叉编译概念**：
- 在x86 PC上编译运行在ARM开发板的程序，称为**交叉编译**
- 需要使用 **交叉编译工具链**，例如 `arm-linux-gnueabihf-gcc`

---

## 7. 实操演示：分步编译与选项对比实验

### 7.1 样本代码

```c
#include <stdio.h>

#define GREETING "hello world"
#define VERSION  1

int main() {
    printf("%s (version %d)\n", GREETING, VERSION);
    return 0;
}
```

### 7.2 分步编译演示

```bash
# 步骤1：预处理
gcc -E hello.c -o hello.i

# 步骤2：编译
gcc -S hello.c -o hello.s

# 步骤3：汇编
gcc -c hello.c -o hello.o

# 步骤4：链接
gcc hello.o -o hello

# 运行
./hello
```

### 7.3 编译选项对比实验

```bash
# 实验1：优化等级对体积的影响
gcc -O0 hello.c -o hello_O0
gcc -O1 hello.c -o hello_O1
gcc -O2 hello.c -o hello_O2
gcc -O3 hello.c -o hello_O3
gcc -Os hello.c -o hello_Os
ls -lh hello_O*           # 比较文件大小

# 实验2：静态链接 vs 动态链接
gcc hello.c -o hello_dynamic
gcc -static hello.c -o hello_static
ls -lh hello_dynamic hello_static

# 查看动态依赖
echo "=== dynamic ===" && ldd hello_dynamic
echo "=== static ===" && ldd hello_static

# 实验3：警告选项对比
cat > warn_test.c << 'EOF'
#include <stdio.h>
int main() {
    int x;
    printf("%d\n", x);    // 未初始化就使用
    return;
}
EOF

gcc warn_test.c -o warn_test            # 无警告
gcc -Wall warn_test.c -o warn_test      # 显示警告
gcc -Wall -Werror warn_test.c 2>&1      # 警告变错误，编译失败

# 实验4：使用 -D 进行条件编译
cat > debug_test.c << 'EOF'
#include <stdio.h>
int main() {
#ifdef DEBUG
    printf("Debug: %s:%d\n", __FILE__, __LINE__);
#endif
    printf("Hello\n");
    return 0;
}
EOF

gcc debug_test.c -o debug_test && ./debug_test         # 无调试输出
gcc -DDEBUG debug_test.c -o debug_test && ./debug_test # 有调试输出

# 实验5：查看编译各阶段调用的子程序
gcc -v hello.c -o hello 2>&1 | grep -E "(cc1|as|collect2)"
```

### 7.4 深入分析ELF文件

```bash
# 查看ELF文件头
readelf -h hello

# 查看节头表
readelf -S hello

# 查看符号表
readelf -s hello | head -30

# 查看程序头（段信息）
readelf -l hello
```

---

## 8. 本讲总结

### 8.1 核心概念

| 概念 | 说明 |
|------|------|
| **GCC** | GNU Compiler Collection，支持C/C++/Fortran/Ada/Go等多语言 |
| **编译四阶段** | 预处理(.c直到.i) 直到 编译(.i直到.s) 直到 汇编(.s直到.o) 直到 链接(.o直到可执行文件) |
| **工具链组成** | GCC编译器（预处理+编译） + Binutils（汇编+链接） |
| **文件类型** | ELF：Linux下标准的可执行文件和目标文件格式 |
| **链接方式** | 动态链接（默认，体积小） vs 静态链接（-static，独立运行） |

### 8.2 常用操作速查

| 操作 | 命令 |
|------|------|
| 编译C程序 | `gcc source.c -o output` |
| 仅预处理 | `gcc -E source.c -o output.i` |
| 仅编译（生成汇编） | `gcc -S source.c -o output.s` |
| 仅汇编（生成目标文件） | `gcc -c source.c -o output.o` |
| 默认输出 | 不加 -o 生成 `a.out` |
| 开启警告 | `gcc -Wall -Wextra source.c -o output` |
| 开启调试信息 | `gcc -g source.c -o output` |
| 开启优化 | `gcc -O2 source.c -o output` |
| 静态链接 | `gcc -static source.c -o output` |
| 查看版本 | `gcc --version` 或 `gcc -v` |
| 查看文件类型 | `file executable` |
| 查看ELF头 | `readelf -h executable` |
| 查看动态库依赖 | `ldd executable` |

### 8.3 推荐编译命令模板

```bash
# 开发调试阶段
gcc -g -O0 -Wall -Wextra source.c -o output

# 发布版本
gcc -O2 -Wall -Werror source.c -o output
```

---

*笔记生成日期：2026-07-22 | 转写：Whisper tiny*
*笔记优化日期：2026-07-24 | 补充：GCC历史、编译选项详解、实操演示等*
