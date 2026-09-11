---
title: "第28讲：Linux系统HelloWorld（上）"
date: 2026-09-11T09:11:00+08:00
draft: false
description: "HelloWorld 虽然简单，却蕴含了 Linux 系统运行程序的完整机制："
series: ["Linux 入门"]
series_order: 16
categories: ["技术笔记"]
tags: ["Linux", "GCC", "编译"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P29)
> **标题**：第28讲 -- Linux系统HelloWorld（上）
> **时长**：14分10秒
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. 为什么要深入分析HelloWorld？

### 1.1 最简单的程序，最复杂的机制

HelloWorld 虽然简单，却蕴含了 Linux 系统运行程序的完整机制：

- **哲学层面**：越简单的表象，背后机制越复杂
- 很多开发者对 HelloWorld 只有模糊印象
- 细节层面往往模糊不清

### 1.2 从单片机转ARM Linux的常见误区

从单片机转过来的学员，容易用**裸机思维**理解 Linux 程序：
- 裸机程序 = 直接操作硬件
- Linux程序 = 通过操作系统间接操作硬件
- 两者运行机制已完全不同

### 1.3 深入分析的意义

| 意义 | 说明 |
|------|------|
| **借鉴意义** | 搞懂HelloWorld，后续开发其他程序可举一反三 |
| **定位错误** | 编译/链接/运行出错时，能准确判断问题环节 |
| **系统性解决问题** | 从全局角度排查，而不是盲目尝试 |

---

## 2. 单片机（裸机）HelloWorld开发流程

```
① 编写C源文件
    └── main -> printf -> puts -> 串口驱动
② 使用IDE（Keil/ADS1.2）编译
    └── IDE封装了底层编译细节，一键编译
③ 烧录到芯片
    └── 常见芯片：STC89、STM32、S3C2440
    └── IDE中一键烧录
④ 运行程序
```

**特点**：
- IDE封装了几乎所有底层细节
- 开发者只需关注业务逻辑
- 不需要了解编译链接原理

---

## 3. Linux系统HelloWorld执行流程

总览：分为 **4个部分**

### 第一部分：预处理与编译

```c
#include <stdio.h>
int main() {
    printf("hello world\n");
    return 0;
}
```

| 步骤 | 操作 | 输入 -> 输出 | 说明 |
|------|------|-------------|------|
| ① | **预处理** | `hello.c` -> `hello.i` | 处理头文件包含、宏定义、条件编译 |
| ② | **编译** | `hello.i` -> `hello.s` | 语法分析、词法分析 -> 生成汇编文件 |
| ③ | **汇编** | `hello.s` -> `hello.o` | 汇编器 -> 生成可重定位目标文件（.o） |
| ④ | **链接** | `hello.o` -> 可执行文件 | 静态链接 / 动态链接 |

```bash
# 分步编译命令
gcc -E hello.c -o hello.i      # ①预处理
gcc -S hello.i -o hello.s      # ②编译
gcc -c hello.s -o hello.o      # ③汇编
gcc hello.o -o hello           # ④链接
```

---

### 第①步详解：预处理（Preprocessing）

预处理是编译过程的第一步，由**预处理器（cpp）**完成。它不生成机器码，只做文本层面的变换。

#### 预处理具体做什么

| 操作 | 说明 | 示例 |
|------|------|------|
| **头文件展开** | 将 `#include` 指定的头文件内容原样插入到当前文件 | `#include <stdio.h>` 会把整个 stdio.h 展开（通常几千行） |
| **宏定义替换** | 将 `#define` 定义的宏名替换为对应的宏体 | `#define MAX 100` -> 后续所有 `MAX` 替换为 `100` |
| **条件编译** | 根据 `#ifdef`/`#ifndef`/`#if` 决定保留或删除某段代码 | `#ifdef DEBUG` ... `#endif`：只有定义了 DEBUG 才保留 |
| **注释删除** | 删除所有 `//` 和 `/* */` 注释 | 注释被替换为空格 |
| **行号标记** | 插入 `#line` 指令，方便后续编译器报错时定位原始行号 | `#line 1 "hello.c"` |

> **要点**：预处理阶段的输出 `hello.i` 仍然是**文本文件**，内容是展开后的纯C语言代码。一个简单的 `#include <stdio.h>` + `printf("hello\n")` 经过预处理后可能变成几百甚至上千行。

#### 查看预处理结果

```bash
# 方式1：生成 .i 文件后查看
gcc -E hello.c -o hello.i
wc -l hello.i          # 查看预处理后多少行（通常几百到几千行）
less hello.i           # 翻阅 .i 文件

# 方式2：直接输出到终端（管道分页）
gcc -E hello.c | less
```

> **实验提示**：打开 `hello.i` 翻到最后，你会看到自己的 `main` 函数依然存在，但顶部的 `#include <stdio.h>` 已经被替换成了 stdio.h 的全部内容。

#### `#include` 头文件搜索路径

预处理器搜索头文件时，按以下顺序查找：

```bash
#include <stdio.h>     # 尖括号：只搜索系统头文件目录
#include "myheader.h"  # 双引号：先搜当前目录，再搜系统目录
```

查看 GCC 默认搜索路径：

```bash
# 查看 C 头文件的默认搜索路径
gcc -xc -E -v - </dev/null 2>&1 | grep "^ "
# 典型输出包含：
#   /usr/include
#   /usr/local/include
#   /usr/lib/gcc/x86_64-linux-gnu/9/include
```

手动指定头文件搜索路径：

```bash
gcc -I ./include -I /path/to/headers hello.c -o hello
```

#### `-D` 选项：在命令行定义宏

```bash
# 等价于在代码中写 #define DEBUG
gcc -DDEBUG hello.c -o hello

# 定义带值的宏
gcc -DBUFFER_SIZE=1024 hello.c -o hello

# 定义宏值为字符串
gcc -DVERSION=\"1.0.0\" hello.c -o hello
```

实用场景：条件编译控制调试输出：

```c
#ifdef DEBUG
    printf("Debug: x = %d\n", x);
#endif
```

```bash
# 调试版本
gcc -DDEBUG -g hello.c -o hello_debug

# 发布版本
gcc -O2 hello.c -o hello_release
```

---

### 第②步详解：编译（Compilation）

编译阶段将预处理后的 `.i` 文件翻译为汇编语言 `.s` 文件。这是**最复杂**的一个阶段。

#### 编译的具体工作

| 阶段 | 说明 |
|------|------|
| **词法分析** | 识别关键字、标识符、字面量、运算符等 token。每个 token 被赋予一个类型码。 |
| **语法分析** | 根据 C 语言的语法规则（BNF范式），将 token 序列组织成**语法树（AST）**，检查语法错误。 |
| **语义分析** | 检查类型匹配、变量声明、函数调用是否合法。例如 `int x = "hello"` 会在此阶段报错。 |
| **中间代码生成** | 生成平台无关的中间表示（IR），GCC 使用的 IR 叫 GIMPLE。 |
| **优化** | 在 IR 层面进行优化：常量折叠、死代码删除、循环优化等。 |
| **目标代码生成** | 将优化后的 IR 翻译为目标平台的**汇编代码**，此时涉及寄存器分配、指令选择等。 |

#### 查看汇编输出

```bash
# 生成汇编文件
gcc -S hello.i -o hello.s
# 或直接从 .c 生成
gcc -S hello.c -o hello.s

# 查看汇编代码
cat hello.s
```

汇编输出示例（简化版，x86_64）：

```asm
    .section    .rodata
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

#### 编译优化的基本原理

GCC 提供多个优化级别：

| 选项 | 说明 |
|------|------|
| `-O0` | 默认，不优化。编译速度快，方便调试。 |
| `-O1` | 基础优化。常量折叠、消除无用代码、减少分支跳转。 |
| `-O2` | 标准优化。包含 `-O1` 所有优化，加上指令调度、循环展开、函数内联等。 |
| `-O3` | 激进优化。在 `-O2` 基础上增加更激进的循环优化、向量化等。编译更慢，程序可能更大。 |
| `-Os` | 优化代码尺寸。等价于 `-O2` 但跳过会增加代码体积的优化。适合嵌入式。 |

```bash
# 对比不同优化级别的汇编输出
gcc -S -O0 hello.c -o hello_O0.s
gcc -S -O2 hello.c -o hello_O2.s
diff hello_O0.s hello_O2.s    # 对比差异
```

---

### 第③步详解：汇编（Assembly）

汇编阶段将汇编语言 `.s` 文件转换为**机器指令**，封装在 `.o` 文件中。

#### ELF 文件格式基本概念

ELF（Executable and Linkable Format）是 Linux 下可执行文件、目标文件、共享库的标准格式。一个 ELF 文件由以下部分组成：

```
+---------------------+
|    ELF Header       |  魔数、架构、入口地址等基本信息
+---------------------+
|  Program Header     |  运行时需要的段信息（可执行文件才有）
|    Table            |
+---------------------+
|  .text (代码段)      |  编译后的机器指令
|  .rodata (只读数据)   |  字符串常量等
|  .data (已初始化数据)  |  已初始化的全局/静态变量
|  .bss (未初始化数据)   |  未初始化的全局变量（不占文件空间）
|  .symtab (符号表)    |  函数名、全局变量名等符号信息
|  .strtab (字符串表)   |  符号名称的字符串
|  .rel.text           |  代码段重定位信息
|  .rel.data           |  数据段重定位信息
+---------------------+
|  Section Header     |  节的描述信息
|    Table            |
+---------------------+
```

#### `.o` 文件（可重定位目标文件）的特点

- `.o` 文件中的地址是**相对地址**（从0开始），尚未确定最终的内存位置
- 函数调用地址是占位符，需要后续链接阶段**重定位**
- 包含符号表，记录了本模块定义了哪些符号、引用了哪些外部符号

#### `objdump -d` 反汇编

```bash
# 生成 .o 文件
gcc -c hello.s -o hello.o

# 反汇编 .o 文件，查看机器码和汇编的对应关系
objdump -d hello.o

# 查看全部信息（符号表、段信息等）
objdump -d -t -r hello.o
```

`objdump -d` 输出示例：

```
0000000000000000 <main>:
   0:   55                      push   %rbp
   1:   48 89 e5                mov    %rsp,%rbp
   4:   48 8d 3d 00 00 00 00    lea    0x0(%rip),%rdi
   b:   e8 00 00 00 00          callq  10 <main+0x10>
  10:   b8 00 00 00 00          mov    $0x0,%eax
  15:   5d                      pop    %rbp
  16:   c3                      retq
```

> **注意**：`.o` 阶段的反汇编中，`callq` 和 `lea` 指令的操作数都是 `00 00 00 00`（占位符），因为此时还不知道字符串和函数的最终地址。这些空白将在链接阶段由链接器填充。

---

### 第④步详解：链接（Linking）

链接是编译的最后一步，由**链接器（ld）**将多个 `.o` 文件和库文件合并成一个可执行文件。

#### 链接的具体工作

| 步骤 | 说明 |
|------|------|
| **符号解析** | 将每个 `.o` 文件中对「外部符号」的引用（如 `printf`）与库中的符号定义进行匹配。如果找不到定义，报 `undefined reference` 错误。 |
| **重定位** | 将 `.o` 文件中相对地址修正为最终的可执行地址。把 `call 00 00 00 00` 修正为 `call <printf的实际地址>`。 |
| **段合并** | 将所有输入文件的同类型段（如多个 `.text` 段）合并为一个段。 |

#### 静态链接 vs 动态链接

| 特性 | 静态链接 | 动态链接 |
|------|---------|---------|
| 链接时机 | 编译时 | 运行时 |
| 链接器行为 | 把库代码完整复制到可执行文件 | 只在可执行文件中记录「需要哪个库的哪个函数」 |
| 文件大小 | 较大（包含所有库代码） | 较小（不包含库代码） |
| 移植性 | 好（不依赖外部环境） | 需目标系统有对应库 |
| 内存占用 | 较高（每个进程各有一份库代码） | 较低（多个进程可共享同一份库的物理内存） |
| 库文件扩展名 | `.a`（archive） | `.so`（shared object） |

```bash
# 默认是动态链接
gcc hello.c -o hello

# 强制静态链接
gcc -static hello.c -o hello_static

# 对比文件大小
ls -lh hello hello_static
# hello:        约 16KB
# hello_static: 约 800KB~2MB
```

#### 常用分析工具命令

**(1) `file` -- 查看文件类型**

```bash
file hello
# 输出: hello: ELF 64-bit LSB executable, x86-64, dynamically linked, ...
file hello.o
# 输出: hello.o: ELF 64-bit LSB relocatable, x86-64, ...
```

**(2) `ldd` -- 查看动态库依赖**

```bash
ldd hello
# 输出示例:
#   linux-vdso.so.1 (0x00007ffe3f3e1000)
#   libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f8b3c000000)
#   /lib64/ld-linux-x86-64.so.2 (0x00007f8b3c200000)
```

输出解读：
- `linux-vdso.so.1`：虚拟动态共享对象，是内核提供的快速系统调用接口
- `libc.so.6`：C 标准库（glibc），`printf` 的定义在这里
- `ld-linux-x86-64.so.2`：动态链接器本身

**(3) `readelf -d` -- 查看动态链接信息**

```bash
readelf -d hello
# 输出包含:
#  NEEDED   libc.so.6          # 需要 libc.so.6
#  NEEDED   ld-linux-x86-64.so.2
#  ...
```

**(4) `nm` -- 查看符号表**

```bash
nm hello.o
# 输出示例:
#                  U _GLOBAL_OFFSET_TABLE_
# 0000000000000000 T main           # T = .text 段，已定义
#                  U printf         # U = Undefined，引用外部符号
#                  U puts           # GCC 可能将 printf 优化为 puts

nm hello | grep printf
# 链接后，printf 不再显示为 U（Undefined），因为已解析
```

**(5) `objdump -d` -- 反汇编可执行文件**

```bash
objdump -d hello | grep -A 15 '<main>:'
# 对比 .o 阶段的反汇编，call 指令的操作数不再是 00 00 00 00
# 而是被重定位为 puts 的 PLT 入口地址
```

---

### 第二部分：Shell执行程序

```bash
# 终端输入
./hello
```

Shell 的执行过程：

#### `fork()` -- 创建子进程

`fork()` 创建一个与父进程**几乎完全相同**的副本。子进程拥有父进程的代码、数据、堆栈、文件描述符的拷贝，但有不同的 PID。

```c
pid_t pid = fork();
if (pid == 0) {
    // 子进程
} else if (pid > 0) {
    // 父进程
} else {
    // fork 失败
}
```

`fork()` 的**写时复制（Copy-on-Write, COW）**机制：
- fork 时不会立即复制父进程的全部内存，而是让父子进程共享同一份物理页
- 只有当任一进程尝试写入某个页时，内核才真正复制该页
- 这大幅提升了 fork 的性能

#### `execve()` -- 替换进程映像

`execve()` 做了 fork 的反面：它保留当前进程的 PID、文件描述符等元信息，但**用新程序替换当前进程的全部代码和数据**。

```c
execve("/path/to/hello", argv, envp);
// 执行成功后不会返回（原进程已经被替换）
// 失败时返回 -1
```

Shell 调用的完整简化逻辑：

```
1. Shell 读取用户输入 "./hello"
2. Shell 调用 fork() 创建子进程
3. 父进程（Shell）等待子进程结束（waitpid）
4. 子进程调用 execve("./hello", ...) 替换自身为 hello 程序
5. 内核开始加载并执行 hello（见第三部分）
```

---

### 第三部分：内核加载（execve系统调用流程）

```
execve()
  +-- sys_execve （系统调用入口）
        +-- do_execve
              +-- load_elf_binary （加载ELF文件）
                    |-- 从文件系统加载Hello程序到内存
                    |-- [分支1] 静态链接程序
                    |     +-- 直接执行 _start 入口
                    +-- [分支2] 动态链接程序
                          +-- 动态链接器加载库文件
                                +-- 建立程序与库的链接
                                      +-- 执行 _start 入口
```

#### ELF 加载后的进程内存布局

```
      高地址
    +---------------+
    |    栈 (Stack)   |  <- 局部变量、函数调用帧。向低地址增长。
    |       v       |
    |       ...     |
    +---------------+
    |               |  <- 内存映射区域（mmap）：动态库 .so 文件映射在此
    |               |
    +---------------+
    |    堆 (Heap)    |  <- malloc 分配的内存。向高地址增长。
    |       ^       |
    +---------------+
    |   .bss 段      |  <- 未初始化的全局/静态变量（运行时清零）
    +---------------+
    |   .data 段     |  <- 已初始化的全局/静态变量
    +---------------+
    |   .text 段     |  <- 代码（机器指令），只读
    +---------------+
      低地址
```

各段说明：

| 段名 | 内容 | 权限 | 来源 |
|------|------|------|------|
| `.text` | 程序的机器指令 | r-x（读+执行） | 从 ELF 文件加载 |
| `.rodata` | 字符串常量、const 变量 | r--（只读） | 从 ELF 文件加载 |
| `.data` | 已初始化的全局/静态变量 | rw-（读写） | 从 ELF 文件加载 |
| `.bss` | 未初始化的全局/静态变量 | rw-（读写） | 运行时分配并清零（不占文件空间） |
| 堆 (Heap) | `malloc/new` 分配的内存 | rw-（读写） | `brk/sbrk` 系统调用动态扩展 |
| 栈 (Stack) | 局部变量、函数参数、返回地址 | rw-（读写） | 运行时自动管理，向下增长 |
| 内存映射区 | 动态库 `.so`、`mmap` 映射 | 取决于用途 | `mmap` 系统调用 |

#### 虚拟内存与物理内存

**虚拟内存**是每个进程「以为」自己拥有的地址空间。在 64 位系统中，理论上可达 2^64 字节（实际上 Linux 用户空间限制为 128TB）。

**物理内存**是实际的 RAM 硬件。

两者的关系：

```
  进程A的虚拟地址空间          物理内存              进程B的虚拟地址空间
  +--------------+          +--------+          +--------------+
  | 0x7fff... (栈)|  --->   | 物理页1 |  <---    | 0x7fff... (栈)|
  | 0x6000...(堆) |  --->   | 物理页2 |          | 0x6000...(堆) |
  | 0x4000...(.text)|  -->  | 物理页3 |  <---    | 0x4000...(.text)| (共享)
  |  ...         |          |  ...   |          |  ...         |
  +--------------+          +--------+          +--------------+
```

关键机制：
- **页表（Page Table）**：由内核维护，记录每个进程的虚拟地址到物理地址的映射
- **MMU（Memory Management Unit）**：CPU 内部硬件，每次内存访问时自动将虚拟地址转换为物理地址
- **共享内存**：多个进程的虚拟地址可以映射到同一物理页（例如共享库的 `.text` 段），节省内存

> **核心思想**：每个进程都有独立的虚拟地址空间，互不干扰。进程A无法访问进程B的内存（除非通过 IPC 机制）。这种**进程隔离**是现代操作系统的基石。

#### 进程地址空间的概念

进程地址空间 = 一个进程能访问的所有虚拟地址的集合。在 `/proc/<PID>/maps` 中可以看到一个运行中进程的完整地址空间布局：

```bash
# 先让 hello 程序在后台运行（需要添加 sleep）
# 或者用 cat /proc/self/maps 查看当前命令的映射
cat /proc/self/maps
```

---

### 第四部分：程序运行

#### 从 `_start` 到 `main` 的完整路径

```
_start（链接器指定的程序入口）
  +-- __libc_start_main（初始化运行环境）
        |-- 设置栈、环境变量
        |-- 调用 atexit 注册的函数
        |-- 初始化线程本地存储（TLS）
        |-- 初始化信号处理
        |-- 初始化标准 I/O（stdin/stdout/stderr）
        +-- main(argc, argv, envp)  <- 用户程序入口
              +-- printf("hello world\n")
                    +-- exit(0) 或 return 0
                          |-- 调用 atexit 注册的清理函数
                          |-- 刷新并关闭所有 stdio 流（fflush）
                          |-- 调用 _exit 系统调用
                          +-- 内核回收进程资源
```

#### `_start` 入口函数

`_start` 不是 C 函数，而是**汇编编写的入口点**（位于 `crt1.o` 目标文件中）。它的工作是：

1. 从栈上取出 `argc`、`argv`、`envp`，将它们作为参数传给 `__libc_start_main`
2. 调用 `__libc_start_main`

你可以用 `objdump` 查看它：

```bash
objdump -d /usr/lib/x86_64-linux-gnu/crt1.o | grep -A 10 '^<start>:'
```

#### `__libc_start_main` 初始化流程

这是 glibc 中的初始化函数，位于 `csu/libc-start.c`。它负责：

| 步骤 | 工作 |
|------|------|
| 1 | `__cxa_atexit` -- 注册 `main` 返回后的清理处理函数 |
| 2 | `__libc_init_first` -- 早期初始化（线程本地存储等） |
| 3 | `__libc_csu_init` -- 调用 `.init_array` 中的所有初始化函数（全局对象的构造函数等） |
| 4 | 调用 `main(argc, argv, envp)` -- 进入用户程序 |
| 5 | `exit(result)` -- main 返回后调用 exit 做清理 |

#### `exit` 函数的清理工作

`exit()` 不是立即退出，而是做了一系列有序的清理：

```
exit(status)
  |-- atexit 注册的函数被逆序调用
  |-- stdio 缓冲区被刷新（fflush(NULL)）
  |-- 所有打开的流被关闭（fclose）
  |-- 临时文件被删除（tmpfile 创建的）
  +-- _exit(status) 系统调用
        |-- 内核关闭所有文件描述符
        |-- 释放所有内存映射
        |-- 向父进程发送 SIGCHLD 信号
        +-- 进程状态变为僵尸（zombie），等待父进程 wait
```

> **区分**：`exit(0)` 做完整清理（库级别 + 系统级别）。`_exit(0)` 是直接系统调用，跳过库级别的清理（不刷新缓冲区，不调用 atexit 函数）。`return 0` 在 `main` 中等价于 `exit(0)`。

---

## 4. 关键对比：裸机 vs Linux

| 对比项 | 裸机（单片机） | Linux系统 |
|--------|--------------|-----------|
| 开发工具 | IDE（Keil等） | GCC工具链 |
| 编译细节 | IDE封装 | 手动分步操作 |
| 程序入口 | main函数 | _start -> __libc_start_main -> main |
| 硬件访问 | 直接访问 | 通过系统调用 |
| 链接方式 | 静态链接 | 静态/动态可选 |
| 进程管理 | 无 | fork + execve |
| 错误排查 | 简单 | 多层次（编译/链接/运行时） |
| 内存模型 | 单一物理地址空间 | 虚拟内存 + 页表映射 |
| 地址空间 | 无隔离，所有代码共享地址空间 | 每个进程独立4GB（32位）或更大（64位）虚拟地址空间 |
| 进程创建 | 无进程概念，只有任务/中断 | fork 创建进程，COW 优化 |

---

## 5. 动手实验

### 实验1：从源码到可执行文件的分步观察

```bash
# 1. 创建 hello.c
cat > hello.c << 'EOF'
#include <stdio.h>
#define GREETING "hello world"
int main() {
    printf("%s\n", GREETING);
    return 0;
}
EOF

# 2. 预处理：观察宏展开和头文件包含
gcc -E hello.c -o hello.i
echo "预处理后行数："
wc -l hello.i
echo "文件末尾（你的 main 函数）："
tail -20 hello.i

# 3. 编译：生成汇编代码
gcc -S hello.i -o hello.s
echo "汇编代码："
cat hello.s

# 4. 汇编：生成目标文件
gcc -c hello.s -o hello.o

# 5. 链接：生成可执行文件
gcc hello.o -o hello

# 6. 运行
./hello
```

### 实验2：使用分析工具检查可执行文件

```bash
# file：确认文件类型
file hello
file hello.o

# ldd：查看动态库依赖
ldd hello

# readelf：查看 ELF 头部和段信息
readelf -h hello          # ELF 头部
readelf -S hello          # 段（Section）信息
readelf -l hello          # 程序头（运行时需要的段）
readelf -d hello          # 动态链接信息

# objdump：反汇编
objdump -d hello | head -50

# nm：符号表
nm hello.o                # 注意 printf 标记为 U（Undefined）
nm hello                  # 链接后 printf 不再是 U

# size：查看各段大小
size hello
size hello.o

# strings：提取可执行文件中的可打印字符串
strings hello | grep hello
```

### 实验3：静态链接与动态链接对比

```bash
# 动态链接（默认）
gcc hello.c -o hello_dynamic

# 静态链接
gcc -static hello.c -o hello_static

# 对比文件大小
ls -lh hello_dynamic hello_static

# 对比符号数量
echo "动态链接符号数："
nm hello_dynamic | wc -l
echo "静态链接符号数："
nm hello_static | wc -l

# 检查动态链接文件依赖
echo "动态链接依赖："
ldd hello_dynamic
echo "静态链接依赖："
ldd hello_static          # 输出: not a dynamic executable

# 对比运行时的系统调用
echo "动态链接程序系统调用："
strace -c ./hello_dynamic 2>&1
echo "静态链接程序系统调用："
strace -c ./hello_static 2>&1
```

### 实验4：观察进程内存布局

```bash
# 创建一个会在运行时暂停的程序
cat > hello_wait.c << 'EOF'
#include <stdio.h>
#include <unistd.h>
int main() {
    printf("PID: %d\n", getpid());
    printf("Press Enter to exit...\n");
    getchar();
    return 0;
}
EOF

gcc hello_wait.c -o hello_wait

# 在一个终端运行它
./hello_wait &
PID=$!

# 查看该进程的内存映射
cat /proc/$PID/maps

# 查看命令行
cat /proc/$PID/cmdline

# 清理
kill $PID 2>/dev/null
```

---

## 6. 本讲要点总结

1. **HelloWorld 不简单**：最简单的程序背后有最复杂的机制
2. **裸机 vs Linux**：运行机制完全不同，不能用裸机思维理解Linux
3. **四阶段编译**：预处理（文本变换）-> 编译（C到汇编）-> 汇编（汇编到机器码）-> 链接（符号解析+重定位）
4. **ELF 文件格式**：Linux 的标准可执行文件格式，由多个段组成（.text, .data, .bss 等）
5. **进程虚拟地址空间**：每个进程有独立的地址空间，通过页表和 MMU 映射到物理内存
6. **静态 vs 动态链接**：静态大但独立，动态小但需依赖（且可共享库的物理内存）
7. **Shell执行流程**：fork（创建进程）-> execve（替换进程映像）-> 内核加载 ELF -> _start -> __libc_start_main -> main -> exit
8. **学习重点**：先掌握整体流程，动手实验加深理解，不要一头扎进内核源码

> **核心思想**：操作系统为用户做了大量底层工作，这些都是裸机开发难以做到的。理解这些机制，才能写出更高效、更可靠的程序。

---

> **下讲预告**：P30（第29讲-Linux系统HelloWorld(下)）将进行实际操作演示。

---

*笔记生成日期：2026-07-22 | 转写：Whisper tiny | 补充优化日期：2026-07-24*
