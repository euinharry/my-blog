---
title: "第44讲：标准IO编程"
date: 2026-09-11T09:00:00+08:00
draft: false
description: "系统 IO（open/read/write）每次调用都需要用户态 ↔ 内核态切换："
series: ["Linux 入门"]
series_order: 27
categories: ["技术笔记"]
tags: ["Linux", "文件IO", "标准IO"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P45)
> **标题**：第44讲 -- 标准IO编程
> **时长**：约8分钟
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. 系统IO vs 标准IO

### 1.1 系统IO的问题

系统 IO（`open`/`read`/`write`）每次调用都需要**用户态 ↔ 内核态切换**：

```
用户程序调用 write(fd, buf, 2)   ← 只写2个字节
    ↓
用户态 → 内核态切换               ← 每次切换开销很大
    ↓
内核执行系统调用
    ↓
内核态 → 用户态切换
    ↓
返回用户程序
```

如果频繁读写少量数据（如每次 1~2 字节），系统调用开销占比极高，性能下降。

**定量分析**：一次用户态/内核态切换的开销通常在几百到几千个 CPU 周期。如果每次只写 2 字节，切换开销可能比实际数据传输开销大数十倍。例如写 1MB 数据：

- 每次写 1 字节，需要 1,048,576 次系统调用，切换开销累积可达数十毫秒
- 每次写 4096 字节，只需 256 次系统调用，开销大幅减少

### 1.2 标准IO的解决思路

标准 IO（C 库函数）在**用户空间也开辟一个缓冲区**（IO缓冲区）：

```
用户程序
    ↓  fwrite()/fread()
┌─────────────────────┐
│     IO缓冲区          │  ← 用户空间（stdio库）
│  (攒够数据才一次提交)  │
└─────────┬───────────┘
          ↓  一次批量系统调用
┌─────────────────────┐
│   Page Cache（页缓存） │  ← 内核空间
└─────────┬───────────┘
          ↓
┌─────────────────────┐
│       磁盘           │
└─────────────────────┘
```

**标准 IO 攒够足够多的数据后，才一次系统调用进入内核**，大幅减少用户态/内核态切换次数。

**工作机制详解**：

- 写入时：`fwrite()` 先把数据写入用户空间的 IO 缓冲区，缓冲区满、遇到换行符（行缓冲模式）或手动调用 `fflush()` 时才触发底层 `write()` 系统调用
- 读取时：`fread()` 先用大块 `read()` 预读数据到 IO 缓冲区，后续小批量读取直接从缓冲区返回，不再反复系统调用
- 默认缓冲区大小：通常为 `BUFSIZ`（在 `<stdio.h>` 中定义），典型值为 4096 或 8192 字节

**标准IO的使用前提**：只能操作普通文件。`stdin`/`stdout`/`stderr` 在重定向到普通文件时也可以使用标准IO。

---

## 2. 标准 IO 核心函数

### 2.1 函数对照表

| 系统IO | 标准IO | 功能 |
|--------|--------|------|
| `open()` | `fopen()` | 打开文件 |
| `close()` | `fclose()` | 关闭文件 |
| `read()` | `fread()` | 读取文件 |
| `write()` | `fwrite()` | 写入文件 |
| `lseek()` | `fseek()` | 设置读写位置 |
| -- | `fflush()` | 刷新 IO 缓冲区 |
| -- | `ftell()` | 获取当前读写位置 |
| -- | `rewind()` | 复位到文件开头 |

**注意**：`FILE *` 是标准 IO 的文件句柄，等价于系统 IO 中的 `int fd`，但两者不能混用。`FILE *` 是库层面的封装结构体，`fd` 是内核层面的整数索引。

### 2.2 `fopen()` -- 打开文件

#### 函数原型

```c
#include <stdio.h>

FILE *fopen(const char *pathname, const char *mode);
```

- `pathname`：文件路径
- `mode`：打开模式（字符串）
- 返回值：成功返回 `FILE *` 指针，失败返回 `NULL`（此时应检查 `errno`）

#### mode 参数完整对比

| mode | 含义 | 文件不存在 | 文件已存在 | 读写位置 | 是否创建 |
|------|------|-----------|-----------|---------|---------|
| `"r"` | 只读 | 返回 NULL | 从头读 | 文件开头 | 否 |
| `"r+"` | 读写 | 返回 NULL | 可读可写 | 文件开头 | 否 |
| `"w"` | 只写 | 创建新文件 | **清空内容** | 文件开头 | 是 |
| `"w+"` | 读写 | 创建新文件 | **清空内容** | 文件开头 | 是 |
| `"a"` | 追加写 | 创建新文件 | 保留内容 | **文件末尾** | 是 |
| `"a+"` | 追加读写 | 创建新文件 | 保留内容 | 读开头/写末尾 | 是 |

**关键区别**：

- `"w"` vs `"a"`：`"w"` 会清空已有文件内容，`"a"` 保留原有内容且写入总在末尾
- `"r+"` vs `"w+"`：`"r+"` 要求文件必须存在，`"w+"` 会清空或创建
- `"a+"` 的特殊性：读操作从文件头开始，写操作总是追加到末尾（即使 `fseek` 改变了位置，写入仍追加到末尾）

#### 各模式示例

```c
#include <stdio.h>
#include <errno.h>
#include <string.h>

/* 模式 "r" -- 只读，文件必须存在 */
FILE *fp_r = fopen("existing.txt", "r");
if (fp_r == NULL) {
    fprintf(stderr, "fopen r failed: %s\n", strerror(errno));
    return -1;
}
/* ... 读取操作 ... */
fclose(fp_r);

/* 模式 "w" -- 只写，创建或清空 */
FILE *fp_w = fopen("output.txt", "w");
if (fp_w == NULL) {
    fprintf(stderr, "fopen w failed: %s\n", strerror(errno));
    return -1;
}
fprintf(fp_w, "Hello World\n");
fclose(fp_w);

/* 模式 "a" -- 追加写 */
FILE *fp_a = fopen("log.txt", "a");
if (fp_a == NULL) {
    perror("fopen a");
    return -1;
}
fprintf(fp_a, "[%s] Log entry\n", __TIME__);
fclose(fp_a);

/* 模式 "r+" -- 读写，文件必须存在 */
FILE *fp_rp = fopen("data.bin", "r+");
if (fp_rp) {
    /* 可读写已有文件 */
    fclose(fp_rp);
}

/* 模式 "w+" -- 读写，创建或清空 */
FILE *fp_wp = fopen("temp.txt", "w+");
if (fp_wp) {
    fprintf(fp_wp, "Write first\n");
    rewind(fp_wp);              /* 回到开头才能读 */
    char buf[128];
    fgets(buf, sizeof(buf), fp_wp);
    printf("Read back: %s", buf);
    fclose(fp_wp);
}

/* 模式 "a+" -- 追加读写 */
FILE *fp_ap = fopen("journal.log", "a+");
if (fp_ap) {
    /* 可以读取已有内容（从开头） */
    char line[256];
    while (fgets(line, sizeof(line), fp_ap))
        printf("Existing: %s", line);
    /* 写入总是追加到末尾 */
    fprintf(fp_ap, "Appended line\n");
    fclose(fp_ap);
}
```

#### `fopen` 与 `open` 的 mode 对应关系

| fopen mode | 等价 open flags | 说明 |
|-----------|----------------|------|
| `"r"` | `O_RDONLY` | 只读 |
| `"r+"` | `O_RDWR` | 读写 |
| `"w"` | `O_WRONLY \| O_CREAT \| O_TRUNC` | 只写，创建并清空 |
| `"w+"` | `O_RDWR \| O_CREAT \| O_TRUNC` | 读写，创建并清空 |
| `"a"` | `O_WRONLY \| O_CREAT \| O_APPEND` | 只写追加 |
| `"a+"` | `O_RDWR \| O_CREAT \| O_APPEND` | 读写追加 |

### 2.3 `fclose()` -- 关闭文件

```c
#include <stdio.h>

int fclose(FILE *stream);
```

- 返回值：成功返回 0，失败返回 `EOF`（-1）
- 关闭文件前会自动调用 `fflush()` 刷新缓冲区
- 进程退出时也会自动关闭所有打开的标准 IO 流，但显式调用 `fclose()` 是良好习惯

```c
FILE *fp = fopen("data.txt", "w");
if (fp == NULL) {
    perror("fopen");
    return 1;
}
/* ... 读写操作 ... */
if (fclose(fp) == EOF) {
    perror("fclose");
    return 1;
}
```

### 2.4 `fread()` -- 块读取

#### 函数原型与参数说明

```c
#include <stdio.h>

size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);
```

| 参数 | 类型 | 含义 |
|------|------|------|
| `ptr` | `void *` | 存放读取数据的缓冲区地址 |
| `size` | `size_t` | 每个元素的字节大小 |
| `nmemb` | `size_t` | 要读取的元素个数 |
| `stream` | `FILE *` | 文件流指针 |
| **返回值** | `size_t` | 实际成功读取的**元素个数**（不是字节数） |

**返回值理解**：`fread()` 返回的是成功读取的**元素个数**，即实际读到的完整 `size` 字节块的数量。如果返回值为 `nmemb`，说明全部读取成功；如果小于 `nmemb`，说明遇到了文件末尾或发生了错误。

#### 完整使用示例

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>

/* 读取结构体数组 */
typedef struct {
    int id;
    char name[32];
    double score;
} Student;

int read_students(const char *filename) {
    FILE *fp = fopen(filename, "r");
    if (fp == NULL) {
        perror("fopen");
        return -1;
    }

    Student students[100];
    size_t items_read;
    size_t total = 0;

    /* 循环读取，每次读一个结构体 */
    while ((items_read = fread(&students[total],
                                sizeof(Student), 1, fp)) == 1) {
        printf("Student %d: %s, score=%.1f\n",
               students[total].id,
               students[total].name,
               students[total].score);
        total++;
        if (total >= 100) break;
    }

    /* 区分是到达文件末尾还是发生错误 */
    if (feof(fp)) {
        printf("Reached end of file, total=%zu records\n", total);
    } else if (ferror(fp)) {
        fprintf(stderr, "Read error: %s\n", strerror(errno));
        fclose(fp);
        return -1;
    }

    fclose(fp);
    return 0;
}

/* 读取整个文件到内存 */
char *read_entire_file(const char *filename, size_t *out_size) {
    FILE *fp = fopen(filename, "r");
    if (fp == NULL) return NULL;

    /* 获取文件大小 */
    fseek(fp, 0, SEEK_END);
    long fsize = ftell(fp);
    rewind(fp);

    char *buf = malloc(fsize + 1);
    if (buf == NULL) {
        fclose(fp);
        return NULL;
    }

    size_t bytes_read = fread(buf, 1, fsize, fp);
    buf[bytes_read] = '\0';
    *out_size = bytes_read;

    fclose(fp);
    return buf;
}
```

### 2.5 `fwrite()` -- 块写入

#### 函数原型与参数说明

```c
#include <stdio.h>

size_t fwrite(const void *ptr, size_t size, size_t nmemb, FILE *stream);
```

参数含义与 `fread()` 完全相同，只是数据方向相反。

**返回值**：实际成功写入的**元素个数**。如果返回值不等于 `nmemb`，说明写入过程中出了问题（磁盘满、权限不足等）。

#### 完整使用示例

```c
#include <stdio.h>
#include <string.h>

typedef struct {
    int id;
    char name[32];
    double score;
} Student;

int save_students(const char *filename, Student *arr, size_t count) {
    FILE *fp = fopen(filename, "w");
    if (fp == NULL) {
        perror("fopen");
        return -1;
    }

    /* 一次写入整个数组 */
    size_t written = fwrite(arr, sizeof(Student), count, fp);
    if (written != count) {
        fprintf(stderr, "fwrite: only wrote %zu/%zu items\n",
                written, count);
        fclose(fp);
        return -1;
    }

    printf("Successfully wrote %zu records\n", written);
    fclose(fp);
    return 0;
}

/* 使用示例 */
int main(void) {
    Student class[] = {
        {1, "Alice",  92.5},
        {2, "Bob",    85.0},
        {3, "Charlie", 78.3},
    };
    size_t n = sizeof(class) / sizeof(class[0]);
    save_students("students.bin", class, n);
    return 0;
}
```

**注意**：`fwrite`/`fread` 操作的是二进制数据，不要用文本编辑器打开生成的 `.bin` 文件。结构体中有字节对齐问题，跨平台读取可能不兼容。

### 2.6 文件定位函数

#### `fseek()` -- 设置读写位置

```c
#include <stdio.h>

int fseek(FILE *stream, long offset, int whence);
```

| whence 常量 | 数值 | 含义 |
|------------|------|------|
| `SEEK_SET` | 0 | 从文件开头偏移 `offset` 字节 |
| `SEEK_CUR` | 1 | 从当前位置偏移 `offset` 字节 |
| `SEEK_END` | 2 | 从文件末尾偏移 `offset` 字节（`offset` 通常为负） |

返回值：成功返回 0，失败返回非 0 值。

#### `ftell()` -- 获取当前读写位置

```c
long ftell(FILE *stream);
```

返回值：当前读写位置距文件开头的字节偏移量。失败返回 `-1L`。

#### `rewind()` -- 回到文件开头

```c
void rewind(FILE *stream);
```

等价于 `(void)fseek(stream, 0L, SEEK_SET)`，但还会清除错误标志。

#### 定位函数完整示例

```c
#include <stdio.h>

int main(void) {
    FILE *fp = fopen("test.txt", "w+");
    if (fp == NULL) return 1;

    /* 写入数据 */
    fprintf(fp, "ABCDEFGHIJKLMNOPQRSTUVWXYZ");

    /* fseek 移动到第5个字节 */
    fseek(fp, 5, SEEK_SET);
    long pos = ftell(fp);
    printf("Position after fseek(5, SEEK_SET): %ld\n", pos);  /* 5 */

    /* 从当前位置跳过3个字节 */
    fseek(fp, 3, SEEK_CUR);
    pos = ftell(fp);
    printf("Position after fseek(3, SEEK_CUR): %ld\n", pos);  /* 8 */

    /* 从末尾往前10个字节 */
    fseek(fp, -10, SEEK_END);
    pos = ftell(fp);
    printf("Position after fseek(-10, SEEK_END): %ld\n", pos); /* 16 */

    /* rewind 回到开头 */
    rewind(fp);
    pos = ftell(fp);
    printf("Position after rewind(): %ld\n", pos);            /* 0 */

    fclose(fp);
    return 0;
}
```

**获取文件大小的常用技巧**：

```c
long get_file_size(const char *filename) {
    FILE *fp = fopen(filename, "r");
    if (fp == NULL) return -1;
    fseek(fp, 0, SEEK_END);   /* 移到文件末尾 */
    long size = ftell(fp);    /* 获取偏移量 = 文件大小 */
    fclose(fp);
    return size;
}
```

### 2.7 `fflush()` -- 缓冲区刷新详解

#### 基本概念

```
用户程序 → fwrite() → IO缓冲区 → fflush() → Page Cache → sync() → 磁盘
                                     ↑                    ↑
                              强制刷新IO缓冲区       强制刷新页缓存
                              到 Page Cache           到磁盘
```

| 函数 | 作用范围 | 说明 |
|------|---------|------|
| `fflush()` | 用户空间 IO 缓冲区 | 将 stdio 缓冲区数据刷到内核 Page Cache |
| `sync()` | 内核空间 Page Cache | 将 Page Cache 数据刷到磁盘 |

#### 函数原型

```c
#include <stdio.h>

int fflush(FILE *stream);
```

- `stream`：要刷新的文件流。如果传入 `NULL`，刷新**所有**打开的输出流。
- 返回值：成功返回 0，失败返回 `EOF`。

#### 调用时机与适用场景

| 场景 | 说明 | 示例 |
|------|------|------|
| 输出到终端 | 行缓冲模式下换行符自动触发刷新，但手动 `fflush(stdout)` 确保立即显示 | 进度提示 |
| 日志文件 | 每次写入后 `fflush()` 防止程序崩溃时丢日志 | `fflush(log_fp)` |
| 多进程写入同一文件 | 每个进程写完刷新，避免缓冲区数据交错 | 并发日志 |
| 管道/网络通信 | 向另一方发送数据后立即刷新 | `popen()` 场景 |
| 关闭文件**前** | `fclose()` 会自动调用，显式调用也可 | 确保数据落盘 |

#### 代码示例

```c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    /* 示例1：进度条需要立即刷新 */
    printf("Processing");
    for (int i = 0; i < 5; i++) {
        printf(".");
        fflush(stdout);          /* 不加这行，5个点会一起出现 */
        sleep(1);
    }
    printf(" Done!\n");

    /* 示例2：日志文件写入后立即刷新 */
    FILE *log = fopen("app.log", "a");
    if (log) {
        fprintf(log, "[INFO] Application started\n");
        fflush(log);             /* 确保日志立即落盘（到Page Cache） */
        /* ... 其他操作 ... */
        fclose(log);             /* fclose 也会自动 fflush */
    }

    /* 示例3：刷新所有输出流 */
    fflush(NULL);

    return 0;
}
```

**重要区分**：`fflush()` 只保证数据到达 Page Cache，不保证到达物理磁盘。要确保落盘，还需要 `fsync(fileno(fp))`（先通过 `fileno()` 获取底层 fd，再调用 `fsync()`）。

### 2.8 缓冲区模式与 `setvbuf()`

标准 IO 有三种缓冲模式，可以通过 `setvbuf()` 函数设置。

#### 三种缓冲模式

| 模式 | 宏常量 | 触发刷新的条件 | 典型使用场景 |
|------|--------|--------------|------------|
| **全缓冲** | `_IOFBF` | 缓冲区满才刷新 | 普通文件读写（默认） |
| **行缓冲** | `_IOLBF` | 遇到换行符 `\n` 或缓冲区满 | 终端输入输出（stdin/stdout） |
| **无缓冲** | `_IONBF` | 每次读写立即执行 | `stderr`（默认），实时性要求高 |

#### `setvbuf()` 函数

```c
#include <stdio.h>

int setvbuf(FILE *stream, char *buf, int mode, size_t size);
```

| 参数 | 含义 |
|------|------|
| `stream` | 要设置的文件流 |
| `buf` | 用户提供的缓冲区（`NULL` 则由库自动分配） |
| `mode` | `_IOFBF` / `_IOLBF` / `_IONBF` |
| `size` | 缓冲区大小（字节） |
| 返回值 | 成功返回 0，失败返回非 0 |

**注意**：`setvbuf()` 必须在打开文件后、进行任何读写操作**之前**调用，否则行为未定义。

#### `setbuf()` 简化版

```c
void setbuf(FILE *stream, char *buf);
```

等价于 `setvbuf(stream, buf, buf ? _IOFBF : _IONBF, BUFSIZ)`。

#### 完整示例

```c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    FILE *fp;
    char mybuf[1024];

    /* 示例1：设置全缓冲，缓冲区大小8192字节 */
    fp = fopen("fullbuf.txt", "w");
    setvbuf(fp, NULL, _IOFBF, 8192);
    for (int i = 0; i < 10000; i++) {
        fprintf(fp, "Line %d\n", i);
        /* 数据先进入缓冲区，满了才 write() */
    }
    fclose(fp);

    /* 示例2：设置行缓冲（即使是普通文件也按行刷新） */
    fp = fopen("linebuf.txt", "w");
    setvbuf(fp, NULL, _IOLBF, 1024);
    fprintf(fp, "This will be written immediately\n");  /* 换行触发刷新 */
    sleep(1);
    fprintf(fp, "This is still in buffer");             /* 没换行，暂不刷新 */
    sleep(1);
    fprintf(fp, "...now flushed\n");                    /* 换行触发刷新 */
    fclose(fp);

    /* 示例3：无缓冲模式（每字节立即写入） */
    fp = fopen("nobuf.txt", "w");
    setvbuf(fp, NULL, _IONBF, 0);
    fputc('A', fp);  /* 立即系统调用 */
    fputc('B', fp);  /* 立即系统调用 */
    fclose(fp);

    /* 示例4：使用自定义缓冲区 */
    fp = fopen("custombuf.txt", "w");
    setvbuf(fp, mybuf, _IOFBF, sizeof(mybuf));
    fprintf(fp, "Using custom buffer of 1024 bytes\n");
    fclose(fp);  /* mybuf 的生命周期必须覆盖整个文件操作期！ */

    return 0;
}
```

**三种模式的选择建议**：

- 普通文件读写：使用默认的**全缓冲**模式，性能最佳
- 终端交互：**行缓冲**，用户体验好（输入完一行立即处理）
- 错误输出（`stderr`）：**无缓冲**，确保错误信息立即显示
- 日志文件：可考虑**行缓冲**或频繁 `fflush()`，防止崩溃丢日志

---

## 3. 文件打开的辅模式补充

在第40讲中介绍了部分辅模式，这里补充剩下的：

### 3.1 `O_APPEND` -- 追加模式

```c
fd = open("file.txt", O_WRONLY | O_APPEND);
```

- 每次写入时，读写位置自动定位到文件末尾
- 不需要手动调用 `lseek()` 到末尾

**原子性保证**：`O_APPEND` 的写入是原子的。即使多个进程同时以 `O_APPEND` 方式打开同一文件写入，每条数据也不会互相覆盖。写入前内核自动将偏移量设为当前文件大小。

### 3.2 `O_DIRECT` -- 直接 IO 模式

```c
fd = open("file.txt", O_RDWR | O_DIRECT);
```

- 读写操作**不经过 Page Cache**，直接读写磁盘
- 适用场景：数据库等自己管理缓存的程序
- **限制**：要求读写缓冲区、偏移量、读写大小都必须满足底层硬件的扇区对齐要求（通常是 512 字节对齐）

### 3.3 `O_SYNC` -- 同步模式

```c
fd = open("file.txt", O_WRONLY | O_SYNC);
```

- 所有 `write()` 操作实时同步到磁盘
- 不需要手动调用 `sync()`
- 性能较低，但数据安全性高

**`O_SYNC` vs `O_DSYNC`**：

- `O_SYNC`：数据和元数据（文件大小、时间戳等）都同步写入
- `O_DSYNC`：只同步数据，元数据可以延迟写入，性能稍好

---

## 4. 文件 IO 的五大模式

| 模式 | 说明 | 学习阶段 |
|------|------|---------|
| **阻塞模式** | 默认模式，无法读写时进程挂起等待 | 本期 |
| **非阻塞模式** | `O_NONBLOCK`，无法读写时立即返回 | 高级IO |
| **IO多路复用** | `select/poll/epoll`，同时监控多个fd | 驱动开发 |
| **异步IO** | `aio_read/aio_write`，后台完成IO操作 | 驱动开发 |
| **信号驱动IO** | 文件可读写时发信号通知进程 | 驱动开发 |

**补充说明**：

- 阻塞模式下，如果读一个空管道或写一个满管道，进程会被挂起直到条件满足
- 非阻塞模式设置为 `fcntl(fd, F_SETFL, O_NONBLOCK)`，返回 `-1` 且 `errno=EAGAIN` 表示暂时无法操作
- 这五种模式是实现高性能网络服务器的核心基础

---

## 5. 系统IO 与 标准IO 代码对比

以下两个程序实现完全相同的功能：复制一个文件。通过对比可以看到两种 API 的使用差异。

### 5.1 系统 IO 版本

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

#define BUF_SIZE 4096

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "Usage: %s <src> <dst>\n", argv[0]);
        return 1;
    }

    int src_fd = open(argv[1], O_RDONLY);
    if (src_fd < 0) {
        perror("open src");
        return 1;
    }

    int dst_fd = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (dst_fd < 0) {
        perror("open dst");
        close(src_fd);
        return 1;
    }

    char buf[BUF_SIZE];
    ssize_t nread, nwrite;
    while ((nread = read(src_fd, buf, BUF_SIZE)) > 0) {
        nwrite = write(dst_fd, buf, nread);
        if (nwrite != nread) {
            perror("write");
            close(src_fd);
            close(dst_fd);
            return 1;
        }
    }

    if (nread < 0) {
        perror("read");
    }

    close(src_fd);
    close(dst_fd);
    return (nread < 0) ? 1 : 0;
}
```

### 5.2 标准 IO 版本

```c
#include <stdio.h>
#include <stdlib.h>

#define BUF_SIZE 4096

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "Usage: %s <src> <dst>\n", argv[0]);
        return 1;
    }

    FILE *src = fopen(argv[1], "r");
    if (src == NULL) {
        perror("fopen src");
        return 1;
    }

    FILE *dst = fopen(argv[2], "w");
    if (dst == NULL) {
        perror("fopen dst");
        fclose(src);
        return 1;
    }

    char buf[BUF_SIZE];
    size_t nread;
    while ((nread = fread(buf, 1, BUF_SIZE, src)) > 0) {
        size_t nwrite = fwrite(buf, 1, nread, dst);
        if (nwrite != nread) {
            perror("fwrite");
            fclose(src);
            fclose(dst);
            return 1;
        }
    }

    if (ferror(src)) {
        perror("fread");
    }

    fclose(src);
    fclose(dst);
    return 0;
}
```

### 5.3 关键差异总结

| 维度 | 系统 IO (`read`/`write`) | 标准 IO (`fread`/`fwrite`) |
|------|--------------------------|---------------------------|
| 文件句柄 | `int fd` | `FILE *stream` |
| 读取单位 | 字节数 | 元素个数 x 元素大小 |
| 缓冲区 | 仅内核 Page Cache | 用户空间 IO 缓冲区 + Page Cache |
| 错误检测 | 返回值 `-1` + `errno` | `ferror()` / `feof()` + `errno` |
| 打开模式 | `O_RDONLY` 等位标志 | `"r"` 等字符串模式 |
| 性能（小量频繁读写） | 差（每次系统调用） | 好（批量提交） |
| 控制粒度 | 精细（直接系统调用） | 粗粒度（库封装） |
| 适用场景 | 性能敏感、需要精确控制 | 通用文件操作 |

---

## 6. 性能测试与对比

### 6.1 不同缓冲区大小的写入性能测试

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>

#define FILE_SIZE (100 * 1024 * 1024)  /* 100MB */
#define DATA_CHUNK 1024                 /* 每次写 1KB */

void test_write(const char *filename, size_t buf_size, int use_stdio) {
    char data[DATA_CHUNK];
    memset(data, 'X', DATA_CHUNK);

    clock_t start = clock();

    if (use_stdio) {
        FILE *fp = fopen(filename, "w");
        if (fp == NULL) { perror("fopen"); return; }

        /* 设置自定义缓冲区大小 */
        char *iobuf = malloc(buf_size);
        setvbuf(fp, iobuf, _IOFBF, buf_size);

        size_t total = 0;
        while (total < FILE_SIZE) {
            fwrite(data, 1, DATA_CHUNK, fp);
            total += DATA_CHUNK;
        }
        fclose(fp);
        free(iobuf);  /* 注意：必须先 fclose 再 free */
    } else {
        int fd = open(filename, O_WRONLY | O_CREAT | O_TRUNC, 0644);
        if (fd < 0) { perror("open"); return; }

        size_t total = 0;
        while (total < FILE_SIZE) {
            write(fd, data, DATA_CHUNK);
            total += DATA_CHUNK;
        }
        close(fd);
    }

    clock_t end = clock();
    double elapsed = (double)(end - start) / CLOCKS_PER_SEC;
    printf("  Buffer=%5zu bytes, %s, Time=%.3f sec\n",
           buf_size, use_stdio ? "stdio" : "sys_io", elapsed);
}

int main(void) {
    printf("=== Write 100MB Performance Test ===\n\n");

    /* 系统IO：无用户空间缓冲，每次1KB都要系统调用 */
    test_write("sys_test.bin", 0, 0);

    /* 标准IO：不同缓冲区大小 */
    size_t buf_sizes[] = {512, 1024, 4096, 8192, 16384, 65536};
    for (int i = 0; i < 6; i++) {
        test_write("stdio_test.bin", buf_sizes[i], 1);
    }

    printf("\nConclusion: Larger stdio buffer = fewer syscalls = better performance.\n");
    return 0;
}
```

编译运行：

```bash
gcc -o perf_test perf_test.c
./perf_test
```

**预期结果**：标准 IO 的缓冲区越大，性能越接近系统 IO 的大块写入。但缓冲区大到一定程度后，收益递减。

### 6.2 系统调用次数对比

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>

/* 这个程序演示：逐字节写入时，标准IO和系统IO的系统调用次数差异 */
int main(void) {
    /* 方式1：系统IO逐字节写入 -- 1024次系统调用 */
    {
        int fd = open("sys_byte.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
        for (int i = 0; i < 1024; i++) {
            write(fd, "X", 1);  /* 每次1次系统调用 */
        }
        close(fd);
    }

    /* 方式2：标准IO逐字节写入 -- 可能只有1次系统调用 */
    {
        FILE *fp = fopen("stdio_byte.txt", "w");
        for (int i = 0; i < 1024; i++) {
            fputc('X', fp);  /* 先进入缓冲区 */
        }
        fclose(fp);  /* 关闭时一次性 write() */
    }

    printf("Compare: sys_byte.txt vs stdio_byte.txt\n");
    printf("sys_byte: ~1024 syscalls\n");
    printf("stdio_byte: 1 syscall (buffered)\n");
    return 0;
}
```

这个例子直观展示了标准 IO 的核心优势：攒够数据，一次提交。

---

## 7. 错误处理：`feof()` / `ferror()` / `clearerr()`

标准 IO 不通过返回值 `-1` 来报告错误，而是用专门的函数检测流状态。

### 函数说明

```c
#include <stdio.h>

int feof(FILE *stream);     /* 是否到达文件末尾 */
int ferror(FILE *stream);   /* 是否发生读写错误 */
void clearerr(FILE *stream); /* 清除错误标志和EOF标志 */
```

- `feof()`：返回非 0 表示已到达文件末尾
- `ferror()`：返回非 0 表示发生了 I/O 错误
- `clearerr()`：重置错误和 EOF 标志，使后续操作可以继续

### 完整使用模式

```c
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>

int read_all_lines(const char *filename) {
    FILE *fp = fopen(filename, "r");
    if (fp == NULL) {
        fprintf(stderr, "fopen '%s': %s\n", filename, strerror(errno));
        return -1;
    }

    char line[256];
    int line_num = 0;

    while (1) {
        if (fgets(line, sizeof(line), fp) == NULL) {
            /* fgets 返回 NULL 有两种可能 */
            if (feof(fp)) {
                /* 正常到达文件末尾 */
                printf("End of file reached after %d lines.\n", line_num);
                break;
            } else if (ferror(fp)) {
                /* 发生了错误 */
                fprintf(stderr, "Read error on line %d: %s\n",
                        line_num, strerror(errno));
                fclose(fp);
                return -1;
            }
            /* 理论上不会到这里 */
            break;
        }
        line_num++;
        printf("Line %d: %s", line_num, line);
    }

    fclose(fp);
    return 0;
}

/* clearerr 使用场景：从错误中恢复 */
int recoverable_read(FILE *fp, char *buf, size_t size) {
    size_t n = fread(buf, 1, size, fp);
    if (n < size && ferror(fp)) {
        fprintf(stderr, "Read error, attempting recovery...\n");
        clearerr(fp);  /* 清除错误标志 */
        /* 可以尝试继续读取 */
        return 0;
    }
    return (int)n;
}
```

---

## 8. 实用函数补充

### 8.1 `tmpfile()` -- 创建临时文件

```c
#include <stdio.h>

FILE *tmpfile(void);
```

- 创建一个临时文件，以 `"w+"` 模式打开
- 文件在关闭时或程序结束时**自动删除**
- 适用于需要临时存储数据又不想手动管理的场景

```c
#include <stdio.h>

int main(void) {
    FILE *tmp = tmpfile();
    if (tmp == NULL) {
        perror("tmpfile");
        return 1;
    }

    fprintf(tmp, "Temporary data here\n");
    rewind(tmp);

    char buf[128];
    fgets(buf, sizeof(buf), tmp);
    printf("Read from tmp: %s", buf);

    /* 无需手动删除，fclose 或程序退出时自动清理 */
    fclose(tmp);
    return 0;
}
```

### 8.2 `freopen()` -- 重定向文件流

```c
FILE *freopen(const char *pathname, const char *mode, FILE *stream);
```

将 `stdout` 重定向到文件：

```c
#include <stdio.h>

int main(void) {
    /* 将 stdout 重定向到文件 */
    if (freopen("output.txt", "w", stdout) == NULL) {
        perror("freopen");
        return 1;
    }

    /* 之后的 printf 都会写入 output.txt */
    printf("This goes to the file, not the terminal.\n");
    printf("So does this line.\n");

    fclose(stdout);
    return 0;
}
```

### 8.3 `fgetpos()` / `fsetpos()` -- 大文件定位

对于超过 2GB 的大文件，`fseek`/`ftell` 的 `long` 类型可能不够用（32位系统上为 4 字节）。此时应使用 `fpos_t` 类型：

```c
#include <stdio.h>

int fgetpos(FILE *stream, fpos_t *pos);
int fsetpos(FILE *stream, const fpos_t *pos);
```

### 8.4 `fileno()` -- 获取底层文件描述符

```c
#include <stdio.h>

int fileno(FILE *stream);
```

从 `FILE *` 获取底层的 `int fd`，以便在需要时混用标准 IO 和系统 IO：

```c
FILE *fp = fopen("data.txt", "w");
int fd = fileno(fp);
/* 使用 fd 调用 fstat、fcntl、fsync 等系统调用 */
fsync(fd);
fclose(fp);
```

**注意**：混用标准 IO 和系统 IO 容易导致缓冲区不一致，谨慎使用。

---

## 9. 线程安全说明

### 标准 IO 的线程安全性

标准 IO 函数（`fread`/`fwrite`/`fprintf` 等）对同一个 `FILE *` 的操作是**线程安全**的。C 标准库内部使用锁（`flockfile`/`funlockfile`）来保护每个 `FILE *` 对象。

但这意味着多线程并发读写同一个 `FILE *` 有锁竞争开销。更好的做法是每个线程使用独立的 `FILE *`。

```c
#include <stdio.h>
#include <pthread.h>

/* 线程不安全：两个线程写同一个文件（输可能交错） */
void *write_thread(void *arg) {
    FILE *fp = (FILE *)arg;
    /* 锁保护单次调用，但不保护多次调用的原子性 */
    fprintf(fp, "Thread %ld: Line 1\n", (long)pthread_self());
    fprintf(fp, "Thread %ld: Line 2\n", (long)pthread_self());
    /* 两行之间可能被另一个线程插入 */
    return NULL;
}

/* 解决方案1：使用 flockfile/ftrylockfile/funlockfile */
void *write_thread_safe(void *arg) {
    FILE *fp = (FILE *)arg;
    flockfile(fp);  /* 手动加锁 */
    fprintf(fp, "Thread %ld: Line 1\n", (long)pthread_self());
    fprintf(fp, "Thread %ld: Line 2\n", (long)pthread_self());
    funlockfile(fp); /* 手动解锁 */
    return NULL;
}

/* 解决方案2：每个线程打开独立的 FILE *（推荐） */
```

### 相关函数

```c
#include <stdio.h>

void flockfile(FILE *stream);     /* 加锁 */
int  ftrylockfile(FILE *stream);  /* 尝试加锁，不阻塞 */
void funlockfile(FILE *stream);   /* 解锁 */
```

---

## 本讲总结

| 对比项 | 系统IO | 标准IO |
|--------|--------|--------|
| 函数 | `open/read/write` | `fopen/fread/fwrite` |
| 缓冲区 | 内核 Page Cache | 用户空间 IO 缓冲区 + Page Cache |
| 性能 | 频繁小数据读写开销大 | 攒够数据批量提交，性能高 |
| 使用场景 | 需要精细控制时 | 通用场景 |

**标准IO的缓冲区层级**：
```
fwrite() → IO缓冲区 → fflush() → Page Cache → sync() → 磁盘
```

**三种缓冲模式**：
```
全缓冲（_IOFBF）：缓冲区满才刷新 → 普通文件默认
行缓冲（_IOLBF）：遇到 \n 或缓冲区满 → 终端默认
无缓冲（_IONBF）：立即刷新 → stderr 默认
```

**核心函数速查**：

| 函数 | 关键点 |
|------|--------|
| `fopen(path, mode)` | 6 种 mode，注意 `w` 清空、`a` 追加 |
| `fclose(fp)` | 自动刷新缓冲区再关闭 |
| `fread(ptr, sz, n, fp)` | 返回元素个数，不是字节数 |
| `fwrite(ptr, sz, n, fp)` | 检查返回值 != n |
| `fseek(fp, off, whence)` | SEEK_SET/CUR/END |
| `ftell(fp)` | 获取当前位置 |
| `rewind(fp)` | 回到开头 + 清除错误 |
| `fflush(fp)` | 刷到 Page Cache，非磁盘 |
| `setvbuf(fp,buf, mode, sz)` | 必须在读写之前调用 |
| `feof(fp)` / `ferror(fp)` | 检测流状态 |

---

## 附录：练习建议

1. 编写一个程序，使用 `fopen("r+")` 打开一个已有文件，在文件中间插入一行（需要移动后续数据）
2. 对比 `fprintf` + `%s` 和 `fwrite` 写入大量二进制数据的性能差异
3. 实验 `setvbuf` 不同模式对逐字节 `fputc` 写入性能的影响
4. 实现一个简易的 `tail -f` 程序：打开文件，`fseek` 到末尾，循环等待新数据并输出

---

*笔记生成日期：2026-07-22 | 转写：Whisper tiny*

*优化日期：2026-07-24 | 新增内容：fopen模式详解、fread/fwrite参数说明、文件定位函数、缓冲区模式、系统IO对比代码、性能测试、错误处理、实用函数、线程安全*
