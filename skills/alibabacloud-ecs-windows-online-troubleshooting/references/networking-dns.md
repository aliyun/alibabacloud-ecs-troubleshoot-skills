# DNS 诊断

## 目录

- [DNS 诊断](#dns-诊断)
  - [目录](#目录)
  - [功能说明](#功能说明)
  - [步骤选取指引](#步骤选取指引)
  - [诊断步骤](#诊断步骤)
    - [Step 1: DNS Client 服务状态检查](#step-1-dns-client-服务状态检查)
    - [Step 2: DNS 服务器配置检查](#step-2-dns-服务器配置检查)
    - [Step 3: DNS 缓存状态检查](#step-3-dns-缓存状态检查)
    - [Step 4: Hosts 文件检查](#step-4-hosts-文件检查)
    - [Step 5: 域名解析测试](#step-5-域名解析测试)
    - [Step 6: DNS 后缀和 NRPT 检查](#step-6-dns-后缀和-nrpt-检查)
  - [交叉引用](#交叉引用)
  - [修复建议](#修复建议)
    - [根因: DNS Client 服务异常](#根因-dns-client-服务异常)
    - [根因: DNS 服务器配置问题](#根因-dns-服务器配置问题)
    - [根因: DNS 缓存异常](#根因-dns-缓存异常)
    - [根因: hosts 文件问题](#根因-hosts-文件问题)
    - [根因: 部分域名解析失败](#根因-部分域名解析失败)
    - [根因: DNS 完全不可用](#根因-dns-完全不可用)
    - [根因: DNS 后缀和 NRPT 配置问题](#根因-dns-后缀和-nrpt-配置问题)

## 功能说明

诊断 Windows DNS 客户端服务和解析问题。覆盖 DNS 服务状态、服务器配置、缓存状态、hosts 文件、域名解析测试、DNS 后缀和 NRPT 配置等场景。

**输入**：用户问题描述（必选）、无法访问的域名（可选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| DNS 完全不可用 | Step 1 (DNS 服务) → Step 2 (服务器配置) → Step 5 (解析测试) |
| 特定域名解析失败 | Step 4 (hosts 文件) → Step 5 (解析测试) → Step 3 (缓存检查) |
| DNS 解析慢 | Step 3 (缓存检查) → Step 2 (服务器配置) |
| 域名解析到错误 IP | Step 4 (hosts 文件) → Step 3 (缓存污染) → Step 5 (解析测试) |
| 内网域名解析失败 | Step 6 (DNS 后缀和 NRPT) → Step 2 (服务器配置) |

## 诊断步骤

### Step 1: DNS Client 服务状态检查

**数据采集**：

> 采集目标：获取 DNS Client 服务(Dnscache)的运行状态和启动类型

```powershell
Get-Service -Name Dnscache | Select-Object Name, Status, StartType
```

**分析思路**：

1. 检查 DNS Client 服务状态：
   - 正常：服务正在运行，启动类型为自动
   - 异常：服务未运行 → **根因**：DNS Client 服务未运行，域名解析失败，**严重程度**：Critical
   - 异常：服务被禁用 → **根因**：DNS Client 服务被禁用，**严重程度**：Critical

### Step 2: DNS 服务器配置检查

**数据采集**：

> 采集目标：获取所有网络接口的 DNS 服务器地址配置，包括接口别名和索引

```powershell
Get-DnsClientServerAddress -AddressFamily IPv4 | Where-Object {$_.ServerAddresses -ne $null} | Select-Object InterfaceAlias, InterfaceIndex, ServerAddresses
```

**分析思路**：

1. 检查 DNS 服务器是否配置：
   - 正常：已配置至少一个有效的 DNS 服务器 IP
   - 异常：未配置 DNS 服务器 → **根因**：未配置 DNS 服务器，**严重程度**：Critical

2. 检查 DNS 服务器是否为阿里云 DNS（云服务器典型配置）：
   - 正常：使用阿里云 VPC DNS（100.100.2.136 或 100.100.2.138）
   - 异常：使用公网 DNS且 VPC 网络策略阻止 → **根因**：DNS 服务器配置不适用于 VPC 环境，**严重程度**：Warning

### Step 3: DNS 缓存状态检查

**数据采集**：

> 采集目标：获取 DNS 缓存记录，用于检查是否有缓存污染、解析失败或 hosts 文件映射

```powershell
Get-DnsClientCache | Select-Object Name, Type, Status, DataLength, TimeToLive, Section
```

**分析思路**：

1. 检查 DNS 缓存状态：
   - 正常：有缓存记录且状态为成功
   - 异常：大量缓存失败记录（Status != Success）→ **根因**：DNS 缓存污染，**严重程度**：Warning
   - 异常：无缓存且频繁查询失败 → **根因**：DNS 解析持续失败，**严重程度**：Warning

2. 检查 hosts 文件映射是否生效：
   - hosts 文件的缓存记录特征：TimeToLive = 0，Section = Answer
   - 正常：hosts 文件中的域名在缓存中存在且 TTL=0
   - 异常：hosts 文件有配置但缓存中无对应记录 → **根因**：hosts 文件未生效或格式错误，**严重程度**：Warning

3. 检查缓存中的错误记录：
   - 查找 Status 为负值的记录（如 9003 = DNS_ERROR_RCODE_NAME_ERROR）
   - 异常：特定域名缓存了大量失败记录 → **根因**：DNS 缓存中存在失败的查询结果，**严重程度**：Warning

### Step 4: Hosts 文件检查

**数据采集**：

> 采集目标：获取 hosts 文件内容，检查是否有手动域名映射或错误配置

```powershell
# Read hosts file content (preserve raw text format)
Get-Content C:\Windows\System32\drivers\etc\hosts | Where-Object {$_ -notmatch '^\s*#' -and $_ -notmatch '^\s*$'}

# Check hosts file permissions
Get-Acl C:\Windows\System32\drivers\etc\hosts | Select-Object Owner, Access | Format-List
```

**分析思路**：

1. 检查 hosts 文件格式：
   - 正常：格式为 `IP地址 域名`，每行一个映射
   - 异常：存在语法错误（如缺少空格、IP 格式错误）→ **根因**：hosts 文件格式错误导致解析失败，**严重程度**：Warning
   - 异常：存在重复域名映射 → **根因**：hosts 文件中存在重复域名，仅第一条生效，**严重程度**：Info

2. 检查 hosts 文件是否包含干扰性映射：
   - 异常：将常用域名映射到错误 IP（如 127.0.0.1 或 0.0.0.0）→ **根因**：hosts 文件阻止了域名解析，**严重程度**：Critical
   - 异常：包含恶意域名重定向 → **根因**：hosts 文件被篡改，**严重程度**：Critical

3. 检查 hosts 文件权限：
   - 正常：仅 Administrators 和 SYSTEM 有写入权限
   - 异常：普通用户有写入权限 → **根因**：hosts 文件权限配置不当，**严重程度**：Warning

### Step 5: 域名解析测试

**数据采集**：

> 采集目标：测试常用域名的 DNS 解析能力，验证解析顺序和结果

```powershell
# Test common domain name resolution
Resolve-DnsName -Name www.aliyun.com -Type A -ErrorAction SilentlyContinue | Select-Object Name, IPAddress, Type | Format-Table -AutoSize
Resolve-DnsName -Name www.baidu.com -Type A -ErrorAction SilentlyContinue | Select-Object Name, IPAddress, Type | Format-Table -AutoSize

# Use nslookup for auxiliary verification (shows DNS server used)
nslookup www.aliyun.com

# Test resolution with a specific DNS server
Resolve-DnsName -Name www.aliyun.com -Server 100.100.2.136 -Type A | Select-Object Name, IPAddress | Format-Table -AutoSize
```

**分析思路**：

1. 检查域名解析结果：
   - 正常：能解析常用域名
   - 异常：特定域名解析失败 → **根因**：部分域名解析失败，可能是 DNS 服务器问题或域名本身问题，**严重程度**：Warning
   - 异常：所有域名解析失败 → **根因**：DNS 完全不可用，**严重程度**：Critical

2. 检查解析到的 IP 是否合理：
   - 正常：IP 地址符合预期
   - 异常：解析到错误或私有 IP → **根因**：DNS 缓存污染或 DNS 劫持，**严重程度**：Critical

3. 对比不同 DNS 服务器的解析结果：
   - 正常：不同 DNS 服务器解析结果一致
   - 异常：特定 DNS 服务器解析失败 → **根因**：DNS 服务器配置不适用于当前网络环境，**严重程度**：Warning
   - 异常：不同 DNS 服务器解析结果不一致 → **根因**：DNS 服务器之间存在解析差异，**严重程度**：Info

### Step 6: DNS 后缀和 NRPT 检查

**数据采集**：

> 采集目标：获取 DNS 后缀搜索列表和名称解析策略表（NRPT）配置

```powershell
# Get DNS suffix search list
Get-DnsClientGlobalSetting | Select-Object SuffixSearchList, UseSuffixSearchList | Format-List

# Get per-adapter DNS suffix configuration
Get-DnsClient | Select-Object InterfaceAlias, ConnectionSpecificSuffix, ConnectionSpecificSuffixSearchList | Format-Table -AutoSize

# Get NRPT rules (domain or VPN scenarios)
Get-DnsClientNrptPolicy | Select-Object Namespace, NameServers, DirectAccessServerAddresses | Format-Table -AutoSize
```

**分析思路**：

1. 检查 DNS 后缀搜索列表：
   - 正常：配置了正确的域名后缀（如阿里云内网域名后缀）
   - 异常：未配置后缀搜索列表 → **根因**：短域名无法解析，**严重程度**：Warning
   - 异常：后缀列表包含无效域名 → **根因**：DNS 后缀配置错误导致解析延迟，**严重程度**：Warning

2. 检查 NRPT 规则（如果存在）：
   - 正常：NRPT 规则配置正确且适用于当前网络环境
   - 异常：NRPT 规则阻止特定域名解析 → **根因**：NRPT 策略阻止域名解析，**严重程度**：Warning
   - 异常：NRPT 规则指向不可达的 DNS 服务器 → **根因**：NRPT 配置的 DNS 服务器不可用，**严重程度**：Critical

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Step 2 未配置 DNS 服务器 | → [networking-tcpip.md](references/networking-tcpip.md)(检查 DHCP 或网络配置) |
| 参数化引用 | Step 5 DNS 完全不可用但 IP 连通正常 | → [networking-firewall.md](references/networking-firewall.md)（检查入站/出站 UDP/TCP 53 端口规则） |
| 链式后继 | 本文件未确认根因 | → 无（DNS 问题通常在此文件内定位） |

## 修复建议

### 根因: DNS Client 服务异常

**修复操作**:

```powershell
# Check current service status
Get-Service -Name Dnscache | Select-Object Name, Status, StartType | Format-Table -AutoSize

# If service is disabled, change startup type first
Set-Service -Name Dnscache -StartupType Automatic

# Start the service
Start-Service -Name Dnscache

# Verify service status
Get-Service -Name Dnscache | Select-Object Name, Status, StartType | Format-Table -AutoSize
```

**验证方法**:

```powershell
Get-Service -Name Dnscache
```

预期结果:Status = Running,StartType = Automatic

**风险说明**:启用 DNS Client 服务无风险

---

### 根因: DNS 服务器配置问题

**修复操作**:

```powershell
# Get active network adapter
$adapter = Get-NetAdapter | Where-Object {$_.Status -eq 'Up'} | Select-Object -First 1

if ($adapter) {
    # Configure Alibaba Cloud VPC DNS (for cloud servers)
    Set-DnsClientServerAddress -InterfaceIndex $adapter.InterfaceIndex -ServerAddresses @("100.100.2.136", "100.100.2.138")
    Write-Host "Configured DNS for adapter: $($adapter.Name)"
} else {
    Write-Host "No active network adapter found, please check network connection"
}
```

**验证方法**:

```powershell
Get-DnsClientServerAddress -AddressFamily IPv4 | Where-Object {$_.ServerAddresses -ne $null} | Select-Object InterfaceAlias, ServerAddresses
```

预期结果:ServerAddresses 包含 100.100.2.136 和 100.100.2.138

**风险说明**:修改 DNS 会影响所有域名解析,确保 VPC 网络策略允许访问新 DNS

---

### 根因: DNS 缓存异常

**修复操作**:

```powershell
# Clear DNS cache
Clear-DnsClientCache

ipconfig /flushdns

# Restart DNS Client service (optional, for thorough cleanup)
Restart-Service -Name Dnscache

# Verify cache is cleared
Get-DnsClientCache | Measure-Object | Select-Object Count
```

**验证方法**:

```powershell
Resolve-DnsName -Name www.aliyun.com -Type A | Select-Object Name, IPAddress
```

预期结果:能成功解析域名,缓存数量归零

**风险说明**:清除缓存后首次解析可能稍慢,通常无害

---

### 根因: hosts 文件问题

**修复操作**:

```powershell
# 1. Backup current hosts file
Copy-Item C:\Windows\System32\drivers\etc\hosts C:\Windows\System32\drivers\etc\hosts.bak

# 2. Check hosts file encoding (should be ANSI or UTF-8 without BOM)
Get-Content C:\Windows\System32\drivers\etc\hosts -Encoding Default

# 3. Edit hosts file with Notepad (requires admin privileges)
notepad C:\Windows\System32\drivers\etc\hosts
```

手动修复内容:
- 删除或注释掉阻止域名解析的行(如 `127.0.0.1 www.example.com`)
- 修正格式错误的行(正确格式:`IP地址 域名`)
- 移除重复的域名映射

```powershell
# 4. Fix file permissions
$acl = Get-Acl C:\Windows\System32\drivers\etc\hosts

# Remove all existing permissions
$acl.Access | ForEach-Object { $acl.RemoveAccessRule($_) }

# Add correct permissions
$systemRule = New-Object System.Security.AccessControl.FileSystemAccessRule("SYSTEM", "FullControl", "Allow")
$adminRule = New-Object System.Security.AccessControl.FileSystemAccessRule("Administrators", "FullControl", "Allow")
$usersRule = New-Object System.Security.AccessControl.FileSystemAccessRule("Users", "ReadAndExecute", "Allow")

$acl.AddAccessRule($systemRule)
$acl.AddAccessRule($adminRule)
$acl.AddAccessRule($usersRule)

Set-Acl C:\Windows\System32\drivers\etc\hosts $acl

# 5. Clear DNS cache to force reload of hosts file
Clear-DnsClientCache
```

**验证方法**:

```powershell
# Check if domains in hosts file appear in cache (TTL=0)
Get-DnsClientCache | Where-Object {$_.TimeToLive -eq 0} | Select-Object Name, Data

# Test resolution of domains in hosts file
Resolve-DnsName -Name myserver.local -Type A | Select-Object Name, IPAddress
```

预期结果:hosts 文件中的域名出现在缓存中且 TTL=0,能正确解析到配置的 IP

**风险说明**:hosts 文件编码必须是 ANSI 或 UTF-8 without BOM;权限设置过于宽松可能导致文件被恶意修改;某些安全软件可能会修改 hosts 文件,删除前确认是否为恶意修改

---

### 根因: 部分域名解析失败

**修复操作**:

```powershell
# Try using alternate DNS servers
$adapter = Get-NetAdapter | Where-Object {$_.Status -eq 'Up'} | Select-Object -First 1

if ($adapter) {
    Set-DnsClientServerAddress -InterfaceIndex $adapter.InterfaceIndex -ServerAddresses @("100.100.2.136", "100.100.2.138", "223.5.5.5", "114.114.114.114")
}
```

**验证方法**:

```powershell
Resolve-DnsName -Name www.aliyun.com -Type A | Select-Object Name, IPAddress
Resolve-DnsName -Name www.baidu.com -Type A | Select-Object Name, IPAddress
```

预期结果:常用域名能正常解析

**风险说明**:使用多个 DNS 服务器可提高解析成功率,但需注意某些域名在不同 DNS 上解析结果可能不同

---

### 根因: DNS 完全不可用

**修复操作**:

```powershell
# 1. Start DNS Client service
Set-Service -Name Dnscache -StartupType Automatic
Start-Service -Name Dnscache

# 2. Configure DNS servers
$adapter = Get-NetAdapter | Where-Object {$_.Status -eq 'Up'} | Select-Object -First 1

if ($adapter) {
    Set-DnsClientServerAddress -InterfaceIndex $adapter.InterfaceIndex -ServerAddresses @("100.100.2.136", "100.100.2.138")
    Write-Host "Configured DNS for adapter: $($adapter.Name)"
} else {
    Write-Host "No active network adapter found"
}

# 3. Clear cache
Clear-DnsClientCache
```

**验证方法**:

```powershell
Resolve-DnsName -Name www.aliyun.com -Type A | Select-Object Name, IPAddress
```

预期结果:能成功解析域名

**风险说明**:DNS 完全不可用通常由服务停止或配置错误引起,修复后需验证网络连通性

---

### 根因: DNS 后缀和 NRPT 配置问题

**修复操作**:

```powershell
# Option 1: Configure DNS suffix search list (for short name resolution failure)
Set-DnsClientGlobalSetting -SuffixSearchList @("internal.aliyuncs.com", "aliyun.com") -UseSuffixSearchList $true

# Option 2: Configure connection-specific suffix for a specific adapter
$adapter = Get-NetAdapter | Where-Object {$_.Status -eq 'Up'} | Select-Object -First 1

if ($adapter) {
    Set-DnsClient -InterfaceIndex $adapter.InterfaceIndex -ConnectionSpecificSuffix "internal.aliyuncs.com"
}

# Option 3: Remove problematic NRPT rule (if NRPT is blocking resolution)
# View current NRPT rules
Get-DnsClientNrptPolicy | Select-Object Namespace, NameServers

# Remove specific NRPT rule (requires admin privileges, proceed with caution)
# Remove-DnsClientNrptRule -Namespace "*.example.com" -Force
```

**验证方法**:

```powershell
# Test short name resolution
Resolve-DnsName -Name myserver -Type A | Select-Object Name, IPAddress

# Check DNS suffix configuration
Get-DnsClientGlobalSetting | Select-Object SuffixSearchList, UseSuffixSearchList
```

预期结果:短域名能自动追加后缀并解析成功

**风险说明**:DNS 后缀搜索列表会影响所有短域名解析,确保后缀配置正确;NRPT 通常用于域环境或 DirectAccess/VPN,删除规则前确认是否为企业安全策略要求
