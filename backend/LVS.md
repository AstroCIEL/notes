# LVS

LVS全称layout versus schematic，即版图与电路原理图的对比。在模拟流程（即使用virtuoso完成原理图设计和版图设计）中，需要比对一个cell的layout和schematic这两个view是否一致。对于数字流程（使用innovus数字后端生成版图并生成网表），则需要将生成的版图和网表进行比对。本文主要是从数字流程的视角来举例。

## 准备工作

如果virtuoso界面左上角工具栏没有calibre选项卡，参见[virtuoso](./virtuoso.md/##在virtuoso显示calibre选项卡)

## 说明文档

用于LVS的规则文件则可以在工艺库目录中找到：`/DISK2/Tech_PDK/TSMC_22NM_RF_ULL/PDK/PDK_20211230_LO_0.8V_2.5V_1P9M_6X1Z1U_UT_ALRDL_StarRC_QRC/Calibre/lvs/calibre.lvs`。在同一个目录下还可以找到一个叫`source.added`的文件，需要手动在cdl网表中include一下：

```spice
.INCLUDE "/DISK2/Tech_PDK/TSMC_22NM_RF_ULL/PDK/PDK_20211230_LO_0.8V_2.5V_1P9M_6X1Z1U_UT_ALRDL_StarRC_QRC/Calibre/lvs/source.added"
```

## 操作步骤

1. 打开需要做lvs的cell的版图，在上方菜单栏中选择 `Calibre -> Run nmLVS`。打开界面后会弹出窗口让你选择recent runset（如果是第一次使用则为空），你可以选择load其中一个runset，也可以不选，重新开始配置。
2. `Rules`选项卡中LVS Rule File选择工艺库中提供的lvs规则文件，例如`/DISK2/Tech_PDK/TSMC_22NM_RF_ULL/PDK/PDK_20211230_LO_0.8V_2.5V_1P9M_6X1Z1U_UT_ALRDL_StarRC_QRC/Calibre/lvs/calibre.lvs`。lvs规则文件不像drc文件那样需要修改文件头中的各种配置。
3. `Rules`选项卡中Run Directory请选择一个专门的文件夹，因为lvs会在该目录下生成很多文件。
4. `Inputs`选项卡中，`Run`可以选择hierarchical和flat，这是两种不同的验证方式，但是原则上来讲，两种验证方法对于 LVS 正确性没有影响。`Step`选择默认的layout vs netlist

> 层次化(hierarchical)验证保留了设计的层次结构，每个模块在验证过程中都作为一个独立的单元进行检查。层次化验证速度更快，可以减少相同子模块的重复验证计算。扁平化(flat)验证将设计的所有模块展开为一个平面结构，即所有模块的内部细节都被展开并作为一个整体进行检查。检查会更加全民啊，但是速度较慢、内存占用更高。

5. `Inputs`选项卡中`layout`相关的部分已经自动填充好，检查是否正确即可（如果就是在要检查lvs的cell的版图下打开的话一般就没问题）。在calibre2023版本（2021的版本中似乎没有这个选项所以不用管）的界面中会出现让你勾选`export from layout viewer"的选项，这个可以不选。如果勾选的话，下面的layout netlist文件是根据layout会自己生成的文件，并不需要提前准备一个已经存在的文件供读取。

![alt text](image.png)

6. `Inputs`选项卡中`netlist`相关的部分(在2023版本界面中称为`source`)，需要选择一个网表文件，这个文件就是lvs中的s（schematic），可以是verilog、vhdl、spice等格式。这里选择spice（不论是对于virtuoso中的schemtic还是数字后端生成的cdl文件）。spice file就选择数字后端生成的cdl文件。对于2023版本界面的不用勾选export from source viewer，这个选项是用于要比对的原理图来自virtuoso schematic的，如果勾选了，也是用于自动生成schematic对应的网表文件，所以无需准备一个已经存在的文件供读取。但是对于数字后端出的cdl，就不要勾选这个选项了。

![alt text](image-2.png)

7. `Options`选项卡（如果没有`Options`选项卡，对于2021版本界面则需要在lvs界面左上方菜单点击 Setup并勾选 LVS Options，对于2023版本界面则是settings--> show pages--> options，然后就可以看到options选项卡了）中，supply部分需要填写所有的pg端口名称

![alt text](image-3.png)

8. `Options`选项卡中在 Connect 中勾选 Connect nets with colon，以及 Don't connect nets by name，如下所示

![alt text](image-4.png)

9. 