# 5.4 → 5.10 可行性评估（证据汇总版）

> 汇总三路并行研究 + 13 轮构建实测 + 实机验证
> 结论口径：**路径清晰、无已知死结、风险集中在第 6 层**

---

## 一、结论先行

| 问题 | 回答 |
| --- | --- |
| 5.10 能不能编出来？ | ✅ **能**（已实测，GKI 阶段完整产出 Image+modules+Image.lz4+Module.symvers） |
| 技术上门开了吗？ | ✅ 三个原判「硬墙」全部证伪 |
| 能不能肯定成功？ | ❌ **不能肯定** —— 风险集中在「内核↔用户态接口」层 |
| 现在该放弃吗？ | ❌ **不该** —— 唯一未知项可用 1-2 周勘测清楚 |

---

## 二、三个原判「硬墙」的证伪

### 墙 1：techpack 拿不到 5.10 源码 → ❌ 不成立

**CLO 上有 5.10 + lahaina 的完整 techpack 源码：**

| 子系统 | 仓库 | 分支 | lahaina 配置证据 |
| --- | --- | --- | --- |
| display | `clo/la/platform/vendor/opensource/display-drivers` [id=13664] | `display-kernel.lnx.5.10.r11-rel` | `config/lahainadisp.conf` |
| audio | `clo/la/platform/vendor/qcom/opensource/audio-kernel-ar` [id=22253] | `audio-kernel.lnx.5.10.r6-rel` | `config/lahainaauto.conf` |
| camera | `clo/la/platform/vendor/opensource/camera-kernel` [id=13577] | `CAMERA.LA.2.0.c26` | `config/lahaina.mk` |

### 墙 2：厂商层与 SoC 耦合，只能借参照 → ❌ 不成立（对 sched_assist 层）

**厂商层耦合的是「内核世代」，不是 SoC：**

| | 5.4 | 5.10 |
| --- | --- | --- |
| `kernel/sched/fair.c` 里 OPLUS 引用 | **231 行** | **0 行** |
| 实现形态 | `sched_assist/sched_assist_common.c` (59.6 KB) | `sched/sched_assist/sa_*.c` **19 文件 ≈ 95 KB** |
| 集成方式 | 直接打补丁进 fair.c + symlink 头文件 | vendor 侧注册 `android_rvh_*` 钩子 + `ANDROID_OEM_DATA_ARRAY` |

→ **5.10 的 sched_assist 是一次重写，可以整套搬。**

### 墙 3：ABI 硬墙 + 厂商源码不公开 → ⚠️ 严重程度需下调

- 5.4 编的 .ko **确实无法**在未改动的 5.10 GKI 上加载（`check_modstruct_version()` 第一个符号就挂，-ENOEXEC）
- **但源码是公开的** —— 用户 CI 已经在编这些模块
- 所以这是**工程量**（把 vendor/techpack 前移），**不是原理阻塞**

> 附带纠正：旧文档里「6 个模块加载失败」是**误读** —— 它们在日志里都是 LIVE 状态。真正原因是 `anykernel.sh` 里 `do.modules=0`，模块一个都没装。

---

## 三、硬件参数：已完全解决

三路独立来源交叉验证（实机 / 5.4 DTS / LineageOS），四子系统全部到型号级：

| 子系统 | 结论 | 决定性证据 |
| --- | --- | --- |
| **屏幕** | Samsung **AMS662ZS01**(DVT)，CMD+DSC，1080×2400，60/120Hz，4-lane 双 DSI | DRM `card0-DSI-1/modes` = `1080x2400x60x183333cmd` |
| **充电** | **65W SuperVOOC**，`chg_ops=plat-pmic`，VOOCPHY 在 ADSP，gauge=ZY0603，**无无线充** | DT `vooc-max 10V/6.5A` + dmesg `ui_power_show: 65000` |
| **触摸** | **Synaptics S3908** @ I²C-5 / 0x4B | `5-004b name=synaptics-s3908` + input7 `touchpanel` |
| **摄像头** | **IMX766 + IMX481 + OV02B + IMX471**，OIS=ROHM BU63169 | `com.qti.sensor.*.20820.so` |

### 两个移植期必踩的坑

1. **面板**：DT 的 `qcom,dsi-default-panel` 指向 **9 Pro 的 1440×3168**，真面板靠 **kernel cmdline** `msm_drm.dsi_display0` 运行时覆盖。**丢掉这个机制就会点亮错屏。**
2. **触摸**：DT 的 `qcom,i2c-touch-active="st,fts"` 是**候选提示**，与实际绑定的 `synaptics-s3908` **不一致**。照搬会选错驱动。

---

## 四、lahaina vendor defconfig：已产出

**关键发现（纠正一个流传的说法）**：5.10 树里的 `build.config.msm.lahaina` 和 `modules.list.msm.lahaina` 是**残留 stub** —— 与 CLO c27 **SHA256 逐字节相同**，清单里列着 `hh_virt_wdt.ko`(Haven)，而 5.10 树里根本没有 `hh_*` 源码（Haven 已被 Gunyah 取代）。

**CLO 确认没有现成的 lahaina vendor config**（全站唯一 5.10 内核仓库 `clo/la/kernel/msm-5.10` [id=29371]，1274 个分支全扫，lahaina/8350/martini **0 命中**）。

**但 lahaina 驱动源码齐全**：gcc/dispcc/camcc/gpucc/videocc/debugcc-lahaina + qnoc-lahaina + pinctrl-lahaina + phy-qcom-ufs-qmp-v4-lahaina + llcc(lahaina_data 已并入 llcc_qcom.c) + clk-aop-qmp + rpmhpd + refgen

**产出**：`lahaina_GKI.config`（752 行 / 21824 B）= 5.10 树的 `waipio_GKI.config` 逐行照抄 + **新增 13 / 删除 31**，每条都有理由与出处。

构建命令：
```bash
cd <work>/kernel_platform && export ROOT_DIR=$PWD
BUILD_CONFIG=msm-kernel/build.config.msm.lahaina VARIANT=gki ./build/build.sh
```

---

## 五、构建链路：已实测跑通

13 轮探测，每次失败都是「缺一个工具/配置」，不是架构问题：

| 轮次 | 突破 |
| --- | --- |
| v1–v3 | 目录布局 + PATH 修复 |
| v4–v7 | 工具链（gcc 交叉、clang、wrapper 清理） |
| **v8** | **★ GKI 阶段编出 Image + vmlinux** |
| v9–v10 | external/dtc + clang 追踪修正 |
| **v11** | **★ ABI 检查通过 → GKI 完整成功**（Image+modules+Image.lz4+Module.symvers） |
| v12 | DTBO 编译成功，卡 dtbo.img 打包工具 |
| v13 | 补 libufdt（进行中） |

**教训**：工具仓库（`tools/libufdt` / `prebuilts/build-tools` / `external/dtc`）是同一类问题，应当一次补齐，而不是一轮补一个。

---

## 六、★ 真正的风险：内核 ↔ 用户态接口

**这是设备侧证据揭示的、之前被漏掉的一层。**

```
/vendor/lib/modules/5.4-gki/        ← 目录名直接写着内核版本
  camera.ko        9.6 MB
  msm_drm.ko       2.8 MB
  haptic.ko / aw8697.ko / uff_fp_driver.ko ...
  几十个 *_dlkm.ko
```

| 层 | 可否重建 | 说明 |
| --- | --- | --- |
| kernel 侧驱动 | ✅ 源码有 | `camera.ko` / `msm_drm.ko` 可重编 |
| **用户态 HAL** | ❌ **预编译** | `com.qti.sensor.imx766.20820.so` 等按 5.4 vendor interface 编的 |
| DT 私有属性 | ⚠️ 依赖 | `oplus,*` / `track,*` / `prevention,*` / `touchpanel,*` / `pxlw,*` |

**关键**：摄像头 sensor 型号**不在 DT 里**（节点无 `reg`），而在用户态库 `com.qti.sensor.*.20820.so`。

→ **内核侧驱动换代会否打断用户态 HAL？** 这取决于高通私有 ioctl（`VIDIOC_MSM_*`）和 V4L2/DRM/ALSA 接口在 5.4→5.10 的差异。**HAL 不可重编，这是唯一可能致命的一层。**

---

## 七、风险分层总表

| 层 | 状态 | 性质 |
| --- | --- | --- |
| 1. 构建链路 | ✅ 已证明 | 已解决 |
| 2. 硬件参数 | ✅ 已解决（型号级 + 实机验证） | 已解决 |
| 3. lahaina defconfig | ✅ 已产出，待编译验证 | 小 |
| 4. techpack 前移 | ⚠️ CLO 有 5.10+lahaina 源码，1839 文件逐接口翻译 | 大工程量 |
| 5. 厂商层接线 | ⚠️ 文件在、接线没做 | 中等工程量 |
| 6. **内核↔用户态接口** | ❌ **未经勘探，可能致命** | **未知** |
| 7. 无参照物（无 SM8350 上过 5.10） | ❌ 成本放大器 | 持续 |

**实例（第 5 层的形状）**：
```
charger_ic/oplus_battery_sm8350.c     ← 文件存在
charger_ic/Makefile                   ← 完全没引用它
5.4 的 CONFIG_OPLUS_CHG_OP9RT_PMIC_VOOCPHY → 5.10 Kconfig 里不存在
```
→ 厂商层不是「缺源码」（死路），而是「源码在、接线没做」（活路，但要一条条接）。

---

## 八、止损点（修正版）

```
【已完成】硬件参数 ✅   lahaina defconfig ✅   构建链路 ✅
    ↓
【B1】切 lahaina 编译一次（1-2 轮）
    通过 → 证明"这棵树能产出 martini 可用的内核"
    ↓
【B2】★ 双向接口勘测（1-2 周）—— 唯一一锤定音的一步
    方向 1 (kernel-internal):  5.4 techpack API → 5.10
    方向 2 (kernel↔userspace): V4L2/DRM/ALSA 私有 ioctl 差异  ← 重点
    ↓
【判据】
    方向 2 差异小 → 用户态 HAL 可复用 → 可行, 4-6 个月
    方向 2 差异大 → 用户态 HAL 成为硬墙 → 需重写 HAL 或放弃
    ↓
【C】能 boot 到 Android
```

**B2 是分水岭。** 方向 2 是重点，因为方向 1 有 CLO 源码兜底，方向 2 没有。

---

## 九、最终判断

> **5.10 在技术上肯定能编出来**（已实测）。
> **但「能不能用」取决于一个尚未勘探的具体问题** —— 内核↔用户态接口在 5.4→5.10 的断裂程度。
>
> **现在不该放弃**，因为所有已知死结都已证伪，唯一未知项可在 1-2 周内勘测清楚。
> **但如果勘测发现用户态接口断裂严重，就该有勇气放弃。**

---

## 十、证据索引

| 文件 | 内容 |
| --- | --- |
| `martini-hw-device.md` | 实机采集（47 KB / 1018 行），每项含可复现命令 |
| `martini-hw-source.md` | 源码侧提取（21 KB / 9 节），每项含文件+行号 |
| `lahaina_GKI.config` | 5.10 lahaina vendor defconfig（752 行） |
| `lahaina-defconfig.md` | 配置分析报告（483 行） |
| `r1-lahaina-510.md` | SM8350 是否存在 5.10 内核的穷举核实 |
| `r2-oplus-510.md` | OPLUS 厂商层跨 SoC 借用可行性 |
| `r3-abi-analyst.md` | 5.4→5.10 模块 ABI 断裂论证 |
| `official-docs-notes.md` | 与 AOSP 官方构建文档的对照 |
| `build-system-notes.md` | 构建系统笔记 |
