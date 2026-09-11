---
title: "第39讲：一切皆文件"
date: 2026-09-11T09:05:00+08:00
draft: false
description: "Linux 屏蔽了不同硬件设备的差异，将所有设备都抽象为文件，提供统一的接口给用户使用。"
series: ["Linux 入门"]
series_order: 22
categories: ["技术笔记"]
tags: ["Linux", "文件IO"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P40)
> **标题**：第39讲 — 一切皆文件（Linux哲学 + 虚拟文件系统VFS）
> **时长**：约13分钟
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. Linux哲学：一切皆文件

### 1.1 核心思想

Linux 屏蔽了不同硬件设备的差异，将**所有设备都抽象为文件**，提供统一的接口给用户使用。

```c
// 无论是普通文件、设备、还是进程信息
// 用户都可以用同样的 open/read/write 接口操作
fd = open("/dev/ttyUSB0", O_RDWR);  // 串口设备
write(fd, "hello", 5);
read(fd, buf, 10);
close(fd);
```

#### 背景解释：为什么需要"一切皆文件"？

在传统的操作系统中，操作不同硬件需要学习不同的 API。比如：
- 操作串口需要一套专门的串口 API
- 读取磁盘需要另一套磁盘 IO API
- 发送网络数据又需要 socket API（参数和用法各不相同）

程序员需要记忆大量不同的函数名、参数顺序和错误处理方式。这极大地增加了开发成本和出错概率。

Linux 的设计者（尤其是 Linus Torvalds 和 Unix 的前辈们）提出了一句口号：

> **Everything is a file.**

翻译过来就是：不管你操作的是硬盘、键盘、鼠标、串口、进程、内存，还是网络连接，统统用同一套接口 `open() / read() / write() / close()` 来操作。

这样，Linux 程序员只需要精通一套 API，就能写任何程序。

### 1.2 设计目标

| 目标 | 说明 |
|------|------|
| 屏蔽差异 | 不同硬件操作方式不同，但文件接口统一 |
| 统一接口 | 用户只需学会 open/read/write 即可操作任何设备 |
| 易于扩展 | 新设备只需实现标准接口，即可被系统识别 |

### 1.3 实操：初探"一切皆文件"

在 Linux 终端中，下面的命令会让你直观感受"一切皆文件"的哲学：

```bash
# 1. 查看普通文件（常规操作）
ls -l /etc/passwd
cat /etc/passwd

# 2. 查看设备文件 —— 磁盘也是一个"文件"
ls -l /dev/sda
# 输出类似: brw-rw---- 1 root disk 8, 0 Jul 21 10:00 /dev/sda

# 3. 终端本身也是一个"文件" —— 当前你正在使用的终端
tty
# 输出类似: /dev/pts/0

# 4. 往终端"文件"里写数据，等同于在屏幕上打印
echo "Hello from file write!" > /dev/pts/0

# 5. 零设备 —— 一个永远返回0的特殊文件
dd if=/dev/zero of=test.bin bs=1024 count=10
# 生成了一个全是0的10KB文件

# 6. 空设备 —— 一个吞噬一切数据的"黑洞"
echo "这条数据被丢弃了" > /dev/null

# 7. 随机数设备 —— 读取它得到随机字节
head -c 16 /dev/urandom | od -A x -t x1z
```

> 上面的 7 个例子说明：不管是硬盘分区(`sda`)、终端(`pts/0`)、还是数据黑洞(`null`)，在 Linux 里它们都是"文件"，都可以用 `ls`、`cat`、`echo` 等命令操作。

---

## 2. 虚拟文件系统（VFS）

### 2.1 背景

Linux 支持多种文件系统，包括：

| 类型 | 文件系统 | 挂载点 |
|------|---------|--------|
| 普通文件系统 | ext4, FAT32, yaffs | `/mnt` 等 |
| 进程文件系统 | **procfs** | `/proc` |
| 设备文件系统 | **sysfs** | `/sys` |
| 设备管理 | **devfs** | `/dev` |

- **procfs**：存放进程相关信息，类似内核的进程管理器
- **sysfs**：存放设备与硬件操作相关的结构信息
- **devfs**：设备节点

这些文件系统的存储格式不同、功能差异大，但都可以通过 `mount` 命令挂载到某个目录下，然后用 `ls` 查看。

### 2.2 问题：如何做到接口统一？

**答案**：VFS 在底层对各种文件系统的格式之上做了**抽象层**。

```
用户应用
    ↓
C库函数 (fopen/fread/fwrite)
    ↓
系统调用 (open/read/write)
    ↓
┌──────────────────────────────────┐
│      VFS 抽象层                   │
│  (屏蔽底层差异，提供统一接口)       │
└──────────────────────────────────┘
    ↓
ext4  FAT32  yaffs  procfs  sysfs
```

这样，无论对普通文件的访问，还是对设备文件的访问，都变成了对 VFS 抽象层的访问。

#### 类比说明：VFS 就像统一的快递接口

想象你有多个快递公司：顺丰、圆通、中通、邮政。每家公司的取件方式、面单格式、追踪系统都不一样。

如果不做抽象，你（用户程序）需要：
- 给顺丰打电话用顺丰的 APP
- 给圆通打电话用圆通的 APP
- 给邮政打电话用邮政的 APP
- ...

但是如果你有一个**统一的快递平台**（比如菜鸟驿站），你把包裹交给它，它会根据目的地自动选择最合适的快递公司。你只需要记住一个取件地址、一个寄件流程。

**VFS 就是这个"统一的快递平台"**：
- `ext4`、`FAT32`、`procfs`、`sysfs` 就是不同的"快递公司"
- `open/read/write` 就是统一的"寄件/取件接口"
- 你不需要知道底层用的是 ext4 还是 procfs，只管调用 `open/read/write` 就行

### 2.3 实操：查看 /proc、/sys、/dev

下面是直接观察 Linux 三个特殊文件系统的命令演示：

```bash
# ===== /proc —— 进程和内核信息的"文件"视图 =====

# 查看 CPU 信息 —— CPU 的型号、频率、核心数都在这
cat /proc/cpuinfo | head -20

# 查看内存信息 —— 总内存、可用内存、缓冲区等
cat /proc/meminfo | head -10

# 查看内核版本
cat /proc/version

# 查看系统运行时间（开机了多久）
cat /proc/uptime

# 查看当前进程列表 —— 每个数字目录就是一个 PID
ls /proc | head -20

# 查看某个进程的命令行（假设 PID=1，即 systemd/init 进程）
cat /proc/1/cmdline
echo ""
cat /proc/1/status | head -10

# 查看当前 Shell 自己的进程信息
cat /proc/$$/status | head -10
# $$ 是当前 Shell 的 PID


# ===== /sys —— 硬件设备和内核模块的"文件"视图 =====

# 查看系统中有哪些块设备（磁盘、U盘等）
ls /sys/block/

# 查看第一个硬盘的大小（单位：扇区，每扇区通常512字节）
cat /sys/block/sda/size

# 查看 CPU 的拓扑结构
ls /sys/devices/system/cpu/

# 查看某个 CPU 核心是否在线
cat /sys/devices/system/cpu/cpu0/online


# ===== /dev —— 设备节点的"文件"视图 =====

# 查看所有设备文件
ls -l /dev/ | head -30

# 注意第一列的第一个字符：
#   b = 块设备(block device)，如硬盘 sda
#   c = 字符设备(character device)，如终端 tty、串口 ttyUSB0

# 查看磁盘分区
ls -l /dev/sd*

# 查看终端设备
ls -l /dev/tty*

# 用 file 命令查看设备文件的类型
file /dev/sda
file /dev/tty
file /dev/null
file /dev/zero
file /dev/urandom
```

> 看得越多，越能体会到：在 Linux 里，**真的什么都是文件**。CPU 信息是文件，内存信息是文件，进程是文件，硬盘是文件，连 `/dev/null` 这个黑洞也是个文件。

---

## 3. VFS 的核心抽象对象

VFS 包含了几个核心抽象对象，封装了底层的读写操作：

#### 背景：为什么需要这些对象？

当用户程序调用 `open("/home/user/hello.txt", O_RDONLY)` 时，VFS 在背后做了大量工作：

1. 需要知道 `/home/user/hello.txt` 在哪个文件系统上（ext4？FAT32？）
2. 需要查找这个路径对应的 inode 是谁
3. 需要把文件名 → inode 的映射关系建好
4. 需要创建一个"打开的文件"对象，记录当前读到哪了（偏移量）

这四个步骤对应了 VFS 的四个核心对象：**super_block**、**inode**、**dentry**、**file**。

### 3.1 `super_block`（超级块）

| 说明 | 细节 |
|------|------|
| 作用 | 表示某一个具体的文件系统实例 |
| 时机 | 每当系统挂载一个新文件系统时，就创建一个 super_block 对象来管理它 |

#### 类比：super_block 就是"土地证"

一个 super_block 就像一块土地的**土地证**。系统里挂载了几个文件系统，就有几个 super_block。

每块地（文件系统）有自己独特的属性：位置（挂载点）、大小、类型等。土地证就是用来管理这些基本属性的。

#### 实操：查看当前挂载的文件系统

```bash
# 查看当前有哪些文件系统被挂载了
mount | head -20

# 更详细的信息（包含文件系统类型）
df -T

# 输出示例：
# Filesystem     Type     1K-blocks    Used Available Use% Mounted on
# /dev/sda1      ext4      51474912 8234108  40585400  17% /
# tmpfs          tmpfs      1638060       0   1638060   0% /dev/shm

# 查看 /proc/mounts —— 内核视角的挂载信息
cat /proc/mounts | head -20
```

### 3.2 `inode`（索引节点）

| 说明 | 细节 |
|------|------|
| 作用 | 表示某一个具体的文件或目录（存元数据） |
| 内容 | 文件的类型、权限、大小、时间戳等 |

#### 类比：inode 就是"身份证号"

每个文件/目录在创建时，系统都会分配一个唯一的 inode 编号。

- 身份证号是唯一的，inode 编号也是唯一的（在同一文件系统内）
- 身份证记录了你的姓名、性别、出生日期（元数据），inode 记录了文件的权限、大小、时间戳
- 身份证号不能改，inode 编号也不能改
- 一个人可以有多个名字（别名/昵称），一个 inode 也可以有多个**硬链接**（多个文件名指向同一个 inode）

#### 实操：查看文件的 inode 信息

```bash
# 创建一个测试文件
echo "hello inode" > test_inode.txt

# 查看文件的 inode 编号（-i 参数）
ls -i test_inode.txt
# 输出示例: 1234567 test_inode.txt

# 更详细地查看 inode 信息（stat 命令）
stat test_inode.txt

# stat 输出示例：
#   File: test_inode.txt
#   Size: 12              Blocks: 8          IO Block: 4096   regular file
# Device: 801h/2049d      Inode: 1234567     Links: 1
# Access: (0644/-rw-r--r--)  Uid: (1000/ user)   Gid: (1000/ user)
# Access: 2026-07-22 10:00:00.000000000 +0800
# Modify: 2026-07-22 10:00:00.000000000 +0800
# Change: 2026-07-22 10:00:00.000000000 +0800

# 创建硬链接 —— 两个文件名指向同一个 inode
ln test_inode.txt test_inode_link.txt
ls -i test_inode.txt test_inode_link.txt
# 两个文件的 inode 编号完全一样！说明它们本质是同一个文件

# 删除原文件，硬链接仍然可以访问
rm test_inode.txt
cat test_inode_link.txt
# 输出: hello inode

# 清理
rm test_inode_link.txt
```

> 注意观察 `stat` 输出中的三个时间：
> - **Access (atime)**：最后一次读取文件的时间
> - **Modify (mtime)**：最后一次修改文件内容的时间
> - **Change (ctime)**：最后一次修改文件元数据（权限、所有者等）的时间

### 3.3 `dentry`（目录项）

| 说明 | 细节 |
|------|------|
| 作用 | 记录文件的路径信息，实现文件名到 inode 的映射 |
| 关联 | 每个路径分量对应一个 dentry |

#### 类比：dentry 就是"通讯录"

想象你的手机通讯录：
- 通讯录的每条记录 = 一个 dentry
- 记录中的"姓名" = 文件名
- 记录中的"电话号码" = inode 编号
- 找人时先查通讯录（dentry），再打电话（访问 inode）

路径 `/home/user/hello.txt` 在 VFS 中会被拆成三个 dentry：
- `home` → 指向 `/home` 目录的 inode
- `user` → 指向 `/home/user` 目录的 inode
- `hello.txt` → 指向最终文件的 inode

VFS 会缓存这些 dentry（dentry cache，简称 dcache），这样下次查找同一个路径时就不用重新解析了，大大加快了文件访问速度。

### 3.4 `file`（打开的文件）

| 说明 | 细节 |
|------|------|
| 作用 | 表示一个已经被进程打开的文件 |
| 特点 | 包含当前读写偏移量，多个进程可同时打开同一文件 |

#### 类比：file 就是"打开的书"

一本书（inode）可以被多个人（进程）同时阅读：每个人有自己的书签（偏移量），记录自己读到哪一页了。

**file 对象就负责管理这个"书签"（偏移量）**：
- 进程 A 读到第 100 字节，A 的 file 对象记录偏移量 = 100
- 进程 B 读到第 50 字节，B 的 file 对象记录偏移量 = 50
- 它们读的是同一个文件（同一个 inode），但互不干扰

#### 实操：查看进程打开了哪些文件

```bash
# 查看当前 Shell 打开的文件描述符
ls -l /proc/$$/fd/

# 输出类似：
# lrwx------ 1 user user 64 Jul 22 10:00 0 -> /dev/pts/0  (标准输入)
# lrwx------ 1 user user 64 Jul 22 10:00 1 -> /dev/pts/0  (标准输出)
# lrwx------ 1 user user 64 Jul 22 10:00 2 -> /dev/pts/0  (标准错误)

# 查看某个进程（如PID=1的systemd）打开了哪些文件
sudo ls -l /proc/1/fd/ 2>/dev/null | head -20

# 用 lsof 查看某个进程打开的所有文件
lsof -p $$ 2>/dev/null | head -20
```

---

## 4. VFS 的操作接口

每个 inode 包含两大类操作接口集合：

#### 背景：接口即"契约"

VFS 定义了两套"契约"（结构体），具体文件系统必须"签署"这份契约：填好自己的实现函数。

这就像定义了两张"表格"，每张表格有一堆"填空项"。每个文件系统必须填满这些空：
- 普通文件系统的 `read` 填空：从磁盘扇区读数据
- procfs 的 `read` 填空：从内核数据结构动态生成数据
- sysfs 的 `read` 填空：从设备驱动获取属性值

虽然内部实现千差万别，但对外暴露的函数签名完全一样。

### 4.1 `inode_operations` — 目录管理

```c
struct inode_operations {
    int (*create)(struct inode *dir, struct dentry *dentry, ...);  // 创建文件
    int (*mkdir)(struct inode *dir, struct dentry *dentry, ...);   // 创建目录
    int (*rename)(struct inode *old_dir, struct dentry *old_dentry, ...);  // 重命名
    int (*link)(struct dentry *old_dentry, struct inode *dir, ...);  // 硬链接
    int (*unlink)(struct inode *dir, struct dentry *dentry);       // 删除文件
    // ...
};
```

#### 实操：观察 inode_operations 在用户态的体现

```bash
# 创建文件 → 内核调用 inode_operations->create
touch new_file.txt

# 创建目录 → 内核调用 inode_operations->mkdir
mkdir test_dir

# 创建硬链接 → 内核调用 inode_operations->link
ln new_file.txt hardlink_to_file.txt
ls -i new_file.txt hardlink_to_file.txt

# 重命名 → 内核调用 inode_operations->rename
mv new_file.txt renamed_file.txt

# 删除文件 → 内核调用 inode_operations->unlink
rm renamed_file.txt hardlink_to_file.txt

# 删除目录
rmdir test_dir
```

### 4.2 `file_operations` — 文件操作

```c
struct file_operations {
    int (*open)(struct inode *inode, struct file *file);     // 打开文件
    ssize_t (*read)(struct file *file, char *buf, ...);      // 读文件
    ssize_t (*write)(struct file *file, const char *buf, ...); // 写文件
    int (*release)(struct inode *inode, struct file *file);  // 关闭文件
    // ...
};
```

#### 实操：用 strace 追踪 file_operations 的系统调用

`strace` 可以追踪用户程序发出的系统调用，让我们"看见" VFS 接口在背后是如何被调用的：

```bash
# 创建一个简单的测试文件
echo "hello VFS" > test_vfs.txt

# 用 strace 追踪 cat 命令的系统调用
strace cat test_vfs.txt 2>&1 | grep -E "open|read|write|close"

# 典型输出（简化）：
# openat(AT_FDCWD, "test_vfs.txt", O_RDONLY) = 3
# read(3, "hello VFS\n", 131072)           = 10
# write(1, "hello VFS\n", 10)              = 10
# close(3)                                 = 0

# 可以看到：cat 命令最终调用了 open → read → write → close
# 这就是 file_operations 在用户态的"影子"

# 清理
rm test_vfs.txt
```

> `strace` 是学习 Linux 系统调用的神器。遇到不明白的命令，用 `strace` 追踪一下，就能看到它到底调用了哪些系统调用。

---

## 5. 各文件系统的接入方式

具体文件系统需要**填充** VFS 定义的操作接口：

```
VFS 定义的操作接口（框架）
    ↑         ↑         ↑
    |         |         |
ext4填充   FAT32填充   sysfs填充
 自己的操作   自己的操作    自己的操作
```

例如：
- **ext4**：实现 ext4_read, ext4_write, ext4_open ...
- **FAT32**：实现 fat_read, fat_write, fat_open ...
- **sysfs**：实现 sysfs_read, sysfs_write ...
- **procfs**：实现 proc_read, proc_write ...

所有具体文件系统填好自己的操作函数后，上层应用就可以通过**统一的接口**调用了。

#### 深入理解：不同文件系统的 fill 方式不同

```bash
# ext4 的 read：从磁盘扇区读取数据
# procfs 的 read：从内核内存动态生成数据（根本没有磁盘！）

# 演示：/proc/cpuinfo 不是一个真实文件
ls -l /proc/cpuinfo
# 会显示大小为 0，因为数据是实时生成的

# 但你可以正常"读取"它
cat /proc/cpuinfo | head -5
# 有内容！因为 proc_read 在内核中动态构造了这些数据

# 对比普通文件
ls -l /etc/hostname
cat /etc/hostname

# 两者都用了同样的命令（ls、cat），但底层走的是完全不同的路径
```

想象一个"文件"就像一个函数：你调用它（read），它给你返回值。不同的是：
- ext4 的 read：从磁盘扇区读出真实存储的数据
- procfs 的 read：当场"计算"出数据并返回
- sysfs 的 read：从内核驱动询问硬件状态然后返回

这就解释了为什么 `/proc/cpuinfo` 没有大小却能"读"出内容。它根本不是文件，它是一个**活的函数调用**。

---

## 6. 完整的调用链路

```
应用层（用户程序）
    ↓ 调用 C 库函数 (fopen/fread/fwrite)
C库函数（glibc）
    ↓ 调用 系统调用
系统调用层 (sys_open/sys_read/sys_write)
    ↓ 调用 VFS 抽象接口
VFS 抽象层
    ↓ 调用 具体文件系统的填充函数
具体文件系统 (ext4/FAT32/procfs/sysfs)
    ↓
硬件设备
```

> 用户在应用层只需要用 `open/read/write`，经过 C 库 → 系统调用 → VFS → 具体文件系统，最终操作到硬件设备。

#### 实操：从用户态到底层的完整追踪

下面用 `strace` 完整追踪一次文件读取，展示完整的调用链路：

```bash
# 准备测试文件
echo "Linux VFS demo" > /tmp/vfs_demo.txt

# 完整追踪 cat 命令（输出到文件以便查看）
strace -o /tmp/strace_output.txt cat /tmp/vfs_demo.txt

# 查看追踪结果中的关键系统调用
grep -E "openat|read|write|close" /tmp/strace_output.txt

# 更详细的追踪（显示每个系统调用的时间消耗）
strace -T cat /tmp/vfs_demo.txt 2>&1 | grep -E "openat|read|write|close"

# 典型输出：
# openat(AT_FDCWD, "/tmp/vfs_demo.txt", O_RDONLY) = 3 <0.000021>
# read(3, "Linux VFS demo\n", 131072) = 15 <0.000015>
# write(1, "Linux VFS demo\n", 15) = 15 <0.000017>
# close(3)                                = 0 <0.000007>

# 这个调用过程就是：
# 用户调用 cat → glibc 翻译为 openat() → 系统调用 sys_openat()
# → VFS → ext4 具体文件系统的 ext4_file_open() → 硬盘驱动 → 硬盘

# 清理
rm /tmp/vfs_demo.txt /tmp/strace_output.txt
```

---

## 7. 深入实操：一切皆文件的经典演示

本节通过一系列实际命令演示，让你亲身体会 Linux "一切皆文件" 的神奇之处。

### 7.1 文件描述符：万物归一

每个进程一启动，内核就自动给它打开了三个"标准文件"：

| 文件描述符 | 名称 | 默认指向 | 用途 |
|-----------|------|---------|------|
| 0 | stdin（标准输入） | `/dev/pts/0`（终端键盘） | 接收用户输入 |
| 1 | stdout（标准输出） | `/dev/pts/0`（终端屏幕） | 输出结果 |
| 2 | stderr（标准错误） | `/dev/pts/0`（终端屏幕） | 输出错误信息 |

```bash
# 查看当前进程的文件描述符
ls -l /proc/$$/fd/

# 文件描述符 0/1/2 都是指向同一个终端设备文件
# 这说明：键盘是文件，屏幕也是文件

# 证明：直接往文件描述符写数据
echo "写入 stdout" >&1     # 等价于 echo 写入显示器
echo "写入 stderr" >&2     # 等价于错误输出

# 把标准错误重定向到文件
ls /nonexistent_dir 2>/tmp/error.log
cat /tmp/error.log
rm /tmp/error.log
```

### 7.2 设备即文件：/dev 深度探索

```bash
# 零设备 —— 产生无限的零字节
dd if=/dev/zero of=/tmp/empty.bin bs=1M count=10
ls -lh /tmp/empty.bin
# 生成了一个 10MB 的全零文件

# 随机数设备 —— 产生无限的随机数据
dd if=/dev/urandom of=/tmp/random.bin bs=1K count=1
hexdump -C /tmp/random.bin | head -5

# 空设备 —— 测试用的"黑洞"
# 扔掉所有输出（常用于禁止命令的屏幕输出）
find / -name "*.log" > /dev/null 2>&1

# 内存设备 —— 把内存当磁盘用
# /dev/shm 是一个挂在内存中的文件系统（tmpfs），速度极快
df -h /dev/shm
echo "fast storage" > /dev/shm/test.txt
cat /dev/shm/test.txt
rm /dev/shm/test.txt

# 清除
rm /tmp/empty.bin /tmp/random.bin 2>/dev/null
```

### 7.3 进程即文件：/proc 深度探索

```bash
# === 查看系统信息（都是"读文件"操作）===

# CPU 型号和核心数量
echo "CPU 型号:"
grep "model name" /proc/cpuinfo | head -1
echo "CPU 核心数:"
grep -c "processor" /proc/cpuinfo

# 内存使用情况
echo "总内存:"
grep "MemTotal" /proc/meminfo
echo "可用内存:"
grep "MemAvailable" /proc/meminfo

# 系统负载
cat /proc/loadavg

# 内核启动参数（boot 时传递的参数）
cat /proc/cmdline


# === 查看进程信息 ===

# 列出所有进程（数字目录名就是 PID）
ls /proc | grep -E '^[0-9]+$' | head -20

# 查看 PID=1 的进程详情（通常是 systemd 或 init）
echo "进程名: $(cat /proc/1/comm)"
echo "进程状态:"
head -3 /proc/1/status

# 查看当前 Shell 进程的环境变量
cat /proc/$$/environ | tr '\0' '\n' | head -10

# 查看进程的内存映射（虚拟地址空间布局）
head -20 /proc/$$/maps
```

### 7.4 硬件即文件：/sys 深度探索

```bash
# === 查看硬件设备树 ===

# 系统中有哪些块设备
ls /sys/block/

# 查看第一个磁盘的容量（扇区数）
if [ -f /sys/block/sda/size ]; then
    SIZE=$(cat /sys/block/sda/size)
    echo "磁盘扇区数: $SIZE"
    echo "磁盘容量: $(($SIZE * 512 / 1024 / 1024 / 1024)) GB（假设扇区=512B）"
fi

# 查看 CPU 信息
ls /sys/devices/system/cpu/
cat /sys/devices/system/cpu/possible  # 可能的 CPU 编号
cat /sys/devices/system/cpu/present   # 当前存在的 CPU

# 查看网卡信息
ls /sys/class/net/
# 输出类似: eth0 lo wlan0

# 查看某个网卡的 MAC 地址
if [ -f /sys/class/net/eth0/address ]; then
    cat /sys/class/net/eth0/address
fi

# === 修改内核参数（通过写文件改变系统行为）===

# 开启 IP 转发（让 Linux 变成路由器）
# 注意：这个操作需要 root 权限，仅作演示
# echo 1 > /proc/sys/net/ipv4/ip_forward

# 查看当前是否开启了 IP 转发
cat /proc/sys/net/ipv4/ip_forward
# 输出 0 = 未开启，1 = 已开启
```

### 7.5 管道：无名文件

管道是 Linux 最经典的 IPC（进程间通信）机制之一，它也是一种"文件"：

```bash
# 管道的本质：一个内核缓冲区，可以被当作文件读写

# 匿名管道 —— 最常用的 |
ls -l / | grep "^d" | wc -l
# ls 的输出通过管道传给 grep，grep 的输出通过管道传给 wc

# 命名管道（FIFO）—— 一个有名字的"管道文件"
mkfifo /tmp/my_pipe
ls -l /tmp/my_pipe
# 注意文件类型：p（p = pipe）
# prw-r--r-- 1 user user 0 Jul 22 10:00 /tmp/my_pipe

# 终端1：往管道写数据（会阻塞，直到有人读取）
# echo "Hello via pipe" > /tmp/my_pipe

# 终端2：从管道读数据
# cat /tmp/my_pipe

# 清理
rm /tmp/my_pipe
```

---

## 8. 常见误区与深入理解

### 8.1 误区：所有东西真的都是"普通文件"吗？

不是。说"一切皆文件"指的是**统一的操作接口**，并不意味着所有东西真的存储在硬盘上。

| 类型 | 存储位置 | read 的来源 | write 的去向 |
|------|---------|------------|-------------|
| 普通文件 | 磁盘扇区 | 从磁盘读取 | 写入磁盘 |
| `/proc/cpuinfo` | **无**（实时生成） | 内核动态构造 | 通常只读，写入报错 |
| `/dev/sda` | 硬盘本身 | 从硬盘读取原始数据 | 直接写入磁盘扇区 |
| `/dev/null` | **无**（数据消失） | 永远返回 EOF | 丢弃所有数据 |
| `/dev/urandom` | **无**（算法生成） | 内核随机数算法 | 通常只读 |
| 管道 `|` | **内核缓冲区** | 从缓冲区读 | 写入缓冲区 |

### 8.2 误区：/proc 和 /sys 的文件可以随便修改吗？

**不能！**`/proc` 和 `/sys` 中的大部分文件是**只读的**。少数文件可写，但写入的不是"存储数据"，而是**改变内核行为**。

```bash
# 错误示范：试图"删除" /proc/cpuinfo
# rm /proc/cpuinfo
# 结果：rm: cannot remove '/proc/cpuinfo': Operation not permitted

# 正确示例：读取硬件温度（如果系统支持）
cat /sys/class/thermal/thermal_zone0/temp 2>/dev/null

# 可写的例子（需 root 权限，仅作概念演示）：
# echo "performance" > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# 这行命令改变了 CPU 调频策略 —— 通过"写文件"来配置内核！
```

### 8.3 误区：文件描述符就是文件本身

**不是。** 文件描述符只是一个**整数编号**（0, 1, 2, 3...），是进程内部用来指代"打开的文件"（file 对象）的句柄。

```bash
# 同一个进程可以打开同一个文件两次
# 它们有不同的文件描述符，但有各自的偏移量

exec 3<> /tmp/fd_demo.txt   # 以读写方式打开，分配 fd=3
echo "line 1" >&3

exec 4<> /tmp/fd_demo.txt   # 再次打开同一个文件，分配 fd=4
echo "line 2" >&4

# 查看文件内容
cat /tmp/fd_demo.txt

# 关闭文件描述符
exec 3>&-
exec 4>&-
rm /tmp/fd_demo.txt
```

### 8.4 初学者常见错误速查

| 错误操作 | 为什么错 | 应该怎么做 |
|---------|---------|-----------|
| `rm /proc/cpuinfo` | `/proc` 是伪文件系统，文件是内核实时生成的，不能删除 | 不需要删除，它不占磁盘空间 |
| `cat /dev/sda > backup.img`（磁盘满了还在写） | 直接 dump 整个磁盘，文件可能非常大 | 先 `df -h` 确认磁盘空间 |
| 在 `/sys` 里随便 `echo` | `/sys` 多数文件是只读的，乱写可能触发内核错误 | 只写文档明确说明可写的文件 |
| 以为 `ls -l` 看到的大小就是 inode 大小 | `ls -l` 显示的是文件内容大小，inode 还有自己的额外开销 | 用 `stat` 查看完整的元数据 |
| 混淆 `rm` 和 `unlink` | `rm` 实际上调用的是 `unlink`（减少硬链接计数），不是直接"删除" | 理解硬链接机制后再用 `rm` |

---

## 本讲结构图

```
┌─────────────────────────────────────┐
│          用户应用                    │
│  (fopen/fread/fwrite)              │
├─────────────────────────────────────┤
│          C 库函数 (glibc)           │
├─────────────────────────────────────┤
│          系统调用                    │
│  (sys_open/sys_read/sys_write)     │
├─────────────────────────────────────┤
│  ┌───────────────────────────────┐  │
│  │        VFS 抽象层             │  │
│  │  ┌──────────┐ ┌──────────┐   │  │
│  │  │inode_ops │ │file_ops  │   │  │
│  │  │┌──────┐│ │┌──────┐│   │  │
│  │  ││create ││ ││open  ││   │  │
│  │  ││mkdir  ││ ││read  ││   │  │
│  │  ││unlink ││ ││write ││   │  │
│  │  ││rename ││ ││release││   │  │
│  │  │└──────┘│ │└──────┘│   │  │
│  │  └──────────┘ └──────────┘   │  │
│  └───────────────────────────────┘  │
├─────────┬──────┬──────┬─────────────┤
│  ext4   │ FAT32│procfs│   sysfs     │
└─────────┴──────┴──────┴─────────────┘
```

---

## 本讲总结

### 核心概念速查

| 概念 | 说明 |
|------|------|
| **一切皆文件** | Linux 将所有设备抽象为文件，统一接口 |
| **VFS** | 虚拟文件系统，底层抽象层 |
| **super_block** | 文件系统实例 |
| **inode** | 文件元数据（权限/大小等） |
| **dentry** | 目录项（路径 → inode 映射） |
| **file** | 已打开的文件（含偏移量） |
| **inode_operations** | 目录管理接口（create/mkdir/unlink） |
| **file_operations** | 文件操作接口（open/read/write） |
| **调用链** | 应用 → C库 → 系统调用 → VFS → 具体文件系统 |

### 关键命令速查

| 命令 | 作用 | 示例 |
|------|------|------|
| `stat` | 查看文件 inode 详细信息 | `stat /etc/passwd` |
| `ls -i` | 查看文件 inode 编号 | `ls -i /etc/passwd` |
| `ls -l /proc/$$/fd/` | 查看当前进程的文件描述符 | — |
| `strace` | 追踪系统调用 | `strace cat /etc/hostname` |
| `lsof` | 列出进程打开的文件 | `lsof -p $$` |
| `mount` | 查看挂载的文件系统 | `mount \| head -20` |
| `df -T` | 查看文件系统类型和空间 | `df -T` |
| `cat /proc/cpuinfo` | 查看 CPU 信息 | — |
| `cat /proc/meminfo` | 查看内存信息 | — |
| `ls /sys/block/` | 查看块设备 | — |
| `ls -l /dev/` | 查看设备文件 | — |
| `mkfifo` | 创建命名管道 | `mkfifo /tmp/my_pipe` |

### 一句话总结

在 Linux 中，"文件"不只是一个存储在硬盘上的数据块。它是一个**抽象概念**，代表任何可以被 `open/read/write/close` 操作的东西。硬盘、键盘、显示器、进程、内存、网络、甚至黑洞（/dev/null），全部都用这同一套接口。这就是 Linux 最核心的设计哲学：**Everything is a file**。

---

## 9. 扩展阅读与思考

### 9.1 推荐实验

如果你有一台 Linux 机器（或虚拟机/WSL），建议亲手做以下实验：

1. **把终端当文件玩**：打开两个终端，在终端1执行 `tty` 得到终端名（如 `/dev/pts/1`），然后在终端2执行 `echo "Surprise!" > /dev/pts/1`。观察终端1发生了什么。

2. **用文件描述符做重定向**：尝试 `exec 3> /tmp/my_log.txt; echo "log entry" >&3; exec 3>&-`，然后 `cat /tmp/my_log.txt`。

3. **创造自己的 /proc 条目**：编写一个简单的内核模块（当你学到驱动开发时），在 `/proc` 下创建自己的条目，然后从用户空间 `cat` 它。

### 9.2 思考题

1. 为什么 `/proc/cpuinfo` 的 `ls -l` 显示大小为 0，但 `cat /proc/cpuinfo` 却能输出内容？
2. 如果一个文件的所有硬链接都被 `rm` 删除了，但还有一个进程仍然打开着这个文件（有 file 对象指向它的 inode），文件数据会被立即删除吗？为什么？
3. `cp` 和 `mv` 在 inode 层面有什么区别？（提示：`cp` 创建新 inode，`mv` 通常只修改 dentry）

---

*笔记生成日期：2026-07-22 | 优化日期：2026-07-24 | 转写：Whisper tiny*
