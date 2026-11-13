# 面捕（OSC / VRCFT 架构）：接入、接线与修复

> **来源**：原 `08-症状速查` 第 142–160 轮实测条目（2026-11-06 至 11-08，LuzzyMeow V2 沐梓溪狼/喵）结构化整理；与 [04-07 VRCFT 参数前缀修复](04-07-vrcft-parameter-prefix-repair.md)、[00-04 面捕基底](../00-基底/00-04-face-tracking.md) 配套。
> **适用**：VRCFT + OSC（`FT/v2/*` 参数）+ FX 动画层方案。**原生 Face Tracking（SDK 3.10.6+）见 [04-09](04-09-face-tracking-native-sdk.md)**。
> 版本标注均为「检查过」的实测版本，不是通用常量；照用前按工程 `Packages/vpm-manifest.json` 核对。

## 目录

1. 架构与数据流（OSCmooth 编码层）
2. 参数与控制器接线（声明完整性 / 类型对齐 / 路径基准 / 控制器抽取 / 直连方案）
3. 眼睑、眨眼与眼追（极性 / 默认值 / 索引槽 / 三源 / 2D 树 / TrackingControl / 手势门控）
4. 面捕菜单与开关（Button+Bool 脉冲 / 归属判定 / Installer 语义 / saved）
5. 与 Avatar Optimizer 的交互（改名 / 删曲线 / 删键链条）
6. 补丁类插件（Hash hpatchz / meta 补丁 / 可绑定率 / Pawlygon 恢复）
7. 网格与形态键迁移（引用统计 / 三角形双射 / 键合并 / globalScale / Blender Transfer）
8. 验证边界与来源

---

## 1. 架构与数据流

```text
头显/面捕硬件 → VRCFT（OSC 发送）→ VRChat 客户端 → avatar 参数 FT/v2/*
  → FX 控制器（MA MergeAnimator 合入）→ BlendTree/clip → SkinnedMeshRenderer blendshape
```

- 参数前缀丢失（`FT/v2/` 变 `v2/` 等）会让整条链静默失效，OSCQuery 秒级定案法见 [04-07](04-07-vrcft-parameter-prefix-repair.md)。
- **OSCmooth 编码层**：作者控制器（如 Pawlygon `FTOnly.controller`）可能带 `_OSCmooth_Gen` + 一批 `_OSCm_<参数>_Encode` 层，把 `FT/v2/*` 打包写进 `OSCm/Proxy/FT/v2/*` 再给主树读。**工程不跑 OSCmooth 补丁时，主树读到的 proxy 恒 0，整张脸不动**（第 159 轮实测：42 层里 38 层是编码层）。
  - **重装/接入必须两步改造**（本工作区实测 2026-11-07，`Assets/Editor/R264FtReinstall.cs`）：
    1. 倒序删掉所有 `name.StartsWith("_OSCm")` 的层（42→4）；
    2. 把参数名 `OSCm/Proxy/FT/v2/X` 归一成 `FT/v2/X`，同步改 `BlendTree.blendParameter / blendParameterY / children[].directBlendParameter` 与全部转移条件（本例 37 参数 / 123 处引用）。
  - 判据：改完主树 direct 参数全部是 `FT/v2/*`，OSCQuery 有值时形态键跟随。
- **同步位预算**：面捕插件参数极易超 256 bits（Bool 1 / Int·Float 8）：
  - LilLeo FT EP：**146 个 synced、共 202 位**；工程烘焙已 209/256 → 直接接入必超（第 142 轮）。
  - Kipfel `Kipfel_ExParameters.asset`：**171 项 / 139 同步 / 209 位**；目标头像已用 65 位 → 合计 274 > 256。只同步 `OSCm/*`（114 位，65+114=179 可过），`FT/v2/*` 37 项设 NotSynced（第 144 轮）。
  - **面捕控制类参数全部 localOnly**（`FT_Enable / EyeTrackingActive / LipTrackingActive / VisemesEnable / FacialExpressionsDisabled / FTEffects / EyeDilationEnable / FTOn / FTOff`）：EP `networkSynced=false`，OSC 仍可写、OSCQuery 仍暴露，但不占位。实测源 EP 90 参数/71 bits → NDMF 烘焙 174 参数/259 bits 超限 → 改后 222 bits（第 159 轮，`Assets/Editor/R270Budget.cs`）。
  - 位宽权威算法与 localOnly 的边界见 [03-01 §10](../03-菜单与参数/03-01-parameter-and-menu-audit.md)。

## 2. 参数与控制器接线

### 2.1 控制器参数声明完整性（差集扫描）

**症状**：眼动/嘴部对参数完全无响应，但同一棵树的眼睑分支正常（第 160 轮）。
**根因**：BlendTree 引用的参数在 AnimatorController 里根本没声明（OSCmooth 剥离时只归一了已有参数，漏掉真实参数）。Animator 读不存在的参数恒 0，整棵子树输出 0。本例缺 `FT/v2/EyeLeftX、EyeRightX、EyeY、JawOpen、MouthClosed、LipFunnelLower、FTEffects/FTOn/FTOff` 共 10 个。
**配方**：
1. 遍历所有 BlendTree 的 `blendParameter / blendParameterY / children[].directBlendParameter` + 全部转移条件参数，与 `ctrl.parameters` 求差集（本例 `used=46 missing=10`）；
2. 缺的按 Float 补进 controller，并在 `ModularAvatarParameters` 同步声明（`localOnly=true`）；
3. **VRChat 内建参数**（`GestureLeftWeight`、`Voice` 等）不算缺；拼写笔误要改正确名（本轮 6 个子项 `FT/EyeTrackingActive → LipTrackingActive`）。
**判据**：补参后 `SetFloat("FT/v2/EyeLeftX",1)` → `eye_look_R=100`。

### 2.2 三处类型对齐（按类型名，不按枚举数字）

同一参数在三处各有一份类型声明（第 151 轮实测）：

| 位置 | 枚举 |
|---|---|
| 菜单 | Toggle / Button（控件类型） |
| EP `valueType` | **Int=0 / Float=1 / Bool=2** |
| Animator `m_Type` | **Float=1 / Int=3 / Bool=4** |

- **Bool 与 Float 的枚举值恰好错位**，按字面数字比对必然误判；核对一律按类型名对齐。
- 转场条件：Bool 用 `If/IfNot`，Float 用 `Greater/Less 0.5`。
- 本例 Bool 参数（`VisemesEnable/FacialExpressionsDisabled/FTOn/FTOff`）三处都是 Bool；Float 参数（`EyeTrackingActive/LipTrackingActive/FTEffects`）三处都是 Float。**Hash 原包 FT EP 把三个 Float 写成 Bool，工程侧已修正为 Float——不要按原包回改**。
- EP↔FX 全量类型校验脚本口径见 [03-01 §10](../03-菜单与参数/03-01-parameter-and-menu-audit.md)。

### 2.3 MergeAnimator 路径基准毁掉面捕（Relative 前置拼接）

**症状**：面捕插件、菜单、`WolfFT.controller` 全在，烘焙后 FT 层被改名 `(... NO VALID ANIMATIONS)`，面部形态键也被删光（第 149 轮）。
**根因**：`/FT_Merge` 的 `ModularAvatarMergeAnimator.pathMode=Relative`，而 `WolfFT` 的 187 条曲线全是根相对写法（唯一路径 `Body`）。MA 把组件所在路径 `FT_Merge` 作为 basePath 前置拼到每条曲线 → `FT_Merge/Body`（不存在）→ 曲线全失效 → AAO `optimizeBlendShape` 判定这些键没人用 → 一并裁掉。
**源码依据**：`MergeAnimatorProcessor.cs:48-59`：`if (pathMode == Absolute) return "";` 否则返回 `RuntimeUtil.RelativePath(avatarRoot, targetObject)`。
**判据与修法**：
- 先看曲线路径写法：clip 的 `path` 以 avatar 根下一级对象名（`Body`、`armature/...`）开头 → 设 `pathMode=Absolute`；`Relative` 只用于「曲线本来就是相对该组件写的」场景。
- 修完断言：烘焙后 FT 层曲线数 >0，且曲线绑定名能在烘焙网格上找到。
- 同构坑（RK Player 音量曲线 `RK Player/RK Player/...` 双拼接）见 [02-11 §2](../02-工程组织与工作流/02-11-plugin-integration-rk-realkiss-props.md)；官方 path mode 语义见 [03-07 §1.3](../03-菜单与参数/03-07-ma-platform-sync-mmd-and-motion.md)。

### 2.4 作者 FT 控制器是整套 FX：只抽 FT 层

**症状**：以为「把面捕控制器挂上去就有面捕」，结果一半动作没有，还可能覆盖目标头像自己的 FX（第 144 轮）。
**根因**：Kipfel 作者控制器 `Kipfel_FXLayer_v1.0.1 FT.controller` 是整套 FX（含 `Outfit Shirt SleeveChange`、`AFK`、`[ USER EDIT ] Face`、`OutfitOneClick`），不是纯 FT 层。
**做法**：复制控制器 → 只抽 `FT` / `Binary_FT` / `Decode` 层 → 再 MA MergeAnimator。
**判据**：烘焙后不出现 Outfit/AFK 层。

### 2.5 参数直连方案（不跑 OSCmooth 打包）

**症状**：直接合并 Kipfel 面捕控制器 44 层，烘焙后参数怎么驱都没反应（第 145 轮）。
**根因**：烘焙产物里 **41 个 `_OSCm_*_Encode` 层权重被置 0**，`FT/v2 → OSCm 打包 → 解码 → 形态键` 整链断掉。
**做法**：本地参数不需要打包——自建 **1 层 Direct BlendTree**，每参数直连一个原生键（`MeowFT_Direct.controller`）。
**两个硬坑**：
1. **`AnimatorState.writeDefaultValues` 必须 `true`**：WD=false 时混合权重归零不写回默认值。实测 0:0 → 1:100 → 回:100（FAIL）；改 true 后 0/100/0 全 PASS。
2. clip 做成 **0→100 两帧**而非常值曲线。

## 3. 眼睑、眨眼与眼追

### 3.1 EyeLid 极性反转（面捕开眼却闭眼的第一根因）

**症状**：开启面捕后眼睛始终闭着，关掉面捕正常（第 156 轮）。
**根因**：**VRCFT `EyeLidLeft/Right` 语义是 0=闭 / 1=睁**（Pawlygon README：*Eyelid values (0 closed, 0.75 normally open)*），而 blendshape `EyeClosedLeft/Right` 是 100=闭。移植脚本把两者 1:1 直连（Direct BlendTree child 权重=参数值 → clip 0→100），于是参数 0.75（睁眼）驱动 EyeClosed=75（闭眼）。
**判据**：OSCQuery 采样 `/avatar/parameters/FT/v2/EyeLidLeft` 停在 0.75~0.99（睁眼）而模型闭眼；烘焙 FX 该通道 clip 只有 `blendShape.EyeClosedLeft` 0→100 且 `m_DirectBlendParameter=FT/v2/EyeLidLeft`。
**修法（必须反相）**：换成 1D 混合树 `Simple1D on FT/v2/EyeLid{Left,Right}`，阈值 **0→闭（EyeClosed=100）/ 0.8→中性（全 0）/ 1.0→睁大（EyeWide=100）**。混合树作 Direct 树子项时外层权重要用常量 1 参数（本例新增 `FT_LidWeight` Float def=1），否则 Direct 树把子项权重乘 0。
**对照**：作者原版 `Assets/Hash's_Things/LilLeo/anim/controllers/FTOnly.controller` L2 的 `FreeformCartesian2D` pos 0.10=Closed / 0.80=Neutral / 1.00=Wide。

### 3.2 反相混合树的默认值必须是「中性睁眼」档

**症状**：改成 1D 混合树后，没接 OSC / 关掉面捕时眼睛全闭（反相只修好一半，第 156 轮）。
**根因**：混合树 `blendParameter` 的默认值决定无输入时落哪一档。`FT/v2/EyeLidLeft` 的 `AnimatorControllerParameter.defaultFloat` 与 EP `defaultValue` 都留 0，而 0 是 thr=0 的全闭档。
**修法**：默认值设 **0.8**（Pawlygon README normal=0.75），**FX 控制器 `defaultFloat` 与 EP `defaultValue` 两处都要改**。另先确认网格真有 `EyeWide*` 键（本例烘焙后没有，wide 档曲线空转，已改写为中性 `EyeClosed=0`）。
**判据**：Play 设 EyeLid=1/0.75/0 三档，`EyeClosedLeft` 默认 0 时恒 100（全闭）；改默认 0.8 后三档恒 0（睁开）。

### 3.3 Descriptor 眼睑槽是 Int32 索引数组

**症状**：关掉面捕也不眨眼（第 156 轮）；换素体后担心眨眼/眼神静默失效（第 143 轮）。
**根因**：`customEyeLookSettings.eyelidsBlendshapes` 是 **Int32 索引数组，语义 [Blink, LookingUp, LookingDown]**，不是名字引用。本例 `[-1, 20, -1]` → Blink 槽空、LookingUp 槽被错填成 `eye_look_UP_VRC`（index 20）。
**修法**：填 **Blink = `EyeClosed` 的索引**（本例 272；网格真正的眨眼键 `EyeClosed`(272) / `eye_blink`(22)），LookingUp/Down 无键留 -1。`EyelidType=Blendshapes` 时 VRChat 只用一个合并眨眼键。
**换网格硬规则**：重建网格时**前 N 个键的顺序必须保持**；换完读回 index 做断言，不能只看名字对。

### 3.4 眨眼三源叠加（眨一次模型眨两次）

**症状**：开面捕后模型自带眨眼仍在 / 眨一次眨两次（第 158 轮）。
**根因**：三源叠加——① VRChat 原生模拟眨眼（`eyelidsBlendshapes[0]=EyeClosed`）；② 作者 `BlinkAuto` 层（写 `eye_blink`）；③ FT `EyeLid→EyeClosed`。而 `FT Eye Tracking` 层默认态是 **Native**（`trackingEyes=1`=Tracking）→ 反而打开原生眼动与眨眼。
**官方依据**：State Behaviors 文档「Setting Eyes & Eyelid to Animation will disable eye movement and eyelid blinking」。
**修法**：默认态改 **`VC Eye Tracking`（trackingEyes=2）**，转移极性 `EyeTrackingActive Less/Greater 0.5`；`BlinkAuto` 加 `FT_Enable` 门控（AnyState→OFF `If`，OFF→ON `IfNot`）。
**判据**：Play 300 帧取样：FT 关 `blinkMax=99.7` / FT 开 `blinkMax=0.0`。

### 3.5 凝视方向：单极直连会被夹到 0..1

**症状**：面捕注视方向瞳孔完全不动（数据在流也没用，第 158 轮）。
**根因**：`WolfFT_Tree`(Direct) 的凝视只接了 3 条单方向正向直连（`EyeLeftX→EyeLookOutLeft`、`EyeRightX→EyeLookOutRight`、`EyeY→EyeLookUpLeft`）；Direct 子项权重=参数值并被夹到 **0..1**，负值恒 0 → 向内/向下/右眼纵向全无输出。
**判据**：Play 实测 `FT/v2/EyeLeftX=+1` 只动 `EyeLookInLeft/EyeLookOutRight`；`=-1` 只动 OutLeft/InRight；`EyeY=-1` 完全不动。
**修法**：用 **SimpleDirectional2D**（`EyeLeftX`×`EyeY` / `EyeRightX`×`EyeY`，5 向 clip：Neutral/In/Out/Up/Down，左右 x 镜像），子树权重参数用 `EyeTrackingActive`（1=FT 驱动 / 0=原生眼动）。工具 `Assets/Editor/R220FtEyeFix.cs`。

### 3.6 TrackingControl 枚举语义（状态名可能是反的）

**症状**：`FT Visemes` 层状态名与参数极性看起来完全反了（`Visemes Enabled --If VisemesEnable--> Visemes Disabled`），差点当 bug 改转场（第 151 轮）。
**根因**：状态名按「动画口型是否启用」命名，参数按「面捕 viseme 是否接管」命名；真正决定行为的是 `VRCAnimatorTrackingControl.trackingMouth` 的枚举值。
**枚举**：`TrackingType：NoChange=0 / Tracking=1 / Animation=2`。本例 `trackingMouth=Tracking(1)` 在 `Visemes Disabled` 态、`Animation(2)` 在 `Visemes Enabled` 态，与 Hash 原版行为等价。
**规则**：**判面捕开关极性只看 `trackingMouth` 枚举值**：`Tracking`=面捕/OSC 接管口型，`Animation`=走 avatar 动画口型。状态名不能作判据。

### 3.7 开面捕时手势表情仍生效：门控与静音 clip

**症状**：开面捕时手势表情仍然生效（不是游戏设置，是接线缺陷，第 158 轮）。
**根因**：作者 4 个表情层（`Left Hand`/`Right Hand`/`BlinkAuto`/`WolfFace TangSmile`）在主 FX 里没有任何面捕门控；`AnimatorConditionMode` 极性易写反（**`If`=参数为真、`IfNot`=参数为假**）；「统一静音 clip」经 AAO rebase 后 159→56 条、丢失 23 条 mouth_* 覆盖。
**修法**：
1. 门控只用 `FT_Enable`：**`If`=面捕开→禁手势，`IfNot`=面捕关→恢复**；
2. 静音 clip 必须**与本层绑定集完全一致、值全 0**（每层一个），AAO rebase 后才保覆盖；
3. 手势态→`Muted` 加转移，`Muted`→Idle 反向转移。
**判据**：出包/烘焙 mute clip 与该层 clip 绑定集比对（gesture 79 条里 23 条未覆盖即缺陷）；Play FED=1 时 gestureSum 应为 0。工具 `Assets/Editor/R228GestureMuteClip.cs`。

### 3.8 舌头通道：两个 LongStep 叠加拉扯与参数笔误

**症状**：舌头「完全变形、不是原样」（第 160 轮）。
**根因**：作者 `TongueOut JawOpenTarget` 分支在 `JawOpen=0` 时走 `TongueOutJawOpen0LongStep1/2` 两个键**同时 100**（一个上抬 avgΔY+0.0029、一个前伸 avgΔZ+0.0061）→ 舌体被两个方向拉扯；且 `Tongue Axis` 的 6 个子项 direct 参数误写成不存在的 `FT/EyeTrackingActive`（应为 `LipTrackingActive`）。
**判据**：读 clip 曲线 `TongueOutJawOpen0LongStep1=100` 且 `...Step2=100`；`Tongue Axis` 子项 `directBlendParameter=="FT/EyeTrackingActive"`。
**修法**：用户要「舌头保持建模原样」→ 舌头分支 clip 全部清零（10 个 tongune 键=0），`Tongue Axis` 的 5 个 clip 也换零 clip，参数怎么变舌头都不动。要做伸舌只写**单个** `TongueOutLongStep2`（前伸），不要 Step1+2 叠加。产物 `Assets/08_动画·菜单·图标/FT_v2/Clips/WOLF_Tongue_*.anim`。

## 4. 面捕菜单与开关

### 4.1 Button + Bool = 脉冲触发器（正确设计，不是 bug）

VRChat 的 `Button` + **Bool 参数**是脉冲语义：置 1 后自动回落 0；配合 `IfNot` 条件做一次性触发器（第 151 轮）。
- 本例 `FT OnOff` 层：`Waiting --If FTOn--> FTOn 1`，`FTOn 1 --IfNot FTOn--> Waiting`，两态 driver 分别把 4 个参数置 1/置 0，`hasExitTime=False, dur=0`。
- **前提：EP 里该参数 `saved=0`**（否则记住 1 卡在触发态）。Toggle 反而会让 `IfNot` 分支永不成立。
- 对照：**Button 绑 Int 做三选一是错误用法**（脉冲回落 0 永远停在 0 档），见 [03-06 §18](../03-菜单与参数/03-06-ma-menu-build-by-reflection.md)。

### 4.2 菜单归属判定（是不是原作者的、有没有被改坏）

三处交叉，齐了才敢说「未改」（第 151 轮）：
1. 资源路径是否落在作者包目录（本例 `Assets/Hash's_Things/LilLeo/menu&params/`）；
2. `.asset.mtime` 是否早于本次会话；
3. Unity 里 `AssetDatabase.LoadAssetAtPath` 读 `controls` 与用户截图逐项比对（含富文本颜色）。
注意：MA 会把菜单项合并进根菜单，磁盘上根 `controls` 只有 2 项、真正的第 3 项在场景组件里——不能只看磁盘资产。

### 4.3 MenuInstaller + MenuItem 同对象是正常写法

`VirtualMenu.PushNode(MenuInstaller)` 源码（`VirtualMenu.cs:145-163`）：`if (GetComponent<MenuSource>() != null) PushNode(...) else if (menuToAppend != null) PushMenuContents(...)` —— 有 MenuItem 时 `else if` 整段不执行（第 151 轮）。
- 两者同对象：MenuItem 提供内容、Installer 提供目标（`installTargetMenu`），**不会**在根菜单和子菜单各出现一次。
- 只有 `menuToAppend != null` 且同对象没有 MenuSource 才会追加外部菜单；不要「修」成两个入口。

### 4.4 面捕参数的 saved 边界

- 记忆只由 EP `saved` 决定，且 **saved 只对同步参数生效**；本地参数（`networkSynced=false`）写 `saved=true` 是空操作，SDK 判 `savedButNotSynced`（第 152 轮）。
- 本例 8 个 FT 参数全在 `synced=False` 集合，saved 开不上属预期。要记忆先改同步（Bool=1 bit / Float=8 bits）。
- **`FTOn/FTOff` 是 Button 脉冲键，不应该 saved**（会卡触发态）；只记忆 `FT_Enable` 等状态量。
- 完整 saved 规则见 [03-01 §8–§10](../03-菜单与参数/03-01-parameter-and-menu-audit.md)。

## 5. 与 Avatar Optimizer（AAO）的交互

### 5.1 optimizeBlendShape 改名毁面捕（面捕头像必须关）

**症状**：面捕「瞳孔不跟视线 / 嘴型很差 / 舌头变形」同时出现；参数有值、层齐全、VRCFT 在发数据，脸上几乎没反应（第 160 轮）。
**根因**：`Anatawa12.AvatarOptimizer.TraceAndOptimize.optimizeBlendShape=true` 把烘焙网格所有形变键合并+重命名为 `AAO_Merged_<原名>_<序号>`（本例 347→185 键），面捕 clip 仍绑原名 → 所有曲线静默失效（Unity 不报错）。
**判据**：烘焙克隆 `Body.sharedMesh.blendShapeCount==185` 且 `GetBlendShapeIndex("eye_look_L")==-1`、键名以 `AAO_Merged_` 开头；源网格同键 idx=16。
**修法**：面捕头像**必须关闭** `optimizeBlendShape`（`SerializedObject(comp).FindProperty("optimizeBlendShape").boolValue=false`）。关后烘焙仍走 AAO 但不改名：`shapes=347`、`AAO_Merged=0`。代价是保留全部键（面捕必需）。

### 5.2 判断形态键存活不能用原名精确匹配

AAO 保留下来的键也会重命名为 `AAO_Merged_<原名>_<序号>` 并同步改写动画曲线（第 149/158 轮）：
- 烘焙后 `AAOGeneratedMeshBody` 148 键，`GetBlendShapeIndex("EyeClosedLeft")==-1`，但 `AAO_Merged_EyeClosedLeft_93` 存在；曲线绑定同名 `Body :: blendShape.AAO_Merged_EyeClosedLeft_93`。
- **动态断言**：按 `AAO_Merged_` 前缀模糊匹配（`name.Contains(原名)`），或「设参数前后全权重差分取 Top-N」；**不要写死原键名**。只有「曲线名在网格里查不到」才是真的被删。

### 5.3 AAO 删除「未使用动画属性」曲线

**症状**：自建 BlendTree 的 `AudioSource.m_Volume` 曲线烘焙后被删光，只剩 `$AvatarOptimizerClipLengthDummy$`（第 156 轮）。
**根因**：AAO「删除未使用动画属性」pass（`debugOptions.skipRemoveUnusedAnimatingProperties` 默认 false）。
**修法**：`TraceAndOptimize.debugOptions.skipRemoveUnusedAnimatingProperties=true` 再烘焙。**不要**改写给父 AudioSource 写曲线（父对象无 clip、mute=true，写了也不响）。
注意与 §2.3 的 Relative 双拼接区分：AAO 开/关双烘焙结果相同 → 是路径问题不是 AAO（隔离实验法）。

### 5.4 路径失效 → AAO 删键的连锁

换头系统拆除后旧 clip 路径仍是 `armature/Hips/Spine/Chest/Neck/Head/狼头/Body`，而狼头已并入根级 `Body` → `av.Find(path)` 全 null，Unity 静默忽略 → AAO 判键无人使用（第 156 轮）。
- 实测：35 个 `.anim` 共 **5,012 条**失效曲线。
- 修法：批量把 path 改成实际存在的路径（本例 `Body`）；改完**必须重导 + 重新统计命中率**（blink_on 1/1、face_zeroshape 193/193）。判据是「命中率 100%」，不是「文件已改」。

## 6. 补丁类插件（Hash / Pawlygon / Kipfel）

### 6.1 hpatchz 二进制补丁：基底必须逐字节相同

Hash 系面捕补丁器（`Tools/Hash/* Patcher`）用 `hpatchz` 做二进制 diff，基底必须是作者打包时逐字节相同的 FBX（第 142/144/145 轮）：

| 案例 | 期望字节 | 实际字节 | 结果 |
|---|---|---|---|
| LilLeo 工程内重导出 FBX | 6,432,208 | 18,161,792 | `oldFile dataSize … != diffFile saved oldDataSize …` |
| LilLeo 260115 原包解出 FBX | 6,432,208 | 6,432,208 | `patch ok`，产出 6,396,620 B |
| Kipfel v1.2.0（工程内） | 9,142,828（只适配 v1.0.3） | 9,233,356 | 失败；meta 补丁却能 ok |

**规则**：
- 先按「期望字节数」反查是否有原版包（如 `LilLeo_260115.unitypackage`）；有原版就别改工程内 FBX。
- 版本不符时**不要**把半成品 prefab 接进 avatar；补丁器不产出 ARKit 形态键时，面捕只能是空壳。
- 同一套 `.meta` diff 能 `patch ok` 不代表 FBX 能打——meta 补丁只恢复 GUID 与导入设置，不恢复几何。
- Kipfel 版本不符的替代路线：参数直连原生键（§2.5），31 通道可覆盖眼/嘴/眉/舌。

### 6.2 meta 补丁成功 ≠ prefab 可用

**症状**：meta 补丁打成功，但 `LilLeo FT.prefab` 仍是 `BrokenPrefabAsset`（Load 返回 null，第 143 轮）。
**根因**：作者 prefab 里 **216 处**引用 FBX 的 GUID `913c0e…`；本工程 FBX 是 `a357…`（重导出过）。
**判据**：`grep -c "guid: 913c0e…"` 于 prefab；`AssetDatabase.GetMainAssetTypeAtPath=BrokenPrefabAsset`；`LoadAllAssetsAtPath` 返回 0。
**做法**：不要走「整套 prefab 拖入」；只从补丁产物里取形态键，挂到现有网格；原 FBX 旁挂不覆盖。

### 6.3 可绑定率：先算再决定方案

**症状**：面捕装上后「眼睛嘴都不动，眉毛偶尔动」（第 144 轮）。
**根因**：Kipfel 插件 1,028 条 `Body` 曲线里 **444 条（43%）目标是 UE 键**（`JawOpen`/`EyeClosedRight`/`MouthSmileLeft`），目标网格只有日系键 → 曲线目标缺失。**缺失曲线 Unity 静默忽略、不报错**。
**配方**：逐条比 `blendShape.<名字>` 与网格键名集合，算可绑定率（本例 584/1028 可绑，核心动作全在缺失侧 → 必须重定向或补键）。

### 6.4 Pawlygon clip 被改坏：从原始 unitypackage 原样恢复

**症状**：第三方面捕补丁包 clip 目标键与作者控制器对不上（曲线在、驱动无效，第 156 轮）。
**根因**：前一阶段「跨模型重定向」把 `Assets/Hash's_Things/Pawlygon/Animations/**` 的 90 个 clip 的 `blendShape.*` 改成了日系键（`eye_close_left` 等），而 `FTOnly.controller` 混合树按 ARKit 键（`EyeClosedLeft`/`EyeWideLeft`）寻址。
**修法**：**从原始 unitypackage 原样恢复**（`tarfile` 读 `<guid>/asset` 写回同名路径），不要手改曲线名。恢复后缺失 152→46 条（余下是该网格本就没有的键）。

## 7. 网格与形态键迁移

### 7.1 换网格前先量「动画侧引用基线」

单看网格键数没用，要看动画侧（第 143/144 轮）：遍历 5 个播放层，统计 `b.path=="Body"` 的 `curveBindings` 去重：
- LilLeo 狼：**2,373 条曲线 / 195 个键名**（193 个实际存在）；视素 15 按名、PhysBone 89 按骨。
- Kipfel 喵：**1,028 条 / 372 键**。
**规则**：先量这组数字再动手；迁移后数量必须完全一致，任一变化即回滚（「零断链」验收基线）。

### 7.2 顶点数不同 ≠ 拓扑不同：三角形角点双射

**症状**：三份网格顶点数不同（5169/5845/6284），以为不能合并形态键（第 145 轮）。
**根因**：顶点数差异只是 UV/法线缝隙拆分不同，三角形数与顺序完全一致（5900/2422/80），包围盒逐位相同。
**做法**：比较 `GetTriangles(i).Length/3` 与 `bounds`；相同即同拓扑。用**三角形角点**建立顶点双射（`poly[i].vertices[k] → polyB[i].vertices[k]`），再按 delta 迁移。实测 4,679/4,679 零冲突（`docs/_evidence/merge_shapes.py`）。

### 7.3 FT 版网格不能直接当新素体（键集合差）

FT 版（315 键）= 现有 208 + 107 面捕键；**不含** 260115 独有的 32 个日语键（`ex_tang`/`ex_tang2`/`ex_tang_off` 等）（第 145 轮）。
- 集合差：`260115独有 vs FT = 32`，`FT独有 = 107`，`当前∩FT = 208`。
- 必须「FT 网格 + 260115 的 32 键」合并成 347 键；单取任一份都会丢东西。

### 7.4 Blender FBX 往返缩放：globalScale=0.01

Blender 合并形态键后 FBX 导回 Unity 模型放大约 100 倍（导出 `FBX_SCALE_ALL`，第 145 轮）：导入后 `mesh.bounds.size≈26.7` 而原版 0.267。
**修法**：Unity 侧 `ModelImporter.globalScale=0.01` 后 `SaveAndReimport`；不必重导。与 [00-05 §globalScale 陷阱](../00-基底/00-05-mesh-blendshape.md) 同源。

### 7.5 Blender 合并形态键的官方口径

官方手册（Shape Keys Panel specials）：`Transfer Shape Key` / `Join as Shapes` **只要求顶点数一致，不校验拓扑顺序**（第 143 轮）。
**做法**：先比坐标/法线序列哈希确认顺序；不一致先重排再 Transfer；或直接按「新基底 + delta」逐顶点计算。

## 8. 验证边界与来源

### 8.1 面捕专用验证边界

- **GM / 动画参数不能伪造面捕输入**：BlendshapesSet 与 FT 层的输入是面捕参数/硬件键，GM 无面捕输入时恒 0；视觉面捕（眯眼/眼追/嘴型）只能用真实 OSC/VRCFT 输入或 VRChat 客户端验证（第 172 轮，详见 [04-09 §11](04-09-face-tracking-native-sdk.md)）。GM 只验参数/菜单/门控链路。
- OSCQuery 参数树采样是秒级定案手段（[04-07 §2](04-07-vrcft-parameter-prefix-repair.md)）。
- 编辑器 Play 不产生 MA 烘焙产物、driver 不执行：动态验证分层与替代证据见 [07-09](../07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md)。
- 编辑器改形态键后出图不变：`ForceSkinnedUpdate` 修法见 [07-09 §6](../07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md)。

### 8.2 来源

- 本工作区实测 2026-11-06 至 11-08（第 142–160 轮），原载 `08-症状速查` 02/03/04 增补册，2026-11-11 结构化并入本册。
- Pawlygon README（Eyelid 0=closed / 0.75 normally open）；VRChat State Behaviors 官方文档（Eyes & Eyelid = Animation 关闭原生眼动与眨眼）。
- MA 源码：`MergeAnimatorProcessor.cs:48-59`、`VirtualMenu.cs:145-163`；Hash 补丁器 BOOTH <https://booth.pm/en/items/7499168>；Kipfel 面捕包（哈希家 5946836）。
- 关键脚本（工程内）：`R264FtReinstall.cs`、`R270Budget.cs`、`R220FtEyeFix.cs`、`R228GestureMuteClip.cs`、`MeowFT_Direct.controller`。
