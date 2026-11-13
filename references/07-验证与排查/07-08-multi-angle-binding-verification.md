# 装骨 / 配件 / 衣装的「多角度独立验证」标准流程

> 本工作区实测：2026-10-27（第 104 轮，任务「迁移小狼尾巴 + 手套 + NewFit 胡萝卜衣服」）
> 适用范围：任何「把东西挂到骨头上」的改动（尾巴、手套、饰品、衣装、ABT 绑点、SPS socket、道具、发色材质联动）。
> **本册已替换旧版「验证」步骤**：以后所有改模收尾一律按本册 5 步执行，旧的「看 Console / 看编辑态一张图」不再算验证。

---

## 0. 为什么必须多步验证

| 只看什么 | 会漏掉什么 |
| --- | --- |
| Console 无 error | 曲线被丢弃（`NO VALID ANIMATIONS`）、参数未注册、菜单没生成都不报错 |
| 编辑态一张正视图 | 背面/侧面穿模、尾巴朝错方向、手套没贴到手背、尺寸比例错误 |
| Play 里截图 | 编辑器 Play 的烘焙克隆 **Animator = null**，看到的是静态默认态；自建相机 + 灯在 Play 里渲染经常是全黑剪影 |
| 只看烘焙产物 | 烘焙产物正确 ≠ 场景对象已保存；下次重开工程可能引用丢失（`subMenu: {fileID: 0}` 事故） |

结论：**数值 + 多角度出图 + Windows-MCP 实看 + 烘焙产物 + 保存后复核**，五件套缺一不可。

---

## 1. 步骤一：数值验证（先算，再看）

### 1.1 骨骼位置

```csharp
Transform arm = wolf.transform.Find("armature");
Transform hips = arm.Find("Hips");
Transform head = arm.Find("Hips/Spine/Chest/Neck/Head");
Debug.Log("Hips=" + hips.position.ToString("F4") + " Head=" + head.position.ToString("F4"));
// 参考基准（本工作区，F 系头像，身高 0.78 m）：
// Hips=(0.0000, 0.4183, 0.0108)  Head=(0.0000, 0.6414, 0.0045)
```

判据：迁移来的骨链，每一节的 world 位置应与**参照头像**逐位相差 < 1 mm；超出即 TRS 抄错 / 父级错。

### 1.2 蒙皮网格真实包围盒（不能用 `Renderer.bounds`）

`Renderer.bounds` 在 AAO 下会被撑成 2×2×2 大盒，**必须** `BakeMesh` + `TransformPoint`：

```csharp
Mesh m = new Mesh();  smr.BakeMesh(m);
Vector3 min = smr.transform.TransformPoint(m.bounds.min);
Vector3 max = smr.transform.TransformPoint(m.bounds.max);
```

判据表（0.78 m 高头像，实测值可直接对照）：

| 部件 | 期望 y 区间 | 期望 x 区间 | 实测（本工作区） |
| --- | --- | --- | --- |
| 尾巴 `Cat_Tail` | 0.19 ~ 0.60（向后向上卷，z ≤ -0.05） | ±0.06 | y 0.19~0.60，z -0.05~-0.31 |
| 手套 `Gloves` | 0.554 ~ 0.602（= 双手高度） | -0.322 ~ 0.322（跨双手） | ✅ |
| 睡帽 `狼头版_睡衣帽子` | 0.605 ~ 0.865 | ±0.16 | ✅ |
| 眼罩（戴好） | 覆盖眼骨 0.686 上下 | ±0.13 | ✅ |
| 胡萝卜衣 Overalls | 0.283 ~ 0.631 | ±0.112 | ✅ |
| 胡萝卜衣 Shoes | -0.01 ~ 0.10 | ±0.06 | ✅ |

> 判定「比例是否正确」的硬标准：**部件 y 区间必须落在身体对应区间内**（头顶 0.767、脚底 0.0、手 0.582、髋 0.418）。任何部件越出头顶或穿过地面 = 缩放/绑定错。

### 1.3 参数与菜单（写盘后）

```csharp
// 菜单递归
int menus = 0, controls = 0, missing = 0;  // missing = SubMenu 但 subMenu==null
// 参数
foreach (var p in ep.parameters) Debug.Log(p.name + " " + p.valueType + " def=" + p.defaultValue + " saved=" + p.saved);
```

判据：`missingSub=0`；新参数出现在 EP 里且 `def` = 设计默认值、`saved=True`；`一键穿衣` 的 `OutfitSelect` 默认 = 默认套装序号。

---

## 2. 步骤二：编辑态离线多角度出图（不是 Play）

```csharp
// 相机：FOV 28，near 0.02，far 30，深色纯色背景
cam.clearFlags = CameraClearFlags.SolidColor;
cam.backgroundColor = new Color(0.07f, 0.08f, 0.10f, 1f);
// 灯光：主光 intensity 1.15 Euler(28,140,0) + 补光 0.55 Euler(20,330,0)
// 出图：700~900 px，ARGB32，antiAliasing=4
```

**朝向铁律**：本工程角色正面朝 **+Z**（`Head.forward.z ≈ +0.996`）。所以

- 正面：相机放 `target + forward * dist`（+Z 侧）；
- 侧面：`Quaternion.Euler(0,90,0) * Vector3.forward`；
- 背面：yaw 180。
写成 `target - forward * dist` 拍到的是后脑勺（本工作区连错两次才发现）。

必拍 6 张：**正 / 侧 / 背 / 手部特写 / 头部特写 / 被测件特写**。渲时隐藏：其它头像根、`ABT`、无关衣装（否则面板/预览副本入镜）。

> 用完立刻 `EditorSceneManager.OpenScene(场景, Single)` 丢弃内存改动，避免临时相机/隐藏状态被误存。

---

## 3. 步骤三：目视判读（含「图太大看不见」的解法）

1. 出图后先 **PIL 缩到长边 ≤ 420 px 再读图**；直接读 800×1100 原图常被 harness 截断/省略，导致「看起来像拍糊」。
2. 判读清单（逐条打勾）：
   - [ ] 部件是否贴在**解剖正确位置**（手套在手背、尾巴在骶骨、帽子在头顶、眼罩在眼骨高度）
   - [ ] 有无**浮空**（部件与身体之间出现背景色缝隙）
   - [ ] 有无**穿模**（身体/其它衣服穿出部件表面）
   - [ ] 比例是否与身体匹配（头饰宽度 ≈ 头宽 ×1.5~2.0；衣长覆盖躯干）
   - [ ] 颜色/材质是否正确（发色联动件应与头发同色）
   - [ ] 侧面/背面没有「半边消失」「反了」

补充：用 PIL 做**背景色差 bbox 统计**辅助判断占画比例（四角色差 > 40 记为内容），但注意线性色彩空间下背景不等于设定色，取角落像素做基准。

---

## 4. 步骤四：Windows-MCP 实机多角度查看（用户明确要求的「实际查看」）

### 4.1 准备

```powershell
# 1) 关掉挡屏窗口（NDMF Console / SDK 面板 / NewFit 窗口 / 性能窗口）
# 2) 置前 Unity 主窗口
& '<TOOL_ROOT>\.skills\windows-mcp\scripts\uia.ps1' -Action focus -Title 'LuzzyMeow V2'
& '<TOOL_ROOT>\.skills\windows-mcp\scripts\uia.ps1' -Action list-windows   # 确认 FOREGROUND + rect
```

### 4.2 每换一个角度

```csharp
// execute_code：设 Scene 视图
SceneView sv = SceneView.lastActiveSceneView;
sv.pivot = new Vector3(0, 0.40f, 0.02f);
sv.size = 0.55f;                       // 角色占视口 ~80%（本机 LookAt 的 size 会被夹，直接赋值更稳）
sv.rotation = Quaternion.Euler(8, 180, 0);   // 正面；侧面 90；背面 0
sv.drawGizmos = false;                 // 关绿色骨骼 gizmo（否则截图全是绿线）
sv.Focus();
InternalEditorUtility.RepaintAllViews();
```

```powershell
# 等 ~0.9 s 让编辑器重绘，再抓屏
Add-Type -AssemblyName System.Drawing
$bmp = New-Object System.Drawing.Bitmap 1920,1080
$g = [System.Drawing.Graphics]::FromImage($bmp)
$g.CopyFromScreen(0,0,0,0,(New-Object System.Drawing.Size(1920,1080)))
$bmp.Save('<VRCHAT_ROOT>\tools\tmp\wmcp_view.png',[System.Drawing.Imaging.ImageFormat]::Png)
$g.Dispose(); $bmp.Dispose()
```

```python
# 裁 Scene 视图区域 + 缩到 600px 再读图
im = Image.open(r"<VRCHAT_ROOT>\tools\tmp\wmcp_view.png").crop((215,100,1230,700))
im.thumbnail((600,380)); im.save(r"<VRCHAT_ROOT>\tools\tmp\scene_crop.png")
```

也可以在 Unity 置前后直接 `mcp__windows__Screenshot` 抓整屏（本工作区可正常返回图，但整屏 1920×1080 缩小后细节有限，**裁剪 + 缩放到 600 px 更可靠**）。

### 4.3 判读要点（实机截图）

- 实机截图能看到 Scene 视图的**选中描边/gizmo/网格**，这些在离线渲染里没有 → 用来确认「Hierarchy 选中是否正确、Gizmo 是否关闭」。
- 同一角度**离线渲染**和**实机截图**要一致；不一致说明场景里有没保存/没恢复的临时状态。

---

## 5. 步骤五：烘焙产物验证（唯一权威）

Play 一次（等 NDMF 完成，Console 0 error），然后读 `Packages/com.vrcfury.temp/Builds/____/`：

| 文件 | 检查点 |
| --- | --- |
| `VRCFury Menu.asset` | 递归 `menus/controls/missingSub`；新菜单项的参数、value、类型（Button/Toggle/SubMenu） |
| `VRCFury Params.asset` | 新参数 `type/defaultValue/saved`；总数 ≤ 256 |
| `VRCFury FX.controller` | 新层 `defaultWeight`、状态数、`AnyState` 条件、clip 曲线数、路径是否解析（`NO VALID ANIMATIONS` = 路径没解析） |

判据：
- 曲线被剥离 = `MergeAnimator.pathMode` 用了 `Relative`（相对宿主 GO）→ 改 **`Absolute`**；
- 层 `weight=0` 但有 StateBehaviour = VRCFury `CleanupEmptyLayersService` 正常行为，VRChat 里 StateBehaviour 仍执行（见 `06-构建上传与备份/06-04`）；
- 动态换装（driver）在**编辑器 Play 里不会跑**（`VRCAvatarParameterDriver.initialized=False`）→ 只能验烘焙产物 + 进游戏实测（NOT_RUN）。

退出 Play 后必做：还原 `Assets/08_动画·菜单·图标/NyaPPu__Generated/LightController/*.anim`（MD5 对齐原件，见下表）、删 `.sln/*.csproj`、清 `tools\tmp`。

| 文件 | 大小 | MD5 |
| --- | --- | --- |
| BacklightBorder.anim | 536683 | 880E26874A77DE8D9EF79BC80E9D2564 |
| BacklightToggle_Off.anim | 314216 | F9A125593E56906B572226DF179D1F37 |
| BacklightToggle_On.anim | 315567 | F1BE1377E235D9952FB4D60BC8E7F396 |
| LightLimit.anim | 384159 | C121A44F008AF2881C6F00342B14BA7E |
| Monochrome.anim | 107331 | CD1D15CC0BC54BB4EB250893D7DB1D7C |
> **2026-11-10 补充**：本表是唯一权威基线，但**原件不在 `F:` 盘**了；可还原的备份在
> `<PROJECT_ROOT>\_asset_reorg\backup\hairdel-20261026-104303\LightController\`。
> 「退出 Play 后必做」这条**必须扩展到「退出 Play 与上传构建之后」**：本轮上传前构建同样触发了重写（`BacklightBorder.anim` 被删）。

---

## 6. 交付前最后一屏

1. 恢复默认形态（默认套装 on、默认关闭件 off）；
2. `Selection.activeGameObject = 被测角色` + `PingObject`；
3. Scene 视图 `pivot=(0,0.42,0.02)`、`size≈0.44`、`rotation=Euler(6,196,0)`、`drawGizmos=false`；
4. `SaveScene`（`dirty=False`）；
5. 截终态图存 `_asset_reorg/evidence/<round>/final_preview_*.png`；
6. 边界写清 NOT_RUN（游戏内实际观感 / 多人同步 / 参数 saved 持久化）。

---

## 7. 踩过的坑（速查）

| 现象 | 根因 | 处理 |
| --- | --- | --- |
| 离线渲染「全黑剪影」 | 在 Play 运行时自建相机/灯光 | 回编辑态渲染 |
| 出图「只有半个身子/贴脸」 | 相机放在 `target - forward*dist`（背面）或 FOV/距离未随部件尺寸调整 | 按 +Z 朝向重算；特写按部件尺寸给 0.3~1.0 m |
| 读图看不见细节 | 原图太大被 harness 截断 | PIL 缩到 ≤420 px 再读 |
| 烘焙 clip 全空 | `pathMode=Relative` | 改 `Absolute` |
| 参数改了不生效 | Int 参数用了 `SetFloat` | Int 用 `SetInteger` |
| Play 里 `SetLayerWeight` 无效 | VRCFury `FixAnimatorPreviewBreakingInPlayModeHook` 每帧回写 | 别在 Play 里试权重 |
| `GetField("rootTransform")` 取不到 VRCPhysBone 字段 | 字段在基类 | 沿 `BaseType` 逐级 `GetField(..., Public\|NonPublic\|Instance)` |


## §9 每个功能 6 方位渲染矩阵（第 120 轮标准化做法）

### 9.1 工具
`Assets/Editor/WolfShot.cs` → `LuzzyTools.WolfShot.Render6(string outDir, string tag, string forcedActive, string angles, float dist, float fov, int size, float targetY)`
- `forcedActive`：分号分隔的相对路径（会自动激活整条父链），用于「只显示这一件」或「把这件打开看」；
- `angles`：`"0,6;60,6;120,6;180,6;240,6;300,6"`（yaw,pitch）；
- 会临时隐藏场景里其它带 `VRCAvatarDescriptor` 的根，渲染完自动还原。
- **注意**：`execute_code` 每次调用会触发域重载，`EditorApplication.delayCall` 回调可能被打断；渲染是同步的，不受影响。

### 9.2 状态矩阵（本工作区实际用例）
| tag | 状态设置 | 参数 |
|---|---|---|
| `op_default` | 默认（睡衣套装） | dist 2.30 / FOV 26 / y 0.45 |
| `op_carrot2` | `睡衣1.SetActive(false)` + `胡萝卜衣服` 整链激活 | 同上 |
| `op_hat` | 睡衣 + `PJ_Hat/PJ_HatRibbon/PJ_Outer/PJ_PocketGhost` 全开 | 同上 |
| `op_acc2` | `眼镜` + `Head/名牌_mesh` + `Gloves` | 特写 dist 0.95 / FOV 30 / y 0.62 |
| `op_tailoff` | `Cat_Tail.SetActive(false)` | dist 2.30 / FOV 26 |
| `op_tangxiao` | 直接把 `小狼唐笑.anim` 里 `blendShape.*` 曲线值（只取 =100 的）写到对应 SMR，渲染后恢复权重 | 特写 dist 0.95 / FOV 30 |

### 9.3 判读
- 6 张拼 2×3 拼图（PIL 210×th 缩放，`sheet.thumbnail((520,520))`），逐张看：头是否在、饰品是否贴脸/贴头、衣服是否穿模、尾巴/耳朵是否变形；
- 特写单独裁（如 `crop((200,120,520,420))`）看眼镜框位置、名牌文字方向。

### 9.4 已发现的坑
1. **相机取景会骗人**：名牌在 y≈0.86，而特写相机 targetY=0.60、dist 0.95、FOV 30 的纵向半高只有约 0.25 → 名牌刚好被切在画面外，误判「名牌没做」。要么放大 dist/缩小 FOV，要么单独渲染该件。
2. **`Renderer.bounds` 在蒙皮件上是坏的**（恒为 1×1×1），不能用来取景；用 `BakeMesh` + `TransformPoint` 求真实包围盒。
3. 唐笑这类「clip 驱动形态键」的状态，编辑态不要用 `clip.SampleAnimation`（会把骨骼写坏，见 `05-08 §16`）；改为**读取曲线值直接 `SetBlendShapeWeight`**，渲染后恢复。
