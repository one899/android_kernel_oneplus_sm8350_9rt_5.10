# techpack 未知 —— 结论（本轮定稿）

> 决定性未知：**5.10 的 lahaina techpack 是否存在。**
> 方法：直接打 CodeLinaro (CAF) 的 GitLab API，用 200/404 做硬判据，不靠命名猜测。

---

## 结论一览

| 子系统 | 5.10 + lahaina | 判据 |
| --- | --- | --- |
| **display** | ✅ **存在** | `display-kernel.lnx.5.10.r11-rel` 里有 `config/lahainadisp.conf`、`config/gki_lahainadisp.conf` |
| **audio** | ❌ **未找到** | `audio-kernel` 的 674 个分支里，内核版本命名族只有 **4.19 / 5.15 / 6.0**，无 5.10 |
| **camera** | ❌ **未找到** | `camera-kernel` 298 个分支，全量过滤 `*5.10*` = **0** |

---

## ① 已确证：display 的 5.10 lahaina 源

**仓库**：`clo/la/platform/vendor/opensource/display-drivers`（project id **13664**）
**分支**：`display-kernel.lnx.5.10.r11-rel`

`config/` 实测 28 项，其中：

```
config/lahainadisp.conf
config/lahainadispconf.h
config/gki_lahainadisp.conf
config/gki_lahainadispconf.h
```

与 `gki_waipiodisp.conf` / `gki_parrotdisp.conf` / `gki_neodisp.conf` /
`gki_holidisp.conf` / `gki_ravelindisp.conf` / `gki_anorakdisp.conf` 并列。
分支族里有 16 个含 `5.10` 的（`.r2-rel`…`.r11-rel`、`.c2`、`.2.c1`、`.2.r1-rel`、`.3.r1-rel`），
另有 `lnx.5.15.10.c1` 佐证 `lnx.<内核版本>` 的命名含义。

---

## ② 决定性判据：两套 train 命名互斥

之前我一度把 `LA.UM.10.9.1` 当成 5.10（因为 `LA.UM.9.14.1` 是 5.4 的 lahaina train，数字更大）。
**用分支探测直接证伪：**

| 分支 | `clo/la/kernel/msm-5.10` | `clo/la/kernel/msm-5.4` |
| --- | --- | --- |
| `LA.UM.10.9.1.c25` | **404** | **200** |
| `LA.UM.10.9.1.r1` | 404 | 200 |
| `LA.UM.9.14.1.c25` | 404 | 200 |
| `KERNEL.PLATFORM.1.0.c25` | **200** | 404 |

（`msm-5.10` 共 1274 个分支；`msm-5.4` 同样规模。）

⇒ **`LA.UM.*` = msm-5.4 世代；`KERNEL.PLATFORM.*` = msm-5.10 世代。两套命名不交叉。**

**推论**：`audio-kernel` / `camera-kernel` 在 `LA.UM.10.9.1.c25` 上的
`config/lahainaauto.conf` / `config/lahainacamera.conf` 是 **5.4 的 techpack**，
不能作为 5.10 的证据。**此前那条推断正式作废。**

---

## ③ audio / camera 的 5.10 源在哪 —— 尚未找到

已排除的：

- `audio-kernel`：674 分支，含 5.10 = 0；内核版本族只有 `audio-kernel.lnx.4.19.*` / `.5.15.*` / `.6.0.*`
- `camera-kernel`：298 分支，含 5.10 = 0；版本族是**驱动版本**（`1.0` / `3.1` / `3.2`…）不是内核版本
- 两者都**没有** `KERNEL.PLATFORM.*` 分支
- `clo/la/platform/vendor/qcom/opensource/github/audioreach/Audio-Kernel` → 分支数 0（可能需登录或已迁走）

**下一步该查的**（按成本排序）：

1. **在 CLO 全站找「带 `KERNEL.PLATFORM.*` 分支」的 audio/camera 仓库。**
   display 用 `lnx.5.10`，audio/camera 可能用 `KERNEL.PLATFORM.*` 或已迁到新仓库。
   方法：搜项目名含 audio/camera，逐个探 `KERNEL.PLATFORM.1.0.c25` 返回码。
2. **查 CLO 的 manifest 仓库**，找 `KERNEL.PLATFORM.*` 那一版 release 的完整仓库清单，
   清单里 audio/camera 的仓与分支就是答案。
3. **反向锚定**：从 display 那条 `lnx.5.10.r11-rel` 出发，找同一 release 的兄弟仓。

---

## ④ 对计划文档的影响

| 计划原文（v2） | 现在 |
| --- | --- |
| 「所有公开 OPLUS/OPPO 5.10 树的 techpack 都是 stub」 | **仍成立**（那是 OPLUS 侧）。但要补一句：**高通 CLO 侧 display 有 5.10 lahaina 源** |
| 「techpack 是死路，不存在从 donor 借的可能」 | **部分推翻**：display 有解；audio / camera 仍未找到 |
| 「Phase 2 工期不可估，建议放弃」 | **需重写**：display 可以借，audio/camera 未知。Phase 2 从「三大驱动栈全部自己前移」变成「display 借 + audio/camera 待定」 |

---

## ⑤ 可复现命令

```bash
# 分支是否存在（最硬的判据）
curl -s -o /dev/null -w '%{http_code}\n' \
  "https://git.codelinaro.org/api/v4/projects/<id>/repository/branches/LA.UM.10.9.1.c25"

# 列分支 / 列目录
curl -s "https://git.codelinaro.org/api/v4/projects/<id>/repository/branches?per_page=100&page=1"
curl -s "https://git.codelinaro.org/api/v4/projects/<id>/repository/tree?ref=<br>&path=config&per_page=100"

# 搜项目
curl -s "https://git.codelinaro.org/api/v4/projects?search=audio-kernel&per_page=100&simple=true"
```

已知 project id：display-drivers=**13664**、audio-kernel=**13492**、camera-kernel=**13577**。
