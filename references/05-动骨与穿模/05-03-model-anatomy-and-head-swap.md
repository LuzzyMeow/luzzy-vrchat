# 换头与模型解剖（本工作区实测：Kipfel 1.2.0 / Shipilka v1.02）

> 状态：**只做了只读测量与可行性分析，尚未在工程里动手**（用户 2026-10-17 提问：「想要 Shipilka 的头 + Kipfel 的身体，要不要用 Blender」）。
> 测量日期 2026-10-17；测量过程**没有修改任何素材，也没有改工程**；量法见 §2。

## §0 何时读这一册

- 想做**换头 / 换身体 / 替换某个部位网格**；
- 要判断「一个模型的头是不是独立网格」、能不能在 Unity 里直接删掉或替换某个部位；
- 要量两个模型的比例、脖子截面、缩放系数；
- 要在 Blender **无头模式**下快速列出一个模型的结构（对象 / 顶点数 / 形状键 / 材质 / 骨骼 / 顶点组）。

## §1 结论速览

1. **两个模型的解剖结构是同构的**：头是**独立网格** `Body`，身体是 `Body_Base` —— 所以不存在「用刀把头从身体上切下来」这种大手术。
2. **换头仍然要用 Blender，但只用在脖子接缝那一小段**：几何挂接、缩放、骨骼合并、材质、菜单与验收都能在 Unity 里完成（MA `Merge Armature` + 删掉原来的头网格）。
3. 直接搬过去需要**整体缩放 ×0.70**（§4）；缩放后身高与头身比基本吻合（差约 6%），脖子截面**能盖住**（不会露洞），但会留一圈可见台阶 —— 用项圈 / 衣领遮掉就能省下 Blender 精修。
4. 真正的重活不是几何，而是**表情 / 菜单 / 眼球 / 面捕的重指、肤色衔接与性能**（§6）。

## §2 怎么量（只读，可复用）

**Blender 无头模式**（本机 `<VRCHAT_ROOT>\tools\Blender\blender-4.5.14-windows-x64\blender.exe`）：

```powershell
# --factory-startup 很关键：否则会加载 CATS / mmd_tools，输出里全是插件注册警告，看不到自己的结果
& '<VRCHAT_ROOT>\tools\Blender\blender-4.5.14-windows-x64\blender.exe' -b --factory-startup '<model>.blend' --python measure.py > out.txt 2>&1
Get-Content out.txt | Select-String '^####'      # 用固定前缀过滤自己的输出
```

`measure.py` 的骨架（对象 / 形状键 / 材质 / 骨骼位置 / 顶点组 / 截面）：

```python
import bpy
for o in bpy.data.objects:
    if o.type == 'MESH':
        me = o.data; sk = len(me.shape_keys.key_blocks) if me.shape_keys else 0
        print('#### OBJ', o.name, len(me.vertices), len(me.polygons), 'shapes=%d' % sk, [m.name for m in me.materials if m])
arm = [o for o in bpy.data.objects if o.type == 'ARMATURE'][0]
for b in arm.data.bones:
    if b.name in ('Hips', 'Spine', 'Chest', 'Neck', 'Head'):
        p = arm.matrix_world @ b.head_local; print('#### BONE', b.name, tuple(round(v, 3) for v in p))
o = bpy.data.objects['Body']                      # 头
g = {}
for v in o.data.vertices:
    for w in v.groups: g[o.vertex_groups[w.group].name] = g.get(o.vertex_groups[w.group].name, 0) + w.weight
print('#### VGROUPS', sorted(g.items(), key=lambda kv: -kv[1])[:10])   # 只有这些组真正驱动头
pts = [o.matrix_world @ v.co for v in o.data.vertices]
z0 = min(p.z for p in pts); ring = [p for p in pts if abs(p.z - z0) < 0.012]
print('#### NECK ring', len(ring), 'x-width', round(max(p.x for p in ring) - min(p.x for p in ring), 3))
```

**Unity 侧**（工程已打开、stdio 桥在跑时用 `mcp__unity__execute_code`）：`GameObject.Find(...)` → `GetComponentsInChildren<SkinnedMeshRenderer>(true)` 读 
`sharedMesh.vertexCount / blendShapeCount`、`sharedMaterials`、`bones.Length`；`PrefabUtility.GetPrefabInstanceStatus` 看 prefab 状态。

## §3 实测解剖数据

### Kipfel 1.2.0（源 `resources/01_模型本体/Kipfel小猫合集-初音的VRC小店/Kipfel模型本体1件/Kipfel_1.2.0/Blend/Kipfel_1.2.0.blend`，59,120,173 B）

| 对象 | 顶点 | 形状键 | 材质 |
|---|---|---|---|
| `Body`（**头**） | 6,944 | **373** | Body_skin / Body_eye / Body_alpha / Hair |
| `Body_Base`（身体） | 4,358 | 52 | Body_skin |
| `Hair_Back` / `Hair_Base` / `Hair_Bun` / `Hair_BunRibbon` / `Hair_Front` / `Hair_Side` | 4,255 / 312 / 831 / 266 / 618 / 1,413 | 0–13 | Hair / Outfit02 / Silver |
| `Cat_Ear` / `Cat_Tail` | 1,008 / 1,910 | 0 / 5 | Hair / Outfit02 |
| 服装与道具 | `Outfit_*` / `UnderWear_*` / `Item_*`（Bag 1,935、Boots 1,960、Glasses 478、FishBone 605、FishToy 176、DressingPad 164 …） | 0–3 | Outfit01 / Outfit02 / Silver / Outfit_alpha |

- 骨骼 **181** 根，命名 `Hips → Spine → Chest → Neck → Head →`（头网格 z 0.622–0.767，见下）。
- `Body`（头）**只绑 `Head` 骨**：整个头只有 5 个顶点组 —— `Head`（权重和 6322.4）、`Eye.L`（320.7）、`Eye.R`（300.9）、`Neck`（2.5）。**眼球是头网格的一部分**，由眼球骨驱动。
- Unity 里对应 `Kipfel/Body`：7,540 顶点 / 10,257 三角面 / 372 形状键（前 8 个是 `vrc.v.sil`、`vrc.v.pp`、`vrc.v.ff`、`vrc.v.th`、`vrc.v.dd`、`vrc.v.kk`、`vrc.v.ch`、
  `vrc.v.ss`），材质 `Body_skin, Hair, Body_eye, Body_alpha`，绑 181 根骨。
- z 范围：`Body` **0.622 – 0.767**（头高 **0.145**）；`Body_Base` 0.009 – 0.645。
- 脖子截面：头网格**底部一圈**（距最低点 ≤12 mm）198 顶点，x 宽 **0.093**；身体网格**顶部一圈** 19 顶点，x 宽 **0.031**。

### Shipilka v1.02（源 `resources/01_模型本体/作者-Shipilka/(密码：TW527E) Shipilka シピルカ - v1.02/v1.02/Shipilka.blend`，22,337,553 B）

| 对象 | 顶点 | 形状键 | 材质 |
|---|---|---|---|
| `Body`（**头**） | 8,324 | 89 | Shipilka_Body / Shipilka_Hair / Facialstencil / Shipilka_eyelash |
| `Body_Base`（身体） | 12,384 | 15 | Shipilka_Body |
| `Hair` | 11,868 | 3（`Shipilka_Hat_ON` / `Shadow_OFF`） | Shipilka_Hair / Facialstencil |
| 服装 / 鞋袜 | Outfit_Coat 10,713 / Outfit_Tops 7,781 / Outfit_Shorts 2,372 / Footwear_Shoe 7,924 / Footwear_Socks 764 / Under_Spats 750 | 0–2 | ACostume / BCostume / Spats |
| 头饰与配件 | HeadItem_Halo 3,116 / HeadItem_Hat 3,560 / Item_Wing 3,634 / Item_Cloudtail 1,330 / Item_SmallWing 216 / Item_Band aid 324 | 0 | BCostume / ACostume |

- 骨骼 **152** 根，MMD 体系英文命名：`Hips`(z 0.638) → `Spine` → `Chest`(0.779–0.936) → **`Neck`**(0.948–0.976，长 0.028) → **`Head`**(0.976–1.184，**长 0.207**)。
- `Body`（头）**只绑 `Head` 骨**：虽然有 216 个顶点组，实际只有 `Head`（8324.0）与 `outline`（1524.0）有权重。
- 形状键是 **`vrc.v_aa` 体系**（与 Kipfel 的 `vrc.v.sil` 写法不同），另有 `---option---`、`hairline_OFF`、`---Lipsync---` 这类分组标记。
- z 范围：`Body` **0.957 – 1.164**（头高 **0.207**）；`Body_Base` 0.024 – 0.978。
- 脖子截面：头网格底部一圈 491 顶点，x 宽 **0.084**；身体顶部一圈 68 顶点，x 宽 **0.043**。

## §4 比例与接缝计算

- **头身比**（头高 ÷ 脖高）：Kipfel 0.145 / 0.645 = **22.5%**；Shipilka 0.207 / 0.978 = **21.2%**（差 6%，属同一个「动画风大头」档位）。
- **整体身高**：Kipfel ≈0.83、Shipilka ≈1.19 → 把 Shipilka 整体缩 **×0.70** 后身高 0.83、头高 0.145，与 Kipfel 对齐。
- **接缝**：缩放后头网格底部宽 = 0.084 × 0.70 = **0.059** > Kipfel 身体顶部截面 **0.031** → **头能罩住脖子，不会露洞**；但比 Kipfel 原头的 0.093 细，衔接处会有一圈可见台阶。
- 于是两条路线：**(a)** 用项圈 / 衣领遮 → **纯 Unity 可完成**；**(b)** 要脖子过渡自然 → 在 Blender 里把 Shipilka 头底那一圈顶点向外扩（或加一段过渡环面），这是**唯一必须用 Blender 的步骤**。

## §5 分工

| 步骤 | 在哪做 | 要点 |
|---|---|---|
| 把 Shipilka 的头相关对象挂到 Kipfel 的 `Head` 骨下 | Unity | 头 `Body`、`Hair`、睫毛/眼、头饰都是各自独立对象，可整棵子树搬 |
| 整体缩放 ×0.70 | Unity | 对搬过来的根做 uniform scale；**先量再缩放**，别拍脑袋 |
| 骨骼合并 | Unity | MA `Merge Armature`（首选，铁律 4）；**注意同名骨**：两边都有 `Head`/`Neck`/`Hips`/`Spine`/`Chest`，必须定好归属 |
| 删掉 Kipfel 原来的头网格 | Unity | `Kipfel/Body` 是**独立 SkinnedMeshRenderer**，删对象即可（源 prefab 资产不动） |
| 脖子接缝 | Blender（或遮住） | 见 §4 |
| 材质 / 肤色 | Unity | 两个模型不是同一画师，脖子肤色衔接要靠材质与贴图调 |
| 菜单 / 参数 / 表情重指 | Unity | 见 §6 |
| 验收 | Unity | 先 Gesture Manager Play 一遍（铁律 5），再 Build & Test |

## §6 换头会连带坏掉的东西（比几何更麻烦）

- **表情全废**：Kipfel 头上有 373 个形状键（`vrc.v.sil` 那套 + 各类表情），换头后这些名字在场景里不复存在 → 现有表情菜单 / 动画 / 表情图标全部要重指或重做。
- **视素命名不同**：Shipilka 是 `vrc.v_aa` 体系，Kipfel 是 `vrc.v.sil` 体系 → 描述符里的 Viseme 配置要重设。
- **眼球**：Kipfel 的眼球是头网格里权重给 `Eye.L` / `Eye.R` 的那部分；Shipilka 用 MMD 体系命名 → 注视骨与 Eye Look 设置要重指，否则眼睛不会看人。
- **面捕**：`resources/01_模型本体/作者-Shipilka/[Shipilka - シピルカ]/` 下有官方 **Face Tracking Add-on**（`Shipika - Face Tracking Add-on.unitypackage`，4,791,609 B）—— 若走 Shipilka 的脸，优先用它，省掉自己映射。
- **性能**：Shipilka 单是头发就 11,868 顶点，比 Kipfel 整个头（6,944）还大 → 性能评级会掉；上传前按 `07-验证与排查/07-01-bake-and-validation.md` 走一遍。

## §7 未做 / 未验证

- **没有**在工程里真正做过一次换头；本文是源文件实测 + 推算。
- Unity 侧 `Merge Armature` 遇到两边同名骨（`Head`/`Neck`/…）的实际表现**未实测**；MA 的 prefix / 骨骼归属写法要现做现验。
- 脖子台阶的实际可视程度**没渲染过**（未做对照图）。
- 头网格的 UV / 贴图边界（脸与脖子接缝处贴图是否连续）**没查**。
- 头发要不要一起搬、表情菜单用哪一套、是否保留 Kipfel 的猫耳 / 尾巴 / 帽子 —— **待用户拍板**。

## §8 相关

- 接缝与几何验收的一般原则：`05-01-outfit-fit-and-clipping.md`。
- Blender 改网格、fbx 替换网格的前提：`05-02-bind-animation-physbone-fbx.md` §4/§6。
- 基底穿模正解与 Blender headless：`../00-基底/00-05-mesh-blendshape.md`。
- 验收与证据分层：`../07-验证与排查/07-01-bake-and-validation.md`、`../07-验证与排查/07-02-evidence-and-ownership.md`。
- 铁律 4（MA 优先）与铁律 5（Gesture Manager 必装必用）见 `../../SKILL.md`。

## §9 外部教程方案（三个换头流程；**外部来源，未在本机复现**）

> 口径：三套方案都是 **Unity-only 换头**（不碰 Blender 也能把头接上）。下面按「方案 → 步骤」记；本机**没有**照做验证过，只作为方案与踩坑索引。来源见 `ACKNOWLEDGEMENTS.md`。

### §9.1 方案 A：最详细的 Unity-only 换头流程

示例：**戚风（せいふぉん）身体 + rusk 头**。

0. **面捕先决**：要做面部追踪就别用「原版」预制件，改用带面部/面捕的预制件，且**第一步就装面捕**——有些面捕安装会替换 FBX，最后再装容易前功尽弃。
1. **比大小**：两个 prefab 都拖进场景 → 用「手」或「下巴」当参考比高度 → 等比缩放供体（示例 0.99 → 0.895~0.9 慢慢试）→ 前后拖动对齐；**两个 prefab 都要先 Unpack（解压）**。
2. **只留需要的部分**：供体只显示头（身体/衣服隐藏），宿主隐藏脸、显示身体；把「位置已合适」的这两个对象 **Ctrl+D 备份**一份（视频反复强调备份）。
3. **骨骼移植**：找到供体的 head 骨骼层，把**整个 head 层**放进宿主的 head 里（neck 那层可以不要）；再把供体的头/脸网格拖进宿主。
4. **清理废骨骼**：删宿主原头的废骨骼（不删会「回归原点」）、宿主原眼骨、不需要的耳朵动骨（PhysBone）→ 进 Play 确认动骨正常。
5. **FX / Animator**：用**供体（头）的 FX 控制器**替换；Animator 的 Avatar 以**身体**为准；然后逐个修丢失引用 —— 嘴型/Viseme 的 Face Mesh 指向新脸网格、`Eye.L`/`Eye.R` 指向新头的眼骨、眨眼/看上看下等动画重新指定。
6. **★ Gesture 层千万不要动**（手势要用身体的手势层），FX 全部用供体头的。
7. **比例微调不必进 Blender**：直接改新头的 XYZ 缩放（例如 Y 拉长脸），脖子长短也有余量；嫌丑就加项圈之类小配件遮。
8. **★ Root Bone 修复**：新头/新头发/耳朵的 SkinnedMeshRenderer **Root Bone 丢失**要重新指到原 head 根骨骼，否则**别人一靠近就消失**（距离剔除）。
9. **举一反三（移植衣服/尾巴/配饰）**：比高度整体缩放（示例 0.95~0.955）→ 解压 prefab → 删掉不用的部件（袜子/内衣等贴身件难适配）→ 先判断涉及哪些骨骼（hips/spine/chest/neck/shoulder/arm/hand/leg/foot）
  → 删掉用不上的手指骨**无影响**，但删 neck 会让吊带归零（**必须保留**）→ 把整套骨骼**重定父级**挂到新身体的同名骨骼 → 脚不合适就调 left foot / lower leg（右键**复制组件**粘到另一侧，数值**加负号**）
  → PhysBone 碰撞球位置差不多可以不动 → 最后用批量改名插件整理骨骼名。
10. **拆单个小配件（示例：mint 发卡）**：Unity 里右键找到 FBX 路径并**先备份 .fbx** → Blender 全选删除 → 导入该 FBX → 只留目标网格、其余全删 → **骨骼千万不要动** → 导出只勾「骨架网格」并**取消「添加叶骨」**，
  变换模式要试（自定义 / 全部局部 / 单位缩放：导错会让 FBX「炸」）→ 回 Unity 只剩所需部件，再绑到 head。
11. **收尾**：把用到的 FBX 都按上面办法精简；新加部件的 Root Bone 重新指定并**重新命名做开关** → 上传测试。
12. **评论补充**：脖子太长/拉丝可能要进 Blender 删面并**带一部分身体前接头**；眼追失效可关掉 eye look 走面捕；参数用「头的」、菜单自己搓；头与身体连体的模型必须进 Blender 切且保留脖子一点身体；牙齿/舌头等要 Blender 更进阶。

### §9.2 方案 B：仅头部的简化流程

该教程**没有字幕轨**（`subtitles=[]`、`need_login_subtitle=false`）→ 用「抽帧读屏上字幕」的办法读的（`bili.mjs download --qn 80` 拿 1080p → ffmpeg 抽帧 → 读图），另参考其 14 条评论。

1. **先调大小**：把被换头的模型（宿主身体）大小调整到与「头的原模型身体」差不多大（屏上字幕原文：「将被换头的模型身体大小调整至与换的头身体差不多大」）。
2. **只留头**：删掉供体「头部以外的东西」（字幕：「删除头部以外的东西」）—— 场景里只剩 `Selestia_body` 这类头网格。
3. **把新头的 Body 网格拖到外面那层**（字幕：「将新头的 Body 拖到外面那层」），即网格从供体层级里提出来挂到宿主层级。
4. **删多余骨骼**：在层级里右键 Delete 掉不用的骨骼（例：带 PhysBone Collider 的 `Hips` 之类）。
5. **Animator / Avatar**：Avatar 指向**宿主身体**的 avatar（例 `ICHIGO_v1.04Avatar`）；此时描述符里 **LipSync 的 Face Mesh 会变成 `Missing`** → 必须重指到新脸网格。
6. **换 Playable Layers**：把 Base / Additive / Gesture / Action / FX 全部换成**供体（头）**那套控制器（视频里左右两个 Inspector 对比 `ICHIGO_*` 与 `SELESTIA_*`）。
7. **★ 替换 FX 里 `Left Hand` / `Right Hand` 的手势与表情动画**（字幕原文）—— 即手势层的动画要用新头/新手的动作。
8. **Gesture Manager 3.9 验收**：`Enter Play Mode` → 看手势列表（FIST / Open / FingerPoint / Victory / Rock&Roll / Gun / ThumbsUp）与径向菜单表情全部生效。
9. **评论里的收尾**：脖子接缝「上 PS 改色」+「把头缩小（小数点后三位慢慢试）再拉长」；**视角球（View Position）要按比例同步调整**（有反馈同比例缩放后 VRC 里实际仍在额头，UP 说是 SDK 的 bug，只能一点点试）；
  换头后 **Avatar 缺失**时「把 FBX 里的 avatar 拖进去」；误删会报「缺少正确的部件」；删掉头部的 Animator 组件动画会消失（动画还在文件里，要自己塞回 base/FX）；**换身体**可以只改 Animator 里的 Avatar。

### §9.3 方案 C：付费插件与手绑两条路

示例：**马努卡的头 + 巧克力的身体**。方案两条：

- **A. 付费换头插件**（视频 desc 给的是 <https://goho-cheat-vrc.booth.pm/items/4519446>，约 800 日元）：工具里分别选「头用哪个模型」「身体用哪个模型」→ 自动锁定工程里的预制件并合成 → 一键出结果，头直接对上；剩下只是**皮肤/材质色差**要自己改。
- **B. 手绑（不用插件）**：① 两个模型都导入 → **鼻尖对鼻尖**重叠、再按**肩**对齐；② 只留供体的头，其余身体部件全删，把「头」当成一件**头发/物品预制件**看待；③ 把宿主的**脸关掉**；④ **优化骨骼**：只留头部链（Hips/Spine/Chest → Head），
  其余身体骨骼全删（留着既没用又占**骨骼上限**）；⑤ 用 **MA 组件（Merge Armature 一类）**把头绑到宿主的 Head 上（**必须挂在头骨骼下层**才能绑）；⑥ 供体自带的**脚本组件必须删掉**；⑦ 头偏大 → 整体**缩放 0.9** 再手动微调位置、确认脖子位置；
  ⑧ 脖子肤色不一致 → 自己 PS/改色；⑨ **Gesture Manager** 进 Play 验收（头接上了，但手势/表情失效）；⑩ **修 FX**：把宿主 FX 的表情层换成供体头的 FX 动画（或直接用 **FaceEmo** 更省事），
  再把脸部从基础层下拖出去/重指路径 → 回 Gesture Manager 确认表情生效；⑪ 收尾：**头发跟头、衣服跟身体**，头饰可能错位 → 按 PhysBone 的办法修。

### §9.4 三套方案的共识（可当 checklist）

1. **不用 Blender 也能换头**：Unity 里比大小 → 解压 prefab → 只留头 → 骨骼移植/重定父级 → 换 Avatar 与 FX → 修丢失引用 → Gesture Manager 验收。
2. **必须先 Unpack**（铁律见 `../../SKILL.md`「导入完立刻完全解压」）——三套方案都要求动手前解压 prefab。
3. **Gesture 层保留身体的、FX 全部用头的**（三套方案一致）。
4. **换头真正的重活是「重指引用」**：Face Mesh / Viseme、`Eye.L`/`Eye.R`、眨眼与看上看下、`Left Hand`/`Right Hand` 手势动画。
5. **两个隐形杀手**：① SkinnedMeshRenderer 的 **Root Bone** 没重指 → 远处/靠近就消失；② **视角球 View Position** 没同步缩放 → 在 VRC 里视角不对（SDK 已知差异，只能微调试）。
6. **脖子接缝**三条路：PS 改色 + 头缩放（小数第三位慢慢试）/ 加项圈衣领遮 / 进 Blender 改面（带一段身体前接头）。
7. **要面捕就第一步装**，且用带面部的预制件。
### §9.5 接缝与肤色的「颜色迁移」工具（2026-10-18 补，外部来源、未实测）

§9.4 第 6 条那条「PS 改色」可以在 Unity 内自动做：**NoNlab 双模颜色同步模块**（`Ninsterrrr/NoNlab-`，**MIT**，V1.0，2026-10-16）——把供体（头）的颜色按像素迁移到宿主（身体）贴图上，省掉手绘 / PS。演示来源见 `ACKNOWLEDGEMENTS.md`。

- **用法要点**：输出模式 **`Texture`** + 迁移模式 **`Direct Replace`**（README 明确：头与脖子匹配用这组）；接缝处理把「影响距离」调大、关「法线加权」、降低混合强度；目标贴图需 **Read/Write Enabled**（工具带一键修复）。
- 双路径：顶点色（快，10–50ms）/ 贴图（精确，面级反投影 + BVH，604K 像素 ~92ms 多线程）；**可 Ctrl+Z 撤销**。
- 详细参数与安装见 [`../06-构建上传与备份/06-03-recover-project-from-uploaded-avatar.md`](../06-构建上传与备份/06-03-recover-project-from-uploaded-avatar.md) **§7**。



---

## §10 Kipfel 1.2.0 去衣 / 去发 / 少年化（2026-10-18 本工作区实测）

**网格结构（实测）**：

- `Body`（SMR，372 形态键）＝ **只有头和脸**（eye/mouth/eyebrow 表情键、`other_earOFF` 猫耳开关）；4 个 submesh：`Body_skin` / `Hair`(124 tri) / `Body_eye` / `Body_alpha`。
- `Body_Base`（SMR，51 形态键）＝ **整个身体**（躯干+四肢），单 submesh `Body_skin`；**内裤画在贴图上**（不是独立网格），换 `Body_skin_naked.mat` 即裸体。
- 独立网格（22 个）：`Hair_Back/Base/Bun/BunRibbon/Front/Side`、`Outfit_Cap/Shirts/Shorts/Socks/Vest`、`UnderWear_Pants/Tops`、
  `Item_Bag/Boots/DressingPad/FishBone/Glasses/NameTag`、`Cat_Ear`(默认 inactive)、`Cat_Tail`、`FishToy`(MeshRenderer)。
- 去衣去发＝对这些网格 `DestroyImmediate(gameObject)`（只删渲染对象；armature 骨骼与 PhysBone 保留，不打断 FX 参数）→ 剩 `Body` + `Body_Base` 即素体。

**少年体型**：`Body_Base` 有 `Bust_slim` / `Waist_slim` / `Leg_slim` / `Ankle_slim`（0–100），本次设 100/80/80/50；另有 `High_heel` / `Foot_down` 脚型键。

**材质**：`Material/PC/Body_skin_naked.mat` 是官方自带裸体皮肤材质（贴图 `Texture/PC/kipfel_body_skin_naked.png`），直接换 `sharedMaterial`，不用改贴图。

**配件挂接（MA MergeArmature）实测**：

1. **整骨架服装**（NoemHoodie：`armature.001` 含 Hips/Spine/Chest/Neck/Head/四肢全套）→ 加 `ModularAvatarMergeArmature`（`mergeTarget` 指 `Kipfel/armature`，
  字段是 `AvatarObjectReference`，要用 `.Set(GameObject)` 设，不能直接给 objectReferenceValue）→ **NDMF Manual Bake 合并成功、衣服穿戴正确 ✅**。
2. **只有头骨的头发 rig**（LooseShort：`armature/Head` 下直接挂头发骨骼）→ **MA MergeArmature 失败**：bake 后骨骼被改名 `Head$<guid>`、网格留在角色原点（改 mergeTarget 指 Neck 也一样；
  MA 的 `RecursiveMerge` 用 `SetParent(..., worldPositionStays=true)`，源骨骼必须在目标位置预先摆好才合得上）。**✅ 已验证修法（2026-10-18 实机，黑短发成功贴头）**：**不要用 MergeArmature**，
  把头发对象直接挂到 `Kipfel/armature/Hips/Spine/Chest/Neck/Head` 下：

   ```csharp
   hair.SetParent(head, false);
   hair.localPosition = Vector3.zero;
   hair.localRotation = Quaternion.Inverse(head.rotation); // 让头发根节点世界旋转=identity（= FBX bindpose 坐标系）
   hair.localScale = Vector3.one;
   ```

   → 头发正确贴合头部、跟随头动（PhysBone 照常工作）。另注：MA 的 **Setup Outfit** 工具对头发无效（它要靠 humanoid `Hips` 找骨架，头发 rig 只有 Head）。
3. `execute_code` 里写 `AssetDatabase.DeleteAsset` 会被安全模式拦；删除用文件系统删（`cmd /c rd`）+ `AssetDatabase.Refresh()`。

### §10.1 发色改造：黑发 → 白发（2026-10-18 实机验收）

**发现**：LooseShort 自带 `black` / `gray` / `baige` 三个变体贴图，**没有白色**；但发色只是 lilToon 材质的 `_MainTex` + `_Color`，可以自己造。

**做法（本机实测）**：
1. 用 Python+Pillow 从原图 `Texture/ls_hair.png` 做亮度重映射：`t = 198 + 57 * (L/255)^0.55`（暗部 ~198、亮部 255，保留层次），B 通道 +5 做**冷白**；**alpha 强制 255**（原 alpha 本来就是 255，防半透明）。
2. 复制材质：`AssetDatabase.CopyAsset("…/ls_hairmat_black.mat", "…/ls_hairmat_white.mat")` → 新材质 `_MainTex` 指新贴图、`_Color`≈(1.02,1.02,1.04)、`_OutlineColor` 调浅 (0.55,0.55,0.62)。
3. `AssetDatabase.ImportAsset(path, ForceUpdate)` 后**重新 `SetTexture`**（否则引用还是旧图）。

**验收（自截图判断）**：第一版用 gamma 0.40 提亮 → **发灰、偏银灰**，不满意；第二版改成 198~255 线性+gamma 0.55 → **干净冷白**、发丝深灰分层、轮廓线不刺眼，配黑卫衣对比好看 ✅。

### §10.2 加回猫耳：从 prefab 复制 Cat_Ear 网格并重绑骨骼（2026-10-18 实机验证）

**发现**：Kipfel 的猫耳是**独立网格 `Cat_Ear`**（1,198 verts，材质 `Hair`），在 `Kipfel.prefab` 里存在但 **`active=false`**（由菜单开关控制）；去衣去发时被一起删掉了，但**耳朵骨骼 `KemoEar.L/R` 与 `PB_CatEar.L/R` PhysBone 都还在 armature 里**。

**做法（不用重新导入包）**：
1. 临时实例化 `Assets/MOCHIYAMA/Kipfel/Prefab/Kipfel.prefab`，取 `transform.Find("Cat_Ear")` 的 `SkinnedMeshRenderer`。
2. 在场景 `Kipfel` 下新建空物体 + `SkinnedMeshRenderer`：`sharedMesh = Cat_Ear`，材质换成白发同色材质（见 §10.1 的重映射做法）。
3. **181 根 bones 按路径重绑**：把临时实例里每根 bone 的「相对 prefab 根的路径」（如 `armature/Hips/Spine/Chest/Neck/Head/KemoEar.L`）在场景 `Kipfel` 下 `Find` 同路径；`rootBone` 同理（`armature/Hips`）。本次实测 **missing=0**（我们删网格时保留了全部骨骼）。
4. 设 `updateWhenOffscreen=true`、激活物体；`localBounds` 抄 prefab 的即可。
5. 删掉临时实例，保存场景。

**判据/验收**：正面/3-4/背面截图耳朵位置正确、耳根埋进头发无缝隙、白色与白发一致、无穿模 ✅。耳朵会跟着 `KemoEar` 骨的 PhysBone 摆动（**未进 Play 模式验证**）。

**★ 颜色匹配（用户 2026-10-18 特别要求，实测达标）**：耳朵材质**不能**只把 Kipfel 原版 `Hair.mat` 复制成白色 —— 它的 lilToon 阴影/描边参数与头发（`ls_hairmat_white`）不同，渲染出来会偏灰、偏暖。正确做法：

1. **以头发材质为模板**新建耳朵材质：`new Material(ls_hairmat_white)`（继承同一套 lilToon 参数），
  只把 `_MainTex` / `_BaseMap` / `_BaseColorMap` 换成**耳朵 UV 可用的重映射贴图**（`kipfel_hair_white.png`，按 `t = 208 + 47*(L/255)^0.5`、B+6 冷白生成），
  `_OutlineWidthMask` 保留耳朵原本的 `kipfel_hair_mask_outline`。
2. 耳朵贴图亮度对齐参考：`ls_hair_white` 的均值亮度 **240.1**（p10/p90 = 238/243）→ 耳朵贴图生成后均值控制在 240 左右。
3. **实测校验（1920×1080 截图取样）**：耳朵白色均值 `(244.4, 242.0, 247.8)` vs 头发 `(243.4, 241.5, 249.2)`，**每通道差 ≤1.5/255、亮度差 0.5/255** → 视觉无色差 ✅。

**采样小坑**：MCP `manage_camera` 返回的 `imageWidth` 是**预览尺寸（640×360）**，但落盘 png 实际是**屏幕分辨率（本机 1920×1080）** —— 像素取样要用**文件真实尺寸**，否则坐标全落到背景上（本次第一轮就取到天空色）。

### §10.3 少年感发型候选与「头骨偏移型 rig」上发法（2026-10-18 实测）

**素材库里的 Kipfel 可用短发候选**（`03_头发`，含非 `Kipfel/` 子目录里发现的）：`Looseshort`（松散短发，キプフェル対応）、`小短发` = **AsymmetricalHalf-upHair VerMAMEHINATA&Kipfel**（短款半扎，
Kipfel fbx + 白色贴图齐全）、`Pyoko hair`（蓬松短发 + 猫耳，Kipfel fbx）、`mochibob`（bob，兼容性未查）、`2A Hair`（**Kipfel.prefab 实际是长 bob，不是预览里的短毛** —— 预览那张短毛是 Sio 版本，不适合少年感）。

**AHH（小短发）上发方法（实测）**：
1. 它的 hair rig 头骨不在原点：`AHH_Kipfel.prefab/AsymmetricalHalf-upHair VerKipfel/Armature_AHH_K/Head` 的 localPos ≈ `(0, -0.002, 0.647)`、
  rot ≈ `(86.1,180,180)` —— 属于「头骨带偏移」的 rig，不能像 LooseShort 那样把根节点世界旋转设成 identity。
2. 用 **Head 骨骼世界对齐**：实例化后父到 `Kipfel`（保持世界变换），然后
   `dq = kipHead.rotation * Quaternion.Inverse(hairHead.rotation)`；`inst.rotation = dq * inst.rotation`；`inst.position += kipHead.position - hairHead.position` → 头发头骨与角色头骨完全重合（实测 world 坐标一致）。
3. 材质：包内自带 `Material/Kipfel/AHH_Color_White.mat`，但 `Texture_white.png` 均值亮度只有 **177.6（偏灰）** → 复制材质为 `AHH_Color_White2.mat`，把贴图按 `t = 200 + 55*(L/255)^0.5`（B+6）重映射到均值 **245.4**，与白发统一。
4. 5 个 SMR：`Back/Front_A/Front_B/Hane/PonyTail`，全换白色材质；旧发型（LooseShort）**隐藏保留**在 Head 下，随时可切回。

**C 款 Pyoko hair**（`Assets/QW/PyoikoHair/PyokoHair_Kipfel_R.prefab`）**也属头骨偏移型**：root 下 `Armature/Head` 的 localPos ≈ (0,0,0.653) → 同样用 Head 世界对齐法；
白色材质由 `Mat_PyokoHair_kipfel.mat` 复制 + `Tex_PyokoHair_2.png` 重映射（均值 163.5 → 244.0）得到。

**判据**：正面/3-4/全身/背面四角度截图正常、白色与猫耳同色、无穿模 ✅；当前场景默认用 AHH，三款都保留在场景里交给菜单切换（见 `03-菜单与参数/03-05-ma-menu-hair-style-color.md`）。

## 11. 双头部一键切换（MA ObjectToggle 组级切换 + 目标头表情层移植）——本工作区实测 2026-10-25

**结论**：可行，且不需要 Blender。核心不是「换头骨」，而是「把目标头整组搬到宿主骨架下 + 用菜单切换两套静态状态 + 让每个头用自己的表情层」。

### 前提
- 宿主头是**独立网格**（本工程：`Body`=猫头、`Body_Base`=身体；Kipfel 天然如此），否则无法单独隐藏宿主头。
- 目标头必须能整体作为一组对象（本工程：`小狼2.0` 的 4 个 SMR：`Body` / `Cat_Ear` / `Hair_Back` / `Hair_Base`，骨骼 125 根、rootBone=Hips）。
- 两个头的形态键名不同 → **必须移植目标头自己的表情层**（手势 + 眨眼 + 视素），不能只借宿主表情。

### 步骤（本工程实测）
1. 在宿主根下建 `狼头` 组，`Instantiate` 目标头 4 个 SMR（保留世界变换后改父到组内）。
2. **骨骼映射**：按「相对路径」把目标头 bones 映射到宿主骨架（`Hips/Spine/Chest/Neck/Head/...` 相对路径精确匹配；再按唯一名称回退）。实测 125 根：猫侧精确 62、克隆发骨 23、剩余 40 为狼头披风骨（`Cape_*`，无对应，无害）。
3. **克隆目标头专属发骨链**：狼头的 `Head/Hair_Root` 子树（23 根）要从目标头像克隆到宿主 Head 下（否则 SMR 的 bones 指向猫骨 → 形变错乱），并**删掉克隆上的 VRCPhysBone**（否则狼发在猫骨架里乱抖）。
4. **菜单**（MA）：`菜单开关/头部` = MenuItem(SubMenu, MenuSource=Children) + MenuInstaller(installTargetMenu=`Kipfel_ExMenu`)，子项：
   - `猫头`：Toggle，参数 `HeadType`=1，`isDefault=true`，OT 1 行 `{Body, Active=true}`；
   - `狼头`：Toggle，`HeadType`=2，OT 12 行：`{狼头,true}` + `{Body,false}`（猫头）+ `{头发,false}`（狼发1）
  + `{头发2,false}` + `{耳朵尾巴,false}` + `{睡衣/PJ_Hat,false}` + `{睡衣/PJ_HatRibbon,false}` + `{眼镜,false}` + `{眼罩,false}` + `{WingA,false}` + 
  `{WingA/WingA,false}` + `{White,false}`。
   - `HeadType` 用 MA Parameters 声明：Int、default=1、saved、synced（放在已有的「一键穿衣设置」MA Parameters 里，builder 要幂等去重）。
5. **目标头表情层移植**：把目标 FX 控制器复制一份，只留面部相关层（本工程狼 FX 20 层 → 留 `FaceDefault` / `Left Hand` / `Right Hand` / `BlinkAuto` / `Viseme Lock`），
  把 clip 里所有 path `Body` 重写为 `狼头/Body`（先复制 clip 资产再改 path），最后把 5 层追加到宿主 FX 控制器末尾（本工程 10 → 15 层）。
6. **场景状态**：`狼头` 组 `SetActive(false)`（Rest Pose 静态状态 = 猫头），猫头/猫发/猫耳尾/头饰保持默认可见。

### 判据 / 烘焙后的实际行为
- 烘焙后 AAO 会把狼头 4 个 SMR **合并成一个网格**，并自动重指 clip 路径 → `狼头/$$AAO_AUTO_MERGE_SKINNED_MESH_1`，曲线数被裁剪（本工程 `LilLeo_face_zeroshape` 193 → 58 曲线）——正常，不要因此以为 clip 丢了。
- 烘焙后 MA 生成 `MA Responsive: 狼头` 层（clip 设 `狼头=1`）与 `MA Responsive: Body`（`猫头=1 / 狼头=0`），猫发/耳尾/头饰层里均有 `狼头=0` 行 → 切 `HeadType=2` 时全套联动。
- 编辑态直接模拟（`狼头.SetActive(true)` + 隐藏猫头/猫发/耳尾/头饰）渲染对比：头部区域 **30%–36%** 像素不同（640–720px 图），可作「切换真的生效」的数值判据。

### 坑（都踩过）
- OT **组行顺序**：狼头组那一行必须是 `Active=true`，其它隐藏行 `false`；全写 false 会导致切狼头时狼头反而被隐藏。
- `狼头` 组在场景里必须 **inactive**，否则 Rest Pose（他人屏蔽动画）会双头同显。
- MenuItem `isDefault` 与 MA Parameters 声明要一致（默认 = 猫头）。
- 克隆发骨链时不要保留 PhysBone；也不要把克隆骨挂到目标头对象外（必须挂宿主 Head 下）。

### 未验证（NOT_RUN）
- 游戏内实际菜单点击、多人同步、远端可见性、狼头表情手势的实际观感 —— 需 Build & Test 或进游戏；GM 菜单点选本轮未跑。

### 换头与「服装/配饰开关」的联动：不要再写「按头型强制显隐」的自定义层（本工作区实测 2026-10-26）

**症状**：换头做完后，用户报「狼头模式下，衣服菜单里关掉帽子对狼头不生效」——狼头出现时帽子仍被强制显示（同理眼罩/眼镜/おめん/visor 全都不受菜单控制）。

**根因**：早期为了「切狼头时把猫头专属头饰藏起来」，我在宿主 FX 控制器里手写了一层 `WolfHideCatHeadGear`（2 个状态：`Idle` 播 `WolfHideGear_Show`（这些配件 `m_Enabled=1`）、
`WolfMode` 播 `WolfHideGear_Hide`（`m_Enabled=0`），`ANY → WolfMode` 条件 `HeadType == 2`）。这一层是**手写常量动画**，
会盖在 MA 为每个对象生成的 `MA Responsive: <对象名>` 层之上（层序靠后/权重满），于是：**任何头型相关的强制显隐，都会压过用户点菜单产生的 MA 反应层**。

**正确做法**：把「切狼头时要隐藏的东西」全部塞进 **`狼头` 这个 Toggle 自己的 `ModularAvatarObjectToggle` 行**，让 MA 在烘焙期按参数生成统一的反向 toggle：

| 行目标 | `Active` | 说明 |
| --- | --- | --- |
| `狼头`（组根） | `true` | `HeadType==2` 时显示 |
| `Body`（猫头 SMR） | `false` | 切狼头时藏猫头 |
| `耳朵尾巴` | `false` | 猫耳/猫尾 |
| 猫发各渲染器（`头发/front`、`头发/back` … 逐个） | `false` | **必须指到子渲染器，不能指 `头发` 根**（见下） |

**坑 1：`ModularAvatarMergeArmature` 会在烘焙时销毁被合并的根对象。** 猫发根 `头发` 上挂着 `MergeArmature`，烘焙后它**不再存在**（内容被并进宿主 `armature/.../Head/头发`），所以 OT 里那一行 `{头发, false}` 会变成空引用/无效行。**要指到它下面的具体渲染器子对象。**

**坑 2：不要在自定义层里写「按头型隐藏服装」**。服装件（帽子/外套/眼罩…）本来就有自己的菜单项和 `MA Responsive` 层；一旦被强制层覆盖，用户关不掉。若某件衣服确实只适配某一个头（例如狼头专属头饰），正确做法是**给它单独一条菜单项 + 自己的 OT**，而不是挂在头型层里。

**判据（烘焙后 FX 控制器里查）**：
- 只应存在 `MA Responsive: <对象名>` 层 + 你显式写的 `换头系统` 层；
- `PJ_Hat` / `PJ_HatRibbon` / `Eyemask` / `おめん` 这类**服装件**的 `m_IsActive` 曲线，只应出现在 `MA Responsive: <它自己>` 层；
- 切 `HeadType=2` 后，逐件点菜单关/开，`activeSelf` 必须跟着变（本工程 4 组断言见 [07-01-bake-and-validation.md](../07-验证与排查/07-01-bake-and-validation.md) §11）。
