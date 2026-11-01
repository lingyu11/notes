第 8 章 SSD 整体架构

> **本章目标**：建立 SSD 从主机接口到 NAND 颗粒的整体硬件视图，理解数据路径与控制路径  
> **前置章节**：第 7 章 3D NAND 与现代闪存技术  
> **下一章**：第 9 章 SSD 控制器（SoC）详解

---

## 章节概述

第二篇我们深入了 NAND 的物理原理与接口协议。从本章起进入**第三篇：SSD 硬件架构**——把 NAND 颗粒、控制器、DRAM、PCB 组装成一块完整的 SSD 产品。

一块 SSD 不是「NAND 芯片 + 线」那么简单。它有一套完整的数据通路（Host 数据如何到达 NAND）、控制通路（固件如何调度 GC/FTL），以及电源、散热、保护等板级设计。本章给出**全局视图**，后续章节将逐模块深入。

---

## 8.1 主机侧接口（SATA / SAS / NVMe over PCIe）

### 三类主流接口

![](images/47506ddd680a.png)

<p align="center">图 8-1：SATA / SAS / NVMe over PCIe 接口对比</p>

| 接口 | 协议 | 带宽（约） | 形态 | 典型场景 |
| --- | --- | --- | --- | --- |
| **SATA III** | AHCI | 600 MB/s | 2.5"、M.2 SATA | 老设备升级、NAS |
| **SAS 12G** | SAS | 1200 MB/s | 2.5" U.2 | 企业级 HDD/SSD 混插 |
| **NVMe** | NVMe over PCIe | 4–16 GB/s | M.2、U.2、AIC | **现代主流** |

### NVMe over PCIe 物理连接

| 形态 | PCIe Lane | 常见速率 |
| --- | --- | --- |
| M.2 2280（Key M） | x4 | Gen3/Gen4/Gen5 |
| U.2 / U.3 | x4 | Gen3/Gen4 |
| Add-in Card | x4/x8/x16 | Gen3/Gen4 |
| E1.S / E3.S | x4/x8 | 数据中心 |

主机通过 **PCIe 总线** 与 SSD 控制器通信，NVMe 协议定义命令格式与队列机制（详见第五篇第 18–19 章）。

---

## 8.2 SSD 控制器（Controller / SoC）的角色

SSD 控制器（也称 **主控、SoC**）是 SSD 的「大脑」，负责协调所有模块：

![](images/17ad7e07e6ae.png)

图 8-2：SSD 控制器作为前端、后端与固件的协调中心

| 职责 | 模块 | 说明 |
| --- | --- | --- |
| **前端（Front-End）** | PCIe/NVMe 引擎 | 接收主机命令，管理 SQ/CQ |
| **后端（Back-End）** | NAND 控制器（NFC） | 向 NAND 发送 Read/Program/Erase |
| **固件 CPU** | ARM / RISC-V 核 | 运行 FTL、GC、NVMe 命令处理 |
| **加速** | ECC、DMA、AES | 硬件 offload 耗时操作 |

> 控制器详解见 第 9 章。

---

## 8.3 数据路径：Host → HMB/DDR → NAND

### 写路径（Host Write）

![](images/46dc7a47d346.png)

<p align="center">图 8-3：SSD 读写数据路径（Host ↔ DRAM/HMB ↔ NAND）</p>

| 步骤 | 位置 | 动作 |
| --- | --- | --- |
| 1 | 主机 | 通过 NVMe Write 命令发送数据 |
| 2 | PCIe DMA | 数据从主机内存 DMA 到 SSD DRAM（或 HMB） |
| 3 | 前端固件 | 解析 LBA，查询 FTL 映射表 |
| 4 | FTL | 分配空闲 PBA，更新 L2P 映射 |
| 5 | 后端固件 | 构造 NAND Program 命令 |
| 6 | NFC + DMA | 数据从 DRAM DMA 到 NAND 页 |
| 7 | ECC 引擎 | 编码 ECC，写入 OOB 区 |
| 8 | 完成 | CQ 返回主机 |

### 读路径（Host Read）

| 步骤 | 动作 |
| --- | --- |
| 1 | 主机 NVMe Read → LBA |
| 2 | FTL 查 L2P → PBA |
| 3 | NFC 发 NAND Read 命令 |
| 4 | 数据 DMA 到 DRAM，ECC 解码 |
| 5 | DMA 到主机内存，CQ 完成 |

### DRAM vs HMB

| 类型 | 位置 | 用途 |
| --- | --- | --- |
| **板载 DRAM** | SSD PCB 上 | 映射表、写缓冲（性能稳定） |
| **HMB** | 借用主机内存 | 替代 DRAM 降成本（需 NVMe 支持） |

---

## 8.4 控制路径：固件、FTL、介质管理

数据路径负责「搬运数据」，控制路径负责「决策与调度」——两者在固件中并行运行：

![](images/744e494e6158.png)

<p align="center">图 8-4：控制路径（Boot、FTL、GC/WL/BBM）与数据路径分离</p>

| 控制模块 | 职责 | 触发时机 |
| --- | --- | --- |
| **Boot 固件** | 初始化硬件、加载主 FW | 上电 |
| **FTL** | LBA→PBA 映射维护 | 每次读写 |
| **GC** | 回收无效页、Erase Block | 空闲或块紧张时 |
| **Wear Leveling** | 均匀消耗 Block P/E | 每次 Erase |
| **BBM** | 坏块替换 | 出厂扫描 / 运行时 Fail |
| **Read Refresh** | 防止 Read Disturb / Retention | 后台定时 |
| **Thermal Control** | 降速保护 | 温度超阈值 |

> 固件分层见 第 10 章，FTL 算法见第四篇。

---

## 8.5 SSD 内部模块框图（必读）

以下是 SSD 硬件架构的**总览图**，建议收藏作为后续章节的参照：

![](images/e71a94559e1b.png)

<p align="center">图 8-5：SSD 内部模块总框图（FE / BE / FTL / DRAM / NAND / PMIC / PLP）</p>

### 模块清单

| 模块 | 位置 | 功能 |
| --- | --- | --- |
| **PCIe PHY** | 控制器内 | 物理层信号 |
| **NVMe 引擎** | 控制器内 | SQ/CQ、Doorbell |
| **Firmware CPU** | 控制器内 | 运行固件 |
| **FTL 引擎** | 固件 + 硬件辅助 | 地址映射 |
| **NFC** | 控制器内 | NAND 命令调度 |
| **ECC 引擎** | 控制器内 | LDPC 编解码 |
| **DRAM** | PCB | 映射表、缓冲 |
| **NAND 颗粒** | PCB | 8–16 颗，多 Channel |
| **PMIC** | PCB | 电源管理 |
| **PLP 电容** | PCB | 断电保护（企业级） |
| **温度传感器** | PCB / 控制器 | 温控 |

### 生活类比：餐厅运营

| SSD 模块 | 餐厅类比 |
| --- | --- |
| 主机 | 顾客点餐 |
| NVMe 前端 | 服务员接单 |
| DRAM | 出菜窗口（暂存） |
| FTL | 菜单到厨房工位的映射 |
| NFC + NAND | 厨房做菜 |
| GC | 收拾用过的桌子 |
| PLP | 备用发电机（停电保护） |

## 本章小结

![](images/4424cacf669a.png)

<p align="center">图 8-6：第 8 章知识要点总览</p>

* SSD = **控制器 SoC** + **NAND 颗粒** + **DRAM** + **PCB 板级设计**
* **数据路径**：Host → DMA → DRAM → FTL 映射 → NFC → NAND
* **控制路径**：Boot → FTL/GC/WL/BBM 调度，与数据路径并行
* 现代 SSD 以 **NVMe over PCIe** 为主流接口
* 图 8-5 总框图是理解后续章节的基础

---

## 常见问题 FAQ

**Q1：SSD 控制器和 NVMe 是什么关系？**

NVMe 是**协议**（主机与 SSD 对话的规则），控制器是**硬件**（执行协议的芯片）。控制器内实现 NVMe 引擎，对外呈现为 PCIe 设备。

**Q2：没有 DRAM 的 SSD 能正常工作吗？**

可以。DRAM-less SSD 用 HMB 或纯 SRAM/闪存存映射表，成本更低，但高负载随机性能通常弱于有 DRAM 的产品。

**Q3：数据路径和控制路径可以分开理解吗？**

可以且应该分开。数据路径是「单次 I/O 的数据流」；控制路径是「固件后台线程」（GC、WL 等）。GC 运行时会占用后端带宽，影响数据路径性能——这是性能抖动的根源之一。

---

## 自测题

**1. 列出 SSD 数据路径中 Host Write 的 5 个关键步骤。**

参考答案：主机 NVMe Write → PCIe DMA 到 DRAM → FTL 查/更新 L2P → NFC 发 Program 命令 → DMA 数据到 NAND + ECC 编码 → CQ 完成。

**2. 控制路径包含哪些后台模块？各自职责是什么？**

参考答案：Boot（上电初始化）、FTL（地址映射）、GC（回收无效页）、WL（磨损均衡）、BBM（坏块管理）、Read Refresh（读刷新）、Thermal Control（温控降速）。

**3. DRAM 和 HMB 的区别？**

参考答案：DRAM 是 SSD 板载内存，稳定独立；HMB 借用主机内存作映射缓存，降成本但依赖 NVMe HMB 特性与主机配置。

---

## 延伸阅读

| 资源 | 说明 |
| --- | --- |
| 本书 第 9 章 | SoC 内部模块详解 |
| 本书 第 10 章 | 固件分层与启动 |
| 本书 第 3 章 | 主机 I/O 栈 |

---

> **下一章预告**：第 9 章 SSD 控制器（SoC）详解