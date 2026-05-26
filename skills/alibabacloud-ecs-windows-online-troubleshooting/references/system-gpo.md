# System Group Policy 诊断

## 功能说明

诊断 Windows 组策略应用状态、AppLocker/软件限制策略、驱动器映射和驱动安装策略。覆盖 4 个已知问题项。

**输入**：用户问题描述（必选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| gpupdate 报错、登录脚本不执行、策略设置不生效 | Step 1 (组策略应用状态) |
| 应用被阻止运行、提示"已被系统管理员阻止" | Step 2 (AppLocker/软件限制策略) |
| 登录时网络驱动器不出现 | Step 3 (驱动器映射) |
| 无法安装新驱动、设备管理器新设备显示感叹号 | Step 4 (驱动安装策略) |

## 诊断步骤

### Step 1: 组策略应用状态检查

**数据采集**：

> 采集目标：获取组策略应用结果，检查是否有应用失败的策略

```powershell
gpresult /r /scope:computer 2>&1
```

```powershell
# Check Group Policy related event logs
Get-WinEvent -FilterHashtable @{LogName='System'} -MaxEvents 20 -ErrorAction SilentlyContinue |
  Where-Object { $_.ProviderName -eq 'Microsoft-Windows-GroupPolicy' -and $_.Level -le 3 } |
  Select-Object TimeCreated, Id, LevelDisplayName, Message |
  Format-Table -AutoSize -Wrap
```

**分析思路**：

1. 检查组策略应用状态：
   - 正常：gpresult 正常返回且无错误 → 组策略应用成功
   - 异常：gpresult 报错或事件日志含 Error/Warning → **根因**：组策略应用失败，**严重程度**：Warning

### Step 2: AppLocker/软件限制策略检查

**数据采集**：

> 采集目标：检查 AppLocker 事件日志中的阻止事件

```powershell
Get-WinEvent -LogName 'Microsoft-Windows-AppLocker/EXE and DLL' -MaxEvents 20 -ErrorAction SilentlyContinue |
  Select-Object TimeCreated, Id, LevelDisplayName, Message |
  Format-Table -AutoSize -Wrap
```

```powershell
# Check Software Restriction Policies (SRP)
Get-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\Safer\CodeIdentifiers' -ErrorAction SilentlyContinue |
  Select-Object DefaultLevel, TransparentEnabled, PolicyScope |
  Format-Table -AutoSize
```

**分析思路**：

1. 检查 AppLocker 阻止事件：
   - 正常：AppLocker 日志无阻止事件
   - 异常：AppLocker 日志存在阻止事件 → **根因**：AppLocker 正在阻止应用运行（AppLockerBlockEvent），**严重程度**：Warning
2. 检查 SRP 默认安全级别：
   - 正常：SRP 未配置或 DefaultLevel 不为 Disallowed
   - 异常：SRP DefaultLevel=0（Disallowed） → **根因**：软件限制策略设置为禁止运行，**严重程度**：Warning

### Step 3: 驱动器映射检查

**数据采集**：

> 采集目标：检查组策略配置的驱动器映射

```powershell
# Check drive mappings in logon scripts (Group Policy Preferences)
Get-ItemProperty -Path 'HKCU:\Network\*' -ErrorAction SilentlyContinue |
  Select-Object PSChildName, RemotePath, UserName |
  Format-Table -AutoSize
# Current network drives
Get-PSDrive -PSProvider FileSystem | Where-Object { $_.DisplayRoot -ne $null } |
  Select-Object Name, DisplayRoot |
  Format-Table -AutoSize
```

**分析思路**：

1. 检查网络驱动器映射：
   - 正常：网络驱动器正常映射
   - 异常：配置了映射但实际未建立网络连接 → 可能是网络不可达或凭据失效

### Step 4: 驱动安装策略检查

**数据采集**：

> 采集目标：检查是否通过组策略禁止了设备驱动安装

```powershell
# Windows 2016+ check
Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\DeviceInstall\Parameters' -Name 'DeviceInstallDisabled' -ErrorAction SilentlyContinue |
  Select-Object DeviceInstallDisabled | Format-Table -AutoSize
# Windows 2012 and earlier check
Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\PlugPlay\Parameters' -Name 'DeviceInstallDisabled' -ErrorAction SilentlyContinue |
  Select-Object DeviceInstallDisabled | Format-Table -AutoSize
```

**分析思路**：

1. 检查 DeviceInstallDisabled 配置：
   - 正常：DeviceInstallDisabled 不存在或为 0 → 驱动安装未被禁止
   - 异常：DeviceInstallDisabled 非 0 → **根因**：驱动安装已被策略禁止（DriverInstallDisabled），**严重程度**：Warning

> 此策略会阻止新硬件驱动的自动安装，影响 VirtIO 驱动更新等场景。

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Step 4 发现驱动安装被禁止 | → [cloud-driver.md](references/cloud-driver.md)（驱动安装策略检查） |
| 条件跳转 | AppLocker 阻止应用运行 | → [security-malware.md](references/security-malware.md)（IFEO 检查排除恶意软件） |

## 修复建议

### Fix 1: 强制刷新组策略

**适用场景**：组策略应用失败

```powershell
gpupdate /force
```

### Fix 2: 处理 AppLocker 阻止（AppLockerBlockEvent）

**适用根因**：AppLockerBlockEvent

1. 查看被阻止的应用路径，确认是否为合法应用
2. 如需允许该应用，在本地安全策略中添加豁免规则：
   ```powershell
   # Open Local Security Policy editor
   secpol.msc
   # Navigate to: Application Control Policies → AppLocker → Executable Rules
   ```
3. 如果策略由域推送，需联系域管理员修改

### Fix 3: 解除驱动安装禁止（DriverInstallDisabled）

**适用根因**：DriverInstallDisabled

```powershell
# Windows 2016+
Remove-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\DeviceInstall\Parameters' -Name 'DeviceInstallDisabled' -ErrorAction SilentlyContinue
# If set by Group Policy, modify via gpedit.msc:
# Computer Configuration → Administrative Templates → System → Device Installation → Set to "Not Configured"
```
