# 把接手来的 Unity 工程 `Assets` 重新分类整理（安全移动法 + 三级验证）

- 本工作区实测：**2026-10-24**，工程 `<VRCHAT_ROOT>\projects\LuzzyMeow V2`（由 `Lilleo2.1.rar` 6.57 GB 解压；Assets 14,209 文件 / 7,252,382,352 B / 6,916 MB）
- 适用：接手别人的 Unity 工程，`Assets` 里是 `20261016归档`、`兔兔` 这类「来源壳层 + 商品夹混放」，需要按用途分类整理，**且不能破坏现有功能**
- 与 [02-06 素材库重组](02-06-resource-library-reorg.md) 的区别：02-06 整理的是**工程外**的 `resources\` 素材库（只按用途分目录）；本册整理**工程内** `Assets`，每动一个文件都要考虑 **GUID 引用 / Unity 特殊目录 / 插件固定路径 / 编辑器脚本硬编码字符串**。

## 1. 五条安全移动铁律（本工作区实测）

1. **资产文件 + 同名 `.meta` 成对移动**（一次 `os.rename` / `AssetDatabase.MoveAsset`）。GUID 不变 → 场景 / prefab / 材质 / animator / MA / FX 里的 `guid:` 引用全部不断。**绝不重新导入后再删除原件**（GUID 会变）。
2. **Unity 特殊目录不能乱移**：`Editor`、`Editor Default Resources`、`Resources`、`Plugins`、`StreamingAssets`。
   - 实测：`Editor Default Resources` **必须在 `Assets` 根**，否则 FaceEmo 预览窗口找不到 `Assets/Editor Default Resources/Icons/Suzuryg.FaceEmo.Detail....png`（`PreviewWindow.cs:66` 硬编码 `iconDir`）。
3. **插件固定路径必须保留**（移之前先 grep 包内 `Editor/*.cs` 的 `"Assets/` 字符串）：
   - FaceEmo（`jp.suzuryg.face-emo`）：`Assets/Suzuryg/{SystemName}/AutoBackup`、`Assets/Suzuryg/FaceEmo/Generated`、`/Prefab`、`Assets/Suzuryg/FaceEmo/Imported/`
   - ModuleCreator（`com.aoyon.module-creator`）：`Editor/AssetSaver.cs:8` `BASEPATH = "Assets/ModuleCreator"`
   - 其它：AudioLink、lilToon 等自带工具同理
4. **先扫硬编码字符串，再动手**：扫工程内 `.cs / .json / .asset / .txt` 里 `Assets/...`（`scan_hardpath.py` 一类脚本）。区分两类：
   - **读取型**（`File.Exists` / `LoadAssetAtPath` / `iconDir`）→ 移动后必须改路径；
   - **新建输出型**（生成工具的输出目录）→ 可改可不改，但**改后要确认建目录方式合法**（见 §4 坑 1）。
5. **只动资产，不动内容**：场景内 GameObject 层级、对象名（动画路径按名引用！）、菜单显示名、参数名、MA/VRCFury 组件、PhysBone、材质属性一律不改。菜单显示名 / 参数名不在资产路径里，但**动画路径与对象名是引用**，改名等于断绑定。

## 2. 操作流程（全部脚本化，可回滚）

1. **清单**：先列 `Assets` 顶层与各来源壳层结构（文件数 / 字节 / `.meta` 是否成对），决定目标 11 类目录。
   - 本次结构：`00_场景`、`01_模型本体`、`02_服装`、`03_头发`、`04_妆容·表情`、`05_配饰`、`06_道具`、`07_材质·贴图·着色器`、`08_动画·菜单·图标`、`09_工具与插件`、`10_预制体`（与素材库 `resources\` 的 10 类同构，便于以后互相搬运）。
2. **dry-run 脚本**（`reorg_v2_assets.py`）：逐项检查「源存在 / 目标不存在 / `.meta` 成对 / 目标父目录 / 同名冲突」，输出 `moves.json`（源→目标、文件数、字节、GUID 列表）。**没有 dry-run 通过不许执行**。
3. **冲突处理**：同名商品夹加来源后缀，不合并：`可乐（归档）` / `可乐（兔兔）`、`泳装（归档）` / `泳装（兔兔）`。
4. **执行**：按清单移动（文件级 `os.rename`，`.meta` 一起）；然后删除清空的来源容器目录（**连同其 `.meta`**）；记录 `applied.json`。
5. **补移动后的修复**：改读取型硬编码路径（§1 铁律 4）；把被误分类的「工具 / 特殊目录 / 生成物」移到正确位置（本次第二轮 6 项，见 §4）。
6. **留回滚**：`rollback.py`（反向按 moves.json + moves2.json 移回，Unity 关闭时 `--go`）+ `分类整理说明.md`（逐项清单 + 原则 + 验证表 + 回滚说明）。
7. `.gitkeep` 这类根占位文件移到记录目录保存（不要留在 `Assets` 根，除非工程需要）。

## 3. 三级验证（缺一不可）

### 3.1 静态 GUID 引用审计（整理前后各跑一次，集合比对）

脚本 `audit_unity_refs.py <project> <out>`：扫 `Assets`+`Packages` 的 `.meta` 建 `guid→path`，再扫 YAML 资产（`.unity/.prefab/.mat/.anim/.controller/.asset/...`）里的 `guid:` 引用。

通过判据：**未解析引用集合 `set-equal`**（不只是数量相等），`dup_guid=[]`、`orphan_meta=[]`、`meta_without_guid=[]`，可解析引用数不变。

> 本次：22,050 条引用边 / 6,092 个被引用 GUID / 5,596 可解析 / **2,489 条未解析边 / 496 个未解析 GUID 前后完全一致**。这 2,489 条是压缩包里就缺的旧第三方脚本（キメラ生成 Generated 控制器、SoundPad、LilLeo_FX），**不是整理造成的**。

### 3.2 字节级内容核验（最硬证据）

从**原始压缩包**单独解出 `Assets\*` 到临时目录（`7z x <rar> "<内部根>\Assets\*" -r -o<tmp>`），对两边全部文件做 **MD5 多重集**比对：

- `hash_multiset_cmp.py`：先比路径集（移动后必然不同，不作判据），再比 **hash 多重集**；输出 `only_in_current` / `only_in_original` 哈希数量。
- `hash_diff_detail.py`：把两侧独有的哈希映射回文件路径，逐个确认它们都是「有意差异」。
- 通过判据：**内容差异清单只剩可解释项**——本次为 4 个有意改路径的 `.cs`、新增目录 `.meta`（新 GUID）、1 个桥接脚本、`.gitkeep` 与 13 个删除容器的 `.meta`；其余 **14,191 / 14,209 文件逐字节相同**。
- **场景文件单独校验 MD5**：`Assets\00_场景\Lilleo2.0.unity` = `3AB7F25E0D5D6D81DAE9BF3A876D3070`，与压缩包一致 → 证明整理/测试没改场景。

### 3.3 Unity 内运行验证

- 编译 **0 error**（首次导入 `AssetDatabase: script compilation time: 60.5s`）
- 打开场景：根节点数、`VRCAvatarDescriptor` 数、各头像的 `expressionsMenu` / `expressionParameters` / `baseAnimationLayers` 5 层控制器**全部 `LoadAssetAtPath` 非空**（脚本批量读回，别只看一眼）
- `10_预制体` 内 prefab 可 `LoadAssetAtPath`
- 静态扫描：`badShader = 0`、missing script 数与前一致、材质无 `InternalErrorShader`
- **Gesture Manager Play 全菜单回归**（AGENTS「模拟测试铁律」）：菜单项总数 / PASS / FAIL / NOT_RUN 全部记录，截图 287+ 张，判据见 [07-06 Gesture Manager 自动化](../07-验证与排查/07-06-gesture-manager-automated-sim-test.md)

## 4. 必修的坑（本次实测，都是真踩过的）

| # | 症状 | 根因 | 正确做法 |
|---|---|---|---|
| 1 | 改完路径常量后，生成工具点击后报错 / 不生成 | `AssetDatabase.CreateFolder("Assets", "08_动画·菜单·图标/NyaPPu__Generated")` —— **第二参数不能含 `/`**（只能传单级目录名） | 写 `EnsureFolder(path)`：按 `/` 逐级 `AssetDatabase.CreateFolder(cur, parts[i])`；再建子目录。本次修 `Generator.cs`（第一轮改路径引入的 bug） |
| 2 | FaceEmo 预览窗口图标空白 / 找不到 | `Assets/Editor Default Resources` 被当成普通分类目录移进 `09_工具与插件` | 移回 `Assets` 根；FaceEmo 备份数据 `faceemo`（169 文件）是**数据**，归 `04_妆容·表情`，`FaceEmo` 插件目录归 `Assets/Suzuryg/FaceEmo` |
| 3 | 测试后「整理前后内容一致」的结论不成立 | Play/Bake 时项目自身 NDMF 插件会**重写生成型资产**：LightController 的 `NyaPPu__Generated/LightController/*.anim`（按 bake 后克隆层级改路径，如去掉 `with armature`）、ABT 的 `Map_Texture.renderTexture`（Unity 重新序列化） | 测试后从压缩包**原样还原**这 6 个文件（MD5 校验）；并在报告里注明「任何 Bake/Play 都会再次重写，属项目自身行为」 |
| 4 | Play 被 VRCFury 模态框卡死、MCP 全部超时 | 头像上有**未激活**的 `VRCFuryHapticPlug`，VRCFury 1.1276 仍执行 `BakeHapticPlugsService` → `Failed to detect plug length`（`VRCFuryHapticPlugEditor.cs:407`）弹 `DisplayErrorPopup` | 测试场景**副本**里删掉这 2 个 plug 组件再 Play；**原场景/原资产不动**（场景 MD5 证明）。原工程的该问题记为既有问题，建议用户决定是否删未用 SPS |
| 5 | 测试后工程里多出一堆文件 | Play/bake 会生成 `Packages/com.vrcfury.temp/Builds`（本次 21 文件 / 40.7 MB）、临时测试场景 | 收尾删除 `_SimTest.unity`、`Packages/com.vrcfury.temp/Builds`；测试脚本移出工程（放 `tools\scripts\`） |
| 6 | 工程根又冒出 `*.sln` / `*.csproj` | Unity 派生文件，打开工程/编译时自动生成 | 属正常现象；需要干净就删（下次打开会再生成），不要当资产移动 |
| 7 | 删除空容器目录后审计多出 `missing_meta` | 容器 `.meta` 已删但 `Assets` 根还有别的无 meta 文件（`.gitkeep`） | 删除容器 `.meta` 属预期；`missing_meta` 列表要逐项解释（本次 49→47，减少的正是 `.gitkeep` 与新增脚本） |

## 5. 判据 / 通过标准（一页纸）

```text
① dry-run 全绿：0 冲突、0 缺失 .meta、0 目标占用
② 审计前后：unresolved 集合 set-equal、dup_guid=0、orphan_meta=0
③ 字节级：内容差异 100% 可解释（有意改的脚本 / 新目录 meta / 新增脚本），场景 MD5 与压缩包一致
④ Unity：编译 0 error、7 个头像 descriptor 菜单/参数/层控制器非空、prefab 可加载
⑤ GM Play：头像自身菜单 0 FAIL（世界功能 / 世界参数显式标 FAIL-不可测 / NOT_RUN）
⑥ 收尾：临时场景 / VRCFury.temp / 插件重写资产 已清或还原；测试脚本移出工程
```

## 6. 本次实测数字（可作为量级参考）

| 项 | 值 |
|---|---|
| 解压 | 6,565,438,250 B rar → 65,599 文件 / 16.37 GB |
| Assets（整理前） | 14,209 文件 / 7,252,382,352 B |
| 移动项 | 121 项（首轮 115 + 第二轮 6）；删除 13 个空容器目录及 `.meta` |
| 整理后 | 14,210 文件 / ≈6,916.4 MB（11 类 + `Editor` + `Editor Default Resources` + `Suzuryg`） |
| 引用审计 | 5,033 YAML / 22,050 边 / 6,092 GUID / 5,596 可解析 / 2,489 悬空边 / 496 悬空 GUID — **前后 set-equal** |
| 字节比对 | **14,191 / 14,209 文件逐字节相同**；差异 = 4 个有意改的 `.cs` + 新目录 `.meta` + 新增桥接脚本 |
| 场景 | `Lilleo2.0.unity` MD5 `3AB7F25E0D5D6D81DAE9BF3A876D3070`，与压缩包一致；10 根 / 7,189 GO / 7 个 descriptor / 391 SMR |
| GM Play | 162 菜单项 / 327 步 / 64 s：153 PASS、6 FAIL（ABT 世界功能不可测）、3 SETMISS（世界参数）、1 NOPARAM；287 截图 0 粉 0 空 |
| 修复 | 7 项（4 处硬编码路径、EDR、FaceEmo、faceemo 数据、tonnatorn tools、ModuleCreator、Generator CreateFolder） |

## 7. 工具与产物（本工作区）

- 脚本（`<VRCHAT_ROOT>\tools\scripts\`）：`reorg_v2_assets.py`（dry-run/--go）、`fix_phase2.py`、`audit_unity_refs.py`、`hash_multiset_cmp.py`、`hash_diff_detail.py`、
  `gen_manifest_doc.py`、`LuzzyMeowV2SimHarness.cs`（Gesture Manager 全菜单回归）
- 本次工程记录：`<VRCHAT_ROOT>\projects\LuzzyMeow V2\_asset_reorg\`（moves.json / moves.tsv / moves2.json / 分类整理说明.md / rollback.py / `素材整理与验证报告.md` / `evidence\`）

## 8. 未验证 / 边界

- 本方法验证的是「资产位置整理不破坏编辑器内功能」；**多人同步、远端可见性、Contact、saved 持久化、ABT 世界功能**必须进游戏/世界实测，标 `NOT_RUN`。
- 不同插件版本的固定路径会变（FaceEmo / ModuleCreator 等）：**动手前按工程里 `Packages\` 的实际版本 grep**，本册数字是 2026-10 版本快照。

## 9. 场景内 Avatar 层级整理（本工作区实测 2026-10-24，LuzzyMeow V2）

**需求**：接手工程后 Avatar 根下 30 个子物体平铺（Body / 服装 9 套 / 头发 5 组 / SPS / ABT / 8 个 MA 开关代理…），想分组、排序，但「不破坏原有功能」。

**根因 / 边界**：Unity 动画曲线绑定的是 **GameObject 路径（名字 + 层级）**，不是 GUID 引用 → 移动或改名被动画引用的对象 = 断动画；而 `ModularAvatar*`、VRCFury 组件、PhysBone、constraints、
SMR rootBone 用的是**对象引用**，移动父级不影响。另有多头像共用同一套 controller 资产：只改一个实例的路径会破坏其它头像。

**正确流程**：
1. **动画路径扫描**（改前必做）：遍历工程内全部 `AnimationClip`，统计 `EditorCurveBinding.path` 按「头像直属子物体名」的命中数。本次 1,608 clips / 484 命中该头像 / 88,
  420 条曲线 + 224 条 object-reference 曲线；`Body` 44,616、`ABT` 2,396、`睡衣` 464…，而 8 个 MA 开关代理与 `LightController` 命中 **0**。
2. **分级**：零命中 = 可安全移入新容器；命中 > 0 = **只允许同级排序**，不可改名/移父级（除非同步重写全部曲线路径）。
3. **共用控制器计数**：统计每个 controller 被多少 `VRCAvatarDescriptor` 引用（本次 Kipfel 系 5 个被 3 个 Kipfel 头像共用、LilLeo 系被 4 个狼共用）→ 涉及路径改名/移动时必须逐头像复制控制器再改。
4. **备份 + 块级 diff 证据**：备份 `.unity`，改后用 `--- !u!<class> &<fileID>` 分块比较（比逐行 difflib 快几个数量级）：只应出现「新增 GO+Transform 块」「`!u!4` 的 `m_Father`/`m_Children` 变化」，不应有脚本 GUID、参数、动画引用变化。
5. **Gesture Manager Play 全菜单回归**，与整理前基线逐项对比（菜单总数 / PASS / FAIL 分类 / scan 全 0）。

**判据**：块级 diff 只有预期变化；GM Play 菜单总数与 PASS/FAIL 分类与基线完全一致；Play 后原场景 `DIRTY=False`、根数与对象数符合预期。

**实测数字（LuzzyMeow V2，2026-10-24）**：30 → 23 个直属子物体（8 个纯开关代理收进新容器 `菜单开关`，其余 22 个按 功能/骨架/外观/服装 排序）；场景块 16,093 → 16,095（+2 = 新容器 GO+Transform，删除 0）；
变化 39 块 = 8 个代理 `m_Father` + 头像根 `m_Children` + 29 个 VRCFury SPS 组件仅 `unityVersion` 元数据 + `RenderSettings.m_IndirectSpecularColor` 缓存值；
GM Play **162 菜单 / 327 步 / 153 PASS / 6 FAIL（ABT 世界功能）/ 1 NOPARAM / 3 SETMISS / 287 图**，与改动前基线逐项一致。

**已知环境性副作用**：① VRCFury 组件里的 `unityVersion` 会被当前编辑器版本重写（`2022.3.22f1` → `2022.3.22f1c1`），无功能影响；
② `RenderSettings.m_IndirectSpecularColor` 在无烘焙缓存的新会话里会被清 0，重开工程 / 烘焙会重算；③ 头像上若有**未激活的 `VRCFuryHapticPlug`**，
Play 时 VRCFury 预处理会抛 `Failed to detect plug length` 并弹模态框（与层级整理无关，测试副本临时删除即可）。

## 10. NDMF 持久化生成根应按当前源码保留（2026-11-12）

**症状**：整理 Assets 时把 `ZZZ_GeneratedAssets` 当未分类素材，拟移入动画目录，仅以 GUID 不变判断安全。

**机制与裁决**：NDMF 1.14.8 的 `Packages/nadena.dev.ndmf/Editor/AvatarProcessor.cs:100–110`，`ManualProcessAvatar` 明确使用 `OverrideTemporaryDirectoryScope("Assets/ZZZ_GeneratedAssets")`，注释说明该目录用于**不会被临时清理的持久化资产**。即使现有 GUID 引用可随 MoveAsset 保留，后续 Manual Bake 仍会在固定根重新生成；因此不能只按文件内容分类移动它。

**正确做法**：把该目录登记为“NDMF 持久化生成基础设施”，保留固定根；`Editor`、`Editor Default Resources` 也按既有规则保留。普通作者素材目录使用 `AssetDatabase.MoveAsset` 整树移动；读取型硬编码字符串必须同步修正，源/目标全部 GUID 集合逐一比较。

**本轮实证**：7 个普通素材目录完成 MoveAsset，逐目录 root GUID 与子资产 GUID 集合相同；Hash's_Things 两个 patcher 的 CustomfbxPath 同步改新前缀。独立复核未发现意外消失 GUID 或新增断链。保留 `ZZZ_GeneratedAssets` 的依据是上述当前 NDMF 源码，**不只是旧轮次脚本还写着旧路径**。

适用范围：NDMF 1.14.8 当前实现；未来版本或不同平台 provider 需重新读取源码。工程记录 `docs/PLAN-001-r520/T001-moves.txt` 与 `research/independent-review-reorg.md`。Unity 特殊根目录官方依据：<https://docs.unity3d.com/2023.2/Documentation/Manual/SpecialFolders.html>；NDMF 上游源码：<https://github.com/bdunderscore/ndmf/blob/main/Editor/AvatarProcessor.cs>（具体以当前包版本为准）。
