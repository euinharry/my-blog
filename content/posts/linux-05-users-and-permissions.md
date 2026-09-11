---
title: "第7讲：用户管理与文件权限"
date: 2026-09-11T09:22:00+08:00
draft: false
description: "用户组是由一个或多个用户组合构成的集合。引入用户组后，管理员可以按组为单位分配权限，而不需要逐个用户设置，大幅简化了权限管理。"
series: ["Linux 入门"]
series_order: 5
categories: ["技术笔记"]
tags: ["Linux", "用户与权限"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P8)
> **标题**：第7讲 -- 用户管理与文件权限
> **时长**：26分36秒
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. 用户与用户组基础概念

### 1.1 用户的三种类型

| 类型 | 说明 |
|------|------|
| **root（超级管理员）** | 拥有系统最高权限，UID固定为0 |
| **系统用户** | 系统自身创建的维护用户，用于运行系统服务（UID范围：1-999） |
| **普通用户** | 使用者创建的用户，用于日常操作（UID >= 1000） |

### 1.2 用户组的概念

用户组是由一个或多个用户组合构成的集合。引入用户组后，管理员可以按组为单位分配权限，而不需要逐个用户设置，大幅简化了权限管理。

### 1.3 用户与用户组的四种关系

| 关系 | 说明 | 示例 |
|------|------|------|
| **1对1** | 一个用户只归属一个用户组 | 独立用户 |
| **1对多** | 一个用户归属多个用户组 | 跨部门成员 |
| **多对1** | 多个用户归属同一个用户组 | 项目团队 |
| **多对多** | 多个用户归属多个用户组 | 复杂组织架构 |

### 1.4 主组与附加组

每个用户有且仅有一个**主组（Primary Group）**，即用户在创建文件时默认归属的组。同时，用户可以属于多个**附加组（Supplementary Groups）**，从而获得这些组的访问权限。

| 概念 | 说明 |
|------|------|
| **主组** | /etc/passwd 第4字段指定的 GID，用户登录后默认生效的组 |
| **附加组** | /etc/group 第4字段列出的组成员，用户可额外获得的组权限 |

---

## 2. 为什么需要用户管理？

### 2.1 Linux的特性
- Linux是**多用户多任务**操作系统
- 多个用户可以同时使用系统，执行不同任务，互不影响

### 2.2 个人开发 vs 团队开发

| 场景 | 做法 | 风险 |
|------|------|------|
| **个人开发** | 直接使用root权限 | 问题不大 |
| **团队开发** | 每个人都给root权限 | **极不安全** |

### 2.3 正确的做法

不同部门只能访问自己部门的资料文件：
- **销售部门** -> 只能访问销售资料
- **研发部门** -> 只能访问研发资料

**三大好处**：
1. 隔离部门职责，保护核心文件安全性
2. 防止误操作（如新员工误删整个文件系统）
3. 对服务器安全性要求越高的公司，越需要建立用户管理制度

---

## 3. UID 和 GID

| 概念 | 全称 | 说明 |
|------|------|------|
| **UID** | User ID | 用户的数字标识，系统通过UID而不是用户名来区分用户 |
| **GID** | Group ID | 用户组的数字标识，用于区分不同的组 |

**关键**：Linux系统不认用户名，只认UID！

### 3.1 查看用户的 UID 和 GID

使用 `id` 命令可以查看当前用户或指定用户的 UID、GID 以及所属的全部组：

```bash
# 查看当前用户信息
id

# 输出示例：
# uid=1000(fire) gid=1000(fire) groups=1000(fire),27(sudo),1001(developers)

# 查看指定用户信息
id username
```

`groups` 命令可以快速列出用户所属的全部组：

```bash
# 查看当前用户所属组
groups

# 查看指定用户所属组
groups username
```

---

## 4. 三个关键的用户管理文件

> 所有文件均位于 `/etc/` 目录下

| 文件 | 用途 |
|------|------|
| `/etc/passwd` | 用户账户信息 |
| `/etc/shadow` | 加密密码信息 |
| `/etc/group` | 用户组信息 |

### 4.1 登录验证流程

当用户登录时，系统执行以下步骤：

```
输入用户名+密码 -> 1. 去 /etc/passwd 找用户名对应的UID
                  -> 2. 去 /etc/group 找用户对应的GID
                  -> 3. 验证通过 -> 去 /etc/shadow 核对加密密码
                  -> 4. 密码匹配 -> 登录成功
```

### 4.2 /etc/passwd 文件详解

以用户 `fire` 为例，一行记录包含7个字段（以冒号分隔）：

```
fire:x:1000:1000:fire,,,:/home/fire:/bin/bash
│    │  │    │   │       │         │
①用户名 ②密码③UID ④GID ⑤注释 ⑥家目录  ⑦登录Shell
```

| 字段 | 内容 | 说明 |
|------|------|------|
| ① 用户名 | fire | 登录时使用的名称 |
| ② 密码 | x | 早期直接存密码，不安全。现用x表示密码在shadow文件中 |
| ③ UID | 1000 | 用户ID（0=root，1-999=系统，1000+=普通） |
| ④ GID | 1000 | 用户组ID（默认与UID相同） |
| ⑤ 注释 | fire,,, | 用户信息说明字段（GECOS字段：全名,办公室,电话,其他） |
| ⑥ 家目录 | /home/fire | 登录后默认进入的目录 |
| ⑦ 登录Shell | /bin/bash | 指定命令解析器 |

**关键技巧**：
- root的UID固定为0 -> 把普通用户的UID改为0，它就变成了root用户
- 如果把Shell改为 `/sbin/nologin` -> 该用户将无法登录终端（常用于系统服务账号）
- 常用的Shell类型：`/bin/bash`、`/bin/sh`、`/bin/zsh`、`/sbin/nologin`、`/bin/false`

### 4.3 /etc/shadow 文件详解（完整字段）

`/etc/shadow` 每一行包含9个字段，以冒号分隔：

```
fire:$6$xxxx...:18765:0:99999:7:14::
│        │         │    │ │     │ │  │
①      ②          ③    ④ ⑤     ⑥ ⑦  ⑧⑨
```

| 字段 | 名称 | 说明 |
|------|------|------|
| ① | 用户名 | 对应的用户名称 |
| ② | 加密密码 | 使用hash算法加密后的密码（`$6$` 表示 SHA-512 加密）。`!` 或 `*` 表示账号被锁定，无法用密码登录 |
| ③ | 最后修改时间 | 从1970年1月1日到最近一次密码修改的总天数 |
| ④ | 最小修改间隔 | 两次修改密码之间必须经过的最小天数（0=无限制） |
| ⑤ | 最大有效期 | 密码有效期的最大天数（99999=约274年，即无限制） |
| ⑥ | 警告天数 | 密码过期前多少天开始警告用户 |
| ⑦ | 宽限天数 | 密码过期后仍可登录的天数（超过则账号锁定） |
| ⑧ | 过期日期 | 从1970-01-01起的天数，超过后账号失效（空=永不过期） |
| ⑨ | 保留字段 | 预留扩展，当前未使用 |

**加密算法识别（字段②前缀）**：
| 前缀 | 算法 | 安全等级 |
|------|------|----------|
| `$1$` | MD5 | 低（已不推荐） |
| `$5$` | SHA-256 | 中 |
| `$6$` | SHA-512 | 高（默认） |
| `$y$` | yescrypt | 现代高安全 |

### 4.4 /etc/group 文件详解

```
fire:x:1000:                 # 组内无其他成员（组名=用户名时不显示）
sudo:x:27:fire              # fire用户属于sudo组
│    │  │    │
①组名 ②密码 ③GID ④组成员列表
```

| 字段 | 说明 |
|------|------|
| ① 组名 | 用户组的名称 |
| ② 密码 | x表示密码存储在 `/etc/gshadow` 文件中 |
| ③ GID | 用户组的数字标识 |
| ④ 组成员 | 该组下的所有用户列表（逗号分隔） |

**注意**：一个用户可以属于多个组（如fire既属于fire组，也属于sudo组）

---

## 5. 用户管理命令详解

### 5.1 useradd -- 创建用户

**基本语法**：

```bash
useradd [选项] 用户名
```

**常用选项**：

| 选项 | 说明 | 示例 |
|------|------|------|
| `-m` | 自动创建家目录（大多数发行版默认开启） | `useradd -m newuser` |
| `-d 目录` | 指定家目录路径 | `useradd -d /data/home/newuser newuser` |
| `-s Shell路径` | 指定登录Shell | `useradd -s /bin/bash newuser` |
| `-u UID` | 指定用户UID | `useradd -u 1500 newuser` |
| `-g 组名/GID` | 指定主组（组必须已存在） | `useradd -g developers newuser` |
| `-G 组1,组2` | 指定附加组（多个组逗号分隔） | `useradd -G sudo,docker newuser` |
| `-c 注释` | 添加用户描述信息 | `useradd -c "张三,研发部" newuser` |
| `-e 日期` | 指定账号过期日期（YYYY-MM-DD） | `useradd -e 2026-12-31 newuser` |
| `-r` | 创建系统用户（UID < 1000） | `useradd -r myservice` |
| `-M` | 不创建家目录 | `useradd -M tempuser` |

**实操示例**：

```bash
# 创建开发组
groupadd developers

# 创建普通开发用户（带家目录、bash shell、加入开发组和sudo组）
useradd -m -s /bin/bash -g developers -G sudo -c "李四,研发部" lisi

# 创建无家目录的临时用户
useradd -M -s /sbin/nologin -e 2026-08-01 temp
```

### 5.2 usermod -- 修改用户属性

**基本语法**：

```bash
usermod [选项] 用户名
```

**常用选项**：

| 选项 | 说明 | 示例 |
|------|------|------|
| `-l 新用户名` | 修改用户名 | `usermod -l newname oldname` |
| `-u UID` | 修改UID | `usermod -u 1500 username` |
| `-g 组名` | 修改主组 | `usermod -g developers username` |
| `-G 组1,组2` | 修改附加组（覆盖原有附加组） | `usermod -G sudo,docker username` |
| `-aG 组名` | 追加附加组（不覆盖，与-G配合使用） | `usermod -aG docker username` |
| `-d 目录` | 修改家目录 | `usermod -d /new/home username` |
| `-m` | 移动家目录内容（与-d配合） | `usermod -md /new/home username` |
| `-s Shell` | 修改登录Shell | `usermod -s /bin/zsh username` |
| `-L` | 锁定用户密码（禁止登录） | `usermod -L username` |
| `-U` | 解锁用户密码 | `usermod -U username` |
| `-c 注释` | 修改用户描述 | `usermod -c "新描述" username` |
| `-e 日期` | 修改过期日期 | `usermod -e 2027-01-01 username` |

**重要提醒**：使用 `-G` 会**覆盖**原有的附加组列表。如果要追加而非覆盖，必须使用 `-aG`：

```bash
# 错误：会覆盖掉原有的附加组！
usermod -G docker username

# 正确：追加docker到已有的附加组列表
usermod -aG docker username
```

### 5.3 userdel -- 删除用户

**基本语法**：

```bash
userdel [选项] 用户名
```

| 选项 | 说明 |
|------|------|
| `-r` | 同时删除用户的家目录和邮件目录 |
| （无选项） | 仅删除用户账户，保留家目录 |

```bash
# 仅删除用户账户，保留家目录数据
userdel username

# 删除用户并清理家目录（推荐：彻底清理）
userdel -r username
```

### 5.4 passwd -- 密码管理

**基本语法**：

```bash
passwd [选项] [用户名]
```

**常用操作**：

| 命令 | 说明 |
|------|------|
| `passwd` | 修改当前用户密码（交互式） |
| `passwd username` | root修改指定用户的密码 |
| `passwd -l username` | 锁定用户密码（在shadow密码前加`!`，禁止登录） |
| `passwd -u username` | 解锁用户密码 |
| `passwd -d username` | 删除用户密码（允许空密码登录，极不安全） |
| `passwd -e username` | 强制用户下次登录时修改密码 |
| `passwd -S username` | 查看用户密码状态 |
| `passwd -n 天数 username` | 设置密码最小修改间隔 |
| `passwd -x 天数 username` | 设置密码最大有效期 |
| `passwd -w 天数 username` | 设置密码过期前警告天数 |
| `passwd -i 天数 username` | 设置密码过期后宽限天数 |

```bash
# 查看用户密码状态
passwd -S fire
# 输出示例：fire P 07/20/2026 0 99999 7 14
#          │     │    │          │   │   │   └-- 宽限天数
#          │     │    │          │   │   └------ 警告天数
#          │     │    │          │   └---------- 最大有效期
#          │     │    │          └-------------- 最小修改间隔
#          │     │    └------------------------- 最后修改日期
#          │     └------------------------------ P=有密码 L=锁定 NP=无密码
#          └------------------------------------ 用户名

# 锁定用户
passwd -l fire

# 解锁用户
passwd -u fire

# 强制用户下次登录修改密码
passwd -e fire
```

---

## 6. 文件权限系统

### 6.1 权限的三种类型

| 权限类型 | 说明 |
|----------|------|
| **所有者权限（Owner）** | 文件创建者拥有的权限 |
| **组权限（Group）** | 与所有者同组的其他用户的权限 |
| **其他用户权限（Other）** | 既不是所有者也不属于同组的用户权限 |

### 6.2 权限的三位表示

每组权限用3个bit位表示：

| 位 | 字母 | 含义 | 数字值 |
|----|------|------|--------|
| 第1位 | r（read） | 读取权限 | 4 |
| 第2位 | w（write） | 写入权限 | 2 |
| 第3位 | x（execute） | 执行权限 | 1 |

没有权限时用 `-` 表示。

### 6.3 权限位图解

以 `/etc/cron.d/updatedb` 为例：

```
- rwx r-x r--
│ │   │   └── Other（其他用户）：r-- 只读
│ │   └────── Group（同组用户）：r-x 读+执行
│ └────────── Owner（文件所有者）：rwx 读+写+执行
└──────────── 文件类型（-=普通文件，d=目录，l=链接...）
```

**逐位分析**：

| 位序 | 所属 | 值 | 含义 |
|------|------|----|------|
| 1-3 | Owner | rwx | 可读、可写、可执行 |
| 4-6 | Group | r-x | 可读、可执行（不可写） |
| 7-9 | Other | r-- | 仅可读 |

### 6.4 文件类型标识

`ls -l` 输出的第一个字符表示文件类型：

| 字符 | 类型 | 说明 |
|------|------|------|
| `-` | 普通文件 | 文本、二进制、数据文件等 |
| `d` | 目录 | 文件夹 |
| `l` | 符号链接 | 软链接（快捷方式） |
| `b` | 块设备 | 硬盘、U盘等块设备文件 |
| `c` | 字符设备 | 键盘、鼠标等字符设备文件 |
| `p` | 管道文件 | FIFO命名管道 |
| `s` | 套接字 | socket文件（网络通信） |

### 6.5 权限数字表示法（八进制）

```
rwx = 4+2+1 = 7
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4
-w- = 0+2+0 = 2
--x = 0+0+1 = 1
--- = 0+0+0 = 0

所以 rwxr-xr-- = 754
```

### 6.6 权限与目录的特殊关系

**目录的 r、w、x 权限含义不同于普通文件：**

| 权限 | 对文件的作用 | 对目录的作用 |
|------|-------------|-------------|
| `r` | 读取文件内容（cat、less） | 列出目录内容（ls） |
| `w` | 修改文件内容 | 在目录中创建/删除文件（touch、rm、mv） |
| `x` | 执行文件（程序/脚本） | **进入目录**（cd），访问目录下文件 |

**典型场景**：

```bash
# 目录只有r权限：可以ls但无法cd进去
dr-------- 2 user group 4096 Jul 22 10:00 mydir

# 目录只有x权限：可以cd进去但无法ls查看内容
d--x------ 2 user group 4096 Jul 22 10:00 mydir

# 目录正常权限：rwx组合
drwx------ 2 user group 4096 Jul 22 10:00 mydir

# 公共目录（任何人都可以进入和查看，但不能删除他人的文件 -> 需要Sticky Bit）
drwxrwxrwt 2 user group 4096 Jul 22 10:00 shared
```

---

## 7. chmod -- 修改文件权限

### 7.1 数字模式（八进制）

```bash
chmod 754 filename

# 7 = Owner:   rwx (4+2+1)
# 5 = Group:   r-x (4+0+1)
# 4 = Other:   r-- (4+0+0)
```

**常用数字权限组合**：

| 数字 | 权限 | 适用场景 |
|------|------|----------|
| `777` | rwxrwxrwx | 所有用户可读写执行（不安全） |
| `755` | rwxr-xr-x | 目录/可执行文件（Owner全权，其他人读+执行） |
| `644` | rw-r--r-- | 普通文件（Owner可写，其他人只读） |
| `700` | rwx------ | 私有脚本/目录（仅Owner可访问） |
| `600` | rw------- | 私密文件（仅Owner可读写，如SSH私钥） |
| `400` | r-------- | 只读文件（受保护配置文件） |

### 7.2 符号模式

符号模式使用 `u`（user/所有者）、`g`（group/组）、`o`（other/其他）、`a`（all/全部）配合 `+`、`-`、`=` 来修改权限：

**操作符**：

| 操作符 | 说明 |
|--------|------|
| `+` | 添加权限 |
| `-` | 移除权限 |
| `=` | 设置为指定权限（覆盖） |

**作用对象**：

| 符号 | 说明 |
|------|------|
| `u` | 文件所有者（User） |
| `g` | 所属组（Group） |
| `o` | 其他用户（Other） |
| `a` | 所有三类用户（All，等同于ugo） |

**实操示例**：

```bash
# 给所有者添加执行权限
chmod u+x script.sh          # 结果：-rwxr--r--

# 给组和其他人添加读权限
chmod go+r file.txt          # 结果：-rw-r--r--

# 移除组的写权限
chmod g-w file.txt           # 结果：-rw-r--r--

# 取消其他人的全部权限
chmod o-rwx secret.txt       # 结果：-rw-r-----

# 同时设置多组权限
chmod u+rwx,g+rx,o= file.sh # 结果：-rwxr-x---

# 给所有人添加执行权限
chmod a+x script.sh           # 等同于 chmod ugo+x script.sh

# 批量设置为 755
chmod u=rwx,g=rx,o=rx file   # 结果：-rwxr-xr-x

# 递归修改目录下所有文件权限
chmod -R 755 /path/to/dir/
```

---

## 8. chown -- 修改文件所有者

### 8.1 基本语法

```bash
chown [选项] [所有者][:组] 文件...
```

### 8.2 常用用法

```bash
# 修改文件所有者
chown alice file.txt

# 同时修改所有者和组（user:group 或 user.group）
chown alice:developers file.txt
chown alice.developers file.txt

# 只修改所属组（保留原所有者）
chown :developers file.txt
chown .developers file.txt

# 递归修改目录及所有子文件的所有者
chown -R alice:developers /home/alice/project/

# 查看修改前后对比
ls -l file.txt
# -rw-r--r-- 1 root root  1024 Jul 22 10:00 file.txt  # 修改前
chown alice:developers file.txt
ls -l file.txt
# -rw-r--r-- 1 alice developers 1024 Jul 22 10:00 file.txt  # 修改后
```

**注意**：修改文件所有者通常需要 root 权限。普通用户不能将文件所有权转给其他用户。

### 8.3 chgrp -- 单独修改所属组

```bash
# 修改文件所属组
chgrp developers file.txt

# 递归修改
chgrp -R developers /path/to/dir/
```

---

## 9. 特殊权限：SUID、SGID、Sticky Bit

除了标准的 rwx 权限外，Linux 还有三种特殊权限位，它们为特定场景提供额外的安全控制。

### 9.1 SUID（Set User ID）-- 数字值：4

当一个可执行文件设置了 SUID 位后，任何用户执行该文件时，**进程的有效用户ID会临时变为文件所有者的UID**，从而以文件所有者的权限运行。

```bash
# 设置 SUID
chmod u+s /usr/bin/passwd
# 或使用数字模式（4在前）
chmod 4755 /usr/bin/passwd

# 权限显示：s 替代了 Owner 的 x 位
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root 68208 Jul 22 10:00 /usr/bin/passwd
#   ↑ 这里的 s 表示 SUID + 执行权限
#   如果显示为 S（大写），表示 SUID 但没有执行权限
```

**典型用例**：`/usr/bin/passwd` 需要修改 `/etc/shadow`（只有root可写），普通用户通过 SUID 临时获得root权限来完成密码修改。

**安全警告**：SUID 是安全敏感功能，不要随意给自定义脚本设置 SUID，否则可能被利用进行提权攻击。

### 9.2 SGID（Set Group ID）-- 数字值：2

**对可执行文件**：执行时进程的有效组ID变为文件所属组。
**对目录**：在目录中创建的新文件/子目录，其所属组自动继承该目录的组（而非创建者的主组），这对于共享工作目录非常有用。

```bash
# 设置 SGID
chmod g+s /shared/project/
# 或使用数字模式（2在前）
chmod 2775 /shared/project/

# 权限显示：s 替代了 Group 的 x 位
ls -ld /shared/project/
# drwxrwsr-x 2 root developers 4096 Jul 22 10:00 /shared/project/
#       ↑ 这里的 s 表示 SGID + 执行权限
```

**典型用例**：团队共享目录。在 `/shared/project/` 目录设置了 SGID 后，无论哪个组成员创建的文件，都会自动归属于 `developers` 组，确保组内所有成员都能访问。

### 9.3 Sticky Bit（粘滞位）-- 数字值：1

Sticky Bit 只对目录有意义。设置了 Sticky Bit 的目录，**只有文件所有者、目录所有者或 root 才能删除或重命名目录中的文件**，即使其他用户对该目录有写权限也不行。

```bash
# 设置 Sticky Bit
chmod +t /tmp
# 或使用数字模式（1在前）
chmod 1777 /tmp

# 权限显示：t 替代了 Other 的 x 位
ls -ld /tmp
# drwxrwxrwt 12 root root 4096 Jul 22 10:00 /tmp
#         ↑ 这里的 t 表示 Sticky Bit + 执行权限
#         如果显示为 T（大写），表示 Sticky Bit 但没有执行权限
```

**典型用例**：`/tmp` 目录。所有人都有写权限（777），但有了 Sticky Bit 后，用户只能删除自己创建的文件，无法删除其他用户的临时文件。

### 9.4 特殊权限组合总结

| 数字组合 | 含义 | 示例场景 |
|----------|------|----------|
| `4755` | SUID + rwxr-xr-x | 需要root权限的可执行文件 |
| `2755` | SGID + rwxr-xr-x | 需要特定组权限的可执行文件 |
| `2770` | SGID + rwxrwx--- | 团队共享目录 |
| `1777` | Sticky + rwxrwxrwx | 公共临时目录（/tmp） |
| `6755` | SUID+SGID + rwxr-xr-x | 同时需要两种特殊权限 |

---

## 10. umask -- 默认权限掩码

### 10.1 概念

`umask`（User Mask）决定了用户创建新文件或目录时的**默认权限**。它不是"赋予"权限，而是"屏蔽"权限：从最大权限中减去 umask 值，得到默认权限。

### 10.2 计算公式

```
新文件的默认权限 = 0666 - umask
新目录的默认权限 = 0777 - umask
```

注意：文件默认没有执行权限，所以基础值是666而非777。

### 10.3 常用 umask 值

| umask | 文件默认权限 | 目录默认权限 | 安全性 |
|-------|-------------|-------------|--------|
| `022` | 644 (rw-r--r--) | 755 (rwxr-xr-x) | 宽松（默认） |
| `027` | 640 (rw-r-----) | 750 (rwxr-x---) | 中等 |
| `077` | 600 (rw-------) | 700 (rwx------) | 严格（最安全） |
| `002` | 664 (rw-rw-r--) | 775 (rwxrwxr-x) | 宽松（适合共享组） |

### 10.4 实操

```bash
# 查看当前 umask
umask
# 输出：0022（第一位0表示八进制数，有效值为022）

# 临时修改 umask（仅当前shell生效）
umask 027

# 永久修改：写入 ~/.bashrc 或 /etc/profile
echo "umask 027" >> ~/.bashrc

# 验证效果
umask 077
touch test_file
mkdir test_dir
ls -l
# -rw------- 1 user group 0 Jul 22 10:00 test_file    # 666-077=600
# drwx------ 2 user group 4096 Jul 22 10:00 test_dir  # 777-077=700
```

---

## 11. ACL（访问控制列表）

### 11.1 为什么需要ACL？

传统的 rwx 权限模型只能为一个所有者、一个组和其他用户设置权限。当需要给**多个特定用户或组**设置不同权限时，传统的9位权限就不够用了。

**ACL（Access Control List）** 允许为单个文件/目录设置更细粒度的权限，可以给任意数量的用户和组分别指定不同的访问权限。

### 11.2 前置条件

```bash
# 确认文件系统支持ACL（多数现代Linux发行版默认支持）
# 检查 /etc/fstab 中是否有 acl 挂载选项
grep acl /etc/fstab

# 如果未启用，可以在挂载时添加acl选项
# mount -o remount,acl /dev/sda1 /

# 安装ACL工具（如果未安装）
# Debian/Ubuntu:
sudo apt install acl

# RHEL/CentOS:
sudo yum install acl
```

### 11.3 getfacl -- 查看ACL

```bash
# 查看文件的ACL信息
getfacl file.txt

# 输出示例：
# # file: file.txt
# # owner: alice
# # group: developers
# user::rw-          # 所有者权限（空用户名表示文件所有者）
# user:bob:r--       # bob用户的额外读权限
# group::r--         # 所属组权限
# group:managers:rw- # managers组的额外读写权限
# mask::rw-          # 权限掩码（ACL的最大有效权限）
# other::---         # 其他人权限
```

### 11.4 setfacl -- 设置ACL

**基本语法**：

```bash
setfacl [选项] 规则 文件名
```

**规则格式**：

```
u:用户名:权限    # 为用户设置权限
g:组名:权限      # 为组设置权限
m:权限          # 修改mask（掩码）
o:权限          # 修改other权限
```

**常用选项**：

| 选项 | 说明 |
|------|------|
| `-m` | 修改或添加ACL规则 |
| `-x` | 删除指定ACL规则 |
| `-b` | 删除所有ACL规则（恢复为基本权限） |
| `-R` | 递归应用到目录及所有子项 |
| `-d` | 设置默认ACL（仅对目录有意义，新创建的文件/目录会继承） |

**实操示例**：

```bash
# 给用户 bob 添加读写权限
setfacl -m u:bob:rw- file.txt

# 给 managers 组添加读和执行权限
setfacl -m g:managers:r-x file.txt

# 给多个用户设置权限
setfacl -m u:alice:rwx,u:bob:r--,u:charlie:--- file.txt

# 删除用户 bob 的ACL规则
setfacl -x u:bob file.txt

# 删除所有ACL规则
setfacl -b file.txt

# 设置目录的默认ACL（目录内新创建的文件自动继承）
setfacl -m d:u:bob:rwx /shared/project/
setfacl -m d:g:developers:rwx /shared/project/

# 递归设置
setfacl -R -m u:bob:rwx /shared/project/
```

**ls -l 中ACL标识**：设置了ACL的文件/目录，权限位末尾会显示 `+` 号：

```bash
ls -l file.txt
# -rw-rwxr--+ 1 alice developers 1024 Jul 22 10:00 file.txt
#          ↑ 这个 + 号表示该文件设置了ACL
```

---

## 12. 完整实操流程：用户管理 + 权限配置

以下是一个模拟团队开发环境的完整操作流程：

### 场景描述

假设计算机学院有3个部门：研发部、测试部、运维部。每个部门有独立的共享目录，部门成员只能访问自己部门的文件。

### 操作步骤

```bash
# ===== 第一步：以root身份操作 =====
sudo -i

# 创建部门组
groupadd dev
groupadd test
groupadd ops

# 创建共享目录
mkdir -p /shared/{dev,test,ops}

# 设置目录所有权和SGID（部门内新建文件自动继承部门组）
chown :dev /shared/dev
chmod 2770 /shared/dev              # SGID + 组内可读写，其他人无权限

chown :test /shared/test
chmod 2770 /shared/test

chown :ops /shared/ops
chmod 2770 /shared/ops

# ===== 第二步：创建部门员工 =====

# 研发部员工：张三（主组dev，附加组无）
useradd -m -s /bin/bash -g dev -c "张三,研发部" zhangsan
echo "zhangsan:Pass1234" | chpasswd

# 研发部员工：李四（主组dev）
useradd -m -s /bin/bash -g dev -c "李四,研发部" lisi
echo "lisi:Pass1234" | chpasswd

# 测试部员工：王五（主组test）
useradd -m -s /bin/bash -g test -c "王五,测试部" wangwu
echo "wangwu:Pass1234" | chpasswd

# 运维部员工：赵六（主组ops）
useradd -m -s /bin/bash -g ops -c "赵六,运维部" zhaoliu
echo "zhaoliu:Pass1234" | chpasswd

# 部门经理：需要跨部门访问（主组dev，附加组test和ops）
useradd -m -s /bin/bash -g dev -G test,ops -c "部门经理" manager
echo "manager:Pass1234" | chpasswd

# ===== 第三步：设置密码策略 =====

# 设置密码90天过期
passwd -x 90 zhangsan
passwd -x 90 lisi
passwd -x 90 wangwu
passwd -x 90 zhaoliu

# 强制所有用户下次登录修改密码
passwd -e zhangsan
passwd -e lisi
passwd -e wangwu
passwd -e zhaoliu

# 设置umask确保新建文件安全
echo "umask 027" >> /home/zhangsan/.bashrc
echo "umask 027" >> /home/lisi/.bashrc
echo "umask 027" >> /home/wangwu/.bashrc
echo "umask 027" >> /home/zhaoliu/.bashrc

# ===== 第四步：测试权限 =====

# 切换到张三
su - zhangsan

# 在研发目录创建文件
cd /shared/dev
touch dev_report.txt
echo "研发进度报告" > dev_report.txt

# 验证权限
ls -l dev_report.txt
# -rw-r----- 1 zhangsan dev 22 Jul 22 10:00 dev_report.txt
# 说明：umask 027 生效，文件权限为 640

# 尝试访问测试部目录（应该被拒绝）
ls /shared/test/
# ls: cannot open directory '/shared/test/': Permission denied  # 正确：无权访问

# 退出张三，切换到李四
exit
su - lisi

# 李四可以访问研发部共享目录
cd /shared/dev
cat dev_report.txt
# 研发进度报告    # 成功：同组有读权限

# 退出李四，切换到经理
exit
su - manager

# 经理可以访问所有部门目录（因为附加组包含了test和ops）
ls /shared/dev/     # 成功
ls /shared/test/    # 成功
ls /shared/ops/     # 成功

# ===== 第五步：使用ACL实现更精细的控制 =====

# 在研发目录上，给经理添加完全控制权
sudo setfacl -m u:manager:rwx /shared/dev/

# 在测试目录上，给经理添加只读权限
sudo setfacl -m u:manager:r-x /shared/test/

# 查看ACL设置
getfacl /shared/dev/

# ===== 第六步：清理（删除测试用户） =====

# 删除所有测试用户
userdel -r zhangsan
userdel -r lisi
userdel -r wangwu
userdel -r zhaoliu
userdel -r manager

# 删除测试组
groupdel dev
groupdel test
groupdel ops

# 删除测试目录
rm -rf /shared/{dev,test,ops}
```

---

## 13. 常用诊断命令速查

```bash
# 查看当前登录用户
whoami

# 查看当前用户的UID、GID和所有组
id
id username

# 查看用户所属组
groups
groups username

# 查看所有用户列表
cat /etc/passwd
getent passwd

# 查看所有组列表
cat /etc/group
getent group

# 查看当前有哪些用户登录
who
w

# 查看最近登录记录
last
last username

# 查看文件详细信息（含权限、所有者、组）
ls -l filename
stat filename          # 更详细的信息

# 查看目录权限
ls -ld directory/

# 查看文件的ACL
getfacl filename

# 查看当前umask
umask
```

---

## 14. 总结

### 核心概念关系图

```
用户管理 ─── 基础 ───→ 文件权限
  │                       │
  ├── UID（用户ID）       ├── Owner权限（文件所有者）
  ├── GID（组ID）         ├── Group权限（同组成员）
  ├── /etc/passwd         ├── Other权限（其他用户）
  ├── /etc/shadow         ├── SUID/SGID/Sticky Bit（特殊权限）
  ├── /etc/group          ├── ACL（访问控制列表：细粒度权限）
  └── umask               └── chmod/chown/chgrp（权限修改命令）
```

### 重要命令汇总

| 命令 | 用途 |
|------|------|
| `useradd` | 添加用户 |
| `usermod` | 修改用户属性 |
| `userdel` | 删除用户 |
| `passwd` | 管理密码 |
| `groupadd` | 创建用户组 |
| `groupdel` | 删除用户组 |
| `id` | 查看用户UID/GID/组信息 |
| `groups` | 列出用户所属组 |
| `chmod` | 修改文件权限（数字/符号模式） |
| `chown` | 修改文件所有者和组 |
| `chgrp` | 修改文件所属组 |
| `umask` | 查看/设置默认权限掩码 |
| `setfacl` | 设置ACL规则 |
| `getfacl` | 查看ACL规则 |
| `whoami` | 查看当前用户名 |
| `who` / `w` | 查看当前登录用户 |

---

*笔记生成日期：2026-07-22 | 转写工具：OpenAI Whisper tiny (CPU)*
*优化日期：2026-07-22 | 新增：命令详解、特殊权限、ACL、umask、实操流程*
