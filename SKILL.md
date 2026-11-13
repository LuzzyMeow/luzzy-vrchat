---
name: luzzy-vrchat
description: Use when working on a VRChat avatar in Unity — importing unitypackages and outfits from scratch, Modular Avatar menus/toggles/parameters,
VRCFury, PhysBone and hair collider tuning, VRCFaceTracking (Quest Pro / Pico), blendshape cover and clipping fixes,
headless Editor automation via flag files, Blender mesh cuts, MemoryOptimizer/Avatar Optimizer compression, bake to _Baked clones,
and VRChat SDK build/upload troubleshooting. Covers root-cause checklists for black face, clipping, black hole chest, three-white-eye,
missing menus and upload failures.
---

# luzzy-vrchat · VRChat Avatar 改模

面向 Unity 里的 VRChat avatar 改造：从零导入、装衣服和插件、配菜单参数、调物理、修穿模、bake、压缩、上传排查。

正文中文，代码和日志标记保持英文（Unity Editor.log 里中文常有编码问题，ASCII 标记更可靠）。

---

## 项目定位

先在当前工作区找 Unity 项目根目录。根目录必须同时包含 `Assets/`、`Packages/`、`ProjectSettings/`。

除非用户明确给出外部资源路径，**所有项目内路径都写相对路径**：`Assets/...`、`Packages/...`。不要在脚本或文档里写死盘符和用户目录。

### 路径占位符（唯一权威表）

本包出现的 `<...>` 都是占位符，**用前先解析，不要把字面量当路径**。全包只认这张表：

| 记号 | 含义 | 解析方式 |
|---|---|---|
| `<PROJECT_ROOT>` | 当前 Unity 项目根目录 | 工作区里同时含 `Assets/` `Packages/` `ProjectSettings/` 的那一层 |
| `<VRCHAT_ROOT>` | VRChat 工作区根（`projects/` `resources/` `tools/` 的父目录） | 读环境变量 `VRCHAT_ROOT`；**未设置就先问用户，不要猜盘符** |
| `<USERPROFILE>` | 当前 Windows 用户目录 | `%USERPROFILE%` |
| `<TOOL_ROOT>` | 工具集根目录（`.skills/` 的父目录） | 脚本可按自身位置推导；本包脚本在 `<TOOL_ROOT>\.skills\<skill>\scripts\` |
| `<UNITY_EXE>` | Unity 编辑器可执行文件 | 参考 `01-环境与工具链/01-01-environment-and-toolchain.md` 的本机实测表 |
| `<CACHE_ROOT>` | VRChat 磁盘缓存目录（可自定义，默认在 LocalLow 下） | 用户设置项；不要假定盘符 |

**硬规则**：文档与脚本只写占位符或工程内相对路径，**不写盘符、用户名、别人机器的目录**。正文里出现的具体路径（如 `C:\ProgramData\Unity\Unity_lic.ulf`、教程里的示例路径）只是「本机参考 / 系统固定位置」，**不是通用常量**，换机器要重新核对。

项目内目录约定：

| 记号 | 含义 |
|---|---|
| `<PROJECT_ROOT>` | 当前 Unity 项目根目录 |
| `Assets/_Gen/` | 脚本生成资产：bake 产物、controller、menu、parameters |
| `Assets/Editor/` | 项目专用 editor automation；优先复用已有脚本 |
| `_stage/` | 项目内日志、flag、报告目录 |

如果项目已经有自己的 stage/flag 目录，**从现有 editor 脚本里读出来，不要猜**。

---

## 六条铁律

### 1. MA 组件分两类：烘焙期才生效 / 编辑态就有反馈

**2026-11-01 按 MA 官方文档修正**：旧版写成「这 5 个组件一律在普通编辑器场景视图里不生效」，**范围过宽**。正确分两类：

- **烘焙期才生效（编辑态看不到效果）**：`ObjectToggle`、`MaterialSetter`、`MaterialSwap`、`MeshCutter`、`MergeAnimator` —— 构建期生成动画层或改网格，未 bake 的编辑器视图里确实看不到。
- **编辑态就有反馈（看到效果不是假象）**：
  - `ShapeChanger`：**效果在场景视图实时显示**（官方原文 *"The effect of the shape changer will be shown in the scene view in real time"*），还可以开 scene view 的 **overdraw** 调试叠加透看衣服下面。
  - `BlendshapeSync`：**编辑模式就把基础网格的形态键值复制到目标对象**（*"In edit mode, it copies the blendshape values from the base automatically"*），改基础形态键立刻能看到目标跟随。
  - 出处：<https://modular-avatar.nadena.dev/docs/reference/reaction/shape-changer>、<https://modular-avatar.nadena.dev/docs/reference/blendshape-sync>；逐组件字段与限制见 [references/04-外观与表情/04-06-ma-reactive-components.md](references/04-外观与表情/04-06-ma-reactive-components.md) §9。

第一类组件仍然只在三个时机应用：

- 进入 Play 模式
- `Tools/Modular Avatar/Manual bake avatar`（编辑态 bake）
- VRChat 上传时

**后果**：第一类组件在场景视图里看到的穿模、乳头、妆容、材质错误，大概率是假象。验证任何视觉问题前先 bake：

```csharp
EditorApplication.ExecuteMenuItem("Tools/Modular Avatar/Manual bake avatar");
// 需要先选中 avatar，会生成 <Avatar>_Baked 副本 → 渲染验证 → 删掉副本
```

**不要对着没 bake 的编辑器视图瞎改。**

### 2. 不要靠缩身体来修穿模

衣服是按**特定身体形态**雕的。缩身体会让衣服浮起来、露缝，比原来的穿模更难看。

正确做法是找到衣服 `BlendshapeSync` 指向的那个身体 blendshape，然后把它激活。详见 [references/00-基底/00-05-mesh-blendshape.md](references/00-基底/00-05-mesh-blendshape.md)。

同理：不要靠隐藏手脚、删 mesh 来"修穿模"。优先保留作者原始逻辑，去排查材质、形态键、MA 组件、PhysBone/collider、constraint、菜单状态。

### 3. 上传是用户动作，不能脚本化

`VRChat SDK → Build & Publish` 需要 VRChat 登录，编辑器脚本无法自动完成。所有自动化只能产出一个**可上传的场景/prefab 状态**，最后一步交给用户点。

### 4. 实现方式优先用 Modular Avatar（MA），尽量不用 VRCFury

**用户 2026-10-17 拍板：头像上的开关 / 菜单 / 换装 / 参数 / 动画合并，一律优先用 MA（+ NDMF）实现；VRCFury 只在 MA 没有对应件、或资产作者只提供 VRCFury 组件时才用，并且要说明理由。**

理由：MA 是**声明式**的 —— 组件本身就是真源，构建时统一 bake，可读、可回滚、能被 NDMF 预览；VRCFury 走自己的构建管线，在构建期改写控制器与资产，排查和回滚都更麻烦，也违背下面「尊重真源」的原则。

- 拿到**带 VRCFury 组件**的 prefab：先找 MA 等价件替换 —— Toggle / Armature Link / Blendshape Link / Full Controller → MA 的 `Object Toggle` / `Merge Armature` / 
  `Blendshape Sync` / `Merge Animator`；换完删掉 VRCFury 组件，再确认 NDMF 构建通过。
- 第三方资产**只提供 VRCFury 组件**、且无法等价替换时：保留并记录，不要硬拆。
- 组件级决策表见 [references/02-工程组织与工作流/02-03-ma-component-guide-workflows.md](references/02-工程组织与工作流/02-03-ma-component-guide-workflows.md)；MA 化习惯与替换纪律见 [references/02-工程组织与工作流/02-02-ma-workflow-h
  abits.md](references/02-工程组织与工作流/02-02-ma-workflow-habits.md)。

### 5. 改完先在 Gesture Manager 里模拟游戏环境验收（新工程必装）

**用户 2026-10-17 强调：Gesture Manager 必须装、必须用 —— 新建 / 重建工程后第一件事之一就是核实它在不在（`Packages/vpm-manifest.json` 里有 `vrchat.blackstartx.gesture-manager`），
之后每轮改完都在它的 Play 模式里过一遍。** 它在编辑器内模拟 VRChat 运行时（VRC3 模块 + Avatar Dynamics），能验收手势→参数映射、FX 层合并、菜单与开关档位、表情滑条、默认外观、PhysBone / 动骨跟随 ——
**比反复 Build & Test 快一个数量级**；而 MA 组件本来只在 Bake / Play / 上传生效（铁律 1），Gesture Manager 的 Play 正是能立刻看到 MA 效果的地方。

**但它是编辑器模拟器，不是客户端**：不证明多人同步、远端可见性、Contact 交互、saved 跨世界持久化 —— 这些必须进游戏验证，没跑就写 `NOT_RUN`。

- 打开：顶部菜单 **`Tools → Gesture Manager Emulator`** → 场景里生成 `Gesture Manager` 对象 → 在它的 Inspector 点 `Enter Play Mode` → Game 窗口里试手势 / 菜单 / 参数；
  `Tools → Test Animation` 可拖 `Idle / Walk / Run / Lie down` 验证动骨与衣服是否跟随（**不跟着动 = 骨骼没绑对**）。
- 安装、截图与验收细节见 [references/01-环境与工具链/01-06-project-launch-and-mcp-bridge.md](references/01-环境与工具链/01-06-project-launch-and-mcp-bridge.md) §2.7；
  验收矩阵见 [references/07-验证与排查/07-01-bake-and-validation.md](references/07-验证与排查/07-01-bake-and-validation.md) §8。

**用户 2026-10-19 加严（模拟轮盘测试，最高优先级）：所有 VRChat 工程 / 模型改模任务收尾，默认必须实际进 Gesture Manager Play，逐项点击本次改模实现的每一个轮盘菜单功能键（新工程 / 新模型 = 全部菜单），并用截图 + 视觉理解确认；
只看 Unity Console / 日志不算通过。** 用户说「进行模拟测试 / 模拟测试 / 验收一下」时更不用说；只有用户明确说「这次不用测试」才可跳过，并在汇报里写 `TEST_SKIPPED_BY_USER`。

- **范围**：新工程 / 新模型 = 全部轮盘菜单（一级、二级、Button / Toggle / SubMenu / Radial / Puppet / 参数 / 手势 / 表情）；普通改模 = 本次新增 / 修改 / 删除的每个菜单项、开关、参数、手势、表情、PhysBone 交互。**不能抽样。**
- **证据**：每项都在 Gesture Manager 的菜单 / 轮盘里实际点，截图（`mcp__windows__*` 或 Unity 相机）→ `read_image` 看图：状态真的变了（开 / 关、换装、发型 / 颜色、道具显隐），没有粉材质 / missing script / 穿模 / 模型消失 / T-pose / 菜单错位 / 功能串位。
- **记录**：每个功能一行「菜单路径 / 功能 / 操作 / 预期 / 实际截图 / PASS-FAIL」；失败先修再重测，不能把失败留到汇报；Gesture Manager 测不了的多人同步 / 远端可见性 / Contact / saved 标 `NOT_RUN（需进游戏）`。
- **汇报**：测试总数 / 通过 / 失败 / 未跑 + 截图路径 + 失败项修复记录；没有截图证据不算测试完成。

### 6. 新增/转换模型、衣服、道具：必须实际绑骨，并做移动态菜单验收

**用户 2026-10-19 强调：不管是衣服模型间转换，还是单独新增模型 / 道具，都要补骨骼并实际绑到目标 avatar 骨架上；只复制 mesh / prefab 不绑骨 = 穿不上、用不了。**

- **绑骨**：`SkinnedMeshRenderer.bones` / `rootBone` 必须指向目标骨架；跨模型按骨骼**基础名**映射（去掉 `$GUID` 后缀），找不到同名骨时挂到最近父骨；衣服、配件、道具都要检查。
- **移动态验收**：Gesture Manager Play → 打开菜单 → 开关该衣服 / 道具 → **用 Gesture Manager 自带的 Movement / 模拟移动切 `Walk` / `Run`**（也可 `Tools → Test Animation` 拖 `Walk` / 
  `Run`）→ 截图对比 Idle 与移动状态。必须看到网格跟着骨骼动：不拉伸、不留在原地、不脱离身体。
- **判据**：静态截图正常、移动时漂移/留在原地 = 绑骨失败；只看 Console / 静态截图不算通过。
- **坑**：不要用 `PlayCustomAnimation(proxy_walk_forward)` 验移动 —— 它会替换 Animator controller，FX / MA toggle 全部失效；用 Gesture Manager 的 Movement，或直接摆 Hips/腿骨姿态。

---


## 工作原则

- **先读项目，再动手**。看 `ProjectSettings/ProjectVersion.txt`、`Packages/manifest.json`、`Assets/Editor/`、已有 scene/prefab、SDK 和插件版本。
- **实现方式 MA 优先**：开关 / 菜单 / 换装 / 参数 / 动画合并一律用 Modular Avatar（+ NDMF），**尽量不用 VRCFury**；只有 MA 没有对应件时才用（见铁律 4）。
- **改完必须逐项跑模拟轮盘测试**：新工程先核实 Gesture Manager 已装，`Tools → Gesture Manager Emulator` → `Enter Play Mode`；本次改模实现的每个轮盘菜单功能键都要实际点，截图 + 视觉确认，
  不能只看 Console / 日志（见铁律 5）。**衣服 / 模型 / 道具还必须绑骨，并在菜单打开后切 Walk / Run 移动态验跟骨（见铁律 6）。**
- **开工程走 ALCOM 的「打开 UNITY」**（不要自己拼 `Unity.exe` 命令行；版本不一致时选「继续」，不升级）。
- **用 AI 干活之前先确认 MCP 桥是 stdio**（`127.0.0.1:6400`）：Unity 默认走 HTTP（8080），会出现「Unity 显示已连接、工具却报 `No Unity Editor instances found`」。判据、
  四种修法与桥断自救见 [references/01-环境与工具链/01-06-project-launch-and-mcp-bridge.md](references/01-环境与工具链/01-06-project-launch-and-mcp-bridge.md)。
- **能脚本化就脚本化**，不要手点 Unity 做可重复的事。项目已有 flag workflow 就用 flag workflow。
- **不要杀 Unity 或中断上传**，除非用户明确要求。
- **不要二次压缩**已经被 MemoryOptimizer / Avatar Optimizer 处理过的 baked avatar。需要重压就从干净源 avatar 重新 bake。
- **上传前保留 blueprint ID**，除非用户明确要创建新 avatar。
- **危险操作前先备份**。改材质、mesh、场景结构之前，先克隆资产或存快照。

---

## 从零导入到上传

完整流程按这个顺序：

1. 定位 Unity 项目根目录，确认 Unity 版本和当前场景。
2. 建立可回滚点：看 git 状态；没有 git 就至少记录将要改动的 `Assets/` 路径和生成目录。
3. 收集资源清单：base avatar、衣服、头发、shader、VRChat SDK、**Modular Avatar（首选实现手段）**、**Gesture Manager（必装：模拟游戏环境验收，见铁律 5）**、lilToon/Poiyomi、互动插件等依赖；**VRCFury 只在 MA 无对应件时才装**（铁律 4）。
4. **按依赖顺序导入**（见下节），每导入一组等 Unity 编译完，修完 Console compile error 再继续。
5. 把 avatar prefab 放进工作场景，配置 `VRCAvatarDescriptor`、view position、lip sync、eye look、playable layers、expression menu/parameters。
6. 装衣服、头发、互动系统和插件。**优先用作者官方 installer**；手写修复时保留作者的 controller、contacts、PhysBones、shape keys 和 material slots。
7. 整理菜单和参数：每页不超过 8 个 control；远端可见的功能才 network synced；本地 UI、reset、灯光 handle、临时状态尽量 local-only / saved=false。
8. Unity 内审计：renderer 是否存在且 enabled、材质是否缺失、shape key 是否异常非零、MA/VRCFury/constraint 是否引用空对象、插件的 menu/animator/parameter 链路是否完整。
9. Bake 生成上传用 clone。生成资产**持久化到 `Assets/_Gen/`**，不要让场景长期依赖内存对象或容易被覆盖的公共 generated 目录。
10. 优化压缩：检查 synced parameter bit budget、texture/mesh/material 性能、PhysBone 数量。压缩后**重新审计**菜单、材质、脸、衣服、互动系统。
11. SDK build 前预检：descriptor、PipelineManager/blueprint ID、expression params、FX controller、avatar validation。
12. 上传并监控日志直到成功或明确失败。
13. 上传后进游戏验证：脸、衣服、裸身、材质、菜单、互动系统、头发动态、性能评级。

### 导入顺序

从零导入资源时按这个顺序，避免 missing script 和材质丢失：

```text
1. Unity / VRChat 项目基础环境
2. VRChat SDK
3. shader 包：lilToon、Poiyomi、作者指定 shader
4. 通用工具：**Modular Avatar（首选）**、**Gesture Manager（必装，模拟游戏环境验收）**、Avatar Optimizer / MemoryOptimizer（VRCFury 列为备选）
5. base avatar
6. 服装、头发、配件、材质贴图
7. 互动系统：PCS、SPS、PCSS、Grab Full Body 等
8. 项目专用 editor automation 和生成脚本
```

导入 `.unitypackage` **用 Unity 自己的导入机制**，不要直接解压覆盖 `Assets/`。导入前看作者 README / Booth 说明，确认依赖版本和安装顺序。

**导入模型后：先保留 prefab 实例，按需再解压（2026-10-19 修订，别默认拆）。**

- 模型拖进场景后**不要急着** `Unpack`：保留预制件实例时，实例上的改动（参数 / 材质 / Transform override）都能 `Prefab → Revert` 或 `Apply` 回退，也能继续吃到作者对 prefab 的更新。
- **一旦「完全解压」（Unpack Prefab Completely）就失去这些回退能力**；只有要给实例的**子物体重定父级到 prefab 之外**（例如把衣服 / 配件 / MA 菜单容器挂到自建的 `Clothes/`、`Deco/` 节点下）
  时 Unity 才会拒绝并静默失败（Console: `Setting the parent of a transform which resides in a Prefab instance is not possible`）——
  此时才用 `PrefabUtility.UnpackPrefabInstance(root, PrefabUnpackMode.Completely, InteractionMode.AutomatedAction)`。
- **更稳的替代**：`Ctrl+D` 生成副本，在**副本**上解压 / 改，原件实例保留可回退；或改用 VRCFury / MA 自带方式。**只解压场景里的实例，源预制件资产 `Assets/<包>/.../*.prefab` 保持不动**。
- 详见分册 [01-06 §2.8](references/01-环境与工具链/01-06-project-launch-and-mcp-bridge.md) 与 [07-05 §1.4](references/07-验证与排查/07-05-ma-gotchas-version-toolchain.md)。

---

## 自动化入口

先搜项目已有的自动化：

```powershell
rg -n "run\.flag|MenuItem|BuildAndUpload|BuildPipeline|MemoryOptimizer|ModularAvatar|VRCFury" Assets/Editor
```

发现 project-specific flag workflow 时：

- 从源码读出 stage 目录和命令名，不要猜。
- 写入一条命令后等 done/error 日志再发下一条。
- **不要并发**发送 scene-changing / build / upload 命令。

没有自动化时，可以新增最小的 `Assets/Editor/<Project>AvatarTools.cs`，只封装可重复动作：导入检查、菜单审计、bake、参数统计、上传前验证。

写法和踩坑见 [references/00-基底/00-01-editor-automation.md](references/00-基底/00-01-editor-automation.md)。

---

## 验证清单

任何一轮改动完成后至少检查：

- 场景里源 avatar 和上传用 avatar 都能找到。
- `VRCAvatarDescriptor`、menu、parameters、FX controller 引用不为空。
- 关键 renderer（脸、身体、衣服、头发）都 enabled 且材质正确。
- 全脱 / 半脱 / 穿衣 / 换发 / 换发色 / 换材质 / 互动衣服，各状态互斥正常。
- 互动插件菜单存在，参数预算合理，效果落在目标身体部位。
- PhysBone / collider 引用完整，动态头发和衣服不明显穿模。
- **Bake 或 optimizer 之后把上面重跑一遍**。
- SDK validation 通过。

---

## 分册导航（references/）

`references/` 分两大类：**00-基底**（外部 MIT 原文保留 + 官方补充节）、**01–07 主题分册**（本工作区自产，含原「症状速查」全部转化而来的结构化知识）。**「08-症状速查」栏目已于 2026-11-11 取消**：全部症状条目已转化为结构化知识并入对应主题分册；**以后的增量更新也直接写成结构化知识**（见下文「知识库自身的更新规则」）。先用下面的分类表定方向，再用「症状 → 结构化知识路由 / 常用入口」定位到具体的册与节。

**命名约定（2026-11-11 统一）**：全部知识文件为 `NN-MM-<slug>.md` —— `NN`=分类号（与目录一致）、`MM`=两位册序、`slug`=英文 kebab-case 主题、后缀统一 `.md`。旧文件名与新名的映射见 [ACKNOWLEDGEMENTS.md](ACKNOWLEDGEMENTS.md) 的「旧名映射」节；正文交叉引用一律使用新编号。

### 基底 7 册（references/00-基底/）

| 文件 | 里面有什么 |
|---|---|
| [references/00-基底/00-01-editor-automation.md](references/00-基底/00-01-editor-automation.md) | flag 文件 + `[DidReloadScripts]` 一次性执行器、PowerShell 强制重编译、编译验证、离屏渲染 PNG、`ExecuteMenuItem`、焦点抢占排查 |
| [references/00-基底/00-02-modular-avatar.md](references/00-基底/00-02-modular-avatar.md) | MA 全组件代码范式（MergeAnimator / Parameters / MenuItem / MaterialSetter / ShapeChanger / ObjectToggle / BlendshapeSync）、服装电台 int 参数 gating、菜单和参数预算 |
| [references/00-基底/00-03-physbone-dynamics.md](references/00-基底/00-03-physbone-dynamics.md) | 反射访问 VRCPhysBone、序列化字段表、实测调参配方、抓取不回弹、头发碰撞、动画驱动 PB 属性 |
| [references/00-基底/00-04-face-tracking.md](references/00-基底/00-04-face-tracking.md) | VRCFT 归一化原理和校准、Pico Connect 设置、面捕套件安装（prefab 型 / 向导型）、`.anim` YAML 编辑陷阱 |
| [references/00-基底/00-05-mesh-blendshape.md](references/00-基底/00-05-mesh-blendshape.md) | 穿模的正确解法、blendshape 遮盖系统、Blender headless bisect / shrinkwrap、globalScale 陷阱、fitdump 诊断法 |
| [references/00-基底/00-06-plugin-ecosystem.md](references/00-基底/00-06-plugin-ecosystem.md) | 互动插件（PCS / SPS / PCSS / Grab Full Body）结构与排查、互动衣服、VRCFury、bake 与 optimizer 原则 |
| [references/00-基底/00-07-troubleshooting.md](references/00-基底/00-07-troubleshooting.md) | 症状 → 根因清单：黑脸、黑洞、穿模、三白眼、菜单丢失、材质异常、上传失败 |

**基底这 7 册的外部 MIT 原文保留**；2026-11-11 起每册末尾追加「补充」节（联网核实的官方口径：PhysBones / VRCFT 眼睑 / lilToon 等，以及到结构化分册的路由），不再宣称「一字不改」。**组件字段级语义**（默认值、限制、烘焙期改写、组件间交互）在 2026-11-01 按 MA 官方文档 67 页补齐，另立 4 册：
[02-10 通用机制/Setup Outfit](references/02-工程组织与工作流/02-10-ma-setup-outfit-and-general-behavior.md)、
[03-07 菜单参数动画器/MMD](references/03-菜单与参数/03-07-ma-platform-sync-mmd-and-motion.md)、
[04-06 反应式组件](references/04-外观与表情/04-06-ma-reactive-components.md)、
[05-10 骨骼与几何组件](references/05-动骨与穿模/05-10-ma-bone-and-geometry-components.md)。
注：基底 `00-05-mesh-blendshape.md` 里「MA 组件一律在编辑器不生效」的同义表述按外部 MIT 原文保留，**以铁律 1 的两类划分口径为准**。

### 主题分册（references/01–07，共 59 册；全包 66 册）

为补齐基底缺口写的内容全部放在 `references/` 下**按主题分类的子目录**里。来源工程的原文保存在外部路径 `<VRCHAT_ROOT>/tools/sources/vrchat-avatar-modding/upstream/`（**包外，不随本包分发，可能不存在；缺失就跳过，不影响阅读**），**不复制进来**。

| 分类 | 册数 | 里面有什么 |
|---|---|---|
| [references/01-环境与工具链/](references/01-环境与工具链/) | 6 | 装环境与版本对齐、国内镜像源、SDK 面板逐字段、Unity 基础操作、素材导入判据、建工程与 AI(MCP) 桥接、**SDK 3.10.6 成对升级与 VPM 镜像取包**（01-01 §8）、**MCP 瞬时失效/flag-file 兜底/环境前提脚本**（01-06 §13） |
| [references/02-工程组织与工作流/](references/02-工程组织与工作流/) | 11 | 工作场景与工作资产、MA 化流程与操作习惯、MA 组件决策与任务工作流、素材库分类与重组、外部教程笔记与 KB 比对、Assets 重新分类整理、VRCFury Write Defaults 弹窗、MA 通用机制/Setup Outfit、**互动插件集成：RK Player / RealKiss / GUN1 道具接线**（02-11） |
| [references/03-菜单与参数/](references/03-菜单与参数/) | 7 | 菜单/参数审计与预算、**EP 权威规则（ValueType/位宽 API/saved/内置参数撞名/类型校验）**（03-01 §10）、MA 开关与轮盘配方、动画控制器与变量、反射批量建菜单、**控件语义（Button 脉冲/RadialPuppet/label 去重）与一键穿衣混搭补全（权威写者/互斥矩阵/Rest Pose）**（03-06 §18–§19）、MA 平台同步/MMD |
| [references/04-外观与表情/](references/04-外观与表情/) | 9 | lilToon 外观属性真伪（含官方变体/槽位补充）、表情曲线与图标、颜色档位与 Radial Puppet、狼脸手势表情、MA 反应式组件全解（**含 OT 极性语义全解 §14**）、VRCFT 参数前缀修复、**OSC 面捕接入与修复**（04-08）、**原生 Face Tracking（SDK 3.10.6+）**（04-09） |
| [references/05-动骨与穿模/](references/05-动骨与穿模/) | 11 | 服装接线与几何验收（**袜子 x108xz / 眼罩 PB 门控 / 约束整定** §12–§15）、PhysBone 全参数与调试循环、模型解剖与换头、跨模型配件迁移、换头拆除与克隆、SPS 绑定、尾巴/手套迁移、跨头像衣装适配（NewFit）、MA 骨骼与几何 17 组件（**道具尺度反推/约束量测 §22–§23**）、**蒙皮绑定与几何修复（bindposes/权重/法线/量测）**（05-11） |
| [references/06-构建上传与备份/](references/06-构建上传与备份/) | 6 | SDK 构建上传与版本管理、备份与打包迁移、从已上传模型恢复工程、反射出包与反射上传、服务端多包并存与 PUT 对齐（**先 security 后 standard** §11）、**上传凭据生命周期与客户端侧取证**（06-06） |
| [references/07-验证与排查/](references/07-验证与排查/) | 9 | 烘焙与骨骼验收、证据阶梯与所有权、Unity MCP 纪律、provider playbook、常见坑与版本实测、GM 自动化模拟测试（**构造签名/组合测试/Action 层/Contact 模拟 §23**）、场景骨骼污染还原、多角度独立验证五件套、**动态验证边界与证据配方（Play 十条边界/FX+Update/四跳/渲染刷新/MCP 兜底）**（07-09） |

**2026-11-11 结构化重组**：原 `08-症状速查` 9 册（第 123–177 轮 + 主表）全部取消，内容按主题并入上表各册；新增 6 册（02-11 / 04-08 / 04-09 / 05-11 / 06-06 / 07-09），其余以追加小节方式并入现有册（各节带「原 08-症状速查 第 N 轮」来源标注）。**2026-11-01（第 135 轮）的 MA 官方文档对齐 4 册**（02-10 / 03-07 / 04-06 / 05-10）保持不变。

**逐册说明、来源映射、旧名映射与基底的对应关系**见 [references/README.md](references/README.md)。

## 症状 → 结构化知识路由（原「症状速查」，2026-11-11 起并入主题分册）

**查法**：`rg -n "<症状关键词>" references/` 定位到册 + 行号，只读命中附近 30–40 行；想不出关键词就按下表定方向。所有条目都带「本工作区实测 YYYY-MM-DD」与判据；带 ⚠️/「取代」标注的小节以最新口径为准。

| 症状大类 | 去哪册 |
|---|---|
| 黑脸 / 粉材质 / 贴图不见 / 高光色斑 | [00-07](references/00-基底/00-07-troubleshooting.md)（含路由表）；同名 shader 覆盖 → [07-01 §33](references/07-验证与排查/07-01-bake-and-validation.md)；多色板错配 → [05-08 §15](references/05-动骨与穿模/05-08-cross-avatar-garment-fit-newfit.md) |
| 穿模 / 黑洞 / 缺件 / 袜子膝穿模 / 手套露指 / 口袋陷入 | [00-05](references/00-基底/00-05-mesh-blendshape.md)、[05-01 §12–§15](references/05-动骨与穿模/05-01-outfit-fit-and-clipping.md)、[05-11](references/05-动骨与穿模/05-11-skinning-binding-and-geometry-repair.md)、[05-08 §13](references/05-动骨与穿模/05-08-cross-avatar-garment-fit-newfit.md) |
| 菜单不出现 / 开关没反应 / 默认态错 / 一键穿衣 / 混搭 / Rest Pose 裸体 | [03-06](references/03-菜单与参数/03-06-ma-menu-build-by-reflection.md)（§18–§19）、[04-06 §12–§14](references/04-外观与表情/04-06-ma-reactive-components.md)、[07-01 §14/§15/§20/§31/§32](references/07-验证与排查/07-01-bake-and-validation.md) |
| 参数超限 / duplicate definition（IsLocal/AFK）/ saved 不记忆 / 默认值写不进 | [03-01 §8–§10](references/03-菜单与参数/03-01-parameter-and-menu-audit.md) |
| 面捕不动 / 闭眼 / 瞳孔不随视线 / 高光漂移 / AAO 删键 / 补丁器失败 | [04-08](references/04-外观与表情/04-08-face-tracking-osc-and-repair.md)（OSC）、[04-09](references/04-外观与表情/04-09-face-tracking-native-sdk.md)（原生）、[04-07](references/04-外观与表情/04-07-vrcft-parameter-prefix-repair.md)（前缀丢失） |
| 表情/手势/口型不生效 / 眨眼覆盖 / 默认表情 | [04-05](references/04-外观与表情/04-05-wolf-face-gesture-expressions.md)、[04-02](references/04-外观与表情/04-02-expressions-and-icons.md)、[04-08 §3](references/04-外观与表情/04-08-face-tracking-osc-and-repair.md) |
| 上传失败 / 上传成功但游戏是旧的 / 凭据失效 / 客户端取证 | [06-05](references/06-构建上传与备份/06-05-avatar-package-version-collision.md)、[06-06](references/06-构建上传与备份/06-06-upload-credentials-and-client-evidence.md)、[06-04](references/06-构建上传与备份/06-04-sdk-publish-with-reflection.md)、[07-01 §30/§35](references/07-验证与排查/07-01-bake-and-validation.md) |
| 烘焙后功能消失 / Play 验证不动 / GM 跑不起来 / 截图取证 | [07-01](references/07-验证与排查/07-01-bake-and-validation.md)、[07-09](references/07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md)、[07-06](references/07-验证与排查/07-06-gesture-manager-automated-sim-test.md) |
| 道具浮空 / 尺度不对 / 握持判据 / 约束量测荒谬值 | [05-10 §22–§23](references/05-动骨与穿模/05-10-ma-bone-and-geometry-components.md)、[05-09](references/05-动骨与穿模/05-09-third-party-tool-prefab-binding-abt.md)、[05-11](references/05-动骨与穿模/05-11-skinning-binding-and-geometry-repair.md) |
| RK Player 没声音 / CD 封面 / RealKiss 无反应 / GUN1 反相 | [02-11](references/02-工程组织与工作流/02-11-plugin-integration-rk-realkiss-props.md) |
| MCP 连不上 / 工具瞬时失效 / 编译卡死 / 模态框阻塞 | [01-06](references/01-环境与工具链/01-06-project-launch-and-mcp-bridge.md)、[07-09 §8](references/07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md) |
| 场景骨骼整体错位 / 工程恢复 / 逆向 | [07-07](references/07-验证与排查/07-07-scene-bone-corruption-recovery.md)、[06-03](references/06-构建上传与备份/06-03-recover-project-from-uploaded-avatar.md) |

**新知识往哪写**：直接写成结构化知识并入 `references/01–07` 对号的主题册（症状 → 机制/根因 → 正确做法 → 判据 → 来源与实测日期）；新主题就在对应分类下新建 `NN-MM-<slug>.md`（MM 取该分类最大册序 +1），并同步更新本文件导航与 `references/README.md` 索引。**不再按时间开「症状增补册」，也不再往 SKILL.md 堆症状行。**

## 常用入口（按处境查）

| 你的处境 | 去读 |
|---|---|
| 刚接手一个工程 / 不确定工具链对不对 | [环境与工具链](references/01-环境与工具链/01-01-environment-and-toolchain.md) |
| 新工程建好后要跑起来测试 / 改完想快速验收（不想反复 Build & Test） | [Gesture Manager 必装与用法](references/01-环境与工具链/01-06-project-launch-and-mcp-bridge.md) §2.7 |
| 要新建 / 克隆工程、第一次打开别人的工程、AI 连不上 Unity | [建工程、开工程与 AI 桥接](references/01-环境与工具链/01-06-project-launch-and-mcp-bridge.md) |
| 从零装环境、换国内镜像源、建项目、用 SDK 面板逐字段上传、补 Unity 基础操作 | [环境与 SDK](references/01-环境与工具链/01-02-environment-and-sdk.md) |
| 上传报错、装环境细节、SDK 面板某项看不懂 | [上传报错与工具链](references/01-环境与工具链/01-03-upload-errors-and-toolchain.md) |
| 认不清 Unity 界面图标、判断下载物进 `Assets` 还是 `Packages`、变量同步范围 | [Unity 界面与周边工具](references/01-环境与工具链/01-04-unity-ui-and-extra-tools.md) |
| 从零装包 / 整理素材 / 上传前自查 / 性能等级太差 | [工具链与上传](references/01-环境与工具链/01-05-babystep-toolchain-and-upload.md) |
| 要建工作场景、挂第一件衣服、或写编辑器脚本 | [工作场景与工作资产](references/02-工程组织与工作流/02-01-workspace-and-work-assets.md) |
| 素材多、想自动匹配 BOOTH 商品编号/名称、做只读封面图库、合并重复包装 | [VRCAL 素材库自动匹配](references/02-工程组织与工作流/02-07-vrcal-asset-library.md) |
| 要把存量头像 MA 化 / 加新套装发型 / 参数互斥与命名习惯 | [MA 化流程与习惯](references/02-工程组织与工作流/02-02-ma-workflow-habits.md) |
| 不知道该用哪个 MA 组件 / 每条任务先检查什么 | [MA 组件决策与工作流](references/02-工程组织与工作流/02-03-ma-component-guide-workflows.md) |
| 菜单放不下、参数算不清、要审计一遍 | [参数与菜单审计](references/03-菜单与参数/03-01-parameter-and-menu-audit.md) |
| 做换装菜单（手搓 Animator Controller/变量 vs MA 快捷 Toggle）、配粒子特效、绑骨踩坑 | [动画控制器与菜单](references/03-菜单与参数/03-02-animator-menu-particles.md) |
| 走完整实战流程、绑骨该加哪个组件、**部位莫名消失 / 开关失效** | [实战与 SDK 面板](references/03-菜单与参数/03-03-case-study-and-sdk-panel.md) |
| 用 MA 做开关 / 换色 / 轮盘、要改别人模型自带的菜单 | [MA 逐组件配方](references/03-菜单与参数/03-04-ma-menu-and-switch-cookbook.md) |
| 调亮度、换色、发光、单色照明 | [lilToon 外观](references/04-外观与表情/04-01-appearance-and-liltoon.md) |
| 修表情曲线 / 生成预览图 / 换菜单图标 / 参数存不存 | [表情曲线与图标](references/04-外观与表情/04-02-expressions-and-icons.md) |
| 加颜色档位或连续滑条 / 调头发层次高光 | [颜色档位与 Puppet](references/04-外观与表情/04-03-color-and-puppet.md) |
| 装衣服、判可移植性、修穿模 | [服装接线与穿模](references/05-动骨与穿模/05-01-outfit-fit-and-clipping.md) |
| 想把 A 模型衣服穿到 B 模型 / 问 NewFit、DressingTools、AvatarDresser 怎么选 | [跨模型衣服适配](references/02-工程组织与工作流/02-08-cross-avatar-clothing.md) |
| 删骨骼删参数、如梦衣柜换装、录动画换色、从 0 做动骨、Blender 改网格后替换网格 | [动骨与 fbx 替换网格](references/05-动骨与穿模/05-02-bind-animation-physbone-fbx.md) |
| 要换头 / 换身体 / 替换某个部位网格、判断「头是不是独立网格」、量比例与脖子接缝 | [模型解剖与换头](references/05-动骨与穿模/05-03-model-anatomy-and-head-swap.md) |
| 要整理素材库、判断素材该进哪个目录、去重与 junction | [素材库分类与重组](references/02-工程组织与工作流/02-06-resource-library-reorg.md) |
| 接手别人的 Unity 工程、要把 `Assets` 里混放的素材按用途重新分类，又怕断引用 / 插件失效 | [工程内 Assets 资产整理](references/02-工程组织与工作流/02-05-unity-project-asset-reorg.md) |
| 要做缩略图 / 改 blueprintId / 判断 SDK 警告该不该消 | [构建上传与版本管理](references/06-构建上传与备份/06-01-sdk-upload-and-versioning.md) |
| 要备份 / 导出 unitypackage / 迁移工程 | [备份与打包](references/06-构建上传与备份/06-02-backup-and-package.md) |
| 要 bake / 排查「明明配了却不生效」/ 做验收 | [烘焙与验收](references/07-验证与排查/07-01-bake-and-validation.md) |
| 要解释结论有多可信 / 判断某文件该不该改 | [证据阶梯与所有权](references/07-验证与排查/07-02-evidence-and-ownership.md) |
| 要走 MCP 动 Unity / 操作超时了 / 怀疑没生效 | [Unity MCP 调用纪律](references/07-验证与排查/07-04-verification-discipline.md) |
| 用 FaceEmo / lilycalInventory / DressingTools / Gesture Manager / Avatar Optimizer | [provider playbook](references/07-验证与排查/07-03-provider-playbooks.md) |
| 拿到整套服装 prefab 要「Setup Outfit」/ 判断某套装能否被 MA 直接处理 / 要手动烘焙看产物 / 要对外分发 | [MA 通用机制与 Setup Outfit](references/02-工程组织与工作流/02-10-ma-setup-outfit-and-general-behavior.md) |
| 多个 MergeAnimator 层序打架 / 要用 `VRCAnimatorLayerControl` / PC·Quest 双平台 / 头像进 MMD 世界 | [MA 菜单参数动画器补充](references/03-菜单与参数/03-07-ma-platform-sync-mmd-and-motion.md) |
| 做换装开关·材质变体·形态键联动·按遮罩裁网格 / 菜单点了没反应 / 两个开关抢同一对象 | [MA 反应式组件全解](references/04-外观与表情/04-06-ma-reactive-components.md) |
| **换装切换后旧套装部件残留（默认态对、切套后错）/ 反相 OT 改了语义但 FX driver 没同步** | [MA 反应式组件 §13](references/04-外观与表情/04-06-ma-reactive-components.md)、[反射建菜单 §19](references/03-菜单与参数/03-06-ma-menu-build-by-reflection.md) |
| **单个开关参数已为 1、对象却永远不显示（同组其它开关正常）/ 怀疑 `ObjectToggle.Inverted`** | [MA 反应式组件 §14（极性语义全解）](references/04-外观与表情/04-06-ma-reactive-components.md) |
| **面捕/自建 MergeAnimator 烘焙后变 `NO VALID ANIMATIONS`、曲线一条不生效 / 形态键疑似被 AAO 删掉** | [OSC 面捕 §2.3/§5](references/04-外观与表情/04-08-face-tracking-osc-and-repair.md)、[MergeAnimator §1.7](references/03-菜单与参数/03-07-ma-platform-sync-mmd-and-motion.md) |
| **编辑器里驱动/参数注入验证不动、`SetFloat` 改不了 Bool、要拆开验证 driver** | [动态验证边界 §1.3](references/07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md)、[GM 自动化 §19–§20](references/07-验证与排查/07-06-gesture-manager-automated-sim-test.md) |
| **Play 里直接 `animator.SetFloat` 没反应 / 菜单项存在但对象不动（MA Responsive 层判读）/ 袜子类屈膝穿模验收** | [GM 自动化 §19–§20](references/07-验证与排查/07-06-gesture-manager-automated-sim-test.md)、[服装接线 §12–§14](references/05-动骨与穿模/05-01-outfit-fit-and-clipping.md) |
| 合并骨架 / 配件挂骨 / 调单骨缩放或整体高度 / 头饰第一人称可见 / 隔离 PhysBone / 全局碰撞体 | [MA 骨骼与几何组件全解](references/05-动骨与穿模/05-10-ma-bone-and-geometry-components.md) |
| 「配了却不生效」/ 要确认 MA 版本行为与字段名 / 要查来源权威顺序 | [常见坑与版本实测](references/07-验证与排查/07-05-ma-gotchas-version-toolchain.md) |
| **开启面捕后眼睛始终闭着 / 关掉面捕也不眨眼 / 眼睑映射看起来是坏的** | [OSC 面捕 §3.1–§3.3](references/04-外观与表情/04-08-face-tracking-osc-and-repair.md) |
| **表情/眨眼/口型层都在但完全不生效（曲线路径失效）/ 第三方补丁包 clip 键名对不上** | [OSC 面捕 §5.4/§6.4](references/04-外观与表情/04-08-face-tracking-osc-and-repair.md) |
| **菜单里三选一点点没反应（Button 绑 Int）/ 径向滑块拖了没效果 / 音量曲线烘焙后消失** | [反射建菜单 §18.1–§18.2](references/03-菜单与参数/03-06-ma-menu-build-by-reflection.md)、[OSC 面捕 §5.3](references/04-外观与表情/04-08-face-tracking-osc-and-repair.md) |
| **反相修好后「没开面捕反而眼睛全闭」/ 烘焙出来的控制器名字变成 `VRCFury FX`** | [OSC 面捕 §3.2](references/04-外观与表情/04-08-face-tracking-osc-and-repair.md)、[动态验证边界 §1.2](references/07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md) |
| **游戏内「还没切到这个头像时的 3D 预览」是裸体 / 要判「出包静态态(Rest Pose)到底对不对」** | [反射建菜单 §19.12](references/03-菜单与参数/03-06-ma-menu-build-by-reflection.md) |
| **面捕注视方向瞳孔不动 / 面捕开着时手势表情仍生效 / 模型自带眨眼导致双眨** | [OSC 面捕 §3.4–§3.7](references/04-外观与表情/04-08-face-tracking-osc-and-repair.md) |
| **CD 盘想换成真实歌曲封面 / 圆盘遮罩在安全系统下失效** | [插件集成 §1.4–§1.7](references/02-工程组织与工作流/02-11-plugin-integration-rk-realkiss-props.md) |
| **Play 里按原形变键名读权重全是 -1（AAO 把键并成 `AAO_Merged_*`）/ 编辑器渲染色变键图不更新** | [OSC 面捕 §5.2](references/04-外观与表情/04-08-face-tracking-osc-and-repair.md)、[动态验证边界 §5](references/07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md) |
| **Pawlygon 面捕原包重装后全脸不动（OSCmooth 层）/ 烘焙后 EP 超 256 bits / 兜帽卫衣鞋子袜子互抢 / 一键穿衣联动多件 / RealKiss 手指碰嘴无反应 / VRCFury 工程 Play 里参数被复位** | [OSC 面捕 §1/§6.5](references/04-外观与表情/04-08-face-tracking-osc-and-repair.md)、[参数审计 §10.2](references/03-菜单与参数/03-01-parameter-and-menu-audit.md)、[反射建菜单 §19.11](references/03-菜单与参数/03-06-ma-menu-build-by-reflection.md)、[插件集成 §2.2](references/02-工程组织与工作流/02-11-plugin-integration-rk-realkiss-props.md)、[动态验证边界 §1.2](references/07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md) |
| **面捕所有通道集体失效但眼睑分支正常 / 烘焙后 Body 键变成 AAO_Merged_* / 眼动只上下不动左右 / 兜帽卫衣开关父级遮断 / 舌头被两个 LongStep 拉扯 / SDK 脚本上传被拒** | [OSC 面捕 §2.1/§3.8/§5.1](references/04-外观与表情/04-08-face-tracking-osc-and-repair.md)、[MA 反应式 §14.5](references/04-外观与表情/04-06-ma-reactive-components.md)、[上传凭据 §2.1](references/06-构建上传与备份/06-06-upload-credentials-and-client-evidence.md) |
| **升级 SDK 3.10.6 后编译报 CS0618 / 想升级 VPM 包但官方源 522 / 原生 Face Tracking 怎么配 / 面捕协议弹窗点不动 / PUT 对齐后 variant 串了** | [环境工具链 §8](references/01-环境与工具链/01-01-environment-and-toolchain.md)、[原生 FT §2](references/04-外观与表情/04-09-face-tracking-native-sdk.md)、[上传凭据 §2.2](references/06-构建上传与备份/06-06-upload-credentials-and-client-evidence.md)、[版本对齐 §11](references/06-构建上传与备份/06-05-avatar-package-version-collision.md) |
| **上传后说不清客户端到底拉没拉到新包 / 日志体积新旧都是 54 MB / 缓存 md5 对不上 .vrca / 重开后 __data 时间戳没变** | [上传凭据与客户端取证 §4](references/06-构建上传与备份/06-06-upload-credentials-and-client-evidence.md) |
| **客户端日志报 Avatar Expression Parameter duplicate definition / EP 与 FX 同名但类型不同 / MA 内部参数被导出到 EP / IsLocal 冲突** | [参数审计 §10.5–§10.6](references/03-菜单与参数/03-01-parameter-and-menu-audit.md) |
| **原生 FT 下瞳孔只不左右动但上下/眨眼/嘴都正常 / 升级 SDK 后面捕回归 / 8 个 gaze 键怎么映射到方向键 / EyeGazeMode Both 不生效 / 静态 blendshape 权重 100 没被重置** | [原生 FT §4–§6/§9–§10](references/04-外观与表情/04-09-face-tracking-native-sdk.md) |
| **改 FT 映射到 eye_look_* 后 Play 里权重被刷回 0 / AAO 报未知 VRCFury 组件 / 新建形变键会不会被 AAO 删 / 骨骼从 125 变 3 / 眯眼时瞳孔高光漂出眼球** | [原生 FT §7–§8](references/04-外观与表情/04-09-face-tracking-native-sdk.md) |
| **GUN1 装备后武器消失 / 万圣节一键穿衣裸体 / CD9·CD10 浮空 / 手套指向露指尖 / AFK 切换无变化 / 上传 TaskCanceledException 但版本已变** | [插件集成 §3.1/§1.5](references/02-工程组织与工作流/02-11-plugin-integration-rk-realkiss-props.md)、[骨骼几何 §21](references/05-动骨与穿模/05-10-ma-bone-and-geometry-components.md)、[蒙皮修复 §4](references/05-动骨与穿模/05-11-skinning-binding-and-geometry-repair.md)、[反射建菜单 §17](references/03-菜单与参数/03-06-ma-menu-build-by-reflection.md)、[版本对齐 §9](references/06-构建上传与备份/06-05-avatar-package-version-collision.md) |
| **道具装到 chibi 头像后停在脚踝 / 场景里量包围盒得到荒谬值 / 握持判据随动画 PASS-FAIL 交替 / `tools.mcp__unity__* is not a function` / 新脚本 refresh 过但反射找不到类型 / 上传后两档 unityVersion 又分叉 / 客户端证据拿不到该怎么写** | [骨骼几何 §22–§23](references/05-动骨与穿模/05-10-ma-bone-and-geometry-components.md)、[MCP 桥 §13](references/01-环境与工具链/01-06-project-launch-and-mcp-bridge.md)、[版本对齐 §10](references/06-构建上传与备份/06-05-avatar-package-version-collision.md)、[上传凭据 §4.6](references/06-构建上传与备份/06-06-upload-credentials-and-client-evidence.md) |
| **客户端报 `Avatar Expression Parameter 'AFK' duplicate definition`（EP 资产本身没有 AFK）/ MA Parameters 声明内置参数名被导出 / 验证脚本 FirstOrDefault 对 EP Parameter 返回 null 抛 NRE / 编辑器在后台 delayCall 永远不触发、烘焙用的是旧程序集** | [参数审计 §10.5](references/03-菜单与参数/03-01-parameter-and-menu-audit.md)、[动态验证边界 §3/§8.3](references/07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md) |
| **相机渲染有衣服但 Scene 视图裸体 / 衣装分组根被关 / Play 了 Animator 为 NULL / 上传日志 -match 误判 / 缩略图幂等 / execute_code 绑定蒸发** | [动态验证边界 §6.5/§1](references/07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md)、[反射建菜单 §19.7](references/03-菜单与参数/03-06-ma-menu-build-by-reflection.md)、[GM 自动化 §19](references/07-验证与排查/07-06-gesture-manager-automated-sim-test.md)、[上传凭据 §6.4/§2.4](references/06-构建上传与备份/06-06-upload-credentials-and-client-evidence.md)、[动态验证边界 §8.1](references/07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md) |
| **Base_B invalid normals 反复 / 休闲装开关反相、口袋陷入衣面 / localOnly 不降预算 / GM 舞蹈不动 / 面捕根级菜单 / RealKiss 口裂定标** | [蒙皮修复 §8/§7](references/05-动骨与穿模/05-11-skinning-binding-and-geometry-repair.md)、[MA 反应式 §14.1](references/04-外观与表情/04-06-ma-reactive-components.md)、[参数审计 §10.4](references/03-菜单与参数/03-01-parameter-and-menu-audit.md)、[GM 自动化 §23.3–§23.4](references/07-验证与排查/07-06-gesture-manager-automated-sim-test.md)、[原生 FT §12.2](references/04-外观与表情/04-09-face-tracking-native-sdk.md)、[插件集成 §2.4](references/02-工程组织与工作流/02-11-plugin-integration-rk-realkiss-props.md) |

## 使用前提（别跳过）

- **基底三条铁律依然优先**：MA 组件分两类（烘焙期才生效 / 编辑态就有反馈，见铁律 1）；不要靠缩身体修穿模；上传是用户动作。扩充分册**不改变**这三条；本工作区在此之上另加三条：**铁律 4 MA 优先（不用 VRCFury）**、**铁律 5 Gesture Manager 必装必用**、**铁律 6 新增/转换必须实际绑骨 + 移动态菜单验收**。
- **⚠️ 验证必须走「多角度独立验证五件套」（用户 2026-10-27 指定，已替换旧验证步骤）**：**① 数值**（骨骼 world 位置逐位对照参照头像、`BakeMesh`+`TransformPoint` 的世界包围盒判据表、烘焙 EP/菜单/参数）
  → **② 编辑态离线多角度出图**（正/侧/背 + 手部/头部/被测件特写；FOV28、深色底、**相机必须放在 `target + forward*dist`（+Z 侧）**、隐藏无关件）→ **③ 目视判读**（贴位/浮空/穿模/比例/颜色六项清单；图先 PIL 缩到 ≤420 px 再读）
  → **④ Windows-MCP 实机多角度查看**（`uia.ps1 -Action focus` 置前 → 设 `SceneView.pivot/size/rotation` 并 `drawGizmos=false` → 抓屏裁剪 600 px 读图，正/侧各一次）
  → **⑤ 烘焙产物验证**（`VRCFury Menu/Params/FX` 三层 + `NO VALID ANIMATIONS`/`weight=0` 判读 + 退出 Play 还原 5 个 LightController anim 的 MD5）。
  完整流程与判据表见 **[07-08 多角度独立验证标准流程](references/07-验证与排查/07-08-multi-angle-binding-verification.md)**，以后所有装骨/配件/衣装/道具改动一律照此执行，
  不允许只截图一张或只看 Console 就交付。
- 本机编辑器位于 `<VRCHAT_ROOT>\tools\Unity\2022.3.22f1\Editor\Unity.exe`（2022.3.22f1c1）。**必须先把工程打开**（走 ALCOM 的「打开 UNITY」），**且编辑器端跑的是 stdio 桥（`127.0.0.1:6400`）**，
  `mcp__unity__*` 才能连上；在那之前所有需要编辑器的步骤只能标 `MCP_REQUIRED`。Unity 里那句 `MCP-FOR-UNITY: Session connected` 只代表 HTTP 链路，**不能**当「AI 已连上」——
  判据与修法见 [01-06-project-launch-and-mcp-bridge.md](references/01-环境与工具链/01-06-project-launch-and-mcp-bridge.md) §3/§4。
- 构建与上传**只能在 Unity GUI 模式下进行**（`-batchmode -nographics` 下 `VRCSdkControlPanel.TryGetBuilder()` 不注册）；账号登录、Build & Test、Build & Upload 与面板二次确认**都是用户动作**。
- 证据要分层标注（`UNITY_RESOLVED` / `PROVIDER_PREVIEW` / `NDMF_BUILT` / `SDK_BUILD` / `CLIENT_RUNTIME` / `UPLOAD_CONFIRMED`），**没跑就写 `NOT_RUN` / `BUILD_REQUIRED`**——别把「看代码觉得对」说成验证过。
- 属性名、阈值、生成规则**都按现场版本读回**；本套文档里的具体数字（如 lilToon 2.3.0 / MA 1.14.3 / SDK 3.10.5）**只是「检查过」的版本标注，不是通用常量**。
- **来源与许可**：上游仓库、教程系列、官方文档、逐册来源映射与 MIT 声明见 [ACKNOWLEDGEMENTS.md](ACKNOWLEDGEMENTS.md)。**教程口径随工具版本漂移**（教程会下架、组件名会改，例如 MA 1.13 起 `MergeArmature` 改名）；转写可能写错插件名/参数名，一律以 Unity 界面实际字段为准；照做前先用工程里的实际组件/菜单核对，对不上就标 `STALE`。

**全部分册（含基底 7 册）的逐册说明与来源映射**见 [references/README.md](references/README.md)；
包总览（**包外，可能不存在**）：`<VRCHAT_ROOT>/tools/sources/vrchat-avatar-modding/README.md`。

---

## 知识库自身的更新规则（本工作区，用户 2026-10-17 要求）

这个包是**活的**：在本工作区干活时发现「它没写 / 写错了 / 过时了」的东西，**必须当轮补进来**，不许只留在对话里。规则以本节为准（**若工作区根存在 `AGENTS.md`，其 🚩 必读区的「★★ 知识库自更新机制」与本节同级，两者冲突时以用户当轮指令为准**），要点：

- **落点（2026-11-11 起，用户指定）**：通用原则 / 铁律 → 本 `SKILL.md`；**一切新实测知识（含症状类）直接转化为结构化知识**，并入 `references/01–07` 对号的主题册（按「症状 → 机制/根因 → 正确做法 → 判据 → 来源」组织，追加小节并标来源轮次/日期）；新主题 → `references/<分类>/NN-MM-<slug>.md`（MM 取该分类最大册序 +1）。**不再设「症状速查/增补册」栏目，不按时间开册**；原 08 目录已于 2026-11-11 删除，其全部内容已结构化并入各主题册。`references/00-基底/` 上游原文保留，补充写在各册末尾「补充」节；册数与索引改 `references/README.md`。
- **写法**：症状 → 根因 → 正确做法 → 判据；带命令、API 全名、文件路径 + 行号、实测数字；标「本工作区实测 YYYY-MM-DD」；**没验证的写「未验证」**，不许写成结论。被取代的旧口径就地标「⚠️ 取代」并指向新小节，不静默删除。
- **三件套同步**：`references/README.md`（册数与索引）→ 本 `SKILL.md`（分册导航 / 症状路由表 / 使用前提）→ **工作区根 `AGENTS.md`（若存在：必读区 + 第九节 + 状态行轮次 +1）**。
- **记账**：`references/_本工作区补充日志.md` 已于 2026-11-09 按用户要求删除，**不再新建台账**；轮次与改动直接体现在对应分册的「本工作区实测 YYYY-MM-DD」标注与 `references/README.md` 的册数 / 索引里；已知缺口清单保留在 `references/README.md` 的「待补」节。

---

## 最近一轮实测的结构化落点（第 175–177 轮 · 2026-11-10 至 11-11）

> 原「症状速查（第 177 轮补充）」表已按用户要求转化为结构化知识，落点如下；机制与判据全文在对应分册。

| 实测主题 | 结构化落点 |
|---|---|
| Scene 视图裸体 ≠ 相机渲染（SceneVisibilityManager 判据） | [07-09 §6.5](references/07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md) |
| 衣装分组根「原子关、壳必开」（`__MA/ActiveSelfProxy` 67 个） | [03-06 §19.7](references/03-菜单与参数/03-06-ma-menu-build-by-reflection.md) |
| Play 里 Animator 为 NULL 不算动态验证（GM 反射 + SUMMARY 判据） | [07-06 §19](references/07-验证与排查/07-06-gesture-manager-automated-sim-test.md) |
| 上传日志 `-match 'DONE'` 大小写坑 / 缩略图幂等 / 版本对齐证据链 | [06-06 §6.4/§2.4](references/06-构建上传与备份/06-06-upload-credentials-and-client-evidence.md)、[06-05 §8](references/06-构建上传与备份/06-05-avatar-package-version-collision.md) |
| execute_code 绑定蒸发的 flag-file 兜底 / DeleteAsset 被拦 | [07-09 §8.1–§8.2](references/07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md) |
| LightController 5 anim 构建期重写（备份路径 + MD5） | [07-01 §25.5](references/07-验证与排查/07-01-bake-and-validation.md) |
| 面捕眯眼高光漂出眼球（hightlight_* 共用顶点 + FT 专用键归零） | [04-09 §8](references/04-外观与表情/04-09-face-tracking-native-sdk.md) |
| 娃娃机「音声OFF」默认 OFF 极性=1（3 字段最小改法） | [03-06 §19.8](references/03-菜单与参数/03-06-ma-menu-build-by-reflection.md) |
| Manual Bake 源场景留 (Clone) 收尾 / Play 卡 is_changing 必须 NOT_RUN | [07-09 §6.3/§1.10](references/07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md) |
| GM 跑不起来时「构建产物 FX + 逐帧 Update」证据配方 / ForceSkinnedUpdate / 特写按骨坐标 | [07-09 §2/§5/§7.1](references/07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md) |
| SDK 占位凭证判别 / APIUser.Login + CookieJar / 主线程 / UpdateAvatarImage | [06-06 §1.4–§1.5/§2.3–§2.4](references/06-构建上传与备份/06-06-upload-credentials-and-client-evidence.md) |
| Base_B invalid normals 双根因（Blender4 + SDK legacy 链） | [05-11 §8](references/05-动骨与穿模/05-11-skinning-binding-and-geometry-repair.md) |
| OT 极性 inv=False+Active=True+authored0 / 口袋陷入 BakeMesh 测量 | [04-06 §14.1](references/04-外观与表情/04-06-ma-reactive-components.md)、[05-11 §7](references/05-动骨与穿模/05-11-skinning-binding-and-geometry-repair.md) |
| EP localOnly 不移除源 EP 参数 / 多声明任一 synced / CalcTotalCost 权威 | [03-01 §10.4/§10.2](references/03-菜单与参数/03-01-parameter-and-menu-audit.md) |
| GM 不启用 Action 层（AnimationMode 采样裁决）/ GM 可模拟 Contact | [07-06 §23.3–§23.4](references/07-验证与排查/07-06-gesture-manager-automated-sim-test.md) |
| 面捕菜单重定向四步（避 FT Controls 自递归）/ RealKiss 口裂定标 | [04-09 §12.2](references/04-外观与表情/04-09-face-tracking-native-sdk.md)、[02-11 §2.4](references/02-工程组织与工作流/02-11-plugin-integration-rk-realkiss-props.md) |

## 2026-11-12 增量路由（PLAN-001，小狼）

| 问题 | 结构化落点 |
|---|---|
| SyncDances只放歌、抽取后旧层索引、AFK覆盖、开启Action后Walk/Run冻结 | [03-07 §14](references/03-菜单与参数/03-07-ma-platform-sync-mmd-and-motion.md#14-抽取舞蹈-action-后只放歌afk-覆盖与走跑冻结2026-11-12)；**07-06 §23.3旧GM限制结论的取代边界已注明** |
| CandySweat/TamagoOverall上下装真拆分、FBX原生单位与共享插值接缝 | [05-11 §13](references/05-动骨与穿模/05-11-skinning-binding-and-geometry-repair.md#13-合并衣装拆为独立上下装原始单位连通岛与共享腰线2026-11-12) |
| 分类素材时是否移动ZZZ_GeneratedAssets | [02-05 §10](references/02-工程组织与工作流/02-05-unity-project-asset-reorg.md#10-ndmf-持久化生成根应按当前源码保留2026-11-12)；以当前NDMF固定输出源码为准 |
| 面捕眯眼时白色高光消失，旧129点归零补丁的副作用 | [04-09 §15](references/04-外观与表情/04-09-face-tracking-native-sdk.md#15-高光归零补丁引起的虹膜遮挡与按眼补偿)；88点随虹膜+4误伤点恢复，§8旧步骤已标取代边界，客户端另验 |
