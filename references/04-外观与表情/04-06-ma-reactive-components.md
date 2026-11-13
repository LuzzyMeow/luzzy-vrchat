# MA 反应式组件全解（Object Toggle / Shape Changer / Material Setter·Swap / Blendshape Sync / Mesh Cutter）

> **适用**：做换装开关、材质变体、形态键联动、按遮罩/UV/骨权重裁网格；排查「菜单点了没反应」「两个开关抢同一个对象」「穿模到底是真穿还是编辑器假象」。
> **来源**：Modular Avatar 官方文档（<https://modular-avatar.nadena.dev/docs/intro> 起 67 页），2026-11-01 逐页抓取原文（MA 1.18.7，changelog 2026-09-01）；本地副本 `<VRCHAT_ROOT>\tools\tmp\ma-docs-txt\`。每节带官方 URL，可逐条核对。
> **定位**：本册是**官方语义参考**（字段 / 默认值 / 限制 / 烘焙期行为 / 组件间交互），与 `00-基底/00-02-modular-avatar.md` 的代码范式互补；基底是外部 MIT 原件，**一字未改**，本册不重复其已有内容。
> **标记**：【文档未说明，需实测】= 官方没写、须本工程验证；【需查源码】= 字段名未公开、需反射；【待复核】= 与 KB 现有表述冲突、需人工定稿；【KB 已有】= 只补语义。组件名 / 字段名保留英文原名。
> **本册来历**：2026-11-01 第 135 轮「MA 官方文档 × KB 缺口审计」的成品稿整册并入（审计报告在 `<VRCHAT_ROOT>\tools\tmp\ma-gap\`）。稿内「建议落地」一节保留为历史记录，落点已在并入时确定。

## 目录

- §1 反应式总则：1.1 定义 / 1.2 生效前提 / 1.3 父 Menu Item 归属 / 1.4 三类驱动源 / 1.5 Invert condition / 1.6 优先级 / 1.7 时序 / 1.8 调试入口 / 1.9 与其它组件的关系
- §2 Object Toggle ｜ §3 Shape Changer ｜ §4 Material Setter ｜ §5 Material Swap ｜ §6 Blendshape Sync
- §7 Mesh Cutter：7.1 主体 / 7.2 By Mask / 7.3 By Axis / 7.4 By Bone / 7.5 By Blendshape / 7.6 By UV Tile / 7.7 combine·intersect / 7.8 效率与 performance rank / 7.9 常见坑
- §8 Reaction Debugger ｜ §9 与本地 KB 的两处口径冲突 ｜ §10 建议落地 ｜ §11 官方出处清单

---

## §1 反应式总则

出处：<https://modular-avatar.nadena.dev/docs/reference/reaction>

### 1.1 定义与成员

反应式组件 = 「根据**自身 GameObject（及全部父级）的 active 状态**，或 **Menu Item 的选中状态**施加某种效果」的组件，用途是**不手写动画**就能做开关、换装联动、形态键调整。

| 组件 | 官方一句话 |
|---|---|
| **Object Toggle** | controls the active state of other game objects |
| **Shape Changer** | modifies blendshapes on a target renderer |
| **Material Setter** | changes materials on a target renderer |
| **Material Swap** | swaps materials with other materials |
| **Mesh Cutter** | deletes or hides vertices on a target mesh |

> **Blendshape Sync 不属这 5 个之列**（独立页 `docs/reference/blendshape-sync`，不在 `reaction/` 下），但同属换装/形态键联动，本稿一并收录（§6）。

### 1.2 生效前提（缺一不生效）

官方：组件被认为 **active**，当且仅当 ①**它自己与所有父级**在层级中都是 active；②若该对象与某 **Menu Item 在同一 GO 上或是其子级**，则该 Menu Item **处于选中状态**。

**推论（本工作区踩过）**：组件挂在**任何一级 inactive 的 GO** 下 → **整体静默失效**，不报错、烘焙照常完成。"参数对、菜单点了没反应"第一步查**宿主（含所有祖先）是否 active**。
### 1.3 父 Menu Item 归属

官方：**only the first parent Menu Item is considered (parent Submenus are ignored)** —— 只认**第一个**父 Menu Item，**父 Submenu 被忽略**。

- 组件挂在 MenuItem 的 GO 上或子级 → 受「该 MenuItem 被选中」驱动。
- 往上若是 **Submenu** 类型（如 `外观 › 衣服 › 睡衣`），Submenu **不产生条件**。
- 想让组件「父级菜单打开时才生效」**不能靠 Submenu 选中态**，得用 OT/动画开关它所在 GO（回到 1.2 条件 ①）。

【文档未说明，需实测】多层嵌套时"第一个"的确切取法；同一 GO 上 MenuItem 是 Submenu 时与反应式组件并存的行为。

### 1.4 烘焙后的三类驱动源

官方：**构建之后**，反应式组件响应 ①**改变 GameObject 状态的动画**；②**影响其它反应式组件 active 状态的 Object Toggle**；③**Menu Item 选中**。

要点：反应式组件**运行时可互相驱动**（第 ② 条，这是"脱外衣→自动还原素体"的实现方式）；**普通动画（含 MergeAnimator 层）也能驱动**反应式组件；限定"构建之后"——编辑器里不进 Play 不参与运行（个别组件的编辑期特例见 §3、§6）。

### 1.5 Invert condition 的正确语义

官方：勾选 Invert condition 后，效果在**上述任一条件不成立时**施加。

**它取反的是"是否生效的条件"，不是被控制的值。** 对 OT：`Invert` ≠ 目标可见性翻转，而是「条件不成立时才执行该 toggle 规则」。条件有多条时（父级 active × MenuItem 选中）是整个条件式的**否定**（德摩根：任一不成立即生效）——"父级被关掉"也会让勾了 Invert 的组件生效，换装嵌套里极易出错。

### 1.6 优先级规则（冲突谁赢）

| 页 | 官方规则 | 结论 |
|---|---|---|
| 总则 | the component **lowest in the hierarchy** takes precedence | 多个反应式同时激活且效果冲突 → **层级最低者胜** |
| Object Toggle | the Object Toggle that appears **last in hierarchy order** will take precedence | 多个 OT 抢同一目标 → **层级顺序最后者优先** |
| Shape Changer | the **lowest one in the hierarchy** will **generally** win | 多个 ShapeChanger 动同一形态键 → **通常**层级最低者胜 |

「层级最低」与「层级顺序最后」在 Hierarchy 面板里是同一件事（由上往下，越靠下越靠后）。**这是确定性规则，不是随机**；反过来也意味着**组件树整理（移动/改名/打包 prefab）会静默改变结果**。需要确定的互斥行为时，正解是**显式动画层**（自建 FX 层写 `m_IsActive`），而不是让多个 OT 抢同一对象。KB 里「按烘焙顺序、难预测」的说法见 §9 待复核 2。

### 1.7 反应时序

**官方 warning**：反应式的**精确时序会随实现优化而变，不要为复杂效果依赖精确时序**。三条机制：

- **一帧延迟**：响应 GO active 变化，**一帧之后**生效。
- **停用延迟一帧**：GO **正在被停用**时，停用**推迟一帧**，与目标更新同帧；官方给的原因：**关外衣时外衣继续显示，直到内层重新启用**，避免换装瞬间裸体一帧。
- **链式延迟**：反应式控反应式，**每级各延迟一帧**。A→B→C，A 被关：

| 帧 | 发生什么 |
|---|---|
| Frame 1 | 什么都不发生（A 的停用被延迟） |
| Frame 2 | A 被停用（B 的停用被延迟） |
| Frame 3 | B 和 C 同时被停用 |

**排障含义**：自动化验证/截图**差一帧就可能拿到相反结果**（改完参数要等帧）；换装"闪一下"多半是这条延迟；链式（OT→OT→ShapeChanger）累积 2~3 帧，逐帧断言要把常数算进去。

### 1.8 调试入口

右键 GameObject → `Modular Avatar → Show Reaction Debugger`；或在反应式组件上点 `Open reaction debugger`。操作法见 §8。

### 1.9 与 MenuItem / MergeAnimator / 手写层的关系

- **MenuItem** = **条件源**：挂它上面/子级 → 选中即生效（§1.3）。
- **MergeAnimator / 任意动画**：能改 GO 的 active 状态就能驱动反应式（§1.4 ①）。
- **反应式 → 反应式**：OT 可驱动另一反应式，代价每级一帧（§1.4 ② / §1.7）。
- **与手写 FX 层、AAO 等烘焙后处理器**：官方未说明谁先谁后（形态键改名、NaN 骨）【文档未说明，需实测】。

---

## §2 Object Toggle

出处：<https://modular-avatar.nadena.dev/docs/reference/reaction/object-toggle>

**定位**：按**控制对象**的 active 状态，改变**一个或多个其它 GameObject** 的 active 状态。

### 2.1 字段表

| 字段（文档描述） | 取值 | 默认值 | 语义 |
|---|---|---|---|
| 目标对象列表（点 `+` 添加） | GameObject 引用，1..N 条 | 空 | 被控制的 GameObject |
| 目标行的勾选框 | 勾 / 不勾 | 【文档未说明，需实测】 | 勾 = 目标 **enabled**；不勾 = **disabled** |
| `Invert condition` | 勾 / 不勾 | 不勾 | 条件取反（§1.5） |

> inspector 实际标签官方未给（原文只说 "click the + and select a target object"）。KB 反射实测结构为 `ToggledObject`（含 `Object`/`Active`），属 KB 结论、非文档事实。

### 2.2 机制与限制

- **冲突**：多个 OT 同时激活并控制**同一目标** → **层级顺序最后出现的**优先。
- **全部不激活**：所有控制该目标的 OT 都不激活 → 用该对象**原始状态**；若有**其它动画在操作它** → 用**动画给出的状态**。
- **时序**：控制对象更新后**一帧**更新目标；OT 被停用时其对象**晚一帧**关（§1.7）。
- **链式**：OT 控另一 OT 时延迟**逐级各加一帧**（F1 无 / F2 A / F3 B+C）。

### 2.3 典型用途

官方场景：**一个网格被另一个网格完全遮住时把它关掉**（如内衣被外衣全遮 → 禁用内衣网格），本身就是**性能优化**。与 Mesh Cutter 的对比见 §7.1。

### 2.4 常见坑

1. **用 Invert 做「默认可见、参数=1 隐藏」**：Invert 取反的是条件（§1.5），必须配合目标对象**场景初始 active 状态**与该行勾选框一起设计，否则出现"两个状态都可见/都隐藏"。
2. **挂在 inactive 宿主下** → 整组静默失效（§1.2）。
3. **目标同时被 OT 与动画操作** → OT 全不激活时回落到动画状态，易误判"OT 没生效"（§2.2）。
4. **靠多个 OT 抢同一对象做互斥** → 结果由层级顺序决定（§1.6），工程整理后会变；正解是显式动画层（§9）。
5. **行内目标路径为空** → KB 实测 `referencePath` 未写入时该行被**静默丢弃**（KB 结论，非文档）。

---

## §3 Shape Changer

出处：<https://modular-avatar.nadena.dev/docs/reference/reaction/shape-changer>

**定位**：Shape Changer **启用**时修改**另一个 renderer** 的形态键；典型用途是在套装网格上**删掉或缩小**被套装遮住/干涉的素体部位。

### 3.1 字段表

| 字段 | 取值 | 默认值 | 语义 |
|---|---|---|---|
| `Target Renderer` | 任意 renderer 引用（通常素体 SMR） | 空 | 被调整形态键的 renderer（**不是**挂组件的这个） |
| 形态键条目（点 `+` 开选择窗，点名字加入） | 每项 = 一个 blendshape + 模式 +（Set 值） | 空 | 操作哪些形态键、怎么操作 |
| 每条的模式 | `Delete` / `Set` | 【文档未说明，需实测】 | 见 3.2 |
| `Set` 模式的值 | float（形态键权重） | 【文档未说明，需实测】 | 激活时设为该值 |
| `Threshold` | float | 【文档未说明，需实测】 | 判定"哪些顶点受该形态键影响"；**调小 → 更多顶点** |

挂载位置：官方说挂到 outfit 里的一个对象，**通常是一个 skinned mesh renderer**；挂在哪条件就从哪来（§1.2）。

### 3.2 两种模式：Delete 与 Set（KB 之前缺的核心）

- **Set**：激活时把该形态键**设成指定值**。
- **Delete**：**删除被该形态键影响的多边形**。该形态键**不被动画驱动** → **真正降低多边形数**（性能收益）；**被动画驱动** → 仍**完全隐藏**这些多边形，但**不降低实测多边形数**（拿不到收益）。
- 官方建议：**当形态键会把某部位缩到（几乎）零时优先用 `Delete`**。想靠它减面/压 rank，前提就是"该形态键不参与任何动画"，否则退化成纯隐藏（详见 3.7）。

### 3.3 Threshold 的调整方向（容易记反）

官方：**切得不够（没删够网格）时把 threshold 调小，就会有更多顶点受影响。** → **切得不够 → 调小 → 更多顶点**。

> ⚠️ 与 Mesh Cutter 的 `By Blendshape` 过滤器**方向相反**（那里"更小 = 更挑 = 更少"，§7.5）。同名 Threshold 语义相反，是全文档最易踩的对照。

### 3.4 官方给的两个工作法

1. **场景视图实时预览**：官方明确**Shape Changer 的效果会在 scene view 实时显示** —— 调参数直接在 Scene 看，不必进 Play。
2. **用 scene view 的 `overdraw` 调试叠加**：被删部位通常藏在衣服下，开 `overdraw` 可**透视衣服**观察底下。

（KB「编辑器场景视图不生效」的说法对这两个组件过宽，见 §9 待复核 1。）

### 3.5 与动画的交互

- 响应**它自己所在 GO** 的动画，也响应**其父级**的动画。
- 停用这些 GO 会**移除**它对目标 renderer 的影响 → 可用于换装、或脱掉某件衣服时**还原基础网格**。
- **不要**用它修改「同时被其它动画驱动」的形态键 —— 官方明确应改为**动画开关那个包含 Shape Changer 的 GO**。

### 3.6 多组件冲突

多个 Shape Changer 同时操作同一形态键 → **层级最低的通常胜出**（原文留了 "generally" 余量）。

### 3.7 性能影响

| 情况 | 结果 |
|---|---|
| `Delete` + 形态键**无**动画 | 烘焙期真删多边形 → **减面，可能改善 performance rank** |
| `Delete` + 形态键**有**动画 | 完全隐藏，但**面数不变**（无性能收益） |
| `Set` | 不删面，只重写权重 |

【文档未说明，需实测】是否在烘焙期生成额外约束/驱动，及其对 rank 的量化影响。

### 3.8 常见坑

1. **以为 Delete 一定减面** → 须确认目标形态键无动画驱动（3.2）。
2. **Threshold 方向记反** → 切不够要**调小**（3.3）。
3. **跟表情/口型动画抢同一形态键** → 官方明令不要（3.5），表现为表情不动或衣服破面。
4. **挂在 inactive 的衣服上**（衣服默认态关）→ 组件不生效（§1.2）。
5. **与 AAO 同用**：KB 实测出现 NaN 骨导致身体成块消失（§9）。官方文档未涉及 AAO 交互。

---

## §4 Material Setter

出处：<https://modular-avatar.nadena.dev/docs/reference/reaction/material-setter>

**定位**：所在 GameObject **启用**时，**按 renderer 指定**地改变某 renderer 的材质（哪个槽换成哪个材质）。

### 4.1 与 Material Swap 的区别（一句话记法）

官方：**Material Swap 里你指定「要换的材质」，Material Setter 里你指定「材质被改的 renderer」。**

| 组件 | 你指定什么 | 影响范围 |
|---|---|---|
| **Material Setter** | **renderer + 材质槽** → 换成哪个材质 | 只影响点名的那个 renderer 的那一个槽 |
| **Material Swap** | **材质对**（源 → 目标） | 按材质引用批量匹配（§5） |

### 4.2 字段表

| 字段 | 取值 | 默认值 | 语义 |
|---|---|---|---|
| 条目列表（点 `+`） | 1..N 条 | 空 | 每条 = 一处替换 |
| 条目内 renderer 对象格 | renderer 引用 | 空 | 要改的 renderer（拖进来） |
| 条目内材质槽下拉 | 该 renderer 的槽序号（`element 0/1/…`） | 第 0 槽 | 要改哪个槽 |
| 条目内 `Set material to` | 材质引用 | 空 | 换成的目标材质 |
| `Inverse condition` | 勾 / 不勾 | 不勾 | 勾选后改为**停用时**换材质 |

### 4.3 挂载位置（官方明列三种）

①一个**会被动画开关**的 GameObject；②**Menu Item 上或其子级**（菜单选中驱动，§1.3）；③一个**始终启用**的 GameObject → **永久替换**该对象材质（"any time" 场景）。默认行为：**GameObject 启用时（和/或关联 Menu Item 被选中时）**换；勾 `Inverse condition` 改为**停用时**换。

### 4.4 性能影响

【文档未说明，需实测】Setter 不新建材质资产，但把材质引用写死在烘焙产物里；是否增加材质槽数/draw call 需用渲染统计确认。

### 4.5 常见坑

1. **想批量换色却用 Setter** → 得一个 renderer 一个 renderer 加条目；批量换色用 Material Swap（§5）。
2. **材质槽选错**（多材质 renderer 的第 N 槽）→ 换错部分。
3. **只想换贴图**：Setter 换的是**整个材质球**，不能只换贴图参数（KB 实测，§9）。
4. **挂在始终启用的对象上却以为受参数控制** → 那是永久替换（4.3 ③）。

---

## §5 Material Swap

出处：<https://modular-avatar.nadena.dev/docs/reference/reaction/material-swap>

**定位**：所在 GameObject **启用**时，**按材质对批量**把头像上的材质换成另外的材质。

### 5.1 字段表

| 字段 | 取值 | 默认值 | 语义 |
|---|---|---|---|
| 条目列表（点 `+`） | 1..N 条 | 空 | 每条 = 一对「源材质 → 目标材质」 |
| 左侧材质格 | 材质引用 | 空 | **要被替换掉的**（源） |
| 右侧材质格 | 材质引用 | 空 | **要换成的**（目标） |
| `Target Root` | GameObject 引用 | 空（= 全头像） | 把本次 swap 的**作用范围限制到某层级** |
| `Inverse condition` | 勾 / 不勾 | 不勾 | 勾选后改为**停用时**换 |
| `Quick Swap` | `None` / `Same Folder` / `Adjacent Folders` | 【文档未说明，需实测】 | 见 5.2 |

机制：按**材质引用**匹配 → 头像上所有用该源材质处一起换（"批量"）；`Target Root` 把作用域限制到指定层级，避免误伤层级外共用同一材质的部件。挂载位置同 Setter（§4.3）。

### 5.2 Quick Swap（编辑期预览辅助）

- `None` 禁用；`Same Folder` 在当前**替换后材质所在目录**找；`Adjacent Folders` 在**相邻目录**找**名字相似**的材质。官方例：`Assets\SomeOutfit\Materials\Blue\Outer.mat` → 可能找 `...\Red\Outer.mat`，适合同套装多部件、按颜色分目录的组织。不确定用哪个 → **两个都试**。
- 开启后 inspector 里**目标材质旁出现两个箭头**，可前后切换候选即时预览。它是**编辑期预览辅助**，是否改变烘焙结果官方未说明。

### 5.3 性能影响

一整套衣服换色只由**一个** Material Swap 完成，比给每个部件挂 Setter **少很多组件与动画层**。【文档未说明，需实测】换材质是否打断 batching / 增加 draw call。

### 5.4 常见坑

1. **误伤**：同一材质被多部件共用 → 一换全换；用 `Target Root` 或改用 Setter（§4）。
2. **左右拖反**：文档明确「左 = 源，右 = 目标」；拖反表现为"没变化"或"变成不相关材质"。
3. **`Adjacent Folders` 找不到预期材质**：它按"名字相似 + 相邻目录"猜，目录不规范就不准。

---

## §6 Blendshape Sync

出处：<https://modular-avatar.nadena.dev/docs/reference/blendshape-sync>（**不在 `reaction/` 下**）

**定位**：让**一个 renderer 上的某形态键**始终**匹配另一个 renderer** 上的对应形态键。

### 6.1 适用与不适用（两条硬限制）

**适用**：给「身体有体型形态键」的头像做/用服装而服装本身有对应 shapekey；基础头像内部也可同步不同对象。

**不适用**：①**不能链式（cannot be chained）**：`A→B`、`A→C` 可以，`A→B` 再 `B→C` **不行**（文档两处重复）；②**运行时只支持由 animator 控制的形态键**；**由 VRChat 内置 eyelook 或 viseme 系统控制的形态键无法被准确同步**（cannot be synced accurately）。

含义：想同步**眨眼/口型（viseme）/眼追**相关形态键 → 做不到准确同步；同步**体型/表情类**（由自己的 animator 层驱动）→ 可以；换装链式同步 → 不行，改成并联（身体→每件各自）。

### 6.2 字段与操作

| 项 | 操作 / 取值 | 默认值 | 语义 |
|---|---|---|---|
| 同步条目（点 `+` 开选择窗） | **双击**形态键 → 加入；加够点 `X` 关窗 | 空 | 要同步哪些形态键 |
| `Target blendshape` 下拉 | 目标对象上的形态键名 | 与源同名 | 源/目标**名字不同**时手动指定 |
| `Remap` 曲线 | AnimationCurve | 【文档未说明，需实测】 | 调整目标对源的响应，可反相（6.3） |
| 多选批量编辑 | 选中多个网格 | — | 一次配置多个 |

### 6.3 Remap 曲线的正确用法

官方：**用 Remap 曲线调整目标形态键对源的响应**（例：让目标相对源**反相**）；**与动画曲线一样，必须给这条曲线设置合适的切线**才能得到预期效果。

Remap 是一维映射曲线：输入 = 源权重，输出 = 目标权重；**可做反相**；**必须设合适切线**，否则默认切线会让曲线过冲/提前饱和。【文档未说明，需实测】取值域/归一化方式（0–1 还是 0–100）。

### 6.4 它到底做了什么（两阶段机制）

官方（What does it do, exactly?）：①**编辑模式**：自动**从 base 复制形态键值**，对 base 的修改**立即**反映到其它对象；②**Play 模式**：**改写所有动画** —— 动画里驱动 base 形态键的曲线会被**同时写到其它对象**上。

**解释三个现象**：①编辑期就能看到同步生效（不必进 Play）；②烘焙后动画曲线数变多（第 ② 条 = 烘焙期重写曲线）；③KB「MA 组件在编辑器不生效」的说法对 Blendshape Sync 不成立（§9）。

### 6.5 性能影响

【文档未说明，需实测】同步会为每个目标对象**复制动画曲线**，曲线总数上升可能影响 rank 相关项；目标越多、参与形态键越多，曲线越多。

### 6.6 常见坑

1. **拿去同步 viseme/眨眼/eyelook** → 官方明说无法准确同步（6.1）。
2. **搭成链式 A→B→C** → 不生效且**不报错**（6.1）。
3. **Remap 没调切线** → 曲线过冲，形态键中段就饱和/反向（6.3）。
4. **源/目标不同名却没设 `Target blendshape`** → 静默不同步。
5. 目标对象本身被禁用时会怎样：【文档未说明，需实测】。

---

## §7 Mesh Cutter

出处：主体 <https://modular-avatar.nadena.dev/docs/reference/reaction/mesh-cutter>；5 个过滤器见 7.2–7.6。

### 7.1 主体

**定位**：按**一个或多个 Vertex Filter** 的选择结果，**删除或隐藏**网格的局部顶点/多边形。

**用途（官方三条）**：①去掉**穿透外层衣物**的底层身体网格部分；②**美术目的**删局部（例：只把丝带一边缩短）；③删掉网格不必要部分以**降低多边形数**。

**不该用**：只是要给「被另一个网格完全覆盖」的网格关掉 → **用 Object Toggle 更高效**，因为 **Mesh Cutter 即使整块被遮住，仍要处理整张网格**。

**设置步骤**：①添加 Mesh Cutter，把要改的对象设进 **`Object` 字段**；②**必须**再添**至少一个 Vertex Filter**（点 `Add Vertex Filter` 或手动加）；③激活时删除/隐藏；**勾 `Invert` → 改为不激活时删**。

| 字段 | 取值 | 默认值 | 语义 |
|---|---|---|---|
| `Object`（By Mask / By Blendshape 两页写作 **"Target Renderer field on the Mesh Cutter component"**） | GameObject / Mesh 引用 | 空 | 要修改的网格对象 |
| `Invert` | 勾 / 不勾 | 不勾 | 勾选 = **不激活时**删除 |
| Vertex Filter 列表 | 0..N 个 `Vertex Filter - By *` | 0 | **必须 ≥1**，否则无效果 |

> ⚠️ 字段名不一致：主体页叫 `Object`，By Mask / By Blendshape 页叫 `Target Renderer`，官方文档内部两处叫法不统一，实际 inspector 标签以已装包为准。【文档未说明，需实测】

### 7.2 Vertex Filter - By Mask

出处：<https://modular-avatar.nadena.dev/docs/reference/reaction/mesh-cutter/by-mask>

**定位**：用**遮罩贴图（mask texture）**选择网格的一部分。前提：必须挂在带 Mesh Cutter 的 GO 上；先确认 Mesh Cutter 的 `Target Renderer`（= `Object`）已设为目标，**再选要删多边形的 `Material Slot`**。

| 字段 | 取值 | 默认值 | 语义 |
|---|---|---|---|
| `Material Slot` | 该 renderer 的材质槽序号 | 第 0 槽 | **从哪个材质槽删多边形** |
| `Mask` | Texture 引用 | 空 | 白/黑区域决定选中范围 |
| `Delete Mode` | 白区 / 黑区 | 【文档未说明，需实测】 | 选**删遮罩的白色区还是黑色区** |

**官方 note（硬限制）**：**单个 By Mask 只能从一个材质槽选多边形**；要删多个槽 → **建多个 GameObject，各带自己的 Mesh Cutter 与 Vertex Filter**。

**遮罩制作**：可放**预制贴图**；若装了 **Mask Texture Editor** 扩展，点 `Edit` 可**直接在 Unity 编辑器里创建/编辑**遮罩。

**坑**：多槽只挂一个 By Mask（只生效一个槽）；`Delete Mode` 选反（删掉保留区）；`Material Slot` 与遮罩对不上。

### 7.3 Vertex Filter - By Axis

出处：<https://modular-avatar.nadena.dev/docs/reference/reaction/mesh-cutter/by-axis>

**定位**：按「**是否位于某平面的一侧**」选择。用于网格**左右共用同一套 UV**、By Mask 选不出单侧的情况（By Mask 选整条 + By Axis 限一侧）。

**坐标系（关键）**：官方明确 **`Center` 与 `Axis` 按 renderer 的局部空间解释**，不是世界空间、不是头像根。

| 字段 | 取值 | 默认值 | 语义 |
|---|---|---|---|
| `Center` | Vector3（renderer 局部空间） | 【文档未说明，需实测】 | 平面上的一点 |
| `Axis` | Vector3（renderer 局部空间） | 【文档未说明，需实测】 | 平面法线方向 |
| `Edit`（按钮） | — | — | 显示 gizmo 调平面 |

**gizmo**：点 `Edit` 调平面**位置与朝向**；平面**无限**，把空间分两半，一侧被选；**白色箭头 gizmo 指示"将被擦除的那一侧"**。

**分发资产注意（官方单列一节）**：终端用户可能用 **Scale Adjuster** 改体型 → 网格形状变化 → **被选区域随之变化**。所以 **By Axis 只适合粗选**（如只分左右），**精选用 By Mask** 等其它过滤器。

**坑**：按世界坐标心思填 `Center`/`Axis`（应按 renderer 局部空间）；拿它做精确选区（用户换体型后错位）；没看**白箭头**方向导致删反。

### 7.4 Vertex Filter - By Bone

出处：<https://modular-avatar.nadena.dev/docs/reference/reaction/mesh-cutter/by-bone>

**定位**：按「**顶点对某根骨骼的权重**」选择网格的一部分。

| 字段 | 取值 | 默认值 | 语义 |
|---|---|---|---|
| `Target Bone` | Transform（骨骼） | 空 | 按哪根骨过滤；**只有对该骨有权重的顶点才被考虑** |
| `Weight Threshold` | **0.0 – 1.0** | 【文档未说明，需实测】 | 该骨权重**最小值**；**权重 ≥ 阈值**的顶点被选中 |

**关键机制**：官方明确**比较阈值前骨权重会先归一化** → 填的是**归一化后的占比**，不是原始权重数值。

| 阈值 | 含义（官方三档） |
|---|---|
| `0.01` | 只要有任何**有意义的权重** |
| `0.5` | 只选**主要由该骨控制**的顶点（primarily controlled） |
| `1.0` | 只选**完全由该骨控制**的顶点（completely controlled） |

**硬限制（官方 Note）**：**只对带骨权重数据的 SkinnedMeshRenderer 有效**；对**普通 MeshRenderer 或没有骨权重的网格无效**。

**坑**：对静态 MeshRenderer 用（无效）；把阈值当"原始权重"（实为归一化）；【文档未说明，需实测】多骨混合时的归一化分母口径。

### 7.5 Vertex Filter - By Blendshape

出处：<https://modular-avatar.nadena.dev/docs/reference/reaction/mesh-cutter/by-shape>

**定位**：按「**某形态键激活时，顶点是否移动**」选择网格的一部分。**典型用途（官方三条）**：①去掉「表情改脸型时**穿过衣服**露出」的身体部位；②隐藏「头部形态键激活时**穿过帽子**」的头发；③删掉「会**干扰形态键动画**」的配件。

| 字段 | 取值 | 默认值 | 语义 |
|---|---|---|---|
| 形态键列表（点 `+` 从该网格形态键里选） | 1..N 个 blendshape | 空 | **选多个 = 并集**（见下） |
| `Threshold` | float（**局部空间**距离） | **`0.001`** | 形态键**完全激活**时顶点**移动距离超过该值**才被过滤 |

**多形态键 = 并集**：官方明确选多个形态键时，**任意一个**激活时会动的顶点都被选（并集，不是交集）。

**Threshold 方向（与 Shape Changer 相反！）**：官方明确**形态键完全激活时移动距离超过该值（局部空间）的顶点被过滤**，**默认 `0.001`** 对多数情况合适；tip：**调小 = 更挑**（更少顶点被过滤），**调大 = 更激进**（更多顶点被过滤）。

> ⚠️ 与 Shape Changer 的 `Threshold` **方向相反**：ShapeChanger「切不够→调小→更多顶点」；By Blendshape「调小→更挑→更少顶点」。

**坑**：两个 Threshold 方向记反；以为多形态键是**交集**（实为并集，会切掉超预期区域）；距离是**局部空间**、网格缩放后阈值含义随之变。

### 7.6 Vertex Filter - By UV Tile

出处：<https://modular-avatar.nadena.dev/docs/reference/reaction/mesh-cutter/by-uv-tile>

**定位**：按「**UV 坐标是否落在指定矩形区域内**」选择网格的一部分。**用途（官方三条）**：①在 **UV tile / UDIM** 布局里选**单个 tile**；②删「占据 UV 图某特定区域」的部分；③与其它过滤器以 **intersect 组合**时**只取可见区域**。

**不该用**：官方明确若 UV **重复**（如网格左右**镜像 UV**），By UV Tile 会**同时选中左右两块** → 这种情况改用 **By Axis** 区分两侧。

**配置结构**：U 与 V 各设 **lower / upper** 两个 bound，**每个 bound 三个控件**：`Enable` 开关（**关掉则该 bound 被忽略** → 可只按 U、只按 V）、`Operator`（严格 `<` = **排除**恰在边界值的顶点；含等 `<=` = **包含**）、`Value`（该 bound 的 UV 坐标值，float）。**左 bound = 最小值**（小于该值被排除）；**右 bound = 最大值**（大于该值被排除）。三个默认值均【文档未说明，需实测】。

| 字段 | 取值 | 默认值 | 语义 |
|---|---|---|---|
| `UV Channel` | **0 – 7** | **`0`**（第一套 UV = 主贴图 UV） | 用哪套 UV 过滤 |
| `Selection Mode` | `Any Vertex` / `All Vertices` / `Centroid` | 【文档未说明，需实测】 | 顶点选择如何转成**图元（三角形）**选择 |
| `Invert` | 勾 / 不勾 | 不勾 | 勾选 = 选**矩形之外**的顶点 |

| `Selection Mode` | 语义 |
|---|---|
| **`Any Vertex`** | 图元**至少一个顶点**在矩形内 → 选中该图元 |
| **`All Vertices`** | 图元**全部顶点**都在矩形内 → 才选中 |
| **`Centroid`** | 按图元**中心点 UV** 判定；官方称它对 UV 过滤**最可预测**，**tile 式选择尤其好用** |

**多 UV 通道**：网格有多套 UV 时用 `UV Channel` 选依据哪套；**默认 0 = 主贴图坐标**。

【文档未说明，需实测】原文档 "How it works" 的矩形判定公式在抓取文本里丢失（只剩 `UMin` 截断段）；已确认每个 bound 的边界判定由它自己的 `Operator` 决定、可混用，边界值 `<` 还是 `<=` 以实测为准。

**坑**：对镜像 UV 用 By UV Tile（左右同删，官方明令改 By Axis）；tile 级选择用 `Any Vertex`（边缘三角形被误选，官方推荐 `Centroid`）；把 `Invert` 与 By Mask 的 `Delete Mode` 混为一谈（前者翻选区，后者选删黑还是白）；多 UV 通道忘改 `UV Channel`。

### 7.7 多过滤器：combine 与 intersect

官方：用多个 Vertex Filter 时，可以 **combine**（把多个过滤器的选区**合并**），也可以 **intersect**（**只保留同时匹配所有过滤器**的顶点）。

| 模式 | 语义 | 典型用法 |
|---|---|---|
| **combine（并集）** | 各过滤器结果**合并** | 多个不同部位各切一块 |
| **intersect（交集）** | **只保留所有过滤器都匹配**的顶点 | 「By Mask 选整条丝带」∩「By Axis 限左侧」 |

**官方示例 1（左右共用 UV）**：网格左右复用同一套 UV → 用 `By Mask` 选一部分，再用 `By Axis` 限制到只留一侧。

**官方示例 2（剪短丝带一侧）**：丝带两侧共用同一套贴图坐标，单个 By Mask 选不出左侧 → By Mask 选整条 + By Axis 选左侧 + 另一个 By Axis 控制剪短量，三条**交集**得到精确选区。

【文档未说明，需实测】默认是 combine 还是 intersect；切换模式的 UI 在 Mesh Cutter 上还是在每个过滤器上。

### 7.8 效率与 performance rank

官方（Efficiency notes）：**Mesh Cutter 始终激活** → MA **彻底删除**选中多边形，可用于**降低多边形数**以压 **performance rank**；**有时会不激活** → 改为**隐藏**选中多边形，**某些情况下需要生成约束**，可能**抬高 performance rank**。

| 使用方式 | 烘焙行为 | 对 performance rank 的影响 |
|---|---|---|
| Mesh Cutter **始终激活**（不受开关/动画影响） | **真删**选中多边形 | **正面**：减面，可压到更低档 |
| Mesh Cutter **有时会不激活**（被菜单/动画开关） | 改**隐藏**；某些情况还要**生成约束** | **可能负面**：约束增加开销，抬高 rank |

**设计建议**：想拿性能收益 → 让 Mesh Cutter **一直挂着不关**（如"永远去掉被外衣盖住的身体部分"）；需要"可开关的切割" → 接受它只是**隐藏**，别指望减面并注意约束开销；只是"整块网格被盖住" → 用 **Object Toggle**（§2）更省。

### 7.9 常见坑

| # | 现象 | 原因 | 处理 |
|---|---|---|---|
| 1 | 加了 Mesh Cutter 完全没效果 | 没有 Vertex Filter（必须 ≥1）/ `Object` 没设 | 按 7.1 补 |
| 2 | 该删时没删、不该删时删了 | `Invert` 勾反 | 7.1 字段表 |
| 3 | 只删了一个材质槽 | By Mask 单槽限制 | 多建 GO，各带 MC + Filter（7.2） |
| 4 | 切出形状差很远 / 用户换体型后切歪 | 组合模式搞反；By Axis 依赖网格形状（Scale Adjuster） | 7.7 / 7.3 |
| 5 | 左右镜像 UV 两边一起被删 | 重复 UV 无法区分 | 改用 By Axis（7.6） |
| 6 | 以为减面了但 rank 没降 / 反而变高 | 有时不激活→只隐藏；可能生成约束 | 7.8；要么常开，要么别指望 |

---

## §8 Reaction Debugger

出处：<https://modular-avatar.nadena.dev/docs/reference/reaction/debugger>

**定位**：**虚拟地**改变菜单项与 GameObject 状态，从而**不实际操作头像**就能测试反应式组件会怎么表现。

**打开方式**：①右键 GameObject → `Modular Avatar → Show Reaction Debugger`；②在任意反应式组件上点 **`Open reaction debugger`** 按钮。

**窗口两段**：**对象状态段**（显示/强制该对象自身状态）+ **反应段**（显示所有影响该对象的 **"incoming" reactions** 及其条件，并可 override）。

| 控件 | 说明 |
|---|---|
| `Inspecting object` 字段 | 选择**要检查哪个对象**；默认 = 当前选中对象，可改成别的 |
| （lock） | 像 inspector 一样**锁定视图**，避免在场景里选别的对象时跟着变 |
| `Clear all active overrides` 按钮 | **清除所有已生效的 override**（强制状态全部复位） |
| `Object state: ACTIVE` | 显示该对象（虚拟地）是 active 还是 inactive |
| 强制对象 active / inactive | 两组 `-` / `+` 按钮：点 `-` 或 `+` 触发强制状态，点**中间空白按钮**清除该 override |
| 强制关联 MenuItem 选中 / 未选中 | 同上，针对「关联菜单项」的选中状态 |

**关键性质**：官方明确**强制 active/inactive 并不会真的改它在层级里的状态**，只是在 **scene view 里显示成启用/禁用** → 强制状态**不落地**，可放心试、不会污染场景。

**建议排查流程**：①选中"没按预期生效"的组件所在对象 → 开 debugger；②看**反应段**有没有列出你期望的 reaction —— **没有** = 组件不在链路里（inactive 宿主 §1.2 / 菜单归属错 §1.3），**有** = 逐个 override 它要求的条件；③看**对象状态段**确认目标 ACTIVE/INACTIVE；④多组件抢同一对象 → 逐个 override 找**谁**在起作用（§1.6）；⑤时序问题回 §1.7（debugger 只展示**稳态**）。

---

## §9 与本地 KB 的两处口径冲突（需人工定稿）

> 以下两条是**审计发现的冲突**（非官方文档事实），KB 原文位置已给，落地时一并修改。

1. **「编辑器里不生效」是否过宽？** KB `SKILL.md` 与 `references/00-基底/00-05-mesh-blendshape.md` 称 `ShapeChanger`/`MaterialSetter`/`ObjectToggle`/`BlendshapeSync`/`MergeAnimator` **在普通编辑器场景视图里不生效**。官方：**Shape Changer 的效果在 scene view 实时显示**（§3.4）、**Blendshape Sync 在编辑模式自动从 base 复制值并立即生效**（§6.4）。建议分组件表述：OT / MaterialSetter（及 MergeAnimator 的烘焙产物）确实要烘焙或进 Play；ShapeChanger、BlendshapeSync 不能一概而论。
2. **「多个组件抢同一对象 → 按烘焙顺序、难预测」**（`03-菜单与参数/03-06-ma-menu-build-by-reflection.md`）：官方有确定性规则（**层级最低者胜 / OT 层级最后者优先**，§1.6）。建议保留"实测会有差异"的提醒，但把官方规则写成**期望行为**。

---

## §10 建议落地（KB 文件 / 新建分册 / 优先级）

> **本稿不改动 KB 任何文件**（KB 目录只读）。以下是给 KB 维护者的建议。

**P0 新建分册（已建）**：`references/04-外观与表情/04-06-ma-reactive-components.md`（即本稿主体；`00-基底/` 是外部 MIT 原件，一字不改，故落在 04）。内容跨 **5 个反应式组件 + 5 个 Vertex Filter + 1 个调试器**，塞进任何现有分册都会撑爆其主题；`00-基底/00-02-modular-avatar.md` 是"代码范式/反射"册，不适合放字段级语义。已在 `references/README.md` 登记（全包 **54 → 58 册**），并写明补的是哪个缺口。

**P0 现有分册补句**：`02-工程组织与工作流/02-03-ma-component-guide-workflows.md` —— §9 补**优先级 / 一帧时序与停用延迟 / 调试器入口**；§10 补 `Delete` vs `Set` + **Threshold 方向**；§11 补**By UV Tile / combine vs intersect / `Object`·`Invert` / 常开才减面**；§8 补**viseme·eyelook 同步不了 / 编辑期复制 + Play 期改写动画 / `Remap`**；§13 补**Setter vs Swap / `Target Root` / `Quick Swap`**。

**P1 机制层补强**：`00-基底/00-05-mesh-blendshape.md` 增「ShapeChanger 的 Delete vs Set（面数/性能条件）」「BlendshapeSync 的运行期限制 + 编辑/Play 两阶段 + Remap 切线」两节，并修正"编辑器不生效"的宽泛表述；`07-验证与排查/07-01-bake-and-validation.md` 增「反应式产物审计」（**验收断言不可单帧**、冲突期望=层级最低者胜、**Mesh Cutter 只有常开才减面**）；`SKILL.md` 症状速查 +3~4 行：换装瞬间闪裸体→停用延迟；同对象多开关结果不定→层级优先级；Mesh Cutter 切得不对→**两个 Threshold 方向相反**；BlendshapeSync 同步不了 viseme/眨眼→官方限制。

**P2 字段级实测**：`07-验证与排查/07-05-ma-gotchas-version-toolchain.md §2` 目前只覆盖 MenuItem / ObjectToggle / ShapeChanger，建议补 **Mesh Cutter、5 个 Vertex Filter、MaterialSetter、MaterialSwap、BlendshapeSync** 的 inspector 实际标签与序列化字段（**以已装包源码为准**——官方文档里 `Object` 与 `Target Renderer` 就存在两处叫法不一致）；`03-06 §14.2` 的"行为按烘焙顺序、难预测"改为官方规则 + 实测备注。

**仍需实测的 11 项**（各组件字段的 inspector 实际标签/默认值，官方未写）：①`Object` vs `Target Renderer`、OT 目标行与勾选框、ShapeChanger threshold；②ShapeChanger `Threshold` 默认值 + `Delete/Set` 默认模式；③OT 勾选框默认值；④By Axis `Center`/`Axis` 默认值；⑤By Bone `Weight Threshold` 默认值与多骨归一化口径；⑥By Mask `Delete Mode` 默认值；⑦By UV Tile 的 bound/Operator/Selection Mode 默认值（原文公式抓取时丢失）；⑧多过滤器默认组合模式与切换 UI 位置；⑨Blendshape Sync `Remap` 默认曲线/取值域、目标禁用时是否同步；⑩Material Swap `Quick Swap` 默认模式与 `Target Root` 匹配语义；⑪MA 生成层与手写 FX 层插入顺序、与 AAO 的交互。

---

## §11 官方出处清单

| # | 页面 | URL |
|---|---|---|
| 1 | Reactive Components | <https://modular-avatar.nadena.dev/docs/reference/reaction> |
| 2 | Object Toggle | <https://modular-avatar.nadena.dev/docs/reference/reaction/object-toggle> |
| 3 | Shape Changer | <https://modular-avatar.nadena.dev/docs/reference/reaction/shape-changer> |
| 4 | Material Setter | <https://modular-avatar.nadena.dev/docs/reference/reaction/material-setter> |
| 5 | Material Swap | <https://modular-avatar.nadena.dev/docs/reference/reaction/material-swap> |
| 6 | Blendshape Sync | <https://modular-avatar.nadena.dev/docs/reference/blendshape-sync> |
| 7 | Mesh Cutter | <https://modular-avatar.nadena.dev/docs/reference/reaction/mesh-cutter> |
| 8 | Vertex Filter - By Axis | <https://modular-avatar.nadena.dev/docs/reference/reaction/mesh-cutter/by-axis> |
| 9 | Vertex Filter - By Bone | <https://modular-avatar.nadena.dev/docs/reference/reaction/mesh-cutter/by-bone> |
| 10 | Vertex Filter - By Mask | <https://modular-avatar.nadena.dev/docs/reference/reaction/mesh-cutter/by-mask> |
| 11 | Vertex Filter - By Blendshape | <https://modular-avatar.nadena.dev/docs/reference/reaction/mesh-cutter/by-shape> |
| 12 | Vertex Filter - By UV Tile | <https://modular-avatar.nadena.dev/docs/reference/reaction/mesh-cutter/by-uv-tile> |
| 13 | Reaction Debugger | <https://modular-avatar.nadena.dev/docs/reference/reaction/debugger> |

> 本地抓取文本在 `<VRCHAT_ROOT>\tools\tmp\ma-docs-txt\`（`docs__reference__*.txt`），以本地文本为准；若 MA 版本已升级，建议对照新版文档复核 §10 末的 11 项待实测。


---

## §12 Object Toggle 的 Active 语义，与「默认态由 FX 动画决定」的关系（本工作区实测 2026-11-05，LuzzyMeow V2 沐梓溪狼 T1）

> 场景：睡衣与泳装2 默认同时可见。想靠改 OT 勾选修默认态 —— 改不动。

**判据**
- OT 每行 = 目标对象 + Active 勾选：勾 = 条件成立时 enabled，不勾 = 条件成立时 disabled。**只在条件成立时写值；条件不成立时不写。**
- 多个 OT 抢同一目标 → 官方「the Object Toggle that appears last in hierarchy order will take precedence」（层级最低者胜）。
- 全部控制该目标的 OT 都不激活 → 目标使用「对象原始状态（场景 activeSelf），或（若其它动画在操作它）动画给出的状态」。
- `Invert condition` 取反的是**条件是否成立**，不是目标的值 → 用 Invert 做「默认显示、参数=1 隐藏」必错。

**正解**：「默认穿哪套」不在 OT 上，在 FX 动画层 —— 部件可见性由 FX 层按 Bool 写 `m_IsActive`，OT 只是「菜单选中时的一次性覆盖」。要改默认态就改默认态 driver 与三处默认值（§1.5–§1.7、§2.2/§2.4）。

**坑**
- 生效前提 = 自己与所有父级 active +（若与 MenuItem 同 GO 或是其子级）该 MenuItem 被选中；挂在 inactive GO 下**整体静默失效、Console 不报错**（`references\07-验证与排查\07-01-bake-and-validation.md` §31 (L1108)）。
- 组件只认第一个父 MenuItem，父 Submenu 被忽略、不产生条件。
- 时序：目标更新滞后一帧；被停用对象的停用也推迟一帧（避免换装瞬间裸体一帧，不是 bug）。链式 A→B→C 逐级延迟。
- 要确定性互斥就用显式动画层，别靠多个 OT 抢同一目标——层级顺序会被「组件树整理」静默改变。
- `ToggledObject` 是 struct：反射改完要写回 `list[i]`，再把整个 list 写回组件（`references\02-工程组织与工作流\02-02-ma-workflow-habits.md` §19 (L349)）。

**适用场景**：修默认衣装重叠（T1）；判断「部件为什么默认显示/隐藏」；排查「删了菜单项但对象还在 / 还在消失」。

> 更新时间 2026-11-05（本次来源：沐梓溪狼衣装菜单重构 / LuzzyMeow V2 T1–T10）

## §13 反相／重定义 ObjectToggle 语义后，必须同步 FX driver 里对该参数的「常量写入」（本工作区实测 2026-11-05，沐梓溪狼 T1/T3 回归修复）

**症状**：衣装「一键穿衣」切换本身正常，但切到泳装1/泳装2、休闲装、原版衣服时，睡衣的「外套 + 口袋小幽灵」仍穿在身上；默认睡衣态正常。

**根因**：`睡衣/外套` 的 OT 语义被改成「参数=1 → 隐藏 `PJ_Outer` / `PJ_PocketGhost`」（Inverted 语义；EP `PJ_Outer` 默认 0 = 可见）；但烘焙 FX `OutfitSync` 层里**非睡衣各态 driver 仍写 `PJ_Outer=0`**（旧非反相语义下的「隐藏」常量），现在 0 表示「显示」→ 每套都穿外套。`Default` 态写 0 恰好正确；`Pajama` 态没有该条目（用户手动关外套后切回睡衣不会重置）。

**判据 / 复现**：
- GM 置 `OutfitOneClick=2/3/4/5` 时，`睡衣/PJ_Outer` 与 `睡衣/PJ_PocketGhost` 仍 `activeSelf=true`，可见组 `groups=2`（睡衣 + 目标套装）。
- 打印烘焙 FX：`LilLeo_FX_Wolf.controller` → layer `OutfitSync` → 各 state 的 `VRCAvatarParameterDriver.parameters`，`PJ_Outer` 值 = Default 0、Pajama 无、Swim/Carrot/Original/Swim2 均 0。

**正确做法**：
1. 用 edit-mode 脚本改**源控制器资产**（不要改烘焙产物）：定位 layer `OutfitSync` → 各 state → driver 的 `parameters` 列表，把 `PJ_Outer` 的 `value` 改成与 OT 语义一致：`Default`/`Pajama` = 0，其余套装态 = 1；`Pajama` 缺失条目则新建（`Activator.CreateInstance(elementType)` 后赋 name/value/type=Set）。
2. `EditorUtility.SetDirty(controller)` + `AssetDatabase.SaveAssets()`；重新进 Play 让 NDMF/VRCFury 重烘。
3. 验收：GM 置 `OC=1..5`，每态 `睡衣/` 组可见数 = 0（睡衣态除外）、`groups=1`；视觉上泳装/休闲装/原版无外套残留；默认态 10 视角无变化。

**通用教训**：OT 的语义（Active / Inverted）、EP 默认值、FX driver 常量是同一套布尔语义的**三处副本**，改一处必须三处一起对。这解释了「默认态对、切套后才错」的典型现象——默认态由 driver 决定（恰好还没改错），非默认态被旧常量覆盖。

**关联文件**：`docs/_fix_outer.txt`（changed=5 added=1）、`docs/_probe_fx.txt`、`docs/_verify_log.txt`（OC 矩阵复测 groups=1）。

> 更新时间 2026-11-05（补充来源：沐梓溪狼衣装菜单重构 · 本轮接手回归修复）

## §14 ObjectToggle 极性语义全解（生成公式 / Inverted / IsConstant / 遮断 / 定案法）

> 来源：原 08-症状速查 第 149/150/156/160/167/175/177 轮实测（2026-11-06 至 11-11）与主表/历史补充行，2026-11-11 结构化并入。与 §2（官方字段）、§12（Active 语义）、§13（driver 同步）配套；一键穿衣架构侧见 [03-06 §14/§19](../03-菜单与参数/03-06-ma-menu-build-by-reflection.md)。

### 14.1 生成语义公式（MA 1.18.7 源码口径）

`ReactiveObjectPass.cs:690-780` 生成语义＝**条件成立态写 `Active` 值；`<default>` 态为 empty motion + WriteDefaults 回退场景 authored 值**（第 177 轮）。由此：
- `inv=True + Active=False` ＝ ON 写 0、OFF 回退 authored(1) ＝ **反相**（「ON 反而隐藏、OFF 反而显示」）。
- **正确组合＝`inv=False + Active=True` 且目标场景 authored `m_IsActive=0`**（OFF 回退 0）。
- 全场景统一三件套（第 149 轮定案）：**`Inverted=false` + `Active=true` + 目标场景 `activeSelf=false` + 参数默认 0**。
- **口袋案例追溯（第 177 轮复核闭环）**：休闲装口袋 `YURU_pokke` 另陷衣面后 9.5mm，`localPosition.z+=0.0176` 露出；修复后同机位 on/off 特写 `30/31_pocket_*_close.png` 像素差 diffPx=562/maxDelta=401（`docs/_r510_review_close.txt`）。修复前 0.6m 机位 on/off 对照无效（口袋被衣面遮挡），不可作判据。

### 14.2 Inverted=true 生成反相状态机（永远进不去开启态）

**症状**：一键穿衣切到睡衣，只出现拖鞋/帽子等配饰，上衣/裤子/外套永远不出现（其余套装正常，第 149 轮）。
**根因**：`ReactionRule.InitiallyActive = (所有控制条件 InitiallyActive) ^ Inverted`。Inverted 为真时整个反应被当作「初始就激活」，生成反相状态机：`<default>` 只有「离开」转场、开启态只有「回到 default」的转场。
**判据**：烘焙后读该层——坏的 `<default> trs=2` / 开启态 `trs=1`；正常的 `<default> trs=1` / 开启态 `trs=2`。运行时 `animator.GetCurrentAnimatorStateInfo(layerIdx).shortNameHash` 恒等于 `<default>` 的 hash，参数已为 1 也不跳。
**修法**：按 §14.1 三件套统一，改完重烘焙并逐层比对转场条数（**1:2 才算对**）。

### 14.3 IsConstant：默认开 + Bool → MA 不生成任何状态

**症状**：开关参数默认 true（默认就穿着），改动后该开关彻底失效：参数 0/1 都不动，`<default>` 与开启态之间 0 条转场（第 150 轮）。
**根因**：`ReactionRule.IsConstant` 成立——参数默认值落在激活区间且是 Bool（只有 0/1）时，MA 认为条件恒成立，只把 clip 写进 `<default>`、不生成任何状态与转场（`ReactionRule.cs:29-38`、`ReactiveObjectPass.cs:642-753`）。
**判据**：默认 0 → `<default>` 1 条 + 开启态 2 条；默认 1 → 两条都是 0 条、`<default>` 的 clip 还是 empty motion。
**修法**：`Objects[].Active` 与 `ObjectToggle.Inverted` **同时取反**（Active true→false、Inverted false→true），`InitiallyActive` 由 true 变 false 跳出常量分支，同时保住「参数 1=显示」语义。

### 14.4 场景态 ≡ MA/EP 默认值 ≡ wantState 三处一致

为防「切套装关不掉」把默认套装节点 SetActive(false) → 一打开场景/进 Play 就是裸模（编辑器不跑 FX，场景态就是最终观感，第 150 轮；`ReactiveObjectPass.cs:371`、`ReactiveObjectAnalyzer.cs:301`）。
- 默认要显示的部件：场景 `true` + 参数默认 `1` + `Active=true`；默认不显示：场景 `false` + 默认 `0`。**不要用「关场景」来防叠加**。
- 新衣装容器场景 `activeSelf=false`；旧套装中参数默认 0 的部件也要 inactive（逐个核对 EP defaultValue），否则切回旧套装时被空 clip 重置为 active（历史补充行）。
- 判据：GM 默认/往返切换后旧套装可见 Renderer 数一致。

### 14.5 父级遮断：OT 目标是整套模型根时，根一关全灭

**症状**：兜帽卫衣「鞋子/袜子」开关点了没反应；一键穿衣关掉后整套菜单全部失效（第 160 轮）。
**根因**：① OT 目标是整套模型根，根一关，子级 `LH_Shoes/LH_Socks` 即使 activeSelf=true 也不渲染；② 参数默认 1 但根默认 inactive，「点一下写 0、再点写 1」全程看不到变化；③ 一键穿衣绑共享 Int，Toggle 关闭写 0（不是回上一档）。注意 `衣装开关/兜帽卫衣`（菜单容器，常驻 active）与头像根下的 `兜帽卫衣`（模型根）是两个对象。
**修法**：**模型根常驻 active**，显隐下放到 4 个 SMR 子对象（场景 activeSelf=false + 各自 Bool 默认 0 + OT 行 Active=true）；一键穿衣删 OT 只留 Toggle，靠 FX driver 一次置 5 个 bool=1（Equals 6）/全 0（NotEqual 6）。实测：6→四件全开、单独关鞋只关鞋、切回 3→四件全关。

### 14.6 分组根「原子关、壳必开」

分组根（睡衣/Riptide/兜帽卫衣/万圣节卫衣）必须 activeSelf=True，叶子网格 False；MA 运行期只改子件 m_IsActive（`__MA/ActiveSelfProxy/<名字>##<n>`，实测 67 个）。详见 [03-06 §19.7](../03-菜单与参数/03-06-ma-menu-build-by-reflection.md)。

### 14.7 自锁：OT 目标 = MenuItem 的父级容器

ObjectToggle 的目标就是 MenuItem 的父级容器时，MA 生成的 `MA Responsive` 层入口条件带 `__MA/ActiveSelfProxy/<该容器>`；容器默认 activeSelf=false → 代理初值 0 → 永远进不了 ON 状态（自锁，2026-11-05 实测，历史补充行）。
- 修法：把 `MenuInstaller + MenuGroup`（单件用 `MenuInstaller + MenuItem`）移到**激活的菜单容器**（如 `菜单开关/...`），MenuItem 不要再放在被开关的容器内部；被开关的衣装容器保持 inactive。
- 判据：烘焙后该层 default→ON 只有参数条件、不再带自身 proxy 条件。
- 相关：MenuItem 宿主 GO 本身 inactive → MA 整条跳过（连同 OT），见 [07-01 §31](../07-验证与排查/07-01-bake-and-validation.md)；MenuItem+OT+Parameters 三者必须同 GO（第 147 轮：MenuItem 放子节点时 OT 不绑定参数，烘焙后 m_IsActive 曲线数=0）。

### 14.8 极性定案法：运行时两态断言 + 三轮隔离

**不要从状态名或转场条件文字推断极性**（第 149/150 轮）：转场文本只描述「条件→目标状态」，On/Off 的 clip 值、层权重、WriteDefaults 都可能反向；作者也可能把 On/Off 命名反着用；切换后立刻读会读到转场未走完的中间态。
- **两态断言**：`SetBool("RT/HotPants", true)` → 目标 activeSelf=True；false → False，即 1=显示。每个参数各跑 true/false 一次。
- **三轮隔离定案**：A 全部 0 记可见数 → B 全部 1 记可见数 → C 只开单个记可见数；三轮一致才定案；每轮之间 `Rebind()` 并跑满 Update 时间。`RT/*` 实测：0→0/12、1→11/12、单参数→1/12，故 **1=显示**。
- 推断错了会连锁改坏驱动表（§13 的 driver 常量必须跟着定案结果改）。

### 14.9 自建 FX 层会被 MA 吸收并清权重（别用它替代 OT）

把自建层并进主 FX 控制器后，MA 识别出 clip 目标就是某个 ObjectToggle 对象，自动把 clip 扩写成整套目标（1 条 → 13 条 m_IsActive 曲线 + ActiveSelfProxy），随后该层被空层清理逻辑置 defaultWeight=0（源里写的是 1.0）（第 156 轮）。
- 判据：烘焙产物 `fx.layers[i].defaultWeight`：自建层=0，作者层 foot/houtai=1；clip 曲线数 1→13。
- **不要用自建 FX 层替代 MA ObjectToggle**。要用自建层驱动显隐，clip 目标不要落在已有 OT 的对象上（换中间空物体做代理），或干脆用 OT + 真机验证。
- 例外：作者自带成对 OFF/ON clip 的部件用 MergeAnimator 两态层（[05-07 §9](../05-动骨与穿模/05-07-tail-gloves-migration.md)）；`Renderer.m_Enabled` 曲线 MA 不碰，可做门控（[02-02 §35.3](../02-工程组织与工作流/02-02-ma-workflow-habits.md)）。

### 14.10 ShapeChanger 运行时驱动陷阱（本地实测补充 §3）

第 167 轮实测（兜帽卫衣「外套」开关只对绷带生效）：
- `ChangedShape.ChangeType` 枚举 **`Delete=0 / Set=1`**（`ModularAvatarShapeChanger.cs:13-17`）。设 0=Delete 走 `ReactiveObjectAnalyzer.LocateReactions.cs:250` 的 `RegisterAction(deleteKey,…,new VertexFilterByShape(...))`，是**构建期顶点过滤**，不生成运行时 blendshape 曲线。运行时驱动形态键必须 **Set(1)**。
- 反应式组件宿主 GO 只有一个 MenuItem、没有任何 ObjectToggle 时，MA 不会为该宿主建立参数驱动层 → **同 GO 补一个 ObjectToggle** 后才出现 `MA Responsive` 层与曲线。
- `AvatarObjectReference` 指向 **GameObject**（源码用 `GetComponent<SkinnedMeshRenderer>(shape.Object.Get(changer))` 解析），不要指向组件。
- 改用 MA 驱动时清空旧控制器里同名 blendshape 曲线，避免双写。
- 判据：ChangeType 改 1 + 补 OT 后立刻出现「外套」状态与 2 条 `blendShape.houtai_off` 曲线（修复前全 FX 扫描=0）。
