# Identity Auth 诊断

## 功能说明

诊断 Windows Kerberos 时钟偏差、NTLM 认证配置和 SPN 配置。覆盖 3 个已知问题项。

**输入**: 用户问题描述(必选)
**输出**: 根因列表(root_cause / severity / evidence / explanation / fix)

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| 域登录失败、提示时钟偏差过大 | Step 1 (Kerberos 时钟偏差) |
| 远程访问被拒绝、NTLMv1 被策略阻止 | Step 2 (NTLM 认证配置) |
| SQL Server 等服务 Kerberos 委派失败 | Step 3 (SPN 配置) |

## 诊断步骤

### Step 1: Kerberos 时钟偏差检查

**数据采集**：

> 采集目标：检查本机时间与域控制器的时间偏差（Kerberos 默认允许 5 分钟偏差）

```powershell
# Local time
$localTime = Get-Date
Write-Output "Local Time: $localTime"
# NTP sync status
w32tm /query /status 2>&1
# If domain-joined, try querying domain controller time (may fail)
w32tm /stripchart /computer:$env:LOGONSERVER.TrimStart('\\') /samples:1 /dataonly 2>&1
```

**分析思路**：

1. 检查与域控时间偏差：
   - 正常：偏差 < 5 分钟
   - 异常：偏差 >= 5 分钟 → **根因**：Kerberos 时钟偏差过大，将导致域认证失败，**严重程度**：Critical
2. 若无法连接域控 → 参考 [identity-ad.md](references/identity-ad.md) 检查域控可达性

### Step 2: NTLM 认证配置检查

**数据采集**：

> 采集目标：检查 NTLM 认证级别配置

```powershell
Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Lsa' -ErrorAction SilentlyContinue |
  Select-Object LmCompatibilityLevel, NoLmHash |
  Format-Table -AutoSize
```

**分析思路**：

1. 检查 LmCompatibilityLevel 值：
   - 0-2：允许 NTLMv1 → 安全风险较高，但兼容性好
   - 3（推荐）：仅发送 NTLMv2 → **正常**
   - 4-5：拒绝 NTLMv1 → 安全性高，但可能导致旧客户端认证失败

> 如果 LmCompatibilityLevel >= 4 且有旧客户端连接失败，可能需要降低此值。

### Step 3: SPN 配置检查

**数据采集**：

> 采集目标：检查 Service Principal Name 注册情况

```powershell
# View SPNs registered on this machine
setspn -L $env:COMPUTERNAME 2>&1
```

**分析思路**：

1. 检查 SPN 注册情况：
   - 正常：SPN 列表正常显示
   - 异常：显示重复 SPN 或 setspn 报错 → 可能存在 SPN 冲突，影响 Kerberos 委派

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | 时钟偏差涉及 NTP 配置 | → [system-time.md](references/system-time.md) |
| 链式后继 | 本文件未确认根因 | → [identity-ad.md](references/identity-ad.md) |

## 修复建议

### Fix 1: 修复时钟偏差

**适用场景**：Kerberos 时钟偏差过大

```powershell
# Force time sync with domain controller
w32tm /resync /force
# If W32Time is not running
Start-Service W32Time
w32tm /resync /force
```

**验证方法**：

```powershell
w32tm /query /status | Select-String "Last successful sync time"
```

预期结果：显示最近的同步时间，与当前时间偏差在 5 分钟以内

### Fix 2: 调整 NTLM 认证级别

**适用场景**：NTLM 级别过高导致旧客户端失败

```powershell
# Set to send NTLMv2 only (balance security and compatibility)
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Lsa' -Name 'LmCompatibilityLevel' -Value 3 -Type DWord
```

**验证方法**：

```powershell
Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Lsa' -Name 'LmCompatibilityLevel' | Select-Object LmCompatibilityLevel
```

预期结果：`LmCompatibilityLevel` 值为 `3`
