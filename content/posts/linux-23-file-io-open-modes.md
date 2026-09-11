---
title: "第40讲：文件编程与打开模式"
date: 2026-09-11T09:04:00+08:00
draft: false
description: "│  标准IO (C库)  │ <-- fopen/fread/fwrite"
series: ["Linux 入门"]
series_order: 23
categories: ["技术笔记"]
tags: ["Linux", "文件IO"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P41)
> **标题**：第40讲 -- 文件编程与打开模式
> **时长**：约15分钟
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. IO 编程的两大类

### 1.1 分类

| 类别 | 说明 | 调用方式 |
|------|------|---------|
| **系统IO** | 直接调用 Linux 系统调用接口 | `open()` / `read()` / `write()` |
| **标准IO** | 调用 glibc C 库函数（C库再封装系统调用） | `fopen()` / `fread()` / `fwrite()` |

### 1.2 调用链路

```
用户程序
    |
┌───────────────┐
│  标准IO (C库)  │ <-- fopen/fread/fwrite
│  (glibc)       │
└───────┬───────┘
        |
┌───────────────┐
│  系统IO        │ <-- open/read/write
│  (系统调用)     │
└───────┬───────┘
        |
      VFS
        |
   具体文件系统
```

> 两种方式都可以用，标准IO（C库函数）更方便，系统IO更底层灵活。

### 1.3 深入对比：系统 IO vs 标准 IO

理解两者的核心差异是写好文件操作程序的基础。下面的表格从多个维度进行对比：

| 对比维度 | 系统 IO (syscall) | 标准 IO (C库) |
|----------|-------------------|---------------|
| **函数前缀** | 无前缀 / `f` 以外的字母 | `f` 前缀：`fopen`, `fread`, `fwrite` |
| **操作对象** | 文件描述符 `fd`（整数） | 文件指针 `FILE *`（结构体） |
| **缓冲区** | 无用户态缓冲区，每次调用直接陷入内核 | 有用户态缓冲区（默认 8192 字节），减少系统调用次数 |
| **每次读写开销** | 较大（每次都是系统调用，用户态/内核态切换） | 较小（攒够数据才发一次系统调用） |
| **实时性** | 高，写完立即落盘（取决于 `O_SYNC`） | 低，数据可能滞留在用户态缓冲区 |
| **适用场景** | 网络编程、驱动开发、实时性要求高的场景 | 普通文件读写、日志输出、配置文件处理 |
| **可移植性** | POSIX 标准，Unix/Linux 专属 | ANSI C 标准，跨平台（Windows 也用） |
| **错误处理** | 返回 `-1`，通过 `errno` 获取错误码 | 返回 `NULL` 或 `EOF`，通过 `ferror()` 判断 |
| **典型函数** | `open`, `read`, `write`, `lseek`, `close` | `fopen`, `fread`, `fwrite`, `fseek`, `fclose` |

**一句话总结**：标准IO在系统IO之上加了一层**用户态缓冲区**，目的是减少系统调用次数，适合大多数日常文件操作；系统IO没有这层缓冲，更适合需要精确控制写入时机或操作非普通文件（设备、管道、套接字）的场景。

### 1.4 标准 IO 的缓冲区机制

标准IO为什么比系统IO"方便"？核心就在于它内置的缓冲区。下面用一个具体例子说明：

```c
// 系统IO版本：每次 write() 都是系统调用
write(fd, "H", 1);    // 系统调用 #1
write(fd, "e", 1);    // 系统调用 #2
write(fd, "l", 1);    // 系统调用 #3
write(fd, "l", 1);    // 系统调用 #4
write(fd, "o", 1);    // 系统调用 #5
// 总共 5 次系统调用，每次都有用户态/内核态切换开销

// 标准IO版本：数据先攒在用户态缓冲区
fputc('H', fp);       // 写入用户态缓冲区
fputc('e', fp);       // 写入用户态缓冲区
fputc('l', fp);       // 写入用户态缓冲区
fputc('l', fp);       // 写入用户态缓冲区
fputc('o', fp);       // 写入用户态缓冲区
fflush(fp);           // 一次性触发系统调用，把缓冲区内所有内容写入内核
// 总共 1 次系统调用（fflush 触发）
```

标准IO的三种缓冲策略：

| 缓冲类型 | 触发写入的条件 | 典型场景 |
|----------|--------------|---------|
| **全缓冲** | 缓冲区满时触发系统调用 | 普通文件读写（默认） |
| **行缓冲** | 遇到换行符 `\n` 时触发 | 终端输入输出（stdin/stdout） |
| **无缓冲** | 每次调用立即触发系统调用 | stderr |

可以使用 `setvbuf()` 函数手动修改缓冲策略：

```c
#include <stdio.h>

char mybuf[1024];
// 将 fp 设置为全缓冲，缓冲区大小为 1024 字节
setvbuf(fp, mybuf, _IOFBF, sizeof(mybuf));
```

### 1.5 什么时候用哪种 IO？

决策流程：

```
你需要写入的数据是否需要"立即"到达磁盘/设备？
    |
    ├── 是 → 用系统 IO（或标准IO + fflush + fsync）
    |
    └── 否 → 是否需要跨平台兼容？
              |
              ├── 是 → 用标准 IO（ANSI C 可移植）
              |
              └── 否 → 操作的是普通文件还是特殊文件？
                        |
                        ├── 普通文件 → 标准 IO（更高效）
                        |
                        └── 设备/管道/套接字 → 系统 IO（更直接）
```

典型场景举例：

| 场景 | 推荐方式 | 原因 |
|------|---------|------|
| 读写配置文件 | 标准 IO | 数据量小，跨平台方便 |
| 日志写入 | 标准 IO + 行缓冲 | 每条日志一行，行缓冲天然匹配 |
| 网络编程（socket） | 系统 IO | socket 是 fd，不能用 FILE*（除非 fdopen） |
| 驱动开发 | 系统 IO | 需要精确控制，无缓冲区干扰 |
| 大文件拷贝 | 系统 IO + 大块读写 | 避免用户态缓冲区二次拷贝 |
| 数据库文件写入 | 系统 IO + O_SYNC | 需要保证写入顺序和持久性 |

---

## 2. 系统 IO 的 5 个核心函数

### 2.1 `open()` -- 打开文件

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);
```

> **注意**：`open()` 是变参函数。当 `flags` 中包含 `O_CREAT` 时，**必须**提供第三个参数 `mode` 来指定新文件的权限；没有 `O_CREAT` 时，第三个参数可以省略。

| 参数 | 说明 |
|------|------|
| `pathname` | 要打开的文件名（路径） |
| `flags` | 打开模式（主模式 | 辅模式） |
| `mode` | 可选，创建文件时的权限（仅当 flags 包含 O_CREAT 时有效） |

**返回值**：成功返回文件描述符（`fd`，非负整数），失败返回 `-1` 并设置 `errno`。

#### 错误处理示例

```c
#include <fcntl.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>

int fd = open("test.txt", O_RDONLY);
if (fd == -1) {
    // errno 是全局变量，记录最后一次系统调用的错误码
    // strerror() 把错误码转换为可读字符串
    fprintf(stderr, "open() failed: %s\n", strerror(errno));
    return 1;
}
```

常见 `errno` 值：

| errno | 含义 |
|-------|------|
| `ENOENT` | 文件不存在 |
| `EACCES` | 权限不足 |
| `EISDIR` | 路径是目录而非文件 |
| `EMFILE` | 进程打开的文件数已达上限 |
| `ENFILE` | 系统打开的文件数已达上限 |
| `ENOSPC` | 磁盘空间不足（创建文件时） |

### 2.2 `lseek()` -- 设置读写位置

```c
#include <unistd.h>

off_t lseek(int fd, off_t offset, int whence);
```

| 参数 | 说明 |
|------|------|
| `fd` | 文件描述符 |
| `offset` | 偏移量（可为负，取决于 whence） |
| `whence` | 基准位置：`SEEK_SET`（文件开头）、`SEEK_CUR`（当前位置）、`SEEK_END`（文件末尾） |

**返回值**：成功返回新的文件偏移量（从文件开头算起的字节数），失败返回 `-1`。

三种 whence 的 offset 含义：

| whence | offset 含义 |
|--------|------------|
| `SEEK_SET` | offset 必须 >= 0，从文件开头偏移 offset 字节 |
| `SEEK_CUR` | offset 可为正或负，从当前位置移动 offset 字节 |
| `SEEK_END` | offset 可为正或负，从文件末尾移动 offset 字节 |

#### 常用技巧

```c
// 获取当前文件位置
off_t cur_pos = lseek(fd, 0, SEEK_CUR);

// 获取文件大小（移到末尾，返回值就是大小）
off_t file_size = lseek(fd, 0, SEEK_END);

// 回到文件开头
lseek(fd, 0, SEEK_SET);
```

> **警告**：`lseek()` 只对普通文件、块设备等"可定位"的文件有效。对管道（pipe）、套接字（socket）、终端设备使用 `lseek()` 会返回 `-1` 并设置 `errno` 为 `ESPIPE`。

### 2.3 `write()` -- 写入文件

```c
#include <unistd.h>

ssize_t write(int fd, const void *buf, size_t count);
```

| 参数 | 说明 |
|------|------|
| `fd` | 文件描述符 |
| `buf` | 要写入的内容缓冲区 |
| `count` | 要写入的字节数 |

**返回值**：成功返回实际写入的字节数（可能小于 `count`），失败返回 `-1`。

> **关键注意**：`write()` 的返回值**不一定等于** `count`！磁盘满、信号中断、管道缓冲区满都可能导致"部分写入"。健壮的代码必须处理这种情况：

```c
// 健壮的写入：确保所有数据都写入
ssize_t write_all(int fd, const void *buf, size_t count) {
    const char *p = (const char *)buf;
    size_t remaining = count;

    while (remaining > 0) {
        ssize_t n = write(fd, p, remaining);
        if (n == -1) {
            if (errno == EINTR) {
                // 被信号中断，重试
                continue;
            }
            return -1;  // 真正的错误
        }
        p += n;
        remaining -= n;
    }
    return count;
}
```

### 2.4 `read()` -- 读取文件

```c
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t count);
```

| 参数 | 说明 |
|------|------|
| `fd` | 文件描述符 |
| `buf` | 存放读取内容的缓冲区 |
| `count` | 要读取的字节数（缓冲区大小） |

**返回值**：
- `> 0`：实际读取的字节数（可能小于 `count`，表示到达文件末尾或数据尚不够）
- `0`：文件末尾（EOF）
- `-1`：出错

> **重要**：和 `write()` 一样，`read()` 也可能返回小于 `count` 的值。不要假设一次 `read()` 就能读到所有想要的数据。

### 2.5 `close()` -- 关闭文件

```c
#include <unistd.h>

int close(int fd);
```

**返回值**：成功返回 `0`，失败返回 `-1`。

> **为什么一定要 `close()`？** 每个进程能打开的文件描述符数量有上限（用 `ulimit -n` 查看，默认通常 1024）。忘记 `close()` 会导致文件描述符泄漏，最终耗尽 fd 导致 `open()` 失败。

健壮的关闭写法：

```c
// 即使 close() 失败也应该记录日志（可能是 NFS 延迟写入失败等原因）
if (close(fd) == -1) {
    perror("close failed");
}
```

### 2.6 典型代码流程

```c
#include <fcntl.h>
#include <unistd.h>

int main() {
    // 1. 打开文件
    int fd = open("test.txt", O_RDWR | O_CREAT, 0644);

    // 2. 定位到文件开头
    lseek(fd, 0, SEEK_SET);

    // 3. 写入数据
    write(fd, "Hello World", 11);

    // 4. 读取文件
    char buf[128];
    lseek(fd, 0, SEEK_SET);  // 回到开头再读
    read(fd, buf, 128);

    // 5. 关闭文件
    close(fd);
    return 0;
}
```

### 2.7 完整可编译示例：带错误处理的文件读写

以下是一个可以在 Linux 上直接编译运行的完整程序：

```c
/*
 * file_demo.c -- 演示系统 IO 五个核心函数的完整用法（含错误处理）
 * 编译：gcc -Wall -o file_demo file_demo.c
 * 运行：./file_demo
 */

#include <stdio.h>      // perror
#include <stdlib.h>     // exit
#include <string.h>     // strlen
#include <errno.h>      // errno
#include <fcntl.h>      // open, O_* 常量
#include <unistd.h>     // read, write, lseek, close

int main(void)
{
    int fd;
    ssize_t n;
    off_t file_size;
    char buf[256];

    /* 步骤1：打开（或创建）文件，读写模式 */
    fd = open("demo.txt", O_RDWR | O_CREAT | O_TRUNC, 0644);
    if (fd == -1) {
        perror("open failed");
        exit(EXIT_FAILURE);
    }
    printf("[INFO] fd = %d\n", fd);

    /* 步骤2：写入数据 */
    const char *msg = "Hello, Linux System IO!\n";
    n = write(fd, msg, strlen(msg));
    if (n == -1) {
        perror("write failed");
        close(fd);
        exit(EXIT_FAILURE);
    }
    printf("[INFO] wrote %zd bytes\n", n);

    /* 步骤3：查看当前文件位置 */
    file_size = lseek(fd, 0, SEEK_CUR);
    if (file_size == -1) {
        perror("lseek (SEEK_CUR) failed");
    } else {
        printf("[INFO] current position: %ld\n", (long)file_size);
    }

    /* 步骤4：回到文件开头，读取刚写入的内容 */
    if (lseek(fd, 0, SEEK_SET) == -1) {
        perror("lseek (SEEK_SET) failed");
        close(fd);
        exit(EXIT_FAILURE);
    }

    n = read(fd, buf, sizeof(buf) - 1);
    if (n == -1) {
        perror("read failed");
        close(fd);
        exit(EXIT_FAILURE);
    }
    buf[n] = '\0';  // 手动补上字符串结束符
    printf("[INFO] read %zd bytes: %s", n, buf);

    /* 步骤5：关闭文件 */
    if (close(fd) == -1) {
        perror("close failed");
        exit(EXIT_FAILURE);
    }
    printf("[INFO] file closed\n");

    return 0;
}
```

**编译和运行**：

```bash
gcc -Wall -o file_demo file_demo.c
./file_demo
# 输出示例：
# [INFO] fd = 3
# [INFO] wrote 24 bytes
# [INFO] current position: 24
# [INFO] read 24 bytes: Hello, Linux System IO!
# [INFO] file closed
```

> **观察**：fd = 3，因为 0/1/2 已经被 stdin/stdout/stderr 占用了。

### 2.8 实战示例：用系统 IO 实现文件复制

这是一个更贴近实际用途的程序：用系统 IO 实现 `cp` 命令的简化版。

```c
/*
 * mycp.c -- 用系统 IO 实现文件复制
 * 编译：gcc -Wall -o mycp mycp.c
 * 用法：./mycp <源文件> <目标文件>
 */

#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>

#define BUFSIZE 4096   // 一次读写 4KB，兼顾效率和内存

int main(int argc, char *argv[])
{
    int fd_src, fd_dst;
    ssize_t nread, nwrite;
    char buf[BUFSIZE];
    off_t total = 0;

    if (argc != 3) {
        fprintf(stderr, "Usage: %s <src> <dst>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    /* 打开源文件（只读） */
    fd_src = open(argv[1], O_RDONLY);
    if (fd_src == -1) {
        fprintf(stderr, "Cannot open %s: %s\n", argv[1], strerror(errno));
        exit(EXIT_FAILURE);
    }

    /* 打开/创建目标文件（只写，截断或创建） */
    fd_dst = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd_dst == -1) {
        fprintf(stderr, "Cannot create %s: %s\n", argv[2], strerror(errno));
        close(fd_src);
        exit(EXIT_FAILURE);
    }

    /* 循环读写，直到源文件读完 */
    while ((nread = read(fd_src, buf, BUFSIZE)) > 0) {
        nwrite = write(fd_dst, buf, nread);
        if (nwrite != nread) {
            fprintf(stderr, "Write error: %s\n", strerror(errno));
            close(fd_src);
            close(fd_dst);
            exit(EXIT_FAILURE);
        }
        total += nwrite;
    }

    if (nread == -1) {
        perror("read error");
    }

    printf("Copied %ld bytes from %s to %s\n", (long)total, argv[1], argv[2]);

    close(fd_src);
    close(fd_dst);
    return 0;
}
```

---

## 3. 文件描述符（fd）的本质

### 3.1 核心问题

`open()` 返回的**文件描述符**（一个小整数）到底是什么？

**答案**：它实际上是一个数组的**下标**。

### 3.2 进程文件管理结构

```
┌─────────────────────────────────┐
│       进程 (task_struct)         │
│  ┌───────────────────────────┐  │
│  │ files (指向files_struct)   │  │
│  └──────────┬────────────────┘  │
└─────────────┼───────────────────┘
              |
┌─────────────────────────────────┐
│       files_struct              │
│  ┌───────────────────────────┐  │
│  │ fd_array[]                │  │
│  │                           │  │
│  │ [0] ---> stdin            │  │
│  │ [1] ---> stdout           │  │
│  │ [2] ---> stderr           │  │
│  │ [3] ---> file对象 (你打开的文件)│  │
│  │ [4] ---> ...              │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

### 3.3 文件描述符的完整映射链

```
open("test.txt", O_RDWR)
    |
VFS 找到文件 -> 获取 file_operations 操作接口
    |
进程 files_struct 的 fd_array[]
找到空闲位置（比如下标3）
    |
创建 file 对象，填充到 fd_array[3]
包含：file_operations、权限、模式、读写位置、路径...
    |
返回 3（文件描述符）
```

### 3.4 `file` 对象包含的内容

| 字段 | 说明 |
|------|------|
| `f_op` | 文件操作接口集合（file_operations） |
| `f_flags` | 打开文件的权限和模式 |
| `f_pos` | 当前读写位置（偏移量） |
| `f_path` | 文件的路径信息 |
| ... | 其他属性 |

### 3.5 为什么用文件描述符？

- 其他函数（`read`/`write`/`lseek`/`close`）只需要传 `fd`（一个整数）
- 内核通过 `fd` 找到对应的 `file` 对象
- 再从 `file` 对象中找到操作接口，执行具体操作

> **文件描述符是一个轻量级的句柄**，对用户程序来说只是一个整数，但内核用它索引到完整的文件管理结构。

### 3.6 类比：酒店房卡

如果觉得文件描述符的概念抽象，可以用一个生活中的类比来理解：

| 现实世界 | Linux 内核 |
|----------|-----------|
| 你去酒店前台说"我要入住" | 程序调用 `open("guest.txt", O_RDWR)` |
| 前台查房，找到一间空房（302号） | 内核在 `fd_array[]` 中找空闲下标（比如3） |
| 给你一张房卡，上面写着"302" | 返回文件描述符 `3` |
| 你用房卡开门、放行李、退房 | 用 `fd=3` 调用 `read`/`write`/`close` |
| 房卡本身只是一个数字，但背后关联着整个房间 | `fd` 是一个整数，但背后关联着 `file` 对象 |
| 退房后，房卡失效，302号可被其他人使用 | `close(3)` 后，`fd=3` 释放，可被下一个 `open()` 复用 |
| 一个酒店最多有 N 间房 | 一个进程默认最多打开 1024 个文件 |

**核心思想**：`fd` 只是一个"索引编号"，真正的资源（文件内容、读写位置、操作函数指针）都在内核的 `file` 对象中。这就是为什么系统 IO 的所有函数都接受 `fd` 作为第一个参数 -- 内核需要这个数字来查找对应资源。

### 3.7 文件描述符的数量限制

每个进程能打开的文件数量是有限的，检查当前限制：

```bash
# 查看软限制（当前有效值）
ulimit -n
# 通常输出: 1024

# 查看硬限制（最大值，需要 root 才能调）
ulimit -Hn
# 通常输出: 4096 或 1048576

# 临时修改软限制（当前 shell 进程生效）
ulimit -n 2048
```

除此之外，还有系统级限制：

```bash
# 查看系统范围内所有进程打开的文件总数上限
cat /proc/sys/fs/file-max
# 通常输出: 100000 或更大

# 查看当前系统已打开的文件总数
cat /proc/sys/fs/file-nr
# 输出三个数字：已分配 / 已使用 / 上限
```

> **常见故障**：服务进程在长期运行中忘记 `close()`，导致 fd 逐渐泄漏，最终 `open()` 返回 `-1` 且 `errno=EMFILE`。这类问题用 `lsof -p <pid>` 可以快速诊断。

### 3.8 `file_operations` 结构体详解

每个 `file` 对象的 `f_op` 字段指向一个 `file_operations` 结构体，里面是一组**函数指针**：

```c
// 简化版 file_operations（内核源码中的真实结构体字段更多）
struct file_operations {
    struct module *owner;
    loff_t (*llseek)   (struct file *, loff_t, int);
    ssize_t (*read)    (struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write)   (struct file *, const char __user *, size_t, loff_t *);
    int     (*open)    (struct inode *, struct file *);
    int     (*release) (struct inode *, struct file *);
    // ... 还有 mmap, ioctl, poll, fsync 等
};
```

**关键理解**：不同类型的文件（ext4 文件、FAT32 文件、设备文件、socket 文件）有不同的 `file_operations` 实现。当你调用 `read(fd, ...)` 时：

```
用户空间: read(fd, buf, 100)
    |   系统调用
    v
内核空间: 通过 fd 找到 file 对象
    |
    v
取出 file->f_op->read 函数指针
    |
    v
调用对应的具体实现
    ├── 普通文件 -> ext4_read() 或 xfs_read()
    ├── 设备文件 -> device_read()
    └── socket   -> sock_read()
```

这就是 Linux "一切皆文件"哲学的底层支撑：**同一套系统调用接口**（`open`/`read`/`write`/`close`），通过 `file_operations` 中的不同函数指针，优雅地实现了**多态**。

---

## 4. 文件打开模式

### 4.1 主模式（三类）

| 宏 | 值 | 说明 |
|----|-----|------|
| `O_RDONLY` | 0 | 只读打开 |
| `O_WRONLY` | 1 | 只写打开 |
| `O_RDWR` | 2 | 读写打开 |

> 主模式**三选一**，不能同时使用。

### 4.2 辅模式（常用5个）

| 宏 | 说明 |
|----|------|
| `O_CREAT` | 文件不存在时**创建**它（需要第三个参数 mode 指定权限） |
| `O_APPEND` | **追加**模式，每次写都在文件末尾 |
| `O_TRUNC` | **截断**模式，打开时清空文件内容 |
| `O_SYNC` | **同步**模式（等待数据真正写入磁盘） |
| `O_NONBLOCK` | **非阻塞**模式（读写不等待，立即返回） |

> 辅模式可以**多个组合使用**，用 `|` 连接。

#### 4.2.1 O_CREAT 详解

`O_CREAT` 告诉内核：**如果文件不存在，就创建它；如果文件存在，不报错**。创建新文件时必须指定权限：

```c
// 第三个参数 0644 表示：所有者可读写，组用户只读，其他用户只读
int fd = open("newfile.txt", O_WRONLY | O_CREAT, 0644);
```

**`mode` 的实际生效权限**：`mode & ~umask`。例如 `mode=0666`，`umask=0022`，实际权限为 `0666 & ~0022 = 0644`。

```bash
# 查看当前 umask
umask
# 通常输出: 0022

# 在 C 程序中获取 umask
```

```c
#include <sys/stat.h>
mode_t old_umask = umask(0);   // 获取当前 umask 并临时设为 0
// ... 创建文件 ...
umask(old_umask);              // 恢复
```

**常见错误：忘记第三个参数**。以下代码是**错误**的：

```c
// 错误：flags 中有 O_CREAT 但没有提供 mode 参数
int fd = open("file.txt", O_WRONLY | O_CREAT);
// 编译器可能不报错（因为 open 是变参函数），
// 但新文件的权限是栈上的随机值，行为不可预测
```

正确写法：

```c
int fd = open("file.txt", O_WRONLY | O_CREAT, 0644);
```

#### 4.2.2 O_APPEND 详解

`O_APPEND` 保证每次 `write()` 的数据都追加到文件**末尾**，即使其他进程也在同时写入该文件。这是**原子操作**。

```c
// 追加模式：每次写都在末尾，不会覆盖已有内容
int fd = open("log.txt", O_WRONLY | O_CREAT | O_APPEND, 0644);
write(fd, "line 1\n", 7);
write(fd, "line 2\n", 7);
// log.txt 内容：
// line 1
// line 2
```

**O_APPEND vs lseek+write 的区别**：

```c
// 方法A：手动 lseek 到末尾再写（非原子，多进程下不安全）
lseek(fd, 0, SEEK_END);   // 步骤1
write(fd, data, len);      // 步骤2
// 风险：步骤1和步骤2之间，另一个进程可能也写了数据，导致覆盖

// 方法B：O_APPEND（原子操作，多进程安全）
// 打开时设置 O_APPEND
// 每次 write() 前，内核自动把 f_pos 设为文件末尾，
// 然后原子地完成"定位+写入"
```

**日志场景**：几乎所有日志写入程序都会使用 `O_APPEND`，确保多进程写同一日志文件时不会互相覆盖。

#### 4.2.3 O_TRUNC 详解

`O_TRUNC` 在打开文件时**立即清空**文件内容（文件长度变为 0），同时保留文件本身的 inode 和权限。

```c
// 打开文件并清空，然后写入新内容
int fd = open("data.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
write(fd, "fresh data", 10);
// data.txt 现在只有 "fresh data"，旧内容全部丢失
```

**注意**：
- `O_TRUNC` 只有在**写权限**打开时才有意义。`O_RDONLY | O_TRUNC` 虽然语法合法但不同系统行为可能不同。
- 如果只想创建新文件而不覆盖已有文件，应该用 `O_CREAT | O_EXCL`（见 4.4.1 节）。

#### 4.2.4 O_SYNC 简介

`O_SYNC` 使每次 `write()` 都等待数据**真正写入磁盘**后才返回。没有 `O_SYNC` 时，数据可能先写入内核的页缓存（page cache）就返回了，实际落盘由内核延迟处理。

| 场景 | 是否设置 O_SYNC |
|------|----------------|
| 数据库 WAL 日志 | 是（数据持久性要求极高） |
| 普通日志文件 | 否（性能优先，允许少量丢失） |
| 临时文件 | 否（进程退出后文件也不需要了） |

#### 4.2.5 O_NONBLOCK 简介

`O_NONBLOCK` 使读写操作**不阻塞**。如果没有数据可读，`read()` 立即返回 `-1` 并设 `errno=EAGAIN`，而不是一直等待。

典型应用场景：
- 管道/FIFO：以非阻塞方式打开，即使另一端还未连接
- 设备文件：读取串口数据时不希望进程卡住
- 网络 socket：非阻塞 IO 是实现高性能网络服务的基础

### 4.3 常见组合示例

```c
// 只读打开
int fd1 = open("file.txt", O_RDONLY);

// 读写打开，不存在则创建，权限 0644
int fd2 = open("file.txt", O_RDWR | O_CREAT, 0644);

// 读写打开，清空再写
int fd3 = open("file.txt", O_RDWR | O_TRUNC);

// 追加模式写入
int fd4 = open("file.txt", O_WRONLY | O_APPEND | O_CREAT, 0644);
```

### 4.4 其他重要辅模式

#### 4.4.1 O_EXCL -- 排他创建

`O_EXCL` 与 `O_CREAT` 配合使用：**如果文件已存在，则 open() 失败并返回 -1（errno=EEXIST）**。这是一个原子操作，常用于实现锁文件（lock file）。

```c
// 原子地检查"文件不存在"并"创建文件"
// 如果文件已存在，open() 返回 -1
int fd = open("myapp.lock", O_WRONLY | O_CREAT | O_EXCL, 0644);
if (fd == -1) {
    if (errno == EEXIST) {
        fprintf(stderr, "Another instance is already running.\n");
        exit(1);
    }
    perror("open");
    exit(1);
}
// 成功获得锁，继续执行...
// 进程退出时，可以 unlink("myapp.lock") 释放锁
```

**为什么用 O_EXCL 而不是先 stat() 再 open()？** 因为 `stat() + open()` 是两步操作，中间存在竞态条件：

```
时刻 T1: 进程A 调用 stat("lock") -> 文件不存在
时刻 T2: 进程B 调用 stat("lock") -> 文件不存在
时刻 T3: 进程A 调用 open("lock", O_CREAT) -> 创建成功
时刻 T4: 进程B 调用 open("lock", O_CREAT) -> 也创建成功！
结果：两个进程都认为自己获得了锁
```

而 `O_CREAT | O_EXCL` 在内核中**原子执行**，彻底消除竞态条件。

#### 4.4.2 O_CLOEXEC -- exec 时自动关闭

```c
int fd = open("secret.txt", O_RDONLY | O_CLOEXEC);
```

设置了 `O_CLOEXEC` 的文件描述符会在进程调用 `exec()` 系列函数（执行另一个程序）时自动关闭。这是**安全实践**：防止敏感文件的 fd 被无意中传递给子程序。

```c
// 不安全：fd 会被 exec 后的程序继承
int fd = open("passwords.txt", O_RDONLY);
execl("/usr/bin/some_program", "some_program", NULL);
// some_program 仍然可以读 passwords.txt！

// 安全：fd 在 exec 时自动关闭
int fd = open("passwords.txt", O_RDONLY | O_CLOEXEC);
execl("/usr/bin/some_program", "some_program", NULL);
// some_program 无法访问 passwords.txt
```

### 4.5 实战场景分析

下面按使用场景分类，展示不同的 flag 组合。

#### 场景1：读取配置文件

```c
// 只读，文件必须存在，不存在则报错
int fd = open("/etc/myapp.conf", O_RDONLY);
if (fd == -1) {
    perror("Cannot open config file");
    exit(1);
}
// 读取配置...
close(fd);
```

#### 场景2：写入日志文件

```c
// 只写，追加，不存在则创建；多进程安全
int log_fd = open("/var/log/myapp.log",
                   O_WRONLY | O_CREAT | O_APPEND,
                   0644);
// 每次 write 自动追加到末尾
dprintf(log_fd, "[%s] Service started\n", get_timestamp());
```

#### 场景3：覆盖写入数据文件

```c
// 读写，清空旧内容，不存在则创建
int fd = open("data.bin", O_RDWR | O_CREAT | O_TRUNC, 0644);
// 旧数据被清空，从头写入新数据
write(fd, new_data, data_len);
close(fd);
```

#### 场景4：实现单实例锁

```c
// 利用 O_EXCL 实现进程互斥
int lock_fd = open("/var/run/myapp.pid",
                   O_WRONLY | O_CREAT | O_EXCL, 0644);
if (lock_fd == -1) {
    fprintf(stderr, "myapp is already running\n");
    exit(1);
}
// 写入 PID
dprintf(lock_fd, "%d\n", getpid());
close(lock_fd);
// ... 程序主逻辑 ...
unlink("/var/run/myapp.pid");  // 退出时释放锁
```

#### 场景5：数据库写入

```c
// 需要保证数据真正落盘
int fd = open("database.db", O_WRONLY | O_CREAT | O_SYNC, 0600);
// 每次 write() 都会等待数据写入磁盘后才返回
write(fd, commit_record, record_len);
// 此时数据已经安全地在磁盘上了
close(fd);
```

#### 场景6：临时文件（安全创建）

```c
// 临时文件只需要短期存在，不需要持久化
int fd = open("/tmp/myapp_XXXXXX", O_RDWR | O_CREAT | O_EXCL, 0600);
// 临时读写...
close(fd);
unlink("/tmp/myapp_XXXXXX");  // 删除临时文件
```

---

## 5. 常见错误与调试技巧

### 5.1 忘记检查 open() 的返回值

```c
// 危险写法：不检查返回值，后续操作可能对 fd=-1 操作
int fd = open("file.txt", O_RDONLY);
read(fd, buf, 100);      // 如果 open 失败，read(-1, ...) 会出错
close(fd);               // close(-1) 也是无效操作

// 正确写法
int fd = open("file.txt", O_RDONLY);
if (fd == -1) {
    perror("open");
    return 1;
}
```

### 5.2 open() 有 O_CREAT 但忘记第三个参数

这是新手最常见的错误之一。`open()` 是变参函数，编译器不会报错，但运行时会读取栈上随机值作为文件权限。

```c
// 错误
int fd = open("new.txt", O_CREAT | O_WRONLY);  // 缺少 mode！

// 正确
int fd = open("new.txt", O_CREAT | O_WRONLY, 0644);
```

### 5.3 文件描述符泄漏

```c
// 泄漏示例：错误路径上没有 close()
void process_file(const char *path) {
    int fd = open(path, O_RDONLY);
    if (fd == -1) return;

    char *buf = malloc(4096);
    if (buf == NULL) return;   // fd 未关闭！泄漏！
    // ... 处理 ...
    free(buf);
    close(fd);
}
```

修复方法：使用 `goto` 统一清理：

```c
void process_file(const char *path) {
    int fd = open(path, O_RDONLY);
    if (fd == -1) return;

    char *buf = malloc(4096);
    if (buf == NULL) goto cleanup_fd;

    // ... 处理 ...

    free(buf);
cleanup_fd:
    close(fd);
}
```

### 5.4 read()/write() 返回值小于请求值

```c
// 不可靠写法：假设一次 read 能读到全部数据
char buf[1024];
ssize_t n = read(fd, buf, sizeof(buf));
// n 可能只有 100，但文件还有更多数据没读完

// 可靠写法：循环读取直到读完
size_t total = 0;
while (total < expected_size) {
    ssize_t n = read(fd, buf + total, expected_size - total);
    if (n == 0) break;       // EOF
    if (n == -1) {
        if (errno == EINTR) continue;
        perror("read");
        break;
    }
    total += n;
}
```

### 5.5 对不可 seek 的文件使用 lseek()

```c
// 管道、socket、终端等不可 seek
int fd = STDIN_FILENO;  // 标准输入（可能是终端或管道）
off_t pos = lseek(fd, 0, SEEK_CUR);  // 失败，errno = ESPIPE
```

判断文件是否可 seek：

```c
off_t pos = lseek(fd, 0, SEEK_CUR);
if (pos == -1) {
    if (errno == ESPIPE) {
        printf("This fd does not support seeking\n");
    }
}
```

### 5.6 调试工具：strace

`strace` 是调试系统 IO 问题的最强工具。它可以追踪进程发出的所有系统调用。

```bash
# 追踪程序的所有系统调用
strace ./mycp source.txt dest.txt

# 只追踪文件相关系统调用（open/read/write/close/lseek）
strace -e trace=open,read,write,close,lseek ./file_demo

# 追踪正在运行的进程（通过 PID）
strace -p 12345

# 统计系统调用次数和耗时
strace -c ./file_demo
```

**strace 输出示例**（运行 `./file_demo` 时）：

```
open("demo.txt", O_RDWR|O_CREAT|O_TRUNC, 0644) = 3
write(3, "Hello, Linux System IO!\n", 24) = 24
lseek(3, 0, SEEK_CUR)                   = 24
lseek(3, 0, SEEK_SET)                   = 0
read(3, "Hello, Linux System IO!\n", 255) = 24
close(3)                                = 0
```

从 strace 输出可以清晰地看到：
- `open()` 返回 fd=3
- `write()` 写了 24 字节
- `read()` 读了 24 字节
- 所有系统调用的参数和返回值一目了然

---

## 6. 标准 IO 函数速查（对比参考）

了解系统 IO 后，对比学习标准 IO 的对应函数：

| 系统 IO | 标准 IO | 说明 |
|---------|---------|------|
| `open()` | `fopen()` | 打开文件，返回 `FILE *` 而非 `int` |
| `close()` | `fclose()` | 关闭文件，同时刷新缓冲区 |
| `read()` | `fread()` | 读取，参数顺序不同：先 buf 再 size |
| `write()` | `fwrite()` | 写入，参数顺序同 fread |
| `lseek()` | `fseek()` / `ftell()` / `rewind()` | 定位，`rewind(fp)` 等价于 `fseek(fp, 0, SEEK_SET)` |
| 无直接对应 | `fgets()` / `fputs()` | 按行读写字符串 |
| 无直接对应 | `fprintf()` / `fscanf()` | 格式化读写 |
| 无直接对应 | `fgetc()` / `fputc()` | 单字符读写 |
| 无直接对应 | `fflush()` | 强制刷新用户态缓冲区 |
| 无直接对应 | `setvbuf()` | 设置缓冲策略 |

标准 IO 打开模式与系统 IO 的对应关系：

| 标准 IO mode 字符串 | 等价的系统 IO flags |
|---------------------|---------------------|
| `"r"` | `O_RDONLY` |
| `"r+"` | `O_RDWR` |
| `"w"` | `O_WRONLY \| O_CREAT \| O_TRUNC` |
| `"w+"` | `O_RDWR \| O_CREAT \| O_TRUNC` |
| `"a"` | `O_WRONLY \| O_CREAT \| O_APPEND` |
| `"a+"` | `O_RDWR \| O_CREAT \| O_APPEND` |

一个标准 IO 版本的"文件复制"程序，与 2.8 节的系统 IO 版本对比：

```c
/*
 * mycp_stdio.c -- 用标准 IO 实现文件复制（对比系统 IO 版本）
 * 编译：gcc -Wall -o mycp_stdio mycp_stdio.c
 */

#include <stdio.h>
#include <stdlib.h>

#define BUFSIZE 4096

int main(int argc, char *argv[])
{
    FILE *fp_src, *fp_dst;
    char buf[BUFSIZE];
    size_t nread;

    if (argc != 3) {
        fprintf(stderr, "Usage: %s <src> <dst>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    fp_src = fopen(argv[1], "rb");
    if (fp_src == NULL) {
        perror("fopen src");
        exit(EXIT_FAILURE);
    }

    fp_dst = fopen(argv[2], "wb");
    if (fp_dst == NULL) {
        perror("fopen dst");
        fclose(fp_src);
        exit(EXIT_FAILURE);
    }

    while ((nread = fread(buf, 1, BUFSIZE, fp_src)) > 0) {
        fwrite(buf, 1, nread, fp_dst);
    }

    fclose(fp_src);
    fclose(fp_dst);
    return 0;
}
```

> 标准 IO 版本的代码更简洁，因为 `fread`/`fwrite` 内部已经处理了缓冲和部分读写的循环逻辑。

---

## 本讲总结

| 类别 | 内容 |
|------|------|
| **IO分类** | 系统IO（系统调用） vs 标准IO（C库函数）；核心差异在于用户态缓冲区 |
| **5大系统IO函数** | `open`, `lseek`, `write`, `read`, `close` |
| **文件描述符本质** | `fd_array[]` 数组的下标，类似酒店房卡编号 |
| **打开主模式** | `O_RDONLY`, `O_WRONLY`, `O_RDWR` |
| **打开辅模式** | `O_CREAT`, `O_APPEND`, `O_TRUNC`, `O_SYNC`, `O_NONBLOCK` |
| **实战技巧** | 错误处理（检查返回值+errno）、防止fd泄漏、循环读写、strace调试 |

---

*笔记生成日期：2026-07-22 | 转写：Whisper tiny | 优化日期：2026-07-24*

---

## 参考资源

- `man 2 open` -- 系统 IO 函数手册
- `man 3 fopen` -- 标准 IO 函数手册
- `man 1 strace` -- 系统调用追踪工具
- 《UNIX环境高级编程》（APUE）第3章 -- 文件IO
- Linux 内核源码 `include/linux/fs.h` -- `file_operations` 结构体定义
