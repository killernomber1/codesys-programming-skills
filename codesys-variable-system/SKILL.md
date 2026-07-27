---
name: codesys-variable-system
description: |
  何时调用：用户在 CODESYS 中需要声明变量、选择数据类型、使用匈牙利命名法、配置保持型变量（RETAIN/PERSISTENT）、定义常量或自定义类型（STRUCT/ENUM/ARRAY）时。
  何时不调用：用户询问 POU 类型选择（FUN/FB/PRG）、ST 语法或语言选型问题。
  关键 trigger 信号：用户问"变量怎么声明"、"匈牙利命名前缀"、"数据类型有哪些"、"保持型变量"、"RETAIN 和 PERSISTENT 区别"、"STRUCT 怎么定义"、"变量初始化"。
source_book: 《CODESYS 基础编程及应用指南》 陆国君
source_chapter: 第 4 章 变量（完整章节）；第 2 章 2.5.2 持续变量；第 6 章 6.4 数据转换指令
tags: [codesys, iec-61131-3, variable, data-type, naming, hungarian-notation, retain, persistent, struct, enum]
---

# 变量声明体系（作用域 + 类型 + 属性 + 命名）

## R — 原文（Reading）

> 给应用程序和库中的变量命名时应当尽可能地遵循匈牙利命名法。每一个变量的基本名字中应该包含一个有意义的简短描述。基本名字中每一个单词的首字母应当大写，其它字母则为小写。依据变量的数据类型，在基本名字之前加上小写字母前缀。
>
> 变量根据他的应用范围可分为局部变量、输入变量、输出变量、输入-输出变量、全局变量、临时变量、静态变量和外部变量。
>
> CoDeSys 内包含的保持型变量有 RETAIN 和 PERSISTENT RETAIN 两种。RETAIN 变量在热复位后保持，PERSISTENT RETAIN 在冷复位/原始复位/重新下载后仍保持。
>
> 在程序中，有时需要对一些变量预先赋予初值，CoDeSys 允许在定义变量时对变量进行赋初值处理。
>
> — 陆国君

---

## I — 方法论骨架（Interpretation）

CODESYS 的变量声明体系可从四个维度理解和运用：

### A) 作用域体系
变量声明关键字控制变量的可见范围和生命周期：

| 关键字 | 作用域 | 说明 |
|--------|--------|------|
| `VAR` | 局部 | POU 内部私有变量，外部不可见 |
| `VAR_INPUT` | 输入接口 | 只读传入，调用者赋值，POU 内部使用 |
| `VAR_OUTPUT` | 输出接口 | 只写传出，POU 计算结果返回给调用者 |
| `VAR_IN_OUT` | 输入输出 | 引用传递，POU 内部可读写调用者的变量（传递的是指针） |
| `VAR_GLOBAL` | 全局 | 整个应用可见，所有 POU 都可读写 |
| `VAR_TEMP` | 临时 | 每次 POU 调用时分配，调用结束释放（不保持值） |
| `VAR_STAT` | 静态 | FB 内部跨调用保持（FB 实例的生命周期内） |
| `VAR_EXTERNAL` | 外部 | 引用其他 POU 中声明的全局变量 |

### B) 数据类型
- **标准类型**：BOOL（位）、BYTE（8位）、WORD（16位）、DWORD（32位）、INT（16位有符号整型）、DINT（32位有符号整型）、REAL（32位浮点）、LREAL（64位浮点）、TIME（时间）、STRING（字符串）
- **扩展类型**：UNION（联合体）、LTIME（长精度时间）、WSTRING（宽字符串）、REFERENCE（引用）、POINTER（指针）
- **自定义类型**：`TYPE...STRUCT...END_STRUCT`（结构体）、`TYPE...ARRAY[...] OF...`（数组）、`TYPE...(...)`（枚举）、子范围类型（如 `INT 0..100`）

### C) 匈牙利命名法前缀表
变量名前加小写类型前缀，后接首字母大写的含义描述：

| 前缀 | 数据类型 | 示例 |
|------|----------|------|
| `b` | BOOL | `bStartButton`, `bAlarmActive` |
| `by` | BYTE | `byStatusByte` |
| `w` | WORD | `wControlWord` |
| `dw` | DWORD | `dwCounter` |
| `i` | INT | `iTemperature` |
| `di` | DINT | `diProductionTotal` |
| `r` | REAL | `rPressureValue` |
| `lr` | LREAL | `lrPrecisePosition` |
| `s` | STRING | `sDeviceName` |
| `e` | ENUM | `eMotorMode` |
| `p` | POINTER | `pBufferAddress` |
| `a` | ARRAY | `aSensorValues` |
| `t` | TIME | `tDelayTime` |
| `st` | STRUCT | `stMotorParams` |
| `fb` | 功能块实例 | `fbTimer_1` |

嵌套声明时按顺序连接前缀（如 `abTempValues` 表示 BOOL 数组）。

### D) 保持属性
- **VAR RETAIN**：热复位后保持值不变；冷复位/重新下载后初始化。适用：计数值、运行模式等需在断电后恢复的数据。
- **VAR_GLOBAL PERSISTENT RETAIN**：热/冷/原始复位均保持，重新下载后仍保持。适用：生产累计量、设备参数、校准值等永久保留数据。（CoDeSys V3.3.0.1 起 PERSISTENT 和 PERSISTENT RETAIN 功能已相同。）
- **VAR CONSTANT**：常量，运行时不可修改。适用：关键参数、系数、枚举名称映射。

### E) 初始化策略
- 声明时使用 `:=` 显式赋初值（如 `iCounter : INT := 0;`）
- 用户初值仅在控制器启动后第一个任务周期写入
- 在线修改后需 PLC 复位才生效
- RETAIN 变量的初始化赋值仅在首次下载/冷复位后生效，热复位不影响已有值

---

## A1 — 书中的应用（Past Application）

### 案例 1: 恒压供水 — 全局持续变量保存累计量
- **问题**: 恒压供水系统需要记录水泵累计运行时间、总供水量等生产数据，要求断电后不能丢失。
- **方法论的使用**: 使用 `VAR_GLOBAL PERSISTENT RETAIN` 声明 `diPump1RunTime`（运行时间）、`diTotalWaterVolume`（累计水量）等变量。在 `GlobVar_SystemParams` 全局变量列表中管理所有持续变量。
- **结果**: 断电重启后累计数据完好保留，无需人工重新设定。

### 案例 2: 交通灯 — 枚举类型提高可读性
- **问题**: 交通灯控制程序中用一个 INT 变量表示状态（1=东西绿灯, 2=东西黄灯, 3=南北绿灯, 4=南北黄灯），代码中出现大量数字魔数，难以理解和维护。
- **方法论的使用**: 定义枚举类型 `E_TrafficPhase : (EW_GREEN, EW_YELLOW, NS_GREEN, NS_YELLOW);`，将 STATUS 变量类型改为 `E_TrafficPhase`。代码中用枚举名代替数字。
- **结果**: 代码可读性显著提升，编译器可检查赋值合法性，避免越界状态。

---

## A2 — 触发场景（Future Trigger）★

### 用户会在什么情境下需要这个 skill?

1. 在声明区写变量时，不确定该用 `VAR`、`VAR_INPUT` 还是 `VAR_OUTPUT`
2. 需要选择数据类型：计算精度要求（REAL vs LREAL）、整数范围（INT vs DINT）
3. 变量命名被团队要求遵循匈牙利命名法，需要查前缀表
4. 需要保留断电重启后的现场数据（计件器、累计运行时间、设备参数）
5. 定义常量参数（轴限位、PID 系数、配方参数）
6. 需要创建结构体或枚举类型来组织相关变量

### 语言信号
- "这个变量能不能断电保持"
- "RETAIN 和 PERSISTENT 有什么区别"
- "匈牙利命名法前缀怎么加"
- "BOOL 变量的前缀是什么"
- "我要定义一个结构体"
- "变量怎么给初值"
- "这个变量应该用全局还是局部"

---

## E — 可执行步骤（Execution）

1. **确定变量作用域**
   - 仅在本 POU 内部使用 -> `VAR`
   - 外部传入的参数 -> `VAR_INPUT`（只读） / `VAR_IN_OUT`（可读写引用）
   - POU 返回给调用者的结果 -> `VAR_OUTPUT`
   - 多个 POU 共享的数据 -> `VAR_GLOBAL`
   - 临时中间变量、每次调用不保持 -> `VAR_TEMP`
   - 完成标准：每个变量选择了正确的作用域关键字

2. **选择合适的数据类型**
   - 位/开关量 -> BOOL
   - 小范围整数（-32768~32767）-> INT；大范围 -> DINT
   - 浮点运算/模拟量 -> REAL（32位）或 LREAL（64位高精度）
   - 时间/延时 -> TIME（格式 `t#100ms`）
   - 字符/文本 -> STRING
   - 相关变量组 -> STRUCT
   - 有限选项 -> ENUM
   - 同类型有序集合 -> ARRAY
   - 完成标准：数据类型满足精度和范围要求，不浪费内存

3. **应用匈牙利命名法**
   - 变量名前加小写类型前缀 + 首字母大写的含义描述
   - 示例：`rPID_Kp : REAL := 1.2;`（PID 比例系数）
   - 功能块实例用 `fb` 前缀（`fbTimer_1 : TON;`）
   - 嵌套声明时按顺序连接前缀（数组的数组：`aaiMatrix : ARRAY[0..3,0..3] OF INT;`）
   - 完成标准：项目内所有变量遵循统一的前缀命名规范

4. **决定保持属性**
   - 断电后需恢复 -> `VAR RETAIN`（如当前工序号）
   - 冷复位/下载后也需保留 -> `VAR_GLOBAL PERSISTENT RETAIN`（如累计产量）
   - 常数/不可修改参数 -> `VAR CONSTANT`（如 `rPI: REAL := 3.14159;`）
   - 完成标准：保持属性选择正确，RETAIN 容量在硬件允许范围内

5. **显式赋初值**
   - 在声明行用 `:=` 运算符赋初值
   - 对于计数器/累计量初始化为 0
   - 对于关键参数明确给出初始值（避免依赖默认值）
   - 完成标准：所有变量在声明区都有明确的初始值

---

## B — 边界（Boundary）★

### 不要在以下情况使用此 skill
- 用户询问 POU 的类型选择（FUN/FB/PRG），应使用 **codesys-pou-design** skill
- 用户询问 ST 语法（IF/CASE/FOR 怎么写），应使用 **codesys-st-language** skill
- 用户需要的是具体通信协议的变量地址映射（如 Modbus 的 %IW 地址分配），属于对应通信 skill
- 用户询问任务配置中的变量采样或调试中的变量强制/写入操作，属于 **codesys-debug-diagnosis** skill
- 变量声明中的隐含检查函数（CheckBounds/CheckRange）配置，属于调试诊断范畴
