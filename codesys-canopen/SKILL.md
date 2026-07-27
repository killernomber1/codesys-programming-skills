---
name: codesys-canopen
description: |
  CoDeSys 中 CANopen 通讯协议 PDO/SDO 的完整配置指南。涵盖对象字典（OD）、
  EDS/DCF 设备描述文件加载、COB-ID 分配、PDO 传输类型选择和 SDO 上传/下载流程。
  当用户需要通过 CANopen 连接分布式 I/O、伺服驱动器、变频器、传感器/执行器网络时调用。
  不适用于高速运动控制中的多轴硬实时同步（此时应选 EtherCAT）或单设备简单 Modbus 通信。
source_book: 《CODESYS 基础编程及应用指南》 陆国君
source_chapter: 第9.1节 — CANopen 通讯
tags: [codesys, iec-61131-3, canopen, pdo, sdo, object-dictionary, eds, industrial-automation, fieldbus]
---

# CANopen PDO/SDO 通讯配置

## R — 原文（Reading）

> CANopen 由 CiA 组织在 CAL 基础上发展而来的基于 CAN 总线的应用层协议。
>
> — 陆国君，《CODESYS 基础编程及应用指南》

---

## I — 方法论骨架（Interpretation）

CANopen 是构建在 CAN 总线物理层之上的应用层协议，定义了标准化的通信机制和设备配置文件。其核心是 **对象字典（Object Dictionary, OD）**——一个从 16#0000 到 16#FFFF 的索引表，每个索引条目描述设备的一个参数、变量或功能，是 CANopen 设备配置和通信的唯一入口。

CANopen 定义了两种截然不同的数据交换通道：

- **PDO（Process Data Object，过程数据对象）**: 高优先级、生产者/消费者模式的实时广播通信。数据无协议开销（1-8 字节纯数据），最大利用 CAN 帧的有效载荷。PDO 通信不产生应答，因此传输效率极高，适合周期性或事件触发的实时数据交换，如驱动器的控制字/状态字、I/O 状态。**PDO 通信只允许在运行态（Operational）进行。**

- **SDO（Service Data Object，服务数据对象）**: 客户端/服务器模式的确认式通信，通过邮箱协议读写对象字典的任意条目。每次传输都需要应答，因此可靠性高但吞吐量低，适合参数配置，如设置驱动器的电流限值、加速度斜坡时间。SDO 上传（Upload）是从服务器设备读取数据，SDO 下载（Download）是向服务器设备写入数据。

在 CoDeSys 中配置 CANopen 的核心工作流：添加主站 → 通过 EDS/DCF 描述文件导入从站 → 配置 PDO 映射和传输参数 → 使能运行态（Operational）。

---

## A1 — 书中的应用（Past Application）

### 案例 1: 分布式 I/O 站数字量采集
- **问题**: 生产线 50 路数字量输入需通过 CAN 总线传输到 PLC，要求刷新周期 < 10 ms。
- **方法论的使用**: 选择 CANopen 分布式 I/O 从站，将 50 个数字量输入映射到 7 个 PDO（每 PDO 最大 8 字节 = 64 位）。PDO 传输类型设为"同步循环"（传输类型 1），由主站 SYNC 对象触发同步采集。
- **结果**: 刷新周期约 5 ms，总线负载 < 20%，通过 PDO 映射一个变量即可读到所有输入状态。

### 案例 2: 变频器参数初始化与运行控制
- **问题**: 调试阶段需通过 CANopen 配置变频器的电机参数（额定电流、额定转速），并在运行时实时控制启停和设定频率。
- **方法论的使用**: 使用 SDO 下载（C/S 模式）在 PreOP 状态下写入参数到对象字典（如 16#2000-16#20FF 区域），进入 OP 状态后通过 TxPDO 读取实际转速、通过 RxPDO 下发目标频率和控制字。
- **结果**: 参数配置一次性完成，运行期间 PDO 通信周期 10 ms，未出现丢帧或数据错乱。

---

## A2 — 触发场景（Future Trigger）★

### 用户会在什么情境下需要这个 skill?

1. 在 CoDeSys 项目中需要通过 CANopen 连接远程 I/O 模块、伺服驱动器或变频器
2. 需要配置 PDO 映射以优化过程数据传输效率，或选择 PDO 传输类型（同步/异步/事件触发）
3. 调试阶段需要通过 SDO 读写从站对象字典中的参数（如驱动器电流限值、加速度）
4. 从站无法进入 Operational 状态，需排查 COB-ID 冲突或 PDO 映射长度

### 语言信号
- "CANopen 对象字典怎么配置"
- "PDO 和 SDO 有什么区别"
- "CANopen 从站加不上"
- "EDS 文件怎么加载"
- "COB-ID 冲突怎么解决"
- "PDO 传输类型怎么选"
- "NMT 状态机"
- "CoDeSys 里用 SDO 读写参数"

---

## E — 可执行步骤（Execution）

1. **添加 CANopen 主站设备**
   - 在 CoDeSys 设备树中右键 → "添加设备" → 选择 "CANopen Master" 或对应硬件平台提供的 CANopen 主站设备。
   - 主站参数配置：
     - `Baudrate`: 与所有从站统一（常见 125k、250k、500k、1M 波特率；总线长度越长，波特率需越低）
     - `Sync Cycle`: SYNC 报文周期（决定同步 PDO 的刷新间隔，典型值 5-50 ms）
     - `Node ID`: 主站自身的节点地址（通常设为 1）
   - 完成标准: 设备树中出现 CANopen_Master 节点，参数可编辑。

2. **加载 EDS/DCF 设备描述文件并添加从站**
   - 右键主站节点 → "添加设备"。
   - 若目标设备不在列表中，点击 "安装设备描述文件" → 选择 .eds 或 .dcf 文件 → 安装成功后即可在设备列表中找到该设备型号。
     - **EDS**（Electronic Data Sheet）: 标准的设备描述文件，文本格式，定义了设备的对象字典内容、PDO 配置和设备特性。
     - **DCF**（Device Configuration File）: 在 EDS 基础上增加了具体配置值，可直接用于批量部署。
   - 添加从站后，配置其 `Node ID`（1-127，网络中必须唯一）。
   - 完成标准: 从站出现在主站子节点中，NMT 状态可观察。

3. **配置 COB-ID**
   - COB-ID（CAN Object Identifier）是 CAN 报文的标识符，决定了报文优先级和通信对象。
   - CANopen 标准预定义连接集（Predefined Connection Set）使用功能码 + 节点 ID 分配 COB-ID：
     - NMT (0)：16#000
     - SYNC：16#080
     - PDO1 Tx (1-4)：16#180 + NodeID ~ 16#300 + NodeID
     - PDO1 Rx (1-4)：16#200 + NodeID ~ 16#380 + NodeID
     - SDO Tx：16#580 + NodeID
     - SDO Rx：16#600 + NodeID
     - Heartbeat：16#700 + NodeID
   - 在从站参数中，可修改各 PDO 和 SDO 的 COB-ID。**务必确保网络中每个 COB-ID 唯一**，否则会导致通信冲突。
   - 完成标准: 所有从站的 COB-ID 无重复（CoDeSys 通常自动分配，手动修改后需仔细检查）。

4. **PDO 映射配置**
   - 每个从站可配置多个 PDO，每个 PDO 最多包含 8 字节数据（1-8 个对象，每个对象占 1-64 位）。
   - **TxPDO**（从站→主站，输入）：将从站对象字典中的变量映射到 PDO，如压力值、状态字。
   - **RxPDO**（主站→从站，输出）：将主站变量映射到 PDO，如控制字、目标速度。
   - 在从站 "PDO Mapping" 选项卡中操作：
     - 选择 PDO 通道（如 PDO1Tx）
     - 添加要映射的对象索引（如 16#6041 状态字，16#6064 实际位置）
     - 设置对象在 PDO 中的排列位置
   - 完成标准: PDO 映射总长度 ≤ 8 字节（64 位），编译无错误。

5. **配置 PDO 传输类型**
   - PDO 参数中 `Transmission Type` 决定通信时机：

   | 类型值 | 传输模式 | 说明 |
   |--------|----------|------|
   | 0 | 同步非循环 | 收到 SYNC 后更新，但由远程请求或事件触发实际发送 |
   | 1-240 | 同步循环 | 每 N 个 SYNC 触发一次（1 = 每个 SYNC，2 = 每 2 个 SYNC，以此类推） |
   | 241-251 | 保留 | — |
   | 252 | 同步触发 | 收到 SYNC 后更新（与类型 0 类似，由制造商特定方式触发） |
   | 253 | 异步事件 | 由事件触发（数据变化或定时器溢出） |
   | 254-255 | 异步制造商/设备 | 由制造商特定机制或设备应用事件触发，通常用于定时触发 |

   - **选型建议**: 周期性 I/O → 同步循环（类型 1）；偶发状态数据 → 异步事件（类型 253）；快速变化的规约数据 → 同步非循环（类型 0）。
   - 完成标准: 传输类型匹配应用实时性要求，主站 SYNC 周期与之协调。

6. **SDO 上传与下载**

   **SDO 下载（Download）** — 向从站写入参数:
   - 通常用于 PreOP 或 OP 状态下的参数配置（如增益、限值）。
   - 在 CoDeSys 中可以通过在从站 "Startup Parameters" 中添加上电 SDO 写入，或在程序中使用 `SDO_Write()` 功能块动态写入。
   - 基本参数：`Index`（16 位索引号）、`SubIndex`（8 位子索引号）、`Data`（写入的数据）、`DataLen`（数据长度）。
   - 示例：向伺服驱动器的 16#2000-01（子索引 1）写入 DINT 值 1000（加速度）。

   **SDO 上传（Upload）** — 从从站读取参数:
   - 使用 `SDO_Read()` 功能块动态读取。
   - 返回参数值及通信状态（`xDone`、`xError`）。
   - 示例：读取驱动器实际温度（对象 16#2100）。

   - 完成标准: SDO 读写功能块 `xDone` 返回 TRUE，`xError` 为 FALSE，读取/写入值符合预期。

7. **启动与监控 NMT 状态**
   - CANopen 从站状态转换: `Initialisation → PreOP → OP`
     - **Initialisation**: 自检和参数初始化
     - **PreOP**: SDO 通信可用，PDO 不可用。参数配置在此阶段进行
     - **OP（Operational）**: PDO 和 SDO 均可用，正常运行态。**PDO 通信仅允许在此状态进行**
   - CoDeSys 主站自动管理从站 NMT 状态转换（发送 NMT 启动命令 16#01 和 16#80）。
   - 监控手段：
     - `Heartbeat`（心跳）：从站定期发送 16#700 + NodeID 报文，主站监控其时间窗口。丢失心跳则触发紧急报警。
     - `Emergency`（紧急报文）：从站检测到故障时主动发送 16#080 + NodeID，内含错误代码和错误寄存器内容。
   - 完成标准: 所有从站稳定在 OP 状态，心跳正常接收，无紧急报文持续出现。

---

## B — 边界（Boundary）★

### 不要在以下情况使用此 skill
- **需要多轴高速硬实时同步（周期 < 1 ms，抖动 < 1 us）**: CANopen 的实时性和带宽上限（1 Mbps）不足以支撑；应选用 EtherCAT。
- **仅需连接 1-2 台简单 Modbus 设备**: 使用 Modbus RTU 配置更简便，成本更低。
- **设备节点 > 127 或总线距离 > 5000 m**: CAN 总线物理层限制了节点数和距离，或需加中继器/网桥，此时考虑工业以太网协议。
- **纯以太网上位机通信**: CANopen 基于 CAN 总线物理层，不能直接运行在标准以太网上（除非使用 CANopen over TCP 网关）。
- **安全完整性等级（SIL）应用**: CANopen 有对应的 CiA 304 安全扩展（CANopen Safety），但配置差异较大，不在本 skill 范围内。
