# Alumy Embedded Debugging Guide

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![中文文档](https://img.shields.io/badge/文档-中文版-red.svg)](README_zh.md)

This project is a comprehensive documentation on embedded system debugging techniques, designed to help embedded developers master various debugging tools and methods, improving development efficiency and troubleshooting capabilities.

## 📋 Table of Contents

- [Introduction](#introduction)
- [Content Overview](#content-overview)
- [Quick Start](#quick-start)
- [Target Audience](#target-audience)
- [Contributing](#contributing)
- [License](#license)

## Introduction

In embedded system development, debugging is a crucial skill. Unlike traditional software development, embedded debugging involves hardware-software interaction and requires mastering specific tools and techniques. This guide systematically introduces common debugging methods in embedded development, from basic printf debugging to advanced JTAG/SWD debugging, covering various practical scenarios.

## Content Overview

This guide covers the following topics:

### 🔧 Debugging Tools

| Tool Type | Description |
|-----------|-------------|
| JTAG/SWD Debuggers | Hardware debuggers like J-Link, ST-Link, CMSIS-DAP |
| GDB Debugging | GNU Debugger applications in embedded systems |
| OpenOCD | Open On-Chip Debugger configuration and usage |
| Serial Debugging | UART log output and terminal debugging tips |
| Logic Analyzers | Signal timing analysis and protocol decoding |

### 📝 Debugging Techniques

- **Breakpoint Debugging** - Using hardware and software breakpoints
- **Memory Inspection** - Real-time memory viewing and modification
- **Register Monitoring** - Peripheral register status checking
- **Stack Backtrace** - Exception and error tracing
- **Performance Analysis** - Code execution time measurement and optimization
- **Fault Diagnosis** - Common issues like Hard Fault, memory overflow

### 🖥️ Development Environments

- Keil MDK
- IAR Embedded Workbench
- STM32CubeIDE
- VS Code + Cortex-Debug
- Eclipse + GNU MCU

### 📚 Special Topics

- **[Network Debugging Guide (Linux)](network_linux.md)** - Linux/Embedded network debugging commands and troubleshooting
- **[Network Debugging Guide (Windows)](network_win.md)** - Windows network debugging commands and troubleshooting

## Quick Start

1. Clone this repository:
   ```bash
   git clone https://github.com/alumy/alumy-docs.git
   cd alumy-docs
   ```

2. Browse the documentation according to the directory structure

3. Refer to specific debugging technique chapters as needed

## Target Audience

- 🎓 Embedded development beginners
- 💼 Embedded software engineers
- 🔬 Hardware debugging engineers
- 📚 Learners interested in embedded systems

## Contributing

Contributions are welcome! If you have any suggestions or find errors, please:

1. Fork this repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

### Documentation Standards

- Write documentation in Markdown format
- Store images in the `images/` directory
- Include necessary comments in code examples
- Keep language concise and clear

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details

---

<p align="center">
  <b>Alumy</b> - Making Embedded Development Easier
</p>
