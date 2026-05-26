# System Management 诊断

## 功能说明

诊断 Windows PowerShell 执行策略、WinRM、WMI 仓库、事件日志服务和 MMC 控制台。覆盖 5 个已知问题项。

**输入**：用户问题描述（必选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 云助手命令执行失败、远程管理连接失败 | Step 2 (WinRM 服务) |
| 脚本无法运行、提示执行策略限制 | Step 1 (PowerShell 执行策略) |
| WMI 查询失败、监控工具无法获取系统信息 | Step 3 (WMI 仓库状态) |
| 事件查看器无法打开或日志损坏 | Step 4 (事件日志服务) |
| 设备管理器、磁盘管理等工具打不开 | Step 5 (MMC 控制台) |

## 诊断步骤

### Step 1: PowerShell 执行策略检查

**数据采集**：

> 采集目标：获取当前 PowerShell 执行策略配置

```powershell
Get-ExecutionPolicy -List | Format-Table -AutoSize
```

**分析思路**：

1. 检查执行策略级别：
   - 正常：MachinePolicy/LocalMachine 为 RemoteSigned 或 Unrestricted
   - 异常：任一级别为 Restricted 或 AllSigned → 可能阻止脚本执行，**严重程度**：Warning
   - 任一级别为 Undefined（且有效策略为 Restricted）：需确认实际生效的策略级别

### Step 2: WinRM 服务检查

**数据采集**：

> 采集目标：检查 WinRM 服务状态和配置

```powershell
# Service status
Get-Service -Name WinRM -ErrorAction SilentlyContinue |
  Select-Object Name, Status, StartType |
  Format-Table -AutoSize
# Listener configuration
winrm enumerate winrm/config/listener 2>&1
```

**分析思路**：

1. 检查 WinRM 服务状态：
   - 正常：WinRM Running 且监听器存在
   - 异常：WinRM 停止或被禁用 → **根因**：云助手和远程管理可能失败，**严重程度**：Critical
2. 检查 WinRM 监听器：
   - 正常：监听器已配置
   - 异常：无监听器 → **根因**：WinRM 未配置监听端口，**严重程度**：Warning

### Step 3: WMI 仓库状态检查

**数据采集**：

> 采集目标：验证 WMI 仓库完整性和服务状态

```powershell
# WMI service status
Get-Service -Name Winmgmt -ErrorAction SilentlyContinue |
  Select-Object Name, Status, StartType |
  Format-Table -AutoSize
# WMI repository verification
winmgmt /verifyrepository 2>&1
```

```powershell
# Quick WMI query test
Get-CimInstance -ClassName Win32_OperatingSystem -ErrorAction Stop |
  Select-Object Caption, Version, BuildNumber |
  Format-Table -AutoSize
```

**分析思路**：

1. 检查 WMI 服务状态：
   - 正常：Winmgmt Running 且 /verifyrepository 输出表示仓库一致
   - 异常：Winmgmt 停止 → **根因**：WMI 服务未运行，**严重程度**：Critical
2. 检查 WMI 仓库完整性：
   - 正常：仓库一致
   - 异常：/verifyrepository 输出表示仓库不一致 → **根因**：WMI 仓库损坏，**严重程度**：Warning
3. 检查 WMI 查询可用性：
   - 正常：Get-CimInstance 查询成功
   - 异常：Get-CimInstance 报错 → **根因**：WMI 查询失败，**严重程度**：Critical

### Step 4: 事件日志服务检查

**数据采集**：

> 采集目标：检查事件日志服务状态

```powershell
Get-Service -Name EventLog -ErrorAction SilentlyContinue |
  Select-Object Name, Status, StartType |
  Format-Table -AutoSize
# Check log file size
Get-WinEvent -ListLog System, Application, Security -ErrorAction SilentlyContinue |
  Select-Object LogName, @{N='SizeMB';E={[math]::Round($_.FileSize/1MB,1)}}, MaximumSizeInBytes, RecordCount, IsLogFull |
  Format-Table -AutoSize
```

**分析思路**：

1. 检查事件日志服务：
   - 正常：EventLog Running 且日志未满
   - 异常：EventLog 停止 → **根因**：事件日志服务未运行，**严重程度**：Warning
   - 异常：IsLogFull=True → **根因**：日志文件已满，新事件无法记录，**严重程度**：Warning

### Step 5: MMC 控制台检查

**数据采集**：

> 采集目标：检查 MMC 相关组件和 .NET Framework 状态

```powershell
# Check if MMC executable exists
Test-Path -Path "$env:SystemRoot\System32\mmc.exe" | Format-Table -AutoSize
# Check .NET Framework version (required by MMC)
Get-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full' -ErrorAction SilentlyContinue |
  Select-Object Release, Version |
  Format-Table -AutoSize
```

**分析思路**：

1. 检查 MMC 组件完整性：
   - 正常：mmc.exe 存在且 .NET Framework 已安装
   - 异常：mmc.exe 不存在 → 系统文件缺失
   - 异常：.NET Framework 未安装或版本过低 → MMC 插件可能无法加载

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | WMI 服务异常影响云助手 | → [cloud-vminit.md](references/cloud-vminit.md)（vminit 服务检查） |
| 条件跳转 | 执行策略阻止脚本运行 | → [system-gpo.md](references/system-gpo.md)（组策略检查） |
| 链式后继 | 本文件未确认根因 | → [cloud-vminit.md](references/cloud-vminit.md) |

## 修复建议

### Fix 1: 修改执行策略

**适用场景**：PowerShell 脚本被执行策略阻止

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine -Force
```

**验证方法**：

```powershell
Get-ExecutionPolicy -Scope LocalMachine
```

预期结果：返回 `RemoteSigned` 或 `Unrestricted`

### Fix 2: 启用并配置 WinRM

**适用场景**：WinRM 服务未运行或未配置

```powershell
# Quick WinRM configuration (start service + create listener + firewall rule)
winrm quickconfig -force
```

**验证方法**：

```powershell
Test-WSMan -ErrorAction SilentlyContinue
# 或检查 WinRM 服务状态
Get-Service WinRM | Select-Object Name, Status
```

预期结果：`Test-WSMan` 返回协议版本信息，WinRM 服务状态为 `Running`

### Fix 3: 修复 WMI 仓库

**适用场景**：WMI 仓库损坏

```powershell
# Stop WMI service
Stop-Service Winmgmt -Force
# Rebuild WMI repository
winmgmt /resetrepository
# Restart service
Start-Service Winmgmt
```

**验证方法**：

```powershell
Get-CimInstance -ClassName Win32_ComputerSystem | Select-Object Name
```

预期结果：成功返回计算机名称，无 WMI 错误

> 重建 WMI 仓库会重置所有 WMI 类定义，某些第三方应用可能需要重新注册 WMI 提供程序。

### Fix 4: 修复事件日志服务

**适用场景**：事件日志服务停止或日志已满

```powershell
# Start Event Log service
Start-Service EventLog
# Clear full logs (after backup)
Clear-EventLog -LogName System
Clear-EventLog -LogName Application
```

**验证方法**：

```powershell
Get-Service EventLog | Select-Object Name, Status
Get-WinEvent -LogName System -MaxEvents 1 | Select-Object TimeCreated, Id, LevelDisplayName, Message | Format-List
```

预期结果：EventLog 服务状态为 `Running`，`Get-WinEvent` 成功返回最近事件
