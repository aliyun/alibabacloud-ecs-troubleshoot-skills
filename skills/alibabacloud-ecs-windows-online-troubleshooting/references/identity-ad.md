# Identity Active Directory 诊断

## 功能说明

诊断 Windows SID 冲突、域安全通道状态、域控制器可达性、计算机账户密码和 LDAPS 连接。覆盖 5 个已知问题项。

**输入**: 用户问题描述(必选)
**输出**: 根因列表(root_cause / severity / evidence / explanation / fix)

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 多台域机器出现身份验证异常或权限混乱 | Step 1 (SID 冲突) |
| 提示"此工作站和主域间的信任关系失败"、无法使用域账户登录 | Step 2 (域安全通道状态) |
| 域登录超时、组策略无法更新、域资源访问失败 | Step 3 (域控制器可达性) |
| 重启后无法加入域、提示计算机账户验证失败 | Step 4 (计算机账户密码) |
| LDAP over SSL 连接失败、域服务安全通信异常 | Step 5 (LDAPS 连接) |

## 诊断步骤

### Step 1: 检查 SID 冲突

**数据采集**：

> 采集目标：获取本机 SID，用于判断是否存在 SID 重复（通常发生在克隆镜像未做 Sysprep 的场景）

```powershell
# Get local machine SID and Sysprep status
$computerSid = (Get-CimInstance Win32_UserAccount -Filter "LocalAccount=True" | Select-Object -First 1).SID
if ($computerSid) {
    $machineSid = $computerSid.Substring(0, $computerSid.LastIndexOf('-'))
    Write-Host "Machine SID: $machineSid"
} else {
    Write-Host "WARNING: Unable to retrieve Machine SID"
}
# Check Sysprep image state
$imageState = (Get-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Setup\State' -Name 'ImageState' -ErrorAction SilentlyContinue).ImageState
Write-Host "ImageState: $imageState"
# Check SID generation marker
$sysprepState = (Get-ItemProperty -Path 'HKLM:\SYSTEM\Setup\Status\SysprepStatus' -ErrorAction SilentlyContinue)
if ($sysprepState) {
    Write-Host "CleanupState: $($sysprepState.CleanupState)"
    Write-Host "GeneralizationState: $($sysprepState.GeneralizationState)"
}
```

**分析思路**：

1. 检查镜像状态：
   - 正常：ImageState 为 `IMAGE_STATE_COMPLETE`
   - 异常：ImageState 不为 `IMAGE_STATE_COMPLETE`（如 `IMAGE_STATE_GENERALIZE_RESEAL_TO_OOBE`） → **根因**：Sysprep 未完成，可能存在 SID 冲突，**严重程度**：Warning
2. 检查 GeneralizationState：
   - 正常：GeneralizationState = 7（已完成泛化）
   - 异常：其他值 → **根因**：系统未经过正确的 Sysprep 泛化处理，多台克隆实例可能共享同一 SID，**严重程度**：Warning

### Step 2: 检查域安全通道状态

**数据采集**：

> 采集目标：验证本机与域控制器之间的安全通道是否正常，以及域成员身份信息

```powershell
# Check if computer is domain-joined
$cs = Get-CimInstance Win32_ComputerSystem | Select-Object Name, Domain, DomainRole, PartOfDomain | Format-List
$cs
# If domain-joined, test secure channel
if ((Get-CimInstance Win32_ComputerSystem).PartOfDomain) {
    try {
        $result = Test-ComputerSecureChannel -Verbose 2>&1
        Write-Host "SecureChannel Test: $result"
    } catch {
        Write-Host "SecureChannel Test Failed: $($_.Exception.Message)"
    }
    # Check Netlogon service status
    Get-Service Netlogon | Select-Object Name, Status, StartType | Format-Table -AutoSize
} else {
    Write-Host "INFO: Computer is not domain-joined, skipping secure channel test"
}
```

**分析思路**：

1. 检查计算机是否加域：
   - 如果 PartOfDomain 为 False，本步骤无需继续
2. 检查安全通道测试结果：
   - 正常：Test-ComputerSecureChannel 返回 True
   - 异常：返回 False 或抛出异常 → **根因**：域安全通道已断开，域账户登录和组策略更新将失败，**严重程度**：Critical
3. 检查 Netlogon 服务状态：
   - 正常：状态为 Running，启动类型为 Automatic
   - 异常：服务未运行 → **根因**：Netlogon 服务未启动导致安全通道不可用，**严重程度**：Critical

### Step 3: 检查域控制器可达性

**数据采集**：

> 采集目标：验证域控制器的 DNS 解析和网络可达性

```powershell
if ((Get-CimInstance Win32_ComputerSystem).PartOfDomain) {
    $domain = (Get-CimInstance Win32_ComputerSystem).Domain
    Write-Host "Domain: $domain"
    # Try to resolve domain controller
    try {
        $dcRecords = Resolve-DnsName -Name "_ldap._tcp.dc._msdcs.$domain" -Type SRV -ErrorAction Stop
        $dcRecords | Select-Object Name, NameTarget, Port, Priority | Format-Table -AutoSize
    } catch {
        Write-Host "ERROR: Failed to resolve DC SRV records: $($_.Exception.Message)"
    }
    # Check LDAP (389) and Kerberos (88) port reachability
    try {
        $dc = (Resolve-DnsName -Name $domain -ErrorAction Stop | Select-Object -First 1).IPAddress
        if ($dc) {
            Write-Host "DC IP: $dc"
            $ldap = Test-NetConnection -ComputerName $dc -Port 389 -WarningAction SilentlyContinue
            Write-Host "LDAP (389): TcpTestSucceeded=$($ldap.TcpTestSucceeded)"
            $kerb = Test-NetConnection -ComputerName $dc -Port 88 -WarningAction SilentlyContinue
            Write-Host "Kerberos (88): TcpTestSucceeded=$($kerb.TcpTestSucceeded)"
        }
    } catch {
        Write-Host "ERROR: Cannot resolve domain: $($_.Exception.Message)"
    }
} else {
    Write-Host "INFO: Computer is not domain-joined"
}
```

**分析思路**：

1. 检查 DC SRV 记录解析：
   - 正常：能解析到至少一条 _ldap._tcp.dc._msdcs 的 SRV 记录
   - 异常：解析失败 → **根因**：DNS 无法解析域控制器 SRV 记录，域服务不可用，**严重程度**：Critical
2. 检查 LDAP 端口连通性：
   - 正常：TCP 389 端口连通
   - 异常：连接失败 → **根因**：域控制器 LDAP 端口不可达，**严重程度**：Critical
3. 检查 Kerberos 端口连通性：
   - 正常：TCP 88 端口连通
   - 异常：连接失败 → **根因**：域控制器 Kerberos 端口不可达，域认证将失败，**严重程度**：Critical

> 如果域控不可达但网络基础连通正常，参见 → [networking-firewall.md](references/networking-firewall.md)（检查出站 TCP 88/389/636 端口规则）

### Step 4: 检查计算机账户密码

**数据采集**：

> 采集目标：获取计算机账户密码的最后更新时间和密码更改策略配置

```powershell
if ((Get-CimInstance Win32_ComputerSystem).PartOfDomain) {
    # Check computer account password last change time
    $pwdAge = Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters' -ErrorAction SilentlyContinue
    if ($pwdAge) {
        Write-Host "DisablePasswordChange: $($pwdAge.DisablePasswordChange)"
        Write-Host "MaximumPasswordAge: $($pwdAge.MaximumPasswordAge) days"
    }
    # Get password update records from Netlogon events
    Get-WinEvent -FilterHashtable @{LogName='System'; Level=2,3} -MaxEvents 5 -ErrorAction SilentlyContinue |
        Where-Object { $_.ProviderName -eq 'NETLOGON' } |
        Select-Object TimeCreated, Id, LevelDisplayName, Message | Format-Table -AutoSize
} else {
    Write-Host "INFO: Computer is not domain-joined"
}
```

**分析思路**：

1. 检查密码更改策略：
   - 正常：DisablePasswordChange 为 0 或不存在
   - 异常：DisablePasswordChange 为 1 → **根因**：计算机账户密码自动更换被禁用，长期可能导致安全通道信任失败，**严重程度**：Warning
2. 检查 Netlogon 事件：
   - 关注 EventID 5722（计算机账户密码验证失败）、5723（安全通道设置失败）
   - 出现此类事件 → **根因**：计算机账户密码与域控不同步，**严重程度**：Critical

### Step 5: 检查 LDAPS 连接

**数据采集**：

> 采集目标：验证 LDAP over SSL (端口 636) 的连通性和证书状态

```powershell
if ((Get-CimInstance Win32_ComputerSystem).PartOfDomain) {
    $domain = (Get-CimInstance Win32_ComputerSystem).Domain
    try {
        $dc = (Resolve-DnsName -Name $domain -ErrorAction Stop | Select-Object -First 1).IPAddress
        if ($dc) {
            # Test LDAPS port
            $ldaps = Test-NetConnection -ComputerName $dc -Port 636 -WarningAction SilentlyContinue
            Write-Host "LDAPS (636): TcpTestSucceeded=$($ldaps.TcpTestSucceeded)"
        }
    } catch {
        Write-Host "ERROR: Cannot resolve domain: $($_.Exception.Message)"
    }
    # Check LDAP client signing requirements
    $ldapSigning = (Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\ldap' -Name 'LDAPClientIntegrity' -ErrorAction SilentlyContinue).LDAPClientIntegrity
    Write-Host "LDAPClientIntegrity: $ldapSigning (0=none, 1=negotiate, 2=require)"
} else {
    Write-Host "INFO: Computer is not domain-joined"
}
```

**分析思路**：

1. 检查 LDAPS 端口连通性：
   - 正常：TCP 636 端口连通
   - 异常：连接失败 → **根因**：LDAPS 端口不可达，加密 LDAP 通信不可用，**严重程度**：Warning
2. 检查 LDAP 客户端签名策略：
   - 正常：值为 0 或 1（不要求或协商签名）
   - 异常：值为 2 且 LDAPS 不可用 → **根因**：LDAP 客户端要求签名但 LDAPS 连接失败，域通信将受阻，**严重程度**：Critical

> 如果 LDAPS 证书问题，参见 → [security-certificates.md](references/security-certificates.md)

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Step 3 域控端口不可达但网络基础连通正常 | → [networking-firewall.md](references/networking-firewall.md)（检查出站 TCP 88/389/636 端口规则） |
| 条件跳转 | Step 5 LDAPS 证书问题 | → [security-certificates.md](references/security-certificates.md) |
| 条件跳转 | 域时间不同步导致 Kerberos 认证失败 | → [system-time.md](references/system-time.md) |
| 链式后继 | 本文件未确认根因 | → 无（域相关问题通常在此文件内定位） |

## 修复建议

### 根因: 域安全通道断开

**修复操作**：

```powershell
# Reset computer account password and repair secure channel (requires domain admin credentials)
Test-ComputerSecureChannel -Repair -Credential (Get-Credential -Message "Enter domain administrator credentials")
```

**验证方法**：

```powershell
Test-ComputerSecureChannel
```

预期结果：返回 True

**风险说明**：重置安全通道会更新计算机账户密码，如果域控侧密码已过期，可能需要在域控上重置计算机账户。

### 根因: Netlogon 服务未启动

**修复操作**：

```powershell
Set-Service -Name Netlogon -StartupType Automatic
Start-Service Netlogon
```

**验证方法**：

```powershell
Get-Service Netlogon | Select-Object Name, Status, StartType | Format-Table -AutoSize
```

预期结果：Status 为 Running，StartType 为 Automatic

**风险说明**：启动 Netlogon 服务本身风险低，但如果安全通道已断开，需要先修复安全通道。

### 根因: 计算机账户密码更换被禁用

**修复操作**：

```powershell
# Enable automatic computer account password change
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters' -Name 'DisablePasswordChange' -Value 0 -Type DWord
```

**验证方法**：

```powershell
(Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\Netlogon\Parameters').DisablePasswordChange
```

预期结果：返回 0

**风险说明**：启用密码自动更换后，系统将按照 MaximumPasswordAge（默认30天）定期更新计算机账户密码，这是推荐的安全配置。
