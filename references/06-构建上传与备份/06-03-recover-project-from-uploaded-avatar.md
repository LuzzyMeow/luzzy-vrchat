# 从「已上传到 VRChat 的模型」恢复工程（AssetBundle 逆向路线）

> **来源**：2026-10-18 网络调研（anysearch + GitHub API + VRChat API 文档）+ **2026-10-19 本机实测**（AssetRipper 2.0 导出真实 `.vrca`、Unity 无头导入，见 §14；缓存加密与反盗模为文档/社区/工具口径，标「未实测」处不要当结论）。
> **红线**：**只能对自己账号上传的模型**这么做 —— 逆向 / 提取他人模型违反 VRChat ToS，可能导致封号；下面列的工具作者也都写明 owner-only。

## 1. 先明确：能恢复什么、不能恢复什么

| 能拿到 | 拿不到 |
|---|---|
| 上传时的 **AssetBundle（`.vrca`，未加密）** | 原始 Booth / 商店 **源包**（衣服、头发、道具原文件） |
| 网格 / 贴图 / 材质 / 动画剪辑 / AnimatorController / 预制件 | 原始 **MA / NDMF / VRCFury 的编辑态设置**（上传前已被 bake 掉） |
| VRChat 白名单组件的序列化数据（Avatar Descriptor、PhysBone、Contact、菜单资产） | 原工程的 **VPM manifest / 包清单 / 工程设置 / 版本历史** |
| 可继续编辑的 Unity 工程骨架（工具会帮你修 script GUID / shader GUID） | 原始文件夹结构与命名规范 |

**结论**：你拿回的不是「丢的那个工程」，而是**成品（baked 结果）重建出来的工程** —— 能继续改，但 MA 开关 / 菜单这些原编辑态要自己补。

## 2. 三步链路

### 第 1 步：拿自己账号的 `.vrca`（走官方 API）

- 官方接口：`GET https://api.vrchat.cloud/api/1/file/{fileId}/{versionId}`，认证用 **`auth` cookie**（文档：`vrchat.community/reference/download-file-version`）。
- `fileId` 来自登录后 `GET /api/1/avatars` 里自己 avatar 对象的文件字段；现成工具会自动枚举。
- 现成下载工具（都限定「仅自己账号」）：

| 工具 | 形态 | 备注 |
|---|---|---|
| `Null-K/VRChatVRCADownloader` | 独立 GUI | 登录后列出自己的模型，双击下载（PuddingKC 2026-02 教程、2026-09 更新） |
| `cocokoishi/VRCALoader` | Unity 内工具（Editor 扩展） | 10★，116 commits：从账号下载 + 加载 + 恢复 `vrca` / `vrcw` |
| `VRCUploader/vrca-download-script` | 脚本 | 2★ |
| `maikyes/vrchat-authorized-asset-downloader` | 浏览器扩展 / Tampermonkey | 2★，owner-authorized VRCA / VRCW |

### 第 2 步：`.vrca` → Unity 工程

| 工具 | 形态 | 说明 |
|---|---|---|
| **`Nickel-JP/avatar-recovery-unity`（AvatarRecovery 1.2.22）** | **VPM 包（推荐）** | 内置 **AssetRipper 1.3.14**；Unity 2022.3 + VRChat SDK Base **3.10.5 ≤ x < 3.11.0**；一键把 `.vrca` / `.vrcw` / `.vrcp` / `.vrcap` 还原进工程，并做后处理：Script GUID / Shader GUID / Missing Script / Prefab 选择 / Pose Reset；建议装 Poiyomi 或 lilToon。VPM 源：`https://nickel-jp.github.io/avatar-recovery-unity/index.json`，装完菜单 `AvatarRecovery → Open Window`。9★，2026-10-12 仍在更新 |
| **`200Tigersbloxed/dVRC`** | Unity 插件（C#） | 69★ / **GPL-3.0** / 2026-06 更新；只恢复「你自己上传」的资产；仓库含 `DownloadTools.cs`（从账号下载）与 `AssetLoader.cs`（把 bundle 载入工程） |
| **`AssetRipper`** | 独立 GUI / CLI | **8659★** 通用引擎，直接把 `.vrca` 喂给它也能导出工程（AvatarRecovery 就是封装它 + 后处理） |
| `AssetStudio` / `AssetStudioMod` | 独立 GUI | 更偏「浏览 + 导出网格贴图」，工程化（组件/预制件还原）弱于 AssetRipper |

### 第 3 步：验收与补全

1. 装回 **lilToon / Poiyomi**（否则材质发紫）；SDK 版本对齐 3.10.x。
2. `read_console` 看 missing script / 材质异常；`Tools → Gesture Manager Emulator` 进 Play 跑一遍（KB 铁律 5）。
3. 恢复出来的 AnimatorController / 菜单资产通常是 **baked 后**的；想回到「MA 可维护」状态要自己重做（见 `03-菜单与参数`）。
4. 脚本 / shader GUID 对不上时，正是 AvatarRecovery 的 GUID 修复功能要解决的场景。

## 3. 备选路线：本机缓存

模型如果已经在游戏里加载过，`.vrca` 会落在 VRChat 缓存目录（`%USERPROFILE%\AppData\LocalLow\VRChat\VRChat\Cache-WindowsPlayer\` 一类）；
可配合缓存工具（`ShayBox/VRC-LOG` 58★ 记录缓存里的 avatar ID、`VRC1337/VRChatStudio` 21★ 解缓存）后再交给 AssetRipper。**缓存可能加密/混淆，且不如官方 API 干净。**

## 4. 风险与红线

- **只对自己上传的模型做**：VRChat ToS 禁止逆向 / 提取他人模型；工具作者也写明 owner-only。盗模会封号。
- 下载要账号 `auth` cookie：**别把 cookie 交给来路不明的工具**，别贴到聊天 / 日志 / 仓库里。
- **「只有 Blueprint ID 就能恢复」是错的** —— VRChat 论坛 2025-10 的官方回复明确：无法只靠 blueprint ID 找回文件；必须有能访问该 avatar 文件的账号 + 文件 ID。

## 5. 未验证 / 待办

- 本机**未实测**上述任何工具；AvatarRecovery 的 VPM 源能否直连、与 VRChat SDK 3.10.5 的实际兼容性均待验证。
- **已核实（2026-10-18，VRChat API 文档 `vrchat.community/reference/get-avatar`）**：avatar 对象里的文件信息在 **`unityPackages[]`** ——
  每项有 `assetUrl`（形如 `https://api.vrchat.cloud/api/1/file/<fileId>/<versionId>/file`，直接带 `auth` cookie GET 即可下载）
  与 `assetVersion` / `platform` / `unityVersion` 等；另有 `unityPackageUrl` / `unityPackageUrlObject` 字段（历史字段，可能指向 unitypackage）。→ **取 `fileId` 不必猜，
  从 `unityPackages[].assetUrl` 里取即可**。
- **待实测**：`unityPackages[].assetUrl` 下到的文件是否就是 `.vrca`（未加密）；`unityPackageUrl` 现在是否还返回**原始 unitypackage**（若能，比 `.vrca` 更接近原工程）。
- 若日后实测：拿一个自己上传的小模型走一遍 1→2→3，记录 AssetRipper 产出的资产清单与 missing script 数量，再回填本册。
- **2026-10-19 更新**：AssetRipper 2.0 的载入 / 导出与 Unity 无头导入**已在本机实测**（§14，真实样本 `Shipilka.vrca`）；缓存加密、反盗模、工具全景见 §15–§19；**FACS 12 步仍未走完**（§10）。
---

## 6. 外部演示版恢复流程（非本机实测）

与 §2 是同一套方案（AvatarRecovery），但演示把**动手顺序**讲清楚了：

1. 复制恢复包的 **VPM 仓库地址**（演示里只在画面上给出；社区流传的是 `https://nickel-jp.github.io/avatar-recovery-unity/index.json`，**装前自己核对来源**）→ VCC / ALCOM「添加存储库」。
2. 新建 **Avatar 项目**。
3. 软件包管理里添加 **`avatar recovery`** 包。
4. 打开 Unity → 工具栏 **`Avatar Recovery → Open Window`** → 选中 `.vrca` → **`Run`**（要等一会儿）。
5. 把恢复出来的 prefab **拖进 Hierarchy**：渲染不正常是正常的 —— **缺 shader**。
6. 工具栏 **`Avatar Recovery → Open Shader List`** → 把 **preset 文件**拖进新窗口 → 它会列出**缺哪些 shader** → **挨个补全**（lilToon / Poiyomi 等）。
7. 补完 shader 即可正常渲染与上传（演示里上传成功）。

- 没有 `.vrca` 时用他提到的下载项目获取（见 §2 第 1 步）。
- 演示评论提醒：这种能力**更容易被用于盗模**（「比起恢复工程，感觉更多人会用于盗模」）—— 再次强调 **owner-only**（§4）。

## 7. 恢复后的颜色 / 肤色修复：NoNlab「双模颜色同步模块」（2026-10-18 补）

VRCA 恢复（以及**换头 / 拼模型**）后最常见的遗留问题是**颜色对不上**：材质色值丢失、脖子/接缝肤色不一致。社区有一个直接解法：

- **项目**：`Ninsterrrr/NoNlab-`（GitHub，**MIT**，V1.0，2026-10-16；据发布说明 + README + 评论整理，来源见 `ACKNOWLEDGEMENTS.md`）
- **包体**：`双模颜色同步模块.unitypackage`，约 28 KB；**纯原生 C#、无第三方依赖**；Unity 2022.3 LTS（Built-in / URP / HDRP 都可）
- **安装**：`Assets → Import Package → Custom Package` 导入 → 菜单 `NoNlab → 双模颜色同步模块`
- **干什么**：把一个模型（**源**）的颜色**迁移**到另一个模型（**目标**）上，像素级精度。双路径：
  - **VertexColor**（顶点色，快，10–50ms）
  - **Texture**（贴图，精确：面级反投影 + 3D BVH 查询 + 像素级采样；604K 像素 / 92ms，多线程）
- **用法**：源槽/目标槽各放一个模型（或「从选中获取」）→ 输出模式选 `Texture` / `VertexColor` → 颜色来源 `Material Texture` / `Manual Texture` / `Vertex Color` → 迁移模式 **`DR`（直接替换）
  ** 或 `HSL`（色相迁移）→ 调「影响距离 / 全染内径 / 渐变外径 / 色调迁移强度」→ 预览 → 应用（**可 Ctrl+Z 撤销**）。
- **脸部 / 脖子接缝**（README 已知问题）：影响距离调大、关「法线加权」、降低混合强度；**头与脖子要匹配时：输出模式 `Texture` + 迁移模式 `Direct Replace`**。
- **坑**：目标贴图必须 **Read/Write Enabled**（工具内置「一键修复」按钮）；贴图路径下可在 Scene 视图点源模型做「屏幕取色」限制采样区域。
- 评论区提到类似旧插件 `texcoladjuster`（可参考）；作者说 V2.0 会改成可视化单像素交互。
- **本机已存档（2026-10-18）**：`<VRCHAT_ROOT>\resources\09_工具与插件\NoNlab-双模颜色同步模块\` —— 含 `双模颜色同步模块.unitypackage`（28,274 B，
  sha256 `E55EED16AE4FEAEA35CD9CBD9EB40F1E72495C004F85CECCF560BAC404699680`）、GitHub 原样 `README.md`、`来源与许可.md`；
  已校验 gzip 头与包结构（内含 `Assets/NoNlab/双模型颜色自适应模块/Editor/` 8 个 C# 脚本）。**仍未装进工程实测。**

## 8. 来源与状态（与 §5 一起看）

- 演示视频与工具来源见 `ACKNOWLEDGEMENTS.md`。
- 文档：`vrchat.community/reference/download-file-version`、AvatarRecovery 的 VPM Catalog 页、`Ninsterrrr/NoNlab-` 的 README
- **状态：两段演示与两个工具均未在本机实测**（§5 的复核点仍然有效）。


---

## 9. 三条 `.vrca` 来源（2026-10-19 补充，★ 路线 2 本机已实测文件真实性）

| 路线 | 前提 | 怎么拿 | 优点 | 缺点 |
|---|---|---|---|---|
| 1. 官方 API | 账号能登录、模型还在自己账号下 | `GET /api/1/file/{fileId}/{versionId}` + `auth` cookie（§2 第 1 步） | 任何电脑都行、文件保证是你的 | 要账号/cookie；账号异常时拿不到 |
| **2. 本地测试资产包（★ 推荐，不需要账号）** | **曾经在 Unity 里点过 Build & Test**（哪怕一次） | `<USERPROFILE>\AppData\LocalLow\VRChat\VRChat\Avatars\<名字>.vrca` | **最快**、离线、明文 | 必须当初做过 Build & Test，且没清过 LocalLow |
| 3. 本机缓存 | 在游戏里加载过该模型 | `%LOCALAPPDATA%Low\VRChat\VRChat\Cache-WindowsPlayer\` 一类，配合缓存工具 | 没上传过/没 Build 过也可能有 | **新版可能加密**；混着他人模型，别碰 |

### 9.1 路线 2 实测证据（本机）

- 文件：`<USERPROFILE>\AppData\LocalLow\VRChat\VRChat\Avatars\Shipilka.vrca`（**18.36 MB**，2026-07-28，说明本机 7 月做过该模型的 Build & Test）
- 文件头 16 字节：`55 6E 69 74 79 46 53 00 00 00 00 08 35 2E 78 2E` = **`UnityFS.....5.x.`** → **明文 UnityFS 5.x AssetBundle，未加密**，可直接喂 AssetRipper ✅
- `LocalAvatarData\` 目录**不是空的**（2026-10-19 复核：2 个 `usr_*` 用户夹，内含 `avtr_*` JSON 参数快照，如 `eyeHeight` / `animationParameters`），它是**本地测试头像的元数据、不是模型文件**；
  `Cache-WindowsPlayer\` 本机不存在（没在游戏里加载过内容）。缓存里的 `__data` 是加密的，见 §15。

### 9.2 缓存路线：**2025 起是 AES 加密的，不是「可能混淆」**（2026-10-19 复核）

- 缓存目录里不是 `.vrca`，而是无扩展名的 `__data`（每条缓存一个文件）；**AES-256 加密**，不是明文 bundle。
- 解密需要 **32 字节 key**：VRChat API 的文件分析响应里有 `encryptionKey`（Base64）；转成 64 位 hex 并把 `__data` 重命名成这个 hex，AssetRipper / `VRChatStudio` 才能认（§15）。
- 可配合：`ShayBox/VRC-LOG`（58★，记录缓存里的 avatar ID）、`VRC1337/VRChatStudio`（22★，MIT，解 VRChat 缓存）、`Escartem/AnimeStudio`（1275★，上游）。
- **不要用魔改客户端 / 注入 / 绕 EAC 的方式抓 key**：VRChat 2026 年已在封禁热上传与修改客户端（2026-03 / 2026-10 两次封禁潮，来源见 `ACKNOWLEDGEMENTS.md`），永久封号；只走官方 API 拿自己账号的 key。
- **即使解得开，也不要提取他人模型**（§4/§12 红线）。

## 10. 标准恢复流程：AssetRipper + FACS_Utilities（★ 权威 12 步）

> 来源：**FACS_Utilities 官方 Wiki「How to rip | Recovering a Project」**（权威一手），与三集外部教程（2024-06）互相印证；来源见 `ACKNOWLEDGEMENTS.md`。

### 10.1 准备

- Unity **2022.3.22f1** 工程 + **VRChat SDK（Base/Avatar）已装**（其它可能用到的脚本包也先装）。
- **AssetRipper 2.x**（本地**网页** GUI，不再是 1.x 的桌面窗口）：下载 Windows x64 版，解压到**无中文的路径**；双击 `AssetRipper.GUI.Free.exe` 会自动开浏览器；要只起服务用 `--headless --port 8123`（API 见 §14）。
- 导出目标文件夹必须**空** —— AssetRipper 会**清空**所选文件夹。

### 10.2 十二步

1. 取 `.vrca`（§9）。
2. AssetRipper → Open File 选 bundle → 载入后 Export All Files → 选空文件夹 → 等完成（大文件更久）。
3. 进 `ExportedProject/`：**要并入现有工程就只取 `Assets`**（`Packages` / `ProjectSettings` 只有想独立打开时才要；AssetRipper 2.0 会一并导出，实测 `ProjectVersion.txt` 就是 `2022.3.22f1c1`），可重命名（如 `MyRip`）。
4. 把 `Assets` 里的 **`Scripts` / `Shader` 改名成 `.Scripts` / `.Shader`**（点开头 = 隐藏，Unity 不编译、防报错；FACS 后面的功能要用到）。**★ 2026-10-19 实测依据**：
  AssetRipper 2.0 会导出 22 个 **dummy class**（`Assets/Scripts/VRCSDK3A/...`，注释明说「bundle 不含 dll / 代码」），并编译成与 VRChat SDK **同名**的 `VRC.SDK3A.dll` 等程序集 ——
  不先隐藏，导入带 SDK 的工程会撞名 / 编译冲突（§14）。
5. 新建或复用**带 VRChat SDK 的 Unity 工程**。
6. 看 `.Shader` 文件夹里用了哪些 shader，**按原版导入**（lilToon / Poiyomi / 商品自带 shader 等）。
7. 把 `MyRip` 整个拖进 `Assets/` 导入。
8. 材质球会 missing / 粉红 —— **正常**。用 `FACS Utils/Shader+Material/Easy Shader Reassign`：把 `Material` 文件夹拖进去 → 扫描出缺失 shader（**青色 = 坏/缺失，橙色 = 正常**）
  → 逐个或 `Group Shaders` 批量指定新 shader → `Apply Shaders`。完成后可在**文件资源管理器**里删掉 `.Shader`（Unity 不显示隐藏文件夹）。
9. `FACS Utils/Script/Fix Scripts` → **Automatic 模式** → `Target Folder` 选 rip 出来的 `Assets` 文件夹 → `Run Fix!`。没修全就导入原包（VPM / 商品包）后再跑一次。
   ★★ **坑：跑 Fix Scripts 之前，千万不要在 Project 里打开、甚至不要选中任何 Animator Controller** —— Unity 的 bug 会在打开时删掉所有 missing 的 `StateMachineBehaviour`，之后修不回来。
10. `Remove Missing Scripts`：头像用它的 prefab、地图用每个根 GameObject。
11. 缺组件时：`FACS Utils/Asset Bundle/FACS Load Bundle` 把原 bundle 载入编辑器（Edit / Play 都可），再用 `Copy Components` 从 bundle 组件复制**数值 / 文字属性**到工程组件。
  **不能复制资产引用**（material / mesh / texture），bundle 一卸载引用就丢（Unity 内置资源例外）。
12. `Fix Animations` 修 ripped 的 AnimationClip。**有 trimming 优化的资产（Poiyomi 等）先打开这些优化再跑**。

### 10.3 验收与还原度预期

- 官方 Wiki 与 B 站教程一致：**约 90% 已算很好了，不可能 100%**。
- 常需手动补的：**轮盘菜单、手势、首饰遮罩层**（上传检查会卡）、部分 shader 参数 / 材质表现。
- 验收：`read_console` 看 missing script / 材质报错 → Gesture Manager Play → `Build & Test`（上传仍要 GUI）。

### 10.4 其它可用工具（2026-10-19 增补）

| 工具 | 星/许可 | 用途 |
|---|---|---|
| **`FACS01-01/FACS_Utilities`** | 252★ / **OSL-3.0** / 2025-05 | 恢复套件：**Easy Shader Reassign / Fix Scripts / Remove Missing Scripts / Fix Animations / Copy Components / FACS Load Bundle / Material List / Remove Components / UPC Importer / Skinned Mesh Optimizer**。不需要工程装 VRChat SDK 也能跑。VPM 源：`https://facs01-01.github.io/FACS-VPM-Listing/`（或在 Releases 下 unitypackage） |
| **`ccd775/BlenderSafeAvatarFbxExporter`** | 21★ / **MIT** / 2026-09 | 把 **MA Manual Bake** 后的成品导出成 **Blender 兼容 FBX**（保留 pose / blendshape / 蒙皮权重 / 贴图；处理 bind pose 冲突、缩放补偿骨、重合控制点、NDMF NaNimation 删骨）；配套 `VRChatAvatarFBXOptimizer`。**恢复后想深改网格再回 Blender 用它**，别用 Unity 自带 FBX Exporter |
| **`Tliks/ModuleCreator`** | VPM 源 `https://tliks.github.io/vpm-repos/` | 从模型 / 素体上**提取任意部件**，自带对应骨骼、可用 AA 一键穿戴（改模辅助） |
| **`Dean2k/SARS`（A.R.C）** | 188★ / **许可证未声明** / 2024-08 停更 | Avatar Recovery Client：从缓存/日志找回 avatar 并尽量恢复成 unitypackage、含 hotswap；**偏 reupload/hotswap**，许可不明，谨慎 |

## 11. 术语：Ripping ≠ Reupload / Hotswap

- **Ripping（本册主题）**：从 bundle **提取资产、重建可编辑的 Unity 工程**。
- **Reupload / Hotswap**：直接改 / 重传 bundle，**不产出工程、不能编辑资产**（A.R.C 等工具也做这个）。
- 后者常被用于盗模；**只对自己的内容做**（§4）。

## 12. 风险与红线（补充）

- **`EkuzNetwork` 的「.vrca 自由上传」客户端**：**闭源、需授权密钥、往 VRChat 客户端里注入** —— 演示自称没开 EAC / 不修改客户端，但本质是绕过正常上传流程；**ToS 风险高 + 来源不明注入程序有安全风险，不推荐**。
- **`AvatarArc` 模型密码找回工具**（官网 `avatararc.win`）：只对自己上锁的模型用（说明称只分析 VRCA、不改文件）；闭源、效果不保证。
- 不要传播 / 分享从 bundle 提取出来的他人资产。

## 13. 来源（2026-10-19 增补）

- 视频来源见 `ACKNOWLEDGEMENTS.md`（本地测试资产包路线、缓存恢复、BlenderSafeAvatarFbxExporter、Module Creator、密码找回、闭源 .vrca 上传等）。
- GitHub：`FACS01-01/FACS_Utilities`（252★, OSL-3.0）、`AssetRipper/AssetRipper`（8659★）、`Dean2k/SARS`（188★）、`ccd775/BlenderSafeAvatarFbxExporter`（MIT, 21★）、`Tliks/ModuleCreator`（VPM）
- 本地实测：`%LOCALAPPDATA%Low\VRChat\VRChat\Avatars\Shipilka.vrca` = 18.36 MB，头 `UnityFS..5.x.`（明文）→ §9 路线 2 可用
- **状态（2026-10-19）**：FACS 12 步仍未走完；但 **AssetRipper 2.0 导出 + Unity 2022.3.22f1c1 无头打开已实测**（§14）。缓存解密、反盗模、VRCALoader / AvatarRecovery 的 Unity 内恢复仍未实测（§20.2）。
---

## 14. 本机实测：Shipilka.vrca → AssetRipper 2.0 → Unity 2022.3.22f1c1（2026-10-19）

> 这一节是**本工作区实测**，不是网络转述；样本、命令、数字都可复现（工具与中间产物在 `tools/tmp/ar/`，收尾会删）。

### 14.1 样本与工具

| 项 | 值 |
|---|---|
| 样本 | `<USERPROFILE>/AppData/LocalLow/VRChat/VRChat/Avatars/Shipilka.vrca`，**19,250,236 B（18.36 MB）**，2026-07-28 |
| 文件头 | `55 6E 69 74 79 46 53 ...` = `UnityFS.....5.x.`（明文 AssetBundle） |
| 提取器 | **AssetRipper 2.0.0**（2026-08-24 发布；win x64 zip 44,439,367 B；8,676★ / GPL-3.0 / 2026-10-10 仍有提交） |
| 运行 | `AssetRipper.GUI.Free.exe --headless --port 8123` → `http://127.0.0.1:8123/`（**2.0 是本地网页 GUI**；`--help` 只有 `--headless / --port / --log / --log-path / --version`，没有 CLI 导出参数） |
| 自动化 | 自带 `/openapi.json`；`POST /LoadFile`、`POST /Export/UnityProject`，`application/x-www-form-urlencoded` 字段 `path` |
| 目标 | `tools/tmp/ar/export1/`（空目录） |

**关键：AssetRipper 2.0 不再有 1.x 的桌面窗口。** 双击 exe 会自动打开浏览器；要脚本化就 `--headless --port <port>` 再走 HTTP API（外部演示也提到「AssetRipper 只有网页 UI」，原因相同）。

### 14.2 实测结果（数字）

- 载入：日志 `Game file 'Shipilka.vrca' has been found` / `Mixed game structure` / `Unknown scripting backend` / `Finished reading files`。
- 导出：`POST /Export/UnityProject` 后 **13.69 秒**完成（日志 `1024/1024` + `Finished exporting assets` + `Finished post-export`）。
- 产出：**2,098 个文件 / 104,664,353 B**，目录 `ExportedProject/`（Assets + Packages + ProjectSettings）+ `AuxiliaryFiles/`。
- 资产：**1,024 个**；按扩展名：663 `.anim`、151 `.png`、95 `.asset`、69 `.mat`、22 `.cs`、17 `.shader`、10 `.controller`、8 `.blendtree`、6 `.asmdef`、4 `.prefab`、2 `.state`、2 `.renderTexture`。
- `ProjectSettings/ProjectVersion.txt` = **`2022.3.22f1c1`**（与本机编辑器完全一致，不弹升级）。
- 导出设置（日志原文）：`BundledAssetsExportMode: DirectExport` / `ScriptExportMode: Hybrid` / `ShaderExportMode: Dummy` / `SpriteExportMode: Yaml` / 
  `TextExportMode: Parse` / `ImageExportFormat: Png` / `ExportUnreadableAssets: False`。
- ★ `ShaderExportMode: Dummy` 的含义：`Assets/Shader` 里的 17 个 `.shader` 是**占位文件**（保留名字/属性，不是原 shader 代码）；原 shader 必须装回 lilToon / Poiyomi / 商品包，**自定义 shader 源码拿不回来**。
- Unity 无头打开：`Unity.exe -batchmode -quit -nographics -projectPath <ExportedProject> -logFile ...` → 脚本编译 **13.965 s**、**0 个 `error CS`**、末尾 `Exiting batchmode successfully now!`、返回码 0。

### 14.3 三个必须知道的坑（实测）

1. **`Assets/Scripts` 是 22 个 dummy class，不是真脚本。** 例如 `Assets/Scripts/VRCSDK3A/VRC/SDK3/Avatars/Components/VRCAvatarDescriptor.cs`，注释原文：
  `Unity asset bundles and serialized files do not contain script information to decompile.` → 它只是占位类；Unity 编译后生成 **`VRC.SDK3A.dll`、
  `VRC.SDK3.Dynamics.PhysBone.dll`** 等与 VRChat SDK **同名**的程序集。→ 并入带 SDK 的工程前**必须**把 `Scripts` 改名 `.Scripts`（或删掉）。**本机实测（2026-10-19）**：
  把导出的 `Assets` 直接并入 `VRChatAvatarTemplate` → **6 条 `error CS1704`**（`VRC.SDK3.Dynamics.Constraint` / `Contact` / `PhysBone`、`VRCCore-Editor`、
  `VRCSDK3A`、`VRCSDKBase` 同名程序集已被导入）；把 `Scripts` / `Shader` 改名 `.Scripts` / `.Shader` 并清 `Library` 后重跑 → **0 个 `error CS`、退出码 0**。FACS 第 4 步就是这个原因。
2. **prefab 里有 19 个空 GUID。** 该 prefab 共 216 个 `m_Script`；`guid: 0000000000000000` 出现 19 次（真正 `m_Script: {fileID: 0}` 为 0）。
  `VRCAvatarDescriptor` 的 dummy GUID = `0b9762aa281b37d234b065bec67ed418`，在 prefab 中被引用 1 次 → Fix Scripts 要把它重指到真 SDK 的 GUID。
3. **没有 `.unity` 场景。** 导出的是「Assets + prefab」，不是可直接运行的场景；把根 prefab 拖进 Hierarchy 才能看模型。Controller 里仍能看到 `VRCFury Base/Action/Additive.controller` 这类名字（说明 VRCFury 在 bake 后留下了控制器，但编辑态组件已不在）。

### 14.4 结论

- **能重建工程骨架**：网格 / 贴图 / 材质 / 动画 / 控制器 / prefab / VRChat 白名单组件的序列化数据都在；`ProjectVersion` 与本机一致，独立打开无编译错误。
- **「能导入」≠「能上传」**：shader 缺失、dummy script、菜单 / 手势 / 遮罩要按 §19 逐项修；还原度按 FACS 官方口径约 90%。

## 15. 缓存文件是加密的：`__data` 与 `Avatars/*.vrca` 的区别（2026-10-19）

### 15.1 四条来源：缓存是加密的，其余三条是明文

| 来源 | 路径 | 是否加密 | 判据 / 取法 |
|---|---|---|---|
| **本地测试资产包（推荐）** | `%LOCALAPPDATA%Low/VRChat/VRChat/Avatars/<名字>.vrca` | **明文** `UnityFS` | 本机实测 `Shipilka.vrca` 头 16 字节 = `UnityFS.....5.x.`；直接喂 AssetRipper |
| 官方 File API | `GET /api/1/file/{fileId}/{versionId}` + `auth` cookie | 服务端存的是**未加密 bundle**（社区/视频口径；本机未测） | `unityPackages[].assetUrl`（§2） |
| 游戏缓存 | `Cache-WindowsPlayer/**/__data` | **AES-256 加密**（2025 起） | 需要 32 字节 key，见 §15.2 |
| VRChat 自带公共模型 | `VRChat_Data/StreamingAssets/Avatars/` | 明文（VRChat 自带示例） | 不是你的模型，别混 |

**本机现状（2026-10-19 复核）**：默认路径 `%LOCALAPPDATA%Low/.../Cache-WindowsPlayer` 不存在，但 `config.json` 把缓存指到了 **`<CACHE_ROOT>`**（本机历史取值为 `D:/VRChatCache/`，**仅为示例，换机器要改成你自己的目录**）——
该目录有 **292 个 `__data` / 30.46 GB**（`Cache-WindowsPlayer`）+ 4,928 个贴图缓存 / 1.02 GB；**每个 `__data` 前 6 字节都是 `UnityFS`，
但包体是加密的**（UnityPy 1.25.4 报 `The BundleFile is encrypted, but no key was provided!`），不能靠文件头判断可读。`Avatars` 只有 `Shipilka.vrca` 一个明文文件；
`LocalAvatarData/usr_*/avtr_*` 是**参数快照 JSON**（`eyeHeight` / `animationParameters`），不是模型文件（本机 38 个 `avtr_*`，37 个属于 `usr_fc85e017-...`）。

**平台 / 世界边界**：`unityPackages[]` 可能有 **PC 与 Quest 多条**，要按 `platform` 选（`standalonewindows` / `android`），别把 Quest bundle 当 PC 用。
世界 `.vrcw` 也能被 AssetRipper 打开、AvatarRecovery 也支持 World 项目，但**场景引用 / Udon 脚本的恢复难度更高**；本册以 avatar 为主，world 路线**未实测、未展开**。

### 15.2 缓存 `__data` 怎么解（文档 + 工具口径，**本机未实测**）

1. **拿 key**：`GET /api/1/analysis/{fileId}/{versionId}/standard`（或 `/security`）的响应里有 **`encryptionKey`**（Base64 字符串）。官方 API 文档的 `FileAnalysis` 结构里确实有这个字段（本机抓 `vrchat.community` 页面核对；**未实际调用**）。
2. **Base64 → Hex**：32 字节 key → 64 位十六进制。
3. **改名**：把 `__data` 重命名成这个 64 位 hex。
4. **喂工具**：AssetRipper 会按文件名识别 key 并解密；或 `VRC1337/VRChatStudio`（22★，MIT，fork 自 `Escartem/AnimeStudio` 1275★）加载后导出。
5. **VRCW（世界）**的 key 不是 `encryptionKey`，而是 `SHA256(id + secret)`（VRChatStudio README 给了示例）—— 本册只讨论 avatar。

**工具**：
- `VRC1337/VRChatStudio`：README 明确「VRCA 用 `encryptionKey`；本工具不提供提取 key 的方法」；AES-256。
- BOOTH「VRCA Key Converter & GUI」（`version_fictus_1`，1,000 JPY）：只做 Base64→hex + 复制文件，**不含解密逻辑**；说明「key 可从 VRChat API 取得」。
- `Escartem/AnimeStudio`：通用 Unity 加密资源提取，VRChatStudio 的上游。

**红线**：**不要用魔改客户端 / 注入 / 绕 EAC 的方式抓 key**。VRChat 2026 年已封禁「热上传」与「修改客户端」，而且是**永久封号**（2026-03-27 / 2026-10-16 两次封禁潮）；买低价模型也可能被牵连。只走官方 API 拿**自己账号**的 key。

### 15.3 本机实测（2026-11-04）：加密比例、key 提取失败、判据

样本：`%LOCALAPPDATA%Low/VRChat/VRChat/Cache-WindowsPlayer`，**53 个 `__data` / 5.5 GB**。

| 项 | 实测结果 |
|---|---|
| 明文包 | **8 个**，全部 ≤ 1.2 MB，内容为 warpeffect / 道具特效（`WarpEffect_Confetti` 类），**没有 avatar** |
| 加密包 | **45 个**，≥ 2 MB，**含全部 avatar 与世界** |
| 文件头误导 | 加密包前 16 字节同样是 `UnityFS.....5.x.` + Unity 版本串 → **不能靠文件头判断可否解密**，必须实际 `UnityPy.load()` |
| UnityPy 报错 | `The BundleFile is encrypted, but no key was provided!`，并给出该包自己的 `key_sig` / `data_sig` |
| `brute_force_key` 对 `global-metadata.dat` | **失败返回 None**。原因：VRChat 的 metadata 已被加密，头 4 字节 `3a 08 77 f3`，**不是标准 IL2CPP 的 `AF 1B B1 FA`**，正则 `(?=(\w{16}))` 扫不到有效 16 字节 key |
| AssetRipper 2.0 读加密包 | 不抛异常，日志只写 `Mixed game structure has been found` → `could not find any valid assets` → `Finished reading files`，**静默产出空工程**。判据：`/Assets/*` 为空而 `/LoadFile` 仍返回 302 |

**结论**：静态文件路线拿不到 key（metadata 被保护）。可选只有两条 —— ① 官方 API 取**自己账号**的明文 bundle（`unityPackages[].assetUrl`，服务端存的是未加密包）；② 运行时内存 dump 再按同样的 `AES_ECB(key).encrypt(key_sig) XOR data_sig == "#$unity3dchina!@"` 判据爆破（**未实测**，且涉 EAC）。

### 15.4 把缓存迁到非系统盘：官方 `config.json` 方案（本机实测 2026-11-04）

官方依据：<https://help.vrchat.com/hc/en-us/articles/1500004572821> + <https://docs.vrchat.com/docs/configuration-file>（Cache Settings → Location）。

**官方明确禁止 symlink**：`Please do not use symbolic links (symlinks) for this, as you may encounter issues loading content`。不要再给「用 mklink /J 做目录联接」这类方案。

配置（**永久生效，每次启动读取**）：

| 项 | 值 |
|---|---|
| 文件路径 | `%LocalAppdata%Low\VRChat\VRChat\config.json`（**必须叫 `config.json`**，默认不存在需新建） |
| 字段 | `"cache_directory" : "<CACHE_ROOT>/"`（本机历史示例为 `D:/VRChatCache/`） |
| 路径写法 | 用 `/` 或 `\\`；**不能用单个 `\`**（JSON 转义符） |
| 生效范围 | 仅缓存迁走；**config.json 与 output_log 仍留在 C 盘默认位置** |
| 可选配套 | `cache_size`（GB，≥30，默认 30）、`cache_expiry_delay`（天，≥30，默认 30） |

最小可用内容（UTF-8 无 BOM）：

```json
{
  "cache_directory": "<CACHE_ROOT>/"
}
```

**目录结构**：`cache_directory` 下保持与默认目录同名的三个子目录，即 `<cache_directory>/Cache-WindowsPlayer/`、`/TextureDiskCache-WindowsPlayer/`、`/HTTPCache-WindowsPlayer/`。迁移时整目录搬过去，旧缓存可继续命中，不必重下。

**本机实测步骤与数字**：

1. `New-Item -ItemType Directory -Path <CACHE_ROOT>`
2. `robocopy <src> <dst> /E /MOVE /R:1 /W:1 /NFL /NDL /NP` 逐目录搬：Cache 5.473 GB / 108 文件（0 失败）、Texture 162.87 MB / 447 文件、HTTPCache 空
3. 写 `config.json`
4. 清理 C 盘残留

**坑**：`robocopy /MOVE` 报 `错误 32 无法删除源文件`，是本工作区的 **AssetRipper 后台服务持有 `__data` 句柄**。先 `taskkill /F /IM AssetRipper.GUI.Free.exe` 再删残留。凡是「删不掉缓存」先查有没有 AssetRipper / VRCX / 杀软在占用。

**迁移后验收锚点**（重启 VRChat 后看日志）：

- 日志 `[AssetBundleDownloadManager]` 行：默认是 `Using default content cache directory.`；配置生效后应变为自定义路径（本机 2026-11-04 待启动验证，标 **NOT_RUN**）
- `[TextureManagement]` 的 warning 路径应从 `...\LocalLow\VRChat\VRChat\TextureDiskCache-WindowsPlayer\...` 变成 `<CACHE_ROOT>\TextureDiskCache-WindowsPlayer\...`
- 直接看 `<CACHE_ROOT>\Cache-WindowsPlayer` 是否新增 `<hash>/<id>/__data`

## 16. 为什么「恢复出来是坏的」：反盗模 / 加密保护（2026-10-19）

### 16.1 保护手段（按机制分）

| 工具 / 方案 | 星 / 许可 | 机制 | 对恢复的影响 |
|---|---|---|---|
| **`PlagueVRC/AntiRip`（Kanna Protecc）** | 232★ / 许可未声明 / 2026-10-12 | **随机化全部顶点**写入磁盘 + 自定义 shader 用 **32-bit key** 在游戏内还原；destructive 改 Basis 层；顺便混淆名称/结构 | 解包出来顶点是乱的；没有 owner 的 key / config 文件**不可逆**；只支持 PC；仅 Poiyomi / Unity-Chan / Suwano / UMXS / GTAATune 等 shader |
| `rygo6/GTAvaCrypt` | 92★ / MIT / 2023-05 | Kanna 的前身（README 说项目从 rygo6 转手给 PlagueVRC） | 同上，老项目 |
| `Shell4026/ShellProtector` | 64★ / MIT / 2026-10-17 | **贴图加密** | 解包贴图花 / 纯色 / 错位 |
| `edonamea/MeshProtect` | 5★ / MIT / 2026-10-07 | mesh 保护 | 顶点/网格异常 |
| `dwgx/Obfuscatevrcavatar`（OVA） | 1★ / 2026-09 | NDMF/VPM blue-team 保护 | 名称/结构不可读 |
| `Ess-Ka/EsskaAV3Obfuscator` | 42★ / MIT / 2025-01 | 参数 / 名称混淆 | 参数名、层级不可读 |
| `cocokoishi/AvatarObfuscator` | 8★ / MIT / v0.4.9 / 2026-08 | NDMF ≥1.6；把所有可读名称换成同形字 `Ì Í Î Ï`，**不改场景原件**、无损、不影响性能评级 | 层级/参数全是同形字，能看出结构但没法读 |
| **DexProtect**（Dextro）/ **ajisaiflow Anti-Ripping** | 商业（Gumroad / 官网） | 多层加密 + 客户端解密（NDMF 管线） | 商业方案，本机未验证 |

### 16.2 症状 → 判断

| 现象 | 大概率 | 处理 |
|---|---|---|
| 模型顶点乱飞 / 网格像噪声 / 形状炸开 | Kanna / GTAvaCrypt | 找 owner 的 key / config；没有就**不要承诺能还原** |
| 贴图花、纯色、错位 | ShellProtector / 贴图加密 | 同上；原贴图可能只存在作者工程里 |
| 名称全是 `ÌÍÎÏ...`、层级不可读 | AvatarObfuscator / Esska | 结构可恢复，但名称无意义；改模要自己重命名 |
| 菜单 / 参数名乱码 | 参数混淆 | 能跑但不可维护；按 §19 重建菜单 |
| 模型只有作者自己能看，别人看是乱的 | Kanna 的 shader key 未同步 | 不是「坏了」，是保护生效 |

**结论**：保护不是「加密文件」，而是**破坏资产语义**（顶点、贴图、名称）。恢复工具只能拿到被破坏后的数据；遇到这类模型，先问「有没有 owner 的 key/config」，不要先跑 AssetRipper。

## 17. Ripping vs Hotswap：为什么「解包」不等于「能重新上传」（2026-10-19）

- **服务端**：VRCA 以**未加密**形式存储；加密发生在**客户端缓存**阶段（§15 口径）。所以魔改客户端 / 读内存能拿到明文，但这是违规的。
- **官方 API**：只能访问**自己账号**的资源（`auth` cookie；`unityPackages[].assetUrl`）。
- **解包 ≠ 重传**：复杂模型直接解包再上传成功率极低（依赖、脚本、GUID、菜单都断了）；社区更常见的是**只提取部分资产**（衣服、头发、贴图）或看 FX/动画逻辑。
- **Hotswap（热替换 / 热上传）**：拦截 SDK 上传 → 从 Unity 生成的缓存 bundle 里读 `blueprintId` / `prefabId` / Unity 版本等验证信息 → 写进**盗来的 VRCA** → 替换缓存 → 恢复上传。**不需要工程、不能编辑资产**，本质是绕过正常上传。
- **工具**：`Sakura1618/VRCA-Toolbox`（17★，**无许可证**，含 BatchHotSwap）、`EkuzNetwork` 的「.vrca 自由上传」（闭源 / 需授权密钥 / 注入客户端，**不推荐**）。
- **风险**：2026-03 起 VRChat 封禁热上传 / 修改客户端（2026-03），2026-10 仍在封（2026-10）；**永久封号**，且买低价模型可能被牵连。
- **术语**：**Ripping** = 提取资产、重建可编辑工程（本册主题）；**Reupload / Hotswap** = 直接换 bundle、不产出工程（§11）。

## 18. 工具全景（2026-10-19 核对）

### 18.1 提取 / 恢复

| 工具 | 版本 / 星 / 许可 | 状态与用途 |
|---|---|---|
| **`AssetRipper/AssetRipper`** | **2.0.0** / 8,676★ / GPL-3.0 / 2026-08-24 | **主力**。本地网页 GUI（§14）；支持 Unity 3.5–6000.4；导出 Unity 工程；自带 OpenAPI，可脚本化 |
| **`Nickel-JP/avatar-recovery-unity`** | **1.2.22** / 11★ / 自定义许可 / 2026-10-12 | VPM 包；**内置 AssetRipper 1.3.14**；Unity 2022.3.22f1 + **VRChat SDK Base >=3.10.5 <3.11.0**；一键恢复 + Shader Lists + Missing Script + Expression Menu；**1.3.x 因可靠性问题已下架，1.2.22 是当前稳定版** |
| **`200Tigersbloxed/dVRC`** | **1.1.2** / 70★ / GPL-3.0 / 2026-01-06 | Unity 插件；修了 AssetRipper 变更导致的通信问题；装过 AR 要点 **Reinstall AssetRipper**；只恢复自己上传的资产 |
| **`cocokoishi/VRCALoader`** | 11★ / AGPL-3.0 / 2026-09-05 | 用官方 SDK 登录下载**自己账号**的 VRCA/VRCW → **内存加载预览**（不可上传）→ **Extract All Assets&Controller** 走 AssetRipper + **Reference Remapper** 修 controller/shader/script 引用；接近一键 |
| **`FACS01-01/FACS_Utilities`** | **25.05.27** / 252★ / OSL-3.0 / 2025-05-27 | 恢复套件：Easy Shader Reassign / Fix Scripts / Remove Missing Scripts / Fix Animations / Copy Components / FACS Load Bundle / Material List / Remove Components / UPC Importer / Skinned Mesh Optimizer；不要求工程装 VRChat SDK |
| **`Dean2k/SARS`（A.R.C）** | 188★ / 许可未声明 / 2024-08 停更 | Avatar Recovery Client；恢复成 unitypackage + hotswap；**偏 reupload，许可不明，谨慎** |
| `Sakura1618/VRCA-Toolbox` | 17★ / **无许可证** / 2026-04 | VRCA 下载器 / 渲染器 / 管理器 / **BatchHotSwap**；作者称是旧版「Vibe 产物」，重构版在收尾 |
| `TeaClearInkII/Unity-AnimationBindingFixer` | 1★ / MIT / 2026-05 | 修 AnimationClip 的 missing binding；恢复后动画路径断掉时用 |
| `ccd775/BlenderSafeAvatarFbxExporter` | 21★ / MIT / 2026-09 | MA Manual Bake 成品 → Blender 兼容 FBX；恢复后想深改网格用 |
| `Tliks/ModuleCreator` | VPM | 从模型/素体提取任意部件（含骨骼） |

### 18.2 通用资产工具（不是专用恢复器）

| 工具 | 版本 / 星 / 许可 | 用途 |
|---|---|---|
| `Perfare/AssetStudio` | 15,588★ / MIT / 2022-12 停更 | 老牌浏览/导出；**已停更**，新版本问题多 |
| `aelurum/AssetStudio`（AssetStudioMod） | **v0.19.0** / 2,130★ / MIT / 2025-09 | 社区维护版；浏览 + 导出网格/贴图，工程化弱于 AssetRipper |
| `nesrak1/UABEA` | 2,459★ / MIT / 2026-05 | AssetBundle 编辑（替换贴图/文本）；**不是**工程恢复 |
| `nesrak1/UABEANext` | 424★ / MIT / 2026-10 | UABEA 的新一代（AssetsTools.NET）；研究/改 bundle |
| `K0lb3/UnityPy` | 1,497★ / MIT / 2026-10 | Python 解析/导出/改资产；适合批处理 |
| `Escartem/AnimeStudio` | 1,275★ / MIT / 2026-10 | 通用 Unity 加密资源提取（VRChatStudio 上游） |

### 18.3 下载 / 缓存 / 保护

| 工具 | 数据 | 用途 |
|---|---|---|
| `Null-K/VRChatVRCADownloader` | 67★ / WTFPL / 2026-08 | 登录自己账号，列出并下载 Avatar 文件 |
| `cocokoishi/VRCALoader` | 11★ / AGPL-3.0 | 官方 SDK 登录 + 下载自己的 VRCA/VRCW（§18.1） |
| `ShayBox/VRC-LOG` | 58★ / MIT / 2026-10 | 记录缓存里的 avatar ID |
| `VRC1337/VRChatStudio` | 22★ / MIT / 2025-11 | 解 VRChat 缓存（AES） |
| `PlagueVRC/AntiRip`（Kanna） | 232★ / 许可未声明 | 反盗模（§16） |
| `Shell4026/ShellProtector` | 64★ / MIT / 3.0.3 | 贴图加密 |
| `Ess-Ka/EsskaAV3Obfuscator` | 42★ / MIT | 参数/名称混淆 |
| `cocokoishi/AvatarObfuscator` | 8★ / MIT / v0.4.9 | NDMF 同形字混淆 |
| `TrippleAWap/Snorlax` | 9★ / 未声明 / 2025-06 | 扫缓存找 public avatar + force clone（**高风险，不建议**） |

## 19. 恢复后修复清单（按症状，2026-10-19 整理）

> 顺序建议：先「能编译」→ 再「能显示」→ 再「能跑」→ 最后「能上传」。FACS 的 12 步见 §10.2，这里只补判据与新增项。

1. **编译冲突 / 同名类**：把 `Assets/Scripts` 改名 `.Scripts`、`Assets/Shader` 改名 `.Shader`（§10.2 第 4 步；§14.3 实测：不改 → 6 × `CS1704`，改后 0 error）。
  **不要**在 Fix Scripts 前打开或选中任何 AnimatorController（Unity bug 会删掉 missing StateMachineBehaviour，修不回来）。
2. **材质粉 / shader 缺失**：装回 lilToon / Poiyomi / 商品 shader → `FACS Utils/Shader+Material/Easy Shader Reassign`（青色=缺失、橙色=正常）；
  AvatarRecovery 用 `Open Shader List` + preset 逐个补。**`Assets/Shader` 是 `ShaderExportMode: Dummy` 的占位文件，不要指望它**（§14.2）。
3. **Missing Script**：`FACS Utils/Script/Fix Scripts`（Automatic + Target Folder）→ 没修全就导入原包（VPM / 商品包）再跑 → `Remove Missing Scripts`（头像用 prefab 根）。
4. **动画丢绑定 / 路径断了**：`FACS Utils/Animation/Fix Animations`；仍缺就用 `Unity-AnimationBindingFixer` 按对象路径重绑。
5. **轮盘菜单 / 手势 / 参数**：恢复出来的是 **baked 结果**（AvatarRecovery 能恢复 Expression Menu，但不是 MA 编辑态）；要可维护就按 `03-菜单与参数` 重建 MA 菜单。
6. **PhysBone / Contact / 约束**：Fix Scripts 后确认组件类型正确、参数在；进 Gesture Manager Play 看动骨/接触是否生效，别只看 Inspector。
7. **部位消失 / 细胳膊细腿**：先查形态键与 **MA Shape Changer / BlendShape Sync**；再查 FX 层权重（逐层设 0 二分）与 Write Defaults（Avatars 3.0 Manager 的 WD 页签）——四步法见外部教程（来源见 `ACKNOWLEDGEMENTS.md`）。
8. **颜色 / 脖子接缝**：NoNlab 双模颜色同步模块（§7）。
9. **验收**：Console 0 error → Gesture Manager Play（手势/菜单/开关/PhysBone）→ Build & Test（上传仍要 GUI）；还原度约 90%，轮盘 / 手势 / 首饰遮罩常要手补（§10.3）。

## 20. 来源与状态（2026-10-19）

### 20.1 本机实测（可复现）

- `Shipilka.vrca` 19,250,236 B / 2026-07-28 / 头 `UnityFS.....5.x.`（§9.1）
- AssetRipper 2.0.0 载入 + 导出（13.69 s / 1,024 资产 / 2,098 文件 / 104,664,353 B）+ Unity 2022.3.22f1c1 无头打开（0 个 `error CS` / 编译 13.965 s / 返回码 0）（§14）
- dummy script / 空 GUID / 无场景三个坑（§14.3）
- **CS1704 冲突复现与修复**：导出 `Assets` 直接并入 SDK 模板 → 6 × `error CS1704`；`Scripts` / `Shader` 改名 `.Scripts` / `.Shader` + 清 `Library` → **0 error / exit 0**（§14.3）

### 20.2 只核对了文档 / 社区，未实测

- 缓存 `__data` 的 AES 解密与 `encryptionKey` 实际调用（本机没有 `Cache-WindowsPlayer`；§15）
- 反盗模各工具的实际效果（§16）；Kanna 的 key/config 恢复流程
- FACS 12 步的完整走通（只走到 AssetRipper 导出 + Unity 打开；§10）
- AvatarRecovery / dVRC / VRCALoader 在 Unity 里的实际恢复（§18）
- Hotswap 的具体工具链（§17；**不建议实操**）

### 20.3 来源

- 视频来源见 `ACKNOWLEDGEMENTS.md`（防盗模 / 封号潮 / AssetRipper / 绑定修复等演示）。
- GitHub：`AssetRipper/AssetRipper`（2.0.0）、`Nickel-JP/avatar-recovery-unity`（1.2.22）、`200Tigersbloxed/dVRC`（1.1.2）、`cocokoishi/VRCALoader`、
  `FACS01-01/FACS_Utilities`（25.05.27）、`PlagueVRC/AntiRip`、`Shell4026/ShellProtector`、`Ess-Ka/EsskaAV3Obfuscator`、`cocokoishi/AvatarObfuscator`、
  `VRC1337/VRChatStudio`、`Escartem/AnimeStudio`、`nesrak1/UABEANext`、`K0lb3/UnityPy`、`TeaClearInkII/Unity-AnimationBindingFixer`
- 文档：`vrchat.community/reference/get-file-analysis-standard`（`FileAnalysis.encryptionKey`）、
  `vrchat.community/reference/get-avatar`（`unityPackages[].assetUrl`）、FACS Wiki「How to rip | Recovering a Project」、
  AvatarRecovery README（Technical Specifications）、AssetRipper README（Unity 3.5–6000.4）
- **状态**：§14 已实测；§15–§19 的方法来自官方文档 + 工具 README + B 站，标「未实测」的不要当结论。

## 21. 从 `.vrca` 到可用 SDK 工程：本机实测修复流程（2026-10-19）

> 样本 **Lilleo狼**；最终工程 `<VRCHAT_ROOT>\projects\LilleoWolf_SDK`。§14 只是 AssetRipper 导出「原料」；要变成能打开、能编译、组件/材质/动画/菜单都正常的 SDK 工程，还要过下面四关。

### 21.1 步骤

1. 复制 `VRChatAvatarTemplate`（含 VRChat SDK 3.10.5 / lilToon / Poiyomi / Gesture Manager），把导出的 `Assets` 放进 `Assets/Recovered`。
2. **在打开 Unity 之前**把 `Recovered/Scripts` 改名 `.Scripts`、`Recovered/Shader` 改名 `.Shader`（避免 §14.3 的 6 条 `CS1704`）。
3. **脚本重指**（Editor C#）：
   - 扫 hidden `.Scripts/*.cs` 的 `namespace`/`class` 与 `.meta` GUID；
   - 在 SDK DLL 上 `AssetDatabase.LoadAllAssetsAtPath(dll)` 取 `MonoScript`，按 `GetClass().FullName` 建表；
   - `AssetDatabase.TryGetGUIDAndLocalFileIdentifier(mono, out guid, out fileID)`；
   - 把 prefab/asset 的 `m_Script: {fileID:..., guid:..., type:3}` 换成真实 `guid/fileID`。
4. **Shader 重指**：解析 hidden `.Shader/*.shader` 的 `Shader "name"` + meta GUID；`Shader.Find(name)`（或按 `t:Shader` 找同名）→ `TryGetGUIDAndLocalFileIdentifier` → 替换材质 `m_Shader`；找不到的用 fallback。
5. **动画路径反混淆（关键）**：
   - 路径形如 `path_0x<6~8位hex>_<随机后缀>`；hex 是 `Animator.StringToHash(原始路径)`，**可能是 7 位**（前导 0 省略）；
   - 把 prefab 每个 Transform 的完整路径算 `Animator.StringToHash(path).ToString("X8")`，建 hash→path 表；
   - 直接改 `.anim` YAML 的 `path:` 行（比 `AnimationUtility.SetEditorCurve` 快很多）；
   - 剩余无法映射的曲线用 `AnimationUtility.SetEditorCurve/SetObjectReferenceCurve(..., null)` 删除。
6. 验收：0 missing component、0 missing animation binding、0 坏材质、菜单/参数可读、渲染 Preview。

### 21.2 本机结果（Lilleo狼）

- 19/19 脚本重指；11 个 shader（9 直接 + 2 fallback）；**510 个动画路径映射、5,918 行 `path:` 改写**；26 个路径 / **107 条曲线**删除；最终 **600 clip / 0 missing binding**。
- **1,843 个组件 / 0 missing**；**82 材质 / 0 bad**；**145 参数 / 4 个一级子菜单**；77 PhysBone、21 PhysBoneCollider、20 ContactReceiver、5 ContactSender、6 Station。
- 工程：`<VRCHAT_ROOT>\projects\LilleoWolf_SDK`；场景 `Assets/Scenes/LilleoWolf.unity`；预览 `Preview.png`。
- **未跑** Gesture Manager Play / Build & Test（需要 GUI / 客户端）。

### 21.3 判据与坑

- 先隐藏 dummy 脚本再打开，否则 `CS1704`；脚本重指必须用 **DLL 的 MonoScript**（`fileID` 不是 11500000），只换 GUID 会错。
- 动画路径 hex 可能是 7 位：正则用 `{6,8}` 并 `PadLeft(8,'0')`，否则会漏 28 个路径。
- fallback shader 会让个别道具视觉略不同（本机：`Aoinu Works/Reflex Sight/Color` → `lilToon`；`LightShaft` → `Hidden/lilToonTransparent`），能接受再用。
- 删除缺失曲线只对「对象确实不在 bundle」的路径做，不要盲删；本机剩余 26 个路径都是这类。




## §22 第 126 轮实测：AssetRipper 2.0 网页版导出两只已上传模型（本工作区 2026-10-29）

### 22.1 工具与启动

- `AssetRipper_win_x64.zip`（GitHub 2.0.0，44,439,367 B）解压后是 **`AssetRipper.GUI.Free.exe`（单文件 130 MB）+ capstone.dll**；
  没有桌面窗口，是本地 Web GUI：
  ```powershell
  AssetRipper.GUI.Free.exe --headless --port 7788 --log-path <VRCHAT_ROOT>\tools\tmp\round126\ar.log
  ```
- API（`/openapi.json` 可查）：`POST /LoadFolder`、`POST /Export/UnityProject`，参数都是 **form 字段**，不是 query：
  ```bash
  curl -X POST --data-urlencode "Path=<VRCHAT_ROOT>/.../赛博猫猫.vrca" http://127.0.0.1:7788/LoadFolder
  curl -X POST --data-urlencode "Path=<VRCHAT_ROOT>/.../rip/cat" --data-urlencode "CreateSubfolder=false" \
       http://127.0.0.1:7788/Export/UnityProject
  ```
  `/LoadFile` 是「弹出 Windows 文件对话框」，不能用在自动化里；直接 POST 会 302 + 日志 `Incorrect Content-Type`。
- 实测：猫 84.2 MB → load 27 s / export 62.6 s / 2,730 文件 / 1,010 MB；狗 86.4 MB → load 27 s / export 59.2 s / 2,711 文件 / 933 MB。

### 22.2 并进工程

1. 把 `ExportedProject/Assets/*` 复制到 `Assets/20_逆向/<名字>/`；**`Scripts` 改名为 `.Scripts`**（隐藏，避免 CS1704），
   **`Shader` 保持可见**（本次导出目录里只有 `.shader`，没有 `.cs`；让自定义 shader 编译，材质才能保持原样）。
2. 脚本重指（同 §21）：索引 `Packages` 下所有 MonoScript（含 DLL 的，fileID 不是 11500000），
   按 `.Scripts/**/*.cs` 的 `namespace + class` 匹配 → 改写 prefab/asset 里的 `m_Script`。本次 24 个类型全部命中（含 `VF.Component.VRCFuryGlobalCollider`）。
3. 动画路径反混淆（同 §21）：`path_0x<6~8位hex>_xxx` → `Animator.StringToHash(完整路径)`。
   本次猫修 **4,738** 条、狗修 **6,973** 条（对象引用曲线 15 条），剩余 72 / 105 条（对象在 bundle 里已不存在，保留为死曲线）。
4. 材质空引用：狗的 11 个特效 renderer `sharedMaterials=null`（`coherance_beam*`、`bullet`、`spark`、`hiteffect_*` 等），
   按名字映射到导出包里已有的 `cc_beam.mat` / `sf02_bullet.mat` / `sf02_spark.mat` / `sf02_laser_hit.mat` / `sf02_effect_charge.mat`。

### 22.3 验收（脚本 + 渲染）

| 项目 | 赛博猫猫 | 赛博小狗 |
|---|---|---|
| transforms | 1,657 | 1,616 |
| SMR | 97 | 72 |
| missing scripts | 0 | 0 |
| bad materials | 0 | 0（修前 11） |
| PhysBone/Collider | 167 | 119 |
| 菜单控件 | 225 | 235 |
| 菜单参数未定义 | 0 | 0 |
| ExpressionParameters | 144 | 158 |

场景：`Assets/00_场景/逆向角色_赛博猫狗.unity`（两只并排，±1.1 m），10 视角渲染正常；粉/黑故障条与白色面板是模型自带的赛博 HUD/特效，
编辑态（无动画）会同时显示，进游戏由 FX 层控制。

> 注意：导出 prefab 的 `PipelineManager.blueprintId` 保留原 id，可用来核对来源；不要误上传。

## 23. 本机实测（2026-11-04）：官方 API → AssetRipper → FACS → shader 重指 → 渲染

样本：`avtr_4f633989-54cd-48a4-905d-b1c92e5bf4ab`「沐梓溪（小狼版）」（自有，private，version 51）。
产出工程：`<VRCHAT_ROOT>/projects/_rips/avtr_4f633989/RipProject`。

### 23.1 ★ Unity 许可证：Personal 手动激活已废弃，只能 Hub 在线

**这是本节最容易被卡住的一步，先看这里。**

| 结论 | 依据 |
|---|---|
| `license.unity3d.com/manual` 上传 `.alf` 换 `.ulf` 的路线，**Personal 已不可用** | 页面原文：`Unity no longer supports manual activation of Personal licenses.` 上传框只剩 Plus/Pro 序列号通道（本机 2026-11-04 实拍） |
| 编辑器若是 `2022.3.22f1c1`（`china_unity/release` 分支） | 版本号带 `c1` 即中国版；`Unity.exe` 的 ProductVersion 会写 `2022.3.22f1c1_xxxx` |
| 判据：`No valid Unity Editor license found` + 日志 `No ULF license found` / `Token not found in cache` | `%LOCALAPPDATA%\..\ProgramData\Unity\config\user-services-config.json` 为 `{}`（2 字节）即登录态没传给 licensing client |
| **在 Hub 里登录 ≠ 拿到许可证** | 必须再执行 `Settings → Licenses → Add license / Get a free personal license`，成功后 `C:\ProgramData\Unity\Unity_lic.ulf` 才会出现 |
| 许可证可验证字段 | `.ulf` 里 `LicenseVersion` / `StartDate` / `SerialMasked` / `MachineID`；`MachineID` 应与 `.alf` 的 `MachineID` 一致 |

本机实际结果：`C:\ProgramData\Unity\Unity_lic.ulf`（2645 B，`SerialMasked F4-WE6H-…-XXXX`，`LicenseVersion 6.x`）。**注意 ULF 落在 `ProgramData\Unity\` 根下而不是 `licenses\` 子目录，编辑器同样认。**

`-createManualActivationFile` 仍能生成 `.alf`（`SupportedLicenseVersion 6.x`），但只对 Plus/Pro 有意义。

### 23.2 官方 API 取包：必须按 `variant` 挑

`GET /api/1/avatars/{id}` 的 `unityPackages[]` 里同一平台会有多条，**按 `variant` 区分**：

| variant | 含义 | 能否用于逆向 |
|---|---|---|
| `standard` | 模型本体 | ✅ **只用这个** |
| `security` | 安全扫描变体（平台同为 standalonewindows） | ❌ |
| `impostor` | 3D 缩略/impostor 模型（PC/Android/iOS 各一份） | ❌ |

本机样本共 5 条（2×standalonewindows + android + ios + 1×standalonewindows/impostor），只有 1 条 `variant=standard`。
`assetUrl` 形如 `/api/1/file/{fileId}/{version}/file`，带 `Cookie: auth=…; twoFactorAuth=…` 直接下载即得**明文 `UnityFS`**（21,368,695 B，头 `UnityFS…..5.x.x.2022.3.22f1c1`）。

### 23.3 ★ VRCFury 烘焙产物在 bundle 里是完整的

与 `SKILL.md` 里「不要从 `.vrca` 反查 animator 层」那条并不矛盾——那条针对的是**未用 VRCFury 烘焙**的普通上传（descriptor 的 `runtimeAnimatorController == null`，控制器由 VRChat 运行时装配）。

本次这个 avatar 用了 VRCFury，烘焙时把控制器/菜单/参数**落成了独立资产**并写进 descriptor，所以导出后：

```
baseAnimationLayers[0..4] = Base/Additive/Gesture/Action/FX   → 全部指向 'VRCFury Base'…'VRCFury FX'
specialAnimationLayers[0..2] = Sitting/TPose/IKPose           → 全部有 controller
Gesture 层 mask = 'Mask for [VF32] Base Mask'
expressionParameters = 'VRCFury Params'   expressionsMenu = 'VRCFury Menu'
Assets 内 AnimatorController 10 个 / AvatarMask 1 个 / VRCExpressionsMenu 28 个
ViewPosition = (0.00, 0.68, 0.07)
```

**判据**：先 `CheckAvatar` 打印 descriptor 的 9 个层与两个菜单引用；只要 `ctrl` 不是 `NULL`，菜单和参数就能直接编辑，不必从零重建。

### 23.4 编译坑（两个，都会让工程打不开）

1. **`com.unity.test-framework` 缺失** → VRChat SDK 自带自测脚本报 `CS0246 TestAttribute/Test`（48 条）。补包后仍有 16 条 `NUnit` 报错（SDK 的 asmdef 没引用 TestRunner，不值得改包）。
   **做法**：把 `Packages/com.vrchat.base/Editor/VRCSDK/VTP` 与 `…/Dependencies/VRChat/Tests` 改名加前缀点（`.VTP` / `.Tests`），连同 `.meta` 一起移。这两处是 SDK 自测（`MockNetworkStream` 只被 `VTPTests` 引用），删掉对功能零影响。
2. **移动源文件后报 `CS2001: Source file … could not be found`** → Bee 编译缓存还记着旧路径。删 `Library/Bee` 与 `Library/ScriptAssemblies` 重编译即可（不要删整个 Library）。

### 23.5 缺失脚本修复：FACS 反射调用（batchmode 可用）

FACS 的入口是 `internal class FixScripts : EditorWindow` + `[MenuItem]`，batchmode 点不了菜单。但它真正的实现是 `private static void RunFix()`，**用反射直接调**：

```csharp
var t = FindType("FACS01.Utilities.FixScripts");           // 包名是 com.facs01.facs_utilities
t.GetField("selectedFolder", NonPublic|Static).SetValue(null, AssetDatabase.LoadAssetAtPath<DefaultAsset>("Assets")); // 不设会弹 DisplayDialog
t.GetField("manual", NonPublic|Static).SetValue(null, false);  // Automatic 模式
t.GetMethod("RunFix", NonPublic|Static).Invoke(null, null);
```

`Automatic` 模式的判据：它用 `Directory.EnumerateDirectories(rootFolder, ".Scripts", AllDirectories)` 找 `.Scripts` 目录，再按 `namespace + class` 匹配真实脚本 —— **所以 `.Scripts` 必须保留到修复完再删**。

本机结果：`prefabs=3 components=1223 missing=211` → `missing=0`（29 秒）。

### 23.6 Shader 重指：lilToon 1.x 占位名 → 2.1.8 实际文件（可离线做）

AssetRipper 的 `ShaderExportMode: Dummy` 只产占位 shader，材质引用它们会变粉。lilToon 2.x 已把文件改名为 `lts*`，**不能按文件名直接对**，要用语义映射：

| 导出占位名 | lilToon 2.1.8 实际 |
|---|---|
| `lilToon` | `lts` |
| `Hidden_lilToonCutout` / `…CutoutOutline` | `lts_cutout` / `lts_cutout_o` |
| `Hidden_lilToonOutline` | `lts_o` |
| `Hidden_lilToonTransparent` / `…TransparentOutline` | `lts_trans` / `lts_trans_o` |
| `Hidden_lilToonFurCutout` / `Hidden_lilToonFur` | `lts_fur_cutout` / `lts_fur` |
| `Hidden_lilToonGem` / `Hidden_lilToonRefraction` | `lts_gem` / `lts_ref` |

替换 `.mat` 里 `m_Shader` 的 GUID（从 `Packages/jp.lilxyzw.liltoon/**/*.shader.meta` 取）即可，**不必进 Unity**。本机 51 个材质替换 29 处，
复查后 `m_Shader` 只剩真实 lilToon 与 Unity 内置（`0000000000000000f…` 等）。

**shader 名会显示成旧名**（如 `Hidden/lilToonOutline`）——lilToon 2.x 故意保留旧 shader 名做兼容，看到旧名不代表没替换成功。

### 23.7 离屏渲染验收（batchmode 可出图，但别加 `-nographics`）

- `-batchmode -nographics` **不能渲染**，要去掉 `-nographics`。
- `SkinnedMeshRenderer.bounds` 在刚 Instantiate 时可能返回默认 `(2,2,2)`，导致相机距离算错、模型只占画面 1/3。
  **改用 `renderer.localBounds` 的 8 个角点逐个 `TransformPoint` 再 Encapsulate**，得到精确包围盒。
- 朝向：Unity 人物通常面朝 **+Z**，相机放 `center + (0,0,+dist)` 才是正面（放 `-Z` 拍到的是后背）。
- 多视角一次渲染完（本机 6 视角：front / front_left / front_right / left / right / back），每张 `ReadPixels → EncodeToPNG`。

### 23.8 本次数字汇总

| 项 | 值 |
|---|---|
| bundle | 21,368,695 B，明文 |
| AssetRipper 导出 | 654 资产 / 1355 文件 |
| 编译 | 0 error CS（3 次迭代：补包 → 排除自测 → 清 Bee） |
| 缺失脚本 | 211 → 0 |
| 材质 / 贴图 | 51 全通过 / 117 全加载 |
| 根 prefab | SMR 51 · MR 11 · Animator 3 · 骨骼 508 · 材质 72 · 空材质 0 |
| 渲染 | 6 视角无粉无洞无穿模 |
