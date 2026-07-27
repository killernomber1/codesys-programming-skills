---
name: codesys-ethercat
description: |
  CoDeSys 中 EtherCAT 主从站通讯的完整配置指南。涵盖 EtherCAT 核心原理
  "Processing on the Fly"、XML 设备描述文件加载、分布式时钟（DC）同步、
  PDO 映射以及 CoE 邮箱通信。当用户需要配置高速运动控制网络、多轴同步、
  或者使用 EtherCAT 从站（伺服驱动器、远程 I/O 站）时调用。
  不适用于非实时以太网通信或 Modbus 替代场景。
source_book: 《CODESYS 基础编程及应用指南》 陆国君
source_chapter: 第9.3节 — EtherCAT 通讯
tags: [codesys, iec-61131-3, ethercat, realtime-ethernet, motion-control, dc-sync, coe, pdo-mapping, industrial-automation]
---

# EtherCAT 主从站通讯配置

## R — 原文（Reading）

> EtherCAT 基于以太网的实时工业现场总线协议，采用飞速数据处理技术，从站设备在报文经过时直接读取/插入数据。
>
> — 陆国君，《CODESYS 基础编程及应用指南》

---

## I — 方法论骨架（Interpretation）

EtherCAT 的核心创新在于 **"Processing on the Fly"（飞速数据处理）**：标准以太网帧中，每个从站设备在报文流经其物理端口时，以硬件逻辑在纳秒级时间内读取属于自己的数据或插入数据，随后将报文转发至下一个从站。整个过程仅引入微秒级的端口延迟，最后一个从站将报文返回给主站。因此，EtherCAT 无需交换机、无需 IP 地址分配，所有从站共享一个以太网帧，带宽利用率极高。

这种架构决定了 EtherCAT 的配置逻辑：**主站集中管理全部通信关系**，每个从站的行为通过 **XML 设备描述文件**（EtherCAT Slave Information, ESI）定义，内容包括可用的 PDO（过程数据对象）、同步管理器通道、分布式时钟能力等。CoDeSys 作为开发环境，将 XML 文件导入后自动解析为从站设备的 I/O 映射接口。

CoE（CANopen over EtherCAT）协议层复用 CANopen 的对象字典和 PDO/SDO 通信机制，使熟悉 CANopen 的工程师能平滑迁移到 EtherCAT。

---

## A1 — 书中的应用（Past Application）

### 案例 1: 6 轴工业机器人控制系统
- **问题**: 6 台伺服驱动器需实现位置环 1 ms 同步，传统脉冲方向控制方案布线复杂且无法保证同步精度。
- **方法论的使用**: 每台伺服通过 EtherCAT 线型拓扑串联（IN/OUT 端口），使用 DC 分布式时钟模式，主站作为参考时钟（Reference Clock），所有从站自动同步到主站时钟。
- **结果**: 6 轴同步抖动 < 1 us，编码器位置数据在 200 us 内完成刷新，支持电子齿轮和电子凸轮功能。

### 案例 2: 分布式远程 I/O 站
- **问题**: 生产线 200 m 范围内分布 20 个 I/O 子站，传统硬接线方案电缆成本 > 3 万。
- **方法论的使用**: 每个 I/O 子站配备 EtherCAT 从站接口，通过标准以太网线（CAT5e）线型连接。在 CoDeSys 中导入各子站的 ESI 文件，拖拽添加从站，自动完成 I/O 变量映射。
- **结果**: 电缆成本降至 5000 元以下，系统上电后 EtherCAT 主站自动枚举所有从站并分配站地址。

---

## A2 — 触发场景（Future Trigger）★

### 用户会在什么情境下需要这个 skill?

1. 项目需要伺服驱动器多轴同步控制，最小周期 < 1 ms，抖动 < 1 us
2. 在 CoDeSys 设备树中需要添加一个 EtherCAT 从站（驱动器或 I/O 模块）并进行 PDO 映射
3. 调试过程中从站无法进入 OP（Operational）状态，需要排查 DC 时钟或 PDO 配置

### 语言信号
- "EtherCAT 从站怎么加"
- "Processing on the Fly 是什么意思"
- "DC 时钟不同步"
- "EtherCAT 状态机卡在 PreOP 或 SafeOP"
- "PDO 映射不对"
- "XML 设备描述文件怎么加载"
- "CoE 和 CANopen 是什么关系"

---

## E — 可执行步骤（Execution）

1. **添加 EtherCAT 主站**
   - 在 CoDeSys 设备树中，右键项目根节点 → "添加设备" → 从设备库中选择 "EtherCAT Master" 或对应 PLC 硬件厂商提供的 EtherCAT 主站设备。
   - 配置主站参数：
     - `SyncUnit Cycle`: 主站通信周期（与任务配置关联，典型值 1-4 ms，运动控制推荐 < 1 ms）
     - `EtherCAT AutoRestart`: 启用后从站掉线可自动恢复
   - 完成标准: 设备树中出现 EtherCAT_Master 节点，主站参数可编辑。

2. **添加从站设备（加载 XML 设备描述文件）**
   - 方法一（自动枚举）: 在主站节点下 → "扫描设备"，CoDeSys 自动识别已连接的从站并导入对应的 ESI 文件。
   - 方法二（手动添加）: 右键主站 → "添加设备" → 如果设备列表中没有目标从站，点击 "安装设备描述文件" → 选择 .xml 或 .esi 文件 → 安装后即可在设备列表中找到该从站。
   - 关键从站参数：
     - `Station Alias`（可选站别名，用于固定站地址而不依赖物理顺序）
     - `Auto Increment Address`: 通常保持默认，在启动时基于物理位置自动分配
     - `Synchronization Setting`: 选择 DC 模式（Distributed Clock）或 FreeRun 模式
   - 完成标准: 从站出现在主站子节点中，状态显示可切换至 PreOP / SafeOP / OP。

3. **配置分布式时钟（DC）同步**
   - 在从站参数中，设置 `SyncUnit` 指向主站的同步单元（同周期）。
   - `DC Mode`: 启用 "Distributed Clock" → 从站自动跟随主站参考时钟。
   - `Shift Time`: 输出相对于输入采样的偏移时间（优化 I/O 响应，通常保持默认或参考驱动器手册）。
   - 完成标准: 运行后监视 `DC 状态变量`，确认从站时钟偏移量稳定在纳秒级（通常 < 100 ns 抖动）。

4. **PDO 映射配置**
   - PDO（Process Data Object）是过程数据传输通道，分为 RxPDO（主站→从站输出数据）和 TxPDO（从站→主站输入数据）。
   - 在从站 "PDO Mapping" 选项卡中，可手动调整每个 PDO 中包含的变量（对象）及其排列顺序。
   - 常见 PDO 内容：
     - 伺服驱动器: 控制字（16#6040）、目标位置（16#607A）、状态字（16#6041）、实际位置（16#6064）、速度（16#606C）
     - 远程 I/O: 各位号对应的数字量输入/输出通道
   - 可通过 "Startup Parameters" 选项卡设置上电时自动写入的 CoE 对象（如伺服驱动器的模式切换）。
   - 完成标准: PDO 映射编译通过，检查 `PDO Length` 不超过从站最大支持长度。运行后在变量映射表中可见 `%IW` / `%QW` 变量。

5. **EtherCAT 状态机管理**
   - EtherCAT 从站状态转换路径: `Init → PreOP → SafeOP → OP`
     - **Init**: 初始状态，仅可读写 EEPROM
     - **PreOP**: 邮箱通信（CoE SDO）可用，PDO 通信不可用
     - **SafeOP**: 输入 PDO 有效，输出 PDO 被阻断（输出保持在安全状态）
     - **OP**: 输入输出 PDO 均有效，正常运行
   - CoDeSys 自动管理状态转换。若从站卡在某一状态，检查错误寄存器（`AL Status Code`）：
     - 16#0000: 正常
     - 16#0011: 邮箱配置错误
     - 16#0012: PDO 映射无效
     - 16#0014: DC 寄存器配置错误
   - 完成标准: 所有从站稳定处于 OP 状态，LED 指示灯正常（通常 RUN 灯绿色常亮或闪烁）。

6. **验证与诊断**
   - 使用 CoDeSys 的 "EtherCAT Diagnosis" 插件或在线视图观察：
     - 所有从站状态是否为 OP
     - 各从站的 DC 时钟偏移量
     - 通信错误计数器（`Rx Error Counter` / `Tx Error Counter`）
     - 帧丢失率
   - 完成标准: 通信错误计数器不持续增加，帧丢失率 < 0.01%。

---

## B — 边界（Boundary）★

### 不要在以下情况使用此 skill
- **非实时以太网或简单数据采集（周期 > 10 ms 可接受）**: 应选用 Modbus TCP 或 EtherNet/IP，EtherCAT 的优势无法发挥且配置成本更高。
- **仅连接少量 HMI 或上位机**: EtherCAT 是控制器与现场设备级总线，不适合上位机通信。
- **设备不支持 EtherCAT**: 许多存量设备仅提供 Modbus 或 CANopen 接口，不应强行使用 EtherCAT 网关。
- **功能安全通信（Safety over EtherCAT / FSoE）**: 需额外配置 Safety 相关参数并遵循 IEC 61508 流程，不在本 skill 范围内。
- **无线网络环境**: EtherCAT 严格要求有线以太网物理层，不支持无线桥接。
