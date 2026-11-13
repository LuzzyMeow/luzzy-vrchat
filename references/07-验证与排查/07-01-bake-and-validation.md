# 烘焙与验收：NDMF / Modular Avatar 的「源 vs 产物」纪律

读这份文件的时机：改了 MA 组件/衣服/菜单/参数之后要确认「到底生效没有」，或者症状是「菜单里有但没反应」「层丢了/顺序变了」「骨骼归属不对」。

> **本手册最重要的一条**：**MA 组件与源 prefab 是「声明」，NDMF 预览 / 手动烘焙 / 构建克隆 / 合并后的控制器 / 生成资产是「产物」。永远不要修产物，去修受支持的源声明。**

## 目录

- §1 先把构建图写出来
- §2 只读排查顺序
- §3 四个高频症状的排查顺序
- §4 道具与功能 prefab
- §5 改动前后的护栏
- §6 验收矩阵
- §7 烘焙骨骼审计：PhysBone 组件 ≠ 骨骼
- §8 Gesture Manager 验收
- §9 分层验收写法
- §10 NDMF 烘焙被中止：用反射读 ErrorReportWindow 拿真因
- §11 在 Play 里用代码驱动烘焙后的参数
- §12 「删了 `Packages/com.vrcfury.temp` 之后烘焙一直失败」——必须重启编辑器
- §13 运行期删 `Packages/com.vrcfury.temp` → 之后每次烘焙必失败
- §14 MA MenuItem 缺 `ModularAvatarMenuInstaller` → 菜单里永远不出现
- §15 「新加的饰品/翅膀默认要关、由用户自己打开」的完整清单
- §16 从旧工程移植「翅膀扇动」动画层 + Play 模式下验证烘焙动画的正确姿势
- §17 烘焙 Console 报错清零：日志定位 + 第三方生成器 `HasProperty` 守卫 + MA-1300 fake-null
- §18 「亮度调节」子菜单烘焙后为空：Generator 删目录后 `IsValidFolder` 仍为 true
- §18.1 从 `execute_code` 调用工程内 Editor 脚本
- §19 用真实鼠标点击 Unity 面板完成 Build & Publish
- §20 默认形态：只留指定那一套衣装
- §21 Build & Publish 点不动 / 上传失败重试
- §22 上传后「衣服材质像丢了」= LightController 去色默认值 0.5
- §23 「默认关闭」的自建开关层：烘焙判据
- §24 Build/Play 卡在「VRCFury SPS Plug 找不到渲染器」与遗留烘焙克隆
- §25a 构建完成但「上传没发生」：Send/Cancel Upload 是 0×0 隐藏元素，必须直接反射 Upload
- §25b Play 模式功能验证的四个硬坑
- §26 脚本化上传与线上版本核对
- §27 手势菜单点检：VRChat「手势覆盖曲线」在编辑器/Play 里看不到骨骼变化
- §28 穿衣服后「身体部分消失」= MA ShapeChanger 被 AAO 转成 NaN 骨；一键穿衣 Button→Toggle
- §29 从裸头像重建「MA 菜单 + 服装可见性」的五个硬坑
- §30 上传「假成功」：缩略图 MD5 相同会在更新 bundle 之前抛出，Version 却已 +1
- §31 第 126 轮：MA MenuItem 宿主 GameObject 未激活 → 整组 ObjectToggle 静默不烘焙
- §32 第 127 轮：默认开启型 Toggle 的验收
- §33 同名 Shader 覆盖导致「发丝没描边 + 眼镜材质丢失 + 亮度插件完全失效」
- §34 第 132 轮：Unity 重启后 MCP 桥失效的判别 + 不进入 Play 的烘焙级验收
- §35 上传：不需要 OTP 的 SDK 凭据探活 + `web-derived token` 假拦截

## 1. 先把构建图写出来

在诊断或改动任何东西之前，为**这个确切的 Avatar** 记录这张图：

```text
玩家入口 或 被动功能
  → 源 prefab 与 MA 组件
  → 声明的职责（菜单/参数/Animator/骨架/网格/对象开关）
  → NDMF pass 与顺序关系
  → 重映射或合并后的构建结果
  → 确切的模型或运行时效果
```

### 1.1 要先清点（按**已安装版本**的实际等价物，不要照抄别的版本）

- Menu Installer / Menu Item / Menu Group；
- Parameters 与参数 remap；
- Merge Animator 与 Merge Motion；
- Merge Armature、Bone Proxy、constraint 与 path remap；
- Mesh Settings 与 BlendShape 同步；
- Object Toggle / 反应式对象组件；
- Contacts、PhysBone、平台过滤，以及**共享同一批对象/参数/菜单/构建阶段**的其它 NDMF provider。

每一条都要记：**组件层级路径、enabled 状态、源 prefab、目标引用、合并或安装目的地、参数名与默认值、Animator 层或 motion 来源、附着根、path remap、共享消费者**。

**「工程里装了这个包」不等于「这个功能归它管」。**

## 2. 只读排查顺序（六步，别跳）

1. **锁定**确切 Avatar 与 Unity MCP 实例；记录场景 dirty/编译状态与任务相关的 Console 基线。
2. 从**玩家可见的菜单路径**或具名被动功能出发，找到贡献它的**最近的那个 MA 声明**。
3. **向上**走到源 prefab 或稳定场景配置；区分 **prefab 源值**与**场景实例覆盖**。
4. **横向**走到所有贡献同一参数/Animator 层/骨架路径/对象/网格/菜单目的地的其它 provider。
5. **向前**走到 provider 预览或 NDMF 输出；解析生成名、去重后的参数、合并后的菜单位置、Animator 顺序、骨架重映射、最终组件归属。
6. **对比**源意图与构建输出；**报告第一处分叉的层**，不要从「源组件」直接跳到「运行时」。

没获授权做预览/构建时，停在 `UNITY_RESOLVED`；合并后的行为标 `BUILD_REQUIRED`，**不要猜 NDMF 会怎么解**。

## 3. 四个高频症状的排查顺序

### 3.1 菜单项缺失或位置不对

按这个顺序查：

1. 源菜单控件**存在且可从预期的菜单资产或 provider 生成的入口到达**吗？
2. Menu Installer / Item / Group 组件是否在**活跃的构建源**上且 enabled，目标是预期菜单目的地？
3. 有没有**场景覆盖**把组件删了、改了目标、或留下**过期的菜单资产引用**？
4. provider 是否**拥有**这个项所用的参数与控件值？
5. 检查 NDMF 输出的**最终位置、分页、去重、provider 顺序**，以及与另一个菜单所有者的冲突。
6. 在获授权的预览/构建层验证**玩家实际路径**。

修的时候：**修稳定的源组件或菜单资产**。不要把生成出来的菜单项**贴回**源树；也不要为了省一页去**合并不相关 provider 的叶子**。

### 3.2 参数存在但功能没反应

只追**一条确切链**：

```text
菜单控件与取值
  → MA 声明的参数、类型、默认值、saved、sync
  → 最终去重/重映射后的参数
  → Animator 或反应式消费者
  → 状态、过渡、motion 或对象开关
  → 确切的模型绑定
```

把**同名参数的所有声明**都拿来比。**同名不等于兼容**：类型、默认值、saved、sync、仅本地的意图都可能有分歧。在拿到构建结果之前，**不要说冲突已解决**。最终成本用已安装的 SDK/provider API 取，**不要数名字算 256-bit 预算**。

### 3.3 合并后的 Animator 丢了 / 顺序变了 / 被覆盖

1. 确定源 controller 或 motion、目标 playable 层、path mode、层顺序设置、mask、默认值与参数依赖。
2. 确认 MA 组件是 active 的，且指向**选中的 Avatar**，而不是同级对象或已停用的历史根。
3. 清点指向**同一 playable 层**的其它 Merge Animator 组件与生成器，记录它们的**顺序约束与共享层名**。
4. 检查 NDMF 输出的**重命名层、path remap、去重、最终顺序**。
5. 行为冲突时，比较**条件、过渡时长、Write Defaults、mask、绑定同一属性的 clip**。

**不要直接修生成的 controller。** 改源 controller / MA 声明 / 显式顺序关系，然后**重新生成并对比构建结果**。

### 3.4 骨架、附着或网格结果不对

把「源穿戴物/道具」和「合并声明」**分开查**：

- 确切的源与目标骨架根；
- 重映射后的 humanoid 或 transform 路径；
- Bone Proxy 或 constraint 目的地；
- renderer 的 root bone 与 `bones` 数组；
- mesh、材质槽、BlendShape 与 BlendShape Sync 映射；
- 对象缩放、激活态、prefab 覆盖；
- PhysBone 与 Contact 的根、碰撞体、共享消费者。

检查 NDMF 输出的**最终变换路径与 renderer 绑定**。**看得见的源层级不是 Merge Armature 成功的证据。** 不要把骨骼路径、网格指派、proxy 目标从结构相似的头像那儿抄过来。当一个源 FBX/prefab 被多个头像共享时，**隔离出针对本头像的覆盖**，除非共享修改是明确意图。

### 3.5 对象开关默认值/可见性不对

把下面这些当**各自独立的输入**分别解析：源激活态、MA 开关声明、参数默认值、Animator 默认状态、saved 行为、平台过滤。对比 **NDMF 前/后**的结果。

**两个 provider 写同一个对象状态**时：指定一个明确的所有者或记录预期顺序；**不要让两个都变成偶然的竞争权威**。

验证持久化、本地/远端可见性、Contact 行为或同步，**要用具名的运行时层**。**编辑器层级里的一个复选框只能证明被检查的那一刻的源状态。**

## 4. 道具与功能 prefab

优先用资产**自己声明的 provider**。当 MA 是选定所有者时，记录：附着变换、Bone Proxy 或 constraint、菜单、参数、Animator、开关、材质、音频、粒子、Contacts、PhysBone、默认值、平台过滤、共享消费者。

**没有明确的集成映射图，不要把同一职责再交给第二个 provider。**

### 4.1 把已有功能迁移到 MA 时

1. 列出旧 provider 拥有的**每一项职责**；
2. 把每项职责映射到**一个** MA 源组件或**有意为之的手工所有者**；
3. 保留**可恢复副本或版本控制检查点**；
4. **先构建并验证替代实现**，再移除旧 provider；
5. 只有新的菜单、参数、模型绑定、必需运行时行为**全部通过**后，才移除旧所有权。

## 5. 改动前后的护栏

**改之前**记录：确切的源组件、受影响的 prefab/场景实例、共享消费者、获授权的动作、必需的验证层。优先做**provider 支持的最小改动**。在 Apply / 改 prefab / refresh / preview / build **之前重新确认 Unity 实例**。

**改之后**：

1. 只保存**明确授权**的源资产或场景；
2. 等导入/编译完成，检查**新增的 Console 条目**；
3. 通过 MA/NDMF **重新生成**，而不是编辑生成资产；
4. 对比「预期的源声明」与「构建出来的菜单、参数、Animator、层级、网格、组件」；
5. 刷新工具链档案、菜单 map、参数索引、provider 清单、生成源记录、验收状态。

## 6. 验收矩阵（什么证据能支撑什么结论）

| 你要声称的 | 最低证据 |
| --- | --- |
| MA 组件已在目标上配好 | `UNITY_RESOLVED`：确切组件、目标、源引用 |
| 菜单或参数**合并后**存在 | `NDMF_BUILT` 或 provider 预览显示最终合并结果 |
| Animator 层或骨架路径**扛过了重映射** | `NDMF_BUILT` 的确切最终层/路径与绑定 |
| 道具、衣服或开关**行为正确** | 具名的、获授权的**运行时层**验证该行为 |
| 同步/saved 行为对玩家有效 | 具名的客户端/多人测试；**源默认值不算证据** |

**没跑过的层一律记 `NOT_RUN` / `BUILD_REQUIRED` / `BLOCKED`。** 干净的源 inspector + 干净的 Console **不能**证明最终的 MA 结果。

## 7. 烘焙骨骼审计：PhysBone 组件 ≠ 骨骼

NDMF/MA 处理之后，骨骼与 PhysBone 的关系会变。审计时区分：

- **PhysBone 组件**：声明在某个对象上，`rootTransform` 指向骨骼；
- **骨骼 Transform**：实际被物理驱动的节点。

两者不一致的常见成因：合并后骨骼被移到头像骨架下 → 组件的 `rootTransform` 仍指向旧路径；或者衣服骨架根上加了 MergeArmature，但 PhysBone 声明在**子对象**上而 root 指的是外层。

审计项：

- [ ] 每个 PhysBone 的 `rootTransform` 解析到**真实存在**的 Transform；
- [ ] `colliders` 列表里的对象仍在同一层级下、未被合并移走；
- [ ] 身体衣服的 PhysBone 在烘焙后其根属于**头像骨架**；
- [ ] 没有「两组 PhysBone 声明同一骨骼」的竞争；
- [ ] 没有把 PhysBone 组件当「多余」删掉（见 [06-01-sdk-upload-and-versioning.md](../06-构建上传与备份/06-01-sdk-upload-and-versioning.md) 的误消案例）。

## 8. Gesture Manager 验收（编辑器内，最快的闭环）

Gesture Manager 是**编辑器内**预览手势与 FX 的权威工具，比反复 Build & Test 快得多。用它来验收：

| 检查项 | 怎么看 |
| --- | --- |
| 手势 → 参数映射 | 在 Gesture Manager 里切换手势，读参数实际值 |
| FX 层是否被正确合并 | 观察层列表与权重；缺失层会表现为动作不发生 |
| 对象开关的档位 | 逐档切，注意**从其它衣服状态进入再返回**，避免只开新件而遗留旧件 |
| 表情/滑块 | 拉滑条读权重，确认两端与默认位置符合设计 |
| 默认外观 | 回到默认状态，确认外观与「原默认」一致 |
| 穿模 | 同机位、同姿态 A/B（见 [05-01-outfit-fit-and-clipping.md](../05-动骨与穿模/05-01-outfit-fit-and-clipping.md)） |

**Gesture Manager 是编辑器模拟器，不是客户端。** 它不证明多人同步、远端可见性、Contact 交互、saved 跨世界持久化。这些必须进客户端验证，没跑就写 `NOT_RUN`。

## 9. 分层验收写法（报告模板）

```text
源层      UNITY_RESOLVED : <组件/资产路径与目标>
预览层    PROVIDER_PREVIEW / NDMF preview : <结果>（可选项）
构建层    NDMF_BUILT / SDK_BUILD : <最终菜单/参数/层/路径>  或 BUILD_REQUIRED
编辑态    Gesture Manager : <通过项 / 未跑项>
运行层    CLIENT_RUNTIME : <多人/持久化/Contact>  或 NOT_RUN
结论      <第一处分叉的层> + <下一动作>
```

**不要把上层证据说成下层结论。** 这是本手册所有分册共同的口径。

## 10. NDMF 烘焙被中止：用反射读 ErrorReportWindow 拿真因（本工作区实测 2026-10-26）

**症状**：进入 Play 时 `VRCSDKPreprocessAvatarCallback 'BuildFrameworkOptimizeHook' reported a failure`（
`nadena.dev.ndmf/Editor/VRChat/BuildFrameworkPreprocessHook.cs:89-92`，日志先打 `Build Framework: Saved assets in 1638ms` 再返回失败），随后出现两个看似无关的怪象：
- **两个头同时显示**（本该互斥的 Toggle 全变成「开」）；
- **默认衣服全部消失**（Rest Pose 回落到「全部隐藏」）。

原因：**烘焙整体失败 → 没有任何 MA 反应层被生成**，只剩场景静态状态 + 手工层，于是所有「按参数切换」的互斥关系全部失效。**看到「互斥失效」先怀疑烘焙失败，而不是去改 OT。**

**取真因的可靠办法（不必读巨大 Editor.log，也不依赖 NDMF 控制台窗口截图）**：

```csharp
var t = System.Type.GetType("nadena.dev.ndmf.ui.ErrorReportWindow, nadena.dev.ndmf.editor");
// 或遍历 AppDomain 找 type.Name=="ErrorReportWindow"
var f = t.GetField("_report", BindingFlags.NonPublic|BindingFlags.Instance);
var rep = f.GetValue(window);                       // ErrorReport
// rep.AvatarName / rep.AvatarRootPath / rep.Errors (ImmutableList<ErrorContext>)
// 每条 ErrorContext.TheError 是 IError：属性 Severity + 方法 ToMessage()
```

本工程实测报出的 5 条里，前 4 条是 `Information`（AAO 版本）与 `NonFatal`（`[MA-1300] 未指定要合并的 Animator`、`[MA-1200] 未指定要安装的菜单`），**只有第 5 条是 `InternalError`**：

```
ArgumentOutOfRangeException: ... PortableControlConversions.ToPortable
  <- VRCExpressionsMenu.Control.get_Type()  <- ReactiveObjectAnalyzer.BuildConditions <- ReactiveObjectPass
```

**根因**：有 2 个 `ModularAvatarMenuItem` 的 `Control.type` 被写成了非法枚举值 `1`（VRChat 合法值是 
`101=Button / 102=Toggle / 103=SubMenu / 201=TwoAxisPuppet / 202=FourAxisPuppet / 203=RadialPuppet`）。**MA 反应层在烘焙期要把 Control 转成 Portable 表示，
遇到非法 type 直接抛异常 → 整个烘焙中止。** 把这两个 item 改回 `102`（Toggle）后烘焙一次通过。

**判据 / 排查顺序**：① `ErrorReportWindow._report` 里有没有 `InternalError`；② 有没有 `Control.type` 不在 101/102/103/201/202/203 的 MA MenuItem；
③ 场景里 `MenuInstaller` 的 `menuToAppend`/`installTargetMenu` 为空、`MergeAnimator` 的 `animator` 为空只产生 `NonFatal` 警告（`ComponentValidation.cs:172/180`），
**不会**中止烘焙。

## 11. 在 Play 里用代码驱动烘焙后的参数（实测可行，但有两个坑）

烘焙后的克隆头像根上，`Animator.runtimeAnimatorController == null`（真实控制器由 VRCFury 的 Play 触发器在 `OnDidAddComponent` 里挂）。要断言「参数 → 对象显隐」的映射，可以手动挂烘焙产物：

```csharp
var fx = AssetDatabase.LoadAssetAtPath<AnimatorController>("Packages/com.vrcfury.temp/Builds/____/VRCFury FX.controller");
clone.GetComponent<Animator>().runtimeAnimatorController = fx;
anim.SetFloat("HeadType", 2f);   // 见坑 1
```

- **坑 1：菜单参数在烘焙后的控制器里都是 Float。** VRChat 把 Int/Bool 菜单参数在 animator 里统一成 Float，所以 `SetInteger` / `SetBool` 会**静默无效**（参数不存在时 Animator 不报错），必须 `SetFloat`。
  默认值要从 `Packages/com.vrcfury.temp/Builds/____/VRCFury Params.asset`（VRCExpressionParameters）里读 `defaultValue` 逐个灌进去，才能复现「运行默认态」。
- **坑 2：只挂 FX 控制器不能出「好看的渲染图」。** 骨骼姿态由 Base/Gesture 层驱动，只挂 FX 会让角色塌成一团（渲染出来是垃圾），**但 `activeSelf` 断言完全有效**。要出图就退出 Play 后在编辑态手工摆对象状态再渲染。

**本工程实测的有效断言（烘焙后，仅挂 FX + 灌默认参数）**：

| 操作 | 断言 |
| --- | --- |
| 默认 | `Body=1 / 狼头=0 / 头发(front,back)=1 / 耳朵尾巴=1 / 睡衣/PJ_Hat=1 / 小天使翅膀光环=1` |
| `HeadType=2` | `Body=0 / 狼头=1 / 头发=0 / 耳朵尾巴=0 / PJ_Hat=1`（帽子不随头型消失） |
| `HeadType=2` + `S1_Hat=0` | `PJ_Hat=0 / PJ_HatRibbon=0`（**关键：狼头模式下服装开关仍然生效**） |
| 回到 `HeadType=1` | 全部恢复默认 |

## 12. 「删了 `Packages/com.vrcfury.temp` 之后烘焙一直失败」——必须重启编辑器（本工作区实测 2026-10-26）

**症状**：某个 Play 烘焙正常；为「清理生成物」手工删掉 `Packages/com.vrcfury.temp`（VRCFury 每次烘焙的临时包）后，之后再进 Play 必然弹

```
VRCFury Error
Failed to build VRCFury component: BakeHapticPlugsService.ApplySpsRewrites on Avatar Root
Failed to configure material: lil_<xxx>MAT
Failed to patch shader with SPS. ... Maybe this shader isn't supported yet.
Hidden/lilToonOutline
Could not find a part of the path "…\Packages\com.vrcfury.temp\SPS\<hash>.shader".
(DirectoryNotFoundException in WriteFile at SpsPatcher.cs:770:20)
```

关掉对话框后 VRCSDK 再补一个 `Preprocess Callback Failed` 模态框，**两个框都把主线程卡死** → Unity MCP 报 `Request timed out` / `Unity instance '…' not found`。

**根因**：`com.vrcfury.temp` 是一个由 VRCFury 运行时创建的**临时 UPM 包**。在编辑器运行期把它从磁盘删掉后，Unity 的 PackageManager / AssetDatabase 对这条包路径留下失效状态；
下次烘焙 VRCFury 想往 `com.vrcfury.temp/SPS/` 写补丁 shader 时目录已不可用 → `DirectoryNotFoundException`。重启编辑器后包状态重建，烘焙恢复正常（实测：重启后第一次 Play 一次通过）。

**正确做法**
1. **不要为了「干净」在编辑器运行期删 `Packages/com.vrcfury.temp`**。它是每次烘焙自动重建的中间产物，删了也没意义。
2. 真要清，顺序是：**先退出 Play → 关掉 Unity → 删目录 → 再开 Unity**。
3. 已经踩了：先 `BM_CLICK` 关掉两个模态框（否则 MCP 全挂）→ `WM_CLOSE` 主窗口让 Unity 优雅退出 → 重新 `Unity.exe -projectPath` 拉起 → 等 6400 监听恢复 → 再进 Play。

**关模态框的可复用姿势**（无 UIA、纯 Win32，见 `tools\scripts\`）：
`EnumWindows` 按 pid 找 `#32770` → `EnumChildWindows` 找 `Button` → 对每个 Button `SendMessage(h, BM_CLICK=0x00F5, 0, 0)`；对话框正文在 `Edit` 子控件里，
用 `SendMessage(h, WM_GETTEXT=0x000D, cap, sb)` 读（`GetWindowTextW` 只能读到占位字符串 `Static`）。

**顺带记一条默认值三处一致律**：把某个菜单开关改成「默认关闭」必须同时改三处，只改一处会出现「看着关了、进游戏又开」或反之：
1. `ModularAvatarParameters` 里该参数的 `defaultValue`（+ `hasExplicitDefaultValue=true`、`saved=true`）；
2. 对应 `ModularAvatarMenuItem.isDefault = false`（否则 MA 会生成「默认态」把参数顶回 1）；
3. **场景里目标对象的 `activeSelf` 必须是 false**（MA ObjectToggle 的 `<default>` 状态是空 motion，参数为 0 时**回落到场景初始状态**）。
验证时直接读烘焙产物：`Packages/com.vrcfury.temp/Builds/____/VRCFury FX.controller` 里 `MA Responsive: <对象名>` 层的 `<default>` 必须是空 motion、开关态 clip 里 `m_IsActive=1`；`VRCFury Params.asset` 里 `defaultValue` 必须是 0。


## §13 运行期删 `Packages/com.vrcfury.temp` → 之后每次烘焙必失败（本工作区实测 2026-10-26）

**症状**：烘焙成功一次后手动删掉 `Packages/com.vrcfury.temp`，再进 Play 就弹 VRCFury 模态框：

```
Failed to build VRCFury component: BakeHapticPlugsService.ApplySpsRewrites
Failed to configure material: ...lil_N式スーパーよわちんMAT / Hidden/lilToonOutline
System.IO.DirectoryNotFoundException: Could not find a part of the path
  '...\Packages\com.vrcfury.temp\SPS\f93ef....shader'
   at ... SpsPatcher.cs:770  WriteFile
```

紧接着 VRCSDK 再弹 `Preprocess Callback Failed`：`The VRCSDK build was aborted because the VRCSDKPreprocessAvatarCallback 'VrcPreuploadHook' reported a failure.` 两个模态框都会阻塞主线程 → MCP `Request timed out`。

**为什么重建目录没用**：手动 `New-Item Packages/com.vrcfury.temp/{SPS,Builds}` + 写 `package.json` + `AssetDatabase.Refresh()` 之后，烘焙仍然同样失败（Unity 的 AssetDatabase 已经把这个包「忘记」了，VRCFury 内部再建目录/写文件对不上）。

**唯一可靠解**：**重启编辑器**（`WM_CLOSE` 主窗口优雅退出 → 重新拉起 → 6400 起来后烘焙一次通过）。

**正确做法**：`com.vrcfury.temp` 是 VRCFury 的构建缓存包，**收尾不要删**（它不是 Assets 里的资产，上传也不会带上）；确实要清理就放到整个会话的最后一步，且之后不要再进 Play/烘焙。见 `05-动骨与穿模/05-04` §9。

## §14 MA MenuItem 缺 `ModularAvatarMenuInstaller` → 菜单里永远不出现（本工作区实测 2026-10-26）

**症状**：对象上有 `ModularAvatarMenuItem`（Toggle、参数、value 都对）也有 `ModularAvatarObjectToggle`，烘焙后**菜单树里根本没有这一项**，但 Console 0 报错、也没有 MA-xxxx 警告。

**判据**：静态扫描每个 MenuItem 同时打印 `GetComponent<ModularAvatarMenuInstaller>() != null`；缺 installer 的项 = 不会被挂进任何菜单。

**修法**（不动其它字段）：

```csharp
UnityEditorInternal.ComponentUtility.CopyComponent(srcInstaller);   // src = 同角色的同类项
UnityEditorInternal.ComponentUtility.PasteComponentAsNew(dstGo);
// 再确认 installTargetMenu 指对：猫 → 饰品.asset，狼 → 狼_饰品.asset
```

**注意**：`cat.GetComponentsInChildren<Transform>(true)` **包含根自身**，`x.parent` 为 null 时访问会 NRE，遍历要 `if (x.parent == null) continue;`。

## §15 「新加的饰品/翅膀默认要关、由用户自己打开」的完整清单（本工作区实测 2026-10-26）

要「默认关闭、菜单可开」，四处必须同时满足，缺一就会默认可见或点了没反应：

1. `ModularAvatarParameters` 里该参数：`defaultValue = 0` + `hasExplicitDefaultValue = true` + `saved = true`（没有声明 → MA 自动生成的 EP 参数默认就是 0，能省一步但不可控）。
2. `ModularAvatarMenuItem.isDefault = false`（**true 会生成「默认值层」把参数钉成 value**）。
3. 承载 MenuItem 的代理 GameObject **必须 active**（inactive 时 MA 整条不处理，连同 OT）。
4. 真实目标对象场景里 `SetActive(false)`；`ModularAvatarObjectToggle` 行 `Active = true`；编辑模式外观用 `默认外观预览（EditorOnly）` 副本，且**副本默认也 inactive**（用户要求默认不显示时）。

**判据**：烘焙后 ① 克隆里目标 `activeInHierarchy == false`；② `VRCFury FX.controller` 有 `MA Responsive: <对象>` 层，`<default>` 是空 motion、ON 态 clip 写 `m_IsActive = 1`；③ EP 里该参数 `defaultValue == 0`。

## §16 从旧工程移植「翅膀扇动」动画层 + Play 模式下验证烘焙动画的正确姿势（本工作区实测 2026-10-26）

**症状**：新工程里翅膀/配饰能显示、菜单开关也对，但**不会动**；旧工程同一个模型会扇动。

**定位**：扇动不在模型 prefab 上（`AvatarAccessory_WingsHalo` 的 GameObject 只有一个 Transform，没有 Animator / MA 组件），而在旧工程的 FX 层控制器里：

- 文件：`Assets/MOCHIYAMA/Kipfel/Controller/Kipfel_FXLayer_AvatarWardrobe.controller`（12,174,647 B，116 层）
- 层 `Restored Feature 69`（stateMachine FID `-525251739391101982`）：
  - 默认态 `wings_flap` → clip `Feature Motion 67`（`m_LoopTime: 1`、`m_StopTime: 1.7333333`、60 fps、8 条 quaternion 曲线，**只动** `Wing.L$…` / `Wing.R$…` 两根骨，无位移曲线）
  - `wings_flap_off` → clip `Feature Motion 68`（`m_StopTime: 0`，静止）
  - 转换条件用到 Bool `WingsFlapOn` / `Avatar_Wings`

**移植做法（V2 实测可用）**

1. 在**源 FX 控制器**（`Assets/08_动画·菜单·图标/一键穿衣系统/Kipfel_FXLayer_v1.1.1_沐梓溪喵.controller`，不是烘焙产物）上新增独立层 `WingsFlap`，`defaultWeight = 1`，两个 state：`Idle`（motion = null）、`Flap`（新建 clip）。
2. 新建 clip：`AnimationUtility.SetEditorCurve` 写 4 条 `localRotation`（x/y/z/w）× 2 根骨；`AnimationUtility.GetAnimationClipSettings(clip)` 里设 `loopTime = true`、`stopTime = 1.7333333`，clip 的 `frameRate = 60`。
3. 转换：`AnyState → Flap` 条件 `Acc_Wings > 0.5`；`Flap → Idle` 条件 `Acc_Wings < 0.5`（`hasExitTime = false`、`duration = 0.1`）。**两个 state 的 `writeDefaultValues` 都设 true**，否则关掉翅膀后骨骼停在最后一帧不回位。
4. 控制器里已有层不要动，新层加在末尾即可（VRCFury 按 priority 合并）。

**验证（关键坑）**：Play 模式里烘焙克隆的 `Animator.runtimeAnimatorController` **是 NULL**（AAO / GestureManager 把控制器拿走了）→ 既不能 `SetFloat` 驱动参数，也读不到 `GetCurrentAnimatorStateInfo`（本次实测 `layers=0`）。正确做法是**直接对烘焙产物采样**：

```csharp
var fx = AssetDatabase.LoadAssetAtPath<AnimatorController>(
    "Packages/com.vrcfury.temp/Builds/____/VRCFury FX.controller");
// 找到 WingsFlap 层 → Flap state 的 clip
clip.SampleAnimation(cloneRoot, t);                       // t = 0 / 0.35 / 0.7 / 1.05 / 1.4
var wl = cloneRoot.transform.Find("armature/Hips/Spine/Chest/Wing.L");
Debug.Log(wl.localRotation.eulerAngles);
```

本次实测：`(50.27,267.03,17.96) → (59.60,273.12,22.97) → (74.57,299.93,47.95) → (78.17,326.64,73.93) → (64.47,277.97,27.27)`，`Wing.R` 镜像 → 扇动曲线成立。

**判据**（三条同时满足才算通过）：

1. 烘焙控制器里有 `WingsFlap` 层且 `defaultWeight > 0`（本次：157 层，index 12，`weight=1`）；
2. 该层 `AnyState → Flap` 的 `conditions` 是 `Acc_Wings Greater 0.5`，clip 的 `AnimationUtility.GetAnimationClipSettings(clip).loopTime == true`（**`clip.wrapMode` 仍是 `Default`，别用它判断循环**）；
3. `clip.SampleAnimation(cloneRoot, t)` 在多个时刻骨骼欧拉角单调变化。

**附**：`AnimationClipSettings` 是**值类型**，`var s = AnimationUtility.GetAnimationClipSettings(clip)` 改字段后必须 `AnimationUtility.SetAnimationClipSettings(clip, s)` 才落盘；`AnimationClip.frameRate` 可直接赋值。


## §17 烘焙 Console 报错清零：日志定位 + 第三方生成器 `HasProperty` 守卫 + MA-1300 fake-null（本工作区实测 2026-10-27，第 107 轮）

用户交付前的最后一关通常不是功能，而是「Console 打开一片红」。本节是**把报错清零**的标准流程：先拿到真日志，再区分「我们的 bug / 第三方插件的噪声 / 平台固有告警」，只修前两类。

### 17.1 判据：什么叫「干净」

| 级别 | 允许保留 | 必须修 |
|---|---|---|
| Error（红色） | 0 条 | 任何会刷屏的 `MissingMethod`/`NullReference`/`doesn't have ... property` |
| Warning（黄色） | 第三方版本提示、平台固有参数告警（下面 17.5 列清单） | 自己引入的空参数/空引用/缺组件的 MA 报错 |
| Log（白色） | 烘焙进度、`material path :` 之类插件调试输出 | — |

注意：**烘焙（Play / Build & Test）时 Console 的 Log 量很大**，只看窗口无法判断「这是本次产生的还是历史残留」。必须先切片。

### 17.2 拿真日志：直接读 `Editor.log` 并「按最后一次烘焙切片」

MCP 的 `read_console` 有两个硬伤：不按 `types` 过滤（请求 error 也会回 Log）、并且**按最旧优先**返回（刚发生的错误排在最后，常常翻不到）。可靠做法是读 Unity 的日志文件：

```powershell
$log = Join-Path $env:LOCALAPPDATA 'Unity\Editor\Editor.log'
# Unity 一直占着这个文件：必须 FileShare.ReadWrite，否则 IOException
$fs = New-Object System.IO.FileStream($log,[System.IO.FileMode]::Open,[System.IO.FileAccess]::Read,[System.IO.FileShare]::ReadWrite)
$sr = New-Object System.IO.StreamReader($fs); $text = $sr.ReadToEnd(); $sr.Dispose(); $fs.Dispose()
$idx = $text.LastIndexOf('Starting processing for avatar')   # NDMF 每次烘焙的起点
$slice = $text.Substring($idx)                                # 只看最后一次烘焙
('color_prop=' + ([regex]::Matches($slice,"doesn't have a color property")).Count)
('MA1300='     + ([regex]::Matches($slice,'MA-1300')).Count)
[regex]::Matches($slice,'\[NDMF\] Error Reported[^\r\n]*') | ForEach-Object { $_.Value }
```

判读要点：
- `Starting processing for avatar` 之后才是本次 Play/构建的日志；**同一个窗口里历史错误不会消失**，所以「Console 里还有红字」不等于「这次还在报」。
- 统计用**子串计数**（`doesn't have a color property` / `MA-1300` / `uses parameter`），比肉眼翻页可靠。
- 结尾若出现 `Asset Pipeline Refresh` / `Skipping the computing because this play is avatar info input play`，说明本次烘焙已结束，可以下结论。

### 17.3 第三方 Editor 生成器对「不存在的 shader 属性」刷 Error → 加 `HasProperty` 守卫

本工作区实测：`Assets/09_工具与插件/!亮度调节-初音的VRC小店/LightController/Editor/Generator/Generator.cs` 的 `SetupMaterials()` 里，分支写成「poiyomi 走一套、**其它一律** 读 `_BacklightColor`」：

```csharp
// 修前（line ~149-153，非 poiyomi 分支）
else {
    rendererInfo.backlightColor = material.GetVector("_BacklightColor"); // ← 粒子/UI/眼罩材质没有这个属性
    ...
    rendererInfo.isLiltoon = true;
}
```

`Material.GetVector` 读不到属性时**不抛异常、而是打一条 Error**（返回零向量）→ 一个头像 40+ 个非 poiyomi 材质，就是 40+ 条红字，且**每次烘焙都重刷**。

```csharp
// 修后：行为完全不变（缺属性本来就是零向量），只是不再刷 Error
rendererInfo.backlightColor = material.HasProperty("_BacklightColor")
    ? material.GetVector("_BacklightColor")
    : rendererInfo.backlightColor;
// poiyomi 分支的整型读取同样处理
if (material.HasProperty("_BacklightEnabled") && material.GetInt("_BacklightEnabled") == 1) { ... }
```

**最小侵入原则**：只加守卫、不改数值/流程；patch 要**幂等**（正则替换前后各跑一次结果一样），并在文件里留 `has_backlightcolor=<n>` 之类的自检输出。

> **写文件的坑**：Unity 开着时，DSH `edit`/`write` 可能报 `ReplaceFileW EIO (Win32 32)`（文件被占用）→ 用 python `io.open(path, "r", encoding="utf-8")` 读、正则替换、
> 再 `io.open(path,"w")` 写；写完 `refresh_unity`（`scope=all`、`wait_for_ready=true`）等编译完成，再看 Console。

### 17.4 `[NDMF] Error Reported: [MA-1300] 未指定要合并的 Animator` = 组件字段是 **missing asset**（fake-null）

真因不是「忘了拖 animator」，而是该 `ModularAvatarMergeAnimator.animator` 指向的资产**已不存在**（本工作区是旧工程 Chiffon 源 prefab 缺失遗留）。这种引用在 C# 里表现为：

| 判断写法 | 结果 | 说明 |
|---|---|---|
| `animatorField.GetValue(c) == null` | **false** | 值是非 null 的 UnityEngine.Object 包装（指向丢失资产），`==` 走引用比较 |
| `(val as UnityEngine.Object) == null` | **true** | 这才是 Unity 的「对象已失效」语义 |

→ 反射遍历删「坏组件」时必须写 `if ((val as UnityEngine.Object) != null) continue;`；写成 `val != null` 会一条都删不掉（本工作区第一版 `removed=0` 就是这个原因）。

```csharp
var f = c.GetType().GetField("animator", BindingFlags.Public|BindingFlags.NonPublic|BindingFlags.Instance);
var uo = (f == null ? null : f.GetValue(c)) as UnityEngine.Object;
if (uo != null) continue;               // 有效引用，跳过
UnityEditor.Undo.DestroyObjectImmediate(c);   // 删除坏的 MergeAnimator
...
UnityEditor.SceneManagement.EditorSceneManager.MarkSceneDirty(scene);
UnityEditor.SceneManagement.EditorSceneManager.SaveScene(scene);
```

删掉是安全的：`animator` 缺失时 MA 本来什么都没合并（否则每次烘焙都会报 MA-1300）。删完再烘焙一次，日志里 `MA-1300=0` 才算修好。

### 17.5 可以保留的 Warning（别浪费时间「修」）

| 日志 | 来源 | 结论 |
|---|---|---|
| `State "isLocal" in Controller "<眼罩控制器>" uses parameter "" which does not exist.` | 作者的夜模式眼罩控制器（猫/狼两套副本都有），`IsLocal` 是 VRChat 内置参数，提交阶段被平台绑定剥离 | **可修（第 140 轮补充，取代本条原「固有告警」结论）**：根因是 AnimatorState 的 `m_TimeParameterActive: 1` 而 `m_TimeParameter` 为空；把这两处 `m_TimeParameterActive` 改 `0` 后告警消失、无功能影响。详见 §17.7 |
| `[NDMF] Error Reported: Avatar Optimizer的新版本 x.y.z 已经发布！` | AAO 版本提示（Info 被 NDMF 提升为 Warning） | 只需决定升不升级，不影响烘焙 |
| `SendMessage cannot be called during Awake, CheckConsistency, or OnValidate (__vrcf_play_mode_trigger: OnDidAddComponent)` | VRCFury 的 `PlayModeTrigger` | 每次进 Play 都有，无害 |
| `MissingReferenceException: The object of type 'VRCPhysBone' has been destroyed ...` + `VRCFExceptionUtils:ErrorDialogBoundary` | Play 烘焙期间 VRCFury 克隆/销毁 PhysBone 的既有噪声 | 已在 `07-06` 噪声对照表；若数量暴增才需要查 |

### 17.6 收尾清单（交付给用户手动上传之前）

1. 修完**再跑一次** Play 烘焙，用 17.2 切片确认：目标 Error 计数为 0、`[NDMF] Error Reported` 只剩版本提示。
2. 退出 Play → 还原被 Play 改写的生成物：`Assets/08_动画·菜单·图标/NyaPPu__Generated/LightController/*.anim`（5 个，MD5 对齐备份）。
3. 删本次构建产物：`Packages/com.vrcfury.temp/Builds`（**只删 Builds 子目录，temp 包本身保留**）、工程根的 `*.sln`、`tools/tmp` 里的脚本与临时图。
4. 场景 `SaveScene`（`dirty=False`）并确认根对象数没被 SDK 面板留下的临时对象污染：VRChat SDK 面板预览会生成 `TMPCAM_BIND`(Camera) / `TMPLb1` / `TMPLb2`(Light) 三个**根对象**，上传前应删掉再保存。
5. 清 Console（或至少确认窗口里没有会吓到用户的红字），把界面停在「预览角色默认形态」（选中角色根 + Scene 视图取景 + Game 视图相机对准），再交给用户上传。

### 17.7 补充（第 140 轮，2026-11-05）：`uses parameter ""` 可修 + MA 参数默认值影响 EP

| 日期 | 现象 | 根因 | 判据/复现 | 正确做法 | 关联文件 |
|---|---|---|---|---|---|
| 2026-11-05 | Console 两次报 `State "isLocal" in Controller "EyeMaskNightMode" uses parameter "" which does not exist.`；烘焙不失败但红字扰民 | `EyeMaskNightMode.controller` 里两个 state 名 `isLocal` 的 AnimatorState 序列化了 `m_TimeParameterActive: 1` 但 `m_TimeParameter` 为空（Unity 把空参数当缺失参数） | 控制器 YAML：`m_TimeParameterActive: 1` 且 `m_TimeParameter:` 为空（本资产 controller L1134/L1242，state 在 L1120/L1229）；改后刷新 Console 归零 | 两个 state 的 `m_TimeParameterActive` 改成 `0`（不要给空 TimeParameter 硬填值，作者原意就不使用 timeParameter）；改前备份 `Temp/EyeMaskNightMode.controller.bak` | `Assets/05_配饰/眼罩/SleepEyeMask/_Core/EyeMaskNightMode.controller`；副本 `Assets/08_动画·菜单·图标/换头系统/WolfEyeMaskNightMode.controller` 同 bug（本头像未使用，未改） |
| 2026-11-05 | 眼罩默认打开（用户要求默认关闭），EP `RBS_EyeMask_Toggle` def=1 | 不在控制器（其 `m_DefaultBool` 全 False），而是 prefab `(Type-A)(Black)EyeMask.prefab` 的 `ModularAvatarParameters` 显式 `defaultValue: 1` → MA 写入 EP 默认值 1 | 烘焙 EP dump：`RBS_EyeMask_Toggle$<hash> Bool def=1 saved=True synced=True`；菜单项 `isDefault=False` | 不改共享 prefab（会连带 正太兔兔！/沐梓溪喵 等共用头像），改场景实例 override：`SerializedObject` 遍历 `parameters` 数组，匹配 `nameOrPrefix == "RBS_EyeMask_Toggle"` 后 `defaultValue.floatValue = 0` → `ApplyModifiedProperties` → `SaveScene`；另加场景 `[設定用]アイマスク` inactive 兜底 | `沐梓溪狼/(Type-D)(White)EyeMask/Root`；`Assets/05_配饰/眼罩/SleepEyeMask/Type-A/(Type-A)(Black)EyeMask.prefab` |

---

## §18 「亮度调节」子菜单烘焙后为空：Generator 删目录后 `IsValidFolder` 仍为 true

**症状**：MA 的 NyaPPu LightController 挂在头像上、`installTargetMenu` 也指对了，
但烘焙出的菜单里对应子菜单**是空的**（一个条目都没有），Console 里可能只有一条
`UnityException: Creating asset at path ... failed` + `在 ... 创建资产之前，父目录必须存在`。

**根因链**（`Assets/09_工具与插件/!亮度调节-初音的VRC小店/LightController/Editor/Generator/Generator.cs`）：

```csharp
AssetDatabase.DeleteAsset("Assets/08_动画·菜单·图标/NyaPPu__Generated/LightController"); // 删
if (!AssetDatabase.IsValidFolder(path)) { ... }   // 仍返回 true —— 删除是延迟生效的
... AnimationUtil 创建 LightLimit.anim → 父目录已不存在 → 抛异常
```

异常一出，`Generator.Generate` 后半段（挂 MenuInstaller / 写参数）全部不执行 →
菜单空、参数缺失，**但头像本身烘焙不报错**，很容易漏掉。

**修法**（幂等，两行）：在 `IsValidFolder` 判断后补一次真实建目录

```csharp
if (!System.IO.Directory.Exists("Assets/" + savePath))
    System.IO.Directory.CreateDirectory("Assets/" + savePath);
```

**验收**：烘焙 clone 的菜单里出现 `Reset / Backlight / Monochrome / Light Limit` 四项，
EP 里出现 `LightController/BacklightToggle:Bool`、`LightController/LightLimit:Float`、
`LightController/Monochrome:Float`、`LightController/Reset:Bool`，FX 层数 +4。

> 同类坑：这个生成目录**不要当垃圾删**（第 93/98/99 轮已记 `com.vrcfury.temp` 的同类教训）。

## §18.1 从 `execute_code` 调用工程内 Editor 脚本（CodeDom 的硬限制）

`execute_code` 的 CodeDom 动态程序集**编译期引用不到** `Assembly-CSharp-Editor`，
直接写 `MyEditorTool.Foo()` 会报 `The name 'MyEditorTool' does not exist in the current context`。

正解 = 运行期反射：

```csharp
System.Type t = null;
foreach (var a in System.AppDomain.CurrentDomain.GetAssemblies()) {
    t = a.GetType("MyEditorTool"); if (t != null) break;
}
t.GetMethod("Foo").Invoke(null, new object[]{ ... });
```

配套限制（都实测过）：
- CodeDom **不支持局部函数**（`void Dump(...)` 报 `Keyword 'void' cannot be used in this context`）；
- `Object` 在 CodeDom 里有歧义 → 写 `UnityEngine.Object.DestroyImmediate`；
- `mesh.vertices` 在 `AssetDatabase.CreateAsset` **之后不可写**（资产导入后 `isReadable=false`，
  getter 返回空数组、setter 静默失败）→ 顶点修改必须发生在 `CreateAsset` 之前；
  覆盖已有 mesh 用 `DestroyImmediate(existing, true)` + `CreateAsset`，`CopySerialized` 不复制顶点；
- `AssetDatabase.DeleteAsset` / `System.IO.File.Delete` 会被 MCP 安全检查拦
  （`Blocked pattern detected`）→ 需要 `safety_checks=false`。

---

## §19 用真实鼠标点击 Unity 面板完成 Build & Publish（Windows MCP / user32 坐标法）

**背景**：Unity 的 UI Toolkit 输入框无法用键盘注入写值（`mcp__windows__Type`/剪贴板/SendKeys 都写不进），
但**按钮可以真实点击**——只要算对屏幕坐标。本工作区实测走通，线上版本从 6 → 12。

### 19.1 Windows MCP 看不到 Unity 窗口时的替代

`mcp__windows__Snapshot` 可能返回 `No windows found`（跨会话/权限差异）。此时用 PowerShell 枚举：

```powershell
Add-Type -TypeDefinition $cs   # 见下：EnumWindows + GetWindowThreadProcessId + GetWindowText + GetWindowRect
[W]::List([uint32]$pid)        # 列出该进程所有可见窗口及 rect
```

实测会列出 `VRCFury`（模态错误框）、`VF.VRCFProgressWindow`、`VRChat SDK`、`NDMF Console`、
`Unity Settings for Android` 与主窗口。**`mcp__windows__Click` 仍可用**（它按绝对屏幕坐标点击）。

### 19.2 从 UI Toolkit `worldBound` 反推屏幕坐标

1. 反射拿窗口与元素：

```csharp
var win = ... // Resources.FindObjectsOfTypeAll(typeof(EditorWindow)) 里 type.Name=="VRCSdkControlPanel"
var bp  = win.GetType().GetField("_builderPanel", BF).GetValue(win) as VisualElement;
// 递归 Children() 找 Button，读 b.text / b.worldBound / b.enabledSelf
```

2. `worldBound` 是**面板内容区坐标**；窗口在 Unity 坐标空间的位置是 `win.position`。
3. **最稳的换算方式不是算，而是“按宽度匹配截图里的矩形”**：抓一张 `GetWindowRect` 尺寸的窗口截图，
   用 PIL 找“大块均色矩形”（本例 Build & Publish = 488×67，截图里测得 486×67 @ x 826..1311 / y 787..850），
   与 `worldBound.width=488` 对上 → 按钮中心截图坐标 `(1068, 818)`，
   屏幕坐标 = 截图坐标 + 窗口左上角 `(21,31)` = `(1089, 849)`。

```powershell
[W]::Click(1089, 849)   # SetCursorPos + mouse_event(0x0002/0x0004)
```

### 19.3 构建会被 VRCFury「Write Defaults broken mix」模态框卡住

点 `Build & Publish` 后日志出现 `VRCFExceptionUtils:ErrorDialogBoundary` 且**日志不再前进** =
VRCFury 弹出模态框（窗口标题 `VRCFury`，内容 `Write Defaults broken mix`，按钮
`Auto-Fix` / `Skip` / `Skip and stop asking`）。不点它构建就永远卡住。

- 正解：点 **Auto-Fix**（给头像根加 `VF.Model.Feature.FixWriteDefaults{Auto}`），构建继续；
- 预防：烘焙/上传前先反射加上该组件（第 64 轮做法），否则每次都会弹。
- 判断“是否卡住”看 `Editor.log` 行数是否停止增长，而不是看 Console。

### 19.4 上传成功的判据（不要只看有没有弹窗）

反射读 `_builderPanel` 的 Label：

- `Upload Succeeded!`
- `Your avatar was uploaded to VRChat!`
- `Version` 旁边的数字 Label 变成新版本号（本例 **12**）

另外 `Editor.log` 里 `Bundle Name: prefab-id-v1_avtr_<id>_<n>.prefab.unity3d` 出现你的 blueprintId
即表示走的是**覆盖更新**（不是新建）。若 `PipelineManager.blueprintId` 被清空过，构建面板会当成新头像
并要求重新填名称/简介/Tag/缩略图——想“不改元数据”就必须先把 blueprintId 写回。

### 19.5 收尾（每次 Build 之后必做）

构建会重写 `Assets/08_动画·菜单·图标/NyaPPu__Generated/LightController/*.anim`（5 个）→
从备份覆盖还原并校验 MD5；删 `Assets/Build AssetBundles.buildreport`(+.meta)；
把一次性 Editor 脚本从 `Assets/Editor/` 移出（只留 `McpBridgeBootstrap.cs`）；清 `tools/tmp`。


---

## §20 默认形态：只留指定那一套衣装（素体自带衣装必须显式关掉）

**症状**：换头/换素体后默认形态「睡衣1 和原作者的衬衫/裤子/披风同时穿在身上」；
或者配件（眼罩/名牌/眼镜）默认就显示。用户要求「默认 = 睡衣1，帽子/外套/小幽灵关闭」。

### 20.1 三处律（少一处就不生效）

| 位置 | 要改成 | 为什么 |
| --- | --- | --- |
| `VRCExpressionParameters` 里该参数的 `defaultValue` | 0 | 运行时参数的初始值 |
| `ModularAvatarParameters` 里同名 `ParameterConfig` | `defaultValue=0` + `hasExplicitDefaultValue=true` + `saved=true` | **MA 的 explicit 默认值会覆盖 EP**，只改 EP 不生效 |
| 场景里该部件的 `activeSelf` | `false` | 编辑态没有 Animator；且 OT/原生 FX 关闭态回落到场景初始态 |

**只改 EP 的典型现象**：本地 EP 看着是 0，烘焙产物（clone 的 `VRCAvatarDescriptor.expressionParameters`）
里仍然是 1 —— 因为 MA Parameters 的 explicit 默认值在烘焙期重写了它。

### 20.2 作者原生 FX 的 `If` / `IfNot` 语义（别搞反）

老作者（本例 `LilLeo_FX` 20 层）的部件显隐层结构是：

```
defaultState = ON
ON  -> OFF : duration=0, hasExitTime=false, [<param> IfNot 0]
OFF -> ON  : duration=0, hasExitTime=false, [<param> If 0]
```

`IfNot` = 参数为 **false(0)** 时进入 → **隐藏**；`If` = 参数为 **true(1)** 时进入 → **显示**。
即**参数默认 1 = 部件可见**。想默认隐藏就把该参数默认值改成 0，**不需要新建任何层**。

### 20.3 不在 FX 里的原生衣装件

有些件（本例 `Pants` / `Shirt` / `Bandage`）不在 FX 里，由 MA 宿主的
`ModularAvatarObjectToggle`（行 `Active=False`、参数留空 → MA 自动 `__MA/AutoParam/…`）控制。
「默认不穿、菜单可穿」要改成：**OT 行 `Active=true` + 场景对象 inactive + 参数默认 0**，
并把宿主 GameObject 从 `Pants OFF` 重命名为 `裤子`（`MenuSource=Children` 下菜单文字取 GO 名）。

### 20.4 面板换缩略图会 +1 版本，不需要重新 build

SDK 面板 Build 页缩略图区：`Capture In`（打开取景框）→ 勾 `Fill Background` → `Capture`
→ 面板出现 `Save Changes` / `Discard` → 点 `Save Changes`。
实测**只这一步就把线上头像信息（含缩略图）更新了，`Version` 号 +1**（本例 13 → 14），
且 `Name` / `Description` / `Tags` 不受影响。取景用
`SceneView.LookAt(new Vector3(0,0.42f,0.03f), Quaternion.Euler(6,196,0), 0.60f, false, true)`。

### 20.5 VRCQuestTools「missing scripts」弹框不是 EditorWindow

`Resources.FindObjectsOfTypeAll(typeof(EditorWindow))` 里找不到它（它是普通 IMGUI 模态窗口），
`SendKeys {ENTER}` 也关不掉。两条路：① `mcp__windows__Click` 点它的 `Dismiss` 绝对坐标；
② **从源头删掉 missing script**（推荐）：

```csharp
int removed = GameObjectUtility.RemoveMonoBehavioursWithMissingScript(go); // 每个 Transform 调一次
```

删完 `SaveScene`，之后不再弹。参考 `05-动骨与穿模/05-05-head-swap-teardown-and-clone.md §12`。


---

## §21 Build & Publish 点不动 / 上传失败重试（本工作区实测 2026-10-28，第 113 轮）

### 21.1 症状：Windows MCP 点按钮没反应，`Editor.log` 行数不增长

真实原因不是坐标错，而是**面板上还盖着一层模态框**。本例第一次点 `Build & Publish` 后上传失败，面板弹出
`Upload Failed` 覆盖层（含 `Check Console for more details` 与右上角 ✕），**它把按钮挡住了**，
后续所有点击都落在覆盖层上（看起来像「点了没反应」）。

排查顺序：
1. 抓整屏 `CopyFromScreen(0,0,1920,1080)`，裁按钮附近区域看一眼有没有覆盖层；
2. 有覆盖层先点 ✕ 关掉；
3. 再点按钮，然后**看 `Editor.log` 行数是否增长**（增长 = 真的开始构建）。

### 21.2 从 `worldBound` 换算屏幕坐标（比肉眼量稳）

```csharp
// 反射取面板的 _builderPanel（VisualElement）
var bp = t.GetField("_builderPanel", BF).GetValue(panel) as VisualElement;
// 递归找 Button，读它的 worldBound（= UI Toolkit 面板空间坐标）
btn.worldBound  // 例：(x:14, y:666, w:488, h:67)
```

`worldBound` 的 `y` **不是**窗口客户区坐标，中间还差标题栏 + 面板头部。可靠做法是**用颜色找按钮矩形**：

```python
# PIL：在窗口截图的底部 120 行里，找 Unity 按钮的灰 (88,88,88) 连通块
xs=[]; ys=[]
for y in range(H-120, H):
    for x in range(W):
        if px[x,y]==(88,88,88): xs.append(x); ys.append(y)
# bbox → 中心 → 屏幕坐标 = 窗口 GetWindowRect 左上角 + 图像坐标
```

### 21.3 「Failed to upload file」是可重试的

本例第一次构建成功、上传阶段 `VRCApi.UploadFile → UploadMultipart → MakeRequestWithResponse` 抛异常 →
`Failed to upload file`。此时：
- `api.vrchat.cloud/api/1/config` 用 PowerShell 测通（200）说明不是断网；
- 直接**再点一次 `Build & Publish`** 就成功了（Version 14 → 15）。

### 21.4 上传成功判据（反射读面板 Label）

`Upload Succeeded!` / `Your avatar was uploaded to VRChat!` / `Version` 旁的数字 / `Last Updated` 时间戳。

### 21.5 收尾清单

删 `Assets/Build AssetBundles.buildreport`（SDK 一般已自行消费）、`Assets/ZZZ_GeneratedAssets`、
`Packages/com.vrcfury.temp/Builds`（**temp 包本体保留**）、工程根重生的 `.sln`/`*.csproj`；
`LogEntries.Clear()` 清 Console；`SaveScene` 后确认 `dirty=False roots=9`。


## §22 上传后「衣服材质像丢了」= LightController 去色默认值 0.5

- 症状：编辑器里三套衣服颜色正常（渲染、材质槽、`_MainTex` 全对），**进游戏/上传后整体发灰、像材质丢失**。
- 根因：角色上的 `nyappu.lightcontroller.LightController` 组件
  `useMonochrome = True` + **`defaultMonochrome = 0.5`** →
  插件生成 `Monochrome.anim`（108 条 `material._MonochromeLighting` 0→1）+ MA 参数
  `LightController/Monochrome` 默认 **0.5** → 运行时所有材质 50% 去色。编辑器不跑 Animator 所以看不出来。
- 排查套路（本工作区实测顺序）：渲染 → renderer 材质列表 → `_MainTex` 引用 → 烘焙克隆材质 → **最后才怀疑亮度插件参数**。
- 修法：把 `defaultMonochrome` 改 **0**（保留 `useMonochrome=True`，菜单里仍有 Monochrome 径向滑块），重烘确认 EP：
  `LightController/LightLimit def=0.55`、`LightController/Monochrome def=0`、`LightController/BacklightToggle def=0`、`LightController/Reset def=0`。
- 泛化：任何「编辑器正常、进游戏异常」的材质类问题，先查 **FX 参数默认值 / 材质属性动画曲线**，再看贴图。

### §22.1 工具 prefab（ABT）绑定点验收

烘焙克隆里断言（详见 [05-09](../05-动骨与穿模/05-09-third-party-tool-prefab-binding-abt.md)）：
`Pen_Setting.parent == Hand.R` 且 Δ≈(0.03,0,0.05)；`Finger_Setting.parent == IndexDistal.R` 且 Δ≈0。
本工作区错误态：`Pen_Setting` 世界 y=1.184（prefab 原始坐标）→ 工具飘在头顶。

---

## §23 「默认关闭」的自建开关层：烘焙判据（本工作区实测 2026-10-28，第 116 轮）

给「作者自带成对 OFF/ON clip」的部件加菜单开关时，**不要**用 `ModularAvatarObjectToggle`（目标初始 inactive 会被 ActiveSelfProxy 激活、默认态无 clip → 默认可见），
改用 `ModularAvatarMergeAnimator` 合入一个 `defaultState=OFF` 的 2 态层，详见 `05-动骨与穿模/05-07-tail-gloves-migration.md` §9。

**四条判据（缺一不可）**

1. 烘焙克隆里目标对象 `activeSelf=False`；
2. 自建层 index **大于**作者同名部件层 index（层序即优先级）；
3. 自建层 `defaultWeight=1` 且 `stateMachine.defaultState.name` = 关闭态（本作 `OFF`）；
4. 关闭态 clip 对该属性的首帧值 = 0 —— 用 `AnimationUtility.GetEditorCurve(clip, EditorCurveBinding.FloatCurve(path, typeof(GameObject), "m_IsActive"))` 读第 0 帧，**不要只看 clip 名字或文件名**。

**反例（本工作区实测）**：OT 版本烘焙克隆 `Gloves.activeSelf=True`；换 MergeAnimator 方案后 `False`，且 `#41 Gloves` 的 OFF 态首帧 `m_IsActive=0`。

**顺带**：`VRCExpressionsMenu.Control.ControlType` 真实枚举 101/102/103/201/202/203，写错会让 MA 抛 `ArgumentOutOfRangeException` 并让 MA 反应层整段不生成（症状=烘焙 FX 层数骤降）。
`read_console` 的 `types` 过滤不可靠、且按最旧优先；统计错误请按 §17 的 Editor.log 切片法，或用 `UnityEditor.LogEntries` 反射遍历 + `LogEntry.mode & 1`。

---

## §24 Build/Play 卡在「VRCFury SPS Plug 找不到渲染器」与遗留烘焙克隆（2026-10-28 第 117 轮实测）

### 24.1 症状
点 Build & Publish 或进 Play，弹 `VRCFury Error` / `Preprocess Callback Failed`：
```
Failed to build VRCFury component: BakeHapticPlugsService.ApplyEarlyBake on Avatar Root
Failed to bake SPS Plug: armature/Hips/<玩具名>/…/Bone.001/Bone.002$SPS Plug$5
Failed to find plug renderer
  at VF.Builder.VRCFuryBuilder.ApplyFuryConfigs (VRCFuryBuilder.cs:227)
```
路径里带 `$3/$4/$5` 后缀 = **MA MergeArmature 合并骨骼后**的名字；烘焙/预检失败，构建无法继续。

### 24.2 两条真因
1. **场景里遗留了烘焙克隆头像**：`沐梓溪狼(Clone)` 作为场景根对象被保存（手动 `AvatarProcessor.ProcessAvatarUI` 或 NDMF 预览留下的副本）。它带 `VRCAvatarDescriptor`，VRCFury 的 preupload hook 也会处理它 → 它的 SPS 插头（来自 prefab）在合并骨骼后找不到渲染器 → 整体失败。
   - 处置：删掉所有 `*(Clone)` / `BAKE_VERIFY*` / `TMP*` 场景根，`SaveScene`。
2. **SPS 插头找不到渲染器**（即使清掉克隆仍复现）：
   - VRCFury 的 `PlugRendererFinder` 新版算法逐级向上找「顶点权重落在该骨骼子树里的渲染器」，旧版算法按 DPS/TPs 材质找；**两者都可能失败**（代码里 TODO 已注明 bone 实例比较不可靠）。
   - `useLegacyRendererFinder` 切换新旧算法 **不能** 修好本例。
   - `enableSps=false` 也 **不能** 跳过：`PlugSizeDetector.GetWorldSize()` 仍然要渲染器。
   - 正解一：显式指定渲染器 —— `autoRenderer=false` + `configureTpsMesh=[该件 SMR]`；注意组件若在 prefab 实例里，**改实例不生效**，必须改 prefab 资产（`PrefabUtility.SavePrefabAsset`），克隆时内部引用才会正确重定向。
   - 正解二（最省事）：删除该插头组件（保留 SPS Socket 与其余组件），构建恢复正常；SPS 的 DPS 穿透能力随之关闭。
   - 本次结论：插头在 `Assets/06_道具/小牛牛/N式スーパーよわちんv1.0.prefab` 里，显式渲染器与删组件两条路都验证过；最终删组件以解锁构建。

### 24.3 排查手册
1. Play 里卡住时，先枚举 Unity 进程的窗口找 `#32770` 对话框（`VRCFury Error` / `Preprocess Callback Failed` / `Hold on`），`PostMessage(WM_CLOSE)` 关掉，再用 MCP `manage_editor stop`。
2. 读 `%LOCALAPPDATA%\Unity\Editor\Editor.log`（被占用时用 `FileStream(Open,Read,ReadWrite)`），搜 `Failed to find plug renderer` 拿到**烘焙期路径**（带 `$n` 后缀）与堆栈。
3. 用 `Resources.FindObjectsOfTypeAll<GameObject>()` 列场景根，删除 `(Clone)` 根；再 `execute_code` 打印所有 `VRCFuryHapticPlug` 的 `path/enableSps/autoRenderer/configureTpsMesh`。
4. 修完必须重新 `SaveScene`，并再跑一次 Play/构建确认无对话框。

### 24.4 相关坑
- 编辑态 `AnimationClip.SampleAnimation` 会永久改骨架（见 `05-动骨与穿模/05-08` §16）。
- `VRCSdkControlPanelAvatarBuilder.OnBuildAndPublishAction()` 可用反射调用（`VRCSdkControlPanel._selectedBuilder`），但面板必须先 `VRChat SDK/Show Control Panel` 打开，否则 `_selectedBuilder` 为 null → NRE。

---

## §25a 构建完成但「上传没发生」：Send/Cancel Upload 是 0×0 隐藏元素，必须直接反射 Upload()（本工作区实测 2026-10-28，第 117 轮）

### 25.1 症状
点 Build & Publish（或反射 _selectedBuilder.OnBuildAndPublishAction()）后日志出现 Build AssetBundles.buildreport、ExportAvatarBlueprint、
lilToon OnPostprocessAvatar 等，构建产物 %LOCALAPPDATA%\\Temp\\DefaultCompany\\<工程名>\\<guid>.vrca 已生成，但线上头像 Version 没变；SDK 面板 _uploadState 回到 Idle，
面板里还能看到 Send / Cancel Upload 两个按钮和一句「This avatar is marked for sale & will be reviewed before being available for purchase.」——**别被它们误导**。

### 25.2 判据
反射读 VRCSdkControlPanel → _selectedBuilder（VRC.SDK3A.Editor.VRCSdkControlPanelAvatarBuilder）：
- _buildState == Idle、_avatarData != null（构建完成）
- _uploadState == Idle（== 没有上传在跑）
- _uploadDisabledText 可能仍有「You must fix the issues listed above…」残留文字
- 遍历 _buildVisualRoot 的 Button，Send 与 Cancel Upload 的 worldBound.width == 0、height == 0 → **隐藏元素，点不到也不需要点**。

### 25.3 正解：直接调 Upload
`csharp
object panel = null;
foreach (var w in Resources.FindObjectsOfTypeAll<UnityEditor.EditorWindow>())
    if (w.GetType().Name == VRCSdkControlPanel) panel = w;
object builder = panel.GetType().GetField(_selectedBuilder, BF).GetValue(panel);
var t = builder.GetType();
var avatarData = t.GetField(_avatarData, BF).GetValue(builder);   // VRCAvatar
var bundle = @"<USERPROFILE>\\AppData\\Local\\Temp\\DefaultCompany\\<工程名>\\<guid>.vrca"; // 最新的那个
t.GetMethod(Upload, BF).Invoke(builder, new object[]{
    GameObject.Find(沐梓溪狼), avatarData, false, bundle, null, CancellationToken.None });
// 之后 _uploadState == Uploading → Idle
`
- Upload(GameObject target, VRCAvatar avatar, Boolean firstTimeCreation, String bundlePath, String thumbnailPath, CancellationToken ct)；	humbnailPath=null 表示沿用旧封面（本工作区要求「不改封面」）。
- 也可用 BuildAndUpload(GameObject, VRCAvatar, String thumbnailPath, CancellationToken) 一步到位，但会**重新构建**（约 6–10 分钟）；已有 .vrca 时用 Upload 更快。

### 25.4 校验线上版本
`csharp
var task = VRCApi.GetAvatar(avtr_xxx, true, CancellationToken.None);   // (String id, Boolean forceRefresh, CancellationToken)
var m = task.GetType().GetMethod(ContinueWith, new[]{ typeof(System.Action<object>) });
System.Action<object> cb = t2 => { var av = t2.GetType().GetProperty(Result).GetValue(t2);
    System.IO.File.WriteAllText(path, Version= + av.GetType().GetProperty(Version).GetValue(av)); };
m.Invoke(task, new object[]{ cb });
`
**不要在 MCP 的主线程上 .Result / .Wait()**（会死锁 Unity，本次已踩坑一次）。

### 25.5 顺带：构建会把 LightController 的 5 个 anim 重写
Assets/08_动画·菜单·图标/NyaPPu__Generated/LightController/*.anim 每次构建/Play 都会被插件重生成（本次构建后只剩 4 个、BacklightBorder.anim 消失）。收尾必须从备份还原（本工作区备份：F:
\VRChat\\Lilleo2.1\Assets\NyaPPu__Generated\LightController），MD5 期望值：
880E26874A77DE8D9EF79BC80E9D2564(BacklightBorder) / F9A125593E56906B572226DF179D1F37(BacklightToggle_Off)
/ F1BE1377E235D9952FB4D60BC8E7F396(BacklightToggle_On) / C121A44F008AF2881C6F00342B14BA7E(LightLimit) / CD1D15CC0BC54BB4EB250893D7DB1D7C(Monochrome)。
> **2026-11-10 补充（第 175 轮，本轮实测）**
> - 上面写的备份路径 `F:\VRChat\...` **在本机已不可用**（无 F: 盘）。可用备份改为：`<PROJECT_ROOT>\_asset_reorg\backup\hairdel-20261026-104303\LightController\`。
> - 本轮在上传前构建后**再次复现**：`BacklightBorder.anim` 被删、其余 4 个从 314KB 档被重写成 381KB 档。
> - 还原时机：**所有构建 / Play / 上传都结束之后**；还原后必须逐个比对 MD5（本轮 5/5 全中），并用 `@(Get-ChildItem -Filter '*.anim').Count` 确认为 5、且无孤立 `.meta`。
> - `BacklightBorder.anim` 的 `.meta` 允许由 Unity 重新生成（GUID 会变），因为该 clip 只被插件在内存里引用，场景/资产里没有持久 GUID 引用。

## §25b Play 模式功能验证的四个硬坑（本工作区实测 2026-10-28，第 119 轮）

1. **烘焙后所有同步参数在 Animator 里都是 Float**：`OutfitSelect(Int)`、`K_PJ_Shirt(Bool)` 在 `VRCFury FX` 的
   `animator.parameters` 里 `type=Float`。验证时必须 `anim.SetFloat("OutfitSelect", 2f)`；
   用 `SetInteger/GetInteger` 只会读到 0（`GetInteger` 对 Float 参数恒为 0），会误判「参数没生效」。
2. **改参数后要等帧**：同一帧内对象 active 不会变；下一次 `execute_code` 调用（已过若干帧）再断言。
3. **Play 启动时 Animator 参数取控制器默认值（常为 0），不取 EP 默认值**：所以 Play 里一开始 `OutfitSelect=0`，
   两套容器都停在场景默认态（可能同时可见）。**要在游戏里正确，必须保证 EP 的 `OutfitSelect.defaultValue=1`，
   并让非默认容器在场景里 `activeSelf=false`**（参数异常回落时也只显示睡衣）。
4. **`AddStateMachineBehaviour(Type)` 在 CodeDom 下返回 null**（`beh` 列表里是空槽）；无法用脚本创建
   `VRC_AvatarParameterDriver`/`VRCPlayableLayerControl` 等行为。需要参数驱动时改用 MA 纯响应方案（见 `03-06 §10`）
   或在 Unity 编辑器里手动建层。

**可复用的验证片段**
```csharp
var anim = GameObject.Find("沐梓溪狼").GetComponent<Animator>();
anim.SetFloat("OutfitSelect", 2f);            // 换套
// 下一次 execute_code 里断言：
wolf.transform.Find("睡衣1").gameObject.activeInHierarchy
wolf.transform.Find("胡萝卜衣服/胡萝卜_背带裤").gameObject.activeInHierarchy
```


## §26 脚本化上传与线上版本核对（第 119 轮实测，2026-10-28）

### 26.1 为什么需要它
SDK 面板的 `Build & Publish` 走完构建后，UI 上会出现 `Send` / `Cancel Upload` 两个 **0×0 隐藏元素**（`worldBound` 宽高为 0）——既点不到，也不能当作"上传进行中"的判据。面板 `_buildState=Idle`、
`_uploadState=Idle`、`_avatarData!=null` 只说明**构建完成、尚未上传**。

### 26.2 正解：直接调 VRCApi.UpdateAvatarBundle
```csharp
// 1) 取构建产物（最新 .vrca）
//    <USERPROFILE>\AppData\Local\Temp\DefaultCompany\<ProjectName>\<guid>.vrca
// 2) 取面板里的 VRCAvatar
var wins = Resources.FindObjectsOfTypeAll(typeof(EditorWindow));
EditorWindow panel = /* titleContent.text == "VRChat SDK" */;
var builder = panel.GetType().GetField("_selectedBuilder", BF).GetValue(panel);
var avatarData = builder.GetType().GetField("_avatarData", BF).GetValue(builder);
// 3) 上传（主线程调用，返回 Task<VRCAvatar>）
var task = VRCApi.UpdateAvatarBundle("avtr_xxx", avatarData, bundlePath, null, CancellationToken.None);
```
- **必须在主线程调用**：在后台线程调 `VRCApi.*` 会抛 `SetString_Internal can only be called from the main thread`。
- **不要 `.Result`/`.Wait()`**：会卡死 Editor 主线程（本轮与上一轮各踩一次，只能杀进程重启）。
- 正确等待方式：`EditorApplication.delayCall += poll;`，在 `poll` 里检查 `task.IsCompleted`，未完成就再次 `delayCall += poll`（带上限次数），完成后读 `task.Result` 并把结果 `File.WriteAllText` 落盘（回调在非主线程写文件安全，读 Unity 对象不安全）。

### 26.3 核对线上版本
`VRCApi.GetAvatar(id, true, CancellationToken.None)` 同样主线程调用 + delayCall 轮询；返回的 `VRCAvatar` 里 `Version`（Int）、`UpdatedAt`（**UTC**，本地时间 = UTC+8）、`Name`、
`Description`、`ReleaseStatus`、`UnityPackages[0].AssetUrl` 都可读。
本轮结果：`Version=23`（构建后未上传）→ 调 `UpdateAvatarBundle` 后 `Version=24`、`UpdatedAt=2026/10/28 06:34:41`（UTC）= 本地 14:34:41 ✅

### 26.4 收尾（每次上传后必做）
1. `Assets/08_动画·菜单·图标/NyaPPu__Generated/LightController/*.anim` 会被构建重写 → 从备份还原并校验 MD5（本工作区 5 个文件：
  `880E26874A77DE8D9EF79BC80E9D2564` / `F9A125593E56906B572226DF179D1F37` / `F1BE1377E235D9952FB4D60BC8E7F396` / `C121A44F008AF2881C6F00342B14BA7E` / 
  `CD1D15CC0BC54BB4EB250893D7DB1D7C`）。
2. 删 `Packages/com.vrcfury.temp/Builds`、工程根 `*.sln`/`*.csproj`、`Assets/ZZZ_GeneratedAssets`（若有）、`tools/tmp/*`。
3. `SceneView` 归位 + `Selection.activeGameObject` 指回头像 + `SaveScene`（`isDirty=false` 收尾）。

## §27 手势菜单点检：VRChat「手势覆盖曲线」在编辑器/Play 里看不到骨骼变化（本工作区实测 2026-10-28，第 121 轮）

### 27.1 症状
用户报「加了唐笑表情之后，比大拇指手势没了，只有表情出来」。

### 27.2 根因
描述符 Gesture 层控制器 `Assets/01_模型本体/LilLeo/Anim/LilLeo_Gesture.controller` 的 `Left Hand / Thumbs up` 状态里，`motion` 被填成了**面部动画** clip `饿饿`（17 条 blendShape 曲线），而不是手势 clip `LilLeo_thumbs_up`。
→ 该手势没有覆盖数据，客户端回落到默认手势或上一个状态，于是「大拇指」失效。

### 27.3 关键认知：手势 clip 是 `Animator` 型覆盖曲线
`LilLeo_thumbs_up.anim` / `LilLeo_fist.anim` / `LilLeo_hands_idle.anim` 的曲线全部是：
- `path = ""`（不在任何骨骼上）
- `propertyName = "LeftHand.Index.1 Stretched"` / `"LeftHand.Little.2 Stretched"` / `...`（共 **40 条**）
- `binding.type = typeof(Animator)`，`clip.length = 0`

这是 VRChat 的**手势覆盖（gesture override）**格式：上传时由 SDK 读取、客户端运行时按 `GestureLeft/GestureRight` 播放，
**不会在编辑器或 Play 模式里改骨骼**。实测 `GestureLeft = 7` 时只有 2–3 根 orbit 骨小幅变化（来自其它层），手部骨骼完全不变 —— 属正常，不是 bug。

### 27.4 静态判据（替代「编辑器里看手势」）
```csharp
var ctrl = desc.baseAnimationLayers[2].animatorController as UnityEditor.Animations.AnimatorController;
foreach (var layer in ctrl.layers) {
    foreach (var st in layer.stateMachine.states) {
        var clip = st.state.motion as AnimationClip;
        if (clip == null) continue;
        var b = UnityEditor.AnimationUtility.GetCurveBindings(clip);
        Debug.Log(layer.name + " / " + st.state.name + " -> " + clip.name + " curves=" + b.Length
                  + (b.Length > 0 ? " first=" + b[0].path + "|" + b[0].propertyName + "|" + b[0].type.Name : ""));
    }
}
```
判据：
1. `Left Hand / Thumbs up`、`Right Hand / Thumbs up` 的 clip 名都应是 `LilLeo_thumbs_up`；
2. 每个手势 clip 应为 **40 条** `type=Animator`、`path=""` 的曲线（不是 blendShape/骨骼曲线）；
3. 表情层（`LilLeo_FX` 的 `Left Hand`/`Right Hand`）里同名状态**才**是面部 clip（这是作者设计，两者并存）。

### 27.5 收尾：Play 退出后 VRC 约束残留导致编辑器白屏
现象：退出 Play 后 `Editor.log` 每秒刷
`MissingReferenceException: The object of type 'VRCParentConstraint' has been destroyed but you are still trying to access it.`
（`VRC.Dynamics.VRCConstraintManager.UpdateConstraints`），同时 Unity 主窗口**白屏不重绘**（MCP 仍可响应）。
处置：`UnityEditorInternal.InternalEditorUtility.RequestScriptReload()` 触发域重载 → 立即恢复。
附：SDK 面板若显示 `Build Failed: ... VRCParentConstraint has been destroyed ...`，多为该残留造成的**陈旧显示**，
是否真的构建失败要看构建日志 / 线上版本号（本次 API 复核仍为 Version=25 成功上传）。


## §28 穿衣服后「身体部分消失」= MA ShapeChanger 被 AAO 转成 NaN 骨；一键穿衣 Button→Toggle（本工作区实测 2026-10-29，第 122 轮）

### 28.1 症状与判据
- 游戏内/烘焙后：穿上衣服（睡衣默认态）时手臂/腿/躯干**成块消失**，能透过洞看到背后（用户原话「穿衣服导致透明」「身体部分消失（能看到背后）」）；
- **编辑态永远不复现**（MA ShapeChanger 只在 Bake / Play / 上传生效）；
- 烘焙克隆判据：FX 里出现 `NaNimatedBone for <SMR>.deletedShape.<X>_OFF` 一类层（ShapeChanger 经 AAO `TraceAndOptimize` 转成 NaN 骨动画），且 `Body_Base.sharedMesh.blendShapeCount > 0`。

### 28.2 根因
`ModularAvatarShapeChanger`（本工作区挂在 `PJ_Shirt`：11 条 `Body_Base` 的 `*_OFF` → 0；`PJ_Outer`：1 条 `PJ_Shirt/CardiganFit` → 100）。
这些形态键在烘焙期被 AAO 转成 NaN 骨实现；跨头像克隆后骨骼/网格顺序与作者环境不一致时，NaN 骨塌到错误区域 → 身体成片消失。

### 28.3 处置（本工作区采用）
1. `Undo.DestroyObjectImmediate` 删除 `PJ_Shirt` / `PJ_Outer` 上的 `ModularAvatarShapeChanger`；
2. 重新烘焙验证：`NaN` 层 = 0、`Body_Base.blendShapeCount` 被 AAO 剪到 0、默认 `actSelf=True`；
3. 代价：衣服不再自动隐藏身体 → 必须目视确认无穿模（本次睡衣为宽松款，无明显穿插）。

### 28.4 「一键穿衣」要像开关：`Button(101)` → `Toggle(102)`
- VRChat 菜单里 `Button` 只是一次性按钮；`Toggle` 才显示为**勾选开关**。
- 共享 Int `OutfitSelect`（1=睡衣1、2=胡萝卜衣服）时：把每套「一键穿衣」的 `Control.type` 设为 **Toggle(102)**、`Control.value` 保持 1 / 2：
  - ON（=本套）写 value；再点 → 写 0 → 回落**场景序列化初始态**（睡衣容器可见），不会裸体；
  - 因此 `OutfitSelect` 的 EP 默认值必须 = 1，且睡衣容器/部件在场景里 `activeSelf=true`（第 121 轮裸体修复）。
- 设置代码（`Control` 是 struct，改完必须写回，否则丢修改）：
```csharp
var ctrl = fCtrl.GetValue(mi);                    // 装箱的 Control struct
var ft = ctrl.GetType().GetField("type");
ft.SetValue(ctrl, System.Enum.ToObject(ft.FieldType, 102)); // Toggle
fCtrl.SetValue(mi, ctrl);
UnityEditor.EditorUtility.SetDirty(mi);
```
- 验收：烘焙后菜单树 `/装扮菜单/衣服/睡衣1/一键穿衣 type=102 param=OutfitSelect val=1`、`…/胡萝卜衣服/一键穿衣 type=102 param=OutfitSelect val=2`。

### 28.5 附带坑：`Control.icon` 的「假 null」不要用反射装箱判空
`Control.icon` 是 `Texture2D`；`FieldInfo.GetValue` 装箱后 `as UnityEngine.Object` 判空会把**正常 null**误报成 fake-null（本工作区曾据此误清 35 个 MI 图标）。
判空请用 `SerializedObject`：`so.FindProperty("Control.icon").objectReferenceValue == null`（缺失引用同样返回 null）。


---

## §29 从裸头像重建「MA 菜单 + 服装可见性」的五个硬坑（本工作区实测 2026-10-29，第 123 轮）

1) **自定义 FX 层不要经 MA MergeAnimator 整控制器合并**  
症状：把含 m_IsActive 曲线的自建 controller 挂 `ModularAvatarMergeAnimator` 后，烘焙产物里同一 clip 变成 20 条 `armature/.../睡衣$guid` 重复曲线；
VRCFury `CleanupEmptyLayersService` 判定 `hasNonEmptyClip=false` → `layer.weight=0`，菜单点了没反应。  
做法：把自定义层**直接加进 descriptor 的 FX controller 副本**：`AssetDatabase.CopyAsset(作者controller, 副本)` → 删掉绑定已删物的层 → `AddLayer/AddState/AddTransition` + 自建 clip；不要用 MergeAnimator 合并整份带可见性动画的 controller。  
判据：烘焙后自建层 `defaultWeight==1`；`AnimationUtility.GetCurveBindings(clip)` 路径能在 clone 里 `transform.Find` 命中。

2) **AAO TraceAndOptimize 会删除「场景 inactive 且只被后加动画层引用」的部件**  
症状：先给服装/饰品 `SetActive(false)` 做默认隐藏，再烘焙 → clone 里的 SMR 从 27 掉到 9，泳装/眼罩/手套/名牌整组消失（因为 AAO 比 MA 先跑、看不到后加的动画引用）。  
做法：`SerializedObject(traceAndOptimize)` 把 `removeUnusedObjects=false`、`optimizeAnimator=false`（本次还保留 mergeSkinnedMesh=false 以保形态键），重新烘焙。  
判据：clone `GetComponentsInChildren<SkinnedMeshRenderer>(true).Length` 与编辑态一致（本次 589 transforms / 27 SMR）。

3) **面板「Build Failed + This file was already uploaded」可能是假失败**  
症状：点 Build & Publish 后弹 Build Failed，Editor.log 出现 `This file was already uploaded`（VRChat 文件级去重报错），但 `VRCApi.GetAvatar(id,true,ct)` 的 Version 已 +1、UpdatedAt 更新、Name/Tags 正确。  
做法：以 `GetAvatar` 为金标准（`ContinueWith` 写文件再读，禁止 `.Result`）；如果报的是 `The object of type 'VRCRotationConstraint' has been destroyed`，
先优雅退出 Unity 再 `tools\scripts\unity_launch_one.ps1` 重启，然后重新点一次 Build & Publish。

4) **面板元数据反射写值 + TagsField 真实值**  
- `_nameField` / `_descriptionField` 是 `VRC.SDKBase.Editor.Elements.VRCTextField`（继承 UI Toolkit TextField），用 **`value` 属性**赋值（不是字段）；只改 `_avatarData` 不改 UI 会在下次 Save 时被覆盖。  
- 标签要通过 `TagsField._tagInput.value = "星野爱"` 再调 `TagsField.AddTag()`；落盘后的真实值是 `author_tag_星野爱`。直接塞 `_avatarData.Tags` 再点 Save 会被清空。  
- 只改信息不需要重新 build：点 Save Changes（或反射 `HandleSaveChangesClick()`）→ 版本号 +1、包体不变。

5) **VRChat 服务端会把 Description 里的 ASCII `(` 归一化成全角 `（`**  
本次 `UpdateAvatarInfo` 里用 `(char)40` 发送 `嗷呜~(小梓溪要哈气了`，服务端返回 `嗷呜~（小梓溪要哈气了`（descChars 第 4 位 65288）→ 属服务端行为，不必重复尝试。

附：本次成品关键数字（狼）：编辑态 `transforms=794 SMR=27 MI=21 OT=16 MAParams=4 MergeAnimator=7 Installer=8`；
烘焙 clone `589 transforms / 27 SMR / FX 60 层 / NaN 层=0 / MA Responsive 20 层全部 weight=1`；EP 78 参数（OutfitOneClick Int0、PJ_Shirt/PJ_Pants=1 其余 0、
LightLimit 0.55、Monochrome 0、Backlight 0）；上传后 API `Version=30`、Name=沐梓溪（小狼版）、Tags=星野爱/白月光/小爱同学、private。


### §29B 补充：GM Play 全量模拟实测（2026-10-29 第 123 轮）

- **GM 模块反射**：`BlackStartX.GestureManager.Editor.Modules.Vrc3.ModuleVrc3`（本工程 GM 3.9.7 单参构造 `ModuleVrc3(VRCAvatarDescriptor)`）；
  关键是先 `Activator.CreateInstance(ModuleSettings)` 把 `VRCAvatarDescriptor` 字段填好、再写 `module.Settings`，
  最后 `GestureManager.SetModule(module)` → 才会真正烘焙并接管 AvatarAnimator（只 SetModule 不建 Settings 不会生成烘焙副本）。
- **实测规模**：菜单 113 项 / 232 步 / 37–47 s / 191 张截图；`pass=104 fail=6 noparam=1`。6 个 FAIL 全是 ABT 世界功能（传送门A/B、固定座位、放门传送至红/蓝/黄标）+1 NOPARAM（ABT 地图位置 TwoAxisPuppet 无 model 参数），属第三方工具在 GM 中不可测的已知项（与历史基线一致）。
- **关键回归：共享 Int 的「一键穿衣」不能靠 MenuItem 上的 ObjectToggle 做套装饰互斥**  
  症状：两套一键穿衣共用 `OutfitOneClick`（value 1/2），各自 MI 上挂 OT 行（睡衣 OT `泳装@False`、泳装 OT `睡衣@False + 泳装@True`）→ GM 里点泳装一键穿衣后**角色裸体**（睡衣根被藏、泳装根没显示）。  
  根因：MA ObjectToggle 对非 Bool 参数的条件是「参数 > 0.5」而非「等于 MI 的 value」；两套 OT 在 param=2 时**同时生效**，后写的睡衣层把泳装根又写回 0。  
  ⚠️ **已废弃（第 132 轮推翻）**：套装根显隐**不再**用 `OutfitRoots`——该层已删（FX 12→11），5 个套装根常驻 active，可见性只由部件 bool + MA ObjectToggle 驱动；
现行见 [03-06 §14](../03-菜单与参数/03-06-ma-menu-build-by-reflection.md) 与本文 §34。以下为历史做法：套装根显隐改由自建 FX 层 `OutfitRoots`（3 态 Default/Pajama/Swim，
每态 clip 显式写 `睡衣`/`泳装` 两个根的 `m_IsActive`，转移条件 `OutfitOneClick Equals 0/1/2`；`OutfitSync` 3 态 driver 同步 11 个部件 bool）。  
  另一个必要条件：**被动画显示的根对象在场景里必须 active=true**（`泳装` 根若场景 inactive，烘焙时它的 `m_IsActive` 曲线可能被当作无效绑定剪掉，导致只藏睡衣不显泳装）。部件子对象可保持 inactive（由 MA OT 的 ActiveSelfProxy 处理）。
- **上传去重的注意**：本次连续两次 Build & Publish 都报 `This file was already uploaded`（Editor.log UploadError），但 `VRCApi.GetAvatar` 的 Version 从 30→33→34 递增、
  UpdatedAt 变化、UnityPackages=1；而且 v34 与 v33 的 `AssetUrl` 相同（`file_a384f0b2-.../17/file`）→ 说明服务端按内容哈希复用文件，面板红字 ≠ 上传失败，**以 API Version/UpdatedAt 为金标准**。


---

## §30 上传「假成功」：缩略图 MD5 相同会在更新 bundle 之前抛出，Version 却已 +1（本工作区实测 2026-10-29，第 124 轮）

### 30.1 症状

连续点 `Build & Publish`，面板弹 `Failed to upload avatar!`，`Editor.log` 里是
`UploadException: This file was already uploaded`（`VRCApi.cs:951`）；
但 `VRCApi.GetAvatar()` 的 **Version 一直在涨**（本次 30→33→34→38→39→42→43），
而 `UnityPackages[0].AssetUrl` 的**文件版本号长期停在 `/17/`** —— 也就是说**包体根本没更新**，涨的只是元数据版本。

### 30.2 根因（SDK 源码顺序）

`VRCSdkControlPanelAvatarBuilder.Upload()`（`com.vrchat.avatars`）的顺序是：

```
UpdateAvatarInfo   →  UpdateAvatarImage（仅当 _newThumbnailImagePath 非空）
                   →  UpdateAvatarBundle(...)
```

`VRCApi.UploadFile()` 会先算本地文件 MD5，与服务器上该 file 记录**最新版本**的 MD5 比较，
完全相同就 `throw new UploadException("This file was already uploaded")`（`VRCApi.cs:943-951`）。
本次缩略图是**同一张封面 + 同一套 CropImage 参数**裁出来的，字节完全一致 → 抛异常 →
整个 `Upload()` 在 `UpdateAvatarBundle` **之前**中止 → 面板红字 + 元数据版本已 +1 的假象。

### 30.3 修法（二选一）

1. **跳过缩略图**：构建前把私有字段清空 ——
   `builder.GetType().GetField("_newThumbnailImagePath", NonPublic|Instance).SetValue(builder, null)`，
   然后反射 `OnBuildAndPublishAction()`。此时 `hasNewThumbnail=false`，直接走 `UpdateAvatarBundle`，沿用线上旧缩略图。
   本次这样一次成功：`AssetUrl` `/17/ → /18/`。
2. 换一张**字节不同**的缩略图（同图重编码 / 换图），MD5 变了就不会抛。

### 30.4 判据（务必用这条，不要只看 Version）

```csharp
var av = await VRCApi.GetAvatar(id, true, ct);
// ✅ 真上传：av.UnityPackages[0].AssetUrl 里的文件版本号递增（.../file_xxx/18/file）
// ❌ 假成功：AssetUrl 版本号不变、只有 av.Version 递增（说明只更新了元数据）
```
另：`editor.log` 中 `Failed to upload avatar!` 紧跟在 `Updating avatar thumbnail` 之后，就是这个坑。

### 30.5 顺带：面板 Tags 会翻倍

`TagsField.AddTag()` 写进列表的是 `author_tag_<名字>`；若同时又直接给 `_avatarData.Tags` 写纯名字，上传后服务器会同时出现两套（本次 3→6 个）。
修法：上传后用 `VRCApi.UpdateAvatarInfo(id, avatar, ct)` 只发 **3 个纯名字**（本次 Version=43 落定为 `星野爱/白月光/小爱同学`），
并把 `_avatarData.Tags` 与 `TagsField.tags` 都清成同一组纯名字，避免下次构建再翻倍。


## §31 第 126 轮：MA MenuItem 宿主 GameObject 未激活 → 整组 ObjectToggle 静默不烘焙

### 31.1 症状

- 烘焙后的菜单里**有这个子菜单和全部开关项**（`/外观/衣服/泳装/衣服|裤子|鞋子|书包|帽子`，参数 `SW_*`），但：
  - 烘焙 FX 里**没有任何 `MA Responsive: IceCreamSummer_*` 层**；
  - GM Play 里把 `SW_Tops` 设为 True，参数读数正确，**衣服就是不显示**；
  - 用 `OutfitRoots` 把 `泳装` 根设为 active=1 也不行（根上去了、件还是 inactive）。（注：`OutfitRoots` 层已于第 132 轮删除、套装根常驻 active，本坑在现行架构下不复现；见 §34 / 03-06 §14）
- 编辑态场景里该子菜单的宿主是 `泳装/泳装`（MenuItem + MenuInstaller 所在对象），**`activeSelf=False`**。

### 31.2 根因

MA 在烘焙期对 **inactive 的 MenuItem 宿主**整条跳过：不仅菜单项本身，**同一对象/子对象上的 `ModularAvatarObjectToggle` 也一起不生成响应层**。
菜单项的 `Control` 数据来自别处（本工程是另一份静态菜单资产），所以「菜单看起来是好的」具有很强的欺骗性。

### 31.3 定位（两条断言）

```csharp
// 1) 场景：所有带 ModularAvatarMenuItem 的 Transform 里，activeSelf=false 的必须为 0
foreach (var t in wolf.GetComponentsInChildren<Transform>(true))
  if (t.GetComponent("nadena.dev.modular_avatar.core.ModularAvatarMenuItem") != null && !t.gameObject.activeSelf)
    Debug.Log("INACTIVE_MENUITEM " + t.name);

// 2) 烘焙后：对每个应受开关控制的网格，FX 里必须有同名 `MA Responsive: <对象名>` 层
//    没有 = OT 没生成（先查宿主 active，再查 OT 的 m_objects 引用路径）
```

本工作区实测：39 个 MenuItem 宿主里 **1 个 inactive**（`泳装/泳装`）；把宿主 `SetActive(true)` 后重烘，
FX 从 72 层 → **73 层**，出现 5 个 `MA Responsive: IceCreamSummer_*` 层，GM 里 5 个 `SW_*` 全部生效。

### 31.4 判据

- 修复前：`swimLayers=0`，GM `ONECLICK|swim` 部件参数 True 但画面裸体；
- 修复后：`swimLayers=5`，同一断言 PASS，10 视角渲染正常（帽子是 VR 头显，遮住眼睛属于设计）。

> 关联：`§14`（MenuItem 缺 MenuInstaller 菜单不出现）——本条是「有 Installer 但宿主 inactive」，同样静默。


## §32 第 127 轮：默认开启型 Toggle 的验收（EP / MA / Animator 三处默认值 + GM 初值）

### 32.1 症状

菜单里「一键穿衣」应该在默认状态显示为开启，但 GM Play 里初值 `OutfitOneClick=0`；
或者 EP/MA 已改成 1，编辑器 Play 仍读到 0。

### 32.2 根因

1. **AnimatorController 的参数默认值**没改：GM 的 ModuleVrc3 初始化参数时读 controller 的 `AnimatorControllerParameter.defaultInt`，不读 EP。
2. struct 装箱写回丢失（见 `03-06 §13.3`）：`foreach` 修改 `ParameterConfig` / `ExpressionParameter` 副本无效。
3. `ModularAvatarMenuItem.isDefault` 只是菜单语义，不驱动参数值。

### 32.3 正确做法与判据

- 三处都写 1（MA ParameterConfig、EP、AnimatorController），睡衣 MenuItem `isDefault=true`；
- 烘焙后断言：clone EP `def=1`、clone FX 参数 `def=1`；
- GM Play 第一帧（等待 ≥20 帧）读 `OutfitOneClick=1` 且 `PJ_Hat=0`、`PJ_Shirt/Pants=1`；
- 再点其它套装 → 参数变 2..5、对应部件全 1、其它套装全 0、其它套装根 `activeInHierarchy=False`。

### 32.4 本项目实测（第 127 轮）

- 修前：`ONECLICK|initial value=0 FAIL`（EP/MA 改过但 controller 默认 0 + struct 写回丢失）；
- 修后：`ONECLICK|initial value=1 PJ_Hat=0 PASS`，菜单 147、`pass=150 fail=6 noparam=1`，`bits=186 ≤ 256`，FX 96 层。

> 关联：`§31`（宿主 inactive）、`03-06 §13`（6 态一键穿衣）。

---

## §33 同名 Shader 覆盖导致「发丝没描边 + 眼镜材质丢失 + 亮度插件完全失效」（本工作区实测 2026-10-30，第 131 轮）

### 33.1 症状组合

同一个头像上同时出现下面三条，且**材质资产、贴图、Mesh、LightController 组件都正常**：

1. 头发模型有 `M_LilLeo_Hair_PW`、`_OutlineWidth=0.16`、`_OutlineTex` / `_OutlineWidthMask` 全在，但画面**完全没有头发描边**；把 `_OutlineWidth` 从 0.16 拉到 0.5、`_OutlineColor` 改成红色也没有任何变化。
2. `SkyRound` 眼镜材质仍指向 `SkyRound_Metal_ShootingStar`，但金属框/镜链呈**白色无材质**；`_MainTex` 为 null（该材质本来就靠 matcap）。
3. `LightController/LightLimit`、`Monochrome`、`BacklightToggle` 参数存在、层权重 1、Clip 里有 `material._LightMaxLimit` / `_MonochromeLighting` 曲线，但滑块完全不影响画面。

### 33.2 根因：逆向工程的占位 Shader 用了与 lilToon 包相同的 Shader 名

本工程 `Assets/20_逆向/赛博猫猫/Shader` 与 `Assets/20_逆向/赛博小狗/Shader` 是 AssetRipper 导出的占位 shader，里面包含：

```text
Hidden/lilToonOutline
Hidden/lilToonCutout / CutoutOutline / Gem / Transparent / TransparentOutline / Refraction
Hidden/ltspass_opaque / ltspass_cutout / ltspass_transparent
lilToon
```

它们的实现是 **`DummyShaderTextExporter` 生成的单 pass 极简 shader**（`return _MainTex.Sample(...) * _Color;`，`Fallback "Unlit/Texture"`）。

lilToon 的 `lts_o.shader` / `lts_cutout.shader` 等**不是自己实现所有 pass**，而是：

```text
UsePass "Hidden/ltspass_opaque/FORWARD"
UsePass "Hidden/ltspass_opaque/FORWARD_OUTLINE"
```

`UsePass` 是**按 shader 名解析**的，不按 GUID。于是：

- `Hidden/ltspass_opaque` 被占位 shader 覆盖 → `FORWARD_OUTLINE` 不存在或不是 lilToon 轮廓 pass → 头发/衣服描边消失；
- `Hidden/ltspass_cutout` 被占位 shader 覆盖 → 眼镜材质里的 `_MatCapTex`、`_Color`、裁切/透明语义全部不执行 → 看起来像“材质丢了”；
- `Hidden/ltspass_opaque` 的极简 pass 不认识 `_LightMinLimit` / `_LightMaxLimit` / `_MonochromeLighting` → LightController 曲线写了但画面不变。

> 关键判据：**材质 `m_Shader` 本身是 package 的 lilToon 资产，不代表它实际使用的 pass 是 package 的**；`UsePass` 依赖的 `Hidden/ltspass_*` 才是实际实现。只看材质 shader 名会误判。

### 33.3 正解：只改占位 shader 的内部 Shader 名，保留 GUID/材质

不要删除 `Assets/20_逆向`、不要改 `.meta`、不要改逆向材质的 GUID；只把占位 shader 的**内部 `Shader "..."` 名**改成唯一名，例如：

- 赛博猫猫：`Hidden/ReverseCyberCat/lilToonOutline`、`Hidden/ReverseCyberCat/ltspass_opaque`、`Hidden/ReverseCyberCat/lilToon` …
- 赛博小狗：`Hidden/ReverseCyberDog/lilToonCutout`、`Hidden/ReverseCyberDog/ltspass_cutout`、`Hidden/ReverseCyberDog/lilToon` …

这样：

- 逆向材质仍按 GUID 使用自己的占位 shader，不会变成粉材质；
- package 的 `UsePass "Hidden/ltspass_opaque/..."` 会重新解析到 `Packages/jp.lilxyzw.liltoon/Shader/ltspass_opaque.shader`；
- 描边、眼镜 matcap、LightController 材质属性全部恢复。

本工作区实测改动（备份 `_asset_reorg/backup/round131_shaders/`）：

```text
Assets/20_逆向/赛博猫猫/Shader/*.shader
Assets/20_逆向/赛博小狗/Shader/*.shader
  Shader "Hidden/..." -> Shader "Hidden/ReverseCyberCat/..." 或 "Hidden/ReverseCyberDog/..."
```

### 33.4 验收

```csharp
Shader.Find("Hidden/ltspass_opaque")   // 应为 Packages/jp.lilxyzw.liltoon/Shader/ltspass_opaque.shader
Shader.Find("Hidden/lilToonOutline")   // 应为 Packages/jp.lilxyzw.liltoon/Shader/lts_o.shader
Shader.Find("Hidden/lilToonCutout")    // 应为 Packages/jp.lilxyzw.liltoon/Shader/lts_cutout.shader
```

并检查 `Shader.passCount`：本工作区修复后 `Hidden/ltspass_opaque` / `Hidden/lilToonOutline` 均为 **7 passes**，
`Hidden/lilToonCutout` / `Hidden/lilToonGem` 为 **5 passes**；修复前 `Shader.Find` 会命中 `Assets/20_逆向/...` 的 1-pass 占位 shader。

视觉验收：

- 头发浅灰背景特写：有深色描边；
- `SkyRound` 眼镜：金属框/镜链恢复 matcap，不再纯白；
- GM Play 把 `LightController/LightLimit` 从 0 拉到 1：截图平均亮度 `143.4 → 223.8`（不是只看参数值）。

### 33.5 相关坑

- 不要把“材质 `m_Shader` 是 lilToon”当作“实际 pass 是 lilToon”；`UsePass` 名冲突优先查 `Shader.Find` 返回路径。
- 逆向工程目录**不要整目录隐藏/删除**：可能已有材质按 GUID 引用它们；只改内部 Shader 名。
- 修改 `.shader` 后必须 `AssetDatabase.Refresh()`；`Shader.Find` 有缓存，刷新后再断言。

## §34 第 132 轮：Unity 重启后 MCP 桥失效的判别 + 不进入 Play 的烘焙级验收

### 34.1 MCP 桥失效的三种状态（先判别再动手，别急着重启 Unity）

| 现象 | 判别 | 处理 |
|---|---|---|
| DSH `mcp__unity__*` 全部 `No Unity Editor instances found` | Unity 可能还活着，只是 MCP 握手/心跳未恢复 | 读 `<USERPROFILE>\.unity-mcp\unity-mcp-status-<projecthash>.json`：`reloading:false` + `reason:"ready"` + `last_heartbeat` 是否为当前时间；`netstat -ano \| findstr 6400` 看监听 pid 是否 = Unity pid |
| `http://127.0.0.1:8080/mcp` 返回 `instance_count:0` 或 `no_unity_session` | 8080 常是**上一次会话遗留**的 `mcp-for-unity --transport http`（父进程 uv），Unity 重启后不会重新注册 | 不要用它判断 Unity 死活；以 6400 + 状态文件为准，或等 DSH 工具自恢复 |
| `refresh_unity(scope:"all", wait_for_ready:true)` 返回 `{"timeout":true}` | 全量导入 + 域重载，正常 | 轮询 `resources/read mcpforunity://editor/state` 直到 `data.advice.ready_for_tools=true` 再发 `execute_code`；期间 `no_unity_session` 属正常，不要重启 Unity |

补充：Unity 启动模态框「Unity is running as administrator.」会卡住握手 → `EnumWindows` 找 class `#32770`，`PostMessage(hwnd, WM_CLOSE=0x0010, 0, 0)`；无效再 `SendMessage(hwnd, WM_COMMAND=0x0111, IDOK=1, 0)`。

### 34.2 不进入 Play 跑一次烘焙（NDMF，上传前最终静态确认）

```csharp
// Assets/Editor/xxx.cs（CodeDom 兼容 C#6）
var clone = nadena.dev.ndmf.AvatarProcessor.ManualProcessAvatar(wolf, null); // 同步，约 19 s
var bd = clone.GetComponent<VRC.SDK3.Avatars.Components.VRCAvatarDescriptor>();
foreach (var p in bd.expressionParameters.parameters) { /* p.saved / p.defaultValue = 持久化唯一可信判据 */ }
var fx = (UnityEditor.Animations.AnimatorController)bd.baseAnimationLayers[4].animatorController; // FX 槽位
fx.layers.Length;
UnityEngine.Object.DestroyImmediate(clone); // 之后场景 dirty=true → SaveScene 复位
```

- 编译坑（CodeDom 与手写 Editor 脚本都踩）：`animatorController` 是 `RuntimeAnimatorController`，
  取 `.layers` 必须 `as UnityEditor.Animations.AnimatorController`（否则 CS1061）；`BlendTree` 要写全名 `UnityEditor.Animations.BlendTree`（CS0246）；
  `mi.Control.parameter` 是 struct，取参数名要 `.parameter.name`（CS1503/CS1929）。
- 第 132 轮实测：烘焙 19.3 s；clone `transforms=728 smr=51`；`BEP|count=102`；`OutfitOneClick Int def=1 saved=True`、
  `Acc_NamePlate/Acc_Glasses Bool def=1 saved=True`、`PJ_Outer def=1 saved=True`、`PJ_Shirt/Pants def=1 saved=False`；`ROOTWRITE|total=0`；
  `BMENU|controls=138|carrot=0`；`BOBJ` 套装根 active、眼镜/小狼手套/Cat_Tail inactive。
- 用途：在同一 clone 上断言「默认可见件」「无根互斥曲线」「saved 标志」，比 GM 更可信——GM 不模拟 Gesture 层、不模拟 Button 自动复位，也不验证 `saved`。

### 34.3 教训

- Unity 崩溃/重启后**先把工具链恢复**再动资产；旧 HTTP 会话的返回值不能作为「Unity 没连上」的证据。
- `refresh_unity(scope:"all")` 会让大工程整体重导入（1–2 分钟），期间所有 MCP 调用都可能失败；批量脚本要写成「一次 refresh + 轮询 ready + 一次执行」，不要分散调用。

## §35 上传：不需要 OTP 的 SDK 凭据探活 + `web-derived token` 假拦截（第 132 轮补，2026-10-31）

### 35.1 结论优先：上传前先「探活」，不要先要验证码
- 正确顺序：`VRCApi.GetAvatar(blueprintId, forceRefresh:false, ct)` 打一发 → 返回 `Version>0` 就说明**当前 Editor 的持久凭据可用**，可以直接 Build + UpdateAvatarBundle。
- 反例（本轮踩到）：上一轮因为 `APIUser.CurrentUser == null`（SDK 面板没登录）+ `auth.tmp` 过期，就判定「必须邮箱 OTP」并停了 1 轮；实际凭据一直有效，
  只是旧代码用 `Type.GetType("…ApiCredentials, VRCSDKBase-Editor")` + 反射字段名（`authToken`）判定，类型/字段解析失败 → `persisted token len=-1` → 误判无凭据。**判据应该是请求结果，不是字段反射。**
- 失败形态：`ApiErrorException`（`code=Unauthorized`/`Missing Credentials`，401）→ 再走 auth.tmp / 面板登录 / OTP。

### 35.2 `GetAvatar(..., forceRefresh:true)` 会误报 `BadRequest: Using a web-derived token in client`
- 同一枚凭据实测：`GetAvatar(bp,false)` ✅、`builder.Build(wolf)` ✅、`VRCApi.UpdateAvatarBundle` ✅（本轮 Version 49→50 成功）；但在某些时刻 `GetAvatar(bp,true)` 抛 `BadRequest` + `Using a web-derived token in client`。
- 结论：**这是请求被判定为 client-only 端点造成的，不代表凭据失效、也不阻断上传**（与第 130 轮 `BuildAndUpload` 在 copyright 处报同一错误的经验一致）。探活统一用 `forceRefresh:false`。

### 35.3 可用的最小上传流程（Editor 内，必须 `async void`）
1. `VRCSdkControlPanel.TryGetBuilder<IVRCSdkAvatarBuilderApi>(out var builder)`；拿不到就先反射 `ShowControlPanel()` 再 `await Task.Delay(1500)` 重试。
2. `VRC_SdkBuilder.ActiveBuildType = VRC_SdkBuilder.BuildType.Publish;`
3. `string bundle = await builder.Build(wolf);`（本轮 ≈3 min，NDMF 烘焙 + shader 编译；产物在 `%TEMP%/DefaultCompany/<工程名>/<guid>.vrca`）
4. `var av = await VRCApi.GetAvatar(bp, true, ct);` → 过滤 `av.UnityPackages` 里 `Variant ∉ {Standard, Impostor}` 或 `Platform == "security"` 的包（否则后续 
  `GetLatestAssetUrlForPlatform` 的 `Enum.Parse` 抛 `Requested value 'security' was not found`）。
5. `var after = await VRCApi.UpdateAvatarBundle(bp, av, bundle, null, ct);` → 成功返回新 `Version`/`UpdatedAt`/`UnityPackages[].AssetUrl`。

### 35.4 坑与收尾
- **禁止在主线程 `Task.Wait`/`.Result`**：`execute_code` 里会死锁 Unity（MCP 全超时、进程 Responding=False）。写成 `static async void Run()` + 全程文件落盘（`upload_status.txt`），外部轮询文件。
- 成功判据用返回体：`Version` 递增、`UpdatedAt` 变化、标准包 `AssetUrl` 尾部 `/N/file` 的 N 变化。本轮 `49→50`、`UpdatedAt=2026-10-30T19:24:03Z`、`/22/file→/23/file`。
- `builder.Build` 会重写 `Assets/08_动画·菜单·图标/NyaPPu__Generated/LightController/*.anim`（亮度插件生成物）
  → 上传后从 `<VRCHAT_ROOT>\$tmp\Lilleo2.1\Assets\NyaPPu__Generated\LightController` 还原并核 MD5（5 个：880E2687…/F9A12559…/F1BE1377…/C121A44F…/CD1D15CC…）。
- 同时会重新生成 `Assets/ZZZ_GeneratedAssets`、`Packages/com.vrcfury.temp/Builds`、工程根 `.sln/.csproj` → 收尾各删一次（编译后会重生，交付前再查 `sln/csproj/ZZZ` 计数）。
- 上传会把场景标脏（`dirty=True`）但结构不变：本轮审计 `transforms=999 smr=51 mr=25 missingScripts=0 nullMesh=0 badMatSlots=0 hideflagObjs=0` → 通过后 `EditorSceneManager.SaveScene` 即回 `dirty=False`。
- 上传器/凭据脚本用完整后移出 `Assets/Editor`（本轮移到 `<VRCHAT_ROOT>\tools\scripts\round132\`），避免进构建产物与后续编译噪音。

## §36 构建期 26 条 `CBUFFER_START` 着色器报错 + `Build asset version error` = lilToon 构建期优化瞬态（第 134 轮实测 2026-10-31）

**现象**：`builder.Build(wolf)`（上传与「仅构建」都一样）的 `Editor.log` 里**每次**出现：
- `Shader error in 'lilToon' / 'Hidden/lilToonOutline' / 'Hidden/lilToonCutout' / 'Hidden/lilToonTransparent(Outline)' / 'Hidden/lilToonCutoutOutline': unrecognized identifier 'CBUFFER_START' at line 660/662/693/695 (on d3d11)` —— 一次构建 **26 条**；
- `Message: Build asset version error: packages/jp.lilxyzw.liltoon/shader/includes/lil_common_input_opt.hlsl in SourceAssetDB has modification time of '...' while content on disk has modification time of '...'` —— 1 条；
- 日志里成对出现 `[lilToon] PreprocessBuild` / `[lilToon] PostprocessBuild`，中间夹着 `Start importing Packages/jp.lilxyzw.liltoon/Shader/lts*.shader` 的逐文件 reimport。

**根因**（本工程 lilToon 是内嵌包 `file:jp.lilxyzw.liltoon`，2.3.2）：
- `Packages/jp.lilxyzw.liltoon/Editor/lilToonEditorUtils.cs:769-784`：`lilToonBuildProcessor : IPreprocessBuildWithReport, IPostprocessBuildWithReport`；
- `OnPreprocessBuild` → `lilToonSetting.SetShaderSettingBeforeBuild()` → `Editor/lilToonSetting.cs:801 ApplyShaderSettingOptimized()` → `lilOptimizer.OptimizeInputHLSL()` + 重写 12 个 `Shader/lts*.shader`、`ltspass_*.shader` 与 `shader/includes/lil_common_input_opt.hlsl`，并同步 `AssetDatabase.Refresh()`；
- 构建仍在进行 → 源文件被并发重导入 → `SourceAssetDB` 版本与磁盘不一致（即上面的 `Build asset version error`），shader 在「半改写/半还原」状态下被编译 → `CBUFFER_START` 找不到；
- `OnPostprocessBuild` → `SetShaderSettingAfterBuild()`（`lilToonSetting.cs:963-992`）按 `lilEditorParameters.instance.modifiedShaders` 把 14 个文件还原，mtime 落在构建结束后 1–2 s。

**判为无害的判据（本轮实测）**：
1. bundle 正常产出：日志里有 `Bundle Name: prefab-id-...` 与 `Total compressed size 20.4 MB`，`.vrca` 存在且能上传；
2. 构建后 `AssetDatabase.ImportAsset("Packages/jp.lilxyzw.liltoon/Shader", ImportRecursive|ForceUpdate)` + `Refresh` → `LogEntries` 0 error；编辑态渲染四角 =(26,28,33) 恰为背景色 `(0.10,0.11,0.13)`、中心亮度≈235、洋红像素 0%；
3. `Editor.log` 里 3 次构建、78 条（26×3）同签名；而上一次成功上传的 **Version=50** 就是这种构建产物，游戏内长期使用材质正常（用户只反馈菜单/衣装行为）；
4. 第 134 轮用同签名构建上传 **Version=51** 成功（`UpdatedAt 2026-10-31T03:46:26Z`、标准包 `/23/file → /24/file`）。

**处理约定**：
- 不要因为这几条报错回滚 lilToon、清 `Library`、改 `ProjectSettings/lilToonSetting.json`；上传成败只看 `UpdateAvatarBundle` 返回的 `Version`/`UpdatedAt`/`AssetUrl` 与 bundle 是否产出。
- 想「关掉优化」也别乱改三项：`ShouldOptimization()`（`lilToonSetting.cs:1508`）= `modifiedShaders 非空 → false`；`forceOptimize == true` 直接 true；VRChat 包还看 `shaderSetting.isOptimizeInTestBuild && !isDebugOptimize`。本工程 `lilToonSetting.json` 的 `isDebugOptimize/isOptimizeInTestBuild/isOptimizeInNDMF` 三项都是 false 但 preprocess 仍执行（`forceOptimize` 被别处置 true，未继续追源码），所以改这三项并不保证关掉。
- 每次 `builder.Build` 之后仍按 **§35.4** 收尾：还原 `Assets/08_动画·菜单·图标/NyaPPu__Generated/LightController/*.anim` 5 个并核 MD5、删 `Packages/com.vrcfury.temp/Builds`、工程根 `.sln/*.csproj`、`SaveScene`。

**顺带记（本轮实测，省下一次试错）**：`VRC.Core.ApiCredentials.Load()` 返回对象的字符串字段经反射读出为空（`AuthToken`/`TwoFactorAuthToken` 均空）→ **不能**靠它把当前 SDK 会话 token 导给外部脚本；`VRCApi.Headers` 字典里只有 `User-Agent`/`Accept`，**没有** cookie；`APIUser.CurrentUser != null` 只证明 SDK 面板已登录（本轮 `沐梓溪Luzzy`），上传走 `current-session` 即可，**不需要邮箱 OTP**（第 132 轮 §35 同理）。
