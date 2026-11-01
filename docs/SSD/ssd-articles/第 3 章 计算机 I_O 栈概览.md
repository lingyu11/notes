第 3 章 计算机 I/O 栈概览

> **本章目标**：理解数据从应用程序到 SSD 的完整路径，建立 I/O 栈分层认知  
> **前置章节**：第 2 章 SSD 是什么：从用户视角理解  
> **下一章**：第 4 章 半导体存储器分类

## 章节概述

你在编辑器里按下 Ctrl+S 保存文件，看起来只是一个简单动作——背后却经过应用程序、文件系统、内核块层、设备驱动，最终到达 SSD 硬件。这条路径就是 **I/O 栈（I/O Stack）**。

本章从这条路径出发，解释各层的职责与协作方式，介绍块设备与字符设备的区别，讲解 DMA 与中断如何让 CPU 从数据搬运中解放出来，对比同步与异步 I/O 的差异（这是理解 NVMe 多队列的基础），并给出 Linux / Windows 下查看 SSD 信息的实用命令。

读完本章，你将能回答：**「我写入的一个 4 KB 文件，在软件层面经历了什么？」**

## 3.1 应用程序 → 文件系统 → 块层 → 驱动 → 设备

### 核心概念

操作系统将 I/O 组织为**分层架构**，每层只与相邻层交互，上层不需要知道底层硬件细节。这种设计叫 **分层抽象（Layered Abstraction）**——应用程序只需调用 `write()`，不必关心数据最终写进了哪块 NAND。

### 完整数据路径

![](images/6c07994909b7.png)

<p align="center">图 3-1：从应用程序到 SSD 的 I/O 栈分层架构</p>

### 各层职责详解

| 层次 | 英文 | 核心职责 | 典型组件 |
| --- | --- | --- | --- |
| **应用层** | Application Layer | 发起 I/O 请求（读/写文件） | 编辑器、数据库、浏览器 |
| **系统调用接口** | System Call Interface | 用户态进入内核态的入口 | `open()` `read()` `write()` `close()` |
| **虚拟文件系统** | VFS（Virtual File System） | 统一不同文件系统的接口 | Linux VFS、Windows I/O Manager |
| **文件系统** | File System | 管理文件/目录、权限、元数据 | ext4、XFS、NTFS、APFS |
| **页缓存** | Page Cache / Buffer Cache | 在内存中缓存文件数据，减少磁盘访问 | Linux Page Cache、Windows Cache Manager |
| **块层** | Block Layer | 将文件 I/O 转换为块（Block）I/O，调度与合并 | Linux Block Layer、bio 结构 |
| **设备驱动** | Device Driver | 与具体硬件通信，发送命令 | `nvme` 驱动、`ahci` 驱动 |
| **总线/协议** | Bus / Protocol | 物理传输通道 | PCIe + NVMe、SATA + AHCI |
| **存储设备** | Storage Device | 实际存储介质 | SSD 控制器 + NAND Flash |

### 一次写操作的完整旅程

以 Linux 下保存一个 4 KB 文件为例：

**第一步：应用程序发起写请求**

```c
// 用户态代码（简化）  
int fd = open("/home/user/doc.txt", O_WRONLY);  
write(fd, buffer, 4096);  // 写入 4 KB 数据  
close(fd);
```

**第二步：系统调用进入内核**

`write()` 触发系统调用（syscall），CPU 从用户态切换到内核态，控制权交给内核的文件系统代码。

**第三步：文件系统处理**

* VFS 根据路径 `/home/user/doc.txt` 找到对应的文件系统（如 ext4）
* ext4 查找 inode，确定数据应写入哪些逻辑块（Logical Block）
* 数据先写入 **Page Cache**（页缓存），此时数据仅在内存中，尚未到达 SSD

**第四步：刷盘（Flush to Disk）**

以下情况之一触发数据从 Page Cache 写入 SSD：

* 调用 `fsync()` / `fdatasync()` 强制刷盘
* 调用 `close()` 且文件系统配置了 write-back 策略
* 内核 `pdflush` / `writeback` 线程定期将脏页刷出
* 内存压力触发回写

**第五步：块层处理**

* 块层收到写请求，构造 **bio**（Block I/O）结构
* 将多个相邻的小 I/O **合并**（merge）为更大的请求以提升效率
* 通过 **I/O 调度器**（如 mq-deadline、none）决定发送顺序
* NVMe SSD 通常建议调度器设为 `none`（多队列本身已足够高效）

**第六步：设备驱动发送命令**

* NVMe 驱动将 bio 转换为 NVMe Read/Write 命令
* 命令写入 **Submission Queue（SQ，提交队列）**
* 通过 Doorbell 寄存器通知 SSD 控制器

**第七步：SSD 执行并返回**

* SSD 控制器接收命令，FTL 将 LBA 映射到 PBA
* 数据通过 DMA 写入 NAND Flash
* 完成后在 **Completion Queue（CQ，完成队列）** 写入完成条目
* 触发 MSI-X 中断通知 CPU（或驱动轮询 CQ）

**第八步：完成通知回到应用**

* 驱动处理完成事件，块层标记 bio 完成
* 文件系统更新元数据
* 若应用调用的是同步 I/O，阻塞的线程被唤醒

### 读操作的关键差异

读路径与写路径类似，但有一个重要优化：

| 操作 | Page Cache 行为 |
| --- | --- |
| **读** | 先查 Page Cache，命中则直接返回（**缓存命中**），无需访问 SSD |
| **写** | 先写 Page Cache（**写回策略**），延迟刷到 SSD |

这就是为什么第二次打开同一个文件比第一次快——数据已在内存中。

### 生活类比：快递寄送

| I/O 栈层次 | 快递类比 |
| --- | --- |
| 应用程序 | 你填写快递单、交包裹 |
| 文件系统 | 快递公司分拣中心，记录包裹信息 |
| Page Cache | 本地暂存仓库，攒一批再发 |
| 块层 | 规划最优路线、合并同方向包裹 |
| 设备驱动 | 快递员，按地址取送 |
| SSD | 目的地仓库 |

---

## 3.2 块设备（Block Device）与字符设备

### 核心概念

Linux 将设备分为两大类，这是理解 `/dev` 目录下设备文件的基础

![](images/7365712dece5.png)

<p align="center">图 3-2：块设备与字符设备的访问方式对比</p>

| 特性 | 块设备（Block Device） | 字符设备（Character Device） |
| --- | --- | --- |
| **访问单位** | 固定大小的块（通常 512 B 或 4 KB） | 单个字节或字符流 |
| **访问方式** | 随机访问（可跳到任意块） | 顺序访问（流式） |
| **是否可 seek** | 可以（`lseek()`） | 通常不可以 |
| **是否有缓存** | 有（Page Cache） | 通常无（或极小） |
| **典型设备** | HDD、SSD、U 盘 | 键盘、鼠标、串口、/dev/null |
| **Linux 设备号** | 主设备号区分驱动 | 独立的主设备号 |

### SSD 是块设备

SSD 在操作系统中以 **块设备** 形式呈现：

```bash
# Linux 下查看块设备  
lsblk  
  
# 典型输出  
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT  
nvme0n1     259:0    0   1.8T  0 disk  
├─nvme0n1p1 259:1    0   512M  0 part /boot/efi  
└─nvme0n1p2 259:2    0   1.8T  0 part /
```

* `nvme0n1`：第一块 NVMe 命名空间（namespace），整盘
* `nvme0n1p1`：第一个分区
* `MAJ:MIN`：主设备号:次设备号，内核用来识别设备类型

### 块大小（Block Size）

| 概念 | 含义 | 典型值 |
| --- | --- | --- |
| **逻辑扇区大小** | OS 与驱动交互的最小单位 | 512 B 或 4 KB |
| **物理扇区大小** | SSD 内部实际页大小 | 通常 4 KB 或 16 KB |
| **文件系统块大小** | mkfs 时设定 | ext4 默认 4 KB |

> **对齐的重要性**：如果文件系统块大小与 SSD 页大小不对齐，一次写入可能触发两次 NAND 页读写（读-改-写），影响性能和寿命。现代 OS 和 SSD 默认 4 KB 对齐，通常无需手动处理。

### 直接访问块设备 vs 通过文件系统

```bash
# 通过文件系统访问（常规方式）  
echo "hello" > /home/user/test.txt  
  
# 直接访问块设备（绕过文件系统，危险！）  
sudo dd if=/dev/zero of=/dev/nvme0n1 bs=4K count=1  
# ⚠️ 这会破坏分区表和数据，仅用于测试环境
```

**DB（Direct I/O）**：应用程序可以绕过 Page Cache，直接与块设备交互：

```c
// O_DIRECT 标志：绕过 Page Cache  
int fd = open("/dev/nvme0n1", O_RDWR | O_DIRECT);
```

数据库（如 MySQL InnoDB）和性能测试工具（如 fio）常用 Direct I/O，避免双重缓存（Page Cache + 应用自身缓存）。

## 3.3 DMA、中断与 I/O 完成通知机制

### 为什么需要 DMA

如果由 CPU 逐字节搬运数据，I/O 期间 CPU 无法做其他工作，效率极低。**DMA（Direct Memory Access，直接内存访问）** 允许设备控制器直接在内存与设备之间传输数据，CPU 只需发起命令和处理完成通知。

![](images/95d1f77d7549.png)

<p align="center">图 3-3：DMA 数据传输与中断完成通知流程</p>

### DMA 工作流程

| 步骤 | 执行者 | 动作 |
| --- | --- | --- |
| 1 | CPU / 驱动 | 构造 I/O 命令，指定数据在内存中的地址（物理地址） |
| 2 | CPU / 驱动 | 将命令写入设备队列，敲 Doorbell 通知设备 |
| 3 | CPU | **继续执行其他任务** （DMA 的关键价值） |
| 4 | 设备（SSD） | DMA 引擎直接从主机内存读取/写入数据 |
| 5 | 设备（SSD） | 操作完成，写入 CQ 完成条目 |
| 6 | 设备 → CPU | 触发 **中断**（或等待驱动轮询） |
| 7 | CPU / 驱动 | 处理完成事件，唤醒等待线程 |

### 中断 vs 轮询（Polling）

NVMe SSD 完成 I/O 后通知 CPU 有两种方式：

| 方式 | 原理 | 优势 | 劣势 |
| --- | --- | --- | --- |
| **中断（Interrupt）** | 设备完成后触发 MSI-X 中断，CPU 跳转到中断处理程序 | CPU 空闲时功耗低 | 高中断频率时开销大 |
| **轮询（Polling）** | 驱动定期主动检查 CQ 是否有新完成条目 | 高 IOPS 场景延迟更低 | CPU 占用更高 |

**中断合并（Interrupt Coalescing）**：在高负载下，SSD 不会每个 I/O 完成都触发一次中断，而是积累若干完成条目后一次性触发，减少中断开销。这是 NVMe 驱动中常见的优化手段。

### NVMe 的 MSI-X 中断

NVMe 使用 **MSI-X（Message Signaled Interrupts Extended）**，相比传统中断有两个优势：

* 支持多达 2048 个独立中断向量，每个 I/O 队列可绑定独立中断
* 多核 CPU 环境下，不同核心的 I/O 完成中断可以路由到对应核心，减少缓存一致性开销

> PCIe 与 MSI-X 详解见 第 18 章。

### 生活类比

* **无 DMA**：老板（CPU）亲自搬货，搬完才能做其他事
* **有 DMA**：老板下单给物流（DMA 引擎），物流搬货期间老板继续开会
* **中断**：物流搬完打电话通知老板「货到了」
* **轮询**：老板每隔 5 分钟自己查物流 App

## 3.4 同步 I/O vs 异步 I/O

### 核心概念

I/O 模型决定了应用程序发起读写后，线程是否阻塞等待结果。这是理解 NVMe 多队列价值的关键——**异步 + 多队列** 才能充分释放 SSD 的并行能力。

![](images/f171fc560839.png)

<p align="center">图 3-4：同步 I/O（阻塞等待）与异步 I/O（提交后继续工作）</p>

### 同步 I/O（Synchronous I/O）

应用程序调用 `read()` / `write()` 后，**线程阻塞**，直到 I/O 完成才返回。

```
线程：提交 I/O → [阻塞等待] → I/O 完成 → 继续执行  
时间：|----空闲等待----|
```

| 特点 | 说明 |
| --- | --- |
| 编程简单 | 顺序执行，逻辑直观 |
| CPU 利用率低 | 等待 I/O 期间线程阻塞 |
| 无法并行 | 一次只能有一个在途 I/O（单线程场景） |

### 异步 I/O（Asynchronous I/O）

应用程序提交 I/O 请求后**立即返回**，继续执行其他逻辑；I/O 完成时通过回调或事件通知。

```
线程：提交 I/O1 → 提交 I/O2 → 提交 I/O3 → 做其他计算 → 检查完成 → 处理结果  
时间：|----充分利用 CPU----|
```

| 特点 | 说明 |
| --- | --- |
| 高吞吐量 | 多个 I/O 同时在途，充分利用 SSD 队列深度 |
| 编程复杂 | 需要处理完成事件、错误、顺序依赖 |
| 适合高并发 | 数据库、NVMe 基准测试、SPDK 应用 |

### Linux 异步 I/O 接口演进

| 接口 | 说明 | 状态 |
| --- | --- | --- |
| **POSIX AIO（libaio）** | `aio_read()` / `aio_write()` | 仅支持 Direct I/O，功能有限，逐渐淘汰 |
| **io\_uring** | Linux 5.1+ 引入，共享环形队列，零拷贝 | 现代高性能首选 |
| **SPDK** | 用户态 NVMe 驱动，完全绕过内核 | 存储系统开发，NVMe-oF |

### io\_uring 简介（现代 Linux 异步 I/O）

io\_uring 是 Linux 目前最高效的异步 I/O 接口，与 NVMe 多队列理念高度契合：

```
用户态                         内核态  
┌─────────────┐               ┌─────────────┐  
│             │  SQ（提交环）  │             │  
│  应用程序    │ ──────────→   │  块层/驱动  │  
│             │  CQ（完成环）  │             │  
│             │ ←──────────   │             │  
└─────────────┘               └─────────────┘
```

* **SQ（Submission Queue Ring）**：用户提交 I/O 请求
* **CQ（Completion Queue Ring）**：内核返回完成结果
* 与 NVMe 的 SQ/CQ 形成「双环」结构，减少系统调用开销

### 为什么 NVMe 需要异步 I/O

NVMe SSD 支持 **65535 个 I/O 队列，每队列 65535 深度**。如果应用使用同步 I/O，每次只提交一个命令，SSD 的大量并行能力被浪费：

| I/O 模型 | 在途命令数（单线程） | SSD 利用率 |
| --- | --- | --- |
| 同步 I/O | 1 | 极低（< 1%） |
| 同步 + 多线程 | 线程数 | 中等 |
| 异步 I/O + 高队列深度 | 数百～数千 | 高 |

**fio 中的体现**：

```bash
# 同步 I/O，队列深度 1（模拟最差情况）  
fio --rw=randread --iodepth=1 --ioengine=sync  
  
# 异步 I/O，队列深度 128（充分压榨 NVMe）  
fio --rw=randread --iodepth=128 --ioengine=libaio
```

同样一块 NVMe SSD，iodepth=128 的 IOPS 可能是 iodepth=1 的 **10–50 倍**。

### NVMe 专用 I/O 路径

![](images/d64fce582baf.png)

<p align="center">图 3-5：NVMe SSD 的 I/O 路径（应用 → 驱动 → SQ/CQ → SSD 控制器）</p>

> NVMe 队列机制详解见 第 19 章，io\_uring 与 SPDK 实践见 第 26 章。

## 3.5 Linux / Windows 下查看 SSD 信息的常用命令

### Linux 命令速查

#### 查看块设备概览

```bash
# 列出所有块设备（树形）  
lsblk  
  
# 显示磁盘拓扑（含 PCIe 路径）  
lsblk -t  
  
# 查看块大小、对齐、是否支持 TRIM  
lsblk -D /dev/nvme0n1
```

#### NVMe 专用工具（nvme-cli）

```bash
# 列出所有 NVMe 设备  
sudo nvme list  
  
# 查看控制器能力（最大队列数、命名空间数等）  
sudo nvme id-ctrl /dev/nvme0  
  
# 查看命名空间信息（容量、LBA 格式）  
sudo nvme id-ns /dev/nvme0n1  
  
# SMART 健康日志  
sudo nvme smart-log /dev/nvme0  
  
# 查看错误日志  
sudo nvme error-log /dev/nvme0  
  
# 查看固件版本  
sudo nvme id-ctrl /dev/nvme0 | grep -i firmware
```

#### PCIe 链路信息

```bash
# 查看 NVMe 设备的 PCIe 链路与速度  
sudo lspci | grep -i nvme  
sudo lspci -vvv -s $(lspci | grep -i nvme | awk '{print $1}') | grep -E "LnkCap|LnkSta"  
  
# 典型输出解读：  
# LnkCap: Speed 16GT/s, Width x4  → 支持 Gen4 x4  
# LnkSta: Speed 16GT/s, Width x4  → 当前运行在 Gen4 x4
```

#### SMART 与通用健康信息

```bash
# smartctl（支持 SATA 和 NVMe）  
sudo smartctl -a /dev/nvme0  
sudo smartctl -H /dev/nvme0   # 仅健康状态  
  
# 关键字段  
# Temperature:                    当前温度  
# Available Spare:                备用块百分比  
# Percentage Used:                已消耗寿命百分比  
# Data Units Written:             总写入量  
# Media and Data Integrity Errors: 数据完整性错误
```

#### I/O 统计与实时监控

```bash
# 实时 I/O 统计（需安装 sysstat）  
iostat -x 1 /dev/nvme0n1  
  
# 关键列解读  
# r/s  w/s     每秒读写次数  
# rkB/s wkB/s  每秒读写 KB 数  
# await         平均 I/O 延迟（ms）  
# %util         设备利用率（接近 100% 说明饱和）  
  
# 查看当前 I/O 调度器  
cat /sys/block/nvme0n1/queue/scheduler  
# 推荐 NVMe SSD 使用：[none] 或 mq-deadline  
  
# 查看队列深度  
cat /sys/block/nvme0n1/queue/nr_requests
```

### Windows 命令速查

#### PowerShell 基础信息

```bash
# 查看物理磁盘  
Get-PhysicalDisk  
  
# 查看磁盘详细信息（MediaType 显示 SSD/HDD）  
Get-PhysicalDisk | Select FriendlyName, MediaType, Size, HealthStatus  
  
# 查看分区  
Get-Partition | Select DiskNumber, PartitionNumber, DriveLetter, Size
```

#### smartctl（需安装 smartmontools）

```bash
# 安装  
winget install smartmontools  
  
# 查看 SMART 信息  
smartctl -a /dev/sda  
# 或  
smartctl -a pd0    # 部分 Windows 版本
```

#### 磁盘管理 GUI 替代

* **CrystalDiskInfo**：图形化 SMART 监控，支持 NVMe 温度和健康状态
* **CrystalDiskMark**：图形化性能测试
* **HWiNFO64**：查看 PCIe 链路速度（Gen3/Gen4）、SSD 温度

### 常用命令对照表

| 需求 | Linux | Windows |
| --- | --- | --- |
| 列出磁盘 | `lsblk` | `Get-PhysicalDisk` |
| NVMe 详情 | `nvme id-ctrl /dev/nvme0` | CrystalDiskInfo |
| SMART 健康 | `smartctl -a /dev/nvme0` | `smartctl -a /dev/sda` |
| PCIe 速度 | `lspci -vvv` | HWiNFO64 |
| 实时 I/O | `iostat -x 1` | 任务管理器 → 性能 → 磁盘 |
| 性能测试 | `fio` | CrystalDiskMark / fio |
| TRIM 状态 | `lsblk -D` | `fsutil behavior query DisableDeleteNotify` |
| 固件版本 | `nvme id-ctrl` | 厂商工具 / CrystalDiskInfo |

### 一次完整的 SSD 健康检查流程（Linux）

```bash
#!/bin/bash  
DEV=/dev/nvme0  
  
echo "=== 设备信息 ==="  
nvme list | grep nvme0  
  
echo "=== PCIe 链路 ==="  
lspci -vvv -s $(lspci | grep -i nvme | awk '{print $1}') | grep -E "LnkCap|LnkSta"  
  
echo "=== SMART 健康 ==="  
nvme smart-log $DEV  
  
echo "=== 温度 ==="  
nvme smart-log $DEV | grep temperature  
  
echo "=== 已写入量 ==="  
nvme smart-log $DEV | grep data_units_written  
  
echo "=== 剩余寿命 ==="  
nvme smart-log $DEV | grep percent_used
```

---

## 本章小结

![](images/1d567b6262b9.png)

<p align="center">图 3-6：第 3 章知识要点总览</p>

* **I/O 栈分层**：应用 → VFS/文件系统 → Page Cache → 块层 → 驱动 → SSD，每层职责独立
* **写路径**：数据先写 Page Cache，内核异步刷到 SSD（写回策略）
* **块设备**：SSD 以固定大小块（512 B / 4 KB）随机访问，支持 Page Cache
* **DMA**：设备直接访问内存搬运数据，CPU 无需参与，完成后通过中断或轮询通知
* **异步 I/O**：提交后立即返回，多个 I/O 并行在途，才能充分利用 NVMe 高队列深度
* **实用命令**：Linux 用 `lsblk` + `nvme-cli` + `smartctl`；Windows 用 PowerShell + CrystalDiskInfo

## 常见问题 FAQ

**Q1：Page Cache 和 SSD 上的数据不一致，断电会丢数据吗？**

会。Page Cache 中的「脏页」（已修改但未刷盘）在断电时会丢失。数据库等关键应用必须调用 `fsync()` 确保数据持久化。企业级 SSD 的 PLP（断电保护）只能保护已到达 SSD DRAM 的数据，无法保护还在 Page Cache 中的数据。

**Q2：为什么 NVMe SSD 推荐 I/O 调度器设为 none？**

传统 I/O 调度器（如 cfq）为 HDD 设计，用于合并和排序寻道。NVMe 多队列架构已有足够的并行能力，额外调度反而增加延迟。`none` 调度器将请求直接下发，适合低延迟 NVMe SSD。

**Q3：io\_uring 和 NVMe 原生队列有什么关系？**

两者是不同层次的「队列」：io\_uring 是用户态与内核之间的提交/完成环；NVMe SQ/CQ 是内核驱动与 SSD 之间的命令队列。io\_uring 减少了系统调用开销，NVMe 多队列提升了硬件并行度，两者配合才能发挥最大性能。

**Q4：直接写 /dev/nvme0n1 和写文件有什么区别？**

写文件经过文件系统，有 Page Cache、权限检查、inode 管理；直接写块设备绕过文件系统，数据以原始块形式写入，可用于性能测试但会破坏文件系统结构，生产环境严禁。

**Q5：如何确认 SSD 运行在 PCIe Gen4 而不是 Gen3？**

Linux：`lspci -vvv` 查看 `LnkSta: Speed`，Gen3 为 8GT/s，Gen4 为 16GT/s。Windows：HWiNFO64 查看 PCIe 总线速度。注意：Gen4 盘插入 Gen3 插槽会自动降速，两者需匹配。

## 自测题

**1. 描述一次写操作从应用程序到 SSD 的完整路径（至少列出 5 层）。**

参考答案：

1. 应用程序调用 `write()` 系统调用
2. VFS / 文件系统（如 ext4）处理，数据写入 Page Cache
3. 内核 writeback 机制将脏页刷到块层
4. 块层构造 bio，NVMe 驱动转换为 NVMe Write 命令
5. 命令写入 SQ，SSD 控制器通过 FTL 写入 NAND
6. 完成后写入 CQ，触发中断，驱动唤醒等待线程

**2. 块设备和字符设备的核心区别是什么？SSD 属于哪一类？**

参考答案：

* **块设备**：以固定大小块为单位，支持随机访问（seek），有缓存，典型如 HDD、SSD
* **字符设备**：以字节流为单位，顺序访问，典型如键盘、串口
* SSD 属于 **块设备**，在 Linux 下呈现为 `/dev/nvme0n1`

**3. DMA 的作用是什么？如果没有 DMA，I/O 性能会受到什么影响？**

参考答案：

* DMA 允许设备控制器直接在内存与设备间传输数据，无需 CPU 逐字节搬运
* 无 DMA 时，CPU 在 I/O 期间被完全占用，无法执行其他任务，系统整体吞吐量大幅下降，且无法有效并行处理多个 I/O

**4. 为什么高队列深度的异步 I/O 对 NVMe SSD 特别重要？**

参考答案：

* NVMe 支持每队列最高 65535 个在途命令，硬件并行能力极强
* 同步 I/O 每次只有 1 个在途命令，SSD 大量并行通道闲置
* 异步 I/O + 高队列深度（如 QD128）可同时提交数百个命令，充分利用 SSD 的 IOPS 能力
* fio 测试：iodepth=128 的 IOPS 可达 iodepth=1 的 10–50 倍

**5. 写出 3 条 Linux 命令，分别用于：查看块设备列表、查看 NVMe SMART 健康、查看 PCIe 链路速度。**

参考答案：

```bash
# 1. 查看块设备列表  
lsblk  
  
# 2. 查看 NVMe SMART 健康  
sudo nvme smart-log /dev/nvme0  
  
# 3. 查看 PCIe 链路速度  
sudo lspci -vvv -s $(lspci | grep -i nvme | awk '{print $1}') | grep LnkSta
```

## 延伸阅读

| 资源 | 说明 |
| --- | --- |
| Linux Block Layer 文档 | 内核块层架构 |
| io\_uring 官方 Wiki | Jens Axboe 的 io\_uring 设计论文 |
| SPDK 文档 | 用户态 NVMe 驱动与应用框架 |
| NVMe CLI 工具 | nvme-cli 命令参考 |
| 本书 第 4 章 | 半导体存储器分类，进入 NAND 原理 |
| 本书 第 19 章 | NVMe SQ/CQ 队列机制详解 |
| 本书 第 22 章 | NVMe 多队列与并发模型 |

---

> **下一章预告**：第 4 章 半导体存储器分类—— 进入第二篇「闪存物理基础」。我们将从 DRAM、SRAM、ROM、Flash 的分类出发，理解为什么 SSD 选择 NAND Flash 作为存储介质，为后续 FTL 和 GC 原理打下基础。
>
> 未分享的章节后续更新，欢迎关注一起学习