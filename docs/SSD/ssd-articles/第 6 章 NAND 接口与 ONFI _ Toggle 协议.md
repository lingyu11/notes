第 6 章 NAND 接口与 ONFI / Toggle 协议

# 第 6 章 NAND 接口与 ONFI / Toggle 协议

> **本章目标**：理解 SSD 控制器如何与 NAND 芯片通信，掌握命令时序、状态寄存器与 ECC 基础  
> **前置章节**：第 5 章 NAND Flash 结构与原理  
> **下一章**：第 7 章 3D NAND 与现代闪存技术  
> **难度标记**：🔧 面向固件/嵌入式开发人员

---

## 章节概述

第 5 章我们理解了 NAND 的物理结构与 Read/Program/Erase 原理。本章进入**工程接口层**——SSD 控制器（主控）如何通过电信号与 NAND 芯片对话。

NAND 芯片不直接理解「文件」或「LBA」，它只认**命令字节、地址字节和数据字节**。主控的 NAND 后端固件必须严格按照 ONFI 或 Toggle 协议时序发送这些字节，并处理 Ready/Busy 信号和 ECC 纠错。

读完本章，你将理解固件工程师调试 NAND 后端时看到的关键概念：CLE/ALE、命令序列、Status Register、OOB 区与 LDPC。

---

## 6.1 ONFI 与 Toggle Mode 简介

### 核心概念

**ONFI（Open NAND Flash Interface，开放 NAND 闪存接口）** 是由 Intel、Micron、SK hynix 等厂商推动的行业标准，目标是统一 NAND 芯片与控制器之间的接口，使不同厂商的 NAND 可以在同一主控上互换。

**Toggle Mode** 是 Samsung 主导的并行接口规范，与 ONFI 在信号定义和时序上有所不同，但功能等价。现代 SSD 主控通常**同时支持两套协议**，通过配置选择。

![](images/49447777df55.png)

<p align="center">图 6-1：ONFI 与 Toggle Mode 信号与架构对比</p>

### 关键信号线

| 信号 | 全称 | 作用 |
| --- | --- | --- |
| **CE#** | Chip Enable | 选中特定 NAND Die（低有效） |
| **CLE** | Command Latch Enable | 高电平表示 I/O 总线上是命令字节 |
| **ALE** | Address Latch Enable | 高电平表示 I/O 总线上是地址字节 |
| **WE#** | Write Enable | 写脉冲，锁存命令/地址/数据 |
| **RE#** | Read Enable | 读脉冲，从 NAND 读取数据 |
| **DQS** | Data Strobe | 源同步模式的数据选通信号（DDR 模式） |
| **R/B#** | Ready/Busy | 低电平表示 NAND 正在执行操作 |
| **I/O[7:0]** | 数据总线 | 8 bit 双向数据（部分高端 NAND 支持 16 bit） |

### ONFI vs Toggle 主要差异

| 对比项 | ONFI | Toggle Mode |
| --- | --- | --- |
| 主导厂商 | Intel / Micron / SK hynix / Kioxia | Samsung |
| 同步方式 | 源同步（Source Synchronous，DQS） | 源同步（DQS） |
| 数据采样 | DQS 上升/下降沿 | DQS 上升/下降沿 |
| 命令集 | ONFI 标准命令集 | Toggle 专有命令集（功能等价） |
| 兼容性 | 多厂商互换 | 主要用于 Samsung NAND |
| 规范版本 | ONFI 5.x（2020 年代） | Toggle DDR 2.0/3.0 |

> **工程实践**：主控固件在初始化阶段读取 NAND 的 ID 表（Read ID 命令），识别厂商和协议类型，加载对应的时序参数表（Timing Mode Table）。

### 为何主控需适配两套协议

SSD 供应链中，主控厂商（如 Phison、Silicon Motion、Innogrit）需要支持多家 NAND 颗粒厂商的芯片。统一 ONFI 标准降低了适配成本，但 Samsung 作为全球最大 NAND 供应商，其 Toggle 协议无法忽略——因此主流主控均实现双协议支持。

---

## 6.2 异步 / 同步 / NV-DDR 接口速率演进

### 速率演进

NAND 接口速度直接影响 SSD 后端带宽——即使 NAND 内部读取只需 25 μs，如果接口只有 50 MT/s，多 Channel 并行也无法喂饱 PCIe 前端。

![](images/0e482a0708cb.png)

<p align="center">图 6-2：NAND 接口从异步到 NV-DDR3 的速率演进</p>

### 各代接口参数

| 接口模式 | 典型速率 | 8-bit 总线带宽 | 时代 |
| --- | --- | --- | --- |
| **异步（Asynchronous）** | 约 50 MT/s | 约 50 MB/s | 2000 年代早期 |
| **ONFI SDR** | 约 50–90 MT/s | 约 50–90 MB/s | — |
| **ONFI NV-DDR2** | 约 400–800 MT/s | 约 400–800 MB/s | 2010 年代 |
| **Toggle DDR 2.0** | 约 400–800 MT/s | 约 400–800 MB/s | 2010 年代 |
| **ONFI NV-DDR3** | 约 800–2400 MT/s | 约 800–2400 MB/s | 2020 年代 |
| **Toggle DDR 3.0** | 约 800–2400 MT/s | 约 800–2400 MB/s | 2020 年代 |

### 带宽计算

```
NAND 单 Channel 带宽 = 接口速率（MT/s）× 总线位宽（Byte）× 效率系数  
  
示例（NV-DDR3，800 MT/s，8-bit 总线）：  
  理论带宽 = 800 MT/s × 1 Byte = 800 MB/s  
  实际可用约 700–750 MB/s（考虑命令开销）  
  
8 Channel × 750 MB/s ≈ 6 GB/s（可匹配 PCIe Gen4 x4 前端）
```

### 与 PCIe 前端带宽的匹配

| SSD 前端 | 理论带宽 | 所需 NAND 后端（约） |
| --- | --- | --- |
| PCIe Gen3 x4 | 约 4 GB/s | 4–8 Channel × NV-DDR2 |
| PCIe Gen4 x4 | 约 8 GB/s | 8 Channel × NV-DDR3 |
| PCIe Gen5 x4 | 约 16 GB/s | 8–16 Channel × NV-DDR3 + 多 Plane 并行 |

> 后端带宽不足是 SSD 顺序写速度「掉速」的常见原因之一——SLC Cache 耗尽后，后端 NAND 真实写入带宽成为瓶颈。

---

## 6.3 基本命令集：Read / Program / Erase / Reset

### 命令/地址/数据周期

NAND 通过 I/O 总线接收字节流，CLE 和 ALE 信号区分当前字节是命令还是地址：

![](images/2a834e269755.png)

<p align="center">图 6-3：Read / Program / Erase 的命令-地址-数据周期</p>

| 周期类型 | CLE | ALE | 内容 |
| --- | --- | --- | --- |
| **命令周期** | 1 | 0 | 操作码（如 0x00 = Read） |
| **地址周期** | 0 | 1 | 行地址（Page）、列地址、Block 地址 |
| **数据周期** | 0 | 0 | 读/写的数据字节 |
| **空闲** | 0 | 0 | I/O 高阻态 |

### 常用命令 Opcode（ONFI 标准）

| 命令 | Opcode（十六进制） | 功能 |
| --- | --- | --- |
| Read (Page) | 0x00 + 0x30 | 读取一页数据 |
| Program (Page) | 0x80 + 0x10 | 编程（写入）一页 |
| Erase (Block) | 0x60 + 0xD0 | 擦除一个 Block |
| Reset | 0xFF | 复位 NAND 状态 |
| Read ID | 0x90 | 读取 NAND 芯片 ID |
| Get Feature | 0xEE | 读取特性寄存器 |
| Set Feature | 0xEF | 设置特性寄存器 |
| Read Status | 0x70 | 读取状态寄存器 |

> 具体 Opcode 因 NAND 代际和厂商而异，以芯片数据手册为准。

### Read 命令序列（伪代码）

```C
// 读取 Block N, Page M 的数据（简化 ONFI Read 流程）  
void nand_read_page(uint32_t block, uint32_t page, uint8_t *buf)  
{  
    // 发送 Read 命令  
    nand_write_cmd(0x00);           // CLE=1: Read 命令第 1 字节  
  
    // 发送地址（5~6 个地址周期，因 NAND 规格而异）  
    nand_write_addr(col_addr);    // 列地址（页内偏移）  
    nand_write_addr(page_addr);   // 页地址  
    nand_write_addr(block_addr);  // 块地址（高/低位）  
    // ... 更多地址周期（LUN/Die 选择）  
  
    // 发送 Read 确认命令  
    nand_write_cmd(0x30);           // CLE=1: Read 确认  
  
    // 等待 R/B# 变高（或轮询 Status Register）  
    nand_wait_ready();  
  
    // 读取数据（含 OOB/Spare Area）  
    nand_read_data(buf, page_size + oob_size);  
    // 硬件 ECC 引擎自动校验并纠正  
}
```

### Program 命令序列（伪代码）

```C
void nand_program_page(uint32_t block, uint32_t page, uint8_t *data)  
{  
    nand_write_cmd(0x80);           // Program Setup 命令  
    nand_write_addr(/* 列/页/块地址 */);  
    nand_write_data(data, page_size + oob_size);  // 写入数据 + OOB  
    nand_write_cmd(0x10);           // Program Confirm 命令  
    nand_wait_ready();  
  
    // 检查 Status Register 是否有 Program Fail  
    if (nand_read_status() & STATUS_FAIL)  
        handle_program_fail(block, page);  
}
```

### Erase 命令序列（伪代码）

```C
void nand_erase_block(uint32_t block)  
{  
    nand_write_cmd(0x60);           // Erase Setup 命令  
    nand_write_addr(/* 块地址 */);  
    nand_write_cmd(0xD0);           // Erase Confirm 命令  
    nand_wait_ready();              // Erase 耗时 1–5 ms  
  
    if (nand_read_status() & STATUS_FAIL)  
        handle_erase_fail(block);  
}
```

### 多 Die / LUN 选择

一块 SSD 通常有多个 NAND Die 并联在同一 Channel 上，通过 **CE#（Chip Enable）** 和 **LUN（Logical Unit Number）** 地址区分：

```
Channel 0:  
  CE0 → Die 0 (LUN 0, 1)  
  CE1 → Die 1 (LUN 0, 1)  
  CE2 → Die 2 ...  
Channel 1:  
  CE0 → Die 4 ...
```

固件调度器根据 Channel/CE/LUN/Plane/Block/Page 六级地址定位物理位置——这就是 **PBA（Physical Block Address）** 的组成。

---

## 6.4 状态寄存器与 Ready/Busy 信号

### Status Register（SR）位定义

执行 Program/Erase/Read 后，NAND 将结果写入 Status Register，主控通过 Read Status 命令（0x70）读取：

![](images/a68d89413bae.png)

<p align="center">图 6-4：Status Register 关键位定义与 Ready/Busy 流程</p>

| 位 | 名称 | 含义 |
| --- | --- | --- |
| Bit 1 | **FAIL** | 1 = 最近一次 Program/Erase 失败 |
| Bit 5 | **ARDY** | 1 = Erase 失败 |
|  |  |  |
| Bit 6（只读时） | **ARDY** | Array Ready |
| Bit 2 | **OIP** | 1 = 操作进行中（Operation In Progress） |

**OIP 位是最常用的轮询标志**：OIP=1 表示 NAND 还在执行，OIP=0 表示完成。

### 两种等待方式

| 方式 | 原理 | 适用场景 |
| --- | --- | --- |
| **R/B# 引脚** | 硬件信号，NAND 忙时拉低，完成后拉高 | 硬件中断驱动，低延迟 |
| **轮询 Status Register** | 软件循环读 SR，检查 OIP 位 | 简单实现，调试常用 |

```C
// 轮询等待 NAND 就绪（固件中最常见的写法）  
void nand_wait_ready(void)  
{  
    uint8_t status;  
    do {  
        nand_write_cmd(0x70);       // Read Status  
        status = nand_read_byte();  
    } while (status & 0x01);        // OIP bit (bit 0) = 1 表示忙  
}
```

### 典型错误处理

| 错误 | SR 标志 | 固件应对 |
| --- | --- | --- |
| **Program Fail** | P\_FAIL=1 | 标记 Page 无效，重试或标记 Block 为坏块 |
| **Erase Fail** | E\_FAIL=1 | 标记 Block 为坏块，从 BBT 替换 |
| **Read UECC** | ECC 不可纠正 | Read Retry → 若仍失败则标记 Page 不可读 |
| **Timeout** | OIP 长时间为 1 | Reset NAND（0xFF），重新初始化 |

> 坏块管理（BBM）详见 第 15 章。

---

## 6.5 ECC 需求：Raw Bit Error Rate 与 BCH / LDPC 入门

### 为什么 NAND 必须依赖 ECC

NAND Cell 在 P/E 循环、Read Disturb、Retention 衰减等因素下，读取时会出现 **bit 翻转**。**Raw Bit Error Rate（RBER，原始位错误率）** 随使用时间上升：

| NAND 状态 | 典型 RBER（每 Page） |
| --- | --- |
| 出厂（P/E=0） | < 10 bit / 512 B |
| 额定 P/E 寿命后 | 约 40–100+ bit / 512 B |
| TLC/QLC 高 P/E 区 | 可达 200+ bit / 4 KB |

没有 ECC，NAND 数据在出厂后短时间内就会出错——**ECC 是 NAND 存储可靠性的基石**。

![](images/ccd8b3b8cc9d.png)

<p align="center">图 6-5：BCH 与 LDPC 纠错能力与应用场景对比</p>

### BCH vs LDPC

| 对比项 | BCH | LDPC |
| --- | --- | --- |
| 全称 | Bose-Chaudhuri-Hocquenghem | Low-Density Parity-Check |
| 纠正能力 | 约 40–80 bit / 512 B | 约 100–200 bit / 4 KB |
| 解码复杂度 | 低（硬件简单） | 高（需迭代解码） |
| 适用 NAND | SLC、早期 MLC | **TLC、QLC 标配** |
| 现状 | 逐步淘汰 | 主流 SSD 标配 |

**LDPC 的优势**：TLC/QLC 的 RBER 远超 BCH 纠正能力，LDPC 通过迭代软判决解码，纠正能力是 BCH 的 3–5 倍，是现代 SSD 不可或缺的技术。

### Page + OOB（Spare Area）布局

ECC 校验码存储在每页的 **OOB（Out-Of-Band，备用区）** 中，与用户数据分开：

```
┌─────────────────────────────────────────────────────────┐  
│  用户数据区（Main Area）          │  OOB / Spare Area   │  
│  4096 B（4 KB Page）              │  约 480 B           │  
│  [Data Byte 0 ... Data Byte 4095] │ [ECC][BBM][FTL Meta]│  
└─────────────────────────────────────────────────────────┘  
  
512 B Page 时代（legacy）：  
  512 B 用户数据 + 16 B OOB  
  
4 KB Page 时代（现代）：  
  4096 B 用户数据 + 480 B OOB（约 40 B/512 B 段 ECC）
```

OOB 区除 ECC 外，还可能存储：

* **Bad Block Marker（BBM）**：标记出厂坏块
* **FTL 元数据**：部分 FTL 将映射表片段存在 OOB
* **Wear Leveling 计数**：块擦写次数

### ECC 在 SoC 中的位置

```
NAND I/O → DMA → ECC 解码引擎 → 数据 RAM  
                  ↑  
            LDPC 硬解码器  
            （XOR / 迭代逻辑）
```

现代 SSD 主控 SoC 内置硬件 LDPC 引擎，对固件透明——固件只需配置 ECC 强度（Correction Capability），硬件自动编解码。固件工程师需关注 **UECC（Uncorrectable ECC Error）** 事件的处理。

### ECC 强度与 NAND 寿命的关系

| ECC 强度 | 含义 | 影响 |
| --- | --- | --- |
| 120 bit / 4 KB | 可纠正 120 个 bit 翻转 | 允许 NAND 在更高 P/E 下仍可靠 |
| 更高 ECC | 更多冗余 bit | OOB 占用更大，有效容量略减 |

更强的 ECC 允许使用 P/E 寿命更高的 QLC 颗粒，但也意味着控制器需要更复杂的 LDPC 硬件——这是 SSD 主控芯片的核心竞争力之一。

> ECC 深入与 Read Retry 见 第 15 章，NAND 后端固件开发见 第 29 章。

---

## 本章小结

![](images/387f1bd8bd80.png)

<p align="center">图 6-6：第 6 章知识要点总览</p>

* **ONFI** 是开放标准（Intel/Micron/Kioxia 等），**Toggle** 是 Samsung 专有协议，主控通常双协议支持
* 关键信号：**CLE**（命令）、**ALE**（地址）、**CE#**（选 Die）、**DQS**（同步时钟）、**R/B#**（忙信号）
* 接口速率从异步 ~50 MT/s 演进到 **NV-DDR3 ~2400 MT/s**，需与 PCIe 前端带宽匹配
* 命令序列：**命令字节 → 地址字节 → 数据字节**，Read/Program/Erase 各有固定 Opcode 流程
* **Status Register**：OIP 位表示忙/就绪，FAIL 位表示 Program/Erase 失败
* **ECC**：RBER 随 P/E 上升，TLC/QLC 必须 LDPC；OOB 区存储 ECC 校验码

---

## 常见问题 FAQ

**Q1：ONFI 和 Toggle 的命令 Opcode 一样吗？**

不完全一样，但功能等价。例如 Read 命令在 ONFI 中是 0x00+0x30，Toggle 可能有不同的 Setup 字节，但「发命令→发地址→等待→读数据」的流程相同。主控固件通过 NAND ID 识别协议，加载对应命令表。

**Q2：为什么 Read 命令也需要等待 R/B#？**

NAND 收到 Read 命令后，需要将 Page 数据从 NAND 阵列加载到内部 Cache Register（称为 Page Read 阶段），这个过程约 25–80 μs。R/B# 在 Cache Register 加载完成后才变高，之后才能通过 I/O 总线读取数据。

**Q3：OOB 区的数据主机能看到吗？**

通常不能。主机通过 NVMe 读写的是用户数据区（4 KB），OOB 由 SSD 控制器内部管理，对主机透明。只有在工厂测试或固件调试模式下，才能通过特殊命令访问 OOB。

**Q4：LDPC 解码失败（UECC）后数据还能恢复吗？**

硬件 LDPC 失败后，固件会尝试 **Read Retry**（调整读取电压阈值重读），若仍失败则报告 UECC。数据可能永久丢失——这就是为什么 SSD 需要 PLP（断电保护）和定期 Read Refresh，以及为什么重要数据需要备份。

**Q5：一个 Channel 上可以挂多少个 Die？**

通常 2–8 个 Die（通过 CE# 区分），具体取决于主控芯片设计和 PCB 布线。更多 Die 意味着同一 Channel 上可以并行操作更多 Plane，提升吞吐量。

---

## 自测题

**1. CLE 和 ALE 信号的作用分别是什么？**

参考答案：

* **CLE（Command Latch Enable）= 1**：I/O 总线上的字节被 NAND 解释为**命令字节**（Opcode）
* **ALE（Address Latch Enable）= 1**：I/O 总线上的字节被 NAND 解释为**地址字节**（Page/Block/Die 地址）
* 两者均为 0 时，I/O 总线为**数据字节**（读/写数据）

**2. 描述 NAND Page Read 的完整命令序列（至少 5 步）。**

参考答案：

1. 发送 Read 命令（0x00，CLE=1）
2. 发送地址字节（列地址、页地址、块地址，ALE=1）
3. 发送 Read 确认命令（0x30，CLE=1）
4. 等待 R/B# 变高（或轮询 SR 的 OIP 位 = 0）
5. 从 I/O 总线读取数据（含 OOB），硬件 ECC 引擎自动校验纠正

**3. Status Register 的 OIP 位和 FAIL 位分别表示什么？**

参考答案：

* **OIP（Operation In Progress，bit 0）= 1**：NAND 正在执行操作（忙）；= 0：操作完成（就绪）
* **FAIL（bit 0，在操作完成后）= 1**：最近一次 Program 或 Erase 操作失败；= 0：操作成功

**4. 为什么 TLC/QLC SSD 必须使用 LDPC 而不是 BCH？**

参考答案：

* TLC/QLC 的 RBER（原始位错误率）在 P/E 循环后可达 100–200+ bit / 4 KB
* BCH 纠正能力约 40–80 bit / 512 B（约 320 bit / 4 KB 的理论上限），实际余量不足
* LDPC 纠正能力约 100–200 bit / 4 KB，通过迭代软判决解码，能可靠纠正 TLC/QLC 的高 RBER

**5. OOB（Spare Area）的作用是什么？**

参考答案：

* 存储 **ECC 校验码**，供硬件 ECC 引擎解码时使用
* 存储 **Bad Block Marker（BBM）**，标记出厂坏块
* 可能存储 **FTL 元数据**（如映射表片段、擦写计数）
* 对主机不可见，由 SSD 控制器内部管理

---

## 延伸阅读

| 资源 | 说明 |
| --- | --- |
| ONFI 规范 | Open NAND Flash Interface 官方规范（需注册下载） |
| JEDEC JESD230 | Universal Flash Storage（UFS）相关标准 |
| SNIA NAND Flash 101 | 闪存基础与 ECC 概述 |
| 本书 第 7 章 | 3D NAND 结构与现代闪存技术 |
| 本书 第 15 章 | ECC 深入与 Read Retry |
| 本书 第 29 章 | NAND 后端固件调度实现 |

---

> **下一章预告**：第 7 章 3D NAND 与现代闪存技术—— 2D NAND 已接近物理极限，3D 堆叠如何突破密度瓶颈？Charge Trap 与 Floating Gate 有何区别？各 NAND 厂商的技术路线有何差异？