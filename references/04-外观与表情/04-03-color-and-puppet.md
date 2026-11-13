# 颜色档位、滑块（Puppet）与 lilToon 观感

读这份文件的时机：要给 Avatar **加 / 改颜色选项、程度滑块**，或**调整头发的层次与高光**。

> 本分册描述**判定顺序与候选做法**；属性名、数值、绑定方式与视觉结果**都要在目标工程现场验证**。
> 材质机制见 [04-01-appearance-and-liltoon.md](04-01-appearance-and-liltoon.md)；菜单上限与参数成本的完整算法见 [03-01-parameter-and-menu-audit.md](../03-菜单与参数/03-01-parameter-and-menu-audit.md)。

## §1 菜单容量与路由

1. ⚠️ **每个最终 `VRCExpressionsMenu` 最多 8 个控件**——**包括该页的子菜单入口和其它选项**，不是「每页还能额外放 8 个颜色」。超出时拆子菜单，或使用工具已支持的分页并**检查生成结果**。
2. **`saved` 与 `networkSynced` 是两件事**：`saved` 控制参数持久保存；`networkSynced` 控制网络同步**及同步参数预算**。⚠️ `saved=true` **不会**让 Puppet 消失，**也不要求** Float 滑块设为 `false`。
   - SDK 3.10.5 已验证示例：Bool 在 `saved=false/true` 两种情况下，`networkSynced=false` 都计 **0 bit**，`networkSynced=true` 都计 **1 bit**。**不要用 `saved` 推断预算**；其它类型与最终总量按目标 SDK 的预算统计验证。
3. **连续轴用 Float，通过 `subParameters` 路由**：Radial Puppet 用 **1** 个轴参数（0–1），Two Axis 用 **2** 个（各 −1–1），Four Axis 用 **4** 个（各 0–1）。检查控件类型、**数组长度**、
  参数名与最终 Float 定义；⚠️ **空数组可能让模拟器直接索引报错**，**不能靠切换 `saved` 来修**。
4. ⚠️ **默认值不是一律 0**：按用户指定的默认外观设置源参数配置，并核对最终表达式参数的 `defaultValue`、Animator 默认值与初始化逻辑。**已有 saved 值或 Parameter Driver 可能覆盖初始值。** 工具生成参数时应改**稳定源配置**再检查构建输出，**不是强行手改两份可能被覆盖的资产**。
5. **菜单显示名要查生成规则**：例如已检查的 MA 1.14.3 中 `ModularAvatarMenuItem` 在 `label` 非空时用它，否则用 GameObject 名生成控件名称。其它工具 / 版本可能不同。**改实际负责显示名的字段并读回生成菜单**；⚠️ **不要为「保险」盲改 GameObject 名**，可能影响动画层级路径。
6. **0–1 程度滑块的一种接法**：Float 参数驱动 Animator 状态的**归一化运动时间**（`timeParameterActive = true`、`timeParameter = <PARAM_NAME>`），clip 在时间 0 与 1 秒写目标属性端点。
  确认 **clip 长度、非循环端点、插值方式、过渡及其它层的覆盖**；需要线性响应就设置并**验证曲线切线**。⚠️ **shader 属性范围不一定等于滑块范围**，要明确映射后实测中间值。

## §2 「挑染」（多色发）到底改哪里

1. **先找颜色来源**：albedo、第二/第三主贴图、Emission、MatCap、遮罩或其它层都可能参与。记录原值，在**可恢复的预览副本**上逐层 A/B。⚠️ **不能凭「看起来像挑染」就认定是发光。**
2. 若目标 shader 确实用 `_EmissionMap` 与 `_EmissionColor` 生成挑染，**主贴图变化可能保留大部分发光色**。调整该发光色是一种候选；彩色发光贴图、混合方式与亮度同样影响结果——**单改一个颜色不保证得到任意目标颜色**。
3. 需要**重烘焙 albedo** 时：先按色相 / 遮罩分区统计，再替换选定区域色相并尽量保留明暗。**保留 S/V 只是一种方法，不保证感知亮度不变**；记录色彩空间并用同光照图像核对。
   - ⚠️ **写 8-bit LDR 图前，把归一化 RGB 限制在 0–1 再转 uint8**，避免越界转换产生回绕或异常；**但不要因此把 HDR 发光值强制截断**（见 appearance-and-liltoon §5）。
4. **读回材质的 HSVG 调整**。若新贴图已烘入同一调整，才在**派生材质**中撤去重复变换；⚠️ **不要无条件清掉作者有意设置的调色**。中性值从**实际 shader** 读取。

## §3 头发层次与高光：可比较的候选方案

以下命名在检查过的 **lilToon 2.3.0 标准 shader** 中可以找到，**不代表其它版本 / 变体都支持**；先按 [04-01-appearance-and-liltoon.md §1](04-01-appearance-and-liltoon.md) 确认目标材质，再逐项 A/B。**没有通用的效果排序或必选数值。**

| # | 候选方向 | 涉及属性 / 要点 |
| --- | --- | --- |
| 1 | **阴影配色与过渡** | 比较 `_UseShadow`、`_ShadowStrength`、`_ShadowBorder`、`_ShadowBlur`、`_ShadowColor`、`_ShadowMainStrength` 对发色的影响。**彩色或中性阴影都是设计选择**，不必强制灰色禁用或某强度为 0 |
| 2 | **第二层阴影** | 查看 `_Shadow2ndColor`、`_Shadow2ndBorder`、`_Shadow2ndBlur` 及功能配置。⚠️ 该版本**没有 `_UseShadow2nd` 这个材质属性**——确认实际启用路径后测试层次，**不要猜一个开关名写入** |
| 3 | **法线贴图候选** | 先检查已有法线、切线与贴图导入设置。只有亮度图**确实适合作高度近似**时，才尝试「平滑高度图 → 梯度 → 构造并归一化法线 → 编码到 0–1」的**派生贴图**，并以 Normal map 导入。⚠️ **albedo 的色块 / 绘制阴影不等于几何高度**；近景检查强度、接缝与方向。**法线贴图不能修复真实穿插或重叠面** |
| 4 | **各向异性高光** | 若 `_UseAnisotropy` 及相关宽度参数确实可用，结合**网格切线 / 发流**测试高光方向。⚠️ **避免仅开功能就宣称解决了「平面感」** |
| 5 | **MatCap** | 一起比较 `_MatCapBlend`、`_MatCapColor`、混合模式与遮罩；**不要求 blend 恒为 1**。换观察角度复查高光是否压掉底色或细节 |
| 6 | **背光 / 边缘光** | ⚠️ `_UseBacklight` 是**启用开关，不是亮度值**；颜色、强度及 `_RimColor` 等按真实属性表调整。检查明暗环境下是否过曝 |
| 7 | **对比条件** | **固定相机、姿态、光照与曝光**，提供**近景 + 全身** A/B；近景看接缝 / 涂抹感，全身看整体风格 |

## §4 形变（blendShape）相关

1. 穿模可能来自**身体与衣物形变不匹配**，也可能来自基准几何、蒙皮、透明排序或层叠薄片。⚠️ **同名形变不证明它们兼容或互相顶撞。** 分别读每个网格的**实际权重与驱动**，临时隔离 renderer 帮助定位，再**恢复共同显示**检查遮挡关系。
2. **淡出区间用近景逐档实测**决定；记录每档的参数、形变与材质状态，**不照搬其它工程的分段**。
3. 削弱高段形变有时能减少穿透，**但也会改变目标体型**；只在用户目标与既有授权范围内采用，并**逐档检查塌陷与其它服装**。⚠️ **透明度变化不会证明几何穿透已经修复。**
4. 某个衣物专用形变**不一定能代替**目标形变；先检查其位移与驱动链再决定用途。A/B 结束后恢复预览状态；用户明确授权的源默认值改动则按该目标保存。

完整的穿模处理优先级、几何验收与可移植性判断见 [05-01-outfit-fit-and-clipping.md](../05-动骨与穿模/05-01-outfit-fit-and-clipping.md)。

## §5 Unity / 工具注意事项

1. **删除资产前**核对源引用、依赖图、**动态路径使用**与备份；⚠️ **文本搜索没有命中不足以证明无人使用**。**没有通用的「删几十个必崩」阈值**；批量策略应以当前 Unity 版本、导入器与日志为依据，并在修改完成后按需刷新 / 导入。
2. ⚠️ **`AnimationCurve.keys` 返回的是副本**。修改取得的 keys 后**赋回 `curve.keys`**，或使用曲线修改 API，再用 `AnimationUtility.SetEditorCurve` 写回 clip。新建曲线时**保留所需切线、权重与 wrap mode**——不必为了改一个 key 丢弃其它曲线行为。
3. **源资产与 Play / 构建副本不是同一个对象。** 改源资产**不保证**刷新已有克隆；需要新生成结果时**退出 Play、确认源修改已保存 / 导入**，再按当前工具配置重新构建。某些直接引用的资产会即时变化——⚠️ **不能把「Play 中所有改动都无效」当规则**。
4. ⚠️ **MCP 超时后先读回状态再决定是否补做**（可能仍在执行）；避免同时重写同一资产，脚本尽量**幂等**。详见 [07-04-verification-discipline.md](../07-验证与排查/07-04-verification-discipline.md)。
5. 读材质时先确认**属性存在与实际类型**，再用匹配 API；⚠️ **错误类型的读取结果不能当作「功能关闭」的证据**，也不要假定所有 Unity 版本都返回相同的 0 值。
6. MA / VRCFury / NDMF 等可能生成控制器、菜单、材质或贴图，**取决于启用的功能与版本**；**生成路径不是固定接口**。定位**稳定源配置**，再检查预览或构建副本；**不要把临时输出当长期修复目标**。

## §6 验证纪律

- **分开记录四层**：源配置 / 编辑器预览或模拟器 / 实际构建输出 / VRChat 客户端运行。Play 是否执行相关构建插件要**现场确认**；**本地模拟通过不证明同步或保存行为正确**。
- 外观结论**结合读回值**（形变权重 / 材质身份 / 参数类型）**与固定条件截图**；结构性检查与视觉验收**各回答自己的问题**。
- 颜色档位用**发色 / 瞳色平铺 montage** 比较，挑染 / 穿透用**近景**。列出**「档位 × 服装」覆盖表**，明确实际测过与未测的组合，**不把抽查称作全覆盖**。
- 导入、本地构建、测试客户端或上传**按已有授权执行**；已明确授权的步骤**无需重复请示**，未运行的层级照实说明。

## §7 Radial Puppet 的最小结构与「一跳就满」排查

**症状**：客户端里滑条一碰就跳满，或控件没有按连续滑条表现；模拟器表现可能不同。

⚠️ **控件是否为 Puppet 由 `type` 决定，不是由 `saved` 或「是否填了主 `parameter`」决定。** Puppet 的主 `parameter` 用于**打开控件时设置激活值、关闭时复位**；真正的**轴值由 `subParameters` 驱动**。
两处误用同一个参数可能发生**写入冲突**，但**同时存在两处参数本身并不把 Puppet 变成 Toggle**。

**普通单轴滑条的最小结构**（可再按需增加独立的激活参数）：

- 最终 SDK 控件的 `type` 为 `VRCExpressionsMenu.Control.ControlType.RadialPuppet`；源工具用**对应枚举 / Inspector 选项**，⚠️ **不盲写整数枚举值**。
- 不需要激活标记时，`parameter.name` **留空**。
- `subParameters` 包含 **1 个轴项**，`subParameters[0].name` 指向**真实的 Float 参数**。
- `value` 是**主参数的激活值，不是连续轴的最大值**；主参数为空时**不靠它**设滑条范围。标签按该控件类型与工具的支持配置。
- 轴参数的 `saved` 按**是否需要持久保存**设置，`networkSynced` 按**是否需要其它客户端同步**设置；**两者独立**。

**排查顺序**：核对最终类型与轴路由 → 查主参数是否误写同一个 Float → 查 Animator / Parameter Driver / 其它控件是否**重复写入** → 检查默认值、时间驱动与端点映射。

可与同工程中**用户确认可用**的滑条比较，但⚠️ **不能仅凭字段照抄就证明新驱动链可用**。

修改稳定源后**重新生成**，读回**实际被测试头像**的菜单与参数，再在**跨帧预览**中检查中间值。真实客户端的**连续输入、持久保存与网络同步，只在相应运行测试后才算验证**；未测试时明确保留该项。

## §8 材质同步层（FX 对象引用曲线）漏 mesh：换发色后耳朵变了、头发没变

### §8.1 症状

某个颜色档位（本例「黑短发」）切换后，**耳朵 / 尾巴 / 发根都变色了，但主头发还是旧颜色**；Console 0 报错，Edit Mode 静态看也「挺正常」。

### §8.2 根因

- 发色同步由 FX 控制器的 **5 个对象引用层**完成（本例 `Kipfel_FXLayer_AvatarWardrobe.controller` 的 L54–L58）：
  - L54 → `Cat_Ear[0]`、L55 → `Body[1]`、L56 → `Cat_Tail[0]`、L57 → `Hair_Base[0]`、L58 → `Cat_Tail[1]`
  - 每层 3 个 state（`白` / `少年黑发` / `绅士灰发`），motion 是子资产 clip 的 `m_Materials.Array.data[N]` 对象引用曲线，`writeDefaults=True`。
- **漏掉的物件**：短髮主网格 `.../looseShorthair_kp_v1.0/ls_hair` 的 `sharedMaterial` 被静态固定在 `ls_hairmat_white`（白），**不在任何颜色层里** → 切黑发时它不跟着改。
- 佐证：层的 state 名本来就叫「少年黑发」→ 设计本意是黑发，正确修法是把主网格也接进颜色同步，而不是改标签。

### §8.3 排查（逐 renderer 隔离渲染取色，最快）

1. 在 Play / Edit Mode 建临时相机，`root.GetComponentsInChildren<Renderer>(true)` 记下 enabled → **全关** → 只开目标 transform 下 renderer → `cam.Render()` 到 RenderTexture → 统计非背景像素均值 / 覆盖率 → 逐个 renderer 还原。
2. 本例 style1 实测：`Cat_Ear` avg #1a181a、`Cat_Tail` #1f1c1e、`Hair_Base` #100f10（黑），**`ls_hair` #e8e8e8（白）** → 一眼看出漏网的是 ls_hair。
3. ⚠️ **不要用 `SkinnedMeshRenderer.bounds` 定位**：baked clone 上 Body / Cat_Ear / Cat_Tail / Hair_Base 的 bounds 会被膨成同一个大盒子（本例 size≈(2.26,2.25,2.27)），
  拿它取景 / 取样必错。用骨骼世界坐标（头骨 `Head` 实测 (0, 0.718, 0.004)）或 `BakeMesh` + `smr.transform.TransformPoint`。

### §8.4 修法

- 若该部件只在对应档位可见（本例 ls_hair 仅 style1 显示，由 Hair Style 三层 + `Restored Feature Defaults` 的 `m_IsActive` 控制）：
  直接把场景实例上该 renderer 的 `sharedMaterial` 改成目标材质（本例 `Assets/AvatarFeatures/Material_212.mat`），`EditorSceneManager.SaveScene` 回存。
  换 sharedMaterial 与 `Avatar Brightness Control` 的实例材质曲线（`material._*`）不冲突。
- 若该部件在多档位都可见：不要静态换材质，应把 renderer 加进对应 FX 层的 clip（新增对象引用曲线）或改用共享材质槽方案。
- 双语 / 变体 prefab（如 `Kipfel_Kisekae_English.prefab`）是另一套材质引用，**不会自动跟着改**，报告里要注明。

### §8.5 判据

- GM Play 三档发色逐档截图 + 头发区域像素采样：本例 头髮取样点 (0.5,0.30) white=(233,235,233) / **blackShort=(33,33,38)** / silver=(186,182,181)；头部 top-60% 均值 182.5 / **138.0** / 171.7。
- 全量模拟测试回归 141/141 PASS（无回归）。

## §9 亮度滑块：用 `timeParameter` 驱动 clip 的「发光」实现

### §9.1 结构（排查这类滑块先看这三点）

- 本例 `外观 → 整体亮度`：FX 第 13 层「Restored Feature 10」，state machine / state 都叫「发光」，motion = AnimationClip `Avatar Brightness Control`（1 s、60 fps）。
- `state.speed = 0`、**`timeParameterActive = true`、`timeParameter = "发光"`** → 滑块值 (0..1) 直接当 clip 的归一化时间，**不需要 blend tree**。
- 曲线关键帧 t=0.00 / 0.50 / 1.00：`_LightMinLimit` 0→0.05→1；`_LightMaxLimit` 0→1→10；`_MainTexHSVG.z` / `_OutlineTexHSVG.z` 0→1→2；
  `_EmissionBlend` / `_Emission2ndBlend` / `_MatCapBlend` / `_MatCap2ndBlend` / `_RimColor.a` / `_BacklightColor.a` / `_GlitterColor.a` / 
  `_ReflectionColor.a` 0→1→1（**0.5 后饱和**）。

### §9.2 视觉基准（600x800 正面全身，实测 2026-10-21）

| 发光 | 0.00 | 0.50 | 0.75 | 1.00 |
|---|---|---|---|---|
| 中央区均值 | 59.41 | 144.08 | 152.52 | 152.58 |

- ⇒ 0.5 = 中性、0 = 全暗、1 = 最亮；因 blend 在 0.5 饱和，0.75 与 1.0 视觉接近。
- 实测方法：play 模式把 baked FX controller 挂到 root 的 Animator（`runtimeAnimatorController` + `Rebind()` + `Update(0.02f)`），用反射设 `发光` 后渲染。
  **同一支 `execute_code` 里「设参数 + 立刻 Render」会拍到旧状态**，必须拆成两次调用（set → 返回 → 再 render）。

### §9.3 三类常见 bug 的落点

| 现象 | 查哪里 |
|---|---|
| 滑块拖了没反应 | state 的 `timeParameterActive` / `timeParameter` 是否指对参数；clip 曲线是否在 0..1 时间 |
| 默认值不对 | EP 参数的 `defaultValue`（baked EP 为准） |
| 调好不保存（切头像回默认） | EP 参数 `saved` 标志 → 见 `03-菜单与参数/03-01-parameter-and-menu-audit.md` §8 |
