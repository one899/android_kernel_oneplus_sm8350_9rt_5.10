# OnePlus 9RT (martini / MT2110 / SM8350-lahaina) 硬件参数 —— 源码提取

> 任务: task-5 (hw-source)　产出: 本文档
> 方法: GitHub API + raw 拉取本地仓库源码; 关键结论用实机 (adb su) 交叉验证, 实机证据单独标注。
> **未获取** 的项一律明写, 不做推测填充。

---

## 0. 来源与版本指纹 (provenance)

| 记号 | 仓库 | 分支 | 分支 HEAD SHA | 提交日期 |
|---|---|---|---|---|
| **K** | `one899/android_kernel_oneplus_sm8350_9rt_295bpf` | `oneplus/sm8350_b_16.0.0_oneplus9rt` | `c5bcf1a50a107fa0e706d83056f48769ca675d70` | 2026-08-21 |
| **D** | `one899/android_kernel_modules_and_devicetree_oneplus_sm8350` | `oneplus/sm8350_b_16.0.0_oneplus9rt` | `7000c72054de42af5ebaef8b9c64efdb1026478a` | 2026-05-09 |

来源 0 (仓库入口, 决定了 DTS 到底在哪):
- 参数: 9RT 的 DTB 目录 = `vendor/oplus/martini`
  来源: **K** `build.config.martini` 第 3-8 行原文:
  ```
  # OnePlus 9 RT specific config
  KERNEL_DIR=kernel/msm-5.4
  export BRAND_SHOW_FLAG=oneplus
  DTB_DIR=vendor/oplus/martini
  ```
- 参数: K 仓库里 `arch/arm64/boot/dts/vendor` 是**符号链接**, 真身在 vendor 仓库
  来源: K `arch/arm64/boot/dts/vendor` (type=symlink, target=`../../../../../../vendor/qcom/proprietary/devicetree`)
  → **martini 的 DTS 全部在仓库 D 里**, 不在内核仓库。
- 参数: 设备项目号 (dtsi_no) = 20820
  来源: **D** `vendor/qcom/proprietary/devicetree/oplus/martini/martini_base_common.dtsi` 第 65 行 `oplus,dtsi_no = <20820>;`
  来源: **D** `.../oplus/martini/martini-20820-lahaina-v2.1-overlay.dts` 第 13 行 `oplus,dtsi_no = <20820>;`

**实机交叉验证 (adb, 序列号 4647b81e, su=KernelSU root)**:
- `getprop ro.product.model` = `MT2110`; `getprop ro.boot.project_name` = `20820` → 确认 20820 = 9RT = martini
- `wm size` = `Physical size: 1080x2400`
- `/proc/cmdline`(root) 原文含:
  `msm_drm.dsi_display0=qcom,mdss_dsi_samsung_ams662zs01_dvt_dsc_cmd:`
  `oplus_bsp_tp_custom.dsi_display0=qcom,mdss_dsi_samsung_ams662zs01_dvt_dsc_cmd:synaptics-s3908`
  (本地留档: `_meta/research/martini-cmdline.txt`)

本地源码留档目录: `D:\295\_meta\research\martini-dts\` (22 个 martini DTS + 相关 panel/驱动/defconfig);
仓库文件清单: `D:\295\_meta\research\martini-devicetree-tree.txt` (仓库 D 全树, 16174 条)。

---

## 1. 屏幕 (panel)

### 1.1 实际点亮的 panel (决定性证据)

- 参数: **面板 = Samsung AMS662ZS01, 1080x2400, DSC, CMD 模式, 6.62 英寸**, 实机为 DVT 版本
  来源: 实机 `/proc/cmdline` (su): `msm_drm.dsi_display0=qcom,mdss_dsi_samsung_ams662zs01_dvt_dsc_cmd`
  来源: **D** `vendor/qcom/proprietary/display-devicetree/display/dsi-panel-samsung_ams662zs01_dvt_dsc_cmd.dtsi`
  - 第 2 行 `dsi_samsung_ams662zs01_dvt_dsc_cmd: qcom,mdss_dsi_samsung_ams662zs01_dvt_dsc_cmd {`
  - 第 3 行 `qcom,mdss-dsi-panel-name = "samsung ams662zs01 fhd cmd mode dsc dsi panel";`
  - 第 4 行 `oplus,mdss-dsi-vendor-name = "AMS662ZS01";`
  - 第 5 行 `oplus,mdss-dsi-manufacture = "samsung1024";`
  - 第 837-841 行 (该 panel 被标为 display-active, 即被点亮的那一支):
    ```
    &soc {
        dsi_samsung_ams662zs01_dvt_dsc_cmd {
            qcom,dsi-display-active;
        };
    };
    ```
- 参数: 同型号还有**非 DVT** 一支, 源码里同样标了 display-active (两支都在树里, 实机走 DVT)
  来源: **D** `.../display/dsi-panel-samsung_ams662zs01_dsc_cmd.dtsi` 第 3-4 行 (panel-name/vendor-name 同上) 和 第 700-704 行 `&soc { dsi_samsung_ams662zs01_dsc_cmd { qcom,dsi-display-active; }; };`
  说明: 哪一支生效由 bootloader 依 cmdline 选择; **实机命令行为 ..._dvt_dsc_cmd**。

### 1.2 panel 基本模式/接口参数

来源统一: **D** `display-devicetree/display/dsi-panel-samsung_ams662zs01_dvt_dsc_cmd.dtsi`

| 参数 | 值 | 行号 |
|---|---|---|
| panel 类型 | `dsi_cmd_mode` (命令模式, 单 DSI) | 7 |
| dsi-ctrl-num / dsi-phy-num | `0` / `0` (走 DSI0 + PHY0) | 14-15 |
| bpp | `24` | 10 |
| traffic-mode | `burst_mode` | 16 |
| lane-map | `lane_map_0123`, lane0-3 全 state | 17, 20-23 |
| DMA/MDP trigger | `trigger_sw` / `none` | 24, 26 |
| 复位时序 | `<1 10>, <0 10>, <1 10>` (ms) | 27 |
| TE | te-pin-select=1, te-dcs-command=1, te-check-enable, te-using-te-pin | 28-29, 33-34 |
| 屏体物理尺寸 | `70 x 153` mm (≈6.62") | 30-31 |
| wr-mem start/continue | `0x2c` / `0x3c` | 36-37 |
| HDR | panel-hdr-enabled, primaries `<15635 16450 34000 16000 13250 34500 7500 3000>` | 40-41 |
| peak/avg brightness(寄存器域) | `5400000` / `2000000` | 42-43 |
| ESD | reg_read 0x0A, 期望值 `0x9F`, 读长 1 | 47-54 |
| 动态模式切换 | `dynamic-resolution-switch-immediate` | 56-57 |
| ULPS | `qcom,ulps-enabled` | 857 |
| 动态时钟 | dsi-dyn-clk-enable, `dsi-dyn-clk-list = <1100000000 1100000000>` (1100 Mbps class) | 858-859 |
| OSC 支持 | oplus,osc-support, mode0=180300, mode1=182300 (kHz) | 860-862 |
| 物理层时序 | `panel-phy-timings = [00 24 0A 0A 1A 19 09 0A 09 02 04 00 1E 0F]` (60Hz 与 120Hz 同值) | 917, 922 |
| display-topology | `<1 1 1>,<2 2 1>`, default-topology-index = 1 | 918-919, 923-924 |

### 1.3 panel 时序 (两条 timing: 60Hz / 120Hz)

来源: 同文件。**只有 2 组 timing, 没有独立 90Hz timing**。

| 参数 | timing0 = 60Hz | timing1 = 120Hz |
|---|---|---|
| framerate | `60` (L61) | `120` (L451) |
| width x height | `1080 x 2400` (L62-63) | `1080 x 2400` (L452-453) |
| h-front-porch | `64` (L65) | `8` (L454) |
| h-back-porch | `48` (L66) | `8` (L455) |
| h-pulse-width | `8` (L67) | `24` (L456) |
| h-sync-skew | `0` (L68) | `0` (L457) |
| v-back-porch | `12` (L69) | `8` (L458) |
| v-front-porch | `8` (L70) | `2` (L459) |
| v-pulse-width | `4` (L71) | `2` (L460) |
| 边框 | h-left/right/v-top/bottom = 0 (L74-77) | — |
| mdp-transfer-time-us | `12000` (L60) | — |

DSC 压缩 (两条 timing 相同):
- 参数: compression-mode = "dsc"; slice-height=30, slice-width=540, slice-per-pkt=2, bit-per-component=8, bit-per-pixel=8, block-prediction-enable
  来源: 第 440-446 行 (timing0) 与 第 822-827 行 (timing1)
- 参数: 双混流切分 `qcom,lm-split = <540 540>`
  来源: 第 439 行

### 1.4 panel 供电 / 引脚 / 亮度

- 参数: 供电方案 = `dsi_panel_oplus_pwr_supply`
  来源: 第 844 行 `qcom,panel-supply-entries = <&dsi_panel_oplus_pwr_supply>;`
  定义来源: **D** `display-devicetree/display/lahaina-sde-display-common.dtsi` 第 126-168 行:
  - 参数: vddio = 1800000 uV (1.8V), enable load 60700
  - 参数: vdd = 3200000 uV (3.2V), enable load 10000
  - 参数: lab / ibb = 4600000~6000000 uV (本机未使用)
  来源: 同文件 第 130-167 行
- 参数: vddi/vddr = 1800000/1850000/1950000
  来源: 第 934-938 行
- 参数: 控制器 = `&mdss_dsi0`
  来源: 第 845 行
- 参数: 背光控制 = bl_ctrl_dcs, dc-backlight-level 520, bl-min 1, bl-normal-max **2047**, bl-max **4095**, brightness-default 4575
  来源: 第 846-853 行
- 参数: TE gpio = tlmm **82**; reset gpio = tlmm **24**; panel-vout gpio = tlmm **25**
  来源: 第 854-856 行
- 参数: Apollo 背光 (DC 调光) 使能, sync brightness level = 540
  来源: 第 929-932 行
- 参数: FOD(屏下指纹)亮度映射表 0..2047 → 0xff..0x24; DC 亮度表 0..520 → 0xff..0x00
  来源: 第 865-887 行 / 第 889-913 行

### 1.5 martini 专属屏幕覆盖

- 参数: martini 关闭 DSI PLL SSC (展频)
  来源: **D** `vendor/qcom/proprietary/devicetree/oplus/martini/display/lahaina-sde-display.dtsi` 第 1-7 行原文:
  ```
  &mdss_dsi_phy0 { oplus,dsi-pll-ssc-disalbed; };
  &mdss_dsi_phy1 { oplus,dsi-pll-ssc-disalbed; };
  ```
- 参数: 该文件由 `martini-20820.dtsi` 第 1 行引入
  来源: **D** `.../oplus/martini/martini-20820.dtsi` 第 1 行 `#include "display/lahaina-sde-display.dtsi"`

### 1.6 树里"编译进来但没有被点亮"的其它 panel (供对照, 不要误当真机面板)

来源: **D** `display-devicetree/display/lahaina-sde-display-common.dtsi` 第 35-53 行的 #include 列表, 其中 OPLUS 的都是 1440p 或别的机型:
`dsi-panel-oplus19065-samsung-1440-3168-dsc-cmd`, `dsi-panel-oplus20031-samsung-1440-3216-*` (含 evt/dvt/id07/id09),
`dsi-panel-oplus21005_boe_1080_2400_dsc_cmd*` (BOE NT37701, 1080x2400, 60/90/120Hz —— **不是 9RT 的点亮面板**),
`dsi-panel-oplus-20085-*`, `dsi-panel-samsung_amb655x_dsc` 等。
> 注意: martini-common.dtsi 里触摸节点的 panel 列表里出现的 `dsi_oplus20031_samsung_1440_3216_*` 是**历史遗留引用**, 与实机 1080x2400 不符; 实际生效面板以 1.1 为准。

---

## 2. 充电 / 电池

### 2.1 充电架构 (IC 层)

- 参数: 充电框架 = **PMIC 充电 (平台 PMIC)**, 由 **ADSP 固件**经 pmic_glink 控制; 无外挂 charge-pump
  来源: **D** `vendor/qcom/proprietary/devicetree/oplus/martini/martini-common.dtsi` 第 695 行
  `oplus,chg_ops = "plat-pmic";`
  来源: **D** `vendor/qcom/proprietary/devicetree/qcom/lahaina.dtsi` 第 5583-5591 行:
  ```
  qcom,pmic_glink {
      compatible = "qcom,pmic-glink";
      qcom,pmic-glink-channel = "PMIC_RTR_ADSP_APPS";
      qcom,subsys-name = "adsp";
      qcom,protection-domain = "tms/servreg", "msm/adsp/charger_pd";
      battery_charger: qcom,battery_charger { compatible = "qcom,battery-charger"; };
  };
  ```
- 参数: VOOCPHY = **ADSP VOOCPHY**, `qcom,voocphy_support = <1>`
  来源: **D** `.../oplus/martini/martini-common.dtsi` 第 682-690 行:
  ```
  &soc {
      oplus,adsp-voocphy { compatible = "oplus,adsp-voocphy"; };
      midas_pdev { compatible = "oplus,midas-pdev"; };
  };
  ```
  来源: 同文件第 697 行 `qcom,voocphy_support = <1>;`
- 参数: 编译开关 = `CONFIG_OPLUS_CHG_OP9RT_PMIC_VOOCPHY=y` (= 9RT 的 **PMIC VOOCPHY** 变体)
  来源: **K** `arch/arm64/configs/9rt_defconfig` **第 2978 行**; 同文件第 2964 行 `CONFIG_OPLUS_CHG=y`
  来源(语义): `LineageOS/android_kernel_oneplus_sm8350 @ lineage-23.2` `drivers/power/Makefile` 原文:
  ```
  ifeq ($(strip $(CONFIG_OPLUS_CHG_OP9RT_PMIC_VOOCPHY)), y)
      obj-$(CONFIG_OPLUS_SM8350_CHARGER) += oplus/
  else
      obj-$(CONFIG_OPLUS_SM8350_CHARGER) += oplus_chg/
  endif
  ```
- 参数: VOOC 通信走 **SE3 双线 UART** (`chg_qupv3_se3_2uart_active/sleep`)
  来源: **D** `.../oplus/martini/martini-common.dtsi` 第 905, 911-912, 1131-1150 行
- 参数: 放电/复位 gpio = `pm8350c_gpios 7`
  来源: 同文件 第 902 行 `qcom,dischg-gpio = <&pm8350c_gpios 7 0x00>;`

**实机交叉验证**: 逐一检查外挂 charge-pump 驱动是否绑定到 i2c 设备:
`ls /sys/bus/i2c/drivers/{hl7138-charger,hl7138-charger-slave,sc8517-charger,sc8547-charger,sc8547-charger-slave,mp2650-charger,nu1619-charger}`
→ **全部为空 (无任何绑定设备)**, 印证"无外挂 charge-pump、纯 PMIC + ADSP VOOC"。
(注: 这些 CP 驱动在 `vendor/oplus/kernel/charger/voocphy/Makefile` 第 1-8 行被一并编入, 属于通用库, 本机未用。)

### 2.2 电池

| 参数 | 值 | 来源 (martini-common.dtsi) |
|---|---|---|
| 典型容量 | `4500` mAh | 第 735 行 `qcom,batt_capacity_mah = <4500>;` |
| 额定容量 | `4400` mAh (源码中被注释掉) | 第 736 行 |
| vbatt_num | `2` | 第 818 行 |
| 满充/复充阈值 | vfloat 4435 mV, recharge 100 mV, iterm 130 mA | 第 733-734, 804 行 |
| 温度档 | cold 20(=-2C) / little_cold 0 / cool 50 / normal 160 / warm 450 / hot 530 decidegc | 第 740, 744, 754, 763, 769, 775, 782 行 |
| 充电超时 | `36000` s | 第 800 行 |
| HV/RECV/LV 阈值 | 9900 / 9500 / 3400 mV | 第 801-803 行 |
| FFC | ffc_support + dual_ffc | 第 819-820 行 |
| 老化 FFC 版本 | `oplus,aging_ffc_version = <1>` | 第 883 行 |

### 2.3 电流 / 协议 (PD/QC/PPS/VOOC)

| 参数 | 值 | 来源 (martini-common.dtsi) |
|---|---|---|
| 普通充电输入上限 | `2000` mA | 第 699 行 |
| PD / QC 输入上限 | `2000` mA (各自) | 第 700-701 行 |
| USB 输入 | `500` mA | 第 703 行 |
| CDP / LED / CAMERA | 1500 / 1200 / 1200 mA | 第 705-706, 713 行 |
| VOOC 常态 / warm / high | 3600 / 3200 / 2200 mA | 第 718, 720, 722 行 |
| VOOC 输入电压/电流上限 | `10000` mV (10V) / `6500` mA (6.5A) = **65W** | 第 914-915 行 |
| vooc_project | `3` | 第 817 行 |
| VOOC 30W 策略 | vooc_charge_strategy_30w 5 段 (含 6000 mA→4200 mV, 2000 mA→4480 mV) | 第 916-950 行 |
| **VOOC 65W 策略** | vooc_charge_strategy_65w 3 段, 峰值 `6500 mA @4200 mV`, 到 4454 mV 后 4500→3500→2000→1500 mA | 第 952-983 行 |
| PPS 支持 | `qcom,pps_support_type = <1>` | 第 860 行 |
| PPS 常态功率 | `6000` (源码注释 //6A) | 第 863 行 |
| PPS 多段策略 | pps_charge_strategy 从 `10000 mV @ 3000 mA` 逐段降到 `10000 mV @ 1000 mA` | 第 985-1046 行 |
| PD/QC 强制 9V | vbatt_pdqc_to_5v_thr / vbatt_pdqc_to_9v_thr = `5000` (始终 9V 档) | 第 896-897 行 |
| 温控联动 | chg_ctrl_by_lcd / _vooc / _camera / _calling | 第 727, 890-892 行 |

> 说明: 以上"电流"均为 **DT 中的策略上限/分段值**, 不是实测值。

---

## 3. 触摸 (TP)

### 3.1 IC / 总线 / 中断

| 参数 | 值 | 来源 |
|---|---|---|
| 触摸 IC | **Synaptics S3908** (TCM, on-cell) | `martini-common.dtsi` 第 2026-2029 行: 节点 `mtp_20031:synaptics20031@4B`, `compatible = "synaptics-s3908"`, `chip-name = "S3908"` |
| I2C 从地址 | `0x4B` | 同文件 第 2028 行 `reg = <0x4B>;` |
| I2C 总线 | `qupv3_se4_i2c` = **`i2c@990000`** | `martini-common.dtsi` 第 1922 行 `&qupv3_se4_i2c`; 总线定义 `qcom/lahaina-qupv3.dtsi` 第 339-341 行 `qupv3_se4_i2c: i2c@990000 { reg = <0x990000 0x4000>; }` |
| 中断 | `tlmm 23`, flags `0x2008` | `martini-common.dtsi` 第 2032-2033 行, 2043 行 |
| 复位脚 | `tlmm 22` | 同文件 第 2044 行 |
| pinctrl | `ts_int_active` + `ts_reset_active` | 同文件 第 2045-2046 行 |
| 供电 | vdd_2v8 = L3C, vcc_1v8 = L8C, vdd_2v8_volt = 3008000 (3.008V) | 同文件 第 2038-2040 行 |
| 触摸分辨率/tx-rx | tx-rx-num = <16 35>; panel-coords = <8640 19200>; display-coords = <1080 2400>; touchmajor-limit = <256 256> | 同文件 第 2049, 2051-2053 行 |
| IC 类型 | tp_ic_type = <2>, panel_type = <8> (TP-SAMSUNG) | 同文件 第 2083-2084 行 |
| 固件名 | `firmware_name = "SS"` | 同文件 第 2086 行 |
| 支持项目 | `platform_support_project = <20820 20821>` (= 本机 20820) | 同文件 第 2087-2088 行 |
| 报点率 | report_rate_default = 60, report_rate_game_value = 3, fps_report_rate = <60 2 90 3 120 3> | 同文件 第 2160-2166 行 |
| 屏下指纹联动 | fingerprint_underscreen_support, screenoff_fingerprint_info_support | 同文件 第 2129, 2132 行 |
| TP 固件 DTS | `mtp20820-s3908-firmware.dtsi` (1,049,200 B) | **D** `.../oplus/martini/mtp20820-s3908-firmware.dtsi` |

驱动绑定 (源码侧):
- 参数: 驱动文件 = `Synaptics/Syna_tcm_oncell/synaptics_tcm_oncell.c`, 匹配串 `"synaptics-s3908"`
  来源: **D** `vendor/oplus/kernel/touchpanel/oplus_touchscreen_v2/Synaptics/Syna_tcm_oncell/synaptics_tcm_oncell.h` 第 18-22 行 `#define TPD_DEVICE "synaptics-s3908"`;
  同目录 `synaptics_tcm_oncell.c` 第 7701-7717 行 (match table + `.name = TPD_DEVICE`)
- 参数: 同树另有 `Syna_tcm_S3910` 驱动 (TPD_DEVICE = "synaptics-s3910", 头文件第 18-22 行), 与 9RT 的 DTS compatible **不匹配**
- 参数: defconfig 同时打开
  来源: **K** `arch/arm64/configs/9rt_defconfig` 第 2400 行 `CONFIG_TOUCHPANEL_SYNAPTICS=y`, 第 2409 行 `CONFIG_TOUCHPANEL_SYNAPTICS_TCM_ONCELL=y`, 第 2410 行 `CONFIG_TOUCHPANEL_SYNAPTICS_TCM_S3910=y`

**实机交叉验证 (决定性)**:
- `/sys/bus/i2c/devices/5-004b/driver` → `/sys/bus/i2c/drivers/synaptics-s3908` (与上面 compatible 完全一致)
- `.../5-004b/of_node` → `.../firmware/devicetree/base/soc/i2c@990000/synaptics20031@4B` (与 3.1 的 i2c@990000 一致)
- 说明: DTS 里 `qupv3_se4_i2c`(i2c@990000) 在 Linux 侧枚举为 **i2c-5**

### 3.2 同总线上被禁用的其它 TP (冗余配置)

- 参数: `st_fts@49` = disabled; 通用 `focaltech@38` = disabled
  来源: `martini-common.dtsi` 第 1926-1931 行
- 参数: `mtp_19065:s6sy791_19065@48` (S6SY791, 1440x3168, platform_support_project = 19065) 与 20820 不匹配 → 本机不生效
  来源: `martini-common.dtsi` 第 1933-2024 行 (关键行 1966)

### 3.3 指纹 (屏下光学, 顺带)

- 参数: Goodix 光学 `G_OPTICAL_G3S`, IRQ = tlmm 38, reset = tlmm 28
  来源: `martini-common.dtsi` 第 1869-1888 行 (chip-name 第 1873 行; gpio_irq 第 1884 行; gpio_reset 第 1885 行)
- 参数: defconfig `CONFIG_OPLUS_FINGERPRINT_GOODIX=y` / `..._GOODIX_OPTICAL=y` / `CONFIG_UFF_FINGERPRINT=m`
  来源: **K** `arch/arm64/configs/9rt_defconfig` 第 2477-2478, 2489 行

---

## 4. 摄像头

来源统一: **D** `vendor/qcom/proprietary/devicetree/oplus/martini/lahaina-camera-sensor-20820.dtsi` (710 行)

### 4.1 槽位 / CSI / CCI / GPIO / 供电

| 槽位 | csiphy-sd-index | cci-master | MCLK gpio | reset gpio | vana/vdig gpio | 其它 | 行号 |
|---|---|---|---|---|---|---|---|
| cam-sensor0 (主摄, 带 AF+OIS+主闪光) | **1** | 0 | tlmm 100 (CAMIF_MCLK0) | tlmm 16 | tlmm 125 (avdd) | eeprom@0, actuator@0, ois bu63169, flash0 | 109-156 |
| cam-sensor1 (副摄, 带辅助闪光) | **2** | 1 | tlmm 101 (MCLK1) | tlmm 106 | avdd 97 / dvdd 93 | eeprom@1, flash1 | 199-245 |
| cam-sensor2 (**前摄**) | **5** | 1 | tlmm 103 (MCLK3) | tlmm 115 | — | eeprom@2 (前) | 282-318 |
| cam-sensor3 (微距, led_flash_micro) | **0** | 0 | tlmm 102 (MCLK2) | tlmm 167 | — | eeprom@3 | 352-389 |

供电轨 (rgltr-min/max-voltage, 单位 uV):
- 主摄: cam_vio=pm8008i_l5 (1800000), cam_vana=S1C (1800000~1952000), cam_vdig=pm8008i_l1 (1100000~1200000), cam_v_custom1=pm8008i_l4 (2800000), cam_vaf=L6I (2696000~2904000)
  来源: 第 120-131 行
- 副摄: cam_vio=L5I, cam_vana=pm8350c_bob, cam_vdig=S12B
  来源: 第 208-212 行
- 前摄: cam_vio=pm8008i_l5, cam_vana=pm8008i_l3, cam_vdig=pm8008i_l2 (1048000~1360000)
  来源: 第 290-298 行
- 微距: cam_vio=pm8008i_l5, cam_vana=pm8008i_l7
  来源: 第 361-364 行

其它:
- 参数: OIS 芯片 = **bu63169**, cam_vdig = L9C, ois_gyro,position = <3>, ois_module,vendor=<1>, ois_actuator,vednor=<2>
  来源: 第 51-67 行 (第 63 行 `ois,name="bu63169";`)
- 参数: 闪光灯 = PMIC 闪光 pm8350c_flash0/flash1 (三路: rear / rear_aux / micro)
  来源: 第 3-29 行 (第 9 行 `qcom,flash-name = "pmic";`)
- 参数: 主摄 EEPROM 读取方式 is-read-eeprom = <2>, reg-setting-ver = <8>
  来源: 第 154-155 行
- 参数: `&cam_cci0` / `&cam_cci1` 两组 CCI; cci0 挂 sensor0/1, cci1 挂 sensor2/3
  来源: 第 36, 246 行

### 4.2 **未获取** 的摄像头参数 (源码里确实没有)

- **sensor 型号 (IMX766 等)**: 未获取 —— DTS 里没有; 该文件只有 `compatible = "qcom,cam-sensor"` 通用节点, 不含 sensor 型号字段。
  证据: 全文件 grep name/compatible 无任何 sensor 型号 (本地 `martini-dts/oplus__martini__lahaina-camera-sensor-20820.dtsi`);
  且 **K** `9rt_defconfig` 里没有任何 OPLUS/QCOM 摄像头 sensor 的 CONFIG (sensor 驱动不在本两个仓库中)。
- **sensor I2C 从地址**: 未获取 —— 节点**没有 `reg` 属性**。
  证据(源码): 上述 dtsi 中 cam-sensor0..3 均无 `reg`;
  证据(实机 DTB, root): `su -c ls /sys/firmware/devicetree/base/soc/qcom,cci0/martini,cam-sensor0` 的属性列表**不含 reg**
  (对照: `qcom,eeprom@2/reg` 可读出 `0 0 0 2`, 说明读取方法本身没问题)。→ 地址由 OPLUS 相机 HAL/用户态 sensor 配置提供, 不在内核 DTS。
- 实机 DTB 节点名佐证: `martini,cam-sensor0..3` + `qcom,cci0`(挂 0/1) + `qcom,cci1`(挂 2/3) + `martini,camera-flash0/1/2`, 与源码一致。

---

## 5. 汇总速查 (一句话版)

| 域 | 结论 (均来自源码, 除标注外) |
|---|---|
| 屏幕 | Samsung **AMS662ZS01**(DVT), 6.62", **1080x2400**, DSI **CMD** 模式 + DSC(8bpc/8bpp, 540x30 slice), 4 lane, 只有 **60Hz / 120Hz** 两组 timing, 复位 gpio24 / TE gpio82, vddio 1.8V + vdd 3.2V, 背光 DCS 0-4095(常态 0-2047); martini 关 PLL SSC。**实机 cmdline 确认** |
| 充电 | 架构 = **PMIC 充电 + ADSP VOOCPHY** (chg_ops="plat-pmic", oplus,adsp-voocphy, defconfig `CONFIG_OPLUS_CHG_OP9RT_PMIC_VOOCPHY=y`), VOOC 走 SE3 双线 UART, **无外挂 charge-pump** (实机 i2c 驱动全部未绑定验证); 电池 **4500mAh**; VOOC 上限 **10V/6.5A(65W)**, 65W/30W 分段策略; PD/QC 默认 2A; **PPS 支持**(10V 多段, 峰值段 3A, DT 注释 6A) |
| 触摸 | **Synaptics S3908** (TCM on-cell), I2C `i2c@990000`(qupv3_se4_i2c, 实机 i2c-5) 地址 **0x4B**, IRQ tlmm **23**, reset tlmm **22**, 16x35, 1080x2400, vdd 3.008V/1.8V; 驱动 `synaptics_tcm_oncell.c`(compatible `synaptics-s3908`); 同总线 S6SY791(19065)/FT@38/ST@49 均 disabled。**实机 driver/节点路径确认** |
| 摄像头 | 4 槽 (0=主摄 CSI1/CCI0+MCLK0+AF/OIS bu63169, 1=副摄 CSI2/CCI1+MCLK1, 2=前摄 CSI5/CCI1+MCLK3, 3=微距 CSI0/CCI0+MCLK2), 供电轨与 gpio 见 4.1; **sensor 型号与 I2C 地址 = 未获取 (源码中不存在)** |

---

## 6. 复现命令 (供 Lead / 复核者)

```powershell
# 1) 列目录 (GitHub API 必须走代理, token 在 D:\295\_meta\gh_token.txt)
$tok=(Get-Content D:\295\_meta\gh_token.txt -Raw).Trim()
$h=@{Authorization="token $tok";"User-Agent"="dsh-agent";Accept="application/vnd.github+json"}
$br="oneplus/sm8350_b_16.0.0_oneplus9rt"
$repo="one899/android_kernel_modules_and_devicetree_oneplus_sm8350"
$u="https://api.github.com/repos/$repo/git/trees/"+[uri]::EscapeDataString($br)+"?recursive=1"
Invoke-RestMethod -Proxy http://127.0.0.1:18888 -Headers $h -Uri $u

# 2) 取原文 (raw)
$raw="https://raw.githubusercontent.com/$repo/$br/vendor/qcom/proprietary/devicetree/oplus/martini/martini-common.dtsi"
Invoke-WebRequest -Proxy http://127.0.0.1:18888 -Headers @{Authorization="token $tok"} -OutFile out.dtsi -Uri $raw

# 3) 实机交叉验证 (root; 序列号 4647b81e)
& 'C:\WINDOWS\system32\adb.exe' -s 4647b81e shell 'su -c "cat /proc/cmdline"'
& 'C:\WINDOWS\system32\adb.exe' -s 4647b81e shell "su -c 'ls /sys/bus/i2c/devices/5-004b'"
```

## 7. 本地留档清单 (`D:\295\_meta\research\`)

- `martini-hw-source.md` ← 本文
- `martini-cmdline.txt` (实机 cmdline, root)
- `martini-devicetree-tree.txt` (仓库 D 全树清单)
- `martini-dts\` : martini 全量 DTS 22 个 + qcom/lahaina.dtsi, lahaina-qupv3.dtsi, lahaina-mtp*.dtsi,
  panel `dsi-panel-samsung_ams662zs01{,_dvt}_dsc_cmd.dtsi`, qcom/pm8350b.dtsi, 触摸驱动
  `synaptics_tcm_{S3910,oncell}.{c,h}`, `kernel_9rt_defconfig`, `kernel_lahaina_QGKI.config`, 相机 dtsi 等。

---

## 8. 来源 2 / 来源 3 交叉对照 (公开仓库)

### 来源 2: LineageOS martini 设备树

- 参数: 存在独立 martini 设备仓库与内核仓库 (公开, 可作对照)
  来源: `LineageOS/android_device_oneplus_martini` (default branch `lineage-23.2`, 75 个文件),
  `LineageOS/android_kernel_oneplus_sm8350` (`lineage-23.2`)
- 参数: **充电 = 9RT 专属 PMIC VOOCPHY 配置** (与我们 5.4 树同一条 defconfig)
  来源: `LineageOS/android_device_oneplus_martini` `BoardConfig.mk` **第 19 行**原文:
  `TARGET_KERNEL_ADDITIONAL_FLAGS := CONFIG_OPLUS_CHG_OP9RT_PMIC_VOOCPHY=y`
  → **独立佐证第 2.1 节的充电架构结论** (PMIC VOOCPHY, 非外挂 CP)
- 参数: 这些设备树的对外投影 = device/oneplus/martini (AOSP 侧路径)
  来源: 同文件 第 10 行 `DEVICE_PATH := device/oneplus/martini`
- 说明: 该仓库 75 个文件里**没有**独立的 panel/touch/charger DTS 或 config (只有 BoardConfig.mk / device.mk /
  overlay 资源); 硬件描述仍来自内核侧 vendor devicetree (与第 1-4 节同源), 因此 LineageOS 侧不提供额外硬件参数。

### 来源 3: 一加 9 / 9 Pro (lemonade / lemonadev, 同 SM8350)

- 参数: 同仓库里 9/9 Pro 的 DTS 目录为 `vendor/qcom/proprietary/devicetree/oplus/lemonadev`
  (`lemonade_common.dtsi` 15,765 B, `lemonade_charge.dtsi` 28,785 B, `lemonade-19825.dtsi` 等)
  来源: **D** `martini-devicetree-tree.txt` 中 oplus/lemonadev 条目
- 对照结论: **9RT(martini) 没有独立的 *_charge.dtsi**, 充电配置被合进 `martini-common.dtsi` (67,541 B);
  且 martini 目录里 `lahaina_base_oplus_common.dtsi` 第 3 行仍残留 9 Pro 的 `oplus,dtsi_no = <19825 19815>;`
  → 说明 martini 树是从 lemonadev 剪裁而来; 这解释了 1.6 节与 3.2 节里那些 1440p panel / S6SY791(19065) 的**遗留引用**,
  评估硬件时必须按第 0 节的实机 cmdline 与 dtsi_no=20820 过滤。

---

## 9. 与设备侧 (另一 teammate) 的对照点 / 潜在冲突

我方在本文中已自行用实机 (root) 做了最小交叉验证, 供 reconcile:
1. **面板**: 源码有两支 ams662zs01 (DVT / 非 DVT) 都标了 qcom,dsi-display-active; 仅靠 DTS **无法判断实机是哪支** ——
   需以实机 cmdline ..._dvt_dsc_cmd (或设备侧读到的 panel name) 为准。若设备侧报的是非 DVT 名称, 需核对读取来源。
2. **触摸**: DTS compatible 为 synaptics-s3908, 而 defconfig 同时编入 ..._TCM_S3910 (compatible synaptics-s3910);
   实机绑定的是 synaptics-s3908。若设备侧看到 synaptics-s3910 字样, 才是冲突。
3. **触摸总线**: DTS 只给到 qupv3_se4_i2c / i2c@990000; i2c-5 的编号是实机 Linux 枚举结果。
4. **充电**: 源码层只能说 "PMIC + ADSP VOOCPHY, 无外挂 CP" (实机 i2c CP 驱动零绑定佐证);
   **具体 PMIC 型号** 与 "65W 是否由 PMIC 直出" 属硬件/固件层, 源码未明写 —— 若设备侧有实测 (充电动画/协议抓包/PMIC 型号), 以设备侧为准并回填本节。
5. **摄像头**: sensor 型号与 I2C 地址在**内核源码中不存在** (见 4.2); 若设备侧能从 HAL/厂商分区读出, 请直接补进本文 4.2。
