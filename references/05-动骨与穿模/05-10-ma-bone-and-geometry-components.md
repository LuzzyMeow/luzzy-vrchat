# MA 骨骼与几何类组件全解（Merge Armature / Bone Proxy / Move To / Adjuster / World Fixed·Scale / Visible Head Accessory / PhysBone Blocker / Global Collider …）

> **适用**：合并骨架、把配件挂到骨上、调单骨缩放或整体高度、让头饰在第一人称可见、隔离 PhysBone、加全局碰撞体、改 contact tag、转换 constraint。
> **来源**：Modular Avatar 官方文档（<https://modular-avatar.nadena.dev/docs/intro> 起 67 页），2026-11-01 逐页抓取原文（MA 1.18.7，changelog 2026-09-01）；本地副本 `<VRCHAT_ROOT>\tools\tmp\ma-docs-txt\`。每节带官方 URL，可逐条核对。
> **定位**：本册是**官方语义参考**（字段 / 默认值 / 限制 / 烘焙期行为 / 组件间交互），与 `00-基底/00-02-modular-avatar.md` 的代码范式互补；基底是外部 MIT 原件，**一字未改**，本册不重复其已有内容。
> **标记**：【文档未说明，需实测】= 官方没写、须本工程验证；【需查源码】= 字段名未公开、需反射；【待复核】= 与 KB 现有表述冲突、需人工定稿；【KB 已有】= 只补语义。组件名 / 字段名保留英文原名。
> **本册来历**：2026-11-01 第 135 轮「MA 官方文档 × KB 缺口审计」的成品稿整册并入（审计报告在 `<VRCHAT_ROOT>\tools\tmp\ma-gap\`）。稿内「建议落地」一节保留为历史记录，落点已在并入时确定。

## 目录

1 Merge Armature ｜ 2 Bone Proxy ｜ 3 Move To ｜ 4 Move Independently ｜ 5 Replace Object ｜ 6 Scale Adjuster ｜ 7 Floor Adjuster ｜ 8 World Fixed Object ｜ 9 World Scale Object ｜ 10 Visible Head Accessory ｜ 11 Remove Vertex Color ｜ 12 Mesh Settings ｜ 13 Outfit Root ｜ 14 PhysBone Blocker ｜ 15 Global Collider ｜ 16 Rename VRChat Collision Tags ｜ 17 Convert Constraints ｜ 18 决策速查表 ｜ 19 建议落地 ｜ 20 待实测

---

## 1. Merge Armature（重点）

**出处**：https://modular-avatar.nadena.dev/docs/reference/merge-armature ｜ https://modular-avatar.nadena.dev/docs/distributing-prefabs/for-outfit-creators

**定位**：把一个 GameObject 子树**合并进基础头像骨架**，专为**蒙皮衣服 prefab** 设计，内含「更新既有 SkinnedMeshRenderer 引用 + 尽量少生成骨」的特殊逻辑。该用：要跟随头像骨架的蒙皮网格（衣服/鞋/发型）。不该用：**分发给不同头像的通用 prefab**——官方举例「手指笔 prefab 不合适」，因为组件**假设被绑定的骨不会移动**，无法泛化。〔KB 已有〕`05-04 §3.1` 只讲了「骨名对得上+尺寸接近」，缺的正是这条设计边界：它不是适配工具。

**生效面**：构建期合并；编辑期靠位置锁定模拟跟随。

### 1.1 字段表

| 字段 | 取值 | 默认 | 语义 |
|---|---|---|---|
| `Merge Target` | Transform | 空 | 合并目标；**内部存的是路径引用**（见 1.8） |
| `Prefix` / `Suffix` | 字符串 | 通常自动填 | 比对前从合并骨名**剥离**，兼容带前后缀的衣服 |
| `Position Lock Mode` | 不锁 / `Base =======> Target`（单向）/ `Base <======> Target`（双向） | Setup Outfit 建**单向** | 编辑模式衣服是否跟随头像 |
| `Avoid name collisions` | 勾选 | **开** | 本资产新增骨默认**改名**避免与其它 Merge Armature 资产撞名 |

> `LockMode=BaseToMerge`、`mangleNames` 等 KB 里出现过的序列化值与上表的对应关系 **【需查源码】**，别凭字面猜（`mangleNames`"改名"与 `Avoid name collisions`"避免撞名"可能语义相反）。

### 1.2 匹配与合并三步

1. **遍历源层级按名字找骨**：从组件所在 GO 往下走，在头像里按名匹配；Prefix/Suffix 比对前剥离。
2. **找到 → 改写引用**：把「指向合并骨」的引用改指头像对应骨；**改不了时退化为新建子骨**。
3. **没找到 → 新建子骨**：挂到对应父骨下。

配套规则（分发指南）：**尽量不生成重复骨**；**含非 Transform 组件的骨不合并**，而是整体**挪成头像对应骨的子级**；**头像本来没有的骨会被保留**（衣服独有上胸骨、饰品骨）；骨名后的附加文本同时承担「多套衣服防撞名」作用。

### 1.3 烘焙期「按属性类型分流改写」（MA 最隐蔽的行为）

官方称会尽力让源层级上配置的、或指向源层级的组件「Just Work」，共四类改写：

| 被改写的对象 | 改成指向 | 关键细节 |
|---|---|---|
| **Animator 引用** | **看动画在动什么属性**：`transform` 类曲线 → **合并后骨**；`GameObject active`（`m_IsActive`）曲线 → **源层级** | 同一条 clip 里不同曲线会被拆到不同目标 |
| **PhysBone / Contact 的 target** | 合并后的新骨 | **即使该 PhysBone 不在 Merge Armature 之下也会被改** |
| **其它组件** | 仍留在源层级 | 但 MA 会**生成 ParentConstraint** 跟踪合并后层级 |
| **源层级本体** | 含非 Transform 组件的部分**保留**，其余**通常删除** | 解释「骨被合并了但挂 PhysBone 的那截还在」 |


### 1.4 位置锁定三档（编辑态调 fit 必用）

| 档位 | 编辑模式行为 | 适用 |
|---|---|---|
| 不锁 `Not locked` | 衣服**不跟随**头像 | 不关心编辑态预览 |
| `Base =======> Target`（单向） | 头像动→衣服动；**动衣服不带动头像**；**保留你调好的 fit** | **日常推荐**；Setup Outfit 默认设它 |
| `Base <======> Target`（双向） | 双向联动 | 进阶：做「用 prefab 动画化头像头发/兽耳」的 prefab |

用途：编辑模式测动作、摆姿势、截图时衣服能跟着走。〔KB 现状〕`02-03:215` 只有清单项、`05-04:41`/`05-08:1047` 只有字段值。

### 1.5 Reset position to base avatar（跨作者套装第一轮粗调）

点 `Do It!`：**把衣服所有骨的位置设为头像对应骨的位置**。三个附加选项：

| 选项 | 语义 | 何时用 |
|---|---|---|
| `Also set rotation` | 同时对齐 localRotation | **特殊场合**：衣服与头像非同一 3D 软件所做时会有怪异结果 |
| `Also set local scale` | 同时对齐 localScale | 你**改过头像骨缩放**、需要衣服跟上 |
| `Adjust outfit overall scale to match base avatar` | **用「臂长」判整体尺寸**，先将整套衣服整体缩放，**再**逐骨对位 | **官方推荐**，一般就用它 |

- **选项不保存**：点一次只做一次性调整。
- 用 Setup Outfit 配过的衣服，**同样的工具在 Outfit Root 组件 Inspector 上也能用**（见 13）。

> 与 KB 关系：`05-08` 跨尺寸套装是手工相似变换（如 `s=0.66830`）、Kabsch/×0.53。**第一轮粗调可直接用此工具**，精调再走 `ProjectOut`/`WallFitTune`。

### 1.6 Match Scale Adjusters to base avatar

对每根匹配骨**把头像上的 Scale Adjuster 配置复制到衣服**：头像有 → 衣服**添加/更新**；头像没有 → 衣服上多余的**删除**。基础头像靠 Scale Adjuster 改了骨尺寸时，衣服必须镜像才能贴合（配 §6）。

### 1.7 骨名对齐与撞名规避

| 操作 | 行为 |
|---|---|
| `Adjust bone names to match target` | 把衣服骨**改名**去匹配头像；**用 Setup Outfit 添加时会自动执行** |
| `Avoid name collisions`（默认开） | 本资产**新增的**、头像没有的骨默认**改名**，避免与「同样用 Merge Armature 又取了同名骨」的资产冲突；特殊情况下可取消勾选 |

注意：光挂组件**不一定**能让 A 头像的衣服在 B 上工作——**匹配是按名字走的**。

### 1.8 对象引用存路径

编辑器里可拖对象进 `Merge Target`，但**内部保存为路径引用**，因此存进 prefab 后能**自动恢复 Merge Target**。〔KB 已有〕`05-04 §4` 讲过 `AvatarObjectReference` 路径脆弱，缺的是「MergeTarget 也走路径引用」——所以「prefab 存回后 Merge Target 还在」是正常现象；真丢了查路径是否被改名/挪层级破坏。

### 1.9 嵌套合并（1.7.0+）

自 **MA 1.7.0** 起支持嵌套：A 合进 B、B 再合进 C，**MA 自动决定正确顺序**。〔KB 现状〕`02-03:41` 只有清单项「是否已存在嵌套组件」，缺版本与自动排序。

### 1.10 「整套装开关」动画会被改写（实战价值最高）

Merge Armature 除了改路径，还会〔for-outfit-creators〕：① **调整动画路径**对应被移动的骨；② **把「开关整套衣服」的动画，重写为「开关这套衣服散落各处之后的全部对象」**。

**推论**：用**任意动画工具**开关「衣服根对象」，烘焙后会自动同时开关**衣服网格 + 该套衣服的 PhysBone 组件**。

> 对 KB 的意义：现有「一键穿衣」是 `MenuItem`+`ObjectToggle`+FX 层显式写 `m_IsActive`。**Merge Armature + 父级动画开关**是官方支持的另一条路，且**连带 PhysBone 一起开关**——OT 做不到（PB 在别的骨上就得单独处理）。

### 1.11 与 Bone Proxy 的边界

| 维度 | Merge Armature | Bone Proxy |
|---|---|---|
| 粒度 | 合并**整棵骨树** | 移动**单个对象**，**完全不合并骨** |
| 人形骨依赖 | **按骨名匹配** → 绑死骨名 | 目标若为 humanoid 骨，引用**不依赖骨名** → 可做通用机关 |
| 位姿 | 由位置锁定模式决定 | 可选位置/旋转处理方式（如无论手在哪都吸附） |
| 开关动画 | **会**改写「开关整套」 | **不会**：只修正路径，**开关 BP 父级的动画不影响 BP 对象及子级** |

单骨配件（发夹挂 Head）用 Bone Proxy 更省事、更好复用。

### 1.12 Setup Outfit 对衣服做了什么（Merge Armature 视角）

1. **识别骨**：找形如 `OutfitRoot → [Armature] → Hips` 的骨，**Hips 必须包含头像 hips 骨名**；骨名前后允许附加文本但**必须全套统一**；`Armature` 那层叫什么都可以。
2. **兜底匹配**：不符合上述规则时拿「常见骨名表」猜、猜中则改名；官方明说这条路**不可靠、不建议依赖**。
3. **A/T-pose 转换**：衣服与头像姿态不同时**旋转手臂**对齐（条件见 1.13）。
4. **建 Merge Armature**：建在 armature 对象上，写入识别出的 prefix/suffix 与头像骨架引用；**衣服有头像没有的上胸骨时，用多个 Merge Armature 组件分别处理**。
5. **建 Outfit Root**（见 13）、6. **建 Mesh Settings**（默认灯探针锚点与头像一致 + 合理包围盒，作用于衣服全部网格，见 12）。

合并后现象：**大部分骨被合并**、**带 PhysBone 的骨留下**、**头像没有的骨保留**、**骨名附加文本避免多套衣服撞名**。

### 1.13 能容忍 / 必须避免

**能容忍**：① **Prefix/Suffix** 全套一致即可（**大小写敏感**；无该前后缀的骨不合并）；② **不同骨轴**（不同 3D 软件 X/Y/Z 朝向不同）**自动处理**，只要**位置对上**；③ **Armature 对象名**任意；④ **A/T-pose 不同**——但**只有臂长与头像完全一致**才转换，差一点就**跳过**，且旋转方向由**骨位置差**推断，位置略偏可能**转到错方向**。

**必须避免**：① **人形骨上挂 PhysBone**（hips/chest 等）——很多衣服会误带「从头像复制来的 PB」，MA 会**尝试删除**导致行为不可预测；正解是 **PB 只放在头像没有的 GO 上，或放在 Merge Armature 所在 GO 之外**。② **骨名与 Armature 名与头像完全同名** → 触发 Unity 怪 bug；Setup Outfit 的规避是给 Armature 名**加 `.1`**，更好是自己改名。

**Preset 语义**：已 Setup Outfit 过的 prefab 分发出去后，用户**再跑一次会检测到已配置并跳过**。

### 1.14 坑

① 拿它做通用 prefab → 不能泛化，通用件用 Bone Proxy。② 只挂组件不调骨名 → 直接不合并。③ 烘焙后「衣服开关少了一半」→ 见 1.10。④ 衣服上 PB 消失/行为怪 → 见 1.13。⑤ **烘焙后骨数变多**是正常的（含非 Transform 的骨不合并 + 独有骨保留 + 改不了引用时新建子骨）。⑥ 编辑态衣服不跟手 → 位置锁定是「不锁」。⑦ 跨尺寸套装死算缩放 → 先用 `Adjust outfit overall scale`。

---

## 2. Bone Proxy

**出处**：https://modular-avatar.nadena.dev/docs/reference/bone-proxy ｜ 差异对比 https://modular-avatar.nadena.dev/docs/distributing-prefabs/for-outfit-creators ｜ Blocker https://modular-avatar.nadena.dev/docs/reference/physbone-blocker

**定位**：把 prefab 里的对象**放进头像已有的对象内部**（官方例：Clap 样例把 contacts 放进手里）。**不要用它配衣服**——那是 Merge Armature 的活。**生效面**：构建期重父 + 改动画路径。

### 2.1 字段

| 字段 | 取值 | 默认 | 语义 |
|---|---|---|---|
| `Target` | Transform | 空 | 目标对象/骨；**设值时自动选 attachment mode**（见 2.3） |
| `Attachment Mode` | `As child at root` / `As child keep world pose` / 只保位置 / 只保旋转 | 设 Target 时自动判定 | 见 2.2 |
| `Match Parent Scale` | 勾选 | **关** | 开 = 移动后 localScale 调成 **(1,1,1)**，补偿目标骨父级缩放 |
| `Advanced` 折叠区 | 可改 | — | 内部保存的 **humanoid 骨 + 相对路径**引用 |

### 2.2 Attachment Mode 四态

| 模式 | 行为 | 适用 |
|---|---|---|
| `As child at root` | 重父到目标，**localPosition 与 localRotation 归零** → 与目标同位姿 | **非头像专用 prefab 推荐** |
| `As child keep world pose` | 重父，**保留世界位姿** | 一般只对**头像专用 prefab**有用；例：给布料摆 collider 到精确相对位置 |
| 只保位置 | 位置保持、**旋转跟随目标骨** | 进阶 |
| 只保旋转 | **位置跟随目标骨**、旋转保持 | 进阶 |

### 2.3 自动选模式

**给 BP 设 Target 时，attachment mode 会按「该对象当前是否正处于目标骨的位姿」自动设定。** → 坑：你以为没动它，它自己变了；想固定就显式再选一次。

### 2.4 Match Parent Scale（KB 只知名字）

- **关（默认）= 经典行为**：移到新目标骨后**保留原世界缩放**。
- **开**：移动后该骨 **localScale 被调成 (1,1,1)**。用于「目标骨父级有缩放，但我需要配件保持相同相对尺寸」。
- 〔对应症状〕KB 的 `SKILL.md`「常用入口」与 `08-症状速查/` 记的是「chibi 工具飘头顶」（`AsChildKeepWorldPose` 取世界坐标）；**尺寸不对**这一半是 Match Parent Scale 的锅，开关各试一次最快。（**按节名 / 症状关键词定位，不引行号**）

### 2.5 烘焙期

① 重父（按模式处理位置/旋转/缩放）；② **改写所有引用旧位置的 animator，指向移动后的新路径**；③ **不改写「开关父级」的动画**——开关 BP 父级**不会**连带影响 BP 对象及子级。

### 2.6 引用与交互

自动把目标翻译成 **humanoid 骨 + 相对路径**引用，保证存进 prefab 后能恢复（这正是它能做通用机关的原因）；内部引用看 `Advanced`。
- **+ PhysBone Blocker**：挂到别人已有 PB 链（尾巴/耳）上时，**同一对象**上再放 Blocker 才能**刚性附着**（见 14）。
- **+ Move To**：BP 是「跟着目标动」，Move To 是「只对齐一次、之后不跟」（见 3）。
- 与 Merge Armature 差异见 1.11。

### 2.7 坑

① 用它配衣服 → 不合并骨、衣服跟不动。② `AsChildKeepWorldPose` 用于 chibi 头像上的通用工具 → 取 prefab 世界坐标，工具飘头顶。③ 忘开 `Match Parent Scale` → 尺寸不对。④ 以为关父级能关掉 BP 对象 → 不会。⑤ 挂在 PB 链上却没加 Blocker → 配件跟着链甩。

---

## 3. Move To

**出处**：https://modular-avatar.nadena.dev/docs/reference/move-to

**定位**：让对象**匹配另一个对象的 transform**——**初始**移动过去，**运行时不再继续跟踪**。**生效面**：编辑期持续更新；**构建期应用一次后把自己删掉**（产物里没有该组件）。

### 3.1 字段

| 字段 | 取值 | 语义 |
|---|---|---|
| `Target` | Transform | 要匹配的对象 |
| `Match position` | 勾选 | 对齐**世界位置** |
| `Match rotation` | 勾选 | 对齐**世界旋转** |
| `Match scale` | 勾选 | 对齐缩放，**父级缩放不同时也对** |

默认值 **【待实测】**（文档未写）。

### 3.2 时机（执行顺序第一手证据）

编辑期**持续更新**；构建期**在 Bone Proxy 处理之后应用一次**，然后**从构建产物里移除自身**。

> 比 KB 现有「先查版本与执行顺序」的经验说法精确得多，可直接写进 `bake-and-validation` 的顺序表。

### 3.3 适用与坑

适用：需要「初始就在某处」但**不该一路跟着**的对象（世界固定道具定位、约束目标摆位）。不适用：要持续跟随 → Bone Proxy / ParentConstraint。
坑：① 想要跟手却用了 Move To → 烘焙后不跟（设计如此）。② 与 BP 同用时预期搞反：**Move To 在其后执行**，对齐的是 BP 处理后的状态。③ `Match scale` 与 BP 的 `Match Parent Scale` 是两件事：前者**烘焙一次性对齐**，后者**是移动时的缩放补偿**。

---

## 4. Move Independently

**出处**：https://modular-avatar.nadena.dev/docs/reference/move-independently

**定位**：**移动一个对象而不影响子对象**；**运行时完全无效，纯编辑器工具**（不进构建产物）。

| 字段 | 取值 | 语义 |
|---|---|---|
| `Objects to move together` | 一组勾选框 | 勾上的对象组成「一起移动」的组 |

**用途**：调衣服 fit，例如**只挪衣服的 hips 对象**而不带动其它件；分组例：**hips 与 upper leg 一起动、lower leg 留在原地**。

**限制**：支持独立缩放，但**非均匀缩放（X/Y/Z 不全相同）不完全支持**、可能出怪行为；需**逐轴**缩放时**配 Scale Adjuster 一起用**。

〔与 KB 关系〕`05-08` 现在调 fit 走「算偏移量批量改 TRS / ProjectOut / Kabsch」，**第一轮试摆**用它更直观。
坑：① 以为运行时有效。② 直接上非均匀缩放 → 子件歪。

---

## 5. Replace Object

**出处**：https://modular-avatar.nadena.dev/docs/reference/replace-object

**定位**：**完全替换**头像上某个 GameObject 的**内容**（官方例：替换头像的 **PhysBone 配置**，或**把身体网格整体换掉**）。**生效面**：构建期，**执行得相当晚**。

| 字段 | 语义 |
|---|---|
| `Replace Target` | 头像上要被替换的对象 |
| `Replacement` | 你的 prefab 里用来替换的对象 |

【待实测】序列化字段名。

### 5.1 硬限制

**一个对象只能被一个对象替换** → 用了它就**降低本资产与其它同样想替换该对象的资产的兼容性**。〔判据价值〕分发/买商业资产时，这是「两个包能否同时用」的关键；KB 现有排查清单没有这条。

### 5.2 四条细节机制

| 行为 | 说明 |
|---|---|
| **子对象** | 只替换「指定的那个对象」；**原对象与替换对象的子对象都挂到替换对象下**（不是单方覆盖） |
| **命名** | **不改**替换对象的名字；名字不同则最终名不同。但会**更新引用原对象的动画路径**指向替换对象 |
| **时机** | 执行**很晚**，多数情况无所谓；但**替换 Body 网格又要保 MMD 世界兼容（或反之）会出问题** |
| **组件引用** | 把「指向旧对象组件」的引用改指新对象；**同类型多个时按索引匹配**（新对象同类不够就**置 null**）；**不做模糊匹配**（BoxCollider 换 SphereCollider → 旧引用变 null） |

### 5.3 交互与坑

交互：**替换 PhysBone 配置**（与 §14/§15 同属物理层改造）、**替换 Body 网格**（与 AAO 网格合并/形态键改名、MMD 兼容相关）；「组件引用置 null」是烘焙后「某字段变空」的常见根因。
坑：① 两个包都想换 Body → 只有一个生效/互相破坏。② 换类型后引用全空。③ 替换后动画路径变了→其实是**有意改写**。④ 想让名字也变→不会。

---

## 6. Scale Adjuster

**出处**：https://modular-avatar.nadena.dev/docs/reference/scale-adjuster ｜ 配合见 https://modular-avatar.nadena.dev/docs/reference/merge-armature

**定位**：调**某一根骨**的 X/Y/Z 缩放，而**不让有旋转的子骨出问题**（Unity 直接缩放父骨会把带旋转的子骨拉歪）。**生效面**：以编辑器工具为主，烘焙时把结果固化。

| 项 | 取值 | 语义 |
|---|---|---|
| 组件本身 | 无配置字段 | 加在目标骨上；**选中 Unity 的 scale 工具后，改动只影响这一根骨** |
| `Adjust child positions` | 勾选 | **编辑器全局设置**（见 6.1），不是组件字段 |

### 6.1 `Adjust child positions` 是编辑器级全局设置（最易误判）

① 控制「父骨缩放时**是否同步调整子骨的相对位置**」；② **这是编辑器级（editor-wide）设置，作用于所有 Scale Adjuster，不是单个组件的字段**；③ **Unity 重启后会重置**；④ 它**只调整子骨的「位置」，不调整子骨「缩放」**。
→ KB 若写「每个组件勾它来控制子骨」是**错的**。

### 6.2 场景与限制

- **该用**：给「不是给你头像做的衣服」调局部尺寸，调特定骨而不破坏子骨。
- **不该用**：**等比整体缩放**（X/Y/Z 同时等比例）时，**用 Unity 普通缩放工具更好**。
- 多骨：给**每根骨都加组件**再**多选**一起缩放。
- 限制：**骨头本身有旋转时缩放不完美**。
- ⚠️ **官方 warning**：它**只接管 Unity 的 scale 工具**；**组合的 Move/Rotate/Scale 工具仍会影响所有子级**。

### 6.3 与 Merge Armature 配合

`Match Scale Adjusters to base avatar`（1.6）把头像的 Scale Adjuster 镜像到衣服上 → 头像用 SA 改了骨尺寸时，衣服必须做这一步才贴合。
坑：① 把它当组件字段（6.1）。② 用组合工具缩放 → 子级全动。③ 用它做等比缩放 → 不如普通缩放。④ 骨头带旋转硬调 → 不完美。⑤ 衣服没配 Match Scale Adjusters → 与头像不同步。

---

## 7. Floor Adjuster

**出处**：https://modular-avatar.nadena.dev/docs/reference/floor-adjuster

**定位**：调**头像整体垂直位置**，让「地面（= avatar root 位置）」对上**鞋底**（加鞋后脚被地面挡住/陷进去时用）。**生效面**：构建期。**无配置字段**——位置由**组件所在 GO 的垂直位置**决定。

**用法**：① 新建空 GameObject + 组件；② 把它的**垂直位置**调到与**鞋底**对齐；③ 官方 tip：**Scene 视图切侧视/等距（side-on, isometric）**最准。

### 7.1 硬限制（必写）

- **VRChat 目前不支持动态调整头像身高** → **多套衣服需要不同地面高度时做不到**（整套头像只能定一个高度）。
- ⚠️ **官方 warning**：**场景里存在多个 Floor Adjuster 时，完全不做任何调整（no adjustment will be made）** → **不报错、不警告，只是没效果**。这是「明明加了却不生效」的头号原因。
- 官方注：若将来 VRChat 支持动态调身高，该行为可能改变 → 版本升级时留意。

〔KB 现状〕只在 `02-03:25` 决策表与 `07-05:126/130` 版本行出现名字。
坑：① 多套装想各调地面→不可能。② 场景残留第二个→全部失效。③ 正视图调→对不准。

---

## 8. World Fixed Object

**出处**：https://modular-avatar.nadena.dev/docs/reference/world-fixed-object

**定位**：让 GameObject **相对世界固定**，即使头像移动也不动（道具/特效留在地上）。**生效面**：构建期生成；运行时靠生成的 constraint。**无任何配置项**。

**烘焙期**：① 在 **avatar root 下自动生成一个「固定在世界的原点」的 GameObject**；② 把你的 GO **移成它的子级**；③ 内部对象位置**仍可用 ParentConstraint 等控制**。

**性能（可放心批量用）**：**无论 1 个还是几十个，只生成一个 constraint**，开销与数量无关。

**交互**：+ Move To（WFO 只把父级钉在世界原点，**初始定位**靠 Move To 或手摆）；+ World Scale Object（一个管「位置不跟」、一个管「缩放不跟」，世界尺寸类机关常成对出现）。
坑：① 以为要手动建父级→自动生成。② 以为多个组件开销叠加→只有一个 constraint。③ 生成后对象位置「跳了」→父级换了，需按世界原点重新定位。

---

## 9. World Scale Object

**出处**：https://modular-avatar.nadena.dev/docs/reference/world-scale-object

**定位**：强制 GameObject **与世界同缩放**、不受当前头像缩放影响；实现 = 构建期挂一个 **(VRC) Scale Constraint** 并设成「相对世界 (1,1,1)」。**无配置项**。

### 9.1 关键限制：编辑器不预览

⚠️ **World Scale Object 目前在 Unity 编辑器里不被预览**，只在**游戏内 / Play 模式**正确生效。
→ 必须进 KB 验证清单：**编辑态看着尺寸不对不是 bug**，与「烘焙后参数全是 Float、Play 初值取控制器默认」同属验证纪律。
用途：让对象跟世界缩放而非跟头像缩放，用于**复杂 constraint 机关**。
坑：① 编辑态判断对错。② 与 WFO 混用顺序搞反（一个管位置一个管缩放，互不替代）。

---

## 10. Visible Head Accessory

**出处**：https://modular-avatar.nadena.dev/docs/reference/visible-head-accessory

**定位**：让**挂在 Head 骨下的对象在第一人称视角可见**（不用照镜子看自己的头发/头饰）。**生效面**：构建期改网格 + 走 VRCHeadChop。**无配置项**，加在 **Head 骨下的某个子对象**上即可。

### 10.1 两条限制

① **不能作为 PhysBone 链的子级**（可以加在**链的父级**上）→ 动态头发（PB 链）上挂会失效/异常。② **给 Head 的「所有」子级都加会持续碍眼**——刘海一直挡视线，要挑着加。

### 10.2 原理（KB 完全没有）

- 在 VRChat 上它内部用 **VRCHeadChop** 让选中骨可见。
- **与直接用 VRCHeadChop 的关键区别**：它会**调整网格**，确保三角形**不会穿过玩家视点**。
- 具体做法：① 找出「**部分顶点权重在可见骨、部分权重在隐藏骨（如 Head 根骨）**」的三角形；② **新增 proxy 骨**把这些顶点**权重切过去**；③ 从而保证该三角形**完全可见**。

> **验收推论（务必写进 KB）**：用了它之后 **烘焙后骨数会增加**（多出 proxy 骨）且**网格被改**。做「transforms / 骨骼数」断言时要把这部分算进去，否则误报。

**交互**：VRCHeadChop（VRCSDK，实现手段）——KB 现有 `02-02:66,83` 只把它当「系统节点永不移动」；AAO（网格被改 → 与 AAO 合并/改名流程相互作用）**【待实测】**顺序。
坑：① 挂 PB 链子级→无效。② 全 Head 子级都挂→挡视线。③ 骨数断言对不上→proxy 骨。

---

## 11. Remove Vertex Color

**出处**：https://modular-avatar.nadena.dev/docs/reference/remove-vertex-color

**定位**：**非破坏地**移除自身及**所有子级**的顶点色。**生效面**：构建期（只改烘焙产物，源资产不动）。

| 模式 | 语义 |
|---|---|
| 默认（移除） | 本对象及子级的顶点色全部移除 |
| `Keep Vertex Colors` | 加在要**排除**的对象上：**该对象及其下不再被移除** |

### 11.1 触发场景（KB 缺的因果链）

有些模型**自带顶点色、但那些顶点色本来不是用来显示的**；一旦换成**会用顶点色的 shader**（官方举例 **VRChat mobile shaders**）就**出现不该有的变色**（官方配图：头发被染色）。用它可**不破坏源资产**地清掉。

> 对 KB 的意义：lilToon/材质章节讲了大量「换材质、换贴图、换 shader」，但**没有一条讲「换 shader 后颜色异常可能是顶点色污染」**——很典型的查不出原因的偏色。

**用法**：通常**直接加在头像根**，其下全部移除；排除个别对象就给该对象**也加一个**组件、模式设 `Keep Vertex Colors`。
坑：① 换 shader 后头发/皮肤偏色→先查顶点色。② 想保留某子的顶点色→加排除组件，而不是删父级。③ 以为会改源 FBX→不会。

---

## 12. Mesh Settings

**出处**：https://modular-avatar.nadena.dev/docs/reference/mesh-settings ｜ 自动配置见 https://modular-avatar.nadena.dev/docs/distributing-prefabs/for-outfit-creators

**定位**：批量设置某 GameObject 下**所有 mesh 的 Anchor Override + Bounds**。**生效面**：构建期。

| 字段 | 取值 | 默认 | 语义 |
|---|---|---|---|
| `Anchor Override Mode` | `Inherit`/`Set`/`Don't set`/`Set or inherit` | **初始=什么都不做** | 灯探针锚点覆盖模式 |
| `Bounds Override Mode` | 同上四档 | **初始=什么都不做** | 包围盒覆盖模式 |
| `Root Bone` | Transform | 空 | Bounds **相对这根骨**计算 |

### 12.1 四档模式（KB 只有两行）

| 模式 | 语义 |
|---|---|
| `Inherit` | **本组件对这一项不做任何事**，继承父级 Mesh Settings 的值 |
| `Set` | 对本 GO 及**所有子级**的 mesh 设置该项 |
| `Don't set` | **阻止任何父级 Mesh Settings 生效**；mesh 保持**默认设置** |
| `Set or inherit` | **父级有 Set 就用父级**，没有父级用得上时才用自己 → **衣服 prefab 专用**，保证「头像全局设置优先」 |

### 12.2 关键细节（KB 缺失）

- **加了组件后初始什么都不做**，必须改两个 Mode 才生效 → 「加了没反应」的直接解释。
- **Bounds 只影响 SkinnedMeshRenderer**；**Anchor Override 还影响 MeshRenderer / LineRenderer 等**。
- **Bounds 相对 `Root Bone` 那根 transform 计算**。

### 12.3 三种用法与分发注意

① **放头像顶层** → 保证**全头像** bounds 与灯探针锚点一致。② **Setup Outfit 会自动给新加衣服配一个**（默认灯探针锚点与头像一致 + 合理包围盒）。③ **排除**下级 mesh 使其不受上层影响。
分发注意：给要分发出去的资产设 bounds/灯探针**要谨慎**（可能与它被应用到的头像不一致），**一般只对「为特定头像设计」的资产设置**。
坑：① 加了没反应→模式还是初始值。② 设了 bounds 但 MeshRenderer 没变→只影响 SMR。③ 通用资产设 bounds→换头像不一致。④ 衣服被头像全局设置覆盖/反过来→用 `Set or inherit`。

---

## 13. Outfit Root

**出处**：https://modular-avatar.nadena.dev/docs/reference/outfit-root ｜ 工具行为同 https://modular-avatar.nadena.dev/docs/reference/merge-armature

**定位**：**标识某个 GameObject 是「一套衣服的根」**。**生效面**：**组件本身什么都不做**——它是「便捷入口 + 给其它工具识别的标记」。

| 字段 | 语义 |
|---|---|
| `Armature Root` | 衣服**主骨层级的根**；手动加时必填 |

### 13.1 通常不需要手加

- **Setup Outfit 会自动加并填好 `Armature Root`**。
- **对旧版 MA 配过的衣服再跑一次 Setup Outfit，也会补加 Outfit Root**。
- 手动加时把 `Armature Root` 指到衣服主骨层级的根 Transform。

### 13.2 Inspector 上三个调衣服工具（KB 完全没有）

| 工具 | 行为 |
|---|---|
| `Adjust bone names to match target` | 把衣服骨改名去匹配头像 |
| `Reset position to base avatar` | 对 `Armature Root` 下**每个 Merge Armature**，把对应骨移到头像骨位置；**其选项还能做 A-pose ↔ T-pose 转换、调整整体缩放、复制骨的旋转或局部缩放** |
| `Match Scale Adjusters to base avatar` | 在匹配骨**添加/更新/删除** Scale Adjuster 以镜像头像 |

> **A↔T pose 转换**是高价值功能：KB 现在遇到「衣服 A-pose、头像 T-pose」只能手工掰；但转换有条件（臂长必须完全一致，见 1.13）。

### 13.3 ⚠️ 术语消歧（写进 KB 前必读）

KB 里的 **`OutfitRoots`** 是**工作区手写的 FX 层**（曾用于「一键穿衣」写 `m_IsActive`，已废弃；见 `SKILL.md` 铁律 1 与「使用前提」、`03-菜单与参数/03-06-ma-menu-build-by-reflection.md` §14、`07-验证与排查/07-01-bake-and-validation.md` 相关小节——**按节名检索，不引行号**），**与官方 `ModularAvatarOutfitRoot` 无关**，新章节必须显式消歧。另 KB `02-02:211,230` 有一条**正确的实战警告**可保留：**普通配件不要用 OutfitRoot**——组件不做事，但会给工具「独立套装」信号，可能导致所在套装容器构建时激活失败（KB 实测，非官方文档）。
坑：① 手动加却没填 `Armature Root`→工具不出现/不工作。② 当运行时开关用→组件不做任何事。③ 给普通配件加→被标记成独立套装。④ 与 KB 废弃的 `OutfitRoots` FX 层混淆。

---

## 14. PhysBone Blocker

**出处**：https://modular-avatar.nadena.dev/docs/reference/physbone-blocker

**定位**：**阻止「父对象上的 PhysBone」影响某个子对象**。**生效面**：构建期（把子对象塞进相关 PB 的 Ignore 列表）。**机制** = **把这个子对象加到「任何影响其父对象的 PhysBone」的 Ignore 列表里**。**无配置项**。

**典型场景**：做**可能被别人挂到尾巴/兽耳这类 PB 链上的配件**时用它让配件**刚性附着**、不被父链甩动。
⚠️ **加了 Blocker 之后，该子对象上仍可挂自己的 PhysBone**（不是禁掉物理）——最容易被误解的一条。

### 14.1 与 Bone Proxy 的配合（关键）

用 BP 把对象挂到**别人已有的 PB 链**上时：加 Blocker 才能**保证刚性附着**；**最好把 Blocker 和 BP 放在同一个对象上**。
〔KB 衔接〕`05-09` 讲透了「用 BP 把第三方工具/配件挂到骨上」，但**完全没有「挂上去之后被父链甩动怎么办」**——本节补这半边。
坑：① 挂 PB 链上不加 Blocker→跟着甩。② 以为加了就不能挂自己的 PB。③ Blocker 与 BP 不在同一对象。

---

## 15. Global Collider

**出处**：https://modular-avatar.nadena.dev/docs/reference/global-collider

**定位**：在**自己头像上**定义「能与其他头像的动力学（PhysBone）全局交互」的 collider。**生效面**：构建期（接管/生成 collider）。

**三种用途**：① 给小道具/机关加 collider，去**碰别人的 PB**；② **移动 VRChat 的 Hand collider**，换「抓取发生的位置」（官方例：挪到嘴部模拟**咬**）；③ 给**武器**做冲击波/后坐——**用动画驱动一个带 collider 的 GameObject**。

### 15.1 硬限制

- 全局 collider 在**某些平台数量有限**，另一些平台**有性能惩罚** → **要省着用**。
- **在 VRChat 上，用超过 6 个全局 collider 会覆盖掉你的 index finger collider（食指碰撞体）**。

### 15.2 字段表

| 字段 | 取值 | 语义 |
|---|---|---|
| `Radius` / `Height` / `Position` / `Rotation` | 胶囊参数 | 半径 / **沿 Y 轴**高度 / 局部位置偏移 / 局部旋转 |
| `Root Transform` | Transform | 默认 = 组件所在 GO；**可覆盖**成别的 transform，让 collider **跟随头像的另一个部位** |
| `Copy Hijacked Shape` | 勾选 | **仅 Manual Remap 开启时可用**：radius/height/position/rotation **从被劫持的 VRChat descriptor collider 复制**，不用本组件里配的值 |
| `Visualize Gizmo` | 勾选 | Scene 视图是否画 collider gizmo |
| `Manual Remap` | 勾选 | **手动指定**劫持哪个 collider（见 15.3） |
| `Low Priority Collider` | 勾选 | **仅 Manual Remap 时**：标记为「可替换」（见 15.3） |

### 15.3 实现策略 / Manual Remap / Low Priority

- 全局 collider 是「能与其他头像交互」的 collider；**VRChat 上通常只有内置某些 collider（如手指）能做到**，因此本组件会**「接管/劫持（hijack）」一个基础 collider（手指）**来实现；其他平台实现方式可能不同。
- **Manual Remap**：开启后可手动定义「劫持哪个 collider」。⚠️ **不是所有列出的 collider 都能与别人 PB 产生物理碰撞**——**Head、Torso、Feet 只是 contact sender（接触发送器）**，选错 = 白做。
- **Low Priority Collider**：标记为可替换；**若另一个 Global Collider 重映射了同一个 collider，低优先级的那个会被无警告覆盖**。

**交互**：+ World Fixed / World Scale（世界尺寸类机关常需要世界固定的 collider）；+ Replace Object（替换 PB 配置时可能牵动 collider 归属）；VRCSDK（劫持的是 AvatarDescriptor 上的 collider，**内置 collider 列表可能随 SDK 版本变化【待实测】**）。
坑：① 用到第 7 个→食指 collider 被覆盖，抓取手感全变。② Manual Remap 选 Head/Torso/Feet→不做物理碰撞。③ 忘设 `Root Transform`→collider 不跟手/头。④ 两个组件抢同一个 collider，低优先级被静默覆盖。

---

## 16. Rename VRChat Collision Tags

**出处**：https://modular-avatar.nadena.dev/docs/reference/rename-vrchat-collision-tags

**定位**：**重命名 VRChat Contacts 系统使用的 collision tags**，避免与其它组件/机关撞名。**生效面**：构建期。

**三种用途**：① 你的 contacts **不要干扰别人的头像**；② 避免与自己头像上其它组件/机关撞名；③ **反过来**让多个机关**用同一个 tag 名**从而**互相交互**。

| 项 | 语义 |
|---|---|
| 加在哪个对象 | 任意 GameObject |
| 标签列表 | 把要改的 tag 加进列表 |
| `auto rename` | 让 MA **自动选一个唯一名字** |
| 手动指定名字 | 自己填 |
| 生效范围 | **同一个 GameObject 之下**的 contact 的 tag 会被相应改名 |

### 16.1 陷阱与进阶

- ⚠️ **把组件放在「位于别处的 contact 的 Root Transform 的父对象」上，不会改到那个 contact 的 tag** → 判据是「该 contact 是否在本组件 GO 之下」，不是「谁是谁的父级」。
- **可嵌套**：子组件规则**先**改名，父组件规则**在子组件结果之上再改一次** → 可用简单组件拼出「重命名规则层级」，搭复杂机关。

〔KB 衔接〕KB 只在 `02-03:40,52,213` 把 Contact 列成检查项，**从未讲 tag 命名层**；「两个机关不触发」「进别人世界我的开关乱动」都可能落在这一层。
坑：① 改了名但机关不生效→位置关系不对（16.1）。② 想复用同一 tag 联动→手动指定同名。③ 嵌套时以为父级先执行→**子先父后**。

---

## 17. Convert Constraints

**出处**：https://modular-avatar.nadena.dev/docs/reference/convert-constraints

**定位**：构建时**非破坏地**把 Unity constraints 转成 **VRChat constraints**。**生效面**：构建期。

### 17.1 范围

- 转换**组件所在对象及其所有子级**上的**任何 constraint**；
- **以及引用这些 constraint 的动画**；
- 还会**尝试修复**「用旧版 MA 时点了 VRCSDK `Auto Fix` 导致损坏的动画」。

### 17.2 为什么放 avatar root

- 放在 avatar root 多数情况**是好主意**：**预转换（preconverting）能显著提升性能**。
- **装了 MA 之后，VRCSDK 的 `Auto Fix` 按钮会自动把这个组件加到 avatar root**（如果还没有）→「我没加过它却出现在根上」是正常的 SDK 行为。

### 17.3 存在的主要目的

官方明说：提供它**主要是为了让用户能移除它来禁用此功能**（当怀疑是这个功能导致问题时）→ 遇到「约束相关行为异常」，**删掉它试一次**是官方建议路径。

〔KB 衔接〕`outfit-fit-and-clipping:117` 已有很有价值的一条实测：该组件把 Unity `ParentConstraint` 烘成 `VRCParentConstraint` 时 **`m_RotationOffsets`/`m_TranslationOffsets` 原样传递、不按当前场景位姿重算**。**本条目不要重复**，两边互链即可（本节讲「组件是什么/为什么在根上」，那节讲「offset 语义」）。
坑：① 以为它是运行时组件。② 只加在深层对象→只转该子树，要全头像转就放根。③ 怀疑约束异常时不知道可以移除它。

---

## 18. 决策速查表（供 `02-03` 决策总表补行）

| 需求 | 用哪个 | 运行时 | 关键提醒 |
|---|---|---|---|
| 蒙皮衣服跟随头像骨架 | **Merge Armature** | 否 | 通用 prefab 别用 |
| 把单个对象塞进头像某骨 | **Bone Proxy** | 否 | 通用件用 `As child at root` |
| 只对齐一次、之后不跟 | **Move To** | 否 | 烘焙后组件自删 |
| 编辑器里挪对象不动子件 | **Move Independently** | **纯编辑器** | 调 fit 用 |
| 换掉头像上某个对象 | **Replace Object** | 否 | 一个对象只能被换一次 |
| 调单根骨尺寸不歪子骨 | **Scale Adjuster** | 编辑器工具 | 等比缩放别用它 |
| 抬高头像让鞋底贴地 | **Floor Adjuster** | 否 | **多个同时存在=全失效** |
| 道具钉在世界不跟头像走 | **World Fixed Object** | 是（constraint） | 只生成一个 constraint |
| 对象跟世界缩放 | **World Scale Object** | 是（Scale Constraint） | **编辑器不预览** |
| 第一人称看见自己头发/头饰 | **Visible Head Accessory** | 否（改网格） | 不能作 PB 链子级 |
| 换 shader 后顶点色污染 | **Remove Vertex Color** | 否 | `Keep Vertex Colors` 排除 |
| 统一全头像 bounds/灯探针 | **Mesh Settings** | 否 | 初始什么都不做 |
| 标记一套衣服的根 | **Outfit Root** | 否 | 组件本身不做任何事 |
| 配件不被父 PB 甩动 | **PhysBone Blocker** | 否 | 与 Bone Proxy 同对象 |
| 与别人头像的 PB 互碰 | **Global Collider** | 是 | **>6 个覆盖食指 collider** |
| 避免 contact tag 撞名 | **Rename VRChat Collision Tags** | 否 | 只改同一 GO 之下的 contact |
| 构建期预转 Unity 约束 | **Convert Constraints** | 否 | 放根；Auto Fix 会自动加 |

---

## 19. 建议落地（KB 补丁方案）

> **2026-11-01 并入后**：本节是审计稿原文。实际落点＝**本文件**（`references/05-动骨与穿模/05-10-ma-bone-and-geometry-components.md`，含第 1–17 节全部组件条目）；`00-基底/` 是上游原件不改，故**没有**采用 20.1 的「扩 00-02-modular-avatar.md」方案，改为独立分册。下表里的插入点建议保留作后续可选增强。

### 20.1 首选：扩 `references/00-基底/00-02-modular-avatar.md`

现该册速查表只有 **8 个组件**（`00-02-modular-avatar.md:11-20`：MenuItem / MenuInstaller / Parameters / MergeAnimator / ObjectToggle / MaterialSetter / ShapeChanger / BlendshapeSync）。建议：

1. **速查表 8 行 → 20 行**（补本文 17 个 + 表里漏的 `MaterialSwap` / `MeshCutter` / `PlatformFilter` 等按实际补）。
2. 表后加三节分组详解，直接把本文粘进去：
   - `## 骨骼与合并类`：Merge Armature(1) / Bone Proxy(2) / Move To(3) / Move Independently(4)
   - `## 物体与几何类`：Replace Object(5) / Visible Head Accessory(10) / Remove Vertex Color(11) / Mesh Settings(12) / Outfit Root(13)
   - `## 约束与物理类`：Convert Constraints(17) / PhysBone Blocker(14) / Global Collider(15) / Rename VRChat Collision Tags(16) / Scale Adjuster(6) / Floor Adjuster(7) / World Fixed Object(8) / World Scale Object(9)
3. **新增 `## Setup Outfit 到底做了什么`**（内容取 1.12 + 13.1 + 12.3）——KB 全书高频引用却从未解释。

### 20.2 备选：拆分册

若 `00-02-modular-avatar.md` 合并后过大，已**新建 `references/05-动骨与穿模/05-10-ma-bone-and-geometry-components.md`**（骨骼/物体/几何组件详解），并在 `references/README.md`（54 → 58 册索引）登记。**不建议拆多册**——这些组件高度耦合（Merge/Bone/Outfit/Scale 是同一套 fit 工具链）。

### 20.3 其它册插入点

| KB 文件 | 插入内容 | 来源 |
|---|---|---|
| `07-验证与排查/07-01-bake-and-validation.md` | 新增「MA 烘焙期对引用与层级的改写」表：animator 按属性类型分流 / PhysBone+Contact target / ParentConstraint / Replace Object / Move To 时序 / Convert Constraints 范围 | 1.3、2.5、3.2、5.2、17.1 |
| `05-动骨与穿模/05-08-cross-avatar-garment-fit-newfit.md` | 新增「官方 fit 工具组」：位置锁定三档、Reset position 三选项（含 arm-length 整体缩放）、Match Scale Adjusters、A↔T pose 转换、Scale Adjuster、Move Independently、Floor Adjuster | 1.4–1.6、6、7、4、13.2 |
| `00-基底/00-03-physbone-dynamics.md` | 新增「全局碰撞与接触标签」：PhysBone Blocker、Global Collider、Rename VRChat Collision Tags | 14、15、16 |
| `05-动骨与穿模/05-09-third-party-tool-prefab-binding-abt.md` | Bone Proxy 表从 2 行扩到 4 行 + Match Parent Scale + 自动选模式 + Advanced + 改动画路径 + 与 Blocker 联用 | 2.2–2.6、14.1 |
| `02-工程组织与工作流/02-03-...md` | 决策总表按 §18 补 9 行；§2 补 1.4–1.7；§14 按 12.1/12.2 扩表；§15 按 6/7 扩写 | §18、1、12、6、7 |
| `02-工程组织与工作流/02-09-project-workflow-notes.md` | `§4.1` 把「右键 → Setup Outfit」扩成 1.12 的 6 步清单 | 1.12 |
| `04-外观与表情/*`（lilToon/材质册） | Remove Vertex Color（换 shader 后偏色的因果链）+ Visible Head Accessory | 11、10 |
| `07-验证与排查/00-07-troubleshooting.md` | World Scale Object 编辑器不预览、Floor Adjuster 多组件 no-op、Convert Constraints 可移除禁用、Replace Object 引用置 null | 9.1、7.1、17.3、5.2 |
| `00-基底/00-02-modular-avatar.md` | 显式消歧：官方 `ModularAvatarOutfitRoot` ≠ KB 已废弃的自建 FX 层 `OutfitRoots` | 13.3 |

---

## 20. 待实测 / 需查源码

| # | 项 | 处理建议 |
|---|---|---|
| 1 | `LockMode=BaseToMerge`（`05-04:41`、`05-08:1047`）对应三档中的哪一档 | 查 MA 1.18.7 `PositionLockMode` 枚举，别凭字面猜 |
| 2 | `mangleNames` 与 `Avoid name collisions` 是否同一字段/是否反向 | 同上；写反会教错操作 |
| 3 | `05-09:32` 的 `WorldFixed` 是否真挂了 `ModularAvatarWorldFixedObject` | 选中对象看 Inspector；若挂了则 WFO 已有间接实测 |
| 4 | `02-03 §15`「先查版本与执行顺序」是否已隐含 Floor Adjuster 1.17.1 顺序坑 | 若是，FA 可从「零语义」升级 |
| 5 | Visible Head Accessory 的 proxy 骨在 AAO 合并后是否改名/被合并 | 烘焙后数骨对名；影响骨架断言写法 |
| 6 | Global Collider 劫持的内置 collider 列表是否随 VRCSDK 版本变化 | 换 SDK 版本复测；影响「>6 覆盖食指」的实际边界 |
| 7 | Convert Constraints 与 KB 已知 offset 原样传递在 1.18.7 是否一致 | 复跑 `outfit-fit-and-clipping:117` 的实验 |

---

> 本文只覆盖 MA 骨骼/物体/几何类组件；Reaction 类、菜单/参数类、通用机制/教程/分发类由 g1/g3/g4 报告覆盖。
> 全部事实出自 `<VRCHAT_ROOT>\tools\tmp\ma-docs-txt\docs__reference__*.txt` 与 `docs__distributing-prefabs__for-outfit-creators.txt`；KB 只读，未修改任何 KB 文件。


---

## 21. 2026-11-09 实测：万圣节衣装 bindpose 塌缩与 MergeArmature 修复

**来源**：<https://modular-avatar.nadena.dev/docs/reference/merge-armature>

**现象**：万圣节卫衣一键穿衣后四件套全部不可见，表现为全身裸体；菜单参数与 ObjectToggle 链路均正常。

**根因**：prefab 网格经过 MA 重定向后，`RootBone` / `ProbeAnchor` 指向了错误层级；`MergeArmature` 的绑定空间与目标骨架不一致，生成网格的 bindpose 使顶点塌缩。只改菜单或 Renderer 的 active/enabled 不能修复。

**本工作区修复**：

1. 从工程内原始 `Assets/mmiao/おばけパーカー/FBX/おばけパーカー.fbx` 恢复网格，不复制无关模型。
2. 生成 `Assets/ZZZ_GeneratedAssets/R400/Halloween/*_R400Rebind.asset`。
3. `RootBone`、`ProbeAnchor` 都设为 `armature/Hips`。
4. `MergeArmature.LockMode = BaseToMerge`。
5. 用 `bone.worldToLocalMatrix * renderer.localToWorldMatrix` 重算 bindposes。

**坑**：`LockMode` 的字符串值不要凭字面猜；本工程当前 MA 版本下 `BaseToMerge` 已实测可用，但升级 MA 后应重新读字段枚举。

---

## 22. 2026-11-10 实测：BoneProxy `AsChildKeepWorldPose` 接「按真人尺度做的道具 prefab」——尺度与平移的定量反推

**来源**：<https://modular-avatar.nadena.dev/docs/reference/bone-proxy>（"As child keep world pose" 明确写着：object 会被 reparent 到目标对象下，但**保留其世界位置与旋转**）

**现象**：Q 版头像（全高 0.64 m）装一把按真人尺度建模的通用手枪 prefab。烘焙后枪**不在手里**，而是停在脚踝高度（实测 body AABB `center=(-0.022, 0.087, -0.010) size=(0.035, 0.139, 0.032)`，且被 Holster 约束钉住）；Play 里无论怎么切装备参数都不动，但动画层、菜单、参数全部正常。

**根因**：4 个挂点都用 `ModularAvatarBoneProxy` + `attachmentMode = AsChildKeepWorldPose`。读 MA 源码 `Packages/nadena.dev.modular-avatar/Editor/BoneProxyProcessor.cs:81-102`：

```csharp
case BoneProxyAttachmentMode.AsChildKeepWorldPose:
    keepPos = keepRot = true;
    break;
```

烘焙时它把挂点**编辑态的世界位姿**原样搬到目标骨下。prefab 按 1.0 真人尺度建模（`Attach_SIHG_Hand_L.localPosition = (-0.535, 1.285, 0.017)`，y=1.285 ≈ 真人手高），而本头像 `Hand.L` 世界坐标只有 `(-0.29046, 0.58217, 0.00683)`。于是挂点被「保位姿」保在了 1.285 那个高度，再经整体缩放后落到脚踝。

**关键判断**：不需要改 prefab、不需要改 `attachmentMode`、不需要改挂点 `localPosition`（改 `localPosition` 会被 keepWorldPose 直接覆盖，白改）。

**定量反推（本工作区实际用法）**：

1. 从挂点 localPosition 读出 prefab 的设计尺度基准：`anchorY0 = 1.28500`（= `Attach_SIHG_Hand_L.localPosition.y`）。
2. 读目标骨当前世界 y：`handY = Hand.L.position.y = 0.58217`。
3. `scale gk = handY / anchorY0 = 0.58217 / 1.28500 = 0.45306`。
4. 缩放后挂点会落到 `anchorLocal * gk`，与目标骨世界位置只差一个平移，逐轴反算根位置：

```csharp
Vector3 rootWorld = new Vector3(hp.x - (-0.53500f * gk),
                                hp.y - ( 1.28500f * gk),
                                hp.z - ( 0.01700f * gk));
inst.transform.localScale    = Vector3.one * gk;
inst.transform.localPosition = av.transform.worldToLocalMatrix.MultiplyPoint3x4(rootWorld);
```

   实测得到 `scale=(0.45305,0.45305,0.45305)`、`rootLocalPos=(-0.04807,0.00000,-0.00087)`。
5. 同时把挂点用的骨骼名解析改稳：**用绝对路径 `transform.Find("armature/Hips/Spine/Chest/Shoulder.L/UpperArm.L/LowerArm.L/Hand.L")`**。本工程同名 `Hand.L` 有 5 个（scale 分别 1.000 / 1.050 / 1.000 / 0.645 / 100.000），深搜必错。

代码位置：`Assets/Editor/R400Fix.cs:136-146`。

**判据（三条都要）**：

- 烘焙后挂点世界位姿 == 目标骨世界位姿。实测 `attach Attach_SIHG_Hand_L wp=(-0.2905,0.5822,0.0068) wr=(0,270,90)` 与 `Hand.L=(-0.2905,0.5822,0.0068)` **逐位一致**。
- 道具整体长度落在合理区间：枪长 `0.28959 * 0.45306 ≈ 0.131 m`，Play 实测 `len=0.1221` ∈ [0.10, 0.16]。
- 道具相对手的距离 ≤ 0.12 m（见 §23 的 pose 无关判据）。

**坑**：

1. 只改 `attachmentMode` 不改尺度：挂点会贴到骨头但**尺寸仍是真人大小**，枪会变成 0.29 m 长、比整个 Q 版角色还高。
2. 只改尺度不改根平移：x 方向仍差 `0.048`（`-0.29046 + 0.535*0.45306 = -0.04807`），枪会偏出 4.8 cm。
3. 这类 prefab 的 4 个挂点（Hand_L / Holster_L / MagHand_L / MagCatch_L）**必须一起缩**，单独缩一个会让弹匣换弹动作对不上。

---

## 23. 2026-11-10 实测：活动约束实例上量几何 = 垃圾数据；pose 无关的握持判据

**来源**：<https://docs.unity3d.com/2022.3/Documentation/ScriptReference/SkinnedMeshRenderer.BakeMesh.html>；<https://docs.unity3d.com/2022.3/Documentation/Manual/class-ParentConstraint.html>

**现象 A（量测基准）**：在**场景实例**上对 8 个 SMR 做 `BakeMesh` + `TransformPoint` 求 AABB，得到 1~6 mm 的尺寸，或 `min=(-0.19432,-0.07076,0.74700)` 这类明显荒谬的值，与目视严重不符；同一物体不同时刻量出的值还不一样。

**根因**：该实例上 28 个 `ParentConstraint` 全部 `constraintActive=true` + `locked=true`。Unity 在编辑器里**实时重排**这些约束，`BakeMesh` 取的骨骼矩阵与随后乘的 `localToWorldMatrix` 不属于同一时刻，乘积无物理意义。`BakeMesh` 的顶点是相对 **SMR 自身 Transform** 的（官方原文：The vertices are relative to the SkinnedMeshRenderer Transform component），前提是骨骼矩阵稳定。

**正确做法（两选一，都不在活动约束实例上量）**：

1. **prefab rest pose 解算**：读 prefab 资产（不实例化到场景），按 bindpose 手算 `bone.worldToLocalMatrix * renderer.localToWorldMatrix`，逐个顶点变换。
2. **Play 里量 MA 烘焙克隆**：进 Play 后 MA/NDMF 已把约束解算成静态层级，此时的 `BakeMesh` 结果可信（本工程 Play 第 6 轮 `[diag] all10gun center=(-0.140,0.375,-0.036) size=(0.043,0.159,0.121)` 才是有意义的值）。

**现象 B（判据设计）**：握持判据写成「道具中心世界 y > 0.45」时，同一份正确修复在 Play 里 PASS/FAIL 交替。

**根因**：手势/待机动画让手上下动，**绝对高度不是 pose 无关量**。实测同一次 Play 中 `Hand.L.y` 自身就在 0.375~0.45 之间波动（`ctrY=0.3749` 与 `Hand.L.y=0.377` 同高，但因为低于 0.45 被判 FAIL）。

**正确判据**：

- **pose 无关**：道具 body 的烘焙中心到 `Hand.L` 骨骼的**距离** ≤ 0.12 m（实测 0.0629 m）。
- 配合尺寸区间：`len ∈ [0.10, 0.16] m`。
- 收起态另判：`self=False`、`vis=0`、`particles=0`。

**坑**：

1. 用绝对坐标/绝对高度做握持判据，会把「动画在动」误判成「装错了」。
2. 在场景实例上量完就下结论，会把「约束在动」误判成「模型坏了」，进而去改本来正确的 prefab。
