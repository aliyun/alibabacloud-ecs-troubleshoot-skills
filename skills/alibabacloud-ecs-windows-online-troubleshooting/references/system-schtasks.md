# Task Scheduler 诊断

## 目录

- [Task Scheduler 诊断](#task-scheduler-诊断)
  - [目录](#目录)
  - [功能说明](#功能说明)
  - [步骤选取指引](#步骤选取指引)
  - [诊断步骤](#诊断步骤)
    - [Step 1: Task Scheduler 服务状态](#step-1-task-scheduler-服务状态)
    - [Step 2: 任务历史日志检查](#step-2-任务历史日志检查)
    - [Step 3: 任务状态与上次运行结果](#step-3-任务状态与上次运行结果)
    - [Step 4: 任务凭据检查](#step-4-任务凭据检查)
    - [Step 5: 触发器配置验证](#step-5-触发器配置验证)
    - [Step 6: 依赖程序路径检查](#step-6-依赖程序路径检查)
    - [Step 7: 电源与条件设置检查](#step-7-电源与条件设置检查)
    - [Step 8: 任务损坏与缓存修复](#step-8-任务损坏与缓存修复)
  - [交叉引用](#交叉引用)
  - [修复建议](#修复建议)
    - [根因: Task Scheduler 服务未运行](#根因-task-scheduler-服务未运行)
    - [根因: 任务启动失败](#根因-任务启动失败)
    - [根因: 任务凭据无效](#根因-任务凭据无效)
    - [根因: 触发器配置错误](#根因-触发器配置错误)
    - [根因: 任务依赖程序缺失](#根因-任务依赖程序缺失)
    - [根因: 任务文件损坏](#根因-任务文件损坏)
    - [根因: SPP 任务权限缺失](#根因-spp-任务权限缺失)

## 功能说明

诊断 Windows 计划任务服务（Task Scheduler）及其相关任务的执行问题。覆盖 Task Scheduler 服务状态、任务启动失败、任务未按预期运行、凭据无效、触发器配置错误、依赖文件缺失、任务损坏与缓存异常等场景。

**输入**：用户问题描述（必选）、任务名称/错误代码/事件ID（可选，用于缩小排查范围）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 所有计划任务都不执行 | Step 1 (Task Scheduler 服务状态) → Step 8 (任务损坏与缓存修复) |
| 特定任务启动失败（Event ID 101） | Step 2 (任务历史日志) → Step 4 (任务凭据检查) → Step 6 (依赖程序路径) |
| 任务未在预期时间运行 | Step 3 (任务状态与上次运行结果) → Step 5 (触发器配置验证) → Step 7 (电源与条件设置) |
| 提示密码错误或账户无效 | Step 4 (任务凭据检查) |
| 任务触发条件不满足 | Step 5 (触发器配置验证) |
| 任务执行程序找不到 | Step 6 (依赖程序路径) |
| SPP 软件保护任务失败 | Step 2 (任务历史日志) → Step 4 (任务凭据检查) → [system-activation.md](references/system-activation.md) |
| Task Scheduler 服务无法启动 | Step 1 (Task Scheduler 服务状态) → Step 8 (任务损坏与缓存修复) |

## 诊断步骤

### Step 1: Task Scheduler 服务状态

**数据采集**：

> 采集目标：Task Scheduler 服务（Schedule）的运行状态、启动类型、依赖服务状态

```powershell
# Check Task Scheduler service status
Get-Service -Name 'Schedule' | Select-Object Name, Status, StartType, DependentServices | Format-Table -AutoSize

# Check RPC service status required by Task Scheduler
Get-Service -Name 'RpcSs' | Select-Object Name, Status, StartType | Format-Table -AutoSize

# Check Remote Procedure Call (RPC) Locator service required by Task Scheduler
Get-Service -Name 'RpcLocator' -ErrorAction SilentlyContinue | Select-Object Name, Status, StartType | Format-Table -AutoSize
```

**分析思路**：

1. 检查服务运行状态：
   - 正常：Status = Running
   - 异常：Status = Stopped → **根因**：Task Scheduler 服务未运行,导致所有计划任务无法执行,**严重程度**：Critical

2. 检查服务启动类型：
   - 正常：StartType = Automatic
   - 异常：StartType = Disabled → **根因**：Task Scheduler 服务被禁用,**严重程度**：Critical

3. 检查依赖服务（RPC）：
   - 如果 RPC 服务未运行,Task Scheduler 也无法启动 → **根因**：RPC 依赖服务未运行,导致 Task Scheduler 无法启动,**严重程度**：Critical

> 如果服务启动失败或无法启动,进入 Step 8 排查任务损坏问题。

### Step 2: 任务历史日志检查

**数据采集**：

> 采集目标：Task Scheduler 操作日志中的最近任务执行失败事件（Event ID 101/102/201/322），以及任务历史记录是否已启用

```powershell
# Check if task history is enabled (via log enablement status)
$operationalLog = Get-WinEvent -LogName 'Microsoft-Windows-TaskScheduler/Operational' -MaxEvents 1 -ErrorAction SilentlyContinue
if ($operationalLog) {
    Write-Output "Task Scheduler Operational log is enabled, last event: $($operationalLog.TimeCreated)"
} else {
    Write-Output "WARNING: Task Scheduler Operational log may be disabled - no events found"
}

# Query task start failure events (Event ID 101 = task start failure)
Get-WinEvent -LogName 'Microsoft-Windows-TaskScheduler/Operational' -MaxEvents 100 -ErrorAction SilentlyContinue | Where-Object {
    $_.Id -in @(101, 102, 201, 322)
} | Select-Object TimeCreated, Id, Message | Format-Table -AutoSize
```

**分析思路**：

1. 检查 Event ID 101（任务启动失败）：
   - 提取错误代码（如 0x80070005、0x80070569）
   - 0x80070005 → 权限不足 → **根因**：任务凭据无效或权限不足,**严重程度**：Critical
   - 0x80070569 → 登录失败 → **根因**：任务配置的账户密码已更改或账户被禁用,**严重程度**：Critical
   - 0x8004131F → 任务实例已在运行 → 可能是任务被配置为"不启动新实例"
   - 其他错误码 → 记录错误信息,继续后续步骤排查

2. 检查 Event ID 102（任务启动成功）：
   - 如果任务频繁启动失败但有成功记录,可能是间歇性问题（如网络依赖、资源竞争）

3. 检查 Event ID 201（操作完成）：
   - 查看任务退出码（非 0 表示程序执行异常）

4. 检查 Event ID 322（任务触发器失败）：
   - 触发器条件未满足 → **根因**：任务触发器配置错误或已过期,**严重程度**：Warning

> 如果发现 Event ID 101 且错误码为 0x80070569,参见 → [identity-account.md](references/identity-account.md)（账户状态检查）

### Step 3: 任务状态与上次运行结果

**数据采集**：

> 采集目标：指定任务的当前状态、上次运行时间、下次运行时间、最后一次运行结果

```powershell
# Replace <TaskName> with the actual task name, or use * to query all tasks
$taskName = "<TaskName>"

# Get basic task information
Get-ScheduledTask -TaskName $taskName -ErrorAction SilentlyContinue | Select-Object TaskName, TaskPath, State | Format-Table -AutoSize

# Get detailed task run information
Get-ScheduledTask -TaskName $taskName -ErrorAction SilentlyContinue | Get-ScheduledTaskInfo | Select-Object TaskName, TaskPath, LastRunTime, NextRunTime, LastTaskResult | Format-Table -AutoSize
```

**分析思路**：

1. 检查任务状态（State）：
   - Ready：正常,等待触发
   - Running：正在执行
   - Disabled：被禁用 → **根因**：任务被手动禁用,**严重程度**：Warning
   - Queued：已排队等待执行

2. 检查 LastTaskResult（最后一次运行结果）：
   - 0：成功
   - 1：通用错误
   - 0x80070005：访问被拒绝 → **根因**：权限不足,**严重程度**：Critical
   - 0x80070569：登录失败 → **根因**：凭据无效,**严重程度**：Critical
   - 0x41301：任务正在运行
   - 0x41303：任务未运行 → **根因**：任务未在预期时间触发,**严重程度**：Warning
   - 0x8004131F：任务实例已在运行
   - 0x80041002：任务找不到（可能被删除或损坏）

3. 检查 NextRunTime：
   - 如果有明确的下次运行时间,说明触发器配置正常
   - 如果为空或无值,说明触发器可能已失效 → 进入 Step 5

### Step 4: 任务凭据检查

**数据采集**：

> 采集目标：任务配置的运行账户、登录类型、运行级别

```powershell
$taskName = "<TaskName>"
$task = Get-ScheduledTask -TaskName $taskName -ErrorAction SilentlyContinue
if ($task) {
    $principal = $task.Principal
    [PSCustomObject]@{
        TaskName   = $task.TaskName
        UserId     = $principal.UserId
        LogonType  = $principal.LogonType
        RunLevel   = $principal.RunLevel
        GroupId    = $principal.GroupId
    } | Format-List
} else {
    Write-Output "Task not found: $taskName"
}
```

**分析思路**：

1. 检查 UserId（运行账户）：
   - 格式应为：`DOMAIN\User` 或 `NT AUTHORITY\SYSTEM` 或 `NT AUTHORITY\LOCAL SERVICE` 或 `NT AUTHORITY\NETWORK SERVICE`
   - 如果账户不存在或已被禁用 → **根因**：任务配置的账户无效,**严重程度**：Critical

2. 检查 LogonType（登录类型）：
   - Password：需要密码（交互式或服务账户），密码更改后任务会失败
   - S4U：无需密码（Service for User），仅本地资源访问，不支持网络身份验证
   - Interactive：需要交互式登录
   - Group：使用组标识运行
   - 如果配置为 Password 但密码已更改 → **根因**：任务凭据过期,**严重程度**：Critical

3. 检查 RunLevel（运行级别）：
   - Highest：管理员权限
   - LeastPrivilege：标准用户权限
   - 如果任务需要管理员权限但配置为 LeastPrivilege,可能导致执行失败

4. 验证账户状态：
   > 如果任务配置了特定用户账户（非 SYSTEM/LOCAL SERVICE/NETWORK SERVICE）,检查该账户是否被锁定或禁用：
   ```powershell
   $userName = "<UserName>"
   Get-LocalUser -Name $userName -ErrorAction SilentlyContinue | Select-Object Name, Enabled, AccountExpires, PasswordExpires | Format-Table -AutoSize
   ```

> 如果账户被锁定或禁用,参见 → [identity-account.md](references/identity-account.md)（账户解锁与重置）

### Step 5: 触发器配置验证

**数据采集**：

> 采集目标：任务的触发器类型、触发时间、启用状态、过期时间、重复执行配置

```powershell
$taskName = "<TaskName>"
$task = Get-ScheduledTask -TaskName $taskName -ErrorAction SilentlyContinue
if ($task) {
    $task.Triggers | ForEach-Object {
        [PSCustomObject]@{
            TriggerType       = $_.CimClass.CimClassName
            Enabled           = $_.Enabled
            StartBoundary     = $_.StartBoundary
            EndBoundary       = $_.EndBoundary
            ExecutionTimeLimit = $_.ExecutionTimeLimit
            RepetitionInterval = $_.Repetition.Interval
            RepetitionDuration = $_.Repetition.Duration
        }
    } | Format-Table -AutoSize
} else {
    Write-Output "Task not found: $taskName"
}
```

**分析思路**：

1. 检查触发器类型：
   - MSFT_TaskTimeTrigger：一次性时间触发
   - MSFT_TaskDailyTrigger：每日触发
   - MSFT_TaskWeeklyTrigger：每周触发
   - MSFT_TaskLogonTrigger：登录时触发
   - MSFT_TaskBootTrigger：启动时触发
   - MSFT_TaskEventTrigger：事件触发

2. 检查 Enabled 状态：
   - True：触发器已启用
   - False：触发器已禁用 → **根因**：触发器被禁用,任务不会执行,**严重程度**：Warning

3. 检查 StartBoundary（开始时间）：
   - 如果 StartBoundary 是过去的时间且为 TimeTrigger（一次性）,任务不会再触发 → **根因**：一次性触发器已过期,**严重程度**：Warning
   - 如果 StartBoundary 是未来时间,任务还未到达执行时间

4. 检查 EndBoundary（结束时间）：
   - 如果 EndBoundary 已过,触发器已过期 → **根因**：触发器已超过结束时间,**严重程度**：Warning

5. 检查 Repetition（重复执行）：
   - 如果配置了重复执行,检查 Interval 和 Duration 是否合理

### Step 6: 依赖程序路径检查

**数据采集**：

> 采集目标：任务配置的执行程序路径、参数、工作目录，以及程序文件是否存在

```powershell
$taskName = "<TaskName>"
$task = Get-ScheduledTask -TaskName $taskName -ErrorAction SilentlyContinue
if ($task) {
    $task.Actions | ForEach-Object {
        $executePath = $_.Execute
        [PSCustomObject]@{
            ActionPath      = $executePath
            Arguments       = $_.Arguments
            WorkingDirectory = $_.WorkingDirectory
            PathExists      = Test-Path -Path $executePath -ErrorAction SilentlyContinue
        }
    } | Format-Table -AutoSize
} else {
    Write-Output "Task not found: $taskName"
}
```

**分析思路**：

1. 检查程序路径是否存在：
   - PathExists = True：程序文件存在
   - PathExists = False → **根因**：任务依赖的程序文件不存在或路径错误,**严重程度**：Error

2. 检查程序文件权限：
   > 如果文件存在但任务执行失败,检查任务运行账户是否有执行权限：
   ```powershell
   $exePath = "<ProgramPath>"
   $acl = Get-Acl -Path $exePath
   $acl.Access | Where-Object { $_.IdentityReference -match "<UserId>" } | Select-Object IdentityReference, FileSystemRights, AccessControlType | Format-Table -AutoSize
   ```

3. 检查 WorkingDirectory（工作目录）：
   - 如果配置了工作目录但该目录不存在,某些程序会执行失败

4. 检查 Arguments（参数）：
   - 参数中包含的文件路径是否有效
   - 参数语法是否正确（如 PowerShell 脚本需要 `-ExecutionPolicy Bypass -File` 前缀）

> 如果程序文件缺失,需要从备份恢复或重新安装相关软件

### Step 7: 电源与条件设置检查

**数据采集**：

> 采集目标：任务的电源条件、空闲条件、网络条件等条件设置

```powershell
$taskName = "<TaskName>"
$task = Get-ScheduledTask -TaskName $taskName -ErrorAction SilentlyContinue
if ($task) {
    $task.Settings | Select-Object AllowHardTerminate, DeleteExpiredTaskAfter, DisallowStartIfOnBatteries, ExecutionTimeLimit, Hidden, MultipleInstances, Priority, RestartCount, RestartInterval, StartWhenAvailable, StopIfGoingOnBatteries, WakeToRun | Format-List
} else {
    Write-Output "Task not found: $taskName"
}
```

**分析思路**：

1. 检查 DisallowStartIfOnBatteries（使用电池时不启动）：
   - True：笔记本电脑使用电池时任务不会触发 → 可能导致任务未运行

2. 检查 StopIfGoingOnBatteries（切换到电池时停止）：
   - True：任务运行中切换到电池会终止任务

3. 检查 WakeToRun（唤醒计算机运行）：
   - False：计算机睡眠时任务不会唤醒系统执行

4. 检查 StartWhenAvailable（错过时立即运行）：
   - True：错过触发时间后会尽快补执行
   - False：错过触发时间后跳过该次执行 → 可能导致任务"未运行"

5. 检查 ExecutionTimeLimit（执行时间限制）：
   - 如果任务执行时间超过限制会被强制终止 → 可能导致长运行任务被中断

6. 检查 MultipleInstances（多实例策略）：
   - IgnoreNew：如果任务已在运行,新触发将被忽略
   - Parallel：允许并行运行
   - Queue：排队等待

### Step 8: 任务损坏与缓存修复

**数据采集**：

> 采集目标：Task Scheduler 任务文件完整性、任务缓存状态、SPP 任务配置（Network Service 权限）

```powershell
# Check if task XML files are corrupted
$taskPath = "$env:SystemRoot\System32\Tasks"
Get-ChildItem -Path $taskPath -Recurse -File | ForEach-Object {
    try {
        [xml]$xml = Get-Content $_.FullName -ErrorAction Stop
    } catch {
        Write-Output "CORRUPTED: $($_.FullName) - $($_.Exception.Message)"
    }
} | Where-Object { $_ -match "CORRUPTED" }

# Check Network Service permissions for SPP tasks (common cause of software protection task failures)
$sppTaskPath = "$env:SystemRoot\System32\Tasks\Microsoft\Windows\SoftwareProtectionPlatform"
if (Test-Path $sppTaskPath) {
    Get-ChildItem -Path $sppTaskPath -File | ForEach-Object {
        $acl = Get-Acl -Path $_.FullName -ErrorAction SilentlyContinue
        $networkService = $acl.Access | Where-Object { $_.IdentityReference -match "NETWORK SERVICE" }
        if (-not $networkService) {
            Write-Output "WARNING: NETWORK SERVICE does not have permissions on $($_.FullName)"
        } else {
            Write-Output "OK: NETWORK SERVICE permissions on $($_.FullName): $($networkService.FileSystemRights)"
        }
    }
} else {
    Write-Output "SPP task directory not found: $sppTaskPath"
}

# Check service crash/start failure events in Task Scheduler related event logs
Get-WinEvent -FilterHashtable @{LogName='System'; Id=7034,7023,7024} -MaxEvents 20 -ErrorAction SilentlyContinue | Where-Object {
    $_.ProviderName -eq 'Service Control Manager' -and $_.Message -match 'Schedule|Task Scheduler'
} | Select-Object TimeCreated, Id, Message | Format-Table -AutoSize
```

**分析思路**：

1. 检查任务 XML 文件损坏：
   - 如果发现无法解析的 XML 文件 → **根因**：任务定义文件损坏,可能导致 Task Scheduler 服务无法启动或任务无法加载,**严重程度**：Critical
   - 损坏的任务文件会导致 "Task Scheduler service is not available" 错误

2. 检查 SPP 任务 Network Service 权限：
   - `Microsoft\Windows\SoftwareProtectionPlatform\SvcRestartTask` 需要NETWORK SERVICE 账户具有读取和执行权限
   - 如果缺少权限 → **根因**：SPP 任务的 Network Service 权限缺失,导致软件保护任务无法重新调度,**严重程度**：Critical

3. 检查 Task Scheduler 服务异常事件：
   - Event ID 7034：服务意外崩溃
   - Event ID 7023：服务异常退出
   - Event ID 7024：服务特定错误
   - 如果发现服务崩溃事件 → **根因**：Task Scheduler 服务因损坏的任务文件或系统文件缺失而崩溃,**严重程度**：Critical

> 如果发现 SPP 任务失败且与激活相关,参见 → [system-activation.md](references/system-activation.md)

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Step 2/4 发现任务账户被锁定或禁用 | → [identity-account.md](references/identity-account.md) |
| 条件跳转 | Step 2 发现 SPP 软件保护任务失败 | → [system-activation.md](references/system-activation.md) |
| 条件跳转 | Step 6 发现程序文件权限不足 | → [identity-permission.md](references/identity-permission.md) |
| 条件跳转 | Step 4 发现账户密码过期需重置 | → [identity-account.md](references/identity-account.md) |
| 参数化引用 | Step 2 发现网络相关错误码,怀疑防火墙阻止 | → [networking-firewall.md](references/networking-firewall.md)（检查出站规则是否阻止任务出站连接） |
| 链式后继 | 本文件所有步骤执行完毕,未确认根因 | → [system-management.md](references/system-management.md) |

## 修复建议

### 根因: Task Scheduler 服务未运行

**修复操作**：

```powershell
# 1. Check current service status
Get-Service -Name 'Schedule' | Format-Table -AutoSize

# 2. If disabled, change the startup type first
Set-Service -Name 'Schedule' -StartupType Automatic

# 3. Start the service
Start-Service -Name 'Schedule'

# 4. Verify service status
Get-Service -Name 'Schedule' | Select-Object Name, Status, StartType | Format-Table -AutoSize
```

**验证方法**：

```powershell
Get-Service -Name 'Schedule'
```

预期结果：Status = Running, StartType = Automatic

**风险说明**：启动 Task Scheduler 服务会影响所有依赖该服务的计划任务,建议在业务低峰期操作。

---

### 根因: 任务启动失败

**修复操作**：

```powershell
$taskName = "<TaskName>"

# 1. View detailed task error information
Get-WinEvent -LogName 'Microsoft-Windows-TaskScheduler/Operational' -MaxEvents 100 -ErrorAction SilentlyContinue | Where-Object {
    $_.Id -eq 101 -and $_.Message -match $taskName
} | Select-Object TimeCreated, Message | Format-List

# 2. If credential issue (error code 0x80070569), update task password
# Note: the correct password is required
$task = Get-ScheduledTask -TaskName $taskName
$userId = $task.Principal.UserId
$password = Read-Host "Enter password for $userId" -AsSecureString
$plainPassword = [System.Runtime.InteropServices.Marshal]::PtrToStringAuto([System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($password))

# Update task credentials
Set-ScheduledTask -TaskName $taskName -User $userId -Password $plainPassword

# 3. If permission issue (error code 0x80070005), run the task with administrator privileges
Set-ScheduledTask -TaskName $taskName -Principal (New-ScheduledTaskPrincipal -UserId $userId -RunLevel Highest)
```

**验证方法**：

```powershell
# Manually run the task for testing
Start-ScheduledTask -TaskName "<TaskName>"

# Check if the task started successfully
Start-Sleep -Seconds 5
Get-ScheduledTask -TaskName "<TaskName>" | Select-Object State | Format-Table -AutoSize
```

预期结果：State = Ready（执行完毕后恢复为 Ready）

**风险说明**：更新任务凭据需要知道正确的密码,如果密码错误会导致任务继续失败。

---

### 根因: 任务凭据无效

**修复操作**：

```powershell
$taskName = "<TaskName>"
$userId = "<UserId>"

# 1. Check account status
Get-LocalUser -Name $userId.Split('\')[-1] -ErrorAction SilentlyContinue | Select-Object Name, Enabled, PasswordExpired | Format-Table -AutoSize

# 2. If account is disabled, enable it
Enable-LocalUser -Name $userId.Split('\')[-1]

# 3. If password expired, prompt user to change password
# Or reset password with administrator privileges
# $newPassword = Read-Host "Enter new password" -AsSecureString
# Set-LocalUser -Name $userId.Split('\')[-1] -Password $newPassword

# 4. Update task credentials
$password = Read-Host "Enter current password for $userId" -AsSecureString
$plainPassword = [System.Runtime.InteropServices.Marshal]::PtrToStringAuto([System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($password))
Set-ScheduledTask -TaskName $taskName -User $userId -Password $plainPassword
```

**验证方法**：

```powershell
# Manually run the task
Start-ScheduledTask -TaskName "<TaskName>"
Start-Sleep -Seconds 3
Get-ScheduledTask -TaskName "<TaskName>" | Get-ScheduledTaskInfo | Select-Object LastTaskResult | Format-Table -AutoSize
```

预期结果：LastTaskResult = 0,任务成功执行

**风险说明**：重置密码会影响该账户的所有登录会话和依赖该凭据的其他任务。

---

### 根因: 触发器配置错误

**修复操作**：

```powershell
$taskName = "<TaskName>"

# 1. View current triggers
Get-ScheduledTask -TaskName $taskName | Select-Object -ExpandProperty Triggers | Format-List

# 2. Disable old triggers
$task = Get-ScheduledTask -TaskName $taskName
$task.Triggers | ForEach-Object { $_.Enabled = $false }
Set-ScheduledTask -InputObject $task

# 3. Add new trigger (example: run daily at 9:00)
$trigger = New-ScheduledTaskTrigger -Daily -At "9:00"
$task = Get-ScheduledTask -TaskName $taskName
$task.Triggers.Add($trigger)
Set-ScheduledTask -InputObject $task
```

**验证方法**：

```powershell
# View updated triggers
Get-ScheduledTask -TaskName "<TaskName>" | Select-Object -ExpandProperty Triggers | Select-Object StartBoundary, Enabled | Format-Table -AutoSize

# View next run time
Get-ScheduledTask -TaskName "<TaskName>" | Get-ScheduledTaskInfo | Select-Object TaskName, NextRunTime | Format-Table -AutoSize
```

预期结果：NextRunTime 显示明确的未来时间,Enabled = True

**风险说明**：修改触发器会改变任务执行时间,需确认不影响业务流程。

---

### 根因: 任务依赖程序缺失

**修复操作**：

```powershell
$taskName = "<TaskName>"

# 1. View the program path configured for the task
$task = Get-ScheduledTask -TaskName $taskName
$action = $task.Actions[0]
Write-Output "Program: $($action.Execute)"
Write-Output "Arguments: $($action.Arguments)"

# 2. Check if the program exists
if (-not (Test-Path $action.Execute)) {
    Write-Output "ERROR: Program not found: $($action.Execute)"
    
    # 3. Try to search for the program in the system
    $exeName = [System.IO.Path]::GetFileName($action.Execute)
    $found = Get-ChildItem -Path "C:\" -Recurse -Filter $exeName -ErrorAction SilentlyContinue | Select-Object -First 5
    
    if ($found) {
        Write-Output "Found possible locations:"
        $found | ForEach-Object { Write-Output $_.FullName }
        
        # 4. Update task program path (execute after confirming the correct path)
        # $correctPath = "<CorrectPath>"
        # Set-ScheduledTask -TaskName $taskName -Action (New-ScheduledTaskAction -Execute $correctPath -Argument $action.Arguments)
    } else {
        Write-Output "Program not found in system. Need to reinstall or restore from backup."
    }
}
```

**验证方法**：

```powershell
# Verify the program path has been updated
$task = Get-ScheduledTask -TaskName "<TaskName>"
Test-Path $task.Actions[0].Execute

# Manually run the task
Start-ScheduledTask -TaskName "<TaskName>"
```

预期结果：Test-Path 返回 True,任务成功执行

**风险说明**：如果程序文件已被删除且无备份,需要重新安装相关软件。

---

### 根因: 任务文件损坏

**修复操作**：

```powershell
# 1. Locate corrupted task files
$taskPath = "$env:SystemRoot\System32\Tasks"
$corruptedFiles = @()
Get-ChildItem -Path $taskPath -Recurse -File | ForEach-Object {
    try {
        [xml]$xml = Get-Content $_.FullName -ErrorAction Stop
    } catch {
        $corruptedFiles += $_.FullName
        Write-Output "CORRUPTED: $($_.FullName)"
    }
}

# 2. Export registry information of corrupted tasks (for subsequent reconstruction)
# Tasks also exist in registry: HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Schedule\TaskCache\Tree
foreach ($file in $corruptedFiles) {
    $taskName = [System.IO.Path]::GetFileNameWithoutExtension($file)
    Write-Output "Corrupted task: $taskName"
    
    # 3. Delete corrupted task file (confirmation required)
    # Remove-Item -Path $file -Force
    # Write-Output "Deleted corrupted task file: $file"
}

# 4. Restart Task Scheduler service
# Restart-Service -Name 'Schedule' -Force
```

**验证方法**：

```powershell
# Verify service status
Get-Service -Name 'Schedule' | Select-Object Name, Status, StartType | Format-Table -AutoSize

# Verify task list is normal
Get-ScheduledTask | Select-Object TaskName, State | Format-Table -AutoSize
```

预期结果：Task Scheduler 服务正常运行,任务列表正常显示

**风险说明**：删除损坏的任务文件会导致该任务配置丢失。建议先导出任务 XML 备份,再删除损坏文件。删除后需重新创建任务。

---

### 根因: SPP 任务权限缺失

**修复操作**：

```powershell
# 1. Locate SPP task files
$sppTaskPath = "$env:SystemRoot\System32\Tasks\Microsoft\Windows\SoftwareProtectionPlatform"

# 2. Add ReadAndExecute permission for NETWORK SERVICE
if (Test-Path $sppTaskPath) {
    Get-ChildItem -Path $sppTaskPath -File | ForEach-Object {
        $acl = Get-Acl -Path $_.FullName
        $rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
            "NT AUTHORITY\NETWORK SERVICE",
            "ReadAndExecute",
            "Allow"
        )
        $acl.AddAccessRule($rule)
        Set-Acl -Path $_.FullName -AclObject $acl
        Write-Output "Added NETWORK SERVICE ReadAndExecute permission to $($_.FullName)"
    }
} else {
    Write-Output "SPP task directory not found: $sppTaskPath"
}

# 3. Confirm SPP tasks exist and are correctly configured
Get-ScheduledTask -TaskPath "\Microsoft\Windows\SoftwareProtectionPlatform\" -ErrorAction SilentlyContinue | Select-Object TaskName, State | Format-Table -AutoSize
```

**验证方法**：

```powershell
# Verify permissions have been added
$sppTaskPath = "$env:SystemRoot\System32\Tasks\Microsoft\Windows\SoftwareProtectionPlatform"
Get-ChildItem -Path $sppTaskPath -File | ForEach-Object {
    $acl = Get-Acl -Path $_.FullName
    $acl.Access | Where-Object { $_.IdentityReference -match "NETWORK SERVICE" } | Select-Object IdentityReference, FileSystemRights | Format-Table -AutoSize
}

# Verify sppsvc service is running normally
Get-Service -Name 'sppsvc' | Select-Object Name, Status, StartType | Format-Table -AutoSize
```

预期结果：NETWORK SERVICE 具有 ReadAndExecute 权限,sppsvc 服务正常运行

**风险说明**：修改 SPP 任务文件权限可能影响 Windows 激活流程,请确保仅添加必要权限。如果问题持续,参见 → [system-activation.md](references/system-activation.md)
