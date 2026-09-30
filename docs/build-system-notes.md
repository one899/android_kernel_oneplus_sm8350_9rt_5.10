# 5.10 构建体系调研笔记

> 调研对象：`OnePlusOSS/android_kernel_modules_and_devicetree_oneplus_sm8450`
> @ `oneplus/sm8450_b_16.0_oneplus_10_pro`（5.10.236）
> 结论全部来自实际读取脚本内容，非推测。

---

## 一、目录布局

```
android_kernel_modules_and_devicetree_oneplus_sm8450/
  kernel_platform/
    build/            ← Google 标准内核构建脚本（build.sh / envsetup.sh / abi/ …）
    oplus/
      build/          ← OPLUS 封装层
      config/
      dts_check/
    qcom/
  vendor/
```

内核源码本身不在这个仓库里，在配套的 `android_kernel_oneplus_sm8450`
（r2 报告中给出了 10 条 symlink 的对应关系）。

---

## 二、构建入口（按用途分两类）

### A. OPLUS 封装层 —— `kernel_platform/oplus/build/`

| 脚本 | 用途 |
| --- | --- |
| `oplus_build.sh` | 总入口。自带用法说明 |
| `oplus_build_kernel.sh` | 内核 + vendor 准备 |
| `oplus_build_ko.sh` | **模块单独构建**（对应第 7 节要修的 do.modules 问题） |
| `oplus_build_boot.sh` / `oplus_build_dtbo.sh` / `oplus_build_repack_bootimg.sh` | 打包 |
| `oplus_rebuild_img.sh` | 重打包镜像 |
| `oplus_setup.sh` | 环境变量与计时（被上面几个 source） |

**`oplus_build.sh` 自带的用法：**

```bash
./kernel_platform/oplus/build/oplus_build.sh waipio consolidate thin all true 2>&1 | tee build.log
#                                          ^1     ^2          ^3   ^4  ^5
#  1=virtants_platform  2=virtants_type  3=LTO  4=target_type  5=REPACK_IMG
```

### B. Google 标准层 —— `kernel_platform/build/`

`build.sh`、`envsetup.sh`、`_setup_env.sh`、`build_abi.sh`、`build_module.sh`、
`build_single_module.sh`、`menuconfig.sh`、`abi/`、`gki/`、`hermetic/`

---

## 三、⚠️ 关键约束：OPLUS 那套需要完整 AOSP 环境

`kernel_platform/build/android/prepare_vendor.sh` 的头部注释原文：

> `prepare_vendor.sh` prepares kernel/build's output for direct consumption in AOSP
> - Script assumes running after **lunch w/Android build environment variables available**

`oplus_build_kernel.sh` 的流程也正是：

```bash
source vendor/oplus/kernel/prebuilt/vendorsetup.sh
./kernel_platform/build/android/prepare_vendor.sh $variants_platform $variants_type
```

⇒ **`oplus_build.sh` / `prepare_vendor.sh` 不是独立内核构建**，它假设当前处在已 `lunch` 的
AOSP 构建树里，并为 AOSP 集成做 vendor 准备。单跑它不会得到我们想要的东西。

**对 CI 的影响**：GitHub Actions 上不可能为验证目的拉整套 AOSP（数百 GB）。
所以验证构建应当走 **B 类的 `kernel_platform/build/build.sh`（独立内核构建）**，
只编内核与 defconfig，不碰 vendor/AOSP 集成。

这条修正了计划文档 §3「按 `oplus_build_kernel.sh` 试编 waipio」的措辞。

---

## 四、⚠️ 另一个坑：脚本硬编码了 SM8450

`oplus_build_kernel.sh`：

```bash
export OPLUS_VND_BUILD_PLATFORM=SM8450     # ← 写死
export TARGET_BOARD_PLATFORM=$variants_platform
```

SM8450 = waipio。所以：

- 要验证「waipio 能否编过」→ 可以直接用，无需改动
- 要切到 **lahaina** → 必须改这一行，或改走 `oplus_build.sh`（它把 platform 作为 $1 传入）

---

## 五、对计划 §3 验证步骤的修正

**原计划**：按 `oplus_build_kernel.sh` 试编 waipio → 换 lahaina。

**修正后**：

1. 拉 `android_kernel_oneplus_sm8450`（5.10.236）与 modules 仓库，按 symlink 关系摆好目录。
2. 用 **`kernel_platform/build/build.sh`** 独立编 **waipio** 目标，确认工具链与 defconfig 能过。
   —— 验证「OPLUS 5.10 内核侧能否独立编出来」。
3. 改 `MSM_ARCH=lahaina`、补一份 lahaina vendor defconfig，再编一次。
   —— **这才是「sched_assist 可平移」结论的证伪点**。
4. 模块由 `oplus_build_ko.sh` 单独编（它也依赖 vendor 环境，需单独评估）。

**注意**：步骤 2 之前仍然要先解决第 5 节那个「5.10 lahaina techpack 是否存在」的未知 ——
如果 techpack 源不存在，技术包相关的目标根本编不出来。

---

## 六、待核实

- `oplus_build_ko.sh` 是否同样依赖 AOSP `lunch` 环境（未读，字节数与 kernel 版相近，推测同样依赖）
- `kernel_platform/build/build.sh` 对 msm 目标的实际调用方式与所需 env（未实测）
- 5.10 的 lahaina vendor defconfig 需要自己写，公开树里只有 `waipio_*` 与 `neo*`（r2 已核实）
