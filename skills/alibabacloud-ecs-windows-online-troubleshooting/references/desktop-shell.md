# Desktop Shell 诊断

## 功能说明

诊断 Windows 登录 Shell 配置、Explorer.exe 进程状态、任务栏/开始菜单响应、Console 会话状态、DWM 和 DPI 缩放。覆盖 7 个已知问题项。

**输入**：用户问题描述（必选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 登录后黑屏或只有命令行窗口、没有桌面 | Step 1 (登录 Shell 配置) |
| 登录后只有壁纸没有任务栏 | Step 2 (Explorer.exe 进程状态) |
| 点击任务栏或开始菜单没有反应 | Step 3 (任务栏/开始菜单响应) |
| 桌面空白、Explorer 未加载桌面文件夹 | Step 2 (Explorer.exe 进程状态) |
| 窗口透明效果消失、拖拽卡顿 | Step 4 (DWM 状态) |
| 应用界面模糊、元素大小不对 | Step 5 (DPI 缩放) |
| VNC 黑屏、控制台无响应 | Step 6 (Console 会话状态) |

## 诊断步骤

### Step 1: 检查登录 Shell 配置

**数据采集**：

> 采集目标：获取 WinLogon Shell 配置，确认登录后启动的默认 Shell 程序

```powershell
# Check system-level Shell configuration
$shellSystem = (Get-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon' -Name 'Shell' -ErrorAction SilentlyContinue).Shell
Write-Host "System Shell (HKLM): $shellSystem"
# Check current user-level Shell configuration (may override system config)
$shellUser = (Get-ItemProperty -Path 'HKCU:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon' -Name 'Shell' -ErrorAction SilentlyContinue).Shell
if ($shellUser) {
    Write-Host "User Shell (HKCU): $shellUser"
} else {
    Write-Host "User Shell (HKCU): Not set (using system default)"
}
# Check Userinit
$userinit = (Get-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon' -Name 'Userinit' -ErrorAction SilentlyContinue).Userinit
Write-Host "Userinit: $userinit"
```

**分析思路**：

1. 检查 Shell 配置：
   - 正常：Shell 为 `explorer.exe`
   - 异常：Shell 为空或指向非标准程序 → **根因**：登录 Shell 配置异常，登录后将无法显示桌面，**严重程度**：Critical
2. 检查 Userinit 配置：
   - 正常：包含 `userinit.exe` 路径
   - 异常：为空或指向异常程序 → **根因**：Userinit 配置异常，登录过程无法正常初始化，**严重程度**：Critical

> 如果 Shell 配置异常且 C:\ 权限也有问题，参见 → [identity-permission.md](references/identity-permission.md)

### Step 2: 检查 Explorer.exe 进程状态

**数据采集**：

> 采集目标：检查 Explorer.exe 进程是否运行以及其资源占用情况

```powershell
# Check Explorer.exe process
$explorer = Get-Process -Name explorer -ErrorAction SilentlyContinue
if ($explorer) {
    $explorer | Select-Object Id, ProcessName, CPU, WorkingSet64, StartTime | Format-Table -AutoSize
    Write-Host "Explorer.exe is running ($($explorer.Count) instance(s))"
} else {
    Write-Host "WARNING: Explorer.exe is NOT running"
}
# Check Explorer crash events
Get-WinEvent -FilterHashtable @{LogName='Application'; Level=2} -MaxEvents 10 -ErrorAction SilentlyContinue |
    Where-Object { $_.ProviderName -eq 'Application Error' -and $_.Message -like '*explorer*' } |
    Select-Object TimeCreated, Id, Message | Format-Table -AutoSize
```

**分析思路**：

1. 检查 Explorer 进程是否存在：
   - 正常：至少一个 explorer.exe 实例在运行
   - 异常：未运行 → **根因**：Explorer.exe 未启动，桌面/任务栏/开始菜单不可用，**严重程度**：Critical
2. 检查崩溃事件：
   - 发现频繁 Explorer 崩溃事件 → **根因**：Explorer.exe 反复崩溃，可能由第三方 Shell 扩展或损坏的配置引起，**严重程度**：Warning

### Step 3: 检查任务栏/开始菜单响应

**数据采集**：

> 采集目标：检查任务栏和开始菜单相关的服务和配置

```powershell
# Check ShellExperienceHost and StartMenuExperienceHost (Windows 10+)
$shellHost = Get-Process -Name ShellExperienceHost -ErrorAction SilentlyContinue
$startMenu = Get-Process -Name StartMenuExperienceHost -ErrorAction SilentlyContinue
Write-Host "ShellExperienceHost: $(if ($shellHost) { 'Running' } else { 'Not running' })"
Write-Host "StartMenuExperienceHost: $(if ($startMenu) { 'Running' } else { 'Not running' })"
# Check taskbar auto-hide setting
$taskbarSettings = Get-ItemProperty -Path 'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\StuckRects3' -ErrorAction SilentlyContinue
if ($taskbarSettings) {
    Write-Host "Taskbar settings registry key exists"
}
# Check AppReadiness service (affects UWP apps and Start menu)
$appReadiness = Get-Service -Name AppReadiness -ErrorAction SilentlyContinue
if ($appReadiness) {
    Write-Host "AppReadiness Service: Status=$($appReadiness.Status), StartType=$($appReadiness.StartType)"
}
```

**分析思路**：

1. 检查 Shell 体验主机进程（Windows 10+）：
   - 正常：ShellExperienceHost 和 StartMenuExperienceHost 均在运行
   - 异常：进程未运行 → **根因**：开始菜单体验进程未启动，开始菜单可能无响应，**严重程度**：Warning

### Step 4: 检查 DWM 状态

**数据采集**：

> 采集目标：检查 Desktop Window Manager 服务和进程状态

```powershell
# Check DWM service
Get-Service -Name 'UxSms' -ErrorAction SilentlyContinue | Select-Object Name, DisplayName, Status, StartType | Format-Table -AutoSize
# Check dwm.exe process
$dwm = Get-Process -Name dwm -ErrorAction SilentlyContinue
if ($dwm) {
    Write-Host "dwm.exe is running (PID: $($dwm.Id))"
} else {
    Write-Host "WARNING: dwm.exe is NOT running"
}
```

**分析思路**：

1. 检查 DWM 服务和进程：
   - 正常：UxSms 服务运行中，dwm.exe 进程存在
   - 异常：UxSms 未运行或 dwm.exe 不存在 → **根因**：DWM 未启动，窗口透明效果和复合渲染不可用，**严重程度**：Warning

### Step 5: 检查 DPI 缩放

**数据采集**：

> 采集目标：获取系统 DPI 缩放设置

```powershell
# Check system-level DPI settings
$logPixels = (Get-ItemProperty -Path 'HKCU:\Control Panel\Desktop' -Name 'LogPixels' -ErrorAction SilentlyContinue).LogPixels
$dpiScaling = (Get-ItemProperty -Path 'HKCU:\Control Panel\Desktop\WindowMetrics' -Name 'AppliedDPI' -ErrorAction SilentlyContinue).AppliedDPI
Write-Host "LogPixels: $logPixels (96=100%, 120=125%, 144=150%, 192=200%)"
Write-Host "AppliedDPI: $dpiScaling"
# Check per-monitor DPI settings
$perMonitorDPI = (Get-ItemProperty -Path 'HKCU:\Control Panel\Desktop' -Name 'DpiScalingVer' -ErrorAction SilentlyContinue).DpiScalingVer
Write-Host "DpiScalingVer: $perMonitorDPI"
# Check DPI awareness settings
$dpiAware = (Get-ItemProperty -Path 'HKCU:\Control Panel\Desktop' -Name 'Win8DpiScaling' -ErrorAction SilentlyContinue).Win8DpiScaling
Write-Host "Win8DpiScaling: $dpiAware (0=DPI-aware apps, 1=use display scaling)"
```

**分析思路**：

1. 检查 DPI 缩放比例：
   - 正常：96 DPI (100%) 或合理的缩放比例
   - 异常：异常高或异常低的 DPI 值 → **根因**：DPI 缩放配置异常，导致应用界面模糊或元素大小不对，**严重程度**：Info

### Step 6: 检查 Console 会话状态

**数据采集**：

> 采集目标：获取所有 RD 会话信息，重点关注 Console 会话状态

```powershell
query session
```

**分析思路**：

1. 检查 Console 会话状态：
   - 正常：Console 会话 State 为 Connected 或 Active
   - 异常：Console 会话 State 非 Connected/Active（如 Disc/Down），或找不到 Console 会话 → **根因**：控制台会话状态异常，可能导致 VNC 连接黑屏或系统无响应（ConsoleSessionStatusError），**严重程度**：Warning

> Console 会话是 Windows 物理控制台会话（Session 0 或 1）。如果其状态异常，可能导致 VNC 连接黑屏或系统无响应。

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Step 1 Shell 配置异常且 C:\ 权限有问题 | → [identity-permission.md](references/identity-permission.md) |
| 条件跳转 | Explorer 崩溃由可疑程序引起 | → [security-malware.md](references/security-malware.md) |
| 链式后继 | 本文件未确认根因 | → [desktop-app.md](references/desktop-app.md) |

## 修复建议

### 根因: 登录 Shell 配置异常

**修复操作**：

```powershell
# Restore default Shell configuration
Set-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon' -Name 'Shell' -Value 'explorer.exe'
# Remove user-level Shell override (if exists)
Remove-ItemProperty -Path 'HKCU:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon' -Name 'Shell' -ErrorAction SilentlyContinue
```

**验证方法**：

```powershell
(Get-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon' -Name 'Shell').Shell
```

预期结果：返回 `explorer.exe`，注销重新登录后桌面正常显示

**风险说明**：如果 Shell 被故意修改为其他程序（如 kiosk 模式），恢复前确认原始意图。

### 根因: Explorer.exe 未启动

**修复操作**：

```powershell
# Manually start Explorer.exe
Start-Process explorer.exe
```

**验证方法**：

```powershell
Get-Process -Name explorer -ErrorAction SilentlyContinue | Select-Object Id, ProcessName | Format-Table -AutoSize
```

预期结果：explorer.exe 进程存在，桌面和任务栏正常显示

**风险说明**：如果 Explorer 启动后立即崩溃，可能是第三方 Shell 扩展导致，可尝试在安全模式下排查。

### 根因: DWM 未启动

**修复操作**：

```powershell
# Start DWM service
Set-Service -Name UxSms -StartupType Automatic
Start-Service UxSms
```

**验证方法**：

```powershell
Get-Service UxSms | Select-Object Name, Status | Format-Table -AutoSize
Get-Process dwm -ErrorAction SilentlyContinue
```

预期结果：UxSms 服务运行中，dwm.exe 进程存在

**风险说明**：在 Windows Server Core 版本中 DWM 可能不可用，这是正常行为。

### 根因: Console 会话状态异常（ConsoleSessionStatusError）

**修复操作**：

1. 重启远程桌面服务：
   ```powershell
   Restart-Service TermService -Force
   ```
2. 如果问题持续，检查是否安装了第三方远程控制软件（如 VNC Server）并确认其未占用 Console 会话
3. 必要时重启实例

**验证方法**：

```powershell
query session
```

预期结果：Console 会话 State 为 Active 或 Connected
