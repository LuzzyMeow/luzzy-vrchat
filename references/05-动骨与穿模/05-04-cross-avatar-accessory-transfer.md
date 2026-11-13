# 跨模型配件迁移（翅膀 / 光环 / 配饰）——真资产 vs 烘焙网格、骨架缩放、MA 开关接线

> 本工作区实测 2026-10-26（LuzzyMeow V2，Unity 2022.3.22f1c1，MA 1.18.7 / NDMF 1.14.8 / AAO 1.9.20）
> 适用：把 A 模型的配件（翅膀、光环、尾巴、饰品）搬到 B 模型上，做成菜单可开关

---

## §1 症状 → 根因速查

| 症状 | 根因 | 去哪一节 |
| --- | --- | --- |
| 搬来的翅膀**浮在半空**（比头高 0.4 m） | 用了 `ModularAvatarMergeArmature`，但配件 prefab 是按**原模型尺寸**做的骨骼 bindpose，MergeArmature 只做骨骼映射、**不做尺寸重映射** | §3 |
| 缩小后翅膀**掉到脚下 / 变得极小** | 同时改了 prefab 根 scale 和 Armature scale → 双重缩放（0.547² ≈ 0.3） | §3.2 |
| 翅膀网格是对的但**材质发灰/发黑** | 用 AAO 烘焙产物（`AAOGeneratedMesh*`）重建时丢了原材质，或用了 `Hidden/lilToonOutline` 之类内部 shader | §2 |
| 菜单开关**点了没反应**，对象永远保持场景初始状态 | `ModularAvatarObjectToggle` 的 `AvatarObjectReference` 存的是**相对 avatar 根的路径字符串**，手动搬动对象后路径失效，烘焙时静默解析失败 | §4 |
| 开关**默认是关的**，但菜单里明明设了 value=1 | 参数没在 `ModularAvatarParameters` 里声明默认值 → MA 生成的 EP 参数默认 0 | §5 |
| 关掉菜单后对象**还在** | OT 只有 `Active=true` 一行时 MA 只生成 ON state，需要靠 `<default>` 空 state 回落；本工作区实测双向正常，但**必须用 `Animator.SetFloat` 实测双向** | §6 |

---

## §2 第一原则：要真资产，不要烘焙网格

**反例（本工作区真实踩过）**：用户要"迁移小天使翅膀和光环"，我从**旧工程的 AAO 烘焙产物**里抠网格：

```
AAOGeneratedMeshWings.asset / AAOGeneratedMeshHalo.asset   ← 烘焙后的合并网格
RebuiltMaterials/*.mat                                     ← 逆向工程重建的材质
```

结果：网格能用，但材质是 `Hidden/lilToonOutline`（内部 shader）、没有形态键、没有原动画，用户一眼看出「这不是那个模型的翅膀」。

**正确做法**：先找到**原始资产包**（BOOTH 购买的 `.unitypackage` / `.fbx` + 材质 + prefab），导入后直接用 prefab 实例。

本工作区实例：
```
<VRCHAT_ROOT>\resources\05_配饰\6103986天使翅膀光环\6103986 AngelNeonWing_すてらべりーにゃ\AngelNeonWing_StellaBerryNya.unitypackage
```
导入后结构：
```
AngelNeonWing                      [Transform, ModularAvatarMeshSettings, Animator(AngelNeonWing)]
└─ Armature                        [ModularAvatarMergeArmature: mergeTarget=AOR("Armature"), LockMode=BaseToMerge, mangleNames=True]
   └─ Hips > Spine > Chest > Neck > Head > Ring     ← 6 根骨
      ├─ Ring   SMR 319v  bones=6  root=Hips  8 形态键
      └─ Wing   SMR 1876v bones=6  root=Hips  6 形态键 2 材质槽
```

判据：**有 prefab + 独立材质 + 形态键 + 原动画 clip = 真资产**；只有 `AAOGeneratedMesh*` / `*.asset` 网格 = 烘焙产物，只能当"临时能用"，交付前要跟用户确认。

---

## §3 骨架缩放：MergeArmature 什么时候能用

### 3.1 能用 MergeArmature 的前提
- 配件骨骼名能对到宿主骨架（`Hips/Spine/Chest/Neck/Head`）
- **尺寸比例接近**（原模型身高 ≈ 宿主身高）

### 3.2 不能用时：手动缩放骨架（本工作区最终方案）

本工作区数据（AngelNeonWing 原尺寸 vs 宿主）：
```
原 prefab：Hips y=0.7641  Chest 0.8675  Head 1.057   Ring 骨 1.1829
           Wing AABB y=[0.857,1.012] x=±0.224  z=[-0.125,-0.119]
           Ring AABB y=[1.301,1.309] x=±0.077
宿主角色：Hips y=0.4183  Chest 0.5061  Head 0.6414
          身高比 = 0.6414/1.057 ≈ 0.607（按 Head 骨）
```

**做法**（关键：只动 `Armature`，不动 prefab 根）：
1. 把 prefab 实例挂到宿主 `armature/Hips` 下（`SetParent`），`localPosition=0 localRotation=identity localScale=1`
2. 删掉 `Armature` 上的 `ModularAvatarMergeArmature`（不再需要合并）
3. 设 `Armature.localScale = 0.5474`（本工作区取 Chest/Head 加权后的值）
4. 设 `Armature.localPosition = (0, -0.4183, 0)`

**第 4 步的坑**：AngelNeonWing 的 `Armature.localRotation = (270.02, 0, 0)` —— 这个 270° 旋转会把 `localPosition` 的 **z 分量旋到世界 y 方向**。所以要让骨骼下降 0.4183 m，必须写 `localPosition = (0, -0.4183, 0)` 而不是 `(0, 0, -0.4183)`。

通用公式：
```
Armature.localPosition = -宿主Hips的世界y × (Armature.localRotation 把 local +Y 映射到世界哪个方向的逆)
```
实操：先在 Inspector 里试 `(0, -Δ, 0)` 和 `(0, 0, -Δ)` 两种，用 `BakeMesh` 算 AABB 判断哪个对。

### 3.3 验收判据（本工作区实测通过值）
```
Wing AABB y=[0.469,0.554] x=±0.117 z=[-0.074,-0.055]   ← 在背后（角色朝 +Z）
Ring AABB y=[0.712,0.717] x=±0.042 z=[-0.031,0.053]    ← 头顶上方
```
参考：宿主 `Body_Base` AABB y=[0.009,0.645] x=±0.343 z=[-0.033,0.073]。

**判定方法**（不要只靠肉眼/截图）：
```csharp
var m = new Mesh(); smr.BakeMesh(m);
// 逐顶点 TransformPoint 求世界 AABB，与参考对象（Head 骨 / Body_Base）比较
```

---

## §4 MA ObjectToggle 的路径陷阱（必看）

`nadena.dev.modular_avatar.core.AvatarObjectReference` 内部存的是 **`referencePath`（相对 avatar 根的路径字符串）**，不是对象引用。

**踩坑过程**：先把翅膀挂到 `沐梓溪喵/小天使翅膀光环`，OT 行 `referencePath = "小天使翅膀光环"`；后来改成挂到 `armature/Hips/小天使翅膀光环`，**OT 没更新** → Play 烘焙后翅膀永远保持场景初始（inactive），菜单开关无效，且**没有任何报错**。

**正确做法**：搬动对象后重新 Set：
```csharp
var aor = (AvatarObjectReference)e.GetType().GetField("Object", flags).GetValue(e);
aor.Set(newTargetGameObject);          // 会自动重算 referencePath
```
（`Set` 会写回 struct，所以要 `list[i] = e` 再赋值回 `m_objects`）

**验证**：烘焙后检查 FX 里有没有 `[MA Responsive: <对象名>/<对象名>] ... m_IsActive keys=2` 这一层。没有 = OT 没生效。

---

## §5 菜单参数默认值：必须在 MA Parameters 里声明

MA 会为菜单里用到的参数自动生成 EP 参数，**默认值 = 0**（Bool/Int/Float 都是）。

要让"饰品默认开启"，必须在某个 `ModularAvatarParameters` 里显式加一条：
```
nameOrPrefix = "Acc_WingHalo"
syncType     = Bool
defaultValue = 1
hasExplicitDefaultValue = true
saved        = true
```
（本工作区放在 `沐梓溪喵/一键穿衣设置` 的 MA Parameters 里，与其它 `Acc_*_v2` 并列）

只写菜单项 `Control.value = 1` **不够** —— 那只表示"点开时写 1"，不表示"初始就是 1"。

---

## §6 开关接线与验证

### 6.1 场景对象要设为 inactive
MA reactive 语义：`param=1 → 对象 Active 值`；`param=0 → 回落到场景初始状态`。
所以"默认开启"的饰品，**场景里要 inactive** + **参数默认 1**。

### 6.2 双向实测（不要只看曲线）
烘焙后根 `Animator.runtimeAnimatorController` 是 NULL（FX 挂在 descriptor 层），要手动赋上再驱动：
```csharp
var an = cat.GetComponent<Animator>();
an.runtimeAnimatorController = fxController;   // 从 descriptor.baseAnimationLayers 取 FX
an.Update(0f);
an.SetFloat("Acc_WingHalo", 1f); an.Update(0f); an.Update(0.02f);
Debug.Log(wing.activeSelf);   // True
an.SetFloat("Acc_WingHalo", 0f); an.Update(0f); an.Update(0.02f); an.Update(0.02f);
Debug.Log(wing.activeSelf);   // False
an.runtimeAnimatorController = null;   // 记得清掉，否则污染场景
```
注意：VRChat 菜单里的 Int/Bool 参数在 Animator 里都是 **Float**，用 `SetFloat`；`SetInteger` 会静默无效。

### 6.3 编辑模式预览副本
MA 未烘焙时对象不可见（场景初始 inactive）。为让编辑模式能看到默认外观，`默认外观预览（EditorOnly）` 容器里要放一份副本：
```csharp
var dup = Object.Instantiate(wing.gameObject, previewContainer, true);
dup.name = "预览_小天使翅膀光环";
dup.transform.position = wing.transform.position;   // 保持世界位姿
dup.SetActive(true);
```
容器 tag = `EditorOnly`（NDMF 烘焙时整容器删除），子对象不必单独标记。
`Instantiate` 会自动把 SMR 的 `bones` 重映射到副本内部的同名骨骼，无需手动重绑。

---

## §7 收尾清单

- [ ] Play 烘焙后确认 `WING_ACT=True`（默认开）+ AABB 与参考对象对齐
- [ ] `Animator.SetFloat` 双向断言（1→True / 0→False / 1→True）
- [ ] 编辑模式预览副本存在且可见（差异渲染 ≥ 2000 px @720²）
- [ ] `LightController` 生成的 5 个 anim 包含新配件路径（本工作区 `LightLimit.anim curves=310 wingHits=4`）
- [ ] 删 `Packages/com.vrcfury.temp`（Play 构建产物）
- [ ] 场景块级 diff：只应有预期的对象增删

---

## §8 从旧工程 prefab 重建配件 + MA 默认开启的两个硬坑（本工作区实测 2026-10-26）

### 8.1 症状
从旧工程 `Kipfel_Kisekae_Chinese.prefab` 里按 GUID 找回 `AvatarAccessory_WingsHalo`（Halo SMR 637v / Wings SMR 708v，bones 分别 1 根 Head 与 6 根 Wing.L/R 链）在 V2 重建后：
- 菜单项、MA Parameters、ObjectToggle 全都正确，但 **Play 烘焙后配件仍然不显示**。

### 8.2 根因（两个，缺一不可）
1. **MenuItem 的 `isDefault` 必须为 True** —— 只把 `MA Parameters` 的 `defaultValue=1 / hasExplicitDefaultValue=true / saved=true` 写对还不够；
  `ModularAvatarMenuItem` 上的 `isDefault` 字段（public field，反射 `GetField("isDefault")`）为 False 时，MA 不会生成「默认值层」，FX 里参数默认仍是 0 → OT 不触发。
   判据：烘焙后 `VRCFury FX.controller` 里该参数的 `m_DefaultFloat` 应为 1；EP（`VRCFury Params.asset`）里 `defaultValue` 也应为 1。
2. **承载 MenuItem 的代理 GameObject 必须是 active** —— 若该代理对象 `activeSelf=False`（常见于复制粘贴后忘了打开），MA 在烘焙时**整条 MenuItem 连同挂在它上面的 ObjectToggle 一起不处理**，表现为「菜单里有这一项、但点了没反应、默认也不显示」。
   判据：静态审查里遍历所有 MenuItem 打印 `c.gameObject.activeSelf`，必须全为 True。

### 8.3 SMR 的 transform 对蒙皮网格无效（重要）
实测：对 `Halo`/`Wings` 这两个 SkinnedMeshRenderer 改 `localPosition` 的 X/Y/Z，世界 AABB 位移 **全为 0**；把父容器整体 +Y 也不动。
原因：SMR 顶点位置 = `bones` 的 `bindposes × 骨骼当前矩阵`，与 renderer 自身 transform 无关。
→ 要移动/缩放蒙皮配件，只能：
- 改**骨骼**（本工作区做法：给 Halo 在 `Head` 下新增一根 `Halo_Offset` 骨，把 `bones[0]` 指过去，之后改这根骨即可 1:1 平移）；或
- 改网格顶点（`mesh.vertices` 按质心缩放/平移，见 `头饰放大` 那套做法）。

### 8.4 世界坐标 1:1 轴映射实测（给用户的操作说明依据）
- 新建偏移骨（父 = Head，rotation 与 Head 一致）：`localPosition` 的 X/Y/Z → 世界 X/Y/Z **1:1**。
- `Wing.L`/`Wing.R`（父 = Chest，rotation 与 Chest 一致）：X 1:1，**Y/Z 只有 0.5 倍效率**（改 0.05 → 世界 0.025），因为这两根骨的父级有非均匀缩放。
- 缩放偏移骨会让**中心同时上飘**（scale 1.3 → 直径 0.1484→0.1929，中心 y 0.7604→0.8271），调完缩放要重新校 Y。

### 8.5 MCP 多实例路由坑
两个 Unity 工程同时开着时，`mcpforunity://instances` 会列出两条（端口 6400 / 6401），MCP 默认路由到**先连上的那个**，于是 `execute_code` 读到的是另一个工程的场景（表现为「改了没效果」「对象找不到」）。
→ 动手前先 `set_active_instance(<Name@hash 或端口>)` 显式钉住目标实例。

### 8.6 收尾核对表（本轮实测数字，LuzzyMeow V2）
- 静态：`transforms=2130 SMR=140 MR=42 missingScripts=1`（既有睡衣 1 个）、`MenuItem=159 ObjectToggle=77 MergeArmature=10`、`MAParams=57 dup=0`
- 引用审计（`tools\scripts\audit_unity_refs.py <proj> <out_dir>`）：`asset_files=11767 meta_files=13766 dup_guid=0 meta_without_guid=0 orphan_meta=0 missing_meta=47 yaml=5127 edges=22325 unresolved=2515 distinct=493`（悬空边与解压基线同量级，无新增）
  
- 动态：Play 烘焙后 `OT Active=True / 目标 sceneActive=False / FX 参数默认 1` → 渲染差异法隐藏 A 得 5196 px、隐藏 B 得 4571 px（700² 背面视角），两套翅膀都真实可见
- 还原 `LightController` 生成的 5 个 anim（`NyaPPu__Generated\LightController\*.anim`，MD5 对原始包）；`Map_Texture`/`Cl_RenderTexture` 两份 renderTexture 若 MD5 未变则无需还原；删 `Packages/com.vrcfury.temp`（71 文件）
- PowerShell 陷阱：`<VRCHAT_ROOT>\$tmp\...` 在双引号里 `$tmp` 会被当变量 → 路径一律用**单引号**字面量

---

## §9 把「一件配件」拆成「两个可独立开关的部件」+ 安全删除配件（本工作区实测 2026-10-26）

### 9.1 症状 / 需求
原来「天使翅膀 + 光环」是一个对象 `AvatarAccessory_WingsHalo`，一个菜单项、一个参数、一个 ObjectToggle 一起开关。
用户要求：拆成「翅膀」和「光环」两个菜单项，各自独立开关；同时删掉另外两套已不需要的翅膀饰品。

### 9.2 正确做法（拆分的四个必须条件）
1. **每个部件一个菜单代理对象**（本工作区：`菜单开关/天使翅膀`、`菜单开关/天使光环`），各自带 `ModularAvatarMenuItem` + `ModularAvatarMenuInstaller` + `ModularAvatarObjectToggle`。
2. **OT 的 `Active` 必须为 True**，且 `AvatarObjectReference` 指向**子部件**（`AvatarAccessory_WingsHalo/Wings`、`.../Halo`），不是父容器；父容器保持 `activeSelf=True`、两个子部件 `activeSelf=False`（这是 MA reactive 的「场景初始状态」）。
   - 用 `AvatarObjectReference.Set(GameObject)` 反射重设后，`referencePath` 会自动变成相对 avatar 根的路径；搬动过对象后**必须重设**，否则路径仍指向旧位置（表现为「菜单有、点了没反应」）。
3. **每个部件一个独立参数**（`Acc_Wings` / `Acc_Halo`），并在 `一键穿衣设置` 的 `ModularAvatarParameters` 里各登记一条 `Bool / defaultValue=0 / hasExplicitDefaultValue=true / saved=true`。
4. **MenuItem 的 `isDefault` 保持 False**（默认关闭）。注意：`isDefault=True` 会生成「默认值层」把参数压到 1，与「默认关闭」相反。

### 9.3 烘焙后的验收判据（比看 Hierarchy 可靠）
烘焙产物 `Packages/com.vrcfury.temp/Builds/____/VRCFury FX.controller`：
- 每个 OT 生成**一层** `MA Responsive: <代理对象名>`；层内状态名为菜单 label，clip 只写两条曲线：
  `bind path='AvatarAccessory_WingsHalo/Wings' prop=m_IsActive v=1` 和 `bind path='' prop='__MA/ActiveSelfProxy/Wings##NNN' v=1`。
- `<default>` 状态的 transition 条件就是该部件自己的参数：`Acc_Wings Greater 0.995; Acc_Wings Less 1.005`。
  → **两层各自独立条件 = 拆分成功**；若两层条件里出现同一个参数，说明 OT 目标或参数没拆开。
- EP（`VRCFury Params.asset`）里应只有新参数、旧参数（`Acc_Wing_v2` / `Acc_WingHalo` / `Acc_WingsHalo`）必须消失；本工作区实测 `EP count=164 唯一=164`，无残留。

### 9.4 安全删除配件的顺序（避免悬空引用）
1. 先删**菜单代理对象**（`菜单开关/*`、`角色菜单_小狼/*`）→ 连带 MenuItem + MenuInstaller + ObjectToggle 一起消失，不会留下指向已删网格的 OT 行。
2. 再删**场景里的真实对象**与**预览副本**（`默认外观预览（EditorOnly）/预览_*`）。
3. 删完做悬空检查：遍历所有 `ModularAvatarObjectToggle`，逐行读 `m_objects[i].Object.targetObject`，**必须全部非 null**（本工作区实测 `dangling OT rows=0`）。
4. 参数清理：删掉 `ModularAvatarParameters` 里对应的登记项（本工作区 `MAParams 57→56`，`dup=0`）。
5. 菜单资产若被删空，记得把父菜单里的 SubMenu 控件也删掉。

### 9.5 两个新踩的坑
- **AAO 烘焙后 `Renderer.bounds` 全部相同、且是整个头像的大盒子**（本工作区所有合并 SMR 都报 `size=(2.000,2.500,2.000) center=(0,0.168,0.011)`）
  → Play 里**不能用 `Renderer.bounds` 量单个部件的位置/尺寸**，要量就在编辑模式（未烘焙）量，或者用「只开该 renderer 渲染一张、看像素 bbox」的办法。
- **Play 里相机朝向**：`Quaternion.Euler(0,90,0)` 的 forward 是 **+X**。相机放在 `(1.9,0.62,0)` 想拍侧面必须用 `y=-90`；用 `y=90` 会拍到背面之外的空白（本工作区第一次侧视图 `nonbg=0` 就是这个原因）。

### 9.6 本轮实测数字（LuzzyMeow V2，删 2 套 + 拆 1 套之后）
- 静态：`transforms=2097 SMR=134 MR=41 missingScripts=1`（既有睡衣 1 个）、`MenuItem=157 ObjectToggle=66 MenuInstaller=40`、`MAParams=56 dup=0`、`dangling OT rows=0`
- 烘焙菜单：`232` 项，含翅膀/光环的只剩 4 项（`小猫菜单/饰品/天使翅膀`、`/天使光环`、`小狼菜单/饰品/天使翅膀`、`/天使光环`），参数 `Acc_Wings` / `Acc_Halo` 均 `def=0 saved=True net=True`
- 默认态：烘焙克隆里 `Halo/Wings` 两个子部件都 `activeInHierarchy=False`
- 视觉：单独渲染 Halo 得 56×9 px 细环（位置与编辑态一致），只开 Wings 相对基线差 2328 px → 两个部件都真实可渲染
- Console：0 error（40 条全是既有的 `_BacklightColor` / `EyeMaskNightMode` 警告）
