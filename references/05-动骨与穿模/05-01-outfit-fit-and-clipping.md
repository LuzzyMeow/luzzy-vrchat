# 服装接入、形变、开关与穿模

读这份文件的时机：**已授权的**服装接线、形变、菜单与穿模工作。

> 本分册**不是**新工具安装授权，也**不是**无条件构建授权。先读基底 [SKILL.md](../../SKILL.md) 并确认工具链角色。
> 参数的同步 / 保存语义与菜单上限见 [03-01-parameter-and-menu-audit.md](../03-菜单与参数/03-01-parameter-and-menu-audit.md) 与 [04-03-color-and-puppet.md](../04-外观与表情/04-03-color-and-puppet.md) §1；
> provider 专属流程见 [07-03-provider-playbooks.md](../07-验证与排查/07-03-provider-playbooks.md)。

## §1 先定位，后判断可移植性

1. 从**玩家菜单或明确的 renderer 入口**追踪控制参数、provider、controller / clip、实际对象与共享消费者；**不要按名称相似就重复导入**。
2. 读取来源 SMR 的：**mesh GUID/fileID、骨骼 Transform、bindposes、根骨骼、材质槽、子网格、UV、全部形变帧与权重、局部变换、默认姿态**。
3. 比较目标、来源与 scene instance。⚠️ **编辑态身体和衣服必须用同一体型**；运行时还要看**哪些层会写这些值**。**用户的身体 / 材质 / 服装手工状态不擅改。**
4. **骨骼名称只帮助找候选。** 重复名、不同骨架、零偏移子骨骼都需要精确 Transform / 路径与**蒙皮矩阵**比较。⚠️ **不同名不等于不同运动，同名不等于相同骨骼。**
5. ⚠️ **缺失骨骼不能静默全部绑到父级。** 查实际权重、rest pose、约束 / PhysBone、provider 的合并规则；**只有用户接受且动作验证支持时**才做替代映射。
6. ⚠️ **同顶点数不足以直接复制形变。** 确认对应索引、三角形 / 子网格拓扑、UV、坐标与资产来源；**索引不同先用可靠映射**。

## §2 源资产、蒙皮与形变

1. **按所有权决定是否复制**：只读引用同项目稳定资产可以保留；需要独立修改、跨项目导入或来源生命周期不受控时**复制，并保留依赖与 `.meta`**。**不要因为资产是 FBX 子资产就一律断开来源。**
2. ⚠️ **`bones[i]`、`bindposes[i]` 与 mesh 权重的 bone index 必须对应。** 追加 / 替换骨骼时**同时校验 mesh 与所有 SMR 消费者**——**不能为一个 renderer 改 `sharedMesh` 索引，却让另一个 renderer 留着旧骨骼数组**。
3. 绑定 / 坐标问题优先比对：**`renderer.worldToLocalMatrix * bone.localToWorldMatrix * bindpose`**。Bake 或数学求值可验证几何；**实际动画 / PhysBone 跟随仍需相应环境中的动作测试**。
4. `ClearBlendShapes()` + `AddBlendShapeFrame(...)` 适用于需要重建形变的版本，但⚠️ **必须缓存并保留每个 shape 的所有 frames / weights / dV / dN / dT 及其顺序，不能只复制 frame0**。顶点、骨权重、bindposes、UV、子网格与索引也要保留 / 校验。
5. ⚠️ **新增形变后必须检查每个消费者的 `m_BlendShapeWeights` 数组与场景覆盖。** 旧数组长度会让新 shape 的**默认权重变成 0**；检查**名称 → 索引映射**，**不能只看 prefab**。
6. **局部平滑前先定义 ROI、边界、接缝 / 重合多层与允许变形量**。先测退化面 / 法线 / 切线再选算法。⚠️ **迭代多不等于精度高，保最大位移也不等于保外观**；**不要把「法线全部重算」或「切线 delta 全归零」当通用修复**。
7. 写回 source 用 Unity API 并保存目标；**重计算离开 Editor 主线程**。只有外部文件恢复 / 替换才显式导入，**再分开读回**。

## §3 开关、参数与菜单

1. **在已选 provider 的稳定源中编辑。** provider 已负责参数 / 菜单时，**不要另往根 Descriptor 追加重复声明**，也不要拆散插件子树。
2. **Outfit 档位 clip 要写全该档所拥有的对象状态**；从其它衣服 / 配件状态**进入再返回**测试，**避免只开新件而遗留旧件**。
3. **参数类型从消费者反推**：二态通常 Bool、多档通常 Int、连续量 / Puppet 通常 Float。读 Animator、菜单 `parameter/subParameters`、driver、blend tree、ExpressionParameters 的**实际类型**；**冲突要解决，不凭控件外观推类型**。
4. `saved` 是**持久化**，`networkSynced` 是**同步**。默认值按用户明确要求设置；⚠️ **查所有写者**（尤其状态进入 driver、初始化 / 延迟驱动），否则 `saved=true` 也会被重置。
5. 新层要明确 **`defaultWeight`、混合模式、mask、Write Defaults、motion**。**独立需要始终激活的层通常设 1**；改已有层需要**定位证据与用户授权**，**不应为了消警告盲改**。
6. 改状态、子资产与 driver 后**保存并独立重读**，必要时重载。⚠️ **不要删了 clip 重建后继续使用旧 Object / GUID 引用。**
7. 菜单位置优先遵从需求与既有结构；**只有缺失会严重影响用户语义的决定才询问**。按**实际控件类型**设置参数，**不用子菜单 / 标签数量猜 `type`**。
8. **保存 / 读回 provider preview。** 最终菜单 / 参数合并、分页、后处理行为在获授权的生成 / 构建层复核；**没跑就写 `BUILD_REQUIRED`**。

## §4 形变档位与连续滑条

- **离散两端分别对应约定的最小 / 最大状态**；**默认参数位置**对应原默认外观。⚠️ **不能要求两个不同端点都等于同一个默认值。**
- 连续滑条可用 **Float 时间参数驱动 clip**：现场核对 `AnimatorState.timeParameter` / `timeParameterActive`、**clip 长度与归一化时间含义**，再读关键点 / 中间值 / 切衣 / 恢复。
- ⚠️ **非线性曲线可以是设计**；避免意外过冲，**不强制一切变化单调**。身体 ↔ 衣服形变常用**不同名字与不同函数**——**不用「位移 max × 倍率」推断有效覆盖率**。
- 先找**最后写者、实际进入状态、参考姿势与输出**。⚠️ **Additive 常量负数不一定减到目标，更不保证夹到 0**；需要归零可用**已测试的 Override 绝对值**，但必须验证**退出 / 全裸 / 停止后下层能恢复**。
- **编辑模式默认值与 runtime clip 默认采样值可能不同**；若需编辑预览与当前用户体型一致，**只同步依赖的衣服，不擅改身体来让画面好看**。

## §5 穿模检查与修复

- **先区分症状层**：编辑态静态 / provider preview / 生成头像 / Animator 动态 / 真实 PhysBone 与客户端。编辑 Scene View 可以显示**实际 SMR 形变**；等编辑器更新、读权重、同机位 A/B。⚠️ **它不证明最终菜单 / PhysBone 动作。**
- **静态问题先排**：默认体型不一致、过期形变数组、mesh / 骨骼覆盖。**运行时问题再查**实际对象、controller 或 PlayableGraph、参数、到达状态、层权重 / mask / 参考姿势与后续写者。
- ⚠️ **「衣服不跟」只是症状。** 可与同工程正常衣服对照，但**要验证全逐点权重 / 矩阵 / 对应关系及动态位移**，**不能把平均权重或骨骼名差异当根因**。
- **分层验**：身体 ↔ 内衣、内衣 ↔ 外衣、裙摆 ↔ 腿。至少约定**体型关键档与动作 / 停止 / 换装 / 全裸转换**；⚠️ **静态图通过不代表晃动通过**。

### §5.1 风险顺序

| 顺序 | 适用手段 | 条件 |
| --- | --- | --- |
| ① | 隐藏**已存在的**独立新增 / 互斥几何，在已有档位写 `m_IsActive=0` | 用户允许隐藏且不丢所需外观；⚠️ **不要以「先隐藏」为借口隐藏身体来假通过** |
| ② | 形变同步、蒙皮 / 绑定修正、provider 配置 | 有证据表明身体与布料关系不对；**验证所有受影响消费者** |
| ③ | 局部顶点修改 / 可逆覆盖形变 | 前两项不适用或已不足；**完整备份**、明确形状代价与限制，并做**数值与图像**验收 |
| 不采用 | 本工作流中的**补片 / 加面** | 项目禁止或未授权的结构性改模不要尝试；⚠️ **这不是「所有补片在 Unity 都不行」的通用定律** |

若**外观明显变差，就恢复该实验**，不要在其上继续累加推移。⚠️ **保留失败结果与完整 ROI**，**不能靠改变阈值或缩小目标区域来宣称通过**。

### §5.2 几何验收

1. **统一坐标**：使用 renderer / 骨骼的**实际变换**；normal 用正确的**逆转置矩阵**；负缩放 / 缠绕 / 多层情况单独核对。
2. 对定义好的 ROI 做**三角形重心 + 边中点或更密**的采样；每点求**目标三角形表面**的最近点 / 相交，⚠️ **不是最近目标顶点**。
3. **预先定义**符号、容差、边界 / 退化面与不可判断情况；⚠️ **开口 / 远面最近点的符号可能无意义，不能把它自动改成「0 穿透」**。
4. 报告：**采样总数、阈值、外侧 / 未覆盖 / 不可信点数、深度与检查范围**。⚠️ **有限采样不等于整个表面无相交**；动态 / 遮挡 / 自遮挡还需对应测试。
5. **同机位、同姿态 / 输入 A/B**，记录源 / 生成 / 客户端层及执行范围、资产回滚方式。⚠️ **几何检查不能替代材质 / 透视图像。**

## §6 数值与发布记录

记录本次**实际路径、版本、参数、输入权重、源文件指纹、共享消费者、备份、验证环境与未跑项**。⚠️ **不要把项目尺寸、每件件数、骨骼编号、shape max 或缓存层号写成通用要求。**

## §7 MA 1.14.3 菜单路由要点（版本限定）

以下内容在 **MA 1.14.3 已安装源码**中核对过；**其它版本请用 schema / source / preview 确认，不要照搬内部 API**。完整 7 条见 [03-01-parameter-and-menu-audit.md §2.3](../03-菜单与参数/03-01-parameter-and-menu-audit.md)。

1. **SDK 控件枚举的公开值与 `SerializedProperty.enumValueIndex` 不同**——用强类型枚举或当前定义，**不按 0..5 或资产字段组合猜 `type`**；已安装 SDK 还可能有新枚举成员。
2. `ModularAvatarMenuItem` **非空 `label` 优先**，否则名称取 GameObject。`MenuAsset` 的子菜单资产与 `Children` 等来源不同；**场景孩子只有被实际 MenuSource 消费才参与**，**单靠 parenting 不保证可达**。
3. Installer 同物体 MenuSource 与 `menuToAppend` 的优先级**须用本版解析器确认**。有安装目标组件时检查其 `installer` 指向及是否参与枚举——**不能只看 enabled 状态推断路由**。
4. 当前版本 `MenuNodesUnder` **遍历直接子节点**；⚠️ **该事实不表示所有 provider 都只遍历一层**。更改来源 / 安装目标前确认需求、共享菜单与全部引用，**不盲删所有 InstallTarget**。
5. 可用本版编辑器的 **`VirtualMenu.ForAvatar(avatar, context)`** 提供 **PROVIDER_PREVIEW**；⚠️ internal 反射签名会变，**必须先找准确方法**。**不要自己复制完整合并算法**，也**不把 preview 称为 SDK 最终结果**。
6. SDK 每页控件上限读 **`VRCExpressionsMenu.MAX_CONTROLS`**；MA 1.14.3 的 `MenuInstallHook` 构建后处理提供 `More` 分页——⚠️ **不是裸 SDK 保证替任意菜单自动分页**。其它 provider 可能不同；**最终在新生成 / 构建对象上验证**。
7. ⚠️ **未指定目标的 installer 可能安装到根。** 整理时保留 **provider 归属与可重生成源**，**不要把不同插件的菜单叶子混在一棵失去所有权的树中**。

## §8 用 BakeMesh 求配饰 / 网格的真实世界坐标（2026-10-21 实测）

- **症状**：想量某个配饰（光环）在世界里的位置 / 尺寸；Play 模式下被 idle 呼吸动画干扰，多角度截图 diff 覆盖全身，定位失败。
- **正确做法**：Edit Mode 里强制 `smr.gameObject.SetActive(true)` → `var m = new Mesh(); smr.BakeMesh(m);` → 每个顶点用 **`smr.transform.TransformPoint(v)`** 转世界，再求 AABB。
- ⚠️ **不要用 `smr.rootBone.TransformPoint(v)`**：`sharedMesh` 的顶点与 `bounds` 是 **mesh 局部空间**（相对 SMR 自身 transform），`rootBone` 是骨骼空间，两者不是一回事；按 rootBone 变换会算到 `(0, 0.48, -0.95)` 之类假位置（曾据此连续拍错方向）。
- **实测 Avatar_Halo**：baked 顶点 min=(-0.084, -0.040, -0.977) / max=(0.083, 0.126, -0.960) → 世界 AABB x(-0.094..0.093) / y(1.076..1.094) / z(-0.045..0.141)，
  中心 (0, 1.085, 0.048)、直径 0.186、厚 0.018 的水平圆环；head 骨 y=0.718 → 环在头顶上方约 0.37 m，与渲染图吻合。
- **判据**：AABB 中心 / 尺寸与视觉一致，再用 3–8 角度渲染（`RenderTexture` + `EncodeToPNG`）确认；截图写 `tools/tmp`，别污染工程。

## §9 ParentConstraint 多目标配饰（眼罩 / 面罩）定位与量化验收（本工作区实测 2026-10-25）

- **症状**：用户说「眼罩初始位置和交互拉取的最低位置都不在眼睛上」；Hierarchy 里 `[設定用]アイマスク` 由 `ParentConstraint` 3 个源（`[初期]` / `Target1` / `Target2`）加权驱动，FX 层按状态改权重（睡眠模式 0 → 0.5 → 1）。
- **先量化三个基准，别靠肉眼猜**：
  1. `Body` SMR `BakeMesh` → 眼带顶点（`|y-眼高| < 0.015 && |x| < 0.055`）的 `z` 前沿（实测 0.0764）与眼睛 y（ViewPosition 0.68，眼带 0.665~0.695）；
  2. 面罩 SMR 的 `MeshFilter.sharedMesh`：取「最前 3cm」（`z >= zmax-0.030`）顶点质心当作**镜片中心**；
  3. 网格 local size 判朝向（本资产 `(0.1345, 0.1445, 0.0606)`：X 宽 / Y 绕头方向 / Z 厚度）。
- **反解 pivot**：把面罩 transform 临时设 `position=Vector3.zero`、`rotation=q`，取镜片质心的世界坐标当 `offset`；则 `pivot = 期望镜片中心 - offset`。
  用同一 `q` 依次摆 `Target2`（镜片中心 y = 眼睛 y，镜片前沿 = 脸前 + 0.009）、`Target1`（前方过渡弧）、`[初期]`（额头上方 ~0.72，**旋转要竖立**；作者原 66° 会让整块埋进头发只露一条边）。
- **验收（没有视觉能力也能做）**：临时相机（`clearFlags=SolidColor`、FOV 26、600×600）+ `RenderTexture` + `EncodeToPNG` 出正视图 → 与「隐藏面罩」基线做像素 diff（阈值 >12）：
  ① 面罩 bbox 中心 x ≈ 画面中心；② bbox 中心 y ≈ 眼角投影 y；③ 面积占比合理（本资产戴上态 8.2%）。实测修正前后：戴上态 bbox 中心 y 343→315（眼角 314.6，差 0.6 px）、初始态可见像素 0.9%→7.2%。
- **单件材质覆盖**：一套耳朵由 3 个 SMR（`S_Ears` / `S_Ears_Halo` / `S_Ears_L_Earring`）共用一个材质时，只想让「光环」
  回原色 → 只把 `S_Ears_Halo.sharedMaterials[0]` 换回原材质（本例 `Ears_Shinra.mat`，原 GUID `5224ba8025d8cc644be4b24ff4b3c396`）；
  从原始压缩包解出的旧场景解析 SMR 的 `m_Materials` GUID 最稳；**EditorOnly 预览副本要同步改**，否则编辑期看到旧颜色。

## 10. ParentConstraint 多目标配饰：约束 offset 不随烘焙重算（本工作区实测 2026-10-25）

**症状**：改了配饰（面罩）自身 transform 的 rest 位姿，进 Play 后佩戴态仍然倾斜/埋进头发（本资产 mask 世界旋转 ~60° 而不是 90°）。

**根因**：Unity `ParentConstraint` 的 `m_RotationOffsets` / `m_TranslationOffsets` 是**作者序列化值**，`ModularAvatar ConvertConstraints` 烘焙成 `VRCParentConstraint` 时**原样传递、不按当前场景位姿重算**。所以：
- 改 mask 自身 rest 位姿 → 只影响「约束源权重=它自己」的那一态（[初期] 待机），不影响 Target 态；
- 最终 = `mask世界旋转 = Target旋转 ∘ m_RotationOffsets[i]`，位置同理（本例 translation offset 全 0，所以位置可以直接改 Target 世界坐标）。

**判据 / 做法**：
1. 先读约束 offset：`var pc = mask.GetComponent<UnityEngine.Animations.ParentConstraint>(); new UnityEditor.SerializedObject(pc)` 迭代 
  `m_RotationOffsets.Array.data[i]`（本例两个 Target 均 `Euler(-28.833,-0.679,-0.668)`）。
2. **反解**：`qNewTarget = qDesired * Quaternion.Inverse(qOffset)`，把 Target1/Target2 的世界旋转设为它；位置直接按「期望镜片中心 − mesh offset」摆。
3. 交叉验证法：连续两次 bake 对比 mask 世界位姿——两次 offset 一致即证明「不重算」（本例 bake1/bake2 解出的 qNew 完全相同 `Euler(61.159,177.923,178.766)`，bake 后 mask = `Euler(89.93,0,0)`）。
4. 该资产实测值（`沐梓溪喵/眼罩/Root/[設定用]アイマスク`）：Target1 `(0,0.706,0.072)`、Target2 `(0,0.678,0.040)`、两者旋转 `Euler(61.159,177.923,178.766)`；[初期]
  待机 `(0,0.7269,0.0716)`。佩戴态 mask 世界 `(0,0.679,0.038) rot(90,0,0)`，镜片正对 `ViewPosition y=0.68`、脸前缘 `z≈0.074~0.079`。

**编辑态预览**：面罩预览副本 `[EditorOnly]プレビュー用アイマスク` 原本是 Target2 的子物体（Target2 在编辑态 inactive → 看不到）；把它移到 `眼罩` 根下并 active，编辑模式即可看到佩戴位置。

**注意**：不要用「改 mask 自身 rest 位姿」来试错定位——每次 bake 都会被 offset 抵掉，只有改 Target 才生效。

## 11. 约束驱动的配饰：做「可拖预览副本」并把位姿反解回约束目标（本工作区实测 2026-10-25，LuzzyMeow V2 眼罩）

- 结构：真实面罩 `眼罩/Root/[設定用]アイマスク` 挂 Unity `ParentConstraint`（3 源 `Target1`/`Target2`/`[初期]`），FX clip 只改源权重做过渡；真实对象默认 inactive，编辑态看不见 → 想手调只能做副本。
- 约束数学（实测）：`mask世界位姿 = 目标世界位姿 ∘ 偏移` —— 位置 `posMask = posTarget + translationOffset`（本工程 0）；
  旋转 `qMask = qTarget * Quaternion.Euler(rotationOffset)`（Target1/Target2 偏移 `(-28.8332,-0.6793,-0.6683)`、[初期] 为 0）。校验：
  旧值 61.159 + 偏移 −28.833 = 89.99 ≈ 90 ✓；反向组合 `offset*target` 得 (89.59,87.56,89.15) ✗。
- 可拖预览副本：复制真实面罩 → 加同样的 `ParentConstraint`（同一 3 源、同 `translationAtRest`/`rotationAtRest`、同每源 offset）。**要能拖**：
  `constraintActive=false; enabled=false; locked=false`；**要跟随**：`constraintActive=true; enabled=true`。两份：初始态权重 `(0,0,1)`、最终态 `(0,1,0)`；
  `tag=EditorOnly`（NDMF 烘焙删除）。
- 反解回目标：读副本世界位姿 = 期望 mask 位姿 → `qTarget = qMask * Quaternion.Inverse(qOffset)`、`posTarget = posMask - translationOffset`；中间过渡 `Target1` 位置取初始/最终中点、
  旋转同最终。写完把副本 `constraintActive=true` 复原权重并读回校验（本工程：初始 `(0,0.7210,0.0685)/90.00°`、最终 `(0,0.6755,0.0444)/89.98°`，与用户拖出的姿态一致）。
- 坑：① 编辑态把真实面罩 `SetActive(true)` 再改权重，约束**不会**立即重算（读回仍是旧位姿）→ 校验必须用「同配置的预览副本」；② 副本 `locked=true` 时拖不动；③ 只改 `[初期]` 目标时用户看到「位置没变」——可见的是副本、不是真实面罩。

---

## 12. 袜子类 skinned mesh 膝盖穿模：顶点级缩放的做法、缩放轴与枢轴风险（本工作区实测 2026-11-05，LuzzyMeow V2 沐梓溪狼 T2）

**判据**：「屈膝时膝盖穿模、放大一点点」的正确做法是**顶点级缩放**，不是改 Transform —— SMR 的 Transform 位移/缩放对蒙皮顶点无效。

**做法**
1. 取 `smr.sharedMesh`，`BakeMesh` 或直接读顶点；顶点在 mesh 局部空间，乘 `smr.transform.localToWorldMatrix` 才是世界坐标。
2. **左右合并网格要分组**：按每顶点主导骨名（`.L`/`.R`）分组，分别算左右各自包围盒，**绕各自中心**缩放；`Hips` 这类无左右属性的顶点绕整体中心。绕整体 `bounds.center` 统一缩 → 左件往左、右件往右平移（中心 ±0.0534 → ±0.0587），表现为「外扩 / 畸形」。
3. 只缩需要的轴：袜子贴腿，XZ 各 ×1.05–1.1、**Y 不动**（Y 放大会把袜筒拉长、顶到膝盖上方）。
4. 新建 mesh 资产替换 `sharedMesh`（保留原资产做回滚点），不要就地改原 mesh。
5. 只想骨级微调 → MA Scale Adjuster（官方：adjust the X/Y/Z scales of a specific bone without causing issues with rotated child bones；primarily intended for use when fitting clothing not originally designed for your avatar），但它只对**单轴缩放工具**生效；多骨带旋转时官方也承认「the scale adjustment will not be perfect」。

**坑**
- **枢轴在脚底 → 整体缩放会上下位移**：缩放中心不在袜子自身，放大后袜口整体位移（视觉上是顶膝盖或往下掉）→ 必须绕每侧顶点包围盒中心缩。
- 真实包围盒不能用 `Renderer.bounds`（世界 AABB，含动画/绑定误差），要用顶点算（`references\07-验证与排查\07-08-multi-angle-binding-verification.md` §1 (L37)）。
- 多骨权重（LowerLeg + foot）在 bind space 缩放下会变成非均匀剪切 → 根治是每侧刚性单骨绑定（`foot.L/R` 下建调整骨，`boneIndex0=0 / weight0=1`，bindposes 用 `worldToLocalMatrix * smr.transform.localToWorldMatrix`），代价是不再随脚踝蒙皮形变。
- 「Y 轴调到 3」这类说法对蒙皮网格无效：要做的是把顶点坐标 `v.y += 0.03`。
- 穿模要有量化判据：BakeMesh 世界顶点 + 法线 → 体素有符号距离（`sd < -1.5 mm` 且 `d < 6 cm` 记穿模，`03-06-ma-menu-build-by-reflection.md` §14.9.4 (L677)），否则「看起来好一点了」无法验收。
- 静态图通过不代表屈膝通过：必须跑动态（GM 里摆屈膝姿态 / 6 方位渲染，本文件 §5）。

**适用场景**：T2 袜子屈膝穿模；鞋子/袜子/袖套这类紧贴多骨蒙皮的部件微调；跨体型移植后的整体放大。

> 更新时间 2026-11-05（本次来源：沐梓溪狼衣装菜单重构 / LuzzyMeow V2 T1–T10）

## 13. 袜子顶点级缩放的实测数据与 A/B 验证法（2026-11-05 补充，LuzzyMeow V2 沐梓溪狼 T2）

**实测：改 SMR Transform 对蒙皮网格的视觉效果 ≈ 0**
- 同一 SMR，`YURU_sox.localScale` 1.00 → 1.03，屈膝特写像素差 **0.06%**（227px / 512×700，bbox 415,294–472,327，肉眼几乎一致）；直腿 10 视角腿部 0.62%–3.01% 的差异来自腿的屏幕位置/姿态而非网格变形。结论：**1.03 的 Transform 缩放不足以改变膝盖穿模**，必须动顶点。
- 正解落地：`Assets/ZZZ_GeneratedAssets/Mesh/RETARGETED__sox_x103b.asset`（从 `Assets/20_逆向/LiIleo狼/Mesh/RETARGETED__sox.asset` 生成的顶点缩放副本，1007 顶点）替换 `sharedMesh`；bounds：原 `(0.1859, 0.1035, 0.2799)` → `(0.1914, 0.1066, 0.2799)`，即 **X×1.0296 / Y×1.030 / Z×1.000**（分别绕左右各自中心；Y 也 ×1.03 是该工具按比例缩放所致）。

**A/B 验证法（可复用）**
1. 在 Play 里摆屈膝：`UpperLeg.R.localEulerAngles=(55°, y, z)`、`LowerLeg.R=(-105°, y, z)`，并 **`root.Animator.enabled=false`**（避免动画覆盖姿态）。
2. 对同一 SMR 依次替换 `sharedMesh`（缩放版 / 原版），相机固定为膝部特写（look = `LowerLeg.R.position + (-0.02, 0.03, 0)`、dist 0.72、fov 32），拍 0°/45°/90° 三角度。
3. 像素差判据：|RGB 差| > 8 记 changed；本轮 152px(0.04%) / 282px(0.08%) / 0px，两态肉眼均无皮肤穿透袜面 → PASS（放大生效但克制）。
4. **方法坑**：不要在 YURU/泳装2 套装隐藏的语境下换 mesh（会把 40-bone 原网格换进 7-bone SMR，差异≈0，白测）；A/B 必须在该套装可见、且两份 mesh 骨骼一致时做。

**关联文件**：`docs/_shots6_diff.txt`、`docs/sheets6/Z_sock_compare.png`、`docs/_final_log.txt`（沐梓溪狼工程 docs）。

> 更新时间 2026-11-05（补充来源：沐梓溪狼衣装菜单重构 · T2 袜子屈膝穿模验收）

> **⚠️ 第 141 轮取代注（2026-11-06）**：`x103b` 版把 **Y 也 ×1.03（沿袜筒轴）**，袜口被拉长顶到膝盖上方、正好压在膝凸上，屈膝仍穿模（bounds Y 0.1035→0.1066；屈膝同机位 A/B 仍见皮肤楔形穿透）。**现行做法见 §14（x108xz：横截面放大、长度轴不动）**；本节数据仅作历史基线。

## 14. 袜子缩放取代版 x108xz：横截面放大、长度轴不动（第 141 轮实测 2026-11-06，2026-11-11 结构化并入）

**做法**：从原网格重建，**局部横截面 X/Y ×1.08、长度轴 Z ×1.00**，±x 分左右各绕自身 bbox 中心。
**产物**：`Assets/ZZZ_GeneratedAssets/Mesh/RETARGETED__sox_x108xz.asset`。
**判据**：屈膝同机位 before/after 渲染无皮肤楔形穿透。
**配套教训（Knee_OFF）**：想用「腿部形态键最低」修袜子膝穿模会把腿压成平面——`Body_Base.Knee_OFF` 是隐藏膝部体积的形变（260 顶点、最大 4.1cm），且袜子是独立 0 形态键网格，身体形态键不带动袜子。袜子类穿模只走「改袜子网格」；身体 OFF 系形态键仅用于被衣服完全遮住的部位。详见 [05-11 §5](05-11-skinning-binding-and-geometry-repair.md)。

## 15. 眼罩类配饰：PhysBone 门控与约束目标定位（第 141 轮实测 2026-11-06，2026-11-11 结构化并入）

### 15.1 隐藏时仍可抓取：Toggle 动画必须同时关 PB_Root

**症状**：眼罩没调出来（隐藏）时，手一碰仍然会交互/拉下。
**根因**：作者 FX `Toggle` 层只写 `[設定用]アイマスク::m_IsActive`，**从不禁用 `PB_Root` 的 `VRCPhysBone`**；抓取体一直活着。
**判据**：Play 里读 `Root/PB_Root.activeSelf`：隐藏态仍 True；`RBS_EyeMask_001_PB`（`Parameter` 前缀 → `_IsGrabbed/_Angle/_Stretch`）照样输出。
**修法**：在 `ToggleOn.anim`/`ToggleOff.anim` 各加一条 `PB_Root :: m_IsActive` = 1/0（官方 `Reset When Disabled` 会让 bones 与参数一起归零，<https://creators.vrchat.com/common-components/physbones/>）；场景把 `PB_Root` 默认 inactive。
**判据**：Play 四态往返 `meshAct/pbAct` = F/F→T/T→F/F→T/T。文件 `Assets/05_配饰/眼罩/SleepEyeMask/_Core/ToggleOn.anim`、`ToggleOff.anim`。

### 15.2 眼罩浮在脸上：只改约束 source 目标，不改面罩自身

**症状**：睡眠眼罩「拉下前/拉下后」都偏前、像浮在脸上。
**根因**：ParentConstraint 的三源（`[初期]` / `Target1` / `Target2`）按时间参数插值，**目标位是按人头摆的**，狼头前发面比人靠后。
**定量**：UP 面板内表面 z=0.1168（前发面 0.1050）、DOWN 0.0839（面部前缘 0.077）；侧面渲染一眼看出 1–4cm 空档。
**修法**：只改三个 source 目标的 world 位（UP/MID 各 −22mm、DOWN −11mm 且 y −4mm），**别改面罩自身**；改完侧面+正面各渲一张复核。对象 `.../EyeMask/設定/[初期]位置調整用オブジェクト`、`.../[装着時]位置調整用オブジェクト/Target1/Target2`。
**通用原理**：约束 offset 不随烘焙重算、可拖预览副本反解见 §10/§11。

### 15.3 改了 localPosition 保存后又弹回：活动约束在整定

**症状**：改了 `localPosition` 保存后过几秒又弹回原值，重启也一样（第 132 轮）。
**根因**：物上有 `ParentConstraint`：`weight=0` 但 `constraintActive=true` + `locked=true` 时，Unity 仍会把 Transform 整定回约束解，直接写 `localPosition` 被覆盖（`localScale` 不受影响，所以会出现「scale 改了、位置没改」的假象）。
**修法**：先 `pc.constraintActive=false`（或 `pc.enabled=false`）再写 `localPosition`，然后 `MarkSceneDirty+SaveScene`，等 5–6 秒复读确认不回弹。
**判据**：连续两次间隔 5 秒读取值一致。活动约束实例上量几何同样不可信（[05-10 §23](05-10-ma-bone-and-geometry-components.md)）。
