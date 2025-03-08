# IO_TOP

> /IO_TOP/

## 特征概述

- 总共三个计算模块（ip core，fp core，tensor core），共用一套顶层IO以及一个DCO产生的时钟信号。顶层可以通过片选信号来选择使用 哪一个计算模块。
- 片上数字逻辑可以使用DCO产生的时钟，也可以使用芯片外灌时钟
- 各计算模块内部，在dcim ip macro外围通过扫描链来配置输入寄存器（串行输入并行输出给到ip），同样通过扫描链来收集ip输出的数据（并行输入ip算出的数据然后串行输出到顶层io）

### 各计算模块概述

- ip core：验证dcim_core (IN1b-W4b-O9b 矩阵-向量乘)是否正常工作；配有mbist自检电路，可自检SRAM读写功能
- fp core：在dcim_core的基础上，加入移位累加、PSUM后处理、指数对齐、输出格式转换等完整外围电路，支持多种定点/浮点数据格式(INT4,INT8,INT12,INT16,FP16,BF16,BBF16,FP8_E4M3, FP8_E5M2)的矩阵-向量乘计算，输出数据格式为INT32/FP32
- tensor core：由2x2个INT-DCIM ENGINE组成，可配置为执行通用矩阵-矩阵乘法。

## 层次

- IO_TOP(IO_TOP.v)
  - config_dco_inst config_cim(config_cim.v)
  - DCO_inst [DCO](/rtl/DCO.md)
  - fp_core_inst0 [fp_core](/rtl/fp_core.md)
  - ip_core_inst1 [ip_core](/rtl/ip_core.md)
  - multi_core_inst [multi_core](/rtl/tensor_core.md)

> multi core就是指tensor core。因为tensor core里例化了多个dcim ip，因此可以进行gemm。

![iotop](image-7.png)
![dco](DCO2.jpg)

## 接口

```verilog
module IO_TOP
(
    input IP_SEL,
    input DCO_RSTN,
    input DCO_SC_EN,
    input DCO_TDI,
    input CLK,
    input CLK_SEL,
    input TCK,
    input RSTN,
    input INSTRUCTION_VALID,
    input LOAD_START,
    input OUT_SEL,
    input OUT_LOAD,
    input OUT_SC_EN,
    input OUT_TDI,
    input MBIST_TEST_H,
    input MBIST_RESET_L,
    input EN_LOAD,
    input [2:0] CONFIG_SEL,
    input CONFIG_SC_EN,
    input CONFIG_TDI,
    input TCK_R,
    input RSTN_R,
    input INSTRUCTION_VALID_R,
    input MODE_R,
    input QUANT_ENABLE_R,
    input LOAD_START_R,
    input [2:0] OUT_SEL_R,
    input OUT_LOAD_R,
    input OUT_SC_EN_R,
    input OUT_TDI_R,
    input EN_LOAD_R,
    input [2:0] CONFIG_SEL_R,
    input CONFIG_SC_EN_R,
    input CONFIG_TDI_R,
    output OUT_TDO,
    output OUT_TDO_R,
    output CONFIG_TDO,
    output CONFIG_TDO_R,
    output MBIST_FAIL,
    output MBIST_DONE,
    output DCO_CLK_DIV,
    output DCO_TDO
);
```


## layout

![layout](image-29.png)