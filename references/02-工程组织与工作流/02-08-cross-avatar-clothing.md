# 跨模型衣服适配：NewFit / DressingTools / AvatarDresser

> 本工作区调研 2026-10-19。对象：把模型 A 的衣服适配到模型 B（例如小狼 `LilleoWolf` → 小猫 `Kipfel`）。
> **状态：NewFit V1.1.6 已下载并导入 `<VRCHAT_ROOT>\projects\LuzzyMeow`（2026-10-19）；DressingTools / AvatarDresser 未安装。**

## 1. 结论：最匹配的是 NewFit

**NewFit – Refit Clothes With Physics**（作者 KumekuMods）——商品页原文：

> “Refit clothes however you want, using clothing physics and body collision. No clipping.”
> “**Any clothes to any avatar. No conversion profiles needed.**”

这与你说的「同一套衣服适配不同模型」完全对应。

- Jinxxy 商品页：<https://jinxxy.com/KumekuMods/NewFit>
- **BOOTH 商品页（确认是同一插件）：<https://booth.pm/zh-cn/items/8960065>**
- 作者页：<https://jinxxy.com/KumekuMods> / <https://kumeku.booth.pm/>
- 演示（把人类衣服改到兽人模型）：<https://youtu.be/i9Mfai8CH0E>
- 版本：**V1.1.6（2026-10-15）**，新增对称编辑。
- 价格：Reddit 讨论里为 **约 15 美元**；**BOOTH 商品页明确标价 `2,500 JPY`（2,500 日元）**，同一作者/同一插件。
- **没有找到免费试用 / 免费版 / 开源版**。Jinxxy / BOOTH 都是购买后下载；不要从 `forum.ripper.store` 等盗版站拿。

**本机安装状态（2026-10-19）**：已从用户提供的 BOOTH 包导入 `<VRCHAT_ROOT>\projects\LuzzyMeow\Packages\com.kumeku.newfit`（V1.1.6；`.unitypackage` 2,080,801 B）；
Unity 2022.3.22f1 无头编译 **0 error**；菜单 **`Tools ▸ NewFit`**。包是 embedded package，删除该文件夹即可卸载。

**资源库归档（2026-10-19）**：原始购买包已存入 `<VRCHAT_ROOT>\resources\09_工具与插件\NewFit\NewFit-1.1.6\NewFit-1.1.6.unitypackage`（2,080,801 B，SHA-256 `0C226D...2D6`），并附 `README.md` / 原始 README。仅限本人使用，不要二次分发。

## 2. NewFit 能做什么

1. **物理重拟合**：用 Unity 的布料物理 + 身体碰撞，把衣服“推”到目标身体上。
2. **任意模型 → 任意模型**：官方明确说“不需要 conversion profile”。
3. **摆姿势/改形状**：掀起衣服、卷袖子、拉裤腿，直接拖拽/旋转/缩放。
4. **自动权重绘制**：拟合完后给衣服重新刷权重，让衣服跟随身体。
5. **Fix Movement Clipping**：自动跑一组动作、检测穿模，生成身体收缩 BlendShape（即使不改衣服也有用）。
6. **布料参数**：弹性、厚度、收缩等滑条。
7. **保存为 Toggle 或 BlendShape**：可接菜单/开关，也可给其他 gimmick 用。
8. **对称编辑**（1.1.6）。
9. **非破坏性**：BOOTH 页明确写「Modular Avatar・NDMFと併用可能」——可与 MA / NDMF 共存，不会把原衣服资产直接改死。

## 3. 官方对“兽人/不同体型”的提醒

NewFit 商品页专门写了 “Notes On Refitting to Furry Avatars”：

- 自动权重绘制对兽人模型 **不是完美的**（能用的，但不如人类标准模型稳定）。
- 先用 Unity 原生 Move/Scale 把衣服尽量摆到“穿在身上”的位置，**袖子/裤腿末端越接近越好**。
- 对兽人：**取消勾选 `Delete Inside Geometry`**，用自带的手动删除工具删不需要的几何。
- **移动衣服之前**先在 Sliders 里调好参数（尤其 `Shrink`）。
- 保存时把 `Fix Movement Clipping` 下的 `Extra Shrink` 调大，建议至少 **5 cm**。

## 4. 安装路径（本机已执行）

1. 在 Jinxxy / BOOTH 购买后下载 **NewFit 的 `.unitypackage`**（或作者给的 zip）。
2. 放到 `<VRCHAT_ROOT>\tools\downloads\`。
3. 打开目标工程（建议先装到 **`<VRCHAT_ROOT>\projects\LuzzyMeow`**，即小猫工程；备份/ git 先提交）。
4. Unity：`Assets → Import Package → Custom Package...` 选该文件；或 Editor 脚本用 `AssetDatabase.ImportPackage(path, false)`。
5. 导入后看菜单（通常 `Tools` 下会出现 NewFit 窗口；以包内 README 为准）。
6. 先复制工程或 git commit，再对衣服做拟合。

> 注意：目前没有找到 NewFit 的公开 VPM 仓库；它是付费 Unity 包，不要把它当成 VPM 包直接 `vrc-get install`。

## 5. 免费替代（不是 NewFit）

### A. DressingTools 2（免费）

- 文档：<https://dressingtools.chocopoi.com/docs/>
- GitHub：<https://github.com/poi-vrc/DressingTools>
- 定位：**非破坏式衣柜/衣橱（cabinet/wardrobe）**，把多件衣服穿到一个 avatar 上，管理开关/菜单/BlendShape。
- 对“不兼容 wearable”：有 **Mapping Editor**，可手动把衣服骨骼映射到 avatar 骨骼；支持 Auto / Override / Manual 三种模式。
- **它不做网格重塑**：衣服本身形状不合适时，DressingTools 只能做骨骼映射，不能像 NewFit 那样用物理把衣服贴合到新身体上。
- 安装：VPM 仓库 `https://vpm.chocopoi.com/index.json`，包名 **`com.chocopoi.vrc.dressingtools`**；也可 OpenUPM / `.unitypackage`。

### B. AvatarDresser（免费，MIT）

- GitHub：<https://github.com/SophieBlueVR/AvatarDresser>
- 定位：给 VRChat avatar **加配饰/简单衣服**，自动匹配骨骼/缩放；可生成开关动画。
- 限制：**不是跨身体重拟合**，复杂衣服/体型差异大时不够。

### C. Blender 手工

- CATS + 权重传递 + 雕刻，最可控，但最慢；适合最终精修。

## 6. 我们这两个模型的具体判断

- 小猫 Kipfel vs 小狼 LilleoWolf：**核心人形骨位置几乎相同**（≤0.7 mm），但小猫根缩放 **1.12×**，且 `Body_Base` 顶点数 **5,451 vs 5,111**、绑定骨 **181 vs 153**。
- 因此：
  - **骨骼动画/姿势**：可迁移，先归一化 1.12 缩放。
  - **衣服**：骨骼可映射，但**身体 mesh 不同**；DressingTools 只能映射骨骼，衣服的形状仍可能穿模 → 需要 **NewFit 的物理重拟合** 或 Blender 手工改。
- 结论：如果目标是“把小狼的衣服真正穿到小猫身上”，**NewFit 是最合适的工具**；DressingTools 只能作为骨骼/衣橱管理的辅助。

## 7. 待办

- [x] 用户购买 NewFit（$15 / ¥2,500）或提供已购 `.unitypackage`。
- [x] 导入 `LuzzyMeow`（V1.1.6，`Packages/com.kumeku.newfit`，编译 0 error）。
- [ ] 按第 3 节兽人注意事项做一次“小狼衣服 → 小猫”的拟合。
- [ ] **实际绑骨**（`SMR.bones` / `rootBone` → 小猫骨架）——只复制 mesh 不算。
- [ ] 按工作区铁律 6：GM Play 打开菜单开关衣服 + **切 Walk / Run 移动态**截图，确认跟骨；再做 Build & Test。
- [ ] 如果用户不想买：先用免费 DressingTools 做骨骼映射，接受可能的穿模，或走 Blender 手工。

## 8. 骨骼绑定与移动态验收（用户 2026-10-19 硬规矩）

**不管衣服模型间转换，还是单独新增模型 / 道具，都必须实际补骨并绑到目标 avatar 骨架上；只复制 mesh / prefab = 穿不上、用不了。**

### 8.1 小狼菜单内容的实测结构

- 小狼 `衣服` 下有 **10 套 outfit**：`MonoRed_Mamehinata` / `Hopscotch_LilLeo` / `yurumokopajamas` / `AngelTechwear` / `MameSamue_Mamehinata` / 
  `Shaka_Shaka_Blue_Mamefriends` / `samehinata` / `Mamehinata特攻服` / `YL_Mame_KAREShirt` / `Blue_Logo`。
- 这些 outfit 的 SMR **共用主 `armature`**，骨骼名带 `$GUID` 后缀（如 `Hips$20e...` / `Head$671...`），直接保存 outfit 文件夹会留下大量**外部骨骼引用**：

| outfit | SMR | 外部骨引用 |
|---|---:|---:|
| MonoRed | 12 | 1608 |
| AngelTechwear | 7 | 777 |
| Shaka_Shaka_Blue | 8 | 632 |
| Hopscotch | 7 | 611 |
| Mamehinata特攻服 | 5 | 500 |
| yurumokopajamas | 7 | 280 |
| samehinata | 7 | 252 |
| MameSamue | 3 | 81 |
| Blue_Logo | 2 | 18 |
| YL_Mame_KAREShirt | 1 | 12 |

- 顶层配件（`Armband` / `Cape_Main` / `Cat_Ear` / `Gloves` / `Socks` / `Whitle` 等）大多绑定 `armature` 的 **125 根骨**；道具多挂在手/头骨链上。

### 8.2 正确绑骨顺序

1. 取出目标 SMR 的 `sharedMesh` / `sharedMaterials` 与骨骼数组，**不要只复制 GameObject**。
2. 对每根骨：去掉 `$GUID`，按基础名在小猫骨架里找同名 Transform；找不到的（衣服专属飘带、耳机、尾巴等）挂到**最近的已映射父骨**，并记录。
3. 设置 `SkinnedMeshRenderer.bones`、`rootBone`；根骨通常是 `Hips`，不要留空。
4. 把对象放到小猫骨架下，确认 `localPosition/localRotation` 不把衣服甩飞；体型差异用 **NewFit** 拟合/刷权重。
5. 加菜单 Toggle 后，必须做移动态验收（§8.3）。

### 8.3 Gesture Manager 移动态验收（必做）

1. `Tools → Gesture Manager Emulator` → `Enter Play Mode`。
2. 打开菜单，**实际点开该衣服 / 道具**，截图（Idle）。
3. 用 **Gesture Manager Play 窗口自带的 Movement / 模拟移动**切 `Walk` / `Run`（也可 `Tools → Test Animation` 拖 `Walk` / `Run` → `play`）。
4. 移动态截图，和 Idle 对比：网格必须跟着骨骼动，不拉伸、不留在原地、不脱离身体。
5. 静态正常但移动漂移 = 绑骨失败；只看 Console / 静态截图不算通过。

### 8.4 本机样板：MameSamue → Kipfel（2026-10-19）

- outfit：`衣服/MameSamue_Mamehinata`，3 个 SMR（`Geta` / `Uwagi` / `Zubon`），27 根骨 × 3 = **81**。
- 去 `$GUID` 后按基础名映射：**81/81 matched，missing 0**。
- 菜单：`狼衣试验 / 和服`，参数 `WolfPilotMameSamue`；Gesture Manager 的 `RadialSliceControl` 点击后 **param=1.00**。
- 视觉：`WolfPilotShots/idle.png` 与 `move.png`（Hips + 双腿 + 手臂姿态）对比，衣摆/裤/鞋跟着骨骼变形，没有留在原地。**PASS（样板）**。
- **关键坑**：用 `module.PlayCustomAnimation(proxy_walk_forward)` 播 Walk 会**替换 Animator controller**，FX / MA 的菜单 toggle 全部失效，衣服会突然变回 Kipfel 默认装；移动态测试应使用 Gesture Manager 的 Movement，或直接摆骨骼姿态，不要替换 controller。
- 当前只做了 `MameSamue` 样板；其余 9 套 + 配件/道具待批量。

### 8.5 猫耳/头发穿模修复（2026-10-19 本机实测）

- 现象：后脑像一块光滑的“光头帽”，猫耳像从发帽里穿出来/漂浮。
- **真正根因不是猫耳**：AHH 发型的后发 `Back` 本身就是一块 scalp cap（原始 `Kipfel.unity` 默认也这样）；猫耳原本在发帽后面，只是被误判成“穿模”。
- **错误尝试**：
  - 直接改 `Cat_Ear` Transform / 把 `SMR.bones` 批量换成偏移骨 → 蒙皮 bindpose 破坏，耳朵消失。
  - 把 `Cat_Ear` mesh 绕 Head 抬起/缩小（A2：scale 0.88、offset (0,+0.04,-0.05)）→ 耳根和头发之间出现空隙，耳朵悬空，**更糟**。
- **正确做法**：隐藏 `Hair_AHH` / `Hair_Pyoko`，启用 `Hair_LooseShort`；`Cat_Ear` 保留原始 mesh，不要挪。`LooseShort` 后发是完整短发，猫耳自然藏在头发后面。
- 验证：`gallery_head_front.png` / `gallery_head_back.png` 后脑完整短发、猫耳在两侧发后、无悬空、无光头帽。
- 结论：**先判断“头发资产本身”是否完整，再动配件；发型后发是 cap 时，换发型，不要靠挪耳朵补偿。**


## 9. 来源

- Jinxxy 商品页（NewFit）：<https://jinxxy.com/KumekuMods/NewFit>
- **BOOTH 商品页（NewFit，2,500 JPY）：<https://booth.pm/zh-cn/items/8960065>**
- Reddit 发布贴（价格/用途讨论）：<https://www.reddit.com/r/VRchat/comments/1x7206g/>
- 兽人演示：<https://youtu.be/i9Mfai8CH0E>
- DressingTools 文档：<https://dressingtools.chocopoi.com/docs/>
- DressingTools Mapping Editor：<https://dressingtools.chocopoi.com/docs/guides/mapping-editor>
- AvatarDresser：<https://github.com/SophieBlueVR/AvatarDresser>
