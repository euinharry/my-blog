---
title: "第6讲：Linux文件目录"
date: 2026-09-11T09:23:00+08:00
draft: false
description: "该标准对根目录（/）下的每一个子目录做了详细的功能规定和说明。"
series: ["Linux 入门"]
series_order: 4
categories: ["技术笔记"]
tags: ["Linux", "文件系统"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P7)
> **标题**：第6讲 -- Linux文件目录
> **时长**：28分18秒
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. 文件目录标准（FHS）

### 1.1 为什么需要目录标准？

**问题背景**：
- Linux是开源免费的内核，任何公司和开发者都可以使用Linux开发自己的产品
- 如果没有统一标准，每个公司都会按自己的喜好组织文件系统
- 在Linux开发中经常需要协作，如果大家的目录结构都不一样，协作成本极高

**解决方案**：
> Linux基金会提出了 **FHS（Filesystem Hierarchy Standard）** 文件系统层次结构标准

### 1.2 FHS 3.0 标准

| 项目 | 内容 |
|------|------|
| **最新版本** | FHS 3.0 |
| **发布日期** | 2015年6月3日 |
| **文档页数** | 50+ 页 |
| **获取方式** | Linux基金会官方网站 |

该标准对根目录（/）下的每一个子目录做了详细的功能规定和说明。

> **补充说明**：FHS 3.0 是目前的现行标准。虽然发布于2015年，但大多数现代Linux发行版（Ubuntu、Debian、Fedora、RHEL等）仍然遵循此标准。标准规定了每个目录"应该存放什么"，而不是"必须存放什么"，各发行版会有细微差异。

---

## 2. 根目录（/）下各子目录详解

> 在Ubuntu系统中，可以通过文件管理器 -> 其他位置 -> 计算机（/）来查看根目录下的所有子目录。

### 2.1 /bin -- 基础命令（二进制文件）

| 属性 | 说明 |
|------|------|
| **全称** | Binary（二进制） |
| **存放内容** | 系统最基础、最核心的二进制命令文件 |
| **典型命令** | cp（复制）、ls（列出目录内容）、mv、cat 等 |
| **访问权限** | 所有用户均可使用 |

**小实验**：临时移除 ls 命令后，系统会提示"没有那个文件或目录"，恢复后才能正常工作。

> **结论**：/bin 目录中的命令是系统正常运行的基础。

> **现代Linux补充**：在较新的发行版（Ubuntu 20.04+、Debian 10+）中，/bin 通常是 /usr/bin 的符号链接。这是"UsrMerge"运动的一部分，目的是简化目录结构。

```bash
# 查看 /bin 是否是指向 /usr/bin 的符号链接
ls -l / | grep bin

# 查看 /bin 中有哪些常用命令
ls -l /bin | head -20

# 统计 /bin 下命令总数
ls /bin | wc -l
```

### 2.2 /boot -- 启动文件

| 属性 | 说明 |
|------|------|
| **全称** | Boot（启动） |
| **存放内容** | 系统启动相关的文件 |
| **关键文件** | vmlinuz -- Linux内核镜像（没有它系统无法启动） |
| | grub/ -- GRUB引导程序（类似嵌入式Linux中的U-Boot） |

```bash
# 查看 /boot 目录内容
ls -lh /boot

# 查看当前系统使用的内核版本
uname -r

# 查看 GRUB 配置文件
cat /boot/grub/grub.cfg | head -20
```

### 2.3 /cdrom -- CD-ROM（已淘汰）

> 以前用于挂载CD镜像，现在已逐渐被淘汰。多媒体设备主要挂载在 /media 目录下。

### 2.4 /dev -- 设备文件

| 属性 | 说明 |
|------|------|
| **全称** | Device（设备） |
| **存放内容** | 所有设备对应的文件 |
| **重要性** | Linux应用编程中操作外设，本质就是通过 /dev 下的设备文件来控制 |

**常见设备文件示例**：

| 设备文件 | 说明 |
|---------|------|
| /dev/sda | 第一块SCSI/SATA硬盘 |
| /dev/sda1 | 第一块硬盘的第一个分区 |
| /dev/sdb | 第二块SCSI/SATA硬盘（如U盘） |
| /dev/tty | 当前终端设备 |
| /dev/ttyUSB0 | USB转串口设备0 |
| /dev/null | 黑洞设备（写入的数据被丢弃） |
| /dev/zero | 零设备（读取时返回无限的零字节） |
| /dev/random | 随机数生成器（真随机，可能阻塞） |
| /dev/urandom | 非阻塞随机数生成器 |
| /dev/loop0 | 回环设备（用于挂载镜像文件） |

```bash
# 查看系统中的块设备
lsblk

# 查看所有磁盘分区信息（需要root权限）
fdisk -l

# 查看 /dev 目录下的设备文件（部分）
ls -l /dev | head -30

# 查看系统中的字符设备和块设备数量
ls -l /dev | grep '^c' | wc -l   # 字符设备数量
ls -l /dev | grep '^b' | wc -l   # 块设备数量
```

### 2.5 /etc -- 配置文件

| 属性 | 说明 |
|------|------|
| **全称** | Etcetera（配置） |
| **存放内容** | 系统配置文件 + 应用程序配置文件 |
| **示例** | apt/（包管理器配置）、dhcp/（动态IP分配配置） |

**常用配置文件**：

| 配置文件 | 说明 |
|---------|------|
| /etc/passwd | 用户账户信息 |
| /etc/shadow | 用户密码（加密存储） |
| /etc/group | 用户组信息 |
| /etc/fstab | 文件系统挂载表（开机自动挂载） |
| /etc/hostname | 主机名 |
| /etc/hosts | 静态主机名-IP映射 |
| /etc/resolv.conf | DNS服务器配置 |
| /etc/ssh/sshd_config | SSH服务端配置 |
| /etc/sudoers | sudo权限配置 |

```bash
# 查看 /etc 目录结构（只看子目录）
ls -d /etc/*/ | head -20

# 查看当前系统主机名
cat /etc/hostname

# 查看用户列表
cat /etc/passwd | head -10

# 查看开机自动挂载的配置
cat /etc/fstab
```

### 2.6 /home -- 用户主目录

| 属性 | 说明 |
|------|------|
| **全称** | Home（家目录） |
| **存放内容** | 所有普通用户的主目录 |
| **机制** | 每新建一个用户，/home 下会自动新增一个对应的子目录 |

```bash
# 查看当前系统有哪些用户
ls -l /home

# 查看当前用户的主目录大小
du -sh ~

# 查看当前用户的主目录
echo $HOME
```

### 2.7 /lib -- 库文件（32位）

| 属性 | 说明 |
|------|------|
| **全称** | Library（库） |
| **存放内容** | /bin 和 /sbin 中应用程序所需的库文件 |
| **适用架构** | 32位系统 |

> **现代Linux补充**：在采用UsrMerge的发行版中，/lib 通常是 /usr/lib 的符号链接。

```bash
# 查看 /lib 目录
ls -l / | grep lib

# 查看某个程序依赖哪些动态库
ldd /bin/ls
```

### 2.8 /lib64 -- 库文件（64位）

与 /lib 功能类似，但针对64位（x86_64）系统。在64位系统上，64位库文件存放于此，32位兼容库存放在 /lib 或 /usr/lib32。

```bash
# 查看64位库文件
ls -l /lib64 | head -10

# 查看库文件总数
ls /lib64 | wc -l
```

### 2.9 /media -- 可移动媒体挂载点

| 属性 | 说明 |
|------|------|
| **用途** | 挂载多媒体设备（U盘、光盘等） |
| **示例** | U盘插入后自动挂载到 /media/用户名/U盘标签/ |

```bash
# 查看当前挂载的可移动媒体
ls -l /media

# 查看所有已挂载的文件系统
mount | grep media
```

### 2.10 /mnt -- 临时挂载点

| 属性 | 说明 |
|------|------|
| **全称** | Mount（挂载） |
| **用途** | 临时挂载文件系统（管理员手动挂载设备时常使用此目录） |

**典型使用场景**：
- 手动挂载U盘或外接硬盘：`mount /dev/sdb1 /mnt`
- 挂载ISO镜像文件：`mount -o loop ubuntu.iso /mnt`
- 挂载网络共享（NFS/SMB）：`mount -t nfs server:/share /mnt`

```bash
# 查看 /mnt 目录内容
ls -l /mnt

# 查看所有挂载点
mount | column -t | head -20
```

### 2.11 /opt -- 可选软件包

| 属性 | 说明 |
|------|------|
| **全称** | Optional（可选） |
| **用途** | 软件测试：将待测试的软件及其依赖全部安装到此目录 |
| **优势** | 测试完成后直接清空 /opt 即可完整卸载，不影响系统 |

**典型使用场景**：
- 安装第三方商业软件（如VMware Tools、Google Chrome独立包）
- 安装开发工具链（如交叉编译工具链 arm-linux-gnueabihf-gcc）
- 存放自行编译的软件

```bash
# 查看 /opt 目录下安装了什么
ls -l /opt

# 查看 /opt 占用的磁盘空间
du -sh /opt
```

### 2.12 /proc -- 进程信息（虚拟文件系统）

| 属性 | 说明 |
|------|------|
| **全称** | Process（进程） |
| **类型** | 虚拟文件系统（存在于内存中，重启后内容清零） |
| **存放内容** | 所有正在运行的进程信息 + 内核运行状态信息 |

结构：/proc/ 下有很多以数字命名的子目录（如 /proc/1/、/proc/13/），这些数字就是运行中的进程PID。

**常用信息文件（原视频+补充）**：

| 文件 | 说明 | 示例内容 |
|------|------|---------|
| /proc/cpuinfo | CPU详细信息（型号、主频、核心数） | Intel i5-9400F @ 2.9GHz |
| /proc/meminfo | 内存使用信息（总量、空闲、缓存） | MemTotal: 16GB |
| /proc/consoles | 控制台信息 | tty0 |
| /proc/loadavg | 系统负载平均值（1分钟、5分钟、15分钟） | 0.15 0.10 0.05 1/324 12345 |
| /proc/version | 内核版本信息 | Linux version 5.15.0... |
| /proc/uptime | 系统运行时间（秒）及空闲时间 | 123456.78 98765.43 |
| /proc/filesystems | 内核支持的文件系统列表 | ext4, xfs, btrfs... |
| /proc/partitions | 磁盘分区信息 | sda, sda1, sda2... |
| /proc/mounts | 当前所有挂载点 | 比 /etc/mtab 更准确的实时数据 |
| /proc/swaps | 交换分区使用情况 | /swapfile |
| /proc/net/dev | 网络设备收发统计 | eth0 收发包数 |
| /proc/sys/ | 内核参数配置目录（可动态修改） | 系统调优 |

```bash
# 查看CPU信息
cat /proc/cpuinfo
# 输出示例：Intel i5-9400F @ 2.9GHz

# 查看内存信息
cat /proc/meminfo | head -5

# 查看系统负载（1分钟、5分钟、15分钟平均值）
cat /proc/loadavg

# 查看内核版本
cat /proc/version

# 查看系统已运行时间（秒）
cat /proc/uptime

# 查看当前有哪些进程在运行
ls /proc | grep -E '^[0-9]+$' | head -10

# 查看某个进程的状态（以systemd为例，PID=1）
cat /proc/1/status | head -15

# 查看系统支持的文件系统
cat /proc/filesystems
```

> **关于 loadavg 的解读**：三个数字分别表示过去1分钟、5分钟、15分钟的CPU平均负载。如果这个值超过CPU核心数，说明系统过载。例如4核CPU，loadavg超过4.0说明有进程在等待CPU资源。

### 2.13 /root -- 超级用户主目录

| 属性 | 说明 |
|------|------|
| **用户** | 系统管理员（root） |
| **访问权限** | 普通用户无权访问（需要认证） |

```bash
# 普通用户尝试访问会显示"权限不够"
ls /root

# 使用 sudo 可以查看（需要sudo权限）
sudo ls -la /root
```

### 2.14 /run -- 运行时信息

| 属性 | 说明 |
|------|------|
| **类型** | tmpfs 临时文件系统（内存中，重启后清空） |
| **存放内容** | 系统启动以来各服务的运行时数据 |
| **出现背景** | FHS 3.0 新增的目录，替代以前分散在 /var/run 的运行时文件 |

**存放内容举例**：

| 子目录/文件 | 说明 |
|------------|------|
| /run/user/1000/ | 用户1000的运行时目录 |
| /run/lock/ | 锁文件（防止程序重复运行） |
| /run/sshd.pid | SSH守护进程的PID文件 |
| /run/systemd/ | systemd 管理相关运行时数据 |
| /run/utmp | 当前登录用户记录（who命令的数据来源） |

```bash
# 查看 /run 目录
ls -l /run

# 查看 /run 使用的文件系统类型
df -Th /run

# 查看当前登录的用户
who

# 查看系统启动以来的时间
uptime
```

> **为什么需要 /run ？** 在FHS 2.3时代，运行时数据（如PID文件、锁文件）存放在 /var/run。但 /var 可能在启动早期尚未挂载（/var 可以是独立分区），导致需要运行时数据的服务无法启动。/run 作为tmpfs总是在内存中可用，解决了这个问题。

### 2.15 /sbin -- 系统管理命令

| 属性 | 说明 |
|------|------|
| **全称** | System Binary（系统二进制） |
| **与 /bin 的区别** | /sbin 中的命令只有 root 用户才能执行；/bin 中的命令所有用户均可使用 |

**典型命令**：
- `fdisk` -- 磁盘分区工具
- `mkfs` -- 创建文件系统
- `ifconfig` -- 网络接口配置
- `iptables` -- 防火墙规则配置
- `shutdown` -- 关机/重启

```bash
# 查看 /sbin 中有哪些系统管理命令
ls -l /sbin | head -20

# 统计数量
ls /sbin | wc -l

# 查看某个命令的位置
which fdisk
```

### 2.16 /snap -- Snap包管理器

Ubuntu系统的新型软件包管理工具。Snap是Canonical推出的跨发行版软件包格式，将应用及其所有依赖打包在一起。

```bash
# 查看已安装的snap包
snap list

# 查看 /snap 目录
ls -l /snap
```

### 2.17 /srv -- 服务数据

| 属性 | 说明 |
|------|------|
| **全称** | Service（服务） |
| **存放内容** | 网络服务相关的数据文件 |

**典型用法**：
- /srv/www/ -- Web服务器（Apache/Nginx）的网站文件
- /srv/ftp/ -- FTP服务器的文件
- /srv/git/ -- Git服务器的仓库

> 实际使用中，很多管理员更习惯将网站放在 /var/www，这也是常见做法。

```bash
# 查看 /srv 目录
ls -l /srv
```

### 2.18 /sys -- 系统硬件接口（sysfs）

| 属性 | 说明 |
|------|------|
| **全称** | System |
| **类型** | 虚拟文件系统（sysfs，存在于内存中） |
| **用途** | 提供硬件操作的接口，向用户空间导出内核对象信息 |
| **内核版本** | Linux 2.6+ 引入 |
| **对比 /proc** | /proc 是进程+杂项信息，/sys 专注于设备和驱动的结构化视图 |

**主要子目录**：

| 子目录 | 说明 |
|--------|------|
| /sys/block/ | 块设备信息（如sda），可查看设备大小、是否可移动 |
| /sys/class/ | 按功能分类的设备（如net/网卡、tty/终端、backlight/背光） |
| /sys/bus/ | 按总线类型分类的设备（如pci、usb、i2c、spi） |
| /sys/devices/ | 所有设备的层级树（最底层视图） |
| /sys/kernel/ | 内核参数和统计信息 |
| /sys/power/ | 电源管理（休眠、唤醒） |

**实际操作示例**：

```bash
# 查看块设备列表
ls /sys/block/

# 查看第一块硬盘的大小（扇区数）
cat /sys/block/sda/size

# 查看某块硬盘是否为可移动设备（1=可移动，0=固定）
cat /sys/block/sda/removable

# 查看网络接口列表
ls /sys/class/net/

# 查看背光设备（笔记本电脑屏幕亮度控制）
ls /sys/class/backlight/

# 查看USB设备树
ls /sys/bus/usb/devices/

# 控制LED（以ThinkPad为例，需要root权限）
echo 1 > /sys/class/leds/tpacpi::power/brightness  # 打开电源LED
echo 0 > /sys/class/leds/tpacpi::power/brightness  # 关闭电源LED
```

> **/sys 与嵌入式开发的关系**：在嵌入式Linux驱动开发中，/sys 是非常重要的调试和测试接口。开发LED驱动后，可以通过 `/sys/class/leds/` 来测试LED是否能正常亮灭。驱动开发中常说的"sysfs接口"就是指 /sys 下的这些文件。

### 2.19 /tmp -- 临时文件

| 属性 | 说明 |
|------|------|
| **全称** | Temporary（临时） |
| **存放内容** | 临时存储的文件（如游戏产生的中间临时文件） |
| **生命周期** | 通常系统重启后清空（取决于发行版配置） |

```bash
# 查看 /tmp 目录
ls -l /tmp

# 查看 /tmp 占用的空间
du -sh /tmp

# 创建临时文件测试
echo "test" > /tmp/test.txt
cat /tmp/test.txt
rm /tmp/test.txt
```

### 2.20 /usr -- 用户系统资源（最大目录）

| 属性 | 说明 |
|------|------|
| **全称** | Unix System Resources |
| **特点** | 存放了系统中大部分软件，是文件系统中最大的目录 |
| **大小** | 通常以GB计 |

> **历史背景**：早期Unix系统中，/usr 确实是"user"的缩写，用于存放用户文件。后来用户文件被移到 /home，/usr 转为存放系统级软件资源，但名称保留了下来。

**/usr 子目录详细结构**：

| 子目录 | 说明 | 典型内容 |
|--------|------|---------|
| /usr/bin | 用户级命令（非系统启动必须） | gcc, python, git, vim, firefox |
| /usr/sbin | 用户级系统管理命令 | sshd, httpd, nginx |
| /usr/lib | 库文件（32位架构，与程序配合） | libc.so, libpthread.so |
| /usr/lib64 | 库文件（64位架构） | libc-2.31.so |
| /usr/local | 本地安装的软件（不受系统包管理器管理） | 自行编译安装的软件 |
| /usr/share | 架构无关的共享数据 | 文档、图标、字体、man手册 |
| /usr/src | 源代码（主要是内核源码） | linux-headers-5.15.0 |
| /usr/include | C/C++ 头文件 | stdio.h, stdlib.h, pthread.h |
| /usr/libexec | 被其他程序调用的可执行文件（非用户直接调用） | 系统服务的辅助程序 |

```bash
# 查看 /usr 各子目录的大小
du -sh /usr/* | sort -rh | head -15

# 查看 /usr 总大小
du -sh /usr

# 查看 /usr/bin 中有多少命令
ls /usr/bin | wc -l

# 查看 /usr/local 下安装了哪些软件
ls -l /usr/local/

# 查看 /usr/local/bin 中的可执行文件（通常是自己编译安装的）
ls -l /usr/local/bin/

# 查看 /usr/share 下的内容
ls /usr/share/ | head -20

# 查看 man 手册目录
ls /usr/share/man/

# 查找某个命令属于 /usr 下的哪个目录
which gcc
which python3
```

> **/usr/bin 与 /bin 的区别**：在传统系统中，/bin 存放系统启动必需的命令（装在小而快的分区），/usr/bin 存放不那么紧急的用户级命令（可以在启动后挂载 /usr 分区再使用）。在采用UsrMerge的现代系统中，/bin 是 /usr/bin 的符号链接，两者已合并。

> **/usr/local 的典型使用场景**：当你从源码编译安装软件时（`./configure && make && make install`），默认会安装到 /usr/local。这样做的好处是：系统包管理器（apt/dnf）安装的软件在 /usr/bin，自己编译的在 /usr/local/bin，互不干扰。卸载时直接删除 /usr/local 下的对应文件即可。

### 2.21 /var -- 可变数据

| 属性 | 说明 |
|------|------|
| **全称** | Variable（可变） |
| **存放内容** | 经常变化的文件 |
| **示例** | 软件运行错误信息、系统日志、电子邮件等 |

**主要子目录**：

| 子目录 | 说明 |
|--------|------|
| /var/log | 系统和应用程序日志文件 |
| /var/spool | 待处理的任务队列（邮件、打印任务、cron） |
| /var/cache | 应用程序缓存数据 |
| /var/lib | 应用程序运行时状态数据（数据库文件等） |
| /var/tmp | 持久化的临时文件（重启后不删除，与/tmp不同） |
| /var/run | 运行时文件（已被FHS 3.0中独立的 /run 替代，通常为符号链接） |

```bash
# 查看系统日志
ls -l /var/log/

# 查看系统主日志（syslog）
sudo tail -20 /var/log/syslog

# 查看认证日志（登录记录）
sudo tail -10 /var/log/auth.log

# 查看 /var 各子目录大小
du -sh /var/* | sort -rh | head -10

# 查看 /var 总大小
du -sh /var
```

---

## 3. Linux常见文件类型

通过 ls -l 命令查看文件详情时，每行最开头的字符代表文件类型：

| 类型标识 | 文件类型 | 说明 |
|---------|---------|------|
| - | **普通文件（Regular File）** | 最常见文件类型，如文本文件、源码等 |
| d | **目录文件（Directory）** | 文件夹，如 /bin、/etc 等 |
| l | **符号链接（Symbolic Link）** | 类似Windows快捷方式，指向另一个文件 |
| c | **字符设备文件（Character Device）** | 按字符流方式访问的设备，如 /dev/tty |
| b | **块设备文件（Block Device）** | 按块方式访问的设备，如 /dev/sda |
| s | **套接字文件（Socket）** | 用于进程间通信（本地socket） |
| p | **管道文件（FIFO/Pipe）** | 用于进程间通信（命名管道） |

**字符设备 vs 块设备**：

| 对比维度 | 字符设备 (c) | 块设备 (b) |
|---------|------------|-----------|
| 访问方式 | 按字符流（字节）顺序读写 | 按固定大小的块（block）随机读写 |
| 缓冲区 | 通常无缓冲 | 有缓冲区（页缓存） |
| 典型设备 | 键盘、鼠标、串口、终端 | 硬盘、U盘、SSD |
| 文件示例 | /dev/tty, /dev/ttyUSB0, /dev/input/mouse0 | /dev/sda, /dev/nvme0n1, /dev/loop0 |

**更多设备文件示例**：

| 设备文件 | 类型 | 说明 |
|---------|------|------|
| /dev/sda | b | 第一块SATA/SCSI硬盘 |
| /dev/sda1 | b | 第一块硬盘的第一个分区 |
| /dev/nvme0n1 | b | 第一块NVMe固态硬盘 |
| /dev/tty | c | 当前终端 |
| /dev/ttyS0 | c | 第一个串口（COM1） |
| /dev/ttyUSB0 | c | 第一个USB转串口设备 |
| /dev/pts/0 | c | 伪终端（SSH连接时创建） |
| /dev/null | c | 数据黑洞（写入后消失） |
| /dev/zero | c | 无限零字节源 |
| /dev/random | c | 真随机数（可能阻塞） |
| /dev/urandom | c | 伪随机数（不阻塞） |
| /dev/input/mice | c | 鼠标输入设备 |
| /dev/fb0 | c | 第一个帧缓冲（framebuffer，显示屏） |

```bash
# 查看根目录下各文件/目录类型
ls -la /

# 查看 /dev 目录下的设备类型分布
ls -l /dev | head -40

# 只显示字符设备
ls -l /dev | grep '^c' | head -10

# 只显示块设备
ls -l /dev | grep '^b'

# 查看磁盘分区信息（块设备）
lsblk

# 查看磁盘使用情况（带文件系统类型）
df -Th
```

---

## 4. 路径概念：绝对路径 vs 相对路径

### 4.1 绝对路径
- 以 /（根目录）开头
- 从根目录开始完整描述文件位置
- 示例：/home/fire

### 4.2 相对路径
- 不以 / 开头
- 相对于当前位置
- 示例：fire/（假设当前在 /home）

### 4.3 特殊目录符号

| 符号 | 含义 | 示例 |
|------|------|------|
| . 或 ./ | **当前目录** | cd . |
| .. 或 ../ | **上级/父目录** | cd .. |
| ~ | **当前用户的主目录** | cd ~ (= cd /home/用户名) |
| - | **上一次所在的目录** | cd - |

```bash
cd .       # 留在当前目录
cd ..      # 返回上级目录
cd ../..   # 返回上上级目录
cd ~       # 回到用户主目录
cd -       # 回到上一个目录（适合在两个目录间快速切换）

# 查看当前路径
pwd
```

---

## 5. 根目录总览表

| 目录 | 全称 | 功能说明 | 权限 | 类型 |
|------|------|---------|------|------|
| /bin | Binary | 核心命令（ls, cp等） | 所有用户 | 通常为/usr/bin的链接 |
| /boot | Boot | 启动文件（内核vmlinuz、GRUB） | root可写 | 真实目录 |
| /dev | Device | 设备文件 | 内核管理 | devtmpfs（虚拟） |
| /etc | Etcetera | 配置文件 | root可写 | 真实目录 |
| /home | Home | 用户主目录 | 各用户私有 | 真实目录 |
| /lib | Library | 32位库文件 | - | 通常为/usr/lib的链接 |
| /lib64 | Library 64 | 64位库文件 | - | 真实目录或链接 |
| /media | Media | 可移动媒体挂载 | - | 挂载点 |
| /mnt | Mount | 临时挂载点 | - | 挂载点 |
| /opt | Optional | 可选软件（测试用） | - | 真实目录 |
| /proc | Process | 进程/内核信息 | 内核管理 | proc（虚拟） |
| /root | Root | 超级用户主目录 | 仅root | 真实目录 |
| /run | Run | 运行时数据 | - | tmpfs（虚拟） |
| /sbin | System Binary | 系统管理命令 | 仅root | 通常为/usr/sbin的链接 |
| /srv | Service | 服务数据 | - | 真实目录 |
| /sys | System | 硬件接口（sysfs） | 内核管理 | sysfs（虚拟） |
| /tmp | Temporary | 临时文件 | 所有人可写 | 通常为tmpfs |
| /usr | Unix Sys Resources | 系统软件（最大目录） | - | 真实目录 |
| /var | Variable | 可变数据（日志等） | - | 真实目录 |

---

## 6. 常用命令分类汇总

### 6.1 目录结构查看

```bash
# 列出当前目录内容
ls

# 列出详细信息（含文件类型、权限、大小、日期）
ls -l

# 列出所有文件（含隐藏文件，即以 . 开头的文件）
ls -la

# 以人类可读格式显示文件大小（K/M/G）
ls -lh

# 按修改时间排序
ls -lt

# 按文件大小排序
ls -lS

# 递归列出子目录内容
ls -lR /path/

# 以树形结构显示目录（需要安装tree包：sudo apt install tree）
tree /                # 显示根目录的树形结构（非常长，谨慎使用）
tree -L 2 /           # 只显示2层深度
tree -d /             # 只显示目录
tree -h /usr          # 显示/usr的树形结构，带文件大小
```

### 6.2 磁盘空间查看

```bash
# 查看所有已挂载文件系统的使用情况（人类可读格式）
df -h

# 查看指定目录所在分区的使用情况
df -h /home

# 查看文件系统类型
df -Th

# 查看当前目录下每个子目录的大小
du -sh *

# 查看指定目录的总大小
du -sh /usr
du -sh /var

# 查看当前目录下最大的10个文件/目录
du -sh * | sort -rh | head -10

# 查看目录深度统计（显示到指定层级）
du -h --max-depth=1 /usr/ | sort -rh

# 查看inode使用情况
df -i
```

### 6.3 路径与导航

```bash
ls                     # 列出当前目录内容
ls -l                  # 列出详细信息（含文件类型、权限）
ls -la                 # 列出所有文件（含隐藏文件）
ls -lh                 # 以人类可读格式显示文件大小
cd /                   # 进入根目录
cd ..                  # 返回上级目录
cd ~                   # 回到用户主目录
cd -                   # 回到上一个目录
pwd                    # 显示当前路径
```

### 6.4 系统信息查询

```bash
# CPU信息
cat /proc/cpuinfo
lscpu                  # 更友好的CPU信息显示

# 内存信息
cat /proc/meminfo
free -h                # 更友好的内存信息显示

# 系统负载
cat /proc/loadavg
uptime                 # 同时显示运行时间和负载

# 内核版本
cat /proc/version
uname -a               # 更全面的系统信息

# 块设备信息
lsblk                  # 树形显示块设备
lsblk -f               # 显示文件系统类型和UUID

# 硬件总览
lshw -short            # 列出所有硬件（需要sudo）
lspci                  # 列出PCI设备
lsusb                  # 列出USB设备
```

### 6.5 设备与挂载

```bash
# 查看块设备
lsblk
fdisk -l               # 查看详细分区信息（需要root）

# 查看挂载信息
mount | column -t
df -Th

# 挂载U盘示例
sudo mount /dev/sdb1 /mnt
sudo umount /mnt

# 查看设备UUID和文件系统类型
blkid
```

---

*笔记生成日期：2026-07-22 | 转写工具：OpenAI Whisper tiny (CPU)*
*优化日期：2026-07-22 | 补充了实操命令、/usr详细结构、/proc更多文件、/sys详细说明、常用命令分类汇总*
