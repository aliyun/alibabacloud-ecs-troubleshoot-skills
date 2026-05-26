# Identity Account 诊断

## 功能说明

诊断 Windows 账户锁定状态、密码策略、账户锁定策略、密码过期和内置 Administrator 账户存在性。覆盖 5 个已知问题项。

**输入**: 用户问题描述(必选)
**输出**: 根因列表(root_cause / severity / evidence / explanation / fix)

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 输入正确密码仍无法登录、提示 "account is currently locked out" | Step 1 (账户锁定状态) |
| 密码设置不上、提示不符合复杂度/长度要求 | Step 2 (密码策略) |
| 密码突然过期、RDP 提示需要更改密码 | Step 4 (密码过期状态) |
| 重置密码后仍无法登录、自动化脚本执行失败 | Step 5 (内置 Administrator 账户) |

## 诊断步骤

### Step 1: 账户锁定状态检查

**数据采集**：

> 采集目标：检查本地用户账户的锁定、禁用状态

```powershell
Get-CimInstance -ClassName Win32_UserAccount -Filter "LocalAccount=True" |
  Select-Object Name, Disabled, Lockout, PasswordExpires, SID |
  Format-Table -AutoSize
```

```powershell
# Check account lockout events in Security log (EventID 4740)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4740} -MaxEvents 10 -ErrorAction SilentlyContinue |
  Select-Object TimeCreated, Id, Message |
  Format-Table -AutoSize -Wrap
```

**分析思路**：

1. 检查账户锁定状态：
   - 正常：无账户被锁定（Lockout=False）
   - 异常：存在账户 Lockout=True 或有 EventID 4740 事件 → **根因**：账户被锁定（RDPAccountLocked），**严重程度**：Critical
2. 检查账户禁用状态（排除 DefaultAccount/WDAGUtilityAccount/Guest）：
   - 正常：目标账户 Disabled=False
   - 异常：用户账户 Disabled=True → **根因**：账户被禁用（RDPAccountDisabled），**严重程度**：Warning

### Step 2: 密码策略检查

**数据采集**：

> 采集目标：获取本地密码策略配置

```powershell
net accounts
```

**分析思路**：

1. 检查密码策略配置：
   - 正常：密码策略合理（最小长度、复杂度等符合预期）
   - 异常：最小密码长度过高或复杂度要求过严 → 可能导致密码设置困难

### Step 3: 账户锁定策略检查

**数据采集**：

> 采集目标：获取账户锁定策略参数

```powershell
net accounts | Select-String -Pattern 'Lockout|lockout'
```

**分析思路**：

1. 检查账户锁定策略：
   - Lockout threshold 为 0（从不锁定）→ 账户不会因密码错误被锁定
   - Lockout threshold 过低（如 3 次）→ 容易触发账户锁定
   - Lockout duration 过长 → 锁定后等待时间过长

### Step 4: 密码过期状态检查

**数据采集**：

> 采集目标：检查用户密码是否过期或即将过期

```powershell
Get-LocalUser | Select-Object Name, Enabled, PasswordExpires, PasswordLastSet, LastLogon |
  Format-Table -AutoSize
```

**分析思路**：

1. 检查密码过期状态：
   - 正常：所有启用用户密码未过期
   - 异常：存在启用用户密码已过期 → **根因**：密码已过期，可能导致 RDP 登录失败，**严重程度**：Warning

### Step 5: 内置 Administrator 账户检查

**数据采集**：

> 采集目标：确认内置 Administrator 账户是否存在

```powershell
Get-CimInstance -ClassName Win32_UserAccount -Filter "LocalAccount=True AND Name='Administrator'" |
  Select-Object Name, Disabled, Lockout, SID |
  Format-Table -AutoSize
```

**分析思路**：

1. 检查 Administrator 账户存在性：
   - 正常：Administrator 账户存在
   - 异常：Administrator 账户不存在 → **根因**：内置管理员账户缺失（AdminUserNotExist），**严重程度**：Warning

> 阿里云 ECS 实例通常通过 Administrator 账户进行密码重置和管理操作。

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | 账户锁定且用户报告 RDP 登录失败 | → [rdp-auth.md](references/rdp-auth.md)（检查远程登录权限与 ForceGuest 配置） |
| 条件跳转 | 域账户相关问题（域登录失败、Secure Channel 异常） | → [identity-ad.md](references/identity-ad.md) |
| 条件跳转 | 账户被禁用且涉及组策略 | → [system-gpo.md](references/system-gpo.md) |
| 条件跳转 | 密码过期或锁定策略导致 Kerberos 认证异常 | → [identity-auth.md](references/identity-auth.md) |
| 链式后继 | 本文件未确认根因，需进一步检查用户权限与组成员关系 | → [identity-permission.md](references/identity-permission.md) |

## 修复建议

### Fix 1: 解锁账户（RDPAccountLocked）

**适用根因**：RDPAccountLocked

```powershell
#requires -RunAsAdministrator
# Unlock local user account (lockout caused by too many failed password attempts)
$user = [ADSI]"WinNT://$env:COMPUTERNAME/<UserName>,user"
$user.IsAccountLocked = $false
$user.SetInfo()

# For domain user account (requires ActiveDirectory module)
# Unlock-ADAccount -Identity <UserName>
```

**验证方法**：

```powershell
$user = [ADSI]"WinNT://$env:COMPUTERNAME/<UserName>,user"
[PSCustomObject]@{
    AccountLocked = $user.IsAccountLocked
} | Format-List
```

预期结果：`AccountLocked` 显示 `False`

### Fix 2: 启用被禁用的账户（RDPAccountDisabled）

**适用根因**：RDPAccountDisabled

```powershell
Enable-LocalUser -Name '<UserName>'
```

**验证方法**：

```powershell
Get-LocalUser -Name '<UserName>' | Select-Object Name, Enabled | Format-List
```

预期结果：`Enabled` 显示 `True`

### Fix 3: 重置过期密码

**适用场景**：密码已过期

```powershell
# Reset password via Cloud Assistant
Set-LocalUser -Name '<UserName>' -Password (ConvertTo-SecureString '<NewPassword>' -AsPlainText -Force)
# Or reset password via ECS console
```

**验证方法**：

```powershell
Get-LocalUser -Name '<UserName>' | Select-Object Name, PasswordExpires, PasswordRequired | Format-List
```

预期结果：命令成功执行，账户密码已更新

### Fix 4: 创建 Administrator 账户（AdminUserNotExist）

**适用根因**：AdminUserNotExist

```powershell
New-LocalUser -Name 'Administrator' -Description 'Built-in Administrator account' -NoPassword
Add-LocalGroupMember -Group 'Administrators' -Member 'Administrator'
# Set password via ECS console or Cloud Assistant
```

**验证方法**：

```powershell
Get-LocalUser -Name 'Administrator' | Select-Object Name, SID, Enabled | Format-List
Get-LocalGroupMember -Group 'Administrators' | Where-Object { $_.Name -eq 'Administrator' } | Format-Table -AutoSize
```

预期结果：Administrator 账户存在、SID 以 -500 结尾、属于 Administrators 组
