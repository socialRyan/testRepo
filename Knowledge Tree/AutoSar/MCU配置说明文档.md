# AURIX3G MCU 模块（Mcu）配置说明文档

| 项目 | 内容 |
|---|---|
| 模块名称 | Mcu（Microcontroller Unit，微控制器驱动） |
| 配置数据模型 | `plugins/Mcu_Aurix3G/config/Mcu.xdm` |
| 模型版本 | 20.0.0（日期 2025-09-04） |
| 目标平台 | Infineon AURIX3G（TC4xx 系列） |
| 配置工具 | EB tresos Studio（DataModel 7.0） |
| AUTOSAR 规范 | R4.6.0（SWS: Specification of MCU） |
| 软件版本 | Sw 2.30.1 / Ar 4.6.0（见 CommonPublishedInformation） |
| 模块 ID / 供应商 | ModuleId = 101 / VendorId = 17（Infineon） |
| 变体类型 | VariantPostBuild（PB，支持后构建变体） |
| 预置配置 | `plugins/Mcu_Aurix3G/config/McuPreConfiguration.xdm`（V2.0.0） |

> 本文档基于 `Mcu.xdm` 数据模型整理，覆盖该模块全部配置容器与参数。文档面向 AUTOSAR MCAL 配置工程师，用于理解每个配置项的含义、取值与依赖关系。

---

## 1. 概述

### 1.1 MCU 模块职责

MCU 驱动是 AUTOSAR MCAL（微控制器抽象层）的基础服务模块之一，负责微控制器级别的初始化与运行时管理，主要包括：

- **时钟系统（Clock）**：振荡器、PLL（系统/外设/音频/PPU）、各总线与外设频率、时钟分发、外部时钟输出、时钟监控的配置与初始化。
- **复位管理（Reset）**：复位原因获取、复位状态查询、应用/系统复位触发、各类复位源（SW / ESR / SMU / STM / PHY / PIN）的复位行为配置。
- **低功耗模式（Low Power / Standby）**：Idle、Sleep、Standby 等电源模式的进入与唤醒源（ESR / PIN / RTC / WUT / 电源）配置。
- **RAM 管理**：RAM 扇区的初始化值、基地址、大小与分区映射。
- **模式管理（MCU Mode）**：定义不同电源模式下 MCU 的工作参数。
- **DEM 事件上报**：时钟失效、初始化失败、寄存器回读失败等生产错误的 Dem 事件引用。
- **地址翻译（可选）**：全局/本地 DSPR/PSPR 地址转换 API。

### 1.2 在 MCAL 中的位置

MCU 是最先被初始化的模块之一（通常由 EcuM 在启动阶段调用 `Mcu_Init`）。它为 Port、Gpt、Icu、Pwm、Can、Spi、Adc、Fr、Eth 等几乎所有外设驱动提供**时钟参考点（McuClockReferencePoint）**——其他模块通过引用 MCU 时钟参考点获取输入频率。因此 MCU 时钟树配置的正确性是整个 MCAL 正常运行的前提。

### 1.3 关键 API（由 `McuGeneralConfiguration` 中各 `*Api` 开关控制可用性）

`Mcu_Init` / `Mcu_InitCheck` / `Mcu_DeInit` / `Mcu_GetVersionInfo` / `Mcu_InitClock` / `Mcu_DistributePllClock` / `Mcu_GetPllStatus` / `Mcu_GetResetReason` / `Mcu_GetResetStatus` / `Mcu_PerformReset` / `Mcu_SetMode` / `Mcu_GetRamState` / `Mcu_SetCpuFrequency` / `Mcu_StartupClockInit` / `Mcu_StandbyEntryOnHwEvent` / `Mcu_TriggerGroupReset` 等。

---

## 2. 配置文件与变体说明

| 文件 | 版本 | 作用 |
|---|---|---|
| `Mcu.xdm` | 20.0.0 | **配置数据模型（schema/ECUC 参数定义）**，定义所有可配置容器、参数、引用、取值范围与校验规则。是 EB tresos 生成配置 GUI 与配置代码（`Mcu_Cfg.h/.c`、`Mcu_PBcfg.c`）的依据。 |
| `McuPreConfiguration.xdm` | 2.0.0 | **预置数据模型**，提供一组开箱即用的默认配置（典型时钟、复位等），可作为新配置的起点。 |

### 2.1 配置类（IMPLEMENTATIONCONFIGCLASS）

本模块参数均为 `VariantPostBuild`（PB 变体），意味着参数值在**后构建阶段**可变，最终生成到 `Mcu_PBcfg.c` 中的后构建配置结构体，支持运行时多变体切换。

### 2.2 参数来源（ORIGIN）

- `AUTOSAR_ECUC`：源自 AUTOSAR 标准 ECUC 规范，跨厂商通用。
- `INFINEON`：Infineon 厂商扩展参数，针对 AURIX3G 硬件特性。

### 2.3 动态取值说明

部分参数默认值或取值范围写作 XPath 表达式（如 `ecu:get('Clock.SysPllMaxFrequency')`、`ecu:list('Clock.ExtClockOutSel0')`），表示该值由**具体派生器件（die，如 TC49A / TC4Dx 等）的 Clock schema 在生成时动态求值**。配置时 EB tresos 会根据所选器件自动给出可用范围与默认值。

---

## 3. 配置容器总览（容器树）

```
Mcu                                       （模块定义 / MODULE-DEF）
│
├── McuGeneralConfiguration               通用配置：API 开关、安全/错误检测、分区映射
│   ├── McuCommonClockConfig              公共时钟资源：ADC/CDSP/HSPHY 睡眠与功能选择
│   └── McuLowPowerModeConfig             低功耗模式：触发 idle/standby/system 模式的核与进入条件
│
├── McuModuleConfiguration                模块配置：模式数、复位设置、时钟树
│   ├── McuClockSrcFailureNotification    (参数) 时钟失效通知
│   ├── McuNumberOfMcuModes / McuRamSectors / McuResetSetting   (保留参数)
│   │
│   ├── McuClockSettingConfig  [列表 1–256]   时钟设置集合（每个含一个时钟参考点 + 时钟系统）
│   │   ├── McuClockSettingId                时钟设置 ID（= Mcu_InitClock 参数）
│   │   ├── McuClockReferencePoint [列表 1–256]  时钟参考点（频率，供其他模块引用）
│   │   └── McuClockingSystemConfig          时钟系统配置（核心时钟树）
│   │       ├── McuClockMonitorConfig            时钟监控（PLL/分频器/Backup 存活监控）
│   │       ├── McuExternalClockOutputConfig     外部时钟输出（EXTCLK0/1）
│   │       ├── McuPeripheralPllSettingConfig    外设 PLL（PLL1/2/3 频率与分频）
│   │       ├── McuPllDistributionSettingConfig  PLL 分发（各总线/外设目标频率）
│   │       ├── McuSystemPllSettingConfig        系统 PLL（主时钟 + PPU）
│   │       └── McuAudioPllSettingConfig         音频 PLL
│   │
│   ├── McuDemEventParameterRefs          DEM 事件引用集合（时钟失效/初始化失败/回读失败）
│   │
│   └── McuModeSettingConf   [列表 1–3]   MCU 模式设置（McuMode 0/1/2）
│       ├── McuMode                          模式编号（0=Normal, 2=Standby，需唯一）
│       ├── McuStandbyModeSettingConf        待机模式进入/供电配置（仅 McuMode=2）
│       └── McuStandbyModeWakeupConf         待机模式唤醒源配置（仅 McuMode=2）
│
├── McuRamSectorSettingConf  [列表 0–256] RAM 扇区设置（地址/大小/初始化值/分区）
│
├── McuResetSettingConf                   复位设置（SW/ESR/SMU/STM/PHY/PIN 复位行为）
│
└── McuPublishedInformation               发布信息
    ├── McuResetReasonConf  [固定 10 项]    复位原因符号（供 EcuM 引用）
    └── CommonPublishedInformation          供应商/版本/模块 ID 信息
```

> **说明**：标 `[列表 N–M]` 的容器为多重实例容器（MAP/列表），可在配置中创建多个实例；其余为单实例容器。

---

## 4. 通用配置 McuGeneralConfiguration

> **用途**：包含 MCU 驱动的总体配置参数，主要是各类 API 的使能开关、开发/安全错误检测开关，以及 ECUC 分区映射。这些参数通常决定生成代码的体积与功能裁剪。

### 4.1 参数清单

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| McuDevErrorDetect | BOOLEAN | false | true/false | 开发错误检测与通知开关（Det 上报） |
| McuGetRamStateApi | BOOLEAN | false | true/false | 是否生成 `Mcu_GetRamState` API |
| McuInitClock | BOOLEAN | true | true/false | MCU 是否负责时钟初始化；FALSE 时禁用时钟初始化（适用于存在 Bootloader 且时钟寄存器为 write-once 的场景） |
| McuNoPll | BOOLEAN | false | true/false | 硬件无 PLL 或 PLL 上电自启时置 true，此时禁用 `Mcu_DistributePllClock`，`Mcu_GetPllStatus` 返回 UNDEFINED（**不可编辑**，由硬件决定） |
| McuPerformResetApi | BOOLEAN | false | true/false | 是否生成 `Mcu_PerformReset` API |
| McuVersionInfoApi | BOOLEAN | false | true/false | 是否生成 `Mcu_GetVersionInfo` API |
| McuInitDeInitApiMode | ENUM | MCU_MCAL_SUPERVISOR | MCU_MCAL_SUPERVISOR / MCU_MCAL_USER1 | Init/DeInit API 的操作模式（Supervisor / User1） |
| McuRuntimeApiMode | ENUM | MCU_MCAL_SUPERVISOR | MCU_MCAL_SUPERVISOR / MCU_MCAL_USER1 | 运行时 API 的操作模式 |
| McuSafetyErrorDetect | BOOLEAN | true | true/false | 使能/禁用 MCU 驱动的安全检查与特性（默认开启） |
| McuTriggerGroupResetApi | BOOLEAN | false | true/false | 是否生成 `Mcu_TriggerGroupReset` API |
| McuAddressTranslationApi | BOOLEAN | false | true/false | 是否生成地址翻译 API（GetGlobal/Local Dspr/Pspr Address） |
| McuSetCpuFrequencyApi | BOOLEAN | false | true/false | 是否生成 `Mcu_SetCpuFrequency` API |
| McuClearColdResetStatusApi | BOOLEAN | false | true/false | 是否生成 `Mcu_ClearColdResetStatus` API |
| McuDeInitApi | BOOLEAN | false | true/false | 是否生成 `Mcu_DeInit` API |
| McuInitCheckApi | BOOLEAN | true | true/false | 是否生成 `Mcu_InitCheck` API（为保证安全初始化，默认 true） |
| McuLowPowerModeApi | BOOLEAN | false | true/false | 是否生成低功耗模式相关 API |
| McuClearIntSMURstFlag | BOOLEAN | true | true/false | 是否清除内部 SMU 复位请求标志；清除可避免看门狗超时连续触发应用复位导致的永久复位（默认 true） |
| McuResetStatusApi | BOOLEAN | false | true/false | 是否生成复位状态查询 API |
| McuHsphyInitEnable | BOOLEAN | false | true/false | `Mcu_Init` 是否初始化 HSPHY 特性（不影响代码生成，仅控制初始化行为） |
| McuEruEifiltConfig | INTEGER | 0 | 0–4294967295 | ERU 输入滤波配置 |
| McuStandbyEntryOnHwEventApi | BOOLEAN | false | true/false | 是否生成 `Mcu_StandbyEntryOnHwEvent` API（基于硬件事件进入待机） |
| McuStartupClockInitApi | BOOLEAN | false | true/false | 是否生成 `Mcu_StartupClockInit` API（启动阶段时钟初始化） |
| McuDefaultClockSettingId | INTEGER | 0 | 0–255 | 默认时钟设置 ID（仅当 `McuStartupClockInitApi=true` 时可配置非 0） |
| McuEcucPartitionRef | REF（列表） | — | 指向 EcuC/EcucPartition（最多 32） | 将 MCU 驱动 API 映射到一个或多个 ECUC 分区；引用的分区必须在 Rma 的 `RmaPartitionToExecutionUnitMapping` 中已配置 |
| McuCommonPartitionRef | REF（可选） | — | 指向 EcuC/EcucPartition | 指定被授权初始化系统寄存器的分区；必须已包含在 McuEcucPartitionRef 中 |

### 4.2 子容器：McuCommonClockConfig

> **用途**：保存配置 ADC、CDSP、HSPHY 等模块时钟资源所需的参数（多与睡眠/PHY 功能选择相关）。

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| McuAdcSleepModeEnabled | BOOLEAN | false | true/false | 系统睡眠时 ADC 是否进入睡眠 |
| McuCdspGlobalCLockEnabled | BOOLEAN | false | true/false | CDSP 全局时钟是否使能（仅当 `Clock.CDSPAvailable>0` 可编辑） |
| McuHSPHYSleepModeEnabled | BOOLEAN | false | true/false | 系统睡眠时 HSPHY 是否进入睡眠（仅当 `McuHsphyInitEnable=true` 可配置） |
| McuHsphyFsp0Config | ENUM | MCU_HSPHY_FSP0_XGETH_SEL0 | XGETH_SEL0 / PCIE_SEL1（或 SGBT_SEL1） | PHY0 功能选择：GETH / PCIe（或 SGBT），取值随器件动态 |
| McuHsphyFsp1Config | ENUM | MCU_HSPHY_FSP1_XGETH_SEL0 | XGETH_SEL0 / SGBT_SEL1 | PHY1 功能选择 |
| McuHsphyFsp2Config | ENUM | (动态) | PCIE_SEL0 / SGBT_SEL1 | PHY2 功能选择 |
| McuHsphyPrs0Config | ENUM | MCU_HSPHY_PRS_PHY0_ALT_CLK_SEL0 | ALT_CLK_SEL0 / PAD_CLK_SEL1 | PHY0 参考时钟源：ALT_CLK / PAD_CLK |
| McuHsphyPrs1Config | ENUM | MCU_HSPHY_PRS_PHY1_ALT_CLK_SEL0 | ALT_CLK_SEL0 / PAD_CLK_SEL1 | PHY1 参考时钟源（仅 McuHsphyInitEnable=true 可配） |
| McuHsphyPrs2Config | ENUM | MCU_HSPHY_PRS_PHY2_ALT_CLK_SEL0 | ALT_CLK_SEL0 / PAD_CLK_SEL1 | PHY2 参考时钟源（仅 McuHsphyInitEnable=true 可配） |

### 4.3 子容器：McuLowPowerModeConfig

> **用途**：保存低功耗模式配置，定义触发 idle / standby / system 模式的 CPU 核与 standby 进入条件。

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| McuIdleModeCpuCore | ENUM | IDLE_CORE0_SEL0 | (动态) IDLE_COREx_SELx | 负责触发 idle 模式的 CPU 核 |
| McuStandbyEntryMode | ENUM | STANDBY_ENTRY_REQ_SLEEP_SEL0 | REQ_SLEEP_SEL0 / ESR_SEL4 | 待机域进入条件：REQ_SLEEP=通过 PMSWCR1.STBYEV；ESR_SEL4=通过 ESR1/NMI（当 `McuStandbyEntryOnHwEventApi=true` 时不可选 ESR_SEL4） |
| McuSystemModeCpuCore | ENUM | SYSTEM_CORE0_SEL1 | (动态) SYSTEM_COREx_SELx | 负责触发系统模式（sleep/standby）的 CPU 核 |

---

## 5. 模块配置 McuModuleConfiguration

> **用途**：MCU 模块的核心配置容器，包含时钟树、DEM 事件引用与电源模式设置。

### 5.1 头部参数

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| McuClockSrcFailureNotification | ENUM | DISABLED | DISABLED / ENABLED | 时钟失效通知使能（HW 不支持时应禁用；保留用于 AUTOSAR 兼容，不可编辑） |
| McuNumberOfMcuModes | INTEGER | 1 | 1–255 | MCU 可用模式数（保留参数，不影响代码生成） |
| McuRamSectors | INTEGER | 0 | 0–4294967295 | MCU 可用 RAM 扇区数（保留参数，不影响代码生成） |
| McuResetSetting | INTEGER | 1 | 1–255 | 复位配置索引，作用于 `Mcu_PerformReset` 的硬件特性（保留用于兼容，不可编辑） |

### 5.2 时钟树配置（核心）

时钟树是 MCU 配置最复杂、最关键的部分。整体结构为 `McuClockSettingConfig` 列表 → `McuClockReferencePoint` + `McuClockingSystemConfig`。

> **配置思路**：先配振荡器频率（`McuOscillatorConfig`）→ 配各 PLL 频率与分频（`McuSystemPllSettingConfig` / `McuPeripheralPllSettingConfig` / `McuAudioPllSettingConfig`）→ 配 PLL 分发到各总线和外设的目标频率（`McuPllDistributionSettingConfig`）→ 设置时钟参考点（`McuClockReferencePoint`，供其他模块引用）→ 配置时钟监控与外部时钟输出。PLL 公式：`Fpll = Fosc × (NDivider+1) / ((KDivider+1) × (PDivider+1))`。

#### 5.2.1 McuClockSettingConfig（列表 1–256）

> 一个时钟设置实例代表一套完整时钟方案（例如"正常时钟"与"低功耗时钟"可各建一个）。`Mcu_InitClock(ClockSettingId)` 在运行时切换。

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| McuClockSettingId | INTEGER | 0（容器序号） | 0–255 | 时钟设置 ID；需在所有 ClockSettings 中唯一且从 0 连续，作为 `Mcu_InitClock` 的参数 |

#### 5.2.2 McuOscillatorConfig（振荡器配置）

> **用途**：振荡器设置，包括晶振频率、负载电容、振荡模式、HSPHY 时钟选择。

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| McuMainOscillatorFrequency | INTEGER | 25 | 20–50 | 外部主晶振频率（**单位 MHz**） |
| McuOscAmpCalibrationEnable | BOOLEAN | false | true/false | 振荡器幅度校准使能（仅 `McuOscillatorMode=EXT_CRYSTAL_CERAMIC_RES_MODE_SEL0` 时可置 true） |
| McuOscX1Capacitance0Enable ~ 3Enable | BOOLEAN | false | true/false | XTAL1 负载电容 CL0/CL1/CL2/CL3 使能 |
| McuOscX2Capacitance0Enable ~ 3Enable | BOOLEAN | false | true/false | XTAL2 负载电容 CL0/CL1/CL2/CL3 使能 |
| McuPHY0ClockSel | BOOLEAN | false | true/false | HSPHY0 PMA 备选参考时钟选择：true=fPCIEREF，false=fOSC（需 PHY0 可用且 HsphyInitEnable=true） |
| McuPHY2ClockSel | BOOLEAN | false | true/false | HSPHY2 PMA 备选参考时钟选择：true=fPCIEREF，false=fOSC（需 HsphyInitEnable=true） |
| McuOscillatorMode | ENUM | EXT_CRYSTAL_CERAMIC_RES_MODE_SEL0 | 见下方 | 振荡器模式（HsphyInitEnable=true 时不可选 DISABLED） |
| McuSysClkFrequency | INTEGER | 20 | 20（固定） | 系统 Sysclk 频率（**MHz**） |

**McuOscillatorMode 取值**：
- `EXT_CRYSTAL_CERAMIC_RES_MODE_SEL0`：外部晶振/陶瓷谐振器模式
- `EXT_INPUT_CLOCK_MODE_NOT_SHBY_SEL2`：外部输入时钟（整形器未旁路）
- `OSC_DISABLED_MODE_SEL3`：振荡器禁用
- `EXT_INPUT_CLOCK_MODE_SHBY_SEL7`：外部输入时钟（整形器旁路）

#### 5.2.3 McuClockReferencePoint（列表 1–256）

> **用途**：定义时钟树中的参考点频率，供其他 MCAL 模块（Port/Adc/Can/Spi...）作为输入频率引用。即使只用一个频率，也必须定义至少一个参考点。

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| McuClockReferencePointFrequency | FLOAT | 2.5E7 | 2.5E7–5.0E8 | 参考点频率（**单位 Hz**） |

#### 5.2.4 McuClockMonitorConfig（时钟监控）

> **用途**：时钟存活（alive）与时钟分频器（div）监控。**重要**：当 `McuSafetyErrorDetect=true` 时，所有监控均不应被禁用。

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| McuAudioPllClockAliveMonEnable | BOOLEAN | true | Audio PLL 时钟存活监控（仅 Clock.AudioMaxFrequency>0 可用） |
| McuAudioPllDivMonEnable | BOOLEAN | true | Audio PLL 分频器监控 |
| McuPpuPllClockAliveMonEnable | BOOLEAN | true | PPU PLL 时钟存活监控（仅 Clock.PPUMaxFrequency>0 可用） |
| McuPpuPllDivMonEnable | BOOLEAN | true | PPU PLL 分频器监控 |
| McuBackupClockAliveMonEnable | BOOLEAN | true | Backup 时钟存活监控 |
| McuEVROscMonEnable | BOOLEAN | true | EVR 振荡器（Backup）时钟范围监控 |
| McuPerClockDivMonEnable | BOOLEAN | true | 外设时钟分频器监控 |
| McuPll0ClockAliveMonEnable / McuPll0KDivMonEnable | BOOLEAN | true | PLL0 存活 / K 分频器监控 |
| McuPll1ClockAliveMonEnable / McuPll1KDivMonEnable | BOOLEAN | true | PLL1 存活 / K 分频器监控 |
| McuPll2ClockAliveMonEnable / McuPll2KDivMonEnable | BOOLEAN | true | PLL2 存活 / K 分频器监控 |
| McuPll3ClockAliveMonEnable / McuPll3KDivMonEnable | BOOLEAN | true | PLL3 存活 / K 分频器监控 |
| McuRampOscClockAliveMonEnable | BOOLEAN | true | Ramp 振荡器时钟存活监控 |
| McuSysClockDivMonEnable | BOOLEAN | true | 系统时钟分频器监控 |

#### 5.2.5 McuExternalClockOutputConfig（外部时钟输出）

> **用途**：配置 EXTCLK0/EXTCLK1 外部时钟输出。

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| McuExtClock0Enable | BOOLEAN | false | true/false | EXTCLK0 是否在外部 pad 上可用 |
| McuExtClock1Enable | BOOLEAN | false | true/false | EXTCLK1 是否在外部 pad 上可用 |
| McuExtClock1Inverted | BOOLEAN | false | true/false | EXTCLK1 输出是否反相 |
| McuExtClockOutSel0 | ENUM | (动态) | 动态列表 Clock.ExtClockOutSel0 | EXTCLK0 时钟源（仅 ExtClock0Enable=true 可配） |
| McuExtClockOutSel1 | ENUM | (动态) | 动态列表 Clock.ExtClockOutSel1 | EXTCLK1 时钟源（仅 ExtClock1Enable=true 可配） |
| McuFoutClockDiv | INTEGER | 1 | 1–128 | 由 fSPB 生成 fOUT 的分频重载值（fOUT = fSPB / DIV；编程值为输入值-1） |

#### 5.2.6 McuPeripheralPllSettingConfig（外设 PLL）

> **用途**：外设 PLL（PLL1/PLL2/PLL3）的频率与分频配置。公式：`Fpll = Fosc × (NDivider+1) / ((KxDivider+1) × (PDivider+1))`。当时钟分发源选 BACKUP 时，需手动将频率配置为 100 MHz。

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| McuPerPll1Frequency | FLOAT | Clock.PerPll1MaxFrequency | PerPll1Min..Max | PLL1 频率（Hz） |
| McuPerPll2Frequency | FLOAT | Clock.PerPll2MaxFrequency | PerPll2Min..Max | PLL2 频率（Hz） |
| McuPerPll3Frequency | FLOAT | Clock.PerPll3MaxFrequency | PerPll3Min..Max | PLL3 频率（Hz） |
| McuPerPllK2Divider | INTEGER | Clock.PerPllK2DividerValue | 0–15 | K2 分频值 |
| McuPerPllK2PreDiv | ENUM | Clock.PerPllK2PreDividerValue | 1.0/1.2/1.6/2.0 | K2 预分频 |
| McuPerPllK3Divider | INTEGER | Clock.PerPllK3DividerValue | 0–7 | K3 分频值 |
| McuPerPllK3PreDiv | ENUM | Clock.PerPllK3PreDividerValue | 1.0/1.1/1.2/1.4/1.6/1.7/2.0 | K3 预分频 |
| McuPerPllK4Divider | INTEGER | Clock.PerPllK4DividerValue | 0–7 | K4 分频值 |
| McuPerPllK4PreDiv | ENUM | Clock.PerPllK4PreDividerValue | 1.0/1.2/1.6/2.0 | K4 预分频 |
| McuPerPllNDivider | INTEGER | Clock.PerPllNDividerValue | 0–127 | N 分频值 |
| McuPerPllPDivider | INTEGER | Clock.PerPllPDividerValue | 0–7 | P 分频值 |
| McuPerPllClkSelSrc1 | ENUM | PER_PLL_CLOCK_SOURCE1_PLL1_SEL1 | DISABLED_SEL0/PLL1_SEL1/SPB_SEL2/SYSCLK_SEL3 | fsource1 时钟源选择 |
| McuPerPllFmEnable | BOOLEAN | false | true/false | 外设 PLL 频率调制（FM）使能 |
| McuPerPllFMModAmp | FLOAT | 1.25 | 0–2 | FM 调制幅度（%），对应 CLOCK_PERPLLCON2.MODCFG[9:0] |

#### 5.2.7 McuPllDistributionSettingConfig（PLL 分发与各模块目标频率）

> **用途**：配置 PLL 时钟分发到各总线/外设的目标频率。这是外设频率的"总调度"，参数众多。**单位均为 Hz**，置 0 通常表示禁用该外设时钟。

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| McuPerClockDistributionSrcSel | ENUM | PLL_INPUT_CLOCK_SRC_SELECT_SEL0 | PLL_INPUT / BACKUP_INPUT | 外设时钟分发源（须与系统分发源一致） |
| McuSysClockDistributionSrcSel | ENUM | PLL_INPUT_CLOCK_SRC_SELECT_SEL0 | PLL_INPUT / BACKUP_INPUT | 系统时钟分发源（须与外设分发源一致） |
| McuAudioClockDistributionSrcSel | ENUM | PLL_INPUT_CLOCK_SRC_SELECT_SEL0 | PLL_INPUT / BACKUP_INPUT | 音频时钟分发源（须与系统一致） |
| McuSRIFrequency | FLOAT | Clock.SRIMaxFrequency | SRIMin..Max | SRI 总线频率（须与 SysPll 成比例；同步 PPU 时 PPU=SRI） |
| McuSPBFrequency | FLOAT | Clock.SPBMaxFrequency | SPBMin..Max | SPB 频率（须与 SysPll 成比例） |
| McuTPBFrequency | FLOAT | Clock.TPBMaxFrequency | 0..TPBMax | TPB 频率（须 ≥ eGTM/2、≥ GTM、≥ SPB×2） |
| McuCPBFrequency | FLOAT | Clock.CPBMaxFrequency | 0..2.0E8 | Converter 外设总线频率（>0） |
| McuADCFrequency | FLOAT | Clock.ADCMaxFrequency | 0..ADCMax | ADC 频率（须 = McuClockReferencePointFrequency1 = McuPerPll1Frequency） |
| McuGTMFrequency | FLOAT | Clock.GTMMaxFrequency | 0..2.0E8 | GTM 频率（0 禁用） |
| McueGTMFrequency | FLOAT | Clock.eGTMMaxFrequency | 0..5.0E8 | eGTM 频率 |
| McuSTMFrequency | FLOAT | Clock.STMMaxFrequency | 6.67E6..STMMax | STM 频率（**不可禁用**，MCAL 内部延时计算依赖） |
| McuMCANClockSrcSel | ENUM | DISABLED_SEL0 | DISABLED/MCANI_SEL1/OSC_SEL2 | MCAN 时钟源 |
| McuMCANFrequency | FLOAT | Clock.MCANMaxFrequency | 0..1.6E8 | MCAN 频率 |
| McuMCANHFrequency | FLOAT | Clock.MCANHMaxFrequency | 0..2.5E8 | MCANH 频率（≥ MCAN） |
| McuCANXLClockSrcSel | ENUM | DISABLED_SEL0 | DISABLED/CANXLI_SEL1/OSC_SEL2 | CANXL 时钟源 |
| McuCANXLFrequency | FLOAT | Clock.CANXLMaxFrequency | 0..CANXLMax | CANXL 频率 |
| McuCANXLHFrequency | FLOAT | Clock.CANXLHMaxFrequency | 0..CANXLHMax | CANXLH 频率（≥ CANXL） |
| McuASCLINFastFrequency | FLOAT | Clock.ASCLINFastMaxFrequency | 0..ASCLINFastMax | ASCLIN fast 频率（0 禁用 fast） |
| McuASCLINSlowFrequency | FLOAT | Clock.ASCLINSlowMaxFrequency | 0..ASCLINSlowMax | ASCLIN slow 频率（0 禁用） |
| McuASCLINSlowClockSrcSel | ENUM | DISABLED_SEL0 | DISABLED/ASCLINSI_SEL1/OSC0_SEL2 | ASCLIN slow 时钟源 |
| McuMSCClockSrcSel | ENUM | DISABLED_SEL0 | DISABLED/SOURCE1_SEL1/SOURCE2_SEL2 | MSC 时钟源 |
| McuMSCFrequency | FLOAT | Clock.MSCMaxFrequency | 0..2.0E8 | MSC 频率 |
| McuQSPIClockSrcSel | ENUM | DISABLED_SEL0 | DISABLED/SOURCE1_SEL1/SOURCE2_SEL2 | QSPI 时钟源 |
| McuQSPIFrequency | FLOAT | Clock.QSPIMaxFrequency | 0..QSPIMax | QSPI 频率 |
| McuxSPIFrequency | FLOAT | 0.0 | 0..2.0E8 | xSPI 频率（须 = McuPerPll3Frequency） |
| McuxSPISlowFrequency | FLOAT | Clock.xSPISlowMaxFrequency | 0..2.0E8 | xSPI slow 频率 |
| McuI2CFrequency | FLOAT | Clock.I2CMaxFrequency | 0..I2CMax | I2C 频率 |
| McuErayFrequency | FLOAT | Clock.ErayMaxFrequency | 0..8.0E7 | FlexRay E-Ray 频率（= PerPll1/2；Backup 源时置 0） |
| McuFSIFrequency | FLOAT | Clock.FSIMaxFrequency | 2.5E7..FSIMax | FSI 频率（≤ SRI 且 ≥ SPB） |
| McuHsctFrequency | FLOAT | Clock.HsctMaxFrequency | 0..8.0E8 | HSCT 频率（= 外设 DCO/2） |
| McuGETHFrequency | FLOAT | Clock.GETHMaxFrequency | 0..2.5E8 | 千兆以太网频率（≤ SRI，0 禁用） |
| McuLETHFrequency | FLOAT | Clock.LETHMaxFrequency | 0..LETHMax | LETH 频率（非 0 时 ≥ 125 MHz） |
| McuLETH100Frequency | FLOAT | Clock.LETH100MaxFrequency | 0..LETH100Max | LETH100 频率（须 = PerPll2） |
| McuPPUFrequency | FLOAT | Clock.PPUMaxFrequency | 0..4.54E8 | PPU 频率（非 0 时 ≥ 25 MHz） |
| McuRCBFrequency | FLOAT | Clock.RCBMaxFrequency | 0..2.0E8 | RCB 频率 |
| McuSDMMCFrequency | FLOAT | Clock.SDMMCMaxFrequency | 0..1.0E8 | SDMMC 频率（须 = PerPll2/2） |
| McuAudioFrequency | FLOAT | Clock.AudioPllDefaultFrequency | 0..1.0E8 | 音频频率（非 0 时在 AudioMin..Max） |
| McuPFlashWaitCycle | INTEGER | Clock.PFlashWaitCycle | 1–32 | PFLASH 访问等待周期（基于 SRI 时钟） |
| McuDFlashWaitCycle | INTEGER | Clock.DFlashWaitCycle | 1–32 | DFLASH 访问等待周期（基于 FSI 时钟） |

> **提示**：上表中频率间的相互约束（如 ADC 须等于参考点频率、SDMMC 须等于 PerPll2/2）是硬件时钟分发路径的硬性要求，配置错误会被 EB tresos 校验拦截。

#### 5.2.8 McuSystemPllSettingConfig（系统 PLL）

> **用途**：系统主时钟 PLL 配置（含可选 PPU PLL）。`Fpll = Fosc × (NDivider+1) / ((K2Divider+1) × (PDivider+1))`。分发源为 BACKUP 时需手动配置为 100 MHz。

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| McuSysPllFrequency | FLOAT | Clock.SysPllMaxFrequency | SysPllMin..Max | 系统 PLL 频率（Hz） |
| McuSysPllNDivider | INTEGER | Clock.SysPllNDividerValue | 0–127 | N 分频值 |
| McuSysPllPDivider | INTEGER | Clock.SysPllPDividerValue | 0–7 | P 分频值 |
| McuSysPllK2Divider | INTEGER | Clock.SysPllK2DividerValue | 0–15 | K2 分频值 |
| McuSysPllK2PreDiv | ENUM | Clock.SysPllK2PreDividerValue | 1.0（仅此值） | K2 预分频 |
| McuSysPllK3Divider | INTEGER | Clock.SysPllK3DividerValue | 0–7 | K3 分频值（仅 PPUMaxFrequency>0 可编辑） |
| McuSysPllK3PreDiv | ENUM | Clock.SysPllK3PreDividerValue | 1.0/1.1/1.2/1.4/1.6/1.7/2.0 | K3 预分频（仅 PPUMaxFrequency>0 可编辑） |
| McuPpuPllFrequency | FLOAT | Clock.PPUMaxFrequency | 0..4.54E8 | K3 后的 PPU PLL 频率（Hz）（仅 PPUMaxFrequency>0 可编辑） |
| McuPllInputSrcSel | ENUM | OSC_CLOCK_SRC_SELECT_SEL1 | BACKUP_SEL0/OSC_SEL1/SYSCLK_SEL2/OSC_F_SEL3 | PLL 输入源：Backup / Oscillator / SYSCLK 引脚 / 带尖峰滤波 Oscillator |
| McuPllFmEnable | BOOLEAN | false | true/false | 系统 PLL 频率调制使能 |
| McuPllFMModAmp | FLOAT | 1.25 | 0.0–2.0 | FM 调制幅度（%），对应 CLOCK_SYSPLLCON2.MODCFG[9:0] |

#### 5.2.9 McuAudioPllSettingConfig（音频 PLL）

> **用途**：音频时钟 PLL 配置。`Faudio_pll = Fosc × (NDivider+1) / ((KDivider+1) × (PDivider+1))`。分发源为 BACKUP 时需配置为 100 MHz；频率填 0 表示禁用。仅当 `Clock.AudioMaxFrequency>0` 时可配置。

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| McuAudioPllFrequency | FLOAT | Clock.AudioPllDefaultFrequency | 0..1.0E8（非 0 时 60–100 MHz） | 音频 PLL 频率（Hz） |
| McuAudioPllNDivider | INTEGER | Clock.AudioPllNDivider | 0–127 | N 分频值 |
| McuAudioPllNFracDivider | FLOAT | Clock.AudioPllNFracDivider | 0–0.9999 | N 小数分频值 |
| McuAudioPllKDivider | INTEGER | Clock.AudioPllKDivider | 0–15 | K 分频值 |
| McuAudioPllPDivider | INTEGER | Clock.AudioPllPDivider | 0–7 | P 分频值 |

### 5.3 DEM 事件引用 McuDemEventParameterRefs

> **用途**：指向 Dem 模块 `DemEventParameter` 的引用集合。当对应错误发生时通过 `Dem_SetEventStatus` 上报，EventId 取自所引用 DemEventParameter 的符号值。容器为可选。

| 引用名 | 类型 | 指向目标 | 说明 |
|---|---|---|---|
| MCU_E_CLOCK_FAILURE | REF | DemEventParameter | 时钟失效事件（已禁用，仅 AUTOSAR 合规保留；OPTIONAL） |
| MCU_E_CLOCK_INIT_FAILURE | REF | DemEventParameter | 初始化阶段时钟失效事件（OPTIONAL） |
| MCU_E_REG_READBACK_FAILURE | REF | DemEventParameter | 运行时寄存器回读失败事件（OPTIONAL） |

### 5.4 MCU 模式设置 McuModeSettingConf（列表 1–3）

> **用途**：定义不同电源模式下 MCU 的工作参数。每个实例以 `McuMode` 编号标识，需在所有模式中唯一。AURIX3G 典型映射：**McuMode=0 → Normal/Reset 模式，McuMode=2 → Standby 待机模式**。待机相关子容器仅在 McuMode=2 实例下生效。

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| McuMode | INTEGER | 0（序号） | 0–2（需唯一） | MCU 模式编号；决定该实例的电源模式语义 |

#### 5.4.1 McuStandbyModeSettingConf（待机进入/供电配置，仅 McuMode=2）

> **用途**：配置待机模式的进入触发与电源/RAM 供电策略。

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| McuStdbyModeSel | ENUM | STDBY0_STATE_SEL0 | STDBY0_SEL0 / STDBY1_SEL1 | 选择待机状态：STDBY0 / STDBY1 |
| McuStdby0ToStdby1TransitionSCREnable | BOOLEAN | false | true/false | SCR 是否可触发 STDBY0→STDBY1 迁移 |
| McuStdbyRamSupplySel | ENUM | STDBY_RAM_NOT_SUPPLIED_SEL0 | (动态) Mcu.StandbyRamSupplySel | 待机模式下哪些待机 RAM 被供电 |
| McuStdbySCRRamSupplyEnable | BOOLEAN | false | true/false | 待机模式下 SCR RAM 是否供电 |
| McuStdbyVDDEXTDCPowerDownEnable | BOOLEAN | false | true/false | 待机时是否允许关断 VDDEXTDC 电源 |
| McuStdbyEntryOnESR2Trig | BOOLEAN | false | true/false | 是否在 ESR2 触发时进入待机（需 StandbyEntryOnHwEventApi=false） |
| McuStdbyEntryOnVDDEXTRampDown | BOOLEAN | false | true/false | VDDEXT 电源降压时是否进入待机 |
| McuStdbyEntryOnVDDRampDown | BOOLEAN | false | true/false | VDD 电源降压时是否进入待机 |
| McuStdbyEntryESR2DigitalFilterEnable | BOOLEAN | false | true/false | 待机进入触发(ESR2)数字滤波使能（70 kHz 待机时钟，≥50 us 低脉冲有效） |

#### 5.4.2 McuStandbyModeWakeupConf（待机唤醒源配置，仅 McuMode=2）

> **用途**：配置从待机模式唤醒的触发源、边沿检测、数字滤波与唤醒定时器（WUT）。唤醒源多支持"直接唤醒到 RUN"或"顺序唤醒（STDBY0→STDBY1→RUN）"两种语义。

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| McuStdbyBlankingFilterDelay | ENUM | BLNKNG_FLTR_DLY_156_uS_SEL1 | 0us/156us/1.25ms…2560ms 共 14 档 | 唤醒消隐滤波延迟 |
| McuStdbyESR0EdgeDetectionCtrl | ENUM | ESR0_NO_TRIG_SEL0 | 无/上升/下降/双沿 | ESR0 唤醒触发边沿（非 NO_TRIG 时启用） |
| McuStdbyESR0DigitalFilterEnable | BOOLEAN | false | true/false | ESR0 数字尖峰滤波使能 |
| McuStdbyESR1EdgeDetectionCtrl | ENUM | ESR1_NO_TRIG_SEL0 | 无/上升/下降/双沿 | ESR1 唤醒触发边沿 |
| McuStdbyESR1DigitalFilterEnable | BOOLEAN | false | true/false | ESR1 数字尖峰滤波使能 |
| McuStdbyESR2EdgeDetectionCtrl | ENUM | ESR2_NO_TRIG_SEL0 | 无/上升/下降/双沿 | ESR2 唤醒触发边沿 |
| McuStdbyESR2DigitalFilterEnable | BOOLEAN | false | true/false | ESR2 数字尖峰滤波使能 |
| McuStdbyESR2WakeupEnable | ENUM | ESR2_WAKEUP_DISABLED_SEL0 | DISABLED/TO_RUN/STDBY0_TO_STDBY1/SEQUENTIAL | ESR2 唤醒方式 |
| McuStdbyPORSTWakeupEnable | BOOLEAN | false | true/false | PORST 引脚唤醒使能 |
| McuStdbyPWRWakeupEnable | ENUM | PWR_WAKEUP_DISABLED_SEL0 | DISABLED/TO_RUN/STDBY0_TO_STDBY1 | VEXT 电源 ramp up 唤醒 |
| McuStdbyPinAEdgeDetectionCtrl | ENUM | PINA_NO_TRIG_SEL0 | 无/上升/下降/双沿 | PinA 唤醒边沿 |
| McuStdbyPinADigitalFilterEnable | BOOLEAN | false | true/false | PinA 数字滤波使能 |
| McuStdbyPinBEdgeDetectionCtrl | ENUM | PINB_NO_TRIG_SEL0 | 无/上升/下降/双沿 | PinB 唤醒边沿 |
| McuStdbyPinBDigitalFilterEnable | BOOLEAN | false | true/false | PinB 数字滤波使能 |
| McuStdbyPinBWakeupEnable | ENUM | PINB_WAKEUP_DISABLED_SEL0 | DISABLED/TO_RUN/STDBY0_TO_STDBY1/SEQUENTIAL | PinB 唤醒方式 |
| McuStdbyPinCEdgeDetectionCtrl | ENUM | PINC_NO_TRIG_SEL0 | 无/上升/下降/双沿 | PinC 唤醒边沿 |
| McuStdbyPinCDigitalFilterEnable | BOOLEAN | false | true/false | PinC 数字滤波使能 |
| McuStdbyPinCWakeupEnable | ENUM | PINC_WAKEUP_DISABLED_SEL0 | DISABLED/TO_RUN/STDBY0_TO_STDBY1/SEQUENTIAL | PinC 唤醒方式 |
| McuStdbyRTCWakeupEnable | ENUM | RTC_WAKEUP_DISABLED_SEL0 | DISABLED/TO_RUN/STDBY0_TO_STDBY1/SEQUENTIAL | RTC 唤醒方式 |
| McuStdbySCRWakeupEnable | BOOLEAN | false | true/false | 8 位待机控制器(SCR) 唤醒使能 |
| McuStdbyVDDEXTPinWakeupEnable | ENUM | VDDEXT_PIN_WAKEUP_DISABLED_SEL0 | DISABLED/TO_RUN | VDDEXT 电源 ramp up 唤醒 |
| McuStdbyVDDEVRSBPinWakeupEnable | ENUM | VDDEVRSB_PINS_WAKEUP_DISABLED_SEL0 | (动态) DISABLED/TO_RUN/TO_STANDBY1 | VDDEVRSB 引脚唤醒 |
| McuStdbyWUTMode | ENUM | WUT_AUTO_RELOAD_MODE_SEL0 | 自动重载/STDBY0 自动停止/STDBY1 自动停止 | 唤醒定时器(WUT) 模式 |
| McuStdbyWUTReloadVal | INTEGER | 0 | 0–16777215（24 位） | WUT 重载值（非 0 时 WUT 才使能） |
| McuStdbyWUTClockDiv | BOOLEAN | false | true/false | WUT 时钟分频：true=fSB(70kHz)，false=fSB/2^10 |
| McuStdbyWUTWakeupEnable | ENUM | WUT_WAKEUP_DISABLED_SEL0 | DISABLED/TO_RUN/STDBY0_TO_STDBY1/SEQUENTIAL | WUT 唤醒方式 |

> **唤醒方式枚举通用语义**：`*_DISABLED` 禁用；`*_TO_RUN_MODE` 任意待机直接唤醒到 RUN；`*_STDBY0_TO_STDBY1` 从 STDBY0 唤醒到 STDBY1；`*_SEQUENTIAL` 顺序唤醒（STDBY0→STDBY1→RUN，分别置 WAKEUP_STAT0 的 WKP/OVRUN 标志）。

---

## 6. RAM 扇区配置 McuRamSectorSettingConf（列表 0–256）

> **用途**：配置需初始化的 RAM 扇区（地址、大小、初始化值、写入粒度与 ECUC 分区映射）。详见 SWS MCU030。

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 说明 |
|---|---|---|---|---|
| McuRamSectionId | INTEGER | 0（序号） | 0–255（需唯一且从 0 连续） | RAM 扇区 ID |
| McuRamSectionBaseAddress | INTEGER | 1879048192 | 0–4294967295 | RAM 扇区基地址 |
| McuRamSectionSize | INTEGER | 8 | 0–4294967295（须为 WriteSize 整数倍） | RAM 扇区大小（字节） |
| McuRamSectionWriteSize | INTEGER | 8 | 1/2/4/8 | 单次可写入 RAM 的数据大小（字节） |
| McuRamDefaultValue | INTEGER | 0 | 0–255 | 待初始化数据的预设填充值 |
| McuRamPartitionRef | REF | — | 指向 EcucPartition | 该 RAM 扇区所属的 ECUC 分区 |

---

## 7. 复位配置 McuResetSettingConf

> **用途**：配置各类复位源（SW / ESR0–2 / SMU safe0&1 / STM0–5 / HSPHY / PIN）触发时执行何种复位（不复位 / 系统复位 / 应用复位 / 模块组 0–3 复位）。每个复位源通过枚举选择响应类型，统一约定后缀：`_NO_RESET`（不复位）、`_SYSTEM_RESET`（系统复位）、`_APPLICATION_RESET`（应用复位）、`_MOD_GROUPx_RESET`（模块组 x 复位）。

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| McuSWResetConf | ENUM | SW_NO_RESET_SEL0 | SW 复位请求响应：不复位 / 系统复位 / 应用复位 |
| McuESR0ResetConf | ENUM | ESR0_NO_RESET_SEL0 | ESR0 复位（无/系统/应用/模块组0–3） |
| McuESR1ResetConf | ENUM | ESR1_NO_RESET_SEL0 | ESR1 复位（无/系统/应用/模块组0–3） |
| McuESR2ResetConf | ENUM | ESR2_NO_RESET_SEL0 | ESR2 复位（无/系统/应用/模块组0–3） |
| McuSMUSafe0ARResetConf | ENUM | SMUSAFE0AR_NO_RESET_SEL0 | SMU safe0 reset0（无/应用/模块组0–3） |
| McuSMUSafe0SRResetConf | ENUM | SMUSAFE0SR_NO_RESET_SEL0 | SMU safe0 reset1（无/系统/模块组0–3） |
| McuSMUSafe1ARResetConf | ENUM | SMUSAFE1AR_NO_RESET_SEL0 | SMU safe1 reset0（无/应用/模块组0–3） |
| McuSMUSafe1SRResetConf | ENUM | SMUSAFE1SR_NO_RESET_SEL0 | SMU safe1 reset1（无/系统/模块组0–3） |
| McuSTM0ResetConf ~ McuSTM5ResetConf | ENUM | *_NO_RESET_SEL0 | STM0–STM5 复位（无/系统/应用/模块组0–3），仅当对应核可用（NoOfCoreAvailable>核号）时可编辑 |
| McuPHYRSTResetConf | BOOLEAN | false | HSPHY0/1 复位时机：true=关断序列 T9 复位；false=复位请求时立即复位（使能 trace 时强制 T9） |
| McuPINRSTResetConf | BOOLEAN | false | 引脚复位时机：true=关断序列末尾复位；false=复位请求时立即复位 |

---

## 8. 发布信息 McuPublishedInformation / CommonPublishedInformation

### 8.1 McuResetReasonConf（固定 10 项）

> **用途**：定义可从 `Mcu_GetResetReason` 获取的复位原因符号，供 EcuM 的 `EcuMResetReason` 引用。

| McuResetReason 符号 | 值 | 含义 |
|---|---|---|
| MCU_LVD_RESET | 1 | 低压检测复位 |
| MCU_CLDPORST_RESET | 2 | 冷上电 POR 复位 |
| MCU_PDCLDRST_RESET | 4 | 电源域冷复位 |
| MCU_POWER_ON_RESET | 8 | 上电复位 |
| MCU_SYSRST_RESET | 16 | 系统复位 |
| MCU_APPRST_RESET | 32 | 应用复位 |
| MCU_DBGRST_RESET | 64 | 调试器复位 |
| MCU_TRRST_RESET | 128 | 看门狗/触发复位 |
| MCU_RESET_MULTIPLE | 254 | 多重复位 |
| MCU_RESET_UNDEFINED | 255 | 未定义复位 |

### 8.2 CommonPublishedInformation

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| ArMajorVersion / ArMinorVersion / ArPatchVersion | INTEGER | 4 / 6 / 0 | AUTOSAR 规范版本号 |
| SwMajorVersion / SwMinorVersion / SwPatchVersion | INTEGER | 2 / 30 / 1 | 软件版本号 |
| ModuleId | INTEGER | 101 | Mcu 驱动模块 ID |
| VendorId | INTEGER | 17 | 供应商 ID（Infineon） |
| Release | STRING | (动态) Rel.Derivate | 实现所用的 TC4xx 派生型号 |

---

## 9. 关键配置流程与建议

### 9.1 时钟树配置流程（推荐顺序）

1. **确定器件与晶振**：在 EB tresos 选择具体派生器件（决定 Clock schema 的可用范围）；设置 `McuMainOscillatorFrequency`（典型 20/25 MHz）与 `McuOscillatorMode`。
2. **配置系统 PLL**（`McuSystemPllSettingConfig`）：设定目标 `McuSysPllFrequency`，驱动会据此反推 N/P/K 分频；选择 `McuPllInputSrcSel`（通常 Oscillator）。
3. **配置外设 PLL**（`McuPeripheralPllSettingConfig`）：按外设最高频率需求设定 PLL1/2/3 频率。
4. **配置音频 PLL**（`McuAudioPllSettingConfig`）：如需音频时钟再配；否则填 0 禁用。
5. **配置 PLL 分发**（`McuPllDistributionSettingConfig`）：设定 SRI/SPB/TPB/CPB 总线频率与各外设（GTM/ADC/CAN/SPI/Ethernet...）目标频率，注意约束（ADC=参考点、SDMMC=PerPll2/2 等）。
6. **设置时钟参考点**（`McuClockReferencePoint`）：将供其他模块使用的频率定义为参考点（如 ADC 频率）。
7. **时钟监控与外部输出**：保持 `McuClockMonitorConfig` 各项使能（安全要求）；按需使能 `McuExternalClockOutputConfig`。
8. **验证**：EB tresos 会校验频率约束、范围与唯一性；解决全部错误后方可生成代码。

### 9.2 低功耗/待机配置流程

1. 在 `McuGeneralConfiguration` 使能 `McuLowPowerModeApi` 与（如需硬件事件触发）`McuStandbyEntryOnHwEventApi`。
2. 在 `McuLowPowerModeConfig` 选定触发 idle/system 模式的核，以及 standby 进入方式（REQ_SLEEP 或 ESR1/NMI）。
3. 创建 `McuModeSettingConf` 实例（McuMode=2），在其中配置：
   - `McuStandbyModeSettingConf`：待机状态（STDBY0/STDBY1）、RAM 供电、电源域关断、进入触发源。
   - `McuStandbyModeWakeupConf`：唤醒源（ESR0–2/PinA–C/RTC/SCR/WUT/PORST/PWR）、边沿检测、数字滤波、消隐延迟、唤醒定时器。
4. 配置 `McuCommonClockConfig` 中 ADC/HSPHY 的睡眠行为。

### 9.3 复位配置要点

- 复位类型分**系统复位**（System Reset，影响范围大）与**应用复位**（Application Reset，范围小，常用）、**模块组复位**（Mod Group 0–3，针对特定模块组）。
- 调试/安全场景下，STM 复位（每核一个）需根据 `NoOfCoreAvailable` 配置。
- `McuPHYRSTResetConf` / `McuPINRSTResetConf` 控制 PHY 与引脚的复位时机，影响复位序列时序，需结合硬件设计确认。

---

## 10. 重要约束与注意事项

1. **频率单位**：`McuMainOscillatorFrequency` 与 `McuSysClkFrequency` 单位为 **MHz**；其余 PLL/分发/参考点频率单位为 **Hz**。配置时务必区分，避免差 10⁶ 量级。
2. **分发源一致性**：`McuSysClockDistributionSrcSel`、`McuPerClockDistributionSrcSel`、`McuAudioClockDistributionSrcSel` 三者必须一致（同为 PLL 输入或同为 Backup）；选 Backup 时相关 PLL 频率需手动设为 100 MHz。
3. **安全监控不可关闭**：`McuSafetyErrorDetect=true` 时，`McuClockMonitorConfig` 中所有监控项均不应禁用，否则可能违反功能安全要求。
4. **唯一性与连续性校验**：`McuClockSettingId`、`McuRamSectionId`、`McuMode` 须各自唯一且从 0 连续，否则生成报错。
5. **参数依赖/可编辑性**：大量参数有编辑前置条件（如 HSPHY 参数依赖 `McuHsphyInitEnable=true`；Standby 参数仅 McuMode=2 可配；STM 复位依赖核数）。EB tresos 会灰显不可编辑项。
6. **保留参数**：`McuClockSrcFailureNotification`、`McuNumberOfMcuModes`、`McuRamSectors`、`McuResetSetting`、`MCU_E_CLOCK_FAILURE` 等为 AUTOSAR 兼容保留，通常不可编辑或不影响代码生成。
7. **分区映射**：`McuEcucPartitionRef` 引用的分区必须在 Rma 模块 `RmaPartitionToExecutionUnitMapping` 中已配置；`McuCommonPartitionRef` 须是前者子集。
8. **代码裁剪**：默认关闭的 `*Api` 开关（PerformReset/VersionInfo/LowPowerMode 等）为减小代码体积而设；按需开启即可。

---

## 附录 A：容器—参数数量统计

| 容器 | 参数/引用数量 | 关键内容 |
|---|---|---|
| McuGeneralConfiguration | 25 | API 开关、错误检测、分区映射 |
| McuCommonClockConfig | 9 | ADC/CDSP/HSPHY 睡眠与 PHY 功能选择 |
| McuLowPowerModeConfig | 3 | 低功耗触发核与进入模式 |
| McuModuleConfiguration（头） | 4 | 模式数、复位设置（保留参数） |
| McuOscillatorConfig | 15 | 晶振频率、电容、模式、PHY 时钟 |
| McuClockReferencePoint | 1 | 参考点频率 |
| McuClockMonitorConfig | 18 | 各 PLL/Backup 时钟存活与分频监控 |
| McuExternalClockOutputConfig | 6 | EXTCLK0/1 输出 |
| McuPeripheralPllSettingConfig | 14 | 外设 PLL1/2/3 频率与分频 |
| McuPllDistributionSettingConfig | 39 | 各总线/外设目标频率（最大容器） |
| McuSystemPllSettingConfig | 11 | 系统 PLL + PPU PLL |
| McuAudioPllSettingConfig | 5 | 音频 PLL |
| McuDemEventParameterRefs | 3 | DEM 事件引用 |
| McuModeSettingConf | 1 | McuMode 模式编号 |
| McuStandbyModeSettingConf | 9 | 待机进入/供电配置 |
| McuStandbyModeWakeupConf | 26 | 唤醒源/边沿/滤波/定时器 |
| McuRamSectorSettingConf | 6 | RAM 扇区设置 |
| McuResetSettingConf | 16 | 各复位源响应类型 |
| McuResetReasonConf | 10 | 复位原因符号 |
| CommonPublishedInformation | 8 | 版本/供应商/模块 ID |

---

> **文档来源**：基于 `plugins/Mcu_Aurix3G/config/Mcu.xdm`（VERSION 20.0.0, 2025-09-04, Infineon AURIX3G）整理。参数默认值/范围中标注为 `(动态)` 或 `Clock.XxxMaxFrequency` 者，由所选派生器件在 EB tresos 中动态求值，请以实际配置界面值为准。
