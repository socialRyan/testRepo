# AURIX3G GPT 模块配置说明文档

| 项目 | 内容 |
|---|---|
| 模块名称 | Gpt（General Purpose Timer，通用定时器） |
| 配置数据模型 | `plugins/Gpt_Aurix3G/config/Gpt.xdm` |
| 模型版本 | 16.0.0（日期 2025-08-26） |
| 目标平台 | Infineon AURIX3G（TC4xx 系列） |
| 配置工具 | EB tresos Studio（DataModel 7.0） |
| AUTOSAR 规范 | R4.6.0（SWS: Specification of Gpt Driver） |
| 变体类型 | VariantPostBuild（PB，支持后构建变体） |

> 本文档基于 `Gpt.xdm` 数据模型整理，覆盖该模块全部配置容器与参数。GPT 是 AUTOSAR MCAL 中相对简单的定时器驱动，本文档对每个配置项给出含义、取值与依赖关系。

---

## 1. 概述

### 1.1 GPT 模块职责

GPT（General Purpose Timer，通用定时器）驱动提供软件可配置的定时器服务，主要功能：

- **周期/单次定时**：每个 GPT 通道可配置为连续模式（CONTINUOUS，周期触发）或单次模式（ONESHOT，到期一次）。
- **定时通知**：到达目标时间时通过回调函数通知上层（`GptNotification`）。
- **预定义定时器（Predef Timer）**：提供固定分辨率的预定义定时器（100µs/32bit、1µs/16/24/32bit），用于 OS 或其他模块的标准时基。
- **唤醒功能**：支持将定时器作为唤醒源，配合 EcuM 在低功耗模式下唤醒 MCU。
- **时间查询**：提供已逝时间（`Gpt_GetTimeElapsed`）与剩余时间（`Gpt_GetTimeRemaining`）查询。

### 1.2 在 MCAL 中的位置与依赖

GPT 是上层服务/应用的定时基础，但**自身不直接驱动硬件定时器**，而是通过引用 GTM 或 GPT12 的硬件通道实现：

- **硬件资源引用**：每个 GPT 通道通过 `GptTimerUsed` 引用 Rma 模块分配的 GTM（TOM/ATOM）或 GPT12（GPT1/GPT2 Block）硬件通道。
- **时钟依赖**：通过 `GptPeripheralBusClockRef` 引用 MCU 模块的时钟配置（`McuClockingSystemConfig`），并经 `GptClockReferencePoint` 引用 MCU 的 `McuClockReferencePoint` 获取输入频率。
- **唤醒依赖**：唤醒源引用 EcuM 的 `EcuMWakeupSource`。

因此 GPT 配置的前置条件是：MCU、GTM、Rma、EcuM（如需唤醒）已正确配置。

### 1.3 关键 API

由 `GptConfigurationOfOptApiServices` 中各开关控制可用性：

`Gpt_Init` / `Gpt_DeInit` / `Gpt_StartTimer` / `Gpt_StopTimer` / `Gpt_GetTimeElapsed` / `Gpt_GetTimeRemaining` / `Gpt_EnableNotification` / `Gpt_DisableNotification` / `Gpt_SetMode` / `Gpt_EnableWakeup` / `Gpt_DisableWakeup` / `Gpt_CheckWakeup` / `Gpt_GetVersionInfo` / `Gpt_InitCheck`。

---

## 2. 配置文件与变体说明

| 项 | 说明 |
|---|---|
| 配置类 | 所有参数为 `VariantPostBuild`（PB 变体），生成到 `Gpt_PBcfg.c`，支持运行时多变体。 |
| 参数来源 | `AUTOSAR_ECUC`（标准 ECUC）或 `INFINEON`（厂商扩展），各参数已标注。 |
| 动态取值 | 部分参数默认值/范围由器件 schema 动态决定，例如 `GptChannelConfiguration` 列表上限、`GptTimerClockSelect` 取值随可用 GTM/GPT12 资源变化（XPath 求值）。 |
| 校验（INVALID） | 含大量交叉参数约束（XPath），如通道 ID 唯一连续、Predef 定时器与使能等级匹配、唤醒依赖等，错误时给出明确提示。 |

---

## 3. 配置容器总览（容器树）

```
Gpt                                              （模块定义）
│
├── GptChannelConfigSet                          通道配置集（支持后构建多变体）
│   └── GptChannelConfiguration [列表]           单个 GPT 通道配置
│       ├── GptChannelId                         通道 ID（唯一、连续）
│       ├── GptTimerChannelType                  通道类型（普通 / Predef 定时器）
│       ├── GptChannelMode                       通道模式（连续 / 单次）
│       ├── GptChannelTickFrequency              tick 频率（Hz，未使用）
│       ├── GptChannelTickValueMax               最大 tick 值（未使用）
│       ├── GptEnableWakeup                      使能唤醒
│       ├── GptNotification                      定时到期的通知回调
│       ├── GptTimerClockSelect                  定时器时钟选择（CMU_CLK / GPT12 预分频）
│       ├── GptChannelClkSrcRef                  时钟参考点引用（未使用）
│       ├── GptChannelEcucPartitionRef           通道到 ECUC 分区映射
│       ├── GptTimerUsed [列表1–2]               使用的硬件定时器通道（TOM/ATOM/GPT12）
│       └── GptWakeupConfiguration               唤醒配置（可选）
│           └── GptWakeupSourceRef               唤醒源引用（→ EcuM）
│
├── GptConfigurationOfOptApiServices             可选 API 服务开关
│   ├── GptDeinitApi                            Gpt_DeInit()
│   ├── GptEnableDisableNotificationApi          Gpt_Enable/DisableNotification()
│   ├── GptTimeElapsedApi                        Gpt_GetTimeElapsed()
│   ├── GptTimeRemainingApi                      Gpt_GetTimeRemaining()
│   ├── GptVersionInfoApi                        Gpt_GetVersionInfo()
│   ├── GptWakeupFunctionalityApi                Gpt_SetMode/EnableWakeup/DisableWakeup/CheckWakeup()
│   └── GptInitCheckApi                          Gpt_InitCheck()
│
├── GptDriverConfiguration                       驱动级配置
│   ├── GptDevErrorDetect                        开发错误检测
│   ├── GptRuntimeErrorDetect                    运行时错误检测
│   ├── GptSafetyErrorDetect                     安全错误检测
│   ├── GptPredefTimer100us32bitEnable           使能 100µs/32bit 预定义定时器
│   ├── GptPredefTimer1usEnablingGrade           1µs 预定义定时器使能等级
│   ├── GptReportWakeupSource                    唤醒源上报
│   ├── GptPeripheralBusClockRef                 外设总线时钟引用（→ MCU）
│   ├── GptEcucPartitionRef [列表0–12]           驱动到 ECUC 分区映射
│   ├── GptKernelEcucPartitionRef                内核分区映射（未使用）
│   └── GptClockReferencePoint [列表1–256]       时钟参考点
│       └── GptClockReference                    引用 MCU 的 McuClockReferencePoint
│
└── CommonPublishedInformation                   发布信息（版本/供应商）
```

> 标 `[列表 N–M]` 为多重实例容器。

---

## 4. 通道配置集 GptChannelConfigSet

> **用途**：GPT 通道配置的集合容器，支持后构建多变体（不同配置集可用于不同的 post-build 变体）。其下挂 `GptChannelConfiguration` 列表。

> **列表上限**：`GptChannelConfiguration` 的最大数量由可用硬件资源动态决定（GTM 的 ATOM/TOM 模块数与 GPT12 模块数），最小 1。

---

## 5. GPT 通道配置 GptChannelConfiguration（列表）

> **用途**：单个 GPT 通道的完整配置。

### 5.1 参数清单

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 来源 | 说明 |
|---|---|---|---|---|---|
| GptChannelId | INTEGER | node:pos（位置序号） | 0–4294967295（需唯一且连续） | AUTOSAR_ECUC | 通道 ID，赋给 `GptChannelConfiguration` 短名派生的符号名 |
| GptTimerChannelType | ENUM | GPT_TIMER_CHANNEL_NORMAL | 见 5.2 | INFINEON | 通道类型：普通定时器或预定义定时器 |
| GptChannelMode | ENUM | GPT_CH_MODE_CONTINUOUS | GPT_CH_MODE_CONTINUOUS / GPT_CH_MODE_ONESHOT | AUTOSAR_ECUC | 到达目标时间后的行为（连续 / 单次） |
| GptChannelTickFrequency | FLOAT | 0.0 | ≥0.0 | AUTOSAR_ECUC | tick 频率（Hz）。**参数未使用**，配置以消除告警 |
| GptChannelTickValueMax | INTEGER | 65535 | 0–18446744073709551615 | AUTOSAR_ECUC | 最大 tick 值（超过则回绕）。**参数未使用**，配置以消除告警 |
| GptEnableWakeup | BOOLEAN | false | true/false | AUTOSAR_ECUC | 使能该通道的 MCU 唤醒能力（须 `GptReportWakeupSource=true`） |
| GptNotification | FUNCTION-NAME | NULL | 函数名（可选） | AUTOSAR_ECUC | 定时到期的回调函数（非唤醒通知）。不使用时应移除容器 |
| GptTimerClockSelect | ENUM | 动态（CMU_CLK_0 / GPT12_PRESCALE_2） | 见 5.3 | INFINEON | 选择该通道使用的 GTM 定时器时钟 |
| GptChannelClkSrcRef | REF | — | GptClockReferencePoint | AUTOSAR_ECUC | 引用本驱动的时钟参考点。**参数未使用**，配置以消除告警 |
| GptChannelEcucPartitionRef | REF（可选） | — | EcucPartition | AUTOSAR_ECUC | 将通道映射到一个 ECUC 分区（须为 `GptEcucPartitionRef` 子集） |

### 5.2 GptTimerChannelType 取值（通道类型，核心）

| 枚举值 | 含义 | 说明 |
|---|---|---|
| GPT_TIMER_CHANNEL_NORMAL | 普通定时器 | 使用 GTM TOM/ATOM 或 GPT12 通道，可配置周期与模式 |
| GPT_PREDEF_TIMERCH_100US_32BIT | 预定义 100µs/32bit | 须 `GptPredefTimer100us32bitEnable=true` |
| GPT_PREDEF_TIMERCH_1US_16BIT | 预定义 1µs/16bit | 须 `GptPredefTimer1usEnablingGrade` 对应使能 |
| GPT_PREDEF_TIMERCH_1US_16_24BIT | 预定义 1µs/16+24bit | 同上，须匹配使能等级 |
| GPT_PREDEF_TIMERCH_1US_16_24_32BIT | 预定义 1µs/16+24+32bit | 同上，须匹配使能等级 |

> **约束**：
> - 同一等级的 Predef 定时器只能选择一个。
> - 1µs Predef 定时器须与 `GptPredefTimer1usEnablingGrade` 的位宽一致，否则报错。
> - Predef 定时器模式下，`GptChannelMode` 必须为 `GPT_CH_MODE_CONTINUOUS`，且不能使能 `GptEnableWakeup` / `GptNotification`。

### 5.3 GptTimerClockSelect 取值

取值随所选硬件资源（GTM TOM/ATOM 或 GPT12）动态提供：

- **GTM ATOM 通道**：`CMU_CLK_0` ~ `CMU_CLK_7`（来自 `Egtm.AtomClkSrc` / `Gtm.AtomClkSrc`）。
- **GTM TOM 通道**：`CMU_FXCLK_0` ~ `CMU_FXCLK_...`（来自 `Egtm.TomClkSrc` / `Gtm.TomClkSrc`）。
- **GPT12 通道**：`GPT12_PRESCALE_2/4/8/16/32/64/128/256/512/1024/2048/4096`（预分频）。

> 默认值：无 GTM 时为 `GPT12_PRESCALE_2`，有 GTM 时为 `CMU_CLK_0`。

### 5.4 GptTimerUsed（硬件定时器引用，列表 1–2）

> **用途**：列出该 GPT 通道使用的硬件定时器通道（GTM TOM/ATOM 或 GPT12 Block）。引用 Rma 模块分配的硬件资源。

| 项 | 说明 |
|---|---|
| 类型 | CHOICE-REFERENCE（可选引用） |
| 多重性 | 1–2（普通定时器=1；位宽≥24 的 Predef 定时器=2） |
| 可选目标 | Rma 模块的：`RmaGtmTomChannelAllocation`、`RmaGtmAtomChannelAllocation`、`RmaEgtmTomChannelAllocation`、`RmaEgtmAtomChannelAllocation`、`RmaGpt1BlockAllocation`、`RmaGpt2BlockAllocation` |

> **约束**：
> - 引用的硬件通道须在 Rma 中标记为"由 GPT 使用"（如 `TOM_CHANNEL_USED_BY_GPT`、`ATOM_CHANNEL_USED_BY_GPT`、`GPT1_BLOCK_USED_BY_GPT`、`GPT2_BLOCK_USED_BY_GPT`）。
> - 同一硬件通道在所有 GPT 配置中须唯一（不能被多个 GPT 通道共用）。
> - Predef 定时器不能选用 GPT12 Block（仅可用 GTM 通道）。

### 5.5 GptWakeupConfiguration（唤醒配置，可选）

> **用途**：配置该通道的唤醒回调与唤醒源。仅当 `GptReportWakeupSource=true` 时可用。

| 参数名 | 类型 | 默认值 | 指向目标 | 说明 |
|---|---|---|---|---|
| GptWakeupSourceRef | REF | — | EcuM/EcuMWakeupSource | 唤醒源引用；唤醒能力启用时此值传给 EcuM |

> **约束**：须 `GptReportWakeupSource=true` 且通道 `GptEnableWakeup=true` 时才生效。

---

## 6. 可选 API 服务 GptConfigurationOfOptApiServices

> **用途**：GPT 驱动可选 API 的开关集合，控制生成代码的功能裁剪与体积。

| 参数名 | 类型 | 默认值 | 控制的 API | 说明 |
|---|---|---|---|---|
| GptDeinitApi | BOOLEAN | false | `Gpt_DeInit()` | 去初始化 API |
| GptEnableDisableNotificationApi | BOOLEAN | false | `Gpt_EnableNotification()` / `Gpt_DisableNotification()` | 通知使能/禁能 API。**关闭时 `GptNotification` 不可配置** |
| GptTimeElapsedApi | BOOLEAN | false | `Gpt_GetTimeElapsed()` | 已逝时间查询 |
| GptTimeRemainingApi | BOOLEAN | false | `Gpt_GetTimeRemaining()` | 剩余时间查询 |
| GptVersionInfoApi | BOOLEAN | false | `Gpt_GetVersionInfo()` | 版本信息查询 |
| GptWakeupFunctionalityApi | BOOLEAN | false | `Gpt_SetMode()` / `Gpt_EnableWakeup()` / `Gpt_DisableWakeup()` / `Gpt_CheckWakeup()` | 唤醒功能 API 集合 |
| GptInitCheckApi | BOOLEAN | true | `Gpt_InitCheck()` | 初始化检查 API（INFINEON 扩展，默认开启） |

> **依赖**：`GptNotification`（§5）依赖 `GptEnableDisableNotificationApi=true`；`GptEnableWakeup` / `GptWakeupConfiguration` 依赖 `GptWakeupFunctionalityApi` 与 `GptReportWakeupSource`。

---

## 7. 驱动配置 GptDriverConfiguration

> **用途**：GPT 驱动的模块级配置（错误检测、预定义定时器、时钟与分区映射）。

### 7.1 参数清单

| 参数名 | 类型 | 默认值 | 取值 / 范围 | 来源 | 说明 |
|---|---|---|---|---|---|
| GptDevErrorDetect | BOOLEAN | false | true/false | AUTOSAR_ECUC | 开发错误检测（Det）开关 |
| GptRuntimeErrorDetect | BOOLEAN | true | true/false | INFINEON | 运行时错误检测开关 |
| GptSafetyErrorDetect | BOOLEAN | true | true/false | INFINEON | 安全错误检测开关 |
| GptPredefTimer100us32bitEnable | BOOLEAN | false | true/false | AUTOSAR_ECUC | 使能 100µs/32bit 预定义定时器 |
| GptPredefTimer1usEnablingGrade | ENUM | GPT_PREDEF_TIMER_1US_DISABLED | 见 7.2 | AUTOSAR_ECUC | 1µs 预定义定时器使能等级 |
| GptReportWakeupSource | BOOLEAN | false | true/false | AUTOSAR_ECUC | 使能唤醒源上报（唤醒功能总开关） |
| GptPeripheralBusClockRef | REF | — | Mcu/McuClockingSystemConfig | INFINEON | 引用 MCU 的外设总线时钟配置 |
| GptEcucPartitionRef | REF（列表 0–12） | — | EcucPartition | AUTOSAR_ECUC | 驱动到 ECUC 分区映射（须已在 Rma `RmaPartitionToExecutionUnitMapping` 配置） |
| GptKernelEcucPartitionRef | REF（可选） | — | EcucPartition | AUTOSAR_ECUC | 内核分区映射（将驱动内核分配到某核）。**参数未使用**，配置以消除告警 |

> **约束（重要）**：`GptRuntimeErrorDetect` 与 `GptSafetyErrorDetect` **不允许同时关闭**（同设 false 会报错）。

### 7.2 GptPredefTimer1usEnablingGrade 取值

| 枚举值 | 含义 |
|---|---|
| GPT_PREDEF_TIMER_1US_DISABLED | 禁用所有 1µs 预定义定时器 |
| GPT_PREDEF_TIMER_1US_16BIT_ENABLED | 使能 1µs/16bit |
| GPT_PREDEF_TIMER_1US_16_24BIT_ENABLED | 使能 1µs/16bit + 24bit |
| GPT_PREDEF_TIMER_1US_16_24_32BIT_ENABLED | 使能 1µs/16bit + 24bit + 32bit |

> 使能等级决定了通道 `GptTimerChannelType` 可选的 Predef 定时器位宽，二者必须匹配。

### 7.3 GptClockReferencePoint（时钟参考点，列表 1–256）

> **用途**：定义 GPT 的时钟参考点，引用 MCU 的 `McuClockReferencePoint` 以获取输入时钟频率。

| 参数名 | 类型 | 默认值 | 指向目标 | 说明 |
|---|---|---|---|---|
| GptClockReference | REF | — | Mcu/McuClockReferencePoint | 引用 MCU 时钟参考点。**参数未使用**，配置以消除告警 |

> 注：AURIX3G GPT 实际经 `GptPeripheralBusClockRef` 与 `GptTimerClockSelect` 获取时钟，`GptClockReferencePoint` / `GptChannelClkSrcRef` / `GptKernelEcucPartitionRef` / `GptChannelTickFrequency` / `GptChannelTickValueMax` 在本实现中标注为"未使用（unused）"，但配置工具会要求填值以消除告警——这是 AUTOSAR 标准 ECUC 与 Infineon 实现的差异点。

---

## 8. 发布信息 CommonPublishedInformation

> **用途**：包含供应商与版本信息（与 MCU/GTM 模块结构一致）。

| 参数名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| ArMajorVersion / ArMinorVersion / ArPatchVersion | INTEGER_LABEL | 4 / 6 / 0 | AUTOSAR 规范版本号 |
| SwMajorVersion / SwMinorVersion / SwPatchVersion | INTEGER_LABEL | （见源文件） | 软件版本号 |
| VendorId | INTEGER_LABEL | （见源文件） | 供应商 ID（Infineon） |
| Release | STRING_LABEL | （动态，派生型号） | 实现所用的 TC4xx 派生型号 |

---

## 9. 关键配置流程与建议

### 9.1 GPT 通道配置流程（推荐顺序）

1. **前置配置**：在 MCU 模块配置好时钟（`McuClockingSystemConfig` 与 `McuClockReferencePoint`），在 Rma 模块为 GPT 预留 GTM TOM/ATOM 或 GPT12 硬件通道（标记为 `*_USED_BY_GPT`）。
2. **驱动级配置**（`GptDriverConfiguration`）：
   - 设置错误检测开关（`GptDevErrorDetect` / `GptRuntimeErrorDetect` / `GptSafetyErrorDetect`，注意后两者不可同时关闭）。
   - 按需使能预定义定时器（`GptPredefTimer100us32bitEnable` / `GptPredefTimer1usEnablingGrade`）。
   - 如需唤醒，使能 `GptReportWakeupSource`。
   - 配置时钟引用（`GptPeripheralBusClockRef` → MCU）。
   - 配置 ECUC 分区映射（`GptEcucPartitionRef`）。
3. **可选 API**（`GptConfigurationOfOptApiServices`）：按需开启各 API 开关。
4. **通道配置**（`GptChannelConfiguration`，每通道）：
   - 设 `GptChannelId`（唯一连续）。
   - 选 `GptTimerChannelType`（普通或 Predef）。
   - 选 `GptChannelMode`（连续/单次；Predef 须连续）。
   - 配置 `GptTimerUsed`（引用 Rma 预留的硬件通道）。
   - 选 `GptTimerClockSelect`（CMU_CLK 或 GPT12 预分频）。
   - 按需配置 `GptNotification`（须 `GptEnableDisableNotificationApi=true`）。
   - 如需唤醒，配 `GptEnableWakeup` + `GptWakeupConfiguration`（须 `GptReportWakeupSource=true`）。
5. **验证**：解决 EB tresos 报出的所有校验错误后生成代码。

### 9.2 普通定时器 vs 预定义定时器

- **普通定时器**（`GPT_TIMER_CHANNEL_NORMAL`）：用户自定义周期，使用 1 个硬件通道（GTM TOM/ATOM 或 GPT12），可配置唤醒与通知。适合应用层周期任务。
- **预定义定时器**（`GPT_PREDEF_TIMERCH_*`）：固定分辨率（100µs 或 1µs），位宽 16/24/32bit。通常用于 OS 调度时基或标准计数。**不可配置唤醒/通知/单次模式**；位宽≥24 时需占用 2 个硬件通道。需先在驱动级使能对应等级。

### 9.3 唤醒配置要点

- 唤醒功能是分层开关：`GptWakeupFunctionalityApi`（API 开关）+ `GptReportWakeupSource`（驱动总开关）+ 通道级 `GptEnableWakeup` + `GptWakeupConfiguration`。
- 唤醒源 `GptWakeupSourceRef` 须指向 EcuM 已配置的 `EcuMWakeupSource`。
- Predef 定时器不支持唤醒。

---

## 10. 重要约束与注意事项

1. **通道 ID 唯一连续**：`GptChannelId` 须在所有通道配置中唯一且从 0 连续，否则报错。
2. **硬件资源独占**：`GptTimerUsed` 引用的硬件通道须在 Rma 中标记为 GPT 使用，且在所有 GPT 通道中唯一（不可共用）。
3. **Predef 定时器约束**：模式必须连续、不可唤醒/通知、不可用 GPT12 Block；位宽须与 `GptPredefTimer1usEnablingGrade` 匹配；位宽≥24 需 2 个硬件通道。
4. **错误检测不可全关**：`GptRuntimeErrorDetect` 与 `GptSafetyErrorDetect` 不能同时为 false（安全要求）。
5. **通知 API 依赖**：`GptNotification` 须 `GptEnableDisableNotificationApi=true`；不使用通知时应移除容器，而非填 NULL 或空值。
6. **唤醒依赖链**：`GptEnableWakeup` / `GptWakeupConfiguration` 须 `GptReportWakeupSource=true`。
7. **"未使用"参数**：`GptChannelTickFrequency`、`GptChannelTickValueMax`、`GptChannelClkSrcRef`、`GptKernelEcucPartitionRef`、`GptClockReference` 在本 Infineon 实现中标注为未使用（AUTOSAR 标准 ECUC 字段，实际时钟经 `GptPeripheralBusClockRef`/`GptTimerClockSelect` 处理），但需配置有效值以消除告警。
8. **动态资源上限**：`GptChannelConfiguration` 列表上限、`GptTimerClockSelect` 取值、`GptTimerUsed` 可选目标均随器件与已配置的 GTM/GPT12 资源动态变化，以配置界面实际值为准。

---

## 附录 A：容器—参数数量统计

| 容器 | 参数/引用数量 | 关键内容 |
|---|---|---|
| GptChannelConfiguration | 10 + GptTimerUsed + Wakeup | 通道 ID、类型、模式、时钟、硬件引用、唤醒 |
| GptConfigurationOfOptApiServices | 7 | 各可选 API 开关 |
| GptDriverConfiguration | 9 + GptClockReferencePoint | 错误检测、预定义定时器、时钟引用、分区 |
| CommonPublishedInformation | 8 | 版本/供应商信息 |

---

> **文档来源**：基于 `plugins/Gpt_Aurix3G/config/Gpt.xdm`（VERSION 16.0.0, 2025-08-26, Infineon AURIX3G）整理，覆盖全部配置容器与参数。标注"动态"或 XPath 表达式者由所选派生器件与已配置的 GTM/GPT12 资源在 EB tresos 中求值，请以实际配置界面值为准。
