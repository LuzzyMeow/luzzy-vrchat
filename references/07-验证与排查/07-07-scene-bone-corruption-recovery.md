# 场景骨骼数据被污染：诊断与按路径还原

> 本工作区实测 2026-10-27（第 102 轮，LuzzyMeow V2）。症状：打开工程测试时「衣服、头发、饰品全部出现在错误位置」，第一反应都是缓存/烘焙问题。
> **结论：不是缓存，是保存进 `.unity` 场景文件的 Transform 数据被改坏了。** 本册记录定位与修复的完整方法。

---

## 1. 症状与第一判据

- 编辑器 Scene 视图里角色**穿好衣服但整体错位**：服装/头发/眼镜在正常高度（Head 世界 y≈0.641），而**身体骨架的 Head 只有 y≈0.211** → 视觉上「头掉到胸口、衣服浮在头上」。
- 进 Play 烘焙后仍然错位（因为 bake 读的就是场景里的 transform）→ **说明不是编辑态假象，也不是缓存**。
- 用 `SkinnedMeshRenderer.BakeMesh + transform.localToWorldMatrix` 量渲染中心最直观：
  - `Body`（猫头）mesh 空间 bounds center y=0.6887 → 实际渲染 center y=**0.2600**（低 0.43）；
  - `Body_Base` center y=**-0.0817**（脚在 -0.38）；
  - 而 `睡衣/PJ_Hat` y=0.7513、`PJ_Shirt` 0.4887、`眼镜/Chain` 0.6655 —— 全部在「正确高度」。
  - 注意：`头发` 的 SMR 烘焙数（y≈72.7）**无意义**，它的 armature `lossyScale=110`，必须先除以缩放再判。

## 2. 诊断：全场景 Transform diff（脚本比肉眼可靠）

### 2.1 解析 `.unity` YAML

场景是 YAML 文本，结构固定：`--- !u!4 &<fileID>` 是 Transform 块，`--- !u!1 &<id>` 是 GameObject 块。要点：

- **非 ASCII 的 `m_Name` 在文件里是 `"\uXXXX"` 转义**，解析时必须 `unescape`（Python：`s.encode().decode('unicode_escape')`，或 `json.loads('"' + s + '"')`）。
- 用 `m_GameObject.fileID` 把 Transform → GameObject 名；用 `m_Father` 建父子关系，再递归拼**完整路径**（本工作区路径形如 `沐梓溪喵/armature/Hips/Spine/...`）。
- 只比 `m_LocalPosition / m_LocalRotation / m_LocalScale`，浮点按 `%.6f` 归一（避免末位噪声）。

### 2.2 与「最近的已知好备份」做集合 diff

本工作区实测（2026-10-27）：

- 当前 5180 个 Transform、13:02 备份 5188 个；
- **CHANGED=57，全部落在一个头像子树内**，其它 7 个根（相机/灯/其余头像/GestureManager）0 变化 → 损坏面精确；
- ADDED=17 / REMOVED=25 = 中间那次「拆翅膀光环」的有意改动（**修复时绝不能回滚**：新增的 Wing/Halo 骨要保留）。

> 时间线定位法：把每个备份的 Hips 值都打印出来，**好值断点**就是损坏时间窗。本工作区：10-26 13:02 备份正常、19:18 备份已坏 → 损坏发生在当天 13:02–19:18（即那几轮「重建翅膀/默认关闭/重启编辑器/删两套翅膀/拆光环/移植扇动」工作期间），场景最后保存 20:07。

### 2.3 与「同基底头像」逐骨对照（找出应恢复的正确值）

同一场景里若有**同源模型**（本工作区：`小狼2.0`、`小狼2.0睡觉版`、`正太兔兔！` 与猫同为一个 Kipfel 基底），它们的静止骨值就是最好的参考：

- 本工作区 4 个头像的 `Hips/Spine/Chest/Neck/Head` 静止值完全一致；
- 猫坏掉的只有：`armature/Hips`（lp 差 0.431009）、`Eye.L/Eye.R`（lp 差 0.001149）、约 40 根四肢/手指的**旋转**（T-pose 化）；
- 姿势唯一性验证（`pose_compare.py`）：猫的 `UpperArm.L q=(-0.331696,-0.000121,0.243852,0.911325)` 与其它 3 个头像（都是 identity 系）**都不同** → 是「猫独有被摆了个姿势」，不是「从别的头像复制过来」；
- 反查元凶：全工程 `*.anim/*.controller/*.asset` 里搜这些特征浮点（如 `0.965914`/`-0.331696`）**0 命中** → 不是采样了工程里某个 clip，而是运行时被脚本/动画模式直接写进场景的（没有留下资产痕迹）。

## 3. 修复：生成路径+TRS 清单，在 Unity 内逐条写回

```python
# make_restore.py（在好备份上跑，输出 restore_list.txt）
# 每行: 相对路径|px,py,pz|rx,ry,rz,rw|sx,sy,sz   （%.9g 全精度）
```

```csharp
// execute_code（CodeDom C#6）：读清单 → root.Find(rel) → 写回
var root = ...;                       // 该头像根
var lines = File.ReadAllLines(listPath);
int applied = 0, miss = 0;
foreach (var line in lines) {
    var parts = line.Split('|');
    var t = root.transform.Find(parts[0]);      // parts[0] 以头像根名开头时，先去掉根名
    if (t == null) { miss++; continue; }
    t.localPosition = ParseV3(parts[1]);
    t.localRotation = ParseQ(parts[2]);
    t.localScale    = ParseV3(parts[3]);
    applied++;
}
EditorSceneManager.SaveScene(scene);
```

本工作区实测：清单 60 行、`APPLIED=60 MISS=0`；写回前必须确认 `AnimationMode.InAnimationMode()==False`（否则写进去的值会被动画覆盖）。
写回后立即数值复核：`Head` 世界坐标回到 `(0, 0.64141, 0.00451)`、`Hips.localPosition=(-0.000114,-0.010764,0.418339)`、`眼罩` 根回 `(0,0.641408,0.004513)`（此前与坏头同位 y=0.211）。

### 3.1 连带被拖走的「跟随物」也要一起还原

本工作区 57 条 CHANGED 里，除骨架外还有**跟着 Hips/头走的位置类对象**（都在同一子树内，必须一起还原，否则修完骨架它们反而错位）：

- `睡衣/OG_GhostAnchor/Ghost_HeadMarker`（挂在头上的幽灵标记）
- `睡衣/OG_GhostAnchor/Ghost_FollowRoot/Ghost_OrbitCenter/*_Base/*_VisualRoot` 等 8 个幽灵视觉根
- `N式スーパーよわちんv1.0/N式スーパー短小包茎Hips固定化用`（它的 lp 恰等于 `R_x(-90°)·Hips.lp`，是随 Hips 同步重算的助手对象）
- `Chiffon_.../N_dick`、`眼罩` 根、`内裤/armature/Hips`

判据：**凡是「位置差 ≈ 0.43039」的都在同一条链上**，按备份值一次性写回即可。

## 4. 修完如何验收（缺一不可）

1. **编辑态渲染**：自建临时相机（`Main Camera` 也可）`RenderTexture` 出图 → 身体回到正确高度（帽在头上、幽灵在胸前、裤/鞋贴合）。
   - ⚠️ 编辑态**袖子仍可能水平**而手臂下垂：衣服 SMR 绑在自己 armature 的 T-pose 绑形上，MA 只在 bake 时把衣服并到宿主骨 → **这是正常假象**，不要据此再改骨骼。
2. **SMR 烘焙中心复测**（见 §1 的方法）：`Body` center y≈0.666、`Body_Base` y≈0.367、`PJ_Hat` y≈0.736、`Eyemask` y≈0.775、`headphones` y≈0.702、`Chain` y≈0.666。
3. **进 Play 烘焙复验**：NDMF 日志出现 `Starting processing for avatar: <名字>(Clone)`，`Console error` 只应有既有警告；game_view 截图确认「袖子跟着手臂下垂、帽子在头上、无浮空件」。
4. 退出 Play 后**还原会被 Play 重写的生成资产**（本工作区：`Assets/08_动画·菜单·图标/NyaPPu__Generated/LightController/*.anim` 5 个，从原始包目录拷回并比 MD5）。
5. 保存场景并复核 `isDirty=False`、根数与子对象数。

## 5. 预防（写进习惯）

- **不要在编辑器里对「场景内的头像」直接播放/采样动画或批量写骨骼**（`AnimationMode`、`Animator.Update`、`clip.SampleAnimation` 都可能把当前姿态**留在场景对象上并随保存落盘**）。要预览就进 Play（NDMF 会烘焙克隆），或在临时副本上做。
- 每次大改前 `_asset_reorg/backup/<场景>.bak-YYYY-MM-DD-before-<动作>` 备份场景，**备份文件名要带语义**，出事后能用「时间断点 + 内容 diff」精确定位。
- 一旦发现「全身错位」，先做 §2 的 diff，**不要**先删缓存/重建 Library/重装包 —— 本工作区实测这些都不是原因，反而浪费几十分钟。

## 6. 边界

- 本册的修复口径 = 「还原到最近已知好备份的静止姿态」。如果损坏时用户**有意**调整过某些骨骼/姿势，还原会覆盖这些有意改动 → 修复前先与用户确认，或只还原确认损坏的子树。
- 本工作区该头像修复后已实际 Build & Publish 成功，但**游戏内实际观感 / 他人视角 / Rest Pose（屏蔽动画）NOT_RUN**。
