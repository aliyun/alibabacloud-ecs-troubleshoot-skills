# Identity Permission 诊断

## 功能说明

诊断 Windows 系统盘根目录权限、远程登录权限、Temp 文件夹权限、ForceGuest 配置、用户组成员关系和 Guest 账户状态。覆盖 6 个已知问题项。

**输入**: 用户问题描述(必选)
**输出**: 根因列表(root_cause / severity / evidence / explanation / fix)

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 登录后黑屏或闪退、Explorer 无法加载 | Step 1 (系统盘根目录权限) |
| 提示 "you need the right to sign in through RDS" | Step 2 (远程登录权限) |
| 更新安装失败、报错 0x80070005 | Step 3 (Temp 文件夹权限) |
| 本地账户远程登录失败、被强制映射为 Guest | Step 4 (ForceGuest 配置) |
| 非预期账户被加入管理员组、或管理员权限不足 | Step 5 (用户组成员关系) |
| 存在未授权访问风险 | Step 6 (Guest 账户状态) |

## 诊断步骤

### Step 1: 系统盘根目录权限检查

**数据采集**：

> 采集目标：检查 C:\ 根目录对 BUILTIN\Users 和 NT AUTHORITY\SERVICE 的访问权限

```powershell
(Get-Acl 'C:\').Access | Format-Table IdentityReference, FileSystemRights, AccessControlType, IsInherited, InheritanceFlags, PropagationFlags -AutoSize
```

**分析思路**：

1. 检查 BUILTIN\Users 权限：
   - 正常：BUILTIN\Users 具有读取+执行权限
   - 异常：BUILTIN\Users 无读取或执行权限 → **根因**：系统盘访问被拒绝（SystemDiskAccessDenied），**严重程度**：Critical
2. 检查 NT AUTHORITY\SERVICE 权限：
   - 正常：NT AUTHORITY\SERVICE 无显式拒绝
   - 异常：NT AUTHORITY\SERVICE 被显式拒绝 → **根因**：服务账户无法访问系统盘，**严重程度**：Critical

### Step 2: 远程登录权限检查

**数据采集**：

> 采集目标：检查“允许通过远程桌面服务登录”权限配置

```powershell
# Check Remote Desktop Users group members
Get-LocalGroupMember -Group 'Remote Desktop Users' -ErrorAction SilentlyContinue |
  Select-Object Name, ObjectClass, PrincipalSource |
  Format-Table -AutoSize
# Check Administrators group members (have RDP permission by default)
Get-LocalGroupMember -Group 'Administrators' -ErrorAction SilentlyContinue |
  Select-Object Name, ObjectClass, PrincipalSource |
  Format-Table -AutoSize
```

**分析思路**：

1. 检查目标用户的远程登录权限：
   - 正常：目标用户在 Administrators 或 Remote Desktop Users 组 → 具备远程登录权限
   - 异常：目标用户不在任何组 → 缺少远程登录权限

### Step 3: Temp 文件夹权限检查

**数据采集**：

> 采集目标：检查 Temp 文件夹对 Administrators 的完全控制权限

```powershell
(Get-Acl $env:TEMP).Access | Format-Table IdentityReference, FileSystemRights, AccessControlType, IsInherited, InheritanceFlags, PropagationFlags -AutoSize
```

**分析思路**：

1. 检查 Temp 文件夹权限：
   - 正常：BUILTIN\Administrators 具有完全控制
   - 异常：BUILTIN\Administrators 无完全控制 → **根因**：Temp 文件夹权限不足（TempFolderAccessDenied），**严重程度**：Warning

### Step 4: ForceGuest 配置检查

**数据采集**：

> 采集目标：检查本地用户是否被强制映射为 Guest

```powershell
Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Lsa' -Name 'ForceGuest' -ErrorAction SilentlyContinue |
  Select-Object ForceGuest |
  Format-Table -AutoSize
```

**分析思路**：

1. 检查 ForceGuest 配置：
   - 正常：ForceGuest 不存在或为 0 → 网络访问使用实际账户
   - 异常：ForceGuest 非 0 → **根因**：本地用户远程访问被强制映射为 Guest（ForceGuestAccess），**严重程度**：Warning

> ForceGuest 启用后，所有本地用户的远程访问都会被映射为 Guest 账户，导致权限严重受限。

### Step 5: 用户组成员关系检查

**数据采集**：

> 采集目标：检查关键用户组的成员

```powershell
foreach ($group in @('Administrators','Remote Desktop Users','Users')) {
  Write-Output "=== $group ==="
  Get-LocalGroupMember -Group $group -ErrorAction SilentlyContinue |
    Select-Object Name, ObjectClass | Format-Table -AutoSize
}
```

**分析思路**：

1. 检查用户组成员关系：
   - 正常：组成员符合预期
   - 异常：发现非预期的管理员账户 → 需确认是否合法

### Step 6: Guest 账户状态检查

**数据采集**：

> 采集目标：检查 Guest 账户是否被启用

```powershell
Get-LocalUser -Name 'Guest' -ErrorAction SilentlyContinue |
  Select-Object Name, Enabled, LastLogon |
  Format-Table -AutoSize
```

**分析思路**：

1. 检查 Guest 账户状态：
   - 正常：Guest 账户 Disabled（安全最佳实践）
   - 异常：Guest 账户 Enabled → **根因**：存在未授权访问风险，**严重程度**：Warning

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Temp 权限问题影响 Windows Update | → [system-update.md](references/system-update.md) |
| 条件跳转 | 系统盘权限影响 RDP | → [rdp-auth.md](references/rdp-auth.md) |
| 链式后继 | 本文件未确认根因 | → [system-gpo.md](references/system-gpo.md) |

## 修复建议

### Fix 1: 修复系统盘根目录权限（SystemDiskAccessDenied）

**适用根因**：SystemDiskAccessDenied

```powershell
icacls 'C:\' /grant 'BUILTIN\Users:(OI)(CI)(RX)'
icacls 'C:\' /grant 'NT AUTHORITY\SERVICE:(OI)(CI)(RX)'
```

**验证方法**：

```powershell
# Verify using Get-Acl to avoid parsing cmd output across locales
$acl = Get-Acl 'C:\'
$acl.Access | Where-Object { $_.IdentityReference -match 'BUILTIN\\Users|NT AUTHORITY\\SERVICE' } | Select-Object IdentityReference, FileSystemRights, AccessControlType | Format-Table -AutoSize
```

预期结果：显示 `BUILTIN\Users:(OI)(CI)(RX)` 和 `NT AUTHORITY\SERVICE:(OI)(CI)(RX)`

### Fix 2: 修复 Temp 文件夹权限（TempFolderAccessDenied）

**适用根因**：TempFolderAccessDenied

```powershell
icacls "$env:TEMP" /grant 'BUILTIN\Administrators:(OI)(CI)(F)'
icacls "$env:TEMP" /grant 'BUILTIN\Users:(OI)(CI)(RX)'
```

**验证方法**：

```powershell
# Verify using Get-Acl to avoid parsing cmd output across locales
$acl = Get-Acl $env:TEMP
$acl.Access | Where-Object { $_.IdentityReference -match 'BUILTIN\\Administrators|BUILTIN\\Users' } | Select-Object IdentityReference, FileSystemRights, AccessControlType | Format-Table -AutoSize
```

预期结果：显示 `BUILTIN\Administrators:(OI)(CI)(F)` 和 `BUILTIN\Users:(OI)(CI)(RX)`

### Fix 3: 禁用 ForceGuest（ForceGuestAccess）

**适用根因**：ForceGuestAccess

```powershell
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Lsa' -Name 'ForceGuest' -Value 0 -Type DWord
```

**验证方法**：

```powershell
Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Lsa' -Name 'ForceGuest' | Select-Object ForceGuest
```

预期结果：`ForceGuest` 值为 `0`

### Fix 4: 禁用 Guest 账户

**适用场景**：Guest 账户被启用

```powershell
Disable-LocalUser -Name 'Guest'
```

**验证方法**：

```powershell
Get-LocalUser -Name 'Guest' | Select-Object Name, Enabled | Format-List
```

预期结果：`Enabled` 显示 `False`
