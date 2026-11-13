# 来源、上游与许可说明（ACKNOWLEDGEMENTS）

本文件集中记录本知识库所参考的上游仓库、教程系列、官方文档与第三方资产来源。**知识库正文只保留技术内容与引用 URL，作者名、UP 名、合集名、BV 号等来源标识集中在本文件**。引用 URL 在正文中保持原样，便于逐条核对。

> 说明：本库在下列来源基础上进行了大规模重组、补写、实测与结构化；技术事实与原始表达仍归属对应来源。`00-基底/` 7 册保留上游原文（MIT），其余分册为改写、扩写或本工作区自产。若需商业发布或主张完全原创，请先完成对应分册的重写或取得授权。

## 1. 上游 skill 仓库（MIT）

根目录 `LICENSE` 为上游 MIT 许可（Copyright (c) 2026 URaux）。以下仓库均为 MIT，保留版权与许可声明即可再分发/修改：

| 仓库 | 版权行 | 本库使用方式 |
|---|---|---|
| URaux/vrchat-avatar-modding-skill | Copyright (c) 2026 URaux | `00-基底/` 7 册原文保留；`SKILL.md` 的铁律与流程框架 |
| felixchaos/vrchat-avatar-modding-skill | Copyright (c) 2026 Felix | 工作资产结构、编辑器自动化 C# 范式、参数/烘焙/默认值审计、SDK 上传与版本管理、故障排查的整合改写 |
| NeonSaya/VRCAvatarOutfitMenuSkill | Copyright (c) 2026 NeonSaya | 备份与打包、颜色档位与 Puppet、表情曲线与图标、lilToon 外观、服装接入与穿模几何验收、MCP 调用纪律的整合改写 |
| XiaoboooOvO/VRChatEditorSkill | Copyright (c) 2026 XiaoboooOvO | 证据阶梯与授权矩阵、只读审计契约、provider 归属与生成输出所有权、工具链选择门、FaceEmo/lilycalInventory/MA playbook、Unity MCP 门控的整合改写 |

上游仓库原件位于包外 `<VRCHAT_ROOT>/tools/sources/vrchat-avatar-modding/upstream/`（不随本包分发）。

## 2. sthsmall/vrc-modding（MIT，v2.5.0，2026-09-02）

- 仓库：`sthsmall/vrc-modding`（OpenCode skill，MIT；作者自述合并自 vrc-model-modding 与 modular-avatar，基于 Milfy 工程实战）。
- 使用范围：`02-02-ma-workflow-habits.md`、`02-03-ma-component-guide-workflows.md`、`07-05-ma-gotchas-version-toolchain.md` 的组织习惯、组件决策、任务工作流、10 条常见坑、MA 1.18.7 字段实测、来源权威顺序、Unity MCP 读写序列、安全授权与 4 个只读脚本。
- 说明：按用户要求，源仓库副本已在吸收完成后从本机删除；上述三册是全部可复用结论。
- 版权行：Copyright (c) sthsmall（MIT）。
- 字段级实测工程：`catlee.kuuta`（2026-09-02）：MA 1.18.7（embedded）、NDMF 1.14.8、VRCSDK 3.10.4、Unity 2022.3.22f1（见 `07-05` §2）。

## 3. 视频教程系列（B 站）

### 3.1 VRLearn（UP：翱翔大使）

- 空间：<https://space.bilibili.com/2796549>
- 配套文档站：<https://start.gamemode1.com>、图文版 <https://vrclearn.gamemode1.com>
- 使用范围：`01-02`（环境/工具链/SDK/Unity 基础）、`01-03`（SDK 面板全字段/报错对照/网络/安装）、`01-04`（界面/变量/周边工具）、`03-02`（手搓控制器/MA 快捷菜单/粒子/绑骨）、`03-03`（两个实战案例）。
- 对应视频（正文已去 BV，保留核对用清单）：
  - [BV1pjSYBsEXX](https://www.bilibili.com/video/BV1pjSYBsEXX/)（镜像下载失败时手动装包）
  - [BV1UsBjB9ELT](https://www.bilibili.com/video/BV1UsBjB9ELT/)（换装菜单：动画控制器与变量，16:25）
  - [BV1x7FdzEEcL](https://www.bilibili.com/video/BV1x7FdzEEcL/)（MA 菜单，4:59）
  - [BV1pqrUB2EQa](https://www.bilibili.com/video/BV1pqrUB2EQa/)（粒子眼部特效，11:49）
  - [BV1jtUVBHEFN](https://www.bilibili.com/video/BV1jtUVBHEFN/)（更换衣物，18:51）
  - [BV1GXV96CEwM](https://www.bilibili.com/video/BV1GXV96CEwM/)（眼睛贴图专用，无字幕轨）
  - [BV1hKZCBnE2r](https://www.bilibili.com/video/BV1hKZCBnE2r/)（看片方案，无字幕）
  - [BV1f7XLB6E5s](https://www.bilibili.com/video/BV1f7XLB6E5s/)（修复消失部位，无字幕）
  - [BV1vV3R6GEw1](https://www.bilibili.com/video/BV1vV3R6GEw1/)（世界/地图，无字幕）
  - [BV1NvuvzUEch](https://www.bilibili.com/video/BV1NvuvzUEch/)（预告）
- 采集方式：B 站字幕（12 条 VRC 相关视频有字幕）＋站点首页/资源页/图文版介绍页；图文版深链页面抓取被拒。

### 3.2 碱性芬达FDOH「VRC改模」系列

- mid：361709024；合集：season_id 9069114；7 条，人工 CC 中文字幕。
- 使用范围：`03-04`（MA 菜单与开关实操手册：菜单顺序/Extract Menu、int 开关、Material Setter 换色、Radial Puppet、动骨参数触发、捏脸数据）。
- 视频：[BV1DrY26qESk](https://www.bilibili.com/video/BV1DrY26qESk/)、[BV1Y7fZBAEJv](https://www.bilibili.com/video/BV1Y7fZBAEJv/)、[BV1QdzCB3E4W](https://www.bilibili.com/video/BV1QdzCB3E4W/)、[BV1zuc6zaEHV](https://www.bilibili.com/video/BV1zuc6zaEHV/)、[BV1BLApzHETP](https://www.bilibili.com/video/BV1BLApzHETP/)、[BV1CTjt6FEQx](https://www.bilibili.com/video/BV1CTjt6FEQx/)、[BV1P2zQBnE5M](https://www.bilibili.com/video/BV1P2zQBnE5M/)。

### 3.3 咸鱼子Luna「宝宝向教程」系列

- mid：3546752913247086；合集：season_id 4023913；7 条，AI 中文字幕（转写可能写错专有名词，以 Unity 界面实际字段为准）。
- 使用范围：`01-05`（工具链/素材/上传）、`05-02`（绑骨录动画/动骨制作/fbx 替换网格）。
- 视频：[BV1SfmNYREWU](https://www.bilibili.com/video/BV1SfmNYREWU/)、[BV1UeDaYqE4u](https://www.bilibili.com/video/BV1UeDaYqE4u/)、[BV1nhUuYWEwt](https://www.bilibili.com/video/BV1nhUuYWEwt/)、[BV1sAdHY2ECn](https://www.bilibili.com/video/BV1sAdHY2ECn/)、[BV1exMHzWEbn](https://www.bilibili.com/video/BV1exMHzWEbn/)、[BV1REQqBxEzQ](https://www.bilibili.com/video/BV1REQqBxEzQ/)、[BV1GCQiBHEru](https://www.bilibili.com/video/BV1GCQiBHEru/)。

### 3.4 Rosmontis_Cloud（牢云）VRC 教程系列

- 空间：`space.bilibili.com/622981359`；9 条 VRC 教程。
- 使用范围：`02-09`（工程流程、菜单组织、git 回档、妆容与瞳色、FaceEmo、RK Player/AudioLink、Blender 入门）。
- 视频：[BV1PPE46cEuu](https://www.bilibili.com/video/BV1PPE46cEuu/)（EP1）、[BV16EEh6KEeC](https://www.bilibili.com/video/BV16EEh6KEeC/)（EP2）、[BV16fJ56ZELg](https://www.bilibili.com/video/BV16fJ56ZELg/)（EP3）、[BV1xajk6uEBw](https://www.bilibili.com/video/BV1xajk6uEBw/)（EP4）、[BV1JWTK6iELJ](https://www.bilibili.com/video/BV1JWTK6iELJ/)（EP6）、[BV17qTy63EEg](https://www.bilibili.com/video/BV17qTy63EEg/)（EP7）、[BV1nEj868EDK](https://www.bilibili.com/video/BV1nEj868EDK/)（你问我答：RK Player）、[BV1tjJp6bEm3](https://www.bilibili.com/video/BV1tjJp6bEm3/)（你问我答：AudioLink）、[BV1cLNG6JERj](https://www.bilibili.com/video/BV1cLNG6JERj/)（地图制作 EP1）。
- 采集方式：AI 字幕 + 评论（含二级评论）；**属外部教程转录，除标注外按「未验证」对待**。

### 3.5 VRCAL / 八云綾

- 视频：[BV1KcJd6UEFA](https://www.bilibili.com/video/BV1KcJd6UEFA/)（「不用再挨个查！我做了个自动匹配商品的 VRChat 素材库」，2:05）。
- 仓库：<https://github.com/NecoAya/VRCAL-VRChat-Asset-Library>（GPL-3.0-only，代码；商品封面/素材内容不在开源授权内）。
- 使用范围：`02-07`。

### 3.6 换头教程三支（外部方法，未本机复现）

- [BV1jn4y197aP](https://www.bilibili.com/video/BV1jn4y197aP/)（VRchat改模102 换头，41:22）
- [BV1cbxXzvEpX](https://www.bilibili.com/video/BV1cbxXzvEpX/)（适用于 VRC 的 Unity 换头教程，4:40，无字幕轨）
- [BV1eAGu6LETN](https://www.bilibili.com/video/BV1eAGu6LETN/)（换头插件与自手绑，16:17）
- 使用范围：`05-03` §9。

### 3.7 其他引用视频（恢复 / 防盗模 / 工具演示）

- 恢复与颜色同步演示：[BV1taJh6nEJ9](https://www.bilibili.com/video/BV1taJh6nEJ9/)（AvatarRecovery 演示）、[BV1daJt6XE5q](https://www.bilibili.com/video/BV1daJt6XE5q/)（NoNlab 颜色同步演示）。
- 防盗模 / 封号 / 工具：[BV1Bdm2B7E8c](https://www.bilibili.com/video/BV1Bdm2B7E8c/)（Kanna Protecc）、[BV1QoqhBNE47](https://www.bilibili.com/video/BV1QoqhBNE47/)、[BV1px4y1u7Xy](https://www.bilibili.com/video/BV1px4y1u7Xy/)（盗模原理）、[BV1GLXNBAE8r](https://www.bilibili.com/video/BV1GLXNBAE8r/)（封号潮）、[BV1cAJb6YEH8](https://www.bilibili.com/video/BV1cAJb6YEH8/)（EAC 更新）、[BV14BRABxENS](https://www.bilibili.com/video/BV14BRABxENS/)（动画绑定修复）、[BV129t6z1Eih](https://www.bilibili.com/video/BV129t6z1Eih/)（AssetRipper Qt UI）、[BV1f7XLB6E5s](https://www.bilibili.com/video/BV1f7XLB6E5s/)（部位修复）、[BV1uMReB7EiC](https://www.bilibili.com/video/BV1uMReB7EiC/) / [BV1V6Ki6VEmv](https://www.bilibili.com/video/BV1V6Ki6VEmv/)（AvatarObfuscator）、[BV18qnu6hEEf](https://www.bilibili.com/video/BV18qnu6hEEf/)（NoNlab V2.0）。
- 其他工具/流程演示：[BV1qyYszNE3G](https://www.bilibili.com/video/BV1qyYszNE3G/)（预制件是否完全解压）、[BV1HEgTegEPM](https://www.bilibili.com/video/BV1HEgTegEPM/) / [BV19kgeeBEuX](https://www.bilibili.com/video/BV19kgeeBEuX/) / [BV1pe68YYEQb](https://www.bilibili.com/video/BV1pe68YYEQb/)（本地测试资产包 + AssetRipper + FACS 三集）、[BV1NH4y1n7Ba](https://www.bilibili.com/video/BV1NH4y1n7Ba/)（缓存恢复）、[BV11o8T6GE4r](https://www.bilibili.com/video/BV11o8T6GE4r/)（BlenderSafeAvatarFbxExporter）、[BV1HCKAzGEnh](https://www.bilibili.com/video/BV1HCKAzGEnh/)（Module Creator）、[BV1zuHC6rEMo](https://www.bilibili.com/video/BV1zuHC6rEMo/)（模型密码找回）、[BV1HvaA6ZEPq](https://www.bilibili.com/video/BV1HvaA6ZEPq/)（闭源 .vrca 上传，不推荐）。
- 使用范围：`06-03`（恢复链路、防盗模与工具全景）。

## 4. 官方文档与工具

正文中保留全部引用 URL；以下为集中索引：

- Modular Avatar：<https://modular-avatar.nadena.dev/docs/intro>（本库对齐 67 页，MA 1.18.7）
- VRChat Creation：<https://creators.vrchat.com/>（PhysBones、Face Tracking、OSC、Animator Parameters、Avatar Performance Ranking、VRCPipelineManager 等）
- VRCFaceTracking：<https://docs.vrcft.io/>
- lilToon：<https://lilxyzw.github.io/lilToon/>（MIT）
- Unity Manual / Scripting API：<https://docs.unity3d.com/2022.3/Documentation/>
- BOOTH 商品页：正文按需保留（如 RealKissSystem <https://booth.pm/ja/items/5354575>、ネガライン <https://booth.pm/ja/items/6867118>）

## 5. 第三方许可全文（MIT）

以下 MIT 文本适用于本库中源自 URaux、Felix、NeonSaya、XiaoboooOvO、sthsmall 的部分；各版权行见上文。根目录 `LICENSE` 保留 URaux 的 MIT 声明。

```text
MIT License

Copyright (c) 2026 URaux
Copyright (c) 2026 Felix
Copyright (c) 2026 NeonSaya
Copyright (c) 2026 XiaoboooOvO
Copyright (c) sthsmall

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## 6. 维护约定

- 新增内容若参考了新的上游/教程/官方文档，只在本文件追加一条来源与 URL，正文不再写作者名/UP 名/合集名/BV 号。
- 正文引用 URL 保持原样，不因去署名而删除。
- 若某册经过完整重写、已不含上游表达，仍建议在本文件保留「曾参考」记录，避免社区误判。

## 7. 逐册来源映射

| 来源 | 本包位置 | 主要贡献 |
|---|---|---|
| URaux/vrchat-avatar-modding-skill（MIT） | `SKILL.md` + `00-基底/`（原文保留 + 补充节） | 铁律、MA 组件范式、PhysBone 实测配方、面捕、穿模、上传报错 |
| felixchaos/vrchat-avatar-modding-skill（MIT） | 整合改写（包外原件） | 工作资产结构、编辑器自动化 C# 范式、参数/烘焙/默认值审计、SDK 上传与版本管理、故障排查 |
| NeonSaya/VRCAvatarOutfitMenuSkill（MIT） | 整合改写（包外原件） | 备份与打包、颜色档位与 Puppet、表情曲线与图标、lilToon 外观、服装接入与穿模几何验收、MCP 调用纪律 |
| XiaoboooOvO/VRChatEditorSkill（MIT） | 整合改写（包外原件） | 证据阶梯与授权矩阵、只读审计契约、provider 归属与生成输出所有权、工具链选择门、FaceEmo/lilycalInventory/MA playbook、Unity MCP 门控 |
| sthsmall/vrc-modding（MIT） | `02-02` / `02-03` / `07-05` | MA 化标准流程与操作习惯、组件决策指南 + 工作流 A–H、10 条常见坑、字段级实测与版本快照、来源权威顺序、官方链接、MCP 读写序列、安全授权、4 个只读脚本 |
| VRLearn（翱翔大使） | `01-02` / `01-03` / `01-04` / `03-02` / `03-03` | 中文从零链路：工具链与国内镜像、SDK 面板与上传报错、Unity 基础与工程组织、手动/MA 绑骨、手搓动画控制器与菜单、粒子眼部特效、形态键被占用排查、外观贴图、周边工具 |
| 碱性芬达FDOH「VRC改模」 | `03-04` | MA 逐组件配方：菜单优先级/Extract Menu、int 开关、Material Setter 换色、Radial Puppet、_IsGrabbed/_IsPosed 触发动画、Blend Shape to Animation |
| 咸鱼子Luna「宝宝向教程」 | `01-05` / `05-02` | 宝宝向全流程：ALCOM/lilEditorToolbox/lilycalInventory/如梦工具箱、素材导入判据、上传前清单与性能等级、模型精简 + 衣柜转移 + 录动画换色、从 0 建骨骼刷权重 + PhysBone 全参数、fbx 替换网格 |
| Rosmontis_Cloud（牢云）VRC 教程系列 | `02-09` | 外部教程笔记与 KB 比对：FX Entry 覆盖根因、MA Menu Installer 组织、git 回档与 detached HEAD、妆容三层与瞳色菜单、FaceEmo、RK Player/AudioLink、Blender 入门、自建 VPM 镜像 |
| 换头教程三支（BV1jn4y197aP / BV1cbxXzvEpX / BV1eAGu6LETN） | `05-03` §9 | Unity-only 换头全流程、Gesture 层与 FX 分工、Root Bone/视角球两个隐形杀手、脖子接缝三条路（外部来源，未本机复现） |
| MA 官方文档（67 页，MA 1.18.7） | `02-10` / `03-07` / `04-06` / `05-10` | 官方逐页语义补齐：Setup Outfit / 烘焙改写 / Manual Bake / 错误面板 / MergeAnimator 层序 / MMD / SyncParameterSequence / 反应式家族 / 17 骨骼几何组件 |
| VRChat 官方文档 | `00-03` / `00-04` 补充节、`04-08` / `04-09` / `03-01` / `06-01` / `06-06` | PhysBones 官方语义与限额、原生 FT 字段与 gaze 通道、内置参数、blueprintId 隔离 |
| VRCFaceTracking 官方文档 / Pawlygon README | `00-04` 补充节、`04-08` | 眼部 18 键语义、眼球旋转应骨驱动、EyeLid 0=闭/0.75=正常睁 |
| lilToon 官方文档（MIT） | `04-01` 补充节 | shader 变体、贴图槽位、构建期自动优化、版本与分发 |
| 工作区自产实测（2026-10-17 至 11-11，第 1–177 轮） | `01-06` / `02-04`–`02-06` / `02-11` / `03-05`–`03-06` / `04-05`–`04-09` / `05-03`–`05-11` / `06-03`–`06-06` / `07-06`–`07-09` 及 `07-01` 各追加节 | 本机环境与 AI 链路、GM 自动化、多角度验证、一键穿衣演进、面捕 OSC/原生接入与修复、上传凭据与客户端取证、蒙皮与几何修复、插件集成；原 08-症状速查 9 册已结构化并入这些册 |

## 8. 旧名映射（2026-11-11 统一命名）

> 历史旧文件名与新名对照。正文交叉引用已全部改用新编号 `NN-MM`；本表仅作改名追溯。

| 旧文件名 | 新文件名 |
|---|---|
| 00-基底/editor-automation.md | 00-基底/00-01-editor-automation.md |
| 00-基底/modular-avatar.md | 00-基底/00-02-modular-avatar.md |
| 00-基底/physbone-dynamics.md | 00-基底/00-03-physbone-dynamics.md |
| 00-基底/face-tracking.md | 00-基底/00-04-face-tracking.md |
| 00-基底/mesh-blendshape.md | 00-基底/00-05-mesh-blendshape.md |
| 00-基底/plugin-ecosystem.md | 00-基底/00-06-plugin-ecosystem.md |
| 00-基底/troubleshooting.md | 00-基底/00-07-troubleshooting.md |
| environment-and-toolchain.md | 01-环境与工具链/01-01-environment-and-toolchain.md |
| vrlearn-lessons-01-environment-and-sdk.md | 01-环境与工具链/01-02-environment-and-sdk.md |
| vrlearn-lessons-04-upload-errors-and-toolchain.md | 01-环境与工具链/01-03-upload-errors-and-toolchain.md |
| vrlearn-lessons-05-unity-ui-and-extra-tools.md | 01-环境与工具链/01-04-unity-ui-and-extra-tools.md |
| vrcmod-extra-02-babystep-toolchain-and-upload.md | 01-环境与工具链/01-05-babystep-toolchain-and-upload.md |
| local-01-project-launch-and-mcp-bridge.md | 01-环境与工具链/01-06-project-launch-and-mcp-bridge.md |
| workspace-and-work-assets.md | 02-工程组织与工作流/02-01-workspace-and-work-assets.md |
| vrcmod-extra-04-sthsmall-ma-workflow-habits.md | 02-工程组织与工作流/02-02-ma-workflow-habits.md |
| vrcmod-extra-05-sthsmall-component-guide-workflows.md | 02-工程组织与工作流/02-03-ma-component-guide-workflows.md |
| local-09-vrcfury-write-defaults-dialog.md | 02-工程组织与工作流/02-04-vrcfury-write-defaults-dialog.md |
| local-08-unity-project-asset-reorg.md | 02-工程组织与工作流/02-05-unity-project-asset-reorg.md |
| local-03-resource-library-reorg.md | 02-工程组织与工作流/02-06-resource-library-reorg.md |
| local-07-vrcal-asset-library.md | 02-工程组织与工作流/02-07-vrcal-asset-library.md |
| local-06-cross-avatar-clothing.md | 02-工程组织与工作流/02-08-cross-avatar-clothing.md |
| local-04-rosmontis-cloud-vrc-course.md | 02-工程组织与工作流/02-09-project-workflow-notes.md |
| local-10-ma-setup-outfit-and-general-behavior.md | 02-工程组织与工作流/02-10-ma-setup-outfit-and-general-behavior.md |
| parameter-and-menu-audit.md | 03-菜单与参数/03-01-parameter-and-menu-audit.md |
| vrlearn-lessons-02-animator-menu-particles.md | 03-菜单与参数/03-02-animator-menu-particles.md |
| vrlearn-lessons-03-case-study-and-sdk-panel.md | 03-菜单与参数/03-03-case-study-and-sdk-panel.md |
| vrcmod-extra-01-ma-menu-and-switch-cookbook.md | 03-菜单与参数/03-04-ma-menu-and-switch-cookbook.md |
| local-05-ma-menu-hair-style-color.md | 03-菜单与参数/03-05-ma-menu-hair-style-color.md |
| local-06-ma-menu-build-by-reflection.md | 03-菜单与参数/03-06-ma-menu-build-by-reflection.md |
| local-07-ma-platform-sync-mmd-and-motion.md | 03-菜单与参数/03-07-ma-platform-sync-mmd-and-motion.md |
| appearance-and-liltoon.md | 04-外观与表情/04-01-appearance-and-liltoon.md |
| expressions-and-icons.md | 04-外观与表情/04-02-expressions-and-icons.md |
| color-and-puppet.md | 04-外观与表情/04-03-color-and-puppet.md |
| local-04-native-face-tracking-two-heads.md | 04-外观与表情/04-04-native-face-tracking-two-heads.md |
| local-05-wolf-face-gesture-expressions.md | 04-外观与表情/04-05-wolf-face-gesture-expressions.md |
| local-06-ma-reactive-components.md | 04-外观与表情/04-06-ma-reactive-components.md |
| local-07-vrcft-parameter-prefix-repair.md | 04-外观与表情/04-07-vrcft-parameter-prefix-repair.md |
| outfit-fit-and-clipping.md | 05-动骨与穿模/05-01-outfit-fit-and-clipping.md |
| vrcmod-extra-03-bind-animation-physbone-fbx.md | 05-动骨与穿模/05-02-bind-animation-physbone-fbx.md |
| local-02-model-anatomy-and-head-swap.md | 05-动骨与穿模/05-03-model-anatomy-and-head-swap.md |
| local-03-cross-avatar-accessory-transfer.md | 05-动骨与穿模/05-04-cross-avatar-accessory-transfer.md |
| local-11-head-swap-teardown-and-clone.md | 05-动骨与穿模/05-05-head-swap-teardown-and-clone.md |
| local-12-sps-and-ndick-binding.md | 05-动骨与穿模/05-06-sps-and-ndick-binding.md |
| local-13-tail-gloves-migration.md | 05-动骨与穿模/05-07-tail-gloves-migration.md |
| local-14-cross-avatar-garment-fit-newfit.md | 05-动骨与穿模/05-08-cross-avatar-garment-fit-newfit.md |
| local-15-third-party-tool-prefab-binding-abt.md | 05-动骨与穿模/05-09-third-party-tool-prefab-binding-abt.md |
| local-16-ma-bone-and-geometry-components.md | 05-动骨与穿模/05-10-ma-bone-and-geometry-components.md |
| sdk-upload-and-versioning.md | 06-构建上传与备份/06-01-sdk-upload-and-versioning.md |
| backup-and-package.md | 06-构建上传与备份/06-02-backup-and-package.md |
| local-05-recover-project-from-uploaded-avatar.md | 06-构建上传与备份/06-03-recover-project-from-uploaded-avatar.md |
| local-06-sdk-publish-with-reflection.md | 06-构建上传与备份/06-04-sdk-publish-with-reflection.md |
| local-07-avatar-package-version-collision.md | 06-构建上传与备份/06-05-avatar-package-version-collision.md |
| bake-and-validation.md | 07-验证与排查/07-01-bake-and-validation.md |
| evidence-and-ownership.md | 07-验证与排查/07-02-evidence-and-ownership.md |
| provider-playbooks.md | 07-验证与排查/07-03-provider-playbooks.md |
| verification-discipline.md | 07-验证与排查/07-04-verification-discipline.md |
| vrcmod-extra-06-sthsmall-gotchas-version-toolchain.md | 07-验证与排查/07-05-ma-gotchas-version-toolchain.md |
| local-08-gesture-manager-automated-sim-test.md | 07-验证与排查/07-06-gesture-manager-automated-sim-test.md |
| local-09-scene-bone-corruption-recovery.md | 07-验证与排查/07-07-scene-bone-corruption-recovery.md |
| local-10-multi-angle-binding-verification.md | 07-验证与排查/07-08-multi-angle-binding-verification.md |
