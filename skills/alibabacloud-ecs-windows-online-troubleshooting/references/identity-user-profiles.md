# Identity User Profiles 诊断

## 功能说明

诊断 Windows 用户配置文件损坏、文件夹重定向失败以及自定义/默认用户配置文件导致的性能问题。覆盖 3 个已知问题项。

**输入**: 用户问题描述(必选)
**输出**: 根因列表(root_cause / severity / evidence / explanation / fix)

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 登录后加载临时配置文件、桌面设置全部丢失 | Step 1 (用户配置文件状态) |
| 桌面/文档文件夹指向不可达路径 | Step 2 (文件夹重定向) |
| 登录缓慢、登录后系统性能下降、自定义/默认配置文件异常应用 | Step 3 (自定义/默认用户配置文件检查) |

## 诊断步骤

### Step 1: 用户配置文件状态检查

**数据采集**：

> 采集目标：检查用户配置文件注册表项，确认是否存在损坏或临时配置文件

```powershell
Get-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList\*' -ErrorAction SilentlyContinue |
  Select-Object PSChildName, ProfileImagePath, State,
    @{N='HasBak';E={Test-Path -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList\$($_.PSChildName).bak"}} |
  Format-Table -AutoSize
```

**分析思路**：

1. 检查配置文件注册表项：
   - 正常：所有配置文件 State 正常且无 .bak 副本
   - 异常：存在 .bak 结尾的配置文件注册表项 → **根因**：用户配置文件损坏，系统加载了临时配置文件，**严重程度**：Critical
   - 异常：ProfileImagePath 指向 TEMP 目录 → **根因**：正在使用临时配置文件，**严重程度**：Critical

> 当用户配置文件损坏时，Windows 会创建临时配置文件，用户会发现桌面设置、文件等全部丢失。

### Step 2: 文件夹重定向检查

**数据采集**：

> 采集目标：检查用户 Shell 文件夹重定向配置

```powershell
Get-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\User Shell Folders' -ErrorAction SilentlyContinue |
  Select-Object Desktop, Personal, 'My Pictures', '{374DE290-123F-4565-9164-39C4925E467B}' |
  Format-Table -AutoSize
```

```powershell
# Check if redirection target is accessible
$desktop = (Get-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\User Shell Folders').Desktop
if ($desktop) { Test-Path -Path $desktop }
```

**分析思路**：

1. 检查 Shell Folders 可访问性：
   - 正常：Shell Folders 路径均可访问
   - 异常：重定向目标不可达（网络路径断开） → **根因**：文件夹重定向失败，用户无法访问桌面/文档，**严重程度**：Warning

### Step 3: 自定义/默认用户配置文件 WebCache 检查

**数据采集**：

> 采集目标：检查 Event ID 454（ESENT 数据库恢复/还原失败）。WebCacheLock.dat 和 WebCache 文件夹在所有用户配置文件中正常存在，不应作为诊断依据。KB 4056823 的根因是：自定义默认用户配置文件时，源账户的缓存数据库锁定副本被复制进 Default，新用户登录时数据库无法初始化，产生 Event ID 454。

```powershell
# Check ESENT Event ID 454 errors (key indicator)
Get-WinEvent -FilterHashtable @{LogName='Application'; Id=454} -MaxEvents 10 -ErrorAction SilentlyContinue |
    Select-Object TimeCreated, Message | Format-List
```

**分析思路**：

1. 检查 Event ID 454：
   - 正常：无 Event ID 454 记录
   - 异常：存在 Event ID 454，且 Message 含 "Database recovery/restore failed" → **根因**：默认用户配置文件包含其他用户缓存数据库的锁定副本，新用户登录时数据库初始化失败，**严重程度**：Critical

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | 重定向目标为网络路径且不可达 | → [networking-dns.md](references/networking-dns.md) |
| 条件跳转 | 登录缓慢 / 性能异常，且 Step 3 未确认根因 | → [performance-slow.md](references/performance-slow.md) |
| 链式后继 | 本文件未确认根因 | → [identity-permission.md](references/identity-permission.md) |

## 修复建议

### Fix 1: 修复损坏的用户配置文件

**适用场景**：配置文件损坏，加载了临时配置文件

1. 在注册表中导航到 `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList`
2. 找到对应用户 SID 的项
3. 如果存在 `.bak` 副本：
   - 删除不带 `.bak` 的项
   - 将 `.bak` 项重命名为去掉 `.bak`
   - 将 `State` 值设为 `0`
4. 重启系统

**验证方法**：

```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\ProfileList\<UserSID>" -Name State, ProfileImagePath -ErrorAction SilentlyContinue | Select-Object State, ProfileImagePath
```

预期结果：`State` 值为 `0`，`ProfileImagePath` 指向正确路径，无 `.bak` 后缀项

### Fix 2: 修复文件夹重定向

**适用场景**：重定向目标不可达

```powershell
# Redirect Desktop back to local default path
Set-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\User Shell Folders' -Name 'Desktop' -Value '%USERPROFILE%\Desktop'
# Also handle other folders (Documents, etc.)
Set-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\User Shell Folders' -Name 'Personal' -Value '%USERPROFILE%\Documents'
```

**验证方法**：

```powershell
Get-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\User Shell Folders' -Name Desktop, Personal | Select-Object Desktop, Personal
```

预期结果：`Desktop` 值为 `%USERPROFILE%\Desktop`，`Personal` 值为 `%USERPROFILE%\Documents`

### Fix 3: 清除 Default 配置文件及受影响用户的 WebCache 锁定副本

**适用场景**：Default 配置文件或用户配置文件中存在 WebCacheLock.dat / WebCache 文件夹

**第 1 步：以管理员身份登录，删除 Default 配置文件中的隐藏文件和文件夹**

删除以下文件和文件夹（若存在）：
- `C:\Users\Default\AppData\Local\Microsoft\Windows\WebCacheLock.dat`
- `C:\Users\Default\AppData\Local\Microsoft\Windows\WebCache`

```powershell
#requires -RunAsAdministrator
# Fix Step 1: Remove WebCacheLock.dat and WebCache from Default profile
# Risk: File deletion is irreversible
# Verify: New users can log on without performance issues

$defaultLock  = "$env:SystemDrive\Users\Default\AppData\Local\Microsoft\Windows\WebCacheLock.dat"
$defaultCache = "$env:SystemDrive\Users\Default\AppData\Local\Microsoft\Windows\WebCache"

if (Test-Path $defaultLock)  { Remove-Item -Path $defaultLock  -Force; Write-Host "Removed: $defaultLock" }
if (Test-Path $defaultCache) { Remove-Item -Path $defaultCache -Recurse -Force; Write-Host "Removed: $defaultCache" }
```

**第 2 步：确认每个受影响用户已完全注销，删除其配置文件中的隐藏文件和文件夹**

对每个受影响用户，删除以下文件和文件夹（将 `<affectedUserFolder>` 替换为实际用户文件夹名，如 `Administrator`）：
- `C:\Users\<affectedUserFolder>\AppData\Local\Microsoft\Windows\WebCacheLock.dat`
- `C:\Users\<affectedUserFolder>\AppData\Local\Microsoft\Windows\WebCache`

```powershell
#requires -RunAsAdministrator
# Fix Step 2: Remove WebCacheLock.dat and WebCache from affected user profiles
# Prerequisite: ensure the affected user is fully logged off before running

$profileRoot = "$env:SystemDrive\Users"
Get-ChildItem -Path $profileRoot -Directory -ErrorAction SilentlyContinue | ForEach-Object {
    $lockFile    = Join-Path $_.FullName 'AppData\Local\Microsoft\Windows\WebCacheLock.dat'
    $cacheFolder = Join-Path $_.FullName 'AppData\Local\Microsoft\Windows\WebCache'
    if (Test-Path $lockFile)    { Remove-Item -Path $lockFile    -Force;          Write-Host "Removed: $lockFile" }
    if (Test-Path $cacheFolder) { Remove-Item -Path $cacheFolder -Recurse -Force; Write-Host "Removed: $cacheFolder" }
}
```

**验证方法**：

```powershell
# Verify all WebCacheLock.dat and WebCache have been removed
$profileRoot = "$env:SystemDrive\Users"
Get-ChildItem -Path $profileRoot -Directory -ErrorAction SilentlyContinue | ForEach-Object {
    $lockFile    = Join-Path $_.FullName 'AppData\Local\Microsoft\Windows\WebCacheLock.dat'
    $cacheFolder = Join-Path $_.FullName 'AppData\Local\Microsoft\Windows\WebCache'
    [PSCustomObject]@{
        UserFolder      = $_.Name
        WebCacheLockDat = Test-Path -Path $lockFile
        WebCacheFolder  = Test-Path -Path $cacheFolder
    }
} | Format-Table -AutoSize
```

预期结果：WebCacheLockDat = False，WebCacheFolder = False；新用户登录后登录速度正常，无 Event ID 454 错误

**风险说明**：执行前必须确认受影响用户已完全注销（配置文件已卸载），否则删除可能失败或导致数据丢失。
