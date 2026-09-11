---
title: "第43讲：lseek 与 sync 函数"
date: 2026-09-11T09:01:00+08:00
draft: false
description: "#include <unistd.h>"
series: ["Linux 入门"]
series_order: 26
categories: ["技术笔记"]
tags: ["Linux", "文件IO"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P44)
> **标题**：第43讲 — lseek 与 sync 函数
> **时长**：约13分钟
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. `lseek()` 函数

### 1.1 头文件

```c
#include <unistd.h>
```

### 1.2 函数原型

```c
off_t lseek(int fd, off_t offset, int whence);
```

| 参数 | 说明 |
|------|------|
| `fd` | 文件描述符 |
| `offset` | 偏移量（可为正或负） |
| `whence` | 基准位置（决定 offset 从哪开始计算） |

**关于 `off_t` 类型**：

`off_t` 是一个有符号整数类型，用于表示文件偏移量。在 32 位系统上通常是 `long`（4 字节），在 64 位系统上通常是 `long`（8 字节）。如果需要支持超过 2GB 的大文件，编译时需要定义 `_FILE_OFFSET_BITS=64` 宏，或在代码中显式使用 `lseek64()`。

```c
// 编译时启用大文件支持
// gcc -D_FILE_OFFSET_BITS=64 -o prog prog.c
```

### 1.3 三种基准位置

| 宏 | 含义 | 最终位置 |
|----|------|---------|
| `SEEK_SET` | 文件**开头** | `0 + offset` |
| `SEEK_CUR` | **当前**位置 | `当前位置 + offset` |
| `SEEK_END` | 文件**末尾** | `文件大小 + offset` |

**使用示例**：

```c
// 定位到文件开头
lseek(fd, 0, SEEK_SET);

// 从当前位置前移 10 字节
lseek(fd, 10, SEEK_CUR);

// 定位到文件末尾前 5 字节
lseek(fd, -5, SEEK_END);
```

### 1.4 返回值

| 返回值 | 含义 |
|--------|------|
| **非负整数** | 成功，返回偏移后的位置值（相对于文件开头的字节数） |
| **-1** | 失败，errno 被设置 |

**注意**：返回值可能是 0，表示文件开头位置，所以不能用 `> 0` 来判断成功。正确的判断方式是 `!= -1` 或 `>= 0`。

```c
// 错误判断
if (lseek(fd, 0, SEEK_SET) > 0) {  /* 错！偏移 0 时无法进入此分支 */ }

// 正确判断
if (lseek(fd, 0, SEEK_SET) != -1) {  /* 正确 */ }
if (lseek(fd, 0, SEEK_SET) >= 0)    {  /* 正确 */ }
```

**常见的 errno 值**：

| errno | 含义 |
|-------|------|
| `EBADF` | fd 不是有效的文件描述符 |
| `EINVAL` | whence 参数无效，或 offset 导致越界 |
| `ESPIPE` | fd 关联的是管道、socket 或 FIFO（不可定位） |
| `EOVERFLOW` | 结果超出 off_t 可表示范围 |

### 1.5 使用 lseek 获取文件大小

`lseek()` 经常被用来快速获取文件大小，而不需要读取文件内容：

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    int fd = open("test.txt", O_RDONLY);

    // lseek(fd, 0, SEEK_END) 返回文件末尾相对于开头的偏移量
    // 即文件大小（字节数）
    off_t size = lseek(fd, 0, SEEK_END);

    if (size != -1) {
        printf("文件大小: %ld 字节\n", (long)size);
    } else {
        perror("lseek");
    }

    close(fd);
    return 0;
}
```

另一种写法（更安全，不会改变文件偏移量）：

```c
// 先保存当前位置，再定位到末尾获取大小，最后恢复位置
off_t current = lseek(fd, 0, SEEK_CUR);  // 保存当前位置
off_t size    = lseek(fd, 0, SEEK_END);  // 定位到末尾获取大小
lseek(fd, current, SEEK_SET);            // 恢复到原来位置

printf("文件大小: %ld 字节\n", (long)size);
```

### 1.6 lseek 对不支持随机访问的文件

`lseek()` 只能用于支持随机访问（seekable）的文件，如普通文件和块设备。对于以下类型的文件描述符，`lseek()` 会返回 -1 并设置 `errno = ESPIPE`：

- **管道（pipe）**：`pipe()` 创建的匿名管道
- **FIFO（命名管道）**：`mkfifo()` 创建的命名管道
- **Socket**：`socket()` 创建的网络套接字
- **终端设备**：`/dev/tty` 等字符设备

```c
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>

int main() {
    int pipefd[2];

    if (pipe(pipefd) == -1) {
        perror("pipe");
        return 1;
    }

    // 尝试对管道写端执行 lseek
    off_t ret = lseek(pipefd[1], 0, SEEK_SET);
    if (ret == -1) {
        printf("lseek on pipe failed: %s\n", strerror(errno));
        // 输出: lseek on pipe failed: Illegal seek
        // errno == ESPIPE
    }

    close(pipefd[0]);
    close(pipefd[1]);
    return 0;
}
```

对于这些不可定位的文件，数据只能按流的方式顺序读写，无法跳转到特定位置。

---

## 2. 实验：lseek 制造文件空洞

### 2.1 实验代码

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main() {
    int fd;

    // 创建文件 test.txt（读写|创建，权限 0666）
    fd = open("test.txt", O_RDWR | O_CREAT, 0666);

    // ① 写入 "123"（3字节）
    write(fd, "123", 3);

    // ② 从当前位置偏移 100 字节
    lseek(fd, 100, SEEK_CUR);

    // ③ 再写入 "ABC"（3字节）
    write(fd, "ABC", 3);

    close(fd);
    return 0;
}
```

### 2.2 运行结果

```bash
$ make
$ ./test_lseek

# 查看文件大小
$ ls -l test.txt
-rw-r--r-- 1 root root 106 Jul 22 12:00 test.txt
```

**文件大小 = 3 + 100 + 3 = 106 字节**

### 2.3 分析

```
文件内容图示：
[1][2][3][ ][ ][ ]...[ ][A][B][C]
 0  1  2  3 ...      102 103 104 105
         ↑ 100个空字节（NUL字符 0x00）↑
         （文件空洞，hole）
```

- 偏移的 100 字节构成了**文件空洞**（hole）
- 空洞在磁盘上并不实际占用物理空间（稀疏文件）
- 读取空洞区域时，内核返回全零字节（`\0`），而不是实际读取磁盘
- 用 `vi` 打开查看时，空洞位置的 NUL 字符（`0x00`）会显示为 `^@`

### 2.4 用 od 工具查看空洞

`od`（octal dump）可以以十六进制或八进制形式查看文件的实际内容字节：

```bash
# 以十六进制查看文件内容
$ od -A x -t x1z -v test.txt
000000 31 32 33 00 00 00 00 00 00 00 00 00 00 00 00 00  >123.............<
000010 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  >................<
...
000060 00 00 00 00 00 00 00 00 41 42 43                 >........ABC<
00006a
```

| 参数 | 含义 |
|------|------|
| `-A x` | 地址以十六进制显示 |
| `-t x1z` | 每个字节以十六进制显示，右侧附带可打印字符 |
| `-v` | 显示所有数据（不省略重复行） |

可以看到，偏移 3 到 102 之间的 100 个字节全是 `00`（NUL），这些就是文件空洞。

---

## 3. 文件空洞与稀疏文件详解

### 3.1 什么是稀疏文件（Sparse File）

稀疏文件是一种特殊的文件：**文件的逻辑大小大于它在磁盘上实际占用的物理空间**。空洞区域不会被分配实际的磁盘块，只有当数据被实际写入时，文件系统才会分配磁盘空间。

```
逻辑视图（应用程序看到的样子）：
┌────┬─────────────────────────────┬────┐
│123 │  100 字节的空洞（全 0x00）    │ABC │
└────┴─────────────────────────────┴────┘
                  共 106 字节

物理存储（磁盘上实际存放）：
┌────┐                           ┌────┐
│123 │    [没有分配任何磁盘块]      │ABC │
└────┘                           └────┘
       3 字节物理占用                   3 字节物理占用
                   共 6 字节物理空间（可能经过块对齐后会多一些）
```

### 3.2 du 与 ls：逻辑大小 vs 物理占用

这是理解稀疏文件最关键的一组对比工具：

| 命令 | 显示 | 含义 |
|------|------|------|
| `ls -l` | 逻辑大小（apparent size） | 程序可见的文件大小，等于 `lseek(fd, 0, SEEK_END)` |
| `du -h` | 物理占用（disk usage） | 文件在磁盘上实际占用的空间 |
| `ls -s` | 物理块数（block count） | 以文件系统块为单位的物理占用 |

```bash
$ ls -l test.txt
-rw-r--r-- 1 root root 106 Jul 22 12:00 test.txt  # 逻辑大小 106

$ du -h test.txt
4.0K    test.txt                                    # 物理占用约 4K（一个文件系统块）

$ ls -ls test.txt
4 -rw-r--r-- 1 root root 106 Jul 22 12:00 test.txt  # 第一列=4个块（通常每块512字节）
```

**为什么物理占用是 4K 而不是 6 字节？** 因为文件系统以块（block）为单位分配空间，通常一个块是 4KB。即使只写了 3 字节，也会分配一整个块。

### 3.3 让空洞更明显

创建一个更大的空洞文件，让差异更加明显：

```bash
# 使用 dd 命令创建稀疏文件
$ dd if=/dev/zero of=sparse.bin bs=1 count=1 seek=100M
1+0 records in
1+0 records out
1 byte copied

$ ls -lh sparse.bin
-rw-r--r-- 1 root root 101M Jul 22 12:00 sparse.bin   # 逻辑大小 ~101MB

$ du -h sparse.bin
4.0K    sparse.bin                                     # 物理占用仅 4KB
```

这里 `dd seek=100M` 让 dd 跳过前 100MB 再写 1 字节，制造了一个巨大的空洞。

### 3.4 用程序创建稀疏文件

```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main() {
    int fd = open("sparse.dat", O_RDWR | O_CREAT | O_TRUNC, 0666);

    // 在偏移 1GB 处写入 1 字节
    lseek(fd, 1024 * 1024 * 1024, SEEK_SET);  // 跳转到 1GB 位置
    write(fd, "X", 1);                          // 写入 1 字节

    // 获取文件大小
    off_t size = lseek(fd, 0, SEEK_END);
    printf("逻辑大小: %.2f GB\n", (double)size / (1024 * 1024 * 1024));

    close(fd);
    return 0;
}
```

运行后，`ls -lh sparse.dat` 会显示约 1GB，但 `du -h sparse.dat` 只会显示几 KB。

### 3.5 稀疏文件的常见用途

| 场景 | 说明 |
|------|------|
| **虚拟机磁盘镜像** | qcow2/raw 格式的磁盘镜像往往包含大量空洞 |
| **数据库文件** | 某些数据库预分配大文件但逐步填充 |
| **BT 下载** | 预分配文件空间，按块下载填充 |
| **日志文件** | 某些日志系统预先创建固定大小的日志文件 |
| **core dump** | 核心转储文件可能只包含部分地址空间 |

### 3.6 检测稀疏文件

```bash
# 比较逻辑大小和物理占用
$ stat test.txt
  File: `test.txt`
  Size: 106             Blocks: 8          IO Block: 4096   regular file
# Size=逻辑大小(Bytes), Blocks=512字节块的数量
# 如果 Size > Blocks * 512，说明存在空洞

# 用 find 查找稀疏文件
$ find . -type f -size +0 -printf `%S\t%p\n` | awk `$1 < 1 {print}`
# %S 是稀疏度（物理占用/逻辑大小），值小于1表示有空洞
```

---

## 4. Linux 的页缓存机制（Page Cache）

### 4.1 为什么需要页缓存？

```
用户空间                          内核空间                      磁盘
┌──────────┐     write()    ┌─────────────┐    后台回写    ┌────────┐
│   Buffer  │ ───────────→  │  Page Cache  │ ───────────→  │  磁盘   │
│ (应用程序) │              │  (页缓存)     │              │ (物理)  │
└──────────┘               └─────────────┘              └────────┘
                   read()        ↑
                ←───────────────┘
```

磁盘 I/O 比内存操作慢几个数量级（毫秒 vs 纳秒）。页缓存是内核在内存中维护的一块区域，用来缓存最近访问过的文件数据，以避免每次都去读慢速磁盘。

### 4.2 工作机制

| 操作 | 流程 |
|------|------|
| **read()** | ① 先去 page cache 找 → ② 找到则直接返回（缓存命中） → ③ 没找到则从磁盘读到 cache → 再返回 |
| **write()** | ① 数据先写入 page cache → ② 等到 cache 满了或超时 → ③ 内核才写回磁盘 |

**关键概念：脏页（Dirty Page）**

被修改过但尚未写回磁盘的缓存页称为**脏页**（dirty page）。脏页既在内存中，又代表磁盘上需要更新的数据。

```
页缓存中的页有两种状态：

  clean page:  缓存的数据与磁盘一致 ← 可以安全回收
  dirty page:  缓存的数据比磁盘新   ← 必须先写回磁盘才能回收
```

### 4.3 问题

`write()` 返回成功后，数据可能还在 page cache 中，并未真正写入磁盘。

如果此时系统崩溃或断电 → **数据丢失**。

### 4.4 脏页回写策略

Linux 内核不是等到用户调用 `sync()` 才回写脏页，而是有一套自动回写机制。回写由内核线程（历史上的 pdflush，现在由每个块设备独立的 `flush-*` 线程）负责。

**触发回写的条件**：

| 触发条件 | 内核参数 | 说明 |
|----------|---------|------|
| 脏页比例过高 | `dirty_background_ratio`（默认10%） | 当脏页占总内存比例超过此阈值时，后台开始回写 |
| 脏页比例即将爆满 | `dirty_ratio`（默认20%） | 当脏页比例超过此阈值时，write() 调用者自身会阻塞并参与回写 |
| 脏页存活过久 | `dirty_expire_centisecs`（默认3000=30秒） | 脏页在内存中停留超过此时间后，在下次回写时被写出 |
| 定期回写 | `dirty_writeback_centisecs`（默认500=5秒） | flush 线程每隔此时间唤醒一次，检查是否有脏页需要回写 |

查看当前系统的脏页参数：

```bash
$ sysctl -a | grep dirty
vm.dirty_background_ratio = 10
vm.dirty_ratio = 20
vm.dirty_expire_centisecs = 3000
vm.dirty_writeback_centisecs = 500
vm.dirty_background_bytes = 0
vm.dirty_bytes = 0

# 或者从 /proc 查看
$ cat /proc/sys/vm/dirty_background_ratio
10
```

**回写流程示意**：

```
应用程序 write() ──→ 标记 page cache 页为 dirty ──→ write() 返回
                                                         │
         ┌───────────────────────────────────────────────┘
         │  (异步，由内核触发)
         ▼
┌─────────────────┐
│  flush-* 线程   │  每隔 dirty_writeback_centisecs 唤醒
│  (每块设备一个)  │  检查脏页数量和年龄
└────────┬────────┘
         │ 满足回写条件
         ▼
    将脏页写入磁盘 → 标记为 clean
```

### 4.5 内核回写线程的演进

| 内核版本 | 回写线程 | 特点 |
|----------|---------|------|
| 2.6 早期 | `pdflush` | 全局线程池，最多 2-8 个线程，所有设备共用 |
| 2.6.32+ | `flush-<major>:<minor>` | 每个块设备一个独立线程，隔离性更好 |

可以通过以下命令看到回写线程：

```bash
$ ps aux | grep flush
root      123  0.0  0.0      0     0 ?        S    Jul22   0:00 [flush-8:0]
# flush-8:0 表示主设备号 8、次设备号 0 的块设备（通常是 /dev/sda）
```

---

## 5. `sync()` 函数

### 5.1 作用

**将 page cache 中的数据强制写回磁盘**，确保数据落盘。

注意：`sync()` 是**发起**回写请求，现代 Linux 内核的 `sync()` 实现会等待所有脏数据真正写入磁盘后才返回（早期版本不等待）。

### 5.2 函数原型

```c
#include <unistd.h>

void sync(void);
```

| 参数 | 返回值 |
|------|--------|
| 无参数 | 无返回值（总是成功，没有错误返回） |

### 5.3 使用场景

```c
write(fd, buf, len);   // 写入数据（可能还在 cache 中）
sync();                 // 强制将 cache 写入磁盘
```

当一个程序对数据安全性要求较高时，在 `write()` 之后调用 `sync()`：

```c
int main() {
    int fd = open("data.txt", O_RDWR | O_CREAT, 0666);

    write(fd, "important data", 14);

    // 确保数据立即写回磁盘
    sync();

    close(fd);
    return 0;
}
```

### 5.4 注意

- `sync()` 是一个**全局**操作，会将所有文件的数据写回磁盘
- 如果只同步单个文件，可以使用 `fsync(fd)`（更精确）
- 频繁调用 `sync()` 会严重降低系统性能，因为它强制所有脏数据立即落盘，抵消了页缓存的优势

---

## 6. 数据同步函数家族对比

`sync()` 只是数据同步系列函数中的一个。Linux 提供了四个不同粒度的同步函数，根据场景选择合适的函数可以兼顾性能和数据安全。

### 6.1 `sync()` — 全局同步

```c
#include <unistd.h>
void sync(void);
```

- 将所有已修改的**文件数据和元数据**写回所有挂载的文件系统
- 作用范围：整个系统的所有文件
- 现代实现会**阻塞等待**直到数据真正写入存储设备
- 开销最大，影响面最广

### 6.2 `syncfs()` — 单文件系统同步

```c
#define _GNU_SOURCE         /* 需要定义以使用 syncfs */
#include <unistd.h>
int syncfs(int fd);
```

- 只同步 `fd` 所属的**整个文件系统**
- 返回 0 成功，-1 失败
- 比 `sync()` 粒度细，比 `fsync()` 覆盖广
- 适用于你知道需要同步某个挂载点上的所有文件，但不影响其他磁盘的场景
- Linux 2.6.39 引入

### 6.3 `fsync()` — 单文件同步（数据 + 元数据）

```c
#include <unistd.h>
int fsync(int fd);
```

- 将 `fd` 对应的**文件数据和元数据**（如文件大小、修改时间等 inode 信息）写回磁盘
- 返回 0 成功，-1 失败
- 只影响一个文件，是数据库等场景最常用的同步方式
- 比 `sync()` 更精确、开销更小

### 6.4 `fdatasync()` — 单文件同步（仅数据）

```c
#include <unistd.h>
int fdatasync(int fd);
```

- 只将 `fd` 对应的**文件数据**写回磁盘
- **不同步**那些不影响数据读取的元数据（如 atime、mtime）
- 只在文件大小发生变化等必要时才同步必要的元数据
- 比 `fsync()` 更快，因为少了一次元数据写入

### 6.5 四函数对比

| 函数 | 作用范围 | 同步内容 | 返回值 | 引入版本 |
|------|---------|---------|--------|---------|
| `sync()` | 全局所有文件系统 | 所有脏数据 + 元数据 | void（总是成功） | 早期 Unix |
| `syncfs(fd)` | fd 所在文件系统 | 该 FS 所有脏数据 + 元数据 | 0 成功 / -1 失败 | Linux 2.6.39 |
| `fsync(fd)` | 单个文件 | 该文件数据 + 元数据 | 0 成功 / -1 失败 | POSIX.1b |
| `fdatasync(fd)` | 单个文件 | 该文件数据（+必要元数据） | 0 成功 / -1 失败 | POSIX.1b |

**性能开销排序**（从高到低）：

```
sync()  >  syncfs()  >  fsync()  >  fdatasync()
```

### 6.6 实际使用对比示例

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int main() {
    int fd = open("test.db", O_RDWR | O_CREAT, 0666);

    write(fd, "database record", 15);

    // 场景1：对安全性要求极高 → 使用 fsync
    fsync(fd);     // 数据和元数据都落盘

    // 场景2：只关心数据不关心时间戳 → 使用 fdatasync
    // fdatasync(fd);  // 只保证数据落盘，更快

    // 场景3：粗暴简单 → 使用 sync
    // sync();  // 所有文件都刷盘，开销大

    close(fd);
    return 0;
}
```

---

## 7. O_SYNC 模式与 sync() 调用的区别

### 7.1 两种同步方式

Linux 提供了两种让数据同步落盘的方式：

| 方式 | 机制 | 粒度 |
|------|------|------|
| **O_SYNC 标志** | `open()` 时设置，每次 `write()` 调用自动同步 | 每次写操作 |
| **sync/fsync 调用** | `write()` 之后手动调用同步函数 | 由程序员决定同步时机 |

### 7.2 O_SYNC 打开模式

```c
// 以同步方式打开文件
int fd = open("data.txt", O_RDWR | O_CREAT | O_SYNC, 0666);

// 每次 write() 都会等待数据写入磁盘后才返回
write(fd, buf, 1024);  // 阻塞直到数据落盘
write(fd, buf, 1024);  // 每写一次就刷一次盘
```

### 7.3 两种方式对比

```c
// 方式A：O_SYNC —— 每次写都同步（性能最差）
int fd_a = open("data.txt", O_WRONLY | O_CREAT | O_SYNC, 0666);
for (int i = 0; i < 1000; i++) {
    write(fd_a, &i, sizeof(i));  // 每次 write 都触发磁盘 I/O，共1000次
}
close(fd_a);

// 方式B：普通模式 + 最后 sync —— 累积后再同步（性能好）
int fd_b = open("data.txt", O_WRONLY | O_CREAT, 0666);
for (int i = 0; i < 1000; i++) {
    write(fd_b, &i, sizeof(i));  // 数据先写入 page cache
}
fsync(fd_b);  // 只在最后同步一次
close(fd_b);
```

| 对比维度 | O_SYNC | 普通模式 + fsync() |
|----------|--------|---------------------|
| 写性能 | 差（每次写都触发磁盘 I/O） | 好（批量回写，减少磁盘寻道） |
| 数据安全性 | 每条记录都实时落盘 | 只在调用 fsync 后才保证落盘 |
| 编程复杂度 | 低（打开时设置即可） | 中（需要记得调用 fsync） |
| 适用场景 | 要求每条记录都绝对不能丢 | 可以容忍批次内的少量数据丢失 |

### 7.4 O_DSYNC 标志

还有一个折中方案 `O_DSYNC`，只同步数据，不同步不影响读取的元数据：

```c
int fd = open("data.txt", O_RDWR | O_CREAT | O_DSYNC, 0666);
// 每次 write 保证数据落盘，但不保证 atime/mtime 等元数据落盘
```

`O_DSYNC` 相当于 `O_SYNC` 但只做 `fdatasync()` 级别的工作。

---

## 8. 数据安全性场景实战指南

### 8.1 场景分类

根据对数据丢失的容忍度，常见应用场景可分为以下几类：

| 场景 | 数据丢失容忍度 | 推荐方案 | 说明 |
|------|--------------|---------|------|
| **数据库事务日志** | 零容忍 | `fsync()` / `fdatasync()` 每次提交后调用 | 事务提交时必须保证日志落盘 |
| **配置文件写入** | 零容忍 | `write()` → `fsync()` → `close()` | 配置写错可能导致系统无法启动 |
| **应用日志** | 低容忍（可丢最后几行） | 普通 write + 定时 sync | 日志丢了影响排查但不影响业务 |
| **临时计算结果** | 可容忍 | 普通 write | 丢了可以重新计算 |
| **流媒体写入** | 高容忍 | 普通 write | 丢几帧不影响整体体验 |
| **缓存文件** | 完全可容忍 | 普通 write | 本来就是可以重建的 |

### 8.2 数据库场景：正确的写入顺序

```c
// 模拟数据库的 write-ahead logging（WAL）
int log_fd = open("wal.log", O_RDWR | O_CREAT, 0666);
int db_fd  = open("data.db", O_RDWR | O_CREAT, 0666);

// 1. 先写日志
write(log_fd, log_entry, log_len);
fdatasync(log_fd);  // 确保日志先落盘

// 2. 再写数据
write(db_fd, data_entry, data_len);
fdatasync(db_fd);   // 确保数据落盘

// 3. 最后标记日志已应用（截断或写 checkpoint）
write(log_fd, checkpoint, cp_len);
fsync(log_fd);

close(log_fd);
close(db_fd);
```

这个顺序保证了：如果崩溃发生在步骤 1 之后、步骤 2 之前，恢复时可以根据日志重做；如果崩溃发生在步骤 2 之后、步骤 3 之前，恢复时可以识别 checkpoint 判断数据已落盘。

### 8.3 配置文件场景：原子写入

```c
// 原子写入配置文件（通过临时文件 + rename）
int tmp_fd = open("config.tmp", O_WRONLY | O_CREAT | O_TRUNC, 0666);

write(tmp_fd, new_config, config_len);
fsync(tmp_fd);   // 先确保临时文件数据落盘
close(tmp_fd);

rename("config.tmp", "config.conf");  // 原子替换
// rename 是原子操作，不会出现部分写入的文件

// rename 后还需要同步目录，确保目录元数据落盘
int dir_fd = open(".", O_RDONLY);
fsync(dir_fd);   // 同步目录的元数据
close(dir_fd);
```

**为什么需要 `fsync()` 目录？** `rename()` 修改的是目录的元数据（目录项），而不是文件本身。如果只有文件数据落盘而目录元数据没落盘，崩溃后可能导致目录找不到新文件。

### 8.4 性能与安全的权衡

```c
// 方案A：最高安全性（数据库常用）
write(fd, buf, len);
fsync(fd);          // 每次写都同步，性能最差

// 方案B：批量提交（推荐）
write(fd, buf1, len1);
write(fd, buf2, len2);
write(fd, buf3, len3);
fsync(fd);          // 多条记录写完后一次同步

// 方案C：定时同步（日志场景）
write(fd, buf, len);
// 不立即 sync，依赖内核脏页回写机制（默认 30s 内落盘）

// 方案D：纯性能（临时数据）
write(fd, buf, len);
// 不调用任何同步函数
```

### 8.5 误区澄清

**误区 1："close() 会自动 sync"**

`close()` **不会**主动发起 fsync。关闭文件描述符只是释放内核数据结构，数据仍然在 page cache 中，由内核后台线程负责回写。

```c
write(fd, "data", 4);
close(fd);  // 此时数据并未保证落盘！
// 如果此时断电，数据可能丢失
```

**误区 2："fclose() 会自动 fsync"**

标准 C 库的 `fclose()` 会调用 `fflush()` 将 stdio 缓冲区的数据通过 `write()` 写入内核，但同样不会调用 `fsync()`。数据还是在 page cache 中。

```c
FILE *fp = fopen("test.txt", "w");
fprintf(fp, "hello");
fclose(fp);  // fflush → write → close，没有 fsync！
```

**误区 3："SSD 写入很快，不需要关心 sync"**

SSD 也有写缓存（DRAM cache），掉电时缓存中的数据同样会丢失。企业级 SSD 有电容保护（PLP, Power Loss Protection），但消费级 SSD 通常没有。

### 8.6 选择指南决策树

```
需要保证数据不丢失？
  ├── 是 → 数据量很大？
  │        ├── 是 → 用 fdatasync()，每次提交后调用
  │        └── 否 → 用 fsync()，写入后立即调用
  └── 否 → 是否可以接受丢失最近几秒的数据？
           ├── 可以 → 依赖内核自动回写（默认 ~30s），不做额外同步
           └── 不行 → 用 fdatasync()，定时（如每 5s）调用一次
```

---

## 9. 综合实验：sync 相关函数的实际效果

### 9.1 实验：验证 close() 不会自动 fsync

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int main() {
    // 1. 写入数据并 close（不 fsync）
    int fd1 = open("file1.txt", O_RDWR | O_CREAT | O_TRUNC, 0666);
    write(fd1, "data without fsync", 18);
    close(fd1);  // 此时数据可能还在 page cache 中
    printf("file1 written and closed (no fsync)\n");

    // 2. 写入数据，fsync 后再 close
    int fd2 = open("file2.txt", O_RDWR | O_CREAT | O_TRUNC, 0666);
    write(fd2, "data with fsync", 15);
    fsync(fd2);   // 强制落盘
    close(fd2);
    printf("file2 written, fsynced, and closed\n");

    // 3. 用 O_SYNC 打开（每次 write 自动同步）
    int fd3 = open("file3.txt", O_RDWR | O_CREAT | O_TRUNC | O_SYNC, 0666);
    write(fd3, "data with O_SYNC", 16);
    close(fd3);
    printf("file3 written with O_SYNC\n");

    // 观察：在系统崩溃/断电前执行 sync() 验证数据是否都存在
    // sync();

    return 0;
}
```

### 9.2 实验：用 shell 命令验证 sync 行为

```bash
# 写入数据并查看 page cache 状态
$ echo "test data" > testfile

# 查看文件在 page cache 中的状态
$ cat /proc/meminfo | grep -i dirty
Dirty:               256 kB      # 脏页大小

# 手动触发 sync
$ sync

# 再次查看
$ cat /proc/meminfo | grep -i dirty
Dirty:                 0 kB      # 脏页被清空
```

---

## 本讲总结

### 函数速查

| 函数 | 功能 | 返回值 |
|------|------|--------|
| `lseek(fd, offset, whence)` | 设置文件读写位置 | 成功→偏移位置（非负整数），失败→-1 |
| `sync()` | 强制所有 page cache 写回磁盘 | void（无返回值，总是成功） |
| `fsync(fd)` | 强制单文件数据+元数据写回磁盘 | 成功→0，失败→-1 |
| `fdatasync(fd)` | 强制单文件数据写回磁盘 | 成功→0，失败→-1 |
| `syncfs(fd)` | 强制单文件系统所有数据写回磁盘 | 成功→0，失败→-1 |

### lseek 三个基准

- `SEEK_SET` → 开头 + offset
- `SEEK_CUR` → 当前位置 + offset
- `SEEK_END` → 末尾 + offset

### 文件空洞

lseek 跳过中间未写入的区域，形成文件空洞。空洞不占磁盘物理空间（稀疏文件）。`ls -l` 显示的是逻辑大小，`du -h` 显示的是实际物理占用。用 `stat` 命令的 Size 和 Blocks 对比可以判断文件是否稀疏。

### Page Cache

用户数据 `write()` 后先到内核 page cache，成为脏页。内核 flush 线程会根据脏页比例（`dirty_background_ratio`）和脏页年龄（`dirty_expire_centisecs`）自动回写磁盘。手动调用 `sync()` / `fsync()` 可以强制立即回写。

### 同步策略选择

| 策略 | 性能 | 安全性 | 典型场景 |
|------|------|--------|---------|
| 不使用任何 sync | 最高 | 最低 | 临时文件、缓存 |
| 依赖内核自动回写 | 高 | 中 | 普通日志 |
| fdatasync 批量提交 | 中 | 中高 | 数据库（侧重数据） |
| fsync 每条提交 | 低 | 高 | 数据库（侧重完整性） |
| O_SYNC 模式 | 最低 | 最高 | 关键配置、金融交易 |

---

*笔记生成日期：2026-07-22 | 转写：Whisper tiny | 优化日期：2026-07-24*
