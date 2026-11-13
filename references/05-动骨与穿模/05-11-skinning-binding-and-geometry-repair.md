# 蒙皮绑定与几何修复（bindposes / 权重 / 法线 / 量测）

> **来源**：原 `08-症状速查` 第 141/146/147/167/168/171/172/177 轮实测条目（2026-11-06 至 11-11）与主表早期行结构化整理。
> **分工**：跨头像衣装等比 bindpose 重绑 / Kabsch / NewFit 见 [05-08](05-08-cross-avatar-garment-fit-newfit.md)；袜子膝穿模顶点缩放见 [05-01 §12–§14](05-01-outfit-fit-and-clipping.md)；道具尺度反推 / 活动约束量测 / pose 无关握持判据见 [05-10 §22–§23](05-10-ma-bone-and-geometry-components.md)；万圣节 bindpose 塌缩的 MergeArmature 修复见 [05-10 §21](05-10-ma-bone-and-geometry-components.md)。

## 目录

1. 重绑骨骼后必须重算 bindPoses（+ localBounds 归零）
2. SMR 自身 Transform 对蒙皮无效（总纲）
3. 跨空间重绑：顶点与 bindposes 必须同空间
4. 手套指部权重（平滑混权 / 三段重分配 / Hand_OFF）
5. 身体 OFF 系形态键只用于被完全遮住的部位（Knee_OFF）
6. 兜帽与发尖：尺寸无解，只能抬升
7. 口袋陷入衣面的 Play 内 BakeMesh 测量
8. invalid normals 双根因（Blender 4 法线缺陷 + SDK legacy 修复链）
9. Humanoid → Generic 舞蹈转换三连坑
10. 批量换 sharedMesh：按网格名取子资产
11. 眼镜 / 名牌跨头像放置

---

## 1. 重绑骨骼后必须重算 bindPoses（+ localBounds 归零）

**症状**：万圣节卫衣（おばけパーカー）从 unitypackage 导入后实例化到 LilLeo 骨架，SMR 骨骼按名重绑成功，但衣服**完全不可见**：Scene/Game 视图、RenderTexture、Standard shader 均不出图；`localBounds` 全零、`isVisible=False`（第 168 轮）。
**两个叠加根因**：
1. prefab 实例化后 SMR 的 `localBounds` 被重置为 `Extents=(0,0,0)`，网格被视锥体剔除；
2. **重绑骨骼后 `mesh.bindPoses` 仍是原 prefab `armature.1` 骨骼的逆矩阵**，与 LilLeo 骨骼 Transform 不匹配 → 蒙皮数学把所有顶点算到错误位置/退化为点。
**判据**：逐 SMR 读回 `localBounds`/`mesh.bindposes`/`boneWeights`：权重全非零、bindposes 非零矩阵、bounds 非零但 `isVisible=False`；换 Standard shader 仍不可见（排除 shader）；重算 bindposes 后 `isVisible=True` 立刻出图。
**修法（公式）**：
```csharp
// 对每根骨：
bindPoses[i] = bones[i].worldToLocalMatrix * smr.transform.localToWorldMatrix;
mesh.RecalculateBounds();
smr.localBounds = mesh.bounds;
smr.updateWhenOffscreen = true;
```
判据 = Play 模式 `smr.isVisible=True` + RenderTexture 截图出图。文件 `Assets/mmiao/おばけパーカー/Prefab/おばけパーカー_orange.prefab`；场景 `万圣节卫衣` 子树 4 个 SMR。
**相关**：第 172 轮一键穿衣后全身裸体同因（MA 重定向后 bindpose 与骨架空间不一致）——从工程内原 FBX 重建 R400 网格，`RootBone`/`ProbeAnchor` 指 `armature/Hips`，`MergeArmature.LockMode=BaseToMerge`，按同一公式重算（`R400HallRebind.cs`，[05-10 §21](05-10-ma-bone-and-geometry-components.md)）。

## 2. SMR 自身 Transform 对蒙皮无效（总纲）

SMR 顶点 = `bones × bindposes × 骨骼矩阵`，**与 renderer 自身 transform 无关**（主表早期行 + 第 137/146 轮反复实证）：
- 改 `SkinnedMeshRenderer.localPosition` / 父容器 +Y，网格一动不动（世界 AABB 位移全 0）；1.03 的 Transform 缩放对屈膝特写像素差仅 0.06%。
- 要平移/缩放蒙皮配件只能改**骨骼**（在宿主骨下插偏移骨，把 `bones[0]` 指过去）或改**网格顶点**（[05-04 §8.3–§8.4](05-04-cross-avatar-accessory-transfer.md)）。
- 蒙皮道具（RK Player 等）摆位同理：重绑骨骼或用插件自带动画，不要只挪父节点（[02-11 §1.8](../02-工程组织与工作流/02-11-plugin-integration-rk-realkiss-props.md)）。
- 静态 MeshRenderer 渲染的是 **bind pose**，不是当前骨架的蒙皮结果；预览要用同骨骼临时 SMR + `BakeMesh`（[05-08 §21](05-08-cross-avatar-garment-fit-newfit.md)）。

## 3. 跨空间重绑：顶点与 bindposes 必须同空间

**症状**：想给袜子「重绑权重」修穿模，结果网格被拉到脚下（第 146 轮）。
**根因**：袜子网格与身体不在同一网格空间，直接把 bindposes 换成身体的会让顶点飞到原点附近（换后 `BakeMesh` 世界 y 范围从 0.02~0.25 变成 -0.06~0.05）。
**规则**：同一绑定的前提是**网格顶点也换成目标空间**（`body.worldToLocal × sock.localToWorld`）；否则只换权重、保留原 bindposes，并按 bindpose 反查可用骨骼。文件 `睡衣_Yurumoko/YURU_sox`。

## 4. 手套指部权重（手指从指瓣穿出）

**症状**：小狼手套开着手势比耶/指向，手指从手套指瓣之间穿出（第 141/167 轮）。
**根因**：手套 5326 顶点里**没有一个顶点主导于手指骨**（IndexIntermediate/IndexDistal 主导数=0），手指区 360 顶点全绑 `Hand.L/R` → 指瓣是刚性的，手指照常张合。第 167 轮复核：指部 3652 顶点主导骨全是 `…Proxima` 或 `Hand`（`MiddleProxima.L=393`、`IndexInterProxima.L=396`、`Hand.L=373`…）。
**修法 A（平滑混权，第 141 轮）**：手指区顶点按「到各指骨段距离」混权：`w_finger = clamp01((30mm-d)/30mm) * 0.85`，其余留 `Hand`（**硬切会撕裂指瓣**）。产物 `Assets/ZZZ_GeneratedAssets/Mesh/Gloves_fingerw.asset`。
**修法 B（三段重分配，第 167 轮，只改 boneWeights、不动顶点/形态键/Transform）**：
1. 按手套自身骨骼名建 10 条手指链（Proximal→Intermediate→Distal，如 `Index.L -> 34,35,36`）；
2. 取 Proximal 与 Distal 的世界位置算 `t = Dot(worldPos-P0, P2-P0)/|P2-P0|²`；
3. `t*2` 在 [0,1] 给 P/I、[1,2] 给 I/D 线性混合；`Hand.*` 与 `LowerArm.*` 主导顶点保持不变；
4. 存新网格资产替换 `SMR.sharedMesh`（`Gloves_FingerFit.asset`）。
**判据**：修复后出现 `IndexIntermediate.L=121`、`IndexDistal.L=171`、`ThumbDistal.R=177` 等主导计数；烘焙后 Intermediate/Distal 主导顶点数 >0 + 指向手势手部特写无穿出。
**配套（第 172 轮）**：手套只是父级开关时身体手部仍显示——在手套菜单对象加 MA `ShapeChanger`：`Hand_OFF`、`Hand_OFF_left`、`Hand_OFF_right` 都 **Set 100**（threshold 0.01）；烘焙后应读到 3 条曲线（修复前 0 条）。产物 `Gloves_RigidMitten.asset`。ShapeChanger 的 Delete/Set 语义见 [04-06 §3/§14](../04-外观与表情/04-06-ma-reactive-components.md)。
**判别坑（第 132 轮）**：手套「网格被改坏」但顶点看起来没问题——顶点/三角形/法线/bindposes 全等，差异在 `boneWeights` 的 boneIndex/weight 被重映射（本工程 3586/5326）。**直接复制原版 mesh 换 `sharedMesh`，不要重算权重**；判据=`mesh.boneWeights` 与原版逐项比 boneIndex0 差=0。

## 5. 身体 OFF 系形态键只用于被完全遮住的部位

**症状**：想用「腿部形态键最低」修袜子膝穿模，结果腿废了（第 141 轮）。
**根因**：`Body_Base` 的 `Knee_OFF` 是**隐藏膝部体积**的形变（260 顶点、最大 4.1cm），100% 会把大腿压成平面；且袜子是独立 0 形态键网格，**身体形态键不带动袜子**。
**判据**：`Knee_OFF=100` 渲染大腿成片状；改前改后袜子顶点不动。
**规则**：袜子类穿模只走「改袜子网格」（[05-01 §12–§14](05-01-outfit-fit-and-clipping.md)）；身体 OFF 系形态键仅用于被衣服完全遮住的部位。

## 6. 兜帽与发尖：尺寸无解，只能抬升

**症状**：兜帽永远包不住头，以为放大骨骼就能解决（第 147 轮）。
**根因**：`Head` 骨骼位于兜帽顶点**之上**（Head 0.6566、兜帽顶 0.5853），以 Head 为中心缩放会让兜帽**向下**移动；真正差距是「发尖 0.829 vs 兜帽 0.5853」（`Cat_Ear` 0.883）。
**规则**：兽耳/发尖与兜帽尺寸**无解**（任何尺寸都会露出）；只能按需抬升兜帽顶部（幂函数 `t^1.6 * lift`，底部不动，避免线性拉伸变尖顶）。产物 `LH_Tops_hood_060_046_018.asset`。

## 7. 口袋陷入衣面的 Play 内 BakeMesh 测量

**症状**：休闲装口袋永远不见（第 177 轮）。
**根因**：`YURU_pokke` 蒙皮实际位置陷入上衣表面后 9.5mm。**编辑态 `SMR.bounds` 不可信**（Play 内曾给出 2.8m 荒谬值）。
**测量**：Play 内 `BakeMesh` 实测口袋 max z=0.0675 vs 衣面窗口 z=0.0770。
**修法**：`localPosition.z += 0.0176` 使袋面露出衣面 8mm。
**判据**：GM Play 默认态口袋 activeSelf=True 且特写可见袋面+袋口线；上衣 OFF 两者同隐；往返恢复；on/off 截图差异仅口袋。
**通用**：活动约束实例上量几何是垃圾数据，量测两条路（prefab rest pose / Play 烘焙克隆）见 [05-10 §23](05-10-ma-bone-and-geometry-components.md)。

## 8. invalid normals 双根因（Blender 4 + SDK legacy 修复链）

**症状**：Console 反复 `[ImportFBX Errors] The mesh Base_B.004..010 has invalid normals`，进 Play 后消失（第 177 轮，2026-11-11）。
**两条独立链**：
1. `Base_B.*` 网格在 `Assets/03_头发/头发4/Side-Half-Up/Prefab/For */支援版/*.fbx`（16 个，**Blender 4.x 导出法线缺陷**，联网证实 blender#123088），`importNormals=Import` 时每次重导都报；
2. SDK 构建面板 `CheckAvatarMeshesForLegacyBlendShapesSetting`（`Packages/com.vrchat.avatars/Editor/VRCSDK/SDK3A/VRCSdkControlPanelAvatarBuilder.cs:4636-4740`）对 `importBlendShapeNormals=Calculate && legacy=false` 的网格每次构建提示修复 → `EnableLegacyBlendShapeNormals` → `SaveAndReimport`，FBX 无平滑组时再报一次 invalid normals（Unity 自动 `Recalculating normals`，**无运行时缺陷**）。
**做法**：未使用的头发 FBX 直接 `importNormals=Calculate`（先确认目标头像 SMR 未引用）；LilLeo 系 FBX 预先 `legacyComputeAllNormalsFromSmoothingGroupsWhenMeshHasBlendShapes=true`（internal 属性，反射写入）一次性重导——SDK 扫描跳过 legacy=true，复发链断开。
**判据**：重导后 `read_console` 无 invalid normals；脸/身特写 shading 无回归（本轮 22_face_final.png 目视 PASS）。若视觉回归则回退 legacy，按用户授权跳过。

## 9. Humanoid → Generic 舞蹈转换三连坑

**症状**：想把小狗的 Humanoid 舞蹈搬到 Generic 的小狼上，`AnimationMode.SampleAnimationClip` 采样后每个时刻姿态完全相同（第 171 轮）。
**三连坑**：
1. `AvatarBuilder.BuildHumanAvatar(go, desc)` 需要 `desc.skeleton=SkeletonBone[]`(name/position/rotation/scale) 且 `desc.human=HumanBone[]`，而**该版本 `HumanBone` 没有 `boneIndex`**（只有 `boneName`/`humanName`/`limit`），设 `boneIndex` 无效；
2. `new AnimationCurve[3]` 的**元素是 null**，`curves[0].AddKey()` 直接 NRE；
3. 即便 Avatar 建好（21 骨），Unity 也**无法在临时 Avatar 上对外部导入的 Humanoid clip 求值**（clip 的 `humanMotion.skeleton` 与 Avatar 骨架的按索引对应关系不成立）→ 输出恒为默认姿态。
**判据**：烘焙日志 `Humanoid Avatar built, bones=21` 但采样 4 个时刻 `Hand.L` 欧拉角完全一致。
**可行替代**：装 Animancer / Animation Rigging 做 Humanoid→Generic 转换，或直接取已转成 Generic 的舞蹈素材。本次只交付框架（Contact 同步 + 参数 + 菜单 + FX 层），舞蹈动作本体标 NOT_RUN。工具 `Assets/Editor/SyncDancesBuilder.cs`。
**GM 侧**：GM 不自动启用 Action 层，舞蹈类验证的降级与 AnimationMode 裁决法见 [07-06 §23](../07-验证与排查/07-06-gesture-manager-automated-sim-test.md)。

## 10. 批量换 sharedMesh：按网格名取子资产

**症状**：批量换 `sharedMesh` 后所有件都变成同一件（9 件睡衣全成帽子，主表早期行）。
**根因**：同一个 FBX 的多个网格共用一条资产路径，`LoadAssetAtPath<Mesh>(fbx)` 只返回第一个子资产。
**做法**：`LoadAllAssetsAtPath(path).OfType<Mesh>().First(m => m.name == name)`；base 记录写成 `资产路径|网格名`；已写坏时 `EditorSceneManager.OpenScene(..., Single)` 重载场景丢弃未保存的坏赋值。

## 11. 眼镜 / 名牌跨头像放置

跨头像克隆后眼镜不贴脸、名牌文字/颜色不对（主表早期行）：眼镜/名牌是自带 `Armature` 的子预制件，父链与骨骼缩放跟原角色不同。
- **眼镜**：照抄原角色同一骨骼下的 `localPosition/localRotation/localScale`。
- **名牌**：把 `平面` mesh 换成透明底贴图 + `Unlit/Transparent` 材质（lilToon `Hidden/lilToonOutline` 是 opaque，透明字会变黑块）；`localScale` 负 X 可镜像文字。自建名牌的完整做法见 [05-08 §17](05-08-cross-avatar-garment-fit-newfit.md)；静态挂饰原样复制（`Object.Instantiate(src, targetHead, false)` + 照抄 local TRS，不要用矩阵 lossyScale 分解）见主表早期行口径。

## 12. 服装「少了一条腿」可能是作者不对称设计（第 132 轮）

**症状**：服装「少了一条腿/只有脚踝一截」，看着像网格缺了一半。
**根因**：原作者做的**不对称设计**——贴图图集里画着两只长度不同的袜子（一只长条纹、一只矮胖波纹），网格连通块按设计分别对应；不是 retarget 丢面。
**判别三步**：
1. 按索引缓冲 BFS 分连通块，看每块 `z`（沿肢轴）跨度；
2. 把左右两侧三角形分色叠画到 `_MainTex` 上（`uv → (u*W,(1-v)*H)`），看各自落在图集哪块图案上；
3. 临时 `Unlit/Color` 纯色渲染看几何本身是否成对。
**判据**：两侧 UV 都落在有图案的区域且图集里本就是两只不同形状 → 设计，不是缺陷。

## 12. 来源

- 本工作区实测 2026-11-06 至 11-11（第 141–177 轮相关条目）与主表早期行，原载 `08-症状速查`，2026-11-11 结构化并入本册。
- Unity 官方：`SkinnedMeshRenderer`、`Mesh.bindposes`、`AvatarBuilder.BuildHumanAvatar`；Blender issue #123088（4.x FBX 法线缺陷，第 177 轮联网核实）。
- 关键脚本/产物（工程内）：`R400HallRebind.cs`、`R400Fix.cs`、`Gloves_fingerw.asset`、`Gloves_FingerFit.asset`、`Gloves_RigidMitten.asset`、`LH_Tops_hood_060_046_018.asset`、`SyncDancesBuilder.cs`。

## 13. 合并衣装拆为独立上下装：原始单位、连通岛与共享腰线（2026-11-12）

**本工作区实测**：LuzzyMeow V2 / 沐梓溪（小狼）Ver.2，Unity 2022.3.22f1c1、MA 1.18.7、NDMF 1.14.8、AAO 1.9.21。素材仅使用当前 Assets 内 CandySweat 与 TamagoOverall。源 FBX、原 prefab、原材质不改，生成新 Mesh 并替换场景实例。

### 13.1 先核对 native mesh 坐标，展示缩放不能进入实施数据

- `sharedMesh.vertices` 是网格局部坐标；FBX/SMR 层级的旋转与缩放还会作用到显示尺寸。不要以截图单位、可视化脚本放大后的数字直接切原网格。
- 本轮独立调研脚本误乘 100，得出展示平面 `z=.52`，但 Unity 原始网格 z 范围仅 `.003220688.. .006474907`；真实平面是 `.0052`。错误平面产生 **0 个交点**，被实施侧数值检查拒绝，不能把“文件生成成功”当拆分通过。
- 原始表面积约 `4.15697837e-05`；乘 100 展示后面积变成约 `.415697837`（放大 10000 倍）。同时记录输入 bounds、平面、面积单位和预计 crossing 数，能迅速发现这类错误。

### 13.2 两类拓扑应分开处理

| 类型 | 做法 | 本轮实测 |
|---|---|---|
| 上下装为独立布片 | 按完整连通岛分配三角形，保留材质与蒙皮通道 | TamagoOverall：上衣 4201 tris / 3384 vertices，背带裤 2391 tris / 2045 vertices；原 6592 tris 全覆盖、无交叠、无共享顶点 |
| 上下身连成一个布片 | 在已确认的腰环切面插入交点；每个被切三角形分别裁剪到两侧并重三角化 | CandySweat：56 个跨面三角形、59 个无向边交点；上衣 18566 tris、下装 1598 tris；总 20164 = 原 20052 + 56×2 |

不能把同一个 SMR 绑到两个 ObjectToggle 来伪装上下装独立；也不能把跨面整三角形整块归侧后宣称接缝平整。连通岛可辅助识别 UV/法线拆点，但焊接容差必须按**原始坐标**选择，不能直接搬用展示单位下的容差。

### 13.3 插点与打包的保留规则

1. 无向边 `(min(vertexA,vertexB), max(...))` 缓存交点，两件共用同一份插值数据；位置固定到原生切面，UV0..7、颜色线性插值，法线与切线方向插值后归一化，切线 handedness 保留。
2. 骨权重按 boneIndex 合并插值，取最大 4 项并归一；`bindposes` 及与其对应的 `bones[]` 顺序不变。本轮新点只涉及 2–3 根骨，未丢失有效影响。
3. 每件可剔除未引用顶点，但必须显式重映射索引及**所有逐顶点通道**。两件最终索引可以不同，接缝的位置、UV、法线/切线与骨权重数值必须仍同源。
4. 源网格有 blendshape 时须同步处理全部 shape/frame/delta，不能只复制 frame0。本轮两源网格 `blendShapeCount=0`，因此该环节不适用；不能把此简化推广到有形态键的衣服。

### 13.4 必须实测的判据与本轮证据

- 全部原三角形被“上侧完整 / 下侧完整 / crossing”三集合完整覆盖，三集合两两不交；索引无越界、无退化三角形。
- 插点后面积守恒：Unity 实际生成原面积 `4.1569783681067212e-05`，拆后 `4.1569783683565214e-05`，相对误差 `6.009176820769233e-11`。
- 接缝两侧各有 59 个同源交点；4 个 Mesh 各 99 个 bindposes，独立逐元素比较相同；最大权重和误差约 `1.28e-7`。
- 剔点后的 CS 上衣 12340 vertices / 下装 1194 vertices；全部新 Mesh 未引用顶点为 0。**顶点数不要求相加等于源顶点数**，因为两件会各保留一份接缝点。
- 数值通过不能代替穿模/蒙皮验证：必须 Bake → GM Play 实际菜单上下装开关 → Walk/Run → 相机腰线特写。若骨骼本身不动，不能把它诊断为衣服绑骨错，也不能把移动验收判 PASS。

工程证据：`docs/PLAN-001-r520/V3-candy-final-geometry.txt`、`geometry-review/*.json`、`research/cs_split.json`、`research/tg_split.json`；生成资产 `Assets/_Gen/WolfR520/Mesh/{CS_Tops,CS_Pants,TG_Tops,TG_Pants}.asset`。本节记录的是几何/绑定数值事实；客户端与 FBT 尚未据此验收。

**官方依据**：<https://docs.unity3d.com/ScriptReference/Mesh.SetVertices.html>、<https://docs.unity3d.com/ScriptReference/Mesh.SetUVs.html>、<https://docs.unity3d.com/ScriptReference/Mesh-bindposes.html>；Mesh 数据结构 <https://docs.unity3d.com/Manual/AnatomyofaMesh.html>。当前文档版本较新时，用本机 Unity 2022.3 反射及实际数据读回确认 API/通道存在，不能假设版本完全相同。
