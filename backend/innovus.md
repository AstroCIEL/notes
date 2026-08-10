# Innovus

Innovus数字后端设计流程可以概括为以下步骤：

1. **初始化设计**：加载网表和约束。
2. **布局规划**：确定芯片轮廓、摆放I/O引脚和宏单元。
3. **电源规划**：设计电源网络。
4. **布局**：摆放标准单元并优化。
5. **时钟树综合**：设计时钟网络。
6. **布线**：连接所有单元和引脚。
7. **签核**：验证设计的正确性并生成GDSII。

---

## **0. 准备工作**

脚本模板见[innovus_flow](https://github.com/AstroCIEL/innovus_flow)。需要准备和修改以下文件：

- 综合网表：将综合后的网表放在./src/netlist/下
- 时序约束：将sdc文件（可以是综合生成的）放在./src/constraints/下
- sram数据：将用到的sram和macro以其名字在src/下创建文件夹，并在改文件夹中存放对应的cdl,gds,lef,lib文件
- 修改Makefile：将TOP变量的名称修改为顶层cell名
- 修改innovus全局配置：./scripts/General/global_config.tcl中，修改相应选项例如顶层金属等。
- 修改innovus初始化配置：./scripts/Step_Init/init_config.tcl中，修改工艺库ip路径，sram调用路径，pg名称。【重要】

---

## **1. 初始化设计（Design Initialization）**

这是整个流程的第一步，目的是将综合后的网表和约束加载到Innovus中，并初始化设计环境。

```tcl
set TopName $env(TOP)
set MMMCFile "../scripts/Step0_Init/mmmc.tcl"

# 以下init_*都是系统保留变量，而非自定义变量
set init_lef_file ${LefFile}
set init_verilog "${ROOT}/src/netlist/${TopName}_postsyn.v"
set init_top_cell ${TopName}
set init_mmmc_file ${MMMCFile}
set init_pwr_net {VDD_AXU}
set init_gnd_net {VSS_AXU}

# 对于顶层，POC也需要加在init_pwr_net中。但是他并不是顶层的端口。
```

其中，需要指定`mmmc.tcl`

```Tcl
# 1. 定义库集合 (Library Sets) ---
# 将之前整理好的 .lib 列表直接关联到 Corner 名称上
create_library_set -name lib_tt -timing ${LibFile_TT}


# 2. 定义 RC 寄生参数角 (RC Corners) ---
# 关联不同温度和电容条件的 qrcTechFile
create_rc_corner -name rc_typ -qx_tech_file "${T22}/PDK/PDK_0.8V_2.5V_1P9M_6X1Z1U_UT_ALRDL_StarRC_QRC/QRC/RC_QRC_cln22ulp_1p09m+ut-alrdl_6x1z1u_typical/qrcTechFile" -T 25


# 3. 定义延迟角 (Delay Corners) ---
# 将 [逻辑库] 与 [RC 参数] 绑定在一起
create_delay_corner -name delay_tt_typ   -library_set lib_tt -rc_corner rc_typ    


# 4. 定义约束模式 (Constraint Modes) ---
# 读取 SDC 时序约束文件
create_constraint_mode -name mode_func -sdc_files "../src/constraint/${TopName}_pnr.sdc"


# 5. 定义分析视图 (Analysis Views) ---
# 最终用于分析的视图：[延迟角] + [约束模式]
create_analysis_view -name func_tt_typ    -constraint_mode mode_func -delay_corner delay_tt_typ 


# 6. 设置当前分析目标 ---
# 告诉工具：用哪个视图查 Setup，哪个视图查 Hold
set_analysis_view -setup func_tt_typ -hold func_tt_typ
```

然后进行一些运行时的设置。要注意23版本的innovus在centos7上使用多线程时会出错。因此最好降级到22版本。

```tcl
# 设置cpu使用的核数
setMultiCpuUsage -localCpu 64

setDesignMode   -process            22 \
                -congEffort         auto \
                -earlyClockFlow     false \
                -expressRoute       false \
                -flowEffort         standard \
                -powerEffort        low \
                -bottomRoutingLayer M2 \
                -topRoutingLayer    M7
# 注意修改此处金属层

# 为了在innovus上方工具栏中出现calibre用于后续直接在innovus中跑DRC图形界面
source /home/EDAtools/mentor/Calibre2023/aoj_cal_2023.2_16.9/lib/cal_enc.tcl
```

---

## **2. 布局规划（Floorplan）**

Floorplan 是确定芯片的宏观结构，包括芯片大小、I/O引脚位置、宏单元（macro）摆放等。

1. **创建芯片轮廓**：

使用 `floorplan` 创建芯片的轮廓（die area）和核心区域（core area）。通常是拿综合结果报的cell总面积（对于T22，综合的结果是shrink之前的）先乘0.72（变成shrink后的版图面积），然后考虑pnr余量再除以0.7左右（希望pnr中整个density在0.7左右）得到floorplan的尺寸。

```tcl
floorPlan -flip s -s $core_sizex $core_sizey $core_margin $core_margin $core_margin $core_margin
```

- `-flip` 用s即second选项，使电源轨从VSS开始。具体见DRC章节中NW违例中的解释。默认是f即first，这将导致第一排endcap使用MX角度摆放引发NW与VDD断掉的情况。
- `-s` 后面跟的数字是core的尺寸（die包括了core和margin），单位是微米。

2. **摆放pin**：

- 使用 `place_io` 自动摆放I/O引脚，或使用 `editPin` 手动调整。
- 可以先在gui中添加，然后去cmd文件中将这个排布pin的指令复制一下，保存为tcl文件。之后就可以脱离gui直接使用脚本。
- 不推荐第一次就去手写脚本，浪费时间也容易写漏，还很麻烦。
- 要注意使用VHV排布，pin的金属层要符合这个走向。
- 对于子模块来说，尽量在一条边或者两条边出pin。例如改子模块将放在顶层的左下角，那么就从顶上或者右边出所有pin。不要在四周出pin！

3. **摆放宏单元+设置halo**：

- 使用 `placeInstance` 或手动摆放宏单元（如RAM、ROM等）。一组sram尽量摆的靠近，使得他们的延迟路径相似。建议一组sram横向间隔10，纵向间隔2
- macro都贴边放，不要放在中间，同一组不要排的太开。否则将会浪费面积。
- 为每个macro添加halo和placeblk。place blk非常重要，对于较紧密排布的macro例如sram，可以防止功能单元被place到两个macro之间的缝隙中。

```tcl
foreach bank $a_banks {
   # Get current bank ptr
   set inst_ptr [dbGetInstByName $bank]

   puts "processing curr_bank_name: $bank"
   # Delete old halo if exists
   catch {deleteHaloFromBlock $bank}

   # Add placement halo
   addHaloToBlock \
     [list 2 2 2 2] \
     $bank

   set bbox [dbGet $inst_ptr.box]
   set llx [lindex $bbox 0 0]
   set lly [lindex $bbox 0 1]
   set urx [lindex $bbox 0 2]
   set ury [lindex $bbox 0 3]
   set blk_llx [expr $llx - $soft_margin]
   set blk_lly [expr $lly - $soft_margin]
   set blk_urx [expr $urx + $soft_margin]
   set blk_ury [expr $ury + $soft_margin]

   # 在macro周围设置placeblk防止功能单元进入macro之间的缝隙
   createPlaceBlockage \
      -type soft \
      -box $blk_llx $blk_lly $blk_urx $blk_ury \
      -name "soft_blk_${bank}" \
      -density 50
}
```

---

## **3. 电源规划（Power Plan）**

电源规划是为芯片设计电源网络，确保电源和地线能够覆盖整个芯片，并满足IR Drop和EM（Electromigration）要求。

1. **globalNetConnect**:

把设计里的电源/地 pin，或者常量 0/1，对应绑定到指定的全局 power/ground net 上.

```tcl
# 所有标准单元的 VDD pin -> VDD_AXU
# 所有标准单元的 VSS pin -> VSS_AXU
globalNetConnect VDD_AXU -type pgpin -pin VDD -all -override
globalNetConnect VSS_AXU -type pgpin -pin VSS -all -override

#把sram中的vdd和vss分别绑定到VDD_AXU和VSS_AXU
foreach bank $a_banks {

    globalNetConnect VDD_AXU -type pgpin -pin VDDCE -sinst $bank -override
    globalNetConnect VDD_AXU -type pgpin -pin VDDPE -sinst $bank -override
    globalNetConnect VSS_AXU -type pgpin -pin VSSE  -sinst $bank -override
}

# tie-high -> VDD_AXU
# tie-low  -> VSS_AXU
globalNetConnect VDD_AXU -type tiehi
globalNetConnect VSS_AXU -type tielo
```

2. **创建电源环**：

一说28nm以下的工艺都没必要对子模块增加电源环，电源环的增加对于ir drop并没有正向的作用。

如果要加电源环，例如对于只有VSS和VDD两条电源环，其至少需要core margin为2*(width+spacing)。

```tcl
addRing -nets {VDD_AXU VSS_AXU} \
      -type core_rings \
      -follow core \
      -layer {top M6 bottom M6 left M7 right M7} \
      -width 1.4 \
      -spacing 1.1 
```

3. **添加电源轨**：

使用 `sroute` 添加电源轨。这是用M1的金属为全局打上间隔0.7um的VDD和VSS电源轨，用于给std cell供电。

可以先打电源轨再打电源条，也可以颠倒。总之powerplan就是只要保证让电源轨、电源条、电源环都连通起来形成一个健壮的电源网格就可以了。此处为何我们先打电源轨再打电源条，只是从经验上来看这样产生的DRC错误更少。

```tcl
sroute  -connect                    { corePin floatingStripe } \
        -nets                       { VSS_AXU VDD_AXU } \
        -layerChangeRange           { M1 M6 } \
        -crossoverViaLayerRange     { M1 M6 } \
        -targetViaLayerRange        { M1 M6 } \
        -allowJogging               0 \
        -allowLayerChange           0 \
        -checkAlignedSecondaryPin   1 \
        -deleteExistingRoutes
```

这边不允许jogging和layer change，是为了生成纯净的M1电源轨（仅生成M1金属），此时电源轨将不会与任何其他金属相连接，包括电源环，这是正常的。我们希望是在下一步打电源条的时候，让整个电源网络连通起来。

4. **创建电源条**：

电源条选择使用的金属通常就是这个模块所用到的最高层的金属了。例如对于一个顶层使用1P9M的工艺，在顶层肯定是需要留M9，M8来打顶层的金属条，因此对于子模块，我们最高就用到M7比较合适。另外，还要看当前模块中用到的macro（即当前模块的子模块）的顶层金属条是多高，例如sram compiler生成的都是最高层为M4的，因此我们当前模块的最高层金属（电源条）至少要是M5。

金属条的宽度和间距，可以参考该层金属在工艺rule中规定的金属密度来进行计算。例如M7金属要求密度是0.2，那么这边两个pg信号，width2.1，set2set间距21，密度=2.1*2/21=0.2。这边设置了create_pin，是为了直接给生成的金属条打上逻辑和物理pin，这样最后就无需再添加重合的物理和逻辑pg pin。

```tcl
setAddStripeMode    -stacked_via_bottom_layer   M1 \
                    -stacked_via_top_layer      M7
addStripe   -nets                               { VDD_AXU VSS_AXU } \
            -layer                              M7 \
            -direction                          vertical \
            -width                              2.1 \
            -spacing                            0.7 \
            -set_to_set_distance                21 \
            -start_offset                       13 \
            -block_ring_bottom_layer_limit      M1 \
            -block_ring_top_layer_limit         M7 \
            -padcore_ring_bottom_layer_limit    M1 \
            -padcore_ring_top_layer_limit       M7 \
            -create_pins 1 \
            -extend_to design_boundary
```

加完之后看一看macro之间的缝隙是否有stripe连到M1电源轨上。如果没有，必须在此处新增stripe（将set_to_set_distance选项改为使用number_of_sets选项），或者将附近的stripe移动到此处。总之要保证每一条电源轨都与stripe连接，即使是macro之间缝隙中的较短的电源轨。

5. **check**

最后做一下检查，在设计早期将电源问题解决掉。

```tcl
verify_drc -limit 99999                                -report ../report/postPowerplan/verify_drc.rpt 
verifyConnectivity -net {VDD_AXU VSS_AXU} -error 99999 -report ../report/postPowerplan/verify_connectivity.rpt
verify_PG_short -net {VDD_AXU VSS_AXU}                 -report ../report/postPowerplan/verify_pgshort.rpt
```

这个阶段可以忽略的违例是M1 dangling wire。其他违例大部分情况下都需要清空。

---

## **4. 布局（Placement）**

布局是将标准单元（standard cells）摆放到芯片的核心区域，同时优化时序、拥塞和功耗。

1. **加物理单元（Physical cell insertion）**：

加endcap，welltap，decap。T22标准单元库没有找到上下专用的boundary cell，因此可以直接使用左右的boundary cell替代。这边需要注意的是，在插physical cell之前就需要设置到placemode，主要是place_detail_legalization_inst_gap这个选项得生效。这是因为标准单元库中没有找到宽度为1的filler，因此我们需要保证左右功能单元和其他物理单元的放置需要满足间隔大于等于filler2的宽度才可以在最后填filler的时候成功填满。

|单元类型|核心作用|解决的底层物理问题|摆放位置与规则|
|--|--|--|--|
|Welltap|衬底 / 阱钳位防止 闩锁效应 (Latch-up)|按 DRC 规定的最大间距|等距穿插在行内|
|Endcap|边界封口保护|防止 工艺应力 / 光刻邻近效应 损坏器件|强制摆在每一行标准单元的最左端和最右端|
|Decap|局部储能稳压|缓解 动态 IR Drop 与电源同步开关噪声|充当 Filler，填充在行内未被占用的空位中|

```tcl
# 插入边缘保护单元
setPlaceMode	\
	-place_global_timing_effort medium	\
	-place_design_refine_macro	false	\
	-place_design_refine_place	true	\
	-place_detail_use_check_drc	true	\
	-place_detail_legalization_inst_gap 2 \
	-place_global_cong_effort high


set LEFT_ENDCAP "BOUNDARY_LEFTBWP7T30P140"
set RIGHT_ENDCAP "BOUNDARY_RIGHTBWP7T30P140"
# 5 site BOUNDARY 和 2 site FILL
set TOP_ENDCAP "${LEFT_ENDCAP} FILL2BWP7T30P140"
set BOTTOM_ENDCAP ${TOP_ENDCAP}
setEndCapMode -rightEdge ${LEFT_ENDCAP} -leftEdge ${RIGHT_ENDCAP} -topEdge ${TOP_ENDCAP} -bottomEdge ${BOTTOM_ENDCAP} -prefix "ENDCAP"

addEndCap

# 插入Welltap保护单元
addWellTap -cell "TAPCELLBWP7T30P140" -cellInterval 14 -checkerBoard -prefix "TAP"
addWellTap -cell "DCAP4BWP7T30P140" -cellInterval 14 -skipRow 3 -prefix "DCAP"
```

2. **Place**：

```tcl
deleteTieHiLo -prefix "TIE"
# 这边首次使用的不是placeDesign而是直接place_opt_design，取得更好的布局效果。前者是过时的指令
place_opt_design

setTieHiLoMode -cell "TIEHBWP7T30P140 TIELBWP7T30P140" -maxFanout 10 -maxDistance 20 -prefix "TIE"
addTieHiLo

# 首次布局完看一下时序
timeDesign -preCTS -outDir ../report/postPlace -prefix ${TopName}_postPlace

# 如果时序不太好，可以进一步优化布局。并多次执行直到时序满意
place_opt_design -incremental -out_dir ../report/postPlace_incr -prefix ${TopName}_postPlace_incr
```

---

## **5. 时钟树综合（Clock Tree Synthesis, CTS）**

时钟树综合是为芯片设计时钟网络，确保时钟信号能够均匀分布到所有时序单元。CTS 的主要任务是构建时钟树的结构，包括插入缓冲器（Buffers）和调整时钟网络的拓扑结构，以满足时序和功耗的要求。CTS会产生实际的绕线和通孔等物理结构。

1. **创建时钟树**：

```tcl
set_interactive_constraint_modes [all_constraint_modes -active]
# 要覆盖掉原来读进去的sdc中的一些设置，写在这里
set_clock_uncertainty -setup 0.15 [get_clocks {SYS_CLK V_CLK}]
set_interactive_constraint_modes {}

# 产生cts_spec文件
create_ccopt_clock_tree_spec -file ../backup/cts/ccopt_cts_spec.tcl
source ../backup/cts/ccopt_cts_spec.tcl

# Check if any clk_nets marked as ideal, which will not be synthesized !
# get_db clock_trees .nets -u -if {.is_ideal  ==  true}

set_ccopt_property update_io_latency true
set_ccopt_property target_skew 0.1

# 一些常用的设置
set_ccopt_property use_inverters true
# 选择驱动为8以上的单元
set_ccopt_property inverter_cells [ list        \
    CKND8BWP7T30P140LVT        \
    CKND12BWP7T30P140LVT        \
    CKND16BWP7T30P140LVT        \
]
set_ccopt_property clone_clock_logic true
# set_ccopt_property clone_clock_gates true
# set_ccopt_property merge_clock_gates true
set_ccopt_property merge_clock_logic true
# 指定非默认布线规则
add_ndr -name cts_w2s2 -width_multiplier {M5:M7 2} -spacing_multiplier {M5:M7 2}
create_route_type -name TRUNK -top_preferred_layer M7 -bottom_preferred_layer M5 -preferred_routing_layer_effort medium -non_default_rule cts_w2s2
create_route_type -name LEAF -top_preferred_layer M7 -bottom_preferred_layer M5 -preferred_routing_layer_effort medium
set_ccopt_property route_type TRUNK -net_type trunk
set_ccopt_property route_type LEAF -net_type leaf

# insertion delay
set memory_pin_name_list [get_object_name [get_pins -of_objects \
    [get_cells -hierarchical -filter "is_memory_cell==true"] \
    -filter "is_clock_pin==true"]]

foreach pin_name $memory_pin_name_list {
    set_ccopt_property insertion_delay 0.1 -pin $pin_name
}


# 22 ver
# 开始ccopt
ccopt_design -check_prerequisites
ccopt_design


set_interactive_constraint_modes [all_constraint_modes -active]
# 在时钟树综合完成后，时钟网络的延迟和偏斜已经确定，需要将时钟从理想时钟切换为传播时钟。
set_propagated_clock [list SYS_CLK]
set_interactive_constraint_modes {}
```

2. **优化时钟树**：

此处可以去检查一下时序，以及设计中需要满足的一些特殊的时序要求。如果还没有满足，可以进一步优化。

```tcl
optDesign -postCTS -drv -outDir ../report/postCTS_opt/ -prefix ${TopName}_postCTS_opt1_drv
saveDesign ../backup/${TopName}_postCTS_opt1_drv.enc

optDesign -postCTS -incr -outDir ../report/postCTS_opt/ -prefix ${TopName}_postCTS_opt2_incr
saveDesign ../backup/${TopName}_postCTS_opt2_incr.enc

optDesign -postCTS -hold -outDir ../report/postCTS_opt/ -prefix ${TopName}_postCTS_opt3_hold
timeDesign -postCTS -hold -outDir ../report/postCTS_opt/ -prefix ${TopName}_postCTS_opt3_hold
```

---

## **6. 布线（Routing）**

布线是将所有单元和引脚通过金属线连接起来，形成完整的电路。

1. **布线**：

```tcl
setNanoRouteMode	\
	-drouteFixAntenna               true \
	-drouteUseMultiCutViaEffort     high \
    -routeDesignRouteClockNetsFirst true \
   	-routeInsertAntennaDiode        true \
    -routeReserveSpaceForMultiCut   true \
   	-routeWithLithoDriven           false \
	-routeWithTimingDriven          true
	# Via 改成 High 让制造良率高一点
	# 预备Via的绕线空间


# set the native RC extraction mode
# engine:               possible value: preRoute | postRoute, default: preRoute
# effortLevel:          possible value: low | medium | high | signoff, default: low
setExtractRCMode    -engine                 postRoute \
                    -effortLevel            medium

# run routing or postroute via or wire optimization using the NanoRoute router
# # if specified without any arguments, it runs global and detail routing
 
# route_opt_design = routeDesign + optDesign -hold -setup

# man IMPOPT-6080 AAE-SI Optimization
# for optDesign
# 建议打开
setAnalysisMode -analysisType onChipVariation
setAnalysisMode -cppr both

route_opt_design -setup -hold
```

2. **优化布线**：

使用 `optDesign` 优化布线，减少时序违例和DRC错误。

```tcl
# 修DRV之前先检查一下确定要不要修
# 如果是时钟树ICG模块的Fanout,这里不会修的，不要浪费时间
report_constraint -drv_violation_type max_fanout -all_violators > max_fanout_vio.rpt
optDesign -postRoute -drv -outDir ./report/optDesign_postRoute1_drv/ -prefix optDesign_postRoute1_drv
saveDesign ./save/optDesign_postRoute1_drv.enc

optDesign -postRoute -incr -outDir ./report/optDesign_postRoute_incr/ -prefix optDesign_postRoute_incr
saveDesign ./save/optDesign_postRoute_incr.enc

optDesign -postRoute -hold -outDir ./report/optDesign_postRoute4_hold/ -prefix optDesign_postRoute4_hold
timeDesign -postRoute -hold -outDir ./report/optDesign_postRoute4_hold/hold/ -prefix optDesign_postRoute4_hold
```

3. **填充空白区域**

```tcl
set filler_cell [list \
    DCAP64BWP7T30P140    \
    DCAP32BWP7T30P140    \
    DCAP16BWP7T30P140    \
    DCAP8BWP7T30P140        \
    DCAP4BWP7T30P140        \
    FILL3BWP7T30P140        \
    FILL2BWP7T30P140        \
]

addFiller -cell ${filler_cell} -prefix "FILL"
```

4. **修复DRC**

检查DRC并使用ecoRoute自动修复。由于innovus中的DRC检查不够准确，往往是先保证innovus中的DRC clean然后再用calibre跑DRC，再根据其结果修DRC。注意要先加filler再导出gds跑calibre的drc。

```tcl
verify_drc -limit 99999 -report ../report/postRoute/verify_drc.rpt

# 如果有drc，先尝试让工具自己修
# 首先配置好 NanoRoute 的 ECO 模式
# 开启 ECO 绕线模式
setNanoRouteMode -route_with_eco true

# 开启深度搜索与修复，死磕顽固 DRC
setNanoRouteMode -route_detail_search_and_repair true

# 如果你想同时顺手修一下天线效应违例，可以打开这个 (看你的需求)
# setNanoRouteMode -route_antenna_diode_insertion true

# 关闭时序驱动绕线设置，专注于修复drc
setNanoRouteMode -route_with_timing_driven false
setNanoRouteMode -route_with_si_driven false

ecoRoute -fix_drc

# 修完之后，先导出一版gds，便于让calibre跑drc
# 需要将所有std cell， macro的gds作为merge file然后include进来
set streamOut_map "${T22}/Doc/CL-PR/PRTF_Innovus_22nm_001_Cad_V11_1a/PR_tech/Cadence/GdsOutMap/PRTF_Innovus_22nm_9M_6X1Z1U.11_1a.map"
streamOut   -mapFile    ${streamOut_map} \
            -merge      "${GdsFile}" \
            -mode       ALL \
            -unit       1000 \
            ../backup/signoff/${TopName}_postSignoff.gds2
```

导出一版gds然后用calibre跑DRC，详见DRC章节。建议在innovus的calibre选项下跑，这样跑完之后可以直接在innovus里面查看和修改。修改后`ecoRoute`然后重新导出gds并且再次跑DRC，保证DRC在calibre中clean（加dummy之前DN问题和LUP问题可以先不管，但是所有.S,.EN问题需要全部解掉）

---

## **7. 签核（Signoff）**

签核是最终验证设计的正确性，确保设计满足时序、功耗和物理规则。

1. **输出文件**

需要输出的文件包括：
- hcell.list：用于加快lvs，非必须
- lef：金属物理信息，必须
- sdf：延迟信息，必须
- lib：时序信息，必须
- gds：版图，必须
- v：verilog网表，用于转成cdl，必须
- cdl：由v通过v2lvs工具转化而来，用于lvs，必须
- spef：寄生参数信息，用于pt，非必须

```tcl
# Hcell list
set std_cell_list [dbGet head.libCells.name]
foreach cell $std_cell_list {echo "$cell $cell" >> ../backup/signoff/hcell.list}


# LEF
write_lef_abstract  -5.8 \
                    -cutObsMinSpacing \
                    -PGpinLayers        {M6 M7} \
                    -specifyTopLayer    M7 \
                    -stripePin \
                    ../backup/signoff/${TopName}_postSignoff.lef
# toplayer是用到的最高那一层
# pgpinlayer是所有power stripe/ring用到的所有金属层

# SDF
write_sdf   -min_view func_tt_typ \
            -typ_view func_tt_typ \
            -max_view func_tt_typ \
            -recompute_delay_calc \
            ../backup/signoff/${TopName}_postSignoff_func_tt_typ.sdf
# 需要指定typ_view，否则sdf中typ_view为空，可能对反标仿真造成问题

# lib
do_extract_model    -view func_tt_typ \
                    ../backup/signoff/${TopName}_postSignoff_func_tt_typ.lib 

set redundant_files [glob -nocomplain "../backup/signoff/model.asrt*"]
foreach redundant_file $redundant_files {
    file delete ${redundant_file}
}


# gds
set GdsFile xxx
set streamOut_map "${T22}/Doc/CL-PR/PRTF_Innovus_22nm_001_Cad_V11_1a/PR_tech/Cadence/GdsOutMap/PRTF_Innovus_22nm_9M_6X1Z1U.11_1a.map"
streamOut   -mapFile    ${streamOut_map} \
            -merge      "${GdsFile}" \
            -mode       ALL \
            -unit       1000 \
            ../backup/signoff/${TopName}_postSignoff.gds2
# 对于full chip，建议在导出gds前设置setStreamOutMode -virtualConnection false，并且在LVS的时候关闭virtual connect colon


# v
# set lvs_exclude_cells xxx可以在此处指定，也可以在之前的脚本如init_config中指定
# v中需要把filler，boundary，tap cell去除。decap不能去除。也就是说lvs_exclude_cells中需要包含filler，boundary，tap cell
saveNetlist -excludeCellInst    ${lvs_exclude_cells} \
            -excludeLeafCell \
            -flat \
            -phys \
            ../backup/signoff/${TopName}_flat_postSignoff.v
# 加了phys就会把VDD/VSS等电源端口也体现在网表中

saveNetlist -excludeCellInst    ${lvs_exclude_cells} \
            -excludeLeafCell \
            -topModuleFirst \
            ../backup/signoff/${TopName}_hier_postSignoff.v

# v转cdl。
# v2lvs option是链接的cdl文件，包括std cell，sram等的cdl。可以在此处设置也可以在之前的脚本如init_config中设置
set v2lvs_option xxx
set cur_dir [pwd]
echo "v2lvs   ${v2lvs_option} \
        -v ${cur_dir}/../backup/signoff/${TopName}_flat_postSignoff.v \
        -o ${cur_dir}/../backup/signoff/${TopName}.cdl" > ../scripts/Step6_SIgnoff/v2lvs_run.csh
exec chmod +x ../scripts/Step6_SIgnoff/v2lvs_run.csh
exec ../scripts/Step6_SIgnoff/v2lvs_run.csh

# spef
setExtractRCMode \
  -engine postRoute \
  -effortLevel high \
  -coupled true 
# 此处effort选择high而不是signoff。选择signoff反而不准
extractRC
rcOut -rc_corner rc_typ -spef ../backup/signoff/${TopName}_postSignoff_rc_typ.spef
```

2. **形式验证**

使用formality对pnr前后的网表进行形式验证，保证pnr过程没有对电路功能造成改变。该过程不需要svf指导文件（只在综合前后形式验证中需要）。示例脚本：

```tcl
#==============================================================================
# 1. 环境与库设置（含 .db 文件设置，关键部分）
#==============================================================================

#--- 1.1 设置搜索路径：让工具能找到库文件、设计文件
set search_path [list . \
    /home/user/project/rtl \
    /home/user/project/syn \
    /home/user/project/lib/std_cell \
    /home/user/project/lib/io \
    /home/user/project/lib/mem \
    /tools/synopsys/dc/libraries/dw ]

#--- 1.2 设置 link_library：用于功能链接，第一项必须为 "*"
#     通常包含标准单元库、IO库、Memory库、IP库以及DesignWare库
set link_library [list * \
    std_cell.db \
    io_cell.db \
    sram.db \
    pll_ip.db \
    dw_foundation.sldb ]

#--- 1.3 设置 target_library：通常与标准单元库一致
set target_library [list std_cell.db]

#--- 1.4 读入工艺库（.db 文件）到实现端容器
#     .db 文件包含标准单元的功能模型，网表中每个单元都需要有对应定义
read_db -i { \
    std_cell.db \
    io_cell.db \
    sram.db \
    pll_ip.db }

#==============================================================================
# 2. 参考设计（Reference, RTL）设置
#==============================================================================

#--- 2.1 读入 RTL 文件（支持 SystemVerilog 使用 -sv 选项）
read_verilog -sv -r [list \
    ../rtl/top.v \
    ../rtl/sub1.v \
    ../rtl/sub2.v ]

#--- 2.2 若 RTL 中使用 DesignWare IP，需设置 DW 根目录
set hdlin_dwroot "/tools/synopsys/dc/2019.12"

#--- 2.3 设置参考设计顶层
set_top r:/WORK/top

#--- 2.4 设置常量约束（例如扫描使能信号置 0，确保功能模式）
set_constant r:/WORK/top/scan_en 0
set_constant r:/WORK/top/scan_mode 0
set_case_analysis r:/WORK/top/test_mode 0

#==============================================================================
# 3. 实现设计（Implementation, 门级网表）设置
#==============================================================================

#--- 3.1 读入综合后网表
read_verilog -i ../syn/outputs/top_mapped.v

#--- 3.2 设置实现设计顶层（与参考端顶层名一致）
set_top i:/WORK/top

#--- 3.3 同样设置实现端扫描信号约束
set_constant i:/WORK/top/scan_en 0
set_constant i:/WORK/top/scan_mode 0
set_case_analysis i:/WORK/top/test_mode 0

#==============================================================================
# 4. 匹配与验证
#==============================================================================

#--- 4.1 执行关键点匹配（寄存器、端口等）
match

#--- 4.2 不允许任何失败点
set verification_failing_point_limit 0

#--- 4.3 执行形式验证
verify

#==============================================================================
# 5. 报告生成与 Session 保存
#==============================================================================

#--- 5.1 创建报告目录
if {![file exist ./reports/rtl2syn]} {
    file mkdir ./reports/rtl2syn
}

#--- 5.2 输出各类报告
report_passing_points  -to ./reports/rtl2syn/passing.rpt
report_failing_points  -to ./reports/rtl2syn/failing.rpt
report_aborted_points  -to ./reports/rtl2syn/aborted.rpt
report_unmatched_points -to ./reports/rtl2syn/unmatched.rpt
report_guidance        -to ./reports/rtl2syn/guidance.rpt

#--- 5.3 保存 session 便于后续调试
save_session fm_sess -replace

exit
```

3. **加dummy**

详见Dummy章节。加完dummy过一遍DRC和LVS。

4. **primetime**

工业界标准signoff必须使用primetime结合抽出的寄生参数进行最精确的时序验证。但是其debug门槛较高，我们可以先使用pt得到违例的值，然后回到innovus中，给uncertainty加上这个值，然后重新Route，也就是让innovus过修，从而保证余量。

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