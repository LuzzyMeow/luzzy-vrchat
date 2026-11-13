# luzzy-vrchat · 分册总索引（references/）

`references/` 是本知识包的全部资料，共 **66 册**（基底 7 册 + 主题分册 **59 册**），按主题分类存放。

- 技能主入口：[../SKILL.md](../SKILL.md)（工作原则、**六条铁律**、从零导入到上传全流程、分册导航与症状路由）
- 包总览与目录树（**包外，可能不存在**）：`<VRCHAT_ROOT>/tools/sources/vrchat-avatar-modding/README.md`
- 本页顺序：**命名约定与更新政策** → **分册一览** → **分册索引**（逐册「何时读 / 补的哪个缺口」）→ **来源与映射** → **与基底的对应关系** → **使用前提（本机现状）** → **待补（已知缺口）** → **旧名映射** → **原 08-症状速查 的结构化去向**。

## 命名约定与更新政策（2026-11-11 用户指定）

1. **统一命名**：全部知识文件为 `NN-MM-<slug>.md`。`NN`=分类号（与所在目录一致，00–07）；`MM`=两位册序（该分类内递增）；`slug`=英文 kebab-case 主题；后缀统一 `.md`。索引文件 `SKILL.md` / `README.md` 不改名。
2. **取消「08-症状速查」栏目**：原 9 册（主表 + 第 123–177 轮增补）已于 2026-11-11 全部删除，内容**转化为结构化知识**并入 00–07 各主题册（新增 6 册 + 现有册追加小节，均带「原 08-症状速查 第 N 轮 / 本工作区实测 YYYY-MM-DD」来源标注）。去向见文末「原 08-症状速查 的结构化去向」。
3. **以后的增量更新直接转化为结构化知识**：新实测（含症状类）一律按「症状 → 机制/根因 → 正确做法 → 判据 → 来源与实测日期」写入对号的主题册；新主题在对应分类下新建 `NN-MM-<slug>.md`（MM 取最大册序 +1），并同步更新本索引与 `../SKILL.md` 导航。**不再按时间开症状增补册，不再往 SKILL.md 堆症状行。**
4. **基底口径**：`00-基底/` 7 册的外部 MIT 原文保留；2026-11-11 起每册末尾追加「补充」节（联网核实的官方口径 + 结构化分册路由）。不再宣称「一字不改」。
5. **记账**：原 `_本工作区补充日志.md` 已于 2026-11-09 删除，不再新建台账；轮次与改动体现在各分册的实测日期标注与本索引里；已知缺口见文末「待补」节。

## 分册一览

| 分类 | 册数 | 里面有什么 |
|---|---|---|
| [00-基底](00-基底/) | 7 | 外部 MIT 原仓库 7 册，**原文保留 + 官方补充节**：编辑器自动化、MA 组件范式、PhysBone、面捕、网格形变、插件生态、故障排查 |
| [01-环境与工具链](01-环境与工具链/) | 6 | 装环境与版本对齐、国内镜像源、SDK 面板逐字段、Unity 基础操作、素材导入判据、建工程/开工程/AI(MCP) 桥接、**SDK 3.10.6 成对升级与 VPM 镜像取包** |
| [02-工程组织与工作流](02-工程组织与工作流/) | 11 | 工作场景与工作资产、MA 化流程与操作习惯、MA 组件决策与任务工作流、素材库分类与重组、外部教程笔记与比对、跨模型衣适配套件、VRCAL 素材库、Assets 重新分类整理、VRCFury Write Defaults 弹窗、MA 通用机制/Setup Outfit、**互动插件集成（RK Player / RealKiss / GUN1 道具）** |
| [03-菜单与参数](03-菜单与参数/) | 7 | 菜单/参数审计与预算、**EP 权威规则**、MA 开关与轮盘配方、动画控制器与变量、实战与部位消失排查、反射批量建菜单（**控件语义 + 一键穿衣混搭补全**）、MA 平台同步/MMD/参数菜单组件语义 |
| [04-外观与表情](04-外观与表情/) | 9 | lilToon 外观（含官方补充）、表情曲线与图标、颜色档位与 Radial Puppet、原生 FT 早期调研、狼脸手势表情、MA 反应式组件全解（**OT 极性语义全解**）、VRCFT 参数前缀修复、**OSC 面捕接入与修复**、**原生 Face Tracking（SDK 3.10.6+）** |
| [05-动骨与穿模](05-动骨与穿模/) | 11 | 服装接线与几何验收（**袜子/眼罩/约束整定**）、PhysBone 全参数与调试循环、Blender 改网格与 fbx 替换、模型解剖与换头、跨模型配件迁移、换头拆除与克隆、SPS 绑定、尾巴/手套迁移、跨头像衣装适配（NewFit）、ABT 工具绑点、MA 骨骼与几何 17 组件（**道具尺度/约束量测**）、**蒙皮绑定与几何修复** |
| [06-构建上传与备份](06-构建上传与备份/) | 6 | SDK 构建上传与版本管理、备份与打包迁移、从已上传模型恢复工程、反射出包与反射上传、服务端多包并存与 PUT 对齐、**上传凭据生命周期与客户端侧取证** |
| [07-验证与排查](07-验证与排查/) | 9 | 烘焙与骨骼验收、证据阶梯与所有权、Unity MCP 纪律、provider playbook、常见坑与版本实测、GM 自动化模拟测试（**GM 补全 §23**）、场景骨骼污染还原、多角度独立验证五件套、**动态验证边界与证据配方** |

## 分册索引

> 下表按目录分组，覆盖 01–07 共 59 册；00-基底 7 册逐册说明见 `../SKILL.md` 的「基底 7 册」表。
> 「何时读」= 什么场景该翻它；「补的是哪个缺口」= 它填的是哪块空白。

### 01-环境与工具链（6 册）

| 文件 | 何时读 | 补的是哪个缺口 |
|---|---|---|
| [01-01-environment-and-toolchain.md](01-环境与工具链/01-01-environment-and-toolchain.md) | 新工程、新机器、版本对不上、要装什么包 | 环境搭建与版本对齐、包清单、导入顺序；Gesture Manager 列为必装；**§8 SDK 3.10.6 成对升级（VRCFury 1.1442.0 / AAO 1.9.21 / MA 1.18.x / NDMF 1.14.x）与 VPM 镜像取包（vpm.vrczh.org）** |
| [01-02-environment-and-sdk.md](01-环境与工具链/01-02-environment-and-sdk.md) | 从零装环境、换国内镜像源、建项目、用 SDK 面板上传、补 Unity 基础操作 | 中文教学的安装与上传实操链路：Hub/编辑器/ALCOM、镜像源、手动装包、SDK 面板逐字段、加速器、覆盖更新、父子级/预制件、贴图导入、手动与 MA 绑骨、GM 跑测试 |
| [01-03-upload-errors-and-toolchain.md](01-环境与工具链/01-03-upload-errors-and-toolchain.md) | 上传报错、装环境细节、看不懂 SDK 面板某一项 | 系统环境三查、安装细节与更新陷阱、镜像源隐藏配置、SDK 面板逐字段 + 设置页签 8 项、5 条上传报错对照 + 网络四不要、覆盖更新与 blueprintId 粘回 |
| [01-04-unity-ui-and-extra-tools.md](01-环境与工具链/01-04-unity-ui-and-extra-tools.md) | 认不清 Unity 界面图标、判断下载物进 Assets 还是 Packages、变量范围 | 三窗口与图标辨识、预制件编辑模式、2by3 布局、Assets vs Packages 判据、形态键基础、Int 0–255 / Float ±1.0 同步硬约束、Photon Studio / OVRAS / 组网看片 |
| [01-05-babystep-toolchain-and-upload.md](01-环境与工具链/01-05-babystep-toolchain-and-upload.md) | 从零装包 / 整理素材 / 上传前自查 / 性能等级太差 | 中文入门工具链：ALCOM 代替 VCC、许可证失效、lilEditorToolbox、lilycalInventory vs MA 联动开关、基本模板工程、素材导入四情形、上传前 6 项检查、Very Poor 四成因 |
| [01-06-project-launch-and-mcp-bridge.md](01-环境与工具链/01-06-project-launch-and-mcp-bridge.md) | 新建/克隆工程、第一次开别人的工程、AI 报 No Unity Editor instances found、Session connected 却连不上 | 本机实测「建工程→开工程→接 AI」全流程：模板拷贝、ALCOM 装包/开工程、stdio 6400 vs HTTP 8080 互斥与四种修法、HTTP 端点自救、退出/崩溃判据、§2.7 GM 必装、§2.8 按需解压、§12 重启配方、**§13 MCP 瞬时失效重试 / refresh≠编译 / FixCookieContainer 环境前提 / flag-file 兜底 / delayCall 失焦** |

### 02-工程组织与工作流（11 册）

| 文件 | 何时读 | 补的是哪个缺口 |
|---|---|---|
| [02-01-workspace-and-work-assets.md](02-工程组织与工作流/02-01-workspace-and-work-assets.md) | 建工作场景/prefab、挂衣服、合并 FX | 工作资产目录约定、基础头像选型、本地变换清零、FX 合并不替换；§7 删资产前 GUID 引用扫描 |
| [02-02-ma-workflow-habits.md](02-工程组织与工作流/02-02-ma-workflow-habits.md) | 要把存量头像 MA 化、加新套装/发型、按什么顺序动手 | MA 操作习惯：套装容器、菜单树、.bk 备份、互斥参数与 auto promotion、参数命名、codegen 中文转义、商业模块、A0 八步、加套装九步、验收 12 条、MA↔VRCFury 对照、§15–§19 一键穿衣演进、§21–§39 各类实测坑 |
| [02-03-ma-component-guide-workflows.md](02-工程组织与工作流/02-03-ma-component-guide-workflows.md) | 不知道该用哪个 MA 组件、每条任务先检查什么 | 组件决策总表 + 前置检查 + 工作流 A–H；实现方式优先级 MA 优先 |
| [02-04-vrcfury-write-defaults-dialog.md](02-工程组织与工作流/02-04-vrcfury-write-defaults-dialog.md) | 进 Play 弹 Write Defaults 弹窗、想永久不再弹 | 根因（多层 WD 混用）、三按钮真实行为、Auto-Fix 等价反射写法、烘焙判据 + 全菜单回归 |
| [02-05-unity-project-asset-reorg.md](02-工程组织与工作流/02-05-unity-project-asset-reorg.md) | 接手工程把 Assets 按用途重新分类、怕断引用/插件失效 | 五条安全移动铁律、dry-run→执行→回滚、三级验证（GUID set-equal / MD5 多重集 / descriptor+GM Play）、7 个必修坑、§9 场景层级整理 |
| [02-06-resource-library-reorg.md](02-工程组织与工作流/02-06-resource-library-reorg.md) | 整理 resources 素材库、判断商品放哪、去重与 junction | 10 个一级目录、文档跟商品走、去重→junction、搬迁脚本纪律、删除审计、库快照与撤销 |
| [02-07-vrcal-asset-library.md](02-工程组织与工作流/02-07-vrcal-asset-library.md) | 素材多、想自动匹配 BOOTH 编号/名称、只读索引/封面图库 | VRCAL v1.10.3：只读扫描、BOOTH 匹配、人工确认、FTS5 搜索、GPL-3.0；尚未安装实测 |
| [02-08-cross-avatar-clothing.md](02-工程组织与工作流/02-08-cross-avatar-clothing.md) | A 模型衣服穿到 B 模型、NewFit/DressingTools/AvatarDresser 怎么选 | NewFit 付费物理重拟合（$15/¥2,500）、DressingTools 免费骨骼映射、AvatarDresser 配饰、兽人注意、§8 骨骼绑定与移动态验收 |
| [02-09-project-workflow-notes.md](02-工程组织与工作流/02-09-project-workflow-notes.md) | 过外部教程、查细胳膊细腿根因、MA 菜单组织、git 回档、换妆容瞳色、FaceEmo、RK Player/AudioLink、国内 VPM 镜像 | 9 集逐集增量 + 评论区补丁，全部标「未验证」并列复核点 |
| [02-10-ma-setup-outfit-and-general-behavior.md](02-工程组织与工作流/02-10-ma-setup-outfit-and-general-behavior.md) | 整套服装 Setup Outfit、判断能否被 MA 直接处理、手动烘焙看产物、对外分发 | MA 通用机制总册（官方文档对齐）：Setup Outfit 两级与找骨判据、骨表回退、A/T-pose、Merge Armature 规则、Outfit Root/Mesh Settings、.1 规避、humanoid 骨 PB 危险、烘焙期改写总表、Manual Bake 产物、错误面板、分发/Internal/SemVer |
| [02-11-plugin-integration-rk-realkiss-props.md](02-工程组织与工作流/02-11-plugin-integration-rk-realkiss-props.md) | RK Player 没声音/CD 封面/扩盘、RealKiss 装上无反应/口裂定标、GUN1 装备反相、道具两级显隐 | **互动插件集成手册（原 08 第 142–177 轮结构化）**：RK 音频接线与控制器四坑、Relative 路径基准、CD 封面三槽与序号映射、CD9/10 四件套、FLAC→OGG 两条路线、QQ 音乐取图、RealKiss 版本/Head 挂载/RKSActivator/菜单名/口裂定标与 GM Contact、GUN1 条件反相、属性→层反查 |

### 03-菜单与参数（7 册）

| 文件 | 何时读 | 补的是哪个缺口 |
|---|---|---|
| [03-01-parameter-and-menu-audit.md](03-菜单与参数/03-01-parameter-and-menu-audit.md) | 菜单开关无效、参数超预算、默认值不对、saved 不记忆、duplicate definition | 参数成本公式与预算、每页控件上限、四表面同步、烘焙后菜单、§8 saved 持久化、§9 saved 盖默认值（含第 166 轮裁决注）、**§10 EP 权威规则：ValueType 实测枚举 / ParameterInfo.ForUI 与 CalcTotalCost 位宽权威 / saved 四象限与本地参数 / localOnly 与多声明 / 内置参数 IsLocal·AFK 撞名两条路径 / EP↔FX 类型校验 / hash 冲突 / m_overrideAnimatorDefaults / 未声明参数 / 出包审计红线** |
| [03-02-animator-menu-particles.md](03-菜单与参数/03-02-animator-menu-particles.md) | 做换装菜单、选 A/B 路线、配粒子特效、绑骨踩坑 | 手搓 Animator vs MA Create Toggle、两处同名参数、Merge Animator 三设置、粒子与 Custom Vertex Streams、FX 孤儿子资产清理（20,446→4,921） |
| [03-03-case-study-and-sdk-panel.md](03-菜单与参数/03-03-case-study-and-sdk-panel.md) | 走完整实战、绑骨选组件、部位消失或开关失效 | 中文实战全流程：Booth 选型、手动绑骨与 Merge Armature vs Bone Proxy 铁律、形态键被占用排查（层权重二分法 + A3M WD 页签）、外观与眼睛贴图 |
| [03-04-ma-menu-and-switch-cookbook.md](03-菜单与参数/03-04-ma-menu-and-switch-cookbook.md) | 用 MA 做开关/换色/轮盘、改别人模型自带菜单 | MA 逐组件配方：菜单优先级与 Extract Menu、int 开关 256 件、Material Setter、Radial Puppet、_IsGrabbed/_IsPosed 动骨动画、富文本按钮、Blend Shape to Animation |
| [03-05-ma-menu-hair-style-color.md](03-菜单与参数/03-05-ma-menu-hair-style-color.md) | 发型/发色切换菜单、MA 菜单 + 动画控制器怎么接 | MA Parameters(Int saved) + Button 单选 + MenuInstaller + MergeAnimator；发型=m_IsActive、发色=m_Materials Ptr；AddLayer 默认权重 0 坑；§7 默认色三处一致 |
| [03-06-ma-menu-build-by-reflection.md](03-菜单与参数/03-06-ma-menu-build-by-reflection.md) | 菜单项太多手工点不完、批量建/改 MA 菜单与 OT、一键穿衣与混搭 | 反射建菜单手册：类型/字段速查、internal 嵌套类型、struct 整条写回、模板复制、More 处理、烘焙验收；§9 Alerts 告警修复；§10 ControlType 真值；§11/§12 菜单层级；§13/§14/§14.9 一键穿衣演进与持久化；§15 改名落点；§16 删部件四件套；§17 第三方道具菜单归位 + AFK 可见性；**§18 控件语义（Button 脉冲/Int 三选一/RadialPuppet subParameters/label 去重/SubmenuSource/prefab 实例拒烘/悬空转移 YAML 修复）；§19 一键穿衣混搭补全（分部件范式/层名=目标名/权威写者/历史档位兜底/Equals N/互斥矩阵/分组根壳必开/音声OFF 极性/混搭持久化五步/driver 层创建硬坑/Rest Pose isDefault）** |
| [03-07-ma-platform-sync-mmd-and-motion.md](03-菜单与参数/03-07-ma-platform-sync-mmd-and-motion.md) | MergeAnimator 层序打架、VRCAnimatorLayerControl、PC·Quest 双平台、MMD 世界 | 官方对齐：MergeAnimator Layer Priority 与 Replace、path mode（§1.7 含 Relative 基准=组件 GO 与双拼接坑）、Merge Motion、MMD 2·3 层 relay/padding、SyncParameterSequence、PlatformFilter、Parameters/MenuItem 字段语义 |

### 04-外观与表情（9 册）

| 文件 | 何时读 | 补的是哪个缺口 |
|---|---|---|
| [04-01-appearance-and-liltoon.md](04-外观与表情/04-01-appearance-and-liltoon.md) | 调亮度、换色、发光、单色照明、判断 lilToon 属性真伪 | 外观属性真伪、实现方式选择、亮度/去色/换色/发光、常见误判；**官方补充：shader 变体一览（lilToonMulti/Lite/Overlay/OutlineOnly/FurOnly）、贴图槽位对应、构建期自动优化与 CBUFFER 噪声、版本与分发口径（MIT）** |
| [04-02-expressions-and-icons.md](04-外观与表情/04-02-expressions-and-icons.md) | 修表情曲线、生成预览图、换菜单图标、参数存不存 | 驱动链排查、clip 修改、表情重设计、FaceEmo 预览、图标、saved/networkSynced、§9 默认表情对齐 |
| [04-03-color-and-puppet.md](04-外观与表情/04-03-color-and-puppet.md) | 加颜色档位或连续滑条、调头发层次高光 | 菜单容量路由、挑染、头发高光候选、形变、Radial Puppet 最小结构、§8 材质同步层漏 mesh、§9 timeParameter 亮度滑块 |
| [04-04-native-face-tracking-two-heads.md](04-外观与表情/04-04-native-face-tracking-two-heads.md) | 原生 FT 早期调研、双头/换头路线评估 | 适用版本前提、官方 beta 要点、本工程盘点、三条路线与代价、最小实装步骤（未验证）；**现行原生 FT 实装见 04-09** |
| [04-05-wolf-face-gesture-expressions.md](04-外观与表情/04-05-wolf-face-gesture-expressions.md) | 换头后狼脸手势表情、加新表情、口型描述符 | 表情系统真实结构、8×8 槽位图（唯一空槽=Fist）、clip 判定、加表情标准流程、§6 视素名字交集、§7 双手握拳唐笑、§8/§9 丝滑化 crossfade |
| [04-06-ma-reactive-components.md](04-外观与表情/04-06-ma-reactive-components.md) | 换装开关/材质变体/形态键联动/按遮罩裁网格、菜单点了没反应、两个开关抢同一对象 | 官方对齐反应式全家族：总则·优先级·一帧时序、Object Toggle、Shape Changer、Material Setter/Swap、Blendshape Sync、Mesh Cutter 五个 Vertex Filter、Reaction Debugger；§12 Active 语义、§13 driver 常量同步、**§14 OT 极性语义全解（生成公式/Inverted 反相状态机/IsConstant/三处一致/父级遮断/壳必开/自锁/三轮定案/自建层被吸收/ShapeChanger Delete·Set）** |
| [04-07-vrcft-parameter-prefix-repair.md](04-外观与表情/04-07-vrcft-parameter-prefix-repair.md) | 面捕装好却完全没反应、怀疑客户端设置 | 参数前缀丢失 + OSCQuery 实测诊断：秒级定案法、四层根因链、位宽安全修复、验证清单、排除假设 |
| [04-08-face-tracking-osc-and-repair.md](04-外观与表情/04-08-face-tracking-osc-and-repair.md) | OSC/VRCFT 面捕接入、闭眼/极性/眼追/眨眼/手势门控、OSCmooth、补丁器、AAO 删键、网格迁移 | **OSC 面捕架构与修复全册（原 08 第 142–160 轮结构化）**：数据流与 OSCmooth 剥离、参数声明差集扫描、三处类型对齐、Relative 路径毁面捕、控制器抽取、Direct BlendTree+WD、EyeLid 反相与默认 0.8、眼睑索引槽、眨眼三源、gaze 2D 树、TrackingControl 枚举、手势门控、舌头通道、Button+Bool 脉冲、菜单归属、saved 边界、AAO 改名/删曲线/删键链、hpatchz 字节匹配、meta 补丁、可绑定率、Pawlygon 恢复、曲线引用基线、三角形双射、FT+32 键、globalScale、Blender Transfer |
| [04-09-face-tracking-native-sdk.md](04-外观与表情/04-09-face-tracking-native-sdk.md) | SDK 3.10.6+ 原生 FT 配置、103 键映射、gaze、瞳孔不左右动、高光漂移、迁移回归 | **原生 Face Tracking 全册（原 08 第 143/161/164/165/172/176/177 轮结构化）**：版本前提、descriptor 字段与最小配置、BlendshapesSet 索引数组与别名表、拆分空名占位、Auto-Fill 边界、gaze 8 单极通道 highest-wins、EyeGazeMode 与预览 bug、瞳孔质心法、Animator 覆盖与 FT 专用键（零顺序依赖）、hightlight_* 高光漂移修法、静态残留、OSC→原生迁移回归、消费者输入边界（GM 不能伪造）、菜单降级与重定向四步、effects 树分工 |

### 05-动骨与穿模（11 册）

| 文件 | 何时读 | 补的是哪个缺口 |
|---|---|---|
| [05-01-outfit-fit-and-clipping.md](05-动骨与穿模/05-01-outfit-fit-and-clipping.md) | 服装接线、穿模、几何验收、袜子膝穿模、眼罩浮空/可抓 | 可移植性判断、骨骼/形变复制前提、风险顺序、几何验收 5 条、§8 BakeMesh 世界坐标、§9–§11 ParentConstraint 眼罩定位与 offset、**§12–§13 袜子顶点缩放与 A/B（x103b 历史）、§14 x108xz 现行版 + Knee_OFF、§15 眼罩 PB 门控/约束目标定位/localPosition 弹回** |
| [05-02-bind-animation-physbone-fbx.md](05-动骨与穿模/05-02-bind-animation-physbone-fbx.md) | 删骨骼/参数、如梦衣柜换装、录动画换色、从 0 做动骨、Blender 改网格后替换 | 模型精简、如梦衣柜 6 步转移、录动画、材质换色、Bone Proxy 绑头发、Blender 刷权重、PhysBone 全参数与调试循环、fbx 替换前提 |
| [05-03-model-anatomy-and-head-swap.md](05-动骨与穿模/05-03-model-anatomy-and-head-swap.md) | 换头/换身体/替换部位网格、量比例与脖子接缝 | Kipfel/Shipilka 解剖对照、脖子截面与缩放推算、Unity↔Blender 分工、换头连带风险、§9 三支 B 站换头视频流程、§10 去衣去发少年化、§11 双头一键切换 |
| [05-04-cross-avatar-accessory-transfer.md](05-动骨与穿模/05-04-cross-avatar-accessory-transfer.md) | 搬翅膀/光环/配饰、浮空或尺寸不对、开关没反应、饰品默认关 | 真资产 vs AAO 烘焙网格判别、MergeArmature 前提、手动缩放骨架法（270° 旋转）、AvatarObjectReference 相对路径、MA Parameters 默认 0、SetFloat 双向断言、§8 isDefault/宿主 active/SMR transform/多实例、§9 拆两个开关与安全删除 |
| [05-05-head-swap-teardown-and-clone.md](05-动骨与穿模/05-05-head-swap-teardown-and-clone.md) | 拆除旧换头系统、以 A 头+B 身克隆新角色 | 删除顺序、Unity 假 null 坑、克隆身份重置、FX 专属副本、默认值三处一致、共享 vs 独立菜单、烘焙产物验收、ABT 绑点验法、发色图标、CodeDom 六坑、§11 图标透明背景、§12 清 Missing Script |
| [05-06-sps-and-ndick-binding.md](05-动骨与穿模/05-06-sps-and-ndick-binding.md) | 克隆头像 SPS socket 浮空/偏低/左右互换、菜单找不到 SPS | 对照参考头像 socket 世界坐标表、修根 TRS 与个别 socket、三层验证、installer 组件级复制 + 菜单 CopyAsset、烘焙验收与 NOT_RUN |
| [05-07-tail-gloves-migration.md](05-动骨与穿模/05-07-tail-gloves-migration.md) | 迁移尾巴/手套/配件、位置对但动画不播/物理不摆/发色不跟 | boneWeights 判驱动骨、TRS 逐骨复刻、PhysBone rootTransform 基类反射、控制器裁剪、MergeAnimator pathMode=Absolute、菜单 Toggle+OT、网格按名重绑、发色 object-reference 曲线、§8 显隐双 clip 层、§9 作者成对 OFF/ON clip 补开关 |
| [05-08-cross-avatar-garment-fit-newfit.md](05-动骨与穿模/05-08-cross-avatar-garment-fit-newfit.md) | 跨头像衣服尺寸/比例不同、作者漏贴图、NewFit 重拟合 | 骨骼名映射与回退、保留 bindpose 只换 bone、FBX 重建与等比 bindpose（禁止叠加）、缺贴图材质替换、NewFit 程序化全流程与边界、菜单接入、§8 形状扭曲正解、§9 手动摆件接回 MA、§10 菜单泄漏、§11–§27 各轮实测（体形切换/Kabsch/有符号距离/保形态键/多色板/骨架写坏/名牌/UV IoU）、§28 Riptide 接法、**§29 指向手势肌肉值** |
| [05-09-third-party-tool-prefab-binding-abt.md](05-动骨与穿模/05-09-third-party-tool-prefab-binding-abt.md) | ABT/笔/地图/传送 Instantiate 后飘在头顶、chibi 复用工具 | BP AsChildKeepWorldPose 语义、abt.localPosition 缩放/落点解法、WorldFixed 反向抵消、ABT 菜单与 4 个 MergeAnimator、烘焙克隆断言 |
| [05-10-ma-bone-and-geometry-components.md](05-动骨与穿模/05-10-ma-bone-and-geometry-components.md) | 合并骨架/配件挂骨/单骨缩放/整体高度/头饰第一人称/隔离 PB/全局碰撞体 | 官方对齐 17 组件全解（Merge Armature 三档锁定/Reset position/骨名匹配/烘焙改写、Bone Proxy 四态、Move To/Independently、Replace Object、Scale/Floor Adjuster、World Fixed/Scale、Visible Head Accessory、Remove Vertex Color、Mesh Settings、Outfit Root、PhysBone Blocker、Global Collider、Rename Tags、Convert Constraints）；§21 万圣节 bindpose 塌缩、**§22 BoneProxy 真人尺度道具定量反推、§23 活动约束量测陷阱与 pose 无关握持判据** |
| [05-11-skinning-binding-and-geometry-repair.md](05-动骨与穿模/05-11-skinning-binding-and-geometry-repair.md) | 重绑后衣服不可见/顶点乱飞、手套露指、口袋陷入、invalid normals、Humanoid 舞蹈转 Generic | **蒙皮绑定与几何修复（原 08 第 132/141/146/147/167/168/171/172/177 轮结构化）**：bindPoses 重算公式 + localBounds 归零、SMR Transform 无效总纲、跨空间重绑、手套指部权重两法 + Hand_OFF + boneWeights 判别、Knee_OFF、兜帽抬升、口袋 Play BakeMesh 测量、invalid normals 双根因（Blender4 #123088 + SDK legacy 链）、Humanoid→Generic 三连坑、LoadAllAssetsAtPath 按名取网格、眼镜/名牌放置、不对称设计判别 |

### 06-构建上传与备份（6 册）

| 文件 | 何时读 | 补的是哪个缺口 |
|---|---|---|
| [06-01-sdk-upload-and-versioning.md](06-构建上传与备份/06-01-sdk-upload-and-versioning.md) | 要构建、要上传、SDK 报错 | 完整上传流程：缩略图、blueprintId（含逆向 prefab 隔离）、releaseStatus、内容警告、告警分级、脚本上传适用面、backup |
| [06-02-backup-and-package.md](06-构建上传与备份/06-02-backup-and-package.md) | 备份、归档、导出 unitypackage、迁移工程 | 先量后设计、依赖分析、包体何时必须保存、导出与校验、还原顺序 |
| [06-03-recover-project-from-uploaded-avatar.md](06-构建上传与备份/06-03-recover-project-from-uploaded-avatar.md) | 工程丢了只剩已上传模型、缓存 __data 打不开、恢复出来 missing script/粉红/顶点乱飞 | 恢复链路与边界：本地 .vrca / File API / 缓存 AES → AssetRipper 2.0 / AvatarRecovery / dVRC / VRCALoader / FACS；缓存 key；反盗模；ripping vs hotswap；§21 修成可用 SDK 工程；§22 网页版导出；§23 API→FACS→shader 重指→渲染；NoNlab 颜色同步；缓存目录迁移 config.json |
| [06-04-sdk-publish-with-reflection.md](06-构建上传与备份/06-04-sdk-publish-with-reflection.md) | SDK 面板填不进、按钮灰、PendingUpload=True、想脚本出包上传 | 面板反射操作手册：_instance 私有字段、构建按钮判据、CropImage 6 参、Build(GameObject) 出包、CreateNewAvatar 反射上传、成功判据、副作用清理、68 条 alert、§8 二次出包排障、§9 菜单落盘与覆盖上传、§10 security variant 补丁 |
| [06-05-avatar-package-version-collision.md](06-构建上传与备份/06-05-avatar-package-version-collision.md) | 上传全绿版本在涨但游戏里一直是旧内容、重进世界/清缓存无效 | 服务端多包并存：UpdateAvatarBundle 只替换 (platform,variant,unityVersion) 同档；Unity CN f1c1 归一化；三指纹交叉定位；UnityPy 解菜单；PUT 顶替；AlignStalePackages；§9 超时假失败；§10 对齐后再分叉；§8 第 175 轮证据链；**§11 PUT 先 security 后 standard + 自动对齐跳过 standard** |
| [06-06-upload-credentials-and-client-evidence.md](06-构建上传与备份/06-06-upload-credentials-and-client-evidence.md) | 上传报 Unauthorized/NOAUTH/web-derived token、上传成功但游戏旧、要证明客户端换了包 | **上传凭据与客户端取证（原 08 第 141–176 轮结构化）**：Build 清空 ApiCredentials、VRCX cookie 注入、ConfigureHeaders 与 token 类型、占位凭证 4 步判别、APIUser.Login+CookieJar、探活优先、SDK 3.10 禁脚本上传、协议 SessionState、主线程、UpdateAvatarImage 签名与幂等、时间线比对、file 版本号判据、.vrca 直读与出包审计、客户端会话缓存重启、缓存目录版本号小端解码、__data vs .vrca、日志体积陷阱、__info/mtime、NOT_RUN 口径、脚本卫生（凭据清理/Find 失活/类型/-cmatch） |

### 07-验证与排查（9 册）

| 文件 | 何时读 | 补的是哪个缺口 |
|---|---|---|
| [07-01-bake-and-validation.md](07-验证与排查/07-01-bake-and-validation.md) | 交付前自检、骨骼绑定可疑、PhysBone 数异常、烘焙后功能消失 | NDMF/MA 烘焙审计、骨骼审计、PB 组件 vs 骨骼、GM 运行时验收；§10 ErrorReportWindow 真因；§11 Play 驱动两坑；§12/§13 temp 包删除；§14 缺 Installer；§15 默认关清单；§17 Console 清零（HasProperty/MA-1300/日志切片）；§18 亮度菜单空；§19 真实鼠标点面板；§20 默认形态三处律；§21 上传覆盖层；§22 去色 0.5；§24 SPS Plug；§25a 上传没发生；§25b Play 四坑；§26 脚本上传核对；§27 手势覆盖曲线；§28 ShapeChanger NaN；§29–§36 第 123–134 轮各坑（含 §25.5 LightController MD5 与备份路径） |
| [07-02-evidence-and-ownership.md](07-验证与排查/07-02-evidence-and-ownership.md) | 改共享资产、跨 provider 迁移、要下结论 | 7 层证据阶梯、授权矩阵、6 类所有权边界、消费者 map、迁移职责、生成输出禁改 |
| [07-03-provider-playbooks.md](07-验证与排查/07-03-provider-playbooks.md) | 用 FaceEmo / lilycalInventory / DressingTools / GM / AAO | 逐 provider 源与生成归属、只读检查序、常见任务诊断、验收矩阵 |
| [07-04-verification-discipline.md](07-验证与排查/07-04-verification-discipline.md) | 用 MCP 动 Unity、要读回验证、超时与失败 | MCP 连接门与调用纪律、分帧验收、prefab 覆盖差分、超时/崩溃/幂等 |
| [07-05-ma-gotchas-version-toolchain.md](07-验证与排查/07-05-ma-gotchas-version-toolchain.md) | 配了却不生效、确认 MA 版本行为、查字段名、来源权威顺序 | 10 条常见坑 + MA 1.18.7 字段实测与版本快照 + 来源权威顺序 + 官方链接 + MCP 读写序列 + 安全授权 + 4 个只读脚本 |
| [07-06-gesture-manager-automated-sim-test.md](07-验证与排查/07-06-gesture-manager-automated-sim-test.md) | 菜单 200+ 项人工点不完、要自动证明每个开关生效、截图证据、中途全黑 | GM 反射创建、update 状态机 harness、两遍法、Button 脉冲判定、RenderTexture 截图与图像分析、归 0 全黑陷阱、§9 play 域文件协议、§10 ANY 转移矛盾闪烁、§11 手工挂 Animator、§13 二次运行复位、§14 SPS 插头、§15 MissingAsset、§16 截图取证、§17 块级 diff、§18 日志噪声、§19 Play 根 Animator 无控制器、§20 MA Responsive 判读、§21 GM 模块四坑、§22 道具两级显隐、**§23 构造签名版本差异/组合测试/Action 层 AnimationMode 裁决/Contact 模拟/叶测试判定/边界清单** |
| [07-07-scene-bone-corruption-recovery.md](07-验证与排查/07-07-scene-bone-corruption-recovery.md) | 头像整个错位（不是某处穿模）、怀疑骨骼被改坏 | 排除渲染假象、全场景 YAML transform diff、同基底逐骨对照、路径+TRS 清单写回、连带跟随物、五步验收、预防 |
| [07-08-multi-angle-binding-verification.md](07-验证与排查/07-08-multi-angle-binding-verification.md) | 任何装骨/配件/衣装/道具改动的验收（用户指定唯一标准） | 多角度独立验证五件套：数值 / 编辑态离线多角度出图（+Z 铁律）/ 目视判读（PIL ≤420px）/ Windows-MCP 实机 / 烘焙产物；交付末屏；§9 六方位渲染矩阵 |
| [07-09-dynamic-verification-and-evidence-recipes.md](07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md) | Play 里验证不动、GM 跑不起来、要出三态截图、渲染不刷新、MCP 失效兜底 | **动态验证边界与证据配方（原 08 第 132–176 轮结构化）**：Play 十条边界（MA 烘焙产物/VRCFury 预览复位/driver 类型分派/Animator 泵送/层索引错位/inactive 绑定失败/destinationState 不可靠/NDMF 预览占线程/Play 域菜单/is_changing NOT_RUN）、构建产物 FX+逐帧 Update 配方、四跳静态证明与批量化、clip 回放渲染、ForceSkinnedUpdate、场景卫生（快照五段式/Play 退出异步/Manual Bake Clone 收尾/dirty 判读/Scene Visibility/SMR 重建）、取景与量测（骨坐标特写/侧视-90/BakeMesh useScale/Play bounds/旧图 mtime）、MCP 兜底（flag-file/DeleteAsset/delayCall 失焦/陈旧程序集/模态框） |

## 来源与映射

完整的上游仓库、教程系列、官方文档、许可证与逐册来源映射见 [../ACKNOWLEDGEMENTS.md](../ACKNOWLEDGEMENTS.md)。本索引不再重复作者/UP 名与合集信息。

## 与基底的对应关系

- 基底 `00-01-editor-automation.md`：[01-01](01-环境与工具链/01-01-environment-and-toolchain.md)（环境）与 [07-04](07-验证与排查/07-04-verification-discipline.md)（MCP 纪律）互补；C# 范式见 [02-01](02-工程组织与工作流/02-01-workspace-and-work-assets.md) 末尾；flag-file 兜底与渲染刷新见 [07-09](07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md)。
- 基底 `00-02-modular-avatar.md`：参数/菜单预算与路由见 [03-01](03-菜单与参数/03-01-parameter-and-menu-audit.md)；组件字段级语义见官方对齐 4 册 [02-10](02-工程组织与工作流/02-10-ma-setup-outfit-and-general-behavior.md) / [03-07](03-菜单与参数/03-07-ma-platform-sync-mmd-and-motion.md) / [04-06](04-外观与表情/04-06-ma-reactive-components.md) / [05-10](05-动骨与穿模/05-10-ma-bone-and-geometry-components.md)；EP 权威规则见 [03-01 §10](03-菜单与参数/03-01-parameter-and-menu-audit.md)。
- 基底 `00-03-physbone-dynamics.md`：组件数与预算审计见 [07-01 §4](07-验证与排查/07-01-bake-and-validation.md)；作者口径逐参数调试见 [05-02 §5](05-动骨与穿模/05-02-bind-animation-physbone-fbx.md)；官方语义补充已追加在本册末尾。
- 基底 `00-04-face-tracking.md`：OSC 架构实装见 [04-08](04-外观与表情/04-08-face-tracking-osc-and-repair.md)，原生 FT 见 [04-09](04-外观与表情/04-09-face-tracking-native-sdk.md)，前缀丢失诊断见 [04-07](04-外观与表情/04-07-vrcft-parameter-prefix-repair.md)；表情曲线与图标见 [04-02](04-外观与表情/04-02-expressions-and-icons.md)。
- 基底 `00-05-mesh-blendshape.md`：衣物形变复制前提与几何验收见 [05-01](05-动骨与穿模/05-01-outfit-fit-and-clipping.md)；换头与解剖见 [05-03](05-动骨与穿模/05-03-model-anatomy-and-head-swap.md)；跨头像衣装与蒙皮修复见 [05-08](05-动骨与穿模/05-08-cross-avatar-garment-fit-newfit.md) / [05-11](05-动骨与穿模/05-11-skinning-binding-and-geometry-repair.md)；形态键被占用排查见 [03-03 §7](03-菜单与参数/03-03-case-study-and-sdk-panel.md)。
- 基底 `00-06-plugin-ecosystem.md`：provider 归属见 [07-03](07-验证与排查/07-03-provider-playbooks.md) 与 [07-02](07-验证与排查/07-02-evidence-and-ownership.md)；插件集成实装见 [02-11](02-工程组织与工作流/02-11-plugin-integration-rk-realkiss-props.md)；国内替代工具链见 [01-05 §1](01-环境与工具链/01-05-babystep-toolchain-and-upload.md)。
- 基底 `00-07-troubleshooting.md` 与 `06-01`：中文社区报错成因见 [01-03 §4](01-环境与工具链/01-03-upload-errors-and-toolchain.md)；上传凭据与客户端取证见 [06-06](06-构建上传与备份/06-06-upload-credentials-and-client-evidence.md)；本册末尾已追加结构化排查路由表。
- 基底 `00-02` / `00-01`：MA 快捷菜单点击路径与手搓控制器见 [03-02 §1–§2](03-菜单与参数/03-02-animator-menu-particles.md)；从零装环境到上传见 [01-02](01-环境与工具链/01-02-environment-and-sdk.md)；MA 组件配方见 [03-04](03-菜单与参数/03-04-ma-menu-and-switch-cookbook.md)；「先自动参数后显式声明」与互斥机制见 [02-02 §6/§7](02-工程组织与工作流/02-02-ma-workflow-habits.md)；版本行为级踩坑见 [07-05 §1/§2](07-验证与排查/07-05-ma-gotchas-version-toolchain.md)；工程怎么建/开/AI 怎么接见 [01-06](01-环境与工具链/01-06-project-launch-and-mcp-bridge.md)。

## 使用前提（本机现状）

- 本机**已装好 Unity 编辑器**：`<VRCHAT_ROOT>\tools\Unity\2022.3.22f1\Editor\Unity.exe`（**2022.3.22f1c1**，Personal 授权已验证可用），另有 Unity Hub 3.22.3、ALCOM 1.1.8 + vrc-get 1.9.2、Windows-MCP。**不需要再装 Unity。** 详见 [01-01 §0.1](01-环境与工具链/01-01-environment-and-toolchain.md)。
- `mcp__unity__*` 的前提是「工程已在编辑器里打开」**且编辑器端跑的是 stdio 桥（`127.0.0.1:6400`）**；在此之前只能做 `STATIC_SOURCE` 级静态检查，需要编辑器的步骤标 `MCP_REQUIRED`。`MCP-FOR-UNITY: Session connected` 只代表 HTTP 链路——判据与修法见 [01-06 §4/§5](01-环境与工具链/01-06-project-launch-and-mcp-bridge.md)。
- 现役改模工程 `<VRCHAT_ROOT>\projects\LuzzyMeow` 是 2026-10-17 从模板新建的，`ProjectVersion.txt=2022.3.22f1c1`，与本机编辑器一致；开工程一律走 ALCOM（[01-06 §2.3](01-环境与工具链/01-06-project-launch-and-mcp-bridge.md)）。旧版 2022.3.62f3c1 的 LuzzyMeow 已移入 `_backups`（可逆）。
- ⚠️ 若日后打开 `ProjectVersion.txt` 是 2022.3.62f3c1 的旧工程：它比 VRChat 官方要求的 2022.3.22f1 新，官方明确别升级；用 2022.3.22f1 打开会弹升级提示——**先问用户，不要点确认**（[01-01 §3](01-环境与工具链/01-01-environment-and-toolchain.md)）。
- 「构建」与「上传」是人机分工：账号登录、Build & Test、Build & Upload、面板二次确认交给用户手工点击（SKILL 铁律 3）。SDK 3.10 起脚本上传入口被移除（[06-06 §2.1](06-构建上传与备份/06-06-upload-credentials-and-client-evidence.md)）。
- 教程类分册（01-02/01-03/01-04/03-02/03-03）与工具链类分册（01-05/05-02/03-04）来自外部视频教程；**教程口径随工具版本漂移，转写会写错专有名词**——照做前先用工程实际组件/菜单核对，对不上标 `STALE`。来源见 [../ACKNOWLEDGEMENTS.md](../ACKNOWLEDGEMENTS.md)。
- `05-03` §9 的换头方法是外部教程口径（非本机实测）；引用时按「教程说法」标注。来源见 [../ACKNOWLEDGEMENTS.md](../ACKNOWLEDGEMENTS.md)。
- MA 规范类分册（02-02/02-03/07-05）整理自外部 MA 实战规范（MIT）；上游写的是 OpenCode 的 `unityMCP_*` 工具，本包前缀 `mcp__unity__*` 一一对应；字段值照用前按 [07-05 §3/§4](07-验证与排查/07-05-ma-gotchas-version-toolchain.md) 核对本机 `Packages/vpm-manifest.json` 与 `packages-lock.json`。

## 待补（已知缺口，下次遇到就补）

- **外部教程里那几套做法本机都还没复现**（见 [02-09 §11](02-工程组织与工作流/02-09-project-workflow-notes.md)）：FX 层 Entry 关键帧删除、MA Menu Installer 组织、git 回档、瞳色菜单、FaceEmo 手势表情、RK Player / AudioLink 安装——下次真正动手做其中一个时，把实测结果（含 Console 原文与截图判据）补回对应小节，并把「未验证」改成「本工作区实测 YYYY-MM-DD」。
- **换头 / 头部移植**只积累了 Kipfel 1.2.0 与 Shipilka v1.02 两个样本的解剖数据（[05-03](05-动骨与穿模/05-03-model-anatomy-and-head-swap.md)），尚未真正做过一次；Merge Armature 同名骨冲突处理、面捕插件对接未实测；§9 三支视频做法未本机复现。
- **工程逆向**：AssetRipper 2.0 载入/导出与 Unity 打开已实测（[06-03 §14/§23](06-构建上传与备份/06-03-recover-project-from-uploaded-avatar.md)）；但 FACS 12 步、AvatarRecovery 1.2.22 / dVRC 1.1.2 / VRCALoader 在 Unity 内的实际恢复、缓存 __data 的 AES 解密都还没走通；下次拿一个自己上传的小模型走完整链路。
- 成人向目录 `07_成人向` 相关插件的**新版**字段变化（PCS/SPS）只有基底方法、无版本快照；遇到新版本再补。
- **原生 FT 的 faceTrackingParameterBehaviors 门控**（[04-09 §12.1](04-外观与表情/04-09-face-tracking-native-sdk.md)）未实测；GM 全身 Humanoid 播放受环境限制（[07-06 §23.3](07-验证与排查/07-06-gesture-manager-automated-sim-test.md)）。

## 旧名映射

历史旧文件名与新名的对照表已移至 [../ACKNOWLEDGEMENTS.md](../ACKNOWLEDGEMENTS.md) 的「旧名映射」节。正文交叉引用已全部改用新编号。

## 原 08-症状速查 的结构化去向（2026-11-11）

> 原 9 册已删除。全部内容按主题转化如下；各落点小节带「原 08-症状速查 第 N 轮」来源标注，实测数字、命令、路径、判据与 NOT_RUN 口径均保留。

| 原册 | 内容范围 | 结构化去向 |
|---|---|---|
| 00-主表.md | 2026-10-17 起 204 行症状表 | 绝大多数行本就是指向主题册的路由（目标册已含全文）；独有细节并入：05-11（SMR Transform/LoadAllAssetsAtPath/眼镜名牌）、07-09（Play bounds/侧视/模态框）、06-06（API 死锁）、03-01 §10（saved/默认值）、04-06 §14（OT 极性/自锁/三轮定案）、03-06 §19（一键穿衣/分组根） |
| 01-增补-2026-10-29至10-31.md | 第 123–135 轮 + 历史补充 | 07-01 §29–§36（已有）；03-06 §13/§14/§18.4/§18.5/§19.9/§19.11；05-08 §18–§27（已有）；05-11 §4/§12；05-01 §15.3；07-09 §6.2/§6.6；04-08 §7.5；06-03 §22（已有） |
| 02-增补-2026-11-05至11-06.md | 第 137–154 轮 | 07-06 §19/§20（已有）；04-06 §12/§13/§14；05-01 §12–§15；03-06 §15/§19；03-01 §10；04-08 §1–§4/§6/§7；02-11 §1/§2；06-06 §1/§3/§6；07-09 §1/§6 |
| 03-增补-2026-11-07.md | 第 156–159 轮 | 04-08 §3/§5/§6；04-09（原生 FT 部分）；03-06 §18/§19.12；02-11 §1.3/§1.4/§2.2/§2.3；07-09 §1/§2/§3/§4；07-06 §23.1；06-06 §3.3/§5；06-05（对齐） |
| 04-增补-2026-11-08.md | 第 160–172 轮 | 04-08 §2/§3.8/§5；04-09 §1–§10/§13；01-01 §8；06-06 §2/§4；06-05 §11；03-01 §10.5/§10.6；03-06 §18.3/§19.3–§19.6；02-11 §1.5–§1.7/§3；05-11 §1/§4/§9；05-10 §21（已有）；07-06 §23.5；07-09 §1.9 |
| 05-增补-2026-11-09.md | 第 172 轮 | 02-11 §3.1/§1.5；05-10 §21（已有）；05-11 §4；03-06 §17（已有）；06-05 §9（已有）；06-06 §6.3 |
| 06-增补-2026-11-10.md | 第 173–174 轮 | 05-10 §22/§23（已有）；01-06 §13（已有+补充）；06-05 §10（已有）；06-06 §4.6/§6.1；07-09 §3/§7.5/§8.3；03-01 §10.5 |
| 07-增补-2026-11-10.md | 第 175 轮 | 07-09 §6.5/§8.1/§8.2；03-06 §19.7；07-06 §19（已有）；06-05 §8（已有）；06-06 §6.4/§2.4；07-01 §25.5（已有） |
| 08-增补-2026-11-11.md | 第 176–177 轮 + 175 轮上传实证 | 04-09 §8/§12.2；03-06 §19.8；07-09 §6.3/§1.10/§2/§5/§7.1；06-06 §1.4/§1.5/§2.3/§2.4；05-11 §8/§7；04-06 §14.1；03-01 §10.4/§10.2；07-06 §23.3/§23.4；02-11 §2.4 |

## 2026-11-12 增量索引（PLAN-001，小狼）

册数不变，只向既有主题册增量追加：

| 主题 | 落点 | 依据与边界 |
|---|---|---|
| 抽取Action只放歌、AFK覆盖、走跑冻结、VRCFury owner展开与权重来源 | [03-07 §14](03-菜单与参数/03-07-ma-platform-sync-mmd-and-motion.md#14-抽取舞蹈-action-后只放歌afk-覆盖与走跑冻结2026-11-12) | 当前包源码 + 真实GM逐菜单/权重/骨骼运动；07-06 §23.3已加取代边界，客户端另验 |
| 合并衣装拆上下装、原生单位、插点腰线与全通道蒙皮保留 | [05-11 §13](05-动骨与穿模/05-11-skinning-binding-and-geometry-repair.md#13-合并衣装拆为独立上下装原始单位连通岛与共享腰线2026-11-12) | 实际Mesh JSON/资产及双路复核；不能把展示缩放或文件生成当通过 |
| NDMF持久化生成根保留 | [02-05 §10](02-工程组织与工作流/02-05-unity-project-asset-reorg.md#10-ndmf-持久化生成根应按当前源码保留2026-11-12) | NDMF1.14.8 ManualProcessAvatar源码 + MoveAsset GUID集合复核 |
| 面捕眯眼高光消失：旧归零补丁导致虹膜遮挡 | [04-09 §15](04-外观与表情/04-09-face-tracking-native-sdk.md#15-高光归零补丁引起的虹膜遮挡与按眼补偿) | 88真实高光点随虹膜移动、4误伤点恢复legacy；源mesh/烘焙/相机矩阵实测，客户端仍另验；§8已加取代边界 |
| SDK3.10.6公开BuildAndUpload与旧入口废弃的边界 | [06-06 §2.1](06-构建上传与备份/06-06-upload-credentials-and-client-evidence.md#21-脚本上传入口被移除) | 官方Public SDK API+当前包源码；旧ExportAndUpload废弃不等于禁止新API，GUI为回退非唯一 |
