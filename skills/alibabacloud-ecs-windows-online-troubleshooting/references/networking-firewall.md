# Windows 防火墙诊断

## 目录

- [Windows 防火墙诊断](#windows-防火墙诊断)
  - [目录](#目录)
  - [功能说明](#功能说明)
  - [步骤选取指引](#步骤选取指引)
  - [诊断步骤](#诊断步骤)
    - [Step 1: 防火墙服务状态检查](#step-1-防火墙服务状态检查)
    - [Step 2: 防火墙依赖服务检查](#step-2-防火墙依赖服务检查)
    - [Step 3: 防火墙配置文件与活动网络检查](#step-3-防火墙配置文件与活动网络检查)
    - [Step 4: 入站规则匹配检查](#step-4-入站规则匹配检查)
    - [Step 5: 出站规则阻止检查](#step-5-出站规则阻止检查)
    - [Step 6: 组策略防火墙规则合并检查](#step-6-组策略防火墙规则合并检查)
    - [Step 7: WFP 丢包与过滤规则定位](#step-7-wfp-丢包与过滤规则定位)
  - [交叉引用](#交叉引用)
  - [修复建议](#修复建议)
    - [根因：防火墙服务或依赖服务未运行](#根因防火墙服务或依赖服务未运行)
    - [根因：防火墙无生效的入站允许规则](#根因防火墙无生效的入站允许规则)
    - [根因：存在显式 Block 规则覆盖了 Allow 规则](#根因存在显式-block-规则覆盖了-allow-规则)
    - [根因：出站阻止规则范围过大](#根因出站阻止规则范围过大)
    - [根因：防火墙配置文件默认阻止出站连接](#根因防火墙配置文件默认阻止出站连接)

## 功能说明

诊断 Windows 防火墙服务、规则配置及流量过滤问题。覆盖防火墙服务及依赖服务状态、配置文件（Domain/Private/Public）启用与默认动作、入站/出站规则优先级与冲突、组策略规则合并策略、WFP 丢包事件与过滤规则定位等场景。

**输入**：用户问题描述（必选）、被阻止的端口/协议/错误代码（可选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix / cross_ref）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 端口无法访问 / 入站被阻止 | Step 4 (入站规则匹配) → Step 3 (配置文件与活动网络) |
| 应用程序无法联网 / 出站被阻止 | Step 5 (出站规则) → Step 3 (配置文件默认出站动作) |
| 防火墙规则不生效 | Step 1 (服务状态) → Step 2 (依赖服务) → Step 3 (配置文件) |
| 组策略下发的规则不生效 / 本地规则被覆盖 | Step 6 (组策略规则合并) → Step 3 (配置文件) |
| 间歇性网络中断 / 不明丢包 | Step 7 (WFP 丢包与过滤规则定位) |
| 防火墙已禁用但仍出现网络异常 | Step 7 (WFP 丢包与过滤规则定位) — WFP 独立于防火墙服务，即使防火墙禁用，第三方软件仍可通过 WFP 注入过滤规则 |
| 防火墙相关服务崩溃或无法启动 | Step 1 (服务状态) → Step 2 (依赖服务) |

## 诊断步骤

### Step 1: 防火墙服务状态检查

**数据采集**：

> 采集目标：获取 Windows 防火墙服务 (MpsSvc) 的运行状态和启动类型

```powershell
Get-Service -Name MpsSvc | Select-Object Name, Status, StartType
```

**分析思路**：

1. 检查防火墙服务运行状态：
   - 正常：服务正在运行，启动类型为自动
   - 异常：服务未运行 → **根因**：Windows 防火墙服务未运行，所有防火墙规则不生效，网络处于无保护状态，**严重程度**：Warning
   - 异常：服务启动类型为禁用 → **根因**：Windows 防火墙服务被禁用，**严重程度**：Warning
   - 注意：防火墙服务停止不代表 WFP 过滤器全部失效，第三方软件仍可通过 WFP 拦截流量；如存在不明丢包，需进一步检查 Step 7

> 注意：防火墙服务停止后所有规则失效，入站和出站流量均不受控制

### Step 2: 防火墙依赖服务检查

**数据采集**：

> 采集目标：获取防火墙正常运行所依赖的关键服务状态，包括 Base Filtering Engine (BFE)、Network Location Awareness (NlaSvc)、Network List Service (netprofm)

```powershell
@('BFE', 'NlaSvc', 'netprofm') | ForEach-Object {
    Get-Service -Name $_ -ErrorAction SilentlyContinue | Select-Object Name, DisplayName, Status, StartType
}
```

**分析思路**：

1. 检查 Base Filtering Engine (BFE) 服务：
   - 正常：服务正在运行，启动类型为自动
   - 异常：BFE 未运行 → **根因**：Base Filtering Engine 服务未运行，WFP 和防火墙完全失效，**严重程度**：Critical

2. 检查 Network Location Awareness 服务：
   - 正常：服务正在运行
   - 异常：NlaSvc 未运行 → **根因**：NlaSvc 未运行导致网络位置无法识别，防火墙配置文件可能选择错误（回退到 Public），**严重程度**：Warning

3. 检查 Network List Service 服务：
   - 正常：服务正在运行
   - 异常：netprofm 未运行 → **根因**：Network List Service 未运行，影响网络配置文件识别，**严重程度**：Warning

### Step 3: 防火墙配置文件与活动网络检查

**数据采集**：

> 采集目标：获取所有防火墙配置文件的启用状态、默认入站/出站动作，以及当前活动网络连接对应的配置文件类型

```powershell
# Get firewall profile settings
Get-NetFirewallProfile -All | Select-Object Name, Enabled, DefaultInboundAction, DefaultOutboundAction | Format-Table -AutoSize

# Get active network connection profile type
Get-NetConnectionProfile | Select-Object Name, InterfaceAlias, NetworkCategory | Format-Table -AutoSize
```

**分析思路**：

1. 检查配置文件启用状态：
   - 正常：至少一个配置文件已启用
   - 正常：所有配置文件均已禁用 → 云上环境属于预期配置，云平台通过安全组提供网络层防护，防火墙规则不生效无安全风险
   - **若所有配置文件均已禁用，跳过后续 Step 4/5 的规则检查**

2. 检查当前活动网络对应的配置文件类型：
   - 正常：网络类型为 DomainAuthenticated、Private 或 Public（云上实例默认为 Public，属于预期配置）
   - 关注：NlaSvc 异常可能导致网络类型误判，应结合 Step 2 的 NlaSvc 状态分析。深入排查，参见 → [networking-tcpip.md](references/networking-tcpip.md)

3. 检查默认出站动作：
   - 正常：默认出站动作为 Allow（大多数部署的预期配置）
   - 异常：默认出站动作为 Block → **根因**：防火墙配置文件默认阻止出站连接，所有未显式允许的出站流量被丢弃，**严重程度**：Critical

### Step 4: 入站规则匹配检查

**数据采集**：

> 采集目标：根据用户报告的被阻止端口，检查是否存在匹配的已启用入站规则，以及是否有显式阻止规则

```powershell
# Get all enabled inbound rules matching target port (including exact port, port range, multi-port list, Any)
# Note: Replace $targetPort with the actual port number reported by the user
$targetPort = 3389
$rules = Get-NetFirewallRule -Direction Inbound -Enabled True -ErrorAction SilentlyContinue
$allowRules = @()
$blockRules = @()
foreach ($rule in $rules) {
    $portFilter = $rule | Get-NetFirewallPortFilter
    $lp = $portFilter.LocalPort
    $matched = $false
    if ($lp -eq 'Any') {
        $matched = $true
    } elseif ($lp -match '^\d+-\d+$') {
        # Port range, e.g. "1000-5000"
        $range = $lp -split '-'
        if ($targetPort -ge [int]$range[0] -and $targetPort -le [int]$range[1]) { $matched = $true }
    } elseif ($lp -match ',') {
        # Multi-port list, e.g. "80,443,8080"
        if ($lp -split ',' -contains [string]$targetPort) { $matched = $true }
    } else {
        if ($lp -eq [string]$targetPort) { $matched = $true }
    }
    if ($matched) {
        $obj = [PSCustomObject]@{
            DisplayName = $rule.DisplayName
            Action      = $rule.Action
            Profile     = $rule.Profile
            Protocol    = $portFilter.Protocol
            LocalPort   = $portFilter.LocalPort
        }
        if ($rule.Action -eq 'Allow') { $allowRules += $obj }
        elseif ($rule.Action -eq 'Block') { $blockRules += $obj }
    }
}
Write-Host "=== Allow Rules ==="
$allowRules | Format-Table -AutoSize
Write-Host "=== Block Rules ==="
$blockRules | Format-Table -AutoSize
```

**分析思路**：

1. 检查是否存在匹配的允许规则（同时验证规则的 Profile 是否覆盖当前活动配置文件）：
   - 正常：目标端口有已启用的 Allow 规则，且规则覆盖当前活动配置文件
   - 异常：无任何匹配的允许规则，或规则存在但 Profile 未覆盖当前活动网络类型（如规则仅应用于 Domain/Private，但当前为 Public） → **根因**：防火墙无生效的入站允许规则，**严重程度**：Critical

2. 检查是否存在显式阻止规则（规则优先级：Block 规则优先于 Allow 规则）：
   - 正常：无匹配的 Block 规则
   - 异常：存在匹配的 Block 规则 → **根因**：存在显式 Block 规则覆盖了 Allow 规则（Windows 防火墙中 Block 规则始终优先于 Allow 规则），**严重程度**：Critical

如果发现 NlaSvc 异常导致网络类型误判，参见 → [networking-tcpip.md](references/networking-tcpip.md)

### Step 5: 出站规则阻止检查

**数据采集**：

> 采集目标：检查默认出站策略和显式出站阻止规则，排查应用程序无法联网的原因

```powershell
# Get default outbound action for current profiles
Get-NetFirewallProfile -All | Select-Object Name, DefaultOutboundAction | Format-Table -AutoSize

# Get all enabled outbound block rules
Get-NetFirewallRule -Direction Outbound -Action Block -Enabled True -ErrorAction SilentlyContinue |
    Select-Object DisplayName, Profile |
    Format-Table -AutoSize
```

**分析思路**：

1. 检查默认出站动作：
   - 正常：默认出站动作为 Allow
   - 异常：默认出站动作为 Block → 此时所有未显式允许的出站流量均被丢弃，需要确认是否有对应的出站 Allow 规则

2. 检查显式出站阻止规则：
   - 正常：无不合理的出站阻止规则
   - 异常：存在范围过大的出站阻止规则（如阻止所有 TCP 出站）→ **根因**：出站阻止规则范围过大，阻止了正常流量，**严重程度**：Warning

### Step 6: 组策略防火墙规则合并检查

**数据采集**：

> 采集目标：检查组策略是否管理防火墙配置，以及本地规则合并策略是否生效

```powershell
# Check if Group Policy configures firewall rules
$gpFirewallPath = 'HKLM:\SOFTWARE\Policies\Microsoft\WindowsFirewall'
if (Test-Path $gpFirewallPath) {
    Get-ChildItem $gpFirewallPath -Recurse -ErrorAction SilentlyContinue | ForEach-Object {
        Get-ItemProperty $_.PSPath -ErrorAction SilentlyContinue
    } | Select-Object PSPath, EnableFirewall, DefaultInboundAction, DefaultOutboundAction, AllowLocalPolicyMerge, AllowLocalIPsecPolicyMerge -ErrorAction SilentlyContinue | Format-Table -AutoSize
} else {
    Write-Host "No Group Policy firewall configuration detected"
}

# Check current firewall profile local rule merge status
Get-NetFirewallProfile -All | Select-Object Name, AllowLocalFirewallRules, AllowLocalIPsecRules | Format-Table -AutoSize
```

**分析思路**：

1. 检查组策略是否管理防火墙：
   - 正常：无组策略防火墙配置，或组策略与本地配置一致
   - 异常：组策略已配置防火墙但设置与预期不符 → 需要确认组策略来源（域控或本地）

2. 检查本地规则合并策略：
   - 正常：AllowLocalFirewallRules 为 True，本地规则和组策略规则同时生效
   - 信息：AllowLocalFirewallRules 为 False → 组策略禁止本地规则合并，本地创建的规则不生效。这是域管理员的安全策略设计，本身不构成故障。如果用户反馈本地添加的规则不生效，说明需要通过组策略下发规则

3. 检查组策略配置文件设置是否覆盖本地设置：
   - 组策略中的 EnableFirewall、DefaultInboundAction 等设置会覆盖本地配置
   - 异常：组策略将默认入站动作设为 Block 但未下发必要的允许规则 → **根因**：组策略防火墙配置阻止入站但缺少必要允许规则，**严重程度**：Critical

如果发现组策略管理防火墙配置，参见 → [system-gpo.md](references/system-gpo.md)

### Step 7: WFP 丢包与过滤规则定位

> **注意**：WFP (Windows Filtering Platform) 是操作系统底层网络过滤框架，独立于 Windows 防火墙服务。即使防火墙服务已停止或配置文件已禁用，第三方软件（如杀毒软件、VPN、安全加固工具）仍可通过 WFP 注入过滤规则拦截流量。因此，WFP 检查**不依赖防火墙启用状态**，只要存在不明丢包或间歇性网络异常，都应执行本步骤。

**数据采集**：

> 采集目标：导出 WFP 网络事件和过滤规则，用于分析是否存在与用户问题相关的丢包事件，并定位导致丢包的具体过滤规则

```powershell
# Export WFP network events (including packet drop records)
netsh wfp show netevents file="$env:TEMP\netevents.xml"

# Export WFP filter rules (including all active filter definitions)
netsh wfp show filters file="$env:TEMP\filters.xml"

# Verify files were exported successfully
@("$env:TEMP\netevents.xml", "$env:TEMP\filters.xml") | ForEach-Object {
    if (Test-Path $_) {
        Write-Host "$_ size: $([math]::Round((Get-Item $_).Length / 1KB, 1)) KB"
    } else {
        Write-Host "Export failed: $_"
    }
}
```

**分析思路**：

**第一步：从 netevents.xml 中筛选丢包事件**

读取 `netevents.xml`，筛选 `type` 为 `FWPM_NET_EVENT_TYPE_PUBLIC_CLASSIFY_DROP` 的事件项（表示 WFP 分类引擎丢弃了数据包）。从每条丢包事件的 `header` 中提取以下字段：

| 字段 | 说明 |
|------|------|
| `localAddrV4` / `localAddrV6` | 本地 IP 地址 |
| `remoteAddrV4` / `remoteAddrV6` | 远端 IP 地址 |
| `localPort` | 本地端口 |
| `remotePort` | 远端端口 |
| `ipProtocol` | 协议号（6=TCP, 17=UDP） |
| `appId` | 触发流量的应用程序路径 |
| `timeStamp` | 事件发生时间 |

从丢包事件的 `classifyDrop` 节点提取 `filterId`（触发丢弃的过滤器运行时 ID）。

根据用户描述的问题现象（目标端口、目标 IP、协议），在丢包事件中匹配相关记录：
- 正常：无匹配的丢包事件 → WFP 层面未拦截用户报告的流量
- 异常：存在匹配的丢包事件 → 记录事件中的 `filterId`，进入第二步定位过滤规则
- 异常：高频出现相同目标端口/IP 的丢弃事件 → **根因**：WFP 持续丢弃特定流量，**严重程度**：Warning

**第二步：根据 filterId 在 filters.xml 中定位丢包规则**

在 `filters.xml` 中搜索第一步获取的 `filterId`，定位对应的 `<item>` 节点，提取以下信息：

| 字段 | 说明 |
|------|------|
| `displayData > name` | 过滤规则名称（通常对应防火墙规则名） |
| `displayData > description` | 过滤规则描述 |
| `action > type` | 动作类型（`FWP_ACTION_BLOCK` 表示阻止） |
| `providerKey` | 提供者 GUID，用于第三步定位具体软件来源 |
| `filterCondition` | 过滤条件（IP、端口、协议、应用程序等匹配条件） |

根据规则名称初步判断丢包原因：
- 规则名称包含 Windows 防火墙规则名（如 "Core Networking"、用户自建规则名）→ 由 Windows 防火墙规则导致丢弃，回到 Step 4/5 检查对应规则
- `filterId` 在 `filters.xml` 中未找到匹配项 → 该过滤器已被删除或为临时规则，无法追溯来源
- 若需确认规则是否来自第三方软件 → 进入第三步，通过 `providerKey` 定位具体软件

**第三步：根据 providerKey 定位规则来源软件**

从第二步获取的过滤规则中提取 `providerKey`（提供者 GUID），在 `filters.xml` 的 `<providers>` 节点中搜索该 GUID，定位对应的 provider 条目，提取：

| 字段 | 说明 |
|------|------|
| `displayData > name` | 提供者名称（如 "Microsoft Windows Filtering Platform"、第三方软件名） |
| `displayData > description` | 提供者描述，通常包含软件厂商和用途信息 |
| `providerKey` | 提供者 GUID |

根据提供者信息判断丢包来源：

1. 提供者名称为 "Microsoft Windows Filtering Platform" 或 `providerKey` 为 `{decc16ca-3f33-4346-be1e-8fb4ae0f3d62}` → 系统默认 WFP 策略导致丢弃
2. 提供者名称指向第三方软件 → **根因**：第三方软件注入了 WFP 过滤规则导致流量被丢弃，**严重程度**：Warning。从 provider 的 `displayData` 中获取具体软件名称和厂商信息
3. `providerKey` 在 `<providers>` 中未找到匹配 → 提供者信息不可用，仅能从 filter 的 `displayData > name` 推断规则来源

如果 WFP 过滤器指向第三方安全软件，参见 → [security-malware.md](references/security-malware.md)

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Step 7 发现第三方 WFP 过滤器导致丢包 | → [security-malware.md](references/security-malware.md) |
| 条件跳转 | Step 3 发现 NlaSvc 异常可能导致网络类型误判 | → [networking-tcpip.md](references/networking-tcpip.md) |
| 条件跳转 | Step 6 发现组策略管理防火墙配置 | → [system-gpo.md](references/system-gpo.md) |
| 链式后继 | 本文件未确认根因，用户报告网络连通性问题 | → [networking-tcpip.md](references/networking-tcpip.md) |


## 修复建议

### 根因：防火墙服务或依赖服务未运行

涵盖：MpsSvc（防火墙服务）未运行/被禁用、BFE（Base Filtering Engine）未运行、NlaSvc（Network Location Awareness）未运行、netprofm（Network List Service）未运行。

**修复操作**：

```powershell
# Start services in dependency order: BFE → MpsSvc → NlaSvc → netprofm
$services = @(
    @{ Name = 'BFE';      StartupType = 'Automatic' },
    @{ Name = 'MpsSvc';   StartupType = 'Automatic' },
    @{ Name = 'NlaSvc';   StartupType = 'Automatic' },
    @{ Name = 'netprofm'; StartupType = 'Manual'    }
)
foreach ($svc in $services) {
    $current = Get-Service -Name $svc.Name -ErrorAction SilentlyContinue
    if ($current -and $current.Status -ne 'Running') {
        Set-Service -Name $svc.Name -StartupType $svc.StartupType
        Start-Service -Name $svc.Name
        Write-Host "Started: $($svc.Name)"
    }
}
```

**验证方法**：

```powershell
Get-Service -Name BFE, MpsSvc, NlaSvc, netprofm | Select-Object Name, DisplayName, Status, StartType | Format-Table -AutoSize
Get-NetConnectionProfile | Select-Object Name, InterfaceAlias, NetworkCategory | Format-Table -AutoSize
```

预期结果：四个服务均 Status = Running，网络配置文件类型正确识别

**风险说明**：
- BFE 是 WFP 核心引擎，重启会短暂中断所有网络过滤
- 启动 MpsSvc 后会立即启用所有已配置的防火墙规则，可能阻止当前允许的流量，启动前确保关键端口（如 3389）有对应的允许规则
- NlaSvc 重启后会重新检测网络位置，可能导致防火墙配置文件切换（如从 Public 切换到 Domain），进而影响生效的防火墙规则集

---

### 根因：防火墙无生效的入站允许规则

涵盖：无匹配的入站允许规则，或规则存在但 Profile 未覆盖当前活动网络类型。

**修复操作**：

```powershell
# Add inbound allow rule for target port (replace port number and protocol)
$port = 3389
$protocol = 'TCP'
New-NetFirewallRule -DisplayName "Allow $protocol Port $port" -Direction Inbound -Protocol $protocol -LocalPort $port -Action Allow -Profile Any

# If an existing rule has mismatched Profile, modify it directly (replace rule name)
# Set-NetFirewallRule -DisplayName "<RuleName>" -Profile Any
```

**验证方法**：

```powershell
Get-NetFirewallRule -DisplayName "Allow $protocol Port $port" | Select-Object DisplayName, Enabled, Action, Profile
```

预期结果：规则存在，Enabled = True，Action = Allow，Profile 覆盖当前活动配置文件

**风险说明**：仅开放业务必需端口。使用 Profile = Any 会在所有网络类型下开放端口，建议仅扩展到实际需要的配置文件（如 Public）

---

### 根因：存在显式 Block 规则覆盖了 Allow 规则

**修复操作**：

```powershell
# First identify conflicting Block rules (replace port number)
$targetPort = '3389'
Get-NetFirewallRule -Direction Inbound -Action Block -Enabled True | Where-Object {
    ($_ | Get-NetFirewallPortFilter).LocalPort -eq $targetPort -or ($_ | Get-NetFirewallPortFilter).LocalPort -eq 'Any'
} | Select-Object DisplayName, Action, Profile

# Disable conflicting Block rule (replace rule name)
# Disable-NetFirewallRule -DisplayName "<RuleName>"
```

**验证方法**：

```powershell
Get-NetFirewallRule -Direction Inbound -Action Block -Enabled True | Where-Object {
    ($_ | Get-NetFirewallPortFilter).LocalPort -eq $targetPort
} | Measure-Object
```

预期结果：Count = 0，无冲突的 Block 规则

**风险说明**：禁用 Block 规则前需确认该规则的用途，避免引入安全风险。Block 规则可能由安全策略要求

---

### 根因：出站阻止规则范围过大

**修复操作**：

```powershell
# First identify overly broad outbound block rules
Get-NetFirewallRule -Direction Outbound -Action Block -Enabled True | ForEach-Object {
    $portFilter = $_ | Get-NetFirewallPortFilter
    [PSCustomObject]@{
        DisplayName = $_.DisplayName
        Profile     = $_.Profile
        Protocol    = $portFilter.Protocol
        RemotePort  = $portFilter.RemotePort
    }
} | Format-Table -AutoSize

# Disable overly broad rules (replace rule name)
# Disable-NetFirewallRule -DisplayName "<RuleName>"
```

**验证方法**：

```powershell
Get-NetFirewallRule -Direction Outbound -Action Block -Enabled True | Measure-Object
```

预期结果：无不合理的广泛阻止规则

**风险说明**：禁用规则前需确认其用途，部分出站阻止规则可能是安全策略要求。建议缩小规则范围而非直接禁用

---

### 根因：防火墙配置文件默认阻止出站连接

**修复操作**：

```powershell
# Change default outbound action to Allow
Set-NetFirewallProfile -Profile Domain,Private,Public -DefaultOutboundAction Allow
```

**验证方法**：

```powershell
Get-NetFirewallProfile -All | Select-Object Name, DefaultOutboundAction
```

预期结果：所有配置文件 DefaultOutboundAction = Allow

**风险说明**：恢复默认出站允许会降低出站流量控制。如果当前配置是安全策略要求，应改为添加特定出站允许规则而非修改默认动作


