# AURIX3G GTM 模块配置说明文档

| 项目 | 内容 |
|---|---|
| 模块名称 | Gtm（Generic Timer Module，通用定时器模块） |
| 配置数据模型 | `plugins/Gtm_Aurix3G/config/Gtm.xdm` |
| 模型版本 | 18.0.0（日期 2025-08-12） |
| 目标平台 | Infineon AURIX3G（TC4xx 系列） |
| 配置工具 | EB tresos Studio（DataModel 7.0） |
| AUTOSAR 规范 | R4.6.0（SWS: Specification of Gtm） |
| 软件版本 | Sw 2.30.1 / Ar 4.6.0 |
| 模块 ID / 供应商 | ModuleId = 2048 / VendorId = 17（Infineon） |
| 变体类型 | VariantPostBuild（PB，支持后构建变体） |
| 硬件类型 | 支持 GTM 与 eGTM 两种 IP（由 GtmHwUsed 选择） |

> 本文档基于 `Gtm.xdm` 数据模型整理。GTM 是 AURIX3G 最复杂的片上外设之一，参数众多（500+），本文档以"容器树 + 子模块章节 + 参数表"组织，对重复型参数（如逐通道、逐源中断使能）采用合并说明，对关键/差异化参数逐一列出。

---

## 1. 概述

### 1.1 GTM 简介

GTM 是 Infineon AURIX 系列的高性能定时器子系统，提供一个可由软件灵活配置的"定时器/PWM/输入捕获"硬件池。它内部由若干子模块（sub-unit）组成，通过 ARU（高级路由单元）互联，可脱离 CPU 自主完成复杂的电机控制、信号生成与测量任务。

### 1.2 子模块总览

| 缩写 | 全称 | 作用 |
|---|---|---|
| **CMU** | Clock Management Unit | 时钟管理单元，为 GTM 各子模块提供可配置时钟 |
| **TBU** | Time Base Unit | 时间基单元，提供 4 个全局时间戳（TS0~TS3） |
| **TIM** | Timer Input Module | 定时器输入模块，信号捕获/测量/滤波 |
| **TOM** | Timer Output Module | 定时器输出模块，PWM/信号生成 |
| **ATOM** | ARU-connected Timer Output Module | ARU 连接输出模块，与 ARU 互联的高级输出 |
| **MCS** | Multi-Channel Sequencer | 多通道序列器，可编程处理单元 |
| **CMP** | Compare Module | 比较模块（含 ABWC/TBWC） |
| **SPE** | Sensor Pattern Evaluator | 传感器模式评估器（电机位置传感器） |
| **DTM** | Dead Time Module | 死区时间模块（互补 PWM 死区） |
| **DPLL** | Digital Phase Locked Loop | 数字锁相环（电机位置/速度同步） |
| **MAP** | Map Module | 信号映射模块 |
| **ARU** | Advanced Routing Unit | 高级路由单元，子模块间数据路由 |
| **BRC** | Broadcast Module | 广播模块，一源到多目的 |
| **FIFO** | First In First Out | CPU 与 ARU 间数据缓存 |
| **F2A** | FIFO to ARU | CPU/外设数据经 FIFO 写入 ARU |

### 1.3 GTM vs eGTM

AURIX3G 上同时存在 GTM（增强型，多集群）与 eGTM（精简型）两种 IP。`GtmHwUsed`（EGTM_HW / GTM_HW）决定当前配置适用哪种硬件。**部分子模块在 eGTM 中不存在**（如 BRC、DPLL、MCS、ARU、CMP、MAP），相关容器在 EGTM_HW 下会报错或禁用——文档各处会标注此约束。

### 1.4 集群（Cluster）结构

GTM 采用多集群（Cluster 0~N）架构，每个集群内含一组 TIM/TOM/ATOM/... 等子模块实例。Cluster 0 为主集群（含 BRC/DPLL/ARU 等全局资源），其余集群为扩展。eGTM 通常单集群。

### 1.5 配置值与硬件寄存器映射

GTM 配置的一大特点是：**几乎每个参数的描述都注明其映射到的硬件寄存器与位域**，如"写入 HW 寄存器 `CLS0_TBU_CH0_BASE`，bit field `BASE`"。本文档在参数表中保留此信息（形式"→ 寄存器/位域"），便于查阅芯片手册对照。

### 1.6 通知回调函数约定

多数子模块的中断通过**通知回调函数**上报，函数原型统一为：

```c
void NotifFn(uint32 HwChannelInfo, uint32 NotifRegVal);
```

- `HwChannelInfo`（32 位打包）：bit 0–7 = 子模块集群号；bit 8–15 = GTM/EGTM 标识（0=EGTM，1=GTM）；bit 16–23 = 通道号；bit 24–31 = 未用。
- `NotifRegVal`：该子模块的中断通知寄存器值（各中断使能位的状态）。

---

## 2. 配置文件与变体说明

| 项 | 说明 |
|---|---|
| 配置类 | 绝大多数参数为 `VariantPostBuild`（PB 变体），生成到 `Gtm_PBcfg.c`。个别为 `PreCompile` 变体。 |
| 参数来源 | 均为 `INFINEON`（厂商扩展），非 AUTOSAR 标准 ECUC。 |
| 动态取值 | 多处默认值/范围写作 XPath 表达式（如 `node:pos(..)` 按容器位置取序号、`ecu:list('Gtm.Xxx')` 由器件 schema 动态提供取值列表），EB tresos 在配置时自动求值。 |
| 校验（INVALID） | GTM 含大量交叉参数约束（XPath 校验），如"某模式不适用""EGTM 不可用""通道号奇偶""分频分子不得小于分母"等，配置错误会在生成时报错。 |

---

## 3. 配置容器总览（容器树）

```
Gtm                                                      （模块定义）
│
├── GtmGeneral                                           通用配置（模块ID/错误检测/分区映射）
│   ├── GtmCommonEcucPartitionRef                        通用分区引用
│   ├── GtmEcucPartitionRef [列表]                       分区映射
│   └── GtmDemEventParameterRefs                         DEM 事件引用
│
├── CommonPublishedInformation                           发布信息（版本/供应商/模块ID）
│
├── GtmHwConfiguration [列表 1–2]                        硬件配置（选 GTM/eGTM、集群时钟）
│   │   ├── GtmHwUsed / GtmSleepModeEnable
│   │   └── GtmCluster0~9ClockDivider                    各集群时钟分频
│   │
│   └── GtmClusterConfiguration [列表]                   集群配置
│       ├── GtmClusterIndex                              集群索引
│       ├── GtmClusterClock0~7SrcSel                     集群内时钟源选择
│       ├── GtmClusterEnableXxx                          集群内各子模块使能
│       ├── GtmClusterTimAuxInput0~7                     TIM 辅助输入源
│       │
│       ├── GtmTimConfiguration                          ── TIM 配置
│       │   └── GtmTimChannelConfiguration [列表≤8]
│       │       ├── GtmTimChannelGeneral                 通用（模式/时钟/捕获源）
│       │       ├── GtmTimChannelFilterConfig            滤波
│       │       ├── GtmTimChannelTimeoutDetection        超时检测(TDU)
│       │       └── GtmTimChannelInterrupt               中断
│       │
│       ├── GtmTomConfiguration                          ── TOM 配置
│       │   ├── GtmTomTriggersForTgc [列表≤2]            TGC 时基触发
│       │   └── GtmTomChannelConfiguration [列表≤16]
│       │       ├── GtmTomChannelEnable                  使能
│       │       ├── GtmTomChannelOutput                  输出
│       │       ├── GtmTomChannelTimerParameters         定时参数（CN0/CM0/CM1/时钟）
│       │       └── GtmTomChannelInterrupt               中断
│       │
│       ├── GtmAtomConfiguration                         ── ATOM 配置
│       │   ├── GtmAtomTriggerForAgc                     AGC 时基触发
│       │   └── GtmAtomChannelConfiguration [列表≤8]
│       │       ├── GtmAtomChannelEnable / Output
│       │       ├── GtmAtomChannelTimerParameters        定时参数（含模式 SOMI/SOMC/SOMP/SOMS/SOMB）
│       │       ├── GtmAtomChannelAruConfiguration       ARU 配置
│       │       └── GtmAtomChannelInterrupt              中断
│       │
│       ├── GtmSpeConfiguration                          ── SPE 传感器模式评估
│       │   ├── GtmSpeGeneral
│       │   ├── GtmSpeInputPattern [列表≤8]
│       │   ├── GtmSpeOutputConfig [固定8]
│       │   └── GtmSpeInterrupt
│       │
│       ├── GtmMcsConfiguration                          ── MCS 多通道序列器（eGTM 无）
│       │   ├── GtmMcsGeneral
│       │   └── GtmMcsChannelConfiguration [列表≤8]
│       │
│       ├── GtmCmpConfiguration                          ── CMP 比较模块（仅 Cluster 1）
│       │   ├── GtmCmpBwcEnable                          ABWC0~11 / TBWC0~11 使能
│       │   └── GtmCmpBwcInterruptEn                     中断使能
│       │
│       ├── GtmAruConfiguration                          ── ARU 高级路由（仅 Cluster 0）
│       │   ├── GtmAruInterruptConfig
│       │   └── GtmAruDynConfig [列表≤2] → DynRoute / DynRouteSr
│       │
│       ├── GtmBrcConfiguration                          ── BRC 广播（仅 Cluster 0，eGTM 无）
│       │   ├── GtmBrcInterrupt
│       │   └── GtmBrcSrcChannel [列表≤12] → Config / DestinationSelect
│       │
│       ├── GtmDpllConfiguration                         ── DPLL 数字锁相环（仅 Cluster 0，eGTM 无）
│       │   ├── GtmDpllGeneral                           通用（模式/策略/SMC 等）
│       │   ├── GtmDpllTriggerSignal                     触发信号 → General / Profile[列表≤1024] / RamPtr
│       │   ├── GtmDpllStateSignal                       状态信号 → General / Profile[列表≤64] / RamPtr
│       │   ├── GtmDpllSubInc1SignalConfig               Sub_Inc1 配置
│       │   ├── GtmDpllSubInc2SignalConfig               Sub_Inc2 配置
│       │   ├── GtmDpllEnableAction [固定32]             动作使能
│       │   ├── GtmDpllRAMRegion2Address                 RAM 区域2 地址
│       │   ├── GtmDpllAruIdToInSigPmtr [固定32]         ARU ID 映射
│       │   └── GtmDpllInterrupt                         中断（28+ 项）
│       │
│       ├── GtmF2aConfiguration                          ── F2A（仅 Cluster 0/1，eGTM 无）
│       │   └── GtmF2aChannelConfiguration [列表≤8]
│       │
│       ├── GtmMapConfiguration                          ── MAP 映射（仅 Cluster 0，eGTM 无）
│       │   ├── GtmMapTriggerSignalConfig
│       │   └── GtmMapStateSignalConfig
│       │
│       ├── GtmFifoConfiguration                         ── FIFO 缓存（仅 Cluster 0/1，eGTM 无）
│       │   └── GtmFifoChannelConfiguration [列表≤8] → GtmFifoIrqConfiguration
│       │
│       └── GtmDtmConfiguration [列表≤6]                 ── DTM 死区时间
│           └── GtmDtmChannelConfiguration [列表≤4]
│
├── GtmCmuConfiguration                                  时钟管理单元（CLK0~8/ECLK/FXCLK/GCLK）
│
├── GtmConnectionsConfiguration                          连接配置
│   ├── GtmAdcOutConfiguration [列表]                    ADC_OUT 选择
│   ├── GtmTimInselConfiguration [列表]                  TIM 输入选择
│   ├── GtmToutselConfiguration [列表]                   TOUT 输出选择
│   ├── GtmMscInselConfiguration                         MSC 输入选择
│   │   ├── GtmMscSetOutputConfiguration [列表]
│   │   ├── GtmMscBusSigSelection [列表]
│   │   │   ├── GtmMscInlconBusSig [固定16]
│   │   │   ├── GtmMscInleconBusSig [固定16]
│   │   │   ├── GtmMscInhconBusSig [固定16]
│   │   │   └── GtmMscInheconBusSig [固定16]
│   │   └── GtmMscStreamSelection [列表]
│   ├── GtmCdtmAuxInConfiguration [列表] → GtmDtmAuxInConfiguration [固定6]
│   └── ...
│
└── GtmTbuConfiguration                                  时间基单元（Channel0~3）
```

> 标 `[列表 N–M]` 为多重实例容器；标 `[固定N]` 为多重性恒等于 N 的列表（如 CMP 的 24 个比较器）。

---

## 4. 通用配置 GtmGeneral

> **用途**：GTM 模块的总体配置参数。

### 4.1 参数清单

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| GtmModuleId | INTEGER | 2048 | 255 或 2048–4095 | GTM 模块 ID（须为 255 或 2048–4095） |
| GtmDevErrorDetect | BOOLEAN | false | true/false | 开发错误检测与上报开关（Det） |
| GtmGetVersionInfoApi | BOOLEAN | false | true/false | 是否生成 `Gtm_GetVersionInfo()` |
| GtmInitCheckApi | BOOLEAN | true | true/false | 是否生成 `Gtm_InitCheck()`（默认开启以保证安全初始化） |
| GtmInitDeInitApiMode | ENUM | GTM_MCAL_SUPERVISOR | SUPERVISOR / USER1 | Init/DeInit API 操作模式 |
| GtmSafetyErrorDetect | BOOLEAN | true | true/false | 使能/禁用 GTM 安全检查（默认开启） |
| GtmCommonEcucPartitionRef | REF（可选） | — | EcucPartition | 通用 ECUC 分区（须为 GtmEcucPartitionRef 子集；**Cluster 0 必须分配到此分区**） |
| GtmEcucPartitionRef | REF（列表 0–12） | — | EcucPartition | 模块到 ECUC 分区的映射（须已在 Rma `RmaPartitionToExecutionUnitMapping` 配置） |

### 4.2 GtmDemEventParameterRefs

> 引用 DemEventParameter，对应错误发生时通过 `Dem_ReportErrorStatus` 上报。

| 引用名 | 类型 | 指向目标 | 说明 |
|---|---|---|---|
| GTM_E_CLOCK_FAILURE | REF（可选） | DemEventParameter | CLC 使能失败的生产错误事件（OPTIONAL） |

---

## 5. 发布信息 CommonPublishedInformation

> 与 MCU 模块结构一致，此处从略。关键字段：ArVersion 4.6.0、SwVersion 2.30.1、ModuleId 2048、VendorId 17、Release（动态，派生型号）。

---

## 6. 硬件配置 GtmHwConfiguration（列表 1–2）

> **用途**：选择 GTM/eGTM 硬件类型，并配置各集群（Cluster 0–9）的时钟分频。每个实例下挂若干 `GtmClusterConfiguration`。

### 6.1 硬件配置参数

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| GtmHwUsed | ENUM | EGTM_HW | EGTM_HW / GTM_HW | 使用的硬件类型（须唯一） |
| GtmSleepModeEnable | BOOLEAN | false | true/false | 硬件睡眠模式配置 |
| GtmCluster0~9ClockDivider | ENUM | 见下 | DISABLED_CLKDIV_0 / ENABLED_CLKDIV_1 / ENABLED_CLKDIV_2 | 各集群时钟分频 → `CLS0_ARCH_CLK_CFG[CLSx_CLK_DIV]`。Cluster 0 必须始终使能（分频 1 或 2）；其余集群频率不得高于 Cluster 0；不支持该集群的硬件类型须设为 DISABLED |
| GtmClusterTimAuxInputSource | ENUM | SEL0_TOM0_TO_TOM15 | SEL0_TOM0_TO_TOM15 / SEL1_TOM0_TO_TOM7 | TIM 辅助输入源 → `CLSi_ARCH_CFG[SRC_IN_MUX]` |

> **GtmCluster0~2ClockDivider 默认 ENABLED_CLKDIV_2，GtmCluster3~9 默认 DISABLED_CLKDIV_0。**

---

## 7. 集群配置 GtmClusterConfiguration（列表）

> **用途**：单个集群的配置容器。其下挂各子模块配置（TIM/TOM/ATOM/...）。本节先列集群级参数，随后各节展开子模块。

### 7.1 集群级参数

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| GtmClusterIndex | INTEGER | node:pos | 0..(NumberOfClusters-1)，需唯一 | 集群索引 |
| GtmClusterEcucPartitionRef | REF（可选） | — | EcucPartition | 集群到分区映射（须为驱动映射子集） |
| GtmClusterClock0~7SrcSel | INTEGER | 0 | 0–2 | 集群内 CLK0~CLK7 使能信号选择 → `CLSi_CCM_CMU_CLK_CFG[CLKx_SRC]` |
| GtmClusterFxClockSrcSel | INTEGER | 0 | 0–1 | 固定时钟 FXCLK0 选择 → `CLSi_CCM_CMU_FXCLK_CFG[FXCLK0_SRC]` |
| GtmClusterEnableTim/TomSpeTdtm/AtomAdtm/Mcs/Psm/CmpMon/Brc/DpllMap | BOOLEAN | false | true/false | 集群内各子模块使能 → `CLSi_CCM_CFG[EN_xxx]`。注意各子模块的存在性约束（见下） |
| GtmClusterTimAuxInput0~7 | ENUM | SELECT_DTM_CHX_OUT0 | 见源文件 | 各 TIM 通道辅助输入源 → `CCM_TIM_AUX_IN_SRC` |

**子模块使能位的存在性约束**（重要）：
- `GtmClusterEnableBrc`：eGTM 无；GTM 中仅 Cluster 0。
- `GtmClusterEnableDpllMap`：eGTM 无；GTM 中仅 Cluster 0。
- `GtmClusterEnableCmpMon`（CMP+MON）：仅 Cluster 1 可使能。
- `GtmClusterEnableMcs`：eGTM 无；GTM 中 cluster index >6 不存在。
- `GtmClusterEnablePsm`：eGTM 无。

---

## 8. TIM 定时器输入模块（GtmTimConfiguration）

> **用途**：TIM 用于输入信号捕获、测量（PWM 脉宽/周期、脉冲计数、边沿事件等）与滤波。每集群最多 8 通道。结构：`GtmTimConfiguration → GtmTimChannelConfiguration[列表≤8]`，每通道含 General / Filter / TimeoutDetection / Interrupt 四子容器。

### 8.1 GtmTimChannelGeneral（关键参数）

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| GtmTimChannelIndex | INTEGER | node:pos | 0–7，需唯一 | TIM 通道索引 |
| GtmTimChannelEnable | BOOLEAN | false | true/false | 使能通道 → `CHx_CTRL[TIM_EN]` |
| GtmTimChannelModeSelect | ENUM | TPWM | **见下** | TIM 工作模式（核心） |
| GtmTimChannelClockSelect | ENUM | SEL0_OPT1_CLK_0 | CLK_0~CLK_7（含 SET1/SET2 两套） | 通道时钟源（CMU 可配置时钟） |
| GtmTimChannelSignalEdgeSelect | ENUM | SEL0_FALLING_EDGE | FALLING / RISING | 检测的信号沿 |
| GtmTimChannelIgnoreSignalEdgeDetect | ENUM | SEL0_USE_DSL | USE_DSL / IGNORE_DSL（双沿） | 是否忽略边沿选择并检测双沿 |
| GtmTimChannelGpr0InputSelect | ENUM | TBU_TS0 | TBU_TS0/TS1/TS2/ECNT/输入值 等 | GPR0 捕获值来源 |
| GtmTimChannelGpr1InputSelect | ENUM | TBU_TS0 | 类似 GPR0 | GPR1 捕获值来源 |
| GtmTimChannelCntsValue | INTEGER | 0 | 0–16777215 | CNTS 值（TIPM/TGPS/TSSM/TBCM 模式） |
| GtmTimChannelOneShotMode | BOOLEAN | false | true/false | 单次模式 |
| GtmTimChannelExtCaptureModeEnable | BOOLEAN | false | true/false | 外部捕获模式 → `EXT_CAP_EN` |
| GtmTimChannelExtCaptureSrc | ENUM | NEW_VAL_IRQ | 14 种源（NEW_VAL/AUX_IN/CNTOFL/...） | 外部捕获源（仅 ExtCaptureModeEnable 时） |
| GtmTimChImmMesurementStart | BOOLEAN | false | true/false | 使能即立即启动测量（仅 TPWM/TPIM） |
| GtmTimChannelSwapCapture | ENUM | SEL0 | SEL0/SEL1 | 捕获时刻交换 SMU 计数器与 GPR1（仅 TPWM/TPIM） |

**GtmTimChannelModeSelect 工作模式**（重要）：
- `SEL0_PWM_MEASUREMENT_MODE_TPWM`：PWM 测量模式（测周期/占空比）
- `SEL1_PULSE_INTEGRATION_MODE_TPIM`：脉冲积分模式
- `SEL2_INPUT_EVENT_MODE_TIEM`：输入事件模式
- `SEL3_INPUT_PRESCALER_MODE_TIPM`：输入预分频模式
- `SEL4_BIT_COMPRESSION_MODE_TBCM`：位压缩模式
- `SEL5_GATED_PERIODIC_SAMPLING_MODE_TGPS`：门控周期采样模式
- `SEL6_SERIAL_SHIFT_MODE_TSSM`：串行移位模式

### 8.2 GtmTimChannelFilterConfig（滤波）

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| GtmTimChannelFilterEnable | BOOLEAN | false | 使能滤波器 → `FLT_EN` |
| GtmTimChFilterCounterFreqSelect | ENUM | CLK_0 | 滤波计数器频率 → `FLT_CNT_FRQ`（CLK_0/1/6/7） |
| GtmTimChFilterModeForRisingEdge | ENUM | IMMEDIATE_EDGE | 上升沿滤波模式（立即传播 / 独立去毛刺） |
| GtmTimChFilterModeForFallingEdge | ENUM | IMMEDIATE_EDGE | 下降沿滤波模式 |
| GtmTimChFilterCounterModeForRisingEdge | ENUM | IMM_EDGE_PROPAGATION | 上升沿计数器模式（上下计数/立即/复位/保持） |
| GtmTimChFilterCounterModeForFallingEdge | ENUM | IMM_EDGE_PROPAGATION | 下降沿计数器模式 |
| GtmTimChFilterTimeForRisingEdge | INTEGER | 0 (0–16777215) | 上升沿滤波时间 |
| GtmTimChFilterTimeForFallingEdge | INTEGER | 0 (0–16777215) | 下降沿滤波时间 |
| GtmTimChFilterInputByLUT | ENUM | LUT_NOT_USED | 通过查找表的滤波输入 → `USE_LUT` |

### 8.3 GtmTimChannelTimeoutDetection（超时检测 TDU）

> 配置输入信号超时检测，含 3 个可级联的 slice 与启动/停止/再同步条件。

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| GtmTimChTimeoutDetectionEnable | ENUM | TIMOUT_DISABLED | 超时检测使能（禁用/上升/下降/双沿） → `TDU[TO_EN]` |
| GtmTimChStartOfTimeoutDetection | ENUM | START_ON_TDU_START_000 | TDU 启动条件 → `TDU_START`（8 种） |
| GtmTimChStopOfTimeoutDetection | ENUM | STOP_ON_TDU_TOCTRL_0 | TDU 停止条件 → `TDU_STOP`（7 种） |
| GtmTimChResyncOfTimeoutDetection | INTEGER | 0 (0–15) | TDU 再同步条件 → `TDU_RESYNC` |
| GtmTimChCmpValueSlice0/1/2 | INTEGER | 0 (0–255) | slice0/1/2 超时比较值 → `TDUV[TOV/TOV1/TOV2]` |
| GtmTimChSliceCascading | ENUM | COMBINE_SLICE0_1_2 | slice 级联方式 → `SLICING` |
| GtmTimChTimeoutClockSelect | ENUM | CLK_0 | 超时时钟 |

### 8.4 GtmTimChannelInterrupt（中断）

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| GtmTimInterruptEnableOnNewVal | BOOLEAN | false | 新测量值中断 |
| GtmTimInterruptEnableOnEcntOfl | BOOLEAN | false | ECNT 溢出中断 |
| GtmTimInterruptEnableOnCntOfl | BOOLEAN | false | CNT 溢出中断 |
| GtmTimInterruptEnableOnGprxOfl | BOOLEAN | false | GPRx 溢出中断 |
| GtmTimInterruptEnableOnToDet | BOOLEAN | false | 超时检测中断 |
| GtmTimInterruptEnableOnGlitchDet | BOOLEAN | false | 毛刺检测中断 |
| GtmTimChToDetSourceSelection | ENUM | TDU_TIMEOUT_EVT | 超时中断源 → `TODET_IRQ_SRC` |
| GtmTimNotification | FUNCTION-NAME | NULL_PTR | 通知回调函数名 |

---

## 9. TOM 定时器输出模块（GtmTomConfiguration）

> **用途**：TOM 用于 PWM 与输出信号生成，每集群最多 16 通道，2 个 TGC（全局触发单元）。结构：`GtmTomConfiguration → GtmTomTriggersForTgc[列表≤2] + GtmTomChannelConfiguration[列表≤16]`。

### 9.1 GtmTomTgcTriggerByTimebase（TGC 时基触发）

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| GtmTomTgcTimeBaseTriggerEnable | BOOLEAN | false | true/false | 使能 TGC 时基触发 → `TGC_ACT_TB[TB_TRIG]` |
| GtmTomTgcTbuTimebaseSelect | ENUM | TBU_TS0 | TBU_TS0 / TS1 / TS2 | 选择时基 → `TBU_SEL` |
| GtmTomTgcTimebaseMatchValue | INTEGER | 0 (0–16777215) | 0–0xFFFFFF | 时基触发匹配值 → `ACT_TB` |

### 9.2 GtmTomChannelEnable / Output（使能与输出）

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| GtmTomChIndex | INTEGER | node:pos (0–15) | TOM 通道索引 |
| GtmTomChEnable | BOOLEAN | false | 初始化时使能通道 → `ENDIS_STAT` |
| GtmTomChEnableOnTgcTrigger | ENUM | NO_CHANGE | TGC 触发时通道使能行为 → `ENDIS_CTRL` |
| GtmTomChFreezeEn | BOOLEAN | false | freeze 模式 → `FREEZE` |
| GtmTomChOutputEnable | BOOLEAN | false | 初始化时使能输出 → `OUTEN_STAT` |
| GtmTomChOutputEnableOnTgcTrig | ENUM | NO_CHANGE | TGC 触发时输出使能 → `OUTEN_CTRL` |
| GtmTomChOutputSignalLevel | ENUM | HIGH | 输出初始电平 → `SL`（LOW/HIGH） |
| GtmTomChOneShotModeTriggerEn | BOOLEAN | false | 单次模式触发 → `OSM_TRIG` |
| GtmTomChTrigOutputPulseLength | ENUM | ALWAYS_HIGH | 触发输出脉冲长度 → `TRIG_PULSE` |

### 9.3 GtmTomChannelTimerParameters（定时参数）

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| GtmTomChCounterValCn0 | INTEGER | 0 | 0–65535 | CN0 计数器值 → `CN0` |
| GtmTomChCompareValCm0 | INTEGER | 0 | 0–65535 | CM0 比较值（周期） → `CM0` |
| GtmTomChCompareValCm1 | INTEGER | 0 | 0–65535 | CM1 比较值（占空比） → `CM1` |
| GtmTomChCompareShadowValSr0 | INTEGER | 0 | 0–65535 | SR0 影子比较值 → `SR0` |
| GtmTomChCompareShadowValSr1 | INTEGER | 0 | 0–65535 | SR1 影子比较值 → `SR1` |
| GtmTomChClockSelect | ENUM | FIXED_CLOCK_0 | FX_CLK_0~4 / TOM_TRIGOUT / EXT_TRIGIN | 时钟源 → `CLK_SRC` |
| GtmTomChCounterCn0Reset | ENUM | COMPARE_MATCH | 比较匹配 / 前通道触发 / 外部触发 | CN0 复位条件 → `RST_CCU0`/`EXT_TRIG` |
| GtmTomChUpDownMode | ENUM | DISABLED | DISABLED / CN0到0 / 触发或CM0 / 始终 | 加减计数模式 → `UDMODE` |
| GtmTomChOneShotMode | BOOLEAN | false | true/false | 单次脉冲生成 → `OSM_TRIG` |
| GtmTomChSpeMode | BOOLEAN | false | true/false | SPE 模式（仅通道 0–7） → `SPEM` |
| GtmTomChGatedCounterMode | BOOLEAN | false | true/false | 门控计数模式（仅通道 0–7） → `GCM` |
| GtmTomChBitReversalMode | BOOLEAN | false | true/false | 位反转（仅通道 15） → `BITREV` |
| GtmTomTgcEnableForceUpdate | BOOLEAN | false | true/false | 强制更新 → `FUPD_CTRL` |
| GtmTomTgcUpdateEnableOnCn0Reset | BOOLEAN | false | true/false | CN0 复位时更新影子值 → `UPEN_CTRL` |

### 9.4 GtmTomChannelInterrupt（中断）

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| GtmTomChIntEnableOnCcu0Trigger | BOOLEAN | false | CCU0 匹配（周期）中断 → `CCU0TC_IRQ_EN` |
| GtmTomChIntEnableOnCcu1Trigger | BOOLEAN | false | CCU1 匹配（占空比）中断 → `CCU1TC_IRQ_EN` |
| GtmTomChNotification | FUNCTION-NAME | NULL_PTR | 通知回调函数名 |

---

## 10. ATOM ARU连接输出模块（GtmAtomConfiguration）

> **用途**：ATOM 与 TOM 类似但支持 ARU 互联，适合与 MCS/DPLL 等配合的高级输出。每集群最多 8 通道。结构：`GtmAtomConfiguration → GtmAtomTriggerForAgc + GtmAtomChannelConfiguration[列表≤8]`。每通道含 Enable/Output/TimerParameters/AruConfiguration/Interrupt。

### 10.1 ATOM 输出模式（核心）

`GtmAtomChModeSelect` 决定通道工作模式 → `MODE`/`SOMB`：

| 枚举值 | 模式 | 说明 |
|---|---|---|
| ATOM_SIG_OUTPUT_MODE_IMMEDIATE | SOMI | 立即输出模式 |
| ATOM_SIG_OUTPUT_MODE_COMPARE | SOMC | 比较输出模式（与时基比较） |
| ATOM_SIG_OUTPUT_MODE_PWM | SOMP | PWM 模式 |
| ATOM_SIG_OUTPUT_MODE_SERIAL | SOMS | 串行输出模式 |
| ATOM_SIG_OUTPUT_MODE_BUFFERED_COMPARE | SOMB | 缓冲比较模式 |

> 各参数的可用性高度依赖所选模式，文档/界面会按模式灰显不适用项（详见各参数 INVALID 校验）。

### 10.2 GtmAtomChannelTimerParameters（主要参数）

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| GtmAtomChIndex | INTEGER | node:pos | 0–7，需唯一 | ATOM 通道索引 |
| GtmAtomChModeSelect | ENUM | SOMI | 见 10.1 | 工作模式 |
| GtmAtomChCounterValCn0 | INTEGER | 0 | 0–16777215 | CN0 计数器值 |
| GtmAtomChCompareValCm0/Cm1 | INTEGER | 0 | 0–16777215 | CM0/CM1 比较值 |
| GtmAtomChCompareShadowValSr0/Sr1 | INTEGER | 0 | 0–16777215 | SR0/SR1 影子值 |
| GtmAtomChClockSelect | ENUM | CMU_CLK_RES_0 | CMU_CLK_0~7 / TRIGOUT / EXT_TRIGIN | 时钟源（仅 SOMP/SOMS） |
| GtmAtomChCounterCn0Reset | ENUM | CCU0_COMPARE_MATCH | CCU0匹配/前通道触发/外部触发 | CN0 复位条件（仅 SOMP） |
| GtmAtomChUpDownMode | ENUM | DISABLED | DISABLED/CN0到0/触发或CM0/始终 | 加减计数（仅 SOMP） |
| GtmAtomChOneShotMode | BOOLEAN | false | true/false | 单次模式（仅 SOMP/SOMS） |
| GtmAtomChModeControlBits | INTEGER | 0 (0–31) | 0–31 | ACB 模式控制位（SOMI/SOMS 仅 ACB0） |
| GtmAtomChTimebaseForComparison | ENUM | TBU_TS1 | TBU_TS1 / TS2 | 比较用时基（SOMC/SOMB） |
| GtmAtomChCompareStrategy | ENUM | GREATER_EQUAL_TO_CMx | ≥ / ≤ | 比较策略（SOMC/SOMB） |
| GtmAtomChSomsDoubleShiftOutput | BOOLEAN | false | true/false | 双移位输出（仅 SOMS） |
| GtmAtomChSompBitReversalMode | BOOLEAN | false | true/false | 位反转（奇数通道，仅 SOMP） |

### 10.3 GtmAtomChannelAruConfiguration（ARU 配置，eGTM 不可用）

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| GtmAtomChAruEnable | BOOLEAN | false | true/false | 使能 ARU → `ARU_EN` |
| GtmAtomChAruReadAddress0 | INTEGER | 510 | 0–511 | ARU 读地址 0 |
| GtmAtomChAruReadAddress1 | INTEGER | 510 | 0–511 | ARU 读地址 1（仅 SOMC） |
| GtmAtomChAruBlockingMode | ENUM | ABM_DISABLED | DISABLED / ENABLED | ARU 阻塞模式（仅 SOMC） |
| GtmAtomChCpuWriteRequest | BOOLEAN | false | true/false | CPU 写请求（仅 SOMC） |

### 10.4 ATOM 中断

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| GtmAtomChIntEnableOnCcu0Trigger | BOOLEAN | false | CCU0 匹配中断 |
| GtmAtomChIntEnableOnCcu1Trigger | BOOLEAN | false | CCU1 匹配中断 |
| GtmAtomChNotification | FUNCTION-NAME | NULL_PTR | 通知回调函数名 |

> ATOM 的使能/输出子容器（GtmAtomChannelEnable/Output）参数与 TOM 类似（Enable、OutputEnable、EnableOnAgcTrigger、SignalLevel、FreezeEn 等），此处不再重复。

---

## 11. SPE 传感器模式评估器（GtmSpeConfiguration）

> **用途**：用于电机控制位置传感器（如霍尔传感器）信号模式评估。结构：`GtmSpeGeneral + GtmSpeInputPattern[列表≤8] + GtmSpeOutputConfig[固定8] + GtmSpeInterrupt`。

### 11.1 GtmSpeGeneral

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| GtmSpeEnableModule | BOOLEAN | false | 使能 SPE 模块 |
| GtmSpeTimChannelsSelect | ENUM | TIM0_CH0_TO_2 | 选择使用的 TIM 通道组 |
| GtmSpeEnableInput0/1/2 | BOOLEAN | false | 使能输入 0/1/2 → `SIE0` |
| GtmSpeTriggerInputSelect | ENUM | SPE_NIPD | 触发源 → `TRIG_SEL`（NIPD / TOM 各通道触发） |
| GtmSpeEnableFastShutOff | BOOLEAN | false | 使能快速关断 |
| GtmSpeInputSigRevCntr | INTEGER | 60 (0–16777215) | 输入信号转数计数器 |
| GtmSpeInputSigRevCntrCmpVal | INTEGER | 60 (0–16777215) | 转数比较值 |
| GtmSpeOutputPatternControlCommand | ENUM | SELECT_SPE_PAT_PTR_AS_INDEX | 输出模式选择控制命令 |

### 11.2 GtmSpeInputPattern / OutputConfig / Interrupt

- **GtmSpeInputPattern**（列表≤8）：`GtmSpeIsPatternValid`（BOOLEAN）、`GtmSpeInputPattern`（INTEGER 0–7，bit0/1/2 对应 3 个 TIM 输入） → `SPEi_PAT[IPx_PAT]`。
- **GtmSpeOutputConfig**（固定 8）：`GtmSpeOutput0~7Select`（枚举：TOM_CH0_SOUR / TOM_CH1_SOUR / LEVEL_0 / LEVEL_1）、`GtmSpeFastShutoffLevel`（LOW/HIGH）。
- **GtmSpeInterrupt**：`GtmSpeEnableIntOnNewPatternDetect`、`GtmSpeEnableIntOnSpeDirBitChange`、`GtmSpeEnableIntOnWrongPattern`、`GtmSpeEnableIntOnBouncingInput`、`GtmSpeEnableIntOnRegComp`、`GtmSpeNotification`。

---

## 12. MCS 多通道序列器（GtmMcsConfiguration）

> **用途**：可编程处理单元，执行复杂时序逻辑。**eGTM 无；GTM 中 cluster index >6 不存在**。

### 12.1 GtmMcsGeneral

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| GtmMcsHaltOnAeiBusMasterError | BOOLEAN | false | AEI 总线错误时是否停止 → `HLT_AEIM_ERR` |
| GtmMcsModifiedHarvardArchitectureEnable | BOOLEAN | false | 改进型哈佛架构 → `EN_HVD` |
| GtmMcsRoutingForTimChannelEnable | BOOLEAN | false | TIM_FOUT 信号路由 → `EN_TIM_FOUT` |
| GtmMcsSchedulingScheme | ENUM | ROUND_ROBIN | 调度方案 → `SCD_MODE`（加速 / 轮询） |
| GtmMcsChannelSelectionForScheduling | ENUM | MCS_CH_0 | 参与调度的通道 → `SCD_CH` |

### 12.2 GtmMcsChannelConfiguration（列表≤8）

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| GtmMcsChannelIndex | INTEGER | node:pos (0–7) | MCS 通道索引（需唯一） |
| GtmMcsChannelInterruptEnable | BOOLEAN | false | 使能 MCS 通道 IRQ → `MCS_IRQ_EN` |
| GtmMcsChannelNotification | FUNCTION-NAME | NULL_PTR | 通知回调函数名 |

---

## 13. CMP 比较模块（GtmCmpConfiguration）

> **用途**：含 ABWC（A 组）与 TBWC（T 组）各 12 个比较器。**仅 Cluster 1 存在**。

### 13.1 参数（结构高度对称）

| 容器 | 参数 | 说明 |
|---|---|---|
| **GtmCmpBwcEnable** | GtmCmpAbwc0~11Enable、GtmCmpTbwc0~11Enable（各 BOOLEAN，默认 false） | 使能各 ABWC/TBWC 比较器 → `CMP_EN[ABWCn_EN]`/`[TBWCn_EN]` |
| **GtmCmpBwcInterruptEn** | GtmCmpAbwc0~11IrqEnable、GtmCmpTbwc0~11IrqEnable（各 BOOLEAN，默认 false） | 使能各比较器中断 → `CMP_IRQ_EN[...]` |
| 同上 | GtmCmpNotification（FUNCTION-NAME，默认 NULL_PTR） | CMP 中断通知回调函数名 |

> 共 24 个使能位 + 24 个中断使能位 + 1 个通知函数，结构完全对称。

---

## 14. ARU 高级路由单元（GtmAruConfiguration）

> **用途**：GTM 内部连接各子模块数据流的路由网络。**仅 Cluster 0 存在；eGTM 无**。

### 14.1 参数

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| GtmAruCaddrEndConfig | INTEGER | 127 (0–127) | ARU CADDR 结束值 → `ARU_CADDR_END[CADDR_END]` |

### 14.2 GtmAruInterruptConfig

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| GtmAruAccAckIrqEn | BOOLEAN | false | 访问应答中断 → `ACC_ACK_IRQ_EN` |
| GtmAruNewData0IrqEn / NewData1IrqEn | BOOLEAN | false | 调试通道 New_data 中断 |
| GtmAruNotification | FUNCTION-NAME | NULL_PTR | ARU 中断通知回调函数名 |

### 14.3 GtmAruDynConfig（动态路由，列表≤2）

> 配置动态路由模式下的 ARU 主机 ID 映射，分两个子容器：
- **GtmAruDynRouteConfig**：`GtmAruDynRouteLowReadId0/1/2`、`HighReadId3/4/5`（各 0–127）、`GtmAruDynRouteClkWait`（0–15）。
- **GtmAruDynRouteSrConfig**（影子寄存器）：`SrLowReadId6/7/8`、`SrHighReadId9/10/11`、`SrClkWait`。

> 每条路由项定义一个 ARU 主机 ID（0–127），通过 `DYN_READ_IDx` 位域写入 `ARU_DYN_ROUTE_LOW/HIGH` 寄存器。

---

## 15. BRC 广播模块（GtmBrcConfiguration）

> **用途**：将一个源（最多 12 个）广播到多个目的（每个源最多 22 个目的 + 1 个 Trashbin）。**仅 Cluster 0 存在；eGTM 无**。

### 15.1 GtmBrcInterrupt

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| GtmBrcEnableDestConfigIrq | BOOLEAN | false | 目的配置错误中断 → `DEST_ERR_IRQ_EN` |
| GtmBrcEnSrc0~11DataInconsistencyIrq | BOOLEAN | false | Source 0–11 数据不一致中断 → `DID_IRQ_EN0~11` |
| GtmBrcNotification | FUNCTION-NAME | NULL_PTR | BRC 中断通知回调函数名 |

### 15.2 GtmBrcSrcChannel（列表≤12）

| 子容器 / 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| **GtmBrcSrcChannelConfig** | | | |
| GtmBrcSourceChannelId | INTEGER | node:pos (0–11) | BRC 源通道号（需唯一） |
| GtmBrcSourceAruAddress | INTEGER | 510 (0–511) | 源数据 ARU 读地址 → `BRC_SRCx_ADDR[ADDR]` |
| GtmBrcModeSelect | ENUM | DCM（数据一致性）/ MTM（最大吞吐） | BRC 模式 → `BRC_MODE` |
| **GtmBrcSrcChannelDestinationSelect** | | | |
| GtmBrcChDestinationAddr0~21Enable | BOOLEAN | false | 各目的地址使能 → `BRC_SRCx_DEST[EN_DESTx]` |
| GtmBrcChDestinationTrashbinEnable | BOOLEAN | false | Trashbin（丢弃桶）使能 |

> **约束**：任两个源不能共用同一目的地址；启用 Trashbin 时该源的其他目的使能须关闭。

---

## 16. DPLL 数字锁相环（GtmDpllConfiguration）

> **用途**：GTM 最复杂的子模块（约 130 个参数），用于电机控制中基于位置传感器（如旋变/霍尔）的位置/速度同步，常配合 DPLL 实现精确的角度时钟。**仅 Cluster 0 存在；eGTM 无**。

> **重要**：本节为 DPLL 配置的概览。DPLL 参数众多且高度专业（涉及 PLL 锁定、Trigger/State 信号、Sub_Inc 生成、RAM 指针等），完整参数表见源文件对应行段（8096–11496）。此处列出关键容器与代表性参数。

### 16.1 GtmDpllGeneral（核心模式参数）

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| GtmDpllEnable | BOOLEAN | false | 使能 DPLL 子模块 |
| GtmDpllReferenceMode | ENUM | NORMAL_MODE | 参考模式 → `RMO`（NORMAL: TRIGGER 生成 SUB_INC1；EMERGENCY: STATE 生成） |
| GtmDpllModeAndCorrectionStrategySel | ENUM | AUTOENDMODE_NEW_PULSES_DELAY | DPLL 模式与修正策略 → `DMO`/`COA`（AUTOENDMODE / CONTINUOUS） |
| GtmDpllSynchronousMotorControl | ENUM | TRIGGER_INPUT_NOT_USED_FOR_SMC | 是否启用同步电机控制 → `SMC` |
| GtmDpllSubIncMuxSel | ENUM | MTG1_DRIVE_TBU_SUB_INC | TBU_TS1 的 sub-increment 源选择 → `subinc_mux_sel` |
| GtmDpllInputFilterPositionContent | ENUM | TIME_RELATED_VALUES | TRIGGER_FT/STATE_FT 内容（时间/位置相关） → `IFP` |
| GtmDpllLockingCondition | ENUM | LC_1_MISSING_TRIGGER | 锁定条件（仅 DPLL 禁用时可配） |
| GtmDpllEnableStateExtension | BOOLEAN | false | 使能状态扩展（最多 128 个 state 事件） → `STATE_EXT` |
| GtmDpllNormal*/GtmDpllEmergency* 系列 | BOOLEAN | false | 各模式下的预测/误差/脉冲修正控制位（影响 `DPLL_CTRL_11` 各位） |

### 16.2 GtmDpllTriggerSignal（Trigger 信号）

- **GtmDpllTriggerGeneral**：`GtmDpllTriggerSignalEnable`、`GtmDpllTriggerSignalActiveEdge`（NONE/RISING/FALLING/BOTH）、`GtmDpllTriggerEventsPerHalfScale`（默认 59，→ `TNU`）、`GtmDpllTrigSigTimeStampResolution`、`GtmDpllTriggerPlausibilityValue`、`GtmDpllMinTriggerNominalIncrement`/`Max`、`GtmDpllTriggerInputDelay` 等约 30 个参数。
- **GtmDpllTriggerProfile**（列表≤1024）：`GtmDpllTriggerInterruptEvent`（中断模式）、`GtmDpllTriggerPDValue`（物理偏差）、`GtmDpllTriggerEvtValue`（事件数）。
- **GtmDpllTriggerActualRamPointerAddress**：DT/RDT/TSF/ADT 的 Actual RAM 指针偏移。

### 16.3 GtmDpllStateSignal（State 信号）

- **GtmDpllStateGeneral**：结构与 TriggerSignal 对称，参数对应 `SSL`/`SNU`/`SYN_NS` 等位域，含常规版与 state-extension 版（二者互斥）。`GtmDpllStateEventsPerHalfScale`（默认 23，→ `SNU`）。
- **GtmDpllStateProfile**（列表≤64）、**GtmDpllStateActualRamPointerAddress**：同 Trigger 对应物。

### 16.4 GtmDpllSubInc1/2SignalConfig

| 容器 | 关键参数 | 说明 |
|---|---|---|
| **SubInc1** | GtmDpllEnableSubInc1Generator、GtmDpllPulseCntBtwnTrigEvntsMlt（默认 599，→ `MLT`）、GtmDpllSubInc1UseDirectLoadMode、GtmDpllSubInc1UsePulseCorrectionMode、GtmDpllSubInc1CounterValue 等 | Sub_Inc1 生成器配置 |
| **SubInc2** | 结构与 SubInc1 对称（含 SGE2、DLM2、PCM2 等） | Sub_Inc2 生成器配置 |

### 16.5 其他 DPLL 子容器

| 容器 | 说明 |
|---|---|
| **GtmDpllEnableAction**（固定 32） | 每项含 `EnableAction`（BOOLEAN）、`ActionPositionRequestValue`（0–16777215，→ `PSA`）、`ActionReactionTimeValue`（→ `DLA`） |
| **GtmDpllRAMRegion2Address** | `GtmDpllOffsetSizeForRegion2`（ENUM：128/256/512/1024） → `OSW[OSS]` |
| **GtmDpllAruIdToInSigPmtr**（固定 32） | `GtmDpllAruIdInfoToInputSignalPmtr`（默认 510，0–511） → `ID_PMTR_x` |
| **GtmDpllInterrupt** | 约 28 个中断使能（DPLL 使能/禁用、Trigger/State 各种错误、Sub_Inc1/2 锁定获取与丢失、Trigger Event 0–4 等） + `GtmDpllNotification` |

> **源文件问题提示**（代理核对发现，非提取错误）：部分参数的 TOOLTIP 文案疑似从其他参数复制（与 DESC 含义不符），如 `GtmDpllTriggerLastRangeMultiplyValue`、`GtmDpllVirtualIncPerHalfScale` 的 INVALID 范围与提示不一致；配置时以 DESC 与芯片手册寄存器位域为准。

---

## 17. F2A / MAP / FIFO / DTM

### 17.1 GtmF2aConfiguration（FIFO 到 ARU，仅 Cluster 0/1，eGTM 无）

**GtmF2aChannelConfiguration**（列表≤8）：

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| GtmF2aChId | INTEGER | node:pos (0–7) | F2A 通道号 |
| GtmPsmF2aCtrlStrConf | ENUM | MAPPED_TO_FIFO_Y | stream 映射（仅通道 4–7） → `STRy_CONF` |
| GtmPsmF2aChAruRdAddr | INTEGER | 510 (0–511) | ARU 读地址 → `ADDR` |
| GtmPsmF2aChStrTmode | ENUM | TRANSFER_LOW_WORD | 传输模式（低/高/双字） → `TMODE` |
| GtmPsmF2aChStrDir | ENUM | FROM_ARU_TO_FIFO | 传输方向 → `DIR` |

### 17.2 GtmMapConfiguration（映射模块，仅 Cluster 0，eGTM 无）

| 子容器 | 关键参数 | 说明 |
|---|---|---|
| **GtmMapTriggerSignalConfig** | GtmMapTriggerSignalSelect、GtmMapEnableTSPP0、GtmMapDisableTim0Ch0~2Input、GtmMapTspp0DirLevelDefinition | 触发信号映射 → `CLS0_MAP_CTRL` |
| **GtmMapStateSignalConfig** | GtmMapStateSignalSelect、GtmMapEnableTSPP1、GtmMapDisableTim0Ch3~5Input、GtmMapTspp1DirSignalLevelDefinition | 状态信号映射 |

### 17.3 GtmFifoConfiguration（FIFO 缓存，仅 Cluster 0/1，eGTM 无）

**GtmFifoChannelConfiguration**（列表≤8）→ **GtmFifoIrqConfiguration**：

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| GtmFifoChId | INTEGER | node:pos (0–7) | FIFO 通道号 |
| GtmPsmFifoChRbmEn | BOOLEAN | false | 环形缓冲模式 → `RBM` |
| GtmPsmFifoChRap | ENUM | FIFO_PRIORITY_HIGH | 通道优先级 → `RAP` |
| GtmPsmFifoChStartAddr/EndAddr | INTEGER | 0 / 0 (0–1023) | 起始/结束地址 |
| GtmPsmFifoChUpperWm/LowerWm | INTEGER | 96 / 32 (0–1023) | 上/下水位地址 |
| **GtmFifoIrqConfiguration** | | | |
| GtmPsmFifoChEmptyIrqEn/FullIrqEn/LwmIrqEn/UwmIrqEn | BOOLEAN | false | 空/满/下水位/上水位中断 → `*_IRQ_EN` |
| GtmPsmFifoChDmaHystEn | BOOLEAN | false | DMA 滞后使能 |
| GtmFifoChNotification | FUNCTION-NAME | NULL_PTR | FIFO 通道中断通知回调函数名 |

### 17.4 GtmDtmConfiguration（死区时间模块，列表≤6）

> **用途**：生成互补 PWM 死区时间，用于电机驱动。每实例最多 4 通道。

**模块级参数**（GtmDtmConfiguration）：

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| GtmDtmIndex | INTEGER | node:pos (0–5) | DTM 模块号（需唯一） |
| GtmDtmClockSelect | ENUM | CLS_CLK_RES | 时钟源 → `DTMd_CTRL[CLK_SEL]` |
| GtmDtmUpdateMode | ENUM | ASYNCHRONOUS_UPDATE | 更新模式（8 种，含关断释放方式） → `UPD_MODE` |
| GtmDtmChShutOffEnable | BOOLEAN | false | 通道独立关断使能 → `CH_SHUTOFF_EN` |
| GtmDtmCrossDtCh0Ch1Enable/CrossDtCh2Ch3Enable | BOOLEAN | false | 通道 0↔1 / 2↔3 交叉死区生成 |
| GtmDtmPsuBlankWindowRelVal | INTEGER | 0 (0–1023) | 相位移位消隐窗口重载值 → `RELBLK` |
| GtmDtmPsuInputSelect/TimInputSelect/ShiftSelect | ENUM | — | PSU 输入/TIM 输入/移位触发选择 |

**GtmDtmChannelConfiguration**（列表≤4，每通道 OUT0/OUT1 两路输出 + 死区值）：

| 参数组 | 代表参数 | 说明 |
|---|---|---|
| OUT0/OUT1 极性与电平 | Out0Polarity/Out1Polarity、Out0Control/Out1Control、Out0SignalLevel/Out1SignalLevel | 输出极性/控制/电平 → `POL/OC/SL` |
| 死区路径 | Out0DtEnable/Out1DtEnable | 是否经死区路径 → `DT0/DT1` |
| 死区时间值 | RisingEdgeDTValue/FallingEdgeDTValue（0–8191） | 上升/下降沿死区值 → `RELDRISE/RELFALL` |
| 影子死区值 | RisingEdgeDTShadowValue/FallingEdgeDTShadowValue、RelEnable、RelEdge | 死区影子寄存器与重装载触发 |
| 关断 | ShutOffSignalSelect、ShutOffSignalInvert、ShutOffUpdMode、ShutOffResetEnable | 通道关断信号选择与释放 |

---

## 18. CMU 时钟管理单元（GtmCmuConfiguration）

> **用途**：GTM 内部时钟核心，提供 CLK0~CLK8、外部时钟 ECLK0~2、固定时钟 FXCLK、全局时钟 GCLK 等多组可配置时钟，供各子模块（TIM/TOM/ATOM...）选用。这是 GTM 时钟体系的源头。

### 18.1 参数（结构对称）

| 参数组 | 参数模式 | 说明 |
|---|---|---|
| **CLK0~CLK5** | XClkCnt（INTEGER 0–16777215，分频计数）、XEnable（BOOLEAN）、XSrcSel（INTEGER 0–1，外部分频器选择） | 可配置时钟 0–5 → `CMU_CLKx_CTRL[CLK_CNT]`、`CLK_EN[EN_CLKx]`、`CLK_CTRL[x_EXT_DIVIDER]` |
| **CLK6/CLK7** | 除上述外含 XClkSel（CLK6: 0–3；CLK7: 0–2） | 时钟源选择 → `CLK_6/7_CTRL[CLK_SEL]` |
| **CLK8** | SrcSel（0–1） | 仅时钟源选择 |
| **ECLK0~2**（外部时钟） | XDen/XNum（INTEGER 1–16777215）、XEnable（BOOLEAN） | 分频分子/分母 + 使能 → `ECLKx_NUM/DEN`。**约束：分子 ≥ 分母** |
| **FXCLK** | FxClkEnable（BOOLEAN）、FxClkSel（INTEGER 0–8） | 固定时钟使能与选择 → `FXCLK_CTRL[FXCLK_SEL]` |
| **GCLK**（全局时钟） | GclkDen/GclkNum（1–16777215） | 全局时钟分频分子/分母。**约束：分子 ≥ 分母** |

> CLK0~7 的默认 Enable 均为 false，ClkCnt/SrcSel 默认 0；ECLK/GCLK 的 Num/Den 默认 1。

---

## 19. 连接配置 GtmConnectionsConfiguration

> **用途**：配置 GTM 内部信号连接路由（将 GTM 输出连到芯片引脚 TOUT、将引脚输入连到 TIM、ADC 输出选择、MSC 信号选择等）。

### 19.1 输入/输出选择（列表容器）

| 容器 | 参数 | 说明 |
|---|---|---|
| **GtmAdcOutConfiguration** | GtmAdcOutselID（唯一）、GtmAdcOutselSelection（动态） | ADC_OUT 选择 → `ADC_OUT[x/4].SEL[x%4]` |
| **GtmTimInselConfiguration** | GtmTimInselID（唯一）、GtmTimInselSelection（动态） | TIM 输入选择 → `TIM[x/8]INSEL.CH[x%8]SEL` |
| **GtmToutselConfiguration** | GtmToutselID（唯一）、GtmToutselSelection（动态） | TOUT 输出选择 → `TOUTSEL[x/6].SEL[x%6]` |

> "动态"取值由器件 schema 的 EcuList 提供（依据 GtmHwUsed 选 Gtm.* 或 Egtm.* 项集）。

### 19.2 MSC 输入选择（GtmMscInselConfiguration）

| 子容器 | 参数 | 说明 |
|---|---|---|
| **GtmMscSetOutputConfiguration** | GtmMscSetOutputSelId、GtmMscSetOutputSel（动态） | MSCSET 输出选择 → `MSCSETi_CONj.SEL` |
| **GtmMscBusSigSelection** | GtmMscBusSigSelectionId | 总线信号配置容器 |
| ├ **GtmMscInlconBusSig**（固定16） | SelId、Sel（动态） | MSC_INLCON 信号选择 |
| ├ **GtmMscInleconBusSig**（固定16） | SelId、Sel（动态） | MSC_INLECON 信号选择 |
| ├ **GtmMscInhconBusSig**（固定16） | SelId、Sel（动态） | MSC_INHCON 信号选择 |
| └ **GtmMscInheconBusSig**（固定16） | SelId、Sel（动态） | MSC_INHECON 信号选择 |
| **GtmMscStreamSelection** | SelectionId + Inlcon/Inlecon/Inhcon/InheconStreamSel（动态） | Egtm→MSC 的 4 类流输入选择 |

### 19.3 DTM 辅助输入（GtmCdtmAuxInConfiguration）

| 容器 | 参数 | 说明 |
|---|---|---|
| **GtmCdtmAuxInConfiguration** | GtmCdtmId（集群号，唯一） | CDTM 模块集群编号 |
| └ **GtmDtmAuxInConfiguration**（固定6） | GtmDtmId（0–5）、GtmDtmAuxInput0/1Select（动态） | 各 DTM 模块的辅助输入 0/1 信号选择 |

---

## 20. TBU 时间基单元（GtmTbuConfiguration）

> **用途**：提供 4 个全局时间基通道（TS0~TS3），供 TIM 捕获时间戳、TOM/ATOM 触发比较、DPLL 等使用。

### 20.1 参数（4 通道）

| 通道 | BaseValue | ClockSource | Enable | Resolution/Mode | 说明 |
|---|---|---|---|---|---|
| **CH0** | INTEGER 0–16777215（默认0）→ `BASE` | INTEGER 0–7（默认0）→ `CH_CLK_SRC` | BOOLEAN（默认false）→ `ENDIS_CH0` | ENUM: GTM_TBU_RES_LOWER/UPPER（默认LOWER）→ `LOW_RES` | 基础时间基 |
| **CH1** | 同上 → `CH1_BASE` | 同上 → `CH1_CTRL` | 同上 → `ENDIS_CH1` | ENUM: GTM_TBU_FREE_RUNNING / GTM_TBU_FWD_BKWD（默认FREE_RUNNING）→ `CH_MODE` | 可前向/后向 |
| **CH2** | 同上 → `CH2_BASE` | 同上 → `CH2_CTRL` | 同上 → `ENDIS_CH2` | ENUM: 同 CH1（默认FREE_RUNNING）→ `CH_MODE` | 可前向/后向 |
| **CH3** | 同上 → `CH3_BASE` | — | BOOLEAN（默认false）→ `ENDIS_CH3` | BaseMark（0–16777215）→ `BASE_MARK`；ModuloCtrChSelect（0–1）→ `USE_CH2` | **eGTM 无 CH3，须禁用且参数为 0** |

> **EGTM 约束**：CH3 的 BaseValue/BaseMark/ModuloCtrChSelect/Enable 在 EGTM_HW 下不可改默认值（强制 0/false），配置时由校验拦截。

---

## 21. 关键配置流程与建议

### 21.1 GTM 配置总体顺序（推荐）

1. **GtmGeneral**：设置模块 ID、错误检测、API 开关、ECUC 分区映射。
2. **GtmHwConfiguration**：选 `GtmHwUsed`（GTM/eGTM）、各集群时钟分频（Cluster 0 必使能）。
3. **GtmClusterConfiguration**：配置每个集群的子模块使能（`GtmClusterEnableXxx`）与时钟源选择。
4. **GtmCmuConfiguration**：配置 CMU 各组时钟（CLK0~8/ECLK/FXCLK/GCLK）的分频与使能——**这是子模块时钟的源头，必须先配好**。
5. **GtmTbuConfiguration**：配置时间基通道（TIM/TOM/ATOM/DPLL 依赖时间戳）。
6. **各子模块**（按需）：TIM（输入）→ TOM/ATOM（输出）→ SPE/DPLL（电机控制）→ MCS/DTM → ARU/BRC/FIFO（数据路由）。
7. **GtmConnectionsConfiguration**：配置引脚映射（TOUT/TIMINSEL）、MSC/ADC 连接。
8. **验证**：解决 EB tresos 报出的所有校验错误（模式约束、唯一性、EGTM 可用性等）后生成代码。

### 21.2 时钟配置要点

- **CMU 是核心**：所有子模块的"可配置时钟"均来自 CMU 的 CLK0~CLK8。CMU 时钟未使能或分频不当，子模块将无时钟。
- **FXCLK**：TOM 默认使用 FX_CLK_0（`GtmTomChClockSelect` 默认 FIXED_CLOCK_0），需确保 `GtmCmuFxClkEnable=true`。
- **集群频率约束**：Cluster x（x≠0）频率不得高于 Cluster 0；Cluster 0 须分频 1 或 2。

### 21.3 模式相关配置（ATOM/TIM/DPLL）

- **ATOM 模式**：`GtmAtomChModeSelect`（SOMI/SOMC/SOMP/SOMS/SOMB）决定后续大量参数的可用性，配置时先选模式再配参数。
- **TIM 模式**：`GtmTimChannelModeSelect`（TPWM/TPIM/TIEM/TIPM/TBCM/TGPS/TSSM）类似。
- **DPLL**：参数极多且专业，建议参考 AURIX GTM 手册的 DPLL 章节，按 Trigger/State/SubInc 分块配置；注意 `GtmDpllEnableStateExtension` 会切换一组互斥参数（常规版 vs 扩展版）。

### 21.4 eGTM 兼容性

配置用于 eGTM（`GtmHwUsed=EGTM_HW`）时，以下子模块不可用（会报错或禁用）：**BRC、DPLL、MCS、ARU、CMP、MAP、F2A、FIFO**，以及 **TBU CH3**。设计多变体时需注意 GTM/eGTM 配置差异。

---

## 22. 重要约束与注意事项

1. **硬件存在性**：各子模块仅存在于特定集群或特定硬件类型，详见各章节标注（如 CMP 仅 Cluster 1、BRC/DPLL/ARU 仅 Cluster 0、诸多模块 eGTM 无）。配置不符会被校验拦截。
2. **唯一性校验**：各类 Index（TIM/TOM/ATOM/MCS/BRC Source/DTM/FIFO/F2A 等）须在同一范围内唯一；多数还会校验"从 0 连续"或"取最小可用值"。
3. **通道适用范围**：部分参数仅特定通道可用（如 TOM 的 SpeMode 仅通道 0–7、BitReversalMode 仅通道 15、ATOM 的 BitReversal 仅奇数通道）。
4. **分频约束**：CMU 外部时钟与全局时钟的分子（Num）不得小于分母（Den）。
5. **通知回调签名**：所有 `*Notification` 须遵循统一的 `void NotifFn(uint32 HwChannelInfo, uint32 NotifRegVal)` 原型；HwChannelInfo 的位域含义见 §1.6。
6. **交叉参数依赖**：GTM 含大量 XPath 校验约束（如 ATOM 某参数仅 SOMP 模式可用、SR0 触发依赖 CounterCn0Reset 配置、BRC Trashbin 与目的使能互斥等），配置错误会给出明确提示。
7. **源文件已知问题**：DPLL 部分 INTEGER 参数的 TOOLTIP 提示文案疑似复制错误（范围描述与实际 INVALID 不符），配置时以参数 DESC、INVALID 范围与芯片手册寄存器位域为准。
8. **动态取值**：标注"动态"的枚举/字符串参数（尤其 Connections 配置）由器件 schema 提供，实际可选值随派生器件与 GtmHwUsed 变化。

---

## 附录 A：子模块—存在性约束速查

| 子模块 | 仅 Cluster 0 | 仅 Cluster 1 | eGTM 无 | 备注 |
|---|---|---|---|---|
| BRC | ✅ | — | ✅ | 广播 |
| DPLL | ✅ | — | ✅ | 数字锁相环 |
| ARU | ✅ | — | ✅ | 路由 |
| MAP | ✅ | — | ✅ | 映射 |
| MCS | ✅（cluster≤6） | — | ✅ | 多通道序列器 |
| CMP/MON | — | ✅ | — | 比较 |
| F2A | ✅（cluster 0/1） | — | ✅ | FIFO→ARU |
| FIFO | ✅（cluster 0/1） | — | ✅ | 缓存 |
| PSM | — | — | ✅ | 外设状态机 |
| TBU CH3 | — | — | ✅（eGTM 须禁用） | 时间基通道3 |
| TIM/TOM/ATOM/SPE/DTM | 各集群均有 | — | 部分有 | 通用子模块 |

---

## 附录 B：容器—参数规模统计

| 容器 | 参数量级 | 关键内容 |
|---|---|---|
| GtmGeneral | ~8 | 模块ID/错误检测/分区 |
| GtmHwConfiguration（含集群时钟） | ~13 | 硬件类型/集群分频 |
| GtmClusterConfiguration | ~25 | 集群使能/时钟源/辅助输入 |
| GtmTimChannelConfiguration | ~45 | TIM 通用/滤波/超时/中断 |
| GtmTomChannelConfiguration | ~30 | TOM 使能/输出/定时/中断 |
| GtmAtomChannelConfiguration | ~40 | ATOM 全模式/ARU/中断 |
| GtmSpeConfiguration | ~30 | SPE 通用/输入模式/输出/中断 |
| GtmMcsConfiguration | ~12 | MCS 通用/通道 |
| GtmCmpConfiguration | ~49 | ABWC/TBWC 使能与中断（各12） |
| GtmAruConfiguration | ~20 | ARU 中断/动态路由 |
| GtmBrcConfiguration | ~50 | BRC 中断/源通道/目的选择 |
| GtmDpllConfiguration | ~130 | DPLL 模式/Trigger/State/SubInc/动作/RAM/中断 |
| GtmF2aConfiguration | ~6 | F2A 通道 |
| GtmMapConfiguration | ~12 | MAP 触发/状态 |
| GtmFifoConfiguration | ~15 | FIFO 通道/水位/中断 |
| GtmDtmConfiguration | ~50 | DTM 模块级/通道死区 |
| GtmCmuConfiguration | ~45 | CMU 各组时钟 |
| GtmConnectionsConfiguration | 动态 | 各类信号选择 |
| GtmTbuConfiguration | ~15 | 4 个时间基通道 |

---

> **文档来源**：基于 `plugins/Gtm_Aurix3G/config/Gtm.xdm`（VERSION 18.0.0, 2025-08-12, Infineon AURIX3G）整理，覆盖全部配置容器与参数。参数默认值/范围中标注"动态"或 XPath 表达式者，由所选派生器件在 EB tresos 中求值，请以实际配置界面值为准；具体寄存器/位域映射请对照 AURIX3G GTM 硬件手册。
