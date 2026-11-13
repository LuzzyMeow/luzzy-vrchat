# 网格、形态键与穿模

---

## 穿模（衣服穿过身体）的正确解法

### 第一步：先排除编辑器假象

MA 的 `BlendshapeSync` / `ShapeChanger` / `ObjectToggle` **在普通编辑器场景视图里不生效**。它们只在 Play 模式、Manual bake、或上传时应用。

**所以：编辑器视图里看到的穿模，绝大多数是假的。** 诊断任何穿模之前先 bake：

```csharp
EditorApplication.ExecuteMenuItem("Tools/Modular Avatar/Manual bake avatar");
// 需要先选中 avatar；生成 <Avatar>_Baked 副本 → 多角度渲染 → 删掉副本
```

渲染**正面、侧面、腰部**至少各一张。只看正面会漏掉侧向穿模。

### 衣服是怎么贴合身体的

两套互补机制：

| 组件 | 作用 |
|---|---|
| `MA BlendshapeSync` | 把身体某个 blendshape 映射到衣服的 blendshape。身体形态变化时衣服自动跟随。 |
| `MA ShapeChanger` | 穿着这件衣服时直接设置身体（或衣服）的 blendshape 值。 |

**每套衣服都是按某一个特定的身体形态雕出来的**，只有身体处于那个形态时衣服才贴合。

### 关键错误：不要缩身体

看到胸部穿模，最容易想到的是加一个收缩 blendshape 把身体压进去。**这是反的：**

- 衣服是按**原始/完整**身体形态做的尺寸。
- 缩身体会让衣服**浮起来、露缝**，不再贴合。
- 结果（飘着的布、布和皮肤之间的空隙）比原来的穿模更难看。

### 正确做法：激活衣服本来要跟随的那个形态键

找出衣服的 `BlendshapeSync` 映射到了身体的哪个 blendshape，然后确保穿这件衣服时那个 blendshape 是激活的（通过挂在衣服上的 `ShapeChanger`）。

**典型例子（无内衣类衣服）**：这类衣服的 `BlendshapeSync` 通常映射到身体的某个 "no bra" 系列形态键，网格是按那个状态雕的。修法：

1. 在这件衣服的胸部覆盖件上加一个 `ShapeChanger`（衣服穿着时生效）。
2. 把身体对应的 no-bra 系列 blendshape 设为 100。
3. 结果：衣服贴合完整尺寸的胸部。身体几何**保持原样，不缩**。

**反例**：作者说明里写"按默认体型设计"（内衣穿着、标准胸型）的衣服，就**绝不能**开 no-bra 形态键 —— 那会把身体推离衣服所期待的形状。

### 先读作者说明

Booth / 作者 README 通常会写清楚需要的身体设置。常见表述：

- 「シェイプキー使用必須」/「形态键必须启用」→ 特定 blendshape 不开必穿模
- 「ノーブラ」/「no bra」→ 需要 no-bra 形态键激活
- 「デフォルト乳形」/「default body」→ 必须用基础体型，不能开 no-bra
- 明确的开关清单（"穿之前请启用 XX 和 YY"）

有些穿模（例如某些衣服的腋下区域）是**作者已知并承认的限制**，不重新拓扑就修不好，说明里可能已经写了。不要在这种地方浪费时间。

### fitdump 诊断法

衣服穿模但原因不明时，用一次编辑器脚本把 MA 配置全 dump 出来：

```csharp
// 每件衣服的 ShapeChanger 都设了什么
foreach (var sc in avatar.GetComponentsInChildren<ModularAvatarShapeChanger>(true))
    foreach (var sh in sc.Shapes)
        Debug.Log($"[fitdump][SC] {sc.gameObject.name}: {sh.ShapeName}={sh.Value} ({sh.ChangeType})");

// 每件衣服的 BlendshapeSync 绑定了什么
foreach (var bs in avatar.GetComponentsInChildren<ModularAvatarBlendshapeSync>(true))
    foreach (var b in bs.Bindings)
        Debug.Log($"[fitdump][BS] {bs.gameObject.name}: body[{b.LocalBlendshape}] -> cloth[{b.Blendshape}]");

// 身体网格当前所有非零 blendshape
var bodySmr = avatar.transform.Find("Body")?.GetComponent<SkinnedMeshRenderer>();
if (bodySmr != null)
    for (int i = 0; i < bodySmr.sharedMesh.blendShapeCount; i++)
        if (bodySmr.GetBlendShapeWeight(i) > 0)
            Debug.Log($"[fitdump][BODY] {bodySmr.sharedMesh.GetBlendShapeName(i)}={bodySmr.GetBlendShapeWeight(i):F0}");
```

然后对比：**衣服的 `BlendshapeSync` 期待身体处于什么状态** vs **身体当前实际是什么状态**。差在哪就加一个 `ShapeChanger` 补上，再 bake + 渲染确认。

---

## 遮盖系统（blendshape cover）

很多 avatar 用一组 blendshape 来处理"穿衣服时压平/隐藏身体细节"：

| 形态键类型 | 效果 | 什么时候设 |
|---|---|---|
| `(Wear_Bra)_*` / `Shrink_*` | 压平、收缩身体局部 | 对应衣服穿着时 |
| 细节显示类（如乳头几何） | 显示身体细节几何 | 裸体 / 暴露款衣服时 |

### 常见需求：薄衣服不透细节

一件不遮胸的衬衫，正确做法是给它加一个 `ShapeChanger`，**只**把细节形态键设为 0，**不要**同时设置压平形态键。这样能隐藏细节又不制造下面说的"黑洞"。

### 黑洞（胸口暗腔）

**根因**：压平类 blendshape（`(Wear_Bra)_*` / `Shrink_Breast`）仍然生效，但对应的遮盖网格已经不在了 —— 身体被压进去而皮肤照常渲染，形成一个暗腔。

**快速诊断（不用渲染）**：直接在编辑态读 ShapeChanger 数据：

```csharp
foreach (var sc in avatar.GetComponentsInChildren<ModularAvatarShapeChanger>(true))
    foreach (var sh in sc.Shapes)
        Debug.Log($"{sc.gameObject.name}: {sh.ShapeName}={sh.Value} ({sh.ChangeType})");
```

如果某个压平形态键没有被正确的衣服 gate 住，它就会**永远生效** → 黑洞。

**修复顺序**：

1. 找出所有设置压平形态键的 ShapeChanger。
2. 确保每个都挂在对应衣服的物体下，并被服装 int 参数 gate（见 [00-02-modular-avatar.md](00-02-modular-avatar.md)）。
3. 衣服通过 toggle 脱掉时，它的 ShapeChanger 也必须一起失效。

---

## Blender 无头网格编辑

### 启动

```powershell
& "<BLENDER_EXE>" -b -P "<SCRIPT>.py"
```

`-b` = 后台无 GUI，`-P` = 执行 Python 脚本，输出走 stdout。

### bisect + split_edges（切开网格）

**关键认知**：`bmesh.ops.bisect_plane` 作用在扁平的 bmesh 上，**形态键要单独处理**（先 bisect 再重新应用 delta，或者接受丢失）。必须精确保留形态键时，在副本上作业。

```python
import bpy, bmesh, mathutils

bpy.ops.wm.read_factory_settings(use_empty=True)
bpy.ops.import_scene.fbx(filepath=r"<SOURCE>.fbx")
obj = bpy.data.objects.get("Top")
me  = obj.data

bm = bmesh.new()
bm.from_mesh(me)
bm.verts.ensure_lookup_table()

geom_in = list(bm.verts) + list(bm.edges) + list(bm.faces)
res = bmesh.ops.bisect_plane(
    bm,
    geom=geom_in,
    dist=0.001,
    plane_co=(0, 0, 0),       # 切面上一点
    plane_no=(1, 0, 0),       # 切面法线（X=0 即正中垂直切）
    use_snap_center=False,
    clear_outer=False,        # 两半都留
    clear_inner=False,
)

# 把切出来的边劈开，形成开口缝
bmesh.ops.split_edges(
    bm,
    edges=[e for e in res["geom_cut"] if isinstance(e, bmesh.types.BMEdge)]
)

bm.to_mesh(me)
bm.free()

bpy.ops.export_scene.fbx(
    filepath=r"<OUT>.fbx",
    use_selection=True,
    global_scale=1.0,             # 见下面的 scale 陷阱
    apply_unit_scale=True,
    apply_scale_options='FBX_SCALE_NONE',
    bake_space_transform=False,
    object_types={'MESH', 'ARMATURE'},
    use_mesh_modifiers=True,
    add_leaf_bones=False,
)
```

### BVHTree 收缩包裹（把切口顶点吸附到身体表面）

```python
import mathutils

body = bpy.data.objects.get("BodyBase")
bm_body = bmesh.new()
bm_body.from_mesh(body.data)
bvh = mathutils.bvhtree.BVHTree.FromBMesh(bm_body)

for v in bm.verts:
    if v.tag:                       # bisect 时给切口顶点打的标记
        loc, norm, idx, dist = bvh.find_nearest(v.co)
        if loc and dist < 0.01:
            v.co = loc

bm_body.free()
```

### globalScale 陷阱

Blender 用 `global_scale=1.0` 导出、Unity `ModelImporter` 默认 `globalScale = 1` —— 但 avatar 厂商的 FBX 通常是按 `global_scale=100` 导出的。结果就是你切出来的网格进 Unity 变成 1/100 大小。

两种修法，任选其一：

```csharp
// Unity 侧
var imp = AssetImporter.GetAtPath("Assets/_Gen/MyCut.fbx") as ModelImporter;
if (imp != null) { imp.globalScale = 100f; imp.SaveAndReimport(); }
```

```python
# 或者 Blender 导出时就对齐
bpy.ops.export_scene.fbx(..., global_scale=100)
```

---

## 身体缺件（手/脚/身体不见了）

不要默认压缩身体来适配衣服，尤其是透明/半透明衣服、全脱、部分脱衣、互动衣服的场景。

排查顺序：

1. body renderer 当前**非零的 blendshape** 有哪些。
2. 衣服菜单的 `ShapeChanger` / `BlendshapeSync`。
3. `ObjectToggle` 是不是关掉了身体的局部 mesh。
4. optimizer 是不是删了身体 renderer 或某个 material slot。
5. 衣服自己是不是带了防穿模 blendshape。

如果全脱之后手臂/小臂/脚仍然缺失，说明**某个身体隐藏/压缩状态没有恢复**。去修参数默认值和状态机，不要继续叠加新的压缩。

---

## 补充：结构化分册路由与 Blender 官方口径（2026-11-11 整理，原文保留）

- 跨头像衣装等比 bindpose 重绑 / Kabsch / NewFit 边界 / UV IoU 贴图移植：[05-08](../05-动骨与穿模/05-08-cross-avatar-garment-fit-newfit.md)。
- 重绑后 bindPoses 重算公式 / localBounds 归零 / 手套权重 / invalid normals / Humanoid→Generic：[05-11](../05-动骨与穿模/05-11-skinning-binding-and-geometry-repair.md)。
- 袜子膝穿模顶点缩放（x108xz 现行版）/ 眼罩 PB 门控：[05-01 §12–§15](../05-动骨与穿模/05-01-outfit-fit-and-clipping.md)。
- **Blender 合并形态键官方口径**（Blender Manual「Shape Keys Panel」specials，第 143 轮核对）：`Transfer Shape Key` / `Join as Shapes` **只要求顶点数一致，不校验拓扑顺序**——先比坐标/法线序列哈希确认顺序，不一致先重排再 Transfer，或按「新基底 + delta」逐顶点计算。顶点数不同但三角形数与顺序一致时用三角形角点双射（[04-08 §7.2](../04-外观与表情/04-08-face-tracking-osc-and-repair.md)）。
- Blender FBX 往返缩放（globalScale=0.01）：[04-08 §7.4](../04-外观与表情/04-08-face-tracking-osc-and-repair.md)；本册 globalScale 陷阱节同源。
