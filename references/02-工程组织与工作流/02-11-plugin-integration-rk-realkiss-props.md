# 互动插件集成：RK Player / RealKiss / 道具接线

> **来源**：原 `08-症状速查` 第 142/146/147/148/156/158/166/167/171/172/177 轮实测条目（2026-11-06 至 11-11，LuzzyMeow V2 沐梓溪狼）结构化整理。
> **分工**：RK Player / AudioLink 的教程口径见 [02-09 §9](02-09-project-workflow-notes.md)；AFK 可见性 MergeAnimator 做法见 [03-06 §17](../03-菜单与参数/03-06-ma-menu-build-by-reflection.md)；道具两级显隐的 GM 验证见 [07-06 §22](../07-验证与排查/07-06-gesture-manager-automated-sim-test.md)；ABT 工具绑点见 [05-09](../05-动骨与穿模/05-09-third-party-tool-prefab-binding-abt.md)。

## 目录

1. RK Player（CD 播放器）：音频接线 / 控制器坑 / 路径基准 / CD 封面 / 扩盘 / 转码 / 取图 / 摆位
2. RealKissSystem：版本安装 / RKSActivator / 菜单与接触参数 / 口裂定标
3. 道具接线通用坑（GUN1 条件反相 / 两级显隐）
4. 属性→层反查脚本（谁在抢同一属性）

---

## 1. RK Player（CD 播放器）

### 1.1 装上没声音：音频要自己接

**症状**：RK Player 装上后没有声音，切歌只换 CD 贴图（第 142 轮）。
**根因**：包内 `song` AudioSource 的 **clip 为空**，`RK song sel con` 层是**空壳**（9 个 state 全无 motion），`RK song select` 只驱动 CD 视觉。
**判据**：读 prefab `AudioSource.clip=null`；读控制器 `RK song sel con` 每态 `motion=null`。
**做法**：8 个 AudioSource（对应 8 个 CD 槽）+ 自建 FX 层按 `RK_song select` 切 `m_Enabled`、按 `RK_pause` 写 `m_Pitch` 0/1 做暂停；层挂在 `RKSongFX` 的 `MergeAnimator(Relative)` 上（路径基准见 §1.3）。
**进游戏直接开始播**：插件 prefab 里 **10 个 AudioSource 的 `playOnAwake` 全是 true**；Animator 只切 clip，并不负责「是否播放」。全部置 false，播放交给菜单触发的动画/开关，默认静音（第 146 轮）。

### 1.2 控制器四个坑（带子常显 / 开关失效 / 状态机丢失 / 音量曲线绑错对象）

1. **黑色斜挎带始终在腰上**（播放器已隐藏、`Belt.mesh` act=False）：`RK_fix pos` 层 `defaultState=fix belt`，其 clip 把 `Belt.mesh`/`belt` 写成 **1**，该层 w=1 每帧覆盖场景值（第 148 轮）。修法：加 `master off` 状态（clip 写 0），所有状态加 `RK_ON IfNot` → `master off`、`master off` 加 `RK_ON If` → `ready`。**配件显隐必须挂在总开关参数上**，不能只依赖位置层。
2. **`RK ON/OFF` 状态机完全不起作用**：`RK_switch` 层 **`defaultWeight=0`** 整层被禁用（第 148 轮）。改控制器前先断言所有参与层 `defaultWeight>0`；`AddLayer()` 新建层默认权重也是 0，必须手动设 1（struct 整体写回）。
3. **换歌/播放/暂停全无反应**：`RKSongFX.controller` YAML 里两层 `m_StateMachine: {fileID: 0}`——**状态机整个丢了**，只剩空壳层（文件仅 54 行，第 148 轮）。判据 `L.stateMachine==null`。**原地重建**：`ctrl.layers = new AnimatorControllerLayer[0]` → `AddLayer` → `sm.AddState` + `AddAnyStateTransition`，再 `SaveAssets`；**不要删资产重建**（GUID 会变，引用全丢）。
4. **音量/暂停曲线都在却没效果**：曲线目标 `Armature/root/cd player/song` 上的 AudioSource 是空壳（`clip=NULL`、`mute=True`）；真正播放的是子节点 `song1..8` 各自的 AudioSource（`playOnAwake=True`、`loop=True`，第 148 轮）。**音量/音高曲线必须绑定真正持有 clip 的那一层**（`song/song1..8`），用 `AudioSource.m_Volume`、`m_Pitch` 建曲线（两者都可被 AnimationUtility 正常读回）。

### 1.3 MergeAnimator 的 Relative 路径以「组件所在 GameObject」为基准

**症状**：自建 `RKSongFX.controller` 挂好后 24 条曲线一条都不生效（第 147 轮）。
**根因**：组件挂在 `RK Player/RKSongFX` 子节点上，路径 `Armature/...` 被解析成 `RK Player/RKSongFX/Armature/...`（不存在）。**不是以父级或 avatar 根为基准**。
**判据**：遍历 clip 曲线用 `SMR.transform.Find(binding.path)`，命中 0/88。
**做法**：把 MergeAnimator **挂在动画路径的起点对象上**（此例 `RK Player` 根），再设 `pathMode=Relative`；校验逐条 `Find(path)` 命中率必须 100%。
**变体（双拼接）**：组件在 `RK Player` 上，曲线 path 又写成 `RK Player/Armature/…` → MA 拼成 `RK Player/RK Player/…` → 曲线被丢、clip 改名 `rebased <原名>` 只剩 `$AvatarOptimizerClipLengthDummy$`（第 156 轮；**与 AAO 无关**，AAO 开/关双烘焙结果相同）。Relative 模式下曲线 path 必须相对组件所在对象（改成 `Armature/root/cd player/song/song1..8`）；改完烘焙读回 `m_Volume` 16 条、路径 `RK Player/Armature/…`。同构的面捕案例见 [04-08 §2.3](../04-外观与表情/04-08-face-tracking-osc-and-repair.md)。
**验证口径**：MA 层是烘焙期合并的，源 FX 只有 11 层看不出 RK 层（11 vs 118）；任何 MA 层验证都必须 Manual bake 后读 clone 的 FX（第 148 轮）。

### 1.4 CD 封面：8 首歌 ↔ 8 张盘 ↔ 8 个材质槽

- 每张盘是**独立材质** `CD 1..8.mat`（shader `Hidden/lilToonTransparent`），**同一个贴图要写 3 个槽**：`_MainTex`、`_BaseMap`、`_BaseColorMap`（后两个是别名，一并写无害）；对应场景对象 `RK Player/CD1..CD8` 的 SMR（第 156 轮）。
- **序号映射**：出包菜单 `/功能/RK Player Menu/Song Select` 的 8 个 Toggle 依次是 `RK_song select = 0..7`；`RKSongFX` 状态 `RSFX_sel0..7` 分别激活 `song1..8`，`song N` 由 `CD N.mat` 出图 ⇒ 序号 N 一一对应。换封面按序号放 `Covers/N.png`；不确定先跑 `CD Cover - Report` 打印每个槽当前指向（第 156 轮）。
- **一键工具**（`Tools/Meow Verify/CD Cover - Report / - Import`，`Assets/Editor/R187CdCoverImport.cs`）：投放目录 `Assets/RIOkun/RK player/Song/CD cover/Covers/`，文件名 `1.png…8.png`（或 `song 1..8.png`）；导入设置 `textureType=Default / mipmapEnabled=false / maxTextureSize≥512`（另一处口径：`Default / sRGB / alphaIsTransparency / mipmap / Clamp / max 512`）。**先 Report 再 Import**。
- **素材缺口不是机制缺陷**：包内官方说明只要求「`Song > CD cover` 的材质主色纹理放 1:1 方图」，**没有**按曲目自动换封面的功能；`Covers/` 空目录时显示的是 `Sample/song N.png`（第 158 轮）。
- **安全系统触发时 lilToon 的 alpha mask 会被忽略** → 圆盘可能变方，封面用不透明深色四角方图更稳（第 158 轮）。

### 1.5 扩展第 9/10 张盘：四件套 + ParentConstraint

**症状**：新增 CD9/CD10 后切歌盘面仍是旧封面、切回旧歌新盘残留、盘永远不显示、或浮在播放器上方（第 167/171/172 轮）。
**根因与四件套**：
1. **材质**：新建 `CD 9.mat` 写 `_MainTex/_BaseMap/_BaseColorMap`；已空出的 song2 槽（值 1）直接改 `CD 2.mat` 即可，不必新建盘。
2. **SMR 与骨骼**：复制 `cd1`（含 `cd1.end`）→ `cd9`，复制 `RK Player/CD1` SMR → `CD9` 并把 `bones[4]` 的 `cd1.end` 换成 `cd9.end`；换骨后重算 `bindPoses`（[05-11 §1](../05-动骨与穿模/05-11-skinning-binding-and-geometry-repair.md)）。
3. **scale 显隐曲线**：`sel N.anim` 靠 `Armature/cd root/cdN` 的 `m_LocalScale.x/y/z` 显隐；`sel 8.anim` 需 `cd9=1` 且 cd1..cd8=0，**所有 `sel 0..7/9/ready` 都要补 cd9/cd10=0 曲线**。判据：逐 clip 读回 `cd1/cd9/cd10` 的 `m_LocalScale.x`（`sel 0`→1/0/0，`sel 8`→0/1/0，`sel 9`→0/0/1）。
4. **激活曲线**：CD 盘 GameObject 靠 `RK ON.anim`/`RK OFF.anim` 的 `m_IsActive` 曲线激活（CD1..CD8 有，CD9/CD10 没有 → 恒 inactive）。**补 `CDN | m_IsActive`**；判据要在烘焙产物里读（路径会被重写成 `RK Player/CDN`）。
**浮空根因**：新建盘只改了模型位置，没有复制 cd1 的 `ParentConstraint` → 对象保留世界坐标不跟随槽位。把 cd9/cd10 的 localPosition/Rotation 对齐 cd1，复制 cd1 的 `ParentConstraint`，运行时 scale 由歌曲动画控制；验证两条约束距离均 < 0.005m（第 172 轮）。
**叠盘根因（第 132 轮）**：新加 CD 盘的骨骼 `localPosition` 与 `cd1` 完全重合（盘叠在同一槽位），且 `localBounds` 某轴为 0 被视锥剔除 → 封面空/不显示。逐盘核对 `Armature/cd root/cdN.localPosition` 互不相同；`sel N.anim` 的 scale 矩阵 10×10 核对；`CDN.localBounds` 统一 (2,2,2)。

### 1.6 音频转码：FLAC → OGG 两条路线

- Unity 音频导入**不支持 FLAC**；`sf.write` 一次性写 1000 万帧会崩（exit 127）。用 **libsndfile 流式写 OGG**（`sf.SoundFile(...)` + 每 50 万帧 `write`），44.1k 立体声 ≈3.8MB/首；8 首共 29MB（第 142 轮，`docs/_conv_audio.py`）。
- **ffmpeg 路线的 Theora 坑**：FLAC 内嵌 `METADATA_BLOCK_PICTURE`（专辑封面）被 ffmpeg 默认当视频流编码成 Theora，生成的 OGG 含 `Stream #0:0: Video: theora`，FMOD 不接受（Unity 报 `Errors during import of AudioClip` + `FSBTool ERROR: Internal error from FMOD sub-system`；由 WAV 转的同批 OGG 正常，第 166 轮）。**只映射音频流**：
  ```
  ffmpeg -i in.flac -map 0:a:0 -vn -sn -dn -c:a libvorbis -q:a 4 -ar 44100 out.ogg
  ```
  判据：`ffmpeg -i` 只有 1 条 Audio 流、`AudioClip.length>0`。

### 1.7 官方专辑封面取图（QQ 音乐）

`new_json=1` / 网易云 / iTunes 接口对当前网络不可用；**`ct=19` 的专辑搜索**与 `y.gtimg.cn` 图片 CDN 可用（第 167 轮）：
1. `https://c.y.qq.com/soso/fcgi-bin/client_search_cp?ct=19&w=<专辑名>&format=json&...` → `data.song.list[0].album.mid`（《叶惠美》=`000MkMni19ClKG`、薛之谦《尘》=`000DMpJ73yeITP`）；
2. `https://y.gtimg.cn/music/photo_new/T002R800x800M000{mid}.jpg` 实测可下载（183,730 / 200,729 B）。
**下载后必须目视核对封面文字再写入材质**；维基百科/百度百科 og:image 在本机直连 403/SSL EOF，不要当主路径。

### 1.8 蒙皮道具摆位：不要只挪父节点

把 `RK Player` 根节点挪到腰上，模型还在原地/看不见：CD/player 是 **SkinnedMeshRenderer**，位置由骨骼决定；移动父节点不改骨骼（bounds≈1 单位、中心在原点，第 146 轮）。蒙皮道具要么重绑骨骼（MA Bone Proxy / 重指 rootBone），要么用插件自带的 Player Position 动画。通用原理见 [05-11 §2](../05-动骨与穿模/05-11-skinning-binding-and-geometry-repair.md)。

## 2. RealKissSystem

### 2.1 版本与安装

- **lilToon 2.x 与 RKS 1.0.x/1.1.x 不兼容**（材质发紫/报错）；工程 lilToon=2.1.8 时选 **1.4.x**（包内同时有 1.3.2 / 1.4.0-beta.2，第 142 轮）。BOOTH <https://booth.pm/ja/items/5354575>。
- 整个 `Assets/SuperNekoya/RealKissSystem` **挂到 Head 骨骼下**再调位置（嘴部），不要挂 root。
- **菜单不在 prefab 根节点**，而在同名子节点 `RealKissSystem/RealKissSystem` 上（根上 `GetComponent<MenuInstaller>` 得 null）；改 `Resources/RealKissSystem MA Menu.prefab` 或子节点的 `installTargetMenu`。
- 同步参数只有 2 个（mode Int + custom.expression.enabled Bool），预算友好。

### 2.2 右手食指碰嘴无反应：缺 RKSActivator

**症状**：菜单齐全、接触体齐全，但手指碰嘴毫无反应（第 159 轮）。
**根因**：本体 `Head/RealKissSystem` 只有 **receiver**（`…contact.mouth` / `…contact.activator` / `…contact.dpsactivator`）；`superneko.realkiss.contact.activator` 的 **sender 来自 `RKSActivator`**，场景里没有这个对象（全场景搜 `Activator`=0）。
**做法**：从 `Assets/SuperNekoya/RealKiss/RKSActivator.prefab` 实例化到**右手食指末节 `IndexDistal.R`**，位置旋转归零；其 Sender tag 即 `superneko.realkiss.contact.activator`，正好对上头部 Receiver。工具 `Assets/Editor/R266Tweaks.cs`。

### 2.3 菜单名与接触参数

- 出包菜单里它叫 **`/功能/RealKissSystem`**（英文，不是「亲吻系统」）；5 项：Auto / World Fixed / DPS Orifice / Disabled / [Saved] CustomExpression。**用中文关键词过滤会误判为缺失**——排查菜单缺失一律全量 dump 菜单树再按名查（第 156 轮）。
- 8 个接触体（Sender3 / Receiver5，r=0.03，Leader 0.06）挂在 `armature/…/Head/RealKissSystem`，世界 z=0.0799；接触体参数被 MA 加 `$<hash>` 后缀（如 `superneko.realkiss.contact.mouth$91388a5c02cd`）属正常。

### 2.4 口裂定标与半径调参（GM 可模拟 Contact）

第 177 轮实测（2026-11-11）：
- **定标**：Body 网格 `JawOpen` blendshape delta 质心（989 顶点）world=(0,0.6418,0.0740) 即口裂中心；RKS root `localPosition.y -= 0.0059` 使口侧 Receiver 从 0.6477（上唇）落到 0.6418。
- **半径**：口侧 Sender/Receiver/Activator/DPS 0.03→0.02（Leader 0.06 不动）。
- **判据（GM 实测）**：临时 Sender 贴口部 activator=1；前移 5.5cm=0（旧半径组合 0.06 会触发、新 0.05 不触发，判别成立）；临时对象 Play 内销毁不存盘。
- **口径取代**：本条实测取代「Contact 必须真机」的过宽旧口径——**GM ModuleVrc3 可模拟接触触发**；但跨玩家 R2R/Leader 链仍 NOT_RUN（需客户端）。
- **姿态口径（复核闭环）**：定标指 rest pose；Play idle 时 Head 骨 +14.2mm、Receiver 随 Head 同步偏移（`docs/_r510_close.txt`），功能判别以 Play 时坐标自洽。

## 3. 道具接线通用坑

### 3.1 GUN1 装备条件反相（装备后武器消失、放下才出现）

**根因**：`SIHG_Control` 的装备条件写反：`AnyState -> Hide_L` 用了 `SIHG_Equip If 0`，`Hide_L -> Mount_L` 又用了 `IfNot 0`（第 172 轮）。
**判据**：读控制器条件 + Play 中把 `SIHG_Equip` 从 0 切到 1，观察 `Hide_L` 与 `Mount_L` 方向。
**修法**：改为 `AnyState -> Hide_L: SIHG_Equip IfNot 0`；`Hide_L -> Mount_L: SIHG_Equip If 0`。保留菜单 `道具 > Sica_HandGun`，并把 `SIHG_FireSelect` 默认值设为 1。文件 `Assets/06_道具/GUN1/Weapon/GUN_HG_Sica/Animation/_L/AM_IRR_SIHG_L.controller`；工具 `R400Fix.cs`。
**通用教训**：作者状态机的 `If/IfNot` 极性要按「参数真值→目标状态」运行时断言，不要按状态名推断（[04-06 §14](../04-外观与表情/04-06-ma-reactive-components.md)）。

### 3.2 道具「菜单开关 ≠ 可见」：两级显隐

gimmick 类道具常是两级：菜单 Bool 只让容器 `m_IsActive=1`，真正显形在另一层状态机按内置 **`GestureLeft`** 驱动（本例 Open=2 拔枪、RockNRoll=5 收枪，`spin→gunon` 与 `off→default` 都带 exitTime）。验证用例必须显式补 `GestureLeft` 并给足帧（≥90 帧）；详见 [07-06 §22](../07-验证与排查/07-06-gesture-manager-automated-sim-test.md)。

### 3.3 AFK 机器动画按钮无可见机器

AFK prefab 只有动画资产，缺参数驱动的可见性层。新建 `R400_AFK_Visibility.controller` 与 `R400_AFK_On/Off.anim`，在 AFK 根挂 MA MergeAnimator（`AFK=true` 显示 / `false` 隐藏），烘焙后读回层与 clip（第 172 轮）。**内置参数 AFK 不要写进 EP**（duplicate definition），见 [03-01 §10](../03-菜单与参数/03-01-parameter-and-menu-audit.md)；完整做法见 [03-06 §17](../03-菜单与参数/03-06-ma-menu-build-by-reflection.md)。

## 4. 属性→层反查脚本（谁在抢同一属性）

多层写同一 `path+property` 时，**后面的层覆盖前面的**（Override 模式）；翻遍 118 层太慢（第 148 轮）。
**配方**：遍历所有层 clip 的 `GetCurveBindings`，按 `path.Contains("Belt.mesh") && propertyName=="m_IsActive"` 收集层名，确认只有一个层负责该属性（本例仅 `RK_switch`(66) 与 `RK_fix pos`(76) 触碰带子，二者用同一 `RK_ON` 收敛）。
**扩展**：一键穿衣多 driver 写同一参数时「后求值的层赢」，每个参数只允许一个权威写者——见 [03-06 §19](../03-菜单与参数/03-06-ma-menu-build-by-reflection.md)。

## 5. 来源

- 本工作区实测 2026-11-06 至 11-11（第 142–177 轮相关条目），原载 `08-症状速查` 02/03/04/05/08 增补册，2026-11-11 结构化并入本册。
- BOOTH：RealKissSystem <https://booth.pm/ja/items/5354575>；RK Player（RIOkun）；GUN1（Sica_HandGun）。
- 关键脚本（工程内）：`R187CdCoverImport.cs`、`R226CdCover.cs`、`R266Tweaks.cs`、`R400Fix.cs`、`R400AfkFix.cs`、`docs/_conv_audio.py`。
