# Performance Slow 诊断

## 目录

- [Performance Slow 诊断](#performance-slow-诊断)
  - [目录](#目录)
  - [功能说明](#功能说明)
  - [步骤选取指引](#步骤选取指引)
  - [诊断步骤](#诊断步骤)
    - [Step 1: CPU 使用率](#step-1-cpu-使用率)
    - [Step 2: 内存使用率与句柄数](#step-2-内存使用率与句柄数)
    - [Step 3: 页面文件配置](#step-3-页面文件配置)
    - [Step 4: 硬件保留内存](#step-4-硬件保留内存)
    - [Step 5: 超线程状态](#step-5-超线程状态)
    - [Step 6: BCD 启动配置限制](#step-6-bcd-启动配置限制)
    - [Step 7: 电源计划](#step-7-电源计划)
    - [Step 8: 文件系统过滤驱动检查](#step-8-文件系统过滤驱动检查)
  - [交叉引用](#交叉引用)
  - [修复建议](#修复建议)
    - [根因：页面文件未配置](#根因页面文件未配置)
    - [根因：BCD 启动配置限制处理器数量](#根因bcd-启动配置限制处理器数量)
    - [根因：BCD 启动配置截断内存](#根因bcd-启动配置截断内存)
    - [根因：超线程未启用](#根因超线程未启用)
    - [根因：电源计划为节能模式](#根因电源计划为节能模式)
    - [根因：第三方文件系统过滤驱动导致系统缓慢](#根因第三方文件系统过滤驱动导致系统缓慢)

## 功能说明

诊断 Windows 系统性能问题：CPU 使用率、内存使用率、系统句柄数、页面文件配置、硬件保留内存、超线程状态、BCD 启动配置中的 CPU/内存限制、电源计划、第三方文件系统过滤驱动。覆盖 8 个诊断步骤。

**输入**：用户问题描述（必选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 系统卡顿、响应缓慢 | Step 1 (CPU 使用率) → Step 2 (内存使用率与句柄数) |
| 新程序无法启动、内存不足 | Step 2 (内存使用率与句柄数) → Step 3 (页面文件配置) |
| 应用无法打开新窗口或文件 | Step 2 (内存使用率与句柄数) |
| 可用内存明显少于总物理内存 | Step 4 (硬件保留内存) |
| CPU 核心数/内存少于实例规格 | Step 5 (超线程状态) → Step 6 (BCD 启动限制) |
| 打开文件或程序缓慢、双击后延迟明显 | Step 8 (文件系统过滤驱动) |
| 全面性能异常、慢但无明确方向 | Step 1 → Step 2 → Step 3 → Step 7 (电源计划) → Step 8 (文件系统过滤驱动) |

## 诊断步骤

### Step 1: CPU 使用率

**数据采集**：

> 采集目标：CPU 总体使用率、各核心使用率、CPU 占用 Top 5 进程

```powershell
# CPU total and per-core usage (sample 2 times, 1-second interval)
Get-Counter '\Processor(*)\% Processor Time' -SampleInterval 1 -MaxSamples 2 -ErrorAction SilentlyContinue | Select-Object -Last 1 | ForEach-Object {
    $_.CounterSamples | Select-Object InstanceName, @{Name='CpuPercent';Expression={[math]::Round($_.CookedValue, 1)}} | Sort-Object CpuPercent -Descending
} | Format-Table -AutoSize

# Top 5 processes by CPU usage
Get-Counter '\Process(*)\% Processor Time' -SampleInterval 1 -MaxSamples 1 -ErrorAction SilentlyContinue | ForEach-Object {
    $_.CounterSamples | Where-Object { $_.InstanceName -notin '_Total','Idle' } | Select-Object InstanceName, @{Name='CpuPercent';Expression={[math]::Round($_.CookedValue / [Environment]::ProcessorCount, 1)}} | Sort-Object CpuPercent -Descending | Select-Object -First 5
} | Format-Table -AutoSize
```

> **Performance note**: CPU 采样窗口为 1 秒（1 次采样），优先保证诊断速度。如需更准确的长时间 CPU 趋势，可将 `-SampleInterval` 调整为 5 秒、`-MaxSamples` 调整为 3 以上。

**分析思路**：

1. 检查 CPU 总体使用率（_Total 实例）：
   - 低于 85% → 正常
   - 高于 85% → **根因**：CPU 总体使用率过高，结合 Top 5 进程分析占用来源，**严重程度**：Warning

2. 检查各核心 CPU 使用率：
   - 所有核心均低于 85% → 正常
   - 存在单核心持续高于 85% → **根因**：部分 CPU 核心负载过高，可能存在单线程应用或中断亲和性问题，**严重程度**：Warning

3. 检查 Top 5 CPU 占用进程：
   - 如果有第三方杀毒软件进程占用较高 → 参见 [desktop-app.md](references/desktop-app.md)（杀毒软件占用资源）

### Step 2: 内存使用率与句柄数

**数据采集**：

> 采集目标：物理内存总量、可用内存、使用率、系统总句柄数、内存/句柄占用 Top 5 进程

```powershell
# System memory overview
$os = Get-CimInstance Win32_OperatingSystem
[PSCustomObject]@{
    TotalPhysicalMemoryMB = [math]::Round($os.TotalVisibleMemorySize / 1024)
    FreePhysicalMemoryMB  = [math]::Round($os.FreePhysicalMemory / 1024)
    MemoryUsagePercent    = [math]::Round(($os.TotalVisibleMemorySize - $os.FreePhysicalMemory) / $os.TotalVisibleMemorySize * 100, 1)
} | Format-List

# Total system handle count
$handleCount = (Get-CimInstance Win32_Process | Measure-Object HandleCount -Sum).Sum
[PSCustomObject]@{ TotalHandleCount = $handleCount } | Format-List

# Top 5 processes by memory usage
Get-CimInstance Win32_Process | Sort-Object WorkingSetSize -Descending | Select-Object -First 5 ProcessId, Name, @{Name='MemoryMB';Expression={[math]::Round($_.WorkingSetSize / 1MB, 1)}}, HandleCount | Format-Table -AutoSize

# Top 5 processes by handle count
Get-CimInstance Win32_Process | Sort-Object HandleCount -Descending | Select-Object -First 5 ProcessId, Name, HandleCount, @{Name='MemoryMB';Expression={[math]::Round($_.WorkingSetSize / 1MB, 1)}} | Format-Table -AutoSize
```

**分析思路**：

1. 检查内存使用率：
   - 使用率低于 80% 且可用内存大于 512MB → 正常
   - 使用率高于 80% 或可用内存低于 512MB → **根因**：内存使用率过高，结合 Top 5 进程分析内存占用来源，**严重程度**：Warning

2. 检查系统总句柄数：
   - 低于 100000 → 正常
   - 超过 100000 → **根因**：系统句柄数过高，可能存在句柄泄漏的应用程序，影响系统资源分配，**严重程度**：Warning

### Step 3: 页面文件配置

**数据采集**：

> 采集目标：页面文件（虚拟内存）配置状态，包括内存管理注册表配置和当前页面文件使用情况

```powershell
# Memory management registry configuration (PagingFiles and ExistingPageFiles)
$mm = Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management" -ErrorAction SilentlyContinue
[PSCustomObject]@{
    PagingFiles       = $mm.PagingFiles
    ExistingPageFiles = $mm.ExistingPageFiles
} | Format-List

# Check if automatic page file management is enabled
(Get-CimInstance Win32_ComputerSystem).AutomaticManagedPagefile

# Current page file usage
Get-CimInstance Win32_PageFileUsage | Select-Object Name, AllocatedBaseSize, CurrentUsage, PeakUsage | Format-Table -AutoSize

# Page file settings (automatic / fixed size)
Get-CimInstance Win32_PageFileSetting | Select-Object Name, InitialSize, MaximumSize | Format-Table -AutoSize

# Crash dump configuration (page file must meet minimum requirements for crash dump)
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\CrashControl" -ErrorAction SilentlyContinue | Select-Object CrashDumpEnabled, DumpFile, MinidumpDir | Format-List
```

**分析思路**：

1. 检查自动管理页面文件状态：
   - AutomaticManagedPagefile 为 True → 系统自动管理页面文件大小（微软推荐配置），正常。注意：自动管理模式下，注册表 PagingFiles 字段可能不包含固定配置值，Win32_PageFileSetting 也可能返回空，不能据此判定「未配置」；应通过 Win32_PageFileUsage 查看实际的页面文件分配和使用情况
   - AutomaticManagedPagefile 为 False → 手动配置模式，进入步骤 2 检查配置状态

2. 检查页面文件是否实际存在：
   - Win32_PageFileUsage 返回非空结果 → 页面文件已配置且正在使用，不论自动管理还是手动管理
   - Win32_PageFileUsage 返回空结果 → 进入步骤 3

3. 检查手动配置的页面文件注册表项（仅在 AutomaticManagedPagefile = False 时适用）：
   - PagingFiles 和 ExistingPageFiles 均为空 → **根因**：手动管理模式下未配置页面文件，物理内存耗尽时应用程序将直接崩溃，**严重程度**：Warning
   - PagingFiles 包含有效路径 → 页面文件已手动配置，需结合 Win32_PageFileSetting 检查大小是否合理

4. 检查手动配置模式下页面文件设置是否合理：
   - Win32_PageFileSetting 返回空但 Win32_PageFileUsage 有数据 → 页面文件由系统自动管理或其他方式创建
   - InitialSize 和 MaximumSize 均有效 → 手动配置了页面文件大小范围
   - InitialSize = MaximumSize → 固定大小页面文件，注意大小是否足够
   - InitialSize < MaximumSize → 动态扩展，系统在范围内自动调整

5. 检查页面文件当前使用情况：
   - CurrentUsage 接近 AllocatedBaseSize → 页面文件即将耗尽，结合内存使用率判断是否需要扩大
   - PeakUsage 远高于 AllocatedBaseSize → 页面文件曾经不足

6. 检查崩溃转储依赖：
   - CrashDumpEnabled = 1（完全内存转储）→ 页面文件必须 ≥ 物理内存 + 1MB
   - CrashDumpEnabled = 2（内核转储）→ 页面文件建议 ≥ 物理内存的 1/3
   - CrashDumpEnabled = 7（自动内存转储，Server 2012+ 默认）→ 系统自动计算所需大小
   - 未配置页面文件时崩溃转储无法生成，蓝屏转储文件将丢失

### Step 4: 硬件保留内存

**数据采集**：

> 采集目标：硬件保留内存量和总物理内存，判断硬件保留是否异常偏高

```powershell
# Use kernel32 GetPhysicallyInstalledSystemMemory for true installed RAM (unaffected by OS activation state)
$kernel32 = Add-Type -MemberDefinition @'
[DllImport("kernel32.dll")]
public static extern bool GetPhysicallyInstalledSystemMemory(out ulong TotalMemoryInKilobytes);
'@ -Name 'NativeMemory' -Namespace 'Win32' -PassThru
[UInt64]$installedKB = 0
[Win32.NativeMemory]::GetPhysicallyInstalledSystemMemory([ref]$installedKB)
$installedPhysical = $installedKB * 1024

# Get OS-visible memory (already excludes hardware reserved)
$os = Get-CimInstance Win32_OperatingSystem
$visiblePhysical = $os.TotalVisibleMemorySize * 1024
$reservedForHardware = $installedPhysical - $visiblePhysical

[PSCustomObject]@{
    InstalledPhysicalMB     = [math]::Round($installedPhysical / 1MB)
    VisiblePhysicalMB       = [math]::Round($visiblePhysical / 1MB)
    ReservedForHardwareMB   = [math]::Round($reservedForHardware / 1MB)
} | Format-List
```

**分析思路**：

1. 检查硬件保留内存量：
   - 硬件保留小于 2GB → 正常
   - 硬件保留超过 2GB 且可见物理内存不超过 2GB → **根因**：硬件保留内存过高，可能由于 Windows 激活异常或驱动分配问题导致系统可用内存远小于物理内存，**严重程度**：Warning

2. 硬件保留内存过高的常见原因：
   - Windows 未激活或许可证异常 → 参见 [system-activation.md](references/system-activation.md)
   - msconfig 中设置了"最大内存"限制（Boot Advanced Options → Maximum Memory）→ 检查 bcdedit /enum 中是否存在 truncatememory 或 removememory 配置（见 Step 6）
   - 32 位操作系统的内存寻址上限（最大 4GB，Server 版本与 PAE 可扩展）
   - BIOS/UEFI 配置分配给集成显卡或其他外设的内存（云服务器场景通常不适用）

### Step 5: 超线程状态

**数据采集**：

> 采集目标：CPU 物理核心数和逻辑处理器数，判断超线程是否启用

```powershell
Get-CimInstance Win32_Processor | Select-Object DeviceID, NumberOfCores, NumberOfLogicalProcessors, @{Name='HyperThreading';Expression={
    if ($_.NumberOfLogicalProcessors -gt $_.NumberOfCores) { 'Enabled' } else { 'Disabled' }
}} | Format-Table -AutoSize

# Check if hyper-threading was disabled via security patch registry settings
$mm = Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management" -ErrorAction SilentlyContinue
[PSCustomObject]@{
    FeatureSettingsOverride     = $mm.FeatureSettingsOverride
    FeatureSettingsOverrideMask = $mm.FeatureSettingsOverrideMask
} | Format-List
```

**分析思路**：

1. 检查超线程状态（比较物理核心数和逻辑处理器数）：
   - 逻辑处理器数 > 物理核心数 → 超线程已启用，正常
   - 逻辑处理器数 = 物理核心数 → **根因**：超线程未启用，系统可见的 CPU 核心数为实例规格的一半，**严重程度**：Warning

2. 检查注册表缓解措施配置：
   - FeatureSettingsOverride 和 FeatureSettingsOverrideMask 用于控制 Spectre、Meltdown、L1TF、MDS 等 CPU 微架构漏洞的缓解策略
   - 当缓解配置包含禁用超线程的值时（如 L1TF/MDS 缓解中 FeatureSettingsOverride 包含位 6，即 0x40 或更高值）→ **根因**：通过注册表应用了 CPU 微架构安全缓解措施，该配置禁用了超线程，**严重程度**：Warning
   - 未配置或值不包含禁用超线程的位 → 正常
   - 注意：这些注册表项位于 `HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management`，移除缓解措施会降低系统对 CPU 微架构漏洞的防护，需权衡性能与安全风险

### Step 6: BCD 启动配置限制

**数据采集**：

> 采集目标：BCD 启动配置中是否存在 CPU 数量或内存大小的限制

```powershell
# BCD boot configuration
bcdedit /enum
```

**分析思路**：

1. 检查 BCD 当前启动项中的处理器限制：
   - 无 numproc 或 usebootprocessoronly 配置 → 正常，系统使用所有可用 CPU
   - 存在 numproc 配置（如 `numproc  1`）→ **根因**：BCD 启动配置限制了可用处理器数量，系统只能使用部分 CPU，**严重程度**：Warning
   - 存在 usebootprocessoronly 且值为 Yes → **根因**：BCD 配置为仅使用启动处理器，系统只能使用 1 个 CPU，**严重程度**：Warning
   - 注意：这些配置也可能通过 msconfig → Boot → Advanced options → "Number of processors" 复选框设置

2. 检查 BCD 中的内存限制：
   - 无 truncatememory 或 removememory 配置 → 正常
   - 存在 truncatememory 或 removememory → **根因**：BCD 启动配置截断或移除了部分内存，系统可用内存小于实际物理内存，**严重程度**：Warning
   - 注意：truncatememory 也可能通过 msconfig → Boot → Advanced options → "Maximum memory" 复选框设置

### Step 7: 电源计划

**数据采集**：

> 采集目标：当前生效的电源计划

```powershell
powercfg /getactivescheme
powercfg /list
```

**分析思路**：

1. 检查当前电源计划：
   - 高性能（High Performance，GUID: `8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c`）或终极性能（Ultimate Performance）→ 正常，CPU 将以最大频率运行
   - 平衡（Balanced，GUID: `381b4222-f694-41f0-9685-ff5bb260df2e`）→ Windows Server 2016+ 默认电源计划，会根据 CPU 负载动态调整频率；对于延迟敏感的服务器场景（如数据库、实时处理），建议切换为高性能
   - 节能（Power Saver，GUID: `a1841308-3541-4fab-bc81-f71556f20b4a`）→ **根因**：电源计划设置为节能模式，CPU 被限制在低频率运行，影响系统整体性能，**严重程度**：Warning

2. 服务器场景补充说明：
   - 微软官方建议服务器使用 Balanced 或 High Performance 电源计划
   - Windows Server 2016+ 的 Balanced 方案已对服务器场景优化，响应速度接近 High Performance
   - 但如果用户确认存在性能问题且当前为 Balanced，切换到 High Performance 可消除频率调整延迟因素

### Step 8: 文件系统过滤驱动检查

**数据采集**：

> 采集目标：获取所有已注册的文件系统微过滤驱动（Minifilter）列表，检查是否存在第三方过滤驱动

```powershell
fltmc filters
```

**分析思路**：

1. 检查是否存在非 Microsoft 签名的第三方过滤驱动：
   - 正常：仅包含 Microsoft 标准过滤驱动（如 WdFilter、fvevol、luafv、FileInfo、Dedup、wcifs 等）
   - 异常：存在非 Microsoft 过滤驱动 → 检查该驱动是否已知会导致性能问题
   - 说明：第三方文件系统过滤驱动会拦截所有文件 I/O 操作（打开、读写、删除等），驱动异常或设计不当会显著拖慢文件操作和程序启动速度

2. 已知会导致系统缓慢的第三方过滤驱动：
   - **CcProtect**：某安全软件的过滤驱动，会延迟所有文件打开操作，表现为双击文件或程序后长时间无响应
   - 其他安全软件、备份软件、加密软件的过滤驱动也可能导致类似问题

3. 检查 Num Instances 列：
   - 正常：每个驱动的实例数在其预期范围内
   - 异常：某个驱动的实例数异常偏高，可能存在资源泄漏

如果发现第三方过滤驱动且确认其为性能问题来源，参见 → [desktop-app.md](references/desktop-app.md)（安全软件进程/过滤驱动排查）

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | CPU 占用 Top 进程中包含杀毒软件 | → [desktop-app.md](references/desktop-app.md) |
| 条件跳转 | 硬件保留内存过高且疑似激活异常 | → [system-activation.md](references/system-activation.md) |
| 条件跳转 | 系统启动慢、频繁意外重启 | → [performance-lifecycle.md](references/performance-lifecycle.md) |
| 条件跳转 | 发现第三方文件系统过滤驱动且怀疑为性能问题来源 | → [desktop-app.md](references/desktop-app.md) |
| 链式后继 | 本文件未确认根因 | → [performance-lifecycle.md](references/performance-lifecycle.md) |

## 修复建议

### 根因：页面文件未配置

**修复操作**：

```powershell
# Enable system-managed automatic page file
$cs = Get-CimInstance Win32_ComputerSystem
$cs | Set-CimInstance -Property @{AutomaticManagedPagefile=$true}
```

**验证方法**：

```powershell
(Get-CimInstance Win32_ComputerSystem).AutomaticManagedPagefile
```

预期结果：返回 True

**风险说明**：启用自动管理页面文件后，需重启系统生效。页面文件大小将由系统自动管理。

### 根因：BCD 启动配置限制处理器数量

**修复操作**：

```powershell
# Remove processor count limit from BCD
bcdedit /deletevalue numproc
bcdedit /deletevalue usebootprocessoronly
```

**验证方法**：

```powershell
# Verify BCD configuration
bcdedit /enum | Select-String -Pattern "numproc|usebootprocessoronly"
```

预期结果：无输出（说明配置已移除），重启后系统将使用所有可用 CPU

**风险说明**：修改 BCD 配置需重启系统生效。如果是有意为之的限制（如应用兼容性问题），移除后可能导致应用异常。

### 根因：BCD 启动配置截断内存

**修复操作**：

```powershell
# Remove memory truncation configuration from BCD
bcdedit /deletevalue truncatememory
bcdedit /deletevalue removememory
```

**验证方法**：

```powershell
# Verify BCD configuration
bcdedit /enum | Select-String -Pattern "truncatememory|removememory"
```

预期结果：无输出，重启后系统将使用全部物理内存

**风险说明**：修改 BCD 配置需重启系统生效。如果内存截断是为解决特定硬件兼容问题，移除后可能导致系统不稳定。

### 根因：超线程未启用

**修复操作**：

超线程由 ECS 实例规格和 BIOS 配置决定，无法在操作系统内部修改。如果是注册表缓解措施导致的禁用：

```powershell
# Remove hyper-threading disable configuration from CPU microarchitecture security mitigations
Remove-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management" -Name FeatureSettingsOverride -ErrorAction SilentlyContinue
Remove-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management" -Name FeatureSettingsOverrideMask -ErrorAction SilentlyContinue
```

**验证方法**：

```powershell
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management" -Name FeatureSettingsOverride, FeatureSettingsOverrideMask -ErrorAction SilentlyContinue | Select-Object FeatureSettingsOverride, FeatureSettingsOverrideMask
```

预期结果：属性不存在（已移除），重启后超线程将恢复

**风险说明**：移除 CPU 微架构安全缓解措施后，系统将不再拥有针对 Spectre/Meltdown 类漏洞的保护。请根据安全需求权衡。修改需重启系统生效。

### 根因：电源计划为节能模式

**修复操作**：

```powershell
# Switch to High Performance power plan
powercfg /setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c
```

**验证方法**：

```powershell
powercfg /getactivescheme
```

预期结果：显示 High Performance

**风险说明**：高性能电源计划下 CPU 始终以最大频率运行，不会根据负载调整频率。对云服务器场景无明显副作用。

### 根因：第三方文件系统过滤驱动导致系统缓慢

> 对应诊断：Step 8 (文件系统过滤驱动检查) - 分析思路第 1、2 条

**修复操作**：

```powershell
# Check third-party filter driver details (replace <FilterName> with actual name, e.g. CcProtect)
fltmc volumes <FilterName>

# Uninstall the corresponding software via Control Panel or use the vendor's uninstall tool
# If uninstall not possible, try stopping and disabling the driver service:
# sc query <DriverServiceName>
# sc stop <DriverServiceName>
# sc config <DriverServiceName> start= disabled
```

**验证方法**：

```powershell
# Verify filter driver has been removed
fltmc filters | Select-String -Pattern "<FilterName>"
```

预期结果：无匹配输出（过滤驱动已移除），文件打开和程序启动速度恢复正常

**风险说明**：卸载安全软件或禁用其过滤驱动会降低系统安全防护。请在确认该驱动为性能问题根因后再操作，建议先联系软件厂商获取官方卸载工具
