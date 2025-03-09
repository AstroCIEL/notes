# ip core测试

![ipcore](image-1.png)

## 0. 信号初始化，选择激活ip core

|信号|取值|说明|
|---|---|---|
|IP_SEL|0|选择ip core|
|RSTN|1-0-1|复位|
|others|0|其他信号初始值|

### 0.1 配置DCO扫描链

写入DCO_config扫描链

```verilog
assign {cc_sel,fc_sel,freq_sel,dco_en,dco_clk_sel}=dco_config[16-1 -:16];
```

|信号|取值|说明|
|---|---|---|
|RSTN|1-0-1|复位|
|TCK|ticking|扫描链时钟|
|DCO_SC_EN|1|允许扫描链开始移位扫描|
|DCO_TDI| |串行输入从低位到高位的dco_config数据|

### 0.2 配置DCO

- 选择外灌时钟

|信号|取值|说明|
|---|---|---|
|CLK|ticking|外灌时钟|
|CLK_SEL|1|选择外灌时钟作为片上clk|
|DCO_SRTN|1-0-1|DCO置位|

- 选择DCO时钟

|信号|取值|说明|
|---|---|---|
|CLK_SEL|0|选择DCO ring时钟作为片上clk|
|CC_SEL| |0 for max, 111111 for min|
|FC_SEL| |0 for max, 111111 for min|
|DCO_SRTN|1-0-1|DCO置位|

> 此后片上时钟将一直保持翻转

## 1. 扫描链写入顶层各种配置信号

### 1.1 扫描链写入cim_configs数据

```verilog
assign {cim_mask_en,sign,ema_rsa,ema_rwl,input_rstn_config,mbist_alg_sel} = cim_configs[80-1 -: 75];
```

|信号|取值|说明|
|---|---|---|
|TCK|ticking|扫描链时钟|
|CONFIG_SEL|3'b000|配置为扫描链写cim_configs模式|
|CONFIG_SC_EN|1|允许扫描链开始移位扫描|
|CONFIG_TDI| |串行输入从低位到高位的cim_configs数据|

### 1.2 扫描链写入instruction数据

```verilog
always @(posedge clk or negedge rstn) begin
  if (rstn == 1'b0) begin
      {cen,ren_global,ren_local,wen,cen_loop,
      ren_global_loop,ren_local_loop,wen_loop,
      cnt_cen_0,cnt_ren_global_0,cnt_ren_local_0,cnt_wen_0} <= 'd0;
  end
  else begin
    if (instruction_valid)
      {cen,ren_global,ren_local,wen,cen_loop,
      ren_global_loop,ren_local_loop,wen_loop,
      cnt_cen_0,cnt_ren_global_0,cnt_ren_local_0,cnt_wen_0} <= instruction;
  end
end
```

|信号|取值|说明|
|---|---|---|
|TCK|ticking|扫描链时钟|
|CONFIG_SEL|3'b001|配置为扫描链写instruction模式|
|CONFIG_SC_EN|1|允许扫描链开始移位扫描|
|CONFIG_TDI| |串行输入从低位到高位的instruction数据|
|INSTRUCTION_VALID|0-1-0|写完后需要拉高一次使得instruction锁存|

### 1.3 扫描链写入config_addr数据

```verilog
assign addr_debug = config_addr[16-1 -: 10];
```

|信号|取值|说明|
|---|---|---|
|TCK|ticking|扫描链时钟|
|CONFIG_SEL|3'b010|配置为扫描链写config_addr模式|
|CONFIG_SC_EN|1|允许扫描链开始移位扫描|
|CONFIG_TDI| |串行输入从低位到高位的config_addr数据|

### 1.4 扫描链写入config_exp数据

```verilog
assign ie_exp = config_exp[16-1 -: 9];
```

|信号|取值|说明|
|---|---|---|
|TCK|ticking|扫描链时钟|
|CONFIG_SEL|3'b101|配置为扫描链写config_exp模式|
|CONFIG_SC_EN|1|允许扫描链开始移位扫描|
|CONFIG_TDI| |串行输入从低位到高位的config_exp数据|

## 2. mbist test模式测试sram阵列读写功能正确

|信号|取值|说明|
|---|---|---|
|MBIST_RESET_L|1-0-1|复位|
|MBIST_TEST_H|1|进入mbist test模式|

此时进行mbist test模式，直到mbist_done升高。期间可以观察mbist_fail是否出现

## 3. 扫描链写入weight以及input数据

### 3.1 扫描链写入input数据

|信号|取值|说明|
|---|---|---|
|TCK|ticking|扫描链时钟|
|CONFIG_SEL|3'b110|配置为扫描链写input模式|
|CONFIG_SC_EN|1|允许扫描链开始移位扫描|
|CONFIG_TDI| |串行输入从低位到高位的input数据|
|EN_LOAD|0-1-0|写完后需要拉高一次使得so加载到q|

### 3.2 扫描链写入weight数据

|信号|取值|说明|
|---|---|---|
|TCK|ticking|扫描链时钟|
|CONFIG_SEL|3'b100|配置为扫描链写weight模式|
|CONFIG_SC_EN|1|允许扫描链开始移位扫描|
|CONFIG_TDI| |串行输入从低位到高位的weight数据|
|EN_LOAD|0-1-0|写完后需要拉高一次使得so加载到q|

### 3.3 将扫描链内的weight数据载入dcim ip

