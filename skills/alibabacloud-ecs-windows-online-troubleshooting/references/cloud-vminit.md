# Cloud Vminit 诊断

## 功能说明

诊断 Windows vminit 服务安装状态、服务启用状态、初始化日志错误和 user-data 执行失败。vminit 是阿里云 ECS 实例初始化服务，负责网络配置、密码重置、磁盘扩展、user-data 脚本执行等。覆盖 4 个已知根因。

**输入**：用户问题描述（必选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 新建实例后网络未自动配置、元数据未获取 | Step 1 (vminit 服务状态) |
| 重启实例后网络配置丢失、密码重置不生效 | Step 1 (vminit 服务状态) |
| VirtIO 驱动安装失败、注册表访问错误、密码设置失败、磁盘扩展失败 | Step 2 (vminit 初始化日志) |
| 实例创建或重启后自定义脚本未执行或报错 | Step 1 (vminit 服务状态) → Step 2 (vminit 初始化日志) → Step 3 (user-data 接口) |

## 诊断步骤

### Step 1: vminit 服务状态检查

**数据采集**：

> 采集目标：获取 vminit 服务的安装状态、运行状态、启动类型以及可执行文件路径

```powershell
# 1) Service status from SCM
Get-Service -Name vminit -ErrorAction SilentlyContinue | Select-Object Name, Status, StartType | Format-Table -AutoSize

# 2) Service info via CIM（Win32_Service.PathName 同时包含 exe 路径与命令行参数，需解析后才能 Test-Path）
$svc = Get-CimInstance -ClassName Win32_Service -Filter "Name='vminit'" -ErrorAction SilentlyContinue
if ($svc) {
    $rawPath = $svc.PathName
    # Strip surrounding quotes 或 trailing arguments 以提取纯 .exe 路径
    if     ($rawPath -match '^"([^"]+)"')   { $exePath = $Matches[1] }
    elseif ($rawPath -match '^(\S+\.exe)')   { $exePath = $Matches[1] }
    else                                       { $exePath = $rawPath }
    [PSCustomObject]@{
        Name      = $svc.Name
        State     = $svc.State
        StartMode = $svc.StartMode
        PathName  = $rawPath
        ExePath   = $exePath
        Exists    = if ($exePath) { Test-Path -LiteralPath $exePath } else { $false }
    } | Format-List
} else {
    Write-Host 'vminit service not found via Win32_Service'
}
```

**分析思路**：

1. 检查 vminit 服务是否安装：
   - 正常：服务存在，启动类型为 Automatic
   - 异常：Get-Service 无输出且 Get-CimInstance 也查不到该服务 → **根因**：vminit 未安装，**严重程度**：Critical
   - 说明：vminit 未安装意味着实例初始化功能缺失，网络配置、密码重置、磁盘扩展等功能无法工作

2. 检查 vminit 服务是否可运行：
   - 正常：服务状态为 Running 或 Stopped（启动类型为 Automatic）
   - 异常：服务存在但启动类型为 Disabled → **根因**：vminit 服务不可运行，**严重程度**：Critical

3. 检查可执行文件是否存在（基于上面采集输出的 `ExePath` / `Exists` 字段）：
   - 正常：`Exists = True`，`ExePath` 指向真实存在的 vminit.exe
   - 异常：`Exists = False`（解析后的 `ExePath` 路径不存在）→ **根因**：vminit 服务不可运行（可执行文件丢失），**严重程度**：Critical
   - 说明：`Win32_Service.PathName` 原始值例如 `C:\ProgramData\aliyun\vminit\vminit.exe service`，附带命令行参数，**不得**直接 Test-Path，MUST 先剥去外层引号 / 尾部参数仅保留 `.exe` 路径后再验证

### Step 2: vminit 初始化日志检查

**数据采集**：

> 采集目标：读取最新 vminit 日志文件的完整内容，由分析阶段按规则识别错误

```powershell
# Read the full content of the latest vminit log file
$logPath = '$env:ProgramData\aliyun\vminit\log'
$latestLog = Get-ChildItem $logPath -File -ErrorAction SilentlyContinue | Sort-Object LastWriteTime -Descending | Select-Object -First 1
if ($latestLog) {
  Write-Host "LogFile: $($latestLog.FullName)"
  Get-Content $latestLog.FullName -ErrorAction SilentlyContinue
} else {
  Write-Host 'No vminit log file found'
}
```

**分析思路**：

1. 检查日志文件是否存在：
   - 正常：日志文件存在且有内容
   - 异常：输出 `No vminit log file found`，说明 vminit 可能从未运行成功

2. 在日志内容中识别已知错误关键词（参考阿里云官方文档）：
   - 文件和磁盘类：`extend_disk_error`（磁盘扩容失败）、`online_disk_error`（磁盘联机失败）、`import_disk_error`（导入动态磁盘失败）、`format_disk_error`（磁盘格式化失败）、`no_system_disk`（无系统盘或系统盘损坏）
   - 注册表类：`registry_access_error`（注册表访问错误）
   - 驱动类：`install_virtio_error`（安装 VirtIO 失败）、`disk_cannot_extend`（磁盘无法扩容）、`disk_data_loss`（磁盘数据丢失）、`netkvm_start_fail_onmoc`（旧网卡驱动）
   - 管控命令类：`unknown_operation`（未知管控命令）、`config_passwd_error`（修改密码失败）、`start_assist_error`（云助手启动失败）、`config_ntp_error`（配置 NTP 失败）、`meta_server_error`（Meta Server 连接失败）
   - 其他：`sysprep_not_ready`（Sysprep 未完成）、`unknown_win_version`（未知系统版本）
   - userdata 类：日志中含 `userdata`、`user_data` 相关错误记录（执行失败、超时、缓存异常）

   - 正常：无匹配的错误关键词
   - 异常（命中非 userdata 类关键词）→ **根因**：vminit 初始化日志错误，**严重程度**：Warning（按命中关键词细分场景）
   - 异常（命中 userdata/user_data 错误上下文）→ **根因**：user-data 脚本执行失败，**严重程度**：Warning
   - 异常（日志存在但无任何 userdata/user_data 记录）→ **根因**：vminit 未触发 userdata 执行，**严重程度**：Warning
   - 常见原因（userdata 类）：替换系统盘后缓存未清理、脚本语法错误、依赖环境缺失

### Step 3: user-data 接口检查

**数据采集**：

> 采集目标：检查 MetaServer user-data 接口是否返回内容

```powershell
# Check if user-data is configured via MetaServer
try {
  $resp = Invoke-WebRequest -Uri 'http://100.100.100.200/latest/user-data' -UseBasicParsing -TimeoutSec 5 -ErrorAction Stop
  Write-Host "HTTP Status: $($resp.StatusCode)"
  Write-Host "Content-Length: $($resp.Content.Length)"
  Write-Host '--- Content (first 500 chars) ---'
  $resp.Content.Substring(0, [Math]::Min(500, $resp.Content.Length))
} catch {
  Write-Host "Error: $($_.Exception.Message)"
}
```

**分析思路**：

1. 检查 MetaServer user-data 接口返回：
   - 正常：HTTP 200 且 Content-Length > 0，说明 user-data 已配置
   - 异常（404 或内容为空）：用户未配置 user-data，非 vminit 问题
   - 异常（超时或连接失败）：MetaServer 不可达 → 跳转 cloud-metaserver.md

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 链式后继 | 日志含 install_virtio_error | → [cloud-driver.md](references/cloud-driver.md)（排查 VirtIO 驱动安装问题） |
| 链式后继 | 日志含 meta_server_error | → [cloud-metaserver.md](references/cloud-metaserver.md)（排查元数据服务连通性） |
| 链式后继 | 日志含 extend_disk_error | → [storage-disk.md](references/storage-disk.md)（排查磁盘状态） |
| 条件跳转 | user-data 接口超时或连接失败 | → [cloud-metaserver.md](references/cloud-metaserver.md)（排查 MetaServer 可达性） |

## 修复建议

### 根因：vminit 未安装

**修复操作**：

```powershell
# Reinstall vminit
# 1. Download the latest vminit installer from Alibaba Cloud official channel
# 2. Run the installer as Administrator
# 3. Verify the service is registered and set to Automatic startup
```

**验证方法**：

```powershell
Get-Service -Name vminit -ErrorAction SilentlyContinue | Select-Object Name, Status, StartType | Format-Table -AutoSize
```

预期结果：vminit 服务存在，启动类型为 Automatic

**风险说明**：安装 vminit 需要管理员权限，安装后建议重启实例以触发初始化流程

---

### 根因：vminit 服务不可运行

**修复操作**：

```powershell
# Enable vminit service
Set-Service -Name vminit -StartupType Automatic

# Start vminit service
Start-Service -Name vminit
```

**验证方法**：

```powershell
Get-Service -Name vminit -ErrorAction SilentlyContinue | Select-Object Name, Status, StartType | Format-Table -AutoSize
```

预期结果：Status 为 Running，StartType 为 Automatic

**风险说明**：如 vminit 可执行文件丢失，启动服务会失败，需先重新安装

---

### 根因：user-data 脚本执行失败

**修复操作**：

```powershell
# 1. Clear stale userdata cache (if system disk was replaced)
$cachePath = "$env:ProgramData\aliyun\vminit\user-data"
if (Test-Path $cachePath) {
  Remove-Item -Path $cachePath -Recurse -Force
  Write-Host 'Cleared userdata cache'
}

# 2. Restart vminit to re-fetch and execute user-data
Restart-Service -Name vminit -ErrorAction SilentlyContinue
```

**验证方法**：

```powershell
$logPath = '$env:ProgramData\aliyun\vminit\log'
$latestLog = Get-ChildItem $logPath -File -ErrorAction SilentlyContinue | Sort-Object LastWriteTime -Descending | Select-Object -First 1
if ($latestLog) {
  Get-Content $latestLog.FullName -ErrorAction SilentlyContinue | Select-String -Pattern 'userdata|user.data|user_data' -SimpleMatch:$false
}
```

预期结果：日志中出现 userdata 执行成功记录，无错误信息

**风险说明**：清除缓存后重启 vminit 会重新执行 user-data 脚本，确认脚本内容幂等后再操作

---

### 根因：vminit 初始化日志错误

**说明**：

vminit 日志错误通常是其他根因的表象，需根据具体错误关键词关联对应的根因：

| 错误关键词 | 关联根因 | 修复方向 |
|---------|---------|--------|
| install_virtio_error | VirtIO 驱动安装失败 | 参见 → [cloud-driver.md](references/cloud-driver.md) |
| meta_server_error | 元数据服务不可达 | 参见 → [cloud-metaserver.md](references/cloud-metaserver.md) |
| config_passwd_error | 密码配置失败 | 检查密码策略和用户账户状态 |
| registry_access_error | 注册表访问被拒绝 | 检查权限和安全软件阻断 |
| no_system_disk | 系统磁盘未识别 | 参见 → [storage-disk.md](references/storage-disk.md) |
| extend_disk_error | 磁盘扩展失败 | 参见 → [storage-disk.md](references/storage-disk.md) |
| copy_files | 文件复制失败 | 检查磁盘空间和文件系统权限 |

**修复操作**：

```powershell
# View full log content to locate specific errors
$logPath = '$env:ProgramData\aliyun\vminit\log'
$latestLog = Get-ChildItem -Path $logPath -Filter '*.log' -ErrorAction SilentlyContinue | Sort-Object LastWriteTime -Descending | Select-Object -First 1
if ($latestLog) { Get-Content $latestLog.FullName -ErrorAction SilentlyContinue }

# If the issue is resolved, restart vminit service to trigger re-initialization
Restart-Service -Name vminit -ErrorAction SilentlyContinue
```

**验证方法**：

```powershell
$keywords = @('install_virtio_error','copy_files','registry_access_error','meta_server_error','config_passwd_error','no_system_disk','extend_disk_error')
$logPath = '$env:ProgramData\aliyun\vminit\log'
$latestLog = Get-ChildItem -Path $logPath -Filter '*.log' -ErrorAction SilentlyContinue | Sort-Object LastWriteTime -Descending | Select-Object -First 1
if ($latestLog) {
  Get-Content $latestLog.FullName -ErrorAction SilentlyContinue | Where-Object {
    $line = $_; ($keywords | Where-Object { $line -match $_ }).Count -gt 0
  }
}
```

预期结果：无输出（无错误关键词）

**风险说明**：重启 vminit 服务会重新执行初始化操作，可能修改网络配置和密码
