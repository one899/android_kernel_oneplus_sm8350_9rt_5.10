# OnePlus 9RT (martini) — Linux 5.10 内核移植

> **状态：规划与验证阶段。尚未开始移植，本仓库目前没有任何可启动的东西。**

本仓库用于 9RT（OnePlus 9RT，代号 **martini**，型号 MT2110/MT2111，SoC 高通 **SM8350 / lahaina**）
从 Linux **5.4（QGKI，当前在用）**迁移到 **5.10** 的全部工作。

---

## 为什么要单独开仓库

当前在用内核是 5.4.295（另一仓库），上面已经做了大量优化（EEVDF 迁移、UX 不公平调度等）。
5.10 是一次**大版本迁移**，与本项目的性质完全不同：

- 需要新的内核源码基底（OPLUS 5.10 线）
- 需要新的构建体系（`kernel_platform` + `oplus_build_kernel.sh`）
- 需要重编全部厂商模块

放在一起会污染现有分支，所以独立成库。

---

## 关键前提（已核实，见 docs/）

| 事实 | 结论 |
| --- | --- |
| 有没有 SM8350 的 5.10 donor？ | **没有。** 官方与社区全部停在 5.4.x（最高 5.4.302）。连高通自家 BSP 清单（LA.UM.9.14.1）拉的仍是 `kernel/msm-5.4` |
| 那 5.10 的基底从哪来？ | **OPLUS 自己的 SM8450 5.10 线**（`OnePlusOSS/android_kernel_oneplus_sm8450`，5.10.66 → **5.10.236**）。它是同一个 msm-5.10 家族，**自带 lahaina 构建目标**与 OPLUS 拼接点 |
| 厂商层能借吗？ | **能。** 5.10 的 `kernel/sched/fair.c` 里 OPLUS 引用数 = **0**（5.4 是 231 行）—— 耦合的是内核世代而非 SoC，可整套搬 |
| 5.4 的 .ko 能直接用在 5.10 吗？ | **不能。** `module_layout` 的 CRC 必不同，第一个门就 `-ENOEXEC`。但这只意味着「必须重编」，不是墙 |
| 真正的未知 | **5.10 的 lahaina techpack（display/audio/camera）是否存在。** 公开的 OPLUS/OPPO 5.10 树里 techpack 都是外置的（5 个独立 CLO 仓库），本次未能核实其 5.10 分支是否含 lahaina 实现 |

**这个未知决定整件事的性质**：存在 ⇒ 6 个月量级的大工程；不存在 ⇒ 要自己把 5.4 的 1839 个 techpack 源文件前移三大驱动栈，建议放弃。

---

## 当前在用 5.4 内核的两个已发现问题（与本仓库相关）

1. **模块从未被安装过。** 构建确实产出 95 个 `.ko` 并打进 AnyKernel3，但 `anykernel.sh` 第 9 行是 `do.modules=0` —— 死代码。设备一直在用 ROM 原厂模块。
2. **符号版本校验被关闭。** 构建期把 `kernel/module.c` 换成第三方文件，其 `check_version()` 在 `bad_version:` 返回 `1`（上游是 `0`），而三个调用点全是 `if (!check_version(...))` —— 校验整个失效，64 个结构体布局不匹配的模块"带病加载"。

Phase 0 实测：设备 `/vendor/lib/modules` 有 99 个 `.ko`，构建产出 95 个，交集 91（**91.9%**）；
差额 8 个（WLAN×3 / RMNET×4 / explorer×1）的源码在配套的 modules 仓库
（`vendor/qcom/opensource/{wlan,datarmnet}`、`vendor/oplus/kernel/explorer`）里。

⇒ **源码覆盖率接近 100%，缺口在构建，不在源码。**

---

## 铁律

1. **在"5.10 lahaina techpack 是否存在"有答案之前，不写任何一行移植代码。**
2. 任何阶段结论必须区分「核实到」与「推测」，且附证据。本仓库不产出没有证据的结论。
3. 不修改 5.4 仓库的任何东西；两个项目互不污染。

---

## 立即要做的验证（半天，成本极低，可证伪）

计划文档 §3 给了最小验证路径：

1. 用 `OnePlusOSS/android_kernel_oneplus_sm8450` @ `oneplus/sm8450_b_16.0_oneplus_10_pro`（5.10.236）
   + 同分支的 modules 仓库，按 `kernel_platform/oplus/build/oplus_build_kernel.sh` **试编 waipio 目标**
   —— 先验证「OPLUS 5.10 厂商层能否在自己家门口编出来」。
2. 通了之后把 `MSM_ARCH` 换成 `lahaina`、补一份 lahaina vendor defconfig，
   看 `oplus_perf_sched` / `oem_sched` / `tuning` 三个 symlink 目录能否一起编过。
   —— **这是「厂商层可平移」结论的直接证伪点。**

## 文档

- [迁移方案 v2](docs/migration-plan.md) —— 主文档，含结论、阶段、工期、决策判据
- [r1 · SM8350 5.10 穷举核实](docs/research/r1-lahaina-510.md)
- [r2 · OPLUS 5.10 donor 与厂商层可借用比例](docs/research/r2-oplus-510.md)
- [r3 · 5.4→5.10 模块 ABI 机制链与 3183 行 CRC 成因](docs/research/r3-abi-analyst.md)

三份研究均分「核实到 / 推测 / 未找到」三节，所有百分比标注为估算。
