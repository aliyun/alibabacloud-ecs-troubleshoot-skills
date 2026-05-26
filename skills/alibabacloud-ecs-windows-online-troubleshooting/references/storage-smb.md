# Storage SMB 诊断

## 目录

- [Storage SMB 诊断](#storage-smb-诊断)
  - [目录](#目录)
  - [功能说明](#功能说明)
  - [步骤选取指引](#步骤选取指引)
  - [诊断步骤](#诊断步骤)
    - [Step 1: SMB 客户端配置检查](#step-1-smb-客户端配置检查)
    - [Step 2: 网络组件状态检查](#step-2-网络组件状态检查)
    - [Step 3: 网络发现配置检查](#step-3-网络发现配置检查)
    - [Step 4: Guest 访问策略检查](#step-4-guest-访问策略检查)
  - [交叉引用](#交叉引用)
  - [修复建议](#修复建议)
    - [根因：SMB 签名要求导致客户端连接失败](#根因smb-签名要求导致客户端连接失败)
    - [根因：Windows Server 2025 默认强制签名导致连接第三方 SMB 失败（系统错误 C05D0003）](#根因windows-server-2025-默认强制签名导致连接第三方-smb-失败系统错误-c05d0003)
    - [根因：Windows 10/Server 2016+ 默认阻止不安全的 Guest 登录](#根因windows-10server-2016-默认阻止不安全的-guest-登录)
    - [根因：Guest 认证被阻止（AllowInsecureGuestAuth = 0）](#根因guest-认证被阻止allowinsecureguestauth-0)
    - [根因：Microsoft 网络客户端未安装或未启用](#根因microsoft-网络客户端未安装或未启用)
    - [根因：LanmanWorkstation 未在网络提供程序顺序中配置](#根因lanmanworkstation-未在网络提供程序顺序中配置)

## 功能说明

诊断 Windows SMB Client 端访问共享文件的问题。覆盖 Guest 认证限制、网络组件缺失、LanmanWorkstation ProviderOrder 配置错误、加密/签名配置不兼容、Windows Server 2025 强制签名导致连接第三方 SMB 失败（系统错误 C05D0003）等问题项。

**输入**：用户问题描述（必选）、错误代码/事件ID/截图（可选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 错误代码 C05D0003（电脑配置为需要 SMB 登录） | Step 1 (SMB 签名配置) |
| 可以看到共享但访问被拒绝 (Access Denied) | Step 4 (Guest 访问策略) → Step 1 (加密/签名配置) |
| 错误代码 0xC0000022 或 0x80070005 | Step 4 (Guest 访问) → Step 1 (加密配置) |
| Windows 10/Server 2016+ 无法访问共享 | Step 4 (Guest 访问策略) → Step 1 (SMB 配置) |
| 错误代码 1272（组织的安全策略阻止未经身份验证的来宾访问） | Step 4 (Guest 访问策略) |
| 共享文件夹无法访问、提示网络路径找不到 | Step 2 (网络组件) → Step 3 (网络发现和端口) |
| SMB 连接慢或传输性能差 | Step 1 (SMB 多通道/压缩) |

## 诊断步骤

### Step 1: SMB 客户端配置检查

**数据采集**：

> 采集目标：获取 SMB 客户端的加密配置、签名要求等安全设置，以及当前连接的协议版本

```powershell
# Check SMB client configuration
Get-SmbClientConfiguration | Select-Object EnableSecuritySignature, EnableEncryption, RequireSecuritySignature | Format-Table -AutoSize

# Check current SMB connection protocol version and security features
Get-SmbConnection | Select-Object ServerName, Dialect, Encrypted, Signed, ShareName | Format-Table -AutoSize
```

**分析思路**：

1. 检查 SMB 签名配置（客户端）：
   - 正常状态：EnableSecuritySignature = True（支持签名），RequireSecuritySignature = False（按需签名）
   - 异常状态：RequireSecuritySignature = True 且连接第三方 SMB 服务器，可能导致连接失败
   - 说明：Windows Server 2025 默认启用 RequireSecuritySignature = True，而第三方 SMB 服务器（如 NAS、Linux Samba、云存储）可能不支持签名，导致连接时报系统错误 C05D0003

2. 检查当前连接的协议版本：
   - 正常状态：使用 SMBv2 或 SMBv3（Dialect 显示为 2.x 或 3.x）
   - 异常状态：连接使用 SMBv1（Dialect = 1.0），性能和安全性较差

3. 检查 SMB 加密配置：
   - 正常状态：EnableEncryption = False（按需加密）或 True（强制加密）
   - 异常状态：加密配置与服务器不兼容，可能导致连接失败

4. 检查 SMB 多通道和压缩特性（SMB 3.x）：
   - 正常状态：Get-SmbConnection 显示连接正常
   - 异常状态：多通道未启用但网络适配器支持，可能导致性能未达最优

### Step 2: 网络组件状态检查

**数据采集**：

> 采集目标：获取 Microsoft 网络客户端组件的安装和启用状态，以及 LanmanWorkstation 服务状态和网络提供程序顺序配置

```powershell
# Check network component installation status (using netcfg command)
netcfg -s n

# Check network adapter binding status (Windows 8/Server 2012+)
Get-NetAdapterBinding | Where-Object { $_.ComponentID -eq "ms_msclient" } | Select-Object Name, ComponentID, DisplayName, Enabled | Format-Table -AutoSize

# Check LanmanWorkstation service status
Get-Service -Name LanmanWorkstation | Select-Object Name, Status, StartType | Format-Table -AutoSize

# Check network provider order (registry)
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\NetworkProvider\Order" | Select-Object ProviderOrder | Format-List
```

**分析思路**：

1. 检查 Microsoft 网络客户端（ms_msclient）：
   - 正常状态：组件已安装并启用
   - 异常状态：组件未安装或未启用，将无法访问 SMB 共享
   - 说明：该组件是访问 SMB/CIFS 网络资源的必需组件

2. 检查 LanmanWorkstation 服务状态：
   - 正常状态：服务状态为 Running，启动类型为 Automatic
   - 异常状态：服务未运行或启动类型为非 Automatic，SMB Client 功能将不可用
   - 说明：LanmanWorkstation 服务是 SMB Client 端的核心服务，负责建立和维护与远程服务器的连接

3. 检查网络提供程序顺序（ProviderOrder）：
   - 正常状态：ProviderOrder 注册表值包含 "LanmanWorkstation"
   - 异常状态：LanmanWorkstation 不在 ProviderOrder 中，SMB 连接将失败

### Step 3: 网络发现配置检查

**数据采集**：

> 采集目标：获取网络发现功能状态、相关服务状态和 SMB 端口监听情况

```powershell
# Check network discovery related services
$services = @("fdPHost", "FDResPub", "lmhosts")
Get-Service -Name $services | Select-Object Name, DisplayName, Status, StartType | Format-Table -AutoSize

# Check firewall rules for network discovery
Get-NetFirewallRule -DisplayGroup "Network Discovery" -ErrorAction SilentlyContinue | Select-Object DisplayName, Enabled, Profile, Direction | Format-Table -AutoSize

# Check SMB port listening status (TCP 445)
Get-NetTCPConnection -LocalPort 445 -State Listen -ErrorAction SilentlyContinue | Select-Object LocalPort, State, OwningProcess | Format-Table -AutoSize
```

**分析思路**：

1. 检查网络发现相关服务：
   - 正常状态：服务启动类型为 Manual（手动），状态为 Stopped 或 Running 均可接受
   - 异常状态：服务被禁用（Disabled），将无法在网络上发现其他设备
   - 异常状态：服务启动类型为 Automatic 但状态为 Stopped 且无法启动，网络发现功能异常
   - 说明：这些服务默认是手动启动，按需运行，不需要一直保持 Running 状态

2. 检查防火墙规则：
   - 首先检查防火墙 profile 状态：
     - 如果当前网络 profile（Private/Public）的防火墙已禁用 → 跳过防火墙规则检查，防火墙不会阻止网络发现
     - 如果防火墙已启用 → 继续检查网络发现规则
   - 正常状态：网络发现防火墙规则已启用（专用网络配置文件）
   - 异常状态：防火墙已启用但网络发现规则被禁用，网络发现将被阻止
   - 注意：需要同时检查中英文规则组名称

3. 检查 SMB 端口监听：
   - 正常状态：TCP 445 端口处于监听状态（SMB over TCP）
   - 异常状态：445 端口未监听，Server 服务未启动或 SMB 端口被占用
   - 说明：现代 Windows 使用 SMB over TCP (445)，不再依赖 NetBIOS (137-139)

### Step 4: Guest 访问策略检查

**数据采集**：

> 采集目标：获取 Windows 系统版本和 Guest 身份认证策略配置，特别是 Windows 10/Server 2016 及以上版本的 Insecure Guest Logons 设置

```powershell
# Check system version
Get-CimInstance Win32_OperatingSystem | Select-Object Caption, Version, BuildNumber | Format-Table -AutoSize

# Check LanmanWorkstation parameters (Guest access policy)
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanWorkstation\Parameters" | Select-Object AllowInsecureGuestAuth | Format-Table -AutoSize

# Check network access model
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" | Select-Object LMCompatibilityLevel, RestrictAnonymous, RestrictAnonymousSAM | Format-Table -AutoSize
```

**分析思路**：

1. 检查系统版本对 Guest 访问的影响：
   - Windows Server 2016/Windows 10 1607 及以上版本默认禁用不安全的 Guest 访问
   - Windows Server 2012 R2/Windows 8.1 及更早版本默认允许 Guest 访问
   - 异常状态：新系统尝试访问需要 Guest 认证的共享但未启用 AllowInsecureGuestAuth，Windows 10/Server 2016+ 默认阻止不安全的 Guest 登录

2. 检查 AllowInsecureGuestAuth 注册表值：
   - 正常状态：AllowInsecureGuestAuth = 1（允许 Guest 访问）或共享不需要 Guest 认证
   - 异常状态：AllowInsecureGuestAuth = 0 或 null（禁止 Guest 访问）且共享需要 Guest 认证，Guest 认证将被阻止
   - 说明：该注册表值控制是否允许不安全的 Guest 身份验证

3. 检查 LSA 匿名访问限制：
   - 正常状态：RestrictAnonymous = 0 或 1（允许枚举）
   - 异常状态：RestrictAnonymous = 2（禁止匿名枚举），可能影响共享浏览
   - 说明：LSA 策略控制匿名用户对 SAM 账户和共享的枚举权限

4. 检查 LMCompatibilityLevel（LAN Manager 身份验证级别）：
   - 正常状态：LMCompatibilityLevel = 3（仅发送 NTLMv2）或更高
   - 异常状态：LMCompatibilityLevel = 0 或 1（允许 LM 和 NTLM），存在安全风险
   - 说明：该值影响 SMB 连接使用的身份验证协议

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Step 2 网络组件缺失或未启用 | → [networking-tcpip.md](references/networking-tcpip.md)（检查网络配置和组件） |
| 参数化引用 | Step 3 网络发现被防火墙阻止或 445 端口未监听 | → [networking-firewall.md](references/networking-firewall.md)（检查入站 TCP 445 端口规则） |
| 条件跳转 | Step 1 或 Step 4 发现 SMB 加密/签名/Guest 策略配置问题导致连接失败 | → [system-gpo.md](references/system-gpo.md)（检查组策略中的 SMB 安全策略配置） |
| 链式后继 | 本文件未确认根因，用户报告共享访问问题 | → [networking-tcpip.md](references/networking-tcpip.md)（检查基础网络连通性） |

## 修复建议

### 根因：SMB 签名要求导致客户端连接失败

> 对应诊断：Step 1 (SMB 客户端配置检查) - 分析思路第 1 条

**修复操作**：

```powershell
# Disable mandatory signature requirement on client (allow connecting to SMB servers that do not support signing)
Set-SmbClientConfiguration -RequireSecuritySignature $false -Force
```

**验证方法**：

```powershell
Get-SmbClientConfiguration | Select-Object RequireSecuritySignature, EnableSecuritySignature
```

预期结果：RequireSecuritySignature = False，EnableSecuritySignature = True

**风险说明**：禁用强制签名会降低 SMB 连接的安全性，在不信任的网络中可能遭受中间人攻击。仅在连接内部可信网络中的 SMB 服务器时禁用

---

### 根因：Windows Server 2025 默认强制签名导致连接第三方 SMB 失败（系统错误 C05D0003）

> 对应诊断：Step 1 (SMB 客户端配置检查) - 分析思路第 1 条

**修复操作**：

```powershell
# Disable client mandatory signature requirement
Set-SmbClientConfiguration -RequireSecuritySignature $false -Force
```

**验证方法**：

```powershell
# Check client signature configuration
Get-SmbClientConfiguration | Select-Object RequireSecuritySignature, EnableSecuritySignature | Format-Table -AutoSize

# Test connection to third-party SMB server
net use Z: \\nas-server\share /user:username password

# Or test using PowerShell
Test-Path \\nas-server\share

# Or use New-SmbGlobalMapping (for Server 2012+)
$credential = Get-Credential  # When entering username, format: DOMAIN\Username Note: must specify domain
New-SmbGlobalMapping -RemotePath "\\nas-server\share" -Credential $credential
```

预期结果：
- RequireSecuritySignature = False
- 成功连接第三方 SMB 服务器，无 C05D0003 错误

**风险说明**：
- 禁用强制签名会降低 SMB 连接的安全性，在不信任的网络中可能遭受中间人攻击
- 建议在以下场景禁用：
  - 连接内部可信网络中的第三方 SMB 服务器（NAS、存储设备）
  - 第三方服务器不支持 SMB 签名功能
  - 已采取其他网络安全措施（如 VLAN 隔离、防火墙策略）
- 在域环境或公共网络中，建议保持签名启用，并确保所有 SMB 服务器支持签名
- Windows Server 2025 更改此默认值是为了提高安全性，修改前请评估安全影响

**背景说明**：
- Windows Server 2025 出于安全考虑，默认启用 SMB 强制签名（RequireSecuritySignature = True）
- 旧版本 Windows（2022 及更早）默认值为 False（按需签名）
- 第三方 SMB 实现（如 Linux Samba、NAS 设备、云存储）可能不完全支持 SMB 签名
- 此问题主要影响 Windows Server 2025 作为 Client 端连接第三方 SMB 服务器的场景
- New-SmbGlobalMapping 是 Windows Server 2012 引入的 PowerShell cmdlet，用于创建全局 SMB 映射，比 net use 更适合脚本化操作
- **注意**：使用 New-SmbGlobalMapping 时必须正确指定凭据格式：
  - 域环境：`DOMAIN\Username`
  - 工作组/本地：`计算机名\Username` 或 `.\Username`（可使用 `whoami` 命令查看完整格式）
  - 如果未指定域，会报错"系统错误 1312：指定的会话不存在"

---

### 根因：Windows 10/Server 2016+ 默认阻止不安全的 Guest 登录

> 对应诊断：Step 4 (Guest 访问策略检查) - 分析思路第 1 条

**修复操作**：

```powershell
# Enable insecure Guest logon (allow access to shares requiring Guest authentication)
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanWorkstation\Parameters" `
  -Name "AllowInsecureGuestAuth" -PropertyType DWORD -Value 1 -Force
```

**验证方法**：

```powershell
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanWorkstation\Parameters" | `
  Select-Object AllowInsecureGuestAuth
```

预期结果：AllowInsecureGuestAuth = 1

**风险说明**：启用不安全的 Guest 登录会降低安全性，允许未经验证的访问。仅在访问可信网络中的共享时启用。最佳实践是配置正确的用户认证而非依赖 Guest 访问

---

### 根因：Guest 认证被阻止（AllowInsecureGuestAuth = 0）

> 对应诊断：Step 4 (Guest 访问策略检查) - 分析思路第 2 条

**修复操作**：

```powershell
# Method 1: Enable Guest access (quick fix)
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanWorkstation\Parameters" `
  -Name "AllowInsecureGuestAuth" -Value 1

# Method 2: Use Credential Manager to save authentication info (recommended)
# cmdkey /add:<ServerName> /user:<UserName> /pass:<Password>

# Method 3: Modify Group Policy
# gpedit.msc -> Computer Configuration -> Administrative Templates -> Network -> Lanman Workstation
# Enable "Enable insecure guest logons"
```

**验证方法**：

```powershell
# Check registry
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanWorkstation\Parameters" | `
  Select-Object AllowInsecureGuestAuth

# Try accessing the share
Test-Path "\\<Server>\<Share>"
```

预期结果：AllowInsecureGuestAuth = 1，共享可访问

**风险说明**：Guest 访问不提供身份验证，任何人都可以访问共享。生产环境建议使用域认证或本地用户认证

---

### 根因：Microsoft 网络客户端未安装或未启用

> 对应诊断：Step 2 (网络组件状态检查) - 分析思路第 1 条

**修复操作**：

```powershell
# Enable using PowerShell (Windows 8/Server 2012+)
Enable-NetAdapterBinding -Name "<AdapterName>" -ComponentID "ms_msclient"
```

**验证方法**：

```powershell
# Check component status
Get-NetAdapterBinding -ComponentID "ms_msclient" | Select-Object Name, Enabled

# Verify using netcfg
netcfg -s n | Select-String "ms_msclient"
```

预期结果：ms_msclient 已安装并启用

**风险说明**：Microsoft 网络客户端是访问 SMB 共享的必需组件，禁用后无法访问任何 SMB/CIFS 网络资源

---

### 根因：LanmanWorkstation 未在网络提供程序顺序中配置

> 对应诊断：Step 2 (网络组件状态检查) - 分析思路第 3 条

**修复操作**：

```powershell
# Add LanmanWorkstation to ProviderOrder
$registryPath = "HKLM:\SYSTEM\CurrentControlSet\Control\NetworkProvider\Order"
$currentOrder = (Get-ItemProperty -Path $registryPath).ProviderOrder

if ($currentOrder -eq "") {
    $newOrder = "LanmanWorkstation"
} else {
    $newOrder = "$currentOrder,LanmanWorkstation"
}

Set-ItemProperty -Path $registryPath -Name "ProviderOrder" -Value $newOrder

# Restart Workstation service
Restart-Service LanmanWorkstation -Force
```

**验证方法**：

```powershell
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\NetworkProvider\Order" | `
  Select-Object ProviderOrder

# Test SMB connection
Test-Path "\\<Server>\<Share>"
```

预期结果：ProviderOrder 包含 LanmanWorkstation，SMB 连接正常

**风险说明**：ProviderOrder 配置错误会导致 SMB 连接失败或优先级问题，修改后需要重启 Workstation 服务
