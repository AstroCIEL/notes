# Synthesis using Design Compiler(Synopsys flow)

2025.2.6

## 概述

逻辑综合（synthesis）就是将前端设计工程师编写的RTL代码，映射到特定的工艺库上，通过添加约束信息，对RTL代码进行逻辑优化，形成门级网表。 对于逻辑综合步骤来说，我们通常使用的工具为Design Compiler，将一个RTL code在DC里做综合时，工具会先将代码转换成一个GTECH网表（generic technology （GTECH）netlist），然后在映射不同的工艺库形成真正的门级网表。

逻辑综合分为三个阶段：

- 转译（Translation）：把电路转换为EDA内部数据库(GTECH)，这个数据库跟工艺库是独立无关的；
- 链接（Linking）：link是介于Translate和Compile之间的一个小的步骤，主要目的在于解决设计中所有的reference。设计中的所有instances必须要找到它们自己的定义。这些instances可以分为三类：已经Translate的GTECH网表、stdcell、macros（RAM、ROM、IP、etc）
- 优化（Optimization）：根据工作频率、面积、功耗来对电路优化，来推断出满足设计指标要求(sdc约束)的门级网表；
- 映射（Mapping）：将门级网表映射到晶圆厂给定的工艺库上，最终形成该工艺库对应的门级网表。优化和映射可以合称为编译（compile）。

![流程](images/v2-5318012f149345d562cfa255c6034d94_r.jpg)

DC在综合过程中会把电路划分为以下处理对象：

- Design：A circuit that performs one or more logicafunctions
- Cell：An instance of a design or library primitivewithin a design
- Reference：The name of the original design that a cellinstance "points to"
- Port：The input or output of a design
- Pin: The input or output of a cell
- Net：The wire that connects ports to pins and/or pinsto each other
- Clock：A timing reference object in DC memory whichdescribes a waveform for timing analysis
- Library：A collection of cells and their associated metadata

![objects](images/image-3.png)

![对象](images/v2-d656d11449c1cce5dfea71318b5d37fc_r.jpg)

一个完整的顶层模块可以分为这几个部分：

![top](images/image-23.png)

上图是一个芯片的顶层设计，可以看到它被分层了两个层次：

- 最外边是芯片的Pad, Pad 是综合工具中没有的，也不是工具能生成的，它由 Foundry 提供，并由设计者根据芯片外围的环境手工选择

- 中间一层被分成四个部分，后三个部分 DC 不能综合，需要其他的办法来解决
  - 其中最里面那个称为 Core，也就是 DC 可以综合的全同步逻辑电路，
  - ASYNCH是异步时序部分，不属于DC的范畴；
  - CLOCK GEN 是时钟产生模块（可能用到 PLL），尽管有一部分同步电路，但也不符合综合的条件；
  - JTAG 是边界扫描的产生电路，这一部分可以由 Synopsys 的另外一个工具 BSD Compiler 自动生成。

## 流程

![process](images/1110317-20170325231311596-1606177239.png)

可以写一个makefile：

```Makefile
dc:
    dc_shell -f tcl/synthesis.tcl | tee ./dc.log
```

> |tee my.log 表示除了定向到 my.log⽂件外还要在屏幕中输出。

### 注意

对于2000.11 版的Design Compiler，用户可以通过四种方式启动Design Compiler，他们是——dc_shell 命令行方式、dc_shell-t 命令行方式、design_analyzer 图形方式和design_vision图形方式。其中后面两种图形方式是分别建立在前面两种命令行方式的基础上的。

![dcshell](images/image-22.png)

- dc_shell 命令行方式

该方式以文本界面运行Design Compiler。在shell 提示符下直接输入”dc_shell”就可以进入这种方式。也可以在启动dc_shell 的时候直接调用dcsh 的脚本来执行(dc_shell –f script)。目前这种方式用的已经不是很普遍。

- dc_shell-t 命令行方式

该方式是以TCL（Tool Command Language 后面章节将有介绍）为基础的，在该脚本语言上扩展了实现Design Compiler 的命令。用户可以在shell 提示符下输入”dcshell-t”来运行该方式。该方式的运行环境也是文本界面。也可以在启动dc_shell-t 的时候直接调用tcl 的脚本来执行(dc_shell-t –f script)。TCL 命令行方式是现在推荐使用的命令行方式，相对shell 方式功能更强大，并且在Synopsys 的其他工具中也得到普遍使用。

- design_analyzer 图形界面方式

Design Analyzer 使用图形界面，如菜单、对话框等来实现Design Compiler 的功能，并提供图形方式的显示电路。用户可以在shell 提示符下打“design_analyzer方式。Design_analyzer 图形方式是今后要经常用到的图形界面方式。由于它所对应的是dc_shell 的命令行方式，所以我们不能在design_analyzer 里运行tcl 命令。另外需要注意的是：Design analyzer 的工作模式不是用于编辑电路图的，它只能用于显示HDL语言描述电路的电路图。

- design_vision 图形界面方式

Design_vision 是与tcl 对应的图形方式，用户可以在shell 提示符下打”design_vision”来运行该方式。由于它是在Windows NT 下开发的，在工作站环境下不太普及。

> 不论dcsh 模式还是tcl 模式都提供了类似于unix 的shell 脚本的功能，包括变量赋值、控制流命令、条件判断等等，但是dcsh 模式的语法规则不同于tcl 的语法规则，所以两者的脚本不能通用。以下介绍的都是tcl脚本。

| Tcl mode           | dcsh mode           |
|--------------------|---------------------|
| get_cells *U*      | find(cell, *U*)     |
| get_nets *         | find(net, "*")      |
| get_ports CLK      | find(port, CLK)     |
| get_clocks CLK     | find(clock, CLK)    |
| all_inputs         | all_inputs()        |
| all_outputs        | all_outputs()       |

### 预综合过程（Pre-Synthesis Processes）：在综合过程之前的一些为综合做准备的步骤

#### DC启动

如果要一行一行地输tcl命令，可以启动dc_shell:

```tcl
dc_shell-t
```

#### 设置各种库文件

| Library type       | Variable          | Default                        | File extension |
|--------------------|-------------------|--------------------------------|----------------|
| Target library     | target_library    | {"your_library.db"}            | .db            |
| Link library       | link_library      | {"*", "your_library.db"}       | .db            |
| Symbol library     | symbol_library    | {"your_library.sdb"}           | .sdb           |
| DesignWare library | synthetic_library | {}                             | .sldb          |

可以用`list_libs`命令查看当前已加载的库

> synopsys不读取lib格式，所以需要将lib格式的库文件转换为db格式的库文件。

```tcl
# 启动library compiler shell
lc_shell 

# 转换
read_lib xxx.lib 
write_lib xxx0 -format db -output xxx.db
# 注意此处的xxx0是原先lib文件中lib名（并非文件名，虽然一般lib名和文件名一样）
```

根据不同的driver model，lib可以分成三类：Concurrent Current Source (CCS), Effective Current Source Model (ECSM), Non-Linear Delay Model (NLDM)

| Library Format | Accuracy | Computation Speed | Filesize | Usage |Suitable for Noise & Power Analysis | Definition |
| --- | --- | --- | --- | --- | --- | --- |
| CCS | High | Slow | Large | Sign off | Yes | A highly accurate library format using multiple current sources to model the behavior of digital gates. |
| ECSM | Medium | Moderate | Medium | Large design | Limited | A library format that uses effective current sources to approximate the behavior of digital circuits, balancing accuracy and computational complexity. |
| NLDM | Low | Fast | Small | Out-dated nodes| No | A library format that models delay and output transition using look-up tables (LUTs) as a function of input slew and output load. |

- **target_library（标准单元，综合后电路网表要最终映射到的库，db格式）**

  > 读入的HDL代码首先由Synopsys自带的GTECH库转成DC内部交换的格式，然后经过映射到目标库，最后生成优化的门级网表。
  > 目标库包含了各个门级单元的行为、引脚、面积、时序信息等，有的还包含了功耗方面的参数。
  > DC在综合时就是根据目标库中给出的单元路径的延迟信息来计算路径的延时，并根据各个单元的延时、面积和驱动能力的不同选择合适的单元来优化电路。

  ```tcl
  set search_path [list /path/to/library_dir]
  set target_library "target_library.db" ;# if search_path is set, target_library is the relative path to the search_path
  ```

  > 注意，在有些教程中会发现使用`set_app_var`而非`set`，其实都是可以的，因为`search_path`, `target_library`, `link_library`, `symbol_library`, `synthetic_library` 都是dc的保留变量，并非自定义的普通变量。

- **link_library（所有DC可能用到的库，以及购买的付费IP、存储器、IO、PAD、PLL等的库，db格式）**

  > 在link_library的设置中必须包含"*"，表示DC在引用实例化模块或者单元电路时首先搜索已经调进DC memory的模块和单元电路（已经translate过的GTECH网表）。
  > 该库中的cells, DC 无法进行映射（成门电路），例如：RAM, ROM 及Pad，在RTL 设计中，这些cells以实例化的方式引用
  > 如果在你的rtl代码中手动例化了foundary厂家提供的stdcell，当然GTECH网表中就会存在这个stdcell实例，那么link lib就必须包含这个stdcell的库（target_library）
  > 如果所有rtl中全部是逻辑描述，没有例化任何macro，sram，特定的std cell等，那么link_library可以为空。

  > 对于dcim的ip，是用模拟后端定制的，它作为一个macro，可以用db格式的库文件来表征引脚、时序信息等。在前端的时候，我们的完整rtl中可能写了一个dcim ip的行为级模型，但那个module只是用来行为级验证的，并不是用来综合的。综合的时候不要把这个行为级模型rtl包进来，而是采用db格式的库文件来作为顶层中dcim ip的reference。此时就应该将这个macro的db库作为link_library。这样在综合的时候，顶层模块例化了一个dcim ip，这个ip的reference并不是一个逻辑的行为级rtl module，而是一个db描述的macro。这样这个macro作为一个整体，内部的逻辑我们是不关心的，dc也是不会去综合这个macro内部结构的，我们关心的其实只是这个macro的pin和时序。

  ```tcl
  lappend search_path "/path/to/link_library_dir" ;# optional
  set link_library [list * $target_library link_library.db] ;# if search_path is set, link_library is the relative path to the search_path
  ```

- **symbol_library（符号库，单元电路显示的原理图库，sdb格式）**

  > Symbol Library用于显示原理图电路；Netlist或GTECH中每一个cell都有一个符号表示
  > 如果不指定symbol library，dc会根据从.db中识别出的Cell的功能找一个合适的符号显示

  ```tcl
  lappend search_path "/path/to/symbol_library_dir" ;# optional
  set symbol_library "symbol_library.sdb" ;# if search_path is set, symbol_library is the relative path to the search_path
  ```

- **synthetic_library（DesignWare library, 算数运算库）**

  > DesignWare：设计构建，一些可重用的电路设计结构，是一种HardCoding的电路，无法修改其结构，但与工艺库无关。
  > Synopsys免费提供基本的数学运算，如+/-/*/</>/<=/>=/selector等,在安装目录下，DC默认会加载。
  > 可以开发或购买额外的designware库，然后设置在synthetic_library中。
  > Multiple architectures for each macro allow DC to evaluate speed/area tradeoffs and choose the best implementation

  ![synthetic_library](images/image-4.png)

  ```tcl
  lappend search_path "/path/to/synthetic_library_dir" ;# e.g. /cadtools/synopsys/syn_vQ-2019.12-SP2/libraries/syn
  set synthetic_library "synthetic_library.sldb"
  ```

#### 其他初始设置

#### 读入设计文件

建议写一个list.f, 包含了所有需要读入的verilog文件

> 读入设计有两种实现方法实现方法：read_verilog 和 analyze & elaborate（实际上read_verilog是analyze与elaborate的打包操作）。analyze & elaborate  可以自由指定设计库，
> 并生成 GTECH中间文件前生成.syn 文件存储于 work 目录下，便于下次 elaborate 节省时间，我们一般选择analyze & elaborate的方法读入设计。

```tcl
analyze -f sverilog "path/to/list.f"
elaborate "top_module_name" <optional [-parameter "DATA_WIDTH = 8,ADDR_WIDTH = 8"]> 
# 只有elaborate可以设定顶层文件的parameter
current_design "top_module_name"
# 要综合哪个模块，就把哪个模块设置为当前设计
link
uniquify ;# Each instance gets a unique design name
```

### 设置环境

![environment](images/image-8.png)

- `set_operating_coditions`: 设置PVT条件

  ![corner](images/image-20.png)

  > PVT代表process（工艺），voltage（电压），temperature（温度）。corner（工艺角）是用来表征process的，包括了tt，ff，ss等。
  > 静态时序分析一般仅考虑Best Case和Worst Case，也称作Fast Process Corner 和Slow Process Corner，分别对应极端的PVT条件。这里的corner就是一种广义的角，与前面提到的狭义的（工艺）corner不同。
  > 时序分析中，best和worst是根据delay而言的，delay小的是best。但是时序越好，功率越差，这两者总是相反的。因此对delay而言best的时候，power可能是worst。

  ![pvt](images/image-21.png)

  | Corner Condition       | Cell Designator | Process (PMOS) | Process (NMOS) | Voltage    | Temperature |
  |------------------------|------------------|----------------|----------------|------------|-------------|
  | Worst                  | WCCOM            | Slow           | Slow           | 0.9*V_dd   | 125°C       |
  | Typical                | NCCOM            | Typical        | Typical        | V_dd       | 25°C        |
  | Best                   | BCCOM            | Fast           | Fast           | 1.1*V_dd   | 0°C         |
  | Low Temperature        | LTCOM            | Fast           | Fast           | 1.1*V_dd   | -40°C       |
  | Worst at Low Temperature | WCLCOM          | Slow           | Slow           | 0.9*V_dd   | -40°C       |
  | Maximum Leakage        | MLCOM            | Fast           | Fast           | 1.1*V_dd   | 125°C       |
  | Worst at 0°C           | WCZCOM           | Slow           | Slow           | 0.9*V_dd   | 0°C         |

  可以通过`report_library`命令查看当前库中所有单元的时序信息。

  ```tcl
  # 做建⽴时间分析的时候需要⽤到最差情况(max)的条件
  set_operating_conditions -max "slow_125_1.62"
  # 如果我们既要分析建⽴时间， ⼜要分析保持时间那么就要同时指定最差(max)和最好情况(min)
  # 先设定作保持时间检查的库，core_slow.db 和 core_fast.db 分别是最差和最好条件下的⼯艺库⽂件
  set_min_library core_slow.cb -min version core fast.db
  # 设定两种时间检查需要⽤到的⼯作条件。
  set_operating_conditions -max "slow_125_1.62" -min "fast_0_1.8"
  ```

  > 一般一个lib文件的文件名中就蕴含了一个pvt条件（例如ttg_v0p9_25c），这样的话一个文件就对应了一个pvt条件下的时序和功耗信息。或者也可以打开lib文件，查看其中的`operating_coditions`属性。因此可以在设置library的时候，读入多个条件的库文件，然后在这一步根据设置的环境条件来让工具选择使用哪一个环境下的库。

- `set_wire_load`: A wire load model is an estimate of a net’s RC parasitics based on the net’s fanout.

  > wire_load 模型的选择很重要，太悲观或太乐观的模型都将产生综合的迭带，在pre-layout 的综合中应选用悲观的模型(ss, high temp, low voltage)

  ![wire load](images/image-10.png)

  DC 在估算连线延时时，会先算出连线的扇出，然后根据扇出查表，得出长度，再在长度的基础上计算出它的电阻和电容的大小。若扇出值超出表中的值（假设为4），那么 DC就要根据扇出和长度的斜率（Slope）外推算出此时的连线长度来。

  > `set_wire_load_model`命令设置是⼀个模块内部连线的负载模型的估计;`set_wire_load_mode`命令设置是连接不同模块之间的连线的负载模型,包括了top，enclosed，segmented。

  ```tcl
  set_wire_load_model -name "10*10" -library my_lib.db
  set_wire_load_model -name "10*10" my_lib.db 
  current_design LOW
  set_wire_load_model  -name "10*10" 
  current_design TOP
  set_wire_load_mode enclosed
  set_wire_load_model  -name "20*20MIN" -min
  ```

- `set_drive`: Sets the rise_drive or fall_drive attributes to specified **resistance** values on specified input and inout ports. (Kohm)

  ```tcl
  set_drive 2.0 {A B C}
  set_drive 2 .0 "A B C"
  set_drive 2 [all_inputs]
  set_drive -rise 1 [get_ports B]
  set_drive -rise drive_of(-rise TECH_LIBRARY/INVERTER/OUT) [get_ports C]
  # drive_of : Returns the drive resistance value of the specified library cell pin.
  ```

- `set_driving_cell`: sets attributes on input or inout ports of the current design that specify that a library cell or output pin of a library cell drives the specified ports.

  > By default, DC assumes that the external signal has a transition time of 0.
  > Placing a driving cell on the input ports causes DC to calculate the actual (non-zero) transition time on the input signal
  > as though the specified library cell was driving it.

  ```tcl
  set_driving_cell -lib_cell AND2 {IN1}
  set_driving_cell -lib_cell INV -pin Z -library tech_lib [all_inputs]
  set_driving_cell -lib_cell INV -dont_scale {IN1}
  set_driving_cell -rise -lib_cell BUF1_TS -pin Z {IN1}
  set_driving_cell -fall -lib_cell DFF_TS -pin Q {IN1}
  ```

- `set_load`: Sets the load attribute to a specified value on specified ports and nets.

  > By default, DC assumes that the external load of ports is 0. You can specify some other constant value, or the `load of` command can be used to specify the external load as the pin load of a cell in your technology library.

  ```tcl
  set MAX_LOAD [load_of slow/AND2X1/A]
  set_load [expr $MAX_LOAD*15] [all_outputs]
  set_load 2 in1
  set port_load [expr 2.5+3*[load_of tech_lib/IV/A]]
  set_load $port_load [all_outputs]
  set_load [load_of tech_lib/IV/Z] {input_1 inoput_2}
  ```

- `set_fanout_load`: model the external fanout effects by specifying the expected fanout load values on output ports. Design Compiler tries to ensure that the sum of the fanout load on the output port plus the fanout load of cells connected to the output port driver is less than the maximum fanout limit of the library, library cell, and design.

  > fanout load is not the same as load. fanout load is a unitless value that represents a numerical contribution to the total fanout. Load is a capacitance value. Design Compiler uses fanout load primarily to measure the fanout presented by each input pin. An input pin normally has a fanout load of 1, but it can have a higher value.

> 在定义完环境属性之后，我们可以使用下面的几个命令检查约束是否施加成功: `check_timing`检查设计是否有路径没有加入约束; `check_design`检查设计中是否有悬空管脚或者输出短接的情况; `write_script`将施加的约束和属性写出到一个文件中，可以检查这个文件看看是否正确.

### 施加设计约束

![constraints](images/image-11.png)

#### 为什么需要时序约束

![dff](images/image-6.png)

- **setup constraints**: Qan从FFa/Q传到FFb/D不能超过一个T。这就是timing path的SETUP检查；SETUP检查要求数据不能传播太慢/太久
- **hold constraints**: Qan传播到FFb/D的过程不能太快，不能再第n个Clk的上升沿就被FFb采到。要求数据不能传播太快

#### 时序类型

![paths](images/image-7.png)

> Path: 每一条路径都由startpoint 和endpoint;

> statrpoint: input ports 或时序cell 的clock pins;

> endpoint: output ports 或时序cell 的data pins;

#### 时序路径（组）

DesignTime 对时序路径的分解是根据时序路径的起点和终点的位置来决定的。每一条时序路径都有一条起点和终点，起点是输入端口或者触发器的时钟输入端；终点是输出端口或者触发器的数据输入端，另外根据终点所在的触发器的时钟不同还可以对这些时序路径进行分组（Path Group），如下图电路中存在4条时序路径，3个路径组，CLK1 和 CLK2组分别表示他们的终点是受CLK1 和 CLK2 控制的，DEFAULT 组则说明他们的终点不受任何一个时钟控制。

![path](images/image-24.png)

再例如下⾯的⼀个电路，⼀共有 12 条时序路径和 3 条路径组。

![path2](images/image-25.png)

#### 设计规则的约束：technology-specific restriction

|command|object|
|-------|------|
|set_max_fanout | input ports and designs|
|set_fanout_load | output ports|
|set_load|ports or nets|
|set_max_transition|ports or designs|
|set_cell_degradation |input ports|
|set_min_capacitance|input ports|

- `set_max_transition`: set the maximum transition time for specified clocks, ports, or designs.(ns)

  ```tcl
  # Port: late riser.
  set_max_transition 2.0 late riser 
  # Design: TEST
  set_max_transition 2.0 TEST 
  # Clock Path: clk
  set_max_transition 2.0 [get_clocks clk1] 
  # Data Path: clk1
  set_max_transition 2.0 -datapath [get_clocks clk1]
  ```

  ![path](images/image-12.png)

- `set_max_fanout`: set the max_fanout attribute to a specified value on input ports and designs

  > set_fanout_load is design envoronment; set_max_fanout is a design constraint.
  > Design Compiler models fanout restrictions by associating a fanout_load attribute with each input pin and a max_fanout attribute with each output (driving) pin on a cell.

  ![fanout](images/image-9.png)

  > An input pin normally has a fanout load of 1, but it can have a higher value. The fanout load imposed by a driven cell (U3)is not necessarily 1.0.
  > Library developers can assign higher fanout loads (for example,2.0) to model internal cell fanout effects.
  > You can also set a fanout load on an output port (OUT1) to model external fanout effects.

- `set_max_capacitance`: It is set as a pin-level attribute that defines the  maximum total capacitive load that an output pin can drive.

  > The max capacitance design rule constraint allows you to control the capacitance of nets directly. (The design rule constraints max fanout and max transition limit the actual capacitance of nets indirectly.)
  > The max capacitance attribute functions independently, so you can use it with max_fanout and  max transition.

  ```tcl
  set_max_capacitance 2.0 [get_ports late_riser]
  set_max_capacitance 2.0 [current_design]
  set_max_capacitance 2.0 [get_clocks clk1]
  set_max_capacitance 2.0 -datapath [get_clocks clk1]
  ```

#### 优化的约束：design goals and requirements

> 对于时序约束，对于简单的设计，有包括如定义时钟，设置模块的输⼊输出延时等等；复杂的情况有非理想的单时钟网络(skew, latency)、同步多时钟网络（顶层有多个时钟，但是都是由同一个源时钟分频而来，所以称为同步）、异步多时钟网络、多周期路径等。

![delay](images/image-17.png)

- `create_clock`: 主要定义一个Clock 的source 源端、周期、占空比（时钟高电平与周期的比例）及信号上升沿及下降沿的时间点。

  > Clock 三要素：Waveform、Uncertainty 和Clock group。

  ![clock](images/image-18.png)

  ```tcl
  create_clock -name SYSCLK -period 20 -waveform {0 5} [get_ports2 SCLK]
  ```

  ```tcl
  # if waveform is not specified, it will be {0,period/2} for default.
  create_clock -period 12.5 [get_ports clkM]

  # Pre-layout阶段，估计时钟树的延时和抖动
  Set_clock_skew –delay 2.5 –uncertainty 0.5 clkM

  Set_clock_transition 0.2 clkM

  # do not re-bufer the clock network.
  set_dont_touch_network [get_clocks clkM]
  ```

- `create_generated_clock`: generated clocks 是从master clock 中取得的时钟定义。master clock就是指create_clock 命令指定的时钟产生点

  ![generated clk](images/image-19.png)

  ```tcl
  #定义master clock
  create_clock -name CLKP -period 10 -waveform {0 5} [get_pins UPLL0/CLKOUT]

  #在Q 点定义generated clock
  create_generated_clock -name CLKPDIV2 -source UPLL0/CLKOUT -master_clock CLKP -divide_by 2 [get_pins UFF0/Q]
  ```

  > 一般我们把时钟的源头会定义成create_clock，而分频时钟则会定义为create_generated_clock. 两者的主要区别在于CTS 步骤，
  > generated clock并不会产生新的clock domain, 而且定义generated clock 后，clock path的起点始终位于master clock,
  > 这样source latency 并不会重新的计算。这是定义generated clock 的优点所在。

- `set_clock_uncertainty`: 主要定义了Clock 信号到时序器件的Clock 端可能早到或晚到的时间。主要是用来降低jitter 对有效时钟周期的影响。值得注意的是，在setup check 中，clock uncertainty 是代表着降低了时钟的有效周期；而在hold check 中，clock uncertainty 是代表着hold check所需要满足的额外margin。

  ```tcl
  set_clock_uncertainty -from VIRTUAL_SYS_CLK -to SYS_CLK -hold 0.05
  set_clock_uncertainty -from VIRTUAL_SYS_CLK -to SYS_CLK -setup 0.3
  ```

- `set_dont_touch_network`: 常用于port 或net 阻止DC 隔离该net，和该net 相连的门具有dont_touch 属性。常用于CLK 和RST

  > 当一个模块例用原始的时钟作为输入，在该模块内部利用分频逻辑产生了二级时钟，则应对二级时钟output port 上设置set_dont_touch_network.
  > 当一个电路包含门时钟逻辑时，若在时钟的输入设置set_dont_touch_network，则阻止DC 隔离该门逻辑，导致DRC 发现时钟信号冲突，对门RESET 同样。

  > 在时钟pin 设置set_dont_touch_network,使DC 不会buffer up 时钟网，这个方法对于不包含门时钟的设计都可满足。对包含门时钟的设计，
  > 因set_dont_touch_network 会沿着时钟网组合逻辑进行传播，直到遇到寄存器，从而使门逻辑具有dont_touch 属性，会阻止DC size up 门逻辑，造成DRC 违例。
  > 为了避免这种情况，需移去set_dont_touch_network 并执行incremental compilation。

- `set_false_path` & `set_clock_groups`: 看起来比较深奥，没有去研究

- `set_input_delay`: constrains input paths. it specify how much time is used by external logic. Then DC calculates how much time is left for the internal logic(Tcycle-Tinput_delay). 定义信号相对于时钟的到达时间。指一个信号，在时钟沿之后多少时间到达。

  ![input delay](images/image-13.png)
  ![input delay2](images/image-14.png)

- `set_output_delay`: constrains output paths. it specify how much time is used by external logic. Then DC calculates how much time is left for the internal logic(Tcycle-Toutput_delay).

  ![output delay](images/image-15.png)
  ![output delay2](images/image-16.png)

- `set_max_area`: To reduce the area as much as possible, you can use `set max_area 0`. The area violation will disclose the total area of your design after synthesis.

- `set_cost_priority`

### 设计综合(compile)

#### 编译策略

- Top-down hierarchical compile：顶层设计和子设计在一起编译，所有的环境和约束设置针对顶层设计，虽然此种策略自动考虑到相关的内部设计，但是此种策略不适合与大型设计，因为 top down 编译策略中，所以设计必须同时驻内存，硬件资源耗费大。
- Time-budget compile ；
- Compile-characterize-write-script-recompile(CCWSR)；

#### 优化策略

To meet the constraints, DC performs several optimization strategies to reduce the delay and area of the design. 优化分为几个层次，从高到低分别为：

- architectural level optimization strategy: 最高层的优化
  - DesignWare 选择: 例如使用哪种加法器（行波、超前进位等等）
  - 共享⼦表达式(Sub-Expressions)
  - 资源共享(Resource Sharing)
  - 运算符排序(Operator Reordering)
- logical level optimization strategy: first flatten then structure(they are two oppose process). global influence
  - flatten : remove structure. `set_flatten true`(false for default) 目的在于通过移去中间变量，减少输入和输出之间的逻辑层次，提高速度，但会带来面积上的压力。
  - structure: minimize generic logic. `set_structure true`通过添加中间变量，使逻辑共享。但会增加逻辑的延时。常用于非关键时序电路，如：随机逻辑和有限状态机。
  > 由于 DC默认是用结构化的方式综合逻辑级电路，而且这种方式可以得到兼顾时序和面积的结果，因此我们可以先用这种方式优化。在优化后的电路中找出关键路径，看看关键路径上有没有符合使用 SOP电路的模块，王的这些亡便使用 SOP 的模块set_flatten，以便取得最佳的效果。
- gate level optimization strategy: make design technology-dependent. local influence
  - Combinational Mapping
  - Sequential Mapping

```tcl
compile –map_effort <low | medium(default) | high>
# high, it does critical path re-synthesis, but it will use more CPU time, in some case the action of compile will not terminate.
# 如果用“-map_effort high”,则DC 不可能structured 设计
# "compile -map_effort high"，相当于使用"compile_ultra"
compile_ultra <-area_high_effort_script | -timing_high_effort_script>

compile –incremental_mapping
# Perform incremental gate level optimization but no logical level optimization
# Incremental会导致大量的计算时间，但是对于将最差的slack减为0，这是最有效的方法。为了减少DC运算时间，可将那些已经满足时序要求的模块设置为dont_touch属性

compile –only_design_rule
# Perform only design rule fixing, take less time than regular compile because it is incremental.
```

### 时序分析

默认的时候 `report_timing` 报告每⼀条路径组中的最⻓路径。 报告⼀共分为 4 个部分:

![report timing](images/image-26.png)

- 第一部分显示了路径的基本信息一一工作状态是 slow_125_1.62，工艺库名称为SSC_core_slow，连线负载模式是 enclosed。接下来指出这条最长路径的起点是 data1（输入端口），终点是u4（上升沿触发的触发器），属于 clk 路径组，做的检查是建立时间检查（max）。这一部分的最后还报告了电路的连线负载模型。

  ![report timing2](images/image-27.png)

- 第二部分列出了这条最长路径所经过的各个单元的延时情况，分成三列：第一列说明的是各个节点名称，第二列说明各个节点的延时，第三列说明路径的总延时，后面所接的f或者r则暗示了这个延时是单元的哪个时钟边沿。例如图中的路径经过了一个反相器，一个二输入与非门，一个二输入 MUX，最后到达 D触发器。其中反相器的延时为 0.12ns，路径总延时为 1.61ns。

  ![report timing3](images/image-28.png)

- 第三部分说明了这条路径所要求的延时，它是设计者通过时序约束施加的。例如时钟周期为5ns，触发器的建立时间为0.19（从工艺库中得到），要满足建立时间的要求，组合路径延时必须在4.81ns 之内。

  ![report timing4](images/image-29.png)

- 第四部分为时序报告的结论，它把允许的最大时间减去实际的到达时间，得到一个差值，这个差值称为时序裕量（Timing margin），如果为正数，则说明实际电路满足时序要求，为负数则说明有时序违反。上图的裕量为3.20，说明最长路径满足建立时间的要求，且有3.20ns的裕量。暗含的意思是所有这个 clk 组的路径都满足建立时间的要求，并且裕量大于 3.20ns。

  ![margin](images/image-30.png)

除了报告电路综合后的时序之外， 还可以帮助我们诊断综合电路中存在的时序问题,如：

![eg](images/image-31.png)

这个例⼦仅仅列出了报告的第⼆部分， 报告的右边有四头鲸， 它们分别指出了电路中存在的四个问题：

- 第⼀个问题出现在 input external delay ⼀栏，这⼀栏对应的就是时序约束中施加的inputLdelay，它的值力 22.4ns，假设时钟周期为30ns，那么可以看出这个 input_delay 就已经占据了整个周期的70%多，因此留给下面单元的裕量就很少了。需要考虑是不是需要设置这么大的input_delay。
- 第二个问题出现在路径中的6个串连的buffer 上，它们对应的单元分别是 INVF、NBF、BF、BF、NBF 以及 NBF。就逻辑功能来看，6个显得多余，而且产生了不必要的延时。
- 第三个问题是由一个延时为 10.72的或门造成的。其他的单元延时一般不超过 2ns，一个 10.72的单元是不是因为负载过重引起的呢？值得仔细审查。
- 最后一个问题反映在这条路径穿过的层次上。从设计分层那一节可以知道，要使延时最小，应该把组合路径全部放在一个模块内。而这条组合逻辑同时穿过了 u_proc、u_proc/u_dcl、u_proc/u_ctl 以及u_init 四个层次，可以通过 group/ungroup 命令把层次重新组织一下。

## 参考资料

联系作者获取文件
