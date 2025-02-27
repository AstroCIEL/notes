# tensor core

> /TCORE_04/

> /TCORE_04/front_end_sim/rtl_sim/list.f

```text
/rtl/dcim_ip_bm.v
/rtl/quantizer.v
/rtl/top.v
/rtl/top_cim.v
/rtl/input_collector.v
/rtL/mini_controller.v
/rtl/dcim_macro_32_bm.v
/rtl/input_register.v
/rtl/weight_collector.v
/rtl/config_cim.v
/rtl/output_fifo.v
/rtl/top_bm.v
/rtl/controller.v
/rtl/cim_weight_collector.v
```

## 层次

```text
top(top.v)
    config_cim_inst config_cim(config_cim.v)
    config_instruction_inst config_cim (config_cim.v)
    config_addr_inst config_cim(config_cim.v)
    top_cim_inst top_cim (top_cim.v)
        mini_controller_inst mini_controller(mini_controller.v)
        weight_collector_inst0~1 weight_collector (weight_collector.v)(2)
        input_collector_inst0~1 input_collector(input_collector.v)(2)
        cim_weight_collector_inst0~3 cim_weight_collector(cim_weight_collector.v)(4)
        input_register0~1 input_register(input_register.v)(2)
        top_bm_inst_0~3 top_bm(top_bm.v)(4)
            controller_inst controller(controller.v)
            dcim_macro_32_bm_inst dcim_macro_32_bm(dcim_macro_32_bm.v)
                dcim_ip_bm_inst dcim_ip_bm(dcim_ip_bm.v)(HARD MACRO)
                quantizer_inst quantizer(quantizer.v)
                output_fifo_inst output_fifo(output_fifo.v)
                shift_acc_block[0:15].shift_acc_0~3:shift_accumulator(dcim_macro_32_bm.v)
```

## top

### 接口

```verilog
module top(
    input                       clk,
    input                       tck,
    input                       rstn,
    input                       instruction_valid,//save instruction
    input                       load_start,
    input                       mode,
    input                       quant_enable,

    input                       out_sel,
    input                       out_load,
    input                       out_sc_en,
    input                       out_tdi,
    output                      out_tdo,

    input [2:0]                 config_sel,
    input                       config_sc_en,
    input                       config_tdi,
    output                      config_tdo
);
```
