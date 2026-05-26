# Cloud MetaServer 诊断

## 目录

- [Cloud MetaServer 诊断](#cloud-metaserver-诊断)
  - [目录](#目录)
  - [功能说明](#功能说明)
  - [步骤选取指引](#步骤选取指引)
  - [诊断步骤](#诊断步骤)
    - [Step 1: 防火墙拦截元数据服务检查](#step-1-防火墙拦截元数据服务检查)
    - [Step 2: 元数据服务可达性](#step-2-元数据服务可达性)
    - [Step 3: NTP 服务器分配](#step-3-ntp-服务器分配)
    - [Step 4: KMS 激活服务器可达性](#step-4-kms-激活服务器可达性)
    - [Step 5: WSUS 更新服务器可达性](#step-5-wsus-更新服务器可达性)
    - [Step 6: 主机名一致性](#step-6-主机名一致性)
  - [交叉引用](#交叉引用)
  - [修复建议](#修复建议)
    - [根因：防火墙拦截元数据服务](#根因防火墙拦截元数据服务)
    - [根因：元数据服务不可达](#根因元数据服务不可达)
    - [根因：NTP 服务器未分配](#根因ntp-服务器未分配)
    - [根因：KMS 激活服务器不可达](#根因kms-激活服务器不可达)
    - [根因：WSUS 更新服务器不可达](#根因wsus-更新服务器不可达)
    - [根因：主机名不一致](#根因主机名不一致)

## 功能说明

诊断 Windows 云平台元数据服务可达性（100.100.100.200）、防火墙是否拦截元数据服务、NTP 时间服务器分配、KMS 激活服务器可达性、WSUS 更新服务器可达性以及主机名与元数据一致性。覆盖 6 个诊断步骤。

**输入**：用户问题描述（必选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 云助手无法工作、自动化运维失败、元数据获取失败 | Step 1 (防火墙拦截检查) → Step 2 (元数据服务可达性) |
| 系统时间不准确、时间漂移导致认证失败 | Step 2 (元数据服务可达性) → Step 3 (NTP 服务器分配) |
| Windows 激活失败、激活报错 | Step 2 (元数据服务可达性) → Step 4 (KMS 服务器可达性) |
| 无法通过 WSUS 安装更新、更新检查失败 | Step 2 (元数据服务可达性) → Step 5 (WSUS 服务器可达性) |
| 监控告警中主机名与控制台不一致、主机名设置后未生效 | Step 6 (主机名一致性) |
| 云平台服务全面异常 | Step 1 → Step 2 → Step 3 → Step 4 → Step 5 → Step 6 |

## 诊断步骤

### Step 1: 防火墙拦截元数据服务检查

**数据采集**：

> 采集目标：Windows 防火墙中是否存在针对元数据服务地址 100.100.100.200 的阻止规则

```powershell
# Check if any firewall rules block 100.100.100.200
Get-NetFirewallRule -Enabled True -Action Block -ErrorAction SilentlyContinue | ForEach-Object {
    $addrFilter = Get-NetFirewallAddressFilter -AssociatedNetFirewallRule $_ -ErrorAction SilentlyContinue
    if ($addrFilter.RemoteAddress -match '100\.100\.100\.200') {
        [PSCustomObject]@{
            DisplayName    = $_.DisplayName
            Name           = $_.Name
            Direction      = $_.Direction
            Action         = $_.Action
            RemoteAddress  = $addrFilter.RemoteAddress
        }
    }
} | Format-Table -AutoSize
```

**分析思路**：

1. 检查是否存在阻止元数据服务地址的防火墙规则：
   - 无输出 → 正常，防火墙未拦截元数据服务
   - 存在阻止规则（输出中包含 100.100.100.200 的 Block 规则）→ **根因**：防火墙规则拦截了元数据服务地址 100.100.100.200，导致元数据服务不可达，**严重程度**：Warning

> 如果需要进一步检查防火墙完整配置，参见 → [networking-firewall.md](references/networking-firewall.md)

### Step 2: 元数据服务可达性

**数据采集**：

> 采集目标：元数据服务端点 100.100.100.200 的网络连通性和 HTTP 响应状态

```powershell
# Test metadata service network connectivity
Test-NetConnection -ComputerName 100.100.100.200 -Port 80 -WarningAction SilentlyContinue | Select-Object ComputerName, RemotePort, TcpTestSucceeded, PingSucceeded | Format-Table -AutoSize

# Try to fetch metadata (instance ID) to verify HTTP service availability
try {
    $response = Invoke-WebRequest -Uri "http://100.100.100.200/latest/meta-data/instance-id" -TimeoutSec 5 -UseBasicParsing -ErrorAction Stop
    [PSCustomObject]@{
        StatusCode = $response.StatusCode
        InstanceId = $response.Content
    } | Format-List
} catch {
    [PSCustomObject]@{
        Error = $_.Exception.Message
    } | Format-List
}
```

**分析思路**：

1. 检查 TCP 连接测试结果：
   - TcpTestSucceeded 为 True → 网络层可达
   - TcpTestSucceeded 为 False → **根因**：元数据服务 100.100.100.200 不可达，所有依赖元数据的云平台服务（NTP、KMS、WSUS）均将失败，**严重程度**：Critical

2. 检查 HTTP 响应：
   - 返回 StatusCode 200 且包含实例 ID → 元数据服务正常
   - TCP 可达但 HTTP 请求失败 → **根因**：元数据服务端口可达但 HTTP 服务异常，**严重程度**：Critical

> 如果元数据服务完全不可达，请先确认防火墙规则（Step 1）和网络路由配置。参见 → [networking-tcpip.md](references/networking-tcpip.md)（检查路由表中 100.100.100.200 的路由）

### Step 3: NTP 服务器分配

**数据采集**：

> 采集目标：从元数据服务获取的 NTP 服务器地址、本地 Windows Time 服务配置的 NTP 服务器

```powershell
# Get NTP server configuration from metadata
try {
    $ntpMeta = (Invoke-WebRequest -Uri "http://100.100.100.200/latest/meta-data/ntp-server" -TimeoutSec 5 -UseBasicParsing -ErrorAction Stop).Content
    [PSCustomObject]@{ MetadataNtpServer = $ntpMeta } | Format-List
} catch {
    Write-Host "Failed to get NTP server from metadata: $($_.Exception.Message)"
}

# Local NTP configuration (registry W32Time\Parameters)
$ntpConfig = Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\W32Time\Parameters" -ErrorAction SilentlyContinue
[PSCustomObject]@{
    NtpServer = $ntpConfig.NtpServer
    Type      = $ntpConfig.Type
} | Format-List

# Windows Time 服务状态
Get-Service W32Time | Select-Object Name, Status, StartType | Format-Table -AutoSize

# Current time sync status and time source
w32tm /query /status 2>&1
w32tm /query /peers 2>&1
```

**分析思路**：

1. 检查元数据中 NTP 服务器是否分配：
   - 返回有效的 NTP 服务器地址（如 time.cloud.aliyuncs.com）→ 正常
   - 返回空或请求失败 → **根因**：元数据服务未分配 NTP 服务器，系统时间可能无法自动同步，**严重程度**：Warning

2. 检查本地 NTP 配置：
   - NtpServer 包含有效服务器地址 → 正常
   - NtpServer 为空 → **根因**：本地未配置 NTP 服务器，系统时间可能漂移导致 Kerberos 认证、证书验证等依赖时间的服务异常，**严重程度**：Warning
   - 检查 Type 值的含义：
     - `NTP`：使用 NtpServer 中配置的外部时间源（独立服务器或工作组场景）
     - `NT5DS`：从域层级同步时间（域成员默认）
     - `NoSync`：不与任何时间源同步
     - `AllSync`：同时使用域层级和手动配置的时间源
   - 如果 Type 为 NoSync → 时间同步被显式禁用，需修改

3. 检查 Windows Time 服务状态：
   - 服务运行中（Running）且 StartType 为 Automatic → 正常
   - 服务未运行 → 时间同步无法工作，应结合 NTP 配置一并修复
   - StartType 为 Manual 或 Disabled → 系统重启后时间服务不会自动启动，可能导致时间漂移

4. 检查 w32tm /query /status 输出：
   - Source 显示有效时间源 → 正常
   - Source 显示 "Local CMOS Clock" 或 "Free-running System Clock" → 系统在使用本地时钟而非网络时间源，时间可能漂移
   - Last Successful Sync Time 距当前时间过久（超过 24 小时）→ 时间同步可能存在问题

### Step 4: KMS 激活服务器可达性

**数据采集**：

> 采集目标：KMS 激活服务器地址及网络连通性（端口 1688）

```powershell
# Get KMS server address from metadata
try {
    $kmsMeta = (Invoke-WebRequest -Uri "http://100.100.100.200/latest/meta-data/kms-server" -TimeoutSec 5 -UseBasicParsing -ErrorAction Stop).Content
    [PSCustomObject]@{ MetadataKmsServer = $kmsMeta } | Format-List
} catch {
    Write-Host "Failed to get KMS server from metadata: $($_.Exception.Message)"
}

# Current system KMS server configuration (registry SoftwareProtectionPlatform)
$kmsReg = Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\SoftwareProtectionPlatform" -ErrorAction SilentlyContinue
[PSCustomObject]@{
    KeyManagementServiceName = $kmsReg.KeyManagementServiceName
    KeyManagementServicePort = $kmsReg.KeyManagementServicePort
} | Format-List

# Current license status summary
cscript //nologo "$env:windir\system32\slmgr.vbs" /dli

# Test KMS server connectivity (default port 1688)
$kmsHost = if ($kmsReg.KeyManagementServiceName) { $kmsReg.KeyManagementServiceName } else { "kms.cloud.aliyuncs.com" }
$kmsPort = if ($kmsReg.KeyManagementServicePort) { $kmsReg.KeyManagementServicePort } else { 1688 }
Test-NetConnection -ComputerName $kmsHost -Port $kmsPort -WarningAction SilentlyContinue | Select-Object ComputerName, RemotePort, TcpTestSucceeded | Format-Table -AutoSize
```

**分析思路**：

1. 检查 KMS 服务器地址是否配置：
   - 元数据或注册表中有 KMS 服务器地址 → 正常
   - 均为空 → **根因**：未配置 KMS 激活服务器，Windows 无法完成自动激活，**严重程度**：Warning

2. 检查 slmgr /dli 许可证状态：
   - License Status: Licensed → 激活正常
   - License Status: Notification → 激活宽限期已过，需尽快激活
   - License Status: Initial grace period → 处于初始宽限期，尚未激活
   - 输出中 KMS machine name 为空 → 未配置 KMS 服务器
   - 输出中 Remaining Windows rearm count 为 0 → 重置激活次数已耗尽

3. 检查 KMS 服务器连通性（TCP 端口 1688）：
   - TcpTestSucceeded 为 True → KMS 服务器可达
   - TcpTestSucceeded 为 False → **根因**：KMS 激活服务器不可达（端口 1688），Windows 激活将失败，**严重程度**：Warning

> 如果怀疑防火墙阻止 KMS 通信，参见 → [networking-firewall.md](references/networking-firewall.md)（检查出站 TCP 1688 端口规则）
> 如果需要深入排查 Windows 激活问题，参见 → [system-activation.md](references/system-activation.md)

### Step 5: WSUS 更新服务器可达性

**数据采集**：

> 采集目标：WSUS 更新服务器地址及 HTTP 连通性

```powershell
# Get WSUS server address from metadata
try {
    $wsusMeta = (Invoke-WebRequest -Uri "http://100.100.100.200/latest/meta-data/wsus-server" -TimeoutSec 5 -UseBasicParsing -ErrorAction Stop).Content
    [PSCustomObject]@{ MetadataWsusServer = $wsusMeta } | Format-List
} catch {
    Write-Host "Failed to get WSUS server from metadata: $($_.Exception.Message)"
}

# WSUS server configured in registry
$wuPolicy = Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -ErrorAction SilentlyContinue
$wuAU = Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -ErrorAction SilentlyContinue
[PSCustomObject]@{
    WUServer       = $wuPolicy.WUServer
    WUStatusServer = $wuPolicy.WUStatusServer
    UseWUServer    = $wuAU.UseWUServer
} | Format-List

# Test WSUS server connectivity
$wsusUrl = if ($wuPolicy.WUServer) { $wuPolicy.WUServer } else { "http://update.cloud.aliyuncs.com" }
try {
    $uri = [System.Uri]$wsusUrl
    $port = if ($uri.Port -gt 0 -and $uri.Port -ne 80) { $uri.Port } else { 80 }
    Test-NetConnection -ComputerName $uri.Host -Port $port -WarningAction SilentlyContinue | Select-Object ComputerName, RemotePort, TcpTestSucceeded | Format-Table -AutoSize
} catch {
    Write-Host "Failed to parse WSUS URL: $($_.Exception.Message)"
}
```

**分析思路**：

1. 检查 WSUS 服务器地址是否配置：
   - 元数据或注册表中有 WSUS 服务器地址 → 正常
   - 均为空 → 系统将使用 Microsoft Update 在线更新（非错误，但如果用户期望使用内网更新则需配置）

2. 检查 UseWUServer 注册表值：
   - UseWUServer = 1 → WSUS 配置生效，客户端将使用 WUServer 指定的服务器
   - UseWUServer = 0 或未配置 → 即使 WUServer 已填写，客户端仍然连接 Microsoft Update 而非 WSUS 服务器
   - 如果 WUServer 已配置但 UseWUServer ≠ 1 → **根因**：WSUS 服务器地址已配置但未启用，需将 UseWUServer 设为 1 才能生效，**严重程度**：Warning

3. 检查 WSUS 服务器连通性：
   - TcpTestSucceeded 为 True → WSUS 服务器可达
   - TcpTestSucceeded 为 False → **根因**：WSUS 更新服务器不可达，Windows Update 无法从内网服务器获取更新，**严重程度**：Warning

> 如果需要深入排查 Windows Update 问题，参见 → [system-update.md](references/system-update.md)

### Step 6: 主机名一致性

**数据采集**：

> 采集目标：系统当前主机名与元数据服务中记录的主机名

```powershell
# Current system hostname
$osHostname = (Get-CimInstance Win32_OperatingSystem).CSName

# Get hostname from metadata
try {
    $metaHostname = (Invoke-WebRequest -Uri "http://100.100.100.200/latest/meta-data/hostname" -TimeoutSec 5 -UseBasicParsing -ErrorAction Stop).Content
} catch {
    $metaHostname = "(Fetch failed: $($_.Exception.Message))"
}

[PSCustomObject]@{
    SystemHostname   = $osHostname
    MetadataHostname = $metaHostname
    Match            = ($osHostname -eq $metaHostname)
} | Format-List
```

**分析思路**：

1. 比较系统主机名与元数据主机名：
   - 两者一致（Match 为 True）→ 正常
   - 两者不一致（Match 为 False）→ **根因**：系统主机名与元数据服务中记录的主机名不一致，可能导致监控告警、实例标识混乱，**严重程度**：Warning

2. 如果元数据获取失败：
   - 说明元数据服务不可达，回到 Step 2 检查元数据服务可达性

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | 防火墙拦截元数据服务地址 | → [networking-firewall.md](references/networking-firewall.md) |
| 条件跳转 | 元数据不可达且路由异常 | → [networking-tcpip.md](references/networking-tcpip.md) |
| 条件跳转 | KMS 端口被防火墙拦截 | → [networking-firewall.md](references/networking-firewall.md)（检查出站 TCP 1688 端口规则） |
| 条件跳转 | KMS 激活深入排查 | → [system-activation.md](references/system-activation.md) |
| 条件跳转 | WSUS 更新深入排查 | → [system-update.md](references/system-update.md) |
| 链式后继 | 本文件未确认根因 | → [networking-firewall.md](references/networking-firewall.md) |

## 修复建议

### 根因：防火墙拦截元数据服务

**修复操作**：

```powershell
# Disable firewall rules blocking 100.100.100.200 (replace with actual rule name)
# Confirm the rule name before executing the following command
Get-NetFirewallRule -Enabled True -Action Block | ForEach-Object {
    $addrFilter = Get-NetFirewallAddressFilter -AssociatedNetFirewallRule $_ -ErrorAction SilentlyContinue
    if ($addrFilter.RemoteAddress -match '100\.100\.100\.200') {
        Write-Host "Disabling rule: $($_.DisplayName)"
        Disable-NetFirewallRule -Name $_.Name
    }
}
```

**验证方法**：

```powershell
Test-NetConnection -ComputerName 100.100.100.200 -Port 80 -WarningAction SilentlyContinue | Select-Object TcpTestSucceeded
```

预期结果：TcpTestSucceeded 为 True

**风险说明**：禁用防火墙规则可能影响安全策略，请确认该规则是误加的阻止规则而非有意为之。

### 根因：元数据服务不可达

**修复操作**：

```powershell
# Check metadata service route
$route = Get-NetRoute -DestinationPrefix "100.100.100.200/32" -ErrorAction SilentlyContinue
if ($route) {
    $route | Select-Object DestinationPrefix, NextHop | Format-Table -AutoSize
    Write-Host "Route to metadata service exists"
} else {
    Write-Host "Route to metadata service is missing"
}

# If route is missing, add route to metadata service via default gateway
$gateway = (Get-NetRoute -DestinationPrefix 0.0.0.0/0 -ErrorAction SilentlyContinue | Select-Object -First 1).NextHop
if ($gateway) {
    New-NetRoute -DestinationPrefix "100.100.100.200/32" -NextHop $gateway -PolicyStore ActiveStore -ErrorAction SilentlyContinue | Out-Null
    Write-Host "Route added: 100.100.100.200/32 via $gateway"
}
```

**验证方法**：

```powershell
Invoke-WebRequest -Uri "http://100.100.100.200/latest/meta-data/instance-id" -TimeoutSec 5 -UseBasicParsing
```

预期结果：返回实例 ID（如 i-bp1234567890abcdef）

**风险说明**：添加路由为临时操作，系统重启后失效。如需永久生效，需将路由配置加入启动脚本或调整网络配置。

### 根因：NTP 服务器未分配

**修复操作**：

```powershell
# Configure Alibaba Cloud NTP server
w32tm /config /manualpeerlist:"time.cloud.aliyuncs.com" /syncfromflags:manual /reliable:yes /update

# Restart time service
Restart-Service W32Time

# Force sync
w32tm /resync
```

**验证方法**：

```powershell
w32tm /query /status
```

预期结果：Source 显示 time.cloud.aliyuncs.com，Last Successful Sync Time 为最近时间

**风险说明**：修改 NTP 配置会立即触发时间同步，如果当前系统时间偏差较大，突然跳变可能影响正在运行的依赖时间戳的应用。

### 根因：KMS 激活服务器不可达

**修复操作**：

```powershell
# Configure KMS server
slmgr /skms kms.cloud.aliyuncs.com:1688

# Try to activate
slmgr /ato
```

**验证方法**：

```powershell
slmgr /dlv
```

预期结果：License Status 显示 Licensed

**风险说明**：slmgr /ato 会尝试在线激活，如果 KMS 服务器仍不可达则激活会失败。请先确保网络连通性。

### 根因：WSUS 更新服务器不可达

**修复操作**：

```powershell
# Option 1: Remove WSUS configuration, revert to Microsoft Update
Remove-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -Name WUServer -ErrorAction SilentlyContinue
Remove-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -Name WUStatusServer -ErrorAction SilentlyContinue
Remove-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Name UseWUServer -ErrorAction SilentlyContinue
Restart-Service wuauserv
```

**验证方法**：

```powershell
# Check Windows Update service status
Get-Service wuauserv | Select-Object Name, Status | Format-Table -AutoSize

# Manually trigger update check
(New-Object -ComObject Microsoft.Update.AutoUpdate).DetectNow()
```

预期结果：wuauserv 服务运行中，更新检查不再报错

**风险说明**：清除 WSUS 配置后系统将直接从 Microsoft Update 在线获取更新，如果网络环境限制外网访问则需保留 WSUS 配置并修复 WSUS 服务器连通性。如果 WSUS 配置由组策略下发，手动修改注册表会在下次 GPO 刷新时被覆盖。

### 根因：主机名不一致

**修复操作**：

```powershell
# Option 1: Rename system hostname to match metadata (replace with actual metadata hostname)
# Rename-Computer -NewName "<hostname-from-metadata>" -Restart -Force

# Option 2: Modify instance hostname via ECS console to match system hostname (requires cloud console operation)
```

**验证方法**：

```powershell
$osName = (Get-CimInstance Win32_OperatingSystem).CSName
$metaName = (Invoke-WebRequest -Uri "http://100.100.100.200/latest/meta-data/hostname" -TimeoutSec 5 -UseBasicParsing).Content
[PSCustomObject]@{ SystemHostname=$osName; MetadataHostname=$metaName; Match=($osName -eq $metaName) } | Format-List
```

预期结果：Match 为 True

**风险说明**：Rename-Computer 需要重启系统才能生效，会中断当前所有连接和服务。如果服务器加入了域，修改主机名需要域管理员权限且可能影响域服务。
