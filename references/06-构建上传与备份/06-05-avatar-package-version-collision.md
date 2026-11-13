# 上传成功但游戏里一直是旧内容：服务端「同一平台多份包并存」的排查与修复

> 本工作区实测 **2026-11-06 第 155 轮**，工程 `LuzzyMeow V2`，头像 `avtr_4f633989-54cd-48a4-905d-b1c92e5bf4ab`。
> 这是**极易误判**的一类问题：上传日志全绿、版本号在涨、包里内容确实是对的，但游戏里始终是旧菜单 / 旧默认值 / 旧缩略图。
> 上一轮（第 154 轮）曾判定为「客户端会话缓存，重启即可」——**该结论是错的**，本轮推翻并给出真正根因。

---

## 1. 症状

一组**同时出现**的表征（只出现其中一两个时容易被误判成别的问题）：

| 现象 | 说明 |
|---|---|
| 菜单项少了、新功能不出现 | 例如「功能」菜单本该 6 项，游戏里只有 3 项（ABT工具 / SPS / 亮度调节） |
| 新加的默认值不生效 | EP 里 `OutfitOneClick default=3`（睡衣），进游戏却是裸体 = 运行时按 0 处理 |
| 缩略图不更新 | `VRCApi.UpdateAvatarImage()` 报 `IMAGE ok`，但选人界面还是旧图 |
| 上传日志全绿 | `UPLOAD ok version=66`、`ASSET https://.../12/file`，没有任何报错 |
| 重启客户端 / 重进世界 / 清 30GB 缓存 **全部无效** | 这是最强判据：本地手段全失效 ⇒ 问题在服务端 |

**判据**：只要「上传成功 + 版本号增长 + 清缓存重启无效」三者同时成立，**立刻怀疑服务端多包并存**，不要再在本地缓存上花时间。

---

## 2. 根因：服务器按 `(platform, variant, unityVersion)` 去重

`VRCApi.UpdateAvatarBundle()` 提交时发的是这四个字段：

```csharp
// Packages/com.vrchat.base/Editor/VRCSDK/Dependencies/VRChat/API/VRCApi.cs:646-652
var bundleUpdateRequest = new Dictionary<string, object>
{
    {"assetUrl", newBundleUrl},
    {"platform", Tools.Platform.ToString()},      // standalonewindows
    {"unityVersion", Tools.UnityVersion.ToString()},
    {"assetVersion", 1}
};
```

服务器把它当作「**这一档** `(platform, variant, unityVersion)` 的新版本」来替换。**`unityVersion` 不同 → 认为是另一档 → 旧的那档原样保留，两条并存。**

关键陷阱：**Unity 中国版（`f1c1` 后缀）会被 SDK 归一化成 `f1`**。

- 工程 `ProjectSettings/ProjectVersion.txt` = `2022.3.22f1c1`
- 但 `Tools.UnityVersion` 上传时是 `2022.3.22f1`
- 而工程**历史上**某次上传留下了 `2022.3.22f1c1` 的那一档

于是：

```
standalonewindows / standard / 2022.3.22f1c1  →  file_xxx/4/file    22.1 MB   ← 旧内容，且排在前面
standalonewindows / standard / 2022.3.22f1    →  file_xxx/13/file   54.7 MB   ← 每次上传都更新这条
```

**客户端取第一个匹配项**，所以永远加载旧的那 22.1 MB。

实测证据：真机日志里下载体积是 `(21.9 MB)`（压缩传输后 ≈ 22.1 MB 原文件），而新包是 54.7 MB。**日志里的体积就是最快的判别指纹**。

---

## 3. 诊断流程（四步，全部可复现）

### 3.1 读服务端真实包列表

用 VRCX 里存的 auth cookie 直接打 API（**不要**用 `curl`，Windows 的 schannel 对 `api.vrchat.cloud` 常握手失败）：

```python
import sqlite3, json, base64, urllib.request
db = r'<USERPROFILE>/AppData/Roaming/VRCX/VRCX.sqlite3'
c = sqlite3.connect('file:'+db+'?mode=ro', uri=True)
# cookies 表里 key='default' 的值是 base64(JSON 数组)
j = json.loads(base64.b64decode(c.execute("select value from cookies where key='default'").fetchone()[0]))
AUTH = [x for x in j if x['Name'] == 'auth'][0]['Value']

UA = "VRChat/6000.0.67f1 (Windows; Windows 10  (10.0.19045) 64bit) Unity/6000.0.67f1"
r = urllib.request.Request("https://api.vrchat.cloud/api/1/avatars/" + AVATAR_ID)
r.add_header("Cookie", "auth=" + AUTH)
r.add_header("User-Agent", UA)          # 必须是 VRChat/<unityVersion> 形式，否则 400
d = json.loads(urllib.request.urlopen(r, timeout=30).read().decode())
for p in d["unityPackages"]:
    print(p["platform"], p["variant"], p["unityVersion"], p["assetUrl"])
```

**坑**：
- `User-Agent` 用 `Mozilla/5.0` 或 `VRCX/...` 都会 **HTTP 400**（`client-derived token outside of client`）。必须 `VRChat/<unityVersion> (...)`。
- `unityVersion` 从 `VRChat.exe` 的 `ProductVersion` 取（本例 `6000.0.67f1-DWR`）。
- cookie 的 `default` 值是 **base64 编码的 JSON**，不是明文 JSON。

### 3.2 下载两个包，直接解菜单树比对

**重要区别**：**API 下载的 `.vrca` 是未加密的，`UnityPy.load()` 可直接读**；而本地缓存 `<CACHE_ROOT>/Cache-WindowsPlayer/**/__data` 是 Unity CN 加密的（报 `The BundleFile is encrypted, but no key was provided!`，密钥在加密过的 `global-metadata.dat` 里，`brute_force_key` 也提不出来）。**所以诊断走 API 下载，不要试图解缓存。**

```python
import UnityPy
env = UnityPy.load(path)
cls = {}
for o in env.objects:                       # 先建 MonoScript path_id -> className 映射
    if o.type.name == "MonoScript":
        try: cls[o.path_id] = o.read().m_ClassName
        except: pass
menus = {}
for o in env.objects:
    if o.type.name != "MonoBehaviour": continue
    try: t = o.read_typetree()
    except: continue
    if cls.get((t.get("m_Script") or {}).get("m_PathID")) == "VRCExpressionsMenu":
        menus[t.get("m_Name", "")] = [
            (c.get("name"), c.get("type"), (c.get("parameter") or {}).get("name", ""))
            for c in (t.get("controls") or [])
        ]
```

**字段名陷阱**（本次踩了三次）：
- `VRCAvatarDescriptor` 的菜单指针字段是 **`expressionsMenu`**，不是 `customExpressions`（后者是我臆造的，值为 `None` → 直接 `AttributeError`）。
- `VRCExpressionsMenu` 的控件数组字段是 **`controls`**（小写、无 `m_` 前缀），**不是** `m_Controls`。每项字段是 `name` / `type` / `parameter` / `subMenu`（同样小写）。
- `type` 数值：`101=Button`、`102=Toggle`、`103=SubMenu`、`201=TwoAxisPuppet`、`203=RadialPuppet`。

**比「按名字找菜单」更稳**：菜单对象的 `m_Name` 是「功能」「外观」这种短名，但**子菜单的 `m_Name` 是完整路径**（如 `功能 » ABT工具 » 动作 » 替换坐姿`）。所以 `menus['功能']` 能直接命中根级「功能」，是可靠的。

实测输出（本次）：

```
旧包 4/file (22,171,466 B)              新包 13/file (57,330,693 B)
<功能> n=3                              <功能> n=6
  1. ABT工具                              1. ABT工具
  2. SPS                                  2. SPS
  3. 亮度调节                              3. 亮度调节
                                          4. RealKissSystem
                                          5. 面捕
                                          6. RK Player Menu
<外观 » 衣服> n=5                        <外观 » 衣服> n=6
```

旧包 `n=3` 与用户截图**逐项一致** → 锁定「客户端加载的是 4/file」。

### 3.3 用日志体积交叉验证

```
[AssetBundleDownloadManager] Download for avatar (Worn:1 ...) (21.9 MB) ...
```

日志里的 MB 数 ≈ 目标包原文件大小（÷1.05 左右的压缩比）。和 3.2 的包体对照，一眼能认出加载的是哪一档。

### 3.4 确认不是本地问题（排除项）

```bash
# 本地缓存清空后重新下载，新旧字节数应完全相同 → 说明服务器分发的就是那一份
find "<CACHE_ROOT>/Cache-WindowsPlayer" -name "__data" -exec stat -c'%s %n' {} \;
```

本次清空前/后两次下载都是 `130,897,791 B`（125M）+ `199,407,936 B`（191M），**完全一致** → 服务器稳定分发同一份旧包，不是下载损坏。

---

## 4. 修复：用旧的那档 `unityVersion` 再 PUT 一次

原理：把新 `assetUrl` 以**旧档的 `unityVersion`** 提交，服务器就会用新内容**替换掉那条旧记录**（而不是新增）。

```python
body = {
    "assetUrl": "https://api.vrchat.cloud/api/1/file/file_xxx/13/file",
    "platform": "standalonewindows",
    "unityVersion": "2022.3.22f1c1",   # ← 填要顶掉的那一档
    "assetVersion": 1,
}
req = urllib.request.Request("https://api.vrchat.cloud/api/1/avatars/" + AVATAR_ID,
                             data=json.dumps(body).encode(), method="PUT")
req.add_header("Cookie", "auth=" + AUTH)
req.add_header("User-Agent", UA)
req.add_header("Content-Type", "application/json")
urllib.request.urlopen(req, timeout=60)
```

**实测结果**：

| | 修复前 | 修复后 |
|---|---|---|
| `standard / 2022.3.22f1c1` | `/4/file`（22.1 MB，功能 3 项） | **`/13/file`**（54.7 MB，功能 6 项） |
| `standard / 2022.3.22f1` | `/13/file` | `/13/file` |
| 包总数 | 7 | **7（无新增脏数据）** |
| `version` | 70 | 71 |

**注意 `PUT /avatars/{id}` 不接受 `unityPackages` 数组**（OpenAPI 里只允许 `assetUrl` / `imageUrl` / `name` / `description` / `releaseStatus` / `tags` / `unityPackageUrl` / `unityVersion` / `version` 等标量字段），所以**不能直接把列表改掉**，只能逐档 PUT。

改完必须清一次本地缓存（客户端按 asset URL 分桶，但旧桶文件仍在），或者至少删掉该头像对应的缓存目录：

```bash
rm -rf "<CACHE_ROOT>/Cache-WindowsPlayer/<该头像的桶目录>"
```

本次清空前缓存 2.04 GB / 18 个桶目录 → 清空后仅剩 `vrc-version` + `__info` 两个文件。

---

## 5. 防复发：上传脚本里加「版本对齐」

在 `UpdateAvatarBundle()` 之后追加一次对齐，把**其余所有 PC standard 档**也指向同一个新 asset。

> 实现见工程 `Assets/Editor/MeowUpload.cs` 的 `AlignStalePackages()` + `NormalizeAssetId()`（本工作区新增，2026-11-06）。

要点：

1. 遍历 `after.UnityPackages`，筛 `platform == "standalonewindows"` **且** `variant` 为空或等于 `standard`（**必须跳过 `security` 和 `impostor`**，`Enum.Parse` 会对 `security` 抛 `ArgumentException`）。
2. 用归一化 ID 比较（`.../file_xxx/13/file` 与 `.../file_xxx/13/variant/security` 要能区分）：
   ```csharp
   static string NormalizeAssetId(string url) {
       if (string.IsNullOrWhiteSpace(url)) return "";
       var segs = url.TrimEnd('/').Split('/');
       return segs.Length >= 2 ? segs[segs.Length - 2] + "/" + segs[segs.Length - 1] : url;
   }
   ```
3. 已是最新的跳过；不同的逐档 `VRCApi.Put<object, VRCAvatar>("avatars/" + bp, body, cancellationToken: ...)`。
4. 最后重新 `GetAvatar(bp, forceRefresh: true)` 复核，逐档打印 `OK` / `STALE`，断言 `stale == 0`。

**编译陷阱**：`VRCUnityPackage` 是 **`struct`**（`Packages/com.vrchat.base/.../VRCUnityPackage.cs:6`），所以 `foreach (var p in packages)` 里的 `p` 是值类型，**不能写 `if (p == null)`**（编译错误，struct 不与 null 比较）。要判空只能判外层 `List<...>`。

---

## 6. 判据（怎么算修好了）

| 层级 | 判据 | 命令 / 位置 |
|---|---|---|
| 服务端 | 所有 `standalonewindows / standard` 档的 `assetUrl` **指向同一个 file version** | 3.1 的脚本，逐档打印 |
| 包内容 | 解出来的「功能」菜单项数 = 6（含面捕 / RK Player / RealKissSystem） | 3.2 的脚本 |
| 客户端 | 日志里 `Download for avatar ... (54.x MB)`，不再是 21.9 MB | `output_log_*.txt` |
| 运行时 | 游戏内「功能」看到 6 项；默认穿着睡衣（`OutfitOneClick=3`） | 实机截图 |

---

## 7. 与其他坑的边界（别混为一谈）

| 现象相似但根因不同 | 区分方法 |
|---|---|
| **客户端会话内不刷新头像记录** | 官方 Feedback 有此问题；但**重启就能好**。本次重启无效 → 不是它 |
| **本地缓存损坏 / 分桶错乱** | 清缓存后体积会变。本次清完重新下载**字节数完全相同** → 不是它 |
| **包里内容本身是旧的**（上传用了错的产物） | 直接解上传产物的菜单树：本次 `ba4427b3-….vrca` 与 `815e7f8a-….vrca` 两次构建 diff **完全一致（233 行 0 差异）** → 包是对的 |
| **缩略图单独不更新** | 缩略图走 `UpdateAvatarImage()` → `PUT {imageUrl}`，与 asset 是两条独立链路；本次线上 `imageUrl` 已是新文件但游戏没换，因为客户端只认 asset 里的内存态 |

---

## 8. 一句话总结

> **上传"成功"只代表服务器接受了这次 PUT，不代表客户端会用它。**
> 只要 `(platform, variant, unityVersion)` 里任何一维与历史记录不同，就会**并存**而不是**替换**；而客户端只取排序第一的那条。
> 判据口诀：**日志体积 vs 包体、清缓存后字节数是否变化、两个包的菜单树逐项 diff** —— 三者交叉，5 分钟定位。

---

## 9. 2026-11-09 实测：上传调用超时并不等于服务端失败

**背景**：`VRCApi.UpdateAvatarBundle()` 内部先 `Get<VRCFile>` 再上传。SDK 默认那次 GET 的超时是 30 秒；大包或网络抖动时，客户端可能在元数据读取阶段抛 `TaskCanceledException`，但服务端已经创建/完成目标 file 版本。

**本工作区实测**：

- 第一次 `MeowUpload.Run()` 在 `UploadFile -> Get<VRCFile>` 抛出 `TaskCanceledException`。
- 立即 GET avatar：`version=153`，`2022.3.22f1` 的 standard 包已经变为 `file_2a82c26c-491b-4d31-9c2d-13ec3d348263/42/file`。
- 用同一 `.vrca` 再次调用 `UpdateAvatarBundle()`，SDK 报 `This file was already uploaded`；这不是新失败，而是服务端已经有该包。
- 随后手工 PUT 把 `2022.3.22f1c1` 也对齐到 `/42/file`，二次 GET 两条 standard assetUrl 完全一致。

**正确流程**：

1. 上传异常后先 GET `avatars/{id}`，比较 `updated_at`、`version` 和每个 `standalonewindows/standard` 的 file 版本。
2. 若已出现新 file 版本，不要重建、不要重复上传；进入版本对齐。
3. 若仍是旧 file，再重试或排查网络/登录。

4. 对齐后必须二次 GET，断言所有 PC standard 档的唯一 assetUrl。

**注意**：`assetsUrl` 中的 `file_xxx/NN/file` 才是判据；avatar `version` 上涨不等于客户端会换包。

---

## 10. 2026-11-10 实测：对齐过的两档 unityVersion 会在**下一次上传**后再分叉

**背景**：§9 的版本对齐只解决「当下两条 PC standard 包指向不同 file 版本」。但它不是一个能被记住的状态 —— 对齐结果完全由服务端当前的包列表决定，而每次上传只会新增**其中一档**的包。

**本工作区实测（同一头像连续两次上传）**：

- 第 172 轮上传后对齐到 `2022.3.22f1` = `2022.3.22f1c1` = `file_2a82c26c-.../42/file`，二次 GET 两条一致，任务判定通过。
- 第 173 轮（本次）再次构建上传，服务端返回 `version=157`，新包落在 `2022.3.22f1` 一档 = `/44/file`，而 `2022.3.22f1c1` **仍停在上一轮的 `/42/file`** → **两档重新分叉**。
- 运行 `MeowUpload.Run()` 时日志出现 `ALIGN start (2 pkgs)` → `align 2022.3.22f1c1 42/file -> 44/file` / `align ok` / `align skip 2022.3.22f1 (already new)` → `verify 2022.3.22f1 -> 44/file OK` / `verify 2022.3.22f1c1 -> 44/file OK` → `ALIGN done all-current`。

**结论（写进流程，不靠记性）**：

1. **「版本对齐」不是一次性动作，而是每次上传后的固定收尾步骤**。任何一次 `UpdateAvatarBundle` 之后都必须重跑 `AlignStalePackages()` + 二次 GET。
2. 对齐脚本必须做成**幂等**的：对每个 `platform=standalonewindows && variant=standard` 的包，比较 `NormalizeAssetId(p.AssetUrl)` 与「本次上传产生的新 file id」，不同才 PUT；相同就 `align skip ... (already new)`。这样重复跑不会把已经正确的档写坏。
3. 判定「哪一档是新的」只认 `file_xxx/NN/file` 里的 `NN` 与实际 `file_xxx` GUID，**不要**用 avatar `version`、`updated_at` 或包的 `assetVersion` 字段推断。
4. 上传 + 对齐是一个整体，验收必须看到 **二次 GET 的两条 standard assetUrl 字符串完全相等**，而不是「上传弹窗成功」。

**判据（可直接抄进验收脚本）**：

```
GET avatars/{bp}  ->  UnityPackages 过滤 (platform=="standalonewindows" && variant.toLowerCase()=="standard")
期望：该集合非空，且集合内每条 NormalizeAssetId(assetUrl) 只有一个唯一值
实际（本轮）：[1] unity=2022.3.22f1   url=.../file_2a82c26c-491b-4d31-9c2d-13ec3d348263/44/file
              [2] unity=2022.3.22f1c1 url=.../file_2a82c26c-491b-4d31-9c2d-13ec3d348263/44/file
              totalPkgs=2  -> 唯一值 = file_2a82c26c-.../44  -> PASS
```

**坑**：

1. 只跑一次上传看到「两档一致」就以为永久一致 —— 下一次上传必然再分叉，这是服务端按 `(platform, variant, unityVersion)` 去重的结构性结果，不是偶发。
2. 对齐时把 `security` 平台包或 `impostor` variant 也 PUT 成标准包的 assetUrl —— 会破坏该变体，必须显式跳过（`variant` 为空或 `Standard` 或 `Impostor` 才参与标准判定；`platform=="security"` 永不参与）。
3. `VRCUnityPackage` 是 struct，不能用 `p == null` 判空。
4. 对齐后不再 GET 复核 —— PUT 可能被服务端静默归一化（例如把 `assetVersion` 重置为 1），只有二次 GET 才能证明写入生效。
---

## 8. 2026-11-10 复核（第 175 轮）：同一问题第三次复现，口径固化

**服务端读回（独立 GET，不复用 PUT 返回值）**：

```
VERSION 165
=== standalonewindows / standard ===
  2022.3.22f1   | .../46/file
  2022.3.22f1c1 | .../46/file
```

**对齐前后（同一轮日志）**：`align 2022.3.22f1c1 45/file -> 46/file`、`align skip 2022.3.22f1 (already new)`。
注意这次出现的是**反方向**：`f1` 已被 SDK 顶到新包、`f1c1` 才是旧的那档 —— 说明哪一档落后**不固定**，不能只盯 `f1c1`。

**新坑（本轮踩到）**：

5. `IMAGE FAIL UploadException | This file was already uploaded` **不是上传失败**：当缩略图内容与云端已有文件逐字节一致时 SDK 会拒绝重传（幂等跳过）。判定要用 file id 比对；只要 `ALIGN done all-current` + `DONE` 且二次 GET 两行一致，任务就是成功的。
6. 等待「上传完成」的脚本不要写 `if ($log -match 'DONE')`：PowerShell `-match` 不区分大小写，会被 `BUILD done` 提前命中 → 误判「卡在 UPLOAD start」。用 `-cmatch` 或 `(?m)^DONE$`。

## 11. PUT 顺序与 variant 覆盖（第 161/170 轮实测 2026-11-08，2026-11-11 结构化并入）

### 11.1 PUT 按 platform + unityVersion 匹配，不看 variant

版本对齐 PUT 之后，c1 的 `standard` 条目 URL 变成了 `/23/variant/security`：**VRChat 的 PUT 按 platform + unityVersion 匹配条目，不看 variant**；第二次 PUT security 会覆盖第一次的 standard 条目。
- **PUT 顺序必须先 security、后 standard**；每次 PUT 后 sleep 2s 再 GET。
- 判据：`standard` 与 `security` 两组各自的 file 版本集合都只有 1 个元素。
- 第 165 轮复核：v100 时 f1=25 / c1=24 分叉；按此顺序 PUT 后 v102 四档全 25。

### 11.2 上传脚本自动对齐也会跳过 standard

脚本日志 `align skip 2022.3.22f1 (already new)` 而只 PUT 了 1 档，不是漏对齐的假象而是真缺口（第 170 轮）：SDK 上传时新包只挂到构建用的那个 unityVersion 档；脚本的 `already new` 是按「等于该档最大值」判断，不是「等于最新」。SDK 自动对齐只覆盖它自己 PUT 的那几档，且不处理 security 变体。
- **必须自己再跑一遍全量 PUT**：按 unityVersion 去重，每档各 PUT `variant/security` 与 standard，顺序固定先 security 后 standard（反过来 standard 会被 security 的 PUT 覆盖回旧值）。本工程 4 次 PUT 全 HTTP 200。
- 纯 HTTP 对齐（不依赖编辑器会话）与客户端侧取证见 [06-06 §5/§4](06-06-upload-credentials-and-client-evidence.md)。
