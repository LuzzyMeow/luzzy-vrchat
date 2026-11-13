# 换头系统的「拆」与「克隆成新角色」

> 本工作区实测 2026-10-27（第 102 轮，LuzzyMeow V2）。触发任务：一只带换头系统的猫系头像要①删掉「狼头」回到纯粹小猫、②用一个「小狼2.0」的狼头 + 猫的身体克隆出**独立的新角色**，两者共存于同一场景但各有自己的菜单。
> 前提：这个头像的「换头」是**同场景内**实现——狼头整组挂在 `armature/.../Head/狼头` 下（默认 inactive），靠 `HeadType`(Int，猫=1/狼=2) 的两个 MenuItem Toggle + 一个 `换头FX`(MA MergeAnimator，
> 挂 `LilLeo_FX_FaceOnly` 7 层狼脸) 切换。详见 [05-03 §11](05-03-model-anatomy-and-head-swap.md)。

---

## 1. 结论先行（三条）

1. **「拆」比「装」简单得多**：装换头要重指引用、克隆骨链、追加 FX 层；拆换头只要删对象 + 清死行 + 删菜单宿主 + 删那一个 MergeAnimator，**不需要重建任何动画**。
2. **「克隆成新角色」比从零拼装安全 10 倍**：`Object.Instantiate` 整个头像根 → 改名 → 清 blueprintId → 复制 EP → 再做减法（删猫件、删换头、删旧菜单、复制 FX 控制器）。这样衣柜/一键穿衣/ABT/亮度/眼罩/PhysBone 全部原样继承，避免逐个重建时漏参数、漏层、漏绑骨。
3. **共享资产必须先复制再改**：菜单 `VRCExpressionsMenu` 资产、FX/SPS 控制器、EP 资产被两个头像共用时，**直接改 = 两个角色一起变**。克隆体的每一次「改菜单/改控制器/改参数默认值」前都要先 `AssetDatabase.CopyAsset` 出副本并 `Set` 到自己身上。

---

## 2. 删除换头系统（回到单一头部）

### 2.1 顺序（必须按这个顺序，否则会留死引用）

1. 先删**菜单代理**：`菜单开关/头部`（SubMenu 宿主，`installTargetMenu` → 根菜单资产）、各角色的 `角色菜单_<另一角色>`（整棵菜单副本宿主）。
2. 再删**换头动画载体**：`换头FX`（`Transform + ModularAvatarMergeAnimator`）。它的 controller 里全是「切换瞬间」的曲线，删掉不会影响任何常驻开关。
3. 再删**另一颗头整组**：`armature/.../Head/狼头`（含 Body/Cat_Ear/Hair_Back/Hair_Base SMR、`狼头版头饰` 变体、狼头眼罩）、`WolfHead_Root*` 发骨链 × N、`狼眼_EyeL/EyeR`。
4. 最后清 `ModularAvatarObjectToggle` 的**死行**（见 §2.2）与 `MA Parameters` 里只服务于另一颗头的参数。

### 2.2 死行清理：Unity「假 null」坑（本工作区实测）

删除 GameObject 后，`AvatarObjectReference.targetObject` 里存的是**已销毁的托管引用**。用 `object tg = row.Object.targetObject; if (tg != null)` 判断 → **永远为 true**，一行都清不掉（实测 `rowsRemoved=0`）。
正确写法：

```csharp
UnityEngine.Object uo = row.Object.targetObject as UnityEngine.Object;
if (uo != null) keep;              // 或者： if (ref.referencePath 在 root.Find() 下解析不到) remove
```

判据：清完打印 `otTotal / rowsKept`，本工作区实测 **63 个 OT、89 行保留、死行 0**；再用「遍历所有 OT 行，`targetObject==null` 或路径解析不到」的审计脚本复核一遍必须为 0。

> 另一个等价判据：`AvatarObjectReference.referencePath` 是**相对 avatar 根**的路径字符串；对象删掉后 `root.Find(path)` 返回 null，比 `targetObject` 更可靠（不受托管引用生命周期影响）。

---

## 3. 克隆成第二个角色（本工作区实测流程）

### 3.1 克隆与身份重置

```csharp
var go = (GameObject)UnityEngine.Object.Instantiate(catRoot, catRoot.transform.parent);
go.name = "沐梓溪狼";
go.transform.SetSiblingIndex(catRoot.transform.GetSiblingIndex() + 1);
// 清 blueprintId（否则会被当成同一只已上传头像）
var pm = go.GetComponent<VRC.SDKBase.VRC.SDK3.Avatars.Components.PipelineManager>(); // 实际类型 PipelineManager
pm.GetType().GetField("blueprintId", BindingFlags.Public|NonPublic|Instance).SetValue(pm, "");
// 复制 EP 资产并重指
AssetDatabase.CopyAsset("Assets/.../Kipfel_ExParameters.asset", "Assets/.../Kipfel_ExParameters_沐梓溪狼.asset");
descriptor.GetType().GetField("expressionParameters", flags).SetValue(descriptor, epCopy);
EditorSceneManager.SaveScene(scene);
```

本工作区实测：克隆体与本体**组件数逐一相等**（MenuItem 157 / ObjectToggle 75 / MenuInstaller 40 / MAParameters 7 / 总组件 3022），`transformsAll=2097`。**克隆后先存盘再动刀**，便于出事回滚。

### 3.2 减法清单（猫→狼为例）

| 类别 | 删除对象 | 备注 |
|---|---|---|
| 另一颗头 | `armature/.../Head/狼头`、`WolfHead_Root*`、狼眼 | 反过来做时删猫头 `Body` 等 |
| 另一套头发/尾巴 | 猫发根、猫耳 3 件、猫名牌 | 尾巴/耳朵如两角色共用可保留 |
| 性别/其他配件 | 内裤、猫版眼罩、猫版眼镜链、翅膀光环 | 只留目标角色需要的 |
| 菜单副本 | 另一角色的整棵菜单宿主（如 `角色菜单_小狼` 18 个宿主） | 新角色的菜单会重建 |
| 换头 | `换头FX`、`菜单开关/头部` | 新角色头部常开，不再切换 |
| 预览副本 | `默认外观预览（EditorOnly）/预览_*` 里属于旧件的 | 保留下来的预览副本会与真实对象重复显示 |

然后把目标头 `SetActive(true)`、`descriptor.VisemeSkinnedMesh` 指到目标头自己的 `Body` SMR（本工作区狼头 `blendShapeCount=208`，原生猫头是 7540v 的另一个网格）。

### 3.3 控制器复制（必须）

`descriptor.baseAnimationLayers[4].animatorController` 若与本体共用同一个 FX 控制器 → **复制一份**再指过去：

```csharp
AssetDatabase.CopyAsset(catFxPath, ".../Kipfel_FXLayer_v1.1.1_沐梓溪狼.controller");
var layers = descriptor.baseAnimationLayers;              // 是 struct 数组，改完要写回
layers[4].animatorController = copied;
layers[4].isDefault = false;
descriptor.baseAnimationLayers = layers;
```

之后再「删掉只属于旧角色的层」（本工作区把猫的 `WolfHairColor` 层从猫控制器里删掉、文件 637,619→623,183 B，消除指向已删狼头的死绑定）。

---

## 4. 默认值：三处一致律（本工作区踩过两次）

「默认穿睡衣1、关帽子/外套/幽灵」「默认发蓝紫色」「亮度 55%、backlight 关」这类需求，**每一处都要同时改**：

| 层 | 位置 | 判据 |
|---|---|---|
| MA 参数 | `ModularAvatarParameters.parameters[i]`：`defaultValue` + `hasExplicitDefaultValue=true` + `saved=true` | 烘焙后 `VRCFury Params.asset` 里读到期望值 |
| 菜单项 | 对应 `ModularAvatarMenuItem.isDefault` | 烘焙后 `Menu.asset` 里该项标记正确 |
| 场景初始态 | 目标对象 `activeSelf` | 烘焙克隆里 `activeInHierarchy==false` |
| EP（头像级） | `VRCExpressionParameters` 资产 | 本工作区实测：`HeadType Int=2 / WolfHairColor Int=6 / LightController/LightLimit Float=0.55 / BacklightToggle Bool=0 / OutfitSelect Int=1 / S1_Hat\|Coat\|Ghost Bool=0` |

> 只改一处会「编辑器里看着关了、进游戏又开」。三处都改完，再用烘焙产物验证（见 §6）。

---

## 5. 菜单/宿主重指：共享 + 独立

- 两个角色都用的分类（本工作区是「衣服」衣柜 `ExMenu/衣柜.asset`）→ **两边都指同一个资产**，内容由各自头像内的 `MenuInstaller` 在烘焙时填充，不会互相污染。
- 角色独有的分类（饰品/功能道具/外观调节）→ **各自新建资产**，把 `ModularAvatarMenuInstaller.installTargetMenu` 重指过去（本工作区：`菜单开关/名牌|乳贴~|眼镜` → `狼_装扮_饰品.asset`、狼头眼罩宿主 → `狼_功能道具.asset`）。
- 三级库菜单（ABT 工具）：`installTargetMenu` 会把它挂到根菜单，若想让它出现在**指定的二级菜单**里，应删掉那个 `MenuInstaller`，改成在父菜单资产里用 `Control.type=SubMenu` + `subMenu=<ABT Menu>` 直接引用；
  若 ABT 的 Menu 只有一层「ABT」包着真正的 `MenuList`，把 `subMenu` 指到 `MenuList` 才是「功能菜单 → ABT工具 → 8 项」的正确层级。
- 菜单资产里 `controls=0` 是正常的：内容由 installer 在烘焙时注入；**判断菜单对不对必须读烘焙产物**，编辑器资产本身是空的。

---

## 6. 验证：读烘焙产物，别信 Play 里的根对象

编辑器 Play 时，场景里被 NDMF/VRCFury 处理过的头像根 **`Animator.runtimeAnimatorController == null`**（烘焙克隆是另一棵树），所以：

- ❌ 不能用 Play 截图判断「默认值/开关是否生效」（截到的是烘焙克隆的静态状态）。
- ✅ 读 `Packages/com.vrcfury.temp/Builds/____/` 下三件套：
  - `VRCFury FX.controller`：查 `MA Responsive: <对象>` 层——`<default>` 为空 motion、ON 态 clip 写 `m_IsActive=1`、AnyState 条件用对应参数（本工作区 `S1_Hat` 条件落在 `Greater 1.00 && Less 1.01`）。
  - `VRCFury Params.asset`：查 EP 默认值（Bool 0/1、Int、Float、`saved/net`）。
  - `VRCFury Menu.asset`：**逐层打印菜单树**，核对项数与顺序。
- ✅ 静态审计另算：`MenuItem / ObjectToggle / MenuInstaller / MAParameters` 组件数、OT 行数、null 行、坏路径、MA 参数重复、installer 目标不可达（本工作区最终：猫 `MI=76 OT=63 INST=18`、狼 `MI=73 OT=59 INST=12`，null 行 0、参数重复 0、不可达 0）。

---

## 7. ABT 工具绑点怎么验（本工作区任务 2.3）

ABT 的「笔/手指/头」不是靠父级，而是靠 `ModularAvatarBoneProxy`：

| 代理对象 | `boneReference` | 烘焙后 `_targetCache` |
|---|---|---|
| `ABT/Pen_Setting` | `RightHand` | `Hand.R` |
| `ABT/Finger_Setting` | `RightIndexDistal` | `IndexDistal.R`（右手食指末节 = 笔尖落点） |
| `ABT/BoneProxy/Head` | `Head` | `Head` |
| `ABT/BoneProxy/Chest` | `Chest` | `Chest` |

验证方法：① 反射读 `boneReference`（枚举名）与私有 `_targetCache`（Transform 名）；② **进 Play** 后量世界坐标对照 —— 本工作区实测 `Finger_Setting` wp=(0.1876,0.3305,0.0237)
与 `IndexDistal.R` wp=(0.1878,0.3342,0.0235) 差 3.7 mm，`Hand.R` wp=(0.1502,0.3590,0.0207) → 绑点正确。
注意：`ABT/AvatarPen` 常驻 inactive，Play 里 `SetActive(true)` 会被 ABT 自己的 Animator 每帧关回去 → **看不到笔模型是正常的**，绑点正确性只能靠 `_targetCache` + 手指世界坐标证明。

---

## 8. 发色菜单图标：用真实渲染的大头截图

需求「发色菜单的每一项用该发色下**实际渲染的头部大头截图**」的做法（本工作区实测）：

1. 临时相机：位置 = `Head 世界坐标 + (0, 0.735-0.6414, 0.95)` ≈ `(0,0.735,0.95)`，`rotation=(0,180,0)`（角色朝 +Z，相机放 +Z 侧回看），FOV 28，near 0.01 / far 10；
  `RenderTexture` 512² ARGB32 **AA4**；灯光用临时 DirectionalLight `intensity≈1.05, Euler(25,185,0)`；背景 `SolidColor(0.06,0.06,0.09,1)`。
2. 渲染时**临时隐藏**：身体 `Body_Base`、所有服装根、`耳朵尾巴`、`眼镜`、`默认外观预览（EditorOnly）`、目标头自己的头饰组；只留狼头 4 个 SMR。
3. 逐色改 4 个材质槽：`狼头/Body.m_Materials[2]`、`狼头/Cat_Ear[0]`、`狼头/Hair_Back[0]`、`狼头/Hair_Base[0]` → `M_LilLeo_Hair_<CODE>`；渲完恢复默认色（本工作区 `M_LilLeo_Hair_PW` 蓝紫）。
4. PNG 导入设置：`TextureImporterType.Default`、`mipmapEnabled=false`、`alphaIsTransparency=true`、`maxTextureSize=256`、`Uncompressed`、`sRGBTexture=true`（源 512² → 导入 256²）。
5. 把生成的 `Texture2D` 逐个赋给 `狼_发色调节.asset` 的 `Control.icon`（注意 `VRCExpressionsMenu.Control` 是 struct，改完要写回 `controls[i]` 再 `EditorUtility.SetDirty(SaveAssets)`）。

---

## 9. 本轮 6 个可复现的坑

| 坑 | 现象 | 正确做法 |
|---|---|---|
| CodeDom(C#6) 限制 | `Object.DestroyImmediate` 报 `'Object' is an ambiguous reference`；`var M = VRC....VRCExpressionsMenu;` 报 `Expression denotes a type`；局部函数/lambda 捕获同名变量报错 | 全写 `UnityEngine.Object.DestroyImmediate(...)`；类型全名写全；不写局部函数；lambda 参数不与外部变量重名 |
| `GameObject.Find` | 目标根被 SetActive(false) 后返回 null（本工作区猫根被临时关过） | 用 `SceneManager.GetActiveScene().GetRootGameObjects()` 遍历按名找 |
| `AssetDatabase.LoadAssetAtPath` | 传绝对路径 `F:\...` 静默返回 null | 一律项目相对路径 `Assets/...` |
| 裸路径字面量 | `F:/x` 在 C# 里被当除号，编译失败 `Unexpected symbol '/'` | 用 `"F:" + "\\VRChat\\..."` 拼接 |
| `List<T>` 下标/SetActive 循环 | 本环境偶发 `ArgumentOutOfRangeException` | 换固定长度数组 `GameObject[]`，行为稳定 |
| 临时相机销毁顺序 | 先 `DestroyImmediate(camGO)` 再 `cam.targetTexture=null` → `MissingReferenceException: The object of type 'Camera' has been destroyed` | **先** `cam.targetTexture=null`，**再** Destroy（相机/灯/RT 都按此顺序） |

---

## 10. 边界

- 本册流程的「视觉最终观感」只在**烘焙 Play** 与**上传后游戏内**才算真验收；编辑器 Scene 视图的 T-pose、袖子水平、饰品浮空都是未合并前的假象。
- 本工作区该角色已实际 Build & Publish 成功（`PendingUpload=False / Version=2 / UnityPackages=1 / ImageUrl` 非空），但**游戏内实际穿戴效果、多人同步、他人视角 NOT_RUN**。

---

## 11. 图标透明背景（2026-10-27 追加）

§8 用深色实底渲染出来的大头图，用户反馈「黑底不好看」，要求**透明背景**。三处改动即可，其余配方不变：

| 项 | 深色底（旧） | 透明底（正解） |
| --- | --- | --- |
| 相机清屏 | `backgroundColor=(0.06,0.06,0.09,1)` | `backgroundColor=(0,0,0,0)`（SolidColor + alpha 0） |
| RenderTexture 格式 | ARGB32（够用） | **必须 `RenderTextureFormat.ARGB32`**（`RenderTexture.GetTemporary` 的默认格式可能是 RGB24/无 alpha，渲出来 alpha 全 1） |
| 导入设置 | 任意 | `alphaIsTransparency=true`、`alphaSource=FromInput`、`sRGB=true`、`mipmap=false`、`maxTextureSize=256`、`Uncompressed` |

验收判据：渲完立刻 `tex.GetPixels32()` 读**四角** alpha，必须全 0（本工作区实测四角 alpha=0.00、不透明像素 9580/262144）；导入后再 `GetPixels32()` 抽查一次，防止导入器把 alpha 丢掉。

坑：`Texture2D` 用 `EncodeToPNG` 之前要 `Apply()`；RT 渲完 `cam.targetTexture=null` 再 Destroy（见 §9 最后一行）。

**坑（2026-10-27 实测，代价是角色「面部变透明」）**：批量换发色时**必须按槽位精确恢复**。狼头 `armature/.../Head/狼头/Body`（SMR，208 blendshapes）的材质槽是固定含义：

| 槽位 | 内容 | 参考材质 |
| --- | --- | --- |
| `[0]` | 皮肤/脸 | `make&skin`（`Assets/04_妆容·表情/Fluffy Bancho/make&skin/make&skin.mat`） |
| `[1]` | 眼睛 | `M_LilLeo_Body_Eye`（`Hidden/lilToonCutoutOutline`，queue 2450） |
| `[2]` | 头发 | `M_LilLeo_Hair_<CODE>` |

若恢复时代码把头发材质写进 `[0]`（例如 `mats[0]=hair` 或整组赋成同一个材质），编辑态与上传包里**脸会被头发材质覆盖**：贴图变成发色、lilToon outline 变体让面部看起来「透明/发蓝」，而 Console 不会报任何错。
判据 = 直接打印 `smr.sharedMaterials[i].name` 与前缀对照（本工作区曾出现 `mat[0]=M_LilLeo_Hair_PW / mat[1]=M_LilLeo_Body_Eye / mat[2]=M_LilLeo_Hair_PW`），
修法 = 从**同骨架的参照头像**（`小狼2.0/Body` 的 `sharedMaterials[0]`）取回原材质赋回 `[0]`，不要凭文件名猜。改完记得 `SaveScene` 并**重新出包上传**（材质是场景实例覆盖，不进资产，但会随包上传）。

---

## 12. 清掉克隆体上的 Missing Script（2026-10-27 追加）

克隆整棵树时，源头像上那个「既有 missing script」会被原样复制过来；VRCQuestTools 会在 Scene 视图左上角弹一条常驻提示 `There are missing scripts. Please check for missing assets or packages you forgot to import.`，截图/预览都会被它挡住。

**删除方法（实测，首选官方 API）**：

```csharp
int removed = GameObjectUtility.RemoveMonoBehavioursWithMissingScript(go);   // 一次删掉该对象上所有 missing script
```

判据：删前 `go.GetComponents<Component>()` 里有 `null`；删后无 `null`。本例 `沐梓溪喵/睡衣` 与 `沐梓溪狼/睡衣` 各 1 个（原作者遗留，不是本轮引入），删后两个头像 `missingScript=0`。

**坑（本工作区实测）**：`睡衣` 是 **prefab 实例**，用 `new SerializedObject(transform.gameObject).FindProperty("m_Component")` 倒序删空元素虽然返回成功、
`ApplyModifiedPropertiesWithoutUndo()` 也不报错，但**组件数不变**（仍是 3 个含 1 个 null）
→ 对 prefab 实例请直接用上面的 `GameObjectUtility.RemoveMonoBehavioursWithMissingScript`。（另：`Transform` 的序列化对象里没有 `m_Component`，必须传 GameObject。）



