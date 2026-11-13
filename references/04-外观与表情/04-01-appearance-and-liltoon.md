# 外观与 lilToon：亮度、单色、换色与发光

读这份文件的时机：用户要求**调亮度 / 调色 / 改发光 / 换颜色 / 改材质观感**，或问「为什么改了没反应」。

> 非 lilToon shader 也可用同一排查顺序；**属性名、取值范围与 keyword 一律按现场版本读回**，不要照搬这里的候选名。
> 先读基底 [SKILL.md](../../SKILL.md) 的视觉资产路由与 [07-01-bake-and-validation.md](../07-验证与排查/07-01-bake-and-validation.md)，**不要只按材质名字猜目标**。

## §1 先确认真实来源与属性

1. **对象精确路径 → 材质槽 → Material → Shader 及其版本 → 属性/贴图/importer → 相关动画/provider → 实际运行实例**，并检查这个材质的**共享消费者**。
2. 读属性前先确认**存在性与类型**，用对应的 Float / Color / Vector / Texture API 读取。**属性不存在或类型不符时读到的值，不能证明该功能是关闭的。** 动画能绑定某个属性，**也不等于**对应 keyword / shader 变体已启用。
3. ⚠️ 两种修改的作用范围完全不同：
   - **修改共享 Material 资产** → 影响所有引用它的消费者。
   - **替换某个 renderer 槽位的材质引用** → 只影响那个 renderer 槽位，**不会**自动改其它 renderer。
   - 另外：访问 `renderer.material` **可能生成实例**（副本），**不能把它误当成共享资产身份**。判断「改的是不是源资产」要看写入路径，不看名字。
4. 在**已检查的 lilToon 2.3.0 标准 shader** 中，以下是**候选**而非通用 API（实际范围/效果以现场读回为准）：

| 候选属性 | 大致用途 | 注意 |
| --- | --- | --- |
| `_LightMinLimit` / `_LightMaxLimit` / `_AsUnlit` | 光照限制 / 无光照混合相关 | 范围与最终效果现场读回 |
| `_MonochromeLighting` | **照明去色**程度（Range） | **不是** albedo 全局去饱和开关 |
| `_Color`、`_MainTexHSVG` | 基础色 + 主贴图 HSVG | HSVG = 色相/饱和度/明度/**Gamma**，**不是 Contrast**；本版中性值 `(0,1,1,1)` |
| `_EmissionColor` / `_UseEmission` / `_EmissionMap` | 发光 | 还要看功能设置、遮罩、混合方式**与其它发光层** |

## §2 选实现方式

- **离散整组属性 / keyword 方案**：可用持久化材质变体 + `SetObjectReferenceCurve` 切 renderer 槽位。**只复制确需独立修改的材质**；多个同效果 renderer 可以共享同一派生材质，**不要求每件都复制**。
- **连续滑杆**：优先检查真正可动画的属性，做最小分帧求值 / 图像试验。**属性存在不代表动画 shader keyword、纹理槽或所有变体行为一致。**
- **每档要说明它拥有哪些属性**（颜色 / 亮度 / 发光）。不同档位不能在未协调时**写同一属性**或**替换整材质互相覆盖**。
- **默认档还原原引用 / 原属性值。** 新材质引用、动画槽位与所有共享消费者逐项检查。

## §3 亮度

提高光照下限可能改善暗场景，**但不等于改变纹理明度**——这两件事的观感完全不同。

- **固定曝光 / 相机 / 灯光**做 A/B，并在**模型 ROI 而非整张背景**里计算亮度。
- 检查**灰雾、过曝、丢细节、方向感丢失**；暗场景与亮场景**分开验收**。
- 回退 = 恢复本次已授权的改动，**不要顺手改用户其它材质**。

## §4 单色照明与纹理去色

- `_MonochromeLighting` 控制的是**照明颜色**。⚠️ **在白光下它可能几乎没有可见差异**——这属于正常现象，不是「改坏了」。
- 若确认无变化：先用**同光强的彩色光**、真实 shader 变体与实际属性读回来验证，**不要默认改成去饱和纹理**。
- **纹理去饱和改变的是 albedo 内容**，那是另一种视觉目标；只有用户目标 / 授权包含它时才做。平均饱和度只衡量所选图像 ROI，**不证明所有场景光照都正确**。

## §5 换色与发光

- 头发 / 眼睛可能是**同一 SMR 的不同材质槽**，也可能**分属不同 renderer**——先确认结构再动手。
- 查清多张主贴图、Emission、MatCap、遮罩与 UV 通道**各自的参与方式**，再决定改哪一层。⚠️ **不要仅因「挑染」就认定 Emission 是唯一来源。**
- 默认档 + 颜色档 / 连续量按菜单结构与 provider 能力配置；明确**叠加关系**，测试切回与组合效果。
- ⚠️ **发光强度 / color 可以是 HDR 值**，**不要为了迎合图像 uint8 转换而去截断材质的 HDR 值**。

## §6 常见误判

- **Vector 动画只写一个分量可以是合法设计**；只有错误默认值或其它层写入才可能破坏其余分量。**读完整向量、keyword 与实际输出**，不要断言「所有 Vector 必须四分量写」。
- 核对当前 source / Material 实例、生成副本、shader feature 与**最后写者**。**新参数 / 新属性写入后无视觉变化，不能靠反复调用「修好」。**
- **纹理看起来正常，不排除** UV / importer / 色彩空间 / alpha / shader / 模型问题；单张 atlas 或单次 A/B **不能排除整类根因**。
- 法线贴图 / 重算法线能改变明暗，**不修真实几何相交**；默认 renderer 不显示新增 shape 可能是**场景旧数组覆盖**，**不能用材质掩盖几何问题**。

## §7 交付

对齐四项：**菜单/参数 → 实际到达状态 → renderer 槽位/材质实例/属性 → 对应输入图像**。记录：共享范围、稳定源与生成边界、默认恢复方式、组合效果、源纹理足迹、构建大小证据状态。

证据分层标注：编辑态 / 隔离 Animator / 模拟器 / VRChat 客户端测试**分别标证据**，未跑的写 `NOT_RUN`。**平台 shader 编译或真实客户端效果未测时，不能声称兼容全部平台**；新 build / upload 按相应授权执行。

---

## 补充：lilToon 官方文档核对（2026-11-11 联网核实）

> 来源：lilToon 官方站 <https://lilxyzw.github.io/lilToon/> 与 <https://lilxyzw.github.io/lilToon/ja_JP/first.html>（2026-11-11 提取）；BOOTH <https://lilxyzw.booth.pm/items/3087170>；MIT License。

### shader 变体一览（官方）

| 名字 | 用途 |
|---|---|
| `lilToon` | 主 shader，一般用途都用它 |
| `_lil/[Optional] lilToonOverlay` | 材质上叠加的透过 shader，已去除不必要 pass，比叠普通透过 shader 低负荷 |
| `_lil/[Optional] lilToonOutlineOnly` | 只画轮廓线；硬边模型另备平滑法线网格赋它，轮廓更漂亮 |
| `_lil/[Optional] lilToonFurOnly` | 只画 fur 毛部分；叠加表现 / cutout+透过 fur 组合 |
| `_lil/lilToonMulti` | 用 shader keyword 的版本；材质多时 build 尺寸易变大，注意 |
| `Hidden/lilToonLite` | 大幅轻量版；建议用通常版做好材质再转换，不要直接设 Lite |

**`_lil` 内的是特殊 shader，基本用途选普通 `lilToon`**（官方 STEP 2 口径）。逆向工程占位 shader 与 lilToon 同名（`Hidden/ltspass_opaque`/`Hidden/lilToonOutline`/`lilToon`）会按名字抢 UsePass → 描边/眼镜/亮度插件全坏，修法见 [07-01 §33](../07-验证与排查/07-01-bake-and-validation.md)。

### 贴图槽位对应（官方 STEP 4）

| 贴图种类 | 赋值位置 |
|---|---|
| 主贴图 | 基本色设定 → メインカラー → テクスチャ（`_MainTex`） |
| 法线贴图 | ノーマルマップ设定 → ノーマルマップ |
| 轮廓 mask | 輪郭線设定 → 輪郭線 → マスクと太さ（`_OutlineWidthMask`） |
| 影 mask | 影设定 → 影 → マスクと強度 |
| MatCap | マットキャップ设定 → マットキャップ |
| MatCap mask | マットキャップ设定 → マスク |
| RimLight mask | リムライト设定 → 色 / マスク |
| Emission（mask） | 発光设定 → 発光テクスチャ → 色 |

透明贴图要把**描画モード**改 Cutout 或半透明（官方 STEP 5）；编辑模式分「简易设定」（只改色）与「详细设定」（新设/移行/细改），预设可一键应用与自存。

### 构建期自动优化（与 bake 噪声的关系）

官方「轻量」口径：**编辑器自动改写 shader 开关功能、剔除不需要的功能**以压低负荷与 build 尺寸。这解释了构建期 lilToon 重写 `lts*.shader` + `lil_common_input_opt.hlsl` 产生的 `CBUFFER_START` 瞬态报错（判无害，不要回滚 lilToon/清 Library）——见 [07-01 §36](../07-验证与排查/07-01-bake-and-validation.md)。

### 版本与分发（官方）

- 对应 Unity 2022.3 / 2023.1–2023.3；shader model 通常版 SM4.0·ES3.0、轻量版 SM3.0·ES2.0、fur SM4.0·ES3.1+AEP/3.2、tessellation SM5.0；Built-in/LWRP/URP/HDRP 均对应（动作确认 Unity 2022.3.22f1）。
- 分发 VRChat 向作品：推荐引导 VCC 安装；同梱 shader 用 BOOTH 下载页快捷方式或原样 unitypackage；**不要把 shader 本体与作品打进同一个 unitypackage**（用户导入时可能以旧版覆盖）。
- 安装三选一：BOOTH zip 内 unitypackage 拖入 / VCC·ALCOM 加 lilToon VPM 源 / UPM git URL。

## 发色整体改色：白灰渐变 remap（本工作区实测 2026-11-12 · PLAN-004 沐梓溪小狼）

**症状/需求**：把蓝白头发改为「白为主、自上而下灰过渡」，耳朵/尾巴/手套同改，默认生效、可回退。
**坑1（蓝青残留来源）**：lilToon 的实际基色输入是 `_BaseMap`/`_BaseColorMap`，只换 `_MainTex` 不够；另有 `_Color2nd`（青）、`_Main2ndTex`/`_Main3rdTex`、`_OutlineTex`/`_RimColorTex` 带色。改色必须全量处理：贴图换 remap 版 + 用 `ShaderUtil.GetPropertyType==Color` 枚举去饱和（**跳过 `_ega*`/`_e2ga*` 等向量伪颜色参数**）。
**坑2（渐变方向）**：头发 UV 与世界高度相关性弱（实测 corr 0.41–0.87、符号不一）→ 贴图 V 向 ramp 不可靠。正解＝**按部件** UV→rest 世界高度 三角形光栅化 bake t 图（numpy 重心插值，2048²），`out = lerp(1.0, 0.55, t^1.4) * clip(L/0.75,0,1)^0.7`，保留 alpha 与发丝明暗（L=Rec.601）。白为主参数组：除数 0.75、gamma 0.7、t^1.4；v1 的「饱和度映射+L/0.9」被用户否决（无白色、偏灰）。
**共享 atlas**：多部件共用一张贴图时按部件出独立纹理/材质（各自只采样自己的 UV island，互不干扰）；不要试图单纹理编码多部件高度。
**回退机制**：改前把每渲染器槽位原材质 GUID 记 json（`docs/_r511_wg_backup.txt`），`R511WG.Rollback` 一键恢复；往返测试后再落最终态。
**判据**：预览目视白为主/上白下灰/无蓝青；烘焙读回五部件材质均为 WG2；EP 预算不变（249）；场景 dirty=false。
**工具**：工程 `docs/_tools/build_wg2.py`（高度法）、`hair_wg_remap.py`（v1 饱和度法，仅留档）、`Assets/Editor/R513WG2.cs`（Export/Assign）、`R511WG.cs`（Apply/Rollback）、`R512WGPreview.cs`（Play 预览，手套/尾巴临时显示不存盘）。
**结局注记**：用户预览后仍不满意（发尾渐变观感不足），2026-11-12 指令整体回退；R511 Rollback 一键恢复六槽原材质并目视确认。方法留档备用；另发现 LilLeo 狼 FX `Tail` 层反相接线（Tail=1→隐藏，菜单「小狼尾巴」默认=隐藏），属作者设计/缺陷待定，未擅自改。
