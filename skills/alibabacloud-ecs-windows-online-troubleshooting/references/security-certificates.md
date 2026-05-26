# Security Certificates 诊断

## 功能说明

诊断 Windows 根证书状态、证书链完整性、TLS 协议版本/加密套件和驱动签名根证书。覆盖 4 个已知问题项。

**输入**：用户问题描述（必选）
**输出**：根因列表（root_cause / severity / evidence / explanation / fix）

## 步骤选取指引

本文件包含多个诊断步骤，**无需全部执行**。根据用户描述的问题现象选取相关步骤：

| 用户问题现象 | 建议执行的步骤 |
|-------------|---------------|
| HTTPS 访问报证书错误、API 调用失败 | Step 1 (根证书状态) |
| SSL/TLS 连接失败 | Step 2 (证书链完整性) |
| TLS 握手失败、部分 HTTPS 站点或 API 无法访问 | Step 3 (TLS 协议版本/加密套件) |
| 驱动安装提示"发布者不受信任" | Step 4 (驱动签名根证书) |

## 诊断步骤

### Step 1: 检查根证书状态

**数据采集**：

> 采集目标：获取本机受信任根证书存储区中的证书列表及过期状态

```powershell
# Check for expired certificates in the root store
$currentDate = Get-Date
$rootCerts = Get-ChildItem -Path Cert:\LocalMachine\Root -ErrorAction SilentlyContinue
$expired = $rootCerts | Where-Object { $_.NotAfter -lt $currentDate }
$expiringSoon = $rootCerts | Where-Object { $_.NotAfter -gt $currentDate -and $_.NotAfter -lt $currentDate.AddDays(30) }
Write-Host "Total root certificates: $($rootCerts.Count)"
Write-Host "Expired certificates: $($expired.Count)"
Write-Host "Expiring within 30 days: $($expiringSoon.Count)"
if ($expired) {
    Write-Host "`nExpired root certificates:"
    $expired | Select-Object Subject, NotAfter, Thumbprint | Format-Table -AutoSize
}
if ($expiringSoon) {
    Write-Host "`nExpiring soon:"
    $expiringSoon | Select-Object Subject, NotAfter, Thumbprint | Format-Table -AutoSize
}
# Check if critical Microsoft root certificates exist
$criticalThumbprints = @(
    'A43489159A520F0D93D032CCAF37E7FE20A8B419',  # Microsoft Root Certificate Authority 2011
    '3B1EFD3A66EA28B16697394703A72CA340A05BD5'   # Microsoft Root Certificate Authority 2010
)
foreach ($tp in $criticalThumbprints) {
    $cert = Get-ChildItem -Path Cert:\LocalMachine\Root | Where-Object { $_.Thumbprint -eq $tp }
    if ($cert) {
        Write-Host "Critical root cert [$tp]: Present, NotAfter=$($cert.NotAfter)"
    } else {
        Write-Host "Critical root cert [$tp]: MISSING"
    }
}
```

**分析思路**：

1. 检查过期根证书：
   - 正常：无过期根证书，或过期证书非关键证书
   - 异常：关键微软根证书过期或缺失 → **根因**：关键根证书缺失或过期，将导致 HTTPS 访问、驱动签名验证、Windows Update 等失败，**严重程度**：Critical

### Step 2: 检查证书链完整性

**数据采集**：

> 采集目标：检查中间证书存储区和个人证书存储区的证书链状态

```powershell
# Check the Intermediate Certificates store
$currentDate = Get-Date
$intermediateCerts = Get-ChildItem -Path Cert:\LocalMachine\CA -ErrorAction SilentlyContinue
$expiredCA = $intermediateCerts | Where-Object { $_.NotAfter -lt $currentDate }
Write-Host "Intermediate certificates: $($intermediateCerts.Count)"
Write-Host "Expired intermediate certificates: $($expiredCA.Count)"
if ($expiredCA.Count -gt 0 -and $expiredCA.Count -le 10) {
    $expiredCA | Select-Object Subject, Issuer, NotAfter | Format-Table -AutoSize
}
# Check certificate chain issues in the Personal store
$personalCerts = Get-ChildItem -Path Cert:\LocalMachine\My -ErrorAction SilentlyContinue
foreach ($cert in $personalCerts) {
    $chain = New-Object System.Security.Cryptography.X509Certificates.X509Chain
    $chain.ChainPolicy.RevocationMode = [System.Security.Cryptography.X509Certificates.X509RevocationMode]::NoCheck
    $valid = $chain.Build($cert)
    if (-not $valid) {
        Write-Host "Certificate chain issue for: $($cert.Subject)"
        Write-Host "  Thumbprint: $($cert.Thumbprint)"
        Write-Host "  NotAfter: $($cert.NotAfter)"
        foreach ($status in $chain.ChainStatus) {
            Write-Host "  ChainStatus: $($status.StatusInformation.Trim())"
        }
    }
}
if (-not $personalCerts) {
    Write-Host "No certificates in LocalMachine\My store"
}
```

**分析思路**：

1. 检查中间证书过期情况：
   - 正常：无影响关键服务的过期中间证书
   - 异常：大量中间证书过期 → **根因**：中间证书过期导致证书链验证失败，**严重程度**：Warning
2. 检查个人证书链状态：
   - 正常：所有证书链构建成功
   - 异常：证书链构建失败（缺少中间证书、根证书不受信任等） → **根因**：证书链不完整，相关 SSL/TLS 服务将失败，**严重程度**：Critical

### Step 3: 检查 TLS 协议版本/加密套件

**数据采集**：

> 采集目标：获取系统 TLS/SSL 协议版本启用状态和加密套件配置

```powershell
# Check TLS/SSL protocol version enablement status
$protocols = @('SSL 2.0','SSL 3.0','TLS 1.0','TLS 1.1','TLS 1.2','TLS 1.3')
foreach ($proto in $protocols) {
    $clientPath = "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\$proto\Client"
    $serverPath = "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\$proto\Server"
    $clientEnabled = (Get-ItemProperty -Path $clientPath -Name 'Enabled' -ErrorAction SilentlyContinue).Enabled
    $serverEnabled = (Get-ItemProperty -Path $serverPath -Name 'Enabled' -ErrorAction SilentlyContinue).Enabled
    $clientDisabled = (Get-ItemProperty -Path $clientPath -Name 'DisabledByDefault' -ErrorAction SilentlyContinue).DisabledByDefault
    $serverDisabled = (Get-ItemProperty -Path $serverPath -Name 'DisabledByDefault' -ErrorAction SilentlyContinue).DisabledByDefault
    Write-Host "$proto - Client: Enabled=$clientEnabled,DisabledByDefault=$clientDisabled | Server: Enabled=$serverEnabled,DisabledByDefault=$serverDisabled"
}
# Check for enforced cipher suite policy
$cipherPolicy = (Get-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Cryptography\Configuration\SSL\00010002' -Name 'Functions' -ErrorAction SilentlyContinue).Functions
if ($cipherPolicy) {
    Write-Host "`nCipher suite policy (group policy):"
    Write-Host $cipherPolicy
} else {
    Write-Host "`nNo cipher suite group policy configured (using system defaults)"
}
```

**分析思路**：

1. 检查 TLS 协议版本：
   - 正常：TLS 1.2 已启用，SSL 2.0/3.0 已禁用
   - 异常：TLS 1.2 被显式禁用（Enabled=0） → **根因**：TLS 1.2 被禁用，大量现代网站和 API 将无法访问，**严重程度**：Critical
   - 异常：仅启用 TLS 1.0/1.1，未启用 TLS 1.2 → **根因**：仅支持过时的 TLS 版本，存在安全风险且兼容性下降，**严重程度**：Warning
2. 检查加密套件策略：
   - 正常：未配置限制性策略，使用系统默认
   - 异常：策略限制了必要的加密套件 → **根因**：加密套件策略过于严格，部分 TLS 连接将失败，**严重程度**：Warning

### Step 4: 检查驱动签名根证书

**数据采集**：

> 采集目标：检查驱动签名所需的微软代码签名根证书是否存在

```powershell
# Check critical certificates related to driver signing
$driverSignCerts = @(
    @{Thumbprint='8FBE4D070EF8AB1BCCAF2A9D5CCAE7282A2C66B3'; Name='Microsoft Code Signing PCA 2011'},
    @{Thumbprint='CDD4EEAE6000AC7F40C3802C171E30148030C072'; Name='Microsoft Root Certificate Authority 2010'}
)
foreach ($c in $driverSignCerts) {
    $found = Get-ChildItem -Path Cert:\LocalMachine\Root -Recurse | Where-Object { $_.Thumbprint -eq $c.Thumbprint }
    if (-not $found) {
        $found = Get-ChildItem -Path Cert:\LocalMachine\CA -Recurse | Where-Object { $_.Thumbprint -eq $c.Thumbprint }
    }
    if ($found) {
        Write-Host "$($c.Name) [$($c.Thumbprint)]: Present, NotAfter=$($found.NotAfter)"
    } else {
        Write-Host "$($c.Name) [$($c.Thumbprint)]: MISSING"
    }
}
# Check driver signing enforcement policy
$driverSigning = (Get-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Driver Signing' -Name 'Policy' -ErrorAction SilentlyContinue).Policy
Write-Host "Driver Signing Policy: $driverSigning (0=None, 1=Warn, 2=Block)"
```

**分析思路**：

1. 检查驱动签名根证书：
   - 正常：所有关键代码签名证书存在且未过期
   - 异常：关键代码签名证书缺失 → **根因**：驱动签名根证书缺失，驱动安装将提示“发布者不受信任”，**严重程度**：Critical
2. 检查驱动签名策略：
   - 正常：Policy 为 1（警告）或 2（阻止）
   - 异常：Policy 为 0（忽略）可能允许未签名驱动加载 → **根因**：驱动签名验证被禁用，存在安全风险，**严重程度**：Warning

## 交叉引用

| 类型 | 触发条件 | 跳转目标 |
|------|---------|--------|
| 条件跳转 | Step 3 TLS 配置异常影响 RDP 连接 | → [rdp-certificate.md](references/rdp-certificate.md) |
| 条件跳转 | Step 4 驱动签名证书缺失 | → [cloud-driver.md](references/cloud-driver.md) |
| 链式后继 | 本文件未确认根因 | → [security-bitlocker.md](references/security-bitlocker.md) |

## 修复建议

### 根因: 关键根证书缺失或过期

**修复操作**：

```powershell
# Update root certificates via Windows Update
certutil -generateSSTFromWU roots.sst
certutil -addstore Root roots.sst
Remove-Item roots.sst -Force
```

**验证方法**：

```powershell
Get-ChildItem -Path Cert:\LocalMachine\Root | Where-Object { $_.Thumbprint -eq 'A43489159A520F0D93D032CCAF37E7FE20A8B419' }
```

预期结果：返回 Microsoft Root Certificate Authority 2011 证书信息

**风险说明**：此操作从微软官方服务器下载根证书，需要网络能访问微软更新服务器。在完全离线环境中可能失败。

### 根因: TLS 1.2 被禁用

**修复操作**：

```powershell
# Enable TLS 1.2 for both Client and Server
$protocols = @('Client','Server')
foreach ($side in $protocols) {
    $path = "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\$side"
    if (-not (Test-Path $path)) { New-Item -Path $path -Force | Out-Null }
    Set-ItemProperty -Path $path -Name 'Enabled' -Value 1 -Type DWord
    Set-ItemProperty -Path $path -Name 'DisabledByDefault' -Value 0 -Type DWord
}
Write-Host "TLS 1.2 enabled. Reboot required to take effect."
```

**验证方法**：

```powershell
$clientEnabled = (Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Client' -Name 'Enabled').Enabled
Write-Host "TLS 1.2 Client Enabled: $clientEnabled"
```

预期结果：返回 1

**风险说明**：启用 TLS 1.2 是推荐操作，但需要重启才生效。极少数老旧应用可能不支持 TLS 1.2，启用前确认应用兼容性。

### 根因: 驱动签名根证书缺失

**修复操作**：

```powershell
# Update root certificates via certutil
certutil -generateSSTFromWU roots.sst
certutil -addstore Root roots.sst
Remove-Item roots.sst -Force
```

**验证方法**：

```powershell
Get-ChildItem -Path Cert:\LocalMachine\Root | Where-Object { $_.Thumbprint -eq 'CDD4EEAE6000AC7F40C3802C171E30148030C072' }
```

预期结果：返回 Microsoft Root Certificate Authority 2010 证书信息

**风险说明**：同上，需要网络可达微软更新服务器。
