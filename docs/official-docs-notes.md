# 官方文档对照 —— 我们做对了什么、漏了什么

来源：
- [Android · 构建内核（中文）](https://source.android.google.cn/docs/setup/build/building-kernels?hl=zh-cn)
- [Linux 内核文档 · 中文翻译](https://docs.kernel.org/translations/zh_CN/index.html)

---

## ① 官方规定的取源码方式：`repo`，不是手工 `git clone`

文档原文：

> 对于最新的内核，可以使用 **repo** 下载源代码、**工具链和构建脚本**。
> 使用 **repo** 方法可确保**源目录设置正确**。

```bash
mkdir android-kernel && cd android-kernel
repo init -u https://android.googlesource.com/kernel/manifest -b BRANCH
repo sync
```

以及一句关键的话：

> 内核树包含内核源代码和**用于构建内核的所有工具**，包括此脚本。

### 我们漏的就这一条

我们为了满足「不落本地」的约束，改用 `git clone` 手工重建目录布局。**源码部分是对的**
—— v1 的 symlink 解析验证（`ws/vendor/oplus/kernel/... → OK`）证明了布局无误。

但 **prebuilts 不在那两个仓库里**，它们由 manifest 里的独立项目提供。这直接造成了后面四轮的全部麻烦：

| 轮次 | 症状 | 真实原因 |
| --- | --- | --- |
| v2–v3 | `HERMETIC_TOOLCHAIN` 导致 PATH 被清空 | 与之相关，但那条是脚本缺陷 |
| v3–v4 | 缺 `arm-linux-gnueabi-gcc` | prebuilts 缺失 |
| v4–v6 | `CLANG_PREBUILT_BIN` 指向的目录不存在 | **prebuilts/clang 缺失** |
| v7 | `build-tools/path/linux-x86/bison` 指向不存在的目标 | **prebuilts/build-tools 缺失** |
| v7 | `hyp-reloc.S: unknown relocation name` | 自备 clang 版本选错（11 而非 12） |

**如果一开始用 `repo`，这五轮里有四轮不会发生。**

---

## ② 官方确认了我们的构建调用方式

| 我们的推导 | 文档 |
| --- | --- |
| `BUILD_CONFIG=<path> build/build.sh` | ✓ 完全一致 |
| 产物在 `out/` 下 | ✓ `out/<BRANCH>/dist` |
| Android 12 及更早用 `build.sh` | ✓「Android 14 及更高版本不支持 build.sh」 |
| v1 反推出的 `KERNEL_DIR` 规则 | ✓ 文档说明 `BUILD_CONFIG`「必须相对于 Repo 根目录定义」 |

文档给出的环境变量表（`build.sh` 旧版）：

| 变量 | 作用 |
| --- | --- |
| `BUILD_CONFIG` | build 配置文件路径，**相对 Repo 根目录**，默认 `build.config` |
| `CC` | 替换编译器，回退到 build.config 的默认值 |
| `DIST_DIR` | 分发输出目录 |
| `OUT_DIR` | 构建输出目录 |
| `SKIP_DEFCONFIG` | 跳过 make defconfig |

---

## ③ 为什么没有直接改用 `repo`

`repo init -u https://android.googlesource.com/kernel/manifest` 用的是 **AOSP 的 manifest**，
有 `common-android12-5.10` 等分支（已实测存在，共 12+ 个），但那是 **AOSP GKI common** 的清单，
**不含 Qualcomm 的 `msm-kernel`**。

OPLUS 这套用的是 `kernel_platform` 布局（`build/` `common/` `msm-kernel/` `oplus/` `qcom/`），
对应的 manifest 在 CLO 上没有找到公开入口（`clo/la/kernelplatform/manifest` 取不到分支）。

### 因此采取的策略

保留手工布局（已验证正确），**按需补 prebuilts**：

1. **clang** → 用 AOSP Gitiles 的 archive 接口单独下 `clang-r416183b`：
   ```
   https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/refs/heads/<branch>/clang-r416183b.tar.gz
   ```
2. **build-tools** → 缺失时移除指向它的 wrapper，让 apt 的系统工具接管（bison/flex 等）

这条路比 `repo sync` 繁琐，但不违反「不落本地」的约束，而且**每一处偏差都已知且可控**。

---

## ④ Linux 内核文档的中文翻译

入口：<https://docs.kernel.org/translations/zh_CN/index.html>

对当前阶段（bring-up）相关性较低 —— 里面的 `kernel-parameters`、`admin-guide`、
`dev-tools/kbuild` 等在「内核能编出来」之后才用得上。已存档备用，特别是：
- `dev-tools/kbuild`（构建系统）
- `process/volunteers` 等流程文档

---

## ⑤ 教训

**遇到「一套陌生的构建系统」时，第一步应该是读它的官方文档，而不是逆向脚本。**

这次我是从「读 build.sh 和 _setup_env.sh 的源码」入手的，虽然最终把调用方式推导对了
（文档可以佐证），但**prebuilts 这一块在脚本里看不出来** —— 因为它由 manifest 提供，
不在任何单个仓库里。这正是文档能一句话省掉四轮迭代的地方。
