# Performance Lifecycle 诊断

## 功能说明

诊断 Windows 关机卡住/超时、待重启操作状态、系统启动耗时、启动阶段分解、关机耗时分析，以及异常关机/重启事件。覆盖关机超时配置、待重启标记检测、启动耗时分析、关机事件日志分析、启动阶段分解、关机耗时分析 6 个诊断步骤。

**输入**：用户问题描述（必选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 关机慢、正在关机界面卡住 | Step 1 (关机超时配置与自动终止策略) |
| 更新安装失败、提示需先重启 | Step 1 → Step 2 (待重启操作状态) |
| 系统启动慢、开机时间长 | Step 3 (启动耗时与运行时长) → Step 5 (启动阶段分解) → Step 4 (关机/重启事件分析) |
| 系统无故重启、异常关机 | Step 3 → Step 4 |
| 不确定系统是否被意外关机过 | Step 3 → Step 4 |
| 关机过程耗时长、关机慢 | Step 6 (关机耗时分析) → Step 1 (关机超时配置) |
| 启动卡在某个阶段（如「正在启动 Windows」） | Step 5 (启动阶段分解) → Step 3 → [device-driver.md](references/device-driver.md) |

## 诊断步骤

### Step 1: 关机超时配置与自动终止策略

**数据采集**：

> 采集目标：关机等待服务超时（WaitToKillServiceTimeout）、等待应用超时（WaitToKillAppTimeout）、挂起应用判定超时（HungAppTimeout）、自动终止未响应任务策略（AutoEndTasks）、GPO 机器级关机脚本

```powershell
# Shutdown core timeout configuration
[PSCustomObject]@{
    WaitToKillServiceTimeout = (Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control" -Name WaitToKillServiceTimeout -ErrorAction SilentlyContinue).WaitToKillServiceTimeout
    WaitToKillAppTimeout    = (Get-ItemProperty -Path "HKCU:\Control Panel\Desktop" -Name WaitToKillAppTimeout -ErrorAction SilentlyContinue).WaitToKillAppTimeout
    HungAppTimeout          = (Get-ItemProperty -Path "HKCU:\Control Panel\Desktop" -Name HungAppTimeout -ErrorAction SilentlyContinue).HungAppTimeout
    AutoEndTasks            = (Get-ItemProperty -Path "HKCU:\Control Panel\Desktop" -Name AutoEndTasks -ErrorAction SilentlyContinue).AutoEndTasks
} | Format-List

# GPO machine-level shutdown scripts
Get-ChildItem -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Group Policy\State\Machine\Scripts\Shutdown" -ErrorAction SilentlyContinue | ForEach-Object {
    $s = Get-ItemProperty -Path $_.PSPath -ErrorAction SilentlyContinue
    [PSCustomObject]@{Scope='Machine'; Script=$s.Script; Parameters=$s.Parameters}
} | Format-Table -AutoSize
```

**分析思路**：

1. 检查 WaitToKillServiceTimeout（服务等待超时，单位 ms）：
   - 默认 5000（5 秒）；未配置时系统使用默认值 → 正常
   - 值被设为极端大（≥ 600000，即 10 分钟）→ **根因**：关机等待服务超时时间过长，单个服务卡住即可导致关机长时间等待，**严重程度**：Warning
   - 值被设为极端小（< 2000）→ **根因**：关机等待服务超时过短，服务可能被强制终止导致数据丢失，**严重程度**：Warning

2. 检查 WaitToKillAppTimeout（应用等待超时，单位 ms）：
   - 默认 20000（20 秒）；未配置 → 正常
   - 值被设为极端大（≥ 120000，即 2 分钟）→ **根因**：关机等待应用程序超时过长，**严重程度**：Warning

3. 检查 HungAppTimeout（挂起应用判定超时，单位 ms）：
   - 默认 5000（5 秒）→ 正常
   - 值异常大（≥ 30000）→ **根因**：系统判定应用"未响应"的等待时间过长，挂起应用会拖延关机流程，**严重程度**：Warning

4. 检查 AutoEndTasks（自动终止未响应任务）：
   - 值为 1 → 系统自动终止未响应任务，关机流程不会卡在等待用户操作
   - 值为 0 或不存在（默认值）→ 关机时可能弹出"等待程序关闭"对话框，需用户手动点击才能继续
   - 在无人值守服务器场景下，默认值会导致关机卡在等待用户操作界面 → **根因**：自动终止任务策略未启用，无人值守时关机可能卡在等待用户操作界面，**严重程度**：Warning

5. 检查 GPO 关机脚本：
   - 关机脚本为空 → 正常
   - 存在关机脚本且关机卡住 → 脚本可能执行超时或挂起

> 如果 GPO 关机脚本阻塞关机流程，参见 → [system-gpo.md](references/system-gpo.md)（组策略应用问题）

### Step 2: 待重启操作状态

**数据采集**：

> 采集目标：PendingFileRenameOperations（挂起文件重命名/删除列表）、CBS（Component Based Servicing）RebootPending 标记、Windows Update RebootRequired 标记、CBS Sessions 中未完成的操作数

```powershell
# PendingFileRenameOperations (pending file operations)
$renameOps = Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager" -Name PendingFileRenameOperations -ErrorAction SilentlyContinue | Select-Object -ExpandProperty PendingFileRenameOperations
if ($renameOps) {
    $renameOps
}

# CBS reboot pending marker
$cbsRebootPending = (Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing" -Name RebootPending -ErrorAction SilentlyContinue).RebootPending
if ($cbsRebootPending -eq 1) {
    [PSCustomObject]@{Key='Component Based Servicing'; RebootPending=$cbsRebootPending} | Format-Table -AutoSize
}

# Windows Update RebootRequired
if (Test-Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired") {
    [PSCustomObject]@{RebootRequired=$true} | Format-Table -AutoSize
}

# CBS Sessions pending operations
Get-ChildItem -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\Sessions" -ErrorAction SilentlyContinue | ForEach-Object {
    $po = Get-ItemProperty -Path $_.PSPath -Name PendingOperations -ErrorAction SilentlyContinue
    if ($po.PendingOperations -gt 0) {
        $to = Get-ItemProperty -Path $_.PSPath -Name TotalOperations -ErrorAction SilentlyContinue
        [PSCustomObject]@{Session=$_.PSChildName; Total=$to.TotalOperations; Pending=$po.PendingOperations}
    }
} | Format-Table -AutoSize
```

**分析思路**：

1. 检查 PendingFileRenameOperations：
   - 空输出 → 正常，无挂起的文件替换或删除操作
   - 非空（有输出）→ **根因**：存在挂起的文件操作（重命名或删除），需重启后完成，**严重程度**：Info
   - 列表中包含 .sys 驱动文件 → 通常由 Windows Update 或驱动安装产生，重启以完成驱动替换
   - 同一批文件操作在多次重启后仍然存在 → **根因**：挂起的文件操作持续无法完成，可能因文件锁定或权限问题，**严重程度**：Warning

2. 检查 CBS RebootPending：
   - 无输出 → 正常，组件服务无待重启操作
   - RebootPending = 1 → **根因**：组件服务（CBS）有待处理的更新操作，需要重启才能完成安装，**严重程度**：Warning

3. 检查 Windows Update RebootRequired：
   - 无输出 → 正常
   - RebootRequired = True → **根因**：Windows Update 已安装补丁，需要重启后生效，**严重程度**：Info

4. 检查 CBS Sessions 中 PendingOperations > 0：
   - 无输出 → 正常
   - PendingOperations > 0 → **根因**：组件服务会话中存在未完成的操作，需重启继续执行，**严重程度**：Info

> 如果确认是 Windows Update 导致的待重启，参见 → [system-update.md](references/system-update.md)（Windows Update 诊断）

### Step 3: 启动耗时与运行时长分析

**数据采集**：

> 采集目标：系统最后一次启动时间、运行时长、启动性能诊断事件（Event ID 100：启动耗时与退化）

```powershell
# System boot time and uptime
Get-CimInstance Win32_OperatingSystem | Select-Object LastBootUpTime, @{Name='UptimeDays';Expression={[math]::Round(((Get-Date) - $_.LastBootUpTime).TotalDays, 2)}} | Format-Table -AutoSize

# Boot performance diagnostic events (Event ID 100: boot duration)
Get-WinEvent -LogName 'Microsoft-Windows-Diagnostics-Performance/Operational' -MaxEvents 20 -ErrorAction SilentlyContinue | Where-Object {
    $_.Id -eq 100
} | Select-Object TimeCreated, Id, Message | Format-Table -AutoSize
```

**分析思路**：

1. 检查系统运行时长（UptimeDays）：
   - 不足 1 天且无明显运维操作 → 可能发生了意外重启 → 进入 Step 4 结合事件日志确认原因
   - 运行天数正常 → 排除频繁意外重启问题

2. 检查 Event ID 100（启动耗时诊断事件）：
   - 从 Message 中提取启动耗时（单位 ms）：
     - < 30000 → 正常
     - 30000~60000 → 偏慢，需关注
     - 60000~120000 → **根因**：系统启动耗时偏长，可能存在驱动加载延迟或服务启动瓶颈，**严重程度**：Warning
     - > 120000 → **根因**：系统启动严重缓慢，严重影响服务器可用性，**严重程度**：Critical
   - 从 Message 中提取启动退化量（单位 ms）：
     - ≤ 0 → 启动速度正常或优于基准
     - > 0 且持续增大 → 启动性能持续退化，需定位退化来源
   - 从 Message 中提取启动是否因系统事件触发（如 Windows Update 重启）→ 若是补丁安装后首次启动，启动耗时会包含补丁配置时间，属于正常现象

3. 检查 Event ID 100 中是否标记为关键性能退化：
   - 标记为 IsDegradation = true → 本次启动比系统记录的基准启动时间明显更慢
   - 同时标记为 IsCritical = true → 启动退化已达到严重级别
   - 连续多次出现 → 问题持续存在，非偶发

> 如果启动耗时偏长，优先进入 → Step 5 (启动阶段分解) 定位瓶颈阶段
> 如果发现驱动加载延迟，参见 → [device-driver.md](references/device-driver.md)（设备驱动检查）

### Step 4: 关机/重启事件分析

**数据采集**：

> 采集目标：系统正常关机事件（Event ID 6006）、用户/进程发起关机事件（Event ID 1074）、意外关机事件（Event ID 6008）、内核电源异常事件（Event ID 41）

```powershell
# Recent 20 shutdown/restart related events
Get-WinEvent -FilterHashtable @{LogName='System'; Id=6006,1074,6008,41} -MaxEvents 20 -ErrorAction SilentlyContinue | Select-Object TimeCreated, Id, LevelDisplayName, Message | Format-Table -AutoSize
```

**分析思路**：

1. 检查 Event ID 6006（Event Log 服务已停止）：
   - 记录 "Event log service was stopped" 的时间 = 系统正常关机的最后记录
   - 无 6006 事件但有 41 事件 → 系统未经过正常关机流程，直接断电或崩溃

2. 检查 Event ID 1074（用户/进程发起关机）：
   - 从 Message 中提取发起关机的进程名和 Reason Code
   - 正常运维重启（如 `C:\Windows\system32\wlms\wlms.exe` expected restart after hotpatch）→ Info
   - 未知进程发起 → **根因**：非预期的用户或进程触发了系统重启，**严重程度**：Warning

3. 检查 Event ID 6008（上次关机意外）：
   - 记录每次意外关机的时间和发生频率
   - 如果频繁出现且与 Event ID 41 配对 → **根因**：系统频繁意外关机，可能由蓝屏、掉电、硬件故障导致，**严重程度**：Critical

4. 检查 Event ID 41（内核电源事件）：
   - BugcheckCode = 0：非蓝屏导致（掉电、强制关机或硬件故障）
   - BugcheckCode ≠ 0：蓝屏导致的重启 → 参见 [system-crash.md](references/system-crash.md)
   - PowerButtonTimestamp = 0：系统未记录电源按钮按下，排除人为操作

> 如果 Event ID 41 的 BugcheckCode ≠ 0，参见 → [system-crash.md](references/system-crash.md)（BugCheck 蓝屏事件）
> 如果频繁意外关机且伴随存储驱动事件异常，参见 → [storage-hardware.md](references/storage-hardware.md)（存储驱动检查）

### Step 5: 启动阶段分解

**数据采集**：

> 采集目标：启动各阶段耗时分解（Event ID 101），包括内核初始化、驱动加载、设备初始化、服务启动等阶段的具体耗时

```powershell
# Boot phase breakdown (Event ID 101: contains per-phase timing)
Get-WinEvent -LogName 'Microsoft-Windows-Diagnostics-Performance/Operational' -MaxEvents 5 -ErrorAction SilentlyContinue | Where-Object {
    $_.Id -eq 101
} | Select-Object TimeCreated, Id, Message | Format-Table -AutoSize
```

**分析思路**：

1. 检查 Event ID 101（启动阶段分解事件）是否可用：
   - 无输出 → 该日志通道未启用或系统尚未记录启动阶段事件（首次启动后需等待下次重启才会生成）
   - 有输出 → 从 Message 中提取各阶段耗时

2. 从 Message 中提取关键阶段耗时并逐一分析（单位均为 ms）：

   | 阶段 | 说明 | 正常范围 | 异常阈值 |
   |------|------|---------|--------|
   | 内核初始化（KernelInit） | 从内核加载到 Smss.exe 启动 | < 5000 | > 10000 |
   | 驱动初始化（DriversInit） | 所有启动类型驱动的加载耗时 | < 10000 | > 30000 |
   | 设备初始化（DevicesInit） | 即插即用设备枚举和初始化 | < 5000 | > 15000 |
   | 预读取（PrefetchInit） | 启动文件预读取完成耗时 | < 10000 | > 30000 |
   | 会话初始化（SessionInit） | 用户会话管理器初始化 | < 15000 | > 60000 |
   | 登录后初始化（PostBoot） | 登录后后台服务启动 | < 30000 | > 60000 |

3. 分阶段诊断判断：
   - 驱动初始化阶段耗时异常高 → **根因**：存在启动时加载缓慢的驱动，可能因驱动初始化超时或驱动冲突导致，**严重程度**：Warning
     - 结合驱动加载延迟定位具体驱动 → 参见 [device-driver.md](references/device-driver.md)
   - 设备初始化阶段耗时异常高 → **根因**：即插即用设备枚举耗时过长，可能存在不存在或故障的设备导致长时间等待，**严重程度**：Warning
     - 检查是否存在未连接的 SAN 存储或 iSCSI 目标导致设备枚举超时 → 参见 [storage-hardware.md](references/storage-hardware.md)
   - 预读取阶段耗时异常高 → 可能与磁盘 I/O 性能相关，参见 → [storage-disk.md](references/storage-disk.md)
   - 会话初始化或登录后初始化阶段耗时异常高 → **根因**：服务启动阶段耗时过长，某服务或启动程序阻塞了启动流程，**严重程度**：Warning
     - 结合服务启动超时诊断 → 参见 [service-list.md](references/service-list.md)
   - 多阶段同时异常 → 可能存在系统级性能瓶颈（如磁盘 I/O 不足、CPU 资源竞争）

4. 检查启动阶段事件是否标记为关键性能退化：
   - 标记为 IsDegradation = true → 本次启动各阶段整体慢于基准
   - 连续多次记录退化 → 启动性能持续下降，非偶发

> 如果 Event ID 101 不可用，可以启动时进入安全模式观察是否仍然缓慢，以排除第三方驱动和服务的影响

### Step 6: 关机耗时分析

**数据采集**：

> 采集目标：关机性能诊断事件（Event ID 200），包括关机总耗时和退化标记

```powershell
# Shutdown performance events (Event ID 200: shutdown duration)
Get-WinEvent -LogName 'Microsoft-Windows-Diagnostics-Performance/Operational' -MaxEvents 5 -ErrorAction SilentlyContinue | Where-Object {
    $_.Id -eq 200
} | Select-Object TimeCreated, Id, Message | Format-Table -AutoSize
```

**分析思路**：

1. 检查 Event ID 200（关机性能事件）是否可用：
   - 无输出 → 可能原因：系统未正常关机（意外断电）、日志通道未启用、或关机前日志服务已停止
   - 有输出 → 从 Message 中提取关机耗时

2. 从 Message 中提取关机耗时（单位 ms）：
   - < 5000 → 正常，关机流程顺畅
   - 5000~30000 → 偏慢，可能存在个别服务或应用延迟响应关机信号
   - 30000~120000 → **根因**：关机耗时明显偏长，存在阻碍关机流程的服务或进程，**严重程度**：Warning
   - > 120000 → **根因**：关机严重缓慢，系统可能被强制终止服务或进程，**严重程度**：Critical

3. 检查关机退化标记：
   - 标记为 IsDegradation = true → 本次关机比系统记录的基准关机时间明显更慢
   - 同时标记为 IsCritical = true → 关机退化已达到严重级别
   - 连续多次退化 → **根因**：关机性能持续退化，需定位阻碍关机的服务或应用，**严重程度**：Warning

4. 关机耗时与非正常关机事件的关联分析：
   - 关机耗时异常且 Event ID 6008 频繁出现 → 系统因关机超时被强制断电，可能已造成文件系统损坏
   - 关机耗时异常但后续无 Event ID 6008 → 关机流程最终完成，但耗时过长影响运维效率

5. 关机缓慢的常见原因排查方向：
   - 关机超时配置不当（WaitToKillServiceTimeout / WaitToKillAppTimeout 过大）→ 参见 Step 1
   - 存在挂起的 Windows Update 操作 → 参见 Step 2
   - 第三方服务拒绝响应关机信号 → 参见 [service-list.md](references/service-list.md)
   - 组策略关机脚本执行超时 → 参见 [system-gpo.md](references/system-gpo.md)
   - 用户配置文件卸载缓慢（User Profile Service）→ 参见 [identity-user-profiles.md](references/identity-user-profiles.md)

> 如果 Event ID 200 不可用但用户明确反馈关机缓慢，优先排查 → Step 1 (关机超时配置)
> 如果关机耗时异常且伴随磁盘相关事件，参见 → [storage-disk.md](references/storage-disk.md)

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Event ID 41 的 BugcheckCode ≠ 0（蓝屏导致的重启） | → [system-crash.md](references/system-crash.md) |
| 条件跳转 | 频繁意外关机且伴随存储驱动事件异常 | → [storage-hardware.md](references/storage-hardware.md) |
| 条件跳转 | Windows Update 导致的待重启操作 | → [system-update.md](references/system-update.md) |
| 条件跳转 | GPO 关机脚本阻塞关机流程 | → [system-gpo.md](references/system-gpo.md) |
| 条件跳转 | 启动耗时中驱动加载延迟 | → [device-driver.md](references/device-driver.md) |
| 条件跳转 | 关机耗时异常且持续退化 | → [service-list.md](references/service-list.md) |
| 条件跳转 | 关机耗时异常且涉及用户配置文件卸载 | → [identity-user-profiles.md](references/identity-user-profiles.md) |
| 链式后继 | 本文件未确认根因 | → [desktop-shell.md](references/desktop-shell.md) |

## 修复建议

### 根因：自动终止任务策略未启用（无人值守关机卡住）

**修复操作**：

```powershell
# Enable auto-termination of unresponsive tasks (current user HKCU)
Set-ItemProperty -Path "HKCU:\Control Panel\Desktop" -Name AutoEndTasks -Value "1" -Type String
```

**验证方法**：

```powershell
Get-ItemProperty -Path "HKCU:\Control Panel\Desktop" -Name AutoEndTasks | Select-Object -ExpandProperty AutoEndTasks
```

预期结果：返回值为 1

**风险说明**：启用后关机时不再弹出"等待程序关闭"提示，未响应的应用程序会被自动强制终止，可能导致未保存数据丢失。

### 根因：关机等待服务超时过长或过短

**修复操作**：

```powershell
# Restore WaitToKillServiceTimeout to default value 5000 (5 seconds, in ms)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control" -Name WaitToKillServiceTimeout -Value 5000 -Type DWord
```

**验证方法**：

```powershell
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control" -Name WaitToKillServiceTimeout | Select-Object -ExpandProperty WaitToKillServiceTimeout
```

预期结果：返回值为 5000

**风险说明**：缩短等待时间可能导致部分服务在关闭前未完成清理操作（如写入日志、释放资源）；如有依赖长时间清理的服务，可适当调大至 20000~60000。修改此注册表项需 Administrator 权限。

### 根因：关机等待应用超时过长

**修复操作**：

```powershell
# Restore WaitToKillAppTimeout to default value 20000 (20 seconds)
Set-ItemProperty -Path "HKCU:\Control Panel\Desktop" -Name WaitToKillAppTimeout -Value "20000" -Type String
```

**验证方法**：

```powershell
Get-ItemProperty -Path "HKCU:\Control Panel\Desktop" -Name WaitToKillAppTimeout | Select-Object -ExpandProperty WaitToKillAppTimeout
```

预期结果：返回值为 20000

**风险说明**：仅对当前用户生效。缩短等待时间后，未保存文档可能来不及提示用户保存。

### 根因：挂起应用判定超时过长

**修复操作**：

```powershell
# Restore HungAppTimeout to default value 5000 (5 seconds)
Set-ItemProperty -Path "HKCU:\Control Panel\Desktop" -Name HungAppTimeout -Value "5000" -Type String
```

**验证方法**：

```powershell
Get-ItemProperty -Path "HKCU:\Control Panel\Desktop" -Name HungAppTimeout | Select-Object -ExpandProperty HungAppTimeout
```

预期结果：返回值为 5000

**风险说明**：缩短后系统更快判定应用"未响应"并进入终止流程，正常但暂时繁忙的应用可能被误判。
