# PhysBone 与动态物理

## 反射访问 VRCPhysBone

`VRCPhysBone` 类型不能直接 import（SDK 程序集未 auto-reference），用反射拿类型：

```csharp
System.Type pbType = null;
foreach (var mb in avatar.GetComponentsInChildren<MonoBehaviour>(true))
    if (mb != null && mb.GetType().Name == "VRCPhysBone") { pbType = mb.GetType(); break; }

var pb = handle.AddComponent(pbType);
var so = new SerializedObject(pb);
so.FindProperty("pull").floatValue = 0.8f;
so.ApplyModifiedProperties();
```

前提是场景里已经存在至少一个 PhysBone 可以"借"类型。avatar 上一般都有。

---

## 序列化字段表（实测有效）

| 字段 | 类型 | 说明 |
|---|---|---|
| `pull` | float | 回复力（0 = 停在被拉到的位置，1 = 迅速弹回） |
| `spring` | float | 振荡 / 弹性 |
| `stiffness` | float | 抵抗外力的刚度 |
| `gravity` | float | 向下拉力（负值 = 向上飘） |
| `gravityFalloff` | float | 直立时的重力衰减 |
| `immobile` | float | 对 avatar 移动的抗性（1 = 世界空间锁死） |
| `immobileType` | int | 0 = All Motions，1 = World（忽略 avatar 位移） |
| `radius` | float | 碰撞半径 |
| `maxStretch` | float | 最大拉伸系数（0.5 = 允许多拉 50% 长度） |
| `maxSquish` | float | 最大压缩系数 |
| `allowGrabbing` | int | 0 = 关，1 = 开 |
| `allowPosing` | int | 0 = 关，1 = 开（抓完可摆姿势） |
| `grabMovement` | float | 被抓住时的跟手刚度 |
| `parameter` | string | 输出参数前缀 → `{prefix}_Stretch`、`{prefix}_Angle`、`{prefix}_IsGrabbed` |
| `colliders` | array | VRCPhysBoneCollider 引用列表 |

---

## 调参配方

| 用途 | pull | spring | immobile | gravity | 备注 |
|---|---|---|---|---|---|
| 胸部软体（不下垂） | 0.8 | 保持原值 | 0.6 | 0 | 重力归零消除下垂，pull 负责回位 |
| 可抓拉链 / 不回弹 | 0 | 0.12 | 0.85 | 0 | 抓完停在原地 |
| 软硬滑块 · 软端 | 0.8 | 0.1 | 0.7 | — | radial puppet 的 0 端 |
| 软硬滑块 · 硬端 | 0.15 | 0.9 | 0.75 | — | radial puppet 的 1 端 |

### "拉到哪停到哪"（可抓、不回弹）

```text
pull          = 0      // 没有回复力
immobile      = 0.85
allowGrabbing = 1
allowPosing   = 1
grabMovement  = 0.5
radius        = 0.012  // 半径小 = 必须凑很近才抓得到
```

抓取用的是 **Grip 键**。松手后骨头停在当前位置，要靠用户手动推回去。

### 胸部软体调参

批量改软体骨链时注意两点：

- **跳过 `immobile >= 0.99` 的骨头** —— 那些是刚性结构锚点，不是弹跳链的一部分。
- **限定在目标骨链内**：改之前检查 `path.Contains("<你的软体 PB 名>")`，避免误伤头发或裙子。

---

## 用动画驱动 PhysBone 属性

PhysBone 的 float 字段可以被动画曲线驱动 —— 做"软硬度"radial puppet 就靠这个：

```csharp
clipSoft.SetCurve(path, pbType, "spring",    AnimationCurve.Constant(0, 1f/60f, 0.1f));
clipSoft.SetCurve(path, pbType, "pull",      AnimationCurve.Constant(0, 1f/60f, 0.8f));
clipSoft.SetCurve(path, pbType, "stiffness", AnimationCurve.Constant(0, 1f/60f, 0.4f));
clipSoft.SetCurve(path, pbType, "immobile",  AnimationCurve.Constant(0, 1f/60f, 0.7f));
```

把 soft / firm 两个 clip 塞进 1D BlendTree，用一个 float 参数插值即可。

---

## 远端同步的注意事项

- `_Stretch`、`_Angle`、`_IsGrabbed` 这些**参数**会同步给远端玩家。
- 但精细的物理运动（spring / gravity 产生的抖动）是**本地模拟**的，远端看到的不会和本地完全一致。
- 结论：抓取类交互一定要在 VR 里和真人对测，光看镜像不算验证。

---

## 头发 / 衣服穿模：多半是 collider 没挂上

动态头发穿身体通常**不是 mesh 问题**，而是没给头发的 PhysBone 分配碰撞体。

排查顺序：

1. 查头发 PhysBone 的 `rootTransform`。
2. 查它引用的 `colliders` 列表和 `bones`。
3. 查身体/头/胸/肩/上臂上有没有合适的 collider。
4. **光在身上加 collider 不会自动生效** —— 必须挂到对应 PhysBone 的 collider list 里。
5. 静态不穿模、一动就穿模时，优先调 collider 的半径、高度、`insideBounds`，以及 PhysBone 的 `stiffness` / `pull` / `spring`。

---

## 性能

PhysBone 数量直接影响 avatar 性能评级。压缩时：

- 优先合并/删除没人看得出来的短骨链（手指、小配件）。
- 头发和胸部这种视觉主体保留。
- 检查有没有插件重复安装导致的 PhysBone 翻倍。

---

## 补充：官方语义核对（2026-11-11 联网核实，原文保留）

> 来源：VRChat 官方 PhysBones 文档 <https://creators.vrchat.com/common-components/physbones/>（2026-11-11 提取全文核对）。本节只补官方口径，不改动上文实测配方；作者口径的逐参数调试循环另见 [05-02 §5](../05-动骨与穿模/05-02-bind-animation-physbone-fbx.md)。

### 版本与积分模式

- **Version 1.0 vs 1.1**：1.1 是 Squishy Bones 更新（骨可 squish、长度受运动影响）；1.1 的 Gravity 是「静止时骨旋转程度的比例」，需要正 Pull 才向重力方向动；Stiffness 是「保持上一朝向的比例」。组件上可直接选版本；旧 PhysBone 不手动升级并重传就保持旧版本。
- **Integration Type**：`Simplified` 更稳、对外力反应慢、易配置（有 Spring）；`Advanced` 反应更快、可配置更复杂（有 Momentum/Stiffness）。默认设置两者接近。
- 大多数滑条可按 C 按钮用 **Curve** 沿骨链长度变化。

### Transforms（官方硬规则）

- `Root Transform` 留空 = 从本 GameObject 开始；`Ignore Transforms` 自动包含其全部子级；`Ignore Other Phys Bones` 默认开，关掉可能产生不支持的行为。
- **单骨、或单根多子（无孙）时必须设 `Endpoint Position`**（通常 +Y），否则 PhysBone 不工作——这点与 Dynamic Bone 不同。也可在每条链尾加 end bone（要改骨架）。
- `Multi-Child Type`：**Ignore**（根不动，适合一个组件管全部头发）/ **First**（根与第一条链连续）/ **Average**（根取各链平均）。

### Forces 与 Limits

- `Pull` 回_rest 的力；`Spring`（Simplified）/`Momentum`（Advanced）摆动；`Stiffness`（Advanced）保持静止朝向；`Gravity` 正下负上；**`Gravity Falloff`=1.0 时静止位完全不受重力**（头发建模即目标姿态时用 1.0；想要自然下垂弧度用 0.5–0.8）。
- `Immobile Type`：**All Motion**（默认， dampen 全部运动）/ **World (Experimental)**（只 dampen 场景根参照的位置移动，动画/IK 照常；playspace 移动照常、摇杆 locomotion 被 dampen）。
- **Limits 比 collider 性能高得多**（官方原话 far more performant）：None / Angle（锥）/ Hinge（扇形）/ Polar（球面段，Max Pitch/Yaw）。**Polar 有非零性能成本，超过 ~64 个会出问题**；Max Pitch≈Max Yaw 时用 Angle 更省。选中链时 Scene 视图有限制可视化。

### Collision / Stretch & Squish / Grab & Pose

- `Radius` 同时用于碰撞与抓取；`Allow Collision` True（含全局 collider 与他人手）/ False（只 Colliders 列表）/ Other（按 avatar/world 细分过滤）。
- `Stretch Motion`（运动引起的拉伸量，0=只有抓取/碰撞才拉伸）、`Max Stretch`/`Max Squish`（原骨长倍数）。
- `Allow Grabbing`/`Allow Posing` 可按「自己/他人」过滤；`Grab Movement` 0=用 pull&spring 趋近、1=立即到位；`Snap To Hand` 吸附到抓取骨。

### Options 与参数

- `Parameter` 前缀生成 `{p}_IsGrabbed`[Bool]、`{p}_IsPosed`[Bool]、`{p}_Angle`[Float 0–1，归一化 180°]、`{p}_Stretch`[Float 0–1]、`{p}_Squish`[Float 0–1]。**这些参数不需要在 EP 里声明 synced**——本地与远端都跑 PhysBones，自动双端更新。
- **`Is Animated` 必须开**，链上任何骨（含根）才会响应动画；每帧按动画更新 rest 位。
- **`Reset When Disabled`**：组件禁用时骨自动回默认位、参数归 false/zero（眼罩隐藏时关 PB_Root 的官方依据，见 [05-01 §15](../05-动骨与穿模/05-01-outfit-fit-and-clipping.md)）。

### 官方禁忌与限额

- **同一 GameObject 不要同时受 Constraint 和 PhysBone 影响**（执行顺序问题）；把 Constraint 挂到父物体，source 仍可指原物体。
- **Humanoid 骨（Hip/Spine/Chest/UpperChest/Neck/Head/四肢）不要当 PhysBone Root**；在 Blender 里复制该骨做新根、把要动的子骨重定父级过去（Cat's Blender Plugin 可做）。MA 侧对应警告见 [02-10 §1.9](../02-工程组织与工作流/02-10-ma-setup-outfit-and-general-behavior.md)。
- **单组件最多影响 256 个 transform**（含根与全部子级；DB 转换同样受限）；超过 ~128 建议拆组件（多线程分摊）；32 骨级别不用纠结。
- **属性（Spring/Pull/Stiffness 等）在初始化时设定，avatar 上不可动画**；「动画属性+开关组件」的偏方不受支持、随时可能坏。world 里可用 Udon 改（改完必须 `ApplyConfigurationChanges()`，且开销大要批处理）。
- 根骨**允许旋转、不允许平移**（与 DB 不同）；`Immobile` 以 root transform 为基准（DB 的 Inert 以组件位置为基准，属 DB bug）。
- **Quest 有硬限额**（按 Very Poor 档，见官方 avatar-performance-ranking-system#mobile-limits）；world 无硬限但仍应节制。
- 碰撞包围盒上限 **10×10×10 m**（超出后骨仍工作，但玩家可能摸不到/抓不到）；只统计 radius>0 的碰撞骨。

### Collider 官方口径

- 形状 Sphere / Capsule / Plane；**Global Collision 只支持 Sphere/Capsule**；`Inside Bounds` 把骨关在里面而不是挡在外面；`Bones As Sphere` 把骨碰撞半径当球而非胶囊。
- **avatar 最多 4 个开 Global Collision 的 collider**（world 无限制）。MA Global Collider 组件超过 6 个会顶掉 index finger collider 的实测见 [05-10 §15](../05-动骨与穿模/05-10-ma-bone-and-geometry-components.md)。
- Descriptor 的 **Standard Colliders**（Head/Torso/Hands L/R/Feet L/R/Fingers L/R 四指）自动设置、**不影响性能评级**；主要作为 Contact Sender（内置身体部位 tag），手/指 collider 同时是全局 PB collider。

### Dynamic Bone 转换

- 客户端加载 avatar 时**自动把 DB 转成 PhysBones**（Advanced 模式 + Multi-Child Ignore）；DB 已在 avatar 语境完全移除。转换不保证行为完全一致；X/Z 方向的 Force 无对应被忽略。
- SDK 手动转换：`VRChat SDK/Utilities/Convert DynamicBones to PhysBones`（先选中 avatar；**删 DB 组件不可轻易逆，先备份**）。
