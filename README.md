# CS-to-EE-Journey

从计算机科学转向电气工程的学习记录与实验仓库。

这个仓库用于整理电路、电力系统、电力电子、智能电网安全以及 AI + EE 交叉方向的学习笔记，同时保存一些 MATLAB/Simulink 仿真实验模型。目标是把基础理论、工程直觉、论文阅读和可运行仿真逐步串起来，形成一条从 CS 背景进入 EE 方向的学习路线。

## 内容概览

- `knowledge/`：课程笔记、研究方向规划、论文阅读和专题整理。
- `knowledge/《电路》/`：电路基础笔记，包括预备知识、基尔霍夫定律、定理、相量法、运放、电容和电感等内容。
- `knowledge/《电力系统的稳态分析》/`：电力系统稳态分析相关笔记，包括预备知识和等值模型。
- `knowledge/papers/`：论文和研究主题整理，例如电力系统综述、电-氢混合能源系统优化、DT-MPC 等。
- `simulation/test/`：Simulink 测试模型，用于验证电力电子和整流电路的基础现象。

## 当前学习方向

重点关注以下方向：

- AI 驱动的电力电子技术
- 电力系统人工智能与智能电网
- 智能电机设计、控制与仿真
- 边缘 AI 与电气装备状态监测
- DC-DC 变换器、DAB/MAB、CLLC 谐振变换器等电力电子拓扑

## 仿真实验

`simulation/test/` 目录中包含若干 Simulink 模型：

- `single_phase_diode_half_wave_rectifier.slx`：单相二极管半波整流仿真。
- `single_phase_scr_half_wave_rectifier.slx`：单相 SCR 半波可控整流仿真。
- `buck_12v_to_5v_100khz.slx`：12V 转 5V、100kHz Buck 变换器仿真。

这些模型主要用于观察输入电压、负载电压、负载电流、触发脉冲、平均值和 RMS 等基础波形特征。

## 使用方式

学习笔记可以直接作为 Markdown 文件阅读，也可以用 Obsidian 打开 `knowledge/` 目录进行双链笔记管理。

仿真模型需要使用 MATLAB/Simulink 打开。建议从 `simulation/test/README.md` 开始查看每个模型的说明。

## 许可证

本项目采用 MIT License，详见 `LICENSE`。
