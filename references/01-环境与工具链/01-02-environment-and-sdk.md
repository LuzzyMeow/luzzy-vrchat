# 环境、工具链与 SDK 上传（Unity 基础）

> 定位：**中文从零教学链路**。本册只记录基底 `../00-基底/` 与其它分册未覆盖的**实操流程与界面路径**，
> 原理性内容（NDMF/MA 所有权、证据阶梯、审计方法）仍以前述分册为准，不在此重复。
> **版本提示**：以下界面文案随工具版本变化，按「记录日期 → 复核时间」标注；图文版优先于视频（视频成型后不易修改），
> 老版教程会逐步下架（老版有「能用但讲错」之处）。核对时以站点当期内容为准。

## §0 与基底的分工

| 本分册负责 | 交给既有分册 |
| --- | --- |
| 工具链安装顺序、镜像源、项目创建、SDK 面板逐字段、上传实操、Unity 基础操作 | 所有权与授权边界 → `07-02-evidence-and-ownership.md` |
| 中文生态的常见坑（中文路径/用户名、Unity 中国版、加速器） | 参数成本与菜单成本模型 → `03-01-parameter-and-menu-audit.md` |
| MA 快捷菜单工作流的**点击路径** | MA 组件语义与烘焙语义 → `07-01-bake-and-validation.md`、`07-03-provider-playbooks.md` |
| 粒子/贴图/材质的中文教学参数 | lilToon 属性真伪判读 → `04-01-appearance-and-liltoon.md` |

## §1 安装链路（-01RE「重置安装教程」＋图文版「工具」板块）

**起因**：作者明说 Unity 中国相关组件导致许多人卡在安装阶段。**前置检查**：

1. 文件资源管理器 → C 盘 → 用户文件夹，确认**用户文件夹名只含英文或数字**；
2. Windows 设置 → 系统 → 详细信息，确认**设备名称也只含英文或数字**；
   含中文/特殊字符的先重命名并**重启电脑**再继续。

**配套文件页**：<https://start.gamemode1.com/pages/download/> 「资源收录」——**必备软件**（ALCOM / Unity Hub / Unity Editor，**推荐版本 2022.3.22f1**）与**课件**（02 史蒂夫模型、03 钻石剑模型、06 动作测试包，共 3 个 zip）。
所有下载都走站点自己的 redirect 跳转，**链接会变，别把跳转地址抄进文档**；找不到就用视频标题里的序号在页面上对照。

三个安装包：① Unity Hub **国际版**（不要中国版）② VRChat 对应版本的 Unity **离线版** ③ ALCOM。
开始前先在文件资源管理器「查看」里勾上**显示文件扩展名**（作者称对后续使用有巨大作用）。

**Unity Hub**：

- 安装时**取消勾选「运行」**，先不要启动；
- 桌面快捷方式右键 →「打开文件所在位置」→ 进 `resources` 文件夹 → **删除 `app-update.yml`**（阻止自动更新）→ 再启动；
- `Sign in` 登录 Unity 账号 → 首次打开**关闭欢迎页、跳过 Hub 推荐的版本** → 证书弹窗选 **`Disagree`**（稍后手动获取证书）；
- 左侧 `Settings`：`Appearance → Language` 改中文；「项目」页改项目位置（**路径不含中文**）；「安装量」页改**安装位置与下载位置**（安装位置不含中文）；「许可证」右上角添加 → **获取免费的个人版许可证**（作者称旧安装方式已失效，现在要手动获取）。

**离线安装编辑器**：运行 Unity Editor 安装包 → 选安装位置 → 装完回 Unity Hub →「安装」列表右上角**选择位置** → 指向刚装的 `Unity.exe`；列表出现版本号即成功。

> 本工作区现状与上述一致：编辑器已装 `<VRCHAT_ROOT>\tools\Unity\2022.3.22f1\Editor\Unity.exe`，
> Hub 在 `<VRCHAT_ROOT>\tools\UnityHub\Unity Hub.exe`，但 `editors-v2.json` 仍为空，需在 Hub GUI 里
> Add/Locate 一次（或用 ALCOM/VCC 直接指向编辑器路径，不经 Hub 亦可）。

**ALCOM**：选安装位置 → 运行并创建桌面快捷方式 → 首次启动欢迎页（**空白说明系统浏览器组件有问题**）→
继续（会自动选 Unity Hub 路径）→ 设置**项目保存位置与备份位置**（都不要含中文 / 特殊字符 / 空格）→ 其余默认。

## §2 国内镜像源替换（-01RE）

ALCOM → 资源列表 → **取消勾选「官方」与「精选」存储库** → 打开工具站下方的镜像站
（两条线路，分别由 **VRCD 的大佬们** 与 **星川真纪** 维护）→ 先导 **`curated`**（VRChat 提供）复制
右侧的镜像仓库地址 → ALCOM 右上角「添加存储库」粘贴 → 再同样导入 **`official`** → 之后用搜索功能
添加 **`VRChat SDK Patcher`、`liltoon`、`Modular Avatar`**，共 **5 个仓库**。

站点另给的两个推荐源：**VRCD**（VCC/ALCOM 通用、更新维护及时、仓库更全）与
**VRCLearn 自家 `https://vpm.gamemode1.top/`**（ALCOM 优化、**仅支持国内访问**、仅提供有限仓库）。

**仓库下载排障**（站点原文 4 条）：① 试手机流量热点；② 换不同镜像；③ 参考 §4 手动装包；
④ VRCD 镜像问题要到**镜像站右上角「文档 - 联系我们」进 VRCD 群**反馈，在主站反馈没用。

> **装完第三方包后的编译等待**：shader 会在后台编译，Unity 可能卡约 1 分钟 → **不要强关**。

## §3 新建项目（-01RE / -06 / 图文版）

1. ALCOM 右上角**新建项目** → 模板 `VRChat 虚拟形象`（世界用世界模板）→ Unity 版本选 VRChat 所需版本；
2. **创建过程中的报错是正常现象**；创建完应有**两个已装（高亮）包**：`VRChat SDK Avatars` 与 `VRChat SDK Base`（灰色 = 没装上，点右侧加号手动加）；
3. 再手动加 `Avatars 3.0 Manager`、`Gesture Manager`、`liltoon`、`Modular Avatar`（**装 MA 会连带新增 NDMF 软件包，一并安装**）；
4. 点上方「安装所选软件包」→ 右上角打开 Unity；
5. **加载期间出现防火墙提示要允许访问专用网络** → **等右下角加载图标转完再操作**（否则易「软件未响应」）；
6. 关掉 VRChat 欢迎界面 → 顶部菜单 `VRChat SDK → Show Control Panel` 能打开 = 项目可用。

**项目命名与路径**：名称与路径**不能含中文**（作者同时反复强调不能含空格）。
ALCOM 里点项目右侧「三个点」→「打开项目路径」→ 回上一级**改文件夹名 = 改项目名**；重启 ALCOM 后原项目变灰 →
点「新建项目」旁按钮选**「添加已有项目」**。导入别人给的完整工程同理。

## §4 完全手动装包（镜像下载失败时的备用路线）

镜像站下不动时走这条路（已同步进图文版「创建项目」章节）：

1. **从 GitHub 下载软件包**：统一取 **`Latest` 版本 + `Zip` 格式（哪怕它同时提供 unitypackage）**。
   VRChat SDK 的 **Base 必下**、**Avatars 用于角色**、**Worlds 用于世界**；另下 `Avatar 3.0 Manager`、
   `GestureManager`、`liltoon`、`Modular Avatar` **及其前置**。
2. **Unity Hub → 新建项目** → 选对编辑器版本 → 核心模板选 **`3D`** → 改项目名 →
   **一定选「新建本地项目」** → 选存放位置 → **不要勾 `Use Unity Version Control`** → 创建。
3. 把下载的所有压缩包**以「原压缩包名称」建同名文件夹解压**；回 Unity 在 `Packages` 文件夹右键
   `Show in Explorer` → 进 `Packages` → 把解压出的文件夹**移进 `Packages` 下**
   （= 手工做 **embedded 包**，与 `06-02-backup-and-package.md` §2.1 的 embedded 语义一致）。
4. 回 Unity 会进入加载状态，**期间不要做其他操作以免加载失败**；加载完**黄色报错不用管、直接清掉**；
   顶部能打开 VRChat SDK 控制面板即成功。

> **作者的另一条声明**：不要再去看他的**老版教程**（有很多「虽然能用但讲错」的地方），
> 老视频会**慢慢下架**只留新的。引用/学习时优先新教程与图文版。

## §5 SDK 控制面板逐字段实操（-05「认识 SDK 控制面板」＋ -02/-03 上传片段）

面板四个标签页：**Authentication（账号） / Builder（构建） / Content Manager（内容管理） / Settings（设置）**。

### §5.1 Authentication

- **不能用 Steam 账号直接登录**，必须 VRChat 账号；已用 Steam 登录的要先去**官网绑定**；
  **绑定前先备份好友名单——绑定后好友可能被全删**。
- 登录后显示游戏名与两个权限：`World Creator Status` / `Avatar Creator Status`；`Logout` 登出。
- 账号**等级至少 New User**，面板要显示 **`Allowed to publish avatars`** 才能上传。

### §5.2 Builder → `Prepare Your Content`

| 字段 | 实操要点（作者口径） |
| --- | --- |
| `Selected Avatar` | 只有「一个场景里放了多个模型」时才需要手动指定 |
| `Name` | 模型显示名 |
| `Visibility` | 默认 **Private**；**非原创模型设 Public 有下架 / 封号风险** |
| `Primary / Secondary Style` | 风格标签 |
| `Content Warning` | **设错也可能封号**，按官方文档选，不确定就不加 |
| `Text` 标签 | 文本标签 |
| `Description` | 留空则自动填模型昵称 |
| 封面 | `Selected Image` **仅接受 PNG**；**推荐 1200×900**，不符合会被自动裁剪；或 `Capture in Scene` 用当前 Scene 视角截图 |

### §5.3 Builder → `Review Any Alerts`

**只有带「八边形红色感叹号」的报错才能点 `Auto Fix`，其它一律忽略。**（与 `06-01-sdk-upload-and-versioning.md`
§6 的告警三级分类配合使用：能点 Auto Fix 的属于「必须先修」，其余先分级再决定。）

### §5.4 Builder → `Build`

- `Build Type` 两种：`Build and Publish Your Avatar Online`（默认）/ `Build and Test Your Avatar`（只本地构建）。
- **上传失败的排查法：换 `Build and Test`——若它能成功，说明失败原因是网络环境而不是工程。**
- `Platforms` 暂时只用 `Windows`。
- 出现 **`Upload Succeeded`** 即上传成功。

### §5.5 Content Manager

`Fetch` 刷新列表；`Make Public`（**谨慎点**）；`Copy ID` 复制蓝图 ID（覆盖更新时用）；`Delete`；`Open on Web`。

### §5.6 网络：加速器用法（-02）

上传前需要**带「路由模式」的游戏加速器**、**选最近节点**、**用路由模式**；
**上传完成后关掉加速器的路由模式**。

### §5.7 覆盖更新已有模型（-03）

SDK 面板 → Content Manager → `Avatars` 列表找到原模型 → **`Copy ID`** → 回到场景选中要上传的模型 →
Inspector 的 **`Pipeline Manager`** → 把 ID 粘进 `Blueprint ID` → 点 **`Attach`**。
（本册 `06-01-sdk-upload-and-versioning.md` §5 的检查顺序仍是权威流程：先克隆、清空、保存、重载、再读、最后开面板。）
同页还有 `Delete`。作者顺带提醒：请人代做要及时改密码、找靠谱卖家。

## §6 Unity 基础操作与工程组织（-02 / -03 / -04）

### §6.1 四个窗口

- `Project` = 文件管理器；新建文件夹/资产的入口（右键 → `Create`）；
- `Hierarchy` = 当前场景里的对象树；
- `Inspector` = 选中对象的组件；
- `Scene` = 3D 视图：**按住鼠标右键 + WASD 飞行**；选中对象会显示轮廓；在 Hierarchy 里**双击对象可自动聚焦**。

**新建场景**：Project 空白处右键 → `Create → Scene`；双击打开。
**恢复默认层**：Unity 右上角 `Layers` 旁的下拉 → 选 `Default`。
**杂项**：取消勾选 Skybox、右上角关掉 Gizmos（或只取消 `3D Icons`）可让视口图标变小、场景更干净。

### §6.2 父子级（-03）

- **铁律：子级随父级变化而变化。** 这是改模的核心知识之一——挂衣服、挂饰品、部分插件全靠它。
- 给史蒂夫挂钻石剑：展开 `Armature` 找 `Right Hand`（模型没有手就绑 `Right Wrist`）；
  **缩放前先打开 Scale 旁边的锁链（等比缩放）**，否则模型外观会被破坏。
- 作者原话：**你自己永远是项目的最终父级**（即所有判断以人为主，工具只给建议）。
- 清理无用组件：组件右侧三点 → `Remove Component`（示例删掉 Box Collider、Rigidbody 与两个失效组件）。

### §6.3 模型偏暗 / 变紫（-03 / -06）

- **模型发暗** = 使用了 Unity 默认 shader → 在材质组件里把 `Shader` 下拉改成 `lilToon`。
- **变紫 = 材质或着色器不对**：回物品详情页核对要求。示例物品要求 **UTS2** 而项目里只装了 lilToon →
  要么补装对应 shader，要么换满足要求的物品。
- **作者建议：改模初期统一使用同一种着色器**，能少掉一整类排查工作。

### §6.4 预制件 Prefab（-04）

- 把 Hierarchy 里的对象**拖到 Project 窗口**即成 Prefab（Hierarchy 里名字与图标变蓝）。
- **双击 Project 里的 Prefab 进入「预制件模式」（场景变蓝）**；从 Hierarchy 左上角返回；
  也可在 Hierarchy 里点对象右侧的箭头进入（进入后场景会变回白色 = 普通编辑态）。
- 在预制件模式里改颜色 → **所有场景同步**。
- **在具体场景里单独改 Prefab 不会影响其它场景**，且预制件名旁会出现一条**蓝色竖线**表示被覆盖。
- Inspector 的 `Prefab → Overrides`：`Revert All` 撤销该实例的全部覆盖；`Apply All` 把覆盖写回所有实例。
- 往 Prefab 里**加**东西可以；**往外拿会弹提示** → 右键 `Prefab → Unpack Completely` **完全解压**
  （解除关联、变成独立对象，名字恢复白色、蓝图标消失）；解压后仍可重新做 Prefab。

### §6.5 目录与「分类空对象」（-06）

- Project 窗口里分类建文件夹：**动画**、**菜单**、**粒子**、**材质**；**路径不含中文**。
- 在角色上右键（**别点到骨骼上**）`Create Empty` 建空对象当分类（如「头发」「衣服」「饰品」），
  名字里**加特殊符号让它显眼**，并点左侧的**眼睛与手指图标锁定**防止误选。
- 物品与空对象的关系：**先建空对象再往里放物品**；场景变复杂后**一律从 Hierarchy 里选对象，别在 Scene 里点**。

## §7 导入模型、选型与摆放（-02 / -06）

- 模型来自「交流群或文档里的官网」下载，得到 **unitypackage**；导入：Unity 菜单 `Assets → Import Package → Custom Package` → **勾 `All`** → `Import`。
- 作者给的对照表：**SteveV1** = 关节不可动（原版 MC）；**SteveV2** = 加了动作模组。
- 商品通常提供**全套版**与**素体版**两种预制件：**新手用全套版**（自带原版衣物，便于对比），
  **改模大佬用素体版**（省性能，但要自己补原版衣物）。
- **摆放**：优先把模型**拖进 Hierarchy**（按预制件设定好的位置摆好）；拖进 `Scene` 则要**手动归零**位置。
- 场景里有 `Directional Light` 时，**把光的颜色改成白色以防色差**（颜色会直接影响你对贴图/材质的判断）。

## §8 贴图导入参数与调色（-02 / -11）

### §8.1 贴图导入设置（示例值）

```text
Alpha Is Transparency  ✅ 勾选（贴图带透明通道时）
Wrap Mode              Clamp
Filter Mode            Point
Max Size               512
Compression            None
```

改完点 `Apply`。**这些是教材里针对像素风/贴图清晰度的示例值**：实际取值看用途——
需要锐利像素边缘才用 `Point` + `Compression None`，写实贴图往往需要更大的 `Max Size` 与压缩。

### §8.2 换皮肤 = 换材质贴图（-02）

选中 `Body`（或对应网格对象）→ Inspector 展开材质 → 把皮肤图**拖到主色（`_MainTex`）旁的
小方块（贴图槽）**上即可。

### §8.3 用曲线调色挑染

- 在 Photoshop 里对贴图用 **`图像 → 调整 → 曲线`**：先调 **RGB 综合通道**改整体亮度，
  再分别调**红/绿/蓝单通道**改色调；需要整体换色时配合`色相/饱和度`。**保存后 Unity 会立刻刷新预览**。
- 在 Unity 侧直接改色相：点材质的 `Color` 小方块 → 拾色器调 `H/S/V` 与 `Alpha`。
  **注意这与 `_MainTexHSVG`（色相/饱和度/明度/Gamma）不是同一回事**，后者是 lilToon 的贴图级调色，
  真伪判读见 `04-01-appearance-and-liltoon.md`。
- 头发类材质还可调 `Smoothness`（光泽/高光强度）—— **但该属性是否真的生效取决于所用 shader**，
  必须按 `04-01-appearance-and-liltoon.md` 的「属性真伪判读」用实际渲染结果验证。

> **未覆盖项**：眼睛贴图专用内容未纳入本册；如需要精确步骤，请查对应图文版章节。
>
> **证据边界**：正文依据为教程字幕与图文版页面（部分深链页面抓取被拒，只能从首页进入）。
> **引用时请按此边界标注可信度**，不要把没读到的内容当成已核实。

## §9 绑定骨骼：手动与 MA 两种做法

### §9.1 骨骼命名与结构

层级 = 简易人体骨骼英文名：`Armature → Hips → Spine → Chest → Neck → Head`，手部沿 `Shoulder → UpperArm →
LowerArm → Hand`（作者示例里的手腕名是 `Right Wrist`）。物品自己的 `Armature` 是**独立骨架**，
不与角色共享——所以才有「绑」这一步。

### §9.2 古法（父子级）绑骨

1. 把物品预制件 `Unpack Completely` **完全解压**；
2. 展开物品的 `Armature` 找到对应骨骼（头发 → `Head`）；
3. 展开人物：`Armature → Hips → Spine → Chest → Neck → Head`；
4. **把物品的 `Head` 拖到人物的 `Head` 之下**（其余骨骼保持原位）；
5. 进 Gesture Manager 跑动画测试是否跟随。
   衣物骨骼结构与人物几乎一致：**把衣物里的骨骼逐一挪到人物同名的骨骼下，从上往下、从 Hips 开始逐层往里挪，要细心。**

### §9.3 插件绑骨（MA Merge Armature）

1. **先备份场景**：Project 里选场景 `Ctrl+D` 复制一份留作插件绑骨用（古法那套做在另一份上）；
2. 回到备份场景，展开头发与衣服，`Ctrl` 同时选中**两个 `Armature`**（物品的与角色的）；
3. Inspector `Add component` → 输入 `ma` → 选名字里含 Armature 的 **`Merge Armature`**；
4. 选中**物品**的 `Armature`，把**人物**的 `Armature` 拖进 `Merge Target`（合并目标）；衣服同理；
5. **命名有前后缀的模型必须再点一次组件里的「调整骨骼名称」按钮**，否则会匹配错；
6. 若头发不按预期运动，就是 `Merge Armature` 把它绑到了错误位置——**它按「同名」匹配，命名不一致必错**。

> 与基底 `02-01-workspace-and-work-assets.md` §3 的差异点值得留意：**该课程把 `Merge Armature` 直接加在物品的
> `Armature` 上，而基底给的优先做法是「无 `MergeArmature`（MA 1.13+ 改名）时优先在衣服骨架根加组件」**。
> 两者不冲突：课程用的是**旧名组件 `MergeArmature`（1.13 前）**，MA 1.13+ 已重命名。
> 实际组件名以项目里 MA 版本为准，**不要照着旧教程找不存在的菜单项**。

### §9.4 跑测试（Gesture Manager，-06）

1. 导入作者给的**动作测试包**（走路 / 奔跑 / 躺下）；
2. 顶部菜单 `tools → Gesture Manager emulator` → 场景里生成 Gesture Manager 对象；
3. Inspector 里点 **`enter play mode`**（每次自动进入 `Game` 窗口，手动切回 `Scene` 继续调）；
4. `tools → extra tools → test animation` → 把动作（如 `Run`）拖进输入框 → 点右侧 `play` 看角色跑；
   `stop` 停止；**正上方的播放按钮退出运行状态**。
5. **头发/衣服不跟着跑 = 骨骼未绑定**（回 §9.2 / §9.3）。

## §10 本节验收清单

- [ ] 用户文件夹名、设备名、项目路径均不含中文/空格/特殊字符；
- [ ] Unity 是国际版 Hub + 对应版本编辑器；Hub 语言/项目位置/安装位置/许可证均已设置；
- [ ] ALCOM 已换成镜像源（`curated` + `official` + 3 个包仓库）；
- [ ] 项目里 `VRChat SDK Avatars/Base` 高亮已装，`Avatars 3.0 Manager`、`Gesture Manager`、`liltoon`、`Modular Avatar`（连带 NDMF）已装；
- [ ] 编辑器加载完成、防火墙已允许专用网络、`VRChat SDK → Show Control Panel` 能打开；
- [ ] 登录的是 **VRChat 账号**（非 Steam 直登），面板显示 `Allowed to publish avatars`；
- [ ] `Review Any Alerts` 里只有八边形红色感叹号被 `Auto Fix`；其它告警已按 `06-01-sdk-upload-and-versioning.md` §6 分级；
- [ ] 上传前开加速器路由模式、选最近节点；上传后关闭路由模式；
- [ ] 覆盖更新时 `Blueprint ID` 来自 Content Manager 的 `Copy ID` 并通过 `Pipeline Manager → Attach` 生效。

## §11 证据与边界提醒（务必一起读）

- 本分册所有步骤**都是「静态来源 + Unity 已解析」层面**的实操；**「上传成功」「别人能看见」属于
  `UPLOAD_CONFIRMED` / `CLIENT_RUNTIME` 层，只有用户自己进游戏验证才算**（见 `07-02-evidence-and-ownership.md` §1）。
- **上传/公开是用户决定**：`Visibility`、`Content Warning`、`Make Public` 涉及合规与账号风险，
  Agent 只能陈述后果，**不能替用户拍板**（见 `07-02-evidence-and-ownership.md` §2 授权矩阵）。
- 教程口径与工具现值有偏差时（如组件改名、Hub 界面改版），**以项目里实际的组件/菜单为准**，
  并在报告里用 `STALE` 标注来源（见 `07-02-evidence-and-ownership.md` §1.2）。

---

**相关分册**：环境搭建 → `01-01-environment-and-toolchain.md`；SDK 上传与版本 → `06-01-sdk-upload-and-versioning.md`；
工作目录与编辑器脚本 → `02-01-workspace-and-work-assets.md`；动画/菜单/参数 → `03-01-parameter-and-menu-audit.md`、
`07-01-bake-and-validation.md`；绑定与穿模 → `05-01-outfit-fit-and-clipping.md`。
