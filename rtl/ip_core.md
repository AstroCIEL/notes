# ip core

> /ip_core_review/rtl/
> /ip_core_review/front_end_sim/list.f

```text
/rtl/psum_collector.v
/rtl/top_cim.v
/rtl/dcim_macro_bm.v
/rtl/input_collector.v
/rtl/mini_controller.v
/rtl/dcim_macro_32_bm.v
/rtl/weight_collector.v
/rtl/config_cim.v
/rtl/cim_weight_collector.v
/rtl/mbist_a_g.v
/rtl/mbist_comp.v
/rtl/dcim_ip_bm.v
```

## 层次

```text
top_cim (top_cim.v)(8)
    mini_controller_inst mini controller(mini controller.v)
    weight_collector_inst weight_collector (weight collector.v)
    input_collector_inst input_collector(input_collector.v)
    mbist_a_g_inst mbist_a_g (mbist_a_g.v)
    dcim_ip_bm_inst dcim_ip_bm(dcim_ip_bm.v)(HARD MACRO)
    cim_weight_collector_inst cim_weight_collector(cim_weight_collector.v)
    psum_collector_inst psum_collector (psum_collector.v)
    mbist_comp_inst:mbist_comp(mbist_comp.v)
```

总体来看，这就是在ip（只能处理整型、没有时序累加）的外面加了输入、输出、控制、检测模块。
ip可以同时计算和写，因为在计算前会把要算的那些行锁存起来。

## rtl文件

- top.v
- mbist_a_g.v
  > mbist_a_g 当mbist_test_h有效的时候，进行检测。自动轮流依次检测多种模式，生成相应的wen，ren，addr等信号。并生成对应的正确数据test_data, 用于与cim的实际输出进行比对。
- mbist_comp.v
- psum_collector.v
  > psum_collector 顺序输入的寄存器so, 每次输入新的lsb，寄存器向高位移位；接受so并行输出load的寄存器q，并且可以以LENGTH位单位进行移位，并行输出LENGTH长度的msbs
- top_cim.v
- input_collector.v
  > input_collector 顺序输入的寄存器so, 每次输入新的lsb，寄存器向高位移位；接受so并行输出load的寄存器q，并且可以以LENGTH位单位进行移位，并行输出LENGTH长度的msbs
- mini_controller.v
  > mini_controller instruction里的都是不带cim_前缀的，例如ren，wen。然后通过instruction_valid来将instruction里的这些信号锁存，得到状态机的转换目标；然后在load_start有效时（时钟上升沿）允许状态转换（因此其实状态转换的目标只取决于load_start前一次instruction_valid时的instruction，因为它已经被锁存。）。状态机输出逻辑是根据状态输出cim_前缀的信号，例如cim_ren, cim_wen等等，这些输出信号会输入到dcim ip里。

  > 第一组状态机：REN_GLOBAL, REN_LOCAL, WEN_LOCAL之类的。当处于IDLE状态时，当load_start拉高的时候，状态才开始转换。在状态没有带LOOP的情况下，cnt达到instructin里给出的cnt上限就变回IDLE。如果是带LOOP的，则只要loop信号有效，就一直继续状态，不回到IDLE。因此如果进入LOOP状态，想要回到IDLE，必须让instruction valid一次来让所有信号归零，从而回到IDLE。

  > 第二组状态机：SCEN，SCEN_LOOP。该状态下输出的是cim_cen有效。cim_cen将输入到input_collector, 在有效时，q以LENGTH为单位进行移位，并行输出高LENGTH位msbs。
- weight_collector.v
  > weight_collector 顺序输入的寄存器so, 每次输入新的lsb，寄存器向高位移位；接受so并行输出load的寄存器q，并且可以以LENGTH位单位进行移位，并行输出LENGTH长度的msbs
- config_cim.v
  > config_cim 顺序输入，并行输出的寄存器。每次输入新的lsb，寄存器向高位移位
- cim_weight_collector.v

## top

### 接口

```verilog
module top(
    input                       clk,
    input                       tck,
    input                       rstn,
    input                       instruction_valid,//save instruction
    input                       load_start,

    input                       out_sel,
    input                       out_load,
    input                       out_sc_en,
    input                       out_tdi,
    output                      out_tdo,

    input                       en_load,
    input [2:0]                 config_sel,
    input                       config_sc_en,
    input                       config_tdi,
    output                      config_tdo,

    input                       mbist_test_h,
    input                       mbist_reset_l,

    output                      mbist_fail,
    output                      mbist_done
);
```

## dcim_ip_bm

> designed through analog flow, unsynthesizable.

> 内部寄存器大小为256x528位，或（64x4）x（16x33）位，即64列4bit权重，33行尺寸为16的subarray。

> 输出是单bit输入feature和4bit权重的按列累加和，没有时序上的累加。（时序上的累加需要在外部接一个shift accumulator）

### 接口

```verilog
module  dcim_ip_bm(  //rstn,
    input CLK, WEN, REN_GLOBAL, REN_LOCAL,
    input   [1:0]  EMA_RSA,
    input   [1:0]  EMA_RWL,
    input   [`ADDR_WIDTH-1:0]      ADDR,
    //input  rstn;  //for input bit shifter and shift-accumulator
    input   [`EXP_WIDTH-1:0]       I_EXP,
    output  [`EXP_WIDTH*`DCIM_COL/2-1:0]    O_EXP,
    input   [`DCIM_COL-1:0]        MASK,
    input   [`SIGN_CONFIG-1:0]     SIGN,

    input   [`MEM_WIDTH-1:0]       D,
    output  [`MEM_WIDTH-1:0]       Q,
    input   [`DCIM_ROW-1:0]        IFN,

    output  [`EXP_WIDTH-1:0]       I_EXPDRV,
    output  [`SIGN_CONFIG-1:0]     SIGNDRV,
    output  [`DCIM_ROW-1:0]        IFNDRV,
    output  [(`OUT_WIDTH_MIN-`IN_WIDTH_MIN)*`DCIM_COL-1:0]  PSUM // reg -> wire
);
```
