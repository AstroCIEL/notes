# IO_TOP

> /IO_TOP/

## 层次

```text
IO_TOP(IO_TOP.v)
    config_pll_inst config_cim(config_cim.v)
    PLL_ADAPTIVE_inst PLL_ADAPTIVE(HARD MACRO)
    fp_core_inst0 fp_core(HARD MACRO)
    ip_core_inst1 ip_core(HARD MACRO)
    multi_core_inst multi_core(HARD MACRO)
    PDDWUW0408SDGH_V/H (io cells)
    DECAP_345x85
    DECAP_750x502
    DECAP_1850x122
```

## top

### 接口

```verilog
module IO_TOP
(
    input IP_SEL,
    input PLL_RSTN,
    input PLL_SC_EN,
    input PLL_TDI,
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
    output PLL_CLK_DIV,
    output PLL_TDO
);
```
