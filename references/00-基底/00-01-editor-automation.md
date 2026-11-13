# 从外部驱动 Unity 编辑器（无头自动化）

在远程桌面 / ToDesk 环境下，Unity 顶部菜单的下拉经常打不开，合成鼠标点击也点不动。可靠的做法是**用文件当命令通道**，让编辑器脚本自己去执行。

---

## flag 文件 + `[DidReloadScripts]` 一次性执行器

最可靠的"在编辑器里跑一段代码而不碰 UI"的模式：

```csharp
const string FLAG = "_myop.flag";   // 相对项目根目录

[UnityEditor.Callbacks.DidReloadScripts]
static void AutoRun()
{
    if (!System.IO.File.Exists(FLAG)) return;

    string what = "default";
    try { what = System.IO.File.ReadAllText(FLAG).Trim(); } catch { }
    try { System.IO.File.Delete(FLAG); } catch { }   // 立刻消费掉 —— 保证幂等

    if (what == "step2") EditorApplication.delayCall += Step2;
    else                 EditorApplication.delayCall += Step1;
}
```

PowerShell 侧：

```powershell
"step2" | Set-Content "<PROJECT_ROOT>\_myop.flag" -NoNewline
# 然后强制重编译（见下）
```

**为什么要 `delayCall`**：`[DidReloadScripts]` 在场景完全加载之前就触发了。包一层 `delayCall` 推迟一帧，场景层级才可查询。

**为什么先删 flag**：如果操作抛异常，flag 已经删掉了，下次编译不会无限重触发。

**为什么用 ASCII 日志标记**：Unity 的 `Editor.log` 里中文常是 GBK 编码，`Select-String` 匹配不可靠。统一用 `[MyTag]` 这种 ASCII 前缀。

---

## PowerShell 强制重编译

写完 flag 之后，需要让 Unity 触发一次脚本重载（`Ctrl+R`）。整套动作：

```powershell
# 1. 先杀掉抢焦点的进程（模拟器、串流、自动化工具等，按你机器上的实际情况填）
Get-Process | Where-Object { $_.ProcessName -match '<FOCUS_STEALER_1>|<FOCUS_STEALER_2>' } |
    Stop-Process -Force

# 2. 拿到正确的 Unity hwnd（排除 hwnd=0 的辅助进程）
$u = Get-Process Unity | Where-Object {
        $_.MainWindowHandle -ne 0 -and $_.MainWindowTitle -match 'Unity'
     } | Select-Object -First 1
$hwnd = $u.MainWindowHandle

# 3. Alt 解锁技巧置顶（后台进程直接 SetForegroundWindow 会失败）
Add-Type @'
using System; using System.Runtime.InteropServices;
public class W32 {
    [DllImport("user32.dll")] public static extern bool SetForegroundWindow(IntPtr h);
    [DllImport("user32.dll")] public static extern IntPtr GetForegroundWindow();
    [DllImport("user32.dll")] public static extern bool ShowWindow(IntPtr h, int n);
    [DllImport("user32.dll")] public static extern void BringWindowToTop(IntPtr h);
    [DllImport("user32.dll")] public static extern void keybd_event(byte bVk, byte bScan, uint dwFlags, UIntPtr dwExtra);
    [DllImport("user32.dll")] public static extern bool SetCursorPos(int x, int y);
    [DllImport("user32.dll")] public static extern void mouse_event(uint dwFlags, int dx, int dy, uint dwData, UIntPtr dwExtra);
}
'@
for ($i = 0; $i -lt 8; $i++) {
    [W32]::keybd_event(0x12, 0, 0, [UIntPtr]::Zero)   # Alt down
    [W32]::ShowWindow($hwnd, 3)
    [W32]::BringWindowToTop($hwnd)
    [W32]::SetForegroundWindow($hwnd)
    [W32]::keybd_event(0x12, 0, 2, [UIntPtr]::Zero)   # Alt up
    Start-Sleep -Milliseconds 200
    if ([W32]::GetForegroundWindow() -eq $hwnd) { break }
}

# 4. 点一下编辑器主面板（例如 Project 窗口），让键盘焦点进去
[W32]::SetCursorPos(430, 560)
[W32]::mouse_event(2, 0, 0, 0, [UIntPtr]::Zero)   # LBUTTONDOWN
Start-Sleep -Milliseconds 80
[W32]::mouse_event(4, 0, 0, 0, [UIntPtr]::Zero)   # LBUTTONUP

# 5. 连发两次 Ctrl+R
Start-Sleep -Milliseconds 300
foreach ($n in 1..2) {
    [W32]::keybd_event(0x11, 0, 0, [UIntPtr]::Zero)   # Ctrl down
    [W32]::keybd_event(0x52, 0, 0, [UIntPtr]::Zero)   # R down
    [W32]::keybd_event(0x52, 0, 2, [UIntPtr]::Zero)   # R up
    [W32]::keybd_event(0x11, 0, 2, [UIntPtr]::Zero)   # Ctrl up
    Start-Sleep -Milliseconds 500
}

Start-Sleep -Seconds 26   # 等编译
```

点击坐标要按实际窗口布局调整，目标只是"落在编辑器某个面板空白处"。

---

## 验证编译真的发生了

**看 DLL 的 mtime，不要看 Editor.log 的报错行**（日志会累积陈旧错误，很容易误判）：

```powershell
$dll = "<PROJECT_ROOT>\Library\ScriptAssemblies\Assembly-CSharp-Editor.dll"
$cs  = "<PROJECT_ROOT>\Assets\Editor\MyTool.cs"
(Get-Item $dll).LastWriteTime -gt (Get-Item $cs).LastWriteTime
# True = 重编译成功
```

确认操作真的跑了，抓 ASCII 标记：

```powershell
Select-String "\[MyTag\]" "$env:LOCALAPPDATA\Unity\Editor\Editor.log" | Select-Object -Last 20
```

---

## 焦点抢占：Ctrl+R 不生效的头号原因

**症状**：发了 Ctrl+R，DLL mtime 纹丝不动。

**原因**：某个进程抢走了前台焦点，Ctrl+R 发给了它。VR 串流服务（PICO Connect 的 `ps_service` + `OVRService`、SteamVR 等）是常见元凶，而且没有管理员权限杀掉后会立刻重生。

**诊断**：

```powershell
Get-Process | Where-Object MainWindowTitle -ne '' |
    Select-Object ProcessName, MainWindowTitle
```

**处理**：让用户从托盘退出该程序；或用管理员权限停服务：

```powershell
Stop-Service ps_service,OVRService -Force   # 需要管理员
```

**另一个坑**：`Get-Process Unity` 会返回多个条目（子编辑器、batchmode 实例）。主编辑器窗口的特征是 `MainWindowHandle != 0` 且 `MainWindowTitle` 含 "Unity"。永远要过滤。

---

## 触发菜单项：用 ExecuteMenuItem，不要合成鼠标点击

远程桌面下点下拉菜单基本必失败（菜单根本不展开）。改用：

```csharp
EditorApplication.ExecuteMenuItem("MyTools/DoThing");
EditorApplication.ExecuteMenuItem("Tools/Modular Avatar/Manual bake avatar");  // 需先选中 avatar
```

配合上面的 flag 模式，就能从 PowerShell 间接触发任意菜单项。

---

## 离屏渲染 PNG（验证用截图）

比 Scene 视图截图可靠得多：可控光照、可控背景、可复现机位。

```csharp
static void RenderShot(Vector3 camPos, Vector3 lookAt, string outPath, int w = 560, int h = 560)
{
    var camGO = new GameObject("_rc");
    var cam = camGO.AddComponent<Camera>();
    cam.clearFlags = CameraClearFlags.SolidColor;
    cam.backgroundColor = new Color(0.25f, 0.5f, 0.3f);   // 绿底，和肤色/白布都有对比
    cam.nearClipPlane = 0.002f;
    cam.transform.position = camPos;
    cam.transform.LookAt(lookAt);
    cam.fieldOfView = 22;

    // 低强度方向光，避免眼睛等高光面出现色斑
    var lgo = new GameObject("_rl");
    var l = lgo.AddComponent<Light>();
    l.type = LightType.Directional;
    l.intensity = 0.6f;
    lgo.transform.rotation = Quaternion.Euler(15, 180, 0);

    RenderSettings.ambientMode  = UnityEngine.Rendering.AmbientMode.Flat;
    RenderSettings.ambientLight = new Color(0.85f, 0.85f, 0.85f);

    var rt = new RenderTexture(w, h, 24, RenderTextureFormat.ARGB32);
    rt.antiAliasing = 8;
    cam.targetTexture = rt;
    cam.Render();

    RenderTexture.active = rt;
    var tx = new Texture2D(w, h, TextureFormat.RGB24, false);
    tx.ReadPixels(new Rect(0, 0, w, h), 0, 0);
    tx.Apply();
    System.IO.File.WriteAllBytes(outPath, tx.EncodeToPNG());

    cam.targetTexture = null;
    RenderTexture.active = null;
    Object.DestroyImmediate(camGO);
    Object.DestroyImmediate(lgo);
}
```

**光照配方**：方向光 `intensity = 0.6` + 平坦环境光 `0.85`。方向光强度超过 0.8 会在光滑眼球 shader 上产生高光色斑，看起来像贴图损坏。

**双状态渲染陷阱**：在同一次 `DidReloadScripts` 调用里连续渲染状态 A 和状态 B，只有第一个状态会正确蒙皮形变 —— Unity 每帧只重写一次 skin。解决办法：用两个 flag 值（`render_a` / `render_b`），让每个状态在各自的编译周期里完成。

**验证穿模要多角度**：正面、侧面、腰部至少各一张。只拍正面会漏掉侧向穿模。

---

## 最小审计脚本建议

项目没有自动化时，在 `Assets/Editor/` 写一个最小审计工具，把报告输出到 `_stage/`：

- avatar roots 和 descriptor
- expression menu 树
- expression parameters bit budget
- FX controller 的 layers / parameters
- renderer → material → shader → texture 摘要
- Modular Avatar / VRCFury installer 引用
- PhysBone / contact / constraint 摘要
- 互动插件 target/socket 的空间位置
- bake / upload 结果

这些文本报告比手点 Inspector 可靠，也方便下一轮接手的人（或 agent）继续工作。

---

## 补充：本工作区扩展路由（2026-11-11 整理，原文保留）

- 建工程 / 开工程 / MCP stdio 桥（6400 vs HTTP 8080 互斥）/ 桥断自救 / 重启配方：[01-06](../01-环境与工具链/01-06-project-launch-and-mcp-bridge.md)。
- `execute_code` 绑定蒸发的 flag-file 兜底、DeleteAsset 拦截、delayCall 失焦、陈旧程序集：[07-09 §8](../07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md)。
- 离屏渲染的蒙皮刷新（ForceSkinnedUpdate）、特写取景按骨坐标、快照五段式：[07-09 §5–§7](../07-验证与排查/07-09-dynamic-verification-and-evidence-recipes.md)。
- GM 自动化 harness（反射 + 两遍法 + 截图判据）：[07-06](../07-验证与排查/07-06-gesture-manager-automated-sim-test.md)。
