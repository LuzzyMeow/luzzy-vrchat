# Modular Avatar

所有 MA 组件的创建必须在**编辑态**完成（不是 Play 模式），否则不会随上传保留。

MA 只在 Build 或 Manual Bake 时合并/应用 —— 没 bake 的编辑器视图显示未合并状态，**这是正常的，不是 bug**。

---

## 组件速查

| 组件 | 作用 |
|---|---|
| `ModularAvatarMenuItem` | 菜单项和子菜单 |
| `ModularAvatarMenuInstaller` | 把菜单 asset 或 children 装进 avatar expression menu |
| `ModularAvatarParameters` | 安装参数 |
| `ModularAvatarMergeAnimator` | 合并 animator controller |
| `ModularAvatarObjectToggle` | 开关 GameObject / renderer |
| `ModularAvatarMaterialSetter` | 设置材质 slot |
| `ModularAvatarShapeChanger` | 设置 blendshape |
| `ModularAvatarBlendshapeSync` | 衣服和身体 blendshape 同步 |

---

## MergeAnimator

```csharp
var holder = new GameObject("MyFX");
holder.transform.SetParent(avatar.transform, false);

var mam = holder.AddComponent<ModularAvatarMergeAnimator>();
mam.animator   = myController;
mam.layerType  = VRCAvatarDescriptor.AnimLayerType.FX;
mam.pathMode   = MergeAnimatorPathMode.Absolute;   // clip 里写完整层级路径
mam.matchAvatarWriteDefaults = true;
mam.deleteAttachedAnimator   = false;
```

`pathMode = Absolute` 表示 clip 里的路径是从 avatar 根算起的完整路径；`Relative` 则相对于挂载 MergeAnimator 的物体。脚本生成 clip 时用 `Absolute` 更好推理。

**动画没生效时**先查：`animator` 是不是空、layer weight、参数名和 FX 里是否一致。

---

## Parameters

```csharp
var pars = holder.AddComponent<ModularAvatarParameters>();
pars.parameters = new List<ParameterConfig> {
    new ParameterConfig {
        nameOrPrefix = "MyFloat",
        syncType     = ParameterSyncType.Float,
        saved        = true,
        defaultValue = 0f,
        localOnly    = false,
    },
    new ParameterConfig {
        nameOrPrefix = "MyBool",
        syncType     = ParameterSyncType.Bool,
        saved        = true,
        defaultValue = 1f,
        localOnly    = false,
    },
};
```

**参数缺失时**查 `ModularAvatarParameters` 是不是挂在了正确的 avatar 层级下。

---

## MenuItem：Toggle / RadialPuppet / SubMenu

```csharp
// Toggle
var mi = go.AddComponent<ModularAvatarMenuItem>();
mi.Control = new VRCExpressionsMenu.Control {
    name      = "My Toggle",
    type      = VRCExpressionsMenu.Control.ControlType.Toggle,
    parameter = new VRCExpressionsMenu.Control.Parameter { name = "MyBool" },
    value     = 1f,
};
mi.isSynced = true;
mi.isSaved  = true;

// Radial Puppet（float 0–1）
mi.Control = new VRCExpressionsMenu.Control {
    name          = "My Slider",
    type          = VRCExpressionsMenu.Control.ControlType.RadialPuppet,
    subParameters = new[] { new VRCExpressionsMenu.Control.Parameter { name = "MyFloat" } },
};

// SubMenu（当作父容器）
mi.Control = new VRCExpressionsMenu.Control {
    name = "My Submenu",
    type = VRCExpressionsMenu.Control.ControlType.SubMenu,
};
// 挂在这个 GameObject 下的子 MenuItem 会自动变成子菜单条目
```

要把一个独立菜单装进现有菜单，加 `ModularAvatarMenuInstaller`：`installTargetMenu = null` 装到根，或指定某个 `VRCExpressionsMenu`。

### 菜单不出现 / 位置不对

1. **孤儿项**：MenuItem 必须挂在一个能从 avatar 根到达的 SubMenu 之下，否则不显示。
2. **重复项**：脚本重复运行会重复添加。每次重建前先按名字清理：
   ```csharp
   foreach (var it in avatar.GetComponentsInChildren<ModularAvatarMenuItem>(true))
       if (it.Control?.name == "My Item") Object.DestroyImmediate(it.gameObject);
   ```
3. **空文件夹**：`ControlType.SubMenu` 但没有子 GameObject → 显示成空文件夹。
4. **Installer 引用为空**：查 `ModularAvatarMenuInstaller.menuToAppend` 或 children source。

---

## MaterialSetter

```csharp
foreach (var ex in t.GetComponents<ModularAvatarMaterialSetter>())
    Object.DestroyImmediate(ex);   // 先清理，避免叠加

var ms = t.gameObject.AddComponent<ModularAvatarMaterialSetter>();
ms.Objects = new List<MaterialSwitchObject> {
    new MaterialSwitchObject { renderer = smr, materialIndex = slotIndex, material = myMat }
};
```

---

## ShapeChanger

```csharp
var sc = go.AddComponent<ModularAvatarShapeChanger>();
sc.Shapes = new List<ChangedShape> {
    new ChangedShape {
        Object     = new AvatarObjectReference { referencePath = "path/to/SMR" },
        ShapeName  = "SomeShape",
        ChangeType = ShapeChangeType.Set,
        Value      = 0f,
    }
};
```

---

## ObjectToggle

```csharp
var ot = go.AddComponent<ModularAvatarObjectToggle>();
ot.Objects = new List<ToggledObject> {
    new ToggledObject {
        Object = new AvatarObjectReference { referencePath = "path/to/obj" },
        Active = false
    }
};
```

---

## BlendshapeSync

把身体 blendshape 映射到衣服 blendshape，身体形态变化时衣服自动跟随。

排查衣服穿模时**先看作者原有的 `BlendshapeSync` 是不是被删了或被 bake 掉了**，不要先动身体。详见 [00-05-mesh-blendshape.md](00-05-mesh-blendshape.md)。

---

## 服装电台：int 参数 gating

一次只穿一套衣服时的标准模式：一个 `int` 参数（例如 `Costume`），每套衣服一个取值（0=裸, 1=A套, 2=B套 …）。

每套衣服的 FX layer 有一个 **Idle 状态**（默认，什么都不写）和一个 Active 状态：

```csharp
var sIdle = sm.AddState("Idle");
sIdle.writeDefaultValues = true;
sm.defaultState = sIdle;

var sActive = sm.AddState("Active");
sActive.motion = myBlendTree;
sActive.writeDefaultValues = true;

var tOn = sIdle.AddTransition(sActive);
tOn.hasExitTime = false; tOn.duration = 0f;
tOn.AddCondition(AnimatorConditionMode.Equals, 1, "Costume");

var tOff = sActive.AddTransition(sIdle);
tOff.hasExitTime = false; tOff.duration = 0f;
tOff.AddCondition(AnimatorConditionMode.NotEqual, 1, "Costume");
```

**Idle 状态为什么关键**：没有 Idle 的话，layer 的默认状态在这套衣服没穿的时候也会写 blendshape 值 —— 造成全局串扰（身体莫名被压扁、某些形态键被强制归零等）。这是"换了别的衣服，身体却还是上一套的形状"这类 bug 的根因。

### 更简单的替代：常驻 ShapeChanger

如果 avatar 一次只穿一套、而且不需要在运行时切换身体形态，最省事的做法是：

- 在一个**常穿**的部件（底层、或一个不可见的根物体）上挂 `ShapeChanger`；
- 把这套衣服需要的身体 blendshape 直接设死；
- 不需要任何动画 gating。

---

## 参数和菜单预算

VRChat expression parameters 有 synced bit 上限。处理顺序：

1. 列出所有参数：类型、`saved`、`networkSynced`、默认值。
2. **远端必须可见**的保留 synced —— 别人需要看到的衣服状态、互动状态。
3. **本地 UI、临时计算、reset、灯光 handle、debug** 参数设成 local-only 或 `networkSynced = false`。
4. 删除菜单没引用、controller 也不用的参数。
5. 仍然超限再上 MemoryOptimizer / Avatar Optimizer 压缩。

**不要盲删看起来 "unused" 的插件参数。** 互动插件可能通过 animator、parameter driver 或 contact 间接使用它们。

菜单本身：每页控件数量控制在 8 个以内，超了就分子菜单。

---

## 补充：官方字段级语义的分册路由（2026-11-11 整理，原文保留）

本册是外部 MIT 原文的组件代码范式速查；**组件字段级语义（默认值、限制、烘焙期改写、组件间交互）已按 MA 官方文档 67 页对齐**，见四册：
- Setup Outfit / 烘焙期改写总表 / Manual Bake / 错误面板 / 分发：[02-10](../02-工程组织与工作流/02-10-ma-setup-outfit-and-general-behavior.md)
- MergeAnimator Layer Priority / Merge Motion / MMD / SyncParameterSequence / PlatformFilter / Parameters / MenuItem 语义：[03-07](../03-菜单与参数/03-07-ma-platform-sync-mmd-and-motion.md)
- 反应式家族（Object Toggle / Shape Changer / Material Setter·Swap / Blendshape Sync / Mesh Cutter）+ OT 极性全解：[04-06](../04-外观与表情/04-06-ma-reactive-components.md)
- 骨骼与几何 17 组件：[05-10](../05-动骨与穿模/05-10-ma-bone-and-geometry-components.md)

**组件分两类（SKILL 铁律 1 口径）**：烘焙期才生效（ObjectToggle/MaterialSetter/MaterialSwap/MeshCutter/MergeAnimator）vs 编辑态就有反馈（ShapeChanger 实时、BlendshapeSync 编辑模式复制值）。本册上文若有「一律编辑态不生效」的同义表述，以两类划分为准。
EP 参数权威规则（ValueType 枚举 / 位宽 API / saved / 内置参数撞名）：[03-01 §10](../03-菜单与参数/03-01-parameter-and-menu-audit.md)。
