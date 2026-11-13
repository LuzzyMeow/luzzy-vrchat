# 改模工具链、素材整理与上传

本册覆盖工程与工具链、素材整理与导入、上传前检查、上传与性能等级、网络问题五个部分。素材来源与视频清单见 `ACKNOWLEDGEMENTS.md`。

---

## §1 工程与工具链

### §1.1 安装阶段的硬性选择

- **不要用「中文特供版」Unity / Hub**——版本号带 `c1` 的（如 `2022.3.22f1c1`）**会造成各种未知问题**；VRChat SDK 指定版本 = **2022.3.22f1**（英文原版）。
  > ⚠️ 与本机现状冲突：本机装的是 `2022.3.22f1c1`（`<VRCHAT_ROOT>\tools\Unity\2022.3.22f1\Editor\Unity.exe`）。作者只给了经验判断、没给可复现故障，**要不要换装原版属于需要用户决定的事**，不要自作主张卸载重装。
- **Android 模块只在要做 Quest / PICO 模型时勾**，否则不勾。

### §1.2 打开已有工程 / 新建工程

- **打开别人给的工程文件夹**：VCC → `Create` 右侧小三角 → `Add` → **必须选到文件夹**（不是压缩包）→ 刷新后 `Open`。
- **新建 Avatar 工程**：VCC `Create` → 选**第二个**（第一个是 World；后面两个是旧版本，别用）。
- **命名建议英文；路径不要含中文**——部分插件/工具识别不了中文路径（与 `01-03` 的系统三查一致）。

### §1.3 必备包与安装方式

| 包 | 用途 / 备注 | 安装方式 |
|---|---|---|
| **Gesture Manager** | **必装**：Play 模式模拟 VRChat 运行时 —— 手势、开关、表情、菜单、PhysBone / 动骨；新工程建好后先核实它已装（`vpm-manifest.json` 里有 `vrchat.blackstartx.gesture-manager`，本机工程是 3.9.9） | VCC / ALCOM 包（详见 `01-06` §2.7） |
| **Avatars 3.0 Manager** | 在 FX 层/参数层做绑定；**多数场景可被 MA/VRCFury 替代，但参数来源不明时必须用它** | VCC 包 |
| **Modular Avatar (MA)** | 菜单 / 绑骨 / 换衣一站式（作者推荐） | VCC 包 |
| **VRCFury** | 一键化流程（作者推荐） | VCC 包 |
| **「如梦」穿衣插件 = `VRCAvatarToolkit`** | Unity 顶部出现「**VRC工具箱**」；**FX 层只占 1 个参数槽**（作者强烈推荐） | 见 §1.4 |
| lilToon + Poiyomi | 两款着色器**都装** | 见 §1.5 |
| 面补（面部追踪）包 | 按各作者文档装对应版本（如 VRCFT） | 各自文档 |

- **安装位置差异（重要）**：**VCC 装的包在 `Packages` 下；自己拖进来的 unitypackage 落在 `Assets` 下**。
  判据见 `01-04`（解压后含 `package.json` 的进 `Packages`；`Packages` 里的文件不要改）。

### §1.4 「如梦」VRC工具箱 = `VRCAvatarToolkit`

- Unity 顶部菜单栏出现「**VRC工具箱**」；作者用它做**一键穿衣 + 生成衣柜菜单**（实操见 `05-02` §2.2）。
- 优点：**FX 层只占 1 个参数槽**（对比手搓换装层的参数开销）。
- 仓库（供核对，非安装教程）：[gitee.com/cmoyuer/vrchat-avatar-toolkit](https://gitee.com/cmoyuer/vrchat-avatar-toolkit)（说明：CC 工具创建 Avatar 工程 → Unity 打开 → 双击导入 unitypackage → 顶部出现「VRC工具箱」）。

### §1.5 着色器文件夹与模型优化插件

- 着色器**专用文件夹不能删、不能移、不能改动**（否则材质丢引用/变紫）。
- 模型优化插件作者只推荐第一个；第二个是 **Quest 转换**用途，小众。
  视频口播名字含混、AI 转写不可信，**不要把插件名当事实抄**；要装就按工程已有清单核对，参考基底 `references\00-06-plugin-ecosystem.md`。

### §1.6 ALCOM 完全可以代替 VCC（④）

- 文档：<https://docs.vrcd.org.cn/books/vrchat/chapter/alcom>。
- 国内网络更友好；**VCC 已装的包都会显示**；**可以手动指定 Unity 路径**。
- 网站上的「添加到 VCC」按钮会被 **ALCOM 拦截并打开 ALCOM**（装了 ALCOM 后不必走 VCC）。
- 常用操作：**「管理 → 一键升级所有软件包 → 应用」**；勾选指定包后 **「安装所选软件包 → 应用」**。
- 与 `01-02/04` 的关系：那两册讲 VCC + 镜像源（含 2026-10-13 的隐藏配置 `ignoreOfficialRepository` / `ignoreCuratedRepository`）；本册给的是 **ALCOM 这条替代路径**。本机已装 ALCOM 1.1.8（`<VRCHAT_ROOT>\tools\ALCOM\ALCOM.exe`）。

### §1.7 Unity Hub 许可证失效的处理（④）

- 症状：**新建 3D 工程无法完成 / 进不去**。
- 处理：Hub → **管理许可证** → **退还许可证** → **重新添加「免费个人许可证」**。
- （本机 Unity 授权文件 `%PROGRAMDATA%\Unity\Unity_lic.ulf` 仍在，通常无需重新登录。）

### §1.8 `lilEditorToolbox`：层级增强（④）

- 站点：<https://lilxyzw.github.io/lilEditorToolbox/>。
- 功能：**Hierarchy 里显示组件图标、快速勾选**（选中对象直接按勾选键，不必把鼠标移到很窄的复选框上）。
- 图标能区分**动骨组件 / 音频 / 粒子**等 → 组件堆很长的模型上找东西快很多。
- 开启方式：**编辑 → 首选项**。

### §1.9 `lilycalInventory` vs MA「联动开关」（④）

**`lilycalInventory`**（<https://lilxyzw.github.io/lilycalInventory/>，魔法のインベントリー，= 快速穿衣）：

1. 选中要加开关的物件 → Hierarchy 里按 `F` 定位 → 搜索 `Lily` → 语言切**简体中文**；
2. 给物件加开关**不需要录动画、不需要改 FX 层**；
3. 运行后一级菜单出现该物件的**独立 Bool 开关**（参数自动建为 bool，`safe` 与同步都勾上）；
4. **可在开关对象里加「附加物品」**，把月亮标志等其它物件一起开关（注意：**会覆盖该物品本身的开关**）；
5. 命令名可以改。

**MA 的联动开关**：在物件下加 **`MA Object Toggle`**，把要联动的物件拖进去——**不占新参数**（原理是改已有动画）。作者提醒：**On/Off 语义容易混乱**（同一状态给 On、不同状态给 Off），Unity 里的默认状态也要设成对应的开/关。

| 需求 | 用哪个 |
|---|---|
| 什么都没有，**凭空生成**快捷开关 | `lilycalInventory` |
| **已有开关**，只想联动且**不占参数** | `MA Object Toggle` |

（provider 归属与生成物所有权见 `07-03-provider-playbooks.md` §2、`07-02-evidence-and-ownership.md` §6。）

### §1.10 「基本模板」工程：一次装包，长期复用（①）

1. 新建工程 → 把 §1.3 的包**全部装好**；
2. **先「打开工程文件夹」**（文件管理器里）→ **把工程目录重命名为「基本模板」**；
3. 以后每次改模 → **复制一份副本** → 用副本开工（避免重复装包）；
4. 复制出来的副本要**在 VCC 里重新 `Add` 一次**才会出现在 Manage 列表里。

---

## §2 素材整理与导入

### §2.1 素材目录约定

- **所有素材放同一个文件夹内分类**（与 `02-01-workspace-and-work-assets.md` §1 的 `Assets/<WorkRoot>/` 约定一致）。
- **文件夹名前加 `0` 或 `!`** → 排序时自动排到最前面（便于一眼找到自己的东西）。

### §2.2 Booth 选购

- 带标签的检索链接：<https://booth.pm/zh-cn/items?in_stock=true&tags%5B%5D=VRChat&adult=include>；
- 标价是**日元**；付款用 **PayPal 或代购**。

### §2.3 导入的四种情形

| 拿到的文件 | 处理 |
|---|---|
| `.unitypackage` | 拖进 Unity 导入；**导入后视图会回到 `Assets` 首页，需要手动把新增文件夹拖到自己的目标目录** |
| 压缩包 | 解压后再拖（含 `package.json` 的进 `Packages`，见 `01-04`） |
| 贴图 / FBX | 直接拖进 `Assets` |
| 模型 prefab | 命名里常带**着色器版本**（例：一个 lilToon 版、一个 UTS 版）→ **没装 UTS 就用 lilToon 版**（否则材质变紫/丢失） |

- **空 FBX 直接拖进 Unity 没用**（没有组件、不成 prefab）。
- 选中模型按 `F` **定位并缩放到视野**。
- 「材质变紫」的成因见 `04-01-appearance-and-liltoon.md` 与基底 `../00-基底/00-07-troubleshooting.md`。

---

## §3 上传前检查清单

1. **视角球（View Position）**：要编辑/锁定位置（决定第一人称视角，歪了在 VR 里像"眼睛长在胸口"）。
2. **脸的形态键与嘴型**：选中脸对象看 BlendShape 列表（基础见 `01-04`）。
3. **Eye Look 两种实现**：① **骨骼追踪**——两个眼球骨骼；② **形态键追踪**——眨眼/上下看的形态键。选错会没有眼动或眼睛翻白。
4. **五个 Animator 层（Base / FX / Gesture / Action / Additive）**：
   - **FX 层最重要**（表情、开关都走它）；
   - ⚠️ **只填 Base + FX 的实测事故症状**：**VRC 自带 collider 碰撞器装不上、上传后手上骨骼缺失、不能抓别人的东西、动骨碰撞失效**；
   - **Additive 为空无所谓**。
   → 用 Avatars 3.0 Manager 一次性核对/补齐更快（见 `03-03` §7 的 WD 页签）。
5. **菜单与参数**：参数共 **256** 槽，**int / float 各占 8 槽、bool 占 1 槽**（作者当时用掉 30 槽）。完整预算与读取链见 `03-01-parameter-and-menu-audit.md` §1。
6. **蓝图（Pipeline Manager 的 Blueprint ID）**：**首次上传必须为空**。**上传过的人把 ID 复制到别人模型上会覆盖旧模型**（同一 ID = 同一内容，后上传的覆盖前一个）。检查顺序与读回见 `06-01-sdk-upload-and-versioning.md` §5。

### §3.1 Gesture Manager 跑测试

1. 先**确认已装**（VCC / ALCOM 包 `vrchat.blackstartx.gesture-manager`；新建工程第一件事就是核实它 —— 见 `01-06` §2.7）；2. 顶部菜单 `Tools → Gesture Manager Emulator` 生成 `Gesture Manager` 对象 → `Enter Play Mode`；
3. 选中相机 → **`Ctrl+Shift+F`**（把相机刷新/对齐到需要观察的位置）；
4. Play 模式下依次检查：**行动是否正常 / 开关是否生效 / 表情是否正常**。
（验收矩阵见 `07-01-bake-and-validation.md` §8；MCP 下的分帧验收纪律见 `07-04-verification-discipline.md`。）
5. **随时 `Ctrl+S` 保存场景**；Inspector 可以**多开选项卡**（锁定一个对象后继续点别的）。

---

## §4 上传与性能等级

### §4.1 面板操作

- SDK 面板（`Show Control Panel`）→ 填**名称 / 描述 / 图片**（图片可以从**当前相机抓一张**，也可以用文件里任意一张）；
- **要勾上「建立上传」那一项**；
- `Build & Test` = **只本地可见**；`Publish` = **真正上传**。

### §4.2 性能等级 `Very Poor` 的四个成因（作者实测）

| 成因 | 作者给的参考值 |
|---|---|
| 三角面过多 | **10 万+**（作者说绿模要求 **7 万以下**） |
| 材质槽过多 | 材质槽偏多会拉低等级 |
| 动骨（PhysBone）过多 | **50 个** |
| 粒子过多 | **23**（作者说上限 **16**） |

> 这些是**作者当时的经验值/当时门槛**，VRChat 的性能阈值随版本变化。**以 SDK 面板实际给出的等级与限制条目为准**；优化手段见基底 `../00-基底/00-07-troubleshooting.md` 与 `05-01-outfit-fit-and-clipping.md`。

### §4.3 一条实战排查经验

- **`Build & Test` 成功、`Publish` 失败 ⇒ 大概率是网络问题**，不是模型问题（先查网络，别乱改模型；出口见 §5）。

---

## §5 网络问题

- 作者专门用 1 分钟视频讲「网络问题是宝宝改模最常见的问题」：**加 QQ 群 `250462914`**，群公告里有各类网络问题的解决办法（视频里部分字样打了码）。
- 不依赖群的既有对策：镜像源替换与隐藏配置见 `01-02/04`；Booth 打不开 / VPM 装不上包 / 官网打不开的分流排查见基底 `../00-基底/00-07-troubleshooting.md`。

---

## §6 与既有分册的对应

| 主题 | 去哪 |
|---|---|
| 中文从零安装链路、镜像源、SDK 面板逐字段、上传报错对照 | `01-02` / `01-03` |
| `Assets` vs `Packages` 判据、界面图标、同步范围 | `01-04` |
| 参数预算与菜单上限、四表面同步 | `03-01-parameter-and-menu-audit.md` |
| 工作目录约定、C# 编辑器脚本、FX 合并 | `02-01-workspace-and-work-assets.md` |
| 证据分层与「有没有验证过」的写法 | `07-02-evidence-and-ownership.md` |
| 绑骨、录动画、衣柜、动骨作者流程、fbx 替换网格 | `05-02-bind-animation-physbone-fbx.md` |
| MA 菜单/开关/换色/轮盘的逐组件配方 | `03-04-ma-menu-and-switch-cookbook.md` |

## §7 纪律

- **教程口径随版本漂移**（VCC/Unity/Hub 界面、性能阈值、插件名都会变）。照做前先用**工程里的实际清单**核对；对不上就标 `STALE` 并说明差异，不要照旧视频硬找。
- 涉及**卸载/重装 Unity、换特供版**这类不可逆动作（§1.1），**必须停下来问用户**（见 `07-02-evidence-and-ownership.md` §2.1）。
- 上传相关动作（登录、`Build & Test`、`Publish`、面板二次确认）在基底 `../../SKILL.md` 铁律 3 里定为**用户动作**（若工作区根存在 `AGENTS.md`，其必读区同级有效）；自动化只能到「本地 GUI 模式下点 `[MenuItem]`」这一层（见 `06-01-sdk-upload-and-versioning.md` §8）。
