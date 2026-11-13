# VRCFT 面捕「插件装了却完全没反应」：参数前缀丢失与 OSCQuery 实测诊断

> 本工作区实战 2026-11-07 · 沐梓溪（小狼）V2 · SDK 3.10.1（**无原生 Face Tracking**）· Hash「Face Tracking Control - UE Blendshapes LilLeo」预制 · 客户端 VRChat 2026.4.x + VRCFaceTracking v5

## 0. 一句话结论

**VRCFaceTracking 不靠「模型里有没有面捕动画」判断，而是靠 avatar 的 Expression Parameters 里有没有 `…/v2/<Shape>` 前缀参数。** 移植面捕层时若把参数名精简成 `JawOpen` / `MouthX` 这类无前缀简名，且 EP 里也没声明，VRCFT 就检测不到这个 avatar 支持面捕，**一条数据都不发**——菜单开关、动画层、形态键全都是好的，但整条链路没有输入。

## 1. 症状（很容易误判成「没开开关」或「客户端设置问题」）

- 游戏里「功能 → 面捕」菜单齐全，`FT On` / `FT Off` / 5 个 Toggle 都能点，但表情完全不动。
- 用户说「这个面捕插件怎么启动啊，我好像用不了」。
- 客户端 OSC 看起来是开的、VRCFT 服务也被 OSCQuery 发现了，但没效果。
- 容易先怀疑：客户端自拍面捕开关、AAO 形态键改名、Blendshape Set 缺失、缩略图。**这些都不是本例根因。**

## 2. 决定性诊断：直接读 VRChat 的 OSCQuery 参数树（强烈推荐，秒级定案）

VRChat 客户端会在本机开一个 OSCQuery HTTP 服务，端口写在启动日志里：

```
output_log_*.txt:
  Advertising Service VRChat-Client-XXXX of type OSCQuery on 61905
  Advertising Service VRChat-Client-XXXX of type OSC on 9000
  Found new OSC Service: VRCFT-550373 at 127.0.0.1:50842
```

直接请求（**必须切上当前想检查的 avatar**，参数树是实时随 avatar 变的）：

```bash
curl -s http://127.0.0.1:61905/avatar/parameters | python -c "import sys,json; d=json.load(sys.stdin); ...递归展开 CONTENTS..."
```

递归展开 `CONTENTS` 得到叶节点，就是 **VRChat 实际暴露给 VRCFT 的全部参数**（包括 `networkSynced=0` 的 local 参数，本例 `FT_Enable` 是 local 却仍出现在树里）。

本例实测结果：

```
叶节点参数总数: 374
  含 Jaw    :   0
  含 Cheek  :   0
  含 Brow   :   0
  含 Tongue :   0
  含 OSCm   :   0
  含 v2     :   1   -> /avatar/parameters/FT/v2  （空容器节点，无任何子节点）
```

> **判据**：`/avatar/parameters/FT/v2` 是空节点 = VRCFT 眼里这个 avatar 没有 Unified Expressions 参数 = 不发送。这一条就能排除「客户端没开 OSC」「VRCFT 没运行」等所有外部猜测。

配套交叉验证：`%LOCALAPPDATA%/appCymple/cache/vrcft-managed-avatar-parameters.json` 是 VRCFT 抓取的 avatar 参数清单（含 `avatarId` + 全部 address），可直接 grep `FT/v2` 是否为 0 条。

### 2.1 日志里的 `OSC enabled: False` 不要当成结论

启动摘要里这行是配置快照，**不代表 OSC 接收端没工作**。本例同时存在：

- 注册表 `UI.Settings.Osc_h1043380067 = 1`、`VRC_INPUT_OSC_h1104161515 = 1`
- UDP `0.0.0.0:9000` LISTENING、OSCQuery `127.0.0.1:61905` LISTENING
- `Found new OSC Service: VRCFT-…`

→ 客户端侧是通的。VRCFT 官方也说明它会自动把 OSC 打开。**先查参数树，再查客户端设置。**

## 3. 根因链（本例四层）

### 3.1 移植时把 Hash 官方 407 参数控制器精简成 36 参数，并丢掉了前缀

实测对比（用 UnityPy / YAML 正则读 `AnimatorController` 的 `m_AnimatorParameters`）：

| 控制器 | 参数总数 | 带 `FT/v2/` 或 `OSCm/` 前缀 |
|---|---:|---:|
| Hash 官方 `LilLeo_FX FT.controller` | 407 | 375 |
| Hash 官方 `FTOnly.controller` | 385 | 375 |
| 本工程移植版 `WolfFT.controller` | **36** | **0** |

移植版 36 个 = **28 个无前缀 Float**（`JawOpen` / `MouthX` / `CheekPuffSuckLeft` / `EyeLidLeft` …，全部只被第 0 层 Direct BlendTree 的 `m_DirectBlendParameter` 引用）+ 8 个控制开关。

来源脚本 `Assets/Editor/FixWolfV2Part2.cs:65-72` 明确写着 `// A. EP: add 8 local params`，只补了 8 个控制开关，**从未把 Hash 官方 EP 的 `FT/v2/*` 与 `OSCm/Binary/*` 合进来**。

### 3.2 VRCFT 的前缀检测规则（官方）

<https://docs.vrcft.io/docs/tutorial-avatars/tutorial-avatars-extras/parameters>

> "VRCFaceTracking uses Expression Parameters on Avatars to drive face tracking."
> "VRCFaceTracking will detect prefixed parameter categories." 合法形式：`.../v2/JawOpen`、`...ExamplePrefix/v2/JawOpen`、`...Example/Nest/v2/JawOpen`

即 **`/v2/` 是关键**，前缀可任意（Hash 用 `FT/v2/`）。没有 `/v2/` 的 `JawOpen` 不会被当成 UE 参数。

### 3.3 EP 里也完全没有这些参数

| | 源 EP `…Ver2.asset` | 烘焙后 EP（VRCFury 合并） |
|---|---:|---:|
| 总参数 | 51 | 138 |
| `FT/v2/` 参数 | **0** | **0** |
| 8 个控制开关 | `saved=0, synced=0` | `saved=0, synced=0` |

→ OSCQuery 自然不会暴露，与 §2 实测一致。

### 3.4 两道默认门还额外把面捕关着

`Assets/Hash's_Things/LilLeo/menu&params/Face Tracking Control - UE Blendshapes LilLeo.asset` 是 5 个 `type:102`(Toggle) + 1 个署名 Button：

| 菜单项 | 参数 | 官方默认值 | 含义 |
|---|---|---:|---|
| Visemes Enabled | `VisemesEnable` | 0 | 口型面捕 **默认关** |
| Eye Tracking | `EyeTrackingActive` | 1 | 眼动 |
| Lip Tracking | `LipTrackingActive` | 1 | 唇动 |
| Facial Expression Disabled | `FacialExpressionsDisabled` | 1 | **=1 时进 Mute 态，表情归零** |
| FTEffects | `FTEffects` | 1 | 特效 |

`WolfFT.controller` 的 `FT Expression Gate` 层：`If(FacialExpressionsDisabled) -> Mute`，Mute 态播归零 clip。所以即使数据进来了，默认也是静音。

> 注：`FacialExpressionsDisabled=1` 是 **Hash 官方 EP 原值**，不是误改；但本工程把 8 个开关都做成 `saved=0/synced=0`，用户手动关掉后重穿头像会复位，体验上等于「永远关着」。

## 4. 修复（本例实做，位宽安全）

### 4.1 位宽约束先算清楚

Hash 官方 EP `LilLeo_ExpressionParameters FT.asset` 有 **178 个参数**：`FT/v2/*` 37 个（5 个 synced）、`OSCm/Binary/*` **114 个全 synced**。全部合入需 **+154 bits**；本工程烘焙 EP 已 **212 bits** → 366 > 256，**直接换官方 EP 会爆参数上限**。这就是当初精简的原因，不能简单"换回去"。

设计取舍：**只恢复本地面捕能力，远端同步暂不补**（OSCm 二进制位另行规划）。

### 4.2 控制器：28 个数据参数加 `FT/v2/` 前缀

权威名单 = Direct BlendTree 的 `m_DirectBlendParameter` 行（28 个），**不要**用「所有 Float 参数」——那会多带上 `EyeTrackingActive` / `LipTrackingActive` / `FTEffects` 这 3 个控制开关，它们按 VRCFT 官方规范就该是无前缀简名。

精确替换两类整行（每个名字在 controller 里恰好出现 2 次，可安全断言）：

```
- m_Name: JawOpen                      -> - m_Name: FT/v2/JawOpen
m_DirectBlendParameter: JawOpen        -> m_DirectBlendParameter: FT/v2/JawOpen
```

本例断言 `参数声明 28 处 / directBlend 28 处` 才落盘。**切勿全局字符串替换**：这些名字同时出现在 clip 文件名（`FT_JawOpen.anim`）和曲线名（`blendShape.JawOpen`）里。

### 4.3 EP：补 28 个 local Float + 修 8 个开关

新增（每个 0 bits，local 不占同步预算）：

```yaml
  - name: FT/v2/JawOpen
    valueType: 1          # Float
    saved: 0
    defaultValue: 0
    networkSynced: 0      # local：OSC 可写、OSCQuery 会暴露，但不占 bits
```

> 关键认知：**`networkSynced=0` 的参数仍会被 OSCQuery 暴露、仍可被 OSC 写入**。本例 `/avatar/parameters/FT_Enable`（local）就在树里。EP 声明的作用是「让参数存在」，不是「必须同步」。

8 个控制开关按 Hash 官方恢复，并修正两道门：

| 参数 | def | saved | synced |
|---|---:|---:|---:|
| `FT_Enable` | 1 | 1 | 1 |
| `FTOn` / `FTOff` | 0 | 0（脉冲键，不能 saved） | 1 |
| `EyeTrackingActive` / `LipTrackingActive` / `FTEffects` | 1 | 1 | 1 |
| `VisemesEnable` | **0 → 1** | 1 | 1 |
| `FacialExpressionsDisabled` | **1 → 0** | 1 | 1 |

位宽：新增 3 个 Float×8 + 5 个 Bool×1 = **+29 bits** → 源 EP 71→**100 bits**，烘焙 EP 212→**241 bits ≤ 256** ✓

### 4.4 Unity 内权威校验（别只信文本正则）

通过 Unity MCP `execute_code` 跑真实 API：

```csharp
AssetDatabase.Refresh(ImportAssetOptions.ForceUpdate);
var ep = AssetDatabase.LoadAssetAtPath<VRCExpressionParameters>(".../…Ver2.asset");
// 统计 parameters.Length / FT/v2/ 数 / synced 数 / bits / 各开关 def+saved+synced
var c = AssetDatabase.LoadAssetAtPath<AnimatorController>(".../WolfFT.controller");
// 统计 c.parameters 里 FT/v2/ 数
```

本例回显：`EP total=79 ftv2=28 synced=51 saved=49 bits=100 | VisemesEnable=def1/savedTrue/syncedTrue FacialExpressionsDisabled=def0/savedTrue/syncedTrue | Controller total=36 ftv2=28` ✓

### 4.5 顺带修掉一个会阻断上传的编译错误

`Assets/Editor/MeowUpload.cs:126` 里 `after?.UnityPackages` —— **`VRCAvatar` 是 struct，不能用 `?.`**，报 `error CS0023`，整个 `Assembly-CSharp-Editor` 编译失败，上传脚本（以及本轮新增的 `AlignStalePackages()`）全都跑不起来。改成 `after.UnityPackages == null`。

> 与第 155 轮的坑同源：`VRCUnityPackage` 也是 struct，不能写 `p == null`。**SDK API 模型里这两个都是值类型，凡是 `?.` / `== null` 都要先确认。**

`AssetDatabase.Refresh(ForceSynchronousImport)` + `CompilationPipeline.RequestScriptCompilation()` 后，程序集时间戳更新、`Editor.log` 无 `error CS` 才算通过。

## 5. 验证清单（按这个顺序，别跳步）

1. 构建产物 `.vrca` 用 UnityPy 读 `VRCExpressionParameters`：应有 28 个 `FT/v2/*` 且 `networkSynced=0`，8 个控制开关 `saved=1/synced=1`，两个默认值 0/1 正确，总 bits ≤256。
2. 上传后确认服务端 `unityPackages` 所有 `standalonewindows/standard` 档都指向同一新 asset（走 `MeowUpload.AlignStalePackages()`，见第 155 轮）。
3. **退出并重开 VRChat → 切上该 avatar → 再查 OSCQuery**：`/avatar/parameters/FT/v2` 下应出现 28 个叶节点，`TYPE:"f"`。
4. VRCFT 面板应显示已检测到 avatar 的 UE 参数；动嘴/眨眼时 OSCQuery 里对应节点 `VALUE` 实时变化。
5. 最后才看模型表情是否跟随；若参数有值但表情不动，再去查 AAO 形态键改名与曲线绑定（另一条线索，本例未走到）。

## 6. 排除掉的假设（避免下次重复劳动）

| 假设 | 实测结论 |
|---|---|
| 客户端 `VRC_TRACKING_ENABLE_SELFIE_FACE_TRACKING=0` | 那是 **原生 Face Tracking**（需 SDK beta + Blendshape Set）；本工程 SDK 3.10.1 无该区块，Hash 走的是 VRCFT OSC 路线，二者无关 |
| 没有 Face Tracking Blendshape Set | 同上，原生系统才需要；VRCFT 路线靠参数 + 动画层，不需要 Set |
| AAO 把 ARKit 形态键改名成 `AAO_Merged_*` 导致失效 | 改名确实存在，但烘焙包里 36 个 `FT_` clip 都在（名字带 `rebased Copied from …` 前缀，别误判成「0 个 clip」）；且曲线是 optimized binding（hash，读不到明文 attribute），**不能据此判定曲线没重写**。参数树为空是更靠前的断点 |
| 烘焙包里 0 个 `FT_` clip | 误判：clip 名是 `rebased Copied from LilLeo_FX_Wolf/FT_CheekPuffLeft`，`startswith("FT_")` 当然为 0 |
| `OSC enabled: False` | 见 §2.1，UDP 9000 / OSCQuery 61905 都在监听，VRCFT 服务已被发现 |

## 7. 可复用技巧

- **UnityPy 读烘焙包 AnimationClip 的 `get_raw_data()` 看不到 `blendShape.` 明文**：烘焙后是 optimized clip，attribute 走 `m_ClipBindingConstant` 的 hash；想看曲线名请读**源工程 `.anim`**（明文 `attribute: blendShape.JawOpen`）或用 Unity `AnimationUtility.GetCurveBindings()`。
- **VRChat 客户端日志 + OSCQuery + VRCFT 参数缓存三件套**：`output_log_*.txt` 找端口，`http://127.0.0.1:<port>/avatar/parameters` 看暴露面，`vrcft-managed-avatar-parameters.json` 看 VRCFT 抓到了什么。
- **参数名带 `/` 是合法的**（Hash 官方就是 `FT/v2/XXX`），Unity Animator / VRChat EP / OSCQuery 都支持，OSCQuery 里会呈现为层级节点。
- **`FTOn` / `FTOff` 是 Button 脉冲**：置 1 后自动回落 0，配 `IfNot` 回 Waiting；`saved` 必须为 0，否则状态机卡死。
- 直改 `.asset` / `.controller` YAML 后，Unity 后台不一定自动导入；用 MCP `execute_code` 主动 `Refresh` + 回读 API 校验，比看文件 mtime 可靠。
