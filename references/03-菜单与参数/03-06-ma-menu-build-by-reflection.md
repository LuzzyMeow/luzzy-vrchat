# 用反射批量建 / 改 Modular Avatar 菜单（含一键穿衣互斥的正解）

> 本工作区实测：2026-10-27（第 104 轮，给「胡萝卜衣服」建整套菜单 + 把 8 套「一键穿衣」改 Button）
> 场景：Unity 2022.3 + MA 1.11.x + `execute_code`（CodeDom C#6）；菜单项很多、手工点不现实时使用。

---

## 1. 类型与字段速查（反射必用）

| 类型 | 关键成员 |
| --- | --- |
| `nadena.dev.modular_avatar.core.ModularAvatarMenuItem` | 字段 `Control`(**struct** `VRCExpressionsMenu+Control`)、`MenuSource`(枚举)、`menuSource_otherObjectChildren`、`label`、`automaticValue`、`isDefault`、`isSynced`、`isSaved` |
| `…VRCExpressionsMenu+Control` | 字段 `name`、`icon`、`type`(ControlType)、`parameter`(**struct** `Control+Parameter`)、`value`、`style`、`subMenu`、`subParameters`、`labels` |
| `…VRCExpressionsMenu+Control+Parameter` | 字段 `name`（**注意：`Parameter` 嵌在 `Control` 里，不在 menu 类型下**） |
| `ModularAvatarMenuInstaller` | 字段 `menuToAppend`、`installTargetMenu`（都是 `VRCExpressionsMenu`） |
| `ModularAvatarParameters` | 字段 `parameters`（`List<ParameterConfig>`，struct） |
| `ModularAvatarObjectToggle` | 字段 `m_objects`（`List<ToggledObject>`，struct：`Object`(`AvatarObjectReference`)、`Active`） |
| `AvatarObjectReference` | 字段 `referencePath`（相对 avatar 根）、`targetObject` |

`ControlType` 枚举值（本 SDK 3.10.x，**实测**）：`Button=101`、`Toggle=102`、`SubMenu=103`、`TwoAxisPuppet=201`、`FourAxisPuppet=202`、`RadialPuppet=203`；`MenuSource` 用 `Children`。

> ⚠️ 早期笔记把 `Button` 写成 `0` 是错的：`0` 是非法值，MA 反应层在烘焙期 `PortableControlConversions.ToPortable` 直接抛 `ArgumentOutOfRangeException`，
> **整批 MA 响应层不生成**（表现为「一键穿衣全失效 + 默认衣装全消失」，但 Console 不报红）。写枚举一律 `Enum.Parse(fieldType, "Button")`，不要写整数。

---

## 2. 取嵌套 internal 类型的正确姿势

```csharp
Type tMenu = typeof(VRC.SDK3.Avatars.ScriptableObjects.VRCExpressionsMenu);
// ✗ 只传 Public → 返回 null → 下一行 NRE
Type tCtrl = tMenu.GetNestedType("Control", BindingFlags.Public);
// ✓
BindingFlags BF = BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance;
Type tCtrl = tMenu.GetNestedType("Control", BF);
Type tPar  = tCtrl.GetNestedType("Parameter", BF) ?? tCtrl.GetField("parameter", BF).FieldType;
```

---

## 3. 造一个 MenuItem 的 Control（struct 必须整体写回）

```csharp
object c = Activator.CreateInstance(tCtrl);
object p = Activator.CreateInstance(tPar);
tPar.GetField("name", BF).SetValue(p, "");                     // 空 = 由 MA 自动命名/继承
tCtrl.GetField("parameter", BF).SetValue(c, p);
tCtrl.GetField("value", BF).SetValue(c, value);
tCtrl.GetField("type", BF).SetValue(c, Enum.Parse(tCtrl.GetField("type", BF).FieldType, "Toggle"));
tMi.GetField("Control", BF).SetValue(mi, c);                   // ← 必须写回！
mi.isDefault = false; mi.isSynced = true; mi.isSaved = true;
EditorUtility.SetDirty(mi);                                    // 只改内存不落盘 → 重启后 subMenu={fileID:0}
```

子菜单：`type=SubMenu` + `mi.MenuSource = SubmenuSource.Children` + `mi.menuSource_otherObjectChildren = hostGO`；若指到现成菜单资产则设 `c.subMenu = menuAsset`。

---

## 4. 复制现成宿主当模板（最快）

```csharp
var src = srcHost.GetComponent<ModularAvatarMenuItem>();   // 找一个参数/结构相近的现成项
var dst = newGO.AddComponent<ModularAvatarMenuItem>();
UnityEditorInternal.ComponentUtility.CopyComponent(src);
UnityEditorInternal.ComponentUtility.PasteComponentAsNew(newGO);
```
再改 `Control.parameter.name` / `value` / `type` 与 `installTargetMenu`。
> Installer 的 `installTargetMenu` 指向分类资产（如 `Assets/01_模型本体/Kipfel/ExMenu/衣柜.asset`，其 `controls` 可能是 0 项，内容由 installer 在烘焙时填）；`menuToAppend=null` + `installTargetMenu=分类` = 「把这棵子树追加进分类」。

---

## 5. 「一键穿衣」必须且只能有一个的正解（⚠️ 历史方案：Button 互斥版；第 132 轮后现行见 §10b/§14）

**需求**：8～9 套衣服，任何时候恰好一套「一键穿衣」处于按下态；默认套装不可关闭；其余互相排斥。

**历史正解（第 132 轮后已不是现行做法）= Button + 共享 Int：**

> ⚠️ 第 132 轮现行架构：删 `OutfitRoots`、套根常驻 active、可见性由部件 bool + `OutfitSync` 全量重写；控件语义用 `Toggle(102)`。见本册 §14（尤其 §14.3/§14.4）与 [bake-and-validation §34](../07-验证与排查/07-01-bake-and-validation.md)。

- 每个一键穿衣 `Control.type = Button(101)`（真值 101；旧版本笔记误写 0，见 §10.2 真值表）、`parameter = OutfitSelect`、`value = 1..N`；
- `OutfitSelect` 在 `ModularAvatarParameters` 里声明 `Int defaultValue=<默认套装序号> explicit+saved`；
- 默认套装 `isDefault = true`（其余 false）；
- Button 在 VRChat 里**只能写值、无法关回 0** → 天然满足「必有一个」；共享同一 Int → 天然互斥。（⚠️ 该结论只适用 Button 互斥思路；现行 Toggle(102) 允许再点写 0 回落场景默认态，见 §14.4）
- 切套装的实际换装仍由驱动层（`OutfitOneClick` 那种 AnyState→Apply_k + `VRCAvatarParameterDriver`）完成，Button 只负责写 `OutfitSelect`。

> 反例：用 Toggle 会出现「再点一下就 0 → 全部回落 → 叠穿/裸装」。（⚠️ 此反例只针对旧 Button 方案；第 132 轮现行 Toggle 方案用 `OutfitSync` 每态全量重写部件 bool 来消除叠穿/裸装，见 §14.4）
> 验证（仅适用于本历史方案）：烘焙 `VRCFury Menu.asset` 里 8 项的 `type` 全是 Button，`OutfitSelect` 默认值 = 默认套装。

---

## 6. MA 自动 More 的坑

一个子菜单超过 **8 项** 时，MA 会自动包一层 `More`。若界面要求「一眼看到全部」：

- 把逻辑相关的项收进自带 SubMenu（`MenuSource=Children`，**不需要单独 Installer**，靠父 Installer 嵌套）；
- 关键项 `SetSiblingIndex(0)` 保证排第一；
- 别用「删了再建」来排序（会留下重复项，见下）。

---

## 7. 踩过的坑

| 现象 | 根因 | 处理 |
| --- | --- | --- |
| `GetNestedType("Control", Public)` 返回 null → NRE | 嵌套类型是 internal | 加 `NonPublic` |
| 菜单点了没反应 / 重启后子菜单空 | 只改内存没 `SetDirty`+`SaveAssets`（`Control` 是 struct） | 改完整条 `Control` 写回 + SetDirty + SaveAssets；验收读 .asset YAML 看 `subMenu: {fileID: 0}` |
| 出现重复项（同名两个，参数不同） | 上一轮失败脚本 `PasteComponentAsNew` 留下的残留 | 列出子项打印 name/param 再删；`DestroyImmediate` 后**不要再访问该对象**（MissingReferenceException 会中断 Save） |
| 一键穿衣点一次后全没了 | 用了 Toggle | 改 Button |
| 新参数不生效 | MA Parameters 没声明 / 没 `explicit` | 声明 `defaultValue + hasExplicitDefaultValue + saved`，且场景/菜单三处默认值一致 |

---

## 8. 验收（烘焙后读产物）

```csharp
// 递归遍历 VRCFury Menu.asset
int menus, controls, missingSub;   // missingSub 必须 0
// 打印每项：/路径/名称 [type] p=参数 v=值 i=图标
```
判据：新分支路径正确、`一键穿衣` 是 `[Button] p=OutfitSelect v=k`、部件开关是 `[Toggle] p=K_xx v=1`、`SubMenu` 的 `subMenu` 非空、参数在 `VRCFury Params.asset` 里 `def` 与设计一致。

---

## 9. 构建面板 `Review Any Alerts (66)`：菜单参数未定义告警的成因、修复与验证

### 9.1 症状

点 `Build & Publish` 后 SDK 面板第 2 段标题变成 `Review Any Alerts (66)`，逐条都是：

```
VRCExpressionsMenu uses a parameter that is not defined.  Parameter: ABT/Cl/Back_Empty
```

参数名成批出现（本项目实测：`ABT/Cl/*`、`CC2/Locomotion/*` 等），而且**烘焙后的头像里这些参数明明存在**（Play 烘焙产物 EP 里有、FX 里也正常驱动），所以很容易误判成「SDK 抽风」。

### 9.2 根因（关键认知）

**SDK 的预检读的是「未烘焙的源资产」，不是烘焙产物：**

| SDK 预检读什么 | 它不读什么 |
| --- | --- |
| `VRCAvatarDescriptor.expressionParameters`（源 EP 资产） | NDMF/MA/VRCFury 烘焙后的临时 EP |
| `VRCAvatarDescriptor.expressionsMenu` 递归出的整棵菜单树 | 烘焙注入的菜单项 |
| 每个 `Control.parameter.name` **以及** `Control.subParameters[*].name` | MA Parameters 里声明的参数名 |

而 Modular Avatar / VRCFury 是在**烘焙期**才把参数注入 EP 的（`ModularAvatarParameters` + MA 的自动参数创建，见官方文档 Menu Item：
「When you specify a parameter name that is not declared in a MA Parameters component or VRC Expressions Parameter asset,
the Menu Item will create a parameter…」）。于是凡是「菜单资产里写了名字、源 EP 里没声明」的参数，SDK 一律报未定义。

本项目实测数据（可对照）：

- 源 EP `Kipfel_ExParameters_沐梓溪狼`：**16** 个参数
- 源菜单树 `狼_根` 递归：**87** 个 Control，主参数 **34** 个去重名，`subParameters` 另 **10** 个
- 未声明：**33** 个去重名 / **51** 处引用 → 面板 66 条（计数按「条目/子参数」逐条累计，不是按去重名）
- 烘焙产物（对照）：EP **153** 参数、菜单 192 控件、`undefined=0`、`typeMismatch=0` → 说明**功能本身没错，错的只是源 EP 声明不全**

### 9.3 修复（本工作区采用，安全且 bake 中性）

把缺失的**每一个名字**补进源 `VRCExpressionParameters`：

```csharp
var ep = descriptor.expressionParameters;
var declared = new HashSet<string>(ep.parameters.Select(p => p.name));
// 递归收集：Control.parameter.name + Control.subParameters[*].name
foreach (var miss in used.Where(n => !declared.Contains(n)))
{
    var p = new VRCExpressionParameters.Parameter {   // 注意：Parameter 是 class
        name = miss,
        valueType = mapType(miss),       // Int=0 / Float=1 / Bool=2，照烘焙产物里的类型填
        saved = shouldSave(miss),
        networkSynced = true,
        defaultValue = 0f,
    };
    // 追加进 ep.parameters（先扩容），EditorUtility.SetDirty(ep); AssetDatabase.SaveAssets();
}
```

要点：

1. 先备份源 EP（`.asset` 复制一份到 `_asset_reorg/backup/`）；
2. 类型/`saved` 照**烘焙产物 EP** 里的同名参数抄，别猜；
3. 补参数是 **bake 中性**：实测补完后重新烘焙，菜单树、FX 层数、参数集合与补之前一致 —— 它只影响 SDK 预检，不影响功能；
4. 不要为了消警告去「删菜单项」或「改参数名」，那会真的把功能删掉；
5. 补完后重开面板 → 面板要**重新跑一次校验**才刷新计数（关掉窗口再 `VRChat SDK/Show Control Panel`，或在 Build 页点一次刷新）。

### 9.4 另一种坑：SDK 已知误报（残留序列化数据）

VRChat 官方 bug 单（SDK 3.10.1，已 tracked）：「Meaningless …not defined… warning」。复现：
把某个菜单项从 `RadialPuppet/TwoAxisPuppet/FourAxisPuppet` 改成 `Button/Toggle/SubMenu`，再把这个参数从 EP 删掉 ——
控件上会**残留**旧的 `parameter` / `subMenu` / `subParameters` 序列化数据，于是「菜单里实际没用到这个参数」也照样报警。

判据与处理：Inspector 右上角菜单 → `Debug` 模式看该 Control，若 `parameter.name` / `subParameters` 里还留着没用的名字，清空即可（反射做法：
`control.subParameters = new VRCExpressionsMenu.Control.Parameter[0];`、必要时 `subMenu = null`，然后 SetDirty）。本项目在 EP 补全后再扫描，
`missingMain=0 / missingSub=0`，面板 `not defined` 命中 0，排除该类误报。

### 9.5 验证（两条独立证据，缺一不可）

```csharp
// A. 反射读面板（VRCSdkControlPanel 在 VRC.SDK3A.Editor / VRCSDKBase 程序集里按类名找）
var tPanel = FindType("VRCSdkControlPanel");
var win = tPanel.GetField("window", BF_Static).GetValue(null);            // 面板开着才有值
var ve  = tPanel.GetField("_validationsContainer", BF).GetValue(win) as VisualElement;
// 递归 Children()，统计 Label.text 里含 "not defined" / "VRCExpressionsMenu" 的条数 → 必须 0
var mAlert = tPanel.GetMethod("GUIAlertCount", BF);                        // 注意：实例方法！
mAlert.Invoke(win, new object[]{ descriptorGameObject });                  // → 0 = 无阻塞性 error
mAlert.Invoke(win, new object[]{ avatarDescriptor });                      // → 0
```

- `GUIAlertCount` 是**实例方法**：`Invoke(null, …)` 会抛 `Non-static method requires a target`，必须把 `win` 当 target；
- 「(N)」里的 N 与 `GUIAlertCount` 不等价：面板标题的 N = 该段列出的所有条目（含性能/环境类 Warning），`GUIAlertCount` 返回的是**告警计数缓存**，重开面板/重跑校验后才刷新；
- B. Windows 实机截图：`user32.ShowWindow(9)` + `SetForegroundWindow` + `GetWindowRect` + `Graphics.CopyFromScreen` 存 PNG → 裁右侧面板 → PIL 缩到 ≤420 px 再读图，
  肉眼确认标题数字与条目内容（本工作区证据：`_asset_reorg/evidence/round108/panel_alerts.png`、`alerts_zoom.png`）。

### 9.6 本项目结果

| 项目 | 修前 | 修后 |
| --- | --- | --- |
| 面板标题 | `Review Any Alerts (66)`，前几条全是参数未定义 | `Review Any Alerts (25)`，全部是性能/环境 Warning（三角形、骨骼、材质槽、PhysBone、粒子…） |
| 参数未定义条目 | 43 条（`GUIErrors` 43 个控件） | **0** |
| `GUIAlertCount(GameObject)` / `(Descriptor)` | 43 / 25 | **0 / 0** |
| 源 EP 参数数 | 16 | 59 |

剩下的 25 条是**性能告警**（`VeryPoor`、Triangles 259896、Bones 954、Material Slots 149…），SDK 允许带 Warning 上传，不会阻塞；要压下去只能真做优化（合并网格/材质、减 PhysBone），不属于「报错」。

### 9.7 参考

- VRChat 官方 Expressions Menu / Parameters 文档：<https://creators.vrchat.com/avatars/expression-menu-and-controls/>
- SDK 3.10.1 误报 bug 单（残留序列化数据）：<https://feedback.vrchat.com/sdk-bug-reports/p/sdk3101-meaningless-vrcexpressionsmenu-uses-a-parameter-that-is-not-defined-warn>
- 旧版同类（子菜单残留引用）：<https://feedback.vrchat.com/sdk-bug-reports/p/vrcsdk3-avatar-202107121852-expression-menu-may-still-hold-references-to-sub-men>
- Modular Avatar Menu Item 官方说明（未声明的参数会在烘焙期自动创建）：<https://modular-avatar.nadena.dev/docs/reference/menu-item>
- 社区整理条目（错误释义）：<https://notes.sleightly.dev/VRCExpressionsMenu-uses-a-parameter-that-is-not-defined-Parameter-4413e0b062b94bd689f7ce5b5e8bd6df>


## §10 `Control.ControlType` 的真实取值是 101/102/103/201/202/203（写成 1 会让 MA 烘焙整批失败）

### 10.1 症状

烘焙「看起来成功」（VRCFury/NDMF 控制台没有红色阻塞框），但生成的 FX 层数**骤降一半以上**、`MA Responsive:` / `OutfitOneClick` / `Body_Base_Fit` 之类反应层全部缺失；菜单树也只剩少数几项。用户侧表现 = 一键穿衣点了没反应 / 部分菜单消失 / 换体形不生效。

### 10.2 真值表（VRC SDK 3.x，实测 `Enum.GetValues`）

| 名称 | 值 |
| --- | --- |
| Button | **101** |
| Toggle | **102** |
| SubMenu | **103** |
| TwoAxisPuppet | **201** |
| FourAxisPuppet | **202** |
| RadialPuppet | **203** |

**不是** 0/1/2/3/4/5。旧文档/旧代码里常见的 `Button=0, Toggle=1, SubMenu=2` 是**另一套已废弃的 int 常量**；写 1 会得到一个「未定义枚举值」。

### 10.3 后果链

`ModularAvatarMenuItem.Control.type` 是 `VRCExpressionsMenu+Control+ControlType`（struct 字段）。写入非法值后，烘焙期 MA 反应层生成时：

```
ArgumentOutOfRangeException: Parameter name: type. Actual value was 1.
  at nadena.dev.modular_avatar.core.PortableControlConversions.ToPortable(...)
  at nadena.dev.modular_avatar.core.PortableMenuControl.get_Type()
  at ...ReactiveObjectAnalyzer.BuildConditions(...)
```

异常被 NDMF 记为 `InternalError` → **MA 的整个 reaction/menu pass 中止**，于是：菜单项不生成、`MA Responsive:` 层不生成、ObjectToggle 不生效、一键穿衣/体形切换/尾巴显隐全失效。而 VRCFury / AAO 自己的层照旧生成 → 烘焙「看起来成功」。

### 10.4 正确写法（反射）

```csharp
var tCtl  = typeof(VRC.SDK3.Avatars.ScriptableObjects.VRCExpressionsMenu.Control);
var fType = tCtl.GetField("type", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
var fCtl  = miType.GetField("Control", BindingFlags.Instance | BindingFlags.Public | BindingFlags.NonPublic);
object ctl = fCtl.GetValue(mi);
fType.SetValue(ctl, System.Enum.Parse(fType.FieldType, "Toggle"));   // 务必按名字 Parse，不要写整数
fCtl.SetValue(mi, ctl);                                              // struct：必须写回
EditorUtility.SetDirty(mi);
```

### 10.5 验收判据（本工作区实测）

| 判据 | 修前（type=1） | 修后（Toggle=102） |
| --- | --- | --- |
| 烘焙 FX 层数 | **62** | **130 ~ 135** |
| `OutfitOneClick` 层 | 缺失 | 10 态（Idle + Apply_1..8 + Apply_0） |
| `MA Responsive: Body_Base` / `Body_Base_Fit` | 缺失 | 各 2 层、转移条件精确 `OutfitSelect ∈ (8.995, 9.005)` |
| 烘焙菜单 一键穿衣 9 项 | `type=1`（非法） | `type=102`、`OutfitSelect` 1..9 |
| Editor.log 切片 | `ArgumentOutOfRangeException` ×1 | **0** |

判定技巧：改完直接数 `bakedFX.layers.Length` 与 `MA Responsive:` 前缀层数；层数掉一半基本就是 MA pass 挂了。

### 10.6 附带坑

- `nadena.dev.modular_avatar.core` **命名空间带下划线**（`modular_avatar`），按 `nadena.dev.modular-avatar.core.*` 取类型永远是 null → 遍历 AppDomain 按 `Type.Name` 找。
- MA 的亮度插件（NyaPPu LightController）烘焙要写 `Assets/08_动画·菜单·图标/NyaPPu__Generated/LightController/` 下的 anim；该目录被清理掉后烘焙报 `UnityException: Creating asset at path ... failed`，补齐目录即可（属于生成目录，不要当垃圾删）。

---

## §11 菜单「多套一层」与标签失效：MenuSource=Children 时标签取 GameObject 名

**症状**：明明把 MA 宿主的 `Control.name` 设成「睡衣1」，烘焙后菜单仍是
`衣服 › EX Manu_OyasumiGhost › …`（多一层、名字还是老 GO 名）。

**根因**：`ModularAvatarMenuItem.MenuSource = Children` 时，菜单项的文字**由承载组件的
GameObject 名生成**，`Control.name` / `label` 字段被忽略（它们只在 `MenuSource = Control`
或 `Automatic` 的部分场景生效）。

**正解**：改标签 = **重命名 GameObject**，不是改 `Control.name`。

```csharp
// 正确：改 GO 名
go.name = "睡衣1";
EditorUtility.SetDirty(go);

// 错误：只改 Control.name（烘焙后无效）
var ctrl = fCtrl.GetValue(mi); fName.SetValue(ctrl, "睡衣1"); fCtrl.SetValue(mi, ctrl);
```

### 11.1 去掉多余的一层（把第三方菜单直接挂到目标一级菜单）

原生/第三方 prefab 自带的菜单宿主几乎都是 `SubMenu`（如 `EX Manu_OyasumiGhost`、
`clothes7 Toggles`、`LilLeo Toggles`），如果再把它们 Installer 到一个「中间菜单资产」，
就会变成 `衣服 › 睡衣1(空壳) › EX Manu_OyasumiGhost › 6 项` 三层。

**做法（三步，全部要保存）**：

1. 把宿主的 `ModularAvatarMenuInstaller.installTargetMenu` **直接指向目标一级菜单资产**
   （如 `狼_衣服`），并重命名 GO 为想要的标签；
2. 删掉中间菜单资产在父菜单里的空 `Control`（按 `subMenu` 引用匹配删除）；
3. `AssetDatabase.SaveAssets()` + `MarkSceneDirty` + `SaveScene`。

```csharp
static void RemoveSubMenuControl(VRCExpressionsMenu menu, VRCExpressionsMenu sub) {
    var l = menu.controls;
    for (int i = l.Count - 1; i >= 0; i--) if (l[i].subMenu == sub) l.RemoveAt(i);
    EditorUtility.SetDirty(menu);
}
```

### 11.2 一键把一组开关收进新子菜单（避免 MA 自动 `More`）

MA 在**同级超过 8 项**时自动生成 `More`。若一组件（如胡萝卜衣服 10 件）超过 8：

- 新建一个 GO（`MI SubMenu` + `MenuSource=Children` + Installer 指父菜单），
  把要分组的开关 GO `SetParent` 进去，并 **`DestroyImmediate` 掉它们各自的
  `ModularAvatarMenuInstaller`**（否则既进组又单独挂一次 → 菜单里出现两份）；
- 结果：`胡萝卜衣服 › {背带裤, 鞋子, 袜子, 手套, 小饰品(6)}` 共 5 项，不再触发 `More`。

### 11.3 烘焙验收（必须做）

`AvatarProcessor.ProcessAvatarUI` 出 clone 后递归打印菜单树，逐项核对
`/路径/名称 [type] p=参数 v=值`；判据：层级数与设计一致、无 `More`、无重复项、
`Control.value` 与 `Control.parameter.name` 都正确（**值在 `Control.value`，
`Control.parameter` 里只有 `name`**）。

## §10b 一键穿衣（无 Driver 版）：容器 + 默认开启件 + 反向行

**背景**：CodeDom 环境里 `AnimatorState.AddStateMachineBehaviour` / 泛型 `AddStateMachineBehaviour<T>()`
都会返回 `null` 并只留下空槽位，无法生成 `VRC_AvatarParameterDriver` 层；因此「一键穿衣」不要依赖参数驱动器。

**正解（纯 MA，实测通过）**
1. 每套衣服一个**容器对象**（父节点），里面挂该套全部部件；
2. 容器显隐交给**共享 Int 参数** `OutfitSelect` 的 MA 响应层（MenuItem Toggle/Button 写 1..N，容器各有一层 `MA Responsive: <容器>`）；
3. **默认穿着的那套**（睡衣）部件在场景里 `activeSelf=true`，其单件 MenuItem 的 `ModularAvatarObjectToggle` 行写 **`Active=false`**（不是 `Inverted=true`！）；
   - 参数 0 → 该规则不生效 → 回落场景默认「显示」；
   - 参数 1 → 规则生效 → 写入 `m_IsActive=0` → 隐藏；
4. **非默认件**（帽子/外套/小幽灵、胡萝卜衣的 6 个饰品——⚠️ 胡萝卜套装已于第 132 轮整组删除，此处仅存档）场景 `activeSelf=false` + 行 `Active=true`（参数 1 才显示）；
5. 非默认套装容器场景 `activeSelf=false`，靠 `OutfitSelect` 响应层打开；EP 里 `OutfitSelect` 默认值设 1（睡衣）。

**★ `ModularAvatarObjectToggle.Inverted` 的真实语义（易踩）**：它不是「把目标 active 值取反」，而是把**规则条件取反**
（`ReactionRule.InitiallyActive = (无条件 || 全部条件初始成立) ^ Inverted`，见 `Packages/nadena.dev.modular-avatar/Editor/ReactiveObjects/AnimationGeneration/ReactionRule.cs:30`）。
所以「默认显示、参数 1 隐藏」用 `Inverted=true` 得到的是：参数 1 → 规则失效 → 回落场景默认（仍然显示）→ 永远关不掉。
**正确做法是行级 `Active=false`，`Inverted` 保持 false。**

**验证（Play 模式实测；以下胡萝卜相关断言为历史记录，该套装已于第 132 轮整组删除）**
- `OutfitSelect=1` → 睡衣容器 on、胡萝卜容器 off；`PJ_Shirt/Pants/Slipper` 显示，`PJ_Hat/Outer/PocketGhost` 隐藏；
- （历史）`OutfitSelect=2` → 胡萝卜容器 on、睡衣 off；背带裤/鞋子/袜子/手套显示，创可贴等 6 件隐藏；⚠️ 胡萝卜套装已于第 132 轮删除，此项仅供理解旧架构；
- `K_PJ_Shirt=1` → 上衣隐藏；`K_PJ_Hat=1` → 帽子显示；（历史）`K_Carrot_0=1` → 背带裤隐藏。


## §11b 菜单图标还原为 VRChat 默认图标（清空 ModularAvatarMenuItem.Control.icon）

### 11b.1 症状
用户要求「所有菜单图标还原成 VRChat 默认图标，只保留发色图标」。检查时用 `Control.icon != null` 判空会**全部命中"有图标"**，但实际游戏里显示的是默认图标——因为那些 icon 字段指向的是**已被删除的贴图资产**（missing asset）。

### 11b.2 判空的两层
```csharp
var icon = ctl.GetField("icon").GetValue(ctl);       // System.Object，永远 != null（只要字段曾赋过值）
bool real = (icon as UnityEngine.Object) != null;    // 真正的判据
```
Unity 的 fake-null：`UnityEngine.Object` 被销毁后其 `System.Object` 引用仍非空，`== null` 运算符重载为 true，但装箱成 `object` 后 `!= null` 为 true。**必须 `as UnityEngine.Object` 再判**。

### 11b.3 清空做法（struct 必须写回）
`VRCExpressionsMenu.Control` 是 **struct**，`GetField("icon").SetValue(ctl, null)` 只改箱子里的副本 → 必须
```csharp
fControl.SetValue(miComponent, ctl);   // 写回组件
EditorUtility.SetDirty(miComponent);
```
菜单资产里的控件是 class 字段，直接 `c.icon = null; EditorUtility.SetDirty(menu);` 即可。

### 11b.4 结果与保留项
- 35 个 `ModularAvatarMenuItem` 的 icon 全部清空；
- 14 个菜单资产里只有 `狼_外观_发色.asset`（8 个发色）保留实渲图标（`PW/BLY/BY/GY/PA/PP/RO/YG`），其余 0 个；
- 清完 `AssetDatabase.SaveAssets()` + `SaveOpenScenes()`，重新 build 后游戏内即为默认图标。

## §12 菜单层级去重：MenuInstaller「多套一层」的两种来源（2026-10-29 本工作区实测）

### 12.1 症状

烘焙后的菜单出现「菜单套菜单」，用户点一项要按 3–4 次：

- `功能 > ABT工具 > ABT > 8 大类`
- `功能 > SPS > SPS > 上半身 / 下半身`
- `外观 > 衣服 > 睡衣 > 睡衣 > 7 项`（泳装同样）

### 12.2 根因 A：`menuToAppend` 追加的是「该资产的 controls」，不是「把它作为子菜单」

`Packages/nadena.dev.modular-avatar/Editor/Menu/VirtualMenu.cs` 的 `PushNode(ModularAvatarMenuInstaller)`：

1. `installer.GetComponent<MenuSource>() != null`（MenuItem 派生）→ `PushNode(menuSourceComp)`：把**这个 GameObject 自己的 MenuItem** 当作一项装进去 —— 如果它是 SubMenu，就必然多一层。
2. 否则 `PushMenuContents(installer.menuToAppend)`：把该菜单资产里的 `controls` **平铺**进目标菜单。

第三方资产的菜单常是「外层壳 + 内层菜单」两个资产（官方 Menu Installer 文档原文：*if you want a submenu of your own, you will need to create two menu assets: One for the submenu control, and one for the inner menu itself*）：

| 工具 | 外层壳（installer 常见填法） | 内层（应填） |
| --- | --- | --- |
| ABT（AvatarBasicTool） | `Expression/Menu.asset` → 只有 1 项 `ABT → MenuList.asset` | `Expression/SubMenu/MenuList.asset`（8 类） |
| SPS（RimoShop Quick settings） | `Data/Menu/Holes/SPS_MM.asset` → 只有 1 项 `SPS → SPS_M.asset` | `Data/Menu/Holes/SPS_M.asset`（上/下半身） |

把 `menuToAppend` 改指内层资产即可去掉中间那一层。

### 12.3 根因 B：installer 与 MenuItem 同 GO，且 target 指向「父菜单里已挂的空壳资产」

睡衣 / 泳装是「GO 上 `ModularAvatarMenuItem(SubMenu, MenuSource=Children)` + `ModularAvatarMenuInstaller`」；父菜单 `沐梓溪狼_菜单_衣服` 里又有一条 `睡衣 → 沐梓溪狼_菜单_睡衣（空壳，controls=0）` 的 control。

installer 把 GO 自己的 SubMenu 装进 `菜单_睡衣` → 结果 `衣服 > 睡衣(空壳) > 睡衣(GO) > 7 项`。

**修法**（与 §11.1 同一思路）：

1. `installTargetMenu` 改指**父菜单**（衣服）；
2. 删掉父菜单里指向空壳资产的那条 control；
3. GO 名就是标签（MenuSource=Children 时 `Control.name` 无效）。

```csharp
inst.GetType().GetField("menuToAppend", flags).SetValue(inst, innerMenu);          // 根因 A
inst.GetType().GetField("installTargetMenu", flags).SetValue(inst, parentMenu);    // 根因 B
var cs = parent.controls;
for (int i = cs.Count - 1; i >= 0; i--)
    if (cs[i].type == VRCExpressionsMenu.Control.ControlType.SubMenu
        && (cs[i].name == "睡衣" || cs[i].name == "泳装")) cs.RemoveAt(i);
parent.controls = cs;
EditorUtility.SetDirty(parent); AssetDatabase.SaveAssets();
```

### 12.4 判据（烘焙后）

`AvatarProcessor.ManualProcessAvatar` 出 clone → 递归打印菜单路径：

- `功能 > ABT工具` 直接 8 项；`功能 > SPS` 直接 2 项；
- `外观 > 衣服 > 睡衣` 7 项、`泳装` 6 项；
- 全树路径 `sort | uniq -d` 为空；
- 本项目实测控件 113 → 109（去掉 ABT / SPS / 睡衣 / 泳装 四个多余层）。

### 12.5 附带：参数驱动在 GM Play 里的 settle 时间

- 烘焙后 Bool 参数会被 MA 转成 `ModularAvatar$synthetic$bool$<hash>` 中转（Copy → Set → Copy）。直接按参数名读 driver 的 `value` 全是 0，要看 **synthetic 的 Set 条目**。
- 编辑器 Play 里驱动写入 GM 参数对象有延迟：状态切换后 **10 帧仍可能读到旧值，20 帧稳定**（本项目实测 `OutfitOneClick=0` 回 Default 时 `PJ_Hat`：10 帧=1、20 帧=0）。验证驱动层时至少等 20 帧。


## §13 第 127 轮：一键穿衣「6 态 + 默认开启」的三处默认值与结构体写回坑

### 13.1 需求

默认穿睡衣（第 1 项一键穿衣在菜单里是**开启**状态），且 6 套衣服互斥；睡衣一键穿衣**不能**开帽子。

### 13.2 默认值必须三处一致（少一处 GM/游戏就不对）

| 位置 | 字段 | 值 |
|---|---|---|
| `ModularAvatarParameters`（参数设置） | `ParameterConfig.defaultValue` + `hasExplicitDefaultValue=true` | 1 |
| `VRCAvatarDescriptor.expressionParameters` | `ExpressionParameter.defaultValue` | 1 |
| `AnimatorController.parameters`（FX） | `AnimatorControllerParameter.defaultInt/defaultFloat` | 1 |
| 睡衣的一键穿衣 `ModularAvatarMenuItem` | `isDefault` | true（其余 5 项 false） |

**GM Play 的初值读的是 AnimatorController 参数默认值，不是 EP**（本轮实测：EP=1、MA=1 但 controller=0 → `ONECLICK|initial value=0`；把 controller 也改 1 后才 PASS）。VRChat 客户端以 EP 为准，但为了编辑器和 GM 一致，三处都写。

### 13.3 三个「改完不生效」的 .NET 装箱坑（本轮全踩）

`ParameterConfig`、`ExpressionParameter`、`AnimatorControllerParameter` 都是 **struct**：

```csharp
// ❌ foreach 拿到的是副本，改字段不会回写
foreach (var p in list) { if (p.name == "OutfitOneClick") p.defaultValue = 1; }

// ✅ 按索引取出 → 改 → 写回
for (int i = 0; i < list.Count; i++) {
    var p = list[i];
    if ((string)p.GetType().GetField("nameOrPrefix").GetValue(p) == "OutfitOneClick") {
        p.GetType().GetField("defaultValue").SetValue(p, 1f);
        list[i] = p;               // 关键
    }
}
// EP：parameters[i] = pp;  controller：var ps=ctrl.parameters; ps[i].defaultInt=1; ctrl.parameters=ps;
```

判据：改完后立刻重新读取 `defaultValue/defaultInt`，必须为 1；再烘焙出的 clone EP/FX 也必须是 1。

### 13.4 睡衣帽子

`OutfitSync/Pajama` 的 `VRCAvatarParameterDriver` 里 `PJ_Hat` 必须是 **0**（第 126 轮误写 1）。
同时 `Roots_Def` / `Roots_Pajama` 两态的 `睡衣` 根都写 1，保证 `OutfitOneClick=0`（用户手动关掉一键穿衣）时仍回落到睡衣而不是裸体。

### 13.5 6 态结构

- `OutfitRoots` 6 态：Default/Pajama/Swim/Carrot/Original/Swim2，每态 clip 5 条根曲线（睡衣/泳装/胡萝卜衣服/原版衣服/Riptide）；
- `OutfitSync` 6 态，每态 **45 条 driver**（PJ 6 + SW 5 + K_Carrot 10 + Orig 3 + RT 21）；
- AnyState→各态 `OutfitOneClick Equals 0..5`，duration=0；
- 菜单：`/外观/衣服/泳装2` = 一键穿衣(v=5) + 衣服(5) + 饰品(8) + 鲨鱼尾 + 水枪 + 冰棒，共 6 项（147 controls 总）。

### 13.6 GM 判据（本轮实测）

- `ONECLICK|initial`：value=1、PJ_Hat=0、PJ_Shirt/Pants=1 → PASS；
- `ONECLICK|pajama`：Hat=0、Shirt/Pants=1、Outer/Ghost/Slipper=0 → PASS；
- `ONECLICK|swim2`：8 个 RT Bool=1、PJ/SW/Orig=0、`Riptide.activeInHierarchy=True` → PASS；
- 回到睡衣：`Riptide.activeInHierarchy=False`、PJ_Hat=0 → PASS；
- 总菜单 147 项：`pass=150 fail=6`（6 条全是 ABT 世界功能）/ `noparam=1`。

---

## §14 第 132 轮：一键穿衣互斥架构重构（删 OutfitRoots）+ 衣装选择持久化 + Button/Toggle 语义（本工作区实测 2026-10-31）

### 14.1 症状：开了某套衣装后，其他套装的部件再也单独开不了

用户原话：「开启某一套衣装后 其他的衣装的部件都无法单独开关」。

根因（**不是菜单，是 MA 规则层**）：

1. 旧实现有一层手写 FX 层 `OutfitRoots`，6 个状态各写 5 条曲线，把**非当前套装的根 GameObject** 的 `m_IsActive` 写成 0；
2. `ModularAvatarObjectToggle` 生成的每条反应规则，除了「最近 MenuItem 参数条件」外，还会对**从目标对象到头像根的每个祖先 GameObject** 各加一条条件：
   `__ActiveSelfProxy/<instanceID>`，`InitialValue` = 烘焙时场景里的 activeSelf，`ParameterValueLo = 0.5`；
3. 于是「睡衣衬衫」的 OT 规则实际是 `PJ_Shirt ≈ 1 && 睡衣根 > 0.5`；在泳装2 状态下 `睡衣根 = 0` → 规则永不成立 → 参数点了也不可见；
4. 更要命的第二层：`AnalyzeConstants` 会删掉「条件恒真/恒假」的规则组；如果根在烘焙时是 inactive 且没有任何 clip 写它，整条规则直接消失。

源码位置：`Packages/nadena.dev.modular-avatar/Editor/ReactiveObjects/AnimationGeneration/ReactiveObjectAnalyzer.LocateReactions.cs` 的 `BuildConditions`（祖先 
proxy）、`ReactiveObjectAnalyzer.cs` 的 `AnalyzeConstants`。

### 14.2 正解：根常驻 active，可见性只交给「部件 bool」

```text
删掉 OutfitRoots 层（或清空 6 个 Roots_*.anim 的 5 条 m_IsActive 曲线）
5 个套装根（睡衣/泳装/原版衣服/Riptide）场景 activeSelf 全部 = true，并保持
可见性只由 OutfitSync 的 45+ 条 VRCAvatarParameterDriver（1=可见）驱动每件部件的 bool
一键穿衣 = 共享 Int OutfitOneClick + AnyState 6 态（0..5）
```

要点：

- 一个 `m_IsActive` 只允许**一个写者**；但冲突解决**有确定规则**（2026-11-01 按 MA 官方文档改正，见 `04-外观与表情/04-06-ma-reactive-components.md` §1.6）：
  - **多个 MA 响应式组件抢同一对象**：**层级最低者胜**（Object Toggle 页写作「层级顺序最后出现者优先」）；全部不激活时回落到场景原状态，或其它动画写入的状态。
  - **多个 MergeAnimator 之间**：按 `Layer Priority` **升序**（数字小在前、大者覆盖），同优先级按层级顺序，**既有 animator 视为 priority 0 且排最前**；`Replace Existing Animator` 保留自己设的 priority，但在该 priority 内排其它 MergeAnimator 之前。
  - 官方只承诺「MergeAnimator 之间」的相对顺序；MA 生成层与**手写层 / 其它插件生成层**的最终相对位置还受构建 pass 影响，**一律以烘焙产物实际层序为准**。
  - 出现「谁压谁」先打印烘焙产物层序 + priority 再判断，别当随机行为。
- 删除「一键穿衣」的根互斥后，跨套装同时穿（睡衣衬衫 + 泳装2 饰品）是**预期行为**，因为每件都有自己的 bool。
- 烘焙验收：烘焙产物里写 `m_IsActive` 的曲线路径**不含任何套装根**（本工程改为只写部件），FX 层数下降 1。

### 14.3 持久化：MenuItem.isSaved 是假的，只能改源 EP + MA Parameters

| 位置 | 字段 | 作用 |
|---|---|---|
| 源 `VRCExpressionParameters` | `Parameter.saved=true` | 唯一真正决定「跨世界/重登保存」的地方 |
| 场景 `ModularAvatarParameters` | `ParameterConfig.saved` / `defaultValue` / `hasExplicitDefaultValue` | 把参数声明/默认值带进烘焙产物 |
| `ModularAvatarMenuItem` | `isSaved` / `isDefault` | **isSaved 对已存在于 EP 的参数无效**（MA 只在参数未被 EP/MA 声明时才用）；isDefault 只在菜单项作为默认值时生效 |

判据：烘焙克隆的 `descriptor.expressionParameters.parameters[i].saved == true`；GM/Play **无法验证** saved，只能真机切世界/重登。

本工程设置（第 132 轮）：

- saved=true：`OutfitOneClick`（Int, default 1）、`Acc_Glasses`（default 1）、`Acc_NamePlate`（default 1）；
- saved=false：所有派生部件 bool（PJ_*/SW_*/Orig_*）——它们由 OutfitSync driver 每次进态全量重写，开 saved 只会造成「半裸」状态错乱；
- `Tail`、`RT/WaterGun`、`RT/WaterGun_Hold` 等作者自带 saved 保持原样，但要把 `RT/WaterGun_Hold` 从 OutfitSync 六态 driver 里删掉，否则换套装会把手持状态重置。
- ⚠️ 与 `02-工程组织与工作流/02-02-ma-workflow-habits.md` §25.1 的「OutfitSelect + 47 个部件 Bool 全部 saved=true」**冲突**：那是第 78 轮旧结论，已被第 132 轮推翻（部件 bool 开 saved 会造成「半裸」）；以本节 §14.3/§14.7 为准。

### 14.4 控件语义：Button 会「1 秒后自动复位」，Toggle 不会

VRChat 官方文档（Expression Menu and Controls）：

- **Button**：`Sets a parameter when clicked, then resets after the sync/reset has been sent — usually after about a second. Cannot be held down.`
- **Toggle**：选中写 value，关闭写 0（对 Int 参数就是标准单选：再点一下写 0）。
- **Sub-Menu**：进入写 parameter，退出清零。
- **Puppet**：退出保留。

本工程两个实际后果：

1. 「水枪 / 右手 / 左手」三项曾被改成 Button → 点完约 1 秒 `RT/WaterGun_Hold` 自动回 0，枪回到腰部；用户看到的是「左手/右手无法保持」。**修复=改回 Toggle（type=102），value 仍 0/1/2，0 值项当复位键**。
2. 「一键穿衣」5 项也应保持 Toggle（选中写 1..5，再点写 0 回落 Default），不要改 Button。

坑：GM/Play **不模拟 VRChat 的 Button 自动复位**，所以 Button 版在 GM 里全绿、进游戏就回跳。历史「GM 全绿 = 没问题」的结论对 Button 类控件不成立。

### 14.5 删除一整套衣装时要同步清的 6 处

以第 132 轮删「胡萝卜衣服」为例，漏任何一处都会留下死数据/异常态：

1. 场景套装根 GameObject（`沐梓溪狼/胡萝卜衣服`）；
2. 菜单子树（`菜单开关/胡萝卜衣服`，DestroyImmediate 整组）；
3. `ModularAvatarParameters` 里的 `K_Carrot_0..9`；
4. FX `OutfitSync` 6 个状态的 driver 条目（每态 10 条 = 共 60 条）；
5. 一键穿衣的「该套装状态」：不要删状态（历史 saved 值可能是 3），改为**把该态 driver 改写成 Default 的取值**，AnyState `Equals 3` 保留；
6. 烘焙后审计：`carrotPaths=0`、`carrotItems=0`、参数表无 `K_Carrot*`。

### 14.6 验收（本轮实测）

- `FX|layers=11`（删 OutfitRoots 后）；`OutfitSync` 6 态每态 `entries=35`；
- 菜单 `gunToggles=3 / gunBtns=0`；烘焙菜单 138 项、`carrotPaths=0`；
- GM：`DEFAULT|glasses=VIS|nameplate=VIS`、`CROSS|pajama_shirt_in_swim2=VIS`、`CROSS|own_part_off_in_swim2=HID`、`GUN|persist_left PASS`。

### 14.7 「泳装2 一键穿衣只开饰品、不穿衣服」的正解（实测）

不要删衣服网格，也不要给 Swim2 单独建根；只让 `OutfitSync/Swim2` 态的 35 条 driver 把衣服类 bool 写 0：

| 类别 | 参数 | Swim2 值 |
|---|---|---|
| 衣服（关） | `RT/TankTop`、`RT/HotPants`、`RT/Belts`、`RT/Sandal`、`RT/RainCoat` | 0 |
| 睡衣/原版/胡萝卜 | `PJ_*`、`Orig_*`、`K_Carrot_*`（第 132 轮已随套装删除） | 0 |
| 饰品（开） | `RT/SharkTail`、`RT/SharkTail_Col`、`RT/Goggles`、`RT/SunVisor`、`RT/Necklace`、`RT/Bracelets`、`RT/Watch`、`RT/ThighStraps`、`RT/Anklet` | 1 |
| 水枪/冰棒 | `RT/WaterGun=1`；`RT/Snorkel=0`、`RT/IceBar=0` | — |
| 原版尾巴 | `Tail`（=1 显示原版尾巴，与 Riptide 鲨鱼尾可同时开） | 保持 |

内衣（比基尼）：用户要求泳装2 不出现内衣。**不要用「保留 GO + 参数 0」**（会被其它层/MA 规则重新拉起）。整组删除要清 6 处：

1. 菜单项 `菜单开关/泳装2/衣服/内衣`（DestroyImmediate）；
2. 手写 FX 层 `RT_Bikini` 与 animator 参数 `RT/Bikini`；
3. `ModularAvatarParameters` 里的 `RT/Bikini` 声明；
4. 场景 `Riptide/Riptide_BikiniTop`、`Riptide_BikiniBottom` 置 inactive（或直接删）；
5. 6 个 `OutfitSync` 状态各删 1 条 bikini driver；
6. 烘焙审计 `bikiniItems=0`、`bikiniPaths=0`。

### 14.8 水枪挂点「大腿 → 腰部」的改法（与 14.4 配套）

- 目标：`Riptide/Riptide_WaterGun/Armature/Root` 上的 `VRCParentConstraint` 的 `sources` 是 `source0/source1/source2` 三个**固定字段**（不是 List 索引，`FindProperty("Sources").arraySize` 为 0 是正常的）。
- 新建挂点 `Riptide/Constraints/WaterGun_Waist_ParC`：`ModularAvatarBoneProxy.boneReference=Hips`、`attachmentMode=AsChildKeepWorldPose`、世界坐标 `(0.0750, 0.4450, -0.0450)`（右腰外侧、鲨鱼尾上方）。
- `source0` 指向新挂点、`source1=Hand_R_ParC`、`source2=Hand_L_ParC`，三个 weight 分别 1/0/0（腰部态）。
- 菜单：`显示`（`RT/WaterGun` Toggle 1）、`腰部`（`RT/WaterGun_Hold` Toggle 0、`isDefault=true`）、`右手`（1）、`左手`（2）。
- 坑：改完挂点后必须把 `RT/WaterGun_Hold` 从**所有**一键穿衣状态的 driver 里删掉，否则每次换套装都被重置成 0（腰部），用户会以为「换手保持不住」。
- 验收：hold=0 时 constraint weights=(1,0,0)、枪位置 ≈`(0.075,0.445,-0.042)`、`dParC≈0.000`、枪口朝向 `muzzleDot≈0.95`；hold=1/2 时到左右手距离 ≈0.012。

## §14.9 第 133 轮：泳装2「只关上衣」语义 + 睡衣外套默认链 + 水枪腰部挂点

### 14.9.1 泳装2 语义修正：用户要的是「只脱上衣」

- 第 132 轮把 `RT/TankTop/HotPants/Belts/Sandal/RainCoat` 全写 0，用户实际语义是「不穿的只是上衣」。
- `LilLeo_FX_Wolf.controller` FX L9 `OutfitSync` 的 `Swim2` 态 driver 改为：
  - 关闭：`RT/TankTop=0`、`RT/Snorkel=0`、`RT/IceBar*=0`、`Tail=0`；
  - 打开：`RT/HotPants=1`、`RT/Belts=1`、`RT/Sandal=1`、`RT/RainCoat=1`，以及饰品 `SharkTail/SharkTail_Col/Goggles/SunVisor/Necklace/Bracelets/Watch/ThighStraps/Anklet/WaterGun=1`。
- GM 断言：`SWIM2|only_top_off|on:…HotPants/Belts/Sandal/RainCoat=VIS|off:TankTop=HID…`。
- 注意：不要用「删菜单项」表达语义，语义在 driver 里；菜单结构保持稳定。

### 14.9.2 睡衣「一键穿衣不穿外套」必须改四处默认链（缺一不可）

| # | 位置 | 字段 | 改前 | 改后 |
|---|---|---|---|---|
| 1 | MA Parameters `PJ_Outer` | `defaultValue`（`hasExplicitDefaultValue=true`、`saved=false`） | 1 | 0 |
| 2 | MenuItem `睡衣/睡衣/外套` | `isDefault` | true | false |
| 3 | MenuItem `睡衣/睡衣/外套` | `isSaved` | true | false |
| 4 | 源 EP `PJ_Outer` | `defaultValue` / `saved` | 1 / false | 0 / false |
| 5 | 场景 `睡衣/PJ_Outer` | `SetActive` | true | false |
| 6 | `OutfitSync` 的 `Default`/`Pajama` driver | `PJ_Outer` 条目 | 有 | 删（数组 35→34） |

- 「外套」菜单是 MA ObjectToggle：`Inverted=true`、两行 `PJ_Outer`/`PJ_PocketGhost` `Active=false`、`nearestMenu=PJ_Outer` → **语义 param0=隐藏、param1=显示**，别按「普通 Toggle」理解。
- GM：`DEFAULT|PJ_Outer=0.000`、`ONECLICK|S1_oneclick_pajama|off:PJ_Outer=HID`。

### 14.9.3 水枪挂点「大腿 → 腰部横放」

- 改 `Riptide/Constraints/WaterGun_Waist_ParC` 的 world pos/rot（`ModularAvatarBoneProxy` 子物体，烘焙保留 world 位姿）；同时把 `Riptide/Riptide_WaterGun/Armature/Root` 设成同位姿做编辑态预览。
- **最终值：world `(0, 0.5313, -0.1295)`、euler `(8.34, 65.67, 288.05)`（= roll 70°）。**
- 枪 PCA 局部轴：a0=(0.0020, 0.4327, 0.9015) 长 0.4168（枪管/muzzle）、a1=(-0.0030, 0.9015, -0.4327) 0.2486、a2 0.0821（薄轴）；roll 70 后世界盒 `(0.4168, 0.1237, 0.2414)`，底面 y=0.465 = 鲨鱼尾顶 0.4646，背面最近面留 1.5 cm。
- 菜单 `菜单开关/泳装2/水枪` 保持单层 Toggle 三值：`显示`(RT/WaterGun=1)、**`腰部`(RT/WaterGun_Hold=0) `isDefault=true`**、`右手`(=1)、`左手`(=2)，全部 `isSaved=true`。
- GM：`GUN|waist|pos=(0.000,0.531,-0.127)|dParC=0.000|muzzleDot=0.950`；右手 `(0.157,0.354,0.003)` `dHandR=0.012`；左手 `(-0.158,0.354,0.003)` `dHandL=0.012`；`persist_left` 60 帧后 hold=2。

### 14.9.4 穿模量化（新方法：体素有符号距离）

- Play 里把除水枪外所有启用 SMR `BakeMesh` → 世界顶点 + 法线 → 12 mm 体素（97407 顶点 / 8846 格）。
- 对水枪 17805 个顶点求最近点有符号距离 `sd=dot(w-bw, bn)`；`sd < -1.5 mm` 且 `d < 6 cm` 记穿模。
- 结果：`inside=7 / maxDepth=-2.0 mm / owner=Riptide_RainCoat`（0.04% 顶点，纯布料贴合）；身体 / 鲨鱼尾 0 穿模。

### 14.9.5 坑：Play 烘焙克隆里 ParC 路径查不到

- Play 烘焙克隆里 `Find("Riptide/Constraints/WaterGun_Waist_ParC")` 返回 **null**（路径被 MA/AAO 处理过）。
- 运行时改挂点必须走水枪 Root 上 `VRCParentConstraint` 的 `Sources.source0.SourceTransform`（`FindProperty("Sources")`，固定字段路径）。

---

## §15 MenuItem 显示名与 GameObject 名是两套东西：改名的正确落点（本工作区实测 2026-11-05，LuzzyMeow V2 沐梓溪狼 T2）

> 场景：把菜单项「睡衣（奶油条纹）」改成「休闲装」。先按 `Control.name` 改，烘焙后菜单还是老名字。

**判据（MA 官方原文 + 本工作区实测一致）**
- 官方 menu-item：「The name of the menu item will be taken from the name of the containing game object. This lets you see the name of, and rename menu items directly from the hierarchy.」→ 菜单文字默认 = **承载该 MenuItem 的 GameObject 名**。
- 本工作区 `MenuSource=Children` 实测同结论：把 `Control.name` 设成「睡衣1」，烘焙后仍显示老 GO 名（症状示例 `衣服 › EX Manu_OyasumiGhost`）。
- 只有 label 非空时才 label 优先（MA 1.14.3 源码口径，见 `references\05-动骨与穿模\05-01-outfit-fit-and-clipping.md` §7）；本工程 label 全空，两种口径结果相同。

**正解（改名三步）**
1. 改 GameObject 名：`go.name = "休闲装"; EditorUtility.SetDirty(go);`
2. 若确实在用 `MenuSource=Control` / label 非空，再同步改 `Control.name` / label。
3. 审计脚本按名字找菜单项时，用「GO 名 + Control 名 + label」三者任一命中，不要只认一个。

**坑**
- **param 留空的自动参数项，参数名由名字派生（`__MA/AutoParam/…`），改名会连带改掉参数名** → FX 条件、driver、EP 全部对不上。只对显式声明参数的固定项（一键穿衣、总开关）改名。
- 中文名改坏功能的真实案例：`references\07-验证与排查\07-05-ma-gotchas-version-toolchain.md` §1.10 (L85)。
- 显示名不是功能键：菜单去重（§12）与「按名找项」的脚本都要按上面三者联合匹配。

**适用场景**：菜单项改名；按名做菜单审计/去重；从别的工程复制菜单结构后统一改名。

---

## §16 删衣服部件的四件套清理清单：菜单项 / 参数 / FX 层 / 网格素材（本工作区实测 2026-11-05，LuzzyMeow V2 沐梓溪狼 T5/T7/T8/T9）

> 场景：删泳装2 的鲨鱼尾、呼吸管、冰棒、上衣。只删菜单项 = **最危险**的删法。

**清理顺序（缺一件就留「看不见但还在」的残留）**
1. **FX 驱动**：先改 FX 里所有写该参数 / 该对象 `m_IsActive` 的层与 driver 条目（该套态改常量或整条删），默认态调到隐藏。
2. **菜单项**：删 MenuItem GO；删完父菜单项数变化，复检 ≤8 项、没多出 next 分页。
3. **参数**：`ModularAvatarParameters` 声明 + 源 `VRCExpressionParameters` + AnimatorController 参数三处一起删；多消费者参数（Hold/Status/Result 这类）整组删。
4. **素材**：SMR / 网格 / 材质 / 相关 PhysBone 与 Collider；删完跑一次烘焙，看 transforms / SMR / 三角面数字是否下降。

**整套衣装的完整版 = 6 处**（本文件 §14.5）：场景套装根 GO、菜单子树、MA Parameters 声明、FX 各态 driver 条目、一键穿衣里「该套装的状态」（**不要删状态**——改为把该态 driver 改成默认取值、保留 AnyState 条件）、烘焙后审计。

**坑**
- 只删菜单项：参数还在、FX 还在，OT 不再介入 → 回落到动画状态，对象可能反而常显或常隐。
- 删状态：saved 参数会把玩家上次选择带回世界，被删状态值可能仍是存档值 → 表现为「一进世界就是空装」。
- 编辑态显隐不代表烘焙后显隐，判据一律以烘焙产物为准（`references\03-菜单与参数\03-01-parameter-and-menu-audit.md` §4 (L123)）。
- 复检其它开关同时确认三件事：有绑定（未绑定的 MenuItem 官方明确「has no effect」）、宿主 GO 在激活链上、默认值三处一致。

**适用场景**：任何删部件/删套装；本次 T5 鲨鱼尾、T7 呼吸管、T8 冰棒、T9 上衣。

> 更新时间 2026-11-05（本次来源：沐梓溪狼衣装菜单重构 / LuzzyMeow V2 T1–T10）

---

## §17 第三方道具 prefab 自带菜单归位到「新建的一级子菜单」（本工作区实测 2026-11-09，LuzzyMeow V2 沐梓溪狼）

> 场景：买来的道具（本例 BOOTH 6867118「ネガライン」/ おぎにり，MA 対応）装到现有头像，用户要求
> **轮盘里新开一个「道具」子菜单，把原作者菜单放进这个子菜单下**。

### 17.1 这类 prefab 的菜单结构（先认清楚再动手）

实测该 prefab 根下：

```
Negaline                      [Animator, ModularAvatarMergeAnimator, ModularAvatarParameters]
  Anim_nega_off/...           （骨架代理与零件，BoneProxy → LeftHand / RightHand / RightLowerArm）
  Nega_menu                   [ModularAvatarMenuInstaller + ModularAvatarMenuItem]
    Gimmick_on                [ModularAvatarMenuItem]
```

- `Nega_menu`：`MenuItem.Control.type = SubMenu`、`MenuSource = Children` → 它**自己就是一个子菜单**，子项来自直接子节点（`Gimmick_on`）。
- `Gimmick_on`：`Toggle`，参数 `negaline`（`ModularAvatarParameters` 声明：Bool / sync / default 0 / saved=false）。
- `MenuInstaller.installTargetMenu = null`、`menuToAppend = null` → **默认装到一级菜单**（这正是用户看到的「道具跑到顶级」）。
- 因为同 GO 上既有 Installer 又有 MenuItem，MA 走 `PushNode(MenuSource)` 分支：**把 GO 自己的 MenuItem 当成一项装进去**（见 §12.2 根因 A），不是平铺 assets。

### 17.2 四步做法（不改作者结构，只换落点）

1. **新建空菜单资产**：`Assets/08_动画·菜单·图标/沐梓溪狼菜单/沐梓溪狼_菜单_道具.asset`
   （`ScriptableObject.CreateInstance<VRCExpressionsMenu>()` + `AssetDatabase.CreateAsset`，`controls = []`；烘焙期由 installer 填内容，源资产保持空是正常的）。
2. **根菜单加一项**：`Control{ name="道具", type=SubMenu, subMenu=道具资产 }`。
   `Control` 是 **struct**，必须「取出 `List<Control>` → Add → 写回 `menu.controls` → `EditorUtility.SetDirty(menu)` → `AssetDatabase.SaveAssets()`」，
   否则 `subMenu` 落盘成 `{fileID: 0}`（同 §9.1）。
3. **改 installTargetMenu**：把 `Nega_menu` 上 installer 的 `installTargetMenu` 指向 `道具` 资产（`menuToAppend` 保持 null）。
   此时「作者的菜单」原样成为 `道具` 下的一项，作者将来更新 prefab 语义不变。
4. 存场景（`EditorSceneManager.SaveScene`）。

### 17.3 烘焙后判据（实测）

```
根 › 道具 › Nega_menu › Gimmick_on   [Toggle] param=negaline value=1
EP: negaline Bool def=0 saved=False synced=True   （182 参数 / 216 bit）
FX: layer 'nega' weight=1 default=gimmick_off（clip: Negaline/Anim_nega_off.m_IsActive=0）
    layer 'blast&shot' default=default / 'sight' default=default
```

> ⚠️ 这里的 `Nega_menu` 这一层是**有意保留**的「作者菜单壳」，**不是** §12.3 那种多余层：
> §12.3 多出来的是「父菜单里已经挂了一条指向空壳资产的 control」再被 installer 装一次；
> 本例 `道具` 是空菜单资产、父菜单里只有一条 `道具` control，烘焙后不会出现 `道具 › 道具 › …`。

### 17.4 两个坑

| 现象 | 根因 | 做法 |
| --- | --- | --- |
| Play 里直接 `GameObject.Find("沐梓溪（小狼）Ver.2")` 后立刻 `transform.Find` 拿到 null / 报 NRE | `EditorSceneManager.OpenScene` 返回时场景还没装完，**同一帧**访问对象会 NRE | 重载场景与读对象**分两帧**（两次 execute_code / 两次 update） |
| 编辑态看这个头像像「裸体」 | MA 的 `MergeArmature / OutfitRoot` 等只在烘焙期改写网格；未烘焙的编辑态 outfit SMR 的 bindpose/root 未重算，**渲染位置与缩放都不对**（本工作区实测：编辑态镜头里只剩 Body + 少量件，看起来没穿） | 外观判据一律看**烘焙克隆**或 Play；编辑态裸/穿错不算缺陷（与 SKILL 铁律 1 同源） |

> 更新时间 2026-11-09（来源：本轮安装 ネガライン 道具 + 新增「道具」一级子菜单；证据 `docs/_r311_verify.txt`、`docs/_r311_play.txt`、`docs/shots_r311/`）


---

## 17. 2026-11-09 实测：AFK 可见性用 MergeAnimator 驱动

**来源**：<https://modular-avatar.nadena.dev/docs/reference/merge-animator>

**现象**：AFK 动画导入后菜单按钮能切换参数，但模型上看不到机器。

**修复**：在 AFK 根对象挂 MA `MergeAnimator`，使用生成的 `R400_AFK_Visibility.controller`：

- `AFK=true` → `R400_AFK_On.anim` 显示机器 Renderer；
- `AFK=false` → `R400_AFK_Off.anim` 隐藏机器 Renderer。

**判据**：烘焙产物中应存在该 MergeAnimator 层及两条 clip；Play 中切换 `AFK` 后 Renderer 可见数从 0 变正、再回 0。不要只在场景里手动勾选机器对象当作验证。

## §18 菜单控件语义补全（Button 脉冲 / RadialPuppet / label 去重 / SubMenu 源）

> 来源：原 08-症状速查 第 132/151/156/166/177 轮实测（2026-10-31 至 11-11），2026-11-11 结构化并入。

### 18.1 Button 是脉冲：绑 Int 做三选一必错，绑 Bool 做触发器是对的

- **错误用法（第 156 轮）**：「鞋子/袜子/裸足」三项都做成 Button 绑同一个 Int（0/1/2）。VRChat Button 置值后约 1 秒自动回落 0 → 永远停在 0 档。判据：点「袜子」后 OSCQuery `LH_foot` 0→1→0，模型闪一下又回鞋子。
  - 修法：三选一必须 **Toggle + 同一参数不同 value（radio 互斥）**；同时把作者动画层条件从 `0/1/2` 改成 `1/2/3`，并补一条 `==0 → 默认档` 回落转移（`LittleHowler.controller` 层 `foot`）。
- **正确用法（第 151 轮）**：Button + **Bool** 是脉冲触发器（置 1 自动回落 0，配 `IfNot` 做一次性触发），前提 EP 该参数 `saved=0`。面捕 FT On/Off 即此设计，详见 [04-08 §4.1](../04-外观与表情/04-08-face-tracking-osc-and-repair.md)。
- **Button 自动复位 GM 不模拟（第 132 轮）**：水枪菜单点「右手/左手」约 1 秒后自己回到腰部——控件是 Button 不是 Toggle。改 `Control.type=Toggle(102)`、value 0/1/2（0 值项当复位），并把 `RT/WaterGun_Hold` 从所有一键穿衣状态 driver 删掉。**GM 全绿也会在游戏里回跳**。

### 18.2 RadialPuppet 的参数槽是 subParameters，不是 parameter

MA MenuItem 的 RadialPuppet 若 `Control.parameter` 与 `Control.subParameters` 都为空，MA 生成 `ModularAvatar$synthetic$float$<hash>` 合成参数，没有任何动画层消费它（第 156 轮：音量滑块怎么拖都没效果）。
- 判据：OSCQuery `/avatar/parameters/` 下大量 `ModularAvatar$synthetic$bool|float$…`；真正的 `RK_volume` 恒 0.5。
- 修法：被驱动参数写进 **`subParameters[0].name`**；写完拖滑块应看到该参数 0..1 变化。

### 18.3 复制 MenuItem 必须清 label（MA 按 label 去重）

`EditorUtility.CopySerialized` 从已有 MenuItem 复制出新 GO 后，烘焙菜单树只出现 SubMenu 节点、子控件全部消失（第 166 轮）：复制把源组件的 **`label`** 一并带入，3 个新项共用同一 label（「花海」），MA 构建子菜单时按 label 去重丢弃。`MenuSource=Children`、`Control.subMenu=null` 都正确也照样丢。
- 判据：读回新项 `SerializedObject.FindProperty("label").stringValue` 全部相同；烘焙菜单 BFS 只命中 SubMenu。
- 修法：**新建 MenuItem 宿主 GO 时清空 `label`**（必要时也清 `Control.icon`）；判据=烘焙菜单树逐项出现子控件名 + `Control.value`。

### 18.4 代码新建 SubMenu 必须显式 MenuSource=Children

`ModularAvatarMenuItem.SubmenuSource` 默认值是 `MenuAsset`（读 `Control.subMenu`，此时 null），不会用子物体组织子菜单 → 新建 SubMenu 在烘焙菜单里整棵消失（第 132 轮）。建 SubMenu 后必须显式 `mi.MenuSource = SubmenuSource.Children`；判据=与场景里可用项（如 `小狼菜单`=Children）对比。

### 18.5 NDMF 烘焙拒绝 prefab 实例

NDMF 烘焙直接抛 `Can't process an avatar that contains prefab instances/assets`（第 132 轮）：`BuildContext..ctor` 对每个 Transform 做 `PrefabUtility.IsPartOfAnyPrefab`，一个都不放过。
- 做法：找出 `GetOutermostPrefabInstanceRoot` 去重后的根（本工程 216 个 Transform → 3 个根：`RKSActivator` / `RK Player` / `兜帽卫衣`），逐个 `UnpackPrefabInstance(Completely, AutomatedAction)`。按需解压的通用纪律见 [01-06 §2.8](../01-环境与工具链/01-06-project-launch-and-mcp-bridge.md)。

### 18.6 菜单归属 / Installer 语义 / 面捕菜单重定向（交叉引用）

- 菜单归属三处交叉判定（路径属作者包 + mtime 未触碰 + 内容与截图逐项一致）：[04-08 §4.2](../04-外观与表情/04-08-face-tracking-osc-and-repair.md)。
- MenuInstaller + MenuItem 同对象是正常写法（`VirtualMenu.PushNode` if/else if，不会双入口）：[04-08 §4.3](../04-外观与表情/04-08-face-tracking-osc-and-repair.md)。
- 面捕菜单重定向四步（新建子菜单资产，MenuItem.subMenu 与 installer.installTargetMenu 同指，避免 FT Controls 自递归）：[04-09 §12.2](../04-外观与表情/04-09-face-tracking-native-sdk.md)。

### 18.7 控制器 YAML 修复：悬空转移（Broken text PPtr）

**症状**：控制器加载正常但 Editor.log 反复刷 `Broken text PPtr in file(...controller)... Local file identifier (NNN) doesn't exist!`（第 156 轮）。
**根因**：删除状态时留下**悬空 `AnimatorStateTransition`**（`m_DstState` 指向已不存在的 AnimatorState），条件里还留着早已删除的参数名（本例 `Shoes`/`Garters`）。
**判据**：正则在 controller YAML 里比对：所有 `--- !u!1101 &` 块中的 `m_DstState` fileID 是否都在文档 id 集合里（本例 4 条悬空）。
**修法**：按 YAML 精确删除这 4 个 `!u!1101` 块，并同步删掉其它块 `m_Transitions` 里对它们的引用行；改完读回 `nullDst=0`。文件 `Assets/08_动画·菜单·图标/沐梓溪狼菜单/LilLeo_FX_Wolf.controller`。
**注意**：这与 [07-09 §1.7](../07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md) 的「多对象 .asset 子资产 destinationState 读不出」是两回事——后者是读取方法不可靠，前者是真损坏（有 Editor.log 报错）。

## §19 一键穿衣 / 分部件混搭补全（权威写者 / 互斥矩阵 / 极性案例）

> 来源：原 08-症状速查 第 141/166/168/169/171/175/176 轮实测（2026-11-06 至 11-11），2026-11-11 结构化并入。架构演进主线（Button→Toggle+Int→删 OutfitRoots→套根常驻 active）见 §14；本节是其后的补全与裁决。

### 19.1 分部件混搭范式（兜帽卫衣标杆，第 169 轮）

一件衣服只有一个「显示」总开关时，单参数+单 OT 天然无法混搭。照 `衣装开关/兜帽卫衣` 范式重建：
1. 服装根 `activeSelf=True`、每个部件 `activeSelf=False`（MA 场景初始态）；
2. 每个部件一个代理 GO，挂 `MenuItem(Toggle, 独立 Bool 参数, value=1, isSaved=true, isDefault=false)` + `ObjectToggle(target=该部件, Active=true, inverted=false)`；
3. 根节点 MA Parameters 集中声明 N 条 `Bool / defaultValue=0 / hasExplicitDefaultValue=true / saved=true`；
4. 源 EP 同步登记 N 条 Bool；
5. 一键穿衣 driver 里把总开关参数替换成 N 条部件参数（同一值）。
**判据**：烘焙后出现 **N 个独立 `MA Responsive` 层、条件互不相同**。

### 19.2 MA Responsive 层名 = OT 目标对象名（不是代理 GO 名）

按「代理对象名」搜层会找不到（第 169 轮）：代理叫「上衣/短裤/袜子/鞋子」，层叫 `MA Responsive: top1 / shorts / sock / shoe`。**验收按 OT 目标 GO 名搜层**；§9.3 旧表述在本工程属补充/修正（代理名与目标名恰好同名才没暴露）。

### 19.3 每个参数只允许一个权威写者（多层 driver 后求值者赢）

**症状**：点「兜帽卫衣一键穿衣」后再点「万圣节卫衣一键穿衣」，模型变裸体（第 171 轮）。
**根因**：`OutfitSync`（FX 层序 12）写 `HW_Tops=1`，但 `LH_OneClick`（层序 **82**，在后面）两个状态的 driver 也写 `HW_Tops=0` → 后求值的层覆盖先求值的层；同时休闲装被 OutfitSync 关、兜帽被 LH_OneClick 关 → 三件都没了。
**判据**：烘焙产物里打印每个参数的写入者（按层索引排序）：`HW_Tops` 同时出现在 `[12]OutfitSync/*` 与 `[82]LH_OneClick/Idle|AllOn`。
**修法**：**每个参数只允许一个权威写者**。`LH_OneClick` 只管自己的 `LH_*`，绝不写别的套装参数；跨套装互斥统一由 `OutfitSync` 负责。修完 `HW_*` 写入者只剩 `[12]OutfitSync/*`。属性→层反查脚本见 [02-11 §4](../02-工程组织与工作流/02-11-plugin-integration-rk-realkiss-props.md)。

### 19.4 每个一键穿衣状态都必须保证身上有衣

`OutfitOneClick=2`（Swim）与 `=4`（Original）可见 Renderer 仅 10（第 171 轮）：菜单入口已随「泳装1/原版衣服」删除，但 `OutfitSync` 的 AnyState 转场与状态仍在，参数被误写时 driver 只开 `Tail`（全 0）。
- 修法：给无入口的历史档位补 `YURU_Tops/Pants/Sox=1` 兜底，或删掉对应 AnyState 转场。**删菜单 ≠ 删状态**。
- 判据：Play+GM 遍历 7 档读 `visibleRenderers`，每档 ≥14（本例 2→10、4→10 即缺陷）。

### 19.5 档位条件用 Equals N，不用浮点区间

6 档转场写 `Greater 5.9 + Less 6.1`，经 VRCFury 中间产物会变成整数边界条件（第 166 轮）。**每一档都用 `Equals N`**；6 档 driver 补齐所有其它套装参数=0，再在 `LH_OneClick/AllOn` driver 追加同一批参数=0 做双保险（注意 §19.3 权威写者边界：双保险只写自己套装的参数）。
- 判据：最终烘焙 FX `ANY|Hoodie|OutfitOneClick:Equals:6` + 截图无叠穿。
- **中间产物特性**：`VRCFury FX.controller` 中间产物里同层 w=0，**最终产物 w=1，不要误判**。

### 19.6 互斥矩阵扩档（万圣节 8 态，第 168 轮）

新套装接入一键穿衣互斥组：① 所有现有状态 driver 追加 `HW_Hoodie=0`；② 新建 `Halloween` 状态（`OutfitOneClick Equals 7` 转入），driver 设 `HW_Hoodie=1, Tail=1` + 全部其它套装参数=0（含 `LH_Tops/Bottoms/Shoes/Socks/OneClick`）；③ `LH_OneClick` 的 Idle/AllOn 也追加 `HW_Hoodie=0`（历史做法；按 §19.3 现行口径应只由 OutfitSync 写）。
**判据**：8 状态 × `HW_Hoodie` 互斥矩阵全闭合。菜单重建时 ObjectToggle 的 `Object.targetObject` 必须指向 GameObject 并回读验证非 NULL。

### 19.7 分组根「原子关、壳必开」（第 175 轮）

`睡衣` / `Riptide` / `兜帽卫衣` / `万圣节卫衣` 这类**分组根**，正确默认态是：**分组根 `activeSelf=True`、其子件（PJ_Shirt / Riptide_BikiniTop / LH_Tops / top1…）`activeSelf=False`**。
- 原因：MA ObjectToggle 为每个受管对象生成 `__MA/ActiveSelfProxy/<名字>##<n>` 参数（本轮实测 67 个），运行期 Animator 改**子件**的 `m_IsActive`。分组根若 False，子件即使被动画设 True 也不渲染 → 「菜单点了没反应 / 衣服永远不出来」。
- 自查口诀：**「原子关、壳必开」**——关的是叶子网格，不是容器。与 §14「删 OutfitRoots、套根常驻 active」同口径。

### 19.8 极性案例：娃娃机「音声OFF」默认 OFF 要写 1（第 176 轮）

用户要「音乐开关默认 OFF」，控件名「音声OFF」，直觉写默认值 0 —— **会改反**。
- **极性裁决（看接线，不看命名）**：OT（m_inverted:0）目标 `まめフレンズAFKくれーんげーむーむ/移動音`、Active:0，「移動音」GO 是唯一 AudioSource。MenuItem「音声OFF」Toggle value=1 参数「音声」：条件成立（音声=1）→ 非反相 OT 把目标设 inactive → **音声=1=静音（音乐 OFF）；音声=0=音乐 ON**。BOOTH 6279290 更新履历 2024/11/13 v0.1.0「アーム移動時の効果音追加」佐证出厂带声。
- **最小改法（3 字段，场景实例级）**：① MA Parameters「音声」defaultValue 0→1；② hasExplicitDefaultValue 0→1（0 会走回退链回 0，不改等于没改）；③「音声OFF」MenuItem isDefault 0→1（MA 1.18.7 `ParameterAssignerPass.cs:105-108`：单 MenuItem + automaticValue + isDefault → defaultValue=1）。
- **不改**：Control.value/type/parameter、OT 的 m_inverted/Active/target、AudioSource、AKF*.anim 的 m_Enabled 曲线、6 个商业 prefab、ExMenu/*.asset、任何 EP/源 controller、AFK 条目。saved:1 保留。
- **saved:1 的坑**：本地上次保存值会覆盖 defaultValue，改默认只对「无本地保存记录的玩家 / Reset Avatar」生效。要「每次加载强制静音」需连带 saved→0（代价：手动开音乐不记住），超出「改默认」最小范围，须用户确认。
- **烘焙判据**：baked EP EP[183] 音声 def=1 saved=True type=2(Bool)；baked 菜单 MENUCTRL[2] 音声OFF param=音声；FX 层 `MA Responsive: 移動音`（weight=1，<default>↔音声OFF 阈值 0.995/1.005）→ clip 写 `移動音 :: m_IsActive`。
- **共享写面**：AKF/AKF右/AKFIN 等 clip 以 classID:82(AudioSource) `m_Enabled` 写同一 GO 的 AudioSource.enabled，OT 写的是 GameObject.m_IsActive —— 同对象不同字段，静态不能断言音效无回归，必须 GM 动态复核（本轮因 Play 卡住标 NOT_RUN）。

### 19.9 混搭持久化的最后一道锁（第 141 轮）

部件 bool 都已 saved=true，重登还是回套装默认：最后一道锁不在 EP 也不在 MenuItem——`参数设置` 上的 MA Parameters 显式写了 `OutfitOneClick def=1 / saved=true`，**MA explicit 会盖掉源 EP**，加载即 1 → 进 Pajama 态 → 56 条 driver 把部件全刷一遍。
- 判据：烘焙 clone EP `OutfitOneClick def=1 saved=True`（源 EP 明明是 0/false）；`OutfitSync/Default drivers=35`。
- 修法三处一起：① MA Parameters `OutfitOneClick def 1→0, saved→false`；② 源 EP + FX 参数默认 0；③ `OutfitSync/Default` 的 driver **全清**（其余套装态保留）。判据=Play 设 `PJ_Shirt=0/YURU_Tops=1` 后把 OC 归 0，两值不变。

**完整五步口径（历史补充行，VRChat 内实测）**：混搭状态（一件套装的若干部件开关）重登后要恢复，需同时满足：
1. 所有部件参数（`PJ_*`/`SW_*`/`Orig_*`/`RT/*`/新套装参数）EP `saved=true`；
2. `OutfitOneClick` EP `default=0, saved=false`，控制器参数默认也设 0；
3. 清空 `OutfitSync` Default 态的 driver，只保留各套装切换态的 driver；
4. 套装切换态里把新套装参数置 0，新套装态把旧套装参数置 0；
5. 一键穿衣 MenuItem `isSaved=false, isDefault=false`。
判据=重登后部件 saved 值恢复、Default 态不覆盖。与 §19.9 三处修法是同一问题的不同切面；saved 总规则见 [03-01 §10.3](03-01-parameter-and-menu-audit.md)。

### 19.11 driver 层创建与接管的硬坑（第 159/160 轮 + 历史补充）

- **`VRC_AvatarParameterDriver` 是抽象类**：实例化要用 `VRC.SDK3.Avatars.Components.VRCAvatarParameterDriver`；**编辑器态 `AddStateMachineBehaviour` 返回 null**，必须 `ScriptableObject.CreateInstance` + `AssetDatabase.AddObjectToAsset` + 往 `m_StateMachineBehaviours` 数组塞引用（第 160 轮）。CodeDom 下 `AddStateMachineBehaviour` 返回 null 时改用 MA 纯响应方案或手动建层（历史补充行）。
- **空层不报错也不产生 driver**：`LH_OneClick` 层 `stateMachine.states.Length==0` 时 MA 合并静默跳过（第 160 轮）。重建：`Idle`(default) ⇄ `AllOn`，条件 `OutfitOneClick Equals/NotEqual 6`，两态各挂一个 driver（AllOn 置 5 个 bool=1，Idle 置 0 保证切走整套隐藏）。判据：读层 `states>0`、`defaultState!=null`。
- **一键穿衣联动多件**：MenuItem 的 Control 只描述「一个参数+一个值」，ObjectToggle 只能控物体——用 driver 层：`Idle →(OutfitOneClick Equals 6) AllOn`，AllOn 挂 N 条 Set，`AllOn →(NotEqual 6) Idle`。**驱动层无 clip，不会被 MA 的 ObjectToggle 吸收/清权重**（第 159 轮；对照 [04-06 §14.9](../04-外观与表情/04-06-ma-reactive-components.md)）。
- **拆独立开关时接管作者层**：作者 `foot` 层用 `LH_foot`(1/2/3) 直接写 `LH_Shoes/LH_Socks` 的 m_IsActive，与 MA OT 生成层写同一属性会互相抢。想拆两个独立 Bool：**把作者 `foot` 层 `defaultWeight` 置 0**（不是删层，便于回滚），再用两个 Bool + OT 接管。`layers` 是数组属性，必须 `var a=ac.layers; a[i]=L; ac.layers=a;` 整体回写，只改 `ac.layers[i]` 不落盘（第 159 轮）。
- **新套装接入旧选择器的空闲档位**：新件 MenuItem 的 `Control.parameter` 设为旧选择器参数（如 `OutfitOneClick`）的空闲档位（3=已删除的 Carrot），VRChat 同参数不同 value 的 Toggle 自动 radio 互斥；ObjectToggle 仍列旧套装父文件夹 `Active=false`；新件不再占用独立 Bool 参数（历史补充行）。
- **新增套装先照抄现有规格**：先检查现模型所有 `衣服` 子菜单的层级和字段，再照抄同一规格（`衣服 → 套装名(SubMenu) → 一键穿衣(Toggle) + 各部件(Toggle)`）；每个部件挂 ObjectToggle，参数命名与现有风格一致；验收=烘焙后菜单层级、控件数量、参数类型与旧套装一一对齐（历史补充行）。

### 19.12 Rest Pose 静态态：游戏内 3D 预览裸体的根因（第 158 轮）

**症状**：游戏内「还没切到这个头像时的 3D 预览」里模型是裸体（不是头像缩略图）。
**根因**：出包 prefab 的**静态态（Rest Pose）**里默认套装件 `activeSelf=false`：MA 用 `MenuItem.isDefault` 判定「该参数默认值对应哪个菜单项」；默认档件参数默认=1 但 `isDefault=false` ⇒ **反向 ObjectToggle 规则被当成初始生效** ⇒ 烘焙期把对象判为隐藏。另 `m_overrideAnimatorDefaults` 默认 false → 烘焙 FX 里 `OutfitOneClick` 默认=0（EP 是 3）。
**判据**：读 `.vrca` prefab：`睡衣_Yurumoko` 7 个 renderer 全 `activeInHierarchy=false`；烘焙 FX `OutfitOneClick.defaultFloat=0`；`MenuItem.isDefault` 与 EP 默认值不一致 **19 处**。
**修法**：① 把「EP/MA 默认值 == 菜单项 value」的 MenuItem 全部 `isDefault=true`；② 给该参数在 MA Parameters 打 `m_overrideAnimatorDefaults=true`（用 SerializedObject 写，SetValue 对 struct 副本无效，见 [03-01 §10.8](03-01-parameter-and-menu-audit.md)）。
**判据**：出包 prefab 默认档件 active=True（本例 6/7）+ 控制器 defaultFloat = EP 默认值。工具 `Assets/Editor/R216RestPoseFix.cs`。
**关联**：别人屏蔽 Avatar Animations 时只渲染序列化静态态，双 OT + isDefault 的完整口径见 [02-02 §26](../02-工程组织与工作流/02-02-ma-workflow-habits.md)。

### 19.10 组合测试与 GM 边界（交叉引用）

GM 单点串行点击测不出跨参数累积（`Vrc3Param.Set` 下一帧被模块重置）；混搭累积要用组合测试脚本一次 Set 多参数、隔 5 帧读 visibleRend 单调递增（15→17→18→19→20）。配方与 Action 层/Contact 模拟见 [07-06 §23](../07-验证与排查/07-06-gesture-manager-automated-sim-test.md)。
