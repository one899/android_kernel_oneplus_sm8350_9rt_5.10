# 9RT (martini / SM8350) 从 Linux 5.4 迁移到 5.10 —— 方案 v2（经三线独立核实）

> v1 是我一个人写的。v2 由三条独立研究线核实后重写，**其中两条结论被推翻、一条被下调、发现一条当下就该处理的隐患**。
> 状态：**仅方案，未执行任何改动**。

---

## 结论先行

1. **前提不成立**：这台机器是 QGKI 设备，Android 版本与内核版本解耦。「Android 12 需要 5.10」对它不适用。
2. **没有 donor 产品**：全世界（官方 + 社区）没有任何 SM8350 设备跑 5.10。官方 BSP 清单自己都还钉在 `kernel/msm-5.4`。
3. **但技术起点比 v1 判断的好得多**：OPLUS 有成熟的 5.10 内核线（SM8450/SM8475，5.10.66→5.10.236），且它们**自带 lahaina 构建目标**。正确做法是以它为基底，不是从干净 msm-5.10 起步。
4. **真正的硬骨头从「ABI」变成了「techpack」**：ABI 只是「必须重编」，而 techpack 的 5.10 源**是否存在尚未核实**，这是整条链上最大的未知。
5. **⚠️ 附带发现（与 5.10 无关，但更紧急）**：当前 workflow 用第三方 `module.c` **把符号版本校验整个关掉了**，导致设备正在"带病加载" 64 个结构体布局不匹配的厂商模块。详见第 7 节。

---

## 0. 前提纠正（v1 保留）

「Android 12 需要 5.10」只对 GKI 设备成立。SM8350/martini 是 QGKI，实测：

| 项目 / 分支 | Android | 内核 |
| --- | --- | --- |
| OnePlusOSS `SM8350_R_11.0` | 11 | 5.4.61 |
| OnePlusOSS `sm8350_s_12.1_martini` | **12.1** | **5.4.147** |
| OnePlusOSS `sm8350_t_13.1.0_oneplus9rt` | 13.1 | 5.4.210 |
| OnePlusOSS `sm8350_u_14.0.0_oneplus9rt` | 14 | 5.4.254 |
| `sm8350_b_16.0.0_oneplus9rt`（当前基线） | 16/17 | 5.4.295 |
| LineageOS `lineage-23.2` | 16 | 5.4.302 |
| crDroid `17.0` | **17** | **5.4.302** |

**先回答：你要 5.10 是为了解决什么？** 不同答案对应不同方案，多数情况下不需要迁移：

| 真实目的 | 5.10 是否对症 |
| --- | --- |
| 卡顿 | ❌ 已实测主因是 ColorOS userspace 热管理（负载升温直接改写 `scaling_max_freq`，内核 thermal 冷却设备全程 `cur=0`） |
| 续航 | ❌ 内核功耗基础已较好（`NO_HZ_IDLE`/EAS/`MENU` idle gov/`RCU_FAST_NO_HZ`+`NOCB`/LTO+CFI 均已启用） |
| 安全补丁 | ⚠️ 升到 **5.4.302** 比换大版本划算得多 |
| 某个 5.10+ 特性 | ✅ 但也可能用子系统级 backport 解决（现有分支已在做 EEVDF） |
| GKI 合规 | ✅ 唯一真正需要 5.10 的理由，代价见第 4 节 |

---

## 1. 三条独立研究线的结论

### 线 1（lahaina-510-scout）：有没有 SM8350 的 5.10？

**结论：没有。** 全部实测停在 5.4.x（最高 5.4.302）。三个边界修正：

1. **`msm-5.10` 里 lahaina 是真实目标**（`ARCH_LAHAINA` + 5 个 lahaina 驱动源文件 + 专属 build config + 97 行模块清单），不是占位。
2. 上游 mainline 从 **v5.15** 起才有 `sm8350.dtsi`（v5.10 没有）—— 字面意义「SM8350 跑过 5.10+」不成立，要从 5.15 起算；且那是 mainline，无厂商层、不能跑 Android。
3. **高通自己的 BSP 清单（LA.UM.9.14.1 / QSSI13）拉的仍是 `kernel/msm-5.4`** —— 连官方 BSP 都没上 5.10。

社区侧：扫了 OnePlusOSS(11 分支)/LineageOS(15)/crDroid(60)/arter97(6)/dev-sm8350(12)/QRD-Development 全部分支，**无任何 5.10 移植**；martini 的 Android 16/17 内核仍是 5.4.302。

**排雷（v1 差点被骗）**：`realme_gt2-AndroidV-kernel-source` = 5.10.209 看着像「Realme GT2(SM8350) 上了 5.10」，**是假阳性** —— 该仓库实际是 RMX3551（GT2 大师探索版 = **SM8475**）。SM8350 版 GT2(RMX3311) 的 Android 14 源码仍是 5.4.254。另外 v1 任务书里两处机型前提也错了：Redmi K40 游戏增强版是天玑1200(4.14.186)、Nothing Phone (1) 是 SM7325，都不是 SM8350。

### 线 2（oplus-vendor-scout）：厂商层能借多少？

**两个官方 donor 组织**（v1 只知道第一个）：

| 仓库 | 分支 | 内核 | 设备 |
| --- | --- | --- | --- |
| OnePlusOSS/android_kernel_oneplus_sm8450 | `sm8450_s_12.1_10_pro` … `sm8450_b_16.0_oneplus_10_pro` | 5.10.66 → **5.10.236** | 一加 10 Pro (waipio) |
| OnePlusOSS/android_kernel_oneplus_sm8475 | `sm8475_s_12.1_oneplus_10t_5g` … | 5.10.81 → 5.10.226 | 一加 10T/11R (cape) |
| **oppo-source/android_kernel_oppo_sm8450** | `sm8450_s_12.1_find_x5_pro` | **5.10.66** | OPPO Find X5 Pro |

**最有价值的发现（推翻 v1 §1.3）**：

> **5.10 的 `kernel/sched/fair.c` 里 OPLUS 引用数 = 0**（311KB 全文正则，0 命中）。
> 5.4 的 fair.c 里 OPLUS 相关行 231 行、6 个直接调用。

5.10 改成 **vendor 侧注册 ACK 的 `android_rvh_*` 钩子** + `ANDROID_OEM_DATA_ARRAY(1,32)/(1,16)`，内核侧拼接只在 QCOM 自己的 WALT 文件（`walt.c`/`walt_cfs.c`/`walt_lb.c`）+ 5 行 Makefile + 10 条 symlink。

⇒ **sched_assist 这一层耦合的是「内核世代」而不是「SoC」**，可以整套借。而 lahaina 恰是同一 msm-5.10 家族的一等目标。

分层比例（估算，依据是符号依赖清单）：sched_assist **60–70% 可平移 / 20–30% 需适配 / ~10% 重做**；内核侧拼接 80% 可平移（因为 donor 树里已现成）。

### 线 3（abi-analyst）：ABI 到底是不是墙？

**5.4 编译的 .ko 能否被 5.10 加载 → 确定性回答：不可能。** 机制链：

1. `check_modstruct_version()` 先查 `module_layout` 的 CRC。已逐字 diff：5.10 的 `struct tracepoint` 新增 `static_call_key/static_call_tramp/iterator`，`struct module` 增删多处 → `__crc_module_layout` 必不同 → 第一个门就 `-ENOEXEC`。
2. 逐符号 `__crc_*` 不同（AOSP 自例：一个 `mm_struct` 改动 → 4495 个符号 CRC 变）。
3. GKI `TRIM_NONLISTED_KMI=1` 取消导出非 KMI 符号 → `Unknown symbol`。
4. 全放行也过不去：布局 + `CFI_CLANG`/`LTO_CLANG_FULL` 不匹配。

**关键修正：VERMAGIC 不是墙。** `same_magic()` 在模块带 CRC 时用 `strcspn` 跳过内核版本号段，两侧剩余段一致 → 死在 CRC，不是死在 vermagic。且两边都没有 `CONFIG_MODULE_FORCE_LOAD`，`insmod -f` 无效。

**Phase 4 从「硬墙/可能不可行」下调为「工程量最大的必做项」**：源码在树里（`techpack/audio/config/lahainaauto.conf` 里 `CONFIG_SND_SOC_BOLERO=m`、`WCD938X=m`、`WCD9XXX_CODEC_CORE_V2=m`、`WCD_MBHC=m`、`SOUNDWIRE=m`、`SWR_HAPTICS=m` 正是我们在 dmesg 里看到的那批 dlkm）。ABI 仍是「二进制不可复用」的硬事实，但它等于「必须重编」，而重编 = 把 5.4 techpack/vendor 前移到 5.10 —— **与 Phase 2/3 是同源工作，不是额外一道墙**。

---

## 2. 可行性盘点（v2 修正版）

### ✅ 有利

1. `msm-5.10` 有真实 lahaina 目标（`build.config.msm.lahaina`、`modules.list.msm.lahaina` 含真 lahaina 模块）。
2. **OPLUS 有成熟 5.10 线，且自带 lahaina 目标与 OPLUS 拼接点** → 基底现成。
3. sched_assist 从「跨 SoC 借参照」升级为「整套搬」（线 2 证据）。
4. 设备树是「搬」不是「写」：5.10 modules 仓库有 ≥40 个 lahaina DTS/DTSI；5.4 有 22 个 martini 专属 DTS（含 `lahaina-sde-display.dtsi`、`lahaina-camera-sensor-20820.dtsi`）。
5. **文档里所有模块的源码都在树/公开仓库里**（线 3 实测）。

### ❌ 不利

1. **所有公开 OPLUS/OPPO 5.10 树的 techpack 都是 stub**（5 个文件）。
   **措辞修正**：准确说是「**外置**」而非「缺失」—— `.gitignore` 语义是「除 stub 外都是外部项目」，`Kbuild` 构建时把实际存在的目录加进来。高通 BSP manifest（`QRD-Development/SM8350_BSP_Sync@LA.UM.9.14.1/target.xml`）显示 display-drivers / audio-kernel / camera-kernel / dataipa / video-driver **是 5 个独立 CLO 仓库**，分别挂到 `techpack/*`。
   **但**：该清单是 lahaina BSP，其 kernel 写死 `msm-5.4`。**5.10 的 techpack 是否存在，未核实**（display-drivers 有 `display-kernel.lnx.5.10.*` 分支；audio-kernel / camera-kernel 搜不到 lnx.5.10 分支；GitLab tree/raw 被 403/429 挡）。**这是整条链上最大的未知。**
2. 5.10 树里**没有 lahaina 的 vendor defconfig**（只有 `waipio_*` 与 `neo*`）。
3. 5.10 无 martini 设备树（martini DTS 只存在于 5.4 modules 仓库，22 个）。
4. 5.10 无 uxmem 对应物（相近的是 `misc/mm_boost_pool` 与 `mm/hybridswap_zram`，是否等价未验证）。
5. 公开的 `waipio_oplus.config` / `modules.list.oplus` 是 **0 字节空文件**。

---

## 3. 推荐路径（v2 核心变更）

> **v1 建议**：干净 msm-5.10 + 重新打 OPLUS 补丁。
> **v2 建议（推翻 v1）**：**直接以 `android_kernel_oneplus_sm8450`（5.10，家族内已带 OPLUS 拼接点与 lahaina 目标）为基底**，把目标从 waipio 换成 lahaina，再把 martini 的设备树/面板/充电 DTS 从 5.4 前移。

这样 Phase 3 的工作量从「补回 OPLUS 补丁」降为「加一个 lahaina 目标」——**省一个数量级**。

### 最小验证（半天，成本极低，可证伪）

1. 用 `android_kernel_oneplus_sm8450` @ `b_16.0_oneplus_10_pro`（5.10.236）+ 同分支 modules 仓库，按 `kernel_platform/oplus/build/oplus_build_kernel.sh` 试编 **waipio** 目标 —— 验证「OPLUS 5.10 厂商层能否在自己家门口编出来」。
2. 通了就把 `MSM_ARCH` 换成 `lahaina`、补一份 lahaina vendor defconfig，看 `oplus_perf_sched` / `oem_sched` / `tuning` 三个 symlink 目录能否一起编过 —— **这是「可平移」结论的直接证伪点**。

---

## 4. 阶段与工期（v2）

| 阶段 | 内容 | 估算 | 变化 |
| --- | --- | --- | --- |
| Phase 0 | 目标确认 + **源码覆盖率**测绘（判据已改，见第 6 节） | 半天 | 判据变更 |
| Phase 1 | 以 sm8450 5.10 树为基底，切到 lahaina 目标，编出可启动内核 | 3–7 天 | **大幅下调**（v1: 1–2 周） |
| Phase 2 | **techpack 前移**（audio 566 / display 396 / camera 669 / dataipa 148 / video 60 文件） | **不可估** | ⬆️ 现在是最大未知 |
| Phase 3 | OPLUS 厂商层落地到 lahaina | 1–2 周 | **下调**（v1: 2–4 周） |
| Phase 4 | 用户空间 ABI：重编并安装全部模块 | 工程量最大但**不是墙** | **下调**（v1: 「可能不可行」） |
| Phase 5 | 集成、回归 | 2–4 周 | — |

**唯一真正的路障是 Phase 2**：如果 5.10 的 lahaina techpack 不存在（且 CLO 不公开），那 5.4 的 1839 个 techpack 源文件就得自己前移三大驱动栈（DRM/KMS、ASoC/ALSA、V4L2），且 OPLUS 的私有定制（`techpack/display/oplus`、相机 tuning）没有 5.10 可对照。

---

## 5. 决策判据（v2 修改）

> **v1 判据**：「换 5.10 后失效模块比例 > 80% 就不值得做」→ **此判据已废**。
> **v2 判据**：**「源码覆盖不到的 .ko 占比」**。

理由（线 3）：模块失效不等于功能失效 —— 只要源码在，就能重编。所以真正的问题是：

```bash
# 设备上所有 .ko 的 basename 集合
#  减 源码树里能产出的 .ko 集合（out/ 下 find -name '*.ko'）
# = 源码覆盖不到的部分
```

脚本已备好：`_meta/research/p0-census.ps1`（**注意：其 L33-35 的启发式要把 `disagrees` 当「加载失败」是错的**，见第 7 节，需要改）。

---

## 6. 替代路径（不变）

按实测证据排序，全部零成本或低成本：

1. **userspace 电源模式**：已实测开启高性能模式把 A78 上限从 1996800 提到 2227200、X1 从 2150400 提到 2496000，并把 A55 保底从 499200 提到 1209600。
2. **thermal-engine / oplus.performance.hal**：负载升温后直接改写 `scaling_max_freq`（A78 掉到 1670400、X1 掉到 1785600），内核 thermal 完全未介入 —— 这才是卡顿主因。
3. **子系统级 backport**：现有分支路线（EEVDF、UX 不公平调度），单特性 1–3 天，收益可实测。
4. **升到 5.4.302**：拿最新安全补丁，成本是重编一次内核。
5. **修模块问题**：见第 7 节 —— 这是当下最高性价比的一项。

---

## 7. ⚠️ 附带发现：当前内核正在「带病加载」厂商模块

**这条与 5.10 无关，但比 5.10 更值得优先处理。**

### 7.1 事实（已由我独立复核）

workflow 在编译前用第三方 release 资产**整文件替换** `kernel/module.c`（`wf_original.yaml` L79-86）。我把该文件拉下来逐字核对，其 `check_version()`：

```c
bad_version:
	pr_warn("%s: disagrees about version of symbol %s\n", info->name, symname);
	return 1;          /* ← 上游 v5.4/v5.10/ACK 全都是 return 0; */
```

而三个调用点（`L1486`、`L1366`、`L3997`）**全部是 `if (!check_version(...))`**。它永远返回 1 ⇒ **符号版本校验被整个废掉**，CRC 不匹配只打一条 warn 然后照常加载。仓库自己那份 `kernel/module.c` 是干净的 `return 0;`。

### 7.2 后果（线 3 实测）

- 3183 行 / **1079 个符号** / **64 个模块** / 280 次 `module_layout`。
- 决定性判别：把「原型不含结构体」的符号（`printk`/`jiffies`/`__kmalloc`/`schedule`/`vmalloc`…）撞这份名单 → **全部 0 次**。⇒ 不是工具链差异，是**两边结构体布局真的不同**。
- **这些模块没有「加载失败」—— 它们是 LIVE 的**（`Modules linked in` 里无 `+`），代码在跑（`wcd_mbhc_init: leave ret 0`、`swr_haptics_probe rc=0`）。
  ⚠️ **此处推翻了 v1 与仓库 `OPTIMIZATIONS.md` §7 的说法**（那里面写「haptic / bolero_cdc_dlkm / … 加载失败」）。
- 280 次 = 64 模块 × 平均 4.4 次 modprobe 重试。

**「带病加载」比「加载失败」更危险**：结构体布局不匹配的模块在跑，可能造成内存踩踏、随机异常、间歇性卡顿。

### 7.3 为什么会在这种状态下跑

因为它**是刻意的**：不关掉 CRC 校验，重编的内核根本加载不了 ROM 的原厂模块，机器起不来。这是「重编内核 + 保留原厂 ROM」路线的必要妥协。

### 7.4 正确的修法（而且工作流已经做了一半）

workflow **已经在编并收集匹配的模块**：

```yaml
mkdir -p AnyKernel3/modules/vendor/lib/modules
find kernel/msm-5.4/out/ -name "*.ko" -exec cp {} AnyKernel3/modules/vendor/lib/modules/ \;
```

但 `anykernel.sh` 第 9 行是 **`do.modules=0`** —— AnyKernel3 直接跳过模块安装。**上面那个 find/cp 是死代码。** 再加上用户刷的是 `boot.img`（`magiskboot repack` 只含 Image），模块从未被替换过。

**要做的**：
1. `do.modules=1`（注意脚本同时 `do.systemless=1`，会走 Magisk/KSU systemless 路线；音频/wlan 这类 post-fs-data 后加载的通常可行，早期模块有风险，需实测）
2. 确认模块落盘路径与原厂同名文件所在目录一致（先 `find /vendor /vendor_dlkm -name '*.ko'`）
3. 处理只读/verity（关 verity 后 rw / systemless bind-mount / 重打包 vendor(_dlkm)）
4. **必须是同一次构建产出的 .ko**（同树 / 同 `out/.config` / 同补丁）—— 判据是「内核与模块的类型布局一致」，不是「自己编的」就行
5. SELinux label / 权限

做完这些，第 7.1 节那个 `module.c` 后门就不再需要了。

---

## 8. 一页版行动建议

```
现在就该做（半天，零风险）：
  ① 改 p0-census.ps1 的判据：把「加载失败」改成「源码覆盖不到的 .ko 占比」
  ② 跑 Phase 0 测绘，拿到真实数字
  ③ 评估把 do.modules=1 打通 —— 这是修复「带病加载」的正路

决定要不要做 5.10 之前（半天）：
  ④ 用 sm8450 5.10 树试编 waipio，再换 lahaina 目标
     —— 过了才谈得上后续；过不了就直接放弃 5.10

最大的未知（需要专门去挖）：
  ⑤ 5.10 的 lahaina techpack 到底存不存在
     不存在 ⇒ Phase 2 = 自己前移三大驱动栈，工期不可估 ⇒ 建议放弃
     存在   ⇒ 整件事从「不可行」变成「6 个月量级的大工程」

不建议：在 ⑤ 有答案之前写任何一行 5.10 代码。
```

---

## 附：研究产出

| 文件 | 内容 | 行数 |
| --- | --- | --- |
| `_meta/research/r1-lahaina-510.md` | 逐设备核实、msm-5.10 lahaina 证据原文、社区扫描、假阳性排雷 | 542 |
| `_meta/research/r2-oplus-510.md` | OPLUS 5.10 donor 清单、厂商层结构、7 个符号级耦合点、分层比例 | 360 |
| `_meta/research/r3-abi-analyst.md` | ABI 机制链、VERMAGIC 修正、3183 行成因、`module.c` 后门、GKI KMI 核实 | 493 |
| `_meta/research/p0-census.ps1` | Phase 0 测绘脚本（**判据需按第 5 节修改**） | 70 |

三份报告均严格分了「核实到 / 推测 / 未找到」三节。凡百分比均为估算并标注依据。
