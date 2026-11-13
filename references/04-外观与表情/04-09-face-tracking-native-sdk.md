# 原生 Face Tracking（SDK 3.10.6+）：配置、映射与修复

> **来源**：原 `08-症状速查` 第 143/161/164/165/172/176/177 轮实测条目（2026-11-06 至 11-11，LuzzyMeow V2 沐梓溪狼）结构化整理。
> **适用**：VRChat Avatars SDK 3.10.6+ 的**原生 Face Tracking**（客户端直接驱动 blendshape，"outside the animator"）。OSC/VRCFT 动画层方案见 [04-08](04-08-face-tracking-osc-and-repair.md)；早期调研见 [04-04](04-04-native-face-tracking-two-heads.md)。
> 官方文档：<https://creators.vrchat.com/avatars/face-tracking/>、<https://creators.vrchat.com/avatars/face-tracking/osc>。

## 目录

1. 版本前提与升级
2. Descriptor 字段与最小配置
3. BlendshapesSet 映射（103 索引 / 别名表 / 拆分 / Auto-Fill 边界）
4. Gaze 通道语义（8 单极 / highest wins / multiplier）
5. EyeGazeMode 三态与 SDK 预览 bug
6. 几何幅度判别（瞳孔质心法）
7. Animator 覆盖与 FT 专用键（零顺序依赖）
8. 眯眼高光漂移（hightlight_* 共用顶点）
9. 静态残留与表情层覆盖
10. OSC → 原生迁移的回归与防复发
11. 消费者输入边界（GM 不能伪造面捕）
12. 面捕菜单（降级 / 重定向四步）
13. effects 动画树与主链分工
14. 来源
15. 高光归零补丁引起的虹膜遮挡与按眼补偿

---

## 1. 版本前提与升级

- **SDK 3.8.2（2025-06-25）没有原生 FT**：Descriptor 序列化属性只有 `lipSync`/`VisemeSkinnedMesh`/`enableEyeLook`；全程序集扫描无 `*FaceTracking*` 类型。原生 FT 自 **Avatars SDK Beta（开发者更新 2026-09-24）** 起提供（第 143 轮）。3.8.2 上继续用 VRCFT + FX 方案；要原生 FT 须单独评估升级，别顺手升。
- **SDK 3.10.6 升级必须成对**：VRCFury ≥1.1442.0（否则 `CS0618 TypeCost obsolete` 编译中断）、AAO ≥1.9.21、MA ≥1.18.x、NDMF ≥1.14.x。升级流程与 VPM 镜像取包见 [01-01 §8](../01-环境与工具链/01-01-environment-and-toolchain.md)。
- 升级后**必须重跑一次 Manual Bake 并逐项比对**（EP n/bits、菜单项数量、`MA Responsive` 层数；本例 EP 175/216 不变、兜帽卫衣 6 项不变、4 个联动层不变），再上传（第 161 轮）。

## 2. Descriptor 字段与最小配置

3.10.6 新增 Face Tracking 区块，字段全在 `VRCAvatarDescriptor` 上（第 161 轮反射实测）：

| 字段 | 类型/语义 |
|---|---|
| `enableFaceTracking` | bool 总开关 |
| `useAllTrackingKeys` | bool 使用全部追踪键 |
| `faceTrackingMeshes` | `List<SkinnedMeshRenderer>` 目标网格 |
| `faceTrackingBlendshapesSetList` | `List<FaceTrackingBlendshapesSet>` 映射集 |
| `faceTrackingParameterBehaviors` | 追踪键 → animator 参数转发（门控用） |

关键枚举：`FaceTrackingKey`（**103 项**，末项 Max）、`EyelidType.FaceTracking=3`、`CustomEyeLookSettings.EyeGazeMode.Both=2`、`LipSyncStyle.VisemeBlendShapeFaceTracking=5`。

**最小可用配置**：
```text
enableFaceTracking = true
useAllTrackingKeys = true
faceTrackingMeshes = [Body]
faceTrackingBlendshapesSetList = [set]
customEyeLookSettings.eyeGazeMode = Both
eyelidType = FaceTracking
lipSync = VisemeBlendShapeFaceTracking   ← 只有这一项才有 Viseme Dampen
```

## 3. BlendshapesSet 映射

### 3.1 按枚举索引的字符串数组（不是字典）

`FaceTrackingBlendshapesSet.faceTrackingBlendshapeNames` 是**按 `FaceTrackingKey` 枚举索引**的字符串数组（**长度必须 = 103**），不是「键名→blendshape」字典（第 161 轮）。配好 set 后仍有 38 个追踪键不生效，就是没按索引填。
**填法**：逐索引 `Enum.ToObject(keyType,i).ToString()` 与 mesh 键名比对——精确同名优先 → 别名表 → 仍无则留空字符串。本例 **exact 39 + alias 26 = 65/103**，viseme 15/15。

**别名表**（本模型实测）：
```text
BrowLowerer        → BrowDown
LipFunnelUpper/Lower → LipFunnel
LipPucker*         → LipPucker
MouthLowerDown*    → MouthLowerDown
MouthCornerPull/Dimple/Slant → MouthSmile*
NoseSneer*         → NoseSneer
CheekSuck*         → CheekSuck
MouthUpperDeepen*  → MouthUpperUp*
```

### 3.2 拆分 Set：长度不变 + 空名占位 + 乘数 0

拆分实现是**长度不变 + 非保留索引置空名 + 乘数 0**；空名不是有效 blendshape，不参与写入（第 172 轮）。
- 读回 4 集均 n=103，但只有对应区间有真实名+乘数 1；空名槽乘数=0。
- **判断拆分是否有效看非空名与乘数，不能看数组 Length**；空名占位是合法做法。工具：`Assets/Editor/PLAN002FTNativeBuild.cs CloneSetBlank`。

### 3.3 Auto-Fill 补不出自定义键

`FaceTrackingAutoFill.cs:56-64` 的别名表**不含 `eye_look_*`**；按「精确同名优先」自动填充只填 ARKit 名。凡自定义键（如 `eye_look_L/R`、`FT_eye_look_*`）**必须手动登记**（第 164 轮）。

## 4. Gaze 通道语义

官方口径（第 164 轮核对 <https://creators.vrchat.com/avatars/face-tracking/osc>）：
- gaze 是 **8 个单极性 0..1 方向通道**（`/tracking/face/v1` 102 字节 blob，索引 0–7 = 右眼 Out/In/Up/Down + 左眼 Out/In/Up/Down），**不支持 signed**。
- **「several Tracking Keys can share the same blendshape」且「highest value wins，不叠加」** → 4 个水平键两两映射到 `eye_look_L/R` 合法。
- multiplier 源码 `Mathf.Clamp(value,0f,10f)`（`FaceTrackingBlendshapesSetDrawer.cs:532`）——**不能为负**；反向只能靠「映射到不同键」，不能靠负 multiplier。
- 映射表必须手动改；改完做一次方向 A/B：**`eye_look_L` 实测使瞳孔向画面右偏，键名与实际方向相反**（以实测为准）。

## 5. EyeGazeMode 三态与 SDK 预览 bug

- `EyeGazeMode=Both` 以为骨骼旋转能补偿水平位移——**补不了**：骨骼 `Eye.L/Eye.R` 存在且各有 548 顶点全权重，但 `eyesLookingLeft/Right` 预设只差 **±2°**（Euler 178°↔182°），`Up` 与 `Straight` 几乎相同 → 4 态截图无可辨差异（第 164 轮）。要么重校准骨骼预设，要么改 `FaceTracking`（仅形变键）；别指望 Both 自动兜底。官方源码 `VRCAvatarDescriptorEditor3EyeLook.cs:132-142` 逐字说明三态。
- **`FaceTracking` 模式有 SDK Preview-only bug**（官方 issue tracked）：Unity 内预览只用骨旋转；`Both` 下 FT blendshapes 正常。反馈看板：<https://feedback.vrchat.com/open-beta/p/eye-gaze-mode-set-to-face-tracking-still-uses-bone-rotations-only-in-the-sdk-pre>。
- **不赌写入顺序，保持 `Both`**：官方文档只说 "VRChat handles tracking and blendshape updates outside the animator"，**没有** before/after、priority、override 规则（第 165 轮逐段核对）。用零顺序依赖方案（§7 新建专用键）绕过；别改成 `FaceTracking`，否则 Unity 内验证会被预览 bug 污染。

## 6. 几何幅度判别（瞳孔质心法）

**症状**：原生 FT 已启用、参数齐全、上游数据在流，但**瞳孔只不随视线左右移动**；上下、眨眼、嘴追全部正常（第 164 轮；第 160 轮 OSC 架构同症）。
**根因**：**终点几何幅度不对称**，不是数据问题：
- ARKit 水平键 `EyeLookIn*` 近乎空键、`EyeLookOut*` 主要沿 **Z 轴前后推眼球**（ΔZ 绝对值 0.00109 > ΔX 0.00074）；
- 垂直键是干净的 ΔY≈0.0041（4–6 倍）；眨眼 maxDisp=0.0255、JawOpen=0.013（一个数量级）；
- 模型原生 `eye_look_L/R` avgΔX=∓0.0041~0.0048（比 ARKit 水平键大 12 倍）。

**判别配方（单键 @100 + 黑色瞳孔质心）**：12 键面部特写，量瞳孔质心位移：

| 键 | Δcx / Δcy |
|---|---|
| `EyeLookInLeft/Right` | 0.32 / 0.28 px（几乎不动） |
| `EyeLookOut*` | 5.0 / 5.12 px |
| `EyeLookUp*` | Δcy = −18 px |
| `EyeLookDown*` | Δcy = +42 px |
| 原生 `eye_look_L/R` | **Δcx = +36.68 / −41.53 px** |

**规则**：不要再去查参数/控制器/AAO；先跑矩阵把「哪一环失效」定量到键本身。**不换模型**——症结是「ARKit 水平键几何幅度 + 映射终点」，换模型会重演（第 160/164 轮两次同症均以 `eye_look_*` 为解）。
**OSC 架构下的对应修法**：眼动 clip 追加模型原生键（新建 Neutral/In/Out/Up/Down 5 clip 各写 `eye_look_L/R/UP/DW` 4 条，对应方向 100 其余 0）；**只在左眼子树写原生键**，右眼子树保留 ARKit，避免两眼权重叠加到 200（第 160 轮，`WOLF_Eye_Look_*.anim`）。

## 7. Animator 覆盖与 FT 专用键（零顺序依赖）

### 7.1 第二条独立根因：常驻默认态层每帧写 0

**症状**：映射表把 gaze 水平键改到 `eye_look_L/R` 后，Unity Play 里手动设 100 会被刷回 0；而「上下、眨眼、嘴正常」（第 165 轮）。
**根因**：`[VF53] Defaults`（VRCFury FixWriteDefaults，1307 曲线）与 `FaceDefault`（w=1 Override，`LilLeo_face_zeroshape`，193 曲线）两个常驻层每帧写 `eye_look_L/R=0`；但这两层默认态 clip 里 `EyeClosedLeft/Right`、`JawOpen`、`EyeLookUp/DownLeft` **全部 absent** → 对照组不受影响。
**定位法**：Play 中挂最终运行时 `VRCFury FX`（127 层），设 `eye_look_L=100` → 数帧后读回 0；立即 set 77 能读到 → setter 正常，是 Animator 覆盖。**扫「常驻默认态层写了哪些 blendshape」，用「用户说正常的通道」做对照组**，一次定位覆盖源。

### 7.2 新建 FT 专用键 = 唯一零顺序依赖解法

Animator 曲线按属性名绑定，新名字不在任何既有 clip 里 → 无论 FT 先写还是后写都不被覆盖（第 165 轮）：
1. `Instantiate(Body.sharedMesh)` → `AddBlendShapeFrame` 复制 `eye_look_L/R` 的 `GetBlendShapeFrameVertices` delta → 存 `.asset` → 替换 `sharedMesh` → FT 映射指向新名（`FT_eye_look_L/R`）。
2. **从本模型自身 FBX 复制顶点增量**（不从无关模型复制）；原 347 键与顺序保持不变。
3. 校验：`maxDeltaDiff=0.000E+000`；场景/烘焙 FX 里 `FT_eye_look*` 曲线数 **0**；骨骼 125 / 材质 3 / 子网格 3 / 权重漂移 0。
4. 产物：`Assets/08_动画·菜单·图标/FT_v2/Mesh/Body_FTGaze.asset`（349 键，idx347/348）。

### 7.3 AAO 之后必须读回 + bones 骤减先打印名字

- **凡新增形变键，必须在 AAO 之后的烘焙产物里读回一次**（只验证场景不够）：`AAOGeneratedMeshBody shapes=349`、`FT_eye_look_L/R` idx347/348、`AAO_Merged=0`、delta −0.004136/+0.004842 未变；`optimizeBlendShape=False`、`mergeSkinnedMesh=False`、`removeUnusedObjects=False`（第 165 轮）。
- 烘焙产物 Body 的 `bones` 从 125 变 3 **不是眼骨丢失**：AAO 移除该 SMR 未使用的骨骼，保留的正好是 `Head`/`Eye.L`/`Eye.R`；材质 `make&skin`/`M_LilLeo_Body_Eye`/`M_LilLeo_Hair_PW` 齐全，十视角渲染无破面。**看到 bones 骤减先打印名字再判断**。
- AAO Inspector 弹「检测到未知的组件类型 `VF.Model.VRCFury` / `VF.Component.VRCFuryHapticSocket`」是共存提示：`Editor.log` 里是 `[NDMF] Error Reported:` 通道标签，正文明确「（不会影响原始 Avatar）」；不是 `error CS`、非构建失败。可在 `Tools/Avatar Optimizer/Optimization Metrics` 关闭该日志。判据看烘焙产物读回 + API 版本（本例 v100、349 键、103 项映射完整）。

## 8. 眯眼高光漂移（hightlight_* 共用顶点）

> **2026-11-12 补充/取代边界**：本节保留旧方案的历史依据，但“129点全部归零”会在眯眼时令前移的虹膜遮住静止高光。新实施不得照搬该步骤，见 §15 的88个真实高光点与4个误伤虹膜点分类修补；旧§8未完成动态验证的记录仍保持原状态。

**症状**：只在开启面捕时出现；眯眼/眨眼时瞳孔里的高光跟着眼睑走，和 gaze 不同步，能溢出眼球轮廓（第 176 轮，本工作区实测 2026-11-11）。
**根因（定量判据，不是猜材质）**：LilLeo 狼头 Body_FTGaze 网格的「眼睛高光」不是独立 mesh、不是 UV 偏移、也不是 MatCap/Emission 材质动画，而是 **9 个作者笔误拼写的 `hightlight_*` blendshape**（多一个 t：`hightlight_A_up/down/small/big/circle`、`hightlight_B_up/down/small/big`，idx 79–87）。这 9 个键的 **129 个顶点（含法线/切线 delta 并集）与瞳孔/眼睑共用同一批顶点**。原生 FT 的 103 项 Set 不直接写 hightlight_*，只写 `EyeClosedRight/Left`(idx 8/9) 与 `EyeSquintRight/Left`(14/15)，于是这些键顺带把共用顶点推走。

**实测数据**（`Mesh.GetBlendShapeFrameVertices`，高光顶点集 129）：
- EyeSquint 100/100 高光顶点被移动，avg=(0,+0.005811,+0.005507)；EyeClosed 100/100，avg=(0,−0.005811,−0.005507) —— **两键对高光的作用完全反向**；
- eye_blink 88/100（不对称），eye_sleep 0/100，EyeSquint+EyeClosed(275) 0/100；
- FT_eye_look_L/R、eye_look_L/R、EyeLookUp/Down 100/100 跟随（gaze 与高光同步，问题只出在眼睑/眯眼通道）；
- 眼材质 slot1 `M_LilLeo_Body_Eye` 有 `_UseMatCap:1`，但 MatCap 是法线/相机投影，不能越过网格轮廓，**只是放大因素，不是根因**。

**修法（第 165 轮「新建 FT 专用键」同构方案，已实施并读回）**：在 Body_FTGaze.asset（小狼专属、唯一消费者）上**追加 4 个键**：`FT_EyeClosed_R/L`、`FT_EyeSquint_R/L`，各复制 `EyeClosedRight/Left`(273/274)、`EyeSquintRight/Left`(277/278) 的 frame0 delta，**但对 129 个高光顶点的 dv/dn/dt 全部归零**；再把 `faceTrackingBlendshapesSetList[0].faceTrackingBlendshapeNames` 的 [8]/[9]/[14]/[15] 改指这 4 个新键。眼睑/眼球形变完整保留（nonHiMoved=876~980），高光不再被眼睑键带动（hiMoved=0/129），gaze 键仍带高光正常跟随。同时清掉静态残留 `EyeLookUpLeft=100`、`eye_sleep=10`（§9）。

**判据**：
- 源网格 349→353 键、verts=5171 / subMesh=3 / bones=125、原 349 键名与顺序不变、GUID 不变；
- 4 新键 hiMoved=0/129 且 nonHiMoved>0；
- **AAO 之后**烘焙产物仍是 `AAOGeneratedMeshBody bs=353`、4 新键 idx 349–352 与 `hightlight_A_up` idx 79 全部保留、无 `AAO_Merged_*`（optimizeBlendShape=0）；
- 十视角 + GM 逐键（EyeClosedLeft=100 / EyeSquintLeft=100 / 组合键）视觉复核为**必需**，静态/构建证据不能替代（第 176 轮因 Play 卡住标 NOT_RUN）。

**不要照抄**：`iPupilX`（本工程 349 键里 0 命中，属其它素体）；[00-07 troubleshooting](../00-基底/00-07-troubleshooting.md) 的「高光色斑」是方向光>0.8 的渲染伪影，不同症状；lilToon MatCap 屏蔽只在材质 A/B 判定成立时才做。

## 9. 静态残留与表情层覆盖

- 场景里 `EyeLookDownLeft/Right` 静态权重 100，默认态瞳孔偏下且没被动画层重置：Blendshape 默认权重是场景序列化值；若 FX 层没有任何 clip 写 `EyeLook*`，运行时不会覆盖它（第 164 轮；实测 FX 全部 clip 的 `blendShape.EyeLook*` 绑定=0 条，单键 @100 Δcy=+41.58/+42.43 px）。**出包前清掉这类静态残留**。
- `eye_look_*` 反向有 **86 条**曲线（手势表情层 + `GestureMute_*`），原生 FT 输出会被表情 clip 覆盖，需门控（`FaceDefault / Left Hand / Right Hand / WolfFace TangSmile` 层；门控做法见 [04-08 §3.7](04-08-face-tracking-osc-and-repair.md)）。

## 10. OSC → 原生迁移的回归与防复发

- **迁移必删旧 FT 动画层**：两套系统都写同一批 blendshape 会「打架/抖动」。删掉 `FT_Merge` 上的 `ModularAvatarMergeAnimator`（**保留 MenuItem/Installer/Parameters，菜单不丢**）。判据：烘焙后 `legacy FT layers remaining=0`（第 161 轮）。
- **迁移回归案例**：旧 OSC 架构用 `WOLF_Eye_Look_*.anim` 在 clip 里追加 `eye_look_*` 曲线；迁到原生 FT 删 MergeAnimator 后那条链路随之消失；新 Set 按「精确同名优先」只填 ARKit 名，`eye_look_*` 命中数=0 → 升级 SDK 后水平眼动「又坏了」（第 164 轮）。
- **铁律：凡是从 OSC/VRCFury 面捕迁到原生 FT，必须把旧 clip 里用过的自定义键名重新登记进新映射表**，否则静默丢失。查 `faceTrackingBlendshapeNames` 中以 `eye_look` 开头的数量即可自检。

## 11. 消费者输入边界（GM 不能伪造面捕）

**症状**：用 GM 驱动动画参数 `FT/v2/EyeSquintLeft=1`，Body 上 EyeSquint/EyeClosed/FT_EyeSquint_L/R 权重仍全为 0，误以为「眯眼坏了」（第 172 轮，2026-11-11）。
**根因**：描述符 BlendshapesSet 的映射输入是 **FaceTrackingKey（硬件面捕键）**，不是动画参数；`FT/v2/*` 是另一套 ForwardToAnimator 输出，二者不通。读回描述符：`FT_EyeSquint_R/L` 在 PLAN002/EyeBrow 集(weight=1, 索引14/15)与 Body 集(weight=0)；GM 无面捕输入 → 恒 0。
**规则**：视觉面捕（眯眼/眼追/嘴型）只能用**真实 OSC/VRCFT 输入**或 VRChat 客户端验证，不能用动画参数伪造通过；**GM 只验参数/菜单/门控链路**。证据：`Assets/Editor/PLAN002Gm.cs`、`.plan-mode/evidence/plan002_gm.txt`。

## 12. 面捕菜单

### 12.1 原生 FT 下旧菜单 Toggle 的降级

原生 FT 由客户端直接驱动 blendshape，avatar 菜单参数不参与；`VisemesEnable/EyeTrackingActive` 等只写参数无消费者（删 MergeAnimator 后烘焙 FX 内这些参数消费者=0，第 161 轮）。两种选择：
1. 接受降级（菜单保留但无效，**交付说明里写清**）；
2. 用 `faceTrackingParameterBehaviors` 把追踪键转发到 animator 参数再做门控（**未实测**）。
**不要**再让旧 FT 层回来。

### 12.2 面捕菜单重定向四步（避免 FT Controls 自递归）

**症状**：`Face Tracking by Hash`（FT Off/FT On/FT Controls）挂在根菜单（第 177 轮）。
**根因**：`PLAN002_FT` 的 MenuInstaller `installTargetMenu=根`；而「功能>面捕」子菜单资产＝`Face Tracking Control - UE Blendshapes LilLeo.asset`，若直接把 SimpleFTControls 装进它会产生 FT Controls→自身递归。
**做法**：新建 `沐梓溪狼_菜单_功能_面捕.asset`；`FT_Merge` MenuItem `Control.subMenu` 与 `PLAN002_FT` `installTargetMenu` **同指新资产**。
**烘焙读回判据**：根>功能>面捕>{FT Off, FT On, FT Controls>{Visemes Enabled/Eye Tracking/Lip Tracking/Facial Expression Disabled/FTEffects}}，根无 FT 项。

## 13. effects 动画树与主链分工

直接合并作者整条 FT 控制器会与描述符原生 set **双驱动眼/口主链**（第 172 轮）：作者 `FTOnly_Wolf` 的 52 子 Direct 树把眼口主链与 effects 耳尾混在一起，整份合入必重复写。
**做法**：只取根 Direct 的 **child43-46（effects）**做动画树，眼口主链交给描述符。
**边界**：Smile/Angry 在本模型无对应 blendShape 属空操作，**不得宣称生效**。

## 14. 来源

- 本工作区实测 2026-11-06 至 11-11（第 143/161/164/165/172/176/177 轮），原载 `08-症状速查` 03/04/08 增补册，2026-11-11 结构化并入本册。
- VRChat 官方：<https://creators.vrchat.com/avatars/face-tracking/>（Blendshapes Set 精确同名驱动、多 Tracking Key 共享 blendshape、Multiplier、Default Weight、可建只含眼睑映射的 Set）、<https://creators.vrchat.com/avatars/face-tracking/osc>（gaze 8 通道）。
- SDK 源码：`VRCAvatarDescriptorEditor3EyeLook.cs:132-142`、`FaceTrackingBlendshapesSetDrawer.cs:532`、`FaceTrackingAutoFill.cs:56-64`。
- 关键脚本（工程内）：`PLAN002FTNativeBuild.cs`、`PLAN002Gm.cs`、`Body_FTGaze.asset`；版本快照 Unity 2022.3.22f1c1 / SDK 3.10.6 / MA 1.18.7 / NDMF 1.14.8 / AAO 1.9.21 / VRCFury 1.1442.0 / lilToon 2.4.1。

## 15. 高光归零补丁引起的虹膜遮挡与按眼补偿

**本工作区实测 2026-11-12，PLAN-001/T015。适用范围**：LilLeo狼头、5171顶点/353形态键的Body_FTGaze、Unity2022.3.22f1c1、SDK3.10.6、MA1.18.7、NDMF1.14.8、AAO1.9.21、lilToon2.4.1；已有§8四个FT专用眼睑键。下列索引是本模型指纹，不可直接套到其他网格。

### 15.1 症状、复现与根因

用户客户端截图显示面捕时白色上下高光消失、虹膜/瞳孔仍在。Unity相机把视线输入与相机角度分开后，`FT_EyeSquint_R/L=60`稳定复现；中性态正常。只把临时眼材质Cull改Off，高光仍消失，排除仅背面剔除。保持原材质、让高光随虹膜移动后，同状态高光恢复。

原因是§8旧补丁把高光位置delta归零，但虹膜仍在移动。虹膜139顶点的主位移在Squint100时Z约+0.0047955，60%时+0.0028773；高光固定，落到虹膜之后。Closed的Z方向相反，不能统一加正Z。虹膜近刚性平移，顶点位移相对逐分量中位向量最大偏差约0.000102，**不是严格零差刚性**。

### 15.2 顶点集合必须分类

| 集合 | 本模型数量 | 处理 |
|---|---|---|
| 8个A/B高光形态键（排除circle）的真实高光岛 | 88，左右各44（A23+B21） | 四个FT眼睑键内，按眼、按键用该眼虹膜岛的中位平移，保持原高光相对虹膜位置 |
| circle额外集合中被旧补丁误伤的左虹膜点 | 4：4859、4861、4862、4863 | 恢复对应legacy EyeClosedLeft/EyeSquintLeft的原dv/dn/dt |
| circle其余额外点 | 37 | 保留旧FT数据，不套高光平移 |

九个`hightlight_*`键的**稀疏记录/位置法线切线并集是129**，不是129个独立高光几何点。按位置delta阈值统计可能得到100，且含瞳孔点，不能当白名单。应以A/B八键与实际三角形连通岛交叉核对，88点成立后再处理4个误伤点；不得把129整体加Z。

### 15.3 实施与验证

从源mesh克隆专用`Assets/_Gen/WolfR520/Mesh/Body_FTHighlightFollow.asset`，保持353键名、顺序、帧权重，只替换小狼Body引用。源mesh和源材质不改。四个FT键：右侧各44点差异、左侧各48点差异；其余349键逐帧**位置**一致，基准顶点/UV0–7/拓扑/蒙皮/bindposes一致。落盘与.vrca读回发现19个非目标键的≤2e-6法线/切线微值被Unity序列化归零，最大误差≤2e-6，不能宣称全通道字节完全不变；须用`LoadAssetAtPath`和实际bundle复核并记录容差。4个circle点是相对旧FT零值的有意行为变更，采用legacy数据，不是恢复旧FT原值；legacy与FT同时驱动的叠加仍需客户端观察。

已执行证据：`docs/PLAN-001-r520/T015-mesh-audit.txt`逐键逐帧差异；`T015-baked-readback.txt`读取实际NDMF产物的353键及四个Set映射；`eye-shots/final_source_*` 15种输入×7种相机角度（105图）。相机样本已判读：中性、上下左右及斜向注视、眯眼25/60/100、注视叠加眯眼、睁大、闭眼50/100；完全闭眼时高光被眼睑遮挡，没有穿透。眼部矩阵是**网格变形视觉实验**，不能冒充原生追踪输入运行。

GM只验真实轮盘菜单事件、参数和门控；原生FaceTrackingKey的消费者在客户端，当前GM不实现该输入，遵守§11。客户端最终效果仍须实机面捕与相机角度往返取证，不能从上述实验外推“客户端已修复”。

**组合边界（2026-11-12相机实测）**：把legacy `EyeSquintRight/Left`与`FT_EyeSquint_R/L`同时设25/60/100并拍yaw0/±35，100%斜角可见高光浮出眼睑；正常只驱动FT专用键的矩阵未见该问题。4个circle点采用legacy数据是有意行为变更，不是恢复旧FT零值。该组合不是本次已验收路径，客户端若出现手势表情与面捕叠加需另行门控/复测，不得把正常FT矩阵PASS外推到组合。

### 15.4 来源与裁决

- KB§8旧归零方案与§11消费者边界；本次独立YAML稀疏delta解析与Unity Mesh API读回交叉核对。
- [VRChat Face Tracking](https://creators.vrchat.com/avatars/face-tracking/)：原生映射在Animator之外更新、多个Tracking Key可共享shape且最高值优先；横向双槽映射未因无证推断而改写。
- [Unity2022.3 BakeMesh](https://docs.unity3d.com/2022.3/Documentation/ScriptReference/SkinnedMeshRenderer.BakeMesh.html)：本轮`unity_docs`按2022.3成功读回实际蒙皮快照与useScale语义；Anysearch网页extract该页失败，属抓取失败而非无资料，未用6000.5页面替代当前版本依据。本次使用false快照后按Renderer.Transform转世界坐标。
- Anysearch通用故障搜索未找到该模型的直接案例，材质搜索返回无关条目；实施依据为官方机制、当前资产和可复现实测，不能写“联网查到了相同修复案例”。
