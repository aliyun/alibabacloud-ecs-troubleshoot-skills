# Desktop Application 诊断

## 功能说明

诊断 Windows .NET Framework 状态、MSI 安装/卸载和 COM/DCOM 组件注册。覆盖 3 个已知问题项。

**输入**：用户问题描述（必选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 应用启动报错"未能加载文件或程序集" | Step 1 (.NET Framework 状态) |
| 软件安装中断或卸载残留 | Step 2 (MSI 安装/卸载状态) |
| 应用报错 COM 组件未注册或权限不足 | Step 3 (COM/DCOM 组件注册) |

## 诊断步骤

### Step 1: 检查 .NET Framework 状态

**数据采集**：

> 采集目标：获取已安装的 .NET Framework 版本及安装状态

```powershell
# Check installed .NET Framework versions
$ndpPath = 'HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP'
# .NET Framework 4.x
$v4Full = Get-ItemProperty -Path "$ndpPath\v4\Full" -ErrorAction SilentlyContinue
if ($v4Full) {
    $releaseMap = @{528040='.NET 4.8'; 461808='.NET 4.7.2'; 461308='.NET 4.7.1'; 460798='.NET 4.7'; 394802='.NET 4.6.2'; 394254='.NET 4.6.1'; 393295='.NET 4.6'; 379893='.NET 4.5.2'; 378675='.NET 4.5.1'; 378389='.NET 4.5'}
    $friendlyName = ''
    foreach ($r in ($releaseMap.GetEnumerator() | Sort-Object Key -Descending)) {
        if ($v4Full.Release -ge $r.Key) { $friendlyName = $r.Value; break }
    }
    Write-Host ".NET Framework 4.x: Version=$($v4Full.Version), Release=$($v4Full.Release) ($friendlyName)"
} else {
    Write-Host ".NET Framework 4.x: Not installed"
}
# .NET Framework 3.5
$v35 = Get-ItemProperty -Path "$ndpPath\v3.5" -ErrorAction SilentlyContinue
if ($v35) {
    Write-Host ".NET Framework 3.5: Version=$($v35.Version), Install=$($v35.Install)"
} else {
    Write-Host ".NET Framework 3.5: Not installed"
}
```

**分析思路**：

1. 检查 .NET Framework 4.x 安装状态：
   - 正常：已安装且版本满足应用需求
   - 异常：未安装或版本过低 → **根因**：.NET Framework 未安装或版本不满足应用需求，**严重程度**：Critical
2. 检查 .NET Framework 3.5：
   - 某些老旧应用需要 3.5，如未安装且应用报错 → **根因**：.NET Framework 3.5 未安装，**严重程度**：Warning

### Step 2: 检查 MSI 安装/卸载状态

**数据采集**：

> 采集目标：检查 Windows Installer 服务状态和最近的安装失败事件

```powershell
# Check Windows Installer service
Get-Service -Name msiserver | Select-Object Name, DisplayName, Status, StartType | Format-Table -AutoSize
# Check recent MSI installation failure events
Get-WinEvent -FilterHashtable @{LogName='Application'; Level=2,3} -MaxEvents 10 -ErrorAction SilentlyContinue |
    Where-Object { $_.ProviderName -eq 'MsiInstaller' } |
    Select-Object TimeCreated, Id, LevelDisplayName, Message | Format-Table -AutoSize
# Check PendingFileRenameOperations (installation residue)
$pending = (Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager' -Name 'PendingFileRenameOperations' -ErrorAction SilentlyContinue).PendingFileRenameOperations
if ($pending) {
    Write-Host "PendingFileRenameOperations: $($pending.Count) entries (may need reboot)"
} else {
    Write-Host "No pending file rename operations"
}
```

**分析思路**：

1. 检查 MSI 服务状态：
   - 正常：msiserver 服务启动类型为 Manual（按需启动）
   - 异常：服务被禁用 → **根因**：Windows Installer 服务被禁用，无法安装/卸载程序，**严重程度**：Critical
2. 检查 MSI 失败事件：
   - 关注错误事件中的具体错误代码和产品名称
3. 检查未完成的文件替换操作：
   - 如果存在大量 PendingFileRenameOperations → **根因**：安装残留操作未完成，可能需要重启，**严重程度**：Warning

### Step 3: 检查 COM/DCOM 组件注册

**数据采集**：

> 采集目标：检查 COM/DCOM 配置和常见问题

```powershell
# Check DCOM service status
Get-Service -Name 'RpcSs' | Select-Object Name, DisplayName, Status, StartType | Format-Table -AutoSize
# Check DCOM permission related events (EventID 10016)
Get-WinEvent -FilterHashtable @{LogName='System'; Id=10016} -MaxEvents 5 -ErrorAction SilentlyContinue |
    Select-Object TimeCreated, Id, Message | Format-Table -AutoSize
# Check COM registration server status
$ole = Get-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Ole' -ErrorAction SilentlyContinue
if ($ole) {
    Write-Host "DCOM EnableDCOM: $($ole.EnableDCOM)"
    Write-Host "DCOM LegacyAuthenticationLevel: $($ole.LegacyAuthenticationLevel)"
}
```

**分析思路**：

1. 检查 RPC 服务状态：
   - 正常：RpcSs 服务运行中
   - 异常：RpcSs 未运行 → **根因**：RPC 服务未启动，COM/DCOM 组件调用将失败，**严重程度**：Critical
2. 检查 DCOM 权限事件：
   - EventID 10016 是常见的 DCOM 权限警告，影响特定 COM 组件的调用
   - 频繁出现 → **根因**：DCOM 权限配置不当，相关应用程序功能可能受影响，**严重程度**：Warning
3. 检查 DCOM 是否启用：
   - 正常：EnableDCOM 为 Y
   - 异常：EnableDCOM 为 N → **根因**：DCOM 被禁用，远程 COM 组件调用将失败，**严重程度**：Critical
4. 检查 DCOM 认证级别：
   - 正常：LegacyAuthenticationLevel 不存在（使用系统默认），或值为 2 (Connect) 及以上
   - 异常：LegacyAuthenticationLevel 为 1 (None) → **根因**：DCOM 认证级别过低，COM 组件调用不验证身份，可能导致应用启动失败或安全策略冲突，**严重程度**：Warning

> 如果怀疑杀毒软件干扰应用运行，参见 → [security-malware.md](references/security-malware.md)

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | 怀疑杀毒软件干扰应用运行 | → [security-malware.md](references/security-malware.md) |
| 链式后继 | 本文件未确认根因 | → [desktop-printing.md](references/desktop-printing.md) |

## 修复建议

### 根因: .NET Framework 未安装

**修复操作**：

```powershell
# Install .NET Framework 3.5 (requires Windows installation source)
Enable-WindowsOptionalFeature -Online -FeatureName NetFx3 -All -NoRestart
# .NET Framework 4.8 needs to be downloaded from Microsoft official website
```

**验证方法**：

```powershell
$v4 = (Get-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full' -ErrorAction SilentlyContinue).Release
Write-Host ".NET 4.x Release: $v4"
```

预期结果：Release 值存在且满足应用需求

**风险说明**：安装 .NET Framework 3.5 可能需要 Windows 安装源（SxS 文件夹）。.NET 4.8 安装需要重启。

### 根因: Windows Installer 服务被禁用

**修复操作**：

```powershell
Set-Service -Name msiserver -StartupType Manual
Start-Service msiserver
```

**验证方法**：

```powershell
Get-Service msiserver | Select-Object Name, Status, StartType | Format-Table -AutoSize
```

预期结果：Status 为 Running，StartType 为 Manual

**风险说明**：Windows Installer 服务通常为手动启动，意味着仅在需要时自动启动，设置为 Manual 是推荐配置。
