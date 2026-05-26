# RDP Licensing 诊断

## 目录

- [RDP Licensing 诊断](#rdp-licensing-诊断)
  - [目录](#目录)
  - [功能说明](#功能说明)
  - [步骤选取指引](#步骤选取指引)
  - [诊断步骤](#诊断步骤)
    - [Step 1: Remote Desktop Session Host 角色检查](#step-1-remote-desktop-session-host-角色检查)
    - [Step 2: RDS 授权模式配置检查](#step-2-rds-授权模式配置检查)
    - [Step 3: Grace Period 状态检查](#step-3-grace-period-状态检查)
    - [Step 4: 授权服务器连通性检查](#step-4-授权服务器连通性检查)
  - [交叉引用](#交叉引用)
  - [修复建议](#修复建议)
    - [根因：RDS 授权模式未配置](#根因rds-授权模式未配置)
    - [根因：RDS 120 天试用期已过期](#根因rds-120-天试用期已过期)
    - [根因：RDS CAL 已耗尽](#根因rds-cal-已耗尽)
    - [根因：RDS 授权服务器不可达](#根因rds-授权服务器不可达)
    - [根因：本地 RD Licensing 服务未运行](#根因本地-rd-licensing-服务未运行)
    - [根因：RDS 授权服务器未配置](#根因rds-授权服务器未配置)

## 功能说明

诊断 Windows 远程桌面服务（RDS）授权状态和授权模式配置问题。覆盖 RDS Session Host 角色安装状态、授权模式配置、Grace Period 过期、授权服务器连通性等 4 个已知问题项。

**输入**：用户问题描述（必选）、RDP 连接时的授权错误信息（可选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 多用户连接时提示 "licensing mode is not configured" | Step 1 (RDSH 角色) → Step 2 (授权模式) |
| 提示 "Remote Desktop license expired" 或超过最大连接数 | Step 3 (Grace Period) → Step 4 (授权服务器) |
| 120 天试用期到期后无法连接 | Step 3 (Grace Period) → Step 2 (授权模式) → Step 4 (授权服务器) |
| RDS CAL 授权问题 | Step 2 (授权模式) → Step 4 (授权服务器) |

## 诊断步骤

### Step 1: Remote Desktop Session Host 角色检查

**数据采集**：

> 采集目标：检查 Remote Desktop Session Host (RDSH) 角色是否安装，判断当前服务器是否需要 RDS 授权

```powershell
@('TermService', 'SessionEnv', 'UmRdpService') | ForEach-Object {
    Get-Service -Name $_ -ErrorAction SilentlyContinue | Select-Object Name, Status, StartType
} | Format-Table -AutoSize

# Check RDSH role installation status
Get-WindowsFeature -Name RDS-RD-Server -ErrorAction SilentlyContinue | Select-Object Name, DisplayName, InstallState | Format-Table -AutoSize

# Check RD Licensing role
Get-WindowsFeature -Name RDS-Licensing -ErrorAction SilentlyContinue | Select-Object Name, DisplayName, InstallState | Format-Table -AutoSize
```

**分析思路**：

1. 快速预检 RDS 基础服务状态：
   - 正常：TermService / SessionEnv / UmRdpService 均为 Running → 继续授权检查
   - 异常：任一服务未运行 → **跳转 [rdp-service.md](references/rdp-service.md)** 排查服务问题，不在本文件中做根因判定

2. 检查 RDSH 角色是否安装：
   - 正常（未安装 RDSH 角色）：服务器仅支持默认的 2 个管理会话，无需 RDS 授权，授权问题不适用
   - 异常（已安装 RDSH 角色但未配置授权）：需要继续检查授权配置 → 执行后续步骤

3. 检查 RD Licensing 角色是否安装：
   - 正常：RD Licensing 角色已安装（本机作为授权服务器）
   - 信息：未安装 → 需要配置外部授权服务器

> 注意：如果未安装 RDSH 角色，后续步骤可以跳过。Windows Server 默认提供 2 个并发管理远程桌面会话，不需要 RDS CAL。

### Step 2: RDS 授权模式配置检查

**数据采集**：

> 采集目标：获取 RDS 授权模式配置（Per Device / Per User）和授权服务器地址

```powershell
# Get TerminalServiceSetting; check TerminalServerMode to determine if RDSH is in application server mode
$tss = Get-CimInstance -Namespace "root/CIMV2/TerminalServices" -ClassName "Win32_TerminalServiceSetting" -ErrorAction SilentlyContinue
if ($tss.TerminalServerMode -eq 0) {
    [PSCustomObject]@{
        TerminalServerMode = 0
        Note               = 'RDSH not in application server mode; RDS licensing check not applicable'
    } | Format-List
    return
}

# Check licensing mode configuration and policy source
$tss | Select-Object LicensingType, LicensingName, PolicySourceLicensingType, PossibleLicensingTypes | Format-Table -AutoSize

# Check specified license server list
Invoke-CimMethod -InputObject $tss -MethodName GetSpecifiedLicenseServerList
```

**分析思路**：

0. 预检 RDSH 应用服务器模式：
   - TerminalServerMode = 0 → **正常**：当前为 Remote Desktop for Administration 模式，仅支持默认 2 管理会话，无需 RDS 授权配置，后续 Step 3/4 可跳过
   - TerminalServerMode = 1 → 已安装 RDSH 并处于 Application Server 模式，继续授权检查

1. 检查授权模式是否配置：
   - 正常：LicensingType = 2 (Per Device) 或 4 (Per User)
   - 异常：LicensingType 未配置或值异常 → **根因**：RDS 授权模式未配置，多用户连接时会提示 "licensing mode is not configured"，**严重程度**：Critical

2. 检查授权服务器是否配置：
   - 正常：GetSpecifiedLicenseServerList 返回有效的授权服务器地址
   - 异常：未配置授权服务器 → **根因**：RDS 授权服务器未配置，Grace Period 到期后将无法获取 CAL，**严重程度**：Warning

### Step 3: Grace Period 状态检查

**数据采集**：

> 采集目标：检查 RDS Grace Period（120 天试用期）的状态和剩余时间

```powershell
# Get TerminalServiceSetting; check TerminalServerMode to determine if RDSH is in application server mode
$tss = Get-CimInstance -Namespace "root/CIMV2/TerminalServices" -ClassName "Win32_TerminalServiceSetting" -ErrorAction SilentlyContinue
if ($tss.TerminalServerMode -eq 0) {
    [PSCustomObject]@{
        TerminalServerMode = 0
        Note               = 'RDSH not in application server mode; Grace Period check not applicable'
    } | Format-List
    return
}

# Query RDS licensing status and Grace Period remaining days
$tss | Select-Object LicensingType, LicensingName | Format-Table -AutoSize
Invoke-CimMethod -InputObject $tss -MethodName GetGracePeriodDays

# Query license diagnostic information
Get-CimInstance -Namespace "root/CIMV2/TerminalServices" -ClassName "Win32_TSLicenseKeyPack" -ErrorAction SilentlyContinue | Select-Object KeyPackType, ProductType, TotalLicenses, IssuedLicenses, AvailableLicenses | Format-Table -AutoSize
```

**分析思路**：

> 前置条件：TerminalServerMode = 1（RDSH 已安装并处于 Application Server 模式）。若 TerminalServerMode = 0，本步骤不适用，正常跳过。

1. 检查 Grace Period 状态：
   - 正常：GetGracePeriodDays 返回剩余天数 > 0，或已正确配置授权无需 Grace Period
   - 异常：Grace Period 已过期（剩余天数 = 0 或调用失败） → **根因**：RDS 120 天试用期已过期，新的远程桌面连接将被拒绝，**严重程度**：Critical

2. 检查已颁发的许可证：
   - 正常：有可用许可证
   - 异常：可用许可证数量为 0 → **根因**：RDS CAL 已耗尽，新用户/设备无法获取授权，**严重程度**：Critical

### Step 4: 授权服务器连通性检查

**数据采集**：

> 采集目标：检查 RDS 授权服务器的可达性和服务状态

```powershell
# Get TerminalServiceSetting; check TerminalServerMode to determine if RDSH is in application server mode
$tss = Get-CimInstance -Namespace "root/CIMV2/TerminalServices" -ClassName "Win32_TerminalServiceSetting" -ErrorAction SilentlyContinue
if ($tss.TerminalServerMode -eq 0) {
    [PSCustomObject]@{
        TerminalServerMode = 0
        Note               = 'RDSH not in application server mode; license server connectivity check not applicable'
    } | Format-List
    return
}

# Get configured license server address
Invoke-CimMethod -InputObject $tss -MethodName GetSpecifiedLicenseServerList

# Check connectivity status to license server
Invoke-CimMethod -InputObject $tss -MethodName GetTStoLSConnectivityStatus

# Check local RD Licensing service
Get-Service -Name TermServLicensing -ErrorAction SilentlyContinue | Select-Object Name, Status, StartType | Format-Table -AutoSize

# Query license server discovery results
Get-CimInstance -Namespace "root/CIMV2/TerminalServices" -ClassName "Win32_TSDeploymentLicensing" -ErrorAction SilentlyContinue | Select-Object LicenseServer, LicenseServerType | Format-Table -AutoSize
```

**分析思路**：

> 前置条件：TerminalServerMode = 1（RDSH 已安装并处于 Application Server 模式）。若 TerminalServerMode = 0，本步骤不适用，正常跳过。

1. 检查授权服务器地址是否配置：
   - 正常：GetSpecifiedLicenseServerList 返回有效的授权服务器地址
   - 异常：未配置 → 参考 Step 2 的根因

2. 检查到授权服务器的连通性：
   - 正常：GetTStoLSConnectivityStatus 返回连通正常
   - 异常：连通性异常 → **根因**：RDS 授权服务器不可达，无法获取或验证 CAL，**严重程度**：Critical

3. 检查本地授权服务状态（如果本机作为授权服务器）：
   - 正常：TermServLicensing 服务正在运行
   - 异常：服务已停止 → **根因**：本地 RD Licensing 服务未运行，**严重程度**：Critical

> 如果怀疑防火墙阻止授权通信，参见 → [networking-firewall.md](references/networking-firewall.md)（检查出站 TCP 135 端口规则）

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 参数化引用 | 防火墙阻止授权服务器通信 | → [networking-firewall.md](references/networking-firewall.md)（检查出站 TCP 135 端口规则） |
| 条件跳转 | RDP 连接本身就无法建立 | → [rdp-service.md](references/rdp-service.md) |
| 链式后继 | 本文件未确认根因，用户报告 RDP 连接问题 | → [rdp-service.md](references/rdp-service.md) |

## 修复建议

### 根因：RDS 授权模式未配置

**修复操作**：

```powershell
# Configure RDS licensing mode to Per User
$tss = Get-CimInstance -Namespace "root/CIMV2/TerminalServices" -ClassName "Win32_TerminalServiceSetting" -ErrorAction SilentlyContinue
if ($tss) { Invoke-CimMethod -InputObject $tss -MethodName ChangeMode -Arguments @{LicensingType=4} }
Write-Host "Configured RDS licensing mode to Per User"

# If need to configure license server address (replace <LicenseServerAddress> with actual address)
# if ($tss) { Invoke-CimMethod -InputObject $tss -MethodName AddLSToSpecifiedLicenseServerList -Arguments @{LicenseServerName="<LicenseServerAddress>"} }
```

**验证方法**：

```powershell
Get-CimInstance -Namespace "root/CIMV2/TerminalServices" -ClassName "Win32_TerminalServiceSetting" -ErrorAction SilentlyContinue | Select-Object LicensingType, LicensingName
```

预期结果：LicensingType = 4 (Per User)

**风险说明**：授权模式的选择取决于购买的 CAL 类型，Per User 和 Per Device 不可混用，请确认后再配置

---

### 根因：RDS 120 天试用期已过期

**修复操作**：

```powershell
# Note: The proper solution is to purchase and configure RDS CAL
# The following steps are for diagnostic confirmation only, administrator needs to configure correct license server in Group Policy

# 1. Configure license server (replace <LicenseServerAddress> with actual address)
# $tss = Get-CimInstance -Namespace "root/CIMV2/TerminalServices" -ClassName "Win32_TerminalServiceSetting" -ErrorAction SilentlyContinue
# if ($tss) { Invoke-CimMethod -InputObject $tss -MethodName AddLSToSpecifiedLicenseServerList -Arguments @{LicenseServerName="<LicenseServerAddress>"} }

# 2. Configure licensing mode
# if ($tss) { Invoke-CimMethod -InputObject $tss -MethodName ChangeMode -Arguments @{LicensingType=4} }

Write-Host "Please contact administrator to configure valid RDS license server and CAL"
```

**验证方法**：

```powershell
Get-CimInstance -Namespace "root/CIMV2/TerminalServices" -ClassName "Win32_TSLicenseKeyPack" -ErrorAction SilentlyContinue | Select-Object KeyPackType, TotalLicenses, AvailableLicenses
```

预期结果：有可用的 RDS CAL 许可证

**风险说明**：Grace Period 过期后的根本解决方案是购买合法的 RDS CAL。删除 Grace Period 注册表项仅适用于测试环境，不建议在生产环境使用

---

### 根因：RDS CAL 已耗尽

**修复操作**：

```powershell
# View current license usage
Get-CimInstance -Namespace "root/CIMV2/TerminalServices" -ClassName "Win32_TSLicenseKeyPack" -ErrorAction SilentlyContinue | Select-Object KeyPackType, ProductType, TotalLicenses, IssuedLicenses, AvailableLicenses | Format-Table -AutoSize

# If using Per Device mode, can revoke unused device CALs
# Need to operate in RD Licensing Manager
Write-Host "Recommendation: Purchase more RDS CALs or clean up unused device licenses"
```

**验证方法**：

```powershell
Get-CimInstance -Namespace "root/CIMV2/TerminalServices" -ClassName "Win32_TSLicenseKeyPack" -ErrorAction SilentlyContinue | Select-Object TotalLicenses, AvailableLicenses
```

预期结果：AvailableLicenses > 0

**风险说明**：撤销设备 CAL 可能导致相关设备下次连接时需要重新获取授权

---

### 根因：RDS 授权服务器不可达

**修复操作**：

```powershell
# 1. Confirm license server address is configured correctly
$tss = Get-CimInstance -Namespace "root/CIMV2/TerminalServices" -ClassName "Win32_TerminalServiceSetting" -ErrorAction SilentlyContinue
if ($tss) { Invoke-CimMethod -InputObject $tss -MethodName GetSpecifiedLicenseServerList }

# 2. Test network connectivity (replace <LicenseServerAddress> with actual address)
# Test-NetConnection -ComputerName <LicenseServerAddress> -Port 135

# 3. If firewall issue, open RPC port
# New-NetFirewallRule -DisplayName "Allow RDS Licensing RPC" -Direction Outbound -Protocol TCP -RemotePort 135 -Action Allow
```

**验证方法**：

```powershell
# Replace <LicenseServerAddress> with actual address
# Test-NetConnection -ComputerName <LicenseServerAddress> -Port 135
```

预期结果：TcpTestSucceeded = True

**风险说明**：开放防火墙端口会增加网络暴露面，建议限定目标 IP

---

### 根因：本地 RD Licensing 服务未运行

**修复操作**：

```powershell
# Start RD Licensing service
Set-Service -Name TermServLicensing -StartupType Automatic
Start-Service -Name TermServLicensing
Write-Host "Started RD Licensing service"
```

**验证方法**：

```powershell
Get-Service -Name TermServLicensing | Select-Object Name, Status, StartType
```

预期结果：Status = Running，StartType = Automatic

**风险说明**：如果本机不是授权服务器，启动此服务不会解决问题，需要配置正确的外部授权服务器

---

### 根因：RDS 授权服务器未配置

**修复操作**：

```powershell
# Configure license server address (replace <LicenseServerAddress> with actual address)
$tss = Get-CimInstance -Namespace "root/CIMV2/TerminalServices" -ClassName "Win32_TerminalServiceSetting" -ErrorAction SilentlyContinue
if ($tss) { Invoke-CimMethod -InputObject $tss -MethodName AddLSToSpecifiedLicenseServerList -Arguments @{LicenseServerName="<LicenseServerAddress>"} }

# Configure licensing mode
if ($tss) { Invoke-CimMethod -InputObject $tss -MethodName ChangeMode -Arguments @{LicensingType=4} }

Write-Host "Configured RDS license server, please confirm server address is correct"
```

**验证方法**：

```powershell
$tss = Get-CimInstance -Namespace "root/CIMV2/TerminalServices" -ClassName "Win32_TerminalServiceSetting" -ErrorAction SilentlyContinue
$tss | Select-Object LicensingType | Format-Table -AutoSize
if ($tss) { Invoke-CimMethod -InputObject $tss -MethodName GetSpecifiedLicenseServerList }
```

预期结果：GetSpecifiedLicenseServerList 返回有效的服务器地址，LicensingType 为 2 或 4

**风险说明**：配置错误的授权服务器地址会导致授权获取失败
