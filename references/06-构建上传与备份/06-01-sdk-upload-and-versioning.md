# SDK 面板、上传与版本化

读这份文件的时机：准备 Build & Test / Build & Upload、要动 `blueprintId` 或缩略图、要判断 SDK 报警告到底要不要处理、或要把「上传」这一步交给用户时。

> **第一条铁律（不可绕过）**：**上传是用户动作。** 登录 VRChat 账号、点 Build & Test、点 Build & Upload、以及面板上的二次确认，都由用户手动完成。Agent 不脚本化这几步，也不代填凭据。

## 1. 为什么不能无头上传

VRChat SDK 的 `VRCSdkControlPanel.TryGetBuilder()` 需要 Editor GUI 初始化。在 `-batchmode -nographics` 下 Builder 实例不会注册，`TryGetBuilder` 拿不到 builder，**上传必定失败**。因此：

- 构建/上传**必须**让 Unity 以 GUI 模式运行；
- 不要设计「全自动无头构建上传」方案；
- 脚本化能做的只有**触发**（见第 4 节），真正的进度、登录与确认仍在面板上。

## 2. 上传面板上的字段

| 字段 | 说明 | 谁改 |
| --- | --- | --- |
| Avatar Name | 展示名 | 用户确认；Agent 可以准备建议值 |
| Thumbnail | 缩略图（见第 3 节） | Agent 可准备好图片资产并指定，最终以面板显示为准 |
| Description | 说明文本 | 用户 |
| Content Warnings / Tags | 内容分级标记 | **用户判断**（涉及账号与社区合规） |
| Release Status | `Private` / `Public` 等发布范围 | **用户决定**；Private 用于自测 |
| Version / 平台 | PC / Android / iOS 构建目标 | 用户确认 |

**Agent 不替用户决定** Release Status、Content Warnings、Description 的合规内容；可以准备事实性内容（名称候选、已实现功能清单），最终由用户在面板上确认。

## 3. 缩略图

上传面板需要一张缩略图。可以先用编辑器脚本把相机对准头像拍一张，保存到工作目录，再由用户在面板上选择或确认。

```csharp
// 简化形状：把相机放到头像正面，渲染到 Texture2D 再编码成 PNG
var camera = new GameObject("ThumbCam").AddComponent<Camera>();
camera.transform.position = avatarRoot.transform.position + new Vector3(0f, 1.4f, -1.6f);
camera.transform.LookAt(avatarRoot.transform.position + Vector3.up * 1.2f);
var rt = new RenderTexture(1024, 1024, 24);
camera.targetTexture = rt;
camera.Render();
// → RenderTexture.active + ReadPixels → EncodeToPNG → File.WriteAllBytes("Logs/thumbnail.png", ...)
```

要点：

- 拍完之后**恢复**相机与 `RenderTexture.active` 状态，并销毁临时相机——临时对象留在场景里就是「交付物不干净」。
- 图片落到工作目录/`Logs` 下，**不要**直接覆盖别人的资产。
- 缩略图不需要通过 MCP 上传；它只是面板输入。

## 4. 脚本化「触发」，而不是脚本化「上传」

官方 **Public SDK API** 提供受支持的触发入口：

```csharp
[MenuItem("My Tools/Build Selected Avatar")]
public static async void BuildSelectedAvatar()
{
    var avatar = Selection.activeGameObject;
    if (!VRCSdkControlPanel.TryGetBuilder<IVRCSdkAvatarBuilderApi>(out var builder)) return;
    try { await builder.BuildAndTest(avatar); }
    catch (Exception e) { Debug.LogError(e.Message); }
}
```

注意：

- `TryGetBuilder` **返回 false 就静默退出**——这是设计上的守卫。如果它总是 false，先确认 Unity 是 GUI 模式、且 SDK 面板已初始化。
- 构建是**异步**的：`await` 之后的代码要在构建结束后才跑。
- `BuildAndTest` 会**启动 VRChat 客户端**，不是纯编辑器内操作。要保证用户在场。
- 构建开始/结束、上传成功/失败等事件可用 **Build Pipeline Callbacks and Interfaces**（<https://creators.vrchat.com/sdk/build-pipeline-callbacks-and-interfaces/>）。
- 官方提醒：仍用**反射**访问 SDK 内部的做法应尽快改用公开 API。Agent 写脚本时优先公开 API；确实只能用反射时（例如读 `PipelineManager.blueprintId`），把反射集中在一个 helper 里并注释原因。

## 5. blueprintId：新建头像容器时的必查项

`blueprintId` 是 SDK 用来把本地工程对象与线上已上传头像关联的标识。

- 从**别人工程/别人 prefab 复制**来的头像，可能带着原作者的 `blueprintId`——这会导致你「更新」到别人的上传条目。
- 从模板新建的头像容器，也要清一次。
- **清完之后，在打开 SDK 上传面板之前再检查一次序列化内容**：面板打开时会把当前值写回，若你在面板打开后才清，可能被旧值覆盖。

清理用反射（无公开 API）：

```csharp
foreach (var component in avatarRoot.GetComponents<Component>())
{
    if (component == null || component.GetType().Name != "PipelineManager") continue;
    var field = component.GetType().GetField("blueprintId");
    if (field == null) continue;
    if (!string.IsNullOrEmpty(field.GetValue(component) as string))
    {
        field.SetValue(component, string.Empty);
        EditorUtility.SetDirty(component);
    }
}
```

检查顺序固定为：克隆 → 清 ID → 保存 → **重开工程/重新加载** → 再读一次确认仍为空 → 再让用户打开上传面板。

**逆向 prefab 的 blueprintId 隔离（第 167 轮实测 2026-11-08，2026-11-11 结构化并入）**：把 AssetRipper 导出的 prefab（如「赛博小狗」）放进当前工程时，它自带 `PipelineManager.blueprintId=avtr_f52d01fc-...`。官方 `VRCPipelineManager` 文档明确：*Don't have more than one PipelineManager in the scene when building… You may end up uploading to the wrong blueprint ID*（<https://creators.vrchat.com/sdk/vrcpipelinemanager/>）。
- 实例化后**立即清空 blueprintId** 并 `SetActive(false)`；读回 `SerializedObject(pm).FindProperty("blueprintId").stringValue` 确认。
- 构建/上传前复核**所有 inactive 头像**的 blueprintId 为空；判据=SDK 面板 Selected Avatar 仍是目标头像。

## 6. SDK 警告分级：读它，而不是盲消它

SDK 面板的警告分三类：

| 级别 | 典型内容 | 处理 |
| --- | --- | --- |
| **必须先修** | 缺 `VRCAvatarDescriptor`、骨骼/Humanoid 映射失败、参数超上限、FX 层缺失、Missing 脚本、缩略图为空导致的硬报错 | 修掉；不修就不能上传 |
| **应当解释** | 未使用的动画层/多余参数、性能等级偏高、Poly/材质槽过多、PhysBone 数量偏多 | 先判断是否有实际功能依赖；能删就删，不能删就**记录理由** |
| **可以保留** | 与本次改动无关的既有提示、平台相关提示、厂商资产自带提示 | 记录，不动 |

**不要**为了「面板全绿」而删有功能依赖的组件/参数。每一条被消除的警告都应能说出「为什么它能被消除」。

典型误消案例：

- 删掉「未使用的 parameter」→ 某个菜单控件失去驱动，点了没反应；
- 删除看似重复的 merge animator → 原衣服不再显示；
- 把 PhysBone 组件当成「多余」删掉 → 该骨骼不再有物理；
- 把参数默认值改成 0 以消警告 → 用户进游戏后默认外观变了。

## 7. 参数与菜单的硬性上限

上传前必须核对：

- **表达参数总体量**：参数有成本（bit）预算，超了 SDK 直接拒绝或报警；具体公式与逐项核算见 [03-01-parameter-and-menu-audit.md](../03-菜单与参数/03-01-parameter-and-menu-audit.md)。
- **每页菜单控件数**：一页菜单控件数量有上限，超了要么拆页要么合并；见 [03-01-parameter-and-menu-audit.md](../03-菜单与参数/03-01-parameter-and-menu-audit.md)。
- **参数名长度/命名规范**：保持与厂商一致，不要批量重命名（会打断别人的动画与菜单引用）。

## 8. 脚本上传（Scripted Upload）的适用面

「脚本上传」指的是用公开 API **触发**构建的脚本入口（第 4 节），不是无头 CI。适用判断：

- ✅ 适合：本地 GUI 编辑器内，用户在场，Agent 写一个 `[MenuItem]`，用户点它；减少手工步骤、保证构建参数一致。
- ❌ 不适合：CI/无人值守、`-batchmode` 自动上传、自动填账号。

另外：`Build & Test` 与 `Build & Upload` 是**两个不同的动作**。改完资产后先 Build & Test 在游戏里看实际效果（NDMF/MA 只在构建时烘焙，编辑器里看到的不是最终结果），确认后再 Build & Upload。

## 9. 备份：上传前一定要能回去

上传前的可回退点（完整清单见 [06-02-backup-and-package.md](06-02-backup-and-package.md)）：

1. **工程层**：整个工程目录的副本或版本控制提交（推荐 git + `.gitignore` 排除 `Library/`、`Temp/`、`obj/`）。
2. **资产层**：本次改动涉及的工作 prefab / 场景 / 菜单 / 参数资产，导出包或复制到带日期的备份目录。
3. **记录层**：`Logs/` 下的报告（改了什么、依据什么、验证到什么程度）+ 当前 `blueprintId` 状态。

**没有可回退点就不要开始上传流程**——尤其当这次改动包含不可逆操作（重命名对象、改骨骼、动材质）时。

## 10. 上传后的验证

上传成功后，验证属于用户（在 VRChat 里进图看效果、切开关、试表情）。Agent 的责任是：

- 把「应该看到什么」写清楚（哪些开关、期望顺序、默认值）；
- 把已知限制写清楚（哪些只能在 PC/Quest 之一生效、哪些依赖特定手势）；
- 把本次改动与依据归档到 `Logs/`，方便下次对照。
