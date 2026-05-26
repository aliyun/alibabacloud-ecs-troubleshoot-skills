# Desktop Printing 诊断

## 功能说明

诊断 Windows Print Spooler 服务、打印驱动安装和打印输出状态。覆盖 3 个已知问题项。

**输入**：用户问题描述（必选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 打印服务停止、打印队列卡住 | Step 1 (Print Spooler 服务) |
| 无法添加打印机、驱动安装报错 | Step 2 (打印驱动安装) |
| 打印任务提交后无输出或输出乱码 | Step 3 (打印输出状态) |

## 诊断步骤

### Step 1: 检查 Print Spooler 服务

**数据采集**：

> 采集目标：获取 Print Spooler 服务状态及打印队列信息

```powershell
# Check Print Spooler service status
Get-Service -Name Spooler | Select-Object Name, DisplayName, Status, StartType | Format-Table -AutoSize
# Check dependent services
$deps = (Get-Service -Name Spooler).DependentServices
if ($deps) {
    Write-Host "Dependent services:"
    $deps | Select-Object Name, Status | Format-Table -AutoSize
}
# Check print jobs in queue
$printJobs = Get-CimInstance Win32_PrintJob -ErrorAction SilentlyContinue
if ($printJobs) {
    Write-Host "Print jobs in queue: $($printJobs.Count)"
    $printJobs | Select-Object Document, JobStatus, Owner, Priority, Size | Format-Table -AutoSize
} else {
    Write-Host "No print jobs in queue"
}
# Check Spooler crash events
Get-WinEvent -FilterHashtable @{LogName='System'; Level=2} -MaxEvents 10 -ErrorAction SilentlyContinue |
    Where-Object { $_.ProviderName -eq 'Service Control Manager' -and ($_.Message -like '*Spooler*' -or $_.Message -like '*Print*') } |
    Select-Object TimeCreated, Id, Message | Format-Table -AutoSize
```

**分析思路**：

1. 检查 Spooler 服务状态：
   - 正常：状态为 Running，启动类型为 Automatic
   - 异常：服务未运行 → **根因**：Print Spooler 服务未启动，打印功能完全不可用，**严重程度**：Critical
   - 异常：服务被禁用 → **根因**：Print Spooler 服务被禁用（可能出于安全加固目的），**严重程度**：Warning
2. 检查打印队列：
   - 异常：大量打印任务堆积且状态为 Error → **根因**：打印队列堵塞，打印任务无法处理，**严重程度**：Warning
3. 检查 Spooler 崩溃事件：
   - 频繁崩溃 → **根因**：Print Spooler 反复崩溃，可能由损坏的打印驱动或打印任务引起，**严重程度**：Critical

### Step 2: 检查打印驱动安装

**数据采集**：

> 采集目标：获取已安装的打印机和打印驱动列表

```powershell
# Get installed printers
$printers = Get-CimInstance Win32_Printer -ErrorAction SilentlyContinue
if ($printers) {
    Write-Host "Installed printers: $($printers.Count)"
    $printers | Select-Object Name, DriverName, PortName, PrinterStatus, Shared | Format-Table -AutoSize
} else {
    Write-Host "No printers installed"
}
# Get installed printer drivers
$drivers = Get-CimInstance Win32_PrinterDriver -ErrorAction SilentlyContinue
if ($drivers) {
    Write-Host "`nInstalled printer drivers: $($drivers.Count)"
    $drivers | Select-Object Name, SupportedPlatform | Format-Table -AutoSize
}
# Check printer ports
$ports = Get-CimInstance Win32_TCPIPPrinterPort -ErrorAction SilentlyContinue
if ($ports) {
    Write-Host "`nTCP/IP Printer Ports:"
    $ports | Select-Object Name, HostAddress, PortNumber | Format-Table -AutoSize
}
```

**分析思路**：

1. 检查打印机状态：
   - 正常：PrinterStatus 为 Idle 或 Printing
   - 异常：PrinterStatus 为 Error 或 Offline → **根因**：打印机状态异常，可能驱动问题或端口配置错误，**严重程度**：Warning
2. 检查驱动安装：
   - 异常：打印机无对应驱动 → **根因**：打印驱动未安装，无法正常打印，**严重程度**：Critical
3. 检查 TCP/IP 端口：
   - 异常：端口地址配置错误或不可达 → **根因**：打印端口配置异常，**严重程度**：Warning

### Step 3: 检查打印输出状态

**数据采集**：

> 采集目标：检查打印 Spool 目录和最近打印事件

```powershell
# Check Spool directory size and files
$spoolDir = "$env:SystemRoot\System32\spool\PRINTERS"
if (Test-Path $spoolDir) {
    $spoolFiles = Get-ChildItem -Path $spoolDir -ErrorAction SilentlyContinue
    $totalSize = ($spoolFiles | Measure-Object -Property Length -Sum).Sum
    Write-Host "Spool directory: $spoolDir"
    Write-Host "Files: $($spoolFiles.Count), Total size: $([math]::Round($totalSize/1MB, 2)) MB"
} else {
    Write-Host "Spool directory not found: $spoolDir"
}
# Check print related events
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-PrintService/Operational'; Level=2,3} -MaxEvents 10 -ErrorAction SilentlyContinue |
    Select-Object TimeCreated, Id, LevelDisplayName, Message | Format-Table -AutoSize
# Fallback: if Operational log is not available, check System log
Get-WinEvent -FilterHashtable @{LogName='System'} -MaxEvents 5 -ErrorAction SilentlyContinue |
    Where-Object { $_.ProviderName -eq 'Microsoft-Windows-PrintService' } |
    Select-Object TimeCreated, Id, LevelDisplayName, Message | Format-Table -AutoSize
```

**分析思路**：

1. 检查 Spool 目录：
   - 正常：目录存在，文件数量少（无堆积）
   - 异常：大量文件堆积或占用大量空间 → **根因**：打印 Spool 文件堆积，可能导致 Spooler 服务崩溃或磁盘空间不足，**严重程度**：Warning
2. 检查打印事件日志：
   - 关注错误事件，特别是驱动加载失败、端口通信失败等
   - 出现此类事件 → 关联到具体根因（驱动问题/网络问题/权限问题）

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | 打印端口网络不可达 | → [networking-tcpip.md](references/networking-tcpip.md) |
| 条件跳转 | Spool 目录权限问题 | → [identity-permission.md](references/identity-permission.md) |
| 链式后继 | 本文件未确认根因 | → [device-driver.md](references/device-driver.md) |

## 修复建议

### 根因: Print Spooler 服务未启动

**修复操作**：

```powershell
Set-Service -Name Spooler -StartupType Automatic
Start-Service Spooler
```

**验证方法**：

```powershell
Get-Service Spooler | Select-Object Name, Status, StartType | Format-Table -AutoSize
```

预期结果：Status 为 Running，StartType 为 Automatic

**风险说明**：Print Spooler 存在历史安全漏洞（PrintNightmare 等），如果服务器不需要打印功能，建议保持禁用状态。

### 根因: 打印队列堵塞

**修复操作**：

```powershell
# Clear print queue
Stop-Service Spooler -Force
Remove-Item -Path "$env:SystemRoot\System32\spool\PRINTERS\*" -Force -ErrorAction SilentlyContinue
Start-Service Spooler
```

**验证方法**：

```powershell
$spoolFiles = Get-ChildItem -Path "$env:SystemRoot\System32\spool\PRINTERS" -ErrorAction SilentlyContinue
Write-Host "Spool files remaining: $($spoolFiles.Count)"
Get-Service Spooler | Select-Object Name, Status | Format-Table -AutoSize
```

预期结果：Spool 目录清空，Spooler 服务运行中

**风险说明**：清空打印队列会删除所有未完成的打印任务，操作前确认无重要打印任务。
