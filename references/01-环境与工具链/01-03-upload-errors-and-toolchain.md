# 上传链路：SDK 面板全字段、报错对照与网络权限

示例 SDK 版本 `3.8.2`。本册补的是 **SDK 面板全字段、报错对照、网络前置规则、安装细节**；界面文案随版本变化。

## §0 与其它分册的分工

- `01-02 §5` = SDK 面板的**流程骨架**；本册 = **逐字段语义 + 8 类已知报错的成因/解法 + 安装与网络细节**。
- 报错的**编辑器侧定位**（Console、层级、参数）见基底 `../00-基底/00-07-troubleshooting.md` 与 `../06-构建上传与备份/06-01-sdk-upload-and-versioning.md`；本册只补**中文社区侧的已知成因与对策**。

---

## §1 开工前的系统环境三查（check-system）

**目的是避免后续出现难以排查的「奇怪问题」**，三项都做，不要跳：

1. **用户名是英文**：打开 `C:\Users`，确认你的用户名文件夹为英文（或英文 + 数字），不能是中文。改用户名不在教程范围内（B 站搜「Win10/Win11 用户名」「本地账户」）。
2. **设备名是英文**：Windows 设置 → 关于，确认设备名为英文（或英文 + 数字）。
3. **显示文件扩展名**：文件资源管理器 → 上方「查看」→ 勾选**文件扩展名**（否则会分不清 `.unitypackage` 与压缩包）。

> 这条与 `summary-of-network-issues` 的报错成因直接相关：**系统环境或项目路径含中文会导致上传失败**（见 §4.3 `Failed to upload file.` 与 `Avatar validation failed.`）。
> 本机核对：用户名 `Administrator`（英文）✅；项目路径 `<USERPROFILE>\AppData\Local\VRChatCreatorCompanion\VRChatProjects\LuzzyMeow`（全英文）✅。**但工作区 `<VRCHAT_ROOT>\` 下的路径含中文**（如 `改模素材\`），新建工程时不要放在那里。

---

## §2 安装链路的细节与坑

### §2.1 Unity Hub

- 下载：官网/国际版收录站（最新版）或网盘（与教程一致的旧版）。
- 安装路径**不要含中文**；**安装完先别急着启动**。
- **国际版必须删掉安装目录下 `resources\app-update.yml`**，否则自动更新会把版本换回国内版。国内版可跳过这步。
- 登录后：**凡是带 Skip/跳过 字样的按钮都点跳过**，避免装到不需要的东西。
- 界面英文：齿轮 → `Appearance` → `Language` → 简体中文。
- **安装量**页设置 **安装位置**（Editor 位置，占空间大，**不要设在 Hub 安装目录下**）与 **下载位置**（建议方便手动清理的文件夹）；**两个路径都不能含中文**。
- **许可证**页右上角「添加」→「获取免费的个人版许可证」（需先登录）。

### §2.2 离线安装 Unity Editor

- 国内 Unity **2022 及以后**版本被重定向到**团结引擎**，**在线安装需要魔法**。
- 没有魔法就用 **`Unity 2022.3.22f1` 离线安装包**：安装（默认 C 盘，可改）→ 装完 Finish。
- 关联 Hub：打开 Unity Hub → **安装**界面 → 点「选择位置」→ 找到 `Unity.exe` 所在位置（即 `<UNITY_EXE>`；教程示例为 `D:\Program Files\Unity 2022.3.22f1\Editor\Unity.exe`，**仅供对照，以你本机实际安装位置为准**）→ 「选择编辑器」→ **列表中出现该编辑器即完成**。

> 本机已按此路线装好：`<VRCHAT_ROOT>\tools\Unity\2022.3.22f1\Editor\Unity.exe`（2022.3.22f1c1）；Hub 3.22.3 在 `<VRCHAT_ROOT>\tools\UnityHub\Unity Hub.exe`。
> **Hub 的 `editors-v2.json` 仍是空数组** → 需要在 Hub GUI 里做一次「选择位置」登记（或用 ALCOM/VCC，它们不需要 Hub）。

### §2.3 ALCOM：首次配置与更新

**首次配置（5 步）**：语言 → 确认 Hub/Editor 路径（已装会自动填入）→ **新建项目保存位置**（**路径不要含空格和中文字符**，否则 Unity 或工具可能出问题）→ **备份保存位置**（压缩格式保持默认）→ 勾选**关联 `vcc://` URL 快捷方式**。

**更新**：下载安装包 → 一路「下一步」到出现界面 → 选 **`Add/Reinstall components`** → 自动启动旧版卸载程序 → 手动卸载后自动装新版。
- ⚠️ **全程保持默认选项**；卸载那一步会出现「**清除配置数据**」勾选框——**默认就是不勾，不要手痒去勾**。

### §2.4 镜像源：2026-10-13 的重要更新（ALCOM 1.1.0 后「软件包」改名「资源」）

- **即使取消勾选官方源并配好镜像，ALCOM 内置的 `curated` 与 `official` 两个官方源在多数情况下仍会被优先访问，导致下载失败**。
- ALCOM 另有**隐藏参数 `ignoreOfficialRepository` 与 `ignoreCuratedRepository`**，**写进配置文件后**才能彻底忽略官方源、完全走镜像。
- **这两个参数在程序界面里没有开关入口**；官方提供一键配置脚本（**仅适用 ALCOM，不适用 VCC**）。
- 步骤：① 用脚本/配置屏蔽官方源 → ② 删除左侧所有官方仓库 → ③ 从镜像站添加 `curated` 与 `official`（点「添加到包管理器」→ 浏览器拉起 ALCOM → 「添加存储库」），后续按需加 liltoon、Modular Avatar。

> 与 `01-02 §2` 的关系：那里记录的是**当年**的替换流程与实测；本节是**2026-10-13 的官方更新**，优先级更高——**光删仓库、光改勾选不够，必须处理隐藏配置**。

---

## §3 SDK 控制面板逐字段（示例 `3.8.2`）

### §3.1 登录页签（Authentication）

- 在这里登录 VRChat 账号；**不支持 Steam 登录**。
- 登录后显示：玩家昵称、**世界地图上传权限**（`Allowed to publish worlds` / `Not yet Allowed`）、**角色模型上传权限**（`Allowed to publish avatars` / `Not yet Allowed`）、退出登录按钮。

**权限门槛（登录提示弹窗的原文含义）**：

- 出现「欢迎来到 VRChat 开发者工具包……」的提示 = **游玩时长不够**：
  - 上传 **Avatar** 需玩家等级达到 **New User**；
  - 上传 **世界地图** 需玩家等级达到 **User**。
  - 弹窗按钮：`Developer FAQ`（官方文档）/ `OK` / `VRChat Discord`。
- 出现「看起来您是从新的地方登录！请查看您的电子邮件……」= **IP 位置有变动**（首次登录或换加速器节点）。去邮箱点 VRChat 的信任链接即可。
  - ⚠️ **QQ 邮箱概率性收不到邮件**（已知）。

### §3.2 上传页签（Builder）

- 通俗说这里是**上传模型**的地方；准确说是**编译/构建**模型的地方。
- **带 `*` 的是必填项**：
  - **选择要上传的模型**：一个场景里同时有多个模型时用来指定。
  - **名称 `*`**：模型昵称。
  - **模型权限（能见度）**：
    - `public`：其他人**可以看见也可以复制**你的模型；
    - `private`：其他人**可以看见但无法复制**。
    - ⚠️ 设为公开时**可能面临版权警告**。
  - **模型类别 / 模型类别 2 / 内容警告 / 标签 / 介绍**：元数据。
  - **封面 `*`**：**尺寸 `1200 x 900`**；两种来源 = 「选择本地图像」（仅 `PNG`）或「从场景中截图」。
- **模型检查器**：检查模型是否符合 VRChat 官方最新的**性能与规格**要求。
- **构建类型**：
  - `Build & Publish Your Avatar Online`：构建并上传到你的 VRChat 账号；
  - `Build & Test Your Avatar`：不上传，仅构建到**本地**测试（**仅支持 Windows 平台**）。
- **上传的平台**：`Windows`（PC 玩家可见）/ `Android` / `iOS`；后两者**需要额外安装 Unity 模块**，且**对模型性能要求极为苛刻**。
- **构建并上传按钮**：**上传前必须先设置模型昵称与封面**。

### §3.3 管理页签（Content Manager）与列表界面

- 管理页 = 管理账号中**已上传的所有内容**，拥有最高权限；也可从网页端管理（作者评价「多此一举」）。
- 功能：刷新、搜索、角色模型列表、**本地测试模型列表**（仅当本地存在测试模型时显示）、世界地图列表。
- 列表界面每项可：看封面/名称/权限 → **调整权限** → **复制 ID**（用于更新或覆盖模型）→ **删除** → 从网页查看。

### §3.4 设置页签（Settings）

**开发人员相关设置**（默认值 / 标题 / 作用）：

| 默认 | 标题 | 作用 |
|---|---|---|
| 关闭 | `Show Extra Options on Account page` | 在 SDK 的 Account 页签展开额外的功能按钮 |
| **启用** | `Show Help Boxes on SDK components` | 在 Inspector 里所有 VRChat 组件下方**多显示一行提示文字**，方便理解参数 |
| 关闭 | `API Logging Enabled` | 把 SDK 与 VRChat 服务器的 HTTP 通信全部输出到 Console；**排查上传错误时开，平时关** |
| 关闭 | `Override ... Detail-Preserving ... Downscaling` | 所有纹理 mipmap 再用 Kaiser 滤波锐化一次，远距离贴图更清晰；**实验性质** |
| **启用** | `Use conservative settings for DPID mipmapping` | 上一项的**保守模式**（官方发现激进锐化在部分贴图上出现锯齿/噪点）；想更锐可取消后对比测试 |
| **启用** | `Execute VRChat Constraints in Edit Mode` | 编辑模式下实时运算约束，方便预览；**场景约束太多导致编辑卡顿就关掉**，只在 Play/上传时生效 |
| 收起 | `Show Local IP Address` | 显示本地 IP 地址 |

**角色模型设置**：`Show All Avatar Performance Details`（关闭，显示所有性能参数细节）。

**VRChat 游戏客户端**：测试世界地图要用；`Edit` 指定到 `VRChat.exe`；`Revert to Default` 恢复默认路径。

---

## §4 上传报错对照表（summary-of-network-issues）

### §4.1 网络环境前置规则（先看这段，能省掉一半报错）

上传模型时：

- **不要使用可暂停时长、免费等廉价加速器**；
- **不要同时开多个加速器**；
- **不要把加速器和魔法同时开**；
- **不要把加速器、魔法和组网工具同时开**。

常见组网工具：`Radmin`、`Easytier`、`ZeroTier`、`Tailscale`、`Hamachi`、`蒲公英`、`游侠对战平台`。**用过这些工具的话，可能需要手动关闭相应的虚拟网卡**。

**实测能稳定上传的加速器**：`biubiu`、`UU`（需开 **路由模式 / TUN 模式 / 虚拟网卡模式**）；UU 的路由模式在「节点选择」的**筛选**里。

### §4.2 一条独立硬规则：不要在预制件编辑界面上传

**在预制件（Prefab）编辑模式里上传模型会直接失败**。必须先退出预制件编辑模式再上传。

> 与基底 `../06-构建上传与备份/06-02-backup-and-package.md` 的「工作 prefab」不冲突，但**上传那一刻必须在主场景**。

### §4.3 报错对照表

| 报错原文 | 出现位置 | 含义 | 成因 | 解决 |
|---|---|---|---|---|
| `Failed to load avatar data: "Missing Credentials".` | Console、SDK 面板 | 缺少登录凭证 | 通常上传前**切换了加速器节点** | 开好加速器后**重启项目**（关 Unity Editor 再从 Hub 打开），重新登录 SDK 面板 |
| `Loaded data for an avatar we do not own, clearing blueprint ID.` | Console | 这不是你的 Avatar，已清除蓝图 ID | 网络质量差；**或你用了别人的工程文件** | 换质量好的加速器（UU/biubiu）开 **TUN 模式**，重新上传 |
| `Failed to upload file.` | Console、SDK 面板 | 上传文件失败 | 网络质量差；**或系统环境/项目路径含中文** | 查系统环境与项目路径（§1）；换加速器并开 TUN 模式 |
| `Avatar validation failed.` | Console、SDK 面板 | Avatar 安全验证不通过 | ① 参数超官方上限（面数、同步参数、内存占用）；② 含**无效组件**（衣物/道具上有 `Missing Script`）；③ 系统环境或项目路径含中文；④ 网络/加速器质量差 | 逐条排查：精简参数 / 清掉 `Missing Script` / 改路径 / 换加速器 |
| `Failed to agree to the ownership agreement.` | Console、SDK 面板 | 未能达成所有权协议 | **没弹出协议同意框**，多由网络质量差或 Bug 导致 | 参考作者的视频安装 **VRChat SDK Patcher** |

### §4.4 与基底的分工

- 本表是**中文社区侧的成因与对策**；组件级、参数级的定位仍走基底 `../00-基底/00-07-troubleshooting.md`（症状 → 根因清单、上传报错对照表）与 `../06-构建上传与备份/06-01-sdk-upload-and-versioning.md` §6 的告警三级分类。
- ⚠️ 「自动修复红色八边形告警」只适用于**八边形 + Auto Fix** 的那类；**其它红色告警不要瞎点**（分册 14 §5 与本册 §3.2 同源）。
- ⚠️ `Loaded data for an avatar we do not own, clearing blueprint ID.` 会在**网络差时误清 blueprintId**——这正是 `../06-构建上传与备份/06-01-sdk-upload-and-versioning.md` §5 要求「克隆后先清、要覆盖才粘回」的原因：**ID 状态必须每次重新读回确认，不能靠记忆**。

---

## §5 覆盖更新已有模型（从零重做后找回旧信息）

1. SDK 面板 → **Content Manager** → 找到要更新的模型 → 点 **Copy ID**。
2. Hierarchy 选模型**最顶层父级** → 展开 **Pipeline Manager** → 在 **Blueprint ID** 里 `Ctrl+V` 粘贴 → 点 **Attach**。
3. 回 SDK 面板，老模型信息恢复 → 按正常流程覆盖上传。

> 只有**确认要覆盖原模型**时才做这一步；克隆/实验用的模型必须保持 blueprintId 为空，否则会误覆盖线上模型。

---

## §6 本地测试模型（Build & Test）的取舍

| 优势 | 缺陷 |
|---|---|
| 没有加速器门槛，编译完即可用 | 其他玩家看不见 |
| 提前生成模型缓存，下次上传更快 | 额外占用存储空间 |
| 上传报错时可判断问题在**模型**还是**网络** | **无法测试网络同步状态**；极少数插件不生效 |

- 查看：游戏菜单 → 虚拟形象 → My Avatars → **SDK Test Avatars**。
- 删除：SDK 面板 → Content Manager → **Test Avatars** → `Delete`。

---

## §7 验收清单

- [ ] 系统用户名、设备名、项目路径、**工作路径**均无中文；已开启文件扩展名显示。
- [ ] Hub 的「安装位置」不在 Hub 目录下且不含中文；国际版已删 `app-update.yml`；许可证已激活。
- [ ] ALCOM 首次配置的项目保存路径**不含空格与中文**；`vcc://` 已关联；更新时没勾「清除配置数据」。
- [ ] 镜像源：**隐藏配置 `ignoreOfficialRepository` / `ignoreCuratedRepository` 已处理**，官方仓库已删，`curated` + `official` 已从镜像站添加。
- [ ] SDK 面板：`Allowed to publish avatars` 已就绪；名称与封面（`1200x900` PNG）已填。
- [ ] 上传时**不在预制件编辑模式**；加速器只开一个、开 TUN/路由模式、组网工具已关。
- [ ] 报错时先开 `API Logging Enabled` 抓 Console，再按 §4.3 对照。
- [ ] 覆盖老模型前确认 blueprintId 是**故意粘回**的。

## §8 证据与边界

- 面板字段与门槛属**官方产品行为**，但版本会变（示例是 `3.8.2`）——**以当前 SDK 实际界面为准**。
- 「加速器稳定上传」是**社区实测结论**（作者的样本），不是官方承诺；换网络环境需重新验证。
- 权限门槛（New User / User）来自面板弹窗原文；本包其它分册若引用二手来源，以本册为准。
