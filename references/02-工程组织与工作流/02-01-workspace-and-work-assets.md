# 工作资产：目录约定、基础头像选型与衣服接线

读这份文件的时机：要为一次改模建工作场景/预制体、把衣服挂到头像上、或要把衣服的动画控制器合并进基础 FX 时。

> 原则：**源资产不动，只在明确命名的工作目录里生成工作资产。** 生成出来的菜单、prefab、场景与编辑器脚本都要能被下一次任务找到，不要散落在工程根上。

## 1. 目录约定

```text
Assets/<WorkRoot>/
├── Scenes/<avatar>_<outfit>.unity        # 工作场景（交付态）
├── Scenes/<avatar>_<outfit>_Test.unity   # 测试场景（预览/开关逐个试）
├── Prefabs/<avatar>_<outfit>.prefab      # 可复用工作 prefab（交付物）
├── Expressions/  或 ExpressionsFixed/     # 克隆/生成的菜单资产
└── ...                                    # 该次任务特有的其它中间资产
Assets/Editor/                             # 编辑器脚本（工具脚本）
<PROJECT_ROOT>/Logs/                       # 报告、审计输出、完成标记
```

`<WorkRoot>` 用仓库已有约定；没有就用 `Assets/VRCWork` 或 `Assets/AvatarWork`，**一次任务内保持一致**。厂商原始资产目录**不要**用来放生成物。

## 2. 实例化「正确」的基础头像

这是最容易做错、也最难回头修的一步。

- **优先用原始完整头像 prefab**——你要保留原衣服与身体开关时必须用它。
- **避免**厂商提供的 `kisekae` / `naked` / `test` / `clothing demo` 这类 prefab：它们会缺原衣服对象，或用一个精简 FX/菜单，症状表现为「原衣服消失」「原开关点了没反应」。
- 确认头像根上有 `VRCAvatarDescriptor`、Humanoid Animator，以及**一个明确的骨架根**。骨架根的实际名字要读出来（常见是 `Armature`，但不同作者会改）。
- 需要新头像容器（例如想从零挂自己的菜单）时：清掉生成的工作场景/prefab 上的 `blueprintId`，并在**打开 SDK 上传面板之前再检查一次**序列化内容里的 `blueprintId`（见 [06-01-sdk-upload-and-versioning.md](../06-构建上传与备份/06-01-sdk-upload-and-versioning.md)）。

## 3. 挂衣服

1. 衣服作为头像根的**子对象**实例化。
2. **本地变换清零**：位置 0、旋转 identity、缩放 1。不做这一步，后面所有贴合判断都不可信。
3. 对衣服下每个 `ModularAvatarMergeArmature`，把 merge target 设为**头像的骨架对象**（要读出来，不要硬编码名字）。
4. 如果衣服**没有** MergeArmature、但已经适配过且骨骼名匹配：**优先在衣服骨架根加组件**，而不是手动重挂骨骼。手挂骨骼会丢失 bindpose 与权重索引的对应关系。
5. 骨骼归属期望：
   - **身体衣服**的 SkinnedMeshRenderer 骨骼，在 NDMF 处理后应属于头像骨架；
   - **饰品/武器/道具**可以保留自身 local 骨骼（有意自包含）；
   - 身体衣服 renderer 出现 `missing` 骨骼 = 必须排查后才能交付（见 [07-01-bake-and-validation.md](../07-验证与排查/07-01-bake-and-validation.md)）。
6. 衣服顶层常常只是个 Transform 容器：`MergeArmature` / `MergeAnimator` / `MenuInstaller` / `Parameters` / `PhysBone` / `BoneProxy` / `SkinnedMeshRenderer` 往往在**子对象**上。判断「这件衣服没挂组件」之前先递归查一遍。

### 3.1 可选 prefab 不在主 prefab 里的情况

有些资源包把可选的 promo/效果 prefab 放在别处，只由 FX clip 的**相对路径**引用，而没有包含在主 prefab 中。表现是绑定路径解析不到。处理办法：

1. 找到对应的可选 prefab；
2. 挂到 merge animator 的**相对根**下，使相对路径成立；
3. 按控制参数 / clip 设置它的**源默认激活态**；
4. **重跑绑定审计**（不要靠看一眼认为对了）。

## 4. 合并动画控制器：合并不等于替换

- **保留基础头像 FX** 作为 descriptor 的 FX 层——原头像的开关（原衣服/头发/身体/尾巴/耳朵/胸围/脸）靠它。
- 衣服 FX 通过 MA / NDMF **合并**进来；**不要**用衣服自带的 test controller 替换基础 FX。这是「原衣服消失」的经典成因。
- 多个 merge animator 时保留它们的**顺序**与 **write defaults** 设置，除非有实证冲突（有证据说明某个顺序/设置导致具体问题）。
- 需要「始终激活」的独立功能层，通常 `defaultWeight = 1`；改已有层的权重前先定位证据并取得授权，不要为了消警告盲改。
- 合并后的真实结果以 NDMF 烘焙后的头像为准（见 [07-01-bake-and-validation.md](../07-验证与排查/07-01-bake-and-validation.md)）。

## 5. 编辑器脚本范式（[需 MCP] 或手工执行）

本机没有 Unity 时，这些范式以「可复用的编辑器工具脚本」形式保存到 `Assets/Editor/`，等编辑器可用时再执行。

### 5.1 基本形状

```csharp
// 放到 Assets/<WorkRoot>/../Editor/ 下；输出写到 Logs/
public static class AvatarWorkUtility
{
    [MenuItem("Tools/Avatar Work/Rebuild Workspace")]
    public static void RebuildWorkspace()
    {
        var log = new StringBuilder();
        try { /* 重建工作资产，过程中把发现写进 log */ }
        finally
        {
            File.WriteAllText("Logs/avatar-work-report.txt", log.ToString(), Encoding.UTF8);
        }
    }
}
```

要点：

- 报告写成文件（`Logs/` 下），**不要** `Debug.Log(report.ToString())` 整份数百行报告——某些环境在关机期的 console 日志会把一次成功运行变成表面崩溃。
- 长任务可以用标记文件驱动：

```csharp
[InitializeOnLoadMethod]
private static void AutoRunWhenRequested()
{
    const string marker = "Temp/run_avatar_work";   // 也可以用别的明确标记
    if (!File.Exists(marker)) return;
    if (EditorApplication.isPlaying) { EditorApplication.isPlaying = false; return; }
    File.Delete(marker);
    EditorApplication.delayCall += RebuildWorkspace;
}
```

### 5.2 配 descriptor

```csharp
private static void ApplyDescriptor(VRCAvatarDescriptor descriptor, VRCExpressionsMenu menu,
                                     VRCExpressionParameters parameters, RuntimeAnimatorController fx)
{
    descriptor.expressionsMenu = menu;
    descriptor.expressionParameters = parameters;
    for (int i = 0; i < descriptor.baseAnimationLayers.Length; i++)
    {
        var layer = descriptor.baseAnimationLayers[i];
        if (layer.type != VRCAvatarDescriptor.AnimLayerType.FX) continue;
        layer.isDefault = false;
        layer.animatorController = fx;
        descriptor.baseAnimationLayers[i] = layer;
    }
    EditorUtility.SetDirty(descriptor);
}
```

注意 `baseAnimationLayers` 是**结构体数组**：改了 `layer` 之后必须**写回数组下标**，否则修改丢失。

### 5.3 清 blueprintId（要新建头像容器时）

```csharp
private static void ClearBlueprintId(GameObject avatarRoot)
{
    foreach (var component in avatarRoot.GetComponents<Component>())
    {
        if (component == null || component.GetType().Name != "PipelineManager") continue;
        var field = component.GetType().GetField("blueprintId");
        if (field == null) continue;
        var value = field.GetValue(component) as string;
        if (!string.IsNullOrEmpty(value))
        {
            field.SetValue(component, string.Empty);
            EditorUtility.SetDirty(component);
        }
    }
}
```

**要在打开 SDK 上传面板之前再确认一次**：清掉之后不要顺手保存面板里的旧 ID。

### 5.4 配衣服：merge target 与菜单 installer

```csharp
private static void ConfigureClothing(Transform avatarRoot, Transform clothingRoot, VRCExpressionsMenu outfitMenu)
{
    var armature = avatarRoot.Find("Armature");          // 名字要按工程实际读，不要硬编码
    if (armature == null) throw new InvalidOperationException("Avatar Armature not found.");

    foreach (var merge in clothingRoot.GetComponentsInChildren<ModularAvatarMergeArmature>(true))
        merge.mergeTarget.Set(armature.gameObject);

    foreach (var installer in clothingRoot.GetComponentsInChildren<ModularAvatarMenuInstaller>(true))
    {
        installer.menuToAppend = outfitMenu;
        installer.installTargetMenu = null;              // 挂到根菜单
    }
}
```

### 5.5 克隆菜单树（不要就地改厂商菜单资产）

```csharp
private static VRCExpressionsMenu CloneMenuTree(VRCExpressionsMenu source, string outputDir,
        Func<string, string> labelTransform, Dictionary<VRCExpressionsMenu, VRCExpressionsMenu> map)
{
    // Object.Instantiate 克隆 → AssetDatabase.GenerateUniqueAssetPath(outputDir + "/" + clone.name + ".asset")
    // → CreateAsset → 递归克隆子菜单；非 SubMenu 控件的 control.subMenu 置 null
}

private static VRCExpressionsMenu CreateWrapper(string label, VRCExpressionsMenu child, string path)
{
    // ScriptableObject.CreateInstance<VRCExpressionsMenu>()，命名 label + "入口"
    // 加一个 ControlType.SubMenu 控件（其 parameter.name 留空），再 CreateAsset
}
```

克隆时**精确保留参数名**；翻译只翻标签，**不翻参数名、不翻资产路径**。复制来的菜单如果出现失效的子菜单引用，从**非 SubMenu** 控件里移除该引用。

## 6. 交付前工作资产自检

- [ ] 工作场景与测试场景都已保存，且 `isDirty = false`；
- [ ] 可复用工作 prefab 已保存；
- [ ] 源包与源资产**未被修改**；
- [ ] 工作资产下的菜单与参数已合并（或明确记录为 BUILD_REQUIRED）；
- [ ] 没有残留的临时 helper 对象、没有新增 Missing 脚本；
- [ ] 保存过的工作 prefab 里没有重复堆叠的 merge animator（反复修复常见产物）；
- [ ] 报告已写到 `Logs/`。

## 7. 删资产前的 GUID 引用扫描（2026-10-21 实测）

- **症状**：`Assets/AvatarFeatures/WardrobePieces/` 下 17 个菜单 `.asset` 看起来像旧衣柜残留（名字都是 `Outfit_00..13`、`AccessoryCandy`），差点按「孤儿」删掉。
- **正确做法（四步，缺一不可）**：
  1. 对每个候选取 `AssetDatabase.AssetPathToGUID(path)`；
  2. 全 `Assets` 文本扫描 `guid:<GUID>`（`.prefab` / `.asset` / `.unity` 的 YAML 里都是这个格式）；
  3. `AssetDatabase.GetDependencies(path, true)` 反查，以及反向查「谁引用了它」；
  4. 确认引用者身份后再决定删 / 留。
- **本例结论**：`Outfit_*` → `Assets/MOCHIYAMA/Kipfel/ExMenu/GeneratedEnglish/Avatar_Wardrobe_English_V2.asset` 等英文菜单群 → 再被 
  `Assets/KipfelShare/Kipfel_Kisekae_English.prefab`（英文变体 prefab）引用；`Kipfel_AvatarWardrobe_Parameters.asset`（95 参数）被中 / 英两个 prefab 引用 → **它们是双语结构的一部分，
  不是死资产，保留**。
- **确是 0 引用才删**：本例 48 个 `Assets/KipfelShare/Menus_Chinese/Menu_001..048.asset` 扫描确认 0 引用后才删。
- **判据**：文本 guid 扫描 + `GetDependencies` 双重确认 0 引用；删后 `AssetDatabase.Refresh()` → Console 0 error → 场景 / 菜单 0 missing。
