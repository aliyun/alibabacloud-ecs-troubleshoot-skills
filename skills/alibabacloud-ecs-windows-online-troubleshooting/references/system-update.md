# System Update 诊断

## 功能说明

诊断 Windows Update 依赖服务、WSUS 配置、更新服务器可达性、已知问题补丁和 WinHTTP 代理配置。覆盖 5 个已知问题项。

**输入**：用户问题描述（必选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 更新检查失败、报错 0x80070422 | Step 1 (Windows Update 依赖服务) |
| 更新源不正确、无法找到可用更新 | Step 2 (WSUS 配置) |
| 更新下载卡住或超时 | Step 3 (更新服务器可达性) |
| 安装特定 KB 后出现蓝屏或功能异常 | Step 4 (已知问题补丁) |

## 诊断步骤

### Step 1: Windows Update 依赖服务检查

**数据采集**：

> 采集目标：检查 Windows Update 所依赖的核心服务状态

```powershell
$services = @('wuauserv','BITS','BrokerInfrastructure','CryptSvc','swprv','Schedule','VSS','mpssvc','Winmgmt','TrustedInstaller','w32time')
Get-Service -Name $services -ErrorAction SilentlyContinue |
  Select-Object Name, DisplayName, Status, StartType |
  Format-Table -AutoSize
```

**分析思路**：

1. 检查核心依赖服务（wuauserv/CryptSvc/TrustedInstaller/BrokerInfrastructure）：
   - 正常：全部为 Running，无服务 Disabled
   - 异常：任一停止或被禁用 → **根因**：更新核心依赖服务异常（UpdateDependentServiceInvalid），**严重程度**：Critical
2. 检查辅助服务（BITS/swprv/VSS/Schedule/w32time/mpssvc/Winmgmt）：
   - 正常：Manual 且 Stopped → 这些服务按需启动，Stopped 为正常状态
   - 异常：任一被禁用 → **根因**：更新辅助服务被禁用，**严重程度**：Warning

> 关键服务说明：wuauserv=Windows Update 主服务，BITS=后台传输，CryptSvc=加密服务，TrustedInstaller=模块安装服务

### Step 2: WSUS 配置检查

**数据采集**：

> 采集目标：检查 Windows Update 服务器配置（注册表）

```powershell
Get-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate' -ErrorAction SilentlyContinue |
  Select-Object WUServer, WUStatusServer |
  Format-Table -AutoSize
Get-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU' -ErrorAction SilentlyContinue |
  Select-Object UseWUServer, NoAutoUpdate, AUOptions |
  Format-Table -AutoSize
```

**分析思路**：

1. 检查 WSUS 配置：
   - 正常：UseWUServer=1 且 WUServer/WUStatusServer 指向正确的 WSUS
   - 异常：UseWUServer=0 或 WUServer/WUStatusServer 不匹配 → **根因**：WSUS 配置错误（WUServerConfigError），**严重程度**：Warning
   - 注册表项不存在：未配置 WSUS，系统使用 Microsoft Update 默认源

### Step 3: 更新服务器可达性检查

**数据采集**：

> 采集目标：测试到 WSUS 服务器的网络连通性

```powershell
# Get configured WSUS server address
$wu = Get-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate' -ErrorAction SilentlyContinue
$server = if ($wu -and $wu.WUServer) { $wu.WUServer } else { 'http://update.cloud.aliyuncs.com' }
Write-Output "WSUS Server: $server"
# Test connectivity
$uri = [System.Uri]$server
Test-NetConnection -ComputerName $uri.Host -Port 80 -WarningAction SilentlyContinue |
  Select-Object ComputerName, RemotePort, TcpTestSucceeded |
  Format-Table -AutoSize
```

**分析思路**：

1. 检查更新服务器可达性：
   - 正常：TcpTestSucceeded=True → 更新服务器可达
   - 异常：TcpTestSucceeded=False → **根因**：更新服务器不可达（UpdateServerUnReachable），**严重程度**：Critical

### Step 4: 已知问题补丁检查

**数据采集**：

> 采集目标：检查是否安装了已知会导致问题的补丁

```powershell
$problematic = @('KB5009624','KB5009595','KB5009546','KB5009557','KB5009555','KB5014738','KB5014702','KB5014692','KB5014678','KB5060842')
Get-HotFix -ErrorAction SilentlyContinue | Where-Object { $problematic -contains $_.HotFixID } |
  Select-Object HotFixID, InstalledOn, Description |
  Format-Table -AutoSize
```

**分析思路**：

1. 检查已知问题补丁：
   - 正常：未发现问题补丁
   - 异常：发现已知问题补丁 → **根因**：已安装可能导致问题的补丁（ProblematicHotfixInstalled），**严重程度**：Warning

> 这些补丁已知会导致 RDP 断开、服务崩溃或启动失败等问题。

### Step 5: WinHTTP 代理配置检查

**数据采集**：

> 采集目标：检查 WinHTTP 系统级代理和 IE 用户级代理配置是否一致

```powershell
# WinHTTP system proxy
netsh winhttp show proxy
# IE user proxy
Get-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings' -ErrorAction SilentlyContinue |
  Select-Object ProxyEnable, ProxyServer, ProxyOverride |
  Format-Table -AutoSize
```

**分析思路**：

1. 检查 WinHTTP 代理配置：
   - 正常：WinHTTP 代理与 IE 代理一致（或均未配置）
   - 异常：IE 配置了代理但 WinHTTP 未配置 → **根因**：WinHTTP 代理不一致（WinhttpConfigError），**严重程度**：Warning

> Windows Update 使用 WinHTTP（系统级）而非 IE 代理（用户级）。如果仅在 IE 中配置了代理，Windows Update 将无法使用该代理。

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Step 3 服务器不可达且涉及防火墙阻断 | → [networking-firewall.md](references/networking-firewall.md) |
| 条件跳转 | WSUS 配置指向元数据服务 | → [cloud-metaserver.md](references/cloud-metaserver.md)（元数据 WSUS 检查） |
| 条件跳转 | Temp 文件夹权限问题导致更新失败 | → [identity-permission.md](references/identity-permission.md)（Temp 权限检查） |
| 链式后继 | 本文件未确认根因 | → [networking-firewall.md](references/networking-firewall.md) |

## 修复建议

### Fix 1: 修复依赖服务（UpdateDependentServiceInvalid）

**适用根因**：UpdateDependentServiceInvalid

```powershell
# Start critical services and set them to automatic startup
foreach ($svc in @('wuauserv','BITS','CryptSvc','TrustedInstaller')) {
  Set-Service -Name $svc -StartupType Automatic -ErrorAction SilentlyContinue
  Start-Service -Name $svc -ErrorAction SilentlyContinue
}
```

### Fix 2: 修正 WSUS 配置（WUServerConfigError）

**适用根因**：WUServerConfigError

```powershell
$wsusPath = 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate'
$auPath = 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU'
Set-ItemProperty -Path $wsusPath -Name 'WUServer' -Value 'http://update.cloud.aliyuncs.com'
Set-ItemProperty -Path $wsusPath -Name 'WUStatusServer' -Value 'http://update.cloud.aliyuncs.com'
Set-ItemProperty -Path $auPath -Name 'UseWUServer' -Value 1 -Type DWord
# Restart Windows Update service to apply the configuration
Restart-Service wuauserv -Force
```

### Fix 3: 卸载问题补丁（ProblematicHotfixInstalled）

**适用根因**：ProblematicHotfixInstalled

```powershell
# Uninstall specified KB (replace KBXXXXXXX with the actual KB number)
wusa /uninstall /kb:XXXXXXX /quiet /norestart
# Restart system after uninstallation
```

> 卸载前建议创建系统还原点或快照，以便回滚。

### Fix 4: 同步 WinHTTP 代理（WinhttpConfigError）

**适用根因**：WinhttpConfigError

```powershell
# Import IE proxy settings into WinHTTP
netsh winhttp import proxy source=ie
# Or directly set WinHTTP proxy
# netsh winhttp set proxy proxy-server="http=proxy:port;https=proxy:port"
```
