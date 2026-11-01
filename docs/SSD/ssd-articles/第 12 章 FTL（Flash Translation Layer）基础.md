第 12 章 FTL（Flash Translation Layer）基础

> **本章目标**：理解 FTL 的核心职责——LBA 到 PBA 的地址映射，以及 Host Read/Write 的完整流程  
> **前置章节**：第 10 章 SSD 固件的分层架构  
> **下一章**：第 13 章 垃圾回收  
> **难度标记**：🔧 面向固件/嵌入式开发人员

## 章节概述

FTL（Flash Translation Layer）是 SSD 固件的**核心中间层**，位于 NVMe 前端与 NAND 后端之间。主机看到的是连续的 LBA 地址空间；NAND 物理介质却是 Block/Page 结构，且**不能原地覆写**。

FTL 的职责就是在这两种地址模型之间建立映射，并管理空闲空间、触发 GC/WL。本章聚焦 FTL 最基础的问题：**映射表是什么、存在哪里、读写怎么走**。

---

## 12.1 为什么需要 FTL：LBA 到 PBA 的映射

![](images/757c9cdba7a4.jpg)

<p align="center">图 12-1：主机 LBA 与 NAND PBA 之间的 FTL 映射关系</p>

### 两套地址体系

| 地址类型 | 全称 | 视角 | 特点 |
| --- | --- | --- | --- |
| **LBA** | Logical Block Address | 主机 | 连续、512 B 或 4 KB 粒度 |
| **PBA** | Physical Block Address | NAND | Block + Page + Plane + Die 等物理坐标 |

主机假设存储是「可随机覆写的扇区数组」；NAND reality 是：

* **Program** 只能把 1 写为 0，最小写入单位是 **Page**
* **Erase** 最小单位是 **Block**（约毫秒级）
* 已写入的 Page **不能原地修改**

因此 SSD 必须有一层软件（FTL）把 LBA 翻译为当前有效的 PBA，并在数据更新时写到**新的空闲 Page**，旧 Page 标记无效。

### Out-of-Place Update 回顾

```
Host 写 LBA 100（第二次）  
  → FTL 查 L2P：LBA 100 当前在 Block 50 Page 10  
  → 分配新位置 Block 80 Page 5  
  → Program 新数据到 Block 80 Page 5  
  → 更新 L2P：LBA 100 → Block 80 Page 5  
  → Block 50 Page 10 标记 Invalid（等待 GC 回收）
```

> NAND 物理限制详见 第 5 章 §5.4。

---

## 12.2 地址映射表（L2P / P2L）数据结构

![](images/a08d447a5a4e.jpg)

<p align="center">图 12-2：L2P（正向映射）与 P2L（反向映射）的关系</p>

### L2P：Logical to Physical

**L2P** 是 FTL 最核心的数据结构：给定 LBA，查出数据当前存放在哪个 PBA。

| 映射粒度 | 说明 | 适用 |
| --- | --- | --- |
| **页级映射（Page-level）** | 每个 LBA 对应一个 PBA | 随机写性能最优，表最大 |
| **块级映射（Block-level）** | 一个 LBA 块对应一个 PBA 块 + 页内偏移 | 表小，随机写需 RMW |
| **混合映射** | 热数据页级 + 冷数据块级 | 主流大容量方案 |

### P2L：Physical to Logical

**P2L** 是反向映射：给定 PBA（某 Page），查出它当前对应哪个 LBA（或标记 Free/Invalid）。

| 用途 | 说明 |
| --- | --- |
| **GC** | 扫描 Block 时，判断每个 Page 是否仍有效 |
| **WL** | 搬迁数据时，知道哪些 LBA 需要更新 L2P |
| **断电恢复** | 从 NAND 重建映射表 |

### 典型数据结构（伪代码）

```c
/* 页级 L2P 表示例（简化） */  
typedef struct {  
    uint32_t block_id;   /* Block 号 */  
    uint16_t page_id;    /* Page 号 */  
    uint8_t  plane;      /* Plane 号 */  
    uint8_t  valid;      /* 映射是否有效 */  
} pba_t;  
  
pba_t l2p_table[MAX_LBA_COUNT];  /* DRAM 或 HMB 中 */  
  
/* P2L：每个 Block 的 Page 反向映射 */  
uint32_t p2l[block_count][pages_per_block];  /* 值 = LBA 或 INVALID */
```

### 元数据与 OOB

部分 FTL 将映射表片段、P/E 计数等**元数据**存在 Page 的 OOB（Out-of-Band）区，与用户数据一起 Program。

---

## 12.3 映射表存储：DRAM 映射 vs 混合映射 vs 全闪存映射

![](images/a5597257f229.jpg)

<p align="center">图 12-3：全 DRAM / 混合 / HMB / 全 Flash 映射方案对比</p>

### 四种主流方案

| 方案 | 映射表位置 | 优点 | 缺点 |
| --- | --- | --- | --- |
| **全 DRAM 映射** | 板载 DRAM | 随机读写延迟最低 | 需 DRAM，成本高 |
| **HMB 映射** | 借用主机内存 | 降成本 | 占用主机 RAM，延迟略高 |
| **混合映射** | 热区 DRAM + 冷区 Flash | 平衡容量与性能 | 实现复杂 |
| **全 Flash 映射** | 映射表存 NAND | 可无 DRAM（DRAM-less） | 随机写需频繁读表，性能受限 |

### 容量估算（页级映射，约）

```
1 TB SSD，4 KB LBA 粒度  
→ LBA 数量 ≈ 1 TB / 4 KB = 268,435,456 条  
→ 每条 PBA 约 4–8 B  
→ 映射表 ≈ 1–2 GB DRAM
```

大容量盘（4 TB+）全页级映射对 DRAM 需求显著，因此厂商普遍采用**混合映射**或 **HMB**。

### DRAM-less 盘

无板载 DRAM 的消费级盘，映射表存 Flash，写缓冲也小——依赖 **SLC Cache** 和 **顺序写检测** 维持可用性能（见 第 16 章）。

---

## 12.4 写入流程：Host Write → 映射更新 → NAND Program

![](images/c1621ca729d5.jpg)

<p align="center">图 12-4：Host Write 从 NVMe 到 NAND 的逐步流程</p>

| 步骤 | 模块 | 动作 |
| --- | --- | --- |
| 1 | **FE** | 接收 NVMe Write，解析 LBA 范围与 PRP/SGL 数据地址 |
| 2 | **DMA** | 主机数据 DMA 到 SSD DRAM 写缓冲 |
| 3 | **FTL** | 为每个 LBA 分配空闲 PBA（从 Free Block 的下一空闲 Page） |
| 4 | **FTL** | 若 LBA 已有旧映射，旧 PBA 对应 Page 标记 Invalid |
| 5 | **BE** | 构造 NAND Program 命令，数据 + ECC 从 DRAM DMA 到 NAND |
| 6 | **ECC** | 硬件 LDPC 编码，ECC 写入 OOB |
| 7 | **FTL** | Program 成功后更新 L2P（及 P2L） |
| 8 | **FE** | 写 CQ Entry，MSI-X 通知主机完成 |

### 写缓冲的作用

| 机制 | 说明 |
| --- | --- |
| **Write Buffer** | 数据先到 DRAM，合并相邻 LBA 写，减少 NAND Program 次数 |
| **Write Coalescing** | 同一 LBA 多次写只保留最后一次 |
| **SLC Cache** | TLC/QLC 盘先写 SLC 模式 Page，后台 Fold 到 TLC（见第 16 章） |

### 延迟构成（典型消费级 NVMe，约）

| 阶段 | 延迟 |
| --- | --- |
| 写缓冲命中（未刷 NAND） | 约 10–50 µs |
| 实际 NAND Program | 约 100–500 µs（TLC） |
| 含 GC 争用 | 可能升至毫秒级 |

---

## 12.5 读取流程：Host Read → 映射查找 → NAND Read

![](images/0450a1216bbe.jpg)

<p align="center">图 12-5：Host Read 从 L2P 查表到数据返回主机</p>

| 步骤 | 模块 | 动作 |
| --- | --- | --- |
| 1 | **FE** | 接收 NVMe Read，解析 LBA 范围 |
| 2 | **FTL** | 查 L2P：LBA → PBA |
| 3 | **BE** | 向 NFC 下发 NAND Read 命令（指定 Channel/CE/Block/Page） |
| 4 | **NAND** | Page 数据 DMA 到 DRAM |
| 5 | **ECC** | LDPC 解码；失败则触发 Read Retry（见 第 15 章） |
| 6 | **DMA** | 数据从 DRAM DMA 到主机 PRP/SGL 缓冲区 |
| 7 | **FE** | 写 CQ Entry，通知完成 |

### 读路径优化

| 优化 | 说明 |
| --- | --- |
| **L2P Cache** | 热点 LBA 映射缓存在 SRAM |
| **Read Ahead** | 顺序读预取后续 Page |
| **多 Plane Read** | 同一 Die 多 Plane 并行读 |
| **Queue Depth** | 高 QD 下多 Channel 并行，提升 IOPS |

### Trim 对读路径的影响

主机发 **Trim/Discard** 后，FTL 将对应 LBA 映射标记为无效（不立即擦 NAND），读这些 LBA 返回空或零——详见 第 16 章 §16.3。

---

## 本章小结

![](images/04bdedf25b2f.jpg)

<p align="center">图 12-6：第 12 章知识要点总览</p>

* FTL 解决 **LBA（主机视角）与 PBA（NAND 视角）** 的映射问题
* **L2P** 正向查表供读写；**P2L** 反向映射供 GC/WL/Recovery
* 映射表可存 **DRAM / HMB / Flash / 混合**，影响性能与成本
* **写路径**：分配 PBA → Program → 更新 L2P；旧 Page 变 Invalid
* **读路径**：L2P 查表 → NAND Read → ECC → DMA 回主机

---

## 常见问题（FAQ）

**Q1：FTL 映射表丢失会怎样？**

上电时需从 NAND 扫描重建（Power Fail Recovery），耗时长；运行中丢失会导致数据不可寻址——因此 PLP 和 Checkpoint 至关重要（见 第 17 章）。

**Q2：页级映射和块级映射怎么选？**

企业级/高性能盘倾向页级或混合；DRAM-less 低成本盘倾向块级或全 Flash 映射，牺牲随机写性能。

**Q3：HMB 映射和板载 DRAM 映射对 FTL 逻辑有区别吗？**

FTL 算法相同，区别在于映射表物理存放位置与访问延迟；HMB 需 NVMe Get/Set Features 与主机协商内存区域。

---

## 自测题

**1. 为什么 SSD 必须有 FTL？NAND 哪两个特性导致了这一点？**

参考答案：NAND 不能原地覆写（Program 只能写 0，需 Erase 后才能重写）且 Erase 最小单位是 Block。FTL 用 Out-of-Place Update 把新数据写到新 Page，并维护 LBA→PBA 映射。

**2. L2P 和 P2L 分别用于什么场景？**

参考答案：L2P 用于 Host Read/Write 时由 LBA 查 PBA；P2L 用于 GC 扫描 Block 判断 Page 有效性、WL 数据搬迁、以及断电后重建映射表。

**3. 1 TB、4 KB LBA 粒度的页级映射表大约需要多少 DRAM？**

参考答案：约 2.68 亿条映射，每条约 4–8 B，合计约 **1–2 GB**（典型估算）。

**4. Host Write 完成后，旧 PBA 对应的 Page 处于什么状态？何时被回收？**

参考答案：标记为 **Invalid（无效页）**；当所在 Block 无效页足够多时，由 **GC** 迁移有效页并 Erase 该 Block（见第 13 章）。

---

## 延伸阅读

| 资源 | 说明 |
| --- | --- |
| 第 13 章 | 无效页如何被 GC 回收 |
| 第 5 章 §5.4 | Out-of-Place Update 物理根因 |
| 第 28 章 | FTL 开发实战 |
| A Survey of Flash Translation Layer 等论文 | 映射方案学术综述 |

---

> **下一章预告**：第 13 章 垃圾回收——无效页如何产生、GC 何时触发、如何与 Host I/O 仲裁