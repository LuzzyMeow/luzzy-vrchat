# VRCFury「Write Defaults」弹窗：是什么、怎么永久解决

> 本工作区实测 2026-10-24（第 64 轮，`LuzzyMeow V2` 工程 / 头像 `沐梓溪喵`）。VRCFury 1.1276.0、Unity 2022.3.22f1c1。
> 触发场景：进 Play 时弹出模态框 `VRCFury has detected a (likely) broken mix of Write Defaults on your avatar base.`，挡住 Gesture Manager / harness，用户问「这个弹窗是什么，能不能永久解决」。

---

## §1 症状

- 进 Play（或 VRCFury 预处理）时弹窗：
  > **VRCFury has detected a (likely) broken mix of Write Defaults on your avatar base.**
  > Auto-Fix / Skip and stop asking / Skip
  > Debug info: Base: 65 on · Additive: 2 additive-on · Gesture: 20 on · Action: 30 on · FX: 333 on 2 direct-on · Sitting: 6 on · T: 1 off · IKPose: 1 off · VRCF will try to convert to ON
- 弹窗是**模态**的：不点掉它，编辑器卡住、MCP 命令超时、Gesture Manager 不可用。
- 「Skip」点过之后，下次进 Play 还会再弹。

## §2 根因

VRChat 动画控制器里每个状态都有一个 `Write Defaults`（WD）开关。同一个 avatar 的多层控制器里 WD 混用（本项目 7 层里 `T` 和 `IKPose` 各 1 个状态是 OFF、其余 ON），VRCFury 会认为「可能有问题」而弹窗提示。

VRCFury 1.1276.0 的判定逻辑（`Editor\VF\Service\FixWriteDefaultsService.cs`）：逐层统计 `on` / `direct-on` / `additive-on` / `off`，
当存在 mix 且没有已保存的 `FixWriteDefaults` 设置时弹窗；若检测到 `MmdCompatibility` 功能则直接 ForceOn、不弹。

## §3 三个按钮的真实行为（读源码结论）

| 按钮 | 实际行为 | 永久性 |
|---|---|---|
| **Auto-Fix** | `FixWriteDefaultsLater.Save(avatar, true)` → 在头像根加一个 VRCFury 组件，`content = FixWriteDefaults { mode = Auto }`；烘焙时把**所有层**统一转 ON | ✅ 永久，且真正修掉混合 |
| **Skip** | 什么都不写 | ❌ 下次进 Play 还弹 |
| **Skip and stop asking** | `FixWriteDefaultsLater.Save(avatar, false)` → `mode = Disabled`，只把弹窗静音，**不改 WD** | ✅ 永久静音，但混合仍在 |

相关源码（`Packages/com.vrcfury.vrcfury/`）：

- `Runtime\VF\Model\Feature\FixWriteDefaults.cs`：`internal class FixWriteDefaults : NewFeatureModel { enum FixWriteDefaultsMode { Auto, ForceOff, ForceOn, Disabled } ; mode = Auto; }`
- `Editor\VF\PlayMode\FixWriteDefaultsLater.cs`：`SaveNow` 里若已有 `FixWriteDefaults` 则跳过，
  否则 `avatar.AddComponent<VRCFury>()` + `vf.content = new FixWriteDefaults { mode = auto ? Auto : Disabled }`；若在 Play 中点击则先存 `EditorPrefs["vrcfFixWd"]`，
  退出 Play 时按头像名回写。
- `Runtime\VF\Model\VRCFury.cs`：`internal class VRCFury : VRCFuryComponent`，`[SerializeReference] public FeatureModel content`。
- **类型都是 internal** → 脚本里要反射：`VF.Model.VRCFury` / `VF.Model.Feature.FixWriteDefaults` / 嵌套枚举 `FixWriteDefaultsMode`（Auto = 0）。

## §4 永久解决（推荐 Auto-Fix 等价写法）

```csharp
// execute_code（CodeDom C#6：不要写 using，类型全名）
System.Type vfType = null, fwdType = null;
foreach (var asm in System.AppDomain.CurrentDomain.GetAssemblies()) {
    if (vfType == null) vfType = asm.GetType("VF.Model.VRCFury");
    if (fwdType == null) fwdType = asm.GetType("VF.Model.Feature.FixWriteDefaults");
}
var comp = avatar.AddComponent(vfType);                    // internal 类型用 AddComponent(Type)
var fwd = System.Activator.CreateInstance(fwdType);
var modeType = fwdType.GetNestedType("FixWriteDefaultsMode");
fwdType.GetField("mode").SetValue(fwd, System.Enum.ToObject(modeType, 0));  // 0 = Auto
vfType.GetField("content").SetValue(comp, fwd);
EditorSceneManager.MarkSceneDirty(scene);
EditorSceneManager.SaveScene(scene);
```

- 效果与在弹窗点 **Auto-Fix** 完全一致（`FixWriteDefaultsLater.SaveNow()` 内部就是这两步）。
- 保存后复查头像根组件里出现 `VF.Model.VRCFury`、content 的 `mode=Auto`；再进 Play **不再弹窗**。
- 只想要「别烦我、别改动画」→ 同上但 `mode = 3`（Disabled）。
- 回滚：删除该 `VRCFury` 组件即可。

## §5 验证（必须做，别只看弹窗消失）

1. 进 Play 后烘焙成功判据：descriptor 的 `expressionsMenu` 已替换为 `VRCFury Menu`、参数数正常、5 个 base layer 全部非空（本工程 `MENU=VRCFury Menu:6|PARAMS=145|LAYERS=5/5`）。
2. **全菜单 GM Play 回归**与改动前基线逐项对比（本工程 162 菜单 / 327 步 / 153 PASS / 6 个 `/ABT/` 世界功能 FAIL / 1 NOPARAM / 287 图，与 Auto-Fix 前完全一致）。
3. Console error 数不变；截图 0 粉 0 空。
4. Auto-Fix 会把所有层 WD 变 ON —— 若作者刻意用 WD OFF 做特殊效果，必须靠第 2 步全菜单回归确认没有行为变化，再保留。

## §6 边界

- 上传后游戏内的实际表现：**NOT_RUN**（编辑器内验证 ≠ 客户端）。
- 该弹窗**按头像**触发：一个工程里多个头像时，其他头像进 Play 可能各自弹；要逐个处理（或用 `mode=Disabled` 静音）。
- 本弹窗与「素材整理 / 移动资产」无关；本工程里它与既有 VRCFury HapticPlug 的 `Failed to detect plug length` 是**两个独立问题**（后者见 `02-05-unity-project-asset-reorg.md` §4）。
