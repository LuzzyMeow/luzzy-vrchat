# VRCAL：VRChat 素材库（自动匹配 BOOTH 商品）

> 一个把本地 VRChat 素材扫描成可搜索封面图库、并自动匹配 BOOTH 商品的 Windows 桌面应用（来源信息见 `ACKNOWLEDGEMENTS.md`）。
> 仓库：https://github.com/NecoAya/VRCAL-VRChat-Asset-Library
> 版本：1.10.3（视频发布时）
> 许可证：GPL-3.0-only（代码；商品封面/素材内容不在开源授权内）
> 状态：已完整克隆源码与文档，尚未安装/运行应用；本文是知识库摘要，不是实测报告。

## 0. 去重说明

现有 KB 没有 VRCAL 条目；与下面几册不重复，只做交叉引用：

- 02-06-resource-library-reorg.md：我们自己的 resources 文件目录重组规则（10 个一级目录、junction、去重）。
- VRCAL 是只读索引/图库/商品匹配工具，不搬动、不删除、不重命名原素材，也不替代 resources 的目录规范。
- 本册只记录 VRCAL 自身功能、匹配逻辑、数据/安全边界和源码结构；通用素材库规则仍以 02-06 为准。

## 1. 它是什么

Windows x64 桌面应用（Electron + React + TypeScript），把本地散落的 VRChat 素材扫描成可搜索的封面图库，并自动匹配 BOOTH 商品（优先编号，其次作者/名称规则）；匹配不确定时在内置 BOOTH 浏览器里人工确认。

- 默认扫描 E:\VRChat（本机工作区在 <VRCHAT_ROOT>，使用时要改选）
- 资料库保存位置必须在 C 盘以外、且不在素材扫描母目录内
- 原素材与 Unity 工程保持原样；程序只登记引用、封面和整理信息
- 当前发布包未做代码签名；从 GitHub Releases 下载并核对 SHA-256

## 2. 主要功能

1. 自动匹配 BOOTH 商品：编号优先；支持作者前缀、版本/副本后缀、商品明确列出的英日名称；日期与编号冲突保守处理。
2. 内置 BOOTH 浏览器：可编辑搜索词，打开商品后「应用此商品」；支持官方网页登录、喜欢、Gift、已购资料库与下载登记。
3. 匹配确认：候选封面按行比较；支持手动填链接/编号、排除、跳过、撤回来源、稳定排序。
4. 分类证据：人工分类优先，其次商品 BOOTH 分类/标题/用途/标签/本地线索；服装内 PNG/Textures/Materials 不会把整套衣装误分到贴图。
5. 本地适配 vs 商品支持分开：灰色只表示没有本地证据，不等于已验证文件缺失。
6. 重复展示合并：同名/编号的单一素材包装目录折叠成一张卡；各层路径与人工标注保留，可展开/右键定位。
7. 图库管理：标签（含别名/合并）、收藏、备注、批量标注、集合、列表、搜索筛选、工程素材线索与人工关联。
8. 路径管理：添加母目录递归扫描；移除路径只停止扫描/显示，原文件、封面和标注保留；Unity 工程作为整体，不被拆成大量卡片。
9. 三语界面：简体中文/日本語/English，切换即时生效；不翻译素材名、路径、作者、人工标签。
10. 四套主题：X 暗色、X 亮色、NASA 朋克、瑞士国际主义；Ctrl+滚轮缩放；主题/语言独立保存。
11. 内置更新：检查/下载 GitHub 新版，校验 SHA-256 后重启安装；失败不破坏旧程序。
12. 网络任务面板：商品读取、下载进度/速度/已接收字节；并发 1–6、暂停/恢复、缓存、重试。
13. 只读归档：ZIP/Unitypackage 在独立进程只读解析；RAR/7z 走系统工具只读列目录；不自动解压、不导入 Unity。
14. 工程素材线索：扫描 Unity 工程 Assets 中的目录/Prefab/模型文件名，和已有素材做名称线索/人工关联；不读取正文上传，不修改工程。
15. 安全边界：没有删除/移动/重命名/修改原素材接口；目录链接和 Unity 缓存不参与索引；联网补全只发送名称/编号等检索线索，不上传素材内容。

## 3. 商品匹配规则

- 编号优先：纯 BOOTH 编号、压缩包名和版本/副本后缀先核对；日期、版本号、编号冲突保守处理。手动来源可输入 1–12 位合法编号。
- 名称规则：清理版本、适配标签、促销装饰、重复标题、Windows 副本后缀；保留有意义的型号/版本差异。作者_商品名的备选检索必须核对商品页作者或店铺别名后才可采用。
- 搜索顺序：先对最多 30 个结果排序，再读取相关商品；同名/证据冲突进入待确认。
- 分类顺序：人工分类 → 商品最细 BOOTH 分类 → 标题/用途 → 标签 → 本地线索。
- 证据：详情「查看名称与分类依据」可看到保存的证据；人工来源、排除、跳过优先，刷新不覆盖人工信息。
- 多商品/多作者/编号冲突：不会自动合并成一款；多个独立包保持独立。
- 升级复评：旧自动结果在开启联网后按队列渐进复评；失败保留已有资料；不会自动重试整库旧结果。

## 4. 数据、隐私与安全

- 资料库位置：首次选择（C 盘外、素材母目录外）；默认在所选上级目录创建 VRChatAssetLibrary。位置同时记录在程序旁和 Windows 账户中，首次设置会重启一次。
- 原素材只读：应用引用原文件；没有删除、移动、重命名、修改原素材的接口。
- BOOTH 登录：用户在官方网页完成登录/验证码；程序不代提交密码。持久会话保存在资料库 booth-session；临时 Cookie 用 Windows 系统加密保存，最长恢复 30 天；不保存明文密码，不在列表/日志/备份显示 Cookie、验证参数或签名下载地址。
- 备份：普通备份只含整理信息、集合、别名、设置、封面、异常记录；不含登录凭证或下载原文件。
- 下载：先在网页选择 C 盘外、原素材目录外的下载目录；每任务独立子目录，不覆盖、不解压、不执行、不导入 Unity。
- 网络：补全用 GET，上传正文为零；素材文件不会上传；不宣称网络协议完全无数据发送；代理检测只读系统配置，不修改代理。
- 解析安全：ZIP/Unitypackage 独立进程只读解析，有时间/内存/条目上限，可取消；独立进程不是操作系统权限沙箱，不宣称能完全隔离解析器漏洞。
- 限制：不提供视觉语义搜索、云端 AI、团队同步、完整工程依赖导出；素材封面/商品内容不属于 GPL 开源范围；BOOTH 网站限制与验证仍由网站决定。

## 5. 安装 / 从源码运行

普通用户（Windows x64）：

1. 下载 VRC-Asset-Library-1.10.3-x64.exe（Releases）。
2. 首次启动选择 C 盘外资料库位置；备份位置另选。
3. 添加素材母目录并扫描；默认 E:\VRChat，本机应改选 <VRCHAT_ROOT> 相关目录。
4. 开启在线补全；有歧义条目去「匹配确认」；内置 BOOTH 浏览器确认商品。

源码（Node.js 24 + npm）：

npm ci
npx install-electron
npm run check
npm run build
npm test
npm run dev
npm run package

输出 build-stage/VRC-Asset-Library-1.10.3-x64.exe；使用者不需要 Node.js 或 Unity。源码构建需要 Electron 运行时下载。

## 6. 源码结构（已完整克隆）

本地完整仓库：<VRCHAT_ROOT>\tools\sources\vrchat-asset-library

- 194 个文件 / 2.5 MB
- Git HEAD：107748a0598de9a52cc9db0aaa850782afb6e4fe（2026-10-20 12:16:17 +0800）
- 远程：https://github.com/NecoAya/VRCAL-VRChat-Asset-Library.git
- 版本：package.json 1.10.3；LICENSE GPL-3.0-only

关键代码：

- src/main/scanner.ts：素材目录扫描、Unity 工程/归档处理
- src/main/matching.ts / matching-catalog.ts：BOOTH 商品编号/名称匹配
- src/main/online.ts：BOOTH 联网、封面、补全队列
- src/main/booth-workspace.ts：内置 BOOTH 浏览器/登录/下载
- src/main/grouping.ts / organization.ts：重复展示合并、分类
- src/main/store.ts：资料库数据、设置、导出
- src/main/archive-*.ts：ZIP/Unitypackage 只读解析
- src/main/project-materials.ts：Unity 工程素材线索
- src/renderer/：React 图库、匹配确认、设置、主题
- src/shared/：消息契约、i18n、对话框
- docs/USER_GUIDE.md：完整使用与开发说明（21,773 B）
- docs/ROADMAP.md：路线图

## 7. 与本工作区的关系

- 可作为 <VRCHAT_ROOT>\resources（约 24k 文件 / 193 GB）的只读索引/图库/BOOTH 商品匹配工具，弥补目前靠目录名和手动记录的不足。
- 它不能替代 02-06 目录规范，也不会自动搬动/去重原文件；两者是索引层和文件层的关系。
- 本机首次使用应把扫描母目录设为 <VRCHAT_ROOT>\resources 的上一级或分类母目录；资料库和备份必须放在素材目录之外（例如 <VRCHAT_ROOT>\tools\VRCALData），否则会提示迁移。
- 发布说明提到下载链接曾 404，后已修复；本次使用 v1.10.3。

## 8. 来源信息

- 标题、发布信息与视频来源见 `ACKNOWLEDGEMENTS.md`；仓库地址：https://github.com/NecoAya/VRCAL-VRChat-Asset-Library

## 9. 未验证 / 待办

- [ ] 未下载/运行 v1.10.3（只读源码与文档）。
- [ ] 未在本机 <VRCHAT_ROOT>\resources 上实际扫描、匹配 BOOTH。
- [ ] 未验证 GPL-3.0 源码构建（Node 24/Electron 44）在本机的可重复性。
- [ ] 若运行：先备份资料库位置，不要放在素材目录内；扫描/匹配结果需抽查，不能把自动分类当作已核实。
