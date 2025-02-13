# 说明

## 服务器使用

1. /home/xxx/下的空间比较有限，基本上所有的文件以及跑eda都放在/work/home/xxx/下
2. 用eda的指令都需要在最前面加一个b,例如：

```bash
b innovus # 进入eda的shell之后就不用加b了
b make all # makefile正常编写，只是在make的前面加b即可
```

总之如果需要跟eda的指令遇到许可证等一些奇怪的报错，就试着在指令前面加一个`b`。

## 文件位置

1. rtl以及上一次流片所用的综合和后端脚本等都是在/work/home/rhxu/SMIC2025/
  - ip_core_review文件夹是ip_core, fp_core_review文件夹是fp_core, tensor_core_review（这个文件夹改过名了，之前是TCORE_04）是tensor_core, 包含的rtl文件应该在每个文件夹下面的front_end_sim/rtl_sim/list.F。也可看之前发的几个md文件
  - 顶层rtl是IO_TOP文件夹。三个core分别做后端后作为hard macro在io_top例化。这一次不使用pll了，改用dco。
  - 请确保使用最新的rtl（在2025.2.12 20:00更新过rtl,虽然是非常小的修改。2.13白天会在IO_TOP文件夹上传DCO的rtl因为现在还在验证DCO）
  - 上一次流片pr所用的脚本在dcim_v2_fp_core, dcim_v2_ip_core, dcim_v2_tensor_core文件夹下。这些文件夹都有macro_lef, macro_lib文件夹，里面是我们的dcim ip. 一些约束也可以参考之前的脚本
  - tensor_core有时也被称为multi_core。
  - 上一次流片最终的gds在top_dcim_project/calibre/drc/top_dcim_project_v2.gds，这次需要保持IO不变。

2. 使用的工艺是1p8m（6m_1tm_1mtt）0.9/1.8v平台，工艺库在/work/home/wumeng/SMIC22_INSTALL/SMIC28HKD_22ULP/
  - std cell在IP/STD/文件夹下
  - StarRC在IP/STD/SCC28NHKD_HDC30P140_RVT_V01p1a/CCI/
  - XRC在SPDK28HKD_0918_OA_CDS_V1.0_REV0.0/smic28HKD_0918_1P8M_6Ic_1YMc_1MTTc_ALPA2_oa_cds_2023_12_15_v1.0_rev0_0/Calibre/XRC/

## 芯片情况说明

1. 芯片运行频率（pll或者dco产生的clk信号）为500M，扫描链时钟信号（测试的时候用fpga产生的外部信号tck）为50M。
2. 最后layout的总面积大概为2300x1780, 裕度很大。
3. 不需要upf
4. 封装模式为wire bond
5. dcim_ip_bm是模拟流程设计出来的hard macro，但是端口电压都使用数字的标准电压（0.9v）

## 特殊要求

1. 需要综合
2. IO需要和上次流片的版图一致，方便测试。
3. 三个core分别做后端后作为hard macro在io_top例化。
4. 在空余的区域尽量多打decap
5. 顶层的四个电源，VDDCLO和VDDCM是给dcim ip供电的，VDD是给数字域供电的，VDDCLK给DCO的所有单元供电
6. pr时候top_bm_inst里面可能有难以meet的违例，因为有些是multicycle path，请与我们确认。

