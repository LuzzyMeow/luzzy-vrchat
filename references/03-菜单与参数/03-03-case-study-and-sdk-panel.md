# 从零改模实战：导入、绑骨、菜单与外观

本册按教程原顺序把两个案例串成可复现流程；只写基础分册未覆盖的实操细节。图文版与视频不一致时**以图文版为准**（文档声明）；文档环境 = Windows 10 + Unity 2022.3.22f1 + Unity 英文界面。

## §0 与其它分册的分工

| 分册 | 内容 |
|---|---|
| 01-02 | 环境、工具链、SDK 面板、Unity 基础**知识点**、绑骨知识点 |
| 03-02 | 换装菜单两条路线（手搓 Animator / MA 快捷菜单）、粒子特效 |
| **本册** | **按教程原顺序把两个案例串成可复现流程**，只写 01-02/03-02 未覆盖的实操细节；重复处标注指向 |

⚠️ 本册是**教学案例**，不是通用最佳实践（教程明确说明「讲的不是专业 Unity 知识」）。照抄参数前先用基底 `SKILL.md` 三条铁律与 `../07-验证与排查/07-02-evidence-and-ownership.md` 核对：**MA 组件只在 Bake/Play/上传生效，场景视图里的穿模多半是假象**。

---

## §1 史蒂夫入门链路（lets-minecraft 1-4 + diamond-sword）

### §1.1 项目更名 / 接手别人的工程

1. 在 ALCOM 或 Unity Hub 里点项目右侧 `···` → **打开项目路径**（Unity Hub 里叫「在资源管理器中显示」）。
2. 回到项目**上一级**文件夹，直接重命名项目文件夹（例：`Avatar_ALCOM` → `Avatar_Steve`）。
3. **重启 ALCOM / Unity Hub**，项目会消失（路径已变）。
4. 点「新建项目」旁的 **添加已有项目**（Unity Hub 里是「添加 → 从磁盘添加项目」），选中改名后的文件夹。

> 接手别人的工程 = 从第 1 步直接跳到第 4 步。基底 `../02-工程组织与工作流/02-01-workspace-and-work-assets.md` 的「先确认基础头像 prefab」在此同样适用。

### §1.2 导入 unitypackage

`Assets → Import Package → Custom Package...` → 选 `.unitypackage` → 弹窗点 **All**（默认全选）→ **Import**。导入后 Project 里出现对应文件夹（本例 `gamemode1`）。

### §1.3 模型全紫 = 着色器缺失（不是模型坏了）

- 紫黑方块 = 缺少模型需要的着色器（本项目用 **liltoon**），与 Minecraft 材质包报错同理。
- 装法：ALCOM 里加 liltoon 镜像仓库 → 回「项目 → 管理」界面找到 liltoon → 添加。
- **加完回 Unity 可能卡 1 分钟左右，不要强制关闭**。
- 场景里图标挡视野：关掉 Scene 的 **Gizmos** 开关，或在下拉里关 **3D Icons**。

### §1.4 加装饰物（钻石剑）

1. 把道具放进角色骨架的对应骨骼**子级**下。本例：放 **`Right Arm`**（手腕）下——作者注明「图示已过时，经测试放 `Right Arm` 比 `Right Wrist` 更合适」。
2. 调 Position / Rotation / Scale；**调 Scale 前先把 Scale 旁的锁链图标点成激活**（等比缩放）。
3. 删除道具上的失效脚本：`Box Collider` / `Rigidbody` / `(Script)`（可能不止一个）。
4. 道具发暗：把材质 `Shader` 从 `Standard` 改成 `liltoon`（除非喜欢原样）。
5. 保存 → 更新 Avatar。

> 装饰物挂骨骼 = 基底 `../00-基底/00-02-modular-avatar.md` 的 Bone Proxy 场景；若道具要跟随 MA 烘焙后的骨骼，优先 Bone Proxy 而不是裸父子级。

### §1.5 换皮肤 = 换材质主色贴图 + 5 项导入设置

1. 把皮肤图拖进皮肤文件夹（默认皮肤有艾利克斯 / 史蒂夫 / 老版史蒂夫可参考）。
2. 选中模型 `Body` → Inspector 展开 `Material` → 展开 **主色/Alpha** → 把图片拖到「主色/透明度」小图标上。
   - `SteveV2` 身体拆成多个部分，选 **`MC_Body`**；该版本已修复贴图反向问题。
   - 支持 `64x64`（Java 版）与 `128x128`（基岩版），**推荐 128x128**。
3. 选中该图片，在 Inspector 里改 5 项后 **Apply**：

| 设置 | 值 | 含义（作者原话） |
|---|---|---|
| Alpha Is Transparency | 勾选 | 激活图片的透明通道 |
| Wrap Mode | `Clamp` | 拉伸贴图边缘（不重复平铺） |
| Filter Mode | `Point (no filter)` | 贴图变为块状（像素风不糊） |
| Max Size | `512` | 限制贴图最大尺寸 |
| Compression | `None` | 不压缩贴图 |

### §1.6 纤瘦版（Slim）皮肤——仅 `SteveV2`

- **关闭**：`MC_Right Arm(Wide)`、`MC_Left Arm(Wide)`
- **开启**：`MC_Left Arm(Slim)`、`MC_Right Arm(Slim)`

### §1.7 上传 / 覆盖更新已有模型

- 上传前开**加速器路由模式**（见分册 17 §4.1）；Unity 菜单 `VRChat SDK → Show Control Panel` 登录；状态栏显示 `Allowed to publish avatars` 才算有权限。
- Builder 里填名称、设封面，**只自动修复「八边形」红色报错**，其它非八边形红色报错不用管 → 点 `Build & Publish`。
- **从零重做导致面板信息变空、但要更新老模型**时：
  1. SDK 面板进 **Content Manager**，找到要更新的模型 → 点 **Copy ID**；
  2. Hierarchy 选模型最顶层父级 → 展开 **Pipeline Manager** → **Blueprint ID** 里 `Ctrl+V` 粘贴 → 点 **Attach**；
  3. 回面板，老模型的信息就回来了。
- 这条与 `../06-构建上传与备份/06-01-sdk-upload-and-versioning.md` §5 的 blueprintId 检查顺序配套：**克隆出来的模型必须先清 blueprintId，只有确定要覆盖原模型时才把原 ID 粘回去**。

### §1.8 本地测试模型（Build & Test）

| 优势 | 缺陷 |
|---|---|
| 没有加速器门槛，编译完即可用 | 其他玩家看不见你的模型 |
| 提前生成模型缓存，下次上传更快 | 额外占用存储空间 |
| 上传报错时，可判断问题在**模型**还是**网络** | **无法测试网络同步状态** |
| — | 极少数插件无法生效 |

- 用：Build Type 改 `Build & Test Your Avatar` → Build & Test。
- 游戏内看：菜单 → 虚拟形象 → **My Avatars** → 左侧列表下方 **SDK Test Avatars**。
- 删除测试模型：SDK 面板 → Content Manager → **Test Avatars** 列表 → **Delete**。

---

## §2 星野爱案例 · 准备（rusk-hoshinoai-1 / -2）

### §2.1 选型（Booth 为主）

前提：已掌握史蒂夫教程；模型必须是 **liltoon 着色器 + SDK 3.0 及以上**；至少准备一件适配的衣服与头发。

| 要买什么 | 搜索与筛选 |
|---|---|
| 角色模型 | 搜角色名 → 类别 `3D Models`、详细分类 `3D Characters`（没有筛选框就点右上角 Filter） |
| 衣服 | 分类 `3D Clothing` |
| 头发 | 关键词 `<角色名> Hair`，**详细分类设为「无指定」**（作者爱乱加标签刷搜索权重） |

- **判断着色器**：看简介里有没有 `liltoon` 字样。
- **判断 SDK 版本**：99% 模型都是 SDK 3.0，**只需避开明确标注 SDK 2.0 的**。
- 渠道：Booth（★推荐）/ Vket / 闲鱼（对主播有版权风险，慎选）。

### §2.2 必需包

`VRChat SDK - Avatars`、`VRChat SDK - Base`（通常自动装）、Avatars 3.0 Manager、Gesture Manager、liltoon、Modular Avatar、NDMF。**版本不一致时统一装最新版或 VCC/ALCOM 推荐版**。

### §2.3 摆场景

1. 导入 `.unitypackage` → 新建 Scene。
2. 角色 prefab 拖入 → **位置与旋转归零**。
3. Directional Light 改成**白色**（防色差影响判断）。
4. 衣服、头发 prefab 拖进角色对象内。
5. 此时**必然穿模**：可以先把原版衣物/头发关掉，便于观察新件。

---

## §3 星野爱案例 · 绑骨（rusk-hoshinoai-3）

### §3.1 先架好测试环境

1. 准备一组测试动作（作者用的动作包含 `Idle / Lie down / Run / TPose / Walk 1 / Walk 2`）。
2. 用 Gesture Manager 的 **Tools → Test Animation**：选中 GestureManager → Enter Play-Mode → 把动作拖进去 → Play。
3. **动手前先 `Ctrl+D` 备份场景**；要拖出子对象前必须先 **`Prefab → Unpack Completely`**（否则拖不动）。

### §3.2 古法绑骨（纯父子级）

思路只有一句：**找同名骨骼 → 放进角色同名骨骼的子级**。

- **头发**：把头发 `Armature` 下的 `Head` 拖到角色 `Armature → Hips → Spine → Chest → Neck → Head` 的 `Head` 下。
- **衣服**：从 `Hips` 起（含 Hips，**不要动车衣服自己的 Armature 本身**），逐层把每个骨骼拖到角色同名骨骼下。
- **只挪有相同名称的骨骼**：衣服自带多出来的骨骼（铃铛、蝴蝶结等）**保留原位**，不要硬塞。
- 挪原版衣物时**只挪位置，不改层级与名称**。

### §3.3 插件法绑骨（MA Merge Armature）

回到备份场景（古法之前的状态）：`Ctrl` 多选头发与衣服的 **`Armature`** → Add Component → **`MA Merge Armature`** → 把人物 `Armature` 拖进「合并目标」→ **点一次「根据合并目标调整骨骼名称」**（模型作者爱加骨骼后缀，这一步是修正关键）。

### §3.4 特例：骨骼结构差太多 → 改用 MA Bone Proxy

本例头发的骨骼只有一层 `Head`，与角色骨架结构差太多 → `MA Merge Armature` 会误判。

改法：**移除头发 Armature 上的 Merge Armature**，改为给**头发的 `Head` 骨骼**加 **`MA Bone Proxy`**，把角色 `Head` 拖进「目标」。

### §3.5 组件选择铁律（最容易弄反的一步）

| 组件 | 装在哪 | 语义 |
|---|---|---|
| `MA Bone Proxy` | **骨骼**（或任何要跟随移动的对象） | 把带此组件的对象挂到目标对象下，跟随它移动 |
| `MA Merge Armature` | **Armature** | 把整套骨架合并到目标骨架；**并非 100% 成功**，受骨骼结构与命名影响 |

- 弄反是常见出错点；**结构差异大时，整套衣服/头发全部改用 Bone Proxy 也可行**。
- 与基底 `../00-基底/00-02-modular-avatar.md` 的分工：基底讲组件语义与 Bake 时机，这里给**判别条件与失败回退**。

---

## §4 菜单与开关（rusk-4 / -5 / -7）——细节见分册 15 §1

### §4.1 判断模型是否自带菜单

Descriptor → `Expressions` 里若显示 **`Customize`** = 该模型**没有**自带菜单（全新创建）；若已有 Menu / Parameters 框 = 自带菜单（扩展时直接双击编辑）。

### §4.2 菜单资产与硬上限

- 新建：右键 `Create → VRChat → Avatars → Expressions Menu`；点 Descriptor 的 `Customize` 会出现 Menu / Parameters 两个框，把菜单拖进 Menu。
- **命名规则：字母 + 数字 + 下划线 + 名称；所有文件名不含中文**（可用拼音）。
- 菜单结构只有两块：`Parameters` 与 `Controls`；**每页最多 8 个选项**（界面显示 `0/8`）。
- 控件字段 = `Name`（可带图标）/ `Type` / `Parameter`；Type：`Toggle`（切换器）/ `SubMenu`（子菜单，可嵌套）/ `RadialPuppet`（径向器，浮点），其它三种本教程不涉及。
- 做法：建子菜单（如 `YiWu`/`TouFa`/`ShiPin`）拖进主菜单选项的 `Sub Menu` 框，Gesture Manager 的 Expressions 面板即可看到。

### §4.3 变量组：Saved / Synced / 256 bit

- 一个角色**只能有 1 个主菜单**（靠 SubMenu 套娃）与 **1 个变量组**；可同步变量容量 **256 Bit**。
- 新建 `Create → VRChat → Avatars → Expression Parameters`（如 `BianLiang`）；默认自带的 3 个表情变量**可安全删除**；拖进 Descriptor 的 Parameters。
- 字段：`Name`（标签，不含中文空格）、`Type`（Int/Float/Bool）、`Default`、`Saved`（**不勾则换世界/重启游戏后配置复位**）、`Synced`（**不勾则只有自己看得见**；把运算放本地、只同步结果时可关）。
- **动画控制器里的变量名与类型必须与变量组完全一致（含大小写）**；控制器的 Parameters 默认是空的，要逐个 `+` 添加。

### §4.4 录制动画的两个硬规矩

- Animation 窗口右上角三点 → **Lock**（防录制被中断）；**用完必须再次点 Lock 解锁**。
- 录制序列：开始录制 → `Ctrl` 多选**所有衣服设为关闭**（保证所有状态值都被写进 clip）→ **再只开启默认衣物**（覆盖该帧的值）→ 结束录制 → 退出预览。每套衣服重复一次。
- 控制器里第一个动画即 `Entry`（橙色）；不是默认状态就在目标上右键 `Set as Layer Default State`。

### §4.5 连线与挂载

- Any State 上右键 `Make Transition` 连到各状态 → 每条线 `Transition Duration(s)` 设 **0** → Conditions 加条件（Greater/Less/Equals/NotEqual），如 `YiWu Equals 0` 走默认衣服。
- 菜单侧：每个选项 `Parameter` 选 `YiWu, Int`，`Value` 填 0/1/2。
- 最后在分类空对象上（如 `====衣服`）加 **`MA Merge Animator`**：拖入控制器、Layer 类型选 **FX**、路径模式选**绝对路径（基于 Avatar 根对象）**、勾选**匹配 Avatar 的 Write Default**。
- **验证完必须把人物 Animator 的 Controller 清空**（选中框按 Delete），否则会把测试控制器带上烘焙。

### §4.6 游戏内测试与上传

- Control Panel 登录后，**只点「八边形 + 带 Auto Fix」的警告**，其它黄/白警告一律不点。
- Build Type 选 `Build & Test Your Avatar` → 进游戏 → 菜单「角色 → SDK 测试角色」套用（本地构建别人看不见）。
- 上传前加速器开**路由（TUN）模式**，Build Type 选 `Build & Publish Your Avatar Online`。

---

## §5 粒子眼部特效（rusk-8）

完整参数表见**分册 15 §3**。本册只补两条案例侧的判断：

- 看不到特效时**先检查 Scene 窗口的图层显示开关**（不是组件坏了）。
- 特效挂法：`MA Bone Proxy` 挂在**头部骨骼**上（而不是裸父子级），这样烘焙后仍跟随头动；对称靠 `Ctrl+D` 复制后把 `Position.X` 与 `Rotation.Y` **取相反数**。

## §6 MA 快捷菜单（rusk-9）

完整 4 步流程见**分册 15 §2**。案例侧要点：动手前**关掉 Unity Editor 做一次项目备份**，并先删掉手搓的菜单、变量与 `====衣物` 上的 `MA Merge Animator`，否则两套系统会互相打架。

---

## §7 形态键被占用导致的「部位消失 / 开关失效」（rusk-10）★ 其它分册未覆盖

> 对应教程章节「修复消失的角色部位」。教程结论：**这类问题没有可靠的自动化修复工具，只能手动排查**。

### §7.1 排查总顺序（照这个顺序走，不要跳）

1. 检查并调整 Avatar 的**形态键参数**；
2. 排查 Avatar 与所属服装**是否装了 MA**（含自己装的与模型自带的）；
3. 确认 Avatar **是否自带开关功能**（最容易与自定义开关冲突）；
4. 核对 **Write Default 是否统一**。

### §7.2 第 1 步：形态键本身

选中 `Body`/`Body2` 等网格，把目标部位的形态键数值改 `0`。

⚠️ 与 `howtouse/how-to-use-unity-5` 的警告配套读：**不是每个形态键都要改 0**，要按模型实际微调，否则可能穿模；若改完**预览不生效**或**模型扭曲超预期**，说明该形态键已被其它功能占用 → **此时先别继续调形态键**，转第 3 步。

### §7.3 第 2 步：MA 的形态键组件

找 **`MA Blendshape Sync`** / **`MA Shape Changer`**，按要求调参或移除。

> 所有权提醒（见 `../07-验证与排查/07-02-evidence-and-ownership.md` §6）：如果这两个组件是**生成输出**（provider 生成的资产/组件），不要直接改，先按 §7.4 找到源再改。

### §7.4 第 3 步：模型自带开关冲突 —— 层权重二分排查法 ★

本例是 Manuka 的袜子开关 `Stocking`：退出运行后**在 Hierarchy 手动关掉袜子，会发现腿部什么都没显示**（说明袜子开关顺带控制了腿）。

1. 把角色的 **FX 控制器拖进 Animator 的 `Controller` 栏**，就能看到动画逻辑（**看完必须清空 Controller**）。
2. 在 Animation 窗口里打开 `Stocking` 开关对应的 clip：它写了 **`Shrink_stocking`** 与 **`Stocking`** 两个形态键的值。
3. 后续自定义开关（例如「隐藏腿部」把这两个形态键设为 `100`）就会和它冲突，导致**袜子开关失效**。
4. **二分定位**：从 `Left Hand`/`Right Hand` 下方第一个控制层开始，把 Animator 的 Layers 面板里该层**权重设为 `0`**（层叫什么名字不重要），运行测试；
   - 问题没修复 → 把该层权重改回 `1`，把**下一层**设 `0`，再测；
   - 循环直到问题改善。本例把 `hideleg` 层权重设为 0 后袜子恢复。
5. **结论**：冲突点是**两个开关共同使用的形态键**。解决 = **删掉袜子开关里控制 `Shrink_stocking` 的属性**，让每个开关只管自己的状态（模块化分离）。

> 这个「按层权重二分」的方法比逐个看 clip 快得多，值得记成通用排查手段。

### §7.5 第 4 步：Write Default 统一（Avatars 3.0 Manager）

1. 装 **Avatars 3.0 Manager**（ALCOM/VCC 里添加）。
2. 顶部菜单 `VRLabs → Avatars 3.0 Manager` → 把角色拖进 `Avatar` 目标框。
3. 切到 **`WD`** 页签：会出现**橙字**提示 WD 状态异常。译文大意：*部分状态未绑定动画片段，WD-OFF 时可能异常；统一 WD 状态会自动用空动画片段填充*。
4. 直接点 **`Set Write Defaults Off`** 或 **`Set Write Defaults On`**，橙字消失。
5. **不同模型适配的状态不同**：点完必须**进运行测试**，保留无异常的那一种。

> 与基底 `../07-验证与排查/07-01-bake-and-validation.md` 的关系：那里讲「MA 声明的 Write Default 要匹配」与烘焙审计；这里是**事后统一与二选一验证**的实操路径。

### §7.6 附带线索：游戏内「只有自己看得见 / 衣物重叠」

切换衣物、头发、特效时出现**仅自己可见 / 他人不可见、衣物重叠**，大概率也是 **Write Default 不合理** → 回到 §7.5。

### §7.7 证据与边界

- §7.2–§7.4 的结论可以在**编辑器内**拿到（`UNITY_RESOLVED` / `PROVIDER_PREVIEW`）。
- §7.4 的「手动关袜子看不到腿」与 §7.6 的「别人看不见」需要**运行时层**证据（Gesture Manager / 客户端多人测试）才能下最终结论。

---

## §8 外观：发色、挑染、眼睛（rusk-11 / -12）

### §8.1 找到要改的网格与材质

- 有些作者把**头发和饰品的贴图塞进同一个网格**：点材质目标框里的材质球，Scene 里会**泛红**高亮该网格覆盖的部分，据此判断。
- 选中头发网格 → 展开 **`Skinned Mesh Renderer` → `Materials`** → 点材质球，Project 会自动定位到该资产。

### §8.2 换色流程（PSD 实时联动）

1. 在材质 Inspector 把 `Language` 设为简体中文 → 进**详细模式** → 纹理设置 → 主色/Alpha → 点「主色」的纹理图案定位贴图（有些材质还有第二/第三主色，本例只改主色）。
2. 贴图右键 `Show in Explorer` → 在 Photoshop 里「文件 → 存储为」→ 保存类型 **Photoshop**、存到**原贴图同位置**、命名如 `Purple_Special.psd`（弹「最大兼容」就勾确定）。
3. 回 Unity 把 `Purple_Special` 拖进材质主色纹理 → 之后 PS 里的改动会**实时反映到 Unity**（注意先选中要改的图层）。
4. 图像 → 调整 → **曲线**，展开通道分别调 **红 / 绿 / 蓝** 曲线调出目标色（可对参考图；涉及色光三原色，没色感可求助他人或 AI）。
5. 回材质详细模式微调（作者标注「仅供参考」）：取消勾选 `阴影设置-阴影`；`法线贴图&法线设置-MatCap-透明度` 适当降低（参考 `0.07`）；取消勾选 `边缘光-边缘光`（**老版 liltoon 里叫 Rim Light**）。

⚠️ **不建议新手直接用材质球调颜色**（易造成光照问题）——与 `../04-外观与表情/04-01-appearance-and-liltoon.md` 的「属性真伪判读」一致：先确认属性确实是你要改的那个，再动手。

### §8.3 挑染

1. PS 新建图层，按贴图方向画一条横线或竖线 → 存盘回 Unity 预览 → 微调线的位置直到命中要挑染处（可再画一条反方向的线继续定位）。
2. 定位后删掉定位用的十字线 → 新图层画挑染。
3. 整理图层，把头发贴图与挑染**分层** → 分别隐藏、各自存为**透明背景 PNG** 放进 `Assets`。
4. 头发贴图放**主色**；勾选**第二主色**并把挑染贴图放进第二主色纹理。
5. 边缘有白边：把挑染贴图的 `Alpha Source` 设 `Input Texture Alpha`、勾 `Alpha Is Transparency`、Apply。

### §8.4 眼睛

- 文档声明**这一章没有详细图文，绘制过程看视频**（对应教程章节；来源见 `ACKNOWLEDGEMENTS.md`）。
- 步骤：打开项目找到脸部/眼睛贴图 → 拖进 PS → 圈出眼睛（用作者给的 PSD / 现成眼睛素材（**必须适配该角色**）/ 钢笔工具手选）→ 调整位置**完全覆盖原眼睛**、隐藏原图层 → **存为 PSD** 放进项目 → 回 Unity 把 **PSD 拖进材质球的「第二主色」** → 调脸部形态键，把 **`眼瞳`** 与 **`高光`** 关掉。

---

## §9 验收清单

- [ ] 模型不紫、材质着色器与模型要求一致（liltoon）。
- [ ] 衣服/头发骨架已合并或已用 Bone Proxy，且**位置旋转归零、骨架根正确**。
- [ ] 动作用 Gesture Manager 验证过；**人物 Animator 的 Controller 已清空**。
- [ ] 菜单每页 ≤ 8 项；参数名/类型在控制器与变量组**完全一致**；`Saved`/`Synced` 按预期勾选。
- [ ] 形态键：目标部位显示正常，且**两个开关没有共用同一形态键**。
- [ ] WD 已按 Avatars 3.0 Manager 统一并**进运行测试验证**。
- [ ] 贴图导入设置（Alpha Is Transparency / Clamp / Point / MaxSize / Compression）按需设置。
- [ ] 构建前只处理**八边形带 Auto Fix** 的告警；上传前开 TUN 模式。

## §10 证据与边界

- 编辑器内所见（绑骨结果、菜单可达性、形态键数值）= `UNITY_RESOLVED`；MA/NDMF 输出 = `PROVIDER_PREVIEW` / `NDMF_BUILT`。
- 「别人看不见」「同步不同步」「Saved 是否生效」= **CLIENT_RUNTIME**，必须多人测试，不能用编辑器里的默认值代替。
- 上传与账号相关动作（登录、Build & Publish、二次确认）按工作区分工执行；本机 `mcp__unity__*` 需要用户先把工程在编辑器里打开。
