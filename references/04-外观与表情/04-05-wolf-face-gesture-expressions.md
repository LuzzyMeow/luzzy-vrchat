# 小狼（LilLeo）换头表情：手势槽位图与「再加一个表情」的完整做法

> 本工作区实测 2026-10-27 · 场景 `Assets/00_场景/Lilleo2.0.unity` · 头像 `沐梓溪狼`（= 小狼2.0 的狼头 + 沐梓溪喵 的身体）

读这份文件的时机：用户说「**这个表情能加进狼头吗**」「**还有哪些手势能加表情**」「狼头表情不对/被覆盖」，或要往换头后的狼脸上加任意 `.anim` 表情。

---

## §1 现有表情系统的真实结构（先认清，别乱加层）

```
沐梓溪狼/换头FX            → ModularAvatarMergeAnimator
   animator = Assets/08_动画·菜单·图标/换头系统/LilLeo_FX_FaceOnly.controller
   layerType = FX    pathMode = Absolute    mergeAnimatorMode = Append
   deleteAttachedAnimator = true    matchAvatarWriteDefaults = true
```

`LilLeo_FX_FaceOnly.controller` 的 7 层（全部 **Override**，`defaultWeight=1`，状态 `writeDefaultValues=True`）：

| 层 | 作用 | 驱动 |
|---|---|---|
| `WolfFace FaceDefault` | 默认脸（193 曲线，`LilLeo_face_zeroshape_WolfBody`） | 常驻 |
| `WolfFace Left Hand` | 左手 8 态表情 | `GestureLeft` |
| `WolfFace Right Hand` | 右手 8 态表情 | `GestureRight` |
| `WolfFace BlinkAuto` | 自动眨眼（`eye_blink` 1 条曲线） | 常驻 |
| `WolfFace Viseme Lock` | 口型锁 | 参数 |
| `WolfFace Viseme` | 15 态口型 | `Viseme` |
| `WolfFace Voice` | Additive 语音开口量 | `Voice` |

**所有 clip 的路径都是** `armature/Hips/Spine/Chest/Neck/Head/狼头/Body`（狼头被换到身体后重写过；不要再写裸 `Body`）。
`狼头/Body` SMR = **208 个形态键**，名字是 `eye_* / brow_* / mouth_* / lower_* / v_* / ex_* / hightlight_*` 这一套（不是 Kipfel 的 `eyebrow_*`）。

---

## §2 手势槽位图（本工作区实测，全部 8×8 扫过一遍）

VRChat 手势值：`0=Neutral 1=Fist 2=HandOpen 3=Point 4=Peace 5=RockNRoll 6=Gun 7=ThumbsUp`

### 左手（GestureLeft）

| 值 | 状态 | clip | 表情 |
|---|---|---|---|
| 0 | Idle2 | 空 | 无（交给右手/默认脸） |
| 1 | Fist0 | **空** | **无 ← 空槽** |
| 2 | Open0 | `LilLeo_お_WolfBody` | 惊讶「お」 |
| 3 | Point0 | `LilLeo_へにょ_WolfBody` | へにょ脸 |
| 4 | Peace0 | `LilLeo_にぱ_WolfBody` | にぱ笑 |
| 5 | RockNRoll0 | `高超脸_WolfBody` | 高超脸 |
| 6 | Gun0 | `LilLeo_じとてれ_WolfBody` | じとてれ |
| 7 | Thumbs up0 | `饿饿_WolfBody` | 饿饿 |

### 右手（GestureRight）

| 值 | 状态 | clip | 表情 |
|---|---|---|---|
| 0 | Idle2 | 空 | 无 |
| 1 | Fist0 | **空** | **无 ← 空槽** |
| 2 | Open0 | `LilLeo_ねこ口_WolfBody` | 猫嘴 |
| 3 | Point0 | `哭哭_WolfBody` | 哭哭 |
| 4 | Peace0 | `LilLeo_ぱあ_WolfBody` | ぱあ |
| 5 | RockNRoll0 | `LilLeo_おこ_WolfBody` | おこ |
| 6 | Gun0 | `LilLeo_ねこめ_WolfBody` | 猫眼 |
| 7 | Thumbs up0 | `LilLeo_えっへん_WolfBody` | えっへん |

**结论：单手 8×2 = 16 个槽位里唯一空着的是 `Fist`（左右各一个）**。
左右同时给值时两层都会写，**后写的右手覆盖左手**（所以「双手 Peace」= 右手ぱあ，不是新表情）。
所以真正还能用的手势位：
1. **双手都是 Fist（GestureLeft=1 且 GestureRight=1）** → 本工作区已用于「唐笑」（见 §4），**推荐**：不会抢任何现有表情；
2. 单手 Fist（另一只手 Neutral）→ 也空，但握拳在抓取/交互里很常见，不建议占用。

### 另外 8 个可用槽位：Action 菜单（VRCEmote）

`[ USER EDIT ] Face` 层在狼克隆里 **`defaultWeight=0`**（VRCFury 判定「无有效动画」后置 0 —— 它的 clip 绑在已被删掉的猫头 `Body` 上），
但 `VRCEmote`（Int）参数还在、Action 菜单 1–8 也还在 → **这 8 个位置是空的**，以后要加更多表情可以走 Action 菜单（而不是继续挤手势）。

---

## §3 判定一个 `.anim` 能不能用在狼头

1. 把 `.anim` 拷进工程任意临时目录（**`AssetDatabase` 不认工程外路径**，`LoadAssetAtPath` 会返回 null）。
2. 读 `AnimationUtility.GetCurveBindings(clip)`：
   - `path` 是否为 `Body`（= 原始 LilLeo 的写法）或已是 `armature/.../狼头/Body`；
   - `propertyName` 是否全是 `blendShape.*`；
   - 形态键名是否都能在狼头 208 个键里找到（本工作区用 `HashSet` 对比，`小狼唐笑.anim` = 188/188 命中）。
3. 看 `GetAnimationClipSettings(clip).loopTime` 与长度：这类「表情预设」通常是 **单帧（length=0）静态姿势**，不要当循环动画播。
4. 对比猫头：猫头 372 键、命名是 `eyebrow_* / mouth_* / eye_joy*`，**狼头表情 clip 用到猫头会大面积缺键**（本工作区 `小狼唐笑.anim` 在猫头缺 181/188）→ 不要跨头混用。

---

## §4 「加一个新表情」标准流程（本工作区已跑通：唐笑）

### 4.1 生成狼头专用 clip

- 新建 `AnimationClip`，`frameRate = 60`，`loopTime = false`；
- 对源 clip 的每条 `blendShape.*` 曲线，用**新 binding**（`path = armature/Hips/Spine/Chest/Neck/Head/狼头/Body`、`type = typeof(SkinnedMeshRenderer)`、`propertyName` 不变）写回；
- **只保留「有值」的曲线**（本工作区取 `|value| > 0.5`，188 → **35 条**）。原因见 4.2：这样 `eye_blink` 等别的层的曲线不会被这个状态压成 0（否则加了表情就不眨眼）；
- 存到 `Assets/08_动画·菜单·图标/换头系统/clips/WolfFace_<名字>.anim`（和作者的 `*_WolfBody.anim` 同目录，便于管理）。

### 4.2 在 `LilLeo_FX_FaceOnly.controller` 末尾加一层（Override + WD=false）

```csharp
var ac = (UnityEditor.Animations.AnimatorController)rac;
ac.AddLayer("WolfFace TangSmile");               // ⚠ AddLayer(string) 返回 void，别接返回值
int idx = ac.layers.Length - 1;
var layers = ac.layers;
layers[idx].defaultWeight = 1f;
layers[idx].blendingMode = AnimatorLayerBlendingMode.Override;
ac.layers = layers;
var sm = ac.layers[idx].stateMachine;

var off = sm.AddState("Off");  off.motion = emptyClip;       // LilLeo_Empty_WolfBody
var on  = sm.AddState("On");   on.motion  = tangClip;
off.writeDefaultValues = false;  on.writeDefaultValues = false;   // ★关键
sm.defaultState = off;

var tOn = sm.AddAnyStateTransition(on);
tOn.hasExitTime = false; tOn.duration = 0.12f; tOn.canTransitionToSelf = false;
tOn.AddCondition(AnimatorConditionMode.Equals, 1f, "GestureLeft");
tOn.AddCondition(AnimatorConditionMode.Equals, 1f, "GestureRight");

var tOff1 = sm.AddAnyStateTransition(off);  // 两个「或」条件要两条转移
tOff1.AddCondition(AnimatorConditionMode.NotEqual, 1f, "GestureLeft");
var tOff2 = sm.AddAnyStateTransition(off);
tOff2.AddCondition(AnimatorConditionMode.NotEqual, 1f, "GestureRight");
```

**为什么要 `writeDefaultValues=false`**：这层是最后覆盖层，OFF 态必须是「什么都不写」，否则 WD=true 会把下层手势表情全部重置成 0。
**为什么 Off 用空 clip、On 只写有值的曲线**：OFF 不干扰；ON 只覆盖表情需要的键，`eye_blink` 等仍由下层眨眼层继续写 → 有表情也能眨眼。

### 4.3 验证（Play 烘焙 + 数值断言，本工作区实测有效）

1. `manage_editor play` → 等 NDMF 烘焙（约 100 s）→ 读 `Packages/com.vrcfury.temp/Builds/____/VRCFury FX.controller`：
   - 应有 `WolfFace <名字>` 层、`blendingMode=Override`、`defaultWeight=1`、ON 态 clip 曲线数 = 保留数；
2. 给 Play 里的烘焙克隆手动挂这个 baked controller（`Animator.runtimeAnimatorController = baked; Rebind();`）——**编辑器 Play 自动烘焙的克隆 `runtimeAnimatorController` 是 null**，不挂它就不会动；
3. 数值断言（编辑器 Play 是**黑剪影**，不能截图判断，只能读权重）：
   ```csharp
   an.SetInteger("GestureLeft", 1); an.SetInteger("GestureRight", 1);
   for (int i = 0; i < 60; i++) an.Update(0.03f);          // 过渡 0.12s，要多跑几帧
   head.GetBlendShapeWeight(head.sharedMesh.GetBlendShapeIndex("brow_smile"))  // 期望 100
   ```
   本工作区实测：中性 = 0；双手握拳 = `brow_smile=100 ex_blush=100 mouth_he=60`；松开 = 回到 0 ✅
4. **回归断言**（证明没破坏原有表情）：把 `GestureLeft` / `GestureRight` 分别扫 2–7，记录 208 个形态键里变化数与名称（本工作区每个手势 5–19 个键变化，全部正常）；
5. 退出 Play 后：还原 5 个 `NyaPPu__Generated/LightController/*.anim`（构建/Play 会重写），删临时目录，删重生 `.sln`。

### 4.4 三个坑（本工作区实测）

1. **别用 Additive 层做这个**：作者自己的 `WolfFace Voice` 是 Additive，但在「手动挂 baked controller + `Animator.Update`」的验证里，Additive 层的形态键**读出来始终是 0**（Override 立即可见）→ 想可验证就用 Override + WD=false。
2. 烘焙后 `matchAvatarWriteDefaults=true` 会把状态 WD 回写成 `True`（源里是 false），但**行为仍然正确**（松开回 0、手势照旧）——验收以数值为准，不要只看 WD 字段。
3. `AnimatorController.AddLayer(string)` 返回 **void**（不是 state machine）；`AnimatorStateMachine.AddState(...)` 返回 `AnimatorState`。
  CodeDom 里 `AnimatorController/AnimatorState` 等类型要写全 `UnityEditor.Animations.*`。
4. `AssetDatabase.DeleteAsset(...)` 会触发安全策略拦截（`Blocked pattern detected`）→ 用「唯一新文件名」而不是删了重建。

---

## §5 边界

- 以上「双手握拳 = 唐笑」的触发**只在 VRChat 客户端里由手势驱动**；编辑器只能验「Animator 参数 → 形态键」这一段，**真机手势 → 表情 NOT_RUN**。
- 想让表情不依赖手势、能菜单开关：再给同一层加一个 `Bool` 参数（MA Parameters + MenuItem），ON 条件改成「手势**或**参数」→ 需要两条 AnyState 转移，别把条件塞进同一条（同一条是 AND）。

## §6 换头后口型（LipSync）描述符：名字必须与「所选网格」的形态键一致（本工作区实测 2026-10-27，第 110 轮）

### 6.1 症状
- 换头后（猫头→狼头）说话时嘴巴完全不动，表情菜单/手势都正常。
- `VRCAvatarDescriptor` 的 `lipSync = VisemeBlendShape` 看着没问题，`VisemeSkinnedMesh` 也指着一个真实存在的 SMR。

### 6.2 根因
描述符里的 **15 个 viseme 名字 + `MouthOpenBlendShapeName` 是「猫头」的命名，而 `VisemeSkinnedMesh` 指到了「狼头」网格**。
本工作区实测：`VisemeBlendShapes` = `vrc.v.sil … vrc.v.ou`（Kipfel 猫头命名），`MouthOpen = Facial_Blends.Jaw_Down`；
而狼头 `狼头/Body` 的 208 个形态键里真实名字是 `v_sil, v_PP, v_TH, v_FF, v_DD, v_kk, v_CH, v_SS, v_nn, v_RR, v_aa, v_E, v_ih, v_oh, v_ou`（后面是 `_________eye_________`、
brow/mouth/nose/ex 等），**交集 = 0** → VRChat 写不进任何一个形态键 = 不张嘴。

### 6.3 判据（一条命令定位）
```csharp
var d = avatar.GetComponent<VRCAvatarDescriptor>();
var m = d.VisemeSkinnedMesh.sharedMesh;
var names = new HashSet<string>();
for (int i = 0; i < m.blendShapeCount; i++) names.Add(m.GetBlendShapeName(i));
int hit = d.VisemeBlendShapes.Count(n => names.Contains(n));
// hit == 0 → 就是本坑；hit == 15 → 描述符配置正确
// 同时检查 names.Contains(d.MouthOpenBlendShapeName)
```

### 6.4 修法（15 名字顺序 = VRChat viseme 顺序）
```csharp
d.lipSync = VRC.SDK3.Avatars.Components.VRCAvatarDescriptor.LipSyncStyle.VisemeBlendShape;
d.VisemeSkinnedMesh = wolfBodySMR;               // 狼头 Body
d.VisemeBlendShapes = new string[] {
  "v_sil","v_PP","v_TH","v_FF","v_DD","v_kk","v_CH","v_SS",
  "v_nn","v_RR","v_aa","v_E","v_ih","v_oh","v_ou" };
d.MouthOpenBlendShapeName = "v_aa";               // 张嘴用最大开口的一个
EditorUtility.SetDirty(d);
```

### 6.5 烘焙后复核（AAO 不会丢名字，实测）
- 烘焙克隆里 `VisemeSkinnedMesh.sharedMesh.name = AAOGeneratedMeshBody`，`blendShapeCount = 208`，15 个名字 **15/15 命中**（`v_aa` idx=10 等）。
- 所以「AAO 的 `optimizeBlendShape` 会把形态键改名」这条**只在不该开的头像上才成立**；本头像已关该项，名字完整保留。
- 复核脚本要点：在 baked clone 上重跑 6.3 的 `hit` 计数，必须 15。

### 6.6 另一个坑：主 FX 的 `LipSync Control` 层对双头头像天然失效
主 FX（猫的控制器）里有 `LipSync Control` 4 态，clip 的 path 是根级 `Body`；双头头像的猫身网格叫 `Body_Base`、根下没有 `Body` → 这层绑定全部悬空（无害，但别把它当成口型来源）。
本工作区口型实际由 `LilLeo_FX_FaceOnly` 的 `WolfFace Viseme`（15 态，直接写 `blendShape.v_*`）+ `WolfFace Voice`（Additive 连续开口）驱动，与描述符并存、互不冲突。


## §7 原生狼 FX（`LilLeo_FX_Wolf`）上做「双手握拳 = 唐笑」的实测复核（第 126 轮，2026-10-29）

### 7.1 与 §4 的差别

第 119 轮的做法是在「换头 FX / `LilLeo_FX_FaceOnly`」上做；第 126 轮是**直接改主 FX `LilLeo_FX_Wolf`**（`沐梓溪狼菜单/LilLeo_FX_Wolf.controller`），
不经过换头层。槽位复核：左右手 8 个单手势里 **只有 Fist（握拳）是空的**（Fist 态播 `LilLeo_Empty`，无曲线），
其余 2–7 都有作者表情 → 用「双手同时 Fist」触发唐笑，不会顶掉任何现有手势。

### 7.2 做法（数值与判据）

- 源 clip：`<VRCHAT_ROOT>\resources\小狼唐笑.anim`（188 条曲线，只有 35 条有值；路径全部是根级 `Body`，即狼头网格）。
- 派生 clip：只保留 35 条有值曲线，`loopTime=false`，路径 `Body`；存 `Assets/08_动画·菜单·图标/沐梓溪狼菜单/clips/WolfFace_TangXiao.anim`。
- 新层：`WolfFace TangSmile`，**Override、weight=1**，`Off` 态=`LilLeo_Empty`、`On` 态=上述 clip；
  两条 AnyState→On 条件 `GestureLeft Equals 1`、`GestureRight Equals 1`（同一转移内条件是 **AND**，所以一条转移写两个条件即可）；
  两条 AnyState→Off 条件 `GestureLeft NotEqual 1`、`GestureRight NotEqual 1`。
- 烘焙后（NDMF `ProcessAvatarUI`）：FX `LilLeo_FX_Wolf` 73 层，`WolfFace TangSmile` 在 **index 14**、w=1、On clip **25 条曲线**。

### 7.3 关键坑：AAO 烘焙会把形态键改名，GM 断言必须查 `AAO_Merged_*`

烘焙克隆的狼头 `Body` 只剩 **90** 个形态键（源 208），名字被 AAO 合并成：

```
AAO_Merged_brow_smile_35 / AAO_Merged_ex_blush_72 / AAO_Merged_mouth_he_52
AAO_Merged_mouth_straight_... / AAO_Merged_teeth_off_68
```

因此 GM Play 里用 `GetBlendShapeIndex("brow_smile")` 会返回 **-1**、读数 -999（本工作区第一次跑就是这样误判 FAIL）。
正确写法：先精确查，找不到再遍历 `GetBlendShapeName(i).Contains("AAO_Merged_" + 名字)`。

### 7.4 实测数值（GM Play，烘焙后克隆）

| 状态 | brow_smile | ex_blush | mouth_he | mouth_straight | teeth_off |
|---|---|---|---|---|---|
| 双手 Fist（GestureLeft=1 且 GestureRight=1） | 100 | 100 | 60 | 100 | 100 |
| 松开（Gesture 0） | 0 | 0 | — | — | — |
| 回归：单手 Peace（Gesture=2） | 4 个形态键变化（非 0） | | | | |

→ 唐笑只在双手握拳出现、松开归零、其它手势表情照旧（未顶替）。

### 7.5 边界

编辑器只能验证「Animator 参数 → 形态键」这一段；**真机手势触发、多人同步 NOT_RUN**。

## §8 第 132 轮：表情切换「丝滑化」——复制 clip 加 0.16 s ramp + AnyState 转移 0.15 s

### 8.1 目标与约束

- 症状：手势表情在 VRChat 里「瞬切」，观感生硬。
- 结构：`LilLeo_FX_Wolf` 的 `Left Hand` / `Right Hand` 各 8 态，由 `GestureLeft/GestureRight` 经 AnyState 转移驱动；唐笑是独立 `WolfFace TangSmile` 层（双手 Fist）。
- 原实现：clip 瞬切（length≈0、1–2 键）+ 转移 `duration=0` → 一帧跳过去。
- 非破坏原则：**原 clip 一律不动**，新增 `_S` 副本，只替换 FX 里的 motion 引用；回滚成本=改回引用。

### 8.2 做法

| 项目 | 原实现 | 第 132 轮 |
|---|---|---|
| clip | 瞬切，length≈0 | 复制 14 个手势 clip + 唐笑，输出 `Assets/08_动画·菜单·图标/动画/smooth/*_S.anim` |
| clip 长度 | 0–0.02 s | **0.160 s**，`wrapMode=Default`（不 loop） |
| 关键帧 | 1–2 | **4 键：0 / 0.02 / 0.10 / 0.16 s**，值 0→0→mid→target（例 `blendShape.ex_blush 0→0→50→100`）；个别曲线 3 键（0/0.04/0.14） |
| AnyState 转移 | duration=0 | **0.150 s**、`hasFixedDuration=true`、`interruptionSource=Destination`、`canTransitionToSelf=false` |
| 唐笑 On/Off | 0 | On 0.120 s / Off 0.140 s |

- `interruptionSource=Destination`：换手势/松手时从当前混合值继续，不跳回 0。
- `canTransitionToSelf=false`：避免同一手势每帧重入，永远停在 ramp 起始。
- 空态 = `LilLeo_Empty`（0 曲线）；回默认靠 Animator 混合，不必在 clip 里补 0 键。
- 4 键而非 2 键：中间两键把线性 ramp 压成「快起慢收」，1 帧跳变摊到 0.16 s，同时保留精确稳定终值。

### 8.3 验收

- 静态 dump FX（`Round132ExprCheck`）：`FX layers=11`；Left/Right Hand 各 8 条 AnyState 转移 `dur=0.150`；14 个 `_S` clip `len=0.160 / loop=false / 4 键`；
  TangSmile On 引用 `WolfFace_TangXiao_S`；`BlinkAuto anyState=0`。
- GM：`RAMP expr_in_open 81.4→208.9`（同一采样窗口内渐变）、`TANGXIAO steady brow_smile=100/ex_blush=100/mouth_he=60`、`tangxiao_out 1160→0`。
- 边界：GM 不模拟 Gesture 层动画；**真机观感仍需进游戏实测**。

## §9 第 133 轮：手势表情「丝滑度」只来自 crossfade（GestureWeight 与 clip ramp 的边界）

**背景**：第 132 轮给 14 个手势 clip 加了 0.16 s / 4 键 ramp，并把 AnyState 转移从 0 改到 0.15 s（见 §8）。本轮实测确认「观感主要由哪个机制决定」，并给出调参方案。

### 9.1 关键事实：`GestureLeft/RightWeight` 只有 Fist 连续

- VRChat 内置参数 `GestureLeftWeight` / `GestureRightWeight` **只有 Fist（握拳）随扳机连续 0→1**；其余 7 种手势是 0/1 阶跃（VRC School、CGE 文档口径一致）。
- 因此状态内 clip 的 0.16 s ramp 只在「该状态被完整播放」时有意义：非 Fist 手势下状态时间恒在 clip 末端，形状由稳定终值决定，ramp 基本不参与观感。
- 本工程两只手的 Fist 槽位空态 clip 都是 0 曲线的 `LilLeo_Empty`，所以「用 weight 模拟渐变」的收益 = 0。
- **结论：状态之间切换的丝滑度只取决于 crossfade（AnyState 转移）时长。**

### 9.2 已落地方案 A：拉长 crossfade（本轮实测）

- `LilLeo_FX_Wolf.controller`：
  - L2 `Left Hand` / L3 `Right Hand`：各 8 条 AnyState 转移 `duration` 0.15 → **0.24**、`hasFixedDuration=true`、`interruptionSource` `Destination` → **`SourceThenDestination`**。
  - L10 `WolfFace TangSmile`：`ANY→On` 0.12 → **0.18**、`ANY→Off` 0.14 → **0.18**，`interruptionSource` 同步改为 `SourceThenDestination`。
- 烘焙复核（`R133SimHarness`）：`EXPRBAKE|layers=Left Hand;Right Hand;WolfFace TangSmile;|anyTransitions=19|minDur=0.18`（手 0.24 / 唐笑 0.18 均命中）。
- GM 数值：`RAMP|expr_in_open … 81.4→208.9 rising`、`TANGXIAO|released brow_smile=0.0 ex_blush=0.0`。
- 边界：GM 只验证烘焙后的层/时长与数值，**真机手感 NOT_RUN**。

### 9.3 可选进阶（未落地，按收益排序）

- **方案 B：用 `GestureWeight` 做 Off/Half/Full 三档**。滞回阈值示例 0.30 / 0.35 / 0.70 / 0.75；直接做 1D BlendTree 会在松拳 weight 跳回 1.0 时 pop，需要 CGE 式「冻结修正」：weight 回跳时冻结当前混合值，再按阈值解锁。
- **方案 C：曲线穷举化**（先做这个更划算）。Left 手势 union 193 / intersect 17 条曲线、Right union 195 / intersect 170；`饿饿_S` 只有 17 条曲线、`哭哭_S` 173 条 → 状态间切换会留脏键。
  另可清 31 个不可达状态、39 个孤儿 BlendTree、空层 `AllParts`；WD 归一化现状为「可达 48 个 On + 2 个 Off」混用，`LilLeo_Empty` 长 1.0 s，建议换成 2 帧 buffer。
- 顺序建议：**C → A → B**（A 已完成）。

### 9.4 参考（External）

- VRChat 官方 Animator Parameters（GestureWeight 定义）：https://creators.vrchat.com/avatars/animator-parameters/
- VRC School 内置参数语义（Fist 连续、其它 flat）：http://vrc.school/docs/Avatars/VRC-Parameters/
- CGE Corrections（transition correction / smoothing correction / exhaustive curves）：https://docs.hai-vr.dev/docs/products/combo-gesture-expressions/additional-pages/corrections
- CGE Integrator（把 GestureWeight 换成平滑参数）：https://docs.hai-vr.dev/docs/products/combo-gesture-expressions/additional-pages/integrator
- Unity `AnimatorStateTransition.interruptionSource`：https://docs.unity3d.com/2022.3/Documentation/ScriptReference/AnimatorStateTransition-interruptionSource.html
- VRChat Canny（GestureWeight 驱动状态历史 bug，已修，改动后仍须实测）：https://vrchat.canny.io/avatar-30/p/adding-a-locomotion-override-breaks-the-gesture-weight-parameters
