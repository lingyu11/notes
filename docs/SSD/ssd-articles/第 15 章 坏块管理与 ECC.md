第 15 章 坏块管理与 ECC

> **本章目标**：理解出厂/运行时坏块、BBT 管理、BCH/LDPC 选型，以及 Read Retry 与 RAID-like 保护  
> **前置章节**：第 14 章 磨损均衡  
> **下一章**：第 16 章 写入放大与性能优化  
> **难度标记**：🔧 面向固件/嵌入式开发人员

## 章节概述

NAND 制造工艺决定了**一定比例的 Block 出厂即不可用**；使用过程中还会因 P/E 耗尽、Program/Erase/Read Fail 产生**运行时坏块**。FTL 必须维护 **坏块表（BBT）**，跳过坏块并用备用块替换。

同时，NAND 原始误码率（RBER）随制程缩小急剧上升，**ECC（Error Correction Code）** 是数据可靠性的最后防线。本章从 FTL/BE 视角深化 第 6 章 §6.5 的 ECC 入门内容。

---

## 15.1 出厂坏块（Factory Bad Block）与运行时坏块

![](images/cb57efb4811b.jpg)

<p align="center">图 15-1：出厂坏块与运行时坏块的产生原因 </p>

### 出厂坏块（Factory Bad Block）

| 特点 | 说明 |
| --- | --- |
| **来源** | 制造缺陷，Wafer 测试阶段标记 |
| **标记** | NAND 厂商在特定 Page/OOB 写入坏块标记 |
| **比例** | 典型约 1–5%（因厂商和容量而异） |
| **处理** | FTL 初始化时扫描，加入 BBT，永不使用 |

### 运行时坏块（Runtime Bad Block / Grown Bad Block）

| 触发 | 说明 |
| --- | --- |
| **Program Fail** | 写入时 ECC 无法纠正或状态寄存器报错 |
| **Erase Fail** | Block Erase 超时或失败 |
| **Read Fail** | Read Retry 耗尽仍无法恢复 |
| **P/E 耗尽** | Block 超过规格 P/E 次数，可靠性不达标 |

### 坏块 vs 保留块

| 概念 | 说明 |
| --- | --- |
| **Bad Block** | 不可用，FTL 跳过 |
| **Spare / Replacement Block** | OP 预留的备用 Block，用于替换坏块 |
| **可用容量** | 标称容量 < NAND 物理容量，差额含 OP + 坏块 + 坏块替换池 |

---

## 15.2 坏块表（BBT）管理

![](images/dca9d8fff785.jpg)

<p align="center">图 15-2：Bad Block Table 结构与备用 Block 替换流程</p>

### BBT 数据结构

FTL 维护全局或分 Die 的坏块表：

```c
typedef struct {  
    uint16_t die;  
    uint16_t plane;  
    uint16_t block;  
    uint8_t  type;   /* FACTORY_BB / RUNTIME_BB */  
} bad_block_entry_t;  
  
bad_block_entry_t bbt[MAX_BAD_BLOCKS];
```

### 初始化流程

```
上电 → NAND ID 读取  
     → 扫描 Factory BB 标记（厂商规定 Page）  
     → 加载持久化 Runtime BBT（存在 NAND 元数据区）  
     → 构建可用 Block 池（排除所有 BB）  
     → 映射表加载
```

### 运行时坏块替换

```
Program/Erase Fail on Block X  
  → 标记 Block X 为 Runtime BB，写入 BBT 持久化  
  → 从 Spare Pool 分配 Replacement Block R  
  → 若 Block X 有 Valid Data：Read → Program 到 R → 更新 L2P  
  → Block X 永久隔离
```

### BBT 持久化

* BBT 变更须**立即或 Checkpoint 时**写入 NAND，防止掉电丢失
* 多副本存储（不同 Die/Block），防止单点损坏

---

## 15.3 ECC 纠错：BCH vs LDPC 原理与选型

![](images/8f7da6109e09.jpg)

<p align="center">图 15-3：BCH（硬判决）与 LDPC（软判决）对比</p>

### 为什么需要 ECC

| 参数 | 典型值（约） |
| --- | --- |
| **Raw BER（原始误码率）** | 3D TLC 约 10⁻² ~ 10⁻³ per Page |
| **可接受 UBER** | 约 < 10⁻¹⁷（企业级更严） |
| **纠错位需求** | 数百 bit/Page（随制程递增） |

### BCH 码

| 特点 | 说明 |
| --- | --- |
| **判决** | 硬判决（Hard Decision） |
| **能力** | 典型纠 40–80 bit/512 B |
| **延迟** | 低，硬件实现成熟 |
| **适用** | SLC、早期 MLC、2D NAND |

### LDPC 码

| 特点 | 说明 |
| --- | --- |
| **判决** | 软判决（Soft Decision），利用 LLR |
| **能力** | 纠 100–300+ bit/KB |
| **延迟** | 较高，需多次迭代 |
| **适用** | **3D TLC/QLC 主流** |

### 选型趋势

```
2D MLC  → BCH  
3D TLC  → LDPC（Strong LDPC）  
3D QLC  → 更强 LDPC + Read Retry + RAID-like
```

SSD 控制器 SoC 内置 **LDPC Encoder/Decoder** 硬件引擎（见 第 9 章），BE 固件配置纠错强度（Code Rate）。

---

## 15.4 Read Retry / Read Recovery 流程

![](images/b3f734fd11e8.jpg)

<p align="center">图 15-4：Read Retry 参考电压步进与 Retry Table</p>

### 读失败原因

* Cell 阈值电压漂移（Retention、Read Disturb、P/E 磨损）
* 温度变化导致最佳读取参考电压偏移
* 相邻 Cell 干扰

### Read Retry 流程

```
NAND Read → LDPC Decode Fail  
  → BE 按 Retry Table 调整 Read Offset（参考电压）  
  → 重新 Read + Decode  
  → 重复 N 次（典型 10–30 级）  
  → 仍失败 → 标记 Page/Block 坏块或触发 RAID-like 恢复
```

### Retry Table

| 内容 | 说明 |
| --- | --- |
| **Offset 序列** | 每组 Read Level 的电压偏移步进 |
| **温度分区** | 不同温度区间用不同 Table |
| **Adaptive Read** | 根据历史成功 Offset 动态调整 |

### Read Recovery（企业级）

* 后台 **Scrub**：定期读旧数据，ECC 高则主动 Refresh 到新 Page
* **Media Scan**：全盘扫描，预防性搬迁

---

## 15.5 RAID-like 保护（Die-level / Page-level RAID）

![](images/eeec74d2377f.jpg)

<p align="center">图 15-5：Die-level / Page-level Parity Stripe 组织</p>

### 动机

当 ECC + Read Retry 仍无法恢复时，RAID-like 提供**额外冗余**，避免单 Die/Page 故障导致数据丢失。

### Die-level RAID

| 特点 | 说明 |
| --- | --- |
| **组织** | N 个 Die 数据 + 1 个 Die Parity（类似 RAID 5） |
| **粒度** | Stripe 跨 Die |
| **开销** | 容量损失约 1/(N+1) |
| **场景** | 企业级 SSD |

### Page-level RAID

| 特点 | 说明 |
| --- | --- |
| **组织** | 同一 Page 地址跨 Die 做 XOR Parity |
| **粒度** | 更细，恢复单 Page |
| **开销** | 略增 WAF（Parity 需更新） |

### 恢复流程

```
Page Read Fail + Retry 耗尽  
  → 读同 Stripe 其他 Die 的 Data + Parity  
  → XOR 重建丢失 Page  
  → 可选：搬迁到新 Block，标记坏块
```

> RAID-like 是 ECC 的补充，**不能替代** ECC；企业级盘常见 Die-level RAID + Strong LDPC 组合。

---

## 本章小结

![](images/26149d69a2ad.jpg)

<p align="center">图 15-6：第 15 章知识要点总览</p>

* **出厂坏块**制造时标记；**运行时坏块**由 Fail 或 P/E 耗尽产生
* **BBT** 记录坏块，**Spare Block** 替换；BBT 须持久化
* **BCH** 硬判决，适合老制程；**LDPC** 软判决，3D TLC/QLC 主流
* **Read Retry** 调整参考电压多次重读；**Adaptive Read** 优化 Offset
* **RAID-like** Die/Page 级 Parity，ECC 失败后的最后防线

---

## 常见问题（FAQ）

**Q1：用户能看到坏块数量吗？**

SMART 中 **Available Spare** 反映剩余备用块比例；**Media Errors** 等字段间接反映坏块增长。一般不提供逐块列表。

**Q2：LDPC 解码失败一定丢数据吗？**

不一定——还有 Read Retry、RAID-like 重建；都失败才可能 UECC（Uncorrectable ECC），上报 NVMe 错误。

**Q3：Factory BB 比例高会影响性能吗？**

FTL 初始化时已排除，不影响运行时性能；但 Spare 池减小会缩短**可替换坏块**的寿命上限。

---

## 自测题

**1. 出厂坏块和运行时坏块有何区别？**

参考答案：出厂坏块是制造缺陷，厂商测试时已标记，FTL 初始化扫描跳过；运行时坏块是使用过程中 Program/Erase/Read Fail 或 P/E 耗尽导致，需动态加入 BBT 并用 Spare Block 替换。

**2. 为什么 3D TLC/QLC 普遍用 LDPC 而非 BCH？**

参考答案：3D TLC/QLC 原始误码率更高，需更强纠错能力；LDPC 软判决可纠更多 bit，虽延迟和硬件复杂度更高，但能满足 UBER 要求。

**3. Read Retry 解决什么问题？**

参考答案：Cell 阈值电压漂移（Retention、温度、干扰）导致默认参考电压读错；Retry 按 Table 步进调整 Read Offset 重新读取，使 LDPC 可成功解码。

**4. Die-level RAID-like 与 ECC 的关系？**

参考答案：ECC 纠正 Page 内随机 bit 错误；RAID-like 在整 Page 或 Die 不可恢复时，用 Parity 重建数据。两者互补，ECC 是第一道防线，RAID-like 是最后冗余。

---

## 延伸阅读

| 资源 | 说明 |
| --- | --- |
| 第 6 章 §6.5 | ECC 入门与 RBER |
| 第 29 章 | BE Read Retry 实战 |
| ONFI / JEDEC NAND 规范 | Factory BB 标记格式 |
| LDPC 相关论文 | 软判决译码算法 |

---

> **下一章预告**：第 16 章 写入放大与性能优化——WAF、OP、Trim、SLC Cache 与 I/O 优化