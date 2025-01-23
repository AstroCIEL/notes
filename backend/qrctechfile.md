# qrcTechfile踩坑之路

## 各家RC提取的工艺文件关系

[长文 - itf, ict, tluplus, capTable, nxtgrd, qrcTechFile以及它们之间的相互转换](https://www.shangyexinzhi.com/article/8134892.html)
[tluplus-->itf-->ict-->captable](https://mp.ofweek.com/it/a556714674327)
[ict转qrctechfile问题](https://bbs.eetop.cn/thread-871289-1-1.html)

| Cadence | Synopsys | 说明 | 备注|
| --- | --- | --- | --- |
| starRC | QRC | rc ext tools | |
| ict | itf | process file | 可以转化| 
| capTable | tluplus | rc model for APR tools | |
| qrcTechfile | nxtgrd | rc model for stand alone rc ext tools | |

## 步骤

现在假设已有itf文件，目标得qrcTechfile文件。

1. 找到innovus自带的itf_to_ict脚本，路径在`/cadtools/cadence/innovus20.10/share/voltus/gift/bin/itf_to_ict`.如果找不到，可以在/cadtools/目录下寻找：`find . -name 'itf_to_ict'`
2. 运行命令：`b /cadtools/cadence/innovus20.10/share/voltus/gift/bin/itf_to_ict input.itf output.ict`

   > 注意：itf文件应该符合书写规范（至少是itf_to_ict能转换的书写规范），否则转换可能失败。

3. 查看Techgen是否能使用：`which Techgen`，可能的结果是`/cadtools/cadence/quantus20.12/tools/bin/Techgen`
4. 如果Techgen可用，运行命令：`b Techgen -si -multi_cpu 8 output.ict`即可生成qrcTechfile文件。

## 踩坑

1. 没有it