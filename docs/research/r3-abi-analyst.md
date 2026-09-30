# R3 — 5.4 → 5.10 的模块 ABI 断裂：机制级论证 + 本机证据复核

> 独立研究线 3（abi-analyst）· task-3 交付物 · 只写本文件
> 证据分两类：**核实到**（有源码/文档/日志 URL 支撑）与 **推测**（无直接证据，给验证方法）。第 ⑥ 节严格分开。
> 本机证据：D:\295\_meta\boot2_dmesg.txt（11883 行，2026-09-29 19:54 boot）、D:\295\_meta\9rt_defconfig、D:\295\_meta\wf_original.yaml。

---

## ⓪ 一页结论

1. **第 2 问的确定性回答：不可能。** 一个为 Linux 5.4（含 QGKI 5.4）编译的 .ko，**在未改动的 GKI 5.10 内核上无法加载**。不是"大概不行"，而是加载路径的第一个 CRC 门（check_modstruct_version() → module_layout）就必然失败，返回 -ENOEXEC。三条独立机制串联阻断（③节）。
2. **VERMAGIC 不是那堵墙。** 带 MODVERSIONS CRC 的模块，其 vermagic 的**内核版本号那一段在比较时被显式跳过**（same_magic() 里 strcspn(amagic," ")）。这解释了为什么日志里一行 version magic ... should be 都没有。墙是 **CRC 类型指纹**，不是版本字符串。
3. **本机那 3183 行的性质被原方案误读了。** 它们证明的是"**内核与模块来自不同构建、类型布局不一致**"，而**不是**"内核有缺陷"。而且那 6 个"加载失败"的模块**实际上都加载成功了**（⑤.5 有日志证据）。
4. **原因不是"内核没编模块"，而是 do.modules=0。** CI 把编出的 .ko 拷进 AK3 包的 modules/vendor/lib/modules/，但刷机脚本 anykernel.sh 里 do.modules=0，AnyKernel3 因此**一个模块都不装**（[anykernel.sh](https://github.com/miaizhe/Kernel_oppo_sm7325_Reno10/releases/download/boot/anykernel.sh)、[ak3-core.sh#L412](https://github.com/osm0sis/AnyKernel3/blob/master/tools/ak3-core.sh)），那句 find ... -exec cp 是死代码。
5. **对 Lead 三问的判断**：
   - (a) 推论**成立**（内核换了、模块还是 ROM 原厂的），但两处要改：不是"6 个模块失败"；修复要求不是"自己编的"，而是"**与运行内核同一次构建/同一 config 编的**"。
   - (b) 修复需要 4 个硬条件（同 config 同树同补丁、落到正确目录、不被原厂同名文件抢先、只读分区/verity 处理）；vermagic **不需要**逐字一致（版本号段被跳过），但 CRC 命名空间必须一致。
   - (c) 原方案「Phase 4 是硬墙 / 可能不可行」的**论据被推翻，结论要下调严重程度**：源码在树里（techpack/ + 公开模块仓库），模块本来就能自己编自己刷。ABI 仍然是"二进制不可复用"的硬事实，但它变成**工程量问题**（把 vendor/techpack 源码前移到 5.10 的 kernel API），不再是"拿不到源码 ⇒ 不可行"。

---

## ① 结论摘要

### 1.1 第 2 问：为 5.4 编译的 .ko 能否被 5.10 内核成功加载？

**确定性回答：没有可能（在未经修改的 GKI/QGKI 5.10 内核上）。**

阻断链条（按加载顺序，任一条即足以致命）：

| # | 机制 | 位置 | 结果 |
|---|---|---|---|
| A | module_layout 的 CRC 必然不同 | check_modstruct_version() | -ENOEXEC（第一个符号就挂） |
| B | 逐符号 __crc_* 指纹不同 | simplify_symbols() → check_version() | 同上 |
| C | 非 KMI 符号被裁剪（TRIM） | GKI 运行时裁剪 | Unknown symbol (err -2) |
| D | 结构布局 / 调用约定 / CFI / LTO 不一致 | 加载之后 | 内存踩踏、oops |

而 CONFIG_MODULE_FORCE_LOAD 在 GKI defconfig 里**没有开**，所以 insmod -f、MODULE_INIT_IGNORE_MODVERSIONS、MODULE_INIT_IGNORE_VERMAGIC 三条"官方后门"全部走不通（见 ③.7）。

**唯一的加载办法是改内核**：把 check_version() 的 bad_version: 分支从 return 0; 改成 return 1;——这正是本机工作流在做的事（⑤.3）。改完能加载，但等于放弃"类型布局一致"这一保证，模块将带着错误的偏移量运行。

### 1.2 与方案结论的对照（哪些要改）

| 方案里的表述 | 复核结论 |
|---|---|
| "换成 5.10 之后，ROM 里所有预编译 .ko 会全部失效" | ✅ **成立**，且是"必然/确定性"，不是概率问题；机制见 ③ |
| "连 5.4 内部的小版本/config 差异都扛不住" | ⚠️ **部分成立**：CRC 确实扛不住（3183 行），但**模块并没有因此加载失败**（内核被打了补丁），所以"扛不住"不等于"起不来" |
| "加载失败: haptic, bolero_cdc_dlkm, wcd9xxx_dlkm, q6_dlkm, swr_dlkm, mbhc_dlkm" | ❌ **被推翻**：6 个模块在日志里都是 LIVE 状态，其代码在跑（⑤.5） |
| "Phase 4 是硬墙，需重编整套 vendor 模块，源码不公开" | ⚠️ **需下调**：源码在树里（techpack/audio/config/lahainaauto.conf 直接把 bolero/wcd938x/wcd9xxx/mbhc/swr 等编成 .ko）。硬墙只剩"**预编译二进制不可复用**"，而"必须重编"是工程量不是原理阻塞 |
| "重编它们需要 OPLUS 的 vendor 源码（不公开）" | ❌ **不成立**：用户 CI 已经在编这些模块，源码来自公开仓库 |

---

## ② QGKI vs GKI 的 ABI 机制

### 2.1 两者的定义（以可核实材料为准）

- **GKI**：Google 的模型。内核以二进制形式发布，vendor 模块在**另一棵树**里编译，两者必须"像一起编出来的一样"工作——这是 AOSP 对 KMI 的原始定义：*"The GKI kernel is built and shipped in binary form and vendor-loadable modules are built in a separate tree. The resulting GKI kernel and vendor modules must work as though they were built together."*（[stable-kmi](https://source.android.com/docs/core/architecture/kernel/stable-kmi)）
- **GKI 的适用边界**：*"Beginning with Android 12, devices shipping with kernel version 5.10 or higher must ship with the GKI kernel."*（[generic-kernel-image](https://source.android.com/docs/core/architecture/kernel/generic-kernel-image)）——反过来说，5.4 设备（本机就是 5.4.295）不被强制 GKI，可以继续用高通/OEM 自己的内核镜像模型。
- **QGKI**：AOSP 文档**没有**给出这个词的定义（我检索了 source.android.com 的相关页面，未找到）。可核实的是本机内核的形态：Linux version 5.4.295-qgki-gb832d7ee034e-dirty，其中 -qgki 来自 defconfig 的 CONFIG_LOCALVERSION="-qgki"（D:\295\_meta\9rt_defconfig）。即：**高通把 GKI 的构建机制搬进了它自己的内核镜像**，"Qualcomm 编内核镜像 + OEM 编/装模块"这一层与 GKI 同构，但**不享受 Google 的 KMI 保证**（理由见 2.4）。
- 关键旁证：这棵 5.4 OPLUS/高通树里**带着完整的 GKI 构建机制与 KMI 工件**：
  - build.config.gki.aarch64：ABI_DEFINITION=android/abi_gki_aarch64.xml、KMI_SYMBOL_LIST=android/abi_gki_aarch64、TRIM_NONLISTED_KMI=1、KMI_SYMBOL_LIST_ADD_ONLY=1、KMI_SYMBOL_LIST_STRICT_MODE=1、KMI_ENFORCED=1，伙伴清单里有 abi_gki_aarch64_qcom / _oplus / _oneplus / _xiaomi / _vivo …（[用户树 build.config.gki.aarch64](https://github.com/one899/android_kernel_oneplus_sm8350_9rt_295bpf/blob/perf/eevdf-optimized/build.config.gki.aarch64)）
  - android/abi_gki_aarch64（78 字节，module_layout + __put_task_struct）、android/abi_gki_aarch64.xml、android/abi_gki_aarch64_qcom、android/abi_gki_aarch64_oplus 都存在（实测 HTTP 200）
  - 结论：**QGKI ≠ "没有 KMI 概念"**；它复用 GKI 的 KMI 工件与工具链，但 KMI 的"冻结/跨构建兼容"承诺属于 Google 的 GKI 分支，不属于 OEM 自编镜像。

### 2.2 KMI 如何定义与冻结（GKI 2.0 / Android 12 / 5.10）

AOSP 原文（[abi-monitor](https://source.android.com/docs/core/architecture/kernel/abi-monitor)）：

- 定义：*"Only the symbols listed in a symbol list and their related structures and definitions are considered part of the KMI."* 未列出的符号不算 KMI；GKI 用 TRIM_UNUSED_KSYMS=y + UNUSED_KSYMS_WHITELIST=<union of all symbol lists> 把非 KMI 符号**取消导出**，"loading a module requiring an unexported symbol is denied"。
- 冻结：*"When the KMI is frozen, no changes are allowed to the existing KMI interfaces; they're stable."* 冻结后仍可**新增**符号；*"Symbols shouldn't be removed from a list for a kernel unless it can be confirmed that no device has ever shipped with a dependency on that symbol."*
- 时间线：*"Incompatible ABI changes are allowed until First KMI for a given branch and during the coordinated KMI updates every two weeks until KMI Freeze."*
- 跨分支：*"Each Android Common Kernel (ACK) KMI kernel branch has its own set of symbol lists. **No attempt is made to provide ABI stability between different KMI kernel branches.** For example, the KMI for android12-5.10 is completely independent of ..."* ← **这是"5.4→5.10 不可能复用二进制"的官方定性。**
- 承诺内容：*"Because binary stability is maintained for the KMI, you can install these boot images without making changes to vendor images."*（[generic-kernel-image](https://source.android.com/docs/core/architecture/kernel/generic-kernel-image)）即**同一个 KMI 分支内**（例如 android12-5.10 的 5.10.x 各 LTS 版本之间）vendor 模块不必重编。
- 稳定性的前提条件（原文，[stable-kmi](https://source.android.com/docs/core/architecture/kernel/stable-kmi)）：
  - *"Only a single configuration, gki_defconfig, can be used to build the kernel."*
  - *"The KMI is only stable within the same LTS and Android version of a kernel, such as android14-6.1, android15-6.6 or android16-6.12."*
  - *"No KMI stability is maintained for android-mainline."*
  - *"Only the specific Clang toolchain supplied in AOSP ... is used for building kernel and modules."*

> **可直接引用的量化事实**：android12-5.10 的 ABI 报告 android/abi_gki_aarch64.xml = **12,877,160 字节**；伙伴符号清单 abi_gki_aarch64_qcom = 72,080 字节、abi_gki_aarch64_oplus = 92,880 字节；基础清单 android/abi_gki_aarch64 只有 78 字节（[ACK android12-5.10](https://github.com/aosp-mirror/kernel_common/tree/android12-5.10/android)）。这说明 KMI 的"面"是由"基础清单 + 各伙伴清单的并集"定义的。

### 2.3 module_layout 与 __crc_ 各扮演什么角色

**module_layout 是 MODVERSIONS 的"总闸/金丝雀"**。它在 kernel/module.c 里被定义成一个**空函数**，但参数故意覆盖了模块机制的核心结构（[v5.4 module.c#L4542](https://github.com/torvalds/linux/blob/v5.4/kernel/module.c#L4542)、[v5.10 module.c#L4596](https://github.com/torvalds/linux/blob/v5.10/kernel/module.c#L4596)）：

```c
#ifdef CONFIG_MODVERSIONS
/* Generate the signature for all relevant module structures here.
 * If these change, we don't want to try to parse the module. */
void module_layout(struct module *mod,
		   struct modversion_info *ver,
		   struct kernel_param *kp,
		   struct kernel_symbol *ks,
		   struct tracepoint * const *tp)
{
}
EXPORT_SYMBOL(module_layout);
#endif
```

- 源码注释就是它的设计意图：**"这些结构一旦改变，我们就不想再去解析这个模块"**。
- 调用点：check_modstruct_version() → find_symbol("module_layout", ...) → check_version(info, "module_layout", mod, crc)；**它在所有普通符号之前被检查**（[v5.10#L1362](https://github.com/torvalds/linux/blob/v5.10/kernel/module.c#L1362)、[v5.4#L1344](https://github.com/torvalds/linux/blob/v5.4/kernel/module.c#L1344)）。
- 所以：**CRC 的输入不是"函数体"，而是"参数类型图"**。struct module、struct tracepoint 等任一结构变化 → __crc_module_layout 变化 → **所有模块**的第一个检查就失败。
- AOSP 官方文档也正是拿它当典型故障示例：XXX: disagrees about version of symbol module_layout / init: Failed to insmod，并给出自查命令 nm vmlinux | grep __crc_module_layout（[abi-monitor](https://source.android.com/docs/core/architecture/kernel/abi-monitor)）。
- module_layout 同时是 GKI 符号清单的**第一个条目**（[abi_gki_aarch64](https://github.com/aosp-mirror/kernel_common/blob/android12-5.10/android/abi_gki_aarch64)：[abi_symbol_list] / # commonly used symbols / module_layout），在 android11-5.4、android12-5.4、android12-5.10 三个分支上写法一致。

**__crc_<sym>**：CONFIG_MODVERSIONS 下，genksyms 为每个导出符号按**类型声明**算出一个 CRC，编进内核的 __kcrctab_*（内核侧）和模块的 __versions 段（模块侧）。加载时逐符号比对：

- 不一致 → pr_warn("%s: disagrees about version of symbol %s") 然后 **return 0（拒绝）**（v5.4 [L1292](https://github.com/torvalds/linux/blob/v5.4/kernel/module.c#L1292)、[v5.10](https://github.com/torvalds/linux/blob/v5.10/kernel/module.c)）。
- 模块里找不到该符号的版本记录 → pr_warn_once("no symbol version for %s") 然后 **return 1（放行）**（这是"工具链坏了"的兜底，不是宽容）。
- 因此：**CRC 是"类型布局相等"的运行时证明**。AOSP 用它来"防止难以调试的运行时问题和内核崩溃"（[abi-monitor](https://source.android.com/docs/core/architecture/kernel/abi-monitor)）。

### 2.4 QGKI 到底有没有 KMI 稳定保证？

**核实的部分：**
- QGKI 这棵树**带着** GKI 的 KMI 工件与构建开关（2.1 的清单）。
- 但 Google 的保证是有前提的：只用 gki_defconfig、只用 AOSP 指定 clang、且只在**同一个 KMI 分支**内。凡是在这棵树里改 config、加内核补丁、混用不同构建产物，KMI 稳定性**按定义就不再成立**。
- 本机实证：ROM 里的原厂模块与本机 5.4.295 内核（同大版本、同 sublevel）之间有 **1079 个符号**的 CRC 不一致（⑤节）。如果存在"跨构建的 ABI 冻结"承诺，这个现象本身就不应该出现。
- 公开的 msm-5.10 镜像 techpack/ 只有 stub（Lead 已核实），说明高通的 5.10 交付形态里 vendor 模块源码是与内核镜像**配套**下发的——配套即一次性，不构成跨版本 ABI 承诺。

**推测（无公开文档支撑，标注为推测）：** 高通内部对同一 QGKI 发布（train）内的模块可能有 ABI 兼容要求，但我**未找到**任何公开承诺文档。任何"QGKI 保证模块 ABI"的说法都应视为未证实。

---

## ③ 5.4 .ko 在 5.10 上为何不可能加载（机制级）

### 3.1 加载路径的检查顺序（决定"第一个死在哪"）

load_module() 的顺序（[v5.10 kernel/module.c](https://github.com/torvalds/linux/blob/v5.10/kernel/module.c)）：

1. ELF 格式/段检查（copy_module_from_user、elf_validity_check）
2. check_modinfo()：**vermagic** → intree taint → retpoline/staging 提示
3. check_modstruct_version()：**module_layout 的 CRC** ← 第一道 MODVERSIONS 闸门
4. simplify_symbols()：逐符号 resolve_symbol() → **check_version()** ← 第二道闸门；符号不存在则 Unknown symbol (err -2)
5. 重定位、do_init_module()

### 3.2 机制 A：module_layout —— 必然失败，且是第一道门

module_layout 的 CRC 由 struct module / modversion_info / kernel_param / kernel_symbol / tracepoint 的类型图决定。5.4 → 5.10 之间这些结构**确定发生了变化**，我逐字对比过：

- struct tracepoint（[5.4](https://github.com/torvalds/linux/blob/v5.4/include/linux/tracepoint-defs.h) vs [5.10](https://github.com/torvalds/linux/blob/v5.10/include/linux/tracepoint-defs.h)）：5.10 新增 struct static_call_key *static_call_key; / void *static_call_tramp; / void *iterator;
- struct module（[5.4](https://github.com/torvalds/linux/blob/v5.4/include/linux/module.h) vs [5.10](https://github.com/torvalds/linux/blob/v5.10/include/linux/module.h)）：5.10 新增 bool using_gplonly_symbols;、void *noinstr_text_start;、unsigned int noinstr_text_size;、#ifdef CONFIG_KPROBES 下的 kprobes_text_start / kprobes_text_size / kprobe_blacklist / num_kprobe_blacklist、#ifdef CONFIG_HAVE_STATIC_CALL_INLINE 下的 num_static_call_sites / static_call_sites；并把 struct mod_kallsyms *kallsyms 改成 struct mod_kallsyms __rcu *kallsyms。
- （未变，作为对照：struct module_layout 本身、struct kernel_symbol、struct kernel_param 在 5.4/5.10 逐字相同——所以 CRC 变化来自上面这些，不是来自 module_layout 结构体本身。）

→ **__crc_module_layout 在 5.4 与 5.10 之间必然不同** ⇒ 每一个 5.4 模块在第 3 步就返回 -ENOEXEC（"Exec format error"）。

### 3.3 机制 B：逐符号 CRC —— 即使 A 被绕过也过不去

genksyms 的 CRC 覆盖参数类型图的**全部可达结构**。AOSP 自己给的例子：一个 mm_struct 字段改动 → *"4492 omitted; 4495 symbols have only CRC changes"*（[abi-monitor](https://source.android.com/docs/core/architecture/kernel/abi-monitor)）。5.4→5.10 里这类改动是海量的（mmap_sem→mmap_lock 改名、struct task_struct / struct file_operations / struct device 等多处变化）。每一个被 5.4 模块引用的符号，只要类型图里碰到任何一个变过的结构，就 disagrees。

### 3.4 机制 C：符号裁剪 —— 结构对了也可能根本不存在

GKI 构建用 TRIM_NONLISTED_KMI=1 + UNUSED_KSYMS_WHITELIST=<所有清单的并集>，非 KMI 符号**不导出**；AOSP：*"All other symbols are unexported, and loading a module requiring an unexported symbol is denied."*（[abi-monitor](https://source.android.com/docs/core/architecture/kernel/abi-monitor)、[build.config.gki.aarch64](https://github.com/aosp-mirror/kernel_common/blob/android12-5.10/build.config.gki.aarch64)）。一个 QGKI 5.4 厂商模块引用的高通私有导出符号，绝大多数不在 5.10 的 KMI 并集里 → Unknown symbol。

### 3.5 机制 D：万一全放行 —— 结构性不兼容

- 布局不同 → 模块按 5.4 的偏移量读写内核对象 → 内存踩踏/oops（这正是 AOSP 说 MODVERSIONS "prevents hard-to-debug runtime issues and kernel crashes" 的原因）。
- GKI 5.10 defconfig 打开 CONFIG_CFI_CLANG=y、CONFIG_LTO_CLANG_FULL=y、CONFIG_SHADOW_CALL_STACK=y（[android12-5.10 gki_defconfig](https://github.com/aosp-mirror/kernel_common/blob/android12-5.10/arch/arm64/configs/gki_defconfig)）。一个用 5.4 工具链编出的模块没有匹配的 CFI/LTO 插桩，跨模块间接调用与类型检查无法成立。
- stable-kmi 也明确：*"The toolchain used to build the GKI kernel must be completely compatible with the toolchain used to build vendor modules."*

### 3.6 VERMAGIC 为什么不是那堵墙（重要修正）

vermagic 字符串由 UTS_RELEASE + SMP + PREEMPT + MODULE_UNLOAD + MODVERSIONS + MODULE_ARCH_VERMAGIC (+RANDSTRUCT) 拼成（[v5.4 vermagic.h](https://github.com/torvalds/linux/blob/v5.4/include/linux/vermagic.h)、[v5.10 vermagic.h](https://github.com/torvalds/linux/blob/v5.10/include/linux/vermagic.h)）。但比较函数在"模块带 CRC"时会**跳过第一段（内核版本号）**：

```c
static inline int same_magic(const char *amagic, const char *bmagic, bool has_crcs)
{
	if (has_crcs) {
		amagic += strcspn(amagic, " ");
		bmagic += strcspn(bmagic, " ");
	}
	return strcmp(amagic, bmagic) == 0;
}
```

v5.4 与 v5.10 逐字相同（[v5.10 kernel/module.c](https://github.com/torvalds/linux/blob/v5.10/kernel/module.c) 的 same_magic；调用处 same_magic(modmagic, vermagic, info->index.vers)）。因为带 MODVERSIONS 的模块一定有 __versions 段（info->index.vers != 0），**"5.4.295-qgki-…" 与 "5.10.xx-android12-…" 的差异被跳过**。

两侧剩余段落在 GKI/QGKI 上都一致：CONFIG_SMP=y、CONFIG_PREEMPT=y、CONFIG_MODULE_UNLOAD=y、CONFIG_MODVERSIONS=y（[android12-5.10 gki_defconfig](https://github.com/aosp-mirror/kernel_common/blob/android12-5.10/arch/arm64/configs/gki_defconfig)、[android12-5.4 gki_defconfig](https://github.com/aosp-mirror/kernel_common/blob/android12-5.4/arch/arm64/configs/gki_defconfig)、本机 D:\295\_meta\9rt_defconfig），MODULE_ARCH_VERMAGIC 在 5.4 与 5.10 都是 "aarch64"（[5.4 arch/arm64/include/asm/module.h](https://github.com/torvalds/linux/blob/v5.4/arch/arm64/include/asm/module.h)、[android12-5.10 asm/vermagic.h](https://github.com/aosp-mirror/kernel_common/blob/android12-5.10/arch/arm64/include/asm/vermagic.h)）。

**所以 vermagic 会"放行"，然后模块死在第 3 步的 CRC 上。** 这也解释了本机日志：0 行 version magic ... should be，却有 3183 行 disagrees。

### 3.7 唯一的例外与唯一的路

- 官方"强制加载"通道全灭：try_to_force_load() 在 CONFIG_MODULE_FORCE_LOAD 未开时直接返回 -ENOEXEC；GKI defconfig 里**没有**这个选项（其余两项 MODULE_INIT_IGNORE_* 的最终效果也依赖它）。本机 9rt_defconfig 同样是 "# CONFIG_MODULE_FORCE_LOAD is not set"。
- 因此**只能改内核**（把 bad_version: 分支改 return 1）——这就是本机工作流做的事，见 ⑤.3。改后能加载，但安全性保证归零。
- 正确的路只有一条：**用 5.10 的树 + 5.10 的 config 重新编译这些模块**。这不是 ABI 问题，是移植问题（④节）。

---

## ④ GKI 化前置条件清单（"上 GKI"到底要交什么）

以 android12-5.10（GKI 2.0）为准，全部来自 [build.config.gki.aarch64](https://github.com/aosp-mirror/kernel_common/blob/android12-5.10/build.config.gki.aarch64)、[stable-kmi](https://source.android.com/docs/core/architecture/kernel/stable-kmi)、[abi-monitor](https://source.android.com/docs/core/architecture/kernel/abi-monitor)、[modules](https://source.android.com/docs/core/architecture/kernel/modules)：

**A. 内核侧工件（必须与 GKI 分支一致）**
1. **确定的 KMI 分支 + LTS**（如 android12-5.10）；分支一换，KMI 全部作废（*"No attempt is made to provide ABI stability between different KMI kernel branches"*）。
2. **只用 gki_defconfig**（可加官方的 GKI_DEFCONFIG_FRAGMENT）；任何改 ABI 的配置项都不允许。
3. **符号清单（symbol list）**：KMI_SYMBOL_LIST=android/abi_gki_aarch64 + ADDITIONAL_KMI_SYMBOL_LISTS="…abi_gki_aarch64_<partner>…"；厂商要把自己用到的符号**提交到 ACK**（伙伴清单，如 abi_gki_aarch64_qcom / _oplus），并接受 KMI_SYMBOL_LIST_STRICT_MODE=1 / KMI_SYMBOL_LIST_ADD_ONLY=1 的约束（只增不改不删）。
4. **ABI 报告**：ABI_DEFINITION=android/abi_gki_aarch64.xml（android12-5.10 上 12.9 MB，libabigail 生成的 ABI 描述）；每次改内核都要重新生成并随改动提交。
5. **强制开关**：TRIM_NONLISTED_KMI=1、KMI_ENFORCED=1、TIDY_ABI=1。
6. **模块清单**：MODULES_ORDER=android/gki_aarch64_modules（列出 GKI 自带模块）——android12-5.10 上该文件为空文件（0 字节），android13 之后才成体系。
7. **工具链与 hermetic build**：AOSP 指定版本的 clang + repo manifest 描述的完整环境；内核与模块**必须**用同一工具链。
8. **ABI 监控 CI**：build/abi/ 工具（extract_symbols / dump_symbols / compare_to_symbol_list / stgdiff）+ ABI Compliance/Compatibility 分析，破坏 KMI 的提交会被 Lint-1 拦下。

**B. 厂商侧工件（"厂商必须提供什么"）**
9. **所有 vendor 模块的源码 + 用 GKI 工具链重编**（不能复用任何旧二进制）；模块**只能使用 KMI 符号**。
10. **自己的 symbol list**（提交进 ACK 的 abi_gki_aarch64_<partner>）——这是"我依赖哪些内核接口"的正式声明，也是 ABI 监控的输入。
11. **模块分发载体**：早启动模块放 vendor_boot ramdisk，其余放 vendor_dlkm 镜像；配 modules.load / modules.dep / modules.softdep 等（[modules](https://source.android.com/docs/core/architecture/kernel/modules)）。
12. **（Android 13+ 才有）protected / unprotected GKI module 的区分**：GKI 模块用构建期签名区分，未签名模块只要只用白名单符号就允许加载。

**C. 验收 / 流程**
13. ABI 报告 + 符号清单必须成对更新，compare_to_symbol_list 必须零差异。
14. 冻结流程：First KMI → 每两周协调更新 → KMI Freeze；冻结后只允许"不破坏既有 KMI"的改动 + 新增符号。

> 一句话：**"上 GKI"不是"把 5.10 编出来"，而是"接受一套单一配置 + 符号清单 + ABI 报告 + 工具链锁定的工程流程"**。本机设备的现实是：内核被改过 config（KPROBES/MEMCG/namespaces/KSU…）、被换过源码文件（见 ⑤.3），已经不在这个流程里。

---

## ⑤ 那 3183 行"disagrees"的成因分析

数据源：D:\295\_meta\boot2_dmesg.txt（本机 2026-09-29 19:54 的完整 boot log，11883 行）。所有计数都是我在该文件上跑出来的。

### 5.1 观测数据（可复现）

| 指标 | 数值 |
|---|---|
| disagrees about version of symbol 行数 | **3183** |
| 涉及的**不同符号**数 | **1079** |
| 涉及的**不同模块**数 | **64** |
| 其中 module_layout 行数 | **280**（= 64 个模块 × 平均 ~4.4 次 modprobe 重试） |
| version magic ... should be | **0** |
| Unknown symbol | **0** |
| Invalid module format | **0** |
| loading out-of-tree module taints kernel | 1（pr_warn 带全局 taint 判断，只打第一条） |

Top 冲突符号：module_layout 280、mutex_lock 47、mutex_unlock 47、kmalloc_caches 46、kmem_cache_alloc_trace 46、__mutex_init 45、_dev_err 37、of_property_read_variable_u32_array 35、devm_kmalloc 33、__platform_driver_register 32、platform_driver_unregister 30。

按模块（前 12）：wlan 491、msm_drm 482、camera 188、explorer 90、mhi_bus 78、q6_dlkm 71、rmnet_core 69、rmnet_shs 69、platform_dlkm 67、tfa98xx_v6_dlkm 64、slimbus_ngd 58、wcd937x_dlkm 57。

### 5.2 决定性判别：这是"类型布局差异"，不是"全局命名空间/工具链差异"

我拿一批**原型里没有结构体**的符号去撞这份 1079 符号名单，结果：

| 符号（原型无结构体） | 出现在冲突名单里 |
|---|---|
| printk / _printk / jiffies / __kmalloc / kfree / memcpy / memset / msleep / schedule / panic / schedule_timeout / __stack_chk_fail / sprintf / snprintf / strlen / strcmp / vmalloc / vfree / ioremap / __kmalloc_track_caller / kmalloc_order / wait_for_completion / complete / __wake_up | **全部 0 次** |
| wake_up_process（struct task_struct *） | 7 |
| __alloc_pages_nodemask（nodemask_t → MAX_NUMNODES） | 3 |

**推论**：两侧的 genksyms/CRC 算法与"无结构体签名"完全一致（否则 printk/jiffies 也会冲突）；冲突**只出现在类型图里含结构体的符号上**。这正是"**两边的结构体布局不同**"的指纹，而不是工具链/算法差异。

### 5.3 为什么警告了还能加载？—— 内核的 module.c 被替换过

工作流在编译前做了整文件替换（D:\295\_meta\wf_original.yaml 第 79-86 行，各版本 workflow 都有）：

```yaml
rm kernel/module.c
curl -LSs "https://github.com/miaizhe/Kernel_oppo_sm7325_Reno10/releases/download/boot/module.c" -o kernel/module.c
```

我把这个被下载的 [module.c](https://github.com/miaizhe/Kernel_oppo_sm7325_Reno10/releases/download/boot/module.c) 拉下来逐字对比（vanilla 5.4 / ACK 5.4 / ACK 5.10 三份对照）：

```c
bad_version:
	pr_warn("%s: disagrees about version of symbol %s\n",
	       info->name, symname);
	return 1;          /* ← 上游与 ACK 都是 return 0; */
}
```

- 上游 v5.4 / v5.10 / ACK android12-5.4 / ACK android12-5.10：**return 0;**（拒绝加载）
- 被替换的这份：**return 1;**（打印警告后**放行**）
- 同一次替换还把 check_modinfo() 里的 add_taint() 换成了 add_taint_module()（所以 Modules linked in 里能看到逐模块的 (O) 标记）。
- **仓库里那份 kernel/module.c 本身是干净的**（bad_version: ... return 0;，[one899/…/kernel/module.c](https://github.com/one899/android_kernel_oneplus_sm8350_9rt_295bpf/blob/perf/eevdf-optimized/kernel/module.c)）→ 也就是说：**是构建期的替换关掉了 CRC 强制**，不是树的问题。

这解释了全部现象：警告照打，模块照装。

### 5.4 成因分解：CONFIG_MODVERSIONS / 配置差异 / 源码差异 / VERMAGIC 各占多少

| 因素 | 占比（以 3183 行为分母） | 依据 |
|---|---|---|
| **CONFIG_MODVERSIONS（机制）** | **100%**：这 3183 行只可能由它产生 | 该字符串只在 check_version() 里（[v5.10](https://github.com/torvalds/linux/blob/v5.10/kernel/module.c)），没有别的代码路径会打印它 |
| **VERMAGIC** | **0 行** | 日志里 0 行 version magic ... should be；源码层面版本号段被 same_magic() 跳过（3.6） |
| **配置差异（config）** | **主因（推测，见 ⑥）** | 冲突只落在含结构体的符号上（5.2）。工作流明确改了配置：-e CONFIG_KPROBES（影响 struct module → 影响**所有**模块的 module_layout）、-e CONFIG_MEMCG/CGROUP_*/SYSVIPC/USER_NS/PID_NS/IPC_NS/OVERLAY_FS/DEVTMPFS 等（影响 struct task_struct / struct kmem_cache / struct nsproxy 等）。本机 defconfig 还开了 MEMCG_KMEM=y、SLUB_DEBUG=y、PSI=y、MUTEX_SPIN_ON_OWNER=y、LTO_CLANG、CFI_CLANG —— 全都是会进结构体布局的开关，而 kmalloc_caches / kmem_cache_alloc_trace / mutex_lock / __mutex_init / wake_up_process 正好都在冲突名单上 |
| **源码/子树差异（OEM 内部树 ≠ 公开树）** | **不能排除，量级未知（推测）** | release_firmware（struct firmware 无 ifdef）、cdev_add 等也在名单上。这两个类型的 CRC 理论上只随源码变；但它们各自仍有可配置的传递依赖，所以此证据**不足以定论** |
| **工具链差异** | **0** | 5.2 的零冲突结论 |

> 一句话给决策用：**"内核被改了 config（+ 可能的树差异）之后，原厂预编译模块的类型指纹就对不上了；内核又被人为改成忽略这个检查，所以模块带病加载。"**

### 5.5 原方案的一处重要误判：那 6 个模块其实**加载成功了**

证据（同一份 log）：

1. Modules linked in: 列表里它们**全部无 + / (O+) 标记**（内核里 + = MODULE_STATE_COMING），即处于 LIVE：
   - L4926（t=4.4047）：… mbhc_dlkm aw8697 wcd9xxx_dlkm bolero_cdc_dlkm q6_dlkm …
   - L8748（t=10.5136）：… swr_haptics_dlkm … mbhc_dlkm aw8697 wcd9xxx_dlkm bolero_cdc_dlkm q6_dlkm … haptic haptic_feedback msm_drm
2. 代码在跑：swr_haptics_probe: register_hbst_off_notifier, rc=0（t=8.12）、wcd_mbhc_init: leave ret 0（t=8.16）、wcd_mbhc_start: enter（t=8.40）；haptic 的 qcom-hv-haptics 在 7.9s 处理 ioctl。
3. 为什么会有 280 行 module_layout？因为 userspace 反复 modprobe 同名模块：CRC 检查发生在 simplify_symbols()（注册之前），所以每次重试都会重打一串警告，最后才走到"已加载"返回。64 个模块 × 平均 4.4 次重试 = 280。

**结论**：把 disagrees 行等同于"加载失败"是启发式误判（D:\295\_meta\research\p0-census.ps1 第 33-35 行就是这么统计的）。真实状态是"**带病加载**"——比"失败"更危险：失败是显性的，带病是隐性的（类型布局不一致仍然按错误偏移量运行）。

### 5.6 回答 Lead 的 (a)(b)(c)

**(a) "内核换了、模块还是 ROM 原厂预编译的"这个推论成立吗？**

**成立**（并已被 do.modules=0 独立证实）。但要精确化两点：

- 冲突的**充分必要条件**是"内核与模块的**类型布局**不同"，而不是"模块是旧的"或"内核是新的"。同样会触发它的机制还有：① 内核 config 任何改动（尤其 debug/tracing/cgroup/namespace 类选项）；② 内核对**头文件/类型**的改动（.c 函数体改动不影响 ABI，但改动被导出符号原型引用到的结构体一定影响——见 ⑥ 对"树内改动 ABI 无关性"的精确化）；③ 设备上混装了不同批次/不同 ROM 版本的模块；④ 内核 sublevel/子树与模块构建时不一致。**工具链差异不会**，**VERMAGIC 也不会**（3.6、5.2）。
- 本次的**具体触发**：内核被改 config（工作流 scripts/config -e …）后重编，模块没跟着换 → 见 5.4 的分解。

**(b) 若要修，需要满足什么条件？**

硬条件 4 条：

1. **同一次构建**：.ko 必须与该内核**同树、同 config（同 out/.config）、同补丁**编出——最稳的是直接用 CI 那一次 make O=out 的产物（find out/ -name '*.ko'）。任何一个 config 位不同都可能再引入一批 CRC 冲突。
2. **落到原厂同名文件所在的目录**：先 su -c "find /vendor /vendor_dlkm /system_dlkm -name '*.ko'" 确认。AK3 的约定是 modules/<分区名>/<原路径>——工作流已经写好了 modules/vendor/lib/modules/，如果原厂模块其实在 /vendor_dlkm/lib/modules，这个路径就要改。
3. **必须覆盖原厂同名文件（不能只是"另外放一份"）**：modprobe / libmodprobe 按模块名在搜索路径里解析（modules.dep 用 base_path 前缀解析依赖，见 [libmodprobe.cpp](https://github.com/aosp-mirror/platform_system_core/blob/main/libmodprobe/libmodprobe.cpp)）。放在**更早被搜索的目录**也能生效，但同目录同名替换是最确定的。
4. **处理只读/verity 的 vendor 分区**：/vendor、/vendor_dlkm 是只读（dm-verity/EROFS），直接写会失败或被还原。可选：① 关 verity 后 rw 挂载再写；② 走 systemless（Magisk/KernelSU 模块 bind-mount）；③ 重打包 vendor(_dlkm) 镜像再刷。

工程条件 2 条：

5. **do.modules=1**（当前 anykernel.sh 是 do.modules=0，所以 AK3 不装任何模块）。注意该脚本同时是 do.systemless=1，而 [ak3-core.sh#L412/L433](https://github.com/osm0sis/AnyKernel3/blob/master/tools/ak3-core.sh#L412) 显示 do.modules=1 && do.systemless=1 会走 **Magisk/KernelSU 的 systemless 模块**路线——那条路对"post-fs-data 之后才加载"的模块（音频、wlan）通常可行，对极早期模块有风险，需要实测。
6. **SELinux 上下文与权限**：拷进去的 .ko 需要与原文件一致的 label/权限（chown 0:0、chmod 644、restorecon），否则 modprobe 会因权限/标签被拒。

**vermagic 要不要完全一致？——不需要（版本号段），但"剩余段"要一致。**
- 若 .ko 带 __versions（CONFIG_MODVERSIONS=y 编译），内核比较时跳过 UTS_RELEASE 那一段；5.4.295-qgki-xxxx 与 5.4.295-qgki-yyyy 甚至 5.10.xx-android12 在前缀上都无所谓。
- 必须一致的是 SMP preempt mod_unload modversions aarch64（分别来自 CONFIG_SMP / CONFIG_PREEMPT / CONFIG_MODULE_UNLOAD / CONFIG_MODVERSIONS / MODULE_ARCH_VERMAGIC）。同一构建自然一致。
- 真正卡死的仍然是 **CRC 命名空间**，所以"同一次构建"是唯一稳妥的做法。
- 补充：CONFIG_MODULE_SIG 在 GKI defconfig 与本机 defconfig 里都**没开**，签名不是障碍。
- 部分替换 vs 全量替换：逐模块独立检查，**部分替换在加载层面可行**；但保留 58 个布局不一致的模块，等于继续承担它们带病运行的隐性风险。要"修好"应当**全量替换**。

**(c) 对"5.4→5.10 是否可行"的影响 —— 结论怎么改**

- **"ABI 是硬墙"在二进制层面完全成立**（③），换 5.10 就意味着**所有**现有 .ko 一律作废，必须全部重编。
- **但原方案据此判定的"Phase 4 不可行"不成立**，因为它的前提是"源码不公开"。已核实：音频这一大类模块的源码就在树里并且就是编成 =m 的（[lahainaauto.conf](https://github.com/one899/android_kernel_oneplus_sm8350_9rt_295bpf/blob/perf/eevdf-optimized/techpack/audio/config/lahainaauto.conf)：CONFIG_SND_SOC_BOLERO=m、CONFIG_SND_SOC_WCD938X=m、CONFIG_WCD9XXX_CODEC_CORE_V2=m、CONFIG_SND_SOC_WCD_MBHC=m、CONFIG_SOUNDWIRE=m、CONFIG_SND_SWR_HAPTICS=m …；techpack/audio/Makefile 确认 soc/ dsp/ ipc/ asoc/ 全源码存在），vendor/device 目录来自公开仓库（[JackA1ltman/android_kernel_modules_and_devicetree_oneplus_sm8350](https://github.com/JackA1ltman/android_kernel_modules_and_devicetree_oneplus_sm8350)）。
- **正确的表述**：Phase 4 从「**原理性硬墙 / 可能不可行**」下调为「**工程量最大的必做项**」——
  - 不是"拿不到源码"，而是"要把 5.4 的 techpack/vendor 源码**前移到 5.10 的 kernel API**"（这本来就是 Phase 2/3 的工作，只是范围从"内核部分"扩到"全部 vendor 模块"）；
  - 真正的**剩余硬约束**变成：**设备上是否存在任何一棵 .ko 的源码不在树里/公开仓库里**。这是一个可枚举的清单问题（Phase 0 该做的正是这个：把设备上的 .ko 名单与一次完整构建产物的 .ko 名单做集合差）。若差集为空 → 没有原理阻塞；若差集非空 → 那些才是硬墙。
  - 工期估计：**不变甚至偏向上修**（原来把 Phase 4 记成 2-6 个月"可能不可行"，现在应记成"前移全部 vendor 模块源码"的工作量，与 Phase 2/3 同源、可合并估算，但仍是大头）。
- **对最终决策的影响**：方案第 4 节"若失效模块比例 > 80% 就不值得做"的判据要换——因为"失效"现在有两种含义：*不能加载*（改 config 后可解）与*必须重编*（5.10 下必然）。真正该算的比例是"**源码覆盖不到的 .ko 占比**"。

---

## ⑥ 我核实到的 vs 我推测的

### 6.1 核实到（每条都有可点击来源或本机可复现数据）

**A. 内核机制（源码级）**
1. check_version() 在上游 5.4 / 5.10 / ACK 5.4 / ACK 5.10 中 bad_version: 都是 return 0;（拒绝）—— [v5.4 module.c#L1292](https://github.com/torvalds/linux/blob/v5.4/kernel/module.c#L1292)、[v5.10 module.c](https://github.com/torvalds/linux/blob/v5.10/kernel/module.c)、[ACK android12-5.4](https://github.com/aosp-mirror/kernel_common/blob/android12-5.4/kernel/module.c)、[ACK android12-5.10](https://github.com/aosp-mirror/kernel_common/blob/android12-5.10/kernel/module.c)。
2. module_layout 是 CONFIG_MODVERSIONS 下导出的空函数，原型覆盖 struct module / modversion_info / kernel_param / kernel_symbol / tracepoint，源码注释明说"这些结构变了就不解析模块" —— [v5.4#L4542](https://github.com/torvalds/linux/blob/v5.4/kernel/module.c#L4542)、[v5.10#L4596](https://github.com/torvalds/linux/blob/v5.10/kernel/module.c#L4596)。
3. check_modstruct_version() 在普通符号之前检查 module_layout —— [v5.10#L1362](https://github.com/torvalds/linux/blob/v5.10/kernel/module.c#L1362)。
4. same_magic() 在有 CRC 时跳过 vermagic 第一段；5.4 与 5.10 实现逐字相同 —— [v5.4](https://github.com/torvalds/linux/blob/v5.4/kernel/module.c)、[v5.10](https://github.com/torvalds/linux/blob/v5.10/kernel/module.c)。
5. struct tracepoint 5.10 比 5.4 多 static_call_key / static_call_tramp / iterator —— [5.4](https://github.com/torvalds/linux/blob/v5.4/include/linux/tracepoint-defs.h) vs [5.10](https://github.com/torvalds/linux/blob/v5.10/include/linux/tracepoint-defs.h)。
6. struct module 5.10 vs 5.4 的字段增删（逐行 diff 结果见 3.2） —— [5.4](https://github.com/torvalds/linux/blob/v5.4/include/linux/module.h) vs [5.10](https://github.com/torvalds/linux/blob/v5.10/include/linux/module.h)。
7. struct module_layout / struct kernel_symbol / struct kernel_param 在 5.4 与 5.10 **逐字相同**（所以 CRC 变化来自 3.2 里列的那些结构，不是它们本身）—— 同上三份头文件。
8. MODULE_ARCH_VERMAGIC 5.4 arm64 与 android12-5.10 arm64 都是 "aarch64" —— [5.4 arch/arm64/include/asm/module.h](https://github.com/torvalds/linux/blob/v5.4/arch/arm64/include/asm/module.h)、[ACK android12-5.10 asm/vermagic.h](https://github.com/aosp-mirror/kernel_common/blob/android12-5.10/arch/arm64/include/asm/vermagic.h)。

**B. GKI 侧（AOSP 文档 + ACK 工件）**
9. KMI 定义、冻结规则、跨分支不保证 ABI、TRIM 运行时强制、MODVERSIONS 运行时强制、module_layout 故障样例、nm vmlinux | grep __crc_module_layout 自检命令 —— [abi-monitor](https://source.android.com/docs/core/architecture/kernel/abi-monitor)。
10. KMI 稳定的前提（单一 gki_defconfig、同一 LTS+Android 版本、AOSP 指定 clang、仅符号清单内符号） —— [stable-kmi](https://source.android.com/docs/core/architecture/kernel/stable-kmi)。
11. GKI 的 KMI 二进制稳定性承诺（同分支内换 boot 不动 vendor）、"Android 12 起 5.10+ 必须 GKI" —— [generic-kernel-image](https://source.android.com/docs/core/architecture/kernel/generic-kernel-image)。
12. vendor 模块分发位置（vendor_boot / vendor_dlkm）、GKI 模块签名与白名单模型 —— [modules](https://source.android.com/docs/core/architecture/kernel/modules)。
13. build.config.gki.aarch64 的全部关键变量（ABI_DEFINITION / KMI_SYMBOL_LIST / ADDITIONAL_KMI_SYMBOL_LISTS / TRIM_NONLISTED_KMI / KMI_ENFORCED / KMI_SYMBOL_LIST_STRICT_MODE / MODULES_ORDER） —— [android12-5.10](https://github.com/aosp-mirror/kernel_common/blob/android12-5.10/build.config.gki.aarch64)。
14. android12-5.10 与 android12-5.4 的 gki_defconfig 都是 CONFIG_MODULES=y / CONFIG_MODVERSIONS=y / CONFIG_MODULE_UNLOAD=y / CONFIG_PREEMPT=y，且**没有** CONFIG_MODULE_FORCE_LOAD、没有 CONFIG_MODULE_SIG —— [5.10](https://github.com/aosp-mirror/kernel_common/blob/android12-5.10/arch/arm64/configs/gki_defconfig)、[5.4](https://github.com/aosp-mirror/kernel_common/blob/android12-5.4/arch/arm64/configs/gki_defconfig)。
15. GKI 符号清单里 module_layout 是第一条；android12-5.10 的 ABI 报告 12,877,160 B、qcom 清单 72,080 B、oplus 清单 92,880 B、基础清单 78 B（实测 Content-Length） —— [android/ 目录](https://github.com/aosp-mirror/kernel_common/tree/android12-5.10/android)。
16. android12-5.10 **没有** android/abi_gki_aarch64.stg（HTTP 404）——STG 是后续版本才引入的形态。

**C. 本机/工程侧**
17. 本机内核：5.4.295-qgki-gb832d7ee034e-dirty，clang r383902 构建，-qgki 来自 CONFIG_LOCALVERSION="-qgki"（D:\295\_meta\9rt_defconfig）。
18. 本机 defconfig：CONFIG_MODVERSIONS=y、CONFIG_MODULE_UNLOAD=y、CONFIG_KPROBES=y、CONFIG_MEMCG=y / MEMCG_KMEM=y、CONFIG_PSI=y、CONFIG_MUTEX_SPIN_ON_OWNER=y、CONFIG_SLUB_DEBUG=y、CONFIG_LTO_CLANG=y、CONFIG_CFI_CLANG=y、"# CONFIG_MODULE_FORCE_LOAD is not set"、"# CONFIG_MODULE_SIG is not set"、"# CONFIG_TRIM_UNUSED_KSYMS is not set"。
19. 工作流在编译前替换 kernel/module.c（以及 fs/overlayfs/util.c、include/linux/sched/user.h，并打 sysvipc kABI 补丁）—— D:\295\_meta\wf_original.yaml L79-86；_meta 下 5 份 workflow 副本都含该替换。
20. 被替换的 [module.c](https://github.com/miaizhe/Kernel_oppo_sm7325_Reno10/releases/download/boot/module.c) 的 bad_version: 是 return 1;（放行），且用 add_taint_module()；而仓库自己的 [kernel/module.c](https://github.com/one899/android_kernel_oneplus_sm8350_9rt_295bpf/blob/perf/eevdf-optimized/kernel/module.c) 是干净的 return 0;。
21. 构建产物打包：find out/ -name "*.ko" → AnyKernel3/modules/vendor/lib/modules/（D:\295\_meta\wf_original.yaml L132-134）。
22. 刷机脚本 [anykernel.sh](https://github.com/miaizhe/Kernel_oppo_sm7325_Reno10/releases/download/boot/anykernel.sh)：do.modules=0、do.systemless=1；AK3 只有在 do.modules=1 时才处理模块，且 do.modules=1 && do.systemless=1 走 systemless 路线 —— [ak3-core.sh#L412](https://github.com/osm0sis/AnyKernel3/blob/master/tools/ak3-core.sh#L412)、[#L433](https://github.com/osm0sis/AnyKernel3/blob/master/tools/ak3-core.sh#L433)。
23. log 统计（3183 / 1079 / 64 / 280 / 0 / 0），以及"无结构体原型符号零冲突"的对照实验 —— D:\295\_meta\boot2_dmesg.txt，统计脚本见附录 A。
24. 6 个"失败"模块实际 LIVE 且代码在运行（Modules linked in 无 + 标记；wcd_mbhc_init: leave ret 0；swr_haptics_probe ... rc=0） —— 同 log。
25. 音频模块源码在树里且以 =m 编译：CONFIG_SND_SOC_BOLERO=m、CONFIG_SND_SOC_WCD938X=m、CONFIG_WCD9XXX_CODEC_CORE_V2=m、CONFIG_SND_SOC_WCD_MBHC=m、CONFIG_SOUNDWIRE=m、CONFIG_SND_SWR_HAPTICS=m、CONFIG_MSM_QDSP6V2_CODECS=m、CONFIG_MSM_ADSP_LOADER=m … —— [lahainaauto.conf](https://github.com/one899/android_kernel_oneplus_sm8350_9rt_295bpf/blob/perf/eevdf-optimized/techpack/audio/config/lahainaauto.conf)、[techpack/audio/Makefile](https://github.com/one899/android_kernel_oneplus_sm8350_9rt_295bpf/blob/perf/eevdf-optimized/techpack/audio/Makefile)。
26. 用户树里带着完整 GKI KMI 工件与开关 —— [build.config.gki.aarch64](https://github.com/one899/android_kernel_oneplus_sm8350_9rt_295bpf/blob/perf/eevdf-optimized/build.config.gki.aarch64)、android/abi_gki_aarch64{,_qcom,_oplus,.xml} 均 HTTP 200。

### 6.2 我推测的（含验证方法）

1. **"config 差异是本次 3183 行的主因"** —— 证据是"冲突只落在含结构体的符号上"+ 工作流确实改了会进布局的配置项（KPROBES/MEMCG/namespaces…）。但**我没有 OEM 侧的 .config / Module.symvers**，无法逐项归因。
   *验证*：① nm <本机 vmlinux> | grep __crc_module_layout 与设备上 modprobe --dump-modversions /path/xxx.ko | grep module_layout 对比，确认前缀差异；② 若能拿到 OEM 内核的 Module.symvers（或 /proc/config.gz），直接 diff config 与 CRC；③ 用 OEM 原厂 defconfig 重编一次内核，看 3183 行是否消失。
2. **"OEM 内部树与公开树存在源码差异"** —— 依据是 release_firmware / cdev_add 这类符号也冲突，而这些类型的 CRC 理论上只随源码变。但两者仍有可配置的传递依赖，**证据不足以定论**。
   *验证*：同 1 的 ③；或对同一个 .c 的导出符号做 CRC 反查。
3. **"QGKI 内部可能有同发布内的 ABI 承诺"** —— **未找到任何公开文档**。请勿在决策中依赖此点。
4. **"把 6 个模块重编并替换即可修复"** —— 机制上成立（CRC 由同一次构建保证一致），但**未实测**；且 5.6 的 4+2 条件里任一不满足都会失败（尤其 do.modules=0 与只读分区）。
   *验证*：do.modules=1（或 insmod 手动）后看 dmesg 里对应模块的 disagrees 是否消失、lsmod 是否列出新构建的版本号。
5. **"内核对树内源码的替换与 ABI 无关"这一说法需要精确化**（回应 Lead 第 1 点）：
   - **.c 函数体改动 → 对模块 ABI 无影响**（CRC 只由类型声明产生），这一点成立。
   - **但前提是"不碰被导出符号的原型"**。kernel/module.c 恰好**定义并导出** module_layout；本次替换的该函数原型与树内一致（逐字对比过），所以**这次**确实没改变 CRC 命名空间——只改了强制行为。若换成另一份原型不同的 module.c，__crc_module_layout 会变，**所有**模块（包括自己编的）都会失配。
   - include/linux/sched/user.h 是**头文件**，它定义了 struct user_struct。本次影响应为 0：5.4 里引用该类型的 free_uid() / alloc_uid() **不是导出符号**（只有 init_user_ns 是 EXPORT_SYMBOL_GPL，[kernel/user.c](https://github.com/torvalds/linux/blob/v5.4/kernel/user.c)）。但这属于"恰好没影响"，不是"头文件改动天然无关"。
   - sysvipc kABI 补丁改的是 struct kern_ipc_perm 一类类型，同理需要确认这些类型不出现在任何导出符号的原型里——**我未逐一验证**。
6. **"一次完整构建即可让全部模块 CRC 对齐"** —— 前提是模块与内核在同一次 make 中生成（同一 Module.symvers）。工作流确实是这么做的（make O=out 一次编完内核+模块），所以推测成立；但**没有在设备上实测过替换后的效果**。

### 6.3 未找到 / 无法核实

1. **QGKI 的官方定义与 ABI 承诺**：AOSP 文档没有 "QGKI" 词条；高通公开文档需要登录，未获取。**未找到**。
2. **OEM（OPLUS）构建 ROM 模块时的 .config 与 Module.symvers**：未获取（需要 OEM 构建产物或设备 /proc/config.gz）。这是把 6.2-1 从推测变成结论的唯一钥匙。
3. **设备当前 /vendor/lib/modules 与 /vendor_dlkm/lib/modules 的实际清单**：本次会话内无设备连接（adb devices 为空），无法执行 Phase 0 的集合差统计。**未核实**。
4. **android11-5.4 是否用 abi_gki_aarch64_whitelist 这一文件名**：android/abi_gki_aarch64_whitelist 在两个分支上都 404。**未找到**（早期 Android 11 的 whitelist 命名不在此路径）。
5. **G:\内核 本地树**：路径已不存在（可能已移动/卸载），我的源码侧核实全部改走 GitHub 与 kernel.org 原始文件。

---

## 附录 A：可复现命令

**A1. 复现 3183 行的统计（在 D:\295\_meta\boot2_dmesg.txt 上）**

```powershell
$f='D:\295\_meta\boot2_dmesg.txt'
$m = Select-String -Path $f -Pattern 'disagrees about version of symbol' -SimpleMatch
"lines   = " + $m.Count
$sym = $m | ForEach-Object { if ($_.Line -match 'symbol ([^ ]+)') { $matches[1] } }
"symbols = " + ($sym | Sort-Object -Unique).Count
$mod = $m | ForEach-Object { if ($_.Line -match '@[0-9]+[ ]+([^ ]+): disagrees') { $matches[1] } }
"modules = " + ($mod | Sort-Object -Unique).Count
"module_layout = " + ($m | Where-Object { $_.Line -match 'module_layout' }).Count
(Select-String -Path $f -Pattern 'version magic'  -SimpleMatch).Count   # => 0
(Select-String -Path $f -Pattern 'Unknown symbol' -SimpleMatch).Count   # => 0
```

**A2. 判定"CRC 命名空间不一致"的两条独立命令**

```bash
# 内核侧（AOSP 官方推荐做法）
nm /path/to/out/vmlinux | grep __crc_module_layout     # 例：0000000008663742 A __crc_module_layout
# 模块侧：模块期望的 CRC
modprobe --dump-modversions /vendor/lib/modules/bolero_cdc_dlkm.ko | grep module_layout
# 两者不等 => CRC 命名空间不一致（模块不是这个内核编的）
```

**A3. Phase 0 应补的集合差统计（下一轮有设备时执行）**

```bash
# 设备上所有 .ko 的名单
su -c "find /vendor /vendor_dlkm /system_dlkm -name '*.ko'" 2>/dev/null | xargs -n1 basename | sort -u > dev_ko.txt
# 一次完整构建产出的 .ko 名单
find out/ -name '*.ko' -printf '%f\n' | sort -u > build_ko.txt
# 差集 = 没有源码可编的模块（这才是 5.10 迁移的真正硬墙清单）
comm -23 dev_ko.txt build_ko.txt
```

---

## 附录 B：本次引用的全部 URL

内核源码（mainline，tag 固定）
- https://github.com/torvalds/linux/blob/v5.4/kernel/module.c
- https://github.com/torvalds/linux/blob/v5.10/kernel/module.c
- https://github.com/torvalds/linux/blob/v5.4/include/linux/vermagic.h
- https://github.com/torvalds/linux/blob/v5.10/include/linux/vermagic.h
- https://github.com/torvalds/linux/blob/v5.4/arch/arm64/include/asm/module.h
- https://github.com/torvalds/linux/blob/v5.4/include/linux/module.h
- https://github.com/torvalds/linux/blob/v5.10/include/linux/module.h
- https://github.com/torvalds/linux/blob/v5.4/include/linux/tracepoint-defs.h
- https://github.com/torvalds/linux/blob/v5.10/include/linux/tracepoint-defs.h
- https://github.com/torvalds/linux/blob/v5.4/include/linux/export.h
- https://github.com/torvalds/linux/blob/v5.4/include/linux/moduleparam.h
- https://github.com/torvalds/linux/blob/v5.4/kernel/user.c

Android / ACK
- https://source.android.com/docs/core/architecture/kernel/stable-kmi
- https://source.android.com/docs/core/architecture/kernel/abi-monitor
- https://source.android.com/docs/core/architecture/kernel/generic-kernel-image
- https://source.android.com/docs/core/architecture/kernel/modules
- https://android.googlesource.com/kernel/common/+/refs/heads/android12-5.10/
- https://github.com/aosp-mirror/kernel_common/blob/android12-5.10/build.config.gki.aarch64
- https://github.com/aosp-mirror/kernel_common/blob/android12-5.10/arch/arm64/configs/gki_defconfig
- https://github.com/aosp-mirror/kernel_common/blob/android12-5.4/arch/arm64/configs/gki_defconfig
- https://github.com/aosp-mirror/kernel_common/blob/android12-5.10/arch/arm64/include/asm/vermagic.h
- https://github.com/aosp-mirror/kernel_common/blob/android12-5.10/android/abi_gki_aarch64
- https://github.com/aosp-mirror/kernel_common/blob/android12-5.10/android/abi_gki_aarch64_qcom
- https://github.com/aosp-mirror/kernel_common/blob/android11-5.4/android/abi_gki_aarch64
- https://github.com/aosp-mirror/kernel_common/blob/android12-5.10/kernel/module.c
- https://github.com/aosp-mirror/kernel_common/blob/android12-5.4/kernel/module.c
- https://github.com/aosp-mirror/platform_system_core/blob/main/libmodprobe/libmodprobe.cpp

本机工程链
- https://github.com/one899/android_kernel_oneplus_sm8350_9rt_295bpf/blob/perf/eevdf-optimized/kernel/module.c
- https://github.com/one899/android_kernel_oneplus_sm8350_9rt_295bpf/blob/perf/eevdf-optimized/build.config.gki.aarch64
- https://github.com/one899/android_kernel_oneplus_sm8350_9rt_295bpf/blob/perf/eevdf-optimized/techpack/audio/Makefile
- https://github.com/one899/android_kernel_oneplus_sm8350_9rt_295bpf/blob/perf/eevdf-optimized/techpack/audio/config/lahainaauto.conf
- https://github.com/miaizhe/android_kernel_oneplus_sm8350_9rt_295
- https://github.com/miaizhe/Kernel_oppo_sm7325_Reno10/releases/download/boot/module.c
- https://github.com/miaizhe/Kernel_oppo_sm7325_Reno10/releases/download/boot/anykernel.sh
- https://github.com/osm0sis/AnyKernel3/blob/master/tools/ak3-core.sh
- https://github.com/JackA1ltman/android_kernel_modules_and_devicetree_oneplus_sm8350

本机文件（工作区内）
- D:\295\_meta\boot2_dmesg.txt（3183 行证据）
- D:\295\_meta\9rt_defconfig（运行内核的 config 基线）
- D:\295\_meta\wf_original.yaml（docker 补丁 + 打包逻辑）
- D:\295\_meta\research\p0-census.ps1（原统计口径，第 33-35 行的启发式会误判"加载失败"）
