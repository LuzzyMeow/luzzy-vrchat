# SDK 面板填不进去 → 反射 Build + 反射上传（2026-10 版 SDK）

> 本工作区实测 2026-10-27（第 102 轮，LuzzyMeow V2 / Unity 2022.3.22f1c1 / VRChat SDK 3.x）。
> 场景：新头像要 Build & Publish，但 **Unity UI Toolkit 输入框键盘输入无效**（面板 Name 填不进去 → 按钮一直禁用），且面板状态混乱。
> 结论：**面板只是壳，真正的 API 都在 `VRC.SDKBase.Editor.Api.VRCApi` + `VRC.SDK3A.Editor.VRCSdkControlPanelAvatarBuilder` 上**，可以用编辑器脚本反射调用完成「出包 + 上传」。

---

## 1. 症状与判断

| 症状 | 原因 |
|---|---|
| `mcp__windows__Type` / `Clipboard + Ctrl+V` / `SendKeys` 全写不进面板输入框（框有蓝焦点但始终空） | 本机键盘事件到不了 Unity 的 UI Toolkit 输入框（鼠标点击有效）。**不要在这上面耗时间** |
| 面板底部一直 "Please set a name and thumbnail before uploading"、按钮灰 | `_avatarData.Name` 为空 或 `_newThumbnailImagePath` 为空（源码 line 2584 判据） |
| Content Manager 一直 "Fetching Records" | 面板侧网络线程/状态机卡住；不影响 API 调用 |
| 点一次 Build & Publish 后，`GetAvatar` 显示 `Version=1`、`PendingUpload=True`、`UnityPackages=0` | 只完成了 `ReserveAvatarId`（`CreateAvatarRecord` 建记录并写 blueprintId），**bundle/图片上传失败或被中断** |

---

## 2. 面板内部结构（填字段用，源码 `Packages/com.vrchat.avatars/Editor/VRCSDK/SDK3A/VRCSdkControlPanelAvatarBuilder.cs`）

```csharp
public class VRCSdkControlPanelAvatarBuilder : IVRCSdkAvatarBuilderApi
private static VRCSdkControlPanelAvatarBuilder _instance;   // 单例，反射取
VRCTextField  _nameField;         // .value 直接可用
VRCTextField  _descriptionField;  // .value
TagsField     _tagsField;
ThumbnailBlock _thumbnail;        // .SetImage(Texture2D)
string        _newThumbnailImagePath;   // ★ 构建按钮判据用的是这个
VRCAvatar     _avatarData;        // struct
void HandleAddTag(object, string tag);                    // 内部拼 author_tag_<lower_snake>
void HandleThumbnailChanged(object, string imagePath);    // 设 _newThumbnailImagePath + 面板图
void HandleNameChange/HandleDescriptionChange(ChangeEvent<string>)
```

- 反射取 `_instance`：`typeof(VRCSdkControlPanelAvatarBuilder).GetField("_instance", BindingFlags.NonPublic|Static).GetValue(null)`；若为 null 先 `execute_menu_item("VRChat SDK/Show Control Panel")` 让面板初始化。
- 缩略图**必须**经过 `VRC.SDKBase.VRC_EditorTools.CropImage(string sourcePath, float width, float height, bool centerCrop, bool forceLinear, bool forceGamma)`（
  internal static，assembly `VRC.SDKBase.Editor`）裁 800×600 —— **反射调用要传满 6 个参数**，少传会 `TargetParameterCountException`。
  它把裁剪图写到 `%TEMP%\<guid>_cropped.png` 并返回绝对路径。
- 填完这三样（name / 缩略图路径 / tags / description），面板按钮才会亮。

---

## 3. 反射出包：调公开的 `Build(GameObject)`

```csharp
// 1) 构建类型（不设会默认 Build & Test）：反射写 VRC.SDKBase.Editor.VRC_SdkBuilder.ActiveBuildType
//    枚举 VRC_SdkBuilder+BuildType.Publish
// 2) 找方法：public async Task<string> Build(GameObject target)  （源码 line 3478）
var m = tBuilder.GetMethods(BindingFlags.Public|Instance)
                .First(x => x.Name == "Build" && x.GetParameters().Length == 1);
var task = (Task<string>)m.Invoke(_instance, new object[]{ avatarGO });
task.ContinueWith(t => File.WriteAllText(logPath,
        t.IsFaulted ? "FAULT=" + t.Exception.GetBaseException() : "DONE\nBUNDLE=" + t.Result));
```

- 返回的就是 AssetBundle 路径：本工作区实测 `<USERPROFILE>\AppData\Local\Temp\DefaultCompany\<工程名>\<guid>.vrca`。
- **耗时长**：本工作区 NDMF + AAO + SPS shader 变体编译约 **6 分钟**，期间 Unity 弹 `Compiling shader variants (busy for 09:16...)`，`Hidden/SPSPatched/<hash>` 变体可到 96 个；
  此时 **`mcp__unity__*` 全部超时属正常**（主线程被占），不要中断、不要点 Cancel。
- CodeDom(C#6) 编译约束：变量名不能在父子作用域重复（`A local variable named 'bt' cannot be declared in this scope…`）→ 统一加后缀；不能有局部函数；lambda 捕获的变量不要和外部同名。

---

## 4. 反射上传：`VRCApi.CreateNewAvatar`（首次上传专用）

`VRCAvatar` 是 **struct**（`Packages/com.vrchat.base/Editor/VRCSDK/Dependencies/VRChat/API/VRCAvatar.cs`）：
`[JsonProperty("id")] public string ID {get;set;}`（**是属性不是字段**，反射读值用 `GetProperty`）、`Name`、`Description`、`List<string> Tags`、`ImageUrl`、
`ThumbnailImageUrl`、`ReleaseStatus`、`bool PendingUpload`、`int Version`、`List<VRCUnityPackage> UnityPackages`。

```csharp
var avType = Type.GetType("VRC.SDKBase.Editor.Api.VRCAvatar, VRC.SDKBase.Editor");
object boxed = Activator.CreateInstance(avType);            // struct 装箱
avType.GetProperty("ID").SetValue(boxed, "avtr_xxx", null);
avType.GetProperty("Name").SetValue(boxed, "沐梓溪狼", null);
avType.GetProperty("Description").SetValue(boxed, "…", null);
avType.GetProperty("Tags").SetValue(boxed, new List<string>{"author_tag_xxx"}, null);
avType.GetProperty("ReleaseStatus").SetValue(boxed, "private", null);

var createM = typeof(VRCApi).GetMethods().First(x => x.Name == "CreateNewAvatar");
// 签名：CreateNewAvatar(string id, VRCAvatar data, string pathToBundle, string pathToImage,
//                      Action<string,float> onProgress = null, CancellationToken ct = default)
var task = (Task)createM.Invoke(null, new object[]{ id, boxed, bundlePath, imagePath, null, CancellationToken.None });
task.ContinueWith(t => File.WriteAllText(log, t.IsFaulted ? "FAULT="+t.Exception.GetBaseException() : Dump(t.Result)));
```

- 前置条件：该 avatar 记录的 `PendingUpload` 必须为 `true`（本工作区正好是「已 Reserve 未上传」状态），否则抛 `UploadException("Avatar creation is only allowed for a reserved avatar ID pending first upload")`。
- 成功判据（实测）：`PendingUpload=False`、`Version=2`、`UnityPackages=1`、`ImageUrl` / `ThumbnailImageUrl` 非空。
- 后续更新（非首次）：`UpdateAvatarInfo` → `UpdateAvatarImage` → `UpdateAvatarBundle`（`VRCApi.cs:598/617`）。
- **别在主线程 `.Result` 等待**（会死锁）→ 用 `ContinueWith` 把结果写文件，再轮询读文件。

---

## 5. 上传/构建的副作用清理（本工作区实测清单）

| 副作用 | 处置 |
|---|---|
| `Assets/08_动画·菜单·图标/NyaPPu__Generated/LightController/*.anim` 5 个被改写（bake 时重生成） | 从原始包目录拷回并核对 MD5（本工作区 5 个 MD5 见 `SKILL.md` 症状速查「亮度插件」行） |
| 工程根重新生成 `*.sln` / `*.csproj` | 收尾全删（`Get-ChildItem -Filter *.csproj -Recurse \| Remove-Item`，`.sln` 同） |
| `Assets/Build AssetBundles.buildreport` 等构建产物 | 删（本次实测导入后即消失） |
| `Assets/Screenshots/*` 截图、临时相机/灯对象 | 移出 `Assets`（到 `_asset_reorg/evidence/`）或删 |
| 场景 blueprintId 变化 | 保存场景；记下 `avtr_` id 便于后续更新 |
| `Packages/com.vrcfury.temp` | **绝对不要删**（删了之后每次 Play 烘焙都会失败，见 `07-验证与排查/07-01-bake-and-validation.md` §13） |

---

## 6. 安全设置/告警怎么读（本次 68 条 alert 的结论）

- 三条红色 alert 中两条是**阈值告警**（Actions & Moods 图片太大 / 贴图没开 Streaming Mip Maps），**不阻断上传**。
- `VRCExpressionsMenu uses a parameter that is not defined`：点面板 Select 会 `Selection.activeObject` 跳到出问题的菜单资产。
  本工作区定位到第三方 ABT 菜单 `Assets/09_工具与插件/abt/AvatarBasicTool/Expression/SubMenu/Clairvoyance.asset` 里一个 **parameter 为空的 RadialPuppet**（`大小[VR专用]`）→ 既有第三方瑕疵，
  不是我们菜单的问题。
- 自己头像的参数审计应另做：从 descriptor 的 `expressionsMenu` 递归遍历可达菜单、收集所有 `Control.parameter.name`，与 EP 声明 + MA Parameters 声明求差集，**差集必须为 0**（本工作区：26 个可达菜单、44 个使用参数、声明 137 个、未声明 0）。

---

## 7. 边界

- 本册是「用 SDK 公开 API 完成上传」的工程手段；**上传本身仍是用户授权行为**（Copyright ownership agreement 必须点 OK，本工作区用鼠标点击 (1266,770) 完成）。铁律 3 的原则不变：不要未经用户同意发布。
- `VRCSdkControlPanelAvatarBuilder` 是 SDK 内部面板类，**SDK 升级可能改字段名/方法签名** → 用前先 grep 实际源码，不要硬编码。
- 上传成功 ≠ 游戏内正确：多人同步 / 远端可见性 / 安全设置屏蔽动画下的 Rest Pose 仍需进游戏验证（NOT_RUN）。

---

## 8. 二次出包与构建排障（同轮追加，2026-10-27）

> 场景：首次 Publish 成功后，为了修一个疑似 bug 要**重新出包**，过程中踩到 4 个坑。以下四条都按「症状 → 根因 → 做法」记录。

### 8.1 `BuilderException: Open the SDK panel to build and upload avatars`

- 根因：`VRCSdkControlPanelAvatarBuilder` 的 `protected VRCSdkControlPanel _builder` 是 **Unity 假 null** —— 字段里还留着对象引用（`object != null` 为真），
  但它指向的 `VRCSdkControlPanel` 窗口已经被销毁（`as UnityEngine.Object == null` 为真）。源码 `VRCSdkControlPanelAvatarBuilder.cs:2741` 的守卫 `if (_builder == null)` 因此命中。
- 做法：**先执行菜单 `VRChat SDK/Show Control Panel`**（`execute_menu_item`），面板打开时会 `RegisterBuilder(VRCSdkControlPanel)` 重新注册活对象；再用反射取 `_instance` 调 `Build` 即可。
- 判据：反射读 `_builder` 时用 `((fB.GetValue(inst) as UnityEngine.Object) == null)`，不要用 `obj != null`。

### 8.2 类型解析与 `ActiveBuildType`

- `VRCSdkControlPanelAvatarBuilder` 在程序集 **`VRC.SDK3A.Editor`**（不是 `VRCSDK3A-Editor`）；`VRC_SdkBuilder` 在 **`VRCSDKBase-Editor`**。
- `VRC.SDKBase.Editor.VRC_SdkBuilder.ActiveBuildType` 是**属性**（`VRC_SDKBuilder+BuildType`），不是字段 → 必须
  `GetProperty("ActiveBuildType").SetValue(null, Enum.Parse(enumType, "Publish"), null);`
- 出包入口是 public 的 `Task<string> Build(GameObject target)`（行号随 SDK 版本浮动，用「名字 = Build 且参数数 = 1」筛选最稳），返回的 Task 用 `ContinueWith` 把结果写文件（不要在主线程 `.Wait()`）。

### 8.3 构建报 `The object of type 'VRCRotationConstraint' has been destroyed...`

- 症状：Play 试跑过之后再点构建，稳定 `FAULT=BuilderException:The object of type 'VRCRotationConstraint' has been destroyed but you are still trying to access it.`（换 `VRCScaleConstraint`/`VRCPositionConstraint` 同理）。
- 根因：退出 Play 后 **VRC Dynamics 的约束管理器（`VRC.Dynamics.VRCConstraintManager` / `VRCDynamicsScheduler`，注册在 PlayerLoop）仍持有已销毁的约束对象**，
  之后每帧抛 `MissingReferenceException`；构建期这个异常被 SDK 的 `HandleBuildError` 捕获并包装成 `BuilderException`。清 Console、重开场景都无效。
- 做法：**重启编辑器**。`SendMessageW(hwnd, WM_CLOSE)` 优雅关闭 → 运行 `tools/scripts/unity_launch_one.ps1`（会自动按 Enter 过 `Unity is running as administrator.` 模态框）
  → 约 45 s 等 6400 端口恢复 → `set_active_instance("<Name@hash>")`。重启后构建一次通过。
- 判据：重启后 Console 不再刷 `MissingReferenceException ... VRCRotationConstraint`。

### 8.4 烘焙后参数驱动层 `defaultWeight = 0` —— 不是 bug，别为此重传

- 现象：`Packages/com.vrcfury.temp/Builds/____/VRCFury FX.controller` 里，**只含 Parameter Driver、没有可解析 clip 的层**（本例 `OutfitOneClick`、`Cat Ear Control`、
  `LipSync Control`、`LightController/Reset`，以及因目标对象被删而 clip 全部失效的 `[ USER EDIT ] Face`）`defaultWeight == 0`；而源控制器里它们是 1。
- 根因：VRCFury `Editor/VF/Service/CleanupEmptyLayersService.cs` 的收尾逻辑 ——
  ```
  if (!hasNonEmptyClip && !hasBehaviour) { 删层或加警告 state }
  else if (!hasNonEmptyClip) { layer.weight = 0; }
  ```
  即「有 behaviour 但没有有效 clip」的层被显式置 0（行为层的位移/曲线本来也没有意义）。
- **VRChat 运行时，层权重 0 的 State Behaviour 仍然执行**（VRC School《VRC State Behaviours》明确写："Contrary to what Unity's documentation says:
  State Behaviours still run even when the layer's weight is set to 0."），所以 `VRCAvatarParameterDriver` 驱动的换装/开关在游戏内照常工作。
- 因此：看到 0 权重先核 ① states / AnyState 转移 / driver 条目是否完整（本例 10 states、9 条 AnyState、142~143 条 Set 全在）② 烘焙菜单 / EP 默认值是否正确；**不要去改烘焙产物，更不要为了这个重新上传**。
- 反例（不要照着做的验证）：编辑器 Play 不是 VRChat 运行时 —— `VRCAvatarParameterDriver.initialized = False`，驱动不执行；
  `Animator.SetLayerWeight` 也会被 VRCFury 的 `FixAnimatorPreviewBreakingInPlayModeHook` 每帧回写默认权重，改不动。可验的是**clip 驱动**的 `MA Responsive:` 层（换帽子/发色能立刻生效），
  驱动层只能验「状态机是否进了目标 state」。

### 8.5 `.vrca` 里查不到 animator 控制器

- `.vrca` 打开后 `GetAllAssetNames()` 只有 `assets/prefab-id-v1_avtr_<id>_<hash>_prefab`，`LoadAllAssets()` 也只有该 GameObject，
  其 4 个 Animator 的 `runtimeAnimatorController == null`（控制器由 VRChat 运行时按 avatar record 装配）。
- 想核层权重/参数只能用烘焙中间产物：`VRCFury FX.controller` / `VRCFury Params.asset` / `VRCFury Menu.asset`；不要试图从 `.vrca` 反查。

### 8.6 二次上传（改了包要覆盖同一个 avatar）

- 首次上传：`VRCApi.CreateNewAvatar(id, data, bundlePath, imagePath, progress, token)`（要求 `PendingUpload == true`）。
- 之后覆盖：`VRCApi.UpdateAvatarBundle(id, avatar, pathToBundle, progress, token)`（内部按 `GetLatestAssetUrlForPlatform` 决定新建/覆盖 fileId，随后 `Put avatars/<id>`），图片另走 `UpdateAvatarImage`。
- 本次实测：首次 `CreateNewAvatar` 成功后 `Version=2`、`UnityPackages=1`、`ThumbnailImageUrl` 生效；后续因查明「权重 0 不是 bug」而**没有重传**，线上版本行为与本地一致。

### 8.7 数字参考（本机）

| 项目 | 值 |
|---|---|
| 首次出包（shader 变体未缓存） | 约 6 分钟，进度框 `Compiling shader variants (busy for 09:16...)` / `Hidden/SPSPatched/... 85 of 96` |
| 二次出包（已缓存） | 91 s |
| bundle 输出 | `%TEMP%\DefaultCompany\LuzzyMeow V2\<guid>.vrca` |
| 构建期间 MCP | `read_console` 会超时（主线程被占），属正常，不要中断/重启 |

---

## 9. 二次修复：新建菜单资产「只在内存生效」+ 覆盖上传（同轮追加）

### 9.1 `VRCExpressionsMenu` 的 `subMenu` 引用必须落盘，否则重启/域重载后变空

- 症状：烘焙出来一切正常，但**重新打开工程/重启编辑器后**，菜单里两个一级项变成「空 SubMenu」（点进去没有子项）；`VRChat SDK` 面板不报错，`MA`/`VRCFury` 也不报错。
- 根因：用脚本 `AssetDatabase.CreateAsset` 新建菜单资产后，若只改内存里的 `controls[i].subMenu` 而没有 `EditorUtility.SetDirty(asset)` + `AssetDatabase.SaveAssets()`，
  **引用不会被序列化进 `.asset`**；当时烘焙读的是内存对象所以正确，重启后读磁盘就是 `subMenu: {fileID: 0}`。本次三处中招：`狼_根.asset`（装扮菜单/功能菜单）、`狼_装扮.asset`（饰品/功能道具/外观调节）、`狼_外观调节.asset`（发色调节）。
- 判据（不用等重启）：直接读 `Assets/.../*.asset` 的 YAML，看 `subMenu:` 是不是 `{fileID: 0}`；或用脚本递归遍历描述符菜单，遇到 `ControlType.SubMenu && subMenu == null` 计数（应为 0）。
- 修法：`var cs = menu.controls; cs[i].subMenu = target; menu.controls = cs; EditorUtility.SetDirty(menu); AssetDatabase.SaveAssets();` ——
  注意 `controls` 是 `List<Control>`、`Control` 是 **struct**，必须「取出整行 → 改 → 写回列表」，直接改 `menu.controls[i].subMenu` 会改到副本上，静默无效果。
- 验收：修完递归遍历应得到 `menus=21 / controls=87 / nullSubIssues=0`，并且**烘焙后**再验一遍 `Packages/com.vrcfury.temp/Builds/____/VRCFury Menu.asset`（本次 
  `menus=30 / controls=165 / issues=0`，8 套衣服 56 个分件开关、饰品 3 项、功能道具 6 项、亮度调节 5 项、发色 8 色全在）。

### 9.2 反射找 `Build` 方法要用「枚举筛选」，不能用 `GetMethod("Build", flags)`

- `VRCSdkControlPanelAvatarBuilder` 里有多个 `Build` 重载 → `Type.GetMethod("Build", BindingFlags.Instance|Public|NonPublic)` 直接抛
  `AmbiguousMatchException: Ambiguous match found.`（`at MCPDynamicCode.Execute ()`）。
- 正确做法：
  ```
  var ms = tBuilder.GetMethods(BindingFlags.Instance|Public|NonPublic);
  MethodInfo mi = null;
  for (var i=0;i<ms.Length;i++){
    if (ms[i].Name != "Build") continue;
    var ps = ms[i].GetParameters();
    if (ps.Length == 1 && ps[0].ParameterType == typeof(GameObject)) { mi = ms[i]; break; }
  }
  ```
  目标签名是 `Task<string> Build(GameObject)`。

### 9.3 覆盖已发布头像（同一 avatar 换包）

- `VRCApi.UpdateAvatarBundle(string id, VRCAvatar data, string pathToBundle, Action<string,float> onProgress, CancellationToken cancellationToken)`：
  传 `id` + 一个「只填 ID/Name/Description/Tags/ReleaseStatus」的 `VRCAvatar`（struct，用 `Activator.CreateInstance` + 反射属性）
  + 新 `.vrca` 绝对路径 + `null` + `CancellationToken.None`。
- 实测结果：`Version 2 → 3`、`UnityPackages` 仍为 **1**（替换既有包，不是新增）、`AssetUrl` 换成新 file id、`ImageUrl`/`ThumbnailImageUrl` 不变、`PendingUpload=False`。
- 判断「到底传上去没有」的金标准：`GetAvatar(id, forceRefresh:true, ct)` 里的 `Version` 递增 + `UnityPackages[0].AssetUrl` 变化 + `pkg.CreatedAt` 是刚才。
- 每次 `Build()` 都会把 `Assets/08_动画·菜单·图标/NyaPPu__Generated/LightController/*.anim` 重写（体积变大），并重新生成工程根 `.sln/.csproj` → **收尾照旧还原 5 个 anim（MD5 对齐 `$tmp\Lilleo2.1`）并删 `.sln/*.csproj`**。



## 10. Requested value 'security' was not found（SDK 面板 Build & Publish 直接失败）

- 症状（2026-11-05 实测，沐梓溪狼 v53→ 面板点 Build & Publish）：底部弹窗
  `Your avatar failed to build or upload because of the following reason: Requested value 'security' was not found.`；
  面板其它一切正常（名字/缩略图/告警都正常、按钮可点）。
- 根因：服务端头像记录 `UnityPackages` 里会出现 `variant="security"` 的安全扫描包（本机 7 条包：2×security、2×standard、3×impostor），
  而 `Packages/com.vrchat.base/Editor/VRCSDK/Dependencies/VRChat/API/VRCAvatar.cs` 的
  `public enum AvatarVariant { Standard, Impostor }` 没有 security，
  第 55 行 `GetLatestAssetUrlForPlatform()` 第 67 行 `(AvatarVariant)Enum.Parse(typeof(AvatarVariant), unityPackage.Variant, true)` 直接抛
  `ArgumentException`。SDK 面板的上传流程在 `VRCApi.UpdateAvatarBundle` 第一步（`VRCApi.cs:629`）就调这个方法 → 整个 Build & Publish 失败；
  world 侧 `VRCApi.cs:488` 同样会中招。
- 判据/复现：反射脚本 `GetAvatar(bp, true)` 打印每条包的 platform/variant/assetUrl，能看到 `variant=security`；
  再调 `GetLatestAssetUrlForPlatform("standalonewindows")`，打补丁前抛 `LATEST_THROW ArgumentException: Requested value 'security' was not found`，
  打补丁后返回 `.../file/<id>/2/file`（standard 包，跳过 security 与 impostor）。
- 正确做法（本机本地补丁，2026-11-05）：把 `VRCAvatar.cs:67` 的 Enum.Parse 改成「只接受 standard，未知 variant 跳过」：
  `csharp
  if (unityPackage.Variant != null)
  {
      // 服务端会返回 variant="security" 的扫描包；AvatarVariant 枚举没有它，Enum.Parse 会抛错
      if (!string.Equals(unityPackage.Variant, "standard", StringComparison.OrdinalIgnoreCase)) continue;
  }
  `
  （`using System;` 已有；改的是工程内嵌包 `Packages/com.vrchat.base/...`，**SDK 升级会被覆盖**，升级后要重打。）
- 关联坑 1：自研上传脚本只要在 `UpdateAvatarBundle` 之前把 `av.UnityPackages` 里的 security 包剔掉（或干脆只处理 standard），就不会触发这个 bug —— 本机
  `Assets/Editor/MeowUpload.cs` 的 `PKG drop variant=security platform=standalonewindows ×2` 就这么绕过的（所以同一时刻脚本上传 v53 成功、面板点却失败）。
- 关联坑 2：面板 Build 按钮灰、显示「Please set a name and thumbnail before uploading」是**另一个**问题（内容信息加载失败 → `_uiEnabled=false`），
  修法是反射调 `VRCSdkControlPanelAvatarBuilder.SelectAvatar(GameObject)` 重跑内容加载；两个问题的共同触发点都是 `VRCApi.GetAvatar` 的异常被 SDK 内部 catch 掉。
- 关联坑 2 补充（2026-11-05 根因闭环）：内容信息加载失败的报错是 `Using a web-derived token in client`（`VRCApi.GetAvatar` 在 `VRCSdkControlPanelAvatarBuilder.HandleAvatarSwitch` 里被 catch 后只 `LogError`，然后接着 NRE），说明当前 Unity 里登录的 token 是 **web/cookie 派生**的，`/avatars/{id}` 这类 client 接口会被服务端拒绝。临时绕过 = 反射改 `VRCApi.Headers`（去掉 `X-SDK-Version/X-Platform/X-Unity-Version/X-MacAddress`、换 `User-Agent`）后再 `SelectAvatar(GameObject)`；本机实测 `_uiEnabled=True`、`_avatarData.Version=54`。但改 headers 只是内存态、且会让 SDK 面板的登录态失效（`APIUser.IsLoggedIn=False` → Builder 标签灰掉、`Build & Publish` 点不了），**根本解法是在 SDK 面板 Authentication 页做 client 登录**（重新登录一次），不要长期挂着 header 补丁。
- 关联文件：`Packages/com.vrchat.base/Editor/VRCSDK/Dependencies/VRChat/API/VRCAvatar.cs`、`.../VRCApi.cs`、
  `Packages/com.vrchat.avatars/Editor/VRCSDK/SDK3A/VRCSdkControlPanelAvatarBuilder.cs`（L1584/L1616/L1650/L2584）。
