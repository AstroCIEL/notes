# EDA

| 类别    | Synopsys   | Mentor Graphics (Siemens EDA) | Cadence      |
|----------------|---------------|-------------|---------------------|
| 仿真    | **VCS (Verilog Compiled Simulator)**   | Questa (QuestaSim)     | Xcelium (原 Incisive 平台)    |
| 逻辑综合| **Design Compiler**   | Precision Synthesis   | **Genus Synthesis Solution**    |
| 形式验证| VC Formal (包括 FPV, LEC, CDC)         | Questa Formal / FormalPro | JasperGold (形式化平台，包括 FPV, CDC) / Conformal (LEC, Low Power)   |
| 时序分析| **PrimeTime**         |      | Tempus       |
| 布局布线| ICC/IC Compiler II  | Olympus / Aprisa | **Innovus Implementation System (原 Encounter EDI)**      |
| 模拟/混合信号设计 | Custom Compiler     | Tanner EDA | **Virtuoso**     |
| 寄生参数提取 | **StarRC**     | calibre xRC| **Quantus Extraction Solution (原 QRC)**     |
| 物理验证| IC Validator   | **Calibre** | Pegasus Verification System (与 Calibre 竞争)  |
| PCB设计 | -             | Xpedition | OrCAD/Allegro|
| 电路仿真| PrimeSim HSPICE (精确的 SPICE) | Eldo / AFS (Analog FastSPICE) | **Spectre / Spectre X**  |
| FPGA设计| Synplify      | Precision FPGA   | **Xilinx Vivado**|
| IP核    | DesignWare    | -         | Tensilica, Verification IP |
| DFT (可测试性设计) | DFT MAX      | Tessent   | Modus DFT    |
| 信号完整性分析 | PrimeSim HSPICE      | HyperLynx | Sigrity      |
| 功耗分析| PrimePower    | PowerPro  | Voltus       |
| 版图设计| Laker      | Tanner     | **Virtuoso**     |
| 3D IC设计      | 3DIC Compiler | -         | Integrity 3D-IC     |
| 系统级设计     | Platform Architect   | SystemVision     | System Design Enablement   |
| 验证IP  | Verification IP      | Questa Verification IP  | Cadence Verification IP    |

## 说明

- Synopsys： 在数字设计实现 (Design Compiler, IC Compiler II/Fusion Compiler) 和时序签核 (PrimeTime) 方面拥有领先地位。

- Cadence： 在定制/模拟设计 (Virtuoso, Spectre) 和形式化验证 (JasperGold) 方面具有强大优势。

- Siemens EDA (Mentor)： 在物理验证 (Calibre) 和数字仿真 (Questa) 方面非常强大，同时在 DFT (可测试性设计) 领域 (Tessent 系列) 也是领导者。

> S家优势是数字流程前端（vcs，dc）和签核（pt），C家优势是模拟电路（Virtuoso，Spectre）和物理实现（Virtuoso，Innovus），M家靠物理验证（Calibre）。
