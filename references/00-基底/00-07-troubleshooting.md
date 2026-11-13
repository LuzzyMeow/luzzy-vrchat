# 症状 → 根因清单

排查通则：**先 bake 再判断**。没 bake 的编辑器视图里，MA 相关的一切视觉问题都可能是假象。

---

## 黑脸 / 脸是黑的

按可能性排序：

1. **材质槽为空** —— renderer 有 null 材质槽 → 洋红或黑。破坏性资产操作之后尤其要查所有材质槽。
2. **shader variant 不对** —— lilToon 有很多变体（Cutout / Transparent / Outline / Lite）。给这个 mesh 选错变体 → 不可见或全黑。
3. **法线反了** —— 导入时法线翻转 → 从错误的一侧受光。在 Blender 里修，或调 `ModelImporter.importNormals`。
4. **贴图丢失** —— `_MainTex` 为 null → 黑。导入后在 Inspector 里确认材质。
5. **renderer 被禁用** —— SkinnedMeshRenderer 的 `enabled = false`。
6. **面捕 merge 覆盖了 FX** —— 装完面捕才出现的话，查是不是面捕的 FX layer 盖掉了原来控制脸部的 layer。
7. **搞错了哪个 renderer 是脸** —— 有些 avatar 的 `Body` SMR 其实只是**脸/头**，身体皮肤在另一个 renderer 上；有些 avatar 的 `Body` 是脸+身体合并的。**永远不要假设 renderer 名字对应它的内容**，先 dump 一遍。

---

## 材质粉色 / 贴图黑或透明

- 粉色 → shader 没导入，或材质引用了被重命名/丢失的 shader。**先导入正确的 shader 包**。
- 黑/透明 → 查 material slot 错位、alpha/cutout、render queue、texture import 设置、shader keyword。
- optimizer 合并材质之后出现 → 从源 avatar 重新 bake，别手工修 clone。

---

## 穿模（衣服穿过身体）

1. **没 bake 就在下结论** —— 头号误报来源。先 bake。
2. **衣服需要的身体形态键没激活** —— 查 `BlendshapeSync` 指向哪个身体 blendshape。**不要缩身体**。详见 [00-05-mesh-blendshape.md](00-05-mesh-blendshape.md)。
3. **服装参数值不对** —— int 参数 gate 失败，衣服和身体状态对不上。
4. **FBX 导入 scale 错了** —— 见 globalScale 陷阱。
5. **作者已知限制** —— 读说明，有些区域作者写明了修不了。

---

## 黑洞（胸口暗腔）

**根因**：压平类 blendshape（`(Wear_Bra)_*` / `Shrink_*`）仍然生效，但对应的遮盖网格已经不在 —— 身体被压进去，皮肤照常渲染，形成暗腔。

修复顺序：

1. 找出所有设置压平形态键的 `ShapeChanger`。
2. 确保每个都被正确的衣服 gate 住（服装 int 参数 + Idle 状态，见 [00-02-modular-avatar.md](00-02-modular-avatar.md)）。
3. 衣服被 toggle 掉时，它的 ShapeChanger 也必须跟着失效。

---

## 三白眼（眼白露太多）

两个 blendshape 叠加造成：`EyeIrisSmall`（缩瞳露白）+ `EyeWide`（眼睑回缩露白）。

三处一起改：动画 clip 里 `EyeIrisSmall = 0`；`EyeWide` 上限从 100 降到 ~25；VRCFT `LocalSettings.json` 里 `eyeWide.max` 调到 0.45 左右。详见 [00-04-face-tracking.md](00-04-face-tracking.md)。

---

## 身体缺手 / 缺腿 / 被压扁

查 body blendshape、`ObjectToggle`、`ShapeChanger`、`BlendshapeSync`。

全脱之后还缺 → 某个隐藏/压缩状态没恢复。**修参数默认值和状态机**，不要继续叠加新的压缩。

---

## 衣服不见了

- 衣服 prefab 没实例化
- MA `ObjectToggle` 默认关
- bake 后 generated asset 丢了
- 菜单参数默认值不一致

---

## 菜单不出现 / 位置不对

1. MenuItem 是孤儿 —— 必须挂在能从 avatar 根到达的 SubMenu 之下。
2. 重复项 —— 脚本重复运行会叠加，重建前先按名字 `DestroyImmediate`。
3. `ControlType.SubMenu` 但没有子物体 → 空文件夹。
4. `ModularAvatarMenuInstaller.menuToAppend` 或 children source 为空。

---

## 参数超限

1. 先区分**远端可见**和**本地状态**。
2. 本地的设 local-only / `networkSynced = false` / `saved = false`。
3. 删掉菜单和 controller 都不用的参数。
4. 仍超限再用 MemoryOptimizer / Avatar Optimizer 压缩。

**不要盲删互动插件的参数** —— 它们可能通过 animator、parameter driver 或 contact 间接使用。

---

## 头发穿身体

不是 mesh 问题，是 collider 没挂给头发的 PhysBone。见 [00-03-physbone-dynamics.md](00-03-physbone-dynamics.md)。

---

## 互动衣服失效 / 出现两条内裤

状态互斥没做对，或者默认 active 状态冲突。不要用整体 `ObjectToggle` 硬关，不要 bake 掉带 contact / PhysBone 的子物体。

---

## Missing script

包导入顺序不对、插件版本不匹配、或 asmdef 编译失败。按依赖顺序重新导入依赖包。

---

## Ctrl+R 不触发编译

DLL mtime 不变 = 键盘事件没进 Unity。多半是别的进程抢了前台焦点（VR 串流服务是常见元凶）。见 [00-01-editor-automation.md](00-01-editor-automation.md) 的焦点抢占一节。

---

## Compile error

**停止后续一切导入和上传**。先修 `Assets/` 里的脚本错误或缺失依赖，再继续。带着编译错误往下走只会让问题叠加。

---

## 上传异常

| 报错 | 含义与处理 |
|---|---|
| `VRC SDK is not logged in` | 先试项目已有的 refresh / login 脚本；失败后让用户手动登录 SDK |
| `owner mismatch` | blueprint ID 属于别的账号。**停止**，不要自动清空 |
| `copyright ownership agreement not confirmed` | 需要在 SDK 或网页确认协议 |
| `empty blueprintId` | 只有用户明确要新建 avatar 时才作为新 avatar 上传 |
| `duplicate bundle already uploaded` | 内容没变化，**通常不是错误** |
| `Build validation failed` | 按 SDK validation 逐项修，不要跳过 |
| Upload 卡住 | 确认没有并发的 build/upload；读 SDK log 和 Unity Editor log |

---

## 渲染验证时的高光色斑

不是贴图坏了。方向光强度超过 0.8 会在光滑的眼球 shader 上产生高光色斑。渲染验证用 `intensity = 0.6` 方向光 + `0.85` 平坦环境光。

---

## 补充：结构化排查路由（2026-11-11 整理，原文保留）

> 本册是外部 MIT 原文的通用症状清单；本工作区实测的深层根因已按主题结构化到下列分册。查症状时先按本册定位大类，再进对应分册看机制与判据。**注意区分**：本册「高光色斑」是方向光>0.8 的渲染伪影；面捕眯眼时高光漂出眼球是 `hightlight_*` 共用顶点问题（[04-09 §8](../04-外观与表情/04-09-face-tracking-native-sdk.md)），两者不同症。

| 症状大类 | 结构化分册 |
|---|---|
| 黑脸 / 粉材质 / 贴图不见 | 本册 §黑脸/材质粉色；同名 dummy shader 覆盖 lilToon → [07-01 §33](../07-验证与排查/07-01-bake-and-validation.md)；多色板变体错配 → [05-08 §15](../05-动骨与穿模/05-08-cross-avatar-garment-fit-newfit.md) |
| 穿模 / 黑洞 / 缺件 | 本册 + [00-05](00-05-mesh-blendshape.md)；量化与外推 → [05-08 §13](../05-动骨与穿模/05-08-cross-avatar-garment-fit-newfit.md)；袜子/手套/口袋 → [05-01 §12–§15](../05-动骨与穿模/05-01-outfit-fit-and-clipping.md)、[05-11](../05-动骨与穿模/05-11-skinning-binding-and-geometry-repair.md) |
| 菜单不出现 / 开关没反应 / 默认态错 | [03-06](../03-菜单与参数/03-06-ma-menu-build-by-reflection.md)；OT 极性全解 → [04-06 §14](../04-外观与表情/04-06-ma-reactive-components.md)；EP 权威规则 → [03-01 §10](../03-菜单与参数/03-01-parameter-and-menu-audit.md) |
| 参数超限 / duplicate definition | [03-01 §10](../03-菜单与参数/03-01-parameter-and-menu-audit.md)（位宽权威 API、内置参数 IsLocal/AFK） |
| 面捕不动 / 闭眼 / 瞳孔不随视线 | [04-08](../04-外观与表情/04-08-face-tracking-osc-and-repair.md)（OSC）/ [04-09](../04-外观与表情/04-09-face-tracking-native-sdk.md)（原生） |
| 上传失败 / 上传成功但游戏是旧的 | [06-05](../06-构建上传与备份/06-05-avatar-package-version-collision.md)（多包并存）/ [06-06](../06-构建上传与备份/06-06-upload-credentials-and-client-evidence.md)（凭据与客户端取证）/ [07-01 §30/§35](../07-验证与排查/07-01-bake-and-validation.md) |
| 烘焙后功能消失 / Play 验证不动 | [07-01](../07-验证与排查/07-01-bake-and-validation.md) / [07-09](../07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md) |
| MCP 连不上 / 工具失效 | [01-06](../01-环境与工具链/01-06-project-launch-and-mcp-bridge.md) / [07-09 §8](../07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md) |
