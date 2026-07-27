---
name: codesys-modbus
description: |
  CoDeSys 中 Modbus RTU（串口）和 Modbus TCP（以太网）通讯的完整组态指南。
  当用户需要在 CoDeSys 项目中配置 Modbus 主站/从站、映射 I/O 变量、
  设置串口参数或连接 HMI/仪表/变频器时调用。
  不适用于实时性 < 5 ms 的高速运动控制场景。
source_book: 《CODESYS 基础编程及应用指南》 陆国君
source_chapter: 第9.2节 — Modbus 通讯
tags: [codesys, iec-61131-3, modbus, modbus-rtu, modbus-tcp, serial-communication, rs485, industrial-automation]
---

# Modbus RTU/TCP 通讯组态

## R — 原文（Reading）

> Modbus RTU 是基于 RS-232/RS-485 串口的二进制协议，Modbus TCP 是基于以太网 TCP/IP 的应用层协议。
>
> — 陆国君，《CODESYS 基础编程及应用指南》

---

## I — 方法论骨架（Interpretation）

Modbus 是工业自动化领域事实上的通用语言，核心优势在**简单、开放、设备支持广泛**。两种变体本质区别在物理层和报文封装：

- **Modbus RTU** — 走 RS-232（点对点）或 RS-485（多点）串口，报文以 16 位 CRC 校验保证数据完整性，无网络层开销，适合短距离（RS-232 < 15 m，RS-485 < 1200 m）、中低速（波特率 9600-115200 bps）场景。
- **Modbus TCP** — 走标准以太网（RJ-45），在标准 Modbus 报文前附加 MBAP（Modbus Application Protocol）报文头（7 字节），通过 TCP 端口 502 通信，天然支持路由和跨网段访问。

在 CoDeSys 中配置 Modbus 的核心思想是：**主站主动发起请求，从站被动响应**，通过功能码（如 01/02/03/04/05/06/15/16）决定操作类型（读线圈、读寄存器、写线圈、写寄存器），再通过 I/O 映射将数据绑定到 PLC 变量。

---

## A1 — 书中的应用（Past Application）

### 案例 1: 水处理系统多台仪表数据采集
- **问题**: 现场 12 台智能仪表（流量、压力、pH）均支持 Modbus RTU，需统一接入一台 CoDeSys PLC。
- **方法论的使用**: 使用 RS-485 总线将所有仪表挂接在同一总线上，PLC 做 Modbus RTU 主站，每台仪表分配不同从站地址（1-12）。在 CoDeSys 中添加 Modbus 串行主站设备，配置波特率 19200、8 位数据位、无校验、1 停止位，通过功能码 04 读取输入寄存器。
- **结果**: 12 台仪表数据刷新周期约 150 ms，满足工艺监控要求，总线布线与 4-20 mA 方案相比节省电缆成本 80%。

### 案例 2: 远程 HMI 通过以太网访问 PLC
- **问题**: 操作员站位于控制室，距现场 PLC 机柜 200 m，超出 RS-485 距离限制。
- **方法论的使用**: 选择 Modbus TCP 通信，PLC 作为 Modbus TCP 从站，HMI 作为主站。在 CoDeSys 中添加 Modbus TCP 从站设备，配置 IP 地址为 192.168.1.10，端口 502，将 HMI 需要读取的变量映射到保持寄存器（功能码 03/06）。
- **结果**: 通过标准以太网线缆实现稳定通信，后续扩展 Web SCADA 时无需更换通信架构。

---

## A2 — 触发场景（Future Trigger）★

### 用户会在什么情境下需要这个 skill?

1. 需要将支持 Modbus 协议的现场仪表、变频器、温控器等设备接入 CoDeSys PLC
2. 在旧设备改造项目中，存量设备仅提供 Modbus RTU 接口（RS-485 端子）
3. 需要实现 PLC 与上位机 SCADA / HMI / MES 之间的数据交换
4. CoDeSys 编程调试阶段出现 Modbus 通信超时或数据错位，需要排查配置

### 语言信号
- "Modbus 地址怎么映射"
- "Modbus RTU 和 TCP 选哪个"
- "CoDeSys 里怎么配 Modbus"
- "RS-485 接仪表读写不到数据"
- "Modbus CRC 校验怎么算"
- "功能码 03 和 04 有什么区别"

---

## E — 可执行步骤（Execution）

### Modbus RTU（串口）组态步骤

1. **添加 Modbus 串口主站设备**
   - 在 CoDeSys 设备树中右键 → "添加设备" → 选择 "Modbus Serial Master" 或对应品牌的具体型号。
   - 确认硬件支持串口（COM 端口、RS-232/RS-485 转换器）。
   - 完成标准: 设备树中出现 Modbus_Serial_Master 节点，无红色错误标记。

2. **配置串口通信参数**
   - 右键 Modbus 主站 → "参数"，设置：
     - `Baudrate`: 与所有从站统一（建议 9600 或 19200，距离越长速率越低）
     - `Data bits`: 通常 8
     - `Parity`: None / Even / Odd（与从站一致，推荐 None 简化调试）
     - `Stop bits`: 1 或 2
     - `Timeout`: 主站等待从站响应的超时时间（默认 1000 ms）
   - 完成标准: 参数与所有从站设备配置一致，物理接线确认 TX+/TX-/RX+/RX- 正确连接（RS-485 需 A/B 线反接确认）。

3. **添加 Modbus 从站**
   - 在 Modbus 主站下右键 → "添加设备" → 选择 "Modbus Slave" 或具体设备型号。
   - 设置 `Slave Address`（1-247，与设备拨码开关或内部配置一致）。
   - 完成标准: 从站出现在主站子节点，地址不重复。

4. **配置通道与映射 I/O 变量**
   - 在从站参数中配置通道类型（功能码）：
     - `Coils (0x)` — 数字量输出，功能码 01/05/15
     - `Discrete Inputs (1x)` — 数字量输入，功能码 02
     - `Input Registers (3x)` — 模拟量输入/只读寄存器，功能码 04
     - `Holding Registers (4x)` — 读写寄存器，功能码 03/06/16
   - 配置 `Quantity`（寄存器数量）和 `Offset`（起始地址，通常 0 基准）。
   - 编译后自动在 I/O 映射表中生成映射变量。可将变量重命名为有含义的名称。
   - 完成标准: 编译无错误，变量映射表中出现相应的 `%IW` 或 `%QW` 地址。

### Modbus TCP（以太网）组态步骤

1. **添加 Modbus TCP 设备**
   - 设备树中右键 → "添加设备" → 选择 "Modbus TCP Master"（主站）或 "Modbus TCP Slave"（从站）。
   - 完成标准: 设备出现在树中，无红色错误标记。

2. **配置 IP 地址和端口**
   - 主站模式参数设置：
     - `IP Address`: 从站的 IP 地址
     - `Port`: 502（标准 Modbus TCP 端口，可自定义）
     - `Unit ID`: 相当于 RTU 的站地址（通常 1-255）
   - 从站模式参数设置：
     - `Listen Port`: 本机端口（默认 502）
     - `Max Connections`: 允许的最大主站连接数
   - 完成标准: IP 地址与设备实际配置一致，通过 ping 确认网络可达。

3. **映射 I/O 变量**
   - 与 RTU 流程一致：配置通道类型（功能码）、数量、偏移量。
   - CoDeSys 自动生成 `stModbusMaster` / `stModbusSlave` 结构体变量，包括通信状态（`xBusy`、`xError`、`xDone`）。
   - 完成标准: I/O 映射表正常生成，状态变量可用于程序诊断。

### 常见配置参数速查

| 参数 | 说明 | 典型值 |
|------|------|--------|
| Baudrate | 串口通信速率 | 9600 / 19200 / 38400 / 115200 |
| Parity | 校验方式 | None / Even / Odd |
| Data bits | 数据位 | 7 / 8（通常 8） |
| Stop bits | 停止位 | 1 / 2 |
| Timeout | 响应超时 (ms) | 100-1000 |
| Slave Address | 从站地址 | 1-247 |
| Port | TCP 端口 | 502 |
| Unit ID | TCP 单元标识 | 1-255（通常与站地址相同） |

### 通信诊断技巧
- 在变量映射表中监视 `xError` 和 `udiError` 变量，常见错误代码：16#01（非法功能码）、16#02（非法数据地址）、16#03（非法数据值）、16#04（从站故障）。
- 串口通信不稳定时，降低波特率或增加 `Timeout` 值。
- 使用 ModScan32 或类似工具先在 PC 端测试通信，排除物理层问题。

---

## B — 边界（Boundary）★

### 不要在以下情况使用此 skill
- **需要 < 5 ms 确定性的高速运动控制**: 应选用 EtherCAT 而非 Modbus。
- **需要等时同步的multi-axis应用**: Modbus 无等时同步机制。
- **数据量 > 100 个寄存器且刷新周期 < 10 ms**: Modbus 性能瓶颈明显，应考虑工业以太网协议。
- **安全完整性等级（SIL）要求的通信**: Modbus 本身不含功能安全机制。
- **无线 Modbus 通信**: 无线链路上的 Modbus 需要额外处理丢包和延迟抖动，本 skill 仅覆盖有线场景。
