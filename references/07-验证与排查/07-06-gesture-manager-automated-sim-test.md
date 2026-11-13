# Gesture Manager 自动化模拟测试（反射驱动 + 两遍法）

> 本工作区实测 2026-10-20（第 55 轮，LuzzyMeow / Kipfel Kisekae 改模后全量测试）
> 适用于铁律 5「Gesture Manager 必装必用」的落地执行：当菜单有 200+ 项时，人工点不完，用本文的 harness 自动跑。

## 0. 结论先行

- Gesture Manager（GM）3.9.9 的运行时模块可以**纯反射**创建，不需要点 UI；参数可用 `Vrc3Param` 的 `Set/IntValue/BoolValue/FloatValue` 读写。
- 配一个 `EditorApplication.update` 逐帧状态机就能在 play 内自动遍历全部菜单：**设参 → 等帧 → 读回 → 截图 → 恢复原值**。
- 单遍「set-to-value」不够：默认态已等于菜单值的 Toggle 看不出变化 → 必须加**第二遍翻转验证**（每项翻到相反状态再翻回）。
- 实测规模：234 项菜单 439 步 / 124 s / 521 张截图；165 项翻转 331 步 / 92 s / 444 张截图；21 个 Button 走脉冲判定。
- **最容易踩的坑：每项测完必须恢复「测试前读到的原值」，不能归 0** —— `/外观/亮度` 的 Radial 参数「发光」默认 0.503937，归 0 会让头像全黑（表现为「后半段截图全黑」，很容易误判成模型坏了）。

## 1. 前置状态

1. 工程已打开、Console 0 error；`Packages/vpm-manifest.json` 有 `vrchat.blackstartx.gesture-manager`（3.9.9）。
2. 场景里已有 `Gesture Manager` 对象（`Tools → Gesture Manager Emulator` 生成），或 harness 里自己 `new GameObject` 加组件。
3. 进 Play 模式后 NDMF 会烘焙，`descriptor.expressionsMenu` 变成 `Packages/nadena.dev.ndmf/__Generated/<avatar>(Clone)/_assets/...` 下的生成资产 —— **必须递归读烘焙后的菜单**，不是工程里的源资产。

## 2. 关键反射 API（asm 名 / 全名）

- 运行时组件：`BlackStartX.GestureManager.GestureManager`（asm `vrchat.blackstartx.gesture-manager`）
- 编辑器模块：`BlackStartX.GestureManager.Editor.Modules.Vrc3.ModuleVrc3`（asm `vrchat.blackstartx.gesture-manager.editor`）
- 设置：`BlackStartX.GestureManager.Modules.ModuleSettings`（运行时 asm）

创建模块（反射，play 内）：

```csharp
// 1) 场景里找 GM 组件（或 new GameObject().AddComponent<T>()）
var gmComp = UnityEngine.Object.FindObjectOfType(gmType); // gmType = typeof 反射取得
// 2) 建 ModuleSettings 并填字段
var setType = /* BlackStartX.GestureManager.Modules.ModuleSettings */;
var settings = System.Activator.CreateInstance(setType);
setType.GetField("favourite").SetValue(settings, desc);        // VRC_AvatarDescriptor
setType.GetField("simulateCulling").SetValue(settings, false);
setType.GetField("loadStored").SetValue(settings, false);
// 其余字段：initialPose、userIndex、cullingDistance、isOnFriendsList、isRemote、vrMode
// 3) 建 ModuleVrc3(desc, null)，设 Settings 字段
var modType = /* ...ModuleVrc3 */;
var module = System.Activator.CreateInstance(modType, new object[]{ desc, null });
modType.GetField("Settings", /* 或 property */).SetValue(module, settings);
// 4) GM.SetModule(module)
gmType.GetMethod("SetModule").Invoke(gmComp, new object[]{ module });
// 之后 module.Active == true；module.Params 是 524 项参数字典
```

参数读写（`Vrc3Param`）：

```csharp
var p = module.Params["1"]; // 或 module.GetParam("1")
p.GetType().GetMethod("IntValue").Invoke(p, null);
p.GetType().GetMethod("BoolValue").Invoke(p, null);
p.GetType().GetMethod("FloatValue").Invoke(p, null);
// Set 有三个重载，按 AnimatorControllerParameterType 选：
var setInt = p.GetType().GetMethod("Set", new System.Type[]{ modType, typeof(int), typeof(object) });
setInt.Invoke(p, new object[]{ module, 1, null });
var setBool = p.GetType().GetMethod("Set", new System.Type[]{ modType, typeof(bool), typeof(object) });
var setFloat = p.GetType().GetMethod("Set", new System.Type[]{ modType, typeof(float), typeof(object) });
```

> `Vrc3Param` 字段还有 `Type`(AnimatorControllerParameterType)、`Sync`、`Name`、`LastUpdate`、`_value`、`_onChange`；方法还有 `SetOnChange(Action<,>)`、`SyncValue()`、`Add()`、`Copy()`、`Random()`。

## 3. Harness 骨架（EditorApplication.update 状态机）

- 写一个 `[MenuItem("Tools/Kisekae Sim/Post-Mod Test")]` 的 Editor 脚本；用静态闭包状态机挂在 `EditorApplication.update` 上，每个 update 只推进一步。
- 注册必须用 `EditorApplication.update += new UnityEditor.EditorApplication.CallbackFunction(Tick)`（CodeDom 里 `+= Action` 不行）。
- 状态机阶段：等烘焙（找 `expressionsMenu != null` 且 5 个动画层都指向真实 controller 的 descriptor）→ 建 GM 模块 → 建 `__SimCam`/`__SimLight` → 递归菜单建项 → 逐项测试 → 专项（头发/音乐）→ 完整性扫描 → 退出。
- 截图：自建 Camera（`clearFlags=SolidColor`、bg=(0.18,0.18,0.22)、near=0.02）+ Directional Light（intensity=1.2），`RenderTexture` + `ReadPixels` + `EncodeToPNG` 写文件；**不要用 `manage_camera` 的 scene_view**（旧视口假图）。
- 全身取景：`cam.pos = root.position + root.forward*1.62 + up*0.46`，LookAt (0,0.46,0)，fov 32，480×600（覆盖率 ≈37–39%）。
- 头像朝 +Z、相机放 +Z 侧；模型多数 SMR 的 bounds 是 2.83×3.01 的**共享大包围盒**，不能用来取景。

## 4. 两遍法（关键）

### 第一遍：set-to-value

- Toggle：设成菜单声明的 value（通常 1）；Int：设成菜单 value；Radial：设成菜单 value。
- 等 6–10 帧 → 读回参数（等于期望值即 PASS）→ 截图（全身；头部/表情类另拍 head 特写）。
- **测完恢复测试前读到的原值**（不是 0！见 §6）。
- SubMenu：只判 `subControls>0`，不设参不截图。
- Button：`VRCEmote` / `WardrobePresetCommand` / `MusicCommand` 这类命令参数由 FX 消费后**自动归 0**，读回 0 不是失败 → 用「点击后连续 8 帧读脉冲值 + before/after 截图 + 功能副作用」判定。

### 第二遍：翻转验证（补齐默认态无变化的项）

- 只测 Toggle/RadialPuppet 且有参数名的项；已取得强视觉证据的项可跳过（记录跳过清单）。
- 翻转规则：Bool → `!cur`；Int → 菜单 value≠0 时 `cur==0?value:0`，value==0 时 `cur==0?1:0`；Radial → `cur>0.5?0:1`。
- before 截图 → 设参 → 等 8 帧 → 读回 + after 截图 → **恢复原值** → 等 3 帧。
- 结果行示例：`FLIP|idx|path|type|prm|orig=|target=|got=|PASS`。

## 5. 图像分析判据

- Python + Pillow：mean / std / 覆盖率（非背景像素占比）/ 粉材质像素占比；前后差异用 `ImageChops.difference + ImageStat`（resize 240×300），并算「变化像素占比」（diff>14 的比例）。
- 判据：参数读回 PASS 是硬判据；截图用于确认「无粉 / 无 missing / 无 T-pose / 无模型消失」和换装是否可见。
- 变化占比很小的项先别判失败：可能是「音频类配饰 / 细环 / 被头发遮挡 / 需要另一个开关同时打开」→ 查 FX 层与状态名是否真的存在映射，报告里标 **LOW_VISUAL**（参数+动画层证明功能存在，视觉证据不足，需进游戏近景复验）。
- 生成 contact sheet（分区拼图 + before|after 配对拼图）后人工目视复核，不能只看数字。

## 6. 踩坑清单（实测）

1. **归 0 陷阱**：`/外观/亮度` 的「发光」默认 0.503937，测完归 0 → 头像全黑，后续截图全黑。**永远恢复原值。**
2. **预设按钮只恢复命令参数**：点「穿上整套」会把一整套衣服参数都改掉，只恢复 `WardrobePresetCommand=0` 不会恢复衣柜状态；如要干净状态，需记录并恢复一整组参数，或接受状态漂移（报告注明）。
3. **GM 参数默认值与 Animator 默认值可能不同**；同一 play 内前后状态会漂移，baseline 截图要尽早拍。
4. **Button 读回 0 属预期**（见 §4）。
5. **CodeDom C# 6 限制**：不能 `using`；`UnityEngine.Object` 有歧义要写全名；`System.IO.*` 全名；不能引用未定义 helper；lambda 参数名与后面局部变量同名会 `CS0136`；
  `VRCAvatarDescriptor.CustomAnimLayer` 没有 `layerType` 字段；`subParameters` 是数组；`l.animatorController` 是 `RuntimeAnimatorController`，
  要 `as UnityEditor.Animations.AnimatorController` 才有 `.layers`。
6. **`mcp__unity__read_console` 的 `types` 必须是 JSON 字符串列表**（如 `'["error","warning"]'`），传数组或字符串都报 validation error。
7. **`mcp__unity__execute_code` 必须带 `action:"execute"`**；返回取 `structuredContent.content[0].text` 的 JSON → `data.result`。
8. **play 内自建对象随退出 play 消失**，不用手动删；但退出前别把 `__SimCam` 存进场景。
9. **音频可听性、多人同步、saved 持久化、Contact 触发 GM 验证不了** → NOT_RUN，别写 PASS。

## 7. 实测数字（2026-10-20，LuzzyMeow）

- 菜单 **234 项**（动作 9 / 衣柜 110 / 配饰 14 / 外观 75 / 实用功能 26，含 5 个一级 SubMenu 自身）
- 第一遍：pass=213 / fail=0 / noparam=0，Button 21（4 `PASS_PULSE_SEEN` + 17 `PASS_AUTO_RESET`），439 步 / 124 s / 521 图
- 第二遍：165 项 pass=165 / fail=0，331 步 / 92 s / 444 图
- 专项：头发三态 PASS 3；音乐播放器功能 PASS 6（曲目 8 切换 / 播放 / 暂停 / 下一首用 `RK_ON/RK_play/RK_pause/RK_song` + Disc 材质 + AudioSource.time + 碟片旋转判定）
- 删除残留扫描 21 关键词 0 残留、missing script 0、坏 shader 0、残留动画层 0；FX 层 226→191、菜单 257→234、子物体 55→49
- 低视觉证据 4 项：光环 / 音频发夹 / 睡眠眼罩·装饰模式 / 摸摸头·开心（各含动画层/状态映射证据）

## 8. 产物存放

- 报告：`<VRCHAT_ROOT>\projects\LuzzyMeow\KisekaeSimReport-PostMod.md`
- 证据：`<VRCHAT_ROOT>\projects\LuzzyMeow\KisekaePostModSim\`（sim3/、flip/、worknotes/，含 965 张截图与拼图）
- Harness 源码：`<VRCHAT_ROOT>\projects\LuzzyMeow\Assets\Editor\KisekaeSimHarness.cs`、`KisekaeFlipHarness.cs`
- 分析脚本：`KisekaePostModSim/sim3/scripts/analyze_sim3.py`、`analyze2.py`、`flip/scripts/analyze_flip.py`

## 9. 大菜单全量复测 141 项 + play 模式下驱动 harness（2026-10-21 第 58 轮补充）

本节是 §1–§8 的补充：一套菜单精简到 141 项后做**全量复测**时踩到的新坑与可复用协议。

### 9.1 play 模式下 `execute_code` 驱动不了 Editor harness（重要）

- 现象：进 Play 后调 `mcp__unity__execute_code`，任何引用 VRChat SDK 类型或 `Assets/Editor` 脚本的代码都编译失败：
  - `error CS0246: The type or namespace name 'VRCAvatarDescriptor' could not be found`
  - `The name 'KisekaeSimHarness' does not exist in the current context`
- 根因：**Play 域的 CodeDom 临时编译缺少工程 `Assembly-CSharp` / `Assembly-CSharp-Editor` 的程序集引用**（Edit Mode 下正常）。这不是脚本写错，换写法也没用。
- 正确做法：给 harness 加一个菜单入口 `[MenuItem("Tools/Kisekae Sim/Post-Mod Test")]` → `mcp__unity__execute_menu_item({ menu_path: "Tools/Kisekae Sim/Post-Mod Test" })` → 用 pwsh 轮询输出文件直到 `DONE.txt` 出现。
- **参数名是 `menu_path`**，写成 `path` 会缺参报错。

### 9.2 harness 的「文件协议」（可复用）

输出目录自定（本例 `<VRCHAT_ROOT>/tools/tmp/kisekae_sim3/`），最小集合：

| 文件 | 内容 | 判据 |
|---|---|---|
| `DONE.txt` | 完成标记 | 存在即完成 |
| `status.txt` | `DONE\|282/282\|16:54:44`（进度/总数/结束时间） | 与 summary 一致 |
| `summary.txt` | `menuTotal=141 pass=134 fail=0 noparam=0` / `buttons pulseSeen=0 autoReset=7 check=0` / `shots=363` | fail=0、pass+button==menuTotal |
| `scan.txt` | 残留关键词扫描、missing script、坏 shader、参数数 | 关键词全 0 |
| `menu_items.txt` / `report_rows.txt` / `shot_rows.txt` | 全量菜单树 / 逐项结果矩阵 / 截图索引 | 项数与 menuTotal 一致 |
| `shots/` | 逐项前后截图 | 数量 = shots |

### 9.3 判据与结果（本例）

- SubMenu：只判结构（子控件数 > 0）；
- Toggle / RadialPuppet：Set 到菜单值 → 等 8 帧 → 读回参数比对；**`INIT_SAME`（原值=菜单值）算 PASS**；
- Button：脉冲判定（点击后连续 8 帧读到期望值）+ autoReset（FX 消费后回 0 属预期）；
- 本例结果：134 TEST + 7 BTNRESULT = **141/141 全覆盖、fail=0**，duration=32 s、363 图；7 个 autoReset = 4 个「穿上整套」（`WardrobePresetCommand`）+ 音乐「下一首 / 暂停 / 播放」；
- 4 套预设按按钮后的全身截图逐张视觉确认（外观不同、无粉材质 / 无 missing / 无 T-pose）。

## 10. 部件逐帧闪烁：FX 层 ANY 转移条件矛盾（2026-10-21 第 58 轮补充，实测修复）

本节是"模拟测试时才暴露"的一类隐蔽 bug 的完整排查协议：部件在 Play 里**每帧显隐跳变（闪烁）**，Console 零报错。

### 10.1 症状

- Gesture Manager Play / 自动模拟测试时，某个部件（本例头顶猫耳 `Cat_Ear`）高频闪烁；
- Edit Mode 静态看场景完全正常，Console **0 error / 0 warning** —— 只有逐帧采样才暴露。

### 10.2 排查顺序（四步，别跳）

1. **逐帧取证**：临时 Editor 脚本在 `EditorApplication.update` 里按渲染帧采样 `go.activeSelf` + 记录时间戳写 CSV，同时渲染小图。判据是 **flips（相邻帧变化次数）**：40 帧 39 flips、True/False 各 20 + 亮度双值交替 = 每帧翻转实锤。
2. **确认驱动源**：把 Gesture Manager 对象 `SetActive(false)` 后 flips=0，且角色 root 的 Animator 只有 Base 层（`runtimeAnimatorController=null`）→ 排除运行时组件/managed 代码，锁定 **FX 层动画**。
3. **静态 dump FX 层转移**（Edit Mode `execute_code`）：
   ```csharp
   var ctrl = AssetDatabase.LoadAssetAtPath<AnimatorController>(path);
   Debug.Log("layers=" + ctrl.layers.Length);
   foreach (var l in ctrl.layers) {
       if (!l.name.Contains("HideEars")) continue;
       var sm = l.stateMachine;
       Debug.Log("anyCount=" + sm.anyStateTransitions.Length);
       foreach (var t in sm.anyStateTransitions) {
           var sb = "-> " + t.destinationState.name + " | ";
           foreach (var c in t.conditions)
               sb += c.parameter + ":" + c.mode + ":" + c.threshold + " ";
           Debug.Log(sb);
       }
   }
   ```
4. **找矛盾条件组**：本例 HideEars 层 7 条 ANY 里
   - ANY[4] → Hidden，条件只有 `AvatarAccessory_02 IfNot` —— 该 bool 默认 false → **条件恒真**，每帧都在藏耳朵；
   - ANY[5] → Shown，条件 `EarEnabled && !AvatarAccessory_04 && AvatarAccessory_02 && !ClothingCapEnabled` —— 与上条互斥，每帧又放出来；
   - 两条都无过渡时长 → Animator 每帧 ping-pong。

### 10.3 修法（只删一条，guard 保护）

```csharp
// guard：条件必须恰好 1 条且完全匹配预期，否则不动
AnimatorStateTransition victim = null;
foreach (var t in sm.anyStateTransitions) {
    if (t.destinationState == null || t.destinationState.name != "Hidden") continue;
    if (t.conditions.Length != 1) continue;
    var c = t.conditions[0];
    if (c.parameter == "AvatarAccessory_02" &&
        c.mode == AnimatorConditionMode.IfNot && Mathf.Approximately(c.threshold, 0f))
        victim = t;
}
if (victim != null) {
    sm.RemoveAnyStateTransition(victim);
    EditorUtility.SetDirty(ctrl);
    AssetDatabase.SaveAssets();
    AssetDatabase.Refresh();
}
```

- 结果：AnyState 转移 7 → 6；控制器文件 12,175,311 → 12,174,647 B；子资产 4,921 → **4,920**（**transition 自己是子资产**，会跟着少 1）。
- 剩余 6 条互不矛盾：`!EarEnabled→Hidden`、`04→Hidden`、`02&&cap→Hidden`、`WP_048&&!02→Hidden`、`EarEnabled&&!04&&!02&&!WP_048→Shown`、`EarEnabled&&!04&&02&&!cap→Shown`。

### 10.4 验证协议（三项都做才算修好）

1. **逐帧复测**：修前 40 帧 39 flips（亮度 173.86/163.65 交替）→ 修后 **40 帧 0 flips**（True=40、亮度稳定 173.85）。
2. **状态矩阵**：新写临时脚本（`[MenuItem]` 入口 + GM 反射设参数），列出所有会显隐该部件的参数阶段，每阶段 settle 8 帧 → 连续采样 10 帧数 flips → 渲染头部近照。本例 9/9 阶段 flips=0：
   - P0 默认 True；P1 EarEnabled=false → False；P2 恢复 True；P3 AvatarAccessory_04=true（耳机）→ False；P4 关 → True；P5 WP_048=true（猫耳帽）→ False；P6 关 → True；P7/P8 描边关/开 → True。
   - 视觉确认：P3 出现黑色猫耳耳机头带、P5 出现浅绿白猫耳帽，自然耳均正确隐藏（不是闪烁中间帧）。
   - 像素差（2px 步长）：P0 vs P2 全图 1.74（完全复原）、P0 vs P1 顶部 88.98（耳区剧变）、P4 vs P3 顶部 102.34、P7 vs P8 顶部 5.99（描边有效）。
3. **全量回归**：修后重跑 141 项 GM Play 全量模拟测试 → `menuTotal=141 pass=134 fail=0`、`autoReset=7`、363 图、扫描关键词全 0、`renderers=81 badShader=0`、`MISSINGSCRIPTS|0`。

### 10.5 可复用判据 / 踩坑

- ANY 转移的**条件必须互斥**：条件恒真、或两条互补条件同时存在，就会逐帧闪烁；
- 删除菜单后**旧参数可能仍被 ANY 转移引用**（本例 `AvatarAccessory_02` / `ClothingCapEnabled` 已无菜单但仍在转移里）——清参数前先清转移；
- 修复后 `execute_code` 返回结构是 `structuredContent.data.result`（不是数组 `[0].result`）；
- 证据样例：`<VRCHAT_ROOT>\projects\LuzzyMeow\KisekaeRound58Evidence\earfix\`（pre_fix / post_fix / matrix / sim_after_fix，21 文件 / 2.29 MB）；报告 `KisekaeRound58EarFixReport.md`。


## 11. play 模式反射：读 baked EP / 手工挂 Animator 渲染（2026-10-21 第 59 轮补充）

本节补 §9.1 的「在 play 模式里怎么做事」的具体反射配方（不引用工程 / VRC 型别也能跑）。

### 11.1 读 baked EP 的参数表（saved / default / synced）

```csharp
// play 模式：execute_code 不能写 VRCAvatarDescriptor，全部反射
var root = GameObject.Find("<AvatarRootName>");
System.Type descType = null;
foreach (var c in root.GetComponents<Component>()) if (c.GetType().Name == "VRCAvatarDescriptor") descType = c.GetType();
object desc = root.GetComponent(descType);
// 1) EP 资产（baked）
object ep = descType.GetField("expressionParameters").GetValue(desc);
Debug.Log("EP=" + AssetDatabase.GetAssetPath((UnityEngine.Object)ep));
// 2) parameters 是 struct 数组：name / valueType / defaultValue / saved / networkSynced
var arr = (System.Array)ep.GetType().GetField("parameters").GetValue(ep);
for (int i = 0; i < arr.Length; i++) {
    var el = arr.GetValue(i); var et = el.GetType();
    if ((string)et.GetField("name").GetValue(el) == "<参数名>")
        Debug.Log("idx=" + i + " default=" + et.GetField("defaultValue").GetValue(el)
                + " saved=" + et.GetField("saved").GetValue(el)
                + " synced=" + et.GetField("networkSynced").GetValue(el));
}
// 3) baked FX controller：baseAnimationLayers 是 struct 数组，元素字段 type=="FX" / animatorController
var bl = (System.Array)descType.GetField("baseAnimationLayers").GetValue(desc);
for (int i = 0; i < bl.Length; i++) {
    var el = bl.GetValue(i); var et = el.GetType();
    if ((string)et.GetField("type").GetValue(el) == "FX")
        Debug.Log("FX=" + ((UnityEngine.Object)et.GetField("animatorController").GetValue(el)).name);
}
```

- 反射读 `UnityEngine.Object` 字段后对 missing ref 取 `.name` 会抛 `UnassignedReferenceException` → 每个都 try/catch。
- GM 元件的 `Module` 在**没开 GM 编辑器窗口时为 null** → 不能直接反射 GM 驱参数；要驱参数就写临时 `[MenuItem]` harness（它自己 `SetupModule`）。

### 11.2 手工挂 Animator 做参数 → 视觉验证

```csharp
var anim = root.GetComponent<Animator>();          // play 时 root 上已有 Animator，但 controller 可能为 null
anim.runtimeAnimatorController = (RuntimeAnimatorController)fxController;
anim.Rebind();
anim.Update(0.02f);                                 // 让参数层立即生效
```

- **同一支 `execute_code` 内「Set 参数 + 立刻 `Camera.Render()`」会拍到旧状态**（本例三档发色图完全一样）→ 必须拆成「一支 call 设参数并返回 → 下一支 call 再渲染」。
- 渲染配方：临时 Camera + RenderTexture + `ReadPixels` + `EncodeToPNG`，结束 `DestroyImmediate` 还原；写盘到 `<VRCHAT_ROOT>\tools\tmp\<主题>\`，别污染工程。
- 判断「参数是否真的生效」：既看 readback 值（即时变），也看**下一帧渲染**（真正套用动画后才变）。

## 12. 把 harness 移植到另一个工程（本工作区实测 2026-10-24，LuzzyMeow V2）

把 V1（小猫 Kipfel 工程）的 `KisekaeSimHarness.cs` 搬到 V2（Lilleo2.1 工程，7 个头像）跑全菜单回归，需要改这几处（清单可直接抄）：

1. **Gesture Manager 版本差异（最容易卡）**：V2 自带 GM **3.9.7**，`ModuleVrc3` 只有 **单参构造函数 `ModuleVrc3(VRCAvatarDescriptor)`**；
  V1 的 GM 3.9.9 是 `ModuleVrc3(desc, null)` 两参。harness 里写死两参会抛 `MissingMethodException` → 改成：
   ```csharp
   object module = null;
   var ctor2 = moduleType.GetConstructor(new[] { descType, typeof(ModuleSettings) /*或 null 类型*/ });
   if (ctor2 != null) module = ctor2.Invoke(new object[] { desc, null });
   else module = moduleType.GetConstructor(new[] { descType }).Invoke(new object[] { desc });
   ```
   （或直接 `try { 两参 } catch { 单参 }`；本次按「先试两参、失败再单参」改。）
2. **输出目录**：`OUT = "<VRCHAT_ROOT>/tools/tmp/<工程名>_sim"`（别复用上个工程的目录）。
3. **`MenuItem` 名**：改成工程专属（本次 `Tools/LuzzyMeowV2 Sim/Asset Reorg Test`），避免与旧 harness 的菜单项冲突。
4. **目标头像选择**：多头像场景按**根对象名**优先选（本次选 `正太猫猫！ (成品模-溪溪溪)`；注意繁体「貓 / 猫」并存，匹配要用包含而不是全等）。
5. **参数阈值**：V2 头像参数 382 个 > V1 的 ~200 → 遍历 `ExpressionParameters` 的阈值要放大（本次 `> 20` → 直接按数组长度）。
6. **测试场景隔离**：复制主场景为 `Assets\00_场景\_SimTest.unity`，只启用被测头像 + GM + `nadena.dev.ndmf__Activator`，其余根 `SetActive(false)`；测完删掉副本。
7. **FAIL 分类**：本次 162 项里 6 项 FAIL 全部是 **ABT（AvatarBasicTool）世界功能**（传送门 A/B、固定座位、放门传送至红/蓝/黄标）——编辑器里没有世界环境，标 `FAIL-不可测` 而不是 bug；
  3 项 `SETMISS` 是世界音频参数（MusicCommand / RK_song select / RK_ON），标 `NOT_RUN`。**头像自身菜单（衣柜/发型/小玩具/饰品/亮度）0 FAIL** 才说明功能正常。
8. **收尾**：删 `_SimTest.unity` + `.meta`；删 `Packages/com.vrcfury.temp/Builds`（bake 临时产物）；把 Play 时被项目插件重写的生成型资产（LightController 的 `.anim` 等）从压缩包还原；截图/日志移出工程（`_asset_reorg\evidence\` 或 `tools\tmp`）。
9. **VRCFury 阻塞**：若 Play 时弹 VRCFury 错误框（本次 `Failed to detect plug length`，头像上有未激活 `VRCFuryHapticPlug`）→ 测试副本里删掉该 plug 组件再跑，原场景勿改；
  详见 [工程 Assets 整理分册](../02-工程组织与工作流/02-05-unity-project-asset-reorg.md) §4。


## 13. 不重载域时 harness 的「二次运行」复位（本工作区实测 2026-10-24，LuzzyMeow V2）

**症状**：harness 跑完一轮（`DONE.txt` 已写）后再次 `ExecuteMenuItem` 触发 **完全无反应**、输出目录无新文件；反射读 statics 看到 `started=True built=True finished=True stepIdx=327 t0=<上一轮时间>`。

**根因**：本工程的 Enter Play Mode Options 关闭了 Domain Reload（或编辑器程序集在进出 Play 时不重载）→ `static bool started` 跨 Play 保留，`Start()` 首行 `if (started) return;` 直接返回；
且 `Finish()` 已 `EditorApplication.update -= Tick`，所以也不会有 Tick 在跑。

**正确做法**（任选其一）：
1. **最稳**：退出 Play → 重启 Unity（域重载，statics 归零）；
2. 不重启时用 `mcp__unity__execute_code` 反射复位全部状态，再调 `Start()`：
   - `started/built/finished/ready = false`；
   - `stepIdx/frameWait/pulseLeft/shots/frameCounter = 0`；
   - `steps`、`items` 两个 `List` 要 **Clear**（只清 `steps` 不清 `items` → `BuildMenu` 追加导致菜单数翻倍：本次 162 → 324，pass 306 全是假数字）；
   - `prms` / `ptypes` 字典 Clear；
   - `desc/camGo/lightGo/cam/gmComp/module/cur/pulseItem = null`。
   复位后 `State()` 应为 `step=0/0`，再 `Start()`。

**附带必修**：`Start()` 里 `Directory.CreateDirectory(OUT)` 只建主目录、**不建 `SHOTS` 子目录** → 截图全部 
`RENDERFAIL|Could not find a part of the path '...\shots\xxx.png'`、`summary` 里 `shots=0`。`Start()` 必须同时 `Directory.CreateDirectory(SHOTS)`。

**判据**：`State()` = `started=True … step=0/0`；跑完 `summary.txt` 的 `menuTotal` 与基线相同、`shots > 0`、`DONE.txt` 时间戳为本次。


## 14. Play 前 VRCFury SPS 插头 bake 失败：缺失 prefab 导致 `Failed to detect plug length`（本工作区实测 2026-10-24）

**症状**：在 Avatar 场景按 Play，弹 VRCFury 模态框（编辑器卡死、MCP 全部超时）：
```
VRCFury encountered an error.
Failed to build VRCFury component: BakeHapticPlugsService.ApplyEarlyBake on Avatar Root
Failed to bake SPS Plug: armature/Hips/.../ちんぽ/sao/SPS Plug
Failed to detect plug length
(VRCFBuilderException in GetWorldSize at PlugSizeDetector.cs:62)
```
随后 SDK 报 `Preprocess Callback Failed ... 'VrcPreuploadHook' reported a failure`，Gesture Manager 面板显示没有可用 descriptor、根物体被标 `(Preprocess hooks failed)`。

**根因**：插头所在物体属于一个**源 prefab 缺失**的 prefab 实例（`PrefabUtility.GetPrefabInstanceStatus == MissingAsset`）。
插头 `autoRenderer=True` 时 `PlugRendererFinder.GetAutoRenderer` 仍能找到场景里的 `SkinnedMeshRenderer`，但该 SMR 的 `sharedMesh == null`（prefab 丢了，mesh/材质/骨骼全空）
→ `PlugSizeDetector.GetWorldSize` 遍历不到顶点、`worldLength <= 0` → 抛 `Failed to detect plug length`。`PlugSizeDetector.cs:21–75` 里其它 throw：
`Failed to find plug renderer`（找不到 renderer）、`Failed to detect plug radius`；
`BakeHapticPlugsService.ApplyEarlyBake` 遍历所有 `VRCFuryHapticPlug` 并包装成 `Failed to bake SPS Plug: {path}`；
`VRCFuryHapticPlugEditor.cs:481` 在 `enableSps && renderers.Count == 0` 时还有一条 `no renderer was found`。

**定位三步**（都能用 `mcp__unity__execute_code` 离线确认）：
1. 反射取 `VF.Component.VRCFuryHapticPlug` 实例（直接引用不 accessible），看它 `GetRenderers()` 找到的 SMR 是否 `sharedMesh == null`；对 `PlugSizeDetector.GetWorldSize` 反射调用可拿到真实异常。
2. `PrefabUtility.GetPrefabInstanceStatus(gameObject)` 找 `MissingAsset`；场景 YAML 里数 `m_SourcePrefab` 的缺失 GUID。
3. 确认缺失资产是否真的丢了：`.meta` 全盘扫描 GUID（本次 50,937 个 meta 0 命中）、原始压缩包里 `7z l` 查同名文件、`Library` 缓存里搜 GUID —— 三者都空 = 作者打包前就删掉了。

**修复（最小、可回滚）**：删掉这些失效的 `VRCFuryHapticPlug` 组件（**只删插头，不删 Socket**），场景里 `VRCFuryHapticSocket`、SPS 菜单、参数、动画全保留；删前把序列化字段记录成文本、整份场景备份。重建 Play 后 bake 成功、
GM 全菜单回归与基线逐项一致。**不要**用「关掉 autoLength/autoRadius 硬填 size」或「关 enableSps」绕过：那会在缺失网格上留假尺寸/假接触，后续再想恢复更难。

**判据**：Play 无 VRCFury 错误框；descriptor 的 `expressionsMenu` 指向 `Packages/com.vrcfury.temp/Builds/.../VRCFury Menu.asset`；参数/层数与基线一致；退出 Play 后场景 `DIRTY=False`、Console 0 error。

**边界**：删除插头 = 该 SPS 发射端失效（本例网格本来就丢了、视觉上不存在）；接收端 Socket 与菜单不受影响；要完整恢复必须找回原 prefab（本次 GUID `7dac145a15982b447880804954d1e4e8`，本机已无任何副本）。

**同类连带坑**：① 进入 Play 后工程里的生成型插件会**往 Assets 里写资产**（LightController 重写 5 个 `.anim`、FaceEmo 建 `Assets/Suzuryg/FaceEmo/Imported/<时间戳>/`），收尾要按压缩包原件还原/删除；
② 这种 prefab 缺失还会让其它对象出现 `m_Avatar/m_Controller/audioClip` 等 null 引用，修完插头后 bake 仍可能暴露下一个错误，要迭代到 Play 干净为止。

## 15. Hierarchy 里某个物体是红色 / Inspector 显示「预制件（缺失）」——MissingAsset 实例怎么处理（本工作区实测 2026-10-24）

**症状**：Hierarchy 里某节点名字是红色；Inspector 顶部「预制件（缺失）」/「Missing (GameObject)」；其下网格可能是空 mesh、材质为空、Animator 控制器为空；进 Play 时 VRCFury / NDMF 可能因此报错（例如 SPS 插头 `Failed to detect plug length`）。

**根因**：该物体是**缺失源 prefab 的实例**——作者打包时删掉了 prefab 资产但场景仍保留实例。判定三步（本工作区实测）：
1. `UnityEditor.PrefabUtility.GetPrefabInstanceStatus(go) == PrefabInstanceStatus.MissingAsset`（嵌套子物体同样返回 MissingAsset，需向上找 outermost root）。
2. 取场景 YAML 里该实例的 `m_SourcePrefab.m_SourcePrefab` GUID，在工程 `Assets/` + `Packages/` 全部 `.meta` 里搜不到。
3. 在作者压缩包条目列表里搜 prefab 名 / 在解出的 `Library` 缓存里搜 GUID 也搜不到 → 本机无法恢复，只能找原作者/购买渠道。

**三个处理选项（权衡不同，先问需求再动手）**：
| 选项 | 红色 | 功能/数据 | 以后导入原 prefab 能否自动重连 |
|---|---|---|---|
| 保留缺失实例 | 仍在 | 全部保留（内部空引用照旧） | ✅ 能（推荐：先别动） |
| `PrefabUtility.UnpackPrefabInstance(obj, PrefabUnpackMode.Completely, InteractionMode.AutomatedAction)` | 消失 | 全部保留（物体、组件、MA Parameters、contact、空引用状态都不变） | ❌ 不能（永久失去） |
| 删除该物体 | 消失 | 丢失其 MA Parameters/contact/动画曲线目标，不推荐 | ❌ |

**解包实测（LuzzyMeow V2，`Chiffon_Modular_Avatar_tintin v5.4`，82 个物体）**：解包保存后 `NotAPrefab`、82 Transform 全在、MA Parameters 10 个 `Nchin_*` 参数全在、
contact `Nchin_touch_contact` 在（所在对象未激活，与解包前一致）；GM Play 回归 `menuTotal=162 / pass=153 / fail=6(全 /ABT/) / noparam=1 / shots=287`，
bake `VRCFury Menu` 顶层 6、参数 145、层 5/5 —— 与解包前逐项一致；Console 0 error。

**判据**：解包只去掉 prefab 关联，不影响组件序列化数据；但**解包前一定要备份场景**，并明确告知用户「以后即使拿到原包也不会自动重连」。若用户还想要真正修复，保留实例、等原 prefab 导入后 Unity 自动重连（最省事）。

## 16. 截图证据怎么出：3D 视图 + GM 径向菜单（本工作区实测 2026-10-24）

**3D 视图（不依赖 Game 视图大小 / 不污染场景）**：
```csharp
var go = new UnityEngine.GameObject("___tmpShotCam");
var cam = go.AddComponent<UnityEngine.Camera>();
cam.transform.position = headPos + new UnityEngine.Vector3(0f, 0.02f, 0.55f); // 角色正面 +Z 侧
cam.transform.LookAt(headPos);
cam.fieldOfView = 40f;
var rt = new UnityEngine.RenderTexture(720, 900, 24);
cam.targetTexture = rt; cam.Render();
UnityEngine.RenderTexture.active = rt;
var tex = new UnityEngine.Texture2D(720, 900, UnityEngine.TextureFormat.RGB24, false);
tex.ReadPixels(new UnityEngine.Rect(0, 0, 720, 900), 0, 0); tex.Apply();
System.IO.File.WriteAllBytes(outPath, tex.EncodeToPNG());
cam.targetTexture = null; UnityEngine.Object.DestroyImmediate(go); // 拍完即删，别存进场景
```
拍完用 `read_image` 目视确认（服装/部件是否正确，有没有粉材质、空帧、T-pose）。

**GM 径向菜单截图**：GM 菜单是独立 `EditorWindow`（`BlackStartX.GestureManager.Editor.Modules.Vrc3.Vrc3FloatingMenu`，字段 `_module`、`_menu`）。
- `ScreenCapture.CaptureScreenshot` **只拍 Game 视图**，不含 EditorWindow 覆盖层。
- 从 `execute_code` 直接调 `ShowRadialMenu(RadialMenu, VisualElement)` 会报 `You can only call GUI functions from inside OnGUI`。
- 可行路径：`EditorWindow.CreateInstance(fmType)` → `ShowUtility()` → 反射设 `_module = gm.Module`、`_menu = 衣柜` → `Repaint()`，
  然后用 PowerShell P/Invoke `EnumWindows` 找该窗口 HWND + `GetWindowRect` + `PrintWindow(hwnd, hdc, 2)`（`PW_RENDERFULLCONTENT`）抓 PNG。
- 只想知道结构时更快：递归 dump 烘焙后的 `VRCExpressionsMenu`（`name/type/parameter/value`）成文本，和截图一起交付。

**判据**：截图非空、无粉色、无 T-pose；菜单截图能看到目标子菜单与控件图标（toggle 打开态有循环箭头徽标）。

## 17. 场景块级 diff 取证：证明「只动了预期的东西」（本工作区实测 2026-10-24）

Unity 场景 `.unity` 是文本 YAML，按 `--- !u!<classID> &<fileID>` 分块。写 python（只读）：
1. `blocks = {(class_id, file_id): text}`；
2. 比较两份场景 → `added` / `removed` / `changed`（文本不同）；
3. 对 `changed` 再做字段级 diff，过滤已知噪声。

**预期噪声（不算改动，别写进结论）**：
- `unityVersion: 2022.3.22f1 → 2022.3.22f1c1`（改过 `ProjectVersion.txt` 后所有块都会变一行）；
- `cachedExecutionGroupIndex: 0 ↔ -1`（NDMF bake 缓存字段）；
- 浮点末位（`0.18028352` → `0.18028378`、`m_LocalRotation` 里的 `0` ↔ `-0`）；
- `RenderSettings.m_IndirectSpecularColor`（环境光缓存，无烘焙时归零）。

**真实改动签名**：新增/删除块、`m_Children` / `m_Father` 变化、`m_RemovedComponents` 变化、组件增删。
**判据**：除上述噪声外，**没有** script GUID / 菜单控件 / 参数 / 动画引用变化，才算「只做了预期改动」。

## 18. Play / bake 日志噪声对照：这些红字不是错误（本工作区实测 2026-10-24）

进入 Play 触发 NDMF/VRCFury bake 时，`Editor.log` 与 Console 常出现下列红字，**全部非致命**，判据是 bake 是否完成（菜单生成、descriptor 正确、参数数正确、`badShader=0`）：

| 日志 | 含义 |
| --- | --- |
| `MissingReferenceException: The object of type 'VRCPhysBone' has been destroyed but you are still trying to access it.`（栈：`VRCPhysBoneBase.GetRootTransform` ← `VRCConstraintBase.DeterminePhysBoneDependency` ← `VRCConstraintManager.RegisterConstraint`） | VRCFury/NDMF 在 bake 克隆期访问已销毁对象，bake 继续 |
| `[NDMF] Error Reported: 检测到未知的组件'VF.Component.VRCFuryHideGizmoUnlessSelected'！` | VRCFury 私有 gizmo 组件，NDMF 不认识，忽略 |
| `Controller VRCFury XXX was changed outside of NDMF animator services; cloning a second time` | VRCFury 二次写控制器 → NDMF 再克隆一次，最终结果正确 |
| `Proxy object was destroyed improperly! Resetting pipeline...` | VRCFury 代理对象回收，bake 继续 |
| `SendMessage cannot be called during Awake, CheckConsistency, or OnValidate` | VRCFury 组件在 Awake 里发消息，无害 |
| `State "isLocal" in Controller "EyeMaskNightMode" uses parameter ""` | 作者控制器里有空参数状态，无害 |
| `Material 'X' with Shader 'Y' doesn't have a color property '_BacklightColor'` | 亮度调节给不支持该属性的材质设色，仅红字 |
| `[anatawa12.gists] Skipping the computing because this play is avatar info input play` | AAO 性能窗口在 avatar info play 里跳过，正常 |

**真正的失败**：出现 `VRCFBuilderException` / 模态错误框（`DisplayErrorPopup`）/ 菜单数为 0 / descriptor 为空 —— 这些才需要处理（例：`Failed to detect plug length` 见 §14）。

## 19. Play 模式下头像根 Animator 没有控制器：直接 `SetFloat` 做验证必然失败（本工作区实测 2026-11-05，LuzzyMeow V2 沐梓溪狼）

**症状**：在 Play 里写探针 `GameObject.Find("沐梓溪狼").GetComponent<Animator>()` 后 `SetFloat("PJ_Ghost",1)`，随后 `GetFloat` 返回 **-999**（或 “Parameter does not exist”），任何对象状态都不变；`layerCount=0 / parameterCount=0`。

**根因**：NDMF/VRCFury 烘焙出的控制器只挂在 `VRCAvatarDescriptor.baseAnimationLayers`（Base/FX/Gesture/Action）与 `specialAnimationLayers` 上；**编辑器 Play 不会把 FX 控制器装到场景头像根的 Animator 上**。只有 Gesture Manager 运行时才会按 descriptor 组装 animator（GM 编辑器窗口未打开时 `GestureManager.Module` 也为 null）。

**判据 / 复现**（execute_code 一次打印即可确认）：
- `root.GetComponent<Animator>()` → `runtimeAnimatorController == null`、`layerCount == 0`、`parameterCount == 0`；
- 同时 `desc.baseAnimationLayers[4].animatorController != null`（FX 控制器是存在的）；
- `animator.SetFloat("任意已有参数", 1)` 不抛错但 `GetFloat` 读出 -999/0。

**正确做法**：动态验证一律走 GM 运行时模块，而不是直接驱动场景 animator ——
```csharp
var gmType  = Type.GetType("BlackStartX.GestureManager.GestureManager, vrchat.blackstartx.gesture-manager");
var setType = Type.GetType("BlackStartX.GestureManager.Modules.ModuleSettings, vrchat.blackstartx.gesture-manager");
var modType = Type.GetType("BlackStartX.GestureManager.Editor.Modules.ModuleVrc3, vrchat.blackstartx.gesture-manager.editor");
var gm  = UnityEngine.Object.FindObjectOfType(gmType);
var st  = Activator.CreateInstance(setType);
setType.GetField("favourite").SetValue(st, desc);
var mod = Activator.CreateInstance(modType, new object[]{ desc });
modType.GetField("Settings").SetValue(mod, st);
gmType.GetMethod("SetModule").Invoke(gm, new object[]{ mod });
// 等 modType.GetProperty("Active") 为 true 后再用 mod.Params[name].Set(mod, value, null)
```
（`ModuleVrc3` 在本工程 GM 3.9.7 上是单参构造；别的版本可能是两参，写 try/catch 兜底，见 §12。）

**关联文件**：`Assets/Editor/MeowGhostPlay.cs`（失败的直驱示范 → `docs/_ghostplay.txt`）、`Assets/Editor/MeowVerify.cs`（正解，44 PASS 即由此驱动）、本工程 `docs/_verify_log.txt`。

## 20. 判读 `MA Responsive` 层：菜单开关「点了没反应」到底断在哪一层（本工作区实测 2026-11-05）

**背景**：MA 的 ObjectToggle / 反应式组件烘焙后，会为每个被控对象生成一层 `MA Responsive: <对象名>`。把它逐段读完，可以判定故障在「菜单→参数→层→条件→曲线→目标网格」链条的哪一环。

**层结构（烘焙 FX 里）**：
- `<default>` 状态：clip 为空 motion（`Copied from ... / empty motion`）；
- 命名状态（如 `帽子` / `外套` / `小幽灵`）：clip 含两条曲线——
  - `path='<对象路径>' prop=m_IsActive type=UnityEngine.GameObject`（真正开关对象）
  - `path='' prop='__MA/ActiveSelfProxy/<对象>##N' type=UnityEngine.Animator`（MA 的代理参数回写）
- 状态转移条件 = **该菜单参数本身**（Bool 型参数用 `Greater 0.995` / `Less 1.005` 阈值组；`<default> → 命名态` 与回程对称）。

**六步判读顺序**：
1. 烘焙菜单树里存在该开关（名称/路径/控件类型正确）；
2. EP 里参数已注册、类型正确（bool 进 FX 变 float 是正常的）；
3. 烘焙 FX 存在对应 `MA Responsive: <对象>` 层且 `defaultWeight=1`；
4. `<default> → 命名态` 的入口条件**只含菜单参数**；若还带 `__MA/ActiveSelfProxy/<自身>` → 自锁（见 SKILL 症状表「目标就是 MenuItem 父级容器」条，修法是把 MenuInstaller/MenuGroup 移到激活容器）；
5. clip 曲线寻址到实际存在的对象路径（父级 active 链必须通；inactive 宿主会让反应式组件整体静默失效）；
6. 目标对象上**是否本来就有可激活的可见网格/动画**——这一步最容易被忽略：开关链路全通也可能「本来就看不见」。

**本例（小幽灵 `PJ_Ghost`）**：第 1–5 步全 PASS，第 6 步发现 `睡衣/OG_GhostAnchor` 是空壳锚点，其下 OyasumiGhost 三只幽灵的可见网格默认 inactive，且作者控制器 `OG_GhostAnchor.controller` 的 25 个 `.anim` 中**没有任何 `m_IsActive` 曲线** → 该开关本来就无法产生可见效果（资产侧问题，非本次改动）。

**关联文件**：`docs/_probe_fx.txt`（层/状态/曲线 dump）、`docs/_ghost.txt`（对象/组件结构 dump）、`docs/_verify_log.txt`（动态断言）。

> 更新时间 2026-11-05（补充来源：沐梓溪狼衣装菜单重构接手验收，LuzzyMeow V2）

## §21 自定义 GM 模块驱动时 4 个必踩坑（2026-11-05 第 140 轮实测：眼罩开关验证）

| 日期 | 现象 | 根因 | 判据/复现 | 正确做法 | 关联文件 |
|---|---|---|---|---|---|
| 2026-11-05 | 自建 GM 模块后 `Params` 里按 `RBS_EyeMask_Toggle` 精确名取参数失败、`Set` 后无变化 | MA 参数烘焙时会加去重后缀 `$<8位hash>`，EP 与 GM `Params` 的键名都带后缀（如 `RBS_EyeMask_Toggle$2b7aeaf9aad8`），只有菜单 Control 的源参数名不带 | dump `Params.Keys` 看实际键名；日志会打 `!! missing <name>` | 用**前缀匹配**（`StartsWith(prefix)`）取键，不要精确匹配；回读也用同一键 | `Assets/Editor/MeowMaskPlay.cs`、`docs/_mask_play.txt` |
| 2026-11-05 | Play 里按固定路径 `沐梓溪狼/(Type-D)(White)EyeMask` 找不到眼罩 | `ModularAvatarBoneProxy(attachmentMode=AsChildAtRoot)` 在烘焙时把对象重挂到目标骨骼（Head）下；编辑态在头像根下，烘焙克隆里路径变成 `.../Head/(Type-D)(White)EyeMask` | 编辑态 `transform.Find` 命中、Play 克隆同路径 miss | 按 `name` 在场景根递归搜索（`SceneManager.GetActiveScene().GetRootGameObjects()` + `GetComponentsInChildren<Transform>(true)`），不要依赖固定路径；绑骨验证就查实际父级是否为 `Head` | `docs/_mask_play.txt`、`docs/_mask_probe2.txt` |
| 2026-11-05 | Play 克隆里查不到 `ModularAvatarParameters`/`ModularAvatarMenuItem`，误以为组件丢了 | MA 在烘焙时把自身组件消费（移除）后只留结果；Play 里的 `沐梓溪狼(Clone)` 没有 MA 组件 | 克隆子树 MonoBehaviour 仅剩 VRCPhysBone/ParentConstraint 等 3 个 | 查 MA 参数/菜单必须回**编辑态**场景实例；Play 里只查 EP/Animator/结果对象 | `docs/_mask_probe2.txt`、`docs/_mask_play.txt` |
| 2026-11-05 | 第二次进 Play 跑 harness 时参数全部 `missing`、Dump 跳过，看似脚本坏了 | 「Enter Play Mode Options: Reload Domain disabled」时 static 字段跨 Play 会话保留（`_started=true`、`_module` 指向已销毁对象、`_inited=true`），`Run()` 直接 return 或用旧模块字典 | 日志只有 `READY`、无 `INIT`/`PARAM KEYS` 段；`!! missing` 全出现 | `Run()` 里显式清零所有 static（`_started/_inited/_gm/_settings/_module/_desc/_root`）并重建模块；或先 Stop Play → 重新编译 → 再进 Play | `Assets/Editor/MeowMaskPlay.cs`、`docs/_mask_play.txt`（17:48 vs 17:57 两次对比） |

**默认态验证模板**（本次眼罩用）：`INIT` 截图/dump → `EP params`（name/type/def/saved/synced）→ `PARAM KEYS`（RBS_ 前缀）→ `DEFAULT` 对象 activeSelf + 可见 Renderer 数 → set true → `ON` dump → set false → `OFF` dump；三态都记录 `[設定用]アイマスク` 的 activeSelf 与 renderer 数（默认/开/关分别 0/1/0）。

> 更新时间 2026-11-05（补充来源：沐梓溪狼眼罩绑骨与菜单迁移，LuzzyMeow V2）

---

## §22 道具类第三方 gimmick 的动态验证：GM 反射三条补充 + 「开关≠可见」（本工作区实测 2026-11-09）

> 场景：给 LuzzyMeow V2 沐梓溪狼 装 BOOTH 6867118「ネガライン」（MA 対応 道具），
> 菜单 `道具 › Nega_menu › Gimmick_on`。用 GM 运行时模块逐态驱动，首轮 3 项 FAIL 全是**用例写错**，不是实现错。

### 22.1 GM 反射三条补充（§2 / §19 的增量，不冲突）

| # | 事实 | 后果 | 正确写法 |
| --- | --- | --- | --- |
| 1 | `ModuleVrc3.Params` 是**字段**（`Dictionary<string, Vrc3Param>`），**不是属性** | `GetProperty("Params")` 返回 null → `NullReferenceException` | 先 `GetField("Params", Instance\|Public\|NonPublic)`，再退回 `GetProperty` |
| 2 | `ModuleVrc3.Menu` 是**属性**（当前页 `VRCExpressionsMenu`） | 想知道「烘焙后 GM 眼里的一级菜单长什么样」只能靠猜 | 直接读 `module.Menu`，打印 `controls[].name/type`；本例得到 `功能/外观/道具` 三项，作为「新子菜单已进轮盘」的第一手证据 |
| 3 | Play 里 `Type.GetType("…ModuleVrc3, vrchat.blackstartx.gesture-manager.editor")` **可能返回 null**（域重载/装配加载时序） | 模块建不起来，日志只有 `[gm] module=False` | 写 `FindType(full)`：先 `Type.GetType`，失败再遍历 `AppDomain.CurrentDomain.GetAssemblies()` 逐个 `asm.GetType(full)` |

### 22.2 「菜单开关 ≠ 可见」：道具类资产的两级显隐

本例 `Gimmick_on.negaline=1` 只让 `nega` 层从 `gimmick_off` 切到 `gimmick_on`，
而 `gimmick_on` 的 clip **只有一条曲线**：`Negaline/Anim_nega_off.m_IsActive = 1`（容器显形）。
真正的「枪出现」在另一层 `blast&shot` 的状态机里，按 **`GestureLeft`** 驱动：

- `default → spin`：条件 `negaline If 0(真) && GestureLeft Equals 2`（左手 Open）；
- `spin → gunon`：`hasExitTime`，约 0.9 s 后才把 `.../pivot/blaster.m_IsActive` 置 1；
- `gunon → off`：`GestureLeft Equals 5`（左手 RockNRoll）→ 收枪方块 `off_square` 出现；
- `off → default`：**`hasExitTime` 1 s**。

**结论**：
1. 只点菜单、不做手势，**枪本来就看不见**——这不是 bug，是作者设计（BOOTH 用法页同样写着「Gimmick_on 有效化 → 左手 Open」）。
2. 用例必须显式补 `GestureLeft`（GM 里同样用 `Set` 写 Int，见 §2），否则会把「正常」判成 FAIL。
3. 「收枪态 → 默认态」有 1 s exitTime，**拔枪/收枪各留 ≥ 90 帧**再断言；40 帧就断言必然误判（本轮 T5 首版就是这么错的）。

### 22.3 本轮判据（`docs/_r311_play.txt`，9/9 PASS）

| 用例 | 操作 | 期望 | 实测（可见 Renderer） |
| --- | --- | --- | --- |
| T0 | 默认（`negaline=0`、`GestureLeft=0`） | 容器关、枪不可见 | Negaline=0 blaster=0 |
| T1 | 点 `Gimmick_on`（开，手仍 Neutral） | 容器开、枪仍隐藏 | 容器 active=True，blaster=0 |
| T2 | 左手 Open（`GestureLeft=2`） | 拔枪 | blaster=16 |
| T3 | 左手 RockNRoll（`=5`） | 收枪、收枪方块可见 | blaster=0，off_square=1 |
| T4 | 点 `Gimmick_on`（关） | 全隐藏 | Negaline=0 |
| T5 / T5b | 第 2 轮：开（过渡 40 帧）/ 再 Open（110 帧） | 过渡期枪未出 / 随后拔枪 | blaster=0 → 16 |
| T6 | 旧饰品（`Acc_WolfGloves=1`）+ 枪 + Open | 两者共存 | blaster=16、手套=1 |
| T7 | 收尾全关 | 全隐藏 | 0 |

> 更新时间 2026-11-09（来源：ネガライン 道具安装与轮盘验收；证据 `docs/_r311_play.txt`、`docs/shots_r311/`）

## §23 GM 补全：构造签名 / 组合测试 / Action 层 / Contact 模拟 / 叶测试判定（2026-11-06 至 11-11）

> 来源：原 08-症状速查 第 156/166/169/172/177 轮实测，2026-11-11 结构化并入。Play 域菜单资产回退、is_changing 卡死等边界见 [07-09 §1](07-09-dynamic-verification-and-evidence-recipes.md)。

### 23.1 ModuleVrc3 构造签名有版本差异（先扫程序集再 CreateInstance）

- GM 编辑器程序集 `BlackStartX.GestureManager.Editor`，模块类型 `BlackStartX.GestureManager.Editor.Modules.Vrc3.ModuleVrc3`（基类 `BlackStartX.GestureManager.Data.ModuleBase`）。**`Type.GetType(全名字符串)` 跨程序集返回 null**，必须扫 `AppDomain.CurrentDomain.GetAssemblies()` 取类型（第 156 轮）。
- 反射 dump `GestureManager` 组件：只有 `SetModule(ModuleBase)` 与字段 `Module`，**没有** `ModuleVrc3()` 方法。
- **签名差异（两处实测都留痕，按本机 GM 版本反射确认）**：
  - 第 156 轮（2026-11-07）：构造函数 **1 参** `(VRCAvatarDescriptor)`，写 2 参 MissingMethodException；
  - 第 166 轮（2026-11-08）：构造签名 `(VRCAvatarDescriptor, ModuleVrc3 source)`，用 1 参 MissingMethodException。
- 可用配方：① 扫程序集取类型；② `Activator.CreateInstance` 按实际签名传参；③ `gm.SetModule(module)`；④ 轮询 `ModuleBase.Active` 变 true（本例 26s）；⑤ 遍历 module 上名字含 param 的字段取 `IDictionary`（本例 430 项 / 另一轮 485 项），对每项调 `Vrc3Param.Set(module, value, null)`（3 参）。工具 `Assets/Editor/R166PlaySim.cs`。
- NDMF 预览场景占满主线程导致 Active 恒 false 的处理见 [07-09 §1.8](07-09-dynamic-verification-and-evidence-recipes.md)。

### 23.2 组合测试：混搭累积不能用单点串行点击

GM 的 `Vrc3Param.Set` 每次写入后下一帧被模块重置，单用例串行点击测不出跨参数累积（`pre` 恒 0.00，第 169 轮）。
- 写**组合测试脚本**（`Assets/Editor/HalloweenComboTest.cs`）一次性同时 Set 多个参数，隔 5 帧读 `parts`/`visibleRend`。
- 判据：`visibleRend` 随开启件数**单调递增**（本工程 15→17→18→19→20），且 `parts` 与参数逐位对应。证据 `docs/_sim_halloween_combo.txt`。

### 23.3 GM 不自动启用 Action 层：舞蹈类验证的降级与裁决法

> ⚠ **2026-11-12 补充/取代适用边界**：本节保留历史症状和采样记录；不能用它证明当前插件接线正确或客户端会动。新一轮实测确认抽取控制器的层索引、AFK首层覆盖、VRCFury owner 级 PLC→ALC 展开都能造成同一现象；当前 GM 能在正确权重下输出 Humanoid 舞蹈。完整机制、修复与证据见 [03-07 §14](../03-菜单与参数/03-07-ma-platform-sync-mmd-and-motion.md#14-抽取舞蹈-action-后只放歌afk-覆盖与走跑冻结2026-11-12)。

**症状**：GM Play 里 SyncDances=1、FX 已进入 `SARDefaultDanceFX`、接触发送器 SD1S active，但身体不动（第 177 轮）。
**根因**：`ModuleVrc3.cs:228-246` 对 Action/Sitting/IKPose/TPose 初始 `weightOff`，GM 无自动启用 Action 的机制（客户端默认启用）。反射 `_layers[Action].Weight.Set(1)`（`LayerData` 为 internal struct，`Weight` 字段需 NonPublic）后状态机进入 `Dance_SARDefaultDance`，但本环境 GM 图仍不应用 Action 的 Humanoid 输出。
**裁决法**：编辑态 `AnimationMode.StartAnimationMode + SampleAnimationClip(avatarRoot, clip, 0.5f)` 读 Hips 欧拉差（本轮 delta=62611）证明 clip 与头像 Humanoid 骨架兼容 → 插件本体可用；**GM 全身播放标 NOT_RUN（环境限制），多人同步标 NOT_RUN（需客户端）**。Humanoid→Generic 的 Unity 限制见 [05-11 §9](../05-动骨与穿模/05-11-skinning-binding-and-geometry-repair.md)。
**追溯（复核闭环 2026-11-11 23:30）**：delta=62611.26 复跑落盘 `docs/_r510_dance_sample.txt`（LuzzyMeow V2）；复核闭环总表 `docs/_r510_review_close.txt`。

### 23.4 GM 可模拟 Contact（取代「Contact 必须真机」的过宽旧口径）

第 177 轮 ModuleVrc3 实测可模拟接触触发：临时 Sender 贴口部 activator=1、前移 5.5cm=0（旧半径组合 0.06 会触发、新 0.05 不触发，判别成立）；临时对象 Play 内销毁不存盘。RealKiss 定标细节见 [02-11 §2.4](../02-工程组织与工作流/02-11-plugin-integration-rk-realkiss-props.md)。
**边界**：跨玩家 R2R/Leader 链仍 NOT_RUN（需客户端）。

### 23.5 叶测试判定按控件类型区分（Button / 自复位 Toggle 读回 0 不算 FAIL）

GM 批量 163 叶测试里 2 项 set=1 read=0（第 172 轮）：二者分别是 **Button（清除选择，脉冲自复位）** 与 **RK_play（播放逻辑复位 Toggle）**，非普通状态参数。[LEAF] 逐条 set/read 对比 161/163 相等。
**规则**：Button/自复位 Toggle 读回 0 不算 FAIL；其余 set≠read 才是问题。证据 `.plan-mode/evidence/plan002_gm.txt`。

### 23.6 GM 验证边界清单（汇总）

| 项 | GM 能不能验 | 替代 |
|---|---|---|
| 菜单→参数→对象显隐 | 能（反射驱动 + 读回） | — |
| Contact 触发（单机） | **能**（§23.4，2026-11-11 起） | — |
| 跨玩家 R2R / Leader 链 | 不能 | 客户端，NOT_RUN |
| saved 跨世界持久化 | 不能 | 烘焙 EP 读 `saved` + 客户端重登，NOT_RUN |
| Gesture 层 / finger tracking | 不模拟 | 进游戏实测（[05-08 §24.4](../05-动骨与穿模/05-08-cross-avatar-garment-fit-newfit.md)） |
| Button 约 1s 自动复位 | 不模拟 | 真机（[03-06 §18.1](../03-菜单与参数/03-06-ma-menu-build-by-reflection.md)） |
| Action 层 Humanoid 全身播放 | 本环境不能（§23.3） | AnimationMode 采样裁决 |
| 面捕视觉（硬件键输入） | 不能伪造 | 真实 OSC/VRCFT 或客户端（[04-09 §11](../04-外观与表情/04-09-face-tracking-native-sdk.md)） |
| 声音可听 | 不能 | 客户端（[07-09 §2](07-09-dynamic-verification-and-evidence-recipes.md)） |

