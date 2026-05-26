# System Activation 诊断

## 目录

- [System Activation 诊断](#system-activation-诊断)
  - [目录](#目录)
  - [功能说明](#功能说明)
  - [步骤选取指引](#步骤选取指引)
  - [诊断步骤](#诊断步骤)
    - [Step 1: sppsvc 服务状态检查](#step-1-sppsvc-服务状态检查)
    - [Step 2: KMS 激活状态检查](#step-2-kms-激活状态检查)
    - [Step 3: 产品密钥与 KMS 配置检查](#step-3-产品密钥与-kms-配置检查)
    - [Step 4: 激活事件日志检查](#step-4-激活事件日志检查)
  - [交叉引用](#交叉引用)
  - [修复建议](#修复建议)
    - [根因: sppsvc 服务损坏或被禁用](#根因-sppsvc-服务损坏或被禁用)
    - [根因: Windows 未通过 KMS 激活](#根因-windows-未通过-kms-激活)
    - [根因: 产品密钥不匹配 GVLK](#根因-产品密钥不匹配-gvlk)
    - [根因: 防火墙阻止 KMS 通信端口](#根因-防火墙阻止-kms-通信端口)
    - [根因: KMS 服务器不可达](#根因-kms-服务器不可达)
    - [根因: Tokens.dat 损坏（激活授权数据库损坏）](#根因-tokensdat-损坏激活授权数据库损坏)

## 功能说明

诊断 Windows 激活相关问题：Software Protection Service（sppsvc）服务状态异常、KMS 激活状态检查、产品密钥（GVLK）验证、KMS 端口防火墙阻止、KMS 服务器连通性、激活相关事件日志错误（含时间偏差检测）。覆盖 6 个已知问题项。

**输入**：用户问题描述（必选）、错误代码/事件ID（可选，用于缩小排查范围）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 桌面出现 Activate Windows 水印、功能缩减 | Step 2 (KMS 激活状态) → Step 3 (产品密钥与 KMS 配置) |
| slmgr /ato 报错、激活失败 | Step 1 (sppsvc 服务) → Step 2 (KMS 激活状态) → Step 3 (产品密钥与 KMS 配置) |
| 激活超时、报错 0x80072EE7 / 0x8007232B | Step 3 (产品密钥与 KMS 配置) → Step 4 (激活事件日志) |
| 报错 0xC004F074（KMS 不可用） | Step 3 (产品密钥与 KMS 配置) → Step 4 (激活事件日志) |
| 报错 0xC004F06C（时间戳无效） | Step 4 (激活事件日志) |
| 事件日志中出现激活错误（Event ID 8198 / 8193） | Step 4 (激活事件日志) → Step 1 (sppsvc 服务) → Step 2 (KMS 激活状态) |
| 产品密钥无效或不匹配 | Step 3 (产品密钥与 KMS 配置) |
| 报错 0xC004E002 / 0xC004E015（授权存储异常） | Step 2 (KMS 激活状态) → Step 1 (sppsvc 服务)，修复参见「Tokens.dat 重建」 |
| 反复激活失败、密钥和网络均正常 | Step 1 (sppsvc 服务) → Step 2 (KMS 激活状态)，修复参见「Tokens.dat 重建」 |

## 诊断步骤

### Step 1: sppsvc 服务状态检查

**数据采集**：

> 采集目标：获取 Software Protection Service（sppsvc）的服务状态、启动类型和运行信息，判断服务是否可正常工作

```powershell
# Query sppsvc service status
Get-CimInstance Win32_Service -Filter "Name='sppsvc'" | Select-Object Name, State, StartMode, Status, ExitCode | Format-List
```

**分析思路**：

1. 检查服务是否存在：
   - 正常：查询返回 sppsvc 服务记录
   - 异常：查询无返回结果，服务不存在 → **根因**：sppsvc 服务损坏或缺失，**严重程度**：Warning

2. 检查服务启动类型：
   - 正常：启动类型为 Auto 或 Manual（sppsvc 按需启动，非 Running 状态本身不算异常）
   - 异常：启动类型为 Disabled → **根因**：sppsvc 服务被禁用，**严重程度**：Warning

3. 检查服务配置完整性：
   - 正常：Status 为 OK，无异常退出码
   - 异常：Status 异常或存在非零退出码 → **根因**：sppsvc 服务配置损坏，**严重程度**：Warning

> 注意：sppsvc 为按需触发型服务（Trigger Start），在无激活操作时处于 Stopped 状态是正常的，只要未被 Disabled 或损坏即可。如果此步骤检测到 sppsvc 异常，后续激活状态检查（Step 2）的结果不可信，应优先修复 sppsvc。

### Step 2: KMS 激活状态检查

**数据采集**：

> 采集目标：从 WMI 软件授权数据库中查询批量许可产品信息，获取激活状态和许可证状态原因代码

```powershell
# Query licensed product information for installed product keys
Get-CimInstance -ClassName SoftwareLicensingProduct -Filter "PartialProductKey IS NOT NULL" |
    Select-Object Name, LicenseStatus, LicenseStatusReason, PartialProductKey, KeyManagementServiceMachine, KeyManagementServicePort | Format-List
```

**分析思路**：

1. 检查是否存在授权产品记录：
   - 正常：至少返回一条带有部分产品密钥的记录
   - 异常：无任何记录 → 说明系统未安装产品密钥，可能需要重新安装 GVLK

2. 检查激活状态（LicenseStatus）：
   - 正常：存在至少一条 LicenseStatus = 1（Licensed）的记录
   - 异常：所有记录的 LicenseStatus 均不为 1 → **根因**：Windows 未通过 KMS 激活，**严重程度**：Critical
     - 常见状态值含义：
       - 0（Unlicensed）：未找到有效许可证
       - 2（OOB Grace Period）：安装后的初始宽限期
       - 3（OOT Grace Period）：容错宽限期
       - 4（Non-Genuine Grace Period）：检测到非正版
       - 6（Extended Grace Period）：扩展宽限期

3. 分析许可证状态原因代码（LicenseStatusReason）：
   - 该值为 HRESULT 错误码，根据错误码可判断问题方向：

     | 错误码 | 含义 | 问题方向 |
     |--------|------|----------|
     | 0xC004F074 | KMS 服务器不可用 | 网络/KMS 配置 |
     | 0xC004F06C | 请求时间戳无效（客户端与 KMS 时间偏差 > 4 小时） | 时间同步，参见 → [system-time.md](references/system-time.md) |
     | 0xC004F042 | 指定的 KMS 无法使用 | KMS 配置 |
     | 0xC004F038 | KMS 计数不足（Server 需 ≥ 5） | KMS 主机配置 |
     | 0xC004F069 | 未找到产品密钥 | 密钥安装 |
     | 0xC004F00F | 产品密钥与 Windows 版本不匹配 | 密钥版本 |
     | 0xC004F063 | OEM 版本激活方式不匹配 | 激活方式（非 KMS） |
     | 0xC004F015 | 输入的许可证密钥不匹配当前安装的 Windows SKU | 密钥版本 |
     | 0xC004E002 | 软件授权服务报告许可证存储格式不一致 | Tokens.dat 损坏 |
     | 0xC004E015 | 许可证消费失败（EULA 接受失败） | Tokens.dat 损坏 |
     | 0xC004C003 | 产品密钥被阻止 | 密钥有效性 |
     | 0x8007232B | DNS 名称不存在 | DNS/网络 |
     | 0x8007267C | DNS 服务器未配置 | DNS/网络 |
     | 0x80072EE7 | 服务器名称或地址无法解析 | DNS/网络 |
     | 0x8007000D | 数据无效（产品密钥格式错误） | 密钥格式 |
     | 0x80070005 | 访问被拒绝 | 权限（需管理员） |

   - 分类规则：
     - 网络/DNS 类错误（0x8007xxxx）→ KMS 服务器可达性问题。其中 0x8007232B / 0x8007267C / 0x80072EE7 为 DNS 解析类错误，参见 → [networking-dns.md](references/networking-dns.md)（检查 DNS 客户端服务和 DNS 服务器配置）
     - 许可证类错误（0xC004xxxx）→ 密钥或授权配置问题，参见本文件 Step 2（激活状态检查）和 Step 3（产品密钥与 KMS 配置检查）
     - 0xC004F06C → 单独指向时间同步问题，参见 → [system-time.md](references/system-time.md)
     - 0xC004E0xx → Tokens.dat 损坏，参见修复建议中的「Tokens.dat 重建」

### Step 3: 产品密钥与 KMS 配置检查

**数据采集**：

> 采集目标：获取操作系统版本信息、已安装的产品密钥（后 5 位）和 KMS 服务器配置，并检查 KMS 端口的防火墙规则

```powershell
# 1. Get operating system version
Get-CimInstance Win32_OperatingSystem | Select-Object Caption, Version, BuildNumber | Format-List

# 2. Get product key and KMS configuration
Get-CimInstance -ClassName SoftwareLicensingProduct -Filter "PartialProductKey IS NOT NULL" |
    Select-Object Name, PartialProductKey, KeyManagementServiceMachine, KeyManagementServicePort |
    Format-List

# 3. Check outbound firewall blocking rules for KMS port (default TCP 1688)
$kmsPort = 1688
Get-NetFirewallRule -Enabled True -Direction Outbound -Action Block -ErrorAction SilentlyContinue | ForEach-Object {
    $rule = $_
    Get-NetFirewallPortFilter -AssociatedNetFirewallRule $rule -ErrorAction SilentlyContinue | Where-Object {
        ($_.Protocol -eq 'TCP') -and (($_.RemotePort -eq $kmsPort) -or ($_.RemotePort -eq 'Any'))
    } | ForEach-Object {
        [PSCustomObject]@{
            RuleName    = $rule.DisplayName
            Direction   = $rule.Direction
            Action      = $rule.Action
            Protocol    = $_.Protocol
            RemotePort  = $_.RemotePort
        }
    }
} | Format-Table -AutoSize

# 4. Test KMS server connectivity (Alibaba Cloud internal KMS address)
Test-NetConnection -ComputerName kms.cloud.aliyuncs.com -Port 1688 -WarningAction SilentlyContinue |
    Select-Object ComputerName, RemotePort, TcpTestSucceeded | Format-List
```

**分析思路**：

1. 验证产品密钥（GVLK）是否匹配操作系统版本：
   - 将已安装密钥的后 5 位字符与该 Windows Server 版本对应的 GVLK 预期值进行比对
   - 各版本预期 GVLK 后 5 位：

     | Windows Server 版本 | 预期后 5 位 |
     |---------------------|------------|
     | Server 2025 Datacenter | YP6DF |
     | Server 2025 Standard | MY832 |
     | Server 2022 Datacenter | 6VM33 |
     | Server 2022 Standard | VMK7H |
     | Server 2019 Datacenter | 63DFG |
     | Server 2019 Standard | J464C |
     | Server 2019 Essentials | YY726 |
     | Server 2016 Datacenter | 8XDDG |
     | Server 2016 Standard | KHKQY |
     | Server 2016 Essentials | 4M63B |
     | Server 2012 R2 Datacenter | Q3VJ9 |
     | Server 2012 R2 Standard | MDVJX |
     | Server 2012 R2 Essentials | M4FWM |
     | Server 2012 Datacenter | 8W83P |
     | Server 2012 Standard | 92BT4 |
     | Server 2008 R2 Datacenter | 7M648 |
     | Server 2008 R2 Enterprise | CPX3Y |
     | Server 2008 R2 Standard | R7VHC |

   - 正常：密钥后 5 位与预期值一致
   - 异常：密钥后 5 位不匹配或为空 → **根因**：产品密钥不匹配当前操作系统版本的 GVLK，**严重程度**：Critical
   - 说明：错误的产品密钥（如来自其他版本、零售密钥或 MAK 密钥）会导致 KMS 激活失败，即使网络连通性和 KMS 服务器配置都正确

2. 检查 KMS 端口防火墙规则：
   - 正常：无出站规则阻止 TCP 1688（或自定义 KMS 端口）
   - 异常：存在出站阻止规则覆盖 KMS 端口 → **根因**：防火墙阻止 KMS 通信端口，**严重程度**：Warning
   - 说明：KMS 激活需要客户端通过 TCP 1688（默认）与 KMS 服务器通信，防火墙阻止该端口将导致激活超时

3. 检查 KMS 服务器连通性：
   - 正常：TcpTestSucceeded 为 True
   - 异常：TcpTestSucceeded 为 False → **根因**：KMS 服务器不可达，**严重程度**：Critical
   - 说明：阿里云 ECS 实例通过内网地址 kms.cloud.aliyuncs.com:1688 与 KMS 服务器通信。连通性失败的可能原因：防火墙规则阻止（见上一项）、路由异常、安全组配置、或 KMS 服务器侧问题

> 如果防火墙规则检查需要更深入排查（如配置文件级规则、WFP 丢包分析），参见 → [networking-firewall.md](references/networking-firewall.md)（检查出站 TCP 1688 端口规则）
>
> 如果连通性失败且防火墙未阻止，参见 → [cloud-metaserver.md](references/cloud-metaserver.md)（检查阿里云内网服务可达性）

### Step 4: 激活事件日志检查

**数据采集**：

> 采集目标：查询最近 24 小时内激活相关的事件日志，识别 SPP 服务和激活客户端的错误/警告事件

```powershell
# Query Security-SPP event log (Warning and Error levels in the last 24 hours)
$startTime = (Get-Date).AddHours(-24)
Get-WinEvent -FilterHashtable @{
    LogName = 'Application'
    Level = @(2, 3)  # 2=Error, 3=Warning
    StartTime = $startTime
} -MaxEvents 20 -ErrorAction SilentlyContinue | Where-Object { $_.ProviderName -eq 'Microsoft-Windows-Security-SPP' } | Select-Object TimeCreated, Id, LevelDisplayName, Message | Format-List
```

**分析思路**：

1. 检查是否存在激活相关的错误事件：
   - 正常：无 Error/Warning 级别的 SPP 事件
   - 异常：存在错误事件 → 根据事件 ID 和错误码判断问题方向

2. 常见激活事件 ID 及含义：
   - **Event ID 8198**（SPP Error）：许可证激活失败，通常包含 HRESULT 错误码
     - 含 0xC004F074 → KMS 服务器不可用，检查 Step 3 连通性测试结果
     - 含 0xC004F042 → 产品无法激活，检查 Step 3 产品密钥
     - 含 0xC004F06C → 时间戳无效，客户端与 KMS 服务器时间偏差超过 4 小时
     - 含 0xC004E002 / 0xC004E015 → Tokens.dat 损坏，参见修复建议中的「Tokens.dat 重建」
   - **Event ID 8200 / 900**（SPP Warning）：许可证验证失败，通常伴随 8198 出现
   - **Event ID 8208**（SPP Warning）：许可证续订失败（已激活系统的 7 天续订尝试失败）
   - **Event ID 8193**（SPP Error）：许可证获取失败
   - **Event ID 12288**（SPP Info）：KMS 客户端发送了激活请求，记录了目标 KMS 主机 FQDN 和端口。仅出现 12288 而无 12289 表示客户端无法到达 KMS 主机或未收到响应
   - **Event ID 12289**（SPP Info）：KMS 激活结果，Activation Flag = 1 表示成功、0 表示失败，同时记录 KMS 主机当前计数值
   - **Event ID 1058**（Application Warning）：激活相关的授权问题警告

3. 事件日志中错误码与前序步骤的关联：
   - 网络类错误码（0x8007xxxx）→ 与 Step 3 防火墙和连通性检查结果关联
   - 许可证类错误码（0xC004xxxx）→ 与 Step 2 激活状态和 Step 3 产品密钥检查结果关联
   - **0xC004F06C**（时间偏差）→ 客户端与 KMS 服务器时间差超过 4 小时，需检查系统时间和 NTP 配置

> 如果事件日志包含 0xC004F06C（时间戳无效），参见 → [system-time.md](references/system-time.md)（检查系统时间和 NTP 同步配置）
>
> 如果事件日志显示 KMS 服务器连接失败，且 Step 3 未发现防火墙阻止，参见 → [cloud-metaserver.md](references/cloud-metaserver.md)（检查 KMS 激活服务器可达性）

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 参数化引用 | Step 3 发现防火墙阻止 KMS 端口，需深入排查防火墙规则 | → [networking-firewall.md](references/networking-firewall.md)（检查出站 TCP 1688 端口规则） |
| 条件跳转 | Step 2 错误码为 0x8007232B / 0x8007267C / 0x80072EE7（DNS 解析失败） | → [networking-dns.md](references/networking-dns.md)（检查 DNS 客户端服务和 DNS 服务器配置） |
| 条件跳转 | Step 2 / Step 4 错误码含 0xC004F06C（时间偏差超 4 小时） | → [system-time.md](references/system-time.md)（检查系统时间和 NTP 同步配置） |
| 条件跳转 | Step 2 / Step 4 错误码含 0xC004E002 / 0xC004E015（Tokens.dat 损坏） | → 本文件修复建议「Tokens.dat 重建」 |
| 条件跳转 | Step 3 KMS 连通性失败且非防火墙原因 | → [cloud-metaserver.md](references/cloud-metaserver.md)（检查阿里云内网服务可达性） |
| 链式后继 | 本文件所有步骤执行完毕，未确认根因 | → [cloud-metaserver.md](references/cloud-metaserver.md) |

## 修复建议

### 根因: sppsvc 服务损坏或被禁用

**修复操作**：

```powershell
#requires -RunAsAdministrator
# Fix: Enable and start the sppsvc service
# Risk: No risk in starting the service itself

# Set sppsvc startup type to Automatic (Delayed Start)
Set-Service -Name 'sppsvc' -StartupType Automatic
# Start the service
Start-Service -Name 'sppsvc'
```

**验证方法**：

```powershell
Get-Service -Name 'sppsvc' | Format-List Name, Status, StartType
```

预期结果：StartType = Automatic，Status = Running（或在无激活操作时为 Stopped 均可，关键是 StartType 不为 Disabled）

**风险说明**：启动 sppsvc 服务无副作用。如果服务因系统文件损坏无法启动，可能需要运行 `sfc /scannow` 修复系统文件。

### 根因: Windows 未通过 KMS 激活

**修复操作**：

```powershell
#requires -RunAsAdministrator
# Fix: Manually trigger KMS activation
# Risk: Only attempts activation without modifying system configuration

# Attempt online activation
cscript //Nologo C:\windows\system32\slmgr.vbs /ato
```

**验证方法**：

```powershell
cscript //Nologo C:\windows\system32\slmgr.vbs /dli
```

预期结果：License Status 显示为 Licensed

**风险说明**：`slmgr /ato` 仅触发激活尝试，不修改任何系统配置。如果激活失败，需根据错误码进一步排查。

### 根因: 产品密钥不匹配 GVLK

**修复操作**：

根据操作系统版本从下表选择对应的 GVLK，然后执行安装和激活命令：

| Windows Server 版本 | GVLK |
|---------------------|------|
| Server 2025 Datacenter | D764K-2NDRG-47T6Q-P8T8W-YP6DF |
| Server 2025 Standard | TVRH6-WHNXV-R9WG3-9XRFY-MY832 |
| Server 2022 Datacenter | WX4NM-KYWYW-QJJR4-XV3QB-6VM33 |
| Server 2022 Standard | VDYBN-27WPP-V4HQT-9VMD4-VMK7H |
| Server 2019 Datacenter | WMDGN-G9PQG-XVVXX-R3X43-63DFG |
| Server 2019 Standard | N69G4-B89J2-4G8F4-WWYCC-J464C |
| Server 2019 Essentials | WVDHN-86M7X-466P6-VHXV7-YY726 |
| Server 2016 Datacenter | CB7KF-BWN84-R7R2Y-793K2-8XDDG |
| Server 2016 Standard | WC2BQ-8NRM3-FDDYY-2BFGV-KHKQY |
| Server 2016 Essentials | JCKRF-N37P4-C2D82-9YXRT-4M63B |
| Server 2012 R2 Datacenter | W3GGN-FT8W3-Y4M27-J84CP-Q3VJ9 |
| Server 2012 R2 Standard | D2N9P-3P6X9-2R39C-7RTCD-MDVJX |
| Server 2012 R2 Essentials | KNC87-3J2TX-XB4WP-VCPJV-M4FWM |
| Server 2012 Datacenter | 48HP8-DN98B-MYWDG-T2DCC-8W83P |
| Server 2012 Standard | XC9B7-NBPP2-83J2H-RHMBY-92BT4 |
| Server 2008 R2 Datacenter | 74YFP-3QFB3-KQT8W-PMXWJ-7M648 |
| Server 2008 R2 Enterprise | 489J6-VHDMP-X63PK-3K798-CPX3Y |
| Server 2008 R2 Standard | YC6KT-GKW9T-YTKYR-T4X34-R7VHC |

> 上表覆盖阿里云 ECS 常见版本。

```powershell
#requires -RunAsAdministrator
# Fix: Install the correct GVLK and reactivate
# Risk: Replaces the current product key; if currently using non-KMS activation (e.g., retail key), this will switch to KMS activation mode
# Note: Select the appropriate GVLK from the table above based on the actual OS version and replace the key below

# Install GVLK
cscript //Nologo C:\windows\system32\slmgr.vbs /ipk <SelectCorrespondingGVLKFromTableAbove>

# Reactivate
cscript //Nologo C:\windows\system32\slmgr.vbs /ato
```

**验证方法**：

```powershell
cscript //Nologo C:\windows\system32\slmgr.vbs /dli
```

预期结果：显示正确的 Windows 版本名称和 License Status: Licensed

**风险说明**：安装 GVLK 会替换当前产品密钥。如果实例原本使用非 KMS 方式激活（如零售密钥或 MAK 密钥），此操作将切换为 KMS 激活模式。执行前需确认用户期望使用 KMS 激活。

### 根因: 防火墙阻止 KMS 通信端口

**修复操作**：

```powershell
#requires -RunAsAdministrator
# Fix: Add firewall rule to allow KMS port outbound communication
# Risk: Adds a new outbound allow rule without affecting existing rules

New-NetFirewallRule -DisplayName "Allow KMS Activation (TCP 1688 Outbound)" `
    -Direction Outbound -Action Allow -Protocol TCP -RemotePort 1688 `
    -Profile Any -Enabled True
```

**验证方法**：

```powershell
# Verify the rule has been created
Get-NetFirewallRule -DisplayName "Allow KMS Activation (TCP 1688 Outbound)" | Format-List DisplayName, Enabled, Direction, Action

# Test KMS port connectivity (Alibaba Cloud KMS server)
Test-NetConnection -ComputerName kms.cloud.aliyuncs.com -Port 1688
```

预期结果：规则已启用且方向为 Outbound、Action 为 Allow；Test-NetConnection 显示 TcpTestSucceeded: True

**风险说明**：仅新增出站允许规则，不删除或修改现有防火墙规则。如果存在显式阻止规则且优先级更高，可能还需要禁用或删除该阻止规则。

### 根因: KMS 服务器不可达

**修复操作**：

```powershell
#requires -RunAsAdministrator
# Fix: Set Alibaba Cloud KMS server address and reactivate
# Risk: Modifies KMS server configuration; applicable only to Alibaba Cloud ECS instances

# Set KMS server to Alibaba Cloud internal address
cscript //Nologo C:\windows\system32\slmgr.vbs /skms kms.cloud.aliyuncs.com:1688

# Reactivate
cscript //Nologo C:\windows\system32\slmgr.vbs /ato
```

**验证方法**：

```powershell
# Verify connectivity
Test-NetConnection -ComputerName kms.cloud.aliyuncs.com -Port 1688 -WarningAction SilentlyContinue |
    Select-Object ComputerName, RemotePort, TcpTestSucceeded | Format-List

cscript //Nologo C:\windows\system32\slmgr.vbs /dli
```

预期结果：TcpTestSucceeded = True，License Status = Licensed

**风险说明**：`/skms` 会修改注册表中的 KMS 服务器配置。如果实例已有自定义 KMS 服务器（非阿里云默认），执行前需与用户确认。

### 根因: Tokens.dat 损坏（激活授权数据库损坏）

当 sppsvc 服务正常、产品密钥正确、KMS 网络连通，但激活仍反复失败时，可能是授权令牌文件（Tokens.dat）损坏。典型错误码：0xC004E002（许可证存储格式不一致）、0xC004E015（许可证消费失败）。

**修复操作**：

```powershell
#requires -RunAsAdministrator
# Fix: Rebuild Tokens.dat license token file
# Risk: Clears all current license states; requires reinstalling the product key and reactivating

# 1. Stop Software Protection service
Stop-Service -Name sppsvc -Force

# 2. Rename Tokens.dat (backup) — select path according to OS version:
# ── Windows Server 2012 R2 and above (including Win 8.1/10/11):
Rename-Item "$env:windir\system32\spp\store\2.0\tokens.dat" "tokens.dat.bak" -Force
# ── Windows Server 2012 (including Win 8):
# Rename-Item "$env:windir\system32\spp\store\tokens.dat" "tokens.dat.bak" -Force
# ── Windows Server 2008 R2 (including Win 7):
# Rename-Item "$env:windir\ServiceProfiles\NetworkService\AppData\Roaming\Microsoft\SoftwareProtectionPlatform\tokens.dat" "tokens.dat.bak" -Force

# 3. Restart Software Protection service (Tokens.dat will be regenerated automatically)
Start-Service -Name sppsvc

# 4. Reinstall license
cscript //Nologo C:\windows\system32\slmgr.vbs /rilc

# 5. Restart computer (first restart)
# Restart-Computer -Force

# 6. After restart, reinstall GVLK (select the corresponding version from the key table above)
cscript //Nologo C:\windows\system32\slmgr.vbs /ipk <CorrespondingVersionGVLK>

# 7. Activate
cscript //Nologo C:\windows\system32\slmgr.vbs /ato

# 8. Restart computer again (second restart to ensure full effect)
# Restart-Computer -Force
```

**验证方法**：

```powershell
cscript //Nologo C:\windows\system32\slmgr.vbs /dli
```

预期结果：License Status = Licensed

**风险说明**：重建 Tokens.dat 会清除所有许可证状态，执行后必须重新安装产品密钥。操作前务必确认已记录当前使用的产品密钥或 GVLK。根据微软官方文档（KB2736303），此修复**需要重启计算机两次**才能完全生效。注意不同 Windows 版本的 Tokens.dat 存储路径不同，务必按操作系统版本选择正确路径。
