# Innovus Procedure

徐若航

## **目录**

Innovus数字后端设计流程可以概括为以下步骤：

1. **初始化设计**：加载网表和约束。
2. **布局规划**：确定芯片轮廓、摆放I/O引脚和宏单元。
3. **电源规划**：设计电源网络。
4. **布局**：摆放标准单元并优化。
5. **时钟树综合**：设计时钟网络。
6. **布线**：连接所有单元和引脚。
7. **签核**：验证设计的正确性并生成GDSII。

## **1. 初始化设计（Design Initialization）**

这是整个流程的第一步，目的是将综合后的网表和约束加载到Innovus中，并初始化设计环境。

```tcl
set init_top_cell $design
setImportMode -keepEmptyModule true
set init_lef_file [concat $tech_lef $lef_files]
set init_verilog $import_netlists
set init_pwr_net "VDD"
set init_gnd_net "VSS"
set init_mmmc_file "scripts/viewDefinition.tcl"

### read design
init_design
### mannual io placing mode
setIoFlowFlag 0 


### globalNetConnect
globalNetConnect VDD -type pgpin -pin VDD -all override
globalNetConnect VSS -type pgpin -pin VSS -all override
```

其中，需要指定`viewDefinition.tcl`

```tcl
### library set (-aocv or lvf, spatial socv)
create_library_set -name "fast" -timing\
    [list \
        /data/proj/library/CADENCE/GPDK045/v1.0/stdcell/liberty/fast_vdd1v2_basicCells.lib\
        /data/proj/library/CADENCE/GPDK045/v1.0/stdcell/liberty/fast_vdd1v2_basicCells_hvt.lib\
        /data/proj/library/CADENCE/GPDK045/v1.0/sram/liberty/MEM1_256X32_slow.lib\
        /data/proj/library/CADENCE/GPDK045/v1.0/sram/liberty/MEM2_128X32_slow.lib\
    ]
create_library_set -name "slow" -timing\
    [list ...
    ]
    
### rc corner
create_rc_corner -name "rc_best"\
    -preRoute_res 1.34236\
    -postRoute_res 1.34236\
    -preRoute_cap 1.10066\
    -postRoute_cap 0.960235\
    -postRoute_xcap 1.22327\
    -preRoute_clkres 0\
    -preRoute_clkcap 0\
    -postRoute_clkcap {0.969117 0 0}\
    -T 0\
    -qx_tech_file /data/proj/library/CADENCE/GPDK045/v1.0/tech/qrc/rcbest/qrcTechFile
create_rc_corner -name "rc_worst"\
    ...
    
### delay corner for each pvt (process voltage temperature) : library set + rc corner
create_delay_corner -name slow_rcworst\
    -library_set slow\
    -rc_corner rc_worst
create_delay_corner -name fast_rcbest\
    ....
    
### mode : (func + shift + capture)
create_constraint_mode -name functional \
    -sdc_files [list data/00_inputs/leon.func.slow_rcworst.sdc.gz]

### define view (mode + delay corner)
create_analysis_view -name func_slow_rcworst -constraint_mode functional -delay_corner slow_rcworst
create_analysis_view -name func_fast_rcbest  -constraint_mode functional -delay_corner fast_rcbest

### set analysis view status
set_analysis_view -setup [list func_slow_rcworst] -hold [list func_fast_rcbest]

```

---

## **2. 布局规划（Floorplan）**

Floorplan 是确定芯片的宏观结构，包括芯片大小、I/O引脚位置、宏单元（macro）摆放等。

1. **创建芯片轮廓**：
   - 使用 `floorplan` 创建芯片的轮廓（die area）和核心区域（core area）。
   - 例如：

   ```tcl
   floorPlan -site CoreSite -d 930 600.28 0 1.71 0 1.71 -fplanOrigin llcorner
   ```

   - `-site` 后面跟的名称可选项可能需要去lef或者lib工艺库文件中去找。常见的一般有`coreSite`, `unitSite`等。
   - `-d` 后面跟的数字是芯片的尺寸（包括了core area和margin），单位是微米。
2. **摆放I/O引脚**：
   - 使用 `place_io` 自动摆放I/O引脚，或使用 `editPin` 手动调整。
   - 例如：

   ```tcl
   set input_ports [dbGet [dbGet top.terms.direction input -p].name]
   editPin -pinWidth 0.08 -pinDepth 0.32 -fixOverlap 1 -unit TRACK -spreadDirection clockwise -layer 5 -spreadType start -spacing 2 -start 0 220 -pin $input_ports -fixedPin -side LEFT
   set output_ports [dbGet [dbGet top.terms.direction output -p].name]
   editPin -pinWidth 0.08 -pinDepth 0.32 -fixOverlap 1 -unit TRACK -spreadDirection counterclockwise -layer 5 -spreadType start -spacing 2 -start 0 170 -pin $output_ports -fixedPin -side RIGHT
   dbSet top.terms.pStatus fixed
   ```

3. **摆放宏单元+设置halo**：
   - 使用 `placeInstance` 或手动摆放宏单元（如RAM、ROM等）。
   - 例如：

   ```tcl
   placeInstance U_DCIM_32_4_64 100 100 -fplanOrigin llcorner
   setInstancePlacementStatus -status fixed -name U_DCIM_32_4_64
   ```

   - 为每个macro添加halo：

   ```tcl
   addHaloToBlock $halo_left $halo_bottom $halo_right $halo_top $macro
    addRoutingHalo -space $rhalo_space -top Metal11 -bottom Metal1 -inst $macro
    ```

4. **endcap**：
   - 使用 `addEndCap` 添加endcap。
   - 例如：

   ```tcl
   setEndCapMode -reset
   set endcap_top "FILL1"
   ...
   setEndCapMode -topEdge $endcap_top -bottomEdge $endcap_bottom
   setEndCapMode -leftEdge $endcap_left -rightEdge $endcap_left
   setEndCapMode -leftBottomCorner $endcap_bottom -leftTopCorner $endcap_top
   setEndCapMode -rightBottomCorner $endcap_bottom -rightTopCorner $endcap_top
   setEndCapMode -leftBottomEdge $endcap_left -leftTopEdge $endcap_left
   setEndCapMode -rightBottomEdge $endcap_right -rightTopEdge $endcap_right
   addEndCap -prefix "ENDCAP"
   ```

5. **welltap**
   - 使用 `addWellTap` 添加welltap。
   - 例如：

   ```tcl
   set welltap_prefix   "WELLTAP"
   # deleteFiller -prefix $welltap_prefix # delete existing filler
   addWellTap -prefix $welltap_prefix -cell $welltap_ref -cellInterval 70 -checkerBoard
   ```

---

## **3. 电源规划（Power Plan）**

电源规划是为芯片设计电源网络，确保电源和地线能够覆盖整个芯片，并满足IR Drop和EM（Electromigration）要求。

1. **创建电源环**：
   - 使用 `addRing` 创建电源环。
   - 例如：

   ```tcl
   selectInst [dbGet [dbGet top.insts.cell.subClass block -p2].name]
   setAddRingMode -stacked_via_bottom_layer Metal1 -stacked_via_top_layer Metal9
   addRing -nets {VDD VSS} -type block_rings -around selected -layer {top Metal9 bottom Metal9 left Metal8 right Metal8} -width {top 5 bottom 5 left 5 right 5} -spacing {top 1.25 bottom 1.25 left 1.25 right 1.25} -offset {top 5 bottom 5 left 5 right 5}
   ```

2. **创建电源条**：
   - 使用 `addStripe` 创建电源条。
   - 例如：

   ```tcl
   setAddStripeMode -stacked_via_bottom_layer Metal1 -stacked_via_top_layer Metal1
   addStripe -nets { VDD } -layer Metal1 -direction horizontal -width 0.120 -spacing 1.710 -set_to_set_distance 3.420 -start [expr 1.71 - 0.120 / 2.0] -stop $die_y1 -area $die_area -area_blockage $macro_region
   addStripe -nets { VSS } -layer Metal1 -direction horizontal -width 0.120 -spacing 1.710 -set_to_set_distance 3.420 -start [expr 1.71*2 - 0.120 / 2.0] -stop $die_y1 -area $die_area -area_blockage $macro_region
   ```

3. **添加电源连接**：
   - 使用 `sroute` 连接标准单元和宏单元的电源引脚。
   - 例如：`sroute -connect {blockPin padPin padRing corePin}`

4. **添加电源通孔**:
   - 使用 `editPowerVia` 创建电源通孔,手动优化电源通孔的布局，以满足设计规则（DRC）和电气规则（ERC）的要求。
   - 例如：`editPowerVia -add_vias 1 -skip_via_on_pin Standardcell -bottom_layer M1 -area {820 635 840 179} -top_layer M6`

---

## **4. 布局（Placement）**

布局是将标准单元（standard cells）摆放到芯片的核心区域，同时优化时序、拥塞和功耗。

1. **全局布局（Global Placement）**：
   - 使用`setPlaceMode`设置布局模式。如果需要设置时序分析的模式，可以使用`setAnalysisMode`。使用 `place_design` 进行全局布局。
   - 例如：

   ```tcl
   ### fix macros and ports
   dbSet [dbGet top.insts.cell.subClass block -p2].pStatus fixed
   dbSet [dbGet top.terms.name * -p].pStatus fixed

   setPlaceMode -reset
   setPlaceMode -place_detail_legalization_inst_gap 2
   set_dont_use [get_lib_cells "*_DFF*"]

   place_design
   ```

2. **优化布局（pre CTS）**：
   - 使用 `place_opt_design` 优化布局，减少时序违例和拥塞。然后可以用`optDesign -preCTS`命令进行全局优化。然后可以用`timeDesign`来生成分析和优化时序并生成报告。
   - 例如：

   ```tcl
   ### create path groups (reg2reg, in2reg, reg2out, in2out)
   group_path -name reg2reg -from [all_registers] -to [all_registers]
   group_path -name in2reg -from [all_inputs] -to [all_registers]
   group_path -name reg2out -from [all_registers] -to [all_outputs]
   group_path -name in2out -from [all_inputs] -to [all_outputs]
   setPathGroupOptions reg2reg -effortLevel high
   setPathGroupOptions in2reg -effortLevel low
   setPathGroupOptions reg2out -effortLevel low
   setPathGroupOptions in2out -effortLevel low

   ### drv
   set_interactive_constraint_modes [all_constraint_modes -active]
   set_max_transition -clock_path 200 [all_clocks ] -override
   set_max_transition -data_path 400 [all_clocks ] -override
   set_interactive_constraint_modes {}

   place_opt_design -expanded_views -out_dir ./myreports/${current_step}/innovus_placeopt -prefix "innovus_placeopt"
   optDesign -preCTS -drv -prefix place_drv
   timeDesign -preCTS -path myreports/timing_reports
   ```

3. **检查布局**：
   - 使用 `checkPlace` 检查布局是否存在问题，如重叠、未摆放的单元等。

---

## **5. 时钟树综合（Clock Tree Synthesis, CTS）**

时钟树综合是为芯片设计时钟网络，确保时钟信号能够均匀分布到所有时序单元。CTS 的主要任务是构建时钟树的结构，包括插入缓冲器（Buffers）和调整时钟网络的拓扑结构，以满足时序和功耗的要求。CTS会产生实际的绕线和通孔等物理结构。

1. **创建时钟树**：
   - 使用 `set_ccopt_property`设置时钟树属性。`ccopt_design` 创建时钟树。
   - 例如：

   ```tcl
   ### clock cells selection (latency (insertion delay), skew, clock power, clock em, long common path(cppr), duty cycle)
   set_ccopt_property buffer_cells {CLKBUFX12 CLKBUFX16 CLKBUFX6 CLKBUFX8}
   set_ccopt_property clock_gating_cells TLATNTSCA*
   # set_ccopt_property inverter_cells {}
   set_ccopt_property use_inverters false

   ### ccopt spec options
   set_ccopt_property effort extreme
   set_ccopt_property max_fanout 16
   set_ccopt_property target_skew 50
   set_ccopt_property target_max_trans 150
   set_ccopt_property target_insertion_delay auto

   ### ndr (Non-Default Rule) and route types
   ### ndr is used in clk routing because timing of clk tree is special and important
   add_ndr -width {Metal1 0.12 Metal2 0.14 Metal3 0.14 Metal4 0.14 Metal5 0.14 Metal6 0.14 Metal7 0.14 Metal8 0.14 Metal9 0.14 } -spacing {Metal1 0.12 Metal2 0.14 Metal3 0.14 Metal4 0.14 Metal5 0.14 Metal6 0.14 Metal7 0.14 Metal8 0.14 Metal9 0.14 } -name cts_2w2s
   create_route_type -name cts_route -non_default_rule cts_2w2s -bottom_preferred_layer Metal5 -top_preferred_layer Metal8
   ## net type : top trunk leaf
   set_ccopt_property route_type cts_route -net_type trunk 
   set_ccopt_property route_type cts_route -net_type top

   ### generate and read ccopt spec
   delete_ccopt_clock_tree_spec
   create_ccopt_clock_tree_spec -file ./${design}.ccopt_spec.tcl
   source ./${design}.ccopt_spec.tcl

   ### cts route options
   set_ccopt_property use_estimated_routes_during_final_implementation false

   ### routing related settings
   setDesignMode -bottomRoutingLayer Metal1 -topRoutingLayer Metal9
   setNanoRouteMode -routeWithTimingDriven true

   ### run ccopt
   ccopt_design -cts -prefix ccopt -expandedViews -outDir myreports/ccopt_reports
   ```

2. **优化时钟树**：
   - 在时钟树综合完成后，时钟网络的延迟和偏斜已经确定，需要将时钟从理想时钟切换为传播时钟。使用 `optimize_clock_tree` 优化时钟树，减少时钟偏差（skew）和延迟。
   - 例如：

   ```tcl
   ### set all clocks to propagated
   set_interactive_constraint_modes [all_constraint_modes -active]
   set_propagated_clock [all_clocks]
   set_interactive_constraint_modes {}

   ### route clock nets (with seperate globalDetailRoute command to minimize drc)
   deselectAll
   selectNet -clock
   dbSet selected.wires.status routed
   dbSet selected.vias.status routed 
   globalDetailRoute -select 
   ### actually ccopt_design included this command, but it's better to do it separately for better drc
   deselectAll
   selectNet -clock
   dbSet selected.wires.status fixed
   dbSet selected.vias.status fixed
   ```

3. **检查时钟树和设计时序**：
   - 例如

   ```tcl
   ## ccopt
   report_ccopt_clock_trees > myreports/${current_step}/report_ccopt_clock_trees.rpt
   report_ccopt_skew_groups > myreports/${current_step}/report_ccopt_skew_groups.rpt

   ### reports and check
   ## timing
   setAnalysisMode -checkType setup
   report_analysis_summary -late > myreports/${current_step}/report_analysis_summary.late.rpt
   report_timing -path_type full_clock -net -nworst 1 -check_type setup -max_paths 500 > myreports/${current_step}/report_timing.late.full.rpt
   report_timing -net -nworst 1 -check_type setup -max_paths 500 > myreports/${current_step}/report_timing.late.short.rpt

   setAnalysisMode -checkType hold
   report_analysis_summary -early > myreports/${current_step}/report_analysis_summary.early.rpt
   report_timing -path_type full_clock -net -nworst 1 -check_type hold -max_paths 500 > myreports/${current_step}/report_timing.early.full.rpt
   report_timing -net -nworst 1 -check_type hold -max_paths 500 > myreports/${current_step}/report_timing.early.short.rpt

   ## physical check
   reportGateCount
   verify_drc -limit 99999 > myreports/${current_step}/verify_drc.rpt
   checkPlace > myreports/${current_step}/checkPlace.rpt
   ```

4. **post-CTS 优化**：
   - 使用 `optDesign -postCTS` 进行后期优化，减少时序违例和DRC错误。

   ```tcl
   ### fix all clock routing
   deselectAll
   selectNet -clock
   dbSet selected.wires.status fixed
   dbSet selected.vias.status fixed

   ### fix macros and ports
   dbSet [dbGet top.insts.cell.subClass block -p2].pStatus fixed
   dbSet [dbGet top.terms.name * -p].pStatus fixed

   optDesign -expandedViews -setup -hold -drv -outDir "myreports/${current_step}/innovus_clockopt" -postCTS -prefix "innovus_clockopt"

   # timing and physical check again
   ...
   ```

---

## **6. 布线（Routing）**

布线是将所有单元和引脚通过金属线连接起来，形成完整的电路。

1. **布线设置**：
   - 使用 `setNanoRouteMode` 进行布线设置。
   - 例如

   ```tcl
   ### routing related settings
   setDesignMode -topRoutingLayer Metal9 -bottomRoutingLayer Metal1
   setNanoRouteMode -routeWithTimingDriven true
   setNanoRouteMode -routeWithSiDriven true
   setNanoRouteMode -routeWithLithoDriven true ;# DFM redundant via insertion
   setNanoRouteMode -drouteFixAntenna true -routeInsertAntennaDiode true
   setNanoRouteMode -routeAntennaCellName "ANTENNA" -routeAddAntennaInstPrefix "ROUTE_ANT_"
   ```

2. **全局+详细布线（Detail Routing）**：
   - 使用 `routeDesign` 进行详细布线。
   - 例如：

   ```tcl
   ### unifix the clock nets and will ECO route clock nets first before routing the remaining signal nets
   routeDesign

   ### if sepecify -globalDetail 
   ### it does not ECO route the clock nets prior to other signals so clock nets are not given priority
   routeDesign -globalDetail
   ```

3. **优化布线**：
   - 使用 `optDesign` 优化布线，减少时序违例和DRC错误。
   - 例如：`optDesign -expandedViews -setup -hold -drv -outDir "myreports/${current_step}/innovus_routeopt" -postRoute -prefix "innovus_routeopt"

---

## **7. 签核（Signoff）**

签核是最终验证设计的正确性，确保设计满足时序、功耗和物理规则。

1. **时序签核**：
   - 使用 `extractRC` 提取寄生参数，并使用 `timing_analysis` 进行时序分析。
   - 例如：`timing_analysis -setup -hold`
2. **功耗签核**：
   - 使用 `power_analysis` 进行功耗分析。
   - 例如：`power_analysis -average`
3. **物理签核**：
   - 使用 `verify_drc` 和 `verify_lvs` 检查DRC和LVS。
   - 例如：`verify_drc` 和 `verify_lvs`
4. **生成GDSII**：
   - 使用 `write_gds` 生成GDSII文件。
   - 例如：`write_gds design.gds`

---

## 附A: 优化模板

在布局、cts、布线优化前可以加上这些设置

```tcl
### routing related settings
setDesignMode -bottomRoutingLayer Metal1 -topRoutingLayer Metal9
setNanoRouteMode -routeWithTimingDriven true

### common optimization settings
setDesignMode -process 45
setAnalysisMode -analysisType onChipVariation -cppr both ;# OCV
setOptMode -addInstancePrefix "POSTCTS_" -addNetPrefix "POSTCTS_NET_" ;# TODO
setOptMode -powerEffort none
setTieHiLoMode -maxFanout 2 -honorDontTouch true -honorDontUse true -prefix "POSTCTS_TIE_" -cell {TIEHI TIELO}

### change optmization views
set_interactive_constraint_modes [all_constraint_modes -active]
source scripts/viewDefinition.postcts.tcl ;# TODO
set_propagated_clock [all_clocks]
set_max_transition -clock_path 200 [all_clocks ] -override
set_max_transition -data_path 400 [all_clocks ] -override
set_interactive_constraint_modes {}

### ndr (Non-Default Rule) and route types
add_ndr -width {Metal1 0.12 Metal2 0.14 Metal3 0.14 Metal4 0.14 Metal5 0.14 Metal6 0.14 Metal7 0.14 Metal8 0.14 Metal9 0.14 } -spacing {Metal1 0.12 Metal2 0.14 Metal3 0.14 Metal4 0.14 Metal5 0.14 Metal6 0.14 Metal7 0.14 Metal8 0.14 Metal9 0.14 } -name cts_2w2s
create_route_type -name cts_route -non_default_rule cts_2w2s -bottom_preferred_layer Metal5 -top_preferred_layer Metal8
## net type : top trunk leaf
set_ccopt_property route_type cts_route -net_type trunk 
set_ccopt_property route_type cts_route -net_type top

### create path groups
group_path -name reg2reg -from [all_registers] -to [all_registers]
group_path -name in2reg -from [all_inputs] -to [all_registers]
group_path -name reg2out -from [all_registers] -to [all_outputs]
group_path -name in2out -from [all_inputs] -to [all_outputs]
setPathGroupOptions reg2reg -effortLevel high
setPathGroupOptions in2reg -effortLevel low
setPathGroupOptions reg2out -effortLevel low
setPathGroupOptions in2out -effortLevel low

### set dont use cells
setDontUse *X1 true
setDontUse *X20 true
```

## 附B: 报告输出

```tcl
### reports and check (redirect)
file mkdir myreports/${current_step}
setAnalysisMode -checkType setup
redirect -tee myreports/${current_step}/report_analysis_summary.late.rpt { report_analysis_summary -late }
redirect -tee myreports/${current_step}/report_timing.late.full.rpt { report_timing -path_type full_clock -net -nworst 1 -check_type setup -max_paths 500 }
redirect -tee myreports/${current_step}/report_timing.late.short.rpt { report_timing -net -nworst 1 -check_type setup -max_paths 500 }

setAnalysisMode -checkType hold
redirect -tee myreports/${current_step}/report_analysis_summary.early.rpt { report_analysis_summary -early }
redirect -tee myreports/${current_step}/report_timing.early.full.rpt { report_timing -path_type full_clock -net -nworst 1 -check_type hold -max_paths 500 }
redirect -tee myreports/${current_step}/report_timing.early.short.rpt { report_timing -net -nworst 1 -check_type hold -max_paths 500 }

## physical check
set rblkg_prefix "MACRO_PG_RBLKG_RAIL"
deleteRouteBlk -name $rblkg_prefix
redirect -tee reportGateCount.rpt { reportGateCount }
redirect -tee myreports/${current_step}/verify_drc.rpt { verify_drc -limit 99999 }
redirect -tee myreports/${current_step}/checkPlace.rpt { checkPlace }
```

## 附C: qrcTechfile怎么得到？

### 各家RC提取的工艺文件关系

#### 参考文章

- [长文 - itf, ict, tluplus, capTable, nxtgrd, qrcTechFile以及它们之间的相互转换](https://www.shangyexinzhi.com/article/8134892.html)
- [tluplus-->itf-->ict-->captable](https://mp.ofweek.com/it/a556714674327)
- [ict转qrctechfile问题](https://bbs.eetop.cn/thread-871289-1-1.html)

| Cadence | Synopsys | 说明 | 备注|
| --- | --- | --- | --- |
| starRC | QRC | rc ext tools | |
| ict | itf | process file | 可以转化| 
| capTable | tluplus | rc model for APR tools | |
| qrcTechfile | nxtgrd | rc model for stand alone rc ext tools | |

### 步骤

现在假设已有itf文件，目标得qrcTechfile文件。

1. 找到innovus自带的itf_to_ict脚本，路径在`/cadtools/cadence/innovus20.10/share/voltus/gift/bin/itf_to_ict`.如果找不到，可以在/cadtools/目录下寻找：`find . -name 'itf_to_ict'`
2. 运行命令：`b /cadtools/cadence/innovus20.10/share/voltus/gift/bin/itf_to_ict input.itf output.ict`. 这样就可以生成ict文件了。

   > 注意：itf文件应该符合书写规范（至少是itf_to_ict能转换的书写规范），否则转换可能失败。

3. 查看Techgen是否能使用：`which Techgen`，可能的结果是`/cadtools/cadence/quantus20.12/tools/bin/Techgen`
4. 如果Techgen可用，运行命令：`b Techgen -si -multi_cpu 8 output.ict`即可由ict生成qrcTechfile文件。
