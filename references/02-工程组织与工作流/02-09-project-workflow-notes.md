# 工程流程、菜单组织与版本管理（实战笔记）

> **性质**：**外部教程笔记**，不是本工作区实测 —— 除标注外**一律按「未验证」对待**；KB 已有的只写「差异 / 增量」，不重复。
> 课程清单与视频来源见 `ACKNOWLEDGEMENTS.md`。

---

## 1. 与 KB 的差异总表（先看这张）

| 主题 | KB 现状 | 本教程带来的**增量** | 落点 |
|---|---|---|---|
| **形态键调好了还是「细胳膊细腿」** | 只讲形态键本身（`00-基底/00-05-mesh-blendshape.md`、`05-01-outfit-fit-and-clipping.md`），没有「被动画覆盖」这一根因 | ★ **素体 FX 层的 `Entry` 动画会在进 Play/游戏时把衣服对应的形态键重新设成防穿模值**，手调的显示状态被覆盖 | 本册 §5 + `SKILL.md` 症状速查 |
| **MA 菜单的组织规范** | `03-菜单与参数` 有 MA 菜单配方 | 每套衣服自带一个 **MA Menu Installer** 菜单控制器，再「安装」到总菜单下；`Object Toggle` 的拖拽细节 | 本册 §4 |
| **默认启用某套衣服** | 无 | 参数值设为 **0** 的衣服＝默认启用；「默认启用」点不动是**自动配置参数值**在拦，先取消自动配置再手填 0/1/2/3… | 本册 §4 |
| **git 回档** | `06-构建上传与备份` 只有一句「推荐 git + .gitignore」 | 完整命令流 + **detached HEAD** 陷阱 + 分支分叉做法 + 在 VRC 工程里实操时 Unity 占用文件的处理 | 本册 §6 |
| **妆容 / 瞳色替换** | `04-外观与表情` 讲表情曲线与颜色，无「换脸贴图 / 换瞳色菜单」步骤 | 主色 / 第二主色 / 第三主色三层替换法；瞳色菜单＝**复制面部材质球 → MA Material Swap + ShapeChanger 关 `iPupilX` 眼仁** + int 参数互斥 | 本册 §7 |
| **FaceEmo** | KB 只有「用 FaceEmo」的出处引用，无流程 | 手势↔表情映射表（7 种）、用形态键捏脸、**眨眼动画会覆盖表情**的处理 | 本册 §8 |
| **AudioLink / RK Player** | **KB 完全没有（grep 0 命中）** | 两个插件的安装与绑定全流程 | 本册 §9 |
| **VPM 国内镜像源** | KB 记了本机 6 个源 | ★ 教程自建镜像源（国内直连）+ 官方/精选/第三方源清单 + 「预发布要勾选」 | 本册 §2 |
| Blender 基础操作 | `<TOOL_ROOT>\.skills\unity-mcp\skills\blender-to-unity`、`00-基底/00-05-mesh-blendshape.md` 讲改网格，无「Blender 入门操作」 | 导航/快捷键/飞行与行走模式/点线面/挤出与内插 | 本册 §10 |

---

## 2. ★ VPM 源与镜像（EP1/EP2/EP7 置顶评论 + EP3 热评）

**教程自建「VRC 改模包镜像源」**（2026 年上线，国内网络直连、不用梯子）：

```text
https://remote.rosmontiscloud.xyz/Rosmontis_Cloud/vrc-vpm-mirror/raw/branch/main/packages/vpm.json
```

- 用法：ALCOM / VCC → **添加存储库** → 粘贴 URL；之后新建项目就能直接选其打包好的 lilToon、Modular Avatar、VRChat SDK、Gesture Manager、FaceEmo、贴图优化工具等。
- **教程二级评论补充**：在 ALCOM 里点「筛选包」时要**把「预发布」属性勾上**，否则识别不到其提供的预发布版本。
- EP7 起他把镜像源做成了「新建项目默认可用」的形态，并说「这些报错不用管，是从 VRC 官网拉取失败，我们用的是镜像源」。

**评论区整理的其它源**（EP3 热评，直接抄录，未逐个验证）：

| 源 | URL |
|---|---|
| VRChat 官方 | `https://packages.vrchat.com/official?download` |
| VRChat 精选（curated） | `https://packages.vrchat.com/curated?download` |
| ArchiTech Assets | `https://vpm.techanon.dev/index.json` |
| Yamadev packages | `https://vpm.kwxxw.net/index.json` |
| 中文镜像站（VRCZ） | `https://vpm.vrczh.org/vpm/curated` —— **EP3 评论区反馈「用不了了 / 镜像站啥也看不到」（2026-10）** |

---

## 3. EP1 增量

- 软件三件套：Unity Hub（注册用 QQ 邮箱）→ 装 Unity **2022.3.22f1**（约 30 分钟）→ **ALCOM**（「VCC 的平替」）。与 KB `01-06` 一致，不重复。
- **工程内归类习惯**：素体 / 衣服 / 妆容各自建文件夹，别把 `Assets` 堆满（他从第一集就强调）。
- **模型发紫＝缺 shader**：亚洲可爱风模型基本是 **lilToon**；装完就正常（KB 已有，此处复述）。
- **素体形态键的 `off` 系列**：`Body_Base` 的 BlendShapes 里有一组 `… off`（`shoulder off` / `upper arm off` / `elbow off` / `chest off` / `spine off` / `hip off` / 
  `toe off`…）。**隐藏素体自带衣服部件、穿上新衣服后必须把它们调回来**，否则素体像「提线木偶」露内部。做法：逐个改数值看模型是否「长回来」。
- **上传面板逐项**：SDK → Control Panel → 登录（账密 + **邮箱验证码**）→ `visibility`（private 只有自己可用、别人克隆不了；public 可克隆）→ name / 标签 / `content warnings`（A = 成人、B = 两者、
  graphic violence）/ description / 缩略图（可选 PNG，或**在场景里调相机 capture**，滚轮＝镜头移动速度）→ 有 `Auto Fix` 就点 → 平台选 **Windows** → `Build & Publish`。
- **上传卡住的判据（教程回复评论）**：**必须在「场景视图」里编译**；编辑器若是蓝色背景，说明你还停在**模型预览状态**（双击 prefab 进来的），模型没加进场景 → 点上传只会卡进度条。

---

## 4. EP2 增量（MA 实战）

### 4.1 穿衣服的执行顺序

1. `Ctrl+S` 先存场景（**只能存进 `Assets/`**）。
2. 导入素体 → **装 lilToon**（否则紫）。
3. 导入衣服：**先材质包、后模型包**（作者常把 `textures` / `material` 与 prefab 分开发）——顺序反了会出错误贴图。
4. 把衣服 prefab **拖进模型层级下**，右键 → `Modular Avatar → Set up Outfit`（这才是真正「穿上」）。
5. 隐藏素体自带的衣服/袜子/鞋，**并把素体形态键调回显示**（见 §3 的 `off` 系列）。
6. 验收：`Tools → Gesture Manager Emulator` 进 Play，转动视角看穿模（KB 铁律 5 已写）。

### 4.2 三种资产的差别（他专门讲）

| 类型 | 是什么 | 用法 |
|---|---|---|
| `FBX` | 只有网格（可能带动骨、没材质） | 基本不用 |
| `prefab`（预制件） | 网格 + 材质 + 绑定，开箱可用 | **首选**，拖进场景/层级 |
| `material`（材质球） | 只有材质 / 贴图 | 用于替换、换色 |

### 4.3 菜单做法（与 KB 不同的组织方式）

- `右键模型 → Modular Avatar → Extract Menu`：把**素体原本的菜单**提取出来；不需要的项（内衣、拖鞋、袜子…）直接删。
- 新建 `GameObject`（建议命名 `outfit`）→ `Add Component → MA Menu Item` → 类型改 **Sub Menu**，再逐级加菜单项。
- 菜单项：选图标（他自己拍的缩略图）→ 选参数（新建 `outfit_choose`，**int 型**，用于多套衣服**互斥** 0/1/2/3…）→ 加 `MA Object Toggle` 组件，把要开关的对象**按住拖**进去。
  - ⚠️ **拖进 Object Toggle 时要点住拖拽，不要双击**（双击会进 prefab 编辑模式）；也不要单击目标对象（会切成选中）。
- **每套衣服自己带一个「衣服控制器」**：在衣服自己的层级里建 `MA Menu Item` + `Menu Installer`，再用 **`MA Menu Installer`** 把它「安装」到总菜单的对应位置。好处：衣服和它的菜单在一起，方便二次开发 / 排错。判据：Installer 选好目标菜单后，总菜单能展开出该衣服的子菜单。
- **教程回复评论的两个补丁**：
  - 「默认启用某套衣服」**点不动** → 是因为参数设了**自动配置参数值**；**取消自动配置**，手动填 0/1/2/3…，**值填 0 的那套＝默认启用**（前提是没改 `outfit_choose` 的默认值）。
  - 「`icon generator` 在哪下」：**不在他的镜像包里**，作者未答（有人问过两次）。

### 4.4 其它（EP2 评论区）

- **没有装 Modular Avatar 时，任何工具箱都不会生效**（观众的实测抱怨）。
- 「衣服太小穿不上」→ **教程答：搜 `mochifitter` 的教程**。已查证：即 **MochiFitter（もちふぃった～）**，作者 **Nine Gates**，BOOTH 商品 <https://yamirin.booth.pm/items/7657840>，**付费（约 2,
  500 日元）**；作用是**把非对应衣装自动适配到目标 avatar** —— 需要目标模型有对应的「转换 profile」（部分模型/作者会随包提供，如 Qs）。参考：
  [vr-lifemagazine 评测（2026-09）](https://vr-lifemagazine.com/mochifitter-review/)、[signyamo 用法](https://signyamo.blog/mochi_fitter/)。
- 小众模型（故障猫 ayusa）挂不上衣服 → 教程回复：手动绑骨；另一位观众补充：**换默认站姿要改 Base 层动画，直接移动骨骼录制不起作用**。

---

## 5. ★ EP3 增量：`细胳膊细腿` 的真正根因

**症状**：把素体 `… off` 形态键都调回显示了、旧衣服也删了，一进 Gesture Manager / 游戏，模型还是「细胳膊细腿 / 没骨头」。

**根因**：**素体自带的 FX 控制器在 `Entry`（进入状态）里有一段默认动画，专门把衣服对应的形态键设成防穿模值**（作者为了衣服打开时不穿模）。手调的形态键显示状态会被这段动画覆盖 —— 这就是「衣服打开时素体部件被隐藏」的来源。UP 在 EP2 置顶评论里也自认「这一期的形态键有点问题」，并在 EP3 开头说明原因。

**做法**：

1. 选中素体文件夹里的 **FX controller**（`.controller`），双击进去看状态机。
2. 找到 **`Entry`** 里那段「默认把衣服/部位打开」的动画，**把其中对应每件衣服/部件（水手服、底裤、暖腿、袜子、鞋、猫耳猫尾…）的关键帧删掉**。
3. 回场景重新进 Play 验证：四肢恢复；再补做衣服菜单。
4. 不需要的衣物条目一起清掉；内衣这类不影响的可留。

**判据**：Console 无报错；Gesture Manager 里四肢正常、衣服可切换；形态键面板数值与期望一致。

**同一期其它增量**：

- **`Object Toggle` 的开关方向**：想让某部件「默认关」，先取消它的显示，再加 Toggle —— 打开选项时才显示。
- **AudioLink 发卡的绑法**：预制件必须拖到**动骨**（Hips→Spine→Chest→Neck→Head）下，**不能拖到模型根**（模型的层级绑定在骨骼上，移动根对象骨骼不动）；绑 `Head` 后调位置 / 旋转 / 缩放。
- **上传前的红色条目**：多是「网格读写权限」类安全提示，有 `Auto Fix` 就点；**只要能构建成功、能上传就不是问题**。
- **测试期的网络纪律**：**测模型时先关加速器**（或先关 Unity 再开加速器）—— 两者同开会**抢代理**导致上传/登录异常。
- 其自建群里有个 AI 机器人，教程明确说「**半真半假，要自己明辨**」（群里 AI 的结论不能当依据）。

---

## 6. EP4 增量：git 版本管理与回档（58 分钟）

### 6.1 最小命令流（他按顺序演示）

```powershell
git init                       # 初始化（生成隐藏的 .git 文件夹；资源管理器要开「显示隐藏项目」）
git status                     # 看状态（on branch master / untracked files）
git add <文件>                  # 加入暂存区（他比喻成「购物车」）
git add *                      # 全部加入
git commit                     # 提交 → 进入 vim：按 i 输入说明 → Esc → 冒号 wq 保存退出
git commit -m 「说明」           # 命令行直接写说明（推荐）
git log                        # 看提交历史（拿到版本号；只取前几位即可）
git checkout <版本号>           # 回档到某个版本
clear / cls                    # 清屏（Git Bash 用 clear，CMD 用 cls）
```

### 6.2 ★ detached HEAD（游离态）—— 新手最容易踩的坑

- `git checkout <旧版本>` 之后处于 **detached HEAD**；**此时修改并 commit，提交不会挂在任何分支上**，切回去就「丢失」。
- **正确做法**：先 `git checkout <想回到的版本>`，再 **`git switch -c <新分支名>`** 从这个版本开一条新分支，然后在新分支上改、提交。
- 分支三件套：`git branch`（列分支）、`git checkout <分支>` / `git switch <分支>`（切换）、`git switch -c <新名>`（从当前节点分叉）。
- 他的判断：**回到旧版本只是为了「看一眼」，不要在游离态改东西**。

### 6.3 在 VRC 工程里实操（他的做法 + 两个关键细节）

1. 工程根目录右键 → `Open Git Bash here`（或在地址栏输入 `cmd`）。
2. `git status` 会列出 `Assets/ Library/ Logs/ Packages/ ProjectSettings/ UserSettings/`；他说**新手可以不做 `.gitignore`**，直接 `git add *` 全加。
   - ⚠️ **与 KB 取向不同**：本工作区（`../06-构建上传与备份/06-01-sdk-upload-and-versioning.md`；若工作区根存在 `AGENTS.md` 同级有效）建议 **`.gitignore` 排除 `Library/`、`Temp/`、`obj/`**；按他的做法会把 `Library/`（几 GB、
  可重建）也提交，仓库大得多。两种都可行：个人单机图省事 → 全加；想仓库小、克隆快 → 加 `.gitignore`（本机 `LuzzyMeow` 用的是后者）。
3. **`Permission denied` / 加不进去**：因为 **Unity 正占着文件** → **先关掉 Unity**，再 `git add *`。
4. 提交策略：**小改动小提交**（快），**阶段性功能完成时大提交**。
5. 评论区的现实提醒（观众提出、教程认同）：**git 对 Unity 工程只能「备份/归档」，没法像纯代码那样 diff**（大量 png/fbx 二进制），所以**每次提交仍要写清更新说明**；UP 补充他的用法是 **git + Claude Code 辅助备份**。
6. 评论区小坑：**记得开「文件扩展名」显示**，否则会出现「写了 .txt 但系统没加扩展名」导致 git 找不到该文件的情况。

---

## 7. EP6 增量：妆容贴图与瞳色替换菜单

### 7.1 换妆贴图（比 KB 更具体）

- 素材形态：作者常给 **PSD + PNG**（本集只做 PNG；PSD 要改图层）。
- 位置：选中 **`Body`（脸）** 或 **`Body_Base`（身体）** 的 `Skinned Mesh Renderer` → **材质 → 纹理设置**看贴图；右键 **Open in Explorer** 能看到素体自带贴图（眼、鼻、耳、舌…）。
- 替换分三层：**主色（Main Color）/ 第二主色 / 第三主色**：
  - 妆容（腮红、眼妆）通常贴 **第二主色**（在第一主色之上，可叠加）。
  - 整体身体贴图 / 肤色放 **主色**（整体替换）。
  - 瞳孔这类「只改一处」的放 **第三主色**，再用菜单替换材质球（见 §7.2）。
- 小技巧：**锁住 Inspector 右上角的小锁**（锁定选择），挑素材时界面不会乱跳。

### 7.2 瞳色切换菜单（本集核心配方）

1. `Extract Menu` 提取素体菜单 → 新建 `GameObject` → 加 `MA Menu Item` 改成 **Sub Menu**（例：`瞳色切换`）。
2. 找到**脸部材质球**（素体 `Material/…_face`）→ **Ctrl+C / Ctrl+V 复制 8 份**（做几个瞳色复制几份），改名 `i1`…`i8`。
3. 每个副本：**打开第三主色**，替换成对应瞳孔贴图，`Ctrl+S` 存资产。
4. 菜单项里加 **`MA Material Swap`**：`Object` 选**原素体脸部材质球**，`Material` 选做好的第 N 号材质球。
5. **去掉「诡异眼仁」**：换瞳孔贴图后中间会露出原来的眼珠 → 同一菜单项上再加 **`MA Shape Changer`**，把 `Body` 的形态键 **`iPupilX`**（控制眼仁前后缩放）设成 `100` 或 `delete`。**很多教程没讲这一步**，它是「换了瞳色眼睛很吓人」的原因。
6. **互斥**：素体 `Parameters` 里新建 **int** 参数（例：`瞳色设置`，默认 0），每个菜单项参数值填 1/2/3…（**并取消「自动配置参数值」**），同一时间只生效一套。
7. 验收：`Gesture Manager` 逐个点 —— 眼睛有星星、无多余眼仁、身体妆容正常，`Ctrl+S` 保存。

### 7.3 其它

- 「默认表情」：`Animation → Facial` 里勾选**更改默认表情**可换默认脸（带眨眼动画的表情不适合当默认）。
- 评论区：有人问「瞳孔眼仁在形态键里找不到」→ **取决于素体作者有没有做 `iPupilX`**，没有就得走别的途径（教程未给通用解）；也有人问 PCSS 光影（未答，留作后续插件篇）。

---

## 8. EP7 增量：FaceEmo 自定义手势表情

### 8.1 装与初始化

1. 项目里加：lilToon、**Modular Avatar**、**Gesture Manager**、**FaceEmo**（+ 贴图优化工具）。
2. 导入模型 → `Extract Menu` → 把 FaceEmo 的 prefab **按住拖进角色层级**（双击只会进预览、等于没加）→ 启动 FaceEmo。
3. **新增表情组**：右键 FaceEmo → `New Menu`；在层级里建一个新对象（例：`表情组01`）。

### 8.2 手势 ↔ 表情映射（他演示的 VR 手柄 7 种）

| 手势 | 手柄操作 | 常见用途 |
|---|---|---|
| `Fist` | 上扳机 + 下扳机同时按住 | 攥拳 → 生气 |
| `HandGun` | 只按**下扳机** | 比枪 |
| `Open` | 什么都不按 / 手掌打开 | 默认表情 |
| `Peace` | 摇杆**轻轻前推** | 比耶 |
| `Point` | **下扳机 + 摇杆前推** | 指东西（放传送门 / 吸引注意） |
| `Rock` | **上扳机 + 摇杆前推** | 摇滚手势（难推） |
| `ThumbsUp` | 上下扳机同时按住 | 点赞 / 握拳（教程口播前后不一致，以 FaceEmo 面板为准） |

（左手 / 右手 / 左右交换都能绑；默认只做**右手**。）

### 8.3 捏表情

- FaceEmo 里选中手势项 → 拖入一个「基础表情预设」（`Animation → Facial` 里自带的），再按小笔编辑。
- 捏脸＝调 **BlendShapes**：眼睛（`Smile` / `Angry` / `Sad` / `Surprised` / `Blink` / `Wink`、各种高光与瞳孔形状）、嘴巴（嘴型、歪嘴、倒三角猫嘴…）；**幅度可调**（他一般 100%）。
- 做好后点 `应用到角色`（会问是否禁用原表情前缀以缩短构建时间，可保持默认）；它生成一个 **FaceEmo prefab**，把菜单挂到你的菜单下即可。

### 8.4 ★ 眨眼动画会覆盖表情（他踩的坑）

- 现象：设了表情后眼睛还是不自然眨眼，或表情里的眨眼不生效。
- 根因：**素体自带的 `Blink` 眨眼动画在 FX/Expression 层与表情抢同一个形态键**。
- 处理：在该项设置里**禁用「眨眼」**（或到 `Expression → FX` 的眨眼动画里处理）；他最后是「把这套表情的眨眼禁用掉」→ Play 里验证不再被覆盖。
- 评论区同类反馈：**「用完 FaceEmo 后原模型表情用不了」**、**「带 FaceEmo 进 MMD 房跳舞，头发动骨直接没了」**（教程未给结论；建议改完先在 Gesture Manager 全量回归再上传）。

---

## 9. 你问我答 增量：两个插件（KB 之前 0 收录）

### 9.1 RK Player 随身听 / CD 播放器

前置：**mp3 / ogg / wav** 音乐 + **1:1 封面图**（mp3 可提取内嵌封面）+ RK Player 的 unitypackage。

1. 导入包 → 找到 `RK Player` prefab → 拖到角色层级下 → 调 `Scale`（他给 0.8）与位置角度（Q/E 升降）。
2. **封面**：`SN → CD cover` 已预制几个材质球；把封面 PNG 拖进工程，在材质球的**主色纹理**里替换（`Ctrl+S` 存素材）。
3. **音乐**：音频拖进工程 → 每个音频在 Inspector 里勾 **`Load In Background`**（后台加载、不阻塞主线程），**质量拉到 1**，`Apply`。
4. **歌单**：双击 `SN list` → 点 `Play` → 音频依次拖进列表（顺序＝播放顺序）。
5. **改名**：`RK Player → RK Player Menu` 里把 `SN select` 各项改成歌名（F2 重命名对象后替换菜单文本）；封面同样可拖到该项图标位。
6. **菜单归属**：`RK Player Menu` 的 **`MA Menu Installer`** 可把它安装到任意 sub menu（例：自建 `插件` 菜单）。
7. 验收：进 **Game 视图**（场景视图听不到声音）；可把 Game 视图拖出来单独摆相机对准 CD 机；Play 里切歌、看 CD 弹出动画。

**评论区两个坑（重要）**：
- **必须用「原版 RK Player」prefab**：有人复制了一个已调好位置的，结果 **Unity 和游戏里都没声音**；解决法是「留一个调好的在资产夹当参考，用的时候重新拖原版，再把位置/缩放抄过去」。
- **默认关闭 / 皮带残留**：总开关只关播放器本体时肩带还在；要彻底关得先切「手持」再关总开关，且下次进世界皮带会回来 —— 这类问题要看**动画层参数**（他提到 belt 参数是 0 但仍复现）。

### 9.2 AudioLink 名牌发卡

1. **它依赖 AudioLink**：项目里要加 **AudioLink / VRCAudioLink** 包，否则没有语音响应（KB 未收录）。
2. 发卡 prefab **发紫**：多半是缺作者指定的 **Poiyomi**（他演示用的是 Poiyomi 卡通渲染），去其发布页 / `Add to VCC` 链接加，或下载 unitypackage 双击导入 —— **预制件发紫＝着色器没装**。
3. 绑定：把 prefab 拖到**动骨 → `Head`** 下，调位置 / 旋转 / 缩放（注意 **三轴一起缩放**，只缩一个轴会变扁）。
4. **自定义名字**：`Player` 那个字是一张 **PSD/贴图**（引用自素体 `Texture`）→ 用 Photoshop 改；作者原本做了在线生成页但**兼容性没搞定**（视频里当场说明改为「进群找作者帮做」）。
5. **自定义头像**：把图片拖进工程 → 在材质里替换对应贴图。
6. **最后一步别忘**：**确认 AudioLink 已经加进项目**，否则挂上去没有任何语音互动。
7. 评论区：有人问「加了 AudioLink 开麦也没互动」—— 教程未答（可先按 KB 的驱动链排查：AudioLink 组件是否在场景、Avatar 是否带 `VRCAudioLink`、贴图是否指向 AudioLink 主题贴图）。

---

## 10. 地图制作 EP1：Blender 入门

- 作者参考 **spooky（Spooky Ghost School / Black Cat 世界作者）** 的地图教程体系；本册只记这一集的软件操作部分（与改模的网格编辑同源）。
- Blender 下载：`blender.org` → Download（免费开源）；他说**在 Unity 里建模「比较难受」**，做地图仍推荐 Blender（ProBuilder 能做，但细致模型不行）。
- 汉化：`Edit → Preferences → Interface → Translation → Language → 简体中文`，**需要重启 Blender 生效**（与本机 2026-10-17 装语言包的结论一致）。
- 基础操作：滚轮缩放；**中键拖动**旋转视角（`Shift+中键`＝平移画布）；小键盘 `.` 把视角聚焦到选中对象；`S` 缩放、`G` 移动、`G+X/Y/Z` 或 `S+X/Y/Z` 锁单轴；`X` 删除；`Shift+A` 新建；`Ctrl+Z` 撤销；`Ctrl+S` 保存。
- **飞行 / 行走模式**（做地图看效果）：`Shift+波浪键` 进飞行 → WASD 移动、E 上升、Q 下降、Shift 加速 → `Tab` 切行走（重力）→ `V` 跳跃、`Tab` 回飞行 → `Esc` / 右键退出。
- **编辑模式**：`Tab` 切物体 / 编辑模式，`1/2/3` 切点 / 线 / 面；选中面按 `E`＝**挤出**、`I`＝**内插（insert）**（他现场做了个小花盆）。
- **游标（Cursor）**：`Shift+右键` 定位游标 → 之后新建的物体**在游标处生成**；让游标回世界原点用 **`Shift+C`**（Center Cursor and View All，光标回原点并框选全部；
  也可 `Shift+S → Cursor to World Origin`）。**教程口播的「Ctrl+C」是口误**，已按 Blender 手册 3D Cursor 页核实。

---

## 11. 未验证清单与复核点

- 本册**没有任何步骤在本机复现过**：全部来自外部教程与评论，属「外部教程笔记」。用之前按 KB 铁律 1 先做小样验证。
- 需要复核：① EP3「删 FX 层 `Entry` 关键帧」在别的素体上是否同样安全（不同素体实现不同）；② `mochifitter` 的确切包名与来源；③ EP2/EP7 提到的「贴图优化工具」名字未听清；④ 地图 EP1 游标归位快捷键口播疑为口误；
  ⑤ 镜像源 URL 的可用性与授权范围（第三方自建，注意「预发布」勾选与来源可信度）；⑥ FaceEmo / AudioLink / RK Player / Poiyomi 的**本机包名与版本**（本工作区尚未安装实测）。
- 与 KB 一致、不重复收录的点：MA 优先（铁律 4）、Gesture Manager 必装（铁律 5）、导入后完全解压预制件、`.unitypackage` 用 `AssetDatabase.ImportPackage` 导入、上传必须 GUI、MA 只在 Bake/Play/上传生效（铁律 1）。

（本册由 Agent 于 2026-10-18 整理；来源清单见 `ACKNOWLEDGEMENTS.md`。）
