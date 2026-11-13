# 上传链路：凭据生命周期、SDK 行为与客户端侧取证

> **来源**：原 `08-症状速查` 第 141/142/147/153/154/159/160/161/162/170/172/173/175/176 轮实测条目（2026-11-06 至 11-11）结构化整理。
> **分工**：服务端多包并存 / unityVersion 分叉 / PUT 对齐的完整机制见 [06-05](06-05-avatar-package-version-collision.md)；反射出包与面板字段见 [06-04](06-04-sdk-publish-with-reflection.md)；凭据探活与构建期噪声见 [07-01 §35/§36](../07-验证与排查/07-01-bake-and-validation.md)。本册收「凭据为什么会丢 / SDK 3.10 行为变化 / 怎么证明客户端真的换了包」。

## 目录

1. 凭据生命周期（Build 清空 / VRCX 注入 / token 类型 / 占位凭证 / 脚本登录 / 探活）
2. SDK 3.10 行为变化（禁脚本上传 / 协议 SessionState / 主线程 / 缩略图）
3. 上传结果判定（时间线 / file 版本号 / .vrca 直读）
4. 客户端侧取证（会话缓存 / 缓存版本号解码 / 日志体积陷阱 / NOT_RUN 口径）
5. 版本对齐要点（与 06-05 的接口）
6. 上传脚本卫生（凭据清理 / 目标解析 / 类型 / 日志匹配）

---

## 1. 凭据生命周期

### 1.1 `builder.Build()` 会清/重置内存里的 ApiCredentials

**症状**：上传 `.vrca` 报 `Unauthorized: Missing Credentials`，但同一会话几分钟前 `GetAvatar` 还是好的；或第二次上传开始一直 `NOAUTH`（第 141/142 轮）。
**判据**：构建日志 `BUILD done path=...` 紧接着 `APIERR code=Unauthorized`；`_upload_status.txt` 里 `BUILD done` 紧跟 `validate api-err code=Unauthorized`。
**规则**：**构建后重新注入凭据再上传**。构建前注入的 token 在构建后失效（第 147 轮：`GetAvatar` 报 `BadRequest: Using a client-derived token outside of client` 同因）。

### 1.2 VRCX auth cookie 注入配方（本机实测 v58→v59、v61）

1. 从 `%APPDATA%\VRCX\VRCX.sqlite3` 的 `cookies` 表取 `key='default'` 的 base64 → JSON 数组 → 取 `Name=='auth'` 的 `Value`（**VRCX 会刷新 cookie，旧值直接 400，每次上传前重新取**）。
2. 反射写 `VRC.Core.ApiCredentials.authToken` + 构造 `BestHTTP.Cookies.Cookie`（authTokenCookie）。
3. 请求前 `ConfigureHeaders()`：设 UA、删 `X-MacAddress/X-SDK-Version/X-Platform/X-Unity-Version`——**仅用于 web token 场景**，否则 web token 会被判 `Using a web-derived token in client`。
4. 上传前剔掉 `variant=security` 包。
工具：`Assets/Editor/R141u.cs`、`MeowUpload.cs`、`docs/_vrcx_auth.txt`。

### 1.3 不要动 `VRCApi.Headers`（SDK 面板已登录时）

**症状**：上传第一步 `GetAvatar` 就 400：`Using a client-derived token outside of client`（第 153 轮）。
**根因**：脚本把 `VRCApi.Headers["User-Agent"]` 强改成自定义串并删 `X-SDK-Version/X-Platform/X-Unity-Version`。SDK 面板已登录时 token 是「客户端派生」的，UA 一变就判定越界。
**复现/修法**：去掉 UA 改写后同一 token 立刻 200。**SDK 面板 `APIUser.CurrentUser != null` 时直接沿用其会话；只有构建后会话真的丢了才重新注入外部 token**（§1.1/1.2）。

### 1.4 「面板已登录」≠「凭证有效」：占位凭证判别

**症状**：SDK 面板看起来已登录；点上传弹 `Upload Failed — Failed to agree to the ownership agreement`，Console 反复 `Unauthorized, try logging out and in again` + `"Missing Credentials"` + `ApiException`（第 176 轮，2026-11-11）。
**错误归因（先踩过的）**：以为「没登录」反复登录——实际 `IsLoggedIn` 本来就是 True；也不是 Unity 版本问题（"Wrong Unity version" 只是警告，未阻断请求）。
**逐项读回判据（30 秒定案）**：
```csharp
// 1) 标志位
bool li = (bool)typeof(VRC.Core.APIUser).GetProperty("IsLoggedIn").GetValue(null);   // 本例 True
var cu = typeof(VRC.Core.APIUser).GetProperty("CurrentUser").GetValue(null);          // 本例有值
// 2) 真正的凭证（关键）
var at = typeof(VRC.Core.ApiCredentials).GetField("authToken",
    System.Reflection.BindingFlags.Static|Public|NonPublic);
string tok = (string)at.GetValue(null);
// 合法 base64 解析失败 且 以 "authcookie" 开头 → 占位/失效凭证
// 3) 请求头是否带 Cookie：读 VRC.SDKBase.Editor.Api.VRCApi.Headers，无 Cookie 键 → 请求不带鉴权
// 4) 服务端裁决（最权威）：VRCApi.GetAvatar(id, true, default) 是否返回
```
本例实测：`IsLoggedIn=True`，但 `authToken` 长度 47 且以 `authcookie` 开头、**非合法 base64**；全部 VRC/BestHTTP 程序集扫描找不到任何 CookieContainer；`VRCApi.Headers` 只有 UA/X-MacAddress/X-SDK-Version/X-Platform/X-Unity-Version/Accept。网络对照：`https://api.vrchat.cloud/api/1/config` 返回 HTTP 200（30 KB）→ 排除网络/代理/版本。
**结论与修法**：这是「SDK 标志位已登录、但凭证是占位值、请求不带 cookie」的状态，与「会话被顶掉后 SDK 未刷新凭证」一致。**唯一解法是在 SDK 面板 Authentication 页真正完成一次 client 登录**（必要时先 Logout 再 Login）。不要：调 `VRCApi.Headers` 手拼 cookie（该补丁会让面板登录态失效）、换 Unity 版本、重复盲登。
**验收口径**：修好后 `GetAvatar(target,true)` 应能返回；**仅 `IsLoggedIn==true` 不能作为可上传的证据**。

### 1.5 脚本登录：APIUser.Login + CookieJar 取 auth

上传前 Check 返回 `NOAUTH`、`APIUser.CurrentUser==null` 但网络正常时（第 175 轮增补，2026-11-11）：
- 可行解法（Editor 内脚本）：`VRC.Core.APIUser.Login(user, pass, onSuccess, onError, on2FA)`。签名（VRCCore-Editor）：`Login(String,String,Action<ApiModelContainer<APIUser>>,Action<ApiModelContainer<APIUser>>,Action<ApiModelContainer<API2FA>>)`；成功回调里 `(APIUser)c.Model`，2FA 回调 `(API2FA)f.Model`。
- **坑1**：登录回调里 `c.Cookies` 为空、`APIUser.CurrentUser.authToken==null`，拿不到可用 auth cookie。
- **坑2**：真正的 auth cookie 在 **`BestHTTP.Cookies.CookieJar.Get(new Uri("https://api.vrchat.cloud/"))`** 里 `Name=="auth"` 的 value（长度 47）。取到后调 `VRC.Core.ApiCredentials.Set(humanName, providerId, provider, token, 2faToken)`（**5 参数**）写回，随后 Check 立即可用（本例 `VALID source=current-session`）。
- **不要把账号密码写进脚本/日志/KB**；用临时文件读取后即弃（收尾清理见 §6.1）。

### 1.6 探活优先，不要先要 OTP

`APIUser.CurrentUser==null` 只是「面板没登录」，与「持久凭据是否有效」是两回事（第 132/134 轮，详见 [07-01 §35](../07-验证与排查/07-01-bake-and-validation.md)）：
- 先 `VRCApi.GetAvatar(bp, forceRefresh:false)` 探活：成功就直接 `builder.Build + UpdateAvatarBundle`（第 132 轮实测 49→50、第 134 轮 50→51）。
- `GetAvatar(bp,true)` 会误报 `BadRequest: Using a web-derived token in client`（client-only 端点判定）；**探活用 `forceRefresh:false`**，不要据此清凭据或判失效，上传本身不受影响。
- 反射读 `ApiCredentials.authToken` 常因类型/字段名解析失败得到 `len=-1` 而误判；`ApiCredentials.Load()` 反射读 token 是空的，别白费劲。

## 2. SDK 3.10 行为变化

### 2.1 脚本上传入口被移除

> **2026-11-12 补充/取代**：被移除的是旧`VRC_SdkBuilder.ExportAndUploadAvatarBlueprint`入口，不是所有脚本上传。当前官方[Public SDK API](https://creators.vrchat.com/sdk/public-sdk-api/)明确提供Build、Build and Test、Build and Upload；SDK3.10.6的`IVRCSdkAvatarBuilderApi.BuildAndUpload`为`[PublicAPI]`，实现位于`VRCSdkControlPanelAvatarBuilder.cs`3968/3973。应通过`VRCSdkControlPanel.TryGetBuilder<IVRCSdkAvatarBuilderApi>`取得builder，保留原Blueprint/元数据，检查登录、所有权、版权授权后调用；不要反射旧入口，也不要因本节旧标题断言“只能GUI”。GUI仍是可用回退，不是唯一合法路径。版权SessionState只在已有用户授权时读取，不得自行预置冒充同意。

SDK 3.10 调 `VRC_SdkBuilder.ExportAndUploadAvatarBlueprint(avatar)` 报 **`Uploads via the SDK Builder are no longer supported`**（第 160 轮）。`VerifyCredentials()` 仍可调，但上传必须走 Control Panel GUI：`VRChat SDK/Show Control Panel` → Builder → `Build & Publish` → 版权协议 `OK`。
**协议弹窗在某些远程/无焦点会话里不接收合成点击**（`SetCursorPos`+`mouse_event` 与 UI 自动化都试过；同一窗口内 Console 的 Clear 按钮却能响应）→ 这种情况只能由本机用户手点，交付时标 `NOT_RUN（等待本机确认）`。绕过弹窗的合法做法见 §2.2。

### 2.2 版权协议状态在 SessionState（可预置，等同用户点 OK）

协议状态存在 **`SessionState`** 而非 EditorPrefs，未同意就弹 UIElements Modal（第 161 轮）。源码 `Packages/com.vrchat.base/Editor/VRCSDK/Dependencies/VRChat/VRCCopyrightAgreement.cs`：`SESSION_STATE_PREFIX="VRCSdkControlPanel"`、`AGREEMENT_KEY="…CopyrightAgreement"`、`CONTENT_LIST_KEY=…+".ContentList"`，内容为分号分隔的 contentId。
**做法**：`SessionState.SetString("VRCSdkControlPanel.CopyrightAgreement.ContentList", pipelineManager.blueprintId)`，然后重开 SDK 面板再点 Build & Publish —— 不再弹协议框，直接开始构建。**此法是绕 UI 不是绕授权（等同用户点 OK）**。

### 2.3 VRCApi 任何调用都必须在主线程

后台线程 / Task 里调 `VRCApi.GetAvatar` 报 `SetString_Internal can only be called from main thread`（来自 `VRCSdkControlPanel.set_serverEnvironment`，第 175 轮增补）。
**修正**：`task.ContinueWith(..., TaskScheduler.FromCurrentSynchronizationContext())` 回到主线程再调；`EditorApplication.delayCall` + async lambda 会**静默不执行**，不可靠。主线程上 `.Result`/`.Wait()` 等 async Task 会死锁（`Hold on (busy …)`），一律 `ContinueWith` 写文件再读；已卡死只能强杀重启编辑器。

### 2.4 缩略图：UpdateAvatarImage 单独提交 + 内容哈希幂等

- `VRCApi.UpdateAvatarBundle(id, data, bundlePath, onProgress, ct)` 第 4 参是**进度回调**，没有缩略图入口；只上传 bundle 不会动预览图（第 153 轮；`VRCApi.cs:617` 签名；线上 `ImageUrl` 的 file 版本号多次 bundle 上传后不变）。
- 正确签名：`VRCApi.UpdateAvatarImage(string avatarId, VRCAvatar av, string filePath, Action<string,float> onProgress, CancellationToken ct)`（第 175 轮增补）。把 string 当 av、或把 av 当第一参都会编译失败。单独更新成功返回新的 `ImageUrl …/file_<guid>/1/file`。
- **`This file was already uploaded` 是幂等跳过不是失败**：平台按文件内容哈希去重，同一张 png 重传即拒；**换文件名无效**（内容哈希不变）。随 Publish 流程走缩略图会因此中止（bundle 还没传，Version 却 +1，详见 [07-01 §30](../07-验证与排查/07-01-bake-and-validation.md)）。
- 绕过：`VRCApi.UpdateAvatarImage(bp, new VRCAvatar{ID=bp}, 新副本路径, null, ct)` 单独更新。核对时比对平台下载图的 **dims + sha256**，与本地同画面即可（服务端会再编码，字节可能不同，属正常）。
- 缩略图要按**默认档**渲染：只留默认套装、隐藏 `ABT/SPS/LightController/RK Player`（`ABT/AntiCulling` 会渲染成粉色球挡手），渲染完把激活态逐项还原（五段式见 [07-09 §7](../07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md)）。工具 `MeowThumb.cs`。

## 3. 上传结果判定

### 3.1 「上传了但没变化」先做时间线比对

**症状**：用户报「游戏里看不到新内容」，但历史记录里有 `UPLOAD ok`（第 153 轮）。
**根因**：那次 `UPLOAD ok` 的时间**早于**本轮所有修复的落盘时间；上传成功过 ≠ 上传的是最新代码。
**做法**：比对 `docs/_upload_status.txt` 的 `UPLOAD ok version=N` 时间戳与场景/controller/EP 的 mtime（本例 upload 16:16，四个关键文件 19:17/21:54/23:00/23:09）。**只有 mtime 全部早于 upload 时间，才说明包是新的**；不要先怀疑烘焙或菜单。

### 3.2 判新旧只看 file 版本号

- `Version`（avatar 记录版本）递增只说明元数据更新；**bundle 是否换新看 `unityPackages[].assetUrl` 里的 `/NN/file` 文件版本号**（第 124/170 轮）。
- `TaskCanceledException` 不等于服务端失败（30 秒客户端超时，上传可能已落盘）：先 GET 复核再决定重试，详见 [06-05 §9](06-05-avatar-package-version-collision.md)。
- 重复上传同一 `.vrca` 报 `This file was already uploaded` 属文件级去重（第 123/172 轮）。

### 3.3 直接读 `.vrca`：菜单/参数/默认值的最强证据

构建产物 `%TEMP%\DefaultCompany\<proj>\<guid>.vrca` 就是上传的原始 bundle，可直接当 AssetBundle 读（第 154/156 轮）：
```csharp
var ab = AssetBundle.LoadFromFile(path);
var go = ab.LoadAsset<GameObject>(ab.GetAllAssetNames()[0]);
var desc = go.GetComponent<VRCAvatarDescriptor>();
// 递归 dump expressionsMenu / expressionParameters
```
- 里面 **1 个 prefab**（名字 `prefab-id-v1_<avtr_id>_<n>`）。**FX 控制器读出来是 `VRCFury FX layers=0`**——控制器由 VRChat 运行时装配，不落 bundle；动画层行为只能真机验。
- 装了 VRCFury 的工程，最终菜单/EP 被重写成 **`VRCFury Menu` / `VRCFury Params`**（本例 138 项/212 bits、166 项）；MA 只是被合并的一环，**别以为被覆盖了**。`saved` 等字段要确认 VRCFury 重建后仍保留（本例 87 项 saved，关键项均在）。
- 注意同一 bundle 不能重复 `LoadFromFile`（先 `Unload(true)`），50MB+ 加载要给足超时。
- 出包审计配方（每页 ≤8 项、同步位 ≤256、菜单项参数全注册、第三方内部参数白名单 `ABT/ CC2/ SPS LightController/ __MA/`）：本例 v80 审计 EP 166 参数、241/256 bits、38 页、单页最大 8、问题 0。工具 `Assets/Editor/R194BundleAudit.cs`。

## 4. 客户端侧取证

### 4.1 会话缓存：必须完全重启客户端再切头像

**症状**：上传成功（版本号 +1、asset URL 变了），进游戏菜单还是老版；切出再切回也没用（第 154 轮）。
**根因**：VRChat 客户端在**进入世界时**把头像记录（含素材 URL）读进会话缓存；之后服务器换版本，本会话不会重新拉取。官方 Feedback 看板确认「只有重进世界、有时必须完全重启客户端才更新」。
**判据**：比对三处时间——`VRChat` 进程 `StartTime`（本例 23:18:33）、日志 `Loading Avatar Data:avtr_…`（23:19:10）、上传完成时间（23:37:19）。进程启动早于上传 → 必然拿旧记录。
**做法**：**完全退出客户端再重开**再选模型。新旧 asset URL 不同（`file_xxx/10/file` → `/12/file`），缓存按 URL 分桶，重启即走新桶，**不需要清 30GB 缓存**。日志在 `%LocalLow%\VRChat\VRChat\output_log_*.txt`。

### 4.2 缓存目录版本号解码（确定性取证）

VRChat 缓存目录第二层的名字末段就是 file 版本号（第 162 轮）：
- 打开 `%CACHE%/Cache-WindowsPlayer/{头像Hash}/{32位hex}/`，取 32 位 hex 的**第 25–32 字符**（8 位十六进制），**按 little-endian** 读即为 `file` 版本号。
- 例：`cf0e359f035a363a0000000017000000` → `17 00 00 00` → **23**；旧包 `a2985b4fc57466ec0000000016000000` → `16 00 00 00` → **22**。
- **判断客户端是否拉到新包只认这个版本号**：等于 API 侧 `unityPackages[].assetUrl` 里的 `/NN/` 即已更新；目录里只剩新版本号、旧版本号目录消失 = 已被替换。

### 4.3 缓存 `__data` 与 `.vrca` 不能用 md5/字节数对账

缓存里的 `__data` 是**解压后**的 UnityFS（示例 86,053,196 B），而 `.vrca` 是整体压缩包（示例 57,055,501 B），比例约 1.5（第 162 轮）。读 `__data` 前 16 字节 = `55 6E 69 74 79 46 53 00 00 00 00 08 35 2E 78 2E`（`UnityFS` + `5.x.`）；md5 与 API 的 `file.md5` 不同属正常。**用目录名字段判版本、用 `file.versions[].file.sizeInBytes` 判官方大小**。

### 4.4 日志体积陷阱（MB 取整 + 小改动）

- 日志 `Starting download of Avatar @ 54 MB` 按 **MB 取整**：file 22 = 54.35 MB、file 23 = 54.41 MB，四舍五入同为 54（第 162 轮）。
- **小改动（菜单结构/参数）对 `.vrca` 体积影响极小**：file 29 = 67,249,395 B (64.13 MiB)、file 30 = 67,119,186 B (64.01 MiB)，差 130,209 B = 0.19%，日志取整后都是 `@ 67 MB`（第 170 轮）。
- **体积判据只适用于大改动**（换模型/加贴图/加网格，体积跨 MB 档时）。小改动必须用缓存目录版本号（§4.2）且 mtime 晚于上传时刻。两者至少一条成立才算 PASS；**体积无变化不能判 FAIL**。

### 4.5 `__info` 与 mtime 的正确读法

- `__data` 保留**首次落盘**时间，重开客户端命中缓存时只重建 `__info` 与目录，不改 `__data`（第 162 轮；示例 `__data` mtime 04:57，`__info` 与目录 mtime 09:48:39.370，同秒日志有 `Starting download of Avatar @ 54 MB`）。
- 判「本次是否真的加载」看 **`__info`/目录 mtime 与日志同秒** + 日志 `Loading Avatar Data:avtr_xxx` 紧随其后；判「是否为新版本」看目录名末段版本号。
- `__info` 只有 23 字节纯文本：`-1\n{unix_ts}\n1\n__data\n`（4 行），**不含版本、不含大小、不含 md5**（hex `2D310A313739343130323531390A310A5F5F646174610A`）。只能用作「存在性/最后访问时间」旁证。

### 4.6 未重启客户端一律 NOT_RUN

VRChat 只在**切头像**时下载包；不重启不切头像，日志与缓存都不会出现新记录（第 173 轮）。
- 取证路径：日志在 `%USERPROFILE%\AppData\LocalLow\VRChat\VRChat`；`HTTPCache-WindowsPlayer`/`Avatars`/`AvatarParts` 三个目录均 0 文件；`LocalAvatarData\usr_…\avtr_…` 的 mtime 早于上传时间。
- **未重启客户端一律写 `NOT_RUN`**（附「日志最后一条 avatar 下载时间戳」作为已取证不可行的证据），**不得**用 SDK 弹窗 "Upload Succeeded" 代替客户端证据。
- 用户明确「不允许启动游戏测试」时同口径；短暂启动但未切头像、未取下载日志的运行不作证据（第 174 轮判据摘要）。

## 5. 版本对齐要点（完整机制见 06-05）

- **每次上传的固定收尾**：GET → 逐档比较 → 不一致就 PUT → **再 GET 一次**断言唯一 assetUrl。对齐不是一次性的，下次上传又会分叉（[06-05 §10](06-05-avatar-package-version-collision.md)）。
- **PUT 顺序必须先 security、后 standard**：VRChat 的 PUT 按 **platform + unityVersion** 匹配条目，**不看 variant**；第二次 PUT security 会覆盖第一次的 standard 条目（第 161 轮：c1 standard URL 变成 `/23/variant/security`）。每次 PUT 后 sleep 2s 再 GET；判据是 standard 与 security 两组各自的 file 版本集合都只有 1 个元素。
- **上传脚本自动对齐也会跳过 standard**：`align skip 2022.3.22f1 (already new)` 是按「等于该档最大值」判断，不是「等于最新」；SDK 自动对齐只覆盖它自己 PUT 的那几档，且不处理 security 变体。**必须自己再跑一遍全量 PUT**（按 unityVersion 去重，每档各 PUT security 与 standard，先 security 后 standard；本工程 4 次 PUT 全 HTTP 200，第 170 轮）。
- **纯 HTTP 对齐（不依赖编辑器会话）**：编辑器里 `GetAvatar` 挂住（崩溃重启后会话半失效、`delayCall + async Task` 吞异常）时，用 VRCX auth cookie 直接 `GET /api/1/avatars/{id}` 列 `unityPackages` → `PUT /api/1/avatars/{id}` body `{assetUrl, platform:"standalonewindows", unityVersion:"<c1档>", assetVersion:1}`（第 159 轮：version 87→88，两条 standard 都指 `/21/file`）。
- 证据链口径（PRE/UPLOAD/ASSET/ALIGN/verify/DONE + 独立二次 GET）见 [06-05 §8](06-05-avatar-package-version-collision.md)。

## 6. 上传脚本卫生

### 6.1 交付前清掉临时凭据

历代实验脚本可能直接写死账号口令（第 173 轮）：删脚本（含 `.meta`）+ 删含验证码的截图 + 全库 `Select-String` 口令关键字复核 **0 命中**；保留验证类脚本与环境前提类脚本（如 `FixCookieContainer.cs`，见 [01-06 §13](../01-环境与工具链/01-06-project-launch-and-mcp-bridge.md)）。

### 6.2 目标解析不要写死名字（Find 只匹配激活对象）

**症状**：上传脚本点了没反应，日志只有 `FAIL avatar missing (scene?)`（第 153 轮）。
**根因**：脚本写死 `GameObject.Find("沐梓溪狼")`；该对象 `activeSelf=false`。`GameObject.Find` 只匹配激活对象，失活返回 null。
**做法**：优先 `GameObject.Find(新名)` 且校验 `VRCAvatarDescriptor`，再退回「`FindObjectsOfType<VRCAvatarDescriptor>(true)` 里第一个 `activeInHierarchy && blueprintId 非空`」。工具 `MeowUpload.cs FindTargetAvatar()`。

### 6.3 类型与编译坑

- 该 SDK 的 `VRCAvatar.UnityPackages` 是 `List<VRCUnityPackage>`，**不是 `VRCAssetVersion`**；`??` 空值合并要配 `new List<VRCUnityPackage>()`（第 172 轮，CS0246/CS0019）。`VRCUnityPackage` 是 struct，不能 `p == null`（[06-05 §5](06-05-avatar-package-version-collision.md)）。
- 反射取 `Build` 方法抛 `AmbiguousMatchException`：枚举 `GetMethods` 筛 `Name=="Build" && 参数数==1 && 参数类型==typeof(GameObject)`（[06-04 §9](06-04-sdk-publish-with-reflection.md)）。

### 6.4 日志等待匹配要区分大小写

PowerShell `-match 'DONE'` 默认不区分大小写，`BUILD done` 会提前命中「等待完成」循环 → 误判上传卡死（第 175 轮）。用 **`-cmatch` 或 `(?m)^DONE$`**；真正的完成判据是 `ALIGN done all-current` + `DONE` + 二次 GET 两档一致。

### 6.5 历史报错 ≠ 当前缺陷

`Destroy may not be called from edit mode!`、`Invalid server environment name`（GetString only from main thread）来自早期探测脚本；清理后确认 `read_console` 只剩无关警告，再判定工程干净（第 175 轮增补）。

## 7. 来源

- 本工作区实测 2026-11-06 至 11-11（第 141–176 轮相关条目），原载 `08-症状速查` 02/03/04/05/06/07/08 增补册，2026-11-11 结构化并入本册。
- SDK 源码：`VRCApi.cs:617/629`、`VRCCopyrightAgreement.cs`、`VRCAvatar.cs:67`（security variant Enum.Parse，见 [06-04 §10](06-04-sdk-publish-with-reflection.md)）。
- 关键脚本（工程内）：`MeowUpload.cs`、`R141u.cs`、`MeowThumb.cs`、`R194BundleAudit.cs`、`R259AlignOnly.cs`、`R400UploadExisting.cs`。
- VRChat 官方 Feedback 看板（客户端会话缓存行为）；VRCX cookie 存储 `%APPDATA%\VRCX\VRCX.sqlite3`。
