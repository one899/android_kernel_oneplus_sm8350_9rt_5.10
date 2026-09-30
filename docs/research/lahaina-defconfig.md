# 5.10 lahaina (SM8350 / martini) vendor defconfig —— 来源考证 + 成品

> 任务: task-6 | 作者: defconfig-lahaina
> 交付物: `lahaina_GKI.config` / `lahaina_consolidate.config` / 本报告 (均在 `_meta/research/`)
> 铁律: 只写证据能支撑的东西; 没验证到的一律标 "未获取/待定", 不编造。

---

## 0. 结论先行 (TL;DR)

| 问题 | 答案 | 证据强度 |
| --- | --- | --- |
| CLO 5.10 树里有现成的 lahaina vendor config 吗? | **没有。** CLO 全站只有一个 5.10 内核仓库 `clo/la/kernel/msm-5.10` [id=29371], 它的 `arch/arm64/configs/vendor/` 只有 waipio / parrot / anorak / neo, 没有任何 lahaina `.config` | 实测 (3 个分支 × 1274 个分支名全扫) |
| 那 CLO 5.10 里跟 lahaina 有关的东西是什么? | 只有两个**残留 stub**: `build.config.msm.lahaina` 和 `modules.list.msm.lahaina`。后者还列了一个 5.10 树里已无源码的模块 `hh_virt_wdt.ko` (Haven), 说明这个 lahaina 目标是从 5.4 抄来后没人维护 | 实测 (文件内容 + 源码树比对) |
| OPLUS 的 5.10 树呢? | 一样没有配置。它的 `build.config.msm.lahaina` / `modules.list.msm.lahaina` 与 CLO c27 **逐字节完全相同** (SHA256 前 16 位相同), 即原样带入、从未适配 | 实测 (哈希) |
| 那 lahaina 驱动在 5.10 树里齐不齐? | **齐。** clk / icc / pinctrl / UFS-PHY / LLCC 数据都在, Kconfig 符号都在 (见 §2) | 实测 (Kconfig / Makefile / 源文件) |
| 所以本任务产出 | 从 5.10 树自带的 `waipio_GKI.config` 改出 `lahaina_GKI.config`: **新增 13 条 / 删除 31 条**, 逐条有据 (见 §3) | — |

**一句话**: 高通没给 5.10 的 lahaina 配置, 但 5.10 树的 lahaina 驱动是齐的; 所以这份配置是 "拿同一棵树里的 waipio 配置做 SoC 替换" 得到的, 而不是从 5.4 搬配置。

---

## 1. 取证: CLO 5.10 到底有没有 lahaina config

### 1.1 先找到仓库

```text
# CLO 直连, 不挂代理
curl -s 'https://git.codelinaro.org/api/v4/projects?search=msm-5.10&per_page=100&simple=true'
  -> 29371|clo/la/kernel/msm-5.10|aosp-new/caf2/caf2/clo/main

# 全站 search=msm-5.10 只有一个结果
#   clo/le/kernel/msm-5.10  -> 404
#   clo/la/kernel/msm-5.15  -> 200 (5.15 世代, 不相关)
# 分支: search=KERNEL.PLATFORM -> KERNEL.PLATFORM.1.0.c25 / c26 / c27
```

注意: 按任务卡给的 `search=msm-kernel` 只能搜到 `securemsm-kernel`。CLO 上这个 5.10 内核仓库的路径是 `clo/la/kernel/msm-5.10` (不是 `kernel_platform/kernel/msm-kernel`)。

### 1.2 列 vendor 目录 (c25 / c26 / c27 三个分支清单完全一致)

```text
GET /projects/29371/repository/tree?path=arch/arm64/configs/vendor&ref=KERNEL.PLATFORM.1.0.c27

blob .gitignore           blob parrot_GKI.config          blob waipio_GKI.config
blob anorak.config        blob parrot_consolidate.config  blob waipio_consolidate.config
blob anorak_debug.config  blob waipio_le.config           blob waipio_tuivm.config
blob neo_la.config        blob waipio_le_debug.config     blob waipio_tuivm_debug.config
blob neo_la_debug.config
blob neo_le.config
blob neo_le_debug.config

-> 没有 lahaina
```

`kernel/configs/vendor/` 也只有 android-base / android-recommended / kvm_guest / nopm / tiny / xen, 无 lahaina 片段。

### 1.3 排除"藏在别的分支名里"

```text
# 把该仓库 1274 个分支全部拉下来过滤
for page in 1..13: GET /repository/branches?per_page=100&page=$page
$all | Where-Object { $_.name -match '(?i)lahaina|8350|martini' }
  -> 0 命中
```

### 1.4 CLO 5.10 里确实存在的两个 lahaina 文件 (全文贴出)

`build.config.msm.lahaina` (881 B, id=29371 @ KERNEL.PLATFORM.1.0.c27):

```sh
################################################################################
## Inheriting configs from ACK
. ${ROOT_DIR}/common/build.config.common
. ${ROOT_DIR}/common/build.config.aarch64

################################################################################
## Variant setup
MSM_ARCH=lahaina
VARIANTS=(gki)
[ -z "${VARIANT}" ] && VARIANT=gki

if [ -e "${ROOT_DIR}/msm-kernel" -a "${KERNEL_DIR}" = "common" ]; then
	KERNEL_DIR="msm-kernel"
fi

DT_OVERLAY_SUPPORT=1

BOOT_IMAGE_HEADER_VERSION=3
BASE_ADDRESS=0x80000000
PAGE_SIZE=4096

if [ "${KERNEL_CMDLINE_CONSOLE_AUTO}" != "0" ]; then
	KERNEL_VENDOR_CMDLINE+=' console=ttyMSM0,115200n8 earlycon=msm_geni_serial,0x0098C000'
fi

################################################################################
## Inheriting MSM configs
. ${KERNEL_DIR}/build.config.msm.common
. ${KERNEL_DIR}/build.config.msm.gki
```

`modules.list.msm.lahaina` (1510 B):

```text
qcom_cpu_vendor_hooks.ko
proxy-consumer.ko
fixed.ko
cmd-db.ko
rpmhpd.ko
qcom-scm.ko
qcom_rpmh.ko
rpmsg_char.ko
rpmsg_core.ko
qcom_pm8008-regulator.ko
rpmh-regulator.ko
refgen.ko
stub-regulator.ko
clk-dummy.ko
clk-qcom.ko
clk-aop-qmp.ko
clk-rpmh.ko
gdsc-regulator.ko
gcc-lahaina.ko
dispcc-lahaina.ko
qnoc-lahaina.ko
icc-bcm-voter.ko
pinctrl-msm.ko
pinctrl-lahaina.ko
qcom-scm.ko
qcom-pdc.ko
iommu-logger.ko
arm_smmu.ko
qcom_iommu_util.ko
iommu-logger.ko
msm_dma_iommu_mapping.ko
qcom_dma_heaps.ko
mem_buf.ko
mem_buf_dev.ko
qcom-arm-smmu-mod.ko
msm-geni-se.ko
msm_geni_serial.ko
phy-qcom-ufs.ko
phy-qcom-ufs-qmp-v4-lahaina.ko
ufshcd-crypto-qti.ko
crypto-qti-common.ko
crypto-qti-hwkm.ko
hwkm.ko
ufs_qcom.ko
qbt_handler.ko
qcom_hwspinlock.ko
smem.ko
socinfo.ko
eud.ko
dwc3.ko
dwc3-msm.ko
roles.ko
phy-generic.ko
phy-msm-snps-hs.ko
phy-msm-ssusb-qmp.ko
secure_buffer.ko
usb_f_gsi.ko
ipa_fmwk.ko
usb_f_mass_storage.ko
usb_f_diag.ko
usb_f_ccid.ko
usb_f_cdev.ko
usb_f_qdss.ko
sps_drv.ko
usb_bam.ko
typec.ko
soc_sleep_stats.ko
sys_pm_vx.ko
secure_buffer.ko
spmi-pmic-arb.ko
regmap-spmi.ko
qcom-spmi-pmic.ko
qpnp-power-on.ko
nvmem_qcom-spmi-sdam.ko
msm-poweroff.ko
qcom_watchdog.ko
msm_qmp.ko
qcom_ipc_logging.ko
qcom_ipcc.ko
cqhci.ko
sdhci-msm.ko
memory_dump_v2.ko
llcc-qcom.ko
qcom_edac.ko
kryo_arm64_edac.ko
qcom-cpufreq-hw.ko
sched-walt.ko
msm_performance.ko
msm_rtb.ko
qcom-dload-mode.ko
qcom-reboot-reason.ko
hh_virt_wdt.ko
qcom_wdt_core.ko
msm_kgsl.ko
governor_msm_adreno_tz.ko
governor_gpubw_mon.ko
minidump.ko
```

### 1.5 这两个文件是残留 stub —— 两条独立证据

**证据 1: `hh_virt_wdt.ko` 在 5.10 树里没有源码。**
NaN Haven 在 5.10 被 Gunyah 取代了, 而这张清单还挂着 `hh_virt_wdt.ko`, 且**没有** 5.10 实际在用的 `gh_*.ko`。 对比 CLO 自己的 `modules.list.msm.waipio` 列的是 `gh_virt_wdt.ko` / `gh_ctrl.ko` / `gh_rm_drv.ko` … ⇒ lahaina 这张清单是 5.4 时代的遗留。

**证据 2: OPLUS 的 5.10 树原样照抄。** 实测哈希:

```text
OPLUS  one899/android_kernel_oneplus_sm8450_5.10_martini @ oneplus/sm8450_b_16.0_oneplus_10_pro
  build.config.msm.lahaina   SHA256 235493897959DDF9...   881 B
  modules.list.msm.lahaina   SHA256 BABA9F87FD3EE23B...  1510 B
CLO    clo/la/kernel/msm-5.10 @ KERNEL.PLATFORM.1.0.c27
  build.config.msm.lahaina   SHA256 235493897959DDF9...   881 B
  modules.list.msm.lahaina   SHA256 BABA9F87FD3EE23B...  1510 B
=> 逐字节相同
```

**结论: 公开世界里没有任何 5.10 的 lahaina vendor defconfig, 只能自己造。**

---

## 2. 5.10 树里 lahaina 的家底 (可用性盘点)

| 硬件块 | 5.10 树里的源文件 | Kconfig 符号 | 本配置里 |
| --- | --- | --- | --- |
| SoC 选择 | `arch/arm64/Kconfig.platforms:204` | `ARCH_LAHAINA` | `=y` |
| 全局时钟 | `drivers/clk/qcom/gcc-lahaina.c` (Makefile:55) | `MSM_GCC_LAHAINA` | `=m` → gcc-lahaina.ko |
| 显示时钟 | `dispcc-lahaina.c` (Makefile:40) | `MSM_DISPCC_LAHAINA` | `=m` |
| 相机时钟 | `camcc-lahaina.c` (Makefile:57) | `MSM_CAMCC_LAHAINA` | `=m` |
| GPU 时钟 | `gpucc-lahaina.c` (Makefile:56) | `MSM_GPUCC_LAHAINA` | `=m` |
| 视频时钟 | `videocc-lahaina.c` (Makefile:58) | `MSM_VIDEOCC_LAHAINA` | `=m` |
| 调试时钟 | `debugcc-lahaina.c` (Makefile:39) | `MSM_DEBUGCC_LAHAINA` | `=m` |
| AOP 时钟 | `drivers/clk/qcom/clk-aop-qmp.c` | `MSM_CLK_AOP_QMP` (dep: COMMON_CLK_QCOM && MSM_QMP) | `=m` → clk-aop-qmp.ko |
| NoC / 互连 | `drivers/interconnect/qcom/lahaina.c` (Makefile:37 `qnoc-lahaina-objs`) | `INTERCONNECT_QCOM_LAHAINA` | `=m` → qnoc-lahaina.ko |
| EPSS L3 | 同上 | `INTERCONNECT_QCOM_EPSS_L3` | `=m` |
| Pin 控制 | `drivers/pinctrl/qcom/pinctrl-lahaina.c` (Makefile:4) | `PINCTRL_LAHAINA` | `=m` |
| UFS PHY | `drivers/phy/qualcomm/phy-qcom-ufs-qmp-v4-lahaina.c` (Makefile:10) | **无独立符号**; 与 waipio/diwali/cape/parrot/anarok 的 v4 PHY 共用 `PHY_QCOM_UFS_V4` | 继承 `=m` |
| LLCC | `drivers/soc/qcom/llcc_qcom.c` **已含** `lahaina_data[]` / `lahaina_cfg` / `compatible = "qcom,lahaina-llcc"` (193 / 427 / 1392 行) | `QCOM_LLCC` | `=m` → llcc-qcom.ko |
| 电源域 | `rpmhpd.c` | `QCOM_RPMHPD` | `=m` (新增) |
| 参考电压 | `drivers/regulator/refgen.c` | `REGULATOR_REFGEN` | `=m` (新增) |

⚠️ 两个容易踩的坑:

1. **5.4 的 `CONFIG_QCOM_LAHAINA_LLCC` 在 5.10 不存在**, 别照抄 5.4。 5.10 把 lahaina 的 slice 数据合并进 `llcc_qcom.c` 了, 用 `CONFIG_QCOM_LLCC` 即可。 (5.10 树里也确实**没有** `llcc-lahaina.c`, 只有 5.4 有。)
2. **UFS PHY 没有 lahaina 专属符号。** `PHY_QCOM_UFS_V4` 一行同时编 lahaina / waipio / diwali / cape / parrot / anarok 六个源文件, `modules.list.msm.lahaina` 要的 `phy-qcom-ufs-qmp-v4-lahaina.ko` 就是这个模块。
---

## 3. `lahaina_GKI.config` 是怎么改出来的 (逐条)

**基线 (逐行照抄, 不发明内容)**:
`one899/android_kernel_oneplus_sm8450_5.10_martini @ oneplus/sm8450_b_16.0_oneplus_10_pro`
`arch/arm64/configs/vendor/waipio_GKI.config` —— 742 行 / 20433 B。

> 为什么用 OPLUS 的 waipio 而不是 CLO 的 waipio? OPLUS 版 = CLO 版 + ~9 KB OPLUS 厂商/机型选项 (20433 B vs CLO 11151 B)。
> 目标是一份**能直接扔进这棵树、保留 OPLUS 特性**的配置, 所以用 OPLUS 版做基线, 只动 SoC 身份相关的行。

### 3.1 新增 13 条 (全部在 5.10 树验证过符号存在)

| # | 新增 | 为什么 |
| --- | --- | --- |
| 1 | `CONFIG_ARCH_LAHAINA=y` | 目标 SoC。Kconfig.platforms:204, `bool`, 依赖 `ARCH_QCOM` 已由 `gki_defconfig:52 CONFIG_ARCH_QCOM=y` 满足 |
| 2-7 | `CONFIG_MSM_{GCC,DISPCC,CAMCC,GPUCC,VIDEOCC,DEBUGCC}_LAHAINA=m` | 6 个 lahaina 时钟控制器, 与 5.4 OPLUS `lahaina_GKI.config` 的集合一一对应 |
| 8 | `CONFIG_INTERCONNECT_QCOM_LAHAINA=m` | NoC (qnoc-lahaina.ko)。5.4 真机是 `=y`, 但 5.10 基线体系一律 `=m`, 遵循基线 |
| 9 | `CONFIG_INTERCONNECT_QCOM_EPSS_L3=m` | lahaina EPSS L3 provider; 5.4 真机 `=y`, 5.10 用 `=m` |
| 10 | `CONFIG_PINCTRL_LAHAINA=m` | TLMM pin 控制 |
| 11 | `CONFIG_MSM_CLK_AOP_QMP=m` | `modules.list.msm.lahaina` 点名要 `clk-aop-qmp.ko`; 基线缺这条 ⇒ 不补就少模块 |
| 12 | `CONFIG_QCOM_RPMHPD=m` | `modules.list.msm.lahaina` 点名要 `rpmhpd.ko`; 5.4 真机也有 |
| 13 | `CONFIG_REGULATOR_REFGEN=m` | `modules.list.msm.lahaina` 点名要 `refgen.ko`; 5.4 真机也有 |

### 3.2 删除 31 条

**(a) SoC 身份替换 (21 条: 删 waipio / diwali / cape / parrot)**

| 删除 | 原因 |
| --- | --- |
| `CONFIG_ARCH_CAPE=y` `CONFIG_ARCH_DIWALI=y` `CONFIG_ARCH_WAIPIO=y` | 换成 `CONFIG_ARCH_LAHAINA=y`。5.4 真机 `_meta/9rt_defconfig` 只有 `ARCH_LAHAINA` (外加 SHIMA/YUPIK), **没有** waipio/diwali/cape |
| `CONFIG_MSM_{GCC,DISPCC,CAMCC,GPUCC,VIDEOCC,DEBUGCC}_WAIPIO=m` (6) | waipio 的时钟控制器对 lahaina 无用 |
| `CONFIG_INTERCONNECT_QCOM_{WAIPIO,DIWALI,PARROT}=m` (3) | 换 `INTERCONNECT_QCOM_LAHAINA` + `EPSS_L3` |
| `CONFIG_PINCTRL_{WAIPIO,DIWALI,CAPE}=m` (3) | 换 `PINCTRL_LAHAINA` |
| `CONFIG_SM_{GCC,GPUCC,DISPCC,VIDEOCC,CAMCC,DEBUGCC}_DIWALI=m` (6) | 整个 diwali 平台移除 |

**(b) waipio 专属子系统 (8 条)** —— 三条独立证据同时指向 "lahaina 没有": ① 5.4 真机 `_meta/9rt_defconfig` 没有; ② CLO `modules.list.msm.lahaina` 没有; ③ CLO `modules.list.msm.waipio` 有。

| 删除 | 判断依据 |
| --- | --- |
| `CONFIG_MSM_TMECOM_QMP=m` | waipio 有 `tmecom-intf.ko`, lahaina 清单没有 |
| `CONFIG_EP_PCIE=m` | waipio/5.10 专属 EP-PCIe 端点; lahaina 清单无 |
| `CONFIG_QCOM_SPSS=m` + `CONFIG_QCOM_SPSS_AC_RESTRICTION=y` | 5.4 真机两条都不存在; lahaina 清单无 spss |
| `CONFIG_MSM_SYSSTATS=m` | 同上 |
| `CONFIG_QCOM_AOSS_QMP=m` | AOSS 是 waipio 世代的常开子系统; 5.4 真机无 |
| `CONFIG_QCOM_DCVS=m` `CONFIG_QCOM_DCVS_FP=m` | 5.4 真机无; lahaina 清单无 `qcom-dcvs.ko` / `dcvs_fp.ko` |

**(c) OPLUS 厂商层里点名 SM8450 的 2 条 (删除 + 文件末尾留 TODO)**

| 删除 | 原因 |
| --- | --- |
| `CONFIG_OPLUS_SM8450_CHARGER=y` | martini 是 SM8350; 5.4 真机用的是 `CONFIG_OPLUS_SM8350_CHARGER=y`。**但** 5.10 modules 仓库里没能定位到 SM8350 的符号定义 (见 §5 第 1 行), 所以不能瞎写 |
| `CONFIG_OPLUS_ADSP_SM8450_CHARGER=y` | 同上 (`OPLUS_ADSP_SM8450_CHARGER` 在 modules 仓库 `charger/v2/Kconfig:143` 有定义, SM8350 对应符号未找到) |

### 3.3 保留不动的 waipio 名字 (及理由)

- **Gunyah (`CONFIG_GUNYAH_DRIVERS=y` + `GH_*`) 保留**: 5.4 的 martini 用的是 Haven (`CONFIG_HAVEN_DRIVERS=y` / `CONFIG_HH_VIRT_WATCHDOG=y`), 但 **5.10 树里 Haven 源码已经没了, 只有 Gunyah**。OPLUS 5.10 树上唯一可编译的虚拟化方案就是 Gunyah。 (副作用: `modules.list.msm.lahaina` 里的 `hh_virt_wdt.ko` 在这棵树上永远编不出来, 清单必须改 — 见 §6.3。)
- **机型级器件选项保留**: `CONFIG_I2C_RTC6226_QCA` / `CONFIG_LEDS_AW210XX` / `CONFIG_CSA37F71_SENSOR_NDT` / `CONFIG_OPLUS_{MP2762,SGM41512,DA9313,SC8517,WL2868C}_*` 等是 OnePlus 10 Pro 的板级器件, 对 martini 属于多余项, 但都是 `=m` 且只在对应 I2C/SPI 设备存在时 probe, **不会编不过**、不阻塞启动 —— 留给 "按 martini 物料表裁剪" 这一步 (§5)。
- **`CONFIG_THERMAL_TSENS` 不采用**: 5.4 真机是 `CONFIG_THERMAL_TSENS=y` + `# CONFIG_QCOM_TSENS is not set`; 但 **5.10 `drivers/thermal/qcom/Kconfig` 里已经没有 `THERMAL_TSENS` 这个符号** (只有 `QCOM_TSENS`), 所以沿用基线的 `CONFIG_QCOM_TSENS=m`。这正是"不能直接搬 5.4 配置"的典型例子。

### 3.4 成品核对

```text
D:\295\_meta\research\lahaina_GKI.config
  751 行 / 21824 B
  SHA256 EA42F64F0B562EF81526A7E6DA82CDB231A2A6A866BF13765507D45E504B2678
  残留 waipio/diwali/cape/parrot 的 config 行: 0 (注释除外)
  13 条新增全部就位, 仍在字母序区间内, 便于下次 rebase
```

---

## 4. `lahaina_consolidate.config`

基线: 5.10 树 `arch/arm64/configs/vendor/waipio_consolidate.config` (39 行 / 1113 B) 逐行照抄, **唯一差异: 删掉 `CONFIG_QCOM_SPSS_AC_RESTRICTION=y`** (GKI 片段里 `CONFIG_QCOM_SPSS=m` 已删, 留着是悬空配置)。文件头写了来源与差异说明, 共 48 行。

装配顺序 (来自 `build.config.msm.gki` 的 `build_defconfig_fragments`):

```text
VARIANT=gki         -> gki_defconfig + vendor/lahaina_GKI.config
VARIANT=consolidate -> gki_defconfig + vendor/lahaina_GKI.config + consolidate.fragment + vendor/lahaina_consolidate.config
```
---

## 5. 哪些必须等 martini 硬件信息才能定

| # | 待定项 | 现状 | 需要什么才能定 |
| --- | --- | --- | --- |
| 1 | **OPLUS 充电** `CONFIG_OPLUS_SM8350_CHARGER` | 基线用的是 SM8450 的名字, 已删除并在文件末尾留 TODO。5.10 modules 仓库里确实存在 `vendor/oplus/kernel/charger/charger_ic/oplus_battery_sm8350.c`, 但我在 `charger/Kconfig` / `charger/v2/Kconfig` / `charger/config/Kconfig` 以及 GitHub code search (`OPLUS_SM8350_CHARGER`, total=0) 里都**没找到该符号的定义** | 克隆 modules 仓库后 `grep -rn 'SM8350_CHARGER' vendor/oplus/kernel/charger/` 确认真实符号名; 或从 martini 5.4 的 `.config` 反查它由哪个 Kconfig 定义 |
| 2 | **触控 IC** | 基线同时开了 Synaptics TCM S3910 / Goodix GT9966 / Focal FT3658U / `TOUCHPANEL_CUSTOM` | martini 物料表, 或 5.4 实机 dmesg 里实际 probe 的 TP 驱动名 |
| 3 | **WLAN/BT 走哪一路** `ICNSS2` vs `CNSS2` | 基线两套都编 (`=m`); 5.4 真机也是 `CNSS2=y` + `ICNSS2=y` 同时存在, 看不出唯一答案 | martini 的 WLAN 模块型号 + firmware 路径, 或 5.4 dmesg 里实际 probe 的是哪一个 |
| 4 | **板级器件裁剪** (充电 IC / gauge / NFC / 传感器 / RTC) | 基线里是 10 Pro 的 (MP2762 / SC8517 / SGM41512 / DA9313 / WL2868C / RTC6226 / AW210XX / CSA37F71) | martini 物料表。不裁剪也能启动, 只是多编几个不会被 probe 的模块 |
| 5 | **DTS / DTBO** | 不在本任务范围, 但必须提醒: 5.10 modules 仓库里**只有高通参考板的 lahaina DTS** (`lahaina-mtp/cdp/qrd/hdk/atp*.dts[i]`), **没有 martini 的 DTS** (全树 grep `martini` = 0 命中)。5.4 上它在 `DTB_DIR=vendor/oplus/martini` | 5.4 的 `vendor/oplus/martini/*.dts*` 必须前移到 5.10 |
| 6 | **`modules.list.msm.lahaina`** | 见 §6.3; 现有的是 CLO 残留 (含 Haven 模块), 必须重做 | OPLUS 5.10 的 waipio 清单 + lahaina SoC 模块名替换 |

**不在待定范围 (已定)**: SoC 选择 / 6 个时钟控制器 / NoC / pinctrl / UFS PHY / LLCC / RPMHPD / REFGEN / AOP 时钟 —— 这些符号和驱动在 5.10 树里都齐, 见 §2。

---

## 6. 确切构建命令

### 6.1 目录布局 (AOSP kernel_platform 标准)

```text
<work>/kernel_platform/
  build/        <- one899/android_kernel_modules_and_devicetree_sm8450_5.10_martini
                   @ oneplus/sm8450_b_16.0_oneplus_10_pro   (即 kernel_platform/build)
  oplus/  qcom/ <- 同一个仓库的其余部分
  common/       <- one899/android_kernel_common_oneplus_sm8450_5.10_martini
                   @ oneplus/sm8450_b_16.0_oneplus_10_pro   (ACK/GKI 树)
  msm-kernel/   <- one899/android_kernel_oneplus_sm8450_5.10_martini
                   @ oneplus/sm8450_b_16.0_oneplus_10_pro   (厂商内核树)
                     arch/arm64/configs/vendor/lahaina_GKI.config          <= 本产物放这
                     arch/arm64/configs/vendor/lahaina_consolidate.config  <= 本产物放这
```

`build.config.msm.lahaina` 自己会选内核目录:

```sh
MSM_ARCH=lahaina ; VARIANTS=(gki)
if [ -e "${ROOT_DIR}/msm-kernel" -a "${KERNEL_DIR}" = "common" ]; then KERNEL_DIR="msm-kernel"; fi
```

`build.config.msm.common:15-16` 再推出 `CONFIG_TARGET=msm.${MSM_ARCH}` ⇒ `msm.lahaina` ⇒ 自动用 `msm-kernel/modules.list.msm.lahaina`。

### 6.2 构建 (走 Google 标准层 `build/build.sh`, 独立编, 不需要 AOSP `lunch`)

```bash
# 0) 放文件
cp lahaina_GKI.config         <work>/kernel_platform/msm-kernel/arch/arm64/configs/vendor/
cp lahaina_consolidate.config <work>/kernel_platform/msm-kernel/arch/arm64/configs/vendor/

# 1) GKI 变体
cd <work>/kernel_platform
export ROOT_DIR=$PWD
BUILD_CONFIG=msm-kernel/build.config.msm.lahaina VARIANT=gki ./build/build.sh

# 2) consolidate 变体
BUILD_CONFIG=msm-kernel/build.config.msm.lahaina VARIANT=consolidate ./build/build.sh
```

生成出来的 defconfig 在 `out/msm-kernel/lahaina-gki/.config` (VARIANT=gki), 可以先只看这一步的结果再决定是否全量编。

❌ **不要用** `kernel_platform/oplus/build/oplus_build.sh` / `oplus_build_kernel.sh`: 它们内部调 `build/android/prepare_vendor.sh`, 头部注释写明 "Script assumes running after lunch w/Android build environment variables available", 而且 `oplus_build_kernel.sh` 里 `OPLUS_VND_BUILD_PLATFORM=SM8450` 是写死的。 (`oplus_build.sh` 的形态: `./kernel_platform/oplus/build/oplus_build.sh <virtants_platform> <virtants_type> <LTO> <target_type> <REPACK_IMG>`。)

### 6.3 `modules.list.msm.lahaina` 必须一起改 (否则编出来也不全)

现在的 `msm-kernel/modules.list.msm.lahaina` 是 CLO 残留 (97 行, 与 CLO 逐字节相同), 两个问题: ① 列了 5.10 树里没有源码的 `hh_virt_wdt.ko`; ② 没有 OPLUS 侧模块 (`oplus_*.ko` 等), 而 OPLUS 自己的 `modules.list.msm.waipio` 有 152 行。

建议: 拿 OPLUS 的 `modules.list.msm.waipio` 为底, 做如下替换:

```text
改名 5 条:
  gcc-waipio.ko                 -> gcc-lahaina.ko
  dispcc-waipio.ko              -> dispcc-lahaina.ko
  pinctrl-waipio.ko             -> pinctrl-lahaina.ko
  qnoc-waipio.ko                -> qnoc-lahaina.ko
  phy-qcom-ufs-qmp-v4-waipio.ko -> phy-qcom-ufs-qmp-v4-lahaina.ko
删掉 waipio/diwali/cape 专属 8 条:
  dispcc-diwali.ko gcc-diwali.ko pinctrl-cape.ko pinctrl-diwali.ko qnoc-diwali.ko
  phy-qcom-ufs-qmp-v4-diwali.ko phy-qcom-ufs-qmp-v4-cape.ko tmecom-intf.ko
补上 lahaina 侧新增 (对应本配置新增的 =m):
  camcc-lahaina.ko gpucc-lahaina.ko videocc-lahaina.ko debugcc-lahaina.ko
  clk-aop-qmp.ko refgen.ko rpmhpd.ko
保留: gh_*.ko (本配置保留了 Gunyah)
同步删掉: qcom_aoss.ko qcom-dcvs.ko dcvs_fp.ko (对应 §3.2(b))
```

### 6.4 ⚠️ 这棵树把 config warning 当 error

`build.config.msm.common:120-128`:

```sh
if grep -q -e "warning:" $output; then
  echo "ERROR! Treating config warnings as errors"; ... exit 1
fi
```

**这就是为什么配置里没有塞任何未验证的符号** (比如充电那两条宁可留成注释): 一个树里不存在的 `CONFIG_X=y` 在 kconfig 阶段就可能带出 warning, 直接让构建失败。 想放开只能用 `OPLUS_BUILD_DO=true`。

---

## 7. 我没做的事 / 残余风险

1. **没有真正编译过。** 本地没有这套 5.10 源码的副本 (仓库在 GitHub 上, 工作目录里是脚本与证据文件), 所以所有 "符号存在" 结论都来自 Kconfig / Makefile / 源文件原文, 不是 "编过了"。**第一次真编译才是最终验证。**
2. **§3.2(b) 组 8 条属于 "精简" 而非 "必须"**: 它们都是 `=m` 的独立驱动, 留着也能编过 (只是多几个不会被 probe 的模块)。 想更保守就把这 8 条加回来, 不影响 lahaina 正常工作。
3. **没有验证 OPLUS 厂商层在 lahaina 上能否全编过** (那是另一个任务: 3 个 symlink 目录 `oplus_perf_sched` / `oem_sched` / `tuning` 的可平移性)。
4. **`ARCH_SHIMA` / `ARCH_YUPIK`** 在 5.4 真机配置里是开着的 (多 SoC 合一编译); 本配置按 "lahaina 单目标" 处理, 没开。 若后续要跟 5.4 一样多 SoC 合一, 把 5.10 树里存在的 `ARCH_SHIMA` 等加回来即可。

---

## 附录 A. 参考文件与出处

| 文件 | 出处 | 用途 |
| --- | --- | --- |
| `waipio_GKI.config` (742 行 / 20433 B) | one899/android_kernel_oneplus_sm8450_5.10_martini @ oneplus/sm8450_b_16.0_oneplus_10_pro, `arch/arm64/configs/vendor/` | **本产物基线** |
| `waipio_consolidate.config` (39 行 / 1113 B) | 同上 | consolidate 基线 |
| `waipio_GKI.config` (399 行 / 11151 B) | CLO clo/la/kernel/msm-5.10 @ KERNEL.PLATFORM.1.0.c27 | 对照, 证明 OPLUS 版多约 9 KB 厂商选项 |
| `lahaina_GKI.config` (515 行) / `lahaina_QGKI.config` (817 行) / `lahaina_consolidate.config` (74 行) / `lahaina_debug.config` (141 行) | one899/android_kernel_oneplus_sm8350_9rt_295bpf @ oneplus/sm8350_b_16.0.0_oneplus9rt (5.4) | SoC / 机型选项对照 |
| `build.config.martini` (10 行) + `build.config.msm.lahaina` (42 行) | 同上 (5.4) | 5.4 的 lahaina 构建目标 (BRANCH=msm-5.4, VARIANTS=(qgki-debug qgki-consolidate qgki gki gki-only), `DTB_DIR=vendor/oplus/martini`) |
| `_meta/9rt_defconfig` (7165 行) | 9RT 真机 5.4 `.config` | **判断 "lahaina 有没有某子系统" 的最强单点证据** |
| `modules.list.msm.{lahaina,waipio}` | CLO 29371 c27 (waipio 另有 OPLUS 版 152 行) | 模块↔config 对应关系; 判断 waipio 专属项 |
| `llcc_qcom.c` / `Kconfig.platforms` / `clk,icc,pinctrl,phy,thermal 的 Kconfig+Makefile` / `build.config.msm.{common,gki}` | 5.10 厂商树 | 符号存在性与依赖验证 |

## 附录 B. 全部取证命令 (可直接重跑)

```bash
# --- CLO (直连, 不挂代理) ---
curl -s 'https://git.codelinaro.org/api/v4/projects?search=msm-5.10&per_page=100&simple=true'
curl -s 'https://git.codelinaro.org/api/v4/projects/29371/repository/branches?per_page=100&search=KERNEL.PLATFORM'
for b in KERNEL.PLATFORM.1.0.c25 KERNEL.PLATFORM.1.0.c26 KERNEL.PLATFORM.1.0.c27; do
  curl -s "https://git.codelinaro.org/api/v4/projects/29371/repository/tree?path=arch/arm64/configs/vendor&ref=$b&per_page=100"; done
curl -s 'https://git.codelinaro.org/api/v4/projects/29371/repository/files/build.config.msm.lahaina/raw?ref=KERNEL.PLATFORM.1.0.c27'
curl -s 'https://git.codelinaro.org/api/v4/projects/29371/repository/files/modules.list.msm.lahaina/raw?ref=KERNEL.PLATFORM.1.0.c27'
curl -s 'https://git.codelinaro.org/api/v4/projects/29371/repository/tree?path=drivers/virt&ref=KERNEL.PLATFORM.1.0.c27'

# --- GitHub (必须走代理 http://127.0.0.1:18888) ---
curl -x http://127.0.0.1:18888 -s -H "Authorization: token $TOKEN" \
  'https://api.github.com/repos/one899/android_kernel_oneplus_sm8450_5.10_martini/git/trees/oneplus%2Fsm8450_b_16.0_oneplus_10_pro?recursive=1' | grep -i lahaina
curl -x http://127.0.0.1:18888 -s 'https://raw.githubusercontent.com/one899/android_kernel_oneplus_sm8350_9rt_295bpf/oneplus/sm8350_b_16.0.0_oneplus9rt/arch/arm64/configs/vendor/lahaina_GKI.config'
# OPLUS vs CLO 的 lahaina 目标文件哈希比对
curl -x http://127.0.0.1:18888 -s -o oplus_bc 'https://raw.githubusercontent.com/one899/android_kernel_oneplus_sm8450_5.10_martini/oneplus/sm8450_b_16.0_oneplus_10_pro/build.config.msm.lahaina'
curl -s -o clo_bc 'https://git.codelinaro.org/api/v4/projects/29371/repository/files/build.config.msm.lahaina/raw?ref=KERNEL.PLATFORM.1.0.c27'
sha256sum oplus_bc clo_bc   # 相同
```

## 附录 C. 成品文件校验和

```text
lahaina_GKI.config          751 行 / 21824 B
  SHA256 EA42F64F0B562EF81526A7E6DA82CDB231A2A6A866BF13765507D45E504B2678
lahaina_consolidate.config   48 行
  由 waipio_consolidate.config 删 1 行 (QCOM_SPSS_AC_RESTRICTION) + 头部来源说明生成
```
