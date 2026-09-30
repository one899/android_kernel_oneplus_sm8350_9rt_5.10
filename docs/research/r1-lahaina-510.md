# R1 —— SM8350 / lahaina 是否存在 5.10+ 内核（穷举核实）

> 研究者：lahaina-510-scout（独立研究线 1） · 任务：task-1
> 取证时间：本次会话实测（所有 URL 均为当场抓取，非记忆）
> 取证工具：curl.exe 直连 + GitHub raw / GitHub API / CodeLinaro GitLab API / git ls-remote
> **本文严格区分「已核实」与「推测」：第 ⑦ 节把两者分开列出。找不到证据一律写「未核实 / 未找到」。**

---

## ① 结论摘要

**结论：截至本次穷举，没有找到任何一款 SM8350/lahaina 手机「出厂搭载或公开了 5.10 私有内核源码」。**
所有能拿到源码的 SM8350 机型（小米 11 系列 / Redmi K40 Pro / 一加 9 系列 / Realme GT·GT2 / 三星 S21·S21 FE(骁龙) / 索尼 Xperia 1 III·5 III / ROG Phone 5 / ZTE Axon 30 Pro / 红魔 6 / Surface Duo 2 …）
以及所有社区内核仓库（LineageOS / crDroid / arter97 / QRD-Development），**全部停在 5.4.x**（最高实测 5.4.302）。

**但有两条必须写清楚的「但是」，它们会改变方案的措辞（不是推翻结论，是修正边界）：**

1. **SM8350 确实存在于高通公开的 msm-5.10 树里，而且是真实目标不是占位**
   `arch/arm64/Kconfig.platforms` 有 `config ARCH_LAHAINA`；`build.config.msm.lahaina`、`modules.list.msm.lahaina` 是实打实的构建目标与模块清单；`drivers/clk/qcom/gcc-lahaina.c`(117 KB)、`dispcc-lahaina.c`、`pinctrl-lahaina.c`、`interconnect/qcom/lahaina.c`(64 KB)、`phy-qcom-ufs-qmp-v4-lahaina.c` 全部在树内。
   → 所以方案文档第 1 节「lahaina 在 5.10 上是一等公民目标」**方向正确，但「一等公民」这个词偏乐观**：见第 ③ 节的缺失面（DTS / vendor 配置 / techpack 三大件都不在这个仓库里）。

2. **上游 mainline Linux 从 v5.15 起就支持 SM8350**
   `arch/arm64/boot/dts/qcom/sm8350.dtsi`：v5.10 = 404，**v5.15 / v5.16 / v6.1 / v6.6 / master = 200**，且存在 `sm8350-mtp.dts`、`sm8350-hdk.dts`；mainline 还有 OnePlus 9 (lemonade) 的补丁系列，postmarketOS wiki 把 Sony Xperia 5 III（sony-pdx214 / SM8350）列为测试中设备。
   → 如果问题问的是字面意义的「SM8350 有没有跑过 5.10 以上内核」，答案是**有（mainline 6.x）**；但这是**无厂商驱动、无 Android/GKI ABI、不能跑 ColorOS** 的 mainline，与方案要解决的「5.10 ROM 迁移」不是同一件事。

**一句话给决策用：**
「没有任何 SM8350 手机厂商 BSP 上过 5.10」——**已核实成立**；
「SM8350 在 5.10 上没有内核支持」——**不成立，msm-5.10 有完整 SoC 内核侧支持**；
「拿到 msm-5.10 就能凑出一台能用的 5.10 martini」——**不成立**，因为 DTS / vendor config / techpack 三块都不在那个仓库里，而高通自己的 SM8350 产品 BSP（LA.UM.9.14.x，Android 12/13）用的仍然是 `kernel/msm-5.4`。

---

## ② 逐设备核实表

**证据等级说明**（重要，请勿混用）：
- **A = 厂商内核源码仓库实测**（直接读该分支 `Makefile` 的 VERSION/PATCHLEVEL/SUBLEVEL）
- **B = 社区内核仓库实测**（同上，但仓库来自社区，通常 rebase 自厂商源码）
- **C = 仓库/分支命名等间接证据**
- **未核实 = 本次没有取到可引用的版本证据（我没有猜）**

### 2.1 小米 / Redmi（全部实测 A 级）

| 机型 | 代号 / 分支 | SoC | 内核 | 等级 | 证据 URL |
| --- | --- | --- | --- | --- | --- |
| 小米 11 | venus-r-oss | SM8350 | **5.4.61** | A | [Makefile](https://raw.githubusercontent.com/MiCode/Xiaomi_Kernel_OpenSource/venus-r-oss/Makefile) |
| 小米 11 Pro / 11 Ultra | star-r-oss | SM8350 | **5.4.61** | A | [Makefile](https://raw.githubusercontent.com/MiCode/Xiaomi_Kernel_OpenSource/star-r-oss/Makefile) |
| Redmi K40 Pro / K40 Pro+ | haydn-r-oss | SM8350 | **5.4.61** | A | [Makefile](https://raw.githubusercontent.com/MiCode/Xiaomi_Kernel_OpenSource/haydn-r-oss/Makefile) |
| 小米 MIX 4 | odin-r-oss | SM8350+ | **5.4.86** | A | [Makefile](https://raw.githubusercontent.com/MiCode/Xiaomi_Kernel_OpenSource/odin-r-oss/Makefile) |
| 小米 11T Pro | vili-r-oss | SM8350 | **5.4.86** | A | [Makefile](https://raw.githubusercontent.com/MiCode/Xiaomi_Kernel_OpenSource/vili-r-oss/Makefile) |
| **Redmi K40 游戏增强版** | ares-r-oss | **MT6893 天玑1200（不是 SM8350！）** | **4.14.186** | A | [Makefile](https://raw.githubusercontent.com/MiCode/Xiaomi_Kernel_OpenSource/ares-r-oss/Makefile) |
| 小米 SM8350 全体（LineageOS 统一树） | lineage-23.2 | SM8350 | **5.4.302** | B | [LineageOS/android_kernel_xiaomi_sm8350](https://github.com/LineageOS/android_kernel_xiaomi_sm8350/blob/lineage-23.2/Makefile) |

> ⚠️ **纠正任务书里的一个前提**：**Redmi K40 游戏增强版（K40 Gaming）不是 SM8350**，是联发科天玑 1200（MT6893），内核 4.14.186（实测 `ares-r-oss` Makefile）。Redmi K40 Pro / Pro+ 才是 SM8350（haydn）。

### 2.2 一加 / OPPO（martini 所在家族）

| 机型 | 分支 | SoC | 内核 | 等级 | 证据 URL |
| --- | --- | --- | --- | --- | --- |
| 一加 9 / 9 Pro / 9RT(martini) 官方 Android 11 | `oneplus/SM8350_R_11.0` 等 | SM8350 | 5.4.61 | A | [OnePlusOSS 分支表](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8350/branches) |
| 一加 9RT 官方 Android 12.1 | `oneplus/sm8350_s_12.1_martini` | SM8350 | 5.4.147 | A | 同上 |
| 一加 9RT 官方 Android 13.1 | `oneplus/sm8350_t_13.1.0_oneplus9rt` | SM8350 | 5.4.210 | A | 同上 |
| 一加 9RT 官方 Android 14 | `oneplus/sm8350_u_14.0.0_oneplus9rt` | SM8350 | 5.4.254 | A | 同上 |
| 一加 9 系列 LineageOS 23.2 | lineage-23.2 | SM8350 | 5.4.302 | B | [LOS 仓库](https://github.com/LineageOS/android_kernel_oneplus_sm8350) |
| **一加 9RT 社区 Android 16 / 17** | Debarpan102 sixteen / seventeen | SM8350 | **5.4.302** | B | [Makefile@sixteen](https://raw.githubusercontent.com/Debarpan102/kernel_oneplus_sm8350/sixteen/Makefile) |
| OPPO Find X3 Pro | — | SM8350 | **未核实** | — | oppo-source org 里最新只到 Reno 9.0 一代，未找到该机型仓库 |
| **OnePlusOSS 是否存在 5.10 分支？** | 全部 11 个分支 | — | **无任何 5.10/5.15 分支** | A | [branches](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8350/branches) |

> **关键结论（与方案文档第 0 节一致）**：一加官方对 martini 的支持从 Android 11 → 14 全程 5.4；社区到 Android 16/17（lineage-23.2 / sixteen / seventeen）**仍然是 5.4.302**。

### 2.3 Realme（这里有一个必须澄清的「假阳性」）

| 机型 | 仓库 / 分支 | SoC | 内核 | 等级 | 证据 URL |
| --- | --- | --- | --- | --- | --- |
| Realme GT / GT2（RMX2202 / RMX3311）Android 12 | realme_GT_GT2-AndroidS-kernel-source | SM8350 | **5.4.86** | A | [Makefile](https://raw.githubusercontent.com/realme-kernel-opensource/realme_GT_GT2-AndroidS-kernel-source/master/Makefile) |
| Realme GT2 Android 13 | realme_gt2-AndroidT-kernel-source | SM8350 | **5.4.147** | A | [Makefile](https://raw.githubusercontent.com/realme-kernel-opensource/realme_gt2-AndroidT-kernel-source/master/Makefile) |
| Realme GT2 Android 14 | realme_gt2-AndroidU-kernel-source | SM8350 | **5.4.254** | A | [Makefile](https://raw.githubusercontent.com/realme-kernel-opensource/realme_gt2-AndroidU-kernel-source/master/Makefile) |
| ⚠️ realme_gt2-AndroidV-kernel-source（Android 15，5.10.209） | master | **实为 RMX3551 = GT2 大师探索版 = SM8475（不是 SM8350）** | 5.10.209 | A（版本）+ A（机型否定） | [Makefile](https://raw.githubusercontent.com/realme-kernel-opensource/realme_gt2-AndroidV-kernel-source/master/Makefile) · [提交信息](https://github.com/realme-kernel-opensource/realme_gt2-AndroidV-kernel-source/commits/master) |
| Realme GT2 Pro（RMX3300） | gt2pro Android T/U/V | SM8450 | 5.10.136 / 5.10.168 / 5.10.209 | A | [AndroidU Makefile](https://raw.githubusercontent.com/realme-kernel-opensource/realme_gt2pro-AndroidU-kernel-source/master/Makefile) |

> **这是本次最容易误判的一条，必须写清楚**：
> 仓库名 `realme_gt2-AndroidV-kernel-source` 看起来是「Realme GT2（SM8350）的 Android 15 内核 = 5.10.209」，
> 但该仓库 **master 最新提交信息为**：`Synchronize code for realme RMX3551_15.0.0.110(CN01) Based on QCOM release TAG:AU_LINUX_KERNEL.PLATFORM.1.0.R1.00.00.00.`
> **RMX3551 = Realme GT2 大师探索版 = 骁龙 8+ Gen 1（SM8475）**，不是 SM8350。
> 该树与 `realme_gt2pro-AndroidV-kernel-source`（RMX3300 / SM8450）**根目录结构完全相同**（都含 `build.config.msm.{lahaina,waipio,parrot,anorak}`，`arch/arm64/configs/vendor/` 里只有 waipio/parrot/anorak/neo 配置，**没有 lahaina 配置**），说明它就是 waipio 系共用 5.10 树，只是被 realme 用 gt2 的名字发布了。
> 另外：**SM8350 版 Realme GT2（RMX3311）确实收到了 Android 15 / realme UI 6.0（`RMX3311_15.0.0.1650(EX01)`）**，但我**没有找到**该机型 Android 15 对应的内核源码仓库；其 Android 14 源码仍是 5.4.254。→ 该机型是否在 Android 15 上升级内核：**未核实，不下结论**。

### 2.4 三星

| 机型 | 树 | SoC | 内核 | 等级 | 证据 URL |
| --- | --- | --- | --- | --- | --- |
| Galaxy S21 FE 5G（骁龙 SM-G990B/B2） | Lightning kernel stock 分支 | SM8350 | **5.4.147** | B | [Makefile@stock](https://raw.githubusercontent.com/prorooter007/Lightning_android_kernel_samsung_sm8350/stock/Makefile) |
| Galaxy S21 FE 5G（骁龙） | 同上 main | SM8350 | 5.4.228 | B | [Makefile@main](https://raw.githubusercontent.com/prorooter007/Lightning_android_kernel_samsung_sm8350/main/Makefile) |
| Galaxy S21 FE 5G（骁龙） | retrozenith lineage-22 | SM8350 | 5.4.274 | B | [Makefile](https://raw.githubusercontent.com/retrozenith/android_kernel_samsung_sm8350/lineage-22/Makefile) |
| Galaxy S21 Ultra（骁龙 p3q） | yanzihan sixteen | SM8350 | **5.4.302** | B | [Makefile](https://raw.githubusercontent.com/yanzihan/android_kernel_samsung_sm8350-p3q/sixteen/Makefile) |
| 三星 SM8350 通用社区树 | saadelasfur vanilla | SM8350 | 5.4.302 | B | [Makefile](https://raw.githubusercontent.com/saadelasfur/android_kernel_samsung_sm8350/vanilla/Makefile) |
| Galaxy S21/S21+/S21 Ultra（骁龙）官方 | opensource.samsung.com | SM8350 | **未核实**（官网只给压缩包，页面无版本号） | — | [Samsung OSS 搜索 SM-G990](https://opensource.samsung.com/uploadSearch?searchValue=SM-G990) |

### 2.5 索尼 / 华硕 / 微软 / Nothing / 夏普

| 机型 | 树 | SoC | 内核 | 等级 | 证据 URL |
| --- | --- | --- | --- | --- | --- |
| Xperia 1 III / 5 III / Pro-I | sonyxperiadev/kernel @ aosp/LA.UM.9.14.r1（LA.UM.9.14 = lahaina 家族） | SM8350 | **5.4.61** | A | [Makefile](https://raw.githubusercontent.com/sonyxperiadev/kernel/aosp/LA.UM.9.14.r1/Makefile) |
| Xperia 1 III / 5 III（LineageOS 23.2） | LineageOS android_kernel_sony_sm8350 | SM8350 | 5.4.302 | B | [仓库](https://github.com/LineageOS/android_kernel_sony_sm8350) |
| ROG Phone 5 / 5s | LineageOS android_kernel_asus_sm8350 @ lineage-18.1 | SM8350 | **5.4.61** | B | [Makefile](https://raw.githubusercontent.com/LineageOS/android_kernel_asus_sm8350/lineage-18.1/Makefile) |
| ROG Phone 5 / 5s（最新） | 同上 @ lineage-23.2 | SM8350 | 5.4.302 | B | [Makefile](https://raw.githubusercontent.com/LineageOS/android_kernel_asus_sm8350/lineage-23.2/Makefile) |
| 微软 Surface Duo 2 | 社区 QGKI 内核 | SM8350 | **5.4**（仓库自述 Linux 5.4 QGKI，分支实测 5.4.233） | B/C | [仓库](https://github.com/dtingley11/SurfaceDuo2-QGKI-Kernel) · [Makefile](https://raw.githubusercontent.com/dtingley11/SurfaceDuo2-QGKI-Kernel/main/Makefile) |
| **Nothing Phone (1)** | NothingOSS 官方 | **SM7325（778G+），不是 SM8350** | 5.4.147 | A | [NothingOSS 仓库](https://github.com/NothingOSS/android_kernel_msm-5.4_nothing_sm7325) |
| Nothing Phone (1) 社区 | ExTV msm-5.4 | SM7325 | 5.4.289 | B | [Makefile](https://raw.githubusercontent.com/ExTV/android_kernel_msm-5.4_nothing_sm7325/main/Makefile) |
| AQUOS R6 | 官方 OSS（tar.gz 归档） | SM8350 | **未核实**（官方只发布归档，无版本标注） | — | [Sharp OSS R6](http://k-tai.sharp.co.jp/support/developers/oss/aquos-r6/index.html) |
| AQUOS R7 | 官方 OSS（.7z，单包 619 MB） | SM8350 | **未核实** | — | [Sharp OSS R7](http://k-tai.sharp.co.jp/support/developers/oss/aquos-r7/index.html) |

> Nothing Phone (1) 是任务书列表里的一个前提错误：它是 **SM7325**，不在 SM8350 讨论范围内。

### 2.6 中兴 / 努比亚 / 摩托罗拉 / 其它

| 机型 | 树 | SoC | 内核 | 等级 | 证据 URL |
| --- | --- | --- | --- | --- | --- |
| ZTE Axon 30 Pro / Nubia Z30 Pro | CircleCashTeam（分支名即 P875A02-LA.UM.9.14.r1-19200-LAHAINA.QSSI12.0） | SM8350 | **5.4.147** | A | [Makefile](https://raw.githubusercontent.com/CircleCashTeam/android_kernel_zte_sm8350/P875A02-LA.UM.9.14.r1-19200-LAHAINA.QSSI12.0/Makefile) |
| Nubia Red Magic 6 | anrui2032 @ lineage-22.2 | SM8350 | 5.4.293 | B | [Makefile](https://raw.githubusercontent.com/anrui2032/android_kernel_nubia_sm8350/lineage-22.2/Makefile) |
| 摩托罗拉 SD888 机型（Edge S30 / Moto G200） | MotorolaMobilityLLC/kernel-msm | SM8350 | **未核实** | — | [仓库](https://github.com/MotorolaMobilityLLC/kernel-msm) |
| Lenovo Legion Duel 2 | — | SM8350 | **未核实** | — | 未找到公开镜像 |
| 魅族 18 / 黑鲨 4 Pro / 荣耀 Magic3 / 华为 P50 | — | SM8350 | **未核实** | — | 未找到逐机型公开源码镜像 |
| vivo iQOO 8 / iQOO Neo5S | opensource.vivo.com（本会话未取到逐机型版本） | SM8350 | **未核实** | — | [Rmh04/iQOOneo5s 为空仓库](https://github.com/Rmh04/iQOOneo5s) |

### 2.7 高通参考设计 QRD8350（社区项目，很重要）

| 对象 | 树 | 内核 | 等级 | 证据 URL |
| --- | --- | --- | --- | --- |
| QRD8350（SM8350 官方参考设计） | QRD-Development/android_kernel_qcom_sm8350 @ lineage-23.2 | **5.4.302** | B | [Makefile](https://raw.githubusercontent.com/QRD-Development/android_kernel_qcom_sm8350/lineage-23.2/Makefile) |
| QRD8350 BSP 同步清单（Android 13 / QSSI13） | QRD-Development/SM8350_BSP_Sync @ LA.UM.9.14.1 | **kernel/msm-5.4**（清单原文见 ③.4） | A | [target.xml](https://raw.githubusercontent.com/QRD-Development/SM8350_BSP_Sync/LA.UM.9.14.1/target.xml) |

> 这一条是「SM8350 官方 BSP 到底停在哪个内核」的最硬证据：**连高通自己的 SM8350 BSP 同步清单（LA.UM.9.14.1 / QSSI13）拉的也是 kernel/msm-5.4**。社区基于它做到了 Android 16（lineage-23.2），仍然是 5.4.302。

---

## ③ msm-5.10 lahaina 证据原文

### 3.0 取证对象与可复现 URL

- 主取证对象（GitHub 镜像，raw 不限流）：
  https://raw.githubusercontent.com/arter97-mirror/caf_msm-5.10/kernel.lnx.5.10.r1-rel/...
- 该镜像对应的高通官方项目（CodeLinaro GitLab API 实测存在）：
  clo/la/kernel/msm-5.10，project id = **29371**，实测 200
- 分支 kernel.lnx.5.10.r1-rel 的 Makefile 实测：**VERSION=5 PATCHLEVEL=10 SUBLEVEL=252**。

### 3.1 build.config.msm.lahaina 原文（29 行 / 881 字节，逐字复制）

来源：<https://raw.githubusercontent.com/arter97-mirror/caf_msm-5.10/kernel.lnx.5.10.r1-rel/build.config.msm.lahaina>

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

**与 waipio 对照（同一仓库，原文）**：<https://raw.githubusercontent.com/arter97-mirror/caf_msm-5.10/kernel.lnx.5.10.r1-rel/build.config.msm.waipio>

```sh
MSM_ARCH=waipio
VARIANTS=(consolidate gki)
[ -z "${VARIANT}" ] && VARIANT=consolidate
...
BOOT_IMAGE_HEADER_VERSION=3
BASE_ADDRESS=0x80000000
PAGE_SIZE=4096
BUILD_VENDOR_DLKM=1
SUPER_IMAGE_SIZE=0x10000000
TRIM_UNUSED_MODULES=1

MODULES_LIST_ORDER="1"
[ -z "${DT_OVERLAY_SUPPORT}" ] && DT_OVERLAY_SUPPORT=1
...
KERNEL_VENDOR_CMDLINE+=' console=ttyMSM0,115200n8 earlycon msm_geni_serial.con_enabled=1'
```

**判读（已核实的差异，不是推测）**：
- lahaina 只有 VARIANTS=(gki) 单一变体；waipio 是 (consolidate gki)。
- lahaina **没有** BUILD_VENDOR_DLKM=1 / SUPER_IMAGE_SIZE / TRIM_UNUSED_MODULES / MODULES_LIST_ORDER="1"，waipio 有。
- 两者都 BASE_ADDRESS=0x80000000、PAGE_SIZE=4096、BOOT_IMAGE_HEADER_VERSION=3。
- 结论：**它是一个真实、可独立构建的 GKI 目标，但配置明显比 waipio 薄**（无 vendor_dlkm 变体、无模块裁剪）。它不像「占位」，但也不是「旗舰完整产品配置」。

### 3.2 modules.list.msm.lahaina 原文（97 行 / 1510 字节）

来源：<https://raw.githubusercontent.com/arter97-mirror/caf_msm-5.10/kernel.lnx.5.10.r1-rel/modules.list.msm.lahaina>

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

**判读**：这是一份**真实的 lahaina 专属模块清单**，含 5 个带 -lahaina 后缀的模块（gcc / dispcc / qnoc / pinctrl / phy-qcom-ufs-qmp-v4），以及 Adreno KGSL、WALT 调度、UFS、USB、IPA 等完整子系统。**不是占位文件。**

**注意它缺失什么**：清单里**没有** display/audio/camera/video 类 techpack 模块（没有 msm_drm、bolero、camera 相关 .ko）。这些正是 techpack 提供的部分，见 3.4。

### 3.3 内核侧支持有多深（逐文件 HTTP 实测）

| 路径（分支 kernel.lnx.5.10.r1-rel） | HTTP | 大小 |
| --- | --- | --- |
| arch/arm64/Kconfig.platforms（含 config ARCH_LAHAINA） | 200 | — |
| drivers/clk/qcom/gcc-lahaina.c | 200 | 117,615 B |
| drivers/clk/qcom/dispcc-lahaina.c | 200 | 43,669 B |
| drivers/pinctrl/qcom/pinctrl-lahaina.c | 200 | 59,575 B |
| drivers/interconnect/qcom/lahaina.c | 200 | 64,606 B |
| drivers/phy/qualcomm/phy-qcom-ufs-qmp-v4-lahaina.c | 200 | 8,592 B |
| drivers/gpu/msm/kgsl.c | 200 | 135,501 B |
| arch/arm64/boot/dts/vendor/qcom/lahaina.dtsi | **404** | — |
| arch/arm64/boot/dts/vendor/qcom/lahaina-mtp.dts | **404** | — |
| arch/arm64/configs/vendor/lahaina_defconfig | **404** | — |
| arch/arm64/configs/vendor/lahaina_GKI.config | **404** | — |
| arch/arm64/configs/vendor/waipio_GKI.config | 200 | — |

**这是本节最重要的两条「缺失」证据（原文引用）：**

1) **DTS 不在仓库里** —— arch/arm64/boot/dts/Makefile 原文（节选）：
```make
dtstree	:= $(srctree)/$(src)
vendor  := $(dtstree)/vendor
ifneq "$(wildcard $(vendor)/Makefile)" ""
    subdir-y += vendor
endif
```
即 dts/vendor 是**外部注入**的（本仓库里连 .gitignore 都没有，vendor/Makefile 不存在就不编译）。

2) **vendor 配置片段也不在仓库里** —— arch/arm64/configs/vendor/.gitignore **全文**：
```text
# SPDX-License-Identifier: GPL-2.0-only
# waipio uses config fragments and gki_defconfig as base
waipio-*_defconfig
# lahaina uses config fragments and gki_defconfig as base
lahaina-*_defconfig
```
—— 这条注释**直接证明 lahaina 是「config fragment + gki_defconfig」模式**，而 fragment（lahaina-*.config）不在这个仓库里。

### 3.4 techpack：你实测到「只有 .gitignore / Kbuild / stub」——**已核实，是真的**，但要补上「它们去哪了」

**(a) 仓库里确实只有 stub**（GitHub API 目录实测）：
```text
techpack/  ->  .gitignore (84 B)   Kbuild (240 B)   stub/ (dir)
```

techpack/Kbuild 原文：
```make
# SPDX-License-Identifier: GPL-2.0-only
TECHPACK?=y

techpack-dirs := $(shell find $(srctree)/techpack -maxdepth 1 -mindepth 1 -type d -not -name ".*")
obj-${TECHPACK} += stub/ $(addsuffix /,$(subst $(srctree)/techpack/,,$(techpack-dirs)))
```

techpack/.gitignore 原文：
```text
# SPDX-License-Identifier: GPL-2.0-only
# ignore all subdirs except stub
!/stub/
*/
```

→ 语义非常明确：**任何被放进 techpack/ 的子目录都是「外部项目」，不进本仓库版本控制**。Kbuild 会在构建时把 techpack/ 下真实存在的目录加进来。所以「这里没有 display/audio/camera」并不等于「高通没有 5.10 的 display/audio/camera」，而是它们**被拆到了独立的 CLO 仓库**。

**(b) 它们被拆到哪些仓库？—— 用高通官方清单实证（不是猜）**

我从一个真实的 SM8350/lahaina BSP 同步清单里拿到了映射关系（repo manifest，官方字段）：
来源：<https://raw.githubusercontent.com/QRD-Development/SM8350_BSP_Sync/LA.UM.9.14.1/target.xml>

```xml
<project remote="clo-la" name="kernel/msm-5.4" path="kernel/msm-5.4" revision="55e1a5c8..." upstream="refs/heads/kernel.lnx.5.4.r1-rel" groups="cyborg"/>
<project remote="clo-la" name="platform/vendor/opensource/audio-kernel"    path="kernel/msm-5.4/techpack/audio"   upstream="refs/heads/audio-drivers.lnx.5.0.r1-rel"  groups="cyborg"/>
<project remote="clo-la" name="platform/vendor/opensource/camera-kernel"   path="kernel/msm-5.4/techpack/camera"  upstream="refs/heads/camera-kernel.lnx.4.0.r1-rel" groups="cyborg"/>
<project remote="clo-la" name="platform/vendor/opensource/dataipa"         path="kernel/msm-5.4/techpack/dataipa" upstream="refs/heads/data-kernel.lnx.1.1.r1-rel"   groups="cyborg"/>
<project remote="clo-la" name="platform/vendor/opensource/display-drivers" path="kernel/msm-5.4/techpack/display" upstream="refs/heads/display-kernel.lnx.5.4.r1-rel" groups="cyborg"/>
<project remote="clo-la" name="platform/vendor/opensource/video-driver"    path="kernel/msm-5.4/techpack/video"   upstream="refs/heads/video-kernel.lahaina.lnx.1.0.r1-rel" groups="cyborg"/>
<project remote="clo-la" name="platform/vendor/qcom/lahaina" path="device/qcom/lahaina" upstream="refs/heads/qcom-devices.lnx.6.0.r3-rel"/>
```

**请特别注意这张表里的三件事**：
1. techpack 五大件（audio / camera / dataipa / display / video）确实是**独立仓库**，按 manifest 挂到 kernel/<ver>/techpack/*。你的观察是对的，但结论应该是「外置」而不是「缺失」。
2. **这个清单是 SM8350 的 BSP（含 device/qcom/lahaina、video-kernel.lahaina.*），而它拉的 kernel 是 kernel/msm-5.4（kernel.lnx.5.4.r1-rel）。** 这是「高通自己的 SM8350 产品 BSP 停在 5.4」的硬证据 —— 且该清单对应 Android 13（QSSI13）/ LA.UM.9.14.1。
3. video-driver 有 lahaina 专属分支（video-kernel.lahaina.lnx.1.0.*，我实测存在 20+ 个），说明 **lahaina 的 techpack 是按 SoC 命名分支的**；而 display/audio/camera 的分支名不含 SoC。

**(c) 那 5.10 的 techpack 有没有 lahaina？—— 本次【未核实完成】，以下是实测边界，请不要当成结论**

已核实存在的 CLO 独立仓库（GitLab API 实测 project id）：
- display-drivers → id **13664**，**存在 5.10 分支**：display-kernel.lnx.5.10.r1-rel、...r2-rel … ...r11-rel、display-kernel.lnx.5.10.2.c1 等
- audio-kernel → id **13492**：搜索 lnx.5.10 **无结果**（只见 audio-drivers.lnx.5.0.* 与 audio-kernel.lnx.5.15.*）
- camera-kernel → id **13577**：搜索 lnx.5.10 **无结果**
- video-driver → id **12562**：lahaina 专属分支存在，但命名是 video-kernel.lahaina.lnx.1.0.*（与 5.4 BSP 对应）

**未能完成的部分（如实报告）**：
- 尝试用 GitLab tree API 读取 display-kernel.lnx.5.10.r1-rel 的内容 → 返回 {"message":"404 Tree Not Found"}；
- 尝试 raw 读取该分支的 Makefile → **404**；msm/Makefile → **429（被限流）**；
- 尝试用 GitLab blob 搜索 lahaina → {"message":"401 Unauthorized"}（该 API 需要登录）。
- 因此：**「5.10 的 display/audio/camera techpack 是否包含 lahaina 实现」本次未核实**。既不能说有，也不能说没有。
- 能确定的只有：① 5.10 的 display techpack 分支**在官方仓库里存在**；② 5.4 的 lahaina BSP 用 display-kernel.lnx.5.4.r1-rel / audio-drivers.lnx.5.0.r1-rel / camera-kernel.lnx.4.0.r1-rel；③ 这些目录都必须**手动挂载**到 kernel/msm-5.10/techpack/* 才能构建。

### 3.5 对方案文档第 1 节「有利条件 1」的直接修正建议

原文：「build.config.msm.lahaina ← lahaina 是独立可构建目标；modules.list.msm.lahaina ← 且有专属模块清单（不是占位）；与 waipio/parrot/ai2202 并列。说明 SM8350 在 5.10 上是**一等公民目标**，不是从零适配 SoC。」

**逐条核实结果**：
- ✅ 两个文件确实存在，确实不是占位（原文见上）。
- ✅ ARCH_LAHAINA 在 arch/arm64/Kconfig.platforms 中，且 **5 个 lahaina 专属驱动源文件在树内**（gcc / dispcc / pinctrl / interconnect / UFS PHY）。
- ⚠️ 但「并列」不准确：仓库根目录里 build.config.msm.* 只有 common / gki / lahaina / parrot / vm / waipio / waipio.tuivm；**没有 ai2202**（ai2202 出现在别的镜像里，本分支没有）。lahaina 的构建配置**只有 gki 一个变体**，waipio 有 consolidate+gki 且带 BUILD_VENDOR_DLKM=1。
- ❌ 「含 drivers / DTS / defconfig 的完整 SoC 支持」**不成立**：DTS（404）、vendor config fragment（被 .gitignore 排除，仓库内 404）、techpack（stub）**三块都不在仓库内**。
- 建议改写为：**「msm-5.10 内核侧对 lahaina 是真实支持（Kconfig + 驱动 + 模块清单齐全），但公开仓库只包含内核，DTS / vendor config / display·audio·camera·video techpack 全在仓库之外；且高通自己的 SM8350 产品 BSP 清单仍指向 msm-5.4。」**

---

## ④ 社区移植尝试：martini / SM8350 → 5.10

**结论：没有找到任何有实质内容的 5.10 移植尝试。**（逐条实测，见下）

| 检查对象 | 结果 | 证据 |
| --- | --- | --- |
| OnePlusOSS 官方 sm8350 内核（11 个分支） | 只有 Android 11→14 分支，**无任何 5.10/5.15 分支** | [branches](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8350/branches) |
| LineageOS android_kernel_oneplus_sm8350（15 个分支，含 lineage-24.0） | 全部 5.4 系（lineage-23.2 = 5.4.302） | [仓库](https://github.com/LineageOS/android_kernel_oneplus_sm8350) |
| crDroid android_kernel_oneplus_sm8350（60 个分支：11.0 → 最新） | 未见 5.10 分支 | [仓库](https://github.com/crdroidandroid/android_kernel_oneplus_sm8350) |
| arter97（master / mglru / sapphire / topaz / aospa / custom） | 6 个分支，无 5.10 | [仓库](https://github.com/arter97/android_kernel_oneplus_sm8350) |
| dev-sm8350/kernel_oneplus_sm8350（12 分支，含 android11-5.4-lts） | 全部 5.4 | [仓库](https://github.com/dev-sm8350/kernel_oneplus_sm8350) |
| Debarpan102/kernel_oneplus_sm8350（martini 的 Android 16/17 社区内核） | sixteen = **5.4.302**、seventeen = **5.4.302**、sixteen-legacy = 5.4.296 | [Makefile@sixteen](https://raw.githubusercontent.com/Debarpan102/kernel_oneplus_sm8350/sixteen/Makefile) |
| wannqn/5.10-sm8350（仓库名直接叫 5.10-sm8350） | **空仓库**：size=0、无 README、无 Makefile、无内容（API 实测 created 2026-07-26） | [仓库](https://github.com/wannqn/5.10-sm8350) |
| QRD-Development（QRD8350 参考设计，AOSPA/LineageOS/CalyxOS） | 内核仓库 lineage-23.2 = **5.4.302**；BSP 同步清单只有 LA.UM.9.14.1 与 LA.UM.9.14.r1 两个分支，都指向 **msm-5.4** | [Makefile](https://raw.githubusercontent.com/QRD-Development/android_kernel_qcom_sm8350/lineage-23.2/Makefile) · [SM8350_BSP_Sync](https://github.com/QRD-Development/SM8350_BSP_Sync) |
| XDA / Gitee / 酷安 关键词检索 | 未找到有代码的 5.10 移植工程；能找到的 martini 社区工程都是 KernelSU 编译指南（基于 5.4） | [KernelSU_Oneplus_martini_Guide](https://github.com/natsumerinchan/KernelSU_Oneplus_martini_Guide) |

**唯一真实存在的「SM8350 + 5.10 以上」是上游 mainline，不是 Android 移植**：
- arch/arm64/boot/dts/qcom/sm8350.dtsi：v5.10 = **404**，v5.15 = **200**，v5.16 / v6.1 / v6.6 / master = **200**（逐 tag 实测）
  → <https://raw.githubusercontent.com/torvalds/linux/v5.15/arch/arm64/boot/dts/qcom/sm8350.dtsi>
- mainline 另有 sm8350-mtp.dts、sm8350-hdk.dts（v6.1 / v6.6 / master 均 200）
- mainline 有 OnePlus 9 (lemonade/lemonadep) 相关补丁系列：<http://mail.spinics.net/lists/linux-iio/msg83038.html>
- postmarketOS wiki 把 Sony Xperia 5 III（sony-pdx214，SM8350）列为测试中设备：<https://wiki.postmarketos.org/wiki/User:Akku/devices_in_testing>
- ⚠️ 但 mainline 没有 OPLUS 厂商层、没有 Adreno KGSL 下游栈、不能跑 Android/ColorOS —— 与方案要解决的问题不是同一件事。

---

## ⑤ 未找到 / 未核实的部分（明确清单）

**未核实内核版本（本次没拿到可引用证据，未做任何猜测）**：
1. OPPO Find X3 Pro（SM8350）—— oppo-source org 最新仓库只到 Reno 9.0 一代，未找到该机型镜像。
2. vivo iQOO 8 / iQOO Neo5S（SM8350，后者出厂 Android 12）—— opensource.vivo.com 未取到逐机型版本；GitHub 上 Rmh04/iQOOneo5s 为空仓库。
3. 摩托罗拉 SD888 机型（Edge S30 / Moto G200 5G）—— 未定位到对应 tag/分支。
4. Lenovo Legion Duel 2、魅族 18、黑鲨 4 Pro、荣耀 Magic3、华为 P50 —— 未找到逐机型公开源码镜像。
5. Sharp AQUOS R6 / R7 —— 官方 OSS 只提供归档（R6 多份 AQUOS_R6_*.tar.gz；R7 为 AQUOS_R7_*.7z，单包 **619,516,769 字节**），文件名不含内核版本，本次未下载解析。
6. 三星 S21 / S21+ / S21 Ultra（骁龙）**官方**源码版本 —— 官网只给压缩包；本表用的是社区树（B 级）。

**未核实的核心技术问题**：
7. **5.10 的 display / audio / camera techpack 是否包含 lahaina 实现** —— 见 ③.4(c)。CLO 的 5.10 分支存在（display），但 tree API 返回 404 Tree Not Found、raw 404、blob 搜索需登录（401），无法确认内容。
8. arch/arm64/boot/dts/vendor 与 arch/arm64/configs/vendor 的**外部来源仓库**具体是哪个 CLO 项目 —— 本次只证明了「它们不在内核仓库内」，没有定位到官方 DTS/config 仓库（未找到）。
9. 高通 msm-5.10 里 lahaina 支持**是为哪个客户/产品准备的** —— **未找到**官方说明。检索中看到 SA8540P（汽车）**不是** SM8350 衍生（mainline sa8540p.dtsi 是 #include "sc8280xp.dtsi"，属 SC8280XP 家族），所以「lahaina 进 5.10 是为了车机」这一常见说法**本次没有得到证据支持**。

---

## ⑥ 参考链接

**msm-5.10 / 高通**
- [CAF msm-5.10 镜像（arter97-mirror，分支 kernel.lnx.5.10.r1-rel）](https://github.com/arter97-mirror/caf_msm-5.10)
- [build.config.msm.lahaina 原文](https://raw.githubusercontent.com/arter97-mirror/caf_msm-5.10/kernel.lnx.5.10.r1-rel/build.config.msm.lahaina)
- [modules.list.msm.lahaina 原文](https://raw.githubusercontent.com/arter97-mirror/caf_msm-5.10/kernel.lnx.5.10.r1-rel/modules.list.msm.lahaina)
- [build.config.msm.waipio（对照）](https://raw.githubusercontent.com/arter97-mirror/caf_msm-5.10/kernel.lnx.5.10.r1-rel/build.config.msm.waipio)
- [techpack/Kbuild 原文](https://raw.githubusercontent.com/arter97-mirror/caf_msm-5.10/kernel.lnx.5.10.r1-rel/techpack/Kbuild)
- [techpack/.gitignore 原文](https://raw.githubusercontent.com/arter97-mirror/caf_msm-5.10/kernel.lnx.5.10.r1-rel/techpack/.gitignore)
- [configs/vendor/.gitignore 原文（lahaina 用 fragment 的直接证据）](https://raw.githubusercontent.com/arter97-mirror/caf_msm-5.10/kernel.lnx.5.10.r1-rel/arch/arm64/configs/vendor/.gitignore)
- [arch/arm64/Kconfig.platforms（ARCH_LAHAINA）](https://raw.githubusercontent.com/arter97-mirror/caf_msm-5.10/kernel.lnx.5.10.r1-rel/arch/arm64/Kconfig.platforms)
- [CodeLinaro 官方项目 clo/la/kernel/msm-5.10（id 29371）](https://git.codelinaro.org/api/v4/projects/clo%2Fla%2Fkernel%2Fmsm-5.10)
- [CodeLinaro display-drivers（id 13664，含 5.10 分支）](https://git.codelinaro.org/clo/la/platform/vendor/opensource/display-drivers)
- [CodeLinaro audio-kernel（id 13492）](https://git.codelinaro.org/clo/la/platform/vendor/opensource/audio-kernel)
- [CodeLinaro camera-kernel（id 13577）](https://git.codelinaro.org/clo/la/platform/vendor/opensource/camera-kernel)
- [CodeLinaro video-driver（id 12562，含 lahaina 专属分支）](https://git.codelinaro.org/clo/la/platform/vendor/opensource/video-driver)

**SM8350 BSP 清单（techpack 去向的直接证据）**
- [QRD-Development/SM8350_BSP_Sync @ LA.UM.9.14.1 · target.xml](https://raw.githubusercontent.com/QRD-Development/SM8350_BSP_Sync/LA.UM.9.14.1/target.xml)
- [QRD-Development/android_kernel_qcom_sm8350（lineage-23.2 = 5.4.302）](https://github.com/QRD-Development/android_kernel_qcom_sm8350)
- [QRD-Development/android_device_qcom_lahaina](https://github.com/QRD-Development/android_device_qcom_lahaina)

**mainline（SM8350 真正进入 5.10+ 的地方）**
- [sm8350.dtsi @ v5.15](https://raw.githubusercontent.com/torvalds/linux/v5.15/arch/arm64/boot/dts/qcom/sm8350.dtsi)（v5.10 为 404）
- [sm8350-mtp.dts @ v6.6](https://raw.githubusercontent.com/torvalds/linux/v6.6/arch/arm64/boot/dts/qcom/sm8350-mtp.dts)
- [sa8540p.dtsi @ v6.6（证明汽车 SA8540P 属 SC8280XP，不是 SM8350）](https://raw.githubusercontent.com/torvalds/linux/v6.6/arch/arm64/boot/dts/qcom/sa8540p.dtsi)

**厂商 / 社区内核源码（逐设备证据）**
- 小米：[MiCode/Xiaomi_Kernel_OpenSource](https://github.com/MiCode/Xiaomi_Kernel_OpenSource)
- 一加：[OnePlusOSS/android_kernel_oneplus_sm8350](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8350) · [音频 techpack（SM8350，仅 R_11.0）](https://github.com/OnePlusOSS/android_vendor_qcom_opensource_audio_kernel_sm8350)
- Realme：[realme-kernel-opensource](https://github.com/orgs/realme-kernel-opensource/repositories)
- 三星：[prorooter007/Lightning_android_kernel_samsung_sm8350](https://github.com/prorooter007/Lightning_android_kernel_samsung_sm8350) · [saadelasfur/android_kernel_samsung_sm8350](https://github.com/saadelasfur/android_kernel_samsung_sm8350) · [Samsung OSS](https://opensource.samsung.com/uploadSearch?searchValue=SM-G990)
- 索尼：[sonyxperiadev/kernel @ aosp/LA.UM.9.14.r1](https://raw.githubusercontent.com/sonyxperiadev/kernel/aosp/LA.UM.9.14.r1/Makefile) · [LineageOS sony_sm8350](https://github.com/LineageOS/android_kernel_sony_sm8350)
- 华硕：[LineageOS asus_sm8350](https://github.com/LineageOS/android_kernel_asus_sm8350)
- 微软：[dtingley11/SurfaceDuo2-QGKI-Kernel](https://github.com/dtingley11/SurfaceDuo2-QGKI-Kernel)
- Nothing：[NothingOSS/android_kernel_msm-5.4_nothing_sm7325](https://github.com/NothingOSS/android_kernel_msm-5.4_nothing_sm7325)（SM7325，非 SM8350）
- 中兴/努比亚：[CircleCashTeam/android_kernel_zte_sm8350](https://github.com/CircleCashTeam/android_kernel_zte_sm8350) · [anrui2032/android_kernel_nubia_sm8350](https://github.com/anrui2032/android_kernel_nubia_sm8350)
- 夏普：[AQUOS R6 OSS](http://k-tai.sharp.co.jp/support/developers/oss/aquos-r6/index.html) · [AQUOS R7 OSS](http://k-tai.sharp.co.jp/support/developers/oss/aquos-r7/index.html)
- 社区 martini：[Debarpan102/kernel_oneplus_sm8350](https://github.com/Debarpan102/kernel_oneplus_sm8350)（sixteen/seventeen = 5.4.302）

---

## ⑦ 我核实到的 vs 我推测的

### 7.1 我核实到的（每条都有上面 URL 支撑，可复现）

1. **一加/小米/Realme/三星(骁龙)/索尼/华硕/中兴/努比亚/Surface Duo 2 的 SM8350 内核，实测到的最高版本全部是 5.4.x**，最新为 5.4.302（LineageOS 23.2 / 社区 Android 16 树）。
2. **OnePlusOSS 的 sm8350 内核仓库只有 11 个分支，全部 Android 11–14，没有任何 5.10/5.15 分支。**
3. **Redmi K40 游戏增强版是 MT6893 天玑 1200（4.14.186），不是 SM8350**；SM8350 对应 K40 Pro/Pro+（haydn, 5.4.61）。
4. **Nothing Phone (1) 是 SM7325，不是 SM8350。**
5. **realme_gt2-AndroidV-kernel-source（5.10.209）的实际对象是 RMX3551（GT2 大师探索版 / SM8475）**，其 master 最新提交信息写明机型；同仓根结构与 realme_gt2pro-AndroidV（RMX3300 / SM8450）完全一致，且 configs/vendor 里没有 lahaina 配置。**它不是 SM8350 的 5.10 内核。**
6. **高通公开 msm-5.10（kernel.lnx.5.10.r1-rel，5.10.252）里 lahaina 是真实 SoC 目标**：ARCH_LAHAINA + build.config.msm.lahaina + 97 行模块清单 + 5 个 lahaina 专属驱动源文件（含 117 KB 的 gcc-lahaina.c）。
7. **该仓库里没有 lahaina 的 DTS、没有 vendor config fragment、techpack 只有 stub**（三处 HTTP 404 / 目录实测）；configs/vendor/.gitignore 的注释明确写「lahaina uses config fragments and gki_defconfig as base」。
8. **techpack 五大件是独立 CLO 仓库**，由官方 repo 清单挂载到 kernel/<ver>/techpack/*；这份清单正是 **SM8350/lahaina 的 BSP（LA.UM.9.14.1 / Android 13）**，而它拉的 kernel 是 **msm-5.4**。
9. **mainline Linux 从 v5.15 起支持 SM8350**（v5.10 无 sm8350.dtsi，v5.15 有），并有 mtp/hdk 板级 DTS。
10. **没有任何社区项目在做 martini 或 SM8350 的 5.10 移植**：扫了 OnePlusOSS / LineageOS / crDroid / arter97 / dev-sm8350 / Debarpan102 / QRD-Development 的全部分支；唯一叫 5.10-sm8350 的仓库是空的。
11. **lahaina 的 video techpack 有专属分支**（video-kernel.lahaina.lnx.1.0.*，20+ 个）；display/audio/camera 的分支名不含 SoC。
12. **CLO 的 display-drivers 存在 5.10 分支名**（display-kernel.lnx.5.10.r1-rel 等），但 audio-kernel / camera-kernel 搜索不到 lnx.5.10 分支。

### 7.2 我推测的（**没有直接证据，请当成假设**）

1. **推测**：没有 SM8350 手机厂商 BSP 上过 5.10 的原因，是芯片生命周期（2021 发布 + Android 11/12 起步）全部落在 5.4 BSP 窗口内；高通把 lahaina 带进 msm-5.10 更像是 BSP 平台统一/衍生客户用途，而非手机产品线。**没有找到任何官方说明**（见 ⑤-9）。
2. **推测**：真机若要做 5.10，现实的显示/音频/相机来源只可能是「CLO 的 5.10 techpack 仓库（display 分支已确认存在）+ 从 5.4 移植的 OPLUS/面板/Codec 配置」。**我没有验证 5.10 techpack 是否含 lahaina 实现**，这一步是整条链上最大的未知。
3. **推测**：arch/arm64/boot/dts/vendor 与 arch/arm64/configs/vendor 的外部来源应是某个 CLO DTS/vendor-config 项目（未定位到具体 project id）。
4. **推测**：Realme GT2（RMX3311）Android 15 是否换了内核，本次无法判定（未找到对应源码仓库；其 Android 14 源码是 5.4.254）。写在这里是为了提醒：**如果后续有人看到「GT2 + 5.10」的字样，先查 RMX 编号** —— 本次就撞到过一次 RMX3551 的假阳性。

### 7.3 对方案文档的净影响（一句话）

- 方案第 0 节结论（**没有任何 SM8350 设备/厂商上过 5.10，社区也没做**）**成立且被独立复核**。
- 方案第 1 节「有利条件 1」需要把「一等公民 / 完整 SoC 支持」降级为「**内核侧真实支持，但 DTS / vendor config / techpack 全在仓库之外，且高通自家 SM8350 BSP 清单仍指向 msm-5.4**」。
- 方案第 1 节「不利条件 4（techpack 不在公开 msm-5.10 里）」**方向正确但要改措辞**：不是「高通没有」，而是「**被拆到独立仓库，需要按 SoC 选分支并手动挂载**」—— 而这恰好把 Phase 2 的风险从「缺失」变成「**分支匹配与 lahaina 覆盖未知**」。
