---
title: "第42讲：read/write 函数"
date: 2026-09-11T09:02:00+08:00
draft: false
description: "#include <unistd.h>"
series: ["Linux 入门"]
series_order: 25
categories: ["技术笔记"]
tags: ["Linux", "文件IO"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P43)
> **标题**：第42讲 -- read/write 函数
> **时长**：约15分钟
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. `read()` 函数

### 1.1 头文件

```c
#include <unistd.h>
```

### 1.2 函数原型

```c
ssize_t read(int fd, void *buf, size_t count);
```

| 参数 | 说明 |
|------|------|
| `fd` | 文件描述符（open 返回的） |
| `buf` | 存放读取内容的缓冲区指针 |
| `count` | 要读取的字节数 |

**参数详解**：

- `fd` 必须是通过 `open()` 系统调用获得的**有效文件描述符**，且文件必须以**可读模式**（`O_RDONLY` 或 `O_RDWR`）打开。如果传入无效 fd（如 -1 或未打开的 fd），`read()` 返回 -1 并设置 `errno` 为 `EBADF`。
- `buf` 指向用户空间的缓冲区，用于接收从内核读取的数据。**缓冲区必须由调用者预先分配**，且大小至少为 `count` 字节。如果 `buf` 是 NULL 或指向无效地址，`read()` 返回 -1 并设置 `errno` 为 `EFAULT`。
- `count` 表示期望读取的字节数。注意 `count` 的类型是 `size_t`（无符号整型），而返回值 `ssize_t` 是有符号的 -- 这是为了用 -1 表示错误。`count` 的最大值受 `SSIZE_MAX`（通常为 2GB-1）限制。

### 1.3 返回值

| 返回值 | 含义 |
|--------|------|
| **count** | 成功读取了 count 个字节 |
| **0 < ret < count** | 文件剩余字节数小于 count，或被异步信号打断 |
| **0** | 已读到文件末尾（EOF） |
| **-1** | 读取失败，需检查 errno |

**返回值深入解释**：

- **返回 count**：说明请求的字节数全部被成功读取。这发生在文件中仍有足够数据时。
- **返回 0 < ret < count**：有两种可能：一是文件中剩余可读数据不足 count 字节（常见于接近文件末尾时），二是系统调用被信号中断（此时 errno 为 `EINTR`）。区分方法：返回 -1 且 errno==EINTR 才是信号中断；返回正数但小于 count 表示正常的部分读取。
- **返回 0**：表示已到文件末尾（EOF）。注意：对于普通文件，后续再次 read 仍返回 0。对于管道、socket 等，返回 0 表示对端已关闭连接。
- **返回 -1**：读取失败。必须检查 `errno` 来获取具体错误原因。

**常见 errno 值**：

| errno | 含义 |
|-------|------|
| `EBADF` | fd 不是有效的文件描述符或未以读模式打开 |
| `EFAULT` | buf 指向的地址不可访问 |
| `EINTR` | 读取操作被信号中断（可以重试） |
| `EIO` | I/O 错误（如硬件故障） |
| `EISDIR` | fd 指向一个目录（不能 read 目录） |

---

## 2. `write()` 函数

### 2.1 头文件

```c
#include <unistd.h>
```

### 2.2 函数原型

```c
ssize_t write(int fd, const void *buf, size_t count);
```

| 参数 | 说明 |
|------|------|
| `fd` | 文件描述符 |
| `buf` | 要写入的内容缓冲区指针 |
| `count` | 要写入的字节数 |

**参数详解**：

- `fd` 必须是以**可写模式**（`O_WRONLY`、`O_RDWR`、`O_APPEND` 等）打开的文件描述符。如果 fd 无效，返回 -1 并设置 `errno` 为 `EBADF`。
- `buf` 的参数类型是 `const void *`，说明 write 不会修改缓冲区内容。
- `count` 表示期望写入的字节数。与 read 类似，count 最大值受 `SSIZE_MAX` 限制。

### 2.3 返回值

| 返回值 | 含义 |
|--------|------|
| **count** | 成功写入 count 个字节 |
| **0 < ret < count** | 写入时被异步信号打断（只写入了部分） |
| **0** | 没有写入任何数据（罕见，通常表示磁盘空间满等） |
| **-1** | 写入失败 |

**返回值深入解释**：

- **返回 count**：理想情况，全部数据写入成功。对于普通文件和磁盘，这通常意味着数据已进入内核缓冲区（page cache），但不一定已经持久化到磁盘。要确保持久化，需要调用 `fsync()` 或 `fdatasync()`。
- **返回 0 < ret < count**：部分写入。通常由信号中断（errno == EINTR）或磁盘空间不足导致。**在生产代码中必须处理这种情况**：记录已写入字节数，继续从剩余位置写入。
- **返回 0**：极少见。可能出现在对端关闭的管道/socket 写操作，或磁盘满了无法写入任何数据。
- **返回 -1**：写入失败。检查 errno 获取原因。

**常见 errno 值**：

| errno | 含义 |
|-------|------|
| `EBADF` | fd 无效或未以写模式打开 |
| `EFAULT` | buf 指向无效地址 |
| `ENOSPC` | 磁盘空间不足 |
| `EIO` | I/O 错误 |
| `EPIPE` | 写入管道时对端已关闭（会收到 SIGPIPE 信号） |
| `EINTR` | 被信号中断（可重试） |

---

## 3. 缓冲区大小对性能的影响分析

`read()` 和 `write()` 每次调用都要**从用户态切换到内核态**（context switch），这是系统调用最昂贵的部分。缓冲区大小的选择直接影响性能：

### 3.1 原理分析

```
用户空间应用程序
      |
      |  read(fd, buf, N)    <-- 系统调用（切换到内核态）
      v
-------------------------------------
     内核态 (VFS / page cache)
      |
      |  如果数据不在 page cache 中，触发磁盘 I/O
      v
-------------------------------------
     磁盘（物理设备）
```

每次 `read()` 调用都会：
1. 触发用户态到内核态的切换
2. 内核检查 page cache 中是否有数据
3. 如果 cache 未命中，从磁盘读取
4. 将数据从内核空间拷贝到用户空间
5. 切换回用户态

**缓冲区越小，系统调用次数越多，上下文切换开销越大。**

### 3.2 性能数据参考

以下是复制 100MB 文件时，不同缓冲区大小的理论对比：

| 缓冲区大小 | 系统调用次数（read+write） | 上下文切换次数 | 预计耗时趋势 |
|-----------|--------------------------|--------------|------------|
| 1 字节 | 约 2 亿次 | 约 2 亿次 | 极慢 |
| 512 字节 | 约 40 万次 | 约 40 万次 | 很慢 |
| 4KB (1页) | 约 5 万次 | 约 5 万次 | 较慢 |
| 64KB | 约 3200 次 | 约 3200 次 | 正常 |
| 1MB | 约 200 次 | 约 200 次 | 快 |
| 8MB | 约 25 次 | 约 25 次 | 很快 |

**最佳实践**：大多数场景下，4KB 到 64KB 是较好的折中。过大的缓冲区（超过数 MB）在栈上分配可能导致栈溢出；应使用堆分配（`malloc`）或静态数组。

### 3.3 实验：不同缓冲区大小的 cp 性能对比

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

double get_time_ms() {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec * 1000.0 + ts.tv_nsec / 1000000.0;
}

int copy_with_bufsize(const char *src, const char *dst, size_t bufsize) {
    int fd_in = open(src, O_RDONLY);
    int fd_out = open(dst, O_WRONLY | O_CREAT | O_TRUNC, 0666);
    if (fd_in < 0 || fd_out < 0) { perror("open"); return -1; }

    char *buf = (char *)malloc(bufsize);
    if (!buf) { perror("malloc"); return -1; }

    ssize_t len;
    double start = get_time_ms();
    while ((len = read(fd_in, buf, bufsize)) > 0) {
        ssize_t written = 0;
        while (written < len) {
            ssize_t ret = write(fd_out, buf + written, len - written);
            if (ret < 0) { perror("write"); free(buf); return -1; }
            written += ret;
        }
    }
    double elapsed = get_time_ms() - start;
    printf("Bufsize: %10zu | Time: %8.2f ms\n", bufsize, elapsed);

    close(fd_in); close(fd_out); free(buf);
    return 0;
}

int main(int argc, char *argv[]) {
    if (argc != 3) { printf("Usage: %s <src> <dst>\n", argv[0]); return -1; }
    size_t bufsizes[] = {1, 16, 64, 256, 512, 1024,
                         4096, 8192, 16384, 65536, 262144};
    int n = sizeof(bufsizes) / sizeof(bufsizes[0]);
    for (int i = 0; i < n; i++) {
        char dst_name[256];
        snprintf(dst_name, sizeof(dst_name), "%s_%zu", argv[2], bufsizes[i]);
        copy_with_bufsize(argv[1], dst_name, bufsizes[i]);
    }
    return 0;
}
```

编译运行：

```bash
dd if=/dev/zero of=test_100mb.bin bs=1M count=100
gcc -o cp_bench cp_bench.c
./cp_bench test_100mb.bin result
```

预期结果：从 1 字节到 64KB，耗时急剧下降；64KB 之后趋于平稳。

---

## 4. 部分读写 (Partial Read/Write) 与信号中断处理

### 4.1 问题场景

`read()` 和 `write()` 返回的字节数**可能小于请求的字节数**，这称为部分读写（partial read/write）。常见原因：

1. **信号中断**：系统调用进行中收到信号，errno == EINTR
2. **文件末尾**：文件中剩余数据不足（read 场景）
3. **磁盘空间不足**：无法写入全部数据（write 场景）
4. **管道/Socket**：对端写入/读取速度不匹配

### 4.2 正确的处理方式

原始代码的问题：

```c
// 有问题 -- 没处理部分写入
while ((len = read(fd1, buf, sizeof(buf))) > 0) {
    write(fd2, buf, len);  // 可能只写入部分！
}
```

改进版本：处理信号中断和部分写入：

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <errno.h>

int main(int argc, char *argv[]) {
    int fd1, fd2;
    char buf[512];
    ssize_t len;

    if (argc != 3) {
        printf("Usage: %s <src> <dst>\n", argv[0]);
        return -1;
    }
    fd1 = open(argv[1], O_RDONLY);
    fd2 = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0666);
    if (fd1 < 0 || fd2 < 0) { perror("open"); return -1; }

    while (1) {
        len = read(fd1, buf, sizeof(buf));
        if (len < 0) {
            if (errno == EINTR) continue;  // 被信号中断，重试
            perror("read"); break;
        }
        if (len == 0) break;  // EOF

        // 处理部分写入
        ssize_t written = 0;
        while (written < len) {
            ssize_t ret = write(fd2, buf + written, len - written);
            if (ret < 0) {
                if (errno == EINTR) continue;
                perror("write");
                goto cleanup;
            }
            written += ret;
        }
    }

cleanup:
    close(fd1);
    close(fd2);
    return 0;
}
```

### 4.3 封装安全的 read/write 函数

```c
// 确保读取 count 字节（处理信号中断和部分读取）
ssize_t read_full(int fd, void *buf, size_t count) {
    size_t bytes_read = 0;
    while (bytes_read < count) {
        ssize_t ret = read(fd, (char *)buf + bytes_read,
                           count - bytes_read);
        if (ret < 0) { if (errno == EINTR) continue; return -1; }
        if (ret == 0) break;
        bytes_read += ret;
    }
    return bytes_read;
}

// 确保写入 count 字节（处理信号中断和部分写入）
ssize_t write_full(int fd, const void *buf, size_t count) {
    size_t bytes_written = 0;
    while (bytes_written < count) {
        ssize_t ret = write(fd, (const char *)buf + bytes_written,
                            count - bytes_written);
        if (ret < 0) { if (errno == EINTR) continue; return -1; }
        bytes_written += ret;
    }
    return bytes_written;
}
```

---

## 5. read/write 的原子性问题

### 5.1 什么是原子性

原子操作（atomic operation）是指**不可分割的操作** -- 要么完全执行，要么完全不执行，不存在中间状态。

### 5.2 普通文件的原子性

对于普通文件，单个 read/write **不保证原子性**。并发场景下，一个进程的 write 可能与另一个进程的 read 交叉：

```c
// 进程 A：write(fd, "AAAA", 4);
// 进程 B：write(fd, "BBBB", 4);
// 文件最终可能是：AAAABBBB、BBBBAAAA、AABBBBAA 等
```

POSIX 保证：管道上 <= PIPE_BUF（4096）字节的 write 是原子的。

### 5.3 保证原子性的方法

| 方法 | 适用场景 | 说明 |
|------|---------|------|
| `O_APPEND` | 追加写入 | open 时加 O_APPEND，偏移更新与写入是原子的 |
| `pwrite()` | 指定偏移写入 | 不改变文件偏移，是原子的 |
| `pread()` | 指定偏移读取 | 不改变文件偏移 |
| `fcntl()` 文件锁 | 通用 | 确保同一时间只有一个进程操作文件 |

### 5.4 O_APPEND 原子写入示例

```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main() {
    int fd = open("log.txt", O_WRONLY | O_CREAT | O_APPEND, 0666);
    if (fd < 0) { perror("open"); return -1; }
    // 多进程同时执行，写入不会交错
    const char *msg = "Process: Hello World\n";
    write(fd, msg, strlen(msg));
    close(fd);
    return 0;
}
```

### 5.5 pread/pwrite

```c
ssize_t pread(int fd, void *buf, size_t count, off_t offset);
ssize_t pwrite(int fd, const void *buf, size_t count, off_t offset);
```

优势：不修改文件偏移量，避免 lseek+read/write 的竞态条件，天然支持多线程。

---

## 6. 文本文件与二进制文件的读写差异

### 6.1 概念区分

| 特性 | 文本文件 | 二进制文件 |
|------|---------|-----------|
| 内容 | 可打印字符 + 换行符等 | 任意字节序列 |
| 换行符 | Unix: `\n` (0x0A), Windows: `\r\n` | 无行概念 |
| 存储 | 通常 UTF-8 或 ASCII | 按类型二进制表示 |
| 示例 | .txt, .c, .csv, .log | .bin, .jpg, .exe |

### 6.2 字节流视角

read/write 对文本和二进制一视同仁，只关心字节序列。读文本文件获得原始字节，需自行处理换行符。

### 6.3 逐行处理文本文件

```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>

#define BUF_SIZE 4096

int main(int argc, char *argv[]) {
    if (argc < 2) { printf("Usage: %s <file>\n", argv[0]); return -1; }
    int fd = open(argv[1], O_RDONLY);
    if (fd < 0) { perror("open"); return -1; }

    char buf[BUF_SIZE];
    int line_count = 0;
    ssize_t len;
    while ((len = read(fd, buf, sizeof(buf))) > 0) {
        for (ssize_t i = 0; i < len; i++)
            if (buf[i] == '\n') line_count++;
    }
    close(fd);
    printf("Total lines: %d\n", line_count);
    return 0;
}
```

### 6.4 结构体序列化

```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>

typedef struct { int id; char name[32]; double score; } Student;

int write_student(const char *fname, Student *s) {
    int fd = open(fname, O_WRONLY | O_CREAT | O_TRUNC, 0666);
    if (fd < 0) { perror("open"); return -1; }
    ssize_t ret = write(fd, s, sizeof(Student));
    close(fd);
    return (ret == sizeof(Student)) ? 0 : -1;
}

int read_student(const char *fname, Student *s) {
    int fd = open(fname, O_RDONLY);
    if (fd < 0) { perror("open"); return -1; }
    ssize_t ret = read(fd, s, sizeof(Student));
    close(fd);
    return (ret == sizeof(Student)) ? 0 : -1;
}

int main() {
    Student s1 = {1001, "Alice", 92.5}, s2;
    write_student("student.bin", &s1);
    read_student("student.bin", &s2);
    printf("id=%d, name=%s, score=%.1f\n", s2.id, s2.name, s2.score);
    return 0;
}
```

注意事项：跨平台可能因 padding/对齐/字节序不兼容；指针成员不能直接序列化。

---

## 7. 扩展实操示例

### 7.1 lseek + read/write 随机访问

```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>

int main() {
    int fd = open("seek_test.txt", O_RDWR | O_CREAT | O_TRUNC, 0666);
    if (fd < 0) { perror("open"); return -1; }

    // 写入 10 行
    for (int i = 0; i < 10; i++) {
        char line[16];
        int len = snprintf(line, sizeof(line), "Line %d\n", i);
        write(fd, line, len);
    }

    // 跳到第4行读取
    lseek(fd, 7 * 3, SEEK_SET);
    char buf[128];
    int len = read(fd, buf, sizeof(buf) - 1);
    buf[len] = '\0';
    printf("Read: %s", buf);

    // 回退并覆盖
    lseek(fd, -len, SEEK_CUR);
    write(fd, "NEW LINE\n", 9);

    // 末尾追加
    lseek(fd, 0, SEEK_END);
    write(fd, "APPENDED\n", 9);

    off_t size = lseek(fd, 0, SEEK_END);
    printf("File size: %ld\n", (long)size);
    close(fd);
    return 0;
}
```

### 7.2 稀疏文件

```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>

int main() {
    int fd = open("sparse.bin", O_RDWR | O_CREAT | O_TRUNC, 0666);
    if (fd < 0) { perror("open"); return -1; }

    // 开头写数据
    write(fd, "Data at start", 13);

    // 跳到 1MB 创建空洞
    lseek(fd, 1024 * 1024, SEEK_SET);
    write(fd, "Data at 1MB", 11);

    off_t size = lseek(fd, 0, SEEK_END);
    printf("Logical size: %ld bytes\n", (long)size);
    printf("(Actual disk usage much smaller!)\n");

    // 读取空洞区域
    lseek(fd, 512 * 1024, SEEK_SET);
    char buf[16] = {0};
    read(fd, buf, 16);
    printf("Hole data: ");
    for (int i = 0; i < 16; i++) printf("\\x%02x ", (unsigned char)buf[i]);
    printf("(all zeros)\n");

    close(fd);
    return 0;
}
```

验证：

```bash
du -h sparse.bin      # 4.0K（逻辑大小 1MB+，实际只占 4KB）
cp sparse.bin normal.bin
du -h normal.bin      # 1.1M（cp 会填补空洞）
```

### 7.3 strace 跟踪系统调用

```bash
gcc -o cp_simple cp_simple.c
strace -e trace=read,write,open,close ./cp_simple 1.txt 2.txt

# 输出示例：
open("1.txt", O_RDONLY)                = 3
open("2.txt", O_WRONLY|O_CREAT, 0666)  = 4
read(3, "12345678\n", 512)             = 9
write(4, "12345678\n", 9)              = 9
read(3, "", 512)                       = 0
close(3)                               = 0
close(4)                               = 0
```

strace 常用选项：

| 命令 | 说明 |
|------|------|
| `strace -e trace=read,write ./prog` | 只跟踪 read/write |
| `strace -c ./prog` | 统计调用次数和耗时 |
| `strace -o output.txt ./prog` | 输出到文件 |
| `strace -p <pid>` | 附加到运行中的进程 |

性能分析示例：

```bash
strace -c ./cp_simple large_file.bin output.bin
# 输出统计信息：系统调用次数、耗时、错误统计
```

---

## 8. 常见错误与调试方法

### 8.1 常见错误

| 错误 | 现象 | 原因 | 修复 |
|------|------|------|------|
| 忘记检查返回值 | 数据不完整 | 没检查 read/write 返回值 | 始终检查 |
| 用 sizeof(buf) 写入 | 目标文件有垃圾数据 | 写了未初始化缓冲区内容 | 用 len |
| 缓冲区溢出 | 段错误 | buf < count | 确保 buf 足够大 |
| 打开模式错误 | EBADF | O_WRONLY 却调 read | 检查 flags |
| 栈溢出 | 段错误 | 栈上 char buf[100MB] | 用 malloc |
| 忘记 close | fd 泄漏 | 循环 open 没 close | 用完即关 |
| 忽略 EINTR | 程序退出 | 被信号中断没重试 | 检查 errno |

### 8.2 调试技巧

```c
// 方法1：perror
int fd = open("nonexistent.txt", O_RDONLY);
if (fd < 0) perror("open failed");

// 方法2：strerror
#include <string.h>
#include <errno.h>
ssize_t ret = read(fd, buf, 1024);
if (ret < 0)
    fprintf(stderr, "read error: %s (errno=%d)\n",
            strerror(errno), errno);

// 方法3：assert
#include <assert.h>
int fd = open("test.txt", O_RDONLY);
assert(fd >= 0);
```

```bash
# strace 定位错误
strace -e trace=open,read,write ./my_program

# 检查 fd 泄漏
ls -la /proc/<PID>/fd/
lsof -p <PID>
```

### 8.3 生产级错误处理模板

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>

int cp_file(const char *src, const char *dst) {
    int fd_src = -1, fd_dst = -1, ret = -1;
    char *buf = NULL;

    fd_src = open(src, O_RDONLY);
    if (fd_src < 0) {
        fprintf(stderr, "open src: %s\n", strerror(errno));
        goto cleanup;
    }
    fd_dst = open(dst, O_WRONLY | O_CREAT | O_TRUNC, 0666);
    if (fd_dst < 0) {
        fprintf(stderr, "open dst: %s\n", strerror(errno));
        goto cleanup;
    }

    buf = malloc(65536);
    if (!buf) { perror("malloc"); goto cleanup; }

    ssize_t len;
    while ((len = read(fd_src, buf, 65536)) > 0) {
        ssize_t written = 0;
        while (written < len) {
            ssize_t w = write(fd_dst, buf + written, len - written);
            if (w < 0) {
                if (errno == EINTR) continue;
                fprintf(stderr, "write: %s\n", strerror(errno));
                goto cleanup;
            }
            written += w;
        }
    }
    if (len < 0) {
        fprintf(stderr, "read: %s\n", strerror(errno));
        goto cleanup;
    }
    ret = 0;

cleanup:
    free(buf);
    if (fd_src >= 0) close(fd_src);
    if (fd_dst >= 0) close(fd_dst);
    return ret;
}

int main(int argc, char *argv[]) {
    if (argc != 3) {
        printf("Usage: %s <src> <dst>\n", argv[0]);
        return EXIT_FAILURE;
    }
    if (cp_file(argv[1], argv[2]) < 0) return EXIT_FAILURE;
    printf("Copy done: %s -> %s\n", argv[1], argv[2]);
    return EXIT_SUCCESS;
}
```

---

## 9. 实验：实现文件复制功能（cp命令）

### 9.1 需求分析

实现类似 `cp` 命令的功能，将一个文件的内容复制到另一个文件：

```bash
./copy src.txt dst.txt
```

### 9.2 实现步骤

```
1. 打开源文件（只读）
2. 创建目标文件（只写 | O_CREAT，权限 0666）
3. 循环：read -> write
4. read 返回 0（EOF）-> 退出
5. 关闭两个文件
```

### 9.3 完整代码

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main(int argc, char *argv[])
{
    int fd1, fd2;
    char buf[512];
    int len;

    if (argc != 3) {
        printf("Usage: %s <src> <dst>\n", argv[0]);
        return -1;
    }

    fd1 = open(argv[1], O_RDONLY);
    fd2 = open(argv[2], O_WRONLY | O_CREAT, 0666);

    if (fd1 < 0 || fd2 < 0) {
        printf("open error\n");
        return -1;
    }

    while (1) {
        len = read(fd1, buf, sizeof(buf));
        if (len == 0) break;
        write(fd2, buf, len);
    }

    close(fd1);
    close(fd2);
    return 0;
}
```

### 9.4 编译与运行

```bash
make
echo "12345678" > 1.txt
./copy 1.txt 2.txt
cat 2.txt        # 输出：12345678
```

### 9.5 关键点

| 要点 | 说明 |
|------|------|
| 缓冲区大小 | 512 字节（可调整） |
| 循环条件 | `read()` 返回 0 = EOF |
| 写入大小 | 用 `len`，不是 `sizeof(buf)` |
| 权限 | 0666 = 所有用户可读写 |

---

## 本讲总结

| 函数 | 头文件 | 原型 | 成功返回 | 失败 |
|------|--------|------|---------|------|
| `read()` | `<unistd.h>` | `read(fd, buf, count)` | 实际字节数 | -1 |
| `write()` | `<unistd.h>` | `write(fd, buf, count)` | 实际字节数 | -1 |

**文件复制核心模式**：

```c
while ((len = read(src_fd, buf, sizeof(buf))) > 0) {
    write(dst_fd, buf, len);
}
```

### 知识体系总结

| 主题 | 关键要点 |
|------|---------|
| 返回值检查 | 区分 EOF(0)、部分读写(>0,<count)、错误(-1) |
| 信号中断 | errno == EINTR 时重试 |
| 部分写入 | 循环写直到写完所有数据 |
| 缓冲区大小 | 4KB~64KB 折中，栈上分配不要超过数 MB |
| 原子性 | O_APPEND 保证追加原子性；pread/pwrite 避免竞态 |
| 错误处理 | perror/strerror 打印错误信息 |
| 调试工具 | strace 跟踪，/proc/<pid>/fd 检查泄漏 |

---

*笔记生成日期：2026-07-22 | 转写：Whisper tiny*
*笔记优化日期：2026-07-24 | 优化：补充实操示例、性能分析、错误处理、原子性等章节*
