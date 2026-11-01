第 7 章 3D NAND 与现代闪存技术

> **本章目标**：理解 2D NAND 的物理极限、3D 堆叠原理、Charge Trap 技术，以及 QLC/PLC 的工程权衡  
> **前置章节**：第 6 章 NAND 接口与 ONFI / Toggle 协议  
> **下一章**：第 8 章 SSD 整体架构

## 章节概述

第 5–6 章我们理解了 NAND 的物理操作和接口协议。本章聚焦 **NAND 技术演进**——为什么 2D 平面 NAND 在 2010 年代触顶，3D 堆叠如何继续提升密度，以及各厂商的技术路线差异。

读完本章，你将理解：为什么现代 SSD 全部基于 3D NAND、Charge Trap 为何取代 Floating Gate 成为 3D 主流、QLC/PLC 在 SSD 中的真实定位，以及作为工程师应如何评估不同 NAND 代际的固件设计影响。

---

## 7.1 2D NAND 的缩放瓶颈

### 平面微缩的物理极限

2D NAND 通过缩小 Cell 特征尺寸（Feature Size）提升密度——从 2000 年代的 90 nm 逐步缩小到 2010 年代初的约 15–19 nm。但继续缩小面临不可逾越的物理障碍：

![](images/35c8419abef7.png)

<p align="center">图 7-1：2D NAND 平面微缩的三大瓶颈</p>

### 三大瓶颈

| 瓶颈 | 机制 | 后果 |
| --- | --- | --- |
| **Cell 间干扰（Cell-to-Cell Interference）** | Cell 间距缩小，Program/Erase 时相邻 Cell 的  被影响 | 读写错误率上升，需更强 ECC |
| **P/E 寿命下降** | 浮栅更薄，每次 Erase 对浮栅损伤更大 | P/E 次数从 SLC 100K 降至 TLC 3K |
| **数据保持力（Retention）恶化** | 浮栅面积减小，电荷泄漏更快 | 断电后数据保存时间缩短 |
| **光刻成本激增** | 低于 20 nm 需 EUV/E-Beam 等昂贵工艺 | $/GB 下降速度放缓 |

### 2D NAND 密度演进停滞

| 年代 | 工艺节点 | 每 Die 容量 | 代表产品 |
| --- | --- | --- | --- |
| 2005 | 90 nm | 约 1 Gb | — |
| 2008 | 34 nm | 约 32 Gb | — |
| 2011 | 20 nm | 约 64 Gb | 最后一代 2D MLC |
| 2013 | 19 nm | 约 128 Gb | 最后一代 2D TLC |
| 2015+ | — | 停止微缩 | 转向 3D |

2013 年后，所有主流 NAND 厂商停止 2D 微缩，全面转向 3D 堆叠——这不是选择，而是物理定律的必然结果。

### 为何必须转向 3D

2D 微缩的路是「把 Cell 做小」；3D 的路是「把 Cell 堆高」——在垂直方向堆叠更多层，而不是在平面方向继续缩小 Cell 尺寸。这绕开了 Cell 间干扰和光刻极限，以成熟的 40–50 nm 工艺节点实现更高密度。

---

## 7.2 3D NAND 堆叠原理（WL / BL / String Stack）

### 基本结构

3D NAND 将 Cell 沿垂直方向堆叠，形成 **NAND String（串）**：

![](images/d9406c83ca2b.png)

<p align="center">图 7-2：3D NAND 的 String Stack、Word Line 与 Bit Line 结构</p>

| 结构元素 | 方向 | 作用 |
| --- | --- | --- |
| **Word Line（WL，字线）** | 水平（横跨 String） | 选择同一层所有 Cell，施加读写电压 |
| **Bit Line（BL，位线）** | 垂直（连接 String 顶部） | 读取 Cell 电流 |
| **Select Gate（SGD/SGS）** | String 顶部/底部 | 选择哪些 String 参与操作 |
| **Channel（沟道）** | 垂直 U 形 | 电子传输通道 |
| **Control Gate（CG）** | 环绕 Cell | 施加控制电压 |

### U 形 Channel 结构

3D NAND 的 Channel 不是直线，而是 **U 形（或环形）**：

```
Bit Line (BL)  
    |  
  [SGD]  ← Select Gate Drain  
    |  
  [Cell N]   ← 第 N 层（最高层）  
  [Cell N-1]  
  [Cell ...]  
  [Cell 1]   ← 第 1 层  
    |  
  [SGS]  ← Select Gate Source  
    |  
  公共源极（Common Source Line）
```

Program/Erase 时，高电压施加在 WL 上，通过 U 形 Channel 中的电子隧穿改变 Charge Trap 层的电荷状态。

### 层数演进

| 时间 | 代表层数 | 每 Die 容量（TLC，约） | 里程碑 |
| --- | --- | --- | --- |
| 2013 | 24 层 | 约 64 Gb | 首款商用 3D NAND |
| 2015 | 32 层 | 约 128 Gb | 规模量产 |
| 2017 | 64 层 | 约 256 Gb | 消费级 SSD 普及 |
| 2019 | 96 层 | 约 512 Gb | — |
| 2020 | 128 层 | 约 512 Gb–1 Tb | 主流 |
| 2022 | 176 层 | 约 1 Tb | — |
| 2024 | 232–236 层 | 约 1 Tb+ | 最新量产 |

> 层数增加并非线性提升容量——每增加一层，工艺复杂度、良率和成本也显著上升。

### CMOS Under Array（CUA）与 4 Plane

**CUA（CMOS Under Array）**：将 NAND 外围电路（Row Decoder 等）移到 Cell 阵列下方，节省面积，允许更大阵列或更多 Plane。

**4 Plane 架构**：一个 Die 内有 4 个独立 Plane，可并行执行 Read/Program/Erase，吞吐量是 2 Plane 的约 2 倍。

| 架构 | Plane 数 | 并行度 | 典型应用 |
| --- | --- | --- | --- |
| 早期 3D NAND | 2 Plane | 中 | 2015–2018 年产品 |
| 现代 3D NAND | 4 Plane | 高 | 2020 年代主流 |
| 企业级 | 4 Plane + 多 Die | 最高 | 高 IOPS 场景 |

---

## 7.3 Charge Trap vs Floating Gate 结构

### 两种 Cell 结构

2D NAND 时代主流是 **Floating Gate（浮栅）** 结构；3D NAND 时代主流转向 **Charge Trap（电荷陷阱，CT）** 结构：

![](images/2ea672ab4323.png)

<p align="center">图 7-3：Floating Gate 与 Charge Trap 结构剖面对比</p>

### 结构对比

| 对比项 | Floating Gate（FG） | Charge Trap（CT） |
| --- | --- | --- |
| 电荷存储位置 | 多晶硅浮栅（完全隔离） | 氮化硅（SiN）层（电荷陷阱） |
| 3D 集成难度 | 高（U 形 Channel 中难以集成） | **低（天然适合 3D 堆叠）** |
| Cell 间干扰 | 较大 | **较小** |
| P/E 寿命 | 较长（SLC 100K） | 略短，但 3D 工艺可优化 |
| 主要厂商 | 早期 2D NAND | Samsung、Kioxia、Micron、YMTC |
| 3D 适用性 | 不适合 | **3D NAND 标配** |

### 为何 3D 时代选择 Charge Trap

1. **3D 兼容性**：CT 层可以均匀沉积在 U 形 Channel 壁上，FG 的多晶硅浮栅在 3D 结构中难以制造
2. **干扰更小**：CT 的电荷被限制在 SiN 层内，不会像 FG 那样通过浮栅耦合干扰相邻 Cell
3. **更薄 Cell**：CT Cell 高度更小，同样高度可堆叠更多层

### 对 SSD 的影响

| 影响维度 | 说明 |
| --- | --- |
| **Retention** | CT 的 Retention 略低于 FG，需更积极的 Read Refresh |
| **P/E 寿命** | 早期 CT 略低，现代 3D CT 已通过工艺优化接近 FG 水平 |
| **Read Retry** | CT 的  分布更宽，TLC/QLC 需要更复杂的 Read Retry 表 |
| **固件复杂度** | 需针对不同 WL 位置（顶部/中部/底部 Cell）使用不同 Read 电压 |

## 7.4 主流厂商技术路线概览

> 以下内容为技术参考，不构成产品推荐。各厂商命名和参数随代际快速更新，以官方发布为准。

![](images/3ec9a35e526d.png)

<p align="center">图 7-4：3D NAND 堆叠层数演进（行业参考，非特定厂商）</p>

### 各厂商技术概览

| 厂商 | 技术名称 | 结构特点 | 最新量产层数（约，2024） |
| --- | --- | --- | --- |
| **Samsung** | V-NAND | CT，U 形 Channel，CUA | 236 层（V9） |
| **Kioxia / WD** | BiCS FLASH | CT，圆形 Channel，CUA | 218 层（BiCS8） |
| **Micron** | CuA（CMOS Under Array） | CT，替换栅（Replacement Gate） | 232 层 |
| **SK hynix** | 4D NAND | CT，PUC（Peri Under Cell） | 238 层 |
| **YMTC（长江存储）** | Xtacking | CT，外围电路与阵列分两层键合 | 232 层（X4） |

### 技术路线差异（工程师视角）

| 差异点 | 说明 | 对固件的影响 |
| --- | --- | --- |
| **String 结构** | U 形 vs 圆形 Channel | Read Retry 表不同 |
| **Plane 数** | 2 vs 4 Plane | 并行调度策略不同 |
| **Page 大小** | 16 KB vs 32 KB | FTL 映射粒度、OOB 布局不同 |
| **接口速率** | NV-DDR2 vs NV-DDR3 | 时序参数表不同 |
| **特性寄存器** | 厂商私有 Feature 地址 | Get/Set Feature 命令需适配 |

**固件工程师的实际工作**：每适配一款新 NAND 颗粒，需要获取厂商的 **NAND 数据手册（Datasheet）** 和 **应用笔记（Application Note）**，配置时序参数、ECC 强度、Read Retry 表和坏块管理策略。

---

## 7.5 QLC / PLC 在 SSD 中的工程权衡

### 密度-寿命-性能三角

每增加 1 bit/Cell，密度提升约 33%，但 P/E 寿命和写入性能显著下降：

![](images/1ccbc5572424.png)

<p align="center">图 7-5：SLC / TLC / QLC / PLC 在密度、寿命、性能三角中的位置</p>

### 各类型在 SSD 中的定位

| 类型 | 密度 | P/E 寿命 | 写入速度 | SSD 定位 |
| --- | --- | --- | --- | --- |
| **SLC 模式** | 1× | 约 50K–100K | 最快 | 企业级缓存区、映射表存储 |
| **TLC** | 3× | 约 1K–3K | 中（+ SLC Cache） | **消费级/企业级主流** |
| **QLC** | 4× | 约 500–1K | 慢（Cache 外 ~100 MB/s） | 大容量消费级、只读为主场景 |
| **PLC** | 5× | 约 300–500 | 极慢 | 研发/归档，尚未大规模 SSD 商用 |

### SLC Cache 策略的工程实现

QLC/TLC SSD 通过 **SLC Cache** 掩盖写入性能差距：

| 阶段 | 实际写入模式 | 用户感知速度 |
| --- | --- | --- |
| Cache 区（SLC 模式） | 1 bit/Cell，高速 | 标称 2000–7000 MB/s |
| Cache 耗尽（直写 QLC/TLC） | 全 bit 写入，低速 | 约 100–500 MB/s |
| 后台 GC | 将 Cache 区数据搬至 QLC/TLC 区 | 可能影响读性能 |

**DWPD 与 QLC**：QLC 的 P/E 寿命低，DWPD 通常 ≤ 0.2（消费级），不适合写入密集型工作负载。

### 企业级 vs 消费级 QLC 策略

| 策略 | 消费级 QLC SSD | 企业级 QLC SSD |
| --- | --- | --- |
| **SLC Cache 大小** | 较大（几十 GB） | 较小或不用 |
| **OP 比例** | 约 7% | 约 14–28% |
| **DWPD** | 约 0.1–0.2 | 约 0.5–1 |
| **ECC 强度** | 标准 LDPC | 更强 LDPC + 更多 Read Retry |
| **目标场景** | 轻办公、游戏存储 | 读密集数据中心、CDN |

### PLC 的前景与限制

PLC（5 bit/Cell，32 种  状态）已在实验室和部分厂商样品中展示，但面临：

| 挑战 | 说明 |
| --- | --- |
| P/E 寿命 | 约 300–500 次，远低于 QLC |
| ECC 需求 | 需极强 LDPC，OOB 开销更大 |
| Read Retry | 32 状态需多次电压步进，读取延迟增加 |
| 应用场景 | 目前适合 WORM（一次写多次读）归档，非通用 SSD |

> 截至 2024–2025 年，主流 SSD 仍以 TLC 为主，QLC 在大容量消费级市场增长，PLC 尚未进入主流 SSD 产品。

---

## 本章小结

![](images/8acbbcb89ba6.png)

<p align="center">图 7-6：第 7 章知识要点总览</p>

* **2D NAND** 在约 15 nm 节点触顶，Cell 干扰、P/E 下降、光刻成本是三大瓶颈
* **3D NAND** 通过垂直堆叠（32→236+ 层）继续提升密度，绕开平面微缩极限
* **String Stack** 结构：WL 水平选择层，BL 垂直读取，U 形 Channel 传输电子
* **Charge Trap** 取代 Floating Gate 成为 3D 主流，更适合 U 形 Channel，干扰更小
* **厂商路线**：Samsung V-NAND、Kioxia BiCS、Micron CuA、YMTC Xtacking 等，固件需逐颗适配
* **QLC/PLC**：密度更高但寿命和性能更低，SLC Cache 是消费级 QLC 的关键工程手段

---

## 常见问题 FAQ

**Q1：3D NAND 的「层数」是指什么？**

层数（Layers/Stacks）指垂直方向上 Word Line 的数量，即 Cell 的堆叠高度。128 层 3D NAND 意味着每个 String 中有 128 个 Cell 垂直排列（不含 Select Gate）。层数越多，同样 Die 面积内容量越大。

**Q2：3D NAND 的 Page 大小和 2D 一样吗？**

不一定。早期 3D NAND 沿用 16 KB Page，现代高端 3D NAND（如 128+ 层）已转向 **32 KB Page**，ECC 和 OOB 布局也相应变化。固件适配新 NAND 时必须确认 Page 大小。

**Q3：为什么企业级 SSD 不用 QLC？**

部分企业级 SSD 已采用 QLC（如数据中心读密集场景），但大多数企业级仍用 TLC，原因是：QLC 的 P/E 寿命（约 500–1K）和 DWPD（约 0.1–0.2）难以满足 1+ DWPD 的企业级 SLA，且写入性能在 Cache 外过低。

**Q4：YMTC Xtacking 和 Samsung V-NAND 有什么本质区别？**

Samsung V-NAND 是在同一硅片上垂直堆叠 Cell 层（单片集成）；YMTC Xtacking 将外围 CMOS 电路和 NAND Cell 阵列分别在两片 wafer 上制造，再通过键合（Bonding）连接。Xtacking 的优势是外围电路不占用阵列面积，但键合工艺增加了制造复杂度。

**Q5：PLC NAND 何时会进入主流 SSD？**

目前尚无明确时间表。PLC 的主要障碍是 P/E 寿命（约 300–500 次）和读取复杂度（32 状态）。短期更可能出现在 **WORM 归档存储**（一次写入、极少擦除）而非通用读写 SSD。QLC 在 SSD 中的普及经验表明，新 Cell 类型需要 3–5 年工程优化才能成熟。

---

## 自测题

**1. 2D NAND 停止微缩的三个主要原因是什么？**

参考答案：

1. **Cell 间干扰**：Cell 间距缩小导致 Program/Erase 时相互影响，错误率上升
2. **P/E 寿命下降**：浮栅变薄，每次 Erase 损伤更大，P/E 次数不可接受
3. **光刻成本**：低于 20 nm 需 EUV 等昂贵工艺，$/GB 下降放缓

**2. 3D NAND 中 Word Line、Bit Line 和 NAND String 的关系是什么？**

参考答案：

* **NAND String**：垂直方向上串联多个 Cell 的单元，是 3D 堆叠的基本组成
* **Word Line（WL）**：水平方向横跨所有 String，选择同一层的 Cell 施加读写电压
* **Bit Line（BL）**：垂直方向连接 String 顶部，读取 Cell 电流以感知数据
* 一个 WL 对应一层 Cell，多条 BL 对应多个 String

**3. Charge Trap 相比 Floating Gate 更适合 3D NAND 的两个原因？**

参考答案：

1. **3D 工艺兼容性**：CT 的 SiN 电荷陷阱层可均匀沉积在 U 形 Channel 壁上，FG 的多晶硅浮栅在 3D 结构中难以制造
2. **Cell 间干扰更小**：CT 电荷被限制在 SiN 层内，不会像 FG 那样通过浮栅耦合干扰相邻 Cell

**4. 为什么 QLC SSD 需要 SLC Cache？不用会怎样？**

参考答案：

* QLC 直写速度约 100–500 MB/s，远低于用户期望的 2000+ MB/s
* SLC Cache 将部分 QLC Cell 以 1 bit 模式运行，获得 SLC 级写入速度（约 2000–7000 MB/s）
* 不用 SLC Cache 的 QLC SSD 在持续写入场景下性能极差，用户体验不可接受

**5. 固件工程师适配新 NAND 颗粒时需要关注哪些关键参数？**

参考答案：

* **Page/Block 大小**：决定 FTL 映射粒度和 OOB 布局
* **Plane 数量**：影响并行调度策略
* **接口时序（Timing Mode）**：NV-DDR2/3 速率，需配置时序参数表
* **ECC 要求**：RBER 规格，决定 LDPC 强度配置
* **Read Retry 表**：厂商提供的电压步进表，TLC/QLC 必须适配
* **Feature 寄存器**：Get/Set Feature 地址，厂商私有

---

## 延伸阅读

| 资源 | 说明 |
| --- | --- |
| Flash Memory Summit 主题演讲 | 3D NAND 技术演进与厂商发布 |
| JEDEC 固态存储标准 | 可靠性测试与 Retention 规范 |
| SNIA: NAND Flash Technology | 闪存技术概述 |
| 本书 第 8 章 | SSD 整体架构，进入第三篇 |
| 本书 第 12 章 | FTL 地址映射，NAND 限制的软件应对 |
| 本书 第 29 章 | NAND 后端固件适配实战 |

---

> **下一章预告**：第 8 章 SSD 整体架构—— 第二篇我们理解了 NAND 的「原材料」，第三篇进入 SSD 硬件架构。控制器、DRAM、PCB、数据路径——一块 SSD 是如何由 NAND 颗粒组装成完整产品的？