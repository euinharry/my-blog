---
title: "第41讲：open/close 函数"
date: 2026-09-11T09:03:00+08:00
draft: false
description: "#include <sys/types.h>"
series: ["Linux 入门"]
series_order: 24
categories: ["技术笔记"]
tags: ["Linux", "文件IO"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P42)
> **标题**：第41讲 -- open/close 函数
> **时长**：约11分钟
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. 头文件

### 1.1 `open()` 需要的头文件

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
```

这三个头文件各有分工：

| 头文件 | 提供的内容 |
|--------|-----------|
| `<sys/types.h>` | 基础类型定义（如 `mode_t`, `off_t`, `ssize_t`） |
| `<sys/stat.h>` | `mode_t` 类型以及 `S_IRUSR`, `S_IWUSR` 等权限宏 |
| `<fcntl.h>` | `open()` 函数声明以及 `O_RDONLY`, `O_CREAT` 等 flags 宏 |

**注意**：这三个头文件是 `open()` 的"最小集合"。实际上 `<fcntl.h>` 内部通常会间接包含 `<sys/types.h>`，但显式写出更规范，也避免跨平台兼容问题。

### 1.2 `close()` 需要的头文件

```c
#include <unistd.h>
```

`<unistd.h>` 是 POSIX 标准中"Unix 标准函数"的头文件，包含了 `close()`, `read()`, `write()`, `lseek()` 等基础 I/O 函数的声明。

---

## 2. `open()` 函数的两种原型

### 2.1 文件已存在时（2 个参数）

```c
int open(const char *pathname, int flags);
```

**适用条件**：要打开的文件在文件系统中**真实存在**时。

### 2.2 文件可能不存在时（3 个参数）

```c
int open(const char *pathname, int flags, mode_t mode);
```

**适用条件**：文件可能不存在，需要创建时（配合 `O_CREAT`）。

| 参数 | 类型 | 说明 |
|------|------|------|
| `pathname` | `const char *` | 要打开的文件路径（绝对路径或相对路径） |
| `flags` | `int` | 访问模式与行为标志的按位或组合 |
| `mode` | `mode_t` | 仅在使用 `O_CREAT` 时需要，指定新文件的初始权限 |

**为什么是可变参数？** C 语言中 `open()` 被声明为可变参数函数（类似 `printf`）。当 `flags` 中包含 `O_CREAT` 时，必须提供第三个参数 `mode`；否则第三个参数被忽略。如果用了 `O_CREAT` 却不传 `mode`，新文件的权限是栈上的随机值，结果是不可预测的。

### 2.3 flags 参数详解

`flags` 参数分为两大类：**访问模式**（必选其一）和**行为标志**（可选，按位或组合）。

#### 2.3.1 访问模式（三选一）

| 宏 | 数值 | 含义 |
|----|------|------|
| `O_RDONLY` | 0 | 只读打开 |
| `O_WRONLY` | 1 | 只写打开 |
| `O_RDWR` | 2 | 读写打开 |

在 Linux 内核中，这三个值正好对应 `fcntl.h` 中的位掩码。`open()` 调用时必须且只能指定其中**一个**，否则行为未定义。

#### 2.3.2 常用行为标志

| 宏 | 含义 | 典型场景 |
|----|------|----------|
| `O_CREAT` | 文件不存在则创建 | 创建新文件 |
| `O_EXCL` | 与 `O_CREAT` 联用，文件已存在则失败 | 原子创建锁文件 |
| `O_TRUNC` | 打开时清空文件内容（长度截断为 0） | 覆写已有文件 |
| `O_APPEND` | 每次写入追加到文件末尾 | 日志文件 |
| `O_NONBLOCK` | 非阻塞模式打开（用于 FIFO、设备文件等） | 管道、socket |
| `O_SYNC` | 每次 write 等待数据写入磁盘后才返回 | 数据库、关键数据 |

**flags 组合示例**：

```c
// 只读打开已有文件
fd = open("data.txt", O_RDONLY);

// 读写打开，不存在则创建，权限 0644（注意三个参数）
fd = open("log.txt", O_RDWR | O_CREAT, 0644);

// 写打开，不存在则创建，已存在则截断为 0
fd = open("output.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);

// 追加写，不存在则创建
fd = open("app.log", O_WRONLY | O_CREAT | O_APPEND, 0644);

// 原子创建：仅当文件不存在时才创建（用于进程互斥锁文件）
fd = open("lock.pid", O_WRONLY | O_CREAT | O_EXCL, 0644);
// 如果 lock.pid 已存在，返回 -1，errno = EEXIST
```

#### 2.3.3 O_EXCL 的原子性

`O_CREAT | O_EXCL` 的组合保证了**检查文件是否存在**和**创建文件**这两个动作是原子的。这在多进程环境中至关重要：

```c
// 错误做法（非原子，存在竞态条件）：
// if (access("lock", F_OK) != 0)  ← 检查时刻文件不存在
//     fd = open("lock", O_CREAT);  ← 创建时可能已被其他进程创建

// 正确做法（原子）：
fd = open("lock", O_WRONLY | O_CREAT | O_EXCL, 0644);
if (fd < 0 && errno == EEXIST) {
    // 文件已存在，说明另一个进程已获取锁
    printf("another instance is running\n");
}
```

### 2.4 mode 参数与权限位详解

#### 2.4.1 八进制权限表示

Linux 文件权限用 3 组 rwx 位表示，每组 3 位，共 9 位：

```
  owner  group  other
  rwx    rwx    rwx
  111    101    101   → 二进制 → 0755（八进制）
```

| 权限位 | 八进制值 | 含义 |
|--------|----------|------|
| `S_IRUSR` | 0400 | owner 读 |
| `S_IWUSR` | 0200 | owner 写 |
| `S_IXUSR` | 0100 | owner 执行 |
| `S_IRGRP` | 0040 | group 读 |
| `S_IWGRP` | 0020 | group 写 |
| `S_IXGRP` | 0010 | group 执行 |
| `S_IROTH` | 0004 | other 读 |
| `S_IWOTH` | 0002 | other 写 |
| `S_IXOTH` | 0001 | other 执行 |

两种写法等价：

```c
// 八进制写法（常用）
fd = open("file.txt", O_CREAT, 0644);

// 宏写法（更可读但不直观）
fd = open("file.txt", O_CREAT, S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH);
```

#### 2.4.2 umask 对最终权限的影响

`open()` 中指定的 `mode` **不是**文件的最终权限。内核会用进程的 `umask` 值对 `mode` 做一次按位取反再与的操作：

```
最终权限 = mode & (~umask)
```

**实例**：

```bash
$ umask
0022         # 当前 umask 值

# 程序：fd = open("test", O_CREAT, 0666);
# 计算：0666 & (~0022) = 0666 & 0755 = 0644

$ ls -l test
-rw-r--r-- 1 user user 0 Jul 24 10:00 test   ← 权限是 0644，不是 0666
```

如果创建文件时想要严格的权限，有两种方法：

```c
// 方法1：先保存并修改 umask（不推荐，影响整个进程）
mode_t old = umask(0);      // 临时清除 umask
fd = open("file", O_CREAT, 0666);
umask(old);                 // 恢复

// 方法2：创建后用 fchmod() 修正（推荐）
fd = open("file", O_CREAT, 0666);
fchmod(fd, 0666);           // 显式设置最终权限
```

### 2.5 返回值

| 返回值 | 含义 |
|--------|------|
| **正整数**（>= 0） | 成功，返回文件描述符（fd），是 fd_array 数组的索引 |
| **-1** | 失败，全局变量 `errno` 被设置为具体错误码 |

---

## 3. `close()` 函数

```c
#include <unistd.h>

int close(int fd);
```

| 返回值 | 含义 |
|--------|------|
| **0** | 关闭成功 |
| **-1** | 关闭失败，`errno` 被设置 |

### 3.1 close 失败的常见原因

虽然 `close()` 很少失败，但在以下场景可能返回 -1：

| errno 值 | 含义 |
|----------|------|
| `EBADF` | `fd` 不是一个有效的文件描述符（例如已经被关闭、或从未打开） |
| `EINTR` | `close()` 被信号中断（某些文件系统上会发生） |
| `EIO` | NFS 等网络文件系统在刷新缓存时发生 I/O 错误 |

**特别注意 `EINTR`**：在 Linux 上，`close(fd)` 即使被信号中断，fd 也**已经被释放**，不应该重试。POSIX 标准对此行为没有明确规定，但 Linux 的实现是：close 总是释放 fd，即使返回 -1 且 errno=EINTR。

### 3.2 close 的错误处理实践

```c
int ret = close(fd);
if (ret == -1) {
    // fd 已被内核释放，不能再使用
    // 根据 errno 决定如何处理
    if (errno == EBADF) {
        fprintf(stderr, "close: invalid fd (was it already closed?)\n");
    } else if (errno == EIO) {
        fprintf(stderr, "close: I/O error, data may be lost\n");
    }
    // 注意：不需要也不应该再次 close(fd)
}
```

---

## 4. errno 错误处理详解

`open()` 失败时返回 -1，并在全局变量 `errno` 中设置具体错误码。使用 `errno` 需要包含头文件：

```c
#include <errno.h>
#include <string.h>   // strerror()
```

### 4.1 open 常见 errno 值

| errno | 含义 | 触发场景 |
|-------|------|----------|
| `EACCES` | 权限不足 | 没有读/写权限访问文件或目录 |
| `ENOENT` | 文件或目录不存在 | 路径中某级目录不存在，或文件不存在且未指定 O_CREAT |
| `EEXIST` | 文件已存在 | 使用了 O_CREAT \| O_EXCL 但文件已存在 |
| `EISDIR` | 是目录 | pathname 指向目录但 flags 要求写操作 |
| `ENOTDIR` | 路径中有非目录组件 | 路径中某个中间组件不是目录 |
| `EMFILE` | 进程 fd 达到上限 | 当前进程打开的文件描述符超过 `ulimit -n` |
| `ENFILE` | 系统 fd 达到上限 | 整个系统打开的文件数超过 `/proc/sys/fs/file-max` |
| `ENOSPC` | 设备空间不足 | 使用 O_CREAT 时磁盘已满 |
| `EROFS` | 只读文件系统 | 在只读文件系统上要求写权限 |
| `ENAMETOOLONG` | 路径名过长 | pathname 超过 PATH_MAX（通常 4096） |
| `EFAULT` | 指针无效 | pathname 指向非法内存地址 |
| `ELOOP` | 符号链接循环 | 路径中包含循环的符号链接 |

### 4.2 使用 errno 的完整示例

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>

int main() {
    // 尝试以只写模式打开一个只读文件
    int fd = open("/etc/passwd", O_WRONLY);

    if (fd < 0) {
        fprintf(stderr, "open failed: %s (errno=%d)\n",
                strerror(errno), errno);

        // 根据 errno 进行不同的错误处理
        switch (errno) {
        case EACCES:
            fprintf(stderr, "Permission denied\n");
            break;
        case ENOENT:
            fprintf(stderr, "File not found\n");
            break;
        case EISDIR:
            fprintf(stderr, "Path is a directory\n");
            break;
        default:
            fprintf(stderr, "Unknown error\n");
            break;
        }
        exit(EXIT_FAILURE);
    }

    printf("fd = %d\n", fd);
    close(fd);
    return 0;
}
```

**输出示例**：

```
open failed: Permission denied (errno=13)
Permission denied
```

### 4.3 perror() 快捷函数

`perror()` 是打印 errno 描述的便捷函数，不需要手动调用 `strerror()`：

```c
fd = open("missing.txt", O_RDONLY);
if (fd < 0) {
    perror("open missing.txt");  // 输出: open missing.txt: No such file or directory
    exit(1);
}
```

`perror()` 的输出格式为：`"用户字符串: 系统错误描述\n"`，始终输出到 stderr。

---

## 5. 实验演示

### 5.1 实验一：只读模式打开不存在的文件

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>

int main() {
    int fd;

    // 只读模式打开 A.txt（当前目录不存在此文件）
    fd = open("A.txt", O_RDONLY);

    if (fd < 0) {
        printf("open error: %s (errno=%d)\n", strerror(errno), errno);
        return 1;              // fd 无效，直接退出，不要 close
    }

    printf("fd = %d\n", fd);
    close(fd);
    return 0;
}
```

**编译运行**：

```bash
$ make           # 编译
$ ./open_close
open error: No such file or directory (errno=2)
```

**修正说明**：原实验中 `fd < 0` 时仍调用了 `close(fd)`（即 `close(-1)`），虽然不会导致程序崩溃，但逻辑上不正确。正确的做法是：fd 无效时不调用 close，直接退出。

### 5.2 创建文件后再运行

```bash
$ touch A.txt    # 创建 A.txt
$ ./open_close
fd = 3           # 打开成功
```

### 5.3 为什么 fd 是 3？

```
进程启动时默认打开 3 个文件描述符：
  fd=0 -> stdin  （标准输入）
  fd=1 -> stdout （标准输出）
  fd=2 -> stderr （标准错误输出）

0/1/2 已被占用，用户打开的第一个文件从 fd=3 开始分配。
内核选择当前可用的最小非负整数作为新 fd。
```

**验证**：可以通过 `/proc` 文件系统查看进程打开的文件：

```bash
$ ./open_close &
[1] 12345

$ ls -l /proc/12345/fd
total 0
lrwx------ 1 user user 64 Jul 24 10:00 0 -> /dev/pts/0
lrwx------ 1 user user 64 Jul 24 10:00 1 -> /dev/pts/0
lrwx------ 1 user user 64 Jul 24 10:00 2 -> /dev/pts/0
lr-x------ 1 user user 64 Jul 24 10:00 3 -> /home/user/A.txt
```

### 5.4 实验二：使用 O_CREAT 创建不存在的文件

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>

int main() {
    int fd;

    // 读写打开 B.txt，若不存在则创建，权限 0644
    fd = open("B.txt", O_RDWR | O_CREAT, 0644);

    if (fd < 0) {
        printf("open error: %s\n", strerror(errno));
        return 1;
    }

    printf("fd = %d\n", fd);
    close(fd);
    return 0;
}
```

**效果**：

```bash
$ ls
open_close.c  Makefile

$ ./open_close
fd = 3

$ ls
open_close.c  Makefile  B.txt    <- 自动创建了 B.txt

$ ls -l B.txt
-rw-r--r-- 1 user user 0 Jul 24 10:00 B.txt   <- 权限 0644
```

> `O_CREAT` 辅模式的作用：当文件不存在时，`open()` 会自动创建该文件。

### 5.5 实验三：O_EXCL 原子创建

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>

int main() {
    int fd;

    fd = open("lockfile", O_WRONLY | O_CREAT | O_EXCL, 0644);

    if (fd < 0) {
        if (errno == EEXIST) {
            printf("lockfile already exists, another process is running\n");
        } else {
            perror("open lockfile");
        }
        return 1;
    }

    printf("lockfile created, fd = %d\n", fd);

    // 写入 PID
    char buf[32];
    snprintf(buf, sizeof(buf), "%d\n", getpid());
    write(fd, buf, strlen(buf));

    close(fd);
    return 0;
}
```

运行两次：

```bash
$ ./lock_demo
lockfile created, fd = 3

$ ./lock_demo
lockfile already exists, another process is running
```

### 5.6 实验四：O_TRUNC 与 O_APPEND 的区别

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main() {
    int fd;

    // 场景 A：O_TRUNC -- 每次打开清空文件
    fd = open("trunc.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    write(fd, "line1\n", 6);
    close(fd);

    fd = open("trunc.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
    write(fd, "line2\n", 6);
    close(fd);
    // trunc.txt 内容只有 "line2\n"，line1 被清除了

    // 场景 B：O_APPEND -- 每次写入追加到末尾
    fd = open("append.txt", O_WRONLY | O_CREAT | O_APPEND, 0644);
    write(fd, "line1\n", 6);
    close(fd);

    fd = open("append.txt", O_WRONLY | O_CREAT | O_APPEND, 0644);
    write(fd, "line2\n", 6);
    close(fd);
    // append.txt 内容为 "line1\nline2\n"，两行都保留

    return 0;
}
```

运行结果：

```bash
$ cat trunc.txt
line2

$ cat append.txt
line1
line2
```

### 5.7 实验五：umask 对文件权限的影响

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main() {
    // 查看当前 umask
    mode_t old_umask = umask(0);   // 设置为 0 并获取旧值
    printf("old umask: %04o\n", old_umask);

    // 在 umask=0 时创建文件
    int fd1 = open("file_umask0", O_CREAT | O_WRONLY, 0666);
    close(fd1);

    // 恢复 umask 后再创建文件
    umask(old_umask);
    int fd2 = open("file_normal", O_CREAT | O_WRONLY, 0666);
    close(fd2);

    return 0;
}
```

```bash
$ ./umask_demo
old umask: 0022

$ ls -l file_umask0 file_normal
-rw-rw-rw- 1 user user 0 Jul 24 10:00 file_umask0   <- 0666 无过滤
-rw-r--r-- 1 user user 0 Jul 24 10:00 file_normal   <- 0666 & ~0022 = 0644
```

---

## 6. 文件描述符管理

### 6.1 fd 泄露：忘记 close 的后果

每个进程能同时打开的文件数是有限的。每次 `open()` 成功都消耗一个 fd，`close()` 将它归还。如果只 open 不 close，fd 会被耗尽：

```c
// 错误示例：fd 泄露
for (int i = 0; i < 100000; i++) {
    int fd = open("data.txt", O_RDONLY);
    if (fd < 0) {
        perror("open");
        break;   // 最终会在这里失败，因为 fd 用完了
    }
    // 忘记 close(fd) <- 严重问题
}
```

运行效果（假设 ulimit -n = 1024）：

```
open: Too many open files       <- 第 1021 次循环后失败
```

**fd 泄露的常见场景**：

| 场景 | 原因 |
|------|------|
| 循环中 open 后忘记 close | 代码遗漏 |
| 异常路径没有 close | 只写了成功分支的 close，错误分支直接 return |
| 信号处理函数中 open | 信号打断正常流程，close 被跳过 |

**预防措施**：

```c
// 使用 goto 统一清理（C 语言经典模式）
int ret = -1;
int fd = open("file", O_RDONLY);
if (fd < 0) {
    perror("open");
    goto out;
}

// ... 使用 fd ...

ret = 0;

out:
if (fd >= 0) close(fd);   // 无论成功或失败都执行
return ret;
```

### 6.2 ulimit：文件描述符上限

`ulimit -n` 控制当前 shell 及其子进程能同时打开的最大文件描述符数量。

```bash
# 查看软限制（soft limit，实际生效值）
$ ulimit -n
1024

# 查看硬限制（hard limit，软限制的上限）
$ ulimit -Hn
1048576

# 临时提高软限制（不能超过硬限制）
$ ulimit -n 4096
```

在 C 程序中可以通过 `getrlimit()` / `setrlimit()` 编程修改：

```c
#include <sys/resource.h>
#include <stdio.h>

int main() {
    struct rlimit rl;

    // 获取当前限制
    getrlimit(RLIMIT_NOFILE, &rl);
    printf("soft limit: %lu, hard limit: %lu\n",
           rl.rlim_cur, rl.rlim_max);

    // 尝试提高软限制到 4096
    rl.rlim_cur = 4096;
    if (setrlimit(RLIMIT_NOFILE, &rl) == 0) {
        printf("raised soft limit to 4096\n");
    }
    return 0;
}
```

**注意**：提高硬限制需要 root 权限。普通用户只能将软限制设置在 [0, 硬限制] 范围内。

系统级别的限制在 `/proc/sys/fs/file-max`：

```bash
$ cat /proc/sys/fs/file-max
9223372036854775807     # 系统范围内所有进程可打开的文件总数上限
```

### 6.3 多个 fd 指向同一文件

同一个进程可以对同一个文件多次调用 `open()`，每次返回不同的 fd：

```c
int fd1 = open("data.txt", O_RDONLY);
int fd2 = open("data.txt", O_RDONLY);
int fd3 = open("data.txt", O_RDONLY);

// fd1=3, fd2=4, fd3=5（假设之前没有其他打开）
// 三个 fd 各有独立的文件偏移量（file position）
```

每个 fd 对应的内核 `file` 对象是独立的，它们的**文件偏移量互不影响**。

```c
// 演示：两个 fd 读取同一文件
int fd1 = open("data.txt", O_RDONLY);  // data.txt 内容: "ABCDEFG"
int fd2 = open("data.txt", O_RDONLY);

char buf1[2], buf2[2];

read(fd1, buf1, 2);   // 从 fd1 读 2 字节 -> "AB"
read(fd2, buf2, 2);   // 从 fd2 读 2 字节 -> "AB"（不是 "CD"！）
// fd2 的偏移量从头开始，不受 fd1 影响
```

而通过 `dup()` 或 `dup2()` 复制出的 fd 会**共享**同一个 file 对象和偏移量：

```c
int fd1 = open("data.txt", O_RDONLY);
int fd2 = dup(fd1);    // fd2 与 fd1 共享内核 file 对象

char buf1[2], buf2[2];
read(fd1, buf1, 2);    // 从共享偏移读取 "AB"，偏移移到 2
read(fd2, buf2, 2);    // 从共享偏移读取 "CD"，偏移移到 4
```

**总结**：

| 方式 | 是否共享偏移量 |
|------|---------------|
| 多次 `open()` 同一文件 | 否（独立） |
| `dup()` / `dup2()` | 是（共享） |
| `fork()` 后父子进程 | 是（共享同一 file 对象） |

---

## 7. 完整的文件打开流程（复习）

```
open("A.txt", O_RDONLY)
    |
    v
VFS 层：根据路径查找 inode
    |-- 路径解析：逐级查找目录项（dentry）
    |-- 权限检查：检查进程是否有读/写/执行权限
    |-- 文件存在？
    |       |
    |       是                    否
    |       |                     |
    |       v                     v
    |   分配 fd_array[]         flags 含 O_CREAT？
    |   中的空闲 slot             |
    |       |                 是          否
    |       v                 |           v
    |   创建 file 对象        创建新 inode  返回 -1
    |       |                + 目录项      errno=ENOENT
    |       v                 |
    |   填充 file_operations   v
    |   file->f_mode         跳转到"分配 fd"
    |   file->f_pos=0
    |   file->f_flags
    |       |
    |       v
    +---> 返回 fd（文件描述符，即数组索引）
```

关于 lseek 偏移量：`open()` 返回时文件偏移量 `f_pos` 的初始值取决于 `flags`：

| flags 包含 | f_pos 初始值 |
|-----------|-------------|
| 无特殊标志 或 O_RDONLY / O_WRONLY / O_RDWR | 0（文件开头） |
| `O_APPEND` | 文件末尾（每次 write 前自动 seek 到末尾） |

---

## 8. 实战示例：日志记录器

综合运用所学知识，写一个简单的日志记录器：

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>
#include <time.h>

#define LOG_FILE "app.log"

void write_log(const char *level, const char *msg) {
    // O_WRONLY | O_CREAT | O_APPEND：只写、不存在则创建、追加
    int fd = open(LOG_FILE, O_WRONLY | O_CREAT | O_APPEND, 0644);
    if (fd < 0) {
        fprintf(stderr, "cannot open log file: %s\n", strerror(errno));
        return;
    }

    // 获取时间戳
    time_t now = time(NULL);
    char *timestr = ctime(&now);
    timestr[strlen(timestr) - 1] = '\0';  // 去掉换行

    // 构造日志行
    char line[512];
    int len = snprintf(line, sizeof(line), "[%s] [%s] %s\n",
                       timestr, level, msg);

    // 写入
    if (write(fd, line, len) != len) {
        fprintf(stderr, "write failed: %s\n", strerror(errno));
    }

    close(fd);
}

int main() {
    write_log("INFO",  "application started");
    write_log("DEBUG", "initializing modules");
    write_log("ERROR", "something went wrong");
    write_log("INFO",  "application stopped");

    printf("Log written to %s\n", LOG_FILE);
    return 0;
}
```

运行效果：

```bash
$ ./logger
Log written to app.log

$ cat app.log
[Thu Jul 24 10:00:00 2026] [INFO] application started
[Thu Jul 24 10:00:00 2026] [DEBUG] initializing modules
[Thu Jul 24 10:00:00 2026] [ERROR] something went wrong
[Thu Jul 24 10:00:00 2026] [INFO] application stopped
```

**设计要点**：
- `O_APPEND` 保证每次写入在文件末尾，不受其他进程影响
- `O_CREAT` 保证日志文件首次使用时会自动创建
- 每次调用 `write_log` 都 open/close 一次：简单但频繁时性能差。生产环境应保持 fd 常开

---

## 本讲总结

| 主题 | 要点 |
|------|------|
| **头文件** | `open()` 需要 `<fcntl.h>` + `<sys/types.h>` + `<sys/stat.h>`；`close()` 需要 `<unistd.h>` |
| **open 原型** | 2 参数（文件已存在）；3 参数（配合 `O_CREAT` 时） |
| **访问模式** | `O_RDONLY`(0) / `O_WRONLY`(1) / `O_RDWR`(2)，三选一 |
| **行为标志** | `O_CREAT`, `O_EXCL`, `O_TRUNC`, `O_APPEND`, `O_NONBLOCK`, `O_SYNC` 等，按位或组合 |
| **mode 权限** | 八进制 `0644` 或宏 `S_IRUSR|S_IWUSR|S_IRGRP|S_IROTH`；最终权限受 umask 影响 |
| **返回值** | 成功返回 fd（>=0），失败返回 -1 并设置 errno |
| **errno** | `EACCES`(权限), `ENOENT`(不存在), `EEXIST`(已存在), `EMFILE`(进程fd耗尽) 等 |
| **close** | 返回 0 成功，-1 失败；Linux 上 close 总是释放 fd |
| **fd 分配** | 从 3 开始；0=stdin, 1=stdout, 2=stderr 已被占用；取最小可用 |
| **fd 泄露** | 忘记 close 导致 fd 耗尽（ulimit -n），长运行进程必须注意 |
| **多 fd 同文件** | 多次 open 互不影响；dup 出的 fd 共享偏移量 |
| **ulimit** | `ulimit -n` 查看；`getrlimit/setrlimit` 编程修改 |

---

*笔记生成日期：2026-07-22 | 转写：Whisper tiny*
*优化扩充日期：2026-07-24 | 补充：errno 处理、flags 详解、mode/umask、fd 泄露、ulimit、多 fd 场景、实战示例*
