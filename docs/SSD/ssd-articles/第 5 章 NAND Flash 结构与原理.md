第 5 章 NAND Flash 结构与原理

> **本章目标**：深入理解 NAND Flash 的物理结构与读写擦原理，掌握「为什么不能原地覆写」的根本原因  
> **前置章节**：第 4 章 半导体存储器分类  
> **下一章**：第 6 章 NAND 接口与 ONFI / Toggle 协议

## 章节概述

第 4 章我们知道了 SSD 选择 NAND Flash 作为存储介质。本章将「钻进 NAND 内部」——从存储单元（Cell）的物理原理，到 Die、Plane、Block、Page 的层级结构，再到 Read（读）、Program（写）、Erase（擦）三种操作的物理过程。

这些知识是理解 FTL、垃圾回收、磨损均衡的**物理基础**。读完本章，你将能回答固件工程师最常面对的问题：**「为什么 SSD 不能原地覆写？」「为什么 GC 不可避免？」「P/E 次数耗尽意味着什么？」**

## 5.1 存储单元：SLC / MLC / TLC / QLC / PLC 详解

### 核心概念

NAND Flash 的基本存储单元是 **Floating Gate（浮栅）** 或 **Charge Trap（电荷陷阱）** 晶体管。通过在浮栅中注入或移除电子，改变晶体管的阈值电压（Threshold Voltage），从而表示不同的数据状态。

**关键思想**：一个 Cell 能存储多少 bit，取决于它能区分多少种不同的  状态。

![](images/630e5bce29cd.jpg)

<p align="center">图 5-1：不同 Cell 类型的 bit 数、电压状态数与寿命对比</p>

### 各类型详解

| 类型 | 全称 | 每 Cell bit 数 | 电压状态数 | P/E 寿命（典型） | $/GB（相对） | 主要用途 |
| --- | --- | --- | --- | --- | --- | --- |
| **SLC** | Single-Level Cell | 1 | 2（0/1） | 约 50K–100K | 最高 | 企业级 SSD 缓存区、工业级 |
| **MLC** | Multi-Level Cell | 2 | 4 | 约 3K–10K | 高 | 早期 SSD、企业级 |
| **TLC** | Triple-Level Cell | 3 | 8 | 约 1K–3K | 中 | **消费级 SSD 主流** |
| **QLC** | Quad-Level Cell | 4 | 16 | 约 500–1K | 低 | 大容量消费级、冷数据 |
| **PLC** | Penta-Level Cell | 5 | 32 | 约 300–500 | 最低 | 研发/归档场景 |

### 为什么更多 bit 意味着更短寿命

以 TLC 为例，一个 Cell 需要区分 8 种  状态来表示 3 bit（000–111）。这意味着：

| 因素 | SLC（2 状态） | TLC（8 状态） |
| --- | --- | --- |
| 相邻状态电压间隔 | 宽，容错大 | 窄，容错小 |
| 读取精度要求 | 低 | **高（需 Read Retry）** |
| 写入干扰容忍度 | 高 | 低 |
| 电子泄漏影响 | 小 | **大（数据保持力下降）** |
| ECC 纠错需求 | BCH 即可 | **必须 LDPC** |

**生活类比**：SLC 像在一个大抽屉里放「有/无」两样东西；QLC 像在同一个抽屉里精确隔出 16 个小格——格子越密，放错位置、互相干扰的概率越高。

### SLC Cache：用 SLC 速度掩盖 TLC/QLC 的写入延迟

消费级 TLC/QLC SSD 常用 **SLC Cache** 策略：将一部分 TLC/QLC Cell 以 SLC 模式（只存 1 bit）运行，获得 SLC 级写入速度：

| 阶段 | 行为 | 用户感知速度 |
| --- | --- | --- |
| Cache 未满 | 数据以 SLC 模式写入 Cache 区 | 标称高速（2000–7000 MB/s） |
| Cache 耗尽 | 直写 TLC/QLC，速度骤降 | 100–500 MB/s（真实速度） |

> SLC Cache 策略见 第 2 章 §2.5，WAF 影响见 第 16 章。

## 5.2 物理结构：Die → Plane → Block → Page → Cell

### 层级结构概览

NAND Flash 从大到小分为以下层级，理解这个层级是阅读 NAND 数据手册和固件代码的基础：

![](images/2b947e26eba1.jpg)

<p align="center">图 5-2：NAND Flash 物理层级（Channel → Die → Plane → Block → Page → Cell）</p>

### 各层级详解

| 层级 | 英文 | 说明 | 典型规模（3D TLC NAND） |
| --- | --- | --- | --- |
| **Channel** | Channel | 独立的数据/命令总线，可并行操作 | SSD 通常 4–16 条 |
| **Die** | Die / Target | 一片完整的 NAND 芯片逻辑单元 | 每 Channel 1–8 个 Die |
| **Plane** | Plane | Die 内的并行操作单元 | 每 Die 2–4 个 Plane |
| **Block** | Block / Erase Block | **擦除的最小单位** | 每 Plane 约 1000–4000 个 Block |
| **Page** | Page | **读/写的最小单位** | 每 Block 约 256–4096 页 |
| **Cell** | Cell | 最小存储单元（1–5 bit） | 每 Page 约 16–32 KB 数据 |

### 典型容量计算示例

```
1 个 Page  = 16 KB 用户数据（+ 约 1.5 KB 备用区用于 ECC）  
1 个 Block = 256 Pages × 16 KB = 4 MB  
1 个 Plane = 2048 Blocks × 4 MB = 8 GB  
1 个 Die   = 2 Planes × 8 GB = 16 GB  
1 个 Channel = 4 Dies × 16 GB = 64 GB  
8 Channels = 512 GB（接近 512 GB 标称 SSD 的 raw NAND 容量）
```

> 实际参数因厂商和代际差异很大，上表仅为帮助理解的量级估算。

### Block 与 Page 的关键约束

这是 NAND 最重要的操作约束，务必牢记：

| 操作 | 最小单位 | 操作后状态 |
| --- | --- | --- |
| **读（Read）** | Page | 不改变 Cell 状态 |
| **写（Program）** | Page | 只能将 bit 从 1 改为 0 |
| **擦（Erase）** | **Block（整块）** | 所有 bit 恢复为 1 |

**关键规则**：

* 写（Program）只能执行在**已擦除（全 1）**的 Page 上
* 已写入的 Page **不能再次写入**，除非先擦除整个 Block
* 擦除的单位是整个 Block，不能只擦一个 Page

### 3D NAND 的 String 结构（预习）

2D NAND 中 Cell 在平面上排列；3D NAND 中 Cell 沿垂直方向堆叠成 **NAND String（串）**：

```
Bit Line (BL)  
    |  
  [Cell]  ← 第 N 层  
  [Cell]  ← 第 N-1 层  
  [Cell]  ← ...  
  [Cell]  ← 第 1 层  
    |  
Word Line (WL) 横跨所有 String
```

3D NAND 详细结构见 第 7 章。

## 5.3 读 / 写 / 擦除操作原理（Read / Program / Erase）

### 三种操作的物理本质

NAND Flash 通过控制浮栅中的电子数量来改变晶体管阈值电压 ，三种操作本质上是「感知」「注入」「移除」电子：

![](images/611536e19f47.jpg)

<p align="center">图 5-3：NAND Flash 的 Read / Program / Erase 物理过程</p>

### Read（读）：感知阈值电压

**过程**：

1. 在 Word Line（WL）上施加读取电压
2. 检测 Bit Line（BL）上是否有电流通过
3. 无电流 → Cell 处于低  状态（已编程，bit = 0）
4. 有电流 → Cell 处于高  状态（已擦除，bit = 1）

| 参数 | 典型值 |
| --- | --- |
| 读取延迟 | 约 25–80 μs（取决于 SLC/TLC/QLC） |
| 是否改变 Cell 状态 | **否** （只读不改） |
| 最小单位 | Page |

**TLC/QLC 的 Read 复杂性**：需要多次施加不同电压，逐步确定 Cell 处于 8/16 种状态中的哪一种（**Sequential Read** 或 **Snapshot Read**）。

### Program（写/编程）：注入电子

**过程**：

1. 在 Control Gate 和 Channel 之间施加高电压（约 15–20 V）
2. 通过 **Fowler-Nordheim 隧穿（FN Tunneling）**，电子从 Channel 注入浮栅
3. 浮栅积累负电荷 →  升高 → bit 从 1 变为 0

| 参数 | 典型值 |
| --- | --- |
| 编程延迟 | 约 200–800 μs/Page |
| 只能写入 | **0** （不能将 0 改回 1） |
| 前提条件 | Page 必须处于擦除态（全 1） |
| 最小单位 | Page |

**重要**：Program 是**单向操作**——只能写 0，不能写 1。要将 0 变回 1，必须 Erase 整个 Block。

### Erase（擦除）：移除电子

**过程**：

1. 在 Control Gate 施加负电压（约 -10 V），Substrate 施加正电压
2. 浮栅中的电子通过 FN 隧穿被拉出
3. 浮栅失去电荷 →  降低 → 所有 bit 恢复为 1

| 参数 | 典型值 |
| --- | --- |
| 擦除延迟 | 约 1–5 ms/Block |
| 操作结果 | Block 内所有 Page 恢复为全 1 |
| 最小单位 | **Block（不能单 Page 擦除）** |
| 消耗 P/E 寿命 | **是（每擦一次计 1 次 P/E）** |

### 三种操作对比表

| 操作 | 英文 | 单位 | 延迟（典型） | 改变 Cell | 前提条件 |
| --- | --- | --- | --- | --- | --- |
| 读 | Read | Page | 25–80 μs | 否 | 无 |
| 写 | Program | Page | 200–800 μs | 是（1→0） | Page 已擦除 |
| 擦 | Erase | Block | 1–5 ms | 是（→1） | 无 |

### 操作时序与并行

现代 SSD 通过多 Channel、多 Die、多 Plane 并行来提升吞吐：

```
Channel 0: Die0 Plane0 Program Page X  ─┐  
Channel 1: Die1 Plane0 Read Page Y      ├─ 同时进行  
Channel 2: Die2 Plane1 Erase Block Z   ─┘  
Channel 3: Die3 Plane0 Program Page W  ─┘
```

**Multi-Plane Operation**：同一 Die 的两个 Plane 可以同时执行 Read 或 Program，吞吐量翻倍。

## 5.4 为什么不能原地覆写（Out-of-Place Update）

### 根本约束

NAND Flash 有一条不可违背的物理规则：

> **Program 只能将 bit 从 1 变为 0，不能将 0 变为 1。要将 0 恢复为 1，必须 Erase 整个 Block。**

这意味着：**已写入数据的 Page 不能直接修改**，即使只改 1 个字节也不行。

### 原地覆写为何不可行

![](images/214d3cde4fda.jpg)

<p align="center">图 5-4：NAND 不能原地覆写，FTL 采用 Out-of-Place Update 写入新 Page</p>

**场景**：Block 100 的 Page 5 存储了文件 A 的数据，现在要修改文件 A 的一个字段。

| 步骤 | 操作 | 问题 |
| --- | --- | --- |
| 1 | 直接 Program Page 5 的新数据 | Page 5 已有数据（0/1 混合），无法直接写入 |
| 2 | 先 Erase Block 100 | Block 100 中其他 Page 的有效数据也被擦除！ |
| 3 | 再 Program Page 5 | 其他 Page 数据丢失 |

**结论**：Erase 的最小单位是 Block，无法只擦一个 Page——这就是原地覆写不可能的根本原因。

### Out-of-Place Update（异地更新）

SSD 固件（FTL）的解决方案：**新数据写到新的空闲 Page，旧 Page 标记为无效，更新映射表**。

```
主机写入 LBA 100 的新数据：  
  
1. FTL 查找 LBA 100 当前映射 → PBA（Block 50, Page 10）  
2. 在空闲 Block 200 的 Page 3 写入新数据（Program）  
3. 将 LBA 100 的映射更新为（Block 200, Page 3）  
4. 将旧 Page（Block 50, Page 10）标记为 Invalid  
5. 当 Block 50 中无效 Page 足够多时，GC 擦除 Block 50
```

| 概念 | 说明 |
| --- | --- |
| **In-Place Update** | 在原位置修改（NAND 不支持） |
| **Out-of-Place Update** | 写新位置，旧位置标记无效（FTL 的核心策略） |
| **Write Amplification** | 因 Out-of-Place 导致实际写入量 > 主机写入量 |

> FTL 映射表（L2P）详解见 第 12 章，GC 如何回收无效 Page 见 第 13 章。

### 生活类比

NAND Block 像一本**只能整本撕掉重写的笔记本**：

* 写（Program）= 在空白页上写字（只能写，不能改）
* 擦（Erase）= 把整本笔记本所有页清空
* 修改内容 = 在新空白页上重写，旧页划掉（标记无效）
* 等划掉的页够多了，整本笔记本撕掉（GC Erase Block）

## 5.5 P/E Cycle（擦写次数）与数据保持时间（Retention）

### P/E Cycle 是什么

**P/E Cycle（Program/Erase Cycle）** 指一个 Block 从擦除态 → 编程 → 再擦除的完整循环次数。每次 Erase 计 1 次 P/E。

| Cell 类型 | 额定 P/E 次数（典型） | 实际影响 |
| --- | --- | --- |
| SLC | 约 50K–100K | 企业级 SSD 缓存区 |
| MLC | 约 3K–10K | 早期企业级 |
| TLC | 约 1K–3K | 消费级 SSD 主流 |
| QLC | 约 500–1K | 大容量消费级 |
| PLC | 约 300–500 | 研发/只读场景 |

### P/E Cycle 耗尽后会发生什么

Block 的 P/E 次数接近上限时：

1. **分布漂移**：浮栅电荷泄漏加剧，状态难以区分
2. **Raw Bit Error Rate（RBER）上升**：读取时出现更多 bit 翻转
3. **ECC 压力增大**：需要更强的 LDPC 纠错，Read Retry 次数增加
4. **Block 退役**：RBER 超过 ECC 纠正能力时，固件将其标记为坏块

SSD **不会突然全部失效**——固件通过磨损均衡（Wear Leveling）和 OP 备用块，在 Block 退役时用备用块替换，直到备用块耗尽。

### 数据保持时间（Data Retention）

**Data Retention** 指 NAND Cell 在**无电源**情况下保持数据不丢失的时间。

![](images/bf7209e7042c.jpg)

<p align="center">图 5-5：P/E Cycle 消耗与 Data Retention 的权衡关系</p>

| 因素 | 对 Retention 的影响 |
| --- | --- |
| **P/E 次数** | 次数越多，浮栅损伤越大，Retention 越短 |
| **温度** | 每升高 10°C，Retention 约减半（Arrhenius 模型） |
| **Cell 类型** | SLC > MLC > TLC > QLC（状态间隔越宽，保持越好） |
| **时间** | 随时间推移，浮栅电子持续泄漏 |

**JEDEC 标准参考**（TLC NAND，25°C）：

| 状态 | Retention 要求 |
| --- | --- |
| 出厂（P/E = 0） | 数据保持 10 年 |
| 额定 P/E 寿命耗尽后 | 数据保持 1 年 |

**实际工程影响**：

* 企业级 SSD 在 40°C 环境下要求 3 个月 Retention
* 长期归档数据需考虑 Retention 衰减，定期「刷新」（Read-Refresh）
* 高温环境（车载、工业）需更保守的 P/E 管理和温控

### TBW 与 P/E Cycle 的关系

```
TBW ≈ 总 NAND 容量 × 平均 P/E 次数 × OP 系数  
  
示例（1 TB TLC SSD）：  
  Raw NAND 容量 ≈ 1.07 TB（7% OP）  
  平均 P/E ≈ 2000 次  
  TBW ≈ 1.07 TB × 2000 ≈ 2140 TB（理论值，厂商标称通常更保守）
```

## 5.6 读干扰、写干扰、数据保持衰减等可靠性问题

### 可靠性问题概览

NAND Flash 在长期使用中面临多种物理干扰，这些干扰是 ECC、Read Retry、Refresh 机制存在的根本原因：

![](images/768861cb6940.jpg)

<p align="center">图 5-6：Read Disturb、Program Disturb 与 Data Retention 衰减</p>

### Read Disturb（读干扰）

**机制**：读取某个 Page 时，同一 Word Line 上其他 Block 的 Cell 也会受到读取电压的影响，导致浮栅中少量电子被注入或移除。

| 参数 | 说明 |
| --- | --- |
| 触发条件 | 对同一 Word Line 读取次数超过阈值（约 10 万–100 万次） |
| 影响 | 相邻 Block 中未读取 Page 的 bit 可能翻转 |
| 后果 | 数据静默损坏（Silent Data Corruption） |
| 应对 | **Read Disturb Refresh** ：固件计数读取次数，超阈值时将数据搬到新 Block |

### Program Disturb（写干扰）

**机制**：Program 某个 Page 时，高电压（15–20 V）施加在 Word Line 上，同一 Block 内其他 Page 的 Cell 也会受到干扰。

| 参数 | 说明 |
| --- | --- |
| 触发条件 | 对同一 Block 内 Page 编程时 |
| 影响 | 同 Block 内已编程 Page 的 bit 可能翻转 |
| 应对 | ECC 纠正 + 写入后验证（Program Verify） |

### Data Retention 衰减（数据保持力下降）

**机制**：浮栅中的电子随时间缓慢泄漏， 漂移，导致读取时状态判断错误。

| 场景 | 风险 |
| --- | --- |
| SSD 长期不通电（如库存、备份盘） | 数据可能损坏 |
| Block P/E 次数接近上限 | Retention 大幅缩短 |
| 高温存储环境 | 泄漏加速 |

**应对**：

* 固件定期 **Read Scrub / Read Refresh**：后台读取并刷新接近 Retention 极限的数据
* SMART 监控 **Media Errors** 和 **Available Spare**

### 其他可靠性机制

| 问题 | 机制 | 应对 |
| --- | --- | --- |
| **Wear Out** | P/E 次数耗尽，Block 不可靠 | 磨损均衡 + 坏块替换 |
| **Power Loss** | 意外断电时正在写入的数据丢失 | PLP 电容 + 映射表持久化 |
| **Cell-to-Cell Interference** | 3D NAND 相邻 String 互相干扰 | 厂商专有算法 + LDPC |
| **Temperature Effects** | 高温加速泄漏，低温影响 | 温度传感器 + 温控降速 |

### ECC 的角色

NAND 的 Raw Bit Error Rate（RBER）随 P/E 和使用时间持续上升，**没有 ECC，NAND 数据不可靠**：

| ECC 类型 | 纠正能力 | 适用场景 |
| --- | --- | --- |
| **BCH** | 约 40–80 bit/512 B | 早期 SLC/MLC |
| **LDPC** | 约 100–200 bit/4 KB | **现代 TLC/QLC 标配** |

> ECC 与 Read Retry 详解见 第 6 章 §6.5 和 第 15 章。

---

## 本章小结

![](images/4e587ea7adcf.jpg)

<p align="center">图 5-7：第 5 章知识要点总览</p>

* **Cell 类型**：SLC（1 bit，寿命长）→ TLC（3 bit，主流）→ QLC（4 bit，高密度），bit 越多密度越高但寿命越短
* **物理层级**：Channel → Die → Plane → Block → Page → Cell；**读按 Page，写按 Page，擦按 Block**
* **Read/Program/Erase**：读感知 ；写注入电子（1→0）；擦移除电子（→全 1）
* **不能原地覆写**：Program 只能写 0，Erase 最小单位是 Block → FTL 采用 Out-of-Place Update
* **P/E Cycle**：每 Erase 一次计 1 次，耗尽后 Block 退役；与 TBW 直接相关
* **Retention**：断电后数据保持时间，随 P/E 和温度衰减
* **可靠性**：Read Disturb、Program Disturb、Retention 衰减 → 需 ECC、Read Refresh、Wear Leveling

---

## 常见问题 FAQ

**Q1：为什么 Erase 比 Program 慢一个数量级？**

Erase 需要对整个 Block（256–4096 页）的所有 Cell 同时移除浮栅电子，电压高、时间长（1–5 ms）；Program 只写一页（200–800 μs）。Erase 还是 P/E 寿命的计数单位，频率远低于 Program。

**Q2：读取操作会损耗 NAND 寿命吗？**

正常读取不消耗 P/E 寿命，但大量读取同一 Word Line 会触发 Read Disturb，可能导致相邻 Cell 数据损坏。固件通过 Read Disturb Refresh 应对，这不是 P/E 消耗，但需要额外的 Program/Erase 操作。

**Q3：3D NAND 和 2D NAND 在操作粒度上有区别吗？**

操作粒度相同（Page 读写，Block 擦除）。3D NAND 的差异在于 Cell 沿垂直方向堆叠（String 结构），密度更高，但 Cell 间干扰（Interference）更复杂，需要更强的 ECC 和厂商专有算法。

**Q4：SSD 出厂时的「全盘写入 0」是 Erase 还是 Program？**

通常是 Program（将全 0 写入所有 Page），不是 Erase。出厂时 NAND 已处于擦除态（全 1），Program 写入 0 后呈现「空白」状态。Secure Erase 才是真正的 Block Erase。

**Q5：如何理解「1 个 P/E Cycle = 1 次 Block Erase」？**

P/E Cycle 以 Erase 为计数单位，因为 Erase 对浮栅的损伤最大。一个 Block 被 Erase 后，可以 Program 多次（写多个 Page），但只计 1 次 P/E。固件的 Wear Leveling 确保各 Block 的 P/E 次数均匀增长。

## 自测题

**1. NAND Flash 的 Read、Program、Erase 三种操作的最小单位分别是什么？**

参考答案：

* **Read（读）**：Page（页）
* **Program（写）**：Page（页），且只能将 bit 从 1 改为 0
* **Erase（擦）**：Block（块），将整个 Block 所有 Page 恢复为全 1

**2. 为什么 NAND Flash 不能原地覆写？FTL 如何解决？**

参考答案：

* Program 只能写 0 不能写 1；要将 0 恢复为 1 必须 Erase 整个 Block
* Erase 会清除 Block 内所有 Page 的有效数据，因此无法原地修改单个 Page
* FTL 采用 **Out-of-Place Update**：新数据写到新 Page，旧 Page 标记 Invalid，更新 LBA→PBA 映射表，后续 GC 回收无效 Page

**3. SLC 和 QLC 在 P/E 寿命和成本上的差异是什么？为什么消费级 SSD 多用 TLC？**

参考答案：

* **SLC**：1 bit/Cell，P/E 约 50K–100K 次，$/GB 最高，用于企业级缓存
* **QLC**：4 bit/Cell，P/E 约 500–1K 次，$/GB 最低，用于大容量冷数据
* **TLC** 是平衡点：P/E 约 1K–3K 次对消费级足够，$/GB 适中，配合 SLC Cache 和 LDPC ECC 可满足日常使用

**4. 什么是 Read Disturb？固件如何应对？**

参考答案：

* **Read Disturb**：反复读取同一 Word Line 时，读取电压对相邻 Block 的 Cell 产生干扰，导致浮栅电子泄漏，bit 翻转
* **应对**：固件对每个 Block/Page 计数读取次数，超过阈值（约 10 万–100 万次）时，将数据 **Read Disturb Refresh** 搬到新 Block，旧 Block 标记无效等待 GC

**5. 1 TB TLC SSD 的 TBW 标称 600 TB，如何从 P/E 次数角度理解这个数值？**

参考答案：

```
Raw NAND 容量 ≈ 1.07 TB（含约 7% OP）  
TLC P/E 次数 ≈ 2000–3000 次  
理论 TBW ≈ 1.07 TB × 2000 ≈ 2140 TB  
  
厂商标称 600 TB 是保守值，考虑了：  
- 磨损均衡不完美（部分 Block 消耗更快）  
- ECC 开销和坏块替换  
- 5 年保修期的安全边际
```

## 延伸阅读

| 资源 | 说明 |
| --- | --- |
| ONFI 规范 | Open NAND Flash Interface 标准 |
| JEDEC JESD218/219 | SSD 可靠性测试标准 |
| Flash Memory Summit 技术演讲 | NAND 物理原理与 3D 演进 |
| 本书 第 6 章 | NAND 接口协议与 ECC 入门 |
| 本书 第 7 章 | 3D NAND String 结构与堆叠原理 |
| 本书 第 12 章 | FTL 如何应对 Out-of-Place Update |
| 本书 第 13 章 | GC 如何回收无效 Page |

---

> **下一章预告**：第 6 章 NAND 接口与 ONFI / Toggle 协议—— 离开物理原理，进入工程接口。我们将讲解 SSD 控制器如何与 NAND 芯片通信：ONFI/Toggle 协议、命令集、ECC 需求，以及固件工程师如何调试 NAND 后端。