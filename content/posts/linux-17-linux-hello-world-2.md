---
title: "第29讲：Linux系统HelloWorld（下）"
date: 2026-09-11T09:10:00+08:00
draft: false
description: "上节课分析的 HelloWorld 执行流程，已整理为 PPT文档，可在野火大学堂获取："
series: ["Linux 入门"]
series_order: 17
categories: ["技术笔记"]
tags: ["Linux", "GCC", "编译"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P30)
> **标题**：第29讲 — Linux系统HelloWorld（下）
> **时长**：14分05秒
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. 前情回顾与资料补充

上节课分析的 HelloWorld 执行流程，已整理为 **PPT文档**，可在野火大学堂获取：

> 野火大学堂 → Linux系列开发 → i.MX6ULL开发 → 基本资料 → i.MX6U Linux开发实战 → **Linux应用编程**章节 → **第5小节**

内容包括：
- 裸机 HelloWorld 程序分析
- Linux 下 HelloWorld 程序分析
- 思维导图
- 程序执行流程图

---

## 2. GCC 四步编译过程演示

### 第一步：预处理（Preprocessing）

```bash
# 将 hello.c 预处理为 hello.i
gcc -E hello.c -o hello.i -v
```

| 说明 | 细节 |
|------|------|
| 选项 | `-E` 只进行预处理 |
| 输出 | `hello.i`（文本文件） |
| 处理内容 | 头文件包含、宏定义展开、条件编译 |
| 实际调用的程序 | **cc1** |

使用 `-v`（verbose）可以看到编译过程的详细信息，包括每一步调用的具体程序。

#### 预处理生成 `.i` 文件的具体变化

预处理后，`.i` 文件会发生以下变化：

**(1) 头文件展开**

以 `#include <stdio.h>` 为例，预处理会把 `stdio.h` 的全部内容插入到 `.i` 文件中。`stdio.h` 本身又包含其他头文件（如 `stddef.h`、`stdarg.h` 等），形成递归展开。一个简单的 hello.c（几行代码）预处理后可能膨胀到几百甚至上千行。

```bash
# 查看原始代码行数
wc -l hello.c
# 输出示例: 6 hello.c

# 查看预处理后行数
wc -l hello.i
# 输出示例: 842 hello.i
```

**(2) 行标记（Line Markers）**

预处理后的 `.i` 文件中会插入以 `#` 开头的行标记，格式如下：

```
# linenum filename flags
```

- `linenum`：原始源文件的行号
- `filename`：原始文件名
- `flags`：标记类型（1=新文件开始，2=返回前一个文件，3=系统头文件，4=视为 extern "C"）

示例：

```c
# 1 "hello.c"       // 从 hello.c 第1行开始
# 1 "<built-in>"    // GCC 内置定义
# 1 "<command-line>"// 命令行定义
# 1 "/usr/include/stdio.h" 1 3 4  // 进入系统头文件
// ... stdio.h 的内容 ...
# 2 "hello.c" 2     // 返回 hello.c 第2行
```

**(3) 宏定义展开**

所有 `#define` 宏在预处理阶段被直接替换为对应的值或代码。

```c
// 预处理前
#define MAX 100
int arr[MAX];

// 预处理后
int arr[100];
```

**(4) 条件编译处理**

`#ifdef`、`#ifndef`、`#if`、`#else`、`#endif` 等条件编译指令被处理，不符合条件的代码块被删除。

**(5) 去除注释**

所有注释（`//` 和 `/* */`）被替换为空格。

#### 使用 `-P` 选项去除行标记

行标记对调试有用，但如果只想看纯粹的预处理结果，可以使用 `-P` 选项：

```bash
# 带行标记（默认）
gcc -E hello.c -o hello_with_markers.i

# 去除行标记
gcc -E -P hello.c -o hello_clean.i
```

对比效果：

```bash
# 带行标记的预处理文件
head -5 hello_with_markers.i
# 输出:
# # 1 "hello.c"
# # 1 "<built-in>"
# ...

# 去除行标记的预处理文件
head -5 hello_clean.i
# 输出: 直接是代码内容，无 # 行标记
```

#### 预处理后的文件大小对比

```bash
# 查看原始文件大小
ls -lh hello.c
# 输出示例: -rw-r--r-- 1 user user  85 Jul 22 10:00 hello.c

# 查看预处理后文件大小
ls -lh hello.i
# 输出示例: -rw-r--r-- 1 user user 23K Jul 22 10:01 hello.i
```

结论：一个几字节的 hello.c，预处理后可能膨胀到几十 KB，因为 `stdio.h` 及其依赖链展开了大量头文件内容。

---

### 第二步：编译（Compilation）

```bash
# 将 hello.i 编译为 hello.s（汇编文件）
gcc -S hello.i -o hello.s -v
```

| 说明 | 细节 |
|------|------|
| 选项 | `-S` 只编译，不汇编 |
| 输出 | `hello.s`（汇编文件） |
| 处理内容 | 词法分析、语法分析 → 生成汇编代码 |
| 实际调用的程序 | **cc1**（与预处理相同） |

> cc1 负责完成预处理和编译两个阶段的工作。

#### 生成的 `.s` 汇编文件内容解读

编译阶段将 C 代码翻译为汇编语言。以 hello.c 为例：

```c
#include <stdio.h>

int main(void) {
    printf("Hello, World!\n");
    return 0;
}
```

生成的 hello.s（**AT&T 语法，默认**）大致如下：

```asm
    .file   "hello.c"
    .text
    .section .rodata
.LC0:
    .string "Hello, World!"
    .text
    .globl  main
    .type   main, @function
main:
    pushq   %rbp              ; 保存栈帧指针
    movq    %rsp, %rbp        ; 设置新的栈帧
    leaq    .LC0(%rip), %rax  ; 加载字符串地址
    movq    %rax, %rdi        ; 第一个参数（字符串地址）
    call    puts@PLT          ; 调用 puts（gcc 优化 printf 为 puts）
    movl    $0, %eax          ; 返回值 0
    popq    %rbp              ; 恢复栈帧
    ret                       ; 返回
    .size   main, .-main
```

**AT&T 语法特点：**
- 源操作数在前，目标操作数在后：`movq %rsp, %rbp` 表示把 `%rsp` 的值复制到 `%rbp`
- 寄存器名前加 `%`：如 `%rax`、`%rbp`
- 立即数前加 `$`：如 `$0`
- 操作数大小后缀：`q`（quad，64位）、`l`（long，32位）、`w`（word，16位）、`b`（byte，8位）

#### AT&T 语法 vs Intel 语法

| 特性 | AT&T 语法（默认） | Intel 语法 |
|------|-----------------|-----------|
| 操作数顺序 | `movq src, dst` | `mov dst, src` |
| 寄存器前缀 | `%rax` | `rax`（无前缀） |
| 立即数前缀 | `$100` | `100`（无前缀） |
| 内存寻址 | `disp(%base, %index, scale)` | `[base + index*scale + disp]` |
| 使用范围 | GCC/GAS 默认，Linux 主流 | Windows 主流，反汇编工具 |

**使用 `-masm=intel` 生成 Intel 风格汇编：**

```bash
gcc -S -masm=intel hello.c -o hello_intel.s
```

Intel 语法版本的 hello.s：

```asm
    .file   "hello.c"
    .intel_syntax noprefix
    .text
    .section .rodata
.LC0:
    .string "Hello, World!"
    .text
    .globl  main
    .type   main, @function
main:
    push    rbp               ; 保存栈帧指针
    mov     rbp, rsp          ; 设置新的栈帧
    lea     rax, .LC0[rip]   ; 加载字符串地址
    mov     rdi, rax          ; 第一个参数
    call    puts@PLT
    mov     eax, 0            ; 返回值 0
    pop     rbp
    ret
    .size   main, .-main
```

#### 不同优化级别生成的汇编差异

GCC 提供多个优化级别，对生成的汇编代码有显著影响：

| 选项 | 说明 | 特点 |
|------|------|------|
| `-O0`（默认） | 无优化 | 代码直观，便于调试，变量都存在栈上 |
| `-O1` | 基本优化 | 减少代码大小和执行时间，不做耗时的优化 |
| `-O2` | 标准优化 | 几乎所有不涉及空间换时间的优化 |
| `-O3` | 激进优化 | 在 O2 基础上增加内联函数、循环展开等 |
| `-Os` | 空间优化 | 在 O2 基础上禁用增加代码大小的优化 |

```bash
# 生成不同优化级别的汇编文件
gcc -S -O0 hello.c -o hello_O0.s
gcc -S -O1 hello.c -o hello_O1.s
gcc -S -O2 hello.c -o hello_O2.s
gcc -S -O3 hello.c -o hello_O3.s

# 比较行数差异
wc -l hello_O*.s
```

**典型差异观察：**

```bash
# 以简单函数为例，不同优化级别的汇编对比
# -O0: 变量都在栈上，每次使用都从内存读写
# -O2: 变量优化到寄存器，减少内存访问
# -O3: 可能进行循环展开、函数内联等激进优化
```

对于 simple hello 程序，由于 GCC 会自动将 `printf` 优化为 `puts`（当只有格式化字符串时），优化级别对代码量的影响不大，但可以从生成的汇编中观察到栈帧管理、寄存器分配策略的变化。

---

### 第三步：汇编（Assembly）

```bash
# 将 hello.s 汇编为 hello.o（可重定位目标文件）
gcc -c hello.s -o hello.o -v
```

| 说明 | 细节 |
|------|------|
| 选项 | `-c` 只汇编，不链接 |
| 输出 | `hello.o`（二进制目标文件） |
| 实际调用的程序 | **as**（GNU汇编器） |

#### `.o` 文件的节区（Section）概念

`.o` 文件是 ELF（Executable and Linkable Format）格式的二进制文件，其内容按 **节区（Section）** 组织：

| 节区名 | 作用 |
|--------|------|
| `.text` | 代码段，存放可执行指令 |
| `.data` | 已初始化的全局/静态变量 |
| `.bss` | 未初始化或初始化为0的全局/静态变量（不占文件空间） |
| `.rodata` | 只读数据，如字符串常量、const 变量 |
| `.comment` | 编译器版本信息 |
| `.symtab` | 符号表（函数名、全局变量名等） |
| `.strtab` | 字符串表（符号名、节区名的字符串） |
| `.rela.text` | 重定位信息（链接时需要修改的地址） |

#### 使用 `objdump -h` 查看节区头部

```bash
objdump -h hello.o
```

典型输出：

```
hello.o:     file format elf64-x86-64

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         0000001e  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000000  0000000000000000  0000000000000000  0000005e  2**0
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  0000000000000000  0000000000000000  0000005e  2**0
                  ALLOC
  3 .rodata       0000000d  0000000000000000  0000000000000000  0000005e  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .comment      0000002c  0000000000000000  0000000000000000  0000006b  2**0
                  CONTENTS, READONLY
  5 .note.GNU-stack 00000000  0000000000000000  0000000000000000  00000097  2**0
                  CONTENTS, READONLY
  6 .eh_frame     00000038  0000000000000000  0000000000000000  00000098  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
```

| 字段 | 含义 |
|------|------|
| `Size` | 节区的大小（十六进制） |
| `VMA` | 虚拟内存地址（链接后才有值，目标文件中为0） |
| `LMA` | 加载内存地址 |
| `File off` | 节区在文件中的偏移 |
| `Algn` | 对齐要求（2的幂） |
| `CONTENTS` | 该节区在文件中有实际内容 |
| `ALLOC` | 运行时需要在内存中分配空间 |
| `LOAD` | 需要从文件加载到内存 |

#### 使用 `size` 命令查看各节区大小

```bash
size hello.o
```

典型输出：

```
   text    data     bss     dec     hex filename
    169       0       0     169      a9 hello.o
```

| 列 | 含义 |
|----|------|
| `text` | 代码段大小（字节），包括 .text 和其他只读段 |
| `data` | 已初始化数据段大小（字节），.data 段 |
| `bss` | 未初始化数据段大小（字节），运行时占内存但不占文件空间 |
| `dec` | 总和（十进制） |
| `hex` | 总和（十六进制） |

#### 使用 `nm` 查看符号表

```bash
nm hello.o
```

典型输出：

```
0000000000000000 T main
                 U puts
```

| 符号类型 | 含义 |
|----------|------|
| `T` | 代码段中的已定义符号（函数） |
| `t` | 代码段中的局部符号 |
| `D` | .data 段中的已定义符号（全局变量） |
| `B` | .bss 段中的已定义符号 |
| `U` | 未定义符号（需要链接时从外部解决） |
| `R` | .rodata 段中的符号（只读数据） |

这里 `main` 是 `T`（已定义的函数），`puts` 是 `U`（未定义，需要从 C 库链接）。

**常用 `nm` 选项：**

```bash
nm -n hello.o      # 按地址排序
nm -u hello.o      # 只显示未定义符号
nm -D /usr/lib/libc.so.6   # 显示共享库的动态符号
```

#### 使用 `objdump -d` 反汇编目标文件

```bash
objdump -d hello.o
```

可以看到 `.o` 文件中机器码和对应的汇编指令，以及尚未被重定位的地址（用 0 占位）。

---

### 第四步：链接（Linking）

```bash
# 将 hello.o 链接为可执行文件 hello
gcc hello.o -o hello -v
```

| 说明 | 细节 |
|------|------|
| 输出 | `hello`（可执行文件） |
| 实际调用的程序 | **collect2**（封装了 **ld** 链接器） |
| 链接内容 | hello.o + crt1.o + crti.o + crtn.o 等 glibc 库文件 |

**关键**：`collect2` 封装了 `ld` 链接器，链接时除了 hello.o 之外，还会链接 glibc 库中的多个目标文件（如 crt1.o、crtn.o 等）。

**程序的真正入口**：`_start` 符号就在 `crt1.o` 文件中。

#### 链接脚本（.ld 文件）的基本概念

链接器 `ld` 通过 **链接脚本（Linker Script）** 来控制输出文件的内存布局。链接脚本决定：

- 各个段（section）在内存中的地址
- 程序的入口点（entry point）
- 代码段、数据段、栈、堆的布局

查看默认链接脚本：

```bash
ld --verbose
```

这个命令会输出 GCC 使用的默认链接脚本，开头类似：

```
GNU ld (GNU Binutils) ...
  .text           :
  {
    *(.text.unlikely .text.*_unlikely .text.unlikely.*)
    *(.text.exit .text.exit.*)
    *(.text.startup .text.startup.*)
    *(.text.hot .text.hot.*)
    *(SORT_BY_NAME(.text.sorted.*))
    *(.text .stub .text.* .gnu.linkonce.t.*)
    ...
  }
```

在嵌入式开发中，经常需要自定义链接脚本来精确控制代码在 FLASH 和 RAM 中的布局。

#### `ld` 链接器的基本用法

虽然通常通过 `gcc` 间接调用链接器，但也可以直接使用 `ld`：

```bash
# 直接用 ld 链接（需要手动指定所有库和启动文件）
ld -dynamic-linker /lib64/ld-linux-x86-64.so.2 \
   /usr/lib/x86_64-linux-gnu/crt1.o \
   /usr/lib/x86_64-linux-gnu/crti.o \
   hello.o \
   -lc \
   /usr/lib/x86_64-linux-gnu/crtn.o \
   -o hello
```

**不建议直接使用 ld**，因为：
- 需要手动指定启动文件路径（crt1.o 等）
- 需要手动指定 C 库的路径
- 需要手动指定动态链接器的路径
- 路径可能因系统不同而变化

推荐始终使用 `gcc` 进行链接，它自动处理上述细节。

#### `-v` 输出中 `collect2` 调用 `ld` 的参数解读

使用 `gcc hello.o -o hello -v` 可以看到 `collect2` 实际调用的 ld 参数。关键参数解读：

```
/usr/lib/gcc/x86_64-linux-gnu/11/collect2 \
  -plugin ...                          # LTO 插件
  -plugin-opt=...                      #
  -m elf_x86_64                        # 目标平台：ELF64 x86-64
  -dynamic-linker                      # 指定动态链接器路径
    /lib64/ld-linux-x86-64.so.2       #
  -o hello                             # 输出文件名
  /usr/lib/x86_64-linux-gnu/crt1.o    # 启动文件1
  /usr/lib/x86_64-linux-gnu/crti.o    # 启动文件2
  /usr/lib/gcc/x86_64-linux-gnu/11/crtbegin.o  # 构造/析构函数支持
  -L/usr/lib/gcc/x86_64-linux-gnu/11  # 库搜索路径
  -L/usr/lib/x86_64-linux-gnu         #
  hello.o                              # 用户的目标文件
  -lgcc                                # GCC 运行时库
  -lgcc_s                              # GCC 共享运行时库
  -lc                                  # 标准C库 (libc)
  -lgcc                                # 再次链接（解决循环依赖）
  -lgcc_s                              #
  /usr/lib/gcc/x86_64-linux-gnu/11/crtend.o    # 构造/析构结束
  /usr/lib/x86_64-linux-gnu/crtn.o    # 启动文件3
```

#### 使用 `-Wl,` 传递参数给链接器

`-Wl,` 是 GCC 的选项，用于将参数原样传递给链接器 `ld`。格式：

```bash
gcc hello.o -o hello -Wl,<ld-option>
```

多个选项用逗号分隔：

```bash
gcc hello.o -o hello -Wl,-Map=output.map,-verbose
```

常用示例：

```bash
# 生成链接 map 文件（查看内存布局）
gcc hello.c -o hello -Wl,-Map=hello.map

# 查看 map 文件
less hello.map
# 包含：段的内存地址分配、符号地址、库文件列表等

# 指定链接脚本
gcc hello.c -o hello -Wl,-T,custom.ld

# 去除未使用的代码段（gc-sections）
gcc -ffunction-sections -fdata-sections hello.c \
    -o hello -Wl,--gc-sections
```

---

## 3. 完整编译过程的命令链

当执行一条完整的命令：

```bash
gcc hello.c -o hello -v
```

GCC 会依次调用（从 verbose 输出可见）：

```
① cc1        → 预处理 + 编译   (hello.c → hello.s)
② as         → 汇编           (hello.s → hello.o)
③ collect2   → 链接           (hello.o + crt*.o → hello)
```

### `-v` 输出的完整解读

执行 `gcc hello.c -o hello -v` 后，输出分为几个关键部分：

**(1) 配置信息**

```
Target: x86_64-linux-gnu
Configured with: ../src/configure ...
Thread model: posix
Supported LTO compression algorithms: zlib zstd
gcc version 11.4.0 (Ubuntu 11.4.0-1ubuntu1~22.04)
```

**(2) 编译器参数（COLLECT_GCC_OPTIONS）**

```
COLLECT_GCC_OPTIONS='-o' 'hello' '-v' '-mtune=generic' '-march=x86-64'
```

**(3) cc1 调用（预处理 + 编译）**

```
/usr/lib/gcc/x86_64-linux-gnu/11/cc1 -quiet -v -imultiarch x86_64-linux-gnu hello.c -quiet -dumpbase hello.c -dumpbase-ext .c -mtune=generic -march=x86-64 -version -o /tmp/ccXXXXXX.s
```

**(4) as 调用（汇编）**

```
as -v --64 -o /tmp/ccXXXXXX.o /tmp/ccXXXXXX.s
```

**(5) collect2 调用（链接）**

```
/usr/lib/gcc/x86_64-linux-gnu/11/collect2 ... -o hello ...
```

注意：在非 verbose 模式下，GCC 使用临时文件处理中间结果，编译完成后自动清理。手动分步执行时需要指定中间文件的输出路径。

---

## 4. 静态链接 vs 动态链接

### 4.1 两种链接方式

| 特性 | 动态链接（默认） | 静态链接 |
|------|----------------|---------|
| 命令 | `gcc hello.c -o hello` | `gcc hello.c -o hello_static -static` |
| 文件大小 | **8.2 KB** | **825 KB**（约100倍） |
| 依赖 | 需目标系统有对应库 | 不依赖外部环境 |
| 链接时机 | 运行时（通过动态链接器） | 编译时 |
| 编译选项中的标识 | `-dynamic-linker` | `-static` |

### 4.2 大小对比

```bash
# 动态链接（默认）
gcc hello.c -o hello
ls -lh hello
# -rwxr-xr-x 1 fire fire 8.2K ... hello

# 静态链接
gcc hello.c -o hello_static -static
ls -lh hello_static
# -rwxr-xr-x 1 fire fire 825K ... hello_static
```

> **动态链接**：8.2 KB（节省空间，依赖环境）
> **静态链接**：825 KB（体积大，独立运行）

### 4.3 动态链接：运行时依赖分析

#### 使用 `ldd` 查看动态链接的可执行文件运行时依赖

```bash
ldd hello
```

典型输出：

```
linux-vdso.so.1 (0x00007ffd4b3f5000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f8a4c000000)
/lib64/ld-linux-x86-64.so.2 (0x00007f8a4c3e0000)
```

| 依赖库 | 说明 |
|--------|------|
| `linux-vdso.so.1` | 虚拟动态共享对象，由内核注入，不占磁盘空间 |
| `libc.so.6` | GNU C 标准库（动态链接版本） |
| `/lib64/ld-linux-x86-64.so.2` | **动态链接器**，负责加载其他共享库 |

**详解各依赖项：**

**(a) `linux-vdso.so.1`（vDSO）：**

- 全称 virtual Dynamic Shared Object
- 由内核自动映射到每个进程的地址空间
- 不占用磁盘空间，不存在于文件系统中
- 加速频繁的系统调用（如 `gettimeofday`、`clock_gettime`），避免从用户态切换到内核态

**(b) `libc.so.6`：**

- GNU C 标准库的共享对象文件
- 包含 `printf`、`malloc`、`read`、`write` 等常用函数

**(c) `ld-linux-x86-64.so.2`：**

- **动态链接器/加载器**（dynamic linker/loader）
- 负责在程序启动时加载所有需要的共享库
- 负责解析符号、重定位
- 每个动态链接的程序在 ELF 头的 `interp` 段中指定了它的路径

**查看 ELF 文件指定的动态链接器：**

```bash
readelf -l hello | grep interpreter
# 输出: [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

#### `LD_LIBRARY_PATH` 环境变量

动态链接器搜索共享库时，按以下顺序查找：

1. **`DT_RPATH`**（ELF 文件中的 rpath，编译时通过 `-Wl,-rpath` 指定）
2. **`LD_LIBRARY_PATH`** 环境变量
3. **`DT_RUNPATH`**（ELF 文件中的 runpath）
4. **`/etc/ld.so.cache`**（由 `ldconfig` 生成）
5. **系统默认路径**：`/lib` 和 `/usr/lib`（64位系统还有 `/lib64`、`/usr/lib64`）

```bash
# 临时添加自定义库搜索路径
export LD_LIBRARY_PATH=/home/user/mylibs:$LD_LIBRARY_PATH
./myprogram

# 查看缓存的库列表
ldconfig -p | grep libcurl

# 更新缓存（需要 root）
sudo ldconfig
```

**常见使用场景：** 自己编译的库安装在非标准路径时，通过 `LD_LIBRARY_PATH` 让动态链接器能找到它们。

### 4.4 静态链接：具体链接了哪些库文件

使用 `-v` 查看静态链接时实际链接的库：

```bash
gcc hello.c -o hello_static -static -v 2>&1 | grep "\.a"
```

静态链接主要链接以下静态库（`.a` 文件）：

| 静态库 | 说明 |
|--------|------|
| `crt1.o` | C 运行时启动（程序入口 `_start`） |
| `crti.o` | C 运行时初始化（调用 `_init`） |
| `crtbeginT.o` | C++ 构造/析构支持（静态链接版本） |
| `libc.a` | C 标准库静态版本 |
| `libgcc.a` / `libgcc_eh.a` | GCC 运行时库 |
| `crtend.o` | 构造/析构结束标记 |
| `crtn.o` | C 运行时清理（调用 `_fini`） |

**`.a` 文件（Archive，归档文件）：** 本质上是多个 `.o` 文件的打包。链接时只提取实际用到的目标文件。

**查看静态库中包含的目标文件：**

```bash
ar t /usr/lib/x86_64-linux-gnu/libc.a | head -20
# 输出示例:
# init-first.o
# libc-start.o
# sysdep.o
# version.o
# ...
```

### 4.5 `.so` 共享对象文件的概念

**【1】什么是 .so 文件**

`.so`（Shared Object）是 Linux 下的动态链接库，等价于 Windows 的 `.dll`。

- 程序运行时按需加载，代码在多个进程间共享（物理内存中只存一份）
- 多个依赖同一 `.so` 的程序共享同一份库代码
- 更新库时无需重新编译所有依赖程序

**【2】so 文件的版本命名**

```bash
# 实际文件
libc.so.6           # 真正的库文件（或符号链接）
libc-2.35.so        # 带版本号的库文件

# 符号链接（编译时使用）
libc.so -> libc.so.6
```

版本号格式：`libname.so.MAJOR.MINOR.PATCH`

- MAJOR：不兼容的 API 变化
- MINOR：向后兼容的新功能
- PATCH：向后兼容的 bug 修复

`soname`（如 `libc.so.6`）是库的运行时标识，嵌入在 ELF 文件中，动态链接器根据它来加载正确的版本。

**【3】创建和使用共享库**

```bash
# 编译为位置无关代码（Position Independent Code）
gcc -fPIC -c mylib.c -o mylib.o

# 创建共享库
gcc -shared -o libmylib.so mylib.o

# 使用共享库
gcc program.c -L. -lmylib -o program

# 运行时需要设置库搜索路径
export LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH
./program
```

---

## 5. 关键启动文件详解

### 5.1 启动文件总览

| 文件 | 作用 |
|------|------|
| `hello.c` | C语言源文件 |
| `hello.i` | 预处理后的文件 |
| `hello.s` | 汇编文件 |
| `hello.o` | 可重定位目标文件（二进制） |
| `hello` | 可执行文件 |
| `crt1.o`、`crti.o`、`crtn.o` | glibc 运行时启动文件，包含 `_start` 入口 |

### 5.2 crt0（C Runtime）的概念

**crt0**（C Runtime Zero）是最早启动的概念，在不同的平台上以不同形式存在：

- 传统 Unix：`crt0.o`，包含最原始的 `_start` 入口
- 现代 Linux/Glibc：拆分为 `crt1.o` + `crti.o` + `crtn.o`，加上 GCC 提供的 `crtbegin.o` 和 `crtend.o`
- 嵌入式裸机：通常需要自己实现 `crt0.S` 或 `startup.s`

**为什么需要 crt0？**

CPU 上电/内核加载程序后，不能直接跳到 `main()`，需要先完成：
1. 设置栈指针（SP）
2. 清零 BSS 段
3. 初始化数据段
4. 调用 C++ 全局构造函数
5. 传递 `argc`、`argv`、`environ`

这些工作由 crt0 系列文件完成。

### 5.3 crt1.o、crti.o、crtn.o 的具体作用

#### crt1.o：入口点和初始设置

**核心内容：**

- 定义 `_start` 符号（程序的真正入口点，ELF 头的 `e_entry` 指向它）
- 设置 `__libc_csu_init` 和 `__libc_csu_fini` 的调用
- 调用 `__libc_start_main` 函数

**关键：** `_start` 不直接调用 `main`，而是调用 `__libc_start_main`，后者是 libc 提供的运行时初始化函数。

源代码位于 glibc 的 `sysdeps/x86_64/start.S`，大致逻辑：

```asm
_start:
    xorl    %ebp, %ebp        ; 标记最外层栈帧
    movq    %rdx, %r9         ; rtld_fini（动态链接器清理函数）
    popq    %rsi              ; argc
    movq    %rsp, %rdx        ; argv
    andq    $-16, %rsp        ; 对齐栈
    pushq   %rax              ; 占位
    pushq   %rsp              ; 栈地址

    ; __libc_csu_init: 初始化函数
    ; __libc_csu_fini: 清理函数
    movq    $__libc_csu_init, %r8
    movq    $__libc_csu_fini, %rcx
    movq    $main, %rdi       ; main 函数地址
    call    __libc_start_main

    hlt                       ; 永远不会执行到这里
```

#### crti.o：初始化/终止的序言

- 定义 `_init` 函数的序言部分（prologue）
- 定义 `_fini` 函数的序言部分

#### crtn.o：初始化/终止的尾声

- 定义 `_init` 函数的尾声部分（epilogue）
- 定义 `_fini` 函数的尾声部分

**crti.o 和 crtn.o 的配合原理：**

链接器将它们分别放在 `.init` 和 `.fini` 节区的开头和结尾，中间插入其他目标文件的 `.init`/`.fini` 节区，拼成一个完整的初始化和清理函数链：

```
crti.o     → _init 的序言
其他 .o   → 各个模块的初始化代码（构造函数）
crtn.o     → _init 的尾声
```

#### crtbegin.o 和 crtend.o（GCC 提供）

- `crtbegin.o`：定义 `__CTOR_LIST__`（构造函数列表）的开始，注册全局析构函数
- `crtend.o`：定义构造函数列表的结束

**静态链接时使用 `crtbeginT.o`（注意后缀 T），动态链接时使用 `crtbeginS.o`。**

### 5.4 Scrt1.o（静态链接的启动文件）

当使用 `-static` 进行静态链接时，使用 `Scrt1.o`（Static C Runtime 1）：

```bash
# 查找 Scrt1.o 位置
find /usr/lib -name "Scrt1.o" 2>/dev/null
# /usr/lib/x86_64-linux-gnu/Scrt1.o
```

**Scrt1.o 与 crt1.o 的区别：**

| 特性 | crt1.o（动态链接） | Scrt1.o（静态链接） |
|------|-------------------|--------------------|
| 动态链接器 | 需要 ld-linux.so | 不需要 |
| 符号解析 | 部分在运行时完成 | 全部在编译时完成 |
| 代码大小 | 较小 | 较大（包含更多静态解析代码） |
| `__libc_start_main` | 来自 libc.so | 来自 libc.a |

### 5.5 `_start` 到 `main` 的完整调用链

```
内核 execve() 系统调用
    |
    v
动态链接器 (ld-linux.so) 或直接跳转（静态链接）
    |
    v
_start (crt1.o / Scrt1.o)
    |
    ├── 设置栈帧、对齐栈
    ├── 准备参数（argc, argv, environ）
    ├── 调用 __libc_csu_init (初始化全局变量、调用 C++ 构造函数)
    v
__libc_start_main (libc)
    |
    ├── 设置线程局部存储（TLS）
    ├── 设置栈保护（stack canary）
    ├── 注册 __libc_csu_fini (通过 atexit)
    ├── 调用 __libc_csu_init (再次确认初始化)
    ├── 调用 main(argc, argv, environ)
    |       |
    |       └── printf("Hello, World!\n")
    |       └── return 0
    |
    └── exit(main的返回值)
        |
        ├── 调用通过 atexit 注册的函数
        ├── 调用 C++ 全局析构函数
        ├── 刷新并关闭所有 stdio 流
        └── _exit() 系统调用 (进程终止)
```

**关键点：**
- `main` 不是程序的第一个执行的函数
- `return 0` 从 `main` 返回时，返回到 `__libc_start_main`
- `__libc_start_main` 将返回值传给 `exit()`，确保 atexit 注册的清理函数被执行
- 如果直接调用 `_exit()` 而非 `exit()`，会跳过清理步骤

### 5.6 查看可执行文件的入口点

```bash
# 方法1: readelf 查看 ELF 头中的入口点
readelf -h hello | grep Entry
# Entry point address: 0x401040

# 方法2: objdump 查看符号
objdump -t hello | grep _start
# 0000000000401040 g     F .text  000000000000002e _start

# 方法3: nm 查看
nm hello | grep _start
# 0000000000401040 T _start
```

---

## 6. 实操实验

### 实验1：使用 `-v` 完整查看编译过程

```bash
# 创建测试程序
cat > hello.c << 'EOF'
#include <stdio.h>
int main(void) {
    printf("Hello, World!\n");
    return 0;
}
EOF

# 完整编译并查看详细信息
gcc hello.c -o hello -v 2>&1 | tee compile_verbose.log

# 关键观察点：
# 1. Target 配置
# 2. cc1 调用的参数（头文件搜索路径等）
# 3. as 调用的参数
# 4. collect2 调用的完整参数链（启动文件、库文件顺序）
```

`-v` 输出中需要注意的关键行：

```
#include "..." search starts here:
#include <...> search starts here:
 /usr/lib/gcc/x86_64-linux-gnu/11/include
 /usr/local/include
 /usr/include/x86_64-linux-gnu
 /usr/include
End of search list.
```

这说明头文件搜索路径按上述顺序依次查找。

### 实验2：分步编译并观察中间文件

```bash
# 第一步：预处理
gcc -E hello.c -o hello.i
echo "=== hello.i 行数和大小 ==="
wc -l hello.i
ls -lh hello.i
echo ""
echo "=== hello.i 前20行（观察行标记） ==="
head -20 hello.i

# 对比：去除行标记的预处理
gcc -E -P hello.c -o hello_clean.i
echo ""
echo "=== 去除行标记后前20行 ==="
head -20 hello_clean.i

# 第二步：编译（生成汇编）
gcc -S hello.i -o hello.s
echo ""
echo "=== hello.s 内容（AT&T 语法） ==="
cat hello.s

# Intel 语法对比
gcc -S -masm=intel hello.c -o hello_intel.s
echo ""
echo "=== hello_intel.s 内容（Intel 语法） ==="
cat hello_intel.s

# 第三步：汇编
gcc -c hello.s -o hello.o
echo ""
echo "=== hello.o 文件类型 ==="
file hello.o

# 第四步：链接
gcc hello.o -o hello
echo ""
echo "=== 可执行文件类型 ==="
file hello
./hello
```

### 实验3：静态链接和动态链接大小对比

```bash
# 动态链接
gcc hello.c -o hello_dynamic

# 静态链接
gcc hello.c -o hello_static -static

# 查看大小
echo "=== 文件大小对比 ==="
ls -lh hello_dynamic hello_static

# 查看段大小
echo ""
echo "=== 动态链接段大小 ==="
size hello_dynamic

echo ""
echo "=== 静态链接段大小 ==="
size hello_static

# 计算倍数
DYNAMIC_SIZE=$(stat -c%s hello_dynamic)
STATIC_SIZE=$(stat -c%s hello_static)
echo ""
echo "静态链接是动态链接的 $((STATIC_SIZE / DYNAMIC_SIZE)) 倍"

# 查看依赖
echo ""
echo "=== 动态链接依赖 ==="
ldd hello_dynamic

echo ""
echo "=== 静态链接依赖（应为 'not a dynamic executable'） ==="
ldd hello_static
```

### 实验4：使用 binutils 工具分析 ELF 文件

```bash
# 4.1 查看 ELF 文件头
echo "=== ELF 文件头 ==="
readelf -h hello

# 4.2 查看节区头部
echo ""
echo "=== 节区头部列表 ==="
readelf -S hello

# 4.3 查看程序头（段信息）
echo ""
echo "=== 程序头 ==="
readelf -l hello

# 4.4 查看符号表
echo ""
echo "=== 符号表（前20个） ==="
readelf -s hello | head -20

# 4.5 查看动态段
echo ""
echo "=== 动态段（动态链接版本） ==="
readelf -d hello

# 4.6 objdump 反汇编 main 函数
echo ""
echo "=== main 函数反汇编 ==="
objdump -d hello | sed -n '/<main>:/,/^$/p'

# 4.7 objdump 查看段信息
echo ""
echo "=== 段信息 ==="
objdump -h hello

# 4.8 nm 查看符号
echo ""
echo "=== 符号扫描 ==="
nm hello | grep -E 'main|puts|printf|_start'

# 4.9 strings 查看嵌入字符串
echo ""
echo "=== 嵌入字符串 ==="
strings hello | grep -i hello

# 4.10 strip 去除符号表（减小文件体积）
cp hello hello_with_symbols
strip hello_with_symbols -o hello_stripped
echo ""
echo "=== strip 前后对比 ==="
ls -lh hello hello_stripped
```

### 实验5：动态链接器细节探索

```bash
# 查看动态链接器路径
echo "=== 程序的动态链接器 ==="
readelf -l hello | grep interpreter

# 查看动态链接库的搜索路径
echo ""
echo "=== 动态库搜索路径 ==="
ldconfig -v 2>/dev/null | head -30

# 查看 ldd 的详细输出
echo ""
echo "=== ldd 详细模式 ==="
ldd -v hello

# 模拟链接器行为：查看会加载哪些库
echo ""
echo "=== LD_DEBUG 追踪动态链接过程 ==="
LD_DEBUG=libs ./hello 2>&1 | head -50
```

### 实验6：不同优化级别的汇编对比

```bash
# 创建一个稍复杂的测试函数
cat > test_opt.c << 'EOF'
#include <stdio.h>

int sum(int n) {
    int result = 0;
    for (int i = 1; i <= n; i++) {
        result += i;
    }
    return result;
}

int main(void) {
    printf("Sum: %d\n", sum(100));
    return 0;
}
EOF

# 生成不同优化级别的汇编
for level in 0 1 2 3 s; do
    gcc -S -O$level test_opt.c -o test_O$level.s
    echo "O$level: $(wc -l < test_O$level.s) lines"
done

# 重点对比 sum 函数的汇编
echo ""
echo "=== O0 的 sum 函数（无优化） ==="
sed -n '/sum:/,/\.cfi_endproc/p' test_O0.s

echo ""
echo "=== O2 的 sum 函数（标准优化） ==="
sed -n '/sum:/,/\.cfi_endproc/p' test_O2.s

echo ""
echo "=== O3 的 sum 函数（激进优化） ==="
sed -n '/sum:/,/\.cfi_endproc/p' test_O3.s
```

### 实验7：制作和使用共享库

```bash
# 创建库源码
cat > mymath.c << 'EOF'
int add(int a, int b) { return a + b; }
int sub(int a, int b) { return a - b; }
EOF

cat > mymath.h << 'EOF'
#ifndef MYMATH_H
#define MYMATH_H
int add(int a, int b);
int sub(int a, int b);
#endif
EOF

cat > use_math.c << 'EOF'
#include <stdio.h>
#include "mymath.h"
int main(void) {
    printf("1 + 2 = %d\n", add(1, 2));
    printf("5 - 3 = %d\n", sub(5, 3));
    return 0;
}
EOF

# 编译为共享库
gcc -fPIC -shared mymath.c -o libmymath.so

# 使用共享库编译程序
gcc use_math.c -L. -lmymath -o use_math

# 运行前设置库路径
export LD_LIBRARY_PATH=.:$LD_LIBRARY_PATH
./use_math

# 对比静态链接
gcc use_math.c mymath.c -o use_math_static
echo ""
echo "=== 动态和静态大小对比 ==="
ls -lh use_math use_math_static
```

---

## 本讲总结

| 阶段 | 命令选项 | 输入 → 输出 | 实际调用 |
|------|---------|-------------|---------|
| 预处理 | `-E` | `.c` → `.i` | cc1 |
| 编译 | `-S` | `.i` → `.s` | cc1 |
| 汇编 | `-c` | `.s` → `.o` | as |
| 链接 | （无） | `.o` → 可执行文件 | collect2 (ld) |
| 一步完成 | 无 | `.c` → 可执行文件 | cc1 → as → collect2 |

### 关键记忆点

1. **`-E` / `-S` / `-c`** 分别停在预处理、编译、汇编之后
2. **`_start` 在 crt1.o 中**，并非用户写的 `main`，程序的真正入口是 `_start`
3. **`collect2` 包装了 `ld`**，自动处理 libc 启动文件
4. 动态链接默认，**体积小但依赖环境**；静态链接用 `-static`，**体积大但独立**
5. 动态链接依赖 `ld-linux.so`，静态链接把所有代码打包进可执行文件
6. 预处理器在 `.i` 文件中插入 **行标记**，可用 `-P` 去除
7. 汇编语法区分 **AT&T（Linux默认）** 和 **Intel（`-masm=intel`）**
8. **`nm`** 看符号表、**`objdump`** 反汇编、**`readelf`** 分析 ELF、**`size`** 看段大小、**`ldd`** 看依赖
9. **`LD_LIBRARY_PATH`** 可以临时指定动态库搜索路径
10. 初始化调用链：**`_start` → `__libc_start_main` → `main` → `exit`**

---

*笔记生成日期：2026-07-22 | 转写：Whisper tiny*
*优化日期：2026-07-24 | 补充：预处理细节、汇编解读、binutils 工具、链接器深入、启动文件详解、实操实验*
