---
title: "第10讲：使用Linux命令行（下）"
date: 2026-09-11T09:19:00+08:00
draft: false
description: "Linux 文件权限使用三位八进制数字表示，每位数字对应一组用户的权限："
series: ["Linux 入门"]
series_order: 8
categories: ["技术笔记"]
tags: ["Linux", "命令行"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P11)
> **标题**：第10讲 -- 使用Linux命令行（下）
> **时长**：17分07秒
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. 文件权限操作命令

### 1.1 权限数字表示法复习

Linux 文件权限使用三位八进制数字表示，每位数字对应一组用户的权限：

```
r=4, w=2, x=1

r-- = 4+0+0 = 4    (只读)
rw- = 4+2+0 = 6    (读写)
rwx = 4+2+1 = 7    (读写执行)
r-x = 4+0+1 = 5    (读执行)
-wx = 0+2+1 = 3    (写执行)

示例：chmod 777 = 所有用户可读可写可执行
      chmod 644 = Owner读写，Group只读，Other只读
      chmod 755 = Owner读写执行，Group和Other读执行
```

### 1.2 chmod -- 修改文件权限

chmod 支持两种模式：**数字模式**和**符号模式**。

#### 数字模式（最常用）

```bash
chmod 777 文件名   # 所有用户可读可写可执行
chmod 644 文件名   # Owner读写，同组和其他用户只读
chmod 755 文件名   # Owner读写执行，同组和其他用户读执行
```

#### 符号模式（更灵活，可增减特定权限）

符号模式的语法：`chmod [用户类别][操作符][权限] 文件`

**用户类别：**
| 字母 | 含义 |
|------|------|
| `u` | user，文件所有者（Owner） |
| `g` | group，文件所属组用户 |
| `o` | other，其他用户 |
| `a` | all，所有用户（相当于 ugo） |

**操作符：**
| 符号 | 含义 |
|------|------|
| `+` | 增加指定权限 |
| `-` | 移除指定权限 |
| `=` | 设置为指定权限（覆盖原有权限） |

**权限符号：**
| 字母 | 含义 |
|------|------|
| `r` | 读权限 |
| `w` | 写权限 |
| `x` | 执行权限 |
| `X` | 仅在目标已有执行权限或目标是目录时添加执行权限 |
| `s` | SUID 或 SGID（特殊权限） |
| `t` | Sticky Bit（粘滞位） |

**符号模式示例：**

```bash
# 给所有者添加执行权限
chmod u+x script.sh

# 给所有用户添加读权限
chmod a+r file.txt

# 移除同组用户的写权限
chmod g-w file.txt

# 同时设置：Owner读写执行，Group读执行，Other无权限
chmod u=rwx,g=rx,o= config.conf

# 所有人加执行权限（常用，等同于 chmod a+x）
chmod +x script.sh

# 去掉所有人的执行权限
chmod -x script.sh
```

#### 数字模式完整示例

```bash
# 查看当前权限
ls -l 123.txt
# 输出：-rw-r--r-- 1 user group 0 Jul 22 12:00 123.txt
# 解读：-rw-r--r-- 即 644

# 修改为 777（所有人可读写执行）
chmod 777 123.txt
ls -l 123.txt
# 输出：-rwxrwxrwx 1 user group 0 Jul 22 12:00 123.txt

# 使用 -R 递归修改目录及其下所有文件的权限
chmod -R 755 /path/to/directory

# 使用 -v 显示修改了哪些文件的权限
chmod -Rv 755 /path/to/directory
```

#### 递归修改权限（-R 选项）

```bash
# 将整个目录及其所有子文件和子目录设为 755
chmod -R 755 project/

# 注意：-R 会同时修改文件和目录，文件通常不需要执行权限
# 更好的做法：目录设 755，文件设 644
find project/ -type d -exec chmod 755 {} \;
find project/ -type f -exec chmod 644 {} \;
```

#### 实战场景

**场景1：设置脚本可执行**
```bash
# 编写了一个 Shell 脚本，默认没有执行权限
echo '#!/bin/bash' > hello.sh
echo 'echo "Hello World"' >> hello.sh
ls -l hello.sh
# -rw-r--r-- 1 user group 15 Jul 22 12:00 hello.sh

# 添加执行权限
chmod +x hello.sh
# 或者指定精确权限
chmod 755 hello.sh

# 现在可以执行了
./hello.sh
# 输出：Hello World
```

**场景2：保护 SSH 私钥**
```bash
# SSH 私钥必须只有所有者可读写，否则 ssh 会拒绝使用
chmod 600 ~/.ssh/id_rsa
ls -l ~/.ssh/id_rsa
# -rw------- 1 user group 2602 Jul 22 12:00 /home/user/.ssh/id_rsa

# 公钥可以更宽松
chmod 644 ~/.ssh/id_rsa.pub
```

**场景3：设置网站目录权限**
```bash
# 网站根目录：所有者可读写执行，组和其他人可读执行
chmod 755 /var/www/html

# 网站文件：所有者可读写，其他人只读
find /var/www/html -type f -exec chmod 644 {} \;
```

### 1.3 chown -- 修改文件所有者

```bash
# 基本语法
sudo chown 用户名 文件名

# 修改文件的所有者
sudo chown fire 123.txt

# -R 递归修改目录及其下所有文件的所有者
sudo chown -R fire project/

# 同时修改所有者和所属组（用户名:组名）
sudo chown fire:fire 123.txt

# 只修改所属组（等同于 chgrp）
sudo chown :fire 123.txt

# 显示修改了哪些文件
sudo chown -v fire:staff *.txt
```

### 1.4 chgrp -- 修改文件所属组

```bash
# 基本语法
chgrp 组名 文件名

# 将文件的所属组改为 fire
chgrp fire 123.txt

# -R 递归修改目录及其下所有文件
chgrp -R fire project/

# -c 只显示实际发生变化的文件
chgrp -c fire *.txt
```

### 1.5 SUID / SGID / Sticky Bit 特殊权限详解

在 Linux 权限体系中，除了基本的 rwx 九位权限外，还有三个**特殊权限位**，它们分别占据一个额外的八进制位（最前面），与基本权限组合成四位数字。

```
SUID SGID Sticky
 4    2     1

示例：
chmod 4755 = 设置 SUID + 755
chmod 2755 = 设置 SGID + 755
chmod 1755 = 设置 Sticky Bit + 755
chmod 6755 = 设置 SUID + SGID + 755
```

#### SUID（Set User ID，4）

**作用**：当一个可执行文件设置了 SUID 位后，任何用户执行它时，进程的**有效用户ID**会变成**文件所有者**的身份，而不是执行者本身。换句话说，普通用户可以临时获得文件所有者的权限来执行该程序。

**设置方法：**
```bash
# 数字模式（在正常权限前加 4）
chmod 4755 /usr/local/bin/myapp

# 符号模式
chmod u+s /usr/local/bin/myapp

# 取消 SUID
chmod u-s /usr/local/bin/myapp
```

**查看 SUID：** 在 `ls -l` 输出中，如果所有者权限位的 `x` 被 `s` 取代，说明设置了 SUID。
```bash
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root 59976 ... /usr/bin/passwd
#   ^
#   注意这里是 s，不是 x，说明设置了 SUID
```

**典型例子 -- /usr/bin/passwd：**
```bash
# passwd 命令用于修改用户密码，需要写入 /etc/shadow 文件
# 而 /etc/shadow 只有 root 用户可写
ls -l /etc/shadow
# -rw-r----- 1 root shadow 1234 Jul 22 12:00 /etc/shadow

# 普通用户执行 passwd 时，借助 SUID 临时获得 root 权限
# 从而能够写入 /etc/shadow
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root 59976 ... /usr/bin/passwd

# 其他带 SUID 的常见命令
# /usr/bin/sudo   -- 允许普通用户以 root 身份执行命令
# /usr/bin/su     -- 切换用户
# /bin/ping       -- 发送 ICMP 包需要 raw socket 权限
```

> **安全提示**：SUID 是一把双刃剑。如果设置了 SUID 的程序存在漏洞，攻击者可能借此提权。不要在不可信的程序上设置 SUID。

#### SGID（Set Group ID，2）

**SGID 有两种作用**，取决于设置在文件上还是目录上：

**1）设置在可执行文件上**：与 SUID 类似，执行时进程的有效组ID变成文件所属组。
```bash
chmod 2755 /usr/local/bin/myapp
# 或
chmod g+s /usr/local/bin/myapp
```

**2）设置在目录上（更常用）**：在设置了 SGID 的目录下创建的**新文件**会自动继承该目录的所属组，而不是创建者的默认组。这非常适合团队共享目录。

```bash
# 创建共享目录并设置 SGID
sudo mkdir /shared/project
sudo chown :developers /shared/project
sudo chmod 2775 /shared/project    # 2 = SGID

# 任何用户在 /shared/project 下创建文件
# 文件所属组都会自动是 developers，而不是创建者的主组
touch /shared/project/test.txt
ls -l /shared/project/test.txt
# -rw-rw-r-- 1 alice developers 0 ... test.txt
#                 ^^^^^^^^^^ 注意组是 developers
```

**查看 SGID：** 在 `ls -l` 输出中，如果组权限位的 `x` 被 `s` 取代。
```bash
ls -ld /shared/project
# drwxrwsr-x 2 root developers 4096 ... /shared/project
#      ^ 注意这里是 s
```

#### Sticky Bit（粘滞位，1）

**作用**：只对**目录**有效。当目录设置了 Sticky Bit 后，目录中的文件只有**文件所有者**、**目录所有者**或 **root** 才能删除。即使目录权限是 777，普通用户也不能删除别人的文件。

**典型例子 -- /tmp 目录：**
```bash
ls -ld /tmp
# drwxrwxrwt 20 root root 4096 ... /tmp
#         ^ 注意最后一位是 t，不是 x

# /tmp 目录权限是 1777，任何人都可以在 /tmp 创建文件
# 但因为 Sticky Bit，用户 A 不能删除用户 B 在 /tmp 中的文件
```

**设置和取消：**
```bash
# 设置 Sticky Bit
chmod +t /shared/upload
# 或
chmod 1777 /shared/upload

# 取消 Sticky Bit
chmod -t /shared/upload

# 大写 T vs 小写 t：
# 如果目录的 other 位本来有 x 权限，显示为小写 t
# 如果没有 x 权限，显示为大写 T（说明设置无效，因为不能进入目录）
```

**实战场景 -- 共享上传目录：**
```bash
# 创建一个所有人都可以写入但只能删除自己文件的共享目录
sudo mkdir /shared/uploads
sudo chmod 1777 /shared/uploads   # 1 = Sticky Bit, 777 = 全部权限

# 测试
cd /shared/uploads
touch alice.txt    # 用户 alice 创建
su - bob           # 切换到 bob
cd /shared/uploads
rm alice.txt       # 失败！bob 不能删除 alice 的文件
# rm: cannot remove 'alice.txt': Operation not permitted
```

### 1.6 umask -- 默认权限掩码

当用户创建新文件或目录时，系统会根据 umask 值来决定默认权限。

**概念**：umask 是一个掩码，它指定了新建文件/目录时**需要屏蔽的权限位**。新建文件的默认最大权限是 666（rw-rw-rw-），目录是 777（rwxrwxrwx）。umask 的值会从这些最大权限中**减去**。

**计算公式：**
```
新建文件默认权限 = 666 - umask
新建目录默认权限 = 777 - umask
```

> 注意：Linux 不允许新建文件默认有执行权限（出于安全考虑），所以文件的计算结果中执行位始终为 0。

**常用 umask 值：**
| umask 值 | 文件默认权限 | 目录默认权限 | 说明 |
|----------|-------------|-------------|------|
| `0022` | 644 (rw-r--r--) | 755 (rwxr-xr-x) | 大多数 Linux 发行版默认值（root 用户） |
| `0002` | 664 (rw-rw-r--) | 775 (rwxrwxr-x) | Ubuntu 等发行版普通用户默认值 |
| `0077` | 600 (rw-------) | 700 (rwx------) | 最严格，仅所有者可见 |

**查看和设置 umask：**
```bash
# 查看当前 umask 值
umask
# 输出：0022

# 以符号模式查看
umask -S
# 输出：u=rwx,g=rx,o=rx

# 临时修改 umask（仅在当前 Shell 会话有效）
umask 0002

# 永久修改：写入 ~/.bashrc 或 /etc/profile
echo 'umask 0002' >> ~/.bashrc

# 测试 umask 效果
umask 0077
touch test1.txt
mkdir test1_dir
ls -ld test1.txt test1_dir
# -rw------- 1 user group 0 ... test1.txt
# drwx------ 2 user group 40 ... test1_dir
```

> 无特殊需求时，保持系统默认值即可。需要共享文件时，可临时改为 0002。

---
## 2. 磁盘管理类命令

### 2.1 df -- 查看文件系统磁盘使用情况

```bash
df              # 查看所有文件系统信息（块数量、已用、可用、挂载点）
df -h           # 以友好单位显示（G、M、K，推荐使用）
df -T           # 显示文件系统类型（ext4、xfs、tmpfs 等）
df -i           # 显示 inode 使用情况（而非磁盘空间）
df --total      # 在最后一行显示总计
df -h /home     # 只查看指定目录所在文件系统
```

**示例：**
```
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       117G   45G   72G  39% /
tmpfs           3.9G     0  3.9G   0% /dev/shm
/dev/sdb1       500G  320G  180G  64% /data

$ df -Th
Filesystem     Type      Size  Used Avail Use% Mounted on
/dev/sda1      ext4      117G   45G   72G  39% /
tmpfs          tmpfs     3.9G     0  3.9G   0% /dev/shm
/dev/sdb1      xfs       500G  320G  180G  64% /data

# 查看 inode 使用情况（耗尽 inode 时即使有空闲空间也无法创建文件）
$ df -i
Filesystem      Inodes  IUsed  IFree IUse% Mounted on
/dev/sda1      7815168 245678 7569490   4% /
```

### 2.2 du -- 统计文件/目录磁盘使用量

```bash
du 目录名           # 递归统计每个文件和子目录的磁盘使用（默认以KB为单位）
du -h 目录名        # 以友好单位显示（K、M、G）
du -hs 目录名       # -s：只显示总计，不列出每个子项目
du --max-depth=N    # 限制显示深度，N=1 只显示一级子目录
du -sh *            # 统计当前目录下每个文件/目录的大小
du --exclude="*.log"  # 排除匹配模式的文件
du --time           # 同时显示文件的修改时间
```

**示例：**
```bash
# 查看 fire 目录下每个文件和子目录的磁盘占用
$ du -h /home/fire
4.0K    /home/fire/.bashrc
12M     /home/fire/Documents
2.5G    /home/fire/Downloads
2.5G    /home/fire

# 只看总大小
$ du -hs /home/fire
2.5G    /home/fire

# 只看一级子目录（常用于找出哪个目录占空间大）
$ du -h --max-depth=1 /var
120M    /var/cache
1.2G    /var/log
3.5G    /var/lib
4.9G    /var

# 排除日志文件
$ du -h --exclude="*.log" --max-depth=1 /var
```

### 2.3 mount -- 挂载文件系统

Linux 中的一切皆文件，设备（硬盘、U盘、光盘等）必须**挂载**到某个目录后才能访问。

```bash
mount                # 查看当前已挂载的所有文件系统
mount -a             # 挂载 /etc/fstab 中记录的所有文件系统
mount 设备 目录       # 将设备挂载到指定目录
mount -t 类型 设备 目录  # 指定文件系统类型挂载
mount -o 选项 设备 目录  # 带选项挂载（ro只读, rw读写, loop挂载镜像等）
```

**常见示例：**

```bash
# 查看已挂载的所有文件系统
mount | grep "^/dev"

# 挂载 USB 设备（假设设备为 /dev/sdb1，挂载到 /mnt/usb）
sudo mkdir -p /mnt/usb
sudo mount /dev/sdb1 /mnt/usb

# 挂载 ISO 镜像文件
sudo mount -o loop ubuntu-22.04.iso /mnt/iso

# 以只读方式挂载
sudo mount -o ro /dev/sdb1 /mnt/usb

# 重新以读写方式挂载（相当于 remount）
sudo mount -o remount,rw /

# 挂载网络文件系统（NFS）
sudo mount -t nfs 192.168.1.100:/shared /mnt/nfs
```

### 2.4 umount -- 卸载文件系统

```bash
umount 目录          # 通过挂载点卸载
umount 设备          # 通过设备名卸载
umount -a            # 卸载系统所有已挂载的文件系统
umount -f 目录       # 强制卸载（即使设备忙，谨慎使用）
umount -l 目录       # 懒惰卸载（lazy unmount）：立即从目录树分离，等到设备不忙时再真正卸载
```

**常见示例：**

```bash
# 正常卸载
sudo umount /mnt/usb

# 如果提示 "target is busy"，先查看谁在使用
sudo lsof /mnt/usb
# 或
sudo fuser -v /mnt/usb

# 懒惰卸载（不等待所有进程释放设备）
sudo umount -l /mnt/usb

# 强制卸载（风险较高，可能导致数据丢失）
sudo umount -f /mnt/usb
```

> **提示**：在卸载前确保没有终端正在该目录下操作，也没有程序正在读写该设备中的文件，否则会提示 "target is busy"。

### 2.5 lsblk -- 查看块设备树状结构

```bash
lsblk                # 以树状形式显示所有块设备
lsblk -f             # 显示文件系统类型和 UUID
lsblk -m             # 显示所有者、组、权限模式
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT  # 自定义输出列
```

**示例：**
```
$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0 120.0G  0 disk
├─sda1   8:1    0 119.0G  0 part /
├─sda2   8:2    0     1K  0 part
└─sda5   8:5    0   1.0G  0 part [SWAP]
sdb      8:16   0 500.0G  0 disk
└─sdb1   8:17   0 500.0G  0 part /data
sr0     11:0    1  1024M  0 rom

$ lsblk -f
NAME   FSTYPE FSVER LABEL UUID                                 MOUNTPOINT
sda
├─sda1 ext4   1.0         a1b2c3d4-e5f6-7890-abcd-ef1234567890 /
├─sda2
└─sda5 swap   1           f1e2d3c4-b5a6-7890-1234-567890abcdef [SWAP]
sdb
└─sdb1 xfs            data  b2c3d4e5-f6a7-8901-bcde-f12345678901 /data
```

从输出可以看出磁盘分区结构：sda 是系统盘（120G），sdb 是数据盘（500G），sr0 是光驱。

### 2.6 blkid -- 查看设备 UUID 和文件系统类型

```bash
blkid                 # 列出所有设备的 UUID 和文件系统类型
blkid /dev/sda1       # 查看指定设备的详细信息
```

```
$ sudo blkid
/dev/sda1: UUID="a1b2c3d4-e5f6-7890-abcd-ef1234567890" TYPE="ext4" PARTUUID="12345678-01"
/dev/sda5: UUID="f1e2d3c4-b5a6-7890-1234-567890abcdef" TYPE="swap" PARTUUID="12345678-05"
/dev/sdb1: UUID="b2c3d4e5-f6a7-8901-bcde-f12345678901" TYPE="xfs" PARTUUID="87654321-01"
```

> UUID 是设备的唯一标识符，在 `/etc/fstab` 中推荐使用 UUID 而非设备名（如 /dev/sda1），因为设备名可能因插拔顺序而变化。

### 2.7 fdisk -- 查看和管理分区表

```bash
sudo fdisk -l                    # 列出所有磁盘的分区表
sudo fdisk -l /dev/sda           # 查看指定磁盘的分区表
```

```
$ sudo fdisk -l /dev/sda
Disk /dev/sda: 120 GiB, 128849018880 bytes, 251658240 sectors
Device     Boot   Start       End   Sectors   Size Id Type
/dev/sda1  *       2048 249561087 249559040 119.0G 83 Linux
/dev/sda2       249563134 251656191   2093058  1022M  5 Extended
/dev/sda5       249563136 251656191   2093056  1022M 82 Linux swap
```

> **警告**：fdisk 也可以用于创建和删除分区，操作前务必确认磁盘是否正确，误操作可能导致数据永久丢失。

### 2.8 /etc/fstab -- 文件系统自动挂载配置

`/etc/fstab` 是系统启动时自动挂载文件系统的配置文件，共 6 个字段：

```
设备      挂载点    文件系统类型  挂载选项      dump  pass
<device> <mount>   <type>       <options>     <d>   <p>
```

**字段解释：**

| 字段 | 名称 | 说明 |
|------|------|------|
| 第1字段 | 设备 | 可以是设备路径（/dev/sda1）、UUID（UUID=xxx）或 LABEL（LABEL=data） |
| 第2字段 | 挂载点 | 设备挂载的目标目录 |
| 第3字段 | 文件系统类型 | ext4、xfs、ntfs、swap、nfs 等 |
| 第4字段 | 挂载选项 | defaults（默认）、ro（只读）、rw（读写）、noexec（禁止执行）、nosuid 等，多个选项用逗号分隔 |
| 第5字段 | dump | 是否被 dump 备份，0=不备份，1=每天备份（通常设为0） |
| 第6字段 | pass | 开机时 fsck 检查顺序，0=不检查，1=根分区优先检查，2=其他分区 |

**示例 -- /etc/fstab 文件：**

```
# <file system>   <mount point>   <type>   <options>          <dump> <pass>
UUID=a1b2...      /               ext4     defaults           0      1
UUID=f1e2...      none            swap     sw                 0      0
UUID=b2c3...      /data           xfs      defaults           0      2
# 挂载 USB 设备（使用 nofail 避免设备不在时启动失败）
UUID=c3d4...      /mnt/usb        ext4     defaults,nofail    0      2
# 使用 LABEL 挂载
LABEL=Backup      /mnt/backup     ext4     defaults           0      2
```

---
## 3. 网络操作类命令

### 3.1 ping -- 检测网络连通性

```bash
ping 目标地址                # 持续发送 ICMP 包（Ctrl+C 停止）
ping -c N 目标地址           # 发送 N 个包后自动停止
ping -i N 目标地址           # 每隔 N 秒发送一次（默认1秒）
ping -4 目标地址             # 强制使用 IPv4
ping -6 目标地址             # 强制使用 IPv6
ping -s N 目标地址           # 指定数据包大小（字节）
```

**示例：**
```bash
# 发送 4 个包测试连通性
$ ping -c 4 baidu.com
PING baidu.com (110.242.68.66) 56(84) bytes of data.
64 bytes from 110.242.68.66: icmp_seq=1 ttl=52 time=8.50 ms
64 bytes from 110.242.68.66: icmp_seq=2 ttl=52 time=8.45 ms
64 bytes from 110.242.68.66: icmp_seq=3 ttl=52 time=8.32 ms
64 bytes from 110.242.68.66: icmp_seq=4 ttl=52 time=8.28 ms

--- baidu.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 8.28/8.38/8.50/0.09 ms

# 每隔 2 秒发送一次
ping -c 5 -i 2 192.168.1.1

# 测试 IPv6 连通性
ping -6 -c 4 ipv6.google.com
```

### 3.2 ifconfig -- 网卡配置（传统工具）

> ifconfig 依赖于 **net-tools** 工具包。在现代 Linux 发行版中，net-tools 已被逐渐弃用，推荐使用 `ip` 命令替代。
> 安装：`sudo apt install net-tools`（Debian/Ubuntu）或 `sudo yum install net-tools`（CentOS/RHEL）

**查看网卡信息：**
```bash
ifconfig                  # 显示所有已激活的网络端口
ifconfig -a               # 显示所有网卡（包括未激活的）
ifconfig eth0             # 只显示 eth0 的信息
```

```
$ ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.100  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::a00:27ff:fe4e:5b1c  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:4e:5b:1c  txqueuelen 1000  (Ethernet)
        RX packets 12345  bytes 9876543 (9.4 MiB)
        TX packets 6789  bytes 1234567 (1.1 MiB)
```

关键字段说明：
- **inet**：IPv4 地址
- **netmask**：子网掩码
- **broadcast**：广播地址
- **ether**：MAC 地址
- **RX/TX packets**：接收/发送数据包统计

**设置 IP 地址：**
```bash
sudo ifconfig eth0 192.168.1.179              # 修改 IP 地址
sudo ifconfig eth0 192.168.1.179 netmask 255.255.255.0  # 同时设置掩码
```

**启用/禁用网卡：**
```bash
sudo ifconfig eth0 down     # 禁用网卡
sudo ifconfig eth0 up       # 启用网卡
```

### 3.3 ip -- 现代网络管理工具（推荐）

`ip` 命令来自 **iproute2** 工具包，是现代 Linux 发行版推荐的网络管理工具，功能比 ifconfig 更强大。

```bash
# 查看所有网络接口（等价于 ifconfig）
ip addr show
ip a                      # 缩写

# 查看指定接口
ip addr show eth0

# 启用/禁用网卡（等价于 ifconfig up/down）
sudo ip link set eth0 up
sudo ip link set eth0 down

# 设置 IP 地址
sudo ip addr add 192.168.1.100/24 dev eth0
sudo ip addr del 192.168.1.100/24 dev eth0

# 查看路由表（等价于 route -n）
ip route show
ip r                      # 缩写

# 添加/删除默认路由
sudo ip route add default via 192.168.1.1
sudo ip route del default

# 查看 ARP 表（等价于 arp -n）
ip neigh show
```

```
$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 ... state UNKNOWN
    inet 127.0.0.1/8 scope host lo
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ... state UP
    inet 192.168.1.100/24 brd 192.168.1.255 scope global dynamic eth0
```

### 3.4 ss -- 查看网络连接（替代 netstat）

```bash
ss -tuln           # 查看所有 TCP/UDP 监听端口
ss -tunlp          # 加上 -p 显示进程信息（需要 sudo）
ss -s              # 显示网络连接统计摘要
ss -t state established   # 查看已建立的 TCP 连接
```

```bash
# 查看哪些端口正在监听
$ sudo ss -tulnp
Netid State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
tcp   LISTEN 0      128    0.0.0.0:22         0.0.0.0:*     users:(("sshd",pid=1234,fd=3))
tcp   LISTEN 0      128    0.0.0.0:80         0.0.0.0:*     users:(("nginx",pid=5678,fd=6))
tcp   LISTEN 0      128    127.0.0.1:3306     0.0.0.0:*     users:(("mysqld",pid=2345,fd=20))

# 解释：SSH 监听 22 端口，Nginx 监听 80，MySQL 监听 3306（仅本地）
```

`ss` 常见选项：
| 选项 | 含义 |
|------|------|
| `-t` | 只显示 TCP 连接 |
| `-u` | 只显示 UDP 连接 |
| `-l` | 只显示 Listening（监听中）的端口 |
| `-n` | 以数字格式显示（不解析主机名和服务名） |
| `-p` | 显示使用该连接的进程信息 |
| `-a` | 显示所有连接（包括已建立和监听中） |

### 3.5 wget -- 命令行下载工具

```bash
wget 下载地址                  # 下载文件到当前目录
wget -O 文件名 下载地址         # 指定保存的文件名
wget -c 下载地址               # 断点续传（继续未完成的下载）
wget -b 下载地址               # 后台下载（日志写入 wget-log）
wget -q 下载地址               # 静默模式（不输出任何信息）
wget -P 目录 下载地址           # 下载到指定目录
```

**示例：**
```bash
# 下载文件
wget https://example.com/file.tar.gz

# 指定文件名保存
wget -O myfile.tar.gz https://example.com/file.tar.gz

# 断点续传（下载大文件时非常有用）
wget -c https://example.com/large-file.iso

# 后台静默下载
wget -bqc https://example.com/file.tar.gz
```

### 3.6 curl -- 数据传输工具

curl 比 wget 更强大，支持更多协议（HTTP、HTTPS、FTP、SFTP 等），常用于 API 测试。

```bash
curl 网址                     # 获取网页内容并输出到终端
curl -o 文件名 网址            # 保存到文件（类似 wget）
curl -O 网址                  # 使用远程文件名保存
curl -L 网址                  # 跟随 HTTP 重定向
curl -I 网址                  # 只获取 HTTP 响应头
curl -s 网址                  # 静默模式
curl -v 网址                  # 详细输出（用于调试）
curl -X POST -d "key=value" 网址  # 发送 POST 请求
```

**示例：**
```bash
# 获取网页内容
curl https://www.example.com

# 下载文件
curl -o file.tar.gz https://example.com/file.tar.gz

# 只查看响应头（检查服务器信息）
$ curl -I https://www.baidu.com
HTTP/1.1 200 OK
Server: bfe/1.0.8.18
Content-Type: text/html

# 测试 REST API
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name":"John","email":"john@example.com"}'
```

### 3.7 hostname -- 查看/设置主机名

```bash
hostname                  # 查看当前主机名
hostname -I               # 查看本机所有 IP 地址
sudo hostname 新名称       # 临时修改主机名（重启后失效）
sudo hostnamectl set-hostname 新名称  # 永久修改主机名（systemd 系统）
```

```bash
$ hostname
my-server

$ hostname -I
192.168.1.100 10.0.2.15
```

---

## 4. 进程管理命令

### 4.1 进程的基本概念

**程序 vs 进程：**
- **程序**：存放在磁盘上的可执行文件（静态的代码）
- **进程**：程序在内存中的运行实例（动态的执行过程）

一个程序可以被运行多次，产生多个独立的进程。每个进程有唯一的 PID（Process ID）。

### 4.2 ps -- 查看进程状态

ps 有两套常用的选项风格：

**BSD 风格（ps aux，不用 -）：**
```bash
ps aux                   # 显示系统中所有进程的详细信息
ps aux | grep nginx      # 查找 nginx 相关进程
```

```
$ ps aux --sort=-%mem | head -5
USER    PID  %CPU %MEM    VSZ   RSS TTY  STAT START  TIME COMMAND
root      1   0.0  0.0  12556  4568 ?    Ss   10:00  0:02 /sbin/init
mysql  1234   0.5 12.3 1234567 246800 ?   Ssl  10:01  1:23 /usr/sbin/mysqld
```

各列含义：
| 列名 | 含义 |
|------|------|
| USER | 进程所有者 |
| PID | 进程 ID |
| %CPU | CPU 使用率 |
| %MEM | 内存使用率 |
| VSZ | 虚拟内存大小（KB） |
| RSS | 实际物理内存使用（KB） |
| TTY | 终端类型（? 表示没有控制终端，通常是后台服务） |
| STAT | 进程状态（S=睡眠, R=运行, Z=僵尸, D=不可中断睡眠） |
| START | 进程启动时间 |
| TIME | 累计 CPU 使用时间 |
| COMMAND | 命令行 |

**标准风格（ps -ef，Unix 风格）：**
```bash
ps -ef                   # 显示所有进程的完整信息
ps -ef | grep java       # 查找 java 进程
```

```
$ ps -ef | head -3
UID     PID  PPID  C STIME TTY      TIME CMD
root      1     0  0 10:00 ?    00:00:02 /sbin/init
root      2     0  0 10:00 ?    00:00:00 [kthreadd]
```

**常用 ps 组合：**
```bash
# 查看某个用户的进程
ps -u username

# 树状显示进程父子关系
ps auxf
ps -ejH

# 按 CPU 使用率降序排列
ps aux --sort=-%cpu

# 按内存使用率降序排列
ps aux --sort=-%mem
```

### 4.3 top -- 实时进程监控

```bash
top                      # 启动 top，实时显示进程信息（按 q 退出）
top -u username          # 只显示指定用户的进程
top -p PID               # 只监控指定 PID 的进程
```

**top 交互式按键（运行时按）：**

| 按键 | 功能 |
|------|------|
| `h` | 显示帮助 |
| `q` | 退出 top |
| `P` | 按 CPU 使用率排序（大写 P） |
| `M` | 按内存使用率排序（大写 M） |
| `N` | 按 PID 排序 |
| `T` | 按运行时间排序 |
| `k` | 杀死一个进程（会提示输入 PID 和信号） |
| `r` | 修改进程优先级（renice） |
| `1` | 切换显示每个 CPU 核心的使用情况 |
| `c` | 切换完整命令行显示 |
| `d` | 修改刷新间隔（秒） |
| `u` | 按用户过滤 |

```
$ top
top - 14:30:15 up 4:20,  2 users,  load average: 0.15, 0.10, 0.08
Tasks: 245 total,   1 running, 244 sleeping,   0 stopped,   0 zombie
%Cpu(s):  5.2 us,  2.1 sy,  0.0 ni, 92.5 id,  0.0 wa,  0.0 hi,  0.2 si,  0.0 st
MiB Mem:   7800.5 total,   3200.3 free,   2500.1 used,   2100.1 buff/cache
MiB Swap:  2048.0 total,   2000.5 free,     47.5 used.   4500.2 avail Mem

  PID USER   PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 1234 mysql  20   0  1.2G   240M   10M  S   3.5 12.3   1:23.45 mysqld
 5678 nginx  20   0  120M    50M   25M  S   0.3  2.5   0:05.67 nginx
```

**顶部信息区解读：**
- **load average**：系统负载，三个数字分别是 1/5/15 分钟平均值
- **Tasks**：进程总数及各状态数量
- **%Cpu(s)**：CPU 使用率分布（us=用户态, sy=内核态, id=空闲, wa=IO等待）
- **MiB Mem**：物理内存使用情况
- **MiB Swap**：交换空间使用情况

### 4.4 htop -- 更友好的进程监控

```bash
htop                     # 需要先安装：sudo apt install htop
```

htop 相比 top 的优势：
- 彩色界面，视觉效果更好
- 支持鼠标点击交互
- 可以垂直和水平滚动
- 更直观的 CPU/内存柱状图
- 支持树状进程视图（F5）

### 4.5 kill -- 终止进程

kill 命令的本质是向进程**发送信号**，不仅仅是"杀死"进程。

**常用信号：**
| 信号编号 | 信号名 | 含义 |
|----------|--------|------|
| `1` | SIGHUP | 挂起信号，常用于让守护进程重新加载配置 |
| `2` | SIGINT | 中断信号（相当于 Ctrl+C） |
| `9` | SIGKILL | 强制终止，进程无法捕获或忽略 |
| `15` | SIGTERM | 优雅终止信号（默认），进程可以捕获并做清理工作 |

```bash
kill PID                     # 发送 SIGTERM（15），优雅终止
kill -15 PID                 # 同上，显式指定信号
kill -9 PID                  # 发送 SIGKILL，强制终止（最后手段）
kill -l                      # 列出所有可用信号

# 先尝试优雅终止，不行再强制
kill PID                     # 先尝试
sleep 2                      # 等待2秒
kill -9 PID                  # 如果还没退出，强制终止
```

```bash
# 实用示例：让 nginx 重新加载配置
kill -1 $(cat /var/run/nginx.pid)    # 或
kill -HUP $(cat /var/run/nginx.pid)  # 等价写法
```

### 4.6 killall / pkill -- 按名称终止进程

```bash
killall 进程名           # 杀死所有匹配名称的进程
killall -9 进程名        # 强制杀死所有匹配的进程
killall -u 用户名 进程名  # 只杀死指定用户的进程

pkill 进程名             # 类似 killall，但支持正则匹配
pkill -f "pattern"        # 匹配完整命令行
```

```bash
# 示例
killall nginx             # 杀死所有 nginx 进程
pkill -f "python server.py"  # 杀死所有运行的特定 Python 脚本
```

### 4.7 后台运行与作业控制

#### 后台运行（&）

```bash
# 在命令末尾加 & 即可让命令在后台运行
sleep 100 &
# 输出：[1] 12345    （[1] 是作业号，12345 是 PID）

# 让已在运行的程序转入后台
# 先按 Ctrl+Z 暂停程序，然后
bg                       # 让它继续在后台运行
```

#### jobs -- 查看后台作业

```bash
jobs                     # 查看当前 Shell 的所有后台作业
jobs -l                  # 同时显示 PID
```

```
$ jobs
[1]   Running          sleep 100 &
[2]-  Stopped          vim file.txt
[3]+  Running          python server.py &
```

#### fg -- 将后台作业调回前台

```bash
fg                       # 将最近的后台作业调回前台
fg %1                    # 将作业号 1 调回前台
fg %sleep                # 通过命令名调回
```

#### 后台作业 vs Ctrl+Z
| 操作 | 效果 |
|------|------|
| `命令 &` | 程序直接在后台运行 |
| `Ctrl+Z` | 暂停前台程序，放入后台，状态为 Stopped |
| `bg` | 让 Stopped 的后台程序继续在后台运行 |
| `fg` | 把后台程序调回前台 |

### 4.8 nohup -- 退出终端后继续运行

默认情况下，关闭终端时 Shell 会向所有子进程发送 SIGHUP 信号，导致进程终止。nohup 可以忽略此信号。

```bash
nohup 命令 &                     # 后台运行，忽略 SIGHUP

# 示例：长时间运行的脚本
nohup python train.py &

# 输出默认重定向到 nohup.out
# 如果想指定输出文件
nohup python train.py > output.log 2>&1 &
```

> **注意**：即使使用了 nohup，如果进程依赖终端输入，它仍然会因无法读取输入而异常。确保程序不依赖 stdin。

### 4.9 systemctl -- 管理系统服务（systemd）

现代 Linux 发行版（Ubuntu 16.04+、CentOS 7+、Debian 8+）使用 systemd 管理服务。

```bash
# 服务状态管理
sudo systemctl start 服务名       # 启动服务
sudo systemctl stop 服务名        # 停止服务
sudo systemctl restart 服务名     # 重启服务
sudo systemctl reload 服务名      # 重新加载配置（不中断服务）
sudo systemctl status 服务名      # 查看服务状态
sudo systemctl enable 服务名      # 设置开机自启
sudo systemctl disable 服务名     # 取消开机自启
sudo systemctl is-enabled 服务名  # 检查是否已设置开机自启

# 查看系统状态
systemctl list-units --type=service  # 列出所有已加载的服务单元
systemctl list-unit-files            # 列出所有单元文件及启用状态
```

**示例：**
```bash
# 管理 nginx 服务
sudo systemctl start nginx
sudo systemctl status nginx
# 输出：
# nginx.service - A high performance web server
#    Loaded: loaded (/lib/systemd/system/nginx.service; enabled)
#    Active: active (running) since Wed 2026-07-22 12:00:00 CST; 2h ago

# 设置开机自启
sudo systemctl enable nginx

# 查看所有运行中的服务
systemctl list-units --type=service --state=running
```

---

## 5. 终端控制与其他常用命令

### 5.1 clear -- 清屏

```bash
clear                # 清除终端所有历史显示信息
# 快捷键：Ctrl + L  （效果相同）
```

### 5.2 date -- 查看/设置系统时间

```bash
date                 # 查看当前日期和时间
date "+%Y-%m-%d %H:%M:%S"    # 自定义输出格式
sudo date -s "2026-07-22 14:30:00"   # 设置系统时间（需要权限）
```

```
$ date
Wed Jul 22 14:30:15 CST 2026

$ date "+%Y-%m-%d %H:%M:%S"
2026-07-22 14:30:15

$ date "+%Y年%m月%d日 %A"
2026年07月22日 Wednesday
```

**常用格式化符号：**
| 符号 | 含义 | 示例 |
|------|------|------|
| `%Y` | 四位年份 | 2026 |
| `%m` | 月份（01-12） | 07 |
| `%d` | 日期（01-31） | 22 |
| `%H` | 小时（00-23） | 14 |
| `%M` | 分钟（00-59） | 30 |
| `%S` | 秒（00-59） | 15 |
| `%A` | 星期几（完整） | Wednesday |
| `%s` | Unix 时间戳 | 1753186215 |

### 5.3 cal -- 显示日历

```bash
cal                  # 显示当月日历
cal 2026             # 显示全年日历
cal 7 2026           # 显示 2026 年 7 月
```

```
$ cal
     July 2026
Su Mo Tu We Th Fr Sa
          1  2  3  4
 5  6  7  8  9 10 11
12 13 14 15 16 17 18
19 20 21 22 23 24 25
26 27 28 29 30 31
```

### 5.4 uname -- 查看系统信息

```bash
uname                # 显示内核名称（默认 Linux）
uname -a             # 显示所有系统信息
uname -r             # 显示内核版本号
uname -m             # 显示硬件架构（x86_64, aarch64等）
```

```
$ uname -a
Linux my-server 5.15.0-91-generic #101-Ubuntu SMP Tue Nov 14 13:30:00 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
```

### 5.5 lscpu -- 查看 CPU 信息

```bash
lscpu                # 显示 CPU 架构信息
```

```
$ lscpu
Architecture:        x86_64
CPU(s):              8
Thread(s) per core:  2
Core(s) per socket:  4
Socket(s):           1
Model name:          Intel(R) Core(TM) i7-8550U CPU @ 1.80GHz
CPU MHz:             1992.000
```

### 5.6 free -- 查看内存使用情况

```bash
free                 # 以 KB 为单位显示内存信息
free -h              # 以友好单位显示（推荐）
free -m              # 以 MB 为单位
free -g              # 以 GB 为单位
free -s 2            # 每隔 2 秒刷新一次
```

```
$ free -h
              total        used        free      shared  buff/cache   available
Mem:          7.8Gi       2.4Gi       3.1Gi       0.1Gi       2.2Gi       5.0Gi
Swap:         2.0Gi       47Mi        1.9Gi
```

**关键字段说明：**
| 字段 | 含义 |
|------|------|
| total | 总内存大小 |
| used | 已使用内存 |
| free | 完全未使用的内存 |
| shared | 共享内存（tmpfs 等） |
| buff/cache | 缓冲区和缓存占用的内存（可被回收） |
| available | 实际可用的内存（包括可回收的缓存） |

> 不要被 free 列的低值吓到。Linux 会把空闲内存用作磁盘缓存（buff/cache），当应用程序需要时会自动释放。真正需要关注的是 **available** 列。

---

## 6. 开关机命令

```bash
reboot               # 重启系统
poweroff             # 关机（不重启）
shutdown -h now      # 立即关机
shutdown -r now      # 立即重启
shutdown -h +10      # 10 分钟后关机
shutdown -c          # 取消已计划的关机
```

---
## 7. 三讲命令总表汇总

### P9 上 -- 基础入门

| 类别 | 命令 | 功能简介 |
|------|------|----------|
| 帮助 | `man` | 查看命令详细手册 |
| 帮助 | `--help` | 查看命令简要帮助 |
| 目录操作 | `ls` | 列出目录内容 |
| 目录操作 | `cd` | 切换工作目录 |
| 目录操作 | `pwd` | 显示当前工作目录路径 |
| 目录操作 | `mkdir` | 创建目录 |
| 目录操作 | `rmdir` | 删除空目录 |
| 目录操作 | `mv` | 移动/重命名文件或目录 |
| 文件操作 | `touch` | 创建空文件或更新时间戳 |
| 文件操作 | `cat` | 显示文件内容 |
| 文件操作 | `echo` | 输出字符串或变量值 |
| 文件操作 | `wc` | 统计文件行数、单词数、字节数 |
| 文件操作 | `rm` | 删除文件或目录 |

### P10 中 -- 进阶操作

| 类别 | 命令 | 功能简介 |
|------|------|----------|
| 链接 | `ln` | 创建硬链接 |
| 链接 | `ln -s` | 创建符号链接（软链接） |
| 复制/打包 | `cp` | 复制文件或目录 |
| 复制/打包 | `tar` | 打包/解包归档文件 |
| 查找/搜索 | `find` | 按条件搜索文件 |
| 查找/搜索 | `grep` | 搜索文件内容（正则匹配） |
| 用户管理 | `sudo` | 以超级用户权限执行命令 |
| 用户管理 | `su` | 切换用户 |
| 用户管理 | `adduser` | 交互式添加用户（Debian系） |
| 用户管理 | `useradd` | 添加用户（通用） |
| 用户管理 | `usermod` | 修改用户属性 |
| 用户管理 | `deluser` / `userdel` | 删除用户 |
| 用户管理 | `passwd` | 设置/修改用户密码 |
| 用户管理 | `addgroup` | 添加用户组（Debian系） |
| 用户管理 | `delgroup` | 删除用户组（Debian系） |

### P11 下 -- 系统管理

| 类别 | 命令 | 功能简介 |
|------|------|----------|
| 权限修改 | `chmod` | 修改文件/目录权限 |
| 权限修改 | `chown` | 修改文件/目录所有者 |
| 权限修改 | `chgrp` | 修改文件/目录所属组 |
| 权限修改 | `umask` | 设置默认权限掩码 |
| 磁盘管理 | `df -h` | 查看文件系统磁盘使用 |
| 磁盘管理 | `du -h` / `du -hs` | 统计文件/目录磁盘占用 |
| 磁盘管理 | `mount` / `umount` | 挂载/卸载文件系统 |
| 磁盘管理 | `lsblk` | 查看块设备树状结构 |
| 磁盘管理 | `blkid` | 查看设备 UUID 和类型 |
| 磁盘管理 | `fdisk -l` | 查看分区表 |
| 网络操作 | `ping` | 检测网络连通性 |
| 网络操作 | `ifconfig` | 网卡配置（传统工具） |
| 网络操作 | `ip` | 网络管理（现代工具） |
| 网络操作 | `ss` | 查看网络连接 |
| 网络操作 | `wget` / `curl` | 下载文件 / 数据传输 |
| 网络操作 | `hostname` | 查看/设置主机名 |
| 进程管理 | `ps` | 查看进程状态 |
| 进程管理 | `top` / `htop` | 实时进程监控 |
| 进程管理 | `kill` | 按 PID 终止进程 |
| 进程管理 | `killall` / `pkill` | 按名称终止进程 |
| 进程管理 | `jobs` / `fg` / `bg` | 管理后台作业 |
| 进程管理 | `nohup` | 退出终端后继续运行 |
| 进程管理 | `systemctl` | 管理系统服务 |
| 系统信息 | `uname -a` | 查看系统内核信息 |
| 系统信息 | `lscpu` | 查看 CPU 信息 |
| 系统信息 | `free -h` | 查看内存使用 |
| 系统信息 | `date` | 查看/设置日期时间 |
| 系统信息 | `cal` | 显示日历 |
| 终端控制 | `clear` | 清屏（Ctrl+L） |
| 开关机 | `reboot` | 重启系统 |
| 开关机 | `poweroff` | 关机 |
| 开关机 | `shutdown` | 计划关机/重启 |

---

*笔记生成日期：2026-07-22 | 转写：Whisper tiny | 优化：补充详解与实操示例*
