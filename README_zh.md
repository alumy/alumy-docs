# Alumy 嵌入式调试指南

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![English](https://img.shields.io/badge/Docs-English-blue.svg)](README.md)

本项目是一份全面的嵌入式系统调试技术文档，旨在帮助嵌入式开发者掌握各种调试工具和方法，提升开发效率和问题排查能力。

## 📋 目录

- [简介](#简介)
- [内容概览](#内容概览)
- [快速开始](#快速开始)
- [适用人群](#适用人群)
- [贡献指南](#贡献指南)
- [许可证](#许可证)

## 简介

嵌入式系统开发中，调试是一项至关重要的技能。与传统软件开发不同，嵌入式调试涉及硬件与软件的交互，需要掌握特定的工具和技术。本指南系统性地介绍了嵌入式开发中常用的调试方法，从基础的 printf 调试到高级的 JTAG/SWD 调试，涵盖了各种实用场景。

## 内容概览

本指南涵盖以下主题：

### 🔧 调试工具

| 工具类型 | 描述 |
|---------|------|
| JTAG/SWD 调试器 | J-Link、ST-Link、CMSIS-DAP 等硬件调试器使用 |
| GDB 调试 | GNU 调试器在嵌入式系统中的应用 |
| OpenOCD | 开源片上调试器配置与使用 |
| 串口调试 | UART 日志输出与终端调试技巧 |
| 逻辑分析仪 | 信号时序分析与协议解码 |

### 📝 调试技术

- **断点调试** - 硬件断点与软件断点的使用
- **内存查看** - 实时查看和修改内存内容
- **寄存器监控** - 外设寄存器状态检查
- **栈回溯** - 异常和错误追踪
- **性能分析** - 代码执行时间测量与优化
- **故障排查** - Hard Fault、内存溢出等常见问题诊断

### 🖥️ 开发环境

- Keil MDK
- IAR Embedded Workbench
- STM32CubeIDE
- VS Code + Cortex-Debug
- Eclipse + GNU MCU

### 📚 专题文档

- **[局域网调试指南 (Linux)](network_linux_zh.md)** - 嵌入式 Linux 系统工控设备局域网调试、IP 配置与连接故障排查
- **[局域网调试指南 (Windows)](network_win_zh.md)** - Windows 系统工控设备局域网调试、IP 配置与连接故障排查

## 快速开始

1. 克隆本仓库到本地：
   ```bash
   git clone https://github.com/alumy/alumy-docs.git
   cd alumy-docs
   ```

2. 根据目录结构浏览相关文档

3. 按需查阅具体调试技术章节

## 适用人群

- 🎓 嵌入式开发初学者
- 💼 嵌入式软件工程师
- 🔬 硬件调试工程师
- 📚 对嵌入式系统感兴趣的学习者

## 贡献指南

欢迎对本项目进行贡献！如果您有任何改进建议或发现错误，请：

1. Fork 本仓库
2. 创建您的特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交您的更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 打开一个 Pull Request

### 文档规范

- 使用 Markdown 格式编写文档
- 图片统一存放在 `images/` 目录下
- 代码示例需包含必要的注释
- 保持语言简洁明了

## 许可证

本项目采用 MIT 许可证 - 详情请参阅 [LICENSE](LICENSE) 文件

---

**Alumy** - 让嵌入式开发更简单
