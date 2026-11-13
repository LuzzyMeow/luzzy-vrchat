# 插件生态：VRCFury、互动系统、bake 与优化

> 本册涉及成人向互动插件（PCS / SPS 等）。这些是 VRChat 社区公开发行的付费/免费插件，这里只记录**结构与排查方法**，不含任何资源本体。

---

## VRCFury

VRCFury 常见用途是把 controller / menu / parameters 打包成一个 FullController 或 feature component。

排查：

- 搜 `VRCFury` component。
- 用反射或 Inspector 看它引用的 controller、menu、parameter asset 是不是空的。
- **bake 后功能消失**：检查 VRCFury 生成的资产有没有持久化，有没有被 optimizer 删掉。

VRCFury 和 Modular Avatar 可以共存，但同一个功能不要两边都装 —— 会出现重复菜单和重复参数。

---

## Bake 原则

- **bake 前保留源 avatar**，永远不要就地破坏源 prefab。
- 生成上传用 clone，命名区分清楚：`_Baked`、`_Optimized`、`_MemOpt`。
- FX controller、expression menu、expression parameters、生成的 clip **必须存成 `Assets/_Gen/` 下的 asset 文件**。
- 不要让场景长期引用内存对象或临时 generated 目录 —— 下次重新生成时引用就断了。
- **bake 之后重新检查**：脸、衣服、材质、菜单、参数、PhysBone、contacts、constraints。

### bake 后脸或衣服没了

优先怀疑：

1. generated asset / fileID 引用断了
2. renderer 的 active / enabled 状态
3. 菜单参数默认值和 bake 前不一致

**不要**因为这个就重新导入所有资源 —— 那通常修不好，还会引入新问题。

---

## MemoryOptimizer / Avatar Optimizer

- **不要对已经压缩过的 clone 再压缩一次。** 需要重压就从干净的源 avatar 重新 bake。
- 压缩后上传前检查 synced bits 和 SDK validation。
- 如果 optimizer 删掉了它认为"没用"的 renderer / shape / contact，去改 preserve 设置，或者从源 avatar 重新 bake，**不要**手工把删掉的东西再加回来。

---

## PCS（互动特效系统）

典型组成：

- 主 FX controller
- direct blendtree / 子 controller
- expression menu
- expression parameters
- target objects：Mouth、Breast、Pussy、Ass、Custom slots
- particles / audio / voice pack
- VRCFury 或 Modular Avatar installer

排查顺序：

1. PCS root 是否存在。
2. installer 引用的 controller / menu / parameters 是否为空。
3. menu 有没有真的进 avatar expression menu。
4. 参数有没有进 expression parameters，有没有超限。
5. **target / particle 的空间位置**是不是在对应身体部位，而不是 avatar root、地面或脚边。
6. particle / audio 组件和 clip 引用是否存在。

**Custom slot 不一定都有实际功能。** 很多 installer 把 Custom 1–8 当通用自定义位置槽；没配置的槽应该删掉或不进菜单，不代表缺资源。

特效位置错了，**优先修 target / socket / constraint**，不要先去改材质或菜单。

---

## SPS

典型组成：

- 一组身体部位 target / socket：Mouth、Breast、Pussy、Ass、Hand、Foot 等
- 打开/关闭各 socket 的 animation clip
- menu toggle 或 radial / 子菜单
- bool / int 参数
- parameter driver，用来联动 PCS 的选择参数或启用开关

**"一次只能开一个"**：说明 controller 用了互斥状态机。要支持多开，通常需要每个 target 一个独立 bool + 独立 layer，再加一个 aggregate layer 统一控制总开关参数。

**吸附位置不对**：查 target / socket 的位置和 humanoid bone fallback。不要拿 avatar root 当默认位置。

---

## PCSS（阴影/光照类）

典型组成：setup prefab、light/shadow controller、expression menu、expression parameters、材质和 shader、light handle / self-light handle 物体。

排查顺序：

1. 菜单是否装进了 expression menu。
2. FX controller 是否被 merge。
3. parameters 是否存在、默认值是否合理。
4. light handle 是否默认关闭、是否 local-only。
5. 材质是否真的在用 PCSS shader，texture / alpha / cutout 是否正常。

注意：**PCSS 的材质效果会随上传被别人看到**；local-only 的 handle 控制本身不会同步给远端。

---

## Grab Full Body

组成一般是：setup prefab 或 installer、contacts/receivers、PhysBone/constraints、menu/parameters、FX controller 或 MA/VRCFury merge。

- 菜单缺失 → 查 installer
- 功能缺失 → 查 contacts / PhysBones 和 FX merge
- 上传后不可见 → 查参数 sync、SDK stripping、optimizer 的 preserve 设置

---

## 互动衣服

常见结构：

- 可见的衣服 mesh
- 备用状态 mesh（正常 / 拉开 / 掉下）
- blendshape：`Down`、`Open`、`Pull`、`Tear`、`Peek` 之类
- contact receiver / sender
- PhysBone，可能开了 grabbing / posing / stretch
- 作者自带的 FX controller 和菜单参数

处理规则：

- **不要**把互动衣服当普通 `ObjectToggle` 整体关掉。
- **不要** bake 掉带 contact / PhysBone / MergeAnimator 的子物体。
- 多状态衣服必须做**互斥**：正常、拉开、掉下同一时间只能有一个主状态。
- **触发后出现两条内裤 / 多件衣服** → 修状态互斥和默认 active，不是加新 toggle。
- **拉不动** → 查 PhysBone 的 `allowGrabbing`、`allowPosing`、`maxStretch`、`rootTransform`，以及 contacts 和 FX layer weight。

---

## 材质：脸、身体、下体应该分开

黑色或透明通常来自：

- shader 缺失，或 shader variant 不支持
- material slot 错位
- alpha / cutout / mask 参数错误
- texture 引用丢失
- **不该共用的脸/身体/下体材质被联动修改**
- optimizer 合并/裁剪后 renderer / material 引用损坏

原则：

- **脸材质、身体材质、下体材质能分开就分开**，避免菜单或 MaterialSetter 误联动。
- 修材质之前先 dump：renderer → materials → shader → texture → alpha/cutout 参数。

### lilToon：用代码切 Cutout

lilToon 的编辑器程序集 `jp.lilxyzw.liltoon` 是 `autoReferenced = false`，`Assets/Editor` 里的脚本 import 不到它的类型。直接用 Material API：

```csharp
static string ToCutoutShader(string shaderName)
{
    bool outline = shaderName.Contains("Outline");
    bool lite    = shaderName.Contains("Lite");
    string s = "Hidden/lilToon";
    if (lite) s += "Lite";
    s += "Cutout";
    if (outline) s += "Outline";
    return s;
}

// Cutout + Alpha Mask Subtract（例如在身体上开一个孔）
mat.shader = Shader.Find(ToCutoutShader(mat.shader.name));
mat.SetFloat("_AlphaToMask",   1f);
mat.SetFloat("_AlphaMaskMode", 4f);   // 4 = Subtract
mat.SetTexture("_AlphaMask",   maskTex);
mat.renderQueue = 2450;
```

改材质前先克隆，别动原始资产：

```csharp
AssetDatabase.CopyAsset(srcPath, dstPath);
var mat = AssetDatabase.LoadAssetAtPath<Material>(dstPath);
// 现在改 mat，原始资产安全
```

---

## 上传流程

上传目标通常是 **baked / optimized 的 clone**，不是源 avatar。上传前确认：

- `VRCAvatarDescriptor` 存在。
- FX controller、expression menu、expression parameters 都是**持久化的 asset 文件**。
- `PipelineManager` 的 blueprint ID 符合意图：更新旧 avatar 就保留，创建新 avatar 才清空。
- synced parameter budget 没超 VRChat 限制。
- SDK validation 没有 blocker。

监控日志的阶段标记：

```text
Validating the Avatar
Creating an Asset Bundle
Building the Asset Bundle
upload start
upload progress
upload success
```

SDK 未登录时，先试项目已有的 login refresh / editor refresh 脚本；自动刷新失败后**才**让用户去 VRChat SDK 面板手动登录。

上传本身是用户动作，不要试图脚本化。

---

## 补充：结构化分册路由（2026-11-11 整理，原文保留）

- RK Player / RealKiss / GUN1 等互动插件的集成坑（音频接线、RKSActivator、口裂定标、CD 封面）：[02-11](../02-工程组织与工作流/02-11-plugin-integration-rk-realkiss-props.md)。
- SPS socket 绑定与菜单重建：[05-06](../05-动骨与穿模/05-06-sps-and-ndick-binding.md)；VRCFury SPS Plug bake 失败：[07-01 §24](../07-验证与排查/07-01-bake-and-validation.md)、[07-06 §14](../07-验证与排查/07-06-gesture-manager-automated-sim-test.md)。
- AAO 与面捕/形态键的交互（optimizeBlendShape 改名、删曲线、bones 精简）：[04-08 §5](../04-外观与表情/04-08-face-tracking-osc-and-repair.md)、[04-09 §7.3](../04-外观与表情/04-09-face-tracking-native-sdk.md)。
- 上传凭据生命周期 / SDK 3.10 禁脚本上传 / 协议 SessionState / 客户端取证：[06-06](../06-构建上传与备份/06-06-upload-credentials-and-client-evidence.md)。
- provider 归属与验收矩阵（FaceEmo / lilycalInventory / DressingTools / GM / AAO）：[07-03](../07-验证与排查/07-03-provider-playbooks.md)。
