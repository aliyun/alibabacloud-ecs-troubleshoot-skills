# Device Driver 诊断

## 功能说明

诊断 Windows 设备驱动状态，包括驱动安装、版本、签名状态、设备管理器异常（黄色感叹号/错误代码）、驱动服务状态。覆盖通用设备驱动问题。

**输入**：用户问题描述（必选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 设备管理器黄色感叹号 | Step 1 (设备状态和错误代码) |
| 设备无法启动 (错误代码 10/28/31/39/43) | Step 2 (驱动安装和签名状态) |
| 驱动未安装或版本过旧 | Step 3 (驱动版本和更新状态) |
| 驱动签名验证失败 | Step 2 (驱动安装和签名状态) |

## 诊断步骤

### Step 1: 检查设备状态和错误代码

**数据采集**：

> 采集目标：获取设备管理器中所有 PnP 设备的状态，识别异常设备（错误代码非 0）

```powershell
# Get all PnP device status and filter abnormal devices
$devices = Get-CimInstance Win32_PnPEntity | Where-Object { $_.ConfigManagerErrorCode -ne 0 }
if ($devices) {
    Write-Host "Found $($devices.Count) devices with errors:"
    $devices | Select-Object Name, DeviceID, ConfigManagerErrorCode, Status | Format-Table -AutoSize
} else {
    Write-Host "All PnP devices are working properly (no error codes)"
}
# View device count overview
$allDevices = Get-CimInstance Win32_PnPEntity
Write-Host "`nTotal PnP devices: $($allDevices.Count)"
```

**分析思路**：

1. 检查设备错误代码：
   - 正常：所有设备错误代码为 0
   - 异常：出现非 0 错误代码 → 根据错误代码判断问题类型：
     - 代码 1: 设备未正确配置 → **根因**：设备配置异常，**严重程度**：Warning
     - 代码 10: 设备无法启动 → **根因**：设备无法启动，可能驱动不兼容或硬件故障，**严重程度**：Critical
     - 代码 28: 驱动未安装 → **根因**：设备驱动未安装，**严重程度**：Critical
     - 代码 31: 设备工作不正常 → **根因**：设备工作异常，**严重程度**：Critical
     - 代码 39: 驱动损坏 → **根因**：驱动程序损坏，**严重程度**：Critical
     - 代码 43: 驱动报告设备失败 → **根因**：驱动报告设备失败，**严重程度**：Critical
     - 其他代码：参考微软文档判断具体问题

> 如果发现网络适配器驱动异常，参见 → [networking-tcpip.md](references/networking-tcpip.md)
> 如果发现存储控制器驱动异常，参见 → [storage-hardware.md](references/storage-hardware.md)
> 如果发现 VirtIO/Xen 驱动问题，参见 → [cloud-driver.md](references/cloud-driver.md)

### Step 2: 检查驱动安装和签名状态

**数据采集**：

> 采集目标：获取已安装的第三方驱动包列表，检查驱动签名状态

```powershell
# Get installed third-party driver packages
$drivers = Get-WindowsDriver -Online -ErrorAction SilentlyContinue | Where-Object { $_.Driver -like 'oem*' }
if ($drivers) {
    Write-Host "Installed third-party drivers: $($drivers.Count)"
    $drivers | Select-Object Driver, OriginalFileName, ProviderName, Date, Version | Format-Table -AutoSize
} else {
    Write-Host "No third-party drivers found or Get-WindowsDriver not available"
}
# Check driver signature verification policy
$codeIntegrity = (Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\CI' -Name 'UMCIAuditMode' -ErrorAction SilentlyContinue).UMCIAuditMode
Write-Host "UMCI Audit Mode: $codeIntegrity"
$testSigning = bcdedit /enum '{current}' 2>$null | Select-String 'testsigning'
if ($testSigning) {
    Write-Host "Test Signing: $testSigning"
} else {
    Write-Host "Test Signing: Not enabled (normal)"
}
```

**分析思路**：

1. 检查第三方驱动包：
   - 正常：所有驱动都有有效签名
   - 异常：发现未签名或签名无效的驱动 → **根因**：驱动签名异常，可能被系统拒绝加载，**严重程度**：Warning
2. 检查 Test Signing 模式：
   - 正常：Test Signing 未启用
   - 异常：Test Signing 已启用 → **根因**：测试签名模式已启用，存在安全风险，**严重程度**：Warning

### Step 3: 检查驱动版本和更新状态

**数据采集**：

> 采集目标：获取异常设备的驱动版本信息和驱动服务状态

```powershell
# Get all device driver service status
$driverServices = Get-CimInstance Win32_SystemDriver | Where-Object { $_.State -ne 'Running' -and $_.StartMode -ne 'Disabled' }
if ($driverServices) {
    Write-Host "Non-running driver services (not disabled):"
    $driverServices | Select-Object Name, DisplayName, State, StartMode, PathName | Format-Table -AutoSize
} else {
    Write-Host "All expected driver services are running"
}
```

**分析思路**：

1. 检查驱动服务状态：
   - 正常：所有非禁用的驱动服务都在运行
   - 异常：应运行但未运行的驱动服务 → **根因**：驱动服务未正常启动，对应设备将无法正常工作，**严重程度**：Critical

### Step 4: 检查驱动安装策略

**数据采集**：

> 采集目标：检查是否存在组策略或注册表设置禁止驱动安装

```powershell
# Check driver installation disable policy (for Windows Server 2012+)
$installDisabled = (Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\DeviceInstall\Parameters' -Name 'DeviceInstallDisabled' -ErrorAction SilentlyContinue).DeviceInstallDisabled
Write-Host "DeviceInstallDisabled: $installDisabled"
# Check older version policy (for Windows Server 2008 R2 and earlier)
$plugPlayDisabled = (Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\PlugPlay\Parameters' -Name 'DeviceInstallDisabled' -ErrorAction SilentlyContinue).DeviceInstallDisabled
Write-Host "DeviceInstallDisabled (PlugPlay): $plugPlayDisabled"
# Check device installation restriction group policy
$denyList = Get-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\DeviceInstall\Restrictions' -ErrorAction SilentlyContinue
if ($denyList) {
    Write-Host "Device Install Restrictions policy found:"
    $denyList | Select-Object -Property * -ExcludeProperty PSPath,PSParentPath,PSChildName,PSDrive,PSProvider | Format-List
} else {
    Write-Host "No device install restrictions policy"
}
```

**分析思路**：

1. 检查驱动安装是否被禁用：
   - 正常：未配置禁用策略
   - 异常：DeviceInstallDisabled 为 1 → **根因**：驱动安装被策略禁用，新设备无法自动安装驱动，**严重程度**：Critical
2. 检查设备安装限制：
   - 关注是否有特定设备类别或硬件 ID 被禁止安装

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Step 1 发现网络适配器驱动异常 | → [networking-tcpip.md](references/networking-tcpip.md) |
| 条件跳转 | Step 1 发现存储控制器驱动异常 | → [storage-hardware.md](references/storage-hardware.md) |
| 条件跳转 | Step 1 发现 VirtIO/Xen 驱动问题 | → [cloud-driver.md](references/cloud-driver.md) |
| 链式后继 | 本文件未确认根因 | → [system-management.md](references/system-management.md) |

## 修复建议

### 根因: 设备驱动未安装（错误代码 28）

**修复操作**：

```powershell
# Try to search and install driver via Windows Update
pnputil /scan-devices
# Or manually install specified driver package
pnputil /add-driver <DriverInfFile> /install
```

**验证方法**：

```powershell
Get-CimInstance Win32_PnPEntity | Where-Object { $_.ConfigManagerErrorCode -ne 0 } | Select-Object Name, ConfigManagerErrorCode | Format-Table -AutoSize
```

预期结果：无异常设备或目标设备错误代码已消失

**风险说明**：安装不兼容的驱动可能导致蓝屏或设备工作异常，建议优先使用设备制造商提供的驱动。

### 根因: 驱动安装被策略禁用

**修复操作**：

```powershell
# Remove driver installation disable policy (Windows Server 2012+)
Remove-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\DeviceInstall\Parameters' -Name 'DeviceInstallDisabled' -ErrorAction SilentlyContinue
# Or remove older version policy (Windows Server 2008 R2 and earlier)
Remove-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\PlugPlay\Parameters' -Name 'DeviceInstallDisabled' -ErrorAction SilentlyContinue
```

**验证方法**：

```powershell
$val = (Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\DeviceInstall\Parameters' -Name 'DeviceInstallDisabled' -ErrorAction SilentlyContinue).DeviceInstallDisabled
Write-Host "DeviceInstallDisabled: $val"
```

预期结果：返回空或属性不存在

**风险说明**：禁用策略可能是管理员出于安全考虑配置的，移除前建议确认原始意图。如果是通过组策略配置，需要通过 GPO 修改而非直接删除注册表。

### 根因: 驱动服务未正常启动

**修复操作**：

```powershell
# Start non-running driver service (replace <ServiceName> with actual service name)
Start-Service -Name '<ServiceName>' -ErrorAction Stop
```

**验证方法**：

```powershell
Get-Service -Name '<ServiceName>' | Select-Object Name, Status, StartType | Format-Table -AutoSize
```

预期结果：Status 为 Running

**风险说明**：启动驱动服务前确认驱动文件完整，损坏的驱动启动后可能导致蓝屏。
