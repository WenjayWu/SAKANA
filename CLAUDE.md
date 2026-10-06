# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 这个仓库是什么

这是 **SAKANA** 的个人克隆仓库——一款基于树莓派 Pico 的开源电化学工作站（恒电位仪，支持 CV/LSV/IT/DPV）。原项目作者为 BSJ INSTRUMENT 泊州仪器，仓库使用者的目标是**复刻这台仪器并学习相关电子学知识**。

**版权约束（来自 README，必须遵守）**：基于本项目进行二次开发或发表内容必须引用 GitHub 页面；**禁止未经作者同意的商业用途**。

## 使用者背景与协作约定

- 使用者是（电）化学工作者，**没有电子/数字电路基础**。讨论电路问题时，请用初学者能理解的方式解释（可借助电化学类比：GND ≈ 参比电极、电压都是相对测量）。
- 日常交流使用**中文**。
- 使用者自加的笔记统一放在 [docs/](docs/)：**文件名用英文 kebab-case，内容用中文**。现有：
  - [docs/parts-shopping-guide.md](docs/parts-shopping-guide.md) — BOM 中未附链接元件（Pico、电阻、电容）的采购指南
  - [docs/electronics-learning-path.md](docs/electronics-learning-path.md) — 面向化学工作者的电子学学习路径
  - [docs/build-log.md](docs/build-log.md) — 复刻进度日志（**协作前先读它的"当前状态"**，从断点继续）

## 仓库结构与三大组成部分

仪器由三层构成，理解它们的交互是改动任何代码的前提：

1. **固件 [main.py](main.py)**（仓库根目录；README 说在 `Sources/` 下，已过时）
   - 运行在 Raspberry Pi Pico 上的 **MicroPython** 单文件固件，无构建步骤。
   - 文件内**内嵌（vendored）了三份驱动**：INA219（电流采样）、ADS1115（校准用 ADC）、MCP4725（DAC 输出），改驱动要改这个文件本身。
   - `handle_uart_commands()` 定义了**纯文本串口命令协议**（如 `cali`、`stopmeas`、`par <slope> <intercept>`、`correct_v`/`correct_i`、DPV 参数命令），上位机通过它控制仪器；`main_loop()` 是测量状态机（校准 / CV / i-t / DPV）。
   - 校准系数（斜率/截距、电压电流校正）保存在 Pico 端（`save_calibration_config()`）。

2. **上位机 [sources_SakanaController/](sources_SakanaController/)**
   - C# **WinForms** 桌面程序，`.NET 8`（`net8.0-windows7.0`），依赖 `System.IO.Ports`（串口）和 `WinForms.DataVisualization`（曲线图）。
   - 构建：`.NET 8 SDK` 环境下 `dotnet build sources_SakanaController/SakanaController.sln`（仅 Windows）。**本仓库没有任何测试**。
   - 通过 USB 串口（经 CH340 模块连 Pico 的 UART 引脚）与固件通信。

3. **硬件 [Hardware/](Hardware/)**（v1.0 / v2.0 两个版本，**新做请选 v2.0**，噪声低一个数量级）
   - `*.epro2` 是**立创EDA 专业版**工程；v2.0 工程内含 **3 块板**（主板 + INA219 贴片小板等），但 `Gerber_SAKANA_PCB.zip` **只导出了主板**——INA219 小板的 Gerber 需在立创EDA 中打开工程自行导出。
   - `Shell*/` 为 3D 打印外壳。`电化学工作站SAKANA使用说明书.pdf`（根目录）是中文使用说明书。

## 硬件层面的关键事实（不读完全部资料容易踩的坑）

- **Pico 自带 ADC 故意不用**（噪声大，是作者改外置方案的原因）：测量链路是 MCP4725（DAC 出电压）→ 模拟电路 → INA219（测电流）；ADS1115 仅用于校准。因此 RP2040 兼容板的 ADC 缺陷不影响本仪器。
- **INA219 采样电阻必须从 0.1 Ω 改为 10 Ω（1% 精度）**，否则电流分辨率不够。两条路线：商品 INA219 模块换贴片电阻（v1.0，适合零基础），或打样自制贴片小板（v2.0 作者方案，需热风枪）。
- v2.0 的两颗 LDO（GM1204 = SOT-223、GM1206 = SOT-23）是**小封装贴片，需要热风枪**。
- 板上电阻电容全部是**直插件**（1/4W 金属膜 1%、独石电容），BOM 未给链接的元件采购方法见 [docs/parts-shopping-guide.md](docs/parts-shopping-guide.md)。

## 修改代码时的注意点

- 固件与上位机的**命令协议是字符串匹配**，改动 `handle_uart_commands()` 的命令字时必须同步改 `Form1.cs` 里的发送逻辑，反之亦然。
- 2026-03-20 版本修复了扫速控制和曲线显示的 bug（固件 + 上位机同时改的），两者版本需配套；从旧版升级后要在软件的 Advance → Reset 清除预置校正参数。
- 固件用 `utime`/`micropython.const` 等 MicroPython 特有 API，不要用 CPython 习惯改它（也没有 lint/测试设施）。
