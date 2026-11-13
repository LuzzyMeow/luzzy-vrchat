# Provider 分册：FaceEmo / lilycalInventory / DressingTools / Gesture Manager / Avatar Optimizer

读这份文件的时机：任务涉及某个具体 provider 的配置、排查或迁移。

> **使用前提**：只在 `TOOLCHAIN_PROFILE` 里**选中了该 provider**，或资产明确要求它，或它**已经在确切目标上拥有**任务相关功能时，才按对应分册办事。
> **并且**：组件类型、字段名、生成路径一律**从已安装版本和实际工程里发现**（经 Unity MCP 读），**不要**照抄别的工程/别的版本的 Launcher target、prefab GUID、clip 路径或生成目录。

---

## 1. FaceEmo（表情）

### 1.1 先建立「源 vs 生成」的所有权链

```text
确切目标 Avatar
  → FaceEmo 场景配置 或 Launcher target
  → 可编辑源 prefab / 配置
  → 稳定的用户 clip 与 additional objects
  → provider Apply 或重生成
  → 生成的菜单、参数、FX
  → NDMF 合并后的 Avatar
  → 具名运行时行为
```

- **编辑入口 = 配置或源 prefab**；
- **带时间戳的 controller/menu/参数/备份/预览/构建克隆 = 生成输出**（除非已安装 provider 明确文档化为可编辑）；
- **自定义 clip 放在稳定的用户自有目录**，保证重生成不会替换它。

### 1.2 Apply / 重生成之前必须记录

- 确切的 Avatar 层级路径与 descriptor；
- 配置/Launcher 组件及**直接目标引用**；
- 可编辑 prefab/配置资产，以及场景实例上**是否有覆盖**；
- 当前生成目录与 Avatar 实际引用的资产；
- **所有共享**同一源 prefab/controller/menu/参数/clip 的**其它 Avatar**；
- 包版本、场景 dirty 状态、Console 基线、获授权的动作。

⚠️ **不要**把「目标路径字符串为空」当作「没有指定 Avatar」的证明：provider 可能用对象引用，或另一个版本文档化的目标字段。

### 1.3 表情职责记录（每个表情/面部功能一份）

```text
显示名与玩家入口
  → 手势 / GestureWeight / viseme / Contact / AFK / 菜单 等条件
  → pattern、阈值、过渡规则
  → 稳定源 clip 或生成 motion
  → 确切的 BlendShape / transform / 材质 / 粒子 / 音频 / 对象绑定
  → 与 Blink、眼追踪、Lip Sync、mouth-cancellation 的交互
  → 生成的层/状态与最终 NDMF 归属
```

**要列出任务相关 clip 里的每一条曲线，包括值为 0 的曲线。** 一个「满脸」clip 可能在只想做一个表情时，顺手把眼睛、嘴、耳、尾、材质、道具都重置了。

### 1.4 只读排查顺序（7 步）

1. 锁定目标 Avatar 与确切的 Unity MCP 实例；
2. 解析当前活跃的 FaceEmo 配置及其可编辑源；
3. 清点表情定义、条件、源 clip、additional objects、直接/共享消费者；
4. 检查 Blink / 眼追踪 / Lip Sync / 手势 / viseme / Contact / AFK / 菜单 的**重叠绑定或条件**；
5. **只把**当前的生成菜单、参数、FX 当**输出证据**看；
6. 把 provider 预览或 NDMF 输出与「职责记录」对比；
7. 找出**第一处**分叉。

没有获授权做预览/构建时，**停在生成结论之下**，标 `BUILD_REQUIRED` 或 `NOT_RUN`。

### 1.5 新增/编辑单个表情（7 步）

1. 快照当前源配置、生成引用、全部共享消费者；
2. 选定**确切**的表情条目与条件。**不要**因为「看起来像」就复用另一个条目；
3. 编辑或创建稳定源 clip：**只包含有意为之的绑定**；逐条复查被继承/复制来的 0 值曲线；
4. 针对 Blink、眼追踪、Lip Sync、mouth cancellation、gesture weight、层顺序、mask、过渡、Write Defaults 逐项检查；
5. 目标与共享影响确认后，**只走 provider 支持的路径** Apply/重生成；
6. 检查新的生成菜单、参数、层/状态、motion 与**确切模型绑定**；
7. 跑获授权的预览/运行时层，验证你要声称的那个行为。

**不要**把整个生成状态或带时间戳的 clip 拷进用户源目录当捷径——要从稳定源绑定重建意图。

### 1.6 手势/菜单表情不触发（顺序查）

1. 确认玩家入口或手势 pattern 与期望参数值；
2. 确认 GestureWeight 阈值、过渡条件，以及左右手/组合手势语义；
3. 确认生成参数的**类型/默认值**与最终 FX 消费者；
4. 检查状态过渡、打断、层权重、mask、Write Defaults，以及**更高优先级状态**是否立刻替换掉表情；
5. 把每条 clip 绑定路径拿**构建后的 Avatar 层级**解析；
6. 检查是否有**另一个表情所有者或自定义 FX 层**写同一 BlendShape/transform/材质/对象。

修**可编辑的条件、clip 或源配置**。不要打补丁到生成 controller 上——重生成会抹掉。

### 1.7 Blink / Lip Sync / 眼追踪坏掉的六类成因

- 表情 clip **显式写了**眼睛或嘴的曲线；
- cancellation / protection 设置**缺失或过宽**；
- 某个过渡或 Write Defaults 状态**重置**了曲线；
- **另一个 provider 或自定义 FX 层有更晚的所有权**；
- 复制来的满脸 clip 里含**非预期的 0 值曲线**；
- 层级重映射后**构建绑定路径解析不到**。

改**拥有该冲突的最小稳定源**。保留有意的非对称性与非面部绑定。**Blink / Lip Sync / 眼追踪必须各自独立验证**——一个通过不证明另外两个。

### 1.8 目标头像不对 / 共享 prefab / 跨头像污染

Apply 之前先解析：目标对象引用、源 prefab GUID、场景覆盖、每一个消费者。多 Avatar 共享源时，**明确决定**这次改动是共享还是隔离。只改一个目标时，优先做 **Avatar 专属的源 prefab 或稳定 clip**。

重生成之后，验证**无关 Avatar 保持原来的源引用与生成所有权**。不要靠层级相似比较，也不要抄别的 Avatar 的 target 路径。

### 1.9 生成资产缺失 / 过期 / 重复

1. 确认确切 provider 配置与当前引用的生成资产；
2. 确认包的导入/编译状态，把无关 Console 错误隔离出去；
3. 检查生成是否完成，以及 Avatar 现在引用的是新输出还是**旧的带时间戳目录**；
4. 检查重复配置、过期场景覆盖、共享源 prefab、provider 顺序；
5. 源与目标都证明之后，**只走受支持路径**重生成。

被替换掉的生成路径标 `STALE`；**除非清理被明确授权且所有消费者都已映射，不要删它们。**

---

## 2. lilycalInventory（衣柜/道具库存）

### 2.1 建立衣柜 provider 链

```text
穿戴物源 或 道具
  → 库存 host/settings 与 provider 组件
  → 生成的 CostumeChanger / ItemToggler 或等价所有者
  → 菜单生成器与菜单位置
  → 参数与 Animator
  → 骨架、renderer、材质、对象职责
  → NDMF 合并后的 Avatar
  → 具名运行时行为
```

**可编辑源** = 穿戴物、provider 组件、库存设置、稳定预设、用户自建资产。**输出** = 生成的菜单资产、controller、参数、构建克隆、NDMF 结果。

### 2.2 诊断或改动之前必须记录

- 确切的 Avatar 与穿戴物层级路径；
- 包版本与**已安装的组件类型名**；
- 库存 host/settings，以及**要求的 host 是否唯一且活跃**；
- 穿戴物源 prefab、场景覆盖、激活/默认状态、provider enabled 状态；
- 菜单文件夹/位置、生成控件名、参数名/类型/默认值、saved/sync 行为、remap；
- 对象开关、预设、材质、骨架、网格、BlendShape、constraint、PhysBone、Contact、非视觉效果；
- **每一个** Avatar、prefab、菜单、参数资产、材质、网格、clip 消费者。

⚠️ **一个非激活的服装对象，只要它的 provider 组件 enabled，仍然可能参与构建。** 在检查过 provider 语义与构建结果之前，不要把「源里非激活」归类为失败。

### 2.3 服装职责图（每个穿戴物，每项职责一个所有者）

| 职责 | 必须记录 |
| --- | --- |
| 玩家入口 | 完整菜单路径、控件类型/取值、图标、位置所有者 |
| 参数 | 名称、类型、默认、saved/sync 意图、remap、最终消费者 |
| 可见性 | 源激活态、生成开关、Animator 状态、平台过滤 |
| 贴合 | 骨架/根、骨骼映射、renderer root bone/`bones`、constraint、缩放 |
| 网格 | renderer、mesh、材质槽、BlendShape、BlendShape 同步 |
| 物理 | PhysBone/Contact 根、碰撞体、过滤、参数、运行时依赖 |
| 预设 | 包含项、互斥性、默认、与单项开关的交互 |
| 生成结果 | 最终菜单、参数、Animator、层级、NDMF 归属 |

**看得见的衣柜菜单只证明菜单可达。** 它不证明贴合、穿模、动作、saved 状态、同步或物理正确。

### 2.4 只读排查顺序（6 步）

1. 锁定确切 Avatar 与 Unity MCP 实例；记录场景、编译、Console 状态；
2. 从玩家衣柜菜单或具名穿戴物出发，定位**确切的 provider 源**；
3. 解析库存 host/settings、provider 组件、穿戴物源、prefab 覆盖、共享消费者；
4. 建立**职责图**，包括**归别的 provider 所有**的那部分职责；
5. 检查 provider 预览或 NDMF 输出中的生成菜单、参数、Animator 层、对象开关、骨架与层级变化；
6. 对比源记录与生成结果，找**第一处分叉**。

### 2.5 加一件穿戴物（8 步）

1. 确认资产作者**支持的安装路径**与**必需 provider**；
2. 记录穿戴物的源 prefab、骨架/网格要求、菜单、参数、材质、物理、预设、非视觉效果；
3. 确认目标库存 host 与预期菜单位置。**已安装版本要求唯一共享 host 时，不要创建第二个 host**；
4. 加/配 **provider 支持的最小**源组件；
5. **有意识地**选择参数/默认值/saved/sync 行为：不要不检查意图与最终预算就继承旁边穿戴物的值；
6. 保留 provider 要求的源激活态语义；
7. 预览/构建，检查最终菜单、参数、Animator、层级、骨架、renderer、开关结果；
8. 跑获授权的贴合/可见性/运行时检查，然后才能说这件穿戴完成。

记录**导入的源占用**。除非另行授权做了**确切可比的体积构建**，边际 SDK bundle 体积写 `NOT_MEASURED`。

### 2.6 菜单项缺失或放错文件夹（顺序查）

1. provider 源与库存 host 是否 enabled 且关联到选中 Avatar；
2. 该条目是否注册在**预期的库存/设置源**里；
3. 菜单文件夹/位置引用是否**有效**、且不是过期的场景覆盖；
4. 生成的控件是否有**兼容的参数和取值**；
5. 检查 NDMF 输出的最终位置、页容量、顺序，以及与另一个菜单 provider 的所有权冲突；
6. 在获授权的预览/构建层验证完整玩家路径。

修 **provider 源或位置声明**。**不要**手工把条目加到「重生成时会替换掉」的生成菜单里。

### 2.7 默认值 / saved / 互斥不对（各自独立解析）

- 源对象激活态；
- provider 条目默认值；
- 生成的 Animator 默认状态；
- 参数默认值、saved、synced 或 local 行为；
- 预设成员关系与互斥组规则；
- **另一个 provider 或手工 FX 状态**是否写了同一对象。

检查**构建后的**参数与 Animator 结果。测默认值要用**全新运行时状态**；测持久化要**第二次启动或显式重置**；测远端同步要用**具名多人测试**——本地预览不是网络行为的证据。

### 2.8 穿戴显示但贴合/动作不对

检查源与构建后的确切值：

- 源/目标骨架根与骨骼映射；
- renderer root bone、`bones` 数组、mesh 绑定；
- 变换、缩放、constraint、Bone Proxy 或 Merge Armature provider；
- BlendShape 与 BlendShape 同步；
- 材质槽、裁剪 mask、遮体（body-hiding）职责；
- PhysBone/Contact 根、碰撞体、过滤；
- 共享的 FBX、网格、材质、prefab 消费者。

**不要**用菜单切换成功当贴合证据；**不要**从结构相似的头像抄骨骼/网格/身体引用；共享源不该为所有消费者改变时，**隔离出目标专属覆盖**。

### 2.9 从别的衣柜 provider 迁移（6 步）

1. 快照旧 provider，保留恢复点；
2. 为菜单、参数、预设、骨架、网格、BlendShape、材质、物理、Contact、音频、对象**每一项职责**完成职责图；
3. 把每项职责指派给 lilycalInventory、另一个 provider、或**有意为之的手工所有者**；**不留意外的双重所有权**；
4. 在**旧源仍可恢复**的前提下构建并验证新 provider；
5. 对比菜单路径、默认值、参数成本、贴合、saved/sync 行为、运行时效果；
6. 只有替代实现通过要求的证据层**且用户授权移除**之后，才移除旧 provider。

⚠️ **不要**仅仅为了「统一」就去迁移一个已经有意选定且工作正常的 provider。

### 2.10 参数与限额

对**确切选中的 Avatar 独立计算**：清点 descriptor 参数 + 每个 provider 的贡献，然后用**已安装的 SDK/provider API 或工程自有的报告**取最终构建成本。**不要数参数名估 256-bit 预算，也不要从相似根抄结果。**

**接近容量是正常的。** 只有在①确认硬溢出、②请求的功能放不下、③用户明确要求时，才进入优化。

---

## 3. DressingTools（衣柜/穿戴的另一套 provider）

**可编辑源** = 衣柜或穿戴物配置，包含骨架映射、网格设置、BlendShape 同步、对象开关、constraint、菜单设置。

用库存 / Modular Avatar / 其它 provider 替代它时，**必须先逐项重建每一项职责**，再移除 DressingTools 源。

**不要**仅凭「菜单里有这一项」或「穿戴物是激活的」推断成功。要在目标根检查：**骨骼路径、BlendShape 绑定、renderer、材质、PhysBone、Contact、默认值**。

---

## 4. Gesture Manager（编辑器内的验收工具）

用途与边界见 [07-01-bake-and-validation.md](07-01-bake-and-validation.md) 第 8 节。要点复述：

- **它是必装项，不是可选项**（用户 2026-10-17 强调）：新建 / 重建工程后先核实 `vrchat.blackstartx.gesture-manager` 已在 `Packages/vpm-manifest.json` 里，之后每轮改动都在它的 Play 模式过一遍；
  安装与打开步骤见 [../01-环境与工具链/01-06-project-launch-and-mcp-bridge.md](../01-环境与工具链/01-06-project-launch-and-mcp-bridge.md) §2.7。
- 它是**编辑器内模拟器**，用来快速验收手势→参数映射、FX 层合并、对象开关档位、表情/滑块、默认外观、穿模 A/B；
- 它**不证明**多人同步、远端可见性、Contact 交互、saved 跨世界持久化；
- 进客户端前，先用它把能排的排干净，能显著减少 Build & Test 往返。

---

## 5. Avatar Optimizer（AAO）

**可编辑源** = 优化器设置/组件，以及原始的网格、材质与 Avatar 配置。

优化**可能**在构建克隆里**移除、合并或重写网格、材质与组件**。**优化后的结果不是稳定的源资产。**

任何依赖「优化前路径」的动画、材质、BlendShape、PhysBone 或 Contact 路线，都**必须**拿 `NDMF_BUILT` 结果核对。典型风险：

- 动画路径指向被合并掉的中间对象；
- 材质槽被合并后 shader 属性写在别的槽上；
- PhysBone 根被移动/删除；
- BlendShape 被优化掉后动画写不存在的 shape。

---

## 6. 通用 provider 判断（不属于上述任何一家）

遇到未列出的 provider，用同一套问题定位它的角色：

1. 它的**可编辑源**是什么（组件 / 源 prefab / 配置资产 / 用户 clip）？
2. 它**声明**了哪些职责（菜单 / 参数 / Animator / 骨架 / 网格 / 对象开关 / 物理）？
3. 它**在哪一个 NDMF pass / 顺序**上产出结果？
4. 它的**生成输出**放在哪个目录，重生成会不会覆盖？
5. 它和现有 provider 有没有**重叠职责**（同一参数/层/对象/菜单位置）？
6. 它要求的最少验证层是什么？

**包存在 ≠ 功能归它管。** 归属跟着**声明与写入路径**走。

---

## 7. 各 provider 共同的改动护栏

**改之前**说明：确切条目、源组件、目标 Avatar、菜单目的地、共享消费者、获授权的操作、必需的验证层。**Apply / 改 prefab / refresh / preview / build 之前重新确认 Unity 实例。** 保持**一个** Unity 所有者负责 Apply、导入、编译、预览与构建。

**改之后**：

1. 等导入/编译完成，检查**任务引起的** Console 变化；
2. 走 provider/NDMF 路径**重生成**；
3. 对比「源职责记录」与生成的菜单、参数、Animator、层级、骨架、网格、开关；
4. **只跑获授权的**运行时与体积层；
5. 更新工具链档案、菜单 map、参数索引、provider 清单、职责图、生成源记录、源占用、体积证据状态、验收记录。

## 8. 各 provider 的验收矩阵（合并版）

| 你要声称的 | 最低证据 |
| --- | --- |
| FaceEmo：源表情已配置 | `UNITY_RESOLVED` 确切配置、条件、clip、目标 |
| FaceEmo：生成菜单/参数/FX 里有该表情 | provider 预览或 `NDMF_BUILT` 生成输出 |
| FaceEmo：clip 绑到预期的模型属性 | **解析后的构建绑定**，不是 clip 名字 |
| FaceEmo：Blink / Lip Sync / 眼追踪保留 | **每一项**各自的具名预览/运行时测试 |
| 库存：条目已为目标配好 | `UNITY_RESOLVED` 确切 host、源、provider、目标 |
| 库存：菜单/参数/Animator 已生成 | provider 预览或 `NDMF_BUILT` 最终结果 |
| 库存：骨架、renderer、开关绑定正确 | `NDMF_BUILT` 确切路径与绑定 |
| 库存：贴合/穿模/动作/可见性正确 | 具名获授权的运行时层 |
| 库存：saved/sync 有效 | 全新持久化测试，或具名多人测试 |
| 库存：bundle 体积影响可接受 | **确切获授权的** SDK 构建对比；否则 `NOT_MEASURED` |

**每一个没跑的层都显式报告。** 源里可见对象、provider 预览成功、Console 干净，都不是最终玩家侧的证据。
