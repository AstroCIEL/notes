# chip_top

## 层次

```text
chip_top.v
├── PAD_TOP
├── config_dco
├── global_ctrl
│   ├── state_ctrl
│   ├── scan_chain_weight
│   ├── read_compare
│   ├── scan_chain_IF
│   └── scan_chain_psum
├── DCO
└── DCIM_32_4_64
```

没有包含DCIM_32_4_64模块的定义以及子模块，因为在综合的时候，DCIM_32_4_64是作为macro例化的，而非行为级模型综合生成。
DCO的rtl也是网表风格描述的，并非行为级风格描述，且例化了具体工艺库中的标准单元，因此在换工艺的时候也需要重写。

> DCO是一种通过数字信号控制输出频率的振荡器; PLL是一种通过反馈控制使输出信号与参考信号相位同步的电路, 包括相位检测器（PD）、低通滤波器（LPF）、压控振荡器（VCO）和反馈分频器。
> 在数字PLL中，DCO替代传统VCO，利用数字控制实现频率调节，提升系统灵活性和精度

## rtl文件

- PAD_TOP.v
- config_dco.v
  > 在sc_en有效的时候，将q从高位往低位移位，tdi作为q的新最高位。时钟信号为tck，一个时钟移一位。
  > 输出的q会作为dco的config，包括了cc_sel, fc_sel, freq_sel, div_sel


## chip_top

### 接口

```verilog
module chip_top  ( 
// pad of global ctrl
    input  wire                     pad_tck,
    input  wire                     pad_tdi,
    input  wire                     pad_tms,
    input  wire                     pad_load,
    input  wire [4:0]               pad_sel,

    input  wire                     pad_rstn,
    input  wire                     pad_cnt_en,
    input  wire                     pad_sel_cnt_en,
//  input  wire                     pad_clk,
    input  wire                     pad_cen,
    input  wire                     pad_cen_loop,
    input  wire                     pad_ren,
    input  wire                     pad_wen,
    input  wire                     pad_mem_en,
// pad of DCO
    input  wire                     pad_dco_en,
    input  wire                     pad_dco_ext_clk,
// 0:DCO; 1:External
    input  wire                     pad_clk_sel,
    output wire                     pad_clk_div,
    input  wire                     pad_clk_div_rstn,

// pad of output
    output wire                     pad_comp_out,

    output wire                     pad_tdo_psum,
    output wire                     pad_q_tdo,
    output wire                     pad_tdo
);
```

## PAD_TOP

### 接口

```verilog
module PAD_TOP (
// pad of global ctrl
    input  wire                     pad_tck,
    input  wire                     pad_tdi,
    input  wire                     pad_tms,
    input  wire                     pad_load,
    input  wire [4:0]               pad_sel,

    input  wire                     pad_rstn,
    input  wire                     pad_cnt_en,
    input  wire                     pad_sel_cnt_en,
//  input  wire                     pad_clk,
    input  wire                     pad_cen,
    input  wire                     pad_cen_loop,
    input  wire                     pad_ren,
    input  wire                     pad_wen,
    input  wire                     pad_mem_en,

// pad of DCO
    input  wire                     pad_dco_en,
    input  wire                     pad_dco_ext_clk,
// 0:DCO; 1:External
    input  wire                     pad_clk_sel,
    output wire                     pad_clk_div,
    input  wire                     pad_clk_div_rstn,

// pad of output
    output wire                     pad_tdo_psum,
    output wire                     pad_q_tdo,
    output wire                     pad_tdo,
    output wire                     pad_comp_out,

// pad of global ctrl
    output  wire                    tck,
    output  wire                    tdi,
    output  wire                    tms,
    output  wire                    load,
    output  wire [4:0]              sel,

    output  wire                    rstn,
    output  wire                    cnt_en,
    output  wire                    sel_cnt_en,
//  input  wire                     clk,
    output  wire                    cen,
    output  wire                    cen_loop,
    output  wire                    ren,
    output  wire                    wen,
    output  wire                    mem_en,

    input   wire                    comp_out,
// pad of DCO
    output  wire                    dco_en,
    output  wire                    dco_ext_clk,
// 0:DCO; 1:External
    output  wire                    clk_sel,
    input   wire                    clk_div,
    output  wire                    clk_div_rstn,

// pad of output
    input wire                      tdo_psum,
    input wire                      q_tdo,
    input wire                      tdo
);
```

### 图例

![pad_top](image.png)

