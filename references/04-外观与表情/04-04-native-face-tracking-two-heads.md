# 原生 Face Tracking（VRChat SDK 新系统）：换头/双头头像怎么接

> 本工作区只读调研 2026-10-26 · 结论来源 = VRChat 官方 beta 文档 + 本工程实测盘点；**未在本工程实装**，所有结论标「未验证实装」。

## 1. 适用版本（关键前提）

- 新的 **Face Tracking** 区块是 **Avatars SDK beta** 功能：官方在 2026-09-24 的 Developer Update 宣布进入 Open Beta（客户端）+ SDK beta；稳定版 **3.10.5（2026-09-04）还没有**。
- 本工程 `Packages/com.vrchat.avatars` = **3.10.1**，`VRCAvatarDescriptor` 反射确认**没有任何 Face Tracking 字段**（只有 `enableEyeLook` / `customEyeLookSettings` / `VisemeSkinnedMesh` / `VisemeBlendShapes`）。
- ⇒ 想用原生面捕必须升级 VRCSDK 到带该区块的 beta 版本；升级后要整体重跑烘焙回归（本工程有 VRCFury / MA / 自建 FX 层 / 大量形态键动画）。

## 2. 新系统能做什么（官方 beta 文档要点）

来源：<https://vrc-beta-docs.netlify.app/avatars/face-tracking/>

- **多个网格**：`Face Tracking Meshes` 最多 **16 个 SMR**，按「同名形态键」同时驱动 → **双头头像可以把猫头 + 狼头都挂进去**。
- **Blendshapes Set**：每个 Set = 一组「Tracking Key → 形态键名 + Multiplier」。可以按角色/风格做多套，`Default Weight` 0–1；
  用 **animator state behavior**（`VRC Animator Face Tracking Blendshapes Set`）或 **Parameter Behavior（Apply Blend Set Weight）** 切换权重。
  - 权重可被动画层按 `HeadType` 切换 → 猫头模式用 cat set、狼头模式用 wolf set（隐藏的那个头也会被驱动，但看不见，无副作用）。
- **映射可以指到任意现有形态键**：不是必须用 ARKit/Unified 命名；`Auto-Fill Blendshapes` 按名字猜，猜不到就手动从下拉里选。⇒ 自定义命名的模型（Kipfel 的 `eye_*`/`mouth_*`、LilLeo 的 `v_*`/`brow_*`）可直接映射，**不必先给网格改名**。
- **视素**：`LipSync → Mode = Viseme Blend Shape Face Tracking` 时视素也能多网格 + 每套映射 + **Viseme Dampen**（和面捕嘴型互相阻尼）→ 可以取代「手搭 15 态 Viseme 动画层」的做法，并解决「原生 viseme 只能指一个网格」的限制。
- **眼动**：`Eye Gaze Mode = Bone Rotation / Face Tracking / Both`；`Eyelids Type = Face Tracking` 时用 `EyeClosedLeft/Right` 映射眨眼。
- **追踪源**：PC 可用 **Selfie Expression（普通摄像头）**，或 **OSC**（`/tracking/face/v1`，Unified Expressions blob；iPhone/Quest Pro/Pico 4 Pro/VRCFaceTracking 都走这个口）。
- **同步**：面捕 tracking 数据走原生同步，**不占自定义同步参数预算**；远端用同样映射重算。代价进 Performance Rank 的 `Face Tracking Operations` / `Forwarded Parameters`。

## 3. 本工程盘点（实测，2026-10-26）

| 项 | 猫头 `Body` | 狼头 `狼头/Body` |
|---|---|---|
| 形态键数 | 372 | 208 |
| 眼/眨眼相关 | 188（`eye_close(_left/_right)`、`eye_joy*`、`eye_open*`、`eye_rare` 等，**无 look 方向键**） | `eye_blink(_L/_R)`、`eye_look_L/R/UP/DW`、`eye_look_UP_VRC/DW_VRC` |
| 眉 | 52（`eyebrow_*`） | 46（`brow_*`） |
| 嘴 | 71（`mouth_a/i/u/e/o`、`mouth_smile*`、`vrc.v_*` 视素等） | 37（`mouth_*` + `v_*` 视素） |
| 特殊 | `option_cheek1..4`；无 jaw/tongue | `nose_*` 6；无 jaw/tongue |

- 描述符：`enableEyeLook=False`（猫眼是**形状驱动**的二次元眼，没有真实瞳孔转动）、`lipSync=VisemeBlendShape`、`VisemeSkinnedMesh=Body`（单网格，狼头靠自建动画层）。
- 结论：**覆盖到「眨眼 + 眼型 + 嘴型 + 眉」没问题**；`JawOpen / TongueOut / CheekPuff / JawSide` 这种 ARKit 细项两个头都没有现成形态键，想接近真人级要在 Blender 里补形状键（这是主要工作量）。

## 4. 三条路线与代价

| 路线 | 前提 | 优点 | 代价/风险 |
|---|---|---|---|
| **A. 原生 Face Tracking（推荐）** | 升级 VRCSDK 到含该区块的 beta | 多网格、Set 切换、视素多网格 + Dampen、面捕数据不占同步参数预算、摄像头即可用 | SDK beta（接口可能变；上传/测试需 beta 环境）；升级后烘焙全量回归；现有表情菜单需与面捕叠（限幅/Set 权重）避免打架 |
| **B. 旧 VRCFT prefab + 参数/动画** | 现在 3.10.1 即可 | 不升级 SDK；可自定义任意映射 | 面捕套件通常 ~160 参数，会把 256 上限顶爆（本工程烘焙 EP 已 186 项）→ 必须先大砍参数；且 OSC 参数本地、**远端看不到表情**（要远端可见还得自己同步） |
| **C. 半套（只眨眼/眼动/嘴型阻尼）** | 任一 SDK | 成本最低 | 质量上限低，仍然要处理映射与 Set/动画 |

## 5. 若走 A，最小实装步骤（未验证）

1. 备份工程（7z 整包）→ 升级 VRCSDK beta → 打开 `VRCAvatarDescriptor ▸ Face Tracking ▸ Enable`。
2. `Face Tracking Meshes` 加 `Body`（猫头）+ `狼头/Body` 两个 SMR。
3. 建两套 Blendshapes Set：`CatFace` / `WolfFace`，`Default Weight` cat=1、wolf=0；手动映射：`EyeClosedLeft→eye_close_left`、`EyeClosedRight→eye_close_right`、
  `JawOpen→mouth_a`、`MouthSmileLeft/Right→mouth_smile*`、`BrowInnerUp→eyebrow_*`；狼头同理用 `eye_blink_L/R`、`v_*`、`brow_*`、`eye_look_*`。
4. 动画层加 `VRC Animator Face Tracking Blendshapes Set` state behavior：`HeadType=1` → CatFace 权重 1 / WolfFace 0；`HeadType=2` 反之。（本工程 `HeadType` 是 Int 1/2，烘焙后可能转 Float，需按烘焙后类型写条件。）
5. 可选：`LipSync Mode = Viseme Blend Shape Face Tracking`，两套 Set 里各配 15 个 viseme 映射，取代手搭 Viseme 层；开 `Viseme Dampen`。
6. 验收：桌面开 **Selfie Expression**（摄像头）→ 眨眼左右独立、四向眼动、张嘴/微笑/撅嘴跟随；另一客户端看远端表情；`HeadType` 切换后表情跟随正确；Performance Rank 不掉到 Very Poor。

## 6. 已知坑（来自 KB 旧文与工程经验）

- 面捕套件 merge FX 后常见「脸不见了 / 表情菜单失效 / 参数爆表」——新系统虽不走参数，但**动画层与 Set 权重会互相覆盖**，务必用 Set 权重/限幅而不是同时硬写同一形态键。
- 猫眼是形状驱动：`Bone Rotation` 眼动模式对它无效（没有可转的眼球），要眼动只能用 `Face Tracking` 映射 look 形状（本工程猫头暂无 look 方向键 → 需补键或用眼型近似）。
- 三白眼：`EyeIrisSmall` 与 `EyeWide` 叠加，需要在映射里把缩瞳设 0、`EyeWide` 上限压到 ~25（见 `references/00-基底/00-04-face-tracking.md`）。
- VRCFT 校准：`max` 越小越灵敏；`continuousCalibration` 不会初始化 max=0 的键（同上）。

## 9. 后续（待用户硬件到位）

- 用户 2026-10-26 说明：**已有第三方面捕设备，尚未到手**，到货后再继续本主题 → 此条保留为待办，不要当成已决策。
- 届时优先评估「路线 D（VRCFaceTracking + OSC + 自建动画层核心子集）」的成本，再决定是否升级 beta SDK 走原生 FT（路线 B）；需用户告知具体设备型号（决定是否支持眼动 / ARKit 口径）。
