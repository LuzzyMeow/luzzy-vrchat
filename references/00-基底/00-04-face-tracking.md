# 面部追踪（VRCFaceTracking + 头显侧设置）

---

## 硬件侧：Pico 4 Pro + Pico Connect

Pico Connect 的 `settings.json`（通常在 `%AppData%\Pico Connect\settings.json`）：

```json
{
  "faceTrackingMode": 1,
  "faceTrackingTransferProtocol": 2
}
```

**只在 VRCFT 完全关闭时改这个文件。** VRCFT 退出时会写 `LocalSettings.json`，覆盖掉运行期间的外部修改。

VRCFT 模块：在 VRCFT 内置的 Module Registry 里装 `Pico4SAFTExtTrackingModule`。

Quest Pro 走官方模块，流程类似。

---

## VRCFT 校准：归一化到底怎么算

VRCFT 输出的是归一化值：

```text
normalized = raw / max
```

推论（很多人搞反）：

- **`max` 越小 = 越灵敏**（同样的 raw 值 → 更大的输出 → 表情动得更夸张）
- **`max` 越大 = 越迟钝**（要动得更多才能吃满表情）

`continuousCalibration: true` 只会**微调已经校准过的**形态键。它**不会**初始化那些 `max` 仍为 0 的形态键 —— 那些必须在一次全新校准过程中，真的做出对应表情才能录进去。

`LocalSettings.json` 结构（改之前关掉 VRCFT）：

```json
{
  "calibration": {
    "eyeWide":     { "min": 0.0, "max": 0.45 },
    "tongueOther": { "min": 0.0, "max": 0.0 }
  },
  "continuousCalibration": false
}
```

- `eyeWide.max = 0.45` → 睁大眼更灵敏（范围收窄），同时也**限制了输出上限**，是治"三白眼"的一环。
- `tongueOther.max = 0` → 关掉舌头的翻卷/扭转动作。

### Parameter Adjustment

VRCFT 界面里的 `Parameter Adjustment → Item1 / Item2` 是**逐参数的幅度上限（乘数）**，和校准的 `max` 是两套东西。校准完之后用它做精调。

---

## Avatar 侧：面捕套件

两类常见方案：

### Prefab 直装型

某些面捕套件是 prefab 形式，把对应头显版本的 prefab 拖进 avatar 就完事，FX 和参数都在里面。

检测是否已安装：

```csharp
bool installed = avatar.GetComponentsInChildren<Transform>(true)
                       .Any(t => t.name.Contains("<套件关键字>"));
```

### 向导生成型

另一些是编辑器向导：选中 avatar → 选头显档位（例如 `HD(Quest,Pico)`）→ apply → 生成一个 prefab variant。

**建议开 `duplicate = true`**，保留原始 avatar 不被就地改写。

**常见坑**：如果 avatar 是 prefab instance，向导自动检测 FBX 版本可能返回 null 直接 NRE。绕过方式是反射注入 —— 从 `allFbxInfo.GetFbxVersions()` 里找 `m_BlendShapeData` 名字匹配的那个版本，手动赋给 mesh patcher 的 `currentFbxVersion` 字段。

---

## 编辑 `.anim` YAML 的陷阱：value 行在 attribute 行**之前**

Unity 的 `.anim` 文件里，`m_FloatCurves` 的结构是这样的：

```yaml
m_Curve:
- serializedVersion: 3
  time: 0
  value: 75                      # ← 值在这里，在 attribute 之前
  inSlope: 0
  ...
attribute: blendShape.EyeWide    # ← 属性名在后面
path: Body
```

用 `awk` / PowerShell 逐行处理时：**记住最后一个 `value:` 行的位置**，等匹配到对应的 `attribute:` 行时再回头改那一行。不要假设 value 在 attribute 后面。

```powershell
$lines = Get-Content $file
$lastVal = -1
for ($i = 0; $i -lt $lines.Count; $i++) {
    if ($lines[$i] -match '^\s+value:\s') { $lastVal = $i }
    if ($lines[$i] -match 'attribute: blendShape\.EyeWide' -and $lastVal -ge 0) {
        $lines[$lastVal] = $lines[$lastVal] -replace '(value:\s*)\S+', '${1}25'
        $lastVal = -1
    }
}
[IO.File]::WriteAllLines($file, $lines)
```

---

## 三白眼（眼白露太多）

**根因是两个 blendshape 叠加**：

- `EyeIrisSmall` —— 缩小瞳孔/虹膜，露出更多眼白
- `EyeWide` —— 上下眼睑回缩，虹膜上下露出更多眼白

**修法**：

1. 面捕动画 clip 里把 `EyeIrisSmall` 直接设为 `0`（缩瞳几乎没有好看的时候）。
2. `EyeWide` 的最大值从 100 降到 ~25。
3. VRCFT 的 `LocalSettings.json` 里把 `eyeWide.max` 调到 `0.45` 左右，从源头限制归一化输出。

三条同时做，问题基本消失。

---

## 面捕与 FX 的冲突

面捕套件通常会 merge 一整套 FX layer 进去。如果装完面捕之后：

- **脸不见了** → 检查面捕 merge 是否覆盖了原有 FX 里控制脸部 renderer 的 layer。
- **表情菜单失效** → 检查参数名冲突（面捕参数和原有表情参数同名）。
- **参数爆表** → 面捕参数量很大，装完必须重新算 synced bit budget，见 [00-02-modular-avatar.md](00-02-modular-avatar.md) 的参数预算一节。

---

## 补充：官方语义与结构化分册路由（2026-11-11 联网核实，原文保留）

### VRCFT 眼部 blendshape 官方口径

来源：VRCFaceTracking 官方文档 <https://docs.vrcft.io/docs/v4.0/tutorial-avatars/tutorial-avatars-extras/eye-blendshapes>（v4.0 页面，v5.0 为当前版）：
- **眼部追踪 blendshape 用于「非眼球」部分**；**眼球旋转应权重到骨骼 transform 上做动画**，而不是靠 blendshape 转眼球。
- SRanipal 眼追可控制 18 个键：`Eye_Left/Right_squeeze`（Blink=100 时紧闭）、`Eye_Left/Right_Blink`（值越大越闭）、`Eye_Left/Right_Left/Right/Up/Down`（眼周肌肉向对应方向移动）、`Eye_Left/Right_Wide`（睁大，应在对侧 Blink=0 时做）、`Eye_Left/Right_Dilation/Constrict`（瞳孔放大/缩小）。
- **眼睑极性**：VRCFT/Pawlygon 口径 `EyeLid` 0=闭 / 0.75=正常睁 / 1=睁大，而日系 `EyeClosed*` 键 100=闭——直连必反相，完整修法（1D 混合树 0/0.8/1.0 三档 + 默认值 0.8）见 [04-08 §3.1–§3.2](../04-外观与表情/04-08-face-tracking-osc-and-repair.md)。

### 两套架构的分册路由

| 架构 | 分册 |
|---|---|
| OSC / VRCFT + FX 动画层（接线、极性、OSCmooth、补丁器、AAO 交互、网格迁移） | [04-08](../04-外观与表情/04-08-face-tracking-osc-and-repair.md) |
| 原生 Face Tracking（SDK 3.10.6+：descriptor 字段、103 键映射、gaze、专用键、高光） | [04-09](../04-外观与表情/04-09-face-tracking-native-sdk.md) |
| 「装好却完全没反应」参数前缀丢失 + OSCQuery 诊断 | [04-07](../04-外观与表情/04-07-vrcft-parameter-prefix-repair.md) |
| 原生 FT 早期调研（双头/换头） | [04-04](../04-外观与表情/04-04-native-face-tracking-two-heads.md) |

官方入口：<https://creators.vrchat.com/avatars/face-tracking/>（Blendshapes Set、Tracking Key 共享、Multiplier、Default Weight）、<https://creators.vrchat.com/avatars/face-tracking/osc>（gaze 8 单极通道）。原生 FT 自 Avatars SDK Beta（开发者更新 2026-09-24）起提供；SDK 3.8.2 没有该面板。
