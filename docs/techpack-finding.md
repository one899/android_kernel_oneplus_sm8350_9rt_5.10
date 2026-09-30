# techpack 未知 —— 最终结论：**墙不存在**

> 决定性未知：5.10 的 lahaina techpack 是否存在。
> **答案：三大子系统全部存在，全部在 CodeLinaro (CAF) 公开可下。**
> 计划文档里「techpack 是死路 / Phase 2 工期不可估 / 建议放弃」的结论**全部推翻**。

---

## ① 最终结果

| 子系统 | 仓库（CodeLinaro） | 5.10 分支 | lahaina 证据 |
| --- | --- | --- | --- |
| **display** | `clo/la/platform/vendor/opensource/display-drivers` [id=**13664**] | `display-kernel.lnx.5.10.r11-rel` | `config/lahainadisp.conf`、`config/gki_lahainadisp.conf` |
| **audio** | `clo/la/platform/vendor/qcom/opensource/audio-kernel-ar` [id=**22253**] | `audio-kernel.lnx.5.10.r6-rel` | `config/lahainaauto.conf` |
| **camera** | `clo/la/platform/vendor/opensource/camera-kernel` [id=**13577**] | `CAMERA.LA.2.0.c26` | `config/lahaina.mk` |

三条都有 200 的分支探测与目录列举佐证，不是推断。

---

## ② 为什么前几轮找不到（两个关键陷阱）

### 陷阱一：audio 换了仓库

我第一次查的是 `clo/la/platform/vendor/opensource/audio-kernel` [13492] —— 674 个分支里
内核版本族只有 `4.19 / 5.15 / 6.0`，**没有 5.10**。

**真正的位置是另一个仓库**：`clo/la/platform/vendor/qcom/opensource/audio-kernel-ar`（`-ar` = AudioReach）。
224 个分支里有 10 个含 5.10（`.c3` / `.c4` / `.r1-rel` … `.r6-rel`）。

**怎么发现的**：不是靠猜，是读 **manifest XML**——
`AUDIO_IOT.LA.11.0.r2-06500-LAHAINA.0.xml` 里明确写着：

```xml
<project name="platform/vendor/qcom/opensource/audio-kernel-ar"
         path="vendor/qcom/opensource/audio-kernel"
         upstream="refs/heads/audio-kernel-iot.lnx.11.0.r5-rel" />
```

### 陷阱二：manifest 的 tag 命名**不可靠地**覆盖 lahaina

这是本轮最重要的方法论教训。我差点据此下错结论。

| manifest 仓库 | tag 总数 | 含 LAHAINA 的 tag |
| --- | --- | --- |
| `techpack/audio/manifest` | — | **18** |
| `techpack/video/manifest` | 939 | **22** |
| `techpack/display/manifest` | 862 | **0** ← 但 display **确实有** 5.10 lahaina 源 |
| `techpack/camera/manifest` | 800+ | **0** |

**display 的反例是关键**：它明明有 `display-kernel.lnx.5.10.r11-rel` + `config/lahainadisp.conf`，
但它的 manifest 里一个 LAHAINA tag 都没有。

⇒ **「manifest 里没有 LAHAINA tag」不等于「没有 5.10 lahaina 源」。**
如果我拿 camera manifest 的 0 命中当结论，就会得出完全错误的答案。

### 正确的方法（本轮最终用的）

**用 5.10 的 SoC（waipio）去反推分支名。**

waipio 的 camera manifest `CAMERA.LA.2.0.c26-00700-WAIPIO.0.xml` 写着：

```xml
<project name="clo/la/platform/vendor/opensource/camera-kernel"
         upstream="CAMERA.LA.2.0.c26" />
```

⇒ 该分支就是 5.10 的 camera techpack。再去它里面找 lahaina →
`config/lahaina.mk` ✓

---

## ③ 顺带确立的两条硬事实

**A. 两套 release train 命名互斥（200/404 实测）**

| 分支 | `clo/la/kernel/msm-5.10` | `clo/la/kernel/msm-5.4` |
| --- | --- | --- |
| `LA.UM.10.9.1.c25` | 404 | **200** |
| `LA.UM.9.14.1.c25` | 404 | **200** |
| `KERNEL.PLATFORM.1.0.c25` | **200** | 404 |

⇒ `LA.UM.*` = 5.4 世代；`KERNEL.PLATFORM.*` = 5.10 世代。
（`msm-5.10` 共 1274 分支、`msm-5.4` 同量级，已探测确认。）

**⇒ 因此 `audio-kernel` / `camera-kernel` 在 `LA.UM.10.9.1.c25` 上的
`config/lahainaauto.conf` / `config/lahainacamera.conf` 是 5.4 的，不能当 5.10 证据。**
（我中途据此下过一次结论，已作废——这也是为什么必须用 train 归属来判，而不是看配置文件是否存在。）

**B. 首批 manifest 仓库清单（可复用）**

```
clo/la/techpack/audio/manifest      [53544]
clo/la/techpack/camera/manifest     [29587]
clo/la/techpack/display/manifest    [29597]
clo/la/techpack/video/manifest      [29596]
clo/la/kernel/manifest              [41276]
clo/la/kernelplatform/manifest      [29372]
```

---

## ④ 对计划文档的影响（需同步）

| 计划原文（v2） | 修正 |
| --- | --- |
| 「所有公开 OPLUS/OPPO 5.10 树的 techpack 都是 stub」 | 仍成立（OPLUS 侧）。但**高通 CLO 侧三大件齐全**，两条线要分开说 |
| 「techpack 是死路，不存在从 donor 借的可能」 | **完全推翻** |
| 「Phase 2 工期不可估，建议放弃」 | **推翻**。Phase 2 从「自己前移 1839 个文件 / 三大驱动栈」降为「取 CLO 对应分支 + 按 martini 硬件调参」 |
| 「整件事从不可行变成 6 个月量级大工程」 | 现在这个判断才成立。**阻塞点已清除，剩下的是工程量问题，不是可行性问题** |

**仍未解决的是 martini 的设备侧适配**（面板 / 充电 IC / 触控 IC / 相机 sensor 的机型参数），
以及 msm-5.10 侧缺 lahaina 的 vendor defconfig（需自己写）。但这两件都是「工作」，不是「没料」。

---

## ⑤ 可复现命令

```bash
# 1) 分支是否存在（最硬判据）
curl -s -o /dev/null -w '%{http_code}\n' \
  "https://git.codelinaro.org/api/v4/projects/<id>/repository/branches/<branch>"

# 2) 列 config 找 SoC（关键：看 .conf/.mk，不是看分支名）
curl -s "https://git.codelinaro.org/api/v4/projects/<id>/repository/tree?ref=<br>&path=config&per_page=100"

# 3) 读 manifest XML（找 release 用的仓与分支）
curl -s "https://git.codelinaro.org/api/v4/projects/29587/repository/files/<TAG>.xml/raw?ref=release"

# 4) 搜项目 / 列 tag
curl -s "https://git.codelinaro.org/api/v4/projects?search=<kw>&per_page=100&simple=true"
curl -s "https://git.codelinaro.org/api/v4/projects/<id>/repository/tags?per_page=100&page=1"
```

**已确认的 project id**
display-drivers=13664 · audio-kernel=13492 · **audio-kernel-ar=22253** · camera-kernel=13577 ·
camera manifest=29587 · display manifest=29597 · video manifest=29596 · audio manifest=53544
