# Storage VSS (Volume Shadow Copy Service) 诊断

## 目录

- [Storage VSS (Volume Shadow Copy Service) 诊断](#storage-vss-volume-shadow-copy-service-诊断)
  - [目录](#目录)
  - [功能说明](#功能说明)
  - [步骤选取指引](#步骤选取指引)
  - [诊断步骤](#诊断步骤)
    - [Step 1: VSS 服务与依赖服务检查](#step-1-vss-服务与依赖服务检查)
    - [Step 2: VSS Writer 状态检查](#step-2-vss-writer-状态检查)
    - [Step 3: VSS Provider 检查](#step-3-vss-provider-检查)
    - [Step 4: VSS 快照与存储空间检查](#step-4-vss-快照与存储空间检查)
    - [Step 5: VSS 事件日志分析](#step-5-vss-事件日志分析)
    - [Step 6: 备份软件状态检查](#step-6-备份软件状态检查)
  - [交叉引用](#交叉引用)
  - [修复建议](#修复建议)
    - [根因：VSS 服务被禁用](#根因vss-服务被禁用)
    - [根因：VSS 依赖服务异常](#根因vss-依赖服务异常)
    - [根因：VSS Writer 状态异常](#根因vss-writer-状态异常)
    - [根因：VSS Provider 注册异常](#根因vss-provider-注册异常)
    - [根因：第三方 VSS Provider 残留](#根因第三方-vss-provider-残留)
    - [根因：VSS 快照存储空间不足](#根因vss-快照存储空间不足)
    - [根因：VSS 事件日志报告错误](#根因vss-事件日志报告错误)
    - [根因：备份执行失败](#根因备份执行失败)

## 功能说明

诊断 Windows 卷影复制服务（VSS）相关问题：VSS 服务及其依赖服务状态异常、VSS Writer 状态异常（Failed / Unstable / Waiting for completion）、VSS Provider 注册问题、VSS 快照创建失败与存储空间不足、VSS 相关事件日志错误（Event ID 8193 / 12289 / 12293）、Windows Server Backup 执行失败。覆盖 9 个已知问题项。

**输入**：用户问题描述（必选）、错误代码/事件ID/截图（可选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 备份失败、提示 VSS 错误 | Step 1 (VSS 服务与依赖) → Step 2 (VSS Writer 状态) → Step 5 (VSS 事件日志) |
| VSS Writer 状态 Failed / Unstable | Step 1 (VSS 服务与依赖) → Step 2 (VSS Writer 状态) |
| 快照创建失败、还原点无法创建 | Step 1 (VSS 服务与依赖) → Step 3 (VSS Provider) → Step 4 (VSS 快照与存储) |
| 事件日志中出现 VSS 错误（8193 / 12289 / 12293） | Step 5 (VSS 事件日志) → Step 1 (VSS 服务与依赖) → Step 2 (VSS Writer 状态) |
| Windows Server Backup 报错 | Step 1 (VSS 服务与依赖) → Step 2 (VSS Writer 状态) → Step 6 (备份软件) |
| 第三方备份软件（Veeam / Acronis 等）失败 | Step 2 (VSS Writer 状态) → Step 3 (VSS Provider) → Step 5 (VSS 事件日志) |

## 诊断步骤

### Step 1: VSS 服务与依赖服务检查

**数据采集**：

> 采集目标：获取 VSS 服务及其核心依赖服务（COM+ Event System、RPC、DCOM Server Process Launcher）的运行状态和启动类型

```powershell
# VSS service and core dependency service status
$serviceNames = @('VSS', 'EventSystem', 'RpcSs', 'DcomLaunch', 'CryptSvc', 'swprv')
Get-Service -Name $serviceNames -ErrorAction SilentlyContinue | Select-Object Name, DisplayName, Status, StartType
```

**分析思路**：

1. 检查 VSS 服务状态和启动类型：
   - 正常：VSS 服务启动类型为 Manual（手动），可按需启动；在无备份或快照操作时处于 Stopped 状态是正常的
   - 异常：VSS 服务启动类型为 Disabled → **根因**：VSS 服务被禁用，**严重程度**：Critical

2. 检查 COM+ Event System 服务：
   - 正常：服务处于 Running 状态，启动类型为 Automatic
   - 异常：服务未运行或被禁用 → **根因**：VSS 依赖服务异常，**严重程度**：Critical
   - 说明：VSS 依赖 COM+ Event System 进行组件间通信，该服务异常会导致所有 VSS 操作失败

3. 检查 RPC 和 DCOM 服务：
   - 正常：RpcSs 和 DcomLaunch 均处于 Running 状态
   - 异常：任一服务未运行 → **根因**：VSS 依赖服务异常，**严重程度**：Critical

4. 检查 Software Shadow Copy Provider 服务（swprv）：
   - 正常：启动类型为 Manual，按需启动
   - 异常：被禁用 → **根因**：VSS 服务被禁用，**严重程度**：Critical
   - 说明：swprv 是 Windows 内置的 VSS 软件快照提供程序，被禁用时无法创建快照

### Step 2: VSS Writer 状态检查

**数据采集**：

> 采集目标：获取所有 VSS Writer 的名称、状态和错误信息，识别处于异常状态的 Writer

```powershell
# List all VSS Writers and their status
vssadmin list writers
```

**分析思路**：

1. 检查所有 Writer 的运行状态：
   - 正常：所有 Writer 状态显示为 Stable
   - 异常：存在 Writer 状态为以下任一值 → **根因**：VSS Writer 状态异常，**严重程度**：Warning
     - **Failed**：Writer 遇到错误，需要重启对应的控制服务
     - **Unstable**：Writer 处于不稳定状态，需要重启对应的控制服务
     - **Waiting for completion**：Writer 正被某个进程占用，或上一次操作未正常完成；如果当前没有任何备份在运行，则说明 Writer 卡住了，同样需要重启

2. 识别异常 Writer 并确定对应的控制服务：
   - 每个 VSS Writer 由一个特定的 Windows 服务控制，重启该服务可将 Writer 恢复到 Stable 状态
   - 常见 Writer 与控制服务的对应关系（用于指导修复）：

     | Writer 名称 | 控制服务 |
     |------------|--------|
     | System Writer | Cryptographic Services (CryptSvc) |
     | WMI Writer | Windows Management Instrumentation (Winmgmt) |
     | COM+ REGDB Writer | COM+ System Application (COMSysApp) |
     | Task Scheduler Writer | Task Scheduler (Schedule) |
     | Registry Writer | VSS (VSS) |
     | VSS Metadata Store Writer | VSS (VSS) |
     | Performance Counters Writer | VSS (VSS) |
     | SqlServerWriter | SQL Server (MSSQLSERVER) |
     | BITS Writer | Background Intelligent Transfer (BITS) |
     | IIS Config Writer | IIS Admin Service (IISADMIN) |
     | Hyper-V Writer | Hyper-V Virtual Machine Management (vmms) |

   - 说明：如果 Writer 名称不在上表中，可通过 Writer 的 Instance 信息在 Windows 服务列表中查找对应服务

3. 检查是否存在多个 Writer 同时异常：
   - 如果只有单个 Writer 异常，通常是该 Writer 对应的服务/应用出了问题
   - 如果大量 Writer 同时异常，通常是 VSS 基础设施（VSS 服务本身或 COM+ Event System）的问题，优先检查 Step 1

### Step 3: VSS Provider 检查

**数据采集**：

> 采集目标：获取系统中注册的所有 VSS Provider，识别是否存在第三方 Provider 以及 Provider 是否正常注册

```powershell
# List all VSS Providers
vssadmin list providers
```

**分析思路**：

1. 检查系统内置 Provider 是否存在：
   - 正常：至少存在 Microsoft Software Shadow Copy provider 1.0（Provider Id: {b5946137-7b9f-4925-af80-51abd60b20d5}）
   - 异常：内置 Provider 缺失 → **根因**：VSS Provider 注册异常，**严重程度**：Critical

2. 检查是否存在第三方 Provider：
   - 正常：仅有 Microsoft 内置 Provider
   - 注意：第三方备份软件（如 Veeam、Acronis、Veritas）可能注册自己的 VSS Provider
   - 异常：第三方 Provider 存在但对应软件已卸载，残留的 Provider 注册信息可能导致 VSS 快照创建失败 → **根因**：第三方 VSS Provider 残留，**严重程度**：Warning

### Step 4: VSS 快照与存储空间检查

**数据采集**：

> 采集目标：获取当前系统上的 VSS 快照列表、VSS 存储空间使用情况和配置上限

```powershell
# List all VSS snapshots
vssadmin list shadows

# VSS storage usage and configuration
vssadmin list shadowstorage
```

**分析思路**：

1. 检查 VSS 快照列表：
   - 正常：能够正常列出快照，或列表为空（未配置自动快照属于正常情况）
   - 异常：命令执行报错 → 结合 Step 1 检查 VSS 服务状态

2. 检查 VSS 存储空间使用率：
   - 正常：已用存储空间远小于最大限制
   - 异常：已用空间接近或达到最大限制（Used Shadow Copy Storage 接近 Maximum Shadow Copy Storage） → **根因**：VSS 快照存储空间不足，**严重程度**：Warning
   - 说明：当快照存储空间耗尽时，新快照创建将失败，且最旧的快照可能被自动删除

3. 检查快照数量：
   - 正常：每个卷的快照数量未达到上限（Windows 每卷最多 512 个快照）
   - 异常：快照数量过多（接近 512 个） → **根因**：VSS 快照存储空间不足，**严重程度**：Warning

### Step 5: VSS 事件日志分析

**数据采集**：

> 采集目标：获取最近 7 天内 Application 日志和 System 日志中 VSS 相关的错误和警告事件

```powershell
# VSS events in Application log
Get-WinEvent -FilterHashtable @{LogName='Application'; Level=1,2,3; StartTime=(Get-Date).AddDays(-7)} -MaxEvents 30 -ErrorAction SilentlyContinue | Where-Object { $_.ProviderName -eq 'VSS' } | Select-Object TimeCreated, Id, LevelDisplayName, Message | Format-Table -AutoSize

# volsnap (snapshot driver) events in System log
Get-WinEvent -FilterHashtable @{LogName='System'; Level=1,2,3; StartTime=(Get-Date).AddDays(-7)} -MaxEvents 20 -ErrorAction SilentlyContinue | Where-Object { $_.ProviderName -eq 'volsnap' } | Select-Object TimeCreated, Id, LevelDisplayName, Message | Format-Table -AutoSize
```

**分析思路**：

1. 检查是否存在 VSS 错误事件，重点关注以下常见 Event ID：

   | Event ID | 含义 | 典型根因 |
   |----------|------|--------|
   | 8193 | VSS 操作调用失败 | 权限问题、COM 组件注册异常、依赖服务不可用 |
   | 8194 | VSS 无法获取 Writer 信息 | Writer 对应的服务异常 |
   | 12289 | VSS Provider 操作失败 | 第三方 Provider 异常、磁盘 I/O 错误 |
   | 12293 | VSS Shadow Copy Provider 调用超时 | 磁盘 I/O 过高导致快照超时 |
   | 12298 | VSS 快照创建超时 | 系统负载高、磁盘响应慢 |

   - 正常：无 VSS 错误或警告事件
   - 异常：存在上述事件 → **根因**：VSS 事件日志报告错误，**严重程度**：Warning
   - 说明：Event ID 12293 / 12298 通常与磁盘 I/O 性能有关，高负载期间磁盘无法在超时窗口内完成快照操作

2. 检查 volsnap 事件：
   - volsnap 是 Windows 快照驱动，记录快照存储层面的问题
   - 异常：出现 diff area（差异区域）空间不足的事件 → 关联到 Step 4 的存储空间检查

如果事件日志指向磁盘 I/O 性能问题，参见 → [storage-hardware.md](references/storage-hardware.md)

### Step 6: 备份软件状态检查

**数据采集**：

> 采集目标：获取 Windows Server Backup 功能安装状态和最近的备份执行结果

```powershell
# Check if Windows Server Backup feature is installed
Get-WindowsFeature -Name Windows-Server-Backup -ErrorAction SilentlyContinue | Select-Object Name, InstallState | Format-Table -AutoSize

# Recent backup-related events
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Backup'; StartTime=(Get-Date).AddDays(-7)} -MaxEvents 20 -ErrorAction SilentlyContinue | Select-Object TimeCreated, Id, LevelDisplayName, Message | Format-Table -AutoSize

```

**分析思路**：

1. 检查备份功能安装状态：
   - 正常：Windows-Server-Backup 功能已安装
   - 异常：功能未安装 → 告知用户需要先安装该功能

2. 检查备份执行结果：
   - 正常：最近的备份事件显示成功
   - 异常：备份事件显示失败 → **根因**：备份执行失败，**严重程度**：Warning
   - 说明：备份失败通常与 VSS Writer 异常相关，需结合 Step 2 的 Writer 状态综合判断；如果备份日志中明确提到某个 Writer 名称，可直接定位到对应的 Writer 和控制服务

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Step 4 发现快照存储空间不足，需检查卷剩余空间 | → [storage-disk.md](references/storage-disk.md)（检查卷空间使用率） |
| 条件跳转 | Step 5 事件日志指向磁盘 I/O 性能或存储驱动异常 | → [storage-hardware.md](references/storage-hardware.md) |
| 参数化引用 | Step 3 发现第三方 Provider 残留涉及驱动/设备问题 | → [device-driver.md](references/device-driver.md)（检查第三方备份软件相关驱动） |
| 链式后继 | 本文件所有步骤执行完毕，未确认根因 | → [storage-disk.md](references/storage-disk.md) |

## 修复建议

### 根因：VSS 服务被禁用

**修复操作**：

```powershell
# Restore VSS service and Software Shadow Copy Provider startup type to Manual
Set-Service -Name VSS -StartupType Manual
Set-Service -Name swprv -StartupType Manual

# Start VSS service
Start-Service -Name VSS
```

**验证方法**：

```powershell
Get-Service -Name VSS, swprv | Select-Object Name, Status, StartType
```

预期结果：启动类型为 Manual，VSS 服务可正常启动

**风险说明**：修改服务启动类型无风险。VSS 和 swprv 的默认启动类型是 Manual（按需启动），恢复为 Manual 即可

---

### 根因：VSS 依赖服务异常

**修复操作**：

```powershell
# Ensure dependent services have correct startup type and are running
$deps = @('EventSystem', 'RpcSs', 'DcomLaunch')
foreach ($svc in $deps) {
    Set-Service -Name $svc -StartupType Automatic -ErrorAction SilentlyContinue
    Start-Service -Name $svc -ErrorAction SilentlyContinue
}

# Restart VSS service
Restart-Service -Name VSS -Force -ErrorAction SilentlyContinue
```

**验证方法**：

```powershell
Get-Service -Name EventSystem, RpcSs, DcomLaunch, VSS | Select-Object Name, Status, StartType
```

预期结果：所有服务处于 Running 状态

**风险说明**：COM+ Event System、RPC、DCOM 是 Windows 核心服务，正常情况下不应被禁用。重启这些服务可能短暂影响依赖它们的其他应用

---

### 根因：VSS Writer 状态异常

**修复操作**：

修复分两步：先尝试重启异常 Writer 对应的控制服务，如果无效再进行 VSS 组件重注册。

```powershell
# Step 1: Restart the control service corresponding to the abnormal Writer
# Based on Step 2 diagnosis, replace <ServiceName> with actual control service name
# Common mappings:
#   System Writer        -> CryptSvc
#   WMI Writer           -> Winmgmt
#   COM+ REGDB Writer    -> COMSysApp
#   Task Scheduler Writer -> Schedule
#   BITS Writer          -> BITS
#   SqlServerWriter      -> MSSQLSERVER
Restart-Service -Name <ServiceName> -Force
```

```powershell
# Step 2 (only execute if Step 1 is ineffective): Re-register VSS components
Stop-Service -Name vss -Force
regsvr32 /s ole32.dll
regsvr32 /s vss_ps.dll
vssvc /register
Start-Service -Name vss
```

**验证方法**：

```powershell
vssadmin list writers
```

预期结果：所有 Writer 状态为 Stable

**风险说明**：重启控制服务会短暂中断该服务管理的功能（例如重启 MSSQLSERVER 会导致 SQL Server 短暂不可用）。需要评估业务影响后再执行

---

### 根因：VSS Provider 注册异常

**修复操作**：

```powershell
# Re-register Windows built-in VSS Provider
Stop-Service -Name swprv -Force
Start-Service -Name swprv

# Re-register VSS service
Stop-Service -Name vss -Force
vssvc /register
Start-Service -Name vss
```

**验证方法**：

```powershell
vssadmin list providers
```

预期结果：Microsoft Software Shadow Copy provider 1.0 正常列出

**风险说明**：重注册操作不修改数据，风险较低

---

### 根因：第三方 VSS Provider 残留

**修复操作**：

```powershell
# First confirm residual third-party Provider (get Provider Id from Step 3 diagnosis)
vssadmin list providers

# Remove residual third-party Provider from registry (replace <Provider-GUID> with actual value)
# Note: This operation requires administrator privileges
# Remove-Item -Path "HKLM:\SYSTEM\CurrentControlSet\Services\VSS\Providers\<Provider-GUID>" -Recurse -Force
```

**验证方法**：

```powershell
vssadmin list providers
vssadmin create shadow /for=C:
```

预期结果：仅列出 Microsoft 内置 Provider，快照创建成功

**风险说明**：删除第三方 Provider 注册表项后，如果对应备份软件仍在使用，该软件的快照功能将不可用。操作前需确认对应的备份软件确实已被卸载

---

### 根因：VSS 快照存储空间不足

**修复操作**：

```powershell
# Option 1: Clean old VSS snapshots to free space (specify by volume)
vssadmin delete shadows /for=<DriveLetter>: /oldest /quiet

# Option 2: Adjust VSS storage space limit
vssadmin resize shadowstorage /for=<DriveLetter>: /on=<DriveLetter>: /maxsize=<Size>GB

# Option 3: Clear all snapshots and reconfigure
# vssadmin delete shadows /all /quiet
```

**验证方法**：

```powershell
vssadmin list shadowstorage
vssadmin create shadow /for=<DriveLetter>:
```

预期结果：存储空间使用率下降，新快照创建成功

**风险说明**：删除 VSS 快照会导致对应的系统还原点和文件版本历史丢失。建议优先使用方案 1（仅删除最旧快照）或方案 2（扩大上限），而非方案 3（全部清除）

---

### 根因：VSS 事件日志报告错误

**说明**：

VSS 事件日志错误通常是其他根因的表象，需要根据具体 Event ID 关联到对应的根因进行修复：

| Event ID | 关联根因 | 修复方向 |
|----------|---------|--------|
| 8193 | VSS 依赖服务异常 / Provider 注册异常 | 检查 COM 组件注册和依赖服务 |
| 8194 | VSS Writer 状态异常 | 重启异常 Writer 的控制服务 |
| 12289 | VSS Provider 注册异常 / 第三方 Provider 残留 | 检查 Provider 注册 |
| 12293 / 12298 | 磁盘 I/O 性能问题 | 降低磁盘负载后重试，或在低峰期执行备份 |

如果 Event ID 12293 / 12298 频繁出现，说明磁盘 I/O 延迟过高导致 VSS 操作在超时窗口内无法完成。排查存储性能问题，参见 → [storage-hardware.md](references/storage-hardware.md)

---

### 根因：备份执行失败

**修复操作**：

```powershell
# Ensure Windows Server Backup feature is installed
Install-WindowsFeature -Name Windows-Server-Backup -IncludeManagementTools

# Re-run backup (example: backup system state, replace <BackupTarget> with actual path)
# wbadmin start systemstatebackup -backuptarget:<BackupTarget>
```

**验证方法**：

```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Backup'; StartTime=(Get-Date).AddDays(-1)} -MaxEvents 5 -ErrorAction SilentlyContinue | Select-Object TimeCreated, LevelDisplayName, Message
```

预期结果：备份成功完成，无错误事件

**风险说明**：备份操作会占用系统资源（CPU、磁盘 I/O），建议在维护窗口执行
