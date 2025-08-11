# DRC(calibre flow)

## 准备工作

### 在virtuoso显示calibre选项卡

如果virtuoso界面没有calibre选项卡，则可以在工作目录下创建一个.cdsinit文件并写入以下内容：

```text
;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

;
; check CALIBRE_HOME
;
cal_home=getShellEnvVar("CALIBRE_HOME")
if( cal_home==nil then
    cal_home=getShellEnvVar("MGC_HOME")
    if( cal_home!=nil then
        printf("// CALIBRE_HOME environment variable not set; setting it to value of MGC_HOME\n");
    )
)

if( cal_home!=nil && isDir(cal_home) && isReadable(cal_home) then

    ; Load calibre.skl or calibre.4.3.skl, not both!

    if( getShellEnvVar("MGC_CALIBRE_REALTIME_VIRTUOSO_ENABLED") && 
        getShellEnvVar("MGC_REALTIME_HOME") && dbGetDatabaseType()=="OpenAccess" then
      load(strcat(getShellEnvVar("MGC_REALTIME_HOME") "/lib/calibre.skl"))
    else
      ; Load calibre.skl for Cadence versions 4.4 and greater
      load(strcat(cal_home "/lib/calibre.skl"))
    )

    ;;;;Load calibre.4.3.skl for Cadence version 4.3
    ;;; load(strcat(cal_home "/lib/calibre.4.3.skl"))

else

    ; CALIBRE_HOME is not set correctly. Report the problem.

    printf("//  Calibre Error: Environment variable ")

    if( cal_home==nil || cal_home=="" then
        printf("CALIBRE_HOME is not set.");
    else
        if( !isDir(cal_home) then
            printf("CALIBRE_HOME does not point to a directory.");
        else
            if( !isReadable(cal_home) then
                printf("CALIBRE_HOME points to an unreadable directory.");
            )
        )
    )
    printf(" Calibre Skill Interface not loaded.\n")

    ; Display a dialog box message about load failure.

    hiDisplayAppDBox(
        ?name           'MGCHOMEErrorDlg
        ?dboxBanner     "Calibre Error"
        ?dboxText       "Calibre Skill Interface not loaded."
        ?dialogType     hicErrorDialog
        ?dialogStyle    'modal
       ?buttonLayout   'Close
    )
)

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
```

这样就可以调用calibre的功能了。

### 说明文档

T1CentOS/T2CentOS服务器上，在工艺库安装路径`/DISK2/Tech_PDK/TSMC_22NM_RF_ULL/Doc/CL-DR/TN22CLDR001_1_5.pdf`。在DRC debug的时候对任何violation有问题应该在该文档中查找说明以及图例。

用于DRC的规则文件则可以在工艺库目录中找到：`/DISK2/Tech_PDK/TSMC_22NM_RF_ULL/PDK/PDK_20211230_LO_0.8V_2.5V_1P9M_6X1Z1U_UT_ALRDL_StarRC_QRC/Calibre/drc/calibre.drc`。可以拷贝一份到自己的工作目录，对其中的文件头设置进行修改，用于当前设计的DRC。

## 标准操作步骤

0.  Prepare drc rule file, and **change the header** in it. Make sure the configurations are suited for your design(e.g. whether it is full_chip, whether it is with seal_ring, the boudary of the chip).

1.  In the layout window, go to Calibre → Run DRC.

2.  If you have already created the runset, skip to Step 9. Otherwise, close the "Load Runset File" form.

3.  In the interactive window, select the "Rules" tab and click ... to choose drc file. Then, select "Load".

4.  Make a new directory(or use an existed one but it will be overwrited) for the drc information to be stored.

5.  **Change your "DRC Run Directory" to this location** so you don't flood your project space with drc files.

6.  Let's save this runset. Go to File → Save Runset.

7.  With the "File Path" empty, press OK.

8.  In the next window, give it a name (e.g. xxx.runset). Select OK.

9.  If you came here from step 2, load the runset we created. Otherwise, continue next step.

10.  Click on the "Inputs" tab and verify that the file information is correct.

11.  In "Run Control" tab, it is recommended to **choose multi thread** rather than single thread to accelerate checking.

12.  Now, select "Run DRC".

13.  Once it is complete (which may take a while, even on campus), a summary file and the Results Viewing Environment (RVE) window will appear.

It is important to note that even though there are a lot of errors, often one fix will solve many other problems.

## 可以waive的DRC violation

- All "CSR.*" (Corner Stress Relief) errors can be ignored if it's not the entire chip. Metals and vias are not allowed in chip corners, but we are not creating the entire chip.
- Any "*.EN" (Enclosure) errors regarding chip edge can be waived; all others should be resolved according to the rule described in the explanation window.
- All "*.S" (Spacing) errors should be resolved. Move metals or vias to meet the minimum spacing requirements. 
- All "*.W" (Width) errors should be resolved. Resize any flagged objects to meet the maximum/minimum width rules.
- All "*.A" (Area) errors should be resolved. Resize the metal/via as described in the check. 
- All "G.*" (Grid) errors should be resolved. The grid is the minimum resolution that can be manufactured. If you used paths with variable ends and specified a length less than the grid resolution (for some unknown reason), you will receive an error. All other path ends are fixed to the grid. Additionally, do not change the grid resolution from 0.005 in the layout display options ("e").
- All ".DN " (Density) errors should be resolved (with the exception of the aforementioned case). Certain metal and oxide densities are required for fabrication. TSMC recommends their autofill script, but I will not cover this as it is better suited for those with sufficient layout experience.

