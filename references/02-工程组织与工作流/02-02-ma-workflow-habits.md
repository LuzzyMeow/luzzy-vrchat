# MA 化工作流与改模习惯

> 本册基于一套 MA 化实战规范整理（源仓库信息见 `ACKNOWLEDGEMENTS.md`）；本册是可复用的全部结论，**不依赖原仓库**。只读辅助脚本的用法见 `07-05-ma-gotchas-version-toolchain.md` §8。
> 本包已有内容不重复：参数成本/菜单路由见 `03-01-parameter-and-menu-audit.md`，MA 声明 vs 构建产物、NDMF 烘焙与验收见 `07-01-bake-and-validation.md`，证据阶梯/授权矩阵/所有权边界见 `07-02-evidence-and-ownership.md`，
> 装衣服与骨架合并见 `02-01-workspace-and-work-assets.md`，MA 逐组件配方（Extract Menu / Material Setter / 形态键轮盘 / 动骨触发）见 `03-04-ma-menu-and-switch-cookbook.md`。
> 本分册补的是**「组织与习惯」这一层**：目录/菜单结构、互斥参数机制、命名与 codegen 约定、商业模块集成、验收清单。

## 0. 适用场景与工具纪律

### 0.1 什么时候读本分册

- 给头像**新加一套衣服 / 发型 / 配饰**。
- 把散乱的层级**重组成** `Clothes/` / `Hairs/` / `Deco/` 分类容器。
- 新建或修复 Clothes/Hairs 菜单树（每套一个子菜单、`all` 总开关、部件开关分组）。
- 排查互斥失效（两套同时穿、默认套装不对、进图后什么都没穿）。
- 集成商业 MA/NDMF 模块（Nova、RBS_Suimin、LightController 等）。
- 选择/配置/校验 MA 组件；用 `AvatarProcessor.ProcessAvatar()` 做构建验证。

### 0.2 工具纪律（不可协商）

- **只用原生 `unityMCP_*` 工具**做 Unity 的实时检查、组件改动、场景操作、Console 读取。这些工具由运行时注入。
- **禁止**用 Python / JavaScript / PowerShell / curl / 裸 JSON-RPC 去调 MCP 端点。Shell 只允许用于**本地包与文件检查**，永远不能用来绕开 MCP 的权限与审计。
- `unityMCP_*` 工具不可用时：**停下并报告「工具未注入」**，不得静默回退到 HTTP 或 batchmode。
- 走 `unityMCP_execute_code` 的 C# 是**方法体**，可用 `UnityEngine`/`UnityEditor` 命名空间；用 `return <string>` 回传结果；受 CodeDom/C#6 限制（**没有局部 void 函数 → 用 lambda**）。
- 本机提醒：DSH 已按 `mcp-unity` 接入（工具前缀 `mcp__unity__*`），但**必须有 Unity 编辑器打开工程**才连得上；未连上时任何 MCP 步骤都只能是纸上方案（见 `07-04-verification-discipline.md`）。

### 0.3 实现方式优先级：MA 优先，尽量不用 VRCFury（本工作区铁律）

用户 2026-10-17 拍板（已写进 `SKILL.md` 铁律 4）：**能用 Modular Avatar 表达的一律用 MA（+ NDMF），不要为了省事引 VRCFury。**

| 想做的事 | 用 MA | 不要用 |
| --- | --- | --- |
| 对象 / 部件开关 | `ModularAvatarObjectToggle`（+ MenuItem） | VRCFury Toggle |
| 带独立骨架的衣服、资产合并进头像 | `ModularAvatarMergeArmature`（+ Mesh Settings） | VRCFury Armature Link |
| 身体与衣服形态键同步 | `ModularAvatarBlendshapeSync` | VRCFury Blendshape Link |
| 把现成控制器并进某个 playable 层 | `ModularAvatarMergeAnimator` | VRCFury Full Controller |
| 菜单 / 参数 | `MenuItem` / `MenuInstaller` / `Parameters`（自动生成优先） | VRCFury Menu / Params |
| 换材质 | `MaterialSwap` / `MaterialSetter` | VRCFury Material |
| 裁掉身体多边形 | `MeshCutter`（+ 顶点过滤器） | —— |

纪律：

- 拿到**带 VRCFury 组件**的 prefab：先换再删 —— 换成 MA 等价件 → 删掉 VRCFury 组件 → 跑一次 NDMF 构建 / Manual bake 验证 → 再继续改。
- 第三方模块**只提供 VRCFury 组件**、无法等价替换时：**保留并记录**，不要为了「纯 MA」硬拆导致功能损坏。
- 想知道「VRCFury 到底做了什么」不要靠猜：先 bake 看**输出**，再回源组件找等价物（原则 7「尊重真源」）。

## 1. 动手前锁定目标（Lock the target）

逐项记下**确切**值，不要凭印象：

1. 工程路径与 Unity 版本；
2. 选中的 Unity MCP 实例，以及**已证实的**该实例的工程路径/Unity 版本；
3. 活动场景、是否已保存/有未保存改动、是否在 Play Mode、编译/导入是否完成；
4. 头像的**层级路径**（注意名字里有空格、重名的风险）；
5. 用户请求的确切改动，以及本次是否**授权写入**。

禁止用下面这些来「推断」目标：对象最近被用过、当前 active 状态、Pipeline Manager 的 ID、结构相似性、上一次会话的编辑器实例。
多头像场景里要**动态定位**（例如按精确层级路径），**不要硬编码**会被改名的头像名字。

## 2. 十条核心原则（不可协商）

1. **按套装分组的组织方式**：衣服/头发放在头像根下的分类容器里 —— `Clothes/<outfit>/`、`Hairs/<outfit>/`。新套装是**同级文件夹，绝不嵌套**进已有套装。
2. **用共享参数实现互斥**：所有套装共用一个 `cloth` 参数（单套 → 自动 Bool；2 套以上 → Int，各套不同值 Default=1、next=2…）；头发用 `hair`。任何时刻只穿一套衣服 / 一款头发。
3. **默认穿着由 MA 决定，不由场景状态决定**：编辑器里所有套装容器都是 `active=False`；真正决定进图穿什么的是**标了 `isDefault=True` 的菜单项**。`isDefault` 指向「用户想要的初始外观」，**不一定**是名字叫 Default 的那个容器。
4. **系统节点留在根层**：`Armature`、`Body`、`Body_base`、`Ground`、`AutoAnchorObject`、`VRCHeadChop` 永不移动。
5. **开关与滑条分开**：on/off 开关 → MA ObjectToggle；连续形态键滑条 → **留在 FX 层**。
6. **绝不改商业资产**：商业模块的路径问题用 MA 配置解决（例如 MergeAnimator 的 `relativePathRoot`），**不要**去改它的控制器/动画。
7. **尊重真源（source of truth）**：MA 组件与源 prefab 是**声明**；NDMF 预览、构建克隆、合并后的控制器、生成资产都是**输出**。能用源组件修好的，绝不修输出。
8. **参数自动优先**：能让 MA 自动生成参数就用自动（空 `param` + `automaticValue=true`）。复用头像原有的 Parameters 资产（**先备份成 `.bk`**）；**只有**互斥需要显式 Int 值、或确实需要特定的 saved/synced/改名语义时，
  才建 `ModularAvatarParameters` 组件。单套衣服的 `cloth` 保持 Bool，**等第二套来了再升级成 Int**。
9. **菜单文件策略**：新建一个**空的**菜单资产作为根（例如 `_MA_Root`），把原根菜单备份（`.bk`），AvatarDescriptor 的 `expressionsMenu` 指向这个空根（构建时由 MA 填充）。原菜单内容：
  能干净重建的用 `ModularAvatarMenuItem` 重建（重做出来的才是干净的 MA 控件）；不方便重建的，把它作为 `原有菜单` 子菜单，**原样引用原菜单资产**。
10. **根菜单分三组**：`_MA_Menu` 根下分 `原有菜单` / `Clothes` / `装饰` 三个顶层子菜单，每个都是 `Control.type=SubMenu` + `MenuSource=Children` + 
  `menuSource_otherObjectChildren=null`（用自己孩子）。散件装饰（kemo 等）放在头像根下的 `Deco/` 容器，配 `装饰` 子菜单。

## 3. 层级组织

```
avatar root
├── Armature / Body / Body_base      基础（永不移动）
├── Ground / AutoAnchorObject        物理/锚点（永不移动）
├── VRCHeadChop                      VRC 组件（永不移动）
├── Clothes/
│   ├── Default/                     默认套装（容器，含各衣物节点 + 内衣）
│   └── GLEIPNIR_Milfy/              新套装（并列添加）
├── Hairs/
│   └── default/                     默认发型（容器）
├── Deco/                            独立装饰容器（kemo 等散件）
│   └── kemo
├── _MA_Menu/                        菜单树根（挂 MA 组件）
└── ...                              独立系统模块平铺根层
```

- 套装容器是**单位变换的空组节点**（identity transform，本身不含网格）。
- **内衣**放**套装内部**（可用部件开关脱掉）；参考工程 Chiffon 的做法是把内衣常驻根层。**二选一，并保持一致**。
- 系统模块（RBS_Suimin、LightController、SPS、小游戏…）**平铺在根层**，各自是独立 MA 模块。
- 不属于任何套装的散件装饰（kemo、别针、独立配饰）放 `Deco/` —— 它是**分类容器，不是套装**，所以它的开关是独立的、不受 `cloth` 影响。
- ⚠️ 很多功能插件要求对象位于**角色的直接子级**（靠路径定位）——分类容器加得太深会让插件失效（见 `05-02-bind-animation-physbone-fbx.md` §1）。

## 4. 菜单组织

```
Root (_MA_Root，空资产；构建时 MA 填充)
├── 原有菜单（原模型菜单，SubMenu, Children）
│   ├── 設定 → kuuta_JP_Settings（MenuAsset 引用原资产）
│   ├── 表情セット → kuuta_JP_FaceEmote_PatternSwitchin_表情セット
│   ├── 表情固定 → kuuta_JP_FaceEmote_FixedEmote_表情固定
│   ├── ハンドサイン → kuuta_JP_HandsignPattern_ハンドサイン切り替え
│   ├── しゃがみポーズ → kuuta_CrouchingChange
│   └── 視線制御 → kuuta_JP_EyeControl
├── Clothes（SubMenu, Children）
│   ├── default（默认套装子菜单）
│   │   ├── all（cloth=1, isDefault=True）
│   │   ├── Shoes / Socks（自动参数）
│   │   └── clothes01 OFF / clothes02 OFF / clothes03 OFF（部件开关, 自动参数）
│   └── Theologica（新增套装子菜单）
│       ├── all（cloth=2）
│       └── coat_off / coat_jacket_off / coat_chain_off / boots_ON/OFF（自带参数）
└── 装饰（SubMenu, Children）
    └── kemo OFF（自动参数, OT→Deco/kemo）
```

规则：

- **每套衣服 = Clothes 下的一个子菜单**（`default/`、`Theologica/`…）；`all` 总开关与部件开关都在**套装子菜单内部**，绝不摊平到 Clothes 下。
- 默认套装的子菜单名用小写 `default`。
- 部件开关超过 8 个时按 **上装/下装/饰品/鞋** 分组，避免 VRChat 自动生成 `More` 翻页。
- 部件开关命名 `<part> OFF`（**开启这个开关 = 隐藏该部件**）。
- 套装总开关用共享 `cloth`，头发总开关用共享 `hair`。
- 顶层菜单：`Control.type=SubMenu`、`MenuSource=Children`、`menuSource_otherObjectChildren=null`（用自己孩子）。
- **原菜单内容优先用 MA MenuItem 重做**（干净）；实在不方便才在 `原有菜单` 下**原样引用**原资产（**永不编辑商业菜单资产**）。
- **商业模块的部件开关**：直接用 MenuItem 绑模块**自己的参数**（`Control.parameter.name` = 例如 `coat_off`，`automaticValue=false`），**不要**去引用模块的菜单资产；不需要重复声明参数（模块控制器自己驱动）。同时**移除/禁用模块自带的 MenuInstaller**，否则它会自己装到头像顶层。

## 5. 菜单与参数资产的备份习惯

MA 化一个存量头像**之前**，把原根菜单与 Parameters 资产**复制成同级 `.bk` 文件**（例如 `kuuta_JP__Root.bk.asset`、`kuuta_Param .bk.asset`）。然后：

- AvatarDescriptor 的 `expressionsMenu` → 新的空 `_MA_Root.asset`（构建时 MA 填充）。
- AvatarDescriptor 的 `expressionParameters` → **仍是原来的** Parameters 资产（已备份、保留）。除非确有必要（见 §2 第 8 条），**不要**建 MA Parameters 组件。

## 6. 开关配置

### 6.1 套装总开关（master）

- 菜单项节点上挂 **1 个 MA ObjectToggle**。
- `Objects` 指向**套装根容器**，`Active=true`。
- MenuItem：`param=cloth`，每套一个不同 value，`isDefault` 标出默认套装。

### 6.2 部件开关（part）

- MenuItem（Toggle）+ MA ObjectToggle，`Active=false`（**开关 ON = 隐藏该部件**）。
- `Objects` 指向**具体部件节点**。

### 6.3 互斥（mutual exclusion）

- 所有套装共用 `cloth`；点某套 = 把 `cloth` 设成它的值；只有值匹配的套装会激活。
- **单套**：`cloth` 自动生成 Bool —— 可以。
- **2 套以上**：`cloth` 必须是 **Int**，用 MA Parameters 组件（或显式参数）声明，`defaultValue` = 默认套装的值。每套的 `all` MenuItem 手动给不同的 `Control.value`（Default=1、next=2…）。
  `automaticValue` **可以保持 true** —— 已有显式 Int 声明时 MA 尊重手动值，互斥不会被破坏（作者实测）。默认套装的 `all` 勾 `isDefault` 标记启动外观。
- **自动提升（auto promotion）行为**：如果 `cloth` 完全不声明，2 个自动开关会编成 **Bool**（值被强制成 0/1）；出现**第 3 个**自动值时 MA 才自动升成 Int，但值是**从 0 顺排**、`defaultValue` 绑到**第一个被声明的**开关 —— 未必是你想要的默认套装。所以多套互斥的**可控路线始终是显式声明 Int**。

## 7. 参数命名与改名规则

| 类别 | 命名 |
| --- | --- |
| 套装总开关 | `cloth`（单套 Bool / 多套 Int） |
| 头发总开关 | `hair`（单套 Bool / 多套 Int） |
| 部件开关 | MA 自动参数 `__MA/AutoParam/<name>$hash`（MenuItem 的 `param` 留空） |
| 滑条 | 沿用原 FX 参数名（BreastSize / Sleeve / BLength…） |
| 系统/小工具 | 命名空间前缀（`RNW/Nade/`、`ABT/`、`SoundPad/`…） |

- **自动优先**：让 MA 自动建参数（`param` 留空 + `automaticValue=true`），除非该参数需要显式的 saved/synced/default/Int 语义。复用原 Parameters 文件；避免 `ModularAvatarParameters`。
- **Radial Puppet 轮盘参数放 `subParameters`，`parameter` 留空。**
- ⚠️ **`param` 留空的节点改名会改掉自动参数名** —— 部件开关、以及用空 param 的滑条（如 Front Length / Twintail Length）**都不要改名**。
- 可以安全改名的是**固定参数**项：`Nail` / `Face` / `FHSharp`、`all` 总开关、`KumaPhone`。
- 品牌名保持英文（Nova / @MA_Lili / GLEIPNIR / KnitNoise / Default）；商业模块内部名字一律不动。

## 8. codegen 约定（走 `unityMCP_execute_code`，C#6）

- **中文菜单名用 Unicode 转义**：`\u4E0A\u88C5` = 上装。直接写中文会变成 `?` / 乱码。
- 不支持局部 `void` 函数 → **用 lambda**。
- `SetParent` 用接受 `Transform` 的重载，注意 `var` 的类型推断。
- **不要在 `foreach` 里修改正在遍历的集合**。
- 用 `Convert.ToBase64String(UTF8.GetBytes(name))` **核对真实名字**（终端里显示乱码是正常的，base64 才是真相）。

## 9. 集成商业 MA 模块

1. 放到**正确分类**（`Hairs/`、`Clothes/` 或根层）。
2. **保留模块自带的 MA 配置**（MergeArmature / BoneProxy / MergeAnimator 等）。
3. **移除/禁用模块自带的 MenuInstaller**（或把 `installTargetMenu` 指到你自己的菜单），否则它会自己装到顶层。
4. 加**总开关 `all`**（共享 `cloth`/`hair` + 手动值）。
5. **部件开关直接绑模块自身参数**（`Control.parameter.name` = `coat_off` 等，`automaticValue=false`），无需重复声明。
6. 模块容器 `active=False`，启动态由 `isDefault` 决定。

### 9.1 重排后修动画路径（比改商业动画优先）

模块控制器如果**从模块名开始绑路径**（例如 `Nova/~Wing`），而你把它移进了 `Hairs/Nova` —— 把该模块 **MergeAnimator 的 `relativePathRoot` 设成模块父文件夹**即可：MA 会给缺失的前缀补路径。**优先于改商业动画**。

### 9.2 NDMF 插件型资产（LightController 模式）

有些资产是 **NDMF 插件**（包内有 `Editor/NDMFPlugin.cs`、`ExportsPlugin`），在**构建期**生成菜单/参数/动画。这类不要手工接线：

- 只确认它生成的菜单**不冲突**；
- 只有需要挪位置时才设 `installTargetMenu`。

## §10 独立配件（精灵耳模式）

自带小骨骼系统的刚性配件：在**配件根**加 **BoneProxy**，指向目标头像骨骼（例如 `Head`）。

⚠️ **普通配件不要用 OutfitRoot** —— 那会把它标记成独立套装，可能导致它所在的套装容器**构建时激活失败**。

## §11 标准流程：把存量头像 MA 化（A0 摘要）

1. 动手前把原根菜单 + Parameters 资产备份成 `.bk` 同级文件。
2. 如果头像是 prefab 实例，先解包：`PrefabUtility.UnpackPrefabInstance(rootGO, PrefabUnpackMode.OutermostRoot, InteractionMode.UserAction)` —— Unity 不允许重定父 prefab 实例的孩子；商业 prefab **资产**保持不动。
3. 新建空根菜单资产（`_MA_Root.asset`），AvatarDescriptor 的 `expressionsMenu` 指向它；建 `_MA_Menu`（MA MenuInstaller + MenuGroup，`menuToAppend=null`）。
4. AvatarDescriptor 的 `expressionParameters` **保持指向原 Parameters 资产**（已备份）。非互斥需要 Int 时不加 MA Parameters。
5. 原菜单顶层控件能重建的用 `ModularAvatarMenuItem` 重建（設定/表情/手势/姿势/視線…）；不方便重建的用 `Control.subMenu` **原样引用**原资产 → 放进 `原有菜单` 子菜单。
6. 重排节点：`Clothes/Default/`（可穿物）、`Deco/`（散件）。坏掉的旧动画逻辑用 ObjectToggle/ShapeChanger **重建**，**不要**去改商业控制器。
7. 加 Clothes/装饰 顶层子菜单（`Control.type=SubMenu`、`MenuSource=Children`、`menuSource_otherObjectChildren=null`）。每套衣服 = Clothes 下一个子菜单（默认 = 小写 `default`），
  `all` + 部件开关在子菜单内部。部件开关：空 param 的 MenuItem + ObjectToggle(`Active=false`)，命名 `<part> OFF`。商业模块部件：直接绑模块自己的参数，移除/禁用它的 MenuInstaller。`all`：
  ObjectToggle → 容器 `Active=true`；单套自动 Bool 可以；多套用 MA Parameters 显式声明 `cloth` Int + 手动不同值（`automaticValue` 可保持 true）。
8. 在**克隆体**上跑 `AvatarProcessor.ProcessAvatar()`（PROVIDER_PREVIEW）：验证构建出的菜单树、参数、动画 clip 绑定；看 Console；保存场景。

## §12 新增一套衣服 / 发型（已 MA 化的头像，9 步）

1. 模型放到 `Clothes/<outfit>/`（或 `Hairs/<outfit>/`）。
2. 有独立骨架？→ 上 MA 组件组：**MergeArmature**（挂它的骨架根，`mergeTarget` = 头像主 Armature）+ **MeshSettings**（RootBone/ProbeAnchor 指向主 Armature）。确认骨骼映射能解析。
  自带小骨骼系统的刚性配件 → 配件根 **BoneProxy** 指向目标头像骨骼。**普通配件不要用 OutfitRoot**。
3. 在 `_MA_Menu/Clothes` 下建套装子菜单（`ModularAvatarMenuItem`，`Control.type=SubMenu`、`MenuSource=Children`）。每套一个子菜单（默认套装 = 小写 `default`）；部件开关在子菜单**内部**。
4. 加 **`all`** 总开关：
   - MenuItem Toggle，`param=cloth`，每套手动不同 `value`（Default=1、next=2…）。单套：不需要 Parameters（自动 Bool 即可）。2 套以上：
  用 MA Parameters 组件把 `cloth` 声明成 **Int**（`defaultValue` = 默认套装值）；`automaticValue` 可保持 `true`（有显式 Int 声明时 MA 尊重手动值）。
   - MA ObjectToggle → 指向套装根容器，`Active=true`。
5. 部件开关：按 上装/下装/饰品/鞋（或头发的各部件）分组，保持每页 < 8 控件。每个部件开关：MenuItem Toggle（`param` 留空 → 自动参数）+ ObjectToggle `Active=false`（开关开启 = 隐藏部件）。商业模块的部件：
  **直接绑模块自己的参数**（`Control.parameter.name` = 例如 `coat_off`，`automaticValue=false`），不要引用它的菜单资产；并移除/禁用模块自带 MenuInstaller。
6. 默认套装的 `all` 勾 `isDefault=True`。
7. 场景里容器保持 `active=False`。
8. 用 codegen 写中文菜单名时要 **Unicode 转义**（直接写中文会变 `?`）。
9. 在克隆体上用 `AvatarProcessor.ProcessAvatar()`（PROVIDER_PREVIEW）或 `ProcessAvatarUI()` 验证，看 Console，保存场景。

## §13 安全执行回路（每次改动都跑）

1. **Observe**：用 `unityMCP_*` 工具查当前对象、组件、路径、资产引用与 Console 基线。
2. **Plan**：写明要创建/修改的确切对象与资产，以及**回滚方式**（Unity Undo / 删除新对象 / 移除新组件 / 从副本恢复 / 丢弃未应用的 prefab 覆盖）。
3. **Approve**：破坏性或对外有影响的动作要先要授权（删资产、Apply 覆盖、改导入的 prefab、只要求检查却要保存、升包、上传头像）。
4. **Apply minimally**：一次只做一个连贯的改动，全部走原生 `unityMCP_*` 工具。
5. **Re-read**：重新读取结果组件与引用，**不要**假设成功。
6. **Preview**：按改动性质选 MA/NDMF 预览、Play 模式或构建验证。
7. **Check Console**：与改动前的基线对比新增报错/告警。
8. **Report**：列出改过的 GameObject、文件、组件、参数与未解决的告警。

## §14 每次改动后的验收清单（12 条）

- [ ] 目标已锁定：确切头像路径、实例、场景脏状态已记录。
- [ ] 场景里套装容器 `active=False`；默认套装靠 `isDefault=True` 生效（在构建克隆上验证，PROVIDER_PREVIEW）。
- [ ] 单套 `cloth` 可以是自动 Bool；**2 套以上 `cloth`/`hair` 必须是 Int**（经 MA Parameters），且 default = 默认套装的值（**不是 0**）。
- [ ] 互斥组内各 `all` 的手动值互不相同；没有两套共用同一个值（参数已显式声明 Int 时 `automaticValue` 可保持 true）。
- [ ] 每个子菜单 ≤ 8 个控件（没有意外出现 `More` 翻页）。
- [ ] ObjectToggle 的 `referencePath` 是**完整的头像相对路径**（例如 `Clothes/Default/Cardigan`）且 `targetObject` 已设、解析结果非 null。
- [ ] 目标节点是 `Untagged`、**不是 `EditorOnly`**（EditorOnly 会被构建剥掉）。
- [ ] 商业模块的动画绑定全部可解析（0 条断链）。
- [ ] SMR 的 `bones` 指向主 Armature（不是旧的 prefab 骨架）。
- [ ] 参数名/类型/default/saved/synced 都是有意为之；没有重复/冲突声明。
- [ ] Console 相对基线没有新增报错/告警。
- [ ] 实际跑过的证据层已标注；未跑的层标 `NOT_RUN`/`BUILD_REQUIRED`；场景已保存。

> ⚠️ **本节（路线 C 反向绑定）已被 §16 取代**：2026-10-24 玩家实测中「其它套装一键穿衣完全无效」——原因是把菜单项自己的祖先（容器）当目标，MA 不生成关闭动画。正确做法见 §16。

## §15 「一键穿衣」互斥：用 ObjectToggle 反向绑定（本工作区实测 2026-10-24）

**症状**：想给每套衣服加一个「一键穿衣」总开关（一次穿整套 + 与其他套装互斥），纯 MA 做法有两条路都失败：
- 路线 A「每个总开关只 toggle 自己这套的容器」→ 烘焙后容器层只有 `m_IsActive=1` 的 clip，**没有 0** → 选中新套装旧套装不消失（多套叠加）。
- 路线 B「总开关的 ObjectToggle 列出该套全部部件对象」→ 部件层同样只有 1 的 clip。

**根因**：MA 的 Reactive Object 只为「参数=打开」一侧生成动画，关闭一侧由**场景初始值**决定；一个开关要同时关掉别人，必须让别人的对象也出现在**自己的** ObjectToggle 列表里（列表里 `Active=false` 就是「我打开时把目标关掉」）。只 toggle 自己，MA 没有任何理由去生成「别人变 0」的曲线。

**正确做法（路线 C，纯 MA，实测烘焙数据正确）**：每个「一键穿衣」的 `ModularAvatarObjectToggle` 列出**全部**套装容器（N 套 → N 个目标）：**自己这行 `Active=true`、其余 N-1 行 `Active=false`**。
共用一个 Int 参数（如 `OutfitSelect`，值 1..N，默认套装 `isDefault=true`），场景初始状态设成「默认套装容器 active、其余 inactive」。烘焙结果（实测 N=8）：
每个容器的 `MA Responsive: <容器>` 层有 N 个动作 state，**恰好 1 个 clip 写 1（自己选中）、N-1 个写 0（别人选中）** → 互斥成立。

**坑**：
- 目标容器**不能是菜单项自己的祖先**（菜单项放在套装子菜单里，祖先就是容器）：MA 对这种「toggle 自己祖先」的条目按常开处理，不会生成 0 的曲线。本设计的「自己=ON」条目虽落在祖先上但只需要 1，因此可用；「别人=OFF」条目都在祖先之外，正常生成 0。
- 同名子物体（如 `浴衣/浴衣` 下有两个同名 `浴衣`）会让 `Transform.Find` 拿错空壳 → 用「带 SubMenu MenuItem 且直接子 MenuItem 最多」的候选定位真节点。
- 验证：烘焙后在 VRCFury FX controller 里逐容器数 clip 的 `m_IsActive` 0/1（应为 1 个 1 + N-1 个 0）；
  Gesture Manager 的 Play 模拟**可能看不到**容器层的效果（`__MA/ActiveSelfProxy/*` 代理参数在编辑器里没有按 VRChat 的 Params 默认值初始化），最终以 **Build & Test / 上传后游戏内**为准，
  编辑器侧标 `NOT_RUN`。
- 该方案的部件开关保持作者原样（部件参数默认 0/1、`Active` 语义不改），不要为了让「图标默认 ON」去翻 `Active` —— 翻了会破坏作者「默认套装的部件默认可见」设计。

## §16 「一键穿衣」最终正确做法（v4：互斥 + 部件默认打开 + 可单关，GM 实测通过）

**目标**：每套衣服子菜单加「一键穿衣」，点一次穿上整套；与其他套装互斥；点完后该子菜单全部部件显示为「打开」状态（图标 ON = 外部可见）；之后还能单独关掉某一项。

**坑（v3 失败原因）**：把「套装容器」当 ObjectToggle 目标、且菜单项/组件就在容器内部（目标是自己的祖先）→ MA 响应层不生成 `m_IsActive=0` 的关闭动画 → 其它套装永远开着，一键穿衣「完全无效」。

**v4 正确结构（纯 MA，无 Animator、无 Parameter Driver）**：
1. 每套子菜单第一项 = `ModularAvatarMenuItem`（Toggle，`automaticValue=false`），参数 = **共享 Int**（如 `OutfitSelect`，1..N，默认套装设 `isDefault=true`，用 `ModularAvatarParameters` 声明 Int / default 1 / synced+saved）。
2. 同一个 GO 上加 `ModularAvatarObjectToggle`，列表**只列其它 N-1 套的容器对象**，每行 `Active=false`。选中本套 → 其它套容器被关；本套容器不在列表（常开），由部件控制显隐。
3. 部件 = `MenuItem` Toggle + 显式 bool 参数；同一 GO 的 `ObjectToggle` 目标 = 该部件对象、`Active=true`（ON=显示）；场景里部件对象初始 `SetActive(false)`；
  `ModularAvatarParameters` 里把这 47 个 bool 的 `defaultValue` 设为 1（`hasExplicitDefaultValue=true`）→ 图标默认 ON，点关能隐藏。
4. 默认套装的部件若原来是 `Active=false`（作者反相写法），翻成 `Active=true` + 默认 1，保持「打开=可见」。
5. 部件对象若与菜单节点同名/是菜单祖先（如浴衣的「衣服」指向浴衣节点），从一键穿衣列表里排除该目标，避免再次形成祖先环。

**验证（本工作区实测 2026-10-24，Kipfel 8 套 47 部件）**：
- 烘焙后 FX 控制器每个非默认容器 `MA Responsive: <容器>` 层有 **7 个 value=0 clip**（正确互斥）；默认容器 0 个。
- GM Play 反射 `GestureManager.Module`（`ModuleVrc3`）→ `Params` 字典 `Set(module, v, null)` 改 `OutfitSelect`：1→睡衣、2→睡衣2、5→休闲装3、8→校服，等 2 秒后场景对象 `activeSelf` 均正确（本套 ON、其它 OFF）。
- `C3_Coat=false` → 外套对象隐藏、其余 6 件仍显示（部件默认 ON 且可单关）。
- 骨骼检查：8 套 54 个 SMR `sharedMesh`/`bones`/`rootBone` 全非空；11 个 MergeArmature；78 个 MA MenuItem（67 Toggle + 9 SubMenu，0 空 Control/参数）；116 条 MA Parameters 0 重名。
- 截图：Play 里 `ScreenCapture.CaptureScreenshot` 只拍 Game 视图；GM 径向菜单是独立 EditorWindow（`Vrc3FloatingMenu`），
  `EditorWindow.CreateInstance` + `ShowUtility()` 后用 PrintWindow 抓；菜单结构递归烘焙后的 `VRCExpressionsMenu`（name/type/parameter/value）导出。

**边界**：只在 Gesture Manager 模拟验证；上传后游戏内多人同步/远端可见性未验证（NOT_RUN）。

## §17 「一键穿衣」必须保证「至少一套衣服」：Toggle 改 Button（本工作区实测 2026-10-24，第 68 轮）

**症状**：一键穿衣做成一键切换后，用户再点一次当前套装会变成「没穿衣服 / 只剩素体」，或出现多套叠加。

**根因（VRChat 菜单语义）**：`Toggle` + `Int` 参数在「关」时会把参数写成 **0**（不是上一套、也不是默认套装的编号）。而「互斥」是用每套一键穿衣的 `ModularAvatarObjectToggle` 反向列出**其它 7 套容器**（`Active=false`）
实现的，条件只覆盖 1..8 的编号；参数 = 0 时没有任何条件命中 → 容器/部件停留在默认状态，出现空装或多套叠加。

**正确做法（纯 MA）**：
1. 8 个「一键穿衣」菜单项 `VRCExpressionsMenu.Control.ControlType` 用 **`Button`**，`parameter` = 共享 Int（如 `OutfitSelect`），`value` = 1..8；Button 无法「关」，参数永远不会被写成 0。
2. `OutfitSelect` 用 `ModularAvatarParameters` 显式声明：`syncType=Int`、`defaultValue=1`（默认那套，如睡衣）、`hasExplicitDefaultValue=true`、`saved=true`。
3. 菜单项 `automaticValue=false`、默认那套 `isDefault=true`（`isDefault` 只决定参数**默认值**，不能阻止 0，见 `ModularAvatarMenuItem.cs` L298-304）。
4. 场景里默认套装的容器 `activeSelf=true`、其余容器 `false`；这样即使参数意外为 0，也回落到默认套装而不是裸体。
5. 部件开关仍保留各自 bool、`defaultValue=1`（图标默认 ON = 外部可见），手动关单项照旧。

**判据**：GM 里置 `OutfitSelect=1/3/7` → 对应容器 ON、其余容器 off、该套部件全部 ON；手动关某部件 → 仅该部件隐藏；烘焙 Console 0 error。**注意**：AAO 的 MergeBone 可能把某些容器（如 `夏装`）
在运行期重命名为 `夏装$子物体$N` 并上提，MA/VRCFury 的动画路径会由 NDMF 自动重映射，属正常现象、不要按名字写死路径。

## §18 编辑模式「默认状态」空装：用 EditorOnly 预览副本让场景里也穿着默认套装（本工作区实测 2026-10-24，第 69 轮）

**症状**：菜单/换装做好了，进 Play 或游戏里默认穿着正常，但 **Unity 编辑模式（没有动画驱动）里头像光着**；用户会认为「默认状态没穿衣服」。

**根因**：MA 官方 Object Toggle 文档写明 —— *所有控制同一对象的 Object Toggle 都不激活时，对象使用**场景初始状态**（original state）*。而「一键穿衣」要求「部件的 ON = 外部可见」，
于是部件必须在场景里 `SetActive(false)`（这样 OFF 才会隐藏）；编辑模式没有动画去把它打开 → 空装。这与「场景里初始可见」是同一枚硬币的两面，靠调 Active 值解决不了。

**正确做法（EditorOnly 预览副本）**：
1. 在头像根下新建容器（如 `默认外观预览（EditorOnly）`），`tag = EditorOnly`、`active = true`。
2. 把默认套装（这里是睡衣）的可见部件对象 **复制** 进去（`Object.Instantiate` 保留世界变换；副本里删掉 Animator 等运行期组件；材质/网格/骨骼直接引用原对象，不增加资源体积）。
3. 编辑模式立刻显示穿好的默认外观；**NDMF 的 `RemoveEditorOnlyPass`**（`Packages/nadena.dev.ndmf/Editor/InternalPasses/RemoveEditorOnlyPass.cs`：
  `foreach Transform t in AvatarRoot.GetComponentsInChildren<Transform>(true) if (t.CompareTag("EditorOnly")) DestroyImmediate(t.gameObject)`，
  在 Resolving 阶段最先执行）会在 Play/bake 时删掉它；VRCSDK 构建同样剔除 EditorOnly。
4. 好处：运行时/上传完全不受影响（零副作用），也不需要为「预览」额外做一套动画或反向参数。

**判据**：① 编辑模式渲染截图 = 默认套装外观；② 进 Play 烘焙后克隆体内 `GetComponentsInChildren<Transform>(true)` 里查不到预览对象，SMR 总数与加预览前一致；③ GM Play 全菜单回归与基线逐项一致；④ 上传后游戏内 NOT_RUN。

## §19 用脚本 / 反射批量改 MA 组件（字段级实测，2026-10-24）

**程序集与类型**：
- `Type.GetType("nadena.dev.modular_avatar.core.ModularAvatarMenuItem, nadena.dev.modular-avatar.core")` —— 程序集名是 **`nadena.dev.modular-avatar.core`**（写成 `nadena.dev.modular-avatar` 一律 null）。

**字段与写法（实测容易踩的）**：
- `ModularAvatarObjectToggle.Objects` 是 `List<ToggledObject>`；`ToggledObject` 是 **struct**（字段：`AvatarObjectReference Object`、`bool Active`）→ 修改要「取元素 → 改副本 → 写回 list[i] → 写回整个 list（属性 setter）」。
- `AvatarObjectReference.referencePath` 是 public；`targetObject` 是 internal（反射需 `BindingFlags.Public|NonPublic|Instance`）；改完可调 `ResolveReferences()`。
- `ModularAvatarMenuItem.Control` 的类型是 `VRCExpressionsMenu.Control`，且在该 SDK 里是 **class 不是 struct** → **没有 `.Value`**；正确写法：
  `mi.Control = new VRCExpressionsMenu.Control { name = "...", type = VRCExpressionsMenu.Control.ControlType.Toggle, parameter = new VRCExpressionsMenu.Control.Parameter { name = "OutfitSelect" }, value = 2f, icon = tex }`；读图标用 `mi.Control.ic
  on`。
- `ModularAvatarMenuItem.label` 通常为空 —— 菜单显示名来自 **GameObject 名字**（按 label 过滤会得到 0 个，按 `go.name` 找）。
- `ModularAvatarParameters.parameters` 是 `List<ParameterConfig>`；字段：`nameOrPrefix`、`remapTo`、`internalParameter`、`isPrefix`、
  `syncType`（`ParameterSyncType.NotSynced/Int/Float/Bool`）、`localOnly`、`defaultValue`、`saved`、`hasExplicitDefaultValue`。
- `ModularAvatarMenuItem.isDefault` 只决定参数**默认值**，不能阻止 Toggle 被关到 0（`ModularAvatarMenuItem.cs` L298-304）；要「至少一项」用 **Button**（见 §17）。

**在 `execute_code`（CodeDom C# 6）里写这类脚本的约束**：
- **不能写 `using` 指令**（报 `Line 1: Unexpected symbol ...`）→ 用全限定名（如 `UnityEngine.GameObject`、`UnityEditor.SceneManagement.EditorSceneManager`）；`Object` 歧义写 `UnityEngine.Object`。
- `GameObject.GetComponentsInChildren<GameObject>()` 会报「必须继承 MonoBehaviour 或 Component」→ 改 `GetComponentsInChildren<Transform>(true)` 再取 `.gameObject`。
- 改完 `EditorSceneManager.SaveScene(scene)` + `AssetDatabase.SaveAssets()`；脚本放 `Assets/Editor/` 下，跑完把脚本删掉并 `refresh_unity`（编译卡死的处理见 01-06 §12）。


## §20 一键穿衣的最终实现：共享 Int + FX 驱动层（本工作区实测 2026-10-24，第 71 轮）

### 症状（一键穿衣）
- 一键穿衣用 Toggle(Int) 时，再点一次会把参数写成 **0**，所有容器/部件都没有对应状态 → 空装或多套叠加；用户原话「规定至少有一套一键穿衣的衣服」「默认不能空衣服」。
- 尝试用 **MA ObjectToggle 直接控制容器**（纯 MA）时：容器作为菜单项祖先，MA 只生成 `m_IsActive=1` 的 clip，**没有 0**（Reactive Object 认为目标恒开），烘焙后多套衣服同时可见。
- 改用 **MA MergeAnimator 挂自建 FX controller** 时：烘焙日志无报错，但 NDMF 生成的虚拟副本 `...\_assets\assets N.asset` **layers=0**，最终 VRCFury FX 里没有这一层 → 参数写了不起作用（实测 `OutfitSelect=3` 后 `OutfitApplied` 仍为 0）。

### 正解（本工作区已验收）
1. 8 个「一键穿衣」= MA MenuItem **Toggle**，共享 Int `OutfitSelect`（1..8，默认套装 `isDefault=true`）。
2. 47 个部件 = MenuItem Toggle + 显式 Bool 参数（`S1_*`/`S2_*`/`C1_*`/`C2_*`/`C3_*`/`Y1_*`/`SS_*`/`XF_*`）；默认套装（睡衣）的参数在 MA Parameters 里 `defaultValue=1`，其余 0；部件 OT `Active=true`、场景初始 inactive。
3. 驱动层：FX 里一个 `OutfitOneClick` 层，**AnyState → Apply_k**，条件 `OutfitSelect Equals k` 且 `OutfitApplied NotEqual k`；
  每个 Apply 状态的 `VRCAvatarParameterDriver` 一次写入 47 个部件的 Set（本套 1、其余 0）+ `OutfitApplied = k`；另有 `Apply_0_DefaultS1` 处理 `OutfitSelect=0` → 回落默认套装（满足「至少一套」）
  。回到 Idle 无条件、时长 0。
4. **注入方式**：不要依赖 MA MergeAnimator——直接 `AssetDatabase.CopyAsset` 复制头像 descriptor 的 FX controller 成
   `Assets/08_动画·菜单·图标/一键穿衣系统/Kipfel_FXLayer_v1.1.1_沐梓溪喵.controller`，在副本上 `AddLayer` + 补 `AnimatorController` 参数（Int `OutfitSelect` default 1、
Int `OutfitApplied` 0、47 bool 默认值），再把 `descriptor.baseAnimationLayers[FX].animatorController` 指向副本；同一 controller 被别的头像共用时**必须复制**，不能改原控制器。

### 判据（一键穿衣）
- 编辑态：`descriptor FX = ..._<头像名>.controller`、该 controller 的 layers 比原控制器多 1、参数含 `OutfitSelect/OutfitApplied`。
- Play：GM 参数 `OutfitSelect=3` 后 2–3 秒 `OutfitApplied=3`，本套部件全 ON、其它套可见网格全 OFF（其它套的 PhysBone/Collider 辅助物件仍在 active 属正常）。
- `OutfitSelect=0` → `OutfitApplied=1`、默认套装 ON、其余 OFF。
- 手动把单个部件 bool 置 false：只隐藏该部件，其余不变；不会被驱动层反复重置（`OutfitApplied` 门控）。
- 收尾：退出 Play 后删 `Packages/com.vrcfury.temp/Builds`，从压缩包还原被 Play 重写的 5 个 `NyaPPu__Generated/LightController/*.anim`（本工作区 MD5 见证据目录）。

### 坑
- 编辑器里 `Animator.runtimeAnimatorController` 在 Play 下为 null，`VRCAvatarDescriptor.baseAnimationLayers[i].animatorController` 是运行时 controller，不能 `.layers`；
  要看烘焙层请直接读 `Packages/com.vrcfury.temp/Builds/____/VRCFury FX.controller`。
- AAO `MergeBone` 会把容器合并、并把网格改名成 `容器$网格$序号`（如 `夏装$IceCreamSummer_bag$146`），烘焙 controller 里的路径仍是原路径，NDMF 运行期重映射——**不要按运行时名字写死**。
- CodeDom 版 `execute_code` 不支持 local function、不支持顶层 `using`。

## §21 两个发型「二选一且必须有一个」：单 Bool + 双目标 ObjectToggle（本工作区实测 2026-10-24，第 72 轮）

### 症状（发型二选一）
- 两个发型开关都关掉 → **光头**；只打开另一个（头发1）→ 仍光头。

### 结构 / 根因
- `头发` 挂头像根：`ModularAvatarMeshSettings` + 子 `_Armature` 走 `MergeArmature` 合进主 armature；`头发2` 走 `ModularAvatarBoneProxy` 挂到 `Head` 下。
- 作者原设计 = 两个**自动** bool 参数 + 各自 `ObjectToggle` **只列自己的目标**、`Active=false`（ON=隐藏），场景初始两个头发根都可见 → 参数都 0 时谁都不动（两套一起显形），单独关一个、开另一个时又只是「隐藏自己」不是「显示对方」→ 光头。
- 动画曲线方案必然失效：AAO `MergeBone` 会改对象名（`头发2$front$23`）/把网格并进 `$$AAO_AUTO_MERGE_SKINNED_MESH_0`，按路径绑定的 `m_IsActive` 曲线对不上；`SetObjectReferenceCurve`（PPtr）在 MA 烘焙后的虚拟层级里也取不到。

### 正解（纯 MA，已实测）
1. 只声明 **一个 Bool 参数**（如 `hair2`，`ModularAvatarParameters`：default=1、saved、synced）。
2. 两个菜单项都是 Toggle 绑同一参数：`头发1` value=**0**、`头发2` value=**1** → 参数只有 0/1 两态，天然「必须有一个」。
3. 每个菜单项同 GO 的 `ModularAvatarObjectToggle` **同时列两个头发根**、`Active` 相反：
   - `头发1`：`头发`=**true**、`头发2`=**false**
   - `头发2`：`头发`=**false**、`头发2`=**true**
4. 场景初始把两个头发根都设 **inactive**：MA 的 OFF 侧回落场景初始值 → 参数任一态时**恰好一个**被打开。
5. 编辑模式要有头发：在 `EditorOnly` 预览副本里再复制默认发型（头发2）的 8 个 SMR（本工作区 preview 10→18 子）；NDMF `RemoveEditorOnlyPass` 在 bake/Play 前删除，运行时无影响。

### 判据（发型二选一）
- 编辑态：`头发`/`头发2` 均 `activeSelf=false`；预览容器含默认发型网格。
- Play：`hair2=1` → `头发2` active、`头发` inactive；`hair2=0` → 反之；两边都能渲染出头发（不光头）。
- 烘焙菜单：`发型` 子菜单 2 项 Toggle、同一参数、value 0/1；EP 参数 Bool default=1。
- 全菜单回归基线不变（本工作区 170 菜单 / 161 PASS / 6 个 ABT FAIL）。


## §22 头发「切换正常但完全不摆动」与头饰微放大（本工作区实测 2026-10-24，第 73 轮）

**症状**：两个发型菜单互斥切换正常，但其中一套头发（本例「头发1」）在游戏/Play 里几乎完全不动；用户会说「没绑上动骨」。

**根因（权重与 PhysBone 根骨错位）**：PhysBone 挂在发丝细分支骨（`Front2`、`Front_C`、`Ahoge`、`Side1..3`），而 SMR 网格顶点权重集中在**父骨**：`Side-Back_Root` 占 96%、`Front_Root` 占 57%（`Front_C` 链占 43%）。被模拟的骨几乎没有权重 → 看起来完全静止。

**定位三步**：
1. 对目标 SMR 读 `mesh.boneWeights`，按骨骼汇总权重排序（`boneNames[bw[i].boneIndex0..3]` 累加 `weight0..3`），看前几名骨；
2. 对照该头发下所有 `VRCPhysBone.rootTransform` / `bones`；
3. 若权重主导骨没有 PB → 就是本症。

**正解**：删掉挂在低权重细分支骨上的 PB，改在权重主导的父骨上各挂 1 个 PB（本例 `Front_Root`、`Side-Back_Root`），**`bones` 列表留空**——实测烘焙时 SDK 会自动按根骨展开：
`Front_Root` → 7 根（本身 + Front2 链 3 + Front_C 链 3）、`Side-Back_Root` → 13 根（本身 + Ahoge/Side1/Side2/Side3 各 3）；参数从温和值起步（pull 0.20、spring 0.35~0.40、
stiffness 0.10~0.15、gravity 0.10~0.20、immobile 0.10~0.25、角度限制 12~15 / 20~30）。

**头饰「略微穿模」微放大 1.05（不要直接改 SMR transform）**：
- SkinnedMeshRenderer 的 transform 缩放对蒙皮结果**无效**（骨骼矩阵会抵消渲染器变换）；
- 正确做法：`Object.Instantiate(sharedMesh)` 复制网格 → 按顶点质心缩放全部顶点 `v' = c + 1.05*(v - c)` → `RecalculateBounds()` → `AssetDatabase.CreateAsset` 存成独立 mesh（本例 
  `Assets/09_工具与插件/改模生成/头饰放大/<名>_x105.asset`）→ 赋回 SMR；**EditorOnly 预览副本用同一个新 mesh**，保持编辑态与运行时一致；
- 实测校验：顶点平均半径比 = 1.0500、包围盒 +4.0%~4.4%；四向渲染（前/右/后/左）确认无破面、帽子与头发不再穿插。
- 坑：`execute_code` 安全策略会拦 `AssetDatabase.DeleteAsset`；重复生成时改名而不是删除。换 mesh 时 `.meta` 不必动（GUID 不变）。

**判据**：烘焙后 PB `bones` 数 = 父骨 + 全部子链骨数；头饰 `sharedMesh.name` 带 `_x105`、bounds 约 +4%；四向图无穿模/无破面。

## §23 菜单「开启反而变隐藏」：把作者式反相 ObjectToggle 改成 ON=可见（本工作区实测 2026-10-24，第 74 轮）

**症状**：饰品菜单里 `名牌 / 小翅膀发饰 / 眼镜 / 耳朵和尾巴！` 四项，点了「开」反而看不到东西，用户报「菜单调反了」。

**根因**：作者用 `ObjectToggle.Active = false` 表达「开关 ON = 隐藏/脱下」（因为物件在场景里默认是开的）。MA reactive 的语义是：`Active=true` 表示「参数 ON → 把目标设为 active」，**OFF 侧回落到场景初始状态**。
所以只要「场景初始=可见」，ON 与 OFF 两态都等于可见，MA 根本不生成 OFF 动画；要想 ON=可见且 OFF=隐藏，场景初始必须是隐藏，默认外观改由「参数默认值 1 的 defaults」在运行时点亮。

**正确做法（本工作区实测）**
1. 目标渲染对象在**场景里 `SetActive(false)`**（隐藏）；
2. `ModularAvatarMenuItem` 改成显式 bool 参数：`automaticValue=false`、`Control.type=Toggle`、`Control.parameter.name=<自定义>`、`Control.value=1`、`isDefault=true`；
3. 在该头像的 `ModularAvatarParameters` 里声明同名 Bool：`defaultValue=1`、`hasExplicitDefaultValue=true`、`saved=true`、同步；
4. `ModularAvatarObjectToggle` 的每一行 `Active=true`，且 `AvatarObjectReference` **必须指向真实目标**（`new AvatarObjectReference(realGameObject)`）；
5. 编辑态外观用 EditorOnly 预览副本（把目标 renderer 复制到 `默认外观预览（EditorOnly）` 下，NDMF `RemoveEditorOnlyPass` 烘焙/上传时删掉）。

**判据**
- 烘焙后的 FX 里出现 `MA Responsive: <目标名>` 层（每个目标一条），文件在 `Packages/nadena.dev.ndmf/__Generated/.../_assets/assets NN.asset`；
- 运行时默认：目标 `activeSelf=True`（defaults 层点亮）；
- GM 反射把参数置 false → 目标 `self=False / hier=False`；置 true → 复原；同一菜单其它项（例如睡衣部件）不受影响；
- 同步位数不超 256（本例 4 个 bool 后 params=153、sync bits=223）。

**坑（实测踩到）**
- 第 4 步若误指向预览副本：副本是 EditorOnly，烘焙时被 NDMF 删掉 → MA 生成不到 reactive 层 → 真实目标既 inactive 又无动画引用，**AAO 会把它整棵从烘焙 clone 里删掉**（现象：运行时 `GetComponentsInChildren<Transform>` 完全找不到这些物体）。
- 预览副本用 `Object.Instantiate(src, previewContainer, true)` 保世界变换、改名 `预览_<名>`；`GameObject.Find` 找不到 inactive 根，用 `EditorSceneManager.GetActiveScene().GetRootGameObjects()`。
- 编辑态改完记得 `EditorSceneManager.SaveScene`，并另存一份场景备份到 `_asset_reorg/backup/`。

**判据一页纸**：4 项 → 13 个目标行、4 个 bool（Acc_Nameplate / Acc_Wing / Acc_Glasses / Acc_EarTail），场景备份 `Lilleo2.0.unity.bak-2026-10-24-before-accessory-toggle-fix`。


## §24 亮度插件 LightController（nyappu）：别手跑 Generator；默认亮度要改组件 + EP 默认值（本工作区实测 2026-10-25，第 77 轮）

**症状**：在场景头像上直接调用 `nyappu.lightcontroller.Generator.Generate(desc)`「刷新」亮度动画之后，
① 头像上的 `LightController` 组件消失（预制件实例的组件被记进 `m_RemovedComponents`）；
② `Assets/08_动画·菜单·图标/NyaPPu__Generated/LightController/` 下 5 个 `.anim` 被删空（只剩文件夹与 meta）；
③ 再调一次（组件已经不在）会把 Generated 目录彻底清空；烘焙时没有 `LightController/LightLimit` 参数、菜单里「亮度控制」失效。

**根因**：`Generator.Generate()` 的设计运行上下文是 **NDMF 烘焙时对克隆头像**（`nyappu.lightcontroller.Editor.NdmfPlugin` 在 `BuildPhase.Transforming` 调用），
它内部先 `RemoveLightControllers()` / 清空生成目录，再按当前头像结构重建。**在编辑态场景头像上手跑 = 破坏源工程。**

**正确做法**
1. 默认亮度：改场景里 `LightController` 组件的 `defaultLightLimit`（本例 0.55 → **0.6**）；不要用 Generator「重新生成」。
2. 生成物保持原样：5 个 `.anim`（BacklightBorder / BacklightToggle_Off / BacklightToggle_On / LightLimit / Monochrome）+ `Map_Texture.renderTexture`；
   若被 Play 重写或误删，从原始压缩包同路径还原（本例 `NyaPPu__Generated/LightController/` 在 `Assets` 根，重组后落在 `08_动画·菜单·图标/` 下）。
3. 烘焙后确认：EP 里 `LightController/LightLimit`（Float）`defaultValue = 组件值`（本例 0.6）。
4. **注意 `saved`**：生成器给 `LightLimit` 硬编码 `saved: true`（`Generator.cs` 的 `new ParameterConfig { saved = true }`），
   所以「用户曾经存过亮度」时，本地 saved 值会盖掉默认值；要让每次加载都套 60%，只能在烘焙后把该参数 `saved` 改 false（或让用户点一次「恢复默认」）。

**判据**：编辑器里 `cat.GetComponentsInChildren<Component>(true)` 能找到 `nyappu.lightcontroller.LightController` 且 `defaultLightLimit=0.6`；
`Assets/08_动画·菜单·图标/NyaPPu__Generated/LightController/` 5 个 anim 都在；烘焙 EP 该项 0.6；Console 0 error。

## §25 参数持久化 / 关闭触发表情 / 头饰缩 2% / 眼罩约束三源对齐（本工作区实测 2026-10-25，第 78 轮）

### 25.1 「每次重进世界装扮都被重置」= 参数 saved=false（不是 bug，是上一轮的修法副作用）

> ⚠️ **本节结论部分已被第 132 轮修正**：「所有部件 bool 都 saved=true」的做法**不要照抄**。现行做法是**只对 3 个入口参数**（`OutfitOneClick` Int def1、`Acc_Glasses` Bool def1、
> `Acc_NamePlate` Bool def1）置 saved=true，派生部件 bool 保持 saved=false，可见性由 `OutfitSync` 每态全量重写。
> 判据与验收见 [03-06 §14.3/§14.7](../03-菜单与参数/03-06-ma-menu-build-by-reflection.md)、`07-验证与排查/07-01-bake-and-validation.md` §34。
- 症状：VRChat 里调好的衣服/配饰，重新进世界又回到默认；用户报「无法保存装扮」。
- 根因：`ModularAvatarParameters` 的 `ParameterConfig.saved=false`（或 EP 里 `saved=False`）= 该参数**不上传保存**，每次加载都套默认值。上一轮为解决「进游戏光着（旧存档值 0 被套用）」把 48+ 个装扮参数全改成 saved=false，于是每次都重置。
- 正确做法（同时满足「默认有衣」+「能保存」）：
  1. ~~`saved=true`（OutfitSelect + 47 个部件 Bool 全部 true，见 `一键穿衣设置` 的 MA Parameters）~~ ⚠️ **已失效（第 132 轮推翻）**：
  只有入口参数该 saved=true（`OutfitOneClick`/`Acc_Glasses`/`Acc_NamePlate`），**派生部件 Bool 必须 saved=false**，否则 OutfitSync 全量重写会造成「半裸」；
  现行见 [03-06 §14.3/§14.7](../03-菜单与参数/03-06-ma-menu-build-by-reflection.md)。
  2. EP/参数 `defaultValue` 给对（`OutfitSelect`=1、睡衣 6 件 Bool=1、其余=0）；
  3. **兜底归一化 state**：FX 驱动层加一条 AnyState→`Apply_0_DefaultS1`，条件 `OutfitSelect Equals 0` 且 `OutfitApplied Equals 0`（OutfitApplied 为 local 参数，加载恒 0）；
  该 state 的 `VRC Avatar Parameter Driver` 把默认套装（睡衣 6 件）置 1、`OutfitApplied=1`、`OutfitSelect=1`。这样即使服务端残留旧的 0，加载第一帧也会自动穿好并写回 1。
- 判据：烘焙后 `ExpressionParameters` 里目标参数 `saved=True`、default 正确；进游戏重进世界仍保留上次选择（编辑器只能验 saved 位，真实持久化 NOT_RUN 需上传实测）。
- 陷阱：`ParameterConfig` 是 **struct**，反射批量改必须 `object boxed=list[i]; field.SetValue(boxed,val); list[i]=boxed;`（直接改 boxed 副本不生效）。
- 相关：`LightController/LightLimit` 同理——`Generator.cs` 里 `saved=true` 才能保存亮度；把默认亮度改 `defaultLightLimit`（本工程 0.6）+ `saved=true`。

### 25.2 关闭「摸头触发表情」但不删除功能
- 链路：`VRCContactReceiver`（param `Pet`）→ FX `Contact Pet Control` 层把 `Pet` 转成 `PetMode` → `[ USER EDIT ] Face` 的 AnyState→`Pet_Happy`/`Pet_Unhappy`（条件 `PetMode Equals 1/2` 且 `Contact If 0`）。
- 关闭方式：只把接收器 `VRCContactReceiver.enabled=false`（不删组件、不动动画层）；烘焙/上传时 disabled 组件会被剥离，克隆里 `Contact_Pet` 上不再有接收器 → 功能消失但随时可改回 `enabled=true` 恢复。
- 判据：编辑场景 `CONTACTS=8 OFF=1`；烘焙克隆里 `Contact_Pet` 无 `VRCContactReceiver`。

### 25.3 头饰微缩 2%：复制 mesh 顶点缩放，替换 sharedMesh
- 帽子偏大 2%：`SkinnedMeshRenderer.sharedMesh` 指向复制网格（`PJ_Hat_x102` / `PJ_HatRibbon_x102`），顶点按质心 `v'=c+1.02*(v-c)` 缩放；预览副本一并替换；原 mesh 文件保留可回退。
- 注意：不要改 SMR transform scale（蒙皮会把缩放抵消），也不要改骨骼缩放（会影响整个头部/头发）。

### 25.4 眼罩（ParentConstraint 配饰）三源对齐
- 结构：`[設定用]アイマスク` 挂 Unity `ParentConstraint`，3 源 = `Target1`（过渡）/`Target2`（戴上最低位）/`[初期]位置調整用オブジェクト`（待机）；动画层只改 3 个 source weight（0→1 过渡）。
- 约束 offset 是作者序列化值、bake 不重算：`maskRot = targetRot * rotOffset`、`maskPos = targetPos + targetRot * transOffset`。要让 mask 到目标位姿，对每个源反解：
  `q_src = q_target * Quaternion.Inverse(rotOffset)`，`p_src = p_target - (q_src * transOffset)`（`transOffset` 为 0 时 `p_src = p_target`）。
- 本次实测：`rotOffset ≈ Euler(-28.83,-0.679,-0.668)`，目标 `pos(0,0.678,0.040) rot Euler(90,0,0)` → 三源统一为 `pos(0,0.678,0.040) rot Euler(61.16,177.92,178.77)`（源
  2 `[初期]` 因 offset 不同仍为 `(90,0,0)`）；三源代入后 `dPos=0 / dAngle=0`。
- 判据：三源分别代入约束公式得到同一目标位姿；烘焙后 mask 实际 `pos/rot` 与目标一致（Play 里用 GM 反射激活后读 transform）。

## §26 Rest Pose 安全：动画被屏蔽时也必须「有衣服」（本工作区实测 2026-10-25）

**症状**：自己看正常，但别人（VRChat 安全设置里屏蔽了 Avatar Animations）看到的是裸模 / T-pose。

**根因**：屏蔽动画后客户端只渲染**序列化静态状态**；而 MA `ModularAvatarObjectToggle` 的 OFF 侧是「回落场景初始状态」。若部件在场景里 inactive、参数默认 0，那么静态状态=隐藏 → 陌生人看到裸体。靠「动画打开衣服」在设计上就不满足 Rest Pose 安全。

**正确做法（实测）**
1. 默认应显示的物体：**场景里 active**（不要靠默认 off + 动画开）。
2. 每个默认可见部件放**两个 ObjectToggle**：
   - 反向隐藏：`Inverted=true`（组件级）+ 行 `Active=false`（参数=1 → 隐藏）；
   - 正向显示：`Inverted=false` + 行 `Active=true`（参数=1 → 显示）。
3. 部件 MenuItem 设 **`isDefault=true`**：MA 的 `ControlCondition.InitialValue = mami.isDefault ? mami.Control.value : -999`；为 -999 时反向规则会被当作「初始生效」，
  烘焙静态状态就变成隐藏（本工作区第一次 bake 实测 9/9 睡衣部件隐藏）。若不想动 MenuItem，也可用显式参数 + MA Parameters 默认值 1 达到同样效果（但 InitValue 仍是 -999，需实测）。
4. **判据**：烘焙后 FX 里 `MA Responsive: <对象>` 层应有 0 与 1 两条 clip；克隆里对象初始 `activeSelf=true`（本工作区实测：`睡衣$PJ_*` 9/9、饰品 4 组、头发2 渲染器均可见；FX 122 层）。

**边界**：Rest Pose 只保证「动画被屏蔽时外观正确」；参数同步本身被屏蔽属客户端行为，模型侧无法绕过。


## §27 换头（双头切换）落地：骨骼映射的隐藏坑（本工作区实测 2026-10-25，LuzzyMeow V2）

### 症状（换头双头切换）
把目标头的 SMR 复制到宿��头像、按「同一相对路径」重指 `bones[]` 后，烘焙/游戏里目标头**看起来没有眼睛**（或眼球错位、牙齿/小配件消失），其它部分正常。

### 根因（换头双头切换）
目标头 mesh 可能用到**宿主没有的附属骨**（眼睛骨 `Eye.L/Eye.R`、牙齿骨等）。这些骨在目标源头的 rest（局部位置/旋转/缩放）与宿主同名/近名骨不同；只按名字/路径映射会把顶点绑到错误的 bone 空间 → 眼球等小体积部件被塞进头里。

LuzzyMeow V2 实测：狼头 `Body` mesh 的 125 根 bones 里只有 3 根被权重实际使用（`Head` + `Eye.L` + `Eye.R`，各约 548 顶点）；`Eye.L` 在狼是 `(-0.0358, 0.0461, 0.0154)`、在猫是 `(-0.0299, 0.0229, 0.0194)` → 直接复用猫眼骨 = 眼球错位到脸内。

### 正确做法（换头双头切换）
1. 对目标头的「非标准/附属骨」，在宿主对应父骨下**克隆空 GameObject**，把目标源头的 `localPosition/localRotation/localScale` 原样抄过来（例：`armature/.../Head/狼眼_EyeL`）。
2. 把这些克隆骨写进复制后 SMR 的 `bones[]` 对应索引（用名字匹配源 `bones[]` 找 index）。
3. 烘焙阶段 AAO 会把「默认隐藏（inactive）」的网格塌缩成单骨（本例 `$$AAO_AUTO_MERGE_SKINNED_MESH_1 bones=1 bone[0]=Head`）——这**不影响外观**（整头刚性跟随 Head），但会吃掉眼动/耳骨物理，属预期取舍，别误判为 bug。
4. 判据（无需看图）：把眼子网格材质临时换成 `Unlit/Color` 洋红，正面渲染 700²，统计洋红像素与左右对称性。本工程修复前=0，修复后=7,859 px（编辑态）/6,852 px（烘焙后），bbox 关于画面中线对称（left 3876 / right 3983）→ 眼睛渲染正常。

### 附带：换头模式下的「头饰/帽子」处理
- 目标头（狼头）自带发/耳，宿主所有「戴在头上」的配饰在换头模式必须隐藏，否则穿模：本工程用**同一个 `HeadType` Int 参数的 ObjectToggle**（换头菜单项）把 12+6 个对象行设 `Active=false`（睡衣帽/帽结、Eyemask、おめん、リボン、visor、
  眼镜 Chain/Glasses_Drop、头发1/2、耳朵尾巴、白名牌、WingA 等），烘焙后每个目标生成 `MA Responsive: <对象名>` 层，实测 clip 值 0 正确。
- 若要让帽子「戴在目标头上」，需再按目标头尺寸单独做变体（缩放/偏移），不能直接复用宿主帽子；本工程当前策略 = 换头模式下隐藏，后续按用户需求逐个适配。

### 判据速记
见眼睛先量权重、再克隆附属骨 rest；改完看 `SMR.bones[]` 索引是否指向新克隆骨；验证用「洋红材质 + 像素计数 + 左右对称」，比肉眼截图可靠（`read_image` 在本机有时只回元数据）。


## §28 换头后的「狼头专用头饰变体」：静态网格副本 + AAO 合并坑（本工作区实测 2026-10-25，第 83 轮）

### 症状（狼头头饰变体）
- 换头后原头饰（睡衣帽 / 帽结 / 睡衣2 眼罩 / 浴衣面具 / 泳装遮阳帽 / 眼镜 / 名牌 / 小翅膀）在目标头上尺寸与位置都不对；直接「换头模式隐藏」不满足需求（用户要「真的戴在狼头上」）。
- 烘焙后狼头表情动画全部 `NO VALID ANIMATIONS`。

### 根因（狼头头饰变体）
1. 头饰是 SkinnedMeshRenderer（甚至挂在服装自己的 armature 上），**SMR 的 transform 对蒙皮无效**（蒙皮只吃 bones 世界矩阵）；直接改 transform 看起来「没反应」。
2. `Object.Instantiate(mesh)` 会连带复制 skin/bounds 数据，`RecalculateBounds()` 对带蒙皮数据的 mesh 不按顶点重算（实测 bounds 偏大 1.2×，但顶点 AABB 正确）。
3. AAO `TraceAndOptimize.mergeSkinnedMesh = true` 把狼头 4 个 SMR 合并成 1 个 → 合并网格 `blendShapeCount=0`，狼头 Face FX 曲线全部失效。

### 正确做法（狼头头饰变体）
- 变体 = **静态网格副本（MeshRenderer）**，几何固定在 Head 局部空间：
  1. 取 rest 世界顶点：`var bm = new Mesh(); smr.BakeMesh(bm); world = smr.localToWorldMatrix.MultiplyPoint3x4(bm.vertices[i])`；**不要手写 boneMats 加权**（实测与 BakeMesh 差 ~1cm，且偶发整体尺寸错 1.2×）。
  2. 世界映射：帽类 `T(x) = WolfHairC + 1.06*(x - CatHairC)`（CatHairC/WolfHairC 取头发剪影 AABB 中心）；脸部（眼镜）按瞳距比 `1.197` 与眼高差 `+0.0235` 做非均匀映射。
  3. `head.worldToLocalMatrix` 转 Head 局部，写入**新建干净 Mesh**（vertices/normals/colors/tangents/uv/uv2/uv3 + `subMeshCount` + `SetTriangles` + 
  `RecalculateBounds`），再建 `MeshFilter` + `MeshRenderer` 的 `狼头版_<名>`。
  4. 全部挂到 `狼头/狼头版头饰` 下，默认 inactive；门控 = 在该原部件菜单项的 **show ObjectToggle** 行追加 `Active=true`（部件参数 ON 才显示）。
- 狼头表情：`TraceAndOptimize.mergeSkinnedMesh = false`（反射写入组件并保存）。代价：全局不合并蒙皮（性能略降），换取 4 个狼头 SMR 独立、`Body.blendShapeCount=58`。
- 隐藏真实猫头饰：自建 FX 层 + **`Renderer.m_Enabled = 0`** 曲线（MA responsive 层排在自建层之后，会覆盖 `m_IsActive`，所以不能用 m_IsActive）。

### 判据（狼头头饰变体）
- 烘焙后狼头模式：真实猫头饰 `renderer.enabled=false`，`狼头版_*` 变体 `activeSelf=true` 且世界 AABB 与狼头剪影匹配（帽类 0.316×0.280×0.367 vs 狼头剪影 0.302×0.276×0.239）；猫头模式：变体 `activeSelf=false`、真实头饰 enabled=true。
- 狼头 Face FX 层在烘焙 FX 里 `validClips=2` 以上、曲线路径 `.../狼头/Body`。

## §29 菜单按「角色」重构：顶层 角色 → 小狼 / 小猫 的独立菜单副本（本工作区实测 2026-10-25，第 84 轮）

### 需求形状
- 顶层 = `角色`；点 `小狼` → 二级 = `启用小狼`（切换，与 `启用小猫` 互斥）+ `小狼菜单`（现有 衣柜 / 发型 / 小玩具 / 饰品 / 亮度控制 的**小狼独立副本**）；`小猫` 同理。

### 正确做法（菜单角色重构）
1. 新建 9 个 `VRCExpressionsMenu` 资产：`角色`/`角色_狼`/`角色_猫`/`小狼菜单`/`小猫菜单`/`狼_衣柜`/`狼_发型`/`狼_小玩具`/`狼_饰品`；描述符 `expressionsMenu` 指向 `角色.asset`；`启用小狼`/`启用小猫` 用同一个 Int 参数（本例 `HeadType`，1=猫 2=狼）→ 天然互斥。
2. 复制菜单骨架：对 20 个「目标分类菜单 = 衣柜/发型/小玩具/饰品」的 `ModularAvatarMenuInstaller` 宿主，
  做**组件级克隆**（新建 GO + `ComponentUtility.CopyComponent/PasteComponentAsNew` 复制 `ModularAvatarMenuItem` / `MenuGroup` / `MenuInstaller`），
  把克隆出的 installer `installTargetMenu` 改到 `狼_*` 副本。
3. **不要复制 `ModularAvatarObjectToggle` 或其它 MA 组件**：菜单项只需写参数；对象显隐由原 OT 驱动，否则同一对象被两套参数同时驱动。
4. 克隆出的 MenuItem **必须有显式参数名**：MA 的自动参数 `__MA/AutoParam/<GO名>` 只在「同级/子级存在 ReactiveComponent」时才分配；克隆里没有 OT → 不会分配 → 开关无效。先把自动参数项转显式（本例 `胖次~`→`Acc_Panty`、
  `乳贴~`→`Acc_Nipple`，并在 `一键穿衣设置` 的 MA Parameters 里声明 Bool def=0 saved=true）。

### 判据（烘焙后）
- 菜单树：`角色` 2 项 → 每角色 2 项（启用X + X菜单）→ 5 分类；`小狼菜单/衣柜` 8 套、每套首项 `一键穿衣`（`OutfitSelect` 1..8），与猫版一致。
- 参数：`HeadType` 仅 1 个 Int、def=1 saved；EP 同步位数仍 ≤256；`__MA/AutoParam/` 不再出现在新副本里。
- 结构独立、状态共享：两套菜单写同一批参数（当前衣柜是共享的）；将来要给小狼专属衣服时再分叉对象/参数。

## §30 NDMF 预览（Scene 视图）proxy 叠加导致的「两个头/模型重合」与持久关闭方法

**症状**：编辑态场景数据里明明只激活了一个头（用 execute_code 扫 `activeInHierarchy` 渲染器只剩 6 个），但 Scene 视图里看到猫头 + 狼头/两套模型重叠；材质也像被“另一个副本”覆盖。

**根因（本工作区实测 2026-10-25）**：NDMF 预览系统会把烘焙结果以 **proxy 渲染器** 叠加到当前场景，加载方式为 additive 打开 `Packages/nadena.dev.ndmf/Runtime/___NDMF Preview___.unity`（
`Editor/PreviewSystem/Rendering/NDMFPreviewSceneManager.cs` 里场景名写死为 `___NDMF Preview___`）。这些 proxy 全是 `Proxy renderer for xxx` 命名，数量可达 30+；只要预览会话在，
就会周期性重建。

**排查（一条 execute_code 即可）**：
```csharp
var scn = UnityEngine.SceneManagement.SceneManager.sceneCount;
for (int i=0;i<scn;i++){ var s=UnityEngine.SceneManagement.SceneManager.GetSceneAt(i);
  var rs=s.GetRootGameObjects();
  for(int r=0;r<rs.Length;r++){ var rr=rs[r].GetComponentsInChildren<UnityEngine.Renderer>(true);
    for(int k=0;k<rr.Length;k++) if(rr[k].name.StartsWith("Proxy renderer for")) Debug.Log(s.name+" / "+rr[k].name);}}
```
看到 `___NDMF Preview___` 场景 + proxy 即确认。

**临时关闭（仅本会话有效，域重载会复位）**：`nadena.dev.ndmf.preview.NDMFPreview.DisablePreviewDepth = 1`（public property，反射要调 **setter**）
+ 菜单 `Tools ▸ NDM Framework ▸ Debug Tools ▸ Force Reset Preview`。注意 `NDMFPreview.cs` 有 `EditorApplication.playModeStateChanged`：
**退出 Play 时会把 DisablePreviewDepth 复位为 0**。

**持久关闭（推荐，本工作区已采用）**：
1. 反射把 `nadena.dev.ndmf.preview.NDMFPreview.EnablePreviewsUI`（internal static property，setter 会写 prefs + `SetPreviewState()`）设为 `false`；
2. 取 `Resources.FindObjectsOfTypeAll(typeof(NDMFPreviewPrefs))[0]`，调 `Save(true)` 把 `[FilePath("nadena.dev.ndmf/NDMFPreviewPrefs.asset", ProjectFolder)]`
  的 `EnablePreview` 落盘为 0 —— 文件路径在**工程根目录** `nadena.dev.ndmf/NDMFPreviewPrefs.asset`（不在 Assets 下，git 一般也不跟踪，换机会丢，需要重设）；
3. 把旧 `PreviewSession` 反射 `Dispose()` 掉、并用 `EditorSceneManager.CloseScene(previewScene, true)` 关掉残留的场景（否则 proxy 可能留到下一次刷新）。
判据：`sceneCount` 里没有 `___NDMF Preview___` 或其中 proxy=0，主场景只剩你想要的渲染器。

**重新打开**：菜单 `Tools ▸ NDM Framework ▸ Enable Previews`（或把 prefs 的 `EnablePreview` 改回 1）。


---

## §31 小狼发色菜单（8 选 1）+ 眼罩变体门控（本工作区实测 2026-10-25）

### 31.1 材质克隆换色（lilToon 头发材质）
- 做法：`Object.Instantiate(sourceMat)` → 只改 `_MainTex` / `_BaseMap` / `_BaseColorMap`（以及必要时的 `_Color`），其余（`_Main2ndTex`、matcap、outline、渲染队列）全部沿用源材质；`_Color` 建议保持接近 1，颜色交给贴图，避免二次染色。
- 图标：`Graphics.Blit` 把贴图渲进 128² RenderTexture → `EncodeToPNG` → `ImportAsset`，`TextureImporter.textureType = Sprite` 后作为菜单 `Control.icon`。
- 菜单：8 个 Toggle 共享 **同一个 Int 参数**（值 1..8），天然互斥；`isDefault` 或参数默认值负责「默认色」；再补一个 0 值兜底态（fallback 到默认色 + `VRCAvatarParameterDriver` 把参数回写为默认值），避免参数被外部写成 0 时无状态匹配。

### 31.2 FX 层直接加在头像 FX 控制器上（不要用 MA MergeAnimator）
- 只被「后合并动画」引用的对象，AAO 会当成未使用对象删除；MA MergeAnimator 在 NDMF 里排在 AAO 之后 → 自建控制器会被虚拟化后只剩 0 层。
- 做法：把层直接 Append 到头像 FX 控制器资产（本项目 = 克隆的 `Kipfel_FXLayer_v1.1.1_沐梓溪喵.controller`），用 PPtr 曲线切 `SkinnedMeshRenderer.m_Materials.Array.data[slot]`。

### 31.3 关键坑：`AnimatorController.AddLayer()` 默认 `defaultWeight = 0`
- 现象：层、状态、曲线、条件全对，但运行时完全不生效（`GetLayerWeight` 恒 0，动画不写目标属性）。**`VRCAvatarParameterDriver` 仍会执行**，所以「参数驱动型」功能看起来正常、「动画型」功能全哑——极易误判。
- 修法：加层后立刻写回：
  ```csharp
  ctrl.AddLayer("MyLayer");
  var ls = ctrl.layers;
  ls[ls.Length - 1].defaultWeight = 1f;
  ctrl.layers = ls;   // AnimatorControllerLayer 是 struct 数组，必须整体写回
  ```
- 判据：烘焙后 `fx.layers[i].defaultWeight == 1`。

### 31.4 参数默认值：VRCFury 会把 Int 参数转成 Float
- 现象：源码控制器里 `HeadType` 是 Int 默认 1，烘焙后变成 `type=Float`、`defaultFloat=0` → 依赖该参数的 reactive 层把「默认状态」判成 0，默认变成狼头、猫头/猫衣被隐藏。
- 修法：在源码控制器里对同名参数**同时**写 `defaultInt` 与 `defaultFloat`（本项目 `HeadType` 写 Int 1 / Float 1），或干脆声明为 Float 参数。
- 判据：烘焙后 `BAKED_PARAM HeadType dFloat=1`，且克隆默认态 `Body=1 / 狼头=0 / 睡衣 10 件 active`。

### 31.5 PPtr 曲线会被 VRCFury 判成 “no valid animations”
- 只有对象引用曲线（材质/贴图切换）的 clip，VRCFury 会打印 `no valid animations ... will be removed during a real upload` 并删除该层。
- 修法：同一 clip 里补一条数值曲线（例如目标对象 `m_Enabled = 1`），层就会被保留。

### 31.6 编辑器内验证的边界
- 克隆头像在 Play 模式下的 `Animator` 没有 controller（VRCSDK 只在真实 Build & Test 里驱动），手动 `runtimeAnimatorController = fx` 后 `SetInteger/SetBool` 也不会生效（读回仍是默认值）。
- 可行的替代验证：① 用 `AnimatorController` 建一个**空 GameObject + Animator** 的临时实例，设参数看状态机是否切换（证明控制器逻辑正确）；② 检查烘焙后层的 `defaultWeight`、状态数、clip 绑定路径与目标引用（证明数据正确）。
- 真正的运行时表现（材质切换、变体显隐）必须进游戏或用 Gesture Manager 由用户实测 → 报告里写 `NOT_RUN`。

## §32 VRCSDK Build 面板报「VRCExpressionsMenu uses a parameter that is not defined」

**症状**（本工作区实测 2026-10-25，LuzzyMeow V2）：Build & Test 面板红色六边形告警，`Review Any Alerts` 里两条：
`VRCExpressionsMenu uses a parameter that is not defined. Parameter: HeadType` / `... WolfHairColor`。
但 Play 模式 NDMF 烘焙出来的克隆 EP 里这两个参数**明明存在**（Int def=1 / def=6，saved=true）。

**根因**：VRCSDK 的 Build 面板预检读的是**源 `VRCExpressionParameters` 资产**（这里是 `Assets/01_模型本体/Kipfel/ExMenu/Kipfel_ExParameters.asset`，只有作者原始的 14 项），
而 `ModularAvatarParameters` 声明的参数是 **NDMF 烘焙时**才合并进克隆 EP 的 —— 源资产里没有。
更隐蔽的是：挂在**子菜单资产**里的控件不会报警（那些资产在源里是空的，靠 `MenuInstaller` 烘焙填充），
只有**直接写进描述符菜单链、源资产里已有控件**的菜单（如 `角色_狼.asset` 的 `启用小狼`、`狼_发色.asset` 的 8 个 Toggle）才会被逐条检查。

**正确做法**：
1. 把这类参数**直接写进源 EP 资产**（`VRCExpressionParameters.parameters`），类型/默认值/saved/synced 与 MA 声明保持一致（Int → `VRCExpressionParameters.ValueType.Int`，注意不是 `VRC_AvatarParameterDriver.ParameterType`）；
2. 同时把 `ModularAvatarParameters` 里的同名声明**删掉**（避免同名重复声明，MA 烘焙只会打警告但没必要）；
3. 保存 EP + 场景，**重新进一次 Play 烘培复核**：`EP_COUNT` 不重复、`DUP=0`、默认值正确（本工程 `HeadType Int def=1 saved` / `WolfHairColor Int def=6 saved`，烘焙 EP 159 项、同步位数仍 < 256）。

**判据**：Build 面板不再出现红字参数告警；Play 烘培 EP 里同名参数唯一且 `saved=True`。

## §33 把一整套 NDMF/MA 配件系统复制到另一个部位（本工作区实测 2026-10-25，小狼眼罩）

场景：猫头有一套完整交互眼罩（`ParentConstraint` 三源 + PhysBone 抓取 + 7 层 FX 控制器 + MA Parameters + 菜单），要把**完全相同**的交互复制到狼头，且两者参数互不干扰。

### 33.1 复制步骤（可复用）
1. `Instantiate` 复制整棵对象树；**不要**沿用猫的 localPosition —— 先 `SetParent(wolf,false)` 再赋 `position/rotation/lossyScale = cat 的 world 值`，否则世界坐标二次叠加（实测 mask 世界 y 从 0.686 变成 1.32）。
2. `AssetDatabase.CopyAsset` 复制控制器 → 新路径，再把所有参数名加前缀（本工作区用 `W_`）：**同时改** animator parameters、每条 transition/blend tree 条件串、`ModularAvatarParameters`、
  `MenuItem.Control.parameter.name`、`VRCPhysBone.parameter`。改完 grep 一遍旧名应为 0。
3. `ModularAvatarMergeAnimator` 指向新控制器；`MAMenuInstaller.installTargetMenu` 指向该角色自己的分类菜单副本。
4. 目标位姿按新头部实测重算：mask 三源（[初期]/Target1/Target2）的世界位置，`targetRot = desired × inverse(rotOffset)`。
5. 删掉复制件里不需要的东西（BoneProxy、EditorOnly 预览副本、重复菜单入口）。

### 33.2 两个必踩的坑
- **AAO 会删掉「只被复制件动画引用」的对象**：`TraceAndOptimize.removeUnusedObjects` 判定这些对象未被任何可达动画引用 → 烘焙后对象消失、clip 被换成 `$AvatarOptimizerClipLengthDummy$`。判据：烘焙克隆里找不到对象、
  clip 的 bindings 全是 `$AvatarOptimizerClipLengthDummy$`；修法 = 反射设 `removeUnusedObjects=false`（代价：优化变少）。
- **`AnimatorController.AddLayer()` 生成的层 `defaultWeight=0`** → 烘焙后整层不生效。修法：`var ls=ctrl.layers; ls[i].defaultWeight=1f; ctrl.layers=ls;`（数组必须整体回写）。
- **复制前必须审计「源层权重 + 无参数默认过渡」**：第三方系统里可能有 `defaultWeight=0` 的层（作者靠它「默认不生效」），复制并重新烘焙后该层权重变 1，其「只依赖系统内部参数（如 `IsLocal`）的默认过渡」会直接跑起来。
  本工作区小狼眼罩 L0 `NightModeValue` 就是这样：源权重 0，复制后权重 1 → 一进游戏本地遮光球 `_Brightness=1` 全黑（详见 §34）。判据：复制后读烘焙 `fx.layers[i].defaultWeight`，
  并逐层列出「没有任何用户参数条件的 transition」是否会把系统推进某个「全开」状态。

### 33.3 参数必须写进源 EP
VRCSDK 的 Build 预检只读 **源 `VRCExpressionParameters` 资产**，MA Parameters 里的声明要等 NDMF 烘焙才合并 → 只写 MA Parameters 会报红字 
`VRCExpressionsMenu uses a parameter that is not defined`。修法：直接往源 EP 写 `Int/Bool`（名字/默认值/synced/saved），并删掉 MA Parameters 里的同名声明避免重复。

### 33.4 菜单隔离（两个角色菜单互不串扰）
- 隔离要隔离**三层**：参数名（`W_` 前缀）、FX 层（复制的层只读 `W_*`）、OT 门控（克隆菜单项的 ObjectToggle 只指本角色变体）。
- 克隆菜单骨架只复制 `ModularAvatarMenuItem` / `MAMenuGroup` / `MAMenuInstaller`，**不要复制 ObjectToggle**（否则同一对象被两套参数驱动）。
- 判据：烘焙后逐层读 `stateMachine` 的 anyState/entry 条件，狼变体层只出现 `W_*`、猫层只出现无前缀名（本工作区实测 `CROSS_PARAM_USAGE=0`）。

### 33.5 变体的「默认可见」用场景初始状态
MA reactive 的「关」= 回落场景初始状态，所以：`Control.value` 必须写 **1**（写 0 会让 0 值组生效）、`isDefault` 决定该参数的默认值语义；默认可见的变体在场景里 `activeSelf=true`，默认隐藏的保持 inactive。判据：
烘焙后变体 `activeSelf` 与默认值一致，`activeInHierarchy` 跟随父级（狼头组）。

## §34 夜模式眼罩「本地遮光球」的 L0 状态机缺陷与门控重写（本工作区实测 2026-10-25）

资产：らずべりー工房「【MA対応】掴める！おやすみアイマスク【ナイトモードギミック設定済み】」（BOOTH 7326371，¥500，PC 限定）。
结构：`沐梓溪喵/眼罩/Root` 挂 `Assets/05_配饰/眼罩/SleepEyeMask/_Core/EyeMaskNightMode.controller`（7 层、11 参数，MA MergeAnimator relativePathRoot=Root）
+ MA Parameters 10 项；遮光球 `Root/Sphere`（shader `RBS/SleepEyeMask/NightMode`，`_Brightness`/`_Darkness`，renderQueue 5000）
+ 遮罩 `Root/[設定用]アイマスク`（ParentConstraint 三源 [初期]/Target1/Target2）。

### 34.1 缺陷（用户报「一进游戏全黑、开关关不掉」）
- L0 `NightModeValue` 源权重 0（烘焙后 1）：默认态 `isLocal` → `NightModeOn(Local)`（只要求 `IsLocal`=true）→ clip 同时做三件事：遮光球 `m_IsActive=1`、`material._Brightness` 0→1、遮罩约束三源权重 0/0/1→0/1/0。
- L6 `Toggle` 只切遮罩网格 `m_IsActive`（眼罩开关），**与遮光球毫无联动** → 本地玩家进游戏就被 100% 黑罩住，菜单关眼罩也不恢复。
- 结论：不是「亮度参数错」，而是「本地变暗的门控缺失」。

### 34.2 修复（cat / wolf 两套控制器同样处理）
- `isLocal`（默认）motion 换成新 clip `EyeMaskDimOff_*`：`Sphere.m_IsActive=0` + `material._Brightness=0`。
- 新增状态 `Dim(本地遮光率)`：1D 混合树（`blendParameter = X_RBS_EyeMask_AAP_Value`；child th=0 → clip `EyeMaskDim0_*`（active 1 / brightness 0）；th=1 → `EyeMaskDim1_*`（active 1 / brightness 1））。
- 转移：`isLocal --IsLocal=true 且 Toggle=true--> Dim`；`Dim --Toggle=false--> isLocal`；`isLocal --IsLocal=false--> NightModeOn(Remote)`（远端行为保持不变）。原 `isLocal→NightModeOn(Local)` 与其它旧转移清空，旧 state 保留但不可达。
- 遮光率入口：菜单 `亮度调节`（RadialPuppet）显式参数设为 `RBS_EyeMask_AAP_Value` / `W_RBS_EyeMask_AAP_Value`（`automaticValue=false`），0=不遮、1=全黑。
- 判据（烘焙克隆）：默认两遮光球 `activeInHierarchy=False`；Toggle 开→对应遮光球 active 且遮罩可见；关→回落。

### 34.3 验证边界与陷阱
- `AnimationClip.SampleAnimation` 与「手动 `Animator.Update()`」在编辑器里都**不会**改变 `material._Brightness`（作者原 clip 也一样）→ 材质亮度只能进游戏验证；编辑器侧用 `m_IsActive` 曲线、混合树参数/阈值、状态转移与烘焙层权重核对。
- `AnimatorController.AddLayer()` 新建层 `defaultWeight=0`，必须补 1，否则烘焙后整层不生效。
- MA 会把控制器内参数烘焙成带 `$hash` 后缀的名字（动画条件里用的就是哈希名），改条件要按烘焙后名字核对。
- 给菜单径向设显式参数要在**场景**的 `ModularAvatarMenuItem.Control.parameter.name` 上改（烘焙时才写进 EP），改控制器参数名无效。

---

## §35 修复「别人写的系统」：优先回滚作者实现，而不是重写状态机（本工作区实测 2026-10-25）

### 35.1 症状与误判
- 原插件（らずべりー工房「掴める！おやすみアイマスク」）本来的交互是：菜单打开 → 面罩出现在**头顶（眉毛）** → 玩家/他人抓取下拉 → 覆盖眼睛 → **遮光球亮度按菜单设定淡入**（默认亮度 → 设定亮度）。
- 我为了修「遮光球全局生效/全黑」把控制器 L0 `NightModeValue` 改写成「`Toggle` 为真才进遮光态」的简化逻辑 → 黑屏没了，但**作者做好的「抓取下拉 + 淡入」也没了**。这种改写本质上是**破坏作者状态机**，后续无论怎么补都难还原。

### 35.2 正确做法（回滚 + 最小集成）
1. **从原始资产包恢复控制器字节**：`Assets/05_配饰/眼罩/SleepEyeMask/_Core/EyeMaskNightMode.controller` 用原包同名文件覆盖（保留 `.meta` → GUID 不变，场景引用不断），只删自己新增的 clip。实测原始 35,848 B / MD5 `FE2F2753E6D8124D1359CBAD924261F5`。
2. **复制一套给第二个角色**时，不要调 `AnimatorController.AddParameter/RemoveParameter + 改条件`（`AnimatorCondition` 是 struct，条件字段不可写）。把 `.controller` 当文本处理最稳：
   ```powershell
   # 复制后做纯文本改名，参数表 / 转移条件 / BlendTree 参数一起改到
   (Get-Content -Raw $ctrl) -replace 'RBS_EyeMask_', 'W_RBS_EyeMask_' | Set-Content -NoNewline $ctrl
   ```
   注意前缀匹配会漏掉 `RBS_EyeMaskReceiver_Value` 这类「EyeMask 后面不是下划线」的参数 → 补一次单独替换。`IsLocal` 这类**全局语义参数必须共享**，不能加前缀。
3. 复制后控制器资产的内部 `m_Name` 会带着临时名（`__tmp_wolf_ctrl`），Unity 会警告 `Main Object Name does not match filename`；文本替换 `m_Name:` 后重新导入即可。

### 35.3 角色切换门控：用 `Renderer.m_Enabled`，不要用 `m_IsActive`
- 需要「切到狼 → 猫的所有交互物件关闭；切回猫 → 恢复上次状态」时，**参数**要保留（`saved=true` 记住），**可见性**用独立 FX 层写 `Renderer.m_Enabled`：
  - 原因：`m_IsActive` 常被 MA reactive / 作者层同时驱动，后写者覆盖（实测猫帽子层把自建层的 0 覆盖回 1）；`m_Enabled` 作者层基本不碰。
  - 层必须 `defaultWeight = 1`（`AddLayer` 默认 0，否则整层不生效）。
  - 曲线路径要在**烘焙后**的克隆里逐条验证（本次 40 条全部解析成功；AAO 会把层级压平、VRCFury 会插 `Head (HeadChop)` 副本，路径别写死）。
- 烘焙后 MA 会给参数加 `$哈希` 后缀（例：`RBS_EyeMask_Toggle$0cb39e909014`、`W_RBS_EyeMask_Toggle$064a1e22aca8`），菜单控件与动画层用的是同一哈希名 → 链路自然闭合。

### 35.4 预检（Build & Test 红字）的真实范围
- `VRCExpressionsMenu uses a parameter that is not defined` 只检查**源菜单资产可达**的控件；由 MA 在烘焙时注入的菜单项（MenuInstaller → 目标子菜单）**不会**被预检点到，所以那类参数只在源 EP 里缺也没关系（本次 `RBS_*` 全在 MA Parameters 里声明，预检通过）。
- 反过来，**直接挂在描述符菜单树上的控件**（本次 `角色` → `小狼/小猫` 的 `HeadType`、`WolfHairColor`）必须在源 EP 里显式存在，否则红字。

### 35.5 判据
| 检查 | 期望 |
|---|---|
| 作者控制器字节 | 与原始资产包一致（MD5 对） |
| 副本控制器 | 层数/状态数/转移数与原版一致，仅参数名不同 |
| 烘焙 FX | 两套系统层同时存在（本次猫 L30–L36、狼 L37–L43） |
| 默认态 | 两套 `mask`/`sphere` 都 `active=false`；`*_Toggle` def=0 saved=true net=1 |
| 门控层 | `defaultWeight=1`；绑定解析 n/n；条件 `HeadType` 上下界正确 |
| 菜单 | 根 = 小狼 / 小猫 / 衣柜（共享）；猫 mask → 小玩具；狼 mask → 狼_小玩具 |

## §36 角色化菜单里的「共用衣服/头饰开关不同步」：克隆菜单项必须共用同一参数（本工作区实测 2026-10-25，第 90 轮）

### 症状（共用开关不同步）
- 衣柜已做成「角色外共用菜单」，但切到小狼后，在共用衣柜里关「睡衣 → 帽子」，只有小猫的帽子消失，小狼的帽子变体仍在；眼镜、名牌、小翅膀、睡衣2眼罩、浴衣面具、泳装遮阳帽同理。

### 根因（共用开关不同步）
- 第 84 轮做角色菜单时，给小狼克隆了 7 个头饰项的 `ModularAvatarMenuItem`，并给克隆项分配了**独立参数**（`W_S1_Hat` / `W_S2_Mask` / `W_Y1_Mask` / `W_SS_Hat` / `W_Acc_Glasses` / `W_Acc_Nameplate` / `W_Acc_Wing`）；
- 共用衣柜里的原件用的是共享参数（`S1_Hat` / `S2_Mask` / … / `Acc_Glasses_v2` / `Acc_Nameplate_v2` / `Acc_Wing_v2`）→ 同一件物品存在两套开关系，菜单点的是共享参数，狼变体听的是 `W_*`，永远不同步。

### 正确做法（共用开关不同步）
1. **共用物品 → 克隆项改回共享参数**：`mi.Control.parameter.name = 共享名`、`mi.Control.value = 1f`、`mi.automaticValue = false`；随后从 `一键穿衣设置` 的 `ModularAvatarParameters` 里删除对应的 `W_*` 声明（本工程 7 项，删后 56 项）。
2. **把「狼变体」行补到共用原件的 ObjectToggle 上**（幂等：按 `AvatarObjectReference.referencePath` 去重）：
   - 双 OT 项（隐藏 + 显示两个 `ModularAvatarObjectToggle`，如 `睡衣/睡衣/帽子`、`菜单开关/眼镜`）：两个 OT 各追加一行（隐藏 OT `Active=false`、显示 OT `Active=true`）；
   - 单 OT 项（开=显示，如 `睡衣2/睡衣2/眼罩`、`浴衣/浴衣/面具`、`夏装/泳装/帽子`）：追加 `Active=true` 一行即可。
   - 场景默认态与参数默认值保持一致：默认=1 的变体场景里 active、默认=0 的 inactive。
3. **不要**给克隆项建第二套 OT：同一对象被两套参数驱动时 MA 会生成互相牵扯的 responsive 层，表现为「点一个另外一个乱跳」。

### 判据（烘焙后，Play 内读 `VRCFury FX.controller`）
- 成对的 `MA Responsive:` 层出现且 transition 条件参数名相同：`MA Responsive: PJ_Hat` 与 `MA Responsive: 狼头版_睡衣帽子` / `狼头版_睡衣帽结` 都是 `S1_Hat`；
  `MA Responsive: Eyemask` 与 `狼头版_睡衣2眼罩` 都是 `S2_Mask`；`MA Responsive: Chain` / `Glasses_Drop` 与 `狼头版_眼镜链` / `狼头版_眼镜垂饰` 都是 `Acc_Glasses_v2`。
- 每个成对层都同时有 `m_IsActive=0` 与 `m_IsActive=1` 两条 clip（只出现 1 侧 = 反相或门控缺失）。
- 烘焙参数表里 `W_` 前缀参数计数为 0（源侧 `MA_PARAMS` 无 `W_`）。
- 场景侧 OT 行数：双 OT 项 rows=4（猫 2 + 狼 2）、单 OT 项 rows=2。

### 验证片段（`execute_code`，CodeDom 可用）
```csharp
var fx = (UnityEditor.Animations.AnimatorController)desc.baseAnimationLayers[4].animatorController;
for (int i = 0; i < fx.layers.Length; i++) {
    string ln = fx.layers[i].name;
    if (!ln.Contains("狼头版_") && !ln.Contains("PJ_Hat")) continue;
    var sm = fx.layers[i].stateMachine;
    for (int s = 0; s < sm.states.Length; s++) {
        var st = sm.states[s].state;
        for (int t = 0; t < st.transitions.Length; t++)
            foreach (var c in st.transitions[t].conditions)
                Debug.Log(ln + " cond " + c.parameter);
    }
}
```
- 注意：`Control.value` 必须写 1；写成默认值 0 会让 MA 生成「参数 ≈ 0 才显示」的条件（本工程踩过一次，表现为所有变体默认反相）。



## §37 换头后新头「说话嘴不动」：VisemeSkinnedMesh 单网格限制 + 自建 Viseme 驱动层（本工作区实测 2026-10-25，第 91 轮）

### 症状（换头后口型）
- 猫/狼双头可切换，但只切到新头（狼头）时，声音捕捉不驱动嘴型；旧头（猫头）正常。

### 根因（两条叠加）
1. **VRChat 原生唇形只能驱动一个网格**：`VRCAvatarDescriptor.lipSync = VisemeBlendShape` + `VisemeSkinnedMesh` 只接受一个 SkinnedMeshRenderer（作者填的是猫头 `Body`），
  `VisemeBlendShapes` 15 项映射 `vrc.v.sil|pp|ff|th|dd|kk|ch|ss|nn|rr|aa|e|ih|oh|ou` → 新头（`狼头/Body`）根本不参与原生唇形。
2. 新头自己的 15 个 viseme 形态键名字不同：`v_sil|v_PP|v_FF|v_TH|v_DD|v_kk|v_CH|v_SS|v_nn|v_RR|v_aa|v_E|v_ih|v_oh|v_ou`，且原作者的 `LilLeo_FX_FaceOnly.controller` 面部层曾被清空（0 states）→ 手势/眨眼/viseme 全失效。

### 做法（自建 Viseme 驱动层，纯 FX 动画，无需改 descriptor）
1. 在换头控制器（`Assets/08_动画·菜单·图标/换头系统/LilLeo_FX_FaceOnly.controller`）里加一层 `WolfFace Viseme`（`defaultWeight = 1`）：
   - 15 个 clip `Assets/08_动画·菜单·图标/换头系统/wolflipsync/WolfViseme_<shape>.anim`；
   - 每个 clip 写 **15 条** `blendShape.v_*` 曲线（path = `armature/Hips/Spine/Chest/Neck/Head/狼头/Body`），目标形状 100、其余 0；
   - 15 个 state + 15 条 **AnyState** 转移，条件 `Viseme Equals k`（k=0..14），`hasExitTime=false`、`duration=0`、`canTransitionToSelf=false`；默认态 = `v_sil`。
   - VRChat 内置 `Viseme` Int 参数 0..14 顺序：0 sil / 1 PP / 2 FF / 3 TH / 4 DD / 5 kk / 6 CH / 7 SS / 8 nn / 9 RR / 10 aa / 11 E / 12 ih / 13 oh / 14 ou（内置参数不计入 256 bits）。
2. 重建被清空的面部层：**不要**用 `ScriptableObject.CreateInstance<AnimatorStateMachine>()`（`AnimatorStateMachine` 不是 ScriptableObject，会 CS0311）——
  正确姿势是 `controller.AddLayer(name)` 由 Unity 自动创建状态机，取 `controller.layers[i].stateMachine` 再往里面 `AddState/AddTransition/AddAnyStateTransition`，
  最后把层 `defaultWeight` 改 1 并写回 `controller.layers`。

### 坑：AAO 会合并/重命名形态键，按名字的动画全失效
- `TraceAndOptimize.optimizeBlendShape = true` 时，烘焙后网格形态键从 208 个被合并成 72 个并改名成 `AAO_Merged_v_aa_9` 之类 → 自建 clip 的 `blendShape.v_aa` 找不到目标，嘴照样不动（而且不报错）。
- 判据：烘焙后在 Play 克隆里读 `SkinnedMeshRenderer.sharedMesh.blendShapeCount` 与 `GetBlendShapeName(i)`。
- 处理：该头像的 `TraceAndOptimize` 关掉 `optimizeBlendShape`（本工作区同时 `mergeSkinnedMesh/removeUnusedObjects/optimizeAnimator = false`）；重烘后形态键恢复 208 且名字保留。

### 验收判据（编辑器内可做，无需进游戏）
- 烘焙 FX 里 `WolfFace Viseme` 层：`states=15`、`anyStateTransitions=15`、每条条件 `Viseme Equals k`、`defaultWeight=1`；
- 每个 clip `GetCurveBindings().Length == 15`，全部 path 可解析，目标形状曲线值 100、其余 0；
- 烘焙网格上 `GetBlendShapeIndex("v_aa") >= 0`；
- 真机说话效果仍需 Build & Test / 进游戏验证（**NOT_RUN**）。

## §38 自建口型/表情层「一卡一卡、硬切」的成因与调法（本工作区实测 2026-10-26）

**症状**：自建的第二张头（换头后目标头）口型看起来帧数很低、嘴形直接跳变；手势表情切换生硬。

**根因（三条同时存在）**
1. **过渡时长 = 0**：状态机里 `AnimatorStateTransition.duration = 0` 且 `hasExitTime=false` → 状态之间是硬切，没有交叉淡化。作者原猫版表情层用的是 `duration = 0.2`（表情）/ `0.1`（AnyState 宠物表情），所以看起来顺滑。
2. **口型 clip 只有 1 个关键帧**：VRChat 内置 `Viseme`（Int 0–14，0=sil、14=ou，见 https://creators.vrchat.com/avatars/animator-parameters/ ）逐帧变化；单关键帧常量曲线 ⇒ 每换一个音素就整值跳变。
3. **没有 interruption 设置**：默认 `interruptionSource = None` → 新音素要等当前过渡播完才能进入，快速说话时会「憋一下」。

**修法（可复用配方）**
- 表情层：状态间过渡 `0.15–0.20s`，AnyState 过渡 `0.1–0.12s`，全部 `hasFixedDuration = true`、`interruptionSource = Destination`。
- 口型层：15 个音素状态，AnyState 过渡 `0.06–0.08s`、`interruptionSource = Destination`；每个 clip 用 3 个关键帧做包络（0s=0、0.02s=100、0.12s=100，`AnimationClip.length=0.12`、状态不循环）→ 进入时淡入、离开时由过渡淡化，视觉上接近原生唇形。
- 想更接近原生「说话即动」：可再用内置 `Voice`（Float 0–1）做一层 additive 的开口量，叠加在口型层之上（本次未做，属可选项）。
- 判据：烘焙后（VRCFury FX）`WolfFace Viseme` 层 15 状态、AnyState `duration≈0.08`、clip 3 keys/length 0.12；表情层 AnyState `0.12`、状态 `0.2`。

**边界**：VRChat 原生唇形（Descriptor 的 Viseme Blendshape 模式）对**单个网格**做逐帧插值，自建 animator 层只能逼近、不能完全等价；实测需进游戏看口型。

## §39 自建口型能做到的「最丝滑」上限：Viseme 状态机调参 + Voice 加法层（本工作区实测 2026-10-26，第 93 轮）

**症状**：换头后新头的自建口型能动了，但和小猫（原生唇形）比明显「一卡一卡、帧数低、切换硬」。

**根因（不是层没复制全）**：VRChat 原生唇形 = 描述符 `lipSync = VisemeBlendShape` 指向**唯一一个** `VisemeSkinnedMesh`，由客户端按音频逐帧连续加权；小狼2.0 的丝滑嘴型正是这个原生通道，
**它的 Animator 里没有任何口型动画层**（`LilLeo_FX` 20 层里只有 FaceDefault / Left Hand / Right Hand / BlinkAuto / Viseme Lock 5 层与表情相关，其余 15 层全是身体部件开关）。所以「复制它的动画层」
拿不到丝滑；原生目标只能给一个头，另一个头只能靠 Animator。

**做法（B 方案：小猫保原生、小狼走 Animator 的极限优化）**
1. **15 态 Viseme 层调参**（`WolfFace Viseme`）：每条 AnyState 过渡
   - `duration = 0.07`（0.05–0.09 是甜点：太小就硬切、太大就糊嘴/延迟）
   - `hasFixedDuration = true`
   - `interruptionSource = Source`（连续换音时从当前混合权重再起混合，避免回跳）
2. **口型 clip 用 3 关键帧包络**：`0@t=0 → 100@t=0.03 → 100@t=0.12`，clip 长度 0.12；每个 clip 写满 15 个 `v_*` 曲线（全姿态插值，避免部分曲线残留）。
3. **`Voice`（Float，0–1 音量）加法层** `WolfFace Voice`：
   - `AnimatorControllerLayer.blendingMode = AnimatorLayerBlendingMode.Additive`，`defaultWeight = 1`
   - 1D Blend Tree，`blendParameter = "Voice"`，3 个 clip：`th=0` 全 0；`th=0.45` → `v_aa+22, v_oh+9, v_ih+6, v_E+5, v_ou+3`；`th=1` → `v_aa+45, v_oh+18, v_ih+12, v_E+10, v_ou+6`
   - 作用：离散口型之间由音量提供**连续开口起伏**，观感平滑一档；不发声时（Voice≈0）不叠加。
4. **判据（烘焙后）**：FX 层数 +1；`WolfFace Viseme` 层 `anyDur=0.07 / interruptionSource=Source / clip curves=15 len=0.12`；
  `WolfFace Voice` 层 `mode=Additive / tree blendParameter=Voice / kids=3`；控制器含 `Voice` Float default 0；狼头 Body 15 个 `v_*` 形状仍在（AAO 未改名）。

**上限**：仍不及原生（原生按音频逐帧加权，Animator 只有 15 个离散口型 + 固定混合时间）→ 听感/观感只能进游戏实测；这是该限制下能做到的最好形式。

**收尾坑**：删除 `Packages/com.vrcfury.temp`（VRCFury 生成的临时包）后如果编辑器还开着，VRCFury 会反复报 `DirectoryNotFoundException: ...\com.vrcfury.temp\PlayModeSettings`。做法：
只删 `Builds/` 内容或干脆保留该包目录；若目录已删，重建空目录 `Packages/com.vrcfury.temp`，VRCFury 会把 `PlayModeSettings`（文件）写回去，Console 即恢复干净（勿把它建成同名文件夹，
否则报 `UnauthorizedAccessException`）。

---

## §40 小狼2.0（LilLeo）原始头像 / 动画层速查：换头时哪些能复用（本工作区实测 2026-10-26）

> 场景：把「小狼2.0」的头挪到小猫身上做菜单切换后，判断「能不能直接复制旧模型的动画层」。实测答案：**表情相关的 5 层可以（已复用），其余 15 层不行；而"嘴型丝滑"根本不在这 20 层里**（走的是原生唇形）。

**描述符（`小狼2.0` 根上的 `VRCAvatarDescriptor`）**
- `lipSync = VisemeBlendShape`；`VisemeSkinnedMesh = Body`（208 个形态键）；`VisemeBlendShapes` 15 项 = `v_sil, v_PP, v_FF, v_TH, v_DD, v_kk, v_CH, v_SS, v_nn, v_RR, v_aa, v_E, v_ih, v_oh, v_ou`
- `enableEyeLook = True`（眼骨驱动）
- 5 层控制器：`Base = LilLeo_Base`、`Additive = NULL`、`Gesture = LilLeo_Gesture`、`Action = NULL`、`FX = LilLeo_FX`

**`LilLeo_FX.controller`：20 层 / 19 参数**
- 表情相关 5 层：`L1 FaceDefault`（1 态）、`L2 Left Hand`（8 态 + 8 AnyState，GestureLeft 1..7）、`L3 Right Hand`（8+8）、`L18 BlinkAuto`（2 态）、`L19 Viseme Lock`（2 态）
- 身体部件 15 层（对换头无意义，网格不存在）：`L0 AllParts(w=0)`、`Armband`、`Cape_Main`、`Cape_Neck`、`Cat_Ear`、`CollarChange`、`Tail`、`Glove`、`Garters`、`Whithle`、`Human_Ear`、
  `Hair_Aho`、`Hair_Front_Short`、`Hair_Side_Short`、`Hair_Side_OFF`

**换头结论（本工程已做）**
1. 复制时只取那 5 个表情层，**层内 clip 路径要重写成新头路径**（本次 `armature/Hips/Spine/Chest/Neck/Head/狼头/Body`），否则 `NO VALID ANIMATIONS`。
2. 口型层它**没有**——它的丝滑来自原生 `VisemeBlendShape`；原生只能指一个网格，所以换头后第二个头只能在 Animator 里重建 15 态（见 §37/§39），丝滑度上限也在那里。
3. 身体部件层不要复制：既无网格，也会把 `Armband/Cape_*` 等参数带进 EP 白占同步位。

---

---

## §41 一键穿衣互斥的标准结构：共享 Int + 多 Toggle 同参不同值 + 0 值回落（本工作区实测 2026-11-05，LuzzyMeow V2 沐梓溪狼 T3）

> 场景：5 套衣服，要求「同时只能开一套、且必须始终有一套」；关掉当前套 = 回到默认套，而不是裸装。

**四件套**
1. **共享 Int 参数**（例 `OutfitOneClick`）：`ModularAvatarParameters` 显式声明 Int、`defaultValue` = 默认套件值、synced + saved。
2. **每套一个 MenuItem Toggle**，绑同一参数、`Control.value` 各不相同（1..N）；**只有默认套件勾 `isDefault`**（多个 isDefault 官方明确「results are undefined」）。
3. **0 值回落态**：Toggle「关」时把参数写 0（Int 也写 0）→ 必须存在「参数 == 0 → 应用默认套件」的状态（FX：AnyState → `Apply_0_Default`，条件 `OutfitSelect Equals 0`），否则 0 值无命中 → 空装或两套叠加。
4. **部件可见性交给 FX 驱动层**：每个套装态的 `VRCAvatarParameterDriver` 写全部部件 Bool（该套 =1、其余 =0），不要靠 N 个 OT 互相覆盖。

**配套校验（三处默认值必须一致）**：MA Parameters 的 `defaultValue`/`hasExplicitDefaultValue`、源 `VRCExpressionParameters.defaultValue`、FX controller 的 `defaultInt` —— Play / GM 初次读的是 AnimatorController 的默认值。

**坑**
- **不要用 Button**：官方「sets a parameter when clicked, then resets… usually after about a second」，只能表达动作不能表达状态（本册 §17 (L315) 的 Button 方案已被 `03-06-ma-menu-build-by-reflection.md` §14.4 实测推翻）。
- <2 套时 MA 自动编成 Bool；≥3 套才升 Int，自动值从 0 顺排、default 绑第一个被声明的开关 → 要可控就显式声明 Int。
- MenuItem 的 `isSaved` **不会**把参数写进 EP；只有源 EP 里该参数的 saved 才决定跨世界保存（`03-01-parameter-and-menu-audit.md` §8 (L168)）。
- 部件 Bool 一律 saved=false，否则玩家以「半裸」状态回归（`03-06-ma-menu-build-by-reflection.md` §14.3 (L564)）。
- `VRCAvatarParameterDriver` 只在 VRC 运行时执行；最小 Animator 场景里 Animator.Update 能驱动图层，但 driver 的 Set 条目不执行（`05-08-cross-avatar-garment-fit-newfit.md` §22.4）。
- Bool 会被 MA 转成 synthetic 中转参数，Play 里写入有延迟，至少等 20 帧再读（`03-06-ma-menu-build-by-reflection.md` §12.5 (L462)）。

**适用场景**：多套衣装/多发型/多形态的「单选且不可空」；本次 T3（0 时回落 1）。

> 更新时间 2026-11-05（本次来源：沐梓溪狼衣装菜单重构 / LuzzyMeow V2 T1–T10）
