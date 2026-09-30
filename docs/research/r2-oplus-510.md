# R2 — 从别的 SoC 的 OPLUS 5.10 内核能借多少？（独立研究线 2）

> 任务：task-2 ｜ 执行：oplus-vendor-scout ｜ 唯一产出文件：`_meta/research/r2-oplus-510.md`
> 方法：GitHub 官方仓库（OnePlusOSS / oppo-source）+ raw.githubusercontent 取证 + 部分克隆（blob:none）拿完整文件清单 + 与本地 5.4 树文件（`_meta/mig/`、`_meta/vnd/`）对照。
> 所有引用的行号来自对应分支的原始文件；凡是估算/推断都在第 ⑦ 节标出。

---

## ① 结论摘要

1. **OPLUS 的 5.10 内核树存在，而且不止一个 SoC。** SM8450（waipio，一加 10 Pro / OPPO Find X5 Pro）、SM8475（cape，一加 10T / 11R / Ace Pro）、MT6895 / MT6983（天玑）都有公开的 5.10 分支。版本证据见第 ② 节（Makefile VERSION/PATCHLEVEL/SUBLEVEL）。
   反例确认：SM8350 没有任何 5.10 分支（OnePlusOSS `android_kernel_oneplus_sm8350` 全部 11 个分支都是 5.4；`oppo-source/android_kernel_oppo_sm8350`（Find X3 Pro）是 5.4.147）。SM7325 是 5.4.210/5.4.254，SM8550 是 5.15.74 —— 都**不是** 5.10。

2. **厂商层源码不在内核仓库里，在配套的 `..._modules_and_devicetree_...` 仓库的 `vendor/` 下，且目录名带内核版本后缀**：`vendor/oplus/kernel/oplus_performance_5.10/`。内核仓库里对应位置只有 **symlink**（例如 `kernel/oplus_perf_sched -> ../../../vendor/oplus/kernel/oplus_performance_5.10/sched/`）。这正是方案里 Phase 3 缺的那一块。

3. **5.10 的 sched_assist 是一次重写，不是 5.4 代码的前移。** 5.4 是 `sched_assist/sched_assist_common.c`（59.6 KB，靠把 fair.c 打补丁 + symlink 头文件编译进内核镜像）；5.10 变成 `sched/sched_assist/sa_common.c / sa_fair.c / sa_binder.c / …`（19 个文件 ≈ 95 KB），集成方式是 **vendor 侧注册 Android GKI 的 `android_rvh_*` 钩子 + 用 `ANDROID_OEM_DATA_ARRAY` 存私有数据**。

4. **最硬的一条证据：5.10 内核的 `kernel/sched/fair.c` 里 OPLUS 引用数 = 0**（对 `oplus|sched_assist|oem_fair|CONFIG_OPLUS` 全量匹配，0 命中）。5.10 的内核侧拼接点被搬到 **QCOM 自己的 WALT 文件**（`kernel/sched/walt/walt.c`、`walt_cfs.c`、`walt_lb.c`）+ 5 行 Makefile + symlink。
   对比 5.4 树：本机 `_meta/mig/kernel__sched__fair.c.base` 里 OPLUS 相关行 **231 行**，fair.c 直接调用 6 个 vendor 函数（`sched_assist_update_record`、`sched_assist_pick_next_entity`、`sched_assist_spread_tasks`、`sched_assist_task_misfit`、`oplus_check_preempt_wakeup_in_list`、`oplus_cap_systrace_c`）。

5. **对 sched_assist 这一层，耦合的是「内核世代」而不是「SoC」**：它依赖 (a) msm-5.10 的 WALT API（`struct walt_task_struct`、`task_util()`、`is_reserved()`、`sched_capacity_margin_down/up`）、(b) ACK 5.10 的 vendor hook 面、(c) `ANDROID_OEM_DATA_ARRAY` 布局。而 **lahaina 恰好是同一个 msm-5.10 家族里的一等目标**：OPLUS 自己的 5.10 内核树里就有 `build.config.msm.lahaina`、`modules.list.msm.lahaina`（里面是真的 lahaina 模块：`gcc-lahaina.ko`、`dispcc-lahaina.ko`、`qnoc-lahaina.ko`、`pinctrl-lahaina.ko`、`phy-qcom-ufs-qmp-v4-lahaina.ko`）、lahaina 的 clk/icc/phy/pinctrl 驱动；配套 devicetree 仓库里有整套 lahaina 参考 DTS。
   → **这条推翻方案 §1.3「厂商层里有大量与 SoC 耦合的部分 → 只能跨 SoC 借参照」对 sched_assist 这一层的判断：sched_assist 可以整套搬，而不是"借参照"。**（对充电/触控/面板/techpack 这类硬件相关的层，方案的判断仍然成立。）

6. **但 techpack 是死路：所有公开的 OPLUS/OPPO 5.10 树里 techpack 都是 stub（5 个文件：`.gitignore`、`Kbuild`、`stub/*`）。** 方案 §1.4 的结论被加强：不存在「从 SM8450 的 OPLUS 5.10 树借 techpack」这条路。你能拿到的那份 1839 文件的 5.4 techpack（见 ⑤）必须自己前移，或另找 5.10 BSP 源。

7. **Phase 4（ROM 里预编译 .ko 的 ABI）不受本报告影响，仍是硬墙。** 本报告只回答「源码层能借多少」。

---

## ② OPLUS 5.10 内核仓库清单（含分支名与内核版本证据）

版本读法：raw 拉各分支根目录 `Makefile`，取 `VERSION/PATCHLEVEL/SUBLEVEL`。

### 2.1 高通 5.10（第一档 donor）

| 仓库 | 分支 | 内核版本 | 设备世代 | 证据链接 |
| --- | --- | --- | --- | --- |
| OnePlusOSS/android_kernel_oneplus_sm8450 | `oneplus/sm8450_s_12.1_10_pro` | **5.10.66** | 一加 10 Pro（waipio / SM8450） | [Makefile](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8450/blob/oneplus/sm8450_s_12.1_10_pro/Makefile) |
| 同上 | `oneplus/sm8450_t_13.0_10pro` | 5.10.101 | — | — |
| 同上 | `oneplus/sm8450_t_13.1.0_10pro` | 5.10.136 | — | — |
| 同上 | `oneplus/sm8450_u_14.0.0_oneplus_10pro` | 5.10.209 | — | — |
| 同上 | `oneplus/sm8450_v_15.0.0_oneplus_10_pro` | 5.10.226 | — | — |
| 同上 | `oneplus/sm8450_b_16.0_oneplus_10_pro` | 5.10.236 | — | — |
| OnePlusOSS/android_kernel_oneplus_sm8475 | `oneplus/sm8475_s_12.1_oneplus_10t_5g` | **5.10.81** | 一加 10T（cape / SM8475） | [Makefile](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8475/blob/oneplus/sm8475_s_12.1_oneplus_10t_5g/Makefile) |
| 同上 | `oneplus/sm8475_t_13.1.0_11r_5g` | 5.10.168 | — | — |
| 同上 | `oneplus/sm8475_u_14.0.0_oneplus_11r_5g` | 5.10.209 | — | — |
| 同上 | `oneplus/sm8475_v_15.0.0_oneplus_11r` | 5.10.226 | — | — |
| **oppo-source/android_kernel_oppo_sm8450** | `oppo/sm8450_s_12.1_find_x5_pro` | **5.10.66** | OPPO Find X5 Pro（同 SM8450） | [Makefile](https://github.com/oppo-source/android_kernel_oppo_sm8450/blob/oppo/sm8450_s_12.1_find_x5_pro/Makefile) |
| OnePlusOSS/android_kernel_common_oneplus_sm8450 | `oneplus/sm8450_s_12.1_10_pro` | 5.10.66（GKI common） | 配套 GKI 树 | [Makefile](https://github.com/OnePlusOSS/android_kernel_common_oneplus_sm8450/blob/oneplus/sm8450_s_12.1_10_pro/Makefile) |

**注意 oppo-source 这个组织是本报告新发现的第二 donor 源**（方案文档里没有）：它是 OPPO 官方开源组织，`android_kernel_oppo_sm8450`（Find X5 Pro，5.10.66）与 `android_kernel_modules_and_devicetree_oppo_sm8450` 都在，且后者的 `vendor/oplus/kernel/oplus_performance_5.10/sched/sched_assist/Makefile` 存在（[链接](https://github.com/oppo-source/android_kernel_modules_and_devicetree_oppo_sm8450/blob/oppo/sm8450_s_12.1_find_x5_pro/vendor/oplus/kernel/oplus_performance_5.10/sched/sched_assist/Makefile)）—— 说明 `oplus_performance_5.10` 是 OPLUS 全线（OnePlus + OPPO）统一的目录约定，不是一加专属。

### 2.2 联发科 5.10（对照用，架构不同不适合直接做 donor）

| 仓库 | 分支 | 内核版本 |
| --- | --- | --- |
| OnePlusOSS/android_kernel_5.10_oneplus_mt6895 | `oneplus/mt6895_s_12.1_oneplus_ace_race` | 5.10.66 |
| OnePlusOSS/android_kernel_5.10_oneplus_mt6983 | `oneplus/mt6983_t_13.0_oneplus_ace_2v` | 5.10.110 |
| oppo-source/android_kernel_5.10_oppo_mt6983 | `oppo/mt6893_S_12.1_findx5_pro` | 未逐一取版本（同代 5.10） |

### 2.3 明确**不是** 5.10 的（用来钉死"SM8350 没有 5.10 厂商层"）

| 仓库 / 分支 | 内核版本 | 说明 |
| --- | --- | --- |
| OnePlusOSS/android_kernel_oneplus_sm8350 全部 11 个分支（`oneplus/SM8350_R_11.0` … `oneplus/sm8350_u_14.0.0_oneplus9rt`） | 抽样 u_14.0.0 = **5.4.254** | 无 5.10 分支 |
| oppo-source/android_kernel_oppo_sm8350 `oppo/sm8350_s_12.1_find_x3_pro` | **5.4.147** | Find X3 Pro，同为 SM8350 |
| one899/android_kernel_oneplus_sm8350_9rt_295bpf `oneplus/sm8350_b_16.0.0_oneplus9rt`（当前 CI 基线） | **5.4.295** | 本机方案文档的基线树 |
| OnePlusOSS/android_kernel_oneplus_sm7325 `oneplus/sm7325_t_13.1.1_nordce3` / `u_14.0.0_nordce3` | 5.4.210 / 5.4.254 | Nord CE3 不是 5.10 |
| OnePlusOSS/android_kernel_oneplus_sm8550 `oneplus/sm8550_t_13.1.0_oneplus11` | **5.15.74** | 一加 11 是 5.15，不是 5.10 |

> 结论：**"SM8350 没有 5.10 厂商层"成立**，而且比方案里写得更强 —— SM8350 在 OPLUS 和 OPPO 两边都只做到 5.4。

---

## ③ vendor/oplus/kernel/oplus_performance 结构对比

### 3.1 位置：不在内核仓库，在 modules 仓库

- 内核仓库根目录（`android_kernel_oneplus_sm8450` @ s_12.1）确实有 `techpack/`，但只有 stub；**没有** `vendor/`。见 [仓库根](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8450/tree/oneplus/sm8450_s_12.1_10_pro)。
- 厂商层在 [android_kernel_modules_and_devicetree_oneplus_sm8450](https://github.com/OnePlusOSS/android_kernel_modules_and_devicetree_oneplus_sm8450/tree/oneplus/sm8450_s_12.1_10_pro)，根目录只有两个：`kernel_platform/` 和 `vendor/`。
- 5.4 侧同理：[android_kernel_modules_and_devicetree_oneplus_sm8350](https://github.com/OnePlusOSS/android_kernel_modules_and_devicetree_oneplus_sm8350/tree/oneplus/sm8350_s_12.1_op9) 根目录是 `kernel/msm-5.4` + `vendor/{oplus,qcom}`。

### 3.2 目录结构（实测文件清单，非猜测）

**5.4（sm8350，`vendor/oplus/kernel/oplus_performance/`，184 个文件，27 个扁平子目录）**

```
cpufreq_bouncing/ foreground_io_opt/ gloom/ gloom_new/ healthinfo/{blk,fs,ion,main,mm}/
im/ input_boost/ iomonitor/ ion_boost_pool/ lowmem_dbg/ memleak_detect/{ion_track,malloc_track,oplus_svelte,stack_depot,svelte,task_mem}/
memory_isolate/ multi_freearea/ multi_kswapd/ oplus_mm/{process_reclaim,zram_opt}/ oplus_nandswap/
process_reclaim/ sched_assist/ special_opt/ task_cpustats/ tpd/ tpp/ uifirst/ uxio_first/
vm_anti_fragment/ zwb_handle/
```

**5.10（sm8450，`vendor/oplus/kernel/oplus_performance_5.10/`，99 个文件，先按子系统分 4 层）**

```
fs/          （Kconfig, Makefile）                       ← 对应内核 fs/oplus_perf_fs symlink
mm/          dump_tasks_mem/ gloom/ hybridswap_zram/{hybridswap*,zram_drv,zcomp}
             memleak_detect/ process_reclaim/ zram_opt/  ← 对应 mm/oplus_perf_mm symlink
misc/        mm_boost_pool/oplus_boost_pool.c
             sched_assist/{oem_fair.c,trace_oem_sched.h} ← 对应 kernel/sched/walt/oem_sched symlink
             sched_input_boost/{frame_boost_group,frame_info,input_boost,sched_assist_ofb,webview_boost} ← kernel/sched/walt/tuning symlink
sched/       healthinfo/ task_cpustats/ task_load/
             sched_assist/{sa_common,sa_fair,sa_binder,sa_exec,sa_mutex,sa_rwsem,sa_sysfs,sa_workqueue,sched_assist}.{c,h}
                           + trace_sched_assist.h + Makefile  ← 对应 kernel/oplus_perf_sched symlink
             Kconfig, Makefile
```

对照表（文件名级别）：

| 5.4（sched 相关） | 5.10（sched 相关） | 关系 |
| --- | --- | --- |
| `sched_assist/sched_assist_common.c` (59,614 B) | `sched/sched_assist/sa_common.c` (22,536 B) + `sa_fair.c` (13,506 B) + `sa_sysfs.c` … | **拆分重写**，不是改名 |
| `sched_assist/sched_assist_futex.c` | 无（改用 ACK 的 mutex/rwsem hooks） | 逻辑并入 sa_mutex/sa_rwsem |
| `uifirst/uifirst_sched_common.c` (24,528 B) | 无同名文件；能力散进 `sa_fair.c`（ux 列表 / UX 任务） | 重写 |
| `input_boost/{frame_boost_group,frame_info,sched_assist_ofb,webview_boost}.c` | `misc/sched_input_boost/` 同名文件（frame_boost_group / frame_info / sched_assist_ofb / webview_boost） | **同名同职责，是最接近"可平"的一块**（只核对了文件名，未逐行比对内容） |
| （5.4 无） | `misc/sched_assist/oem_fair.c` (5,766 B) | 5.10 新增：编到 `kernel/sched/walt/` 里的 WALT 侧负载均衡/扩散逻辑 |
| `sched_assist/sched_assist_slide.c`、`task_cpustats/task_sched_info.c`、`tpd/` | `sched/task_load/`、`sched/healthinfo/` | 位置与实现都变了 |

代码体量（我按文件字节实测，非估算）：

| 目录 | 文件数 | 字节 |
| --- | --- | --- |
| 5.4 `oplus_performance/sched_assist/` | 18 | 96,810 |
| 5.4 `oplus_performance/uifirst/` | 15 | 41,601 |
| 5.4 `oplus_performance/input_boost/` | 13 | 57,778 |
| **5.4 sched 三块合计** | 46 | **196,189** |
| 5.10 `oplus_performance_5.10/sched/sched_assist/` | 19 | 95,146 |
| 5.10 `misc/sched_assist/oem_fair.c` | 1 | 5,766 |

> 即：5.10 这一层**总量不比 5.4 小**（还要加上 5.10 的 sched_input_boost 与 task_load/healthinfo），但组织形式完全不同 —— 不是"精简"，是"换架构"。

### 3.3 内核侧的"拼接图"（5.10 的核心机制，也是移植时的真实工作量所在）

内核仓库里的 **symlink**（实测 symlink 内容，mode 120000）：

| 内核仓库路径 | 指向 | 证据 |
| --- | --- | --- |
| `kernel/oplus_perf_sched` | `../../../vendor/oplus/kernel/oplus_performance_5.10/sched/` | [link](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8450/blob/oneplus/sm8450_s_12.1_10_pro/kernel/oplus_perf_sched) |
| `kernel/sched/walt/oem_sched` | `../../../../../vendor/oplus/kernel/oplus_performance_5.10/misc/sched_assist/` | [link](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8450/blob/oneplus/sm8450_s_12.1_10_pro/kernel/sched/walt/oem_sched) |
| `kernel/sched/walt/tuning` | `../../../../../vendor/oplus/kernel/oplus_performance_5.10/misc/sched_input_boost/` | [link](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8450/blob/oneplus/sm8450_s_12.1_10_pro/kernel/sched/walt/tuning) |
| `mm/oplus_perf_mm` | `../../../vendor/oplus/kernel/oplus_performance_5.10/mm/` | 同上（120000） |
| `fs/oplus_perf_fs` | `../../../vendor/oplus/kernel/oplus_performance_5.10/fs/` | 同上 |
| `net/oplus_modules` | `../../../vendor/oplus/kernel/network` | 同上 |
| `drivers/soc/oplus/{power,system,thermal}` | `../../../../../vendor/oplus/kernel/{power,system,thermal}` | 同上 |
| `include/soc/oplus/system` | `../../../../../vendor/oplus/kernel/system/include` | 同上 |
| `drivers/input/touchscreen/oplus_touchscreen_v2` | `../../../../../vendor/oplus/kernel/touchpanel/oplus_touchscreen_v2/` | 同上 |
| `arch/arm64/boot/dts/vendor` | `../../../../../qcom/proprietary/devicetree` | 同上 |

内核仓库里的 **Kbuild 拼接行**（实测）：

- `kernel/Makefile`：`obj-$(CONFIG_OPLUS_FEATURE_PERFORMANCE_SCHED) += oplus_perf_sched/`
- `kernel/sched/walt/Makefile`：
  ```make
  sched-walt-$(CONFIG_OPLUS_FEATURE_INPUT_BOOST) += tuning/input_boost.o
  sched-walt-$(CONFIG_OPLUS_FEATURE_INPUT_BOOST) += tuning/webview_boost.o tuning/sched_assist_ofb.o tuning/frame_boost_group.o tuning/frame_info.o
  sched-walt-$(CONFIG_OPLUS_FEATURE_SCHED_ASSIST) += oem_sched/oem_fair.o
  ```
  [链接](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8450/blob/oneplus/sm8450_s_12.1_10_pro/kernel/sched/walt/Makefile)
- `mm/Makefile`：`obj-$(CONFIG_OPLUS_FEATURE_PERFORMANCE_MM) += oplus_perf_mm/`；`fs/Makefile` 同构。

内核仓库里的 **C 调用点**（全部在 QCOM 的 WALT 文件里，不在 GKI common 的 fair.c 里）：

- [`kernel/sched/walt/walt.c`](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8450/blob/oneplus/sm8450_s_12.1_10_pro/kernel/sched/walt/walt.c)：`#include <../kernel/oplus_perf_sched/sched_assist/sa_fair.h>`（L38），L1965–1968 `#ifdef CONFIG_OPLUS_FEATURE_SCHED_SPREAD … update_load_flag(p, rq);`，另有 `CONFIG_OPLUS_FEATURE_OCH`、`CONFIG_OPLUS_FEATURE_INPUT_BOOST` 分支多处。
- [`kernel/sched/walt/walt_cfs.c`](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8450/blob/oneplus/sm8450_s_12.1_10_pro/kernel/sched/walt/walt_cfs.c)：L15–16 同时 include `sa_fair.h` 与 `sa_common.h`；L314–322 调 `sched_assist_spread_tasks(...)`。
- [`kernel/sched/walt/walt_lb.c`](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8450/blob/oneplus/sm8450_s_12.1_10_pro/kernel/sched/walt/walt_lb.c)：L12 `#include <../../oplus_perf_sched/sched_assist/sa_fair.h>`。

**5.4 的拼接方式**（对照，来自实测 symlink）：`include/linux/sched_assist -> ../../../../vendor/oplus/kernel/oplus_performance/sched_assist/`、`kernel/sched_assist -> ../../../vendor/oplus/kernel/oplus_performance/sched_assist/`，然后 **fair.c 本体被改**（`#include <linux/sched_assist/sched_assist_common.h>`，231 处 OPLUS 相关行）。

---

## ④ 符号/文件级耦合点分析（≥5，每条给证据 + 断点 + 可平移性）

### 耦合点 1：`android_rvh_replace_next_task_fair`（最典型的"机制断点"）

- **5.10（vendor 侧注册，内核零改动）**：`sched_assist.c` 里
  `REGISTER_TRACE_RVH(android_rvh_replace_next_task_fair, android_rvh_replace_next_task_fair_handler);`
  处理函数签名（`sa_fair.h`）：
  `void android_rvh_replace_next_task_fair_handler(void *unused, struct rq *rq, struct task_struct **p, struct sched_entity **se, bool *repick, bool simple, struct task_struct *prev)`
  → [sched_assist.c](https://github.com/OnePlusOSS/android_kernel_modules_and_devicetree_oneplus_sm8450/blob/oneplus/sm8450_s_12.1_10_pro/vendor/oplus/kernel/oplus_performance_5.10/sched/sched_assist/sched_assist.c) ／ [sa_fair.h](https://github.com/OnePlusOSS/android_kernel_modules_and_devicetree_oneplus_sm8450/blob/oneplus/sm8450_s_12.1_10_pro/vendor/oplus/kernel/oplus_performance_5.10/sched/sched_assist/sa_fair.h)
- **5.4（直接改 fair.c 调 vendor 函数）**：`_meta/mig/kernel__sched__fair.c.base` L8246–8249 与 L8282–8289：
  `android_rvh_replace_next_task_fair_handler(rq, &p, &se, &repick, false);` —— **少一个 `prev` 参数**，且不是钩子调用而是硬编码调用。
- **断点**：5.4 树的 `include/trace/hooks/sched.h` 里只有 **20 个** hook，**没有** `android_rvh_replace_next_task_fair`；5.10 树的同名文件有 **86 个**（我逐名比对过）。
- **可平移性**：5.10 那一份可以整套搬到任何带 ACK 5.10 hook 面的 msm-5.10 树；反过来往 5.4 搬必须先给 5.4 补 hook（不是配置问题，是补内核代码）。

### 耦合点 2：`android_rvh_place_entity` / `android_rvh_pick_next_entity`（CFS 内部钩子）

- 5.10：vendor 注册 `REGISTER_TRACE_RVH(android_rvh_place_entity, …)`、`REGISTER_TRACE_RVH(android_rvh_pick_next_entity, …)`；处理函数形参直接用 `struct cfs_rq *cfs_rq, struct sched_entity *se`（见 `sa_fair.h`）。
- 5.4：fair.c L4339–4341 在 `place_entity()` 内部直接 `#ifdef CONFIG_OPLUS_FEATURE_VT_CAP` 调 `android_rvh_place_entity_handler(NULL, cfs_rq, se, initial, &vruntime);`。
- 断点：这两个 hook 名在 5.4 树的 hook 头文件里**不存在**（20 vs 86 的差集里就有它们）。
- 含义：5.10 的 vendor 代码**假定 CFS 内部结构（`cfs_rq`/`sched_entity`）可以在 hook 边界上被外部模块看见**。这不是 SoC 耦合，是内核世代耦合；对同代 msm-5.10 的 lahaina 完全成立。

### 耦合点 3：私有数据存放——`ANDROID_OEM_DATA_ARRAY`（ABI 契约断点）

- 5.10 内核：`include/linux/sched.h` L1380–1381
  ```c
  /* Please add your member in struct oplus_task_struct */
  ANDROID_OEM_DATA_ARRAY(1, 32);
  ```
  `kernel/sched/sched.h` L1082–1083 同样给 `struct rq` 加了 `ANDROID_OEM_DATA_ARRAY(1, 16);`
- 5.10 vendor：`sched_assist.c` 里用编译期断言把两边的尺寸钉死：
  ```c
  OPLUS_OEM_DATA_SIZE_TEST(struct oplus_task_struct, struct task_struct);
  OPLUS_OEM_DATA_SIZE_TEST(struct oplus_rq, struct rq);
  ```（宏体是 `BUILD_BUG_ON(sizeof(ostruct) > sizeof(u64) * ARRAY_SIZE(...android_oem_data1))`）
- 5.4 树：`include/linux/sched.h` / `kernel/sched/sched.h` 里**没有** `ANDROID_OEM_DATA_ARRAY`，而是被 OPLUS 直接改结构体（本机 5.4 `include/linux/sched.h` 里有 42 行 OPLUS 相关行，多个 `#ifdef CONFIG_OPLUS_*` 块）。
- 断点：两代的"内核 ↔ vendor 私有数据"契约完全不同（一个走 GKI 保留数组 + 尺寸断言，一个走改结构体 + 宏开关）。**这条恰好是对迁移有利的**：5.10 的契约是 SoC 无关的，`struct task_struct`/`struct rq` 的 OEM 槽位在任何 msm-5.10 目标（含 lahaina）上都存在。

### 耦合点 4：WALT API 与头文件位置

- 5.10 vendor 代码：`#include <linux/sched/walt.h>`，用 `walt_task_struct`（4 处）、`task_util()`（8 处）、`capacity_orig_of()`、`is_reserved()`、`sched_cpu_high_irqload()`、`sched_capacity_margin_down/up`、`num_sched_clusters`、`cpu_array`、`uclamp_eff_value()`。
- 5.10 内核：`include/linux/sched/walt.h` 存在（5,440 B，`walt_task_struct` 出现 7 次）；`kernel/sched/walt/walt.h` 27,437 B，含 `task_util`(9)、`is_reserved`(1)、`capacity_orig_of`(6)、`sched_capacity_margin_down`(3)、`struct walt_rq`(27)。
- 5.4 树：**没有** `include/linux/sched/walt.h`（该路径 raw 返回 404 文本）；`kernel/sched/walt/walt.h` 只有 7,614 B，且**不含** `walt_task_struct` / `task_util` / `is_reserved` / `capacity_orig_of` / `sched_capacity_margin_*` / `struct walt_rq`（逐符号计数全为 0）。
- 断点：这是 5.4→5.10 之间 WALT 的一次结构性换代（5.10 把 task/rq 侧的 WALT 状态提升为正式结构并公开了头文件）。
- 含义：5.10 vendor 代码不能编译在 5.4 的 WALT 上；但在 msm-5.10 家族内（waipio/cape/lahaina 同一棵树）它们共享同一套 WALT。

### 耦合点 5：构建期拼接（symlink + Makefile + 内部头文件）

- vendor 代码 `#include <kernel/sched/sched.h>`（`sa_fair.c`、`sched_assist.c`）、`#include <fs/proc/internal.h>`（`sa_common.c`）——**内核内部头文件**，说明它不是干净的外部模块，必须"长在"内核源码树里。
- 拼接靠：内核仓库 symlink（3.3 表）+ `kernel/Makefile`/`kernel/sched/walt/Makefile` 的一行 `obj-$(CONFIG_…)`。
- 5.4 侧完全同构（`include/linux/sched_assist`、`kernel/sched_assist` 两个 symlink + fair.c 里 include 厂商头）。
- 断点：**拼接点本身在移植时是"低风险但必须有"的工作**；两个树的 symlink 目标路径不同（`oplus_performance/` vs `oplus_performance_5.10/`），Makefile 变量名也不同（5.10 用 `sched-walt-$(CONFIG_OPLUS_FEATURE_SCHED_ASSIST)` 挂到 `sched-walt.o`）。

### 耦合点 6（反例，说明"不能想当然"）：`struct sched_class` 在这两棵树上**不是**断点

- 我逐 op 比对了两棵树的 `kernel/sched/sched.h` 里 `struct sched_class`（5.4 树也已经有 `set_next_task`、`balance`、`task_change_group`，op 列表两树完全一致，差集为空）。
- 说明：这台机器的 5.4 是 CAF/OPLUS 深度改造树，不是 vanilla 5.4；**不能拿 vanilla 5.4↔5.10 的差异表来估算工作量**，必须以这两棵实际树为准。

### 耦合点 7：功能开关面（`CONFIG_OPLUS_*`）

5.10 的 `arch/arm64/configs/vendor/waipio_GKI.config` 里（[链接](https://github.com/OnePlusOSS/android_kernel_oneplus_sm8450/blob/oneplus/sm8450_s_12.1_10_pro/arch/arm64/configs/vendor/waipio_GKI.config)）：

```
CONFIG_SCHED_WALT=m
CONFIG_OPLUS_FEATURE_PERFORMANCE_SCHED=y
CONFIG_OPLUS_FEATURE_SCHED_ASSIST=m
CONFIG_OPLUS_FEATURE_SCHED_SPREAD=y
CONFIG_OPLUS_FEATURE_SF_BOOST=y
CONFIG_OPLUS_BINDER_PRIO_SKIP=y
CONFIG_OPLUS_FEATURE_HWC_BOOST=y
CONFIG_OPLUS_FEATURE_TASK_CPUSTATS=m
CONFIG_OPLUS_FEATURE_TASK_LOAD=m
CONFIG_OPLUS_FEATURE_INPUT_BOOST=y
CONFIG_OPLUS_SYSTEM_KERNEL_QCOM=y     ← vendor 代码里有 #ifdef/ifndef 分支
```

注意 `CONFIG_OPLUS_FEATURE_SCHED_ASSIST=m` **同时**有 `kernel/oplus_perf_sched/…/Makefile` 的 `obj-$(CONFIG_OPLUS_FEATURE_SCHED_ASSIST) += oplus_bsp_sched_assist.o`（可 built-in 也可 module），和它 include 内核内部头的事实并存 —— 实际配置里是靠 `=m` + 内核侧 `EXPORT_SYMBOL` 的组合；具体符号可见性必须在真机构建时逐条验证（列进第 ⑥ 节未确认项）。

---

## ⑤ 比例判断与依据（分层）

> 先声明口径：**"比例"是估算（第 ⑦ 节标注），依据是上面实测的符号/文件证据 + 公开可得的源码完整性**。分母按"完成 lahaina/5.10 上同等功能所需的总工作量"计。

### 5.1 OPLUS sched_assist 厂商层（vendor 源码本身）

| 归类 | 占比（估） | 具体内容与依据 |
| --- | --- | --- |
| **可平移** | **≈60–70%** | `sa_common.c/h`、`sa_sysfs.c`、`sa_binder.c`、`sa_exec.c`、`sa_mutex.c`、`sa_rwsem.c`、`sa_workqueue.c`、`sched_assist.c`（≈95 KB，实测 19 文件）——它们只依赖 `<linux/sched.h>`、`<kernel/sched/sched.h>`、`<linux/sched/walt.h>` 和 ACK hooks；这些在任何 msm-5.10 树（含 lahaina 目标）里都存在。sched_input_boost 那 4 个同名文件同属这一档（同名已核实，内容未逐行比对）。 |
| **需适配** | **≈20–30%** | `oem_fair.c`（5,766 B，编进 `kernel/sched/walt/`，用 `task_high_load()`、`capacity_orig_of()`、`sched_capacity_margin_down/up`、`task_lb_sched_type()`）需要按 lahaina 的簇数/容量表核对；`update_ux_sched_cputopo()` 之类按 CPU 容量分簇的逻辑要重新校验（sched_cls/capacity 表）；`CONFIG_OPLUS_SYSTEM_KERNEL_QCOM` 这类分支要按目标树重选；trace/sysfs 节点名与 userspace 约定要对齐。 |
| **必须重做** | **≈10%** | 与具体硬件耦合的调参/表：CPU 簇容量、cpufreq/thermal 联动阈值、`sched_capacity_margin_*` 的 SoC 相关常数、按设备提供的 DTS 参数（例如 boost 组、UX 识别名单的设备侧来源）。 |
| 依据 | | 上述符号依赖清单来自对 5.10 vendor 源码的全量 grep（`extern` 76 条、`EXPORT_SYMBOL` 18 处、hook 注册 20 处），不是读文档得出的。 |

### 5.2 内核侧拼接（把上面那层接回内核）

| 归类 | 占比（估） | 内容 |
| --- | --- | --- |
| **可平移** | **≈80%** | symlink（10 条，见 3.3）+ `kernel/Makefile`/`kernel/sched/walt/Makefile`/`mm/Makefile`/`fs/Makefile` 的 `obj-$(CONFIG_…)` 行 + `waipio_GKI.config` 的 OPLUS 段 —— 只要基底是「OPLUS 自己的 msm-5.10 树」，这些**已经现成**。 |
| **需适配** | **≈20%** | 换成 lahaina 目标要新写 vendor defconfig（该树只有 `waipio_*` 与 `neo*` 四个 vendor config，**没有 lahaina 的**，见第 ⑥ 节）；把 `build.config.msm.lahaina` 与 OPLUS 的 `build_oplus_defconfig_fragments()` 串起来（`kernel_platform/oplus/config/build.config.msm.wapio.oplus` 是 waipio 版本）。 |
| **必须重做** | ≈0% | 没有发现必须从零写的内核侧逻辑。 |

**结论性建议（可执行）：不要从"干净的 msm-5.10 + 重新打 OPLUS 补丁"起步。直接以 `android_kernel_oneplus_sm8450`（5.10，msm-5.10 家族、已带 OPLUS 拼接点与 lahaina 支持）为基底，把目标改成 lahaina，再把 martini 的设备树/面板/充电 DTS 从 5.4 前移。** 这能让 5.1/5.2 两项的工作量从"补回 OPLUS 补丁"降为"加 lahaina 目标"。

### 5.3 techpack（方案里的 Phase 2 硬骨头）

| 层 | 5.4 本机实测文件数 | 公开 5.10 是否有源码 | 可平移 / 需适配 / 需重写（估） |
| --- | --- | --- | --- |
| `techpack/audio` | 566 | **无**（OnePlusOSS/oppo-source 的 5.10 树全是 stub） | 5% / 25% / **70%** |
| `techpack/display` | 396（含 `display/oplus` 定制子目录） | **无** | 5% / 20% / **75%** |
| `techpack/camera` | 669 | **无** | 5% / 15% / **80%** |
| `techpack/dataipa` + `techpack/video` | 148 + 60 | **无** | 5% / 25% / **70%** |

依据（不是拍脑袋）：
1. 我把 OnePlusOSS `android_kernel_oneplus_sm8450` 的 **全部 6 个分支**逐个探过 `techpack/{audio,display,camera}/Makefile`，**全 404**；该仓库 `techpack/` 只有 5 个文件（`.gitignore`、`Kbuild`、`stub/{Makefile,stub.c,include}`）。OPPO Find X5 Pro 的 5.10 树同样没有。modules 仓库里也没有 techpack 相关路径（0 命中）。
2. 5.4 的 techpack 是**深度依赖内核版本**的驱动栈（DRM/KMS、ASoC/ALSA、V4L2/CAM 三级子系统的内部 API 在 5.4→5.10 之间都有改动），而 OPLUS 的私有定制（`techpack/display/oplus`、相机 tuning）没有 5.10 版本可对照。
3. 也就是说：**techpack 这一层"从 5.10 donor 借"的答案是 0**。要 5.10 techpack，只能拿到高通 msm-5.10 的 lahaina BSP（非公开）或另行逆向/前移（这就是方案里 Phase 2 的 4–8 周，我倾向于认为它被低估了）。
   - 唯一的好消息：5.10 的 modules 仓库里有 **整套 lahaina 参考 DTS/DTSI**（`kernel_platform/qcom/proprietary/devicetree/qcom/lahaina-{mtp,cdp,qrd,...}.dts{,i}`，实测 ≥40 个 lahaina 文件），5.4 modules 仓库里有 **martini 专属 DTS**（22 个，含 `lahaina-sde-display.dtsi`、`lahaina-camera-sensor-20820.dtsi`）。设备树这一块是"搬"，不是"写"。

### 5.4 内核树内的 OPLUS 驱动（对比项，非本任务重点）

5.4 树里 OPLUS 相关路径 503 个（含 `drivers/power/oplus_chg`、`drivers/input/oplus_fp_drivers`、`drivers/input/touchscreen/oplus_touchscreen_v2`、`drivers/soc/oplus/*`、`include/linux/sched_assist` 等 118 个 symlink）。5.10 侧对应物在 modules 仓库 `vendor/oplus/kernel/`（power 43 / system 157 / thermal 9 / audio 45 / touchpanel 402 / secureguard 56 / explorer 46 / network 8 / device_info 4 个文件）。

**判断（估）：可平移 30% / 需适配 45% / 需重写 25%。** 依据：这批代码在 5.10 里已经是"重新组织过的一版"（充电从 `kernel/charger` 挪到 `kernel/power`，指纹/触控走 `vendor/oplus/kernel/touchpanel`），框架逻辑可借；但 IC 驱动（充电 IC、触控 IC、指纹）与具体机型强绑定，必须按 9RT 的硬件重配甚至重写。

---

## ⑥ 未找到的部分（明确写"未找到"，不编）

1. **未找到任何公开的 OPLUS/OPPO 5.10 techpack 源码。** OnePlusOSS sm8450 全部分支、oppo-source sm8450、以及两者的 modules 仓库都是 stub / 无。
2. **未找到 OPLUS 的 SM8350(lahaina) 5.10 分支**：OnePlusOSS `android_kernel_oneplus_sm8350` 11 个分支全为 5.4；oppo-source 同 SoC 也是 5.4。
3. **未找到 OPLUS/C 5.10 树里的 lahaina defconfig**：`arch/arm64/configs/vendor/` 只有 `waipio_GKI.config`、`waipio_consolidate.config`、`waipio_tuivm*.config`、`neo*.config`。lahaina 只有 `build.config.msm.lahaina` + `modules.list.msm.lahaina`。
4. **未找到 OPLUS 公开的 5.10 真机 vendor config fragment**：modules 仓库里 `kernel_platform/oplus/config/waipio_oplus.config` 与 `modules.list.oplus` 都是 **0 字节空文件**（真实内容未公开）。
5. **未找到 5.10 上的 martini（9RT）设备树**：5.10 modules 仓库 devicetree 里 `martini` 命中 0；5.4 里有 22 个。9RT 的 DTS/面板/相机 sensor dtsi 必须自己前移。
6. **未找到 5.10 版的 uxmem / ux_page_pool 对应物**：modules 仓库 5.10 侧无 `uxmem|ux_page_pool` 命中；相近能力是 `oplus_performance_5.10/misc/mm_boost_pool/oplus_boost_pool.c` 与 `mm/hybridswap_zram/*`（是否等价未验证）。
7. **未逐一验证** 5.10 vendor 代码在 `=m`（模块）配置下的符号可见性：`sa_*.c` include 了 `<kernel/sched/sched.h>`、`<fs/proc/internal.h>`，同时又声明 `EXPORT_SYMBOL`；真正能编过并加载的组合需要一次真机构建验证（本报告无编译环境）。
8. **未找到 SM8450 之外、同代但更接近 lahaina 的 OPLUS 5.10 高通树**：5.10 高通 donor 只有 waipio(SM8450)/cape(SM8475) 两个世代。

---

## ⑦ 我核实到的 vs 我推测的

### 7.1 我核实到的（每条都有可点开的 URL 或本机文件行号）

1. sm8450/sm8475 分支的内核版本（5.10.66 / .101 / .136 / .209 / .226 / .236；sm8475 5.10.81 / .168 / .209 / .226）——逐个分支读根 `Makefile`。
2. oppo-source 组织存在，且 `android_kernel_oppo_sm8450`（Find X5 Pro）= 5.10.66，其 modules 仓库含 `vendor/oplus/kernel/oplus_performance_5.10/sched/sched_assist/Makefile`。
3. sm8350 无 5.10（OnePlusOSS 11 分支 / oppo-source 5.4.147 / 本机基线 5.4.295）；sm7325=5.4.210/254；sm8550=5.15.74。
4. 5.10 厂商层在 modules 仓库的 `vendor/oplus/kernel/oplus_performance_5.10/`，99 个文件，4 个一级子目录（fs/mm/misc/sched）；文件清单见第 ③ 节（来自 blob:none 部分克隆的 `git ls-tree -r`，82,035 条目）。
5. 5.10 内核仓库通过 10 条 symlink 指向 vendor 目录（逐条读 symlink 内容）。
6. `kernel/Makefile`、`kernel/sched/walt/Makefile`、`mm/Makefile`、`fs/Makefile` 的 OPLUS 拼接行（原文引用见 3.3）。
7. `walt.c` / `walt_cfs.c` / `walt_lb.c` 里的 OPLUS include 与调用点（行号见 3.3）。
8. **5.10 内核 `kernel/sched/fair.c` 中 OPLUS 引用数 = 0**（311,570 B 全文正则匹配 `oplus|sched_assist|oem_fair|CONFIG_OPLUS`，0 命中）。
9. 5.10 vendor `sched_assist.c` 的 hook 注册清单、`sa_fair.h` 的全部 handler 原型、`Makefile` 的 `oplus_bsp_sched_assist.o` 目标、`Kconfig` 的 `tristate` 定义（原文引用）。
10. hook 面差异：5.4 树 `include/trace/hooks/sched.h` 20 个 hook vs 5.10 树 86 个；5.10 vendor 依赖的 `android_rvh_place_entity` / `pick_next_entity` / `replace_next_task_fair` 在 5.4 里没有。
11. WALT 头文件差异：5.10 有 `include/linux/sched/walt.h`（5,440 B，含 `walt_task_struct`）；5.4 无该文件，`kernel/sched/walt/walt.h` 7,614 B 且不含 `walt_task_struct`/`task_util`/`is_reserved`/`capacity_orig_of`/`sched_capacity_margin_*`/`struct walt_rq`；5.10 对应文件 27,437 B 且都含。
12. `struct sched_class` 在两棵树里 op 列表**完全相同**（含 `set_next_task`/`balance`），位置都在 `kernel/sched/sched.h`。
13. `ANDROID_OEM_DATA_ARRAY(1,32)`（task_struct）/`(1,16)`（rq）在 5.10；5.4 树无该机制，改为直接改结构体（本机文件佐证）。
14. 5.4 树 `fair.c` 中 OPLUS 相关行 231 行、直接调用 6 个 vendor 函数；L4340 与 L8248/L8284 的硬调用点。
15. techpack：5.4 本机 CI 树 1,839 个源文件（audio 566 / camera 669 / display 396 / dataipa 148 / video 60）；**所有公开 OPLUS/OPPO 5.10 树 techpack 均为 stub**（每条 404 是我逐个路径探过的）。
16. lahaina 在 5.10 树里的真实存在：`build.config.msm.lahaina`（`MSM_ARCH=lahaina`）、`modules.list.msm.lahaina`（含 `gcc-lahaina.ko`、`dispcc-lahaina.ko`、`qnoc-lahaina.ko`、`pinctrl-lahaina.ko`、`phy-qcom-ufs-qmp-v4-lahaina.ko`）、lahaina clk/icc/phy/pinctrl 驱动、以及 modules 仓库 devicetree 里 ≥40 个 lahaina DTS/DTSI。
17. 5.4 modules 仓库有 martini 专属 DTS 22 个（含 `lahaina-sde-display.dtsi`、`lahaina-camera-sensor-20820.dtsi`、`lahaina-v2.1.dts`）；5.10 modules 仓库 martini = 0。
18. `waipio_GKI.config` 的 OPLUS 开关段（`CONFIG_SCHED_WALT=m`、`CONFIG_OPLUS_FEATURE_SCHED_ASSIST=m`、`CONFIG_OPLUS_FEATURE_SCHED_SPREAD=y` 等）原文。
19. 5.10 vendor 代码的 `extern` 声明 76 条、`EXPORT_SYMBOL` 18 处、hook 注册 20 处（全量 grep）。

### 7.2 我推测的（没有直接证据，使用需谨慎）

1. **⑤ 里的所有百分比**都是估算。方法是"依赖面反推"（符号依赖清单 + 公开源码完整性），不是逐文件编译验证。特别是 sched_assist 的 60–70%/"可平移"结论，前提是**基底为 msm-5.10 家族树且 `CONFIG_SCHED_WALT` 打开**。
2. **techpack 各层的 70–80% "需重写"**：我没有做 5.4↔5.10 的 techpack API 逐文件 diff（5.10 侧根本没源码可 diff），依据是"该子系统在内核 5.4→5.10 间有 API 变更 + 无 5.10 参照源"。
3. **"以 OPLUS sm8450 5.10 树为基底更省事"**这条建议：我没有实际构建过，也未确认 `msm-kernel` 与 `kernel_platform/common` 在这套发布里的拼装细节（modules 仓库里没有 `msm-kernel` 目录，内核仓库放哪个位置是我从 symlink 相对路径反推的）。
4. **`vendor/oplus/kernel/oplus_performance_5.10` 之外那些 vendor 目录（power/system/thermal/touchpanel）与 5.4 对应物是否同源**：我只做了文件数与目录名对照，没有逐文件比对。
5. **uxmem 与 5.10 `mm_boost_pool/hybridswap_zram` 是否能力等价**：未验证。
6. **9RT(martini) 的面板/充电 IC/触控 IC 是否能直接套用 waipio 的 5.10 vendor 驱动**：未验证（需要具体 IC 型号比对）。
7. **`CONFIG_OPLUS_FEATURE_SCHED_ASSIST=m` 时那批 include 内核内部头的源码能否真的编成模块**：未验证（见 ⑥.7）。

### 7.3 与方案文档（`kernel-5.10-migration-plan.md`）的差异（含推翻项）

| 方案里的说法 | 我的核实结果 |
| --- | --- |
| §1.1「msm-5.10 里 lahaina 是一等公民」 | **成立且更强**：连 OPLUS 自己的 5.10 树都带 `build.config.msm.lahaina` + 真 lahaina 模块清单；modules 仓库还有整套 lahaina DTS。 |
| §1.3「厂商层里有大量与 SoC 耦合的部分 → 只能跨 SoC 借参照」 | **对 sched_assist 这一层被推翻**：耦合对象是内核世代（WALT API + ACK 5.10 hook 面 + OEM data 契约），不是 SoC；这一层可以整套借，且能在同一 msm-5.10 家族里直接落到 lahaina。对充电/触控/面板/techpack 仍然成立。 |
| §1.4「techpack 不在公开 msm-5.10 里」 | **成立且被加强**：连 OPLUS/OPPO 官方的 5.10 树里 techpack 也全是 stub，因此不存在"从 5.10 donor 借 techpack"。 |
| §2 Phase 3「OPLUS 厂商层移植 2–4 周」 | 若走"以 OPLUS sm8450 5.10 树为基底"的路径，**sched_assist 这一层**（5.1+5.2）我估 1–2 周量级；但技术包（techpack）与 userspace ABI 两处不变，整体工期判断不变。 |
| §5.1「零成本 userspace 电源模式 / thermal-engine 改写 scaling_max_freq」 | 与本报告无冲突；本报告不改变"先做 Phase 0"的建议。 |
| 方案未提及的事实 | **oppo-source 组织（Find X5 Pro 5.10）= 第二个官方 donor**；**5.10 树里没有 lahaina defconfig**；**5.10 的 OPLUS 公开 config fragment 是空文件**。 |

### 7.4 给 Lead 的最小验证建议（半天内可做，成本极低）

1. 用 `android_kernel_oneplus_sm8450` @ `b_16.0_oneplus_10_pro`（5.10.236）+ `android_kernel_modules_and_devicetree_oneplus_sm8450` 同分支，按 `kernel_platform/oplus/build/oplus_build_kernel.sh` 试编 **waipio** 目标 —— 验证"OPLUS 5.10 厂商层能否在自己家门口编出来"。这一步通了，5.1/5.2 的比例判断才算落地（我现在只能给符号级证据）。
2. 在上一步的树里把 `MSM_ARCH` 换成 `lahaina`、补一份 lahaina vendor defconfig，看 `oplus_perf_sched` / `oem_sched` / `tuning` 三个 symlink 目录能否一起编过 —— 这是"可平移"结论的直接反证点。
3. techpack 另开一线核实：是否有任何 5.10 lahaina BSP（高通/其它 OEM 完整发布）含 techpack 源；没有的话，方案 Phase 2 就是不可压缩的硬墙。
