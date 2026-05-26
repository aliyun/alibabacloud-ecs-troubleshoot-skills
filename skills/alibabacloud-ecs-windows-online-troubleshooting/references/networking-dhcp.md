# DHCP 诊断

## 目录

- [DHCP 诊断](#dhcp-诊断)
  - [目录](#目录)
  - [功能说明](#功能说明)
  - [步骤选取指引](#步骤选取指引)
  - [诊断步骤](#诊断步骤)
    - [Step 1: DHCP Client 服务状态检查](#step-1-dhcp-client-服务状态检查)
    - [Step 2: 网卡 DHCP 启用状态检查](#step-2-网卡-dhcp-启用状态检查)
    - [Step 3: DHCP 租约状态检查](#step-3-dhcp-租约状态检查)
    - [Step 4: DHCP 客户端事件日志检查](#step-4-dhcp-客户端事件日志检查)
    - [Step 5: 网卡驱动与链路状态检查](#step-5-网卡驱动与链路状态检查)
    - [Step 6: DHCP 服务器连通性检查](#step-6-dhcp-服务器连通性检查)
  - [交叉引用](#交叉引用)
  - [修复建议](#修复建议)
    - [根因：DHCP Client 服务异常](#根因dhcp-client-服务异常)
    - [根因：网卡未启用 DHCP](#根因网卡未启用-dhcp)
    - [根因：DHCP 获取失败（APIPA 地址）](#根因dhcp-获取失败apipa-地址)
    - [根因：DHCP 租约过期且续租失败](#根因dhcp-租约过期且续租失败)
    - [根因：DHCP 服务器不可达](#根因dhcp-服务器不可达)
    - [根因：网卡被禁用](#根因网卡被禁用)
    - [根因：DHCP 服务器未下发默认网关](#根因dhcp-服务器未下发默认网关)

## 功能说明

诊断 Windows DHCP 客户端服务和 IP 地址自动获取问题。覆盖 DHCP Client 服务状态、网卡 DHCP 启用状态、租约获取与续租、APIPA 地址检测、DHCP 客户端事件日志、DHCP 服务器连通性验证等场景。

**输入**：用户问题描述（必选）、网卡名称或 IP 地址（可选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 无法获取 IP 地址 / IP 全部丢失 | Step 1 (DHCP 服务) → Step 2 (网卡 DHCP 配置) → Step 3 (租约状态) |
| 169.254.x.x (APIPA) 地址 | Step 3 (租约状态) → Step 4 (DHCP 事件日志) → Step 6 (DHCP 服务器连通性) |
| IP 地址频繁变化 / 租约续租失败 | Step 3 (租约状态) → Step 4 (DHCP 事件日志) |
| 部分网卡无法获取 IP | Step 2 (网卡 DHCP 配置) → Step 5 (网卡驱动与链路状态) |
| DHCP 获取 IP 后无法上网 | Step 3 (租约状态) → Step 6 (DHCP 服务器连通性) |
| 防火墙阻止 DHCP 通信 | → [networking-firewall.md](references/networking-firewall.md)（检查入站 UDP 67/68 端口规则） |

## 诊断步骤

### Step 1: DHCP Client 服务状态检查

**数据采集**：

> 采集目标：获取 DHCP Client 服务（Dhcp）的运行状态和启动类型

```powershell
Get-Service -Name Dhcp | Select-Object Name, Status, StartType
```

**分析思路**：

1. 检查 DHCP Client 服务状态：
   - 正常：服务正在运行，启动类型为自动
   - 异常：服务未运行 → **根因**：DHCP Client 服务未运行，无法自动获取 IP 地址，**严重程度**：Critical
   - 异常：服务被禁用 → **根因**：DHCP Client 服务被禁用，**严重程度**：Critical

### Step 2: 网卡 DHCP 启用状态检查

**数据采集**：

> 采集目标：获取所有活动网卡的 DHCP 启用状态和 IP 配置方式

```powershell
Get-NetIPInterface -AddressFamily IPv4 | Where-Object {$_.ConnectionState -eq 'Connected'} | Select-Object InterfaceAlias, InterfaceIndex, Dhcp, ConnectionState
```

**分析思路**：

1. 检查各网卡的 DHCP 启用状态：
   - 正常：DHCP 为 Enabled
   - 异常：DHCP 为 Disabled → **根因**：网卡未启用 DHCP，使用静态 IP 配置，**严重程度**：Warning

2. 检查是否所有活动网卡都禁用了 DHCP：
   - 异常：所有活动网卡均为静态配置且用户期望使用 DHCP → **根因**：DHCP 未在任何网卡上启用，**严重程度**：Critical

### Step 3: DHCP 租约状态检查

**数据采集**：

> 采集目标：获取当前 IP 地址分配信息，包括 DHCP 服务器地址、租约获取时间、租约过期时间

```powershell
# Get IPv4 address information (including APIPA detection)
Get-NetIPAddress -AddressFamily IPv4 | Where-Object {$_.InterfaceAlias -notlike 'Loopback*'} | Select-Object InterfaceAlias, IPAddress, PrefixLength, AddressState, SuffixOrigin, ValidLifetime | Format-Table -AutoSize

# Get DHCP lease details (via WMI)
Get-CimInstance -ClassName Win32_NetworkAdapterConfiguration | Where-Object {$_.IPEnabled -eq $true} | Select-Object Description, DHCPEnabled, DHCPServer, DHCPLeaseObtained, DHCPLeaseExpires, IPAddress, DefaultIPGateway | Format-Table -AutoSize
```

**分析思路**：

1. 检查是否存在 APIPA 地址（169.254.x.x）：
   - 正常：无 169.254.x.x 地址
   - 异常：存在 169.254.x.x 地址（SuffixOrigin 为 Link） → **根因**：DHCP 获取失败，系统分配了 APIPA 自动私有地址，**严重程度**：Critical

2. 检查 DHCP 租约信息：
   - 正常：DHCPServer 有有效地址，租约时间在有效期内
   - 异常：DHCPServer 为空或无效 → **根因**：未能从 DHCP 服务器获取租约，**严重程度**：Critical
   - 异常：租约已过期（DHCPLeaseExpires 早于当前时间） → **根因**：DHCP 租约已过期且续租失败，**严重程度**：Warning

3. 检查是否缺少默认网关：
   - 正常：DefaultIPGateway 有有效网关地址
   - 异常：DHCP 分配了 IP 但未分配网关 → **根因**：DHCP 服务器未下发默认网关，**严重程度**：Warning

如果出现 APIPA 地址且 DHCP 服务正常，可能是防火墙阻止了 DHCP 通信，参见 → [networking-firewall.md](references/networking-firewall.md)（检查入站 UDP 67/68 端口规则）

### Step 4: DHCP 客户端事件日志检查

**数据采集**：

> 采集目标：获取 DHCP 客户端的管理日志事件，重点关注 Event ID 1001（租约获取失败）、1002（租约续订失败）、1003（DHCP 服务错误）

```powershell
Get-WinEvent -LogName 'Microsoft-Windows-Dhcp-Client/Admin' -MaxEvents 30 -ErrorAction SilentlyContinue | Select-Object TimeCreated, Id, LevelDisplayName, Message
```

**分析思路**：

1. 检查 Event ID 1001 事件：
   - 含义：Your computer was not assigned an address from the network (by the DHCP Server)
   - 异常：频繁出现 → **根因**：DHCP 服务器未响应地址请求，可能是服务器不可达或地址池耗尽，**严重程度**：Critical

2. 检查 Event ID 1002 事件：
   - 含义：The IP address lease for the Network Card has been denied by the DHCP server
   - 异常：出现 → **根因**：DHCP 服务器拒绝了租约续订请求，**严重程度**：Warning

3. 检查 Event ID 1003 事件：
   - 含义：DHCP service encountered an error
   - 异常：出现 → **根因**：DHCP 客户端服务遇到内部错误，**严重程度**：Warning

4. 检查事件时间规律：
   - 异常：短时间内大量失败事件 → 可能是网络基础设施问题
   - 异常：仅特定时间段出现 → 可能是网络间歇性故障

### Step 5: 网卡驱动与链路状态检查

**数据采集**：

> 采集目标：获取网卡的连接状态和驱动信息，排除物理层问题

```powershell
Get-NetAdapter | Select-Object Name, InterfaceDescription, Status, LinkSpeed, MediaConnectionState, DriverVersion
```

**分析思路**：

1. 检查网卡连接状态：
   - 正常：Status 为 Up，MediaConnectionState 为 Connected
   - 异常：Status 为 Disabled → **根因**：网卡被禁用，无法进行 DHCP 通信，**严重程度**：Critical
   - 异常：MediaConnectionState 为 Disconnected → **根因**：网卡物理链路断开，**严重程度**：Critical

如果网卡驱动异常，参见 → [device-driver.md](references/device-driver.md)

### Step 6: DHCP 服务器连通性检查

**数据采集**：

> 采集目标：检查与 DHCP 服务器的网络连通性，尝试重新获取 DHCP 租约

```powershell
# Get current DHCP server addresses
$dhcpServers = Get-CimInstance -ClassName Win32_NetworkAdapterConfiguration | Where-Object {$_.DHCPEnabled -eq $true -and $_.DHCPServer} | Select-Object Description, DHCPServer
$dhcpServers

# Test DHCP server connectivity (if DHCP server address is available)
if ($dhcpServers) {
    foreach ($item in $dhcpServers) {
        if ($item.DHCPServer -and $item.DHCPServer -ne '255.255.255.255') {
            Test-Connection -ComputerName $item.DHCPServer -Count 2 -ErrorAction SilentlyContinue | Select-Object Address, StatusCode, ResponseTime
        }
    }
}

# View ipconfig /all DHCP-related information
ipconfig /all
```

**分析思路**：

1. 检查 DHCP 服务器地址：
   - 正常：有有效的 DHCP 服务器地址
   - 异常：无 DHCP 服务器记录 → **根因**：从未成功获取过 DHCP 租约，**严重程度**：Critical

2. 检查 DHCP 服务器连通性：
   - 正常：能 ping 通 DHCP 服务器
   - 异常：无法 ping 通 → **根因**：DHCP 服务器不可达，可能是网络链路问题或服务器故障，**严重程度**：Critical

3. 检查 ipconfig /all 输出中的 DHCP 相关信息：
   - 确认 Autoconfiguration Enabled 状态
   - 确认 DHCP Enabled 状态
   - 检查 Lease Obtained 和 Lease Expires 时间

如果 DHCP 服务器可达但无法获取 IP，可能是服务器端问题（地址池耗尽、MAC 过滤等），需要联系网络管理员。

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Step 2 网卡为静态配置且 IP 配置异常 | → [networking-tcpip.md](references/networking-tcpip.md)（检查静态 IP 配置） |
| 条件跳转 | Step 5 网卡驱动异常 | → [device-driver.md](references/device-driver.md)（检查驱动状态） |
| 参数化引用 | Step 3 APIPA 地址且 DHCP 服务正常，怀疑防火墙阻止 | → [networking-firewall.md](references/networking-firewall.md)（检查入站 UDP 67/68 端口规则） |
| 链式后继 | 本文件未确认根因 | → [networking-tcpip.md](references/networking-tcpip.md) |

## 修复建议

### 根因：DHCP Client 服务异常

**修复操作**：

```powershell
# Set DHCP Client service to automatic startup and start it
Set-Service -Name Dhcp -StartupType Automatic
Start-Service -Name Dhcp
```

**验证方法**：

```powershell
Get-Service -Name Dhcp | Select-Object Name, Status, StartType
```

预期结果：Status = Running，StartType = Automatic

**风险说明**：启用 DHCP Client 服务无风险

---

### 根因：网卡未启用 DHCP

**修复操作**：

```powershell
# Get target adapter (replace <AdapterName> with actual name)
$ifIndex = (Get-NetAdapter -Name '<AdapterName>').InterfaceIndex

# Enable DHCP (remove static IP, switch to DHCP)
Set-NetIPInterface -InterfaceIndex $ifIndex -Dhcp Enabled

# Remove manually configured DNS (allow DHCP-assigned DNS to take effect)
Set-DnsClientServerAddress -InterfaceIndex $ifIndex -ResetServerAddresses

# Trigger DHCP acquisition
ipconfig /release
ipconfig /renew
```

**验证方法**：

```powershell
Get-NetIPInterface -InterfaceIndex $ifIndex -AddressFamily IPv4 | Select-Object Dhcp | Format-Table -AutoSize
Get-NetIPAddress -InterfaceIndex $ifIndex -AddressFamily IPv4 | Select-Object IPAddress, SuffixOrigin | Format-Table -AutoSize
```

预期结果：Dhcp = Enabled，SuffixOrigin = Dhcp

**风险说明**：切换为 DHCP 会丢失原有静态 IP 配置，操作前记录当前 IP/网关/DNS 以便回退

---

### 根因：DHCP 获取失败（APIPA 地址）

**修复操作**：

```powershell
# Restart DHCP Client service
Restart-Service -Name Dhcp

# Release current address and reacquire
ipconfig /release
ipconfig /renew
```

**验证方法**：

```powershell
Get-NetIPAddress -AddressFamily IPv4 | Where-Object {$_.InterfaceAlias -notlike 'Loopback*'} | Select-Object InterfaceAlias, IPAddress, SuffixOrigin
```

预期结果：IP 地址不是 169.254.x.x 段，SuffixOrigin 为 Dhcp

**风险说明**：如果 DHCP 服务器不可用，release 后将失去当前 IP（包括 APIPA），短暂完全无网络连接

---

### 根因：DHCP 租约过期且续租失败

**修复操作**：

```powershell
# Restart DHCP Client service
Restart-Service -Name Dhcp

# Force lease renewal
ipconfig /release
ipconfig /renew

# If still failing, reset Winsock and TCP/IP stack (requires reboot to take effect)
netsh winsock reset
netsh int ip reset
```

**验证方法**：

```powershell
Get-CimInstance -ClassName Win32_NetworkAdapterConfiguration | Where-Object {$_.DHCPEnabled -eq $true} | Select-Object Description, DHCPServer, DHCPLeaseObtained, DHCPLeaseExpires
```

预期结果：DHCPLeaseExpires 晚于当前时间

**风险说明**：netsh winsock reset 和 netsh int ip reset 需要重启系统才能生效，可能影响其他网络配置

---

### 根因：DHCP 服务器不可达

**修复操作**：

```powershell
# Check network adapter link status
Get-NetAdapter | Select-Object Name, Status, MediaConnectionState | Format-Table -AutoSize

# Try to reacquire DHCP lease
ipconfig /release
ipconfig /renew

# If still failing, temporarily configure static IP to restore network connectivity (replace <> with actual values)
# New-NetIPAddress -InterfaceAlias '<AdapterName>' -IPAddress '<IPAddress>' -PrefixLength <SubnetPrefix> -DefaultGateway '<Gateway>'
# Set-DnsClientServerAddress -InterfaceAlias '<AdapterName>' -ServerAddresses @('<DNS1>', '<DNS2>')
```

**验证方法**：

```powershell
Get-NetIPAddress -AddressFamily IPv4 | Where-Object {$_.InterfaceAlias -notlike 'Loopback*'} | Select-Object InterfaceAlias, IPAddress | Format-Table -AutoSize
Test-Connection -ComputerName 100.100.2.136 -Count 2
```

预期结果：有有效 IP 且能 ping 通网关

**风险说明**：DHCP 服务器不可达通常是网络基础设施问题，静态 IP 仅作为临时恢复方案，需联系网络管理员排查 DHCP 服务器

---

### 根因：网卡被禁用

**修复操作**：

```powershell
# Enable network adapter (replace <AdapterName> with actual name)
Enable-NetAdapter -Name '<AdapterName>'

# Wait for adapter to be ready, then acquire DHCP
Start-Sleep -Seconds 3
ipconfig /renew
```

**验证方法**：

```powershell
Get-NetAdapter -Name '<AdapterName>' | Select-Object Name, Status, MediaConnectionState
Get-NetIPAddress -InterfaceAlias '<AdapterName>' -AddressFamily IPv4 | Select-Object IPAddress
```

预期结果：Status = Up，有有效 IP 地址

**风险说明**：启用网卡无风险

---

### 根因：DHCP 服务器未下发默认网关

**修复操作**：

```powershell
# Manually add default gateway (replace <> with actual values)
$ifIndex = (Get-NetAdapter -Name '<AdapterName>').InterfaceIndex
New-NetRoute -InterfaceIndex $ifIndex -DestinationPrefix '0.0.0.0/0' -NextHop '<GatewayAddress>'
```

**验证方法**：

```powershell
Get-NetRoute -InterfaceIndex $ifIndex -DestinationPrefix '0.0.0.0/0' | Select-Object NextHop
Test-Connection -ComputerName www.aliyun.com -Count 2
```

预期结果：有默认路由且能 ping 通外网

**风险说明**：手动添加网关仅作为临时方案，下次 DHCP 续租可能覆盖或冲突，需联系网络管理员修正 DHCP 服务器配置


