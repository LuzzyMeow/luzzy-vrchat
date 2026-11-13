# MA 菜单：发型切换 + 发色切换（本工作区实测搭建）

> 状态：2026-10-18 本工作区在 `projects\LuzzyMeow` 的 Kipfel 上搭好并 NDMF bake 验证；**运行时（Play/上传）行为未在 Gesture Manager 验证（NOT_RUN）**。
> 目标：一套菜单里 ① 三套发型单选切换；② 6 种发色单选，且**任何发型**都能套用所选颜色。

## 1. 结构总览

| 部件 | 内容 |
|---|---|
| 参数 | `HairStyle`（Int，saved，默认 1）、`HairColor`（Int，saved，默认 0）——用 `ModularAvatarParameters` 声明（字段 `nameOrPrefix` / `syncType`=Int / `saved` / `hasExplicitDefaultValue` / `defaultValue`） |
| 菜单 | 一个 `VRCExpressionsMenu` 资产（`menu_root`，含两个 SubMenu：`发型`、`发色`）；子菜单控件用 **Button + parameter + value** 实现单选；`MA MenuInstaller.menuToAppend = menu_root`、`installTargetMenu = null`（装到根菜单） |
| 动画 | 自建 `AnimatorController`（2 层）：`HairStyle` 3 状态、`HairColor` 6 状态；状态间用 Int 参数 `Equals` 过渡（`duration=0`、`hasExitTime=false`） |
| 合并 | 头像根加 `MA MergeAnimator`：`animator=controller`、`layerType=FX`、`pathMode=Absolute`、`mergeAnimatorMode=Append`、`deleteAttachedAnimator=false` |

## 2. 发型切换：动画 m_IsActive

```csharp
var cb = EditorCurveBinding.FloatCurve("<hierarchy path>", typeof(GameObject), "m_IsActive");
var cv = new AnimationCurve(); cv.AddKey(new Keyframe(0f, active ? 1f : 0f));
AnimationUtility.SetEditorCurve(clip, cb, cv);
```

每个状态一个 clip：只让目标发型 `m_IsActive=1`，另两套 0。路径从**头像根**算起（如 A 款挂在头下：`armature/Hips/Spine/Chest/Neck/Head/Hair_LooseShort`）。

## 3. 发色切换：动画材质引用

```csharp
var cb = EditorCurveBinding.PPtrCurve("<renderer path>", typeof(SkinnedMeshRenderer), "m_Materials.Array.data[0]");
var k0 = new ObjectReferenceKeyframe(); k0.time = 0f; k0.value = colorMaterial;
AnimationUtility.SetObjectReferenceCurve(clip, cb, new ObjectReferenceKeyframe[] { k0 });
```

每个颜色状态一个 clip，把**所有三套发型**每个 SMR 的第 0 号材质都指向「该发型 × 该颜色」的材质 → 于是任何发型都能套用颜色。本项目颜色材质由「白色材质复制 + 换贴图」得到（贴图按亮度重映射：`out = color * (0.55 + 0.45*(L/255)^0.6)`，见 05-03 §10.1 同类做法），6 色 = 白/黑/灰/金/粉/蓝，共 18 个材质。

## 4. ★ 踩坑（实测）

1. **`AnimatorController.AddLayer()` 新建的层 `defaultWeight = 0`** → 颜色层不生效（bake 后 FX 里看到 `HairColor w=0`）。必须显式：`var ls = ctrl.layers; ls[i].defaultWeight = 1f; ctrl.layers = ls;`。
2. **单选不要用 Toggle**：VRChat 菜单 Toggle 写 bool；用 **Button + 参数 + value** 写 Int 参数才能真正「点一下就切换并保持」。
3. 菜单资产可以直接给 `MA MenuInstaller`（`menuToAppend`），不必手搭 `MA MenuItem` 树。
4. 层名会被合并流程保留；**默认情况下 MA 层追加在既有 animator 之后**（既有 animator 视为 `Layer Priority = 0` 且排最前）。层序由 `Layer Priority` 决定：升序排列（数字小在前、大者覆盖），同优先级按层级顺序；`Replace Existing Animator` 保留自己设的 priority，并在该 priority 内排其它 MergeAnimator 之前。**调过 priority 或用过 Replace 后就不要假设「一定在末尾」**，一律以烘焙产物实际层序为准；同时确保自己的层在动画目标（m_IsActive / m_Materials）没有冲突。（2026-11-01 按官方 Merge Animator 页补全，见 `03-07-ma-platform-sync-mmd-and-motion.md` §1.4）

## 5. 验证（本机）

- NDMF Manual Bake 后检查：FX 控制器 20 层，含 `HairStyle`(3 states)/`HairColor`(6 states)，两层权重均为 1；
  `expressionsMenu` 根 5 个控件（原 Kipfel 的 Outfit/Pet/Motion + 新增 `发型`/`发色`），参数表里 `HairStyle`/`HairColor` = **Int + saved** ✅。
- 静态组合截图（白/黑/金/粉 × 三款发型）颜色与发型显示正常 ✅；**菜单点击后的切换逻辑未跑 Play（NOT_RUN）**，下一步应在 Gesture Manager 里点一遍。

## 6. 相关

- 发型本体与贴图/材质手法：`../05-动骨与穿模/05-03-model-anatomy-and-head-swap.md` §10.1–§10.3
- 文件落点（本项目）：`Assets/LuzzyMeow/HairMenu/`（controller + clips + menu assets）、`Assets/[ Unless Heaven ]`（2A 试验已删）、`Assets/Asymmetric…`/`Assets/QW`（B/C 款发型）

---

## 7. 发色菜单「默认色显示成未打开」——参数默认值要三处一致（2026-10-27 追加）

**现象**：8 个发色项都是同一 Int 参数 `WolfHairColor` 的 Toggle（value 1..8），EP 里默认值确实是 6（蓝紫），但编辑器 Gesture Manager / 预览里 **8 项全显示 OFF**，用户以为「当前颜色没打开」。

**根因**：菜单项显示状态由**动画器参数默认值**决定，不看 EP。源 FX 控制器 `Kipfel_FXLayer_v1.1.1_沐梓溪狼.controller` 和猫的同名控制器里 `WolfHairColor` 的 `defaultFloat` 是 **0**（历史上从别的角色继承来的），
而 EP 是 6 → 编辑器里没有一项匹配 0（`WolfHairColor` 层还有 `Color_0_Fallback` 状态兜底，所以颜色看着对，但菜单没一项亮）。

**修法（三处一起改，缺一处就会「看着改了进游戏又不对」）**：

1. **播放器参数默认值**：`AnimatorController.parameters` 里把该 Int 的 `defaultFloat` 改成目标值（`var ps = ctrl.parameters; … ; ctrl.parameters = ps;` 再 `EditorUtility.SetDirty(ctrl)` + `AssetDatabase.SaveAssets()`）。
2. **EP 初值**：`Kipfel_ExParameters*.asset` 里该参数 `defaultValue` 必须同值（本例 6）。
3. **MA 参数声明**：在 `ModularAvatarParameters.parameters` 里加同名 `ParameterConfig`（`defaultValue=6`、`hasExplicitDefaultValue=true`、`saved=true`），否则 MA 烘焙时仍按自动值走。

判据：改完读烘焙产物 `VRCFury Params.asset`，`WolfHairColor` 应为 `Int def=6 saved=True`；同时菜单里 value=6 的那一项在烘焙 Menu 资产里存在。

**互斥**：同一 Int 参数 + 每个 Toggle 各自的 value 天然互斥（点 ON 写入该 value，点 OFF 写 0 → 落到 `Color_0_Fallback`）。要「永远有一项亮」，就不要把默认值留 0，并把 fallback 状态指向默认色。

> 相关：`../06-构建上传与备份/06-04-sdk-publish-with-reflection.md` §9（菜单 `subMenu` 只改内存不落盘，重启后变空）。

