# 9RT (martini / SM8350) 从 Linux 5.4 迁移到 5.10 —— 方案 v3

> v1 我一人所写；v2 经三条独立研究线核实（推翻 2 条、下调 1 条）；
> **v3 修正了 v2 最大的错误结论：techpack 不是死路，三大子系统的 5.10 + lahaina 源全部公开存在。**
> 状态：**方案已定稿，阻塞点已清除，可以进入验证构建阶段。**

---

## 结论先行

1. **前提不成立**：这台机器是 QGKI 设备，Android 版本与内核版本解耦。「Android 12 需要 5.10」对它不适用。
2. **没有现成 donor 产品**：全世界（官方 + 社区）没有任何 SM8350 设备跑 5.10。连高通自家 BSP 清单都钉在 `kernel/msm-5.4`。
3. **但技术起点齐了**：OPLUS 有成熟 5.10 内核线（SM8450，5.10.66 → 5.10.236，自带 lahaina 目标与 OPLUS 拼接点）；
   **高通 CLO 侧 display / audio / camera 三大 techpack 的 5.10 + lahaina 源全部确认存在。**
4. **v2 的「建议放弃」已作废。** 现在这是一件**工程量很大但完全可行**的事，阻塞点不在「有没有料」，而在「做多少活」。

---

## 0. 前提纠正

「Android 12 需要 5.10」只对 GKI 设备成立。SM8350/martini 是 QGKI，实测：

| 项目 / 分支 | Android | 内核 |
| --- | --- | --- |
| OnePlusOSS `SM8350_R_11.0` | 11 | 5.4.61 |
| OnePlusOSS `sm8350_s_12.1_martini` | **12.1** | **5.4.147** |
| OnePlusOSS `sm8350_u_14.0.0_oneplus9rt` | 14 | 5.4.254 |
| `sm8350_b_16.0.0_oneplus9rt`（当前基线） | 16/17 | 5.4.295 |
| LineageOS `lineage-23.2` | 16 | 5.4.302 |
| crDroid `17.0` | 17 | 5.4.302 |

**先明确目的**：卡顿 / 续航的主因已实测为 ColorOS userspace 热管理（负载升温直接改写 `scaling_max_freq`，
内核 thermal 冷却设备全程 `cur=0`），**换 5.10 不解决这两项**。若目的是安全补丁，升到 5.4.302 更划算。
**只有「上 GKI / 要 5.10+ 特性」才真正需要这次迁移。**

---

## 1. 关键事实（均已核实）

### 1.1 高通的 5.10 里有 lahaina，且是完整目标

`msm-5.10` 树中：`build.config.msm.lahaina`、`modules.list.msm.lahaina`（含真 lahaina 模块
`gcc-lahaina.ko` / `dispcc-lahaina.ko` / `qnoc-lahaina.ko` / `phy-qcom-ufs-qmp-v4-lahaina.ko`）、
lahaina clk/icc/phy/pinctrl 驱动齐备。

### 1.2 OPLUS 厂商层可以整套搬，不是「借参照」

**5.10 的 `kernel/sched/fair.c` 里 OPLUS 引用数 = 0**（5.4 是 231 行 + 6 个直接调用）。
5.10 改成 vendor 侧注册 ACK 的 `android_rvh_*` 钩子 + `ANDROID_OEM_DATA_ARRAY(1,32)/(1,16)`，
内核侧拼接只在 QCOM 自己的 WALT 文件（`walt.c` / `walt_cfs.c` / `walt_lb.c`）+ 5 行 Makefile + 10 条 symlink。

⇒ **耦合的是「内核世代」而非「SoC」**。而 lahaina 是同一 msm-5.10 家族的一等目标。

**两个官方 donor**：`OnePlusOSS/android_kernel_oneplus_sm8450`（5.10.66 → 5.10.236）、
`oppo-source/android_kernel_oppo_sm8450`（Find X5 Pro，5.10.66）。

### 1.3 ⭐ techpack：三大子系统的 5.10 + lahaina 源全部存在（v3 新增，推翻 v2）

| 子系统 | 仓库（CodeLinaro / CAF） | 5.10 分支 | lahaina 证据 |
| --- | --- | --- | --- |
| **display** | `clo/la/platform/vendor/opensource/display-drivers` [13664] | `display-kernel.lnx.5.10.r11-rel` | `config/lahainadisp.conf`、`config/gki_lahainadisp.conf` |
| **audio** | `clo/la/platform/vendor/qcom/opensource/audio-kernel-ar` [**22253**] | `audio-kernel.lnx.5.10.r6-rel` | `config/lahainaauto.conf` |
| **camera** | `clo/la/platform/vendor/opensource/camera-kernel` [13577] | `CAMERA.LA.2.0.c26` | `config/lahaina.mk` |

**两个坑（记录以便复现）**：

1. **audio 在 `-ar` 仓库**（AudioReach），不是 `platform/vendor/opensource/audio-kernel`。
   后者 674 个分支的内核版本族只有 `4.19 / 5.15 / 6.0`，**没有 5.10**。
2. **manifest 的 tag 命名不可靠地覆盖 lahaina**：audio(18)/video(22) 的 manifest 有 LAHAINA 标签，
   但 **display manifest 862 个 tag 里 0 个 LAHAINA —— 而 display 确实有 5.10 lahaina 源**。
   ⇒ 不能用「manifest 无 LAHAINA 标签」推断「没有源」。**必须用 5.10 的 SoC（waipio）反推分支名。**

### 1.4 5.4 的 .ko 不能用在 5.10（机制级结论）

`module_layout` 的 CRC 必不同（5.10 的 `struct tracepoint` / `struct module` 均变），第一个门就 `-ENOEXEC`。
**但这只意味着「必须重编」，不是墙** —— 而源码（1.3 + 1.2）都有。

顺带修正：**VERMAGIC 不是障碍**（`same_magic()` 会跳过内核版本段）；真正的判据是类型布局一致。

### 1.5 两套 release train 命名互斥（200/404 实测）

| 分支 | `msm-5.10` | `msm-5.4` |
| --- | --- | --- |
| `LA.UM.10.9.1.c25` | 404 | **200** |
| `KERNEL.PLATFORM.1.0.c25` | **200** | 404 |

`LA.UM.*` = 5.4 世代，`KERNEL.PLATFORM.*` = 5.10 世代。

---

## 2. 可行性盘点

### ✅ 有利

1. OPLUS 5.10 线成熟且**自带 lahaina 目标与 OPLUS 拼接点**。
2. 厂商层 sched_assist 可整套借（1.2）。
3. **techpack 三大件齐备**（1.3）—— v2 认为这是死路，实际是全通。
4. 设备树是「搬」不是「写」：5.10 modules 仓库有 ≥40 个 lahaina DTS/DTSI；5.4 有 22 个 martini 专属 DTS。
5. 全部模块源码可得：Phase 0 实测设备 99 个 `.ko`，构建产出 95 个（交集 **91 = 91.9%**），
   差额 8 个（WLAN×3 / RMNET×4 / explorer×1）的源码在 OPLUS modules 仓库
   （`vendor/qcom/opensource/{wlan,datarmnet}`、`vendor/oplus/kernel/explorer`）—— **源码覆盖接近 100%**。

### ❌ 仍需解决的（是「活」不是「没料」）

1. **martini 设备侧适配**：面板 / 充电 IC / 触控 IC / 相机 sensor 的机型参数要从 5.4 搬。
2. **msm-5.10 没有 lahaina 的 vendor defconfig**（公开树里只有 `waipio_*` 与 `neo*`），需自己写。
3. **OPLUS 5.10 的构建体系需要 AOSP `lunch` 环境**（`prepare_vendor.sh` 头部注释明确），
   CI 上不可行；验证构建须走 `kernel_platform/build/build.sh` 独立路径。
4. **公开的 `waipio_oplus.config` / `modules.list.oplus` 是 0 字节空文件**。

---

## 3. 推荐路径

> **以 `android_kernel_oneplus_sm8450`（5.10，家族内已带 OPLUS 拼接点与 lahaina 目标）为基底**，
> 把目标从 waipio 换成 lahaina，再把 martini 的设备树 / 面板 / 充电 DTS 从 5.4 前移，
> techpack 三大件直接取 CLO 的 5.10 + lahaina 分支（1.3）。

**不要**从干净的 msm-5.10 重新打 OPLUS 补丁 —— 那会白白多出一个数量级的工作量。

---

## 4. 阶段与工期（v3）

| 阶段 | 内容 | 估算 | 相比 v2 |
| --- | --- | --- | --- |
| Phase 0 | 目标确认 + 源码覆盖率测绘 | 半天 | 已完成（覆盖率 ~100%） |
| Phase 1 | 以 sm8450 5.10 为基底，切 lahaina 目标，编出可启动内核 | 3–7 天 | 不变 |
| **Phase 2** | **techpack 落地**：取 CLO 三分支 + 按 martini 硬件调参 | **2–4 周** | ⬇️ 从「不可估 / 建议放弃」降为可估 |
| Phase 3 | OPLUS 厂商层落地到 lahaina | 1–2 周 | 不变 |
| Phase 4 | 用户空间 ABI：重编并安装全部模块 | 2–6 周（工程量最大） | 不变 |
| Phase 5 | 集成、回归、稳定性 | 2–4 周 | 不变 |
| **合计** | | **约 2–4 个月**（单人） | ⬇️ 从「6–12 个月且可能不可行」 |

**唯一仍可能失控的点**：martini 的硬件参数（尤其相机 sensor tuning）能否从 5.4 直接前移。

---

## 5. 决策判据（v3）

- ~~v1：「失效模块比例 > 80% 就不值得做」~~ —— 已废（模块失效 ≠ 功能失效）
- ~~v2：「源码覆盖不到的 .ko 占比」~~ —— **已测：约 0%，判据通过**
- **v3：剩下的判据只有一条 —— 验证构建能否跑通。**

### 立即要做的验证（半天，可证伪）

1. 拉 `android_kernel_oneplus_sm8450`（5.10.236）+ 同分支 modules 仓库，按 symlink 关系摆好目录。
2. 用 **`kernel_platform/build/build.sh`** 独立编 **waipio** 目标 —— 验证工具链与 defconfig 能过。
   （**不要用** `oplus_build.sh` / `prepare_vendor.sh`，它们要 AOSP `lunch` 环境。）
3. 改 `MSM_ARCH=lahaina`、补一份 lahaina vendor defconfig，再编一次 —— **这才是「可平移」结论的证伪点**。
4. 模块由 `oplus_build_ko.sh` 单独评估（是否同样依赖 AOSP 环境待确认）。

---

## 6. 与 5.4 项目的关系

**当前在用的 5.4.295 内核有两个已发现的问题，它们与 5.10 计划独立，且更该优先处理：**

1. **模块从未被安装过**：workflow 确实编出并收集了 95 个 `.ko` 到 `AnyKernel3/modules/vendor/lib/modules/`
   （路径与设备实际 `/vendor/lib/modules` 一致），但 `anykernel.sh` 第 9 行是 `do.modules=0` —— **死代码**。
2. **符号版本校验被关闭**：构建期把 `kernel/module.c` 换成第三方文件，其 `check_version()` 在
   `bad_version:` 返回 `1`（上游是 `0`），而三个调用点全是 `if (!check_version(...))` —— 校验整个失效。
   结果是 **64 个结构体布局不匹配的模块「带病加载」**（已在跑，不是失败）。

详见 5.4 仓库的 `OPTIMIZATIONS.md`。

---

## 7. 一页版行动建议

```
已完成：
  ① Phase 0 源码覆盖率测绘 → ~100%，判据通过
  ② techpack 三大件的 5.10 + lahaina 源 → 全部确认（CLO）
  ③ OPLUS 厂商层可整套借 → 已证（fair.c 里 OPLUS 引用 = 0）

下一步（按顺序）：
  ④ 验证构建：sm8450 5.10 树 → 编 waipio → 切 lahaina       ← 现在就能做
  ⑤ 若 ④ 通过：写 lahaina vendor defconfig，取 CLO 三分支接进构建
  ⑥ 若 ④ 不通过：定位是工具链、defconfig 还是拼接问题，再决定

建议同时处理（与 5.10 无关，成本低、收益实）：
  ⑦ 5.4 侧：do.modules=1，让自编模块真正刷进去
  ⑧ 5.4 侧：评估是否移除那个关掉 CRC 校验的 module.c 后门
```

---

## 附：文档索引

| 文件 | 内容 |
| --- | --- |
| `docs/techpack-finding.md` | **techpack 最终结论**（三大件、两个坑、对照实验、可复现命令、project id） |
| `docs/build-system-notes.md` | OPLUS 5.10 构建体系调研（AOSP `lunch` 依赖、SM8450 硬编码） |
| `docs/research/r1-lahaina-510.md` | SM8350 5.10 穷举核实、假阳性排雷 |
| `docs/research/r2-oplus-510.md` | OPLUS 5.10 donor 清单、厂商层结构、7 个符号级耦合点 |
| `docs/research/r3-abi-analyst.md` | ABI 机制链、VERMAGIC 修正、3183 行 CRC 成因、`module.c` 后门取证 |

三份研究均严格分「核实到 / 推测 / 未找到」三节。
