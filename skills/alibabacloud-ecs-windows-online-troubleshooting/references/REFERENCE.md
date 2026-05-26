# 诊断能力与问题路由

本文档定义本 skill 的诊断能力清单和问题路由表,供路径规划时加载参考。

---

## 一、诊断能力声明

### 1. Network(网络)

| # | Reference 文件 | 简述|
|---|---------------|------|
| 1 | `references/networking-tcpip.md` | 网卡状态检查、IP/子网/网关配置验证、协议绑定、路由表分析、代理配置、端到端连通性探测|
| 2 | `references/networking-firewall.md` | 防火墙服务状态、配置文件(Domain/Private/Public)检查、关键端口规则(80/443/3389/53)、WFP 丢包事件、元数据服务端口|
| 3 | `references/networking-dns.md` | DNS Client 服务状态、DNS 服务器配置验证、DNS 缓存检查、域名解析测试|
| 4 | `references/networking-dhcp.md` | DHCP Client 服务状态、网卡 DHCP 启用检查、租约获取与续租、APIPA 地址检测、DHCP 客户端事件日志、防火墙 UDP 67/68 端口检查、DHCP 服务器连通性|

### 2. Storage(存储)

| # | Reference 文件 | 简述|
|---|---------------|------|
| 5 | `references/storage-disk.md` | 磁盘在线/离线状态、SAN 策略、动态磁盘、UniqueID 冲突、分区表与分区状态、盘符挂载、卷空间使用率、扩容可行性、Cluster Size 限制、MFT 大小、文件系统错误、组策略隐藏驱动器|
| 6 | `references/storage-hardware.md` | SCSI 控制器状态、磁盘驱动状态、过滤驱动检查(残留/缺失/异常)、VirtIO 存储错误事件(Event 11)、设备移除失败事件(Event 225)|
| 7 | `references/storage-vss.md` | VSS Writer 状态、快照创建、备份软件执行|
| 8 | `references/storage-smb.md` | SMB 协议版本配置、Server/Workstation 服务状态、网络发现与文件共享、共享权限与 NTFS 权限、DFS Namespace 故障、SMB 端口(445)检查|

### 3. Remote Access(远程访问)

| # | Reference 文件 | 简述|
|---|---------------|------|
| 9 | `references/rdp-service.md` | TermService 服务状态及依赖服务、WinStation 监听器配置与端口监听（含端口冲突、第三方 Station 检测、第三方 WinStation 与 RDP-Tcp 冲突导致闪退、Session Listener、MaxInstanceCount、注册表 ACL）、RDP 启用状态与组策略覆盖检查、UMBus 设备枚举 |
| 10 | `references/rdp-auth.md` | CredSSP 配置与加密 Oracle 修补、账户锁定状态、远程登录权限（含 SeDenyRemoteInteractiveLogonRight 拒绝策略）、安全层配置、密码错误事件（Event 4625）、ForceGuest 访问模式|
| 11 | `references/rdp-certificate.md` | RDP 证书状态检查(过期/无效/自签)、MachineKeys 目录与 TLS 私钥文件权限、证书自动更新机制、系统盘根目录权限|
| 12 | `references/rdp-licensing.md` | RDS 授权状态、Grace Period 剩余天数、授权模式配置、RDS 相关服务状态、GracePeriod/GNR 注册表项导致 RDP 闪退|

### 4. Identity & Access(身份与访问控制)

| # | Reference 文件 | 简述|
|---|---------------|------|
| 13 | `references/identity-account.md` | 账户锁定状态、密码策略、账户锁定策略、密码过期、内置 Administrator 账户存在性|
| 14 | `references/identity-auth.md` | Kerberos 时钟偏差、NTLM 认证配置、SPN 配置 |
| 15 | `references/identity-user-profiles.md` | 用户配置文件损坏、文件夹重定向失败、自定义/默认配置文件导致登录缓慢或性能下降|
| 16 | `references/identity-permission.md` | 系统盘根目录权限、远程登录权限、Temp 文件夹权限、ForceGuest 配置、用户组成员关系、Guest 账户状态|
| 17 | `references/identity-ad.md` | SID 冲突、域安全通道状态、域控制器可达性、计算机账户密码、LDAPS 连接 |

### 5. Performance(性能与稳定性)

| # | Reference 文件 | 简述|
|---|---------------|------|
| 18 | `references/performance-slow.md` | CPU/内存/句柄使用率、内存资源耗尽事件、页面文件配置(含崩溃转储依赖)、硬件保留内存、超线程状态(Spectre/Meltdown 缓解)、BCD CPU/内存限制、电源计划、文件系统过滤驱动(第三方 Minifilter) |
| 19 | `references/performance-lifecycle.md` | 关机卡住/超时、待重启操作状态、启动耗时与退化分析、异常关机/重启事件、启动阶段分解、关机耗时分析|

### 6. System Services(系统服务与配置)

| # | Reference 文件 | 简述|
|---|---------------|------|
| 20 | `references/system-activation.md` | sppsvc 服务状态、Windows 激活状态、产品密钥验证、KMS 服务器可达性、防火墙端口检查、SPP 事件检查、Tokens.dat 重建|
| 21 | `references/system-update.md` | Windows Update 依赖服务、WSUS 配置、更新服务器可达性、已知问题补丁、WinHTTP 代理配置 |
| 22 | `references/system-time.md` | 时区/夏令时配置、RealTimeIsUniversal 硬件时钟解释、W32Time 服务状态、Secure Time Seeding (STS) 检查、NTP 服务器配置 |
| 23 | `references/system-gpo.md` | 组策略应用状态、AppLocker/软件限制策略、驱动器映射、驱动安装策略|
| 24 | `references/system-management.md` | PowerShell 执行策略、WinRM、WMI 仓库、事件日志服务、MMC 控制台|
| 25 | `references/system-schtasks.md` | Task Scheduler 服务状态、任务启动失败、任务未运行、凭据无效、触发器配置错误、依赖程序缺失、任务损坏与缓存、SPP 权限|
| 26 | `references/system-crash.md` | BugCheck 蓝屏事件(Event 1001)、内存资源耗尽事件(Event 2004)、Crash Dump 配置 |

### 7. Device Management(设备管理)

| # | Reference 文件 | 简述|
|---|---------------|------|
| 27 | `references/device-driver.md` | 设备驱动状态、驱动安装/版本/签名、设备管理器异常(黄色感叹号/错误代码)、驱动服务状态|

### 8. Desktop & Application(桌面与应用)

| # | Reference 文件 | 简述|
|---|---------------|------|
| 28 | `references/desktop-shell.md` | 登录 Shell 配置、Explorer.exe 进程状态、任务栏/开始菜单响应、桌面图标、Console 会话状态、DWM、DPI 缩放 |
| 29 | `references/desktop-app.md` | .NET Framework 状态、MSI 安装/卸载、COM/DCOM 组件注册 |
| 30 | `references/desktop-printing.md` | Print Spooler 服务、打印驱动安装、打印输出状态|

### 9. Security(安全防护)

| # | Reference 文件 | 简述|
|---|---------------|------|
| 31 | `references/security-malware.md` | IFEO 映像劫持检测、IFEO 堆栈提交值、可疑 IFEO 配置、第三方杀毒软件进程|
| 32 | `references/security-bitlocker.md` | BitLocker 加密状态、恢复密钥状态|
| 33 | `references/security-certificates.md` | 根证书状态、证书链完整性、TLS 协议版本/加密套件、驱动签名根证书 |

### 10. Cloud Platform(云平台)

| # | Reference 文件 | 简述|
|---|---------------|------|
| 34 | `references/cloud-metaserver.md` | 元数据服务可达性(100.100.100.200)、NTP 时间同步(W32Time 服务与时间源配置)、KMS 激活服务器(许可证状态诊断)、WSUS 更新服务器(UseWUServer 配置)、主机名一致性 |
| 35 | `references/cloud-vminit.md` | vminit 服务安装状态、服务启用状态、初始化日志错误、user-data 执行失败 |
| 36 | `references/cloud-driver.md` | VirtIO 驱动版本、VirtIO 驱动存在性、VirtIO 驱动签名状态、VirtIO 驱动服务状态、Xen 驱动残留 |

---

## 二、问题路由表

根据用户问题描述,匹配到一组 reference 的排查序列。匹配规则:精确匹配 → 模糊匹配 → LLM 动态规划。

> 路由表按**诊断频率**排序（非能力清单的分类编号顺序），高频场景靠前。

### 2.1 Network(网络)

| 典型用户问题 | 关键词| 排查序列(按优先级) |
|-------------|--------|-------------------|
| **网络不通/无法上网** | ping 不通、无法上网、网络断开、网络图标红叉| `references/networking-tcpip.md` → `references/networking-firewall.md` → `references/networking-dns.md` |
| **DNS 解析失败** | DNS、域名解析失败、nslookup 报错、网站打不开但能 ping 通| `references/networking-dns.md` → `references/networking-tcpip.md` |
| **防火墙阻止端口** | 端口被阻止、防火墙、无法访问某端口 | `references/networking-firewall.md` → `references/networking-tcpip.md` |
| **无法获取 IP 地址** | 169.254、APIPA、DHCP 失败、自动私有地址 | `references/networking-dhcp.md` → `references/networking-tcpip.md` |
| **共享文件夹无法访问** | SMB、网络共享、DFS、访问共享被拒绝 | `references/storage-smb.md` → `references/networking-firewall.md` |

### 2.2 Storage(存储)

| 典型用户问题 | 关键词| 排查序列(按优先级) |
|-------------|--------|-------------------|
| **磁盘不可见/数据盘未显示** | 磁盘不可见、数据盘、新挂载、脱机、找不到磁盘、磁盘没有盘符、磁盘未联机| `references/storage-disk.md` → `references/storage-hardware.md` |
| **磁盘扩容后容量未变** | 扩容、磁盘大小、未分配空间、MBR 2TB、Cluster Size | `references/storage-disk.md` |
| **磁盘 I/O 异常/性能下降** | 磁盘慢、IO 错误、MFT、Event ID 11、Event ID 225 | `references/storage-hardware.md` → `references/storage-disk.md` |
| **备份失败/VSS 错误** | 备份失败、VSS、快照、还原点 | `references/storage-vss.md` |

### 2.3 Device Management(设备管理)

| 典型用户问题 | 关键词| 排查序列(按优先级) |
|-------------|--------|-------------------|
| **设备管理器黄色感叹号** | 黄色感叹号、设备无法启动、错误代码、驱动异常| `references/device-driver.md` |
| **驱动安装失败/版本过旧** | 驱动安装、驱动版本、驱动更新、签名验证失败| `references/device-driver.md` |
| **网卡消失/网络适配器异常** | 网卡不见、网络适配器黄色感叹号、无网络连接、网卡驱动 | `references/device-driver.md` → `references/networking-tcpip.md` |

### 2.4 Remote Access(远程访问)

| 典型用户问题 | 关键词| 排查序列(按优先级) |
|-------------|--------|-------------------|
| **远程桌面连不上** | RDP、远程桌面、3389、mstsc、连接被拒绝、无法远程、远程失败、远程超时 | `references/rdp-service.md` → `references/rdp-auth.md` |
| **RDP 认证失败/登录错误** | 认证错误、凭据、密码错误、CredSSP、登录被拒绝 | `references/rdp-auth.md` → `references/rdp-certificate.md` |
| **RDP 证书警告** | 证书错误、证书过期、证书不受信任、Internal error | `references/rdp-certificate.md` → `references/rdp-auth.md` |
| **RDS 授权错误** | 授权过期、licensing mode、多用户连接限制 | `references/rdp-licensing.md` |
| **RDP 闪退** | 远程连接闪退、连接后立即断开、远程登录后被注销、RDP window closes immediately | `references/rdp-service.md` → `references/rdp-licensing.md` → `references/identity-user-profiles.md` |

### 2.5 Identity & Access(身份与访问控制)

| 典型用户问题 | 关键词| 排查序列(按优先级) |
|-------------|--------|-------------------|
| **账户被锁定/无法登录** | 账户锁定、locked out、密码正确但无法登录 | `references/identity-account.md` → `references/identity-permission.md` |
| **域登录失败/信任关系失败** | 域登录、信任关系、Secure Channel、域控制器| `references/identity-ad.md` → `references/identity-auth.md` |
| **登录后加载临时配置文件** | 临时配置文件、桌面设置丢失、配置文件损坏| `references/identity-user-profiles.md` |
| **登录缓慢/配置文件性能下降** | 登录慢、配置文件加载慢、自定义默认配置文件、CopyProfile、Default 配置文件 | `references/identity-user-profiles.md` |
| **权限不足/访问被拒绝** | 访问被拒绝、权限不足、管理员权限、Guest | `references/identity-permission.md` |
| **域认证配置异常** | Kerberos、NTLM、SPN、时钟偏差、认证失败 | `references/identity-auth.md` → `references/identity-ad.md` |

### 2.6 Performance(性能与稳定性，含崩溃分析)

| 典型用户问题 | 关键词| 排查序列(按优先级) |
|-------------|--------|-------------------|
| **系统卡顿/响应缓慢** | 卡顿、慢、CPU 高、内存不足、无响应 | `references/performance-slow.md` |
| **打开文件或程序缓慢** | 文件打开慢、程序启动延迟、双击无反应 | `references/performance-slow.md` |
| **蓝屏/系统崩溃** | 蓝屏、BSOD、BugCheck、崩溃重启| `references/system-crash.md` → `references/cloud-driver.md` |
| **系统完全挂起** | 挂起、死机、无反应、强制重启| `references/desktop-shell.md`（Console 会话状态）→ `references/performance-lifecycle.md`（挂起事件分析）→ `references/performance-slow.md` |
| **关机卡住/关机慢** | 关机慢、正在关机、关机超时| `references/performance-lifecycle.md` |
| **开机慢/启动耗时过长** | 开机慢、启动慢、一直转圈、系统启动卡住 | `references/performance-lifecycle.md` → `references/device-driver.md` |

### 2.7 System Services(系统服务与配置)

| 典型用户问题 | 关键词| 排查序列(按优先级) |
|-------------|--------|-------------------|
| **Windows 未激活** | 激活、Activate Windows、水印、sppsvc | `references/system-activation.md` → `references/cloud-metaserver.md` |
| **Windows Update 失败** | 更新失败、WSUS、0x80070422、更新卡住| `references/system-update.md` → `references/networking-firewall.md` |
| **系统时间不准确** | 时间不对、时钟偏差、时区、夏令时 | `references/system-time.md` → `references/cloud-metaserver.md` |
| **组策略不生效** | 组策略、GPO、gpupdate、登录脚本、AppLocker | `references/system-gpo.md` |
| **云助手/WMI/WinRM 异常** | 云助手、WMI、WinRM、远程管理 | `references/system-management.md` → `references/cloud-vminit.md` |
| **计划任务不执行/启动失败** | 计划任务、Task Scheduler、定时任务、Event ID 101、sppsvc 任务失败 | `references/system-schtasks.md` → `references/system-activation.md` |

### 2.8 Desktop & Application(桌面与应用)

| 典型用户问题 | 关键词| 排查序列(按优先级) |
|-------------|--------|-------------------|
| **登录后黑屏/无桌面** | 黑屏、无桌面、只有壁纸、Explorer | `references/desktop-shell.md` |
| **应用启动失败/报错** | 应用无法启动、缺少 DLL、.NET、MSI 失败 | `references/desktop-app.md` |
| **打印服务异常** | 打印失败、Print Spooler、打印机不出现| `references/desktop-printing.md` |

### 2.9 Security(安全防护)

| 典型用户问题 | 关键词| 排查序列(按优先级) |
|-------------|--------|-------------------|
| **程序无法启动/被劫持** | 双击无反应、找不到文件、IFEO、映像劫持| `references/security-malware.md` |
| **BitLocker 锁定/需要恢复密钥** | BitLocker、恢复密钥、加密锁定| `references/security-bitlocker.md` |
| **HTTPS/SSL 证书错误** | 证书错误、HTTPS 失败、TLS、证书链 | `references/security-certificates.md` |

### 2.10 Cloud Platform(云平台)

| 典型用户问题 | 关键词| 排查序列(按优先级) |
|-------------|--------|-------------------|
| **元数据服务不可达** | 元数据、100.100.100.200、云助手失败 | `references/cloud-metaserver.md` → `references/networking-firewall.md` |
| **vminit 初始化失败** | vminit、cloud-init、初始化失败、密码重置不生效、user-data、自定义脚本 | `references/cloud-vminit.md` |
| **VirtIO 驱动异常/迁移后设备不可用** | VirtIO、驱动、Xen、KVM 迁移、磁盘不可用 | `references/cloud-driver.md` → `references/storage-hardware.md` |

### 2.11 兜底机制

如果用户问题未命中路由表:
- 从诊断能力清单中选择可能相关的 reference,组合排查序列
- MUST 向用户说明:「此为动态规划的排查路径,非预设场景。」
- 如果所有参考均不相关,如实告知用户当前诊断能力未覆盖该场景,给出建议的排查方向
