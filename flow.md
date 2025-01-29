# Digital IC Design Flow

## 1. RTL(一般是行为级)

## 2. （综合前）仿真（behavioral simulation）

### 写testbench

通过自己编写testbench来先对设计做一个了解，看一看每个信号的时序等信息。

例如top_cim的testbench在

```text
/work/home/rhxu/ip_periph_rtl/front_end_sim/verify/tb_top_cim.v
```

要在testbench里加上

```verilog
initial begin
  $fsdbDumpfile("tb_plot.fsdb");//this file is generated in the directory of makefile
  $fsdbDumpvars("+all");
end
```

### 仿真（VCS+VERDI）

编写makefile

```Makefile
VCS = vcs +v2k -override_timescale=100ps/1ps -full64 -fsdb -debug_all -sverilog +vcs+flush+all +lint=TFIPC-L -notice +notimingcheck -o simv -l compile.log

all: vcs sim verdi

vcs:
    $(VCS) -f list.F

sim:
    ./simv -l simulation.log

verdi:
    verdi -top tb_top_cim -sv -f list.F -ssf tb_plot.fsdb +define+IS_SIM &

clean:
    rm -rf ./csrc *.daidir ./csrc \
    *.log *.vpd *.vdb simv* *.key stack* vc_hdrs.h novas.rc \
    +race.out* novas.conf novas_dump.log verdi* *fsdb* apb2apb_asyno \
    ddr_addr.txt ddr_data.txt
```

编写list.f文件

```text
../../rtl/psum_collector.v
../../rtl/top_cim.v
../../rtl/dcim_macro_bm.v
../../rtl/input_collector.v
../../rtl/mini_controller.v
../../rtl/dcim_macro_32_bm.v
../../rtl/weight_collector.v
../../rtl/config_cim.v
../../rtl/cim_weight_collector.v
../../rtl/mbist_a_g.v
../../rtl/mbist_comp.v

../../rtl/dcim_ip_bm.v

../verify/tb_top_cim.v
```

在ncc服务器上命令行，make指令前面要加上一个b，例如

```bash
b make all
```

这样就可以自动打开verdi看波形。verdi中，选择代码文本中的信号，按ctrl+w可以将信号加到波形中查看。

## 3. （综合前）uvm验证

## 4. 综合（synthesis）

## 5. （综合后）验证

## 6. 数字后端