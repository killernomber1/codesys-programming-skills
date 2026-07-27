# CODESYS 编程技能库

![CODESYS](https://img.shields.io/badge/CODESYS-3.5%2B-blue)
![License](https://img.shields.io/github/license/yourusername/codesys-programming-skills)

一套系统的 **CODESYS 编程技能**集合，涵盖 POU 设计、变量系统、ST 语言、通信协议、运动控制、任务调度与调试诊断。适用于 PLC 程序员和工业自动化工程师。

---

## 📚 目录

- [模块概览](#模块概览)
- [快速开始](#快速开始)
- [模块详情](#模块详情)
- [如何贡献](#如何贡献)
- [许可证](#许可证)

---

## 模块概览

| 序号 | 仓库子模块 | 说明 | 分类 |
|------|------------|------|------|
| 1 | [codesys-pou-design](./codesys-pou-design) | FUN/FB/PRG 分层设计与命名规范 | ST 编程 |
| 2 | [codesys-variable-system](./codesys-variable-system) | 变量作用域/数据类型/匈牙利命名/RETAIN | ST 编程 |
| 3 | [codesys-st-language](./codesys-st-language) | ST 语法（IF/CASE/FOR）+ 6 种语言混编 | ST 编程 |
| 4 | [codesys-language-select](./codesys-language-select) | ST/SFC/LD/FBD/CFC/IL 快速选型决策 | ST 编程 |
| 5 | [codesys-communication-select](./codesys-communication-select) | 5 种通信协议对比矩阵与选型框架 | 通信 |
| 6 | [codesys-modbus](./codesys-modbus) | Modbus RTU/TCP 组态全流程 | 通信 |
| 7 | [codesys-ethercat](./codesys-ethercat) | EtherCAT 主从站配置/DC同步/PDO映射 | 通信 |
| 8 | [codesys-canopen](./codesys-canopen) | CANopen PDO/SDO/对象字典/EDS配置 | 通信 |
| 9 | [codesys-softmotion](./codesys-softmotion) | SoftMotion 四层架构/PLCopen/CNC | 运动控制 |
| 10 | [codesys-task-scheduling](./codesys-task-scheduling) | 任务类型/优先级/看门狗核心不等式 | 工程 |
| 11 | [codesys-debug-diagnosis](./codesys-debug-diagnosis) | 五级调试体系/隐含检查/采样跟踪 | 工程 |

> 💡 每个子模块内均包含独立的 `README.md` 文件，提供详细说明、代码示例和最佳实践。

---

## 快速开始

1. **克隆仓库**
