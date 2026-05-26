# Network TCP/IP 诊断

## 目录

- [Network TCP/IP 诊断](#network-tcpip-诊断)
  - [目录](#目录)
  - [功能说明](#功能说明)
  - [步骤选取指引](#步骤选取指引)
  - [诊断步骤](#诊断步骤)
    - [Step 1: 网络适配器状态检查](#step-1-网络适配器状态检查)
    - [Step 2: 网络协议绑定检查](#step-2-网络协议绑定检查)
    - [Step 3: IP 地址配置检查](#step-3-ip-地址配置检查)
    - [Step 4: 默认网关检查](#step-4-默认网关检查)
    - [Step 5: DNS 服务器配置检查](#step-5-dns-服务器配置检查)
    - [Step 6: 端到端连通性探测](#step-6-端到端连通性探测)
    - [Step 7: TCP 端口范围检查](#step-7-tcp-端口范围检查)
    - [Step 8: 代理配置检查](#step-8-代理配置检查)
  - [交叉引用](#交叉引用)
  - [修复建议](#修复建议)
    - [根因：网络适配器被禁用](#根因网络适配器被禁用)
    - [根因：TCP/IP 协议未绑定](#根因tcpip-协议未绑定)
    - [根因：无网络适配器](#根因无网络适配器)
    - [根因：网络适配器状态未知](#根因网络适配器状态未知)
    - [根因：网络适配器未连接或底层链路断开](#根因网络适配器未连接或底层链路断开)
    - [根因：网卡链路未建立](#根因网卡链路未建立)
    - [根因：检测到第三方网络协议绑定](#根因检测到第三方网络协议绑定)
    - [根因：未分配 IPv4 地址](#根因未分配-ipv4-地址)
    - [根因：DHCP 获取失败，使用自动私有地址](#根因dhcp-获取失败使用自动私有地址)
    - [根因：IP 地址状态异常](#根因ip-地址状态异常)
    - [根因：默认路由缺失，无法访问外网](#根因默认路由缺失无法访问外网)
    - [根因：默认网关配置无效](#根因默认网关配置无效)
    - [根因：未配置 DNS 服务器](#根因未配置-dns-服务器)
    - [根因：DNS 服务器不可达](#根因dns-服务器不可达)
    - [根因：TCP 动态端口范围过小](#根因tcp-动态端口范围过小)
    - [根因:TCP 排除端口过多](#根因tcp-排除端口过多)
    - [根因：网关不可达](#根因网关不可达)
    - [根因：DNS 解析失败](#根因dns-解析失败)
    - [根因：WinHTTP 代理与 IE 代理不一致](#根因winhttp-代理与-ie-代理不一致)
    - [根因：检测到环境变量代理配置](#根因检测到环境变量代理配置)
    - [根因：代理配置错误](#根因代理配置错误)

## 功能说明

诊断 Windows 网络适配器和 TCP/IP 协议栈配置问题。覆盖网卡状态、IP 配置、路由表、协议绑定、代理配置、端到端连通性等 10 个已知问题项。

**输入**：用户问题描述（必选）、错误代码/事件ID/截图（可选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 网络完全不通 | Step 6 (连通性) → 如失败则倒查 Step 1-5 |
| 网络图标异常/偶尔断流 | Step 1 (网卡状态) → Step 2 (协议绑定) |
| 获取不到 IP 地址 | Step 3 (IP 配置) → Step 4 (网关) |
| 某些网站打不开 | Step 5 (DNS) → Step 8 (代理) |
| 网络慢/延迟高 | Step 6 (连通性测试延迟) |
| 应用无法联网但 ping 正常 | Step 8 (代理配置) |
| 应用无法绑定端口/报错端口被占用 | Step 7 (TCP 端口范围) |
| 大量并发连接失败 | Step 7 (TCP 端口范围) → Step 6 (连通性) |

## 诊断步骤

### Step 1: 网络适配器状态检查

**数据采集**：

> 采集目标：获取所有网络适配器的基本状态信息，包括名称、描述、连接状态、MAC 地址和链路速度

```powershell
Get-NetAdapter | Select-Object Name, InterfaceDescription, Status, MacAddress, LinkSpeed
```

**分析思路**：

1. 检查是否有网络适配器：
   - 正常：至少有一个适配器
   - 异常：无适配器 → **根因**：无网络适配器，**严重程度**：Critical

2. 检查适配器启用状态：
   - 正常：适配器处于已启用或已启动状态
   - 异常：适配器被禁用 → **根因**：网络适配器被禁用，**严重程度**：Warning
   - 异常：适配器状态未知 → **根因**：网络适配器状态未知，**严重程度**：Warning

3. 检查适配器连接状态：
   - 正常：适配器已连接
   - 异常：适配器未连接或底层链路断开 → **根因**：网络适配器未连接或底层链路断开，**严重程度**：Critical

4. 检查链路速度：
   - 正常：链路速度大于 0
   - 异常：链路速度为 0 → **根因**：网卡链路未建立，**严重程度**：Critical
> 如果发现有网卡状态是 Down，可能需要在设备管理器检查驱动 → 参见 [device-driver.md](references/device-driver.md)

### Step 2: 网络协议绑定检查

**数据采集**：

> 采集目标：获取所有网络适配器上绑定的协议组件，用于检查 TCP/IP 协议绑定状态和第三方协议

```powershell
Get-NetAdapterBinding | Select-Object Name, ComponentID, Enabled, DisplayName
```

**分析思路**：

1. 检查 TCP/IP 协议是否绑定到活动网卡：
   - 正常：活动网卡已绑定 TCP/IP 协议
   - 异常：TCP/IP 协议未绑定 → **根因**：TCP/IP 协议未绑定到网卡,**严重程度**:Critical

2. 检查是否存在第三方协议绑定：
   - 正常：仅绑定 Microsoft 标准协议
   - 异常：存在非 Microsoft 协议 → **根因**：检测到第三方网络协议绑定，可能导致网络访问异常，**严重程度**：Warning
   - 常见第三方协议：杀毒软件网络过滤驱动、VPN 客户端协议、虚拟化网络协议、流量监控工具等

### Step 3: IP 地址配置检查

**数据采集**：

> 采集目标：获取所有 IPv4 地址配置，包括 IP 地址、前缀长度、地址状态和分配来源

```powershell
Get-NetIPAddress -AddressFamily IPv4 | Select-Object InterfaceAlias, IPAddress, PrefixLength, AddressState, PrefixOrigin, SuffixOrigin
```

**分析思路**：

1. 检查是否有 IPv4 地址：
   - 正常：至少有一个首选状态的 IPv4 地址
   - 异常：无 IPv4 地址 → **根因**：未分配 IPv4 地址，**严重程度**：Critical

2. 检查是否为 APIPA 地址（169.254.x.x）：
   - 正常：非 169.254.x.x 段
   - 异常：IP 地址以 169.254 开头 → **根因**：DHCP 获取失败，使用自动私有地址，**严重程度**：Warning

3. 检查地址状态：
   - 正常：地址状态为首选
   - 异常：地址状态为暂定或已弃用 → **根因**：IP 地址状态异常，**严重程度**：Warning

如果发现 APIPA 地址,检查 DHCP 服务 → 参见 [networking-dhcp.md](references/networking-dhcp.md)

### Step 4: 默认网关检查

**数据采集**：

> 采集目标：获取默认路由(0.0.0.0/0)配置，包括下一跳网关地址和路由度量值

```powershell
Get-NetRoute -DestinationPrefix "0.0.0.0/0" -AddressFamily IPv4 | Select-Object InterfaceAlias, NextHop, RouteMetric, Protocol
```

**分析思路**：

1. 检查默认路由是否存在：
   - 正常：存在至少一条默认路由
   - 异常：无默认路由 → **根因**：默认路由缺失，无法访问外网，**严重程度**：Critical

2. 检查下一跳网关是否有效：
   - 正常：下一跳为有效的 IP 地址
   - 异常：下一跳为 0.0.0.0 → **根因**：默认网关配置无效,**严重程度**:Warning

### Step 5: DNS 服务器配置检查

**数据采集**：

> 采集目标：获取所有网络接口的 DNS 服务器地址配置

```powershell
Get-DnsClientServerAddress -AddressFamily IPv4 | Where-Object {$_.ServerAddresses -ne $null} | Select-Object InterfaceAlias, ServerAddresses
```

**分析思路**：

1. 检查 DNS 服务器是否配置：
   - 正常：已配置至少一个 DNS 服务器 IP
   - 异常：未配置 DNS 服务器 → **根因**：未配置 DNS 服务器，**严重程度**：Warning

2. 检查 DNS 服务器可达性（可选）：
   - 正常：能 ping 通 DNS 服务器
   - 异常：DNS 服务器不可达 → **根因**：DNS 服务器不可达，**严重程度**：Warning

如果发现 DNS 配置有异常,执行 DNS 诊断 → 参见 [networking-dns.md](references/networking-dns.md)

### Step 6: 端到端连通性探测

**数据采集**：

> 采集目标：测试网关连通性、外网连通性和 DNS 解析能力

```powershell
# Get default gateway
Get-NetRoute -DestinationPrefix "0.0.0.0/0" -AddressFamily IPv4 | Select-Object -First 1 -ExpandProperty NextHop

# Test gateway connectivity (replace <GatewayIP> with actual gateway address)
Test-Connection -ComputerName <GatewayIP> -Count 2 -ErrorAction SilentlyContinue | Select-Object Address, IPv4Address, ResponseTime | Format-Table -AutoSize

# Test external connectivity
Test-Connection -ComputerName 223.5.5.5 -Count 2 -ErrorAction SilentlyContinue | Select-Object Address, IPv4Address, ResponseTime | Format-Table -AutoSize

# Test DNS resolution
Resolve-DnsName -Name www.aliyun.com -Type A -ErrorAction SilentlyContinue | Select-Object Name, IPAddress | Format-Table -AutoSize
```

**分析思路**：

1. 检查网关连通性：
   - 正常：能 ping 通网关
   - 异常：网关不可达 → **根因**：网关不可达，**严重程度**：Critical

2. 检查外网连通性：
   - 正常：能 ping 通外网地址
   - 异常:外网不可达 → 参见 [networking-firewall.md](references/networking-firewall.md)(检查防火墙规则) 或 [networking-dns.md](references/networking-dns.md)(检查 DNS 配置)

3. 检查 DNS 解析：
   - 正常：能解析域名
   - 异常：解析失败但 IP 连通 → **根因**：DNS 解析失败，**严重程度**：Warning

### Step 7: TCP 端口范围检查

**数据采集**：

> 采集目标:获取 TCP/UDP 动态端口范围和系统保留(排除)端口范围,检查是否被异常修改

```powershell
# Query TCP dynamic port range
netsh int ipv4 show dynamicport tcp
netsh int ipv6 show dynamicport tcp

# Query TCP excluded port range
netsh int ipv4 show excludedportrange tcp

# Query UDP dynamic port range
netsh int ipv4 show dynamicport udp

# Query UDP excluded port range
netsh int ipv4 show excludedportrange udp
```

**分析思路**：

1. 检查 TCP 动态端口范围:
   - 正常:启动端口 49152,端口数 16384(范围 49152-65535)
   - 异常:端口数过少(< 10000) → **根因**:TCP 动态端口范围过小,**严重程度**:Warning

2. 检查排除端口范围:
   - 正常:排除端口数量合理(< 50 个)
   - 异常:排除端口过多(> 100 个) → **根因**:TCP 排除端口过多,**严重程度**:Warning
   - 异常:常用端口被排除(如 80, 443, 3389) → **根因**:关键端口被系统保留,**严重程度**:Critical

### Step 8: 代理配置检查

**数据采集**：

> 采集目标：获取 WinHTTP 代理、IE 代理注册表配置以及环境变量中的代理设置

```powershell
# Check WinHTTP proxy configuration
netsh winhttp show proxy

# Check IE/system proxy configuration (registry - current user)
Get-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings" -Name ProxyEnable, ProxyServer, ProxyOverride, AutoConfigURL -ErrorAction SilentlyContinue | Select-Object ProxyEnable, ProxyServer, ProxyOverride, AutoConfigURL

# Check proxy configuration in environment variables
Write-Output "HTTP_PROXY=$($env:HTTP_PROXY)"
Write-Output "HTTPS_PROXY=$($env:HTTPS_PROXY)"
Write-Output "NO_PROXY=$($env:NO_PROXY)"
```

**分析思路**：

1. 检查是否启用代理：
   - 正常：未启用代理（直连）
   - 异常：已启用代理且代理服务器有效 → 确认代理服务器是否可达
   - 异常：已启用代理但代理服务器不可达 → **根因**：代理配置错误导致网络不可达，**严重程度**：Warning

2. 检查 WinHTTP 与 IE 代理是否一致：
   - 正常：两者一致或均为空
   - 异常：不一致 → **根因**：WinHTTP 代理与 IE 代理不一致,Windows Update 等服务可能受影响,**严重程度**:Warning

3. 检查环境变量中的代理配置：
   - 正常：未配置代理环境变量
   - 异常：配置了代理环境变量 → **根因**：检测到环境变量代理配置，可能影响部分应用程序的网络访问,**严重程度**:Warning

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Step 3 发现 APIPA 地址(169.254.x.x) | → [networking-dhcp.md](references/networking-dhcp.md) |
| 条件跳转 | Step 5/6 DNS 解析失败 | → [networking-dns.md](references/networking-dns.md) |
| 条件跳转 | Step 6 连通性测试失败但配置正常 | → [networking-firewall.md](references/networking-firewall.md) |
| 链式后继 | 本文件未确认根因,用户报告网络问题 | → [networking-firewall.md](references/networking-firewall.md) |

## 修复建议

### 根因：网络适配器被禁用

**修复操作**：

```powershell
# Enable network adapter (replace <AdapterName> with actual name)
Enable-NetAdapter -Name "<AdapterName>" -Confirm:$false
```

**验证方法**：

```powershell
Get-NetAdapter -Name "<AdapterName>" | Select-Object Name, State, Status
```

预期结果：网卡状态为已启用且已连接

**风险说明**：启用网卡无风险，但如果网卡驱动有问题可能无法成功启用

---

### 根因：TCP/IP 协议未绑定

**修复操作**：

```powershell
# Re-bind TCP/IP protocol (replace <AdapterName> with actual name)
Enable-NetAdapterBinding -Name "<AdapterName>" -ComponentID ms_tcpip
```

**验证方法**：

```powershell
Get-NetAdapterBinding -Name "<AdapterName>" -ComponentID ms_tcpip | Select-Object Name, ComponentID, Enabled
```

预期结果：TCP/IP 协议绑定已启用

**风险说明**：重新绑定协议可能导致短暂网络中断

---

### 根因：无网络适配器

**修复操作**：

在设备管理器中扫描硬件更改，或检查虚拟化平台是否正确挂载了虚拟网卡

```powershell
# Scan for hardware changes
pnputil /scan-devices
```

**验证方法**：

```powershell
Get-NetAdapter | Select-Object Name, InterfaceDescription, Status
```

预期结果：至少显示一个网络适配器

**风险说明**：如果是虚拟化环境，需要在云控制台检查实例的网络配置

---

### 根因：网络适配器状态未知

**修复操作**：

```powershell
# Disable then re-enable the network adapter (replace <AdapterName> with actual name)
Disable-NetAdapter -Name "<AdapterName>" -Confirm:$false
Start-Sleep -Seconds 2
Enable-NetAdapter -Name "<AdapterName>" -Confirm:$false
```

**验证方法**：

```powershell
Get-NetAdapter -Name "<AdapterName>" | Select-Object Name, State, Status
```

预期结果：网卡状态从未知变为已启用

**风险说明**：重置网卡会导致短暂网络中断

---

### 根因：网络适配器未连接或底层链路断开

**修复操作**:

```powershell
# 1. Check network adapter physical connection status
Get-NetAdapter | Where-Object {$_.MediaConnectionState -eq 'Disconnected'} | Select-Object Name, MediaConnectionState, InterfaceDescription | Format-Table -AutoSize

# 2. Try to enable the adapter (if it was disabled)
$disconnectedAdapter = Get-NetAdapter | Where-Object {$_.MediaConnectionState -eq 'Disconnected' -and $_.State -eq 'Disabled'}
if ($disconnectedAdapter) {
    Enable-NetAdapter -Name $disconnectedAdapter.Name -Confirm:$false
}

# 3. If in a virtualized environment, check VirtIO network adapter driver status
Get-PnpDevice -Class Net | Where-Object {$_.FriendlyName -like '*VirtIO*'} | Select-Object FriendlyName, Status | Format-Table -AutoSize
```

**验证方法**:

```powershell
Get-NetAdapter | Select-Object Name, MediaConnectionState, LinkSpeed
```

预期结果:网卡连接状态显示为已连接,链路速度显示具体数值(非 0)

**风险说明**:如果云平台层面网络配置异常,需要在云控制台检查实例网络配置或联系技术支持

---

### 根因：网卡链路未建立

**修复操作**：

```powershell
# Reset network adapter driver (replace <AdapterName> with actual name)
Restart-NetAdapter -Name "<AdapterName>"

# If still failing, check driver status
Get-PnpDevice | Where-Object {$_.FriendlyName -like "*<AdapterName>*"} | Select-Object Status, Class
```

**验证方法**：

```powershell
Get-NetAdapter -Name "<AdapterName>" | Select-Object Name, LinkSpeed
```

预期结果：LinkSpeed > 0

**风险说明**：重启网卡会导致网络短暂中断，如果驱动有问题可能需要更新驱动

---

### 根因：检测到第三方网络协议绑定

**修复操作**：

```powershell
# View third-party protocols (replace <AdapterName> with actual name)
Get-NetAdapterBinding -Name "<AdapterName>" | Where-Object {$_.ComponentID -notlike 'ms_*'} | Select-Object Name, ComponentID, DisplayName

# Disable suspicious third-party protocol (proceed with caution)
# Disable-NetAdapterBinding -Name "<AdapterName>" -ComponentID <ThirdPartyProtocolID>
```

**验证方法**：

```powershell
Get-NetAdapterBinding -Name "<AdapterName>" | Select-Object Name, ComponentID, Enabled
```

预期结果：仅保留 Microsoft 标准协议

**风险说明**：禁用第三方协议可能影响杀毒软件、VPN等功能，需确认协议用途后再操作

---

### 根因：未分配 IPv4 地址

**修复操作**：

```powershell
# If using DHCP, release and renew IP (replace <AdapterName> with actual name)
ipconfig /release
ipconfig /renew

# If using static IP, manually configure IP (example)
# New-NetIPAddress -InterfaceAlias "<AdapterName>" -IPAddress "192.168.1.100" -PrefixLength 24 -DefaultGateway "192.168.1.1"
```

**验证方法**：

```powershell
Get-NetIPAddress -AddressFamily IPv4 | Select-Object InterfaceAlias, IPAddress, AddressState
```

预期结果：至少有一个首选状态的 IPv4 地址

**风险说明**：如果使用静态 IP，需确保 IP 地址、子网掩码、网关配置正确

---

### 根因：DHCP 获取失败，使用自动私有地址

**修复操作**：

```powershell
# Check DHCP service status
Get-Service -Name Dhcp | Select-Object Name, Status, StartType

# Restart DHCP client service
if ((Get-Service -Name Dhcp).Status -ne 'Running') {
    Start-Service -Name Dhcp
}

# Renew IP
ipconfig /release
ipconfig /renew
```

**验证方法**：

```powershell
Get-NetIPAddress -AddressFamily IPv4 | Where-Object {$_.IPAddress -notlike '169.254.*'} | Select-Object IPAddress
```

预期结果：IP 地址不是 169.254.x.x 段

**风险说明**：如果 DHCP 服务器不可用，需要检查网络基础设施

---

### 根因：IP 地址状态异常

**修复操作**：

```powershell
# Reset network adapter (replace <AdapterName> with actual name)
Reset-NetAdapterAdvancedProperty -Name "<AdapterName>" -All
Restart-NetAdapter -Name "<AdapterName>"
```

**验证方法**：

```powershell
Get-NetIPAddress -AddressFamily IPv4 | Select-Object InterfaceAlias, IPAddress, AddressState
```

预期结果：AddressState = Preferred

**风险说明**：重置网卡高级属性会恢复默认设置，可能影响自定义配置

---

### 根因：默认路由缺失，无法访问外网

**修复操作**：

```powershell
# Add default route (replace <GatewayIP> and <AdapterName> with actual values)
New-NetRoute -DestinationPrefix "0.0.0.0/0" -NextHop "<GatewayIP>" -InterfaceAlias "<AdapterName>"
```

**验证方法**：

```powershell
Get-NetRoute -DestinationPrefix "0.0.0.0/0" | Select-Object InterfaceAlias, NextHop
```

预期结果：存在默认路由，NextHop 为有效网关地址

**风险说明**：添加错误的路由可能导致网络不通，需确认网关地址正确

---

### 根因：默认网关配置无效

**修复操作**：

```powershell
# Remove invalid route (replace <AdapterName> with actual name)
Get-NetRoute -DestinationPrefix "0.0.0.0/0" -InterfaceAlias "<AdapterName>" | Remove-NetRoute -Confirm:$false

# Re-add correct default route (replace <GatewayIP> with actual gateway)
New-NetRoute -DestinationPrefix "0.0.0.0/0" -NextHop "<GatewayIP>" -InterfaceAlias "<AdapterName>"
```

**验证方法**：

```powershell
Get-NetRoute -DestinationPrefix "0.0.0.0/0" | Select-Object InterfaceAlias, NextHop
```

预期结果：NextHop 为有效的 IP 地址（不是 0.0.0.0）

**风险说明**：配置错误的网关会导致外网不可达

---

### 根因：未配置 DNS 服务器

**修复操作**：

```powershell
# Get active network adapter (replace <AdapterName> with actual name)
$adapter = Get-NetAdapter -Name "<AdapterName>"
if ($adapter) {
    # Set DNS servers
    Set-DnsClientServerAddress -InterfaceIndex $adapter.InterfaceIndex -ServerAddresses @("100.100.2.136", "100.100.2.138")
}
```

**验证方法**：

```powershell
Get-DnsClientServerAddress -AddressFamily IPv4 | Where-Object {$_.ServerAddresses -ne $null} | Select-Object InterfaceAlias, ServerAddresses
```

预期结果：ServerAddresses 包含有效的 DNS 服务器 IP

**风险说明**：错误的 DNS 配置会导致域名解析失败

---

### 根因：DNS 服务器不可达

**修复操作**：

```powershell
# Change DNS servers (replace <AdapterName> with actual name)
$adapter = Get-NetAdapter -Name "<AdapterName>"
if ($adapter) {
    Set-DnsClientServerAddress -InterfaceIndex $adapter.InterfaceIndex -ServerAddresses @("100.100.2.136", "100.100.2.138")
}
```

**验证方法**：

```powershell
Test-Connection -ComputerName 100.100.2.136 -Count 2
```

预期结果：DNS 服务器可达

**风险说明**：使用公网 DNS 可能受防火墙策略限制

---

### 根因：TCP 动态端口范围过小

**修复操作**：

```powershell
# Restore default dynamic port range (49152-65535, 16384 ports total)
netsh int ipv4 set dynamicport tcp start=49152 num=16384
netsh int ipv6 set dynamicport tcp start=49152 num=16384
netsh int ipv4 set dynamicport udp start=49152 num=16384
netsh int ipv6 set dynamicport udp start=49152 num=16384
```

**验证方法**：

```powershell
# Verify dynamic port range restored
netsh int ipv4 show dynamicport tcp
```

预期结果：启动端口 = 49152,端口数 = 16384

**风险说明**：修改后立即生效,现有连接不受影响,但新建连接会使用新的端口范围

---

### 根因:TCP 排除端口过多

**说明**：

TCP 排除端口由系统服务(Hyper-V、NAT、Windows Update 等)自动管理,用于保留特定端口范围供系统服务使用。排除端口过多通常是以下原因:

1. **Hyper-V 服务**: 创建虚拟机或容器时会动态分配排除端口
2. **Windows NAT 服务**: WSL2、Docker 等会注册端口排除范围
3. **其他系统服务**: 某些 Windows 服务启动时会申请端口保留

排除端口属于系统正常行为,不建议手动删除,可能导致相关服务异常。如果排除端口影响应用运行,可尝试重启相关服务释放端口。

---

### 根因：网关不可达

**修复操作**：

```powershell
# Check ARP table
# Check ARP table
Get-NetNeighbor | Where-Object { $_.AddressFamily -eq 'IPv4' } | Select-Object IPAddress, LinkLayerAddress, State | Format-Table -AutoSize

# Check network adapter configuration
Get-NetAdapter | Where-Object {$_.Status -eq 'Up'} | Select-Object Name, InterfaceDescription

# If in a virtualized environment, check cloud platform routing table configuration
```

**验证方法**：

```powershell
Test-Connection -ComputerName <GatewayIP> -Count 2
```

预期结果：网关可达

**风险说明**：网关不可达可能是网络基础设施问题，需要联系网络管理员

---

### 根因：DNS 解析失败

**修复操作**：

```powershell
# Clear DNS cache
Clear-DnsClientCache

# Refresh DNS registration
ipconfig /flushdns
ipconfig /registerdns
```

**验证方法**：

```powershell
Resolve-DnsName -Name www.aliyun.com -Type A | Select-Object Name, IPAddress
```

预期结果：能成功解析域名

**风险说明**：清除缓存后首次解析可能稍慢

---

### 根因：WinHTTP 代理与 IE 代理不一致

**修复操作**：

```powershell
# Sync WinHTTP proxy with IE settings
netsh winhttp import proxy source=ie
```

**验证方法**：

```powershell
netsh winhttp show proxy
```

预期结果：WinHTTP 代理与 IE 代理配置一致

**风险说明**：同步代理配置可能影响某些服务的网络连接

---

### 根因：检测到环境变量代理配置

**修复操作**：

```powershell
# View current proxy environment variables
Write-Output "HTTP_PROXY=$($env:HTTP_PROXY)"
Write-Output "HTTPS_PROXY=$($env:HTTPS_PROXY)"

# Clear proxy environment variables (temporary, current session)
$env:HTTP_PROXY = $null
$env:HTTPS_PROXY = $null
$env:NO_PROXY = $null

# To permanently remove, modify system environment variables
# [Environment]::SetEnvironmentVariable("HTTP_PROXY", $null, "Machine")
# [Environment]::SetEnvironmentVariable("HTTPS_PROXY", $null, "Machine")
```

**验证方法**：

```powershell
Write-Output "HTTP_PROXY=$($env:HTTP_PROXY)"
Write-Output "HTTPS_PROXY=$($env:HTTPS_PROXY)"
```

预期结果：环境变量代理为空

**风险说明**：清除环境变量代理可能影响依赖代理的应用程序

---

### 根因：代理配置错误

**修复操作**：

```powershell
netsh winhttp reset proxy

# Disable IE proxy (registry)
Set-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings" -Name ProxyEnable -Value 0 -ErrorAction SilentlyContinue
```

**验证方法**：

```powershell
netsh winhttp show proxy
Get-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings" -Name ProxyEnable, ProxyServer | Select-Object ProxyEnable, ProxyServer
```

预期结果：代理已禁用，配置为直连

**风险说明**：如果网络环境确实需要代理，禁用代理会导致外网不可达
