# 第三方工具预制件（ABT 等）的骨骼绑定点适配

> 适用：ABT（Avatar Basic Tool）、笔 / 地图 / 传送 / 小道具这类「工具类 prefab」，内部用
> `ModularAvatarBoneProxy`（下称 BP）把物件挂到标准人形骨（`RightHand`、`RightIndexDistal`、`Spine` …）。
> 来源：本工作区 2026-10-28（第 114 轮）修复「沐梓溪狼 ABT 工具没绑对地方」的实测记录。

## §1 症状

- 把工具 prefab 直接 `Instantiate` 到新角色根节点下，**工具全部飘在头顶/身体外**，不在手上。
- 本工作区实测：狼身高 ≈0.88 m、`Head` 世界 y=0.641、`Hand.R` y=0.582；
  而 BP 子物体 `Pen_Setting` 世界 y=**1.184**（= prefab 按 ~1.6 m 标准身高作者化的坐标）→ 飘在头顶 0.5 m。
- 编辑器里把 `AvatarPen` / `PenUI` 的 `SetActive(true)` 能看到「一个巨大的调色盘/笔」，也是同一原因（尺寸按标准身高）。

## §2 原理（为什么「绑」= 摆位）

| `attachmentMode` | 烘焙行为 | 适用 |
| --- | --- | --- |
| `AsChildKeepWorldPose` | 取对象**当前世界位姿**，挂为目标骨子物体后长期跟随 | 手握道具（笔/餐具/武器） |
| `AsChildAtRoot` | 位置相对 avatar 根解释 | 世界固定/贴地类（轨迹、脚印） |

结论：`KeepWorldPose` 类绑定点**完全由 prefab 实例在世界里的初始位置决定**。
「绑对地方」= 调实例的 `localPosition` / `localScale`，让 BP 子物体世界位置落在目标骨上；
改 BP 的 `boneReference`（humanoid 引用）不需要动。

## §3 做法（本工作区定版，chibi 角色）

1. 读目标骨世界坐标：`Hand.R`、`IndexDistal.R`、`Spine`。
2. 读 prefab 里两个约束点的局部坐标：`Finger_Setting`（BP→`RightIndexDistal`）、`Pen_Setting`（BP→`RightHand`；`UI` 与它同位置）。
3. 选统一缩放 `s`：chibi 取 **0.6**（≈ 目标身高 0.88 / 作者身高 1.5 量级；0.55–0.65 都可用）。
4. 让 `Finger_Setting` 精确落点（它是最硬的约束）：
   `abt.localPosition = IndexDistal.R.position − s × Finger_Setting.localPosition`
5. `WorldFixed.localPosition = −abt.localPosition / s`（保证世界固定物件仍在原点）。
6. 校验 `Pen_Setting` 世界位置 ≈ 手骨 + 前向偏移（本工作区 Δ=(0.029, 0.002, 0.054) m）。
7. `localScale=s` 会让笔、调色盘、地图 UI 一起等比缩小 → chibi 比例正常。
8. 菜单：`ModularAvatarMenuInstaller.installTargetMenu` = 目标子菜单（如 `狼_功能_ABT`），
   `menuToAppend` = ABT 的 `Expression/SubMenu/MenuList.asset`；ABT 自带 4 个
   `ModularAvatarMergeAnimator`（Base / FX_Tracking / Gesture / FX，`pathMode=Absolute`）必须保留。

## §4 烘焙后判据（唯一可靠）

在 `nadena.dev.ndmf.AvatarProcessor.ProcessAvatarUI` 的克隆里检查：

| 检查 | 期望 |
| --- | --- |
| `Pen_Setting.parent.name` | `Hand.R` |
| `Pen_Setting.position − Hand.R.position` | ≈ (0.03, 0.00, 0.05) m |
| `Finger_Setting.parent.name` | `IndexDistal.R` |
| `Finger_Setting.position − IndexDistal.R.position` | ≈ (0, 0, 0) |
| 反例 | parent 仍是 `ABT` → BP 没生效；`Pen` 世界 y≈1.18 → 仍是 prefab 原始坐标 |

## §5 踩坑

- `SetActive` 打开 `AvatarPen`/`PenUI` 只影响编辑器静态态；**实际显隐由 ABT 自己的 FX 动画层控制**，默认关闭。
- 只改父节点 `localScale` 不改 `localPosition` → Finger 会沉到地下（必须按 §3.4 重解位置）。
- 同一骨架的另一角色已调好的偏移可以直接复制（本工作区先复刻猫的 `(-0.147, -0.603, -0.039)`，再按 chibi 缩放重解）。
- BP 的目标骨是 **humanoid 引用**（`RightHand`）而不是名字，换骨架时只要 humanoid 映射有效就能复用。
