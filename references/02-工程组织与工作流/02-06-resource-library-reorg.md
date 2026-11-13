# 素材库分类与重组（`<VRCHAT_ROOT>/resources`）—— 本工作区实测手册

> 本工作区实测 2026-10-18（第 27 轮）：全库重组（解散来源壳层、模型专属进模型夹、拆掉 `11_大包套件`、重复项改 junction、文档跟商品走）。
> 适用：往库里放新素材、判断某件商品该进哪个目录、要把重复件去重、要写搬迁/整理脚本。

## 0. 何时读

- 用户说「存一下 / 归位 / 整理素材」「库里的眼罩/头发在哪」「这个包该放哪」。
- 要写批量搬迁、改名、去重脚本（§5 的纪律必须先看，踩过两次）。
- 要判断「这个商品是模型专属还是通用」「说明书跟谁走」。

## 1. 一级目录（只有 10 个，按用途）

| 目录 | 放什么 |
| --- | --- |
| `01_模型本体` | 整套 avatar：模型本体、vrm/blend/fbx、配套贴图与授权；店级说明（解压密码 / 教程地址 / 常见问题） |
| `02_服装` | 衣服、套装、鞋、睡衣；`!请先阅读.txt`（材质包提示） |
| `03_头发` | 发型、发饰 |
| `04_妆容·表情·面捕` | 妆容贴图、眼妆、表情集、FaceTracking |
| `05_配饰` | 发卡、名牌、光环、眼镜、耳饰、戒指、猫耳、手环、眼罩 |
| `06_道具` | 可互动道具、玩具、武器与特效、小游戏 |
| `07_成人向` | SPS / PCS / ERP 等成人向插件与道具（**不做批量展示、分享、截图外发**） |
| `08_材质·贴图·着色器` | 皮肤/身体贴图、材质、shader 与素材 |
| `09_工具与插件` | unitypackage 工具、Editor 脚本、优化器、改模插件；`说明（建议看）.txt`、`更新日志.txt` |
| `10_教程与文档` | **只剩无主的店级/库级文档**（店铺横幅、浏览图、来源不明预览图） |

`11_大包套件` 这类「来源不明整包」**已取消**（用户 2026-10-18 拍板）：整包一律按用途拆开归位。

## 2. 命名与模型夹

- 二级及以下**保持原文件标题不改名**（原商品名 / 作者名 / booth 商品码是出处与授权的唯一线索）；`NNN - X` 这种纯编号壳例外 → 改成模型名。
- **模型专属资源进 `<类别>/<模型名>/`**，模型夹固定英文名：`Kipfel`(小猫) / `Mamehinata` / `Maya`(舞夜) / `Rindo`(龍胆) / `Minahoshi` / `Rusk` / `Shipilka` / `Karin` / `Selestia` / `Shinano`。
- **通用商品（多模型适配）平铺在一级目录根部**，不塞进单一模型夹（例：`04_妆容·表情·面捕/4707040_-_Feather_Heart_Eye` 同时给 Karin 与 Shipilka）。
- 同一模型的不同版本（`v1.0.3` / `v1.1.1` / `v1.2.0`）作为并列商品夹保留，**不合并**。
- 压缩包原件跟着商品走，不建「原包馆」；库里默认不留压缩包（用户购入的单件除外，2026-10-18 起库内唯一例外）。

## 3. 文档 / 说明书 / 预览图跟商品走（用户 2026-10-18 强调）

- 商品自带的说明、教程、预览图 → 放进**该商品夹**，不许单独堆在 `10_教程与文档`。
- 只有**店级 / 库级 / 无主**的才留在 `10_教程与文档`：店铺横幅 `闲鱼-初音的VRC小店.png`、`浏览图 1..10.png`、来源不明的预览图。
- 店级说明（解压密码 / 教程地址 / 常见问题 / 说明（建议看）/ 更新日志）放对应一级目录根部。
- 反查方法：预览图文件名通常带商品名前缀（如 Shipilka 的 `(4656724) ShipilkaRoyalCloth-1.avif`），按名字在全库找同名商品夹即可；搬迁日志 `finish_log_shells.tsv` 也能查每张图的原始来源路径。

## 4. 重复项：只留一份，其余做 junction

- 判重优先级：① 目录签名（目录内「相对路径|字节数」排序后 sha1）相同；② 文件映射 `(relpath,size)` 完全一致；③ 名字归一化后相同。
- canon 选择顺序：**无 `(2)` 后缀 > 文件数多 > 字节多 > 路径短 > 路径字典序**。
- 被并入的那份：先 `merge_into(canon)` 逐项合并（同名目录递归、同名同内容删源、同名不同内容改名 `(N)`），再删掉空壳，最后 `cmd /c mklink /J <原路径> <canon>` 建目录联接（junction），保证旧路径仍然可用。
- 已建 junction 24 个；**junction 不算 symlink**：`os.walk(followlinks=False)` 仍会走进去，统计与判重必须自己 `os.lstat(p).st_reparse_tag != 0` 跳过，否则会把自己链接出来的目录再报一遍重复（曾把 7.83 GB 虚报成可回收）。

## 5. 搬迁脚本纪律（踩过的坑，照抄）

1. **`--dry` 必须守住所有副作用**：删、移、改名全都要走 `if not DRY:` 守卫。只守删除不守 `os.rename` 就等于没有 dry-run —— 本工作区真出过两次：`--dry` 跑了 1016 + 345 次真实移动。
2. **改名要防自我嵌套**：`os.rename(A, A/B)` 会 `OSError: [WinError 87] 参数错误`（源与目标同树）；先把子项移出再改名，或对「目录里只有同名子夹」的情况改用 `move-contents`。
3. **命名兜底要有层级**：非通用单子夹名 > 最大 `.blend` > `.fbx` > `.vrm/.pmx` > `.unitypackage` 词干（跳过 `texture/psd/body/fbx/preview` 这类通用名）> `模型-<编号>`；**兜底不能直接用店名**（`N - 淘宝 VRChat模型小店铺` 会让十几家店撞成一个），要 `模型-<N>`。
4. **去重分组的键要排除通用名**：`model (2)` / `Model (3)` / `texture_psd` 这种归一化后全是通用词的名字不能配对，否则会把不相干的模型并到一起。
5. **合并前先看目标存在性**：`run_move(src,dst,can_merge=True)` 目标存在就走 `merge_into`，否则整目录平移。

## 6. 删除审计（删了必须能自证）

- 每次 `run_delete` 都记 TSV：`path / reason / files / bytes`（本工作区落在 `resources/整理日志/2026-10-18-搬迁明细/finish_deleted_*.tsv`）。
- 复算法：把每个删除行的「文件名 + 字节数」拿去全库索引里查同名同尺寸文件，全命中才算安全。本轮 199 + 201 + 31 + 123 行删除**全部命中**（1.24 GB + 2.46 GB + 94.6 MB + 752.5 MB），零丢失。
- 只允许删三类：**同名同内容副本**、**空壳目录**、**解压标记**（`.extract-ok.json`）。

## 7. 本机现状与文件位置（2026-10-18）

- 规则：`<VRCHAT_ROOT>/resources/README.md` ｜ 本轮明细：
  `<VRCHAT_ROOT>/resources/整理日志/_整理日志-2026-10-18-重组.md` ｜ 26 个 TSV：
  `<VRCHAT_ROOT>/resources/整理日志/2026-10-18-搬迁明细/`
  （以上均为**包外**本机实测产物，不随本包分发；目录不存在属正常，不要据此判定文档有误。）
- 一键撤销：`<VRCHAT_ROOT>/docs/撤销-reorg-2026-10-18.ps1`（276 项反向 move/rename；**删除项不可还原**）。
- 规模：**24,136 文件 / 192.896 GB / 23 junction**（基线 34,196 文件 / 198.53 GB；2026-10-18 自检后 +NoNlab 3 文件、-1 个自环 junction，另清掉 `LowPony Bunny` 夹带的 Unity 缓存 9,400 文件 / 427 MB）。

## 8. 交叉引用

- 导入素材进工程：`01-环境与工具链/01-06-project-launch-and-mcp-bridge.md` §2.5（`ImportPackage`）与 §2.8（立刻完全解压）。
- 素材筛选与授权：`07-验证与排查/07-02-evidence-and-ownership.md`。
- 备份与归档：`06-构建上传与备份/06-02-backup-and-package.md`。
---

## 9. ★ Windows 大小写不敏感：判重 / junction 必须按 normcase 比较（2026-10-18 自检补，**血泪教训**）

**事故**：重组后 `01_模型本体\RINDO` 变成**自环 junction**（目标 `01_模型本体\Rindo`），递归工具直接 `ELOOP`；进一步核查确认 **`Rindo` 模型本体（103 文件 / 771,486,244 B ≈ 0.77 GB）已不在库内、
本地不可恢复**（原始压缩包早已按规则删除；全库搜 `.extract-ok.json` 8,191 B / 文件名 `rindo` / `ICON\EMOJI` 结构均无命中）。

**根因**：`RINDO` 与 `Rindo` 在 **Windows/NTFS 上是同一个目录**，但 Python 处理路径字符串时当成两个不同路径 → 判重脚本把同一份内容「合并」到自己身上，删掉了 canon 的文件，再 `mklink /J` 建出目标指向自己的自环。

**为什么原审计没发现**：审计脚本的「全库索引」会顺着 junction 走 → 自环里的同一份文件被当成「同名同尺寸副本」匹配（自己匹配自己），于是报「无数据丢失」。

**正确做法（以后必须遵守）**：

1. 路径比较用 `os.path.normcase(p)`（Windows 下转小写）或统一 `str.lower()`；**判重键 = normcase(路径)**，不是原始字符串。
2. 判重/建 junction 前先检查「源与目标是否同一路径（normcase 后相等）」→ 相等就直接报错或跳过，**绝不允许建 junction**。
3. 建完 junction 立刻验证：`os.path.realpath(link) != normcase(link)`，且 `os.path.realpath(link)` 不是 link 自己（防自环）；再跑一次目录列举确认能打开。
4. **删除审计不能用「顺着 junction 的全库索引」**：索引必须跳过 reparse point（`os.lstat(p).st_reparse_tag != 0`），否则自环/重复链接会自己匹配自己。
5. 审计数字与文件都要能区分「真实文件」和「junction 重复遍历」：本工作区统计口径一直是 junction-aware（跳过 reparse point）。

**用户决定（2026-10-18）**：**Rindo 模型本体不再恢复**（「那个模型本体不用了」）—— 接受 0.77 GB 损失。

**处置记录**：自环 junction 已 `cmd /c rmdir` 删除；junction 24 → 23；`resources\README.md` 与整理日志 §八已同步。


---

## 10. 找素材：别只翻 `<类别>/<模型名>/`（用户 2026-10-18 经验）

`<类别>/<模型名>/` 只是「已归类的模型专属资源」；**没归到模型夹、甚至在别的模型夹里的资源，也可能是这个模型适配的** —— 找兼容资源要整个类别目录一起搜。

**本次实测例子（Kipfel）**：

- 头发：`03_头发/Kipfel/` 之外还有 `Looseshort/`（キプフェル対応）、`小短发/`（VerMAMEHINATA&Kipfel）、`ChibiFuwaTwin/`、`5914038/`、`matinee/` 等可用；判定依据＝readme 或 unitypackage/fbx 文件名里带 Kipfel。
- 服装：`02_服装/Kipfel/` 之外还有 `pomae/`（PomaePenPenParka_Kipfel）、`森うさぎおーばーおーる/`（MoriUsagiOverall_Kipfel）、`nemunemu pajama/`（Mamehinata＆Kipfel）、`Amplifier/`、
  `Blanchir/`、`BunnyPicnic/`、`House Bunny/`、`CREPSCOLO/` 等。

**检索命令**：

```powershell
# 目录名 / 文件名 / readme 里出现模型名即命中
Get-ChildItem '<VRCHAT_ROOT>\resources\03_头发' -Recurse -File -Force |
  Where-Object { $_.Name -match '(?i)kipfel' }
```

**注意**：unitypackage 可能在多一层子目录（例 `02_服装/Kipfel/70065708Hoodie set/NoemHoodie_Kipfel_ver1.00/NoemHoodie_Kipfel_ver1.00.unitypackage`），别只列第一层就断言没有。

**装资源**：MCP `execute_code` 里调 `AssetDatabase.ImportPackage` 会阻塞主线程导致 MCP 请求超时（大包尤其明显、甚至把 stdio 桥卡死）→ 改用「手动解 unitypackage」：gzip→tar，
每个 GUID 目录含 `pathname` / `asset` / `asset.meta`，按 pathname 落盘并写 `asset.meta`（**目录条目只有 asset.meta，也要写**）以保留 GUID，最后 `AssetDatabase.Refresh()`。
