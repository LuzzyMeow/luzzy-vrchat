# 动画控制器、MA 快捷菜单与粒子特效

> 定位：**中文实操路径**。原理与成本模型仍以 `03-01-parameter-and-menu-audit.md`、`07-01-bake-and-validation.md`、
> `07-03-provider-playbooks.md` 为准，本分册不重复论证，只补齐「在哪点点什么、填什么值」。

## §0 两条路线的选择（先读这一节）

| 路线 | 做法 | 何时用 |
| --- | --- | --- |
| **A：手搓动画控制器 + 变量 + 菜单** | 录动画片段 → 建 Animator Controller → 建 Expression Parameters → 手搓 Expressions Menu → 用 `MA Merge Animator` 挂上 | 需要**精细控制过渡/条件/参数成本**、需要学原理、菜单形态特殊 |
| **B：MA 快捷菜单** | 选对象 → 右键 `Modular Avatar → Create Toggle for Selection` → 改 `MA Menu Item` | 常规「换衣服 / 换头发 / 开饰品」，**省去录动画** |

**B 的关键限制（教程明确说明）**：快捷功能**只是生成菜单并省去录动画**，
**绑定骨骼仍然需要 `MA Merge Armature` 等组件**——菜单和绑骨是两件事，别指望 Toggle 顺手把骨绑了。

## §1 路线 A：手搓动画控制器

### §1.1 先看 Descriptor 的 Expressions

`VRC Avatar Descriptor → Expressions`：

- **只有一个 `Customize` 按钮** → 点它生成两个空目标框（Menu / Parameters），然后自己建菜单与参数资产；
- **已经有**（示例是 Manuka 这类自带菜单的模型）→ **直接双击 Menu 框里的文件编辑**，
  **但必须保留已有变量**（不要删原模型的参数，除非已确认无消费者）。

### §1.2 菜单资产（Expressions Menu）

菜单文件夹右键 → `Create → VRChat → Avatars → Expressions Menu`。

- 文件里**第一行是参数文件**（不用管），**第二行开始是选项**；
- **每个菜单选项上限 8 个**（右下角加减号增删）——这与 SDK 的
  `VRCExpressionsMenu.MAX_CONTROLS` 一致，参见 `03-01-parameter-and-menu-audit.md` §2.1；
- `Name`：**可中文**，也支持**部分 Unity 富文本标签**；
- 选项 `Type`：`Toggle` 切换器 / `SubMenu` 子菜单 / `RadialPuppet` 径向器（也可用右侧快捷按钮改）；
  `Type` 改成 `SubMenu` 后下方会出现 `SubMenu` 目标框，把子菜单资产拖进去；
- 子菜单文件同样用 `Expressions Menu` 建；
- **最后把「主菜单」资产拖到角色 `VRC Avatar Descriptor → Expressions → Menu`**
  —— 别把子菜单文件放进去。

### §1.3 概念映射（作者原话）

- 分散的动画片段需要「**导演**」= **Animator Controller**；
- 按功能分组 = **Layers（层）**：动作组 / 特效组 / 服装组；
- 参照物 = **变量**。VRChat 三种变量：**Int 存 0–255 整数 / Float 存 −1.0 到 +1.0 小数 / Bool 存真假**；
- **同步容量上限 256 bits**（成本表与易错规则见 `03-01-parameter-and-menu-audit.md` §1）。

### §1.4 制作步骤（按顺序）

1. 建**动画文件夹**（**路径不含中文**）；
2. `Create → Animation` 给**每件衣服**建一个动画片段；
3. `Create → Animator Controller`（如「衣物控制器」），**双击**进入逻辑编辑面板；
4. **录制方法**：创建 `Animation` 文件后**先点录制**，然后改动对象 → Inspector 上带录制标记的字段会被记进该片段；
   `Preview` 按钮退出预览、恢复 T-pose；
5. **录完务必解锁录制窗口**（否则日后改动会录进错误文件）；
6. 复制默认衣物已录好的帧 → 粘贴到其它动画 → 再手动调整（这是最快的做法）；
7. 录完**清空人物身上 Animator 组件的 Controller**（单击目标框按 `Delete`，或点右侧按钮选 `None`）
   —— 控制器要交给 MA 在构建期合并，**不要让它直接挂在人物上生效**。

### §1.5 变量：两个地方都要声明（最容易漏的一步）

**（1）给 VRChat 看的变量组**：菜单文件夹右键 → `Create → VRChat → Avatars → Expression Parameters`。

- 新建的变量组**默认带 3 个「高度自定义表情」用变量，可安全删除**（前提是菜单没用它们）；
- 加变量：名字**不能用中文、不能含空格**；示例 `Type` = `Int`；`Default` = 0；
- **`Saved`**：不勾 → **换世界或重启客户端后变回默认值**（等于每次换图衣服复位）；
- **`Synced`**：不勾 → **只本地改变，别人看不见**；勾选则**占用同步容量**；
- 把变量组拖进 Descriptor 的 `Expressions → Parameters`。

**（2）给 Animator 看的变量**：控制器左上角 `Parameters` → 右上角加号 → 把**名字与类型完全一致**地加进去。

- **大小写敏感**；
- **数值不用管，会被变量组 Default 覆盖**；
- 漏了这一步的表现：菜单点了没反应 / 动画不切。

### §1.6 控制器连线

1. `Any State` 右键 `Make Transition` 连到各衣物动画；
2. 选中该过渡 → Inspector `Settings → Transition Duration` 改成 **0**（立刻切换，不做过场）；
3. `Conditions` 点加号：① 选参数（如「衣物」）② 选比较方式
   （`Greater` / `Less` / `Equals` / `NotEqual`，本例 `Equals`）③ 写对比值（默认衣服 = 0）。

### §1.7 把控制器挂上（MA Merge Animator）

在角色里用于分割衣物的**空对象**上 `Add Component` → 搜 `Merge` → 选 **`MA Merge Animator`**
（**注意是 Animator，不是 Armature**）：

- 把衣物控制器拖进「要合并的 Animator」；
- **`Layer` 类型选 `FX`**；
- **路径模式改成「绝对路径」（Absolute）**；
- **勾「匹配 Avatar 的 Write Default」**。

> 这三项的语义与坑见 `07-03-provider-playbooks.md` §6/§7 与 `02-01-workspace-and-work-assets.md` §4：
> 多 `Merge Animator` 的**顺序与 Write Defaults 必须保留**，否则会改变原层行为。

### §1.8 菜单选项接参数

进衣物菜单的每个选项 → 把 `Parameter` 改成「衣物」、`Value` 改成对应数值（默认衣服 = 0）。
**该选项点击后的作用 = 把变量值改成该数值**，再由 §1.6 的过渡把动画切过去。

### §1.9 验证

用 Gesture Manager 跑起来（进入 Play）→ **角色应直接切到默认衣服**；
逐个点菜单项确认切换。**Gesture Manager 只能证明「编辑器内已解析 / 预览」，
不能证明多人同步与远端可见**——见 `07-01-bake-and-validation.md` §8。

## §2 路线 B：MA 快捷菜单

### §2.1 前置：备份 + 清理旧资产

- **备份时先关闭 Unity**，把项目目录打包；出问题只需解压压缩包并「添加至 ALCOM」即可恢复。
  （包体与还原顺序见 `06-02-backup-and-package.md` 全文。）
- 删掉路线 A 手搓出来的东西：菜单项、参数、以及挂上的 `MA Merge Animator` 组件。

### §2.2 头发菜单

1. `Ctrl` 多选**包括原版头发在内的所有头发状态**，**统一调为关闭**（Inspector 里把所有勾取消）；
2. 在选中状态下**右键 → `Modular Avatar → Create Toggle for Selection`**（最下方自动生成对象）；
3. 选中父级 → 改 **`MA Menu Item`** 组件：**最上方 = 子菜单名称**，**下方 = 各选项名称**；
4. 勾选**想默认启用**的头发。

- **不指定参数时，MA 会给每个头发自动创建独立参数**（各自占用条目名额）；
- 要**互斥切换**（同时只能有一种发型）→ 就在子选项里**手动填同一个参数**（作者示例用「头发的拼音」，
  **这些参数一定要一致**）；原理与 §1.6 的「导演」案例相同。

### §2.3 衣物菜单（与头发有两处不同）

1. `Ctrl` 多选**含原版在内的所有衣物** → **统一调为关闭**；
2. **单击空白取消选中 → 按住 `Ctrl` 只选一件原版衣物**（这是与头发不同的地方）→ 再选中其他衣物；
3. 右键 `Modular Avatar → Create Toggle for Selection`；
4. 改菜单与选项名称、**参数统一填「衣服的拼音」**、勾默认启用的衣服；
5. **默认衣物显示不全** → 选中默认衣服对象，把**其余默认衣物对象**拖进
   `MA Object Toggle` 组件的**「反转条件」**里。

### §2.4 饰品菜单

1. 选中所有饰品 → 统一调为关闭；
2. 右键 `Create Toggle for Selection`；
3. 改菜单信息；**饰品一般是独立开关，无需填写参数**（各自独立），勾想默认启用的即可。

### §2.5 新增一个选项（4 步，作者给的固定流程）

1. 把**新增物品状态关闭**；
2. 在新增物品上右键 `Modular Avatar → Create Toggle for Selection`；
3. 选中新生成的对象，**移除 `MA Menu Installer` 组件**（避免它另开一个入口）；
4. 把对象挪到**对应菜单对象**的子级下，改选项名称。

### §2.6 整理

菜单对象按功能整理区分（头发 / 衣服 / 饰品各一组），并沿用 §6.5 的
「分类空对象 + 加符号 + 眼睛/手指图标锁定」避免误拖。

## §3 粒子系统：眼部特效

### §3.1 贴图

任意绘图软件建 **512×512**、**黑背景 + 白图案**（示例从阿里巴巴矢量图库找白色图案，
下载 SVG 或 PNG 后与黑背景合并）→ 导出 → 导入项目。

### §3.2 材质

1. `Create → Material` → `Shader` 选 **`Particles → Standard Unlit`**；
2. 图案拖进 `Maps` 下的小图标（主贴图槽）；
3. 点 `Maps` 右侧的调色板改颜色 → 适当增大 `Intensity`（会有白色渐变）；
4. **`Rendering Mode` 改 `Additive`**；
5. 勾 Main Options 里的 `Two sided`。

### §3.3 粒子对象

新建**空场景**做特效（可点 Scene 窗口下方 Grid 隐藏格子、右上角 Gizmos 隐藏图标，避免干扰）；
Hierarchy 右键 `Effect → Particle System`，**双击对象看全貌**。

主模块（从上到下）：

```text
Duration           1
Start Lifetime     3.5
Start Speed        0.01      // 让粒子聚成一团
Start Size         0.0075
Simulation Speed   0.8
Scaling Mode       Hierarchy
Max Particles      15
Culling Mode       Always Simulate
```

额外模块（勾选需要的模块后设置）：

| 模块 | 设置 |
| --- | --- |
| `Emission` | `Rate over Time` = `5` |
| `Shape` | `Angle` = `10`，`Radius` = `0.005` |
| `Velocity over Lifetime` | `Linear` 的 `Y` = `0.0005`，`Speed Modifier` = `0.8` |
| `Color over Lifetime` | 在调色板上约 **1/4 与 3/4 处各点一下**（得到 4 个控制点），再把**第一个点与第四个点的 `Alpha` 改成 0**（头尾淡入淡出） |
| `Size over Lifetime` | 选 `Random between two curves`：面板里**左键拖点、线上双击加点、右键删点**，把上下两条线拖成交叉，**越随机越好** |
| `Size by Speed` | 选 `Random between two constants`，值 = `0.25` 与 `0.5` |
| `Renderer` | `Render Mode` 改 `Mesh`；`Meshes` 改 `Quad`；材质拖进 `Material` |

**`Renderer → `Custom Vertex Streams`（关键）**：勾上后场景会「鬼畜」（这是正常现象）→
在列表里**除 `Position` 外全用减号移除** → 再用加号**依次添加 `Color`、`UV` 里的 `UV1`、`UV` 里的 `AnimFrame`**。
（顺序：`Position` 保留 → `Color` → `UV1` → `AnimFrame`。）

### §3.4 装到眼睛

1. 做成预制件放粒子文件夹；
2. 选中人物 → `VRC Avatar Descriptor → Eye Look → Eyes → Transforms` 能看到**两眼锚点**
   （**单击锚点，Unity 会自动定位到眼睛**）；
3. 预制件拖到眼睛子级下 → **`Transform → Reset`**（使其处于眼睛中间）→ 再用移动/旋转/缩放细调；
4. 按 §6.5 建**饰品分类空对象**，**把特效从眼睛挪到饰品分类下**（保持层级干净）；
5. `Ctrl+D` 复制另一只 → 把 **`Position` 的 X** 与 **`Rotation` 的 Y 改成相反数**（左右镜像）；
6. 同时选中两个特效 → 右键 `Create Empty Parent` 建空父级并改名；
7. **此时无法预览粒子** → 给父级 `Add Component → Particle System` 把**所有勾取消**
   （只为占位/让层级结构正确），再加 **`MA Bone Proxy`** 组件使其跟随头部运动。

### §3.5 两个观察

- **运行时看不到粒子特效** → 打开 **Scene 窗口的 `Layers` 筛选**功能（可能被层过滤挡掉了）；
- 改完记得**跑 Gesture Manager 确认跟随头部**再继续。

## §4 动手绑骨速查（与分册 14 §9 配套）

- **古法（父子级）**：`Unpack Completely` → 物品 `Armature → Head` 拖到人物 `Armature → … → Head` 之下；衣物逐层从 `Hips` 往上挪。
- **插件（MA Merge Armature）**：`Ctrl` 同选两个 `Armature` → `Add Component` 搜 `ma` → `Merge Armature` → 物品的 `Merge Target` 指向人物骨架 → **点一次「调整骨骼名称」**。
- **先备份场景**（`Ctrl+D`）再动手，两套做法各做一份，**用实测选择更好的那套**，不要在同一份上一层层叠。

## §5 本节验收清单

- [ ] 路线 A：每个菜单选项都有对应 `Parameter` + `Value`；主菜单资产已拖进 Descriptor 的 `Menu` 槽；
- [ ] 变量组与 Animator `Parameters` **两处名字与类型完全一致**（大小写敏感）；
- [ ] `Saved` / `Synced` 按需求勾选（要跨世界保留就勾 Saved；要别人看见就勾 Synced），并已计入 256 bit 预算；
- [ ] 控制器过渡 `Transition Duration` = 0，`Conditions` 比较方式正确；
- [ ] `MA Merge Animator` 的 `Layer` = `FX`、路径模式 = 绝对路径、已勾匹配 Write Defaults；
- [ ] 人物身上**没有残留直接生效的 Animator Controller**；
- [ ] 路线 B：头发/衣物/饰品的**统一关闭 → 选择 → `Create Toggle for Selection` → 改 `MA Menu Item`** 流程走完；
- [ ] 需要互斥的选项**共用同一个参数名**；默认衣物显示不全时已配置 `MA Object Toggle` 的**反转条件**；
- [ ] 新增选项已**移除自动生成的 `MA Menu Installer`** 并挪到对应菜单下；
- [ ] 粒子：`Custom Vertex Streams` 仅保留 `Position / Color / UV1 / AnimFrame`；父级挂了 `MA Bone Proxy`；左右眼镜像值正确；
- [ ] 全部改动用 Gesture Manager 跑过一遍；**并把「多人可见 / 跨世界保留」留给用户实测**。

## §6 证据与边界提醒

- 本分册的操作属 `UNITY_RESOLVED` / `PROVIDER_PREVIEW` 层；**「Synced 是否真的同步」「Saved 是否真的跨世界保留」
  必须在 `CLIENT_RUNTIME` 层验证**（见 `07-02-evidence-and-ownership.md` §1）。
- **参数成本以构建结果为准**：MA 的自动参数、路径重写后的实际条目，都要在 `NDMF_BUILT` 层复核；
  本分册给的「自动创建独立参数」只是预期行为，见 `03-01-parameter-and-menu-audit.md` §1 的读取链。
- 教程里的组件名（`Merge Armature`、`Create Toggle for Selection` 菜单路径）**随 MA 版本变化**；
  找不到时先查项目里 MA 的版本与菜单，不要盲猜，并在报告里标 `STALE`。

## FX AnimatorController 孤儿子资产清理（2026-10-21 第 58 轮实测）

- **症状**：`Kipfel_FXLayer_AvatarWardrobe.controller` 体积 43.8 MB；`AssetDatabase.LoadAllAssetsAtPath` 返回 **20,446** 个子资产，但从 layer 实际可达仅 **4,974**。
- **根因**：反复增删 layer / state / clip 后，旧的 sub-asset（clip、blendTree、stateMachine、behaviour）仍留在 `.controller` 文件里；Unity **不会**自动回收。
- **正确做法**：
  1. walk 可达集（见下），得到 `HashSet<Object>`；
  2. `LoadAllAssetsAtPath(path)` 中凡不在可达集、且不是 controller 本身的 → `AssetDatabase.RemoveObjectFromAsset(o)` + `Object.DestroyImmediate(o, true)`；**分批 500 个**，
  每批 `AssetDatabase.SaveAssets()`（本例 15,525 个孤儿分 2 批，约 3.8 s）；
  3. 再清可达 clip 的悬空曲线（见下）。
- **可达集 walk 要点（最易漏的一项加粗）**：每个 layer 的 `stateMachine` → `states[].motion`（`BlendTree` 递归 `children[].motion`）
  / `states[].behaviours` / `states[].transitions` / `anyStateTransitions` / `entryTransitions` / 子 `stateMachines`（递归）/**`stateMachine.behaviours`**。
  SM 自身的 `VRCAvatarParameterDriver` 挂在 `stateMachine.behaviours` 上，只扫 `states[].behaviours` 会把它当孤儿删掉。
- **实测**：子资产 20,446 → **4,921**；文件 43,822,350 B → **12,175,311 B（-72%）**；layers=115 / params=437 不变。
- **悬空曲线清理**：对可达 clips 用 `AnimationUtility.GetCurveBindings(clip)` / `GetObjectReferenceCurveBindings(clip)`，
  若 binding.path 在 avatar root 下 `root.transform.Find(path) == null` → `SetEditorCurve(clip, binding, null)` / 
  `SetObjectReferenceCurve(clip, binding, null)`；本例 316 个 clip 删 **386 条**（已删配饰 / 衣服的 `m_IsActive`、材质曲线等）。
- **判据**：子资产数 ≈ 可达数（差值来自指向**外部 .anim 资产**的 motion，本例 4,974 vs 4,921 属正常）；清理后 `danglingCurves=0`；Play 后菜单功能不变（最好跑一遍全量模拟测试）。

---

**相关分册**：环境与 SDK 流程 → `01-02-environment-and-sdk.md`；
参数与菜单成本 → `03-01-parameter-and-menu-audit.md`；烘焙与验收 → `07-01-bake-and-validation.md`；
provider 细节 → `07-03-provider-playbooks.md`；绑定与穿模 → `05-01-outfit-fit-and-clipping.md`；
外观/材质 → `04-01-appearance-and-liltoon.md`；备份 → `06-02-backup-and-package.md`。
