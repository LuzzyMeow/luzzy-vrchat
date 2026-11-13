# 参数与菜单审计：成本、上限、四表面同步

读这份文件的时机：加了开关/滑块/道具之后要核对是否超限，或菜单点了没反应需要排查参数归属，或要做「上传前参数体检」。

> **核心口径：VRChat 的表达式参数预算按 bit 计，不按个数计。** 数参数个数是最常见的第一号错误。

## 1. 成本怎么算

VRChat 表达参数（Expression Parameters）的同步成本按类型：

| 类型 | 成本 |
| --- | --- |
| `Bool` | 1 bit |
| `Float` | 8 bit |
| `Int` | 8 bit |
| `Bool`（作为 Puppet / 二态控件） | 1 bit |

总预算 **256 bit**（每个头像独立计算）。**最终成本以构建结果为准**，不要靠手工累加、不要从结构相似的头像复制一个数字。

### 1.1 三个极易搞错的规则

1. **非同步参数也要占参数条目名额。** `networkSynced = false` 或「只给 Animator 用」的参数 **sync bit 为 0**，但它**仍然计入参数条目上限**。所以「不同步 = 不占预算」是错的。
2. **显式写进 descriptor 资产的参数会被计费**，哪怕它的名字长得像内置输入（`Viseme`、`GestureLeft`、`Voice` 之类）。名字相似 ≠ 免费。
3. **重映射与重名规则必须算进去。** 同一个参数在 MA / provider 里被 remap、或被 provider 去重合并，最终成本可能低于你数出来的名字数，也可能因类型冲突而变高。

### 1.2 正确的读取链（顺序固定）

```text
确切根对象(exact root)
  → VRCAvatarDescriptor.expressionParameters（根资产声明）
  → 每个相关 provider 的参数源（Modular Avatar Parameters / FaceEmo / 道具库存 / 衣柜 / 手势 provider）
  → 名称/类型/default/saved/networkSynced 与 remap 关系
  → provider 自省 或 源级估算（PROVIDER_PREVIEW）
  → NDMF 合并后的参数列表（NDMF_BUILT）
  → SDK 最终成本与平台清单（SDK_BUILD）
```

**每个目标根单独算**；不要手工往场景里加组件试、也不要把结构相似根的成本抄过来。

### 1.3 只能标「待构建」的情况

以下情况**不要给数字**，记录 `BUILD_REQUIRED` / `NOT_RUN`：

- 目标根不明确（`AMBIGUOUS_TARGET`）；
- 同名参数类型不兼容；
- provider 版本或自省结果拿不到；
- 手里只有旧的源级估算。

源级估算可以给，但要标 **provisional（暂定）**，并明确最终成本 `BUILD_REQUIRED`。

### 1.4 常见错误清单

- ❌ 数参数个数而不是 bit；
- ❌ 以为 `networkSynced = false` 就完全不占；
- ❌ 忽略 remap 与重名去重规则；
- ❌ 把头像 A 的预算数字套到头像 B；
- ❌ 把源级报告当成上传已通过的证据。

## 2. 菜单控件与分页

### 2.1 每页控件上限

`VRCExpressionsMenu` 每页控件数有硬上限，**上限值要读 SDK 当前定义**（`VRCExpressionsMenu.MAX_CONTROLS`），不要记死一个数字——SDK 版本不同。超限的处理只有两条路：

1. **拆页**（把一部分控件移到子菜单）；
2. **合并**（把多个二态合并成一个多档 Int / Puppet）。

### 2.2 分页不是「裸 SDK 自动帮你做」

实测核对（MA 1.14.3）：Modular Avatar 的 `MenuInstallHook` 在**构建后处理**阶段提供「More」分页。也就是说：

- 分页是**provider 的构建期行为**，不是裸 SDK 对任意菜单的通用保证；
- 手工 `CreateAsset` 出来的一棵新菜单树**不会**自动获得分页；
- 其他 provider 的分页行为各自不同；
- **最终以新生成/构建后的对象为准**，不要凭编辑器里看到的层数判断。

### 2.3 菜单路由（MA 1.14.3 源码核对，其他版本用 schema/source/preview 重新确认）

1. **SDK 控件枚举公开值 ≠ `SerializedProperty.enumValueIndex`。** 用强类型枚举或当前版本定义，不要按 `0..5` 或资产字段组合猜 type；已安装 SDK 还可能有新枚举成员。
2. `ModularAvatarMenuItem` 取标签的顺序：**非空 `label` 优先**，否则取 GameObject 名。
3. `MenuAsset` 的子菜单资产来源与 `Children` 不同；**场景里的子对象只有被实际 MenuSource 消费才参与**——仅仅 parent 在一起不保证可达。
4. Installer 同物体上 `MenuSource` 与 `menuToAppend` 的**优先级必须用当前版本的解析器确认**。有安装目标组件时，检查它的 `installer` 指向以及是否参与枚举，**不能只看 enabled 状态推断路由**。
5. 当前版本 `MenuNodesUnder` **只遍历直接子节点**；这只说明 MA 1.14.3 的行为，不代表所有 provider 都只遍历一层。改来源/安装目标前先确认需求、共享菜单与全部引用，**不要盲删所有 InstallTarget**。
6. 用本版编辑器的 `VirtualMenu.ForAvatar(avatar, context)` 可以得到 **PROVIDER_PREVIEW**；internal 反射签名会变，必须先找准确方法。**不要自己复制一份完整合并算法**，也不要把 preview 说成 SDK 最终结果。
7. **未指定目标的 installer 可能安装到根。** 整理时保留 provider 归属与重生成源，不要把不同插件的菜单叶子混进一棵失去所有权的树里。

## 3. 「四表面」同步：一个参数必须在四处一致

一个表达参数在工程里同时存在于多个表面。改任何一处都必须同步另外三处，否则就是「面板上有但点了没反应」或「点了但值不对」。

| # | 表面 | 位置 | 内容 |
| --- | --- | --- | --- |
| 1 | **参数定义资产** | `VRCExpressionParameters` 资产（descriptor.expressionParameters） | 名称、类型、默认值、`saved`、`networkSynced` |
| 2 | **Animator 动画层** | FX controller 的层/状态/参数 | 参数名、类型、层权重、Write Defaults |
| 3 | **菜单控件** | `VRCExpressionsMenu` 的 `Controls[]` | 控件类型、`parameter`（或 `subParameters`）、图标、`subMenu` |
| 4 | **provider 配置** | MA Parameters / MA MenuItem、FaceEmo 配置、道具库存配置等 | 参数声明、菜单注入点、默认值 |

核对方法：**从实际控件反向追**（先读控件的 `parameter`/`subParameters` 与最终菜单，再往上找它的身份），而不是从名字猜。**名称前缀不表示保存/同步属性**。

### 3.1 类型必须从消费者反推

- 二态开关 → 通常 `Bool`；
- 多档（换色/换装档位）→ 通常 `Int`；
- 连续量/滑条 → 通常 `Float`；
- Radial Puppet → `Float`（见 [04-03-color-and-puppet.md](../04-外观与表情/04-03-color-and-puppet.md)）。

但**实际类型要读**：Animator 参数、菜单 `parameter/subParameters`、driver、blend tree、ExpressionParameters 五处的实际声明。冲突必须解决，**不要凭控件外观推类型**。

### 3.2 `saved` 与 `networkSynced` 是两个独立开关

- `saved = true`：要求**客户端持久化**该参数（跨世界/重载保留）；
- `networkSynced = true`：是否**网络同步**（别人能不能看到）。

两个互相独立。**设置 `saved` 不证明跨世界测试通过**——进入状态时的 driver、初始化/延迟驱动仍可能把它重置回去。所以：

1. 先找全部**写者**（尤其状态进入 driver、初始化/延迟驱动），否则 `saved=true` 也会被覆盖；
2. 修改**拥有它的稳定源配置**，记录类型、default/saved/synced/local 与所有写者；
3. **不要为了一个头像去改整个第三方包的源码**；
4. 只有旧标签包含「已不成立的保存行为描述」时才修实际生成源的标签；**不要无端改 GameObject 名**（会影响动画路径）；
5. 改完独立读回源定义；新生成后再核对最终参数身份/类型/default/saved/networkSynced 及菜单，**不要误改邻接参数**；
6. 在获授权的 VRChat 客户端里设一次 → 换世界/重载头像 → 检查是否恢复，并说明与模拟器的边界。没测客户端就写 `saved 已配置，跨世界 NOT_RUN`。

## 4. 烘焙后的菜单与参数才是真的

编辑器里看到的菜单/参数是**源**；用户看到的最终菜单是 **NDMF 合并 + MA 安装 + provider 构建后处理**的产物。差异来源：

- 菜单注入（installer 把衣服菜单挂到根/子菜单）；
- 参数去重与 remap；
- 生成器改写标签/控件类型/图标；
- 分页；
- 构建期移除未使用项。

因此审计结论分三层写清楚：

| 层 | 能证明什么 | 记录方式 |
| --- | --- | --- |
| **SOURCE / READBACK** | 源资产里确实这样写 | 可以直接下结论 |
| **PROVIDER_PREVIEW** | provider 自省/预览的估计 | 标 estimates（如「228 bit」），最终 `NOT_RUN` |
| **NDMF_BUILT / SDK_BUILD** | 用户实际会看到的 | 需要构建；没构建就 `BUILD_REQUIRED` |

## 5. 限额与优化策略

- **记录实际用量与剩余容量**，不要把「接近上限」本身当成缺陷。用户面向的结论用中性词：`WITHIN_LIMIT` / `AT_LIMIT` / `OVER_LIMIT` / `NOT_MEASURED` / `BUILD_REQUIRED` / `AMBIGUOUS_TARGET`。
- **不要主动问用户「要不要优化」**，除非：① 已超出平台硬上限；② 不加就会超；③ 用户明确要求优化。
- 超限时的盘点方式：**按 provider 分别盘点成本，不要跨 provider 归属**；把「菜单控件数」「同步参数 bit」「Animator/组件数量」「源资产」「构建总体积」「实测增量体积」分开列。
- **不要用文件夹体积代替 bundle 贡献**。

## 6. 审计产出模板

```text
目标根        : <确切 prefab 路径 / 场景实例>
参数来源      : <descriptor / MA / provider …>
参数清单      : 名称 | 类型 | default | saved | synced | 写者数
成本估算      : <NNN bit>@PROVIDER_PREVIEW 或 BUILD_REQUIRED
菜单控件      : 每页 N 个 / 上限 <读 SDK>
分页          : <provider 分页 / 手工拆页 / 未处理>
烘焙后        : NDMF_BUILT / SDK_BUILD（未跑写 BUILD_REQUIRED）
四表面一致性  : 资产 ✅/❌ | Animator ✅/❌ | 菜单 ✅/❌ | provider ✅/❌
剩余风险      : …
```

## 7. 边界

- 本分册不代替 SDK 的最终校验；所有数字都标清来源层级。
- 涉及第三方包内部实现时，**先读已安装版本的实际源码/schema**，不照搬旧版本结论。
- 参数重命名属于高风险操作：会打断别人的动画、菜单与 provider 引用。**没有授权不要做**。

## 8. 已存在 EP 参数的 `saved` 持久化：MA 菜单项不会回头写 EP（2026-10-21 第 59 轮实测）

### 8.1 症状

游戏内把某个滑块 / 开关调好（本例 `外观 → 整体亮度`，Float 参数 `发光`），**切出再切回模型就回默认值**；菜单、Animator、MA 组件看上去全对，Console 也不报错。

### 8.2 根因（先读 baked EP，别只读源资产）

- 持久化只看 EP（`VRCExpressionParameters`）里该参数的 `saved` 标志；本例 `发光`（Float，idx=22）`default=0.5`、**`saved=False`**、`networkSynced=True`。
- 场景里的 `ModularAvatarMenuItem` 写着 `isSaved=True` / `isSynced=True`，但**它只对 MA 自己新建 / 接管的参数生效，不会把「已存在的 EP 参数」的 saved 标志改回来** —— NDMF bake 生成的 EP 里仍是 `saved=False`。
- 判据：直接读 **baked EP**：`Packages/nadena.dev.ndmf/__Generated/<Avatar>(Clone)/_assets/assets <N>.asset` 的 `parameters[i].saved`。只看 Edit Mode 的源 EP 不够（两者可能不一致，以 baked 为准）。

### 8.3 修法（直接改源 EP 资产）

```csharp
var ep = AssetDatabase.LoadAssetAtPath<VRC.SDK3.Avatars.ScriptableObjects.VRCExpressionParameters>(path);
var ps = ep.parameters;
int i = -1;
for (int k = 0; k < ps.Length; k++) if (ps[k].name == "发光") { i = k; break; }
ps[i].defaultValue = 0.75f;   // 需要改默认值时一起改
ps[i].saved = true;           // ← 关键：持久化
ep.parameters = ps;
EditorUtility.SetDirty(ep);
AssetDatabase.SaveAssets();
AssetDatabase.Refresh();
```

- 回读验证：重新进 Play 等 NDMF bake，反射读 baked EP → 同名字段 `default=0.75`、`saved=True` 才算修好（本例通过）。
- 改前备份资产 + `.meta`；本工作区 `<VRCHAT_ROOT>\projects\LuzzyMeow` **2026-10-21 实测没有 `.git`**，只能靠 `.bak` 回滚。

### 8.4 边界与判据

- **Gesture Manager 不能验证「换头像后保留」**（那是登录态 / VRChat 本地保存行为）→ 未上传实测就写 `NOT_RUN（需进游戏）`。
- `saved` 与 `networkSynced` 是两个独立开关：本例只打开 saved、保留 synced=True。
- 默认值的「视觉含义」必须实测再定：本例 `发光` 的曲线是 0.5=中性、0=全暗、1=最亮，且 blend 曲线在 0.5 已饱和 → 0.75 与 1.0 视觉接近（详见 `04-外观与表情/04-03-color-and-puppet.md` §9）。


## 9. EP 的 `saved` 会盖掉默认值：为什么「改好默认有衣」进游戏仍然光着（本工作区实测 2026-10-25，第 77 轮）

> **⚠️ 第 166 轮裁决（2026-11-08）取代本节的历史取舍**：用户要求「所有开关都能记忆上次修改」后，现行口径是——菜单可见的 Toggle/Radial 对应参数一律 `saved=true`（源 EP 与 MA Parameters 两处口径一致），`defaultValue` 与场景默认态对齐以免无本地记录时裸体；面捕实时流（`FT/v2/*`）与瞬时 Button 参数保持 false。本节「53 项 saved=false」只作历史方案存档。详见 §10.3。

**症状**：菜单/参数默认值都确认是「有衣服」（`OutfitSelect=1`、47 个部件 bool `defaultValue=1`），
但用户上传后进游戏、或自己在编辑器 Play 里，仍然**光着**。

**根因（两条，分别对应两种“光着”）**
1. **进游戏仍光着**：VRChat 参数表里 `saved=true` 的参数会把本地上次的值当作初始值——用户之前把衣服开关点成 0 并被保存，
   下次加载时 `saved` 值覆盖 EP 的 `defaultValue`。改默认值只对「没有本地保存记录」的玩家生效。
   - 修法（本工作区采用）：把「一键穿衣/部件」相关 53 个 `ParameterConfig.saved` 从 true 改 **false**（MA Parameters 里改），
     这些参数就每次加载都回落 `defaultValue`（= 默认那套衣服）。代价：玩家自己换的衣服不会被记住。
2. **编辑器 Play 里也光着**：这是**编辑器态假象**——VRCFury/NDMF 的 Play 克隆里根 `Animator.controller = null`（控制器在运行时才挂上），
   所以 FX 层的 entry 动画不跑；此时用 GM 反射建 module 也读不到参数（`Params=0`）。
   判断「游戏里的真实默认」要看**烘焙后的 EP 默认值 + 烘焙后 FX animator 的参数默认值 + entry 转移条件**，不要拿 Play 克隆的可见性当结论。

**判据**：烘焙后 `EP.parameters[i].saved == false`（本例 53 项）、`defaultValue` 正确；FX animator 参数默认值 `S1_Hat=1 … OutfitSelect=1`、
entry 状态转移条件 `S1_Hat ≈ 1` 成立；此时上传后新实例默认即为「睡衣」那套。

## 10. EP 参数权威规则（枚举 / 位宽 / saved / 内置参数 / 类型校验）

> 来源：原 08-症状速查 第 147/150/152/153/163/174/177 轮实测（2026-11-06 至 11-11），2026-11-11 结构化并入。

### 10.1 ValueType 实测枚举顺序与位宽

`VRCExpressionParameters.ValueType` 实测顺序是 **`Int=0 / Float=1 / Bool=2`**，不是 `Bool=0/Int=1/Float=2`（第 152 轮）。位宽 `Bool=1`、`Int/Float=8`。
- 验证法：Unity 里 `Enum.GetNames/GetValues(typeof(VRCExpressionParameters.ValueType))` 直接打印；对照 `VRCEmote`（官方 Int）为 0、`VRCFaceBlendH`（Float）为 1、`PJ_Shirt`（Bool）为 2。
- 位宽只能按实测枚举算：`valueType==Bool ? 1 : 8`，不要猜数字。本工程真实值：51 项 / 43 synced / **71 bits**。
- Animator 侧 `m_Type` 是另一套枚举（Float=1/Int=3/Bool=4），三处类型对齐按类型名不按数字（[04-08 §2.2](../04-外观与表情/04-08-face-tracking-osc-and-repair.md)）。

### 10.2 位宽权威算法（手工累加一律不采信）

- **NDMF 权威计算器**：`nadena.dev.ndmf.ParameterInfo.ForUI.GetParametersForObject(avatar)`（反射调用，`BitUsage` 已内含 `IsAnimatorOnly/WantSynced/null` 判定）→ 本例 `params=190 TOTAL_BITS=195 LIMIT=256`（第 153 轮）。文档写「133 项/413 bits 超限」是手工按名字累加+类型映射搞反的数字，不是构建真实值。
- **SDK 权威**：`VRCExpressionParameters.CalcTotalCost()`（第 177 轮：265→249≤256）。
- **合并产物才是真值**：NDMF/VRCFury 合并后的 EP 把所有插件参数都算进去（`ABT/*` 42 项、`CC2/*` 5 项、`LightController/*` 4 项…），不是源 EP 的 39 项（第 150 轮：源 51 项 100 bits 正常，合并后 413/256 超限）。上传前单独核对合并产物位宽；超限时优先把可选功能参数改 NotSynced 或裁掉。
- 构建成功本身就是 ≤256 的充分证据（SDK 先校验再构建）。saved 以烘焙 EP 读回为准。

### 10.3 saved 四象限与现行裁决

- **记忆只由 EP 的 `saved` 决定**，与菜单项类型（Toggle/Button）、MA 面板 Saved 勾选无直接关系；MA 面板的 `saved` 只在构建时影响它**新追加**的参数（第 152 轮）。
- 先跑 `saved/synced` 四象限统计再决定改哪里：`saved && networkSynced` 才真正会被记忆；synced 但未 saved → 只改 EP；**本地参数（networkSynced=false）写 saved=true 是空操作**，SDK 判 `savedButNotSynced`，要记忆必须先改同步。
- **改 EP 的 saved 是安全且充分的**：`ParameterAssignerPass` 先用 `declaredParams`（EP 已有参数按名去重）建字典，只有 EP 中不存在的参数才进 `newParameters`，最后 `parameters.Concat(newParameters.Values)` 追加；已有条目原样保留（`ParameterAssignerPass.cs:60-70,190-200`；实测改完 EP 再 SaveAssets，回读 43 项 saved=True 稳定）。MA 侧只需同步改「它自己声明且值为 false」的条目，避免语义分叉。
- **现行裁决（第 166 轮，取代 §9 历史取舍）**：菜单可见的 Toggle/Radial 对应参数一律 `saved=true`；同步改源 EP 与 MA Parameters 两处；`defaultValue` 与场景默认态对齐。面捕流/瞬时按钮参数保持 false。**GM 不验证跨世界持久化 → 客户端未实测写 NOT_RUN**。
- 混搭持久化的最后一道锁可能在 MA Parameters 显式声明（`OutfitOneClick def=1/saved=true` 盖掉源 EP）：三处一起改 + 清 `OutfitSync/Default` driver（第 141 轮，[03-06 §19](03-06-ma-menu-build-by-reflection.md)）。

### 10.4 localOnly 不移除源 EP 已有参数；同名多声明任一 synced 即 synced

第 177 轮实测（2026-11-11）：
- MA Parameters 设 `localOnly=true` 后烘焙 EP 仍含该参数、synced 预算不降：**源 EP 资产显式存在的参数（如 `OutfitOneClick`）不受 MA localOnly 移除**；须直接改源 EP 的 `networkSynced=false`。
- 同名参数多处声明时 MA 合并取「**任一 synced 即 synced**」（OutfitOneClick 三处声明需全 local）。
- 判据：预算以 `CalcTotalCost()` 权威取数（本轮 265→249≤256）；saved 以烘焙 EP 读回为准（`superneko.realkiss.mode$<hash>`、`negaline` saved=True）。

### 10.5 内置参数撞名：duplicate definition（IsLocal / AFK 两条触发路径）

客户端加载报 `Error - [Behaviour] Avatar Expression Parameter 'X' duplicate definition, New definition will be ignored and original type 'Bool' will be used.` 时，是 EP 里的声明与 VRChat 内置参数撞名。**编辑器不报，只有运行时才有**。

**路径 A（第 163 轮，IsLocal）：EP 已有同名 Float + MA 误导出**
- EP 里有一条 `IsLocal` Float（networkSynced=False, saved=False），FX 里 `IsLocal` 是 Bool；VRChat 按「先定义者胜」丢弃后者。真正违纪点是 `FT_Merge/ModularAvatarParameters` 的 `IsLocal` 被当成普通同步参数（`internalParameter=false, syncType=2`）导出到 EP。
- 修法双管齐下：① 从 EP **删掉** `IsLocal`（Float）；② MA Parameters 该条 `internalParameter=true`、`syncType=Bool(3)` → 变成 MA 内部名 `IsLocal$<hash>`（示例 `IsLocal$308be432de39`，Bool、20 字符）。

**路径 B（第 174 轮，AFK）：EP 源资产干净、MA Parameters 显式声明内置名**
- `AFK` 是 VRChat **内置只读 Bool**（HMD proximity / End 键），官方文档明确内置参数不可由菜单/OSC 写。EP 源资产 90 条无 AFK；但场景节点 `まめフレンズAFKくれーんげーむーむ` 的 MA Parameters 有一条 `nameOrPrefix='AFK' syncType=Bool(3)` → MA 把 synced 条目导出进 EP → 客户端撞名（本次会话 5 次；历史 24 份日志 AFK 计数 0，确认是新引入）。
- 修法：该条 `syncType` 改 **`NotSynced(0)`**（UI 名 "Animator Only"）：**不删条目、不删 FX 层**。内置参数由客户端直接驱动 animator 绑定，无需 EP 登记；MA 源码 `ParameterPolicy.cs:412` 对 NotSynced 显式 `continue` 跳过 EP 导出。
- 全场景扫描确认只有这一处 AFK 声明（防 `RenameParametersHook.MergeCommon` 从其它声明吸附 syncType）。
- 判据：四跳 dump 修复前 `HOP2 EP type=Bool def=0 saved=False synced=True`；修复后 **`HOP2 EP MISSING`** 且 `AFK`、`R400_AFK_Visibility` 两层仍 weight=1.00、If/IfNot 转移与 m_IsActive clip 齐全；全量菜单 51/叶子 222/层 142 不变。GM animator 键仍含 AFK（内置绑定不需要 EP 条目）。
- 官方出处：<https://creators.vrchat.com/avatars/animator-parameters/>、<https://modular-avatar.nadena.dev/docs/reference/parameters>。

### 10.6 EP↔FX 全量类型校验（提前发现隐患）

编辑器不校验 EP↔FX 类型一致性，SDK Alerts 也不含这一项（第 163 轮）。
- 配方：遍历 `ep.parameters` 与 `fx.parameters` 按 name 建字典比对：Float↔Float、Int↔Int、Bool↔(Bool 或 Float) 视为兼容，其余计入 mismatch。本工程源资产修完 IsLocal 后 `match=17 mismatch=0 notInFx=72 total=89`。
- 每次改 MA Parameters 或 EP 后跑一次；`notInFx` 大是正常的（EP 里很多参数只给菜单用，FX 不声明）。
- 参数名长度不必担心：本工程 EP 现存最长 **66 字符**（`superneko.realkiss.auto.getleadertimingcounter_nosync$91388a5c02cd`）已正常上传；SDK `FaceTrackingParameterBehaviorDrawer.cs:285` 对 outputParameterName 截断到 **64**。

### 10.7 同名参数 hash 冲突（两处声明 = 一方被改名）

菜单写 `LH_foot`、衣服控制器也写 `LH_foot`，结果菜单点了没反应（第 147 轮）：两处各自注册同名 MA 参数，**MA 判定冲突后给其中一方加 `$hash` 后缀**（烘焙产物 `LH_foot$61256ccb7a41`），控制器只认带 hash 的那个。
- 判据：烘焙 EP 里同时出现 `LH_foot` 与 `LH_foot$61256ccb7a41`。
- 修法：参数**只保留一处定义**——删掉 prefab 自带的 MA Parameters，统一由菜单节点的 Parameters 定义（同名同默认值也不行，必须唯一）。

### 10.8 默认值写不进控制器：m_overrideAnimatorDefaults 与未声明参数

- **`ParameterConfig.m_overrideAnimatorDefaults` 是 internal 字段、默认 false**；只有它为 true（或 `syncType=NotSynced` 且有默认值）才会进 `InitialValueOverrides`、被 `ApplyAnimatorDefaultValuesPass` 写进控制器（第 150 轮；`ModularAvatarParameters.cs:34-43`、`RenameParametersHook.cs:190-194`、`ApplyAnimatorDefaultValuesPass.cs:23-53`）。MA Parameters 写了 defaultValue=1 但烘焙后控制器参数仍 0，就是它。
  - 修法：用 **SerializedObject** 改（`SetValue` 对 struct 装箱副本无效，静默丢写）：`new SerializedObject(mp).FindProperty("parameters")` → 元素 `FindPropertyRelative("m_overrideAnimatorDefaults").boolValue=true`。官方字段语义见 [03-07 §6.4](03-07-ma-platform-sync-mmd-and-motion.md)。
- **MenuItem 引用了参数 ≠ 它在 MA Parameters 里有默认值**：未声明的 Bool 按 false 处理，EP 也按 0 合并（第 150 轮：`PJ_*` 有声明正常、`YURU_*` 未声明全不显示）。默认要开的开关必须显式声明 + `defaultValue=1` + `hasExplicitDefaultValue=true`。
- **第三方换装控制器的参数不在 EP 里**：运行时按 0 初始化；若控制器 `defaultState=On` 且 On 写 `m_IsActive=1`，则 0=显示 → 没穿那套也全显示（第 150 轮：`RT/*` 12 项）。把这类参数**补进 EP 并显式给默认值**；默认值取哪个由极性三轮实测定案（[04-06 §14](../04-外观与表情/04-06-ma-reactive-components.md)）。

### 10.9 出包审计两条红线

每个菜单页最多 **8 项**、同步参数最多 **256 位**（Bool 1 / Int·Float 8）。加新功能前先看：同步位剩多少、目标页还剩几格。出包后审计配方（遍历菜单页容量 / 位成本 / 菜单项参数注册 / 第三方内部参数白名单）见 [06-06 §3.3](../06-构建上传与备份/06-06-upload-credentials-and-client-evidence.md)。
