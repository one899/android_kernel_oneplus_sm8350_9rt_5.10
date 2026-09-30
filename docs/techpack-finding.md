# techpack 未知的进展 —— 部分确证，部分待定

> 本文件记录对「5.10 是否有 lahaina techpack」这一**决定性未知**的调查进展。
> 方法：直接查 CodeLinaro（`git.codelinaro.org`）的 GitLab API。
> 结论按置信度分开写，**不要把推断当成已确证**。

---

## ① 已确证：5.10 的 lahaina **display** techpack 公开存在 ✅

**仓库**：`clo/la/platform/vendor/opensource/display-drivers`（project id 13664）
**分支**：`display-kernel.lnx.5.10.r11-rel`

分支名里的 `lnx.5.10` **就是内核版本**（该仓库同时存在 `display-kernel.lnx.5.15.10.c1`）。
该分支有 16 个含 `5.10` 的分支，包括 `.r2-rel` … `.r11-rel`、`.c2`、`.2.c1`、`.2.r1-rel`、`.3.r1-rel`。

`config/` 目录里直接有 lahaina 的配置（实测列出 28 项）：

```
config/lahainadisp.conf
config/lahainadispconf.h
config/gki_lahainadisp.conf
config/gki_lahainadispconf.h
```

并与 `gki_waipiodisp.conf` / `gki_parrotdisp.conf` / `gki_neodisp.conf` /
`gki_holidisp.conf` / `gki_ravelindisp.conf` / `gki_anorakdisp.conf` 并列
—— 典型的「多芯片统一代码库 + 每目标 conf」。

**⇒ 计划文档里「techpack 是死路、5.10 无 lahaina 源」的结论，至少对 display 已被推翻。**

---

## ② 待定：audio / camera

### 已查到的

| 仓库 | `config/` 里有 lahaina | 分支 |
| --- | --- | --- |
| `clo/la/platform/vendor/opensource/audio-kernel` (13492) | `config/lahainaauto.conf` + `.h` ✅ | `LA.UM.10.9.1.c25` |
| `clo/la/platform/vendor/opensource/camera-kernel` (13577) | `config/lahainacamera.conf` + `.h` ✅ | `LA.UM.10.9.1.c25` |

camera 的 config 里同时有 `waipiocamera.conf` / `shimacamera.conf` / `yupikcamera.conf` / `monacocamera.conf`，
也是统一代码库形态。

### 为什么不能直接下结论

这两个仓库**没有版本号命名的分支**（全量拉取实测）：

- audio-kernel：674 个分支，含 `5.10` 的 = **0**
- camera-kernel：298 个分支，含 `5.10` 的 = **0**

它们用发布 train 命名（`LA.UM.*`、`CAMERA.LA.*`、`audio-drivers.lnx.1.0.*`、
`auto-camera-kernel.lnx.1.0.*`）—— 注意后两者里的 `.lnx.1.0` 是**驱动自己的版本号**，
不是内核版本，和 display 仓库的 `.lnx.5.10` 含义不同。

### 一条被我自己推翻的推断

我一度假设 `LA.UM.10.9.1` = 5.10（因为 `LA.UM.9.14.1` 是 5.4 的 lahaina train，数字更大应该更新）。
**实测否定了这个假设**：

- `clo/la/kernel/msm-5.4` 里有 `LA.UM.10.9.1.c25`、`LA.UM.10.9.1.r1`、`LA.UM.10.9.1.r1.1`
- `clo/la/kernel/msm-5.10` 扫描 800 个分支，**`LA.UM.*` 前缀出现 0 次**（它只用 `KERNEL.PLATFORM.*` 与 `aosp-new/*`）

⇒ `LA.UM.*` 是 **msm-5.4 世代**的命名，train 编号与内核版本无关。
所以「audio/camera 的 `LA.UM.10.9.1.c25` 是 5.10」**不成立**，那更可能是 5.4 的 techpack。
本节结论**撤回**，保持待定。

---

## ③ 下一步怎么把 audio / camera 钉死

三个可选路径，按成本排序：

1. **反查 display 那条分支对应的 train。** `display-kernel.lnx.5.10.r11-rel` 属于某个 BSP 发布，
   同一发布里的 audio/camera 分支就是答案。可查 CLO 的 manifest 仓库
   （`clo/la/` 下的 manifest 项目，需要先确认其项目路径）。
2. **直接验证候选分支能不能配 5.10 内核编译。** 取 audio/camera 的 lahaina conf，
   在 msm-5.10 树上试编 —— 能过就是对的。这是最硬的判据，但要先有可用的构建环境。
3. **查 `KERNEL.PLATFORM.*` 命名的 techpack。** 既然内核侧 5.10 用这个命名，
   对应世代的 techpack 可能也有同名分支 —— 本项目**尚未在 audio/camera 里搜过 `KERNEL.PLATFORM`**，
   这是最便宜的下一个动作。

---

## ④ 对计划文档的影响（待同步）

| 计划原文 | 修正 |
| --- | --- |
| 「所有公开 OPLUS/OPPO 5.10 树的 techpack 都是 stub」 | 仍然成立（那是 OPLUS 的树）。但**高通 CLO 侧**有 5.10 的 lahaina display 源 —— 两条线要分开说 |
| 「techpack 是死路，不存在从 donor 借的可能」 | **display 已推翻**。audio/camera 待定 |
| 「Phase 2 工期不可估，建议放弃」 | **需要重新评估**。若 audio/camera 也确认，Phase 2 从「自己前移三大驱动栈」降为「取 CLO 对应分支 + 按 martini 硬件调参」 |

---

## ⑤ 本次用到的可复现命令

```bash
# 列出仓库分支（GitLab API）
curl -s "https://git.codelinaro.org/api/v4/projects/clo%2Fla%2Fplatform%2Fvendor%2Fopensource%2Fdisplay-drivers/repository/branches?per_page=100"

# 列目录 / 取文件树
curl -s "https://git.codelinaro.org/api/v4/projects/<id>/repository/tree?ref=<branch>&path=config&per_page=100"

# 搜项目
curl -s "https://git.codelinaro.org/api/v4/projects?search=display-drivers&per_page=20&simple=true"
```

已知 project id：display-drivers=13664、audio-kernel=13492、camera-kernel=13577。
