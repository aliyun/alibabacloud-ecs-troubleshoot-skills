# RDP 服务诊断

## 目录

- [RDP 服务诊断](#rdp-服务诊断)
  - [目录](#目录)
  - [功能说明](#功能说明)
  - [步骤选取指引](#步骤选取指引)
  - [诊断步骤](#诊断步骤)
    - [Step 1: TermService 服务状态检查](#step-1-termservice-服务状态检查)
    - [Step 2: RDP 监听器（WinStation）与端口检查](#step-2-rdp-监听器winstation与端口检查)
    - [Step 3: RDP 启用状态与组策略检查](#step-3-rdp-启用状态与组策略检查)
    - [Step 4: UMBus 设备枚举检查](#step-4-umbus-设备枚举检查)
  - [交叉引用](#交叉引用)
  - [修复建议](#修复建议)
    - [根因：TermService 服务停止 / 被禁用](#根因termservice-服务停止--被禁用)
    - [根因：TermService 依赖服务异常](#根因termservice-依赖服务异常)
    - [根因：RDP 监听器配置丢失](#根因rdp-监听器配置丢失)
    - [根因：RDP 监听端口已更改为非常规端口](#根因rdp-监听端口已更改为非常规端口)
    - [根因：RDP 监听器未运行](#根因rdp-监听器未运行)
    - [根因：第三方远程组件](#根因第三方远程组件)
    - [根因：并发连接数被限制（MaxInstanceCount）](#根因并发连接数被限制maxinstancecount)
    - [根因：WinStation 注册表读权限异常](#根因winstation-注册表读权限异常)
    - [根因：RDP 端口被其他进程占用](#根因rdp-端口被其他进程占用)
    - [根因：远程桌面已禁用（通过注册表）](#根因远程桌面已禁用通过注册表)
    - [根因：组策略禁止远程桌面连接 / 组策略覆盖本地配置](#根因组策略禁止远程桌面连接--组策略覆盖本地配置)
    - [根因：组策略覆盖 RDP 安全配置](#根因组策略覆盖-rdp-安全配置)
    - [根因：第三方 WinStation 与 RDP-Tcp 冲突导致闪退](#根因第三方-winstation-与-rdp-tcp-冲突导致闪退)
    - [根因：UMBus 设备枚举异常](#根因umbus-设备枚举异常)
  - [Gotchas（易错点）](#gotchas易错点)

## 功能说明

诊断远程桌面服务（TermService）及其相关组件问题。覆盖 TermService 服务状态与依赖服务、RDP 监听器（WinStation）配置与端口监听（含端口冲突进程检测、Session Listener 状态、第三方 Station 检测、MaxInstanceCount、WinStation 注册表 ACL）、RDP 启用状态与组策略（含 SecurityLayer/UserAuthentication/MaxInstanceCount 策略覆盖检查）、UMBus 设备枚举等 4 个诊断步骤。

**输入**：用户问题描述（必选）、RDP 连接错误信息（可选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| RDP 连接直接被拒绝 | Step 1 (TermService 服务) → Step 2 (WinStation 与端口) |
| 提示 "This computer can't connect" | Step 2 (WinStation 与端口) → Step 3 (启用状态与组策略) |
| RDP 端口被占用 | Step 2 (WinStation 与端口) → Step 1 (服务状态) |
| RDP 连接超时 | Step 2 (WinStation 与端口) → [networking-firewall.md](references/networking-firewall.md)（检查入站 TCP 端口规则） |
| 提示 "Remote Desktop has been disabled" 或组策略禁止远程桌面 | Step 3 (启用状态与组策略) |
| 多用户连接时提示超过最大连接数 | Step 2 (WinStation 与端口，检查 MaxInstanceCount) → Step 3 (组策略 MaxInstanceCount) |
| 安装第三方远程软件后 RDP 异常 | Step 2 (WinStation 与端口，检查第三方 Station) |
| mstsc 远程连接闪退（连上后立即断开） | Step 2 (WinStation 与端口，重点检查第三方 WinStation 与 RDP-Tcp 端口/配置冲突) |
| RDP 连接后设备映射失败 | Step 4 (UMBus 设备) |

## 诊断步骤

### Step 1: TermService 服务状态检查

**数据采集**：

> 采集目标：获取 TermService 服务及其依赖服务的运行状态和启动类型

```powershell
Get-Service -Name TermService | Select-Object Name, Status, StartType | Format-Table -AutoSize
Get-Service -Name TermService -RequiredServices | Select-Object Name, Status, StartType | Format-Table -AutoSize
```

**分析思路**：

1. 检查 TermService 服务状态：
   - 正常：服务正在运行，启动类型为自动
   - 异常：服务已停止 → **根因**：TermService 服务已停止，远程桌面无法连接，**严重程度**：Critical
   - 异常：服务被禁用 → **根因**：TermService 服务被禁用，**严重程度**：Critical

2. 检查依赖服务状态：
   - 正常：所有依赖服务正在运行
   - 异常：依赖服务未运行 → **根因**：TermService 依赖服务异常（如 RpcSs），**严重程度**：Critical

### Step 2: RDP 监听器（WinStation）与端口检查

> ⚠️ **MUST**: You must iterate **ALL** sub-keys under `..\WinStations\` (excluding `Console`). **DO NOT** stop at `RDP-Tcp` or assume a single listener exists. Multi-listener configurations (e.g., `RDP-Tcp-3389`) are common.

**数据采集**：

> 采集目标：枚举所有 WinStation 的注册表配置（含 WdName、MaxInstanceCount），检查各端口的监听状态和占用进程，检查 Session Listener 状态和 WinStation 注册表 ACL，采集 TerminalServices 会话层关键事件日志

```powershell
# Enumerate all WinStations (excluding Console), get configuration and port listening status
$winStationsPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations"
$stations = Get-ChildItem -Path $winStationsPath -ErrorAction SilentlyContinue |
    Where-Object { $_.PSChildName -ne "Console" }

# 1. WinStation registry configuration (including WdName, MaxInstanceCount)
$stations | ForEach-Object {
    $props = Get-ItemProperty -Path $_.PSPath -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        StationName       = $_.PSChildName
        PortNumber        = $props.PortNumber
        fEnableWinStation = $props.fEnableWinStation
        WdName            = $props.WdName
        MaxInstanceCount  = $props.MaxInstanceCount
    }
} | Format-Table -AutoSize

# 2. Port consistency check for EACH WinStation (configured port vs actual listening port)
$stations | ForEach-Object {
    $props = Get-ItemProperty -Path $_.PSPath -ErrorAction SilentlyContinue
    $configuredPort = $props.PortNumber
    $stationName = $_.PSChildName
    if ($configuredPort) {
        $listening = Get-NetTCPConnection -LocalPort $configuredPort -State Listen -ErrorAction SilentlyContinue
        $actualService = if ($listening) {
            ($listening | ForEach-Object {
                (Get-CimInstance Win32_Service -Filter "ProcessId=$($_.OwningProcess)" -ErrorAction SilentlyContinue).Name
            }) -join ','
        } else { $null }
        $isListening = [bool]$listening
        [PSCustomObject]@{
            StationName    = $stationName
            ConfiguredPort = $configuredPort
            IsListening    = $isListening
            ServiceName    = $actualService
        }
    }
} | Format-Table -AutoSize

# 3. Session Listener status (qwinsta)
qwinsta 2>&1

# 4. WinStation registry ACL check (BUILTIN\Users read permission)
$stations | ForEach-Object {
    $acl = Get-Acl -Path $_.PSPath -ErrorAction SilentlyContinue
    $usersAccess = $acl.Access | Where-Object { $_.IdentityReference -match 'BUILTIN\\Users' }
    [PSCustomObject]@{
        StationName = $_.PSChildName
        UsersAccess = if ($usersAccess) { $usersAccess.RegistryRights } else { 'NO_ACCESS' }
    }
} | Format-Table -AutoSize

# 5. TerminalServices session-layer event log (conflict / initialization failure)
Get-WinEvent -LogName 'Microsoft-Windows-TerminalServices-RemoteConnectionManager/Operational' -ErrorAction SilentlyContinue |
    Where-Object { $_.Id -in @(261, 1035, 1036, 1042, 1103) } |
    Select-Object TimeCreated, Id, Message -First 20 | Format-List

# 6. Enumeration completeness verification
Write-Output "Total WinStations found (excluding Console): $($stations.Count)"
```

**分析思路**：

1. 检查是否存在 WinStation：
   - 正常：至少存在一个 WinStation（通常为 RDP-Tcp）
   - 异常：无任何 WinStation → **根因**：RDP 监听器配置丢失，**严重程度**：Critical

2. 检查 Session Listener 状态（qwinsta 输出）：
   - 正常：存在 State = Listen 的 rdp-tcp 会话
   - 异常：无任何 Listen 状态的会话 → **根因**：RDP 监听器未运行，无法接受新的 RDP 连接，**严重程度**：Critical

3. 对每个 WinStation 检查配置端口与实际监听的一致性：
   - 正常：配置端口有活跃的 TCP 监听，且 ServiceName = TermService
   - 异常：配置端口无监听 → **根因**：RDP 端口未被监听（标注 StationName 和 PortNumber），**严重程度**：Critical
   - 异常：配置端口被非 TermService 进程占用 → **根因**：RDP 端口被其他进程占用（标注 StationName、PortNumber 和 ServiceName），**严重程度**：Critical
   - 异常：所有 WinStation 均未配置 3389 但 3389 正在监听 → 可能存在残留监听器或配置不一致，**严重程度**：Warning

4. 全局检查：是否有任何 WinStation 配置了标准端口 3389：
   - 正常：至少一个 WinStation 的 PortNumber = 3389
   - 异常：所有 WinStation 均为非标准端口 → **根因**：RDP 监听端口已更改为非常规端口，客户端需指定端口才能连接，**严重程度**：Warning

5. 对每个 WinStation 检查是否被禁用：
   - 正常：fEnableWinStation = 1 或未设置
   - 异常：fEnableWinStation = 0 → **根因**：WinStation 被禁用（标注 StationName），**严重程度**：Warning

6. 对每个 WinStation 检查 WdName 是否为 Microsoft 标准组件：
   - 正常：WdName 包含 "Microsoft"（如 "Microsoft RDP"）或 WdName 为 "RDPWD"
   - 异常：WdName 不包含 "Microsoft" 且不为 "RDPWD" → **根因**：第三方远程组件（标注 StationName 和 WdName），可能与标准 RDP 冲突，**严重程度**：Warning

7. 对每个 WinStation 检查 MaxInstanceCount：
   - 正常：MaxInstanceCount 未设置或值较大（如 0xFFFFFFFF）
   - 异常：MaxInstanceCount 值过低（如 1 或 2）→ **根因**：并发连接数被限制（标注 StationName 和当前值），多用户场景下可能无法连接，**严重程度**：Warning

8. 对每个 WinStation 检查注册表 ACL：
   - 正常：BUILTIN\Users 具有读取权限（ReadKey）
   - 异常：BUILTIN\Users 无读取权限 → **根因**：WinStation 注册表读权限异常（标注 StationName），用户无法查询 RDP 配置，**严重程度**：Warning

9. 检查第三方 WinStation 与 RDP-Tcp 端口冲突（mstsc 闪退场景）：
   - 异常：存在多个非 Console 的 WinStation 配置了相同的 PortNumber（如第三方 Station 与 RDP-Tcp 均为 3389） → **根因**：第三方 WinStation 与 RDP-Tcp 端口冲突导致闪退（标注冲突的 StationName 和端口），第三方 WinStation 抢占监听导致 mstsc 会话初始化失败立即断开，**严重程度**：Critical
   - 异常：RDP-Tcp 的 WdName 被第三方驱动覆盖（WdName 非 "RDPWD" 且非 "Microsoft" 前缀），同时用户报告 mstsc 闪退 → **根因**：第三方驱动覆盖 RDP-Tcp 的 WdName 导致闪退（标注当前 WdName），mstsc 无法与非标准协议驱动完成会话协商，**严重程度**：Warning

> 如果端口正常监听但仍无法连接，可能是防火墙阻止，参见 → [networking-firewall.md](references/networking-firewall.md)（检查入站 TCP 端口规则）

10. 检查 TerminalServices 会话层事件日志（Event ID 261/1035/1036/1042/1103）：
    - 正常：日志中无上述 ID 的错误事件
    - 异常：出现 Event 261（监听器失败）、Event 1035/1036（连接请求拒绝）、Event 1042（层已失败）、Event 1103（连接终止），并关联到对应的 WinStation 或端口冲突根因，**严重程度**：Warning

### Step 3: RDP 启用状态与组策略检查

**数据采集**：

> 采集目标：检查 RDP 启用状态的本地注册表配置和组策略配置（含 SecurityLayer、UserAuthentication、MaxInstanceCount 等策略值）

```powershell
# Check local registry RDP enablement status
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server" -Name "fDenyTSConnections" -ErrorAction SilentlyContinue | Select-Object fDenyTSConnections | Format-Table -AutoSize

# Check Group Policy registry values (full policy check)
$gpPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services"
Get-ItemProperty -Path $gpPath -Name fDenyTSConnections, SecurityLayer, UserAuthentication, MaxInstanceCount, fDisableCdm, fDisableClip -ErrorAction SilentlyContinue | Select-Object fDenyTSConnections, SecurityLayer, UserAuthentication, MaxInstanceCount, fDisableCdm, fDisableClip | Format-Table -AutoSize
```

**分析思路**：

1. 检查 RDP 是否通过本地注册表禁用：
   - 正常：fDenyTSConnections = 0（远程桌面已启用）
   - 异常：fDenyTSConnections = 1 → **根因**：远程桌面已禁用（通过注册表），**严重程度**：Critical

2. 检查组策略是否覆盖本地配置：
   - 正常：组策略路径无此项（未配置组策略限制）或 fDenyTSConnections = 0
   - 异常：组策略 fDenyTSConnections = 1 → **根因**：组策略禁止远程桌面连接，**严重程度**：Critical
   - 异常：组策略与本地配置冲突（组策略禁止但本地启用） → **根因**：组策略覆盖本地配置，RDP 实际被禁用，**严重程度**：Critical

3. 检查组策略是否覆盖 RDP 安全配置：
   - SecurityLayer 被策略设置 → **根因**：组策略覆盖 RDP 安全层配置（标注策略值：0=RDP Security, 1=Negotiate, 2=TLS），可能与客户端不兼容，**严重程度**：Warning
   - UserAuthentication 被策略设置 → **根因**：组策略强制 NLA（Network Level Authentication）配置（标注策略值：0=不要求, 1=要求），旧客户端可能无法连接，**严重程度**：Warning
   - MaxInstanceCount 被策略设置 → **根因**：组策略限制最大并发连接数（标注策略值），多用户场景下可能无法连接，**严重程度**：Warning
   - fDisableCdm / fDisableClip 被策略设置 → 功能限制提示（非连接性问题），**严重程度**：Info

### Step 4: UMBus 设备枚举检查

**数据采集**：

> 采集目标：检查 UMBus 设备和远程桌面相关设备的状态

```powershell
Get-PnpDevice -Class 'System' -ErrorAction SilentlyContinue | Where-Object {$_.FriendlyName -like '*UMBus*'} | Select-Object FriendlyName, Status, Class | Format-Table -AutoSize
Get-PnpDevice -ErrorAction SilentlyContinue | Where-Object {$_.FriendlyName -like '*Remote Desktop*' -or $_.FriendlyName -like '*Terminal*'} | Select-Object FriendlyName, Status, Class | Format-Table -AutoSize
```

**分析思路**：

1. 检查 UMBus 设备状态：
   - 正常：设备状态正常
   - 异常：设备状态异常 → **根因**：UMBus 设备枚举异常，RDP 连接后设备映射可能失败，**严重程度**：Warning

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 参数化引用 | 防火墙阻止 RDP 端口 | → [networking-firewall.md](references/networking-firewall.md)（检查入站 TCP 3389 端口规则） |
| 条件跳转 | Step 1 TermService 依赖服务异常 | → [networking-tcpip.md](references/networking-tcpip.md)(如依赖网络服务) |
| 链式后继 | 本文件未确认根因,用户报告 RDP 问题 | → [rdp-auth.md](references/rdp-auth.md) |

## 修复建议

### 根因：TermService 服务停止 / 被禁用

**修复操作**：

```powershell
# Start dependent services
Get-Service -Name TermService -RequiredServices | ForEach-Object {
    if ($_.Status -ne 'Running') {
        Start-Service -Name $_.Name -ErrorAction SilentlyContinue
        Write-Host "Started dependent service: $($_.Name)"
    }
}

# Enable and start TermService
Set-Service -Name TermService -StartupType Automatic
Start-Service -Name TermService
Write-Host "Started TermService"
```

**验证方法**：

```powershell
Get-Service -Name TermService | Select-Object Name, Status, StartType | Format-Table -AutoSize
Get-NetTCPConnection -LocalPort 3389 -State Listen -ErrorAction SilentlyContinue | Format-Table -AutoSize
```

预期结果：TermService Status = Running，StartType = Automatic，3389 端口处于 Listen 状态

**风险说明**：启用 TermService 会开启远程桌面功能，确保配置了强密码。如果服务因配置错误崩溃可能再次停止

---

### 根因：TermService 依赖服务异常

**修复操作**：

```powershell
# Start all dependent services of TermService (replace <ServiceName> with actual dependent service name)
Start-Service -Name "<ServiceName>"

# Start TermService
Start-Service -Name TermService
```

**验证方法**：

```powershell
Get-Service -Name TermService -RequiredServices | Select-Object Name, Status, StartType | Format-Table -AutoSize
```

预期结果：所有依赖服务 Status = Running

**风险说明**：依赖服务可能有自己的依赖，需要按顺序启动

---

### 根因：RDP 监听器配置丢失

**修复操作**：

```powershell
# Recreate RDP-Tcp listener registry key
New-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" -Force

# Set default port
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" -Name "PortNumber" -Value 3389

# Restart TermService
Restart-Service -Name TermService -Force
```

**验证方法**：

```powershell
Get-ChildItem -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations" |
    Where-Object { $_.PSChildName -ne "Console" } |
    ForEach-Object { [PSCustomObject]@{ Station = $_.PSChildName; PortNumber = (Get-ItemProperty $_.PSPath).PortNumber } }
```

预期结果：至少存在 RDP-Tcp，PortNumber = 3389

**风险说明**：重新创建监听器配置会中断现有 RDP 连接

---

### 根因：RDP 监听端口已更改为非常规端口

**修复操作**：

```powershell
# Change RDP port back to default 3389 for specified WinStation (replace <StationName> with actual station name)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\<StationName>" -Name "PortNumber" -Value 3389

# Restart TermService
Restart-Service -Name TermService -Force

# Add firewall rule
New-NetFirewallRule -DisplayName "Allow RDP" -Direction Inbound -Protocol TCP -LocalPort 3389 -Action Allow -Profile Any
```

**验证方法**：

```powershell
Get-ChildItem -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations" |
    Where-Object { $_.PSChildName -ne "Console" } |
    ForEach-Object { [PSCustomObject]@{ Station = $_.PSChildName; PortNumber = (Get-ItemProperty $_.PSPath).PortNumber } }
Get-NetTCPConnection -LocalPort 3389 -State Listen
```

预期结果：目标 WinStation PortNumber = 3389，端口处于监听状态

**风险说明**：更改端口会中断现有连接，如果使用了非标准端口连接方式需要更新

---

### 根因：RDP 监听器未运行

**修复操作**：

```powershell
# Check current session status
qwinsta

# Restart TermService to restore listener
Restart-Service -Name TermService -Force

# If still no listener, check underlying causes (port conflict, service errors, etc.)
Get-Service -Name TermService | Select-Object Name, Status, StartType | Format-Table -AutoSize
```

**验证方法**：

```powershell
qwinsta | Select-String -Pattern "Listen"
```

预期结果：存在 rdp-tcp 会话，State = Listen

**风险说明**：如果重启服务后仍无 Listen 会话，需排查端口冲突、WinStation 禁用、配置丢失等底层原因

---

### 根因：第三方远程组件

**修复操作**：

```powershell
# List all WinStations and their WdName
Get-ChildItem -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations" |
    Where-Object { $_.PSChildName -ne "Console" } | ForEach-Object {
        [PSCustomObject]@{
            StationName = $_.PSChildName
            WdName = (Get-ItemProperty $_.PSPath -ErrorAction SilentlyContinue).WdName
        }
    } | Format-Table -AutoSize

# If third-party Station causes conflict, disable that Station
# Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\<StationName>" -Name "fEnableWinStation" -Value 0
```

**验证方法**：

```powershell
# Confirm standard RDP-Tcp listener is normal
qwinsta | Select-String -Pattern "rdp-tcp"
```

预期结果：rdp-tcp 会话 State = Listen，第三方 Station 已禁用或卸载

**风险说明**：禁用第三方 Station 可能影响使用该协议的远程连接；建议先确认该第三方软件的用途再操作

---

### 根因：并发连接数被限制（MaxInstanceCount）

**修复操作**：

```powershell
# View current MaxInstanceCount value (replace <StationName> with actual station name)
(Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\<StationName>" -ErrorAction SilentlyContinue).MaxInstanceCount

# Remove restriction (set to unlimited)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\<StationName>" -Name "MaxInstanceCount" -Value 0xFFFFFFFF

# Restart TermService
Restart-Service -Name TermService -Force
```

**验证方法**：

```powershell
(Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\<StationName>").MaxInstanceCount
```

预期结果：MaxInstanceCount = 4294967295（0xFFFFFFFF，无限制）

**风险说明**：增加并发连接数会占用更多系统资源；如果组策略设置了 MaxInstanceCount，需同步修改组策略

---

### 根因：WinStation 注册表读权限异常

**修复操作**：

```powershell
# Add WinStations registry read permission for BUILTIN\Users
$regPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations"
$acl = Get-Acl -Path $regPath
$rule = New-Object System.Security.AccessControl.RegistryAccessRule("BUILTIN\Users", "ReadKey", "ContainerInherit,ObjectInherit", "None", "Allow")
$acl.AddAccessRule($rule)
Set-Acl -Path $regPath -AclObject $acl
Write-Host "Added WinStations registry read permission for BUILTIN\Users"
```

**验证方法**：

```powershell
(Get-Acl -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations").Access |
    Where-Object { $_.IdentityReference -match 'BUILTIN\\Users' } |
    Select-Object IdentityReference, RegistryRights, AccessControlType
```

预期结果：BUILTIN\Users 具有 ReadKey 权限，AccessControlType = Allow

**风险说明**：修改注册表权限需要管理员权限；如果权限被域策略控制，可能会在下次组策略刷新时被覆盖

---

### 根因：RDP 端口被其他进程占用

**修复操作**：

```powershell
# Find process occupying port 3389
$processId = (Get-NetTCPConnection -LocalPort 3389 -ErrorAction SilentlyContinue | Select-Object -First 1).OwningProcess
if ($processId) {
    $process = Get-Process -Id $processId -ErrorAction SilentlyContinue
    Write-Host "Process occupying port 3389: $($process.Name) (PID: $processId)"
    
    # Prompt user to confirm whether to terminate the process
    Write-Host "Recommended to terminate the process or change RDP port"
}
```

**验证方法**：

```powershell
Get-NetTCPConnection -LocalPort 3389 -ErrorAction SilentlyContinue
```

预期结果：3389 端口无其他进程占用

**风险说明**：终止其他进程可能影响系统功能，需谨慎操作

---

### 根因：远程桌面已禁用（通过注册表）

**修复操作**：

```powershell
# Enable Remote Desktop
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server" -Name "fDenyTSConnections" -Value 0
Write-Host "Remote Desktop enabled"
```

**验证方法**：

```powershell
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server" -Name "fDenyTSConnections"
```

预期结果：fDenyTSConnections = 0

**风险说明**：启用 RDP 会增加系统暴露面，确保配置了强密码和防火墙规则

---

### 根因：组策略禁止远程桌面连接 / 组策略覆盖本地配置

**修复操作**：

```powershell
# Modify Group Policy configuration
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" -Name "fDenyTSConnections" -Value 0 -ErrorAction SilentlyContinue

# Also modify local configuration
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server" -Name "fDenyTSConnections" -Value 0

# Update Group Policy
gpupdate /force
```

**验证方法**：

```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" -Name "fDenyTSConnections" -ErrorAction SilentlyContinue | Select-Object fDenyTSConnections
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server" -Name "fDenyTSConnections" | Select-Object fDenyTSConnections
```

预期结果：两处 fDenyTSConnections 均为 0

**风险说明**：如果服务器受域控管理，组策略可能会被域控覆盖，需要联系域管理员

---

### 根因：组策略覆盖 RDP 安全配置

**修复操作**：

```powershell
# View current Group Policy overrides for RDP security configuration
$gpPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services"
Get-ItemProperty -Path $gpPath -Name SecurityLayer, UserAuthentication, MaxInstanceCount -ErrorAction SilentlyContinue | Select-Object SecurityLayer, UserAuthentication, MaxInstanceCount | Format-List

# Remove unnecessary Group Policy override values (replace <PolicyName> with actual policy name)
# Remove-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services" -Name "<PolicyName>" -ErrorAction SilentlyContinue

# Update Group Policy
gpupdate /force

# Restart TermService
Restart-Service -Name TermService -Force
```

**验证方法**：

```powershell
$gpPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows NT\Terminal Services"
Get-ItemProperty -Path $gpPath -Name SecurityLayer, UserAuthentication, MaxInstanceCount -ErrorAction SilentlyContinue | Select-Object SecurityLayer, UserAuthentication, MaxInstanceCount | Format-List
```

预期结果：对应的策略值已被移除（Value 为空）或修改为期望值

**风险说明**：如果服务器加入域，组策略修改可能在下次 gpupdate 时被域控策略覆盖，需要联系域管理员在 GPMC 中修改

---

### 根因：第三方 WinStation 与 RDP-Tcp 冲突导致闪退

**修复操作**：

```powershell
# List all WinStations and identify port conflicts
$winStationsPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations"
$stations = Get-ChildItem -Path $winStationsPath | Where-Object { $_.PSChildName -ne "Console" }
$stations | ForEach-Object {
    $props = Get-ItemProperty -Path $_.PSPath -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        StationName       = $_.PSChildName
        PortNumber        = $props.PortNumber
        WdName            = $props.WdName
        fEnableWinStation = $props.fEnableWinStation
    }
} | Format-Table -AutoSize

# Disable conflicting third-party WinStation (replace <ThirdPartyStationName> with actual name)
# Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\<ThirdPartyStationName>" -Name "fEnableWinStation" -Value 0

# If RDP-Tcp WdName was overwritten, restore to standard value
# Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" -Name "WdName" -Value "RDPWD"

# Restart TermService to apply changes
# Restart-Service -Name TermService -Force
```

**验证方法**：

```powershell
qwinsta
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" | Select-Object WdName, PortNumber, fEnableWinStation
```

预期结果：rdp-tcp State = Listen，WdName = RDPWD，第三方 WinStation 已禁用或卸载，mstsc 连接不再闪退

**风险说明**：禁用第三方 WinStation 可能影响依赖该组件的远程连接方式；恢复 WdName 前建议备份注册表键 `HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp`。**执行此操作前，必须确认已通过 VNC 或其他控制台通道备份访问渠道**

---

### 根因：UMBus 设备枚举异常

**修复操作**：

```powershell
# Rescan plug and play devices
pnputil /scan-devices

# Check UMBus device status
Get-PnpDevice -Class 'System' -ErrorAction SilentlyContinue | Where-Object {$_.FriendlyName -like '*UMBus*'} | Select-Object FriendlyName, Status | Format-Table -AutoSize
```

**验证方法**：

```powershell
Get-PnpDevice -Class 'System' -ErrorAction SilentlyContinue | Where-Object {$_.FriendlyName -like '*UMBus*'} | Select-Object FriendlyName, Status | Format-Table -AutoSize
```

预期结果：所有 UMBus 设备 Status = OK

**风险说明**：如果设备仍然异常，可能需要更新或重新安装驱动

---

## Gotchas（易错点）

- **遗漏非默认 WinStation**：管理员常创建额外 WinStation（如 `RDP-Tcp-3389`）实现多端口 RDP 监听。诊断时必须完整遍历 `Get-ChildItem` 结果，不能只查 `RDP-Tcp`。
- **多监听器端口冲突**：多个 WinStation 配置相同端口时，只有一个能实际绑定监听，另一个静默失败，必须逐一核对 `IsListening` 状态。
- **脚本执行完整性**：数据采集脚本末尾会输出 `Total WinStations found`，如果数量为 0 或 1，需警惕是否存在遗漏或配置丢失。
