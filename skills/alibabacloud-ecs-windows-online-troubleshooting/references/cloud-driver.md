# Cloud Driver 诊断

## 功能说明

诊断 Windows VirtIO 驱动版本、存在性、驱动安装策略和 Xen 驱动残留。覆盖 3 个已知根因。

**输入**：用户问题描述（必选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 虚拟磁盘或网卡性能不佳、间歇性断连 | Step 1 (VirtIO 驱动包) → Step 2 (VirtIO 驱动版本) |
| 迁移到 KVM 平台后磁盘或网络不可用 | Step 1 (VirtIO 驱动包) → Step 4 (Xen 驱动残留) |
| 安装新驱动后设备管理器仍显示旧驱动 | Step 3 (驱动安装策略) |
| 从 Xen 迁移到 KVM 后网卡或磁盘不工作 | Step 4 (Xen 驱动残留) |
| 对应虚拟设备无法正常工作 | Step 1 (VirtIO 驱动包) → Step 2 (VirtIO 驱动版本) |

## 诊断步骤

### Step 1: VirtIO 驱动包检查

**数据采集**：

> 采集目标：获取系统中所有已安装的第三方驱动包，筛选存储、网络、系统类别的驱动

```powershell
# List all third-party driver packages (SCSIAdapter / Net / System categories include VirtIO drivers)
Get-WindowsDriver -Online -ErrorAction SilentlyContinue |
  Where-Object { $_.ClassName -eq 'SCSIAdapter' -or $_.ClassName -eq 'Net' -or $_.ClassName -eq 'System' } |
  Select-Object Driver, OriginalFileName, ClassName, ProviderName, Version, Date | Format-Table -AutoSize
```

**分析思路**：

1. 检查 VirtIO 核心驱动是否存在：
   - 正常：输出中包含以下关键驱动包
     - `viostor.inf`：VirtIO 存储驱动
     - `netkvm.inf`：VirtIO 网络驱动
     - `balloon.inf`：VirtIO 内存气球驱动
     - `vioser.inf`：VirtIO 串口驱动
   - 异常：缺少 viostor.inf 或 netkvm.inf → 可能导致 KVM 平台上磁盘或网络不可用，**严重程度**：Critical

2. 识别 ProviderName 确认驱动来源：
   - 阿里云官方 VirtIO 驱动的 ProviderName 通常包含 "Red Hat" 或 "Alibaba"

### Step 2: VirtIO 驱动版本检查

**数据采集**：

> 采集目标：获取 VirtIO 驱动的详细版本信息

```powershell
# Get VirtIO driver detailed version (read from driver files directly)
Get-CimInstance -ClassName Win32_PnPSignedDriver -ErrorAction SilentlyContinue |
  Where-Object { $_.DeviceName -like '*VirtIO*' -or $_.DeviceName -like '*Red Hat*' } |
  Select-Object DeviceName, DriverVersion, IsSigned, DriverDate | Format-Table -AutoSize
```

**分析思路**：

1. 检查 VirtIO 驱动版本是否过旧：
   - 正常：驱动版本号的第 4 段 >= 58017（如 `100.86.104.58200`，第 4 段为 58200 >= 58017）
   - 异常：驱动版本号第 4 段 < 58017 → **根因**：VirtIO 驱动版本过旧，**严重程度**：Warning
   - 说明：版本阈值 58017 是阿里云官方推荐的最低版本，旧版本可能存在性能问题或已知 Bug

### Step 3: 驱动安装策略检查

**数据采集**：

> 采集目标：检查是否通过组策略或注册表禁止了驱动自动安装

```powershell
# Check driver installation policy (Windows Server 2016+)
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Services\DeviceInstall\Parameters' -Name DeviceInstallDisabled -ErrorAction SilentlyContinue |
  Select-Object DeviceInstallDisabled
```

```powershell
# Check driver installation policy (Windows Server 2012 and earlier)
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Services\PlugPlay\Parameters' -Name DeviceInstallDisabled -ErrorAction SilentlyContinue |
  Select-Object DeviceInstallDisabled
```

**分析思路**：

1. 检查驱动安装是否被禁止：
   - 正常：注册表项不存在，或值为 0
   - 异常：`DeviceInstallDisabled` 值为非 0 → **根因**：驱动安装被策略禁止，**严重程度**：Warning
   - 说明：不同 Windows 版本使用不同的注册表路径（`DeviceInstall\Parameters` 或 `PlugPlay\Parameters`），值名均为 `DeviceInstallDisabled`

### Step 4: Xen 驱动残留检查

**数据采集**：

> 采集目标：检查是否存在从 Xen 平台迁移后的残留驱动包和服务

```powershell
# Check if Xen driver packages exist
Get-WindowsDriver -Online -ErrorAction SilentlyContinue |
  Where-Object { $_.OriginalFileName -match 'xen' } |
  Select-Object Driver, OriginalFileName, Version | Format-Table -AutoSize
```

```powershell
# Check XenPCI service hide_devices parameter
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Services\XenPCI\Parameters' -Name hide_devices -ErrorAction SilentlyContinue |
  Select-Object hide_devices
```

**分析思路**：

1. 检查 Xen 驱动包残留：
   - 正常：无任何 Xen 相关驱动包（xennet.inf、xenpci.inf、xenscsi.inf、xenstub.inf、xenvbd.inf）
   - 异常：存在 Xen 驱动包且 XenPCI 服务的 `hide_devices` 参数非空 → **根因**：Xen 驱动残留，**严重程度**：Warning
   - 说明：Xen 驱动的 `hide_devices` 参数会隐藏 VirtIO 设备，导致 KVM 平台上磁盘、网卡等虚拟设备不可见

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | 驱动策略被 GPO 禁止 | → [system-gpo.md](references/system-gpo.md)（排查组策略配置） |
| 条件跳转 | 磁盘不可见但驱动正常 | → [storage-hardware.md](references/storage-hardware.md)（排查存储硬件问题） |
| 条件跳转 | 网络不通但驱动正常 | → [networking-tcpip.md](references/networking-tcpip.md)（排查网络配置） |

## 修复建议

### 根因：VirtIO 驱动版本过旧

**修复操作**：

```powershell
# Download the latest VirtIO driver (via Alibaba Cloud official channel)
# Installation steps:
# 1. Download the latest VirtIO driver installer
# 2. Run the installer as Administrator
# 3. Restart the instance after installation completes
```

**验证方法**：

```powershell
Get-CimInstance -ClassName Win32_PnPSignedDriver -ErrorAction SilentlyContinue |
  Where-Object { $_.DeviceName -like '*VirtIO*' -or $_.DeviceName -like '*Red Hat*' } |
  Select-Object DeviceName, DriverVersion | Format-Table -AutoSize
```

预期结果：驱动版本号第 4 段 >= 58017

**风险说明**：更新驱动需要重启，建议在维护窗口执行；更新前创建系统快照以便回滚

---

### 根因：驱动安装被策略禁止

**修复操作**：

```powershell
# Windows Server 2016+ remove block policy
Remove-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Services\DeviceInstall\Parameters' -Name DeviceInstallDisabled -ErrorAction SilentlyContinue

# Windows Server 2012 and earlier
Remove-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Services\PlugPlay\Parameters' -Name DeviceInstallDisabled -ErrorAction SilentlyContinue
```

**验证方法**：

```powershell
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Services\DeviceInstall\Parameters' -Name DeviceInstallDisabled -ErrorAction SilentlyContinue | Select-Object DeviceInstallDisabled
```

预期结果：注册表项不存在（无输出）

**风险说明**：如该策略由域 GPO 设置，手动修改可能被 GPO 覆盖，需在 GPO 中修改

---

### 根因：Xen 驱动残留

**修复操作**：

```powershell
# Remove XenPCI service hide_devices parameter
Remove-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Services\XenPCI\Parameters' -Name hide_devices -ErrorAction SilentlyContinue

# Uninstall Xen driver packages (one by one)
$xenDrivers = Get-WindowsDriver -Online | Where-Object { $_.OriginalFileName -match 'xen' }
foreach ($d in $xenDrivers) {
  pnputil /delete-driver $d.Driver /uninstall /force
}
```

**验证方法**：

```powershell
Get-WindowsDriver -Online -ErrorAction SilentlyContinue |
  Where-Object { $_.OriginalFileName -match 'xen' } |
  Select-Object Driver, OriginalFileName | Format-Table -AutoSize
```

预期结果：无 Xen 相关驱动包输出

**风险说明**：卸载 Xen 驱动后需重启生效；确保已安装 VirtIO 驱动后再执行，否则可能导致磁盘或网络不可用
