第 14 章 磨损均衡（Wear Leveling）

> **本章目标**：理解静态/动态磨损均衡、冷热数据分离，以及 P/E 计数与 SSD 寿命建模  
> **前置章节**：第 13 章 垃圾回收  
> **下一章**：第 15 章 坏块管理与 ECC  
> **难度标记**：🔧 面向固件/嵌入式开发人员

## 章节概述

NAND Flash 每个 Block 的 **P/E（Program/Erase）次数有限**（TLC 典型约 1,000–3,000 次，SLC 可达 10 万+）。若某些 Block 被反复 Erase 而其他 Block 几乎不动，盘片会**局部先磨损耗尽**。

**磨损均衡（Wear Leveling, WL）** 的目标是让所有 Block 的 P/E 次数尽量均匀，最大化 SSD 整体寿命。WL 与 GC 紧密协作——GC 选 Victim Block 时可兼顾 P/E，静态 WL 则主动搬迁冷数据。

---

## 14.1 静态磨损均衡 vs 动态磨损均衡

![](images/c36c4a2358ab.jpg)

<p align="center">图 14-1：动态 WL（写时选低 P/E 块）与静态 WL（搬迁冷数据）</p>

### 动态 WL（Dynamic Wear Leveling）

**在 Host Write 分配新 Block 时**，优先选择 **P/E 次数最低** 的空闲 Block。

| 特点 | 说明 |
| --- | --- |
| 触发 | 每次 Write 分配 PBA 时 |
| 开销 | 几乎无额外搬迁 |
| 局限 | **冷数据** 长期占用某 Block 不移动，该 Block P/E 低但相邻热 Block 可能耗尽 |

### 静态 WL（Static Wear Leveling）

**主动搬迁长期不更新的冷数据**，使其占用的 Block 也能被 Erase，参与 P/E 轮换。

| 特点 | 说明 |
| --- | --- |
| 触发 | 某 Block P/E 远高于平均值，且含大量冷数据 |
| 过程 | Read 冷数据 → Program 到低 P/E Block → 更新 L2P → Erase 源 Block |
| 开销 | 额外 Read/Program，增加 WAF |

### 对比

| 维度 | 动态 WL | 静态 WL |
| --- | --- | --- |
| 实现复杂度 | 低 | 高 |
| 额外 WAF | 几乎无 | 有（冷数据搬迁） |
| 寿命均衡效果 | 一般 | 好 |
| 适用 | 所有 SSD | 企业级 / 长寿命盘 |

> 主流 SSD **两者结合**：动态 WL 日常分配 + 静态 WL 定期或阈值触发。

---

## 14.2 冷热数据分离策略

![](images/be7d22457223.jpg)

<p align="center">图 14-2：热数据区与冷数据区的分离策略</p>

### 热数据 vs 冷数据

| 类型 | 特征 | 示例 |
| --- | --- | --- |
| **热数据** | 频繁 Overwrite | 日志、数据库 WAL、Swap |
| **冷数据** | 写入后长期只读 | 归档文件、 OS 安装文件 |

### 分离策略的价值

* 热数据写入集中到 **SLC Cache 区或专用 Block 池**，高 P/E 块专门服务热数据
* 冷数据放入 **低更新频率 Block 池**，减少静态 WL 搬迁需求
* GC 选 Victim 时，**热区 Block** 无效页比例通常更高，回收更高效

### 实现手段

| 手段 | 说明 |
| --- | --- |
| **写入计数** | 每 LBA 或每 Block 记录更新频率 |
| **时间戳** | 最后写入时间，久未更新视为冷 |
| **分区隔离** | FTL 维护 Hot Pool / Cold Pool 空闲块列表 |

---

## 14.3 块年龄管理与 P/E Cycle 计数

![](images/f6a9b8ebb827.jpg)

<p align="center">图 14-3：每 Block 的 Erase Count 与静态 WL 触发阈值</p>

### P/E Cycle 计数

FTL 为**每个 Block** 维护 Erase Count（存在 DRAM 或 NAND 元数据区）：

```c
typedef struct {  
    uint32_t erase_count;    /* 累计 Erase 次数 */  
    uint32_t valid_pages;    /* 当前有效页数 */  
    uint8_t  block_state;    /* FREE / ACTIVE / GC_VICTIM / BAD */  
} block_meta_t;  
  
block_meta_t block_table[MAX_BLOCKS];
```

### 块年龄（Block Age）

| 概念 | 含义 |
| --- | --- |
| **Erase Count** | Block 被 Erase 的次数，直接反映磨损 |
| **Age** | 自上次 Erase 以来的时间或写入量，辅助 GC Cost-Benefit |

### 静态 WL 触发条件（典型）

```c
if (block.erase_count > avg_pe + threshold)  
   && (block.valid_pages > 0)  
   && (data_is_cold(block))  
then  
   trigger_static_wl(block);
```

`threshold` 典型为平均 P/E 的 10–30%（因产品而异）。

---

## 14.4 磨损均衡对寿命的影响建模

![](images/6a32b734e7cf.jpg)

<p align="center">图 14-4：静态 WL 冷数据搬迁流程</p>

![](images/bca68981533e.jpg)

<p align="center">图 14-5：TBW、DWPD 与 P/E、WAF、OP 的关系</p>

### 关键寿命指标

| 指标 | 含义 |
| --- | --- |
| **TBW** | Total Bytes Written，全生命周期可写入总字节数 |
| **DWPD** | Drive Writes Per Day，每天可写入容量倍数 |
| **MTBF** | 平均无故障时间（与 NAND 无关，偏系统级） |

### 简化寿命公式（约）

```
TBW ≈ (NAND 总容量 × 每 Cell P/E 次数 × OP 系数) / WAF
```

| 因素 | 影响 |
| --- | --- |
| **P/E 次数** | TLC 约 1K–3K，QLC 约 1K，SLC 约 100K |
| **WAF** | 越大，有效 TBW 越少（GC/WL 额外写入） |
| **OP** | 预留空间越大，可轮换 Block 越多，寿命略增 |
| **WL 质量** | 静态 WL 使 min/max P/E 差距小，接近理论 TBW |

### DWPD 计算示例

```
1 TB 企业盘，TBW = 3,600 TB，保修 5 年  
DWPD = 3600 TB / (1 TB × 365 × 5) ≈ 2 DWPD
```

### WL 对建模的实际意义

* **无 WL**：某 Block 先耗尽 → 坏块替换 → 备用块耗尽 → 盘失效，实际 TBW 远低于标称
* **良好 WL**：所有 Block 均匀磨损 → 实际 TBW 接近标称

---

## 本章小结

![](images/550d9038d3cd.jpg)

<p align="center">图 14-6：第 14 章知识要点总览</p>

* **动态 WL**：Write 时选低 P/E 空闲 Block，开销小
* **静态 WL**：搬迁冷数据，使高 P/E Block 可 Erase，均衡更好
* **冷热分离**减少无效搬迁，优化 GC 与 WL 效率
* 每 Block 维护 **Erase Count**，超阈值触发静态 WL
* **TBW ≈ 容量 × P/E × OP / WAF**；WL 质量决定能否达到标称寿命

---

## 常见问题（FAQ）

**Q1：动态 WL alone 够用吗？**

对写入均匀的场景基本够用；若存在大量冷数据（如系统盘），静态 WL 必要，否则冷数据 Block 永不 Erase，热 Block 先耗尽。

**Q2：静态 WL 会增加多少 WAF？**

取决于冷数据比例和触发频率，典型增加 0.1–0.5；企业级盘可接受，以换取寿命和可靠性。

**Q3：SMART 里的 Wear Leveling Count 是什么？**

多为 **最大 P/E 与最小 P/E 的差值** 或 **平均 P/E**，用于评估 WL 效果和剩余寿命。

---

## 自测题

**1. 动态 WL 和静态 WL 的核心区别是什么？**

参考答案：动态 WL 在 Host Write 分配 Block 时选低 P/E 块，无额外搬迁；静态 WL 主动搬迁冷数据，使长期不更新的 Block 也能被 Erase 参与轮换。

**2. 为什么需要冷热数据分离？**

参考答案：热数据频繁 Overwrite 适合高 P/E Block；冷数据长期占用 Block 阻碍 WL。分离后可减少静态 WL 搬迁开销，GC 也更容易选到无效页多的 Victim Block。

**3. 静态 WL 的典型触发条件是什么？**

参考答案：某 Block 的 Erase Count 显著高于平均值（如超过 avg + threshold），且 Block 内仍有 Valid Page，且数据被判定为冷数据。

**4. WAF 增大对 TBW 有什么影响？**

参考答案：TBW 与 WAF 成反比——同样 P/E 次数下，WAF 越大，Host 可写入的有效数据越少，标称 TBW 需按更高 WAF 折减或 NAND 需更多 OP。

---

## 延伸阅读

| 资源 | 说明 |
| --- | --- |
| 第 2 章 §2.4 | TBW / DWPD 用户解读 |
| 第 5 章 §5.5 | P/E Cycle 物理限制 |
| 第 16 章 | WAF 与 OP |
| 第 28 章 §28.6 | WL 模块实现 |

---

> **下一章预告**：第 15 章 坏块管理与 ECC——出厂坏块、BBT、LDPC 与 Read Retry