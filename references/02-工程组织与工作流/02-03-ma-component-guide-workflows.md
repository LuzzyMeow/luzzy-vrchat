# MA 组件决策指南与任务工作流

> 本册整理自一套 MA 组件决策与任务工作流规范（源仓库信息见 `ACKNOWLEDGEMENTS.md`）。
> 与既有分册的分工：**本分册是「该选哪个组件 / 每个组件要先检查什么 / 每条任务按什么顺序做」的决策层**；逐组件的实战配方（Extract Menu、Material Setter 用法、形态键轮盘、动骨触发）
> 见 `03-04-ma-menu-and-switch-cookbook.md`；MA 声明 vs 构建产物的判读、NDMF 烘焙与验收见 `07-01-bake-and-validation.md`；参数成本与菜单路由见 `03-01-parameter-and-menu-audit.md`；
> 组织与命名习惯见 `02-02-ma-workflow-habits.md`。
> ⚠️ 组件**显示名**与**序列化字段名**必须按**本机已安装**的 MA 版本核对（本包工作区工程 `LuzzyMeow` 的 MA 版本以 `Packages/vpm-manifest.json` / `packages-lock.json` 为准；上游的字段实测值是在 MA 1.18.7 上做的，见 `07-05 §3`）。

## 1. 组件决策总表（速查）

| 需求 | 组件 |
| --- | --- |
| 带独立骨架的蒙皮衣服/资产，要合并进头像骨架 | **Merge Armature**（+ Mesh Settings） |
| 刚性配件要跟随某个骨骼（眼镜、帽子、耳环、道具、手部 Contact/Collider） | **Bone Proxy** |
| 用层级对象定义表情控件并装进菜单 | **Menu Item**（+ Menu Installer / Menu Group） |
| 需要自定义动画/表情参数（saved/synced/default/Int 语义） | **Parameters**（**先试自动生成**） |
| 把控制器合并进某个 playable 层 | **Merge Animator** |
| 简单的 motion / blend tree 合并 | **Merge Motion / Merge Blend Tree** |
| 身体与衣服的形态键保持同步 | **Blendshape Sync** |
| 模块激活时设置身体形态键 | **Shape Changer** |
| 删/遮身体多边形 | **Mesh Cutter**（+ 顶点过滤器） |
| 整块对象/渲染器开关 | **Object Toggle** |
| 反应式换材质 / 设置材质槽 | **Material Swap / Material Setter** |
| 共享渲染器设置（bounds、probe anchor） | **Mesh Settings** |
| 缩放/贴地修正 | **Scale Adjuster / Floor Adjuster**（先查版本与执行顺序） |
| 分平台差异 | **Platform Filter** |

> **实现方式优先级（本工作区铁律，用户 2026-10-17）**：上表这些需求**全部优先用 MA 组件实现，尽量不用 VRCFury**。碰到带 VRCFury 组件的资产，先按 [02-02 §0.3](02-02-ma-workflow-habits.md) 的对照表换成 MA 等价件再动手。

## 2. Merge Armature（合并骨架）

用于：**自带独立骨架**、需要合并进头像骨架的蒙皮衣服/资产。

先检查：

- 衣服骨架根 与 头像的 merge 目标；
- 骨骼**名字是否匹配**、前缀/后缀；
- 根变换、缩放、姿态差异；
- SkinnedMeshRenderer 的 `bones` / `rootBone` 引用；
- PhysBone、Contact、Constraint、动画的引用；
- 是否已存在**嵌套**的 Merge Armature 组件。

不要在**单骨骼刚性配件**上用（那是 BoneProxy 的活）；也不要因为「骨骼名能对上」就假设**给别的头像做的衣服**能穿上 —— 适配性是几何/权重问题，不是名字问题（参见 `05-01-outfit-fit-and-clipping.md`）。

## 3. Bone Proxy（骨骼代理）

用于：**刚性模块对象**要挂到某个已有头像对象/骨骼下，同时保持引用可移植。

典型用途：

- 眼镜、帽子、耳环、道具；
- 挂在手/身体骨骼上的 Contact、Collider；
- 用**人形骨骼名**做目标的通用配件。

检查：附加模式（attachment mode）、是否保持位置/旋转、缩放行为。

**不要**拿它替代蒙皮衣服的 Merge Armature。

## 4. Menu Item / Menu Installer / Menu Group（菜单）

用于：用**层级对象**定义表情控件并安装进头像菜单。

**绑定规则（重要）**：MenuItem 只有在下面三种情况之一才会被安装 ——

1. 与某个 **Menu Installer 在同一个节点**上；
2. 是某个 **Menu Group 的直接子节点**（通常 Menu Group 就在 Installer 节点上）；
3. 是某个父 MenuItem 的子节点，且该父项处于 `MenuSource=Children` 模式。

**没有 Installer / MenuGroup / 父链的 MenuItem 是「未绑定」的，会被静默忽略** —— 构建后的菜单缺项时，先查绑定。

Menu Installer **默认装到头像表情菜单的顶层**。要让 MA 完全接管根菜单：新建一个空的 `VRCExpressionsMenu` 资产，AvatarDescriptor 的 `expressionsMenu` 指向它，并把 Installer 的 `menuToAppend` 设为 null。此时**原菜单必须手动折进来**（例如用 `原有菜单` 子菜单引用原资产），否则会丢。

检查：

- 该 MenuItem 是否**真的**绑到了目标菜单；
- 控件类型与值；
- 子菜单来源（`Children` vs 菜单资产）；
- 参数名/类型/default；
- 是否有**重复的 default**、菜单页是否溢出；
- 图标资产是否兼容（见 `04-02-expressions-and-icons.md`）。

MenuItem 在参数未声明时可以**自动创建参数**。优先用自动参数（`__MA/AutoParam/...`）；只有 saved/synced/default/Int 语义重要时才用显式 Parameters 组件。

## 5. Parameters（参数）

用于：需要**显式语义**的自定义动画/表情参数 —— 但**仅在自动生成不够时**。

**自动优先**：`param` 留空 + `automaticValue=true` 的 MenuItem 会让 MA 自动创建 `__MA/AutoParam/<name>$hash`；能复用头像原有 Parameters 资产就复用。下面这些情况才用 `ModularAvatarParameters`：

- 互斥需要显式 Int 值（2 套以上套装共享 `cloth`/`hair`）；
- 可复用模块必须自带 saved/synced/default/改名语义；
- 参数必须是 local-only、带前缀、或只给 Animator 用。

检查：

- 本版是否支持 Bool/Int/Float/Animator Only/Prefix 这些语义；
- saved 与 synced 标志；
- default 与 animator-default 的覆盖行为；
- 自动改名 / local-only 行为；
- 重名与参数预算；
- 本版是否支持 PhysBone/Contact/Raycast 的参数前缀。

**永远不要**只从菜单控件反推参数类型；要把所有引用它的动画与驱动都读一遍。
单一套装的 `cloth` 自动生成 Bool 没问题；**第二套来了就升级成 Int**。
## 6. Merge Animator（合并动画控制器）

用于：把控制器合并进某个头像 playable 层，**不直接编辑原控制器**。

检查：

- 目标 playable 层；
- 路径模式（path mode）与动画绑定路径；
- 参数声明；
- **Write Defaults 策略**；
- 层控制/状态行为引用；
- animator mask 与默认状态；
- 与其它模块的**先后顺序**。

## 7. Merge Motion / Merge Blend Tree

用于：行为确实能用 motion/blend tree 表达时。

- 核对**本版**的组件名与限制；
- **不要**把复杂状态机行为硬塞进 blend tree。

## 8. Blendshape Sync（形态键同步）

用于：目标渲染器的某个形态键应跟随源渲染器的同名/对应形态键。

检查：

- **源/目标渲染器是否准确**；
- 形态键**名字是否存在**（精确匹配）；
- 同步**方向**；
- 是否存在链式（A→B→C）—— 除非本版支持，否则避免；
- 眨眼/viseme 行为能否被准确表达；
- 本版是否支持曲线重映射（remap）。

## 9. 反应式组件（Reactive）总则

反应式组件响应对象的激活状态或关联的菜单状态。

- 先读**本版**的 Reactive Component 规则；
- 有 Reaction Debugger 就用它。

## 10. Shape Changer

用于：某个套装/模块激活时设置**身体**形态键 —— 常见于把衣服下的身体区域缩掉/藏掉。

- **不要**用它去对抗**持续动画同一个形态键**的其它动画（会互相打架）；
- 校验阈值（threshold）与删除（delete）行为，并**预览**结果。

## 11. Mesh Cutter

用于：删除或隐藏被选中的多边形，配一个或多个**顶点过滤器**。

过滤器可选：

- 遮罩贴图（mask texture）；
- 轴/平面（axis/plane）；
- 骨骼权重（bone weights）；
- 形态键位移（blendshape movement）。

**精确切割优先用 mask**；轴过滤更适合粗放地取一侧/一块区域。
**如果整个渲染器都该消失，就用 Object Toggle，不要走 Mesh Cutter。**

## 12. Object Toggle

用于：启用/禁用**完整对象或渲染器**。

比走 Mesh Cutter 处理整块网格**更便宜、更好验证**。

## 13. Material Swap / Material Setter

用于：整体替换材质，或反应式设置渲染器材质槽。

- 语义与**材质槽匹配**规则要按本版核对；
- **复制出用户可编辑的材质**，不要去改导入的原始材质。

## 14. Mesh Settings

用于：共享渲染器设置，例如 bounds 与 probe anchor 的一致性。

检查：这些设置是否**沿层级继承**；是否有别的头像优化器/构建插件会改同一批属性。

## 15. Scale / Floor Adjuster

只有在确认了**本版存在**且**插件执行顺序**正确之后才用。

缩放后必须验证：视角（viewpoint）、脚/地面贴合、PhysBone、Contact、Constraint、以及衣服是否还合身。

## 16. Platform Filter 与分平台工作

**只在工程确实有意做桌面/移动差异时**才用平台过滤。

- **两个分支都要验证**，以及任何 VRChat 分平台覆盖头像；
- **永远不要**假设「非激活平台那条路径」是有效的。
## 任务工作流 A–H

## 17. 总则

每条工作流都遵守同一个骨架：**先记录状态 → 只在自己拥有的对象上写 → 走原生 `unityMCP_*` 工具 → 复读 → 预览/构建 → 看 Console → 报告证据层**。

- **A0「把存量头像 MA 化」** 见 `02-02 §11`（本册不再重复）。
- **加套装/发型九步** 见 `02-02 §12`；本册 A 是它的**前置兼容性判定与检查清单**版本，两者互为补充。

## 18. 工作流 A：安装一套兼容的衣服

1. 定位头像根 与 衣服 prefab 实例。
2. **确认这套衣服是给目标头像/该版本做的**（尺寸与骨架体系；不确定就停）。
3. 记录 prefab 与场景状态；**不要改导入的源 prefab**。
4. 自己的 wrapper/模块对象放到**用户自己的**文件夹/层级里。
5. 检查衣服骨架、renderer 的 `bones`、`rootBone`、材质、形态键、PhysBone、Contact、Constraint。
6. 在正确的**源骨架根**上加/配 Merge Armature（先解析确切字段）。
7. 核对 merge 目标、前缀/后缀、位置锁定选项、**重名处理行为**。
8. **只在源与目标都存在同名形态键时**才加 Blendshape Sync 条目。
9. **只对真实穿模区域**加 Shape Changer 或 Mesh Cutter；优先非破坏性的身体改动。
10. 需要的话，为套装可见性加菜单/参数控件。
11. 在编辑/播放模式预览，试几个身体姿势，看 Console，跑构建校验。

**停止条件**：如果骨骼名、rest pose 或权重映射表明这套衣服**需要 Blender 处理**，就停下报告，不要在 MA 层叠补丁。

## 19. 工作流 B：挂一个刚性配件

1. 确定目标人形骨骼 或 稳定的目标路径。
2. 在**导入内容之外**创建模块对象/prefab。
3. 摆好配件的位置，确定意图中的世界/局部姿态。
4. 加 Bone Proxy，选合适的附加模式与缩放行为。
5. 测试头像骨骼运动、缩放变化，以及**镜像/左右手**假设是否成立。
6. 如果要能开关：加层级菜单控件并核对参数声明。

## 20. 工作流 C：加一个对象开关

1. 确定**确切的目标对象**与期望的默认状态。
2. 选一个唯一的参数名，判断该用 Bool 还是 Int 语义。
3. 够用就优先 **MenuItem + 反应式对象行为**。
4. 需要显式 saved/synced/default/改名语义时才用 Parameters。
5. 核对**没有**别的 animator、父级激活状态或构建插件覆盖这个目标。
6. 测 开/关/**默认** 三种状态，以及菜单安装路径。

## 21. 工作流 D：加一个材质变体

1. 把材质**复制**进自己的资产目录，保留原件。
2. 确认 shader 与贴图依赖。
3. 按已安装版本支持情况选 Material Swap/Setter 或走 animator 的模块。
4. **逐 renderer/材质槽显式映射**。
5. 核对回退行为与移动/平台差异（shader 差异）。
6. 测所有变体，检查生成的动画与材质引用。
## 22. 工作流 E：同步身体与衣服的形态键

1. 枚举源/目标 renderer 的形态键**名字与索引**。
2. 建一张**显式映射表**。
3. **拒绝只靠名字相似**、没有视觉验证的映射。
4. 加 Blendshape Sync 条目。
5. 查**本版**是直接复制数值，还是按曲线重映射。
6. 测代表性数值：**0、50、100**，以及头像用到的**负数/超过 100** 的值。
7. **viseme、眨眼/视线、被动画驱动的形态键分开验证**。

## 23. 工作流 F：处理身体穿模（决策顺序）

1. **整个身体 renderer 都被盖住** → Object Toggle。
2. **已有身体隐藏形态键** → Shape Changer。
3. **需要精确的永久/动态多边形区域** → Mesh Cutter + 合适过滤器（遮罩贴图优先）。
4. **拓扑/权重根本不对** → 回 Blender 改，**不要**在 MA 里堆越来越复杂的规则。

永远要测**极端姿势**与身体的**自定义形态键**（体型滑条拉满/拉空）。

## 24. 工作流 G：合并一个动画控制器 / 小工具

1. 检查控制器的层、参数、动画绑定、状态行为、mask、Write Defaults。
2. 确保每个自定义参数都是**有意声明**的。
3. 选正确的 playable 层 与 路径模式。
4. 把 Merge Animator 挂在**自包含**的模块上。
5. 构建后校验**层顺序与引用**。
6. 测参数默认值、过渡、被打断、重置行为。

## 25. 工作流 H：排查构建报错

1. **先保存改动前的 Console 基线**（准确文本）。
2. 用**最小的模块**复现（逐个启用/禁用）。
3. 用 stack trace 与 **NDMF 构建报告**定位插件/pass/组件。
4. 检查引用：丢失对象、prefab stage 路径、不支持的组件组合、版本不匹配。
5. 把**已安装的 MA/NDMF/VRCSDK/VRCFury 版本**与官方兼容性说明对照。
6. **禁用或复制**可疑模块；**不要删除原始资产**。
7. 重新构建，比较新错误与基线。
8. 报告根因、证据、可逆的修复方案。

## 26. 与既有分册的分工（避免重复劳动）

- 组件**怎么选**、每条任务**按什么顺序做** → 本分册。
- 组件的**逐字段实测字段名与序列化写法** → `02-02 §7/§8` 与 `03-04`（那里有 `Control.type`/`MenuSource` 的实测枚举、ObjectToggle 的 `Object.targetObject` 写法）。
- 烘焙后该信哪一层、怎么验收 → `07-01-bake-and-validation.md`。
- 参数成本、菜单路由、四表面同步 → `03-01-parameter-and-menu-audit.md`。
- 穿模的几何验收标准 → `05-01-outfit-fit-and-clipping.md`（本分册 §23 只给 MA 侧的决策顺序）。
- 证据层级与授权边界 → `07-02-evidence-and-ownership.md`（本包口径）与 `07-05`（外部规范的差异口径）。
