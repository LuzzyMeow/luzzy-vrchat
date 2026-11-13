# MA 通用机制：Setup Outfit 完整工作流、烘焙期改写、手动烘焙与分发

> **适用**：拿到一整套服装 prefab 要「Setup Outfit」、判断某套装能否被 MA 直接处理、排查「配了没效果」、要看手动烘焙产物、要对外分发 MA 兼容 prefab。
> **来源**：Modular Avatar 官方文档（<https://modular-avatar.nadena.dev/docs/intro> 起 67 页），2026-11-01 逐页抓取原文（MA 1.18.7，changelog 2026-09-01）；本地副本 `<VRCHAT_ROOT>\tools\tmp\ma-docs-txt\`。每节带官方 URL，可逐条核对。
> **定位**：本册是**官方语义参考**（字段 / 默认值 / 限制 / 烘焙期行为 / 组件间交互），与 `00-基底/00-02-modular-avatar.md` 的代码范式互补；基底是外部 MIT 原件，**一字未改**，本册不重复其已有内容。
> **标记**：【文档未说明，需实测】= 官方没写、须本工程验证；【需查源码】= 字段名未公开、需反射；【待复核】= 与 KB 现有表述冲突、需人工定稿；【KB 已有】= 只补语义。组件名 / 字段名保留英文原名。
> **本册来历**：2026-11-01 第 135 轮「MA 官方文档 × KB 缺口审计」的成品稿整册并入（审计报告在 `<VRCHAT_ROOT>\tools\tmp\ma-gap\`）。稿内「建议落地」一节保留为历史记录，落点已在并入时确定。

## 目录

- §0 三张速查表
- §1 Setup Outfit 完整工作流（1.1 两级支持 / 1.2 流程 / 1.3 找骨判据 / 1.4 回退模式 / 1.5 T-A pose / 1.6 Merge Armature 规则 / 1.7 Outfit Root + Mesh Settings / 1.8 全同名 `.1` / 1.9 humanoid PhysBone / 1.10 分工 / 1.11 整套装开关动画 / 1.12 重复执行与 Preset / 1.13 验收三招 / 1.14 排查树）
- §2 烘焙期改写总表（2.1 激活条件 / 2.2 优先级 / 2.3 时序 / 2.4 动画路径 / 2.5 PhysBone-约束 / 2.6 反应层与 ShapeChanger / 2.7 MMD / 2.8 阶段与层序不稳定 / 2.9 Debugger）
- §3 Manual Bake 产物与清理
- §4 错误处理
- §5 安装与升级
- §6 实验特性
- §7 分发规范（7.1 不发本体 / 7.2 嵌套 prefab / 7.3 内部参数 / 7.4 Logo / 7.5 SemVer）
- §8 Parameters 的 Internal 语义
- §9 教程里的官方推荐做法
- §10 KB 现有说法需修正
- §11 建议落地
- §12 出处索引

---

## §0 三张速查表

### 0.1 Setup Outfit 一键做了什么（按执行顺序）

| # | 动作 | 位置 | 关键点 |
|---|---|---|---|
| 1 | 找骨 `OutfitRoot → [Armature] → Hips` | — | Hips 名须**包含**基础头像 hip 骨名；前后缀全骨一致；大小写敏感；Armature 名随意 |
| 2 | T/A-pose 转臂（自动） | — | **仅臂长与基础头像完全一致**才做 |
| 3 | 建 **Merge Armature** | outfit armature | 带 prefix/suffix + armature 引用；upper chest 用多个 MA |
| 4 | 建 **Outfit Root** | 套装根对象 | 只是标记 + 工具入口 |
| 5 | 建 **Mesh Settings** | 套装根对象 | probe anchor 对齐基础头像 + 合理 bounds |
| 6 | 设 **Position lock = Base ⇒ Target（单向）** | MA inspector | **编辑态转基础骨、衣服跟随的来源** |
| 7 | 已配置则跳过 | — | 可安全重复执行 |

### 0.2 烘焙期改写了什么

| 类别 | 官方明确会改 |
|---|---|
| 动画路径 | 改到合并后的骨；**active 动画指向 source hierarchy** |
| 整套装开关 | 「开关原套装整体」动画改写成覆盖该套装**散落各处全部对象**（含 PhysBone） |
| 骨骼引用 | SMR bone 引用重写到基础骨；匹配不上则**新建子骨** |
| 动骨/Contact | `PhysBone`、`Contact` 的 **target 字段**改到新骨（**组件不在 MA 之下也改**） |
| 其它组件 | 留在原层级，**生成约束**跟踪；必要时生成 **ParentConstraint** |
| 复制来的 PhysBone | humanoid 骨上、从基础头像复制来的 → MA **会尝试删除** |
| 反应层 | Reactive 组件按激活条件生成 FX 层（ObjectToggle→`m_IsActive`、ShapeChanger→blendshape） |
| MMD 层 | 自动加 relay 层保护「原本第 2、3 层」；MergeAnimator 层加 padding 保护 |
| 参数 | 内部参数构建期唯一化；`Change name to` / auto rename 生效 |

### 0.3 症状 → 根因

| 症状 | 先查 | 根因 |
|---|---|---|
| 头发 rig Setup 失败 | 有没有 `Hips` | 判据要求 Hips；只有 Head 的 rig 天然不满足 |
| 部分骨不跟随 | 前后缀是否全骨一致、大小写 | 不一致的骨不与基础骨合并，改为新建子骨 |
| 手臂歪/反向 | 臂长是否完全一致 | 差一点就静默跳过自动转换；方向按骨位置差 |
| 动骨异常/消失 | humanoid 骨上是否有 PhysBone | MA 尝试删「从基础头像复制来的」PhysBone |
| 出现 `Armature.1` | 是否与基础头像全同名 | 触发 Unity bug，MA 加 `.1` 规避 |
| 切套装根动骨也被关 | — | 整套装开关动画被改写到了含 PhysBone 的全部对象 |
| 编辑器里形态键「没变」 | 是 ShapeChanger / BlendshapeSync | **这两个编辑态有预览**（与 KB 现说法冲突，见 §10） |

---

## §1 Setup Outfit 完整工作流

### 1.1 两级支持：`MA Compatible` vs `MA Preset`

| 级别 | 含 MA 组件 | 用户要装 MA | 用途 | 可用 Logo |
|---|---|---|---|---|
| **MA Compatible** | 无（只需骨骼命名 + PhysBone 摆放符合约定） | 不需要 | 大多数现成衣服 | 不可以 |
| **MA Preset** | 有（已跑 Setup Outfit，常预配 BlendshapeSync/ShapeChanger/ObjectToggle） | 需要 | 作者分发的开箱即用件 | 可以（符合 Logo 条款） |

- 官方 tl;dr：做 Compatible **只需保证 Setup Outfit 能跑通**；做 Preset 提前跑 Setup Outfit，并考虑预配三项增强。
- 作者可同时提供两版 prefab 覆盖两类用户。
- 出处：<https://modular-avatar.nadena.dev/docs/distributing-prefabs/for-outfit-creators>

**KB 对照**：KB 只知「Setup Outfit 存在 + 对头发无效」，无分级概念 → 补在 `02-08-cross-avatar-clothing.md` 开头（判「这资产该不该走 MA 路线」）。

### 1.2 标准流程

1. 衣服 prefab **拖进 avatar 层级**（不用 unpack 头像或衣服 prefab）。
2. 右键衣服根 → `[ModularAvatar] Setup Outfit`；成功即自动加组件。
3. 进 Play 看合并结果；**卸载 = 删除该对象**（非破坏性，不改原始资产）。
4. 不 unpack 的原因：保留 prefab 实例才能在作者发新版时直接更新（官方 tip）。

### 1.3 找骨判据（最重要的规则）

```
OutfitRoot（套装根）
└─ [Armature]（名字随意）
   └─ Hips（必须包含基础头像 hip 骨的名字）
```

| 规则 | 说明 |
|---|---|
| Hips 名**包含**基础头像 hip 骨名 | 基础叫 `Hips` 时，`Kipfel_Hips`、`Hips.001` 可命中 |
| 前后可带额外文字 | 但**必须在所有骨上保持一致** |
| **大小写敏感** | `Hips` ≠ `hips` |
| **Armature 层叫什么都行** | 只找 outfit hips |
| 不满足 → 回退 | 按「常见骨名表」匹配并**重命名**衣服骨（不可靠，见 1.4） |

- 机制：MA 用 Hips 作锚点推断整套骨的命名前后缀，再按「去前后缀后同名」逐骨匹配基础头像。
- 坑：Hips 前缀对、其它骨不一致 → 那些骨**不合并**，改新建子骨挂在对应父骨下（表现为「部分骨不跟」）。
- 头发 rig 只有 Head 无 Hips → **必然失败**（KB `05-03:234` 已记「对头发无效」，但未写判据本身）。
- 出处：同上（Identify bones 节）

### 1.4 回退模式为什么不可靠

- 机制：命名不符合规则时，拿衣服骨名去匹配「常见骨名表」，匹配上就**重命名**衣服骨。
- 官方原话：best to avoid this mode as it can be less reliable → 推荐**精确对齐**命名。
- 做法：先统一命名（去/加前后缀使「去前后缀后 = 基础头像骨名」）再跑 Setup Outfit；或直接用 `Merge Armature` 的 `Adjust bone names to match target`（Setup Outfit 走的就是这套逻辑）。
- 出处：同上

### 1.5 T-pose / A-pose 自动转换

机制：衣服与头像姿势不同（如头像 T、衣服 A）时，Setup Outfit 会**旋转手臂骨**去匹配基础头像。

两条硬条件：
1. **只有臂长与基础头像完全一致才转**；差超过「极小量」→ **静默跳过**（不报错）。
2. **旋转方向由骨位置差决定** → 骨位置略偏就可能**转错方向**。

做法：
- 期望自动转：先确认两套臂长一致；不一致先整体对齐（`Reset position to base avatar` + `Adjust outfit overall scale to match base avatar`，见 1.7）。
- 自动转错：用 Outfit Root / Merge Armature 的 **`Reset position to base avatar`**（勾 `Also set rotation` 可做 A↔T 转换）手动修。
- 【待实测】「极小量」容差官方未给数值，可在两套骨量上臂长度后回填。

出处：for-outfit-creators（A vs T pose 节）、<https://modular-avatar.nadena.dev/docs/reference/outfit-root>

### 1.6 Merge Armature 的合并规则

| 规则 | 说明 |
|---|---|
| 按名匹配 | 遍历 MA 下所有 GO，用「去掉 prefix/suffix 的名字」找基础骨 |
| 匹配成功 | 重写「指向该骨的引用」；有时无法重写 → **新建子骨** |
| 匹配失败 | 在对应父骨下**新建子骨** |
| 避免重复骨 | 尽量合并；**带非 Transform 组件的骨不合并**，挪到对应基础骨下面 |
| 保留原层级 | 含非 Transform 组件的部分保留，否则一般删除 |
| 动骨/Contact | target 改到新骨，**组件不在 MA 之下也改** |
| 其它组件 | 留在原层级，**生成约束**跟踪；必要时生成 ParentConstraint |
| 新骨命名 | 默认给「本次新增、基础头像没有的骨」改名避免撞名（可关 `avoid name collisions`） |
| 嵌套合并 | MA 1.7.0+ 支持（A→B、B→C），自动决定顺序 |

定位：
- 适合：**带 skinned mesh、要跟头像骨的衣服**。
- 不适合：**要通用到任意头像的配件**（官方例：finger-pen）——MA **假设要绑的骨不会移动**，只对「当初设定的那个头像」成立 → 这类用 **Bone Proxy**。
- 参考对象存的是**路径引用**，prefab 保存后能恢复 Merge Target。

出处：<https://modular-avatar.nadena.dev/docs/reference/merge-armature>

### 1.7 Outfit Root 与 Mesh Settings

**Outfit Root**：Setup Outfit 自动加在**套装根**，记录 `Armature Root`；**本身不处理任何东西**，只是标记 + 工具入口：
- `Adjust bone names to match target`（改名匹配基础头像）
- `Reset position to base avatar`（可做 A↔T 转换、整体缩放、复制旋转/局部缩放）
- `Match Scale Adjusters to base avatar`
- 对旧版 MA 做的套装重跑 Setup Outfit 也会补上它；手动加时 `Armature Root` 指向套装主骨层级根 Transform。

**Mesh Settings**：Setup Outfit 自动加，给套装下所有网格设 `Anchor Override`（与基础头像一致）+ 合理 bounds。手动配置先选模式 `Inherit` / `Set` / `Don't set` / `Set or inherit`（套装 prefab 推荐 `Set or inherit`，让头像级设置优先）。坑：对**对外分发**资产设 bounds/probe 要小心（可能与用户头像不符）；bounds 只影响 Skinned Mesh Renderer，Anchor Override 还影响 MeshRenderer/LineRenderer。

出处：<https://modular-avatar.nadena.dev/docs/reference/outfit-root>、<https://modular-avatar.nadena.dev/docs/reference/mesh-settings>

### 1.8 骨名与 Armature 全同名：Unity bug 与 `.1`

- 机制：**所有骨 + Armature** 都与基础头像**完全同名** → 触发烦人的 Unity bug；Setup Outfit 把 Armature 改名加 **`.1`** 规避。
- 做法：**从一开始就避免**——改 armature 名或 hips 名其一即可。
- **KB 解释**：`05-08-cross-avatar-garment-fit-newfit.md:751` 记录的「子级 `Armature.1` 上有 `ModularAvatarMergeArmature`」正是此规避产物，**不是** AAO/NDMF 改名、不是资产损坏 → 回填该行。
- 出处：for-outfit-creators（Exactly the same naming 节）

### 1.9 禁止项：humanoid 骨上的 PhysBone

- 机制：humanoid 骨（hips/chest…）上的 PhysBone 不推荐；不少衣服的 PhysBone 是**从基础头像复制来的**，MA 在这种情况下**会尝试删除** → **行为不可预测**。
- 推荐：PhysBone 放在**基础头像上不存在**的 GO，或 **Merge Armature 组件所在 GO 之外**。
- 做法：查衣服 armature 上所有 `VRCPhysBone` → humanoid 骨上的删掉或移到「衣服独有 GO」→ 重烘确认动骨在、target 正确。
- 出处：for-outfit-creators（PhysBones on humanoid bones 节）

### 1.10 Merge Armature / Bone Proxy / Preset 分工

| 维度 | Merge Armature（Setup 自动建） | Bone Proxy |
|---|---|---|
| 对象 | 整棵**骨层级 + skinned mesh** | **单个对象** |
| 合并骨 | 按名合并 | **不合并** |
| humanoid 骨 | 依赖骨名匹配 | **不依赖骨名**（可做通用配件） |
| 位置/旋转 | Position lock（单向/双向/不锁）+ 重置工具 | Attachment Mode（`As child; keep position` / `As child; at root`，可吸附到手） |
| 动画路径修正 | 会 | 会 |
| **整套装开关动画改写** | **会**（见 1.11） | **不会**：切父级**不影响** BP 对象及子级 |
| 典型 | 衣服 | 发夹、手持道具、通用小配件 |

选型：衣服 → MA；单件刚性/换头像仍要能用 → BP；需要「切父级一起消失」→ MA 或把 BP 放进会被切的对象内部并自补动画。

出处：for-outfit-creators（Differences from Bone Proxy）、merge-armature（When shouldn't I use it）

### 1.11 「整只套装开关」动画会被自动改写

- 机制：Merge Armature 做两类修正：① 改动画路径对应移动后的骨；② 把「开/关原套装整体」的动画**改写成开/关来自该套装的所有对象**（因为合并后被分散到头像各处）。
- 效果：用**任何动画工具**开关套装根对象，MA 会自动把该套装散落的 meshes **以及 PhysBone** 一起开关。
- 为什么重要：KB 的「一键穿衣」靠 `OutfitSync` 逐件写 bool（`03-菜单与参数/03-06` §14）。官方语义意味着「整只套装一起开/关」可**直接动画套装根**；逐件 bool 的价值在「跨套装混搭 + 单件独立」。
- 坑：只对 Merge Armature 管理的对象成立，**BP 对象不受影响**；不想动骨跟着关就别放进该对象树。

出处：for-outfit-creators（Animation corrections 节）

### 1.12 重复执行与 Preset 化

- 可重复执行：再跑 Setup Outfit 会「看到已配置」并**跳过**（官方口径）。
- Preset = 交付**已跑过 Setup Outfit** 的 prefab，用户拖进去即可用。
- 官方建议再加三项：① **Blendshape Sync**（体型形态键跟随基础头像）；② **Shape Changer**（部件启用时缩小/隐藏底下身体，可响应动画，见 2.6）；③ **Object Toggle + 菜单**（开关部件）。
- 多个开关合并成一个子菜单（官方做法）：新建 GO 作为这些 toggle 的父对象 → `Menu Item` Type = `Sub Menu`（教程：新建 GO + Menu Installer + Menu Item，Type 设 Sub Menu，再 Add Toggle）。
- 出处：for-outfit-creators（Presets 节）、<https://modular-avatar.nadena.dev/docs/tutorials/advanced-toggles>

### 1.13 验收三招

| 招式 | 做法 | 能看到 | KB 落地 |
|---|---|---|---|
| ① 编辑态同步 | Setup Outfit 后**转几根基础骨**看衣服跟不跟 | 骨绑定/合并是否成功（最便宜） | `SKILL.md` 铁律 1 后补「①」；`05-03` 补步骤 |
| ② Play 合并 | 进 Play（可用 **Avatar 3.0 Emulator / Gesture Manager** 更接近上传态） | 菜单、开关、反应层、形态键联动 | KB 已有 GM 流程；补「Avatar 3.0 Emulator」选项名 |
| ③ Manual Bake | 右键 → `Modular Avatar -> Manual Bake Avatar` 看克隆 | 最终烘焙产物（含 AAO 全部改动） | KB 已有；补产物清理见 §3 |

为什么三条都要：① 只证明骨跟随；② 看不到 AAO 合并后网格；③ 唯一能看「上传后真正长什么样」（但编辑态看不到下游联动，见 9.6）。
出处：for-outfit-creators（How to check 节）、<https://modular-avatar.nadena.dev/docs/manual-processing>

### 1.14 装衣服失败排查树

```
Setup Outfit 没反应/报错
├─ 有 Hips 骨吗？ ── 无 → 判据不满足（头发 rig）→ 用 BP 或直接挂 Head
├─ Hips 名含基础 hip 骨名吗？ ── 否 → 回退改名（不可靠）→ 先统一命名
├─ 前后缀/大小写全一致吗？ ── 否 → 部分骨不合并 → 统一后重跑
├─ 与基础头像全同名（含 Armature）？ ── 是 → 出现 Armature.1（正常规避）→ 可改名
├─ 手臂姿势不对 ── 查臂长是否完全一致 → 用 Reset position + Also set rotation
├─ 动骨异常/消失 ── 查 humanoid 骨上 PhysBone → 移到独有 GO / MA 之外
└─ 切套装动骨被关 ── 预期（整套装动画改写）→ 不想关就别放进去
```

---

## §2 MA 烘焙期改写总表

> 用途：解释「编辑器所见 vs 上传后」的差异，区分「MA 保证的行为」与「不可依赖的行为」。KB 目前只在 `07-01-bake-and-validation.md` 以烘焙审计记录现象，缺这层官方语义。

### 2.1 Reactive 系统的激活条件与响应范围

Reactive 组件（Object Toggle / Shape Changer / Material Setter / Material Swap / Mesh Cutter）按「自身是否 active」施加效果。**算 active（全部满足）**：
1. 自身 GO **及其所有父级** active；
2. 若在某个 **Menu Item** 的同物体或子级：该 Menu Item **被选中**（**只看第一个父 Menu Item**，父级 Submenu 不算）。

**烘焙后响应**：① 改变 GO active 的**动画**；② **Object Toggle** 对其它反应组件 active 的影响；③ **Menu Item 选中**。
`Invert condition`：上述任一条件**不成立**时才施加效果。
出处：<https://modular-avatar.nadena.dev/docs/reference/reaction>

### 2.2 多条反应规则冲突：优先级

- 官方规则：多个反应组件同时 active 且效果冲突（一个开一个关）→ **层级最低者胜**。
- Shape Changer 页同款措辞：「the lowest one in the hierarchy will generally win」（注意「generally」——同层级并列官方未写）。
- KB 对照：KB 有「MA Reactive 层排在自建层之后、同一属性后写者胜」（**观测**，见本包 `SKILL.md` 铁律 1 与 `08-症状速查/` 中层序相关条目；不引行号，避免改版后漂移），但没有「层级最低胜」这条官方规则。两者不矛盾：前者是「MA 层 vs 手写层」的层序观测，后者是「多个 MA 反应组件之间」的规则 → §10.3 要求分开写。
- 出处：reaction（Priority rules）、<https://modular-avatar.nadena.dev/docs/reference/reaction/shape-changer>

### 2.3 反应时序：一帧延迟与链式延迟

| 情形 | 时序 |
|---|---|
| 响应「GO active 变化」 | **延迟一帧**；对象被关闭时关闭动作也**推迟一帧**对齐 |
| A 控制 B，A 关 | 链式：A 延迟 1 帧 → B 再 1 帧；A→B→C 时第 1 帧无变化、第 2 帧 A 关、第 3 帧 B 与 C 同关 |
| 整体 | 官方 warning：**时序未来可能变，不要依赖精确时序做复杂效果** |

- 为什么重要：KB 有「参数已切但下一帧才生效」「Play 断言要等帧」的经验——官方说清了这是**反应系统实现特性**，不是 VRChat 参数同步。
- 坑：Play 断言切换后至少要跨 1~2 帧（链式更久），不能同帧读两次判断。
- 出处：reaction（Reaction timing）

### 2.4 动画路径与整套装开关动画

见 1.11。补充：改路径时按**被动画属性**决定指向——**transform 动画指向合并后的骨**，**GO active 动画指向 source hierarchy**。即同一段动画里 transform 与 `m_IsActive` 路径可能落在不同对象上 → 排查烘焙路径时不能假设整段统一。
出处：merge-armature（How does it work）

### 2.5 PhysBone / Contact / 约束与参考重定向

| 项 | 处理 |
|---|---|
| 动画引用 | 按属性重定向（见 2.4） |
| **PhysBone / Contact** | **target 字段**更新到新骨，**组件不在 MA 之下也改** |
| 其它组件 | 留在 source hierarchy，**生成约束**跟踪合并后层级 |
| 层级残留 | 含非 Transform 组件的保留，否则一般删除 |
| 必要时 | **生成 ParentConstraint** 让东西 Just Work |
| SMR 引用 | 匹配到的骨改写引用；匹配不到则新建子骨 |

- KB 对照：`03-06 §14.9` 记录过「Play 克隆出现 ParC、`source0` 指向某骨」（第 132 轮按「Play 克隆 ParC 走 constraint source0」处理）——本节给出官方解释：**这是为「留在原层级的组件」生成的跟踪约束** → 回填该段。
- 出处：merge-armature

### 2.6 反应层与 ObjectToggle / ShapeChanger 语义

1. **ObjectToggle 的「关」一侧靠场景初始 active 回落**，不生成「关」动画；列表 `Active=false` = 「我启用时把目标关掉」（KB 已覆盖）。
2. **Shape Changer `Delete` vs `Set`**：`Set` = 把形态键设为指定值；`Delete` = **删除受该形态键影响的多边形**——**未被动画引用时**真的减少面数（性能收益），**被动画引用时**仍完整隐藏但**不减少统计面数**。官方建议：形态键把部位缩到「几乎为零」时用 `Delete`；想永远保留多边形用 `Set`。
3. **`threshold`**：调「哪些顶点算受影响」；删得不够就**调小**。
4. Shape Changer 响应自身**及父级**的 active，可用于换装/移除部件后恢复底下身体。
5. **不应**用它改「同时被其它动画驱动的形态键」——要动画就动画「Shape Changer 所在对象」的开关。
- 出处：shape-changer、reaction

### 2.7 MMD 世界关层与 relay/padding（PC 专属）

- 机制：部分 MMD 世界会**关掉 FX 第 2、3 层**（为了用自己的表情覆盖）。
- 自动处理：MA 安排 **relay 层**，让**原本第 2、3 层**在该世界仍按预期被关/开；**Merge Animator 加的层（含 replace）不受影响**，必要时加 **padding 层**保护。
- opt-in：把 **MA MMD Layer Control** 状态机行为**挂到层上**。
- 整体关闭：加 **VRChat Settings** 组件，关 `MMD World Support`。
- 边界：只对「确实关第 2、3 层」的世界有效；受 VRChat 限制无通用解。
- 致命坑：该行为**只能挂层**；挂到单个 state 上**会把构建搞坏**（官方原话）。
- KB 对照：只有无结论评论「带 FaceEmo 进 MMD 房头发动骨直接没了」（`02-09:239`）——正是本机制 + FaceEmo 自建层的结果 → 新增小节并标注结论。
- 适配：Quest 不进 MMD 世界（PC 世界）→ 只影响 PC 分支。
- 出处：<https://modular-avatar.nadena.dev/docs/general-behavior/mmd>、<https://modular-avatar.nadena.dev/docs/reference/vrchat-settings>

### 2.8 NDMF 阶段与「层顺序不稳定」

- MA 行为挂在 **NDM Framework** 的 pass 里；组件之间**没有**「后加优先」保证。
- 版本策略明确：内部类名/方法名**不是稳定 API**；NDMF pass 的 **qualified name 不稳定**；**MA 各 pass 执行顺序也不稳定**；唯一稳定的是**插件限定名 `nadena.dev.modular-avatar`**。
- 若确需依赖某 pass：官方建议去 GitHub 提 feature request，不要硬编码 pass 名。
- 含义：任何「靠层 index / pass 顺序推断谁覆盖谁」的 KB 说法都要降级为「本版观测」——见 §10.4。
- 出处：<https://modular-avatar.nadena.dev/docs/distributing-prefabs/versioning-policy>

### 2.9 调试工具：Reaction Debugger

- 入口：右键 GO → `Modular Avatar -> Show Reaction Debugger`。
- 用途：**模拟**开关对象/菜单项，观察反应组件效果，不必进 Play。
- KB 对照：`02-03:144` 只写「有就用它」，无入口与用途 → 补一行。
- 出处：reaction（Debugging problems）

---

## §3 Manual Bake：产物、Unpack 与清理

- 机制：平时 MA 在 Play/构建时自动处理；Manual Bake 手动在编辑态跑一遍，生成**已应用全部变换的克隆**。
- 步骤：选中/右键头像 → `Modular Avatar -> Manual Bake Avatar` → 生成 `<Avatar>_Baked`。
- 产物：在**工程主 `Assets/` 的 `ModularAvatarOutput` 文件夹**；初期**打包成单个文件**（为绕 Unity bug + 提速度），Project 视图选中 → Inspector 点 **`Unpack`** 拆成独立资产。
- 清理：**不管有没有 Unpack，删掉克隆的 baked 头像后该目录可安全删除**；**不会自动清理**（FAQ 重申）。
- 坑：忘了删 → 长期堆积、下次 bake 复用/覆盖，排查「资产变多变乱」时易误判；`Unpack` 前不要手改打包文件。
- 用途不止调试：**给非 VRChat 平台导出**（如 VRM）要先 Manual Bake 再对普通头像导出。
- 出处：<https://modular-avatar.nadena.dev/docs/manual-processing>、<https://modular-avatar.nadena.dev/docs/faq>

---

## §4 错误处理

### 4.1 错误报告窗口

- 机制：处理出错时 MA **打开错误报告窗口**。
- 用法：**点对象名** → Scene 里选中对应对象；**多数错误随编辑自动刷新**，但**部分类型需重新 build 才更新**（「改了还报错」≠ 没改对）；关掉后重开：`Tools -> Modular Avatar -> Show Error Report`。
- KB 对照：KB 只靠反射读 `ErrorReportWindow._report`（`07-验证与排查/07-01-bake-and-validation.md` §10「NDMF 烘焙被中止：用反射读 ErrorReportWindow 拿真因」）做自动化验收，从未写「人怎么用」→ 补本节。
- 出处：<https://modular-avatar.nadena.dev/docs/problems>

### 4.2 「什么都没被处理」——先查 Apply On Play

- 官方首查：确认 **`Apply On Play`** 已勾选。
- 为什么：Play 自动处理由 NDMF 的 ApplyOnPlay 触发；关掉后进 Play 不跑任何 MA 变换，看起来像「MA 全坏」。
- KB 补充：KB 有「场景根 `nadena.dev.ndmf__Activator` 丢失 → ApplyOnPlay 不触发」（本包 `SKILL.md` 使用前提 + `08-症状速查/00-主表.md`「NDMF Activator 丢失 → ApplyOnPlay 不触发」条）→ 合成**三步排查**：① 勾选 `Apply On Play`；② 场景有无 `nadena.dev.ndmf__Activator`（丢了补 `ApplyOnPlayGlobalActivator`）【KB 实测】；③ Console 有无 NDMF/MA 报错（见 4.1）。
- 出处：problems（Nothing is getting processed at all 节）

---

## §5 安装与升级

### 5.1 VCC 报 `Failed to add Repo`

- 机制：**VCC 自身 bug**；多数表示**仓库已加过**。
- 步骤：点 `Cancel` → 到 **Community Repositories** 列表找 **`bd_`** 仓库 → 确认**复选框勾选**（仓库被禁用时包列表看不到 MA）→ 回包列表确认出现。
- 为什么：MA 的 VPM 仓库挂 `bd_` 名下；禁用后包列表过滤掉该仓库的包，与「没添加成功」表现相同。
- 出处：<https://modular-avatar.nadena.dev/docs/problems/install>

### 5.2 「资产由更新版 MA 创建」警告：停手先升级

- 机制：版本警告 = 当前 MA 低于**创建该资产时用的版本**，资产在当前版本**可能不正常**。
- **致命警告**：警告存在期间**编辑并保存**资产/场景，Unity **可能删掉「新版才有的设置信息」** → **先停手、升级，再改**。
- 升级：**ALCOM** 项目列表 `Manage` → 点 MA 旁**版本号**；**VCC** `Manage Project` → 点 MA 旁**版本指示器**；升级后点 **`Refresh Asset Database`**，警告应消失。
- 压制警告（为兼容旧版做资产）：① 建降级用工程，装**目标版本** MA，导入资产；② **目标 < 1.18.0 时先装 1.18.0**（顺序重要）；③ 出现不匹配警告点 `ignore`；④ 右键 prefab/GO → `Modular Avatar -> Reset version information`；目标 ≥1.18.0 点 **`Set required version to current`**，<1.18.0 点 **`clear required version data`**；⑤ 验证；⑥ 若要 <1.18.0，现在**降级**再验；⑦ 最后导入最新版工程再验。
- 为什么绕：版本信息是「最低 MA 版本」的序列化数据，<1.18.0 与 ≥1.18.0 表示方式不同，需先在 1.18.0 写成新格式再降级。
- 出处：<https://modular-avatar.nadena.dev/docs/upgrading>

---

## §6 实验特性

- 机制：MA / NDMF 含**默认关闭**的实验特性，仍在开发、可能无完整文档。
- 风险（warning）：**可能以不向后兼容的方式改变**——用了之后未来可能需**更新头像或资产**。
- 开启：先确保 MA 与 NDMF 为**最新（beta）**；菜单栏 `Tools -> NDM Framework -> Experimental Features`。
- 当前列表：`Resonite support`、`Portable avatar components`。
- 与纯 VRChat 改模：两者都与上传无关（Resonite 是另一平台导出；Portable 是平台无关头像组件）→ **日常不需要开**；看到他人工程里有 `NDMF*` 组件先确认是否用了实验特性。
- 出处：<https://modular-avatar.nadena.dev/docs/experimental-features>

---

## §7 分发规范（给 prefab / 衣服作者）

### 7.1 不要把 MA 本体塞进分发

许可允许，但用户可能装到**很旧版本**甚至**降级弄坏别的 prefab**；官方建议**引导到 GitHub 仓库**（VCC 安装方式官方仍在等 VCC 改进）。
出处：<https://modular-avatar.nadena.dev/docs/distributing-prefabs>

### 7.2 用嵌套 prefab 兼容非 MA 用户

有 MA 组件 → 用户**必须装 MA**。做法：① 正常做 prefab；② 双击进 **prefab mode**；③ 把 prefab 根拖到 Project → 弹窗点 **`Create Base`** → 改名（如「Outfit without Modular Avatar」）；④ **MA 组件配原 prefab、非 MA 设置配 base prefab**；⑤ 填设置可用 **prefab overrides 菜单**（在测试场景改好后选择应用到哪个 prefab）。
出处：同上

### 7.3 内部参数

用内部参数避免与别的 prefab 撞名：**内部参数构建期自动改成唯一名**（语义见 §8）。
出处：同上

### 7.4 Logo 使用边界（简）

只有 **prefab 里已预配置 MA 组件**的商品可用 Logo（`MA Compatible` 不符合）；编辑器扩展须**非破坏性**（卸载=删 prefab 对象）、**不得要求用户手动触发处理/生成**、**不得在自家 UI 用 MA Logo**、**不得暗示官方背书**；Logo 不得改形/变色/动画化/当背景（视频转场除外），用官方多色版本；判定权在 bd_/pumo，被要求须撤下（bd_@nadena.dev）。**纯自用改模不涉及**，仅在对外发布商品/工具时查。
出处：<https://modular-avatar.nadena.dev/docs/distributing-prefabs/logo-usage>

### 7.5 SemVer 与「唯一稳定锚点」

| 位 | 含义 | 对 prefab |
|---|---|---|
| major | 不兼容变更 | 旧 prefab 可能不能用 |
| minor | 向后兼容新功能 | **1.1.0 做的 prefab 可能在 1.0.0 不能用**；反之前向可用 |
| patch | 不涉存档格式的修复 | 一般无碍，除非修复影响你的 prefab |

- 推荐：同一 major（1.x.x）内**用最新版**。
- 稳定性边界：内部类名/方法名**非 API**（patch 也可能变）；NDMF pass 的 **qualified name** 与 **MA pass 顺序**都不稳定；**唯一稳定 = 插件限定名 `nadena.dev.modular-avatar`**。
- 含义：自动化/反射脚本唯一可硬编码的锚点是 `nadena.dev.modular-avatar`，pass 名与层序要「按版查询 + 运行期断言」。
- 出处：<https://modular-avatar.nadena.dev/docs/distributing-prefabs/versioning-policy>

---

## §8 Parameters 的 Internal 语义

**基本**：`MA Parameters` 声明 gimmick 用的 animator 参数，**重命名以避免冲突**，并定义是否同步/保存/默认值。类型：`Bool` / `Int` / `Float` / **`Animator Only`**（不进 EP 列表但仍参与重命名）/ **`Prefix`**（用于 PhysBone、Raycast 按前缀生成的一批参数）。

**重命名字段**：`Change name to`（在该 MA Parameters 对象**及其子级之外**改名，可用于避冲突或**故意让两个 gimmick 共用参数**）；`Auto rename`（自动选未占用名）。

**Internal 复选框（教程页）**：
- **勾** → MA **保证**不与头像上别的同名 gimmick 冲突（构建期唯一化）；
- **不勾** → 终端用户**可改名**；也可**让多个 prefab 共用同一参数**（故意连接）。

**为什么重要**：自己做 prefab 不想撞 → 勾；希望用户能改/能接 → 不勾；读别人 prefab 时参数名带 `$hash` 后缀**正是 Internal 唯一化的结果**，不要试图对齐原始名。

**KB 对照**：KB 只有字段名 `internalParameter`（`02-02:361`）与「烘焙后带 `$hash`」现象（`:767,793`），无此语义 → 回填本节。

**待实测 / 版本差异**：UI 上该能力可能表现为 `Internal`（教程）或 `Change name to`/`Auto rename`（reference 页），实际字段名与位置以本版 inspector 为准。嵌套规则：`Saved` 取**最外层**（非嵌套多个给不同值时**任一 saved 即 saved**）；`Default Value` 取最外层，外层留空取最内层非空，非嵌套冲突会**报警告**。
出处：<https://modular-avatar.nadena.dev/docs/reference/parameters>、<https://modular-avatar.nadena.dev/docs/tutorials/object_toggle_manual>

---

## §9 教程里 KB 没有的官方推荐做法

> 只列 KB 未收的；KB 已覆盖的（Create Toggle 基本用法、Extract menu、BlendshapeSync 基本用法等）不重复。

**9.1 simple clothing**（`/tutorials/clothing`）
- 拖入 → 右键 `[ModularAvatar] Setup Outfit` → Play 验证；**卸载 = 删除对象**（KB 有流程，无此非破坏性语义）。
- **不需要 unpack 头像或衣服 prefab**（KB 有「按需解压」铁律，未写「衣服 prefab 也不要 unpack」）。
- `Blendshape Sync` 的 **`Target blendshape` 可改名，留空 = 用原名**（KB 无）。
- 同步在**编辑器/inspector 与上传时都生效**（与 KB「编辑态不生效」冲突 → §10）。

**9.2 adv clothing**（`/tutorials/adv_clothing`）
- collider 跟随：**多选**所有 `Hips_Collider` → 一次性加 MA Bone Proxy → 把基础 `Hips` 拖进 Target；**Attachment Mode 自动变 `As child; keep position`**（KB 有模式表，无「多选批量 + 自动切换」细节）。
- **反例**：可用 Merge Animator 设基础身体形态键（`torso_thin=100`、`elbow_off=0`、`bra_off=100`）并关原衣服，**但会干扰换装（outfit changer）配置，不推荐**（KB 大量用改形态键绕穿模，缺这条取舍）。

**9.3 object_toggle**（`/tutorials/object_toggle`）
- `Modular Avatar -> Create Toggle` 生成子对象（`Menu Item` + `Menu Installer` + `Object Toggle`）。
- ObjectToggle 点 `+` 拖目标；**不勾 checkbox = 「这一项用于关掉」**。
- **点 Menu Item 的 `Default` 勾选框可立即预览效果**（KB 只知 `isDefault` 是默认值）。

**9.4 object_toggle_manual**（`/tutorials/object_toggle_manual`）
- 录动画时**额外挂普通 Animator** 指向同一 controller（只为让 Unity 允许录），勾 `delete attached animator` 让 MA **构建期删掉**。
- `MA Parameters` 勾 **`Show Prefab Developer Options`** 才列出「现有参数」。
- `Internal` 语义见 §8。
- 组件排序：把用户可能想改的（Menu Installer、未勾 Internal 时的 Parameters）**拖到 inspector 顶部**。
- `MA Parameters` 必须在**所有 Merge Animator / Menu Installer 的同物体或父级**（KB 有等价铁律）。
- 手搓要点：BP `As child; at root`；**删掉 Cube 的 Box Collider**；AnyState 转移 `Duration=0`、`Can Transition to Self` 关（KB 有）。

**9.5 menu**（`/tutorials/menu`）
- `[Modular Avatar] Extract menu` → 菜单变对象；子菜单 `extract to objects` 逐层展开；可拖排序、`Add menu item` 新增（KB 有）。
- 子菜单：`Type = Sub Menu` + `Submenu Source = Children`（KB 有）。
- 复用资产：**Menu Installer + Menu Item 同物体** = 单个控件/子菜单；**Menu Installer + Menu Group** = 不套子菜单装多个控件（extract 系统就这么还原原菜单）（KB 有）。
- 参数名输入框旁**箭头搜索会带上父级 MA Parameters 的声明**（未烘焙参数也能搜到）（KB 无）。

**9.6 advanced-toggles**（`/tutorials/advanced-toggles`）
- ShapeChanger `Delete`/`Set` 语义见 2.6；「缩到几乎为零」用 `Delete`。
- 编辑态会**预览** ShapeChanger 效果；用 **debug overlay → Overdraw** 透视看底下；缩太多在 Scene 里立刻能看出（KB 无 Overdraw 法）。
- **限制**：菜单 toggle 对**形态键的下游联动**当前版本**编辑态不可预览**，要用 **Avatar 3.0 Emulator / Gesture Manager** 进 Play 验（KB 靠 GM 验但没单列这条）。
- **反向提示**：Reactive 系统会响应**其它 NDMF 兼容系统**产生的动画 → 用户**不一定**要用 MA 的 toggle 才能享受作者预设的 ShapeChanger 效果（KB 无）。

---

## §10 KB 现有说法需修正

> 四处均逐行读过（只读，未修改）；按「原文 → 精确化写法」替换。

### 10.1 `SKILL.md` 铁律 1 范围过宽

**原文**：「`ShapeChanger`、`MaterialSetter`、`ObjectToggle`、`BlendshapeSync`、`MergeAnimator` **在普通编辑器场景视图里不生效**。它们只在下面三个时机应用（Play / Manual bake / 上传）……」

**问题**：官方明确 **ShapeChanger 在场景视图实时预览**；**BlendshapeSync 在编辑器/inspector 与上传时同步**；**Setup Outfit 后编辑态转基础骨衣服会跟随**（Position lock 单向）。「这几个组件编辑态全不生效」是**过度概括**，会让人把真实预览当假象。

**建议改法**：
> MA 组件的**最终烘焙结果**只在 Play / Manual Bake / 上传时出现；但**部分组件编辑态就有预览/同步**：
> - `ShapeChanger`：**编辑态实时预览**（可用 Overdraw 查看）
> - `BlendshapeSync`：**编辑态同步**形态键值
> - Setup Outfit 配好的 `MergeArmature`：编辑态衣服**跟随基础骨**（Position lock = 单向）
> - `ObjectToggle` / `MergeAnimator` / 菜单安装：编辑态无最终效果，需 Play/Bake 验证
> 判据：**现象依赖不依赖反应层/合并结果** —— 依赖就 bake，不依赖先看预览。

### 10.2 `00-05-mesh-blendshape.md:9` 同上

**原文**：「MA 的 `BlendshapeSync` / `ShapeChanger` / `ObjectToggle` 在普通编辑器场景视图里不生效。」

**问题**：同 10.1；`BlendshapeSync`/`ShapeChanger` 是**有编辑态效果**的两项，导致「形态键没跟上」被误判为「必须 bake 才能看」。
**建议**：换成分组件说明；并在「诊断穿模前先 bake」处补一句「只查形态键同步/ShapeChanger 预览时，可先在编辑态看」。

### 10.3 `03-菜单与参数/03-06-ma-menu-build-by-reflection.md:556` 改官方规则表述

**原文**：「一个 `m_IsActive` 只允许**一个写者**；MA 响应层与手写层抢同一对象时行为按烘焙顺序，难预测。」

**问题**：把两类事混成一句「难预测」。官方其实给了**两条可执行规则**：
1. **多个 MA 反应组件之间冲突** → **层级最低者胜**（reaction「Priority rules」；shape-changer 同款）；
2. **MA pass 顺序 / NDMF 层顺序不是稳定 API**（versioning-policy）→ 「MA 生成层 vs 手写层」谁覆盖谁**不能靠层 index 推断**，只能烘焙后实测。

**建议改法**：
> - 同一属性**尽量只留一个写者**（最稳）。
> - 必须多个 MA 响应组件抢同一目标时：官方规则是**层级最低者胜**，可据此设计，不必猜。
> - MA 层与**手写/第三方控制器层**抢同一属性时：官方不保证层顺序（pass 顺序与层顺序都非稳定 API），**只能烘焙后实测**；KB 旧观测「后写者胜」只能当本版现象，不能作设计依据。
> - `ShapeChanger` 抢同一形态键同理（层级最低胜）。

### 10.4 `03-菜单与参数/03-05-ma-menu-hair-style-color.md` §4 第 4 条「层顺序」待复核

**原文（约 :40-42）**：「层名会被合并流程保留，但**层顺序**在 FX 末尾追加；确保自己的层在动画目标（`m_IsActive`/`m_Materials`）没有冲突。」

**问题**：「层顺序在 FX 末尾追加」是**单版本观测**；官方明确 MA pass 顺序与 NDMF 层顺序**都不稳定**，写成规则会诱导按层 index 判断（KB 已有因此误判的先例）。
**建议**：
> 自建层烘焙后**通常**追加到 FX 末尾（**本版观测，非承诺**：官方明确 pass 顺序/层顺序都不是稳定 API）。因此：① **不要按层 index 判断**，按层名 + `defaultWeight` + 目标属性断言；② 与 MA 反应层抢同一属性时按 §10.3 处理。

### 10.5 附带：`05-动骨与穿模/05-03-model-anatomy-and-head-swap.md:234`

**原文**：「MA 的 **Setup Outfit** 工具对头发无效（它要靠 humanoid `Hips` 找骨架，头发 rig 只有 Head）。」
**建议**：机制对，补判据原文（`OutfitRoot → [Armature] → Hips`，Hips 名须含基础 hip 骨名），并注明「头发 rig 无 Hips → 判据不满足 → 回退模式也救不了」。

---

## §11 建议落地

> **2026-11-01 并入后**：本节是审计稿原文。**§0–§10 正文已全部落在本文件**；下表「目标文件」列保留作后续把要点回填到旧册的可选清单（其中 `SKILL.md` 铁律 1、`04-外观与表情/04-06-ma-reactive-components.md`、`04-外观与表情/04-05` §4-4 三处**已在本轮直接改正**；此处按**节名**引用，不按行号）。

| 内容 | 目标文件 | 位置 | 优先级 |
|---|---|---|---|
| §1.1–1.14 Setup Outfit 全流程 | `02-工程组织与工作流/02-08-cross-avatar-clothing.md` | 开头新章；与手绑路线做「先试 MA / 不行再手绑」取舍 | P0 |
| §1.3 判据 + §1.5 T/A + §1.8 `.1` + §1.9 PhysBone | `05-动骨与穿模/05-03-model-anatomy-and-head-swap.md` | 骨名章新小节；顺带解释 `05-08:751` 的 `Armature.1` | P0 |
| §10 五处修正 | `SKILL.md`（铁律 1）、`00-基底/00-05-mesh-blendshape.md`（开头口径段）、`04-外观与表情/04-06-ma-reactive-components.md`、`04-外观与表情/04-05` §4-4、`05-动骨与穿模/05-03-model-anatomy-and-head-swap.md` | 直接替换（**按节名定位，不写行号**） | **P0（先改，避免误导）** |
| §3 Manual Bake 产物清理 | `07-验证与排查/07-01-bake-and-validation.md` + `SKILL.md` 收尾清单 | 新小节 + 收尾 +1 行 | P0 |
| §4 错误处理（面板 + 三步排查） | `07-验证与排查/07-01-bake-and-validation.md` | 新小节 | P0 |
| §2.1–2.3 Reactive 规则/优先级/时序 + §2.9 Debugger | `00-基底/00-02-modular-avatar.md` 或 `02-03` §9 | 把「有 Debugger 就用它」扩成总则 | P1 |
| §2.6 Delete/Set + §9.6 Overdraw/下游不可预览 | `00-基底/00-05-mesh-blendshape.md` | 穿模解法章 + 新「ShapeChanger 模式与调试」 | P1 |
| §2.7 MMD relay | `07-验证与排查/07-01-bake-and-validation.md` + `SKILL.md` 症状 +1 行 | 新小节 | P1（PC 专属） |
| §5 安装/升级 + §6 实验特性 | `01-环境与工具链/01-01-environment-and-toolchain.md`、`07-05` 版本节 | 各一段 | P1/P3 |
| §8 Internal 语义 | `03-菜单与参数/03-01-parameter-and-menu-audit.md` + `00-02-modular-avatar.md` Parameters 段 | 参数身份判定处 | P1 |
| §7.1–7.3、§7.5 分发规范 | `02-工程组织与工作流/02-01-workspace-and-work-assets.md` | 新节「分发 prefab 的 MA 规范」 | P2 |
| §9 教程推荐做法 | 分散到 `03-06` / `05-09` / `02-03` | 按表逐条 | P2 |

---

## §12 出处索引

- for-outfit-creators — <https://modular-avatar.nadena.dev/docs/distributing-prefabs/for-outfit-creators>
- Merge Armature — <https://modular-avatar.nadena.dev/docs/reference/merge-armature>
- Outfit Root — <https://modular-avatar.nadena.dev/docs/reference/outfit-root>
- Mesh Settings — <https://modular-avatar.nadena.dev/docs/reference/mesh-settings>
- Parameters — <https://modular-avatar.nadena.dev/docs/reference/parameters>
- Reactive Components — <https://modular-avatar.nadena.dev/docs/reference/reaction>
- Shape Changer — <https://modular-avatar.nadena.dev/docs/reference/reaction/shape-changer>
- VRChat Settings — <https://modular-avatar.nadena.dev/docs/reference/vrchat-settings>
- MMD World Workarounds — <https://modular-avatar.nadena.dev/docs/general-behavior/mmd>
- Manual processing — <https://modular-avatar.nadena.dev/docs/manual-processing>
- Dealing with problems — <https://modular-avatar.nadena.dev/docs/problems>
- Installation issues — <https://modular-avatar.nadena.dev/docs/problems/install>
- How to update — <https://modular-avatar.nadena.dev/docs/upgrading>
- Experimental features — <https://modular-avatar.nadena.dev/docs/experimental-features>
- Distributing Prefabs — <https://modular-avatar.nadena.dev/docs/distributing-prefabs>
- Versioning policy — <https://modular-avatar.nadena.dev/docs/distributing-prefabs/versioning-policy>
- Logo Usage — <https://modular-avatar.nadena.dev/docs/distributing-prefabs/logo-usage>
- FAQ — <https://modular-avatar.nadena.dev/docs/faq>
- Tutorials — clothing / adv_clothing / object_toggle / object_toggle_manual / menu / advanced-toggles：
  <https://modular-avatar.nadena.dev/docs/tutorials/clothing>、`.../adv_clothing`、`.../object_toggle`、`.../object_toggle_manual`、`.../menu`、`.../advanced-toggles`

---

### 附：与 KB 的关系

- 本册已作为**独立分册**并入（`02-工程组织与工作流/02-10-ma-setup-outfit-and-general-behavior.md`），无需再归档或拆散。
- **不重复** KB 已有内容：菜单/开关反射做法、`ControlType` 真值、`isSaved` 三处链、AAO 合并与烘焙审计流程、一键穿衣的 `OutfitSync` 实现——这些在 `SKILL.md` 与 `03/05/07` 已有，本文只在「与官方语义冲突/需解释」处引用。
- **待实测**：§1.5 臂长容差、§8 的 Internal 实际 UI 名、§2.3 一帧延迟在本工程烘焙产物中的层结构、§2.5 生成 ParentConstraint 的具体目标——建议各做一次最小实验后回填，不要照抄。

