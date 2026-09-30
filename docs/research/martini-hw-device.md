# OnePlus 9RT (martini / MT2110) 硬件参数 —— 设备侧实证

> 采集对象：**已连接实机**，adb 序列号 `4647b81e`
> 采集方式：`adb shell` + `su`（设备已 root，su 走 KernelSU，context `u:r:ksu:s0`）
> 采集日期：设备内时间 2026-09-30
> 原则：**每一项都附可复现命令 + 原始输出片段；设备侧拿不到的一律写「未获取」，不做推测填充。** 推断项单独标注 `【推断】`。

---

## 0. 采集环境与方法

### 0.1 设备在线确认

```
$ & 'C:\WINDOWS\system32\adb.exe' devices -l
List of devices attached
4647b81e               device product:MT2110_CH model:MT2110 device:OP5154L1 transport_id:1
```

### 0.2 root 可用性（关键前提）

```
$ adb -s 4647b81e shell "id"
uid=2000(shell) gid=2000(shell) ... context=u:r:shell:s0

$ adb -s 4647b81e root
adbd cannot run as root in production builds          # <- user build，adb root 不可用

$ adb -s 4647b81e shell "su -c id"
uid=0(root) gid=0(root) groups=0(root) context=u:r:ksu:s0   # <- su 可用（KernelSU）
```

> 注意：`/proc/device-tree/*` 对 shell uid 是 **Permission denied**，但 `ls` 目录可以。
> 本文所有 device-tree 内容均由 `su -c` 读取。

```
$ adb shell "cat /proc/device-tree/model"            # 以 shell 身份
cat: /proc/device-tree/model: Permission denied

$ adb shell "su -c 'cat /proc/device-tree/model'"    # 以 root 身份
Qualcomm Technologies, Inc. Lahaina MTP
```

### 0.3 本文用到的两个辅助脚本（可复现）

**（a）整棵 device-tree 路径枚举** —— 该内核上 `find /proc/device-tree` 不可靠（只返回根节点），改用递归 shell 函数：

```sh
#!/system/bin/sh
OUT=/data/local/tmp/dtpaths.txt
: > $OUT
walk() {
  for e in "$1"/*; do
    [ -e "$e" ] || continue
    echo "$e" >> $OUT
    if [ -d "$e" ]; then walk "$e"; fi
  done
}
walk /proc/device-tree
echo "TOTAL: $(wc -l < $OUT)"
```
结果：`TOTAL: 44843`（44843 个 DT 节点/属性），已 pull 到本地用于检索。

**（b）属性十六进制转储器** —— device-tree 属性是裸二进制，字符串属性以 NUL 结尾，数值属性是 big-endian。
直接 `cat` 会被当字符串打印成乱码，因此统一 `od -A n -v -t x1` 转 hex 后本地解码：

```sh
hex() { od -A n -v -t x1 "$1" | tr -d ' \n'; }
emit() { p="$1"; h=$(hex "$p"); printf '%s\t%s\t%s\n' "$p" "$((${#h}/2))" "$h"; }
walkp() {
  if [ -d "$1" ]; then for e in "$1"/*; do [ -e "$e" ] && walkp "$e"; done
  elif [ -e "$1" ]; then emit "$1"
  else printf '%s\tMISSING\t\n' "$1"; fi
}
while IFS= read -r p; do [ -n "$p" ] && walkp "$p"; done < /data/local/tmp/list.txt
```
（下文出现的「`u32be=N`」即由此得到：4 字节属性按 big-endian 读作 u32。）

---

## 1. 设备身份与平台

| 参数 | 值 |
|---|---|
| 型号 | MT2110（中国版；`product:MT2110_CH`，MT2111 为海外版） |
| 代号 | martini（项目号 **20820**） |
| 设备节点名 | OP5154L1 |
| SoC | SM8350 / lahaina |
| 平台 ID | 415 |
| HW version | 22 |
| RF version | 11 |
| dtbo 索引 | 1 |
| 内核 | `5.4.295-qgki-gaaf8d3fac817-dirty` |
| 编译时间(内核) | Tue Sep 29 21:12:15 UTC 2026 |
| 编译时间(ROM) | Sun Sep 13 00:21:15 CST 2026 |
| ROM 版本 | MT2110_17.0.0.100(SP09CN01) |

**证据**

```
$ adb shell "su -c 'cat /proc/device-tree/model; echo; cat /proc/device-tree/compatible; echo'"
Qualcomm Technologies, Inc. Lahaina MTP
qcom,lahaina-mtp qcom,lahaina qcom,mtp

$ adb shell "su -c 'od -A n -t x1 /proc/device-tree/oppo,dtsi_no'"
 00 00 51 54                       # <- 0x5154 = 20820 十进制，即 OPLUS 项目号 martini

$ adb shell "su -c 'od -A n -t x1 /proc/device-tree/qcom,msm-id'"
 00 00 01 9f 00 02 00 01  00 00 01 c8 00 02 00 01  00 00 01 f5 00 02 00 01
 # 三元组 (SoC-ID, 0x20001) x 3：415 / 456 / 501
 # 415 = SM8350（与 androidboot.platform_id=415 一致）

$ adb shell "cat /proc/version"
Linux version 5.4.295-qgki-gaaf8d3fac817-dirty (runner@runnervmtr4k5)
(Android (6443078 based on r383902) clang version 11.0.1 ...) #1 SMP PREEMPT
Tue Sep 29 21:12:15 UTC 2026

$ adb shell "getprop ro.product.model; getprop ro.product.device; getprop ro.build.fingerprint"
MT2110
OP5154L1
OnePlus/MT2110/OP5154L1:17/CP2A.260605.016/B.3e2f3d0-1859d0c-1859d08:user/release-keys

$ adb shell "getprop ro.board.platform; getprop ro.soc.model; getprop ro.soc.manufacturer; getprop ro.vendor.qti.soc_name; getprop ro.vendor.qti.soc_model"
lahaina
SM8350
Qualcomm
lahaina
SM8350

$ adb shell "getprop ro.separate.soft; getprop ro.boot.hw_version; getprop ro.boot.rf_version"
20820
22
11

$ adb shell "getprop ro.build.type; getprop ro.debuggable; getprop ro.secure"
user
0
1
```

---

## 2. 屏幕子系统

### 2.1 面板型号（型号级 OK）

| 参数 | 值 |
|---|---|
| 面板名（DT） | `samsung ams662zs01 fhd cmd mode dsc dsi panel` |
| 厂商名 | **AMS662ZS01**（三星） |
| 制造商标识 | `samsung1024` |
| 激活的 DT 面板节点 | `qcom,mdss_dsi_samsung_ams662zs01_dvt_dsc_cmd`（**DVT** 版） |
| 显示模式 | `dsi_cmd_mode`（CMD 命令模式）+ DSC 压缩 |
| 流量模式 | `burst_mode` |
| 色序 | `rgb_swap_rgb` |

**证据**

```
$ adb shell "cat /proc/cmdline"     # 挑出与显示/触摸相关的两个 kernel cmdline 参数
... msm_drm.dsi_display0=qcom,mdss_dsi_samsung_ams662zs01_dvt_dsc_cmd: \
    oplus_bsp_tp_custom.dsi_display0=qcom,mdss_dsi_samsung_ams662zs01_dvt_dsc_cmd:synaptics-s3908 ...

$ adb shell "su -c 'cat /proc/device-tree/soc/qcom,mdss_mdp@ae00000/qcom,mdss_dsi_samsung_ams662zs01_dvt_dsc_cmd/qcom,mdss-dsi-panel-name'; echo"
samsung ams662zs01 fhd cmd mode dsc dsi panel

$ adb shell "su -c 'cat .../oplus,mdss-dsi-vendor-name'; echo"
AMS662ZS01

$ adb shell "su -c 'cat .../oplus,mdss-dsi-manufacture'; echo"
samsung1024

$ adb shell "su -c 'cat .../qcom,mdss-dsi-panel-type'; echo"
dsi_cmd_mode

$ adb shell "su -c 'cat .../qcom,mdss-dsi-traffic-mode'; echo"
burst_mode
```

**DT 里的面板候选与「运行时选中哪一个」**（重要：DT 默认面板 != 本机面板）

```
# /proc/device-tree/soc/qcom,dsi-display-primary/qcom,dsi-default-panel = <0x5cb 0x5cc>
# 通过扫描全部 phandle 属性解析：
phandle 0x5cb -> /soc/qcom,mdss_mdp@ae00000/qcom,mdss_dsi_oplus19065_samsung_1440_3168_dsc_cmd
phandle 0x5cc -> /soc/qcom,mdss_mdp@ae00000/qcom,mdss_dsi_oplus20031_samsung_1440_3216_dsc_cmd
phandle 0x637 -> /soc/qcom,mdss_mdp@ae00000/qcom,mdss_dsi_samsung_ams662zs01_dsc_cmd
phandle 0x638 -> /soc/qcom,mdss_mdp@ae00000/qcom,mdss_dsi_samsung_ams662zs01_dvt_dsc_cmd   # <- 本机
```
即：DT 的 `qcom,dsi-default-panel` 指向 **1440x3168 / 1440x3216** 两块一加 9 Pro 类面板；本机实际面板由内核 cmdline
`msm_drm.dsi_display0=` **运行时覆盖**为 `ams662zs01_dvt`（phandle 0x638）。
移植时必须保留这套「DT 默认 + cmdline 覆盖」的选择机制，否则会点亮错面板。

```
$ adb shell "su -c 'ls /proc/device-tree/soc/'" | grep -i dsi_samsung
dsi_samsung_amb655x_dsc_cmd
dsi_samsung_amb670yf01_dsc_cmd
dsi_samsung_amb670yf01_dsc_cmd_2nd
dsi_samsung_amb670yf01_o_dsc_cmd
dsi_samsung_amb670yf01_o_dsc_cmd_2nd
dsi_samsung_ams662zs01_dsc_cmd
dsi_samsung_ams662zs01_dvt_dsc_cmd      # <- 本机
dsi_samsung_oneplus_dsc_cmd_display
dsi_samsung_sofef00_m_video
```

### 2.2 分辨率 / 物理尺寸 / 刷新率

| 参数 | 值 |
|---|---|
| 分辨率 | **1080 x 2400** |
| 物理尺寸 | 宽 70 mm x 高 153 mm -> 对角 168.3 mm ≈ **6.63 英寸** |
| 屏占比 prop | `ro.oplus.display.screenSizeInches.primary=6.62` |
| 刷新率 | **60 Hz / 120 Hz**（两档 timing，均为 cmd 模式） |
| 像素格式 | `qcom,mdss-dsi-bpp = 24`（RGB888） |
| 挖孔 | 有（`ro.oplus.display.screenhole.positon`） |

**证据**

```
$ adb shell "su -c 'cat /sys/class/drm/card0-DSI-1/status; cat /sys/class/drm/card0-DSI-1/enabled; cat /sys/class/drm/card0-DSI-1/modes'"
connected
enabled
1080x2400x60x183333cmd 1080x2400x120x183333cmd
#                        ^^ 60Hz            ^^^ 120Hz   <- 内核 DRM 报出的真实模式表

$ adb shell "su -c 'od -A n -v -t x1 .../timing0/qcom,mdss-dsi-panel-width'"
 00000438        # 1080
$ ... timing0/qcom,mdss-dsi-panel-height
 00000960        # 2400
$ ... timing0/qcom,mdss-dsi-panel-framerate
 0000003c        # 60
$ ... timing1/qcom,mdss-dsi-panel-framerate
 00000078        # 120

$ adb shell "su -c 'od -A n -v -t x1 .../qcom,mdss-pan-physical-width-dimension'"
 00000046   # 70 mm
$ ... /qcom,mdss-pan-physical-height-dimension
 00000099   # 153 mm

$ adb shell "su -c 'od -A n -v -t x1 /proc/device-tree/soc/qcom,mdss_mdp@ae00000/qcom,sde-qos-refresh-rates'"
 0000003c 00000078        # [60, 120]

$ adb shell "getprop ro.oplus.display.screenSizeInches.primary"
6.62
```

### 2.3 DSI 控制器 / PHY / 时序 / DSC

| 参数 | 值 |
|---|---|
| DSI 控制器 | `qcom,mdss_dsi_ctrl0@ae94000` + `ctrl1@ae96000`（**双 DSI**） |
| DSI PHY | `qcom,mdss_dsi_phy0@ae94900` + `phy1@ae96900`（v4.2，5nm PLL） |
| CTRL HW 版本 | `qcom,dsi-ctrl-hw-v2.5` |
| 选择时钟 | `mux_byte_clk0`, `mux_pixel_clk0` |
| DSI lane | lane 0-3 齐备（**4 lane**），lane-map `lane_map_0123` |
| SSC | `qcom,dsi-pll-ssc-en` 存在，`qcom,dsi-pll-ssc-mode = down-spread`，`oplus,dsi-pll-ssc-disalbed` 存在 |
| DSC | slice 540x30，`slice-per-pkt=2`，8 bpc / 8 bpp，block-prediction 开，`lm-split = <540 540>` |
| topology | `qcom,display-topology = <1 1 1 2 2 1>`，默认 index 1 |
| 面板复位时序 | `qcom,mdss-dsi-reset-sequence = <1 10 0 10 1 10>` |
| MDP 传输时间 | `qcom,mdss-mdp-transfer-time-us = 12000` |
| 帧阈值 | `frame-threshold-time-us = 800` |

**时序（timing0=60Hz / timing1=120Hz）**

| 参数 | timing0 (60Hz) | timing1 (120Hz) |
|---|---|---|
| h-front-porch | 64 | 8 |
| h-back-porch | 48 | 8 |
| h-pulse-width | 8 | 24 |
| v-front-porch | 8 | 2 |
| v-back-porch | 12 | 8 |
| v-pulse-width | 4 | 2 |

**证据**

```
# dsi-display-primary 的引用属性（属性值为 phandle），扫描全部 phandle 属性后解析：
#   qcom,dsi-display-primary/qcom,dsi-ctrl = <0x51f 0x520>
phandle 0x51f -> /soc/qcom,mdss_dsi_ctrl0@ae94000
phandle 0x520 -> /soc/qcom,mdss_dsi_ctrl1@ae96000
#   qcom,dsi-display-primary/qcom,dsi-phy = <0x521 0x522>
phandle 0x521 -> /soc/qcom,mdss_dsi_phy0@ae94900
phandle 0x522 -> /soc/qcom,mdss_dsi_phy1@ae96900
#   qcom,dsi-display-primary/qcom,mdp = <0x2a3>
phandle 0x2a3 -> /soc/qcom,mdss_mdp@ae00000

$ adb shell "su -c 'cat /proc/device-tree/soc/qcom,mdss_dsi_phy0@ae94900/compatible'; echo"
qcom,dsi-phy-v4.2
$ adb shell "su -c 'cat /proc/device-tree/soc/qcom,mdss_dsi_phy0@ae94900/pll-label'; echo"
dsi_pll_5nm
$ adb shell "su -c 'cat /proc/device-tree/soc/qcom,mdss_dsi_ctrl0@ae94000/compatible'; echo"
qcom,dsi-ctrl-hw-v2.5

# 时序（od 转 hex 后 big-endian 解码 / 或字符串即十进制）
timing0/qcom,mdss-dsi-h-front-porch   00000040  = 64
timing0/qcom,mdss-dsi-h-back-porch    00000030  = 48
timing0/qcom,mdss-dsi-h-pulse-width   00000008  = 8
timing0/qcom,mdss-dsi-v-front-porch   00000008  = 8
timing0/qcom,mdss-dsi-v-back-porch    0000000c  = 12
timing0/qcom,mdss-dsi-v-pulse-width   00000004  = 4
timing1/qcom,mdss-dsi-h-front-porch   00000008  = 8
timing1/qcom,mdss-dsi-h-back-porch    00000008  = 8
timing1/qcom,mdss-dsi-h-pulse-width   00000018  = 24
timing1/qcom,mdss-dsi-v-front-porch   00000002  = 2
timing1/qcom,mdss-dsi-v-back-porch    00000008  = 8
timing1/qcom,mdss-dsi-v-pulse-width   00000002  = 2

timing0/qcom,mdss-dsc-slice-width       0000021c  = 540
timing0/qcom,mdss-dsc-slice-height      0000001e  = 30
timing0/qcom,mdss-dsc-slice-per-pkt     00000002  = 2
timing0/qcom,mdss-dsc-bit-per-pixel     00000008  = 8
timing0/qcom,mdss-dsc-bit-per-component 00000008  = 8
timing0/qcom,compression-mode           "dsc"
timing0/qcom,display-topology           1 1 1 2 2 1
timing0/qcom,lm-split                   540 540
```
### 2.4 背光 / 亮度

| 参数 | 值 |
|---|---|
| 背光控制方式 | `bl_ctrl_dcs`（**DCS 命令调光**，非 PWM/BL IC） |
| 背光 class 节点 | `/sys/class/backlight/panel0-backlight` -> `.../soc/ae00000.qcom,mdss_mdp/backlight/panel0-backlight` |
| max_brightness | 4095 |
| 当前 brightness | 2540 |
| BL 等级范围 | min 1 / max 4095 / normal-max 2047 / default 4575 |
| DC 调光等级 | 520（`qcom,mdss-dsi-dc-backlight-level = 0x208`） |
| 峰值亮度（原始值） | `qcom,mdss-dsi-panel-peak-brightness = 0x005265C0` |
| 平均亮度（原始值） | `qcom,mdss-dsi-panel-average-brightness = 0x001E8480` |
| 黑白度 | `qcom,mdss-dsi-panel-blackness-level = 0xFA0 = 4000` |
| HDR | `qcom,mdss-dsi-panel-hdr-enabled` 存在；`ro.surface_flinger.has_HDR_display=true` |

> 注意：`panel-peak-brightness` / `average-brightness` 的单位在 QCOM/OPLUS 代码里不是纯 nit，本文只给**原始值 + 十六进制**，不做单位换算（避免编造）。设备侧另有厂商态亮度曲线表（见 §2.7）。

**证据**

```
$ adb shell "su -c 'cat /sys/class/backlight/panel0-backlight/max_brightness; cat /sys/class/backlight/panel0-backlight/brightness'"
4095
2540

$ adb shell "su -c 'cat .../qcom,mdss-dsi-bl-pmic-control-type'; echo"
bl_ctrl_dcs
$ ... qcom,mdss-dsi-bl-max-level          00000fff = 4095
$ ... qcom,mdss-dsi-bl-normal-max-level   000007ff = 2047
$ ... qcom,mdss-dsi-bl-min-level          00000001 = 1
$ ... qcom,mdss-brightness-max-level      00000fff = 4095
$ ... qcom,mdss-brightness-default-level  000011df = 4575
$ ... qcom,mdss-dsi-dc-backlight-level    00000208 = 520
$ ... qcom,mdss-dsi-panel-peak-brightness     005265c0
$ ... qcom,mdss-dsi-panel-average-brightness  001e8480
$ ... qcom,mdss-dsi-panel-blackness-level     00000fa0 = 4000

$ adb shell "cat /sys/class/backlight/panel0-backlight/type"
raw
```

### 2.5 面板供电（regulator 逐个溯源到 PMIC）

| 供电 | 源（phandle 解析后） |
|---|---|
| `vddio-supply` | `rpmh-regulator-ldoc12` -> **PM8350C L12** |
| `vdd-supply` | `rpmh-regulator-ldoc13` -> **PM8350C L13** |
| `px_v18r-supply` | `rpmh-regulator-ldoc8` -> **PM8350C L8** |
| `avdd-supply` | `display_gpio_regulator@1`（GPIO 控制的 AVDD，+1.8V） |
| 供电表 | `dsi_panel_oplus_pwr_supply` |
| 复位 GPIO | `qcom,platform-reset-gpio = <0x93 0x18 0x00>` |
| TE GPIO | `qcom,platform-te-gpio = <0x93 0x52 0x00>` |

（`0x93` = `/soc/pinctrl@f000000` 的 phandle；`0x18`=GPIO24，`0x52`=GPIO82）

**证据**

```
# dsi-display-primary 里的 supply phandle，扫描全部 phandle 属性解析：
avdd-supply    = 0x5d3 -> /soc/display_gpio_regulator@1
vddio-supply   = 0x027 -> /soc/rsc@18200000/rpmh-regulator-ldoc12/regulator-pm8350c-l12
vdd-supply     = 0x028 -> /soc/rsc@18200000/rpmh-regulator-ldoc13/regulator-pm8350c-l13
px_v18r-supply = 0x2e9 -> /soc/rsc@18200000/rpmh-regulator-ldoc8/regulator-pm8350c-l8
qcom,panel-supply-entries = 0x5c6 -> /soc/dsi_panel_oplus_pwr_supply
```

### 2.6 MDSS/SDE 与 Pixelworks iris

| 参数 | 值 |
|---|---|
| MDSS 节点 | `qcom,mdss_mdp@ae00000`（compatible `qcom,sde-kms`） |
| DSC HW 版本 | `dsc_1_2` |
| CSC 类型 | `csc-10bit` |
| mixer 数 | 6 |
| CTL 数 | 5 |
| DRAM 通道 | 2 |
| 最大带宽 | `qcom,sde-max-bw-high-kbps = 0x00EC82E0` ≈ 15,499,904 kbps |
| Pixelworks iris | 面板节点有 `pxlw,soft-iris-enable`；`dsi-display-primary/pxlw,iris-lightup-config` -> `pxlw,iris_cfg_samsung_amb670yf01_dsc_cmd`；服务 `vendor.pixelworks.hardware.display` running |
| LTM/QDCM 校准文件 | `/vendor/etc/ltm_config_samsung_ams662zs01_dvt_dsc_cmd_mode_panel.xml`、`ltm_config_samsung_ams662zs01_fhd_cmd_mode_dsc_dsi_panel.xml` |
| DRM 设备 | `card0-DSI-1`, `card0-DP-1`, `card0-Virtual-1`；`sde-crtc-0..4` |

> 注意：iris lightup config 指向的是 **amb670yf01** 面板的配置（DT 里唯一一份 `pxlw,iris_cfg_samsung_amb670yf01_dsc_cmd`），不是 ams662zs01 专属配置 —— **移植时这里是个坑**。

**证据**

```
$ adb shell "su -c 'cat /proc/device-tree/soc/qcom,mdss_mdp@ae00000/compatible'; echo"
qcom,sde-kms
$ adb shell "su -c 'cat .../qcom,sde-dsc-hw-rev'; echo"
dsc_1_2
$ adb shell "su -c 'ls /proc/device-tree/soc/pxlw/'"
name pxlw,iris_cfg_samsung_amb670yf01_dsc_cmd

$ adb shell "su -c 'ls /sys/class/drm/'"
card0 card0-DP-1 card0-DSI-1 card0-Virtual-1 renderD128 sde-conn-1-DSI-1
sde-crtc-0 sde-crtc-1 sde-crtc-2 sde-crtc-3 sde-crtc-4 ttm version

$ adb shell "getprop | grep -i pixelworks"
[init.svc.vendor.pixelworks.hardware.display]: [running]
```

### 2.7 厂商态亮度曲线（OPLUS HAL 下发，非内核）

```
$ adb shell "getprop | grep 'ro.oplus.display.brightness'"
[ro.oplus.display.brightness.apollo.xs]: [0,2,4,...,8600]      # 输入档位
[ro.oplus.display.brightness.apollo.ys]: [212,1387,...,8191]   # 输出 DCS 值
[ro.oplus.display.brightness.apollo.normal_max_brightness]: [8191]
[ro.oplus.display.brightness.apollo.hbm_ys]: [8191,8694,8894,9244,9624,10239]
[ro.oplus.display.brightness.xs]: [0,1,2,3,8,16,36,60,100,260,540,1000,2000,3500,5900,8500]
[ro.oplus.display.brightness.ys]: [168,258,327,377,585,720,924,928,943,995,1096,1279,1588,1799,1954,2047]
[ro.oplus.display.brightness.default_brightness]: [1600]
[ro.oplus.display.sell_mode.max_normal_nit]: [800]
[ro.oplus.display.peak.brightness.duration_time]: [15]
[ro.oplus.display.vrr]: [1]
[persist.oplus.display.ogfr.exclusive]: [144,165]
```
> `max_normal_nit=800` 是厂商态标称的「正常模式最高 800 nit」。`ogfr.exclusive=144,165` 是 OPLUS 的全局帧率白名单，
> **不代表本面板支持 144Hz** —— 内核 DRM 只报 60/120Hz（见 §2.2），以 DRM 为准。

---

## 3. 充电子系统

### 3.1 电池与充电规格

| 参数 | 值 |
|---|---|
| 电池 | Li-ion，**双电芯**（`qcom,vbatt_num = 2`） |
| 设计容量 | `charge_full_design = 4176000 uAh` ≈ **4176 mAh** |
| DT 标称容量 | `qcom,batt_capacity_mah = 4500` |
| 满充电压 | `voltage_max = 4435 mV`；`qcom,vbatt_full_thr = 4435` |
| 当前电压/温度 | 4254000 uV / 40.6 摄氏度（`temp=406`，单位 0.1 度） |
| 循环次数 | 5 |
| **最大有线充电** | **65 W**（10 V x 6.5 A） |
| 快充协议 | **SuperVOOC（ADSP VOOCPHY）+ PD + PD PPS + QC** |
| 无线充电 | **无**（DT 有 idt9412 固件名与 wls 策略，但 `/sys/class/power_supply/wireless` `present=0 online=0`） |

**证据（最大功率 65W 有两处独立实证）**

```
$ adb shell "su -c 'cat .../qcom,vooc-max-input-volt-support' | od -A n -t u4"
 0000000000002710   -> 0x2710 = 10000  (mV)
$ adb shell "su -c 'od -A n -v -t x1 .../qcom,vooc-max-input-current-support'"
 00001964           -> 6500 (mA)
# 10000 mV x 6500 mA = 65 W

$ adb shell "su -c 'dmesg | grep ui_power_show'"
[OPLUS_CHG][ui_power_show]ui_power_show: 2500 65000 0 0 0 0 -1
#                                        ^^^^ ^^^^^ = 65000 (mW) = 65W

$ adb shell "su -c 'ls .../qcom,battery_charger/ | grep vooc_charge_strategy'"
vooc_charge_strategy_30w  vooc_charge_strategy_65w
```

### 3.2 充电 IC / 充电路径（型号级 OK 部分）

| 参数 | 值 |
|---|---|
| 充电主路径 | `qcom,chg_ops = "plat-pmic"` -> **走 PMIC 直充**（非独立充电 IC） |
| SPMI PMIC 清单 | `pm8350@1`, `pm8350b@3`, `pm8350c@2`, `pmk8350@0`, `pmr735a@4`, `pmr735b@5` |
| 电荷泵/VOOCPHY | `soc/oplus,adsp-voocphy`（compatible `oplus,adsp-voocphy`）-> **VOOCPHY 跑在 ADSP 上** |
| VOOC 支持位 | `qcom,voocphy_support = 1`, `qcom,vooc_project = 3`, `track,voocphy_type` 存在 |
| 燃料计 | **ZY0603**（dmesg `fg_zy0603_*`） |
| 充电框架 | `qcom,pmic_glink/qcom,battery_charger`，protection-domain = `tms/servreg msm/adsp/charger_pd`，subsys = `adsp` |
| OPLUS 充电 class | `/sys/class/oplus_chg/{ac,battery,common,usb,wireless}` |
| Type-C | `/sys/class/typec/port0`（+ `port0-partner`） |
| USB 控制器 | `ssusb@a600000`（compatible `qcom,dwc-usb3-msm`），UDC `a600000.dwc3` |

**证据**

```
$ adb shell "su -c 'cat .../qcom,battery_charger/qcom,chg_ops'; echo"
plat-pmic

$ adb shell "su -c 'for f in /proc/device-tree/soc/qcom,spmi@c440000/*/compatible; do printf "%s = " "$(basename $(dirname $f))"; cat "$f" | tr -d "\0"; echo; done'"
qcom,pm8350@1 = qcom,spmi-pmic
qcom,pm8350b@3 = qcom,spmi-pmic
qcom,pm8350c@2 = qcom,spmi-pmic
qcom,pmk8350@0 = qcom,spmi-pmic
qcom,pmr735a@4 = qcom,spmi-pmic
qcom,pmr735b@5 = qcom,spmi-pmic

$ adb shell "su -c 'cat /proc/device-tree/soc/oplus,adsp-voocphy/compatible'; echo"
oplus,adsp-voocphy

$ adb shell "su -c 'dmesg | grep -m2 fg_zy0603'"
[OPLUS_CHG][fg_zy0603_get_afi_update_done]read afi update success, afi update done=0
[OPLUS_CHG][oplus_chg_fast_switch_check]zy gauge afi_update_done ing...

$ adb shell "su -c 'od -A n -t u4 .../qcom,voocphy_support'"
 0000000000000001
$ ... qcom,vooc_project
 0000000000000003
$ ... qcom,protection-domain  (pmic_glink)
tms/servreg msm/adsp/charger_pd

$ adb shell "su -c 'ls /sys/class/oplus_chg/'"
ac battery common usb wireless
```

### 3.3 PD / PPS / QC 支持

| 参数 | 值 |
|---|---|
| PPS | `qcom,pps_support_type = 1`；完整 `pps_charge_strategy`（4 个 SOC 段 x 4 个温度段） |
| PD | `qcom,pd_input_current_charger_ma = 2000`；`qcom,pd_temp_*_fastchg_current_ma` 分温区 |
| QC | `qcom,qc_input_current_charger_ma = 2000` |
| usb_type 运行时 | `Unknown [SDP] DCP CDP ACA C PD PD_DRP PD_PPS BrickID` -> **PD 与 PD_PPS 都在支持列表** |
| 当前 usb 输入限制 | `input_current_limit = 2000000 uA`（2 A），`voltage_max = 5000000 uV` |
| 热降流阶梯 | `qcom,thermal-mitigation = <3000000 1500000 1000000 500000>` uA |

**证据**

```
$ adb shell "su -c 'cat /sys/class/power_supply/usb/usb_type'"
Unknown [SDP] DCP CDP ACA C PD PD_DRP PD_PPS BrickID

$ adb shell "su -c 'for f in /sys/class/power_supply/usb/*; do printf "%s=%s\n" "$(basename $f)" "$(cat $f 2>/dev/null)"; done'"
current_max=900000
current_now=500000
input_current_limit=2000000
online=1
type=USB
usb_type=Unknown [SDP] DCP CDP ACA C PD PD_DRP PD_PPS BrickID
voltage_max=5000000
voltage_now=4736000

$ adb shell "su -c 'od -A n -v -t x1 .../qcom,pps_support_type'"
 00000001

$ adb shell "su -c 'ls .../qcom,battery_charger/pps_charge_strategy/'"
name strategy_soc_0_to_50 strategy_soc_50_to_75 strategy_soc_75_to_85 strategy_soc_85_to_90
# 每段形如 <输入电流mA 电压mV 功率mW 是否截断 保留>，例：
strategy_temp_0_to_50 = [10000,4180,2000,0,0, 10000,4420,1500,0,0, 10000,4430,1000,1,0]
```

### 3.4 无线充电：有 DT 定义、无硬件

```
$ adb shell "su -c 'cat .../qcom,battery_charger/qcom,wireless-fw-name'; echo"
idt9412.bin
$ adb shell "su -c 'cat /sys/class/power_supply/wireless/present; cat /sys/class/power_supply/wireless/online; cat /sys/class/power_supply/wireless/type'"
0
0
Wireless
```
**结论：**`wireless` power_supply 设备存在但 `present=0 / online=0`，`current_max=0`；
DT 里的 `idt9412.bin` 与 `wls_*` 策略来自共享基类 DTSI，**本机（9RT）无无线充电硬件**。

### 3.5 电池/整机实测状态（采集瞬间）

```
[battery] capacity=100  status=Full  health=Good  technology=Li-ion
          charge_full=4176000  charge_full_design=4176000  charge_counter=3478000
          voltage_now=4254000  voltage_max=4435  temp=406  cycle_count=5
```

---
## 4. 触摸子系统

### 4.1 触摸 IC（型号级 OK）

| 参数 | 值 |
|---|---|
| **触摸 IC** | **Synaptics S3908** |
| compatible | `synaptics-s3908` |
| DT 节点名 | `/soc/i2c@990000/synaptics20031@4B`（名字里的 `20031` 是项目号，非本机项目 20820） |
| chip-name | `S3908` |
| **总线** | **I2C**，主机 `i2c-5`（= `990000.i2c` = qupv3_se4），**从地址 0x4B** |
| 中断 | `interrupts = <23 8200>`（GIC SPI 23），`irq-gpio` 存在（`irq_need_dev_resume_ok` 存在） |
| 复位 GPIO | `reset-gpio` 存在 |
| 供电 | `vdd_2v8-supply` = PM8350C L3（`vdd_2v8_volt=3008000 uV`）；`vcc_1v8-supply` = PM8350C L8 |
| 驱动 | `/sys/class/synaptics_tcm_oncell/tcm0` -> **Synaptics TCM OnCell** 驱动；`vendor.touch-aidl-1` running |
| 固件名 | `firmware_name = "SS"`（设备侧未定位到对应固件文件，见 §7） |
| 触控规格 | TX x RX = **16 x 35**；显示坐标 1080x2400；面板坐标 8640x19200；最大 10 点；`tp_ic_type = 2` |
| 手势/功能 | `black_gesture_support`, `smart_gesture_support`, `fingerprint_underscreen_support`, `screenoff_fingerprint_info_support`, `kernel_grip_support`, `health_monitor_support`, `fw_update_app_support` 等均存在 |

### 4.2 「总线上一堆候选 IC，到底哪颗在用」——四重独立证据

`i2c@990000` 上同时挂了 4 个**互斥的触摸驱动候选**节点：

| 节点 | compatible | status | input 设备 | 是否本机触摸 |
|---|---|---|---|---|
| `synaptics20031@4B` | `synaptics-s3908` | （默认 okay） | **input7 "touchpanel"** | **是** |
| `st_fts@49` | `st,fts` | **disabled** | 无 | 否 |
| `focaltech@38` | `focaltech,fts_ts` | **disabled** | 无 | 否 |
| `s6sy791_19065@48` | `sec-s6sy791` | 已绑定（5-0048） | 无（非 touchpanel） | 否 |

```
$ adb shell "su -c 'cat /proc/bus/input/devices'"    # 只看触摸那两条
I: Bus=0018 Vendor=0901 Product=0902 Version=0000
N: Name="touchpanel"
S: Sysfs=/devices/platform/soc/990000.i2c/i2c-5/5-004b/input/input7
H: Handlers=cpufreq touch_boost_cpufreq event7 kgsl
B: EV=b   B: KEY=420 ...   B: ABS=6e5800200000000
--
I: Bus=0000 Vendor=0000 Product=0000 Version=0000
N: Name="touchpanel_kpd"
S: Sysfs=/devices/platform/soc/990000.i2c/i2c-5/5-004b/input/input8

$ adb shell "su -c 'for d in /sys/bus/i2c/devices/*/; do printf "%s name=" "$(basename $d)"; cat $d/name 2>/dev/null; echo; done'" | grep -i -E "syn|s6sy|fts|focal"
5-0048 name=sec-s6sy791
5-004b name=synaptics-s3908          # <- 实际绑定的触摸设备

$ adb shell "su -c 'ls /sys/class/synaptics_tcm_oncell/'"
tcm0
```

**证据 3（最有决定性）：DT 里「触摸节点 -> 面板节点」的 phandle 白名单**

```
# 本机面板节点 qcom,mdss_dsi_samsung_ams662zs01_dvt_dsc_cmd 的 phandle = 0x638
# 各触摸节点的 panel 属性（phandle 列表）：
synaptics20031@4B/panel  = <0x5cc 0x633 0x634 0x635 0x636 0x637 0x638>   # <- 含 0x638
st_fts@49/panel          = <0x57b 0x57c 0x57d>                            # <- 不含
focaltech@38/panel       = <0x57e 0x5f0 0x57f 0x5f1>                      # <- 不含
s6sy791_19065@48/panel   = <0x5cb>                                        # <- 不含
# 且 s6sy791 的 platform_support_project_commandline =
#   "mdss_dsi_oplus19065_samsung_1440_3168_dsc_cmd"  （一加 9 Pro 的面板，不是本机）
```

**证据 4：kernel cmdline 显式把面板与触摸驱动配成一对**

```
$ adb shell "cat /proc/cmdline" | tr ' ' '\n' | grep oplus_bsp_tp_custom
oplus_bsp_tp_custom.dsi_display0=qcom,mdss_dsi_samsung_ams662zs01_dvt_dsc_cmd:synaptics-s3908
```

> **陷阱**：DT 里 `i2c@990000/qcom,i2c-touch-active` 的值是 **`st,fts`**，与实际绑定的 `synaptics-s3908` **不一致**。这是一个「候选驱动名提示」属性，**不能当作实际 IC 的依据**；真正决定权在 `oplus_bsp_tp_custom.*` cmdline + 各节点的 `panel` phandle 白名单。移植时若照搬该属性会选错触摸驱动。

```
$ adb shell "su -c 'cat /proc/device-tree/soc/i2c@990000/qcom,i2c-touch-active'; echo"
st,fts
$ adb shell "su -c 'cat /proc/device-tree/soc/i2c@990000/st_fts@49/status'; echo"
disabled
```

### 4.3 关键触摸属性原始值

```
synaptics20031@4B/chip-name             "S3908"
synaptics20031@4B/compatible            "synaptics-s3908"
synaptics20031@4B/firmware_name         "SS"
synaptics20031@4B/name                  "synaptics20031"
synaptics20031@4B/panel_type            0x08 = 8
synaptics20031@4B/reg                   "  K"  -> 0x4B           (I2C 从地址)
synaptics20031@4B/touchpanel,tx-rx-num        [16, 35]
synaptics20031@4B/earsense,tx-rx-num          [17, 18]
synaptics20031@4B/touchpanel,display-coords   [1080, 2400]
synaptics20031@4B/touchpanel,panel-coords     [8640, 19200]
synaptics20031@4B/touchpanel,max-num-support  10
synaptics20031@4B/touchpanel,tp_ic_type       2
synaptics20031@4B/touchpanel,int-mode         1
synaptics20031@4B/touchpanel,button-type      4
synaptics20031@4B/vdd_2v8_volt                3008000 (uV)
synaptics20031@4B/platform_support_project    "QT QU"
synaptics20031@4B/platform_support_project_dir "QT QT"
synaptics20031@4B/interrupts                  <23 8200>
```

---

## 5. 摄像头子系统

### 5.1 平台侧

| 参数 | 值 |
|---|---|
| SoC 相机子系统 | SM8350 CPAS / ICP / IPE / BPS / JPEG |
| CCI | **CCI0 + CCI1**（两组 I2C 主控） |
| CSIPHY | csiphy0 - csiphy5（**6 路**） |
| CSID / IFE | csid0/1/2（+ csid-lite0/1）、ife0/1/2（+ ife-lite0/1） |
| 相机供电 | `cam_vio-supply` = `/soc/i2c@a94000/pm8008i@9/qcom,pm8008i-regulator/regulator@4400` -> **PM8008i** 摄像头 PMIC |
| 相机 provider | `vendor.camera-provider-2-4`, `cameraserver` running |
| 摄像头内核模块 | `/vendor/lib/modules/camera.ko`（**9.6 MB**，已加载，refcount 34） |

### 5.2 4 个 sensor 槽位（DT 证据）

```
$ adb shell "su -c 'ls /proc/device-tree/soc/qcom,cci0/ /proc/device-tree/soc/qcom,cci1/'"
qcom,cci0: martini,cam-sensor0  martini,cam-sensor1  martini,ois0
           qcom,actuator@0  qcom,eeprom@0  qcom,eeprom@1
qcom,cci1: martini,cam-sensor2  martini,cam-sensor3
           qcom,eeprom@2  qcom,eeprom@3
```

| 槽位 | cell-index | cci-master | csiphy-sd-index | GPIO 标签 | 附属 | status |
|---|---|---|---|---|---|---|
| `martini,cam-sensor0` | 0 | 0 | 1 | `CAMIF_MCLK0 / CAM_RESET0 / CAM_AVDD0` | `actuator@0` + `eeprom@0` + `ois0` | ok |
| `martini,cam-sensor1` | 1 | 1 | 2 | `CAMIF_MCLK1 / CAM_RESET1 / CAM_AVDD1 / CAM_DVDD1` | `eeprom@1` | ok |
| `martini,cam-sensor2` | 2 | 1 | 5 | `CAM_front_MCLK / CAM_front_RESET` | `eeprom@2` | ok |
| `martini,cam-sensor3` | 3 | 0 | 0 | `CAMIF_MCLK2 / CAM_RESET2` | `eeprom@3` | ok |

> `cam-sensor2` 的 GPIO 标签是 `CAM_front_*` -> **前摄 = 槽位 2**（DT 直证，不是推断）。

```
$ adb shell "su -c 'od -A n -v -t x1 /proc/device-tree/soc/qcom,cci0/martini,cam-sensor0/cell-index'"
 00000000
$ ... cam-sensor0/cci-master
 00000000
$ ... cam-sensor0/csiphy-sd-index
 00000001
$ ... cam-sensor0/gpio-req-tbl-label | tr -d '\0'
CAMIF_MCLK0 CAM_RESET0 CAM_AVDD0
$ ... cam-sensor0/compatible | tr -d '\0'
qcom,cam-sensor

$ ... cam-sensor2/gpio-req-tbl-label | tr -d '\0'
CAM_front_MCLK CAM_front_RESET

$ adb shell "su -c 'cat /proc/device-tree/soc/qcom,cci0/martini,cam-sensor0/status'; echo"
ok
# 4 个 sensor 节点 status 全为 ok
```

### 5.3 摄像头型号（型号级 OK）

**项目号 20820 专属的 sensor 库**（`androidboot.prjname=20820`，与 `oppo,dtsi_no=0x5154=20820` 对得上）：

```
$ adb shell "su -c 'ls /odm/lib64/camera/' | grep 20820"
com.qti.sensor.imx766.20820.so
com.qti.sensor.imx481.20820.so
com.qti.sensor.ov02b.20820.so
com.qti.sensor.imx471.20820.so
com.qti.tuned.sm8350_sunny_imx766_20820.bin
com.qti.tuned.sm8350_ofilm_imx481_20820.bin
com.qti.tuned.sm8350_truly_imx471_20820.bin
```

**camx 模块校准 bin**（同样只出现在本机需要的那几颗）：

```
$ adb shell "su -c 'ls /odm/lib64/camera/' | grep sensormodule"
com.qti.sensormodule.qtech_imx766.bin        # Q Technology, IMX766
com.qti.sensormodule.ofilm_imx481.bin       # O-Film, IMX481
com.qti.sensormodule.shine_ov02b.bin        # Shine, OV02B
com.qti.sensormodule.truly_imx471.bin       # Truly, IMX471
# 另有 lemonade / lemonadep（一加 9/9Pro）的 imx789 / gc02m1b / ov08a10 等，属其它项目
```

**尺寸校验（独立佐证）**

```
$ adb shell "getprop ro.vendor.oplus.camera.backCamSize; getprop ro.vendor.oplus.camera.frontCamSize; getprop ro.vendor.oplus.camera.isHasselbladCamera"
50MP+16MP+2MP
16MP
0
```

**camera 设备数校验**

```
$ adb shell "su -c 'dumpsys media.camera' | head -12"
Number of camera devices: 5
Number of normal camera devices: 4          # <- 4 颗物理 sensor，与 4 个 DT 节点一致
Number of public camera devices visible to API1: 2
```

**槽位 -> 型号 映射**（`【推断】`：DT 的 sensor 节点只写 `compatible = qcom,cam-sensor`，**不含型号**；型号来自上面的库文件名集合，槽位归属按 csiphy 编号 / 主摄附属件推断）：

| 槽位 | 推断型号 | 推断角色 | 依据 |
|---|---|---|---|
| 0 | **IMX766** | 主摄 50 MP（带 OIS + VCM + EEPROM） | 唯一带 `actuator@0` + `ois0` 的槽位；50MP 档 |
| 1 | **IMX481** | 超广 16 MP | 16MP 档，非前摄（无 `CAM_front_*` 标签） |
| 2 | **IMX471** | 前摄 16 MP | GPIO 标签 `CAM_front_MCLK/RESET` |
| 3 | **OV02B** | 微距 2 MP | 2MP 档，剩余槽位 |

> 关于 `/odm/etc/camera/camera_engmode.xml`：该文件列出了 `imx686 / imx766 / imx481 / imx471 / ov02b` 五颗 sensor（`imx686` 属其它项目），且 `CameraId` 出现两个 0 —— 它是**跨项目共享文件**，**不能作为本机 sensor 清单的依据**，仅作交叉参考。

### 5.4 OIS / AF / EEPROM / 闪光灯

| 参数 | 值 |
|---|---|
| **OIS IC** | **ROHM BU63169**（`martini,ois0/ois,name = "bu63169"`） |
| OIS 附属 | `ois,fw=1`, `ois,type=0`, `ois_actuator,vednor=2`, `ois_gyro,type=3`, `ois_gyro,position=3`, `ois_module,vendor=1` |
| OIS 供电 | `cam_vdig-supply`，2.8-3.008 V |
| OIS 调试节点 | `/sys/kernel/ois_control/dump_registers` |
| AF (VCM) | `qcom,actuator@0`（cell-index 0，`cam_vaf-supply`）—— **DT 未给 VCM 型号** |
| EEPROM | 5 个节点：`cci0/eeprom@0,@1`、`cci1/eeprom@2,@3`；主摄另有 `is-read-eeprom = 2` |
| 闪光灯 | DT 3 个 `martini,camera-flash0/1/2`（`compatible = qcom,camera-flash`, `qcom,flash-name = "pmic"`） |
| 闪光灯实现 | **PM8350C `flash_led@ee00`**：`led:flash_0..3`(max 1500)、`led:torch_0..3`(max 500)、`led:switch_0..2`(max 255) |
| LED class | `/sys/class/leds/` 下另有 `red/green/blue`（PM8350C `leds@ef00`，max 255） |

**证据**

```
$ adb shell "su -c 'cat /proc/device-tree/soc/qcom,cci0/martini,ois0/ois,name'; echo"
bu63169
$ adb shell "su -c 'cat /proc/device-tree/soc/qcom,cci0/martini,ois0/compatible'; echo"
qcom,ois
$ adb shell "su -c 'ls /sys/kernel/ois_control/'"
dump_registers

$ adb shell "su -c 'cat /proc/device-tree/soc/martini,camera-flash0/qcom,flash-name'; echo"
pmic
$ adb shell "su -c 'cat /proc/device-tree/soc/martini,camera-flash0/compatible'; echo"
qcom,camera-flash

$ adb shell "su -c 'for d in /sys/class/leds/*/; do n=$(basename $d); printf "%s max=%s dev=%s\n" "$n" "$(cat $d/max_brightness 2>/dev/null)" "$(readlink -f $d/device)"; done'"
led:flash_0 max=1500 dev=.../qcom,pm8350c@2:qcom,flash_led@ee00
led:flash_1 max=1500 dev=.../qcom,pm8350c@2:qcom,flash_led@ee00
led:flash_2 max=1500 ...
led:flash_3 max=1500 ...
led:torch_0 max=500  ...
led:switch_0 max=255 ...
red max=255 dev=.../qcom,pm8350c@2:qcom,leds@ef00
green max=255 ...
blue max=255 ...
```

### 5.5 I2C 地址 —— **DT 未提供**

```
$ adb shell "su -c 'ls /proc/device-tree/soc/qcom,cci0/martini,cam-sensor0/'"
actuator-src cam_clk-supply cam_v_custom1-supply cam_vaf-supply cam_vana-supply
cam_vdig-supply cam_vio-supply cci-master cell-index clock-cntl-level clock-names
clock-rates clocks compatible csiphy-sd-index eeprom-src gpio-no-mux
gpio-req-tbl-flags gpio-req-tbl-label gpio-req-tbl-num gpio-reset gpio-vana gpios
is-read-eeprom led-flash-src name ois-src pinctrl-0 pinctrl-1 pinctrl-names
reg-setting-ver regulator-names rgltr-cntrl-support rgltr-load-current
rgltr-max-voltage rgltr-min-voltage sensor-position-pitch sensor-position-roll
sensor-position-yaw status
#                                          ^^^^ 无 reg 属性 -> I2C 地址不在 DT
```
CAMX 的 sensor I2C 地址在 `/odm/lib64/camera/com.qti.sensor.<model>.<prj>.so` / `com.qti.sensormodule.*.bin` 里，**设备侧 DT 拿不到 -> 需源码侧/固件侧补**。

---
## 6. 其他相关子系统（顺带取得，供移植参考）

### 6.1 内核模块 / 驱动清单（`/vendor/lib/modules/`）

```
$ adb shell "ls /vendor/lib/modules/"
5.4-gki                                     # <- 注意：模块目录本身以 "5.4-gki" 命名
adsp_loader_dlkm.ko apr_dlkm.ko aw8697.ko aw882xx_dlkm.ko bolero_cdc_dlkm.ko
bt_fm_slim.ko btpower.ko camera.ko e4000.ko explorer.ko ... haptic.ko
haptic_feedback.ko hdmi_dlkm.ko hid-aksys.ko horae_shell_temp.ko
last_boot_reason.ko llcc_perfmon.ko machine_dlkm.ko mbhc_dlkm.ko msm_drm.ko
native_dlkm.ko oplus_bsp_ir_core.ko oplus_bsp_kookong_ir_spi.ko
oplus_bsp_proactive_compact.ko oplus_bsp_sigkill_diagnosis.ko
oplus_connectivity_routerboost.ko ordump.ko platform_dlkm.ko q6_dlkm.ko ...
radio-i2c-rtc6226-qca.ko rdbg.ko rmnet_*.ko snd_event_dlkm.ko swr_*.ko
tfa98xx-v6_dlkm.ko uff_fp_driver.ko wcd9xxx_dlkm.ko wcd_core_dlkm.ko ...
```
（完整清单另见 `modules.load` / `modules.dep` / `modules.softdep`）

**已加载模块**（`/proc/modules`，选摘大小）

```
camera                          9674752  34 explorer, Live      # 相机栈整体是一个 9.6MB 模块
platform_dlkm                   3534848  57 native_dlkm, Live
msm_drm                         2846720  13 - Live              # 显示
q6_dlkm                         1126400  13 ...
aw8697                           458752   0 - Live              # 线性马达
haptic                           348160   0 - Live
oplus_bsp_sigkill_diagnosis       20480   0 - Live
oplus_bsp_proactive_compact       24576   0 - Live
uff_fp_driver                     61440   0 - Live              # 屏下指纹
horae_shell_temp                  24576   0 - Live
```

### 6.2 其它板载器件（顺带）

| 器件 | 型号/节点 | 证据 |
|---|---|---|
| 屏下指纹 | `goodix_fp` DT 节点 + `oplus_fp_common` + `uff_fp_driver.ko` | `ls /proc/device-tree/soc/ | grep fp` |
| 触摸/Audio 放大器 | `tfa98xx` @ i2c-4 0x34/0x35 | `/sys/bus/i2c/devices/4-0034/name` |
| 霍尔传感器 | `hall-mxm1120,up/down` @ i2c-8 0x0c/0x0d；`hall-ist8801,up/down` @ 0x18/0x19 | `/sys/bus/i2c/devices/` |
| NFC | `nq-nci` @ i2c-8 0x28 | 同上 |
| 音频 switch | `fsa4480-i2c` @ i2c-7 0x42；`rtc6226` @ 0x64 | 同上 |
| 显示 iris 通路 | `iris` @ i2c-6 0x26、`iris-i2c` @ 0x22 | 同上 |
| 摄像头 PMIC | `pm8008i@9` @ i2c@a94000 | DT |
| Wi-Fi/BT | `qca6490`（DT `soc/bt_qca6490`、`qcom,cnss-qca6490@b0000000`） | DT |
| 触觉 | `qcom-hv-haptics`（PM8350B `hav-haptics@f000`）+ `aw8697` | `/proc/bus/input/devices` |
| 存储 | UFS `1d84000.ufshc` | `androidboot.bootdevice=1d84000.ufshc` |
| 调试 | `msm_rtb.filter=0x237`（cmdline）；但 `/sys/kernel/debug` **未挂载**，`/proc/rtb` 不存在 | cmdline / ls |

### 6.3 开机日志可得性（重要限制）

```
$ adb shell "su -c 'dmesg | wc -l; dmesg | head -1'"
11500
[35216.440468] ...                    # 设备已运行约 9.8 小时
$ adb shell "su -c 'ls /sys/fs/pstore/'"        # 空
$ adb shell "su -c 'ls /proc/last_kmsg /sys/kernel/boot_kmsg'"
ls: /proc/last_kmsg: No such file or directory
ls: /sys/kernel/boot_kmsg: No such file or directory
$ adb shell "su -c 'ls /sys/kernel/debug/'"     # debugfs 未挂载
ls: /sys/kernel/debug/: No such file or directory
```
-> **屏幕/触摸/摄像头的 probe 期 dmesg（boot 阶段）已被 ring buffer 滚掉，设备侧无法取到**。
本文所有结论因此都建立在 device-tree / sysfs / classes 与 cmdline 上，而非 probe 日志。

---

## 7. 设备侧拿不到、需要源码侧补

| # | 缺失项 | 为什么设备侧拿不到 | 建议从哪补 |
|---|---|---|---|
| 1 | 面板初始化序列**命令语义**（`qcom,mdss-dsi-on-command` 的 0x39/0x15/0x05 裸 hex） | DT 存的是原始字节流，语义在驱动/DTSI 里 | `oplus_display` / `dsi_panel_ams662zs01*` 驱动源码，或 OPLUS DTSI |
| 2 | DSI 实际 **link 速率 (Mbps/lane)**、PLL 分频 | `qcom,mdss-dsi-panel-phy-timings = 00240a0a1a19090a090204001e0f` 是压缩编码 | `dsi_phy_*` 驱动 + panel timing 计算代码 |
| 3 | **峰值/平均亮度真实 nit** | 属性是 QCOM 私有单位（本文只给原值） | `sde`/OPLUS 亮度曲线代码 + panel spec |
| 4 | Synaptics S3908 **固件文件** | `firmware_name="SS"`；`/vendor/firmware`（147 项）内无 synaptics/触摸相关文件 | OPLUS `synaptics_tcm_oncell` 驱动的 fw 搜索路径 / `/odm/firmware` |
| 5 | Sensor **I2C 地址**（4 颗全部） | DT 的 `qcom,cam-sensor` 节点**无 `reg` 属性** | `com.qti.sensor.<model>.<prj>.so` / `com.qti.sensormodule.*.bin` |
| 6 | Sensor **槽位与型号精确对应** | DT 只写 `compatible = qcom,cam-sensor`，型号不在 DT | CAMX sensor .so 的 slot 信息 / OPLUS 相机 XML（`camera_engmode.xml` 是**跨项目共享文件**，不可直接用） |
| 7 | VCM/马达 **型号** | `qcom,actuator@0` 只有 `compatible = qcom,actuator` | sensormodule bin 命名（本机为 `qtech_imx766`，未见 ak7375 后缀） |
| 8 | **VOOCPHY 的 ADSP 固件与协议接口** | 充电协议栈跑在 ADSP（`oplus,adsp-voocphy`, `charger_pd`） | `oplus_chg` / adsp voocphy 内核源码 + ADSP 侧 |
| 9 | ZY0603 燃料计**寄存器映射** | 只有 dmesg 打了 `fg_zy0603_*` 函数名 | `oplus_chg` gauge 驱动 |
| 10 | **Pixelworks iris 配置不匹配** | `/proc/device-tree/soc/pxlw/` 下**只有** `pxlw,iris_cfg_samsung_amb670yf01_dsc_cmd`；martini 面板只有 `pxlw,soft-iris-enable` | iris 驱动源码确认「soft iris」含义与是否真需要 PX8568 |
| 11 | 各 CSIPHY / CCI **实际使用映射**（哪些真接 sensor） | DT 声明 6 路 csiphy，实际只用 4 路 | 相机 DTSI + CAMX 配置 |
| 12 | `msm_drm`/`camera.ko` 等模块的**编译配置**（.config） | 设备上无内核 config（`/proc/config.gz` 未验证/通常不存在） | 内核源码 `arch/arm64/configs/vendor/*` |
| 13 | 触摸 `platform_support_project = "QT QU"` 的编码含义 | 是 OPLUS 内部项目编码表 | OPLUS 触摸驱动源码的项目表 |
| 14 | boot 阶段 probe 日志（屏幕/触摸/相机） | ring buffer 已滚掉，无 pstore / last_kmsg / debugfs | 复现时抓 `dmesg -w` 或开启 `msm_rtb` |

---

## 8. 与「5.4 -> 5.10 可行性」相关的设备侧观察（**仅设备侧视角，不构成结论**）

> 本节只列**实测到的、与移植工作量直接相关**的事实；内核/厂商层/ABI 的最终判断属 task-1/2/3。

1. **本机已在跑自编译内核**：`5.4.295-qgki-gaaf8d3fac817-dirty`，编译器 clang 11.0.1 (r383902)，
   build 用户 `runner@runnervmtr4k5`，`-dirty` 后缀；`su` 是 KernelSU（`u:r:ksu:s0`），
   `ro.build.type=user` 但 `ro.debuggable=0` —— 说明**第三方内核已可正常启动到 UI 并 root**，
   这是「设备侧可刷」的正面证据。
2. **四个子系统的实现都深度绑定厂商驱动**：
   - 屏幕：面板时序/DSC/iris 全在 DT + `msm_drm.ko`（2.8 MB，模块化）
   - 触摸：`synaptics_tcm_oncell`（class 节点存在，驱动在树内）
   - 充电：`oplus_chg` + **ADSP 侧 VOOCPHY** + `pmic_glink`（`charger_pd` protection domain 在 adsp）
   - 摄像头：**`camera.ko` 单模块 9.6 MB**（refcount 34），依赖 CAMX 用户态 provider
3. **大量功能以 .ko 形式存在**：`camera.ko`, `msm_drm.ko`, `haptic.ko`, `aw8697.ko`,
   `uff_fp_driver.ko`, 数十个 `*_dlkm.ko`（音频）, `oplus_bsp_*.ko`；模块目录名直接叫 **`5.4-gki`**。
4. **DT 本身不含版本强耦合信息**，但它引用了大量 OPLUS 私有属性（`oplus,*`, `track,*`,
   `prevention,*`, `touchpanel,*`, `pxlw,*`, `oplus_bsp_tp_custom.*` cmdline），
   这些属性只有配套的 OPLUS 厂商驱动会解析。
5. **DT 里有明显的「多机型共用 + 运行时选择」结构**（默认面板是 9 Pro 的 1440x3168/3216，触摸节点名是项目 20031），
   说明这份 DT/驱动集合本身就是为「一树多机」设计的 —— 结构上有利于既有 5.10 树复用，但**每一条私有属性都要有对应驱动**才生效。

**综合（设备侧口径）**：证据支持「**有明确的技术方向**」，但**不支持「肯定可以」**——因为本机四大子系统的可用性几乎全部落在**按 5.4 QGKI 编译的厂商驱动/模块**上，设备侧并不提供任何「这些驱动在 5.10 下可复用」的证据。

---

## 附 A：本次采集的命令清单（可复现）

```
# 在线与身份
adb devices -l
adb -s 4647b81e shell "su -c id"
adb -s 4647b81e shell "su -c 'cat /proc/device-tree/model; echo; cat /proc/device-tree/compatible; echo'"
adb -s 4647b81e shell "su -c 'cat /proc/cmdline'"
adb -s 4647b81e shell "su -c 'cat /proc/version'"
adb -s 4647b81e shell "getprop"
adb -s 4647b81e shell "su -c 'cat /proc/device-tree/oppo,dtsi_no'"

# 屏幕
adb -s 4647b81e shell "su -c 'cat /proc/device-tree/soc/qcom,mdss_mdp@ae00000/qcom,mdss_dsi_samsung_ams662zs01_dvt_dsc_cmd/qcom,mdss-dsi-panel-name'"
adb -s 4647b81e shell "su -c 'cat /proc/device-tree/soc/qcom,dsi-display-primary/qcom,dsi-default-panel'"
adb -s 4647b81e shell "su -c 'cat /sys/class/drm/card0-DSI-1/modes'"
adb -s 4647b81e shell "su -c 'cat /sys/class/backlight/panel0-backlight/max_brightness'"

# 充电
adb -s 4647b81e shell "su -c 'cat /proc/device-tree/soc/qcom,pmic_glink/qcom,battery_charger/qcom,chg_ops'"
adb -s 4647b81e shell "su -c 'cat /proc/device-tree/soc/oplus,adsp-voocphy/compatible'"
adb -s 4647b81e shell "su -c 'dmesg | grep -E "ui_power_show|fg_zy0603"'"
adb -s 4647b81e shell "su -c 'cat /sys/class/power_supply/usb/usb_type'"

# 触摸
adb -s 4647b81e shell "su -c 'cat /proc/bus/input/devices'"
adb -s 4647b81e shell "su -c 'cat /sys/bus/i2c/devices/5-004b/name'"
adb -s 4647b81e shell "su -c 'cat /proc/device-tree/soc/i2c@990000/synaptics20031@4B/compatible'"
adb -s 4647b81e shell "su -c 'cat /proc/device-tree/soc/i2c@990000/st_fts@49/status'"

# 摄像头
adb -s 4647b81e shell "su -c 'ls /proc/device-tree/soc/qcom,cci0/ /proc/device-tree/soc/qcom,cci1/'"
adb -s 4647b81e shell "su -c 'ls /odm/lib64/camera/'"
adb -s 4647b81e shell "su -c 'cat /proc/device-tree/soc/qcom,cci0/martini,ois0/ois,name'"
adb -s 4647b81e shell "su -c 'dumpsys media.camera'"

# 通用
adb -s 4647b81e shell "su -c 'ls /vendor/lib/modules/'"
adb -s 4647b81e shell "su -c 'cat /proc/modules'"
adb -s 4647b81e shell "su -c 'ls /sys/class/'"
adb -s 4647b81e shell "su -c 'cat /proc/device-tree/__symbols__/*'"
```

---

## 附 B：本次采集明确**未获取**的项

| 项 | 状态 |
|---|---|
| 一加 9RT 各摄像头 I2C 地址 | **未获取**（DT 无 `reg`） |
| VCM 马达型号 | **未获取**（DT 无型号属性） |
| Synaptics 固件二进制路径 | **未获取**（`/vendor/firmware` 无匹配） |
| 面板峰值亮度的 nit 值 | **未获取**（仅有 QCOM 私有单位原值） |
| DSI 实际 Mbps/lane | **未获取**（phy-timings 为压缩编码） |
| MT2111（海外版）差异 | **未获取**（本机为 MT2110_CH） |
| boot 阶段 probe 日志 | **未获取**（ring buffer 已滚，无 pstore/debugfs） |
| 电池厂商（ATL/欣旺达等） | **未获取**（`/sys/class/power_supply/battery/model_name` 为空） |
| 无线充电硬件 | 实测**无**（`present=0 / online=0`） |
