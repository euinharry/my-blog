---
title: "第27讲：ARM-GCC与交叉编译"
date: 2026-09-11T09:12:00+08:00
draft: false
description: "arm-linux-gnueabihf-gcc hello.c -o hello_arm"
series: ["Linux 入门"]
series_order: 15
categories: ["技术笔记"]
tags: ["Linux", "GCC", "交叉编译"]
---

> **视频来源**：野火嵌入式Linux教学视频系列
> **BV号**：BV1JK4y1t7io (P28)
> **标题**：第27讲 — ARM-GCC与交叉编译
> **时长**：24分24秒
> **转写方式**：Whisper tiny (CPU) 语音识别

---

## 1. 本地编译 vs 交叉编译

### 1.1 本地编译（Native Compilation）

> **编译器 与 目标程序 运行在相同平台**

| 示例 | PC上运行 | 目标平台 |
|------|---------|---------|
| Ubuntu上 `gcc hello.c -o hello` → `./hello` | x86_64 PC | x86_64 (本地) |
| 开发板上 `gcc hello.c -o hello` → `./hello` | ARM开发板 | ARM (本地) |

### 1.2 交叉编译（Cross Compilation）

> **编译器 与 目标程序 运行在不同架构**

```bash
# 在 x86_64 PC 上编译 ARM 架构的可执行程序
arm-linux-gnueabihf-gcc hello.c -o hello_arm
```

**交叉编译工具链命名规则（完整版）**：

交叉编译器的命名遵循格式：`arch-vendor-kernel-system`

| 字段 | 含义 | 示例值 |
|------|------|--------|
| `arch` | 目标 CPU 架构 | `arm`, `aarch64`, `riscv64`, `mips` |
| `vendor` | 工具链提供商（可省略） | `none` (无特定厂商), `linaro` |
| `kernel` | 目标操作系统 | `linux`, `none` (裸机) |
| `system` | 系统库/ABI | `gnueabi`, `gnueabihf`, `musleabi` |

**常见架构的交叉编译器示例**：

| 架构 | 交叉编译器 | 适用场景 |
|------|-----------|---------|
| ARM (32位) | `arm-linux-gnueabihf-gcc` | ARM Cortex-A 系列，运行 Linux |
| ARM 裸机 | `arm-none-eabi-gcc` | ARM Cortex-M 系列，裸机/RTOS |
| AArch64 (64位ARM) | `aarch64-linux-gnu-gcc` | ARM Cortex-A53/A72 等 64位 ARM |
| RISC-V (64位) | `riscv64-linux-gnu-gcc` | RISC-V 架构 Linux 系统 |
| MIPS (32位) | `mips-linux-gnu-gcc` | MIPS 架构嵌入式 Linux |

### 1.3 为什么需要交叉编译？

- 编译过程对CPU运算能力要求高
- 嵌入式设备（ARM/MCU）性能低
- 单片机（MCU）根本无法安装编译器
- 用强大PC快速完成编译工作

### 1.4 分辨本地可执行文件与交叉编译可执行文件

使用 `file` 命令可以查看可执行文件的目标架构：

```bash
# 查看 x86_64 本地编译的可执行文件
file hello_x86
# 输出示例：
# hello_x86: ELF 64-bit LSB executable, x86-64, version 1 (SYSV),
# dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, ...

# 查看 ARM 交叉编译的可执行文件
file hello_arm
# 输出示例：
# hello_arm: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV),
# dynamically linked, interpreter /lib/ld-linux-armhf.so.3, ...
```

**关键字段解读**：

| 字段 | x86_64 程序 | ARM 程序 |
|------|-----------|---------|
| 格式 | `ELF 64-bit` | `ELF 32-bit` |
| 架构 | `x86-64` | `ARM` |
| 解释器 | `/lib64/ld-linux-x86-64.so.2` | `/lib/ld-linux-armhf.so.3` |

---

## 2. ARM GCC 是什么？

### 2.1 定义

> **ARM-GCC = 针对 ARM 平台的 GCC 编译器**

- GCC = GNU Compiler Collection
- GCC 是一大类编译器集合，可针对不同平台做进一步细分
- ARM-GCC 是 GCC 针对 ARM 架构的分支

### 2.2 GCC工具链命名解析

#### Ubuntu PC 上的 GCC（x86_64）

```bash
which gcc                    # /usr/bin/gcc
ls -l /usr/bin/gcc           # 链接 → gcc-7
ls -l /usr/bin/gcc-7         # 链接 → x86_64-linux-gnu-gcc-7
```

命名规则：`x86_64-linux-gnu-gcc-7`

| 部分 | 含义 |
|------|------|
| `x86_64` | 目标架构（x86_64） |
| `linux` | 目标系统（Linux） |
| `gnu` | 使用GNU组织的glibc库 |
| `gcc` | GCC编译器 |
| `7` | 版本号 |

#### 开发板上的 GCC（ARM）

```bash
which gcc                    # /usr/bin/gcc
ls -l /usr/bin/gcc           # 链接 → arm-linux-gnueabihf-gcc-8
```

命名规则：`arm-linux-gnueabihf-gcc-8`

| 部分 | 含义 |
|------|------|
| `arm` | 目标架构（ARM） |
| `linux` | 目标系统（Linux） |
| `gnu` | 使用GNU组织的glibc库 |
| `eabi` | 嵌入式应用二进制接口（Embedded ABI） |
| `hf` | 硬浮点（Hard Float） |
| `gcc` | GCC编译器 |
| `8` | 版本号 |

### 2.3 AAPCS：ARM 过程调用标准

**AAPCS** = ARM Architecture Procedure Call Standard

这是在 ARM 架构上进行函数调用时需要遵守的规范，定义了：

| 要点 | 说明 |
|------|------|
| **寄存器用途** | r0-r3 用于传参，r0 用于返回值 |
| **栈对齐** | 栈指针必须 8 字节对齐 |
| **参数传递** | 前 4 个参数通过寄存器传递，其余通过栈 |
| **浮点寄存器** | 硬浮点 ABI 使用 VFP/NEON 寄存器传浮点参数 |

`eabi` 中的 "e" 就是指嵌入式环境下的 ABI 扩展，相比桌面 ABI 更为精简。

---

## 3. ARM GCC 的进一步分类

### 3.1 ARM GCC 的分类维度

ARM GCC 可以从以下三个维度进行细分：

#### 按目标架构

| 架构 | 示例编译器 | 说明 |
|------|-----------|------|
| x86_64 | `x86_64-linux-gnu-gcc` | PC/服务器 |
| ARM (32位) | `arm-linux-gnueabihf-gcc` | 32位 ARM (armv7及以下) |
| AArch64 (64位ARM) | `aarch64-linux-gnu-gcc` | 64位 ARM (armv8及以上) |
| RISC-V | `riscv64-linux-gnu-gcc` | RISC-V 架构 |
| MIPS | `mips-linux-gnu-gcc` | MIPS 架构 |

#### 按是否支持操作系统

| 类型 | 说明 | 典型编译器 | 应用场景 |
|------|------|-----------|---------|
| `linux` | 支持Linux系统（应用程序） | `arm-linux-gnueabihf-gcc` | Linux 应用、驱动模块 |
| `none` | 裸机/无操作系统 | `arm-none-eabi-gcc` | uboot、固件、RTOS、MCU程序 |

**`arm-linux-gnueabihf-gcc` 与 `arm-none-eabi-gcc` 的核心区别**：

| 对比项 | arm-linux-gnueabihf | arm-none-eabi |
|--------|--------------------|---------------|
| 目标系统 | Linux | 裸机 / RTOS |
| C 库 | glibc | newlib (精简版) |
| 系统调用 | 有 (通过 glibc) | 无 |
| 标准输入输出 | 完整支持 printf/scanf | 需自行实现底层 I/O |
| 动态链接 | 支持 (.so) | 不支持 (仅静态链接) |
| 典型应用 | Linux 应用程序 | STM32 固件、uboot |
| 可执行格式 | ELF (动态链接) | ELF / BIN / HEX |

#### 按浮点支持

| 类型 | 全称 | 说明 | 性能 | 代码体积 |
|------|------|------|------|---------|
| `hf` | Hard Float | CPU 有硬件浮点运算单元 (FPU/VFP) | 高 | 小 |
| 无标记 (sf) | Soft Float | 用软件模拟浮点运算 | 低（慢10-50倍） | 大 |

**硬浮点 vs 软浮点的选择策略**：

1. **首选 hf（硬浮点）**：只要目标 CPU 有硬件 FPU（如 Cortex-A 系列都有），就应该使用硬浮点版本。
2. **软浮点的适用场景**：
   - 目标 CPU 没有硬件 FPU（如 Cortex-M0/M1）
   - 需要跨不同 ARM 芯片的二进制兼容性（软浮点程序不依赖特定 FPU）
3. **注意**：硬浮点编译器编译的程序不能在无 FPU 的 CPU 上运行。软浮点程序则可以在有 FPU 的 CPU 上运行（但不会利用硬件加速）。

---

## 4. 安装 ARM GCC 交叉编译工具链

### 方式一：apt 安装（推荐）

```bash
# Ubuntu 上安装 ARM GCC 交叉编译工具链
sudo apt update
sudo apt install gcc-arm-linux-gnueabihf

# 验证安装 — 查看版本
arm-linux-gnueabihf-gcc --version
# 输出示例：
# arm-linux-gnueabihf-gcc (Ubuntu 9.3.0-17ubuntu1~20.04) 9.3.0

# 验证安装 — 查看编译器所在路径
which arm-linux-gnueabihf-gcc
# 输出：/usr/bin/arm-linux-gnueabihf-gcc

# 验证安装 — 查看编译器支持的目标架构
arm-linux-gnueabihf-gcc -dumpmachine
# 输出：arm-linux-gnueabihf
```

> 在 Ubuntu PC 上直接 `apt install gcc` 安装的是 x86_64 本地编译器。
> 要安装 ARM 交叉编译器，必须指定完整包名。

**其他架构的 apt 安装命令**：

```bash
# AArch64 (64位ARM) 交叉编译器
sudo apt install gcc-aarch64-linux-gnu

# ARM 裸机交叉编译器（STM32等MCU开发）
sudo apt install gcc-arm-none-eabi
```

### 方式二：从 Linaro 官网下载

**Linaro简介**：
- 由ARM公司发起
- TI、三星、NXP等半导体公司共同参与
- 非营利性组织
- 提供预编译的交叉编译工具链

官网地址：`https://www.linaro.org/`

**下载步骤**：

1. 访问 Linaro 官网，进入 Downloads 页面
2. 选择对应的架构和版本（如 `arm-linux-gnueabihf`）
3. 下载 `.tar.xz` 压缩包

```bash
# 示例：下载并解压 Linaro 工具链
wget https://releases.linaro.org/components/toolchain/binaries/latest-7/arm-linux-gnueabihf/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf.tar.xz

# 解压到 /opt 目录（需要 sudo）
sudo tar -xJf gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf.tar.xz -C /opt/

# 验证
/opt/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf-gcc --version
```

> 由于下载链接在国外，速度较慢，不推荐此方式。建议直接用 apt 安装。

### 方式三：Bootlin 工具链（备选）

Bootlin (前身为 Free Electrons) 也提供预编译的交叉编译工具链：

- 官方网站：`https://toolchains.bootlin.com/`
- 支持多种架构：ARM、AArch64、MIPS、RISC-V 等
- 提供 glibc / musl / uClibc 多种 C 库选项

### 将交叉编译器添加到 PATH

如果手动下载安装（未通过 apt），需要将工具链的 `bin` 目录添加到 `PATH`：

```bash
# 临时添加（当前终端有效）
export PATH=/opt/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf/bin:${PATH}

# 永久添加（写入 ~/.bashrc）
echo 'export PATH=/opt/gcc-linaro-7.5.0-2019.12-x86_64_arm-linux-gnueabihf/bin:${PATH}' >> ~/.bashrc
source ~/.bashrc
```

---

## 5. 实验验证

### 实验1：在开发板运行 x86 程序 — 失败

```bash
# PC上编译的 x86 程序
gcc hello.c -o hello

# 查看文件类型（确认是 x86_64）
file hello
# 输出：hello: ELF 64-bit LSB executable, x86-64, ...

# 复制到开发板
sudo cp hello /mnt/ubuntushare/

# 开发板上运行 — 出错
./hello
# 错误：Exec format error（可执行格式错误）
```

> **结论**：不同平台的可执行文件格式不同，不能混用。

**为什么 x86 程序不能在 ARM 上运行？**

- x86_64 使用 x86 指令集（CISC），ARM 使用 ARM 指令集（RISC）
- 两者的二进制编码完全不同，CPU 无法解码对方的指令
- 即使用户态指令碰巧相同，系统调用接口、内存布局等也有根本差异

### 实验2：交叉编译 — 在开发板运行 — 成功

```bash
# 1. PC上编写hello程序（hello_arm.c）
#include <stdio.h>
int main() {
    printf("hello arm\n");
    return 0;
}

# 2. 用ARM交叉编译器编译
arm-linux-gnueabihf-gcc hello_arm.c -o hello_arm

# 3. 查看编译产物类型
file hello_arm
# 输出：hello_arm: ELF 32-bit LSB executable, ARM, ...

# 4. 复制到共享文件夹
sudo cp hello_arm /mnt/ubuntushare/

# 5. 开发板上执行
./hello_arm
# 输出：hello arm

# 6. 再次确认开发板上运行的文件确实是 ARM 格式
# 在开发板上执行：
file hello_arm
# 输出：hello_arm: ELF 32-bit LSB executable, ARM, ...
```

### 实验3：使用 readelf 对比 x86 和 ARM 可执行文件

`readelf -h` 可以查看 ELF 文件的头部信息，包含目标架构：

```bash
# 查看 x86_64 程序的 ELF 头
readelf -h hello
# Machine:                           Advanced Micro Devices X86-64
# Class:                             ELF64
# Entry point address:               0x1050

# 查看 ARM 程序的 ELF 头
readelf -h hello_arm
# Machine:                           ARM
# Class:                             ELF32
# Entry point address:               0x3f9
```

**关键区别**：

| 字段 | x86_64 程序 | ARM 程序 |
|------|-----------|---------|
| `Machine` | `Advanced Micro Devices X86-64` | `ARM` |
| `Class` | `ELF64` (64位) | `ELF32` (32位) |
| 字节序 | `Little endian` | `Little endian` (通常) |

### 实验4：反汇编对比 ARM 与 x86 指令

使用 `objdump -d` 查看编译后的汇编代码：

```bash
# 反汇编 x86_64 程序（只看 main 函数）
objdump -d hello | grep -A 20 '<main>:'
# 输出示例（x86_64 AT&T 语法）：
# 0000000000001149 <main>:
#   1149:   55             push   %rbp
#   114a:   48 89 e5       mov    %rsp,%rbp
#   114d:   48 8d 3d b0 0e lea    0xeb0(%rip),%rdi
#   1154:   e8 e7 fe ff ff call   1040 <puts@plt>
#   1159:   b8 00 00 00 00 mov    $0x0,%eax
#   115e:   5d             pop    %rbp
#   115f:   c3             ret

# 反汇编 ARM 程序（只看 main 函数）
arm-linux-gnueabihf-objdump -d hello_arm | grep -A 20 '<main>:'
# 输出示例（ARM 32位指令）：
# 000003f9 <main>:
#    3f9:   b580          push   {r7, lr}
#    3fb:   af00          add    r7, sp, #0
#    3fd:   f240 0400     movw   r4, #0
#    401:   f2c0 0400     movt   r4, #0
#    405:   4620          mov    r0, r4
#    407:   f7ff ef7e     blx    308 <puts@plt>
#    40b:   2300          movs   r3, #0
#    40d:   4618          mov    r0, r3
#    40f:   bd80          pop    {r7, pc}
```

对比可以看出：
- x86_64 使用 `push/pop rbp`, `mov`, `call`, `ret`
- ARM 使用 `push/pop {r7, lr}`, `blx`, `mov` 等完全不同的指令

### 实验5：使用 -v 查看交叉编译的详细过程

```bash
# -v 选项输出编译器的详细工作过程
arm-linux-gnueabihf-gcc -v hello_arm.c -o hello_arm
# 输出包括：
# 1. COLLECT_GCC_OPTIONS：编译器选项
# 2. cc1：预处理器和编译器（C → 汇编）
# 3. as：汇编器（汇编 → 目标文件）
# 4. collect2：链接器（目标文件 → 可执行文件）
# 5. 使用的库路径和链接的库文件
```

`-v` 输出的关键信息：
- 编译器的配置选项（`--target=`, `--prefix=`, `--with-` 等）
- 头文件搜索路径
- 库文件搜索路径
- 传递给子进程（cc1, as, collect2）的具体参数

---

## 6. NFS 挂载共享文件夹

### 6.1 NFS 简介

NFS (Network File System) 允许开发板通过网络访问 Ubuntu 上的目录，避免了反复拔插 SD 卡或手动复制文件的麻烦。

### 6.2 Ubuntu 端 NFS 服务配置

```bash
# 1. 安装 NFS 服务器
sudo apt update
sudo apt install nfs-kernel-server

# 2. 创建共享目录
sudo mkdir -p /nfs_share
sudo chmod 777 /nfs_share

# 3. 配置 NFS 导出（编辑 /etc/exports）
# 添加以下行（* 表示允许所有客户端，可根据需要限制 IP）
# /nfs_share *(rw,sync,no_subtree_check,no_root_squash)

# 更安全的配置（仅允许特定网段）：
# /nfs_share 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)

# 4. 重新加载 NFS 配置
sudo exportfs -arv

# 5. 启动 NFS 服务
sudo systemctl restart nfs-kernel-server
sudo systemctl enable nfs-kernel-server
```

**`/etc/exports` 常用选项说明**：

| 选项 | 含义 |
|------|------|
| `rw` | 读写权限 |
| `ro` | 只读权限 |
| `sync` | 同步写入（数据安全，但性能较低） |
| `async` | 异步写入（性能高，但断电可能丢数据） |
| `no_subtree_check` | 不检查子目录（提高可靠性） |
| `no_root_squash` | 允许远程 root 用户保持 root 权限 |
| `root_squash` | 将远程 root 用户映射为匿名用户（更安全） |

### 6.3 查看 NFS 导出目录

```bash
# 在开发板上查看 Ubuntu 导出（共享）的 NFS 目录
showmount -e 192.168.1.179
# 输出示例：
# Export list for 192.168.1.179:
# /nfs_share 192.168.1.0/24
```

### 6.4 开发板挂载 NFS

```bash
# 开发板上执行挂载
mount -t nfs -o nolock 192.168.1.179:/nfs_share /mnt/ubuntu
```

**mount 命令选项说明**：

| 选项 | 含义 |
|------|------|
| `-t nfs` | 指定文件系统类型为 NFS |
| `-o nolock` | 不使用 NFS 文件锁（嵌入式环境常见） |
| `-o rw` | 以读写模式挂载 |
| `-o nfsvers=3` | 指定 NFS 版本（3 或 4） |
| `-o proto=tcp` | 使用 TCP 协议（默认可能是 UDP） |

**完整挂载示例（推荐）**：

```bash
# 开发板上
mount -t nfs -o nolock,nfsvers=3,proto=tcp 192.168.1.179:/nfs_share /mnt/ubuntu

# 验证挂载
df -h | grep nfs
# 输出示例：
# 192.168.1.179:/nfs_share   50G   20G   30G  40% /mnt/ubuntu

ls /mnt/ubuntu
# 应能看到 Ubuntu 共享目录中的文件
```

挂载成功后，开发板可以直接访问 Ubuntu 中 `/nfs_share` 目录的文件，无需手动复制。

**卸载 NFS**：

```bash
# 开发板上卸载 NFS
umount /mnt/ubuntu
```

---

## 7. 拓展知识

### 7.1 主流交叉编译工具链提供商

| 提供商 | 说明 | 获取方式 |
|--------|------|---------|
| **GNU 官方** | GCC 官方源码，需自行编译 | `gcc.gnu.org` |
| **Linaro** | ARM 发起的优化工具链 | `releases.linaro.org` |
| **Bootlin** | 多架构预编译工具链 | `toolchains.bootlin.com` |
| **ARM 官方** | ARM Compiler (armclang) | `developer.arm.com` |
| **发行版源** | Ubuntu/Debian 官方软件源 | `apt install gcc-arm-linux-gnueabihf` |
| **Yocto/OpenEmbedded** | 嵌入式 Linux 构建系统自带的工具链 | `yoctoproject.org` |
| **Buildroot** | 轻量级嵌入式 Linux 构建工具 | `buildroot.org` |
| **crosstool-NG** | 灵活的交叉工具链构建工具 | `crosstool-ng.org` |

### 7.2 Makefile 预告

在实际嵌入式开发中，不会每次都手动输入编译命令。后续章节将介绍 `Makefile` 的基本用法：

- Makefile 是自动化构建工具 `make` 的配置文件
- 定义编译规则、依赖关系和构建目标
- 一次编写，`make` 命令自动完成所有编译步骤
- 支持增量编译（只重新编译修改过的文件）

```makefile
# 简单的交叉编译 Makefile 示例（预告）
CC = arm-linux-gnueabihf-gcc
CFLAGS = -Wall -O2

hello_arm: hello_arm.c
	$(CC) $(CFLAGS) hello_arm.c -o hello_arm

clean:
	rm -f hello_arm
```

---

## 本讲核心概念总结

| 概念 | 说明 |
|------|------|
| **本地编译** | 编译器与目标程序在同一平台 |
| **交叉编译** | 编译器与目标程序在不同架构 |
| **ARM-GCC** | 针对ARM平台的GCC分支 |
| **工具链命名** | `arch-vendor-kernel-system` 格式 |
| **命名示例** | `arm-linux-gnueabihf-gcc` (ARM, Linux, GNU, 硬浮点) |
| **安装ARM GCC** | `apt install gcc-arm-linux-gnueabihf` |
| **关键验证** | x86程序不能运行在ARM上，须交叉编译 |
| **file 命令** | 查看可执行文件的目标架构 |
| **readelf -h** | 查看 ELF 头中的 Machine (架构) 字段 |
| **objdump -d** | 反汇编查看 ARM 指令 vs x86 指令 |
| **NFS** | 网络共享文件系统，方便开发板访问 PC 文件 |

---

*笔记生成日期：2026-07-22 | 转写：Whisper tiny | 优化日期：2026-07-24*
