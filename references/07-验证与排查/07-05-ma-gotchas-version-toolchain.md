# MA 常见坑、版本快照与工具纪律

> 本册整理自一套 MA 常见坑与版本实测规范（源仓库信息见 `ACKNOWLEDGEMENTS.md`），以及 4 个只读辅助脚本的用法。
> 本册的**坑与版本**部分是实打实的新增（本包既有分册没有这些 MA 版本行为）；**证据层级与授权**部分与本包 `07-02-evidence-and-ownership.md` 高度重合 —— 本文只记外部规范口径的**差异与补充**，不重复既有分册。
> ⚠️ 本包工具前缀是 `mcp__unity__*`（DSH 接入的 unity-mcp），外部规范写的是 OpenCode 的 `unityMCP_*`，一一对应。

## 1. 十条常见坑（symptom → root cause → fix）

### 1.1 EditorOnly 标签会在构建时剥掉节点

- **症状**：构建后**一整类节点消失**，而菜单/参数/开关全都正常。
- **原因**：`EditorOnly` 标签（或被标成 EditorOnly 的文件夹）会在构建时**静默剥掉整棵子树**；被剥掉的对象**不会有反应式层**。
- **修法**：确保**容器与目标节点都是 `Untagged`**。遇到「某一类东西构建后全没了」，**先查容器/父节点的 tag**。

### 1.2 ObjectToggle 的 referencePath 必须完整

- **症状**：ObjectToggle **静默失效**。
- **原因**：`referencePath` 为空或写错。
- **修法**：写**完整的头像相对路径**（例如 `Clothes/Default/Cardigan`）并设 `targetObject`。

### 1.3 automaticValue 陷阱（互斥会静默失效）

MA 的默认值逻辑（`ParameterAssignerPass`）：**只有 `isDefault && !automaticValue` 的项才会成为参数的默认值**。

- 互斥组里 **`isDefault + automaticValue=true`** → 参数默认值变成 **0** → 加载时**什么都没穿**（当参数**未被显式声明**时）。
- **非默认项 + Bool/Float 自动参数** → MA **强制 value=1** → **两套同穿**、点了不切换。
- **未声明的 Int 参数** → MA 会**自动顺排值**（意外形成互斥）。

**症状**：两个 `all` 开关都变成 value=1；点某套不切换；构建出的 `cloth`/`hair` 是 **Bool 而不是 Int**；启动时空着。

**范围与真正的修法**：这些只在**2 套以上共用同一个参数**时发生。稳妥修法是**用 MA Parameters 把 `cloth` 显式声明为 Int**（`defaultValue` = 默认套装的值），并给每个 `all` 一个**手动且互不相同**的 `Control.value`。
有显式 Int 声明时，`automaticValue` **可以保持 true** —— MA 尊重手动值，互斥正常（实测：`cloth` 构建为 Int def=1、默认容器激活、其余不激活）。**单套**的 `cloth` 自动生成 Bool（不加 Parameters 组件）也没问题。
如果 `cloth` **完全不声明**，MA 只在**出现第 3 个不同的自动值时**才把它提升为 Int（2 个值 → Bool，值 0/1）—— 但那时值是**从 0 自动顺排**，`defaultValue` 绑到**第一个** switch，启动时穿的是「碰巧最先声明的」那套，不一定是想要的。

**构建后核对**：多套的 `cloth`/`hair` 是 **Int**、default = 默认套装的值（**不是 0**）、默认容器激活、其余不激活。

### 1.4（原文 3b）prefab 实例会阻止重定父级

- **症状**：`Transform.SetParent()` **静默无效**；Console 报 `Setting the parent of a transform which resides in a Prefab instance is not possible`；对象仍留在头像根下。
- **原因**：Unity 禁止把 **prefab 实例的子对象**重定父到场景里新建的容器下。
- **修法**：先 `PrefabUtility.UnpackPrefabInstance(rootGO, PrefabUnpackMode.OutermostRoot, InteractionMode.UserAction)`。**商业 prefab 资产本身不动**；只有场景实例变成独立对象。确认 `IsPartOfPrefabInstance=false` 并保存场景。

### 1.5（原文 4）codegen 里的中文会变 `?`

CodeDom/C#6 直接写中文字符会落成 `?` 乱码。**永远用 Unicode 转义**（例如 `\u4E0A\u88C5`）。可以用 base64 反查真实名字。

### 1.6（原文 5）移动节点后 SMR 的骨骼引用会断

- **症状**：某个蒙皮网格（常是从 prefab 克隆来的）重定父级后**不再跟随身体**。
- **原因**：SMR 的 `bones` 仍指向**旧的 prefab 骨架**，不是头像的主 Armature。
- **检测**：统计**父链能走到主 Armature 的骨骼数**（坏的时候是 `0/222`）。
- **修法**：按名字从**主 Armature** 重建 `smr.bones`；设 `smr.rootBone = AutoAnchorObject`。核对 `222/222` 在主 Armature 下，并且要**看真实引用**、不能只看名字。

### 1.7（原文 6）重定父级后 FX 动画路径会断

重新分组节点会打断 FX clip 里绑的旧根路径（`path: Cardigan` → `path: Clothes/Default/Cardigan`）。要**为 FX 控制器可达的所有 clip 重写绑定**。滑条层留在 FX；开关**搬到 MA**（并把旧的 FX 开关层删掉，避免双重控制）。

**重建模式（存量头像）**：当移动节点打断原厂控制器的 clip（例如 `item_Default/kutsushita/hadashi` 绑裸路径 `socks`/`shoes`）时，**不要改商业控制器**。在菜单项节点上用 MA 组件把这段逻辑重做：

- GameObject 激活开关 → `ModularAvatarObjectToggle`（`m_objects` = `{Object.targetObject, Active}`）。
- 形态键 → `ModularAvatarShapeChanger`（`m_shapes` = `{Object.targetObject, ShapeName, ChangeType=Set, Value}`，threshold ≈0.01）。
- **空 param 的 MenuItem + `automaticValue=true`** → MA 自动为每个开关创建 Bool 参数。
- 原厂那些过期 clip 仍留在 FX 里、绑着死路径 → **无害**（没有东西驱动它们）。验证方法：在构建克隆上扫 clip，确认 MA 的 `Property Overlay controlled by <name>` 项已经在驱动**新路径**。

**序列化构造的坑**：填充 MA ObjectToggle/ShapeChanger 要走 `SerializedObject` ——
`m_objects[i].Object` 是嵌套的 `AvatarObjectReference`（含 `referencePath` + `targetObject`），
所以要写 `FindPropertyRelative("Object.targetObject")` / `("Object.referencePath")`，**不能**直接写 `"Object"`。`shoeMI.Control.type` 直接赋值**能编译但可能不可靠**，
要用 `SerializedObject.FindProperty("Control.*")` 设置 `Control.type/name/parameter.name/value`。

### 1.8（原文 7）多头像场景

- **不要在脚本里硬编码头像名**（`root.Find("mango.milfy.26.08.05")` 一改名就废）；要**动态定位活动头像**。
- **非激活头像的 NDMF 构建不可靠**（会缺 SMR 等）→ 构建前**先激活**目标。
- 跨头像共享的表情资产：MA 在构建时按克隆生成副本，**原始共享资产不受影响** —— 这是正常的，不是参数丢失。
- 只留**一份工作副本（最新版）**，旧版本当备份。

### 1.9（原文 8）商业模块与自动生成的 NDMF 菜单

- 只要**同一节点上有 MenuItem**，模块的 MenuInstaller 就会生效 → 菜单可能**重复出现**；应该把它**折进自己的菜单**，而不是删掉。
- NDMF 插件型资产（LightController）在构建时**生成菜单** → **不需要**手工接线。
- 核对生成的菜单**不冲突**。

### 1.10（原文 9）改中文菜单名会改坏功能

- **不要**改「挂 `ModularAvatarObjectToggle` + 空 param」的节点名（自动参数名会变，开关就坏）。
- **不要**改空 param 滑条的节点名（Front Length / Twintail Length）。
- 品牌名保持英文（`Nova`/`@MA_Lili`/`GLEIPNIR`/`KnitNoise`/`Default`），商业模块内部不要动。
- **可以安全改名**：固定参数项（`Nail`/`Face`/`FHSharp`）、`all` 总开关（cloth/hair）、KumaPhone。

### 1.11（原文 10）残留/过期菜单项

- 移动节点后旧的 group 引用会**过期** → 删掉残留 group 或更新引用。
- 留意**裸名字的杂项**（例如鞋类子菜单里一个名字等于 GameObject 名的项），那是改名失误留下的。
- 重新分组后核对 **group 名与内容是否还匹配**。
## 2. 字段级实测（MA 1.18.7，可直接复用但要按本机版本复验）

外部工程实测（2026-09-02）：已装 **Modular Avatar 1.18.7**（embedded）、NDMF 1.14.8、VRCSDK 3.10.4、Unity 2022.3.22f1。以下行为经**已安装源码**核对：

- `ModularAvatarMenuItem`：`Control.type` 枚举值是 `Button=101 / Toggle=102 / SubMenu=103` —— 而 `SerializedProperty.enumValueIndex` 是 **0-based（0/1/2）**，
  **不是**原始值；通过 `SerializedObject.FindProperty("Control.*")` 设置。
- `MenuSource`：`MenuAsset=0 / Children=1`；`menuSource_otherObjectChildren` 会覆盖 children 来源。
- `ModularAvatarObjectToggle`：字段 `m_objects` = `ToggledObject{Object(AvatarObjectReference), Active}`。
  `AvatarObjectReference` 序列化为 `referencePath` + `targetObject`（私有）；用 SerializedProperty 写 `Object.targetObject` / `Object.referencePath`。
- `ModularAvatarShapeChanger`：`m_shapes` = `ChangedShape{Object, ShapeName, ChangeType, Value}`，另有 `m_threshold`（默认 ≈0.01）。
- **空 param 的 MenuItem + `automaticValue=true`** → MA 自动创建 Bool `__MA/AutoParam/<name>$hash`（**saved+synced**）。Toggle 上的 `cloth` 参数会**自动生成 Bool** —— **一套够用**；加第二套前必须变成 Int（走 Parameters 或显式声明）。
- **显式声明 Int 时 `automaticValue=true` 仍然可用**：`ModularAvatarParameters` 声明 `cloth`（Int，defaultValue=1）+ 每套 `all` 的 MenuItem 持手动 `Control.value`（1/2/…）→ MA 构建 `cloth` 为 Int def=1 并切换**正确的**容器（在双套头像上验证过）。
- **未声明的参数在第 3 个不同自动值时才被提升为 Int**：完全不声明时，2 个自动开关构建为 Bool（值被强制 0/1）；加第 3 个自动值时 `cloth` 升级为 Int，但值从 0 自动顺排（0/1/2），`defaultValue` 绑**第一个**声明的开关 —— 不是手动值，默认也不一定是想要的那套。**多头像互斥仍建议显式 Int 声明**。
- **商业模块的部件开关**：直接绑模块自己的 Bool 参数（如 `coat_off`，`automaticValue=false` + 固定参数名）**不需要重新声明**，模块控制器会驱动它。**移除模块自带的 `ModularAvatarMenuInstaller`** 可阻止它的菜单装到头像顶层。
- `AvatarProcessor.ProcessAvatar(GameObject)`（命名空间 `nadena.dev.modular_avatar.core.editor`）**在传入对象上跑完整构建** —— 用在 `Instantiate` 出来的克隆上做 `PROVIDER_PREVIEW` 验证。

⚠️ 本包工作区工程 `LuzzyMeow` 的 MA/NDMF 实际版本以 `Packages/vpm-manifest.json` 与 `packages-lock.json` 为准（分册 24 §8 有枚举与序列化写法的展开）。

## 3. 版本快照（时效性提醒）

> 这只是一个**发现辅助**，不是永久事实来源。

检查日 **2026-08-02**：

- 观察到的最新稳定版：**1.17.1**。
- 观察到的预发布版：**1.18.0-alpha.0**。

官方 release notes 里值得注意的近期变化：

- **1.17.0** 新增 **Floor Adjuster**。
- **1.17.0** 给 Bone Proxy 新增 **Match scale** 选项。
- **1.17.0** 在相关形态键选择器里新增**多选**。
- **1.17.0** 给 Parameters 新增 **VRCRaycast** 组件/参数支持。
- **1.17.1** 调整 Floor Adjuster 相对 **TexTransTool** 与部分 NDMF 插件的**执行顺序**。
- **1.18.0-alpha.0** 新增 **Blendshape Sync 的曲线重映射**，改进 **Mesh Cutter 性能**。

**在已安装包版本/源码确认之前，不要假设这些功能存在。** 预发布 API 与行为可能**不兼容地变更**。

## 4. 来源与权威顺序（source-policy）

**权威顺序（从高到低）**：

1. `Packages/manifest.json`
2. `Packages/packages-lock.json`
3. 已安装包的 `package.json`
4. 已安装包的 **C# 源码 / inspector / samples**
5. **与安装版本匹配**的官方文档
6. 官方 release notes / changelog
7. 社区材料（**仅作线索**）

包位置：`Packages/<pkg>/` 与 `Library/PackageCache/<id>@<version>/`。**不要编辑 PackageCache。**

版本声明模板（结论里要写）：

- Installed MA version:
- Documentation version consulted:
- Compatibility: `supported | unsupported | uncertain`

**不确定就不写**。

## 5. 官方资料清单（official-sources）

- MA 总入口：`https://modular-avatar.nadena.dev/`（intro / `references/reference` / changelog / FAQ / samples）。
- 仓库与发布：`https://github.com/bdunderscore/modular-avatar`、`https://github.com/bdunderscore/modular-avatar/releases`。
- 组件参考页（`references/reference/…`）：`merge-armature`、`bone-proxy`、`menu-item`、`menu-installer`、`menu-group`、`parameters`、`merge-animator`、
  `merge-blend-tree`、`blendshape-sync`、`reaction`、`reaction/shape-changer`、`reaction/mesh-cutter`、`reaction/material-swap`、`reaction/material-setter`、
  `reaction/debugger`、`mesh-settings`。
- 教程页：`tutorials/clothing`、`tutorials/menu`、`tutorials/advanced-toggles`、`distributing-prefabs/for-outfit-creators`。
- 另：opencode.ai 的 agent skills 文档两页（原文来源，与本包无关）。

⚠️ 文档页会**随版本变化**：引用前先确认页面描述的是**你安装的那个版本**。
## 6. Unity MCP 读写序列（unity-mcp-playbook）

### 6.1 只读发现（8 步）

1. 列出可用实例。
2. **选中正确的工程实例**。
3. 读活动场景与 dirty 状态。
4. 找 `VRCAvatarDescriptor`。
5. 查层级 / prefab 实例状态 / 组件。
6. 找 MA/NDMF 组件。
7. 读 Console。
8. 需要时截图。

### 6.2 安全写入（8 步）

1. 写明目标。
2. **尽量新建 wrapper**，不要在导入内容里改。
3. **一次一个连贯改动**。
4. 用**真实对象引用 / 序列化属性**写字段。
5. **复读**结果。
6. 标 dirty（**未获批准不保存**）。
7. 预览/校验 + 看 Console。
8. 需要时**先要批准**再保存 / 应用 prefab / 构建 / 上传。

### 6.3 交给 agent 的 prompt 模板硬要求

- **只用原生 `unityMCP_*` 工具**做 Unity 操作；
- **不得**用 Python / curl / PowerShell / 裸 JSON-RPC 打 MCP 端点；
- 先查**已安装的 MA 版本**与**确切目标**；
- **不许猜**类名 / 字段 / 路径 / GUID；
- 先只读，并**提交回滚方案等批准**。

### 6.4 兜底规则（通用工具太底层时）

生成**最小的 Editor 脚本**放到用户自己的工程目录 →

- 用 `Undo.RegisterCreatedObjectUndo` / `Undo.AddComponent` / `Undo.RecordObject`；
- 用**编译期引用或反射**解析组件类型；
- **字段缺失要大声失败**，绝不静默跳过；
- 交给 Unity 跑、看 Console，验证后**经批准删除临时脚本**；
- **脚本永远不许直接调 MCP 服务器**。

## 7. 安全与授权（safety-validation + evidence-and-authorization 的补充）

### 7.1 必须显式批准的动作（10 类）

删对象/资产；广域 Apply 或回滚 prefab 覆盖；改导入的源 prefab / 控制器 / 贴图 / 材质 / 网格；**只要求检查却要保存**；升包或升 Unity；破坏性网格/骨骼转换；发布/上传/覆盖 VRChat 头像；改 Blueprint ID 或 pipeline 设置；运行下载资产里的未知代码。

### 7.2 通常只读（6 项）

读取工程文件、读场景/层级/组件、读 Console、跑只读报告、比对版本、列实例与状态。

### 7.3 回滚方式（6 种）

Unity Undo；删掉新建的模块；移除新增组件；从版本控制恢复；**丢弃未应用的 prefab 覆盖**；恢复工程备份。

### 7.4 四类验证清单

- **对象/引用**：对象存在、引用解析、prefab 覆盖符合预期。
- **骨架/网格**：骨骼层级、SMR bones/rootBone、权重与形态键。
- **动画/菜单/参数**：层与 clip 绑定、菜单可达性、参数声明与默认值。
- **构建管线**：构建无新错误、生成结果符合预期。

### 7.5 证据标准（一句话）

**不要因为工具返回 `success` 就宣称成功** —— 要复读状态、跑预览/构建、**看 Console**。

### 7.6 与本包 `07-02-evidence-and-ownership.md` 的差异（只记增量）

证据阶梯（7 层）、状态词、授权矩阵、六类所有权边界、生成输出禁改 —— 本包既有分册**已经覆盖**，此处不重复。值得补的三点：

1. **「按结论强弱决定验证强度」对照**：字面序列化改动 → 静态 diff + 导入/编译；已解析组件/prefab → 选中的 Unity MCP 实例；合并后的菜单/参数/Animator 层/优化网格/生成组件 → **NDMF/provider 构建产物**；
  视觉/音频/手势/Contact/PhysBone/Blink/Lip Sync/视线/同步行为 → **具名授权的运行时层**；参数上限 → 优先用 SDK/provider API 而不是手算，并**把源码估算与最终构建成本分开写**；包体上限 → 用**当前那次确切构建**，
  不要从场景/FBX/贴图/结构相似的头像推断。
2. **MA/NDMF 所有权说明**：可编辑源 = 声明菜单项/参数/merge animator/bone proxy/骨架改动的**组件或源 prefab**；NDMF 在预览/构建期解析顺序、重映射、去重与生成名字，**合并克隆不是源组件的替代品**；
  改共享源 prefab 或菜单会影响**所有**消费者，做单头像改动前要先隔离源；源码级结论用 provider preview，最终合并结论用 `NDMF_BUILT`。
3. **失败与重试行为**：超时或响应不明时，**先查耐久证据**（编辑器状态、Console、源改动、场景 dirty、产物），**不要盲目重放**可能已经成功的写操作；**两次实质相同的失败后就停**，报告尝试的层、最后已证明的状态、以及继续所需的最小用户/环境动作。

## 8. 四个辅助脚本（纯只读）

### 8.1 `inspect-modular-avatar.ps1` / `inspect_modular_avatar.py`

用途：**只读**扫描一个 Unity 工程，报告 MA 与 NDMF 的**包解析情况**。

- 用法：`powershell -File inspect-modular-avatar.ps1 [-ProjectRoot <工程路径>] [-AsJson]`；`python inspect_modular_avatar.py [工程根]`（默认当前目录）。
- 先校验工程根（必须有 `Assets`、`Packages`、`ProjectSettings`，否则报「Not a Unity project root」）。
- 从 `ProjectSettings/ProjectVersion.txt` 首行的 `m_EditorVersion:` 取 Unity 版本。
- 在 `Packages/manifest.json` 与 `Packages/packages-lock.json` 的 `dependencies` 里用正则 `(?i)modular.?avatar` 找候选包（拿 `manifestSpec` / `resolvedVersion` / `source` / `depth`）。
- 解析两个可能位置：嵌入式 `Packages/<pkgId>` 与 `Library/PackageCache/<pkgId>@*`。
- 读各位置的 `package.json`：`name / displayName / version / unity`（异常时记 `error`）。
- 另列出 lock 里正则 `(?i)ndmf` 的包（`packageId / version / source`）。
- 输出 JSON：`{projectRoot, unityVersion, manifestPath, lockPath, modularAvatar:[…], ndmf:[…], notes:[…]}`。
- 固定 notes 三条：**不要修改 `Library/PackageCache`**；从定位到的安装源码解析**确切字段**；找不到包就核对 manifest/lock 里的包 ID。

### 8.2 `find-ma-symbol.ps1` / `find_ma_symbol.py`

用途：在**已定位的包目录**里搜符号，确认某个类/字段/枚举在本机版本里**是否真的存在**。

- 用法：`powershell -File find-ma-symbol.ps1 -PackageSource <包目录> -Symbol <符号>`；`python find_ma_symbol.py <包目录> <符号>`。
- ps1 用 `Select-String`（正则转义、忽略大小写）扫 `*.cs,*.md,*.json`；py 用 `re.escape` + 忽略大小写扫 `.cs/.md/.json/.uxml/.uss`（带 BOM 容忍）。
- 输出 `路径:行号: 行内容`；**找到返回 0，没找到返回 1**（可直接在脚本里当判据）。

### 8.3 在本包工作区怎么用

- 本包工具前缀是 `mcp__unity__*`；这两个脚本与 Unity 无关，**直接跑在工程目录**即可（改模工程：`<USERPROFILE>\AppData\Local\VRChatCreatorCompanion\VRChatProjects\LuzzyMeow`）。
- 拿到版本后再去读 `Packages/<pkg>/Editor/…` 的源码确认字段名，**不要凭记忆写 `Control.type` 之类的枚举值**。
- 脚本产物不要落在工程内；按工作区规范放 `<VRCHAT_ROOT>\tools\tmp\` 并**用完即删**。
