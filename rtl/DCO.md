# DCO

## 片上时钟产生模块

- VCO（Voltage-Controlled Oscillator，电压控制振荡器）
- PLL（Phase-Locked Loop，锁相环）
- DCO（Digitally Controlled Oscillator，数字控制振荡器） 
- DPLL（Digital Phase-Locked Loop，数字锁相环）

> DCO通过数字信号控制频率，VCO通过模拟电压控制频率。DCO适合数字IC，VCO适合模拟或混合信号IC; 
> VCO是PLL的核心，PLL通过控制VCO的频率实现同步; 
> DPLL 的核心是数字化的相位锁定机制，而振荡器可以是 DCO 或其他类型的振荡器（如 VCO）。如果 DPLL 采用数字控制方式，DCO 是常见选择。

DCO可以独立使用，作为片上时钟的的产生模块。其输入都是数字信号。

## 原理图

> 这里介绍的DCO是基于贾老师设计的tsmc22的DCO修改而来。贾老师的原文件在`/project/work/home/tyiia/common/example/tsmc22nm/DCO 22nm`

![dco](image-4.png)

此次修改后的DCO基于smic22工艺。文件位置在`/work/home/rhxu/DCO_smic/`.

![DCO1](DCO1.png)

| pin       | in/out   |description                        |
|-----------|----------|-----------------------------------|
| RSTN      | in       | Reset divider, reset:0            |
| EN        | in       | Enable the DCO oscillation, enable: 1     |
| CC_SEL    | in[5:0]  | Coarse tune bits, the larger value, the lower CLK frequency    |
| FC_SEL    | in[5:0]  | Fine tune bits, the larger value, the lower CLK frequency      |
| EXT_CLK   | in       | External clock source (Backup clock)                           |
| CLK_SEL   | in       | Select clock. 0: DCO clock, 1: External clock                  |
| FREQ_SEL  | in[1:0]  | Divider for tile clock. 00: DCO clk, 01: DCO÷2, 11: DCO÷4      |
| DIV_SEL   | in[1:0]  | Divider ratio for test clock: 00: ÷1, 01: ÷2, 11: ÷4|
| CLK       | out      | High-speed clock for tile logics                               |
| CLK_DIV   | out      | Divided clock. Routed off chip for testing.                    |

CLK是用于片上所有逻辑的时钟，CLK_DIV一般是接到芯片外部用于测试。

> 修改sel配置后可能需要用RSTN复位后才能正常产生时钟信号。

![DCO2](DCO2.jpg)

以上是与顶层IO相连的部分以及与顶层模块的关系。

## 综合

注意DCO的rtl是用工艺库中的标准单元写成的，而非行为级模型。所以在工艺迁移的时候需要更换std cell。为了更准确的仿真，需要一些器件和导线的时序信息，所以需要先综合得到sdf文件后，反标到testbench中进行仿真。

> 在SDF格式中可以指定固有延迟（intrinsic delays），互连延迟（interconnect delays），端口延迟（port delays），时序检查（timing checks），时序约束（timing constraints）和路径脉冲（PATHPULSE）。

> 使用VCS读取SDF文件时，会将延迟值“反向标注（back-annotates）”到设计中，即在源文件中添加或者更改延迟值。


### design compiler综合脚本

具体文件在`/work/home/rhxu/DCO_smic/DCO_syn/scripts/script.tcl`

```tcl
set search_path "/work/home/wumeng/SMIC22_INSTALL/SMIC28HKD_22ULP/IP/STD/SCC28NHKD_HDC30P140_RVT_V0p1a/liberty/0.9v" ;# TODO
set target_library "scc28nhkd_hdc30p140_rvt_ss_v0p9_25c_ccs_lvf.db" ;# TODO
set link_library "* $target_library"

analyze -f sverilog "DCO.v" ;# TODO
elaborate "DCO"
current_design "DCO"
link
set_dont_touch "DCO"
uniquify
write_sdf "DCO.sdf" ;# TODO
```

## 仿真（反标时序）

文件在`/work/home/rhxu/DCO_smic/DCO_sim/`。

### 编写testbench

注意在testbench里加上语句：

```verilog
initial begin
    $sdf_annotate("DCO.sdf",U_DCO,,,"TYPICAL","1.6:1.4:1.2","FROM_MTM");
end
```

`$sdf_annotate(“sdf_file”[,module_instance][,“sdf_configfile”][,“sdf_logfile”][,“mtm_spec”][,“scale_factors”][,“scale_type”]);`

其中：

- "sdf_file"：指定SDF文件的路径。
- "module_instance"：指定反标设计的范围（scope）
- “sdf_configfile”：指定SDF配置文件
- “sdf_logfile”：指定VCS保存error 和warnings消息的SDF日志文件。也可以使用+sdfverbose runtime option来打印所有反标消息
- "mtm_spec"：指定延迟类型"MINIMUM（min）", "TYPICAL（typ）“或者"MAXIMUM（max）”。
- "scale_factors"：分别指定min:typ:max的缩放因子，默认是"1.0:1.0:1.0"
- “scale_type”：指定缩放之前延迟值得来源，“FROM_TYPICAL”,“FROM_MIMINUM”, “FROM_MAXIMUM"和"FROM_MTM" (default).
