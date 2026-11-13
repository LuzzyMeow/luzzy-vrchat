# SPS / 牛牛（N式）绑定与菜单重建

本工作区实测 2026-10-27（LuzzyMeow V2 / 沐梓溪狼）。适用：RimoShop SPS 全身化插件、N式スーパーよわちん、以及任何「bone proxy + 菜单 installer」型第三方成人插件在**克隆出来的新头像**上绑定错位/菜单缺失。

## 现象速查

| 现象 | 根因 | 修法 |
| --- | --- | --- |
| SPS 所有 socket（嘴/手/脚/穴）整体浮在身体**一侧**约半米 | 场景实例 `SPS` 根 `localPosition` 被改（本例 `(-0.4875,0,0)`，原 prefab 是 `(0,0,0)`） | 根 lp 归零；子物体 lp 是**按头像原点坐标**写的，根一动全体平移 |
| socket 位置在身体**内部/偏低**（手环落在手腕下方 7 cm） | 根 `localScale` 被改（本例 `0.893`；原 prefab `1`） | 根 scale 恢复 1；socket 的世界坐标应按比例回到解剖位置 |
| 只有脚的两个 socket 左右互换/一个贴地一个悬空 | 实例里 `L_Feet` / `R_Feet` 的 lp 被错误编辑（本例 L_Feet 抄了右脚值 `(0.0484,-0.0010,0.0039)`，R_Feet 多了 `+0.0542` 的 y） | 逐项比参考头像的 lp/lr：`L_Feet=(-0.0492,-0.0004,0.0091) lr=(0,230,0)`、`R_Feet=(0.0484,-0.0010,0.0039) lr=(0,180,0)` |
| 菜单里没有 SPS / 牛牛 子菜单 | 克隆时把「猫专属菜单宿主」整对象删了 → 这些根上的 `ModularAvatarMenuInstaller` 一起没了 | 用组件级复制把 installer 补回（见下） |
| 菜单有但有开关却点不动 | `VRCFuryHapticSocket.addMenuItem=false` 且 installer 的 target 指向错误菜单资产 | installer 的 `installTargetMenu` 必须指到「功能道具」这类真实存在的 VRCExpressionsMenu |

## 1. 先查 KB 与参考头像，别猜位置

`references/00-基底/00-06-plugin-ecosystem.md` §SPS 已写明：**「吸附位置不对：查 target / socket 的位置和 humanoid bone fallback。不要拿 avatar root 当默认位置。」** 本条是它的展开。

判据顺序：
1. **对照原作者工程**：本例 `小狼2.0` 是同一套骨架（Hips/Head/Hand.L/foot.L 世界坐标逐位相同），它的 `SPS` 根 `lp=(0,0,0)`、`ls=(1,1,1)`，socket 世界坐标就是正确答案。**同骨架 = 可以直接抄数值**。
2. **逐 socket 比世界坐标**，别比 lp（父级不同则 lp 无意义）。
3. 差值若与「根 lp」同向同量级 → 根位置错；差值若与坐标成比例（0.893 倍）→ 根缩放错；只有个别 socket 错 → 该 socket 的 lp/lr 被手动改过。

参考值（`小狼2.0`，SPS 根 lp=0/ls=1，同一套骨架通用）：

```
Head  wp=(0.000,0.641,0.005)   Hips wp=(0.000,0.418,0.011)
Mouth  wp=(0.000,0.647,0.068)  lr=(25,0,0)     boneProxy=Head
L_Hand wp=(-0.325,0.576,0.005) lr=(0,180,180)  boneProxy=LeftHand
R_Hand wp=( 0.325,0.576,0.005) lr=(0,180,180)  boneProxy=RightHand
L_Feet wp=(-0.049,0.000,0.009) lr=(0,230,0)    boneProxy=LeftFoot
R_Feet wp=( 0.048,-0.001,0.004) lr=(0,180,0)   boneProxy=RightFoot
Ass    wp=(0.000,0.353,-0.003) lr=(70,180,180) boneProxy=Hips
```

（`Hand.L` 在 (-0.290,0.582,0.007)：socket 比腕骨再外 3.5 cm = 掌心；`Ass` 比 Hips 低 6.5 cm = 骨盆。）

## 2. 修法与验证

```csharp
// 根归位（猫/狼各一遍）
var sps = avatar.transform.Find("SPS");
sps.localPosition = Vector3.zero;
sps.localScale = Vector3.one;          // 原 prefab 是 1，被改成 0.893 会让所有 socket 缩向原点

// 个别 socket
var lf = sps.Find("L_Feet"); lf.localPosition = new Vector3(-0.0492f,-0.0004f,0.0091f); lf.localRotation = Quaternion.Euler(0,230,0);
```

验证（**三层，缺一不可**）：

1. **数值**：打印 6 个 `SPS Socket` 的世界坐标 + 对应 humanoid 骨世界坐标，差值应在解剖位置（手≈掌心、嘴≈吻部、脚≈脚底、Ass≈骨盆）。
2. **截图识别**：给每个 socket 位置临时放一个 `PrimitiveType.Sphere`（`Unlit/Color`，0.014~0.022 m）当标记，用临时相机 + `RenderTexture` 多角度出图（正面/侧面/背面 + 手部/头部/脚部特写），
  逐张目视标记是否落在该落的地方。渲完 `DestroyImmediate` 标记与相机，并 `OpenScene` 丢弃内存改动（否则标记会留在场景里）。
3. **Windows-MCP 实看**：Unity 窗口截图 + 旋转 Scene 视图多角度确认（与截图识别互补）。

> 相机方向坑：角色正面朝 **+Z**（`Head.forward=(0,-0.088,0.996)`），所以"正面"相机要放在 `target + forward*distance`（即 +Z 侧）。写成 `target - forward*dist` 会拍到后脑勺。

## 3. 菜单 installer 重建（克隆头像常见缺口）

克隆后删「猫专属宿主」很容易把第三方插件的 installer 一并删掉。补法（组件级复制，保留原配置）：

```csharp
// 从猫（完好原件）复制到狼
var src = cat.transform.Find("SPS").GetComponent<ModularAvatarMenuInstaller>();
var dst = wolf.transform.Find("SPS").gameObject.AddComponent<ModularAvatarMenuInstaller>();
UnityEditorInternal.ComponentUtility.CopyComponent(src);
UnityEditorInternal.ComponentUtility.PasteComponentValues(dst);   // 或 PasteComponentAsNew 后取回
dst.installTargetMenu = AssetDatabase.LoadAssetAtPath<VRCExpressionsMenu>(
    "Assets/01_模型本体/Kipfel/ExMenu/角色/狼_功能道具.asset");
```

N式同理；**菜单资产要复制成角色专属副本**（`AssetDatabase.CopyAsset`），否则两个头像共享同一份菜单资产，改一个动两个。

菜单资产结构（RimoShop 原包，可对照定位）：
- `.../Quick settings_SPS/Data/Menu/Holes/SPS_MM.asset` → `SPS_M.asset`（上半身/下半身）→ `SPS_Chest`（嘴巴=1/乳交=2/左手=3/右手=4）、`SPS_Body`（穴=7/屁穴=9/足底L=5/足底R=6），全部 Toggle 到同一个 Int 参数 `SPS`。
- `.../N式スーパーよわちん/system/N_smchin_main_<角色>.asset` → `N_smchin_sub.asset`（ONOFF/DPS/SPS_ON/changeshape_A|B|C/syasei/bokki）。

N式本体绑定无需改：`N式スーパー短小包茎Hips固定化用` 上的 `ModularAvatarBoneProxy` 已经是 `boneReference=Hips`、`attachmentMode=AsChildAtRoot`，世界坐标 = Hips 世界坐标即正确。

## 4. 烘焙验收（读产物，不信编辑器 Play）

`Packages/com.vrcfury.temp/Builds/____/`：
- `VRCFury Menu.asset`：功能道具应出现 `[SPS SubMenu][眼罩 SubMenu][牛牛 SubMenu]` 三项。
- 克隆里 6 个 `SPS Socket` 世界坐标应与编辑态一致（bone proxy 烘焙后仍贴骨），另有 1 个 `SPS Plug`（本例在 `Bone.002` wp=(0,0.375,0.055)）。
- N式 6 个 SMR 默认 `activeSelf=false`（参数 `N_smchin_*` 默认 0），菜单打开后才显示 —— 属正常。

**边界（NOT_RUN）**：编辑器 Play 不初始化 `VRCAvatarParameterDriver`（`initialized=False`），SPS/牛牛的开关效果只能进游戏实测；本工作区以「烘焙产物 + 位置数值 + 截图」三层证据代替。
