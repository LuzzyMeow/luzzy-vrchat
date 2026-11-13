# MA 菜单 / 参数 / 动画器补充：MergeAnimator 层序、Merge Motion、跨平台同步、MMD

> **适用**：多个 MergeAnimator 层序互相压制、要用 `VRCAnimatorLayerControl`、做 PC/Quest 同参数跨平台上传、头像要进 MMD 世界、要用 PlatformFilter。
> **来源**：Modular Avatar 官方文档（<https://modular-avatar.nadena.dev/docs/intro> 起 67 页），2026-11-01 逐页抓取原文（MA 1.18.7，changelog 2026-09-01）；本地副本 `<VRCHAT_ROOT>\tools\tmp\ma-docs-txt\`。每节带官方 URL，可逐条核对。
> **定位**：本册是**官方语义参考**（字段 / 默认值 / 限制 / 烘焙期行为 / 组件间交互），与 `00-基底/00-02-modular-avatar.md` 的代码范式互补；基底是外部 MIT 原件，**一字未改**，本册不重复其已有内容。
> **标记**：【文档未说明，需实测】= 官方没写、须本工程验证；【需查源码】= 字段名未公开、需反射；【待复核】= 与 KB 现有表述冲突、需人工定稿；【KB 已有】= 只补语义。组件名 / 字段名保留英文原名。
> **本册来历**：2026-11-01 第 135 轮「MA 官方文档 × KB 缺口审计」的成品稿整册并入（审计报告在 `<VRCHAT_ROOT>\tools\tmp\ma-gap\`）。稿内「建议落地」一节保留为历史记录，落点已在并入时确定。

## 目录

| 节 | 内容 |
|---|---|
| §1–§2 | `MergeAnimator`（层序 / Replace / WD / pathMode / LayerControl）、`Merge Motion`（单 FX 层合并、等时长坑） |
| §3 | `VRChat Settings` + MMD 行为（FX 2/3 层、relay、padding、MMD Layer Control） |
| §4–§5 | `SyncParameterSequence`（PC/Android 同序）、`PlatformFilter`（按 VRSNS 平台 include/exclude） |
| §6–§10 | 菜单/参数五件套：`Parameters`、`MenuItem`、`MenuInstaller`、`MenuInstallTarget`、`MenuGroup` |
| §11–§13 | 组合关系与排错速查、建议落地、出处清单 |

---

# §1 `MergeAnimator`

**一句话**：把一个自带的 animator controller **追加**到 avatar 指定 layer（最常用 FX），让 prefab 拖上去就带着自己的动画能力。

**用**：gimmick 要播动画；给 prefab 打包"拖上即生效"的层。**不用**：想**彻底替换** avatar 某层——官方明确说它只增不替换，那种情况让用户按传统方式改 descriptor 更合适。

## 1.1 字段表

| Inspector 字段 | 取值 | 默认 | 语义 |
|---|---|---|---|
| `Animator to merge` | `AnimatorController` 资产 | 空 | 要合并的控制器 |
| `Layer Type` | `Base` / `Additive` / `Gesture` / `Action` / `FX` | —（必填） | 合并进哪个 avatar layer；gimmick 一般 `FX` |
| `Path mode` | `Relative` / `Absolute` | **Relative** | 路径解释方式，见 1.3 |
| `Relative Path Root` | `Transform` | 组件所在 GO | Relative 模式下路径的根；可另指 |
| `Layer Priority` | `int` | `0` | 合并顺序，见 1.4 |
| `Merge Mode` | `add`（默认）/ `Replace Existing Animator` | `add` | 追加 vs 替换 descriptor 的 animator |
| `Match Avatar Write Defaults` | `bool` | `false` | 自动匹配头像 WD，见 1.6 |
| `Delete attached animator` | `bool` | `false` | **构建期**删同 GO 的 `Animator` 组件（录动画用） |

**反射字段名（实测）**：`animator`、`layerType`、`pathMode`、`relativePathRoot`、`layerPriority`、`mergeAnimatorMode`（`Append` / `Replace`，**不是** `mergeMode`）、`matchAvatarWriteDefaults`、`deleteAttachedAnimator`。

## 1.2 机制

1. **只追加，不替换**已有层（默认模式）；要替换请改 descriptor。
2. **录动画**：路径默认相对本组件 → 给自身 GO 挂 `Animator`、在 `Animation` 面板直接录；录完勾 `Delete attached animator`，构建时自动删掉那个临时 `Animator`。
3. **Humanoid 骨骼动画例外**：动 humanoid 骨的动画**忽略相对路径逻辑，永远作用于整个 avatar**——现成的 humanoid 动画（如 AFK 动画）可**原样**合进来。
4. **`pathMode = Absolute`**：不解释相对路径，用于动"已挂在 avatar 上、不在你 prefab 下"的对象；代价是必须用 **avatar 根上的 animator** 录制。

## 1.3 Path mode 选择判据

| 场景 | 选 | 理由 |
|---|---|---|
| 动画目标在 prefab 子物体下、prefab 会被挪位置 | `Relative` | 路径跟着 prefab 走，可复用 |
| 动画目标是 avatar 既有对象（身体、骨骼、作者已有路径） | `Absolute` | 以 avatar 根为基准，与挂哪无关 |
| 想用自身 `Animator` + Animation 面板快速录 | `Relative` | 默认工作流 |
| 抄了别人的 controller（路径以 avatar 根为基准） | `Absolute`，否则**静默丢曲线** | 见 1.7 坑 1 |

## 1.4 Layer Priority 排序（官方，务必记牢）

- 按 priority **升序**放入最终 animator：**数字小的在前，数字大的覆盖前面的**。
- **同 priority** 的 MergeAnimator **按层级（hierarchy）顺序**排列。
- **预先存在的 animator（descriptor 上原有那个）视为 priority 0，且排在所有 priority-0 的 MergeAnimator 之前**。

> 推论：想盖住作者原层 → priority **≥1**；想被作者原层盖住 → 保持 0（仍排在既有 animator 之后）。
> ⚠ KB 现有"层顺序在 FX 末尾追加"的实测说法只在"未设 layerPriority / 单个 MergeAnimator"场景成立，**与本条不冲突但不能互相替代**——补丁里两条并列写，见 §12。

## 1.5 Merge Mode = Replace Existing Animator

- **替换** VRChat Avatar Descriptor 上配置的 animator（layer 级替换，不是追加）。
- 被替换的 animator **保留你指定的 priority**，但会在**同 priority 的其它 MergeAnimator 之前**应用。
- ⚠ **多个 MergeAnimator 同时设为同一 `Layer Type` + Replace → 直接报错**（硬失败，不是警告）。

## 1.6 Write Defaults 与 VRCAnimatorLayerControl

**`Match Avatar Write Defaults`**：勾选后 MA **检测头像是否一致地**使用 WD ON 或 OFF——**一致** → 你的 animator 状态被调整成匹配；**不一致** → 你的 animator **保持不变**（等于没勾）。⚠ **某些 blend tree 用法被排除**（部分类型不支持 WD "off"）。

**`VRCAnimatorLayerControl` 限制**：
- 只支持引用**同一 animator 内部层**的 state behavior——不能控制别人的层。
- `Playable` 字段必须**匹配 MergeAnimator 上选的 Layer Type**；`Layer` 字段填**你的 animator 内部的层索引**（不是最终 avatar animator 里的索引）。

## 1.7 常见坑

1. **默认 Relative 抄别人的 FX → 曲线全丢**：别人的路径以 avatar 根为基准，Relative 会把它前缀到 prefab 下 → 绑定全解析不到，烘焙输出 `NO VALID ANIMATIONS` 但**不报错**。判据：路径以 `Armature/...`、`Body` 这类 avatar 根路径开头 → 必须 `Absolute`；或 `Absolute` + 实例挂 avatar 根且命名与路径首段一致（本工作区 Riptide 套装即此模式）。
2. **`relativePathRoot` 补缺失前缀**：controller 路径少一段前缀时，把根指到对应父节点，比重录便宜。
3. **`layerPriority` 不设** → 自己的层和作者层都是 0，只按层级顺序排 → "加了层却没生效/被覆盖"多半在此。
4. **头像 WD 本身是混的**（本工作区常见）→ `matchAvatarWriteDefaults` **不会**动你的 animator，别指望它救场。
5. **`Delete attached animator` 只在构建期删**，编辑态那个 `Animator` 还在。
6. **Replace 冲突报错**：两个 prefab 都想 Replace 同一 Layer Type → 构建失败；分发场景统一用 append + priority。
7. **Relative 的基准是「组件所在 GameObject」，不是父级或 avatar 根**（本工作区第 147/156 轮实测）：组件挂在 `RK Player/RKSongFX` 子节点上，路径 `Armature/...` 被解析成 `RK Player/RKSongFX/Armature/...`（命中 0/88）→ 把 MergeAnimator 挂在动画路径起点对象上，逐条 `Find(path)` 命中率必须 100%。**变体：双拼接**——组件在 `RK Player` 上、曲线又写 `RK Player/Armature/…` → 拼成 `RK Player/RK Player/…`，曲线被丢、clip 改名 `rebased <原名>` 只剩 dummy（与 AAO 无关，AAO 开/关双烘焙结果相同）。面捕同构案例（`FT_Merge/Body` 毁掉面捕 + AAO 删键）见 [04-08 §2.3](../04-外观与表情/04-08-face-tracking-osc-and-repair.md)；RK Player 案例见 [02-11 §1.3](../02-工程组织与工作流/02-11-plugin-integration-rk-realkiss-props.md)。

## 1.8 自检清单

- [ ] `Layer Type` = `FX`（或确实需要的层）
- [ ] `pathMode` 与控制器路径基准一致
- [ ] `layerPriority` 明确想过"盖住谁/被谁盖住"
- [ ] 只有一个组件对本 Layer Type 用 Replace
- [ ] 用了 `VRCAnimatorLayerControl`：`Playable` = 本组件 Layer Type，`Layer` = controller 内层索引

**出处**：https://modular-avatar.nadena.dev/docs/reference/merge-animator

---

# §2 `Merge Motion`（旧名 `Merge Blend Tree`）

**一句话**：把多个 blend tree / animation clip **合并进单个 FX 层**，用于降低 animator 开销，或让某个动画**常驻播放**。

**命名与 API（重要）**：
- **1.12 之前**叫 `Merge Blend Tree`；**1.12 起**扩展支持直接合并 animation clip，改名 `Merge Motion`。
- 1.11 及更早的资产会**自动升级**到新组件。
- 但**内部 API 名仍是 `ModularAvatarMergeBlendTree`**——反射/`Type.GetType` 必须按老名找，按 "MergeMotion" 找会得到 null。

**用**：有 motion（clip 或 blend tree）你希望**永远激活**。**不用**：需要**启用/禁用**该 motion，或动画需要**随时间变化**。

## 2.1 字段表

| Inspector 字段 | 取值 | 默认 | 语义 |
|---|---|---|---|
| `Motion (or Blend Tree) to merge` | `BlendTree` / `AnimationClip` | 空 | 要合并的 motion；动画会**持续播放** |
| `Path Mode` | `Relative` / `Absolute` | `Relative` | **与 MergeAnimator 完全相同** |
| `Relative Path Root` | `Transform` | 本组件 GO | 同上 |

**反射**：类型 `nadena.dev.modular_avatar.core.ModularAvatarMergeBlendTree`（按 1.12 前老名）；字段名同 MergeAnimator 的 `pathMode` / `relativePathRoot`，motion 字段名未反射【待实测】。

## 2.2 合并机制（官方明确）

1. 在 **FX controller 顶部**新建**一层**；
2. 该层**只有一个 state**，**Write Defaults = on**；
3. state 内是 **Direct Blend Tree**；
4. 每个被合并的 motion 挂进该 DBT，**其参数恒为 1**。

> 推论：合并后的层不受任何参数控制（参数恒 1 = 永远生效）——"想开关"的需求天然不适合它，该用 `MergeAnimator` + 参数/ReactiveObject。

## 2.3 ⚠ 核心警告：所有动画被调成相同有效时长

官方语义：**Merge Motion 把动画合并进单个 direct blend tree；在 Unity 里这意味着 blend tree 中所有动画会被调整为拥有相同的 effective duration；这会导致播放速度不可预测，因此不推荐用于随时间改值的动画——这类需求改用 `MergeAnimator`。**

- 后果：合并"1 秒呼吸" + "3 秒挥手"，时长被强行统一 → 实际速度偏离作者意图，**且不报错**。
- 判据：**只要动画的速度/节奏/延迟有依赖 → 不要用 Merge Motion**；仅"恒定姿态/常驻循环且对速度不敏感"可用。
- 与 MergeAnimator 分工：需要状态、开关、精确速度 → `MergeAnimator`；只要"叠一层常驻姿态且想省层" → `Merge Motion`。

## 2.4 常见坑

1. **拿它做表情/眨眼/呼吸等速度敏感动画** → 速度错乱且难排查（烘焙结果看不出异常，进游戏才发现）。
2. **以为它受参数控制** → 参数恒 1，没有开关能力。
3. **反射按 "MergeMotion" 找组件** → null；必须 `ModularAvatarMergeBlendTree`。
4. **合并 clip 以为只播一次** → 官方明确"animation will be constantly played"（持续循环）。
5. **与 MergeAnimator 抢 FX 顶部** → Merge Motion 层在顶部；需要压住它用 priority 处理，别硬调顺序。

**出处**：https://modular-avatar.nadena.dev/docs/reference/merge-blend-tree （页面标题已改为 Merge Motion，URL 未变）

---

# §3 `VRChat Settings` 与 MMD World 行为

**一句话**：`VRChatSettings` 是"头像级、仅适用 VRChat"的配置组件；目前只有一个开关——**MMD World Support**，控制 MMD 世界的动画层特殊处理。

## 3.1 `VRChat Settings` 字段表

| 项目 | 取值 / 约束 | 默认 | 语义 |
|---|---|---|---|
| 组件位置 | 任意对象 | — | 官方："It can be placed anywhere on the avatar." |
| 实例数 | **只能 1 个** | — | 同一 avatar 只能一个；**没有则用默认设置** |
| `MMD World Support` | `bool` | 文档未写【待实测】 | 启用"为 MMD 世界做特殊 animator 处理"，细节见 3.2 |

**反射**：`nadena.dev.modular_avatar.core.ModularAvatarVRChatSettings`【待实测：本工作区未反射，字段名未核】。

## 3.2 MMD World 行为（general-behavior/mmd）

**现象**：VRChat 某些 **MMD 世界**会**禁用你 FX animator 的第 2、3 层**——这是世界故意的，意图**关掉控制面部表情的层**，好让 MMD 世界接管表情。

**MA 的自动应对**：
1. MA 自动安排"**原本就是第 2、3 层**的层"在该情形下被禁用（按"原始层位置"记账，不按最终位置）。
2. 若前面插入了新层（把它们挤到 4、5 位），MA 会加 **relay 层**去驱动"第 2、3 层"的关/开。
3. **通过 `MergeAnimator` 添加的层（即使 `Replace` 模式）不受该 MMD 行为影响**；必要时 MA 会加 **padding 层**保护（不让它们被顶到 2/3 位）。
4. 想让某些层**参与**该行为（愿意被 MMD 世界关掉）→ 给**那一层**挂 MA 的 **MMD Layer Control** state machine behavior。
5. 也可整体关掉：给 avatar 挂 `VRChat Settings` 并关掉对应设置。

## 3.3 ⚠ 限制与警告

- **MMD Layer Control 只能挂在"层"上**。官方警告：状态机 behavior 机制上允许挂到**单个 state**，但**这会让构建失败**——别挂 state。
- 该 workaround **只对"真的禁用第 2、3 层"的世界有效**；官方说明受 VRChat 限制无法提供更通用方案。
- 该行为是 **PC 侧 MMD 世界**的问题（Quest/Android 无 MMD 世界）；官方页未明写"PC 专属"【待复核】。
- 【待实测】**relay 层与 padding 层是否真的生成、几层、命名**——本工作区未在真实 MMD 世界或模拟禁用 2&3 层的环境验证；写 KB 必须保留此标记。

## 3.4 与本工作区已记录症状的对应

KB 有一条**只有症状没有机制**的观察：带 FaceEmo 进 MMD 房跳舞，头发/动骨"没了"（`02-09` 评论区原文）。按 3.2 解释：
- 世界禁掉 FX 第 2、3 层 → 若发丝物理/表情/动作层原本落在 2、3 位（作者 controller 原始层序），整层停止 → 表现为"头发不动/表情没了"。
- 应对：① 确认出问题的层是否 MergeAnimator 加的（**加的层不受影响**，仍坏说明另有原因）；② 调层序让关键层不在 2/3 位；③ 关 `MMD World Support` 看是否恢复；④ 给层挂 MMD Layer Control 是"主动参与"，方向相反，别乱用。

**出处**：
- https://modular-avatar.nadena.dev/docs/reference/vrchat-settings
- https://modular-avatar.nadena.dev/docs/general-behavior/mmd
- https://modular-avatar.nadena.dev/docs/reference/mmd-layer-control （该页仅指向上面 MMD 页）

---

# §4 `SyncParameterSequence`

**一句话**：VRChat 要求"跨平台共享的同步参数"在 Expressions Parameters 列表里**位于开头且顺序一致**；该组件自动调整 EP 顺序、按需补参数，让 PC 与 Android 之间同步正确。

## 4.1 字段表

| Inspector 字段 | 取值 | 默认 | 语义 |
|---|---|---|---|
| （组件位置） | 任意对象 | — | 官方："attach to any object on your avatar" |
| `Primary Platform` | 平台（PC / Android / iOS …） | —（必选） | **主平台 = 定义了你所有想同步的参数的平台** |

**反射**：【待实测：本工作区未反射该类型】。

## 4.2 机制与正确工作流（顺序不能错）

1. **VRChat 硬要求**：不同平台版本之间共享的同步参数必须出现在 EP 列表**开头**，且**顺序相同**；否则同步错位（参数值串台）。
2. 组件做两件事：**调整 EP 顺序** + **在需要的地方补参数**，保证 PC↔Android（及其它平台）一致。
3. **工作流（官方强调）**：选 `primary platform` → **先在主平台 build & upload** → **再为其它平台 build & upload**。只要先传主平台，之后所有平台顺序自动一致。
4. **用 VRChat per-platform overrides 时**：只给**原头像**加该组件，**override 头像不必加**——组件会自动把主平台头像上**缺失的参数补进 override 头像**。
5. ⚠ **参数要求**：**主平台头像必须包含"任何变体上需要同步的全部参数"；缺任何一个，build 直接失败**。

## 4.3 什么时候用 / 不用

- **用**：同一 avatar 分别上传 PC 与 Android，且两版本都用了**同步**参数。
- **不用**：只传一个平台；或全部参数都是 unsynced。

## 4.4 限制与冲突

- 官方明说**可能与某些 VRCFury 组件冲突**，点名 **`Parameter Compressor`**（两者都在重排/压缩 EP，天然打架）。
- 与 per-platform overrides 组合：只需原头像加，不要 override 也加。

## 4.5 常见坑

1. **顺序反了**（先传 Android 再传 PC）→ 顺序不一致、同步错位，且不一定报错。
2. **主平台漏参数** → build fail；补参数要从变体头像反查。
3. **与 Parameter Compressor 同用** → 冲突；二选一或明确测过再用。
4. **以为它会自动补"变体独有的 synced 参数"** → 不会；主平台必须先定义全。
5. 【待实测】本工作区**没有做过 PC + Android 双平台上传**，该组件未实测；以上全部为官方文档语义。

**出处**：https://modular-avatar.nadena.dev/docs/reference/sync-parameter-sequence

---

# §5 `PlatformFilter`

**一句话**：按**目标 VRSNS 平台**（VRChat、Resonite 等）**包含（Include）或排除（Exclude）**指定 GameObject。

## 5.1 字段表

| Inspector 字段 | 取值 | 默认 | 语义 |
|---|---|---|---|
| `Platform` | 目标 VRSNS 平台（如 `VRChat` / `Resonite`） | —（必选） | 这个 filter 针对哪个平台 |
| `Filter Mode` | `Include` / `Exclude` | —（必选） | `Include` = **只在指定平台构建时存在**；`Exclude` = **在指定平台被移除** |
| 实例数 | **同一 GameObject 可挂多个** | — | 每个组件指定一个平台；多平台就多个组件 |

**反射**：【待实测：本工作区未反射该类型】。

## 5.2 机制与规则

1. **`Include`**：该 GameObject **只在**指定平台存在于构建产物中（其它平台被剔除）。
2. **`Exclude`**：该 GameObject 在**指定平台**被移除（其它平台保留）。
3. **同一 GameObject 同时有 include 与 exclude filter → 报错**（官方："an error will be reported"）。
4. **不一定必要**：官方明确说很多 MA 特性已自动处理平台限制——**例：`Merge Animator` 本来就只在 VRChat 运行**。加之前先确认真的需要。

## 5.3 典型用法

- 只在 VRChat 显示某 gimmick → `Platform = VRChat` + `Include`。
- 在 Resonite 隐藏某物 → `Platform = Resonite` + `Exclude`。
- 多平台差异 → 同一 GO 挂多个 filter。

## 5.4 常见坑

1. **当成"PC/Quest 差异"万能开关** → 它按 **VRSNS 平台**过滤，不是按性能等级/是否 Quest；Quest 精简应走 avatar 变体/移除资源。
2. **同一 GO 既 include 又 exclude** → 报错。
3. **给 MergeAnimator 加 filter** → 多余（它本来就只对 VRChat 生效）。
4. **以为编辑态就消失** → 构建期过滤，编辑态仍在【待复核：官方未明写编辑态表现】。

**出处**：https://modular-avatar.nadena.dev/docs/reference/platform-filter

---

# §6 `Parameters`

**一句话**：声明 gimmick 用到的 animator 参数（内部用或与其它组件通信）、**改名避免冲突**、定义 synced/unsynced 与默认值。

## 6.1 字段表（每条 entry）

| Inspector 字段 | 取值 | 默认 | 语义 |
|---|---|---|---|
| 名称 / 前缀（第一行） | `string` | — | 参数名，或 PhysBone 用的**前缀** |
| `Type`（右上） | `Bool` / `Int` / `Float` / `Animator Only` / `Prefix` | 文档未写【待实测】 | 类型，见 6.2 |
| `Change name to` | `string` | 空 | 对**本组件及其子物体之外**的一切改名，见 6.3 |
| `Auto rename` | `bool` | `false` | 自动选未被使用的名字 |
| `Default Value` | 数值 / bool | **空** | avatar **重置**时使用；留空有回退链，见 6.4 |
| `Override Animator Defaults` | `bool` | `false` | 把资产 animator controller 的默认值改成该值，见 6.4 |
| `Saved` | `bool` | `false` | 跨 **avatar 切换**与**重启 VRChat** 保存 |
| `Synced` | `bool` | `true` | 跨网络同步；**清空后不占参数空间** |

**反射字段名（实测）**：`ModularAvatarParameters.parameters` 是 `List<ParameterConfig>`，字段 = `nameOrPrefix`、`remapTo`、`internalParameter`、`isPrefix`、`syncType`（`NotSynced / Int / Float / Bool`）、`localOnly`、`defaultValue`、`saved`、`hasExplicitDefaultValue`。
> ⚠ `Animator Only` **不是** `syncType` 的枚举值，而是 `internalParameter = true`；`Prefix` 是 `isPrefix = true`。写脚本别去 `syncType` 里找 `AnimatorOnly`。

## 6.2 五种参数类型

| 类型 | 进 EP？ | 用途 | 备注 |
|---|---|---|---|
| `Bool` | 是 | 开关 | 1 bit |
| `Int` | 是 | 多态选择（如套装编号） | 8 bit |
| `Float` | 是 | 连续值（发色、BlendShape 权重） | 8 bit |
| `Animator Only`（`internalParameter`） | **否** | 只给 animator 内部用、**不该出现在 VRChat 参数面板**的 gimmick 参数；**仍可参与改名** | 与 MenuItem 自动参数、EP 声明互不冲突 |
| `Prefix`（`isPrefix`） | **否** | 你在 **PhysBones / Raycast 组件**上填的参数前缀；这些组件按前缀**生成多个参数**，用 Prefix 可**一次配全** | 同样不进 EP |

## 6.3 改名（`Change name to` / `remapTo`）的作用域

- 你填的名字会应用到**"MA Parameters 对象及其子物体之外"的一切**——外部 animator/菜单/其它组件看到的参数名变成它。
- 用途二选一：① **避免冲突**（两个 prefab 都用 `Toggle` 时各改各名）；② **故意串接**（两个 gimmick 想共用一个参数 → 对外名改成同一个）。
- `Auto rename`：让 MA 自动挑**未被使用**的名字（分发 prefab 时最省事）。

## 6.4 默认值回退链与 `Override Animator Defaults`

**默认值**：
1. 该值在 **avatar 重置（Reset Avatar）** 时使用。
2. `Default Value` **留空** → 取**主 Expressions Parameters 资产**里的值（如果有）。
3. 主 EP 也没有 → **`0`（或 `false`）**。

**`Override Animator Defaults`**：勾选后把**资产里 animator controller** 中该参数的默认值**改成这里填的值**（官方说"对特别复杂的 gimmick 偶尔有用"）。
⚠ **例外**：类型选 **`Animator Only` 且填了默认值**时，此勾选框**被忽略**，而 **animator controller 的默认值总会被替换**。

> 本工作区"三处默认值必须一致"（MA Parameters / EP / AnimatorController）就是这条的实战版；三者不一致时重置值与烘焙后值会分叉。

## 6.5 Saved / Synced

- `Saved`：**跨 avatar 切换 + 重启 VRChat** 保留（**不是**"跨世界"）。
- `Synced`：跨网络同步；**清空 = 不占你有限的参数空间**（但非同步参数仍占 EP **条目名额**，只是 sync bit 为 0——本工作区 `parameter-and-menu-audit` 的更严谨口径，官方页没写这么细）。
- 两者**互相独立**：可 saved 不 synced、也可 synced 不 saved。

## 6.6 新建参数的两条路径

1. **`+` 按钮**（列表底部）→ 点新条目旁 chevron 展开详细视图 → 填名字/类型/属性。
2. **`Unregistered Parameters` 段** → 列出**在该 GameObject 及其子级组件里被检测到、但尚未注册**的参数 → 点 **`Add`** 加入，或点**放大镜**跳到检测位置（排查"这个参数是谁声明的"最有用）。

> 两条路径建完后一样：点 chevron 展开，设置类型（类型同时决定是否 synced）与其它属性。

## 6.7 嵌套规则（官方明列）

MA Parameters **可嵌套**（多个子组件拼复杂系统）；**每个**组件都能对**其所有子物体**施加改名。
例：内层把 `foo` 改名成 `bar`，外层把 `bar` 设 auto rename → **中间对象仍可用 `bar` 访问**。

| 属性 | 规则 |
|---|---|
| `Saved` | **取最外层**。但：多个**非嵌套**组件对该参数设了不同 `Saved` → **只要任一设为 saved，该参数就是 saved** |
| `Default Value` | **取最外层**；外层留空则取**最内层的非空**默认值。但：多个**非嵌套**组件都设了非空默认值 → **显示警告**（不知道该用哪个） |

## 6.8 常见坑

1. **`isSaved` 对已存在的 EP 参数无效**：只对"MA 新建/接管的参数"生效；EP 里早已存在的参数要直接改 EP（本工作区实测：`03-06 §14.3`、`parameter-and-menu-audit §8`）。
2. **三处默认值只改一处** → 重置值/烘焙值分叉。
3. **`AnimatorOnly` 去 `syncType` 里找** → 找不到；它是 `internalParameter`。
4. **Prefix 当普通参数用** → 不写进 EP，菜单也绑不到。
5. **嵌套里期望"内层 saved 覆盖外层"** → 反了，最外层优先。
6. **自动参数名 `__MA/AutoParam/<GO名>`**：MenuItem 自动建参数的命名规则（本工作区实测），别与手写声明重复。

## 6.9 自检清单

- [ ] 每个手写参数都有明确归属组件（不靠 MenuItem 自动建）
- [ ] 类型选对：内部用 `Animator Only`，PhysBone/Raycast 前缀用 `Prefix`
- [ ] `Change name to` 的作用域想清（避冲突 vs 故意串联）
- [ ] 默认值三处一致；`Saved` / `Synced` 分别确认
- [ ] 嵌套时确认 saved / default 实际取值符合"最外层优先"

**出处**：https://modular-avatar.nadena.dev/docs/reference/parameters

---

# §7 `MenuItem`

**一句话**：在 Unity 层级里直接定义单个 expressions 菜单项（不用手写 VRC Expressions Menu 资产）。

## 7.1 字段表

| Inspector 字段 | 取值 | 默认 | 语义 |
|---|---|---|---|
| 名称 | — | — | **取承载它的 GameObject 名**（改 GO 名即改菜单标签） |
| `Icon` | `Texture2D` | 空 | 菜单图标 |
| `Type` | `Button` / `Toggle` / `SubMenu` | `Button` | 控件类型；**反射真值 `Button=101` / `Toggle=102` / `SubMenu=103`**（另有 201/202/203 变体，见 `03-06`） |
| `Parameter` | `string` | 空 | 绑定参数名；未声明会自动创建，见 7.3 |
| `Submenu Source` | `Expressions Menu Asset` / `Children` | — | 子菜单来源，见 7.2 |
| `source object override` | `GameObject` | 空 | 仅 Children：改从**该对象的子物体**取内容 |
| `Is Default` | `bool` | `false` | 设为该参数默认值，见 7.4 |
| `Is Saved` / `Is Synced` | `bool` | — | 自动创建参数时才出现 |

**反射字段名（实测）**：`Control`（本工作区 SDK 下 `VRCExpressionsMenu.Control` 是 **class 不是 struct**）、`label`、`automaticValue`、`isDefault`、`isSynced`、`isSaved`、`MenuSource`（`MenuAsset=0` / `Children=1`）、`menuSource_otherObjectChildren`。

## 7.2 子菜单

- `Type = SubMenu` 时内容来源二选一：① **`Expressions Menu Asset`** = 引用传统菜单资产；② **`Children`** = 用**本菜单项直接子物体上的 MenuItem** 填充（对象式建菜单最常用）。
- **自动分页**：菜单项超过 VRC 单页上限时自动创建 **"next" 项**拆成多页。
- **`source object override`**（仅 Children）：指定后改从**那个对象的子物体**取内容。

## 7.3 自动创建参数

- 参数名**没有在 MA Parameters 或 VRC Expressions Parameter 资产里声明**时，MenuItem 会**自动创建**该参数，inspector 出现 saved / synced 复选框。
- **没填参数名也会出现这些选项**——**只要该菜单项控制了任何 Reactive Object，就会自动创建参数**。所以"纯显示菜单项却多出一个参数"是正常的：去掉它控制的 Reactive Object 即可。

## 7.4 ⚠ `Is Default` 硬警告

- 勾 `Is Default` = 该菜单项成为**该参数的默认值**。
- **多个菜单项对同一参数都设 `Is Default` → 结果未定义**（官方原话："the results are undefined. Don't do that!"）。不要指望"后者胜"或"层级顺序胜"。
- 需要默认项时：**只给一个**菜单项勾，其它同参数项一律不勾。

## 7.5 绑定到菜单树（三种方式，否则无效）

1. 作为**另一个 MenuItem（SubMenu / Children 模式）的子物体**；
2. 与 **Menu Installer 在同一 GameObject**；
3. 作为 **Menu Group 的子物体**（Menu Group 通常也在挂了 Menu Installer 的 GO 上）。

> **未绑定的 MenuItem 没有任何效果**（静默忽略，不报错）。

## 7.6 常见坑

1. **改 `Control.name` / `label` 不改菜单名** → 名称取 GameObject 名（本工作区实测；label 通常为空，非空时才优先）。
2. **ControlType 写 `0/1/2`** → `ArgumentOutOfRangeException`，MA 反应层整段不生成（烘焙"看起来成功"但开关全失效）。必须 `101/102/103`。
3. **`automaticValue` 陷阱**：只有 `isDefault && !automaticValue` 才成为参数默认值；非默认项 + Bool/Float 自动参数，MA 强制 value=1（`07-05` 实测）。
4. **多个 isDefault** → 未定义，见 7.4。
5. **超 8 项** → MA 自动塞 "next" 分页（`SKILL.md` 症状表）；手工 CreateAsset 的菜单树**不会**自动分页（分页是 MA MenuInstallHook 后处理）。

**出处**：https://modular-avatar.nadena.dev/docs/reference/menu-item

---

# §8 `MenuInstaller`

**一句话**：把 prefab 自带的菜单/菜单项装进 avatar 的 expressions 菜单树。

## 8.1 字段表

| Inspector 字段 | 取值 | 默认 | 语义 |
|---|---|---|---|
| （组件位置） | 任意 | — | 默认把 prefab 菜单装到 **avatar action menu 顶层** |
| `Select Menu`（按钮） | 点按钮 → **双击**目标菜单 | 空（= 顶层） | 选安装目标；**双击**才选定 |
| `Menu to install`（Prefab Developer Options） | 菜单资产 | 空 | prefab 作者指定要分发的菜单 |
| `Install To` | 菜单资产 / 其子菜单 | 空 | 可指向**另一个 Menu Installer 正在安装的菜单资产（或其子菜单）**，用于扩展别人的菜单 |
| 左上 disable 复选框 | `bool` | 关 | 勾上 = **完全关闭该 installer 的安装** |

**反射字段名（实测）**：`installTargetMenu`（目标菜单资产，`null` = 装到 avatar 根）、`menuToAppend`（要追加的菜单资产；**它平铺的是 `controls`，不是子菜单包装壳**）。

## 8.2 机制

1. **默认**：prefab 菜单装到 **action menu 顶层**。
2. **选目标**：点 `Select Menu`，**双击**要装的菜单；选中菜单满了会自动拆成多页（submenu）。
3. **Prefab 开发者**：
   - 先建装着你要加的控件的 expressions menu 资产；它会被**追加到选定子菜单**。
   - ⚠ **想要自己的子菜单，需要建两个菜单资产**：一个给"子菜单控件"，一个给"子菜单内部内容本身"（否则只会把控件平铺进对方菜单）。
   - Menu Installer 应放在**与 Parameters 组件同一层（同一 GameObject）**——使用者拿到 prefab 时两组件配置能一起生效/管理。
   - 在 `Prefab Developer Options` 里把菜单填进 `Menu to install`。
4. **Menu sources**：除装现成菜单资产外，若在 installer 上**附加 `MenuItem` 或 `MenuGroup` 组件**，installer 会按该组件配置**生成菜单项**。
5. **扩展别人菜单**：在 `Install To` 指定"**另一个 Menu Installer 正在安装的菜单资产（或其子菜单）**"，把内容扩展进别人 prefab 安装的菜单。

## 8.3 常见坑（本工作区实战为主）

1. **`menuToAppend` 填了包装壳** → 只平铺其中 `controls`；若那是个只含 1 个 SubMenu 控件的壳，效果就是"菜单多套一层"。正解：填**内层内容菜单**，或按 8.2 第 3 条建两个资产。
2. **Installer 与 SubMenu MenuItem 同 GO** → SubMenu 会**把自己也装进目标菜单**（多一层）。应分开放。
3. **`installTargetMenu` 指向的分类资产 `controls` 为 0 项** → 正常，内容在烘焙期填充；判断"装没装上"要看烘焙产物。
4. **禁用开关勾上后完全不装**，但组件还在 → 排查"菜单没出现"先看它。
5. **多个 prefab 装到同一分类菜单** → 允许；留意菜单上限与命名冲突。

**出处**：https://modular-avatar.nadena.dev/docs/reference/menu-installer

---

# §9 `MenuInstallTarget`

**一句话**：支撑 `MenuInstaller` 上 `Select Menu` 按钮的辅助组件——把 installer 的菜单"**拉过来**"，按**自己所在 GameObject 的位置**安装。

## 9.1 字段与语义

| 项目 | 说明 |
|---|---|
| 字段 | 基本没有用户可配字段；**它挂在哪里，就等于"菜单被粘贴到哪里"** |
| 谁创建 | **MA 在你点 `Select Menu` 时自动创建**；多数情况**不需要手建** |
| 作用对象 | 与某个 **Menu Installer** 配对，从该 installer 拉菜单 |

## 9.2 它做了什么

1. **覆盖** installer 上选定的目标菜单选项；
2. installer 于是表现得**好像它的菜单被 copy-paste 到了 MenuInstallTarget 所在位置**；
3. 目的：让"用 Menu Installer 打包的 prefab"能**融入对象式菜单系统**（把安装点下沉到菜单树某节点）。

## 9.3 常见坑

1. **手工删/搬 `MenuInstallTarget`** 会改变安装位置；它不是垃圾组件（KB 原话："不要盲删所有 InstallTarget"）。
2. **排查"菜单装错位置"**：先看 installer 指向哪，**再看有没有配套 InstallTarget（以及那个 GO 在哪）**——位置由 InstallTarget 决定。
3. **烘焙前看不出效果**：构建期才参与菜单组装。
4. 【待实测】烘焙后菜单路径与 InstallTarget 位置的精确对应（本工作区未做对照实验）。

**出处**：https://modular-avatar.nadena.dev/docs/reference/menu-install-target

---

# §10 `MenuGroup`

**一句话**：让**单个 MenuInstaller 安装多个菜单项，而不把它们聚成一个 submenu**。

## 10.1 字段与机制

| 项目 | 说明 |
|---|---|
| 默认内容 | **包含"自身直接子物体上挂的所有 MenuItem"** |
| `source object override` | 与 MenuItem 的 children 模式一样：可指定另一个对象作为来源 |
| 定位 | 官方明说**主要是内部组件**（"primarily intended as an internal component for use when converting expressions menus"）——手写菜单一般不需要 |

## 10.2 什么时候用 / 坑

- **用**：要把**一批菜单项平铺**进目标菜单，而不是收进新子菜单；或做 expressions menu → 对象式菜单的自动转换。
- **坑 1**：误当常规绑定方式——常规写法是 MenuItem 作为父 SubMenu 子物体、或与 installer 同 GO；MenuGroup 只在"要平铺多项"时才有意义。
- **坑 2**：以为它会生成子菜单——不会，它恰恰是"不聚合"。
- **坑 3**：`source object override` 与 MenuItem 的混淆——语义一致（改从哪个对象取子物体），只是字段挂在不同组件上。

**出处**：https://modular-avatar.nadena.dev/docs/reference/menu-group

---

# §11 组合关系与排错速查

## 11.1 组件关系（文字版）

```
MenuItem：定义菜单项
  ├─ 子菜单：Expressions Menu Asset ｜ Children（子物体上的 MenuItem）
  ├─ 未声明参数 → 自动建参数（出现 saved / synced）
  └─ 绑定：父 SubMenu 子物体 ｜ 与 MenuInstaller 同 GO ｜ MenuGroup 子物体
MenuInstaller：装菜单（默认 action menu 顶层；Select Menu → 自动建 MenuInstallTarget）
  └─ Prefab Developer：Menu to install ＋两个菜单资产（控件壳 + 内容）；Menu sources：同 GO 的 MenuItem / MenuGroup
MenuGroup：单 installer 平铺多项，不聚合 submenu
Parameters：声明 / 改名 / 默认值 → 类型 Bool·Int·Float·Animator Only·Prefix；remapTo 对外改名；嵌套 saved 取最外层、default 取最外层空则最内层
MergeAnimator：加层 → Layer Type + Priority（升序，既有 animator = 0 且最前）；add / Replace（同 LayerType 多 Replace 报错）；Relative·Absolute；humanoid 骨动画不受路径影响；VRCAnimatorLayerControl 只能控制同 animator 内层
Merge Motion：单 FX 顶部层（DBT、参数恒 1、WD on）→ 所有动画被拉成相同有效时长
VRChat Settings（唯一实例）→ MMD World Support；SyncParameterSequence → 跨平台同序；PlatformFilter → 按平台 include / exclude
```

## 11.2 症状 → 先查哪

| 症状 | 先查 |
|---|---|
| 菜单没出现 | Installer 的 disable 复选框、MenuItem 是否绑定、`installTargetMenu` 是否装到预期节点 |
| 菜单多套一层 / 自己套自己 | `menuToAppend` 是否包装壳、Installer 与 SubMenu 是否同 GO |
| 菜单项名字改不动 | 改的是 GO 名还是 `Control.name`（后者无效） |
| 开关点了没反应 | ControlType 是否 101/102/103；参数是否声明；ReactiveObject 目标是否解析 |
| 默认值不对 | 三处默认值是否一致；`isDefault` 是否唯一 |
| 层没生效 / 被覆盖 | `layerPriority`；是否有 Replace；合并顺序 |
| 曲线全丢 / NO VALID ANIMATIONS | `pathMode` 是否 Absolute；`relativePathRoot` |
| 动画速度怪异 | 是否用了 Merge Motion（统一时长坑） |
| 进 MMD 房掉表情/掉物理 | FX 2/3 层是否被禁；层是否 MergeAnimator 加的（后者不受影响）；MMD World Support |
| PC/Android 参数串台 | SyncParameterSequence：主平台先后顺序、主平台是否含全参数 |
| 某物在别的平台也出现了 | PlatformFilter 是否缺失/写反（include+exclude 同 GO 会报错） |

---

# §12 建议落地（KB 文件级）

> **2026-11-01 并入后**：本节是审计稿原文，保留作历史记录。**§1–§10 的正文已全部落在本文件**（即上表第 1 行所指分册）；与其它册的重叠点只在「引用」层面，不重复搬运。
|---|---|---|
| §4 SyncParameterSequence、§3 VRChatSettings + MMD、§2 Merge Motion、§5 PlatformFilter | **新建分册（已建）** `references/03-菜单与参数/03-07-ma-platform-sync-mmd-and-motion.md` | 四主题独立且有版本分界（1.12 改名），独立成册最清晰 |
| §1 MergeAnimator 的 1.4 层序 / 1.5 Replace / 1.6 WD 与 LayerControl / Humanoid | `references/00-基底/00-02-modular-avatar.md`（组件表扩到 13 个）+ `02-03` §6 | 现有组件表只有 8 个组件、字段级语义缺失 |
| §6 Parameters 的 6.2~6.7 | `references/02-工程组织与工作流/02-02`（参数段 `:354-363`） | 该处已有字段名清单，正好接语义 |
| §7 MenuItem 7.4、§8/§9/§10 菜单组件语义 | `references/03-菜单与参数/03-06` §12 + §14.3 | 该册是反射建菜单手册，缺这几个组件的语义 |
| §4 跨平台同步 | `references/03-菜单与参数/03-01-parameter-and-menu-audit.md` 新增 §10 | 该册管参数预算/EP，跨平台同步是自然延伸 |
| §5 PlatformFilter 字段语义 | `references/02-工程组织与工作流/02-03` §16（现仅 2 行） | 决策层已有，缺字段 |
| MMD 层行为验证方法（relay/padding 是否生成） | `references/07-验证与排查/07-01-bake-and-validation.md` | 属"烘焙后断言"类 |
| 症状"进 MMD 房头发/动骨没了" | `SKILL.md` 症状速查表 → 指 §3.2 | 现 KB 只有评论原文，无根因 |

**补丁优先级**：§1（MergeAnimator 三条硬语义）> §3（MMD）> §2（Merge Motion 时长坑）> §4（跨平台同步）> §6（Parameters 字段语义）> §7~§10（菜单组件语义）> §5（PlatformFilter）。

---

# §13 出处清单

| 组件 / 主题 | URL |
|---|---|
| MergeAnimator | https://modular-avatar.nadena.dev/docs/reference/merge-animator |
| Merge Motion（MergeBlendTree） | https://modular-avatar.nadena.dev/docs/reference/merge-blend-tree |
| VRChat Settings | https://modular-avatar.nadena.dev/docs/reference/vrchat-settings |
| MMD World Workarounds | https://modular-avatar.nadena.dev/docs/general-behavior/mmd |
| MMD Layer Control | https://modular-avatar.nadena.dev/docs/reference/mmd-layer-control |
| SyncParameterSequence | https://modular-avatar.nadena.dev/docs/reference/sync-parameter-sequence |
| PlatformFilter | https://modular-avatar.nadena.dev/docs/reference/platform-filter |
| Parameters | https://modular-avatar.nadena.dev/docs/reference/parameters |
| MenuItem | https://modular-avatar.nadena.dev/docs/reference/menu-item |
| MenuInstaller | https://modular-avatar.nadena.dev/docs/reference/menu-installer |
| MenuInstallTarget | https://modular-avatar.nadena.dev/docs/reference/menu-install-target |
| MenuGroup | https://modular-avatar.nadena.dev/docs/reference/menu-group |

> 快照：`<VRCHAT_ROOT>\tools\tmp\ma-docs-txt\`（2026-11-01 抓取，以官方站点为准）。

# §14 抽取舞蹈 Action 后只放歌、AFK 覆盖与走跑冻结（2026-11-12）

**本工作区实测版本**：Unity 2022.3.22f1c1、SDK 3.10.6、MA 1.18.7、NDMF 1.14.8、VRCFury 1.1442.0、GM 3.9.9。目标 LuzzyMeow V2 的小狼；仅据此版本和当前控制器结构作结论。

## 14.1 补充/取代旧的“GM 不跑 Action，所以客户端会动”推断

⚠ **取代边界**：[07-06 §23.3](../07-验证与排查/07-06-gesture-manager-automated-sim-test.md) 记录的“状态进入但骨骼不动”是历史现象，不能据此证明插件接线正确，更不能推导客户端正常。编辑态 `AnimationMode.SampleAnimationClip` 只能证明 clip 可采样、骨架兼容，不能证明运行期权重、覆盖关系与 tracking 正确。

本轮从真实 GM 菜单事件入手，逐层读取 `AnimatorControllerPlayable` 权重、当前状态与逐帧骨骼旋转，得到三层根因：

1. 从 VRCFury 烘焙头像抽出的 `SyncEmotes_Action` 仅留舞蹈层，沿用旧 `layer=3` 控制而新控制器只有一层；原始全局 Action 启用机制也未随抽取保留。
2. MA 会把每个待合并控制器的首层当作权重 1（当前 `MergeAnimatorProcessor.cs:181–183`）。后置 AFK 首层默认 `WaitForActionOrAFK` 播全身 `proxy_stand_still`，会覆盖舞蹈。Play 内只把该层降 0，Hips 每帧角变化从 0 变为 15.63°，证明覆盖关系。
3. 手补一个常驻 PLC(Action=1) 也不能直接算修复：VRCFury 多 owner 冲突解析器可能把它展开成“抬升该 owner 全部 Action 子层”的多个 ALC；SDK 默认站姿层随之被抬到 1，覆盖 Base Locomotion，造成 Walk/Run 不动。

## 14.2 分清全局播放层、内部子层与构建期 owner

| 对象 | 控制方式 | 判据 |
|---|---|---|
| 整个 Action 播放层 | PLC `layer=Action`；此枚举在当前 SDK 中是 0 | 读 GM 的 `AnimatorControllerWeight.Weight`，不是某个子层 defaultWeight |
| 当前控制器自己的子层 | ALC `playable=Action` + **本控制器局部索引** | MA 合并后再读实际索引；进入、退出分别为 1/0 |
| 其他插件的子层 | 不用硬编码最终索引跨控制器写 | 各插件控制自己的层，避免覆盖 Base 或别的动作 |
| VRCFury 多 owner 协调 | 当前 `ActionConflictResolverService.Apply()` 自动在 InitBehaviours 状态加入全局 Action PLC，并将功能子层默认置 0 | 必须检查真实 VRCFury 产物及 InitBehaviour；不能声称“子层 weight=1 就代表整个 Action weight=1” |

当前 VRCFury 源码位置：`Packages/com.vrcfury.vrcfury/Editor-Avatars/Service/ActionConflictResolverService.cs`、`AnimatorLayerControlOffsetService.cs`。索引 offset 服务对全部 used controllers 生效，**descriptor 原控制器的 ALC 也会重映射**；本轮一度怀疑 descriptor 漏 remap，后被真实产物否定，不能把该推断写成事实。

## 14.3 本轮落地与回归方法

- 舞蹈控制器加空基础层，把原舞蹈层放到索引 1、默认 0；原 5 个 ALC（Wait/Start×2/Stop/Restore）局部索引 3→1，保留各自 goal 与 blend。**不能删光这些 ALC 后只把舞蹈常开**。
- 为小狼复制 AFK Action/FX，作者源资产不改。Action 内控制全局的 PLC 改为只控自身层的 ALC；Wait/Sit 显式自身 0。FX 中两条原本也指向 Action 的重复 PLC 去掉，其他动画、参数与 TrackingControl 保留。
- 复制 SDK 默认 Action 为本次专用副本，前置空层 0、原动作层 1/default0；PLC 改为自身 ALC1，Wait/Sit 自身 0。保留原标准动作，不直接删除 SDK 全部功能。
- 移除自加 Enable 中的冗余全局 PLC；当前工程由 VRCFury 的多 owner InitBehaviour 负责全局 Action 启用。**这是带前提的当前工程方案**，未装 VRCFury/单 owner 时不能照搬“删掉所有 PLC”。
- 抽取控制器还残留写 `VF74_TC_*` 的 Driver，但本头像没有对应 Tracking Control Actor 消费层。改为舞蹈 Start 直接 Animation、退出 Tracking；Eyes/Mouth 依原动作语义保留或恢复。FBT 真机效果仍需单列验收，GM 不代替硬件客户端。
- `SyncDances` 是运行选择值，设 saved=false；Contact 瞬态 `SyncDance1..8` saved=false/localOnly=true，源 EP 若已有同名参数也同步检查。音乐/Contact 禁用偏好可保存。

验证顺序：默认态 → 原生 GM 菜单点击舞蹈 → 动作层权重与实际骨骼角变化 → Stop 归零 → Walk/Run → 舞蹈中 AFK → 退出 AFK 恢复 → 再选舞蹈。必须在**真实最终 VRCFury 产物**检查，不能只看 MA Manual Bake 控制器。首轮112项107 PASS/5 FAIL、第二轮108 PASS/4 FAIL的记录保留；移除冗余 Enable 后短程 Movement Probe **11/11 PASS**，Walk最大骨骼角62.49°、Run48.35°。完整最终结果写在工程 `docs/PLAN-001-r520/GM-full-validation.txt`，以有 SUMMARY 的最后一轮为准；客户端未据此验收。

**官方依据与来源**：
- <https://creators.vrchat.com/avatars/playable-layers/>（Action 默认权重 0）。
- <https://creators.vrchat.com/avatars/state-behaviors/>（PLC/ALC/Tracking/Driver 语义，0号内部层不可像普通子层控制）。
- <https://modular-avatar.nadena.dev/docs/reference/merge-animator>（同控制器索引、Playable匹配、层优先级与WD语义）。
- 当前 MA/VRCFury/GM 包源码；工程 `GM-dance-isolation.txt`、`GM-movement-probe.txt`、`GM-full-validation-first-FAIL5.txt`、`GM-full-validation-second-FAIL4.txt` 与独立复核报告。

