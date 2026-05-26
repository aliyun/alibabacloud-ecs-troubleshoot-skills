# Storage Hardware and Drivers 诊断

## 目录

- [Storage Hardware and Drivers 诊断](#storage-hardware-and-drivers-诊断)
  - [目录](#目录)
  - [功能说明](#功能说明)
  - [步骤选取指引](#步骤选取指引)
  - [诊断步骤](#诊断步骤)
    - [Step 1: SCSI 控制器状态检查](#step-1-scsi-控制器状态检查)
    - [Step 2: 磁盘驱动状态检查](#step-2-磁盘驱动状态检查)
    - [Step 3: 注册表过滤驱动检查](#step-3-注册表过滤驱动检查)
    - [Step 4: 磁盘类驱动检查](#step-4-磁盘类驱动检查)
    - [Step 5: 存储事件日志检查](#step-5-存储事件日志检查)
  - [交叉引用](#交叉引用)
  - [修复建议](#修复建议)
    - [根因：SCSI 控制器状态异常](#根因scsi-控制器状态异常)
    - [根因：SCSI 控制器无关联磁盘](#根因scsi-控制器无关联磁盘)
    - [根因：磁盘驱动状态异常](#根因磁盘驱动状态异常)
    - [根因：注册表中残留过滤驱动](#根因注册表中残留过滤驱动)
    - [根因：磁盘类驱动未找到](#根因磁盘类驱动未找到)
    - [根因：磁盘类过滤驱动异常](#根因磁盘类过滤驱动异常)
    - [根因：非标准过滤驱动存在](#根因非标准过滤驱动存在)
    - [根因：标准过滤驱动缺失](#根因标准过滤驱动缺失)
    - [根因：VirtIO 存储驱动内部错误（Event ID 11）](#根因virtio-存储驱动内部错误event-id-11)
    - [根因：磁盘设备移除失败（Event 225）](#根因磁盘设备移除失败event-225)

## 功能说明

诊断 Windows 存储硬件和驱动栈问题：SCSI 控制器状态、磁盘驱动状态、注册表过滤驱动残留、磁盘类驱动配置、存储相关事件日志（VirtIO 错误、设备移除失败）。覆盖 10 个已知问题项。

**输入**：用户问题描述（必选）、错误代码/事件ID/截图（可选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 磁盘控制器异常、黄色感叹号 | Step 1 (SCSI 控制器) → Step 2 (磁盘驱动) |
| 控制台已挂载磁盘但系统内看不到 | Step 1 (SCSI 控制器) → Step 2 (磁盘驱动) → Step 4 (类驱动) |
| 磁盘驱动错误、蓝屏 | Step 2 (磁盘驱动) → Step 3 (注册表过滤驱动) → Step 4 (类驱动) |
| 卸载安全软件后磁盘无法访问 | Step 3 (注册表过滤驱动) → Step 4 (类驱动) |
| 磁盘 I/O 间歇性失败、Event ID 11 | Step 5 (存储事件日志) |
| 热卸载云盘后资源泄漏、Event 225 | Step 5 (存储事件日志) |

## 诊断步骤

### Step 1: SCSI 控制器状态检查

**数据采集**：

> 采集目标：获取所有 SCSI 控制器的状态、驱动名称、设备管理器错误码，以及关联的磁盘设备

```powershell
# SCSI controllers and their associated disk devices (via Win32_SCSIControllerDevice association)
Get-CimInstance -ClassName Win32_SCSIController | ForEach-Object {
    $ctrl = $_
    $device = Get-CimAssociatedInstance -InputObject $ctrl -Association Win32_SCSIControllerDevice -ResultClassName Win32_PnPEntity -ErrorAction SilentlyContinue
    [PSCustomObject]@{
        DeviceID     = $ctrl.DeviceID
        Status       = $ctrl.Status
        DriverName   = $ctrl.DriverName
        ErrorCode    = $ctrl.ConfigManagerErrorCode
        DeviceCount  = @($device).Count
        DeviceInfo   = if ($device) { ($device | ForEach-Object { "$($_.DeviceID) [$($_.Status)]" }) -join '; ' } else { 'None' }
    }
}
```

**分析思路**：

1. 检查 SCSI 控制器状态：
   - 正常：状态为 OK，无错误码
   - 异常：状态非 OK 或存在错误码 → **根因**：SCSI 控制器状态异常，**严重程度**：Critical
   - 注意：排除 Microsoft Storage Spaces 控制器（ROOT\SPACEPORT），它没有关联磁盘是正常的

2. 检查控制器是否关联磁盘：
   - 正常：每个非 SPACEPORT 控制器至少关联一个磁盘
   - 异常：控制器正常但无关联磁盘 → **根因**：SCSI 控制器无关联磁盘，**严重程度**：Critical

如果发现 SCSI 控制器异常且为 VirtIO 驱动，参见 → [cloud-driver.md](references/cloud-driver.md)

### Step 2: 磁盘驱动状态检查

**数据采集**：

> 采集目标：获取所有磁盘驱动器设备的状态、驱动服务名称和错误码

```powershell
Get-PnpDevice -Class DiskDrive | Select-Object InstanceId, FriendlyName, Status, ConfigManagerErrorCode, Service
```

**分析思路**：

1. 检查磁盘驱动状态：
   - 正常：状态为 OK，无错误码
   - 异常：状态非 OK 或存在非零错误码 → **根因**：磁盘驱动状态异常，**严重程度**：Critical
   - 常见错误码：10（无法启动）、28（驱动未安装）、31（设备工作不正常）、43（设备已停止）

如果发现驱动状态异常，参见 → [device-driver.md](references/device-driver.md)（设备管理器错误码排查）

### Step 3: 注册表过滤驱动检查

**数据采集**：

> 采集目标：获取 SCSI 总线下磁盘设备实例的注册表过滤驱动配置，检查是否有残留的第三方过滤驱动

```powershell
# Get disk device instances via SCSI controller association, check registry filter drivers and corresponding service status
Get-CimInstance -ClassName Win32_SCSIController | ForEach-Object {
    $devices = Get-CimAssociatedInstance -InputObject $_ -Association Win32_SCSIControllerDevice -ResultClassName Win32_PnPEntity -ErrorAction SilentlyContinue
    foreach ($dev in $devices) {
        $path = "HKLM:\SYSTEM\CurrentControlSet\Enum\$($dev.DeviceID)"
        $upper = (Get-ItemProperty -Path $path -Name UpperFilters -ErrorAction SilentlyContinue).UpperFilters
        $lower = (Get-ItemProperty -Path $path -Name LowerFilters -ErrorAction SilentlyContinue).LowerFilters
        if ($upper -or $lower) {
            $allFilters = @($upper) + @($lower) | Where-Object { $_ }
            $svcStatus = $allFilters | ForEach-Object {
                $svc = Get-Service -Name $_ -ErrorAction SilentlyContinue
                "$_=$(if ($svc) { $svc.Status } else { 'NotFound' })"
            }
            [PSCustomObject]@{
                DeviceID       = $dev.DeviceID
                UpperFilters   = $upper -join ', '
                LowerFilters   = $lower -join ', '
                ServiceStatus  = $svcStatus -join '; '
            }
        }
    }
}
```

**分析思路**：

1. 检查是否存在过滤驱动：
   - 正常：无过滤驱动，或过滤驱动对应的服务正常运行
   - 异常：过滤驱动存在但对应服务不存在或已停止 → **根因**：注册表中残留过滤驱动，**严重程度**：Critical
   - 常见场景：卸载杀毒软件（如赛门铁克、McAfee、卡巴斯基）后残留存储过滤驱动

### Step 4: 磁盘类驱动检查

**数据采集**：

> 采集目标：获取磁盘相关设备类（SCSIAdapter、DiskDrive、Volume）的类级过滤驱动配置

```powershell
# SCSIAdapter class filter drivers
$scsiPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4d36e97b-e325-11ce-bfc1-08002be10318}"
[PSCustomObject]@{
    Class = 'SCSIAdapter'
    UpperFilters = (Get-ItemProperty -Path $scsiPath -Name UpperFilters -ErrorAction SilentlyContinue).UpperFilters -join ', '
    LowerFilters = (Get-ItemProperty -Path $scsiPath -Name LowerFilters -ErrorAction SilentlyContinue).LowerFilters -join ', '
}

# DiskDrive class filter drivers
$diskPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4d36e967-e325-11ce-bfc1-08002be10318}"
[PSCustomObject]@{
    Class = 'DiskDrive'
    UpperFilters = (Get-ItemProperty -Path $diskPath -Name UpperFilters -ErrorAction SilentlyContinue).UpperFilters -join ', '
    LowerFilters = (Get-ItemProperty -Path $diskPath -Name LowerFilters -ErrorAction SilentlyContinue).LowerFilters -join ', '
}

# Volume class filter drivers
$volPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Class\{71a27cdd-812a-11d0-bec7-08002be2092f}"
[PSCustomObject]@{
    Class = 'Volume'
    UpperFilters = (Get-ItemProperty -Path $volPath -Name UpperFilters -ErrorAction SilentlyContinue).UpperFilters -join ', '
    LowerFilters = (Get-ItemProperty -Path $volPath -Name LowerFilters -ErrorAction SilentlyContinue).LowerFilters -join ', '
}
```

**分析思路**：

1. 检查 SCSIAdapter 类过滤驱动：
   - 正常：无过滤驱动（标准配置）
   - 异常：存在过滤驱动且对应服务异常 → **根因**：磁盘类过滤驱动异常，**严重程度**：Critical

2. 检查 DiskDrive 类过滤驱动：
   - 正常标准配置：UpperFilters 包含 `partmgr`；Win8 及以上 LowerFilters 包含 `EhStorClass`
   - 异常（类驱动未找到）：DiskDrive 类注册表项不存在 → **根因**：磁盘类驱动未找到，**严重程度**：Critical
   - 异常（过滤驱动异常）：存在标准之外的过滤驱动且对应服务异常 → **根因**：磁盘类过滤驱动异常，**严重程度**：Critical
   - 异常（非标准过滤驱动）：存在标准之外的过滤驱动但服务正常 → **根因**：非标准过滤驱动存在，**严重程度**：Warning
   - 异常（标准过滤驱动缺失）：`partmgr` 或 `EhStorClass` 缺失 → **根因**：标准过滤驱动缺失，**严重程度**：Critical

3. 检查 Volume 类过滤驱动：
   - 正常标准配置：Win10 及以上 UpperFilters 包含 `volsnap`
   - 异常（类驱动未找到）：Volume 类注册表项不存在 → **根因**：磁盘类驱动未找到，**严重程度**：Critical
   - 异常（过滤驱动异常）：存在标准之外的过滤驱动且对应服务异常 → **根因**：磁盘类过滤驱动异常，**严重程度**：Critical
   - 异常（非标准过滤驱动）：存在标准之外的过滤驱动但服务正常 → **根因**：非标准过滤驱动存在，**严重程度**：Warning
   - 异常（标准过滤驱动缺失）：`volsnap` 缺失 → **根因**：标准过滤驱动缺失，**严重程度**：Critical

### Step 5: 存储事件日志检查

**数据采集**：

> 采集目标：获取最近 7 天内的 VirtIO 存储驱动错误事件（Event ID 11）和 PnP 设备移除失败事件（Event ID 225）

```powershell
# VirtIO storage driver internal errors (Event ID 11, source disk/viostor)
Get-WinEvent -FilterHashtable @{LogName='System'; Id=11; StartTime=(Get-Date).AddDays(-7)} -MaxEvents 20 -ErrorAction SilentlyContinue | Where-Object { $_.ProviderName -eq 'disk' } | Select-Object TimeCreated, Id, LevelDisplayName, Message | Format-Table -AutoSize

# PnP device removal failure (Event ID 225, source Kernel-PnP)
Get-WinEvent -FilterHashtable @{LogName='System'; Id=225; StartTime=(Get-Date).AddDays(-7)} -MaxEvents 10 -ErrorAction SilentlyContinue | Where-Object { $_.ProviderName -eq 'Microsoft-Windows-Kernel-PnP' } | Select-Object TimeCreated, Id, LevelDisplayName, Message | Format-Table -AutoSize
```

**分析思路**：

1. 检查 VirtIO 存储驱动错误（Event ID 11）：
   - 正常：无 Event ID 11 事件
   - 异常：存在 Event ID 11 事件 → **根因**：VirtIO 存储驱动内部错误（Event ID 11），**严重程度**：Warning
   - 说明：通常是 VirtIO 存储驱动 (viostor/vioscsi) 报告的 I/O 错误，可能是驱动版本过旧或底层存储瞬时异常

2. 检查 PnP 设备移除失败（Event ID 225）：
   - 正常：无 Event ID 225 事件，或事件不涉及 VirtIO 设备
   - 异常：存在针对 VirtIO 设备（PCI\VEN_1AF4&DEV_1001 或 PCI\VEN_8086&DEV_5845）的 Event 225 → **根因**：磁盘设备移除失败（Event 225），**严重程度**：Warning
   - 说明：通常发生在热卸载云盘时，进程仍持有磁盘句柄导致移除失败

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Step 1/2 发现 VirtIO 驱动相关异常 | → [cloud-driver.md](references/cloud-driver.md) |
| 条件跳转 | Step 2 发现设备管理器错误码 | → [device-driver.md](references/device-driver.md) |
| 链式后继 | 本文件未确认根因，用户报告磁盘不可见 | → [device-driver.md](references/device-driver.md) |

## 修复建议

### 根因：SCSI 控制器状态异常

**修复操作**：

```powershell
# Rescan devices
pnputil /scan-devices
```

**验证方法**：

```powershell
Get-PnpDevice -Class SCSIAdapter | Select-Object FriendlyName, Status
```

预期结果：所有 SCSI 控制器状态为 OK

**风险说明**：如果控制器硬件或驱动有问题，重新扫描可能无法解决，需要更新驱动或联系云平台支持

---

### 根因：SCSI 控制器无关联磁盘

**修复操作**：

```powershell
# Rescan disks
"rescan" | diskpart

# Rescan devices
pnputil /scan-devices
```

**验证方法**：

```powershell
Get-Disk | Select-Object Number, FriendlyName, OperationalStatus
```

预期结果：控制器关联的磁盘出现在磁盘列表中

**风险说明**：如果云平台侧磁盘挂载正常但系统内仍看不到，可能是驱动问题

---

### 根因：磁盘驱动状态异常

**修复操作**：

```powershell
# Re-enable abnormal disk devices (operate one by one to avoid accidentally affecting system disk)
# Replace <InstanceId> with the abnormal device instance ID confirmed during diagnosis
$instanceId = '<InstanceId>'
Disable-PnpDevice -InstanceId $instanceId -Confirm:$false
Enable-PnpDevice -InstanceId $instanceId -Confirm:$false
```

**验证方法**：

```powershell
Get-PnpDevice -Class DiskDrive | Select-Object FriendlyName, Status
```

预期结果：所有磁盘设备状态为 OK

**风险说明**：禁用再启用磁盘设备会导致短暂的磁盘不可用，不要对系统盘执行此操作

---

### 根因：注册表中残留过滤驱动

**修复操作**：

```powershell
# View residual filter drivers (first confirm which device instances have issues)
# Delete device instance-level residual filter drivers (replace <InstanceId> with actual value)
$path = "HKLM:\SYSTEM\CurrentControlSet\Enum\<InstanceId>"

# View current values
Get-ItemProperty -Path $path -Name UpperFilters, LowerFilters -ErrorAction SilentlyContinue | Select-Object UpperFilters, LowerFilters

# Delete residual filter drivers (caution: confirm they are residual items)
# Remove-ItemProperty -Path $path -Name UpperFilters -ErrorAction SilentlyContinue
# Remove-ItemProperty -Path $path -Name LowerFilters -ErrorAction SilentlyContinue
```

**验证方法**：

重启后检查磁盘是否正常：

```powershell
Get-Disk | Select-Object Number, OperationalStatus
```

预期结果：磁盘状态正常

**风险说明**：修改注册表过滤驱动后需要重启。如果删除了正在使用的过滤驱动，可能导致磁盘无法访问或蓝屏。操作前建议创建系统还原点

---

### 根因：磁盘类驱动未找到

**修复操作**：

```powershell
# Reinstall disk class driver (VirtIO/SCSI controller driver)
# Scan for hardware changes to trigger driver re-detection
pnputil /scan-devices

# If scan does not resolve, force reinstall the storage controller driver
# First, identify the problematic storage controller
Get-PnpDevice -Class SCSIAdapter -ErrorAction SilentlyContinue | Where-Object { $_.Status -ne 'OK' } | Select-Object InstanceId, FriendlyName, Status | Format-Table -AutoSize

# Disable and re-enable the storage controller to trigger driver reload
$controller = Get-PnpDevice -Class SCSIAdapter -ErrorAction SilentlyContinue | Where-Object { $_.Status -ne 'OK' } | Select-Object -First 1
if ($controller) {
    Disable-PnpDevice -InstanceId $controller.InstanceId -Confirm:$false
    Start-Sleep -Seconds 2
    Enable-PnpDevice -InstanceId $controller.InstanceId -Confirm:$false
}
```

**验证方法**：

```powershell
Get-Disk | Select-Object Number, OperationalStatus, HealthStatus
Get-PnpDevice -Class SCSIAdapter -ErrorAction SilentlyContinue | Select-Object FriendlyName, Status | Format-Table -AutoSize
```

预期结果：磁盘状态正常，SCSI 控制器状态为 OK

**风险说明**：重装驱动可能导致磁盘短暂离线。对于云服务器，如果 VirtIO 驱动异常，可能需要通过云平台提供的驱动修复工具或重装实例解决。

---

### 根因：磁盘类过滤驱动异常

**修复操作**：

```powershell
# View current class filter drivers
$diskPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4d36e967-e325-11ce-bfc1-08002be10318}"
Get-ItemProperty -Path $diskPath -Name UpperFilters, LowerFilters -ErrorAction SilentlyContinue | Select-Object UpperFilters, LowerFilters

# Remove abnormal filter drivers (only remove confirmed residual items, keep standard filter drivers partmgr, EhStorClass)
# Example: suppose need to remove "badfilter"
# $current = (Get-ItemProperty -Path $diskPath -Name UpperFilters).UpperFilters
# $new = $current | Where-Object { $_ -ne 'badfilter' }
# Set-ItemProperty -Path $diskPath -Name UpperFilters -Value $new
```

**验证方法**：

重启后检查磁盘状态：

```powershell
Get-Disk | Select-Object Number, OperationalStatus | Format-Table -AutoSize
Get-PnpDevice -Class DiskDrive | Select-Object FriendlyName, Status | Format-Table -AutoSize
```

预期结果：磁盘和设备状态正常

**风险说明**：修改类级过滤驱动影响所有同类设备，操作前务必确认要删除的是残留项。修改后需要重启

---

### 根因：非标准过滤驱动存在

**说明**：

检测到非 Windows 标准的过滤驱动，通常由第三方存储管理软件（如杀毒软件、加密软件、磁盘缓存软件）安装。如果服务状态正常，一般不影响磁盘功能，但可能增加 I/O 延迟或引入兼容性问题。

建议：
- 确认非标准过滤驱动的来源和用途
- 如果是已卸载软件的残留，按照"磁盘类过滤驱动异常"的方法清理
- 如果是正在使用的软件，评估是否需要保留

---

### 根因：标准过滤驱动缺失

**修复操作**：

```powershell
# Restore standard filter drivers (using DiskDrive class as example)
$diskPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4d36e967-e325-11ce-bfc1-08002be10318}"

# Set correct standard filter drivers according to OS version
# Windows 8/Server 2012 and above: UpperFilters = partmgr, LowerFilters = EhStorClass
# Windows 7/Server 2008 R2: UpperFilters = partmgr, LowerFilters empty
Set-ItemProperty -Path $diskPath -Name UpperFilters -Value @('partmgr')
# Set-ItemProperty -Path $diskPath -Name LowerFilters -Value @('EhStorClass')  # Win8+
```

**验证方法**：

重启后检查磁盘状态：

```powershell
Get-Disk | Select-Object Number, OperationalStatus
```

预期结果：磁盘状态正常

**风险说明**：恢复标准过滤驱动通常安全，但需要重启才能生效

---

### 根因：VirtIO 存储驱动内部错误（Event ID 11）

**修复操作**：

```powershell
# Check current VirtIO storage driver version
Get-PnpDevice -Class SCSIAdapter | Where-Object { $_.FriendlyName -like '*VirtIO*' } | Select-Object FriendlyName, Service | Format-Table -AutoSize
driverquery /v | findstr /i "viostor vioscsi"
```

**验证方法**：

更新驱动后观察 7 天内是否还有 Event ID 11：

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; Id=11; StartTime=(Get-Date).AddDays(-7)} -MaxEvents 5 -ErrorAction SilentlyContinue | Where-Object { $_.ProviderName -eq 'disk' }
```

预期结果：无 Event ID 11 事件

**风险说明**：更新 VirtIO 驱动需要重启实例

---

### 根因：磁盘设备移除失败（Event 225）

**说明**：

Event ID 225 表示 PnP 管理器无法安全移除设备，通常发生在热卸载云盘时。常见原因：
- 进程仍在访问磁盘上的文件或目录
- 磁盘上的卷仍被挂载
- 第三方软件（如杀毒软件、备份软件）持有磁盘句柄

建议在卸载云盘前：
1. 关闭所有访问该磁盘的程序
2. 在磁盘管理器中先将磁盘设置为离线
3. 再从云平台控制台卸载
