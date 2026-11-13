# 环境与工具链：从「什么都没有」到一个可开工的工程

读这份文件的时机：换机器、重装 Unity、接手一个旧工程、或不确定「该装哪个版本／该装哪些包」时。基底 SKILL.md 假设环境已经就绪；本册补的是**就绪之前**的部分。

> 本册所有路径都写成 `<PROJECT_ROOT>` / `<UNITY_EXE>` 占位符。不要照抄别的机器的盘符、编辑器路径或工程目录。

## 0. 先确认目标，再动手

开始装东西之前先把这四件事问清楚并记录（缺一项就先问，不要猜）：

1. **头像包路径**：要改的模型是 `.fbx`、`.unitypackage` 还是已经存在的工程？
2. **衣服包路径**：要接的衣服是哪个来源包？包里有几个尺码/变体？
3. **Unity 安装位置与工程目录**：用哪个 Unity 版本、工程放在哪、是不是已有工程？
4. **交付意图**：保留原始衣服菜单还是新增菜单？要不要翻译标签？**保留厂商标签还是统一改名**？（这一项会直接决定后面菜单与参数怎么建。）

同时确认**本次授权范围**：只在编辑器里看（读）、要改源资产（写）、要构建、还是要上传？构建与上传是两个独立的授权边界，预览/测试通过不等于可以上传（见 [07-02-evidence-and-ownership.md](../07-验证与排查/07-02-evidence-and-ownership.md) §2）。

### 0.1 本机已经就绪（截至 2026-10-16，不必重装）

> 下面是**这台机器**的实测现状，只为省掉重复劳动；写进文档/脚本时仍用占位符，见 §5 末。

| 东西 | 位置 / 状态 |
| --- | --- |
| Unity 编辑器 | `<VRCHAT_ROOT>\tools\Unity\2022.3.22f1\Editor\Unity.exe`，版本 **2022.3.22f1c1**（中国版，revision 12831699），约 5.3 GB |
| 安装包（重装用） | `<VRCHAT_ROOT>\tools\downloads\UnitySetup64-2022.3.22f1.exe`（2.83 GB）→ `UnitySetup64-2022.3.22f1.exe /S /D=<VRCHAT_ROOT>\tools\Unity\2022.3.22f1` |
| 授权 | 免费 **Personal 已验证可用，无需登录**（复用 `C:\ProgramData\Unity\Unity_lic.ulf`；headless 建工程 exit 0，日志有 `Successfully resolved entitlements` / `IsPro: 0`） |
| Unity Hub | `<VRCHAT_ROOT>\tools\UnityHub\Unity Hub.exe`（3.22.3，便携部署）—— ⚠️ **不会自动登记工作区里的编辑器**（`editors-v2.json` 仍是空数组），需在 GUI 里 Add/Locate 一次；VCC/ALCOM 已直接指向编辑器路径，不经 Hub 也能用 |
| VCC 设置 | `<USERPROFILE>\AppData\Local\VRChatCreatorCompanion\settings.json`：`pathToUnityHub` = `<VRCHAT_ROOT>/tools/UnityHub/Unity Hub.exe`，`preferredUnityEditors."2022"` = `<VRCHAT_ROOT>/tools/Unity/2022.3.22f1/Editor/Unity.exe`；备份在 `<VRCHAT_ROOT>\tools\backup\VCC-settings.json.bak` |
| ALCOM（VCC 开源替代） | `<VRCHAT_ROOT>\tools\ALCOM\ALCOM.exe`（1.1.8）；配套 CLI **vrc-get 1.9.2** 在 `...\WinGet\Packages\anatawa12.vrc-get_...\vrc-get.exe`，读的就是 VCC 那份 `settings.json` |
| MCP 服务端 | `mcp-for-unity.exe`（mcpforunityserver 10.3.0，`uv tool install`），已配进 DSH（`serverName: unity` → 工具前缀 `mcp__unity__*`） |
| Unity 端包 | `com.coplaydev.unity-mcp` **10.3.1-beta.7**，作为**嵌入式包**放在工程 `Packages\com.coplaydev.unity-mcp`；**卸载 = 直接删这个文件夹** |
| Windows GUI 自动化 | `windows-mcp` 0.8.5（`mcp__windows__*`，20 个工具）—— ⚠️ 等于把整台机器的鼠标键盘交给 AI，**只在需要时用，用完停掉，别设开机自启** |
| 改模工程 | `<VRCHAT_ROOT>\projects\LuzzyMeow`（2026-10-17 按模板重建，2022.3.22f1c1、约 1,687 MB；旧工程 2026-10-18 已按用户要求删除，原归档位置 `projects\_backups\LuzzyMeow-old-2026-10-17`） |
| 该工程已装的包 | VRCSDK Base 3.10.4（locked 3.10.5）/ Avatars **3.10.5** + VRCFury 1.1434.0 + Modular Avatar 1.18.7 + NDMF 1.14.8 + lilToon 2.3.4 + Poiyomi 10.0.25（+ ThryEditor 2.74.3）+ **Gesture Manager 3.9.9（必装，见 §1.1 与铁律 5）** + Avatars 3.0 Manager 2.1.2 + Thry APT 1.3.7 + lilycalInventory 1.5.2 + Limitex 0.9.0 + AAO 1.9.20 + VPM Resolver 0.1.29 + 嵌入式 `com.coplaydev.unity-mcp` 10.3.1-beta.7 |

⚠️ **唯一的剩余前提**：`mcp__unity__*` 需要**用户先把这个工程在编辑器里打开**、**且编辑器端跑的是 stdio 桥（`127.0.0.1:6400`）**才能连上（Unity 默认走 HTTP 8080，
状态栏那句 `MCP-FOR-UNITY: Session connected` 不代表 AI 连上 —— 见 `01-06-project-launch-and-mcp-bridge.md` §3/§4）。编辑器已装好、授权已可用 —— **不需要再装 Unity**。

⚠️ **版本差异（动手前必看）**：官方要求 **2022.3.22f1**，官方页明确**别升级版本**（升级后上传的内容可能不加载）。现役 `projects\LuzzyMeow` 与模板都已是 **2022.3.22f1c1**（与本机编辑器一致、**不弹提示**）；
只有归档的旧工程 `projects\_backups\LuzzyMeow-old-2026-10-17` 写的是 `2022.3.62f3c1` —— 若打开它，弹版本提示时**选「继续」不升级**（见 §3）。

★**踩坑（可复用）**：本机 pwsh 会话继承了 DSH 的 `ELECTRON_RUN_AS_NODE=1`，任何 **Electron GUI**（Unity Hub 等）会被当成 node 启动、**秒退且无窗口**。启动前先执行：

```powershell
Remove-Item Env:ELECTRON_RUN_AS_NODE -EA SilentlyContinue
```

网络：`pypi.org` 直连很慢 → 用清华源 `UV_DEFAULT_INDEX=https://pypi.tuna.tsinghua.edu.cn/simple`；`github.com` 直连不可用 → 用 `gh-proxy.com` 下载 release；**读 GitHub 页面 / API 最稳的路子是 anysearch 的 extract（服务端抓取）**。

其它：本工作区要求**软件一律装在工作区内**（`<VRCHAT_ROOT>\tools\`）；放不进去的只有 DSH 的 MCP 配置、`%APPDATA%\UnityHub`、`%LOCALAPPDATA%\VRChatCreatorCompanion`。另有两个失效开始菜单文件夹与一个幽灵卸载项 `Unity 2022.3.22f1c1` 清不掉（需管理员），**无害**。

## 1. 工具链清单与检测（只读）

### 1.1 需要装/需要验证的东西

- **Unity Hub**（管理版本）+ **VRChat 支持的 Unity 版本**（见 §3）。
- **VCC（VRChat Creator Companion）**：VPM 包管理入口，社区包一键装。
- **VRChat SDK：Base + Avatars**（World 项目另算；本工作流只针对 Avatar）。
- **NDMF** + **Modular Avatar**（衣服与功能的合并层）。
- **着色器依赖**：通常 **lilToon**，部分资产要求 **Poiyomi**——按材质实际用的 shader 装，不要「两个都装上以防万一」。
- **Gesture Manager**：**必装**（用户 2026-10-17 强调）。新建 / 重建工程后第一件事之一就是核实它在不在（`Packages/vpm-manifest.json` 里有 `vrchat.blackstartx.gesture-manager`）；
  它是**编辑器内模拟 VRChat 运行时**（手势 / 参数 / 菜单 / 开关 / 表情 / PhysBone 动骨）的唯一权威工具，能省掉大量 Build & Test 往返 ——
  见 `SKILL.md` 铁律 5 与 `01-06-project-launch-and-mcp-bridge.md` §2.7。
- **Avatar Optimizer**：只在明确有优化目标或实测硬溢出时装。
- **Blender**：仅当需要真正的网格修改（权重/裁剪/形变烘焙）时才需要；Unity 内做不到的适配不要硬做。

### 1.2 检测 5 项（不改任何东西）

1. Unity 与 VRChat SDK 版本；
2. VPM 清单（`Packages/vpm-manifest.json`）与 Unity 包清单（`Packages/manifest.json`、`packages-lock.json`）；
3. **目标头像**上真实存在的 provider 组件（不是「包装了」就算用）；
4. 菜单 installer、参数组件、merge animator、衣橱源、表情生成器、预览 helper、优化器设置；
5. 资产作者声明的依赖与支持的安装路径（资产包里的 README / 声明文件）。

**不要从文件夹名或「包已安装」推断 provider 归属。** 一个包可以装着但当前头像完全没用它。检测结果是只读证据，不构成安装或改工程的许可。

### 1.3 `TOOLCHAIN_PROFILE`（记在项目自有文档里）

```text
target_avatar: <精确层级路径或 prefab 路径>
unity_mcp: <server/toolset 与精确实例，或 unavailable>
clothing: <选定的工具，或 manual>
expression: <选定的工具，或 custom owner>
props: <选定的 provider 规则>
preview: <工具或 none>
optimization: <工具或 none>
installed_evidence: <清单/组件/版本来源>
missing_tools: <已选但缺失的工具>
shared_consumers: <已知的其它根或 prefab>
authorized_actions: <inspect/import/configure/preview/build/runtime/upload>
status: CONFIRMED | NEEDS_USER_CHOICE | MISSING_TOOL | MCP_REQUIRED | STALE>
```

**用户选择 + 已装证据**两者齐全才激活某个 provider 的 playbook；缺任一项就只用通用源审计，并在任何 provider 变更前停下。

推荐角色（可逐类让用户独立选择，不要替他决定）：

| 类别 | 默认推荐 | 什么时候改成「保留现状」 |
|---|---|---|
| 服装/衣橱 | lilycalInventory | 衣服声明了另一个受支持 provider，或现有工作流有意保留 |
| 面部表情 | FaceEmo | 头像有用户想保留的定制表情 owner |
| 道具/功能 | 资产声明的 provider，否则 Modular Avatar | prefab 已有受支持的 provider 配置 |
| Unity 预览 | **Gesture Manager：必装必用**（新工程建好即核实；见 `SKILL.md` 铁律 5） | 只有用户明确说不做编辑器内测试时 |
| 优化 | 仅在要求或有实测需求时用 Avatar Optimizer | 没有优化目标或硬限制问题 |

DressingTools 已经存在时**它是受支持的**：不要只为「统一项目」推荐迁移。多类别请求要各角色独立选，然后检查组合结果之间是否在菜单、参数、Animator、骨架、网格、开关、Contact、PhysBone、生成输出上互相重叠。

复用与刷新：后续任务直接复用已确认的 profile，低成本重查包清单、目标组件、资产依赖与授权；只有**头像变了、provider 增删、工具版本实质变化、导入资产声明了另一 provider、用户想换工作流**时才重新询问。旧 profile 标 `STALE`，不要静默重写用户的选择。

## 2. 工程：新建还是复用

- **新建干净工程**与**原始下载分离**：下载来的资源包不要直接解到工作工程里再原地改。
- 复用已有工程时，先确认它属于谁、有没有别人的未保存工作；**不要**为了干净而覆盖或重置一个你只是接手的工程。
- 工作资产统一放一个明确命名的工作目录（见 [02-01-workspace-and-work-assets.md](../02-工程组织与工作流/02-01-workspace-and-work-assets.md)）。
- 工程要有版本控制或至少有一个可回滚的备份点；源资产不在 Git 里时，不能指望 Git 帮你恢复（见 [06-02-backup-and-package.md](../06-构建上传与备份/06-02-backup-and-package.md)）。

## 3. 版本对齐（最容易踩的坑）

- **以 VRChat 官方当前要求的 Unity 版本为准**，不要以「我手上装的是哪个」为准。官方当前要求可查 <https://creators.vrchat.com/sdk/upgrade/current-unity-version/>；有疑问时先查官方来源，不要凭记忆或旧博客。
- 一个已有工程的 `ProjectSettings/ProjectVersion.txt` 里写的版本可能**不是** VRChat 当前要求的版本。**直接双击打开会触发升级提示**，一旦确认升级，包解析、导入设置和序列化都可能变化。
- 正确顺序：先确认要装哪个版本 → 用 Hub 装好该版本 → 用 VCC 打开/创建工程（VCC 会管理 VPM 包与版本）→ 再让它解析版本差异。
- 同机可能装着多个 Unity 版本。**调用编辑器/自动化时永远指定精确版本与 `-projectPath`**，不要依赖「最近打开的那个」。
- 版本对齐属于**用户决定**：如果工程版本与官方要求不一致，先把差异说明清楚并让用户选（升级工程 / 装对应版本重开），不要擅自升级。

## 4. 导入顺序与等待编译

固定顺序：

1. **先头像**（`.fbx` / `.unitypackage` + 贴图）；
2. **再 shader 与依赖**（lilToon / Poiyomi、NDMF、Modular Avatar、其它被资产声明的包）；
3. **最后衣服**。

每步之间：

- **等编译与导入彻底结束**再跑编辑器脚本。导入/编译未完成时跑脚本，会拿到半成品状态，甚至把陈旧验证结果带进构建。
- 改完外部文件要显式触发导入/刷新；没有外部写入时不要全工程 ForceUpdate。
- 日志与报告写到工程或工作区的 `Logs/` 下，便于回溯（Windows 上 Unity 编辑器日志另有其固定位置，见 [07-04-verification-discipline.md](../07-验证与排查/07-04-verification-discipline.md) §日志）。
- 编译失败时**先修所有无关错误**：一个坏的编辑器脚本会让 Unity 编译不出**所有**新菜单项与后续工具，症状会表现成「工具消失了」。

## 5. 工程内该有的东西（开工前自检）

- [ ] Unity 版本已对齐，工程能被 VCC 正常打开；
- [ ] VPM 清单里能看到 SDK Base/Avatars 与所需的 NDMF/MA/shader 包；
- [ ] 目标头像能在场景里实例化并出现 `VRCAvatarDescriptor`；
- [ ] 有 Humanoid Animator 与一个明确的骨架根（通常叫 `Armature`，但**必须实际读出来**，不要按名字硬编码）；
- [ ] 有原始完整头像 prefab（不是厂商的 kiseeae/naked/test 变体）；
- [ ] 工作目录、`Logs/` 目录已建；
- [ ] 有备份点或版本控制；
- [ ] `TOOLCHAIN_PROFILE` 已写并 `status: CONFIRMED`；
- [ ] 交付物里的路径写的是 `<PROJECT_ROOT>` / `<UNITY_EXE>` 占位符或**工程内相对路径**，**没有把本机盘符写死**（§0.1 的表格是「本机参考」，不是通用常量）。

## 6. 常见环境问题对照

| 症状 | 先查什么 |
|---|---|
| 打开工程提示升级 Unity | `ProjectVersion.txt` 与官方要求版本的差异——**先问用户再决定**，不要点确认 |
| 包管理器报解析错误 | VPM 清单与锁文件是否一致、registry 是否可达、是否引用了 `file:` 的本地包而路径已变 |
| 菜单里找不到某个工具 | 是否有编译错误（一个坏脚本会连带屏蔽所有新菜单项） |
| 看不到 MA 组件效果 | MA 组件只在 Bake / Play / 上传时生效（见基底 SKILL.md 铁律） |
| 场景视图里穿模 | 先 bake 再判断，编辑器视图里的穿模大概率是假象 |
| 编辑器卡住/无窗口 | 正在编译/导入？有 Unity 进程持有工程？陈旧进程优雅退出后用 `-projectPath` 重开精确版本；**只有确认没有 Unity 进程持有时**才删 `Temp/UnityLockfile` |
| Unity MCP 连不上 | 编辑器没开、**工程没开**（本机编辑器已装好，缺的就是这一步）、宿主里 MCP 条目没生效（重开编辑器/重启宿主后再试） |

## 7. 交付这条链的边界

环境搭建本身**不授权**任何后续动作。装包是用户决定；导入、改源、预览、构建、上传各有自己的授权边界。装完之后先停下来汇报：装了什么版本、什么包、工程在哪、还缺什么，再问下一步。

## 8. SDK 3.10.6 成对升级与 VPM 镜像取包（本工作区实测 2026-11-08，第 161 轮）

### 8.1 SDK 与插件必须成对升级

把 SDK 升到 3.10.6 后 Unity 编译中断：`error CS0618: 'VRCExpressionParameters.TypeCost' is obsolete`（报在 VRCFury）——VRCFury 1.1276.0 调用了 3.10.6 已废弃的 API，且 Unity 把 obsolete 当 error（`grep -c "error CS" Editor.log`=3，全部指向 `Packages/com.vrcfury.vrcfury/Editor/VF/Utils/VRCExpressionParameterExtensions.cs`）。

**版本配对（实测编译 0 error）**：

| 包 | 最低版本 |
|---|---|
| VRChat Avatars SDK | 3.10.6 |
| VRCFury | **1.1442.0** |
| Avatar Optimizer | ≥1.9.21 |
| Modular Avatar | ≥1.18.x |
| NDMF | ≥1.14.x |

升级后必须重跑一次 Manual Bake 并逐项比对（EP n/bits、菜单项数、MA Responsive 层数）再上传；原生 Face Tracking 的接入见 [04-09](../04-外观与表情/04-09-face-tracking-native-sdk.md)。

### 8.2 官方 VPM 源不可达时的镜像取包

官方 `packages.vrchat.com` 返回 522 源站超时 / 403，`vrchat.github.io/vpm-repos` 404，VCC/ALCOM 都没装时：
1. `GET https://vpm.vrczh.org/repos` 列出所有镜像仓库（Official=`/vpm/official`、nadena=`/vpm/nadena`、anatawa12、vrcfury、lilxyzw、curated）；
2. `GET https://vpm.vrczh.org/vpm/<repo>` 拿 VPM index JSON（33 万字符）；
3. 从 `packages.<name>.versions.<ver>.url` 直接下 zip；
4. zip 根即包内容，解压覆盖 `Packages/<name>` 并把旧目录移出工程（下载缓存 `_stage/pkgs/`）。

国内镜像源清单（社区自建源等）另见 [02-09 §2](../02-工程组织与工作流/02-09-project-workflow-notes.md)；ALCOM 常规装包见 [01-06 §2.2](01-06-project-launch-and-mcp-bridge.md)。
