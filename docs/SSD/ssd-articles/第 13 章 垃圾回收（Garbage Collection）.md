第 13 章 垃圾回收（Garbage Collection）

> **本章目标**：理解无效页如何产生、GC 触发时机与算法，以及 GC 对 SSD 性能的影响  
> **前置章节**：第 12 章 FTL 基础  
> **下一章**：第 14 章 磨损均衡  
> **难度标记**：🔧 面向固件/嵌入式开发人员

## 章节概述

FTL 采用 Out-of-Place Update 后，NAND 中会积累大量**无效页（Invalid Page）**——它们占据物理空间但不再被任何 LBA 引用。**垃圾回收（Garbage Collection, GC）** 的任务就是：迁移 Block 中的有效页，Erase 整个 Block，将其归还空闲块池。

GC 是 SSD 性能抖动（Stuttering）和写入放大（WAF）的主要来源之一。理解 GC，是分析 SSD 性能曲线和优化 FTL 的基础。

---

## 13.1 无效页（Invalid Page）的产生

![](images/c5fcff1faad9.jpg)

<p align="center">图 13-1：Out-of-Place Update 导致无效页，以及页的三种状态</p>

### 页的三种状态

| 状态 | 含义 |
| --- | --- |
| **Valid** | 被某 LBA 通过 L2P 引用，数据有效 |
| **Invalid** | 曾被写入，但 L2P 已指向新位置，数据废弃 |
| **Free** | Block Erase 后，尚未 Program 的空闲页 |

### 无效页产生的三种典型场景

| 场景 | 过程 |
| --- | --- |
| **Overwrite** | Host 写同一 LBA 的新数据 → 新 Page Valid，旧 Page Invalid |
| **Trim / Discard** | 主机声明 LBA 不再使用 → L2P 标记无效，对应 Page 变 Invalid |
| **GC / WL 搬迁** | 数据搬到新 Block → 源 Page 变 Invalid |

### 为何不能立即 Erase Invalid Page？

NAND **Erase 最小单位是 Block**，不能只 Erase 单个 Page。必须等 Block 内无效页足够多、搬迁有效页成本可接受时，才 Erase 整个 Block。

---

## 13.2 GC 触发条件与时机选择

![](images/ace341458145.jpg)

<p align="center">图 13-2：GC 触发条件——空闲块阈值与 Foreground / Background 触发</p>

### 主要触发条件

| 条件 | 说明 |
| --- | --- |
| **空闲块不足** | Free Block 数量低于阈值（如 OP 的 10–20%） |
| **Host Write 压力** | 写入速率持续高于 GC 回收速率 |
| **OP 预警** | 预留空间即将耗尽，需主动 GC 释放 |
| **Idle 检测** | 主机 I/O 空闲，利用窗口做后台 GC |

### Foreground GC vs 预防性 GC

| 类型 | 触发时机 | 特点 |
| --- | --- | --- |
| **Foreground GC** | 空闲块耗尽，Host Write 必须等待 | 延迟尖刺，用户可感知 |
| **Background GC** | 空闲块接近阈值，I/O 负载较低时 | 提前回收，平滑性能 |
| **Idle GC** | 盘片一段时间无 I/O | 不影响用户体验 |

### 阈值设计权衡

* 阈值**过高**：GC 过于频繁，WAF 增大，NAND 磨损加速
* 阈值**过低**：容易触发 Foreground GC，性能抖动

典型 OP（Over-Provisioning）约 7–28%（见 第 16 章），为 GC 提供缓冲空间。

---

## 13.3 GC 算法：Greedy / Cost-Benefit / Wear-Aware

![](images/3171d827d076.jpg)

<p align="center">图 13-3：Greedy、Cost-Benefit、Wear-Aware 三种 GC 算法</p>

### Greedy（贪心）

**选择无效页最多的 Block** 作为 Victim。

| 优点 | 缺点 |
| --- | --- |
| 每次 GC 释放空间最多 | 可能反复擦同一 Block，WL 差 |
| 实现简单 | 忽略 Erase 成本 |

```
Victim = Block with max(invalid_page_count)
```

### Cost-Benefit（收益/成本）

综合考虑**回收收益**与**迁移成本**：

```
Score = (Invalid Pages / Total Pages) / (1 + Age_factor)
```

* 无效页比例高 → 收益大
* Block 年龄大（数据冷）→ 有效页少，迁移成本低

### Wear-Aware（磨损感知）

在 Cost-Benefit 基础上加入 **P/E Cycle** 因素：

* 优先回收 **高 P/E、无效页多** 的 Block（反正快磨损完）
* 或优先保留 **低 P/E** 的 Block 给热数据

> GC 与 WL 紧密耦合，详见 第 14 章。

### GC 执行步骤（通用）

```
1. 选择 Victim Block  
2. 扫描 P2L，找出 Valid Pages  
3. Read Valid Pages → DRAM  
4. Program 到新的 Free Block  
5. 更新 L2P  
6. Erase Victim Block  
7. Victim Block 加入 Free Block List
```

---

## 13.4 GC 对性能的影响（Write Amplification、性能抖动）

![](images/d9ca4ec52fb2.jpg)

<p align="center">图 13-4：GC 导致的 WAF 增加、延迟尖刺与 IOPS 下降</p>

### Write Amplification（WAF）

GC 需要将有效页**重新 Program** 到新 Block，产生额外 NAND 写入：

```
WAF = 总 NAND 写入量 / Host 写入量
```

GC 是 WAF 的主要贡献者之一（另含 WL 搬迁、PLP 刷写等，见 第 16 章）。

| 场景 | WAF（约） |
| --- | --- |
| 顺序写满盘 | 1.0–1.5 |
| 随机写稳态 | 2–5（视 OP 和算法） |
| OP 不足 + 随机写 | 可达 5–10+ |

### 性能抖动（Stuttering）

| 现象 | 原因 |
| --- | --- |
| **fio 曲线锯齿** | Foreground GC 与 Host Write 争用 NAND |
| **延迟尖刺** | GC Erase（约 2–5 ms）阻塞 Host I/O |
| **IOPS 骤降** | 多 Channel 被 GC 占满 |

消费级盘常见「**缓外写**」：SLC Cache 写满后 TLC 直写 + GC 叠加，速度从 GB/s 降至数百 MB/s。

---

## 13.5 前台 GC vs 后台 GC vs Idle GC

![](images/467c8244ab66.jpg)

<p align="center">图 13-5：Foreground / Background / Idle GC 的触发时机</p>

| 类型 | 触发 | Host I/O 影响 | 典型场景 |
| --- | --- | --- | --- |
| **Foreground GC** | 无空闲块，Write 必须等待 | **高** ，Write 延迟增加 | 随机写满盘、OP 不足 |
| **Background GC** | 空闲块低于阈值，负载低 | **低** ，可限速 | 日常混合负载 |
| **Idle GC** | 无 I/O 超过 N 秒 | **无** | 系统空闲、夜间 |

### Idle GC 的价值

利用主机不访问 SSD 的时间窗口， aggressively 回收无效页，使下次写入时 Free Block 充足——这是消费级盘维持「突发后恢复性能」的常见策略。

---

## 13.6 GC 与 Host I/O 的优先级仲裁

### 典型优先级（从高到低）

```
Host Read  >  Host Write  >  Background GC  >  Idle GC / WL
```

### 仲裁策略

| 策略 | 说明 |
| --- | --- |
| **可中断 GC** | Host Read 到达时暂停 GC，读完继续 |
| **Partial GC** | 只迁移部分 Valid Page 即暂停，下次继续 |
| **GC 限速** | 限制 GC 占用 Channel 比例（如 ≤ 30%） |
| **Write Stall** | Foreground GC 时 Host Write 排队等待 |

### 企业级 vs 消费级

| 类型 | 策略倾向 |
| --- | --- |
| **企业级** | 严格 QoS，GC 限速，保证读延迟 SLA |
| **消费级** | 允许 Foreground GC，优先成本与容量 |

---

## 本章小结

![](images/49a2c00df2ae.jpg)

<p align="center">图 13-6：第 13 章知识要点总览</p>

* **无效页**由 Overwrite、Trim、搬迁产生；Erase 单位是 Block，故需 GC
* GC 在**空闲块不足**或**后台预防**时触发；Foreground GC 导致性能抖动
* 算法：**Greedy**（最多无效页）、**Cost-Benefit**（收益/成本）、**Wear-Aware**（兼顾 P/E）
* GC 增加 **WAF** 和 **延迟尖刺**；OP 和 Background/Idle GC 可缓解
* 优先级：**Host Read > Host Write > GC**；可中断与 Partial GC 降低影响

---

## 常见问题（FAQ）

**Q1：GC 能关掉吗？**

不能。没有 GC，无效页占满 NAND 后无法写入。ZNS 等方案把 GC 责任转移给主机，但 SSD 内部仍需 Reclaim 类操作。

**Q2：Trim 命令对 GC 有什么帮助？**

Trim 后 LBA 对应 Page 直接变 Invalid，不需等 Overwrite，GC 可选 Victim Block 时无效页比例更高，回收更高效，WAF 降低。

**Q3：为什么满盘后 SSD 变慢？**

空闲块减少，几乎每次 Write 都触发 GC；Foreground GC 与 Host Write 争用 NAND，且 SLC Cache（如有）也可能已满。

---

## 自测题

**1. 无效页产生的三个典型场景是什么？**

参考答案：Host Overwrite（写同一 LBA 新数据）、Trim/Discard（主机声明 LBA 废弃）、GC/WL 数据搬迁（源 Page 被替代）。

**2. Greedy GC 和 Cost-Benefit GC 的主要区别？**

参考答案：Greedy 选无效页最多的 Block，实现简单但可能忽略迁移成本和磨损；Cost-Benefit 综合无效页比例与 Block 年龄（有效页迁移成本）计算得分选 Victim。

**3. Foreground GC 和 Background GC 对用户体验的影响有何不同？**

参考答案：Foreground GC 在空闲块耗尽时触发，Host Write 必须等待 GC 完成，导致明显延迟尖刺；Background GC 在阈值预警时提前回收，与 Host I/O 并行或限速，用户基本无感。

**4. 为什么 GC 会增加 WAF？**

参考答案：GC 需将 Victim Block 中的 Valid Page Read 出来再 Program 到新 Block，这些 NAND 写入不是 Host 直接请求的，属于额外写入，计入 WAF 分子。

---

## 延伸阅读

| 资源 | 说明 |
| --- | --- |
| 第 14 章 | GC 与 WL 的协同 |
| 第 16 章 | WAF 计算与 OP |
| 第 28 章 §28.5 | GC 线程设计实战 |
| A Reconfigurable FTL 等论文 | GC 算法学术参考 |

---

> **下一章预告**：第 14 章 磨损均衡——静态/动态 WL、冷热分离与寿命建模