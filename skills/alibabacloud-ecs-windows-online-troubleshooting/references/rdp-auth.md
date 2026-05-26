# RDP 认证诊断

## 目录

- [RDP 认证诊断](#rdp-认证诊断)
  - [目录](#目录)
  - [功能说明](#功能说明)
  - [步骤选取指引](#步骤选取指引)
  - [诊断步骤](#诊断步骤)
    - [Step 1: CredSSP 配置检查](#step-1-credssp-配置检查)
    - [Step 2: 账户锁定状态检查](#step-2-账户锁定状态检查)
    - [Step 3: 远程登录权限检查](#step-3-远程登录权限检查)
    - [Step 4: 安全层配置检查](#step-4-安全层配置检查)
    - [Step 5: 密码错误审计事件检查](#step-5-密码错误审计事件检查)
    - [Step 6: ForceGuest 访问模式检查](#step-6-forceguest-访问模式检查)
  - [交叉引用](#交叉引用)
  - [修复建议](#修复建议)
    - [根因：CredSSP 策略导致认证失败 / 补丁不匹配](#根因credssp-策略导致认证失败--补丁不匹配)
    - [根因：账户锁定策略未配置](#根因账户锁定策略未配置)
    - [根因：RDP 使用原生安全层](#根因rdp-使用原生安全层)
    - [根因：NLA 未启用](#根因nla-未启用)
    - [根因：存在暴力破解尝试或凭据配置错误](#根因存在暴力破解尝试或凭据配置错误)
    - [根因：用户缺少远程登录权限](#根因用户缺少远程登录权限)
    - [根因：用户被显式拒绝远程登录](#根因用户被显式拒绝远程登录)
    - [根因：账户被锁定](#根因账户被锁定)
    - [根因：ForceGuest 强制访客模式](#根因forceguest-强制访客模式)

## 功能说明

诊断远程桌面认证和权限问题。覆盖 CredSSP 配置、账户锁定状态、远程登录权限（含 SeDenyRemoteInteractiveLogonRight 拒绝策略）、安全层配置、密码错误审计事件、ForceGuest 访问模式等 6 个诊断步骤。

**输入**：用户问题描述（必选）、认证错误信息（可选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 报错 "An authentication error has occurred. The function requested is not supported" 或提示 "CredSSP encryption oracle remediation" | Step 1 (CredSSP 配置) |
| 认证错误 (An authentication error)，非 CredSSP 相关 | Step 4 (安全层配置) → Step 1 (CredSSP 配置) |
| 账户被锁定 | Step 2 (账户锁定状态) → Step 5 (密码错误事件) |
| 密码错误、凭据不正确 | Step 5 (密码错误事件) → Step 2 (账户锁定) |
| 没有远程登录权限 | Step 3 (远程登录权限) |
| 非管理员账户远程登录失败（如被映射为 Guest） | Step 6 (ForceGuest 访问模式) |
| 提示“登录被拒绝”但用户在 Remote Desktop Users 组中 | Step 3 (远程登录权限，检查 SeDenyRemoteInteractiveLogonRight) |

## 诊断步骤

### Step 1: CredSSP 配置检查

> 背景：CVE-2018-0886 是 CredSSP 协议的远程代码执行漏洞。微软于 2018 年发布安全更新，并在后续更新中将默认策略从 Vulnerable 改为 Mitigated，导致已打补丁的客户端无法连接未打补丁的服务器（反之亦然）。

**数据采集**：

> 采集目标：获取本机 CredSSP 补丁状态（TSpkg.dll 版本）和 AllowEncryptionOracle 策略配置

```powershell
# 1. Check TSpkg.dll version (determine if CVE-2018-0886 patch is installed)
Get-Item "$env:SystemRoot\System32\TSpkg.dll" -ErrorAction SilentlyContinue | Select-Object FullName, @{N='FileVersion';E={$_.VersionInfo.FileVersion}} | Format-Table -AutoSize

# 2. Check AllowEncryptionOracle policy configuration
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" -Name "AllowEncryptionOracle" -ErrorAction SilentlyContinue | Select-Object AllowEncryptionOracle

# 3. Check Encryption Oracle Remediation configuration in Group Policy
# Path: Computer Configuration > Administrative Templates > System > Credentials Delegation > Encryption Oracle Remediation
Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CredentialsDelegation" -Name "AllowEncryptionOracle" -ErrorAction SilentlyContinue | Select-Object AllowEncryptionOracle
```

**分析思路**：

AllowEncryptionOracle 三个值的含义：
- **0 (Force Updated Clients)**：最安全，仅允许已打补丁的客户端连接
- **1 (Mitigated)**：默认值，拒绝出站到未打补丁服务器的连接，但允许未打补丁客户端的入站连接
- **2 (Vulnerable)**：不安全，允许连接到未打补丁的 CredSSP 主机

连接行为判定矩阵（服务器视角）：

| 服务器补丁 | 客户端补丁 | Force Updated (0) | Mitigated (1) | Vulnerable (2) |
|-----------|-----------|-------------------|---------------|----------------|
| 已安装 | 未安装 | 阻止 | 允许 | 允许 |
| 未安装 | 已安装 | 阻止 | 阻止 | 允许 |
| 已安装 | 已安装 | 允许 | 允许 | 允许 |

1. 检查 TSpkg.dll 版本判断补丁状态：
   - 通过文件版本判断是否包含 CVE-2018-0886 修复。参考最低版本：Windows Server 2016 需 ≥ 10.0.14393.2248，Server 2012 R2 需 ≥ 6.3.9600.18999，Server 2008 R2 需 ≥ 6.1.7601.24117
   - 如果版本低于对应基线 → **根因**：服务器未安装 CredSSP 安全更新 (CVE-2018-0886)，已打补丁的客户端将无法连接，**严重程度**：Critical

2. 检查 AllowEncryptionOracle 策略：
   - 正常：值为 0 或 1，或未配置（等同于 1）
   - 异常：值为 2 → **根因**：CredSSP 策略设置为 Vulnerable，允许不安全连接，**严重程度**：Warning
   - 如果用户报告认证错误且值为 0 → 可能是 Force Updated 模式阻止了未打补丁客户端的连接，需结合补丁状态判断

### Step 2: 账户锁定状态检查

**数据采集**：

> 采集目标：获取所有已启用的本地用户账户的锁定状态

```powershell
# Check lockout status of all enabled local user accounts (filter by Name if user specified an account)
Get-CimInstance -ClassName Win32_UserAccount -Filter "LocalAccount=True AND Disabled=False" |
    Select-Object Name, Lockout, Status, Disabled | Format-Table -AutoSize
```

**分析思路**：

1. 检查账户是否被锁定：
   - 正常：所有账户 Lockout = False
   - 异常：某账户 Lockout = True → **根因**：用户账户已被锁定，**严重程度**：Critical
   - 如果用户指定了特定账户名，重点关注该账户的 Lockout 状态

### Step 3: 远程登录权限检查

**数据采集**：

> 采集目标：获取 Remote Desktop Users 和 Administrators 组的成员列表，本地安全策略中的远程登录允许和拒绝配置

```powershell
# Get all members of Remote Desktop Users and Administrators groups
Get-CimInstance -ClassName Win32_GroupUser |
    Where-Object { $_.GroupComponent.Name -in @('Remote Desktop Users', 'Administrators') } |
    Select-Object @{N='Group';E={$_.GroupComponent.Name}}, @{N='Member';E={$_.PartComponent.Name}} |
    Format-Table -AutoSize

# Check remote logon permissions in local security policy (allow + deny)
secedit /export /cfg "$env:TEMP\secpol.cfg" /quiet
Write-Host "--- Allowed remote logon ---"
Select-String -Path "$env:TEMP\secpol.cfg" -Pattern "SeRemoteInteractiveLogonRight"
Write-Host "--- Denied remote logon ---"
Select-String -Path "$env:TEMP\secpol.cfg" -Pattern "SeDenyRemoteInteractiveLogonRight"
Remove-Item "$env:TEMP\secpol.cfg" -ErrorAction SilentlyContinue
```

**分析思路**：

1. 检查用户是否有远程登录权限：
   - 正常：目标用户在 Administrators 或 Remote Desktop Users 组中
   - 异常：目标用户不在任何允许远程登录的组 → **根因**：用户缺少远程登录权限，**严重程度**：Critical
   - 如果用户未指定账户名，列出两个组的所有成员供 LLM 判断

2. 检查是否被显式拒绝远程登录：
   - 正常：SeDenyRemoteInteractiveLogonRight 未配置，或目标用户及其所属组不在拒绝列表中
   - 异常：目标用户或其所属组出现在 SeDenyRemoteInteractiveLogonRight 拒绝列表中 → **根因**：用户被显式拒绝远程登录（SeDenyRemoteInteractiveLogonRight），即使在 Remote Desktop Users 组中也无法登录，**严重程度**：Critical
   - 注意：拒绝策略优先于允许策略，即使用户同时在允许和拒绝列表中，最终仍被拒绝

### Step 4: 安全层配置检查

**数据采集**：

> 采集目标：获取所有 WinStation 的安全层（SecurityLayer）和网络级别身份验证（NLA）的配置值

```powershell
# Check security layer and NLA configuration for all WinStations
$winStationsPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations"
Get-ChildItem -Path $winStationsPath -ErrorAction SilentlyContinue |
    Where-Object { $_.PSChildName -ne "Console" } |
    ForEach-Object {
        $props = Get-ItemProperty -Path $_.PSPath -ErrorAction SilentlyContinue
        [PSCustomObject]@{
            StationName        = $_.PSChildName
            SecurityLayer      = $props.SecurityLayer
            UserAuthentication = $props.UserAuthentication
        }
    } | Format-Table -AutoSize
```

**分析思路**：

1. 对每个 WinStation 检查安全层配置：
   - 正常：SecurityLayer = 1（协商）或 2（强制 SSL）
   - 异常：SecurityLayer = 0 → **根因**：RDP 使用原生安全层，安全性较低（标注 StationName），**严重程度**：Warning

2. 对每个 WinStation 检查 NLA 配置：
   - 正常：UserAuthentication = 1（启用 NLA）
   - 异常：UserAuthentication = 0 → **根因**：NLA 未启用，可能影响某些客户端连接（标注 StationName），**严重程度**：Warning

### Step 5: 密码错误审计事件检查

**数据采集**：

> 采集目标：获取最近 1 小时内的登录失败事件原始记录，交由 LLM 分析账户名、来源 IP 和失败原因

```powershell
# Query login failure events in the last hour, output raw information directly
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    ID = 4625
    StartTime = (Get-Date).AddHours(-1)
} -MaxEvents 10 -ErrorAction SilentlyContinue | Select-Object TimeCreated, Id, Message
```

**分析思路**：

1. 检查登录失败事件：
   - 正常：无 4625 事件或少量
   - 异常：大量 4625 事件 → **根因**：存在暴力破解尝试或凭据配置错误，**严重程度**：Warning
   - 从事件中提取账户名和来源 IP，确认是否为合法用户

### Step 6: ForceGuest 访问模式检查

**数据采集**：

> 采集目标：检查本地安全策略中的 ForceGuest 配置，判断非管理员远程用户是否被强制映射为 Guest

```powershell
# Check ForceGuest configuration (0=Classic mode, 1=Guest only mode)
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name "ForceGuest" -ErrorAction SilentlyContinue | Select-Object ForceGuest
```

**分析思路**：

1. 检查 ForceGuest 配置：
   - 正常：ForceGuest = 0（经典模式，使用用户自己的身份进行认证）
   - 异常：ForceGuest = 1 → **根因**：ForceGuest 强制访客模式，所有非管理员的远程用户被映射为 Guest 账户，导致权限不足无法登录，**严重程度**：Critical
   - 说明：该设置主要影响工作组环境，域环境通常不受影响

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Step 2 账户被锁定 | → [identity-account.md](references/identity-account.md) |
| 链式后继 | 本文件未确认根因,用户报告 RDP 认证问题 | → [rdp-certificate.md](references/rdp-certificate.md) |

## 修复建议

### 根因：CredSSP 策略导致认证失败 / 补丁不匹配

> 场景一：已打补丁的客户端无法连接未打补丁的服务器（最常见）
> 场景二：已打补丁的服务器拒绝未打补丁客户端的连接（Force Updated 模式）

**临时修复**（立即恢复连接，但会降低安全性）：

```powershell
# Set AllowEncryptionOracle to Vulnerable (2) on server to allow unpatched clients to connect
New-Item -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" -Force -ErrorAction SilentlyContinue | Out-Null
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" -Name "AllowEncryptionOracle" -Value 2 -Type DWord -Force
```

**永久修复**（推荐）：

1. 在服务器和客户端上都安装包含 CVE-2018-0886 修复的累积更新（KB 编号见下表）
2. 安装完成后，将 AllowEncryptionOracle 恢复为安全值：

| 操作系统 | 补丁 KB |
|---------|--------|
| Windows Server 2016 / Windows 10 1607 | KB4103723 |
| Windows Server 2012 R2 / Windows 8.1 | KB4103725 |
| Windows Server 2012 | KB4103730 |
| Windows Server 2008 R2 SP1 / Windows 7 SP1 | KB4103718 |

```powershell
# After installing patch, restore policy to secure value
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" -Name "AllowEncryptionOracle" -Value 0 -Type DWord -Force
```

**验证方法**：

```powershell
# Check policy value
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" -Name "AllowEncryptionOracle" -ErrorAction SilentlyContinue | Select-Object AllowEncryptionOracle

# Check TSpkg.dll version to confirm patch is installed
Get-Item "$env:SystemRoot\System32\TSpkg.dll" | Select-Object @{N='FileVersion';E={$_.VersionInfo.FileVersion}} | Format-Table -AutoSize
```

预期结果：临时修复后 AllowEncryptionOracle = 2；永久修复后 AllowEncryptionOracle = 0 且 TSpkg.dll 版本达到对应基线

**风险说明**：临时修复将 AllowEncryptionOracle 设置为 2 会允许不安全的 CredSSP 连接，存在中间人攻击风险。必须尽快安装补丁并恢复为安全值 (0)

---

### 根因：账户锁定策略未配置

**修复操作**：

```powershell
# Configure account lockout policy (example: lock after 5 failures, auto-unlock after 30 minutes)
net accounts /lockoutthreshold:5 /lockoutduration:30 /lockoutwindow:30
```

**验证方法**：

```powershell
net accounts
```

预期结果：Lockout threshold / Lockout duration / Lockout observation window 显示对应配置值

**风险说明**：配置锁定策略可以防止暴力破解，但需要确保用户知道密码策略

---

### 根因：RDP 使用原生安全层

**修复操作**：

```powershell
# Change specified WinStation security layer to Negotiate mode (replace <StationName> with actual station name)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\<StationName>" -Name "SecurityLayer" -Value 1

# Restart TermService
Restart-Service -Name TermService -Force
```

**验证方法**：

```powershell
Get-ChildItem -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations" |
    Where-Object { $_.PSChildName -ne "Console" } |
    ForEach-Object { [PSCustomObject]@{ Station = $_.PSChildName; SecurityLayer = (Get-ItemProperty $_.PSPath).SecurityLayer } }
```

预期结果：目标 WinStation SecurityLayer = 1

**风险说明**：更改安全层会中断现有 RDP 连接

---

### 根因：NLA 未启用

**修复操作**：

```powershell
# Enable Network Level Authentication (NLA) for specified WinStation (replace <StationName> with actual station name)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\<StationName>" -Name "UserAuthentication" -Value 1

# Restart TermService
Restart-Service -Name TermService -Force
```

**验证方法**：

```powershell
Get-ChildItem -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations" |
    Where-Object { $_.PSChildName -ne "Console" } |
    ForEach-Object { [PSCustomObject]@{ Station = $_.PSChildName; UserAuthentication = (Get-ItemProperty $_.PSPath).UserAuthentication } }
```

预期结果：目标 WinStation UserAuthentication = 1

**风险说明**：启用 NLA 后，旧版 RDP 客户端可能无法连接，需要确保客户端支持 NLA

---

### 根因：存在暴力破解尝试或凭据配置错误

**修复操作**：

```powershell
# View recent login failure events
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    ID = 4625
    StartTime = (Get-Date).AddHours(-1)
} -MaxEvents 20 | Select-Object TimeCreated, Message

# If brute force attack, configure IP security policy or use firewall to block source IP
# If credential error, remind user to use correct password
```

**验证方法**：

```powershell
# Confirm whether login failure events have stopped
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    ID = 4625
    StartTime = (Get-Date).AddMinutes(-10)
} -MaxEvents 5
```

预期结果：无新的登录失败事件

**风险说明**：如果是暴力破解攻击，建议配置账户锁定策略并监控安全日志

---

### 根因：用户缺少远程登录权限

**修复操作**：

```powershell
# Add user to Remote Desktop Users group (replace <UserName> with actual username)
Add-LocalGroupMember -Group "Remote Desktop Users" -Member "<UserName>" -ErrorAction SilentlyContinue
Write-Host "Added user to Remote Desktop Users group"
```

**验证方法**：

```powershell
Get-LocalGroupMember -Group "Remote Desktop Users" -ErrorAction SilentlyContinue | Select-Object Name, ObjectClass
```

预期结果：目标用户名出现在列表中

**风险说明**：授予远程登录权限会增加系统暴露面

---

### 根因：用户被显式拒绝远程登录

**修复操作**：

```powershell
# View current SeDenyRemoteInteractiveLogonRight configuration
secedit /export /cfg "$env:TEMP\secpol.cfg" /quiet
Select-String -Path "$env:TEMP\secpol.cfg" -Pattern "SeDenyRemoteInteractiveLogonRight"
Remove-Item "$env:TEMP\secpol.cfg" -ErrorAction SilentlyContinue

# Fix methods:
# 1. Modify via Local Security Policy editor (secpol.msc):
#    Local Policies > User Rights Assignment > Deny log on through Remote Desktop Services
#    Remove target user or group from the list
# 2. Or use ntrights tool (requires Windows Resource Kit):
#    ntrights -u <UserName> -r SeDenyRemoteInteractiveLogonRight
```

**验证方法**：

```powershell
secedit /export /cfg "$env:TEMP\secpol.cfg" /quiet
Select-String -Path "$env:TEMP\secpol.cfg" -Pattern "SeDenyRemoteInteractiveLogonRight"
Remove-Item "$env:TEMP\secpol.cfg" -ErrorAction SilentlyContinue
```

预期结果：目标用户或其所属组不再出现在 SeDenyRemoteInteractiveLogonRight 列表中

**风险说明**：移除拒绝策略会允许该用户远程登录；如果是域策略设置的拒绝，需联系域管理员修改

---

### 根因：账户被锁定

**修复操作**：

```powershell
# Unlock account (replace <UserName> with actual username)
$user = [ADSI]"WinNT://$env:COMPUTERNAME/<UserName>,user"
$user.IsAccountLocked = $false
$user.SetInfo()
```

**验证方法**：

```powershell
# Replace <UserName> with actual username
Get-CimInstance -ClassName Win32_UserAccount -Filter "Name='<UserName>' AND LocalAccount=True" | Select-Object Name, Disabled, Lockout, Status
```

预期结果：Lockout = False

**风险说明**：解锁账户前确认不是暴力破解攻击，如果是攻击应先封锁来源 IP

---

### 根因：ForceGuest 强制访客模式

**修复操作**：

```powershell
# Set ForceGuest to Classic mode (0)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name "ForceGuest" -Value 0
Write-Host "ForceGuest set to Classic mode (authenticate with user's own identity)"
```

**验证方法**：

```powershell
(Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name "ForceGuest").ForceGuest
```

预期结果：ForceGuest = 0

**风险说明**：关闭 ForceGuest 后，远程用户将使用自己的身份认证，确保账户密码强度足够
